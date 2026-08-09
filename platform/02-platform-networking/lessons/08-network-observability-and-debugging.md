---
lesson: "A02.8"
title: "Network observability and debugging"
module: "A-02"
concept: "disciplined bisection + eBPF flow telemetry"
status: not-started
est_time: "3 hrs"
artifacts: ["network-debugging decision-tree runbook"]
---

# A02.8 · Network observability and debugging

> **Concept.** Staff network debugging is disciplined bisection down a decision tree, not tool-flailing — and at fleet scale the tree runs on identity-based eBPF flow telemetry, not IP-based packet sampling.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Why this matters
The senior-to-staff jump in network debugging is not more tools — it's a *procedure* that converges. Given "intermittent timeouts in prod," a staff engineer bisects: DNS? MTU? conntrack? LB/mesh? fabric? — each with a one-command test and a characteristic signature — instead of tcpdumping randomly. This is also the highest-signal interview payoff: the tree *is* the answer. And at fleet scale the old dataplane (sample-and-grep IPs) is broken because pod IPs churn faster than you can query them; the staff move is identity-based flow telemetry from the eBPF layer, where a flow is "service A → service B," not "10.0.3.7 → 10.0.9.2 (already gone)."

## Core notes
**Skip (you already know):** tcpdump/wireshark capture and filters; ping/traceroute/mtr; reading a cloud flow log; that eBPF can observe the kernel; basic "is the port open / is the process listening" triage.

**The debugging decision tree IS the deliverable.** Symptom shapes the first branch:

- **Is it DNS?** Signature: latency in exact **5-second multiples** → conntrack race on parallel UDP `A`/`AAAA` lookups from the same socket (the classic musl/Alpine + kube-dns DNAT insert race; UDP replies collide, one is dropped, the resolver waits the 5s timeout). `NXDOMAIN`-then-success or many failed lookups → **ndots:5** in the pod's `resolv.conf` walking the search list (`svc.cluster.local`, `cluster.local`, …) before the real name. Tests: `dig` the FQDN with a trailing dot vs bare; check `resolv.conf` ndots; watch for the 5s quantization.
- **Is it MTU?** Signature: small requests work, large payloads **hang** (not reset) — a PMTU black hole. An overlay/tunnel (VXLAN/IPsec/WireGuard) shrinks the usable MTU (~50 bytes for VXLAN) but a firewall drops the ICMP "fragmentation needed," so the sender never learns. Handshake and small GETs succeed; the first full-window transfer stalls. Test: ping with DF bit at descending sizes to find the real MTU; compare overlay MTU vs fabric MTU.
- **Is it conntrack?** Signature: random drops under load, `nf_conntrack: table full, dropping packet` in `dmesg`. Either the table is exhausted (`nf_conntrack_count` near `_max`) or DNAT/insert races under high new-flow churn. Test: read count/max, watch the dmesg line, check `insert_failed`/`drop` in `conntrack -S`.
- **Is it the LB/mesh?** Signature: retries, sudden latency cliffs, or partial availability tied to one service. Envoy/mesh mechanics: outlier detection **ejecting** a healthy-looking host, **panic mode** (too many hosts unhealthy → the LB ignores health and sends to all), sidecar startup races, retry amplification. Test: mesh/proxy stats (ejections, panic, 5xx by upstream), not the app logs.
- **Is it the fabric?** Signature: RDMA/collective slowness, no app-level errors. PFC pause storms, ECN marks, one flapping link setting the pace (callback to A02.7). Test: PFC pause counters, ECN marks, `NCCL_DEBUG=INFO` collective logs.

Teaching this tree — symptom → first command → likely cause — is the interview move and the runbook's spine.

**eBPF-based observability replaces sampling.** Cloud flow logs are sampled, delayed minutes, and keyed on IPs that no longer exist. The eBPF dataplane sees every packet at the kernel hook with **identity**:
- **Cilium Hubble** — L3-L7 flow visibility from the eBPF dataplane, no sidecar. `hubble observe --verdict DROPPED` shows not just that a packet dropped but *why* — e.g. `Policy denied` with the **source and destination identities** (service labels), so you find the offending NetworkPolicy in one command. Service maps, DNS- and HTTP-level visibility included.
- **Microsoft Retina** — cloud- and CNI-agnostic counterpart; works without Cilium and can feed Hubble's UI, so you get the same flow telemetry on EKS/AKS/GKE regardless of dataplane.
- Contrast: cloud flow logs are sampled, delayed, and IP-not-identity — useful for forensics, useless for live bisection.

**Packet capture at scale is a different discipline.** You cannot tcpdump a fleet. Targeted, eBPF-triggered capture instead:
- **`pwru`** (packet, where are you?) — traces a single packet's path through *every* kernel network hook (netfilter, tc, routing, XDP), so you see exactly which hook dropped it. The tool for "the packet leaves but never arrives."
- **BPF drop monitors** — `dropwatch`, `retis` — attach to `kfree_skb`/drop tracepoints and report drop location + reason kernel-wide without a full capture.
- Sampled/rolling captures for the cases you must see on the wire.
- **Offloads distort host captures (A02.1 callback):** GRO/TSO mean a host-side tcpdump shows coalesced 64KB "segments" that never existed on the wire; the NIC segments/reassembles below your capture point. Capture on the wire (SPAN/tap) or account for it when MTU/segmentation is the question.

**Latency/loss diagnosis — separate the three causes:**
- **Loss** vs **queueing (bufferbloat)** vs **app stall**. `ss -ti` on the socket gives `retrans`, `rtt`/`rttvar`, and `cwnd` — rising retrans = loss; high RTT with no retrans and a large cwnd = queueing/bufferbloat; healthy socket stats with slow progress = the app isn't reading/writing (stall).
- RTT histograms via `bpftrace`; per-hop attribution to localize.
- For RDMA/lossless, "the collective is slow" enters through PFC pause counters, ECN marks, and `NCCL_DEBUG=INFO` — same fabric branch as A02.7.

**Identity vs IP is the fleet-scale reframe.** In Kubernetes an IP is valid for the life of a pod — minutes. By the time a flow log surfaces `10.0.3.7 → 10.0.9.2`, both pods may be gone and reassigned. Staff-grade telemetry keys on stable identity: Cilium/Hubble security identities (label-derived), or mesh **SPIFFE** IDs. A flow reads "`checkout` → `payments`," survives churn, and is the only thing that lets you debug an intermittent issue spanning redeploys.

## Worked example
Build the runbook by breaking things, in order:
1. **Policy drop:** apply a NetworkPolicy that denies a real path; find it with `hubble observe --verdict DROPPED` — read the source/dest **identities** and the `Policy denied` reason, not the IPs.
2. **MTU regression:** set the overlay MTU wrong (or block ICMP frag-needed); reproduce the large-payload hang while small requests succeed; diagnose via DF-bit ping sweep, not the app.
3. **conntrack full:** drive new-flow churn until `nf_conntrack: table full` appears in dmesg; catch the drops in `conntrack -S`.
4. **`pwru`:** trace one of the dropped packets through the kernel hooks to name the exact drop point.

**Deliverable:** a one-page network-debugging **decision-tree runbook** — symptom → first command → likely cause — covering the DNS / MTU / conntrack / LB-mesh / fabric branches.

## Practice
Produce the decision-tree runbook and the four break-and-find exercises above. Then run the flagship scenario in [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md): **"the all-reduce got slower this week."** Force the full tree — a NUMA-misaligned reschedule (A02.7), a single link in PFC pause, an MTU regression on the RDMA net, or a noisy-neighbor softirq on the storage NIC (A02.1) — and show which branch each maps to, using Hubble/flow telemetry + fabric counters + an nccl-tests busbw regression as the combined evidence stack.

## Self-check
- A pod sees intermittent request latencies that are always almost exactly 5 seconds. What's your first hypothesis and why 5s specifically? **Answer:** A DNS conntrack race on parallel UDP A/AAAA lookups (classic musl + kube-dns DNAT insert race). The 5s is the resolver's UDP retry timeout — one of the two racing replies is dropped by a conntrack insert collision, so the client waits one full timeout before retrying. The quantization to a fixed multiple is the tell that it's a timeout, not variable load.
- Why does a host-side tcpdump mislead you when MTU/segmentation is the question, and what do you do instead? **Answer:** GRO/TSO offloads coalesce packets below the capture point, so tcpdump shows 64KB "segments" that never existed on the wire — you can't see real fragmentation or the true on-wire MTU. Capture on the wire (SPAN/tap), disable the offload for the test, or use a drop monitor / `pwru` that reads the actual kernel path.
- At fleet scale, why is a cloud flow log showing `10.0.3.7 → 10.0.9.2` nearly useless for debugging an intermittent cross-service issue, and what replaces it? **Answer:** Pod IPs live minutes; by the time the sampled, delayed flow log surfaces, both IPs are likely reassigned, so you can't map the flow to services or reproduce it. Replace it with identity-based eBPF telemetry — Hubble security identities or mesh SPIFFE IDs — where the flow is "serviceA → serviceB" and survives pod churn and redeploys.

## References
- https://docs.cilium.io/en/stable/observability/hubble/
- https://github.com/microsoft/retina
- https://github.com/cilium/pwru
- https://github.com/iovisor/bpftrace
