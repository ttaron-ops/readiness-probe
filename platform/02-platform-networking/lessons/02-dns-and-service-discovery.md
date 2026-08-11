---
lesson: "A02.2"
title: "DNS and service discovery"
module: "A-02"
concept: "DNS failure amplification"
status: not-started
est_time: "5 hrs"
prev: "01-tcpip-and-the-packet-path.md"
next: "03-load-balancing.md"
artifacts: ["5s-timeout repro histogram", "ndots query-count calc", "NodeLocal DNSCache tail-latency before/after", "resolver-behavior comparison (glibc/musl/Go)"]
sources: 8
---

# A02.2 · DNS and service discovery

> **Concept.** DNS sits in the request path of everything, so its failure modes are amplifiers — an ndots search-storm or a single dropped UDP query turns into fleet-wide tail latency, and the fix is almost always caching topology, not record content.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 01 established that a correctly-delivered packet is governed by a kernel pipeline with a nameable bottleneck at every stage — but before any of that machinery runs, the client has to resolve a name to an address. This lesson picks up exactly there: DNS is the hop that happens *before* the TCP handshake, on the critical path of every connection, every retry, and every service startup, and it inherits the same discipline — name the exact mechanism and the exact command, don't wave at "DNS is slow." It unlocks lesson 03 (load balancing), where you'll see the same push-vs-poll discovery tension resurface as VIPs and health checks instead of records and TTLs.

## Why this matters

DNS is the most under-respected item in the request path. It resolves before every connection, it is on the critical path of service startup and of every retry, and its failures are *amplified*: a resolver that flaps for 30 seconds browns out every service that calls it during that window; a negative-cache TTL set wrong turns a 30-second blip into a 30-minute outage; a conntrack race that drops one UDP packet in a hundred injects a hard 5-second stall into a random 1% of all lookups. At the synchronized start of a 512-GPU training job, that same 5-second stall on the slowest rank's peer lookup delays the entire collective. Staff-level fluency here is knowing *why* these amplify and naming the caching-topology fix, not reciting record types.

## What's new here (calibration)

- You already know A/AAAA/CNAME/SRV/PTR record types, recursion vs iteration, TTL and CNAME chains, `dig` basics, and that Kubernetes ships CoreDNS serving records for Services and pods — none of that is re-taught here.
- New: which client resolver (glibc/musl/Go) you're actually running against, because the same conntrack race manifests differently depending on that answer.
- New: DNS scaling and failure as a shared-dependency, multi-tenant blast-radius problem — CoreDNS is infrastructure, not a library call.
- New: the precise asymmetry between how SERVFAIL, NXDOMAIN, and a dropped/slow non-response are each cached and retried differently, and why that asymmetry is what turns a blip into a storm.
- New: the ndots tax expressed as a formula you can use for capacity planning, not just "it's slow, add a trailing dot."

## Core concepts

**DNS as a failure amplifier.** Because it's in the request path of everything, DNS failures don't stay local:

- A slow or flapping resolver → **fleet-wide brownout** during the flap; every caller's connection setup blocks on it.
- **Negative caching decides outage duration.** The SOA minimum / negative-cache TTL and how clients treat **SERVFAIL** (retry vs cache-the-failure) is the difference between a 30-second and a 30-minute recovery. Per RFC 2308, negative responses (NXDOMAIN and "no data") are cacheable against the zone's SOA minimum — but most resolver implementations deliberately do **not** cache SERVFAIL the same way, precisely so a transient upstream failure doesn't get pinned as authoritative for the TTL. The flip side: a flapping-but-not-fully-down upstream gets hammered by every retry, every time, because nothing is caching the failure. RFC 8020 tightens this further for NXDOMAIN specifically: an NXDOMAIN at a given name means the whole subtree beneath it is also nonexistent, which resolvers can use to shortcut further queries — but this only applies to true NXDOMAIN, not SERVFAIL, which is exactly why conflating the two is a real production bug, not a pedantic distinction.
- The **circular bootstrap**: DNS resolution depends on a service (the resolver, or the control plane it reads from) that itself depends on DNS to come up → an un-bootable cluster after a cold start. Break the cycle with static hosts entries or a resolver that can bootstrap from IPs.

**ndots:5 — the Kubernetes latency tax, generalized.** The injected `/etc/resolv.conf` uses `ndots:5` plus a search list of length *N* (typically `<ns>.svc.cluster.local`, `svc.cluster.local`, `cluster.local`, plus whatever the cloud/VPC injects). The rule: any name with **fewer than 5 dots** is treated as relative and walked through the search list *first*. Generalized formula: for a search list of length **N**, a relative name issues up to **(N+1) × 2** queries before success or final failure — N attempts against each search-list suffix, plus one absolute attempt, each doubled for parallel A/AAAA. With Kubernetes' typical 3-entry search list, that's up to `(3+1) × 2 = 8` queries for a single external hostname resolution in the worst case; a heavier VPC-injected search list of 5 entries pushes it to 12. Fixes: append a **trailing dot** to make the name fully-qualified (skips the search list entirely, dropping straight to 2 queries), lower `ndots` via per-pod `dnsConfig`, or add **NodeLocal DNSCache** / **CoreDNS `autopath`** so the wasted queries are absorbed locally or server-side instead of crossing the node boundary each time.

**The racy-conntrack 5-second timeout** (the canonical staff "intermittent latency" story). The mechanism, precisely: glibc's default resolver behavior sends the A and AAAA queries **in parallel over a single UDP socket** (same source port, `send-request`-style). On a busy node, the two replies race through netfilter's conntrack, and a known **DNAT/SNAT insert race** — both queries opening a conntrack entry near-simultaneously — can cause one of the two UDP packets to be dropped by the kernel because the second query, hashing to the same conntrack slot, evicts or collides with the first before the reply lands. The resolver then waits for its **5-second retransmit timeout** before retrying — so roughly 1 in 50–100 lookups on a busy node eats a flat 5.0-second penalty. Signature: a latency histogram with a sharp spike exactly at 5.0 s, not a smear. This is DNS resolving correctly *both times* — the bug is a dropped UDP packet plus a fixed OS retry timer, not a resolver fault, which is why "just restart the resolver" never fixes it.

**The fix is per-resolver-implementation, and conflating them is a real mistake:**

- **`single-request`** — serializes the A query then the AAAA query on the *same* socket. Avoids the parallel-send race entirely, at the cost of one extra RTT (sequential instead of concurrent).
- **`single-request-reopen`** — keeps the parallel send, but opens a **new socket** for the second query the moment a problem is detected (rather than always serializing). Different tradeoff: avoids the fixed extra-RTT tax of `single-request` in the common case, but only reacts after detecting trouble.
- **NodeLocal DNSCache** — the structurally different fix: it puts a caching agent on a link-local IP on the node itself, talking to CoreDNS over **TCP** for cache misses. This removes node-local conntrack NAT from the DNS hot path entirely for cache hits. Important nuance: it does **not** eliminate the underlying race in general — if the NodeLocal-to-CoreDNS hop still traverses UDP+NAT somewhere (e.g., cache miss forwarding), the same conntrack race can recur on that hop. The actual fix mechanism is using **TCP** for that upstream hop, not merely "being node-local."
- **Cilium eBPF dataplane** — bypasses conntrack for this path via eBPF-based service routing instead of iptables NAT, sidestepping the race at the mechanism level rather than papering over it.

**Client resolver differences matter — name the runtime before reciting the fix.** The "parallel A/AAAA over one socket" behavior is glibc's default (musl and Go do something different), and Alpine-based images (musl libc) are a classic offender for a related-but-distinct reason: musl's resolver has historically had weaker/absent search-list and retry tuning compared to glibc, so the same symptom can have a different root fix on Alpine than on a Debian/glibc base image. Go's `net` package ships its **own** resolver (pure-Go by default, not glibc's) with separate serialization knobs (`GODEBUG=netdns=go` vs `cgo`, and its own internal timeout/retry behavior) — a Go binary's DNS behavior on the same host can differ from a glibc C binary's, because it isn't going through `/etc/resolv.conf` parsing and `getaddrinfo()` the same way. Staff debugging discipline: identify glibc vs musl vs Go's resolver on the affected pod *before* reaching for a fix, because the three don't share a remediation.

**CoreDNS internals and the shared-dependency framing.** It's a **plugin chain**, and ordering matters: typically `kubernetes` (answers cluster names from the API) → `cache` → `forward` (upstream), with `autopath` optionally collapsing the search-domain walk server-side by watching pods to know their namespace (trading memory for eliminating the client-side search-list round-trips entirely). Know: positive vs negative cache TTLs are tunable separately; scale CoreDNS with an **HPA on QPS** and never run a single replica — treat it exactly like you'd treat any other shared stateful dependency (a database, a cache tier): a single-replica CoreDNS or an HPA that lags a pod-churn spike is a fleet-wide SPOF, not a per-tenant inconvenience, because *every* pod on *every* node routes through it. **Forward health-checking** matters because a silently-bad upstream in the forward list poisons every recursive answer downstream of it.

**Service discovery patterns beyond DNS.** Client-side discovery (Consul/etcd **watch** — clients get pushed updates, react in ms) vs server-side (DNS/VIP — clients poll and cache). Push vs poll is the axis. **DNS TTL caching is a poor fit for fast failover**: during a scale-down, clients holding cached A records keep dialing endpoints that are already gone → connection resets until the TTL expires. That's why fast-failover systems use watch-based discovery or a VIP/L4LB that removes the backend synchronously, not DNS TTLs.

**GPU tie-in.** Multi-node training resolves peer hostnames at job start — rank0's address, **headless Service** endpoints, `torchrun`/`c10d` rendezvous. A `StatefulSet` + headless Service gives stable per-pod DNS names that are the discovery substrate for ranks. But all ranks resolve at the *same synchronized instant*, so an ndots storm or a 5-second conntrack stall on any single rank's lookup delays the whole collective — the barrier waits for the slowest DNS query in the fleet. A headless Service's DNS answer is not free or instant just because it skips a VIP: it still transits the same CoreDNS path and is subject to the same ndots/conntrack mechanics as any other lookup; "headless" only changes what the *answer contains* (pod IPs directly instead of a cluster IP), not how it's resolved. NodeLocal DNSCache and headless-Service stable names are the standard hardening. (The collective/fabric mechanics themselves are module 09.)

## Perspectives

**Client-library.** glibc, musl, and Go's resolver are three different implementations with three different failure signatures under the same conntrack race. glibc's parallel-A/AAAA-on-one-socket behavior is the textbook trigger; musl (Alpine) inherits related but distinct weaknesses in search-list and retry handling; Go bypasses `getaddrinfo()` with its own pure-Go resolver and its own `GODEBUG=netdns=...` knobs. A staff engineer names which runtime is on the affected pod before reaching for `single-request-reopen` or any other glibc-specific fix, because applying a glibc remediation to a musl or Go workload does nothing.

**Multi-tenant blast-radius.** CoreDNS is not "a pod that answers DNS," it's a shared dependency on the same tier as etcd or a shared cache — every workload in the cluster routes through it, so its capacity and health should be reasoned about the way you'd reason about any other single-point-of-failure shared service: HPA lag during a pod-churn spike, or a single replica, is a fleet-wide brownout risk, not a namespace-local one.

**Cache-topology.** The asymmetry between how SERVFAIL, NXDOMAIN, and a slow/dropped non-response are cached and retried is the crux of whether a bad upstream blip stays a blip or becomes a storm. NXDOMAIN is cacheable and RFC 8020 lets a resolver shortcut an entire nonexistent subtree from one answer; SERVFAIL is typically *not* cached, so a flapping upstream gets re-hit on every single client retry with no dampening; a dropped UDP reply with no response at all triggers the fixed 5-second OS-level retransmit wait per query, which multiplies badly under ndots. Getting this taxonomy right is what separates "cache more aggressively" (right fix for NXDOMAIN storms) from "that won't help" (for SERVFAIL storms, where the fix is upstream health-checking and forwarder failover, not caching).

**Economics / scale.** The ndots tax is a QPS multiplier, not a fixed cost — 4-6x (or the full `(N+1)×2` formula in the worst case) against CoreDNS and upstream resolvers, for every relative-name lookup a workload makes. At high pod-churn rates (frequent rollouts, autoscaling, GPU job scheduling with many short-lived pods) this is a genuine capacity-planning line item: the QPS CoreDNS must sustain isn't "number of unique names resolved," it's that number times the ndots multiplier times churn rate, and sizing the HPA/replica count without that multiplier baked in is how a fleet gets a DNS-shaped outage during a routine scale event.

## Real-world use cases

- **Datadog — "It's always DNS . . . except when it's not: A deep dive through gRPC, Kubernetes, and AWS networking"** (https://www.datadoghq.com/blog/engineering/grpc-dns-and-load-balancing-incident/): a real incident where gRPC clients reconnecting roughly every 300ms across ~900 clients saturated the **AWS VPC-level conntrack table** — not a node-local resource — and the symptom looked exactly like a DNS problem but was actually reverse-path-filtering drops on deleted-pod IPs. A staff-level case study in not trusting the obvious hypothesis and instead following the actual mechanism.
- **Weave Works — "Racy conntrack and DNS lookup timeouts"** (https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts): the canonical primary source documenting the exact conntrack DNAT/SNAT insert race behind the flat 5.0-second DNS stall — the mechanism this lesson's worked example reproduces.

## Worked example

Reproduce the 5-second timeout and prove the fix.

1. **Repro:** from a pod on a busy node, loop `for i in $(seq 1000); do /usr/bin/time -f '%e' dig +short some.svc.cluster.local >/dev/null; done` and histogram the durations. You'll see a dense cluster near the true RTT plus a distinct spike at **5.0 s** — that's the dropped-UDP retransmit wait, not slow resolution. Note the libc in the pod's base image (glibc vs musl) before you interpret the result.
2. **Confirm the mechanism:** `conntrack -S` for insert failures / `nf_conntrack` drop counters climbing in lockstep with the 5 s events; the parallel A+AAAA on one socket is the trigger (glibc default; confirm this is actually the resolver in play).
3. **Fix:** deploy **NodeLocal DNSCache** (a per-node caching agent bound to a link-local IP, talking to CoreDNS over TCP for the upstream hop). Rerun the loop — the 5.0 s tail collapses because the hot path no longer traverses conntrack NAT for cache hits, and the upstream hop uses TCP instead of UDP+NAT.
4. **ndots calc (parallel deliverable):** with `ndots:5` and a 3-entry Kubernetes search list (N=3), count queries for `s3.amazonaws.com` (relative, <5 dots) using the formula `(N+1) × 2`: `(3+1) × 2 = 8` queries in the worst case before the absolute lookup succeeds. For `s3.amazonaws.com.` (trailing dot → absolute): just 2 (A+AAAA). Attach both counts, plus the same calculation for a 5-entry VPC-injected search list (`(5+1) × 2 = 12`), as the deliverable.

## Practice

<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

Add the "intermittent 5s DNS latency" runbook entry: symptom (sharp 5.0 s spike in a lookup-latency histogram), root cause (parallel A+AAAA over one socket + conntrack DNAT insert race dropping one UDP reply), confirmation (`conntrack -S` insert_failed climbing, and identifying glibc vs musl vs Go as the resolver in play), and the fix ladder (NodeLocal DNSCache with TCP upstream → `single-request-reopen`/`single-request` → Cilium eBPF bypass). Also add the "external lookups slow / NXDOMAIN storm" entry keyed on `ndots:5` with the trailing-dot and `dnsConfig` fixes, including the `(N+1)×2` query-count arithmetic for FQDN vs relative names. Add a third entry for "SERVFAIL storm" distinct from the NXDOMAIN entry, keyed on the negative-caching asymmetry (RFC 2308/8020) and forwarder health-checking as the fix, not caching.

## Common pitfalls

- **"The 5s stall means DNS is down."** DNS resolved correctly both times — one dropped UDP reply plus a fixed OS retransmit timer caused the wait, not a resolver failure or an unreachable server. Restarting CoreDNS won't fix a conntrack race.
- **"Lower ndots to 1 everywhere and you're safe."** This breaks legitimately-relative in-cluster short names (`myservice` instead of the FQDN) unless every calling app is updated to always use FQDNs. It's a per-workload tradeoff, not a global safe default.
- **"NodeLocal DNSCache eliminates the 5s bug entirely."** It removes node-local conntrack NAT from the hot path for cache hits, but the same race can still occur on the NodeLocal-to-CoreDNS hop if that hop still uses UDP+NAT. The actual fix mechanism is that hop using TCP, not merely being "node-local."
- **"SERVFAIL is cached like NXDOMAIN."** Most resolver implementations deliberately do *not* cache SERVFAIL the way they cache NXDOMAIN under the SOA-minimum rule — a flapping upstream gets hammered on every single retry with no dampening, which is a materially different incident shape than an NXDOMAIN storm.
- **"A headless Service's DNS is instant and free."** It still goes through the exact same CoreDNS path and is subject to the same ndots/conntrack mechanics as any other lookup — "headless" only changes what the answer contains (pod IPs vs a cluster VIP), not the resolution mechanics or cost.

## Self-check

- A pod's latency histogram for outbound calls shows a clean bimodal shape: most requests at ~2 ms, a distinct ~1% spike at exactly 5.0 s. DNS or application? What's the mechanism? **Answer:** DNS, and specifically the racy-conntrack timeout. The flat 5.0 s (not a smear) is the resolver's UDP retransmit timeout firing because one of the parallel A/AAAA queries — sent over a single socket, the glibc default — had its reply dropped by a conntrack DNAT/SNAT insert race on the busy node. Confirm with `conntrack -S` insert_failed counts and confirm glibc is actually the resolver in play; fix with NodeLocal DNSCache using TCP for the upstream hop.

- Resolving `api.internal.example.com` from inside a pod is issuing 8+ queries per lookup, most returning NXDOMAIN. Why, and what's the one-character fix? **Answer:** `ndots:5`. The name has 3 dots, which is fewer than 5, so the resolver treats it as relative and walks the full search list — for a 3-entry search list that's `(3+1) × 2 = 8` queries (search-list attempts plus the absolute attempt, doubled for A+AAAA) — before succeeding or exhausting. One-character fix: a trailing dot (`api.internal.example.com.`) makes it fully-qualified so the search list is skipped entirely, dropping to 2 queries; systemic fix is lowering `ndots` via per-pod `dnsConfig` or enabling CoreDNS `autopath`.

- Why is DNS TTL-based discovery a bad primitive for fast failover during a scale-down, and what do you use instead? **Answer:** Clients cache A records for the TTL, so after a backend is removed they keep dialing the dead endpoint until the cache expires — a window of connection resets and errors proportional to the TTL, and you can't safely set TTL to near-zero without a query storm. Fast failover needs synchronous endpoint removal: watch-based client-side discovery (etcd/Consul push), or a VIP/L4 load balancer that drops the backend from rotation immediately, rather than waiting on DNS TTL expiry.

- Your upstream resolver starts intermittently returning SERVFAIL. Ten minutes later, request volume against that upstream has *increased*, not decreased, even though nothing else changed. Why, and what's the fix? **Answer:** SERVFAIL isn't cached the way NXDOMAIN is (per RFC 2308's negative-caching rules, most implementations exempt SERVFAIL), so every client that would otherwise be served from cache instead re-queries the flapping upstream on every attempt — and if clients themselves retry on SERVFAIL, that's an additional multiplier. The fix isn't more caching; it's forwarder health-checking in CoreDNS's `forward` plugin to route around the bad upstream, and/or client-side retry backoff to stop hammering it.

- A DNS-shaped incident turns out to actually be a VPC-level conntrack table exhaustion. What made it look like DNS, and what was it really? **Answer:** This mirrors the Datadog gRPC incident: clients reconnecting very frequently (roughly every 300ms across hundreds of clients) exhausted the AWS VPC-level conntrack table — a shared resource above the node level — and the resulting drops were on reverse-path-filtering for stale/deleted pod IPs, which manifested as connection stalls indistinguishable from a DNS resolution problem at first glance. The lesson: verify the actual failing hop (conntrack table state, RPF drops) rather than assuming DNS just because the symptom is a connection-establishment stall.

## Connections & what's next

This lesson connects directly back to lesson 01's backpressure framing — the conntrack race is a UDP-specific instance of a kernel-datapath drop with its own counter (`conntrack -S`), the same discipline of "name the exact stage" applies. It also sets up lesson 03: DNS's poll-based, TTL-cached discovery model is one half of the push-vs-poll axis, and load balancers are where you'll see the other half (active health checks, connection draining) solve the same "how does a client learn a backend is gone" problem with a different mechanism and different cost profile. Lesson 05 (Kubernetes networking) will return to CoreDNS as one more workload behind a Service VIP, subject to the same iptables/IPVS/eBPF dataplane mechanics covered there.

## References & further reading

**Primary sources**
- RFC 2308 — Negative Caching of DNS Queries: https://www.rfc-editor.org/rfc/rfc2308
- RFC 8020 — NXDOMAIN: There Really Is Nothing Underneath: https://www.rfc-editor.org/rfc/rfc8020
- Kubernetes, DNS for Services and Pods: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- Kubernetes, NodeLocal DNSCache: https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/
- CoreDNS plugins reference: https://coredns.io/plugins/

**Real-world engineering blogs**
- Datadog, "It's always DNS . . . except when it's not": https://www.datadoghq.com/blog/engineering/grpc-dns-and-load-balancing-incident/
- Weave Works, "Racy conntrack and DNS lookup timeouts": https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts

**Deeper dives**
- Kubernetes issue #56903 (the original glibc/ndots/conntrack race discussion thread): https://github.com/kubernetes/kubernetes/issues/56903
