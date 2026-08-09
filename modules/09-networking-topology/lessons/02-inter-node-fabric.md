---
lesson: "09.2"
title: "The inter-node fabric: Clos, bisection, and rail-optimized oversubscription"
module: "09"
concept: "The inter-node fabric: Clos, bisection, and rail-optimized oversubscription"
status: not-started
est_time: "6h"
artifacts: []
---

# 09.2 · The inter-node fabric: Clos, bisection, and rail-optimized oversubscription

> **Concept.** A GPU cluster's fabric is a Clos/fat-tree, and its cost is set by one dial — how much bisection bandwidth you buy at each tier; rail-optimized design lets you keep that dial low at the spine *for free* because LLM traffic is rail-local and NVLink absorbs the rest.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Why this matters

The single most expensive procurement lever in a GPU build after the GPUs themselves is fabric bandwidth — switches, optics, and cabling scale super-linearly with the bisection you demand. Argue for full any-to-any bisection across 24K GPUs and you have doubled the network bill for bandwidth LLM training never uses; argue for the *right* oversubscription and you fund another pod of GPUs. This lesson is the math and the vocabulary — Clos tiers, full bisection vs oversubscription ratio, rail-optimized — that lets you sit across from the fabric team and say "1:7 at aggregation is fine here, and here's why," which is exactly the credibility the platform track needs to be trusted with capacity decisions.

## What's new here

**02b** and **09.1** got you to the leaf switch: the `GPU → NIC → leaf` on-ramp and the rail. **08** told you the traffic pattern — ring/tree all-reduce, communication as the bottleneck. **06** placed gangs topology-aware. None of them described the *shape of the network above the leaf* or its *cost*.

This lesson builds the tiers above the leaf: **leaf-spine (2-tier) and 3-tier Clos/fat-tree**, the definition of **full bisection bandwidth** and its opposite, **oversubscription**, and the design that reconciles them for AI — **rail-optimized topology**. The key inversion versus classic datacenter networking: general cloud fabrics chase near-full bisection because traffic is any-to-any; LLM training traffic is *rail-local* (09.1), so you can deliberately oversubscribe the spine and lose almost nothing. We ground every number in Meta's Llama-3 24K-GPU RoCE cluster (§3.3.1) — a published, real, non-blocking-pod / oversubscribed-cluster design you can reproduce as a sketch.

## Core notes

### Clos / fat-tree: what and why

A **Clos network** (fat-tree is the folded, same-radix Clos used in datacenters) is a multi-tier tree of fixed-radix switches that provides many equal-cost paths between any two endpoints. You use it because you cannot build a single switch with 24,000 400G ports — so you build tiers of small switches and rely on **ECMP** (or adaptive routing) to spread flows across the many parallel paths. Tiers, bottom-up:

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

Therefore the **rail (leaf) tier must be non-blocking — 1:1** — because that's where the collectives live, but the **spine/aggregation tier can be heavily oversubscribed** with negligible impact on training throughput. You are not sacrificing performance to save money; the traffic pattern genuinely doesn't use the spine bandwidth you'd otherwise buy. That is what "oversubscribe the spine for free" means: rail-local + NVLink means the spine carries only rare cross-pod / cross-job / storage / checkpoint traffic, so 1:7 there costs training nothing while saving a fortune in core switches and long-reach optics.

### Grounding it: the Llama-3 24K-GPU cluster (§3.3.1)

Meta trained Llama 3 on (among others) a **24,576-GPU H100 cluster on a RoCEv2 (Ethernet) fabric** — the anchor build. Its three Clos tiers:

- **Node / rack.** 8 GPUs/server, **16 GPUs per rack** (two servers). Intra-node is NVLink/NVSwitch (900 GB/s/GPU, 4th-gen NVLink), *not* fabric — 02b territory. Each GPU has a 400G RoCE NIC (ConnectX-class).
- **Tier 1 — ToR (leaf).** Each rack's ToR (a Minipack2 OCP switch) aggregates its 16 GPUs.
- **Tier 2 — Cluster/pod switches (spine).** **192 racks form one pod = 3,072 GPUs**, wired at **full bisection bandwidth** — the pod is **non-blocking (1:1)**. Any GPU in the pod reaches any other at line rate. This is the rail-local domain: keep a training job inside a pod and its collectives never contend.
- **Tier 3 — Aggregation.** **8 pods** connect at the aggregation layer to form the 24,576-GPU cluster — but this tier is **1:7 oversubscribed**, not full bisection. Cross-pod bandwidth is one-seventh of what full bisection would provide.

The design decision in one sentence: **non-blocking inside the 3,072-GPU pod (where the collectives run), 1:7 oversubscribed between pods (where they rarely need to go)** — and Meta reports this cost them little because they *placed jobs to respect pod locality*, keeping heavy communication rail-local and intra-pod. This is 09.1's rule and rail-optimized design operating at 24K scale.

### The generation numbers to anchor procurement

When you argue fabric, quote current silicon:

| Component | Generation | Line rate |
|---|---|---|
| InfiniBand switch | **Quantum-2** | NDR **400G** |
| InfiniBand switch | **Quantum-X800** | XDR **800G** |
| Ethernet (RoCE) fabric | **Spectrum-X** | RoCE-optimized Ethernet, 400/800G |
| NIC | **ConnectX-7** | 400G |
| NIC | **ConnectX-8** | 800G |
| DPU | **BlueField-3** | offload/RoCE congestion control, isolation |

Two fabric camps: **InfiniBand** (Quantum-2/X800 + ConnectX) — lossless by design, credit-based flow control, the traditional HPC/AI default; and **RoCE Ethernet** (Spectrum-X + ConnectX/BlueField) — Ethernet economics and operability made loss-tolerant enough for collectives via PFC/ECN and Spectrum-X's adaptive routing + congestion control. Meta's Llama-3 24K cluster is the proof that a well-built **RoCE** fabric trains frontier models; InfiniBand remains common where the fabric team wants lossless guarantees out of the box. BlueField-3 DPUs sit at the edge to offload congestion control and enforce tenant isolation — increasingly the multi-tenant neocloud default.

## Worked example

**Read the Llama-3 fabric and compute the ratios.**

*Pod (Tier 2), full bisection:* 3,072 GPUs, each a 400G NIC → 3,072 × 400G = ~1.23 Pb/s of endpoint demand. Non-blocking means the ToR+spine of the pod provides matching uplink capacity — downlink:uplink = **1:1**. Any all-reduce among the pod's GPUs runs at line rate; nothing contends. This is affordable *per pod* because 3,072 GPUs is a bounded fan-in a spine tier can fully serve.

*Cluster (Tier 3), oversubscribed:* 8 pods × 3,072 = 24,576 GPUs. If the aggregation tier were full bisection it would carry 8 × 1.23 Pb/s = ~9.8 Pb/s across the core — enormous switch radix and long-reach optics. Instead it's built **1:7**: for every 7 units of intra-pod (downlink) capacity, 1 unit climbs to aggregation. So cross-pod bandwidth per GPU is ~**400G ÷ 7 ≈ 57G** worst-case, versus 400G intra-pod. Training tolerates this because jobs are placed pod-local: the collectives that need 400G stay inside the non-blocking pod, and only rare cross-pod traffic pays the 1:7 tax.

*Why non-blocking pod + oversubscribed cluster is the right split:* the collective's communicator is sized to fit a pod, so 100% of its bandwidth-critical traffic is intra-pod line-rate; the spine only ever sees the residue. Buying full bisection at Tier 3 would 7× the core fabric cost to serve bandwidth the training pattern doesn't consume. That saved capital funds more GPUs — the FinOps argument in one line.

## Practice

Feeds the deliverable **Network architecture read**.

**Task.** From Llama-3 §3.3.1's network description, **sketch the 3-tier Clos** and annotate it.

**Requirements / acceptance:**
1. A **3-tier topology sketch** (hand-drawn or ASCII/diagram): GPUs/rack → ToR (leaf) → pod spine → aggregation, with the real counts — 16 GPUs/rack, 192 racks = 3,072-GPU pod, 8 pods = 24,576 GPUs.
2. **Label the bisection/oversubscription at each tier:** intra-node = NVLink (not fabric); leaf/pod (Tier 1–2) = **full bisection, 1:1, non-blocking**; aggregation (Tier 3) = **1:7 oversubscribed**.
3. One short paragraph computing *why* the 3,072-GPU pod is non-blocking (uplink capacity = downlink capacity, so line-rate any-to-any) while the 24K cluster is 1:7 (aggregation uplinks = 1/7 of intra-pod capacity), and *why that's acceptable* (rail-local + pod-local placement keeps heavy collectives off the spine).

Combine with the 09.1 GPU→rail→NIC table and you have the full intra-plus-inter network-architecture read for the deliverable.

## Self-check

**(a) A 4:1-oversubscribed spine — what's the worst-case per-GPU bandwidth for an all-to-all spanning two leaves vs within one leaf?**
**Answer:** Within one leaf the traffic never touches the uplinks, so it runs at full line rate (e.g. 400G) regardless of the oversubscription. Spanning two leaves, the flows must cross the 4:1 tier and share the reduced uplinks: worst-case per-GPU is line rate ÷ 4 ≈ **100G**. The oversubscription ratio only bites traffic that actually crosses the oversubscribed tier — which is exactly why keeping collectives leaf/rail-local makes the ratio harmless.

**(b) Why can rail-optimized designs oversubscribe the spine "for free" for LLM training?**
**Answer:** Because LLM training traffic is rail-local — the heavy collectives run GPU-N ↔ GPU-N, so they ride a single leaf and never climb to the spine — and NVLink absorbs the only cross-rail traffic by shuffling it sideways inside the node before it leaves. The spine therefore carries only rare cross-pod/cross-job/storage traffic. Oversubscribing it (e.g. 1:7) removes bandwidth the training pattern never uses, so throughput is unaffected while core switch and optics cost drops sharply. Meta's Llama-3 build is the proof: non-blocking 3,072-GPU pods, 1:7 between pods.

**(c) What does "full bisection bandwidth" mean and why is it expensive?**
**Answer:** Cut the network into two equal halves; the bisection bandwidth is the aggregate capacity of the links crossing the cut — the worst-case a tier can carry between its halves. *Full* (non-blocking) bisection means uplink capacity = downlink capacity at every tier, so every endpoint can send to any other at line rate with no contention (a 1:1 ratio). It's expensive because each tier must carry the *sum* of everything below it: uplink count, spine/core switch radix, and — dominant at scale — long-reach optics all scale with that sum. Full bisection across 24K GPUs at Tier 3 would ~7× the core fabric cost versus a 1:7 build, for bandwidth rail-local training never uses.

## Resources

1. **Llama 3 paper, §3.3.1 "Network"** — https://arxiv.org/abs/2407.21783 — *deep, the anchor.* The 24,576-GPU RoCE cluster: 16 GPUs/rack, 3,072-GPU non-blocking pods, 8 pods at 1:7 aggregation oversubscription, plus their congestion-control and load-balancing notes. Read §3.3.1 closely — every number in this lesson traces to it, and your sketch reproduces its topology.
2. **Ghobadi et al., "How to Build Low-Cost Networks for Large Language Models"** (HotNets 2023) — https://people.csail.mit.edu/ghobadi/papers/rail_llm_hotnets_2023.pdf — *deep.* The rail-optimized argument stated cleanly: LLM traffic is rail-local, so a rail-only fabric with a slimmed/oversubscribed spine matches full-bisection performance at a fraction of the cost. This is the academic backbone of the "oversubscribe the spine for free" claim you'll make in an interview.
3. **NVIDIA HGX AI Factory Reference Architecture** — https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/ — *skim.* Vendor-blessed leaf-spine fabric with current silicon (Quantum-2/X800, Spectrum-X, ConnectX-7/8, BlueField-3) — the generation numbers and cabling to quote in procurement.
