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
sources: 15
---

# 08.7 · Data pipeline

> **Concept.** A GPU idling on JPEG decode is burning money; find and fix input-pipeline starvation.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

08.6 got the gang expressed, admitted, rendezvoused and restarting correctly. 08.5's
elastic layer keeps it alive across failures. This lesson assumes all of that worked: the
job is running, healthy, gang-scheduled, fully rendezvoused — **and still slow**, for a
reason that has nothing to do with NCCL, checkpointing or the scheduler.

It is the last "is the GPU actually working" diagnosis in the module before 08.8 turns
everything into a dollar figure. 08.3 taught you *comms-bound* (GPUs busy, waiting on the
network). 08.5 taught you *failure-bound* (GPUs gone, waiting on a restart). This lesson
teaches the third shape of wasted GPU-hours: **input-bound** — GPUs idle, waiting on a
Python process to turn a 112 KB JPEG into a tensor.

## Why this matters

You already know how to read a roofline (08.3) and GPU SM% telemetry (module 05). Here is
the failure mode those skills expose that nobody staffs an on-call rotation for: the
accelerator is *starved*. The GPU sits at 20–40% SM-active while CPU cores peg at 100%,
because the input pipeline — read, decode, resize, augment, collate, copy — cannot deliver
batches at the rate the GPU consumes them.

The reason this is *your* problem and not the ML engineer's is arithmetic: **a starved GPU
bills at the full rate.** An 8×H100 node does not get cheaper because the loader is slow.
Landed training progress scales with SM-active, so a job at 40% instead of 90% costs
`0.90/0.40 = 2.25×` per unit of progress. Nothing in the loss curve tells anyone this is
happening. The job completes. It just costs 2.25× and takes 2.25× longer, and only someone
watching utilisation-versus-spend notices.

This is not a hypothetical multiplier. Uber's engineering team documented a production
training pipeline pinned at **10–15% GPU utilisation** with a **22-hour** run, fixed it by
re-architecting the Petastorm-based loader rather than by adding GPUs, and reported **>60%
utilisation, a ~3-hour run, and roughly an 80% reduction in compute cost** for the identical
training job. The model did not change. That is the shape of the win, and it is why the
diagnosis belongs to platform.

## What's new here (calibration)

- **Three causes of low SM%, and how to tell them apart in thirty seconds.** *Comms-bound*
  (08.3: GPUs busy inside a collective, CPU idle), *sync-bound* (a straggler rank at a
  barrier), and *input-bound* (this lesson: CPU pegged, GPU sawtoothing). The CPU signature
  and the shape of the SM% trace separate them. We do not re-teach 08.3's roofline or
  module 05's DCGM field semantics; we add the one diagnosis they deliberately left out.
- **The pipeline as a rate-matching problem, with numbers.** Every stage has a throughput.
  The GPU's demand is a number you can compute. The gap is the bug. Sections 2–5 make this
  a budget you can write down and defend, anchored on MLPerf Storage's published workload
  parameters rather than on rules of thumb.
- **The `DataLoader` is a distributed system.** Worker processes, per-worker index queues,
  round-robin dispatch, a bounded prefetch window, a shared-memory transport, a pinning
  thread, and in-order delivery that head-of-line blocks. Knowing that machine is what turns
  "raise `num_workers`" from a guess into a calculation.
- **Two structural escapes and what each actually costs.** Sharded sequential streaming
  (WebDataset / MosaicML Streaming) converts an IOPS problem into a bandwidth problem.
  GPU-side decode (NVIDIA DALI, nvJPEG) converts a CPU problem into an SM problem. Both are
  trades, not free wins.
- **The FinOps framing, applied to one bottleneck.** The win is only real if recovered
  GPU-hours exceed the cost of the CPUs, RAM, storage tier and engineering time you spent
  recovering them. 08.8 generalises this; here it is one line item you can compute.

## Core concepts

### 1. The problem: a training step has a deadline

A training step is a synchronous consumer. The GPU executes forward, backward and optimizer
in some fixed compute time `t_compute`, then immediately calls `next(loader)` for the next
batch. If the batch is not already sitting in memory, the GPU idles until it is.

```
  ONE STEP'S BUDGET
  ═══════════════════════════════════════════════════════════════════════════════

  HEALTHY (data hidden behind compute)
    GPU  ├────── compute batch n ──────┤├────── compute batch n+1 ──────┤
    CPU  ├─ build n+1 ─┤     idle       ├─ build n+2 ─┤     idle
                                        ▲
                                        batch n+1 already in pinned memory:
                                        step time = t_compute.  SM% ≈ 95%

  STARVED (data on the critical path)
    GPU  ├── compute n ──┤▒▒▒▒ WAIT ▒▒▒▒├── compute n+1 ──┤▒▒▒▒ WAIT ▒▒▒▒┤
    CPU  ├──────── build n+1 ───────────┤──────── build n+2 ───────────┤
                          ▲
                          step time = t_data.  SM% ≈ t_compute / t_data
```

Two consequences follow immediately and are worth stating as rules.

**Rule 1 — only the maximum matters.** If the loader can produce batches at rate
`R_data` (batches/s) and the GPU can consume at `R_gpu`, the achieved rate is
`min(R_data, R_gpu)` and SM-active is `min(1, R_data / R_gpu)`. Making the loader *faster
than* the GPU buys nothing. Making it *slower* costs linearly. This is why "the loader is
fine, it's only 20% slower than the GPU" is not fine: it is a 20% cost increase forever.

**Rule 2 — the pipeline hides latency, not throughput.** Prefetching *n* batches ahead
smooths jitter: if one sample takes 10× as long as usual, the buffer absorbs it. It does
nothing about a sustained rate deficit. A queue in front of an under-provisioned producer
drains at exactly the rate of the deficit and then stays empty forever. **If SM% starts
high and decays over the first minute of an epoch, you are watching a prefetch buffer
drain — that is the signature of a throughput deficit, not a latency spike.**

### 2. The pipeline as a chain of rated stages

Every input pipeline is the same five stages. Draw it with a rate on each edge and the
bottleneck names itself.

```
  THE INPUT PIPELINE, WITH RATES  (8×H100 node, ResNet-50-class vision training)
  ═════════════════════════════════════════════════════════════════════════════════

   ① STORAGE          ② TRANSPORT       ③ DECODE         ④ AUGMENT      ⑤ H2D COPY
   object store       page cache /       JPEG → RGB       crop/flip/     pinned host
   or filesystem      socket buffer      (libjpeg)        normalise      → device
        │                  │                  │                │              │
        │ 114,660 B        │                  │  ~270–840      │  ~2–5×       │ ~25 GB/s
        │ per sample       │                  │  img/s/core    │  cheaper     │ over PCIe
        │                  │                  │  (decode only) │  than decode │ Gen5 x16
        ▼                  ▼                  ▼                ▼              ▼
   ┌─────────┐        ┌─────────┐       ┌─────────┐      ┌─────────┐   ┌─────────┐
   │ 1.6 GB/s│───────▶│         │──────▶│ 48 cores│─────▶│         │──▶│  DMA    │
   │ needed  │        │         │       │ ≈14k/s  │      │         │   │         │
   └─────────┘        └─────────┘       └─────────┘      └─────────┘   └─────────┘
                                                                              │
                                                                              ▼
                                                                    ┌──────────────────┐
                                                                    │  8 × H100        │
                                                                    │  DEMAND          │
                                                                    │  14,286 img/s    │
                                                                    │  = 1.64 GB/s     │
                                                                    └──────────────────┘

   THE STARVATION POINT is wherever a stage's rate < 14,286 img/s.
   With 12 decode cores instead of 48:  3,600 img/s  ⇒ SM-active ≈ 25%
   With an S3 bucket at 4 connections:  ~350 MB/s ⇒ 3,050 img/s ⇒ SM-active ≈ 21%
   With 1M small files at 5,500 GET/s/prefix:  IOPS-capped, not byte-capped.
```

The numbers in that diagram are derived in §3 and §4; none of them are guesses. What
matters structurally is that **stages ①–④ run on the CPU and the network, stage ⑤ is a DMA
engine, and only the GPU box is what you are paying $3–7 per hour for.** Every stage that
falls below the demand line converts that spend into waiting.

### 3. Computing the demand: how many bytes/second must arrive

You do not have to guess the GPU's appetite. It is
`demand [B/s] = N_gpu × (samples/s per GPU) × bytes_per_sample`, and both factors are
measurable — or, if you have not measured yet, available from MLPerf Storage's published
workload configurations, which exist precisely to answer this question.

The MLPerf Storage benchmark (`mlcommons/storage`) encodes each workload as a per-accelerator
compute time and a record size. Reading `configs/dlio/workload/*.yaml` directly:

| Workload | Accelerator | `batch_size` | `computation_time` (s) | `record_length_bytes` | Target AU |
|---|---|---|---|---|---|
| ResNet-50 | H100 | 400 | 0.224 | 114,660 | 0.90 |
| ResNet-50 | A100 | 400 | 0.435 | 114,660 | 0.90 |
| 3D U-Net | H100 | 7 | 0.323 | 146,600,628 | 0.90 |
| 3D U-Net | A100 | 7 | 0.636 | 146,600,628 | 0.90 |
| CosmoFlow | H100 | 1 | 0.00350 | 2,828,486 | 0.70 |

Now carry the units through. Per accelerator:

```
  samples_per_s   = batch_size / computation_time
  bandwidth_ideal = samples_per_s × record_length_bytes
  bandwidth_req   = bandwidth_ideal × AU        ← the rate that just sustains the
                                                   benchmark's utilisation floor

  ResNet-50 / H100
    400 / 0.224 s              = 1,785.7 samples/s per GPU
    × 114,660 B                = 204.7 MB/s per GPU     (ideal)
    × 0.90                     = 184.3 MB/s per GPU     (at the 90% AU floor)
    × 8 GPUs                   = 1.64 GB/s per node
    × 1,024 GPUs               = 210 GB/s for the cluster

  3D U-Net / H100
    7 / 0.323 s                = 21.7 samples/s per GPU
    × 146,600,628 B            = 3.18 GB/s per GPU      (ideal)
    × 8 GPUs                   = 25.4 GB/s per node      ← a parallel filesystem,
                                                            not an object store

  CosmoFlow / H100
    1 / 0.00350 s              = 285.7 samples/s per GPU
    × 2,828,486 B              = 808 MB/s per GPU        (ideal)
    × 0.70                     = 566 MB/s per GPU        (at the 70% AU floor)
```

Those bottom lines — roughly **190 MB/s, 3 GB/s and 0.5 GB/s per H100** for ResNet-50, 3D
U-Net and CosmoFlow respectively — are what MLPerf Storage's published per-accelerator
requirements amount to, and the derivation above reproduces them from the shipped config
files to within rounding. **A 16× spread across three ordinary vision workloads on
identical hardware is the point.** "How much storage bandwidth does training need" has no
answer; "how much does *this* workload need on *this* accelerator" has an exact one.

**The LLM contrast, because it inverts the intuition.** Take an 8B-parameter dense model on
H100s sustaining 400 TFLOP/s per GPU (08.3's typical figure). The `6ND` rule (08.1) says
each token costs `6 × 8×10⁹ = 4.8×10¹⁰` FLOP, so:

```
  tokens/s per GPU = 400×10¹² / 4.8×10¹⁰  = 8,333 tokens/s
  pre-tokenised at 4 B per token id       = 33.3 kB/s per GPU
  × 1,024 GPUs                            = 34 MB/s for the whole cluster
```

**34 MB/s.** One S3 connection delivers three times that. LLM pretraining is not
read-bandwidth-bound and never was — which is why the LLM data-pipeline problems that
actually bite are *different ones*: online tokenisation burning CPU, random access into a
globally shuffled index over trillions of tokens (an IOPS and metadata problem), and
checkpoint write bandwidth (08.4). If you carry "data loading is a bandwidth problem" from
a vision background into an LLM shop you will instrument the wrong thing. Note the ratio: 3D
U-Net needs about **100,000×** the per-GPU read bandwidth of 8B LLM pretraining.

### 4. Computing the supply: cores, IOPS and the storage tier

Now the other side of the ledger.

**Decode.** A published multi-decoder benchmark on ImageNet-scale JPEGs (arXiv:2501.13131)
measures single-thread decode throughput spanning roughly **270 img/s** (Arm Neoverse N1)
to **840 img/s** (Zen 5), with libjpeg-turbo-family decoders and OpenCV clustered near the
top on every architecture. That is *decode only*; resize, crop, colour jitter, normalise and
collate add meaningfully on top — assume 1.5–3× the decode cost for a standard augmentation
stack unless you have measured yours. So budget **150–400 img/s per core** end-to-end and
then measure.

Against the ResNet-50 demand of 14,286 img/s for an 8-GPU node:

```
  cores_needed = 14,286 img/s ÷ 300 img/s/core ≈ 48 cores  (just for the input pipeline)
```

A typical 8×H100 node ships 96–224 vCPUs, i.e. 12–28 vCPUs per GPU. Forty-eight cores for
the loader is a large but feasible fraction — and it is *why* those nodes have that many
cores. It also explains the most common misconfiguration in the field: a Kubernetes pod that
requests `nvidia.com/gpu: 8` and `cpu: "16"` has capped its input pipeline at roughly
`16 × 300 = 4,800 img/s`, one third of what the GPUs can eat, before a single line of
Python runs. **The CPU request on a training pod is a throughput decision, not a
formality.**

**Bytes versus requests.** Storage systems have two independent limits and training
pipelines usually die on the wrong one. Reading 1.28M individual JPEGs of ~112 KB each:

| Limit | Amazon S3 (published guidance) | What it means here |
|---|---|---|
| Per-connection throughput | ~85–90 MB/s per request/connection | 1.64 GB/s needs **≈19 concurrent connections**, minimum |
| Request rate | 5,500 GET/HEAD per second **per prefix** | 14,286 img/s needs **≥3 prefixes**, or you are rate-limited regardless of bandwidth |
| Scaling behaviour | scales up gradually; returns HTTP 503 Slow Down while adapting | a burst at epoch start throttles even when the steady state is fine |

That table is the whole IOPS-versus-bandwidth argument in concrete form. At 112 KB per
object you need 14,286 requests/s to move 1.64 GB/s — three prefixes' worth of request
budget for less than two gigabytes per second. Pack the same bytes into 1 GB shards and you
need **1.6 requests/s**. The bytes are identical; the request count fell by four orders of
magnitude. **This is why "put it on faster storage" so often fails to fix a starved
pipeline: a faster device still pays a per-request cost on the same number of requests.**

**Which tier delivers it.** Numbers below are published product characteristics as of
2026-08; substitute your own provider's.

| Tier | Realistic sustained read | Request/latency behaviour | Fits which demand |
|---|---|---|---|
| Page cache (node RAM) | 10s of GB/s | ~µs | anything — but only if the working set fits, which it usually does not |
| Local NVMe SSD (per drive) | ~3–7 GB/s sequential | ~100 µs, ~10⁵–10⁶ IOPS | 3D U-Net node demand (25 GB/s) with several drives striped |
| FSx for Lustre, Persistent-2 SSD | provisioned **125 / 250 / 500 / 1000 MB/s per TiB** | parallel FS, POSIX | 25 GB/s needs e.g. 25 TiB at 1000 MB/s/TiB — capacity you buy *for throughput* |
| FSx for Lustre, Persistent-1 SSD | **50 / 100 / 200 MB/s per TiB** | ″ | the older tier; the same capacity buys 5× less bandwidth |
| S3 / object store | ~85–90 MB/s **per connection**, aggregate scales with connections and prefixes | 100s of ms first-byte; 5,500 GET/s per prefix | fine at 1.6 GB/s with ~20+ connections **if** objects are large |

The Lustre row is the one people miss in planning: on a per-TiB-provisioned throughput
model, **bandwidth is bought as capacity**. A 25 GB/s requirement on Persistent-2 at
1000 MB/s/TiB means provisioning ≈25 TiB whether or not your dataset is that big. That is a
line item, and 08.8's model has a place for it.

**A caching caveat that invalidates most homemade benchmarks.** If your dataset fits in the
node's page cache, your second epoch reads from RAM and your measurement is meaningless.
MLPerf Storage encodes this as a validation rule: the generated dataset must be at least
`HOST_MEMORY_MULTIPLIER = 5` times the host's memory, and the submission validator
recomputes the required dataset size and *fails the run* if it does not match. Adopt the
same discipline: **benchmark on ≥5× RAM of data, or you are benchmarking your page cache.**

### 5. The PyTorch `DataLoader`, mechanically

`num_workers > 0` turns the loader into a small multiprocess system. Knowing its parts is
what makes the knobs predictable. Read off `torch/utils/data/dataloader.py` at `main`
(PyTorch 2.15 development head):

```
  _MultiProcessingDataLoaderIter — WHAT ACTUALLY RUNS
  ═══════════════════════════════════════════════════════════════════════════════

   main process                                worker processes (num_workers)
   ─────────────                               ──────────────────────────────
   sampler ──▶ _next_index()
        │                                        ┌── worker 0 ──┐
        │  _try_put_index():                     │ loop:        │
        │    max_tasks = prefetch_factor         │  idx = q.get │
        │              × num_workers             │  batch =     │
        │    round-robin over an itertools.cycle │   collate(   │
        │    of worker queue indices             │    [ds[i]    │
        ▼                                        │      for i]) │
   index_queues[w] ──────────────────────────────▶  put(batch)  │
   (one multiprocessing.Queue PER WORKER)        └──────┬───────┘
                                                        │  tensors are moved
   _worker_result_queue  ◀───────────────────────────────┘  through /dev/shm
        │                                                    (file-descriptor
        │                                                     passing, not copy)
        ▼
   pin_memory thread  (only if pin_memory=True)
        │   copies each tensor into page-locked host memory
        ▼
   _data_queue ──▶ __next__() returns to the training loop

   REORDER BUFFER: with in_order=True (default), __next__ returns batches in
   sampler order. A batch that arrives out of order is parked in _task_info
   until its predecessor arrives.  ⇒ ONE SLOW WORKER STALLS EVERYTHING.
```

Three mechanisms in that picture explain most real behaviour.

**The prefetch window is a hard cap.** `_try_put_index` asserts
`_tasks_outstanding < prefetch_factor × num_workers`. That product is the total number of
batches that can be in flight — being built, sitting in a queue, or waiting in the reorder
buffer. It is simultaneously your jitter absorber and your memory consumption:
`prefetch_factor × num_workers × sizeof(batch)` bytes are resident in shared memory at
steady state. With 32 workers, `prefetch_factor=4`, and 400×3×224×224 float32 images
(230 MB per batch) that is **29 GB of `/dev/shm`**, which is where the classic failure
comes from (below).

**Dispatch is round-robin, not work-stealing.** The main process cycles through worker
queue indices and hands the next batch to the next *active* worker regardless of how loaded
it is. Combined with in-order delivery, one worker that hits a pathological sample — a
20-megapixel JPEG, an NFS stall, a page fault — blocks delivery of every batch behind it.
That is what a periodic sawtooth in SM% with a period of `num_workers` steps looks like.
`in_order=False` changes exactly this: `_try_put_index` then skips a worker whose
outstanding-task count is at or above its fair share, and `__next__` returns batches as they
finish. You trade a reproducible sample order for immunity to one slow worker. (`in_order`
is a relatively recent addition; check that your PyTorch has it before designing around it.)

**Workers communicate through shared memory, and containers cap it at 64 MiB.** Worker
processes write result tensors into `/dev/shm` and pass file descriptors, not bytes, back to
the parent. Container runtimes default `/dev/shm` to **64 MiB**. Exceed it and you get
`RuntimeError: DataLoader worker (pid N) is killed by signal: Bus error` — a SIGBUS from
writing past the end of a tmpfs mapping, which reads as a mysterious crash rather than as
"out of space". The fix in Kubernetes is the `emptyDir{medium: Memory}` mount at `/dev/shm`
that the 08.6 `PyTorchJob` YAML already carries; size it to at least
`prefetch_factor × num_workers × sizeof(batch)`.

**Every knob, with its real default:**

| Knob | Default | What it changes | The wall it hits |
|---|---|---|---|
| `num_workers` | **0** (load in the main process — guaranteed starvation with any real transform) | parallel batch construction | physical cores; RAM; `/dev/shm`; per-worker dataset copy |
| `prefetch_factor` | **`None` if `num_workers==0`, else 2** | in-flight batches = `prefetch_factor × num_workers` | RAM/`/dev/shm`; raises latency-to-first-batch |
| `pin_memory` | **False** | stages batches in page-locked memory so H2D is an async DMA | pinned memory is unswappable; over-pinning starves the node |
| `pin_memory_device` | `""` — **deprecated**; the current accelerator is used | — | — |
| `persistent_workers` | **False** | keeps workers alive across epochs | holds memory between epochs; skips per-epoch dataset re-init |
| `in_order` | **True** | strict sampler-order delivery | `False` removes head-of-line blocking but skews sample order |
| `timeout` | **0** (wait forever) | seconds to wait for a batch before raising | a nonzero value converts a hung mount into an exception |
| `batch_size` | **1** | samples per batch | — |
| `drop_last` | **False** | drop a short final batch | a ragged last batch can desync collectives in DDP |
| `multiprocessing_context` | `None` (OS default: `fork` on Linux) | how workers start | `fork` after a CUDA context exists corrupts state; `spawn` re-imports and is slower to start |

`pin_memory` deserves one sentence of mechanism because it is routinely set without its
partner. A normal (pageable) host buffer cannot be the source of an async DMA, because the
OS may move the page mid-transfer; CUDA therefore stages it through an internal pinned
buffer synchronously. `pin_memory=True` allocates the tensor in page-locked memory up front,
which lets `cudaMemcpyAsync` run on the copy engine concurrently with compute — **but only
if you also issue the copy as `tensor.to(device, non_blocking=True)`.** With a blocking
`.to(device)` you have paid for pinning and got none of the overlap.

### 6. Diagnosing it in four steps

```
  "SM% IS LOW." — WHICH KIND OF LOW?
  ═══════════════════════════════════════════════════════════════════════════════

  ① SHAPE OF THE SM% TRACE          nvidia-smi dmon -s u   |   DCGM_FI_PROF_SM_ACTIVE
     ────────────────────────                                    (fraction of time
      ╱╲  ╱╲  ╱╲  ╱╲   SAWTOOTH ──▶ input-bound or sync-bound      ≥1 warp resident)
     ╱  ╲╱  ╲╱  ╲╱  ╲              (compute, stall, compute)
     ──────────────   FLAT-LOW  ──▶ occupancy/kernel problem, or a
                                     long-running collective (08.3)

  ② THE CPU SIGNATURE               mpstat -P ALL 1
     ────────────────────
     one or more cores at ~100% %usr, in python processes  ──▶ INPUT-BOUND
     all cores idle, GPU still low                          ──▶ comms- or sync-bound;
                                                                 go back to 08.3

  ③ ATTRIBUTE THE STEP              torch.profiler, or a hand timer around next()
     ─────────────────
     t_data  = seconds blocked in next(loader)
     t_step  = seconds per full iteration
     data_fraction = t_data / t_step        ──▶ this single number is the before/after
                                                 evidence for the whole fix

  ④ CONFIRM WITH A SYNTHETIC-DATA A/B          ← the decisive test
     ────────────────────────────────
     replace the dataset with one that returns a preallocated random tensor.
     • step time collapses  ⇒ the pipeline was the bottleneck. Ceiling now known.
     • step time unchanged  ⇒ it was never the pipeline. Stop tuning workers.
```

Step ④ is the one to internalise, because it is cheap, unambiguous, and gives you the
ceiling. A ten-line `class Synthetic(Dataset)` that returns `torch.empty(3,224,224)` from
`__getitem__` tells you `R_gpu` exactly. Everything after that is closing the gap between
`R_data` and a number you now know.

Two measurement traps worth naming. **Timing only `next(loader)` under-reports**, because
prefetching means the wait is often absorbed and reappears later as a slower H2D copy or a
stalled CUDA stream — profile with `torch.profiler` and look at gaps on the CUDA timeline,
not just at Python wall-clock. **And do not benchmark epoch 2**: by then the page cache is
warm and you are measuring RAM (see the `HOST_MEMORY_MULTIPLIER = 5` rule in §4).

Once you know it is input-bound, find the hot stage by ablation: comment out augmentations
one at a time, swap the decoder, or point the dataset at a local copy of 1,000 files. The
usual culprits, in order of frequency: (a) millions of small files, so you are IOPS-bound
and no amount of CPU helps; (b) a slow decoder, so you are core-bound; (c) one expensive
augmentation on the critical path; (d) `num_workers` capped by a container CPU limit you
forgot about.

### 7. Structural fix 1 — sharding: turn IOPS into bandwidth

When you are request-bound rather than byte-bound, the only real fix is to issue fewer,
larger requests. **WebDataset** does this by storing samples in POSIX **tar archives** —
"shards", conventionally a few hundred MB to a few GB — read strictly sequentially and
streamed. Files belonging to one sample share a basename and differ by extension
(`000123.jpg`, `000123.cls`, `000123.json`), so the reader groups consecutive tar members
into a sample dictionary without any index, seek or metadata lookup.

```python
import webdataset as wds
from torch.utils.data import DataLoader

url = "pipe:aws s3 cp s3://bucket/imagenet/train-{000000..001023}.tar -"

dataset = (
    wds.WebDataset(url, shardshuffle=100, nodesplitter=wds.split_by_node)
       .shuffle(2000)              # WINDOW shuffle: a 2000-sample reservoir.
                                   # NOT a global shuffle — see the pitfall below.
       .decode("pil")
       .to_tuple("jpg", "cls")
       .map(preprocess)
       .batched(64)                # batch INSIDE the worker, not in the DataLoader
)

loader = DataLoader(dataset, batch_size=None, num_workers=8, pin_memory=True)
```

Three mechanisms behind that snippet.

**Sharding across ranks is integer striping, and it constrains your shard count.**
`split_by_node` and `split_by_worker` are implemented as
`islice(src, rank, None, world_size)` and `islice(src, worker, None, num_workers)` — plain
strided slices over the shard list. So the shard list is partitioned `world_size ×
num_workers` ways. **If you have fewer shards than `world_size × num_workers`, some workers
get an empty list, produce nothing, and in a DDP job the ranks that ran out of data hang at
the next all-reduce waiting for ranks that still have data.** The rule that falls out: make
the shard count a comfortable multiple of `world_size × num_workers` — 1,024 shards for a
64-rank job with 8 workers each (512 slices) is a reasonable ratio. Uneven shard sizes cause
the same hang more subtly, which is why `.repeat().with_epoch(n)` or `resampled=True` are
the recommended patterns for DDP.

**The shuffle is a two-level approximation of a global shuffle.** `shardshuffle=100`
shuffles the *order of shards* within a window of 100; `.shuffle(2000)` maintains a
2000-sample reservoir and emits a random element from it as each new sample arrives. The
composition is good enough for most supervised training and materially different from a
global permutation. Where sample correlation within a shard matters — some contrastive
setups, or any dataset written in sorted-by-class order — this is a real statistical
trade, and the mitigation is to shuffle *when writing the shards* so that within-shard
correlation is already destroyed.

**The cost is a repack step.** Converting a dataset to shards is a one-off map job whose
cost is proportional to the dataset size, plus the duplicate storage while both copies
exist. Budget it explicitly.

**MosaicML Streaming** (Databricks) is an independently built production system aimed at
exactly the same failure — object-store latency starving GPUs — with a different design
point: a binary shard format plus an index that supports deterministic resumption
mid-epoch and elastic world sizes. That two organisations converged on "shard and stream"
is the durable signal; which library you pick is a local decision.

### 8. Structural fix 2 — GPU-side decode with DALI

If you are core-bound rather than request-bound, the other escape is to stop using CPU
cores. NVIDIA **DALI** runs the decode-and-augment graph on the GPU, using the nvJPEG
library and — on Ampere and later — a **dedicated hardware JPEG decode engine** (the A100
has a 5-core engine) that is physically separate from the SMs.

```python
from nvidia.dali import pipeline_def, fn, types
import nvidia.dali.plugin.pytorch as dali_torch

@pipeline_def(
    batch_size=256,
    num_threads=8,          # CPU-stage thread pool. NO DEFAULT — you must set it.
    device_id=0,
    prefetch_queue_depth=2, # DEFAULT 2. Pipeline-level lookahead, the DALI analogue
                            # of prefetch_factor.
    py_num_workers=1,       # DEFAULT 1. Only used by parallel external_source.
    py_start_method="fork", # DEFAULT "fork" — must start workers BEFORE any CUDA
                            # context exists, or use "spawn".
)
def train_pipe(shard_id, num_shards):
    jpegs, labels = fn.readers.webdataset(
        paths=["/data/train-{000000..001023}.tar"],
        ext=["jpg", "cls"],
        random_shuffle=True,   # DEFAULT False
        initial_fill=1024,     # DEFAULT 1024 — the shuffle reservoir size
        prefetch_queue_depth=1,# DEFAULT 1 — the READER's own lookahead, distinct
                               # from the pipeline's
        read_ahead=False,      # DEFAULT False; True helps large sequential files
        dont_use_mmap=True,    # DEFAULT False. Set True on network filesystems:
                               # mmap only pays off on local disks.
        shard_id=shard_id, num_shards=num_shards,
        stick_to_shard=False,  # DEFAULT False
        pad_last_batch=True,   # DEFAULT False; True keeps every rank's batch count
                               # equal, which matters for DDP
        name="Reader",
    )
    images = fn.decoders.image_random_crop(
        jpegs,
        device="mixed",              # "mixed" = CPU parse + GPU/HW decode.
                                     # "cpu" keeps it all on the host.
        output_type=types.RGB,       # DEFAULT RGB
        hw_decoder_load=0.65,        # DEFAULT 0.65 — fraction of images routed to the
                                     # hardware JPEG engine (Ampere+). Tune empirically.
        hybrid_huffman_threshold=1000*1000,  # DEFAULT 1e6 px: images larger than this
                                     # use the hybrid Huffman path; smaller ones use
                                     # the host-side one.
        device_memory_padding=16*1024*1024,  # DEFAULT 16 MiB per thread — preallocated
        host_memory_padding=8*1024*1024,     # DEFAULT 8 MiB × 2 (double buffered)
                                     # Both exist to avoid reallocation stalls when a
                                     # larger-than-expected image arrives.
    )
    images = fn.resize(images, resize_x=224, resize_y=224)
    images = fn.crop_mirror_normalize(
        images,
        dtype=types.FLOAT,
        output_layout="CHW",
        mean=[0.485*255, 0.456*255, 0.406*255],
        std=[0.229*255, 0.224*255, 0.225*255],
        mirror=fn.random.coin_flip(probability=0.5),
    )
    return images, labels

pipe = train_pipe(shard_id=rank, num_shards=world_size)
loader = dali_torch.DALIGenericIterator(
    [pipe], ["data", "label"],
    reader_name="Reader",
    auto_reset=True,
    last_batch_policy=dali_torch.LastBatchPolicy.PARTIAL,
)
```

**What `device="mixed"` actually does.** The JPEG bitstream is parsed on the host (headers,
Huffman tables), then the entropy-decode and IDCT stages run on the GPU — on the dedicated
hardware engine for the fraction of images selected by `hw_decoder_load`, and on SM-based
nvJPEG kernels for the rest. `hybrid_huffman_threshold` splits large from small images
because the hybrid path's GPU Huffman decode only pays off above roughly a megapixel. The
result tensor is already in device memory, so **stage ⑤ of §2 disappears entirely** — there
is no host-to-device copy of decoded pixels, only of compressed bytes, which are ~10× smaller.

**What it costs.** DALI consumes GPU memory (the padding parameters above preallocate per
thread), consumes SMs for the software decode fraction and all augmentation, and adds a
dependency with its own pipeline API that your training loop must adopt. NVIDIA's own
ResNet-50 case study and an independent AWS SageMaker reproduction both report a *range* of
end-to-end improvement — from single-digit percent on a lightly CPU-bound configuration up
to roughly 70% under heavy augmentation — not a uniform win. NVIDIA's A100 hardware-decoder
material reports up to ~20× faster decode than CPU-only and around 7,000 img/s with
`hw_decoder_load` near 0.75. Treat all of these as vendor-published and configuration-dependent.

**The decision rule is mechanical:** GPU-side decode helps if and only if the CPU was the
bottleneck *and* the GPU has SM headroom. Applying it to a job that was comms-bound (08.3)
or already compute-saturated spends SMs to buy nothing. Confirm with §6 step ④ first.

### 9. Overlap, and why the last 5% is a stream problem

Even with supply ≥ demand, you can leave SM% on the table if the host-to-device copy is
serialised with compute. The full overlap recipe is three things together: `pin_memory=True`
on the loader (page-locked source buffer), `.to(device, non_blocking=True)` at the call site
(async DMA on a copy engine), and — for the last increment — issuing the copy for batch
*n+1* on a separate CUDA stream while batch *n* computes, with an event to synchronise.
PyTorch's `torch.cuda.Stream` plus a small prefetcher class is the standard implementation.
Expect this to buy single-digit percent, and only after the throughput problem is solved;
it is a polish step, not a fix.

### 10. What starvation costs, as a formula

Everything above reduces to one line that 08.8 will fold into a larger model:

```
  SM_active            = min(1, R_data / R_gpu)
  cost_multiplier      = 1 / SM_active
  GPU-hours wasted     = N_gpu × wall_hours × (1 − SM_active)
  $ wasted             = GPU-hours wasted × r          [r = $/GPU-hr]

  and the fix is only worth doing if
      $ recovered  >  Δ(CPU cores) × r_cpu  +  Δ(storage tier)  +  engineering
```

The asymmetry that makes this almost always worth doing: an H100 rents for roughly
**$2–7/GPU-hr** on specialist clouds and up to **~$7–12/GPU-hr** at hyperscaler list price
(2026-08 market snapshot; substitute your own rate), while a vCPU-hour is measured in *cents*.
Trading 30 extra vCPUs to recover half an H100's output is a trade at roughly 100:1 in your
favour. That ratio, not any specific benchmark, is why this lesson exists.

## Perspectives

**Developer / ML-engineer view.** From the research seat, starvation is nearly invisible:
the job runs, the loss curve is normal, checkpoints land, and "training is slow today" does
not trigger the alarm that a crash does. Researchers optimise for iteration speed and
augmentation correctness, not GPU-hour accounting. Unless someone watches SM% against spend,
a starved pipeline can run for months.

**Operator view.** The signature is a two-metric join you should have on a dashboard:
`DCGM_FI_PROF_SM_ACTIVE` low **and** container CPU throttling or `%usr` high on the same
pod. Either one alone is ambiguous; together they are diagnostic. Add the container CPU
*limit* to the panel — a job throttled by its own cgroup quota looks exactly like a job
short of cores, and the fix is a one-line manifest change.

**Storage view.** The MLPerf Storage per-accelerator numbers (§3) are the cleanest existing
statement that "how fast must storage be" is a per-workload question with a 16× spread. They
also make the buy-side argument concrete: 3D U-Net on eight H100s needs 25 GB/s from one
node, which on a per-TiB-provisioned parallel filesystem means buying capacity you do not
need in order to get bandwidth you do.

**GPU-versus-CPU cost-asymmetry view.** The 100:1 rate ratio between an H100-hour and a
vCPU-hour is the single most useful number in this lesson. It means the correct default
posture is *over*-provision the input pipeline: the cost of 20 wasted cores is invisible
next to the cost of one GPU at 60%.

**Interview view.** "GPU at 100% utilisation but low SM-active, CPU pegged" is a standard
probe. The answer that lands is not "increase num_workers" — it is the rate-matching frame:
compute the GPU's demand in samples/s and bytes/s, measure each stage's supply, name the
one that is short, and only then pick between more cores, a different format, or GPU-side
decode. Then price it.

## Real-world use cases

- **Uber — "Accelerating Deep Learning: How Uber Optimized Petastorm for High-Throughput
  and Reproducible GPU Training"** — <https://www.uber.com/us/en/blog/accelerating-deep-learning/>.
  A production Parquet/Petastorm pipeline pinned at **10–15% GPU utilisation** with a
  **22-hour** run; network I/O and CPU-bound transforms were the constraint. After
  re-architecting the loader: **>60% utilisation, ~3-hour run, ~80% lower compute cost**,
  same model. What it shows: the biggest available win in a training platform is often not
  in the model or the network but in the loader — and it is a cost line, not a perf footnote.
  *uber.com is blocked from this build environment; figures are as reported by the blog and
  consistently reproduced in secondary coverage.*
- **MLCommons — MLPerf Storage (`mlcommons/storage`)** — <https://github.com/mlcommons/storage>.
  The workload configs in `configs/dlio/workload/` are the primary source for §3's demand
  table, and `Rules.md` is the source of the accelerator-utilisation definition
  (`AU = total_compute_time / total_benchmark_running_time`) and the
  `HOST_MEMORY_MULTIPLIER = 5` anti-cache rule. What it shows: an industry-standard way to
  state a storage requirement per accelerator per workload, and a validator that refuses
  submissions whose dataset is small enough to be cached.
- **Databricks/MosaicML — StreamingDataset** — <https://www.databricks.com/blog/mosaicml-streamingdataset>.
  A second production system independently built for object-store-latency starvation, with
  deterministic mid-epoch resumption and elastic world sizes. What it shows: "shard and
  stream" is the converged answer, not a single vendor's opinion. *Blocked from this
  environment; not fetched in this pass.*
- **NVIDIA — DALI hardware-decoder material and the ResNet-50 case study** —
  <https://developer.nvidia.com/blog/leveraging-hardware-jpeg-decoder-and-nvjpeg-on-a100/>
  and <https://developer.nvidia.com/blog/case-study-resnet50-dali/>. The A100's 5-core
  hardware JPEG engine, ~20× decode speedup versus CPU-only, ~7,000 img/s at
  `hw_decoder_load ≈ 0.75`, and an end-to-end improvement range that depends heavily on
  configuration. What it shows: the offload is real, the win is not automatic.
  *developer.nvidia.com is blocked from this environment; these figures were confirmed via
  search against NVIDIA's own posts, not by fetching them, and the DALI defaults quoted in
  §8 come from the DALI source tree instead.*
- **AWS — "Accelerate computer vision training using GPU preprocessing with NVIDIA DALI on
  Amazon SageMaker"** — an independent reproduction reporting **37–72%** training-time
  improvement across ResNet-18/50/152. What it shows: a second party measuring the same
  wide range, which is the honest way to quote DALI. *aws.amazon.com blog is blocked here;
  cited as reported, not fetched.*

## Worked example

**The run.** One node, 8×H100 SXM, ResNet-50-class vision training, ImageNet-scale dataset
of 1.28M JPEGs averaging 114,660 B. 90 epochs. Blended rate `r = $3.00/GPU-hr` — inside the
2026-08 observed on-demand band for H100 SXM on specialist clouds; **substitute your own and
re-run every line below.**

**Step 1 — the ceiling (synthetic-data A/B).** Replace the dataset with a synthetic one and
measure. Using MLPerf Storage's H100 ResNet-50 parameters as the reference compute rate
(`batch 400 / 0.224 s`):

```
  R_gpu  = 1,785.7 img/s per GPU × 8 = 14,286 img/s per node
  epoch  = 1,280,000 / 14,286        = 89.6 s
  90 epochs                          = 8,064 s = 2.24 h wall clock
  GPU-hours                          = 8 × 2.24 = 17.9 GPU-hr
  cost floor                         = 17.9 × $3.00 = $53.76 per run
```

**Step 2 — the demand on storage.**

```
  bytes/s = 14,286 img/s × 114,660 B = 1.638 GB/s per node

  Delivered by:
    • local NVMe (3–7 GB/s/drive)                    ✓ comfortably, one drive
    • FSx Lustre Persistent-2 @ 250 MB/s/TiB          ✓ needs ≥ 6.6 TiB provisioned
    • S3, large shards                                ✓ needs ≥ ⌈1638/87⌉ = 19 concurrent
                                                        connections
    • S3, one GET per 112 KB image                    ✗ 14,286 GET/s needs ≥ 3 prefixes
                                                        (5,500 GET/s per prefix) AND 19+
                                                        connections. Feasible, fragile,
                                                        and 503-prone on epoch-start bursts.
```

**Step 3 — the supply from CPU.** Suppose the pod requests `cpu: "16"` and the augmentation
stack runs at 300 img/s/core end-to-end:

```
  R_data = 16 cores × 300 img/s = 4,800 img/s
  SM_active = min(1, 4800 / 14286) = 0.336
  step time = 400 / (4800/8) = 0.667 s   (vs 0.224 s compute-bound)
  epoch  = 1,280,000 / 4,800 = 266.7 s
  90 epochs = 24,000 s = 6.67 h
  GPU-hours = 8 × 6.67 = 53.3 GPU-hr
  cost = 53.3 × $3.00 = $160.00 per run
```

**The starvation tax is `$160.00 − $53.76 = $106.24` per run, a 2.98× multiplier.**
Confirm it independently: `1 / 0.336 = 2.98`. ✓

**Step 4 — fix A: give the pipeline the cores it needs.**

```
  cores needed = 14,286 / 300 = 47.6 → request 52 vCPU (with headroom for the
                                        training process itself and the pinning thread)
  set: num_workers=48, prefetch_factor=4, pin_memory=True, persistent_workers=True
  /dev/shm needed ≥ 48 × 4 × sizeof(batch)
                  = 192 × (50 img × 3 × 224 × 224 × 4 B)   [per-worker batch of 50]
                  = 192 × 30.1 MB = 5.8 GB   ⇒ mount an 8Gi memory-backed emptyDir.
                                                (The 64 MiB container default would
                                                 SIGBUS in the first seconds.)

  R_data = 48 × 300 = 14,400 img/s ≥ 14,286  ⇒ SM_active ≈ 0.95 (residual copy overhead)
  cost   ≈ 17.9 / 0.95 × $3.00 = $56.6 per run
  recovered = $160.00 − $56.6 = $103.4 per run per node
  cost of the fix: 36 extra vCPU-hours × 2.24 h ≈ 81 vCPU-hr. At a generous
                   $0.05/vCPU-hr that is $4.05.
  ⇒ net $99 recovered for $4 spent, a 24:1 return, per run, per node.
```

**Step 5 — fix B: when the cores are not available.** Suppose the node type caps you at 24
usable vCPUs, so `R_data = 7,200 img/s` and `SM_active = 0.50` — a $107.5 run. Move decode
to the GPU with the §8 DALI pipeline. The CPU now only parses JPEG headers and feeds
compressed bytes; call it 2,000 img/s per core, so 24 cores is no longer the constraint. The
GPU pays: the hardware JPEG engine handles `hw_decoder_load = 0.65` of the images, nvJPEG
kernels the rest, and augmentation runs on SMs. Assume this costs 8% of SM time:

```
  effective compute rate = 14,286 × 0.92 = 13,143 img/s
  epoch = 1,280,000 / 13,143 = 97.4 s ; 90 epochs = 2.44 h ; 19.5 GPU-hr
  cost = 19.5 × $3.00 = $58.5   (vs $107.5 starved)
  recovered = $49 per run per node — bought with SMs rather than cores.
```

Note the decision this makes explicit: DALI's 8% SM tax is a *bargain* against a 50%
starvation loss and a *waste* against a 5% one. Measure first (§6 step ④).

**Step 6 — scale it, which is the number for the ticket.** A 64-node (512-GPU) fleet running
this experiment weekly for a quarter:

```
  per run per node recovered (fix A)  = $103.4
  × 64 nodes                          = $6,618 per run
  × 13 weekly runs                    = $86,000 per quarter
```

State the rate and its date, state the SM% before and after, and show the two lines of
arithmetic. That is a defensible number — and it is exactly the MFU input 08.8 consumes.

## Practice

Feeds the **Survive-a-failure lab** deliverable with a *before/after starvation fix, with
GPU-hours and dollars quantified*. See
[`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md) for the
full lab spec; this lesson supplies its data-pipeline artifact.

1. **Establish the ceiling.** On one GPU, run the training loop against a synthetic dataset
   (`__getitem__` returns a preallocated random tensor). Record steps/s and
   `DCGM_FI_PROF_SM_ACTIVE` (or `nvidia-smi dmon -s u`). This is `R_gpu`.
2. **Instrument the real pipeline.** Same job on real data. Log SM-active, per-core CPU
   (`mpstat -P ALL 1`), and `t_data` vs `t_step` from `torch.profiler`. Compute
   `data_fraction` and `SM_active`. Make sure your dataset is at least 5× host RAM, or
   explicitly note that you are measuring a warm page cache.
3. **Induce starvation deliberately.** Set `num_workers=1` and add an expensive PIL
   transform. Capture the sawtooth SM% trace alongside the pegged CPU core — that image pair
   is the artifact.
4. **Fix it in two stages and measure each.** (a) Raise `num_workers` toward
   `R_gpu / measured_img_per_core`, set `prefetch_factor`, `pin_memory=True` +
   `non_blocking=True`, `persistent_workers=True`; size `/dev/shm` and record the value you
   computed. (b) If you plateau, repack a slice of the data as WebDataset shards (make the
   shard count a multiple of `world_size × num_workers`) and show the change in request
   count, or build the DALI pipeline from §8 and show where the SM time went.
5. **Compute the storage budget.** For your measured samples/s and sample size, compute the
   bytes/s needed for 1 GPU, for 8, and for 512; then state which tier from §4's table you
   would buy and what it costs.
6. **Quantify.** GPU-hours before and after for a fixed epoch count, × your `$/GPU-hr` with
   the date, minus the cost of the extra CPU/storage. One line.

**Acceptance:** a committed before/after artifact showing (a) the starvation signature — low,
sawtoothing SM% with pegged CPU, and the measured `data_fraction`; (b) the fix, with the
`num_workers` figure *derived* from a measured per-core rate rather than guessed; (c) a
storage-bandwidth budget for 1 / 8 / 512 GPUs with the tier that serves it; and (d) recovered
GPU-hours and dollars, net of the resources spent. That number is the MFU input the lab and
08.8 consume.

## Common pitfalls

- **"More `num_workers` always fixes it."** *Symptom:* raising workers stops moving SM%, or
  makes it worse. *Mechanism:* you have hit a different wall — physical cores, the container
  CPU quota (cgroup throttling looks identical to being short of cores), RAM, `/dev/shm`, or
  IOPS. When you are request-bound on millions of small files, more workers issue *more*
  concurrent small random reads and increase contention. *Fix:* identify which of the five
  walls you are against before turning the knob.
- **"`pin_memory=True` speeds up the copy."** On its own it does almost nothing. *Mechanism:*
  pinning only enables `cudaMemcpyAsync`; without `.to(device, non_blocking=True)` at the
  call site the copy is still synchronous and you have merely made allocation more expensive
  and the memory unswappable.
- **"The `Bus error` is a PyTorch bug."** *Symptom:*
  `DataLoader worker (pid N) is killed by signal: Bus error`. *Mechanism:* workers pass
  tensors through `/dev/shm`, which defaults to **64 MiB** in a container; writing past a
  tmpfs mapping raises SIGBUS. *Fix:* mount an `emptyDir{medium: Memory}` at `/dev/shm`
  sized to at least `prefetch_factor × num_workers × sizeof(batch)`.
- **"A faster disk fixes a slow pipeline."** Only if you are byte-bound. On 112 KB objects
  at 14,286 samples/s you need 14,286 requests/s; a faster device still pays a per-request
  cost on each of them. *Fix:* reduce the request count — shard.
- **"WebDataset shuffling is equivalent to a global shuffle."** It is a two-level
  approximation: shard-order shuffle plus an in-memory reservoir. If your shards were written
  in class order, samples within a reservoir window are correlated. *Fix:* shuffle at write
  time, and size the reservoir deliberately.
- **"Any shard count works."** `split_by_node`/`split_by_worker` are integer strides over
  the shard list. Fewer shards than `world_size × num_workers` leaves workers with nothing,
  and in DDP the ranks that run dry hang the collective. *Fix:* shards ≫ `world_size ×
  num_workers`, sized evenly.
- **"DALI is a free win."** It consumes GPU memory and SMs. Published results range from
  single-digit percent to ~70% end-to-end depending on configuration. Applied to a job that
  was comms-bound or compute-saturated it spends SMs for nothing. *Fix:* run the synthetic-data
  A/B first.
- **"Flat-low SM% and sawtooth SM% mean the same thing."** They do not. Sawtooth with a
  pegged CPU is starvation; flat-low with an idle CPU is comms- or occupancy-bound (08.3).
  Applying the wrong fix wastes engineering time and buys nothing.
- **"Benchmark the second epoch, it's warmed up."** By epoch two the page cache holds your
  dataset and you are measuring RAM. MLPerf Storage encodes the correction as a hard rule
  (dataset ≥ 5× host memory) and fails submissions that violate it.
- **"The data pipeline is the ML team's problem."** A starved GPU bills at the full rate
  regardless of whose code caused it, and the researcher's dashboard does not show it. Uber's
  ~80% measured cost reduction on an unchanged model is the evidence that this is a platform-owned
  line item.

## Self-check

- **GPU SM-active is low but a CPU core is pegged at 100%. Diagnosis, next check, and how do
  you rule out the alternatives?**
  **Answer:** Input-pipeline starvation — the GPU is blocked in `next(loader)` while a
  worker burns CPU on decode/transform. The SM% trace will be *sawtoothing*, not flat-low.
  Rule out the alternatives by the CPU signature: if all cores were idle with low SM%, it is
  comms-bound (a long collective, 08.3) or sync-bound (a straggler at a barrier). Next check:
  attribute the step with `torch.profiler` into `t_data` (blocked in `next`) versus
  `t_step`, then run the decisive test — swap in a synthetic dataset that returns a
  preallocated tensor. If step time collapses, the pipeline was the bottleneck and you now
  know `R_gpu` exactly; if it does not move, stop tuning the loader. Also check the container
  CPU limit: cgroup throttling is indistinguishable from being short of cores at the
  `mpstat` level.

- **Compute the storage read bandwidth needed to keep 64 H100s fed for ResNet-50-class
  training, and name a tier that delivers it.**
  **Answer:** Demand per accelerator = `batch_size / computation_time × record_length_bytes`.
  With MLPerf Storage's H100 ResNet-50 parameters (400 samples per 0.224 s, 114,660 B per
  record): `1,785.7 samples/s × 114,660 B = 204.7 MB/s` per GPU, so **64 GPUs need
  ≈13.1 GB/s** (≈11.8 GB/s if you size to the benchmark's 90% AU floor). Tiers: local NVMe
  at 3–7 GB/s per drive needs several drives striped per node — easy inside an 8-GPU node
  that needs only 1.64 GB/s; FSx for Lustre Persistent-2 at 250 MB/s/TiB needs ≈52 TiB
  provisioned (or ≈13 TiB at 1,000 MB/s/TiB) — you buy capacity to get bandwidth; S3 needs
  `⌈13,100/87⌉ ≈ 151` concurrent connections at ~85–90 MB/s each, which is fine **if** the
  objects are large. If instead you read one 112 KB object per sample you need
  `64 × 1,786 = 114,000 GET/s`, which at 5,500 GET/s per prefix requires ≥21 prefixes — this
  is the case where you shard rather than buy.

- **Why is loader starvation a *cost* bug rather than a performance bug, and what is the
  multiplier?**
  **Answer:** Because the GPU bills at the full hourly rate whether it is 95% or 34%
  SM-active. Landed progress scales with `SM_active = min(1, R_data / R_gpu)`, so the cost
  per unit of training progress is `1 / SM_active`. At 34% versus 95% that is a **2.8×**
  multiplier — you rent a $3–7/GPU-hr accelerator to watch it wait on cores that cost cents.
  The asymmetry is the argument: an H100-hour is roughly 100× a vCPU-hour, so
  over-provisioning the input pipeline is nearly always correct. Uber's production case —
  10–15% → >60% utilisation, 22 h → ~3 h, ~80% compute cost reduction on an unchanged model —
  is the real-world instance.

- **Name the two structural fixes, the failure each addresses, and what each costs.**
  **Answer:** (1) **Sharded sequential streaming** (WebDataset, MosaicML Streaming) addresses
  the *request-rate* wall: packing many samples into ~1 GB tar shards read sequentially turns
  14,286 requests/s into ~1.6, converting an IOPS problem into a bandwidth problem that both
  object stores and disks handle well. Costs: a one-off repack job and duplicate storage;
  only window shuffling (shard-order shuffle plus an in-memory reservoir), not a global
  permutation; and a hard constraint that shard count ≫ `world_size × num_workers`, because
  `split_by_node`/`split_by_worker` are integer strides and a worker with no shards hangs
  the DDP collective. (2) **GPU-side decode** (NVIDIA DALI, `device="mixed"`) addresses the
  *core* wall: nvJPEG plus the dedicated hardware JPEG engine on Ampere+ take decode off the
  CPU and eliminate the host-to-device copy of decoded pixels. Costs: GPU memory (the
  `device_memory_padding`/`host_memory_padding` preallocations), SM time for the software
  decode fraction and augmentation, a second pipeline API, and a benefit range published
  anywhere from single-digit percent to ~70% depending on configuration — so verify the CPU
  is genuinely the bottleneck first.

- **How many batches are in flight in a PyTorch `DataLoader`, and what does that number
  control?**
  **Answer:** `prefetch_factor × num_workers` — enforced in `_try_put_index`, which refuses
  to dispatch when `_tasks_outstanding` reaches that product. Defaults are `num_workers=0`
  (no worker processes at all — the main process loads synchronously) and, when
  `num_workers > 0`, `prefetch_factor=2`. That product controls three things: how much
  *jitter* the pipeline can absorb (a slow sample is hidden if the buffer is deep), how much
  memory is resident in `/dev/shm` (`product × sizeof(batch)`, which is what SIGBUSes against
  the 64 MiB container default), and how long the first batch takes to appear. It does **not**
  help with a sustained rate deficit: a buffer in front of an under-provisioned producer
  drains at the deficit rate and then stays empty — which is why SM% that starts high and
  decays over the first minute of an epoch is the signature of a throughput shortfall rather
  than a latency spike.

## Connections & what's next

08.7 closes the "is the GPU actually doing useful work" thread that 08.3 (comms), 08.5
(failures) and this lesson (starvation) cover between them. With all three diagnoses in
hand, 08.8 folds them into a single formula: the GPU-hours a run actually consumes are
`work / (peak × MFU)` times a failure-overhead multiplier — and starvation is a direct,
usually unlabelled, tax on that MFU term. The `SM_active` number and the recovered-GPU-hours
figure from this lesson's Practice are literal inputs to it.

Two forward links worth holding: the storage-bandwidth budget from §3–§4 reappears in 08.8
as a non-GPU cost that the loading multiplier `L` has to recover, and the container CPU-limit
pitfall is the same class of error as module 06's quota mistakes — a resource request that
silently caps throughput.

## References & further reading

**Primary sources — read in this pass**

1. **PyTorch — `torch/utils/data/dataloader.py`** (`main`, version string `2.15.0a0`) —
   <https://github.com/pytorch/pytorch>. **Read the `DataLoader` docstring and
   `_MultiProcessingDataLoaderIter`.** Source of every default in §5's table
   (`num_workers=0`, `prefetch_factor=None`→2, `pin_memory=False`,
   `persistent_workers=False`, `in_order=True`, `timeout=0`, `batch_size=1`,
   `drop_last=False`, `pin_memory_device` deprecated), the in-flight cap
   `max_tasks = prefetch_factor × num_workers` in `_try_put_index`, the round-robin
   `_worker_queue_idx_cycle` dispatch, and the `in_order=False` fair-share branch. Fetched
   directly from the repository; **docs.pytorch.org is blocked by this environment's egress
   proxy**, so the rendered documentation page was not read.
2. **MLCommons — `mlcommons/storage`** — <https://github.com/mlcommons/storage>. **Read
   `configs/dlio/workload/{resnet50,unet3d,cosmoflow}_{a100,h100}.yaml` and `Rules.md`.**
   Source of §3's entire demand table — `batch_size`, `computation_time`,
   `record_length_bytes` and the `au` target for each workload — and of the AU definition
   (`AU = total_compute_time / total_benchmark_running_time`) plus the
   `HOST_MEMORY_MULTIPLIER = 5` anti-page-cache rule enforced by `trainingRecalculateDatasetSize`.
   Verified by cloning at HEAD. The per-accelerator requirements derived in §3
   (~190 MB/s, ~3 GB/s, ~0.5 GB/s per H100) reproduce the figures quoted in public MLPerf
   Storage v2.0 coverage to within rounding.
3. **NVIDIA DALI — source tree, version `2.4.0dev`** — <https://github.com/NVIDIA/DALI>.
   **Read `dali/python/nvidia/dali/pipeline.py`, `dali/operators/imgcodec/decoder_schema.cc`,
   `dali/operators/reader/loader/loader.cc`.** Source of every DALI default in §8:
   `prefetch_queue_depth=2`, `py_num_workers=1`, `py_start_method="fork"`,
   `hw_decoder_load=0.65`, `hybrid_huffman_threshold=1,000,000` px,
   `device_memory_padding=16 MiB`, `host_memory_padding=8 MiB` (double-buffered), and the
   reader defaults `random_shuffle=False`, `initial_fill=1024`, `prefetch_queue_depth=1`,
   `read_ahead=False`, `dont_use_mmap=False`, `stick_to_shard=False`, `pad_last_batch=False`.
   Verified by cloning at HEAD; **docs.nvidia.com is blocked from this environment**.
4. **WebDataset — `webdataset/webdataset`** — <https://github.com/webdataset/webdataset>.
   **Read `src/webdataset/shardlists.py` (`split_by_node`, `split_by_worker`) and the README
   pipeline section.** Source of the `islice(src, rank, None, world_size)` striding that
   produces §7's shard-count constraint, and of the two-level shuffle
   (`shardshuffle` + `.shuffle(bufsize)`). Verified by cloning at HEAD.
5. **Ray Train / Ray Data configuration** — <https://github.com/ray-project/ray>. Used in
   08.6 for the orchestration side; noted here because Ray Data is the third production
   answer to the streaming problem. *docs.ray.io is blocked from this environment; not
   relied on for any claim in this lesson.*

**Published measurements and product characteristics**

6. **"Need for Speed: A Comprehensive Benchmark of JPEG Decoders in Python"** —
   <https://arxiv.org/abs/2501.13131>. Single-thread ImageNet JPEG decode throughput
   spanning roughly **270 img/s** (Arm Neoverse N1) to **840 img/s** (Zen 5), with
   libjpeg-turbo-family decoders and OpenCV near the top on every architecture — the basis
   for §4's per-core budget. *arxiv.org is blocked from this environment; the figures were
   confirmed via search against the paper's abstract and coverage, not by reading the PDF.*
7. **Amazon S3 performance guidelines** —
   <https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance-guidelines.html>.
   **5,500 GET/HEAD requests per second per prefix**, 3,500 PUT/COPY/POST/DELETE per prefix,
   and "one concurrent request per **85–90 MB/s** of desired throughput", with gradual
   scale-up and HTTP 503 Slow Down during adaptation. *docs.aws.amazon.com is blocked here;
   these are AWS's own published guideline figures, search-verified.*
8. **Amazon FSx for Lustre storage classes** —
   <https://docs.aws.amazon.com/fsx/latest/LustreGuide/using-fsx-lustre.html>. Per-TiB
   provisioned throughput: **Persistent-2 SSD 125 / 250 / 500 / 1000 MB/s per TiB**,
   **Persistent-1 SSD 50 / 100 / 200 MB/s per TiB**. The basis for §4's "bandwidth is bought
   as capacity" point. *Blocked here; search-verified against AWS's published documentation.*
9. **NVIDIA — "Leveraging the Hardware JPEG Decoder and nvJPEG on A100"** and
   **"Loading Data Fast with DALI and the New Hardware JPEG Decoder in A100"** —
   <https://developer.nvidia.com/blog/leveraging-hardware-jpeg-decoder-and-nvjpeg-on-a100/>.
   A100's 5-core hardware JPEG engine, up to ~20× faster decode than CPU-only, ~7,000 img/s
   with `hw_decoder_load` near 0.75. *Blocked from this environment; vendor-published,
   search-verified, and treated as directional. The DALI *defaults* in §8 come from source
   (entry 3), not from these posts.*
10. **NVIDIA — "Case Study: ResNet50 with DALI"** —
    <https://developer.nvidia.com/blog/case-study-resnet50-dali/>, and **AWS —
    "Accelerate computer vision training using GPU preprocessing with NVIDIA DALI on Amazon
    SageMaker"**, reporting **37–72%** improvement across ResNet-18/50/152. Together they
    establish that the DALI win is a wide range, not a constant. *Both hosts blocked; cited
    as reported, not fetched.*

**Real-world engineering**

11. **Uber — "Accelerating Deep Learning: How Uber Optimized Petastorm for High-Throughput
    and Reproducible GPU Training"** — <https://www.uber.com/us/en/blog/accelerating-deep-learning/>.
    10–15% → >60% GPU utilisation, 22 h → ~3 h, ~80% compute-cost reduction. *uber.com is
    blocked here; search-verified.*
12. **Databricks — "MosaicML StreamingDataset"** — <https://www.databricks.com/blog/mosaicml-streamingdataset>.
    The second production shard-and-stream system. *Blocked; not fetched in this pass.*
13. **LAION-5B** — <https://laion.ai/blog/laion-5b/>. A 5.85-billion-pair, ~240 TB dataset
    distributed as WebDataset tar shards — the existence proof that shard streaming scales to
    the largest public vision-language corpora. *Not fetched in this pass; **an earlier
    version of this lesson attributed a specific ">10,000 samples/sec per GPU node"
    throughput figure to the OpenCLIP training pipeline, which could not be substantiated
    against any primary source and has been removed.***

**Deeper dives**

14. **Chaim Rand — "A Caching Strategy for Identifying Bottlenecks on the Data Input
    Pipeline"** — <https://chaimrand.medium.com/a-caching-strategy-for-identifying-bottlenecks-on-the-data-input-pipeline-8e52060b402f>.
    A practitioner's profiler-based, stage-by-stage attribution method, complementary to §6's
    synthetic-data A/B. *Not fetched in this pass.*
15. **[08.3 · Communication as the bottleneck](03-communication-bottleneck.md)** and
    **[08.8 · Training economics](08-training-economics.md)** — the two lessons this one
    sits between: 08.3 owns the roofline and the comms-bound diagnosis this lesson rules out,
    08.8 owns the cost model that consumes this lesson's `SM_active` figure.

> **Snapshot (2026-08).** GPU rates move. On-demand H100 SXM spans roughly **$1.4–$7/GPU-hr**
> on specialist clouds and up to **~$7–12/GPU-hr** at hyperscaler list price; module 11 uses a
> blended **$2.99/GPU-hr** for the same hardware. The `$3.00/GPU-hr` used in the Worked example
> is a stand-in — the arithmetic is linear in the rate, so substitute your own blended number
> and re-run it. Storage product characteristics (FSx per-TiB tiers, S3 request limits) are
> also dated; re-check before quoting in a design review.
