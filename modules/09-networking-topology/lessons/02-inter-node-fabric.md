---
lesson: "09.2"
title: "The inter-node fabric: Clos, bisection, and rail-optimized oversubscription"
module: "09"
concept: "The inter-node fabric: Clos, bisection, and rail-optimized oversubscription"
status: not-started
est_time: "8h"
prev: "01-intra-to-inter-handoff.md"
next: "03-rdma-fundamentals.md"
artifacts: []
sources: 11
---

# 09.2 · The inter-node fabric: Clos, bisection, and rail-optimized oversubscription

> **Concept.** A GPU cluster's fabric is a Clos/fat-tree, and its cost is set by one dial — how much bisection bandwidth you buy at each tier; rail-optimized design lets you keep that dial low at the spine *for free* because LLM traffic is rail-local and NVLink absorbs the rest. But "3-tier Clos, 1:7 at the top" is one design, not the only correct one — this lesson also shows you when and why hyperscalers deviate from it.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

09.1 got you to the leaf switch: the `GPU → NIC → leaf` on-ramp, and the *rail* — the vertical slice of GPU-N-to-leaf-N cabling that keeps a collective's heaviest traffic off the spine entirely. That lesson deliberately stopped there. It told you *where* the fabric begins; it did not tell you what the fabric looks like once you're standing inside it, or what it costs. This lesson builds the tiers above the leaf — leaf-spine, then 3-tier Clos — gives you the one ratio (oversubscription) that determines most of a fabric's bill, and shows you the design move (rail-optimized topology) that lets that ratio be aggressive at the spine without hurting training. It closes by showing you real hyperscalers who did *not* build the textbook 3-tier answer, and why — because the interview-ready version of this material isn't "here's the formula," it's "here's the formula, and here's when a team with real traffic data threw it out."

## Why this matters

The single most expensive procurement lever in a GPU build after the GPUs themselves is fabric bandwidth — switches, optics, and cabling scale super-linearly with the bisection you demand. Argue for full any-to-any bisection across 24K GPUs and you have doubled the network bill for bandwidth LLM training never uses; argue for the *right* oversubscription and you fund another pod of GPUs. This lesson is the math and the vocabulary — Clos tiers, full bisection vs oversubscription ratio, rail-optimized — that lets you sit across from the fabric team and say "1:7 at aggregation is fine here, and here's why," which is exactly the credibility the platform track needs to be trusted with capacity decisions. Increasingly, it's also the credibility to say "1:7 is *not* fine here" — because the ratio that was right for one hyperscaler's traffic pattern in 2024 was already being revised by 2025.

## What's new here (calibration)

- **Already yours (02b, 09.1, 08, 06):** the `GPU → NIC → leaf` on-ramp and the rail concept; ring/tree all-reduce as the traffic pattern communication cost is built around; topology-aware gang placement. We build on top of these without re-deriving them.
- **Genuinely new:** the tiers *above* the leaf — leaf-spine and 3-tier Clos/fat-tree — the formal definitions of full bisection bandwidth and oversubscription, the rail-optimized design that reconciles a non-blocking rail tier with an oversubscribed spine, and — critically — the real production evidence that "3-tier Clos at 1:7" is a design choice tied to a specific traffic pattern and point in time, not a law of physics. We also cover ECMP hash polarization as a distinct failure mode from raw oversubscription ratio, and a same-hyperscaler before/after (Meta's own 1:7 → 1:2.8 revision) that shows the ratio itself is a moving target.
- **Deliberately deferred:** *why* RoCE Ethernet can carry lossless, collective-grade traffic at all (PFC/ECN/DCQCN) is lesson 04's job. Here, RoCE and InfiniBand appear only as line-rate numbers to anchor procurement — the lossless-Ethernet mechanism is intentionally out of scope until 04.

## Core concepts

### Clos / fat-tree: what and why

A **Clos network** (fat-tree is the folded, same-radix Clos used in datacenters) is a multi-tier tree of fixed-radix switches that provides many equal-cost paths between any two endpoints. You use it because you cannot build a single switch with 24,000 400G ports — so you build tiers of small switches and rely on **ECMP** (Equal-Cost Multi-Path routing, or adaptive routing) to spread flows across the many parallel paths. Tiers, bottom-up:

- **Tier 1 — Leaf / ToR (top-of-rack).** Endpoints (GPU NICs) plug in here. In rail-optimized builds each leaf *is* a rail switch (09.1): GPU-N of every node in the group homes to leaf-N.
- **Tier 2 — Spine (within a pod).** Connects all leaves of a pod so any leaf reaches any leaf in one up-one-down hop. A leaf+spine pod is a full **2-tier** fat-tree.
- **Tier 3 — Aggregation / core / super-spine.** Connects *pods* into the full cluster. This is the **3-tier** Clos of a large build.

### Full bisection vs oversubscription — the one ratio that sets cost

Cut the network into two equal halves; the **bisection bandwidth** is the total bandwidth of the links crossing that cut — the worst-case aggregate the two halves can exchange. A tier has **full (non-blocking) bisection** when its *uplink* capacity equals its *downlink* capacity: every endpoint can send at line rate to any other simultaneously with no contention. Concretely, at a leaf with 64×400G downlinks to NICs, full bisection means 64×400G of uplinks to the spine — a **1:1** ratio.

**Oversubscription** is the deliberate opposite: fewer uplinks than downlinks. A **4:1** oversubscribed leaf has 64×400G down but only 16×400G up — four ports of GPU demand contend for one port of uplink. The **oversubscription ratio** = (downlink bandwidth) : (uplink bandwidth).

Why it dominates cost: full bisection means every tier must carry the *sum* of everything below it. Uplinks, spine switch radix, and optics all scale with that sum, and long-reach optics between tiers are a huge fraction of the fabric bill. Halving the uplinks at a tier ~halves that tier's switch-and-optics cost. So the design question is never "full bisection or not" in the abstract — it's "at which tier can the traffic pattern tolerate oversubscription," and that's where rail-optimized design earns its keep.

Worst-case bandwidth is easy to reason about once you have the ratio: a flow **within one leaf** (both endpoints on the same ToR) never touches the uplinks, so it gets full line rate — the oversubscription is irrelevant. A flow that must **cross the oversubscribed tier** shares the reduced uplinks: under X:1 oversubscription with all endpoints active across the cut, worst-case per-endpoint bandwidth through that tier is **line rate ÷ X**.

### Rail-optimized design: oversubscribe the spine for free

Classic clouds can't oversubscribe much because any VM talks to any VM. LLM training is different, and 09.1 already gave you why:

1. **Traffic is rail-local.** In data/tensor/pipeline-parallel training, the heavy collectives (all-reduce, all-gather, reduce-scatter) run **GPU-N ↔ GPU-N** — every participant is on the *same rail*, so its bytes ride *one leaf* and terminate there. Rail-local traffic never climbs to the spine at all.
2. **NVLink absorbs cross-rail.** The only traffic that *would* cross rails (GPU3 wanting GPU5) is shuffled sideways over NVLink inside the node first, so it too leaves on its own rail. The spine sees only the residue.

Therefore the **rail (leaf) tier must be non-blocking — 1:1** — because that's where the collectives live, but the **spine/aggregation tier can be heavily oversubscribed** with negligible impact on training throughput. You are not sacrificing performance to save money; the traffic pattern genuinely doesn't use the spine bandwidth you'd otherwise buy. That is what "oversubscribe the spine for free" means: rail-local + NVLink means the spine carries only rare cross-pod / cross-job / storage / checkpoint traffic, so a heavily oversubscribed spine costs training nothing there while saving a fortune in core switches and long-reach optics.

### Grounding it: the Llama-3 24K-GPU cluster (§3.3.1)

Meta trained Llama 3 on (among others) a **24,576-GPU H100 cluster on a RoCEv2 (Ethernet) fabric** — the module's anchor build. Its three Clos tiers:

- **Node / rack.** 8 GPUs/server, **16 GPUs per rack** (two servers). Intra-node is NVLink/NVSwitch (~900 GB/s/GPU, 4th-gen NVLink), *not* fabric — 02b/09.1 territory. Each GPU has a 400G RoCE NIC.
- **Tier 1 — ToR (leaf).** Each rack's ToR (a Minipack2 OCP switch) aggregates its 16 GPUs.
- **Tier 2 — Cluster/pod switches (spine).** **192 racks form one pod = 3,072 GPUs**, wired at **full bisection bandwidth** — the pod is **non-blocking (1:1)**. Any GPU in the pod reaches any other at line rate. This is the rail-local domain: keep a training job inside a pod and its collectives never contend.
- **Tier 3 — Aggregation.** **8 pods** connect at the aggregation layer to form the 24,576-GPU cluster — but this tier is (as published) **1:7 oversubscribed**, not full bisection. Cross-pod bandwidth is one-seventh of what full bisection would provide.

The design decision in one sentence: **non-blocking inside the 3,072-GPU pod (where the collectives run), 1:7 oversubscribed between pods (where they rarely need to go)** — and Meta reported this cost them little because they *placed jobs to respect pod locality*, keeping heavy communication rail-local and intra-pod. This is 09.1's rule and rail-optimized design operating at 24K scale.

> **The 1:7 ratio is a snapshot, not a law.** The number above is real, published, and specific to Llama 3's 2024 cluster and its traffic pattern — treat it as a well-sourced example, never a constant to memorize as "the" oversubscription ratio. Meta's own engineering team has since described tightening it: their August 2024 engineering post on RoCE networks for distributed AI training describes reducing the **cross-AI-Zone oversubscription ratio from 1:7 to 1:2.8** on newer infrastructure, to give multi-dimensional-parallelism training more cross-domain bandwidth as world sizes and parallelism strategies grew (see References). The lesson isn't "1:2.8 is now correct" either — it's that the safe ratio moves with the traffic pattern, the parallelism plan, and how much in-network reduction (SHARP, lesson 05) offloads work from the wire. Quote 1:7 as the Llama-3-era number, always with the year attached, and be ready to say why a newer build might not use it.

### Not everyone builds 3-tier Clos: Alibaba's 2-tier dual-plane counter-example

The Llama-3 build is a real, working 3-tier Clos — but it is not the only architecture a hyperscaler has shipped for LLM training at comparable scale, and knowing the counter-example is what separates "I memorized a diagram" from "I understand the design space." Alibaba's **HPN** (published at SIGCOMM 2024) interconnects **15,000 GPUs in a single pod** using a **2-tier, dual-plane** architecture — fewer tiers than Llama-3's design, for a pod of comparable order of magnitude.

The reason is a specific, named failure mode, not a cost cut: LLM training produces **few, large, bursty, periodic flows** — a host can generate a single elephant flow near 400 Gbps that turns on and off in sync with the collective's rhythm. Standard ECMP hashing was built for many small, independent flows; when traffic looks like a handful of huge synchronized flows instead, ECMP's hash function can send a disproportionate share of them down the *same* physical path — **hash polarization** — leaving other equal-cost paths idle while the chosen one saturates. That is a distinct failure mode from raw oversubscription: even a fabric with *plenty* of aggregate uplink capacity can bottleneck badly if ECMP keeps picking the same few links for the big flows. Alibaba's 2-tier dual-plane design reduces the number of hops (and therefore the number of times a flow gets re-hashed), shrinks the ECMP search space so the network can more deterministically steer elephant flows onto paths sized to hold them, and runs a dual-ToR design per rack specifically to avoid a single point of failure that a 2-tier design would otherwise introduce.

The takeaway for the design-space question an interviewer might ask ("why not always 3-tier Clos?"): **tier count and oversubscription ratio are traffic-pattern-driven engineering choices, not universal constants** — Alibaba went to *fewer* tiers to fight a routing failure mode that a naive reading of "more tiers = more scale" wouldn't predict.

### The generation numbers to anchor procurement

When you argue fabric, quote current silicon (all figures below are 2024–2026-era; treat as dated snapshots that will move):

| Component | Generation | Line rate |
|---|---|---|
| InfiniBand switch | **Quantum-2** | NDR **400G** |
| InfiniBand switch | **Quantum-X800** | XDR **800G** |
| Ethernet (RoCE) fabric | **Spectrum-X** | RoCE-optimized Ethernet, 400/800G |
| NIC | **ConnectX-7** | 400G |
| NIC | **ConnectX-8** | 800G |
| DPU | **BlueField-3** | offload/RoCE congestion control, isolation |

Two fabric camps: **InfiniBand** (Quantum-2/X800 + ConnectX) — lossless by design, credit-based flow control, the traditional HPC/AI default; and **RoCE Ethernet** (Spectrum-X + ConnectX/BlueField) — Ethernet economics and operability made loss-tolerant enough for collectives via PFC/ECN and Spectrum-X's adaptive routing + congestion control. Meta's Llama-3 24K cluster is the proof that a well-built **RoCE** fabric trains frontier models; InfiniBand remains common where the fabric team wants lossless guarantees out of the box. BlueField-3 DPUs sit at the edge to offload congestion control and enforce tenant isolation — increasingly the multi-tenant neocloud default. (Why RoCE needs lossless Ethernet at all — PFC, ECN, DCQCN — is lesson 04's material; here you only need the line-rate numbers.)

**One dated, third-party cost data point worth having, flagged as such:** independent 2025 TCO analysis (not a vendor figure) put a 512-GPU cluster's 3-year network TCO at roughly **$4.6M for InfiniBand vs $2.4M for Ethernet/RoCE** — a gap large enough to fund meaningfully more GPUs, which is the shape of the argument (not the exact numbers, which will drift) worth internalizing for a procurement conversation. Treat this specific figure as a 2025 snapshot from a single third-party source, not a vendor-published or peer-reviewed number.

## Perspectives

**Theory.** The Clos/bisection-bandwidth math is topology-agnostic — it's graph theory, the same whether you're wiring a 24K-GPU AI cluster or a 1990s telephone switch. Bisection bandwidth, full vs. oversubscribed, ECMP path-spreading — these concepts don't change; what changes across contexts is *which tier you're willing to oversubscribe*, and that depends entirely on the traffic pattern riding on top.

**Practice.** Hyperscalers deviate from the textbook 3-tier answer the moment their own traffic telemetry justifies it. Alibaba didn't go 2-tier dual-plane to save money — the same paper is explicit that the driver was a routing failure mode (ECMP hash polarization) specific to LLM training's bursty flow shape, not the cost argument that usually motivates fewer tiers. Real fabric teams reason from measured traffic, not from "what does the reference architecture diagram say."

**Failure mode.** Rail-optimized design's entire safety case rests on one assumption: *training traffic is rail-local*. That assumption breaks the moment a workload's parallelism plan doesn't respect pod (or rail) boundaries — a cross-pod checkpoint write, a cross-pod evaluation job, or a parallelism strategy that was tuned for a smaller pod and now spans two. When that happens, "acceptable" oversubscription at the spine stops being acceptable, because now real bandwidth-hungry traffic is crossing the tier that was sized assuming it wouldn't have to.

## Real-world use cases

- **Alibaba HPN (SIGCOMM 2024).** [ACM: Alibaba HPN — A Data Center Network for Large Language Model Training](https://dl.acm.org/doi/10.1145/3651890.3672265) (PDF mirror: [Stanford-hosted copy](https://cs.stanford.edu/~keithw/sigcomm2024/sigcomm24-final878-acmpaginated.pdf)) — a named hyperscaler choosing a 2-tier dual-plane architecture over 3-tier Clos, specifically to avoid ECMP hash polarization from LLM's bursty ~400 Gbps/host flow pattern, at 15,000 GPUs in one pod.
- **ByteDance MegaScale (NSDI '24).** [USENIX: MegaScale — Scaling LLM Training to More Than 10,000 GPUs](https://www.usenix.org/conference/nsdi24/presentation/jiang-ziheng) — a production 3-layer rail-optimized Clos, independently validating the design family Llama-3 uses, with measured 55.2% model-FLOPs utilization at 12,288 GPUs.
- **SemiAnalysis, "100,000 H100 Clusters."** [SemiAnalysis: 100,000 H100 Clusters — Power, Network Topology, Ethernet vs InfiniBand](https://newsletter.semianalysis.com/p/100000-h100-clusters-power-network) — a concrete, named alternative topology at 100K-GPU scale: four ~24,576-GPU non-blocking domains ("pods" in this design's own terminology, larger than Meta's 3,072-GPU pod) joined at 1:7 oversubscription, with detailed power and reliability tradeoffs. Note the terminology overload versus Meta's build — "pod" means a different GPU count in each source; always check which non-blocking unit a source means before quoting its ratio. (Third-party analysis, 2024 snapshot — flag accordingly.)
- **Meta, Llama 3 paper §3.3.1.** [arXiv: The Llama 3 Herd of Models](https://arxiv.org/abs/2407.21783) — the module's anchor: a real, published 24,576-GPU RoCEv2 3-tier Clos, 3,072-GPU non-blocking pods, 1:7 aggregation oversubscription. Flagged above as a dated, Llama-3-specific design snapshot rather than a universal ratio.

## Worked example

**Read the Llama-3 fabric and compute the ratios.**

*Pod (Tier 2), full bisection:* 3,072 GPUs, each a 400G NIC → 3,072 × 400G = ~1.23 Pb/s of endpoint demand. Non-blocking means the ToR+spine of the pod provides matching uplink capacity — downlink:uplink = **1:1**. Any all-reduce among the pod's GPUs runs at line rate; nothing contends. This is affordable *per pod* because 3,072 GPUs is a bounded fan-in a spine tier can fully serve.

*Cluster (Tier 3), oversubscribed:* 8 pods × 3,072 = 24,576 GPUs. If the aggregation tier were full bisection it would carry 8 × 1.23 Pb/s = ~9.8 Pb/s across the core — enormous switch radix and long-reach optics. Instead it's built **1:7**: for every 7 units of intra-pod (downlink) capacity, 1 unit climbs to aggregation. So cross-pod bandwidth per GPU is ~**400G ÷ 7 ≈ 57G** worst-case, versus 400G intra-pod. Training tolerates this because jobs are placed pod-local: the collectives that need 400G stay inside the non-blocking pod, and only rare cross-pod traffic pays the 1:7 tax.

*Why non-blocking pod + oversubscribed cluster is the right split (for this workload, at this snapshot in time):* the collective's communicator is sized to fit a pod, so 100% of its bandwidth-critical traffic is intra-pod line-rate; the spine only ever sees the residue. Buying full bisection at Tier 3 would 7× the core fabric cost to serve bandwidth the training pattern doesn't consume. That saved capital funds more GPUs — the FinOps argument in one line.

*Now recompute for the reported 1:2.8 revision:* if the cross-domain ratio tightens to 1:2.8, worst-case cross-pod bandwidth per GPU becomes 400G ÷ 2.8 ≈ **143G** — roughly 2.5× the 1:7 figure. That buys real headroom for multi-dimensional parallelism strategies (or evaluation/checkpoint jobs) that cross pod boundaries more often than Llama-3's original placement assumed, at proportionally higher aggregation-tier cost. The arithmetic is the same formula in both cases — only the ratio, and therefore the bill, changed.

## Practice

Feeds the deliverable **Network architecture read**.

**Task.** From Llama-3 §3.3.1's network description, **sketch the 3-tier Clos** and annotate it — then contrast it with Alibaba HPN's 2-tier design.

**Requirements / acceptance:**
1. A **3-tier topology sketch** (hand-drawn or ASCII/diagram): GPUs/rack → ToR (leaf) → pod spine → aggregation, with the real counts — 16 GPUs/rack, 192 racks = 3,072-GPU pod, 8 pods = 24,576 GPUs.
2. **Label the bisection/oversubscription at each tier:** intra-node = NVLink (not fabric); leaf/pod (Tier 1–2) = **full bisection, 1:1, non-blocking**; aggregation (Tier 3) = **1:7 oversubscribed** (labeled as the 2024 Llama-3 snapshot).
3. One short paragraph computing *why* the 3,072-GPU pod is non-blocking (uplink capacity = downlink capacity, so line-rate any-to-any) while the 24K cluster is 1:7 (aggregation uplinks = 1/7 of intra-pod capacity), and *why that's acceptable* (rail-local + pod-local placement keeps heavy collectives off the spine).
4. Two to three sentences naming Alibaba HPN's 2-tier dual-plane alternative, the ECMP hash-polarization failure mode it targets, and why "fewer tiers" was the fix rather than "more oversubscription."

Combine with the 09.1 GPU→rail→NIC table and you have the full intra-plus-inter network-architecture read for the deliverable.

## Common pitfalls

- **Treating 1:7 as a universal constant.** It's a Llama-3-specific, 2024-dated design choice, not a rule. Meta's own later material describes tightening it to roughly 1:2.8 on newer infrastructure. Always attach the year and the source cluster when you quote an oversubscription ratio.
- **Assuming 3-tier Clos is always the right shape.** Alibaba's 2-tier dual-plane HPN interconnects 15K GPUs in one pod — fewer tiers than Llama-3's design — because the traffic pattern (few, huge, bursty flows) and a specific routing failure mode (ECMP hash polarization) made fewer tiers the better fix, not a compromise.
- **Computing bisection bandwidth only for the happy, rail-local path.** The oversubscription math above assumes jobs stay pod-local. The moment a job's placement or communicator spans pod boundaries — a cross-pod evaluation run, a checkpoint write to shared storage outside the pod — the worst-case number is the *oversubscribed-tier* number, not the non-blocking one. Always state which case ("rail-local" vs "crosses the oversubscribed tier") a bandwidth claim assumes.
- **Ignoring ECMP hash polarization as a distinct failure mode from the oversubscription ratio.** A fabric can have generous aggregate uplink capacity and still bottleneck badly if ECMP keeps sending large synchronized flows down the same physical path. Oversubscription ratio tells you how much capacity exists; hash polarization tells you whether that capacity is actually being used evenly. They are different questions and a good design answers both.

## Self-check

**(a) A 4:1-oversubscribed spine — what's the worst-case per-GPU bandwidth for an all-to-all spanning two leaves vs within one leaf?**
**Answer:** Within one leaf the traffic never touches the uplinks, so it runs at full line rate (e.g. 400G) regardless of the oversubscription. Spanning two leaves, the flows must cross the 4:1 tier and share the reduced uplinks: worst-case per-GPU is line rate ÷ 4 ≈ **100G**. The oversubscription ratio only bites traffic that actually crosses the oversubscribed tier — which is exactly why keeping collectives leaf/rail-local makes the ratio harmless.

**(b) Why can rail-optimized designs oversubscribe the spine "for free" for LLM training?**
**Answer:** Because LLM training traffic is rail-local — the heavy collectives run GPU-N ↔ GPU-N, so they ride a single leaf and never climb to the spine — and NVLink absorbs the only cross-rail traffic by shuffling it sideways inside the node before it leaves. The spine therefore carries only rare cross-pod/cross-job/storage traffic. Oversubscribing it (e.g. Llama-3's published 1:7) removes bandwidth the training pattern never uses, so throughput is unaffected while core switch and optics cost drops sharply.

**(c) What does "full bisection bandwidth" mean and why is it expensive?**
**Answer:** Cut the network into two equal halves; the bisection bandwidth is the aggregate capacity of the links crossing the cut — the worst-case a tier can carry between its halves. *Full* (non-blocking) bisection means uplink capacity = downlink capacity at every tier, so every endpoint can send to any other at line rate with no contention (a 1:1 ratio). It's expensive because each tier must carry the *sum* of everything below it: uplink count, spine/core switch radix, and — dominant at scale — long-reach optics all scale with that sum. Full bisection across 24K GPUs at Tier 3 would ~7× the core fabric cost versus Meta's published 1:7 build, for bandwidth rail-local training never uses.

**(d) Why did Alibaba build a 2-tier dual-plane fabric instead of a 3-tier Clos for a 15K-GPU pod?**
**Answer:** LLM training's traffic pattern is a small number of large, bursty, periodic flows (~400 Gbps/host), and standard ECMP hashing — built for many small independent flows — can send a disproportionate share of those big flows down the same physical path, a failure mode called hash polarization, leaving other equal-cost paths idle. Alibaba's 2-tier dual-plane design reduces hop count (and re-hashing events), shrinks the ECMP search space so elephant flows can be steered onto paths sized to hold them more deterministically, and uses dual-ToR per rack to avoid a single point of failure. It's a routing-correctness fix, not primarily a cost cut — the counter-example to "more tiers always scales better."

**(e) If Meta's public 1:7 ratio has since moved toward roughly 1:2.8 on newer infrastructure, what does that imply for how you should use the 1:7 number?**
**Answer:** It implies the safe oversubscription ratio isn't a fixed constant — it shifts as traffic patterns, world sizes, parallelism strategies, and how much reduction happens in-network (SHARP) change over time. 1:7 was the right, well-sourced answer for the Llama-3-era traffic pattern and cluster size; quoting it as "the" industry ratio without a year attached overstates its generality. The correct interview answer names 1:7 as a real, specific data point and immediately adds that the ratio is a design variable tuned per traffic pattern, not a universal law — citing Meta's own reported move to ~1:2.8 is exactly the kind of evidence that makes that claim credible rather than hand-wavy.

## Connections & what's next

This lesson turns 09.1's rail into a cost argument: the same rail-locality that told you which NIC to use now tells you which fabric tier can be oversubscribed and by how much. It also sets up lesson 04 (IB vs RoCEv2 + lossless), the module's highest-yield lesson — the InfiniBand/RoCE line-rate numbers you saw here in a table become a real procurement argument once 04 explains *why* RoCE needs PFC/ECN/DCQCN to be lossless enough for collectives at all. And it sets up lesson 05 (GPUDirect + SHARP), where in-network reduction changes how much traffic a collective puts on the wire in the first place — one of the levers that shifts the "right" oversubscription ratio discussed above.

The next lesson, **09.3 (RDMA fundamentals)**, changes altitude one more time: instead of asking "what shape is the network," it asks "how does a byte actually get from one GPU's memory to another's without the CPU in the way." You've been assuming GPUDirect RDMA works throughout 09.1 and this lesson — 09.3 opens that assumption up, tracing exactly which stages of the ordinary kernel networking datapath (syscalls, socket-buffer copies, conntrack, softirq) RDMA deletes, and why deleting the *stateful, shared* stages — not just going faster on average — is what keeps a barrier-synchronous collective's tail latency flat under load. Carry the Clos/oversubscription vocabulary forward; 09.3 explains the mechanism that makes the rail-local, non-blocking tier you just sized actually deliver the bandwidth you computed.

## References & further reading

**Primary sources**
- Meta, [The Llama 3 Herd of Models, §3.3.1 "Network"](https://arxiv.org/abs/2407.21783) — read for: the anchor build — 24,576-GPU RoCEv2 cluster, 3,072-GPU non-blocking pods, 8 pods at 1:7 aggregation oversubscription.
- Meta Engineering, [RoCE networks for distributed AI training at scale](https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/) (Aug 2024) — read for: Meta's own reported tightening of cross-domain oversubscription from 1:7 to roughly 1:2.8 on newer infrastructure — the primary evidence that 1:7 is a snapshot, not a constant. (Fetched via search summary; if the direct link is unreachable from your network, the canonical URL is still the one to cite.)
- Wang, Ghobadi et al., [Rail-only: A Low-Cost High-Performance Network for Training LLMs with Trillion Parameters](https://arxiv.org/abs/2307.12169) (HotNets 2023 / HotI 2024) — read for: the academic backbone of the rail-optimized argument — LLM traffic is rail-local, so a rail-only fabric with a slimmed spine matches full-bisection performance at a fraction of the cost.
- NVIDIA, [HGX AI Factory Reference Architecture](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/) — read for: the generation numbers to quote in procurement (Quantum-2/X800, Spectrum-X, ConnectX-7/8, BlueField-3) and a vendor-blessed leaf-spine fabric diagram.

**Real-world engineering blogs**
- Alibaba, via ACM SIGCOMM 2024, [HPN: A Data Center Network for Large Language Model Training](https://dl.acm.org/doi/10.1145/3651890.3672265) (PDF mirror: [Stanford-hosted copy](https://cs.stanford.edu/~keithw/sigcomm2024/sigcomm24-final878-acmpaginated.pdf)) — what it shows: a named hyperscaler choosing 2-tier dual-plane over 3-tier Clos specifically to dodge ECMP hash polarization at 15K GPUs/pod.
- ByteDance, via USENIX NSDI '24, [MegaScale: Scaling LLM Training to More Than 10,000 GPUs](https://www.usenix.org/conference/nsdi24/presentation/jiang-ziheng) — what it shows: a production 3-layer rail-optimized Clos at 12,288+ GPUs, 55.2% measured MFU.
- SemiAnalysis, [100,000 H100 Clusters: Power, Network Topology, Ethernet vs InfiniBand](https://newsletter.semianalysis.com/p/100000-h100-clusters-power-network) — what it shows: a concrete named alternative topology (four ~24,576-GPU non-blocking domains at 1:7) with power/reliability tradeoffs at 100K-GPU scale. Dated 2024 third-party analysis, not a vendor primary source — flag accordingly when citing its specific numbers.
- Vitex, [InfiniBand vs Ethernet for AI Clusters](https://www.vitextech.com/blogs/blog/infiniband-vs-ethernet-for-ai-clusters-effective-gpu-networks-in-2025) — what it shows: an independent 2025 TCO comparison (~$4.6M IB vs ~$2.4M Ethernet at 512 GPUs over 3 years) — a dated, third-party snapshot useful for the shape of the cost argument, not as a vendor-verified figure.

**Deeper dives**
- Glenn K. Lockwood, ["Networking for LLM training" — practitioner notes](https://www.glennklockwood.com/garden/networking-for-LLM-training) — an independent, critical (non-vendor) breakdown of why LLM training traffic doesn't require a fully non-blocking fat tree, and what topologies are defensible instead.
- At Scale Conference, [talk recording: "Alibaba HPN — A Data Center Network for Large Language Model Training"](https://atscaleconference.com/videos/alibaba-hpn-a-data-center-network-for-large-language-model-training/) — the HPN authors presenting the 2-tier dual-plane design and the ECMP hash-polarization problem directly, useful for hearing the tradeoffs in their own words.
