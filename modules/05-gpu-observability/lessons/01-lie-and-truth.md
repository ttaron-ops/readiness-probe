---
lesson: "05.1"
title: "The lie and the truth: GPU_UTIL vs the PROF metrics"
module: "05"
concept: "The lie and the truth: GPU_UTIL vs the PROF metrics"
status: not-started
est_time: "5h"
artifacts: []
---

# 05.1 · The lie and the truth: GPU_UTIL vs the PROF metrics

> **Concept.** `DCGM_FI_DEV_GPU_UTIL` measures whether a kernel was *resident*, not whether the silicon was *working* — the four PROF metrics measure the work, and each drives a different money decision.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Why this matters

This is the lesson that turns "the GPUs look busy" into "the GPUs are burning money,"
and it is the single most quoted result in a platform-cost interview at CoreWeave,
NVIDIA, Datadog, or any neocloud. When a hiring panel asks "our fleet dashboard shows
95% GPU utilization but the training team says throughput is bad — reconcile that," they
are asking for exactly this: the difference between a kernel being *resident* and the
silicon doing *work*. Every dollar figure your cost operator (module 11) emits sits
downstream of choosing the right numerator here; alert on the wrong metric and you page
on nothing while the fleet idles at $2–4/GPU-hour.

## What's new for you

Module **03** taught the *concept* of the utilisation lie — GPU-Util vs SM-occupancy vs
tensor-active vs MFU, and the roofline that explains why decode is memory-bound. Module
**04** gave you dcgm-exporter deployment and the pod-resources join. You already run
Prometheus/PromQL/Grafana/alerting across 40+ clusters. **None of that is re-taught
here.** What this lesson adds is the *exact DCGM wiring*: the precise field IDs, the
one-sentence semantics of each (the words that survive an interview), which come from
NVML vs the profiling subsystem, the real H100 numbers, and the alerting policy that
says *page on this field, never that one*. This is the concept lesson made operational
in DCGM field IDs.

## Core notes

### The lie: `DCGM_FI_DEV_GPU_UTIL` (field ID 203)

State it in one sentence, without the word "utilisation":

> **`DCGM_FI_DEV_GPU_UTIL` is the fraction of the sample window during which at least one kernel was resident on the GPU.**

That is the *literal* NVML definition. Field 203 is a straight passthrough of
`nvmlDeviceGetUtilizationRates().gpu`, which NVML documents as "Percent of time over the
past sample period during which one or more kernels was executing on the GPU." Note
every load-bearing word:

- **"one or more kernels"** — a *count* threshold of ≥1, not an intensity. One kernel and
  ten thousand kernels both read 100.
- **"executing" / resident** — scheduled on the GPU, not necessarily computing. A kernel
  stalled on an HBM load is still "executing."
- **"past sample period"** — NVML's own internal window (order ~1s, driver-decided),
  *then* re-sampled by DCGM at your update frequency. It is a duty cycle of *presence*,
  integrated over that window.

There is no notion of *how many* SMs, *how full* they are, or *which* pipes ran. It is a
binary "was anything on the GPU?" averaged to a percentage. That binary is why it lies.

`DCGM_FI_DEV_MEM_COPY_UTIL` (field 204) is its sibling from the same NVML call —
"percent of time the memory *controller* was read or written" — and it lies the same
way: it is a duty-cycle of the copy engine, not achieved HBM bandwidth. Do not confuse
it with `DCGM_FI_PROF_DRAM_ACTIVE`.

### Why batch-1 autoregressive decode pins GPU_UTIL to 100%

Autoregressive LLM decode generates **one token per forward pass**. At batch size 1,
each forward pass is a sequence of *tiny* kernels — a GEMV against each weight matrix
(matrix × vector, not matrix × matrix), plus attention over the KV-cache. Each kernel:

- launches, reads weights from HBM, does a sliver of math, writes back, exits;
- is immediately followed by the next kernel in the layer stack;
- the whole token takes, say, 10–20 ms, and across that entire time **there is always
  exactly one kernel resident** on the GPU.

So the "≥1 kernel resident" predicate is true for ~100% of every sample window →
**`GPU_UTIL` reads 100**. Meanwhile a GEMV uses a handful of SMs for a few microseconds
each and spends most of the kernel's wall-clock *waiting on HBM* — the SMs are
idle-but-occupied. The math is memory-bound (module 03's roofline: arithmetic intensity
of a GEMV is ~1–2 FLOP/byte, far left of the ridge), so the tensor cores essentially
never fire.

**Real case (H100, Llama-class 7B, batch 1, FP16 decode):**

| Field | Value | Reading |
|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` (203) | **100** | a kernel is always resident |
| `DCGM_FI_PROF_SM_ACTIVE` (1002) | **0.18** | only ~18% of SM-cycles had a warp |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (1004) | **~0.02** | tensor cores idle — it's GEMV, not GEMM |
| `DCGM_FI_PROF_DRAM_ACTIVE` (1005) | **0.6–0.8** | HBM is the bottleneck |

The dashboard says 100% busy. The truth is you are paying for a full H100 to run at ~18%
breadth and ~2% tensor throughput, memory-bound. That gap — 100 vs 0.18 — is the
flagship exhibit of the whole module.

### The truth: the four PROF metrics + framebuffer

These come from the **profiling subsystem** (field IDs 1001–1012), sampled from hardware
performance counters — a fundamentally different, more expensive path than NVML (that
cost and its multiplexing is lesson 05.2). They are **ratios in [0.0, 1.0]**, not
percentages. The four that matter, and the decision each drives:

1. **`DCGM_FI_PROF_SM_ACTIVE` (1002) — breadth.**
   *The fraction of elapsed cycles during which at least one warp was resident on an SM,
   averaged over all SMs.* This is the honest answer to "is the GPU doing anything?" Low
   SM_ACTIVE with a GPU allocated = **idle but paid for**. **Decision: cost / reclaim.**

2. **`DCGM_FI_PROF_SM_OCCUPANCY` (1003) — depth.**
   *The fraction of warp *slots* filled — resident warps ÷ the SM's maximum warp capacity,
   averaged over time and over all SMs.* SM_ACTIVE says "an SM had ≥1 warp"; OCCUPANCY says
   "how full was it." High breadth + low depth = a launch/occupancy problem (small grids,
   register/shared-memory pressure). **Decision: kernel/config tuning, batch size.**

3. **`DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (1004) — tensor throughput.**
   *The fraction of cycles the tensor (HMMA) pipe was active.* This is the closest single
   field to "are we getting the FLOPs we paid for" for training and prefill. It is the
   numerator of a rough MFU estimate. Low here on a training job means you are compute-idle
   on the most expensive units on the die. **Decision: right-sizing, precision (module
   03.5), kernel choice.**

4. **`DCGM_FI_PROF_DRAM_ACTIVE` (1005) — memory-bandwidth duty cycle.**
   *The fraction of cycles the device-memory (HBM) interface was actively sending or
   receiving.* Decode is memory-bound, so *high* DRAM_ACTIVE with *low* TENSOR_ACTIVE is
   the signature of healthy-but-memory-bound inference — not a defect, but the signal that
   batching or a bigger model is the only way to raise tensor throughput. **Decision:
   batch/throughput tuning, workload placement.**

Plus one residency field:

- **`DCGM_FI_DEV_FB_USED` (252) — framebuffer bytes in use** (with `FB_FREE` 251,
  `FB_TOTAL` 250, `FB_RESERVED` 253). For inference this is **KV-cache residency**: a
  decode server can be idle on compute (SM_ACTIVE ~0) yet hold 60 GB of KV-cache, meaning
  the GPU is *memory-pinned* and cannot be reclaimed even though it looks idle. FB_USED is
  what stops you from wrongly reclaiming a paused-but-loaded serving replica. **Decision:
  reclaim safety, memory right-sizing.**

Related, sometimes useful: `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (1001, graphics/compute
engine active — a coarser "was the engine busy"), and the precision-specific pipes
`PIPE_FP16_ACTIVE` (1008), `PIPE_FP32_ACTIVE` (1007), `PIPE_FP64_ACTIVE` (1006) when you
need to know *which* math units ran.

### The alerting policy (the load-bearing rule)

You already run alertmanager at scale; here is the GPU-specific policy. The mental
model: **breadth answers "is it idle?" (a cost question); tensor/DRAM answer "is it
efficient?" (a right-sizing question); GPU_UTIL answers nothing you can act on.**

- **"Idle but allocated" → page/reclaim on `SM_ACTIVE`.** A GPU is `Allocated`
  (pod-resources join from 04) but `SM_ACTIVE` sits below threshold for a sustained window
  → wasted spend. Gate with `FB_USED` so you don't reclaim a loaded-but-paused replica.

  ```promql
  # GPUs allocated to a pod but doing ~no SM work for 30m
  (
    DCGM_FI_PROF_SM_ACTIVE < 0.05
    and on (gpu, UUID) (DCGM_FI_DEV_FB_USED < 2048)   # not holding a KV-cache
  )
  # join to pod/namespace via the kube_pod_* / dcgm exporter labels from 04
  ```

- **"Efficient?" → dashboard/warn on `PIPE_TENSOR_ACTIVE` and `DRAM_ACTIVE`.** A running
  training job with `TENSOR_ACTIVE` far below its roofline expectation is a
  right-sizing/tuning signal, not a page. Pair the two: high DRAM + low TENSOR =
  memory-bound (expected for decode, a batching lever for training); low DRAM + low TENSOR
  = genuinely stalled (input pipeline, sync, small kernels).

- **NEVER alert on `DCGM_FI_DEV_GPU_UTIL`.** It cannot distinguish a 100%-tensor GEMM from
  a 2%-tensor GEMV; both read 100. An SLO or reclaim rule built on field 203 will *never
  fire* on the most expensive failure mode you have — a fleet of decode servers pinned at
  100% GPU_UTIL and 18% SM_ACTIVE. Put it on a dashboard only as the *foil*, next to
  SM_ACTIVE, to make the lie visible.

## Worked example

A serving namespace holds 8× H100. The Grafana tile (built on GPU_UTIL) is solid green
at **99%**; the platform team is about to approve a request for 8 *more*. You pull the
PROF fields for one representative GPU over a 10-minute window:

```
DCGM_FI_DEV_GPU_UTIL          = 99      # "busy"
DCGM_FI_PROF_SM_ACTIVE        = 0.16    # breadth: ~16% of SM-cycles had a warp
DCGM_FI_PROF_SM_OCCUPANCY     = 0.09    # depth: warp slots nearly empty
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE = 0.03  # tensor cores essentially idle
DCGM_FI_PROF_DRAM_ACTIVE      = 0.71    # HBM interface ~71% busy
DCGM_FI_DEV_FB_USED           = 58000   # ~58 GB resident (KV-cache)
```

**Interpretation.** GPU_UTIL=99 is the resident-kernel lie: batch-1 decode keeps one
small kernel resident continuously. SM_ACTIVE=0.16 and OCCUPANCY=0.09 say the SMs are
almost empty. TENSOR_ACTIVE=0.03 confirms this is GEMV, not GEMM — the tensor cores you
pay a premium for are idle. DRAM_ACTIVE=0.71 says the *actual* bottleneck is HBM
bandwidth — textbook memory-bound decode (module 03 roofline). FB_USED=58 GB says the
GPU is memory-pinned by KV-cache, so it *cannot* simply be reclaimed.

**Money conclusion.** The fleet is not compute-saturated; it is memory-bound at batch 1.
The correct action is **not** to buy 8 more H100s. It is to raise batch size / enable
continuous batching so TENSOR_ACTIVE climbs and each GPU serves more tokens — or to
right-size onto smaller/MIG partitions if latency SLOs allow. At ~$3/GPU-hr, avoiding
the 8-GPU expansion by fixing batching is ~$210k/year. That sentence is the interview
answer, and GPU_UTIL would have hidden it completely.

## Practice

On a rented GPU (any H100/A100/L4 from a neocloud or Lambda/RunPod) with DCGM +
dcgm-exporter + Prometheus + Grafana already stood up (module 04):

1. **Generate the lie.** Run a batch-1 autoregressive decode loop — the cleanest is a small
   HF `model.generate(..., num_beams=1)` in a `while True` on a 7B model, or if you want it
   dependency-free, a tiny custom kernel that does one small `cudaMemcpy`/GEMV per
   iteration in a tight loop. The point is a continuous stream of tiny kernels.
2. **Scrape the fields side by side.** Query `DCGM_FI_DEV_GPU_UTIL`,
   `DCGM_FI_PROF_SM_ACTIVE`, `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`, `DCGM_FI_PROF_DRAM_ACTIVE`,
   `DCGM_FI_DEV_FB_USED` in Prometheus for the same GPU/time range. (If a PROF field
   returns 0/absent, that's the multiplexing/enablement issue — lesson 05.2.)
3. **Build the exhibit.** One Grafana panel (or `dcgmi dmon -e 203,1002,1004,1005`) showing
   `GPU_UTIL ≈ 100` next to `SM_ACTIVE`/`TENSOR_ACTIVE` near zero, from *your own* cluster.
   **Screenshot it.**

**Acceptance:** a side-by-side capture from your own cluster showing
`DCGM_FI_DEV_GPU_UTIL ≈ 100` beside `SM_ACTIVE` and `PIPE_TENSOR_ACTIVE` near zero. This
screenshot is the flagship exhibit of the "Your GPU dashboard is lying to you"
deliverable.

## Self-check

**(a) Why does a batch-1 LLM decode read 100% GPU_UTIL?**
**Answer:** GPU_UTIL (field 203) is the fraction of the window with ≥1 kernel resident.
Batch-1 decode emits an unbroken stream of tiny GEMV/attention kernels — there is always
exactly one kernel resident — so the predicate is true ~100% of the time, even though
each kernel uses a few SMs briefly and spends most of its time stalled on HBM. It
measures presence, not work.

**(b) `SM_ACTIVE = 0.9` with `TENSOR_ACTIVE = 0.1` — what is the workload doing wrong?**
**Answer:** The SMs are broadly busy (90% of cycles have a resident warp) but the tensor
pipe is nearly idle (10%). The work is running on the wrong units — CUDA-core / non-HMMA
math, or a GEMM shape/precision that never dispatches to tensor cores. On a tensor-heavy
job (training/prefill) that is low MFU: you're paying for tensor silicon and running
scalar/FP32 work. Fix = precision (FP16/BF16/FP8), tensor-core-eligible kernels, or
better GEMM tiling — not more GPUs.

**(c) Which metric do you page on for "idle but allocated," and why not GPU_UTIL?**
**Answer:** Page on `DCGM_FI_PROF_SM_ACTIVE` (gated by `FB_USED` so you don't reclaim a
KV-cache-loaded replica). SM_ACTIVE is the honest breadth-of-work signal; a low value on
an allocated GPU means wasted spend. GPU_UTIL cannot be used because it reads ~100 on
the exact idle-but-resident workloads (batch-1 decode, tiny-kernel loops) you most need
to catch — a reclaim rule on field 203 never fires on your worst waste.

## Resources

1. **Superorbital — "GPU Underutilization in Kubernetes"** —
   https://superorbital.io/blog/gpu-kubernetes-underutilization/ — *deep.* The best single
   write-up of exactly this GPU_UTIL-vs-PROF gap in a Kubernetes/DCGM context, with the
   same field IDs and the reclaim argument; read it end-to-end as the narrative spine of
   your deliverable.
2. **DCGM field-identifiers reference** —
   https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html —
   *deep.* The canonical, exact semantics of every `DCGM_FI_*` field ID (203, 204,
   1001–1012, 250–253). This is the source of truth for the one-sentence definitions above;
   verify the wording here, not from memory.
