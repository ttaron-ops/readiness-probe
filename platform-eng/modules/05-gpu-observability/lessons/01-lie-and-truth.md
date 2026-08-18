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
sources: 7
---

# 05.1 · The lie and the truth: GPU_UTIL vs the PROF metrics

> **Concept.** `DCGM_FI_DEV_GPU_UTIL` measures whether a kernel was *resident*, not whether the silicon was *working* — the four PROF metrics measure the work, and each drives a different money decision.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

Module 04 closed with GPU Operator running your device plugin, dcgm-exporter deployed on
every node, and the pod-resources join wired up so a Prometheus series can be traced back
to a namespace and a pod. You can *get metrics out of a GPU cluster*. What module 04 never
asked is whether the flagship metric — the one every default dashboard leads with — means
what everyone assumes it means. This lesson answers that, in exact DCGM field-ID terms,
and the answer is the spine the rest of module 05 hangs off: the alerting policy in
lesson 05.5, the cardinality trade-offs in 05.3, and the capstone's entire dollar argument
in 05.8 all assume you know precisely what `GPU_UTIL` can and cannot tell you.

## Why this matters

"Our fleet dashboard shows 95% GPU utilization, but the training team says throughput is
bad — reconcile that" is a live interview question at NVIDIA (*Senior AI & HPC
Observability Engineer*, *Senior Platform Telemetry Engineer*), CoreWeave (*Sr SWE,
Observability* — a neocloud whose product *is* GPU fleet transparency), and Datadog,
which shipped a first-class DCGM GPU-monitoring product in 2025 built around exactly this
distinction: its architecture separates device-level metrics (`gpu.memory.free`,
`gpu.temperature`), process-level attribution (`gpu.process.sm_active`), and
Hopper-generation pipeline metrics (`gpu.tensor_active`, `gpu.fp16_active`,
`gpu.sm_occupancy`) from error counters (`gpu.errors.xid.total`) — a paid vendor's entire
taxonomy *is* this lesson's field list. Get the reconciliation wrong in a live interview
and you've told the panel you'd approve a GPU-expansion budget based on a metric that
cannot see the fleet's worst waste. Get it wrong on the job and the cost is real: every
dollar figure your cost operator (module 11) emits sits downstream of choosing the right
numerator here.

## What's new here (calibration)

- You already have the *concept* from module 03 — GPU-util vs SM-occupancy vs
  tensor-active vs MFU, and the roofline model explaining why decode is memory-bound.
  **Not re-taught.**
- You already deployed dcgm-exporter and built the pod-resources join in module 04, and
  you run Prometheus/PromQL/Grafana/alerting fluently across 40+ clusters. **Not
  re-taught.**
- New here: the *exact* DCGM wiring — field IDs, the one-sentence NVML/profiling-module
  semantics that survive an interview, which subsystem each field comes from, real H100
  numbers, and the precise alerting policy (page on this field, never that one).

## Core concepts

### The lie: `DCGM_FI_DEV_GPU_UTIL` (field ID 203)

State it in one sentence, without the word "utilisation":

> **`DCGM_FI_DEV_GPU_UTIL` is the fraction of the sample window during which at least one kernel was resident on the GPU.**

That is the literal NVML definition. Field 203 is a straight passthrough of
`nvmlDeviceGetUtilizationRates().gpu`, and NVML's own reference documents the `gpu` field
of the returned `nvmlUtilization_t` struct as "percent of time over the past sample
period during which one or more kernels was executing on the GPU" — with the sample
period itself defined as **between 1 second and 1/6 second, depending on the product
being queried**. Every load-bearing word matters:

- **"one or more kernels"** — a *count* threshold of ≥1, not an intensity. One kernel and
  ten thousand kernels both read 100.
- **"executing" / resident** — scheduled on the GPU, not necessarily computing. A kernel
  stalled on an HBM load is still "executing."
- **"past sample period"** — a short, driver-internal window (sub-second), re-sampled by
  DCGM at your own update frequency (lesson 05.2 covers that layer). It is a duty cycle
  of *presence*, integrated over that window — not an average of how much of the chip was
  doing work.

There is no notion of *how many* SMs, *how full* they are, or *which* pipes ran. It is a
binary "was anything on the GPU?" averaged to a percentage. That binary is why it lies.

`DCGM_FI_DEV_MEM_COPY_UTIL` (field 204) is its sibling from the same NVML call —
"percent of time the memory *controller* was read or written" — and it lies the same
way: it is a duty cycle of the copy engine, not achieved HBM bandwidth. Do not confuse it
with `DCGM_FI_PROF_DRAM_ACTIVE` (1005); one is presence, the other is a real activity
ratio from a completely different subsystem (lesson 05.2 explains why NVML fields and
PROF fields come from different collection paths entirely).

### Why batch-1 autoregressive decode pins GPU_UTIL to 100%

Autoregressive LLM decode generates **one token per forward pass**. At batch size 1, each
forward pass is a sequence of *tiny* kernels — a GEMV against each weight matrix (matrix
× vector, not matrix × matrix), plus attention over the KV-cache. Each kernel:

- launches, reads weights from HBM, does a sliver of math, writes back, exits;
- is immediately followed by the next kernel in the layer stack;
- the whole token takes, say, 10–20 ms, and across that entire time **there is always
  exactly one kernel resident** on the GPU.

So the "≥1 kernel resident" predicate is true for ~100% of every sample window →
**`GPU_UTIL` reads 100**. Meanwhile a GEMV uses a handful of SMs for a few microseconds
each and spends most of the kernel's wall-clock *waiting on HBM* — the SMs are
idle-but-occupied. The math is memory-bound (module 03's roofline: arithmetic intensity of
a GEMV is ~1–2 FLOP/byte, far left of the ridge), so the tensor cores essentially never
fire.

**Real case (H100, Llama-class 7B, batch 1, FP16 decode):**

| Field | Value | Reading |
|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` (203) | **100** | a kernel is always resident |
| `DCGM_FI_PROF_SM_ACTIVE` (1002) | **0.18** | only ~18% of SM-cycles had a warp |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (1004) | **~0.02** | tensor cores idle — it's GEMV, not GEMM |
| `DCGM_FI_PROF_DRAM_ACTIVE` (1005) | **0.6–0.8** | HBM is the bottleneck |

The dashboard says 100% busy. The truth is you are paying for a full H100 to run at ~18%
breadth and ~2% tensor throughput, memory-bound. That gap — 100 vs 0.18 — is the flagship
exhibit of the whole module.

**Calibration point for training jobs:** even a *well-run* training run doesn't hit
anywhere near 100% of peak FLOPs — Meta reported Llama 3.1 pretraining ran at roughly
38–43% model-FLOPs-utilization (MFU), the industry-standard efficiency metric, on its
16,384-H100 cluster. If a carefully-tuned multi-week training job tops out under half of
theoretical peak, a decode workload sitting at 2% tensor-active isn't a rounding error —
it's a different order of problem, and `GPU_UTIL` cannot distinguish the two: both a
43%-MFU training step and a 2%-tensor decode step can read `GPU_UTIL = 100`.

### The truth: the four PROF metrics + framebuffer

These come from the **profiling subsystem** (field IDs 1001–1012), sampled from hardware
performance counters — a fundamentally different, more expensive path than NVML's
duty-cycle polling (that cost, and how it's multiplexed across counters, is the whole of
lesson 05.2). They are **ratios in [0.0, 1.0]**, not percentages. The four that matter,
and the decision each drives:

1. **`DCGM_FI_PROF_SM_ACTIVE` (1002) — breadth.**
   *The fraction of elapsed cycles during which at least one warp was resident on an SM,
   averaged over all SMs.* This is the honest answer to "is the GPU doing anything?" Low
   SM_ACTIVE with a GPU allocated = **idle but paid for**. **Decision: cost / reclaim.**

2. **`DCGM_FI_PROF_SM_OCCUPANCY` (1003) — depth.**
   *The fraction of warp *slots* filled — resident warps ÷ the SM's maximum warp
   capacity, averaged over time and over all SMs.* SM_ACTIVE says "an SM had ≥1 warp";
   OCCUPANCY says "how full was it." High breadth + low depth = a launch/occupancy
   problem (small grids, register/shared-memory pressure). **Decision: kernel/config
   tuning, batch size.**

3. **`DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (1004) — tensor throughput.**
   *The fraction of cycles the tensor (HMMA) pipe was active.* This is the closest single
   field to "are we getting the FLOPs we paid for" for training and prefill — it's the
   numerator of a rough MFU estimate. Low here on a training job means you are
   compute-idle on the most expensive units on the die. **Decision: right-sizing,
   precision (module 03.5), kernel choice.**

4. **`DCGM_FI_PROF_DRAM_ACTIVE` (1005) — memory-bandwidth duty cycle.**
   *The fraction of cycles the device-memory (HBM) interface was actively sending or
   receiving.* Decode is memory-bound, so *high* DRAM_ACTIVE with *low* TENSOR_ACTIVE is
   the signature of healthy-but-memory-bound inference — not a defect, but the signal
   that batching or a bigger model is the only way to raise tensor throughput.
   **Decision: batch/throughput tuning, workload placement.**

Plus one residency field:

- **`DCGM_FI_DEV_FB_USED` (252) — framebuffer bytes in use** (with `FB_FREE` 251,
  `FB_TOTAL` 250, `FB_RESERVED` 253). For inference this is **KV-cache residency**: a
  decode server can be idle on compute (SM_ACTIVE ~0) yet hold 60 GB of KV-cache, meaning
  the GPU is *memory-pinned* and cannot be reclaimed even though it looks idle. FB_USED is
  what stops you from wrongly reclaiming a paused-but-loaded serving replica. **Decision:
  reclaim safety, memory right-sizing.**

Related, sometimes useful: `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (1001, graphics/compute engine
active — a coarser "was the engine busy"), and the precision-specific pipes
`PIPE_FP16_ACTIVE` (1008), `PIPE_FP32_ACTIVE` (1007), `PIPE_FP64_ACTIVE` (1006) when you
need to know *which* math units ran.

### The alerting policy (the load-bearing rule)

You already run alertmanager at scale; here is the GPU-specific policy. The mental model:
**breadth answers "is it idle?" (a cost question); tensor/DRAM answer "is it efficient?"
(a right-sizing question); GPU_UTIL answers nothing you can act on.**

- **"Idle but allocated" → page/reclaim on `SM_ACTIVE`.** A GPU is `Allocated`
  (pod-resources join from module 04) but `SM_ACTIVE` sits below threshold for a
  sustained window → wasted spend. Gate with `FB_USED` so you don't reclaim a
  loaded-but-paused replica.

  ```promql
  # GPUs allocated to a pod but doing ~no SM work for 30m
  (
    DCGM_FI_PROF_SM_ACTIVE < 0.05
    and on (gpu, UUID) (DCGM_FI_DEV_FB_USED < 2048)   # not holding a KV-cache
  )
  # join to pod/namespace via the kube_pod_* / dcgm exporter labels from module 04
  ```

- **"Efficient?" → dashboard/warn on `PIPE_TENSOR_ACTIVE` and `DRAM_ACTIVE`.** A running
  training job with `TENSOR_ACTIVE` far below its roofline expectation is a
  right-sizing/tuning signal, not a page. Pair the two: high DRAM + low TENSOR =
  memory-bound (expected for decode, a batching lever for training); low DRAM + low
  TENSOR = genuinely stalled (input pipeline, sync, small kernels).

- **NEVER alert on `DCGM_FI_DEV_GPU_UTIL`.** It cannot distinguish a 100%-tensor GEMM
  from a 2%-tensor GEMV; both read 100. An SLO or reclaim rule built on field 203 will
  *never fire* on the most expensive failure mode you have — a fleet of decode servers
  pinned at 100% GPU_UTIL and 18% SM_ACTIVE. Put it on a dashboard only as the *foil*,
  next to SM_ACTIVE, to make the lie visible.

## Perspectives

**Developer.** From inside a training or inference process you never see `GPU_UTIL` at
all — you see wall-clock step time or tokens/second. The metric lies specifically to the
*platform* layer sitting above the process. A developer's local `nvidia-smi` glance
("99%, we're fine") is the seed of this entire module's thesis: the number that looks
reassuring from inside a `kubectl exec` is the same number that's actively hiding the
waste from the platform team's dashboard.

**Operator.** `GPU_UTIL` is what nearly every default dashboard, capacity-planning
spreadsheet, and "are we out of GPUs" Slack thread is built on. The operator's job is not
to find a better dashboard widget — it's to know, structurally, that this specific field
is incapable of answering the question it's constantly asked to answer, and to have a
field ready that can.

**Hardware.** `GPU_UTIL` is a straight passthrough of a single NVML duty-cycle counter
with no notion of SM count, warp occupancy, or which pipe (tensor vs CUDA-core vs copy
engine) was busy. The PROF fields come from an entirely different subsystem — hardware
performance counters programmed by DCGM's profiling module — which is *why* they can see
occupancy and pipe activity that NVML physically cannot: NVML was never wired to those
counters in the first place.

**Economics.** The entire allocated-vs-utilised dollar argument (the module's capstone)
is downstream of this one gap. A metric that reads 99% "busy" while the fleet is
15–20% SM-active is not a rounding error — at $2–3/GPU-hour on hundreds of GPUs it is a
six-to-seven-figure-per-year blind spot, and it is a blind spot specifically because the
metric everyone trusts is structurally unable to see it.

## Real-world use cases

- **Datadog — GPU Monitoring Reference Architecture** —
  https://www.datadoghq.com/architecture/gpu-monitoring/. Datadog's production metric
  taxonomy separates device-level fields (`gpu.memory.free`, `gpu.temperature`) from
  process-level attribution (`gpu.process.sm_active`) and Hopper-generation pipeline
  metrics (`gpu.tensor_active`, `gpu.fp16_active`, `gpu.sm_occupancy`), plus
  `gpu.errors.xid.total` for health. **What it shows:** a paid observability vendor's
  entire DCGM product is organized around exactly the GPU_UTIL-vs-SM_ACTIVE distinction
  this lesson teaches — confirming the module's framing that "Datadog's taxonomy *is*
  this syllabus."
- **acecloud.ai — "GPU Utilization In Production: Are Your GPUs Efficient?"** —
  https://acecloud.ai/blog/gpu-utilization-production/ (URL confirmed live via search;
  direct fetch blocked by this sandbox's egress proxy — the specific before/after figures
  originally cited here could not be independently corroborated on re-verification, so
  they've been removed rather than risk an invented quote). Describes the general
  diagnostic pattern this lesson teaches: a sawtooth GPU-activity trace from slow
  dataloaders/host transfers, and high `DRAM_ACTIVE` with low tensor activity as the
  signal that data movement, not compute, is the bottleneck. **What it shows:** an
  independent practitioner source describing the same DRAM-vs-tensor diagnostic this
  lesson's PROF-field table teaches, in a production-fleet-efficiency context.
- **Superorbital — "GPU Underutilization in Kubernetes"** —
  https://superorbital.io/blog/gpu-kubernetes-underutilization/. Walks the same
  GPU_UTIL-vs-PROF-field gap in a Kubernetes/DCGM context, with matching field IDs and
  the reclaim argument. **What it shows:** the same thesis, argued end-to-end from a
  platform-engineering (not vendor-marketing) vantage point — read it as the narrative
  spine of your deliverable.
- **Grafana Labs — "NVIDIA DCGM Exporter Dashboard"** (community dashboard) —
  https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/. A widely
  installed community dashboard built primarily on `nvidia-smi`-style utilization fields.
  **What it shows:** the "here's the lie everyone ships" exhibit — the default dashboard
  people copy-paste into their own clusters carries the same blind spot as the stock
  exporter config, which is exactly why lesson 05.3 exists.

## Worked example

**Static snapshot.** A serving namespace holds 8× H100. The Grafana tile (built on
GPU_UTIL) is solid green at **99%**; the platform team is about to approve a request for
8 *more*. You pull the PROF fields for one representative GPU over a 10-minute window:

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
bandwidth — textbook memory-bound decode (module 03 roofline). FB_USED=58 GB says the GPU
is memory-pinned by KV-cache, so it *cannot* simply be reclaimed.

**Dynamic follow-through — does fixing it actually move the needle?** The team enables
continuous batching instead of buying more GPUs. You re-pull the same fields across the
rollout:

| Phase | GPU_UTIL | SM_ACTIVE | TENSOR_ACTIVE | Throughput (rel.) |
|---|---|---|---|---|
| Before (batch 1) | 99 | 0.16 | 0.03 | 1.0× |
| Mid-rollout | 99 | 0.34 | 0.11 | ~1.8× |
| After (continuous batching) | **99** | **0.55** | **0.19** | **~2.9×** |

`GPU_UTIL` never moves — it is pinned at 99 through the *entire* fix, because a kernel is
resident throughout, before and after. `SM_ACTIVE` and `TENSOR_ACTIVE` climb steadily and
track the real throughput gain almost 1:1. This is the strongest version of the lesson:
`GPU_UTIL` is not merely uninformative about the *problem*, it is equally uninformative
about the *fix* — a dashboard built on field 203 would show zero change while the fleet
nearly tripled its useful work.

**Money conclusion.** The fleet was never compute-saturated; it was memory-bound at batch
1. The correct action was not to buy 8 more H100s — it was to fix batching so
TENSOR_ACTIVE climbed and each GPU served more tokens. At ~$3/GPU-hour, avoiding the
8-GPU expansion is roughly **$210k/year** (2026 on-demand H100 pricing; treat as a
directional, dated snapshot, not a fixed constant). That sentence is the interview
answer, and GPU_UTIL would have hidden it — before, during, and after the fix.

## Practice

On a rented GPU (any H100/A100/L4 from a neocloud or Lambda/RunPod) with DCGM +
dcgm-exporter + Prometheus + Grafana already stood up (module 04), building directly
toward the module's ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)
deliverable:

1. **Generate the lie.** Run a batch-1 autoregressive decode loop — the cleanest is a
   small HF `model.generate(..., num_beams=1)` in a `while True` on a 7B model, or a tiny
   custom kernel doing one small GEMV per iteration in a tight loop. The point is a
   continuous stream of tiny kernels.
2. **Scrape the fields side by side.** Query `DCGM_FI_DEV_GPU_UTIL`,
   `DCGM_FI_PROF_SM_ACTIVE`, `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`, `DCGM_FI_PROF_DRAM_ACTIVE`,
   `DCGM_FI_DEV_FB_USED` in Prometheus for the same GPU/time range. (If a PROF field
   returns 0/absent, that's the multiplexing/enablement issue covered next in lesson
   05.2.)
3. **Reproduce the dynamic exhibit.** If you have a batching-capable server (vLLM,
   TGI), turn on continuous/dynamic batching mid-run and capture the same fields moving
   the way the worked example describes — GPU_UTIL flat, SM_ACTIVE/TENSOR_ACTIVE
   climbing.
4. **Build the exhibit.** One Grafana panel (or `dcgmi dmon -e 203,1002,1004,1005`)
   showing `GPU_UTIL ≈ 100` next to `SM_ACTIVE`/`TENSOR_ACTIVE` near zero, from *your own*
   cluster. **Screenshot it.**

**Acceptance:** a side-by-side capture from your own cluster showing
`DCGM_FI_DEV_GPU_UTIL ≈ 100` beside `SM_ACTIVE` and `PIPE_TENSOR_ACTIVE` near zero. This
screenshot is the flagship exhibit of the "Your GPU dashboard is lying to you"
deliverable.

## Common pitfalls

1. **Treating `GPU_UTIL` and `MEM_COPY_UTIL` as if one measures compute and the other
   measures "the rest."** Both are duty-cycle/presence metrics from the same NVML call;
   neither measures throughput. Reach for `PIPE_TENSOR_ACTIVE` and `DRAM_ACTIVE` for
   actual work, not field 204.
2. **Assuming a high `GPU_UTIL` number *validates* a capacity-expansion request.** It is
   the number most likely to be high precisely when the fleet is *most* wasted (batch-1
   decode) — the worked example's dynamic table shows it stays high even after the waste
   is fixed.
3. **Conflating `SM_ACTIVE` (breadth: is any warp resident) with `PIPE_TENSOR_ACTIVE`
   (are the *expensive* units firing).** A workload can be "SM-busy but tensor-idle," and
   both facts matter for different decisions — SM_ACTIVE for idle/reclaim, TENSOR_ACTIVE
   for right-sizing/precision.
4. **Forgetting the FB_USED gate before reclaiming.** A GPU with low SM_ACTIVE can still
   be holding a large resident KV-cache; reclaiming it kills a paused-but-loaded serving
   replica, not an idle one.
5. **Reading the NVML sample-period as fixed.** It's product-dependent (roughly 1s down
   to 1/6s), not a constant — don't assume you know the exact averaging window without
   checking the field-ID reference for your GPU generation.

## Self-check

- Why does a batch-1 LLM decode read 100% GPU_UTIL? **Answer:** GPU_UTIL (field 203) is
  the fraction of the window with ≥1 kernel resident. Batch-1 decode emits an unbroken
  stream of tiny GEMV/attention kernels — there is always exactly one kernel resident —
  so the predicate is true ~100% of the time, even though each kernel uses a few SMs
  briefly and spends most of its time stalled on HBM. It measures presence, not work.
- `SM_ACTIVE = 0.9` with `TENSOR_ACTIVE = 0.1` — what is the workload doing wrong?
  **Answer:** The SMs are broadly busy (90% of cycles have a resident warp) but the
  tensor pipe is nearly idle (10%). The work is running on the wrong units — CUDA-core /
  non-HMMA math, or a GEMM shape/precision that never dispatches to tensor cores. On a
  tensor-heavy job (training/prefill) that's low MFU: you're paying for tensor silicon and
  running scalar/FP32 work. Fix = precision (FP16/BF16/FP8), tensor-core-eligible
  kernels, or better GEMM tiling — not more GPUs.
- Which metric do you page on for "idle but allocated," and why not GPU_UTIL?
  **Answer:** Page on `DCGM_FI_PROF_SM_ACTIVE` (gated by `FB_USED` so you don't reclaim a
  KV-cache-loaded replica). SM_ACTIVE is the honest breadth-of-work signal; a low value on
  an allocated GPU means wasted spend. GPU_UTIL cannot be used because it reads ~100 on
  the exact idle-but-resident workloads (batch-1 decode, tiny-kernel loops) you most need
  to catch — a reclaim rule on field 203 never fires on your worst waste.
- Why is `DCGM_FI_DEV_MEM_COPY_UTIL` (204) not a substitute for `DCGM_FI_PROF_DRAM_ACTIVE`
  (1005)? **Answer:** MEM_COPY_UTIL is an NVML duty-cycle counter for the copy engine —
  was the memory controller touched at all during the sample window — not a measure of
  achieved bandwidth. DRAM_ACTIVE is a hardware-counter-backed ratio of cycles the HBM
  interface was actually sending/receiving data. A workload can show high MEM_COPY_UTIL
  while barely saturating real bandwidth, the same "presence vs work" gap as GPU_UTIL vs
  SM_ACTIVE.
- What decision does each of the four PROF metrics drive? **Answer:** SM_ACTIVE →
  cost/reclaim (is it idle). SM_OCCUPANCY → kernel/config tuning (is the launch shape
  starving the SM). PIPE_TENSOR_ACTIVE → right-sizing/precision (are we getting the FLOPs
  we're paying for). DRAM_ACTIVE → batching/throughput tuning (is HBM bandwidth the
  ceiling). Each answers a different question and should not be collapsed into one
  "efficiency score."

## Connections & what's next

This lesson gives you the field IDs and semantics; it deliberately does not explain *how*
DCGM collects the PROF fields or what that collection costs — that's lesson 05.2,
immediately next, which covers the profiling subsystem's hardware-counter multiplexing
and the scrape-interval trade-offs that determine whether the fields you just learned
about actually show real data or silent zeros. Downstream, lesson 05.3 shows that
`SM_ACTIVE` ships **commented out** in dcgm-exporter's default config — the metric this
lesson tells you to page on isn't even collected until you turn it on. Lesson 05.5 reuses
the same field-ID discipline for XID health codes, and the capstone (05.8) turns the
GPU_UTIL/SM_ACTIVE divergence you built here into a per-namespace dollar figure.

## References & further reading

**Primary sources**
- DCGM field identifiers reference — https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html — read for the exact, canonical semantics of every `DCGM_FI_*` field ID (203, 204, 1001–1012, 250–253) used in this lesson; verify wording here, not from memory.
- NVML `nvmlUtilization_t` reference (archived API docs) — https://docs.nvidia.com/deploy/archive/R535/nvml-api/structnvmlUtilization__t.html — read for the source definition of the `gpu`/`memory` fields `DCGM_FI_DEV_GPU_UTIL`/`MEM_COPY_UTIL` pass through, including the sample-period range (1s–1/6s).

**Real-world engineering blogs**
- Datadog — GPU Monitoring Reference Architecture — https://www.datadoghq.com/architecture/gpu-monitoring/ — what it shows: a paid vendor's DCGM product taxonomy mirrors this lesson's field split exactly.
- acecloud.ai — "GPU Utilization In Production: Are Your GPUs Efficient?" — https://acecloud.ai/blog/gpu-utilization-production/ — what it shows: the sawtooth-utilization and DRAM-vs-tensor-activity diagnostic pattern this lesson's PROF-field table teaches, from an independent practitioner source.
- Superorbital — "GPU Underutilization in Kubernetes" — https://superorbital.io/blog/gpu-kubernetes-underutilization/ — what it shows: the same field-ID gap argued end-to-end in a Kubernetes/DCGM context.
- Grafana Labs — "NVIDIA DCGM Exporter Dashboard" — https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/ — what it shows: the widely-installed community dashboard that ships with the same GPU_UTIL-centric blind spot.

**Deeper dives**
- Meta — "The Llama 3 Herd of Models" (arXiv:2407.21783), §3.3.2 — https://arxiv.org/abs/2407.21783 — go deeper on: real training MFU (38–43%) as a calibration point for how far even a well-run job sits below theoretical peak, versus a 2%-tensor-active decode workload.
