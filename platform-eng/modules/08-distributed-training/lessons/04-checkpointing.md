---
lesson: "08.4"
title: "Checkpointing"
module: "08"
concept: "checkpointing"
status: not-started
est_time: "9h"
prev: "03-communication-bottleneck.md"
next: "05-failure-and-elasticity.md"
artifacts: []
sources: 13
---

# 08.4 · Checkpointing

> **Concept.** On a cluster that fails every few hours, effective throughput is set by how cheaply you can save state and restart — the Young/Daly optimal interval turns MTBF and checkpoint cost into a number.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

08.3 named three causes of lost throughput — communication, data starvation, and **failure overhead** — and gave the first two a model each. It also warned that a long-window MFU silently absorbs the third. This lesson is that third cause, fully worked: how much wall-clock failure overhead actually costs, and the two levers you own to shrink it.

It is the module's **throughput lever**, the single decision with the largest and most directly defensible dollar impact in this module, which is why it gets the extra hours. The material divides cleanly: **this lesson owns the checkpoint** — what is in it, how it is written, how long that takes, and how often to do it. **08.5 owns the loop around it** — what detects the failure, what drains the node, what re-forms the world, and what reloads the file this lesson produced.

It also inherits directly from 08.1 and 08.2. The bytes you write are exactly the model-state bytes 08.1 accounted for, minus the transient ones. And a distributed checkpoint is itself a collective operation: it has a metadata all-gather, it is a barrier, and it therefore has the same "one slow participant sets the pace" property 08.2 spent a whole lesson on.

## Why this matters

Llama 3 405B trained on **16,384 H100s for 54 days** and hit **466 job interruptions — 419 of them unexpected**, or roughly **one every three hours**; about **78% were traced to hardware**. At that failure rate "how fast is the model" is the wrong question. The operative one is "**how much of the 54 days was effective**," and Meta reports **>90% effective training time**.

The entire delta between that 90% and the ~50% a naive setup would deliver is checkpoint and restart engineering. Get the interval wrong in either direction and you burn money: too frequent and the cluster spends its time writing state instead of training; too rare and every failure rolls back hours of 16,384-GPU compute. **There is a one-line formula that picks the interval, it is fifty years old, and most teams do not use it.** That is your edge, and it is a standing interview probe in this module's checkpoint.

Put the arithmetic in front of you. At a nominal $3/GPU-hour, that fleet costs **$49,152 per hour**. Each percentage point of effective training time over 54 days is worth **$637,000**. Moving from 85% to 94% effective — which §6 shows is exactly what dropping checkpoint cost from two minutes to fifteen seconds buys — is **$5.7 M on one run**. (GPU counts and $/GPU-hour are dated snapshots; the arithmetic outlives any specific price.)

## What's new here (calibration)

08.1 gave you the model-state accounting — `16Ψ` bytes of parameters, gradients, master weights and Adam moments. 08.2 gave you collectives, barriers, and the fact that a distributed operation runs at the pace of its slowest participant. 08.3 gave you the step-time model and MFU, and named failure overhead without quantifying it. Module 05 gave you the health signals; **MTBF is an input here, not something derived — 08.5 owns the signal-to-MTBF story.**

Genuinely new in this lesson:

- **What is actually in a checkpoint**, derived from 08.1's accounting — including the term everyone forgets, which causes silent, invisible corruption rather than a crash.
- **The write path as a four-stage pipeline** with a bandwidth at each stage, so "checkpoint cost" stops being one number and becomes something you can profile and attack stage by stage.
- **PyTorch Distributed Checkpoint's actual mechanics** — the planner, the metadata collective, the on-disk layout, and the `FileSystemWriter` defaults that decide your write bandwidth (one of which is a genuine footgun).
- **Async checkpointing's real boundary**: exactly which part still blocks training and which part does not, plus the thread-versus-process distinction and the process-group configuration that async silently requires.
- **Young/Daly derived**, not asserted — including why the two waste terms are equal at the optimum, what the second-order correction is for, and when it matters.
- **Expected time-to-failure for `N` GPUs derived** from the exponential model, cross-checked against the Llama 3 numbers, and folded into a **goodput** expression that includes restart cost as a separate term.
- **Why restart cost does not change the optimal interval but does set a floor on waste** — which is the reason 08.5 exists as a separate lesson.

The mental model to carry: a bathtub of wasted time with two taps. **Overhead** is time spent writing checkpoints; **rework** is time recomputing progress lost since the last one. Checkpoint more often and overhead rises while rework falls. The optimum balances them, and Young/Daly names it.

## Core concepts

### 1. What is actually in a checkpoint

**The requirement, stated precisely:** after a restart the job must produce **bit-identical subsequent behaviour** to the run that would have continued. Not "close enough" — a resumed run that diverges silently is worse than a crash, because you find out weeks later from a loss curve.

Start from 08.1's `16Ψ` accounting and ask, term by term, whether it must be persisted:

| Term | Bytes/param | Persist? | Why |
|---|---:|---|---|
| bf16 parameters (compute copy) | 2 | **usually** | derivable by downcasting the fp32 master, but saved for fast load and for export |
| gradients | 2–4 | **no** | transient within a step; recomputed on resume |
| fp32 master parameters | 4 | **yes** | the authoritative weights; losing them loses precision permanently |
| Adam first moment `m` | 4 | **yes** | dropping it restarts momentum from zero → a visible loss spike |
| Adam second moment `v` | 4 | **yes** | dropping it makes the first steps after resume effectively huge |
| **model state subtotal** | **~14** | | |

So **checkpoint bytes ≈ 14Ψ**, not 16Ψ: gradients are the one term you can throw away. If your framework can reconstruct the bf16 copy on load, it is 12Ψ.

**Cross-check against a real checkpoint.** BLOOM-176B's checkpoints were 2.3 TB (`stas00/ml-engineering`, from a BLOOM engineer). `2.3e12 / 176e9 = 13.1 bytes per parameter` — landing exactly between the 12Ψ and 14Ψ bounds above. The accounting is right.

**Scale it:**

```
   7B    :  14 × 7e9    =   98 GB
   70B   :  14 × 70e9   =  980 GB
   176B  :  14 × 176e9  = 2.46 TB   (measured: 2.3 TB — see above)
   405B  :  14 × 405e9  = 5.67 TB
```

**Now the terms that are not weights, and that people forget.** A checkpoint that contains only model state resumes *approximately*, which is the worst failure mode in this lesson because nothing errors:

| Also required | What breaks without it |
|---|---|
| **step / iteration counter** | the LR schedule restarts, and so does any warmup |
| **LR scheduler state** | same, but subtler — a cosine schedule silently resets its phase |
| **data-loader position** | you re-train on data you already saw and skip data you have not; on a single-epoch pre-training run this quietly changes the data distribution |
| **RNG state** (CPU, CUDA, and per-rank) | dropout masks and any data augmentation diverge from the counterfactual run; also breaks reproducibility |
| **gradient-scaler state** (fp16 runs) | the loss scale restarts from its initial value and you eat a burst of skipped steps while it re-converges |
| **the parallelism layout it was saved under** | a checkpoint sharded for TP=8/PP=4 cannot be loaded into TP=4/PP=8 without resharding |

**The platform-side tell for a bad resume** is a loss curve with a small discontinuity at the restart point that recovers over a few hundred steps. Momentum reset produces a visible bump; data-loader position loss produces nothing visible at all and is only detectable by auditing the sampler. That is why the acceptance criterion in the Practice section is "the loss curve continues without a discontinuity," not "it loaded without error."

### 2. The write path is a pipeline, not a bandwidth number

The most expensive mistake in this area is treating `C` as `checkpoint_bytes / storage_bandwidth` and concluding that faster storage is the fix. `C` is a four-stage pipeline, and the stages have wildly different bandwidths.

```
  THE CHECKPOINT WRITE PATH — where the seconds actually go
  ═══════════════════════════════════════════════════════════════════════════

   ┌─ STAGE 1 ─ gather / plan ────────────────────────────────────────────┐
   │  build the state_dict; for a sharded model, work out who owns what.  │
   │  DCP: create_local_plan on every rank → ALL-GATHER of the plans →    │
   │       create_global_plan on the coordinator → scatter back.          │
   │  ▸ COST: one small collective. Bytes ≈ nothing. But it is a BARRIER  │
   │    (08.2), so it runs at the pace of the SLOWEST rank.               │
   │  ▸ FSDP1 additionally needed all-gathers of the PARAMETERS to build  │
   │    a full state dict; FSDP2/DTensor gives sharded state dicts with   │
   │    no parameter communication at all. This is why 08.1 said the      │
   │    FSDP2 migration is a checkpointing win.                           │
   └──────────────────────────────────────────────────────────────────────┘
                                    │
   ┌─ STAGE 2 ─ HBM → pinned host memory ────────────────────────────────┐
   │  cudaMemcpyAsync device→host into page-locked ("pinned") memory.     │
   │  ▸ BANDWIDTH: the PCIe link. Measured on 8×H200 (PCIe 5 x16):        │
   │       55.06 GB/s device→host, 87% of the 63 GB/s spec.               │
   │       On Grace-Blackwell the same copy rides NVLink-C2C and is an    │
   │       order of magnitude faster — same code, different fabric.       │
   │  ▸ THIS IS THE STAGE THAT BLOCKS TRAINING, and the only one that     │
   │    async checkpointing cannot eliminate.                             │
   │  ▸ 98 GB (7B model) / 55 GB/s / 8 GPUs  ≈  0.22 s per GPU            │
   └──────────────────────────────────────────────────────────────────────┘
                                    │  ◀── async: TRAINING RESUMES HERE
   ┌─ STAGE 3 ─ serialise + write ───────────────────────────────────────┐
   │  each rank writes its own file(s): __{rank}_{n}.distcp               │
   │  ▸ BANDWIDTH: whatever the client can push. DCP's FileSystemWriter   │
   │    defaults to thread_count = 1 — ONE I/O thread per rank.           │
   │  ▸ This is where the "bandwidth paradox" lives (§7): a 63-node B200  │
   │    cluster reached only ~16.0% of its storage's 250 GB/s write       │
   │    ceiling, bottlenecked on a 128-slot NFS RPC layer, not media.     │
   └──────────────────────────────────────────────────────────────────────┘
                                    │
   ┌─ STAGE 4 ─ durability ──────────────────────────────────────────────┐
   │  fsync each file (FileSystemWriter: sync_files = True by default),   │
   │  then a BARRIER, then rank 0 writes the global .metadata file.       │
   │  ▸ The metadata write is LAST ON PURPOSE: its presence is what makes │
   │    the checkpoint valid. A crash mid-write leaves .distcp files with │
   │    no .metadata, and the loader correctly refuses them.              │
   │  ▸ COST: a barrier plus one small write. Small — unless one rank's   │
   │    fsync is slow, in which case everyone waits.                      │
   └──────────────────────────────────────────────────────────────────────┘

  ── SYNCHRONOUS ──────────────────────────────────────────────────────────
   GPU  │██ step ██│░░░░░░░░ IDLE: stages 1-4 ░░░░░░░░│██ step ██│
                    └──────────── C = ALL OF IT ──────┘

  ── ASYNCHRONOUS ─────────────────────────────────────────────────────────
   GPU  │██ step ██│▓ 1-2 ▓│██ step ██│██ step ██│██ step ██│██ step ██│
   bg   │          │       └── stages 3-4, on a background thread ──────▶│
                    └─┬───┘
              C = STAGES 1-2 ONLY.  Everything right of the arrow is
              overlapped with training — the same trick as 08.3 §6,
              applied to storage instead of the fabric.
```

**Read the two timelines together and the entire lever is visible.** Async does not make checkpointing free; it moves the boundary of `C` from "the whole pipeline" to "the gather plus the device-to-host copy." §4 quantifies how much that is worth.

### 3. Sharded (distributed) checkpointing — the mechanics

**The problem it solves.** Two of them, actually. First, once the model is sharded across ranks (FSDP, ZeRO, TP), **no single rank can hold the full state** — rank 0 does not have the memory to gather 5.67 TB. Second, funnelling every byte through one rank makes write bandwidth a constant regardless of how many nodes you own.

**How DCP works.** `torch.distributed.checkpoint` (DCP) is built around a planner/storage split:

1. Every rank calls `dcp.save(state_dict, checkpoint_id=...)`. Each rank's `state_dict` contains only its own shards — `DTensor`s that know their global shape and their local slice.
2. Each rank produces a **local save plan**: a list of write items describing what it owns.
3. Plans are **all-gathered**; the coordinator (rank 0) deduplicates and produces a **global plan**, which is scattered back. This is the metadata collective, and it is the barrier from stage 1.
4. Each rank writes its own items to files named `__{rank}_{n}.distcp`, then everything is fsynced, then rank 0 writes the global `.metadata`.

**Write bandwidth now scales with world size** instead of being pinned to rank 0's link. That is the whole point, and it is why sharded checkpointing is mandatory at frontier scale rather than merely nice.

**On load, DCP operates *in place*.** Unlike `torch.load`, which materialises tensors and hands them to you, `dcp.load` requires that the model already allocated its storage and reads directly into it. This has a consequence worth knowing: **DCP can reshard on load.** Because `.metadata` records each tensor's global shape and each file's slice, a checkpoint saved under one degree of sharding can be read under a different one — the loader computes which byte ranges of which files satisfy its own slice. That is what makes elastic restarts at a different world size (08.5) possible at all.

**The `FileSystemWriter` defaults, and the footgun.** Verified in `torch/distributed/checkpoint/filesystem.py`:

| Parameter | Default | What it means |
|---|---|---|
| `single_file_per_rank` | `True` | one `.distcp` file per rank rather than per tensor — far fewer files, much kinder to a metadata server |
| `sync_files` | `True` | fsync before declaring success. **Turning it off makes the checkpoint non-durable on a crash** — the docs say so explicitly |
| `thread_count` | **`1`** | ← **the footgun.** One I/O thread per rank by default. On storage that rewards concurrency, raising this is often the single highest-return checkpoint tuning knob |
| `per_thread_copy_ahead` | `10_000_000` (10 MB) | how many bytes each thread stages ahead from the GPU |

On-disk, a 4-rank checkpoint therefore looks like:

```console
$ ls -la /ckpt/step-1000/
-rw-r--r--  1 u u  24746393600  __0_0.distcp
-rw-r--r--  1 u u  24746393600  __1_0.distcp
-rw-r--r--  1 u u  24746393600  __2_0.distcp
-rw-r--r--  1 u u  24746393600  __3_0.distcp
-rw-r--r--  1 u u        18432  .metadata
```

**`.metadata` is the commit record.** It is written last, and its absence is how a torn checkpoint is detected. Any cleanup policy you write ("keep the latest 3") must treat a directory without `.metadata` as invalid, not as the newest checkpoint.

**`torch.save` is still fine — at small scale.** For a job whose full state fits in one rank's host memory, `torch.save` on rank 0 is simpler and perfectly correct. The crossover is not a model size, it is the moment the state stops fitting or the moment rank-0-only write bandwidth starts to dominate `C`. Know which regime you are in.

### 4. Async checkpointing — what still blocks, and what does not

`dcp.async_save` has the same signature as `dcp.save` plus a staging configuration, and returns a future. The **staging** step (stages 1–2) runs inline and blocks; the **write** (stages 3–4) runs in the background while training continues.

The default staging behaviour, from `StagingOptions` in `torch/distributed/checkpoint/staging.py`:

| Option | Default | Effect |
|---|---|---|
| `use_pinned_memory` | `True` | page-locked host buffers, so the device→host copy can be a true DMA rather than a staged copy |
| `use_shared_memory` | `True` | put the staging buffer in shared memory, so a separate *process* can write it without another copy |
| `use_async_staging` | `True` | run staging itself on a background thread pool |
| `use_non_blocking_copy` | `True` | `cudaMemcpyAsync` plus stream synchronisation, so the CPU keeps working during the transfer |

**How much this is worth, measured.** PyTorch's engineering write-up on distributed asynchronous checkpointing reports, for a **7B model, checkpoint "down time" falling from an average of 148.8 s to 6.3 s — a 23.6× reduction** — with 7B–13B models generally landing at 6–14 s of visible downtime, and characterises the overall effect as a 10–20× reduction in effective checkpointing time. Independently, torchtitan's checkpointing work reports the same order of improvement.

**The thread-versus-process distinction, and why it exists.** `async_save` takes `async_checkpointer_type=AsyncCheckpointerType.THREAD` (the default) or `.PROCESS`. The thread-based implementation runs serialisation and I/O on a background Python thread — which contends for the **GIL** with the training loop's own Python work. In production this quietly ate part of the win: the training loop stalled during the background thread's serialisation and metadata phases. The fix was the process-based executor: a separate process with its own interpreter does the serialisation and I/O, communicating with the trainer through **shared memory** so the staged tensors are not copied again.

**The generalisable lesson: "async" is a design goal, not a guarantee.** Whether you actually get overlap depends on the implementation, and the way to find out is to measure step time *during* a background checkpoint versus outside one. If steps slow down while the write is in flight, your async is not async.

**The configuration gotcha that will bite you first.** `async_save` asserts, verbatim from the source:

```
A CPU backend must be enabled for async save;
try initializing process group with 'cpu:gloo,cuda:nccl'
```

The background writer needs to run collectives (the metadata exchange) from the CPU side while the GPU stream is busy, so the process group must have a CPU backend. Initialise with `dist.init_process_group("cpu:gloo,cuda:nccl")`, not plain `"nccl"`. This is a hard error, not a silent degradation — but it appears the first time you enable async, at which point people conclude async is broken.

**Three more knobs you will meet in real configs**, from torchtitan's `CheckpointManager.Config`:

- `async_mode` ∈ `disabled` | `async` | `async_with_pinned_mem` — the third pre-allocates and reuses the pinned staging buffer across checkpoints, avoiding a large page-locked allocation every time.
- `keep_latest_k` — bounded retention. Non-negotiable in practice: at 5.67 TB per checkpoint and one every ten minutes, unbounded retention fills any filesystem in a day.
- `last_save_model_only` and `export_dtype` — the final artefact usually does not need optimizer state, so the last save can be `2Ψ` instead of `14Ψ`.

**The rule you must not break:** with async, **the previous checkpoint is not durable until its future resolves.** Never delete checkpoint `n−1` before checkpoint `n`'s write has completed and its `.metadata` exists. Retention must key off completion, not off initiation. The complementary practice from BLOOM's operators: keep the **two** most recent checkpoints locally, not one, because the newest may be torn or still in flight.

### 5. The two waste terms, and Young/Daly derived

**Set up the model.** Over a long run, wall-clock divides into forward progress and waste. Let:

- `C` = checkpoint cost in seconds — **the blocking part**, so with async this is stages 1–2 only;
- `τ` = the interval between checkpoints, in seconds of *useful work*;
- `M` = MTBF of the job, in seconds;
- `R` = restart cost per failure: detect + drain + re-rendezvous + NCCL init + reload (08.5's subject).

**Term one: overhead.** You pay `C` seconds of blocking every `τ` seconds of progress, so the fraction of wall-clock spent checkpointing is `C/τ`. It falls as `1/τ`.

**Term two: rework.** When a failure hits, you lose everything since the last checkpoint. **Where in the interval does the failure land?** Under the standard assumption — failures are a Poisson process, i.e. the time to failure is exponential and therefore *memoryless* — the failure instant is uniformly distributed over the interval. So the expected lost work is `τ/2`. Failures arrive at rate `1/M`, giving a rework fraction of `(τ/2)/M = τ/(2M)`. It rises linearly in `τ`.

**Term three: restart.** Each failure also costs `R` seconds that have nothing to do with `τ`: `R/M`.

```
                    C         τ         R
    waste(τ)  =    ───   +   ────   +  ───
                    τ         2M         M
                    │         │          │
              overhead    rework     restart
              falls 1/τ   rises τ    constant in τ
```

**Minimise.** Differentiate with respect to `τ` and set to zero:

```
    d/dτ [ C/τ + τ/(2M) + R/M ]  =  −C/τ²  +  1/(2M)  =  0

                                       τ²  =  2CM

                            ┌──────────────────────┐
                    τ_opt = │     √( 2 · C · M )   │
                            └──────────────────────┘
```

This is the **Young (1974) / Daly (2006)** first-order optimal checkpoint interval, the same `√(2μC)` that HPC has used for fifty years. Note that `R` differentiates away: **restart cost does not change the optimal interval.** It only adds a constant floor to the waste. That is a genuinely useful separation — it means you can tune the interval without knowing your restart time, but you cannot reach 90% goodput without also attacking it.

**The waste at the optimum is unusually clean.** Substitute `τ_opt = √(2CM)`:

```
    overhead  =  C / √(2CM)      =  √( C / (2M) )
    rework    =  √(2CM) / (2M)   =  √( C / (2M) )        ← THE SAME
    ─────────────────────────────────────────────────────
    waste(τ_opt)  =  2·√( C/(2M) ) + R/M  =  √( 2C/M ) + R/M
```

**At the optimum the two tunable terms are exactly equal.** That is a free correctness check on any interval you are handed: compute both and see whether they match. If overhead is much larger than rework you are checkpointing too often; if much smaller, too rarely.

```
  THE WASTE CURVE — why both directions hurt
  ═══════════════════════════════════════════════════════════════════════════
  M = 3 h = 180 min,  C = 2 min,  R excluded (it is flat in τ)

  waste
   40% ┤ *
       │  *
   30% ┤   *                                                        ·
       │    *                                                  ·
   20% ┤      *                                          ·
       │        *  *                              ·
  14.9%┤─────────── *  *  ●  *  *  ·  ·  ·  ────────────────────────────
       │              ╲       ╱                    ·
   10% ┤   overhead C/τ ╲   ╱ rework τ/2M
       │                 ╲ ╱
    0% ┼──────┬──────┬───●──┬──────┬──────┬──────┬──────┬──────┬─────▶ τ
       0      10    20  26.8 40     60     80    100    120   (minutes)
                            ▲
                        τ_opt = √(2·2·180) = √720 = 26.8 min
                        overhead = 2/26.8   = 7.5%   ┐
                        rework   = 26.8/360 = 7.4%   ┘ equal, as derived
                        total    = √(4/180) = 14.9%

  τ =  5 min : overhead 40.0% + rework  1.4%  =  41.4%   ✖ writing, not training
  τ = 26.8   : overhead  7.5% + rework  7.4%  =  14.9%   ✔ the minimum
  τ = 90 min : overhead  2.2% + rework 25.0%  =  27.2%   ✖ losing hours per failure

  THE CURVE IS FLAT NEAR THE MINIMUM — anywhere in 20–35 min costs under
  15.6%. You do not need τ to three decimal places; you need to not be at
  5 or at 90. That flatness is why a one-line first-order formula is enough.
```

**The second-order correction, and when to care.** The derivation above assumes no failure occurs *during* a checkpoint write and that `C ≪ M`. Daly's higher-order estimate relaxes this; in the common regime `C < 2M` it takes the form

```
    τ_Daly  =  √(2CM) · [ 1 + (1/3)·√( C/(2M) ) + (1/9)·( C/(2M) ) ]  −  C
```

with the degenerate case `τ = M` when `C ≥ 2M` — which is the formula's way of saying "your checkpoint is so expensive relative to your MTBF that you should give up on regular checkpointing and rethink." For the worked case, `C/(2M) = 2/360 = 0.00556`, so the bracket is `1 + 0.0248 + 0.0006 = 1.0254` and `τ_Daly = 26.83 × 1.0254 − 2 = 25.5` min against the first-order 26.8. **A 5% difference in `τ`, on a curve whose waste differs by under 0.1 percentage points between the two (14.92% at both). Use the one-liner.** The correction earns its keep only when `C` is a meaningful fraction of `M` — which, for a well-tuned async checkpoint, it is not.

### 6. Expected time to failure for `N` GPUs, and goodput

Young/Daly needs `M`, the job's MTBF. Where does that come from?

**Derive it.** Model each GPU as failing independently with a constant hazard rate `λ = 1/m`, where `m` is the per-GPU MTBF. The probability that one GPU survives to time `t` is `e^{−λt}`. For a synchronous job, **any** GPU failing kills the whole job, so the job survives only if all `N` do:

```
    P_job(survive to t)  =  (e^{−λt})^N  =  e^{−Nλt}
```

That is again an exponential, with rate `Nλ`. So:

```
                    ┌──────────────────────────┐
        M_job   =   │   m_gpu  /  N            │
                    └──────────────────────────┘
```

**The job's MTBF falls linearly with GPU count.** This is the reason large runs are qualitatively different from small ones, and it needs no more machinery than "any one failing kills it."

**Cross-check against Llama 3, both directions.** The run reports 419 unexpected interruptions over 54 days on 16,384 GPUs:

```
    54 days                       = 1,296 hours
    M_job = 1296 / 419            = 3.09 hours          ← "one every ~3 hours" ✓
    implied m_gpu = 3.09 × 16,384 = 50,600 GPU-hours    ≈ 5.8 years per GPU
```

**A 5.8-year per-accelerator MTBF is an entirely ordinary reliability figure for server hardware.** Nothing about Meta's fleet was unusually fragile — the drama comes entirely from the `N` in the denominator. Turn it around and the number becomes a planning tool:

| Job size | `M_job = 50,600 / N` | Reading |
|---:|---:|---|
| 8 GPUs | 6,325 h ≈ 264 days | you will probably never see a hardware failure |
| 64 | 791 h ≈ 33 days | one per month; manual restart is fine |
| 512 | 99 h ≈ 4.1 days | weekly; you want automation |
| 2,048 | 24.7 h | daily; checkpointing is now load-bearing |
| 8,192 | 6.2 h | twice a day |
| 16,384 | 3.1 h | the Llama 3 regime |
| 100,000 | 0.5 h | every 30 minutes — the regime that motivates 08.5's partial-restart architectures |

**Now assemble goodput.** Goodput is the fraction of wall-clock converted into forward progress:

```
    goodput  =  1 − waste  =  1 − [ √(2C/M) + R/M ]
```

Evaluate it, holding `M = 3 h = 180 min` — the Llama 3 regime — and varying the two things you actually control:

| `C` (blocking) | `R` (restart) | `τ_opt` | `√(2C/M)` | `R/M` | goodput |
|---|---|---:|---:|---:|---:|
| 2 min (sync) | 10 min | 26.8 min | 14.9% | 5.6% | **79.5%** |
| 2 min (sync) | 3 min | 26.8 min | 14.9% | 1.7% | 83.4% |
| 15 s (async) | 10 min | 9.5 min | 5.3% | 5.6% | 89.1% |
| **15 s (async)** | **3 min** | **9.5 min** | **5.3%** | **1.7%** | **93.0%** |
| 15 s (async) | 30 s | 9.5 min | 5.3% | 0.3% | 94.4% |

```
  WHERE THE WALL-CLOCK GOES — the same 3-hour-MTBF fleet, four designs
  ═══════════════════════════════════════════════════════════════════════════
  Each bar is 100% of wall-clock.  █ progress  ▒ checkpoint overhead C/τ
                                   ░ rework τ/2M   ▓ restart R/M

  sync C=2m, R=10m   ███████████████████████████████████████░░▒▒▒▒▒▒▒▓▓▓▓▓  79.5%
                                                            └─7.5%─┘└7.4%┘└5.6%┘

  sync C=2m, R=3m    █████████████████████████████████████████░░▒▒▒▒▒▒▒▓▓   83.4%

  async C=15s, R=10m ████████████████████████████████████████████▒▒░░▓▓▓▓▓  89.1%
                                                                 └2.6%┘└5.6%┘
                                                              ▲ C shrank; the
                                                                RESTART term is
                                                                now the big one

  async C=15s, R=3m  ██████████████████████████████████████████████▒▒░░▓    93.0%
                                                                        ▲
                                              Llama-3's ">90% effective" region.
                                              NEITHER LEVER ALONE GETS HERE.

  ── the shape of each lever ───────────────────────────────────────────────
     C : waste ∝ √C     → halving C cuts waste by only 29%  (diminishing)
     R : waste ∝ R      → halving R halves that term         (linear)
     M : waste ∝ 1/√M and 1/M → and M = m_gpu/N, which you do not control
```

**Read the table as a strategy, because it is one.** Fixing checkpoint cost alone (row 1 → row 3) buys 9.6 points. Fixing restart cost alone (row 1 → row 2) buys 3.9. **Doing both gets you to 93%, which is Llama 3's ">90% effective training time" territory — and neither lever alone gets you there.** That is precisely why this module splits the material: 08.4 owns `C`, 08.5 owns `R`, and the headline number needs both.

Note also the shape of the `C` lever: **waste scales as `√C`**, so halving checkpoint cost cuts the unavoidable waste by only `√2 ≈ 29%`, not 50%. Diminishing returns are built in, which is why the table's last row buys so little over the fourth.

### 7. Where checkpoint cost actually goes — the empirical counter-example

It is tempting to compute `C = bytes / storage_bandwidth` and buy faster storage. Here is a measured reason not to.

A cross-organisational operational study of a **63-node, 504-GPU NVIDIA B200 production cluster** (Lablup with SKT, Upstage, NVIDIA Korea and VAST Data) instrumented 55 days of Prometheus data and 73 days of operational logs across 224 multi-node training sessions, and traced **523 checkpoint events** end to end from GPU VRAM to the NFS store. What they found:

- **checkpoint-save bursts reached only ~16.0%** of the storage's maximum write bandwidth (250 GB/s);
- **restart loads reached only ~21.5%** of maximum read bandwidth (700 GB/s);
- the network was almost idle — **1.4–10.4% utilisation of the 200 Gbps RoCE** links;
- the bottleneck they identify is **saturation of a 128-slot NFS RPC layer** — orchestration and metadata-path concurrency, not media throughput.

They also report that this bottleneck **did not appear in 2–4-node tests** and only emerged at 60-node scale, which is the kind of finding you can only get from production instrumentation.

**The operational reading:** "our checkpoint I/O uses only 20% of what we provisioned" is a symptom of a *coordination* bottleneck, not evidence that more bandwidth would help. Before buying storage, profile which stage of §2's pipeline is saturated. The fixes for a coordination bottleneck are different in kind: fewer, larger concurrent write streams (`single_file_per_rank=True`, already the default); more I/O threads per rank (`thread_count`, whose default of 1 is often wrong); batching metadata operations; or moving the durable write off the critical path entirely with async, which converts a latency problem into a background one.

**The other end of the range, for calibration.** BLOOM-176B wrote a **2.3 TB checkpoint in 40 seconds across 384 concurrent processes** on GPFS-over-NVMe — `2.3e12 / 40 = 57.5 GB/s` aggregate, or ~150 MB/s per process. They checkpointed roughly every 3 hours over ~3 months, giving ~720 checkpoints × 40 s = **8 hours of checkpoint overhead across 90 days, or 0.37% of training time**. Their own note is the important part: had the I/O been 5× slower — "not uncommon on the cloud unless one pays for premium IO" — that becomes ~2% of training time. **Same code, same interval, same model: a 5× storage difference is a 5× difference in the overhead term.**

### 8. How checkpoint cost scales with model size — and what that does to the interval

`C ≈ 14Ψ / effective_write_bandwidth`. Bytes scale **linearly** with parameter count; your durable write bandwidth is roughly fixed by the storage system. So `C` grows linearly with the model, and a 405B model's raw checkpoint is `405/7 = 58×` a 7B model's.

Feed that back into Young/Daly and two things happen at once:

```
    τ_opt      =  √(2 C M)     grows as √C   → bigger models want LONGER intervals
    waste_min  =  √(2 C / M)   grows as √C   → and are STRICTLY more expensive
                                               to protect at the same MTBF
```

Both scale as `√C`, so a 58× larger checkpoint means a `√58 = 7.6×` longer interval **and** `7.6×` the floor waste. That second half is the uncomfortable one: at fixed MTBF, protecting a big model simply costs more, and there is no interval that makes it not so.

**Which is why sharding and async are arithmetically forced at frontier scale, not stylistic choices.** Sharding makes each rank's `C` grow far more slowly than total state, because write bandwidth grows with world size at the same time bytes do. Async removes stages 3–4 from `C` entirely. Together they keep both `τ_opt` and the floor waste tolerable at multi-terabyte checkpoint sizes. Without them, `C` for a 5.67 TB checkpoint through one rank's link would be measured in hours, `C ≥ 2M` would hold, and Daly's degenerate case ("checkpoint once per MTBF and accept it") would be the honest answer.

**One frontier concern worth naming.** DeepSpeed's **Universal Checkpointing** work decouples a checkpoint's on-disk structure from the exact TP/PP/DP split it was saved under, so a run can resume with a *different* parallelism configuration than it saved with. That matters when elastic scaling (08.5) changes world size across a restart and the old shard boundaries no longer line up. PyTorch DCP gets part of the way there natively via its resharding-on-load (§3); the general TP/PP case is harder.

## Perspectives

**FinOps / formula view.** Young/Daly turns two *measured* numbers — `C` and `M` — into one policy decision. It is striking that a result derived for tape dumps on 1970s mainframes transfers verbatim to a 2026 GPU fleet, and the reason is that nothing in the derivation cares what is failing: it needs only memoryless failures and a save cost you control. The talking point that separates candidates is the difference between "we checkpoint every 30 minutes" and "we checkpoint every 27 minutes because `C` measures 2 minutes and our MTBF is 3 hours, and here are the two waste terms showing they balance."

**Systems / engineering view.** `C` is a pipeline, and each stage has a different limiter: a collective barrier, a PCIe copy, a client-side I/O path, an fsync-plus-barrier. The 504-GPU study's finding that production checkpoint I/O used only 16–21% of available storage bandwidth is a concrete rebuttal to "buy faster storage." Profile the stages first; the answer is often concurrency, thread count, or getting the write off the critical path.

**Storage / infra view.** Sharded checkpointing converts checkpointing from "funnel everything through rank 0" into "every rank writes concurrently," which turns a serialisation problem into a storage IOPS-and-metadata problem at the durable store. That is why storage vendors are now co-authors on training-reliability papers: hundreds of GPUs durably writing at once is a first-class storage-system design problem. It also means your checkpoint performance is a property of the *filesystem's metadata path* as much as its bandwidth — which is exactly the thing that does not show up in a 4-node proof of concept.

**Model-scale view.** Checkpoint bytes are `~14Ψ`, linear in parameters, so a 405B model's checkpoint is ~58× a 7B model's. Since both `τ_opt` and floor waste scale as `√C`, frontier models are strictly more expensive to protect at a given MTBF — arithmetic, not taste. That is why disproportionate engineering goes into async and sharded checkpointing at the top end: without them the optimal interval becomes financially unworkable.

**Reliability-theory view.** The `M_job = m_gpu / N` result is the quiet centre of this module. A perfectly ordinary 5.8-year per-GPU MTBF becomes a 3-hour job MTBF at 16,384 GPUs purely through the `N`. Nothing is broken; scale is doing it. That single relation explains why an architecture that is over-engineering at 100 GPUs (partial-world restart, per-step fault tolerance) is *forced* at 100,000, and it is the bridge from this lesson's arithmetic to 08.5's mechanisms.

## Real-world use cases

- **Meta — Llama 3 405B, 16,384 H100s, 54 days.** 466 interruptions, 419 unexpected, ~78% hardware-attributed, **>90% effective training time**. *What it shows:* both halves of this lesson's model, at the largest published scale. The failure rate matches `M = m/N` with an unremarkable per-GPU MTBF (~50,600 GPU-hours implied), and the >90% effective time is only reachable with **both** a small `C` and a small `R` — as the goodput table in §6 makes explicit.

- **BigScience — BLOOM-176B, 384 processes on GPFS-over-NVMe.** 2.3 TB checkpoints written in **40 seconds** (~57.5 GB/s aggregate, ~150 MB/s per process), saved every ~3 hours over ~3 months, for ~8 hours of total checkpoint overhead — **0.37% of training time**; the operators note it would have been ~2% on 5× slower cloud storage. *What it shows:* the best case, with the arithmetic done by the people who ran it, and a clean confirmation of the `~14Ψ` byte accounting (2.3 TB / 176 B params = 13.1 bytes per parameter).

- **"From Detection to Recovery: Operational Analysis on LLM Pre-training with 504 GPUs"** (Lablup / SKT / Upstage / NVIDIA Korea / VAST Data). 63-node B200 cluster, 55 days of Prometheus data, 73 days of logs, 224 sessions, **523 checkpoint events** traced from VRAM to NFS. Saves at ~16.0% of the 250 GB/s write ceiling, loads at ~21.5% of the 700 GB/s read ceiling, RoCE at 1.4–10.4% utilisation, bottleneck attributed to a **128-slot NFS RPC layer**; the effect did not appear in 2–4-node tests. *What it shows:* the "bandwidth paradox" in production — a rare, measured account of where checkpoint cost actually goes, and the strongest available argument for profiling before purchasing.

- **PyTorch / Meta — distributed asynchronous checkpointing.** Reports a 7B model's checkpoint down time falling from an average of **148.8 s to 6.3 s (23.6×)**, with 7B–13B models generally at 6–14 s of visible downtime, and documents the GIL-contention problem in the thread-based design plus the process-based fix that communicates through shared memory. *What it shows:* the concrete magnitude of the async lever, and the reason "async" needs verifying rather than assuming — the first implementation was async in design and partly synchronous in effect.

- **Meta — OPT-175B chronicles** (`facebookresearch/metaseq`). A day-by-day logbook of a live 992-GPU run, including checkpoint-driven recovery decisions, at least 35 manual restarts and 100+ hosts cycled over roughly two months. *What it shows:* what the `R` term looks like before it is automated — human-in-the-loop restart, with the interval chosen by judgement rather than by formula. It is the baseline 08.5's machinery replaces.

## Worked example

**The ticket.** "We are planning a 70B pre-training run on 512 H100s. Storage gives us 20 GB/s aggregate write. What checkpoint interval should we use, what will it cost us, and where should we spend engineering time?"

**Step 1 — checkpoint bytes.**

```
   Ψ = 70e9,  Adam mixed precision, persist bf16 params + fp32 master + m + v
   bytes/param = 2 + 4 + 4 + 4                  = 14
   checkpoint size = 14 × 70e9                  = 980 GB
   (gradients are transient — not saved. That is 2Ψ = 140 GB you do not write.)
```

**Step 2 — synchronous `C`, stage by stage.** This is the number to beat.

```
   stage 1  plan + metadata all-gather (512 ranks, small payload) ≈  0.5 s
   stage 2  HBM → pinned host, per GPU: 980 GB / 512 = 1.91 GB
            at 55 GB/s measured PCIe 5 device-to-host                0.03 s
   stage 3  durable write: 980 GB / 20 GB/s aggregate               49.0 s
   stage 4  fsync + barrier + .metadata                           ≈  1.5 s
   ──────────────────────────────────────────────────────────────────────
   C_sync                                                          ≈ 51 s
```

Stage 3 is 96% of it. **That is the shape async is designed for.**

**Step 3 — job MTBF.** Use the Llama-3-implied per-GPU figure, and say so:

```
   m_gpu ≈ 50,600 GPU-hours   (implied by 419 interruptions / 54 d / 16,384 GPUs)
   M     = 50,600 / 512       = 98.8 hours ≈ 4.1 days
```

**Step 4 — the interval, synchronous.**

```
   C = 51 s,  M = 98.8 h = 355,680 s
   τ_opt = √(2 × 51 × 355,680) = √36,279,360 = 6,023 s = 100.4 min

   check the two terms balance:
     overhead = 51 / 6,023          = 0.85%
     rework   = 6,023 / (2×355,680) = 0.85%   ✓ equal, as derived
   floor waste = √(2×51/355,680)    = 1.69%
```

**Checkpoint every ~100 minutes, and the tunable waste is only 1.7%.** At 512 GPUs the failure rate is low enough that even a 51-second synchronous checkpoint is not the problem.

**Step 5 — so where is the cost?** Add restart. Assume the unautomated case: someone notices in 20 minutes, the node is drained, the gang re-forms, NCCL re-initialises, and 980 GB is read back at 20 GB/s (~49 s) — call it `R = 25 min = 1,500 s`.

```
   R/M = 1,500 / 355,680 = 0.42%
   total waste = 1.69% + 0.42% = 2.11%   →  goodput 97.9%
```

**At 512 GPUs, none of this matters much.** That is a legitimate and useful finding — say it out loud rather than over-engineering.

**Step 6 — now re-run the same arithmetic at 16,384 GPUs**, because that is the decision the org is really asking about:

```
   M = 50,600 / 16,384 = 3.09 h = 11,124 s
   checkpoint size at 405B instead of 70B = 14 × 405e9 = 5.67 TB
   C_sync = 5,670 GB / 20 GB/s ≈ 284 s  (plus barriers ≈ 290 s)

   τ_opt = √(2 × 290 × 11,124) = √6,451,920 = 2,540 s = 42.3 min
   floor waste = √(2×290/11,124) = 22.8%          ◀── and that is BEFORE restart
   with R = 25 min: R/M = 1500/11,124 = 13.5%
   total waste 36.3%  →  goodput 63.7%            ✖ unacceptable
```

**Step 7 — apply the two levers and re-derive.**

```
   ASYNC: C collapses to stages 1-2 only.
     stage 1 (512→16,384 ranks, larger metadata all-gather)  ≈ 2 s
     stage 2 HBM→pinned: 5,670 GB / 16,384 = 0.35 GB per GPU
             at 55 GB/s                                      ≈ 0.01 s
     C_async                                                  ≈ 3 s

   τ_opt = √(2 × 3 × 11,124) = √66,744 = 258 s = 4.3 min
   floor waste = √(2×3/11,124) = 2.32%             ← from 22.8%. 9.8× better.

   FAST RESTART (08.5: automated drain, elastic re-rendezvous, sharded
   parallel reload): R = 3 min = 180 s
   R/M = 180 / 11,124 = 1.62%

   total waste 2.32% + 1.62% = 3.94%   →  goodput 96.1%
```

**Sanity-check that against reality before you present it.** Meta reports >90%, not 96%, at this scale — so the model is optimistic. What it omits: interruptions that are not clean failures (stragglers degrading throughput without stopping the job), the ramp back to full speed after a restart, planned maintenance (47 of Llama 3's 466 interruptions), and correlated failures that violate the independence assumption behind `M = m/N`. **Present the model as a bound and an attribution, not a forecast.** Its value is that it tells you *which lever* moves the number, and by how much, before you spend a quarter on either.

**The recommendation you deliver.** At 512 GPUs, a 100-minute synchronous checkpoint interval costs ~2% and needs no engineering. At 16,384 GPUs with a 405B model the same design costs ~36% and is unshippable. The two changes that fix it are async checkpointing (22.8% → 2.3% floor waste, the larger win by far) and automated fast restart (13.5% → 1.6%). **Do async first** — it is a library-level change with a documented 10–20× effect, versus the restart path which is a whole control loop. And measure `C` on the real cluster before trusting any of this: the 504-GPU study says to expect roughly 16% of nominal write bandwidth, which would make the synchronous `C` above optimistic by ~6×.

## Practice

Feeds the **Survive-a-failure lab** deliverable ([`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md)). Extend the DDP job from 08.1 on your **2 rented GPUs**.

1. **Save the whole state, not just the weights.** Checkpoint model, optimizer, **step counter, LR-scheduler state, per-rank RNG state, and data-loader position**. Use `torch.save` for a first pass; then redo it with `torch.distributed.checkpoint` (`dcp.save` / `dcp.load` with `get_state_dict` / `set_state_dict` from `torch.distributed.checkpoint.state_dict`) so you see the sharded path.

2. **Prove the resume is exact, not approximate.** Run 200 steps logging loss per step. Restart from the step-100 checkpoint and run to 200 again. **Plot both loss curves on the same axes.** They should overlay. Then deliberately break it: drop the RNG state from the checkpoint and re-run. Note whether you can see the difference by eye — for most jobs you cannot, which is the point of the exercise.

3. **Measure `C`, decomposed.** Time each stage of §2 separately with `torch.cuda.synchronize()` around the boundaries: (a) building the state dict, (b) the device→host copy, (c) the write, (d) the fsync. Report the median over ≥5 saves and the checkpoint size in bytes. **Compute your effective write bandwidth and compare it against what your storage claims.** If you are far below, you have reproduced the 504-GPU study's finding on two GPUs.

4. **Measure `C` again with async.** Switch to `dcp.async_save`. Remember to initialise the process group as `"cpu:gloo,cuda:nccl"` or it will refuse. Record the new blocking `C`. **Then verify it is genuinely async**: compare median step time *during* a background write against steps outside one. If steps slow down while the write is in flight, say so — that is the GIL-contention effect, and it is the finding, not a failure.

5. **Inspect the on-disk layout.** `ls -la` the DCP checkpoint directory. Confirm the `__{rank}_{n}.distcp` files and the `.metadata`. Then delete `.metadata` and attempt a load. **Record the error.** That is the torn-checkpoint detection mechanism, and knowing what it looks like is worth the two minutes.

6. **Do the arithmetic.** With your measured `C` (both sync and async) and a stated **MTBF of 3 hours**, compute `τ_opt = √(2CM)`, both waste terms at that interval (they must be equal — check), and the floor waste `√(2C/M)`. Then repeat for MTBF = 30 days, the realistic figure for a 2-GPU job from §6's table, and note how different the answer is.

7. **Measure `R` separately from `C`.** `kill -9` a rank, restart, and time from the kill to the first completed step after resume. Break it into detection, process restart, `init_process_group` (NCCL re-init), checkpoint load, and warmup. **Report `R` and `C` as separate line items** — conflating them is the most common error in recovery-time estimates, and at 16,384 GPUs a one-minute error in `R` is 273 GPU-hours per failure.

**Acceptance:** a note recording (i) checkpoint size in bytes and bytes-per-parameter, with your model's parameter count; (ii) `C` decomposed into the four stages, synchronous and async; (iii) evidence the resume is exact (the overlaid loss curves); (iv) `τ_opt` and the waste terms for MTBF = 3 h and MTBF = 30 d; (v) a measured `R`, itemised; and (vi) one paragraph: how much did async buy you, and what would `R` cost multiplied across 16,384 GPUs. Commit it under `../practice/survive-a-failure/`.

## Common pitfalls

- **"Checkpoint more often to be safer."** *Mechanism:* past `τ_opt`, the `C/τ` overhead term dominates and total waste *rises*. At `M = 3 h, C = 2 min`, halving the interval from 26.8 to 13.4 minutes takes waste from 14.9% to 18.7% (overhead 14.9% + rework 3.7%) — you are strictly worse off. "More" is not "safer" past the optimum; it is a computable trade, not a hunch to round up on. The useful check: at the optimum the two terms are equal, so if overhead ≫ rework you are over-checkpointing.

- **"Async checkpointing means checkpointing is free."** *Mechanism:* async hides stages 3–4 (serialise, write, fsync) but the state-dict gather and the HBM→pinned-host copy still block. That residue is the `C` in the formula. It is a large reduction — 148.8 s to 6.3 s measured on a 7B model — not an elimination, and treating it as zero makes your `τ_opt` optimistic. Worse, thread-based async can *silently* fail to overlap due to GIL contention; verify by comparing step time during and outside a background write.

- **"Checkpoint write bandwidth is the bottleneck, so buy faster storage."** *Mechanism:* the 504-GPU production study measured saves at ~16% of the storage's write ceiling and loads at ~21.5% of its read ceiling, with RoCE at 1.4–10.4%, and traced the limit to a 128-slot NFS RPC layer — orchestration and metadata concurrency, not media. Faster media against a coordination bottleneck buys nothing. Profile §2's stages, then consider `thread_count` (whose default is 1), fewer larger streams, metadata batching, or moving the write off the critical path.

- **"A bigger model just needs a proportionally longer interval, and that's fine."** *Mechanism:* `τ_opt` does grow as `√C` — but so does the **floor waste** `√(2C/M)`. A 58× larger checkpoint means a 7.6× longer interval *and* 7.6× the unavoidable waste. Bigger models are strictly more expensive to protect at a given MTBF, which is why sharded + async is arithmetically forced at frontier scale rather than optional.

- **"Resume from checkpoint is just loading the file."** *Mechanism:* a full restart also pays detection, drain, re-rendezvous and NCCL re-initialisation before the load even starts (08.5). Reload cost and restart cost are separate line items and both belong in a recovery estimate. In the goodput expression they occupy different places: `C` sits inside `√(2C/M)` and sets the interval; `R` contributes a flat `R/M` and does not affect the interval at all. Conflating them undercounts the real pause and mis-attributes the fix.

- **"The checkpoint contains the model, so we can resume."** *Mechanism:* omitting the step counter restarts the LR schedule; omitting scheduler state resets a cosine's phase; omitting RNG state diverges dropout; omitting data-loader position re-shows data and skips other data. **None of these throws an error.** They produce a run that trains, converges to something, and is not the run you thought you were doing. Only the momentum reset is visible on a loss curve; the rest are invisible without an audit.

- **"Keep only the latest checkpoint to save space."** *Mechanism:* with async, the newest checkpoint may still be mid-write, and a crash during a write leaves `.distcp` files with no `.metadata`. Retention must key off *completion*, and you want at least the two most recent — the practice BLOOM's operators arrived at the hard way. A cleanup job that sorts directories by mtime and deletes all but the newest will eventually delete your only valid checkpoint.

- **"Young/Daly is a 1974 result about tape, so it does not apply."** *Mechanism:* the derivation assumes exactly two things — memoryless (exponential) failures and a save cost you control. Neither mentions the technology. A GPU falling off the PCIe bus satisfies the same model as a 1970s hardware fault. What *would* break it is strongly correlated failures (a rack losing power, a fabric-wide event), which violate independence; those are the cases where the formula's `M` is not a meaningful average and you need a different model.

## Self-check

**(a) Derive the optimal checkpoint interval for MTBF = 3 h and checkpoint cost = 2 min, showing the Young/Daly calculation.**
**Answer:** Work in minutes: `M = 180`, `C = 2`. Total waste is `waste(τ) = C/τ + τ/(2M)` — the first term because you pay `C` seconds of blocking every `τ` seconds of progress, the second because a memoryless failure lands uniformly in the interval so expected lost work is `τ/2`, and failures arrive at rate `1/M`. Differentiate: `d/dτ = −C/τ² + 1/(2M) = 0`, so `τ² = 2CM` and **`τ_opt = √(2CM) = √(2 × 2 × 180) = √720 ≈ 26.8 min`**. Verify by evaluating both terms at that point: overhead `2/26.8 = 7.5%`, rework `26.8/360 = 7.4%` — equal, which is the general result (`both equal √(C/2M)` at the optimum) and a free check on any interval you are given. Total unavoidable waste is `√(2C/M) = √(4/180) ≈ 14.9%`. Daly's second-order correction gives 25.5 min here — a 5% difference on a curve that is flat to within 0.6% across that range, so the one-liner is fine unless `C` is a substantial fraction of `M`.

**(b) What do async and sharded checkpointing each buy you, mechanically?**
**Answer:** They attack different stages of the same pipeline. **Sharded (distributed) checkpointing** has every rank write only its own shard, in parallel, rather than gathering everything to rank 0. Write bandwidth then scales with world size instead of being pinned to one link, and — decisively — it is *mandatory* once the full state does not fit on one rank, which is any frontier model. In PyTorch DCP each rank writes `__{rank}_{n}.distcp` files and rank 0 writes a global `.metadata` last, which is also the commit record. **Async checkpointing** moves the boundary of the blocking cost: the state-dict gather and the HBM→pinned-host copy still block, but serialisation, the durable write and the fsync run on a background thread or process while training continues. Measured, that takes a 7B model's visible downtime from ~148.8 s to ~6.3 s (23.6×). Because both `τ_opt = √(2CM)` and the floor waste `√(2C/M)` scale as `√C`, halving `C` cuts unavoidable waste by only ~29% — so the gains are real but sub-linear, and getting to Llama-3's >90% effective time needs the restart term attacked too.

**(c) Why does checkpoint cost grow with model size, and how does that change the optimal interval?**
**Answer:** Checkpoint bytes are model weights plus optimizer state — persisting bf16 params (2 B), the fp32 master copy (4 B) and Adam's two moments (4+4 B) gives **~14 bytes per parameter**; gradients are transient and are not saved. That is linear in parameter count (98 GB for 7B, 2.46 TB for 176B — measured at 2.3 TB for BLOOM, i.e. 13.1 B/param — and 5.67 TB for 405B), while durable write bandwidth is roughly fixed, so `C ≈ 14Ψ/bandwidth` grows linearly with the model. In Young/Daly, `τ_opt = √(2CM)` grows as `√C`, so bigger models want *longer* intervals — but the floor waste `√(2C/M)` also grows as `√C`, so they are **strictly more expensive to protect at the same MTBF**. A 58× bigger checkpoint means a 7.6× longer interval and 7.6× the waste. That arithmetic, not taste, is why frontier runs invest so heavily in sharding (per-rank `C` grows far slower than total state, because write concurrency grows with world size too) and async (removes the write from `C` entirely).

**(d) A checkpoint is only using 20% of your provisioned storage bandwidth. Is faster storage the fix?**
**Answer:** Probably not — profile before buying. A 63-node, 504-GPU B200 production study traced 523 checkpoint events from GPU VRAM to NFS and measured saves at **~16.0% of the 250 GB/s write ceiling** and loads at **~21.5% of the 700 GB/s read ceiling**, with the 200 Gbps RoCE links at 1.4–10.4% utilisation. The bottleneck was saturation of a **128-slot NFS RPC layer** — orchestration and metadata-path concurrency, not media throughput — and it did not appear at all in 2–4-node tests. Faster media against a coordination bottleneck buys nothing. Work the pipeline stage by stage instead: the state-dict gather (a collective barrier, so it runs at the slowest rank's pace), the HBM→pinned-host copy (PCIe, ~55 GB/s measured on Gen5 x16), the client-side write (check `FileSystemWriter`'s `thread_count`, whose default is **1**), and the fsync-plus-barrier. Then consider fewer larger streams, more I/O threads, metadata batching, or async to take the write off the critical path entirely.

**(e) Derive the expected time between failures for an `N`-GPU job, and use it to explain why 100,000-GPU runs need a different architecture.**
**Answer:** Model each GPU as failing independently with constant hazard rate `λ = 1/m` where `m` is per-GPU MTBF, so one GPU survives to time `t` with probability `e^{−λt}`. A synchronous job dies if *any* rank dies, so the job survives only if all `N` do: `P = (e^{−λt})^N = e^{−Nλt}` — again exponential, with rate `Nλ`. Therefore **`M_job = m_gpu / N`**, falling linearly with GPU count. Cross-check against Llama 3: 419 unexpected interruptions over 54 days (1,296 h) on 16,384 GPUs gives `M = 3.09 h`, implying `m_gpu ≈ 50,600 GPU-hours ≈ 5.8 years` — an entirely ordinary per-device figure. The drama is the `N`, not the hardware. Scaling that: 512 GPUs → 4.1 days between failures; 16,384 → 3.1 hours; 100,000 → **about 30 minutes**. At 30 minutes, a stop-the-world restart costing even 3 minutes of dead time on 100,000 GPUs is 10% of wall-clock *before* any rework, which is why that regime forces partial-world fault tolerance (08.5) rather than the full restart that is perfectly adequate at 512.

**(f) You measure checkpoint reload at 30 seconds. Is that the full cost of a restart?**
**Answer:** No — reload is one line item of several, and the distinction is load-bearing in the maths. A full restart pays **detection** (how long until something notices — a NCCL watchdog timeout is 10 minutes at PyTorch's default, per 08.2), **drain and reschedule** (taint, evict, gang re-form), **re-rendezvous and NCCL re-initialisation** (rebuilding communicators for the new world size), **then** the checkpoint load, **then** warmup back to steady-state throughput. Call the whole thing `R`. In the waste model, `C` and `R` sit in different places: `waste = C/τ + τ/(2M) + R/M`, and `R` differentiates away, so **restart cost does not change the optimal interval** — it adds a flat `R/M` floor. That separation is why they must be measured separately: at MTBF = 3 h, `R = 10 min` costs 5.6% of wall-clock and `R = 3 min` costs 1.7%, and no checkpoint-interval tuning touches either. At 16,384 GPUs a one-minute error in `R` is 273 GPU-hours per failure, mis-costed straight into 08.8's cost-per-successful-run.

**(g) What must a checkpoint contain beyond weights and optimizer state, and what breaks for each omission?**
**Answer:** Five things, and every omission fails *silently*. **Step/iteration counter** — without it the LR schedule and any warmup restart from the beginning. **LR-scheduler state** — a cosine or one-cycle schedule resets its phase, so the effective LR after resume is wrong in a way no error reports. **Data-loader position** (sampler state, epoch, shuffle seed) — you re-train on seen data and skip unseen data, quietly changing the data distribution of a single-epoch pre-training run. **RNG state** for CPU, CUDA and each rank — dropout masks and augmentation diverge from the counterfactual run, breaking reproducibility and, in principle, the training dynamics. **Gradient-scaler state** on fp16 runs — the loss scale restarts at its initial value and you eat a burst of skipped steps. Only the optimizer-moment omission is visible on a loss curve (a bump that recovers over a few hundred steps); the rest are invisible without auditing. That is why the acceptance test is "the loss curve overlays the counterfactual," not "it loaded without error."

## Connections & what's next

You now have the arithmetic behind the failure-overhead term 08.3 named but did not quantify: what a checkpoint contains (`~14Ψ`), what it costs as a four-stage pipeline, how sharding and async each attack a different stage, and the Young/Daly interval that turns `C` and `M` into a policy. You can also derive `M` itself from GPU count, which is the relation that makes frontier scale qualitatively different.

**08.5 is the other half of the goodput expression.** This lesson's model has three terms, and it only gave you a lever on one of them: `C`. The `R/M` term — detection, drain, re-rendezvous, NCCL re-init, reload, warmup — is 08.5's entire subject, and §6's table shows why it cannot be skipped: async alone reaches 89%, fast restart alone reaches 83%, and only both together reach 93%. 08.5 also owns the health-signal-to-MTBF story that this lesson took as an input, and the elastic machinery that makes DCP's resharding-on-load (§3) useful in practice.

**08.6** shows the Kubernetes object that expresses the restart; **08.8's capstone** takes `C`, `R`, `M`, `τ` and 08.3's MFU and produces one cost-per-successful-run number, with this lesson's goodput expression as its spine.

Backward: **08.1** supplied the model-state accounting the `14Ψ` figure is derived from, and the FSDP2 sharded-state-dict property that makes stage 1 cheap; **08.2** supplied the fact that the metadata exchange is a barrier and therefore runs at the slowest rank's pace; **08.3** supplied the MFU framing that failure overhead silently corrupts when averaged over long windows; **05** supplies the health signals that determine when a failure is even noticed.

## References & further reading

**Primary sources**

1. **The Llama 3 Herd of Models, §3.3 "Infrastructure, Scaling, and Efficiency"** — <https://arxiv.org/abs/2407.21783>. The module's anchor: 16,384 H100s over 54 days, **466 interruptions of which 419 unexpected** (roughly one every three hours), ~78% attributed to hardware, and **>90% effective training time**. The implied per-GPU MTBF of ~50,600 GPU-hours used throughout §6 is derived from those figures here, not quoted from the paper. *`arxiv.org` is blocked by this environment's egress proxy; the interruption counts, hardware fraction and effective-time figure were confirmed via search snippets of the paper's own text and contemporaneous technical reporting, not by reading the PDF in this pass.*

2. **J. W. Young (1974), "A first order approximation to the optimum checkpoint interval," CACM 17(9); and J. T. Daly (2006), "A higher order estimate of the optimum checkpoint interval for restart dumps," Future Generation Computer Systems 22, 303–312.** The origin of `τ_opt = √(2μC)` and of the higher-order form quoted in §5, including the degenerate `τ = M` case when the checkpoint cost reaches `2M`. A readable modern derivation is A. Benoit et al., *"Checkpointing à la Young/Daly: An Overview"* — <https://icl.utk.edu/files/publications/2022/icl-utk-1569-2022.pdf>. *Neither the original papers nor the overview PDF were fetched in this pass; the first-order derivation in §5 is worked from first principles above and does not depend on reading them. The second-order expression is reproduced as the standard published form and should be checked against Daly's paper before being quoted precisely.*

3. **PyTorch — `torch/distributed/checkpoint/` (`state_dict_saver.py`, `filesystem.py`, `staging.py`, `state_dict.py`)** — <https://github.com/pytorch/pytorch>. **Read `filesystem.py`'s `FileSystemWriter.__init__` and `staging.py`'s `StagingOptions`.** Source of everything mechanical in §3 and §4: the `save` / `async_save` signatures and the `AsyncCheckpointerType.THREAD | PROCESS` choice; the `__{rank}_{n}.distcp` naming and the `.metadata` commit file; the `FileSystemWriter` defaults `single_file_per_rank=True`, `sync_files=True`, **`thread_count=1`**, `per_thread_copy_ahead=10 MB`; the `StagingOptions` defaults `use_pinned_memory`, `use_shared_memory`, `use_async_staging`, `use_non_blocking_copy` all `True`; and the async assertion message *"A CPU backend must be enabled for async save; try initializing process group with 'cpu:gloo,cuda:nccl'"*. *`docs.pytorch.org` is blocked by this environment's egress proxy; verified against the in-repo Python at HEAD.*

4. **PyTorch — `docs/source/distributed.checkpoint.md`** (same repository). The DCP contract in prose: multiple files per checkpoint with at least one per rank, in-place loading against pre-allocated storage, and **load-time resharding** enabling save-in-one-topology / load-into-another — the property §3 relies on and that 08.5's elastic restarts depend on. Verified in-source.

5. **torchtitan — `docs/checkpoint.md` and `torchtitan/components/checkpointer/dcp.py`** — <https://github.com/pytorch/torchtitan>. Source of the production configuration surface named in §4: `async_mode ∈ {disabled, async, async_with_pinned_mem}`, `interval`, `keep_latest_k`, `enable_first_step_checkpoint`, `last_save_model_only`, `export_dtype`, and the seed-checkpoint pattern for reproducible initialisation across device counts. Verified by cloning at HEAD.

**Real-world engineering blogs**

6. **PyTorch / Meta — "Reducing Model Checkpointing Times by Over 10x with PyTorch Distributed Asynchronous Checkpointing"** — <https://pytorch.org/blog/reducing-checkpointing-times/>. Source of the async magnitude in §4: a 7B model's checkpoint down time falling from an average of **148.8 s to 6.3 s (23.6×)**, 7B–13B models generally at 6–14 s of visible downtime, and the two-phase description (GPU→CPU copy, then background CPU→disk). **Correction:** the previous version of this lesson cited a "5–15×" async reduction attributed to TorchTitan; that specific multiplier could not be confirmed against any primary source in this pass, and has been replaced with the measured 148.8 s → 6.3 s figure and the blog's own 10–20× characterisation. *`pytorch.org` is blocked by this environment's egress proxy; the figures were confirmed via search snippets of the blog, not by reading it.*

7. **PyTorch / Meta — "Distributed Checkpoint: Efficient checkpointing in large-scale jobs"** — <https://pytorch.org/blog/distributed-checkpoint-efficient-checkpointing-in-large-scale-jobs/>. The first-party account of DCP at Meta scale, including the GIL-contention problem in the thread-based async design and the process-based fix communicating through shared memory, described in §4. *`pytorch.org` blocked here; the thread/process distinction and shared-memory mechanism are independently verified in the `_async_thread_executor.py` / `_async_process_executor.py` source (reference 3), which is what the claim above rests on.*

8. **"From Detection to Recovery: Operational Analysis on LLM Pre-training with 504 GPUs"** (Lablup, SKT, Upstage, NVIDIA Korea, VAST Data) — <https://arxiv.org/abs/2605.09370>. **The most valuable empirical source in this lesson.** A 63-node B200 cluster, 55 days of Prometheus time-series and 73 days of operational logs across 224 multi-node sessions; **523 checkpoint events** traced from GPU VRAM to the NFS store, with saves reaching ~16.0% of the storage's 250 GB/s write ceiling and restart loads ~21.5% of its 700 GB/s read ceiling, RoCE utilisation of 1.4–10.4%, and the bottleneck attributed to saturation of a **128-slot NFS RPC layer** rather than the media — an effect absent from 2–4-node tests. *`arxiv.org` is blocked by this environment's egress proxy; the cluster description and all five figures were confirmed via search snippets of the paper's abstract and HTML rendering, not by reading the PDF.*

9. **`stas00/ml-engineering` — `training/fault-tolerance/README.md`** — <https://github.com/stas00/ml-engineering>. **Read the "Frequent checkpoint saving" section in full.** Source of the BLOOM-176B case study used in §1 and §7: a **2.3 TB checkpoint written in 40 seconds across 384 concurrent processes** on GPFS-over-NVMe, saved roughly every 3 hours over ~3 months for ~720 checkpoints and ~8 hours total overhead — **0.37% of training time**, with the operators' own note that 5× slower cloud storage would make it ~2%. Also the two-most-recent-checkpoints-local retention practice quoted in §4. Verified by cloning at HEAD.

10. **`stas00/ml-engineering` — `network/README.md`** (same repository). Source of the PCIe measurement used in §2's stage 2: on an 8×H200 node, `nvbandwidth` reports **55.06 GB/s device-to-host** against the 63 GB/s PCIe 5 x16 spec (87%), with the eight links within 0.10 GB/s of one another — and the caveat that the same test on a Grace-Blackwell system measures NVLink-C2C instead and reports several hundred GB/s. Verified by cloning at HEAD.

11. **Meta — OPT-175B chronicles** — <https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles>. **Skim the logbook.** A day-by-day account of checkpoint-driven recovery on a live 992-GPU run — at least 35 manual restarts and 100+ hosts cycled over roughly two months. The pre-automation baseline for the `R` term.

**Deeper dives**

12. **"Universal Checkpointing: Efficient and Flexible Checkpointing for Large Scale Distributed Training" (DeepSpeed)** — <https://arxiv.org/abs/2406.18820>. Decoupling a checkpoint's on-disk structure from the TP/PP/DP split it was saved under, so a run can resume with a different parallelism configuration — the frontier version of the resharding property §3 describes in DCP. *`arxiv.org` blocked here; **not read** in this pass and not relied upon for any claim above beyond the existence and stated purpose of the work.*

13. **"DataStates-LLM: Lazy Asynchronous Checkpointing for Large Language Models"** — <https://arxiv.org/abs/2406.10707>. A research treatment of lazy and asynchronous checkpointing that goes beyond DCP's staging design. *`arxiv.org` blocked here; **not read** in this pass, listed as optional depth only.*
