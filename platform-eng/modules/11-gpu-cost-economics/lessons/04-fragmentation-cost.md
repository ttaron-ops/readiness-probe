---
lesson: 04
title: "Fragmentation: the cost of GPUs you can't schedule"
module: 11
concept: "Unschedulable free capacity"
status: not-started
est_time: "4.5 hrs"
prev: "03-idle-detection.md"
next: "05-unit-economics.md"
artifacts: ["Fragmentation-ratio + stranded-$ calculation over a request-shape histogram, plus a packing-policy recovery estimate, added to the gpu-cost synthesis deliverable"]
sources: 14
---

# Fragmentation: the cost of GPUs you can't schedule

> Module: [💰 11 — GPU cost and unit economics](../README.md) · Deliverable: [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md)

## Where this fits

Lesson 03 took gap B — capacity a tenant holds and does not use — and split it into states you can act on. It deliberately handed off gap A: the paid-for capacity that **nobody holds**. On lesson 02's worked fleet that was $6,028 a week, and the report line said "→ platform (L04)". This is L04.

The instinct is that gap A is easy. Nobody has it, so give it to someone. That instinct is wrong for a specific and mechanical reason: **a GPU is only usable if its shape matches the shape of something waiting to run.** Free GPUs scattered one and two at a time across a hundred nodes cannot host an eight-way tensor-parallel job. A GPU carved into seven MIG slices cannot host a whole-GPU request until it is drained. A GPU with 190 milli-GPU of capacity left after a sharing pod took 810 cannot serve any of the asks that actually arrive. In every case the dashboard says *free* and the scheduler says *pending*, and you pay full price for both.

This lesson makes that quantity measurable and priced. It defines fragmentation as an **expectation over a demand distribution** rather than a single number, computes it against a real production trace you can download and re-run, and prices both the stranded capacity and the defragmentation that would recover it. Module 06's lesson 07 built the per-job-size placeability formula; this lesson keeps that arithmetic exactly and adds the two things a cost owner needs on top: the distribution, and the money.

Lesson 05 then needs this number, because a unit cost computed on directly-attributed GPU-hours understates true cost by exactly the fragmentation tax — you pay for the whole fleet including the part nothing can be scheduled onto.

## Why this matters

Fragmentation is the only large cost line that is invisible on every naive dashboard *by construction*. Idle capacity at least shows up as a low utilisation number; fragmented capacity shows up as **green**. Every node looks busy, the fleet reads 90% allocated, and the queue at the head of your training pipeline has been pending for six days because no node has eight free GPUs. "We're basically full, buy more" is the sentence that follows, and it is how a GPU organisation spends a seven-figure capital line on capacity it already owns.

The magnitude is not speculative. Against Alibaba's published production trace — 1,213 GPU nodes, 6,212 physical GPUs, 8,152 tasks, fetched and replayed in §6 — a single placement-policy change moves **795 GPU-equivalents** between "stranded" and "usable" with no hardware bought and no workload changed. At a rate of `r` per GPU-hour that is `795 × r` per hour, forever, and every term of it is a scheduler config decision.

The reason this is a *platform-engineering* skill rather than a scheduling-research one is that the fix is almost always free and the diagnosis is almost always missing. Kubernetes' default node scoring is `LeastAllocated` — spread — over `cpu` and `memory` only; **`nvidia.com/gpu` is not in the default scoring resource set at all**, so out of the box the scheduler makes placement decisions that are actively hostile to GPU gang placement and does not even look at the resource you care about. Flipping that is a config change. Knowing to flip it, and being able to put a number on what the flip is worth, is the differentiator.

And there is a trap on the other side, which this lesson prices too: defragmentation is not free. Consolidating a fleet means evicting running work. Fragmentation is a **stock**; defrag is a **flow with a price**; and the go/no-go is an inequality, not an instinct.

## What's new here (calibration)

- **Already yours (skip):** gang scheduling and all-or-nothing admission (module 06 lessons 01–02); Kueue's queueing model and cohorts (06.03–06.04); topology-aware placement, NVLink domains and rail alignment (06.06, module 09); the per-job-size effective-capacity formula `placeable_real(k) = Σ_n ⌊free_n / k⌋` and its 12-node worked inventory (06.07); the MIG partition geometry and its structurally unallocatable memory remainder (lesson 01); DRA's object model (lesson 01).
- **New angle 1 — fragmentation as an expectation, not a percentage.** A fleet does not have *a* fragmentation ratio; it has one per demand distribution. §4 defines `F = Σ_m p_m · F(m)` precisely, shows it reduces exactly to module 06's per-`k` stranded column, and demonstrates the same fleet reading 16% or 33% fragmented purely from a change in the job mix.
- **New angle 2 — computed against a real trace, reproducibly.** Alibaba's `cluster-trace-gpu-v2023` is public. §6 gives the node inventory, the full request-shape histogram, and a placement replay under two policies, with the script. Every number in the worked example can be regenerated on your laptop in seconds.
- **New angle 3 — the sliver problem, quantified.** Fractional GPU sharing creates remainders, and a remainder is only worth something if the arriving asks are small enough to use it. On the real trace an 810-milli allocation leaves 190 milli that **1.50% of arrivals can use**. That curve — servable fraction versus remainder size — is the honest way to decide a sharing granularity.
- **New angle 4 — the money, both directions.** Stranded `$/hr` parameterised in `r`; defrag priced as lost work plus requeue; and the break-even horizon that says whether to consolidate.
- **New angle 5 — the levers with their actual formulas.** kube-scheduler's `MostAllocated` scorer as written in the source, Volcano's `binpack` scoring function and its `binpack.resources` gate, DRA partitionable devices' `CounterSet`/`ConsumesCounters` as the structural fix for MIG-shaped fragmentation, and the drain cost of dynamic MIG.
- **New angle 6 — why no cost tool reports this.** OpenCost's idle allocation is `asset − allocated` (lesson 03, §9): it will tell you gap A's size and can never tell you whether gap A is *placeable*, because it has no model of shape. That gap is the reason your operator exists.

## Core concepts

### 1. Free, allocatable, and usable are three different numbers

Start with the definitions, because the whole lesson is the distance between them.

```
  PHYSICAL     G      every GPU you pay for                       (area ① )
  ALLOCATED    A      bound to some pod right now                 (area ② )
  FREE         F = G − A     not bound to anything                (gap A  )
  USABLE       U(m)   free capacity that could accept the NEXT
                      request of shape m, given placement rules

                      U(m) ≤ F,  and the inequality is usually strict.

  FRAGMENTED   Φ(m) = F − U(m)          free but unusable for shape m
  RATIO        φ(m) = Φ(m) / F          scale-free, in [0, 1]

  φ = 0.0  every free GPU is placeable for m — perfect packing
  φ = 1.0  every free GPU is stranded for m — the fleet is full for m
           WHILE THE DASHBOARD SAYS F GPUs ARE FREE
```

The formula module 06 derived is the whole-GPU, single-shape instance of this. For an indivisible, co-located request of `k` GPUs:

```
  placeable_naive(k) = ⌊ F / k ⌋                 ← the lie: GPUs as a fluid
  placeable_real(k)  = Σ_n ⌊ free_n / k ⌋        ← the truth: node boundaries

  Φ(k) = F − k · placeable_real(k) = Σ_n ( free_n mod k )
```

That last identity is worth pausing on: **the stranded capacity for shape `k` is the sum of the per-node remainders modulo `k`.** It falls straight out of the floor function and it is the bridge to §4's distributional version.

Module 06's inventory, carried forward unchanged so the two lessons cannot disagree:

```
  free GPUs per node: [8, 8, 2, 2, 5, 5, 5, 1, 1, 4, 4, 3]     F = 48

   k   naive  real   Σ(free_n mod k) = Φ(k)   φ(k)      what it means
  ───────────────────────────────────────────────────────────────────
   1     48    48            0                0.0 %   every GPU placeable
   2     24    21            6               12.5 %
   4     12     9           12               25.0 %
   8      6     2           32               66.7 %   ⟵ 4 of 6 "slots" gone
  ───────────────────────────────────────────────────────────────────
  SAME FLEET. SAME INSTANT. FOUR DIFFERENT ANSWERS.
```

**There is no such thing as "the fleet's fragmentation percentage."** Reporting one to leadership is worse than reporting nothing, because it invites the wrong fix: a single alarming number reads as "buy capacity", when the right fix is usually a placement-policy change that costs nothing. Report per dominant shape, or report the expectation from §4.

### 2. Why GPU jobs fragment and CPU jobs do not

Five properties, none of which a CPU workload has, and each of which removes a degree of freedom from the packer.

**Indivisibility.** A container asking for `nvidia.com/gpu: 8` gets eight whole devices or nothing. There is no equivalent of "give it 6.4 cores and let it run slower" — extended resources are integer-quantised, and Kubernetes additionally requires request to equal limit for them, so there is not even a burst path. Every packing problem with integer items and integer bins has remainders; the smaller the bin relative to the item, the larger the remainders.

**Gang atomicity.** A tensor-parallel or pipeline-parallel job needs all its ranks simultaneously; partial placement is not slow progress, it is a deadlock where half a gang holds GPUs waiting for the other half that will never be scheduled. Gang scheduling (module 06) fixes the deadlock by making admission all-or-nothing — which means the scheduler now demands a *contiguous* multi-device hole, and holes are exactly what fragmentation destroys.

**Co-location.** Intra-node NVLink/NVSwitch bandwidth is an order of magnitude above inter-node fabric, so an eight-way job wants eight GPUs on **one node**, not eight GPUs anywhere. That converts a fleet-wide bin-packing problem into a per-node one and is why the `Σ_n ⌊free_n/k⌋` denominator is per node.

**Topology within the node.** Even inside one machine, not all eight GPUs are equal: rail alignment and NVSwitch domain membership determine whether a collective runs at full bandwidth. A placement that satisfies the count but not the topology produces a job that runs — badly. This is fragmentation with **no pending pod as a symptom**; it hides inside a degraded MFU number (module 08).

**Quantised memory.** MIG profiles come from a fixed menu whose compute and memory axes do not tile evenly (lesson 01: 7 compute slices, 8 memory slices), and fractional sharing quantises a device into milli-GPU units that leave remainders. Both create *intra-device* holes that no node-level packer can see.

### 3. The four shapes of fragmentation

Same symptom — paid-for capacity nothing can use — four distinct mechanisms, four distinct measures, four distinct levers. Confusing them is why fragmentation programmes stall.

| Shape | Mechanism | Measure | Lever | Blind spot if you only look at (a) |
|---|---|---|---|---|
| **(a) Node / gang** | free GPUs scattered across nodes; no node has `k` free | `Φ(k) = Σ_n (free_n mod k)` | bin-pack scoring, gang scheduling, consolidation | — |
| **(b) MIG profile geometry** | device carved into a geometry that does not match the ask; reconfiguration drains the whole physical GPU | unallocatable slice-capacity + the structural remainder (17% of HBM at 7×`1g.5gb` on A100-40GB) | dynamic MIG with hysteresis; DRA partitionable devices | a 7×1g card counts as "0 free GPUs" and never appears in your node-level free count at all |
| **(c) In-GPU memory / sliver** | fractional sharing leaves a remainder too small for arriving asks; or HBM is free in aggregate but not as one block | remainder-servability curve (§5) | granularity standardisation, best-fit within device, allocator changes | a device with 190 free milli-GPU shows as "in use" — its stranded capacity is invisible to a whole-GPU count |
| **(d) Topology / rail** | count satisfied, interconnect not; job runs degraded rather than pending | achieved collective bandwidth vs domain-local baseline; MFU shortfall | topology-aware scheduling, domain-complete placement | **there is no pending pod** — this shape produces no scheduler symptom whatsoever |

Shape (d) is the one that gets missed by every measurement built around pending pods, and it is the one whose cost shows up on the *unit economics* side (lesson 05) rather than the capacity side: you did not lose the GPU-hours, you lost the work they should have produced.

Here is (a) and (c) drawn together, because seeing the two granularities on one picture is what makes the distinction stick:

```
  ONE FLEET, TWO GRANULARITIES OF THE SAME DISEASE.
  ░ = allocated      ▒ = free-and-placeable      ▓ = free-but-STRANDED

  ── SHAPE (a): NODE-LEVEL, whole GPUs, pending ask = 8-GPU gang ────────
                    GPU 0 1 2 3 4 5 6 7
       node-01      ░░░░░░░░░░░░▓▓▓▓          6 used, 2 free → stranded
       node-02      ░░░░░░░░▓▓▓▓▓▓            4 used, 4 free → stranded
       node-03      ░░░░░░░░░░░░░░▓▓          7 used, 1 free → stranded
       node-04      ▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒          0 used, 8 free → PLACEABLE
       node-05      ░░░░░░▓▓▓▓▓▓▓▓▓▓          3 used, 5 free → stranded
                                    ────────────────────────────────────
                                    F  = 2+4+1+8+5 = 20 free GPUs
                                    U(8) = 8   (only node-04)
                                    Φ(8) = 12  → φ(8) = 12/20 = 60 %
                                    ⚠ THE DASHBOARD SAYS "20 GPUs FREE".
                                      YOU CAN START ONE JOB.

  ── SHAPE (c): IN-DEVICE, fractional sharing, milli-GPU units ──────────
       One physical GPU = 1000 milli.  Arriving asks: 810 / 470 / 320 …

       GPU-A   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░▓▓▓▓▓▓▓
               │←──────── pod @ 810 ─────────→│←190→│
                                               ▲
                                               └ 190 milli free.
                                                 1.50 % of the real ask
                                                 stream can use it (§6).
                                                 The other 98.5 % cannot.

       GPU-B   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░▓▓▓▓▓▓▓▓▓▓▓
               │←── 470 ──→│←── 470 ──→│←──── 60 ────→│
                                          ▲
                                          └ 60 milli. 0.10 % servable.
                                            Effectively dead capacity.

       ⇒ Every fractional placement MANUFACTURES a remainder. Whether
         that remainder is capacity or waste is decided entirely by the
         SHAPE OF THE ARRIVING DEMAND — not by how much of it there is.
```

### 4. The measure that survives contact with a real fleet: fragmentation as an expectation

A single `k` gives you a single answer, and real clusters serve a mix. Define fragmentation against the *distribution* of what arrives.

Let `M` be the set of request shapes with popularity `p_m` (measured from your own admitted-and-pending queue, `Σ p_m = 1`). For a node `n` with free capacity `free_n`, define the fragmentation of that node **with respect to one shape** as the free capacity that cannot serve that shape after packing as many copies of it as fit:

```
  F_n(m)  =  free_n  −  size(m) · ⌊ free_n / size(m) ⌋
          =  free_n  mod  size(m)                    (whole-GPU shapes)

  and, if free_n < size(m), this is exactly free_n — the whole node's
  free capacity is stranded with respect to m, which is the behaviour
  you want.

  NODE fragmentation over the distribution:   F_n = Σ_m p_m · F_n(m)
  FLEET fragmentation:                        Φ   = Σ_n F_n
  FLEET fragmentation ratio:                  φ   = Φ / Σ_n free_n
  STRANDED COST PER HOUR:                     C   = Φ × r
```

Two properties make this the right definition rather than one of several.

**It reduces to module 06's table.** Put all the popularity on one shape `k` and `Φ` becomes `Σ_n (free_n mod k)`, which is exactly the stranded column computed in §1. The distributional measure is a strict generalisation, so the two lessons cannot produce contradictory numbers.

**It is linear in the popularity vector**, which means you can decompose the fleet's fragmentation by shape and see which asks are responsible — the input to a conversation about *standardising job shapes*, which is often cheaper than any scheduler change.

Work it on module 06's twelve-node inventory, with a training-heavy mix:

```
  free per node: [8, 8, 2, 2, 5, 5, 5, 1, 1, 4, 4, 3]      F = 48
  demand mix p:  k=1 : 0.50    k=2 : 0.20    k=4 : 0.15    k=8 : 0.15

  Σ_n (free_n mod k)  — the per-shape stranded totals from §1:
      k=1 →  0        k=2 →  6        k=4 → 12        k=8 → 32

  Φ = 0.50(0) + 0.20(6) + 0.15(12) + 0.15(32)
    = 0 + 1.2 + 1.8 + 4.8
    = 7.8 GPU-equivalents stranded, in expectation
  φ = 7.8 / 48 = 16.3 %
  C = 7.8 × $2.99 = $23.32 / hour = $204,300 / year
      (r = $2.99/GPU-h, FOCUS EffectiveCost basis, 2026-08 snapshot —
       substitute yours; C is linear in r)

  ── NOW CHANGE ONLY THE MIX, NOT ONE GPU ──────────────────────────────
  p:  k=1 : 0.30    k=2 : 0.10    k=4 : 0.20    k=8 : 0.40

  Φ = 0.30(0) + 0.10(6) + 0.20(12) + 0.40(32)
    = 0 + 0.6 + 2.4 + 12.8  =  15.8 GPU-equivalents
  φ = 32.9 %        C = $47.24 / hour = $413,800 / year

  ⇒ THE SAME FLEET IS 16 % OR 33 % FRAGMENTED DEPENDING ONLY ON WHAT
    PEOPLE ASK FOR. Fragmentation is a property of the (supply, demand)
    PAIR. Any number quoted without its demand distribution is unfalsifiable.
```

The operational consequence is that **your request-shape histogram is a first-class artifact**, not a by-product. It is the input to this measure, the input to procurement (buy node shapes that match the mix), and the input to the standardisation conversation (fewer distinct shapes ⇒ smaller remainders).

### 5. The sliver problem: fractional sharing and the remainder-servability curve

Whole-GPU fragmentation is intuitive. Fractional fragmentation is where most people's intuition fails, because sharing is *sold* as the cure for fragmentation and it introduces a new axis of it.

The mechanism is exact. A device is 1,000 milli-GPU. A pod asking for `q` milli leaves a remainder of `1000 − q` on that device (and further placements chip it down). That remainder is only capacity if some future ask is `≤` it. So the value of a remainder is not a quantity — it is a **quantile of the arriving demand**:

```
  servable(ρ)  =  P( ask ≤ ρ )   over the arriving single-GPU asks

  The remainder ρ is worth  ρ × r × servable(ρ)  in expectation,
  and worth NOTHING when servable(ρ) ≈ 0.
```

Measured on the real trace (§6 has the provenance; this is the full curve over its 7,064 GPU asks):

```
  remainder ρ (milli)   asks that fit    share of arrivals   verdict
  ────────────────────────────────────────────────────────────────────
        30                    0               0.00 %        dead
        60                    7               0.10 %        dead
       190                  106               1.50 %        near-dead
       350                  654               9.26 %        marginal
       530                1,600              22.65 %        useful
       680                2,000              28.31 %        useful
     1,000                6,989              98.94 %        a whole GPU
  ────────────────────────────────────────────────────────────────────
  THE CURVE IS NOT LINEAR AND IT IS NOT SMOOTH. It is a staircase whose
  steps are the popular ask sizes. On this trace, 810-milli is the single
  most common fractional ask (15.3 % of all GPU asks) and it leaves a
  190-milli remainder that essentially nothing can use.
```

Three design rules fall out, and they are the practical content of this section.

**Standardise the granularity to divide evenly.** If asks are quantised to `{250, 500, 1000}` milli, every remainder is itself a legal ask size and the staircase disappears. The trace's 20 distinct fractional sizes (`50, 110, 140, 160, 220, 230, 270, 290, 320, 330, 350, 370, 440, 460, 470, 480, 550, 590, 650, 810`) guarantee slivers no matter how good the scheduler is. **Shape standardisation is a fragmentation lever that costs zero engineering and is almost never used.**

**Best-fit within the device, not first-fit.** Placing a 320 on a device with 350 free (leaving 30, dead) is worse than placing it on one with 1,000 free (leaving 680, 28% servable) *only if* you will not need that whole device later — which is precisely the trade-off a fragmentation-aware scorer should make explicitly rather than by accident.

**Sharing does not reduce fragmentation; it relocates it.** Fractional allocation converts node-level fragmentation (whole GPUs stranded by gang shape) into device-level fragmentation (slivers stranded by ask size). Whether that is a win depends entirely on the ask distribution — which is exactly the finding the ATC '23 fragmentation work reports from production and why it exists as a research problem at all.

### 6. Measuring it against a real production trace

You do not have to take any of this on trust. Alibaba publishes the trace behind the ATC '23 fragmentation work as `cluster-trace-gpu-v2023` in `alibaba/clusterdata`, and it is small enough to analyse in seconds. Everything below was computed directly from the published CSVs.

**The supply side** (`openb_node_list_gpu_node.csv`):

```
  1,213 GPU nodes,  6,212 physical GPUs

  node size   count   GPUs      cumulative share of GPUs
  ──────────────────────────────────────────────────────
   1 GPU        24      24            0.4 %
   2 GPU       518   1,036           17.1 %
   4 GPU        54     216           20.5 %
   8 GPU       617   4,936          100.0 %
  ──────────────────────────────────────────────────────
  models present: G2, G3 (undisclosed internal codes), V100M16,
                  V100M32, T4, P100, A10 — a HETEROGENEOUS fleet

  ⇒ STRUCTURAL BOUND, BEFORE ANY POD IS SCHEDULED:
    1,276 GPUs (20.5 %) live on nodes with fewer than 8 devices and can
    NEVER host an 8-GPU co-located gang. That fraction of the fleet is
    permanently invisible to your largest job shape, and no scheduler,
    consolidation pass or defrag run changes it. It is a PROCUREMENT
    finding, not a scheduling one.
```

**The demand side** (`openb_pod_list_default.csv`, 8,152 tasks, of which 7,064 ask for GPU):

```
  shape                     count    share of GPU asks
  ────────────────────────────────────────────────────
  1 whole GPU (1000 milli)  3,911         55.4 %
  1 fractional GPU          3,078         43.6 %   ← 20 distinct sizes
  8-GPU gang                   44          0.62 %
  2-GPU gang                   16          0.23 %
  4-GPU gang                   15          0.21 %
  ────────────────────────────────────────────────────
  most popular fractional asks:
      810 milli  1,078 pods (15.3 % of all GPU asks)
      470 milli    750       (10.6 %)
      320 milli    328        (4.6 %)
      650 milli    253        (3.6 %)

  TOTAL DEMAND = 6,086.8 GPU-equivalents against 6,212 supplied = 98.0 %

  ⇒ AND THE POINT: gangs are 1.06 % of the ASKS and 7.3 % of the DEMAND
    (444 GPU-equivalents), yet they are what fragmentation blocks. A
    fragmentation programme optimised for "most pods placed" will
    cheerfully starve the 1 % that carries the flagship workloads.
```

**The replay.** Place every GPU pod in creation order onto the real inventory under two policies — best-fit ("bin-pack": among devices/nodes that fit, pick the one with the least remaining) and worst-fit ("spread": pick the one with the most remaining) — with the honest simplifications stated below. This is ~60 lines of Python and the whole point is that you can re-run it:

```python
#!/usr/bin/env python3
"""Replay a real GPU trace under two placement policies and price the
   stranded capacity.  Data: alibaba/clusterdata cluster-trace-gpu-v2023.

   SIMPLIFICATIONS — state them whenever you quote the output:
     · single-shot: every pod is placed once, ignoring creation/deletion
       times, so concurrency is the trace's total rather than its peak
     · CPU and memory requests are ignored (GPU is the binding resource)
     · GPU-type constraints ignored (the default pod file has none; the
       *gpuspec33* variant adds them to ~33 % of GPU tasks)
     · a gang needs `n` WHOLE free GPUs on ONE node (co-location)
"""
import csv, collections

nodes = [(n['sn'], int(n['gpu']))
         for n in csv.DictReader(open('openb_node_list_gpu_node.csv'))
         if int(n['gpu']) > 0]
pods  = [p for p in csv.DictReader(open('openb_pod_list_default.csv'))
         if int(p['num_gpu']) > 0]
pods.sort(key=lambda p: int(p['creation_time']))

# demand distribution M: (num_gpu, milli) -> popularity, straight from the trace
dist = collections.Counter((int(p['num_gpu']), int(p['gpu_milli'])) for p in pods)
M = [(k, v / len(pods)) for k, v in dist.items()]

def place(policy):
    """state[node] = list of per-GPU remaining milli-GPU."""
    state = {sn: [1000] * g for sn, g in nodes}
    placed, failed = 0, collections.Counter()
    for p in pods:
        n, q = int(p['num_gpu']), int(p['gpu_milli'])
        if n == 1 and q < 1000:                     # fractional ask
            c = [(rem, sn, i) for sn, gs in state.items()
                              for i, rem in enumerate(gs) if rem >= q]
            if not c:
                failed['fractional'] += 1; continue
            c.sort(key=lambda x: x[0], reverse=(policy == 'spread'))
            _, sn, i = c[0]; state[sn][i] -= q; placed += 1
        else:                                        # whole-GPU or gang
            c = [(sum(1 for r in gs if r == 1000), sn)
                 for sn, gs in state.items()]
            c = [x for x in c if x[0] >= n]
            if not c:
                failed[f'gang{n}'] += 1; continue
            c.sort(key=lambda x: x[0], reverse=(policy == 'spread'))
            sn = c[0][1]; taken = 0
            for i, r in enumerate(state[sn]):
                if r == 1000 and taken < n:
                    state[sn][i] = 0; taken += 1
            placed += 1
    return state, placed, dict(failed)

def fragmentation(state):
    """Expected stranded free capacity over the trace's own demand mix."""
    frag = free = 0.0
    for gs in state.values():
        f = sum(gs) / 1000.0
        free += f
        if f == 0:
            continue
        whole = sum(1 for r in gs if r == 1000)
        part  = sum(r for r in gs if 0 < r < 1000) / 1000.0
        for (n, q), p in M:
            if n == 1 and q < 1000:      # a sliver smaller than the ask is lost
                frag += p * sum(r for r in gs if r < q) / 1000.0
            else:                        # gang: no k whole GPUs ⇒ all of it is lost
                frag += p * (f if whole < n else part)
    return frag, free

for policy in ('binpack', 'spread'):
    st, placed, failed = place(policy)
    phi, free = fragmentation(st)
    print(f'{policy:8s} placed {placed:5d}/{len(pods)}  unplaced {failed}')
    print(f'         free {free:7.1f} GPU-eq | stranded (expected) {phi:7.1f}'
          f' | ratio {phi/free:.3f}')
```

Its output — the numbers the worked example is built on:

```
binpack  placed  6910/7064  unplaced {'gang1': 47, 'fractional': 106, 'gang8': 1}
         free    251.0 GPU-eq | stranded (expected)   246.6 | ratio 0.983
spread   placed  6423/7064  unplaced {'gang8': 40, 'gang4': 9, 'gang2': 3,
                                      'gang1': 433, 'fractional': 156}
         free   1046.6 GPU-eq | stranded (expected)   858.3 | ratio 0.820
```

Read it carefully, because two of the four findings are counter-intuitive.

**Policy moved 487 pods and 795.6 GPU-equivalents.** Bin-pack placed 6,910 of 7,064 requests; spread placed 6,423 and left 641 unplaced, including **40 of the 44 eight-GPU gangs**. Same hardware, same arrivals, same instant — the difference is a scoring function.

**Spread fails the gangs specifically.** Its unplaced list is dominated by shapes that need contiguity. That is the mechanism in §2 made visible: worst-fit deliberately keeps every node partly occupied, which is exactly the state in which no node has eight whole devices free.

**Both policies end with a high fragmentation *ratio*, and that is not a contradiction.** At 98% demand-to-supply the leftovers are slivers by definition, so φ approaches 1 under any policy. **The ratio measures the quality of what is left; the absolute Φ measures how much is left.** Bin-pack's φ = 0.983 looks worse than spread's 0.820 while stranding a quarter as much capacity — which is why you must publish `Φ` in GPU-equivalents and dollars, with `φ` as a secondary diagnostic. A programme steered by φ alone would pick the worse policy.

**The structural 20.5% is untouched by either.** No policy places an 8-GPU gang on a 2-GPU node. That bound is set at purchase time.

### 7. MIG and DRA: fragmentation inside the device

Shape (b) has a geometry problem that is worth stating precisely because it is where "sharing solves fragmentation" breaks down hardest.

From lesson 01, using the A100-40GB counter model published in the DRA partitionable-devices KEP (KEP-4815): the card exposes **98 MIG-addressable multiprocessors**, **8 memory slices**, 7 copy engines and 40,960 MiB; a `1g.5gb` partition consumes 14 MP, one memory slice, one copy engine and 4,864 MiB. So:

```
  7 × 1g.5gb :  compute   7 × 14 MP    =  98 / 98 MP     = 100.0 %
                memory    7 × 4,864    = 34,048 / 40,960 =  83.1 %
                                          ─────────────────────────
                STRUCTURALLY UNALLOCATABLE  6,912 MiB    =  16.9 %

  Compute tiles perfectly; memory cannot, because there are 8 memory
  slices and 7 compute slices. On a card costing r/hour that is 0.169 × r
  per hour with NO POSSIBLE TENANT. It is not schedulable-around and not
  tunable — it is geometry, and it must be booked to a platform overhead
  cost centre or your per-GPU shares will not sum to 1.0.
```

On top of that structural floor sits the *dynamic* problem: a card carved 7×`1g` cannot serve a `3g` request without a reconfiguration, and reconfiguration **drains the entire physical GPU** — every existing slice is evicted, the device is reset, the geometry rewritten. So:

```
  STATIC MIG   simple, no churn, predictable.
               STRANDS whenever the ask mix drifts from the geometry.
               Steady-state cost = Φ(b) × r, paid continuously.

  DYNAMIC MIG  fits supply to demand, cutting Φ(b).
               Each reconfigure costs a full-device drain:
                 C_drain = (evicted work lost + restart) × slices × r
               and if demand OSCILLATES you pay it repeatedly.

  DECISION:    reconfigure iff   Φ_saved × r × T_stable  >  C_drain
               where T_stable is how long the new geometry will hold.
               ⇒ RATE-LIMIT and HYSTERESIS-GATE reconfiguration, always.
                 Dynamic MIG trades a STOCK for a FLOW; if the flow rate
                 is high enough you have converted a fragmentation
                 problem into a thrash problem and made it worse.
```

**DRA's partitionable devices are the structural fix**, and the mechanism is worth knowing because it is genuinely different from "the device plugin pre-slices the card." Under KEP-4815 the driver publishes a **`CounterSet`** per physical device — for the A100 example: `memory: 40Gi`, `multiprocessors: 98`, `copy-engines: 7`, plus eight individually named `memorySliceN` counters — and each advertisable partition declares via **`ConsumesCounters`** exactly what it draws. The scheduler subtracts as it allocates, and a partition becomes unallocatable the moment any single counter is exhausted:

```
  WHY THIS KILLS SHAPE-(b) FRAGMENTATION AT ITS ROOT

  device-plugin model            DRA partitionable model
  ───────────────────            ───────────────────────
  geometry chosen BEFORE         geometry chosen AT SCHEDULE TIME, from
  demand is known                the claim in front of the scheduler

  scheduler counts opaque        scheduler subtracts typed counters and
  integers                       KNOWS why a partition no longer fits

  overlapping geometries         overlapping geometries are impossible
  prevented by convention        by counter exhaustion (two profiles
                                 needing memorySlice3 cannot coexist)

  reshaping = drain + rewrite    reshaping = the next allocation
```

What DRA does **not** fix, exactly as in lesson 01: nothing about attribution, and nothing about shapes (a) or (d). It makes intent legible and partitioning schedule-time; it does not give the hardware a counter it never had, and it does not put eight GPUs on a two-GPU node.

### 8. The levers, with their actual scoring functions

The single highest-value change on most GPU clusters is a scheduler config edit, so know precisely what you are editing.

**kube-scheduler `NodeResourcesFit`.** The default scoring strategy is `LeastAllocated` — spread — and the default resource set is `cpu` and `memory` with equal weight (`pkg/scheduler/apis/config/v1/defaults.go`: `SetDefaults_NodeResourcesFitArgs` installs `Type: LeastAllocated` and `defaultResourceSpec`). **`nvidia.com/gpu` is not scored at all unless you list it.** The `MostAllocated` scorer, from the source (`most_allocated.go`):

```go
// score = Σ_r [ 100 · requested_r / allocatable_r · weight_r ] / Σ_r weight_r
func mostRequestedScore(requested, capacity int64) int64 {
    if capacity == 0 { return 0 }
    if requested > capacity { requested = capacity }
    return (requested * fwk.MaxNodeScore) / capacity   // MaxNodeScore = 100
}
```

so the config that actually changes GPU behaviour is:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated          # bin-pack, NOT the LeastAllocated default
        resources:
        - name: nvidia.com/gpu       # ← without this line the GPU is not
          weight: 100                #   considered in scoring AT ALL
        - name: cpu
          weight: 1                  # weights are validated to 1..100
        - name: memory
          weight: 1
```

**Volcano's `binpack` plugin** uses the same idea with an explicit resource gate. From `pkg/scheduler/plugins/binpack/binpack.go`, the per-resource score is `(used + requested) × weight / capacity`, summed over resources **that appear in the configured resource map**, normalised by the weight sum and scaled by `MaxNodeScore × binpack.weight`:

```go
score := usedFinally * float64(weight) / capacity   // usedFinally = used + requested
...
if weightSum > 0 { score /= float64(weightSum) }
score *= float64(fwk.MaxNodeScore * int64(weight.BinPackingWeight))
```

and the configuration that turns it on for GPUs — note that a resource **not** listed in `binpack.resources` is skipped by `continue` in the scoring loop and contributes nothing:

```yaml
actions: "enqueue, allocate, backfill"
tiers:
- plugins:
  - name: binpack
    arguments:
      binpack.weight: 10                        # plugin weight overall
      binpack.cpu: 1
      binpack.memory: 1
      binpack.resources: nvidia.com/gpu         # ← the gate
      binpack.resources.nvidia.com/gpu: 10      # per-resource weight
```

**The rest of the ladder**, each mapped to the shape it fixes:

| Lever | Fixes shape | Mechanism | Cost |
|---|---|---|---|
| bin-pack scoring | (a) | future free capacity coalesces into whole-node holes | none (config) |
| gang / all-or-nothing admission | (a) | prevents partial placements that half-fill nodes and then deadlock | queue wait |
| consolidation (KAI, descheduler) | (a) | actively relocates running work to open contiguous holes | eviction — priced in §9 |
| topology-aware placement | (d) | places gangs within an NVLink/rail domain | reduced placement freedom ⇒ slightly higher (a) |
| granularity standardisation | (c) | remainders become legal ask sizes | a policy conversation, no code |
| dynamic MIG | (b) | geometry follows demand | full-device drain per reconfigure |
| DRA partitionable devices | (b), (c) | partition chosen at schedule time from typed counters | Kubernetes version floor; driver support |
| buying the right node shape | structural | 8-GPU nodes host 8-GPU gangs | capital |

**Anti-thrash floors apply here exactly as in lesson 03.** NVIDIA's KAI Scheduler pairs its consolidation feature with a *min-guaranteed-runtime* — a window during which a workload must not be preempted or reclaimed even if preemptible. Consolidation without such a floor is a machine for evicting jobs that just started.

### 9. Defrag economics: a stock, a flow, and the break-even

Fragmentation is a **stock** measured in GPU-equivalents. Consolidation is a **flow** that reduces the stock and costs GPU-hours. The decision is an inequality.

```
  GAIN   = Φ_recovered × r × T_useful
           Φ_recovered : GPU-equivalents that become placeable
           T_useful    : hours the recovered capacity is actually
                         OCCUPIED by work that could not previously run.
                         ⚠ NOT the hours it merely exists. Capacity you
                           free and nobody uses is worth exactly zero.

  COST   = Σ over migrated jobs of
             ( time_since_checkpoint + restart + queue_wait ) × GPUs_job × r

  CONSOLIDATE IFF  GAIN > COST, i.e.

           Φ_recovered × T_useful  >  Σ_j (t_ckpt_j + t_restart_j + t_q_j) × G_j

  — note r cancels. THE DECISION IS RATE-INDEPENDENT. It is an argument
    about GPU-hours, which is why it survives a pricing dispute.
```

Worked, on module 06's twelve-node fleet:

```
  PLAN: consolidate the occupants of two nodes that each have 5 free
        (i.e. 3 GPUs busy each) onto one node, freeing one whole node.

  COST  2 jobs migrated, 3 GPUs each.
        checkpoint interval 30 min ⇒ mean loss 15 min
        restart (image + weights + warm-up)     10 min
        queue wait to re-admit                   5 min
        ────────────────────────────────────────────────
        per job: 0.50 h × 3 GPUs               = 1.5 GPU-h
        two jobs                                = 3.0 GPU-h
                                                = $8.97 at r = $2.99

  GAIN  one whole 8-GPU node becomes placeable for the flagship shape.
        Φ(8) falls from 32 to 24 GPUs (the two 5-free nodes become one
        8-free node and one 2-free node: remainders 5+5 = 10 → 0+2 = 2).
        Recovered = 8 GPU-equivalents of PLACEABLE capacity.

  BREAK-EVEN HORIZON
        8 GPU-equiv × T_useful  >  3.0 GPU-h
        T_useful > 0.375 h = 22.5 minutes of the recovered node
        actually running a job that could not previously be placed.

  ⇒ If the pending 8-GPU job starts immediately and runs for hours, the
    defrag pays for itself in under half an hour. If nothing is waiting
    for that shape, T_useful = 0 and the defrag is a PURE LOSS.

    THE TEST IS NOT "IS THE FLEET FRAGMENTED". IT IS "IS SOMETHING
    PENDING THAT THE DEFRAG WOULD UNBLOCK." Consolidate against a
    queue, never against a ratio.
```

Two riders. **Non-checkpointing workloads are not migratable at any price** — the cost term contains all work since process start, so the inequality effectively never holds; this is why checkpoint hygiene is a *capacity* lever and not only a reliability one. And **consolidation has a second-order cost**: packing tightly reduces the fleet's ability to absorb the next arrival without another migration, so a fleet held permanently at maximum packing pays a continuous migration flow. Target a packing level, not maximum packing.

### 10. Why no cost tool reports any of this

Lesson 03 §9 established what OpenCost's idle allocation is: `assetTotal − allocTotal` per node or cluster, clamped at zero. In this lesson's vocabulary that is `F × r` — **the size of gap A, with no model of shape whatsoever.** It cannot tell you `U(m)`, `Φ`, or `φ`, because computing them requires two inputs a cost tool does not have:

1. **The free-capacity layout**, per node and (for fractional sharing) per device — not just the total.
2. **The demand distribution**, from the pending queue and recent admissions — which lives in the scheduler or the queueing system (Kueue/Volcano/KAI), not in the billing pipeline.

That is a genuine, statable gap and it is the reason this lesson's output belongs in your operator rather than in a dashboard you install. The join is: *free-capacity layout* (from the node inventory and allocation records you already collect for lesson 02's allocated ledger) **×** *request-shape histogram* (from the queue) **×** *rate card*. Nothing else in the ecosystem performs it, and the artifact that does is a strong portfolio piece precisely because the inputs are boring and the output is money.

### 11. Instrumenting it on a live cluster

Both inputs are already in your cluster; neither is in your cost tool. Here is how to extract them.

**The free-capacity layout, per node.** From the API server directly, which is the version to trust because it needs no exporter:

```bash
# Free whole GPUs per node = allocatable − requested-by-non-terminal-pods.
kubectl get nodes -o json | jq -r '
  .items[]
  | select(.status.allocatable["nvidia.com/gpu"] != null)
  | {node: .metadata.name,
     alloc: (.status.allocatable["nvidia.com/gpu"] | tonumber)}' \
| jq -s '.' > /tmp/nodes.json

kubectl get pods -A --field-selector=status.phase!=Succeeded,status.phase!=Failed \
  -o json | jq -r '
  .items[]
  | select(.spec.nodeName != null)
  | . as $p
  | [ $p.spec.nodeName,
      ([ $p.spec.containers[].resources.requests["nvidia.com/gpu"] // "0"
       | tonumber ] | add) ]
  | @tsv' \
| awk -F'\t' '{used[$1]+=$2} END {for (n in used) print n"\t"used[n]}' \
  > /tmp/used.tsv
# join → free_n per node → feed straight into Σ_n (free_n mod k)
```

The same quantity in PromQL, if you already run `kube-state-metrics` and want it as a series (note the resource label is `nvidia_com_gpu` — dots and slashes are sanitised):

```promql
# free whole GPUs per node
  kube_node_status_allocatable{resource="nvidia_com_gpu"}
- on (node) group_left() (
    sum by (node) (
      kube_pod_container_resource_requests{resource="nvidia_com_gpu"}
      * on (namespace, pod) group_left()
        (kube_pod_status_phase{phase="Running"} == 1)
    )
  )

# placeable k-GPU gangs, fleet-wide — the honest capacity number.
# floor() over per-node free capacity, then sum. Compare against the
# naive sum(free)/k to get the fragmentation loss for shape k.
sum( floor( node:gpu_free:count / 8 ) )
```

**The demand distribution**, from what is actually waiting rather than from what you imagine:

```promql
# pending GPU pods by requested gang size — the p_m histogram, live
count by (gpu_request) (
  label_replace(
    kube_pod_container_resource_requests{resource="nvidia_com_gpu"}
      * on (namespace, pod) group_left()
        (kube_pod_status_phase{phase="Pending"} == 1),
    "gpu_request", "$1", "__name__", ".*"
  )
)
```

On a Kueue cluster prefer the queue's own view — `kueue_pending_workloads` and the workload objects carry the *admitted-or-waiting* shape including gang size, which is more truthful than pod-level requests once all-or-nothing admission is in play.

**Then emit the three numbers as series**, so fragmentation joins the same rate card as everything else in the module:

```
  gpu:free_gpus:count                      Σ_n free_n
  gpu:placeable_jobs:count{shape="8"}      Σ_n ⌊free_n / 8⌋
  gpu:stranded_equiv{shape="8"}            Σ_n (free_n mod 8)
  gpu:stranded_cost_usd_per_hour           Σ_shape p_shape · stranded_equiv · r
```

**And watch the shape evolve, not just its level.** Fragmentation is produced by a *sequence* of placements, which is why a point-in-time ratio can look fine an hour before a flagship job cannot start:

```
  THE SAME FOUR ARRIVALS, TWO POLICIES. ░ allocated  ▒ free-placeable
  ▓ free-stranded (for the pending 8-GPU gang).  Nodes are 8 wide.

  t0  fresh 4-node fleet, 32 free            SPREAD          BIN-PACK
      n1 ▒▒▒▒▒▒▒▒  n2 ▒▒▒▒▒▒▒▒               (LeastAllocated) (MostAllocated)
      n3 ▒▒▒▒▒▒▒▒  n4 ▒▒▒▒▒▒▒▒               placeable 8-gangs: 4    4

  t1  ← 2-GPU job arrives
      SPREAD  : goes to the emptiest node    n1 ░░▒▒▒▒▒▒  n2 ▒▒▒▒▒▒▒▒
      BIN-PACK: goes to the fullest that fits n1 ░░▒▒▒▒▒▒  n2 ▒▒▒▒▒▒▒▒
                                             placeable: 3        3

  t2  ← 2-GPU job arrives
      SPREAD  : n2 (now the emptiest)        n1 ░░▓▓▓▓▓▓  n2 ░░▓▓▓▓▓▓
      BIN-PACK: n1 (already has 2 used)      n1 ░░░░▒▒▒▒  n2 ▒▒▒▒▒▒▒▒
                                             placeable: 2        3

  t3  ← 2-GPU job arrives
      SPREAD  : n3                           n1 ░░▓▓▓▓▓▓ n2 ░░▓▓▓▓▓▓
                                             n3 ░░▓▓▓▓▓▓ n4 ▒▒▒▒▒▒▒▒
      BIN-PACK: n1                           n1 ░░░░░░▒▒ n2 ▒▒▒▒▒▒▒▒
                                             n3 ▒▒▒▒▒▒▒▒ n4 ▒▒▒▒▒▒▒▒
                                             placeable: 1        3

  t4  ← 2-GPU job arrives
      SPREAD  : n4  → EVERY node is dirty    n1..n4 all ░░▓▓▓▓▓▓
                     placeable 8-gangs: 0    Φ(8) = 24 of 24 free
      BIN-PACK: n1  → n1 full, rest pristine n1 ░░░░░░░░ n2..n4 ▒▒▒▒▒▒▒▒
                     placeable 8-gangs: 3    Φ(8) = 0

  ⇒ IDENTICAL WORKLOAD. IDENTICAL FREE COUNT (24 GPUs in both).
    Spread destroyed every large hole in four small placements, and it
    did so INVISIBLY — no alert fires when a node goes from clean to
    dirty. The damage is done by the ORDER of decisions, so the metric
    to watch is placeable-jobs-by-shape OVER TIME, not free-GPU count.
```

## Perspectives

**Scheduler policy.** Bin-pack versus spread is not a tuning knob on a GPU fleet; it is the single largest determinant of fragmentation available, and Kubernetes' default is the wrong one for this workload class. `LeastAllocated` exists for good reasons on fungible CPU services — blast radius, noisy-neighbour isolation, HA — and those reasons do not transfer to indivisible gang-scheduled jobs on scarce hardware. The correct posture is *per queue*: spread for stateless prod serving where blast radius dominates, bin-pack for training where placeability of the next gang dominates. And regardless of policy, list `nvidia.com/gpu` in the scoring resources, because otherwise the scheduler is optimising the placement of the resource you do not care about.

**Hardware and topology.** "Enough GPUs" and "the right GPUs" diverge the moment NVLink domains and rail alignment matter. A count-only scheduler will happily satisfy an eight-GPU request with eight devices spanning two NVSwitch domains; the job runs, the all-reduce crawls, and nothing pends. That is shape (d): fragmentation with no scheduler-visible symptom, showing up as a lower MFU and therefore as a *higher cost per unit of work* rather than as idle capacity. It is the one shape you must instrument on the throughput side, not the capacity side.

**Finance and capital.** Stranded fragmented capacity is a different financial object from lesson 02's utilisation waste and lesson 03's idle. Utilisation waste says "we bought the right amount and ran it inefficiently" and its owner is the tenant. Fragmentation says "we bought the right amount and arranged it so it cannot be used" and its owner is the platform. Reporting them in one bucket misdirects the fix — toward workload teams who cannot change node shapes or scoring functions — and misdirects the money, toward hardware that will fragment exactly like the hardware you already have.

**Capacity planning.** The request-shape histogram is a procurement input, not just an observability output. A fleet whose pending queue is persistently eight-GPU gangs should be buying eight-GPU nodes and biasing MIG geometry toward larger profiles *before* the fragmentation appears. The real trace makes the point at scale: 20.5% of its GPUs sit on nodes too small to ever host its flagship shape, and that number was fixed by purchase orders, not by a scheduler. Designing the demand — standardising on fewer request shapes — is equally a lever and is usually cheaper than either.

## Real-world use cases

- **Alibaba `cluster-trace-gpu-v2023` (fetched and analysed directly).** **What it shows:** a real, downloadable production trace — 1,213 GPU nodes across seven GPU models, 6,212 physical GPUs, 8,152 tasks of which 7,064 request GPU, with fractional requests expressed in `gpu_milli` and QoS classes (LS / Burstable / BE). The measured facts used throughout this lesson: total demand is 98.0% of supply; 55.4% of GPU asks are whole single GPUs and 43.6% are fractional across 20 distinct sizes; gangs are 1.06% of asks but 7.3% of demand; 20.5% of GPUs live on nodes with fewer than eight devices. **Why it matters:** every claim in §5 and §6 is reproducible in seconds on your own machine, which is a materially stronger position than citing a percentage from a paper.

- **"Beware of Fragmentation: Scheduling GPU-Sharing Workloads with Fragmentation Gradient Descent" (USENIX ATC '23).** **What it shows:** the formal treatment — a fragmentation *measure* defined over a task distribution plus a greedy placement heuristic (FGD) that descends it, evaluated by trace replay on an emulated cluster of over 6,200 GPUs. The reported headline is that FGD reduces unallocated GPUs by up to **49%** versus packing-based baselines, recovering the use of about **290 GPUs**. **Correction recorded:** an earlier version of this lesson attributed a "21–42% of unallocated GPU resources become fragmentation" figure to this paper; that figure could not be confirmed against any reachable primary source (usenix.org and the authors' PDF host are both blocked from this environment), so it has been removed in favour of the 49% / 290-GPU result, which multiple reachable secondary sources report consistently. Treat both as evidence of magnitude on that specific fleet, not as constants.

- **Alibaba `cluster-trace-gpu-v2026` and the OSDI '26 characterization.** **What it shows:** the successor trace, published alongside a six-month characterization covering up to 155,410 GPUs across 37,707 servers, treating stranded capacity as a first-class named phenomenon worth a dedicated paper. **Why it matters:** it is the scale answer to "isn't fragmentation a toy problem" — one of the largest operators in the world spends research effort measuring it because the dollar value justifies the effort. Use the 2023 trace to build your calculator and the 2026 trace to stress it.

- **Meta — "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale" (OSDI '24).** **What it shows:** production global scheduling built on temporal decoupling, scope decoupling and exhaustive search, explicitly to fix regional GPU demand/supply imbalance — capacity that exists but is in the wrong place is the fleet-scale version of capacity that exists but is in the wrong shape. The paper reports the most overloaded region's high-priority demand-to-supply ratio falling from **2.63 to 0.98** under MAST. **Provenance note:** usenix.org was unreachable from this environment, so that pair of figures is carried from the prior version of this lesson and re-verified only at the level of the paper's stated goal; read the PDF before quoting it in an interview.

- **NVIDIA — "Practical Tips for Preventing GPU Fragmentation for Volcano Scheduler."** **What it shows:** a single scheduling-policy change — spread to bin-pack via Volcano's `binpack` plugin — driving node fragmentation from measurable double digits to under 1% with GPU occupancy climbing toward ~90% on a DGX Cloud-provisioned cluster, with no hardware purchased. **Provenance note:** developer.nvidia.com is blocked from this environment; the canonical URL is confirmed and the plugin's scoring function in §8 is quoted from the Volcano source, which is the checkable part.

- **Volcano — `pkg/scheduler/plugins/binpack/binpack.go` (fetched directly).** **What it shows:** the exact scoring function (`(used + requested) × weight / capacity`, normalised by the weight sum, scaled by `MaxNodeScore × binpack.weight`), the `binpack.weight` / `binpack.cpu` / `binpack.memory` / `binpack.resources` argument names, and — the load-bearing detail — that a resource absent from `binpack.resources` is skipped entirely by the scoring loop. **Why it matters:** "we enabled binpack" and "we enabled binpack for GPUs" are different statements, and only the second one changes anything.

- **Kubernetes — `pkg/scheduler/framework/plugins/noderesources/most_allocated.go` and `apis/config/v1/defaults.go` (both fetched directly).** **What they show:** `MostAllocated`'s formula (`100 × requested/allocatable`, weighted, averaged) and `SetDefaults_NodeResourcesFitArgs` installing `LeastAllocated` over `cpu` + `memory` with weights defaulting to 1 and validated to the range 1–100. **Why it matters:** it is the primary source for the two facts that most surprise people — the default is spread, and GPUs are not scored unless you say so.

- **DRA partitionable devices — KEP-4815 (`keps/sig-scheduling/4815-dra-partitionable-devices`).** **What it shows:** the `CounterSet` / `ConsumesCounters` model, with the A100-40GB worked as `memory: 40Gi`, `multiprocessors: 98`, `copy-engines: 7` plus eight named memory-slice counters, and a `1g.5gb` partition consuming 4,864 MiB / 14 MP / 1 copy engine / one memory slice. **Why it matters:** it is both the primary source for the MIG geometry arithmetic in §7 and the mechanism by which schedule-time partitioning eliminates shape-(b) fragmentation — overlapping geometries become impossible by counter exhaustion rather than by convention.

## Worked example

Two parts: a small fleet you can verify by hand, then the real trace.

### Part A — a 12-node fleet, priced

```
  FLEET     12 nodes × 8 H100, r = $2.99/GPU-h
            (FOCUS EffectiveCost basis, 2026-08 snapshot; every dollar
             figure below is linear in r — substitute yours)
  SNAPSHOT  free GPUs per node: [8, 8, 2, 2, 5, 5, 5, 1, 1, 4, 4, 3]
            F = 48 free of 96 physical → the dashboard says "50 % free"
  DEMAND    pending queue, measured: 50 % k=1, 20 % k=2, 15 % k=4, 15 % k=8
```

**Step 1 — placeability per shape.**

```
   k   naive ⌊F/k⌋   real Σ⌊free_n/k⌋   Φ(k)=Σ(free_n mod k)   φ(k)
  ─────────────────────────────────────────────────────────────────
   1       48              48                    0             0.0 %
   2       24              21                    6            12.5 %
   4       12               9                   12            25.0 %
   8        6               2                   32            66.7 %

  For the flagship 8-GPU shape: 2 jobs fit, not 6. 32 of 48 free GPUs
  cannot join one.
```

**Step 2 — the expectation over the actual mix.**

```
  Φ = 0.50(0) + 0.20(6) + 0.15(12) + 0.15(32) = 7.8 GPU-equivalents
  φ = 7.8 / 48 = 16.3 %

  STRANDED COST
     per hour   7.8 × $2.99  = $   23.32
     per day                 = $  559.68
     per year                = $204,283

  DECOMPOSED BY SHAPE — which asks are responsible:
     k=8 contributes 4.8 of 7.8   (61.5 %) on 15 % of the demand
     k=4 contributes 1.8          (23.1 %)
     k=2 contributes 1.2          (15.4 %)
     k=1 contributes 0.0
  ⇒ Two-thirds of the stranding is caused by one shape that is 15 % of
    the queue. That is the sentence that starts a shape-standardisation
    conversation, and it is only visible because the measure is linear.
```

**Step 3 — the packing-policy recovery.** Re-lay the 48 allocated GPUs so free capacity coalesces. Bin-pack the same occupancy into as few nodes as possible: the 48 busy GPUs fill exactly 6 nodes, leaving 6 nodes entirely free.

```
  AFTER (bin-pack):  free per node = [8, 8, 8, 8, 8, 8, 0, 0, 0, 0, 0, 0]
                     F = 48  (UNCHANGED — not one GPU was added)

   k    real   Φ(k)   φ(k)
  ─────────────────────────
   1     48      0     0 %
   2     24      0     0 %
   4     12      0     0 %
   8      6      0     0 %          ⟵ 6 flagship jobs fit, not 2

  Φ (over the mix) = 0 GPU-equivalents.   φ = 0 %.
  RECOVERED: 7.8 GPU-equivalents = $23.32/hour = $204,283/year,
  with ZERO hardware purchased and ZERO workload change.

  ⇒ THE HEADLINE OF THIS LESSON, IN ONE LINE:
    the free GPU count did not change (48 both times); only its SHAPE
    did; and the shape is a scheduler configuration.
```

**Step 4 — is the defrag worth doing?** Use §9's rate-independent test.

```
  To reach that layout you must migrate the occupants of the sparse
  nodes. Assume 4 jobs move, 3 GPUs each, 30-min checkpoint interval:
      cost = 4 jobs × (0.25 h ckpt loss + 0.17 h restart + 0.08 h queue)
                    × 3 GPUs
           = 4 × 0.50 h × 3 = 6.0 GPU-h  = $17.94

  gain  = Φ_recovered × T_useful = 7.8 × T_useful  GPU-h
  break-even: 7.8 × T_useful > 6.0  ⇒  T_useful > 0.77 h ≈ 46 minutes

  ⇒ Worth it IF something is pending that the recovered shape unblocks
    and will keep it busy for ~an hour. If the 8-GPU queue is empty,
    DO NOT DEFRAG — you would spend 6 GPU-hours to create capacity
    nobody wants.
```

### Part B — the real trace, and the sensitivity table

The replay in §6 gives the following, straight from the script's output:

```
  FLEET        1,213 GPU nodes, 6,212 GPUs (Alibaba openb trace, 2023)
  DEMAND       7,064 GPU-requesting tasks, 6,086.8 GPU-equivalents
               = 98.0 % of supply

  POLICY      placed    unplaced   free GPU-eq   Φ (expected)   φ
  ────────────────────────────────────────────────────────────────
  bin-pack    6,910       154         251.0         246.6     0.983
  spread      6,423       641       1,046.6         858.3     0.820
  ────────────────────────────────────────────────────────────────
  DELTA         +487      −487        −795.6        −611.7

  UNPLACED BY SHAPE
    bin-pack  1 of 44 eight-GPU gangs, 47 whole-GPU, 106 fractional
    spread   40 of 44 eight-GPU gangs, 9 of 15 four-GPU, 3 of 16 two-GPU,
             433 whole-GPU, 156 fractional
```

Now price it, and price it honestly — this trace's hardware is V100/T4/P100-class, so the module's H100 rate is the wrong anchor. Give the range and let the reader place their own fleet:

```
  STRANDED COST OF THE SPREAD POLICY, relative to bin-pack:
  611.7 GPU-equivalents of extra expected fragmentation

     r ($/GPU-h)     $ / hour      $ / year
     ───────────────────────────────────────────
        0.50         $  305.85    $  2,679,000
        1.00         $  611.70    $  5,358,500
        2.99         $1,828.98    $ 16,022,000

  ⚠ READ THE SIMPLIFICATIONS BEFORE QUOTING ANY OF THIS. The replay is
    single-shot (all 7,064 tasks placed at once, ignoring the trace's
    creation and deletion times), ignores CPU/memory requests and GPU-type
    constraints, and therefore represents a fully loaded cluster rather
    than a steady state. THE MAGNITUDE IS AN UPPER BOUND; THE DIRECTION
    AND THE MECHANISM ARE THE FINDING. Re-run it with lifetimes and
    resource constraints before putting a number in front of finance.
```

**And the sliver finding, which needs no simulation at all** — it is a property of the demand distribution alone, so it survives every simplification above:

```
  15.3 % of all GPU asks in this trace are for 810 milli-GPU.
  Each one leaves a 190-milli remainder on its device.
  1.50 % of arriving asks can use a 190-milli remainder.

  ⇒ 1,078 devices each carrying ~0.19 GPU of near-dead capacity
    = ~205 GPU-equivalents stranded BY THE ASK GRANULARITY ALONE,
      independent of scheduler, policy, topology or node shape.
      At r = $1.00 that is $205/hour, $1.8M/year, and the fix is a
      QUOTA POLICY: quantise fractional asks to sizes that tile 1000.
```

That last item is the one to lead with in a review, because it costs nothing to fix and no scheduler change can substitute for it.

## Practice

Feeds the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md).

1. **Build the request-shape histogram for your own fleet.** From the pending queue plus the last 30 days of admissions, tabulate `(gang size, fractional size, topology constraint)` with counts, and normalise to a popularity vector `p_m`.
   **Acceptance:** the histogram covers ≥95% of GPU-requesting workloads and is stored as data, not a screenshot — it is an input to three later calculations.

2. **Compute `Φ` and `φ` over that distribution.** Take a free-capacity layout (per node, and per device where fractional sharing is in use), apply `F_n(m) = free_n mod size(m)`, weight by `p_m`, and report GPU-equivalents, the ratio, and the dollar figure with the rate's basis and date.
   **Acceptance:** the per-shape decomposition is shown, so the reader can see which asks cause the stranding; `Φ` is reported in absolute GPU-equivalents *and* as a ratio, with a sentence explaining why the ratio alone can rank policies backwards.

3. **Replay the public trace and reproduce §6's numbers.** Download `cluster-trace-gpu-v2023`, run the script, and confirm 6,910 versus 6,423 placements. Then extend it in one of two directions: honour pod lifetimes (creation/deletion times) so the result is a steady state, or honour CPU/memory requests.
   **Acceptance:** your extended replay's numbers are reported alongside the single-shot ones with the difference explained.

4. **Plot your own remainder-servability curve.** For fractional asks, compute `P(ask ≤ ρ)` over the arriving distribution and identify the granularities that produce dead remainders.
   **Acceptance:** a proposed quantisation (e.g. quarters or halves of a device) with the expected reduction in sliver stranding computed from your own curve.

5. **Price one defrag decision.** Pick a real consolidation opportunity: compute `Φ_recovered`, the migration cost in GPU-hours (checkpoint interval, restart time, queue wait, GPUs per job), and the break-even `T_useful` — then check whether anything is actually pending that the recovered shape would unblock.
   **Acceptance:** the decision is stated as an inequality in GPU-hours (rate-independent) and includes the "nothing is pending, therefore do not defrag" branch.

6. **Classify one instance of each of the four shapes** in your fleet or a synthetic one, and name the specific lever for each — bin-pack scoring, gang admission, dynamic MIG or DRA, granularity standardisation, topology-aware placement.
   **Acceptance:** shape (d) is evidenced by a throughput or collective-bandwidth measurement rather than by a pending pod, because it produces no pending pod.

## Common pitfalls

1. **Quoting a single fleet-wide fragmentation percentage.** *Mechanism:* fragmentation is defined against a demand distribution; the same fleet is 16% or 33% fragmented purely from a change in job mix, and 0% or 67% depending on which single shape you pick. *Symptom:* a number that invites "buy more capacity" when the fix is a scoring change that costs nothing. *Fix:* report `Φ` per dominant shape or as the expectation over a measured `p_m`, always with the distribution attached.

2. **Steering by the ratio `φ` instead of the absolute `Φ`.** *Mechanism:* at high occupancy every leftover is a sliver, so `φ` approaches 1 under *any* policy — in §6's replay bin-pack scores a worse ratio (0.983 vs 0.820) while stranding a quarter as much capacity. *Fix:* publish `Φ` in GPU-equivalents and dollars as the primary figure; `φ` is a secondary diagnostic about the *quality* of what remains.

3. **Assuming Kubernetes' defaults are neutral.** *Mechanism:* `NodeResourcesFit` defaults to `LeastAllocated` (spread) over `cpu` and `memory` only — `nvidia.com/gpu` is not scored at all unless explicitly listed with a weight. *Symptom:* free GPUs smeared one and two per node across a fleet that reads as healthy. *Fix:* `MostAllocated` with the GPU resource listed, per queue; and in Volcano, remember that a resource missing from `binpack.resources` is skipped by the scoring loop entirely.

4. **Believing that more free GPUs means more usable capacity.** *Mechanism:* usability is `Σ_n ⌊free_n/k⌋`, not `⌊F/k⌋`; layout, not count, decides. Part A's fleet goes from 2 placeable flagship jobs to 6 with the identical 48 free GPUs. *Fix:* never report a free-GPU count without the placeability count for the dominant shape beside it.

5. **Treating sharing as a fragmentation cure.** *Mechanism:* fractional allocation moves fragmentation from between nodes to inside devices, manufacturing a remainder on every placement whose value depends entirely on the arriving ask sizes — a 190-milli remainder serves 1.50% of the real trace's arrivals. *Fix:* measure the remainder-servability curve before choosing a granularity, and standardise ask sizes so remainders tile.

6. **Treating dynamic MIG reconfiguration as free.** *Mechanism:* every reconfigure drains the entire physical GPU — all slices evicted, device reset, geometry rewritten — so chasing an oscillating demand mix converts a steady stock into a continuous flow and can cost more than the stranding it targets. *Fix:* rate-limit and hysteresis-gate reconfiguration; require `Φ_saved × T_stable > C_drain` before acting.

7. **Consolidating against a ratio instead of against a queue.** *Mechanism:* the gain term is `Φ_recovered × T_useful`, and `T_useful` is the time the recovered capacity is *occupied by work that could not previously run* — if nothing is pending in that shape, `T_useful = 0` and the defrag is a pure loss of the migration cost. *Fix:* gate every consolidation on a pending workload of the shape it would unblock.

8. **Ignoring topology fragmentation because the count fits.** *Mechanism:* a gang can be placed on the right *number* of GPUs spanning two NVLink domains; the job runs, the collective crosses a slow path, and nothing pends. *Symptom:* none at the capacity layer — it appears as reduced MFU and a higher cost per unit of work (lesson 05). *Fix:* measure achieved collective bandwidth against the domain-local baseline; a capacity dashboard cannot see this shape.

9. **Charging fragmentation to tenants.** *Mechanism:* stranded capacity is caused by node shapes, scoring policy and geometry — none of which a workload owner controls. Loading it onto tenant bills as an overhead surcharge, especially silently, destroys the credibility of every other number in the report. *Fix:* keep gap A on the platform's own line, published on the same chart as the tenants' gap B (lesson 02's rule).

10. **Assuming a bigger cluster fixes it.** *Mechanism:* buying more GPUs reduces the *probability* of hitting a fragmented wall but does not change the structural ratio — a fleet at twice the size with the same node shapes, mix and scoring policy strands the same *fraction*, which is twice the absolute capacity. On the real trace, 20.5% of GPUs can never host the flagship shape regardless of how many more identical nodes you add. *Fix:* fix the policy first, then the shapes you buy, and only then the count.

## Self-check

- **Why is 90% allocated not 90% usable, and what is the exact formula for the difference?**
  **Answer:** "Allocated" counts devices handed out; "usable" counts free capacity that could accept the *next* request given its placement constraints. Because a multi-GPU job is indivisible and wants co-location, free GPUs scattered across many partly-full nodes cannot form a large gang. The honest denominator is `placeable_real(k) = Σ_n ⌊free_n / k⌋`, not `⌊F/k⌋`, and the stranded capacity is the sum of the per-node remainders, `Φ(k) = Σ_n (free_n mod k)`. On the module's twelve-node inventory `[8,8,2,2,5,5,5,1,1,4,4,3]` with 48 free GPUs, the naive count says six 8-GPU jobs fit while only two actually do, stranding 32 GPUs — a 66.7% fragmentation ratio at that shape and 0% at `k=1`, from the same instant.

- **State the distributional fragmentation measure and show it generalises the per-`k` formula.**
  **Answer:** With demand shapes `m` of popularity `p_m` (`Σ p_m = 1`), define per-node fragmentation with respect to one shape as the free capacity that cannot serve it after packing as many copies as fit: `F_n(m) = free_n − size(m)·⌊free_n/size(m)⌋ = free_n mod size(m)`, which correctly equals the node's entire free capacity when `free_n < size(m)`. Then `F_n = Σ_m p_m F_n(m)`, `Φ = Σ_n F_n`, `φ = Φ / Σ_n free_n`, and the stranded cost is `Φ × r`. Concentrating all popularity on one shape `k` reduces `Φ` to `Σ_n (free_n mod k)` — exactly the per-`k` stranded column — so the two are consistent by construction. The generalisation buys two things: a single defensible fleet number, and linearity in `p_m`, which lets you attribute stranding to specific ask shapes (on the worked fleet, `k=8` causes 61.5% of the stranding while being 15% of the queue).

- **Name the four shapes of fragmentation, their measures and their levers.**
  **Answer:** **(a) Node/gang** — free GPUs scattered so no node has `k` free; measured by `Σ_n (free_n mod k)`; fixed by bin-pack scoring, gang admission and consolidation. **(b) MIG profile geometry** — a device carved into a geometry that cannot host the arriving profile, plus a structural remainder because 7 compute slices and 8 memory slices do not co-tile (7×`1g.5gb` on A100-40GB uses 98/98 MP but only 34,048 of 40,960 MiB, leaving 16.9% with no possible tenant); fixed by dynamic MIG with hysteresis, or structurally by DRA partitionable devices choosing the partition at schedule time from `CounterSet`/`ConsumesCounters`. **(c) In-GPU memory/sliver** — fractional sharing leaves remainders too small for arriving asks, or HBM is free but not contiguous; measured by the remainder-servability curve `P(ask ≤ ρ)`; fixed by standardising ask granularity so remainders tile, and by best-fit within the device. **(d) Topology/rail** — the count is satisfied but the interconnect is not, so the job runs degraded rather than pending; measured by achieved collective bandwidth or MFU shortfall, never by a pending-pod count; fixed by topology-aware placement.

- **You run the replay and bin-pack shows a *worse* fragmentation ratio than spread. Did you pick the wrong policy?**
  **Answer:** No — this is the ratio trap. In §6's replay bin-pack ends with `φ = 0.983` against spread's `0.820`, yet bin-pack placed 6,910 of 7,064 requests versus 6,423 and left 251 GPU-equivalents free versus 1,046.6. The ratio measures the *quality* of whatever capacity remains, and a policy that successfully consumes almost everything necessarily leaves only slivers, so its ratio approaches 1. The absolute `Φ` measures how much is stranded: 246.6 versus 858.3 GPU-equivalents. Report `Φ` in GPU-equivalents and dollars as the primary figure, with placements-achieved beside it, and treat `φ` as a diagnostic about what is left rather than as a score to optimise. A programme steered by `φ` alone selects the worse policy.

- **What does defragmentation cost, and what is the go/no-go test?**
  **Answer:** Consolidation requires evicting and rescheduling running work: the cost is `Σ_j (time_since_checkpoint + restart + queue_wait) × GPUs_j × r`, where `time_since_checkpoint` averages half the checkpoint interval for a uniformly-timed preemption, and a workload that does not checkpoint contributes all of its elapsed runtime — which is why non-checkpointing jobs are effectively immovable. The gain is `Φ_recovered × r × T_useful`, where `T_useful` is the time the recovered capacity is actually occupied by work that could not previously be placed, *not* the time it merely exists. Setting gain above cost cancels `r`, so the test is a rate-independent statement in GPU-hours: `Φ_recovered × T_useful > Σ_j (t_ckpt + t_restart + t_queue) × G_j`. On the twelve-node example, freeing one whole node costs 6.0 GPU-hours and recovers 7.8 GPU-equivalents, so it pays after about 46 minutes of real use — and pays never if nothing is pending in that shape. Consolidate against a queue, not against a ratio.

- **A vendor tells you fractional GPU sharing will solve your fragmentation. What do you check first?**
  **Answer:** The remainder-servability curve of your own demand. Sharing does not remove fragmentation; it relocates it from between nodes to inside devices, and every fractional placement manufactures a remainder whose value is `P(ask ≤ remainder)` over the arriving distribution. On the published Alibaba trace, the single most common fractional ask is 810 milli-GPU (15.3% of all GPU requests), leaving 190 milli that only 1.50% of arrivals can use — roughly 205 GPU-equivalents of near-dead capacity created by the ask granularity alone, independent of any scheduler. So: measure the fractional ask histogram, compute the servability curve, and check whether the popular sizes tile 1,000 evenly. If they do not, the first intervention is a quota policy quantising asks to sizes that do — which costs nothing — and only then the sharing mechanism.

- **Why can't your cost tool tell you this, and what does the join look like?**
  **Answer:** OpenCost's idle allocation is `assetTotal − allocTotal` per node or cluster, clamped at zero (lesson 03 §9) — that is `F × r`, the *size* of the free bucket with no model of shape. Computing usability needs two inputs a billing pipeline does not hold: the free-capacity **layout** (per node, and per device where sharing is in use, not just the total) and the **demand distribution** (from the pending queue and recent admissions, which lives in the scheduler or the queueing system). The join your operator performs is therefore: node inventory and allocation records (which you already collect for lesson 02's allocated ledger) × the request-shape histogram from the queue × the per-model rate card, producing `Φ`, `φ` and a stranded dollar figure per shape. Nothing in the ecosystem does that join, which is precisely why it belongs in the deliverable.

## Connections & what's next

Backward: this lesson consumes lesson 02's gap A — the paid-for-but-unallocated bucket that lesson 03 deliberately handed off — and gives it a mechanism, a measure and a price. It reuses module 06's placeability identity unchanged so the two lessons' numbers agree exactly, and it consumes lesson 01's MIG geometry (the 16.9% structurally unallocatable memory at 7×`1g.5gb` on A100-40GB) as the floor under shape (b), plus DRA's object model as the structural fix.

Forward: lesson 05 needs `Φ` directly. A unit cost computed from directly-attributed GPU-hours is a *direct* cost; the *fully-loaded* number must absorb the fleet's stranded capacity along with its idle, and the loading multiplier is built from exactly the buckets lessons 03 and 04 measured. Lesson 06 needs it too, from the other direction: a fleet with high structural fragmentation must contract for **more** capacity than its raw utilisation suggests, because a fraction of every commitment will be unplaceable — commit against effective capacity, not physical. Lesson 08 decides who is told about it, and lesson 02's rule applies: gap A is the platform's own number and does not belong on a tenant's invoice.

Next: **lesson 05** — the synthesis, where GPU-hours and dollars finally become a number the business recognises.

## References & further reading

**Primary sources — the data**

1. **Alibaba `clusterdata` — `cluster-trace-gpu-v2023`** — https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2023 — fetched and analysed directly. `openb_node_list_gpu_node.csv` (1,213 GPU nodes, 6,212 GPUs across G2/G3/V100M16/V100M32/T4/P100/A10) and `openb_pod_list_default.csv` (8,152 tasks; `num_gpu`, `gpu_milli` for fractional asks, `gpu_spec` for type constraints, QoS, phase, creation/deletion/scheduled times). The README notes ~33% of GPU tasks in the production cluster carry GPU-type constraints, captured in the `gpuspec33` variant. Every trace figure in §5, §6 and the worked example was computed from these two files.
2. **Alibaba `clusterdata` — `cluster-trace-gpu-v2026`** — https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2026 — the successor trace behind the OSDI '26 characterization (up to 155,410 GPUs across 37,707 servers over six months), for stress-testing a calculator built on the 2023 data.
3. **hkust-adsl — `kubernetes-scheduler-simulator`** — https://github.com/hkust-adsl/kubernetes-scheduler-simulator — fetched directly. The FGD authors' simulator, which evaluates FGD against Best-fit, Dot-product, GPU Packing, GPU Clustering and Random-fit on these traces; the reference implementation to compare your own replay against.

**Primary sources — the schedulers**

4. **Kubernetes — `pkg/scheduler/framework/plugins/noderesources/most_allocated.go`** — https://github.com/kubernetes/kubernetes — fetched directly. `mostRequestedScore` = `requested × MaxNodeScore / capacity` with `MaxNodeScore = 100`, averaged across resources by weight.
5. **Kubernetes — `pkg/scheduler/apis/config/v1/defaults.go` and `types_pluginargs.go`** — fetched directly. `SetDefaults_NodeResourcesFitArgs` installs `LeastAllocated` over the default resource set (`cpu`, `memory`), weights default to 1 and are validated to 1–100. This is the primary source for "spread is the default and GPUs are not scored unless listed."
6. **Volcano — `pkg/scheduler/plugins/binpack/binpack.go`** — https://github.com/volcano-sh/volcano — fetched directly. `ResourceBinPackingScore` = `(used + requested) × weight / capacity`; the `binpack.weight` / `binpack.cpu` / `binpack.memory` / `binpack.resources` / `binpack.resources.<name>` arguments; and the `continue` that skips any resource absent from the configured map.
7. **NVIDIA KAI Scheduler** — https://github.com/NVIDIA/KAI-Scheduler — fetched directly. Bin-packing versus spread, **workload consolidation** (relocating running workloads to open contiguous capacity), **min-guaranteed-runtime** (the anti-thrash floor consolidation requires), GPU sharing and queue reclaim strategies.
8. **Kubernetes SIG-Scheduling — KEP-4815, DRA partitionable devices** — https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/4815-dra-partitionable-devices — the `CounterSet` / `ConsumesCounters` model and the A100-40GB counter example (memory 40Gi, 98 multiprocessors, 7 copy engines, eight named memory slices; `1g.5gb` consuming 4,864 MiB / 14 MP / 1 copy engine / one memory slice) that both §7's arithmetic and lesson 01's use.
9. **Kueue — concepts and gang admission** — https://kueue.sigs.k8s.io/docs/concepts/ — all-or-nothing admission, the lever that prevents partial placements from half-filling nodes; also the queue that supplies the pending-shape histogram this lesson's measure requires.
10. **NVIDIA — MIG user guide** — https://docs.nvidia.com/datacenter/tesla/mig-user-guide/ — profile menu, placement rules, and the requirement that reconfiguration drains the physical GPU. (docs.nvidia.com was unreachable from this environment; the geometry arithmetic in §7 is taken from KEP-4815's counter model, which is reachable and is the more precise source anyway.)

**Research and production reports**

11. **Weng et al. — "Beware of Fragmentation: Scheduling GPU-Sharing Workloads with Fragmentation Gradient Descent" (USENIX ATC '23)** — https://www.usenix.org/conference/atc23/presentation/weng — the fragmentation measure plus the FGD heuristic, trace-evaluated on an emulated cluster of 6,200+ GPUs, reporting up to **49%** fewer unallocated GPUs and roughly **290 GPUs** recovered versus packing baselines. **Correction:** the "21–42% of unallocated GPU becomes fragmentation" figure previously attributed to this paper here could not be confirmed against any reachable primary source and has been removed. usenix.org and the authors' PDF host are blocked from this environment; the trace repository (reference 1) is the reachable primary source and is what this lesson's own numbers come from.
12. **Li et al. — "Heterogeneity at Hyperscale: Characterization and Scheduling of Large Production AI Clusters at Alibaba" (OSDI '26)** — https://www.usenix.org/conference/osdi26/presentation/li-suyi — the six-month, 155,410-GPU characterization that treats stranded capacity as a first-class phenomenon. (usenix.org unreachable here; the companion trace at reference 2 is fetchable.)
13. **Choudhury et al. — "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale" (OSDI '24)** — https://www.usenix.org/conference/osdi24/presentation/choudhury — temporal and scope decoupling plus exhaustive search, reported to move the most overloaded region's high-priority demand-to-supply ratio from 2.63 to 0.98. **Provenance:** usenix.org unreachable from this environment; those two figures are carried from the previous version of this lesson and should be re-checked against the PDF before being quoted.
14. **NVIDIA Developer Blog — "Practical Tips for Preventing GPU Fragmentation for Volcano Scheduler"** — https://developer.nvidia.com/blog/practical-tips-for-preventing-gpu-fragmentation-for-volcano-scheduler/ — spread → bin-pack driving node fragmentation from double digits to under 1% with occupancy toward ~90%, no hardware added. **Provenance:** developer.nvidia.com is blocked from this environment; the URL is confirmed and the plugin mechanics in §8 are quoted from the Volcano source instead.

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
