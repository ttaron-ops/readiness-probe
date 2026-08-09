---
lesson: "A02.2"
title: "DNS and service discovery"
module: "A-02"
concept: "DNS failure amplification"
status: not-started
est_time: "3 hrs"
artifacts: ["5s-timeout repro histogram", "ndots query-count calc", "NodeLocal DNSCache tail-latency before/after"]
---

# A02.2 · DNS and service discovery

> **Concept.** DNS sits in the request path of everything, so its failure modes are amplifiers — an ndots search-storm or a single dropped UDP query turns into fleet-wide tail latency, and the fix is almost always caching topology, not record content.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Why this matters

DNS is the most under-respected item in the request path. It resolves before every connection, it is on the critical path of service startup and of every retry, and its failures are *amplified*: a resolver that flaps for 30 seconds browns out every service that calls it during that window; a negative-cache TTL set wrong turns a 30-second blip into a 30-minute outage; a conntrack race that drops one UDP packet in a hundred injects a hard 5-second stall into a random 1% of all lookups. At the synchronized start of a 512-GPU training job, that same 5-second stall on the slowest rank's peer lookup delays the entire collective. Staff-level fluency here is knowing *why* these amplify and naming the caching-topology fix, not reciting record types.

## Core notes

**Skip (you already know):** A/AAAA/CNAME/SRV/PTR record types; recursion vs iteration; TTL and CNAME chains; `dig` basics; that Kubernetes ships CoreDNS serving A/SRV records for Services and pods.

**DNS as a failure amplifier.** Because it's in the request path of everything, DNS failures don't stay local:

- A slow or flapping resolver → **fleet-wide brownout** during the flap; every caller's connection setup blocks on it.
- **Negative caching decides outage duration.** The SOA minimum / negative-cache TTL and how clients treat **SERVFAIL** (retry vs cache-the-failure) is the difference between a 30-second and a 30-minute recovery. SERVFAIL is *not* cached the way NXDOMAIN is — get this wrong and a briefly-bad upstream poisons everything, or conversely you hammer a recovering server.
- The **circular bootstrap**: DNS resolution depends on a service (the resolver, or the control plane it reads from) that itself depends on DNS to come up → an un-bootable cluster after a cold start. Break the cycle with static hosts entries or a resolver that can bootstrap from IPs.

**ndots:5 — the Kubernetes latency tax.** The injected `/etc/resolv.conf` uses `ndots:5` plus a ~5-entry search list (`<ns>.svc.cluster.local`, `svc.cluster.local`, `cluster.local`, …). The rule: any name with **fewer than 5 dots** is treated as relative and walked through the search list *first*. So resolving `s3.amazonaws.com` (2 dots) fires the search-list permutations — `s3.amazonaws.com.<ns>.svc.cluster.local`, `.svc.cluster.local`, `.cluster.local`, … each returning NXDOMAIN — before finally trying the name as absolute. That's 4–6 wasted round-trips per external lookup, ×2 for parallel A+AAAA. Fixes: append a **trailing dot** to make the name fully-qualified (skips the search list entirely), lower `ndots` via per-pod `dnsConfig`, or add **NodeLocal DNSCache** so the wasted queries hit a local cache instead of crossing the node boundary.

**The racy-conntrack 5-second timeout** (the canonical staff "intermittent latency" story). glibc/musl send the A and AAAA queries **in parallel over a single UDP socket** (same source port). On a busy node, the two replies race through netfilter's conntrack, and a known **DNAT/SNAT insert race** can drop one of the two UDP packets. The resolver then waits for its **5-second retransmit timeout** before retrying — so ~1 in 50–100 lookups on a busy node eats a flat 5.0-second penalty. Signature: a latency histogram with a sharp spike exactly at 5.0 s. Mitigations, in order of preference: **NodeLocal DNSCache** (queries go to a local cache over the loopback / TCP, no conntrack NAT on the hot path), `single-request-reopen`/`single-request` resolver options (serialize or separate the sockets), or a dataplane that bypasses conntrack for this path entirely (**Cilium eBPF**).

**CoreDNS internals.** It's a **plugin chain**, and ordering matters: typically `kubernetes` (answers cluster names from the API) → `cache` → `forward` (upstream), with `autopath` optionally collapsing the search-domain walk server-side. Know: positive vs negative cache TTLs are tunable separately; **autopath** trades memory (it watches pods) to eliminate the client's search-list round-trips; scale CoreDNS with an **HPA on QPS** and never run a single replica (SPOF for the whole cluster); **forward health-checking** matters because a silently-bad upstream in the forward list poisons every recursive answer.

**Service discovery patterns beyond DNS.** Client-side discovery (Consul/etcd **watch** — clients get pushed updates, react in ms) vs server-side (DNS/VIP — clients poll and cache). Push vs poll is the axis. **DNS TTL caching is a poor fit for fast failover**: during a scale-down, clients holding cached A records keep dialing endpoints that are already gone → connection resets until the TTL expires. That's why fast-failover systems use watch-based discovery or a VIP/L4LB that removes the backend synchronously, not DNS TTLs.

**GPU tie-in.** Multi-node training resolves peer hostnames at job start — rank0's address, **headless Service** endpoints, `torchrun`/`c10d` rendezvous. A `StatefulSet` + headless Service gives stable per-pod DNS names that are the discovery substrate for ranks. But all ranks resolve at the *same synchronized instant*, so an ndots storm or a 5-second conntrack stall on any single rank's lookup delays the whole collective — the barrier waits for the slowest DNS query in the fleet. NodeLocal DNSCache and headless-Service stable names are the standard hardening. (The collective/fabric mechanics themselves are module 09.)

## Worked example

Reproduce the 5-second timeout and prove the fix.

1. **Repro:** from a pod on a busy node, loop `for i in $(seq 1000); do /usr/bin/time -f '%e' dig +short some.svc.cluster.local >/dev/null; done` and histogram the durations. You'll see a dense cluster near the true RTT plus a distinct spike at **5.0 s** — that's the dropped-UDP retransmit wait, not slow resolution.
2. **Confirm the mechanism:** `conntrack -S` for insert failures / `nf_conntrack` drop counters climbing in lockstep with the 5 s events; the parallel A+AAAA on one socket is the trigger.
3. **Fix:** deploy **NodeLocal DNSCache** (a per-node caching agent bound to a link-local IP, talking to CoreDNS over TCP). Rerun the loop — the 5.0 s tail collapses because the hot path no longer traverses conntrack NAT.
4. **ndots calc (parallel deliverable):** with `ndots:5` and a 5-domain search list, count queries for `s3.amazonaws.com` (relative, <5 dots): 5 search-domain attempts × 2 (A+AAAA) = up to ~10 queries, then the absolute attempt (2 more) — ~10–12 lookups. For `s3.amazonaws.com.` (trailing dot → absolute): just 2 (A+AAAA). Attach both counts as the deliverable.

## Practice

<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

Add the "intermittent 5s DNS latency" runbook entry: symptom (sharp 5.0 s spike in a lookup-latency histogram), root cause (parallel A+AAAA over one socket + conntrack DNAT insert race dropping one UDP reply), confirmation (`conntrack -S` insert_failed climbing), and the fix ladder (NodeLocal DNSCache → `single-request-reopen` → Cilium eBPF bypass). Also add the "external lookups slow / NXDOMAIN storm" entry keyed on `ndots:5` with the trailing-dot and `dnsConfig` fixes. Include the query-count arithmetic for FQDN vs relative names.

## Self-check

- A pod's latency histogram for outbound calls shows a clean bimodal shape: most requests at ~2 ms, a distinct ~1% spike at exactly 5.0 s. DNS or application? What's the mechanism? **Answer:** DNS, and specifically the racy-conntrack timeout. The flat 5.0 s (not a smear) is the resolver's UDP retransmit timeout firing because one of the parallel A/AAAA queries — sent over a single socket — had its reply dropped by a conntrack DNAT/SNAT insert race on the busy node. Confirm with `conntrack -S` insert_failed counts; fix with NodeLocal DNSCache (removes conntrack NAT from the DNS hot path).

- Resolving `api.internal.example.com` from inside a pod is issuing 8+ queries per lookup, most returning NXDOMAIN. Why, and what's the one-character fix? **Answer:** `ndots:5`. The name has 3 dots, which is fewer than 5, so the resolver treats it as relative and walks the full search list (`.<ns>.svc.cluster.local`, `.svc.cluster.local`, `.cluster.local`, …) — each ×2 for A+AAAA — all NXDOMAIN, before trying it absolute. One-character fix: a trailing dot (`api.internal.example.com.`) makes it fully-qualified so the search list is skipped; systemic fix is lowering `ndots` via per-pod `dnsConfig` or enabling CoreDNS autopath.

- Why is DNS TTL-based discovery a bad primitive for fast failover during a scale-down, and what do you use instead? **Answer:** Clients cache A records for the TTL, so after a backend is removed they keep dialing the dead endpoint until the cache expires — a window of connection resets and errors proportional to the TTL, and you can't safely set TTL to near-zero without a query storm. Fast failover needs synchronous endpoint removal: watch-based client-side discovery (etcd/Consul push), or a VIP/L4 load balancer that drops the backend from rotation immediately, rather than waiting on DNS TTL expiry.

## References

- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/
- https://coredns.io/plugins/
- https://github.com/kubernetes/kubernetes/issues/56903
- https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts
