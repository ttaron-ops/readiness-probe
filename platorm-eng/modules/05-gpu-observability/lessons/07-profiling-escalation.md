---
lesson: "05.7"
title: "Profiling escalation — from metrics to Nsight"
module: "05"
concept: "Profiling escalation — from metrics to Nsight"
status: not-started
est_time: "6h"
prev: "06-inference-slos.md"
next: "08-capstone-allocated-vs-utilised.md"
artifacts: []
sources: 5
---
# 05.7 · Profiling escalation — from metrics to Nsight

> **Concept.** Metrics tell you *that* a GPU is inefficient; profiling tells you *why*. Read the DCGM shape, then climb the ladder — PyTorch Profiler → Nsight Systems → Nsight Compute — one rung at a time.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

L1–L5 taught you to read the fleet honestly (`SM_ACTIVE` vs `GPU_UTIL`, XID triage), and L6 taught you the inference-specific split (TTFT vs TPOT). Both leave you at the same wall: you now have a *finding* — "this GPU is busy but unproductive," "TTFT blew up while TPOT stayed flat" — but a finding is not a fix. DCGM's 1 Hz, device-level counters cannot see inside a process: no call stack, no per-kernel attribution, no host-side view. This lesson is the bridge from "the metric says something is wrong" to "here is the line of code, the kernel, or the config that's wrong," and it hands the capstone (L8) the one thing it's still missing — a documented before/after profiling round-trip that proves the honest metric can be *moved*, not just observed.

## Why this matters

Every senior platform interview that gets past "what does `GPU_UTIL` measure" moves to the follow-up: *"OK, it's inefficient. Why, and what do you do?"* Fleet telemetry cannot answer that question by construction, and neither can a shrug. Meta's own engineering team has published exactly this problem at production scale: a training job showing a repeating **GPU-idle, GPU-active, GPU-idle** pattern, with the GPU sitting idle **more than half the total training time** — found not with an exotic tool but with the same class of framework-level profiler this lesson opens with (Meta's internal MAIProf, built on Kineto, the same tracing layer PyTorch's public `torch.profiler` uses). That is the concrete cost of not knowing this skill: half your paid GPU-hours evaporating in a pattern that's invisible to DCGM and trivially visible to a profiler, on one of the best-resourced ML infra teams in the industry. The corollary interview trap is reaching for the heaviest tool first — Nsight Compute replays a kernel to collect counters, at 10–100× slowdown — and platform engineers who "profile everything with everything" burn cluster time and credibility. The skill graded here is judgment: which rung, on which GPU, for how long.

## What's new here (calibration)

You already know how to read the DCGM PROF fields (L1–L2) and how they're labeled per pod (L3–L4) — this lesson doesn't re-teach metric semantics, it starts from a metric *shape* as a given and asks what to do next. Also assumed: you can read a Python traceback and a shell profiler invocation; this isn't a "how to use a terminal" lesson. What's genuinely new:

- **The escalation ladder as a decision procedure**, not a tool tour — which shape sends you to which rung, and the discipline of stopping as soon as the evidence answers the question.
- **Framework-level tracing internals** (Kineto/`libcupti`) that connect what `torch.profiler` shows you to the CUDA-level tracing infrastructure Nsight tools also use — so you understand *why* rung 1 is cheap and rungs 2–3 aren't just "the same thing with a fancier UI."
- **Warp-stall-level diagnosis** (Nsight Compute's stall reasons, roofline position) — a level of hardware detail the earlier lessons' fleet counters cannot express at all.

## Core concepts

### Rung 0 — the metric decides *which* GPU and *which* direction

You never profile blind. The DCGM shape from L1/L2/L6 tells you both the target and the first rung. The three canonical fields (all `DCGM_FI_PROF_*`, the DCP profiling set, off by default — see L2 for why multiplexing bounds how many you can sample at once):

- **`SM_ACTIVE`** — ratio of cycles an SM had ≥1 warp resident. "Something is scheduled." High `SM_ACTIVE` means the *scheduler* is busy — it does **not** mean useful math is happening.
- **`PIPE_TENSOR_ACTIVE`** — ratio of cycles the tensor (HMMA) pipe is active. This is the "am I actually doing the matmuls I bought this GPU for" number.
- **`DRAM_ACTIVE`** — ratio of cycles the HBM interface is moving data. High here + low compute = memory-bound.

The diagnostic is the **gap between `SM_ACTIVE` and `PIPE_TENSOR_ACTIVE`.** A GPU can report 95% `SM_ACTIVE` and 5% `PIPE_TENSOR_ACTIVE` — warps are resident and issuing instructions, but they're fp32 CUDA-core ops, elementwise/norm/activation kernels, or tiny GEMMs that never touch the tensor cores. That single gap is the trigger to profile. (`SM_OCCUPANCY`, the fraction of *resident warp slots* filled, refines it: low occupancy + low tensor = launch/latency bound; high occupancy + low tensor = wrong op mix.)

Three canonical shapes map to three different first rungs:

| DCGM shape (15-min avg) | What it means | Escalate to | Looking for |
|---|---|---|---|
| `SM_ACTIVE` high, `PIPE_TENSOR_ACTIVE` low | SMs busy, tensor cores idle → wrong precision / op mix / non-GEMM work | **Nsight Systems** first, then **Compute** on the hot kernel | fp32 path, tiny GEMMs, elementwise-dominated graph |
| `DRAM_ACTIVE` saturates *before* `SM_ACTIVE`/`TENSOR_ACTIVE` | memory-bandwidth bound — SMs starved waiting on HBM | **Nsight Compute** (roofline) | kernel sitting on the memory roof, fusion opportunities |
| `SM_ACTIVE` low *and* spiky, GPU util oscillating (Meta's "idle/active/idle" case, above) | GPU idle between bursts → host-side stall | **PyTorch Profiler** first | dataloader gaps, `cudaStreamSynchronize`, H2D copies |

The mental model is the **escalation ladder** — each rung is more invasive, more precise, and narrower in scope than the one below it:

```
  CHEAP / FLEET-WIDE / ALWAYS-ON                        EXPENSIVE / ONE PROCESS / ON-DEMAND
  ───────────────────────────────────────────────────────────────────────────────────────►
  DCGM metrics        PyTorch Profiler       Nsight Systems          Nsight Compute
  (1 Hz counters)     (framework trace)      (system timeline)       (single-kernel counters)
  "THAT it's bad"     "which Python line     "where the wall-clock    "why THIS kernel is
                       & CUDA op"             gaps & CPU↔GPU stalls"    slow: warp stalls"
  whole fleet         one training step      one process, one run     one kernel, replayed
```

**Rule: never skip a rung upward without evidence, and never climb a rung you don't need.** If the PyTorch trace already shows a 40 ms dataloader gap every step, you have your answer — stop, fix the dataloader, don't open Nsight Compute. You only reach Nsight Compute when the timeline says *one kernel* dominates and you need to know its warp-stall reasons. Spheron's practitioner guide to this ladder makes the same point from the opposite direction: it observes that "most LLM inference slowdowns come from 2–5 kernels that a 10-minute profile would identify" — the ladder isn't slow, *skipping straight to the top of it* is what's slow.

### Rung 1 — PyTorch Profiler: is the GPU even the problem?

Start here for *anything training-shaped*, because the most common cause of a wasted GPU is that **the host isn't feeding it.** The PyTorch Profiler wraps the framework and correlates each Python/aten op with its CUDA kernels, so you see the *step* structure:

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

- **`key_averages().table(sort_by="cuda_time_total")`** — the op ranking. If the top entries are `aten::copy_`, `aten::to`, or elementwise ops rather than `aten::mm`/`addmm`/`_scaled_dot_product`, your compute is going to plumbing, not math. This is the framework-level view of the low-`TENSOR_ACTIVE` shape.
- **The trace timeline** (Chrome trace / TensorBoard / Perfetto) — look for a **gap at the start of each step** where the GPU stream is empty: that's the dataloader blocking on CPU. Look for a **`cudaStreamSynchronize` / `cudaDeviceSynchronize`** stretching the CPU thread: that's a Python-side `.item()`, `.cpu()`, `print(loss)`, or a `.numpy()` forcing a sync every step.
- **Precision** — if you expected mixed precision but see fp32 kernels (`sgemm`, no `autocast`), that's your low-tensor cause, found without leaving Python.

Three of the four most common wasted-GPU causes — **dataloader stall, host-sync stall, fp32 path** — are visible *here*, at the cheapest rung. Only the fourth (a genuinely slow single kernel) forces you higher. Architecturally, `torch.profiler`'s CUDA-side traces are captured through **Kineto**, PyTorch's internal profiling library, which itself wraps NVIDIA's **`libcupti`** (the CUDA Profiling Tools Interface) — the same low-level tracing substrate that underlies Nsight Systems' kernel-timeline view. That's *why* rung 1 is cheap: you're reading a lightweight trace of the same events, not replaying anything.

### Rung 2 — Nsight Systems: the system timeline

Escalate to `nsys` when the PyTorch trace is ambiguous, when it's not a PyTorch workload, or when the stall is *below* the framework — CUDA API, NCCL, driver, H2D DMA. Nsight Systems captures a **whole-process, whole-timeline** view: CPU threads, CUDA runtime calls, kernel executions, memcpys, NCCL collectives, all on one wall-clock axis.

```
nsys profile -t cuda,nvtx,osrt,cudnn,cublas \
     -o step_trace --capture-range=cudaProfilerApi python train.py
```

What the timeline tells you that a counter can't:

- **Gaps** — dead space on the GPU rows = the GPU is idle. Line it up against the CPU rows to see *what the host was doing* during the gap (dataloader, Python, a blocking copy). This is the tool that would have caught Meta's idle/active/idle pattern directly, if rung 1 hadn't already answered it.
- **CPU↔GPU overlap** — are kernel launches pipelined ahead of execution, or is every launch serialized behind a sync? Poor overlap = launch-bound / latency-bound.
- **NCCL exposure** — in multi-GPU, whether `all_reduce` overlaps with backward compute or stalls it (an L8/module-06 concern, but you see it first here).
- **Which kernel dominates** — Nsight Systems ranks kernels by total GPU time. This is how you decide whether to climb to rung 3: *if one kernel is >~30–50% of GPU time, it's worth a single-kernel deep dive; if time is spread across hundreds of small kernels, the answer is fusion/launch overhead, not one kernel.*

Annotate your code with **NVTX ranges** (`torch.cuda.nvtx.range_push/pop` or `nvtx.annotate`) so the timeline is labelled by *your* phases (forward/backward/optimizer/dataload) instead of raw kernel names — this is the difference between a readable trace and a wall of CUDA.

### Rung 3 — Nsight Compute: one kernel, warp-stall level

Only climb here when rung 2 named **one hot kernel** and you need to know *why that kernel* is slow. Nsight Compute (`ncu`) profiles a **single kernel invocation**, replaying it to collect the full hardware counter set — and that replay is why it's 10–100× slower and why you *never* run it fleet-wide or on a whole training loop. The replay requirement is a direct consequence of the same physical limit L2 introduced for DCGM: the GPU has a **finite number of hardware performance-counter collection slots**, so a full counter set for an arbitrary kernel can't be gathered in one live pass — Nsight Compute re-executes the kernel deterministically, once per counter group, to get around it. Scope it hard:

```
ncu --set full --launch-skip 200 --launch-count 1 \
    -k regex:"scaled_dot_product|flash" -o kern python train.py
```

What only Nsight Compute sees:

- **Roofline position** — is this kernel compute-bound or memory-bound? Confirms the `DRAM_ACTIVE` hypothesis at kernel granularity and tells you the ceiling you're fighting.
- **Warp-stall reasons** — the scheduler samples each SM's active warp PC and stall state, so you get *why* warps aren't issuing: `Long Scoreboard` (waiting on global/HBM loads → memory latency), `MIO Throttle`, `Barrier`, `Not Selected`, `Wait`. This is the deepest "why," and nothing below this rung can produce it.
- **Occupancy limiters** — whether registers, shared memory, or block size is capping occupancy, which tells you the exact knob to turn.

The magnitude is real, not theoretical: NVIDIA's own "Using Nsight Compute to Inspect your Kernels" walkthrough documents a case where profiling and optimizing a single kernel produced an **87.5% reduction in memory transactions** and a **68% reduction in kernel execution duration** — that's the scale of win the 10–100× profiling overhead is buying you, on the *one* kernel it's worth paying for.

If the fix at this level is "rewrite the kernel," for most platform work the real fix is upstream: switch to a fused/flash-attention kernel, change the precision, or reshape so a library GEMM hits the tensor cores. You rarely hand-write PTX — you use Nsight Compute to *prove which library change* will pay off.

### What profiling sees that DCGM structurally cannot

DCGM is a 1 Hz, device-level, counter-only view. It has **no notion of time within a step, no call stack, no per-kernel attribution, and no host side.** So it is blind to, by construction:

- **Dataloader / host-input stalls** — the GPU is idle *because the CPU didn't deliver a batch*. DCGM shows low util; it cannot show that the cause is a Python `DataLoader` with too few workers.
- **CPU↔GPU synchronization stalls** — a per-step `.item()` that serializes the pipeline. Invisible to a device counter; obvious on a timeline.
- **Kernel launch / latency bound** — thousands of tiny kernels where launch overhead dominates. DCGM might even show *decent* `SM_ACTIVE` while wall-clock throughput is terrible.
- **Warp-stall reasons and roofline** — *why* a kernel is slow. DCGM gives you the symptom (`DRAM_ACTIVE` high), never the mechanism (`Long Scoreboard` stalls, memory-bound roofline).

That list is the answer to "name a bottleneck metrics can't see" — pick any one.

## Perspectives

**Developer.** From inside the training script, you never see `SM_ACTIVE`; you see step time and, if you look, a profiler trace. Most of the four common wasted-GPU causes — dataloader stalls, host syncs, fp32 paths — resolve entirely at rung 1, without ever touching Nsight. A developer doesn't need Nsight literacy to fix the majority of real bugs; they need profiler literacy and the discipline to look before guessing.

**Operator/platform.** The escalation discipline — climb only as far as the evidence demands — is what separates a platform engineer who can *unblock* a stuck team from one who can only report "the GPU looks inefficient" and hand the problem back. Knowing *which rung* to reach for, on *which GPU*, is the actual senior skill; running every tool on everything is not rigor, it's noise (and, at 10–100× overhead on Nsight Compute, real cluster cost).

**Hardware.** Nsight Compute's replay-based counter collection isn't a UI choice — it's forced by a physical constraint: GPUs have a finite number of hardware performance-counter collection slots, the same constraint from L2 that makes DCGM multiplex PROF fields. A full counter set for an arbitrary kernel can only be gathered by re-running it once per counter group. Profiling cost is a hardware fact you're paying for, not a software inefficiency someone could patch away.

**Cost/failure-mode.** Meta's production case is the sobering data point: a real training job burned more than half its wall-clock time idle, on infrastructure run by one of the best-funded ML platform teams anywhere, found with a rung-1-class tool. The lesson isn't "big companies also have bugs" — it's that this failure mode is common enough, and cheap enough to detect, that *not* profiling routinely is itself the anomaly.

## Real-world use cases

- **PyTorch (Meta) — "Performance Debugging of Production PyTorch Models at Meta"** — https://pytorch.org/blog/performance-debugging-of-production-pytorch-models-at-meta/. A production PyTorch training job showed a repeating "GPU-idle, GPU-active, GPU-idle" pattern with the GPU idle more than half the training time; Meta's internal MAIProf (built on Kineto/`libcupti`, the same layer under public `torch.profiler`) found it by inspecting CPU/GPU timelines side by side. **What it shows:** the exact rung-1 methodology this lesson teaches, applied at hyperscale production, and it's the direct real-world analog of this lesson's own worked example.
- **NVIDIA Developer Blog — "Using Nsight Compute to Inspect your Kernels"** — https://developer.nvidia.com/blog/using-nsight-compute-to-inspect-your-kernels/. A worked case study where profiling a single kernel with Nsight Compute drove an 87.5% reduction in memory transactions and a 68% reduction in kernel execution duration. **What it shows:** a concrete, numeric rung-3 result — proof that the 10–100× profiling overhead is buying a real, large fix, not a marginal one.

*(Both URLs are the canonical pages confirmed via search; this session's sandbox blocks direct fetch to `pytorch.org` and `developer.nvidia.com` — treat as [SEARCH-VERIFIED], not independently re-fetched here.)*

## Worked example

Take the L1/L6 workload flagged as `SM_ACTIVE ≈ 0.9`, `PIPE_TENSOR_ACTIVE ≈ 0.06` on an H100 — busy scheduler, idle tensor cores. Round-trip:

**1. Metric shape → hypothesis.** High SM, near-zero tensor, `DRAM_ACTIVE` moderate. Not memory-bound (DRAM isn't saturated), not idle (SM is high). Shape says **wrong op mix / precision**. Start at rung 1, not rung 3.

**2. PyTorch Profiler.** `key_averages(sort_by="cuda_time_total")` shows the top kernels are `aten::sgemm` (fp32) and a pile of `aten::layer_norm` / `aten::add`. No `autocast`, no tensor-core GEMMs. The trace also shows a ~20 ms gap at each step's start. **Two causes, one trace:** an fp32 path *and* a dataloader stall.

**3. Confirm with Nsight Systems (optional at this scope).** `nsys` timeline with NVTX ranges confirms: forward/backward run fp32 kernels, and the step-start gap lines up with a single-worker `DataLoader` blocking the CPU thread. No single kernel dominates enough to warrant `ncu`.

**4. Fix both.**
```python
# precision: put the matmuls on the tensor cores
with torch.autocast("cuda", dtype=torch.bfloat16):
    out = model(batch)
# input pipeline: stop starving the GPU
loader = DataLoader(ds, batch_size=bs, num_workers=8,
                    pin_memory=True, persistent_workers=True, prefetch_factor=4)
```

**5. Re-read the honest metric.** After redeploy, DCGM shows `PIPE_TENSOR_ACTIVE` climbing from **~0.06 → ~0.55**, `SM_ACTIVE` steady, step time down ~2×, and the util-lie condition (`GPU_UTIL>90 AND SM_ACTIVE<0.2`) no longer fires. **The metric moved because the profiler told you where to push.** That before/after — metric shape, trace finding, fix, metric delta — is the round-trip note the deliverable wants.

**Contrast shape — the Meta-style idle/active case.** Same discipline, different starting symptom: instead of a steady-but-wrong shape, `SM_ACTIVE` oscillates — periods near 0.8 alternating with periods near 0.02 every few seconds, GPU util flapping in lockstep. Rung 1's trace shows the idle windows correlate 1:1 with the CPU thread blocked in `DataLoader.__next__` — a pure host-input stall, confirmed and fixed (more workers, pinning, prefetch) *without ever opening Nsight Systems*. The lesson: two very different-looking DCGM shapes (steady-wrong vs. oscillating) can both resolve at rung 1 — the shape tells you *where to start looking*, not automatically *how far you'll have to climb*.

## Practice

Feeds the deliverable ([`../practice/gpu-dashboard-lie/README.md`](../practice/gpu-dashboard-lie/README.md)).

1. Take a **low-`PIPE_TENSOR_ACTIVE`** workload from L1/L6 (or reproduce one: an fp32 training loop with a single-worker dataloader on a rented GPU). Record the baseline DCGM shape.
2. **Rung 1 — PyTorch Profiler.** Capture a trace over 3–5 steps. Produce the op ranking and the timeline. Identify the cause(s): dataloader stall / host-sync / fp32 path / tiny batch.
3. **Rung 2 — Nsight Systems.** Capture an `nsys` timeline with NVTX ranges over a few steps. Confirm the gap/overlap story and rank kernels. Note whether one kernel would justify `ncu` (climb to rung 3 only if so).
4. **Fix** the identified cause (autocast/bf16, dataloader workers+pin_memory, remove per-step sync, raise batch size).
5. **Re-measure** the same DCGM field and screenshot the before/after.

**Acceptance:** a one-page **before/after profiling round-trip note** for the blog: baseline metric shape → which rung → the specific trace finding (with the screenshot) → the fix (diff) → the **DCGM metric moving** (before/after value + graph). It must show `PIPE_TENSOR_ACTIVE` (or the relevant field) *change*, and it must justify *why you stopped where you did* on the ladder.

## Common pitfalls

1. **Reaching for Nsight Compute first because it's "the real profiler."** Most real production waste — including Meta's own case — is visible and fixable at rung 1. Climbing straight to rung 3 spends the 10–100× replay overhead on a question rung 1 could have answered for free.
2. **Under-profiling shared cloud infrastructure because of GUI/permission friction.** `nsys`/`ncu` traditionally assumed a local GUI and elevated counter-collection permissions; on shared cloud GPUs those are often unavailable or gated, which is a documented, practical reason teams skip profiling even when they know they should. Learn the CLI/headless capture path (`nsys profile ... -o file`, exported and viewed later) so friction isn't an excuse.
3. **Treating a Nsight Compute finding as "the fix" itself.** A warp-stall reason or occupancy limiter tells you *why* a kernel is slow, not what to change. The actual lever is usually upstream — a fused kernel, a precision change, a reshape — not hand-tuned PTX.
4. **Skipping a rung without evidence.** If the DCGM shape alone "feels" like a memory-bound problem, going straight to Nsight Compute without a rung-2 confirmation that one kernel dominates risks profiling the wrong kernel in exhaustive, expensive detail.
5. **Confusing `GPU_UTIL`/util% with productivity while reading any of these traces.** All three profiling rungs exist because util% cannot distinguish "occupied" from "productive" — that's the entire reason this lesson exists on top of L1.

## Self-check

- `SM_ACTIVE` is high but `PIPE_TENSOR_ACTIVE` is low. Which tool do you reach for next, and what are you looking for? **Answer:** Start with the **PyTorch Profiler** (rung 1) if it's a framework workload, escalating to **Nsight Systems** — *not* Nsight Compute yet. The shape means the SMs are busy but the tensor cores are idle, so you're looking for **wrong precision or op mix**: fp32 GEMMs (no `autocast`), tiny GEMMs that don't reach the tensor cores, or an elementwise/norm-dominated graph. You only reach Nsight Compute if the timeline then names one hot kernel you need to dissect.
- Nsight Systems vs Nsight Compute — when do you use each? **Answer:** **Nsight Systems** is the *system timeline* profiler: whole process, wall-clock axis, CPU threads + CUDA API + kernels + memcpys + NCCL. Use it to find **where the time goes** — gaps, CPU↔GPU overlap, host stalls, and *which kernel dominates*. **Nsight Compute** is the *single-kernel* profiler: it replays one kernel to collect the full counter set (roofline, occupancy limiters, warp-stall reasons). Use it only after Nsight Systems has isolated **one hot kernel**. Systems answers "which kernel / which phase"; Compute answers "why this kernel," at 10–100× overhead because it replays.
- Name a bottleneck DCGM metrics can't see but a profiler can. **Answer:** A **dataloader / host-input stall** — the GPU is idle because the CPU-side `DataLoader` isn't delivering batches. DCGM (1 Hz, device-level, no host visibility, no intra-step timeline) shows low utilization but cannot attribute it; the PyTorch Profiler or Nsight Systems timeline shows the GPU stream going empty at each step boundary while the CPU thread blocks in the loader. (Equally valid: a per-step `cudaStreamSynchronize`, kernel-launch/latency-bound tiny kernels, or a memory-bound kernel's warp-stall *reason*.)
- In Meta's production case, what tool found the GPU-idle/GPU-active pattern, and what tracing layer is it built on? **Answer:** Meta's internal **MAIProf**, built on **Kineto** (PyTorch's profiling library), which wraps NVIDIA's **`libcupti`** — the same CUDA tracing substrate that underlies public `torch.profiler`'s CUDA-side traces and, at a lower level, Nsight Systems' kernel timeline.
- Why does Nsight Compute have to *replay* a kernel instead of collecting all counters in one pass? **Answer:** Because the GPU has a **finite number of hardware performance-counter collection slots** — the same physical constraint from L2 that forces DCGM to multiplex PROF fields across GPUs. A full counter set for an arbitrary kernel can't be gathered live in one execution, so Nsight Compute deterministically re-runs the kernel once per counter group, which is what makes it 10–100× slower and unsuitable for fleet-wide or whole-loop use.

## Connections & what's next

This lesson is the action layer on top of every metric lesson before it: L1/L2 gave you the shape vocabulary (`SM_ACTIVE`/`TENSOR_ACTIVE`/`DRAM_ACTIVE`), L3/L4 gave you the per-pod label join to find *which* workload to profile, L5 gave you the sibling failure class (hardware faults you cordon, not profile), and L6 gave you the inference-specific shapes (queue-vs-decode) that also map onto this same ladder. The capstone (L8) is where this pays off directly: the deliverable's blog post explicitly wants an "and here's how you fix an inefficient GPU once you've found it" section, and this lesson's before/after round-trip — baseline shape, trace finding, fix, metric delta — is that section, verbatim.

## References & further reading

**Primary sources**
- PyTorch Profiler recipe (official) — the API for rung 1: schedule, trace export, `key_averages`. https://pytorch.org/tutorials/recipes/recipes/profiler_recipe.html
- NVIDIA Nsight Systems User Guide — timeline semantics, NVTX ranges, `nsys` capture flags for rung 2. https://docs.nvidia.com/nsight-systems/UserGuide/index.html

**Real-world engineering blogs**
- PyTorch (Meta) — "Performance Debugging of Production PyTorch Models at Meta" — the GPU-idle/GPU-active production case study this lesson's "Why this matters" is built on. https://pytorch.org/blog/performance-debugging-of-production-pytorch-models-at-meta/
- NVIDIA Developer Blog — "Using Nsight Compute to Inspect your Kernels" — a concrete rung-3 fix with real before/after numbers (87.5% / 68% reductions). https://developer.nvidia.com/blog/using-nsight-compute-to-inspect-your-kernels/

**Deeper dives**
- Spheron — "GPU Profiling for AI Workloads: Nsight Compute, Nsight Systems, and PyTorch Profiler" (2026) — the fullest practitioner map of the three-tool ladder and the "why teams under-profile shared GPUs" friction this lesson's pitfalls section draws on. https://www.spheron.network/blog/gpu-profiling-ai-workloads-nsight-compute-pytorch-profiler-guide/
