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
sources: 14
---

# 05.7 · Profiling escalation — from metrics to Nsight

> **Concept.** Metrics tell you *that* a GPU is inefficient; profiling tells you *why*. Read the DCGM shape, then climb the ladder — PyTorch Profiler → Nsight Systems → Nsight Compute — one rung at a time.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

05.1–05.5 taught you to read the fleet honestly (`SM_ACTIVE` vs `GPU_UTIL`, per-namespace attribution, XID triage), and 05.6 taught you the inference-specific split (TTFT vs TPOT vs ITL) with the roofline arithmetic behind it. Both leave you at the same wall: you now have a *finding* — "this GPU is busy but unproductive," "TPOT regressed with a flat queue" — but a finding is not a fix.

DCGM's device-level counters cannot see inside a process. There is **no call stack, no per-kernel attribution, no host-side view, and no resolution below the sampling interval.** This lesson is the bridge from "the metric says something is wrong" to "here is the line of code, the kernel, or the config that is wrong," and it hands the capstone (05.8) the one thing it is still missing: a documented before/after profiling round-trip that proves the honest metric can be **moved**, not just observed.

It is also the lesson where cost discipline becomes a technical skill rather than a slogan. Every rung on this ladder is more expensive than the one below it — in wall-clock, in GPU-hours, in blast radius, and in one case in *counter contention with the fleet telemetry you already run*. Knowing which rung, on which GPU, for how long, is the graded competency.

## Why this matters

Every senior platform interview that gets past "what does `GPU_UTIL` measure" moves to the follow-up: *"OK, it's inefficient. Why, and what do you do?"* Fleet telemetry cannot answer that by construction, and neither can a shrug.

Meta's engineering team published exactly this problem at production scale: a training job showing a repeating **GPU-idle / GPU-active / GPU-idle** pattern, with the GPU sitting idle **more than half the total training time** — found not with an exotic tool but with the same class of framework-level profiler this lesson opens with (Meta's internal MAIProf, built on **Kineto**, the same tracing layer PyTorch's public `torch.profiler` uses). That is the concrete cost of not knowing this skill: half your paid GPU-hours evaporating in a pattern that is invisible to DCGM and trivially visible to a profiler, on one of the best-resourced ML infra teams in the industry.

Put a number on it. A 64-GPU H100 training job at $2.50/GPU-hr costs **$3,840/day**. A 50% host-side idle fraction is **$1,920/day, $57,600/month**, burned on a bug that a 15-minute rung-1 profile finds and a four-line `DataLoader` change fixes. The profiling session costs about **$0.07** of GPU time. That ratio — five orders of magnitude — is why "we didn't have time to profile" is never a real answer.

The corollary interview trap is reaching for the heaviest tool first. Nsight Compute *replays* a kernel to collect counters, and platform engineers who "profile everything with everything" burn cluster time and credibility. The skill graded here is judgment.

## What's new here (calibration)

You already know how to read the DCGM PROF fields (05.1–05.2) and how they are labelled per pod (05.3–05.4). This lesson does not re-teach metric semantics; it starts from a metric *shape* as a given and asks what to do next. Also assumed: you can read a Python traceback and a shell profiler invocation.

What is genuinely new:

- **The escalation ladder as a decision procedure**, not a tool tour — which shape sends you to which rung, and the discipline of stopping as soon as the evidence answers the question.
- **The counter-contention problem.** DCGM's profiling module and Nsight's counter collection want the *same physical hardware counters*. Running a profiler on a node whose GPUs are being sampled by dcgm-exporter is not free — it is a resource conflict with observable symptoms. This is the interaction almost nobody teaches, and it is the reason the ladder's rungs cannot simply all be left on.
- **Framework-level tracing internals** — Kineto wrapping `libcupti`, the profiler's four-state schedule machine, and why "warmup" is a correctness requirement rather than politeness.
- **Replay mechanics** — *why* Nsight Compute has to re-run a kernel, how many times, and what that does to your wall-clock and your numbers.
- **Warp-stall-level diagnosis** — the stall-reason taxonomy, a level of hardware detail the fleet counters cannot express at all.
- **Profiling inside Kubernetes** — the permission model (`ERR_NVGPUCTRPERM`), the security-context implications, and how to get a trace off a pod that you do not want to grant `CAP_SYS_ADMIN` to permanently.

## Core concepts

### 1. The visibility matrix — what each layer structurally cannot see

Before the ladder, understand *why* it has rungs. Each tool observes a different layer of the stack at a different time resolution, and the gaps are structural, not fixable with configuration.

```
                     ┌──────────────────────────────────────────────────────────┐
                     │  what it observes                                        │
  ┌──────────────────┼──────────────────────────────────────────────────────────┤
  │ Python / model   │  torch.profiler  ✔ (op names, shapes, stacks, modules)   │
  │ code             │  nsys            ~ (only via NVTX ranges you added)      │
  │                  │  ncu             ✘                                       │
  │                  │  DCGM            ✘                                       │
  ├──────────────────┼──────────────────────────────────────────────────────────┤
  │ CPU threads,     │  torch.profiler  ✔ (CPU-side op time, Python stacks)     │
  │ dataloader,      │  nsys            ✔✔ (OS runtime, thread state, backtrace)│
  │ syscalls         │  ncu             ✘                                       │
  │                  │  DCGM            ✘  ← the host does not exist to DCGM    │
  ├──────────────────┼──────────────────────────────────────────────────────────┤
  │ CUDA API calls,  │  torch.profiler  ✔ (via CUPTI)                           │
  │ streams, memcpy, │  nsys            ✔✔ (full API trace, correlation IDs)    │
  │ NCCL collectives │  ncu             ~ (only around the profiled kernel)     │
  │                  │  DCGM            ✘                                       │
  ├──────────────────┼──────────────────────────────────────────────────────────┤
  │ kernel executions│  torch.profiler  ✔ (name, duration, correlation to op)   │
  │ on the timeline  │  nsys            ✔✔ (name, duration, grid/block, stream) │
  │                  │  ncu             ✔ (one kernel, exhaustively)            │
  │                  │  DCGM            ✘  ← no notion of a kernel at all       │
  ├──────────────────┼──────────────────────────────────────────────────────────┤
  │ inside a kernel: │  torch.profiler  ✘                                       │
  │ warp stalls,     │  nsys            ✘                                       │
  │ occupancy limits,│  ncu             ✔✔  ← ONLY here                         │
  │ roofline, memory │  DCGM            ✘                                       │
  │ hierarchy hits   │                                                          │
  ├──────────────────┼──────────────────────────────────────────────────────────┤
  │ device aggregate │  torch.profiler  ✘                                       │
  │ over ALL procs,  │  nsys            ✘  (one process)                        │
  │ continuously,    │  ncu             ✘  (one kernel)                         │
  │ across the fleet │  DCGM            ✔✔  ← ONLY here                         │
  └──────────────────┴──────────────────────────────────────────────────────────┘

     TIME RESOLUTION      DCGM ~1 s   │  torch/nsys ~µs  │  ncu: per-instruction
     SCOPE                whole fleet │  one process     │  one kernel launch
     COST                 ~0          │  1.05–2×         │  10–100× on that kernel
```

**The two "ONLY here" rows are the whole argument.** DCGM is the only thing that can watch 500 GPUs continuously; Nsight Compute is the only thing that can tell you why one kernel is slow. Everything in between is a trade of scope against depth, and the ladder is the ordered walk from one extreme to the other.

Concretely, DCGM is blind by construction to:

- **Dataloader / host-input stalls.** The GPU is idle *because the CPU did not deliver a batch*. DCGM shows low `SM_ACTIVE`; it cannot show that the cause is a `DataLoader` with `num_workers=0`.
- **CPU↔GPU synchronisation stalls.** A per-step `loss.item()` or `print(loss)` forces a `cudaStreamSynchronize` that serialises the pipeline. Invisible to a device counter; unmistakable on a timeline.
- **Kernel-launch / latency-bound execution.** Thousands of tiny kernels where launch overhead dominates. DCGM may even show *decent* `SM_ACTIVE` while wall-clock throughput is terrible, because "an SM had ≥1 warp resident" is true for a 3 µs kernel too.
- **Warp-stall reasons and roofline position.** DCGM gives you the symptom (`DRAM_ACTIVE` high), never the mechanism (`Stall Long Scoreboard`, memory-bound at 78% of the roof).

That list is the answer to "name a bottleneck metrics cannot see" — pick any one, and name the tool that *can* see it.

### 2. Rung 0 — the metric shape picks the target and the direction

You never profile blind. The DCGM shape from 05.1/05.2/05.6 tells you both which GPU and which rung. The four fields that matter (all `DCGM_FI_PROF_*`, the DCP profiling set — see 05.2 for why they multiplex and 05.3 for why `SM_ACTIVE` ships commented out):

| Field | Reads | Means |
|---|---|---|
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` (1001) | fraction of time the graphics/compute engine was active | "the engine had work." Close cousin of `GPU_UTIL`, still a duty cycle. |
| `DCGM_FI_PROF_SM_ACTIVE` (1002) | fraction of cycles an SM had ≥1 warp resident, averaged over SMs | "warps were scheduled." **Not** "useful math happened." |
| `DCGM_FI_PROF_SM_OCCUPANCY` (1003) | fraction of resident warp *slots* filled | how full the SMs were, not how busy. |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (1004) | fraction of cycles the tensor (HMMA) pipe was active | "am I doing the matmuls I bought this GPU for." |
| `DCGM_FI_PROF_DRAM_ACTIVE` (1005) | fraction of cycles the HBM interface was moving data | memory-bound indicator. |

**The primary diagnostic is the gap between `SM_ACTIVE` and `PIPE_TENSOR_ACTIVE`.** A GPU can report 0.95 `SM_ACTIVE` and 0.05 `PIPE_TENSOR_ACTIVE`: warps are resident and issuing, but they are fp32 CUDA-core ops, elementwise/norm/activation kernels, or GEMMs too small to reach the tensor cores. `SM_OCCUPANCY` refines it — low occupancy plus low tensor means launch/latency-bound; high occupancy plus low tensor means the wrong op mix.

Five canonical shapes, and where each one sends you:

| DCGM shape (15-min average) | Reading | First rung | Looking for |
|---|---|---|---|
| `SM_ACTIVE` high, `PIPE_TENSOR_ACTIVE` low, `DRAM_ACTIVE` moderate | SMs busy, tensor cores idle → wrong precision or op mix | **torch.profiler** | fp32 path (no `autocast`), tiny GEMMs, elementwise-dominated graph |
| `DRAM_ACTIVE` saturates *before* `SM_ACTIVE`/`TENSOR_ACTIVE` | memory-bandwidth bound; SMs starved on HBM | **Nsight Systems** to find the kernel, then **Nsight Compute** (roofline) | a kernel on the memory roof; fusion opportunities |
| `SM_ACTIVE` low **and oscillating** (0.8 / 0.02 / 0.8 …), `GPU_UTIL` flapping | GPU idle between bursts → host-side stall (Meta's case) | **torch.profiler** | dataloader gaps, `cudaStreamSynchronize`, H2D copies |
| `SM_ACTIVE` moderate and *flat*, throughput poor, many kernels | launch/latency bound | **Nsight Systems** | thousands of short kernels; gaps between launches; poor CPU↔GPU overlap |
| `SM_ACTIVE` high, `PIPE_TENSOR_ACTIVE` high, still slow | genuinely compute-bound; you are near the machine's limit | **Nsight Compute** (roofline) or *stop* | is there a better algorithm/kernel, or are you done? |

**A note on averaging that trips people up.** A 15-minute average of `SM_ACTIVE` at 0.45 is compatible with two completely different realities: steady 45% occupancy, or perfect 90% occupancy half the time and dead the other half. Before you pick a rung, look at the raw series at scrape resolution, not the average — the *oscillating* shape above is invisible in a 15-minute mean and is the single highest-yield finding in this lesson.

### 3. Rung 0.5 — on-demand DCGM profiling, and the counter-contention trap

Between "the always-on fleet metrics" and "attach a profiler" there is a rung people skip: **turn on more DCGM PROF fields, on one node, temporarily.** It is cheap, it needs no code change, and it often narrows the shape enough to skip a rung.

But it is not free, and the reason is the same physical constraint that governs Nsight Compute.

**The mechanism.** Streaming multiprocessors expose their performance counters through a fixed number of hardware collection slots. A DCGM PROF field is not one counter — it is derived from raw counters that live in specific counter banks, and a bank holds one counter-select register at a time. Two metrics that read the same bank **cannot both be programmed in one pass**. DCGM's **profiling module** (module ID 8) knows which groups can be collected together on the hardware in front of it, and if you ask for a combination that cannot, the API returns **`DCGM_ST_PROFILING_MULTI_PASS`** — "I would have to multiplex these across time slices, so each one gets a fraction of the sample window."

You can read the grouping directly:

```console
$ dcgmi profile -l
+------------------+----------+------------------------------------------------------+
| Group.Subgroup   | Field ID | Field Tag                                            |
+------------------+----------+------------------------------------------------------+
| A.1              | 1002     | sm_active                                            |
| A.1              | 1003     | sm_occupancy                                         |
| A.1              | 1004     | tensor_active                                        |
| A.1              | 1007     | fp32_active                                          |
| A.2              | 1006     | fp64_active                                          |
| A.3              | 1008     | fp16_active                                          |
| B.0              | 1005     | dram_active                                          |
| C.0              | 1009     | pcie_tx_bytes                                        |
| C.0              | 1010     | pcie_rx_bytes                                        |
| D.0              | 1001     | gr_engine_active                                     |
| E.0              | 1011     | nvlink_tx_bytes                                      |
| E.1              | 1012     | nvlink_rx_bytes                                      |
+------------------+----------+------------------------------------------------------+
```

*(Representative of the documented layout on a T4. **Run it on your own hardware** — the grouping is a property of that generation's counter-bank layout and changes between architectures. That variability is exactly why the command exists.)*

Read it as a co-residency map: **fields in the same subgroup share a pass; fields in different groups may need separate passes.** `sm_active` + `tensor_active` + `sm_occupancy` are all `A.1` on this part, so the three-field shape diagnosis in §2 is one pass. Add `dram_active` (`B.0`) and you have crossed a group boundary. DCGM will still give you both, by multiplexing — but each field is now sampled over a fraction of each interval, so short-lived behaviour gets smeared, and *that* is the cost you are paying, not CPU cycles.

**Now the contention trap, which is the part almost nobody tells you.** Those hardware counters are a *device-global* resource. `dcgm-exporter` scraping PROF fields on a node has them programmed. When you then launch Nsight Compute on the same GPU, both want to own the counter configuration. The symptoms are ugly and non-obvious:

- Nsight Compute failing to initialise counters, or reporting `ERR_NVGPUCTRPERM`-adjacent errors even when permissions are correct.
- Nsight Compute succeeding but DCGM's PROF fields going to `N/A`/blank for the duration — **your fleet dashboard develops a hole exactly on the node you are investigating**, which is how a profiling session turns into a false "the GPU went idle" alert.
- Numbers from either tool that are quietly wrong because the other tool changed the counter programming mid-flight.

**The operational rule: before a rung-2 or rung-3 profile, stop PROF collection on that node** — either scale the `dcgm-exporter` DaemonSet off that node (`nvidia.com/gpu.deploy.dcgm-exporter=false`, the surgical label from 04.1), or run a counters CSV without `DCGM_FI_PROF_*` fields. Put it in the runbook, and **annotate the resulting dashboard gap** so nobody pages on it. This is the "counter contention between rungs" that makes the ladder a ladder: you cannot simply leave every rung switched on.

### 4. Rung 1 — the framework profiler: is the GPU even the problem?

Start here for anything training-shaped, because **the most common cause of a wasted GPU is that the host is not feeding it.**

#### How it works internally

`torch.profiler` is a thin Python layer over **Kineto**, PyTorch's profiling library, which wraps NVIDIA's **CUPTI** (CUDA Profiling Tools Interface). CUPTI provides two things: an *activity API* that streams records of kernel launches, memcpys and API calls with timestamps and **correlation IDs**, and a *callback API* that fires on CUDA runtime entry/exit. Kineto consumes the activity stream; the Python-side profiler records `aten::` operator entry/exit itself. The two are stitched together by correlation ID, which is why the trace can say *"this `aten::mm` on the CPU at t=1.203 s launched `sm90_xmma_gemm_...` on the GPU at t=1.205 s."*

**That correlation is the entire value of rung 1**, and it is why rung 1 is cheap: you are reading a lightweight event stream the driver produces anyway, not replaying anything.

#### The schedule, and why `warmup` is mandatory

`torch.profiler` runs a four-state machine driven by `prof.step()`:

```
   skip_first │        repeat cycle 1              │  repeat cycle 2   │
  ────────────┼────────────────────────────────────┼───────────────────┤
              │  wait   │ warmup │      active     │  wait │ warmup │…
   steps:  0  │ 1       │ 2      │ 3    4    5     │ 6     │ 7      │
              │         │        │                 │
   tracing:  off        off      ON (buffers armed) │
                        ▲        ▲                  ▲
                        │        │                  └─ on_trace_ready(prof) fires
                        │        └─ records kept
                        └─ CUPTI initialised, buffers allocated, first-call
                           overhead paid HERE and discarded
```

`schedule(wait=1, warmup=1, active=3, repeat=1)` reads as: skip one step, spend one step warming up, record three, then stop.

**Warmup is not politeness.** The first profiled step pays CUPTI's lazy initialisation, buffer allocation, and per-context metric setup. Record it and that one-time cost lands inside your measurement, inflating the first step by a large and variable amount. PyTorch will literally warn you — *"Profiler won't be using warmup, this can skew profiler results"* — if you pass `warmup=0`. **A trace with `warmup=0` is a trace you cannot compare against anything.**

#### The invocation

```python
import torch
from torch.profiler import profile, ProfilerActivity, schedule, tensorboard_trace_handler

with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    schedule=schedule(skip_first=10, wait=1, warmup=1, active=3, repeat=1),
    on_trace_ready=tensorboard_trace_handler("./tb", worker_name="rank0"),
    record_shapes=True,     # input shapes per op — needed to spot tiny GEMMs
    profile_memory=True,    # allocator events
    with_stack=True,        # Python source file:line for each op  (costs CPU time)
    with_flops=True,        # FLOP estimate for matmul/conv — the productivity number
) as prof:
    for step, batch in enumerate(loader):
        train_step(batch)
        prof.step()          # <-- drives the state machine; forgetting this is
                             #     the #1 reason people get an empty trace
```

`skip_first=10` matters in practice: the first several steps of a real training loop include cuDNN autotuning, `torch.compile` compilation, allocator growth and NCCL warmup. Profile them and you will "discover" a bottleneck that exists only at startup.

#### Reading the output — the op table

```python
print(prof.key_averages(group_by_input_shape=True)
          .table(sort_by="self_cuda_time_total", row_limit=15))
```

```
-------------------------------  ------------  -----------  ------------  ------------  ---------------------------
Name                             Self CUDA %   Self CUDA    CUDA total    # of Calls    Input Shapes
-------------------------------  ------------  -----------  ------------  ------------  ---------------------------
aten::mm                              31.20%     412.3ms       412.3ms            288   [[8192, 4096],[4096,4096]]
aten::native_layer_norm               18.44%     243.7ms       243.7ms            192   [[8, 1024, 4096], ...]
aten::add                             11.02%     145.6ms       145.6ms           1152   [[8, 1024, 4096], ...]
aten::mul                              9.87%     130.4ms       130.4ms            960   [[8, 1024, 4096], ...]
aten::copy_                            8.31%     109.8ms       109.8ms            394   [[8, 1024, 4096], ...]
aten::_softmax                         6.55%      86.6ms        86.6ms            192   [[8, 32, 1024, 1024], ...]
-------------------------------  ------------  -----------  ------------  ------------  ---------------------------
Self CUDA time total: 1.321s
```

*(Representative shape of the output, not a captured transcript.)*

**How to read it in one pass.** Sum the "real math" rows (`aten::mm`, `addmm`, `bmm`, `_scaled_dot_product_*`, convolutions) against everything else. Here that is 31% math and **69% plumbing** — layer norms, elementwise add/mul, softmax, and 8% in `aten::copy_`, which is data movement pretending to be work. That distribution *is* the framework-level view of the low-`PIPE_TENSOR_ACTIVE` shape from §2, and it points at the fix (kernel fusion, `torch.compile`, a fused attention path) without any deeper tool.

Two more things the table tells you that a counter cannot:

- **`# of Calls` reveals launch-bound workloads.** 1,152 `aten::add` calls in three steps means 384 per step. If each is 15 µs of work with a ~5 µs launch, a third of that row is overhead.
- **`Input Shapes` reveals tiny GEMMs.** A `[[64, 128],[128, 128]]` matmul does not reach the tensor cores no matter what precision you set. `record_shapes=True` is what makes that visible.

And check precision directly: if you expected mixed precision but the kernel names are `sgemm`/`s16816gemm` fp32 variants with no `autocast` in the stack, that is your low-tensor cause, found without leaving Python.

#### Reading the output — the timeline

Export a Chrome/Perfetto trace and look at the *shape of a step*:

```python
prof.export_chrome_trace("trace.json")   # open in chrome://tracing or ui.perfetto.dev
```

Three patterns, three diagnoses:

```
  (a) HOST-INPUT STALL  ← the Meta pattern; the single most common real bug
  CPU  │███ dataloader ███│ fwd │ bwd │███ dataloader ███│ fwd │ bwd │
  GPU  │                  │████████████│                 │████████████│
       └── 40 ms of dead GPU ─┘         └── 40 ms of dead GPU ─┘
       Fix: num_workers, pin_memory, persistent_workers, prefetch_factor

  (b) HOST-SYNC STALL
  CPU  │ fwd │ bwd │ .item() ─── blocked ───│ fwd │ bwd │ .item() ───│
  GPU  │██████████│                          │██████████│
       └ launches stop while Python waits ┘
       Fix: remove per-step .item()/.cpu()/print(loss); accumulate on device

  (c) LAUNCH-BOUND
  CPU  │k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│k│
  GPU  │▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪ ▪│
       thousands of 3–20 µs kernels; gaps ≈ launch latency
       Fix: torch.compile / CUDA graphs / fusion — NOT a kernel rewrite

  (d) HEALTHY
  CPU  │ launch launch launch launch launch launch launch launch    │
  GPU  │████████████████████████████████████████████████████████████│
       CPU runs ahead, GPU never starves. Now the question is
       whether the kernels themselves are efficient → rung 2/3.
```

**Three of the four most common wasted-GPU causes — dataloader stall, host-sync stall, fp32 path — are visible here, at the cheapest rung.** Only a genuinely slow kernel forces you higher.

#### Scaling the read: Holistic Trace Analysis

A Kineto trace from a 64-rank job is tens of gigabytes of JSON and no human reads that. **Holistic Trace Analysis (HTA)** — open-sourced by Meta, built to consume exactly these traces — turns them into the numbers you actually want:

- **Temporal breakdown** — how each rank's GPU time splits into computation, communication, memory, and **idle**.
- **Idle-time breakdown** — the killer feature: idle time categorised into **host wait** (the CPU is not enqueuing kernels fast enough) versus **kernel wait** (the short overhead between consecutive launches) versus unknown. That single split distinguishes diagram (a) from diagram (c) *automatically, across every rank*, which by hand is a day of squinting.
- **Kernel breakdown** — longest-duration kernels per rank, which is your rung-3 candidate list.
- **Communication–computation overlap** — the percentage of time collectives overlap compute; the multi-GPU efficiency number.
- **CUDA kernel launch statistics** — the launch-bound detector.

If you profile distributed jobs at all, HTA is the difference between rung 1 being a per-job investigation and rung 1 being a routine report.

#### Memory, separately

Allocator behaviour is its own axis and has its own tool. `export_memory_timeline` is deprecated; the current path is:

```python
torch.cuda.memory._record_memory_history(max_entries=100_000)
train_step(batch)
torch.cuda.memory._export_memory_snapshot("snap.pickle")   # view at pytorch.org/memory_viz
```

Use it when the symptom is OOM-at-step-N, sawtooth memory, or fragmentation — not when the symptom is slowness.

### 5. Rung 2 — Nsight Systems: the whole-process timeline

Escalate to `nsys` when the framework trace is ambiguous, when it is not a PyTorch workload, or when the stall is *below* the framework — CUDA API, driver, NCCL, H2D DMA, or another process on the same GPU.

Nsight Systems captures a **whole-process, whole-timeline** view on one wall-clock axis: CPU threads and their OS-level states, CUDA runtime and driver calls, kernel executions, memcpys, NCCL collectives, cuDNN/cuBLAS calls, and your NVTX ranges.

#### The invocations, with the flags that matter

```bash
# Baseline: GPU timeline + CUDA/OS-runtime calls. LOW overhead — this is the
# one you run first and the one that most represents production behaviour.
nsys profile \
  -t cuda,nvtx,osrt,cudnn,cublas \    # trace domains
  -s none \                           # NO CPU sampling  ← the big overhead knob
  -w true \                           # show the app's stdout/stderr
  -f true \                           # overwrite existing report
  -x true \                           # stop profiling when the app exits
  -o step_trace \
  python train.py

# When you need to know WHICH Python line is blocking: add CPU sampling and
# backtraces. Measurably heavier — mcarilli's PyTorch profiling notes report
# ~2× or more slowdown from `-s cpu` alone.
nsys profile -t cuda,nvtx,osrt,cudnn,cublas -s cpu \
  --cudabacktrace=true --cudabacktrace-threshold=10000 \  # backtrace CUDA API calls >10 µs
  --osrt-threshold=10000 \                                # backtrace OS calls >10 µs
  -w true -f true -x true -o step_trace_bt python train.py

# Best practice for long jobs: profile ONLY the region you care about, so the
# report is megabytes instead of gigabytes and startup noise is excluded.
nsys profile -t cuda,nvtx,osrt,cudnn,cublas -s none \
  --capture-range=cudaProfilerApi --stop-on-range-end=true \
  -f true -x true -o step_trace_focused python train.py
```

…with the application cooperating:

```python
for step, batch in enumerate(loader):
    if step == 20:
        torch.cuda.cudart().cudaProfilerStart()      # capture-range opens here
    torch.cuda.nvtx.range_push(f"step_{step}")
    torch.cuda.nvtx.range_push("dataload");  batch = next_batch();  torch.cuda.nvtx.range_pop()
    torch.cuda.nvtx.range_push("forward");   out   = model(batch);  torch.cuda.nvtx.range_pop()
    torch.cuda.nvtx.range_push("backward");  out.backward();        torch.cuda.nvtx.range_pop()
    torch.cuda.nvtx.range_push("optimizer"); opt.step();            torch.cuda.nvtx.range_pop()
    torch.cuda.nvtx.range_pop()
    if step == 23:
        torch.cuda.cudart().cudaProfilerStop()       # --stop-on-range-end ends the run
```

**NVTX annotation is the difference between a readable trace and a wall of CUDA.** Without it the timeline is `sm90_xmma_gemm_bf16...` a thousand times. With it, you can see that "backward" is 62% of the step and "dataload" is 18%, and you have your answer before opening a single kernel row.

#### Reading it headlessly

You do not need the GUI, which matters enormously on a shared cluster:

```bash
# Kernels ranked by total GPU time — the rung-3 candidate list.
nsys stats --report cuda_gpu_kern_sum step_trace.nsys-rep

 Time(%)  Total Time(ns)  Instances   Avg(ns)   Med(ns)   Min(ns)   Max(ns)  Name
 -------  --------------  ---------  ---------  --------  --------  -------  ------------------------------
    41.7    2,140,882,304        288  7,433,619  7,401,112  7,102,400  8,930,112  sm90_xmma_gemm_bf16...
    19.3      991,204,352        192  5,162,522  5,140,224  5,010,944  5,882,368  void flash_fwd_kernel<...>
    11.8      605,884,416       1152    525,942    519,168    498,176    901,120  void at::native::vectorized_elementwise_kernel<...>
     8.2      421,036,032        394  1,068,619  1,041,920  1,001,472  2,110,464  void at::native::(anonymous)::CatArrayBatchedCopy<...>
 ...

# Memory transfers — is the H2D path the bottleneck?
nsys stats --report cuda_gpu_mem_time_sum step_trace.nsys-rep

# CUDA API calls ranked by time — is the CPU stuck in a synchronize?
nsys stats --report cuda_api_sum step_trace.nsys-rep

# NVTX range summary — YOUR phases, ranked.
nsys stats --report nvtx_sum step_trace.nsys-rep
```

*(Representative output structure; run it on your own trace.)*

#### The four questions only a timeline answers

1. **Where are the gaps?** Dead space on the GPU rows is idle GPU. Line it up against the CPU rows to see what the host was doing. This is the tool that finds Meta's idle/active/idle pattern directly, if rung 1 has not already.
2. **Is the CPU running ahead?** Kernel launches should be pipelined many iterations ahead of execution. If every launch sits immediately before its kernel with no queue, you are launch-bound or synchronising.
3. **Do collectives overlap compute?** In multi-GPU, whether `ncclAllReduce` runs concurrently with backward compute or serialises it. (Module 06's territory, but you see it first here.)
4. **Which kernel dominates?** The `cuda_gpu_kern_sum` ranking is the *gate to rung 3*, and the gate has a rule: **if one kernel is more than roughly 30–50% of GPU time, a single-kernel deep dive is worth it. If time is spread across hundreds of small kernels, the answer is fusion or launch overhead, not one kernel** — and no amount of Nsight Compute will tell you that, because Nsight Compute only ever looks at one kernel.

### 6. Rung 3 — Nsight Compute: one kernel, warp-stall level

Climb here **only** when rung 2 named one hot kernel and you need to know why *that kernel* is slow.

#### Why it has to replay — the mechanism

Same physical constraint as §3, taken to its conclusion. A full metric set is dozens of raw counters spread across counter banks that cannot all be programmed simultaneously. Nsight Compute's answer is to **group the requested metrics into the minimum number of passes the hardware allows, then re-execute the kernel once per pass**, saving and restoring the memory the kernel touches between passes so each replay sees identical inputs and is therefore deterministic.

```
   ONE KERNEL LAUNCH, as the application sees it
   ───────────────────────────────────────────────────────────────────
      launch ──▶ [ kernel executes, 7.4 ms ] ──▶ done


   THE SAME LAUNCH UNDER `ncu --set full`
   ───────────────────────────────────────────────────────────────────
      launch ──▶ ┌──────────────────────────────────────────────┐
                 │ ONE-TIME per context: build metric config    │  ← big, paid once
                 └──────────────────────────────────────────────┘
                 ┌───────────┐ save memory the kernel will write
                 │  PASS 1   │ program counter banks {a}, run kernel, read
                 └───────────┘ restore memory
                 ┌───────────┐
                 │  PASS 2   │ program banks {b}, run kernel, read, restore
                 └───────────┘
                        ⋮        (N passes; N grows with the metric set)
                 ┌───────────┐
                 │  PASS N   │
                 └───────────┘
                 + kernel launches SERIALISED (no concurrency during profiling)
                 + optionally clocks LOCKED to a fixed base for comparability
                                                  ──▶ done, much later

   COST ≈  one-time context setup
         + N × (kernel time + save/restore of touched memory)
         + loss of concurrency across the whole app

   ⇒ a 7.4 ms kernel with a full metric set is easily seconds of wall clock.
     Multiply by every launch you did not filter out. THIS is why you scope it.
```

Two consequences to internalise:

- **The one-time setup is per CUDA context, not per kernel.** NVIDIA documents a relatively high one-time overhead for the first profiled kernel in each context to generate the metric configuration, which does *not* recur for later kernels in the same context if the metric list is unchanged. So profiling 1 kernel and profiling 20 kernels of the same shape are not 20× apart in cost — but profiling across many contexts is expensive again.
- **Serialisation changes what you are measuring.** Under default (kernel) replay, kernels do not run concurrently. If your workload's performance depends on kernel overlap, the profiled timing is *not* your production timing. That is what the other replay modes exist for:

| Replay mode | What it replays | Metrics attributed to | Use when |
|---|---|---|---|
| **kernel** (default) | one kernel, N times | that kernel | normal single-kernel analysis |
| **application** | the whole application, N times | individual kernels | the kernel's behaviour depends on prior application state that save/restore cannot reproduce |
| **range** | a captured range of CUDA API calls + launches | **the entire range**, not individual kernels | kernels must run *concurrently* for correctness or for the performance you are studying |
| **application range** | a range, by re-running the whole application without memory save/restore | the entire range | same as range, but the range cannot be captured/replayed in-process |
| **graph** | a CUDA graph as one workload entity | the graph | `torch.compile` / CUDA-graph-captured workloads, where per-node profiling is meaningless |

**The graph mode matters more every year**: as `torch.compile` and CUDA graphs become the default for inference and increasingly for training, "profile the kernel" stops being the right unit and "profile the graph" starts being it.

#### The invocation — scope it hard

```bash
ncu \
  --set full \                                  # the metric set (see below)
  --target-processes all \
  -k regex:"flash_fwd|sm90_xmma_gemm" \         # ONLY these kernels
  --launch-skip 200 \                           # skip warmup/autotune launches
  --launch-count 3 \                            # profile 3 launches, then stop
  --nvtx --nvtx-include "step_21/backward/" \   # restrict to one NVTX range
  --import-source yes \                         # embed source for SourceCounters
  -o kern_report \                              # writes kern_report.ncu-rep
  python train.py

# Read it headlessly — no GUI needed:
ncu --import kern_report.ncu-rep --page details
ncu --import kern_report.ncu-rep --csv --page raw > metrics.csv
```

**`--launch-skip` and `--launch-count` are not optional in practice.** Without them, `ncu` profiles *every* matching launch, and a training loop that launches your target kernel 300 times per step with a full metric set will not finish this week.

The `--set` levels trade breadth for passes (run `ncu --list-sets` on your version — the exact set names drift):

| Set | Roughly | Passes | When |
|---|---|---|---|
| `basic` | Speed-of-Light only | fewest | quick "is it compute or memory bound?" |
| `default` | SoL + Launch + Occupancy + a few | few | the normal starting point |
| `detailed` | adds memory workload, scheduler, warp state, instruction stats | more | when you need the *why* |
| `full` | everything, including source counters | most | the deep dive; expect the biggest slowdown |
| `roofline` | the roofline chart sections | few | answering exactly one question cheaply |

Or skip sets and name sections directly — `--section SpeedOfLight --section MemoryWorkloadAnalysis --section WarpStateStats` — which is how you keep pass count down when you know what you are looking for. Useful section names: `SpeedOfLight`, `SpeedOfLight_RooflineChart`, `ComputeWorkloadAnalysis`, `MemoryWorkloadAnalysis` (+ `_Chart`, `_Tables`), `SchedulerStats`, `WarpStateStats`, `Occupancy`, `LaunchStats`, `InstructionStats`, `SourceCounters`.

#### What only Nsight Compute sees

**(a) Roofline position.** Where this kernel sits against the compute roof and the memory roof — the same roofline you derived arithmetically in 05.6, now measured for one kernel. It answers "is there headroom, and in which direction," and it tells you the ceiling you are fighting. A kernel at 92% of the memory roof is *done*; no amount of tuning inside it will help, and the fix is algorithmic (fuse, re-use, change layout).

**(b) Warp stall reasons.** The SM's scheduler samples each warp's PC and stall state, so you get the *distribution of why warps were not issuing*. This is the deepest "why" available, and nothing below this rung can produce it:

| Stall reason | Warps are waiting on | Typical fix |
|---|---|---|
| **Long Scoreboard** | a global/local memory (L2 or HBM) load result — memory latency | improve coalescing/locality, increase ILP, prefetch, more occupancy to hide latency |
| **Short Scoreboard** | shared-memory access results, or long-latency math (MUFU: `exp`, `rsqrt`) | fix shared-memory bank conflicts; reduce special-function traffic |
| **MIO Throttle** | the memory-IO instruction queue is full (local/global/shared/constant loads, decoupled math) | reduce instruction *rate* into MIO — vectorise loads, cut redundant shared traffic |
| **LG / Tex Throttle** | local-global or texture pipe queue full | same idea, different pipe |
| **Math Pipe Throttle** | the arithmetic pipe is saturated | you are compute-bound. Good news, usually. |
| **Barrier** | `__syncthreads()` — some threads arrived early | rebalance work per thread; reduce sync points |
| **Not Selected** | the warp was *eligible* but another was chosen | **this is healthy** — it means you have surplus parallelism |
| **Wait** | a fixed instruction dependency latency | more ILP |
| **No Instruction** | instruction-cache miss / fetch | very large or badly laid-out kernels; unrolling gone too far |
| **Drain / Dispatch** | pipeline drain at exit, or dispatch conflicts | usually a symptom of something above |

**Read `Not Selected` as the good stall.** A profile dominated by `Not Selected` means the SM has more eligible warps than it can issue — the machine is saturated. A profile dominated by `Long Scoreboard` means the SM is idle waiting for HBM, which is the memory-bound diagnosis at instruction granularity, and it is what `DRAM_ACTIVE` was gesturing at from a mile away.

**(c) Occupancy limiters.** Whether registers per thread, shared memory per block, or block size is capping achieved occupancy — and by how much. That is a directly actionable knob (`__launch_bounds__`, smaller tiles, fewer live registers), and it is the difference between "occupancy is low" and "occupancy is capped at 37.5% because you use 96 registers per thread."

**(d) Source-level attribution.** With `--import-source yes` and `-lineinfo` at compile time, stalls attach to source lines. For platform work this usually confirms *which library call* to change rather than driving a hand-written kernel edit.

#### What you actually do with a rung-3 finding

For platform engineering, the fix is almost never "rewrite the kernel." It is: switch to a fused / flash-attention kernel, change precision so the GEMM lands on the tensor cores, reshape so a library GEMM is used instead of a fallback, enable `torch.compile` so elementwise chains fuse, or bump a block/tile parameter that a library exposes. **You use Nsight Compute to prove which upstream change will pay off**, and that proof is what makes the change worth a deploy.

NVIDIA's own "Using Nsight Compute to Inspect your Kernels" walkthrough documents the magnitude available: profiling and optimising a single kernel produced an **87.5% reduction in memory transactions** and a **68% reduction in kernel execution duration**. That is the scale of win the replay overhead buys — on the *one* kernel it is worth paying for.

### 7. Profiling inside Kubernetes — the permission problem

This is where the ladder meets reality on a shared cluster, and it is the most common reason teams silently skip rungs 2 and 3.

**GPU performance counters are privileged.** The NVIDIA kernel module parameter `NVreg_RestrictProfilingToAdminUsers` gates access:

- `NVreg_RestrictProfilingToAdminUsers=1` (the default on many distributions) — only root, or a process with **`CAP_SYS_ADMIN`** (or `CAP_PERFMON`), may program the counters. A normal container gets **`ERR_NVGPUCTRPERM: Permission issue with Performance Counters`** from `ncu`.
- `NVreg_RestrictProfilingToAdminUsers=0` — any user may profile. Set via a `.conf` file in `/etc/modprobe.d`, and it needs a module reload or reboot, plus an initrd rebuild on some distributions (`dracut --regenerate-all -f` on RHEL-family, `update-initramfs -u -k all` on Debian-family).

That leaves you three options on a cluster, with real trade-offs:

| Option | How | Trade-off |
|---|---|---|
| Relax the module parameter fleet-wide | `options nvidia NVreg_RestrictProfilingToAdminUsers=0` in `/etc/modprobe.d`, reboot | Any tenant can read GPU counters. On a multi-tenant cluster that is a genuine side-channel consideration — counters are shared silicon state. |
| Relax it on a **profiling node pool** only | Same, but on a taint-isolated pool you schedule profiling jobs onto | The pragmatic answer. Blast radius bounded; profiling becomes "reschedule the job onto the profiling pool." |
| Grant `CAP_SYS_ADMIN` per profiling pod | `securityContext.capabilities.add: ["SYS_ADMIN"]` on a purpose-built pod | Narrow in scope but a large capability. Gate it behind an admission policy and make it short-lived. |

A workable profiling-pod shape:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: profile-run
  annotations:
    # Announce the dashboard gap you are about to create (§3).
    observability.internal/dcgm-prof-suspended: "gpu-node-17"
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: gpu-node-17
  containers:
  - name: trainer
    image: registry.internal/trainer:profiling      # includes nsys/ncu
    command: ["nsys","profile","-t","cuda,nvtx,osrt,cudnn,cublas","-s","none",
              "-f","true","-x","true","-o","/traces/step_trace","python","train.py"]
    securityContext:
      capabilities:
        add: ["SYS_ADMIN"]        # required for ncu; nsys kernel-trace also benefits
    resources:
      limits:
        nvidia.com/gpu: 1
    volumeMounts:
    - { name: traces, mountPath: /traces }
    - { name: dshm,   mountPath: /dev/shm }         # dataloader workers need this
  volumes:
  - name: traces
    persistentVolumeClaim: { claimName: profiling-traces }   # traces are GBs
  - name: dshm
    emptyDir: { medium: Memory, sizeLimit: 8Gi }
```

Three practical notes that save an afternoon each:

- **Traces are large.** A few steps of a multi-GPU `nsys` capture is easily gigabytes. Write to a PVC or object store, not the container filesystem, and use `--capture-range` so you capture three steps rather than three hours.
- **Learn the headless path.** `nsys stats --report …` and `ncu --import … --page details` mean you never need a GUI on the cluster. Copy the `.nsys-rep` / `.ncu-rep` down and open it locally if you want the UI. GUI friction is a documented reason teams under-profile shared GPUs; remove the excuse.
- **Do not leave `CAP_SYS_ADMIN` in the production manifest.** Profiling is a separate, short-lived deployment of the same image.

### 8. The decision procedure

Now the ladder, with the costs and the contention marked. **This is the diagram to be able to redraw from memory.**

```
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  RUNG 0 · DCGM fleet metrics                         ALWAYS ON                │
  │  scope: every GPU, continuously      resolution: ~1 s, device aggregate       │
  │  cost:  ~0 (already running)         blast radius: none                       │
  │  answers: THAT it is bad, and roughly which way                               │
  └──────────────────────────────────┬────────────────────────────────────────────┘
                                     │ shape says "investigate this GPU"
                                     ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  RUNG 0.5 · on-demand DCGM PROF fields                ON DEMAND, ONE NODE     │
  │  scope: one node's GPUs              resolution: ~1 s, more fields            │
  │  cost:  multiplexing → each field sampled over a fraction of the window       │
  │  blast radius: that node's telemetry fidelity                                 │
  │  answers: refines the shape (occupancy vs tensor vs dram)                     │
  │  ⚠ dcgmi profile -l tells you which fields share a pass on THIS silicon       │
  └──────────────────────────────────┬────────────────────────────────────────────┘
                                     │ still ambiguous, or it's a host-side shape
                                     ▼
  ╔═══════════════════════════════════════════════════════════════════════════════╗
  ║  ⚡ COUNTER CONTENTION BOUNDARY ⚡                                              ║
  ║  Below this line: DCGM owns the HW performance counters.                      ║
  ║  Above this line: Nsight wants them. THEY CONFLICT.                           ║
  ║  → Disable DCGM_FI_PROF_* on this node (or evict dcgm-exporter from it)       ║
  ║    before climbing. Expect — and ANNOTATE — a hole in the fleet dashboard.    ║
  ╚═══════════════════════════════════════════════════════════════════════════════╝
                                     │
                                     ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  RUNG 1 · torch.profiler (Kineto → CUPTI)             ONE JOB, A FEW STEPS    │
  │  scope: one process              resolution: µs, per-op, per-kernel, +host    │
  │  cost:  ~1.05–1.3× on the profiled steps (with_stack costs most)              │
  │  blast radius: that job's 3–5 steps                                           │
  │  answers: WHICH Python line / aten op; dataloader & sync stalls; precision    │
  │  ➜ ~3 of the 4 most common real bugs are FIXED HERE. Try to stop here.        │
  └──────────────────────────────────┬────────────────────────────────────────────┘
                                     │ ambiguous, non-PyTorch, or below the framework
                                     ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  RUNG 2 · Nsight Systems (nsys)                       ONE PROCESS, ONE RUN    │
  │  scope: whole process + OS       resolution: µs, full timeline                │
  │  cost:  low with `-s none`; ~2×+ with `-s cpu` sampling & backtraces          │
  │  blast radius: that process; GB-scale trace files                             │
  │  answers: WHERE the wall-clock goes — gaps, overlap, NCCL, memcpy, which      │
  │           kernel dominates                                                     │
  │  ➜ GATE TO RUNG 3: does ONE kernel exceed ~30–50% of GPU time?                │
  │        NO  → the answer is fusion / launch overhead / host. STOP.             │
  │        YES → climb.                                                            │
  └──────────────────────────────────┬────────────────────────────────────────────┘
                                     │ one kernel dominates
                                     ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  RUNG 3 · Nsight Compute (ncu)                        ONE KERNEL, REPLAYED    │
  │  scope: one kernel launch        resolution: per-instruction, per-warp        │
  │  cost:  N replay passes × kernel time + per-context setup + SERIALISATION;    │
  │         one to two orders of magnitude on the profiled kernel is normal       │
  │  blast radius: the profiled process is effectively stopped; needs CAP_SYS_ADMIN│
  │  answers: WHY this kernel is slow — roofline, warp stalls, occupancy limiters │
  │  ➜ the finding is a DIAGNOSIS, not a fix. The fix is usually upstream.        │
  └───────────────────────────────────────────────────────────────────────────────┘

   ── never skip a rung upward without evidence ───────────────────────────────▶
   ◀── never climb a rung you don't need ──────────────────────────────────────
```

**The stopping rule is the skill.** If the rung-1 trace already shows a 40 ms dataloader gap every step, you have your answer — fix the dataloader, do not open Nsight Compute. Spheron's practitioner guide makes the same point from the other side: most LLM inference slowdowns come from a handful of kernels that a short profile would identify. The ladder is not slow; *skipping to the top of it* is what is slow.

### 9. The cost of a profiling escalation, across a fleet

Put numbers on the ladder, because "expensive" is not an engineering statement. Fleet: 500 H100s at **$2.50/GPU-hr** (dated 2026 rate-card snapshot — verify yours).

```
  ── RUNG 0: always-on DCGM ────────────────────────────────────────────────
  Marginal cost of the PROF sampler: sub-1% of GPU time (05.2).
      500 GPUs × 24 h × 1% × $2.50 = $300/day fleet-wide
  This is the price of having ANY honest metric. Do not economise here.

  ── RUNG 1: torch.profiler, one job ───────────────────────────────────────
  5 steps × 0.9 s/step × 1.2× overhead = 5.4 s of GPU time on 1 GPU
      1 GPU × 5.4 s ÷ 3600 × $2.50            = $0.004
  Even at 64 ranks (profile rank 0 only in practice):
      64 × 5.4 s ÷ 3600 × $2.50               = $0.24
  ⇒ effectively free. There is NO cost argument for not doing this.

  ── RUNG 2: nsys, one job, 3 steps ────────────────────────────────────────
  Baseline `-s none`: 3 steps × 0.9 s × 1.1×   ≈ 3 s
  With `-s cpu` backtraces: 3 steps × 0.9 s × 2.2× ≈ 6 s
  Plus job restart to attach: ~120 s of 64 idle GPUs   ← THE REAL COST
      64 × 120 s ÷ 3600 × $2.50               = $5.33
  ⇒ the profiling is free; the RESTART is what you pay for. Batch your
    questions into one capture.

  ── RUNG 3: ncu, one kernel, `--set full`, 3 launches ─────────────────────
  Kernel 7.4 ms × ~20 passes × 3 launches      ≈ 0.44 s of kernel work
  + per-context metric-config setup            ≈ 10–60 s
  + serialisation of everything else in the run
  Realistic single-GPU session, end to end:    ≈ 10 min
      1 × 600 s ÷ 3600 × $2.50                = $0.42
  ⇒ ALSO cheap — on ONE GPU.

  ── THE FAILURE MODE: rung 3 without the rung-2 gate ──────────────────────
  "Profile everything with ncu" on a 64-GPU job:
      restart + attach                            120 s
      full-set profiling of 40 distinct kernels
        × per-context setup + N passes each     ≈ 25 min of wall clock
      64 GPUs held idle for the whole session (serialised, no useful work)
      64 × (120 + 1500) s ÷ 3600 × $2.50      = $72.00   per attempt
  Do that on three jobs a week for a quarter:
      3 × 13 × $72                             = $2,808
  …to answer a question rung 1 would have answered for $0.004.

  ── AND THE OTHER SIDE OF THE LEDGER ──────────────────────────────────────
  The bug you are chasing (Meta's case: >50% host-side idle) on that 64-GPU job:
      64 × 24 h × $2.50 × 0.50                = $1,920/day
                                              = $57,600/month
  ⇒ RATIO: a $0.004 rung-1 profile against $57,600/month of waste.
    Five orders of magnitude. The escalation discipline is not about
    saving profiling cost — profiling is cheap. It is about not turning a
    5-minute question into a 25-minute cluster-wide stall, AND about
    actually doing it instead of guessing.
```

**The number to remember for interviews:** rung 1 costs less than a cent and finds three of the four most common causes; rung 3 done wrong costs $72 a shot and finds nothing you did not already have. **The expensive mistake is not the tool, it is the missing gate.**

## Perspectives

**Developer.** From inside the training script you never see `SM_ACTIVE`; you see step time and, if you look, a profiler trace. Most of the common wasted-GPU causes — dataloader stalls, host syncs, fp32 paths, launch-bound elementwise chains — resolve entirely at rung 1 without ever touching Nsight. A developer does not need Nsight literacy to fix the majority of real bugs; they need profiler literacy and the discipline to look before guessing. Give them `torch.profiler` boilerplate in the project template and most of this lesson never reaches you.

**Operator / platform.** The escalation discipline — climb only as far as the evidence demands — is what separates a platform engineer who can *unblock* a stuck team from one who can only report "the GPU looks inefficient" and hand the problem back. Two operator-specific responsibilities nobody else will take: owning the **counter-contention runbook** (disable PROF collection before profiling, annotate the dashboard gap) and owning the **permission model** (a taint-isolated profiling node pool beats granting `CAP_SYS_ADMIN` cluster-wide).

**Hardware.** Nsight Compute's replay is not a UI choice; it is forced by silicon. The SM's counter banks hold one counter-select configuration at a time, so a full metric set cannot be gathered in one execution — the same constraint that makes DCGM return `DCGM_ST_PROFILING_MULTI_PASS` and makes `dcgmi profile -l`'s groupings architecture-dependent. **Profiling cost is a hardware fact you are paying for, not a software inefficiency somebody could patch away** — and the fact that both tools want the same banks is why they conflict.

**Cost / failure-mode.** Meta's production case is the sobering data point: a real training job burned more than half its wall-clock idle, on infrastructure run by one of the best-funded ML platform teams anywhere, found with a rung-1-class tool. The lesson is not "big companies have bugs too." It is that this failure mode is common enough, and cheap enough to detect, that **not profiling routinely is itself the anomaly**.

## Real-world use cases

- **PyTorch (Meta) — "Performance Debugging of Production PyTorch Models at Meta"** — https://pytorch.org/blog/performance-debugging-of-production-pytorch-models-at-meta/. A production PyTorch training job showed a repeating GPU-idle / GPU-active / GPU-idle pattern with the GPU idle **more than half** the training time; Meta's internal MAIProf — built on Kineto/`libcupti`, the same layer under public `torch.profiler` — found it by inspecting CPU and GPU timelines side by side. **What it shows:** the exact rung-1 methodology this lesson teaches, applied at hyperscale, finding a bug worth (at this lesson's rates) tens of thousands of dollars a month on a single job.

- **Meta / `facebookresearch/HolisticTraceAnalysis`** — https://github.com/facebookresearch/HolisticTraceAnalysis. Open-source library that consumes Kineto traces and produces a **temporal breakdown** (compute / communication / memory / idle per rank), an **idle-time breakdown** that separates *host wait* (CPU not enqueuing fast enough) from *kernel wait* (inter-launch overhead), a kernel-duration breakdown, communication–computation overlap, and CUDA launch statistics. **What it shows:** rung 1 scales to distributed jobs only if something automates the read — and the host-wait/kernel-wait split is precisely the diagram-(a)-vs-diagram-(c) distinction this lesson teaches you to make by eye, computed for every rank at once.

- **NVIDIA Developer Blog — "Using Nsight Compute to Inspect your Kernels"** — https://developer.nvidia.com/blog/using-nsight-compute-to-inspect-your-kernels/. A worked case where profiling a single kernel drove an **87.5% reduction in memory transactions** and a **68% reduction in kernel execution duration**. **What it shows:** a concrete, numeric rung-3 result — proof that the replay overhead buys a large fix, not a marginal one, when spent on the *one* kernel that deserves it.

- **NVIDIA — "ERR_NVGPUCTRPERM: Permission issue with Performance Counters"** — https://developer.nvidia.com/nvidia-development-tools-solutions-err_nvgpuctrperm-permission-issue-performance-counters. The official statement of the permission model: `NVreg_RestrictProfilingToAdminUsers=1` restricts counter access to root / `CAP_SYS_ADMIN` / `CAP_PERFMON`; setting it to `0` in `/etc/modprobe.d` opens it, with an initrd rebuild required on RHEL-family (`dracut --regenerate-all -f`) and Debian-family (`update-initramfs -u -k all`) systems. **What it shows:** the concrete, checkable reason profiling fails inside an ordinary Kubernetes pod — and the three deployment options in §7 that follow from it.

- **`mcarilli` — "Favorite nsight systems profiling commands for PyTorch scripts"** — https://gist.github.com/mcarilli/376821aa1a7182dfcf59928a7cde3223. Concrete `nsys profile` invocations from a PyTorch core developer, including the `-s cpu` vs `-s none` distinction (CPU sampling observed at **~2× or more** overhead, so `-s none` better represents production behaviour), `--cudabacktrace-threshold=10000` to backtrace only CUDA API calls over 10 µs, and the `--capture-range=cudaProfilerApi --stop-on-range-end=true` pattern paired with `cudaProfilerStart()`/`cudaProfilerStop()` in the training loop. **What it shows:** the practical flag set §5 is built on, and the empirical overhead figure that makes `-s none` the default choice.

## Worked example

Take the workload flagged in 05.1 and re-flagged in 05.6: `team-research/llama-sft-0` on `gpu-node-17`, GPU 3 (H100 80GB). Fleet dashboard, 15-minute averages:

```
DCGM_FI_DEV_GPU_UTIL                        98 %
DCGM_FI_PROF_GR_ENGINE_ACTIVE             0.93
DCGM_FI_PROF_SM_ACTIVE                    0.89
DCGM_FI_PROF_SM_OCCUPANCY                 0.41
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE           0.06     ← the finding
DCGM_FI_PROF_DRAM_ACTIVE                  0.34
```

### Step 1 — read the shape, form a hypothesis, pick a rung

High `SM_ACTIVE`, near-zero tensor activity, `DRAM_ACTIVE` only moderate. Not memory-bound (DRAM is not saturated). Not idle (SM is high). Moderate occupancy. Per §2 this is the **wrong precision / wrong op mix** shape → **start at rung 1**, not rung 3.

But first, look at the raw series rather than the average:

```promql
DCGM_FI_PROF_SM_ACTIVE{Hostname="gpu-node-17",gpu="3"}[30m]
```

It oscillates: ~0.95 for roughly 2.5 s, then ~0.05 for roughly 0.4 s, repeating. **The 0.89 average concealed a 14% duty-cycle hole.** Two findings for the price of one look, and neither would have survived the 15-minute mean.

### Step 2 — clear the counter-contention boundary

```bash
# Evict dcgm-exporter from this node so it stops owning the PROF counters.
kubectl label node gpu-node-17 nvidia.com/gpu.deploy.dcgm-exporter=false --overwrite
kubectl -n gpu-operator get pods -o wide | grep gpu-node-17    # confirm it's gone

# Silence and annotate, so nobody pages on the hole you just created.
amtool silence add Hostname=gpu-node-17 \
  --duration=2h --comment="profiling session — PROF fields suspended (lesson 05.7 §3)"
```

### Step 3 — rung 1: torch.profiler

Wrap three steps with the schedule from §4, then read the table and the timeline.

**Op ranking** (`sort_by="self_cuda_time_total"`):

```
Name                             Self CUDA %   Self CUDA    # Calls   Input Shapes
-------------------------------  -----------  -----------  --------  --------------------------
aten::addmm                          28.7%       318.4ms        288  [[4096],[8192,4096],[4096,4096]]
aten::native_layer_norm              21.3%       236.4ms        192  [[8,1024,4096], ...]
aten::mul                            14.9%       165.3ms        960  [[8,1024,4096], ...]
aten::add                            12.1%       134.2ms       1152  [[8,1024,4096], ...]
aten::_softmax                        9.8%       108.7ms        192  [[8,32,1024,1024], ...]
aten::copy_                           7.4%        82.1ms        394  [[8,1024,4096], ...]
```

Kernel names under `aten::addmm` are `sm80_xmma_sgemm_f32...` — **fp32 GEMMs.** No `autocast` anywhere in the recorded stacks. That is the low-`PIPE_TENSOR_ACTIVE` cause, found in Python, in under a minute of reading.

And note the second-order finding: only 28.7% of GPU time is in the GEMM at all. **Over 60% is layer norm, elementwise mul/add, softmax and copies** — a fusion problem sitting behind the precision problem.

**Timeline**: at the start of every step, a ~400 ms window where the GPU stream is completely empty while the CPU thread sits in `DataLoader.__next__`. That is diagram (a) from §4, and it is exactly the 0.4 s of the oscillation you spotted in step 1. **The metric shape and the trace agree**, which is how you know you are looking at the real thing.

### Step 4 — rung 2: confirm, and decide whether to climb

```bash
nsys profile -t cuda,nvtx,osrt,cudnn,cublas -s cpu \
  --capture-range=cudaProfilerApi --stop-on-range-end=true \
  --osrt-threshold=10000 -f true -x true -o step_trace python train.py

nsys stats --report nvtx_sum step_trace.nsys-rep
```

NVTX summary: `dataload` 14.2% of wall clock, `forward` 31.1%, `backward` 48.8%, `optimizer` 5.9%. The OS-runtime rows show the dataloader gap lines up with a single worker thread blocked in `read`/`futex` — a `num_workers=0`/`1` loader.

`nsys stats --report cuda_gpu_kern_sum` shows the top kernel at **28%** of GPU time, with the rest spread across a long tail of elementwise kernels.

**Apply the gate: no single kernel exceeds ~30–50%. Do not climb to rung 3.** The evidence already names the fix, and a rung-3 session here would cost $72 (§9) to tell you something you cannot act on differently.

### Step 5 — fix

```python
# (1) Precision: put the GEMMs on the tensor cores.
with torch.autocast("cuda", dtype=torch.bfloat16):
    out  = model(batch)
    loss = criterion(out, target)

# (2) Input pipeline: stop starving the GPU.
loader = DataLoader(
    ds, batch_size=bs,
    num_workers=8,            # was 0
    pin_memory=True,          # page-locked staging → async H2D
    persistent_workers=True,  # don't respawn workers every epoch
    prefetch_factor=4,        # queue batches ahead
)

# (3) Fusion: collapse the layer-norm/elementwise tail.
model = torch.compile(model)
```

### Step 6 — re-measure, and prove the metric moved

Re-enable PROF collection, remove the silence, and read the same fields after a soak:

```bash
kubectl label node gpu-node-17 nvidia.com/gpu.deploy.dcgm-exporter=true --overwrite
```

| Metric | Before | After | Why |
|---|---|---|---|
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | 0.06 | **0.55** | bf16 GEMMs now hit the HMMA pipes |
| `DCGM_FI_PROF_SM_ACTIVE` | 0.89 (oscillating) | **0.94 (steady)** | dataloader no longer starves the GPU |
| `DCGM_FI_PROF_SM_ACTIVE` duty-cycle hole | 14% | **<1%** | prefetch + workers |
| `DCGM_FI_DEV_GPU_UTIL` | 98% | **99%** | ← **unchanged, and therefore useless** |
| step time | 3.1 s | **1.4 s** | 2.2× |
| util-lie detector (`GPU_UTIL>90 AND SM_ACTIVE<0.2`) | — | no longer relevant | the honest metrics moved; the lying one never did |

**That last row is the punchline the deliverable wants.** `GPU_UTIL` went from 98% to 99% across a change that made the job **2.2× faster** and increased useful tensor work **9×**. The metric everyone dashboards did not move. The metrics you learned to read moved a lot.

### Step 7 — the money

```
  64-GPU job, $2.50/GPU-hr, 2.2× step-time improvement.
  Same total work now takes 1/2.2 of the wall clock:
      before: 64 × 24 h × $2.50            = $3,840/day
      after:  64 × (24/2.2) h × $2.50      = $1,745/day
      saved                                = $2,095/day  ≈ $63k/month

  Cost of finding it:
      rung 1 (5 steps, 1 GPU)              = $0.004
      rung 2 (3 steps + one job restart)   = $5.33
      engineer time                        = ~2 hours
      total GPU cost                       ≈ $5.33
```

**A $5.33 investigation against $63,000/month.** Write that ratio into the round-trip note — it is the sentence that gets profiling made routine instead of exceptional.

### Contrast: when you *do* climb to rung 3

Same discipline, different evidence. A serving namespace shows `SM_ACTIVE` 0.72, `PIPE_TENSOR_ACTIVE` 0.68 — high on both, so it is genuinely doing tensor math — but TPOT (05.6) regressed 40% after a model change. Rung 1 shows no host gaps and no fp32. Rung 2's `cuda_gpu_kern_sum` shows **one attention kernel at 61% of GPU time**. That clears the gate.

```bash
ncu --set detailed -k regex:"attention" --launch-skip 500 --launch-count 3 \
    --section SpeedOfLight_RooflineChart --section WarpStateStats \
    -o attn python serve.py

ncu --import attn.ncu-rep --page details
```

Findings: the kernel sits at 81% of the **memory** roof and 12% of the compute roof; warp state is dominated by **Stall Long Scoreboard**; achieved occupancy 31%, limited by **registers per thread**. Translation: the kernel is memory-latency-bound with too little parallelism to hide it. The fix is **not** to hand-tune it — it is to switch to a fused flash-attention path that keeps the working set in shared memory and stops round-tripping the attention matrix through HBM. Nsight Compute's job was to *prove* which upstream change pays, and it did.

## Practice

Feeds the deliverable ([`../practice/gpu-dashboard-lie/README.md`](../practice/gpu-dashboard-lie/README.md)). Develop against the [fake GPU fleet lab](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md) where you can, but rungs 1–3 need a real GPU.

1. **Baseline the shape.** Take a low-`PIPE_TENSOR_ACTIVE` workload from 05.1/05.6, or reproduce one deliberately: an fp32 training loop with `num_workers=0` on a rented GPU. Record all five PROF fields **and the raw series**, not just the 15-minute averages. Note whether `SM_ACTIVE` oscillates.

2. **Map your hardware's counter groups.** Run `dcgmi profile -l` on the GPU you are using and write down which of `sm_active` / `sm_occupancy` / `tensor_active` / `dram_active` share a subgroup. **Acceptance:** you can state which pairs of fields your dashboard collects in one pass and which are multiplexed on *your* silicon.

3. **Clear the contention boundary.** Disable `DCGM_FI_PROF_*` collection on the node (evict `dcgm-exporter` with the deploy label, or swap in a counters CSV without PROF fields), and silence the resulting alerts with an explanatory comment. **Acceptance:** a written runbook step, and a screenshot of the annotated dashboard gap.

4. **Rung 1 — torch.profiler.** Capture 3–5 steps with `skip_first`, `wait`, `warmup`, `active` set deliberately. Produce (a) the `key_averages` op ranking with `group_by_input_shape=True`, and (b) a Chrome/Perfetto timeline screenshot. Identify the cause(s): dataloader stall / host-sync / fp32 path / launch-bound. **Acceptance:** you can point at the specific rows and the specific gap that support your diagnosis, and you can say what fraction of GPU time was *not* in a GEMM.

5. **Rung 2 — Nsight Systems.** Capture an `nsys` timeline with NVTX ranges over a few steps using `--capture-range=cudaProfilerApi`. Run `nsys stats --report nvtx_sum` and `--report cuda_gpu_kern_sum`. **Acceptance:** an explicit gate decision — "top kernel is X% of GPU time, therefore I am / am not climbing to rung 3" — with the number.

6. **Rung 3, only if the gate opened.** If one kernel dominates, run `ncu` with `--launch-skip`/`--launch-count`/`-k regex:` and at least `SpeedOfLight_RooflineChart` + `WarpStateStats`. Record the roofline position, the top warp-stall reason, and the occupancy limiter. **Acceptance:** you can state what the kernel is bound by and what *upstream* change that implies.

7. **Fix and re-measure.** Apply the fix (autocast/bf16, dataloader workers + pin_memory, remove per-step sync, `torch.compile`, larger batch). Re-enable PROF collection and record the same fields.

8. **Price it.** Compute (a) the GPU-cost of each profiling step you actually ran, and (b) the monthly waste the fix eliminated, using your own $/GPU-hr.

**Acceptance — the deliverable artifact:** a one-page **before/after profiling round-trip note** containing: baseline metric shape (raw series, not just the mean) → which rung and *why that rung* → the specific trace finding with a screenshot → the fix as a diff → the **DCGM metric moving** (before/after values plus a graph) → **and the `GPU_UTIL` row showing it barely moved** → the cost of the investigation against the cost of the bug. It must justify *why you stopped where you did* on the ladder, and it must include the counter-contention step, because a round-trip note that skips it will not reproduce.

## Common pitfalls

1. **Reaching for Nsight Compute first because it is "the real profiler."** Most production waste — including Meta's own case — is visible and fixable at rung 1. *Mechanism:* `ncu` profiles one kernel and cannot see the host, so if your bug is a dataloader stall it will show you a perfectly healthy kernel while the actual problem sits in Python. Cost: §9's $72-per-attempt versus $0.004.

2. **Profiling with DCGM PROF collection still running.** The hardware performance counters are a device-global resource; DCGM and Nsight both want to program them. *Symptoms:* counter-init failures, DCGM PROF fields going blank on that node (and a false "GPU went idle" alert), or quietly wrong numbers from either tool. Disable PROF on the node first, and annotate the gap.

3. **`warmup=0` in the profiler schedule.** The first profiled step absorbs CUPTI initialisation and per-context metric setup. PyTorch warns you explicitly. *Consequence:* a first step inflated by a variable one-time cost, which you will then "optimise."

4. **Forgetting `prof.step()`.** The schedule is a state machine driven by that call. Without it the profiler never transitions out of `wait` and you get an empty trace, which reads as "nothing to see here."

5. **Profiling the first steps of a training loop.** cuDNN autotuning, `torch.compile` compilation, allocator growth and NCCL warmup all happen there. Use `skip_first` (and `--launch-skip` for `ncu`), or you will discover a startup artefact and ship a fix for it.

6. **Trusting a 15-minute `SM_ACTIVE` average.** 0.89 steady and "0.95 for 2.5 s / 0.05 for 0.4 s" produce nearly the same mean and demand completely different fixes. Look at the raw series before choosing a rung.

7. **Skipping the rung-2 gate.** Going straight to `ncu` because the DCGM shape "feels" memory-bound risks profiling the wrong kernel in exhaustive, expensive detail. The gate is a number — top kernel's share of GPU time — not an intuition.

8. **Treating a rung-3 finding as the fix.** A warp-stall reason or occupancy limiter is a *diagnosis*. The lever is almost always upstream: a fused kernel, a precision change, a reshape, `torch.compile`. You rarely hand-write PTX; you use `ncu` to prove which library change pays.

9. **Reading `Not Selected` as a problem.** It means the warp was eligible but another was chosen — i.e. you have surplus parallelism. It is the *healthy* stall reason. Chasing it is chasing saturation.

10. **Under-profiling shared cloud GPUs because of permission friction.** `ERR_NVGPUCTRPERM` is a documented, solvable problem (§7), not a wall. Set up a profiling node pool or a `CAP_SYS_ADMIN` profiling pod once, and learn the headless `nsys stats` / `ncu --import` path so the missing GUI stops being an excuse.

11. **Comparing profiled timings to production timings.** Under default kernel replay, Nsight Compute serialises launches and may lock clocks. If your workload's performance depends on kernel overlap, use range or graph replay — and never quote a profiled kernel's wall-clock as its production cost.

12. **Confusing `GPU_UTIL` with productivity while reading any of these traces.** All three profiling rungs exist because util% cannot distinguish "occupied" from "productive." The worked example's 98% → 99% across a 2.2× speedup is the proof.

## Self-check

- **`SM_ACTIVE` is high but `PIPE_TENSOR_ACTIVE` is low. Which tool next, and what are you looking for?**
  **Answer:** Start at **rung 1, `torch.profiler`** (if it is a framework workload), not Nsight Compute. The shape means warps are resident and issuing but the tensor pipes are idle, so the hypothesis is **wrong precision or wrong op mix**. In the trace you look for: fp32 GEMM kernel names (`sgemm`/`s16816gemm`) with no `autocast` in the recorded stacks; GEMMs with input shapes too small to reach the tensor cores (which is why you set `record_shapes=True`); and an op ranking dominated by layer norms, elementwise ops and `aten::copy_` rather than `mm`/`addmm`/`_scaled_dot_product`. You escalate to Nsight Systems if that is ambiguous, and to Nsight Compute only if Nsight Systems then shows one kernel taking more than roughly 30–50% of GPU time. Before either escalation, disable DCGM PROF collection on the node.

- **Nsight Systems vs Nsight Compute — when do you use each?**
  **Answer:** **Nsight Systems** is the *system timeline* profiler: whole process, wall-clock axis, CPU threads and OS runtime + CUDA API + kernels + memcpys + NCCL + your NVTX ranges. Use it to find **where the time goes** — gaps, CPU↔GPU overlap, host stalls, collective exposure, and *which kernel dominates*. Overhead is low with `-s none` and roughly 2× or more with `-s cpu` sampling. **Nsight Compute** is the *single-kernel* profiler: it re-executes one kernel once per counter-bank pass to collect a full metric set (roofline position, occupancy limiters, warp-stall reasons), serialising launches while it does so. Use it **only after** Nsight Systems has isolated one hot kernel. Systems answers "which kernel / which phase"; Compute answers "why this kernel," at one to two orders of magnitude of overhead on the profiled kernel because it replays.

- **Name a bottleneck DCGM cannot see but a profiler can, and say why DCGM cannot see it.**
  **Answer:** A **dataloader / host-input stall** — the GPU is idle because the CPU-side `DataLoader` is not delivering batches. DCGM is a device-level, roughly 1 Hz, counter-only interface: it has no notion of the host, no call stack, no per-kernel attribution and no intra-step time resolution, so it can show low `SM_ACTIVE` but cannot attribute it. The PyTorch profiler or an `nsys` timeline shows the GPU stream going empty at each step boundary while the CPU thread blocks in `DataLoader.__next__`. Equally valid answers: a per-step `cudaStreamSynchronize` caused by `loss.item()`; kernel-launch-bound execution across thousands of tiny kernels (where `SM_ACTIVE` can even look *decent*); and a kernel's warp-stall *reason*, which only Nsight Compute can produce.

- **Why does Nsight Compute have to replay a kernel, and what does that share with DCGM?**
  **Answer:** The SM's hardware performance counters live in a fixed number of counter banks, and each bank holds one counter-select configuration at a time — so a full metric set cannot be programmed simultaneously. Nsight Compute groups the requested metrics into the minimum number of passes the hardware allows and **re-executes the kernel once per pass**, saving and restoring the memory the kernel writes so every replay sees identical inputs and the result is deterministic. It also serialises launches and pays a substantial **one-time per-CUDA-context** cost to build the metric configuration. DCGM hits the identical constraint from the other direction: its profiling module knows which PROF fields can be collected together on that architecture, returns **`DCGM_ST_PROFILING_MULTI_PASS`** when they cannot, and `dcgmi profile -l` prints the groupings. **Because both tools want the same banks, they conflict** — which is why you disable DCGM PROF collection on a node before running Nsight there.

- **You are about to run `nsys` on `gpu-node-17`. What do you do first, and why?**
  **Answer:** Stop `DCGM_FI_PROF_*` collection on that node — either label it `nvidia.com/gpu.deploy.dcgm-exporter=false` so the exporter is evicted, or roll a counters CSV without PROF fields — and **silence/annotate the resulting alerts**. Reason: the GPU's performance counters are a device-global resource that both DCGM's profiling module and Nsight's collection want to program. Leaving both running gives you counter-initialisation failures, blank PROF fields on that node (a hole in the fleet dashboard that reads exactly like "the GPU went idle" and will page someone), or silently wrong numbers from one or both tools. Re-enable it after the session and before you re-measure, since the whole point of the round-trip is to show the DCGM metric moving.

- **In Meta's production case, what tool found the GPU-idle/GPU-active pattern, what tracing layer is it built on, and what would you use to find the same thing across 64 ranks?**
  **Answer:** Meta's internal **MAIProf**, built on **Kineto** (PyTorch's profiling library), which wraps NVIDIA's **`libcupti`** — the same CUDA tracing substrate under public `torch.profiler` and, at a lower level, Nsight Systems' kernel timeline. The GPU was idle **more than half** the total training time. Across many ranks you would not read traces by hand: you would use **Holistic Trace Analysis (HTA)**, which consumes Kineto traces and produces a per-rank temporal breakdown plus an **idle-time breakdown that separates host wait** (CPU not enqueuing kernels fast enough — exactly this failure) **from kernel wait** (inter-launch overhead), along with kernel-duration rankings and communication–computation overlap.

- **Why does profiling fail inside an ordinary Kubernetes pod, and what are your three options?**
  **Answer:** GPU performance counters are privileged. With the NVIDIA kernel module parameter `NVreg_RestrictProfilingToAdminUsers=1` (the common default), only root or a process holding **`CAP_SYS_ADMIN`** (or `CAP_PERFMON`) may program them, so `ncu` in a normal container returns **`ERR_NVGPUCTRPERM`**. Options: (1) set `NVreg_RestrictProfilingToAdminUsers=0` via `/etc/modprobe.d` fleet-wide — simple but lets any tenant read shared counter state; (2) do the same on a **taint-isolated profiling node pool** only, and reschedule jobs there to profile them — the pragmatic answer, bounded blast radius; (3) grant `CAP_SYS_ADMIN` on a short-lived, purpose-built profiling pod behind an admission policy. Changing the module parameter requires a module reload or reboot and, on some distributions, an initrd rebuild (`dracut --regenerate-all -f` / `update-initramfs -u -k all`).

- **Put a cost on getting the ladder wrong.**
  **Answer:** At $2.50/GPU-hr: rung 1 on a 64-rank job costs about **$0.24** (and about $0.004 if you profile rank 0 only); rung 2 costs a few dollars, dominated not by the profiling but by the **job restart** needed to attach; rung 3 on one GPU is about **$0.42**. The failure mode is rung 3 *without* the rung-2 gate — running `ncu --set full` across 40 kernels on a 64-GPU job holds all 64 GPUs serialised for ~25 minutes plus restart, roughly **$72 per attempt**, to answer a question rung 1 answers for less than a cent. Against that, the bug itself — Meta's >50% host-side idle on a 64-GPU job — is about **$1,920/day, $57,600/month**. So the discipline is not about saving profiling cost (profiling is cheap); it is about not converting a five-minute question into a cluster-wide stall, and about actually profiling instead of guessing.

## Connections & what's next

This lesson is the action layer on top of every metric lesson before it. 05.1/05.2 gave you the shape vocabulary (`SM_ACTIVE`/`SM_OCCUPANCY`/`TENSOR_ACTIVE`/`DRAM_ACTIVE`) and the multiplexing constraint that turns out to be the same silicon fact behind Nsight Compute's replay. 05.3 gave you the counters-CSV control that you now use *in reverse* — turning PROF fields **off** on a node to clear the counter-contention boundary — and 04.1's `nvidia.com/gpu.deploy.*` labels are the surgical way to do it. 05.4 gave you the per-pod label join that tells you *which* workload to profile. 05.5 gave you the sibling failure class: hardware faults you cordon rather than profile, and `dcgmi diag -r 2` is the cheap filter that rules out "it's dying" before you spend rung-2 effort on "it's slow." 05.6 gave you the inference-specific shapes (queue-vs-decode, ITL-vs-TPOT divergence) that map onto this same ladder.

The capstone (05.8) is where it pays off directly: the deliverable's blog post explicitly wants an "and here is how you *fix* an inefficient GPU once you have found it" section, and this lesson's round-trip — baseline shape, rung choice with a stated gate, trace finding, fix, metric delta, **and the `GPU_UTIL` row that did not move** — is that section verbatim. It is also the evidence that converts the capstone's dollar gap from an observation into a recoverable number: the reshuffle panel says *which* namespace is wasting capacity, and this lesson says *what to change*.

## References & further reading

**Primary sources**

1. **PyTorch — `torch/profiler/profiler.py`** — https://github.com/pytorch/pytorch/blob/main/torch/profiler/profiler.py — the authoritative API surface: `profile(...)` arguments and defaults (`record_shapes`, `profile_memory`, `with_stack`, `with_flops`, `acc_events`, `experimental_config`), `schedule(skip_first, wait, warmup, active, repeat)`, `ProfilerActivity` values, `tensorboard_trace_handler`, `key_averages(group_by_input_shape=...)`, `export_chrome_trace`, `export_stacks`. Also the source of the deprecation notes used above: `export_memory_timeline` is superseded by `torch.cuda.memory._record_memory_history` / `_export_memory_snapshot`, and `with_modules` is deprecated (TorchScript only).
2. **PyTorch — Profiler recipe** — https://docs.pytorch.org/tutorials/recipes/recipes/profiler_recipe.html — the worked rung-1 walkthrough: schedule, trace export, and reading `key_averages`.
3. **PyTorch — "Introduction to Holistic Trace Analysis"** — https://docs.pytorch.org/tutorials/beginner/hta_intro_tutorial.html and https://github.com/facebookresearch/HolisticTraceAnalysis — temporal breakdown, the host-wait vs kernel-wait idle split, kernel breakdown, communication–computation overlap, and CUDA launch statistics, computed across all ranks from Kineto traces.
4. **NVIDIA — Nsight Systems User Guide** — https://docs.nvidia.com/nsight-systems/UserGuide/index.html — `nsys profile` flags (`-t`, `-s`, `-w`, `-f`, `-x`, `--capture-range`, `--stop-on-range-end`, `--cudabacktrace`), NVTX semantics, the `.nsys-rep` format, and the `nsys stats --report …` report names used in §5.
5. **NVIDIA — Nsight Compute Profiling Guide** — https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html — the replay model and its four modes (kernel / application / range / application-range, plus CUDA-graph profiling as a single workload entity), the note that per-context metric-configuration overhead is a large one-time cost that does not recur for later kernels with an unchanged metric list, the section and set taxonomy, and the warp-state and roofline sections.
6. **NVIDIA — Nsight Compute CLI reference** — https://docs.nvidia.com/nsight-compute/NsightComputeCli/index.html — `--set` / `--section` / `--list-sets`, `-k regex:`, `--launch-skip`, `--launch-count`, `--nvtx-include`, `--replay-mode`, `--import`, `--csv`.
7. **NVIDIA — "ERR_NVGPUCTRPERM: Permission issue with Performance Counters"** — https://developer.nvidia.com/nvidia-development-tools-solutions-err_nvgpuctrperm-permission-issue-performance-counters — `NVreg_RestrictProfilingToAdminUsers` semantics, the `/etc/modprobe.d` configuration, the `CAP_SYS_ADMIN` / `CAP_PERFMON` alternative, and the distribution-specific initrd rebuild commands.
8. **NVIDIA DCGM — Profiling module documentation** — https://docs.nvidia.com/datacenter/dcgm/latest/learn/modules/profiling.html and the `dcgmi` command-line reference — `dcgmi profile -l`'s Group.Subgroup / Field-ID layout, which field groups can be collected concurrently, and the `DCGM_ST_PROFILING_MULTI_PASS` status that surfaces the counter-bank constraint.
9. **NVIDIA — CUPTI documentation** — https://docs.nvidia.com/cupti/index.html — the activity and callback APIs, and correlation IDs, which are the mechanism behind `torch.profiler` stitching a Python op to the GPU kernel it launched.

**Real-world engineering**

10. **PyTorch (Meta) — "Performance Debugging of Production PyTorch Models at Meta"** — https://pytorch.org/blog/performance-debugging-of-production-pytorch-models-at-meta/ — the GPU-idle/GPU-active production case (>50% idle) this lesson's stakes and worked example are built on, found with a Kineto-based profiler.
11. **NVIDIA Developer Blog — "Using Nsight Compute to Inspect your Kernels"** — https://developer.nvidia.com/blog/using-nsight-compute-to-inspect-your-kernels/ — a concrete rung-3 result: 87.5% fewer memory transactions and 68% shorter kernel duration on one kernel.
12. **`mcarilli` — "Favorite nsight systems profiling commands for PyTorch scripts"** — https://gist.github.com/mcarilli/376821aa1a7182dfcf59928a7cde3223 — the practical `nsys` flag sets in §5, the `-s cpu` ≈ 2×-or-more overhead observation, `--cudabacktrace-threshold` / `--osrt-threshold`, and the `cudaProfilerStart`/`Stop` + `--capture-range` pattern.

**Deeper dives**

13. **NVIDIA Developer Blog — "Advanced Kernel Profiling with the Latest Nsight Compute"** — https://developer.nvidia.com/blog/advanced-kernel-profiling-with-the-latest-nsight-compute/ — range replay, graph profiling, and the newer analysis rules, for when `torch.compile`/CUDA graphs make per-kernel profiling the wrong unit.
14. **Spheron — "GPU Profiling for AI Workloads: Nsight Compute, Nsight Systems, and PyTorch Profiler" (2026)** — https://www.spheron.network/blog/gpu-profiling-ai-workloads-nsight-compute-pytorch-profiler-guide/ — a practitioner map of the three-tool ladder and the "why teams under-profile shared GPUs" friction the pitfalls section draws on.
