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
sources: 8
---

# 06.7 · Fragmentation & effective capacity

> **Concept.** Indivisible bin-packing means an allocated fleet is not a usable fleet — measure the gap and you find hidden GPUs.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

Lesson 06 taught you how to place a gang *correctly* — on GPUs that share a fast interconnect, so the all-reduce doesn't crawl. This lesson asks a different question about the same fleet: even with perfect topology-aware placement, how much of your fleet can you actually *use*? The gap between "allocated" and "usable" is fragmentation, and it is invisible on every dashboard that counts busy GPUs instead of *placeable* jobs. Once you can measure it, the next lesson (08) asks what you do with the capacity you recover — price it, protect it with priority tiers, and decide how much of it to pre-commit versus buy on demand. Fragmentation math is the input; commitment strategy is the output.

## Why this matters

You manage 40+ clusters. Your dashboards say the fleet is 90% allocated, so "we're basically full, buy more." That sentence, repeated across a quarter, is how a GPU org burns a seven-figure capex line on capacity it already owns.

Here is the FinOps edge that a pure SWE will miss: **allocated capacity and usable capacity are different numbers, and the gap between them is invisible.** It does not show up as an idle GPU on a Grafana panel — every node looks busy — yet an 8-GPU training job cannot start because the free GPUs are smeared two here, five there, one over there. Nobody is "wasting" a GPU; the fleet is *fragmented*. That is the single best "I found $X of hidden capacity" story you can walk into a CoreWeave/NVIDIA interview with, because it is quantifiable, it is real money, and most engineers cannot even name it — Alibaba's own OSDI'26 characterization of its production AI fleet treats "stranded capacity" as a first-class, named phenomenon worth a dedicated paper, not a rounding error.

The durable claim of this lesson: **you can only place indivisible, co-located jobs, so the right denominator for "how full are we?" is not free GPUs — it is placeable jobs.** Learn to compute that number and you can defend "don't buy" with arithmetic.

## What's new here (calibration)

Everything before this lesson (gang scheduling, Kueue, topology) told you *how* the scheduler places work. This lesson is about *what's left over* and how to price it. New ideas:

- **Effective (usable) capacity** vs **allocated capacity** vs **raw free capacity.** Three different numbers; only the first pays your bills.
- **Node-level vs GPU-level fragmentation.** A node with 3 free GPUs fragments differently than a GPU sliced into MIG partitions.
- **Consolidation / defrag has a cost** — it is not free `kubectl` magic; a running job has to *die and move*. Fragmentation is a stock; defrag is a flow with a price.
- **MIG changes the fragmentation surface**: finer granularity reduces waste for small asks but introduces a *new* axis of fragmentation (profile geometry).
- **Fragmentation is now a hyperscale-measured phenomenon, not a toy problem** — Alibaba's 2026 trace gives you a real, six-month, 155,410-GPU data point to reason from instead of a single emulated cluster.

## Core concepts

### The bin-packing framing

A multi-GPU training job needs **N GPUs on as few nodes as possible, co-located** (NVLink/NVSwitch bandwidth within a node dwarfs inter-node; recall Lesson 06). For an 8-GPU job on 8-GPU nodes, "co-located" means *one whole free node*. The job is **indivisible**: you cannot run 6 GPUs of it here and 2 there and call it done. This is the classic **bin-packing** constraint, and bin-packing is why the naive capacity formula lies.

Naive (wrong) capacity for job size `k`:

```
placeable_naive = floor(total_free_GPUs / k)
```

This treats GPUs as a fungible liquid. They are not. The honest formula respects node boundaries:

```
placeable_real(k) = sum over nodes of floor(free_gpus_on_node / k)
```

The difference between these two is **fragmentation loss**.

### Worked intuition: the same fleet, two answers

Twelve 8-GPU nodes. Free-GPU counts per node:

```
[8, 8, 2, 2, 5, 5, 5, 1, 1, 4, 4, 3]   -> total_free = 48
```

Naive answer for 8-GPU jobs: `floor(48 / 8) = 6`. Sounds like six jobs fit.

Real answer: only the two nodes with 8 free can host an 8-GPU co-located job.

```
floor(8/8)+floor(8/8)+floor(2/8)+...+floor(3/8) = 1+1+0+...+0 = 2
```

**Two jobs, not six.** You are carrying 48 free GPUs and can start exactly 2 of your flagship jobs. Fragmentation loss = (6 − 2)/6 = **67% of the naive capacity is unusable at this job size.** If those 48 GPUs are H100s billed internally at, say, ~$3/GPU-hr (a *neocloud-segment* snapshot — see 06.8 for why the market segment matters), the 32 stranded GPUs that *look* available but cannot form a job represent ~$96/hr, ~$70k/month of capacity your showback cannot allocate to anyone. That is the story.

### Fragmentation is job-size-dependent

There is no single "fragmentation %." The same fleet is 67% fragmented for 8-GPU asks and **0% fragmented for 1-GPU asks** (every free GPU is placeable when k=1). This is why you must report effective capacity *per job profile*, not as one fleet number. A fleet perfectly tuned for a fleet of 1-GPU notebooks is a disaster for 8-GPU pretraining, and vice versa. **A single fleet-wide fragmentation percentage reported to leadership is actively misleading** — it invites the wrong fix (buying capacity) when the real fix (a placement-policy change, or a job-size-aware queue split) costs nothing.

### Node-level vs GPU-level fragmentation

- **Node-level fragmentation**: free GPUs are spread across too many nodes to assemble a large co-located job. This is what the example above shows. Cured by **bin-packing placement** (pack new jobs onto the most-full eligible node, not the emptiest) and by **consolidation** (migrate to empty nodes).
- **GPU-level fragmentation**: a single GPU is *partially* consumed — MIG slices, or MPS/time-slicing — so a whole-GPU ask cannot use it even though capacity is free *inside* it. Cured by aligning slice profiles to demand, or by draining and reconstituting the GPU as whole.

The **NVIDIA data point to memorize**: their Volcano bin-packing plugin work reports substantial node-fragmentation reduction — from measurable double digits down to under 1% — simply by changing placement policy from spread to bin-pack, and reports GPU occupancy climbing toward ~90% on the resulting DGX Cloud-provisioned cluster — no new hardware. That is the shape of every good fragmentation win: a *scheduling* change that unlocks *capital*.

### Bin-pack vs spread: the policy lever

The default scheduler tends to **spread** (balance load, good for HA of stateless services). For scarce indivisible GPU jobs you want the opposite: **bin-pack / consolidate**, so free capacity coalesces into whole-node holes big enough for the next big job. Volcano's `binpack` plugin and Kueue's placement, scoring nodes by how *full* they'd become, are the mechanisms. The FinOps read: spread optimizes availability; bin-pack optimizes *utilization of a fixed asset*. On owned GPUs, the fixed asset is the expensive thing. This is a deliberate **reversal of default cloud-native instinct** — most engineers' reflex is "spread is always safer" — which is exactly why stating it correctly, with the reasoning, is a strong interview differentiator.

### The cost of defrag (consolidation)

Fragmentation looks free to fix — "just move the small jobs together." It is not. To consolidate, a **running job must be evicted from its node and rescheduled**. That means:

- **Checkpoint + kill + reschedule + restart + warm up.** Lost wall-clock = time since last checkpoint + queue wait + container/model reload. For a large training job that can be 10–30 min of GPU-time thrown away *per migration*.
- **Gang re-admission**: a multi-pod gang can't partially move; the whole gang re-queues, and may wait behind other work (deadlock risk — Lesson 01).
- **No checkpoint = no safe defrag.** If the victim job doesn't checkpoint, you either can't move it or you destroy hours of work. This is the hinge that ties directly into 06.8: *preemption and defrag are only economically usable on checkpointing workloads.*

So defrag is a **flow cost** you pay to reduce a **stock** of fragmentation. The FinOps decision is: does the recovered placeable capacity (over the horizon it stays recovered) exceed the GPU-hours burned migrating? Consolidate the *cheap-to-move* (best-effort, checkpointing) jobs; leave prod pinned.

NVIDIA's **KAI Scheduler** (Lesson 05) makes this an explicit, automated feature called **consolidation**: it actively relocates running (non-pinned) pods to free up contiguous node-level capacity for pending multi-GPU jobs, rather than waiting for jobs to complete naturally. That is defrag-as-policy rather than defrag-as-manual-runbook — worth naming as the production-grade version of the "migrate the cheap-to-move jobs" idea above.

### How MIG changes the surface

MIG (Multi-Instance GPU — you did this in 04) partitions one physical GPU into up to 7 isolated instances. Effect on fragmentation:

- **Finer granularity → less waste for small asks.** A 1g.10gb inference job no longer strands a whole 80GB H100. GPU-level packing improves; the "one big ask parked on a tiny job" problem shrinks.
- **…but MIG introduces a new fragmentation axis: profile geometry.** MIG profiles are not free-form. An A100/H100 exposes a fixed menu (e.g. `1g.10gb, 2g.20gb, 3g.40gb, 4g.40gb, 7g.80gb`) and the on-GPU layout is constrained — you cannot always mix arbitrary profiles, and reconfiguring a GPU's MIG geometry typically requires **draining every workload on it**. So you trade node-level fragmentation for **intra-GPU profile fragmentation**: a GPU carved into small slices cannot serve a whole-GPU job until it is drained and re-provisioned — itself a defrag with a cost.
- Net: MIG is a fragmentation *reducer for heterogeneous small demand* and a fragmentation *creator when demand shifts back toward whole-GPU jobs.* Match MIG geometry to a stable demand profile, or you'll be paying drain costs to reshape GPUs constantly.

## Perspectives

**Developer.** A researcher whose job "should fit" — the naive free-GPU math checks out — but won't admit, experiences fragmentation as an unexplained scheduler failure. From inside the training loop there is no signal that distinguishes "the cluster is genuinely full" from "the cluster has plenty of free GPUs that happen to be unusable for my job shape." A platform team that can hand back the `effective_capacity.py`-style breakdown converts a confusing "the scheduler is broken" ticket into a legible "here's why, here's the fix" conversation — and that legibility is itself a platform-engineering deliverable.

**Operator.** Fragmentation is a *policy* choice as much as an emergent property. Bin-pack vs spread is a scheduler config decision the operator controls; spread optimizes HA of stateless services, bin-pack optimizes utilization of a fixed asset. The operator's job is to know which regime a given queue is in (prod-serving wants spread for blast-radius reasons; training wants bin-pack for capacity reasons) and to configure placement policy *per queue*, not fleet-wide.

**Research/theory.** The FGD (Fragmentation Gradient Descent) paper is the formal treatment of this lesson's intuition. Its actual contribution is precise: it defines a fragmentation *measure* (not just an eyeballed intuition) and a greedy placement heuristic that provably reduces it, evaluated against real Alibaba production traces (~6,200 GPUs, 2023). Naming this precisely — "a measure plus a heuristic that reduces it, trace-evaluated" — is a stronger interview answer than "there's a paper about GPU fragmentation."

**Economics.** The "$X of hidden capacity" framing is the strongest FinOps story in the whole module. It's strengthened enormously by having a hyperscale anchor: Alibaba's OSDI'26 paper characterizes stranded capacity across a **155,410-GPU, six-month production trace** — this is not an academic toy, it's what one of the largest cloud operators in the world spends research effort measuring, because the dollar value of the gap is large enough to justify a paper.

## Real-world use cases

- **Alibaba — "Heterogeneity at Hyperscale: Characterization and Scheduling of Large Production AI Clusters at Alibaba" (OSDI '26)** — https://www.usenix.org/conference/osdi26/presentation/li-suyi. What it shows: an anonymized **six-month production trace covering up to 155,410 GPUs across 37,707 servers**, explicitly characterizing stranded capacity, locality constraints, and heterogeneous-GPU scheduling at a scale that dwarfs the 2023 FGD paper's ~6,200-GPU emulated evaluation — the strongest, most current hyperscale anchor available for this lesson's thesis.
- **Alibaba — `cluster-trace-gpu-v2026` public trace release** — https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2026 (fetched directly). What it shows: Alibaba publishes the *actual* GPU cluster trace data behind the OSDI'26 paper — pod-level workload mix, priority classes, resource requests/utilization, GPU/server inventory, and topology — publicly and downloadably. A motivated learner can pull real production data and run this lesson's `effective_capacity.py` calculator against it directly instead of a toy fleet.
- **NVIDIA — "Practical Tips for Preventing GPU Fragmentation for Volcano Scheduler"** — https://developer.nvidia.com/blog/practical-tips-for-preventing-gpu-fragmentation-for-volcano-scheduler/ (search-verified; blocked by this session's egress proxy, canonical URL confirmed). What it shows: a single scheduling-policy change (spread → bin-pack via the Volcano `binpack` plugin) drove node fragmentation from measurable double digits down to under 1% and GPU occupancy up toward ~90% on a DGX Cloud-provisioned Kubernetes cluster — no new hardware purchased.
- **Alibaba — `cluster-trace-gpu-v2023` public trace release** — https://github.com/alibaba/clusterdata/blob/master/cluster-trace-gpu-v2023/README.md (fetched directly). What it shows: the earlier, smaller (~6,200-GPU, ~1,200-machine) real trace that the FGD paper's own evaluation is built from — confirms the academic paper's evidence base is real production data, not synthetic, and gives you a second, smaller dataset to sanity-check the calculator against before tackling the 2026 trace.

## Worked example

Fleet (12 nodes, 8-GPU each), free-GPU inventory and a job mix arriving in order:

```
inventory (free GPUs/node): [8, 8, 2, 2, 5, 5, 5, 1, 1, 4, 4, 3]  total_free = 48
job queue (GPU asks):        [8, 8, 4, 4, 2, 2, 1]
```

**Naive capacity view** (management's slide): `48 free / 40-cluster avg... we're 90% allocated, near full.` For 8-GPU jobs, naive says `floor(48/8)=6` fit.

**Real placement (bin-pack, largest-first):**

1. `8` → node0 (8 free → 0). ✅
2. `8` → node1 (8 free → 0). ✅
3. `4` → node9 (4 free → 0). ✅ (pick an exact-fit node to avoid stranding)
4. `4` → node10 (4 free → 0). ✅
5. `2` → node2 (2 free → 0). ✅
6. `2` → node3 (2 free → 0). ✅
7. `1` → node7 (1 free → 0). ✅

All 7 mixed jobs placed — because the *mix* matches the holes. But the two 8-GPU asks consumed the only two whole-node holes. **A third 8-GPU job cannot start**, despite 48−29 = 19 GPUs still free (on nodes with 5,5,5,1,3 free). Those 19 GPUs are real, paid-for, and **unplaceable for the flagship job size.**

The headline for your write-up: *"At our current 8-GPU job profile, effective capacity is 2 jobs, not the 6 the free-GPU count implies. 19 free GPUs (~$57/hr internal at a ~$3/GPU-hr snapshot) are stranded by node-level fragmentation. A bin-pack placement policy plus consolidating two of the three 5-free nodes recovers one whole node = one more flagship job, at a defrag cost of ~1 checkpoint-restart."*

**Extending it — is the defrag worth it?** Consolidating the two 5-free nodes (moving their occupants onto one node, freeing the other) requires migrating the running jobs on those nodes: assume 1 checkpoint-restart cycle per job moved, ≈20 minutes of lost wall-clock on the 2 consolidated jobs' 3 occupied GPUs each (6 GPU-slots total) — call it 2 GPU-hours of defrag cost. The payoff is one recovered whole-node (8-GPU) slot. At the ~$3/GPU-hr internal snapshot used above, the recovered slot is worth ~$24/hr the moment a flagship job occupies it — so the defrag cost (2 GPU-hours ≈ $6 at the same rate) pays for itself in **15 minutes** of the recovered slot running a real job. Below that horizon (a slot that sits idle for under 15 minutes before someone else reclaims it) the defrag is a net loss; above it, it's a clear win. This is the actual go/no-go arithmetic behind "should we consolidate," not a hand-wave.

## Practice

**Paper first (20 min).** Take the inventory `[8,8,2,2,5,5,5,1,1,4,4,3]`. By hand compute `placeable_real(k)` for k = 1, 2, 4, 8 and the fragmentation % vs naive for each. Confirm: k=1 → 0% frag; k=8 → 67% frag. Notice the number moves with job size — that is the whole point.

**Build the calculator (feeds the deliverable).** Write `practice/kueue-showback/effective_capacity.py`. Commit it.

```python
#!/usr/bin/env python3
"""Effective (usable) GPU capacity under indivisible, co-located bin-packing.

Given a fleet inventory (free GPUs per node) and a job size k, returns how many
k-GPU co-located jobs actually fit, versus the naive free/k count, and the
fragmentation loss between them.
"""
from __future__ import annotations
from dataclasses import dataclass


@dataclass
class Capacity:
    job_size: int
    total_free: int
    naive_placeable: int      # floor(total_free / k) -- the lie
    real_placeable: int       # sum floor(free_on_node / k) -- the truth
    stranded_gpus: int        # free GPUs that cannot join any k-job
    fragmentation_pct: float  # 1 - real/naive, at this job size


def effective_capacity(inventory: list[int], k: int) -> Capacity:
    if k <= 0:
        raise ValueError("job size k must be >= 1")
    total_free = sum(inventory)
    naive = total_free // k
    real = sum(free // k for free in inventory)
    used_by_real = real * k
    stranded = total_free - used_by_real
    frag = 0.0 if naive == 0 else 1.0 - (real / naive)
    return Capacity(k, total_free, naive, real, stranded, round(frag * 100, 1))


def report(inventory: list[int], sizes=(1, 2, 4, 8)) -> None:
    print(f"fleet: {len(inventory)} nodes, {sum(inventory)} free GPUs")
    print(f"{'k':>3} {'naive':>6} {'real':>5} {'stranded':>9} {'frag%':>7}")
    for k in sizes:
        c = effective_capacity(inventory, k)
        print(f"{c.job_size:>3} {c.naive_placeable:>6} {c.real_placeable:>5} "
              f"{c.stranded_gpus:>9} {c.fragmentation_pct:>6}%")


if __name__ == "__main__":
    fleet = [8, 8, 2, 2, 5, 5, 5, 1, 1, 4, 4, 3]
    report(fleet)
    # extend: pipe `kubectl get nodes -o json` free-GPU counts in as `fleet`.
```

Expected output:

```
fleet: 12 nodes, 48 free GPUs
  k  naive  real  stranded   frag%
  1     48    48         0    0.0%
  2     24    21         6   12.5%
  4     12     9        12   25.0%
  8      6     2        32   66.7%
```

**Acceptance (deliverable):** committed `effective_capacity.py` that (1) reads a fleet inventory + job size and returns real placeable count, stranded GPUs, and fragmentation %, and (2) a short worked example in the showback README showing the naive-vs-real gap for your fleet's dominant job size, with the stranded GPUs converted to a $/hr snapshot (flag the rate as a snapshot, and label which market segment it comes from — see 06.8). Stretch: feed real `kubectl get nodes -o json` GPU allocatable/used into the calculator, or — going further still — download Alibaba's public `cluster-trace-gpu-v2023` or `cluster-trace-gpu-v2026` trace and run the calculator against real production inventory snapshots instead of a toy fleet.

## Common pitfalls

- **Reporting a single fleet-wide fragmentation percentage to leadership.** Fragmentation is job-size-dependent; a single number invites the wrong fix (buying capacity) when a placement-policy change or job-size-aware queueing would solve it for free. Always report per dominant-job-size profile, the way the worked example's k=1/2/4/8 table does.
- **Believing "just move the small jobs together" is free.** Defrag requires evicting and rescheduling a running job — checkpoint, kill, re-queue, restart, warm up. If the victim doesn't checkpoint, you either can't move it safely or you destroy hours of progress. Defrag is a priced flow, not free `kubectl` magic.
- **Treating a bigger cluster as the fix for fragmentation-induced blocking.** More free GPUs reduce the *odds* of hitting a fragmented wall, but they don't remove the structural gap between allocated and usable capacity — the ratio can stay just as bad at 2× the fleet size if placement policy doesn't change.
- **Assuming MIG only helps.** Finer granularity reduces waste for small asks but adds a new fragmentation axis (profile geometry): a GPU carved into small slices can't serve a whole-GPU job until it's drained and re-provisioned. MIG needs to match a stable demand profile, or you pay drain costs constantly.

## Self-check

- Why is 90% *allocated* capacity not 90% *usable* capacity? **Answer:** "Allocated" counts GPUs handed out; "usable" counts GPUs you can still assemble into a *new indivisible, co-located job*. Because a multi-GPU job needs its GPUs on as few nodes as possible and cannot be split, free GPUs scattered across many partially-full nodes cannot form a large job. So 10% free smeared as 1–2 GPUs per node can yield **zero** placeable 8-GPU jobs — the free capacity exists but is unusable at the job size that matters. Usable capacity is `sum(floor(free_on_node / k))`, not `total_free / k`.
- What does consolidation/defrag cost you — what has to happen to a running job? **Answer:** A running job must be **evicted and rescheduled**: checkpoint (if it can), get killed, re-queue (possibly waiting behind other work / as a whole gang), restart on the new node, reload the container and model, and warm up — throwing away all GPU-time since its last checkpoint. It is a real GPU-hour cost, and if the job doesn't checkpoint you either can't move it safely or you destroy hours of progress. Defrag is a priced flow, not free magic.
- How does MIG change the fragmentation surface? **Answer:** MIG makes granularity finer, so small asks (inference, notebooks) stop stranding whole 80GB GPUs — less GPU-level waste for heterogeneous small demand. *But* it adds a new fragmentation axis: MIG profiles come from a fixed menu with layout constraints, and reshaping a GPU's geometry requires draining every workload on it. A GPU sliced into small partitions cannot serve a whole-GPU job until it is drained and re-provisioned — so MIG trades node-level fragmentation for intra-GPU profile fragmentation, and shifting demand back toward whole-GPU jobs turns MIG into a fragmentation *source* with its own drain cost.
- Alibaba's OSDI'26 trace covers 155,410 GPUs over six months and explicitly models stranded capacity. Why is a fragmentation number computed from a six-month trace more trustworthy than one computed from a single point-in-time snapshot? **Answer:** A point-in-time snapshot captures one arrival/departure state of the queue, which can look artificially fragmented (mid-defrag) or artificially clean (right after a big consolidation). A six-month trace averages over the natural churn of job arrivals, completions, and diurnal/weekly demand cycles, so the fragmentation figure reflects a *steady-state* cost rather than a lucky or unlucky moment — which is what you actually need to justify a capacity or scheduling-policy decision.
- Using the module's `effective_capacity.py` formula, if a defrag operation costs 20 minutes of GPU-time per job migrated and recovers one 8-GPU flagship slot, at what internal $/GPU-hr rate does the defrag pay for itself within one hour of the recovered slot being used? **Answer:** Set defrag cost equal to the value of one recovered-slot GPU-hour. If 2 jobs are migrated at 20 min each on, say, 3 GPUs apiece (the worked example's numbers), that's 2 GPU-hours of defrag cost; one 8-GPU-hour of recovered slot needs to be worth at least that, i.e. `2 × rate ≤ 8 × rate` is true at *any* positive rate — the defrag always pays for itself within one hour of use at this migration cost, because the recovered slot (8 GPU-hours) is worth 4× the defrag cost (2 GPU-hours) regardless of the dollar rate. The rate only matters for *how much* profit, not *whether* it's profitable — a useful distinction to make explicit in an interview.

## Connections & what's next

Fragmentation math is downstream of gang scheduling (01–02, which made co-located admission atomic) and topology-aware placement (06, which made co-location interconnect-aware) — both determine *what a job needs*; this lesson determines *whether the fleet can supply it*. It's also the direct input to Kueue's Topology Aware Scheduling placement algorithm change in v0.15 (BestFit → LeastFreeCapacity, covered in Lesson 06): that change exists specifically to bias placement toward *preserving* future large-gang placeability, i.e. to fight the exact fragmentation this lesson quantifies, before it happens rather than after. The next lesson, **08 — Priority, preemption, and capacity economics**, takes the "$X of hidden capacity" number this lesson produces and asks the natural follow-up questions: who gets to reclaim it, at what priority, and how does the answer change what you commit to buy versus rent.

## References & further reading

**Primary sources**
- Alibaba — "Heterogeneity at Hyperscale: Characterization and Scheduling of Large Production AI Clusters at Alibaba" (OSDI '26) — https://www.usenix.org/conference/osdi26/presentation/li-suyi — read for the current, hyperscale (155,410-GPU, six-month) production evidence on stranded capacity that anchors this lesson.
- Weng et al. — "Beware of Fragmentation: Scheduling GPU-Sharing Workloads with Fragmentation Gradient Descent" (USENIX ATC '23), paper PDF https://www.cse.ust.hk/~weiwa/papers/fgd-atc23.pdf and conference page https://www.usenix.org/conference/atc23/presentation/weng — read for the formal fragmentation measure and the greedy heuristic that reduces it, trace-evaluated.
- Alibaba clusterdata — `cluster-trace-gpu-v2026` — https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2026 — read to pull the real production trace behind the OSDI'26 paper and run it through your own effective-capacity calculator.

**Real-world engineering blogs**
- NVIDIA Developer Blog — "Practical Tips for Preventing GPU Fragmentation for Volcano Scheduler" — https://developer.nvidia.com/blog/practical-tips-for-preventing-gpu-fragmentation-for-volcano-scheduler/ — what it shows: bin-pack placement drove node fragmentation from double digits to under 1%, no new hardware.
- Alibaba clusterdata — `cluster-trace-gpu-v2023` — https://github.com/alibaba/clusterdata/blob/master/cluster-trace-gpu-v2023/README.md — what it shows: the real, smaller (~6,200-GPU) production trace the FGD paper's own evaluation is built from, and a good first dataset to validate your calculator against.

**Deeper dives**
- Volcano — Plugins docs (binpack) — https://volcano.sh/en/docs/plugins/ — the scheduler-side mechanics of the bin-pack scoring function referenced throughout this lesson.
- NVIDIA — KAI Scheduler — https://github.com/NVIDIA/KAI-Scheduler — see its **consolidation** feature, an automated, production-grade version of the "migrate the cheap-to-move jobs to defrag" idea in this lesson (bridges to Lesson 05).
