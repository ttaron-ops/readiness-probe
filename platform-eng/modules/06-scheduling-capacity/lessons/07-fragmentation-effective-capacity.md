---
lesson: "06.7"
title: "Fragmentation & effective capacity"
module: "06"
concept: "Fragmentation & effective capacity"
status: not-started
est_time: "9h"
prev: "06-topology-aware-placement.md"
next: "08-priority-preemption-capacity-economics.md"
artifacts: []
sources: 11
---

# 06.7 · Fragmentation & effective capacity

> **Concept.** Indivisible bin-packing means an allocated fleet is not a usable fleet — measure the gap and you find hidden GPUs.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

Lesson 6 taught you to place a gang *correctly* — inside one topology domain, so the all-reduce
runs on NVLink rather than on the spine. It ended on a hook: a `required` topology constraint can
refuse to admit a job on a fleet whose free-GPU count looks perfectly healthy, and TAS will wait
rather than defragment for you. This lesson is the general form of that observation.

The question here is not "where should this job go?" but **"how much of this fleet can I actually
use, and for what job shapes?"** The gap between allocated capacity and usable capacity is
fragmentation, and it is invisible on every dashboard that counts busy GPUs instead of placeable
jobs. Lesson 8 then takes the number you produce here and asks the follow-up: who gets to reclaim
the stranded capacity, at what priority, and how much of the fleet do you commit to buying versus
renting. **Fragmentation math is the input; commitment strategy is the output.**

## Why this matters

Your dashboards say the fleet is 90% allocated, so "we're basically full, buy more." Repeated
across a quarter, that sentence is how a GPU org spends a seven-figure capex line on capacity it
already owns.

The FinOps edge a pure SWE misses: **allocated capacity and usable capacity are different
numbers, and the gap is structurally invisible.** It does not show up as an idle GPU on a Grafana
panel — every node looks busy — yet an 8-GPU training job cannot start because the free GPUs are
smeared two here, three there, one over there. Nobody is wasting a GPU. The fleet is
*fragmented*.

This is no longer a theoretical concern you have to argue for. Alibaba's OSDI '26 characterisation
of its production AI fleet — a **six-month trace covering up to 155,410 GPUs across 37,707
servers**, jobs from 81 departments — states the finding directly: high GPU demand does not
produce high effective utilisation, because idle GPUs frequently become *unallocatable*. It names
three mechanisms, and you should be able to name all three too: free capacity **stranded across
nodes**, free GPUs with **no matching CPU** on the same node, and free GPUs that **violate
network-locality constraints**. A fourth, deliberate one: users reserving headroom for production
safety. Their deployed fixes recovered real capacity — a defragmentation algorithm that cut the
number of nodes carrying slack resources by **20.2%**, and a preemption-aware harvesting framework
(SpotGPU) that raised the GPU allocation ratio from **68% to 93%**.

Read that last pair of numbers twice. A hyperscaler with world-class engineering was running at
68% allocation, and the recovery came from *scheduling*, not from procurement. That is the shape
of the story you want to be able to tell, with your own fleet's arithmetic behind it.

The durable claim: **you can only place indivisible, co-located jobs, so the right denominator for
"how full are we?" is not free GPUs — it is placeable jobs.** Learn to compute that number and you
can defend "don't buy" with arithmetic.

## What's new here (calibration)

- **Lessons 1–2** gave you why a job is indivisible (the collective barrier) and how atomic
  admission enforces it. **Lesson 6** gave you why it must also be co-located. Both are assumed;
  this lesson takes "the job is an indivisible, co-located block of `k` GPUs" as given and asks
  what that does to capacity accounting.
- **Lesson 5** named the scheduler-side levers — Volcano's `binpack` plugin scoring function,
  KAI's `consolidate` action. Not re-derived; §9 and §11 connect them to the arithmetic and add
  the `kube-scheduler` side, which lesson 5 did not cover.
- **Genuinely new here:**
  - **Three distinct capacity numbers** — raw, free, placeable — and the exact formula separating
    them, with the derivation rather than the assertion.
  - **A closed-form fragmentation model.** Under independent random occupancy at utilisation `u`
    on `G`-GPU nodes, the fraction of naive capacity that survives for whole-node jobs is exactly
    `(1−u)^(G−1)`. That single expression explains why spread placement is catastrophic for large
    jobs and why the number collapses so fast.
  - **A measured spread-vs-bin-pack comparison** on a simulated 128-GPU fleet across five
    utilisation levels, showing the same fleet with *more* free GPUs being worth *less*.
  - **The FGD fragmentation measure**, and how it generalises the single-job-size formula to a
    demand distribution — the mechanism behind the ATC '23 paper, not just its title.
  - **Multi-dimensional fragmentation.** GPUs are not the only axis; a node with 4 free GPUs and
    no free CPU is stranded just as thoroughly. Alibaba names this explicitly.
  - **`kube-scheduler`'s actual scoring functions** (`LeastAllocated`, `MostAllocated`,
    `RequestedToCapacityRatio`) with their formulas from source — and the default that silently
    breaks GPU bin-packing for almost everyone.
  - **Defrag as a priced flow**, with the payback inequality and the checkpoint term that turns
    it from cheap to catastrophic.
  - **MIG's second fragmentation axis**: profile geometry, with the real H100-80GB profile menu.

## Core concepts

### 1. Three numbers, and only one of them pays your bills

Write these down separately, because conflating any two of them is the whole error this lesson
exists to prevent.

| Name | Definition | Where you see it |
|---|---|---|
| **Raw capacity** | GPUs the fleet physically owns | the invoice, `Σ node.status.capacity` |
| **Allocated** / **free** | GPUs currently handed to a pod / not handed to a pod | every dashboard, `kube_node_status_allocatable` minus requests |
| **Placeable** (effective, usable) | GPUs that can still form a **new whole job** of a given shape | nowhere, unless you compute it |

The first two are scalars over a fungible pool. The third is not, because the thing you are
placing is not a scalar. That is the entire lesson in one sentence, and the next section makes it
precise.

### 2. Why a job is a block, not a quantity

A distributed training job needs `k` GPUs that are **simultaneously present** (lesson 1: the
collective barrier means N−1 ranks make zero progress), **atomically admitted** (lesson 2), and
**co-located within a topology domain** (lesson 6: a scattered gang pays the collective tax).
Put those together and the resource request is not "k GPUs from the fleet". It is:

> a contiguous block of `k` GPUs drawn from **one** node — or from one topology domain, if `k`
> exceeds a node — all free at the same instant.

You cannot run six GPUs of it here and two there and call it done. That is the classic
**bin-packing** constraint: items of fixed size, bins of fixed capacity, and an item that does not
fit in any single bin does not run, no matter how much total free space exists across bins.

The consequence for capacity accounting is immediate. The naive formula treats GPUs as a fungible
liquid:

```
  placeable_naive(k) = floor( total_free_GPUs / k )                  ← WRONG
```

The honest formula respects node (or domain) boundaries:

```
  placeable_real(k)  = Σ over nodes i of  floor( free_i / k )        ← RIGHT

  stranded_GPUs(k)   = total_free − k × placeable_real(k)
  fragmentation(k)   = 1 − placeable_real(k) / placeable_naive(k)
```

Both formulas take the same input. They can differ by 100%.

### 3. The same fleet, two answers

Sixteen 8-GPU nodes — 128 GPUs, which is the fleet the module's deliverable designs for. Free-GPU
counts per node, at 75% allocated:

```
  inventory = [3, 3, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 1, 1]        total_free = 32
```

Naive answer for 8-GPU jobs: `floor(32/8) = 4`. Four flagship jobs' worth of headroom, says the
dashboard.

Real answer:

```
  floor(3/8) + floor(3/8) + floor(2/8)×12 + floor(1/8)×2
    = 0 + 0 + 0 + 0
    = 0
```

**Zero.** You are carrying 32 free GPUs — a quarter of the fleet, and at a **$2.35/GPU-hr H100
on-demand snapshot** (specialized-neocloud segment; see lesson 8's market-segment table before
you quote any rate) that is `32 × $2.35 = $75.20/hr` or **$658,752/yr** of paid-for capacity —
and you cannot start a single 8-GPU job. Nor a single 4-GPU job. The fragmentation at k=8 is
`1 − 0/4 = 100%`.

That inventory is not adversarial. It is what a **spread** placement policy produces at 75%
utilisation, which is what `kube-scheduler` does by default (§9).

### 4. Fragmentation is job-size dependent — always report the table

There is no single "fragmentation %". Run the same inventory through every job size that matters:

```
  fleet: 16 nodes, 32 free GPUs, rate $2.35/GPU-hr (SNAPSHOT — neocloud on-demand)
    k  naive  real  stranded   frag%      $/hr         $/yr
    1     32    32         0    0.0%      0.00            0
    2     16    14         4   12.5%      9.40       82,344
    4      8     0        32  100.0%     75.20      658,752
    8      4     0        32  100.0%     75.20      658,752
```

The same fleet is **0% fragmented for 1-GPU notebooks and 100% fragmented for 4- and 8-GPU
training**. A fleet perfectly tuned for a horde of single-GPU notebooks is a disaster for 8-GPU
pretraining, and vice versa.

**A single fleet-wide fragmentation percentage reported to leadership is actively misleading**,
because it invites the wrong fix. "We're 25% fragmented" sounds like a capacity problem and gets
answered with a purchase order. "We are 100% fragmented at our flagship job size and 0% at our
notebook size, because placement policy spreads" is a scheduling problem and gets answered with a
config change that costs nothing.

If you must produce one number for a mixed workload, weight it by demand — §7 gives the principled
way to do that.

### 5. The closed form: why spread placement collapses so fast

The table above is one inventory. To reason about fleets in general, model it.

**Assumptions** (state them, because the model is only as good as they are): nodes have `G` GPUs
each; each GPU is independently occupied with probability `u` (the fleet utilisation); jobs need
`k` co-located GPUs. Independence is the "no packing logic at all" case — it is what you get from
placement that ignores node fullness, and it is a *lower bound* on what a real scheduler achieves.

Free GPUs on a node then follow a binomial distribution, `F ~ Binomial(G, 1−u)`, and:

```
  E[ placeable_real(k) ] = N × E[ floor(F/k) ] = N × Σ_{f=0..G} C(G,f)(1−u)^f u^(G−f) × floor(f/k)
  placeable_naive(k)     = N × G × (1−u) / k
```

For the case that matters most — a job that needs a **whole node**, `k = G` — the expectation
collapses to a single term, because `floor(f/G)` is 1 only when `f = G`:

```
  E[ placeable_real(G) ] = N × (1−u)^G
  placeable_naive(G)     = N × (1−u)
  ────────────────────────────────────────
  ratio = (1−u)^(G−1)
```

**That is the whole phenomenon in one expression.** On 8-GPU nodes, the fraction of naive
whole-node capacity that survives random placement is `(1−u)^7`:

| Utilisation `u` | free GPUs/node | `E[placeable]` k=1 | k=2 | k=4 | k=8 | naive k=8 | ratio `(1−u)^7` |
|---|---|---|---|---|---|---|---|
| 50% | 4.00 | 4.000 | 1.750 | 0.641 | 0.0039 | 0.500 | **0.78%** |
| 60% | 3.20 | 3.200 | 1.350 | 0.407 | 0.00066 | 0.400 | 0.16% |
| 70% | 2.40 | 2.400 | 0.950 | 0.194 | 0.00007 | 0.300 | 0.023% |
| 80% | 1.60 | 1.600 | 0.554 | 0.056 | ~0 | 0.200 | 0.0013% |
| 90% | 0.80 | 0.800 | 0.192 | 0.005 | ~0 | 0.100 | 8×10⁻⁶ |

*(Per node, `N = 1`; multiply by node count for a fleet.)*

Two things to take from this. First, the collapse is **exponential in node size**, not linear in
utilisation: even at a comfortable-sounding 50% utilisation, random placement retains under 1% of
its nominal whole-node capacity. Second, the exponent is `G−1`, so **bigger nodes make it worse**.
Moving from 8-GPU to 16-GPU nodes squares an already tiny number. GB200 NVL72's 72-GPU NVLink
domain, treated as one bin, is the extreme case — which is exactly why topology-aware schedulers
have to reason about domains rather than let placement fall out of per-node scoring.

The model's job is not to predict your fleet. It is to tell you that **for large-`k` jobs,
placement policy is not a 10% optimisation — it is the difference between a working fleet and a
fleet that cannot start its most important jobs.** The next section measures how much of that a
real policy recovers.

### 6. Measured: spread versus bin-pack on the same fleet

The closed form assumes no packing logic. Real schedulers have some. To size the gap, here is a
simulation of the 128-GPU fleet (16 × 8 GPUs) under a job mix of 55% 1-GPU, 25% 2-GPU, 15% 4-GPU
and 5% 8-GPU by job count, with random departures, averaged over 200 seeded runs at each
utilisation level. Two policies:

- **spread** — place on the node with the *most* free GPUs (`kube-scheduler`'s default
  `LeastAllocated`).
- **bin-pack** — place on the node with the *fewest* free GPUs that still fits
  (`MostAllocated`).

| Utilisation | free GPUs (spread / pack) | placeable k=4 (spread / pack) | placeable k=8 (spread / pack) | naive k=8 |
|---|---|---|---|---|
| 50.0% | 63.5 / 63.1 | 11.84 / **14.36** | 0.01 / **6.46** | 7.9 |
| 62.5% | 47.6 / 47.2 | 5.80 / **10.21** | 0.00 / **4.29** | 5.9 |
| 75.0% | 31.8 / 31.2 | 0.24 / **6.11** | 0.00 / **2.19** | 3.9 |
| 87.5% | 16.1 / 15.4 | 0.00 / **2.06** | 0.00 / **0.27** | 2.0 |
| 90.0% | 13.1 / 12.5 | 0.00 / **1.46** | 0.00 / **0.10** | 1.6 |

Read the 75% row as the headline. **Spread placement leaves you 32 free GPUs and zero placeable
8-GPU jobs. Bin-pack leaves you 31 free GPUs and two.** The fleet with *more* free GPUs is worth
*less*, because the free GPUs are in the wrong shape. Two representative single-seed inventories
from that row, so you can verify the arithmetic by hand:

```
  BIN-PACKING PICTURE — 16 nodes × 8 GPUs, 75% allocated, two placement policies
  ══════════════════════════════════════════════════════════════════════════════════════
   █ = allocated GPU     · = free GPU     ✗ = free but UNUSABLE by an 8-GPU job

   SPREAD  (kube-scheduler default: LeastAllocated — "put it where there's most room")
   each row is one node, eight cells = eight GPUs
   n00 █████✗✗✗     n01 █████✗✗✗     n02 ██████✗✗     n03 ██████✗✗
   n04 ██████✗✗     n05 ██████✗✗     n06 ██████✗✗     n07 ██████✗✗
   n08 ██████✗✗     n09 ██████✗✗     n10 ██████✗✗     n11 ██████✗✗
   n12 ██████✗✗     n13 ██████✗✗     n14 ███████✗     n15 ███████✗
       free = 3,3,2,2,2,2,2,2,2,2,2,2,2,2,1,1  = 32 GPUs
       placeable 8-GPU jobs: 0        placeable 4-GPU jobs: 0
       EVERY free GPU is ✗ for the flagship job size.  $75.20/hr of nothing.

   BIN-PACK  (MostAllocated — "put it on the fullest node that still fits")
   n00 ········     n01 ········     n02 ███·····     n03 ████····
   n04 ██████··     n05 ██████··     n06 ███████·     n07 ████████
   n08 ████████     n09 ████████     n10 ████████     n11 ████████
   n12 ████████     n13 ████████     n14 ████████     n15 ████████
       free = 8,8,5,4,2,2,1,0,0,0,0,0,0,0,0,0  = 30 GPUs
       placeable 8-GPU jobs: 2 (n00, n01)      placeable 4-GPU jobs: 6
       stranded at k=8: 14 GPUs ($32.90/hr) — still real, but 57% less

   ═══ SAME FLEET.  SAME UTILISATION.  TWO FEWER FREE GPUs, TWO MORE FLAGSHIP JOBS. ═══
```

The economic statement, computed rather than asserted: at k=8, spread makes **0 GPUs** placeable
and bin-pack makes **16**. At $2.35/GPU-hr that is `16 × 2.35 = $37.60/hr`, `$329,376/yr` of
capacity that becomes schedulable from a scheduler configuration change.

**State the honest caveat with the number**, because a strong candidate does and a weak one does
not: recovered capacity is only worth money to the extent there is queued demand to consume it.
The correct claim is `min(recovered_placeable_GPUs, queued_demand_GPUs) × rate`. If nobody has an
8-GPU job waiting, the recovery is worth exactly zero this hour. Your queue depth metrics —
Kueue's pending-Workload counts and `kueue_quota_reserved_wait_time_seconds`, from lesson 3 — are
the cap on the claim.

### 7. One number for a mix of job sizes: the FGD measure

Reporting a table per job size is right, but sometimes you need a single scalar — to rank nodes
during scheduling, or to trend fragmentation over time. The principled way to collapse the table
is to weight it by the demand distribution, and that is exactly what the FGD paper (Weng et al.,
USENIX ATC '23) formalises.

Their construction, in the paper's own terms: for a node `n` and a workload distribution `M` over
task classes,

```
  F_n(M) = Σ_{m ∈ M}  p_m · F_n(m)

     p_m    = popularity (probability) of task class m in the historical workload
     F_n(m) = the amount of node n's UNALLOCATED GPU that a task of class m cannot use
```

`F_n(m)` is the per-node, per-class version of the `stranded` column in §4's table. Summing over
nodes gives cluster fragmentation; weighting over classes gives the expected number of free GPUs
that the *next arriving job* will not be able to touch.

The scheduling policy that falls out — **Fragmentation Gradient Descent** — is greedy on the
derivative: for each candidate node, compute how much `F(M)` would *increase* if this task were
placed there, and choose the node that minimises the increase. It is the same instinct as
bin-packing, but generalised: instead of "prefer the fullest node", it is "prefer the node where
placing this job destroys the least future placeability, given what we know about future demand."
Bin-packing is the special case where the demand distribution is a point mass at the largest job
size.

The reported result on Alibaba production traces, evaluated on an emulated cluster of over 6,200
GPUs: FGD **reduces unallocated GPUs by up to 49%**, putting an additional **290 GPUs** to work
versus packing-based baselines including Best-Fit and Dot-Product. Note what is and is not being
claimed — this is an emulation driven by real traces, not a production A/B test, and "up to 49%"
is the best case in their sweep. Naming that precisely is a stronger interview answer than "there
is a paper about GPU fragmentation."

For your own fleet the practical version is a one-liner over the same table:

```python
def expected_stranded(inventory, mix):
    """mix: {job_size: probability}. Returns E[stranded GPUs] for the next arrival."""
    return sum(p * effective_capacity(inventory, k).stranded_gpus for k, p in mix.items())
```

On §3's spread inventory with a mix of `{1: 0.55, 2: 0.25, 4: 0.15, 8: 0.05}`, that evaluates to
**7.40 expected stranded GPUs** — a defensible single number, with its assumption (the mix)
stated on the same line.

### 8. GPUs are not the only axis: multi-dimensional fragmentation

Everything so far treated a node as a one-dimensional bin. It is not. A pod requests CPU, memory,
ephemeral storage and GPUs, and `NodeResourcesFit`'s `Filter` rejects the node if *any* dimension
fails. This is **vector bin-packing**, and it is strictly harder than the scalar version — the
scalar problem is already NP-hard, and First-Fit-Decreasing's classic `11/9 · OPT + 6/9` guarantee
does not carry over to the vector case in general.

Alibaba's trace names this as one of the three stranding mechanisms: free GPUs that **lack
matching CPUs**. The failure is easy to construct and depressingly common. A dataloader-heavy
training pod asks for 8 GPUs and 96 CPU cores. A node has 8 free GPUs and 40 free cores, because
a batch of CPU-heavy preprocessing pods landed there. The GPUs are free, visible, billed — and
unusable.

The fix in the calculator is to take the minimum across dimensions:

```python
def effective_capacity_2d(nodes, k_gpu, k_cpu):
    """nodes[i] = (free_gpus, free_cpu_cores). A node hosts
       min(free_gpu // k_gpu, free_cpu // k_cpu) copies of the job."""
    real = sum(min(g // k_gpu, int(c // k_cpu)) for g, c in nodes)
    ...
```

Two operational consequences:

- **Reserve CPU and memory alongside GPUs, in proportion.** If your GPU nodes have 8 GPUs and 192
  cores, the natural per-GPU ration is 24 cores. Admitting a non-GPU pod that eats 100 cores on a
  GPU node has stranded roughly four GPUs' worth of CPU. Taints on GPU nodes plus tolerations only
  on GPU workloads is the blunt instrument; a per-queue CPU quota alongside the GPU quota
  (lesson 3) is the precise one.
- **The third mechanism, network locality, is lesson 6 in this vocabulary.** A `required` topology
  constraint adds a dimension that is not even a quantity — it is a *label equality* constraint
  across the whole block of `k` GPUs. Free GPUs in two different racks cannot combine into one
  rack-required job, which is fragmentation with an extra dimension attached.

So the honest fleet report has three axes, not one: job size `k`, resource vector, and topology
level. In practice: report `placeable_real` per (job profile, topology level) pair. That is a
small table, and it is the table that decides purchases.

### 9. The policy lever: how placement actually gets chosen

`kube-scheduler` decides *where* a pod lands with the `Score` phase, and the plugin that matters
is `NodeResourcesFit`. Three scoring strategies, straight from
`pkg/scheduler/framework/plugins/noderesources/` in the `kubernetes/kubernetes` tree:

**`LeastAllocated`** — the default. Favours the *emptiest* node:

```
  score_r  = (allocatable_r − requested_r) × 100 / allocatable_r      per resource r
  node     = Σ_r (score_r × weight_r) / Σ_r weight_r
```

More free capacity → higher score. This is a **spread** policy. It is the right default for
stateless services (blast radius, noisy neighbours) and precisely wrong for scarce indivisible
GPU jobs.

**`MostAllocated`** — favours the *fullest* node:

```
  score_r  = requested_r × 100 / allocatable_r
  node     = Σ_r (score_r × weight_r) / Σ_r weight_r
```

This is **bin-packing**. It coalesces free capacity into whole-node holes.

**`RequestedToCapacityRatio`** — lets you draw the curve yourself, as a piecewise-linear function
over utilisation:

```
  score_r  = shape( requested_r × 100 / allocatable_r )
  where shape is a broken-linear function defined by (utilization, score) points
```

With shape `[{0,0},{100,10}]` it reproduces `MostAllocated`; with `[{0,10},{100,0}]` it
reproduces `LeastAllocated`; and you can express things neither can, such as "strongly prefer
nodes above 50% but do not care above 90%".

**Now the trap, and it is a big one.** The default `ScoringStrategy.Resources` list is:

```go
var defaultResourceSpec = []configv1.ResourceSpec{
    {Name: string(v1.ResourceCPU),    Weight: 1},
    {Name: string(v1.ResourceMemory), Weight: 1},
}
```

**`nvidia.com/gpu` is not in it.** Switching to `MostAllocated` and stopping there gives you
bin-packing *on CPU and memory*, which on a GPU fleet is close to noise. You must name the
extended resource explicitly and weight it above CPU and memory, or the change does nothing for
the thing you care about. This is the single highest-value configuration fact in the lesson.

A complete, correct configuration:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: MostAllocated              # bin-pack, not the LeastAllocated default
            resources:
              - name: nvidia.com/gpu         # MUST be listed — not scored by default
                weight: 10                   # dominate the score
              - name: cpu
                weight: 1
              - name: memory
                weight: 1
    plugins:
      score:
        enabled:
          - name: NodeResourcesFit
            weight: 5                        # default plugin weight is 1
        disabled:
          - name: NodeResourcesBalancedAllocation   # see below
```

Two annotated lines deserve explanation:

- **`weight: 5` on the plugin.** The default score weights are `TaintToleration: 3`,
  `NodeAffinity: 2`, `PodTopologySpread: 2`, `InterPodAffinity: 2`, `NodeResourcesFit: 1`,
  `NodeResourcesBalancedAllocation: 1`, `ImageLocality: 1`. At weight 1, your bin-packing signal
  is one vote among several and gets outvoted by image locality and topology spread.
- **Disabling `NodeResourcesBalancedAllocation`.** That plugin scores a node by how *balanced* the
  resulting CPU/memory fractions would be — its score sits in `[50, 100]` and moves toward 100
  when placing the pod improves the node's balance. On a GPU fleet it is orthogonal noise at best
  and actively counter-packing at worst, and it defaults to enabled at weight 1. Evaluate whether
  you want it; do not leave the decision unmade.

Also note what scoring is *not*: `percentageOfNodesToScore` (adaptive by default, with hard floors
of 100 nodes / 5%) means the scheduler may not even look at every feasible node on a large fleet,
so on a 2,000-node cluster your bin-packing preference is applied to a sample. That is a
throughput/quality tradeoff, and it caps how good the packing can be.

The equivalents in the other schedulers, so you can answer this in any of the three (lesson 5 has
the depth):

| Scheduler | Bin-packing control | Notes |
|---|---|---|
| `kube-scheduler` | `NodeResourcesFit` `scoringStrategy: MostAllocated`, GPU listed and weighted | GPU absent from default resource list |
| Volcano | `binpack` plugin: `binpack.weight`, `binpack.resources: nvidia.com/gpu`, `binpack.resources.nvidia.com/gpu` | scores by resulting fullness; same GPU-weighting requirement |
| Kueue TAS | `unconstrained` placement uses `LeastFreeCapacity` by default since v0.15 (`TASProfileMixed`), explicitly "to prioritize minimizing fragmentation" | operates on topology domains, not raw nodes (lesson 6 §10) |
| KAI | domain selection prefers the *fullest* domain that still fits, by construction; plus the `consolidate` action | packing is a property of the loop, not a plugin |

The FinOps read, stated as the reversal it is: **spread optimises availability of a replaceable
thing; bin-pack optimises utilisation of a fixed, expensive asset.** Most engineers' reflex is
"spread is always safer", inherited from stateless-service operations. On owned GPUs that reflex
costs you the flagship job size. Being able to state the reversal *and* say when spread is still
correct — prod inference, where blast radius matters more than packing — is the differentiator.

### 10. Fragmentation is a stock; defrag is a flow with a price

You have a stock of stranded GPUs. Reducing it requires *moving running work*, and that is not
free `kubectl` magic. Mechanically, consolidating a node means:

```
  DEFRAG TIMELINE — freeing one 8-GPU node by relocating 3 GPUs of work
  ══════════════════════════════════════════════════════════════════════════════════════

  t0   SELECT victims
         │  eligible = checkpointing, low-priority, small, not a partially-movable gang
         │  ineligible = prod inference, anything without a resume path
         ▼
  t1   DRAIN signal: SIGTERM to the victim pods
         │  the clock now running is terminationGracePeriodSeconds (default 30 s)
         │  a well-behaved trainer catches SIGTERM and writes a final checkpoint
         ▼
  t2   ── LOSS WINDOW ─────────────────────────────────────────────────────────
         │  work destroyed = (time since last checkpoint) + (any failed final save)
         │  expected value over a uniformly-timed eviction = T_ckpt / 2
         ▼
  t3   POD DELETED, GPUs released on the source node
         │
         ▼
  t4   RE-QUEUE.  For a gang this is the whole gang (lesson 2) — it re-admits
         │  atomically or not at all, and it now queues behind everyone else.
         │  Queue wait here is NOT bounded by you.
         ▼
  t5   RESTART on the destination node: image (usually cached) + model + optimizer
         │  state load + framework/NCCL init + warm-up   = T_restart
         ▼
  t6   the source node is now WHOLE.  ┌────────────────────────┐
         │                            │  8 free, co-located    │
         │                            └────────────────────────┘
         ▼
  t7   a queued 8-GPU job claims the slot and runs for D hours
         └──── payback reached when  8 × D  >  Σ_j m_j (T_ckpt,j/2 + T_restart,j)

       IF NO 8-GPU JOB IS QUEUED, D = 0 AND THE ENTIRE OPERATION IS PURE LOSS.
```

The formal go/no-go:

```
  defrag_cost    = Σ over migrated jobs j of  m_j × ( T_ckpt,j / 2 + T_restart,j )   [GPU-hours]
  defrag_benefit = G × D × P(slot claimed within the horizon)                        [GPU-hours]

  do it  ⟺  benefit > cost

  where m_j = GPUs held by job j,  G = GPUs in the recovered hole,
        D   = hours the recovered slot is actually occupied by work that could
              not otherwise have run.
```

The `T_ckpt/2` term is the expectation of "time since last checkpoint" when the eviction lands at
a uniformly random point in a checkpoint cycle of length `T_ckpt`. Lesson 8 derives it properly
and then optimises `T_ckpt` itself.

**Worked instance.** Take §6's bin-packed inventory `[8,8,5,4,2,2,1,0,…]`. You want a *third*
whole node. Node n02 has 5 free (3 GPUs occupied); node n03 has 4 free (4 occupied). Move n02's
three GPUs of work onto n03 (which has room), and n02 becomes whole:

```
  migrate: 3 GPUs of work,  T_restart = 5 min = 0.083 h

  case A — synchronous checkpointing every 30 min (T_ckpt = 0.5 h):
     cost    = 3 × (0.5/2 + 0.083)          = 3 × 0.333 h  = 1.00 GPU-hours
     payback = 1.00 / 8 GPUs                = 0.125 h      = 7.5 minutes of slot use

  case B — async sharded checkpointing every 5 min (T_ckpt = 0.083 h; see lesson 8):
     cost    = 3 × (0.083/2 + 0.083)        = 3 × 0.125 h  = 0.375 GPU-hours
     payback = 0.375 / 8                    = 0.047 h      = 2.8 minutes of slot use

  case C — the victim does NOT checkpoint and has been running 40 h on 3 GPUs:
     cost    = 3 × (40 + 0.083)             = 120 GPU-hours
     payback = 120 / 8                      = 15 hours of slot use
     …and you have destroyed 40 hours of someone's research to save a scheduling problem.
```

Cases A and B say: **if your workloads checkpoint, defrag is essentially free** — under ten
minutes of payback. Case C says: **if they do not, defrag is a hostage situation.** That is the
hinge into lesson 8, and it is worth stating as a policy rather than a calculation:
*defragmentation is only economically usable on checkpointing workloads, and the platform's job is
to make checkpointing cheap enough that everything qualifies.*

Two further costs the arithmetic above hides:

- **Gang re-admission risk.** A multi-pod gang cannot partially move. Evicting it re-queues the
  whole thing, and it may wait behind other work — potentially a long time on a busy fleet, and
  potentially forever if it has a `required` topology constraint on a fleet you just made *less*
  suitable. Migrate the cheap-to-move (best-effort, single-pod, checkpointing) work and leave
  prod and large gangs pinned.
- **The stock refills.** Defrag is a one-shot flow against a stock that regenerates as jobs
  arrive and depart. If placement policy is still `LeastAllocated`, you will be defragging
  forever. **Fix the flow (policy) before you pay for the stock (defrag).** That ordering is the
  single most useful piece of operational advice in this lesson.

NVIDIA's KAI Scheduler (lesson 5) turns this into policy rather than a runbook: its `consolidate`
action runs *before* any disruptive action and evicts a pod only when it has already found it a
new home. Volcano's nearest analogue is the `shuffle` action driven by the `rescheduling` plugin,
which is a rebalancing hook rather than a fit-driven defrag. Kueue has no equivalent step at all —
it admits or waits.

### 11. MIG changes the surface, in both directions

MIG (Multi-Instance GPU, from module 04) partitions one physical GPU into isolated instances with
their own SM slices, memory slices and memory-bandwidth share. Its effect on fragmentation is
genuinely two-sided, and the second side is the one people miss.

**Direction one — finer granularity reduces waste for small demand.** A 1-GPU inference pod on a
full 80 GB H100 strands most of a device. Split into `1g.10gb` instances, seven such pods share it.
GPU-level packing improves and the "one big asset parked on a tiny job" problem shrinks.

**Direction two — MIG introduces a *new* fragmentation axis: profile geometry.** Profiles come
from a fixed menu with layout constraints, not free-form sizes. For an **H100-80GB** (verified
against NVIDIA's `mig-parted` shipped configuration, `nvidia-mig-manager-example-hopper-blackwell.yaml`):

| Profile | Compute slices | Memory | Max per GPU |
|---|---|---|---|
| `1g.10gb` | 1 | 10 GB | 7 |
| `1g.10gb+me` | 1 (+ media engines) | 10 GB | 1 |
| `1g.20gb` | 1 | 20 GB | 4 |
| `2g.20gb` | 2 | 20 GB | 3 |
| `3g.40gb` | 3 | 40 GB | 2 |
| `7g.80gb` | 7 | 80 GB | 1 |

The device exposes **seven** compute slices total, and mixed geometries must partition them. The
shipped "balanced" H100-80GB layout is `1g.10gb × 2 + 2g.20gb × 1 + 3g.40gb × 1` — 2 + 2 + 3 = 7,
exactly. You cannot have `3g.40gb × 2 + 2g.20gb × 1`, because that would need 8 slices. And
because instance placement is constrained to legal slice positions, a GPU carved into small
instances cannot serve a whole-GPU job until it is **drained of every workload and
reconfigured** — which is itself a defrag with all of §10's costs.

Net: MIG is a fragmentation *reducer* for stable, heterogeneous small demand and a fragmentation
*creator* when demand shifts back toward whole-GPU jobs. Match the geometry to a demand profile
that is actually stable, or you will pay drain costs continuously reshaping GPUs. The
decision-relevant question is not "should we use MIG" but "**is our small-job demand stable enough
that a fixed geometry will still be right in three months?**"

### 12. Deliberate stranding: headroom, reservations, and the bridge to preemption

Not all stranded capacity is an accident. Alibaba's trace names a fourth mechanism alongside the
three structural ones: **users reserve ample headroom for production safety.** A team that owns
32 GPUs of quota and runs 20 is not fragmenting anything — it is buying insurance, rationally,
against a traffic spike or a failed node.

This matters for the report you produce, because it is a different kind of number with a different
fix:

| Kind of stranding | Cause | Fix |
|---|---|---|
| Structural | indivisible jobs + spread placement | placement policy (§9), defrag (§10) |
| Dimensional | GPUs free, CPU/memory/locality not | proportional quota, taints, topology (§8) |
| Geometric | MIG profile mismatch | geometry matched to stable demand (§11) |
| **Reserved headroom** | deliberate safety margin, idle by design | **backfill with preemptible work** |

The last row is not a scheduling bug and you should not try to "fix" it by shrinking the
reservation — the team is right that they need the headroom. You fix it by running *someone else's
preemptible work* in the gap and yanking it back instantly when the owner needs it. That is
exactly Alibaba's SpotGPU result: preemption-cost-aware harvesting of idle-but-reserved capacity
took the GPU allocation ratio from **68% to 93%**.

And that mechanism is lesson 8's entire subject: priority tiers, reclaim, and the checkpointing
contract that makes eviction survivable. Fragmentation math tells you *how much* capacity is
recoverable; preemption economics tells you *how* to recover the deliberate portion and what it
costs when it goes wrong.

### 13. What to actually measure and publish

Concretely, for the deliverable and for a real platform:

1. **`placeable_real(k)` per dominant job profile**, sampled on a schedule (every 5 minutes) and
   trended — not a point-in-time snapshot. A snapshot can catch you mid-defrag (artificially
   fragmented) or right after a big job finished (artificially clean). Alibaba's six-month trace
   is the extreme version of this argument: only an average over the natural churn of arrivals,
   completions and diurnal/weekly cycles tells you the *steady-state* cost, which is what a
   capacity decision needs.
2. **`stranded_GPUs(k)` in dollars**, with the rate flagged as a snapshot and labelled by market
   segment, and the claim capped at queued demand.
3. **Queue depth per job size**, because it is the cap in (2) and because a fleet with zero
   8-GPU-job queue depth does not have an 8-GPU fragmentation problem worth spending on.
4. **The counterfactual under the other placement policy**, computed offline from the same
   inventory. This is the number that turns "we should change a config" into a funded decision.

The one-sentence summary for a capacity review: *"At our current 8-GPU job profile, effective
capacity is 0 jobs, not the 4 the free-GPU count implies; 32 free GPUs (~$75/hr at a snapshot
neocloud rate) are stranded by node-level fragmentation, and a placement-policy change recovers 16
of them at zero capital cost."*

## Perspectives

**Developer.** A researcher whose job "should fit" — the naive free-GPU arithmetic checks out —
but will not admit experiences fragmentation as an unexplained scheduler failure. From inside the
training loop there is no signal at all distinguishing "the cluster is genuinely full" from "the
cluster has 32 free GPUs that happen to be the wrong shape for my job." A platform team that can
hand back the per-`k` breakdown converts a "the scheduler is broken" ticket into a legible
conversation with an actual next step ("run at 4 GPUs today, or wait ~40 minutes for a node to
drain"). That legibility is itself a platform deliverable, not a nicety.

**Operator.** Fragmentation is a *policy* outcome, not only an emergent property, and the policy
is per-queue rather than fleet-wide. Prod-serving queues want spread — blast radius from a node
failure is the dominant risk. Training queues want bin-pack — utilisation of a fixed asset is the
dominant risk. `kube-scheduler` profiles let you run both: two `schedulerName`s with different
`NodeResourcesFit` scoring strategies, and pods selecting the profile that matches their queue.
The operator also owns the ordering rule from §10: fix the flow before paying for the stock.

**Research / theory.** The formal treatment is FGD (Weng et al., ATC '23), and its contribution is
precise enough to name: a fragmentation *measure* `F_n(M) = Σ_m p_m F_n(m)` — expected unusable
free GPU on a node, weighted by the historical popularity of each task class — plus a greedy
policy that minimises the measure's increase, evaluated against real Alibaba traces on an emulated
6,200-GPU cluster, reducing unallocated GPUs by up to 49%. "A measure plus a heuristic that
reduces it, trace-evaluated" is a much stronger sentence in an interview than "there's a paper
about GPU fragmentation." The wider theory is vector bin-packing, NP-hard, with no
FFD-style constant-factor guarantee carrying over from the scalar case.

**Economics.** "$X of hidden capacity" is the strongest FinOps story in the module, and its
strength comes entirely from being falsifiable: an inventory, a formula, a rate, a queue-depth
cap. Anchor it on the hyperscale evidence — 155,410 GPUs, six months, 68% → 93% allocation ratio
from scheduling changes — and then show your own fleet's number computed the same way. The
discipline that keeps it credible: report per job size, cap the claim at queued demand, and label
every rate as a dated snapshot from a named market segment.

## Real-world use cases

- **Alibaba — "Heterogeneity at Hyperscale: Characterization and Scheduling of Large Production AI
  Clusters at Alibaba" (OSDI '26)** — https://www.usenix.org/conference/osdi26/presentation/li-suyi.
  What it shows, with numbers: a six-month trace of Alibaba Serverless Infrastructure covering up
  to **155,410 GPUs across 37,707 GPU servers**, jobs from **81 departments**, spanning ad-hoc
  development, training, and online/offline inference. The central finding is this lesson's
  thesis stated by an operator at scale — high GPU *demand* does not yield high *effective*
  utilisation, because idle GPUs become unallocatable through three named mechanisms (stranded
  across nodes; no matching CPU; network-locality violations) plus deliberate user headroom. The
  deployed fixes: a defragmentation algorithm that cut nodes carrying slack resources by
  **20.2%**, and **SpotGPU**, a preemption-cost-aware harvesting framework that raised the GPU
  allocation ratio from **68% to 93%**. *(Search-verified this session; usenix.org is unreachable
  from this environment's egress proxy, so this is cited by canonical URL with the figures
  cross-checked across independent restatements.)*

- **Alibaba — `cluster-trace-gpu-v2026` public trace release** —
  https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2026. What it shows: the
  actual data behind the paper, downloadable. Four tables — `asi_opensource_pod_hourly` (pod
  workload, request, utilisation, priority, state, days 0–184 at hourly granularity),
  `asi_opensource_server_hourly` (server inventory and anonymised ASW/rack topology),
  `asi_opensource_network_hourly` (node-level rx/tx samples, days 109–115), and
  `asi_opensource_job_execution_summary`. Calendar dates, pod names, server serials and
  organisation fields are stripped. **This is the dataset to run your `effective_capacity.py`
  against** instead of a toy fleet: server inventory plus hourly pod requests is exactly the join
  the calculator needs, and the rack topology column lets you add lesson 6's locality dimension.
  **Fetched directly this session.**

- **Weng et al. — "Beware of Fragmentation: Scheduling GPU-Sharing Workloads with Fragmentation
  Gradient Descent" (USENIX ATC '23)** — https://www.usenix.org/conference/atc23/presentation/weng.
  What it shows: the formal fragmentation measure reproduced in §7 and the greedy policy that
  descends it, evaluated on Alibaba production traces over an emulated cluster of **more than
  6,200 GPUs**, against baselines including Best-Fit, Dot-Product, GPU Packing, GPU Clustering and
  Random-Fit. Result: **unallocated GPUs reduced by up to 49%**, putting an additional **290 GPUs**
  to work. Artifacts (the policy plus the baselines, and the trace) are public. Note the honest
  scope: emulation driven by real traces, and "up to 49%" is the best case in their sweep.

- **NVIDIA — "Practical Tips for Preventing GPU Fragmentation for Volcano Scheduler"** —
  https://developer.nvidia.com/blog/practical-tips-for-preventing-gpu-fragmentation-for-volcano-scheduler/.
  What it shows: on a DGX Cloud-provisioned Kubernetes cluster, integrating a bin-packing
  algorithm into the Volcano scheduler — consolidating workloads onto fewer nodes so other nodes
  stay whole for large jobs — achieved a **90% GPU occupancy rate against an 80% contractual
  target**, with no new hardware. The mechanism is §9's, applied through Volcano's `binpack`
  plugin. *(Search-verified; developer.nvidia.com is blocked by this environment's egress proxy.
  **Correction to the previous version of this lesson**, which claimed this source reported node
  fragmentation falling "from measurable double digits to under 1%" — that figure could not be
  confirmed against the source and has been replaced with the occupancy figures that could.)*

- **Alibaba — `cluster-trace-gpu-v2023`** —
  https://github.com/alibaba/clusterdata/blob/master/cluster-trace-gpu-v2023/README.md. What it
  shows: the earlier, smaller (~6,200-GPU) production trace that FGD's own evaluation is built
  from — confirming the paper's evidence base is real production data, and giving you a
  manageable first dataset to validate a calculator against before tackling the 155k-GPU 2026
  release. **Fetched directly this session.**

## Worked example

The 128-GPU fleet from the module deliverable: **16 nodes × 8 GPUs**, at 75% allocated. The task
is to produce the "$X of hidden capacity" claim and defend it.

**Step 1 — get the inventory.** Free GPUs per node, from the cluster rather than from a
dashboard:

```bash
# allocatable GPUs per node
$ kubectl get nodes -o json | jq -r '
    .items[] | select(.status.allocatable["nvidia.com/gpu"] != null)
    | "\(.metadata.name) \(.status.allocatable["nvidia.com/gpu"])"' > /tmp/alloc.txt

# committed GPUs per node (sum of limits over scheduled pods)
$ kubectl get pods -A -o json | jq -r '
    .items[] | select(.spec.nodeName != null)
    | .spec.nodeName as $n
    | [ .spec.containers[].resources.limits["nvidia.com/gpu"] // "0" | tonumber ] | add
    | select(. > 0) | "\($n) \(.)"' | awk '{a[$1]+=$2} END {for (n in a) print n, a[n]}' \
  > /tmp/used.txt

$ join <(sort /tmp/alloc.txt) <(sort /tmp/used.txt) \
    | awk '{printf "%s free=%d\n", $1, $2-$3}'
gpu-node-00 free=3
gpu-node-01 free=3
gpu-node-02 free=2
...
gpu-node-15 free=1
```

*(Representative transcript. The `join` drops nodes with zero GPU pods — add `-a1` and default the
missing column to 0 in a real script.)* Result:

```
  inventory = [3, 3, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 1, 1]     total_free = 32
```

**Step 2 — run the calculator.**

```
$ python3 effective_capacity.py
fleet: 16 nodes, 32 free GPUs, rate $2.35/GPU-hr (SNAPSHOT — label the market segment)
  k  naive  real  stranded   frag%      $/hr         $/yr
  1     32    32         0    0.0%      0.00            0
  2     16    14         4   12.5%      9.40       82,344
  4      8     0        32  100.0%     75.20      658,752
  8      4     0        32  100.0%     75.20      658,752

demand-weighted E[stranded GPUs] over mix {1: 0.55, 2: 0.25, 4: 0.15, 8: 0.05}: 7.40
```

**Step 3 — read it, honestly.** The dashboard says 25% free. The truth is: 32 GPUs free, **zero**
placeable 4-GPU or 8-GPU jobs, and 14 placeable 2-GPU jobs. If your flagship workload is 8-GPU
pretraining, your effective capacity for it is *nil*, and adding a seventeenth node would give you
one more whole node — one job — for the price of a whole node.

**Step 4 — compute the counterfactual.** What would the same fleet, same utilisation, look like
under `MostAllocated` with GPU weighted? From the simulation in §6:

```
$ python3 effective_capacity.py     # second report block
same fleet, same 75% utilisation, MostAllocated (bin-pack) placement:
fleet: 16 nodes, 30 free GPUs, rate $2.35/GPU-hr (SNAPSHOT)
  k  naive  real  stranded   frag%      $/hr         $/yr
  1     30    30         0    0.0%      0.00            0
  2     15    14         2    6.7%      4.70       41,172
  4      7     6         6   14.3%     14.10      123,516
  8      3     2        14   33.3%     32.90      288,204
```

```
  placeable 8-GPU jobs:      0  →  2          (+16 GPUs made schedulable)
  placeable 4-GPU jobs:      0  →  6          (+24 GPUs made schedulable)
  stranded at k=8:          32  →  14         (−18 GPUs, −56%)
  value of the k=8 recovery: 16 GPUs × $2.35  =  $37.60/hr  =  $329,376/yr
  capital cost of the change: a KubeSchedulerConfiguration edit and a restart
```

**Step 5 — cap the claim.** Before that $329k goes on a slide, check whether anyone wants it:

```bash
$ kubectl get workloads -A -o json | jq -r '
    .items[] | select(.status.conditions[]? | select(.type=="QuotaReserved" and .status=="False"))
    | [ .spec.podSets[] | (.count * (.template.spec.containers[].resources.limits["nvidia.com/gpu"]
        // "0" | tonumber)) ] | add | "pending workload wants \(.) GPUs"'
pending workload wants 8 GPUs
pending workload wants 8 GPUs
pending workload wants 4 GPUs
```

Two 8-GPU workloads queued. The bin-pack change makes exactly two 8-GPU slots placeable, so the
recovery is **fully realisable right now**: `min(16 recovered GPUs, 16 queued GPUs) × $2.35 =
$37.60/hr`. Had the queue been empty, the correct claim would have been *zero this hour*, with the
recovery framed as capacity insurance rather than realised savings. **This step is what separates
a defensible number from a marketing number.**

**Step 6 — is a further defrag worth it?** You want a third 8-GPU slot. Under the bin-packed
inventory `[8,8,5,4,2,2,1,0,…]`, moving n02's 3 GPUs of work onto n03 frees n02 entirely:

```
$ python3 effective_capacity.py     # third report block
defrag: move 3 GPUs of work off a node to free a whole 8-GPU node
  sync checkpoint, 30 min    payback after   7.5 min of the slot being used
  async DCP, 5 min           payback after   2.8 min of the slot being used
```

With a third 8-GPU workload queued and checkpointing victims, the defrag pays for itself in under
eight minutes. Without checkpointing victims it costs the full elapsed runtime of whatever you
kill — §10 case C — and the answer flips to no.

**Step 7 — the write-up.** The paragraph that goes in the 128-GPU capacity document:

> At our 8-GPU flagship job profile, effective capacity is **0 jobs**, not the 4 the free-GPU
> count implies. All 32 free GPUs — **$75/hr, $659k/yr at a $2.35/GPU-hr specialized-neocloud
> on-demand snapshot** — are stranded by node-level fragmentation produced by the default
> `LeastAllocated` spread policy. Switching `NodeResourcesFit` to `MostAllocated` with
> `nvidia.com/gpu` weighted at 10 recovers **2 flagship slots (16 GPUs, $37.60/hr)** at zero
> capital cost, and there are currently two 8-GPU workloads queued to consume them. A subsequent
> single-node defrag would recover a third slot with a 7.5-minute payback, conditional on the
> victims checkpointing. **Recommendation: change the policy first; do not buy nodes to solve a
> placement problem.**

## Practice

**Paper first (20 minutes, no cluster).** Take the inventory
`[3,3,2,2,2,2,2,2,2,2,2,2,2,2,1,1]`. By hand, compute `placeable_real(k)` and `stranded_GPUs(k)`
for k = 1, 2, 4, 8, plus the fragmentation percentage against naive at each size. Confirm k=1 → 0%
and k=8 → 100%. Then do the same for `[8,8,5,4,2,2,1,0,0,0,0,0,0,0,0,0]` and note that the fleet
with *fewer* free GPUs has *more* placeable flagship jobs. **Do this before writing any code** —
the point is that the arithmetic is trivial and the conclusion is counter-intuitive, which is why
nobody computes it.

**Build the calculator (feeds the deliverable).** Write
`practice/kueue-showback/effective_capacity.py`. Commit it. A complete, runnable starting point:

```python
#!/usr/bin/env python3
"""Effective (usable) GPU capacity under indivisible, co-located bin-packing.

Three numbers that are NOT the same:
  raw       total GPUs the fleet owns
  free      GPUs not currently allocated              (what dashboards show)
  placeable GPUs that can still form a whole job      (what you can actually sell)
"""
from __future__ import annotations

import json
import sys
from dataclasses import dataclass


@dataclass
class Capacity:
    job_size: int              # k, GPUs per job
    total_free: int            # Σ free GPUs across the fleet
    naive_placeable: int       # floor(total_free / k)   -- the lie
    real_placeable: int        # Σ floor(free_i / k)     -- the truth
    stranded_gpus: int         # free GPUs that cannot join ANY k-GPU job
    fragmentation_pct: float   # 1 - real/naive, AT THIS JOB SIZE


def effective_capacity(inventory: list[int], k: int) -> Capacity:
    """inventory[i] = free GPUs on node i. k = GPUs one job needs, co-located."""
    if k <= 0:
        raise ValueError("job size k must be >= 1")
    total_free = sum(inventory)
    naive = total_free // k
    real = sum(free // k for free in inventory)
    stranded = total_free - real * k
    frag = 0.0 if naive == 0 else 1.0 - (real / naive)
    return Capacity(k, total_free, naive, real, stranded, round(frag * 100, 1))


def effective_capacity_2d(nodes: list[tuple[int, float]],
                          k_gpu: int, k_cpu: float) -> Capacity:
    """nodes[i] = (free_gpus, free_cpu_cores).

    A node hosts min(free_gpu // k_gpu, free_cpu // k_cpu) copies of the job.
    This is the 'idle GPUs with no matching CPU' failure mode, made countable.
    """
    total_free = sum(g for g, _ in nodes)
    naive = total_free // k_gpu
    real = sum(min(g // k_gpu, int(c // k_cpu)) for g, c in nodes)
    stranded = total_free - real * k_gpu
    frag = 0.0 if naive == 0 else 1.0 - (real / naive)
    return Capacity(k_gpu, total_free, naive, real, stranded, round(frag * 100, 1))


def expected_stranded(inventory: list[int], mix: dict[int, float]) -> float:
    """mix maps job size k -> probability the next job has that size.

    E[stranded GPUs] = Σ_k p_k × (free GPUs unusable by a k-GPU job).
    Fleet-level analogue of FGD's per-node measure F_n(M) = Σ_m p_m F_n(m).
    """
    total = sum(mix.values())
    if abs(total - 1.0) > 1e-9:
        raise ValueError(f"mix probabilities must sum to 1.0, got {total}")
    return sum(p * effective_capacity(inventory, k).stranded_gpus
               for k, p in mix.items())


def stranded_cost(stranded_gpus: float, rate_per_gpu_hour: float) -> dict[str, float]:
    """Rate is a SNAPSHOT. Label the market segment wherever you print it."""
    per_hour = stranded_gpus * rate_per_gpu_hour
    return {"per_hour": per_hour,
            "per_day": per_hour * 24,
            "per_year": per_hour * 8760}


def defrag_payback_hours(migrations: list[tuple[int, float, float]],
                         recovered_gpus: int) -> float:
    """migrations: (gpus_in_job, checkpoint_interval_h, restart_overhead_h).

    Expected work lost per migration = gpus × (T_ckpt/2 + T_restart) GPU-hours:
    a job killed at a uniformly random point in its checkpoint cycle has lost
    T_ckpt/2 of progress on average.

    Returns hours the recovered slot must be OCCUPIED BY REAL WORK to break even.
    """
    cost = sum(g * (t_ckpt / 2 + t_restart) for g, t_ckpt, t_restart in migrations)
    if recovered_gpus <= 0:
        return float("inf")
    return cost / recovered_gpus


def report(inventory: list[int], sizes=(1, 2, 4, 8), rate: float = 2.35) -> None:
    print(f"fleet: {len(inventory)} nodes, {sum(inventory)} free GPUs, "
          f"rate ${rate:.2f}/GPU-hr (SNAPSHOT — label the market segment)")
    print(f"{'k':>3} {'naive':>6} {'real':>5} {'stranded':>9} "
          f"{'frag%':>7} {'$/hr':>9} {'$/yr':>12}")
    for k in sizes:
        c = effective_capacity(inventory, k)
        m = stranded_cost(c.stranded_gpus, rate)
        print(f"{c.job_size:>3} {c.naive_placeable:>6} {c.real_placeable:>5} "
              f"{c.stranded_gpus:>9} {c.fragmentation_pct:>6}% "
              f"{m['per_hour']:>9,.2f} {m['per_year']:>12,.0f}")


def inventory_from_kubectl(doc: dict) -> list[int]:
    """Free GPUs per node from `kubectl get nodes -o json`.

    NOTE: node status carries capacity/allocatable but NOT current usage, so this
    reports ALLOCATABLE. Join with `kubectl get pods -A -o json` and subtract
    per-node committed nvidia.com/gpu limits to get the real free count.
    """
    out = []
    for node in doc.get("items", []):
        alloc = node.get("status", {}).get("allocatable", {})
        out.append(int(alloc.get("nvidia.com/gpu", 0)))
    return [g for g in out if g > 0]


if __name__ == "__main__":
    if "--from-kubectl" in sys.argv:
        fleet = inventory_from_kubectl(json.load(sys.stdin))
    else:
        # 16 nodes x 8 GPUs, 75% allocated, LeastAllocated (spread) placement
        fleet = [3, 3, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 1, 1]

    report(fleet)

    mix = {1: 0.55, 2: 0.25, 4: 0.15, 8: 0.05}
    print(f"\ndemand-weighted E[stranded GPUs] over mix {mix}: "
          f"{expected_stranded(fleet, mix):.2f}")

    print("\nsame fleet, same 75% utilisation, MostAllocated (bin-pack) placement:")
    report([8, 8, 5, 4, 2, 2, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0])

    print("\ndefrag: move 3 GPUs of work off a node to free a whole 8-GPU node")
    for t_ckpt, label in ((0.5, "sync checkpoint, 30 min"), (5 / 60, "async DCP, 5 min")):
        h = defrag_payback_hours([(3, t_ckpt, 5 / 60)], recovered_gpus=8)
        print(f"  {label:<26} payback after {h * 60:5.1f} min of the slot being used")
```

Expected output is the transcript reproduced in the Worked example. Run it before you extend it.

**Then extend it, in this order:**

1. **Feed it real data.** Wire the `kubectl` join from the Worked example step 1 so
   `--from-kubectl` reports *free*, not allocatable. This is where you discover that node status
   does not carry usage and you have to sum pod limits yourself.
2. **Add the topology dimension.** Group nodes by their `Topology` rack label (lesson 6) and
   compute `placeable_real(k)` per rack for a `required`-annotated job. Compare it to the
   node-level number. The rack-level figure is usually worse, and it is the one that matters for
   your comm-bound jobs.
3. **Add the second resource dimension.** Use `effective_capacity_2d` with your fleet's real CPU
   ratio (cores per GPU) and see whether CPU is stranding GPUs on any node.
4. **Compare policies offline.** Simulate the same job mix under spread and bin-pack placement to
   produce the §6 table for *your* node size and job mix. This is the counterfactual that funds
   the config change.
5. **Stretch — real production data.** Download Alibaba's `cluster-trace-gpu-v2023` (~6,200 GPUs,
   the manageable one) or `cluster-trace-gpu-v2026` (up to 155,410 GPUs) and run the calculator
   against real hourly inventory snapshots. Trend `placeable_real(8)` over the trace and note how
   much it moves with the diurnal cycle — that variance is exactly why a point-in-time snapshot is
   not a capacity decision.

**Acceptance (deliverable):** a committed `effective_capacity.py` that (1) takes a fleet inventory
and job size and returns real placeable count, stranded GPUs, and fragmentation percentage; (2)
supports at least one second dimension (CPU or topology level); (3) converts stranded GPUs to a
$/hr and $/yr figure with the rate flagged as a dated snapshot **and labelled by market segment**;
and (4) is exercised in the showback README on your 128-GPU inventory with the naive-versus-real
gap for your dominant job size, the policy counterfactual, and the queue-depth cap on the claim.
A reviewer should be able to run `python3 effective_capacity.py` and get the documented output.

## Common pitfalls

- **Reporting one fleet-wide fragmentation percentage.** Fragmentation is job-size dependent —
  the same inventory is 0% fragmented at k=1 and 100% at k=8. A single number invites the wrong
  fix (buy capacity) when a placement-policy change solves it for free. Mechanism: the metric
  averages over job sizes with implicit equal weight, which is never the real demand
  distribution. Report the per-`k` table, or a demand-weighted scalar with the mix stated.

- **Switching to `MostAllocated` and expecting GPU bin-packing.** The default
  `ScoringStrategy.Resources` is `cpu:1, memory:1`; `nvidia.com/gpu` is not scored unless you list
  it. Symptom: you changed the config, restarted the scheduler, and the inventory looks identical.
  Mechanism: the strategy applies only to the resources named in `resources`, and CPU/memory
  fullness on a GPU fleet is nearly uncorrelated with GPU fullness. Also check the plugin's score
  *weight* — the default is 1, against `TaintToleration: 3` and `PodTopologySpread: 2`.

- **Believing "just move the small jobs together" is free.** Defrag requires evicting and
  rescheduling running work: checkpoint, SIGTERM, grace period, delete, re-queue (as a whole gang
  if it is a gang), restart, reload state, warm up. Cost is `Σ m_j (T_ckpt,j/2 + T_restart,j)`
  GPU-hours, and if the victim does not checkpoint the first term becomes its full elapsed
  runtime. Mechanism: eviction destroys everything since the last durable checkpoint, because
  there is no other resume point.

- **Defragging with no queued demand.** The benefit term is `G × D`, where `D` is hours the
  recovered slot is *actually occupied by work that could not otherwise have run*. With an empty
  queue `D = 0` and the whole operation is pure loss. Always name the specific pending workload
  the defrag is for.

- **Fixing the stock instead of the flow.** Defrag reduces fragmentation once; placement policy
  determines how fast it comes back. If you are defragging weekly under `LeastAllocated`, you are
  paying an eviction tax to work around a config default. Change the policy first, then defrag the
  residue.

- **Treating a bigger cluster as the fix.** More free GPUs lower the *odds* of hitting a
  fragmented wall but do not change the structure — §5's ratio `(1−u)^(G−1)` depends on
  utilisation and node size, not on node count. Double the fleet at the same utilisation and
  placement policy and the *fraction* stranded is identical; you have simply bought a bigger
  version of the same problem.

- **Assuming MIG only helps.** Finer granularity reduces waste for small asks but adds a
  fragmentation axis: profiles come from a fixed menu (seven compute slices on an H100-80GB, with
  legal-placement constraints), and reshaping a GPU's geometry requires draining every workload on
  it. A GPU sliced into `1g.10gb` instances cannot serve a whole-GPU job until it is drained and
  reconfigured — itself a defrag with all of §10's costs.

- **Quoting a point-in-time fragmentation number as a steady-state cost.** A snapshot can land
  mid-defrag (artificially fragmented) or just after a large job exited (artificially clean).
  Sample on a schedule and trend it; a capacity decision needs the distribution, not one draw
  from it.

## Self-check

- **Why is 90% *allocated* capacity not 90% *usable* capacity?** *Answer:* "Allocated" counts GPUs
  handed out; "usable" counts GPUs that can still form a **new indivisible, co-located job**.
  Because a multi-GPU job needs its GPUs on one node (or one topology domain) and cannot be split,
  free GPUs scattered across many partially-full nodes cannot form a large job. Usable capacity is
  `Σ_nodes floor(free_i / k)`, not `total_free / k`. Concretely: 16 nodes × 8 GPUs at 75%
  allocated under spread placement leaves 32 free GPUs and **zero** placeable 8-GPU jobs, against
  a naive count of 4 — 100% fragmentation at that job size, ~$75/hr of paid-for capacity that
  nobody can use. The gap is not waste in the usual sense: every free GPU is real and idle and
  simply the wrong shape.

- **Derive how fast fragmentation grows with utilisation, and say what the exponent depends on.**
  *Answer:* Model each GPU as independently occupied with probability `u` on `G`-GPU nodes. Free
  GPUs per node are `Binomial(G, 1−u)`, so `E[placeable_real(k)] = N·Σ_f C(G,f)(1−u)^f u^(G−f)
  ⌊f/k⌋`. For a whole-node job (`k = G`) only the `f = G` term survives, giving
  `E[placeable_real(G)] = N(1−u)^G` against a naive `N(1−u)`, so the surviving fraction is exactly
  **`(1−u)^(G−1)`**. On 8-GPU nodes that is `(1−u)^7`: **0.78% at 50% utilisation**, 0.16% at 60%,
  0.02% at 70%. The exponent is `G−1`, so **larger nodes are exponentially worse** — 16-GPU nodes
  square an already-tiny number, and a 72-GPU NVL72 domain treated as one bin is the extreme. The
  model assumes no packing logic, so it is a lower bound; a real `MostAllocated` policy recovers a
  large share of it (measured: 0 → 2.19 placeable 8-GPU jobs at 75% utilisation on a 128-GPU
  fleet). The point of the model is that for large `k`, placement policy is not a marginal
  optimisation.

- **What does consolidation cost, and when is it worth doing?** *Answer:* A running job must be
  evicted and rescheduled: SIGTERM, `terminationGracePeriodSeconds` (default 30 s) for a final
  checkpoint, delete, re-queue — as a *whole gang* if it is a gang, which means queueing behind
  everyone else — restart on the new node, reload model and optimizer state, warm up. Expected
  cost is `Σ_j m_j (T_ckpt,j/2 + T_restart,j)` GPU-hours, where `T_ckpt/2` is the expected progress
  lost when the eviction lands at a uniformly random point in the checkpoint cycle. Benefit is
  `G × D`, hours the recovered whole-node slot is occupied by work that could not otherwise have
  run. Worked: moving 3 GPUs of work with 30-minute checkpoints and 5-minute restart costs 1.0
  GPU-hour and pays back after **7.5 minutes** of the recovered 8-GPU slot being used; with
  5-minute async checkpoints it is 0.375 GPU-hours and **2.8 minutes**. If the victim does not
  checkpoint at all, the cost is its full elapsed runtime — 120 GPU-hours for a 40-hour, 3-GPU job —
  and the payback stretches to 15 hours. **Defrag is only economically usable on checkpointing
  workloads, and it is worthless with an empty queue, because `D = 0`.**

- **Name every mechanism that strands a GPU, not just the obvious one.** *Answer:* Four.
  (1) **Structural / node-level** — free GPUs spread across too many nodes to assemble a
  co-located block; cured by bin-packing placement and defrag. (2) **Dimensional** — free GPUs on
  a node with no matching CPU or memory; this is vector bin-packing, and the fix is proportional
  per-queue CPU quota alongside GPU quota plus taints keeping non-GPU work off GPU nodes.
  (3) **Locality** — free GPUs that exist but in different topology domains, so a `required`
  topology constraint cannot combine them (lesson 6). (4) **Deliberate headroom** — teams reserving
  safety margin they are right to want; the fix is not to shrink it but to **backfill it with
  preemptible work** (lesson 8). Alibaba's OSDI '26 trace names the first three explicitly as the
  reasons idle GPUs become unallocatable, plus user headroom as the fourth, and its SpotGPU
  harvesting framework took the allocation ratio from **68% to 93%** by attacking the fourth.

- **How does MIG change the fragmentation surface?** *Answer:* Both ways. Finer granularity means
  small asks — inference, notebooks — stop stranding whole 80 GB devices, so GPU-level waste for
  heterogeneous small demand drops. But MIG adds a new axis: **profile geometry**. An H100-80GB
  exposes seven compute slices and a fixed profile menu (`1g.10gb` ×7, `1g.20gb` ×4, `2g.20gb` ×3,
  `3g.40gb` ×2, `7g.80gb` ×1, plus `1g.10gb+me` ×1), and mixed layouts must partition exactly seven
  slices — the shipped "balanced" config is `1g.10gb×2 + 2g.20gb×1 + 3g.40gb×1` = 2+2+3 = 7.
  Placement of instances is constrained to legal slice positions, and reconfiguring a GPU's
  geometry requires **draining every workload on it**. So a GPU carved into small instances cannot
  serve a whole-GPU job until it is drained and re-provisioned — node-level fragmentation traded
  for intra-GPU profile fragmentation, with a drain cost every time demand shifts. MIG pays off
  only when the small-job demand profile is stable.

- **What does the FGD measure add over `Σ floor(free_i / k)`, and what did it achieve?**
  *Answer:* It generalises from one job size to a demand distribution.
  `F_n(M) = Σ_{m∈M} p_m · F_n(m)`, where `p_m` is the historical popularity of task class `m` and
  `F_n(m)` is the amount of node `n`'s unallocated GPU that a task of class `m` cannot use — so
  the measure is the *expected* unusable free capacity for the next arrival, rather than the
  unusable capacity for one assumed job size. The policy that follows is greedy on the derivative:
  place each task on the node that minimises the *increase* in expected fragmentation. Plain
  bin-packing is the special case where the distribution is a point mass at the largest job size.
  Evaluated on Alibaba production traces over an emulated cluster of more than 6,200 GPUs (Weng et
  al., USENIX ATC '23), it reduced unallocated GPUs by **up to 49%**, putting **290 additional
  GPUs** to work versus Best-Fit, Dot-Product and other packing baselines. Scope caveat worth
  stating: trace-driven emulation, and 49% is the best case in their sweep.

- **A fragmentation number computed from a six-month trace versus a point-in-time snapshot — why
  trust the first?** *Answer:* A snapshot captures one arrival/departure state, which can look
  artificially fragmented (caught mid-defrag, or just after a wave of small jobs landed) or
  artificially clean (right after a large job exited and freed a whole node). Fragmentation is
  driven by the *sequence* of arrivals and completions, which has strong diurnal and weekly
  structure. A six-month trace averages over that churn, so the figure reflects a steady-state
  cost rather than a lucky or unlucky moment — and steady state is what a capacity or
  scheduling-policy decision has to be justified against. Practically: sample `placeable_real(k)`
  on a schedule and report the distribution, not one draw.

## Connections & what's next

Fragmentation math is downstream of everything the module has built so far. Lessons 1–2 established
*why* a job is indivisible (the collective barrier, atomic admission); lesson 6 established *why*
it must also be co-located, and showed Kueue's `LeastFreeCapacity` default for unconstrained
placement as the scheduler pre-empting this lesson's problem with a policy rather than a
calculation. Lesson 5's `binpack` weights and KAI `consolidate` action are the scheduler-side
attacks on the stock and the flow respectively; you can now say which one each is and price both.

The handoff to **[08 — Priority, preemption, and capacity economics](08-priority-preemption-capacity-economics.md)**
is direct and has two threads. First, §12: the fourth kind of stranding — deliberate reserved
headroom — is not a placement problem at all, and the only way to recover it is to run preemptible
work in the gap and reclaim it instantly. Second, §10: every defrag and every reclaim is an
eviction, and the `T_ckpt/2` term that appeared in the payback inequality here is the *same term*
lesson 8 optimises when it derives the cost-minimising checkpoint interval. Fragmentation tells
you how much capacity is recoverable; lesson 8 tells you who gets to reclaim it, what the reclaim
costs, and how much of the fleet you should have committed to buying in the first place.

## References & further reading

**Primary sources — read directly from cloned repositories this session**

Note on method: this environment's egress proxy blocks `kubernetes.io`, `usenix.org`,
`developer.nvidia.com` and several other domains. Rather than cite pages that could not be
reached, the mechanism and default-value claims were verified against upstream *source trees*
cloned during this session; where a canonical URL is given for convenience, its reachability is
stated honestly.

1. **`kubernetes/kubernetes` — `pkg/scheduler/framework/plugins/noderesources/`** —
   https://github.com/kubernetes/kubernetes. Read `least_allocated.go`
   (`leastRequestedScore = (capacity − requested) × MaxNodeScore / capacity`), `most_allocated.go`
   (`mostRequestedScore = requested × MaxNodeScore / capacity`),
   `requested_to_capacity_ratio.go` (the broken-linear shape function), and
   `balanced_allocation.go` (score in `[50,100]` based on whether placing the pod improves the
   node's CPU/memory balance). **Cloned and read directly this session.**

2. **`kubernetes/kubernetes` — `pkg/scheduler/apis/config/v1/defaults.go` and
   `default_plugins.go`.** The two facts §9 turns into a pitfall: `SetDefaults_NodeResourcesFitArgs`
   defaults `ScoringStrategy.Type` to `LeastAllocated` and `Resources` to
   `defaultResourceSpec = [{cpu, weight 1}, {memory, weight 1}]` — **`nvidia.com/gpu` is not
   scored by default** — and the default score weights (`TaintToleration: 3`, `NodeAffinity: 2`,
   `PodTopologySpread: 2`, `InterPodAffinity: 2`, `NodeResourcesFit: 1`,
   `NodeResourcesBalancedAllocation: 1`, `ImageLocality: 1`). **Cloned and read directly this
   session.**

3. **`kubernetes/kubernetes` — `pkg/scheduler/apis/config/types_pluginargs.go`.** The three
   `ScoringStrategyType` values (`LeastAllocated`, `MostAllocated`, `RequestedToCapacityRatio`) and
   the `RequestedToCapacityRatioParam` shape-point structure used in §9's configuration.
   **Cloned and read directly this session.**

4. **`NVIDIA/mig-parted` — `deployments/container/nvidia-mig-manager-example-hopper-blackwell.yaml`** —
   https://github.com/NVIDIA/mig-parted. The authoritative shipped profile menu behind §11's table:
   for H100-80GB / H800-80GB, `1g.10gb` ×7, `1g.10gb+me` ×1, `1g.20gb` ×4, `2g.20gb` ×3,
   `3g.40gb` ×2, `7g.80gb` ×1, and the `all-balanced` layout `1g.10gb×2 + 2g.20gb×1 + 3g.40gb×1`
   summing to exactly seven compute slices. Also carries the H100 NVL, H200, B200, B300, GB200 and
   GB300 menus if you need a different generation. **Cloned and read directly this session.**

**Research**

5. **Weng, Yang, Yu, Wang, Tang, Yang, Zhang — "Beware of Fragmentation: Scheduling GPU-Sharing
   Workloads with Fragmentation Gradient Descent" (USENIX ATC '23)** —
   https://www.usenix.org/conference/atc23/presentation/weng, paper PDF
   https://www.cse.ust.hk/~weiwa/papers/fgd-atc23.pdf. The formal fragmentation measure
   `F_n(M) = Σ_m p_m F_n(m)` reproduced in §7, the greedy gradient-descent policy over it, and the
   evaluation on Alibaba traces across an emulated cluster of more than 6,200 GPUs: unallocated
   GPUs reduced by up to **49%**, **290 additional GPUs** utilised, against Best-Fit, Dot-Product,
   GPU Packing, GPU Clustering and Random-Fit baselines. *(Search-verified this session; both the
   USENIX page and the author's PDF host are blocked by this environment's egress proxy, so the
   measure and headline figures are cited from multiple independent restatements rather than from
   a page fetch.)*

6. **Li Suyi et al. — "Heterogeneity at Hyperscale: Characterization and Scheduling of Large
   Production AI Clusters at Alibaba" (OSDI '26)** —
   https://www.usenix.org/conference/osdi26/presentation/li-suyi. The current hyperscale evidence
   base: six-month trace, up to **155,410 GPUs across 37,707 servers**, **81 departments**; the
   finding that high demand does not yield high effective utilisation because idle GPUs become
   unallocatable (stranded across nodes / no matching CPU / network-locality violations / user
   headroom); a defragmentation algorithm cutting nodes with slack resources by **20.2%**; and
   **SpotGPU** raising the GPU allocation ratio from **68% to 93%**. *(Search-verified;
   usenix.org unreachable from this environment.)*

**Data you can run the calculator against**

7. **Alibaba clusterdata — `cluster-trace-gpu-v2026`** —
   https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2026. Four released
   tables: `asi_opensource_pod_hourly` (pod workload, request, utilisation, priority, state; days
   0–184 hourly), `asi_opensource_server_hourly` (server inventory + anonymised ASW/rack
   topology), `asi_opensource_network_hourly` (node-level rx/tx, days 109–115), and
   `asi_opensource_job_execution_summary`. Peak hourly coverage 155,410 GPUs / 37,707 servers.
   **Fetched directly this session.**

8. **Alibaba clusterdata — `cluster-trace-gpu-v2023`** —
   https://github.com/alibaba/clusterdata/blob/master/cluster-trace-gpu-v2023/README.md. The
   smaller (~6,200-GPU) production trace behind FGD's evaluation — the sensible first dataset for
   validating a calculator. **Fetched directly this session.**

**Engineering accounts**

9. **NVIDIA — "Practical Tips for Preventing GPU Fragmentation for Volcano Scheduler"** —
   https://developer.nvidia.com/blog/practical-tips-for-preventing-gpu-fragmentation-for-volcano-scheduler/.
   Bin-packing integrated into Volcano on a DGX Cloud-provisioned Kubernetes cluster, reaching a
   **90% GPU occupancy rate against an 80% contractual target**, with no new hardware — the
   mechanism of §9 applied in production. *(Search-verified; fetch blocked by egress. **This entry
   corrects the previous version of this lesson**, which attributed a "double digits to under 1%
   node fragmentation" figure to this source; that figure could not be confirmed and has been
   replaced with the occupancy numbers that could.)*

10. **Volcano — `binpack` plugin** — https://volcano.sh/en/docs/plugins/ and
    `pkg/scheduler/plugins/binpack/binpack.go` in `volcano-sh/volcano`. The scoring function
    (`score_r = (requested_r + used_r)/allocatable_r × weight_r`, normalised over the weight sum
    and scaled by `MaxNodeScore × binpack.weight`) and the per-resource weight arguments that let
    you make GPU fullness dominate. Lesson 5 covers this in depth; read it here as the Volcano
    column of §9's table. **`volcano-sh/volcano` cloned and read directly this session.**

**Deeper dives**

11. **NVIDIA KAI Scheduler — the `consolidate` action** —
    https://github.com/NVIDIA/KAI-Scheduler. Defragmentation as a first-class scheduling action
    that runs before any disruptive action and evicts a pod only once it has already found it a new
    home — the production-grade version of §10's "migrate the cheap-to-move jobs", and the only one
    of the three schedulers that attacks the fragmentation *stock* automatically. Lesson 5 §3 has
    the full action pipeline. **Repository cloned and read directly this session.**
