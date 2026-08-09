---
lesson: "08.4"
title: "Checkpointing"
module: "08"
concept: "Checkpointing"
status: not-started
est_time: "6h"
artifacts: []
---

# 08.4 · Checkpointing

> **Concept.** On a cluster that fails every few hours, effective throughput is set by how cheaply you can save state and restart — the Young/Daly optimal interval turns MTBF and checkpoint cost into a number.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Why this matters

This is the lesson where a FinOps-minded platform engineer wins. Llama 3 405B trained on
16,384 H100s for 54 days and hit **466 interruptions — 419 of them unexpected — roughly
one every ~3 hours** (arXiv 2407.21783, §3.3). At that failure rate the naive question
"how fast is the model" is the wrong question; the operative one is "**how much of the
54 days was effective**." Meta reports **>90% effective training time**, and the entire
delta between that and ~50% is *checkpoint and restart engineering*. Get the checkpoint
interval wrong in either direction and you burn money: too frequent and you spend the
cluster writing state instead of training; too rare and every failure rolls back hours of
16K-GPU compute. There is a formula that picks the interval, it is one line, and most
teams don't use it. That's your edge.

## What's new here (the platform view)

Lesson 08.3 framed lost MFU; lesson 08.5 will handle *detecting* the failure and rescheduling.
This lesson owns the **arithmetic of recovery**: given a failure rate and a cost-to-save,
what interval minimises wasted GPU-hours. Two levers, and you own both:

- **Checkpoint cost `C`** — seconds to durably write model + optimizer + RNG/dataloader
  state. You lower it with async, sharded, and distributed checkpointing.
- **Checkpoint interval `τ`** — how often you pay `C`. You set it, ideally from `C` and MTBF,
  not from a round number someone typed once.

The mental model is a bathtub of wasted time with two taps: **overhead** (time spent
writing checkpoints) and **rework** (time recomputing progress lost since the last
checkpoint). Checkpoint more often → overhead up, rework down. The optimum balances them,
and Young/Daly names it.

## Core notes

### The two waste terms

Over a long run, fraction of wall-clock **wasted** (not converted to forward progress):

```
waste(τ)  ≈   C / τ        +      τ / (2 · MTBF)
              └───┬───┘             └─────┬──────┘
          checkpoint overhead        expected rework
        (write cost each interval)  (lost work per failure ×
                                     failures per unit time)
```

- **Overhead `C/τ`**: you pay `C` seconds every `τ` seconds → fraction `C/τ`. Bigger
  interval, smaller overhead.
- **Rework `τ/(2·MTBF)`**: when a failure hits, the lost work is whatever you did since the
  last checkpoint — uniformly distributed on `[0, τ]`, so **expected loss ≈ τ/2**. Failures
  arrive at rate `1/MTBF`, so the rework fraction is `(τ/2)/MTBF`. Bigger interval, more
  rework.

Note the trade: overhead falls as `1/τ`, rework rises as `τ`. Their sum has a minimum.

### The Young/Daly formula

Minimise `waste(τ)`: `d/dτ [ C/τ + τ/(2M) ] = −C/τ² + 1/(2M) = 0`, where `M = MTBF`. Solve:

```
              ┌──────────────────┐
   τ_opt  ≈  √  2 · C · MTBF
              └──────────────────┘
```

This is the **Young (1974) / Daly (2006)** first-order optimal checkpoint interval — the
same `√(2μC)` HPC has used for 50 years, `μ = MTBF`. It assumes exponential (memoryless)
failures and `C ≪ MTBF`; Daly's second-order refinement (`τ = √(2C·M)·[1 + ⅓√(C/2M) + …] − C`)
matters only when `C` is a non-trivial fraction of `MTBF`, which for a well-tuned checkpoint
it isn't. Use the one-liner.

The **minimum waste** at `τ_opt` is clean — both terms become equal to `√(C/2M)`, so:

```
waste(τ_opt)  ≈  2 · √( C / (2 · MTBF) )  =  √( 2C / MTBF )
```

That is the entire lever in one expression: **wasted fraction scales as √(C / MTBF)**. You
can't change the cluster's MTBF much (that's hardware/08.5), so you attack `C`. Halving
checkpoint cost cuts the *unavoidable* waste by `√2 ≈ 29%`. This is why Meta invested in
fast checkpointing: at ~3-hr MTBF, driving `C` from minutes to tens of seconds is what
buys the ">90% effective" headline.

### Async, sharded, distributed checkpointing — how you lower `C`

`C` is not one number; it's a pipeline, and each technique attacks a different stage:

- **Naive / synchronous**: training stalls, every rank serialises its shard, GPU→CPU copy,
  CPU→durable-store write, everyone blocks until fsync. `C` = the whole chain. Worst case.
- **Sharded (distributed) checkpoint** (PyTorch DCP, `torch.distributed.checkpoint`): each
  rank writes only *its* shard of a sharded (FSDP/ZeRO) model, **in parallel**. Write
  bandwidth scales with world size instead of funnelling through rank 0. This is mandatory
  once the model doesn't fit on one GPU — rank 0 can't hold the full state anyway.
- **Async checkpoint**: the *blocking* part is only the GPU→pinned-CPU **staging copy**
  (fast, HBM→host bandwidth); the slow durable write to object store/parallel FS happens on
  a background thread while training **continues**. The `C` in the formula becomes just the
  staging stall, often **seconds**, not the full write. This is the biggest single win.
- **In-memory / peer redundancy** (e.g. CheckFreq, Gemini-style): stage to a *peer node's*
  RAM so a single-node failure recovers without touching durable storage at all; async-flush
  to durable less often as a backstop. Pushes effective `C` toward the copy cost only.

The FinOps read: async+sharded is the difference between `C ≈ 2 min` and `C ≈ 15 s`, which
via `√(2C·MTBF)` and `√(2C/MTBF)` compounds into both a longer safe interval *and* lower
floor waste. Also budget the **restart** side — detection + reschedule + reload + NCCL
re-init is dead time on *every* GPU (08.5); a 16K-GPU restart that takes 10 min is 2,700
GPU-hours per failure, so restart speed is as financial as write speed.

### Why checkpoint cost grows with model size — and what it does to the interval

Checkpoint bytes ≈ model + **optimizer state**. For Adam in mixed precision the optimizer
carries first + second moments plus often an FP32 master copy → roughly **≥12–16 bytes per
parameter** (2-byte weight + 2-byte grad-ish + 4+4 moments + 4 master ≈ 16B/param is a fair
upper bound; ~14 GB for a 7B model, ~5–6 TB for 405B). `C ≈ checkpoint_bytes / write_bandwidth`,
and bytes scale **linearly with parameter count** while your durable write bandwidth is
roughly fixed → **`C` grows with the model**.

Feed that back into Young/Daly: `τ_opt = √(2·C·MTBF)` grows as `√C`, so bigger models want
*longer* intervals — but the floor waste `√(2C/MTBF)` also grows as `√C`, so bigger models
are **strictly more expensive to protect** at a given MTBF. That is exactly why frontier
runs invest so hard in **sharded + async**: sharding makes the *effective* per-rank `C`
grow much slower than the total state, and async hides most of it, keeping the interval and
the floor waste tolerable even at multi-terabyte checkpoints.

## Worked example

Same cluster shape as Llama 3: **MTBF = 3 h**, and suppose your measured checkpoint cost is
**C = 2 min**. Convert to one unit (minutes): `MTBF = 180 min`, `C = 2 min`.

```
τ_opt = √(2 · C · MTBF) = √(2 · 2 · 180) = √720 ≈ 26.8 min
```

**Checkpoint roughly every 27 minutes.** Sanity-check the waste:

```
overhead = C/τ        = 2 / 26.8      = 0.075   (7.5%)
rework   = τ/(2·MTBF) = 26.8 / 360    = 0.074   (7.4%)   ← the two terms match, as they should
waste    ≈ √(2C/MTBF) = √(4/180)      = 0.149   (~14.9%)
```

So at `C = 2 min`, ~15% of wall-clock is unavoidable overhead+rework → ~85% effective. Now
apply the lever: async-checkpoint drops the blocking `C` to **15 s = 0.25 min**:

```
τ_opt = √(2 · 0.25 · 180) = √90 ≈ 9.5 min
waste ≈ √(2·0.25/180) = √0.00278 = 0.053   (~5.3%)  → ~94.7% effective
```

Same cluster, same MTBF — dropping `C` from 2 min to 15 s took effective time from ~85% to
~95%, i.e. into Llama-3 ">90%" territory, *and* let you checkpoint more often (9.5 min) for
less total waste. That is the whole game, and it's an engineering choice, not a hardware one.

## Practice

Feeds the **Survive-a-failure lab** deliverable. Extend the DDP job from lesson 08.1 on your
**2 rented GPUs**.

1. Add **save/resume**: checkpoint model + optimizer + RNG + step counter (`torch.save`,
   or `torch.distributed.checkpoint` if you're sharded). Resume must be *exact* — reload,
   confirm the loss curve continues without a discontinuity, not a fresh start.
2. **Measure `C`**: time the checkpoint write (wall-clock seconds), median of ≥5 saves.
   Measure it **twice** — synchronous, then async (background thread doing the durable
   write) — and record both. Report checkpoint size in bytes so you can see the
   bytes/param → seconds relationship.
3. **Kill and resume**: `kill -9` a rank mid-run, restart from the last checkpoint, and
   confirm correctness. Note the *restart* wall-time (reload + NCCL re-init) separately from `C`.
4. **Compute `τ_opt`** for a stated **MTBF = 3 h** using your measured `C`
   (`τ_opt = √(2·C·MTBF)`), and report the resulting waste `√(2C/MTBF)`.

**Acceptance:** a **measured checkpoint cost `C`** (sync and async) **+ a computed optimal
interval `τ_opt` for MTBF = 3 h**, with the waste fraction, committed under
`../practice/survive-a-failure/`. One paragraph: how much did async buy you, and what's the
restart cost you'd multiply across 16K GPUs.

## Self-check

**(a) Derive the optimal checkpoint interval for MTBF = 3 h and checkpoint cost = 2 min
(show the Young/Daly calc).**
**Answer:** Work in minutes: `MTBF = 180`, `C = 2`. Waste `= C/τ + τ/(2·MTBF)`;
differentiate and set to zero: `−C/τ² + 1/(2·MTBF) = 0 → τ² = 2·C·MTBF`. So
`τ_opt = √(2·C·MTBF) = √(2·2·180) = √720 ≈ 26.8 min` — **checkpoint every ~27 minutes**.
The two waste terms then match (overhead `2/26.8 ≈ 7.5%`, rework `26.8/360 ≈ 7.4%`) and
total unavoidable waste `≈ √(2C/MTBF) = √(4/180) ≈ 15%`.

**(b) What does async/sharded checkpointing buy you?**
**Answer:** They cut the `C` that enters the formula. **Async** moves the slow durable
write to a background thread; only the fast GPU→pinned-CPU staging copy blocks training, so
`C` drops from the full write time (minutes) to seconds. **Sharded/distributed** has each
rank write only its own shard in parallel, so write bandwidth scales with world size instead
of bottlenecking through rank 0 (and it's mandatory when the full state won't fit on one
GPU). Because both `τ_opt = √(2C·MTBF)` and the floor waste `√(2C/MTBF)` scale as `√C`,
halving `C` cuts unavoidable waste by ~29% — the concrete mechanism behind Llama 3's >90%
effective time on a ~3-hr-MTBF cluster.

**(c) Why does checkpoint cost grow with model size, and how does that change the optimal
interval?**
**Answer:** Checkpoint bytes = model weights + optimizer state (Adam mixed-precision ≈
12–16 bytes/param), which scales **linearly with parameter count**, while durable write
bandwidth is roughly fixed — so `C ≈ bytes/bandwidth` grows with the model (~14 GB for 7B,
multi-TB for 405B). In Young/Daly, `τ_opt = √(2·C·MTBF)` grows as `√C`, so bigger models
want *longer* intervals — but the floor waste `√(2C/MTBF)` also grows as `√C`, so they're
strictly more expensive to protect at a given MTBF. That's why frontier runs lean on
**sharded** (per-rank `C` grows far slower than total state) and **async** (hides most of
the write), keeping both the interval and the waste tolerable at terabyte scale.

## Resources

1. **Llama 3 paper, §3.3 "Infrastructure, Scaling, and Efficiency"** —
   https://arxiv.org/abs/2407.21783 — the anchor. 16K H100s, 466 interruptions (419
   unexpected) in 54 days, >90% effective training time, and the checkpoint/restart
   engineering behind it. Read it deeply; it's the module's spine.
2. **Young/Daly optimal-checkpoint-interval formula** — Daly, *"A higher order estimate of
   the optimum checkpoint interval for restart dumps"* (FGCS 2006), building on Young (1974);
   overview: https://icl.utk.edu/files/publications/2022/icl-utk-1569-2022.pdf — the
   derivation of `√(2μC)` and when the second-order correction matters.
3. **PyTorch Distributed Checkpoint (DCP)** —
   https://pytorch.org/docs/stable/distributed.checkpoint.html — the sharded + async APIs
   you'll actually call in the practice task.
