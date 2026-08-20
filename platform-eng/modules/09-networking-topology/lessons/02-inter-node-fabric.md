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
sources: 12
---

# 09.2 · The inter-node fabric: Clos, bisection, and rail-optimized oversubscription

> **Concept.** A GPU cluster's fabric is a Clos/fat-tree, and its cost is set by one dial — how much bisection bandwidth you buy at each tier; rail-optimized design lets you keep that dial low at the spine *for free* because LLM traffic is rail-local and NVLink absorbs the rest. But "3-tier Clos, 1:7 at the top" is one design, not the only correct one — this lesson also shows you when and why hyperscalers deviate from it.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)
>
> **Advanced Learning** — [Radix, Ratios and ECMP](../../../learning/02-inter-node-fabric.html): fat-tree capacity arithmetic, when oversubscription is free, and ECMP's assumption against what training sends. Optional visual companion; this lesson stays the source of truth.

## Where this fits

09.1 got you to the leaf switch. It gave you the `GPU → NIC → leaf` on-ramp, the 9:1 bandwidth cliff at the NIC, the *rail* as a vertical slice of GPU-N-to-leaf-N cabling, and a three-phase model of a hierarchical all-reduce whose inter-node term is `2·(M−1)/M · (S/8) / B_nic`. That model assumed one thing it never justified: that every GPU actually gets `B_nic` — the full 50 GB/s of its NIC — when it talks across nodes.

That assumption is only true if the fabric above the leaf can carry every rail at line rate simultaneously. This lesson is about what happens when it can't, and about the fact that **you should usually not pay for it to be true everywhere**. It builds the tiers above the leaf from switch radix upward, defines bisection bandwidth and oversubscription with the arithmetic that goes with them, shows how rail-optimised design makes an oversubscribed spine nearly free for LLM training, and then replaces `B_nic` in 09.1's model with an oversubscription-aware term so you can compute what a given ratio actually costs a named job. It closes with real hyperscalers who did *not* build the textbook answer, and why — because the interview-ready version of this material is not "here is the formula," it is "here is the formula, and here is when a team with real traffic data threw it out."

## Why this matters

After the GPUs themselves, fabric bandwidth is the largest procurement lever in a GPU build, and it is the one platform engineers are least often equipped to argue about. Switches, optics and cabling scale super-linearly with the bisection you demand: each additional tier multiplies the number of long-reach optical transceivers, and at 400G/800G those transceivers, not the switch ASICs, are frequently the dominant line item. Argue for full any-to-any bisection across 24,000 GPUs and you have doubled the network bill to buy bandwidth that LLM training never uses. Argue for the *right* oversubscription and you have funded another pod of GPUs.

The failure in the other direction is just as expensive and much harder to see. An oversubscription ratio that was correct for one parallelism strategy becomes wrong when the model, the world size, or the placement policy changes — and the symptom is not an alarm, it is a step time that drifts up and a cluster that quietly returns less throughput per GPU-hour than the capacity plan promised. This lesson is the vocabulary and the arithmetic that let you sit across from the fabric team and say "1:7 at aggregation is fine here, and here is the number behind that" — and, equally, "1:7 is *not* fine for this job, and here is the number behind *that*."

## What's new here (calibration)

- **Already yours (02b, 09.1, 08, 06):** the `GPU → NIC → leaf` on-ramp and the rail concept; the 9:1 cliff; ring and tree all-reduce as the traffic pattern; topology-aware gang placement; the three-phase all-reduce time model. We build on these rather than re-deriving them.
- **Genuinely new:** switch **radix** as the primitive that forces a tree in the first place, and the `k²/2` and `k³/4` endpoint formulas that fall out of it; a port-by-port build of a non-blocking pod, with the switch and optic counts that make the cost argument concrete; formal definitions of **bisection bandwidth** and **oversubscription ratio** with worked numbers; the rail-optimised fat tree drawn tier by tier with ratios marked; the **mechanics of ECMP hashing on a RoCE fabric** — why flow entropy is a function of *queue pairs*, not of GPUs, and why that makes elephant-flow hash collisions structural rather than unlucky; and an oversubscription-aware extension of 09.1's all-reduce model.
- **Deliberately deferred:** *why* an Ethernet fabric can carry lossless, collective-grade RDMA traffic at all — PFC, ECN, DCQCN, and the deadlock story — is 09.4's job. Here, InfiniBand and RoCE appear only as line rates and radix numbers to anchor the structural argument.

## Core concepts

### 1. The problem a Clos exists to solve: switch radix

Start from the constraint that makes everything else necessary. A switch has a fixed number of ports at a given speed — its **radix**. Two current examples to anchor the arithmetic:

| Switch | Fabric | Radix |
|---|---|---|
| NVIDIA Quantum-2 (QM9700 class) | InfiniBand NDR | 64 × 400 Gb/s |
| NVIDIA Spectrum-4 (SN5600 class) | Ethernet | 64 × 800 Gb/s, or 128 × 400 Gb/s, 51.2 Tb/s aggregate |

You want to connect 24,576 endpoints. No switch has 24,576 ports, and none ever will, because radix is bounded by the switch ASIC's total switching capacity divided by the per-port rate — 51.2 Tb/s buys you 64 ports at 800G or 128 at 400G, and that is the trade, not a choice you can escape by buying a bigger box.

So you build a **tree of switches**, and you make it a tree with *many equal-cost paths* rather than one path, because a single path would make the tree's root a bottleneck and a single point of failure. That structure is a **Clos network**; the datacentre form, where every tier uses the same-radix switch and the tree is "folded" so endpoints hang only off the bottom, is a **fat tree**.

The counting is worth doing once, because it tells you how many tiers a given cluster needs and therefore how many optics you are about to buy. For a switch of radix `k`, in a fat tree where each switch splits its ports evenly between "down" and "up":

```
   2-tier fat tree (leaf + spine):
       each leaf: k/2 ports down to endpoints, k/2 ports up to spines
       max endpoints  =  (number of leaves) × k/2
                      =  k × k/2  =  k²/2

   3-tier fat tree (leaf + spine + super-spine / aggregation):
       max endpoints  =  k³/4

   Run the numbers:
       k = 64   (Quantum-2, 400G):   2-tier →  2,048 endpoints
                                     3-tier → 65,536 endpoints
       k = 128  (SN5600 at 400G):    2-tier →  8,192 endpoints
                                     3-tier → 524,288 endpoints
```

**Read the consequence, because it is the whole reason tier count is a design variable.** A 2,048-GPU cluster on 64-port switches fits in two tiers. A 24,576-GPU cluster on the same switches does not — it needs three, and the third tier is the one whose links are long-reach (rack row to rack row, hall to hall), which is where optics costs concentrate. Doubling switch radix from 64 to 128 quadruples what a 2-tier design can hold, which is exactly why a higher-radix Ethernet switch can sometimes let you *delete a tier* — and deleting a tier deletes an entire population of optical transceivers, not just some switches. That is the structural version of "Ethernet is cheaper," and it is more durable than any price quote.

### 2. Building a non-blocking pod, port by port

Abstract definitions of "non-blocking" are where people lose the thread. Build one instead, with a real radix, and count the parts.

**Target:** 2,048 GPU endpoints, each with a 400 Gb/s NIC, all-to-all at line rate, using 64-port 400G switches.

```
  TIER 2 — SPINE:  32 switches × 64 ports
                   ┌────┐ ┌────┐            ┌────┐
                   │ S0 │ │ S1 │  …  ×32    │S31 │
                   └─┬──┘ └─┬──┘            └─┬──┘
        each spine has 64 ports, one down to EVERY leaf
                     │      │                 │
        ┌────────────┴──────┴────────  …  ────┴──────────┐
        │           2,048 leaf↔spine links               │
        └──┬──────────┬──────────┬───────  …  ────┬──────┘
  TIER 1 — LEAF:  64 switches × 64 ports
        ┌──┴──┐   ┌──┴──┐    ┌──┴──┐          ┌──┴──┐
        │ L0  │   │ L1  │    │ L2  │    …     │ L63 │
        └┬┬┬┬┬┘   └┬┬┬┬┬┘    └┬┬┬┬┬┘          └┬┬┬┬┬┘
      32 GPUs   32 GPUs    32 GPUs           32 GPUs      = 2,048 endpoints

  THE COUNT:
     leaf switches   = 2048 endpoints ÷ 32 down-ports  = 64
     spine switches  = 64 up-ports per leaf ÷ ... → 32 spines of radix 64
     leaf↔spine links= 64 leaves × 32 uplinks           = 2,048
     optical ends    = 2,048 links × 2 transceivers     = 4,096
     plus            = 2,048 host links × 2             = 4,096
                                            TOTAL ≈ 8,192 transceivers

  THE RATIO AT THE LEAF:  32 down : 32 up  =  1:1  →  NON-BLOCKING
```

Every number above follows from radix and the 1:1 split. Notice what "non-blocking" actually bought: the leaf tier can absorb all 32 of its endpoints sending simultaneously to endpoints on *other* leaves, because it has exactly as much capacity going up as coming in. Notice also what it cost: **2,048 inter-switch links and about 4,096 transceivers on top of the host cabling**, for 2,048 GPUs. Roughly two optics per GPU, per tier, is the rule of thumb this build establishes — and it is why each additional tier is a step change in cost, not a marginal one.

### 3. Bisection bandwidth, defined and computed

**Bisection bandwidth** is the aggregate capacity of the links you must cut to split the network into two equal halves, minimised over all such cuts. It is the honest worst-case measure of "how much can one half of the cluster say to the other half at once."

For the pod above:

```
   Cut the pod in half: 1,024 GPUs on each side.
   Worst-case demand across the cut = 1,024 senders × 400 Gb/s
                                    = 409.6 Tb/s

   Capacity across the cut: every leaf reaches every spine, so a cut that
   splits leaves evenly must cross the leaf↔spine links of one half:
        1,024 uplinks × 400 Gb/s     = 409.6 Tb/s

   demand = capacity  →  FULL (non-blocking) BISECTION
   Per-GPU worst case across the cut = 400 Gb/s = 50 GB/s = line rate.
```

That last line is the one to carry: in a non-blocking tier, **the worst case equals the line rate**, so 09.1's model can use `B_nic` unmodified. Everything in the rest of this lesson is about what to substitute for `B_nic` when that is not true.

Two clarifications that prevent common errors. First, bisection is a property of a *cut*, so it is meaningful per tier: a fabric can be non-blocking at the leaf and 1:7 at the aggregation tier, and quoting "the bisection" without naming the tier is meaningless. Second, full bisection is a statement about *capacity*, not about *achieved throughput*: a fabric with full bisection can still deliver far less than line rate if routing piles flows onto a subset of the paths (§6).

### 4. Oversubscription: the ratio that sets the bill

**Oversubscription** is the deliberate decision to provide less uplink capacity than downlink capacity at a tier. The **oversubscription ratio** is `downlink bandwidth : uplink bandwidth`.

```
   NON-BLOCKING LEAF                 4:1 OVERSUBSCRIBED LEAF
   ┌────────────────────┐            ┌────────────────────┐
   │  32 up  (12.8 Tb/s)│            │   8 up   (3.2 Tb/s)│
   │  32 down(12.8 Tb/s)│            │  32 down(12.8 Tb/s)│
   └────────────────────┘            └────────────────────┘
        ratio 1:1                          ratio 4:1

   Consequences of the 4:1 leaf:
     • a flow between two GPUs on the SAME leaf                → 400 Gb/s
       (it never touches an uplink; the ratio is irrelevant)
     • all 32 GPUs sending simultaneously OFF the leaf         → 3.2 Tb/s
       shared 32 ways = 100 Gb/s each  =  line rate ÷ 4
     • cost: 24 fewer uplinks per leaf, 48 fewer transceivers,
       and a proportionally smaller spine tier
```

Two rules fall out, and they are the ones you will use constantly:

1. **Oversubscription only taxes traffic that crosses the oversubscribed tier.** Intra-leaf traffic is unaffected. This is the entire foundation of the rail-optimised argument in §5.
2. **Worst-case per-endpoint bandwidth through an X:1 tier is `line rate ÷ X`, when every endpoint under the tier is active across it.** If only a fraction of endpoints are crossing, the survivors get proportionally more; the ratio is a worst case, not a constant tax.

Why the ratio dominates cost: at a fully non-blocking tier, that tier must carry the *sum* of everything beneath it. Uplink count, spine switch count, and — dominant at scale — optical transceivers all scale with that sum. Halving the uplinks at a tier roughly halves that tier's switch-and-optics bill. So the design question is never "full bisection or not" in the abstract. It is **"at which tier can this traffic pattern tolerate oversubscription, and by how much"** — and answering it requires knowing the traffic pattern, which for LLM training we do.

### 5. Rail-optimised design: why the spine can be oversubscribed almost for free

A general-purpose cloud cannot oversubscribe aggressively because any VM may talk to any VM; the traffic matrix is dense and unpredictable. LLM training's traffic matrix is neither, and 09.1 already gave you both reasons:

1. **The heavy collectives are rail-local.** In data-, tensor- and pipeline-parallel training the bandwidth-dominant operations — all-reduce, all-gather, reduce-scatter — run **GPU-N ↔ GPU-N**. Every participant in a given inter-node exchange sits on the *same rail*, so its bytes ride one leaf and terminate there.
2. **NVLink absorbs the cross-rail residue.** The only traffic that would naturally cross rails (GPU3 wanting GPU5) is shuffled sideways inside the node over NVLink first — 450 GB/s per direction against the NIC's 50 — so it too leaves on its own rail. NCCL implements this as PXN (09.1 §4).

Put those together and you get the design:

```
   RAIL-OPTIMISED FAT TREE, ratios marked per tier

   TIER 3 — AGGREGATION / super-spine          ┌───────────────┐
   joins PODS.  Oversubscribed, e.g. 1:7       │  AGG SWITCHES │
   Carries: cross-pod jobs, checkpoints,       └───┬───────┬───┘
            storage, evaluation, control        1:7│       │1:7   ◄── the
                                              ┌────┴──┐ ┌──┴────┐     cheap
   TIER 2 — POD SPINE   1:1 within the pod    │POD 0  │ │POD 1  │     tier
   joins all rails inside one pod             │ SPINE │ │ SPINE │
                                              └┬─┬─┬─┬┘ └───────┘
                                          1:1  │ │ │ │
   TIER 1 — LEAF = THE RAIL     ┌──────┬───────┴─┴─┴─┴──────┬──────┐
   one leaf per GPU position    │LEAF 0│LEAF 1│  …   │LEAF 6│LEAF 7│
   MUST be 1:1 — the            └───┬──┴───┬──┴──────┴───┬──┴───┬──┘
   collectives live here        rail 0  rail 1        rail 6  rail 7
                                    │      │              │      │
   NODES  ── each node contributes GPU-N → mlx5_N → LEAF N ───────┘
                                    │
   INTRA-NODE — NVLink, 450 GB/s/dir, NOT fabric at all.
   Absorbs every cross-rail shuffle so the tiers above never see it.

   TRAFFIC BUDGET BY TIER (LLM training, job placed pod-local):
      NVLink   : the majority of all bytes moved       (free, on-board)
      Leaf/rail: 100% of the inter-node collective     (must be 1:1)
      Pod spine: rail-to-rail residue + control        (1:1, affordable)
      Aggregation: checkpoints, storage, cross-pod jobs (1:7 is fine)
```

Therefore: **the rail (leaf) tier must be non-blocking, because that is where the collectives live; the aggregation tier can be heavily oversubscribed with little impact on training throughput.** You are not trading performance for money — the traffic pattern genuinely does not use the bandwidth you would otherwise buy at the top. That is what "oversubscribe the spine for free" means, and stating it as a traffic-budget argument rather than as a slogan is what makes it credible.

The safety case has one explicit assumption, and you should name it every time you make the argument: **it holds only while jobs are placed to respect the non-blocking domain.** A job whose communicator spans two pods pays the 1:7 ratio on its heaviest traffic, and §7 computes exactly what that costs.

### 6. How traffic actually finds a path: ECMP, flow entropy, and why LLM traffic breaks the assumption

A fat tree gives you many equal-cost paths. Something has to choose among them, and the default answer on an Ethernet fabric is **ECMP** — equal-cost multi-path — which hashes a set of packet header fields and uses the hash to pick an uplink. Hashing (rather than round-robin) is deliberate: every packet of a flow hashes identically, so packets in a flow keep their order, which matters enormously to a transport that recovers from reordering badly.

The mechanism only works if there are **many more flows than paths**. That is where LLM training breaks it, and the mechanics are precise enough to reason about:

- On a RoCEv2 fabric, the hash inputs are the IP five-tuple. The **destination UDP port is always 4791** — it identifies RoCE, not the flow — so it contributes no entropy at all. The entropy comes from the **source UDP port, which the NIC randomises per queue pair** (this is documented behaviour of the RoCEv2 encapsulation and is exactly why the UDP header exists in RoCEv2 at all; Microsoft's SIGCOMM 2016 production paper states it explicitly: destination port always 4791, source port randomly chosen per QP, and intermediate switches use standard five-tuple hashing, so traffic on one QP follows one path).
- So **flow entropy on a RoCE fabric is the number of queue pairs**, not the number of GPUs and not the number of messages.
- And the collective library, by default, uses **one queue pair per connection**: NCCL v2.31.2 sets `NCCL_PARAM(IbQpsPerConn, "IB_QPS_PER_CONNECTION", 1)`. One QP per peer connection means one path per peer connection.

Now put that next to the traffic shape. A rail-local all-reduce among *M* nodes is a ring: each GPU has essentially **one** heavy connection to its ring neighbour, running at close to line rate, starting and stopping in lockstep with every other GPU in the job. So the fabric is asked to spread a handful of ~400 Gb/s flows, all synchronised, across a set of equal-cost uplinks, using a hash over a field whose entropy is the number of QPs.

```
   WHAT ECMP WAS DESIGNED FOR              WHAT LLM TRAINING SENDS
   thousands of small, independent,         a handful of huge, synchronised,
   short-lived flows                        long-lived flows

   uplink 0  ▓▓▓▓▓▓▓▓▓▓  ~equal             uplink 0  ████████████████ 400G
   uplink 1  ▓▓▓▓▓▓▓▓▓▓  ~equal             uplink 1  ████████████████ 400G
   uplink 2  ▓▓▓▓▓▓▓▓▓▓  ~equal             uplink 2  (idle)
   uplink 3  ▓▓▓▓▓▓▓▓▓▓  ~equal             uplink 3  (idle)
        law of large numbers                  two hashes collided; half the
        smooths the imbalance                 tier is saturated, half is idle
```

This is **hash polarisation**, and the crucial point for your mental model is that it is *structural, not unlucky*: with few flows, the variance of a hash-based assignment is large, and there is no averaging effect to rescue you. A fabric with generous aggregate capacity can bottleneck badly while half its uplinks sit idle, and no oversubscription ratio you compute will predict it — **oversubscription tells you how much capacity exists; hashing determines whether that capacity is reachable.** They are different questions and a design has to answer both.

There are three families of fix, and you should be able to name all three:

- **Add entropy.** Use more queue pairs per connection so one logical transfer becomes several hashed flows (`NCCL_IB_QPS_PER_CONNECTION`, and `NCCL_IB_SPLIT_DATA_ON_QPS`, default `0`, which controls whether a message is split across those QPs rather than round-robined whole). More QPs means more paths and better spread, at the cost of more NIC state and more reordering exposure.
- **Change the routing.** Adaptive routing chooses the output port on congestion rather than on a hash — InfiniBand does this in the switch under subnet-manager control, and NCCL exposes `NCCL_IB_ADAPTIVE_ROUTING` (default `-2`, meaning "let NCCL decide based on the fabric") to opt in or out. Ethernet equivalents exist as vendor features (per-packet spraying with reordering handled at the endpoint).
- **Change the topology.** Reduce the number of times a flow gets re-hashed by reducing tiers, or arrange planes so the path choice is structurally constrained rather than hashed. That is the Alibaba answer in §8.

### 7. Worked math: what an oversubscription ratio costs a real collective

Now extend 09.1's model. Its inter-node term assumed each GPU gets the full `B_nic`. Replace that with the bandwidth actually available through the worst tier the job's traffic must cross:

```
   B_eff = B_nic / X      where X is the oversubscription ratio of the
                          most constrained tier the collective crosses
                          (X = 1 for a job that stays inside a
                           non-blocking pod)

   T = 2·(7/8)·S/B_nvl  +  2·(M-1)/M·(S/8)/B_eff
       └── NVLink term ──┘  └───── fabric term ─────┘
```

**Scenario.** A 512-GPU data-parallel job, `M` = 64 nodes, `S` = 1 GB per GPU, H100 + ConnectX-7 (`B_nvl` = 450 GB/s, `B_nic` = 50 GB/s per direction). Compare three placements.

```
  (a) POD-LOCAL — job fits inside one non-blocking pod.  X = 1
      B_eff = 50 GB/s
      NVLink term  = 2 × 0.875 × 1 / 450                = 3.89 ms
      fabric term  = 2 × (63/64) × 0.125 / 50           = 4.92 ms
      step         = 8.81 ms         effective all-reduce = 114 GB/s

  (b) SPREAD ACROSS PODS — every inter-node byte crosses the 1:7
      aggregation tier.  X = 7
      B_eff = 50/7 = 7.14 GB/s
      fabric term  = 2 × (63/64) × 0.125 / 7.14         = 34.5 ms
      step         = 3.89 + 34.5 = 38.4 ms
      effective all-reduce = 26 GB/s      →  4.4× SLOWER than (a)

  (c) SPREAD ACROSS PODS on the tightened 1:2.8 fabric.  X = 2.8
      B_eff = 50/2.8 = 17.9 GB/s
      fabric term  = 2 × (63/64) × 0.125 / 17.9         = 13.8 ms
      step         = 3.89 + 13.8 = 17.7 ms
      effective all-reduce = 57 GB/s      →  2.0× slower than (a),
                                             2.2× faster than (b)
```

Three things to take from this, each of which is a sentence you can say in a design review:

1. **"Keep it in the pod" is worth 4.4× on this job.** That is the placement argument, quantified. It is also why a scheduler that is topology-aware (module 06) is a throughput feature and not an operational nicety.
2. **The ratio maps almost linearly onto step time once the fabric term dominates.** At X = 7 the fabric term is 90% of the step, so the job's speed is essentially `B_nic/X`. That linearity is what makes the ratio such a powerful and dangerous dial.
3. **Partial crossing is the realistic case, and it interpolates.** If a fraction `f` of a job's ranks sit outside the non-blocking domain, the step is set by the slowest rank, so you use `X` for those ranks and `1` for the rest and take the maximum — which means **even a small `f` costs the full penalty**, because a collective is a barrier. One node placed outside the pod slows all 512 GPUs to (b). This is the single most counter-intuitive and most important consequence of barrier-synchronous communication, and it is why "mostly pod-local" is not a meaningful claim.

**Now do the same arithmetic in the cost direction.** Take the 2,048-endpoint pod from §2 and ask what tightening or loosening the aggregation tier costs in parts, using only counting, with no invented prices:

```
   8 pods × 2,048 GPUs = 16,384 GPUs.
   Each pod presents U uplinks to the aggregation tier.

   Full bisection at aggregation (X = 1):
        U = 2,048 per pod → 16,384 aggregation-facing links
                          → 32,768 long-reach transceivers

   X = 7 at aggregation:
        U ≈ 293 per pod   →  2,344 links
                          →  4,688 long-reach transceivers
        → about 86% fewer of the most expensive optics in the build,
          plus a proportionally smaller aggregation switch tier.

   X = 2.8 at aggregation:
        U ≈ 731 per pod   →  5,848 links → 11,696 transceivers
        → 2.5× the optics of X = 7, still 64% below full bisection.
```

That is the trade in its honest form: 1:7 versus full bisection is not "a bit cheaper," it is roughly an order of magnitude of the most expensive component class in the fabric — and per §7(b) it costs 4.4× on any job foolish enough to span it. Both halves of that sentence are needed; quoting either alone produces a bad decision.

### 8. The published designs, and what they disagree about

Three real builds, three different answers. All three sets of figures below come from published sources that could **not** be re-fetched while writing this pass (see the source-access note in References) — treat them as well-known citations to verify, and always attach the year.

**Meta's Llama-3 24,576-GPU RoCE cluster (2024)** is the module's anchor, described in the Llama 3 paper's network section. Its structure as published:

| Tier | Unit | Ratio |
|---|---|---|
| Intra-node | 8 GPUs, NVLink/NVSwitch, ~450 GB/s per direction per GPU | not fabric |
| Rack | 16 GPUs (two servers), one Minipack2 OCP ToR | — |
| Pod | 192 racks = **3,072 GPUs**, full bisection | **1:1, non-blocking** |
| Cluster | **8 pods = 24,576 GPUs** at the aggregation layer | **1:7 oversubscribed** |

Each GPU has a 400G RoCE NIC. The design decision in one sentence: **non-blocking inside the 3,072-GPU pod where the collectives run, 1:7 between pods where they rarely need to go** — and it cost little because jobs were *placed* to respect pod locality, which is §7's point (a) rather than a happy accident.

> **The 1:7 ratio is a snapshot, not a law.** It is real, published and specific to that cluster and that traffic pattern. Meta's own 2024 engineering write-up on RoCE networks for distributed AI training describes tightening the cross-domain ratio to roughly **1:2.8** on newer infrastructure, to give multi-dimensional parallelism more cross-domain bandwidth as world sizes and parallelism strategies grew. The lesson is not "1:2.8 is now correct" either; it is that the safe ratio **moves with the traffic pattern, the parallelism plan, and how much reduction happens in-network** (SHARP, 09.5). Quote 1:7 as the Llama-3-era figure, always with the year attached, and be ready to say why a newer build might not use it.

**Alibaba HPN (SIGCOMM 2024)** is the counter-example that matters most, because it went in the direction a naive reading would not predict: **fewer tiers**. HPN interconnects **15,000 GPUs in a single pod** using a **2-tier, dual-plane** architecture with twin top-of-rack switches, where the textbook answer would have been three tiers. The driver was not cost — it was §6's failure mode. LLM training generates a small number of periodic, bursty flows near 400 Gb/s per host, which predisposes ECMP to hash polarisation. Fewer tiers means fewer re-hashing events and a smaller path-selection space, so the fabric can steer elephant flows more deterministically; the dual-plane arrangement gives each rail two structurally separate paths rather than a hashed choice among many; and dual ToR per rack removes the single point of failure that a 2-tier design would otherwise introduce.

**Third-party analyses of 100,000-GPU builds** describe another shape again: several very large non-blocking domains (in one widely-circulated 2024 analysis, four domains of roughly 24,576 GPUs each) joined at 1:7. Note the terminology overload: "pod" means 3,072 GPUs in Meta's description and roughly 24,576 in that analysis. **Always check which non-blocking unit a source means before quoting its ratio**, because the ratio is meaningless without the unit it applies to.

The design-space lesson, which is the actual interview answer to "why not always 3-tier Clos?": **tier count and oversubscription ratio are traffic-pattern-driven engineering choices, not universal constants.** One team oversubscribed the top tier 7:1 and placed jobs to avoid it. Another deleted a tier entirely to fight a routing pathology. A third built enormous non-blocking domains and accepted the optics bill. All three are defensible; none is "the" answer.

### 9. The generation numbers to anchor procurement

When you argue fabric, quote current silicon and mark it as a dated snapshot:

| Component | Generation | Line rate / radix |
|---|---|---|
| InfiniBand switch | Quantum-2 | NDR 400G, 64 ports |
| InfiniBand switch | Quantum-X800 | XDR 800G |
| Ethernet switch | Spectrum-4 (SN5600) | 64 × 800G or 128 × 400G, 51.2 Tb/s |
| NIC | ConnectX-7 | 400 Gb/s = 50 GB/s per direction |
| NIC | ConnectX-8 | 800 Gb/s = 100 GB/s per direction, PCIe Gen6 x16 |
| DPU | BlueField-3 | offload, congestion control, tenant isolation |

Two camps sit behind those rows. **InfiniBand** (Quantum + ConnectX) is lossless by construction via credit-based link-level flow control, with a centralised subnet manager and in-network reduction (SHARP). **RoCE Ethernet** (Spectrum + ConnectX/BlueField) reuses the Ethernet control plane you already run — BGP/EVPN, ECMP, your existing telemetry — and manufactures losslessness with PFC and ECN/DCQCN. Meta's 24K RoCE cluster is the existence proof that a well-built Ethernet fabric trains frontier models; InfiniBand remains common where the team wants determinism without a tuning project. **Why RoCE needs any of that machinery is 09.4's subject**; here the rows exist so your radix and line-rate arithmetic uses real numbers.

### 10. What a constrained tier looks like from inside the job: a step timeline

Bandwidth arithmetic tells you the average. What you actually observe is a *timeline*, and the shape of that timeline is what makes fabric problems so hard to attribute — the symptom appears on GPUs that are not the problem.

```
  ONE TRAINING STEP, 8 RANKS SHOWN OF 512.  Time runs to the right.
  ▓ = compute (forward+backward)   ▒ = collective traffic   · = STALLED at barrier

  (a) HEALTHY, pod-local, every rail 1:1
  rank 0  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒│
  rank 1  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒│
  rank 2  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒│      all ranks finish the all-reduce
  rank 3  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒│      together; the barrier costs nothing
   …                            │
  rank 7  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒│
                                └─ step boundary: 8.9 ms

  (b) SIX RANKS CROSS A 1:7 TIER (job split across pods)
  rank 0  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒·······························│
  rank 1  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒·······························│
  rank 2  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│ ← crosses
  rank 3  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│   the 1:7
   …                                                          │   tier at
  rank 7  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒·······························│   7.1 GB/s
                                                              └─ 38.8 ms
        The ranks that are FINE spend 76% of the step stalled.
        Their GPU utilisation counters look busy; their SMs are idle.

  (c) ECMP HASH COLLISION on an otherwise non-blocking tier
  rank 0  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒················│
  rank 1  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒················│
  rank 2  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│ ← its flow shares an
  rank 3  ▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒▒▒▒▒▒················│   uplink with rank 2's
   …                                            │   while other uplinks idle
                                                └─ ~2× the healthy step
        Aggregate capacity is FINE.  Utilisation of the tier is ~50%.
        No counter anywhere says "oversubscribed", because nothing is.
```

Three diagnostic lessons come out of that picture, and they are the ones that separate a useful incident review from a shrug:

1. **The stall is where the problem is not.** In (b), six of eight ranks are idle for three quarters of the step because two ranks are slow. Any per-rank profile will show most GPUs "waiting on NCCL," which tells you nothing about *which* link is the constraint. The way to attribute it is to correlate the slow ranks with the topology — which pod, which rail, which leaf — not to look harder at the stalled ones.
2. **(b) and (c) look identical from the GPU and require opposite fixes.** Both present as an inflated collective time with most ranks blocked. (b) is a capacity problem and the fix is placement (or a different ratio). (c) is an entropy problem and the fix is more queue pairs, adaptive routing, or a topology change. Distinguishing them requires switch-side data: in (b) the crossed tier's uplinks are all saturated; in (c) some are saturated and others idle. **The tell is the *variance* across uplinks of one tier**, and that is the single most useful fabric metric to have on a dashboard for a training cluster.
3. **The barrier turns a local problem into a global bill.** In every panel, the step time is set by the slowest path, so the cost is `(number of GPUs in the job) × (step-time increase)` in GPU-hours, not the affected fraction. That is the arithmetic that justifies both topology-aware scheduling and the fabric telemetry needed to tell (b) from (c).

### 11. Adaptive routing: fixing what hashing broke, and what it costs

§6 left three families of fix; the middle one deserves its own mechanism, because it is the main structural difference between how an InfiniBand fabric and a default Ethernet fabric behave under collective traffic.

**Static hashing** decides a packet's output port from header fields alone. It is stateless, order-preserving per flow, and blind to congestion: if the chosen uplink is full, the packet queues there while a parallel uplink sits idle.

**Adaptive routing** lets the switch choose among the equal-cost output ports using *observed state* — output-queue occupancy, credits available on the next hop — rather than a hash. On InfiniBand the set of legal alternative ports is computed by the subnet manager as part of routing, and the switch picks among them per packet or per burst. The consequence is that a hot uplink drains onto its neighbours automatically, which is exactly the failure mode in panel (c) above.

The cost is **reordering**. Packets of one message may arrive out of order, and the transport has to cope. That is why adaptive routing is a fabric-plus-endpoint feature rather than a switch feature: the receiving NIC must either tolerate out-of-order placement or reassemble, and the collective library has to be told it is safe. NCCL exposes this as `NCCL_IB_ADAPTIVE_ROUTING`, whose default in v2.31.2 is `-2` — "decide automatically based on what the fabric reports" — and there is a companion `NCCL_IB_OOO_RQ` (default `0`) governing out-of-order receive queues. Enabling adaptive routing on a fabric or a NIC generation that does not handle reordering well converts a bandwidth problem into a correctness-adjacent performance problem, which is why it is a fabric-team decision rather than a job-level flag.

The Ethernet equivalents are vendor features rather than a standard: per-packet spraying with endpoint reordering, or flowlet-based load balancing that re-hashes only at gaps in a flow (which works because a gap means the previous burst has drained, so reordering risk is bounded). Where a vendor sells "AI-optimised Ethernet," this — plus congestion control, which is 09.4 — is usually most of what is in the box.

**The procurement-relevant summary:** a fabric's usable bandwidth is `capacity × how evenly traffic can be spread across it`. Oversubscription sets the first term, routing sets the second, and a design that gets one right and the other wrong delivers neither.

## Perspectives

**Theory.** The Clos and bisection mathematics is topology-agnostic graph theory — the same whether you are wiring a 24K-GPU AI cluster or a 1960s telephone exchange, which is literally where Clos networks come from. Radix bounds fan-out, tiers multiply fan-out, and bisection measures the worst cut. None of that changes. What changes across contexts is *which tier the traffic pattern lets you starve*, and that is an empirical question about workloads, not a theorem.

**Practice.** Real fabric teams reason from measured traffic, not from reference diagrams. Alibaba's move to two tiers was driven by a measured flow-size distribution and an observed hashing pathology, not by a cost model. Meta's 1:7 was justified by a placement policy they controlled and then revised when the parallelism strategy changed. The transferable habit is that **every ratio in a fabric design should have a traffic measurement behind it**, and if nobody can produce that measurement, the ratio is a guess wearing a number.

**Operator.** Oversubscription is invisible in normal operation and brutal at the boundaries. The tier ratios are baked in at build time, so the only lever you have afterwards is *placement* — which makes the scheduler's topology awareness the operational control surface for a decision that was made in a cabling plan a year earlier. Practically: know which switch each node's rails land on, expose that as node labels, and make it schedulable. A cluster where nobody can answer "which pod is this node in?" cannot honour the design its fabric was built around.

**Economics.** The bill is dominated by things that scale with tier count: long-reach optics, the switches to terminate them, and the power and cooling to run both. §7's parts count is the shape of the argument — 1:7 versus 1:1 at the top tier of a 16K-GPU build is roughly an 86% reduction in long-reach transceivers. The corresponding risk is equally concrete: that saving is only real while jobs stay inside the non-blocking domain, so the fabric's cost model and the scheduler's placement policy are the same decision viewed from two sides.

## Real-world use cases

- **Meta, Llama 3 (2024) — the anchor.** [The Llama 3 Herd of Models](https://arxiv.org/abs/2407.21783), network section. What it shows: a published 24,576-GPU RoCEv2 three-tier Clos with 3,072-GPU non-blocking pods and 1:7 aggregation oversubscription, plus the explicit statement that job placement respected pod locality. It is the canonical worked example for the deliverable, and the one to redraw with per-tier ratios.
- **Meta Engineering (Aug 2024) — the revision.** [RoCE networks for distributed AI training at scale](https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/). What it shows: the same organisation tightening its cross-domain oversubscription from 1:7 toward roughly 1:2.8 on newer infrastructure — the best available evidence that these ratios are design variables tracking workload evolution, not constants.
- **Alibaba HPN (SIGCOMM 2024) — the counter-design.** [Alibaba HPN: A Data Center Network for Large Language Model Training](https://dl.acm.org/doi/10.1145/3651890.3672265). What it shows: 15,000 GPUs in one pod on a 2-tier dual-plane architecture with dual ToR, chosen specifically because LLM training's small number of ~400 Gb/s bursty flows predisposes ECMP to hash polarisation. Fewer tiers as a *correctness* fix, not a cost cut.
- **ByteDance MegaScale (NSDI '24) — independent validation.** [MegaScale: Scaling LLM training to more than 10,000 GPUs](https://www.usenix.org/conference/nsdi24/presentation/jiang-ziheng). What it shows: a production three-layer rail-optimised Clos at 12,288 GPUs achieving 55.2% MFU — the same design family as Llama-3's, from an unrelated team, which is what makes the pattern credible rather than idiosyncratic.
- **Microsoft Azure production RoCE (SIGCOMM 2016) — the mechanics of paths.** [RDMA over Commodity Ethernet at Scale](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/rdma_sigcomm2016.pdf). What it shows, and what was verified directly from the paper for this lesson: RoCEv2's destination UDP port is always 4791 while the **source port is randomised per queue pair**, and intermediate switches use standard five-tuple hashing, so **traffic on one QP follows one path** — the mechanical basis of §6's entropy argument. The paper also documents the physical distances that make a fabric a fabric: ~2 m server-to-ToR copper, 10–20 m ToR-to-leaf, and **200–300 m leaf-to-spine**, which is where long-reach optics and their cost live.

## Worked example

**Read a published fabric, tier by tier, and predict where a named job bottlenecks.** This is the deliverable in miniature; do it once here and the artefact is most of the way written.

**Step 1 — redraw the topology with counts and ratios on every tier.**

```
  META LLAMA-3 CLUSTER (2024, as published) — 24,576 × H100, RoCEv2

  ┌──────────────────────────────────────────────────────────────────┐
  │ AGGREGATION                                       ratio  1:7      │
  │ joins 8 pods → 24,576 GPUs                                       │
  └───┬────────────┬────────────┬───────  …  ────────┬───────────────┘
      │            │            │                    │
  ┌───┴───┐    ┌───┴───┐    ┌───┴───┐            ┌───┴───┐
  │ POD 0 │    │ POD 1 │    │ POD 2 │    …       │ POD 7 │   8 pods
  │3,072  │    │3,072  │    │3,072  │            │3,072  │
  │ GPUs  │    │ GPUs  │    │ GPUs  │            │ GPUs  │
  │ 1:1   │    │ 1:1   │    │ 1:1   │            │ 1:1   │  ← non-blocking
  └───┬───┘    └───────┘    └───────┘            └───────┘
      │  192 racks per pod
  ┌───┴──────────────┐
  │ ToR (Minipack2)  │  16 GPUs per rack = 2 servers        ratio 1:1
  └───┬──────────────┘
      │
  ┌───┴──────────────┐
  │ SERVER: 8 × H100 │  NVLink/NVSwitch, 450 GB/s per dir per GPU
  │ 8 × 400G RoCE NIC│  NOT fabric — absorbed before the wire
  └──────────────────┘

  ENDPOINT DEMAND PER POD:  3,072 × 400 Gb/s = 1.229 Pb/s
  CLUSTER DEMAND:           24,576 × 400 Gb/s = 9.83 Pb/s
```

**Step 2 — state the bisection at each tier, in the form "worst case per GPU."**

| Tier | Ratio | Worst-case per-GPU bandwidth across it | What crosses it |
|---|---|---|---|
| NVLink domain (intra-node) | n/a | 450 GB/s per direction | cross-rail shuffles, intra-node collective phases |
| ToR / rail (leaf) | 1:1 | 400 Gb/s = 50 GB/s | 100% of the inter-node collective, rail-local |
| Pod spine | 1:1 | 400 Gb/s = 50 GB/s | rail-to-rail residue inside the pod |
| Aggregation | **1:7** | 400/7 ≈ **57 Gb/s = 7.1 GB/s** | cross-pod jobs, checkpoints, storage, eval |

**Step 3 — pick a named job and predict two placements.** Take a 2,048-GPU all-reduce-heavy training run, `S` = 1 GB of gradients per GPU, `M` = 256 nodes.

```
  PLACEMENT 1 — entirely inside one 3,072-GPU pod
      every inter-node byte crosses ToR and pod spine only, both 1:1
      B_eff = 50 GB/s
      NVLink term = 2 × 0.875 × 1 / 450                    = 3.89 ms
      fabric term = 2 × (255/256) × 0.125 / 50             = 4.98 ms
      step        = 8.87 ms      effective all-reduce      = 113 GB/s
      BOTTLENECK: the NIC / rail tier — and it is the designed bottleneck.
      Nothing above the pod is touched.

  PLACEMENT 2 — split across two pods (e.g. the scheduler packed
                what it could find, 1,024 GPUs in each pod)
      the ring crosses the aggregation tier on every lap
      B_eff = 50/7 = 7.14 GB/s
      fabric term = 2 × (255/256) × 0.125 / 7.14           = 34.9 ms
      step        = 3.89 + 34.9 = 38.8 ms
      effective all-reduce                                  = 26 GB/s
      BOTTLENECK: the 1:7 aggregation tier.
      PENALTY: 4.4× slower step; the job needs 4.4× the wall-clock
               and therefore 4.4× the GPU-hours for the same number
               of steps.
```

**Step 4 — say the placement argument in one sentence with the number in it.** *"Keeping this 2,048-GPU job inside a single 3,072-GPU pod preserves 50 GB/s per GPU of inter-node bandwidth and an 8.9 ms all-reduce; spreading it across two pods caps the collective at the 1:7 aggregation tier — 7.1 GB/s per GPU — and takes the step to 38.8 ms, a 4.4× regression, because a collective is a barrier and every rank waits for the slowest."*

**Step 5 — sanity-check the argument against its own assumptions.** Three checks, all of which a good reviewer will run:

- *Does the job actually fit in a pod?* 2,048 ≤ 3,072, yes. A 4,096-GPU job does not, and for that job the honest answer is "it must cross aggregation, so either the ratio needs revisiting for this workload class or the parallelism plan must be arranged so the cross-pod dimension carries the *lightest* traffic" — which is exactly why multi-dimensional parallelism maps its pipeline stages, not its data-parallel all-reduce, across the expensive tier.
- *Is the fabric term really dominant?* At `B_eff` = 7.14 GB/s the fabric term is 90% of the step, so yes — and that means the ratio maps nearly linearly onto step time. At `B_eff` = 50 GB/s it is 56%, so improvements to either term matter.
- *Is anything else stealing rail bandwidth?* Checkpoint writes and dataset staging on a rail NIC show up as stragglers. This is what the off-rail storage/management NIC from 09.1 §9 exists to prevent, and it is worth naming in the write-up because a reviewer who knows the reference architecture will look for it.

**Step 6 — recompute for the revised ratio.** If the cross-domain ratio is tightened to 1:2.8, Placement 2's `B_eff` becomes 17.9 GB/s, its fabric term 13.9 ms, its step 17.8 ms, and its effective all-reduce 56 GB/s — 2.0× slower than pod-local rather than 4.4×. Same formula, one input changed: that is the shape of the argument to carry into a procurement conversation, because it lets you price a ratio against a workload instead of arguing about it in the abstract.

## Practice

Feeds the deliverable **Network architecture read**.

**Task.** From the published Llama-3 cluster description (or the InfiniBand variant of your choice), **redraw the tiers and compute the ratios** — then contrast the design with Alibaba HPN's two-tier answer.

**Requirements / acceptance:**

1. A **tier-by-tier topology sketch** (ASCII is fine and is what the deliverable expects) showing GPUs per server → GPUs per rack → ToR → pod spine → aggregation, with the real counts: 8 GPUs per server, 16 per rack, 192 racks = 3,072-GPU pod, 8 pods = 24,576 GPUs.
2. **Label every tier with its ratio and its worst-case per-GPU bandwidth**: intra-node NVLink (not fabric, 450 GB/s per direction); leaf/rail and pod spine at 1:1 (50 GB/s per GPU); aggregation at 1:7 (≈7.1 GB/s per GPU), marked as the 2024 published figure.
3. A short derivation of **why the pod is non-blocking and the cluster is not** — uplink capacity versus downlink capacity at each tier — and why that split is defensible for this workload (rail locality plus pod-local placement).
4. A **bottleneck prediction for one named job under two placements**, quantified with the ratio, following the six steps of the worked example. State the penalty as a step-time multiplier *and* as a GPU-hours consequence.
5. Two to three sentences on **Alibaba HPN's 2-tier dual-plane alternative**: what ECMP hash polarisation is in mechanical terms (flow entropy comes from queue pairs, one QP per connection by default, a handful of synchronised ~400 Gb/s flows), and why "fewer tiers" was the fix rather than "more oversubscription" or "more tuning."
6. One paragraph on **what would change your answer**: name at least two conditions under which the 1:7 ratio stops being safe for a workload you care about (for example, a job larger than the non-blocking domain; a parallelism plan whose heaviest dimension spans pods; heavy cross-pod checkpoint traffic sharing the tier).

Combine this with 09.1's GPU→rail→NIC table and bandwidth ladder and you have the complete intra-plus-inter architecture read the deliverable asks for.

## Common pitfalls

- **Quoting a ratio without naming the tier and the year.** "1:7" is meaningless on its own. It was the *aggregation* ratio of a *specific 2024* cluster whose non-blocking unit was 3,072 GPUs, and the same organisation has described tightening it to roughly 1:2.8 since. A ratio without a tier, a unit and a date is not a fact, it is a rumour.
- **Assuming "pod" means the same thing across sources.** Meta's pod is 3,072 GPUs; a widely-circulated 2024 analysis of 100K-GPU builds uses "pod" for roughly 24,576-GPU non-blocking domains. The oversubscription number attached to each is only interpretable relative to its own unit. Always resolve the unit before comparing ratios.
- **Computing bisection only for the happy, rail-local path.** All the reassuring arithmetic assumes the job stays inside the non-blocking domain. The moment placement or the communicator spans it — a cross-pod evaluation run, a checkpoint to storage outside the pod, a job larger than the pod — the governing number is the *oversubscribed-tier* number. Always state which case a bandwidth claim assumes.
- **Treating "mostly pod-local" as nearly as good as pod-local.** A collective is a barrier: the step time is set by the slowest rank. One node placed outside the non-blocking domain imposes the crossed-tier penalty on the entire job. There is no partial credit in the arithmetic, which is why gang scheduling with topology constraints is a correctness feature, not an optimisation.
- **Ignoring ECMP hash polarisation as a failure mode distinct from the ratio.** Oversubscription tells you how much capacity exists; hashing determines whether it can be reached. A non-blocking tier whose uplinks are chosen by a five-tuple hash over a workload with a handful of synchronised elephant flows can run at half its capacity with no oversubscription anywhere. The fixes live in different places — more queue pairs, adaptive routing, or a different topology — and none of them is "buy more uplinks."
- **Assuming more tiers always means more scale.** Alibaba went to *fewer* tiers at 15,000 GPUs per pod, deliberately, to reduce re-hashing and constrain path selection. Tier count is a design variable driven by radix, distance and routing behaviour, not a monotone function of cluster size.
- **Forgetting that the fabric term has a floor.** The ring factor `2(M−1)/M` converges to 2, so the inter-node term stops growing with node count. If a job's step time is growing linearly with scale, the cause is not the algorithm — look for a crossed oversubscribed tier, congestion, or a straggler.

## Self-check

**(a) A 4:1-oversubscribed spine — what is the worst-case per-GPU bandwidth for an all-to-all spanning two leaves, versus within one leaf?**

**Answer:** Within one leaf the traffic never touches the uplinks, so it runs at full line rate — 400 Gb/s = 50 GB/s per GPU on a 400G fabric — and the oversubscription ratio is irrelevant to it. Spanning two leaves, every byte must cross the 4:1 tier and share the reduced uplinks: worst case is `line rate ÷ 4` = 100 Gb/s = 12.5 GB/s per GPU, assuming all endpoints under the tier are active across it. The general rule is that an X:1 tier taxes only the traffic that crosses it, at `line rate ÷ X` in the worst case — which is precisely why keeping collectives rail- and pod-local makes an aggressive ratio harmless.

**(b) Why can rail-optimised designs oversubscribe the spine "for free" for LLM training?**

**Answer:** Because of the traffic budget. The bandwidth-dominant collectives run GPU-N ↔ GPU-N, so they are rail-local: they ride a single leaf and terminate there, never reaching the spine. The only traffic that would naturally cross rails is shuffled sideways over NVLink inside the node first (450 GB/s per direction against the NIC's 50), so it too leaves on its own rail. The tiers above the leaf therefore carry only the residue — cross-pod jobs, checkpoint and storage I/O, evaluation, control-plane traffic — which is a small fraction of total bytes and is not barrier-synchronous. Oversubscribing that tier removes capacity the training pattern does not consume, so throughput is unaffected while the count of long-reach optics (the dominant cost class) falls sharply. The argument's one assumption, which must be stated: it holds only while jobs are placed to stay inside the non-blocking domain.

**(c) What does "full bisection bandwidth" mean, how do you compute it, and why is it expensive?**

**Answer:** Cut the network into two equal halves; the bisection bandwidth is the aggregate capacity of the links crossing that cut, minimised over all such cuts. *Full* or non-blocking bisection means uplink capacity equals downlink capacity at the tier, so demand across the worst cut equals capacity across it and every endpoint can send at line rate simultaneously — a 1:1 ratio. Computing it is counting: for the 2,048-endpoint pod built in §2, a cut separating 1,024 GPUs must cross 1,024 leaf-to-spine links at 400 Gb/s = 409.6 Tb/s, exactly matching the 1,024 × 400 Gb/s of demand. It is expensive because every tier must carry the sum of everything beneath it, so uplink count, spine switch count and — dominant at scale — optical transceivers all scale with that sum: the §2 build needs about 4,096 transceivers for its inter-switch links alone, roughly two per GPU per tier. At the top tier of a 16,384-GPU build, going from full bisection to 1:7 removes roughly 86% of the long-reach transceivers.

**(d) Why did Alibaba build a 2-tier dual-plane fabric instead of a 3-tier Clos for a 15,000-GPU pod?**

**Answer:** To eliminate ECMP hash polarisation structurally rather than react to it. LLM training generates a small number of large, bursty, periodic flows near 400 Gb/s per host. ECMP picks an uplink by hashing packet headers, which works when there are far more flows than paths; with a handful of synchronised elephant flows there is no averaging effect, so collisions leave some uplinks saturated while others idle. Fewer tiers means fewer re-hashing events and a smaller path-selection space, so flows can be steered onto paths sized to hold them; the dual-plane arrangement gives each rail structurally separate paths instead of a hashed choice; and dual ToR per rack removes the single point of failure a two-tier design would otherwise create. It is a routing-correctness fix, not a cost cut — which is why it is the sharpest available counter-example to "more tiers scale better."

**(e) On a RoCE fabric, what determines how many equal-cost paths a collective's traffic can actually use?**

**Answer:** The number of queue pairs. RoCEv2 encapsulates RDMA in UDP with a **fixed destination port of 4791**, so the destination port contributes no hash entropy; the entropy comes from the **source UDP port, which the NIC randomises per queue pair**, and switches hash the five-tuple, so all traffic on one QP follows one path (Microsoft's 2016 production RoCE paper states exactly this). NCCL defaults to **one QP per connection** (`NCCL_IB_QPS_PER_CONNECTION` = 1 in v2.31.2), so a ring all-reduce presents roughly one heavy flow per rank per direction. That is why raising `NCCL_IB_QPS_PER_CONNECTION` (optionally with `NCCL_IB_SPLIT_DATA_ON_QPS`) spreads a collective across more paths, and why adaptive routing — which selects the output port on observed congestion instead of on a hash — is the structural alternative. Oversubscription ratio and path entropy are independent properties: you need both to be right.

**(f) If Meta's published 1:7 ratio has since moved toward roughly 1:2.8, how should you use the 1:7 number?**

**Answer:** As a dated, well-sourced example, never as a constant. The safe ratio tracks the traffic pattern: world size, the parallelism strategy and which dimension of it crosses the tier, how much reduction happens in-network (SHARP), and the placement policy the scheduler can actually enforce. 1:7 was defensible for the Llama-3-era cluster because jobs were placed pod-local and the pod was 3,072 GPUs — large enough to hold the jobs of that era. As world sizes grew past the non-blocking domain, the cross-domain tier stopped being a rarely-used residue path and started carrying real collective traffic, which is exactly the condition under which §7's arithmetic says the ratio maps almost linearly onto step time. The correct answer in an interview names 1:7 with its year and unit, then immediately adds that the ratio is a design variable tuned per traffic pattern, and cites the revision as evidence rather than asserting it.

## Connections & what's next

This lesson turns 09.1's rail into a cost argument: the same rail locality that told you which NIC to use now tells you which tier can be oversubscribed and by how much. It also reaches forward twice. **09.4 (IB vs RoCE and lossless)** picks up the line rates and the ECMP mechanics from §6 and asks the question this lesson deliberately deferred — how an Ethernet fabric carries RDMA traffic without dropping it, and what PFC, ECN and DCQCN cost you in tuning risk. **09.5 (GPUDirect and SHARP)** attacks the fabric term from the other side: in-network reduction changes how many bytes a collective puts on the wire at all, which shifts the ratio a workload can tolerate. And backwards, **06's topology-aware scheduling** is the mechanism that makes §7's placement argument enforceable rather than aspirational.

The next lesson, **09.3 (RDMA fundamentals)**, changes altitude rather than scope. Instead of asking what shape the network is, it asks how a byte actually gets from one GPU's memory into another's without the CPU in the path: queue pairs, memory registration, work requests, completion queues, and exactly which stages of the kernel networking datapath disappear. Carry the Clos vocabulary and the oversubscription arithmetic forward; 09.3 explains the mechanism that lets the non-blocking tier you just sized actually deliver the bandwidth you computed.

## References & further reading

**Source-access note for this pass.** Several publisher and vendor domains (arxiv.org, dl.acm.org, usenix.org, docs.nvidia.com, engineering.fb.com, IEEE) are blocked by this environment's egress proxy and could not be fetched while writing. One primary source *was* reachable and was read directly — Microsoft's SIGCOMM 2016 RoCE paper — and the facts drawn from it are marked below. Entries marked **not re-verified in this pass** carry their original attribution from the previous version of this lesson; treat their figures as citations to check rather than as verified-here numbers. Radix and line-rate figures are 2026-era snapshots and will move with silicon generations.

**Verified against source in this pass**

- Guo et al., Microsoft, [RDMA over Commodity Ethernet at Scale](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/rdma_sigcomm2016.pdf), SIGCOMM 2016 — read directly for this lesson. The substance used here: RoCEv2's destination UDP port is always 4791 while the source port is randomly chosen per queue pair, and intermediate switches use standard five-tuple hashing, so traffic belonging to one QP follows one path while different QPs between the same endpoints may take different paths — the mechanical basis of §6. Also the physical distances of a real three-tier fabric: ~2 m of copper server-to-ToR, 10–20 m ToR-to-leaf, and 200–300 m leaf-to-spine.
- NCCL v2.31.2 source (`github.com/NVIDIA/nccl`) — read directly. Verified here: `NCCL_PARAM(IbQpsPerConn, "IB_QPS_PER_CONNECTION", 1)` and `NCCL_PARAM(IbSplitDataOnQps, "IB_SPLIT_DATA_ON_QPS", 0)` in `src/transport/net_ib/common.cc` and `connect.cc`, and `NCCL_PARAM(IbAdaptiveRouting, "IB_ADAPTIVE_ROUTING", -2)` in `src/transport/net_ib/init.cc`. Read for: the actual default flow entropy a collective presents to an ECMP fabric.
- Course lesson 09.1 — the source of the 9:1 cliff, the rail definition, the per-generation NVLink and NIC figures, and the three-phase all-reduce model this lesson extends with an oversubscription term.

**Cited, not re-verified in this pass**

- Meta, [The Llama 3 Herd of Models](https://arxiv.org/abs/2407.21783), network section — cited for the anchor build: 24,576-GPU RoCEv2 cluster, 8 GPUs per server and 16 per rack, 192 racks forming a 3,072-GPU non-blocking pod, 8 pods joined at 1:7 aggregation oversubscription, with placement respecting pod locality.
- Meta Engineering, [RoCE networks for distributed AI training at scale](https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/) (Aug 2024) — cited for the reported tightening of the cross-domain oversubscription ratio from 1:7 toward roughly 1:2.8, the primary evidence that the ratio is a moving design variable.
- Qian et al., Alibaba, [HPN: A Data Center Network for Large Language Model Training](https://dl.acm.org/doi/10.1145/3651890.3672265), SIGCOMM 2024 — cited for the 2-tier dual-plane architecture interconnecting 15,000 GPUs in one pod with twin ToR switches, and for hash polarisation under a small number of periodic ~400 Gb/s bursty flows as the motivating failure mode.
- Jiang et al., ByteDance, [MegaScale: Scaling LLM training to more than 10,000 GPUs](https://www.usenix.org/conference/nsdi24/presentation/jiang-ziheng), NSDI '24 — cited for a production three-layer rail-optimised Clos at 12,288 GPUs reporting 55.2% MFU.
- Wang, Ghobadi et al., [Rail-only: a low-cost, high-performance network for training LLMs with trillion parameters](https://arxiv.org/abs/2307.12169) — cited for the academic form of the rail-locality argument: because LLM traffic is rail-local, a fabric with a slimmed spine can match full-bisection performance at a fraction of the cost.
- NVIDIA, [HGX AI Factory reference architecture](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/) — cited for the generation table in §9 (Quantum-2/X800, Spectrum-4 SN5600 radix and 51.2 Tb/s, ConnectX-7/8, BlueField-3) and a vendor-blessed leaf-spine reference diagram.
- SemiAnalysis, [100,000 H100 clusters: power, network topology, Ethernet vs InfiniBand](https://newsletter.semianalysis.com/p/100000-h100-clusters-power-network) — cited for an alternative shape at 100K-GPU scale (several ~24,576-GPU non-blocking domains joined at 1:7) and for the terminology-overload warning about the word "pod." Third-party 2024 analysis, not a vendor or peer-reviewed primary source.

**Deeper dives**

- Glenn K. Lockwood, [Networking for LLM training — practitioner notes](https://www.glennklockwood.com/garden/networking-for-LLM-training) — an independent, deliberately sceptical treatment of why LLM fabrics do not need a fully non-blocking fat tree, and which topologies are defensible instead. Not re-verified in this pass.
- At Scale Conference, [Alibaba HPN talk recording](https://atscaleconference.com/videos/alibaba-hpn-a-data-center-network-for-large-language-model-training/) — the HPN authors presenting the dual-plane design and the hash-polarisation problem in their own words. Not re-verified in this pass.
