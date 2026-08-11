---
lesson: "03.1"
title: "Execution model and the utilisation lie"
module: "03"
concept: "Execution model and the utilisation lie"
status: not-started
est_time: "7h"
prev: null
next: "02-compute-vs-memory-bound-roofline.md"
artifacts: []
sources: 11
---

# 03.1 · Execution model and the utilisation lie

> **Concept.** Just enough of the SM/warp/kernel execution model to say precisely what `nvidia-smi` "GPU-Util" measures — and to prove it is not throughput.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Module 02 ended with you able to reason about control-plane internals — the machinery underneath the Kubernetes objects you operate. This module shifts one layer down again, from "what is the scheduler doing" to "what is the accelerator itself doing once a pod lands on it." This first lesson is the module's opening move and its thesis statement: the number every dashboard shows you for a GPU (`GPU-Util`) is not a measure of work, and until you can say precisely *why* it isn't, every other lesson in this module — roofline placement, memory bottlenecks, batching, precision, SKU choice, cost capstone — is standing on sand. Nothing here assumes prior GPU-specific knowledge; it builds the execution-model vocabulary from first principles, just deep enough to read a profiler honestly.

## Why this matters

The single most expensive mistake in GPU fleet operations is trusting `nvidia-smi` GPU-Util as a measure of work done. A GPU can report **100% utilisation while doing less than 1% of the work it is capable of** — and you are paying the same $2–4/hr either way (SemiAnalysis's ClusterMAX cohort-median rental was ~$3.15/GPU-hour as of their most recent published newsletter data — a 2026-dated snapshot, see references). If you cannot articulate, in an interview or a capacity review, exactly what that number counts and why it lies, you cannot build the cost/observability story that differentiates a Senior/Staff platform engineer at a GPU-fleet operator.

This is not hypothetical rigor. Google Cloud's own published benchmark of a 96×B200 GKE Autopilot cluster serving live LLM decode traffic measured **4.4% FLOPS utilization and tensor cores active only 1.5% of the time — while GPU-Util itself would read near 100%** ("What Does 4.4% GPU Utilization Actually Mean?", Google Cloud, Medium). That is a real production system, not a synthetic demo, exhibiting exactly the lie this lesson names. CoreWeave's job postings for *GPU Performance Engineer* explicitly ask for "hardware validation across the fleet" and "visibility into metrics/performance/health" — meaning candidates are expected to know which metric tells the truth. This lesson is the first rung of that competence: **utilisation is not throughput, and throughput is what you pay for.**

## What's new here (calibration)

Per the module README, you are not becoming a CUDA kernel developer — 02b already covered topology/power, and this module explicitly skips kernel authoring, PTX/SASS, deep microarchitecture, and occupancy-tuning-as-coding. What this lesson adds instead:

- **Precise vocabulary for "busy" at three granularities** — temporal (GPU-Util), spatial (SM_ACTIVE/SM_OCCUPANCY), and functional (TENSOR_ACTIVE) — just deep enough to pick the right metric to alert and bill on, not to write faster kernels.
- **The exact NVML definition of GPU-Util, decoded word by word**, so you can defend in an interview *why* it lies rather than asserting that it does.
- **MFU as the metric that ties utilisation to money** — a single ratio that appears in every lesson downstream of this one.
- **A second, independently-verified real-world instance of the lie** (Google Cloud's B200 benchmark) alongside the classic synthetic single-block-kernel demo — so you can cite production evidence, not just a thought experiment.

## Core concepts

### The execution model, in four words you need

- **Kernel** — a function launched to run on the GPU. One `torch.matmul` may dispatch one or several kernels. GPU-Util cares only whether *any* kernel is currently executing.
- **Thread block (CTA)** — the unit the scheduler places onto hardware. A block is assigned to exactly **one** SM and stays there until it finishes. This is the key fact: a launch of *one block* occupies *one SM* and leaves the rest idle.
- **SM (Streaming Multiprocessor)** — the independent compute unit. An **H100 SXM5 has 132 SMs**. Real throughput requires most of them busy, most of the time.
- **Warp** — 32 threads executed in lockstep; the actual scheduling quantum inside an SM. Each H100 SM can hold up to 64 resident warps (2,048 threads). "Occupancy" = resident warps ÷ max warps.

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

| Metric (DCGM field) | Field ID | What it measures | The question it answers |
|---|---|---|---|
| `GPU-Util` (NVML) | — | Temporal: any kernel running | Is it switched on? |
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | 1001 | Fraction of time the graphics/compute engine was active | Slightly better "on" signal |
| `DCGM_FI_PROF_SM_ACTIVE` | 1002 | Fraction of time ≥1 warp was resident on an SM, **averaged over all SMs** | How much of the machine floor is running? |
| `DCGM_FI_PROF_SM_OCCUPANCY` | 1003 | Resident warps ÷ max warps, averaged over SMs | How densely packed are the running SMs? |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | **1004** | Fraction of cycles the tensor (HMMA) pipe was active | Are the tensor cores — the thing you bought — doing anything? |
| `DCGM_FI_PROF_DRAM_ACTIVE` | 1005 | Fraction of cycles HBM was reading/writing | Are you memory-bound? (feeds lesson 03.2) |

(Field IDs per NVIDIA's DCGM documentation and `dcgmi dmon` output, where `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` shows as the abbreviated column `TENSO`.)

The one that kills the GPU-Util lie is **SM_ACTIVE**. Because it is *averaged over all SMs*, a one-block kernel on H100 gives roughly `1 / 132 ≈ 0.008` → **0.8%**, while GPU-Util reads 100%. That gap *is* the busy-but-idle signature.

Two subtleties to keep straight:

- **SM_ACTIVE is ambiguous on its own.** A value of 0.5 can mean "all 132 SMs busy half the time" *or* "half the SMs busy all the time." That is fine for our purpose — either way it caps the useful-work ceiling, and it is still infinitely more honest than GPU-Util. Pair it with SM_OCCUPANCY and TENSOR_ACTIVE to disambiguate.
- **TENSOR_ACTIVE is the money metric for AI.** You are paying an H100 premium *for the tensor cores*. FP64/FP32 CUDA-core work, or memory-bound elementwise ops, can pin SM_ACTIVE high while TENSOR_ACTIVE sits near zero — meaning the expensive silicon is idle even though the "GPU is busy."

**An operational trap worth knowing before you trust a dashboard:** DCGM profiling fields are not free, and they have historically had serialization bugs. NVIDIA's own `dcgm-exporter` GitHub issue #662 documents `DCGM_FI_PROF_GR_ENGINE_ACTIVE` silently reporting **0** until a separate profiling session (e.g. `dcgmi dmon -e 1001`) had been run first — a real, filed bug, not a hypothetical. The lesson: verify your exporter is actually exporting the field you think it is, on the exact driver/DCGM version you run, before you build an alert or a chargeback number on top of it.

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
- **Bill / show-back on:** integrate `SM_ACTIVE` (or, for AI tenants, `TENSOR_ACTIVE`) over time to get *useful GPU-seconds*, then divide by *allocated GPU-seconds* → the allocated-vs-utilised gap that later modules monetise. Never bill on GPU-Util; it would show every idle-but-on GPU as fully used and hide the entire waste story.
- **MIG caveat:** on a MIG-partitioned H100, per-instance profiling fields (SM_ACTIVE, TENSOR_ACTIVE) are what tell you whether each *slice* is doing work; whole-GPU GPU-Util is meaningless across partitions. Watch per-instance fields when you slice for multi-tenancy (lesson 06 goes deeper on MIG's hardware isolation).

### MFU — the one number that ties it to money

**MFU (Model FLOPs Utilisation)** is the end-to-end honesty metric: the useful FLOP/s your model actually achieved divided by the GPU's theoretical peak FLOP/s.

```
MFU = (useful model FLOP/s achieved) / (peak FLOP/s of the hardware)
```

For LLM training the standard estimate of "useful model FLOPs" is `6 · N_params · tokens` (forward + backward, per the Chinchilla/PaLM convention). So:

```
MFU = (6 · N_params · tokens_per_sec) / (num_GPUs · peak_FLOP/s_per_GPU)
```

**Real, verified numbers to calibrate against** (with the important nuance of what each counts):

| System | MFU | Note |
|---|---|---|
| GPT-3 (as cited in the PaLM paper) | 21.3% | Comparison baseline in Google's PaLM paper |
| Gopher (as cited in the PaLM paper) | 32.5% | Comparison baseline |
| **PaLM 540B** | **46.2%** (with attention FLOPs counted) / **45.7%** (without) | The canonical "good MFU" citation — note the two conventions differ by whether attention FLOPs are in the numerator; be explicit which you're quoting |
| Llama 3.1 405B pretraining, 16K H100s | **~38–43%** (reports vary; ~40% commonly cited) | A frontier, at-scale training run |
| Databricks/MosaicML, FP8 + Composer stack | **>50%** | Claimed highest published at the time; see lesson 05 for the FP8 mechanics |
| **Meta GEM (ads foundation model, LLM-scale)** | **20–25%** | A real production non-LLM-chat training system at "several thousand" latest-gen GPUs — doubling MFU was Meta's explicit 12-month engineering goal |

Using **H100 dense BF16 peak = 989 TFLOP/s** as the denominator (see the sparsity note below), **a good, achievable target is 40–60% MFU** for well-tuned large-scale pretraining in 2026; 50%+ is considered excellent. Anything under ~30% is a cost problem worth investigating — 15% means you are burning roughly 3× the GPU-hours you should for the same useful work.

The Meta GEM number is worth sitting with: even a top-tier engineering organization runs *production* training at 20–25% MFU, well below the PR-quotable 40%+ numbers from headline pretraining runs. That is a **2–2.5× real dollar difference** for identical hardware, and it tells you MFU targets are workload-regime-specific, not a universal constant — know which regime (a from-scratch frontier pretrain vs. a continuously-retrained production model) you are being asked about before quoting a "good" number.

A cousin metric, **HFU (Hardware FLOPs Utilisation)**, counts *all* FLOPs the hardware executed including activation recomputation. MFU ≤ HFU always. For cost reasoning prefer **MFU**, because it measures FLOPs that advance the model — the thing you are actually buying — not FLOPs spent recomputing to save memory.

### Occupancy is not throughput either

A trap one level deeper, and a favourite interview follow-up: **high SM_OCCUPANCY does not guarantee high throughput.** Occupancy measures resident warps ÷ max warps — how *full* the SM's scheduler slots are. A memory-bound kernel can have 100% occupancy while every warp is *stalled waiting on HBM*; the SM is packed but the ALUs idle. Conversely a well-tuned GEMM can hit near-peak FLOP/s at only ~50% occupancy because it has enough in-flight work to hide latency. So the ladder of honesty is: GPU-Util (is it on?) → SM_ACTIVE (is the floor running?) → SM_OCCUPANCY (are the SMs packed?) → **TENSOR_ACTIVE / achieved FLOP/s (is it doing the work you paid for?)**. Only the last rung is throughput; the next lesson (03.2 roofline) is what turns it into a cost number.

### Collection gotchas (ops reality)

- **Profiling fields cost something.** DCGM `PROF_*` fields require the profiling API and add small overhead; historically some fields *serialised* with each other and could not all be sampled simultaneously on older DCGM/driver combos (see the dcgm-exporter #662 issue above). Verify your `dcgm-exporter` config actually exports SM_ACTIVE and TENSOR_ACTIVE together before trusting a dashboard.
- **GPU-Util is free and always there; that is exactly why it is over-trusted.** It comes from NVML with zero setup, so it is the default in every quick-look tool — which is how the lie propagates.
- **Sample windows differ.** `nvidia-smi`'s duty cycle and DCGM's cycle-fraction fields are measured over different windows; do not expect them to reconcile arithmetically. Use each for its purpose, not to cross-check the other.

### The sparsity asterisk (a common interview trap)

NVIDIA's H100 datasheet advertises **1,979 BF16 TFLOPS** and **3,958 FP8 TFLOPS**. Those are the **structured-sparsity** figures (2× marketing multiplier that assumes a 2:4 sparse weight pattern most workloads do not use). The **dense** numbers — the honest denominator for MFU on ordinary dense training/inference — are **989 BF16 / 1,979 FP8 TFLOP/s**. H100 SXM also carries **80 GB HBM3 at 3.35 TB/s**. Quoting the sparse number as if it were your dense ceiling silently halves your apparent MFU target; know which one you are dividing by. (Figures per NVIDIA's Hopper architecture whitepaper, corroborated across multiple vendor datasheet mirrors — treat as a dated hardware-generation snapshot, not something that changes, but re-verify against the current whitepaper if citing in a report.)

## Perspectives

**Developer/ML-engineer view.** MFU is the single number that tells an ML engineer whether their training run's recipe — parallelism strategy, kernel choices, precision — is well-tuned. The PaLM (46.2%/45.7%), Llama 3.1 (~40%), Databricks (>50%), and Meta GEM (20–25%) numbers above are the calibration points you carry into any conversation about "is this run efficient." Knowing that PaLM's own paper reports *two* MFU numbers depending on whether attention FLOPs are counted is the kind of precision that separates someone who read the paper from someone who memorized a headline figure.

**Platform/SRE view.** The fleet-alerting conjunction (`GR_ENGINE_ACTIVE` high AND `SM_ACTIVE` low AND `TENSOR_ACTIVE` low) is the shape of every real busy-but-idle alert you will write. The DCGM serialization gotcha (issue #662) is the operational trap that turns a theoretically correct alert into a silently-broken one — a metric that looks like it's exporting but is actually stuck at zero from a stale profiling session is worse than no metric, because it creates false confidence.

**Hardware view.** SM/warp/tensor-core granularity is a *physical* fact — 132 independent compute islands on an H100 die, each with its own warp scheduler and tensor-core pipe. GPU-Util's lie is structural, not a monitoring bug: NVML was never designed to report per-SM occupancy, because its job is coarse power/thermal/duty-cycle telemetry, not a performance-counter API. That's what DCGM's `PROF_*` fields exist to add.

**Economics/FinOps view.** Meta's 20–25% vs. Databricks' 50%+ shows there's a real dollar spread — a 2–2.5× cost difference — for the identical hardware bill, driven entirely by software efficiency rather than SKU choice. And Google Cloud's 4.4%-utilization B200 benchmark makes the sharper point: low utilization is not automatically a problem to fix. At batch=1 decode, the workload is fundamentally memory-bound (next lesson's vocabulary) and low FLOPS-util is the physically correct outcome — the actual business metric (1M tokens/sec served) came from batching across many concurrent streams, not from raising any single stream's utilization. Knowing when low utilization is waste versus physics is the FinOps judgment call this whole module builds toward.

## Real-world use cases

- **["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — Google Cloud (Medium, official publication).** A real benchmark on GKE Autopilot, 96×B200 GPUs, serving ~1M tok/s at decode batch=1: **4.4% FLOPS utilization, 10.9% memory-bandwidth utilization, tensor cores active only 1.5% of the time** — while GPU-Util itself would read near 100%. Explicitly frames this as roofline physics, not a bug. What it shows: the GPU-Util lie and its correct diagnosis, on real 2026-current production hardware, not a synthetic demo.
- **NVIDIA/dcgm-exporter GitHub, [issue #662](https://github.com/NVIDIA/dcgm-exporter/issues/662).** A real, filed bug where `DCGM_FI_PROF_GR_ENGINE_ACTIVE` silently returns 0 until a separate profiling session is run. What it shows: even the "honest" metrics have collection failure modes — verify your exporter, don't just trust the field name.
- **CoreWeave, ["HPC Verification" docs](https://docs.coreweave.com/docs/platform/fleet-management/hpc-verification).** CoreWeave's real, hourly, per-idle-node hardware-validation framework — runs 20–30 minute tests including tensor-core benchmarking at FP8/FP16/BF16 and "all-SMs-at-100%" thermal checks before a node is handed to a customer. What it shows: fleet-scale observability built on exactly the granular metrics (not GPU-Util) this lesson teaches, at the company this module's job-posting hook is drawn from.
- **Meta Engineering, ["GEM Training: How Meta Doubled the Efficiency of Its LLM-Scale Ads Foundation Model"](https://engineering.fb.com/2026/08/03/ml-applications/training-gem-at-llm-scale-meta-ads-recommendation-foundation-model/).** Shows MFU (20–25%) as the operating metric for a real large-scale production training system, with doubling efficiency as an explicit 12-month engineering goal. What it shows: MFU-chasing is a real job function with a real roadmap behind it, not an academic exercise — and that "good" MFU is regime-dependent.

## Worked example

You rent one H100 SXM ($/hr, on-demand ≈ $3 — a 2026 snapshot, verify current pricing) and launch a deliberately trivial kernel — a single-threaded elementwise op on a tiny tensor, looped so it never stops:

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

The cost reading: you are paying **$3/hr for a GPU that is 100% "utilised" and ~99% wasted.** Across a 40-cluster fleet, alerting on GPU-Util would show this node as perfectly healthy. Alerting on SM_ACTIVE / TENSOR_ACTIVE catches it.

**A second, real-hardware data point for the same argument** — Google Cloud's B200 benchmark, worked through: at decode batch=1, weights in BF16 (2 bytes/param, ~2 FLOP/param for the matmul), arithmetic intensity ≈ 1 FLOP/byte. B200's own compute:bandwidth ridge point sits roughly **292× higher** than that (the article's own stated multiple — you'll formalize what a "ridge point" is in lesson 03.2). The result: tensor-active 1.5%, FLOPS-util 4.4%, bandwidth-util 10.9% — and yet the cluster served **1M tokens/sec** in aggregate. The nuance worth internalizing: at scale, via batching across many concurrent decode streams, low *per-GPU* utilization does not mean a bad business outcome. The single-block-kernel demo above shows the pathological case (wasted spend, no excuse); the B200 benchmark shows the legitimate case (physically-bound, and the fix is architectural — batching, disaggregation — not "increase utilization" as a goal in itself). Distinguishing these two is exactly the judgment this lesson is building.

## Practice

**Task (rent one H100 or A100 by the hour; budget ~1 hr ≈ $2–4).**

1. Install DCGM / `dcgm-exporter` (or use `dcgmi dmon`) alongside `nvidia-smi`.
2. Run a **busy-but-idle** workload: a single-block CUDA kernel, or the trivial PyTorch loop above, so GPU-Util pins near 100%.
3. Simultaneously capture, in one log with timestamps: `nvidia-smi` GPU-Util, `DCGM_FI_PROF_SM_ACTIVE`, `DCGM_FI_PROF_SM_OCCUPANCY`, and `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (field 1004).
4. Save a table / screenshot showing **~100% GPU-Util next to ~0% SM-active / tensor-active.**
5. Before you trust the numbers, sanity-check for the collection gotcha above: confirm `DCGM_FI_PROF_GR_ENGINE_ACTIVE` is actually non-zero and moving, not silently stuck (the dcgm-exporter #662 failure mode) — run a second `dcgmi dmon` session and see if the reading changes.

**Acceptance:** a single artifact (log + one annotated screenshot or table) in which GPU-Util and SM/tensor-active are shown *side by side for the same instant*, with a one-sentence caption stating why the GPU is "busy but idle." This is Exhibit A of the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md) — keep the raw log, you will reuse the same rented-GPU session across lessons 01–06.

## Common pitfalls

1. **Believing GPU-Util at 100% during decode is automatically a code smell.** It usually isn't — the Google Cloud benchmark explicitly makes the "this is physics, not a bug" point for batch-1 decode. Check which roof you're under (lesson 03.2) before assuming waste.
2. **Reading SM_ACTIVE = 50% as "half the SMs are always busy."** It's ambiguous by construction — it could equally mean "all SMs half the time." Pair it with SM_OCCUPANCY and TENSOR_ACTIVE to disambiguate; don't over-read a single number.
3. **Trusting a DCGM dashboard without checking the field is actually exporting.** The dcgm-exporter #662 bug shows a field can silently read 0 due to a profiling-session conflict, not because the GPU is actually idle.
4. **Quoting the sparse TFLOPS figure (e.g. 1,979 BF16) as your MFU denominator.** This silently doubles your apparent efficiency — always confirm dense vs. sparse before dividing.
5. **Assuming one "good MFU" number applies everywhere.** Meta's own production ads-training system runs at 20–25% while their (and others') PR-quotable pretraining runs claim 40%+. Know which regime — from-scratch frontier pretrain vs. continuously-retrained production system — you're being asked about.

## Self-check

- Why can a 1-SM (single-block) kernel report 100% GPU-Util? **Answer:** GPU-Util is a *temporal* duty cycle — "percent of the sample window during which one or more kernels was executing." A single always-resident block means a kernel is *always* running, so the window is 100% covered. It counts *time non-idle*, not *SMs engaged* (1 of 132) and not *FLOP/s*, so it saturates while ~99% of the hardware sits idle.
- Which metric would you fleet-alert on to catch a "busy but idle" GPU, and what's the one operational check you must run before trusting it? **Answer:** `DCGM_FI_PROF_SM_ACTIVE` (averaged over all SMs) as the primary "is the floor actually running" signal, and `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (field 1004) for AI workloads to confirm the tensor cores you paid for are engaged. Alert on *high GPU-Util combined with low SM_ACTIVE/TENSOR_ACTIVE* — never alert on GPU-Util alone. Before trusting the dashboard, confirm the profiling field is actually populating (not silently stuck at 0 from a stale/conflicting profiling session — the real dcgm-exporter #662 failure mode).
- Define MFU and give a good target, citing at least two real reported numbers. **Answer:** MFU (Model FLOPs Utilisation) = useful model FLOP/s achieved ÷ theoretical peak FLOP/s of the hardware, using `6·N_params·tokens` as the useful-FLOP estimate for training. A good, achievable target is **40–60%** for well-tuned large-scale pretraining in 2026. PaLM reported 46.2% (with attention FLOPs) / 45.7% (without); Databricks/MosaicML claimed >50% with FP8; but Meta's production GEM ads-training system runs at only 20–25% — showing "good MFU" is regime-dependent, not a single universal number. Always divide by the *dense* peak (989 BF16 TFLOP/s on H100), not the sparse datasheet figure.
- Why can Google's own B200 benchmark show 4.4% FLOPS utilization while serving 1M tokens/sec, and why is that *not* a problem to fix? **Answer:** Decode at batch=1 has arithmetic intensity ≈1 FLOP/byte, roughly 292× below the B200's compute:bandwidth ridge point — so the workload is fundamentally memory-bound and low FLOPS-utilization is the physically correct ceiling for a single stream, not waste. The 1M tok/s aggregate throughput comes from batching many concurrent decode streams to fill the same fixed bandwidth budget, not from raising any single stream's utilization — the fix for "low utilization" here is architectural (batching, disaggregation), not chasing a utilization number for its own sake.

## Connections & what's next

This lesson supplies every vocabulary term the rest of the module depends on: DRAM_ACTIVE and the "roof" concept feed directly into lesson 03.2's roofline model; MFU as a ratio reappears as the cost-per-token denominator in lesson 03.7's capstone; the DCGM fields you learned to read here are the exact ones the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md) asks you to capture and interpret across every subsequent lesson's practice section. The economics thread — that a "busy" GPU can be burning money doing nothing useful — is the thread every later lesson in this module refines into a sharper cost argument.

Next: **[03.2 · Compute-bound vs memory-bound and the roofline](02-compute-vs-memory-bound-roofline.md)** takes "TENSOR_ACTIVE is low" and "DRAM_ACTIVE is high" from this lesson's metric table and turns them into a formal, predictive model — the roofline — that tells you *in advance*, before you spend a dollar, whether a workload even has room to get faster.

## References & further reading

**Primary sources**
- [NVIDIA DCGM documentation](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html) — canonical source for `DCGM_FI_PROF_*` field definitions, including `PIPE_TENSOR_ACTIVE` (field 1004) used throughout this lesson.
- [NVIDIA H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — canonical source for the 132-SM count, dense/sparse TFLOPS figures, and HBM3 bandwidth; read to separate dense from with-sparsity numbers before doing MFU math.
- PaLM paper (Chowdhery et al.), arXiv 2204.02311 — source of the 46.2%/45.7% MFU figures (with/without attention FLOPs) and the GPT-3 (21.3%) / Gopher (32.5%) comparison points.

**Real-world engineering blogs**
- Google Cloud, ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — real B200 production benchmark showing the GPU-Util lie and correct roofline-based diagnosis; reused as this lesson's headline worked example.
- Meta Engineering, ["GEM Training: How Meta Doubled the Efficiency of Its LLM-Scale Ads Foundation Model"](https://engineering.fb.com/2026/08/03/ml-applications/training-gem-at-llm-scale-meta-ads-recommendation-foundation-model/) — real production MFU (20-25%) and a 12-month efficiency-doubling engineering goal, the regime-dependence counterpoint to headline pretraining MFU numbers.
- CoreWeave, ["HPC Verification" docs](https://docs.coreweave.com/docs/platform/fleet-management/hpc-verification) — CoreWeave's real hourly fleet hardware-validation framework, grounded in the same granular metrics this lesson teaches.
- NVIDIA/dcgm-exporter GitHub, [issue #662](https://github.com/NVIDIA/dcgm-exporter/issues/662) — a real, filed bug where a "trusted" DCGM field silently reports 0; the concrete instance behind this lesson's "verify your exporter" pitfall.

**Deeper dives**
- Modal, ["I paid for the whole GPU, I am going to use the whole GPU" (GPU utilisation guide)](https://modal.com/blog/gpu-utilization-guide) — a clear practitioner walk-through of why GPU-Util lies and which DCGM fields to trust; production framing for the deliverable.
- ["Measuring GPU utilization one level deeper"](https://arxiv.org/html/2501.16909v1) (arXiv) — rigorous treatment of GPU-Util vs. SM-occupancy vs. tensor-active; useful for making the "one metric, three granularities" argument defensible in an interview.
- Programming Massively Parallel Processors (Kirk & Hwu), ch. 3–4 — read solely to firm up SM/warp/block vocabulary; do not go down the kernel-tuning path.
