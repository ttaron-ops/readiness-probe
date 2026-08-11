---
lesson: "09.4"
title: "InfiniBand vs RoCEv2: engineering losslessness, and when each wins"
module: "09"
concept: "InfiniBand vs RoCEv2: engineering losslessness, and when each wins"
status: not-started
est_time: "8h"
prev: "03-rdma-fundamentals.md"
next: "05-gpudirect-and-sharp.md"
artifacts: []
sources: 9
---
# 09.4 · InfiniBand vs RoCEv2: engineering losslessness, and when each wins

> **Concept.** InfiniBand is lossless *by construction* (credit-based hardware flow control); RoCEv2 is RDMA bolted onto Ethernet that you must *engineer* into losslessness with PFC + ECN + DCQCN — and the senior skill is naming which trade wins for a given cluster, not chanting "IB is faster." A fourth path exists too: Meta's production answer to "RoCE congestion control is hard" wasn't better DCQCN tuning, it was moving the job to a different layer entirely.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

Lesson 03 established RDMA's mechanism — kernel bypass, zero-copy, one-sided WRITEs over a hardware-reliable RC queue pair — and drew one deliberate boundary: RC's reliable-delivery guarantee (packets that arrive get delivered correctly, in order) is *not* the same claim as "the fabric never drops packets." That gap matters enormously, because RDMA's classic reliable transport recovers from a drop with a **go-back-N** retransmit that collapses throughput for the whole window, not just the lost packet. This lesson is entirely about closing that gap: how InfiniBand and RoCEv2 each make the fabric itself loss-free enough for RC to actually deliver on its promise — one by building losslessness into the hardware from the ground up, the other by engineering it on top of Ethernet, a medium explicitly designed to tolerate drops.

This is the module's highest-yield lesson because it's where the interview stops being about vocabulary and starts being about judgment: not "what is PFC" but "given this cluster, do I want IB's determinism or RoCE's economics, and can I defend the number attached to that answer."

## Why this matters

This is the highest-yield interview topic in the module because it is where a hiring panel at CoreWeave / NVIDIA / a neocloud finds out whether you *design fabrics* or just recite specs. The junior answer is "InfiniBand, it's lower latency." The answer that gets the senior offer is: *"Depends. IB buys you determinism and SHARP but locks you to NVIDIA and a thinner optics ecosystem; RoCEv2 reuses my Ethernet/EVPN-BGP muscle and a fat optics market but I pay for losslessness in PFC/ECN tuning risk — and at 24K+ GPUs Meta didn't just tune DCQCN harder, they turned it off and moved congestion avoidance into the collective library. Here's when I'd pick each."* You need mechanism, failure modes, and a cost model.

The stakes are literal money and literal uptime. Get losslessness wrong on RoCE and you don't get "a bit slower" — you get **PFC deadlock** that wedges a fabric region, or **head-of-line blocking** that tanks collective efficiency and silently burns 20–30% of GPU-hours. Get vendor lock wrong on IB and you've signed a multi-year, single-source optics-and-switch bill on a fabric you can't second-source. Both failure modes have a dollar figure your FinOps instinct should be itching to attach.

## What's new here (calibration)

You already know 01b's "loss is normal, TCP recovers from it" model — this lesson exists precisely because RDMA fabrics invert that assumption, and you built the RDMA-side half of that inversion in lesson 03. You know 02b's PCIe/NVLink topology matrix and 08's NCCL collectives, so the *consumer* of a lossless fabric (a barrier-synchronous all-reduce) is not re-explained here. What's genuinely new:

- The **two structurally different ways** to get a lossless RDMA fabric — intrinsic (IB) vs engineered (RoCEv2) — and why that's a real architectural fork, not a marketing distinction.
- The **PFC / ECN / DCQCN triangle**, each mechanism's job, and the specific failure modes (deadlock, head-of-line blocking, and — new here — ECMP hash polarization) that show up when the triangle is mistuned.
- A **fourth path**, evidenced by real production practice: substituting library-level (NCCL) coordination for network-level congestion control entirely, which is what Meta actually shipped at scale — a genuinely different answer than "IB vs RoCE vs Spectrum-X."
- A **cost model** you can defend with real numbers (optics economics, port density, headcount), not a single "IB is 15% faster" factoid.

## Core concepts

### 1. Why RDMA is so loss-allergic (the mechanism behind "must be lossless")

In 01b, "the network" was the Linux kernel deciding what to do with a packet — and crucially, **dropping was normal and fine**: TCP's whole design assumes loss and recovers from it with retransmits and a congestion window. RDMA inverts that assumption. Classic RDMA reliable transport (RC, from lesson 03) recovers from a loss with **go-back-N**: on a dropped packet it retransmits from the lost packet *and everything after it*, not just the missing one. At 400G, "everything after it" is a large window, so one drop collapses that flow's effective throughput for a recovery interval — a single lost packet is not a small slowdown, it's a throughput cliff. TCP tolerates loss because it was designed around it (selective ACK, fast retransmit, a congestion window that treats loss as signal, all covered in 01b). Classic RoCE was not. (Newer NICs add selective-retransmit/"improved" RoCE modes that soften this, but the fabric is still engineered to make drops *rare*, not to recover gracefully from them.) This is *why* the entire PFC/ECN/DCQCN edifice below exists: not to go faster, but to drive the congestion-drop probability toward zero so go-back-N almost never triggers.

So the core idea of this lesson: **there are two fundamentally different ways to get a lossless RDMA fabric.**

- **InfiniBand:** losslessness is *intrinsic*. A separate L2 protocol, purpose-built, with **credit-based flow control** in hardware — a sender may only transmit if the receiver has advertised buffer credits, so congestion means "wait for credits," never "drop." A centralized **Subnet Manager (SM)** computes routing and manages the fabric. There is no "engineering losslessness"; it's the substrate.
- **RoCEv2 (RDMA over Converged Ethernet v2):** take RDMA and run it as a **UDP/IP payload over ordinary routed Ethernet** (UDP dest port 4791; being IP-routable is the "v2" upgrade over RoCEv1's raw-L2). Ethernet drops under congestion, so you must **manufacture** losslessness with a stack of Ethernet features: **PFC** to stop drops, **ECN** to signal congestion, **DCQCN** to react to it. You are re-using the Ethernet control plane you already know (BGP/EVPN, ECMP) — and inheriting all of its failure modes.

The contrast to hold: **IB deletes the datapath *and* gives you a lossless substrate for free; RoCE deletes the datapath but hands you the substrate as a tuning project.**

### 2. InfiniBand mechanism (the "free lossless" path)

- **Credit-based link-level flow control.** Every link tracks receiver buffer credits; a sender transmits only against available credits. Congestion → backpressure via credit starvation → **zero congestion drops** by construction. No tuning knob to get wrong.
- **Subnet Manager (SM).** A centralized control plane (e.g., NVIDIA UFM, or `opensm`) discovers topology, assigns LIDs, computes forwarding tables, and can drive **adaptive routing**. One brain; deterministic behavior; also one thing to make HA.
- **SHARP (Scalable Hierarchical Aggregation and Reduction Protocol).** *In-network compute*: the switches themselves perform the reduction for all-reduce/reduce-scatter, so data is summed in the fabric instead of shuttled around a ring. This is a real, IB-only throughput multiplier for collectives. Anchor the generation mapping precisely, because it's load-bearing for the next lesson too: **Quantum-2 (NDR, 400G) ships SHARPv3; Quantum-X800 (XDR, 800G, 144×800G ports) ships SHARPv4**, which adds FP8 support and ReduceScatter/ScatterGather. When someone says "IB isn't just lower latency, it changes the collective's algorithmic cost" — SHARP is what they mean.
- **Latency/scale anchors:** ~100–130 ns switch hop, sub-µs end-to-end small message; deterministic tail. NICs: **ConnectX-7 (NDR/400G)**, **ConnectX-8 (XDR/800G)**, **BlueField-3** DPU when you want to offload/manage on the NIC.
- **The catch:** it's a **single-vendor** stack — NVIDIA/Mellanox switches, NICs, cables, SM, and a comparatively **narrow, premium optics ecosystem**. You cannot second-source it.

### 3. RoCEv2 mechanism (the "engineered lossless" path)

RoCEv2 is RDMA-in-UDP over routed Ethernet. To make routed Ethernet lossless enough for RDMA you stack three mechanisms — and you must understand each *separately*:

- **PFC (Priority Flow Control, 802.1Qbb).** Per-priority (per traffic class) PAUSE. When a switch's ingress buffer for the RDMA priority fills past a threshold, it sends a **PAUSE** frame **upstream** telling the previous hop to stop sending *that class*. This is the mechanism that turns "drop" into "backpressure" — RoCE's answer to IB credits. It is **coarse** (stops a whole priority, not a flow) and **hop-by-hop**, which is exactly why it's dangerous (see failure modes).
- **ECN (Explicit Congestion Notification).** An *end-to-end, proactive* congestion signal: a congested switch, instead of (or before) dropping, **marks** the IP ECN bits (a CE — congestion-experienced — bit) on packets crossing a queue-depth threshold. The receiver sees the mark and reflects a **CNP (Congestion Notification Packet)** back to the sender. ECN says "you, specific flow, are causing congestion" — information PFC's blunt PAUSE cannot convey. The framing that Azure's production tuning teams use, and that's worth memorizing: ECN is the *proactive, end-to-end* signal; PFC is the *coarse, reactive backstop* — a hierarchy, not two independent knobs.
- **DCQCN (Data Center Quantized Congestion Notification).** The **congestion-control algorithm** that consumes ECN/CNP signals and **rate-limits the offending flow at the sender NIC**, then gently ramps back up. It's the RoCE analog of a congestion window, running in the NIC.

You must also **calibrate PFC headroom** (buffer reserved so in-flight packets during the PAUSE round-trip aren't dropped) and **ECN thresholds** (mark early enough that DCQCN reacts *before* PFC has to PAUSE). Getting these numbers right, per switch model and per link speed, **is** the RoCE tuning project — and getting the *ordering* wrong (ECN threshold set too aggressively low, at or above PFC's trigger) inverts the intended hierarchy: PFC ends up leading instead of backstopping, and you get exactly the pause storms and throughput collapse the whole stack exists to prevent. Vendor deployment guides (Arista/Broadcom, referenced below) exist specifically to give you defensible starting values for headroom and thresholds per switch generation and link speed, rather than deriving them from scratch.

### 4. Why DCQCN exists if you already have PFC (the classic interview trap)

Because **PFC alone is a disaster at scale.** PFC is coarse (pauses an entire priority class, not the one bad flow) and propagates **backward hop-by-hop**: a hot spot at one switch sends PAUSE upstream, that switch's buffers fill and it PAUSEs *its* upstream, and the backpressure **spreads like a traffic jam** — this is **congestion spreading / head-of-line blocking**, where innocent flows sharing the paused class are stalled by one victim link. Worse, cyclic PAUSE dependencies can **deadlock** the fabric. PFC is a blunt "stop *everything* on this class *now*" hammer with no notion of *who* is at fault.

DCQCN exists to **keep PFC from ever firing.** ECN+DCQCN is the *fine-grained, end-to-end, flow-specific* controller: it slows the *actual* offending flow at its source before buffers fill enough to trigger a PAUSE. The intended operating regime: **DCQCN does the everyday congestion control; PFC is the last-resort safety net that catches microbursts DCQCN was too slow for.** If your fabric is relying on PFC in steady state, your ECN/DCQCN tuning is wrong.

### 5. Failure modes you must be able to name

- **PFC deadlock.** Cyclic buffer dependency: switch A's RDMA-class buffer is full and PAUSEs B; B's fills and PAUSEs C; C's fills and PAUSEs A — a **PAUSE cycle** with no packet able to drain. The fabric region wedges. Root cause is topology + routing that allows a cyclic dependency among paused buffers (non-minimal or reordered paths make it worse). Mitigations: careful up/down or deadlock-free routing, watchdogs that drop the paused class after a timeout, and — the strategic move — **minimize reliance on PFC** via good DCQCN so PAUSE is rare.
- **Head-of-line blocking / congestion spreading.** As above: one congested egress PAUSEs a whole class upstream, stalling unrelated flows. Kills collective efficiency because a collective is a barrier — stalling *any* participant stalls the step.
- **ECMP hash polarization — a third, distinct failure mode.** Standard Ethernet spreads flows across parallel paths with per-flow (5-tuple) ECMP hashing. LLM training produces a *small number of very large, bursty, long-lived* flows rather than many small ones, so the hash frequently **collides multiple elephant flows onto the same link** while parallel links sit idle — instant, avoidable congestion that has nothing to do with real oversubscription and *still* triggers PFC/ECN pressure downstream. Alibaba's HPN paper (SIGCOMM 2024) documents this as a first-class failure mode distinct from PFC deadlock and HOL blocking, and their fix wasn't more congestion-control tuning — it was topological: a **2-tier, dual-plane architecture** with dual-ToR uplinks that sidesteps the hash-collision problem structurally rather than reacting to it after the fact. The lesson for you: not every RoCE pathology is a PFC/ECN tuning bug — some are routing/topology problems that tuning cannot fix.
- **Slow/silent misconfig.** Mismatched ECN thresholds vs PFC headroom, wrong DSCP-to-priority mapping across a vendor boundary, or a firmware change to DCQCN behavior — none of these page you; they just quietly cost GPU-hours.

### 6. The three RoCE mechanisms in one line each (memorize)

- **PFC** = *link-level, per-class PAUSE* — the last-resort "stop dropping" hammer; coarse and deadlock-prone.
- **ECN** = *end-to-end congestion signal* — the switch marks packets so the sender learns *which* flow is hot.
- **DCQCN** = *NIC-based congestion-control algorithm* — consumes ECN, rate-limits the offending flow so PFC rarely fires.

Mnemonic: **ECN sees it, DCQCN slows it, PFC catches what slips through.** If you can only say one sentence in an interview, say that.

### 7. Control-plane operations — the part that decides who you hire

- **InfiniBand:** one **Subnet Manager** is the brain. Simplicity (deterministic routing, one source of truth) but you must make the SM **highly available** (redundant SMs, failover) and you need **IB-specific tooling and skills** (`ibdiagnet`, UFM) that are scarcer and pricier on the hiring market.
- **RoCE:** the control plane is your existing **Ethernet fabric** — **BGP/EVPN** underlay/overlay, ECMP, your current telemetry (streaming stats, sFlow/gNMI), your current on-call. No new control-plane brain to make HA, and the skills are commodity. The catch is that "the fabric config" now silently includes the entire PFC/ECN/DCQCN tuning surface — so you trade *control-plane novelty* for *data-plane tuning burden*.

### 8. Spectrum-X — "managed RoCE"

NVIDIA's answer to "RoCE tuning is a research project": **Spectrum-X** = the **Spectrum-4 switch (SN5600)** + **BlueField-3 / ConnectX SuperNICs**, co-designed so the Ethernet fabric behaves like a lossless RDMA fabric out of the box — hardware **adaptive routing**, fast **telemetry-based congestion control**, and per-flow load balancing that plain ECMP+PFC can't do. Concrete port-density argument for the "commodity Ethernet economics" claim: the **SN5600** switches **64×800G or 128×400G ports at 51.2 Tb/s** aggregate switching — compare that radix directly against a Quantum-2 NDR IB switch at **64×400G ports**, and you can see why RoCE's port density (and thus fewer switch tiers/hops for a given endpoint count) is part of its cost story, independent of the optics-market argument. The pitch: **Ethernet ecosystem, IB-like determinism, someone else did the tuning.** The cost: you're back toward a **single-vendor** stack (Spectrum-X's magic wants NVIDIA switches *and* NVIDIA SuperNICs), so it partially trades away the "reuse commodity Ethernet" benefit that made RoCE attractive, and it is **not** a fully automatic, zero-tuning-risk product — it's *managed and co-designed*, meaning you still configure PFC/ECN, just against vendor-validated defaults instead of deriving them yourself. It's the middle box in the decision, not a free lunch.

(Dollar/hardware figures above — SN5600 port counts, Quantum-2/X800 generations — are a **2026 snapshot**; treat them as current-generation anchors, not permanent constants, since silicon generations turn over roughly annually in this market.)

### 9. Meta's real answer: change which layer owns congestion avoidance

Here is the finding that makes this lesson current rather than textbook, and it's worth stating precisely because the loose version ("Meta disabled DCQCN, so no congestion control") is wrong and will read as a misunderstanding in an interview.

Meta's SIGCOMM 2024 paper, *"RDMA over Ethernet for Distributed Training at Meta Scale,"* documents that Meta ran **PFC-only, DCQCN-disabled 400G RoCE in production for a multi-year period** at 24K+ GPU scale. The mechanism they substituted is specific: they **co-designed the collective library (NCCL) and the RoCE transport to enforce receiver-driven traffic admission** — the receiver, which knows the actual traffic pattern of the collective it's running, controls when data is allowed to arrive, rather than a generic NIC-level algorithm reacting after the fact to ECN marks it didn't have context for. They additionally tuned ECMP itself for their specific traffic shape — an **Enhanced ECMP (E-ECMP)** scheme hashing on destination-QP fields, reported to improve collective performance (e.g. AllReduce) by up to **~40%** by reducing exactly the elephant-flow hash-collision problem described in §5 above.

Read this correctly: **this is not an absence of congestion control — it's congestion control implemented at a different layer.** DCQCN is a *generic*, NIC-resident algorithm that reacts to congestion signals without knowing what application generated the traffic. Receiver-driven admission is a *workload-specific*, library-resident mechanism that knows exactly what collective is running and can pace accordingly, arguably more precisely than a generic algorithm ever could. What made this viable rather than reckless: deep-buffer switches (Arista 7800-class) that absorb microbursts and reduce dependence on PFC ever firing, plus Meta's own network-engineering scale to validate the substitution rigorously before trusting it in production.

This is the genuine **fourth path** beyond the IB / RoCE / Spectrum-X trichotomy: not a different fabric technology, but a different *architectural boundary* for where congestion avoidance lives — network-level (DCQCN) vs library-level (receiver-driven admission co-designed with NCCL). It's evidence that "RoCE tuning risk" is not an immutable tax; a sufficiently capable org can retire part of it by moving the responsibility rather than perfecting the network-level knob.

### 10. The RoCE tuning checklist (what "engineering losslessness" concretely means when you don't have Meta's option)

For everyone who isn't running Meta's scale and engineering org, this is the project a senior candidate can enumerate:

- **Lossless traffic class:** map RDMA traffic to a dedicated PFC-enabled priority (DSCP → traffic class), and keep it **consistent across every switch and NIC** in the path — a single mismatched DSCP-to-priority map at a vendor boundary silently breaks losslessness.
- **PFC headroom:** reserve enough buffer per port so packets already in flight during the PAUSE propagation round-trip are not dropped. Headroom scales with **link speed × cable length** — the numbers change when you go 200G→400G→800G, which is exactly the trap that motivates careful per-generation revalidation. Vendor deployment guides (Arista/Broadcom, referenced below) publish concrete starting values per switch model and link speed.
- **ECN thresholds:** set the marking threshold **below** the PFC trigger so DCQCN reacts *before* PAUSE fires. If ECN marks too late — or too aggressively, inverting the intended hierarchy — PFC does all the work and you get congestion spreading. This exact tuning tension is what Azure's production ECN/PFC blog documents from real operational experience.
- **DCQCN parameters + firmware:** rate-decrease/increase constants, CNP behavior, and — the operational landmine — **NIC firmware version**, because DCQCN lives in NIC firmware and its behavior can change (or regress) across releases.
- **Deep-buffer vs shallow-buffer switches:** deep buffers absorb microbursts and reduce PFC dependence (Meta's Arista 7800 choice); shallow-buffer switches are cheaper but lean harder on PFC/ECN being perfect.
- **Deadlock-free routing + PFC watchdog:** ensure the topology/routing can't form a PAUSE cycle, and run a watchdog that drops a stuck paused class after a timeout.
- **ECMP hashing scheme:** default per-flow 5-tuple hashing collides elephant flows (§5); consider destination-QP-aware or otherwise workload-tuned hashing, or a topology (like Alibaba's dual-plane) that avoids the collision structurally.

Every line above is a place to be subtly wrong in a way that costs GPU-hours without paging anyone. That risk *is* the price of RoCE's cost savings — the trade you're naming for the interviewer.

### 11. The cost dimension (your FinOps differentiator)

- **Optics ecosystem:** Ethernet optics/DAC/AOC at 400G/800G are a **broad, multi-vendor, price-competitive** market; IB optics are **narrower and premium** and single-sourced. At 24K endpoints, cabling+optics is a serious line item — this is often where RoCE's TCO advantage actually lives, not in the switch ASIC.
- **Port density.** Spectrum-X's SN5600 at 64×800G/128×400G and 51.2 Tb/s vs a Quantum-2 IB switch at 64×400G is a concrete, quotable density argument for RoCE/Ethernet's cost edge at a given endpoint count — more endpoints per switch, or fewer tiers for a given fan-out.
- **Operational cost:** IB needs **IB-specific skills** (SM, fabric tooling) that are scarcer/pricier to hire; RoCE reuses your existing **Ethernet/EVPN-BGP** team and monitoring. But RoCE spends that saving back as **tuning/validation engineering** (PFC/ECN/DCQCN qualification per switch model) — unless, like Meta, you've retired part of that tax by moving congestion avoidance into the library.
- **Lock-in / second-sourcing:** IB = single vendor, weak negotiating position, roadmap risk. RoCE (non-Spectrum-X) = multi-vendor switches, real price leverage. Spectrum-X sits in between.
- **SHARP:** if your training is all-reduce-bound and you can exploit in-network reduction, IB's SHARP is a *performance* win that can change the TCO math back in IB's favor — it's not purely a cost line, it's a throughput multiplier (SHARPv3 on Quantum-2, SHARPv4 on Quantum-X800).

**How to actually model it (don't hand-wave "cheaper"):** TCO here is roughly `switch/NIC capex + optics/cabling capex + power/cooling opex + (fabric engineering + operations headcount) − (GPU-hours saved by higher collective efficiency)`. IB tends to win the *efficiency* term (determinism + SHARP) and the *headcount* term at small scale; RoCE tends to win the *optics capex*, *port-density*, and *second-sourcing/negotiation* terms and the *skills-reuse* term at large scale. The verdict flips based on **which term dominates for your cluster size** — that sentence is the whole senior competency being tested. (All dollar/hardware figures here are a 2026 snapshot — state the year whenever you quote one in an interview or a doc.)

### 12. The one-line summary to carry into an interview

**IB buys determinism and in-network compute with money and lock-in; RoCE buys ecosystem and cost leverage with tuning risk — and at extreme scale with a capable network org, that tuning risk can itself be retired by moving congestion avoidance into the collective library rather than perfecting the network-level knob. Small/latency-sensitive/all-reduce-bound → IB. Huge/Ethernet-fluent-org/optics-and-lock-in-dominated → RoCE (or Spectrum-X to buy down the tuning risk, or Meta's library-driven path if you have the engineering scale to validate it).** Everything else in this lesson is the justification for that sentence.

## Perspectives

**Theory.** IB's credit-based flow control and RoCE's PFC+ECN+DCQCN stack are two solutions to the exact same problem — "never let a buffer overflow to the point of dropping" — arrived at from opposite starting points. IB is *architectural*: losslessness is a property of the link protocol itself, verified once at design time, present everywhere by construction. RoCE is *operational*: losslessness is an emergent property of correctly configured, correctly interacting mechanisms that must be validated per switch model, per link speed, and re-validated on every firmware or topology change. Same goal, fundamentally different engineering posture — one you buy, one you build and maintain.

**Practice.** The real lesson from Meta's production fabric isn't "tune DCQCN harder" — it's that the *layer* responsible for congestion avoidance is itself a design choice. A generic NIC-level algorithm reacting to ECN marks and a collective library that knows its own traffic pattern and paces receivers accordingly are both valid answers to "how do we avoid congestion," and the second one can outperform the first *if* you have the engineering capacity to co-design and validate it. This is a genuinely different axis from "IB vs RoCE vs Spectrum-X" — it's "network-owned vs application-owned" congestion avoidance, and it's evidence that the RoCE tuning-risk tax is not fixed; it can be paid down by moving where the responsibility lives, not just by tuning the existing knobs better.

**Failure mode.** PFC deadlock and head-of-line blocking are the textbook failure modes, and you should be able to draw the PAUSE cycle on a whiteboard cold. But Meta's actual production posture on these was not "engineer a fabric where PFC never fires" — it was "make PFC survivable when it does fire": deep-buffer switches that absorb microbursts, watchdogs, and validated headroom, so that an occasional PAUSE is an absorbed event rather than a cascading incident. That's a subtly different engineering philosophy than "avoid PFC at all costs," and it's worth naming explicitly: resilience to the failure mode, not just avoidance of it, is what real production fabrics build for.

## Real-world use cases

- **Meta / SIGCOMM 2024 — "RDMA over Ethernet for Distributed Training at Meta Scale."** The canonical real-world account of a DCQCN-disabled RoCE fabric at 24K+ GPU scale, substituting receiver-driven traffic admission co-designed into the collective library, plus destination-QP-aware ECMP tuning reporting up to ~40% collective-performance gains. This is the single most load-bearing real-world citation in the module. Paper PDF: https://engineering.fb.com/wp-content/uploads/2024/08/sigcomm24-final246.pdf — blog version: https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/
- **Microsoft Azure — "Striking the Right Balance: ECN and PFC Thresholds for AI Clusters."** A second named hyperscaler's production ECN/PFC tuning playbook: explains why ECN must lead and PFC must trail, and documents the concrete instability (pause storms, throughput collapse, tail-latency blowups) that results when the threshold ordering is inverted. Useful as a counterpoint to Meta's story — a team that kept DCQCN and instead focused on getting the ECN/PFC threshold relationship right. https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/striking-the-right-balance-ecn-and-pfc-thresholds-for-ai-clusters/4468629
- **Alibaba HPN (SIGCOMM 2024) — "A Data Center Network for Large Language Model Training."** Documents ECMP hash polarization from LLM training's small number of large, bursty flows as a first-class failure mode distinct from PFC deadlock/HOL blocking, and shows a topological fix (2-tier dual-plane architecture with dual-ToR) rather than a congestion-control tuning fix — evidence that not every RoCE pathology is solved at the CC layer. https://dl.acm.org/doi/10.1145/3651890.3672265
- **Arista/Broadcom — "Lossless Network for AI/ML/Storage/HPC with RDMA" (RoCE Deployment Guide).** Vendor-primary, concrete configuration guidance for PFC headroom, ECN thresholds, and buffer allocation across real switch families (7050X/7060X/7280R/7800R) — the ground truth for what "the RoCE tuning project" actually involves in a config file, not just in theory. https://www.arista.com/assets/data/pdf/Broadcom-RoCE-Deployment-Guide.pdf

## Worked example — the decision table + two verdicts

Build the table on the axes below, then pick a winner for **two** clusters with **different** answers.

| Axis | InfiniBand (Quantum-X800 / ConnectX-8) | RoCEv2 (commodity Eth) | Spectrum-X (managed RoCE) |
|---|---|---|---|
| Small-msg latency / determinism | Best; sub-µs, deterministic tail | Good if tuned; tail at risk from PFC | Near-IB; adaptive routing tames tail |
| NCCL bus bandwidth (bound) | Highest; line-rate + SHARP in-network reduce | ~90–95% of IB *when perfectly tuned* | High; co-designed to approach IB |
| Control plane | Subnet Manager (one brain; make it HA) | BGP/EVPN + PFC/ECN/DCQCN you own (or library-level, à la Meta) | NVIDIA-managed CC + adaptive routing |
| Tuning risk | Low (lossless by construction) | **High** (PFC deadlock, ECN/headroom, DCQCN) — reducible by moving CC to the library, at engineering cost | Low–medium (vendor did the tuning) |
| Cost / optics / density | Premium, narrow, single-source optics; 64×400G radix | **Cheapest**, broad multi-vendor optics | Mid; NVIDIA switch+SuperNIC premium; 64×800G/128×400G radix |
| Vendor lock-in | **High** (NVIDIA end-to-end) | **Low** (multi-vendor Ethernet) | Medium–high (NVIDIA switch+NIC) |
| SHARP / in-network compute | **Yes** (SHARPv3 on Quantum-2, SHARPv4 on Quantum-X800) | No | No (in-network compute is IB-only) |
| Skills reused | IB-specific (scarcer) | Existing Ethernet/EVPN-BGP team | Ethernet team + vendor support |

**Scenario A — 512-GPU research cluster, mixed/bursty workloads, small ops team, latency-sensitive, wants max out-of-box performance and minimal fabric-engineering headcount.**

**Verdict: InfiniBand.** At 512 GPUs the vendor-lock and premium-optics penalties are **small in absolute dollars** and easily dominated by engineer-time. This team cannot afford to run a PFC/ECN/DCQCN qualification project, chase a PFC-deadlock incident, or — like Meta — build and validate a custom receiver-driven admission scheme; that substitution only pays off at a scale and engineering headcount this team doesn't have. IB's *lossless-by-construction* substrate means the fabric "just works" with a deterministic tail, and if the research is all-reduce-heavy, **SHARP** is a direct win. The thing RoCE saves — optics cost and reused Ethernet skills — is exactly what this small, latency-sensitive team values *least*. Buy determinism with money, not headcount.

**Scenario B — 24,000-GPU production trainer, large network org already fluent in EVPN-BGP, multi-year build where optics/switch spend and second-sourcing dominate TCO.**

**Verdict: RoCEv2 (very plausibly with Meta-style library-level congestion coordination, or Spectrum-X if the org prefers vendor-managed losslessness over building its own).** At 24K endpoints the **optics/cabling line item and vendor lock-in are enormous**, and this org **has the network engineering muscle** to run the PFC/ECN/DCQCN qualification and operate it — or, if it has Meta's scale of investment, to go further and retire part of the network-level tuning risk entirely by moving congestion avoidance into the collective library. This is precisely the Meta/Llama-3 configuration: a 24K-GPU RoCE Clos on Arista/OCP switches, and Meta's SIGCOMM paper shows they didn't stop at "tune DCQCN carefully" — they ran **PFC-only, DCQCN-disabled, with deep-buffer switches and receiver-driven traffic admission co-designed into NCCL**, proving a top-tier team can operate RoCE at extreme scale and even discard the "required" network-level congestion-control component by moving that responsibility up a layer. The tuning risk that disqualified the 512-GPU team is *affordable and even reducible* here because the org can absorb and re-architect around it, and the TCO/lock-in savings are *decisive* at 24K. If leadership wants to buy down the tuning risk without going full IB or building Meta's custom stack, **Spectrum-X** is the middle path — Ethernet ecosystem, vendor-managed losslessness.

The senior signal is that **the same two technologies produce opposite verdicts** once you weight the axes by *cluster size, team skills, and TCO structure* rather than raw latency — and that a genuinely sophisticated team's answer isn't limited to "pick IB or RoCE," it can be "change which layer owns congestion avoidance."

## Practice (feeds the deliverable)

Produce the **IB-vs-RoCE decision table + two justified verdicts** for the Network Architecture Read deliverable.

**Task.** (1) Build a decision table on these axes: **latency/determinism, NCCL bus GB/s (bound), control plane, tuning risk, cost/optics/density, vendor lock-in, SHARP availability** (add a Spectrum-X column). (2) Pick a winner for **two** clusters with **different** verdicts — a **512-GPU research cluster** and a **24K-GPU production trainer** — and justify each in a short paragraph.

**Acceptance criteria:**
- Every axis is filled for **both** IB and RoCE (Spectrum-X column optional but rewarded).
- The two verdicts **differ**, and each justification names **which axes drove it** (not "IB is faster") — e.g., cite optics cost + team skills for the 24K case, engineer-time + determinism for the 512 case.
- You correctly name at least **one RoCE failure mode** (PFC deadlock, head-of-line blocking, or ECMP hash polarization) and state the mitigation, and you state that **SHARP / in-network compute is IB-only**.
- You reference the **Meta 24K RoCE fabric** as real-world evidence for the large-scale verdict, and state the **actual mechanism** they substituted for DCQCN (receiver-driven admission in the collective library) — not the shorthand "they turned off congestion control."
- One page; a reviewer can check each claim against facts.

## Common pitfalls

- **Treating "Meta disabled DCQCN" as "Meta ran with no congestion control."** They replaced a network-level, NIC-resident algorithm with a library-level, receiver-driven admission scheme co-designed into NCCL — a different mechanism at a different layer, not an absence of one. Saying "Meta doesn't need congestion control" in an interview is a tell that you read a headline, not the paper.
- **Citing an inconsistent SHARP version mapping.** The correct mapping is **Quantum-2 (NDR) = SHARPv3; Quantum-X800 (XDR) = SHARPv4** (v4 adds FP8 and ReduceScatter/ScatterGather). Getting this backwards or citing "SHARPv2" for current-generation IB is a small but real credibility hit with anyone who's touched the hardware recently.
- **Assuming Spectrum-X eliminates all RoCE tuning risk.** It's *managed and co-designed*, not automatic — you still configure PFC/ECN against vendor-validated defaults, and it still requires NVIDIA switches and SuperNICs to get the adaptive-routing benefit. It buys down risk; it doesn't buy it to zero, and it reintroduces meaningful vendor lock-in.
- **Believing IB is lossless at every layer.** The **link layer** is lossless via credit-based flow control — that part is true and structural. But application-level or endpoint-level drops (buffer exhaustion at the endpoint, a misbehaving QP, a software bug) are still possible; "lossless fabric" describes the wire, not a blanket guarantee that no message is ever lost for any reason.
- **Quoting a single "IB gets 85–95% of RoCE performance" (or the reverse) figure as universal.** Any such percentage is scoped to a specific workload, tuning quality, and hardware generation — always state what it's scoped to (e.g. "~90–95% of IB bus bandwidth, RoCE, well-tuned, NCCL all-reduce, 400G class") rather than repeating a bare number as a law of physics.

## Self-check

**(a) What is a PFC deadlock and where does it come from?**

**Answer:** A PFC deadlock is a **cyclic PAUSE dependency**: switch A's ingress buffer for the RDMA priority fills and it PAUSEs its upstream B; B's buffer then fills and it PAUSEs C; C PAUSEs A — forming a cycle in which no switch can drain because each is waiting on the next, and the fabric region wedges. It comes from **PFC being coarse (per-class) and hop-by-hop backpressure combined with a topology/routing that permits a cyclic buffer dependency** (non-minimal or reordered paths, or misrouted classes, make it likely). Mitigations: deadlock-free/up-down routing, PFC watchdogs that drop the paused class after a timeout, correct headroom, and strategically **keeping PFC out of steady state** by tuning ECN/DCQCN (or, as Meta did, replacing DCQCN with library-level admission) so PAUSE almost never fires.

**(b) Why does DCQCN exist if you already have PFC?**

**Answer:** Because PFC alone is too blunt and dangerous to run as your congestion control. PFC PAUSEs an **entire priority class hop-by-hop**, so one hot flow stalls innocent flows sharing that class (**head-of-line blocking / congestion spreading**) and cyclic PAUSEs can **deadlock**. PFC has no idea *which* flow is at fault. DCQCN is the **fine-grained, end-to-end, flow-specific** controller: it consumes ECN marks (reflected as CNPs) and **rate-limits the actual offending flow at its source NIC** before buffers fill enough to trigger a PAUSE. The design intent is DCQCN handling everyday congestion so **PFC only ever fires as a last-resort safety net** for microbursts DCQCN was too slow to catch.

**(c) What did Meta replace DCQCN with, and why does that still count as congestion control?**

**Answer:** Meta replaced the network-level, NIC-resident DCQCN algorithm with **receiver-driven traffic admission co-designed into the NCCL collective library and the RoCE transport** — the receiver, which knows the exact traffic pattern of the collective it's running, controls when data is allowed to arrive, rather than a generic algorithm reacting to ECN marks after congestion has already started forming. This still counts as congestion control because its function — preventing more traffic from arriving than the fabric/receiver can absorb, avoiding the buffer buildup that would otherwise trigger PFC — is identical to DCQCN's function. What changed is *which layer* implements that function and *how much application context* it has access to: a workload-aware library mechanism instead of a generic NIC-level reaction.

**(d) Which SHARP version ships on Quantum-X800, and which ships on Quantum-2?**

**Answer:** **Quantum-X800 (XDR, 800G) ships SHARPv4**, adding FP8 support and ReduceScatter/ScatterGather aggregation. **Quantum-2 (NDR, 400G) ships SHARPv3.** Getting this mapping backwards is a common and easily-caught error — anchor it as "the newer switch generation ships the newer SHARP version, v4 on X800, v3 on Quantum-2," and don't default to citing an older "v2" figure from pre-Quantum-2 material.

**(e) Besides PFC deadlock and head-of-line blocking, what third RoCE-adjacent failure mode does Alibaba's HPN paper document, and how did they fix it?**

**Answer:** **ECMP hash polarization**: LLM training generates a small number of very large, bursty, long-lived flows, and standard per-flow (5-tuple) ECMP hashing frequently collides multiple of these "elephant flows" onto the same physical link while parallel links sit idle — causing avoidable congestion (and downstream PFC/ECN pressure) that has nothing to do with genuine oversubscription. Alibaba's fix was **not** a congestion-control tuning change but a **topological** one: a 2-tier, dual-plane architecture with a dual-ToR design that structurally avoids the hash-collision pattern rather than reacting to it after it occurs. The takeaway: some RoCE pathologies live in the routing/topology layer, not the PFC/ECN/DCQCN layer, and tuning the latter won't fix the former.

## Connections & what's next

This lesson is the direct sequel to 03: RDMA's RC transport guarantees reliable delivery of what arrives, and this lesson is entirely about the separate, upstream question of how the fabric avoids dropping packets in the first place — intrinsically for IB, engineered for RoCEv2, or substituted-to-a-different-layer entirely in Meta's production case. It also connects backward to 02's fabric-shape lesson: oversubscription ratios and rail-optimized design tell you *how much* bandwidth exists at each tier, and this lesson tells you *how reliably* that bandwidth can be used without collapsing under congestion — the two are complementary halves of "can this fabric actually run a training job at line rate." The Llama-3 §3.3.1 anchor (already load-bearing in lesson 02) reappears here as the canonical large-scale RoCE data point.

Next is lesson 05: **GPUDirect over the fabric, plus SHARP.** This lesson introduced SHARP as an IB-only throughput multiplier and gave you the version mapping (SHARPv3 on Quantum-2, SHARPv4 on Quantum-X800); the next lesson goes inside the mechanism — how in-network reduction actually restructures the collective's algorithm, which collective shapes benefit and which don't, and how GPUDirect RDMA's same-root-complex precondition (introduced in lesson 03, rooted in 02b) extends fabric-wide as "same rail." Where this lesson answered "which fabric, and why," lesson 05 answers "how do I prove the fabric is actually being used correctly for a specific job," down to the `nvidia-smi topo -m` and NCCL environment-variable level.

## References & further reading

**Primary sources**
- Meta / SIGCOMM 2024 — *"RDMA over Ethernet for Distributed Training at Meta Scale"* (paper PDF). The primary technical account of DCQCN-disabled RoCE plus receiver-driven admission at 24K+ GPU scale — the most load-bearing citation in this lesson. https://engineering.fb.com/wp-content/uploads/2024/08/sigcomm24-final246.pdf
- Llama 3 paper, §3.3.1 ("Network") — https://arxiv.org/abs/2407.21783 — the canonical large-scale RoCE data point (24K-GPU Clos, oversubscription and congestion-control choices); read for the network section specifically.
- Alibaba HPN (SIGCOMM 2024) — *"A Data Center Network for Large Language Model Training."* Read for the ECMP hash-polarization failure mode and the dual-plane topological fix. https://dl.acm.org/doi/10.1145/3651890.3672265
- Arista/Broadcom — *"Lossless Network for AI/ML/Storage/HPC with RDMA"* (RoCE Deployment Guide). Read for concrete, vendor-primary PFC headroom and ECN threshold configuration guidance across real switch families. https://www.arista.com/assets/data/pdf/Broadcom-RoCE-Deployment-Guide.pdf
- Arista/Broadcom — *"AI Networking Deployment Guide."* Read for deep-buffer vs shallow-buffer switch tradeoffs and end-to-end AI-fabric deployment guidance. https://www.arista.com/assets/data/pdf/Arista-Broadcom-AI-Networking-Deployment-Guide.pdf

**Real-world engineering blogs**
- Meta Engineering — *"RoCE networks for distributed AI training at scale."* What it shows: the blog-form account of the DCQCN-off decision and the E-ECMP tuning that improved AllReduce performance by up to ~40%. https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/
- Microsoft Azure — *"Striking the Right Balance: ECN and PFC Thresholds for AI Clusters."* What it shows: a second named hyperscaler's production ECN/PFC tuning playbook and the real instability that results from getting the threshold ordering wrong. https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/striking-the-right-balance-ecn-and-pfc-thresholds-for-ai-clusters/4468629

**Deeper dives**
- Glenn Lockwood — *"Networking for LLM training."* The best skeptical, first-principles treatment of why LLM fabrics are built the way they are; read for the *reasoning*, not just conclusions. https://glennklockwood.com/garden/networking-for-LLM-training
- vcluster — *"GPU cluster networking: InfiniBand vs RoCE" (read critically).* Good axis-by-axis framing; treat vendor-leaning claims skeptically and check them against the failure modes and Meta/Alibaba evidence above. https://vcluster.com/blog/gpu-cluster-networking-infiniband-roce
