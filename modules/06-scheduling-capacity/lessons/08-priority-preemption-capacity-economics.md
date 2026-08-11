---
lesson: "06.8"
title: "Priority, preemption, and capacity economics"
module: "06"
concept: "Priority, preemption, and capacity economics"
status: not-started
est_time: "12h"
prev: "07-fragmentation-effective-capacity.md"
next: null
artifacts: []
sources: 8
---

# 06.8 · Priority, preemption, and capacity economics

> **Concept.** Priority tiers with *survivable* (checkpoint-aware) preemption, and a reserved / on-demand / spot commitment ladder for a fleet you cannot autoscale.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

This is the module's last lesson, and it is deliberately the one that pulls every earlier thread into a single decision. Lesson 07 taught you to measure the gap between allocated and usable capacity and put a dollar figure on it. Lessons 03–04 taught you how Kueue expresses quota, cohorts, and reclaim. Lesson 02 taught you why a preempted *gang* has to re-admit atomically, not pod-by-pod. This lesson asks: once you know what capacity you have and who can reclaim it, how do you *price* the reclaiming, and how much of your fleet do you commit to buying ahead of time versus renting as you go? Fairness (who gets preempted), survivability (what preemption actually costs), and commitment strategy (what you buy) are one connected decision, not three separate ones — and it is the decision your target companies pay Staff-level compensation for someone to own.

## Why this matters

This is the lesson where your FinOps background lets you out-answer a pure SWE. Preemption is how you sweat a fixed fleet toward ~100% utilisation without starving production — but only if the preempted work *survives*, which makes it a joint scheduler + training-loop decision, not a scheduler knob. And capacity planning for accelerators breaks every reflex a cloud engineer has: you cannot autoscale your way out of scarcity, so the lever is *commitment strategy*, not elasticity. "Design the reserved/on-demand/spot mix and defend the break-even" is a standard senior interview question, and it's yours to dominate — provided you can also say, correctly, *which market* your numbers came from. Getting that qualification wrong (quoting a neocloud rate as if it were universal) is the single fastest way to look junior in an otherwise strong answer.

## What's new here (calibration)

You know spot instances, priority classes, and reserved-instance math from AWS. What's new is the accelerator-specific inversion of all three, plus two facts that didn't exist when this lesson was first written:

- **Priority + preemption** here means *Kueue* reclaim of *borrowed* quota (Lesson 04), plus pod `PriorityClass` — and the hard constraint that a preempted training job is worthless unless it checkpointed.
- **Spot** for GPUs is a different animal: preemptible capacity that is often *unavailable at any price* during a shortage, not merely pricier.
- **Reserved-instance math** inverts: committed GPU prices can *rise* while on-demand *falls* — the opposite of the CPU world you're used to, and, as of early 2026, the market has moved further still (see below).
- **New lever**: async, sharded distributed checkpointing changes the actual cost formula for preemption — this lesson's core `T_ckpt/2` waste term is not fixed; modern checkpointing shrinks it, and knowing the mechanism (not just the policy) is itself interview-differentiating.

## Core concepts

### Priority tiers

A shared GPU fleet runs at least three tiers, expressed as pod `PriorityClass` values and mirrored in Kueue queue/workload priority:

| Tier | Example | Preemptible? | Guarantee |
|------|---------|--------------|-----------|
| **prod** | inference serving, on-call | no | protected floor; preempts others |
| **research** | interactive training, experiments | reclaimable when over quota | nominal quota, can borrow |
| **best-effort** | batch sweeps, backfill | freely | runs only on idle capacity |

The design goal: prod never waits; research gets a guaranteed floor plus borrowable headroom; best-effort mops up the remaining idle so the fleet approaches 100% *useful* utilisation. Kueue expresses this with `ClusterQueue` priorities + cohort borrowing (Lesson 04); the best-effort queue has a low `lendingLimit` claim and its workloads carry a low `WorkloadPriorityClass`, so they are the first evicted when an owner reclaims.

This three-tier design is not a Kubernetes invention. It descends directly from **Google's Borg** paper (Verma et al., EuroSys 2015) — the original large-scale cluster manager, predating Kubernetes by roughly a decade — which describes a priority-band hierarchy (monitoring, then production, then batch, then best-effort/"gratis" work) with one deliberate, non-obvious rule: **production-tier tasks do not preempt each other**, even though they are nominally equal priority and preemption "should" be neutral between equals. The reason is that allowing same-tier preemption invites **preemption cascades** — task A evicts task B, which now needs to be re-placed and may itself evict task C to fit, and so on, turning one legitimate reclaim into a chain reaction that destabilizes the whole tier. The fix is structural, not a tie-breaker rule: same-tier tasks simply cannot preempt each other, full stop. This is worth stating explicitly and with its reasoning in an interview — copying the rule without the cascade-avoidance justification behind it reads as memorization, not understanding.

### Survivable preemption is a training-loop property

Preemption is only *economically* usable if the preempted workload loses no more than the time since its last checkpoint. A job that restarts from scratch on preemption turns a scheduling win into a compute-hours loss — you preempted to *save* money and spent more.

So preemption design is a contract with the workload:
- The training loop must **checkpoint** (model + optimizer + step) at an interval `T_ckpt`.
- On preemption, the job is re-queued and **resumes from the last checkpoint**, losing at most `T_ckpt` of work plus one checkpoint-restore round-trip.
- The scheduler should give a **grace period** (`terminationGracePeriodSeconds`, or Kueue's preemption with a delay) long enough for a final checkpoint-on-SIGTERM where feasible.

The economic rule: **expected wasted work per preemption ≈ T_ckpt/2** (uniform arrival), so checkpoint frequently enough that `T_ckpt/2 × preemption_rate` is small relative to the cheaper capacity preemption buys.

> **Interview one-liner:** *"Preemption without checkpointing isn't a cost optimisation, it's a compute-hours bonfire."*

### The checkpoint frequency ceiling just moved: async, sharded checkpointing

The classic framing above treats `T_ckpt` as roughly fixed — "checkpointing is slow, so you do it every N minutes and eat the risk in between." That folk wisdom is dated. PyTorch's `torch.distributed.checkpoint` (DCP) — production-mature since PyTorch 2.1, with async and sharded saves as first-class, documented APIs — changes what `T_ckpt` can be:

- **Sharded, parallel saves.** Each rank writes only *its own shard* of the model/optimizer state, in parallel with every other rank, instead of gathering the full state onto one rank and serializing it — the classic bottleneck at scale.
- **Async save.** The GPU-blocking portion of a checkpoint shrinks to the device→host memory copy; the actual durable write to storage happens on background CPU threads/async I/O, overlapped with the next steps of training instead of stalling the GPU.
- **Net effect**: checkpoint *frequency* is decoupled from checkpoint *training-stall cost* far more than a naive synchronous-save model implies. A team that could only afford a checkpoint every 30 minutes under synchronous saves can often afford one every 5 minutes under async sharded saves, for a similar stall cost.

This directly attacks the lesson's core formula: `expected_waste ≈ T_ckpt/2 × preemption_rate` shrinks proportionally as `T_ckpt` shrinks. **The checkpointing mechanism, not just the policy of "checkpoint often," is itself a cost lever a platform engineer should be pushing training teams toward** — and it's a concrete, current (2025–2026) technical fact that most engineers reciting the `T_ckpt/2` formula won't mention.

### Why you cannot autoscale a GPU cluster

CPU autoscaling assumes elastic supply: the cloud always has more `m5.large`s. GPUs break that assumption on three axes:

1. **Supply is finite and booked forward.** In a shortage, on-demand H100 capacity is simply *out of stock* in your region/zone for hours or days — Cluster Autoscaler asks for nodes that never come. Forward capacity is reserved months ahead.
2. **Cold-start is minutes, not seconds.** A GPU node must pull a multi-GB driver + container image and often 10s–100s of GB of model weights; scale-up latency is minutes (see Module 07 cold-start), useless for reactive autoscaling of spiky demand.
3. **The unit is lumpy and expensive.** You scale in 8-GPU nodes at a meaningfully-priced-per-hour cost each (see the market-segment table below); a mis-sized scale-up is a large, immediate bill, so you plan capacity rather than react to it.

The consequence: for GPUs you **commit capacity ahead of demand** and *schedule* within it (this whole module) rather than autoscaling to meet demand.

### The commitment ladder

Because you buy capacity ahead, you ladder commitments against uncertain demand:

- **Reserved / committed (1–3 yr, or capacity blocks):** cheapest per hour, but you pay whether or not you use it. Sized to your **P50 baseline** — the demand you're confident exists.
- **On-demand:** most expensive per hour in the *neocloud* segment (see below), no commitment. Absorbs the **burst above baseline** — and, crucially, is *not guaranteed available* in a shortage.
- **Spot / preemptible:** cheapest when available, evictable. Runs **best-effort/checkpointable** work only, and only opportunistically.

Blended cost of a laddered fleet:

```
blended $/GPU-hr = (reserved_hrs × reserved_rate
                  + ondemand_hrs × ondemand_rate
                  + spot_hrs × spot_rate) / total_hrs
```

**Break-even for a commitment:** a 1-yr reserve beats on-demand once your *sustained* utilisation of that capacity exceeds `reserved_rate / ondemand_rate`.

### Market-segment discipline: three different markets, four different numbers

**This is the single most important correction to make to the naive "$X/GPU-hr" interview answer.** GPU pricing is not one market — it is at least three, and they can move in opposite directions at the same time:

| Segment | What it is | Rough 2026 snapshot* |
|---|---|---|
| **Hyperscaler retail** | AWS/Azure/GCP on-demand, billed by the standard public rate card | high — roughly $7–13/GPU-hr for H100-class on-demand depending on provider and instance shape |
| **Specialized neocloud** | CoreWeave-style GPU-native clouds, on-demand | roughly $2–4/GPU-hr on-demand; committed/reserved often lower still |
| **Spot / preemptible** | evictable capacity at any provider, when available at all | as low as ~$0.42–$1/GPU-hr when available; broad-market H100 spot pricing fell roughly 88% from Jan 2024 to Sep 2025 as supply caught demand |

*Snapshot as of **August 2026**; these figures move fast — always re-pull a live tracker before quoting a number, and always name the segment when you do.

An unqualified "$X/GPU-hr" answer in an interview reads as naive precisely because the spread between segments is 4–6×. If a worked example in this lesson uses ~$2.35 (committed) and ~$3.90 (on-demand), **those numbers sit inside the specialized-neocloud band, not hyperscaler retail** — say so explicitly whenever you use them, the same way you'd flag any dated pricing figure as a snapshot.

### The price-inversion thesis (know the shape, not the numbers)

Through 2024–2026, on-demand and committed GPU pricing in the **neocloud rental segment specifically** have at times moved in *opposite* directions: on-demand rates fell as new supply came online, while 1-yr committed rates *rose* as buyers rushed to lock in guaranteed capacity ahead of the next wave of scarcity. By early-to-mid 2026, market trackers describe renewed tightness: 1-year committed rental pricing pushing above $2/GPU-hr and rising further within weeks, with on-demand capacity across GPU types effectively sold out at several providers. The durable lesson isn't any single number (they are all snapshots — re-pull before quoting) but the **structure**: committed and on-demand pricing can diverge, sometimes sharply, and the spread between them *is the market's price on scarcity*. Your commitment ladder is a bet on where that spread goes, sized against your own measured utilisation floor (the showback report from this module's deliverable), not against a headline number from a blog post.

## Perspectives

**Developer/training-loop.** Checkpoint frequency is a decision the *training code* owner makes, but its economic consequences are paid by the *platform*. A training team that checkpoints rarely "because it's slow" — without knowing async sharded checkpointing exists — is unknowingly making the platform's preemption strategy economically worse. This is a classic cross-team incentive gap: the platform engineer has to surface it with data (the `T_ckpt/2` math) and a concrete fix (point them at `torch.distributed.checkpoint`'s async API), not just ask nicely for more frequent checkpoints.

**Operator.** Grace-period sizing (`terminationGracePeriodSeconds`, or Kueue's preemption delay) is a hard tradeoff: long enough for a clean final checkpoint, short enough that a hung job doesn't block reclaim indefinitely. Ground it in the checkpointing numbers above — an operator who knows a team's checkpoints are async and cheap can set a *shorter* grace period than one dealing with a team still on synchronous, minutes-long saves, because the expected in-flight work at eviction time is smaller.

**Hardware/systems.** Async, sharded distributed checkpointing works by exploiting a real hardware asymmetry: the GPU-blocking cost of a checkpoint is only the device→host memory copy (fast, on the order of the PCIe/NVLink bandwidth to host memory), while the durable write to network storage — the slow part — happens on CPU threads, overlapped with the *next* training step instead of stalling the GPU. Understanding *why* this decouples frequency from cost (it's a hardware-bandwidth argument, not a software cleverness argument) is what separates "I heard checkpointing got faster" from an answer that survives a follow-up question.

**Economics/market-structure.** The 2026 price-inversion thesis is specifically a **neocloud GPU-rental market** phenomenon; it must be explicitly distinguished from the **hyperscaler retail** market, where on-demand pricing sits far higher in absolute terms and moves on a different cycle. Treating "the price of an H100" as one number across both markets is the single fastest way to sound naive discussing GPU economics with someone who works in the market day to day — the market-segment table above exists to make that distinction reflexive.

## Real-world use cases

- **PyTorch — "Distributed Checkpoint: Efficient checkpointing in large-scale jobs"** — https://pytorch.org/blog/distributed-checkpoint-efficient-checkpointing-in-large-scale-jobs/ (search-verified; blocked by this session's egress proxy, canonical URL confirmed). What it shows: the PyTorch team's own account of async, process-based checkpointing, save-plan caching, and rank-local checkpointing — the concrete engineering mechanism that operationalizes "checkpoint frequently enough that `T_ckpt/2 × preemption_rate` is small."
- **PyTorch/IBM — "Performant Distributed Checkpointing in Production with IBM"** — https://pytorch.org/blog/performant-distributed-checkpointing/ (search-verified; blocked by this session's egress proxy, canonical URL confirmed). What it shows: a named production deployment (IBM) running sharded/async DCP at scale — evidence this isn't a benchmark-only feature but a real cost lever in use.
- **OpenAI — "Scaling Kubernetes to 7,500 Nodes"** — https://openai.com/index/scaling-kubernetes-to-7500-nodes/ (search-verified; blocked by this session's egress proxy, canonical URL confirmed). What it shows: "team taints" and priority-weighted "balloon" deployments used in production as the soft-reclaim mechanism this lesson's tier design generalizes — a first-party account of borrowing/reclaim at hyperscale, reused here from a capacity-economics angle rather than the gang-scheduling angle of Lessons 01–02.
- **Google — "Large-scale cluster management at Google with Borg"** (Verma et al., EuroSys 2015) — https://research.google.com/pubs/archive/43438.pdf (search-verified; blocked by this session's egress proxy, canonical URL confirmed). What it shows: the foundational priority-band model — monitoring > production > batch > best-effort, with same-tier production tasks deliberately *not* preempting each other to avoid cascades — that the lesson's three-tier table directly descends from, predating Kubernetes by a decade.
- **SemiAnalysis — "The Great GPU Shortage: Rental Capacity"** — https://newsletter.semianalysis.com/p/the-great-gpu-shortage-rental-capacity (search-verified; blocked by this session's egress proxy, canonical URL confirmed). What it shows: the neocloud rental market's structure and 2026 scarcity dynamics behind the committed-vs-on-demand price inversion — read for the structure of the argument, not the specific numbers, which move week to week.

## Worked example

**A 3-team + prod fleet, 128 GPUs, laddered (numbers are a specialized-neocloud snapshot — see the market-segment table above).**

Demand: prod inference needs a firm 32 GPUs; three research teams average 60 GPUs combined but spike to 110; batch sweeps are unbounded and interruptible.

- **Reserved (1-yr): 96 GPUs @ ~$2.35/GPU-hr** — covers prod (32) + research P50 (64). Confident baseline.
- **On-demand: up to 32 GPUs @ ~$3.90/GPU-hr** — absorbs research spikes to 128.
- **Spot: opportunistic** — batch sweeps run on any idle reserved/on-demand headroom *and* on spot when cheap, all in a preemptible Kueue queue that yields instantly to research reclaim.

Blended cost at 80% sustained utilisation of the reserved block plus modest burst lands well under all-on-demand, and the preemptible tier pushes *useful* utilisation toward 95% because idle reserved GPUs (already paid for) run batch work that vanishes the moment an owner reclaims. Break-even check: `reserved_rate / ondemand_rate ≈ 2.35/3.90 ≈ 60%`; research sustains ~70% on its 64 reserved GPUs > 60%, so the commit is justified; the marginal 32 stay on-demand because their duty cycle is spiky and below break-even.

**Now add the checkpoint-mechanism lever.** Suppose the preemption rate on the best-effort queue is 4 evictions/day per running job. Under **synchronous, infrequent checkpointing** (`T_ckpt = 30 min`), expected waste per preemption ≈ 15 min of GPU-time; at 4 evictions/day across, say, 20 concurrently-running best-effort jobs, that's `20 × 4 × 15min = 1,200 GPU-minutes/day ≈ 20 GPU-hours/day` burned purely to preemption-restart overhead. Under **async, sharded checkpointing** (`T_ckpt = 5 min`, made affordable by `torch.distributed.checkpoint`'s overlap of I/O with compute), expected waste per preemption ≈ 2.5 min; the same 20 jobs at 4 evictions/day burn `20 × 4 × 2.5min = 200 GPU-minutes/day ≈ 3.3 GPU-hours/day` — a **~83% reduction in preemption-waste overhead** (20 → 3.3 GPU-hr/day), at essentially zero additional infrastructure cost, just from adopting a modern checkpointing library. At the ~$2.35–$3.90/GPU-hr neocloud snapshot, the ~16.7 GPU-hr/day of avoided waste is roughly $39–$65/day recovered on this one queue alone — small on paper, but it compounds across every preemptible queue in the fleet, and it's a lever the platform team can push *without* touching the commitment ladder at all.

The showback report then bills each team for *used* GPU-hours against *reserved* quota, exposing the idle-reserved cost as a line item — which is exactly the signal that tells you whether next year's commit should grow or shrink.

## Practice

Two artifacts, both feeding the deliverable — runnable on the kind cluster.

1. **Survivable preemption demo.** Configure a best-effort preemptible `ClusterQueue` *below* a prod queue in the same cohort. Run a fake "training" pod that writes a checkpoint file (step counter) every N seconds to a mounted volume and, on start, resumes from it. Submit prod work that forces reclaim; confirm the best-effort pod is evicted and, when re-admitted, **resumes from its checkpoint** rather than step 0. Capture the eviction event and the resumed step. Stretch: vary N (simulating `T_ckpt`) and log the wasted-step count per eviction to reproduce the worked example's "async vs sync checkpointing" comparison yourself, even without real async I/O.
2. **Commitment-ladder model.** Write a small spreadsheet or Python script: inputs = baseline/burst GPU demand, reserved/on-demand/spot rates **(labeled by market segment)**, sustained utilisation; outputs = the reserved/on-demand/spot split, blended $/GPU-hr, and the break-even utilisation for the 1-yr commit. Apply it to the 128-GPU fleet from the capstone doc.

**Acceptance:**
- The preempted best-effort pod demonstrably **resumes from checkpoint** (log shows resumed step > 0), with the eviction captured.
- The ladder model outputs a blended rate + break-even and is applied to the 128-GPU inventory, with every $/hr flagged as a dated snapshot **and labeled by market segment** (hyperscaler retail / specialized neocloud / spot).

## Common pitfalls

- **Quoting a single "$/GPU-hr" number in an interview without naming the market segment.** Hyperscaler retail, specialized neocloud, and spot are three different markets with 4–6× spreads between them; an unqualified number reads as naive. Always say which segment and which month.
- **Assuming synchronous checkpointing sets the checkpoint-frequency ceiling.** With async/sharded checkpointing (`torch.distributed.checkpoint`), far more frequent checkpoints are affordable than the "checkpointing is slow, do it rarely" folk wisdom assumes. Failing to mention this in an interview is a missed opportunity to show current knowledge, not just a missed optimization.
- **Copying Borg's "production never preempts production" rule without the reason.** It's not arbitrary — it exists to avoid preemption cascades, where evicting one same-tier task forces a chain of further evictions. State the rule *and* the reasoning.
- **Believing "just autoscale the GPU cluster" is a real option.** GPU supply is not elastic. In a shortage, on-demand capacity is out of stock; forward capacity is booked months ahead; cold-start is minutes, not seconds; and the unit is a lumpy, expensive node. You commit capacity ahead of demand and schedule within it — you don't react your way out of scarcity.

## Self-check

- Why is preemption economically useless without checkpointing? **Answer:** Preemption's value is buying cheaper/oversubscribed capacity by reclaiming it when a higher tier needs it. If the preempted job restarts from scratch, you lose all compute-hours since it started — often *more* than the capacity saving. With checkpointing, you lose at most ~`T_ckpt/2` of work per preemption, so the expected waste is bounded and small. So preemption is only a cost optimisation when the workload resumes from a checkpoint; otherwise it's a compute-hours bonfire. It's a joint scheduler + training-loop property, not a scheduler knob.
- At roughly what sustained utilisation does a 1-yr reserved commitment beat on-demand? **Answer:** When sustained utilisation of the reserved capacity exceeds `reserved_rate / ondemand_rate`. With ~$2.35 committed vs ~$3.90 on-demand (a specialized-neocloud snapshot) that's ≈ **60%**: below 60% sustained use the reserved block is idle enough that on-demand would've been cheaper; above it, commit. The FinOps move is to measure your true utilisation floor (the showback report) and commit exactly up to that floor, bursting the spiky remainder on on-demand.
- Why is "just autoscale the GPU cluster" wrong — name the constraint? **Answer:** GPU supply is not elastic. In a shortage, on-demand capacity is *out of stock* — the autoscaler requests nodes that never arrive; forward capacity is booked months ahead. Compounding it, GPU node cold-start is minutes (driver + image + 10s–100s GB of weights), too slow to react to spikes, and the unit is a lumpy, expensive multi-GPU node. So you *commit capacity ahead of demand and schedule within it* rather than autoscaling to meet demand.
- Why does Google's Borg design explicitly forbid production-tier tasks from preempting each other, even though they're equal priority and preemption "should" be neutral between equals? **Answer:** Because allowing same-tier preemption invites preemption cascades: task A evicts task B, B has to be re-placed and may itself evict task C to fit, and so on — a chain reaction that can destabilize an entire tier of production traffic. Forbidding same-tier preemption structurally is the fix, not a tie-breaker rule layered on top of allowed preemption.
- How does async, sharded distributed checkpointing (`torch.distributed.checkpoint`) change the `T_ckpt/2` expected-waste calculation from this lesson's core formula, and what term does it shrink? **Answer:** It shrinks `T_ckpt` itself, not the formula's structure. By having each rank save its own shard in parallel and overlapping the durable write with training compute (only the device→host copy blocks the GPU), teams can checkpoint far more frequently for the same stall cost than synchronous saves allow — so the achievable `T_ckpt` drops (e.g., from ~30 min to ~5 min in the worked example), directly shrinking the expected-waste-per-preemption term `T_ckpt/2` and the total wasted GPU-hours across every preemption, without changing the preemption rate or the commitment ladder at all.

## Connections & what's next

This lesson closes the loop the module opened in Lesson 01: the default scheduler's atomicity gap (01–02) is fixed by gang admission; Kueue (03–04) turns admission into a queueable, quota-bearing, cohort-sharing system; alternatives (05) and topology (06) refine *how* work is placed; fragmentation (07) measures what's left over; and this lesson prices it — who gets to reclaim it, how survivably, and how much of the fleet you commit to owning versus renting. That "commit versus rent" output is exactly what feeds forward: **Module 11 (GPU cost economics)** consumes this module's capacity-economics work as an input — the commitment ladder, the break-even utilisation, and the showback report you build here become the raw data a FinOps-facing cost model builds on.

There is no next lesson in this module — the [**checkpoint**](../checkpoint.md) is next. Use it to prove, unaided, that you can reproduce the deadlock and fix, explain Kueue cold, compute effective capacity, design the 128-GPU queues, build the commitment mix, and produce the showback report. That checkpoint, plus the [Kueue setup + per-queue showback](../practice/kueue-showback/README.md) deliverable, is the gate for the whole module.

## References & further reading

**Primary sources**
- Google — "Large-scale cluster management at Google with Borg" (Verma et al., EuroSys 2015) — https://research.google.com/pubs/archive/43438.pdf — read for the original priority-band design and the cascade-avoidance reasoning behind "production never preempts production."
- Kueue — Concepts: Preemption — https://kueue.sigs.k8s.io/docs/concepts/preemption/ — read for the reclaim/priority mechanics (`WorkloadPriorityClass`, cohort reclaim) that make the tiers in this lesson real.
- PyTorch — `torch.distributed.checkpoint` official docs — https://docs.pytorch.org/docs/stable/distributed.checkpoint.html — read for the `save`/`async_save`/`load` API that operationalizes the checkpoint-frequency lever.

**Real-world engineering blogs**
- PyTorch — "Distributed Checkpoint: Efficient checkpointing in large-scale jobs" — https://pytorch.org/blog/distributed-checkpoint-efficient-checkpointing-in-large-scale-jobs/ — what it shows: async, sharded, per-rank-parallel checkpoint saves, direct from the team that built them.
- PyTorch/IBM — "Performant Distributed Checkpointing in Production with IBM" — https://pytorch.org/blog/performant-distributed-checkpointing/ — what it shows: a named production deployment running this checkpointing mechanism at scale.
- OpenAI — "Scaling Kubernetes to 7,500 Nodes" — https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — what it shows: team taints and priority "balloons" as the production reclaim mechanism this lesson's tiers generalize.
- SemiAnalysis — "The Great GPU Shortage: Rental Capacity" — https://newsletter.semianalysis.com/p/the-great-gpu-shortage-rental-capacity — what it shows: the neocloud rental market structure and 2026 scarcity dynamics behind the committed-vs-on-demand price inversion; read for structure, not the (dated) numbers.

**Deeper dives**
- getdeploying.com — "H100 Cloud Pricing: Compare 48+ Providers" — https://getdeploying.com/gpus/nvidia-h100 — a live, continuously updated cross-provider pricing tracker spanning hyperscaler retail to specialty-cloud spot; pull a fresh snapshot the day you build your ladder model, and note the segment for whatever number you cite.
