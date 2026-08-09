---
lesson: "06.8"
title: "Priority, preemption, and capacity economics"
module: "06"
concept: "Priority, preemption, and capacity economics"
status: not-started
est_time: "8h"
artifacts: []
---

# 06.8 · Priority, preemption, and capacity economics

> **Concept.** Priority tiers with *survivable* (checkpoint-aware) preemption, and a reserved / on-demand / spot commitment ladder for a fleet you cannot autoscale.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + showback](../practice/kueue-showback/README.md)

## Why this matters

This is the lesson where your FinOps certification lets you out-answer a pure SWE. Preemption is how you sweat a fixed fleet toward ~100% utilisation without starving production — but only if the preempted work *survives*, which makes it a joint scheduler + training-loop decision, not a scheduler knob. And capacity planning for accelerators breaks every reflex a cloud engineer has: you cannot autoscale your way out of scarcity, so the lever is *commitment strategy*, not elasticity. "Design the reserved/on-demand/spot mix and defend the break-even" is a standard senior interview question, and it's yours to dominate.

## What's new here

You know spot instances, priority classes, and reserved-instance math from AWS. What's new is the accelerator-specific inversion of all three:

- **Priority + preemption** here means *Kueue* reclaim of *borrowed* quota (Lesson 04), plus pod `PriorityClass` — and the hard constraint that a preempted training job is worthless unless it checkpointed.
- **Spot** for GPUs is a different animal: preemptible capacity that is often *unavailable at any price* during a shortage, not merely pricier.
- **Reserved-instance math** inverts: in 2026, committed GPU prices *rose* while on-demand *fell* — the opposite of the CPU world you're used to.

## Core notes

### Priority tiers

A shared GPU fleet runs at least three tiers, expressed as pod `PriorityClass` values and mirrored in Kueue queue/workload priority:

| Tier | Example | Preemptible? | Guarantee |
|------|---------|--------------|-----------|
| **prod** | inference serving, on-call | no | protected floor; preempts others |
| **research** | interactive training, experiments | reclaimable when over quota | nominal quota, can borrow |
| **best-effort** | batch sweeps, backfill | freely | runs only on idle capacity |

The design goal: prod never waits; research gets a guaranteed floor plus borrowable headroom; best-effort mops up the remaining idle so the fleet approaches 100% *useful* utilisation. Kueue expresses this with `ClusterQueue` priorities + cohort borrowing (Lesson 04); the best-effort queue has a low `lendingLimit` claim and its workloads carry a low `WorkloadPriorityClass`, so they are the first evicted when an owner reclaims.

### Survivable preemption is a training-loop property

Preemption is only *economically* usable if the preempted workload loses no more than the time since its last checkpoint. A job that restarts from scratch on preemption turns a scheduling win into a compute-hours loss — you preempted to *save* money and spent more.

So preemption design is a contract with the workload:
- The training loop must **checkpoint** (model + optimizer + step) at an interval `T_ckpt`.
- On preemption, the job is re-queued and **resumes from the last checkpoint**, losing at most `T_ckpt` of work plus one checkpoint-restore round-trip.
- The scheduler should give a **grace period** (`terminationGracePeriodSeconds`, or Kueue's preemption with a delay) long enough for a final checkpoint-on-SIGTERM where feasible.

The economic rule: **expected wasted work per preemption ≈ T_ckpt/2** (uniform arrival), so checkpoint frequently enough that `T_ckpt/2 × preemption_rate` is small relative to the cheaper capacity preemption buys. This ties back to Module 08 (checkpoint frequency vs failure rate) — the same math governs preemption and hardware failure.

> **Interview one-liner:** *"Preemption without checkpointing isn't a cost optimisation, it's a compute-hours bonfire."*

### Why you cannot autoscale a GPU cluster

CPU autoscaling assumes elastic supply: the cloud always has more `m5.large`s. GPUs break that assumption on three axes:

1. **Supply is finite and booked forward.** In a shortage, on-demand H100 capacity is simply *out of stock* in your region/zone for hours or days — Cluster Autoscaler asks for nodes that never come. Forward capacity is reserved months ahead.
2. **Cold-start is minutes, not seconds.** A GPU node must pull a multi-GB driver + container image and often 10s–100s of GB of model weights; scale-up latency is minutes (see Module 07 cold-start), useless for reactive autoscaling of spiky demand.
3. **The unit is lumpy and expensive.** You scale in 8-GPU nodes at ~$15–25/hr each; a mis-sized scale-up is a large, immediate bill, so you plan capacity rather than react to it.

The consequence: for GPUs you **commit capacity ahead of demand** and *schedule* within it (this whole module) rather than autoscaling to meet demand.

### The commitment ladder

Because you buy capacity ahead, you ladder commitments against uncertain demand:

- **Reserved / committed (1–3 yr, or capacity blocks):** cheapest per hour, but you pay whether or not you use it. Sized to your **P50 baseline** — the demand you're confident exists.
- **On-demand:** most expensive per hour, no commitment. Absorbs the **burst above baseline** — and, crucially, is *not guaranteed available* in a shortage.
- **Spot / preemptible:** cheapest when available, evictable. Runs **best-effort/checkpointable** work only, and only opportunistically.

Blended cost of a laddered fleet:

```
blended $/GPU-hr = (reserved_hrs × reserved_rate
                  + ondemand_hrs × ondemand_rate
                  + spot_hrs × spot_rate) / total_hrs
```

**Break-even for a commitment:** a 1-yr reserve beats on-demand once your *sustained* utilisation of that capacity exceeds `reserved_rate / ondemand_rate`. If a 1-yr commit is ~$2.35/GPU-hr and on-demand is ~$3.90, break-even ≈ `2.35/3.90 ≈ 60%` sustained utilisation — below that, on-demand is cheaper; above it, commit. The FinOps job is to *know your utilisation floor* (which the showback report from this module's capstone measures) and commit exactly up to it.

### The 2026 price inversion (know this cold)

Unlike CPUs, GPU pricing moved in *opposite directions by billing tier* through 2024→2026: **on-demand H100 fell ~58%** as supply caught up, while **1-yr committed rates *rose* ~40%** (~$1.70 → ~$2.35/GPU-hr, Oct-2025 → Mar-2026) as buyers locked in scarce guaranteed capacity. "GPU prices are rising and falling at the same time, depending on how you buy." The durable lesson isn't the numbers (snapshots — re-pull before quoting) but the **structure**: committed and spot diverge, and the spread *is* the risk premium on scarcity. Your commitment ladder is a bet on where that spread goes.

## Worked example

**A 3-team + prod fleet, 128 GPUs, laddered.**

Demand: prod inference needs a firm 32 GPUs; three research teams average 60 GPUs combined but spike to 110; batch sweeps are unbounded and interruptible.

- **Reserved (1-yr): 96 GPUs @ $2.35** — covers prod (32) + research P50 (64). Confident baseline.
- **On-demand: up to 32 GPUs @ $3.90** — absorbs research spikes to 128.
- **Spot: opportunistic** — batch sweeps run on any idle reserved/on-demand headroom *and* on spot when cheap, all in a preemptible Kueue queue that yields instantly to research reclaim.

Blended cost at 80% sustained utilisation of the reserved block plus modest burst lands well under all-on-demand, and the preemptible tier pushes *useful* utilisation toward 95% because idle reserved GPUs (already paid for) run batch work that vanishes the moment an owner reclaims. Break-even check: research sustains ~70% on its 64 reserved GPUs > 60%, so the commit is justified; the marginal 32 stay on-demand because their duty cycle is spiky and below break-even.

The showback report then bills each team for *used* GPU-hours against *reserved* quota, exposing the idle-reserved cost as a line item — which is exactly the signal that tells you whether next year's commit should grow or shrink.

## Practice

Two artifacts, both feeding the deliverable — runnable on the kind cluster.

1. **Survivable preemption demo.** Configure a best-effort preemptible `ClusterQueue` *below* a prod queue in the same cohort. Run a fake "training" pod that writes a checkpoint file (step counter) every N seconds to a mounted volume and, on start, resumes from it. Submit prod work that forces reclaim; confirm the best-effort pod is evicted and, when re-admitted, **resumes from its checkpoint** rather than step 0. Capture the eviction event and the resumed step.
2. **Commitment-ladder model.** Write a small spreadsheet or Python script: inputs = baseline/burst GPU demand, reserved/on-demand/spot rates, sustained utilisation; outputs = the reserved/on-demand/spot split, blended $/GPU-hr, and the break-even utilisation for the 1-yr commit. Apply it to the 128-GPU fleet from the capstone doc.

**Acceptance:**
- The preempted best-effort pod demonstrably **resumes from checkpoint** (log shows resumed step > 0), with the eviction captured.
- The ladder model outputs a blended rate + break-even and is applied to the 128-GPU inventory, with every $/hr flagged as a dated snapshot.

## Self-check

**(a) Why is preemption economically useless without checkpointing?**
**Answer:** Preemption's value is buying cheaper/oversubscribed capacity by reclaiming it when a higher tier needs it. If the preempted job restarts from scratch, you lose all compute-hours since it started — often *more* than the capacity saving. With checkpointing, you lose at most ~`T_ckpt/2` of work per preemption, so the expected waste is bounded and small. So preemption is only a cost optimisation when the workload resumes from a checkpoint; otherwise it's a compute-hours bonfire. It's a joint scheduler + training-loop property, not a scheduler knob.

**(b) At roughly what sustained utilisation does a 1-yr reserved commitment beat on-demand?**
**Answer:** When sustained utilisation of the reserved capacity exceeds `reserved_rate / ondemand_rate`. With ~$2.35 committed vs ~$3.90 on-demand that's ≈ **60%**: below 60% sustained use the reserved block is idle enough that on-demand would've been cheaper; above it, commit. The FinOps move is to measure your true utilisation floor (the showback report) and commit exactly up to that floor, bursting the spiky remainder on on-demand.

**(c) Why is "just autoscale the GPU cluster" wrong in 2026 — name the constraint?**
**Answer:** GPU supply is not elastic. In a shortage, on-demand capacity is *out of stock* — the autoscaler requests nodes that never arrive; forward capacity is booked months ahead. Compounding it, GPU node cold-start is minutes (driver + image + 10s–100s GB of weights), too slow to react to spikes, and the unit is a lumpy ~$15–25/hr 8-GPU node. So you *commit capacity ahead of demand and schedule within it* rather than autoscaling to meet demand.

## Resources

- **Kueue — Preemption & workload priority** — [concepts/preemption](https://kueue.sigs.k8s.io/docs/concepts/preemption/) · *(deep)* the reclaim/priority mechanics that make the tiers real; pair with `WorkloadPriorityClass`. Why: this is how you express "best-effort yields to prod" in practice.
- **SemiAnalysis — "The Great GPU Shortage: Rental Capacity"** — [newsletter.semianalysis.com](https://newsletter.semianalysis.com/p/the-great-gpu-shortage-rental-capacity) · *(deep)* the reserved/on-demand/spot + commitment thesis and the market structure behind the price inversion. Why: it's the argument you'll make in the capacity interview; read for the structure, not the (dated) numbers.
- **A current GPU pricing tracker** (e.g. a $/hr history page) · *(skim, snapshot only)* pull live reserved-vs-on-demand-vs-spot rates the day you build the ladder model. Why: every rate here is time-sensitive — quote it as a snapshot and re-pull before use.
