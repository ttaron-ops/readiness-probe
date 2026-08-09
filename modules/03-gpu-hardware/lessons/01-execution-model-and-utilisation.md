---
lesson: "03.1"
title: "Execution model and the utilisation lie"
module: "03"
concept: "Execution model and the utilisation lie"
status: not-started
est_time: "4h"
artifacts: []
---

# 03.1 · Execution model and the utilisation lie

> **Concept.** Just enough of the SM/warp/kernel execution model to say precisely what `nvidia-smi` "GPU-Util" measures — and to prove it is not throughput.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Why this matters

The single most expensive mistake in GPU fleet operations is trusting `nvidia-smi` GPU-Util as a measure of work done. A GPU can report **100% utilisation while doing less than 1% of the work it is capable of** — and you are paying the same $2–4/hr either way. If you cannot articulate, in an interview or a capacity review, exactly what that number counts and why it lies, you cannot build the cost/observability story that differentiates a senior platform engineer at a GPU-heavy shop. This lesson is the module's thesis: **utilisation is not throughput, and throughput is what you pay for.**

## The platform-engineer's lens

**Extract this:** a crisp mental model of *when a GPU counts as "busy"* at three different granularities — temporal (a kernel is running), spatial (how many SMs / warps are resident), and functional (are the tensor cores actually the thing running). You need this only so you can pick the *right metric to alert and bill on*, and so you can look at a `nvidia-smi` screenshot and a DCGM dashboard and immediately say "busy but idle."

**Do NOT master:** how to *write* efficient kernels — occupancy tuning, warp divergence, shared-memory bank conflicts, launch-bound tuning. That is CUDA-kernel-developer depth. You are here to *read* the meters and reason about cost per unit of work, not to move the needle by rewriting kernels. Learn the vocabulary (SM, warp, block, kernel, occupancy) to the depth of "I can interpret a profiler field," and stop there.

## Core notes

### The execution model, in four words you need

- **Kernel** — a function launched to run on the GPU. One `torch.matmul` may dispatch one or several kernels. GPU-Util cares only whether *any* kernel is currently executing.
- **Thread block (CTA)** — the unit the scheduler places onto hardware. A block is assigned to exactly **one** SM and stays there until it finishes. This is the key fact: a launch of *one block* occupies *one SM* and leaves the rest idle.
- **SM (Streaming Multiprocessor)** — the independent compute unit. An **H100 SXM5 has 132 SMs**. Real throughput requires most of them busy, most of the time.
- **Warp** — 32 threads executed in lockstep; the actual scheduling quantum inside an SM. Each H100 SM can hold up to 64 resident warps (2048 threads). "Occupancy" = resident warps ÷ max warps.

Mental picture: the GPU is a **factory with 132 machines** (SMs). Tensor cores are a *specialised station* inside each machine. GPU-Util only reports "is the factory's power switch on and at least one worker moving." It says nothing about how many of the 132 machines are running, nor whether the specialised (tensor) stations are being used at all.

### What `nvidia-smi` GPU-Util actually measures

NVIDIA's own NVML definition:

> **GPU-Util** = "Percent of time over the past sample period during which one or more kernels was executing on the GPU."

Decode every word:

- **"percent of time"** → it is a *temporal duty cycle*, not an amount of work.
- **"one or more kernels"** → *one* is enough. One block on one of 132 SMs, tensor cores stone-cold idle, still counts.
- **"sample period"** → it is coarse (order of a second); a stream of tiny kernels with no gaps pins it to 100%.

So GPU-Util answers "was the GPU non-idle?" It does **not** answer "how many SMs?", "were the tensor cores fed?", or "how close to peak FLOP/s?". A single-threaded, single-block kernel that never stops → **100% GPU-Util, ~0.8% of the hardware engaged.** That is the lie.

### The metrics that actually tell you about work (DCGM)

DCGM (NVIDIA Data Center GPU Manager) exposes profiling fields that measure the granularities GPU-Util hides. These are the fields you build fleet observability and chargeback on:

| Metric (DCGM field) | What it measures | The question it answers |
|---|---|---|
| `GPU-Util` (NVML) | Temporal: any kernel running | Is it switched on? |
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | Fraction of time the graphics/compute engine was active | Slightly better "on" signal |
| `DCGM_FI_PROF_SM_ACTIVE` | Fraction of time ≥1 warp was resident on an SM, **averaged over all SMs** | How much of the machine floor is running? |
| `DCGM_FI_PROF_SM_OCCUPANCY` | Resident warps ÷ max warps, averaged over SMs | How densely packed are the running SMs? |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | Fraction of cycles the tensor (HMMA) pipe was active | Are the tensor cores — the thing you bought — doing anything? |
| `DCGM_FI_PROF_DRAM_ACTIVE` | Fraction of cycles HBM was reading/writing | Are you memory-bound? (feeds lesson 03.2) |

The one that kills the GPU-Util lie is **SM_ACTIVE**. Because it is *averaged over all SMs*, a one-block kernel on H100 gives roughly `1 / 132 ≈ 0.008` → **0.8%**, while GPU-Util reads 100%. That gap *is* the busy-but-idle signature.

Two subtleties to keep straight:

- **SM_ACTIVE is ambiguous on its own.** A value of 0.5 can mean "all 132 SMs busy half the time" *or* "half the SMs busy all the time." That is fine for our purpose — either way it caps the useful-work ceiling, and it is still infinitely more honest than GPU-Util. Pair it with SM_OCCUPANCY and TENSOR_ACTIVE to disambiguate.
- **TENSOR_ACTIVE is the money metric for AI.** You are paying an H100 premium *for the tensor cores*. FP64/FP32 CUDA-core work, or memory-bound elementwise ops, can pin SM_ACTIVE high while TENSOR_ACTIVE sits near zero — meaning the expensive silicon is idle even though the "GPU is busy."

### The other way GPU-Util misleads: overhead-bound workloads

The single-block kernel is the *high-util, no-work* failure. The mirror image is just as common in real fleets: **overhead-bound** workloads where the GPU is genuinely starved by everything *around* the kernels.

- **Launch-bound / tiny-kernel storms.** Thousands of microsecond kernels back-to-back (unfused elementwise ops, an un-`torch.compile`'d model) keep GPU-Util pinned near 100% because there is always *a* kernel running — yet each kernel finishes before the SMs fill, so SM_OCCUPANCY and TENSOR_ACTIVE stay low. Same lie, different cause: the fix is **fusion**, not a bigger GPU.
- **Host/input starvation.** A slow data loader, CPU-side preprocessing, or Python `GIL`-bound dispatch leaves *gaps* between kernels. Here GPU-Util may read a *deceptively low but non-zero* 40–70% — and the naive reaction ("GPU is only 60% used, pack more on it") is wrong, because the ceiling is the CPU/IO pipeline, not the GPU. The tell is GPU-Util that rises when you increase `num_workers` or prefetch depth.

Both cases reinforce the rule: **GPU-Util is a symptom, never a diagnosis.** You need SM_ACTIVE + TENSOR_ACTIVE + DRAM_ACTIVE together to tell "starved of parallelism" from "starved of data" from "actually working."

### The fleet recipe: what to alert and bill on

For a 40-cluster fleet, the alert that catches waste is the *conjunction*, expressed against `dcgm-exporter` Prometheus series:

```promql
# "Busy but idle": switched on, floor empty, tensor cores cold — for 15 min
  (DCGM_FI_PROF_GR_ENGINE_ACTIVE  > 0.9)
and (DCGM_FI_PROF_SM_ACTIVE          < 0.2)
and (DCGM_FI_PROF_PIPE_TENSOR_ACTIVE < 0.05)
```

- **Alert on:** the conjunction above (busy-but-idle) and, separately, sustained low SM_ACTIVE on expensive SKUs.
- **Bill / show-back on:** integrate `SM_ACTIVE` (or, for AI tenants, `TENSOR_ACTIVE`) over time to get *useful GPU-seconds*, then divide by *allocated GPU-seconds* → the allocated-vs-utilised gap that module 11 monetises. Never bill on GPU-Util; it would show every idle-but-on GPU as fully used and hide the entire waste story.
- **MIG caveat:** on a MIG-partitioned H100, per-instance profiling fields (SM_ACTIVE, TENSOR_ACTIVE) are what tell you whether each *slice* is doing work; whole-GPU GPU-Util is meaningless across partitions. Watch per-instance fields when you slice for multi-tenancy.

### MFU — the one number that ties it to money

**MFU (Model FLOPs Utilisation)** is the end-to-end honesty metric: the useful FLOP/s your model actually achieved divided by the GPU's theoretical peak FLOP/s.

```
MFU = (useful model FLOP/s achieved) / (peak FLOP/s of the hardware)
```

For LLM training the standard estimate of "useful model FLOPs" is `6 · N_params · tokens` (forward + backward, per the Chinchilla/PaLM convention). So:

```
MFU = (6 · N_params · tokens_per_sec) / (num_GPUs · peak_FLOP/s_per_GPU)
```

Using **H100 dense BF16 peak = 989 TFLOP/s** as the denominator (see the sparsity note below), **a good, achievable target is 40–50% MFU** for well-tuned large-scale training. Google's PaLM reported 46.2%; frontier training runs live in that band. Anything under ~30% is a cost problem worth investigating; 15% means you are burning roughly 3× the GPU-hours you should.

A cousin metric, **HFU (Hardware FLOPs Utilisation)**, counts *all* FLOPs the hardware executed including activation recomputation. MFU ≤ HFU always. For cost reasoning prefer **MFU**, because it measures FLOPs that advance the model — the thing you are actually buying — not FLOPs spent recomputing to save memory.

### Occupancy is not throughput either

A trap one level deeper, and a favourite interview follow-up: **high SM_OCCUPANCY does not guarantee high throughput.** Occupancy measures resident warps ÷ max warps — how *full* the SM's scheduler slots are. A memory-bound kernel can have 100% occupancy while every warp is *stalled waiting on HBM*; the SM is packed but the ALUs idle. Conversely a well-tuned GEMM can hit near-peak FLOP/s at only ~50% occupancy because it has enough in-flight work to hide latency. So the ladder of honesty is: GPU-Util (is it on?) → SM_ACTIVE (is the floor running?) → SM_OCCUPANCY (are the SMs packed?) → **TENSOR_ACTIVE / achieved FLOP/s (is it doing the work you paid for?)**. Only the last rung is throughput; the next lesson (03.2 roofline) is what turns it into a cost number.

### Collection gotchas (ops reality)

- **Profiling fields cost something.** DCGM `PROF_*` fields require the profiling API and add small overhead; historically some fields *serialised* with each other and could not all be sampled simultaneously on older DCGM/driver combos. Verify your dcgm-exporter config actually exports SM_ACTIVE and TENSOR_ACTIVE together before trusting a dashboard.
- **GPU-Util is free and always there; that is exactly why it is over-trusted.** It comes from NVML with zero setup, so it is the default in every quick-look tool — which is how the lie propagates.
- **Sample windows differ.** `nvidia-smi`'s duty cycle and DCGM's cycle-fraction fields are measured over different windows; do not expect them to reconcile arithmetically. Use each for its purpose, not to cross-check the other.

### The sparsity asterisk (a common interview trap)

NVIDIA's H100 datasheet advertises **1,979 BF16 TFLOPS** and **3,958 FP8 TFLOPS**. Those are the **structured-sparsity** figures (2× marketing multiplier that assumes a 2:4 sparse weight pattern most workloads do not use). The **dense** numbers — the honest denominator for MFU on ordinary dense training/inference — are **989 BF16 / 1,979 FP8 TFLOP/s**. H100 SXM also carries **80 GB HBM3 at 3.35 TB/s**. Quoting the sparse number as if it were your dense ceiling silently halves your apparent MFU target; know which one you are dividing by.

## Worked example

You rent one H100 SXM ($/hr, on-demand ≈ $3) and launch a deliberately trivial kernel — a single-threaded elementwise op on a tiny tensor, looped so it never stops:

```python
import torch, time
x = torch.ones(1024, device="cuda")          # tiny — one block's worth
while True:
    x = x + 1.0                                # trivial, no tensor cores
```

Side-by-side capture during the loop:

| Meter | Reading | Interpretation |
|---|---|---|
| `nvidia-smi` GPU-Util | **~100%** | A kernel is always executing → duty cycle pinned |
| `DCGM_FI_PROF_SM_ACTIVE` | **~0.01 (1%)** | ~1 of 132 SMs ever resident |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | **~0.00** | Tensor cores — the thing you rented — untouched |
| Implied MFU | **≪ 1%** | Effectively zero useful FLOP/s vs 989 TFLOP/s peak |

The cost reading: you are paying **$3/hr for a GPU that is 100% "utilised" and ~99% wasted.** Across a 40-cluster fleet, alerting on GPU-Util would show this node as perfectly healthy. Alerting on SM_ACTIVE / TENSOR_ACTIVE catches it. This single screenshot is the seed of the deliverable's "util lies" demo — it is the most persuasive artifact in the whole report because every reviewer has seen a GPU-Util dashboard and believed it.

## Practice

**Task (rent one H100 or A100 by the hour; budget ~1 hr ≈ $2–4).**

1. Install DCGM / `dcgm-exporter` (or use `dcgmi dmon`) alongside `nvidia-smi`.
2. Run a **busy-but-idle** workload: a single-block CUDA kernel, or the trivial PyTorch loop above, so GPU-Util pins near 100%.
3. Simultaneously capture, in one log with timestamps: `nvidia-smi` GPU-Util, `DCGM_FI_PROF_SM_ACTIVE`, `DCGM_FI_PROF_SM_OCCUPANCY`, and `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`.
4. Save a table / screenshot showing **~100% GPU-Util next to ~0% SM-active / tensor-active.**

**Acceptance:** a single artifact (log + one annotated screenshot or table) in which GPU-Util and SM/tensor-active are shown *side by side for the same instant*, with a one-sentence caption stating why the GPU is "busy but idle." This is Exhibit A of the GPU Efficiency & Cost Report.

## Self-check

**(a) Why can a 1-SM (single-block) kernel report 100% GPU-Util?**
**Answer:** GPU-Util is a *temporal* duty cycle — "percent of the sample window during which one or more kernels was executing." A single always-resident block means a kernel is *always* running, so the window is 100% covered. It counts *time non-idle*, not *SMs engaged* (1 of 132) and not *FLOP/s*, so it saturates while ~99% of the hardware sits idle.

**(b) Which metric would you fleet-alert on to catch a "busy but idle" GPU?**
**Answer:** `DCGM_FI_PROF_SM_ACTIVE` (averaged over all SMs) as the primary "is the floor actually running" signal, and `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` for AI workloads to confirm the tensor cores you paid for are engaged. Alert on *high GPU-Util combined with low SM_ACTIVE/TENSOR_ACTIVE* — that conjunction is the busy-but-idle fingerprint. Never alert on GPU-Util alone.

**(c) Define MFU and give a good target.**
**Answer:** MFU (Model FLOPs Utilisation) = useful model FLOP/s achieved ÷ theoretical peak FLOP/s of the hardware, using `6·N_params·tokens` as the useful-FLOP estimate for training. A good, achievable target is **40–50%** for well-tuned large-scale training (PaLM hit 46.2%). Divide by the *dense* peak (989 BF16 TFLOP/s on H100), not the sparse datasheet figure.

## Resources

1. **Modal — "I paid for the whole GPU, I am going to use the whole GPU" (GPU utilisation guide)** — https://modal.com/blog/gpu-utilization-guide — *(deep)* The clearest practitioner walk-through of why GPU-Util lies and which DCGM fields to trust; mirrors this lesson's thesis and gives you production framing for the report.
2. **"Measuring GPU utilization one level deeper" (arXiv)** — https://arxiv.org/html/2501.16909v1 — *(deep)* Rigorous treatment of GPU-Util vs SM-occupancy vs tensor-active; use it to make the "one metric, three granularities" argument defensible in an interview.
3. **MFU explainer** — https://zeroentropy.dev/concepts/mfu/ — *(skim)* Compact reference for the MFU/HFU formulas and typical target bands. PMPP (Kirk & Hwu) ch. 3–4 — *(skim, vocabulary only)* — read solely to firm up SM/warp/block terms; do not go down the kernel-tuning path.
