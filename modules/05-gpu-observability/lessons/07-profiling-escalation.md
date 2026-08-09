---
lesson: "05.7"
title: "Profiling escalation — from metrics to Nsight"
module: "05"
concept: "Profiling escalation — from metrics to Nsight"
status: not-started
est_time: "4h"
artifacts: []
---
# 05.7 · Profiling escalation — from metrics to Nsight
> **Concept.** Metrics tell you *that* a GPU is inefficient; profiling tells you *why*. Read the DCGM shape, then climb the ladder — PyTorch Profiler → Nsight Systems → Nsight Compute — one rung at a time.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Why this matters

By L6 you can prove a GPU is busy but *unproductive*: `SM_ACTIVE` high, `PIPE_TENSOR_ACTIVE`
near zero. That is a finding, not a fix. The next question in every senior interview — and
every real incident — is: **"OK, it's inefficient. Why, and what do you do?"** Fleet telemetry
cannot answer that. DCGM samples hardware counters at 1 Hz across thousands of GPUs; it has no
idea whether the SMs are stalling on a Python dataloader, a synchronous `.item()` call, an
fp32 code path, or a memory-bound kernel. Those causes live *inside one process on one node*,
and you need a different class of tool to see them.

The skill that separates a platform engineer from a dashboard-watcher is **knowing when to
escalate and where to stop.** Profiling is expensive — Nsight Compute can slow a kernel 10–100×
by replaying it to collect counters — so you don't reach for it first, and you don't reach for
it on the whole cluster. You use the *cheap, always-on* metric to decide *which* GPU deserves
the *expensive, invasive* profiler, then climb exactly as far up the ladder as the evidence
requires. This lesson makes that decision procedure explicit, and the before/after round-trip
you produce here is a section of the flagship blog post: *"we saw the metric lie, we profiled,
we fixed it, and the honest metric moved."*

## What's new here

L2–L6 were about *reading* counters. This lesson is about *acting* on their **shape** — the
relationship between counters is the diagnostic, not any single value. Three shapes, three
escalations:

| DCGM shape (15-min avg) | What it means | Escalate to | Looking for |
|---|---|---|---|
| `SM_ACTIVE` high, `PIPE_TENSOR_ACTIVE` low | SMs busy, tensor cores idle → wrong precision / op mix / non-GEMM work | **Nsight Systems** first, then **Compute** on the hot kernel | fp32 path, tiny GEMMs, elementwise-dominated graph |
| `DRAM_ACTIVE` saturates *before* `SM_ACTIVE`/`TENSOR_ACTIVE` | memory-bandwidth bound — SMs starved waiting on HBM | **Nsight Compute** (roofline) | kernel sitting on the memory roof, fusion opportunities |
| `SM_ACTIVE` low *and* spiky, GPU util oscillating | GPU idle between bursts → host-side stall | **PyTorch Profiler** first | dataloader gaps, `cudaStreamSynchronize`, H2D copies |

The new mental model is the **escalation ladder** — each rung is more invasive, more precise,
and narrower in scope than the one below it:

```
  CHEAP / FLEET-WIDE / ALWAYS-ON                        EXPENSIVE / ONE PROCESS / ON-DEMAND
  ───────────────────────────────────────────────────────────────────────────────────────►
  DCGM metrics        PyTorch Profiler       Nsight Systems          Nsight Compute
  (1 Hz counters)     (framework trace)      (system timeline)       (single-kernel counters)
  "THAT it's bad"     "which Python line     "where the wall-clock    "why THIS kernel is
                       & CUDA op"             gaps & CPU↔GPU stalls"    slow: warp stalls"
  whole fleet         one training step      one process, one run     one kernel, replayed
```

**Rule: never skip a rung upward without evidence, and never climb a rung you don't need.**
If the PyTorch trace already shows a 40 ms dataloader gap every step, you have your answer —
stop, fix the dataloader, don't open Nsight Compute. You only reach Nsight Compute when the
timeline says *one kernel* dominates and you need to know its warp-stall reasons.

## Core notes

### Rung 0 — the metric decides *which* GPU and *which* direction

You never profile blind. The DCGM shape from L3/L6 tells you both the target and the first
rung. The three canonical fields (all `DCGM_FI_PROF_*`, the DCP profiling set, off by default):

- **`SM_ACTIVE`** — ratio of cycles an SM had ≥1 warp resident. "Something is scheduled." High
  `SM_ACTIVE` means the *scheduler* is busy — it does **not** mean useful math is happening.
- **`PIPE_TENSOR_ACTIVE`** — ratio of cycles the tensor (HMMA) pipe is active. This is the
  "am I actually doing the matmuls I bought this GPU for" number.
- **`DRAM_ACTIVE`** — ratio of cycles the HBM interface is moving data. High here + low compute
  = memory-bound.

The diagnostic is the **gap between `SM_ACTIVE` and `PIPE_TENSOR_ACTIVE`.** A GPU can report
95% `SM_ACTIVE` and 5% `PIPE_TENSOR_ACTIVE` — warps are resident and issuing instructions, but
they're fp32 CUDA-core ops, elementwise/norm/activation kernels, or tiny GEMMs that never touch
the tensor cores. That single gap is the trigger to profile. (`SM_OCCUPANCY`, the fraction of
*resident warp slots* filled, refines it: low occupancy + low tensor = launch/latency bound;
high occupancy + low tensor = wrong op mix.)

### Rung 1 — PyTorch Profiler: is the GPU even the problem?

Start here for *anything training-shaped*, because the most common cause of a wasted GPU is that
**the host isn't feeding it.** The PyTorch Profiler wraps the framework and correlates each
Python/aten op with its CUDA kernels, so you see the *step* structure:

```python
from torch.profiler import profile, ProfilerActivity, schedule

with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    schedule=schedule(wait=1, warmup=1, active=3, repeat=1),
    record_shapes=True, with_stack=True, profile_memory=True,
    on_trace_ready=torch.profiler.tensorboard_trace_handler("./tb"),
) as prof:
    for step, batch in enumerate(loader):
        train_step(batch)
        prof.step()
```

What you read out:

- **`key_averages().table(sort_by="cuda_time_total")`** — the op ranking. If the top entries are
  `aten::copy_`, `aten::to`, or elementwise ops rather than `aten::mm`/`addmm`/`_scaled_dot_product`,
  your compute is going to plumbing, not math. This is the framework-level view of the
  low-`TENSOR_ACTIVE` shape.
- **The trace timeline** (Chrome trace / TensorBoard / Perfetto) — look for a **gap at the start
  of each step** where the GPU stream is empty: that's the dataloader blocking on CPU. Look for a
  **`cudaStreamSynchronize` / `cudaDeviceSynchronize`** stretching the CPU thread: that's a
  Python-side `.item()`, `.cpu()`, `print(loss)`, or a `.numpy()` forcing a sync every step.
- **Precision** — if you expected mixed precision but see fp32 kernels (`sgemm`, no `autocast`),
  that's your low-tensor cause, found without leaving Python.

Three of the four most common wasted-GPU causes — **dataloader stall, host-sync stall, fp32
path** — are visible *here*, at the cheapest rung. Only the fourth (a genuinely slow single
kernel) forces you higher.

### Rung 2 — Nsight Systems: the system timeline

Escalate to `nsys` when the PyTorch trace is ambiguous, when it's not a PyTorch workload, or when
the stall is *below* the framework — CUDA API, NCCL, driver, H2D DMA. Nsight Systems captures a
**whole-process, whole-timeline** view: CPU threads, CUDA runtime calls, kernel executions, memcpys,
NCCL collectives, all on one wall-clock axis.

```
nsys profile -t cuda,nvtx,osrt,cudnn,cublas \
     -o step_trace --capture-range=cudaProfilerApi python train.py
```

What the timeline tells you that a counter can't:

- **Gaps** — dead space on the GPU rows = the GPU is idle. Line it up against the CPU rows to see
  *what the host was doing* during the gap (dataloader, Python, a blocking copy).
- **CPU↔GPU overlap** — are kernel launches pipelined ahead of execution, or is every launch
  serialized behind a sync? Poor overlap = launch-bound / latency-bound.
- **NCCL exposure** — in multi-GPU, whether `all_reduce` overlaps with backward compute or stalls
  it (an L8/L9 concern, but you see it first here).
- **Which kernel dominates** — Nsight Systems ranks kernels by total GPU time. This is how you
  decide whether to climb to rung 3: *if one kernel is >~30–50% of GPU time, it's worth a
  single-kernel deep dive; if time is spread across hundreds of small kernels, the answer is
  fusion/launch overhead, not one kernel.*

Annotate your code with **NVTX ranges** (`torch.cuda.nvtx.range_push/pop` or `nvtx.annotate`) so
the timeline is labelled by *your* phases (forward/backward/optimizer/dataload) instead of raw
kernel names — this is the difference between a readable trace and a wall of CUDA.

### Rung 3 — Nsight Compute: one kernel, warp-stall level

Only climb here when rung 2 named **one hot kernel** and you need to know *why that kernel* is
slow. Nsight Compute (`ncu`) profiles a **single kernel invocation**, replaying it to collect the
full hardware counter set — and that replay is why it's 10–100× slower and why you *never* run it
fleet-wide or on a whole training loop. Scope it hard:

```
ncu --set full --launch-skip 200 --launch-count 1 \
    -k regex:"scaled_dot_product|flash" -o kern python train.py
```

What only Nsight Compute sees:

- **Roofline position** — is this kernel compute-bound or memory-bound? Confirms the `DRAM_ACTIVE`
  hypothesis at kernel granularity and tells you the ceiling you're fighting.
- **Warp-stall reasons** — the scheduler samples each SM's active warp PC and stall state, so you
  get *why* warps aren't issuing: `Long Scoreboard` (waiting on global/HBM loads → memory
  latency), `MIO Throttle`, `Barrier`, `Not Selected`, `Wait`. This is the deepest "why," and
  nothing below this rung can produce it.
- **Occupancy limiters** — whether registers, shared memory, or block size is capping occupancy,
  which tells you the exact knob to turn.

If the fix at this level is "rewrite the kernel," for most platform work the real fix is upstream:
switch to a fused/flash-attention kernel, change the precision, or reshape so a library GEMM hits
the tensor cores. You rarely hand-write PTX — you use Nsight Compute to *prove which library
change* will pay off.

### What profiling sees that DCGM structurally cannot

DCGM is a 1 Hz, device-level, counter-only view. It has **no notion of time within a step, no
call stack, no per-kernel attribution, and no host side.** So it is blind to, by construction:

- **Dataloader / host-input stalls** — the GPU is idle *because the CPU didn't deliver a batch*.
  DCGM shows low util; it cannot show that the cause is a Python `DataLoader` with too few workers.
- **CPU↔GPU synchronization stalls** — a per-step `.item()` that serializes the pipeline. Invisible
  to a device counter; obvious on a timeline.
- **Kernel launch / latency bound** — thousands of tiny kernels where launch overhead dominates.
  DCGM might even show *decent* `SM_ACTIVE` while wall-clock throughput is terrible.
- **Warp-stall reasons and roofline** — *why* a kernel is slow. DCGM gives you the symptom
  (`DRAM_ACTIVE` high), never the mechanism (`Long Scoreboard` stalls, memory-bound roofline).

That list is the answer to "name a bottleneck metrics can't see" — pick any one.

## Worked example — a low-`TENSOR_ACTIVE` training job, profiled and fixed

Take the L6 workload flagged as `SM_ACTIVE ≈ 0.9`, `PIPE_TENSOR_ACTIVE ≈ 0.06` on an H100 — busy
scheduler, idle tensor cores. Round-trip:

**1. Metric shape → hypothesis.** High SM, near-zero tensor, `DRAM_ACTIVE` moderate. Not
memory-bound (DRAM isn't saturated), not idle (SM is high). Shape says **wrong op mix / precision**.
Start at rung 1, not rung 3.

**2. PyTorch Profiler.** `key_averages(sort_by="cuda_time_total")` shows the top kernels are
`aten::sgemm` (fp32) and a pile of `aten::layer_norm` / `aten::add`. No `autocast`, no tensor-core
GEMMs. The trace also shows a ~20 ms gap at each step's start. **Two causes, one trace:** an fp32
path *and* a dataloader stall.

**3. Confirm with Nsight Systems (optional at this scope).** `nsys` timeline with NVTX ranges
confirms: forward/backward run fp32 kernels, and the step-start gap lines up with a single-worker
`DataLoader` blocking the CPU thread. No single kernel dominates enough to warrant `ncu`.

**4. Fix both.**
```python
# precision: put the matmuls on the tensor cores
with torch.autocast("cuda", dtype=torch.bfloat16):
    out = model(batch)
# input pipeline: stop starving the GPU
loader = DataLoader(ds, batch_size=bs, num_workers=8,
                    pin_memory=True, persistent_workers=True, prefetch_factor=4)
```

**5. Re-read the honest metric.** After redeploy, DCGM shows `PIPE_TENSOR_ACTIVE` climbing from
**~0.06 → ~0.55**, `SM_ACTIVE` steady, step time down ~2×, and the util-lie condition
(`GPU_UTIL>90 AND SM_ACTIVE<0.2`) no longer fires. **The metric moved because the profiler told you
where to push.** That before/after — metric shape, trace finding, fix, metric delta — is the
round-trip note the deliverable wants.

## Practice — profiling round-trip that moves a DCGM metric

Feeds the deliverable ([`../practice/gpu-dashboard-lie/README.md`](../practice/gpu-dashboard-lie/README.md)).

1. Take a **low-`PIPE_TENSOR_ACTIVE`** workload from L6 (or reproduce one: an fp32 training loop
   with a single-worker dataloader on a rented GPU). Record the baseline DCGM shape.
2. **Rung 1 — PyTorch Profiler.** Capture a trace over 3–5 steps. Produce the op ranking and the
   timeline. Identify the cause(s): dataloader stall / host-sync / fp32 path / tiny batch.
3. **Rung 2 — Nsight Systems.** Capture an `nsys` timeline with NVTX ranges over a few steps.
   Confirm the gap/overlap story and rank kernels. Note whether one kernel would justify `ncu`
   (climb to rung 3 only if so).
4. **Fix** the identified cause (autocast/bf16, dataloader workers+pin_memory, remove per-step
   sync, raise batch size).
5. **Re-measure** the same DCGM field and screenshot the before/after.

**Acceptance:** a one-page **before/after profiling round-trip note** for the blog: baseline
metric shape → which rung → the specific trace finding (with the screenshot) → the fix (diff) →
the **DCGM metric moving** (before/after value + graph). It must show `PIPE_TENSOR_ACTIVE` (or the
relevant field) *change*, and it must justify *why you stopped where you did* on the ladder.

## Self-check

**(a)** `SM_ACTIVE` is high but `PIPE_TENSOR_ACTIVE` is low. Which tool do you reach for next, and
what are you looking for?

**Answer:** Start with the **PyTorch Profiler** (rung 1) if it's a framework workload, escalating to
**Nsight Systems** — *not* Nsight Compute yet. The shape means the SMs are busy but the tensor cores
are idle, so you're looking for **wrong precision or op mix**: fp32 GEMMs (no `autocast`), tiny GEMMs
that don't reach the tensor cores, or an elementwise/norm-dominated graph. In the profiler you read
the op ranking (is time in `sgemm`/copies instead of tensor-core `mm`?) and, on the timeline, whether
the compute is even the bottleneck. You only reach Nsight Compute if the timeline then names one hot
kernel you need to dissect.

**(b)** Nsight Systems vs Nsight Compute — when do you use each?

**Answer:** **Nsight Systems** is the *system timeline* profiler: whole process, wall-clock axis,
CPU threads + CUDA API + kernels + memcpys + NCCL. Use it to find **where the time goes** — gaps,
CPU↔GPU overlap, host stalls, and *which kernel dominates*. **Nsight Compute** is the *single-kernel*
profiler: it replays one kernel to collect the full counter set (roofline, occupancy limiters,
warp-stall reasons). Use it only after Nsight Systems has isolated **one hot kernel** and you need to
know *why that kernel* is slow. Systems answers "which kernel / which phase"; Compute answers "why
this kernel." Compute is 10–100× slower (replay), so you never run it fleet-wide or over a whole loop.

**(c)** Name a bottleneck DCGM metrics can't see but a profiler can.

**Answer:** A **dataloader / host-input stall** — the GPU is idle because the CPU-side `DataLoader`
isn't delivering batches. DCGM (1 Hz, device-level, no host visibility, no intra-step timeline) shows
low utilization but cannot attribute it; the PyTorch Profiler or Nsight Systems timeline shows the
GPU stream going empty at each step boundary while the CPU thread blocks in the loader. (Equally
valid: a per-step `cudaStreamSynchronize` from a `.item()` call, kernel-launch/latency-bound tiny
kernels, or the warp-stall *reason* behind a memory-bound kernel — all structurally invisible to a
device counter.)

## Resources

1. **Spheron — GPU Profiling for AI Workloads (Nsight Compute, Nsight Systems, PyTorch Profiler), 2026.**
   The single best map of the three-tool ladder and when to escalate.
   https://www.spheron.network/blog/gpu-profiling-ai-workloads-nsight-compute-pytorch-profiler-guide/
2. **PyTorch Profiler recipe (official).** The API you'll actually call for rung 1 — schedule,
   trace export, `key_averages`.
   https://pytorch.org/tutorials/recipes/recipes/profiler_recipe.html
3. **NVIDIA Nsight Systems User Guide.** Timeline semantics, NVTX ranges, `nsys` capture flags for
   rung 2. https://docs.nvidia.com/nsight-systems/UserGuide/index.html
