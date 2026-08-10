---
lesson: "08.7"
title: "Data pipeline"
module: "08"
concept: "Data pipeline"
status: not-started
est_time: "6h"
prev: "06-job-orchestration.md"
next: "08-training-economics.md"
artifacts: []
sources: 8
---

# 08.7 · Data pipeline

> **Concept.** A GPU idling on JPEG decode is burning money; find and fix input-pipeline starvation.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

06 got the gang placed and rendezvoused; 05's elastic layer keeps it alive across failures. This
lesson assumes the job is running, healthy, gang-scheduled, fully rendezvoused — and still slow,
for a reason that has nothing to do with NCCL or checkpointing. It is the last "is the GPU actually
working" diagnosis in the module before 08.8 turns everything into a dollar figure. Where 03 taught
you comms-bound (GPUs busy, waiting on the network) and 05 taught you failure-bound (GPUs gone,
waiting on a restart), this lesson teaches the third shape of wasted GPU-hours: input-bound (GPUs
idle, waiting on a Python process to decode a JPEG).

## Why this matters

You already know how to read a roofline (03) and GPU SM% telemetry (05). Here is the failure
mode those skills expose that nobody staffs an on-call rotation for: the accelerator is *starved*.
The GPU sits at 30–40% SM active while a CPU core (or all of them) pegs at 100%, because the input
pipeline — read file, decode JPEG, resize, augment, collate — cannot deliver batches fast enough.

The reason this is *your* lesson and not the ML engineer's: **a starved GPU still bills at the full
rate.** An 8×H100 node does not get cheaper because the data loader is slow. If SM% averages 40%
instead of 90%, you are paying ~2.25× the true cost of the compute you actually landed. On a
fleet of 40+ clusters that is a line item, not a curiosity. This is not a hypothetical multiplier —
Uber's own engineering team measured a real pipeline stuck at ~12% GPU utilization, a 22-hour
training run, and fixed it by re-architecting the loader (not by adding GPUs). The result: >60%
utilization, a 3-hour run, and roughly an **80% cut in compute cost** for the identical training
job. Nothing about the model changed. This lesson turns "the GPU looks bored" into recovered
GPU-hours you can put in a ticket.

## What's new here (calibration)

- **Starvation vs. the other two idle causes.** Low SM% has three usual suspects: *comms-bound*
  (NCCL all-reduce stalls — lesson 03, GPUs busy waiting on the network, CPU idle), *sync-bound*
  (a straggler / barrier), and *input-bound* (this lesson — CPU pegged, GPU waiting on `next(batch)`).
  The CPU signature is what separates starvation from the rest. We are not re-teaching 03's roofline
  or 05's DCGM/XID reading — we are adding the one diagnosis those lessons deliberately left out.
- **The data loader is a distributed system too.** `num_workers`, `prefetch_factor`, `pin_memory`,
  and `persistent_workers` are the knobs; shared memory (`/dev/shm`), file-system IOPS, and object-store
  latency are the constraints.
- **Two escalating fixes.** When more CPU workers stop helping, you change the *format* (sharded
  streaming — WebDataset) or move decode *onto the GPU* (NVIDIA DALI's nvJPEG path).
- **The FinOps framing.** Every fix has a resource cost (CPU cores, RAM, GPU SMs, complexity). The
  win is only real if recovered GPU-hours > the cost of the CPUs/engineering you spent to recover them.
  This is the same lens 08.8 formalizes for the whole module — here it is applied to one bottleneck.

## Core concepts

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
   your before/after evidence. A profiler-based, component-by-component version of this same
   attribution — isolating raw read, decode, augment, and host→device copy separately via a caching
   trick — is worth reading once; see Chaim Rand's walkthrough in Resources.
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
  Shards also shard cleanly across DDP ranks. This is not a toy technique: LAION-5B, a 5.85-billion-pair
  (~240 TB) dataset, is stored *entirely* as WebDataset shards, and the OpenCLIP training pipeline reads
  those shards at over 10,000 samples/sec per GPU node — a concrete demonstration that sequential shard
  reads scale to some of the largest public training sets in existence. Cost: a preprocessing/packing
  step and a shuffle-buffer (you shuffle within a window, not globally — see Pitfalls). Databricks'
  MosaicML StreamingDataset is a second, independently-built production system aimed at the identical
  problem (object-store latency starving the GPU at training scale), worth knowing as an alternative
  design point to WebDataset.
- **GPU-side decode — NVIDIA DALI.** Move decode + resize + augment off the CPU and onto the GPU's
  hardware JPEG decoder (nvJPEG) with a DALI pipeline. This directly attacks the "CPU pegged on JPEG
  decode" case. Cost: DALI is another dependency and a different pipeline API, it consumes some GPU
  memory and a slice of SMs for decode, and it only wins when the CPU (not the GPU) was the bottleneck
  — see the Perspectives section for how workload-dependent the win actually is.

### IOPS vs. bandwidth — the framing underneath both fixes

A local SSD and an object store both quote a bandwidth number (GB/s), but training pipelines usually
die on **IOPS**, not bandwidth: millions of small files (a 50 KB JPEG) mean millions of small random
reads, and both local filesystems and object stores have a per-request latency floor that dominates at
that file size. The fix in both structural techniques above is the same underlying move — **stop
issuing millions of small requests; issue thousands of large sequential ones.** WebDataset shards do
this explicitly (pack many samples into one tar, read the tar sequentially). This is why "read from a
faster disk" alone rarely fixes a starved pipeline: if you are IOPS-bound, a faster disk still pays the
same per-request latency tax on the same number of requests.

## Perspectives

**The FinOps/cost view.** Every recovered SM-percentage-point is a `1/SM_active` cost reduction — see
the worked example below. Uber's real 22h→3h, 12%→60%+ result is a dramatically bigger number than
almost any synthetic classroom example; if anything, the arithmetic in this lesson is conservative
next to what's been measured in production.

**The ML-engineer's view (and why it differs from yours).** A researcher watching their own job does
not usually see starvation as a problem: the job runs, the loss curve looks normal, checkpoints land,
and "training is just slow today" doesn't trigger alarm the way a crash does. They are optimizing for
iteration speed and augmentation correctness, not GPU-hours accounting. Unless someone is watching
SM%/cost dashboards, a starved pipeline is invisible from the research seat — which is exactly why
this diagnosis is platform's job, not a footnote in someone else's code review.

**The storage/IOPS view.** The WebDataset/LAION-5B case is the cleanest demonstration of "convert an
IOPS problem into a bandwidth problem." Sequential shard reads at >10,000 samples/sec per node from an
object store are only possible *because of* the sharding — the object store did not get faster, the
access pattern got friendlier to it.

**The GPU-vs-CPU cost-asymmetry view.** DALI's own published benchmarks (NVIDIA's ResNet-50 case study,
and AWS's independent SageMaker reproduction) show throughput gains from single digits up to roughly
70%+ depending on configuration and augmentation load — a wide range, not a guaranteed win. GPU-side
decode consumes GPU memory and SMs; it only pays off when CPU decode was truly the bottleneck and the
GPU had SM headroom to spare. Treat "just turn on DALI" the same way you'd treat any other capacity
trade: verify the bottleneck first.

## Real-world use cases

- **Uber — "Accelerating Deep Learning: How Uber Optimized Petastorm for High-Throughput and
  Reproducible GPU Training"** — <https://www.uber.com/us/en/blog/accelerating-deep-learning/> — a
  production data-loading pipeline stuck at ~12% GPU utilization and a 22-hour run, fixed by removing
  I/O bottlenecks in the loader. What it shows: GPU utilization rose to 60%+, wall-clock dropped to
  ~3 hours, and compute cost fell roughly 80% — for the *same* training job. The clearest real-world
  proof that starvation is a cost bug, not a performance footnote.
- **Databricks/MosaicML — "MosaicML StreamingDataset: Fast, Accurate Streaming of Training Data from
  Cloud Storage"** — <https://www.databricks.com/blog/mosaicml-streamingdataset> — a production
  streaming-dataset library purpose-built to stop object-store latency from starving the GPU at
  training scale. What it shows: WebDataset is not the only production answer to this problem; a
  second major ML infra org independently arrived at the same "shard + stream" structural fix.
- **LAION-5B / OpenCLIP** — dataset description at <https://laion.ai/blog/laion-5b/>, training
  pipeline via the WebDataset/OpenCLIP tooling — a 5.85-billion-pair, ~240 TB dataset stored entirely
  as WebDataset tar shards, read at >10,000 samples/sec per GPU node in the OpenCLIP training
  pipeline. What it shows: sequential shard streaming scales to some of the largest public
  vision-language training sets that exist, not just small demo pipelines.
- **NVIDIA — "Case Study: ResNet50 with DALI"** — <https://developer.nvidia.com/blog/case-study-resnet50-dali/>
  — NVIDIA's own benchmark of DALI's GPU-side nvJPEG decode across DGX-2, AWS P3, and TITAN RTX
  configurations. What it shows: the improvement is real but highly configuration-dependent — as
  low as single digits on a lightly CPU-bound config, much larger under heavier augmentation loads
  (AWS's independent SageMaker reproduction below measured 37–72% depending on the ResNet variant) —
  a useful corrective against assuming GPU-side decode is a uniform win.

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
still **$3,468/run/node**, and it compounds across every repeat of the experiment and every node. For
scale, this synthetic example's 58% GPU-hour reduction is *smaller* than Uber's measured real-world
result (~80% compute cost cut) — the arithmetic here is, if anything, conservative.

## Practice

Feeds the **Survive-a-failure lab** deliverable with a *before/after starvation fix, GPU-hours
quantified*. See [`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md)
for the full lab spec; this lesson supplies its data-pipeline artifact.

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

## Common pitfalls

- **"More `num_workers` always fixes it."** Bounded above by physical cores and RAM, and actively
  harmful when you're IOPS-bound on millions of tiny files — more workers just means more contending
  random reads. Uber's actual fix was caching/format-level, not raising a worker count; if adding
  workers stops moving SM%, you've hit a structural wall, not a tuning problem.
- **"GPU-side decode (DALI) is a free win."** NVIDIA's own benchmark and AWS's independent
  reproduction both show a *range* — from single digits to 70%+ — not a guaranteed large gain. DALI
  consumes GPU memory and SMs; applying it to a job that was actually comms-bound or compute-bound
  wastes the SMs it consumes for no benefit. Confirm CPU is the bottleneck first.
- **"A flat low SM% and a sawtooth low SM% mean the same thing."** They don't: sawtooth with CPU
  pegged is starvation; flat-low with CPU idle is comms- or sync-bound (back to lesson 03). Treating
  the wrong one with the wrong fix wastes engineering time and buys nothing.
- **"WebDataset shuffling is equivalent to full-dataset shuffling."** WebDataset only window-shuffles
  within a shuffle buffer, not globally across the dataset. For shuffle-quality-sensitive regimes
  (e.g., some contrastive-learning setups), this is a real statistical tradeoff to name explicitly,
  not a free format swap.
- **"The data pipeline is the ML team's problem, not platform's."** A starved GPU bills at full rate
  regardless of whose code caused it. Uber's ~80% measured cost reduction is concrete evidence this is
  a platform-owned cost line item, not a research-team code-quality footnote.

## Self-check

- **GPU SM% is low but a CPU core is pegged at 100%. Diagnosis and next check?**
  **Answer:** Input-pipeline starvation — the GPU is blocked waiting on `next(loader)` while a data-loader
  worker burns CPU on decode/transform. Next check: attribute the step with `torch.profiler` (or a timer)
  to confirm *data time* dominates *compute time*, and identify the hot transform (usually JPEG decode).
  Contrast with the CPU-idle case, which would be comms- or sync-bound, not starved.

- **Why is data-loader starvation a *cost* bug, not just a performance bug?**
  **Answer:** The GPU bills at the full hourly rate whether it is 95% or 40% utilized. Landed compute
  scales as `1/SM_active`, so 40% vs 90% is ~2.25× the true cost per unit of training progress — you are
  renting a $40/GPU-hr accelerator to watch it wait on a CPU that costs cents. Uber's real fix (loader
  re-architecture, not more GPUs) cut compute cost ~80% on an unchanged model. That asymmetry is why it
  belongs on the cost ledger, not just the perf dashboard.

- **Name two structural fixes and their tradeoffs.**
  **Answer:** (1) *More `num_workers` + `prefetch_factor` + `pin_memory`* — cheapest first move, but
  bounded by cores/RAM and useless (or harmful) when IOPS-bound on tiny files. (2) *WebDataset sharding*
  (sequential streamed reads, proven at LAION-5B scale) fixes the IOPS wall but needs a repack step and
  only window-shuffling; or *NVIDIA DALI GPU decode* offloads decode to the GPU's nvJPEG unit but adds a
  dependency and consumes GPU memory/SMs, with benchmarked gains ranging from single digits to 70%+
  depending on the workload — it only wins when the CPU, not the GPU, was the bottleneck.

- **Why doesn't a faster disk alone fix a starved pipeline reading millions of small files?**
  **Answer:** Because the bottleneck is usually IOPS (request count and per-request latency), not raw
  bandwidth. A faster disk still pays the same per-request latency tax on the same number of small
  random reads. The real fix is reducing the *number* of requests — sharding many small files into
  fewer large sequential reads (WebDataset's approach) — which converts an IOPS-bound problem into a
  bandwidth-bound one that both local disks and object stores handle far better.

## Connections & what's next

07 closes the "is the GPU actually doing useful work" thread that 03 (comms), 05 (failures), and this
lesson (starvation) together cover. With all three diagnoses in hand, 08.8 folds them into a single
formula: MFU (which starvation directly depresses if left unfixed) times `$/GPU-hr` times a failure-
overhead multiplier. The recovered-GPU-hours number from this lesson's Practice section is a direct
input to that formula's MFU term — carry it forward.

## References & further reading

- **Primary sources**
  - PyTorch `DataLoader` docs — `num_workers`, `prefetch_factor`, `pin_memory`, `persistent_workers`
    semantics and the multi-process loading model. <https://docs.pytorch.org/docs/stable/data.html>
  - NVIDIA DALI docs — GPU-accelerated data loading / nvJPEG decode, the fix when CPU decode is the
    wall. <https://docs.nvidia.com/deeplearning/dali/user-guide/docs/>
  - WebDataset repo — tar-shard streaming format for sequential, object-store-friendly reads at
    scale. <https://github.com/webdataset/webdataset>
  - NVIDIA — "Case Study: ResNet50 with DALI" — first-party before/after benchmark methodology
    across DGX-2/P3/TITAN RTX. <https://developer.nvidia.com/blog/case-study-resnet50-dali/>

- **Real-world engineering blogs**
  - Uber — "Accelerating Deep Learning: How Uber Optimized Petastorm for High-Throughput and
    Reproducible GPU Training" — the 22h→3h, 12%→60%+, ~80% cost-cut result.
    <https://www.uber.com/us/en/blog/accelerating-deep-learning/>
  - Databricks — "MosaicML StreamingDataset: Fast, Accurate Streaming of Training Data from Cloud
    Storage" — a second production system solving object-store-latency starvation.
    <https://www.databricks.com/blog/mosaicml-streamingdataset>
  - AWS — "Accelerate computer vision training using GPU preprocessing with NVIDIA DALI on Amazon
    SageMaker" — independent reproduction, 37–72% training-time improvement across ResNet18/50/152.
    <https://aws.amazon.com/blogs/machine-learning/accelerate-computer-vision-training-using-gpu-preprocessing-with-nvidia-dali-on-amazon-sagemaker/>

- **Deeper dives**
  - Chaim Rand — "A Caching Strategy for Identifying Bottlenecks on the Data Input Pipeline" —
    a practitioner's profiler-based, component-by-component data-time-attribution walkthrough,
    complementary to this lesson's diagnosis steps.
    <https://chaimrand.medium.com/a-caching-strategy-for-identifying-bottlenecks-on-the-data-input-pipeline-8e52060b402f>
  - LAION-5B dataset writeup — the ~240 TB, all-WebDataset-shards production scale reference.
    <https://laion.ai/blog/laion-5b/>

> **Snapshot (2026-08).** On-demand H100 SXM lands roughly **$3–12/GPU-hr** depending on
> neocloud vs. hyperscaler; the `$40/GPU-hr` used above is a round fully-loaded stand-in. Rates move —
> substitute your own blended number and re-run the arithmetic.
