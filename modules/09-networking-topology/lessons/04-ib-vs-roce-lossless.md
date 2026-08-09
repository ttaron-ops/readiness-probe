---
lesson: "09.4"
title: "InfiniBand vs RoCEv2: engineering losslessness, and when each wins"
module: "09"
concept: "InfiniBand vs RoCEv2: engineering losslessness, and when each wins"
status: not-started
est_time: "6h"
artifacts: []
---
# 09.4 · InfiniBand vs RoCEv2: engineering losslessness, and when each wins

> **Concept.** InfiniBand is lossless *by construction* (credit-based hardware flow control); RoCEv2 is RDMA bolted onto Ethernet that you must *engineer* into losslessness with PFC + ECN + DCQCN — and the senior skill is naming which trade wins for a given cluster, not chanting "IB is faster."
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Why this matters

This is the highest-yield interview topic in the module because it is where a hiring panel at CoreWeave / NVIDIA / a neocloud finds out whether you *design fabrics* or just recite specs. The junior answer is "InfiniBand, it's lower latency." The answer that gets the senior offer is: *"Depends. IB buys you determinism and SHARP but locks you to NVIDIA and a thinner optics ecosystem; RoCEv2 reuses my Ethernet/EVPN-BGP muscle and a fat optics market but I pay for losslessness in PFC/ECN tuning risk — and at 24K GPUs Meta shipped RoCE and even turned DCQCN **off**. Here's when I'd pick each."* You need mechanism, failure modes, and a cost model.

The stakes are literal money and literal uptime. Get losslessness wrong on RoCE and you don't get "a bit slower" — you get **PFC deadlock** that wedges a fabric region, or **head-of-line blocking** that tanks collective efficiency and silently burns 20–30% of GPU-hours. Get vendor lock wrong on IB and you've signed a multi-year, single-source optics-and-switch bill on a fabric you can't second-source. Both failure modes have a dollar figure your FinOps instinct should be itching to attach.

## What's new here (contrast with 01b)

In 01b, "the network" was the Linux kernel deciding what to do with a packet — and crucially, **dropping** was normal and *fine*: TCP's whole design assumes loss and recovers from it with retransmits and a congestion window. RDMA fabrics invert that assumption. RDMA (especially RC over the classic go-back-N model) is **allergic to loss**: a single dropped packet triggers a coarse retransmit that collapses collective throughput. So the fabric itself must be **lossless** — packets must not be dropped due to congestion, ever. That is a property Ethernet was explicitly designed *not* to have.

So the new idea is: **there are two fundamentally different ways to get a lossless RDMA fabric.**

- **InfiniBand:** losslessness is *intrinsic*. A separate L2 protocol, purpose-built, with **credit-based flow control** in hardware — a sender may only transmit if the receiver has advertised buffer credits, so congestion means "wait for credits," never "drop." A centralized **Subnet Manager (SM)** computes routing and manages the fabric. There is no "engineering losslessness"; it's the substrate.
- **RoCEv2 (RDMA over Converged Ethernet v2):** take RDMA and run it as a **UDP/IP payload over ordinary routed Ethernet** (UDP dest port 4791; being IP-routable is the "v2" upgrade over RoCEv1's raw-L2). Ethernet drops under congestion, so you must **manufacture** losslessness with a stack of Ethernet features: **PFC** to stop drops, **ECN** to signal congestion, **DCQCN** to react to it. You are re-using the Ethernet control plane you already know (BGP/EVPN, ECMP) — and inheriting all of its failure modes.

The contrast to hold: **IB deletes the datapath *and* gives you a lossless substrate for free; RoCE deletes the datapath but hands you the substrate as a tuning project.**

## Core notes

### 1. InfiniBand mechanism (the "free lossless" path)

- **Credit-based link-level flow control.** Every link tracks receiver buffer credits; a sender transmits only against available credits. Congestion → backpressure via credit starvation → **zero congestion drops** by construction. No tuning knob to get wrong.
- **Subnet Manager (SM).** A centralized control plane (e.g., NVIDIA UFM, or `opensm`) discovers topology, assigns LIDs, computes forwarding tables, and can drive **adaptive routing**. One brain; deterministic behavior; also one thing to make HA.
- **SHARP (Scalable Hierarchical Aggregation and Reduction Protocol).** *In-network compute*: the switches themselves perform the reduction for all-reduce/reduce-scatter, so data is summed in the fabric instead of shuttled around a ring. This is a real, IB-only throughput multiplier for collectives (Quantum-2 = NDR 400G with SHARPv3; **Quantum-X800 = XDR 800G, 144×800G ports, SHARPv4 adding FP8 and ReduceScatter/ScatterGather**). When someone says "IB isn't just lower latency, it changes the collective's algorithmic cost" — SHARP is what they mean.
- **Latency/scale anchors:** ~100–130 ns switch hop, sub-µs end-to-end small message; deterministic tail. NICs: **ConnectX-7 (NDR/400G)**, **ConnectX-8 (XDR/800G)**, **BlueField-3** DPU when you want to offload/manage on the NIC.
- **The catch:** it's a **single-vendor** stack — NVIDIA/Mellanox switches, NICs, cables, SM, and a comparatively **narrow, premium optics ecosystem**. You cannot second-source it.

### 2. RoCEv2 mechanism (the "engineered lossless" path)

RoCEv2 is RDMA-in-UDP over routed Ethernet. To make routed Ethernet lossless enough for RDMA you stack three mechanisms — and you must understand each *separately*:

- **PFC (Priority Flow Control, 802.1Qbb).** Per-priority (per traffic class) PAUSE. When a switch's ingress buffer for the RDMA priority fills past a threshold, it sends a **PAUSE** frame **upstream** telling the previous hop to stop sending *that class*. This is the mechanism that turns "drop" into "backpressure" — RoCE's answer to IB credits. It is **coarse** (stops a whole priority, not a flow) and **hop-by-hop**, which is exactly why it's dangerous (see failure modes).
- **ECN (Explicit Congestion Notification).** An *end-to-end* congestion signal: a congested switch, instead of (or before) dropping, **marks** the IP ECN bits. The receiver sees the mark and reflects a **CNP (Congestion Notification Packet)** back to the sender. ECN says "you, specific flow, are causing congestion" — information PFC's blunt PAUSE cannot convey.
- **DCQCN (Data Center Quantized Congestion Notification).** The **congestion-control algorithm** that consumes ECN/CNP signals and **rate-limits the offending flow at the sender NIC**, then gently ramps back up. It's the RoCE analog of a congestion window, running in the NIC.

You must also **calibrate PFC headroom** (buffer reserved so in-flight packets during the PAUSE round-trip aren't dropped) and **ECN thresholds** (mark early enough that DCQCN reacts *before* PFC has to PAUSE). Getting these numbers right, per switch model and per link speed, **is** the RoCE tuning project.

### 3. Why DCQCN exists if you already have PFC (the classic interview trap)

Because **PFC alone is a disaster at scale.** PFC is coarse (pauses an entire priority class, not the one bad flow) and propagates **backward hop-by-hop**: a hot spot at one switch sends PAUSE upstream, that switch's buffers fill and it PAUSEs *its* upstream, and the backpressure **spreads like a traffic jam** — this is **congestion spreading / head-of-line blocking**, where innocent flows sharing the paused class are stalled by one victim link. Worse, cyclic PAUSE dependencies can **deadlock** the fabric. PFC is a blunt "stop *everything* on this class *now*" hammer with no notion of *who* is at fault.

DCQCN exists to **keep PFC from ever firing.** ECN+DCQCN is the *fine-grained, end-to-end, flow-specific* controller: it slows the *actual* offending flow at its source before buffers fill enough to trigger a PAUSE. The intended operating regime: **DCQCN does the everyday congestion control; PFC is the last-resort safety net that catches microbursts DCQCN was too slow for.** If your fabric is relying on PFC in steady state, your ECN/DCQCN tuning is wrong.

### 4. Failure modes you must be able to name

- **PFC deadlock.** Cyclic buffer dependency: switch A's RDMA-class buffer is full and PAUSEs B; B's fills and PAUSEs C; C's fills and PAUSEs A — a **PAUSE cycle** with no packet able to drain. The fabric region wedges. Root cause is topology + routing that allows a cyclic dependency among paused buffers (non-minimal or reordered paths make it worse). Mitigations: careful up/down or deadlock-free routing, watchdogs that drop the paused class after a timeout, and — the strategic move — **minimize reliance on PFC** via good DCQCN so PAUSE is rare.
- **Head-of-line blocking / congestion spreading.** As above: one congested egress PAUSEs a whole class upstream, stalling unrelated flows. Kills collective efficiency because a collective is a barrier — stalling *any* participant stalls the step.
- **Slow/silent misconfig.** Mismatched ECN thresholds vs PFC headroom, wrong DSCP-to-priority mapping across a vendor boundary, or a firmware change to DCQCN behavior (exactly what bit Meta) — none of these page you; they just quietly cost GPU-hours.

### 5. Spectrum-X — "managed RoCE"

NVIDIA's answer to "RoCE tuning is a research project": **Spectrum-X** = the **Spectrum-4 switch (SN5600)** + **BlueField-3 / ConnectX SuperNICs**, co-designed so the Ethernet fabric behaves like a lossless RDMA fabric out of the box — hardware **adaptive routing**, fast **telemetry-based congestion control**, and per-flow load balancing that plain ECMP+PFC can't do. The pitch: **Ethernet ecosystem, IB-like determinism, someone else did the tuning.** The cost: you're back toward a **single-vendor** stack (Spectrum-X's magic wants NVIDIA switches *and* NVIDIA SuperNICs), so it partially trades away the "reuse commodity Ethernet" benefit that made RoCE attractive. It's the middle box in the decision, not a free lunch.

### 6. The cost dimension (your FinOps differentiator)

- **Optics ecosystem:** Ethernet optics/DAC/AOC at 400G/800G are a **broad, multi-vendor, price-competitive** market; IB optics are **narrower and premium** and single-sourced. At 24K endpoints, cabling+optics is a serious line item — this is often where RoCE's TCO advantage actually lives, not in the switch ASIC.
- **Operational cost:** IB needs **IB-specific skills** (SM, fabric tooling) that are scarcer/pricier to hire; RoCE reuses your existing **Ethernet/EVPN-BGP** team and monitoring. But RoCE spends that saving back as **tuning/validation engineering** (PFC/ECN/DCQCN qualification per switch model).
- **Lock-in / second-sourcing:** IB = single vendor, weak negotiating position, roadmap risk. RoCE (non-Spectrum-X) = multi-vendor switches, real price leverage. Spectrum-X sits in between.
- **SHARP:** if your training is all-reduce-bound and you can exploit in-network reduction, IB's SHARP is a *performance* win that can change the TCO math back in IB's favor — it's not purely a cost line, it's a throughput multiplier.

## Worked example — the decision table + two verdicts

Build the table on the axes below, then pick a winner for **two** clusters with **different** answers.

| Axis | InfiniBand (Quantum-X800 / ConnectX-8) | RoCEv2 (commodity Eth) | Spectrum-X (managed RoCE) |
|---|---|---|---|
| Small-msg latency / determinism | Best; sub-µs, deterministic tail | Good if tuned; tail at risk from PFC | Near-IB; adaptive routing tames tail |
| NCCL bus bandwidth (bound) | Highest; line-rate + SHARP in-network reduce | ~90–95% of IB *when perfectly tuned* | High; co-designed to approach IB |
| Control plane | Subnet Manager (one brain; make it HA) | BGP/EVPN + PFC/ECN/DCQCN you own | NVIDIA-managed CC + adaptive routing |
| Tuning risk | Low (lossless by construction) | **High** (PFC deadlock, ECN/headroom, DCQCN) | Low–medium (vendor did the tuning) |
| Cost / optics | Premium, narrow, single-source optics | **Cheapest**, broad multi-vendor optics | Mid; NVIDIA switch+SuperNIC premium |
| Vendor lock-in | **High** (NVIDIA end-to-end) | **Low** (multi-vendor Ethernet) | Medium–high (NVIDIA switch+NIC) |
| SHARP / in-network compute | **Yes** (SHARPv4: FP8, ReduceScatter) | No | No (in-network compute is IB-only) |
| Skills reused | IB-specific (scarcer) | Existing Ethernet/EVPN-BGP team | Ethernet team + vendor support |

**Scenario A — 512-GPU research cluster, mixed/bursty workloads, small ops team, latency-sensitive, wants max out-of-box performance and minimal fabric-engineering headcount.**

**Verdict: InfiniBand.** At 512 GPUs the vendor-lock and premium-optics penalties are **small in absolute dollars** and easily dominated by engineer-time. This team cannot afford to run a PFC/ECN/DCQCN qualification project or chase a PFC-deadlock incident; IB's *lossless-by-construction* substrate means the fabric "just works" with a deterministic tail, and if the research is all-reduce-heavy, **SHARP** is a direct win. The thing RoCE saves — optics cost and reused Ethernet skills — is exactly what this small, latency-sensitive team values *least*. Buy determinism with money, not headcount.

**Scenario B — 24,000-GPU production trainer, large network org already fluent in EVPN-BGP, multi-year build where optics/switch spend and second-sourcing dominate TCO.**

**Verdict: RoCEv2 (very plausibly Spectrum-X for the CC, depending on risk appetite).** At 24K endpoints the **optics/cabling line item and vendor lock-in are enormous**, and this org **has the network engineering muscle** to run the PFC/ECN/DCQCN qualification and operate it. This is precisely the Meta/Llama-3 configuration: a 24K-GPU RoCE Clos on Arista/OCP switches — and notably they found DCQCN's firmware behavior unreliable at 400G and ran **PFC-only with deep-buffer switches and receiver-driven traffic shaping**, proving a top-tier team can operate RoCE at extreme scale and even discard the "required" congestion-control component. The tuning risk that disqualified the 512-GPU team is *affordable* here because the org can absorb it, and the TCO/lock-in savings are *decisive* at 24K. If leadership wants to buy down the tuning risk without going full IB, **Spectrum-X** is the middle path — Ethernet ecosystem, vendor-managed losslessness.

The senior signal is that **the same two technologies produce opposite verdicts** once you weight the axes by *cluster size, team skills, and TCO structure* rather than raw latency.

## Practice (feeds the deliverable)

Produce the **IB-vs-RoCE decision table + two justified verdicts** for the Network Architecture Read deliverable.

**Task.** (1) Build a decision table on these axes: **latency/determinism, NCCL bus GB/s (bound), control plane, tuning risk, cost/optics, vendor lock-in, SHARP availability** (add a Spectrum-X column). (2) Pick a winner for **two** clusters with **different** verdicts — a **512-GPU research cluster** and a **24K-GPU production trainer** — and justify each in a short paragraph.

**Acceptance criteria:**
- Every axis is filled for **both** IB and RoCE (Spectrum-X column optional but rewarded).
- The two verdicts **differ**, and each justification names **which axes drove it** (not "IB is faster") — e.g., cite optics cost + team skills for the 24K case, engineer-time + determinism for the 512 case.
- You correctly name at least **one RoCE failure mode** (PFC deadlock *or* head-of-line blocking) and state the mitigation, and you state that **SHARP / in-network compute is IB-only**.
- You reference the **Llama-3 / Meta 24K RoCE** data point (bonus: that they ran without DCQCN) as real-world evidence for the large-scale verdict.
- One page; a reviewer can check each claim against facts.

## Self-check

**(a) What is a PFC deadlock and where does it come from?**

**Answer:** A PFC deadlock is a **cyclic PAUSE dependency**: switch A's ingress buffer for the RDMA priority fills and it PAUSEs its upstream B; B's buffer then fills and it PAUSEs C; C PAUSEs A — forming a cycle in which no switch can drain because each is waiting on the next, and the fabric region wedges. It comes from **PFC being coarse (per-class) and hop-by-hop backpressure combined with a topology/routing that permits a cyclic buffer dependency** (non-minimal or reordered paths, or misrouted classes, make it likely). Mitigations: deadlock-free/up-down routing, PFC watchdogs that drop the paused class after a timeout, correct headroom, and strategically **keeping PFC out of steady state** by tuning ECN/DCQCN so PAUSE almost never fires.

**(b) Why does DCQCN exist if you already have PFC?**

**Answer:** Because PFC alone is too blunt and dangerous to run as your congestion control. PFC PAUSEs an **entire priority class hop-by-hop**, so one hot flow stalls innocent flows sharing that class (**head-of-line blocking / congestion spreading**) and cyclic PAUSEs can **deadlock**. PFC has no idea *which* flow is at fault. DCQCN is the **fine-grained, end-to-end, flow-specific** controller: it consumes ECN marks (reflected as CNPs) and **rate-limits the actual offending flow at its source NIC** before buffers fill enough to trigger a PAUSE. The design intent is DCQCN handling everyday congestion so **PFC only ever fires as a last-resort safety net** for microbursts DCQCN was too slow to catch.

**(c) When does RoCE / Spectrum-X beat IB on total cost of ownership?**

**Answer:** At **large scale with a capable network org**, where three things line up: (1) the **optics/cabling line item is huge** and RoCE's broad multi-vendor 400G/800G Ethernet optics market plus second-sourcing beats IB's premium single-source optics; (2) the team **already runs Ethernet/EVPN-BGP** so it reuses existing skills and tooling instead of hiring scarce IB/SM expertise; and (3) the org can **absorb the PFC/ECN/DCQCN qualification and operations** cost that RoCE demands. Meta's 24K-GPU Llama-3 RoCE fabric is the proof point. It flips **against** RoCE when the cluster is small (lock-in/optics savings are trivial next to engineer-time), when the team can't own the tuning risk, or when the workload is all-reduce-bound enough that IB's **SHARP** in-network reduction is a decisive throughput (hence $/token) win. Spectrum-X narrows RoCE's tuning-risk gap but reintroduces NVIDIA lock-in, so it wins TCO mainly when you want RoCE's ecosystem *and* vendor-managed losslessness and are willing to pay the switch+SuperNIC premium.

## Resources

1. **Vendor-neutral GPU-cluster networking: InfiniBand vs RoCE (read critically).** Good axis-by-axis framing; treat vendor-leaning claims skeptically and check them against the failure modes above. https://vcluster.com/blog/gpu-cluster-networking-infiniband-roce
2. **Glenn Lockwood — "Networking for LLM training" (critical practitioner take).** The best skeptical, first-principles treatment of why LLM fabrics are built the way they are; read for the *reasoning*, not just conclusions. https://glennklockwood.com/garden/networking-for-LLM-training
3. **Llama 3 paper, §3.3.1 (RoCE + congestion-control choices).** The canonical large-scale RoCE data point — 24K-GPU Clos, and their decision to run **without DCQCN** at 400G using deep-buffer switches and receiver-driven shaping. https://arxiv.org/abs/2407.21783
