---
lesson: "09.7"
title: "Bandwidth as a cost input"
module: "09"
concept: "Bandwidth as a cost input"
status: not-started
est_time: "7h"
prev: "06-k8s-multi-nic.md"
next: null
artifacts: []
sources: 5
---

# 09.7 · Bandwidth as a cost input

> **Concept.** Turn topology into dollars — oversubscription as a capex lever, the IB premium vs RoCE reuse, SHARP as byte-reduction, and the widening scale-up-vs-scale-out $/GB/s gap — and convert a placement choice into a bandwidth-and-cost statement.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

Lesson 06 gave you the *mechanism*: Multus, SR-IOV, the Network Operator, Topology Manager — the K8s plumbing that puts a job's pods on the right NUMA node next to the right NIC VF. That lesson answered "how does a placement decision get enforced." This lesson answers the question procurement and your manager actually ask: "why does that placement decision matter, and what does the alternative cost?" It is the last lesson in the module because it is the synthesis — every earlier lesson (the 02b topology matrix, the Clos fabric of L2, the RDMA/lossless-Ethernet protocol stack of L3–L5, the K8s wiring of L6) feeds into one final move: turning a diagram into a dollar figure a procurement committee will sign off on. That is also exactly what the module's deliverable, the [Network architecture read](../practice/network-architecture-read/README.md), asks you to produce.

## Why this matters

The back-end fabric is one of the largest capex lines in a GPU cluster after the accelerators themselves — switches, transceivers, cables, and the DC power/space they draw. On a rail-optimized H100/H200 build the network BOM routinely lands at **10–20% of total system cost**, and almost all of that swing lives in one knob: **how much bisection bandwidth you buy at the spine tier**. Full non-blocking bisection at cluster scale is the most expensive fabric you can build; a 1:7 oversubscribed spine can cut the spine-tier switch and optics count by roughly 7×.

This is your FinOps edge applied to silicon and glass. The lessons before this one taught you the *physics* of the fabric and the *mechanism* to place jobs on it. This lesson is the **argument**: the ability to walk into a procurement review and say "this workload's traffic shape tolerates a 1:4 spine, saving $X on optics and switches, and here is the placement policy that guarantees the collective never pays the oversubscription tax." A senior platform engineer at a neocloud (CoreWeave, Lambda, Crusoe, etc.) is measured on exactly that sentence — turning a placement decision into a bandwidth number and a dollar number. Anyone can quote a switch datasheet. The signal is converting **$/GB/s of bisection** into a defensible fabric tier for a named workload, and knowing that the numbers you're quoting are dated snapshots, not physical constants.

## What's new here (calibration)

You already know (02b) how to read `nvidia-smi topo -m`, (08) NCCL collective behavior, (06 of this module) how Kueue TAS and Topology Manager enforce a placement — so none of that is re-taught here. What's new:

- **The economics layer on top of the topology you already understand.** Every earlier lesson answered "does it work / how fast." This lesson answers "what does that architecture cost, and what's the counterfactual."
- **Reading a cost model, not just a topology diagram.** $/GB/s of bisection, spine-tier scaling with 1/oversubscription, and the scale-up vs scale-out cost gap are new tools, not new fabric mechanics.
- **Currency discipline.** Every dollar figure in GPU networking is a snapshot that's stale within a year. Part of the new skill here is stating figures *as* snapshots and reasoning from the structural ratios underneath them — which is what survives a hardware generation.
- **Vendor-"ideal" literacy.** Knowing what NVIDIA's own current reference architecture recommends (and that it changed generations) is now part of the argument, not background trivia.

## Core concepts

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

The spine tier and its long-reach optics scale **linearly with 1/oversubscription**. The leaf tier and the GPU-facing optics do not move. So oversubscription is a lever on precisely the most expensive, longest-reach part of the fabric — which is why it is the single most cost-sensitive knob in the whole BOM. **This is the same design Meta shipped for Llama 3** (§3.3.1): non-blocking *inside* a 3,072-GPU pod, **1:7 oversubscribed** at the aggregation tier between pods, with the scheduler kept topology-aware so the expensive cross-pod traffic stays rare. At a full order of magnitude larger, SemiAnalysis's analysis of hypothetical 100,000-H100 builds walks the same lever at cluster scale: a fully non-blocking 100K design needs one more Clos tier (4 tiers) than an oversubscribed one, or the alternative — four **24,576-GPU pods**, each internally non-blocking (3 tiers), joined by a **7:1 oversubscribed** top tier. Same structural argument as Llama 3, ten times the GPU count.

### 3. The IB "tax" vs RoCE "reuse" — and a real, current port-density number

The fabric-technology choice is a second cost axis, orthogonal to oversubscription:

| | InfiniBand | RoCEv2 (Ethernet / Spectrum-X) |
|---|---|---|
| Switch/optics $ | Premium; NVIDIA/Mellanox-dominated supply | Cheaper, multi-vendor, commodity optics |
| Control plane | Dedicated **subnet manager** (SM), separate fabric | Reuses Ethernet/EVPN/BGP — skills you already have |
| Congestion control | Mature, credit-based, in-fabric | PFC/ECN + DCQCN tuning — operationally fiddly |
| In-network compute | **SHARP** (see §4) | No portable equivalent |
| Lock-in | High — vendor + separate ops discipline | Low — standard Ethernet estate |
| Current spine switch example | Quantum-2 NDR: **64 × 400G ports** | Spectrum-X SN5600: **64 × 800G or 128 × 400G**, 51.2 Tb/s in 2U |

That port-density line is a *hard, vendor-confirmable number*, not a soft "Ethernet is cheaper" claim: the current-generation Spectrum-X switch fits double the 400G port count of the current-generation IB switch in the same rack unit (and Broadcom's Tomahawk 5 ASIC — the merchant-silicon competitor — also lands at 128×400G). More ports per switch at the spine tier means fewer spine switches for the same bisection, which is a direct multiplier on the capex savings from §2. The caveat: don't compare port counts without normalizing for radix and reach — a switch configurable between 64×800G and 128×400G is trading port count against per-port speed, not simply "beating" a fixed-radix IB switch on ports alone (see Pitfall 3 below).

Third-party TCO estimates try to put a number on the whole stack: one widely cited 2025/2026 analyst snapshot puts a 512-GPU cluster's 3-year fabric TCO at roughly **$4.6M for InfiniBand vs $2.4M for Ethernet/RoCE** — a ~$2.2M (2.3×) gap — with RoCE estimated to deliver **85–95% of IB's performance for typical training workloads**. Treat that figure the way you'd treat any analyst estimate: directionally useful, not something to defend to the decimal in a procurement review. The *structural* reasons behind it — port density, multi-vendor optics, no dedicated SM to staff — are the durable part.

The **IB premium** buys deterministic congestion behavior and SHARP, at the cost of a dedicated fabric, an SM to run, and vendor lock. The **RoCE reuse** case is that your team already operates Ethernet/EVPN, the optics are cheaper and multi-sourced, port density is currently higher, and a large fraction of real training traffic can be kept off the contended paths by placement. Meta's Llama 3 RoCE cluster is the existence proof that a frontier run does not *require* IB — it requires topology-aware software (NCCLX + a topology-aware scheduler) to live within an oversubscribed, commodity-Ethernet fabric. The FinOps question is never "IB or RoCE" in the abstract; it is "does this workload's traffic shape and my team's skill estate justify the IB premium, or does RoCE + smart placement deliver the same effective bandwidth for less?"

### 4. SHARP as byte-reduction — an effective-bandwidth multiplier

SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) performs the reduction **inside the switch/NVSwitch** instead of shuffling data around a ring or tree of endpoints. For an all-reduce, a ring collective moves roughly **2×(N−1)/N × message_size** of data per GPU over the wire; SHARP collapses that to each endpoint sending its contribution once and receiving the result once — roughly **halving the bytes on the wire** and removing the multi-hop traversal latency.

Reframe that as cost: SHARP is a **byte-reduction lever**, complementary to the oversubscription capex lever. Fewer bytes for the same reduction means the *same physical fabric delivers more effective all-reduce bandwidth* — so a reduction-heavy trainer can hit its target step time on a **cheaper (more oversubscribed) fabric**, or reach a higher effective bandwidth on the fabric you already bought. The catch: SHARP is part of the **IB premium** (IB switches, or NVLink-SHARP/NVLS inside the NVSwitch domain). RoCE has no portable equivalent. So SHARP is often the specific reason the IB tax pays for itself on a workload dominated by gradient all-reduce.

### 5. Placement → bandwidth conversion (the whole point)

A placement decision *is* a bandwidth statement, once you know the topology:

- **Co-locate the job inside one non-blocking pod** → the collective sees **full per-GPU line-rate bisection**. On a 400 Gb/s (400G) back-end, each GPU can drive ~400 Gb/s into the all-reduce simultaneously. Cost of this bandwidth: **$0 incremental** — you are spending bisection you already bought. The price you pay is an *opportunity cost*: you consume a scarce non-blocking domain (3,072 GPUs in the Llama 3 design) and constrain the scheduler.
- **Spread the same job across a 1:7 aggregation tier** → any part of the collective that crosses the spine is capped at **~1/7 of line rate**, ~57 Gb/s per GPU on a 400G fabric. A comms-bound all-reduce then slows in proportion to that cut.

So the scheduler policy from lesson 06 is not a scheduling nicety — it is what lets an **oversubscribed-fabric capex deliver full-bisection performance** for the jobs that matter. Co-location converts a cheap fabric into an expensive-fabric experience *for traffic that stays inside the blast radius*. That single sentence is the procurement throughline: buy the oversubscribed spine, then use placement to make sure your bandwidth-hungry collectives never touch it.

### 6. Scale-up vs scale-out — and the gap is *widening*, not narrowing

The single most common procurement error is treating "bandwidth" as one number. There are two fabrics with an order-of-magnitude different price per GB/s, and the gap between them has grown, not shrunk, across the last two GPU generations:

| | H100 generation | GB200/GB300 (Blackwell Ultra) generation |
|---|---|---|
| Scale-up (NVLink, per GPU) | ~900 GB/s (NVLink 4) | **~1.8 TB/s (NVLink 5)** — bundled into the server SKU |
| Scale-out (back-end fabric) | 400G (ConnectX-7) | 800G (ConnectX-8 SuperNIC) |

Both tiers roughly doubled generation-over-generation. But they doubled on completely different cost bases. NVLink bandwidth is **bundled into the server SKU** — you don't buy it per-port, so its *incremental* $/GB/s is close to zero regardless of how high the raw number climbs. Scale-out bandwidth, by contrast, still carries the full per-port cost of a switch ASIC, an optical transceiver, and the DC power to run both, even at 800G. Doubling a near-zero-marginal-cost number and doubling a fully-costed number does not close the gap between them — it **widens** it in absolute dollars, because the scale-out side's already-nonzero $/GB/s just doubled its raw throughput on the same cost structure. A mental model calibrated to H100-era numbers under-states how valuable it now is to keep parallelism inside the NVLink domain on the newest hardware.

The FinOps consequence: the cheapest bytes are the ones that **never leave the NVLink domain**, and that's more true every generation. A parallelism plan that puts tensor-parallel inside the 8-GPU NVLink island and reserves the expensive scale-out fabric for the lighter pipeline/data-parallel traffic is *also* a cost plan — it maximizes use of the near-free scale-up bandwidth and minimizes demand on the priciest tier. When you argue an oversubscription ratio (§2, §5), you are implicitly arguing that the parallelism plan keeps most GB off the scale-out fabric in the first place.

### 7. The opex tail — power, cabling, and the SM

Capex is only the visible half. Every spine switch and every transceiver you *don't* build in an oversubscribed design also removes a recurring cost: transceivers draw ~10–20 W each and switches hundreds of watts, so a 400G/800G-heavy full-bisection spine adds tens of kW of continuous draw plus the cooling multiplier — a real annual line at DC power rates, and exactly the kind of concern SemiAnalysis's cluster-power analysis treats as inseparable from the topology choice at 100K-GPU scale. Cabling labor and structured-cabling cost also scale with the spine link count. On the IB side, the subnet manager is an *operational* cost — a fabric to run, patch, and staff — that RoCE folds into an Ethernet estate you already operate. So the "1/oversubscription scales the spine" rule (§2) is doubly true: it scales the capex *and* the multi-year opex tail. Price the fabric as **$capex + $power·years**, not $capex alone.

### 8. Even NVIDIA's own "ideal" reference now defaults to Ethernet

If your mental model of "the vendor-blessed ideal fabric" is InfiniBand, it's out of date. NVIDIA's current **HGX AI Factory reference architecture** — its own full-bisection, rail-optimized design guide — is built around **Spectrum-X Ethernet**, not InfiniBand: a "2-8-9-800" configuration (2 CPUs, 8 GPUs, 9 NICs, 800 Gb/s per GPU) on HGX B300 (Blackwell Ultra) servers, with ConnectX-8 SuperNICs and BlueField-3 DPUs, at 32/64/128-node design points. This matters for the argument in this lesson two ways: first, it means the "ideal, full-bisection" endpoint you contrast against an oversubscribed real build is no longer automatically an IB endpoint — you have to say *which* fabric's full-bisection cost you mean. Second, it's a signal from the vendor with the most to gain from IB lock-in that RoCE/Spectrum-X is now good enough to be the flagship recommendation, which should update how hard you lean on "but IB is the safe default" in a procurement argument.

## Perspectives

**Developer.** A model developer never sees the fabric invoice. But the single decision they make — the parallelism plan, i.e. how much communication volume is tensor-parallel (stays inside the NVLink island) versus data-parallel or pipeline-parallel (crosses the scale-out fabric) — is the biggest lever on that invoice that exists anywhere in the stack. A developer who defaults to a parallelism config without thinking about NVLink-domain size is silently setting the fabric team's budget.

**Operator / procurement.** The spine tier is where money and blast-radius risk both concentrate: it's the tier with the fewest, most expensive switches and the longest-reach optics, and it's the tier a bad oversubscription bet degrades for every job that crosses it. Procurement's job is to turn "how much bisection do we need" into a signed BOM line, and that number is only defensible if it's tied to a named workload's traffic shape, not a generic "buy the biggest fabric" instinct.

**Economics / FinOps.** Not all cost claims carry equal weight. Port density — SN5600's 128×400G in 2U versus Quantum-2's 64×400G — is a hard, vendor-datasheet-confirmable number you can defend line by line. A blanket "RoCE is 2.3× cheaper" TCO estimate is a soft, dated, third-party snapshot — useful for framing a conversation, not for defending a specific dollar figure under scrutiny. Know which kind of number you're holding before you cite it in a review.

**Failure mode.** A wrong oversubscription bet does not show up as a network alert. There's no red dashboard tile that says "spine undersized." It shows up as a slower step time or lower MFU (model FLOPs utilization), and the first instinct in most orgs is to blame the model, the data pipeline, or "GPU flakiness" — long before anyone traces the regression back to a collective crossing an oversubscribed tier it was never supposed to touch. Being the engineer who can trace that chain backwards, from a slow step to a specific tier's bandwidth cap, is the entire point of this lesson.

## Real-world use cases

- **SemiAnalysis, "100,000 H100 Clusters: Power, Network Topology, Ethernet vs InfiniBand, Reliability, Failures, Checkpointing"** — https://newsletter.semianalysis.com/p/100000-h100-clusters-power-network — an industry cost/topology deep-dive at exactly the scale and question this lesson's worked example addresses: fully non-blocking (4-tier) vs 4-pod/7:1-oversubscribed (3-tier-per-pod) designs at 100K-GPU scale, the current Spectrum-X (128×400G) vs Quantum-2 IB (64×400G) port-density gap, and power/reliability tradeoffs tied to the topology choice. *What it shows:* the IB-vs-Ethernet, oversubscription-tier decision this lesson teaches, worked at the largest real scale publicly analyzed.
- **Meta, Llama 3 paper §3.3.1 "Network"** — https://arxiv.org/abs/2407.21783 — read here as a **procurement case study**: a named org (Meta) choosing RoCE at 24K-GPU scale over IB, and defending it with exactly the placement-preserves-bandwidth argument this lesson teaches (non-blocking 3,072-GPU pods, 1:7 aggregation oversubscription, topology-aware scheduling to keep the expensive cross-pod traffic rare). *What it shows:* a frontier training run does not require InfiniBand — it requires the software discipline to live within an oversubscribed commodity-Ethernet fabric.
- **NVIDIA, HGX AI Factory reference architecture (current)** — https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/ — the vendor's own current "ideal" full-bisection design guide, now built on Spectrum-X Ethernet (ConnectX-8, BlueField-3) rather than InfiniBand. *What it shows:* a useful contrast against older IB-centric framing — even the vendor with the most to gain from IB lock-in ships an Ethernet-first flagship reference build today.

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

**Sanity-check against a third-party whole-cluster TCO snapshot.** At a much smaller reference point (512 GPUs, whole-fabric 3-year TCO, not just the spine tier), one 2025/2026 analyst estimate puts InfiniBand at ~$4.6M vs Ethernet/RoCE at ~$2.4M — a ~$2.2M gap. That figure bundles leaf + spine + optics + 3 years of the difference in operational overhead, so it isn't directly comparable to the spine-only delta above, but the direction and rough order of magnitude (multi-million-dollar swings at four-figure GPU counts) agree. Treat both as dated snapshots — the ratios (spine scales with 1/oversubscription; RoCE undercuts IB on port-normalized $) are what's durable, not the dollar figures.

**Throughput risk of each.** The saving is only free if the workload's collectives stay inside the non-blocking domain. If a comms-bound all-reduce spans the spine, effective bisection is cut in proportion: full bisection sustains line-rate all-reduce; 1:4 caps cross-spine all-reduce at ~¼ line-rate; 1:7 at ~⅐. For a data-parallel-heavy trainer where every step ends in a global all-reduce, that cut translates almost linearly into **slower step time and lower GPU utilization** — you would be buying idle H100s to save on switches. The design only pays off when placement (lesson 06) keeps the heavy collective inside the pod, or when SHARP (§4) halves the bytes so the oversubscribed tier is no longer the bottleneck.

## Practice

For the deliverable's procurement section ([network-architecture-read](../practice/network-architecture-read/README.md)), take a **stated job on a stated fabric** and produce the full argument:

1. **Placement → bandwidth.** Pick a concrete job (e.g. a 512-GPU data-parallel Llama-3-class run) on a fabric with a named non-blocking domain and a named oversubscription ratio (e.g. 3,072-GPU pods, 1:7 aggregation, 400G rails). Compute the per-GPU effective all-reduce bandwidth (a) co-located in one pod and (b) spread across the 1:7 tier. State both as aggregate bisection numbers.
2. **Cost statement.** Attach a dollar/opportunity-cost sentence to each: co-location = $0 incremental fabric, but consumes a scarce non-blocking domain; the full-bisection alternative = the ~7× spine BOM you computed in the worked example. Note which numbers you're citing are hard/vendor-confirmable (port density) versus soft/dated (whole-cluster TCO).
3. **Tolerance argument.** Argue what oversubscription ratio *this workload's traffic shape* can tolerate. Tie it to the parallelism plan: tensor-parallel and NVLink-domain traffic never leaves the node (and is getting relatively cheaper every GPU generation — §6); pipeline-parallel is point-to-point and light; the data-parallel all-reduce is the one that stresses bisection. If DP is small or SHARP-accelerated, a 1:4–1:7 spine is defensible; if DP is the dominant collective across the whole job, argue for a larger non-blocking domain instead of a bigger spine.
4. **IB-vs-RoCE verdict.** For the same scenario, pick IB or RoCE/Spectrum-X and defend it on ≥4 axes (latency, NCCL GB/s, PFC/ECN tuning risk, SHARP availability, cost, lock-in) — this is checkpoint item 4 and section §4 of the deliverable spec.

**Acceptance:** a written **placement → bandwidth → $** argument for a named job and fabric, defensible in a procurement review — not a switch datasheet, but a workload-specific fabric-tier recommendation with the number that justifies it.

## Common pitfalls

- **Quoting a single "$/GB/s" or TCO delta as universal or timeless.** Every number in this lesson — transceiver price, switch price, the 2.3× IB/RoCE TCO ratio — is a dated snapshot. Prices *and* architecture move fast (NVIDIA's own reference design moved from IB to Spectrum-X between generations). State the year, and lean on the structural ratios, not the absolute dollars, when you want an argument that survives a hardware refresh.
- **Assuming the scale-up/scale-out gap narrows as NVLink gets faster.** It's the opposite: NVLink bandwidth is bundled into the server SKU at near-zero incremental $/GB/s, while scale-out bandwidth still carries full per-port switch/optics/power cost even as it doubles too. A mental model anchored to H100-era ~900 GB/s NVLink under-states how much more valuable it now is, at ~1.8 TB/s NVLink 5, to keep traffic inside the NVLink domain.
- **Comparing switch port counts without normalizing for radix and reach.** Spectrum-X SN5600's "128×400G or 64×800G" flexibility is a real port-density edge, but it's trading port count against per-port speed — it isn't simply "twice the switch" of a fixed-radix IB switch. Always state the comparison at matched aggregate bandwidth (Tb/s per switch), not raw port count.
- **Treating "oversubscription saves money" as free.** It always trades against a specific workload's tolerance. The savings in the worked example only materialize if placement (lesson 06) keeps the bandwidth-hungry collective inside the non-blocking domain; otherwise you're paying in step time what you saved in capex.
- **Assuming IB is still the vendor-blessed default.** NVIDIA's own current HGX AI Factory reference architecture is Spectrum-X/Ethernet-based. Don't structure a procurement argument around "IB is the safe/ideal choice and RoCE is the discount option" — that framing is now out of date even from the IB vendor itself.

## Self-check

**1. Turn "co-locate this 512-GPU job in one pod" into a per-GPU bandwidth number versus spreading it across a 1:7 tier.**

**Answer:** On a 400G back-end, co-located inside a non-blocking pod each GPU drives ~400 Gb/s into the all-reduce simultaneously → aggregate bisection ≈ 512 × 400 Gb/s = **~204.8 Tb/s**. Spread across a 1:7 aggregation tier, the cross-spine portion is capped at ~400/7 ≈ **57 Gb/s per GPU** → effective aggregate ≈ **~29 Tb/s**, roughly a 7× cut. Cost framing: the co-located option spends bisection already bought ($0 incremental) at the price of consuming a scarce 3,072-GPU non-blocking domain; matching that bandwidth at cluster scale instead would mean buying the ~7× full-bisection spine tier.

**2. When is oversubscription the RIGHT call, not just the cheap one?**

**Answer:** When the workload's *traffic shape* keeps the heavy collectives off the oversubscribed tier. Rail-optimized topology already lands each GPU's traffic on a dedicated leaf so that same-rail all-reduce is one hop; tensor-parallel and NVLink-domain traffic never leaves the node; pipeline-parallel is light point-to-point. If the dominant data-parallel all-reduce can be **kept inside one non-blocking pod by placement**, then the cross-pod bandwidth is genuinely rarely needed, and paying for full bisection at the spine would be buying bandwidth the workload never touches. Oversubscription is *right* — not merely cheap — when topology-aware scheduling can guarantee the expensive traffic stays local, which is exactly the Llama 3 design point (1:7 aggregation + topology-aware scheduler) and the 4-pod/7:1 option SemiAnalysis walks at 100K-GPU scale. It is *wrong* when jobs routinely span pods or the scheduler cannot constrain them, because then step time degrades in proportion to the bisection cut.

**3. How does SHARP change the bandwidth-vs-cost math for a reduction-heavy trainer?**

**Answer:** SHARP does the reduction in-switch, cutting all-reduce bytes on the wire from ~2× to ~1× the message size and removing multi-hop traversal — an **effective-bandwidth multiplier** on the same physical fabric. For a trainer dominated by gradient all-reduce, that either lets a **more oversubscribed (cheaper) fabric** hit the target step time, or raises effective bandwidth on the fabric already bought. It shifts the IB-vs-RoCE calculus: SHARP is part of the IB premium (IB switches / NVLink-SHARP in the NVSwitch domain) with no portable RoCE equivalent, so on a reduction-heavy workload the byte savings can be the specific line item that makes the IB tax pay for itself — cheaper spine *because* fewer bytes cross it.

**4. Why does the scale-up vs scale-out $/GB/s gap widen, not narrow, going from H100 to GB200/GB300?**

**Answer:** NVLink bandwidth roughly doubled (900 GB/s → 1.8 TB/s, NVLink 4 → NVLink 5) while scale-out also roughly doubled (400G → 800G). But NVLink is bundled into the server SKU — its *incremental* $/GB/s is near zero regardless of the raw number — while scale-out still carries the full per-port switch/optics/power cost at 800G that it carried at 400G, just doubled in throughput on that same nonzero cost base. Doubling a near-free number and doubling a fully-costed number does not close the gap between them; the fully-costed side's absolute dollar cost per unit of bandwidth stays roughly flat or grows, so NVLink's relative cost advantage grows every generation. A parallelism plan calibrated to H100-era intuition therefore under-values keeping tensor-parallel traffic inside the NVLink domain on newer hardware.

**5. What does NVIDIA's own current HGX AI Factory reference architecture choose at the fabric layer, and why is that noteworthy?**

**Answer:** Spectrum-X Ethernet with ConnectX-8 SuperNICs and BlueField-3 DPUs, not InfiniBand — a "2-8-9-800" configuration on HGX B300 servers at 32/64/128-node design points. It's noteworthy because this is the vendor's own "ideal, full-bisection" reference build, and the same vendor sells both fabrics; if InfiniBand were still unambiguously the performance-leading choice, it's the natural default for a flagship reference architecture. Its absence reinforces that "IB is the default" is a dated mental model, and that the IB-vs-RoCE decision from lesson 04 is genuinely live even at the top of the market, not just a budget-constrained compromise.

## Connections & what's next

This lesson closes the module's arc, so it's worth naming the whole thread explicitly: **02b** gave you the topology matrix inside one node (`nvidia-smi topo -m`, NVLink vs PCIe, `SYS`/`PXB` penalties). **L1–L2** extended that matrix outward past the NIC into the inter-node **fabric** — Clos/fat-tree structure, rail optimization, oversubscription. **L3–L5** covered the **protocol** that rides that fabric — RDMA's kernel-bypass, IB vs RoCEv2 and lossless Ethernet (PFC/ECN/DCQCN), GPUDirect and SHARP. **L6** was the **K8s** mechanism that turns a placement decision into an enforced pod scheduling constraint (Multus, SR-IOV, Topology Manager). This lesson is the **cost** layer on top of all of it — the same topology, protocol, and placement facts, reframed as a $/GB/s argument a procurement committee will act on. Every number in this lesson traces back to a concept from an earlier lesson: bisection bandwidth (L2), the IB/RoCE tradeoff table (L4), SHARP (L5), and the placement mechanism that makes oversubscription safe (L6).

What's next is not another lesson — it's the module's proof of work. Take everything from 02b through this lesson and produce the **[Network architecture read](../practice/network-architecture-read/README.md)**: redraw the Llama-3 (or IB-variant) topology with per-tier bisection and oversubscription labeled, predict where a named job bottlenecks under two placements and quantify the penalty with the ratio, make the co-location argument with real bandwidth numbers, defend an IB-vs-RoCE verdict on ≥4 axes, argue an oversubscription tolerance for the workload, and ground all of it with a real 2-GPU `nccl-tests all_reduce_perf` capture. Then close the module against the **[checkpoint](../checkpoint.md)** — it gates on exactly the skills this lesson chain built: reading a `topo -m` (02b/L1), computing an oversubscription ratio (L2/this lesson), explaining lossless RoCE (L3/L4), arguing IB vs RoCE on ≥4 axes (L4/this lesson), defining rail-optimized topology (L2/L5), tracing GPUDirect/RDMA end to end (L3/L5), explaining the K8s path (L6), and — the module's whole point — turning a topology into a bandwidth number and a placement argument into a dollar figure.

## References & further reading

**Primary sources**
- Meta, **Llama 3 paper, §3.3.1 "Network Topology"** — https://arxiv.org/abs/2407.21783 — read for the canonical real-world oversubscribed BOM: 24K-GPU RoCE cluster, 3-tier Clos, non-blocking 3,072-GPU pods, 1:7 aggregation oversubscription, topology-aware scheduling. Deep-read §3.3.1–§3.3.2; skim the rest.
- NVIDIA, **HGX AI Factory reference architecture (current)** — https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/ — read for the vendor's current "ideal," full-bisection, rail-optimized design point (now Spectrum-X/Ethernet-based, 2-8-9-800 on HGX B300, 32/64/128-node scale). Skim the networking-hardware section for the switch-radix and port-count details.

**Real-world engineering blogs**
- SemiAnalysis, **"100,000 H100 Clusters: Power, Network Topology, Ethernet vs InfiniBand, Reliability, Failures, Checkpointing"** — https://newsletter.semianalysis.com/p/100000-h100-clusters-power-network — what it shows: a detailed industry cost/topology analysis at 100K-GPU scale, working the same non-blocking-vs-oversubscribed-pod tradeoff this lesson teaches, with current Spectrum-X vs Quantum-2 port-density numbers and a power/reliability angle.
- Meta, **Llama 3 paper §3.3.1** (also listed above) — what it shows, read as a procurement case study: a named org defending a RoCE choice at 24K-GPU scale with the placement-preserves-bandwidth argument this lesson teaches.
- NVIDIA, **HGX AI Factory reference architecture** (also listed above) — what it shows: the current vendor "ideal" is Ethernet/Spectrum-X, not InfiniBand — a live contrast to older IB-centric framing of "the ideal fabric."

**Deeper dives**
- CoreWeave, **NCCL configuration reference** — https://docs.coreweave.com/products/networking/hpc-interconnect/nccl-configuration-reference — production `NCCL_IB_HCA` / GDR / CollNet settings from an operator running both Quantum-X800 IB and Spectrum-X RoCE at scale; useful for grounding the abstract cost argument in what a real ops team tunes.
- NVIDIA, **Network Operator repository** — https://github.com/Mellanox/network-operator — the `NicClusterPolicy` stack from lesson 06, useful background for understanding what the K8s side of a placement policy actually deploys.

> **Snapshot flag.** All dollar figures in this lesson (400G single-mode transceiver ~$300–600, spine switch ~$25–40K, spine-tier delta ~$1.15M at 1,024 GPUs, the ~$4.6M-vs-$2.4M 512-GPU 3-year TCO estimate, "85–95% of IB performance" for RoCE) are **2025–2026 snapshots** — several from third-party analyst estimates, not vendor-primary numbers — and move fast with supply, generation (400G→800G→1.6T), and volume contracts. Re-quote at procurement time; use the *ratios* (spine tier scales with 1/oversubscription; scale-up $/GB/s advantage widens each NVLink generation) which are structural, not the absolute dollars.
