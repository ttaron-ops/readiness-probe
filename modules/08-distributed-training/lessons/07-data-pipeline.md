---
lesson: "08.7"
title: "Data pipeline"
module: "08"
concept: "Data pipeline"
status: not-started
est_time: "5h"
artifacts: []
---

# 08.7 · Data pipeline

> **Concept.** A GPU idling on JPEG decode is burning money; find and fix input-pipeline starvation.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Why this matters

You already know how to read a roofline (03) and GPU SM% telemetry (05). Here is the failure
mode those skills expose that nobody staffs an on-call rotation for: the accelerator is *starved*.
The GPU sits at 30–40% SM active while a CPU core (or all of them) pegs at 100%, because the input
pipeline — read file, decode JPEG, resize, augment, collate — cannot deliver batches fast enough.

The reason this is *your* lesson and not the ML engineer's: **a starved GPU still bills at the full
rate.** An 8×H100 node does not get cheaper because the data loader is slow. If SM% averages 40%
instead of 90%, you are paying ~2.25× the true cost of the compute you actually landed. On a
fleet of 40+ clusters that is a line item, not a curiosity. This lesson turns "the GPU looks bored"
into recovered GPU-hours you can put in a ticket.

## What's new here

- **Starvation vs. the other two idle causes.** Low SM% has three usual suspects: *comms-bound*
  (NCCL all-reduce stalls — lesson 03, GPUs busy waiting on the network, CPU idle), *sync-bound*
  (a straggler / barrier), and *input-bound* (this lesson — CPU pegged, GPU waiting on `next(batch)`).
  The CPU signature is what separates starvation from the rest.
- **The data loader is a distributed system too.** `num_workers`, `prefetch_factor`, `pin_memory`,
  and `persistent_workers` are the knobs; shared memory (`/dev/shm`), file-system IOPS, and object-store
  latency are the constraints.
- **Two escalating fixes.** When more CPU workers stop helping, you change the *format* (sharded
  streaming — WebDataset) or move decode *onto the GPU* (NVIDIA DALL, i.e. DALI's nvJPEG path).
- **The FinOps framing.** Every fix has a resource cost (CPU cores, RAM, GPU SMs, complexity). The
  win is only real if recovered GPU-hours > the cost of the CPUs/engineering you spent to recover them.

## Core notes

### The diagnosis, step by step

1. **Confirm the symptom.** `nvidia-smi dmon -s u` or DCGM `DCGM_FI_PROF_SM_ACTIVE`
   (fraction of time ≥1 warp resident) reads low and *choppy* — sawtooth, not a flat low line. A flat
   low line is more often comms/occupancy; a sawtooth (compute a bit, stall, compute a bit) is the GPU
   waiting on batches. Pair it with `DCGM_FI_DEV_GPU_UTIL` for the coarse view.
2. **Check the CPU.** `mpstat -P ALL 1` or `htop`. Input starvation shows one or more cores pinned at
   100% in the data-loader worker processes (`%usr` high, doing decode/resize). If CPU is *idle* while
   SM% is low, stop — you are comms- or sync-bound, not starved; go back to lesson 03.
3. **Attribute the time.** Wrap the loop with `torch.profiler` (or a hand timer) and separate
   *data time* (time blocked in `next(loader)`) from *compute time* (forward+backward+step). If data
   time is a large fraction of step time, the loader is on the critical path. This single number is
   your before/after evidence.
4. **Find the hot transform.** Usually one operation: JPEG decode (Pillow is slow; `Pillow-SIMD` or
   `torchvision.io.decode_jpeg` faster), a heavy augmentation, or reading millions of tiny files
   (IOPS-bound, not CPU-bound — the fix there is sharding, not more workers).

### The knobs (PyTorch `DataLoader`)

- **`num_workers`** — subprocesses that build batches in parallel. `0` means the *main* process loads
  synchronously (guaranteed starvation under any real transform). Rule of thumb: start around
  `4 × GPUs_per_node`, then tune. Bounded above by physical cores and RAM — each worker holds a copy of
  dataset state and pushes tensors through `/dev/shm`.
- **`prefetch_factor`** — batches each worker preloads ahead of demand (default `2`). Total in-flight
  buffer ≈ `num_workers × prefetch_factor`. Raising it hides *bursty* latency (e.g. S3 tail latency) at
  the cost of RAM.
- **`pin_memory=True`** — stages fetched tensors in page-locked host memory so the host→device copy is a
  DMA `cudaMemcpyAsync` that overlaps compute. Only pays off when you also copy with
  `tensor.to(device, non_blocking=True)`.
- **`persistent_workers=True`** — keep workers alive across epochs; avoids paying respawn + dataset
  re-init every epoch (matters for many short epochs).

### When more workers stop helping

Adding workers has diminishing returns and hard walls: you run out of cores, you exhaust `/dev/shm`
(the classic `Bus error` / `DataLoader worker killed`), or you are **IOPS-bound** reading millions of
small files where more workers just add more random-read contention. Two structural fixes:

- **Sharded / streaming format — WebDataset.** Pack samples into tar *shards* (e.g. 1 GB each) and read
  them *sequentially* and streamed straight from object storage (S3/GCS). Sequential large reads convert
  an IOPS problem into a bandwidth problem — far friendlier to object stores and to the page cache.
  Shards also shard cleanly across DDP ranks. Cost: a preprocessing/packing step and a shuffle-buffer
  (you shuffle within a window, not globally).
- **GPU-side decode — NVIDIA DALI.** Move decode + resize + augment off the CPU and onto the GPU's
  hardware JPEG decoder (nvJPEG) with a DALI pipeline. This directly attacks the "CPU pegged on JPEG
  decode" case. Cost: DALI is another dependency and a different pipeline API, it consumes some GPU
  memory and a slice of SMs for decode, and it only wins when the CPU (not the GPU) was the bottleneck.

### The cost lens

Let a node's fully-loaded rate be `$R` per GPU-hour (use your own blended number — see the snapshot
note in [08.8](08-training-economics.md); this lesson uses **$40/GPU-hr** as a round stand-in).
The *effective* cost of landed compute scales as `1 / SM_active`. Going from 40% → 90% SM active is a
`0.40/0.90 = 0.44` multiplier on cost-per-unit-of-training — a **56% reduction** — for the price of
extra CPU workers that cost a small fraction of a GPU. That asymmetry (CPU is cheap, GPU is not) is the
whole argument for over-provisioning the data pipeline.

## Worked example

**Setup.** 8×H100 node, ResNet-style training, batch 256, dataset 1.28M images, so 5000 steps/epoch.
Compute-bound step time (pure forward+backward) is **200 ms**. Rate `$R = $40/GPU-hr` →
`$320/node-hr`.

**Before (throttled on purpose).** `num_workers=2`, plain-Pillow decode+resize, `pin_memory=False`.
Data time per step = 300 ms, so it *hides* only 200 ms of it behind compute and the step is
gated at **500 ms**.

- SM active ≈ `200 / 500 = 40%`.
- `mpstat` shows the 2 worker cores at 100% `%usr` (decode), GPU sawtoothing at ~40%.
- Epoch = `5000 × 0.5 s = 2500 s = 0.694 hr`.

**Fix.** `num_workers=16`, `prefetch_factor=4`, `pin_memory=True` + `non_blocking=True` copies,
`persistent_workers=True`. Now aggregate decode throughput exceeds demand; data time is fully hidden
behind the 200 ms compute (plus ~10 ms residual copy).

- Step ≈ **210 ms**, SM active ≈ `200/210 = 95%`.
- Epoch = `5000 × 0.21 s = 1050 s = 0.292 hr`.

**Recovered GPU-hours (the deliverable number).** Per epoch, per node:

```
before = 0.694 hr/epoch × 8 GPUs = 5.55 GPU-hr/epoch
after  = 0.292 hr/epoch × 8 GPUs = 2.34 GPU-hr/epoch
saved  = 3.21 GPU-hr/epoch  (58% less)
```

Over a 90-epoch run: `3.21 × 90 = 289 GPU-hr` recovered per node.
At `$40/GPU-hr`: **`289 × 40 = $11,560` saved per run, per node** — bought with ~14 extra CPU cores
that cost single-digit dollars/hr. Even at a realistic H100 rate of `$12/GPU-hr` (snapshot below) it is
still **$3,468/run/node**, and it compounds across every repeat of the experiment and every node.

## Practice

Feeds the **Survive-a-failure lab** deliverable with a *before/after starvation fix, GPU-hours
quantified*.

1. **Instrument.** On a single-GPU or single-node job, log SM active (DCGM `DCGM_FI_PROF_SM_ACTIVE`
   or `nvidia-smi dmon -s u`), per-core CPU (`mpstat -P ALL 1`), and *data time vs compute time* from
   `torch.profiler`. Record steps/sec (throughput).
2. **Induce starvation.** Set `num_workers=1` (or `0`) and add a deliberately slow transform (e.g. a
   Python-side sleep or an expensive PIL op). Watch SM% collapse into a sawtooth while a CPU core pegs
   at 100%. Capture the plot/log.
3. **Fix.** Raise `num_workers` toward `4×`GPUs, set `prefetch_factor`, `pin_memory=True` +
   `non_blocking=True`, `persistent_workers=True`. Re-measure SM% and throughput. If you are IOPS-bound
   (tiny files), repack a slice as WebDataset shards and show the read pattern change.
4. **Quantify.** Compute throughput before vs after, convert to GPU-hours for a fixed epoch count, and
   multiply by your `$/GPU-hr`. Write the one-line result.

**Acceptance:** a committed before/after artifact showing (a) the starvation signature
(low SM% + pegged CPU), (b) the fix, and (c) **recovered GPU-hours and dollars quantified** for a
fixed workload. That number is the input the lab and lesson 08.8 consume.

## Self-check

**(a) GPU SM% is low but a CPU core is pegged at 100%. Diagnosis and next check?**
**Answer:** Input-pipeline starvation — the GPU is blocked waiting on `next(loader)` while a data-loader
worker burns CPU on decode/transform. Next check: attribute the step with `torch.profiler` (or a timer)
to confirm *data time* dominates *compute time*, and identify the hot transform (usually JPEG decode).
Contrast with the CPU-idle case, which would be comms- or sync-bound, not starved.

**(b) Why is data-loader starvation a *cost* bug, not just a performance bug?**
**Answer:** The GPU bills at the full hourly rate whether it is 95% or 40% utilized. Landed compute
scales as `1/SM_active`, so 40% vs 90% is ~2.25× the true cost per unit of training progress — you are
renting a $40/GPU-hr accelerator to watch it wait on a CPU that costs cents. The fix (more workers) is
cheap CPU; the waste it removes is expensive GPU. That asymmetry is why it belongs on the cost ledger.

**(c) Name two fixes and their tradeoffs.**
**Answer:** (1) *More `num_workers` + `prefetch_factor` + `pin_memory`* — cheapest first move, but
bounded by cores/RAM and useless (or harmful) when IOPS-bound on tiny files. (2) *WebDataset sharding*
(sequential streamed reads) fixes the IOPS wall but needs a repack step and only window-shuffling; or
*NVIDIA DALI GPU decode* offloads decode to the GPU's nvJPEG unit but adds a dependency and consumes GPU
memory/SMs, so it only wins when the CPU — not the GPU — was the bottleneck.

## Resources

1. **PyTorch `DataLoader` docs** — `num_workers`, `prefetch_factor`, `pin_memory`, `persistent_workers`
   semantics and the multi-process loading model. <https://docs.pytorch.org/docs/stable/data.html>
2. **NVIDIA DALI** — GPU-accelerated data loading / nvJPEG decode, the fix when CPU decode is the wall.
   <https://docs.nvidia.com/deeplearning/dali/user-guide/docs/>
3. **WebDataset** — tar-shard streaming format for sequential, object-store-friendly reads at scale.
   <https://github.com/webdataset/webdataset>

> **Snapshot (2026-08).** On-demand H100 SXM lands roughly **$3–12/GPU-hr** depending on
> neocloud vs. hyperscaler; the `$40/GPU-hr` used above is a round fully-loaded stand-in. Rates move —
> substitute your own blended number and re-run the arithmetic.
