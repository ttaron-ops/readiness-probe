---
lesson: "08.4"
title: "Checkpointing"
module: "08"
concept: "Checkpointing"
status: not-started
est_time: "9h"
prev: "03-communication-bottleneck.md"
next: "05-failure-and-elasticity.md"
artifacts: []
sources: 9
---

# 08.4 · Checkpointing

> **Concept.** On a cluster that fails every few hours, effective throughput is set by how cheaply you can save state and restart — the Young/Daly optimal interval turns MTBF and checkpoint cost into a number.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

08.3 gave you MFU and named "failure overhead" as the third of the three usual causes of
lost throughput — the one a long-window MFU silently absorbs. This lesson is that cause,
fully worked: the arithmetic of how much wasted time failure overhead actually costs, and
the two levers (checkpoint cost `C`, checkpoint interval `τ`) you own to shrink it. It is
the module's **throughput lever** — the single decision with the largest, most directly
FinOps-defensible dollar impact of any lesson in this module — which is why it gets the
extra depth here. 08.5 picks up immediately after: once a checkpoint exists, *detecting*
the failure and re-forming the world around that checkpoint is its job. This lesson owns
the checkpoint; 08.5 owns the recovery loop around it.

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
teams don't use it. That's your edge. (16,384 H100s and ~$2–4/GPU-hr are dated snapshots —
the arithmetic in this lesson outlives any specific price point.)

## What's new here (calibration)

Lesson 08.3 framed lost MFU; lesson 08.5 will handle *detecting* the failure and rescheduling.
This lesson owns the **arithmetic of recovery**: given a failure rate and a cost-to-save,
what interval minimises wasted GPU-hours. Two levers, and you own both:

- **Checkpoint cost `C`** — seconds to durably write model + optimizer + RNG/dataloader
  state. You lower it with async, sharded, and distributed checkpointing.
- **Checkpoint interval `τ`** — how often you pay `C`. You set it, ideally from `C` and MTBF,
  not from a round number someone typed once.

We assume 02b's topology vocabulary, 06's gang-scheduling vocabulary, and 05's XID/DCGM
vocabulary throughout — MTBF is a given input here, not derived; 08.5 is where the
health-signal-to-MTBF story lives. The mental model is a bathtub of wasted time with two
taps: **overhead** (time spent writing checkpoints) and **rework** (time recomputing
progress lost since the last checkpoint). Checkpoint more often → overhead up, rework
down. The optimum balances them, and Young/Daly names it.

## Core concepts

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
buys the ">90% effective" headline. It's worth pausing on *why* a 50-year-old HPC result
(originally derived for tape-dump checkpointing on 1970s mainframes) is exactly the right
tool for a 2024 GPU fleet: the underlying assumption — exponential, memoryless failures,
and a save cost you control — holds just as well for a GPU falling off the bus as it did
for a punch-card-era hardware fault. The math doesn't care what's failing.

### Async, sharded, distributed checkpointing — how you lower `C`

`C` is not one number; it's a **pipeline**, and each technique attacks a different stage.
Think of it as four hops, each with its own bottleneck: **GPU HBM → pinned host memory
(staging) → local/network buffer → durable store (fsync)**. Different techniques collapse
different hops:

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
  PyTorch's own engineering blog on Distributed Checkpoint (References) documents a
  concrete production wrinkle worth knowing: the naive thread-based async implementation
  hit **GIL contention** between the background checkpoint thread and the trainer's
  Python threads, which quietly ate back part of the win by stalling the training loop
  during the staging/metadata-collective phase. The fix was **process-based** async
  checkpointing — a separate process with its own GIL does the serialization and I/O,
  communicating with the trainer via shared memory so staged tensors don't need to be
  copied between processes. The lesson generalizes: "async" is a design goal, not a
  guarantee — the implementation detail (thread vs process) determines whether you
  actually get it.
- **In-memory / peer redundancy** (e.g. CheckFreq, Gemini-style): stage to a *peer node's*
  RAM so a single-node failure recovers without touching durable storage at all; async-flush
  to durable less often as a backstop. Pushes effective `C` toward the copy cost only.

The FinOps read: async+sharded is the difference between `C ≈ 2 min` and `C ≈ 15 s`, which
via `√(2C·MTBF)` and `√(2C/MTBF)` compounds into both a longer safe interval *and* lower
floor waste. TorchTitan's production numbers back this with a real multiplier: Meta
reports **async DCP checkpointing cuts overhead 5–15× versus synchronous** for a Llama
3.1 8B training run (arXiv 2410.06511) — a large, honest reduction, not an "async makes
it free" story. Also budget the **restart** side — detection + reschedule + reload + NCCL
re-init is dead time on *every* GPU (08.5); a 16K-GPU restart that takes 10 min is 2,700
GPU-hours per failure, so restart speed is as financial as write speed.

### Checkpointing is a pipeline, not a bandwidth number — the empirical counter-example

It's tempting to treat `C` as "checkpoint bytes ÷ storage bandwidth" and conclude that
faster storage is always the fix. A rare, concrete, *measured* production counter-example:
a cross-organizational operational study of a 63-node, 504-GPU NVIDIA B200 cluster
(Lablup/SKT/Upstage/NVIDIA Korea/VAST Data, arXiv 2605.09370) traced 523 checkpoint events
end-to-end from GPU VRAM to the NFS durable store and found **checkpoint-save bursts reach
only ~16.0% of the storage's maximum write bandwidth**, and **restart loads reach only
~21.5% of maximum read bandwidth**. The bottleneck the paper identifies isn't the storage
media at all — it's saturation of a 128-slot NFS RPC layer, i.e. **orchestration and
metadata-path concurrency**, not raw throughput. The practical takeaway: before buying
faster storage to shrink `C`, profile *which hop* in the pipeline is actually saturated.
"Checkpoint I/O uses only ~20% of what we provisioned" is a symptom of a coordination
bottleneck, not evidence that more bandwidth would help.

### Why checkpoint cost grows with model size — and what it does to the interval

Checkpoint bytes ≈ model + **optimizer state**. For Adam in mixed precision the optimizer
carries first + second moments plus often an FP32 master copy → roughly **≥12–16 bytes per
parameter** (2-byte weight + 2-byte grad-ish + 4+4 moments + 4 master ≈ 16B/param is a fair
upper bound; ~14 GB for a 7B model, ~5–6 TB for 405B). `C ≈ checkpoint_bytes / write_bandwidth`,
and bytes scale **linearly with parameter count** while your durable write bandwidth is
roughly fixed → **`C` grows with the model**. Concretely, a 405B model has roughly 58× the
parameters of a 7B model, so at fixed effective bandwidth its raw `C` is on the order of
**~100×** larger once you account for the higher-precision master-weight overhead
frontier labs carry at that scale.

Feed that back into Young/Daly: `τ_opt = √(2·C·MTBF)` grows as `√C`, so bigger models want
*longer* intervals — but the floor waste `√(2C/MTBF)` also grows as `√C`, so bigger models
are **strictly more expensive to protect** at a given MTBF. That is exactly why frontier
runs invest so hard in **sharded + async**: sharding makes the *effective* per-rank `C`
grow much slower than the total state, and async hides most of it, keeping the interval and
the floor waste tolerable even at multi-terabyte checkpoints. DeepSpeed's **Universal
Checkpointing** work (arXiv 2406.18820) pushes this one step further, toward a frontier
concern this lesson only needs to name: it decouples the checkpoint's on-disk structure
from the exact TP/PP/DP split it was saved under, so a run can resume with a *different*
parallelism configuration than it checkpointed with — useful when elastic scaling (08.5)
changes world size across a restart and the old shard boundaries no longer line up cleanly.

## Perspectives

**FinOps/formula view.** Young/Daly turns two measured numbers — `C` and `MTBF` — into one
policy decision: the checkpoint interval. It's striking that a 50-year-old HPC result is
being rediscovered by ML infra almost verbatim; the "why does an old formula matter here"
answer is that the underlying failure model (memoryless, exponential) hasn't changed, only
the thing that fails has. This is a talking point that separates "I set the checkpoint
interval to a round number" from "I derived it."

**Systems/engineering view.** `C` is not a single number, it's a **pipeline** — stage →
write → durable-store — and each stage has a different bottleneck. The 504-GPU paper's
finding that checkpoint I/O uses only 16–21% of available storage bandwidth in production
is a concrete counter to "just write faster storage fixes it." Often the real bottleneck
is orchestration or metadata-path concurrency, not raw bandwidth — profile before you buy.

**Storage/infra view.** Sharded checkpointing converts checkpoint writing from "funnel
through rank 0" to "every rank writes its own shard in parallel" — it turns a
network/serialization problem into a storage-IOPS/bandwidth problem at the durable store.
That's exactly why storage vendors care about this space: VAST Data is a co-author on the
504-GPU operational study above, because "many GPUs all durable-writing at once" is now a
first-class storage-system design problem, not an afterthought.

**Model-scale view.** Checkpoint bytes scale linearly with parameters — roughly
12–16 B/param for Adam mixed precision — so `C` for a 405B model is on the order of ~100×
`C` for a 7B model at fixed bandwidth. That arithmetic, not taste, is why frontier labs
invest disproportionately in async+sharded checkpointing: at multi-terabyte scale, a naive
synchronous checkpoint would make Young/Daly's optimal interval financially unworkable.

## Real-world use cases

- **PyTorch/Meta — "Distributed Checkpoint: Efficient checkpointing in large-scale jobs"** —
  <https://pytorch.org/blog/distributed-checkpoint-efficient-checkpointing-in-large-scale-jobs/> —
  official first-party account of production `torch.distributed.checkpoint` (DCP) at Meta
  scale, including the GIL-contention issue found in the thread-based async design and the
  process-based fix. Directly describes the "how you lower C" mechanics above.
- **Meta — TorchTitan** — <https://arxiv.org/abs/2410.06511> — reports async DCP
  checkpointing reduces overhead **5–15×** vs synchronous for Llama 3.1 8B — the concrete
  citable multiplier behind the "2 min → 15 s" style example in the worked section below.
- **Meta — OPT-175B logbook** —
  <https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles> —
  a real, warts-and-all account of checkpoint-driven recovery decisions made during a live
  992-GPU run; read the logbook PDF for the texture of what an actual incident-driven
  checkpoint cadence looks like, not just the theory.
- **"From Detection to Recovery: Operational Analysis on LLM Pre-training with 504 GPUs"**
  (Lablup/SKT/Upstage/NVIDIA Korea/VAST Data) — <https://arxiv.org/abs/2605.09370> — a
  cross-organizational empirical study on a 63-node B200 cluster; reports checkpoint-save
  bursts reach only ~16.0% of storage's maximum write bandwidth and restart loads reach
  ~21.5% of maximum read bandwidth — a rare, measured account of checkpoint I/O
  inefficiency in a real multi-tenant production cluster.

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

**Now decompose `C` itself**, using the 504-GPU study's ratios as a sanity check on your
own measurement. If your storage is provisioned for, say, 20 GB/s write bandwidth but your
measured checkpoint write only achieves ~16% of that (~3.2 GB/s effective), the naive
read is "our storage is too slow." The correct read, per the operational study, is "profile
the pipeline" — is the GPU→host staging copy the limiter, is it the metadata/RPC layer
coordinating hundreds of concurrent shard writers, or is it actually durable-store
bandwidth? Only after that breakdown do you know whether the fix is more storage bandwidth,
fewer concurrent RPC streams, bigger shard chunks, or a different async design. Buying
faster storage against the wrong bottleneck buys nothing.

## Practice

Feeds the **Survive-a-failure lab** deliverable ([`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md)). Extend the DDP job from lesson 08.1 on your
**2 rented GPUs**.

1. Add **save/resume**: checkpoint model + optimizer + RNG + step counter (`torch.save`,
   or `torch.distributed.checkpoint` if you're sharded). Resume must be *exact* — reload,
   confirm the loss curve continues without a discontinuity, not a fresh start.
2. **Measure `C`**: time the checkpoint write (wall-clock seconds), median of ≥5 saves.
   Measure it **twice** — synchronous, then async (background thread or process doing the
   durable write) — and record both. Report checkpoint size in bytes so you can see the
   bytes/param → seconds relationship.
3. **Kill and resume**: `kill -9` a rank mid-run, restart from the last checkpoint, and
   confirm correctness. Note the *restart* wall-time (reload + NCCL re-init) separately from `C`
   — these are two different line items in a true recovery-time estimate; don't conflate them.
4. **Compute `τ_opt`** for a stated **MTBF = 3 h** using your measured `C`
   (`τ_opt = √(2·C·MTBF)`), and report the resulting waste `√(2C/MTBF)`.

**Acceptance:** a **measured checkpoint cost `C`** (sync and async) **+ a computed optimal
interval `τ_opt` for MTBF = 3 h**, with the waste fraction, committed under
`../practice/survive-a-failure/`. One paragraph: how much did async buy you, and what's the
restart cost you'd multiply across 16K GPUs.

## Common pitfalls

- **"Checkpoint more often to be safer."** Past `τ_opt`, more frequent checkpointing
  *increases* total waste — the `C/τ` overhead term dominates. "More" is not strictly
  "safer" past the Young/Daly optimum; it's a straightforward trade you can compute, not a
  hunch to round up on.
- **"Async checkpointing means checkpointing is now free."** Async only hides the
  *durable write*. The GPU→pinned-CPU staging copy still blocks training — that's the `C`
  that survives in the formula. It's a large reduction (TorchTitan's real, honest 5–15×),
  not an elimination; treating it as free will make your τ_opt calculation optimistic.
- **"Checkpoint write bandwidth is the bottleneck."** The 504-GPU empirical study found
  checkpoint I/O uses only 16–21% of available storage bandwidth in production — the real
  bottleneck is often orchestration/staging/metadata overhead, not raw storage throughput.
  "Buy faster storage" is not automatically the fix; profile the pipeline stage by stage
  first.
- **"A bigger model just needs a proportionally longer checkpoint interval and that's
  fine."** `τ_opt` does grow as `√C`, and `C` grows linearly with model size, but the
  *floor waste* also grows as `√C`. Bigger models are strictly more expensive to protect
  at a given MTBF — which is exactly why sharded+async isn't optional at frontier scale,
  it's arithmetically forced.
- **"Resume from checkpoint is just loading the file."** A full resume also pays NCCL
  re-initialization and world re-formation time — 08.5's territory. Checkpoint *reload*
  cost and *restart* cost are separate line items that both belong in a true
  recovery-time estimate; conflating them undercounts the real pause, and undercounting
  it undercounts the dollar figure in 08.8's capstone.

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
write to a background thread or process; only the fast GPU→pinned-CPU staging copy blocks
training, so `C` drops from the full write time (minutes) to seconds — TorchTitan reports
a real 5–15× reduction, not "free." **Sharded/distributed** has each rank write only its
own shard in parallel, so write bandwidth scales with world size instead of bottlenecking
through rank 0 (and it's mandatory when the full state won't fit on one GPU). Because both
`τ_opt = √(2C·MTBF)` and the floor waste `√(2C/MTBF)` scale as `√C`, halving `C` cuts
unavoidable waste by ~29% — the concrete mechanism behind Llama 3's >90% effective time on
a ~3-hr-MTBF cluster.

**(c) Why does checkpoint cost grow with model size, and how does that change the optimal
interval?**
**Answer:** Checkpoint bytes = model weights + optimizer state (Adam mixed-precision ≈
12–16 bytes/param), which scales **linearly with parameter count**, while durable write
bandwidth is roughly fixed — so `C ≈ bytes/bandwidth` grows with the model (~14 GB for 7B,
multi-TB for 405B, roughly a ~100× gap between them). In Young/Daly, `τ_opt = √(2·C·MTBF)`
grows as `√C`, so bigger models want *longer* intervals — but the floor waste
`√(2C/MTBF)` also grows as `√C`, so they're strictly more expensive to protect at a given
MTBF. That's why frontier runs lean on **sharded** (per-rank `C` grows far slower than
total state) and **async** (hides most of the write), keeping both the interval and the
waste tolerable at terabyte scale.

**(d) A checkpoint is only using 20% of your provisioned storage bandwidth. Is faster
storage the fix?**
**Answer:** Not necessarily — check the pipeline before buying hardware. A 504-GPU
operational study measured exactly this pattern (checkpoint saves at ~16% of max write
bandwidth, restarts at ~21.5% of max read bandwidth) and traced the actual bottleneck to
saturation of the metadata/RPC coordination layer handling hundreds of concurrent shard
writers, not the storage media itself. Faster storage against a coordination bottleneck
buys nothing; the fix there is fewer/larger concurrent write streams or better metadata
batching. Profile which of the pipeline's stages (GPU→host staging, metadata collective,
durable write) is actually saturated before spending on bandwidth.

**(e) You measure checkpoint reload at 30 seconds. Is that the full cost of a restart to
plug into a recovery-time estimate?**
**Answer:** No — checkpoint reload is only one line item. A full restart also pays
**re-rendezvous and NCCL re-initialization** (08.5's territory: agents detecting the exit,
re-forming the process group, rebuilding communicators for the new world size) on top of
the checkpoint reload itself. Conflating "time to load the checkpoint file" with "time
until training resumes" undercounts the real pause — and at 16K GPUs, undercounting even a
minute of restart time is thousands of GPU-hours miscosted in a cost-per-successful-run
figure.

## Connections & what's next

This lesson gave you the two-lever arithmetic (`C`, `τ`) behind the "failure overhead"
term 08.3 named but didn't quantify — the formula that turns a checkpoint engineering
choice into a defensible percentage of effective wall-clock. 08.5 is the natural next
step: it takes the checkpoint this lesson produces as a *given* (the durable floor state)
and builds the loop around it — health signal → drain → re-rendezvous → resume-from-
checkpoint — that actually triggers the restart and re-forms the world. The restart-cost
side this lesson flags but doesn't fully price (NCCL re-init, world re-formation) is 08.5's
subject in full. Together, 08.3's comms term and 08.4/08.5's failure term are the two
biggest inputs to 08.8's cost-per-successful-run capstone.

## References & further reading

**Primary sources**
1. **Llama 3 paper, §3.3 "Infrastructure, Scaling, and Efficiency"** —
   <https://arxiv.org/abs/2407.21783> — the anchor. 16K H100s, 466 interruptions (419
   unexpected) in 54 days, >90% effective training time, and the checkpoint/restart
   engineering behind it. Read it deeply; it's the module's spine.
2. **Young/Daly optimal-checkpoint-interval formula** — J. Daly, *"A higher order estimate
   of the optimum checkpoint interval for restart dumps,"* Future Generation Computer
   Systems 22 (2006), 303–312, building on J. W. Young (1974); overview/derivation:
   A. Benoit et al., *"Checkpointing à la Young/Daly: An Overview"* —
   <https://icl.utk.edu/files/publications/2022/icl-utk-1569-2022.pdf> — the derivation of
   `√(2μC)` and when the second-order correction matters.
3. **PyTorch Distributed Checkpoint (DCP) docs** —
   <https://pytorch.org/docs/stable/distributed.checkpoint.html> — the sharded + async APIs
   you'll actually call in the practice task.

**Real-world engineering blogs**
4. **PyTorch/Meta — "Distributed Checkpoint: Efficient checkpointing in large-scale
   jobs"** — <https://pytorch.org/blog/distributed-checkpoint-efficient-checkpointing-in-large-scale-jobs/> —
   the async-staging architecture and the GIL-contention lesson learned in production.
5. **Meta — TorchTitan** — <https://arxiv.org/abs/2410.06511> — the 5–15× async
   checkpointing overhead reduction on Llama 3.1 8B, plus the broader 4D-parallelism
   production training stack it sits in.
6. **Meta — OPT-175B logbook** —
   <https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles> —
   ground-truth account of checkpoint-driven recovery decisions during a live run.

**Deeper dives**
7. **"From Detection to Recovery: Operational Analysis on LLM Pre-training with 504
   GPUs"** — <https://arxiv.org/abs/2605.09370> — read in full for the checkpoint
   I/O-bandwidth-utilization numbers (16.0% write / 21.5% read), a rare empirical data
   point on where checkpoint cost actually goes.
8. **"DataStates-LLM: Lazy Asynchronous Checkpointing for Large Language Models"** —
   <https://arxiv.org/abs/2406.10707> — a research treatment of lazy/async checkpointing
   that goes beyond PyTorch's DCP design.
9. **"Universal Checkpointing" (DeepSpeed)** — <https://arxiv.org/abs/2406.18820> —
   checkpoint interoperability across different parallelism configurations (resuming with
   a different TP/PP/DP split than saved with) — a frontier concern worth knowing exists.
