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
sources: 14
---

# 03.1 · Execution model and the utilisation lie

> **Concept.** Just enough of the SM/warp/kernel execution model to say precisely what `nvidia-smi` "GPU-Util" measures — and to prove it is not throughput.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Module 02 ended with you able to reason about control-plane internals — the machinery underneath the Kubernetes objects you operate. This module shifts one layer down again, from "what is the scheduler doing" to "what is the accelerator itself doing once a pod lands on it." This first lesson is the module's opening move and its thesis statement: the number every dashboard shows you for a GPU (`GPU-Util`) is not a measure of work, and until you can say precisely *why* it isn't, every other lesson in this module — roofline placement, memory bottlenecks, batching, precision, SKU choice, cost capstone — is standing on sand.

Nothing here assumes prior GPU-specific knowledge. The lesson builds the execution model from first principles — why the hardware is shaped the way it is, how a kernel launch turns into work on silicon, and which of those layers each available counter can and cannot see — just deep enough to read a profiler honestly. Lesson 03.2 then takes the counters you learn to read here and turns them into a predictive model.

## Why this matters

The single most expensive mistake in GPU fleet operations is trusting `nvidia-smi` GPU-Util as a measure of work done. A GPU can report **100% utilisation while doing less than 1% of the work it is capable of** — and you are paying the same $2–4/GPU-hour either way. If you cannot articulate, in an interview or a capacity review, exactly what that number counts and why it lies, you cannot build the cost/observability story that differentiates a Senior/Staff platform engineer at a GPU-fleet operator.

This is not hypothetical rigour. Google Cloud's own published benchmark of a 96×B200 GKE Autopilot cluster serving live LLM decode traffic measured **4.4% FLOPS utilisation and tensor cores active only 1.5% of the time — while GPU-Util itself would read near 100%** ("What Does 4.4% GPU Utilization Actually Mean?", Google Cloud on Medium). That is a real production system, not a synthetic demo, exhibiting exactly the lie this lesson names. CoreWeave's job postings for *GPU Performance Engineer* explicitly ask for "hardware validation across the fleet" and "visibility into metrics/performance/health" — meaning candidates are expected to know which metric tells the truth.

The stakes scale with the fleet. One H100 node (8 GPUs) rented at roughly $3/GPU-hour is about $210,000/year. A 40-node fleet is on the order of $8M/year. If your only signal is a counter that reads 100% for a GPU doing a rounding error of useful work, you have no instrument capable of detecting a 2× efficiency loss across that spend. **Utilisation is not throughput, and throughput is what you pay for.**

## What's new here (calibration)

Per the module README you are not becoming a CUDA kernel developer — 02b already covered topology/power, and this module explicitly skips kernel authoring, PTX/SASS, deep microarchitecture, and occupancy-tuning-as-coding. What this lesson adds instead:

- **The execution model as a two-sided mapping** — the software hierarchy (grid → block → warp → thread) and the hardware hierarchy (GPU → GPC → SM → processing block → cores/tensor cores), and exactly where they meet. That mapping is what makes the utilisation lie *structural* rather than a monitoring bug.
- **Warp-level latency hiding as a mechanism you can draw** — the reason a GPU can look permanently busy while its ALUs idle, worked as a timeline rather than asserted as a slogan.
- **Precise vocabulary for "busy" at four granularities** — temporal (GPU-Util), spatial (SM_ACTIVE), density (SM_OCCUPANCY), and functional (PIPE_TENSOR_ACTIVE) — enough to pick the right metric to alert and bill on.
- **The exact NVML definition of GPU-Util decoded word by word**, including its sampling window and the mechanism that produces the number, so you can defend *why* it lies rather than asserting that it does.
- **MFU as the metric that ties utilisation to money**, with the `6·N·D` FLOP convention derived rather than quoted, plus a real per-model spec table so your denominators are dense figures and not marketing sparse ones.

## Core concepts

### 1. The problem the GPU exists to solve

Start with the constraint, not the chip. A CPU core is built to minimise the latency of a single instruction stream: deep out-of-order windows, aggressive branch prediction, big private caches, high clocks. It spends most of its transistor budget on *control* so that one thread runs as fast as physics allows.

Dense linear algebra does not need that. A matrix multiply is millions of independent multiply-accumulates with a completely predictable access pattern. What it needs is *arithmetic throughput per watt* and *memory bandwidth*, and it does not care if any individual operation takes 500 ns to complete, as long as thousands of them are in flight.

So the GPU makes the opposite trade. It spends transistors on ALUs and on memory interfaces, and it makes latency somebody else's problem by keeping enormous numbers of independent work items resident simultaneously. An H100 SXM5 has 16,896 FP32 lanes and 528 tensor cores across 132 SMs; each SM can hold **2,048 threads** resident at once, so the whole chip can have **270,336 threads in flight** (132 × 2,048) with their register state live in hardware. When one group stalls on memory, another issues in the same cycle. That is the entire architectural idea: **hide latency with concurrency instead of eliminating it with caches.**

Two consequences follow immediately, and they are the root of everything else in this lesson:

1. **Work must be supplied in enormous parallel quantities or most of the machine sits idle.** There is no out-of-order engine to find parallelism for you.
2. **"The GPU is executing something" and "the GPU is doing a meaningful amount of work" are almost unrelated statements**, because the machine is wide and any single kernel can occupy an arbitrarily tiny slice of it.

### 2. The software side: grid, block, warp, thread

CUDA gives you a three-level launch hierarchy (four on Hopper and later, which adds an optional cluster level).

- **Thread** — one instance of the kernel function. Has its own registers and program counter.
- **Warp** — **32 threads**, always. This is not a tunable; it is the hardware's issue quantum. A warp's 32 threads share one instruction stream: the scheduler issues one instruction and all 32 lanes execute it. If threads in a warp take different branches, the warp executes both paths with the inactive lanes masked off (*warp divergence*), so a divergent branch costs you throughput.
- **Thread block (CTA, cooperative thread array)** — up to **1,024 threads** (i.e. up to 32 warps). This is the unit the hardware *places*. A block is assigned to exactly one SM and stays on that SM until every one of its threads has finished. Threads in a block can share data through the SM's shared memory and synchronise with `__syncthreads()`; threads in different blocks cannot (they may not even be resident at the same time).
- **Grid** — all blocks launched by one kernel invocation. `kernel<<<gridDim, blockDim>>>(...)`.
- **Cluster** (compute capability 9.0 / Hopper and later, optional) — a group of blocks guaranteed to be co-scheduled on the same GPU Processing Cluster so they can address each other's shared memory. The portable maximum is **8 blocks per cluster** (CUDA C++ Programming Guide, thread block clusters).

The load-bearing sentence: **a block is indivisible and lands on exactly one SM.** Launch a grid of one block and you have used one SM. The other 131 are powered, clocked, billed — and empty.

### 3. The hardware side, and where the two hierarchies meet

An H100 SXM5 (GH100 die, per NVIDIA's Hopper architecture whitepaper) is organised as: 8 GPCs → 66 TPCs (2 SMs each) → **132 SMs**, each with 128 FP32 CUDA cores and 4 fourth-generation tensor cores, giving 16,896 FP32 cores and 528 tensor cores per GPU. Each SM is itself split into **four processing blocks** (also called SM sub-partitions, SMSPs). Each processing block has its own warp scheduler, its own dispatch unit, 32 FP32 lanes, one tensor core, and its own 16,384-entry slice of the SM's 65,536 32-bit registers (256 KB of register file per SM).

That last level is where most people's mental model is too coarse. **Warps are not scheduled by the SM; they are scheduled by one of the SM's four processing blocks.** A warp is assigned to a processing block when its block is placed, and it stays there. Each scheduler picks, every clock, one *eligible* resident warp and issues one instruction for it.

```
   SOFTWARE (what you launch)                HARDWARE (what runs it)
   ─────────────────────────                 ───────────────────────

   GRID  kernel<<<N_blocks, 256>>>           GPU  (H100 SXM5, GH100)
     │                                        │
     │  ┌─ block 0 ─┐ ┌─ block 1 ─┐ …         ├── GPC 0 … GPC 7        (8 GPCs)
     │  │ 256 thr   │ │ 256 thr   │           │     └── TPC (2 SMs) ×66
     │  └───────────┘ └───────────┘           │
     │        │                               └── 132 SMs total
     │        │  the GigaThread engine /            │
     │        │  work distributor assigns           │
     │        └── each whole block ───────────▶  ONE SM  (never split)
     │                                              │
     │  a block is chopped into warps               │
     │  of exactly 32 threads:                      │
     │     256 thr → 8 warps                        │
     │        │                                     │
     │        └── warps are handed to ──────▶  one of 4 PROCESSING BLOCKS
     │                                              │   (SM sub-partitions)
     ▼                                              ▼
                                    ┌──────────────────────────────────┐
                                    │  SM  (1 of 132)                  │
                                    │  256 KB register file            │
                                    │  228 KB shared memory (max)      │
                                    │  ┌────────────┐  ┌────────────┐  │
                                    │  │ proc blk 0 │  │ proc blk 1 │  │
                                    │  │ warp sched │  │ warp sched │  │
                                    │  │ 16 warp    │  │ 16 warp    │  │
                                    │  │   slots    │  │   slots    │  │
                                    │  │ 32 FP32    │  │ 32 FP32    │  │
                                    │  │ 1 TENSOR   │  │ 1 TENSOR   │  │
                                    │  │   CORE     │  │   CORE     │  │
                                    │  └────────────┘  └────────────┘  │
                                    │  ┌────────────┐  ┌────────────┐  │
                                    │  │ proc blk 2 │  │ proc blk 3 │  │
                                    │  │    (same)  │  │    (same)  │  │
                                    │  └────────────┘  └────────────┘  │
                                    │  L1 data cache / shared (256 KB) │
                                    └───────────────┬──────────────────┘
                                                    │
                                    ┌───────────────▼──────────────────┐
                                    │  50 MB L2  ──▶  80 GB HBM3       │
                                    │                 3.35 TB/s        │
                                    └──────────────────────────────────┘

   WHAT EACH COUNTER CAN SEE
   ─────────────────────────
   GPU-Util (NVML)     ▸ "is ANY kernel resident anywhere on this box?"  ── whole-GPU bit
   SM_ACTIVE   (1002)  ▸ per-SM: ≥1 warp resident, averaged over all 132 SMs
   SM_OCCUPANCY(1003)  ▸ per-SM: resident warps ÷ 64 warp slots
   TENSOR_ACTIVE(1004) ▸ per-SM: cycles the 4 tensor-core pipes issued
   DRAM_ACTIVE (1005)  ▸ HBM controller: cycles spent reading/writing
```

Read the diagram once more with the counters in mind. `GPU-Util` is a property of the **top box**. Everything you actually pay for lives four levels down.

### 4. Placement and the four occupancy limiters

When you launch a kernel, the GPU's work distributor walks the grid and places blocks onto SMs with free capacity. A block fits on an SM only if *all four* of the following resources are available:

1. **Warp slots.** Each SM has a fixed number of resident-warp slots — **64 on Ampere (CC 8.0) and Hopper (CC 9.0)**, i.e. 2,048 threads, 16 per processing block.
2. **Thread-block slots.** Each SM can hold at most **32 resident blocks** (CC 8.0 and 9.0). So blocks of 64 threads (2 warps) cap you at 32 × 2 = 64 warps — exactly full; blocks of 32 threads cap you at 32 warps, i.e. **50% occupancy no matter what else you do.** This is the reason tiny blocks are a mistake.
3. **Registers.** The SM has **65,536 32-bit registers** (CC 8.0 and 9.0), and a single thread may use at most **255**. Registers are allocated per thread, so a kernel compiled to 64 registers/thread supports 65,536 / 64 = 1,024 threads = 32 warps = 50% occupancy, and no more.
4. **Shared memory.** Per-SM shared memory is **up to 164 KB on A100 (CC 8.0)** and **up to 228 KB on H100 (CC 9.0)**, carved out of a unified 192 KB / 256 KB L1+shared block. A kernel asking for 100 KB of shared memory per block gets at most 2 blocks resident on H100.

| Technical spec (CUDA C++ Programming Guide, "Technical Specifications per Compute Capability") | V100 (7.0) | A100 (8.0) | H100 (9.0) |
|---|---|---|---|
| Warp size | 32 | 32 | 32 |
| Max threads per block | 1,024 | 1,024 | 1,024 |
| Max resident warps per SM | 64 | 64 | 64 |
| Max resident threads per SM | 2,048 | 2,048 | 2,048 |
| Max resident thread blocks per SM | 32 | 32 | 32 |
| 32-bit registers per SM | 65,536 | 65,536 | 65,536 |
| Max registers per thread | 255 | 255 | 255 |
| Max shared memory per SM | 96 KB | 164 KB | 228 KB |
| Unified L1 + shared per SM | 128 KB | 192 KB | 256 KB |
| Thread block clusters | — | — | yes (8 blocks portable) |

**Occupancy** is the ratio you get out of this: `occupancy = resident warps per SM ÷ 64`. It is a *static-ish* property of a kernel's resource footprint, computable before you run anything (`nvcc --ptxas-options=-v` prints registers and shared memory per thread; `cudaOccupancyMaxActiveBlocksPerMultiprocessor()` computes the resulting block count at runtime).

Notice what occupancy is not: it is not a measure of work. It tells you how many warps are *parked* on the SM, not how many are *issuing*.

### 5. Latency hiding: the mechanism behind a busy-looking, idle GPU

Here is the timeline that explains everything.

Each processing block issues at most one instruction per clock, to one warp. A dependent arithmetic instruction on Ampere/Hopper takes on the order of **4 clock cycles** before its result is available; a global-memory (HBM) load takes on the order of **hundreds** of cycles — measured microbenchmarks on A100 report **shared memory ≈ 29 cycles, L1 ≈ 38 cycles, L2 ≈ 262 cycles, HBM ≈ 466 cycles** (Luo et al., "Benchmarking and Dissecting the Nvidia Hopper GPU Architecture", arXiv:2402.13499, which reports these tiers for A100/H800-class parts and finds L2 ≈ 6.5× L1 and global ≈ 1.9× L2).

So a warp that issues a load then needs the value spends ~466 idle cycles. To keep the scheduler issuing every cycle, you need enough *other* resident warps that at least one is eligible in each of those cycles. That is latency hiding, and this is what it looks like:

```
  ONE WARP SCHEDULER (1 of 4 in an SM). Time flows right. Each cell = 1 clock.
  I = issues an instruction   . = stalled (waiting on a dependency)

  CASE A — 2 resident warps (occupancy 2/16 on this scheduler ≈ 12%)
  ─────────────────────────────────────────────────────────────────────
  warp 0   I . . . . . . . . . . . . . . . . . . . . I . . . . . . . .
  warp 1   . I . . . . . . . . . . . . . . . . . . . . I . . . . . . .
           ▲ ▲                                       ▲ ▲
  issued:  1 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 1 0 0 0 0 0 0
           └── scheduler issues on 4 of 29 cycles → ~14% issue rate ──┘
           ALUs idle 86% of the time.  A kernel IS resident the whole time.
           NVML GPU-Util for this window: 100%.

  CASE B — 12 resident warps, same kernel, more blocks placed
  ─────────────────────────────────────────────────────────────────────
  warp 0   I . . . . . . . . . . . I . . . . . . . . . . . I . . . . .
  warp 1   . I . . . . . . . . . . . I . . . . . . . . . . . I . . . .
  warp 2   . . I . . . . . . . . . . . I . . . . . . . . . . . I . . .
  warp 3   . . . I . . . . . . . . . . . I . . . . . . . . . . . I . .
  warp 4   . . . . I . . . . . . . . . . . I . . . . . . . . . . . I .
  warp 5   . . . . . I . . . . . . . . . . . I . . . . . . . . . . . I
  warp 6   . . . . . . I . . . . . . . . . . . I . . . . . . . . . . .
  warp 7   . . . . . . . I . . . . . . . . . . . I . . . . . . . . . .
  warp 8   . . . . . . . . I . . . . . . . . . . . I . . . . . . . . .
  warp 9   . . . . . . . . . I . . . . . . . . . . . I . . . . . . . .
  warp10   . . . . . . . . . . I . . . . . . . . . . . I . . . . . . .
  warp11   . . . . . . . . . . . I . . . . . . . . . . . I . . . . . .
  issued:  1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1
           └────── scheduler issues EVERY cycle → 100% issue rate ─────┘
           NVML GPU-Util for this window: also 100%.

  SAME COUNTER VALUE. ~7× DIFFERENT THROUGHPUT.
```

Two things fall out of this picture that you should carry for the rest of the module:

- **GPU-Util cannot distinguish Case A from Case B.** Both have a kernel resident for the whole window. The counter is a *residency* bit sampled over time, and residency is identical in both.
- **Occupancy buys you latency hiding, and only that.** Once you have enough warps to issue every cycle, more warps add nothing — which is why a well-tuned GEMM can hit near-peak FLOP/s at ~50% occupancy (it has huge instruction-level parallelism inside each warp and doesn't need 64 of them), while a pointer-chasing kernel at 100% occupancy can still stall constantly. Occupancy is an *input* to throughput, never a measure of it.

### 6. What `nvidia-smi` GPU-Util actually measures

NVML's own definition of `utilization.gpu`, the field `nvidia-smi` prints in the `Volatile GPU-Util` column:

> **GPU-Util** = "Percent of time over the past sample period during which one or more kernels was executing on the GPU. The sample period may be between 1 second and 1/6 second depending on the product."

Decode it clause by clause.

- **"percent of time"** → it is a *temporal duty cycle*. It has units of time-over-time. It is dimensionally incapable of expressing work, FLOPs, bytes, or parallelism.
- **"one or more kernels"** → *one* is sufficient. One block, on one of 132 SMs, using one of that SM's four schedulers, with tensor cores stone-cold, satisfies the condition completely.
- **"was executing"** → residency, not issue rate. Case A above satisfies "executing" on every sample.
- **"the sample period may be between 1 second and 1/6 second"** → the window is coarse and product-dependent. Mechanically, the driver reads a busy/idle indication from the GPU's performance-monitoring block at a fixed cadence within that window and reports `busy_samples ÷ total_samples`. A stream of 20 µs kernels back to back therefore pins the counter to 100% just as surely as one long kernel — there is never a sample point that lands in a gap.

So GPU-Util answers exactly one question: **"was this GPU non-idle?"** It does not answer *how many SMs*, *how full were they*, *were the tensor cores fed*, or *how close to peak FLOP/s*.

**The arithmetic of the lie.** Launch a grid of exactly one block of 32 threads and loop it forever:

```
SMs engaged                = 1 of 132                        = 0.76 %
Warp slots used on that SM = 1 of 64                          = 1.6 %
Warp slots used GPU-wide   = 1 of (132 × 64 = 8,448)          = 0.012 %
FP32 lanes usable          = 32 of 16,896                     = 0.19 %
Tensor cores used          = 0 of 528                         = 0 %
─────────────────────────────────────────────────────────────────────
nvidia-smi Volatile GPU-Util                                  = 100 %
```

That is the lie in one block of arithmetic: a counter reading 100% for a machine that is **99.988% idle by warp-slot occupancy**.

**A field-by-field reading of the rest of `nvidia-smi`**, because the checkpoint asks you to interpret every one and the same trap recurs:

| Field | What it actually is | Common misreading |
|---|---|---|
| `Volatile GPU-Util` | % of sample period with ≥1 kernel resident | "% of the GPU's capability in use" |
| `Memory-Usage` (e.g. `41GiB / 80GiB`) | **allocated** HBM, not accessed HBM. Frameworks (PyTorch caching allocator, vLLM's `gpu_memory_utilization` reserve) allocate up front and hold | "how much memory the model needs" |
| `Utilization: Memory` in `nvidia-smi -q` | % of sample period during which device memory was being **read or written** — a bandwidth duty cycle, *not* a capacity figure | "how full the memory is" — it is not, that is `Memory-Usage` |
| `Pwr:Usage/Cap` | instantaneous board power vs limit. The single most honest free proxy for real work: an idle-but-resident H100 draws roughly 70–110 W against a 700 W cap; a saturated GEMM sits at the cap | ignored entirely |
| `Perf` (`P0`–`P12`) | performance state; `P0` means max clocks are permitted, not that they are being used | "P0 = flat out" |
| `Compute M.` | Default / Exclusive_Process / Prohibited — a policy, not a measurement | — |
| `MIG M.` | whether the GPU is partitioned. If enabled, whole-GPU util is meaningless (see below) | — |

The power column deserves a moment. It is free, always present, needs no profiling session, and is nearly impossible to fake: producing FLOPs costs joules. **A GPU reading 100% GPU-Util at 90 W is definitionally not doing work.** That single cross-check catches most instances of the lie without any DCGM setup at all.

### 7. The metrics that actually tell you about work: DCGM PROF fields

DCGM (NVIDIA Data Center GPU Manager) exposes a family of profiling fields, IDs **1001–1013**, that read the SM and memory-system performance counters directly instead of the driver's coarse busy bit. These are what you build fleet observability and chargeback on.

| Metric (DCGM field) | ID | What it measures (DCGM user guide) | The question it answers |
|---|---|---|---|
| `GPU-Util` (NVML, not DCGM PROF) | — | % of sample period with ≥1 kernel resident | Is it switched on? |
| `DCGM_FI_PROF_GR_ENGINE_ACTIVE` | 1001 | Fraction of time the graphics/compute engine was active | A slightly better "on" signal |
| `DCGM_FI_PROF_SM_ACTIVE` | 1002 | Ratio of cycles an SM had **at least 1 warp assigned**, averaged over all SMs | How much of the machine floor is running? |
| `DCGM_FI_PROF_SM_OCCUPANCY` | 1003 | Ratio of **resident warps to the maximum**, averaged over SMs | How densely packed are the running SMs? |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | 1004 | Ratio of cycles the **tensor (HMMA/IMMA) pipe** was active | Are the tensor cores — the thing you bought — doing anything? |
| `DCGM_FI_PROF_DRAM_ACTIVE` | 1005 | Ratio of cycles the device memory interface was **sending or receiving data** | Are you memory-bound? (feeds lesson 03.2) |
| `DCGM_FI_PROF_PIPE_FP64/FP32/FP16_ACTIVE` | 1006–1008 | Per-precision CUDA-core pipe activity | Which pipe is doing the work? |
| `DCGM_FI_PROF_PCIE_*` / `NVLINK_*` | 1009–1012 | Interconnect bytes tx/rx | Is the fabric the bottleneck? (02b territory) |

In `dcgmi dmon` output these appear as abbreviated columns — `GRACT`, `SMACT`, `SMOCC`, `TENSO`, `DRAMA`, `PCITX`/`PCIRX`.

**The one that kills the lie is SM_ACTIVE**, precisely because of the averaging in its definition. It is computed per SM and then averaged over all SMs, so the one-block kernel gives roughly `1 ÷ 132 ≈ 0.0076` → **0.76%**, while GPU-Util reads 100%. That gap *is* the busy-but-idle signature, and it is a direct consequence of the block-lands-on-one-SM fact from §2.

Three subtleties to keep straight:

- **SM_ACTIVE is ambiguous on its own.** A value of 0.5 can mean "all 132 SMs busy half the time" *or* "half the SMs busy all the time." Either way it caps the useful-work ceiling, so it is still vastly more honest than GPU-Util, but pair it with SM_OCCUPANCY (density) and TENSOR_ACTIVE (function) to disambiguate.
- **SM_ACTIVE ≠ issue rate.** Re-read Case A of the timeline: one resident warp stalling on HBM for 466 cycles still counts as "at least 1 warp assigned" for all 466 of them. **SM_ACTIVE can read 1.0 on a GPU issuing an instruction 5% of cycles.** This is the second-order version of the same lie and the follow-up question a good interviewer asks. The counters that see through it are the per-pipe ones (TENSOR_ACTIVE, FP16/32/64_ACTIVE), or Nsight Compute's `sm__inst_issued` metrics.
- **TENSOR_ACTIVE is the money metric for AI.** You are paying the H100 premium *for the 528 tensor cores*. FP64/FP32 CUDA-core work, or memory-bound elementwise ops, can pin SM_ACTIVE high while TENSOR_ACTIVE sits near zero — the expensive silicon idle while "the GPU is busy."

**Why you cannot collect all of them at once.** The SM exposes its counters through a fixed set of hardware collection slots, and several PROF metrics are derived from counters that live in the same physical bank — a bank holds one counter-select at a time, so two metrics that read it cannot be programmed in a single pass. DCGM's own documentation states this plainly and gives the generational example: **SM Activity and SM Occupancy cannot be collected together with Tensor Utilization on V100, but can be on T4.** That is why co-residency is a property of the silicon's counter layout and changes between generations, and why DCGM will *multiplex* (time-slice) incompatible groups rather than refuse. Run `dcgmi profile -l` on the hardware in front of you to print the groups; metrics in different groups are the pairs that will be multiplexed, and multiplexed samples are averages over shorter, non-overlapping windows — fine for trends, wrong for instant-by-instant correlation.

**An operational trap before you trust any dashboard.** NVIDIA's own `dcgm-exporter` GitHub issue **#662** documents `DCGM_FI_PROF_GR_ENGINE_ACTIVE` silently reporting **0** until a separate profiling session (e.g. `dcgmi dmon -e 1001`) had been run first — a real, filed bug caused by profiling-session contention, not by an idle GPU. A field that looks like it is exporting but is pinned at zero is worse than no field, because it creates false confidence. Verify the exact driver/DCGM combination you run before an alert or a chargeback number depends on it.

### 8. The three diagnoses, and the matrix that separates them

The single-block kernel is the *high-util, no-work* failure. Real fleets show two more shapes, and GPU-Util alone cannot tell them apart.

- **Launch-bound / tiny-kernel storms.** Thousands of microsecond kernels back to back — unfused elementwise ops, an eager-mode model that never saw `torch.compile`. Every sample point finds a kernel resident, so GPU-Util pins near 100%. But each kernel's grid drains before the SMs fill and the next launch has fixed overhead, so SM_OCCUPANCY and TENSOR_ACTIVE stay low. Same reading, different cause; the fix is **kernel fusion / CUDA graphs**, not a bigger GPU.
- **Host/input starvation.** A slow dataloader, CPU-side preprocessing, or GIL-bound Python dispatch leaves real *gaps* between kernels. GPU-Util reads a deceptively middling 40–70%, and the naive reaction ("only 60% used, pack more on it") is wrong: the ceiling is the CPU/IO pipeline. The tell is GPU-Util that *rises* when you increase `num_workers` or prefetch depth.
- **Genuinely working.** High on all three.

```
              THE DIAGNOSIS MATRIX
              (values are DCGM PROF fractions, 0.0–1.0)

  GPU-Util  SM_ACTIVE  SM_OCC   TENSOR   DRAM    Power    →  DIAGNOSIS
  ────────  ─────────  ──────   ──────   ────    ─────       ─────────
   ~1.00      ~0.01     ~0.01    ~0.00   ~0.00   ~10%    →  ONE-BLOCK / tiny grid
                                                             "the util lie", pure waste
                                                             FIX: parallelism, or evict

   ~1.00      ~0.9      ~0.15    ~0.02   ~0.10   ~25%    →  LAUNCH-BOUND
                                                             kernels too small/too many
                                                             FIX: fuse, CUDA graphs

   0.4–0.7    0.4–0.7   mid      mid     mid     ~40%    →  HOST-STARVED
                                                             real gaps between kernels
                                                             FIX: dataloader, pinned mem,
                                                                  NUMA (01b.6)

   ~1.00      ~0.95     ~0.5     ~0.05   ~0.85   ~60%    →  MEMORY-BOUND (correct!)
                                                             physics, not a bug
                                                             FIX: batch/fuse (03.2–03.4)

   ~1.00      ~0.98     ~0.5     ~0.75   ~0.4    ~95%    →  COMPUTE-BOUND, working
                                                             this is what you paid for

  Read left to right and stop at the first row that matches. GPU-Util is
  identical (~1.00) in four of the five rows — it carries almost no
  diagnostic information on its own.
```

**GPU-Util is a symptom, never a diagnosis.** You need SM_ACTIVE + SM_OCCUPANCY + TENSOR_ACTIVE + DRAM_ACTIVE together, with power as a free sanity check, to tell "starved of parallelism" from "starved of data" from "correctly memory-bound" from "actually working."

### 9. The fleet recipe: what to alert on and what to bill on

For a multi-cluster fleet the alert that catches waste is the *conjunction*, expressed against `dcgm-exporter`'s Prometheus series:

```promql
# "Busy but idle": switched on, machine floor empty, tensor cores cold — sustained.
(
      avg_over_time(DCGM_FI_PROF_GR_ENGINE_ACTIVE[15m])  > 0.9
  and avg_over_time(DCGM_FI_PROF_SM_ACTIVE[15m])          < 0.2
  and avg_over_time(DCGM_FI_PROF_PIPE_TENSOR_ACTIVE[15m]) < 0.05
)
```

```promql
# Useful-GPU-seconds over a day, for show-back. Integrate the honest counter,
# not the duty cycle. sum_over_time × scrape interval = seconds of real work.
sum by (exported_pod, gpu) (
  sum_over_time(DCGM_FI_PROF_PIPE_TENSOR_ACTIVE[1d]) * 15
)
/
(86400)   # ÷ allocated GPU-seconds → the allocated-vs-utilised gap
```

- **Alert on** the conjunction above, and separately on sustained low SM_ACTIVE for expensive SKUs.
- **Bill / show back on** integrated SM_ACTIVE (general tenants) or TENSOR_ACTIVE (AI tenants). Never bill on GPU-Util: it would score every idle-but-resident GPU as fully used and hide the entire waste story, which is precisely the story chargeback exists to surface.
- **MIG caveat.** On a MIG-partitioned H100 (up to **7 instances**), whole-GPU GPU-Util is meaningless across partitions — it reports the union. Per-instance PROF fields are what tell you whether each *slice* is working. Lesson 03.6 covers MIG's hardware isolation.

### 10. MFU — the one number that ties utilisation to money

**MFU (Model FLOPs Utilisation)** is the end-to-end honesty metric: the useful FLOP/s your model actually achieved, divided by the hardware's theoretical peak FLOP/s.

```
MFU = (useful model FLOP/s achieved) ÷ (peak FLOP/s of the hardware)
```

The interesting half is the numerator, and it should be derived rather than memorised. Every parameter in a dense transformer participates in exactly one multiply and one add per token in the forward pass — a matrix-vector product against the weight is one MAC per weight — so:

```
forward FLOPs per token   = 2 · N            (1 multiply + 1 add per parameter)
backward FLOPs per token  ≈ 4 · N            (gradient wrt input + gradient wrt weight,
                                              each about as expensive as the forward)
─────────────────────────────────────────────────────────────────────────────────
training FLOPs per token  ≈ 6 · N
```

This is the Kaplan et al. convention (*Scaling Laws for Neural Language Models*, arXiv:2001.08361), `C ≈ 6·N·D` for N non-embedding parameters and D training tokens. It deliberately **excludes attention's quadratic term**, which adds roughly `12 · L · S² · d_head · n_heads` per sequence and matters at long context. That omission is why the same run can be quoted at two different MFUs — see the PaLM row below.

Substituting into the definition:

```
MFU = (6 · N_params · tokens_per_sec) ÷ (num_GPUs · peak_FLOP/s_per_GPU)
```

**Worked, so the units are unambiguous.** A 70B-parameter model trained on 512 H100s at an aggregate 3,500 tokens/s/GPU:

```
useful FLOP/s per GPU = 6 × 70e9 params × 3,500 tok/s
                      = 1.47e15 FLOP/s        = 1,470 TFLOP/s   ← impossible, see below
```

That result exceeds the 989 TFLOP/s dense BF16 roof, which tells you the assumed token rate is wrong — a useful property of the formula: **it self-checks.** Invert it to get the real ceiling instead:

```
tokens/s/GPU at 100% MFU = 989e12 ÷ (6 × 70e9) = 2,355 tok/s
tokens/s/GPU at  45% MFU = 0.45 × 2,355        = 1,060 tok/s
tokens/s/GPU at  20% MFU = 0.20 × 2,355        =   471 tok/s
```

The difference between those last two rows — 2.25× — is the *entire* difference in GPU-hours, and therefore in dollars, for identical hardware and identical model quality.

**Published numbers to calibrate against**, with the nuance of what each counts:

| System | MFU | Note |
|---|---|---|
| GPT-3 (as reported in the PaLM paper) | 21.3% | Comparison baseline in Google's PaLM paper |
| Gopher (as reported in the PaLM paper) | 32.5% | Comparison baseline |
| **PaLM 540B** | **46.2%** (attention FLOPs counted) / **45.7%** (not counted) | The canonical "good MFU" citation. The two figures differ *only* by the attention term the `6N` convention drops — always say which you are quoting |
| Llama 3.1 405B pretraining, 16K H100s | ~38–43% (≈40% commonly cited) | A frontier, at-scale run |
| Databricks/MosaicML, FP8 + Composer | >50% | Claimed highest published at the time; lesson 03.5 covers the FP8 mechanics |
| **Meta GEM (ads foundation model, LLM-scale)** | **20–25%** | A real *production* training system on several thousand latest-gen GPUs; doubling MFU was an explicit 12-month engineering goal |

Using **H100 dense BF16 = 989 TFLOP/s** as the denominator (see the sparsity note below), **40–60% MFU is a good, achievable target** for well-tuned large-scale pretraining; 50%+ is excellent; under ~30% is a cost problem worth investigating. 15% means roughly 3× the GPU-hours for the same useful work.

Sit with the Meta GEM number. Even a top-tier engineering organisation runs *production* training at 20–25%, well below the PR-quotable 40%+ from headline pretraining runs. That is a 2–2.5× real dollar spread on identical hardware, and it tells you MFU targets are regime-specific. Establish which regime — from-scratch frontier pretrain, or a continuously-retrained production model — before quoting a "good" number.

A cousin metric, **HFU (Hardware FLOPs Utilisation)**, counts *all* FLOPs the hardware executed, including activation recomputation done to save memory. **MFU ≤ HFU always.** For cost reasoning prefer MFU, because it counts only FLOPs that advance the model — the thing you are actually buying. A run with heavy gradient checkpointing can show 55% HFU and 40% MFU; the 15-point gap is real silicon time spent recomputing.

### 11. The spec table — and the sparsity asterisk

Every ratio in this module divides by a hardware peak, so the peaks have to be right, and they have to be **dense**. NVIDIA's marketing figures are frequently the *structured-sparsity* numbers: a 2× multiplier that assumes weights follow a 2:4 sparse pattern (two zeros in every group of four), which ordinary dense training and inference do not. Quoting sparse as your denominator silently halves your apparent MFU.

| SXM part | A100 80GB | H100 SXM5 80GB | H200 SXM 141GB | B200 SXM |
|---|---|---|---|---|
| Architecture / die | Ampere GA100 | Hopper GH100 | Hopper GH100 (same die) | Blackwell, 2× GB100 dies |
| SMs | 108 | **132** | 132 | **148** (74 enabled per die × 2) |
| FP32 CUDA cores | 6,912 | 16,896 | 16,896 | — |
| Tensor cores | 432 (3rd gen) | 528 (4th gen) | 528 (4th gen) | 5th gen (FP4/FP6) |
| Boost clock | 1,410 MHz | 1,980 MHz | 1,980 MHz | — |
| **FP16/BF16 tensor, dense** | **312 TFLOP/s** | **989 TFLOP/s** | **989 TFLOP/s** | **2,250 TFLOP/s** |
| FP16/BF16, 2:4 sparse | 624 | 1,979 | 1,979 | 4,500 |
| **FP8 tensor, dense** | *not supported* | **1,979 TFLOP/s** | **1,979 TFLOP/s** | **4,500 TFLOP/s** |
| FP8, 2:4 sparse | — | 3,958 | 3,958 | 9,000 |
| FP4 tensor, dense | — | — | — | 9,000 TFLOP/s |
| HBM capacity | 80 GB HBM2e | 80 GB HBM3 | 141 GB HBM3e | 180 GB HBM3e |
| **HBM bandwidth** | **2.039 TB/s** | **3.35 TB/s** | **4.8 TB/s** | **7.7 TB/s** |
| L2 cache | 40 MB | 50 MB | 50 MB | larger, 4 partitions |
| TDP | 400 W | 700 W | 700 W | 1,000 W |

Sources: NVIDIA A100 / H100 / H200 datasheets and the Hopper and Blackwell architecture whitepapers; B200 SM count per the Blackwell microbenchmarking literature (arXiv:2507.10789), which reports 74 of 80 SMs enabled per die across a dual-die package. NVIDIA quotes B200 at *up to* 192 GB / 8 TB/s in some materials; the shipping HGX/DGX B200 configuration is **180 GB at 7.7 TB/s per GPU**, which is the figure to plan against.

**Cross-check the table against the architecture, because that is how you catch a typo in someone's slide.** A100's 312 TFLOP/s dense BF16 across 108 SMs at 1.41 GHz implies

```
312e12 ÷ (108 SM × 1.41e9 Hz) = 2,049 ≈ 2,048 BF16 FLOP per clock per SM
```

— a clean power of two, exactly what a 3rd-gen tensor core delivers. Hopper doubles per-SM tensor throughput to **4,096 BF16 FLOP/clock/SM**, so:

```
132 SM × 4,096 FLOP/clk = 540,672 FLOP/clk
989e12 ÷ 540,672 = 1.83e9 Hz = 1.83 GHz
```

which is *below* the 1.98 GHz max boost clock. That is not an error in the datasheet — it is the honest admission that **a GPU running its tensor cores flat out cannot hold max boost inside a 700 W envelope.** The published peak already bakes in a clock-down. Which is worth knowing before you go looking for the "missing" 8% in your MFU number.

## Perspectives

**Developer / ML-engineer view.** MFU is the single number that says whether a training recipe — parallelism strategy, kernel choices, precision, recompute policy — is well tuned. The PaLM (46.2%/45.7%), Llama 3.1 (~40%), Databricks (>50%) and Meta GEM (20–25%) figures are the calibration points to carry into any "is this run efficient" conversation. Knowing that PaLM's own paper reports *two* MFU numbers depending on whether attention FLOPs are in the numerator is the kind of precision that separates someone who read the paper from someone who memorised a headline.

**Platform / SRE view.** The alerting conjunction (`GR_ENGINE_ACTIVE` high AND `SM_ACTIVE` low AND `TENSOR_ACTIVE` low) is the shape of every real busy-but-idle alert you will write. The counter-bank multiplexing constraint is what stops you from just requesting every field at once, and the dcgm-exporter #662 bug is what turns a theoretically correct alert into a silently broken one. Both are collection-layer facts, invisible from the query language, and both will bite an alert you did not test end to end on the exact driver/DCGM pair you run.

**Hardware view.** The lie is structural, not a monitoring bug. NVML was designed as coarse power/thermal/duty-cycle telemetry for a driver that has to work on a laptop GPU and a datacenter part alike; it was never a performance-counter API. The 132 independent compute islands, each with four independently scheduling sub-partitions, are a *physical* fact — nothing at the NVML layer has visibility into how many of them are populated. That is exactly the gap DCGM's PROF fields exist to fill, and why they need a profiling session and cost something to collect.

**Economics / FinOps view.** Meta's 20–25% against Databricks' 50%+ is a 2–2.5× dollar spread on an identical hardware bill, driven purely by software efficiency rather than SKU choice. Google Cloud's 4.4%-utilisation B200 benchmark makes the sharper point: **low utilisation is not automatically a problem.** At batch-1 decode the workload is fundamentally memory-bound (next lesson's vocabulary) and low FLOPS-util is the physically correct outcome — the business metric (≈1M tokens/sec served in aggregate) came from batching across many concurrent streams, not from raising any single stream's utilisation. Knowing when low utilisation is waste versus physics is the FinOps judgement this whole module builds toward.

## Real-world use cases

- **["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — Google Cloud (Medium, official publication).** A benchmark on GKE Autopilot, 96×B200, serving roughly 1M tok/s of decode traffic: **4.4% FLOPS utilisation, 10.9% memory-bandwidth utilisation, tensor cores active 1.5% of the time** — while GPU-Util would read near 100%. The piece frames this explicitly as roofline physics, not a bug, and states decode sits ~292× below B200's ridge point. That multiple is checkable from the spec table above: `2,250 TFLOP/s ÷ 7.7 TB/s = 292 FLOP/byte`, against decode's ≈1 FLOP/byte. What it shows: the lie and its correct diagnosis on current production hardware, with a number you can re-derive yourself.
- **NVIDIA/dcgm-exporter GitHub, [issue #662](https://github.com/NVIDIA/dcgm-exporter/issues/662).** A filed bug where `DCGM_FI_PROF_GR_ENGINE_ACTIVE` returns 0 until a separate profiling session is run. What it shows: the "honest" metrics have their own collection failure modes — verify the exporter, do not trust the field name.
- **CoreWeave, ["HPC Verification" docs](https://docs.coreweave.com/docs/platform/fleet-management/hpc-verification).** CoreWeave's hourly, per-idle-node hardware-validation framework: 20–30 minute test suites including tensor-core benchmarking at FP8/FP16/BF16 and all-SMs-at-100% thermal checks before a node is handed back to a customer. What it shows: fleet-scale validation built on exactly the granular metrics this lesson teaches, at the company this module's job hook is drawn from — and the point that a *synthetic* all-SM load is how you prove the hardware is healthy, separate from whether tenants are using it well.
- **Meta Engineering, ["GEM Training: How Meta Doubled the Efficiency of Its LLM-Scale Ads Foundation Model"](https://engineering.fb.com/2026/08/03/ml-applications/training-gem-at-llm-scale-meta-ads-recommendation-foundation-model/).** MFU (20–25%) as the operating metric for a real production training system, with doubling it as an explicit engineering roadmap item. What it shows: MFU-chasing is a job function with a budget behind it, and "good" MFU is regime-dependent.

## Worked example

You rent one H100 SXM (on-demand roughly $3/GPU-hour as a 2026-era snapshot — verify current pricing; lesson 03.7 covers sourcing live rates) and deliberately construct the lie.

**Step 1 — build a workload that cannot use the machine.**

```python
# util_lie.py — one block, one warp, forever.
import torch, time

# A tensor small enough that PyTorch launches a single small grid.
x = torch.ones(256, device="cuda", dtype=torch.float32)

torch.cuda.synchronize()
t0 = time.time()
while time.time() - t0 < 300:          # run 5 minutes so DCGM has many samples
    x = x + 1.0                        # elementwise add: no tensor cores, tiny grid
    # deliberately NOT synchronising: keeps the launch queue full,
    # so a kernel is resident at every NVML sample point.
```

**Step 2 — capture both meters against the same wall clock.**

```bash
# Terminal 1 — the coarse, free counter.
$ nvidia-smi --query-gpu=timestamp,utilization.gpu,utilization.memory,power.draw \
             --format=csv -l 1

# Terminal 2 — the honest counters. -e takes field IDs; -d is the delay in ms.
$ dcgmi dmon -e 1001,1002,1003,1004,1005 -d 1000
```

**Step 3 — read the two transcripts side by side.** Representative output, not a captured run — reproduce it yourself in Practice:

```
# nvidia-smi
timestamp,             utilization.gpu [%], utilization.memory [%], power.draw [W]
2026/08/17 10:22:01.0, 100 %,               3 %,                    88.4 W
2026/08/17 10:22:02.0, 100 %,               3 %,                    88.1 W
2026/08/17 10:22:03.0, 100 %,               2 %,                    87.9 W

# dcgmi dmon -e 1001,1002,1003,1004,1005
#Entity  GRACT   SMACT   SMOCC   TENSO   DRAMA
GPU 0    0.998   0.008   0.002   0.000   0.011
GPU 0    0.997   0.008   0.002   0.000   0.010
GPU 0    0.998   0.008   0.002   0.000   0.011
```

Line by line:

| Meter | Reading | What it means here |
|---|---|---|
| `utilization.gpu` | **100%** | A kernel was resident at every sample point. Duty cycle pinned. |
| `GRACT` (1001) | 0.998 | Confirms the same thing from the engine's side. |
| `SMACT` (1002) | **0.008** | ≈1 of 132 SMs ever had a warp assigned → `1/132 = 0.0076`. Matches to within noise. |
| `SMOCC` (1003) | 0.002 | Warp slots used GPU-wide: a couple of warps out of 8,448. |
| `TENSO` (1004) | **0.000** | The 528 tensor cores — the reason an H100 costs what it does — never issued once. |
| `DRAMA` (1005) | 0.011 | HBM essentially idle; the working set fits in L1. |
| `power.draw` | **88 W / 700 W** | 12.6% of the power cap. The free cross-check: no joules, no FLOPs. |

**Step 4 — turn it into money.**

```
Rented:        1 × H100 SXM @ ~$3.00 / GPU-hour
Peak capability: 989 TFLOP/s dense BF16
Achieved:      ~256 FP32 adds per kernel launch; call it < 1 GFLOP/s sustained
Implied MFU:   < 1e9 / 989e12  ≈  0.0001 %

Waste rate:    ~$3.00/hr × (1 − 0.000001) ≈ $3.00/hr burned
Over a month:  ~$2,160 for one GPU producing nothing
Across 8 GPUs
on one node:   ~$17,300/month, and every GPU-Util dashboard shows the node GREEN.
```

**Step 5 — the contrast case, so you can tell waste from physics.** Google Cloud's 96×B200 benchmark shows the *same* GPU-Util reading with a completely different verdict. There, decode at low batch has arithmetic intensity ≈1 FLOP/byte against B200's ridge point of `2,250e12 ÷ 7.7e12 = 292 FLOP/byte` — 292× below. Tensor-active 1.5%, FLOPS-util 4.4%, bandwidth-util 10.9%; and the cluster still served ~1M tokens/sec. Low utilisation there is the *correct ceiling* for that arithmetic intensity, and the lever is architectural (batching, disaggregation — lesson 03.4), not "make the number go up."

The two cases have identical GPU-Util and opposite conclusions. **That is the whole reason you need the other four counters.**

## Practice

**Task (rent one H100 or A100 by the hour; budget ~1 hr ≈ $2–4). Keep the raw logs — you will reuse this same rented session across lessons 01–06.**

1. **Install the tooling.** Install DCGM (`datacenter-gpu-manager`) and either run `dcgmi dmon` directly or stand up `dcgm-exporter` and scrape it. Confirm the driver, CUDA, and DCGM versions with `nvidia-smi`, `nvcc --version`, and `dcgmi --version`, and record them in the log — every number below is version-conditional.
2. **Print the counter groups for your silicon.** Run `dcgmi profile -l`. Record which groups `SM_ACTIVE`, `SM_OCCUPANCY` and `PIPE_TENSOR_ACTIVE` fall into on *your* GPU. If they are in different groups, note that your simultaneous readings are multiplexed, not concurrent.
3. **Run the busy-but-idle workload** (the loop above, or a single-block CUDA kernel) so GPU-Util pins near 100%.
4. **Capture both meters with timestamps into one log**: `utilization.gpu`, `utilization.memory`, `power.draw` from `nvidia-smi`, and fields 1001–1005 from `dcgmi dmon`. Interleave or post-join them on timestamp.
5. **Sanity-check for the collection gotcha.** Before believing the readings, confirm `DCGM_FI_PROF_GR_ENGINE_ACTIVE` is non-zero and *moving* — start a second `dcgmi dmon` session and see whether the values change (the #662 failure mode).
6. **Run a contrast workload** for 60 seconds: a large square BF16 `torch.matmul` (N ≈ 8192) in a loop. Capture the same fields. This gives you the "actually working" row of the diagnosis matrix on your own hardware.
7. **Compute SM_ACTIVE by hand for the single-block case** — `1 ÷ (SM count of your SKU)` — and check it against the measured value. Note the SM count you used and where you got it.

**Acceptance:** a single artifact (log plus one annotated table or screenshot) in which GPU-Util and SM/occupancy/tensor-active are shown **side by side for the same instant**, for *both* the trivial kernel and the GEMM, with:

- a one-sentence caption for the trivial case stating why the GPU is "busy but idle";
- the hand-computed `1/N_SMs` prediction next to the measured SM_ACTIVE;
- the power draw for both cases, as the free corroborating signal;
- the driver/CUDA/DCGM versions and the `dcgmi profile -l` group listing.

This is Exhibit A of the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md).

## Common pitfalls

1. **Believing 100% GPU-Util during decode is automatically a code smell.** Usually it is not — the Google Cloud benchmark makes the "this is physics, not a bug" case for low-batch decode. Establish which roof you are under (lesson 03.2) before calling it waste. *Mechanism:* GPU-Util saturates on residency, which a correctly-memory-bound kernel also achieves.
2. **Reading SM_ACTIVE = 0.5 as "half the SMs are always busy."** It is ambiguous by construction — the field averages a per-SM ratio over SMs, so "all SMs half the time" and "half the SMs all the time" produce the same number. Pair with SM_OCCUPANCY and TENSOR_ACTIVE.
3. **Reading SM_ACTIVE = 1.0 as "the SMs are working."** SM_ACTIVE counts cycles with ≥1 warp *assigned*, and a warp stalled 466 cycles on an HBM load is assigned for all 466. High SM_ACTIVE with near-zero TENSOR_ACTIVE and near-zero power headroom used is the signature of stalled residency, not work.
4. **Trusting a DCGM dashboard without checking the field is exporting.** The #662 bug shows a field silently reading 0 because of a profiling-session conflict, not because the GPU is idle. Test the alert against a known-busy GPU before you trust it against a suspected-idle one.
5. **Requesting incompatible PROF fields and believing the timestamps.** Metrics in different counter groups are multiplexed; the values you see for the same timestamp were sampled in different, shorter windows. Fine for trend lines, wrong for instant-by-instant correlation. `dcgmi profile -l` tells you which is which.
6. **Quoting the sparse TFLOPS figure (e.g. 1,979 BF16 on H100) as your MFU denominator.** This silently doubles your apparent efficiency. Always confirm dense vs 2:4-sparse before dividing.
7. **Assuming one "good MFU" applies everywhere.** Meta's production ads-training system runs at 20–25% while headline pretraining runs claim 40%+. Establish the regime before quoting a target.
8. **Comparing MFU and HFU across teams without saying which you computed.** MFU ≤ HFU always; a team using heavy activation recomputation can quote an HFU that looks 10–15 points better than their MFU for the same run.

## Self-check

- **Why can a single-block kernel report 100% GPU-Util?** GPU-Util is a *temporal duty cycle* — NVML defines it as the percent of the sample period (1 s to 1/6 s depending on the product) during which one or more kernels was executing. A block is placed on exactly one SM and stays there, so with a loop there is always a kernel resident and every sample point finds the GPU non-idle. The counter measures *time non-idle*, not SMs engaged (1 of 132 on H100), not warp slots filled (1 of 8,448), and not FLOP/s. It saturates while the machine is ~99.99% idle by warp-slot occupancy.
- **What does GPU-Util measure, in one sentence, without using the word "utilisation"?** The fraction of sampled instants in the last window at which at least one kernel was resident on the GPU.
- **Which metric would you fleet-alert on to catch a busy-but-idle GPU, and what must you verify before trusting it?** `DCGM_FI_PROF_SM_ACTIVE` (field 1002, "ratio of cycles an SM had ≥1 warp assigned," averaged over all SMs) as the "is the floor running" signal, plus `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (1004) for AI tenants to confirm the tensor cores are engaged. Alert on the *conjunction* — high GR_ENGINE_ACTIVE with low SM_ACTIVE and low TENSOR_ACTIVE — never on GPU-Util alone. Before trusting it: (a) confirm the field is actually populating and moving, not stuck at 0 from a stale/conflicting profiling session (dcgm-exporter #662); (b) run `dcgmi profile -l` to see whether the fields you want share a counter group, because fields in different groups are multiplexed rather than sampled concurrently.
- **Why is high SM_OCCUPANCY not the same as high throughput?** Occupancy is resident warps ÷ 64 warp slots — how *full* the scheduler's slots are, not how often it issues. A memory-bound kernel can sit at 100% occupancy with every warp stalled on a ~466-cycle HBM load, so the ALUs idle while the slots are full. Conversely a well-tiled GEMM can reach near-peak FLOP/s at ~50% occupancy because each warp carries enough independent work to hide latency by itself. Occupancy is an input to latency hiding; issue rate and pipe-active counters are the output.
- **Define MFU, derive the numerator, and give a target with two real reported numbers.** MFU = useful model FLOP/s achieved ÷ theoretical peak FLOP/s. The numerator uses `6 · N_params · tokens/sec` for training: `2N` for the forward pass (one multiply and one add per parameter), `≈4N` for the backward pass (gradients with respect to both inputs and weights), per the Kaplan et al. `C ≈ 6ND` convention, which excludes attention's quadratic term. A good target is 40–60% for well-tuned large-scale pretraining. PaLM reported 46.2% with attention FLOPs counted / 45.7% without; Meta's production GEM system runs at 20–25% — so "good" is regime-dependent. Always divide by the *dense* peak (989 BF16 TFLOP/s on H100), never the 2:4-sparse marketing figure.
- **A GPU shows GPU-Util 100%, SM_ACTIVE 0.9, SM_OCCUPANCY 0.15, TENSOR_ACTIVE 0.02, power 180 W of 700 W. Diagnosis?** Launch-bound / tiny-kernel storm. The SMs are receiving work (SM_ACTIVE high) but grids drain before filling the warp slots (occupancy low) and no tensor math is happening (TENSOR_ACTIVE ~0), with power far below cap confirming little real arithmetic. The fix is operator fusion / `torch.compile` / CUDA graphs to make fewer, bigger kernels — not a bigger GPU, which would run the same overhead at a higher rate.
- **Why can Google's B200 benchmark show 4.4% FLOPS utilisation while serving ~1M tokens/sec, and why is that not a bug?** Decode at low batch has arithmetic intensity ≈1 FLOP/byte, against B200's ridge point of `2,250 TFLOP/s ÷ 7.7 TB/s = 292 FLOP/byte` — about 292× below it. The workload is therefore fundamentally memory-bound and low FLOPS-utilisation is the physically correct ceiling for a single stream, not waste. The aggregate throughput comes from batching many concurrent decode streams against the same fixed bandwidth budget. The lever is architectural (batching, disaggregation), not chasing a utilisation number.

## Connections & what's next

This lesson supplies every vocabulary term the rest of the module depends on: `DRAM_ACTIVE`, `TENSOR_ACTIVE` and the notion of a hardware "roof" feed directly into lesson 03.2's roofline model; the spec table's dense TFLOP/s and TB/s figures are the inputs to every ridge-point and ceiling calculation from here on; MFU as a ratio reappears as the cost-per-token denominator in the 03.7 capstone; and the DCGM fields you learned to read here are exactly the ones the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md) asks you to capture across every subsequent lesson's practice. The economics thread — a "busy" GPU can burn money doing nothing useful — is refined into a sharper cost argument in every later lesson.

Next: **[03.2 · Compute-bound vs memory-bound and the roofline](02-compute-vs-memory-bound-roofline.md)** takes "TENSOR_ACTIVE is low" and "DRAM_ACTIVE is high" from this lesson's metric table and turns them into a formal, predictive model that tells you *in advance*, before you spend a dollar, whether a workload even has room to get faster.

## References & further reading

**Primary sources**

1. [NVIDIA CUDA C++ Programming Guide — "Compute Capabilities" appendix](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#compute-capabilities) — the canonical technical-specifications-per-compute-capability table reproduced in §4 (warp size 32, 64 resident warps/SM, 32 blocks/SM, 65,536 registers/SM, 255 registers/thread, 164 KB shared/SM on CC 8.0 and 228 KB on CC 9.0), plus the thread-block-cluster semantics (portable maximum 8 blocks) introduced at CC 9.0.
2. [NVIDIA CUDA C++ Best Practices Guide](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html) — the occupancy and latency-hiding chapter behind §5: why the warp scheduler switches to another resident warp rather than waiting, and how occupancy relates to (but does not equal) throughput.
3. [NVIDIA H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — source for the GH100 SXM5 organisation used throughout: 8 GPCs, 66 TPCs, **132 SMs**, 128 FP32 cores and 4 fourth-gen tensor cores per SM (16,896 / 528 per GPU), 5 HBM3 stacks, 50 MB L2, and the dense/sparse TFLOP/s figures. Read the fine print specifically to separate dense from with-sparsity numbers before any MFU math.
4. [NVIDIA DCGM documentation — Feature Overview and Profiling](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html) — canonical definitions of the `DCGM_FI_PROF_*` fields (IDs 1001–1013) in §7, including SM_ACTIVE as "ratio of cycles an SM has at least 1 warp assigned" and SM_OCCUPANCY as "ratio of resident warps to the maximum," plus the multi-pass/counter-group constraint and its V100-vs-T4 example.
5. [NVML / `nvidia-smi` documentation](https://docs.nvidia.com/deploy/nvidia-smi/index.html) — the `utilization.gpu` definition decoded in §6, including the "sample period may be between 1 second and 1/6 second depending on the product" clause, and the separate `utilization.memory` field (a bandwidth duty cycle, not a capacity figure).
6. Kaplan et al., ["Scaling Laws for Neural Language Models"](https://arxiv.org/abs/2001.08361) (arXiv:2001.08361) — the `C ≈ 6ND` FLOP-accounting convention derived in §10, including the explicit note that it counts non-embedding parameters and excludes the attention term.
7. Chowdhery et al., "PaLM: Scaling Language Modeling with Pathways" (arXiv:2204.02311) — source of the 46.2% / 45.7% MFU pair (with and without attention FLOPs) and the GPT-3 (21.3%) and Gopher (32.5%) comparison points, and the paper that popularised MFU as the reporting standard.
8. Luo et al., ["Benchmarking and Dissecting the Nvidia Hopper GPU Architecture"](https://arxiv.org/abs/2402.13499) — measured memory-hierarchy latencies used in §5 (A100: shared ≈29, L1 ≈38, L2 ≈262, global ≈466 cycles; L2 ≈6.5× L1, global ≈1.9× L2 across the parts tested). The independent microbenchmark counterweight to vendor documentation.

**Real-world engineering blogs**

9. Google Cloud, ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — the 96×B200 production benchmark: 4.4% FLOPS-util, 10.9% bandwidth-util, 1.5% tensor-active, ~292× below the ridge point, ~1M tok/s served. This lesson's contrast case for "low utilisation as physics, not waste."
10. Meta Engineering, ["GEM Training: How Meta Doubled the Efficiency of Its LLM-Scale Ads Foundation Model"](https://engineering.fb.com/2026/08/03/ml-applications/training-gem-at-llm-scale-meta-ads-recommendation-foundation-model/) — production MFU of 20–25% and an explicit efficiency-doubling goal; the regime-dependence counterpoint to headline pretraining MFU.
11. CoreWeave, ["HPC Verification" docs](https://docs.coreweave.com/docs/platform/fleet-management/hpc-verification) — hourly per-node validation including tensor-core benchmarking and all-SM thermal checks, grounded in the same granular metrics this lesson teaches.
12. NVIDIA/dcgm-exporter GitHub, [issue #662](https://github.com/NVIDIA/dcgm-exporter/issues/662) — the filed bug where a trusted DCGM field silently reports 0 until another profiling session runs; the concrete instance behind the "verify your exporter" pitfall.

**Deeper dives**

13. Modal, ["I paid for the whole GPU, I am going to use the whole GPU"](https://modal.com/blog/gpu-utilization-guide) — a practitioner walk-through of why GPU-Util lies and which DCGM fields to trust; useful production framing for the deliverable.
14. ["Measuring GPU utilization one level deeper"](https://arxiv.org/html/2501.16909v1) — a rigorous treatment of GPU-Util versus SM occupancy versus tensor-pipe activity; the reference to reach for when you need to make the "one metric, four granularities" argument defensible under pushback.
