---
lesson: "09.7"
title: "Bandwidth as a cost input"
module: "09"
concept: "Bandwidth as a cost input"
status: not-started
est_time: "5h"
artifacts: []
---

# 09.7 · Bandwidth as a cost input

> **Concept.** Turn topology into dollars — oversubscription as a capex lever, the IB premium vs RoCE reuse, SHARP as byte-reduction — and convert a placement choice into a bandwidth-and-cost statement.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Why this matters

The back-end fabric is one of the largest capex lines in a GPU cluster after the accelerators themselves — switches, transceivers, cables, and the DC power/space they draw. On a rail-optimized H100/H200 build the network BOM routinely lands at **10–20% of total system cost**, and almost all of that swing lives in one knob: **how much bisection bandwidth you buy at the spine tier**. Full non-blocking bisection at cluster scale is the most expensive fabric you can build; a 1:7 oversubscribed spine can cut the spine-tier switch and optics count by roughly 7×.

This is your FinOps edge applied to silicon and glass. The lessons before this one (02b, 08) taught you the *physics* of the fabric; lesson 06 taught you the *mechanism* to place jobs on it. This lesson is the **argument**: the ability to walk into a procurement review and say "this workload's traffic shape tolerates a 1:4 spine, saving $X on optics and switches, and here is the placement policy that guarantees the collective never pays the oversubscription tax." A senior platform engineer at a neocloud is measured on exactly that sentence — turning a placement decision into a bandwidth number and a dollar number. Anyone can quote a switch datasheet. The signal is converting **$/GB/s of bisection** into a defensible fabric tier for a named workload.

## What's new here

Lesson 06 owns the **how**: Kueue TAS, node labels, topology domains — the scheduler machinery that places a job inside a blast radius. This lesson owns the **why, in bandwidth and dollars**. We are not re-teaching how to co-locate a job; we are teaching how to *price* the co-location — what bisection you preserve by staying in one pod, what you forfeit by spreading across an oversubscribed tier, and how that maps to a line on a BOM. Same placement decision, seen from procurement instead of the scheduler.

## Core notes

### 1. $/GB/s of bisection — the lens

Stop pricing the fabric per port and start pricing it per **unit of bisection bandwidth delivered to the collective**. Bisection is the bandwidth across the worst-case cut of the network — the number that actually bounds an all-reduce that spans the whole job. The cost model:

```
fabric_$ / bisection_Gbps  =  ($ switches + $ optics + $ cable + $ power·years) / usable_bisection
```

Two fabrics with identical port counts can differ 5–7× in bisection because one is non-blocking and the other is oversubscribed at the spine. The GPU-facing (leaf) tier is essentially fixed — you need one NIC port per GPU no matter what — so **all the cost-vs-bisection tradeoff lives in the spine tier**. That is the tier of highest-radix switches and longest-reach (single-mode) optics, which is exactly why shrinking it moves the BOM so much.

### 2. Oversubscription is a spine-tier capex lever

In a two/three-tier Clos (fat-tree), oversubscription is set by the **downlink : uplink ratio at the leaf**. Non-blocking (1:1) means every GPU-facing downlink has a matching uplink toward the spine. Cut the uplinks and you cut the spine directly:

- **Full bisection (1:1):** uplinks = downlinks. Spine switch count and spine-facing transceiver count are maximal. Every GPU can drive line-rate to every other GPU simultaneously.
- **1:4:** one uplink per four downlinks → spine tier ≈ **¼** the switches/optics of full bisection.
- **1:7:** spine tier ≈ **⅐**. Cross-spine bandwidth is capped at ~1/7 of per-GPU line rate for any traffic that must traverse it.

The spine tier and its long-reach optics scale **linearly with 1/oversubscription**. The leaf tier and the GPU-facing optics do not move. So oversubscription is a lever on precisely the most expensive, longest-reach part of the fabric — which is why it is the single most cost-sensitive knob in the whole BOM. **This is the same design Meta shipped for Llama 3** (§3.3.1): non-blocking *inside* a 3,072-GPU pod, **1:7 oversubscribed** at the aggregation tier between pods, with the scheduler kept topology-aware so the expensive cross-pod traffic stays rare.

### 3. The IB "tax" vs RoCE "reuse"

The fabric-technology choice is a second cost axis, orthogonal to oversubscription:

| | InfiniBand | RoCEv2 (Ethernet) |
|---|---|---|
| Switch/optics $ | Premium; NVIDIA/Mellanox-dominated supply | Cheaper, multi-vendor, commodity optics |
| Control plane | Dedicated **subnet manager** (SM), separate fabric | Reuses Ethernet/EVPN/BGP — skills you already have |
| Congestion control | Mature, credit-based, in-fabric | PFC/ECN + DCQCN tuning — operationally fiddly |
| In-network compute | **SHARP** (see §4) | No portable equivalent |
| Lock-in | High — vendor + separate ops discipline | Low — standard Ethernet estate |

The **IB premium** buys deterministic congestion behavior and SHARP, at the cost of a dedicated fabric, an SM to run, and vendor lock. The **RoCE reuse** case is that your team already operates Ethernet/EVPN, the optics are cheaper and multi-sourced, and a large fraction of real training traffic can be kept off the contended paths by placement. Meta's Llama 3 RoCE cluster is the existence proof that a frontier run does not *require* IB — it requires topology-aware software (NCCLX + a topology-aware scheduler) to live within an oversubscribed, commodity-Ethernet fabric. The FinOps question is never "IB or RoCE" in the abstract; it is "does this workload's traffic shape and my team's skill estate justify the IB premium, or does RoCE + smart placement deliver the same effective bandwidth for less?"

### 4. SHARP as byte-reduction — an effective-bandwidth multiplier

SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) performs the reduction **inside the switch/NVSwitch** instead of shuffling data around a ring or tree of endpoints. For an all-reduce, a ring collective moves roughly **2×(N−1)/N × message_size** of data per GPU over the wire; SHARP collapses that to each endpoint sending its contribution once and receiving the result once — roughly **halving the bytes on the wire** and removing the multi-hop traversal latency.

Reframe that as cost: SHARP is a **byte-reduction lever**, complementary to the oversubscription capex lever. Fewer bytes for the same reduction means the *same physical fabric delivers more effective all-reduce bandwidth* — so a reduction-heavy trainer can hit its target step time on a **cheaper (more oversubscribed) fabric**, or reach a higher effective bandwidth on the fabric you already bought. The catch: SHARP is part of the **IB premium** (IB switches, or NVLink-SHARP/NVLS inside the NVSwitch domain). RoCE has no portable equivalent. So SHARP is often the specific reason the IB tax pays for itself on a workload dominated by gradient all-reduce.

### 5. Placement → bandwidth conversion (the whole point)

A placement decision *is* a bandwidth statement, once you know the topology:

- **Co-locate the job inside one non-blocking pod** → the collective sees **full per-GPU line-rate bisection**. On a 400 Gb/s (400G) back-end, each GPU can drive ~400 Gb/s into the all-reduce simultaneously. Cost of this bandwidth: **$0 incremental** — you are spending bisection you already bought. The price you pay is an *opportunity cost*: you consume a scarce non-blocking domain (3,072 GPUs in the Llama 3 design) and constrain the scheduler.
- **Spread the same job across a 1:7 aggregation tier** → any part of the collective that crosses the spine is capped at **~1/7 of line rate**, ~57 Gb/s per GPU on a 400G fabric. A comms-bound all-reduce then slows in proportion to that cut.

So the scheduler policy from lesson 06 is not a scheduling nicety — it is what lets an **oversubscribed-fabric capex deliver full-bisection performance** for the jobs that matter. Co-location converts a cheap fabric into an expensive-fabric experience *for traffic that stays inside the blast radius*. That single sentence is the procurement throughline: buy the oversubscribed spine, then use placement to make sure your bandwidth-hungry collectives never touch it.

### 6. Scale-up vs scale-out — two very different $/GB/s

The single most common procurement error is treating "bandwidth" as one number. There are two fabrics with an order-of-magnitude different price per GB/s:

- **Scale-up (intra-node NVLink / NVSwitch domain):** ~900 GB/s per GPU on H100, ~1.8 TB/s on the NVLink5 generation. This bandwidth is *bundled into the server SKU* — you do not buy it per-port at the spine. Its $/GB/s is extraordinarily low, and NVLink-SHARP/NVLS runs the reduction inside the NVSwitch.
- **Scale-out (inter-node IB/RoCE):** the 400G/800G back-end this lesson has been costing. Its $/GB/s is 10×+ higher because it carries switches, long-reach optics, and DC power.

The FinOps consequence: the cheapest bytes are the ones that **never leave the NVLink domain**. A parallelism plan that puts tensor-parallel inside the 8-GPU NVLink island and reserves the expensive scale-out fabric for the lighter pipeline/data-parallel traffic is *also* a cost plan — it maximizes use of the near-free scale-up bandwidth and minimizes demand on the priciest tier. When you argue an oversubscription ratio (§2, §5), you are implicitly arguing that the parallelism plan keeps most GB off the scale-out fabric in the first place.

### 7. The opex tail — power, cabling, and the SM

Capex is only the visible half. Every spine switch and every transceiver you *don't* build in an oversubscribed design also removes a recurring cost: transceivers draw ~10–20 W each and switches hundreds of watts, so a 400G-heavy full-bisection spine adds tens of kW of continuous draw plus the cooling multiplier — a real annual line at DC power rates. Cabling labor and structured-cabling cost also scale with the spine link count. On the IB side, the subnet manager is an *operational* cost — a fabric to run, patch, and staff — that RoCE folds into an Ethernet estate you already operate. So the "1/oversubscription scales the spine" rule (§2) is doubly true: it scales the capex *and* the multi-year opex tail. Price the fabric as **$capex + $power·years**, not $capex alone.

## Worked example

**Relative spine-tier BOM at 1,024 GPUs, full-bisection vs 1:4 vs 1:7.**

Assumptions (stated so the ratios are auditable): 1 × 400G NIC per GPU; 64-radix 400G switches; rail-optimized two-tier fat-tree; leaf = 32 GPU-facing downlinks. Only the uplink count changes with oversubscription. Leaf count is fixed at 1,024 / 32 = **32 leaves** in all three designs, and GPU-facing optics are fixed at **1,024 links** in all three.

| Design | Uplinks/leaf | Spine-facing links | Spine switches (64-radix) | Spine-tier index |
|---|---|---|---|---|
| **1:1 full bisection** | 32 | 32 × 32 = **1,024** | 1,024 / 64 = **16** | **1.0** |
| **1:4** | 8 | 32 × 8 = **256** | 256 / 64 = **4** | **0.25** |
| **1:7** | ~5 | 32 × 5 ≈ **160** | ⌈160/64⌉ = **3** | **~0.15** |

Read the last column as "fraction of the full-bisection spine tier you actually build." Going from full bisection to 1:7 removes ~13 of 16 spine switches and ~864 of 1,024 spine-facing links — and those are the **long-reach single-mode links** (leaf↔spine), the priciest transceivers in the build.

**Dollar sketch (snapshot — flag as volatile).** Spine-facing links are typically single-mode 400G (DR4/FR4). Taking a *2025–2026 street-price snapshot* of ~$300–600 per single-mode 400G transceiver (two per link) → ~$700–1,200 per spine link, plus the spine switches (~$25–40K each, snapshot). Then, at 1,024 GPUs:

- Full bisection spine optics ≈ 1,024 links × ~$900 ≈ **~$0.9M** + 16 switches ≈ **~$0.5M** ≈ **~$1.4M** spine tier.
- 1:7 spine optics ≈ 160 links × ~$900 ≈ **~$0.14M** + 3 switches ≈ **~$0.1M** ≈ **~$0.24M** spine tier.
- **Delta ≈ ~$1.15M** saved on the spine tier alone, for a 1,024-GPU pod — before power and cabling, which scale the same direction. *(All $ figures are snapshots; optics pricing moves fast — re-quote at procurement time.)*

**Throughput risk of each.** The saving is only free if the workload's collectives stay inside the non-blocking domain. If a comms-bound all-reduce spans the spine, effective bisection is cut in proportion: full bisection sustains line-rate all-reduce; 1:4 caps cross-spine all-reduce at ~¼ line-rate; 1:7 at ~⅐. For a data-parallel-heavy trainer where every step ends in a global all-reduce, that cut translates almost linearly into **slower step time and lower GPU utilization** — you would be buying idle H100s to save on switches. The design only pays off when placement (lesson 06) keeps the heavy collective inside the pod, or when SHARP (§4) halves the bytes so the oversubscribed tier is no longer the bottleneck.

## Practice

For the deliverable's procurement section ([network-architecture-read](../practice/network-architecture-read/README.md)), take a **stated job on a stated fabric** and produce the full argument:

1. **Placement → bandwidth.** Pick a concrete job (e.g. a 512-GPU data-parallel Llama-3-class run) on a fabric with a named non-blocking domain and a named oversubscription ratio (e.g. 3,072-GPU pods, 1:7 aggregation, 400G rails). Compute the per-GPU effective all-reduce bandwidth (a) co-located in one pod and (b) spread across the 1:7 tier. State both as aggregate bisection numbers.
2. **Cost statement.** Attach a dollar/opportunity-cost sentence to each: co-location = $0 incremental fabric, but consumes a scarce non-blocking domain; the full-bisection alternative = the ~7× spine BOM you computed in the worked example.
3. **Tolerance argument.** Argue what oversubscription ratio *this workload's traffic shape* can tolerate. Tie it to the parallelism plan: tensor-parallel and NVLink-domain traffic never leaves the node; pipeline-parallel is point-to-point and light; the data-parallel all-reduce is the one that stresses bisection. If DP is small or SHARP-accelerated, a 1:4–1:7 spine is defensible; if DP is the dominant collective across the whole job, argue for a larger non-blocking domain instead of a bigger spine.

**Acceptance:** a written **placement → bandwidth → $** argument for a named job and fabric, defensible in a procurement review — not a switch datasheet, but a workload-specific fabric-tier recommendation with the number that justifies it.

## Self-check

**(a) Turn "co-locate this 512-GPU job in one pod" into a per-GPU bandwidth number versus spreading it across a 1:7 tier.**

**Answer:** On a 400G back-end, co-located inside a non-blocking pod each GPU drives ~400 Gb/s into the all-reduce simultaneously → aggregate bisection ≈ 512 × 400 Gb/s = **~204.8 Tb/s**. Spread across a 1:7 aggregation tier, the cross-spine portion is capped at ~400/7 ≈ **57 Gb/s per GPU** → effective aggregate ≈ **~29 Tb/s**, roughly a 7× cut. Cost framing: the co-located option spends bisection already bought ($0 incremental) at the price of consuming a scarce 3,072-GPU non-blocking domain; matching that bandwidth at cluster scale instead would mean buying the ~7× full-bisection spine tier.

**(b) When is oversubscription the RIGHT call, not just the cheap one?**

**Answer:** When the workload's *traffic shape* keeps the heavy collectives off the oversubscribed tier. Rail-optimized topology already lands each GPU's traffic on a dedicated leaf so that same-rail all-reduce is one hop; tensor-parallel and NVLink-domain traffic never leaves the node; pipeline-parallel is light point-to-point. If the dominant data-parallel all-reduce can be **kept inside one non-blocking pod by placement**, then the cross-pod bandwidth is genuinely rarely needed, and paying for full bisection at the spine would be buying bandwidth the workload never touches. Oversubscription is *right* — not merely cheap — when topology-aware scheduling can guarantee the expensive traffic stays local, which is exactly the Llama 3 design point (1:7 aggregation + topology-aware scheduler). It is *wrong* when jobs routinely span pods or the scheduler cannot constrain them, because then step time degrades in proportion to the bisection cut.

**(c) How does SHARP change the bandwidth-vs-cost math for a reduction-heavy trainer?**

**Answer:** SHARP does the reduction in-switch, cutting all-reduce bytes on the wire from ~2× to ~1× the message size and removing multi-hop traversal — an **effective-bandwidth multiplier** on the same physical fabric. For a trainer dominated by gradient all-reduce, that either lets a **more oversubscribed (cheaper) fabric** hit the target step time, or raises effective bandwidth on the fabric already bought. It shifts the IB-vs-RoCE calculus: SHARP is part of the IB premium (IB switches / NVLink-SHARP in the NVSwitch domain) with no portable RoCE equivalent, so on a reduction-heavy workload the byte savings can be the specific line item that makes the IB tax pay for itself — cheaper spine *because* fewer bytes cross it.

## Resources

- **Llama 3 paper, §3.3.1 "Network Topology"** — https://arxiv.org/abs/2407.21783
  *What for:* the canonical real-world **oversubscribed BOM** — 24K-GPU RoCE cluster, 3-tier Clos, non-blocking 3,072-GPU pods, **1:7 aggregation oversubscription**, topology-aware scheduling to keep cross-pod traffic rare. *Depth:* **deep-read** §3.3.1–§3.3.2 (network + collective communication); skim the rest. *Why:* it is the proof that a frontier run lives happily on an oversubscribed commodity-Ethernet fabric *because* placement keeps the heavy collectives local — the exact argument this lesson teaches you to make.
- **NVIDIA HGX AI Factory reference architecture** — https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/
  *What for:* the **"ideal" full-bisection, rail-optimized** design to contrast against Llama 3's oversubscribed reality — the fabric you build when you refuse to oversubscribe. *Depth:* **skim** the network topology / rail-optimized fat-tree sections for the non-blocking spine-leaf structure and switch-radix math; you do not need the deployment details. *Why:* it anchors the top end of the $/GB/s spectrum, so your procurement argument can name both endpoints — full bisection vs 1:7 — and price the gap.

> **Snapshot flag.** All optics/switch dollar figures in this lesson (400G single-mode transceiver ~$300–600, spine switch ~$25–40K, spine-tier delta ~$1.15M at 1,024 GPUs) are **2025–2026 street-price snapshots** and move fast with supply, generation (400G→800G→1.6T), and volume contracts. Re-quote at procurement time; use the *ratios* (spine tier scales with 1/oversubscription), which are structural, not the absolute dollars.
