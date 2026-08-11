---
lesson: "A02.5"
title: "Kubernetes networking"
module: "A-02"
concept: "service dataplanes & conntrack"
status: not-started
est_time: "4.5 hrs"
prev: "04-cloud-networking.md"
next: "06-service-mesh.md"
artifacts: ["packet-path-teardown", "mtu-drop-repro", "dataplane-migration-checklist"]
sources: 8
---

# A02.5 · Kubernetes networking

> **Concept.** A Service VIP has no listener — it is resolved by one of three dataplanes (iptables, IPVS, eBPF), and conntrack is the villain in most fleet-scale failures.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits
Lesson 04 put a price and a latency number on every hop a packet takes across a cloud boundary. This lesson follows that packet the rest of the way down: once it lands inside the cluster and hits a Service VIP, *something* has to translate that virtual address into a real pod IP, and which mechanism does that translation determines your reprogramming latency, your conntrack exposure, and your NetworkPolicy security boundary. It is also the substrate lesson 06 sits on top of (the mesh is one more hop *after* this resolution) and the substrate lesson 07 deliberately routes around (RDMA bypasses all of it).

## Why this matters
At single-cluster scale, "Services just work" and kube-proxy is invisible. At fleet scale — thousands of Services, high connection churn, GPU nodes carrying control-plane plus NCCL-fallback traffic — the *dataplane implementation* becomes the dominant variable in latency, reprogramming lag, and outages. Staff-level fluency means naming exactly what translates a VIP to a backend, where SNAT hides source IPs, and why conntrack is the recurring failure surface. This is also the substrate lesson 07 builds on: RDMA is deliberately kept *off* this path, and knowing why requires knowing precisely what this path costs.

## What's new here (calibration)
- You already know the pod-per-IP model, Service types, that a CNI plugs pods into the network, and that NetworkPolicy is default-allow-until-added — none of that is repeated here.
- New: the internal mechanics of the three dataplanes that resolve a VIP, and why moving between them is a live-fleet migration project, not a config flag.
- New: conntrack as the unifying failure mode across DNS timeouts, incident postmortems, and GPU-node capacity planning.
- New: the control-plane scaling problem (Endpoints vs EndpointSlices) that only bites at fleet scale, and the identity-vs-IP security distinction in NetworkPolicy enforcement.

## Core concepts

### A Service VIP is not a real endpoint
ClusterIP is a virtual IP with nothing listening on it. Something must intercept traffic destined for it and DNAT to a real pod IP. There are three dataplanes that do this, and "which one, and why" is the whole interview.

**1. iptables kube-proxy (legacy default).** kube-proxy watches the API server and programs netfilter rules. A packet to a ClusterIP hits a `KUBE-SERVICES` chain, jumps to a per-service chain, and backend selection is a *linear walk* of `statistic --mode random --probability` rules — one rule per endpoint. Consequences: rule set grows O(n) with Services × endpoints; a single endpoint change can force rewriting large chains, so *reprogramming latency* climbs into seconds at thousands of Services; every flow rides **conntrack** (netfilter's connection-tracking table) because DNAT requires tracking the reverse translation.

**2. IPVS.** kube-proxy in IPVS mode uses in-kernel hash tables — O(1) lookup instead of a linear chain — and real LB scheduler algorithms. Reprogramming scales far better than iptables. But it is **still conntrack-bound** — this is the single most-missed fact about IPVS: it is *not* stateless like eBPF socket-level DNAT, it just tracks state faster. It also carries its own quirks: sync-time for connection state, and graceful-termination edge cases where terminating endpoints keep receiving traffic. IPVS scheduler algorithms worth naming: `rr` (round robin), `lc` (least connection), `wlc` (weighted least connection), `sh` (source hashing), and `mh` (Maglev hashing) — `mh` connects directly back to lesson 03's Maglev material: the same consistent-hashing scheme that made Google's software LB resilient to backend churn is available as an in-kernel IPVS scheduler.

**3. eBPF / Cilium kube-proxy replacement (the modern answer).** For east-west traffic, Cilium does socket-level DNAT in a BPF program attached to `connect()` — the VIP is rewritten to a backend *before a packet ever exists on the wire*. VIP→backend is an eBPF hashmap lookup (O(1)), it bypasses the iptables chain and, for that traffic class, bypasses conntrack entirely, supports native DSR (Direct Server Return) and Maglev-consistent backend selection. This is the "what replaces kube-proxy and why" answer: not just faster, but *structurally* off the conntrack/iptables scaling wall. It has a real gotcha though: because translation happens inside `connect()`, application code that calls `getpeername()`/`getsockname()` on its own socket can observe the *backend* address rather than the VIP — code that logs or asserts on "who am I talking to" can silently start reporting pod IPs instead of the Service VIP after a migration to eBPF mode. That is a genuine bug class discovered late in kube-proxy-replacement rollouts.

### Dataplane choice is a migration project, not a flag
Moving a live fleet from iptables → IPVS → eBPF is a project with its own risk profile, the same way any dataplane swap is: it needs dual-stack testing (run old and new in parallel on a canary node pool), feature-parity checks (does the new dataplane enforce NetworkPolicy the same way? does it support `externalTrafficPolicy: Local`? DSR?), and an explicit rollback plan, because a mid-migration state where some nodes DNAT one way and others DNAT differently is exactly the kind of asymmetry that produces "it works from this pod but not that one" tickets. Treat "switch to Cilium kube-proxy-free" as a migration with a canary cohort and a kill switch, not a Helm value change.

### Observability parity across dataplanes
Each dataplane exposes a *different* debugging surface, and knowing which commands even apply is a prerequisite to debugging (this feeds directly into lesson 08's decision tree):

| Dataplane | Inspect the rules | Inspect live flows |
|---|---|---|
| iptables | `iptables-save \| grep <svc-ip>` | `conntrack -L \| grep <svc-ip>` |
| IPVS | `ipvsadm -Ln` | `conntrack -L` (still conntrack-bound) |
| eBPF (Cilium) | `cilium service list` | `cilium monitor -v` (no conntrack entry for east-west eBPF-translated flows) |

Running the iptables playbook against an eBPF cluster gets you an empty, misleading result — `conntrack -L` shows nothing because there *is* no conntrack entry, not because there's no traffic.

### conntrack is the recurring villain
Table exhaustion (`nf_conntrack_max`) drops new flows silently — no error, no log line, just packets that vanish. The classic **DNS 5-second timeout** is a conntrack *insert race*: parallel UDP lookups (A + AAAA) get the same source tuple, one insert loses the race and its reply is dropped, the resolver retries after its 5s timeout. High-connection-churn workloads (short-lived HTTP, per-request connections) blow the table up fastest. eBPF's own CT map scales and evicts differently, which is a large part of why GPU clusters move to Cilium as the primary CNI.

Table sizing is a real capacity-planning exercise, not a "just set it very high" tuning knob: every conntrack entry consumes kernel memory, so an oversized table on a memory-constrained node risks memory pressure instead of connection drops — you have traded one failure mode for another. **GPU-node conntrack sizing is a distinct problem from web-tier sizing**: a training node's control-plane and checkpoint-coordination traffic generates many short-lived connections, and NCCL-fallback-to-TCP behavior adds more, in a pattern that is invisible under steady state and only spikes exactly when a training job's control traffic spikes — meaning the table can be sized correctly for average load and still exhaust during the one event (a checkpoint storm, a rank restart) that matters most.

### The two packet paths, precisely
(a) **pod→pod**: handled by the CNI — either native routing (BGP-advertised pod CIDRs, no encapsulation) or an overlay (VXLAN/Geneve) that wraps each packet in a ~50-byte header. Overlay = simpler L3 underlay requirements, at an MTU and per-packet encap/decap CPU cost — **not** an inherent throughput penalty; "overlays are always slower" conflates a fixable overhead with a structural ceiling.

(b) **pod→Service**: VIP DNAT happens at *client egress* (iptables/IPVS on the source node, or in-socket for eBPF). **SNAT/masquerade** is applied when traffic leaves the pod network (e.g., to off-cluster, or across nodes for certain modes), which hides the source pod IP behind the node IP — but not always: `externalTrafficPolicy: Local` preserves the real client IP by only routing to node-local backends, at the cost of load imbalance across nodes.

### NetworkPolicy enforcement is CNI-dependent, and the model matters
Some CNIs no-op NetworkPolicy entirely (flannel historically) — a security-critical gotcha to verify, not assume. Where it is enforced, there are two distinct models: **IP-based** (translate label selectors to pod IPs, program rules against those IPs) — this races pod churn, because IPs get reassigned faster than rules propagate, creating a real window where traffic is briefly allowed or denied against the wrong pod; **identity-based** (Cilium assigns a numeric identity per label-set and enforces on identity, not IP) — robust to IP reuse because the policy decision never depends on a point-in-time IP-to-pod mapping. This is a genuine multi-tenancy security-boundary distinction, not a performance nuance: on a shared cluster, the IP-based window of incorrectness is an exploitable gap during high pod churn (autoscaling events, rolling deploys). Mesh authz (L7) and NetworkPolicy (L3/4) are defense-in-depth, not substitutes for one another (see lesson 06).

### MTU across overlays is a top silent failure
Overlay header shrinks usable payload (1500 − 50 VXLAN = 1450). A mismatch — pods think MTU is 1500 but the path only carries 1450 — produces *intermittent large-packet drops*: small requests and TCP handshakes succeed, large responses hang. It presents as "the app hangs on big payloads" and is misdiagnosed as an application bug for days.

### EndpointSlices: the control-plane scaling fix
The original Endpoints API puts **every** backend for a Service into a single object. At high endpoint counts (hundreds to thousands of pods behind one Service — common for a large stateless tier) that single object becomes large enough that every endpoint change forces the API server and every watching kube-proxy to process the whole object again, which is a direct driver of the reprogramming-latency problem described above. **EndpointSlices** shard backends into slices of roughly 100 endpoints each, specifically so a single pod churn event only touches one small slice instead of one giant object — this is a control-plane scaling fix, independent of which dataplane (iptables/IPVS/eBPF) consumes the slices downstream.

### GPU-fleet tie
This is the foundation lesson 07 builds on: the *primary* pod network (this CNI) carries control-plane, storage, and NCCL-fallback traffic, while RDMA rides a *secondary* network attached via Multus that bypasses all of the above. conntrack limits and overlay MTU cost are precisely *why* RDMA is kept off the primary CNI, and Cilium's eBPF dataplane is increasingly the primary CNI on GPU clusters to avoid the iptables/conntrack scaling walls. (Module 09's lesson on Kubernetes multi-NIC for RDMA covers the Multus/SR-IOV/device-plugin composition itself — this lesson stays on the ordinary Service dataplane that the *rest* of the traffic uses.)

## Perspectives

**Dataplane-migration.** Choosing iptables vs IPVS vs eBPF is a live-fleet migration project with its own risk profile: dual-stack canary testing, feature-parity verification (NetworkPolicy enforcement, DSR support, `externalTrafficPolicy: Local` behavior), and a rollback plan — treat it like any other dataplane swap, because a fleet straddling two DNAT mechanisms mid-migration is a genuine asymmetric-failure surface.

**Observability-parity.** The commands that answer "why is this Service unreachable" are dataplane-specific: `iptables-save`/`conntrack -L` for legacy, `ipvsadm -Ln` for IPVS, `cilium service list`/`cilium monitor` for eBPF. Know which dataplane you're on *before* you start debugging — running the wrong playbook produces a confidently wrong answer, not an error.

**Multi-tenancy/identity.** Identity-based NetworkPolicy enforcement (Cilium) closes a real security window that IP-based enforcement (iptables-legacy) leaves open during pod churn. On a multi-tenant cluster this is a security-boundary decision, not a performance footnote — the IP-based model's window of incorrectness is exactly the kind of gap that shows up in a security review.

**GPU-fleet.** conntrack table sizing on GPU nodes is a different capacity-planning problem than on web-tier nodes: many short-lived control-plane and checkpoint-coordination connections, plus NCCL-fallback TCP, can exhaust the table in a pattern invisible at steady state and visible only when a training job's control traffic spikes — size for the spike, not the average.

## Real-world use cases

**Mark Betz — "Exhausting conntrack table space crippled our k8s cluster."** https://medium.com/@betz.mark/exhausting-conntrack-table-space-crippled-our-k8s-cluster-98564f6f34e0 — a widely-cited production incident where `nf_conntrack_max` exhaustion silently dropped new connections. What it shows: conntrack exhaustion produces no clean error signal, just connections that mysteriously fail to establish — exactly the "invisible until it isn't" failure mode this lesson's Core concepts warn about.

**Datadog — "It's always DNS . . . except when it's not."** https://www.datadoghq.com/blog/engineering/grpc-dns-and-load-balancing-incident/ — the root-cause chain runs through Cilium's pod-deletion IP-rule removal, VPC-level conntrack saturation, and reverse-path-filtering drops, touching CNI internals, conntrack, and cloud VPC networking in a single incident. What it shows: real fleet-scale incidents rarely stay inside one layer of this module — a debugging session that starts "it's DNS" can end at eBPF rule lifecycle and cloud-provider conntrack limits.

**Cloudflare — "Lost in transit: debugging dropped packets from negative header lengths."** https://blog.cloudflare.com/lost-in-transit-debugging-dropped-packets-from-negative-header-lengths/ — an IPVS-native-encapsulation dataplane bug found using `pwru` (packet-walk-record-you), a kernel-level packet-tracing tool. What it shows: IPVS has its own dataplane-internals bug surface distinct from iptables and eBPF, and `pwru`-style tracing (previewed here, covered fully in lesson 08) is how you find bugs at that depth.

## Worked example
Trace a `curl` to a ClusterIP three ways on the same cluster.

1. **iptables path.** `iptables-save | grep <svc-ip>` shows the `KUBE-SVC-*`/`KUBE-SEP-*` DNAT rules and the `statistic` random-probability backend selection. Fire the curl, then `conntrack -L | grep <svc-ip>` to see the tracked flow with its DNAT (original `dport` vs reply). This *is* the conntrack entry that fails to insert under the DNS race.

2. **IPVS path.** `ipvsadm -Ln` shows the virtual server and its scheduler (e.g., `rr` or `mh`) plus real-server weights. `conntrack -L` still shows an entry — the reminder that IPVS is fast, not stateless.

3. **eBPF path (Cilium kube-proxy-free).** `cilium service list` shows the VIP→backend map; `cilium monitor -v` during the curl shows the socket-level translation and — the key observation — **no new conntrack entry** for the east-west flow, because translation happened in `connect()`.

4. **MTU repro.** Compute 1500 − VXLAN(50) = 1450. Set a pod veth or overlay device MTU to 1500 while the path only carries 1450, request a large response (`curl` a multi-MB endpoint), and watch it hang while small requests succeed — then fix the MTU and watch it recover.

5. **EndpointSlice scale check.** `kubectl get endpointslices -l kubernetes.io/service-name=<svc>` against a Service with hundreds of backends — count the slices (~100 endpoints each) and compare against the single monolithic Endpoints object for the same Service, to see the sharding directly.

Deliverable: the five traces plus a one-paragraph "how I'd tell these apart in an incident."

## Practice
<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>
Produce a runbook page: for a reported "Service unreachable / intermittent" symptom, the decision tree that distinguishes (a) VIP-not-programmed / kube-proxy lag, (b) conntrack exhaustion or insert race, (c) NetworkPolicy drop (and whether it's an IP-based race or a genuine identity-based deny), (d) overlay MTU mismatch — with the exact command that confirms each and the fix. Put the "which dataplane am I on" check up front, since it changes every subsequent command. Separately, sketch a one-page **dataplane-migration checklist** (canary cohort, feature-parity checks, rollback trigger) as if proposing an iptables→eBPF migration to a fleet-ops review.

## Common pitfalls
- **"IPVS is stateless like eBPF DNAT."** It isn't — IPVS still tracks every flow in conntrack, it just does the lookup in O(1) instead of walking a chain. Only eBPF socket-level DNAT genuinely sidesteps conntrack for east-west traffic.
- **"NetworkPolicy is enforced the same way regardless of CNI."** Some CNIs silently no-op it. This is a security-critical assumption to verify on every cluster, not infer from the manifest applying successfully.
- **"SNAT/masquerade always hides the source IP."** `externalTrafficPolicy: Local` preserves the real client IP by routing only to node-local backends — the tradeoff is load imbalance across nodes, not a fixed loss of visibility.
- **"Overlay networks are always slower than native routing."** The real cost is MTU/encapsulation overhead and per-packet encap/decap CPU — not an inherent throughput ceiling. Size for the actual overhead, don't reject overlays on a vibe.
- **"conntrack table size should just be set very high."** Every entry costs kernel memory; an oversized table on a memory-constrained node trades connection drops for memory pressure. Size it from measured connection churn, not a round number.

## Self-check
- Why does a ClusterIP need no process listening on it, and what actually delivers the packet to a pod? **Answer:** The VIP is virtual — nothing binds it. kube-proxy (iptables/IPVS) or an eBPF program intercepts traffic to the VIP and DNATs it to a selected backend pod IP; iptables walks a linear random-probability chain, IPVS/eBPF use O(1) hash maps.
- What is the mechanism behind the classic 5-second DNS timeout in Kubernetes, and how does eBPF change it? **Answer:** A conntrack insert race — parallel UDP A/AAAA lookups share a source tuple, one insert loses and its reply is dropped, the resolver retries after 5s. eBPF's kube-proxy replacement does socket-level translation with its own CT map, sidestepping the netfilter insert-race path for east-west flows.
- Why is overlay MTU a "silent" failure and how does it present? **Answer:** The overlay header (~50 bytes for VXLAN) shrinks usable payload; if pod MTU exceeds what the path carries, only large packets drop while handshakes and small requests succeed — so it looks like an app hang on big responses, not a network fault.
- Why does IPVS scale better than iptables, and why is it still not "as good as" eBPF socket-level DNAT? **Answer:** IPVS replaces the linear iptables chain walk with an O(1) in-kernel hash lookup, so reprogramming and backend selection scale much better. But it is still conntrack-bound — every flow still gets a tracked entry — whereas eBPF's `connect()`-time DNAT avoids creating a conntrack entry at all for that traffic class.
- What problem do EndpointSlices solve that the original Endpoints API doesn't, and why does it matter at fleet scale? **Answer:** Endpoints puts every backend for a Service in one object, so any single endpoint change forces the API server and every watching kube-proxy to reprocess the whole object — a scaling wall at high endpoint counts. EndpointSlices shard backends into ~100-endpoint slices so churn only touches one small slice, cutting reprogramming latency at scale.
- Why is IP-based NetworkPolicy enforcement a security concern that identity-based enforcement closes? **Answer:** IP-based enforcement programs rules against pod IPs, which get reassigned faster than rules propagate during pod churn — creating a real window where the wrong pod is allowed or denied. Identity-based enforcement (Cilium) ties the policy decision to a numeric identity derived from labels, not a point-in-time IP, so it's robust to that churn.

## Connections & what's next
This lesson is the dataplane substrate under everything downstream in the module: lesson 03's Maglev material resurfaces as the IPVS `mh` scheduler; lesson 04's cost-per-hop lens continues into conntrack-table memory cost and eBPF CPU savings; lesson 07's RDMA path is defined by what it deliberately bypasses from this lesson; lesson 08's debug decision tree assumes you already know which dataplane-specific commands apply. Next: **06 · Service mesh** — what gets added, at what cost, once a proxy sits on top of the VIP resolution this lesson just built.

## References & further reading

**Primary sources**
- https://kubernetes.io/docs/reference/networking/virtual-ips/
- https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/

**Real-world engineering blogs**
- https://medium.com/@betz.mark/exhausting-conntrack-table-space-crippled-our-k8s-cluster-98564f6f34e0
- https://www.datadoghq.com/blog/engineering/grpc-dns-and-load-balancing-incident/
- https://blog.cloudflare.com/lost-in-transit-debugging-dropped-packets-from-negative-header-lengths/

**Deeper dives**
- https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
- https://docs.cilium.io/en/stable/network/concepts/masquerading/
