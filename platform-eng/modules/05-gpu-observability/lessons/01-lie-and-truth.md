---
lesson: "05.1"
title: "The lie and the truth: GPU_UTIL vs the PROF metrics"
module: "05"
concept: "GPU_UTIL vs the PROF metrics"
status: not-started
est_time: "7h"
prev: null
next: "02-dcgm-architecture.md"
artifacts: []
sources: 15
---

# 05.1 · The lie and the truth: GPU_UTIL vs the PROF metrics

> **Concept.** `DCGM_FI_DEV_GPU_UTIL` measures whether a kernel was *resident*, not whether the silicon was *working* — the four PROF metrics measure the work, and each drives a different money decision.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)
>
> **Advanced Learning** — [The GPU Utilisation Lie](../../../learning/01-lie-and-truth.html): the two measurement paths drawn apart, the lie derived from H100 datasheet numbers, and the SM_ACTIVE × TENSOR quadrants. Optional visual companion; this lesson stays the source of truth.

## Where this fits

Module 04 closed with the GPU Operator running your device plugin, dcgm-exporter deployed
on every node, and the pod-resources join wired up so a Prometheus series can be traced
back to a namespace and a pod. You can *get metrics out of a GPU cluster*. What module 04
never asked is whether the flagship metric — the one every default dashboard leads with —
means what everyone assumes it means. This lesson answers that, mechanically, in exact
field-ID terms, and the answer is the spine the rest of module 05 hangs off: the
architecture in 05.2, the counter-set audit in 05.3, the attribution joins in 05.4, and
the capstone's entire dollar argument in 05.8 all assume you know precisely what
`GPU_UTIL` can and cannot tell you, and *why the hardware makes it that way*.

Everything below is checked against **DCGM 4.6.0** (the `NVIDIA/DCGM` release tagged in
July 2026), **dcgm-exporter 4.8.3** (image `4.6.0-4.8.3-distroless`), and **NVML API
version 13** as shipped in `NVIDIA/go-nvml`'s vendored `nvml.h`. Field IDs, struct
comments, short names and code paths quoted below came out of those trees — `dcgm_fields.h`,
`dcgmlib/src/dcgm_fields.cpp`, `dcgmlib/src/DcgmCacheManager.cpp`,
`dcgmlib/src/DcgmGpmManager.cpp`, and `nvml.h` — not out of a blog post. Where a number
depends on your silicon or driver, that is said explicitly.

## Why this matters

"Our fleet dashboard shows 95% GPU utilization, but the training team says throughput is
bad — reconcile that" is a live interview question at NVIDIA (*Senior AI & HPC
Observability Engineer*, *Senior Platform Telemetry Engineer*), CoreWeave (*Sr SWE,
Observability* — a neocloud whose product *is* GPU fleet transparency), and Datadog, whose
2025 DCGM product splits its metric taxonomy along exactly this seam: device-level fields,
process-level attribution, and pipeline-level activity ratios. A paid vendor's entire
information architecture is this lesson's field list.

The concrete cost of not knowing it: **the number everyone trusts is structurally highest
exactly when the fleet is most wasted.** A batch-1 decode server pins `GPU_UTIL` at 100
while using something on the order of one percent of the tensor throughput you are paying
for. A capacity-planning spreadsheet built on field 203 will read "we are out of GPUs" and
authorise an expansion that buys nothing. At the ~$2.50/GPU-hour H100 on-demand rate the
Silicon Data H100 rental index reported in mid-2026 (a dated, directional snapshot —
provider quotes in the same window ranged from roughly $2.19 to over $11 per GPU-hour),
an unnecessary 64-GPU expansion is on the order of **$1.4M/year** of hardware bought on the
strength of a metric that cannot see the problem it was cited to prove.

Getting it right is worth the same money in the other direction, and it is worth it
*repeatedly*, because the same reasoning drives reclaim policy, right-sizing, chargeback,
and the decision of which profiling tool to reach for next.

## What's new here (calibration)

- You already have the *concept* from module 03 — GPU-util vs SM-occupancy vs
  tensor-active vs MFU, and the roofline model explaining why decode is memory-bound.
  **Not re-taught as a concept.** What is new is deriving it from the actual counters.
- You already deployed dcgm-exporter and built the pod-resources join in module 04, and
  you run Prometheus/PromQL/Grafana/alerting fluently across 40+ clusters. **Not
  re-taught.**
- New here, and the point of the lesson:
  - **Two independent collection paths.** `GPU_UTIL` and `SM_ACTIVE` are not two views of
    one measurement pipeline. One is an NVML driver-maintained duty-cycle counter read
    through `nvmlDeviceGetUtilizationRates()`; the other is a hardware performance-monitor
    sample differenced across an interval. They can disagree by two orders of magnitude
    without either being broken.
  - **The exact predicate** NVML evaluates, and the arithmetic that turns a true predicate
    into a false conclusion.
  - **The full field table** with real IDs, real semantics from the header, which path
    each field arrives on, and whether it is safe to alert on.
  - **A renaming you will hit in DCGM 4.x**: `DCGM_FI_PROF_SM_ACTIVE` is now a
    backward-compatibility alias, not the primary symbol.

## Core concepts

### 1. The problem: one word, two questions

A platform team asks a GPU dashboard two completely different questions and uses the same
word for both:

1. **"Is this GPU doing anything at all?"** — an *occupancy/reclaim* question. If the
   answer is no for thirty minutes on an allocated card, someone is paying rent on an
   empty room.
2. **"Is this GPU doing as much as it can?"** — an *efficiency/right-sizing* question. If
   the answer is no on a running job, the fix is batching, precision, kernel choice, or a
   smaller GPU, not more GPUs.

Both get called "utilisation". They have different answers, different owners, and
different remediations. The failure mode this whole module exists to correct is that the
default metric answers **neither** — it answers a third question nobody asked: *"was
anything scheduled on this device during the sampling window?"*

That third question is cheap to answer, which is why the driver answers it, which is why
it is the one number every tool surfaces. `nvidia-smi`'s `GPU-Util` column, the "GPU
Utilization" panel in the official NVIDIA DCGM Grafana dashboard, your cloud provider's
GPU graph, and `DCGM_FI_DEV_GPU_UTIL` are all the same underlying counter.

### 2. Two collection paths, drawn side by side

Before any field semantics, hold this picture. **The two families of metric come from
different silicon, through different driver interfaces, at different cost, with different
failure modes.** Almost every confusion in GPU telemetry dissolves once you can draw this.

```
                        ONE GPU, TWO INDEPENDENT MEASUREMENT PATHS
 ═══════════════════════════════════════════════════════════════════════════════════════

  PATH A — NVML DUTY CYCLE                    PATH B — HARDWARE PERFORMANCE MONITORS
  (cheap, always available, coarse)           (costed, Volta+, fine-grained)
  ─────────────────────────────────           ──────────────────────────────────────

   ┌──────────────────────────────┐            ┌────────────────────────────────────┐
   │ GPU scheduler / channel state│            │ per-SM + per-partition HW counters  │
   │                              │            │  · elapsed cycles                   │
   │  "is ≥1 kernel resident on   │            │  · cycles with ≥1 warp assigned     │
   │   the device right now?"     │            │  · resident-warp accumulator        │
   │        yes / no  (1 bit)     │            │  · tensor-pipe issue counters       │
   └──────────────┬───────────────┘            │  · DRAM interface active cycles     │
                  │                            └───────────────┬────────────────────┘
       driver integrates the bit                               │
       over a sample period of                    Hopper+ : nvmlGpmSampleGet()
       1 s … 1/6 s  (product-dependent)           snapshots ALL counters at once
                  │                               Volta–Ampere : DCP profiling module
                  ▼                               programs counter banks per group
   ┌──────────────────────────────┐                            │
   │ nvmlUtilization_t            │                            ▼
   │   .gpu     = 0..100 (%)      │            ┌────────────────────────────────────┐
   │   .memory  = 0..100 (%)      │            │ DCGM keeps 2+ samples per entity   │
   └──────────────┬───────────────┘            │ metric = f(sample[t-Δ], sample[t]) │
                  │                            │ Δ = the WATCH updateFreq           │
   nvmlDeviceGetUtilizationRates()             │ value = percent / 100 → ratio 0..1 │
                  │                            └───────────────┬────────────────────┘
                  ▼                                            ▼
   ┌──────────────────────────────┐            ┌────────────────────────────────────┐
   │ DCGM_FI_DEV_GPU_UTIL     203 │            │ DCGM_FI_PROF_* 1001 … 1085         │
   │ DCGM_FI_DEV_MEM_COPY_UTIL 204│            │   1002 SM_ACTIVE                   │
   │   int64, units of PERCENT    │            │   1003 SM_OCCUPANCY                │
   │   no root required           │            │   1004 PIPE_TENSOR_ACTIVE          │
   │   works on any Fermi+ device │            │   1005 DRAM_ACTIVE                 │
   │   NOT available when MIG is  │            │   double, units of RATIO 0.0–1.0   │
   │   enabled on the device      │            │   needs CAP_SYS_ADMIN / root       │
   └──────────────┬───────────────┘            └───────────────┬────────────────────┘
                  │                                            │
                  └────────────┬───────────────────────────────┘
                               ▼
                    dcgm-exporter :9400 /metrics
                    (same Prometheus series namespace,
                     same labels — which is exactly why
                     people assume the same provenance)
```

The last box is the trap. Both families arrive as `DCGM_FI_*` gauges on the same
`/metrics` endpoint carrying the same `gpu`/`UUID`/`modelName` labels, so they *look* like
peers. They are not. Path A has no notion of how many SMs exist. Path B has no notion of
whether a context was bound. Neither can be derived from the other.

### 3. What field 203 actually is, line by line

`DCGM_FI_DEV_GPU_UTIL` is field ID **203**. In DCGM 4.x the primary symbol is
`DCGM_FI_DEV_GPU_UTIL_RATIO` and `DCGM_FI_DEV_GPU_UTIL` is a backward-compatibility
`#define` retained in `dcgm_fields.h` under the comment *"Deprecated: renamed per DCGM
Field Naming Standard"*. The exporter's CSV, and therefore your Prometheus series name,
still uses the old spelling. (This renaming caught the whole `DCGM_FI_PROF_*` family too —
see §7.)

Its collection is a straight passthrough. In `DcgmCacheManager.cpp` the handler for
`DCGM_FI_DEV_GPU_UTIL_RATIO` and `DCGM_FI_DEV_MEM_COPY_UTIL` is *one shared case*: it calls
`nvmlDeviceGetUtilizationRates()` once, takes `.gpu` for 203 and `.memory` for 204, casts
the `unsigned int` to `long long`, logs an error if the value exceeds 100, and appends it
to the cache. There is no averaging, no scaling, no derivation. **Field 203 is
`nvmlUtilization_t.gpu`, unmodified.**

So the semantics are NVML's semantics, and NVML states them precisely in `nvml.h`:

```c
/**
 * Utilization information for a device.
 * Each sample period may be between 1 second and 1/6 second,
 * depending on the product being queried.
 */
typedef struct nvmlUtilization_st
{
    unsigned int gpu;     //!< Percent of time over the past sample period during which
                          //!< one or more kernels was executing on the GPU
    unsigned int memory;  //!< Percent of time over the past sample period during which
                          //!< global (device) memory was being read or written
} nvmlUtilization_t;
```

Say it in one sentence with no ambiguous word in it:

> **`DCGM_FI_DEV_GPU_UTIL` is the percentage of a short driver-internal sample window
> during which at least one kernel was resident on the GPU.**

Every clause is load-bearing, and each one is where an assumption dies:

- **"at least one kernel"** — this is a *threshold at one*, not an intensity. One kernel
  and ten thousand concurrent kernels both evaluate the predicate to true. The counter has
  no arity.
- **"resident" / "executing"** — the kernel is scheduled on the device. A kernel whose
  every warp is stalled waiting on an HBM load is still executing by this definition. It
  will still be executing four microseconds later when it is still waiting.
- **"a short driver-internal sample window"** — between **1 second and 1/6 second,
  product-dependent**, per the struct comment above. You do not choose it, you cannot read
  it back for `.gpu` (unlike `nvmlDeviceGetEncoderUtilization()`, which returns its
  sampling period as an out-parameter), and DCGM re-samples that already-integrated number
  at whatever `updateFreq` your watch specifies. Two averaging layers, neither aligned to
  your Prometheus scrape.
- **"percentage"** — units of percent, `0..100`, integer. Not a ratio. This alone catches
  people: `DCGM_FI_DEV_GPU_UTIL` is `95`, `DCGM_FI_PROF_SM_ACTIVE` is `0.95`, and a PromQL
  expression that compares them directly is off by 100×.

Two more facts from `nvml.h` that matter operationally and are almost never mentioned:

- **`nvmlDeviceGetUtilizationRates()` is not supported on MIG-enabled GPUs.** The header
  note is explicit: *"On MIG-enabled GPUs, querying device utilization rates is not
  currently supported."* So on your A100/H100 MIG nodes, `DCGM_FI_DEV_GPU_UTIL` is either
  absent or blank — and dcgm-exporter *drops blank values entirely* rather than exporting
  a zero (its `toString()` returns the sentinel `"SKIPPING DCGM VALUE"` for any
  `DCGM_*_IS_BLANK` value, and the collector skips the sample). A dashboard panel that
  goes empty when a node is switched to MIG is this, not a broken exporter.
- **ECC scrubbing inflates it at driver init.** *"During driver initialization when ECC is
  enabled one can see high GPU and Memory Utilization readings. This is caused by ECC
  Memory Scrubbing."* A node that reports 100% util for the first seconds after a driver
  load is not running anything.

And `DCGM_FI_DEV_MEM_COPY_UTIL` (**204**) is its sibling from the same call. Note what it
is *not*: it is **not** a copy-engine metric and it is **not** achieved bandwidth. It is
the percentage of the sample window during which global device memory was being read or
written — a presence duty cycle for the memory subsystem, exactly as `.gpu` is a presence
duty cycle for the compute subsystem. A workload that issues one lazy read per microsecond
saturates `MEM_COPY_UTIL` at 100 while using a fraction of a percent of HBM bandwidth. Its
honest counterpart is `DCGM_FI_PROF_DRAM_ACTIVE` (1005), on Path B.

### 4. Deriving the lie: why batch-1 decode pins 203 at 100

The general shape of the lie is: **any workload that keeps a continuous stream of small
kernels resident satisfies the predicate ~100% of the time regardless of how much silicon
each kernel uses.** Autoregressive LLM decode at batch size 1 is the purest instance, and
you can derive the numbers rather than quote them.

Take **Llama-3-8B in BF16 on one H100 SXM5**. The three hardware numbers you need, from
NVIDIA's H100 datasheet and the Hopper tuning guide:

| Quantity | Value | Source |
|---|---|---|
| SMs enabled (H100 SXM5) | **132** | H100 datasheet / GH100 die config |
| Max resident warps per SM (compute capability 9.0) | **64** (2048 threads ÷ 32) | Hopper Tuning Guide |
| HBM3 bandwidth | **≈ 3.35 TB/s** | H100 SXM5 datasheet |
| Dense BF16 tensor throughput | **≈ 989 TFLOP/s** (1,979 with 2:4 sparsity) | H100 SXM5 datasheet |

Now the per-token arithmetic for one decode step, batch 1:

```
  MEMORY SIDE ────────────────────────────────────────────────────────────────
  Every weight must be read from HBM once per token (batch 1 → no reuse):
      bytes_moved  = 8e9 params × 2 bytes/param              = 16 GB
      t_memory     = 16 GB ÷ 3.35 TB/s                       ≈ 4.78 ms

  COMPUTE SIDE ───────────────────────────────────────────────────────────────
  A forward pass is ~2 FLOP per parameter per token:
      flops        = 2 × 8e9                                 = 16 GFLOP
      t_compute    = 16 GFLOP ÷ 989 TFLOP/s                  ≈ 16.2 µs

  RATIO ──────────────────────────────────────────────────────────────────────
      t_compute / t_memory = 16.2 µs / 4780 µs               ≈ 0.0034   ( 0.34 % )

  ROOFLINE POSITION ──────────────────────────────────────────────────────────
      arithmetic intensity = 16 GFLOP / 16 GB                ≈ 1 FLOP/byte
      H100 ridge point     = 989e12 / 3.35e12                ≈ 295 FLOP/byte
      → the workload sits ~295× to the LEFT of the ridge: hard memory-bound
```

Two conclusions fall straight out, and they are the entire lesson:

**(a) `PIPE_TENSOR_ACTIVE` must be ≈ 0.003.** The tensor pipes are issuing for 16 µs out of
every 4.78 ms. There is no tuning that changes this at batch 1; it is arithmetic. Measured
values land a little higher (0.005–0.03) because real kernels are not perfectly
bandwidth-efficient and the attention path adds work, but the order of magnitude is
derived, not guessed.

**(b) `DCGM_FI_DEV_GPU_UTIL` must be ≈ 100.** During those 4.78 ms there is *always exactly
one kernel resident*: a GEMV against a weight matrix, then the next GEMV, then attention
over the KV cache, then the next layer. The runtime enqueues them back to back on one
stream. The gap between kernels is a few microseconds of launch latency, far below the
1 s … 1/6 s resolution of NVML's sample window. The predicate "≥1 kernel resident" is true
for essentially the whole window. **The field is not malfunctioning. It is correctly
reporting a true fact that is useless.**

Here is the same thing on a timeline — the exhibit this module is named for. Time runs
left to right across one 1-second NVML sample window; each `▓` block is one small kernel.

```
      ONE NVML SAMPLE WINDOW (1 s) — H100, Llama-3-8B, BF16, batch 1
 ───────────────────────────────────────────────────────────────────────────────

 kernels          ▓▓▓ ▓▓ ▓▓▓▓ ▓▓ ▓▓▓ ▓▓▓▓ ▓▓ ▓▓▓ ▓▓▓▓ ▓▓ ▓▓▓ ▓▓▓▓ ▓▓ ▓▓▓ ▓▓▓▓ ▓▓
 resident?        ████████████████████████████████████████████████████████████▉
                  └──── "≥1 kernel resident" true for ~99.8% of the window ────┘
 DCGM_FI_DEV_GPU_UTIL  ────────────────────  100  ───────────────────────────────


 what the 132 SMs are doing inside the same window
 ───────────────────────────────────────────────────────────────────────────────
 SM   0..31      ▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░
 SM  32..63      ░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░░░▓░░░
 SM  64..95      ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
 SM  96..131     ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                 ▓ = ≥1 warp assigned      ░ = SM idle
 DCGM_FI_PROF_SM_ACTIVE  ─────────────────  0.16  ──────────────────────────────


 what the warp slots inside a lit SM look like  (64 slots/SM on cc 9.0)
 ───────────────────────────────────────────────────────────────────────────────
 slots filled    ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                 └ 6 of 64 ┘
 DCGM_FI_PROF_SM_OCCUPANCY  ──────────────  0.09  ──────────────────────────────


 what the tensor pipes are doing
 ───────────────────────────────────────────────────────────────────────────────
 tensor issue    ▏      ▏      ▏      ▏      ▏      ▏      ▏      ▏      ▏
 DCGM_FI_PROF_PIPE_TENSOR_ACTIVE  ────────  0.01  ──────────────────────────────


 what HBM is doing
 ───────────────────────────────────────────────────────────────────────────────
 DRAM active     ███████████████████░████████████████░██████████████████░█████
 DCGM_FI_PROF_DRAM_ACTIVE  ───────────────  0.71  ──────────────────────────────

 ═══════════════════════════════════════════════════════════════════════════════
  SAME GPU. SAME SECOND.   GPU_UTIL = 100        SM_ACTIVE = 0.16
                           OCCUPANCY = 0.09      TENSOR    = 0.01
  The bottleneck is HBM, and the only field that says so is on Path B.
 ═══════════════════════════════════════════════════════════════════════════════
```

(The SM/warp/pipe rows are illustrative of the shape; the `GPU_UTIL`/`TENSOR_ACTIVE`
figures are derived above and the `SM_ACTIVE`/`OCCUPANCY` figures are representative of
what a batch-1 7–8B decode measures on an H100. Reproduce them yourself in the Practice
section — that screenshot is the deliverable.)

**Generalise the failure.** It is not about LLMs. `GPU_UTIL` reads ~100 for any of:

| Workload | Why the predicate stays true | What is actually wasted |
|---|---|---|
| Batch-1 autoregressive decode | continuous stream of tiny GEMV kernels | tensor pipes (≈99% idle) |
| A `while(true)` polling kernel / spin-wait | one kernel never exits | everything |
| A dataloader-starved training loop | tiny kernels between long host stalls, each ≥1 resident | SMs, tensor pipes |
| Single-threaded CUDA benchmark on 1 SM | one block on one of 132 SMs | 131 SMs |
| Collective-communication stall (NCCL wait) | the wait is implemented as a resident kernel | all compute |

That last row is worth pausing on, because it is the one that bites at scale: **an NCCL
all-reduce that is blocked waiting on a straggler rank is a resident kernel.** A 512-GPU
training job with one slow node shows a fleet-wide `GPU_UTIL` of 100 while 511 GPUs do
nothing but spin. `SM_ACTIVE` on the waiting ranks stays high too (the wait kernel does
occupy SMs), but `PIPE_TENSOR_ACTIVE` collapses — which is why the *pair* matters and no
single field is sufficient.

### 5. Path B, mechanically: how DCGM produces an activity ratio

Now the honest side. The `DCGM_FI_PROF_*` fields do not poll a driver-maintained
percentage; they **difference two hardware-counter snapshots**. On Hopper and newer this is
NVML's GPM (GPU Performance Monitoring) interface, and DCGM's implementation
(`DcgmGpmManager.cpp`) is short enough to describe exactly:

1. **On watch.** When any PROF field is watched for an entity, DCGM creates a
   `DcgmGpmManagerEntity` holding a `std::map<timestamp, DcgmGpmSample>` plus a watch
   table. The watch carries an `updateIntervalUsec`, a `maxAgeUsec` and a `maxKeepSamples`.
2. **On read.** `GetLatestSample()` first prunes samples older than a *derived* horizon,
   then calls `MaybeFetchNewSample()`. If the newest sample in the map is younger than the
   watch's update interval, it is reused; otherwise `nvmlGpmSampleGet()` is called and the
   result inserted at `now`.
3. **The key property.** `nvmlGpmSampleGet()` takes **one opaque snapshot containing every
   GPM metric at once** — SM util, occupancy, all the tensor sub-pipes, DRAM bandwidth,
   NVDEC/NVJPG/NVOFA, PCIe and per-link NVLink. Not one counter. All of them.
4. **Computing the value.** DCGM picks a *baseline* sample at
   `latest_timestamp − updateInterval`, then calls `nvmlGpmMetricsGet()` with
   `sample1 = baseline`, `sample2 = latest`, and the single metric ID it wants. NVML
   differences the two snapshots. If no old-enough baseline exists, DCGM returns without
   setting a value, leaving it blank.
5. **Scaling.** GPM returns percentages `0.0–100.0`. DCGM divides by 100 for the fields
   flagged as percentage fields, producing the `0.0–1.0` ratio you see in Prometheus.
   `NaN` becomes `DCGM_FP64_BLANK`.
6. **Retention.** The prune horizon is not the user's `maxKeepAge`; it is derived:
   `2 × maxUpdateInterval + min(maxUpdateInterval, 1 s)`. The source comment says why —
   *"`nvmlGpmMetricsGet` needs two samples spanning an update interval. One interval of
   retention is too tight: samples can be lost due to jitter."*

Three consequences you should be able to state cold:

- **The averaging window of a PROF field is the watch's `updateFreq`, not a fixed
  constant.** If dcgm-exporter watches at its default 30 s collect interval, every
  `SM_ACTIVE` value you scrape is the mean over the preceding ~30 s. Set the watch to 1 s
  and it is the mean over the preceding second. This is a knob, and lesson 05.2 derives
  where to set it.
- **The first read after a watch starts is blank**, because there is no baseline yet. A
  freshly restarted exporter shows device fields immediately and PROF fields one interval
  later. That gap is expected, not a bug.
- **Blank is not zero.** DCGM's blank sentinels are enormous positive values
  (`DCGM_FP64_BLANK = 140737488355328.0`, `DCGM_INT64_BLANK = 0x7ffffffffffffff0`), with
  `+1`, `+2`, `+3` offsets meaning `NOT_FOUND`, `NOT_SUPPORTED` and `NOT_PERMISSIONED`.
  dcgm-exporter checks `IS_BLANK` and **omits the series**. So a PROF field you lack
  permission for produces *absence*, and absence in PromQL is not `0` — it makes
  `avg by (namespace)` silently skip the GPU rather than drag the average down. This is the
  single most common way an "everything looks fine" GPU dashboard is lying by omission.

On **Volta through Ampere** the same field IDs are served by the separate **DCP profiling
module**, which programs counter banks directly and is shipped as a proprietary component
(dcgm-exporter's README: *"for Ampere and earlier generation GPUs, profiling metrics depend
on the datacenter-gpu-manager-4-proprietary package"*). That path has the multiplexing
constraint 05.2 takes apart. The *field IDs and semantics are identical*; the acquisition
mechanism is not.

### 6. What each honest field actually counts

Now the semantics, taken from the field definitions in `dcgm_fields.h` and their GPM
metric mappings in `dcgm_fields.cpp`. For each: the definition, the mechanism, the
arithmetic, and the failure mode.

#### `DCGM_FI_PROF_SM_ACTIVE` — 1002 — *breadth*

Header definition: *"The ratio of cycles an SM has at least 1 warp assigned (computed from
the number of cycles and elapsed cycles)."* GPM metric: `NVML_GPM_METRIC_SM_UTIL`,
documented as *"Percentage of SMs that were busy."* `dcgmi` short name: `SMACT`.

Mechanically, per SM the hardware maintains two counters: elapsed cycles, and cycles during
which at least one warp was assigned. The ratio is computed per SM and **averaged across
all SMs on the device**. Which means the arithmetic is:

```
                        Σ over SMs ( active_cycles[sm] )
  SM_ACTIVE   =        ─────────────────────────────────
                        Σ over SMs ( elapsed_cycles[sm] )

  On an H100 SXM5 with 132 SMs, a kernel occupying exactly 32 SMs
  continuously for the whole interval yields:

      SM_ACTIVE = 32/132 = 0.24     ← even though those 32 SMs are 100% busy
```

**This is the most important arithmetic in the lesson after §4.** `SM_ACTIVE` is a *breadth*
measure. A single-block kernel pinned to one SM on an H100 reads `1/132 ≈ 0.0076` — and
`GPU_UTIL` reads 100. That factor of ~130 between the two numbers, on the same device, in
the same second, is the whole exhibit.

Note the subtlety that `SM_ACTIVE` is *still* not intensity: an SM with one stalled warp
counts as active for that cycle exactly as much as an SM with 64 running warps. That is
what 1003 is for.

- **Use it for:** "is this allocated GPU idle?" — the reclaim/cost question.
- **Failure mode:** high `SM_ACTIVE` with low everything else means the SMs are lit but
  stalled — spin-waits, `__syncthreads()` storms, NCCL waits, memory-latency-bound kernels.

#### `DCGM_FI_PROF_SM_OCCUPANCY` — 1003 — *depth*

Header: *"The ratio of number of warps resident on an SM (number of resident as a ratio of
the theoretical maximum number of warps per elapsed cycle)."* GPM:
`NVML_GPM_METRIC_SM_OCCUPANCY`, *"Percentage of warps that were active vs theoretical
maximum."* Short name `SMOCC`.

The denominator is architectural, not workload-dependent:

```
  theoretical max resident warps (whole device)
      = SMs × max_warps_per_SM

  H100 SXM5 (cc 9.0):   132 × 64 = 8,448 warp slots
  A100 SXM4 (cc 8.0):   108 × 64 = 6,912 warp slots
  L4 / L40S (cc 8.9):    58/142 × 48
  V100 SXM2 (cc 7.0):    80 × 64 = 5,120 warp slots

  SM_OCCUPANCY = time-averaged (resident warps) / (that number)
```

So `SM_OCCUPANCY = 0.09` on an H100 means roughly 760 of 8,448 warp slots were occupied on
average. Occupancy is bounded from above by register pressure (64K 32-bit registers per SM
on Hopper), shared-memory-per-block, and grid size. A kernel that uses 128 registers per
thread caps at `65536 / (128 × 32) = 16` warps per SM = 0.25 occupancy no matter what you
do to the launch config.

- **Use it for:** kernel/launch-shape tuning. Low occupancy with high `SM_ACTIVE` = your
  grids are too small or your kernels too register-hungry.
- **Failure mode / trap:** high occupancy does **not** imply high throughput. Occupancy is
  a latency-hiding budget, not work. A memory-bound kernel at 100% occupancy is still
  memory-bound.

#### `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` — 1004 — *are the expensive units firing*

Header: *"The ratio of cycles the any tensor pipe is active (off the peak sustained elapsed
cycles)."* GPM: `NVML_GPM_METRIC_ANY_TENSOR_UTIL` — *"Percentage of time the GPU's SMs were
doing ANY tensor operations."* Short name `TENSO`.

**Correct a widely-copied error here:** dcgm-exporter's default CSV help string calls this
*"Ratio of cycles the tensor (HMMA) pipe is active"*, and that text has been copy-pasted
into a hundred dashboards and blog posts. It is **not HMMA-specific** — it is *any* tensor
pipe, and DCGM exposes the per-pipe breakdown separately as `DCGM_FI_PROF_PIPE_TENSOR_IMMA_ACTIVE`
(1013, integer MMA), `..._HMMA_ACTIVE` (1014, half-precision MMA), `..._DFMA_ACTIVE`
(1015, double-precision FMA tensor), and `DCGM_FI_PROF_PIPE_INT_ACTIVE` (1016). If you need
"is this FP8 or BF16 work", 1004 will not tell you; the sub-pipe fields will.

The "off the peak sustained elapsed cycles" wording matters: the denominator is the
*sustainable* issue rate, not the theoretical instantaneous one, so 1.0 is achievable in
principle by a perfectly-fed GEMM. In practice a well-tuned large-model training step lands
around 0.4–0.7 here; Meta reported roughly 38–43% model-FLOPs-utilisation for Llama 3.1
pretraining on 16,384 H100s (*The Llama 3 Herd of Models*, arXiv:2407.21783), and MFU and
`PIPE_TENSOR_ACTIVE` track each other closely for tensor-dominated training.

- **Use it for:** right-sizing and efficiency. The closest single field to "are we getting
  the FLOPs we are renting".
- **Failure mode:** low tensor activity on a training job means precision, GEMM shape, or
  data-pipeline problems — not a hardware fault, and not a reason to page.

#### `DCGM_FI_PROF_DRAM_ACTIVE` — 1005 — *is HBM the wall*

Header: *"The ratio of cycles the device memory interface is active sending or receiving
data."* GPM: `NVML_GPM_METRIC_DRAM_BW_UTIL` — *"Percentage of DRAM bw used vs theoretical
maximum."* Short name `DRAMA`. Note the entity level in `dcgm_fields.cpp` is `DCGM_FE_GPU_I`
(GPU instance) rather than `DCGM_FE_GPU_CI` like the SM fields — memory bandwidth is a
GPU-instance-level property under MIG.

Multiply it by your part's peak bandwidth to get an actual number a capacity conversation
can use: `0.71 × 3.35 TB/s ≈ 2.4 TB/s achieved` on an H100 SXM5.

- **Use it for:** distinguishing memory-bound from compute-bound, and therefore whether
  batching or a different GPU is the lever.
- **Failure mode:** high `DRAM_ACTIVE` with low `PIPE_TENSOR_ACTIVE` is the *signature* of
  healthy-but-memory-bound inference. That is not a defect; it is a batching decision.

#### `DCGM_FI_PROF_GR_ENGINE_ACTIVE` — 1001 — *the one that looks helpful and isn't*

Header: *"Ratio of time the graphics engine is active. The graphics engine is active if a
graphics/compute context is bound and the graphics pipe or compute pipe is busy."* GPM:
`NVML_GPM_METRIC_GRAPHICS_UTIL` — *"Percentage of time any compute/graphics app was active
on the GPU."* Short name `GRACT`.

Read that definition against §3 and the conclusion is uncomfortable: **`GR_ENGINE_ACTIVE`
is a Path-B reimplementation of the same presence question `GPU_UTIL` asks.** Context bound
plus pipe busy is a duty cycle, not a breadth or intensity measure. On a batch-1 decode
server it reads ~1.0 alongside `SM_ACTIVE ≈ 0.16`.

This matters enormously for lesson 05.3, because `GR_ENGINE_ACTIVE` **ships enabled** in
dcgm-exporter's default counter CSV while `SM_ACTIVE` and `SM_OCCUPANCY` ship
**commented out**. The default configuration therefore gives you two presence metrics and
zero breadth metrics, and because 1001 carries a `DCGM_FI_PROF_` prefix it *looks* like the
honest family. It is not. If your dashboard's "real" utilisation panel is built on 1001,
you have rebuilt the lie with a longer metric name.

#### The framebuffer fields — 250–254 — *the reclaim gate*

| Field | ID | Type | Meaning |
|---|---|---|---|
| `DCGM_FI_DEV_FB_TOTAL` | 250 | int64, MB | Total frame buffer |
| `DCGM_FI_DEV_FB_FREE` | 251 | int64, MB | Free frame buffer |
| `DCGM_FI_DEV_FB_USED` | 252 | int64, MB | Used frame buffer |
| `DCGM_FI_DEV_FB_RESERVED` | 253 | int64, MB | Reserved frame buffer |
| `DCGM_FI_DEV_FB_USED_RATIO` (was `FB_USED_PERCENT`) | 254 | double | `Used / (Total − Reserved)`, range 0.0–1.0 |

Units are **MB, not MiB or bytes** per the header comments (the exporter's CSV help string
says MiB; the header says MB — if a byte-exact number matters to you, verify against
`nvidia-smi` on your own hardware rather than trusting either string).

These are the *reclaim gate*. A vLLM replica that has finished serving still holds its
weights and a preallocated KV-cache arena — `SM_ACTIVE ≈ 0`, `FB_USED ≈ 60,000 MB`. Killing
it because it looks idle destroys a warm replica and pays a multi-minute model-load penalty
on the next request. **Never build a reclaim rule on activity alone.**

### 7. The renaming you will trip over

In DCGM 4.x every field in this lesson was renamed under a "DCGM Field Naming Standard",
with the old names kept as `#define` aliases in a block headed
*"Deprecated: renamed per DCGM Field Naming Standard"*. The numeric IDs did not change.

| Legacy symbol (what dcgm-exporter's CSV and your Prometheus series use) | Current primary symbol | ID |
|---|---|---|
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | `DCGM_FI_PROF_GR_ENGINE_UTIL_RATIO` | 1001 |
| `DCGM_FI_PROF_SM_ACTIVE` | `DCGM_FI_PROF_SM_UTIL_RATIO` | 1002 |
| `DCGM_FI_PROF_SM_OCCUPANCY` | `DCGM_FI_PROF_SM_OCCUPANCY_RATIO` | 1003 |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | `DCGM_FI_PROF_TENSOR_UTIL_RATIO` | 1004 |
| `DCGM_FI_PROF_DRAM_ACTIVE` | `DCGM_FI_PROF_DRAM_UTIL_RATIO` | 1005 |
| `DCGM_FI_PROF_PIPE_FP64/FP32/FP16_ACTIVE` | `DCGM_FI_PROF_FP64/FP32/FP16_UTIL_RATIO` | 1006–1008 |
| `DCGM_FI_DEV_GPU_UTIL` | `DCGM_FI_DEV_GPU_UTIL_RATIO` | 203 |
| `DCGM_FI_DEV_XID_ERRORS` | `DCGM_FI_DEV_XID_ERROR` | 230 |
| `DCGM_FI_DEV_FB_USED_PERCENT` | `DCGM_FI_DEV_FB_USED_RATIO` | 254 |

**Operational rule: reason in field IDs, write configs in whatever spelling your exporter
version accepts, and never assume a symbol name is stable across a DCGM major version.**
`DCGM_FI_DEV_GPU_UTIL_RATIO` is, confusingly, still in units of percent — the rename was
cosmetic consistency, not a units change.

### 8. The complete field table

Everything you need for the module, in one place. "Path" is A (NVML duty cycle) or B
(hardware counters / GPM). "Alert?" is the policy §9 derives.

| Field (legacy name) | ID | Path | Units | What it counts | Alert? |
|---|---|---|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | 203 | A | % (int) | fraction of NVML sample window with ≥1 kernel resident | **Never** |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | 204 | A | % (int) | fraction of window with global memory being read/written | **Never** |
| `DCGM_FI_DEV_ENC_UTIL` | 206 | A | % (int) | NVENC duty cycle (returns its own sample period) | no |
| `DCGM_FI_DEV_DEC_UTIL` | 207 | A | % (int) | NVDEC duty cycle | no |
| `DCGM_FI_DEV_FB_TOTAL / FREE / USED / RESERVED` | 250–253 | A | MB | frame-buffer accounting | gate only |
| `DCGM_FI_DEV_FB_USED_RATIO` | 254 | A | ratio | `Used/(Total−Reserved)` | warn (OOM proximity) |
| `DCGM_FI_DEV_XID_ERRORS` | 230 | A | code | last XID error code (lesson 05.5) | **yes** |
| `DCGM_FI_DEV_POWER_USAGE` | 155 | A | W | board power draw | warn |
| `DCGM_FI_DEV_GPU_TEMP` | 150 | A | °C | GPU temperature | warn |
| `DCGM_FI_DEV_SM_CLOCK` / `MEM_CLOCK` | 100 / 101 | A | MHz | clocks (throttle evidence) | warn |
| `DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION` | 156 | A | mJ | energy since boot (counter) | no |
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | 1001 | B | ratio | context bound **and** a pipe busy — a presence duty cycle | **Never** |
| `DCGM_FI_PROF_SM_ACTIVE` | 1002 | B | ratio | mean over SMs of cycles with ≥1 warp assigned — **breadth** | **yes (idle/reclaim)** |
| `DCGM_FI_PROF_SM_OCCUPANCY` | 1003 | B | ratio | resident warps ÷ architectural max — **depth** | dashboard |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | 1004 | B | ratio | cycles any tensor pipe active — **efficiency** | warn (SLO) |
| `DCGM_FI_PROF_DRAM_ACTIVE` | 1005 | B | ratio | HBM interface active cycles ÷ theoretical max | dashboard |
| `DCGM_FI_PROF_PIPE_FP64/FP32/FP16_ACTIVE` | 1006–1008 | B | ratio | non-tensor math pipes | diagnostic |
| `DCGM_FI_PROF_PCIE_TX_BYTES` / `RX_BYTES` | 1009 / 1010 | B | B/s | PCIe throughput, GPU's perspective | dashboard |
| `DCGM_FI_PROF_NVLINK_TX_BYTES` / `RX_BYTES` | 1011 / 1012 | B | B/s | aggregate NVLink throughput | dashboard |
| `DCGM_FI_PROF_PIPE_TENSOR_IMMA/HMMA/DFMA_ACTIVE` | 1013–1015 | B | ratio | per-tensor-pipe breakdown | diagnostic |
| `DCGM_FI_PROF_PIPE_INT_ACTIVE` | 1016 | B | ratio | integer pipe | diagnostic |
| `DCGM_FI_PROF_NVDEC*/NVJPG*/NVOFA*_ACTIVE` | 1017–1034 | B | ratio | media engines | media fleets only |
| `DCGM_FI_PROF_SM_CYCLES_ELAPSED_TOTAL` / `_ACTIVE_TOTAL` | 1084 / 1085 | B | cycles | raw cumulative counters behind 1002 | advanced |
| `DCGM_FI_DRIVER_VERSION` | 1 | — | label | rendered as a label, not a series | — |

### 9. The diagnostic matrix, and the alerting policy that falls out

Two fields together classify a workload; one field never does. This is the decision tree
to carry into an interview:

```
                            PIPE_TENSOR_ACTIVE (1004)
                       low  ◀──────────────────────────▶  high
              ┌────────────────────────────┬────────────────────────────┐
         high │  A · SM-BUSY, TENSOR-IDLE  │  D · HEALTHY TRAINING      │
              │  spin-waits, NCCL stalls,  │  the target state.         │
              │  non-tensor math, FP32     │  0.4–0.7 typical for a     │
   SM_ACTIVE  │  where BF16 would do.      │  tuned large-model step.   │
     (1002)   │  → 05.7 profiling ladder   │  → leave it alone          │
              ├────────────────────────────┼────────────────────────────┤
              │  B · IDLE / WASTED         │  C · rare & suspicious     │
          low │  the reclaim case.         │  few SMs lit but tensor    │
              │  Gate on FB_USED before    │  pipes hot — usually a     │
              │  acting.                   │  tiny well-fed GEMM, or a  │
              │  → RECLAIM (this is where  │  MIG slice reported at     │
              │    the money is)           │  device scope.             │
              └────────────────────────────┴────────────────────────────┘

  Overlay DRAM_ACTIVE (1005) to split quadrant A and B further:
      low SM,  low DRAM   → genuinely idle              → reclaim
      low SM,  high DRAM  → memory-bound small-batch    → batch it, don't buy GPUs
      high SM, low DRAM   → stalled on sync/launch      → Nsight Systems
      high SM, high DRAM  → memory-bound at scale       → different GPU / better kernels

  GPU_UTIL (203) is 100 in every one of these eight cases.
```

The policy, stated as rules you could paste into a runbook:

**Rule 1 — Page on `SM_ACTIVE`, gated by `FB_USED`.** The reclaim alert is "allocated and
doing no work for long enough that it is not a gap between steps".

```promql
# Allocated GPUs doing essentially no SM work for 30 minutes, and not
# holding a large resident model/KV-cache. Fires per GPU.
#
#   0.05  : 5% of SM-cycles. A between-steps gap on a healthy training job
#           never sustains below this for 30m; tune against your own fleet.
#   2048  : MB. Below ~2 GB the card is not holding a served model.
#           Raise it if your smallest served model is bigger.
(
    avg_over_time(DCGM_FI_PROF_SM_ACTIVE[30m]) < 0.05
  and on (gpu, UUID, instance)
    DCGM_FI_DEV_FB_USED < 2048
  and on (gpu, UUID, instance)
    count by (gpu, UUID, instance) (
      DCGM_FI_PROF_SM_ACTIVE{exported_namespace!=""}   # only allocated GPUs
    ) > 0
)
```

Note `exported_namespace` rather than `namespace`: dcgm-exporter's own `namespace` label
collides with the one Prometheus attaches from Kubernetes service discovery, and with
`honor_labels: false` — **the default in both Prometheus and the dcgm-exporter Helm
chart's ServiceMonitor** — Prometheus renames the exporter's version to
`exported_namespace`. Lesson 05.4 takes this apart properly; know now that which spelling
you get is a scrape-config property, not an exporter property.

**Rule 2 — Warn, never page, on `PIPE_TENSOR_ACTIVE`.** Efficiency is a conversation, not
an incident. It belongs in a weekly report and a dashboard, not in PagerDuty at 03:00.

```promql
# Training namespaces whose tensor pipes are under 15% for 2 hours.
# Route to a Slack channel, not a pager.
avg by (exported_namespace) (
  avg_over_time(DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{exported_namespace=~"train-.*"}[2h])
) < 0.15
```

**Rule 3 — Never alert on `DCGM_FI_DEV_GPU_UTIL` or `DCGM_FI_PROF_GR_ENGINE_ACTIVE`.**
Both are presence duty cycles. A reclaim rule built on either will *never fire on your most
expensive waste*, because that waste is precisely the case that keeps a kernel resident.
Keep 203 on exactly one dashboard panel: directly beside `SM_ACTIVE`, as the foil that
makes the gap visible.

**Rule 4 — The detector query.** This is the one that goes in the blog post:

```promql
# "Your dashboard is lying" — GPUs claiming ≥90% busy while <20% of SMs are lit.
# Both sides must be present, so this also proves you enabled SM_ACTIVE (lesson 05.3).
  DCGM_FI_DEV_GPU_UTIL > 90
and on (gpu, UUID, instance)
  DCGM_FI_PROF_SM_ACTIVE < 0.2
```

**Rule 5 — Alert on absence.** Because blank PROF values are *dropped*, not zeroed, a
permissions or watch failure looks like silence, and silence looks like health:

```promql
# SM_ACTIVE stopped being exported for a GPU that still reports device fields.
# This is the permissions / stale-watch failure mode, and it is invisible otherwise.
  count by (instance, gpu) (DCGM_FI_DEV_GPU_UTIL)
unless
  count by (instance, gpu) (DCGM_FI_PROF_SM_ACTIVE)
```

## Perspectives

**Developer.** From inside a training or inference process you never see `GPU_UTIL` at all
— you see step time, or tokens per second, and those are honest. The metric lies
specifically to the *platform* layer above the process. A developer glancing at
`nvidia-smi` inside a `kubectl exec` and reporting "99%, we're fine" is the seed of this
entire module: the number that is reassuring from the inside is the same number hiding the
waste from the outside. The developer-side fix is to instrument throughput, not
utilisation — a tokens/sec or samples/sec counter beats every GPU field for answering "did
my change help".

**Operator.** Your job is not to find a better dashboard widget. It is to know structurally
that field 203 is incapable of answering the question it is constantly asked, to have
`SM_ACTIVE` available (which, as 05.3 shows, requires a config change you must make
yourself), and to know that a *missing* PROF series is a failure signal rather than a zero.
The operator-visible consequence of §5 is that PROF fields have three distinct
"not-a-number" states — blank, not-supported, not-permissioned — all of which the exporter
collapses to absence.

**Hardware.** The split in §2 is a silicon fact. NVML's utilisation counter is maintained by
the driver from scheduler state; it was never wired to the SM performance monitors, so it
*cannot* be taught about occupancy no matter how the software above it is written. Going
the other way, the GPM unit that produces `SM_ACTIVE` has no visibility into whether a CUDA
context is bound — it counts cycles. The two are not layered; they are parallel. That is
also why Hopper's GPM changed the economics of Path B: one `nvmlGpmSampleGet()` returns
every metric at once, whereas the pre-Hopper DCP module had to program counter banks per
metric group, which is where multiplexing comes from (05.2).

**Economics.** The entire allocated-vs-utilised dollar argument is downstream of this one
gap. A fleet reading 99% "busy" while sitting at 15–20% `SM_ACTIVE` is not a rounding
error: at the ~$2.50–3.00/GPU-hour H100 on-demand band reported in mid-2026, 500 GPUs
running a year is roughly $11–13M, and an 80-point gap between what you pay for and what
you use is most of it. The reason it stays invisible is not that nobody looks at the
dashboard. It is that the dashboard's headline number is structurally unable to show it.

## Real-world use cases

- **NVIDIA's own official DCGM Grafana dashboard ships without `SM_ACTIVE`.** The
  dashboard JSON in `NVIDIA/dcgm-exporter` (`grafana/dcgm-exporter-dashboard.json`,
  published as grafana.com dashboard 12239) contains eight panels: GPU Temperature, GPU
  Avg. Temp, GPU Power Usage, GPU Power Total, GPU SM Clocks, **GPU Utilization**
  (`DCGM_FI_DEV_GPU_UTIL`), GPU Framebuffer Mem Used, and **Tensor Core Utilization**
  (`DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`). There is no `SM_ACTIVE` panel, no `SM_OCCUPANCY`
  panel and no `DRAM_ACTIVE` panel — verified by grepping the shipped JSON for
  `DCGM_FI_*`. **What it shows:** the most widely installed GPU dashboard in existence
  leads with the presence metric and offers exactly one efficiency metric. This is the
  "here's the lie everyone ships" exhibit, straight from the vendor, and lesson 05.3
  explains *why* (the missing fields are commented out in the default counter CSV, so the
  series does not exist to be graphed).

- **`NVIDIA/dcgm-exporter` issue #22 — "Profiling metrics not being collected."** A user on
  a Tesla K80 finds every `DCGM_FI_PROF_*` field empty, with DCGM reporting *"This request
  is serviced by a module of DCGM that is not currently loaded"* and the Profiling module
  showing "Failed to load" while Core and NvSwitch load fine. **What it shows:** Path B is
  a *loadable module with hardware prerequisites* (Volta and newer), not an intrinsic
  property of the GPU. On unsupported silicon you get Path A only, and the failure is a
  silently missing series rather than an error on the dashboard — exactly the case Rule 5
  above exists to catch.

- **`NVIDIA/dcgm-exporter` issue #34 — profiling privileges.** On an A100 node with MIG
  enabled on one device, dcgm-exporter failed to start with *"CacheManager Init Failed.
  Error: -17"* / *"Error starting nv-hostengine: DCGM initialization error"*, printing the
  warning *"dcgm-exporter doesn't have sufficient privileges to expose profiling metrics.
  To get profiling metrics with dcgm-exporter, use `--cap-add SYS_ADMIN`."* **What it
  shows:** Path B needs elevated privileges that Path A does not, and the gap between them
  produces a half-populated dashboard rather than a loud failure. Current versions have
  learned from this — the dcgm-exporter Helm chart now ships
  `securityContext.capabilities.add: ["SYS_ADMIN"]` with the inline comment *"Required for
  profiling metrics (DCGM_FI_PROF_*)"* — but any hardened deployment that drops
  capabilities re-creates the bug. Lesson 05.2 owns this in depth.

- **Meta, *The Llama 3 Herd of Models* (arXiv:2407.21783).** Llama 3.1 pretraining on a
  16,384×H100 cluster ran at roughly 38–43% model-FLOPs-utilisation. **What it shows:** the
  calibration point that makes the module's numbers meaningful. If a team with that budget
  and that much tuning effort tops out below half of theoretical peak, then a serving fleet
  at 0.01 `PIPE_TENSOR_ACTIVE` is not a little inefficient — it is three orders of
  magnitude away, and `GPU_UTIL` reads 100 for both.

- **Datadog's DCGM GPU-monitoring product (2025).** Its metric taxonomy splits device-level
  fields (`gpu.memory.free`, `gpu.temperature`), process-level attribution
  (`gpu.process.sm_active`), pipeline-level activity (`gpu.tensor_active`, `gpu.fp16_active`,
  `gpu.sm_occupancy`) and health (`gpu.errors.xid.total`). **What it shows:** a commercial
  vendor independently arrived at the same taxonomy this lesson teaches — the Path A / Path
  B / attribution / health split is not a course conceit, it is what you converge on when
  you have to sell the answer.

## Worked example

**Setup.** A serving namespace `svc-chat` holds 8× H100 SXM5. The Grafana tile — the stock
dashboard 12239, panel "GPU Utilization" — is solid green at 99%. The team has opened a
ticket requesting 8 more H100s, citing the panel. You have ten minutes before the capacity
meeting.

**Step 1 — get both paths on screen for one GPU.** SSH to the node and run `dcgmi dmon`
directly rather than fighting Grafana. The field list is `203` (GPU_UTIL), `1002`
(SM_ACTIVE), `1003` (SM_OCCUPANCY), `1004` (TENSOR), `1005` (DRAM), `252` (FB_USED):

```console
$ dcgmi discovery -l
2 GPUs found (Active).
+--------+----------------------------------------------------------------------+
| GPU ID | Device Information                                                   |
+========+======================================================================+
| 0      | Name: NVIDIA H100 80GB HBM3                                          |
|        | PCI Bus ID: 00000000:1B:00.0                                         |
|        | Device UUID: GPU-8f2c1a44-9b0d-5e17-a3c8-6d21e4b7f095                |
+--------+----------------------------------------------------------------------+
| 1      | Name: NVIDIA H100 80GB HBM3                                          |
|        | PCI Bus ID: 00000000:43:00.0                                         |
|        | Device UUID: GPU-c47e93b1-2f6a-51d8-8b42-19ac05fe7d63                |
+--------+----------------------------------------------------------------------+

$ dcgmi dmon -e 203,1002,1003,1004,1005,252 -i 0 -d 1000
#Entity        GPUTL       SMACT   SMOCC   TENSO   DRAMA   FBUSD
ID
GPU 0          99          0.163   0.088   0.011   0.706   58122
GPU 0          100         0.157   0.085   0.009   0.718   58122
GPU 0          100         0.171   0.093   0.013   0.699   58122
GPU 0          99          0.160   0.087   0.010   0.712   58122
GPU 0          100         0.166   0.090   0.012   0.704   58122
```

Read the columns: `dcgmi dmon` uses each field's registered five-character short name and
prints doubles to three decimals (`GPUTL` = 203, `SMACT` = 1002, `SMOCC` = 1003, `TENSO` =
1004, `DRAMA` = 1005, `FBUSD` = 252). The header reprints every 14 rows. Blank values
appear as `N/A` — if `SMACT` were `N/A` here you would be looking at a privileges problem,
not an idle GPU. Default polling is `-d 1000` ms; this transcript is representative of a
batch-1 decode workload on an H100, not a capture from a specific machine.

**Step 2 — interpret, in the order that builds the argument.**

| Reading | Value | What it proves |
|---|---|---|
| `GPUTL` | 99–100 | ≥1 kernel resident essentially always. True. Uninformative. |
| `SMACT` | 0.16 | ~21 of 132 SM-equivalents lit. **86% of the compute die is dark.** |
| `SMOCC` | 0.09 | ~760 of 8,448 warp slots occupied. Grids are tiny. |
| `TENSO` | 0.011 | Tensor pipes issuing ~1% of cycles. Matches the §4 derivation (0.34% floor, plus attention and inefficiency). |
| `DRAMA` | 0.71 | ≈ 0.71 × 3.35 TB/s ≈ **2.4 TB/s achieved**. HBM is the actual bottleneck. |
| `FBUSD` | 58,122 MB | ~58 GB resident — weights plus KV-cache arena. **The card cannot simply be reclaimed.** |

**Step 3 — the diagnosis, in one sentence.** *The fleet is not compute-saturated; it is
memory-bandwidth-bound at batch size 1, using 1% of its tensor throughput and 16% of its
SM breadth, and buying more of the same hardware will reproduce the same 1%.*

**Step 4 — prove the fix moves the truthful metrics and not the lie.** The team enables
continuous batching (vLLM `--max-num-seqs` raised, scheduler switched from one-request-at-a-time)
and you re-pull the same fields across the rollout:

| Phase | `GPUTL` | `SMACT` | `SMOCC` | `TENSO` | `DRAMA` | Throughput (rel.) |
|---|---|---|---|---|---|---|
| Before (effective batch 1) | 99 | 0.16 | 0.09 | 0.011 | 0.71 | 1.0× |
| Mid-rollout (batch ~8) | 99 | 0.34 | 0.21 | 0.11 | 0.74 | ~1.8× |
| After (continuous batching, batch ~32) | **99** | **0.55** | **0.38** | **0.19** | **0.68** | **~2.9×** |

Why the numbers move the way they do, mechanically: batching amortises each weight read
across *B* sequences, so bytes-moved per token falls by ~B while FLOPs per token stay
constant — arithmetic intensity rises from ~1 to ~B FLOP/byte and the workload walks
rightward along the roofline toward the ridge at 295. `TENSO` rises ~17× because the tensor
pipes now have real GEMMs (matrix×matrix) instead of GEMVs (matrix×vector). `SMACT` rises
because larger batches produce larger grids that fill more SMs. `DRAMA` stays roughly flat
because HBM is still working hard — it is simply doing useful work for 32 sequences instead
of one.

**And `GPUTL` never moves.** It was 99 before the fix, during the fix, and after the fix.
This is the strongest form of the lesson: **field 203 is uninformative about the problem
*and* uninformative about the solution.** A dashboard built on it would have shown a
perfectly flat line while the fleet nearly tripled its useful output.

**Step 5 — the money sentence for the meeting.** Serving throughput rose ~2.9× on the same
eight cards. The 8-GPU expansion is unnecessary. At ~$2.50–3.00/GPU-hour (the mid-2026
H100 on-demand band; verify against your own contract, and state the date when you quote
it), 8 GPUs × 8,760 hours is roughly **$175k–210k/year avoided** — plus the reclaimed
headroom to absorb the next quarter's traffic growth without buying anything. That sentence
is the interview answer, and `GPU_UTIL` would have hidden every part of it.

## Practice

On a rented GPU (H100/A100/L4 from a neocloud, Lambda or RunPod) with DCGM +
dcgm-exporter + Prometheus + Grafana already stood up from module 04, building directly
toward ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md).
If you have no hardware yet, develop every query against the
[fake GPU fleet lab](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md) first
and spend the rented hours validating rather than debugging.

1. **Confirm you can even see the truth.** Query `DCGM_FI_PROF_SM_ACTIVE` in Prometheus.
   If it returns nothing, do not debug PromQL — the field ships commented out in
   dcgm-exporter's default counter CSV (lesson 05.3) and may additionally need
   `SYS_ADMIN` (lesson 05.2). Note which of the two it was; that note goes in the write-up.

2. **Generate the lie.** Run a batch-1 autoregressive decode loop. Cleanest options, in
   descending order of realism: a vLLM server with `--max-num-seqs 1` driven by a
   single-threaded client; a HuggingFace `model.generate(..., num_beams=1)` on a 7–8B model
   in a `while True`; or, with no model at all, a tight loop launching one small
   `cublasSgemv` per iteration. The requirement is *a continuous stream of small kernels*,
   nothing more.

3. **Capture both paths side by side.** Run
   `dcgmi dmon -e 203,1002,1003,1004,1005,252 -d 1000` on the node and let it scroll for
   at least a minute. Separately, build a single Grafana panel with `DCGM_FI_DEV_GPU_UTIL / 100`
   and `DCGM_FI_PROF_SM_ACTIVE` on the same axis so both are in `0..1` and the gap is
   visually undeniable. **The `/100` matters** — plotting 99 against 0.16 on one axis makes
   the honest series a flat line at the bottom.

4. **Do the arithmetic for your own hardware.** Look up your GPU's SM count, max warps per
   SM, HBM bandwidth and dense tensor throughput; redo the §4 calculation for the model you
   actually ran; and check whether your measured `PIPE_TENSOR_ACTIVE` is within an order of
   magnitude of your predicted `t_compute / t_memory`. If it is not, work out why before
   moving on — that is the real learning.

5. **Reproduce the dynamic exhibit.** If you have a batching-capable server, raise the
   batch limit mid-run and capture the same fields across the transition: `GPU_UTIL` flat,
   `SM_ACTIVE` and `PIPE_TENSOR_ACTIVE` climbing. This is a stronger artifact than the
   static screenshot because it kills the "maybe the GPU really was idle" objection.

6. **Write the absence alert.** Add Rule 5's `unless` query as a real Prometheus rule.
   Then prove it works: drop `SYS_ADMIN` from the exporter's `securityContext`, confirm
   the PROF series vanish while device series continue, and confirm the alert fires.

**Acceptance:** a side-by-side capture from your own cluster showing
`DCGM_FI_DEV_GPU_UTIL ≈ 100` beside `SM_ACTIVE` and `PIPE_TENSOR_ACTIVE` near zero, plus a
short note recording (a) the arithmetic you did in step 4 with your hardware's numbers, and
(b) whether the missing-metric absence alert fired when you removed the capability. The
screenshot is the flagship exhibit of the deliverable; the note is the paragraph in the
blog post that makes readers believe it.

## Common pitfalls

1. **Comparing 203 and 1002 without dividing by 100.** Field 203 is an integer percentage;
   the PROF fields are `0.0–1.0` doubles. `DCGM_FI_DEV_GPU_UTIL > DCGM_FI_PROF_SM_ACTIVE`
   is true for every busy GPU on earth and means nothing. **Mechanism:** DCGM stores 203 as
   the raw `unsigned int` from `nvmlUtilization_t.gpu` and stores PROF fields after
   dividing GPM's percentage by 100.

2. **Treating a missing PROF series as zero.** dcgm-exporter drops blank values entirely
   (`toString()` returns the `"SKIPPING DCGM VALUE"` sentinel and the collector skips the
   sample), so a permissions failure, an unsupported GPU, or a not-yet-warm watch all
   produce *absence*. **Symptom:** `avg by (namespace)(DCGM_FI_PROF_SM_ACTIVE)` looks
   healthy because the broken GPUs are not in the average at all. **Fix:** alert on
   absence (Rule 5), and never let a PROF field be the sole denominator of a cost figure
   without a presence check.

3. **Using `GR_ENGINE_ACTIVE` (1001) as "the honest one" because it has a `PROF` prefix.**
   Its own definition is a presence duty cycle: context bound *and* a pipe busy.
   **Symptom:** you "fixed" the dashboard, the number is still ~1.0 on an idle-but-resident
   decode server, and you conclude the lie was overstated. **Mechanism:** it maps to
   `NVML_GPM_METRIC_GRAPHICS_UTIL`, not to any SM-breadth counter.

4. **Reading `MEM_COPY_UTIL` (204) as bandwidth.** It is the percentage of the NVML sample
   window during which device memory was being touched *at all* — same presence semantics
   as 203, applied to memory. **Symptom:** `MEM_COPY_UTIL` at 95 while a bandwidth
   benchmark shows you are at 8% of peak. **Fix:** `DCGM_FI_PROF_DRAM_ACTIVE` (1005), and
   multiply by your part's peak bandwidth to get a number in TB/s.

5. **Reclaiming on `SM_ACTIVE` alone.** A paused-but-loaded serving replica has
   `SM_ACTIVE ≈ 0` and 58 GB of resident weights and KV-cache. **Symptom:** you reclaim a
   warm replica and eat a multi-minute cold start on the next request, then get told the
   reclaim system is dangerous and it gets switched off. **Fix:** gate every reclaim rule
   on `DCGM_FI_DEV_FB_USED`.

6. **Assuming `GPU_UTIL` exists everywhere.** `nvmlDeviceGetUtilizationRates()` is
   explicitly unsupported on MIG-enabled GPUs. **Symptom:** a fleet panel that goes blank
   for exactly the nodes you switched to MIG last week. **Mechanism:** NVML returns an
   error, DCGM stores a blank, the exporter omits the series.

7. **Treating the NVML sample period as a knob or a constant.** It is 1 s to 1/6 s,
   product-dependent, chosen by the driver, and not readable for `.gpu`. Setting DCGM's
   `updateFreq` to 100 ms does not make field 203 finer-grained; it makes you re-read the
   same driver-integrated number more often.

8. **Assuming symbol names are stable.** In DCGM 4.x, `DCGM_FI_PROF_SM_ACTIVE` is an alias
   for `DCGM_FI_PROF_SM_UTIL_RATIO`, and `DCGM_FI_DEV_GPU_UTIL` for
   `DCGM_FI_DEV_GPU_UTIL_RATIO`. The IDs (1002, 203) did not move. **Rule:** reason in IDs.

## Self-check

- **State what `DCGM_FI_DEV_GPU_UTIL` measures in one sentence without using the word
  "utilisation".** *Answer:* it is the percentage of a short, driver-chosen sample window
  (1 s to 1/6 s, product-dependent) during which at least one kernel was resident on the
  GPU. It is field 203, an unmodified passthrough of `nvmlUtilization_t.gpu` obtained from
  `nvmlDeviceGetUtilizationRates()`. It has no notion of how many SMs were used, how full
  they were, or which pipes ran — it evaluates a one-bit predicate and integrates it.

- **Why does a batch-1 LLM decode read 100% `GPU_UTIL`?** *Answer:* decode emits an
  unbroken stream of tiny kernels — one GEMV per weight matrix plus attention over the KV
  cache — enqueued back to back on one stream, with inter-kernel gaps of microseconds
  against a sample window of at least ~167 ms. The predicate "≥1 kernel resident" is
  therefore true for essentially the whole window. Meanwhile each kernel spends its life
  waiting on HBM: for an 8B BF16 model on an H100 SXM5, reading 16 GB of weights at
  3.35 TB/s takes ~4.78 ms while the 16 GFLOP of arithmetic takes ~16 µs at 989 TFLOP/s, so
  the tensor pipes are busy ~0.34% of the time. Presence is 100%; work is under 1%.

- **`SM_ACTIVE = 0.9` with `TENSOR_ACTIVE = 0.1` — what is the workload doing wrong?**
  *Answer:* the SMs are broadly lit (90% of SM-cycles have ≥1 warp assigned) but the tensor
  pipes are nearly idle. The work is running on the wrong units. Common causes: FP32
  non-tensor math where BF16/FP8 would dispatch to tensor cores; GEMM shapes that do not
  tile onto the tensor path; elementwise/normalisation kernels dominating; or warps that
  are resident but stalled — spin-waits, `__syncthreads()` contention, or an NCCL
  collective blocked on a straggler rank (the wait *is* a resident kernel). Check
  `DRAM_ACTIVE` to split them: high DRAM means memory-latency/bandwidth stalls, low DRAM
  means synchronisation or launch overhead, and low-DRAM-low-tensor-high-SM is the
  signature that sends you to Nsight Systems (lesson 05.7).

- **Which field do you page on for "idle but allocated", and why not `GPU_UTIL`?**
  *Answer:* `DCGM_FI_PROF_SM_ACTIVE` (1002), gated on `DCGM_FI_DEV_FB_USED` (252) so you
  do not reclaim a loaded-but-paused replica, and scoped to GPUs carrying a pod/namespace
  label from the pod-resources join. Not `GPU_UTIL`, because 203 reads ~100 on exactly the
  workloads you most need to catch — batch-1 decode, spin-wait kernels, NCCL stalls,
  dataloader-starved training. A reclaim rule on 203 never fires on your most expensive
  waste. Not `GR_ENGINE_ACTIVE` (1001) either, for the same reason: it is a presence duty
  cycle wearing a `PROF` prefix.

- **Why is `DCGM_FI_DEV_MEM_COPY_UTIL` (204) not a substitute for
  `DCGM_FI_PROF_DRAM_ACTIVE` (1005)?** *Answer:* they come from different paths. 204 is
  `nvmlUtilization_t.memory` — the percentage of the NVML sample window during which global
  device memory was being read or written at all, a presence duty cycle. 1005 is a hardware
  counter ratio of cycles the DRAM interface was actively moving data, relative to its
  theoretical maximum, so multiplying it by peak bandwidth gives an achieved TB/s. A
  workload issuing one small read per microsecond drives 204 to ~100 while using a fraction
  of a percent of real bandwidth.

- **What does `SM_ACTIVE = 0.24` on an H100 SXM5 tell you about how many SMs were busy?**
  *Answer:* roughly 32 of 132 SM-equivalents were lit, because `SM_ACTIVE` is the ratio of
  summed active cycles to summed elapsed cycles across all SMs. It does *not* tell you they
  were 24% busy each — it is equally consistent with 32 SMs at 100% and 132 SMs at 24%.
  Distinguishing those needs `SM_OCCUPANCY` (how full the lit SMs were) or a profiler
  (05.7).

- **A GPU's `SM_ACTIVE` series disappears from Prometheus while its `GPU_TEMP` keeps
  reporting. What happened, and why is this dangerous?** *Answer:* the PROF field returned
  a DCGM blank — `NOT_PERMISSIONED` (the container lost `SYS_ADMIN`), `NOT_SUPPORTED`
  (pre-Volta silicon, or the profiling module failed to load), or simply no baseline sample
  yet after a watch restart. dcgm-exporter checks `DCGM_FP64_IS_BLANK` and omits the sample
  rather than exporting a zero. It is dangerous because absence is not zero in PromQL:
  `avg by (namespace)` silently drops the GPU, so a fleet where half the cards stopped
  reporting the honest metric shows an *unchanged, healthy-looking* average. Alert on
  absence with an `unless` between a device field and a PROF field.

- **What is the averaging window of `DCGM_FI_PROF_SM_ACTIVE`?** *Answer:* whatever the
  watch's `updateFreq` is — the value is computed by differencing two GPM snapshots
  separated by that interval. dcgm-exporter's default collect interval is 30,000 ms, so out
  of the box every `SM_ACTIVE` sample is a ~30-second mean. DCGM retains samples for
  `2 × maxUpdateInterval + min(maxUpdateInterval, 1 s)` so that jitter cannot leave it
  without a baseline. It is *not* a fixed hardware constant, and changing it changes the
  meaning of the number, not just its freshness.

## Connections & what's next

This lesson gives you the field IDs, the two collection paths, and the mechanism behind the
lie. It deliberately stops short of *how* DCGM is deployed and what Path B costs — that is
lesson 05.2, immediately next, which takes apart `nv-hostengine` (standalone vs embedded),
field groups and watches, the profiling module's counter-bank multiplexing on pre-Hopper
silicon, the `DCGM_ST_PROFILING_MULTI_PASS` error, and the scrape-interval band that falls
out of the §5 sampling model. Lesson 05.3 then shows that `SM_ACTIVE` and `SM_OCCUPANCY`
ship **commented out** in dcgm-exporter's default counter CSV — the metric this lesson tells
you to page on is not collected until you turn it on — and works the cardinality arithmetic
for a real fleet. Lesson 05.4 takes the now-truthful series and joins them to namespaces
through the pod-resources API, including the case where the join is impossible. Lesson 05.5
reuses the same field-ID discipline for XID health codes, 05.7 escalates from these metrics
to PyTorch Profiler and Nsight when the quadrant diagram says "stalled", and the capstone
(05.8) turns the `GPU_UTIL`/`SM_ACTIVE` divergence you derived here into a per-namespace
dollar figure.

## References & further reading

**Primary sources**

- `NVIDIA/DCGM` — `dcgmlib/dcgm_fields.h` (DCGM 4.6.0) — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_fields.h — the authoritative field-ID list. Read for: the numeric IDs, the per-field doc comments quoted throughout, and the "Deprecated: renamed per DCGM Field Naming Standard" alias block that maps `DCGM_FI_PROF_SM_ACTIVE` → `DCGM_FI_PROF_SM_UTIL_RATIO` (§7). *Correction vs. the previous version of this lesson: field 1004 is "any tensor pipe", not HMMA-specific; the HMMA-only field is 1014.*
- `NVIDIA/DCGM` — `dcgmlib/src/DcgmCacheManager.cpp` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/src/DcgmCacheManager.cpp — read the shared `case DCGM_FI_DEV_GPU_UTIL_RATIO: case DCGM_FI_DEV_MEM_COPY_UTIL:` handler to see that fields 203 and 204 are one `nvmlDeviceGetUtilizationRates()` call with no post-processing.
- `NVIDIA/DCGM` — `dcgmlib/src/DcgmGpmManager.cpp` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/src/DcgmGpmManager.cpp — the whole of §5: the sample map, `MaybeFetchNewSample()`, the baseline lookup at `latest − updateInterval`, the `/100` percentage conversion, and the `2×interval + jitter` retention derivation.
- `NVIDIA/DCGM` — `dcgmlib/src/dcgm_fields.cpp` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/src/dcgm_fields.cpp — field metadata: the `dcgmi dmon` short names (`GPUTL`, `SMACT`, `SMOCC`, `TENSO`, `DRAMA`), column widths, and the `DcgmFieldIdToNvmlGpmMetricId()` mapping from DCGM field IDs to `NVML_GPM_METRIC_*`.
- `NVIDIA/go-nvml` — `gen/nvml/nvml.h` (NVML API version 13) — https://github.com/NVIDIA/go-nvml/blob/main/gen/nvml/nvml.h — the `nvmlUtilization_t` struct comment (the definition of the lie), the `nvmlDeviceGetUtilizationRates()` notes on MIG and ECC scrubbing, the `nvmlGpmQueryDeviceSupport()` "Hopper or newer" statement, and the `NVML_GPM_METRIC_*` enum with its per-metric descriptions.
- `NVIDIA/dcgm-exporter` — `etc/default-counters.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — which of these fields ship enabled. Audited in full in lesson 05.3; note here only that 1001, 1004 and 1005 are enabled and 1002/1003 are not.
- `NVIDIA/dcgm-exporter` — `internal/pkg/collector/gpu_collector.go` — https://github.com/NVIDIA/dcgm-exporter/blob/main/internal/pkg/collector/gpu_collector.go — `toString()` and the `skipDCGMValue` sentinel: the mechanism behind "blank PROF values become absent series, not zeros".
- NVIDIA DCGM field-ID reference (rendered docs) — https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html — the same table as `dcgm_fields.h` in browsable form. The header is the more current source; use the docs when you want prose.
- NVIDIA Hopper Tuning Guide — https://docs.nvidia.com/cuda/hopper-tuning-guide/index.html — read for compute-capability 9.0 limits used in §6: 64 maximum resident warps per SM, 2048 threads per SM, 64K 32-bit registers per SM.

**Real-world engineering blogs and issues**

- `NVIDIA/dcgm-exporter` issue #22, "Profiling metrics not being collected" — https://github.com/NVIDIA/dcgm-exporter/issues/22 — what it shows: on unsupported silicon the DCGM Profiling module reports "Failed to load" and every PROF field silently disappears; Path B is a loadable module with hardware prerequisites, not a given.
- `NVIDIA/dcgm-exporter` issue #34, "Error starting nv-hostengine: DCGM initialization error" — https://github.com/NVIDIA/dcgm-exporter/issues/34 — what it shows: the documented `--cap-add SYS_ADMIN` warning and an A100/MIG init failure; the origin of the `securityContext.capabilities.add: ["SYS_ADMIN"]` default now shipped in the exporter's Helm chart.
- `NVIDIA/dcgm-exporter` — `grafana/dcgm-exporter-dashboard.json` — https://github.com/NVIDIA/dcgm-exporter/blob/main/grafana/dcgm-exporter-dashboard.json — what it shows: the official dashboard (grafana.com #12239) has eight panels and no `SM_ACTIVE`, `SM_OCCUPANCY` or `DRAM_ACTIVE`; the lie ships in the box.
- Datadog — GPU Monitoring Reference Architecture — https://www.datadoghq.com/architecture/gpu-monitoring/ — what it shows: a commercial vendor's DCGM taxonomy splits device / process / pipeline / health exactly along this lesson's seams.

**Deeper dives**

- Meta — *The Llama 3 Herd of Models* (arXiv:2407.21783), §3.3.2 — https://arxiv.org/abs/2407.21783 — go deeper on: real training MFU (roughly 38–43% on 16,384 H100s) as the calibration point for how far even a heavily tuned job sits below theoretical peak, and therefore how far a 1%-tensor-active decode fleet really is.
- Superorbital — "GPU Underutilization in Kubernetes" — https://superorbital.io/blog/gpu-kubernetes-underutilization/ — go deeper on: the same field-ID gap argued end to end from a platform-engineering vantage point; useful as a narrative model for your own deliverable write-up.
</content>
</invoke>
