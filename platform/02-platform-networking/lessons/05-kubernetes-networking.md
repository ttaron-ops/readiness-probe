---
lesson: "A02.5"
title: "Kubernetes networking"
module: "A-02"
concept: "service dataplanes & conntrack"
status: not-started
est_time: "3 hrs"
artifacts: ["packet-path-teardown", "mtu-drop-repro"]
---

# A02.5 · Kubernetes networking

> **Concept.** A Service VIP has no listener — it is resolved by one of three dataplanes (iptables, IPVS, eBPF), and conntrack is the villain in most fleet-scale failures.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Why this matters
At single-cluster scale, "Services just work" and kube-proxy is invisible. At fleet scale — thousands of Services, high connection churn, GPU nodes carrying control-plane plus NCCL-fallback traffic — the *dataplane implementation* becomes the dominant variable in latency, reprogramming lag, and outages. Staff-level fluency means naming exactly what translates a VIP to a backend, where SNAT hides source IPs, and why conntrack is the recurring failure surface. This is also the substrate lesson 07 builds on: RDMA is deliberately kept *off* this path, and knowing why requires knowing precisely what this path costs.

## Core notes
**Skip (you already know):** pod-per-IP model, Service types (ClusterIP/NodePort/LB), that CNI plugs pods into the network, Ingress vs Service, NetworkPolicy is default-allow-until-added, basic `kubectl`/`exec` net debugging.

**A Service VIP is not a real endpoint.** ClusterIP is a virtual IP with nothing listening on it. Something must intercept traffic destined for it and DNAT to a real pod IP. There are three dataplanes that do this, and the "which one and why" is the whole interview:

**1. iptables kube-proxy (legacy default).** kube-proxy watches the API server and programs netfilter rules. A packet to a ClusterIP hits a `KUBE-SERVICES` chain, jumps to a per-service chain, and backend selection is a *linear walk* of `statistic --mode random --probability` rules — one rule per endpoint. Consequences: rule set grows O(n) with Services × endpoints; a single endpoint change can force re-writing large chains, so *reprogramming latency* climbs into seconds at thousands of Services; every flow rides **conntrack** (netfilter's connection tracking table) because DNAT requires tracking the reverse translation.

**2. IPVS.** kube-proxy in IPVS mode uses in-kernel hash tables — O(1) lookup instead of a linear chain — and real LB algorithms (rr, lc, sh, etc.). Reprogramming scales far better. But it is *still conntrack-based*, and it carries its own quirks: sync-time for connection state, graceful-termination edge cases where terminating endpoints keep receiving traffic.

**3. eBPF / Cilium kube-proxy replacement (the modern answer).** For east-west traffic, Cilium does socket-level DNAT in a BPF program attached to `connect()` — the VIP is rewritten to a backend *before a packet ever exists on the wire*. VIP→backend is an eBPF hashmap lookup (O(1)), it bypasses the iptables chain and much of conntrack, supports native DSR (Direct Server Return) and Maglev-style consistent backend selection. This is the "what replaces kube-proxy and why" answer: not just faster, but *structurally* off the conntrack/iptables scaling wall.

**conntrack is the recurring villain.** Table exhaustion (`nf_conntrack_max`) drops new flows silently. The classic **DNS 5s timeout** is a conntrack *insert race*: parallel UDP lookups (A + AAAA) get the same source tuple, one insert loses the race and its reply is dropped, the resolver retries after its 5s timeout. High-connection-churn workloads (short-lived HTTP, per-request connections) blow the table up. eBPF's own CT map scales and evicts differently, which is a large part of why GPU clusters move to Cilium as the *primary* CNI.

**The two packet paths, precisely.** (a) **pod→pod**: handled by the CNI — either native routing (BGP-advertised pod CIDRs, no encap) or an overlay (VXLAN/Geneve) that wraps each packet in a ~50-byte header. Overlay = simpler L3 underlay but MTU cost. (b) **pod→Service**: VIP DNAT happens at *client egress* (iptables/IPVS on the source node, or in-socket for eBPF). **SNAT/masquerade** is applied when traffic leaves the pod network (e.g. to off-cluster, or across nodes for certain modes), which *hides the source pod IP* behind the node IP. `externalTrafficPolicy: Local` preserves the real client IP by only routing to node-local backends — at the cost of load imbalance across nodes.

**NetworkPolicy is CNI-dependent.** Some CNIs no-op it entirely (flannel historically). Enforcement is via iptables or eBPF. Two models: **IP-based** (translate label selectors to pod IPs, program rules) — races pod churn because IPs are reassigned faster than rules propagate, briefly allowing/denying the wrong pod; **identity-based** (Cilium assigns a numeric identity per label-set, enforces on identity) — robust to IP reuse. Mesh authz (L7) and NetworkPolicy (L3/4) are defense-in-depth, not substitutes (see lesson 06).

**MTU across overlays is a top silent failure.** Overlay header shrinks usable payload (1500 − 50 VXLAN = 1450). A mismatch — pods think MTU is 1500 but the path only carries 1450 — produces *intermittent large-packet drops*: small requests and TCP handshakes succeed, large responses hang. It presents as "the app hangs on big payloads" and is misdiagnosed as an application bug for days.

**GPU-fleet tie.** This is the foundation lesson 07 builds on: the *primary* pod network (this CNI) carries control-plane, storage, and NCCL-fallback traffic, while RDMA rides a *secondary* network attached via Multus that bypasses all of the above. conntrack limits and overlay MTU cost are precisely *why* RDMA is kept off the primary CNI, and Cilium's eBPF dataplane is increasingly the primary CNI on GPU clusters to avoid the iptables/conntrack scaling walls.

## Worked example
Trace a `curl` to a ClusterIP three ways on the same cluster.

1. **iptables path.** `iptables-save | grep <svc-ip>` shows the `KUBE-SVC-*`/`KUBE-SEP-*` DNAT rules and the `statistic` random-probability backend selection. Fire the curl, then `conntrack -L | grep <svc-ip>` to see the tracked flow with its DNAT (original `dport` vs reply). This *is* the conntrack entry that fails to insert under the DNS race.

2. **eBPF path (Cilium kube-proxy-free).** `cilium service list` shows the VIP→backend map; `cilium monitor -v` during the curl shows the socket-level translation and — the key observation — **no new conntrack entry** for the east-west flow, because translation happened in `connect()`.

3. **MTU repro.** Compute 1500 − VXLAN(50) = 1450. Set a pod veth or overlay device MTU to 1500 while the path only carries 1450, request a large response (`curl` a multi-MB endpoint), and watch it hang while small requests succeed — then fix the MTU and watch it recover. Deliverable: the three traces plus a one-paragraph "how I'd tell these apart in an incident."

## Practice
<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>
Produce a runbook page: for a reported "Service unreachable / intermittent" symptom, the decision tree that distinguishes (a) VIP-not-programmed / kube-proxy lag, (b) conntrack exhaustion or insert race, (c) NetworkPolicy drop, (d) overlay MTU mismatch — with the exact command that confirms each and the fix. Put the eBPF-vs-iptables "which dataplane am I on" check up front, since it changes every subsequent command.

## Self-check
- Why does a ClusterIP need no process listening on it, and what actually delivers the packet to a pod? **Answer:** The VIP is virtual — nothing binds it. kube-proxy (iptables/IPVS) or an eBPF program intercepts traffic to the VIP and DNATs it to a selected backend pod IP; iptables walks a linear random-probability chain, IPVS/eBPF use O(1) hash maps.
- What is the mechanism behind the classic 5-second DNS timeout in Kubernetes, and how does eBPF change it? **Answer:** A conntrack insert race — parallel UDP A/AAAA lookups share a source tuple, one insert loses and its reply is dropped, the resolver retries after 5s. eBPF's kube-proxy replacement does socket-level translation with its own CT map, sidestepping the netfilter insert-race path.
- Why is overlay MTU a "silent" failure and how does it present? **Answer:** The overlay header (~50 bytes for VXLAN) shrinks usable payload; if pod MTU exceeds what the path carries, only large packets drop while handshakes and small requests succeed — so it looks like an app hang on big responses, not a network fault.

## References
- https://kubernetes.io/docs/reference/networking/virtual-ips/
- https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://docs.cilium.io/en/stable/network/concepts/masquerading/
