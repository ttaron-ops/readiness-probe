---
lesson: "03.2"
title: "Compute-bound vs memory-bound and the roofline"
module: "03"
concept: "Compute-bound vs memory-bound and the roofline"
status: not-started
est_time: "8h"
prev: "01-execution-model-and-utilisation.md"
next: "03-memory-hierarchy-hbm.md"
artifacts: []
sources: 13
---

# 03.2 · Compute-bound vs memory-bound and the roofline

> **Concept.** Compute arithmetic intensity, place a workload under the compute roof or the memory roof, find the ridge point — the single lens for the whole cost story.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 03.1 gave you the vocabulary to distrust `GPU-Util` and to read `SM_ACTIVE`, `SM_OCCUPANCY`, `PIPE_TENSOR_ACTIVE` and `DRAM_ACTIVE` as four separate, honest questions about a running GPU. It also gave you the spec table: dense TFLOP/s and TB/s for A100, H100, H200 and B200. What it did not give you is a way to *predict* — before you run anything, before you rent anything — whether a workload has room to get faster, or which of those DCGM fields you should even expect to be high.

This lesson supplies that. The roofline model turns "is my GPU busy" into "which physical ceiling is this specific workload up against, and can a bigger GPU help at all." It is the module's anchor: everything from lesson 03.3 (memory hierarchy) through the 03.7 capstone is this model applied to a specific workload shape. By the end you will be able to compute an arithmetic intensity from a kernel description on a whiteboard, place it against a machine balance you also computed, and name the single lever that can possibly move it.

## Why this matters

Every GPU cost decision reduces to one question: **which roof am I under, and how close to it am I?** If a workload is *memory-bound*, a faster-compute GPU buys nothing — you pay more per hour for zero speedup. If it is *compute-bound and far from the roof*, the fix is software (fusion, batching, precision), not a bigger SKU. If it is *near the roof*, you are done at that layer and the remaining win is a different algorithm or a different machine.

This is precisely the competence NVIDIA's *Solutions Architect* job description names: "inference optimization via FP16/INT8/FP8… arithmetic intensity vs peak compute and memory bandwidth using the roofline model to classify bottlenecks." An interviewer asking *"100% util but terrible throughput — why?"* or *"why is decode memory-bandwidth bound and prefill compute-bound?"* is asking you to run this lesson's decision procedure live.

Get it wrong in the expensive direction and you propose a six- or seven-figure hardware refresh that changes nothing. Concretely: swapping a fleet from H100 to B200 raises dense BF16 compute by 2.27× (989 → 2,250 TFLOP/s) but bandwidth by only 2.30× (3.35 → 7.7 TB/s). For a workload sitting on the memory roof, you get the bandwidth ratio and *none* of the compute ratio — and if you had instead bought a same-generation part with more bandwidth and identical compute (H100 → H200: compute ×1.00, bandwidth ×1.43), you would have paid for exactly the thing that moved. Being able to say which of those two upgrades applies, with the arithmetic on the board, is the job.

## What's new here (calibration)

The module README is explicit: you are not here to hand-derive rooflines per kernel, build cache-level hierarchical rooflines for tuning, or squeeze the last 5% via kernel scheduling. That is CUDA-kernel-developer depth. What this lesson adds:

- **The formal roofline model** (Williams, Waterman & Patterson, CACM 2009) derived rather than quoted — including *why* `min(peak, AI × BW)` is the right functional form, and what the original paper's "ceilings" idea adds that the two-roof cartoon leaves out.
- **Machine balance computed for every SKU this module targets**, at every precision, so you have the ridge points memorised as a small table rather than one number.
- **Actual arithmetic-intensity derivations** — GEMM, elementwise, attention prefill (fused and unfused), attention decode, and the weight-streaming GEMM that decode really is — carried through to the numbers, so you can do a new one you have never seen.
- **The critical batch size falling out of the algebra**, which is the single most useful derived quantity in inference serving and the bridge into lesson 03.4.
- **The practitioner's third regime** (Horace He's compute/memory/overhead trichotomy), because the pure two-roof chart cannot depict overhead-bound at all — it is not a function of arithmetic intensity.
- **How to measure AI on real hardware** rather than only estimate it, using the DCGM fields from lesson 03.1 plus Nsight Compute.

## Core concepts

### 1. The problem: "how fast will this run?" has two answers

Take a kernel. It performs some number of floating-point operations, and it moves some number of bytes between the GPU's compute units and its off-chip DRAM. Two independent, physical facts bound how quickly it can finish:

- The chip can retire at most **P** floating-point operations per second. On an H100 SXM5 that is 989 × 10¹² FLOP/s in dense BF16 through the tensor cores, or 67 × 10¹² FLOP/s if the kernel only uses the FP32 CUDA cores.
- The memory system can move at most **B** bytes per second. On the same H100 that is 3.35 × 10¹² B/s from HBM3.

These bounds are *simultaneous*, not alternatives. The kernel needs both resources, and it finishes no sooner than the slower of the two allows:

```
t ≥ FLOPs / P            (compute time floor)
t ≥ Bytes / B            (memory time floor)
⇒ t ≥ max( FLOPs/P , Bytes/B )
```

Divide FLOPs by that time to get attainable performance:

```
                     FLOPs                 FLOPs
Attainable FLOP/s = ─────── = ────────────────────────────────
                       t       max( FLOPs/P , Bytes/B )

                  = min( P , (FLOPs/Bytes) × B )
```

The quantity `FLOPs / Bytes` is doing all the work, and it deserves its own name.

### 2. Arithmetic intensity — and what "bytes" means

```
Arithmetic Intensity (AI) = FLOPs performed ÷ bytes moved to and from DRAM
                          [ FLOP / byte ]
```

Williams, Waterman and Patterson call it **operational intensity** and define it precisely as "the number of floating-point operations per byte of data traffic *between the processor and off-chip DRAM*." That qualification is the whole subtlety:

- **It is DRAM traffic, not total traffic.** Bytes served from L1 or L2 do not count. Two implementations of the same mathematical operation can have wildly different AI purely because one of them keeps data on-chip and the other does not.
- **It is therefore a property of the algorithm *and* its implementation *and* the cache size** — not of the algorithm alone. This is why "fusion raises arithmetic intensity" is a real mechanism and not a slogan: fusing two kernels removes an intermediate round-trip to HBM, which shrinks the denominator while leaving the numerator alone.
- **It says nothing about the machine.** AI is measured in FLOP/byte; the machine's capability is measured in FLOP/s and byte/s. The model's entire content is the comparison between them.

High AI means each byte dragged across the bus is amortised over lots of arithmetic, so the bus can keep the ALUs fed. Low AI means the ALUs starve.

### 3. The roofline, the ridge point, and machine balance

Plot AI on the x-axis (log) against attainable FLOP/s on the y-axis (log). `min(P, AI × B)` becomes a **slanted line of slope 1** (the memory roof, `AI × B`, which on log-log axes is a 45° line whose height is set by B) rising until it meets a **flat line** (the compute roof, P). Where they intersect is the **ridge point**:

```
Ridge-point AI = P ÷ B   [ FLOP/byte ]
```

The paper calls the ridge point's x-coordinate "the minimum operational intensity required to achieve maximum performance," and it is also known as the machine's **balance**: the arithmetic intensity at which the machine's two capabilities are exactly matched.

**Interpretation you can quote:** on an H100 in dense BF16 you need ~295 FLOPs of work per byte of HBM traffic *just to stop being memory-bound*. Below that, no kernel however perfect reaches the compute roof. Above it, you *can* be compute-bound — nothing guarantees it, because a bad kernel underperforms both roofs.

**The historical grounding is worth ten seconds.** The 2009 paper's lead example is a 2.2 GHz dual-core AMD Opteron X2 2214 in a dual-socket system: 17.6 GFLOP/s peak double precision, ~15 GB/s stream bandwidth, ridge point ≈ **1.0 FLOP/byte**. Its successor, the four-core Opteron X4, doubled per-core floating-point issue rate and moved the ridge to **4.4 FLOP/byte** — the paper's own headline observation that the ridge point marches right as core counts grow faster than memory systems.

Now put that next to today:

| Machine | Peak (relevant precision) | Bandwidth | Ridge point (FLOP/byte) |
|---|---|---|---|
| Opteron X2 2214 (2006, dual socket) | 17.6 GFLOP/s FP64 | 15 GB/s | **~1.0** |
| Opteron X4 (2008) | — | — | **4.4** |
| V100 SXM2 (2017) | 125 TFLOP/s FP16 tensor | 900 GB/s | **~139** |
| A100 SXM 80GB | 312 TFLOP/s BF16 dense | 2.039 TB/s | **153** |
| H100 SXM5 | 989 TFLOP/s BF16 dense | 3.35 TB/s | **295** |
| H200 SXM | 989 TFLOP/s BF16 dense | 4.8 TB/s | **206** |
| B200 SXM | 2,250 TFLOP/s BF16 dense | 7.7 TB/s | **292** |

**In twenty years the ridge point moved from 1 to ~300 FLOP/byte.** That single trend is why "memory-bound" is the default condition of modern accelerated computing and why so much of ML systems engineering is data-movement engineering. It is also why the H200 row is interesting: it is the only entry that moves the ridge point *left* relative to its predecessor, because NVIDIA raised bandwidth while holding compute fixed. A lower ridge point means *more* workloads land in the compute-bound region, i.e. more of them can actually use the FLOPs you own.

### 4. Machine balance for every SKU and precision — the table to memorise

The ridge point is not one number per GPU; it is one number per *(GPU, precision)* pair, because lowering precision multiplies compute without touching bandwidth. All figures are **dense** (not 2:4 sparse), from the spec table in lesson 03.1.

| SKU | Precision | Dense peak | HBM BW | **Ridge (FLOP/byte)** |
|---|---|---|---|---|
| A100 SXM 80GB | FP32 (CUDA cores) | 19.5 TFLOP/s | 2.039 TB/s | 9.6 |
| A100 SXM 80GB | TF32 tensor | 156 TFLOP/s | 2.039 TB/s | 76.5 |
| A100 SXM 80GB | **FP16/BF16 tensor** | 312 TFLOP/s | 2.039 TB/s | **153** |
| A100 SXM 80GB | INT8 tensor | 624 TOP/s | 2.039 TB/s | 306 |
| H100 SXM5 | FP32 (CUDA cores) | 67 TFLOP/s | 3.35 TB/s | 20 |
| H100 SXM5 | TF32 tensor | 495 TFLOP/s | 3.35 TB/s | 148 |
| H100 SXM5 | **FP16/BF16 tensor** | 989 TFLOP/s | 3.35 TB/s | **295** |
| H100 SXM5 | **FP8 tensor** | 1,979 TFLOP/s | 3.35 TB/s | **591** |
| H200 SXM | FP16/BF16 tensor | 989 TFLOP/s | 4.8 TB/s | **206** |
| H200 SXM | FP8 tensor | 1,979 TFLOP/s | 4.8 TB/s | 412 |
| B200 SXM | FP16/BF16 tensor | 2,250 TFLOP/s | 7.7 TB/s | **292** |
| B200 SXM | FP8 tensor | 4,500 TFLOP/s | 7.7 TB/s | 584 |
| B200 SXM | FP4 tensor | 9,000 TFLOP/s | 7.7 TB/s | 1,169 |

Two readings that pay off later:

- **Lower precision raises the ridge point.** FP8 doubles compute and leaves bandwidth alone, so the bar to *stay* compute-bound doubles: 295 → 591 on H100. A kernel that is comfortably compute-bound in BF16 can flip to memory-bound in FP8. This is the load-bearing bridge to lesson 03.5: FP8 does not speed up a memory-bound kernel by 2×, and now you can say exactly why.
- **The B200 BF16 ridge (292) is the number in Google Cloud's benchmark.** They report decode sitting "292× below" the ridge point; decode's AI is ≈1 FLOP/byte, and `2,250e12 ÷ 7.7e12 = 292`. The multiple and the ridge point are the same number because decode's intensity happens to be 1. Re-deriving a vendor's figure from first principles is exactly the habit this module is trying to build.

### 5. Deriving arithmetic intensity — six workloads, carried through

This is the section to be able to reproduce cold. All figures assume FP16/BF16 (2 bytes per element) unless stated.

**(a) Dense GEMM, C = A·B.** A is M×K, B is K×N, C is M×N.

```
FLOPs = 2 · M · N · K              (one multiply + one add per (m,n,k) triple)
Bytes = 2 · (M·K + K·N + M·N)      (read A, read B, write C; 2 B/element)

Square case M=N=K:
  AI = 2N³ ÷ (2 · 3N²) = N/3   FLOP/byte
```

| N | AI (FLOP/byte) | vs H100 BF16 ridge (295) |
|---|---|---|
| 512 | 171 | below → memory-bound |
| 1,024 | 341 | just above |
| 4,096 | 1,365 | 4.6× above → compute-bound |
| 8,192 | 2,731 | 9.3× above → firmly compute-bound |

**AI grows linearly with N.** That is the arithmetic reason bigger matmuls run closer to peak, and the reason "batch more" is the universal first answer in ML performance work.

**The critical caveat:** `N/3` assumes each matrix is read from DRAM *exactly once*, which requires blocking the computation so that tiles stay resident in shared memory and registers. A naive triple loop re-reads a row of A and a column of B for every output element, giving `Bytes ≈ 2·(2N³ + N²)` and `AI ≈ 0.5 FLOP/byte` — memory-bound, at N=8192, by a factor of 5,000. **Same mathematics, same FLOPs, 5,000× difference in arithmetic intensity, entirely from data reuse.** This is why AI is a property of the implementation and why cuBLAS exists.

**(b) Elementwise add, `C = A + B`, N elements.**

```
FLOPs = N
Bytes = 3N elements × 2 B = 6N
AI    = N ÷ 6N = 0.167 FLOP/byte
```

**(c) SAXPY, `y = a·x + y`, FP32, N elements.**

```
FLOPs = 2N                        (one multiply, one add)
Bytes = 3N × 4 B = 12N            (read x, read y, write y)
AI    = 2N ÷ 12N = 0.167 FLOP/byte
```

Both sit ~1,770× below the H100 BF16 ridge. Their attainable performance is:

```
0.167 FLOP/byte × 3.35e12 B/s = 5.6e11 FLOP/s = 0.56 TFLOP/s
                              = 0.057 % of the 989 TFLOP/s compute roof
```

Nothing is wrong with these kernels. **0.56 TFLOP/s *is* the ceiling at that arithmetic intensity.** A faster-compute GPU raises a roof they never touch.

**(d) Attention prefill, fused (FlashAttention-style), one head, one layer.** Sequence length S, head dimension d.

```
FLOPs:  Q·Kᵀ  = 2 · S² · d
        P·V   = 2 · S² · d
        total ≈ 4 · S² · d

Bytes (fused — the S×S score matrix never touches HBM):
        read Q, K, V  = 3 · S · d elements
        write O       = 1 · S · d elements
        total         = 4 · S · d × 2 B = 8 · S · d

AI = 4S²d ÷ 8Sd = S/2   FLOP/byte
```

**AI depends only on sequence length**, and grows linearly with it. At S = 590 you are exactly at H100's BF16 ridge; at S = 4,096, AI = 2,048 FLOP/byte — 7× above it, firmly compute-bound. **This is the formal statement of "prefill is compute-bound."**

**(e) Attention prefill, unfused (the pre-FlashAttention baseline).** Now the S×S score matrix is written to HBM and read back, twice (once for softmax, once for the P·V matmul):

```
Bytes ≈ 8·S·d + (≈4 round trips of S² elements × 2 B) ≈ 8·S·d + 16·S²
For S=4096, d=128:   8·4096·128 = 4.2 MB
                     16·4096²    = 268 MB          ← dominates completely
AI ≈ 4S²d ÷ 16S² = d/4 = 32 FLOP/byte
```

**32 FLOP/byte, versus 2,048 fused.** The same attention, the same FLOPs, a 64× difference in arithmetic intensity, moving the kernel from firmly compute-bound to firmly memory-bound. That is the entire FlashAttention result stated as a roofline movement: Dao et al. describe the mechanism as IO-awareness — tiling and online softmax so the score matrix never materialises in HBM. Fusion is not a micro-optimisation; **it is a change of which roof you are under.**

**(f) Attention decode, one new token against a KV cache of length S.** One query vector, S cached keys and values.

```
FLOPs:  q·Kᵀ = 2 · S · d
        p·V  = 2 · S · d
        total = 4 · S · d

Bytes:  read K cache + V cache = 2 · S · d elements × 2 B = 4 · S · d
        (q and the output are O(d) — negligible)

AI = 4Sd ÷ 4Sd = 1.0 FLOP/byte      ← exactly 1, independent of S and d
```

**Attention decode has an arithmetic intensity of exactly 1 FLOP/byte in FP16**, and 2 FLOP/byte if the KV cache is stored in FP8. It cannot be improved by making S or d bigger, because both numerator and denominator scale identically. And critically — **batching does not help it either**, because every sequence in the batch has its *own* KV cache; there is nothing shared to amortise. Hold that; it is lesson 03.4's central tension.

**(g) The weight-streaming GEMM that decode actually is.** The bulk of decode FLOPs are not attention, they are the feed-forward and projection matmuls against the model weights. Batch B tokens against a weight matrix K×N:

```
FLOPs = 2 · B · N · K
Bytes = e · (B·K + K·N + B·N)         where e = bytes per element
      ≈ e · K·N                        for B ≪ K, N  (the weights dominate)

AI ≈ 2·B·N·K ÷ (e · K·N) = 2B/e   FLOP/byte
```

| Precision | e (B/elem) | AI | H100 ridge | **Critical batch B\*** |
|---|---|---|---|---|
| FP16/BF16 | 2 | B | 295 | **295** |
| FP8 | 1 | 2B | 591 | **295** |
| FP4 (B200) | 0.5 | 4B | 1,169 (B200) | **292** |

Two results worth boxing:

- **The weight-streaming GEMM's arithmetic intensity is essentially the batch size.** At batch 1 it is 1 FLOP/byte — matching the attention-decode result and Google Cloud's measured B200 figure. At batch 295 on an H100 it reaches the ridge.
- **The critical batch size is invariant under precision.** FP8 halves the bytes (doubling AI) *and* doubles the compute roof (doubling the ridge), so the crossover batch stays at ≈295 on Hopper and ≈292 on Blackwell. Quantisation does not let you reach the compute roof at a smaller batch; it moves both goalposts equally. What it *does* buy is 2× throughput everywhere below the crossover, because you are moving half the bytes — which is exactly the memory-bound regime that inference lives in.

### 6. The roofline, plotted, with these workloads on it

```
  H100 SXM5 ROOFLINE — dense, log-log, both axes decades
  ═══════════════════════════════════════════════════════════════════════════

  attainable
  FLOP/s
    │
 2 P┤                                        ┌────────────────────────────
    │                                       /│   FP8 tensor roof  1,979 T
 1 P┤                                      / │
    │                                     /  ▲ ridge 591
989 T┤                        ┌───────────/──┴──────────────────────────────
    │                        /                   BF16 tensor roof  989 T
    │                       /│                       ● GEMM N=8192
100 T┤                     / ▲ ridge 295             (AI 2731, meas. 780 T)
    │                     /  │                     ● prefill S=4096
 67 T┤            ┌──────/───┴──────────────────────  FP32 CUDA-core roof 67 T
    │            /    ▲ ridge 20
 10 T┤          /     │
    │         /
  1 T┤       /            ← every kernel in this wedge is MEMORY-BOUND:
    │      /                its ceiling is AI × 3.35 TB/s, nothing else
560 G┤  ●─/  SAXPY / elementwise (AI 0.167) — attainable 0.56 T = 0.06% of roof
    │  ●/   decode, batch 1 (AI 1.0)        — attainable 3.35 T = 0.34% of roof
    │  /
    └──┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴───▶ AI (FLOP/byte)
      0.1   1    3.3  10   32   100  295  591  1k   2.7k  10k

  slope of the slanted line = HBM bandwidth = 3.35 TB/s
  ▲ marks a ridge point: one per compute roof, because each roof has a
    different P over the same B.

  MOVING A POINT:                        MOVING A ROOF:
   → right = fuse, batch, block           ↑ up   = lower precision, newer SKU
     (fewer DRAM bytes per FLOP)          ↗ tilt = more bandwidth (H100→H200)
```

Note what the three compute roofs buy you: a kernel that only uses FP32 CUDA cores is capped at 67 TFLOP/s no matter how high its AI, so its ridge point is at AI = 20. **This is the original paper's "ceilings" idea** — below the topmost roof there are lower roofs corresponding to instruction mixes the kernel fails to exploit (in 2009: no SIMD, no FMA, no memory affinity; in 2026: no tensor cores, no FP8, no 2:4 sparsity). Their diagnostic value is ordering: if you are under the FP32 ceiling, using the tensor cores is worth more than any bandwidth work, and no amount of bandwidth will help you.

### 7. Naive AI is an upper bound on the denominator, i.e. a lower bound on trouble

Everything in §5 computed *compulsory* DRAM traffic — each input read once, each output written once. Real kernels move more:

- **Capacity misses.** If a tile does not fit in L2 (50 MB on H100), it gets re-read. A 8192² BF16 matrix is 134 MB; the tiling schedule decides how many times each element crosses the bus.
- **Read-modify-write amplification.** An in-place update reads and writes the same line.
- **Kernel boundaries.** Each un-fused kernel in a chain writes its output to HBM and the next reads it back. A five-op elementwise chain moves 10 tensor-sized round trips instead of 2.

So the practical rule is: **compute the compulsory-traffic AI as your optimistic estimate, then measure.** If measured AI is far below the compulsory estimate, you have found reuse that is being thrown away, and fusion or blocking will recover it. If measured AI matches the estimate and you are still slow, you are somewhere else on the roofline — or in the third regime.

**Measuring AI on real hardware.** Two routes, both usable inside a rented hour:

```bash
# Route 1 — Nsight Compute. Gives per-kernel FLOP and DRAM byte counters directly.
$ ncu --metrics \
  sm__sass_thread_inst_executed_op_fadd_pred_on.sum,\
sm__sass_thread_inst_executed_op_fmul_pred_on.sum,\
sm__sass_thread_inst_executed_op_ffma_pred_on.sum,\
dram__bytes_read.sum,dram__bytes_write.sum \
  ./my_kernel
# AI = (fadd + fmul + 2×ffma) / (dram_bytes_read + dram_bytes_write)
# ncu also has a built-in roofline chart: --set full, then open in the UI.

# Route 2 — DCGM, whole-GPU, no instrumentation, works on a live serving process.
$ dcgmi dmon -e 1004,1005 -d 1000       # TENSO (tensor-pipe), DRAMA (DRAM active)
# DRAM_ACTIVE × peak_bandwidth ≈ achieved bytes/s.
# Achieved FLOP/s from your own instrumentation (tokens/s × FLOPs/token, or
# matmuls/s × 2N³).  AI = achieved FLOP/s ÷ achieved bytes/s.
```

Route 2 is the one you will actually use in production, because it needs no kernel replay and no profiling stop-the-world. Its output is exactly the pair of numbers the roofline wants.

### 8. The third regime the roofline cannot draw: overhead-bound

The roofline has two roofs. Real workloads have a third way to be slow, and it is not a function of arithmetic intensity at all, so it cannot appear on the chart: **overhead-bound** — wall-clock spent on kernel-launch latency, Python dispatch, host synchronisation and gaps, rather than on either compute or memory.

Horace He's framing in "Making Deep Learning Go Brrrr From First Principles" is the clean version: every workload is compute-bound, memory-bound, or overhead-bound, and you must know which before spending a dollar. The tell for overhead is unambiguous once you have lesson 03.1's counters: **achieved FLOP/s is far below *both* roofs, and neither `DRAM_ACTIVE` nor `PIPE_TENSOR_ACTIVE` is anywhere near saturated.** Memory-bound looks different (DRAM_ACTIVE high); compute-bound looks different (TENSOR_ACTIVE high). Overhead-bound is the case where nothing is high and the GPU is nonetheless "100% utilised" by NVML's reckoning — lesson 03.1's launch-bound row of the diagnosis matrix.

Why it matters for placement: **rule out overhead before you conclude "compute-bound, buy a faster GPU."** If you are overhead-bound, the fix is `torch.compile` / CUDA graphs / operator fusion, and it is free. A bigger SKU would run the same overhead and show the same low utilisation at a higher hourly rate. He argues operator fusion is the single most impactful technique precisely because it is the one lever that helps the memory-bound *and* the overhead-bound case at once: fewer kernels means both fewer launches and fewer HBM round-trips.

### 9. The decision procedure

```
                  PLACING A WORKLOAD — the whole lesson as a flowchart
  ═════════════════════════════════════════════════════════════════════════

     measure: achieved FLOP/s, DRAM_ACTIVE, TENSOR_ACTIVE
                          │
                          ▼
        ┌─────────────────────────────────────────┐
        │ Is achieved FLOP/s far below BOTH roofs │
        │ AND DRAM_ACTIVE low AND TENSOR_ACTIVE   │──yes──▶ OVERHEAD-BOUND
        │ low?                                    │         fuse / torch.compile
        └──────────────────┬──────────────────────┘         / CUDA graphs.
                           │ no                             Hardware change: NONE.
                           ▼
        ┌─────────────────────────────────────────┐
        │ compute AI = FLOPs ÷ DRAM bytes         │
        │ compare with ridge = P ÷ B for the      │
        │ precision you actually run              │
        └──────────────────┬──────────────────────┘
                           │
         ┌─────────────────┴─────────────────┐
         │ AI ≪ ridge                        │ AI ≫ ridge
         ▼                                   ▼
   MEMORY-BOUND                        COMPUTE-BOUND
   ceiling = AI × B                    ceiling = P
         │                                   │
         ▼                                   ▼
   achieved / (AI×B) ?                 achieved / P ?
    ├─ >70% → at the roof.              ├─ >70% → at the roof.
    │   Levers: raise AI (fuse,         │   Levers: lower precision
    │   block, batch) or buy            │   (FP8/FP4 — but recheck the
    │   BANDWIDTH (H100→H200            │   ridge, it doubles!), or buy
    │   = ×1.43 for ×0 compute).        │   a higher-FLOP SKU.
    │   Buying FLOPs: ZERO effect.      │   Buying bandwidth: ZERO effect.
    └─ <70% → kernel/tiling problem     └─ <70% → occupancy, tiling, or
        before any hardware talk.           instruction mix (are you under
                                            a lower ceiling, e.g. FP32
                                            CUDA cores instead of tensor?)
```

Six rules in prose, for the whiteboard:

1. Estimate AI = FLOPs ÷ compulsory DRAM bytes.
2. Compare with the ridge point **for the precision you actually run** (§4 table).
3. **AI ≫ ridge → compute-bound.** Levers: lower precision, higher-FLOP SKU, better tiling. A faster GPU can pay off.
4. **AI ≪ ridge → memory-bound.** Levers: fusion (raises AI), batching (raises AI for weight-streaming GEMMs), higher-*bandwidth* SKU. A higher-FLOP-same-bandwidth GPU pays off **nothing**.
5. **Already at ~70–90% of the relevant ceiling?** Stop optimising that layer. The remaining win is a different SKU or a different algorithm.
6. **Rule out overhead first**, always.

### 10. From roofline to dollars

The roofline becomes a cost model the moment you divide by price:

```
$ per unit of useful work = (GPU $/hr) ÷ (achieved useful throughput)
```

For a memory-bound workload, achieved throughput is proportional to bandwidth, so the comparison collapses to a bandwidth-per-dollar contest. Worked for LLM decode on Hopper (illustrative 2026-era on-demand rates; lesson 03.7 covers pulling live prices):

| SKU | Dense BF16 | HBM BW | ≈$/hr | Relative decode throughput | Relative $/token |
|---|---|---|---|---|---|
| H100 SXM | 989 TFLOP/s | 3.35 TB/s | ~$3.00 | 1.00× | 1.00 |
| H200 SXM | 989 TFLOP/s | 4.8 TB/s | ~$3.70 | **1.43×** | **0.86** |

```
H200 throughput ratio = 4.8 ÷ 3.35             = 1.433
H200 price ratio      = 3.70 ÷ 3.00            = 1.233
H200 $/token ratio    = 1.233 ÷ 1.433          = 0.860   → 14% cheaper per token
```

Note that the H200 has **identical compute** — the entire 43% throughput gain comes from the roof the workload is actually under. Now run the same comparison for a compute-bound training GEMM: throughput ratio 989/989 = 1.00, price ratio 1.233, **$/unit-work ratio 1.23 — the H200 is 23% *more* expensive for that workload.** Same two SKUs, opposite ranking, decided entirely by which roof the workload sits on.

**That inversion is the whole reason this lesson exists.**

## Perspectives

**Theory.** The formal model is `Attainable = min(peak_FLOP/s, AI × peak_BW)` with ridge point `peak ÷ BW`, from Williams, Waterman & Patterson (CACM 52(4), 2009). Its value is that it is *bound-and-bottleneck* analysis: it does not predict performance, it predicts the *ceiling*, which is the only prediction robust enough to survive contact with a real kernel. The paper's own additions beyond the two roofs — ceilings for unexploited instruction-level features, and the observation that ridge points march right as core counts outpace memory — are both directly live in 2026 GPU work.

**Practice.** Horace He's compute/memory/overhead trichotomy is the practitioner's version, plus the crucial third regime the chart cannot depict because it is not a function of AI. He is a PyTorch core engineer diagnosing real model slowdowns with this framework, which is why the framing survives contact with eager-mode Python in a way the pure HPC version does not.

**Hardware.** The ridge point is a hardware constant that moves with precision, and hardware vendors move it deliberately. H100 → H200 lowers it (206 vs 295) by adding bandwidth to an unchanged die; H100 → B200 holds it roughly constant (292 vs 295) by scaling both. Reading a new SKU announcement as "which direction did the ridge move, and does my workload live on the side that improved" is a five-second analysis that most people never do.

**Economics.** The roofline is dollars the instant you divide by $/hr, and the H100-vs-H200 inversion above shows the ranking is workload-dependent, not absolute. Google Cloud's B200 decode benchmark (4.4% FLOPS-util, 292× below ridge) becomes a cost-per-token argument the moment you attach a rental rate — which is precisely the move lesson 03.7's capstone asks you to make for a SKU recommendation you have to defend.

## Real-world use cases

- **Google Cloud, ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0).** A hyperscaler computing a B200 ridge point on live production LLM-serving traffic and showing decode sitting 292× below it — with 4.4% FLOPS utilisation, 10.9% bandwidth utilisation and 1.5% tensor-active while serving ~1M tok/s across 96 GPUs. What it shows: a formal roofline placement done operationally, and the correct conclusion (this is the ceiling, not a bug). The 292 is re-derivable: `2,250 TFLOP/s ÷ 7.7 TB/s = 292 FLOP/byte`, against decode's AI of 1.
- **Dao et al., FlashAttention (arXiv:2205.14135), as a roofline movement.** The paper's stated premise is that attention's bottleneck is HBM traffic, not FLOPs. In this lesson's terms the contribution is moving unfused attention's AI from ≈`d/4` (32 FLOP/byte at d=128 — memory-bound) to ≈`S/2` (2,048 at S=4096 — compute-bound), by tiling and an online softmax so the S×S score matrix never reaches HBM. What it shows: the single most consequential ML-systems result of the decade is, mechanically, a denominator reduction on the roofline's x-axis.
- **Horace He, ["Making Deep Learning Go Brrrr From First Principles"](https://horace.io/brrr_intro.html).** A PyTorch-core engineer's account of diagnosing real model slowdowns via compute/memory/overhead classification, with operator fusion as the highest-leverage technique. What it shows: the model applied by a practitioner to production framework performance work, and the overhead regime that HPC roofline literature under-weights because HPC codes rarely launch thousands of microsecond kernels from Python.
- **NERSC, ["Roofline Performance Model" docs](https://docs.nersc.gov/tools/performance/roofline/).** Lawrence Berkeley/NERSC — the lab that produced the original paper — maintaining roofline analysis as operational guidance for real scientific kernels on real supercomputers, nearly two decades later, including GPU-specific hierarchical rooflines (separate roofs for L1, L2 and HBM). What it shows: the model has held up well enough to remain a facility's standard performance-analysis method.

## Worked example

Rent one H100 SXM. Run two kernels, measure both, place both, and read off the lever.

**(1) Compute-bound: `torch.matmul`, N = 8192, BF16.**

```python
import torch, time
N, iters = 8192, 200
a = torch.randn(N, N, device="cuda", dtype=torch.bfloat16)
b = torch.randn(N, N, device="cuda", dtype=torch.bfloat16)
torch.matmul(a, b)                      # warm up: cuBLAS heuristic + autotune
torch.cuda.synchronize(); t0 = time.perf_counter()
for _ in range(iters):
    c = torch.matmul(a, b)
torch.cuda.synchronize(); dt = time.perf_counter() - t0
print(f"{iters * 2 * N**3 / dt / 1e12:.1f} TFLOP/s")
```

```
Theoretical FLOPs per matmul = 2 · 8192³            = 1.0995e12 FLOP
Compulsory bytes             = 2 · 3 · 8192² · 1    = 4.03e8 B    (2 B/elem, 3 matrices)
AI                           = 1.0995e12 ÷ 4.03e8   = 2,728 FLOP/byte
                             (matches N/3 = 2,731; rounding)

Measured: 200 matmuls in 0.282 s
Achieved = 200 × 1.0995e12 ÷ 0.282 = 7.80e14 FLOP/s = 780 TFLOP/s

Placement: AI 2,728 ≫ ridge 295          → COMPUTE-BOUND
Fraction of roof: 780 ÷ 989 = 78.9 %
Cross-check: PIPE_TENSOR_ACTIVE should be high (~0.7–0.8); DRAM_ACTIVE modest.
```

Reading: near the ceiling. The only levers left are precision (FP8 → a 1,979 TFLOP/s roof, but the ridge moves to 591 and AI 2,728 is still far above it, so the flip is safe here) or a newer SKU. Kernel work is not worth doing at 79% of peak.

**(2) Memory-bound: SAXPY over a 1 GiB FP32 buffer.**

```python
import torch, time
N = 268_435_456                                  # 2^28 floats = 1 GiB
x = torch.randn(N, device="cuda"); y = torch.randn(N, device="cuda")
a = 2.0
y.add_(x, alpha=a); torch.cuda.synchronize()     # warm up
t0 = time.perf_counter()
for _ in range(50):
    y.add_(x, alpha=a)                           # read x, read y, write y
torch.cuda.synchronize(); dt = time.perf_counter() - t0
print(f"{50 * 3 * N * 4 / dt / 1e12:.2f} TB/s")
```

```
Bytes per pass = 3 × 2.684e8 × 4 B = 3.22e9 B     (read x, read y, write y)
FLOPs per pass = 2 × 2.684e8       = 5.37e8 FLOP
AI             = 5.37e8 ÷ 3.22e9   = 0.167 FLOP/byte

Measured: 50 passes in 0.0537 s → 1.074 ms/pass
Achieved bandwidth = 3.22e9 ÷ 1.074e-3 = 3.00e12 B/s = 3.00 TB/s  (89.5 % of 3.35)
Achieved FLOP/s    = 5.37e8 ÷ 1.074e-3 = 5.00e11    = 0.50 TFLOP/s (0.051 % of 989)

Placement: AI 0.167 ≪ ridge 295          → MEMORY-BOUND
Fraction of the roof that applies: 3.00 ÷ 3.35 = 89.5 % of the MEMORY roof
Cross-check: DRAM_ACTIVE should be ~0.85–0.95; PIPE_TENSOR_ACTIVE ~0.
```

Reading: **saturated at 0.05% of compute peak, and that is a success, not a failure.** 89.5% of achievable bandwidth is about as good as a real kernel gets (the missing ~10% is refresh, ECC, row-activation overhead and imperfect access alignment). A bigger-compute GPU is worthless here; only bandwidth or a higher-AI algorithm helps.

**(3) The three-point verdict, and what each one licenses you to buy.**

| Point | AI | Roof it sits under | Achieved | % of its own roof | The one lever |
|---|---|---|---|---|---|
| GEMM N=8192 BF16 | 2,728 | compute (989 T) | 780 TFLOP/s | 79% | FP8, or a higher-FLOP SKU |
| SAXPY 1 GiB FP32 | 0.167 | memory (3.35 TB/s) | 3.00 TB/s | 89% | bandwidth SKU, or raise AI by fusing |
| Decode batch 1 (B200, cited) | 1.0 | memory | — | ~11% of BW roof | batching (raises AI toward 295) |

The third row is the one that maps to money at inference scale, and it is the only one of the three whose fix is neither a kernel change nor a hardware change: it is **more concurrent work**, which is lesson 03.4.

**(4) A cost conclusion from the same two measurements.** Suppose the workload is 70% SAXPY-like elementwise and norm work and 30% GEMM by wall clock. On H200 the GEMM half is unchanged (same compute die) and the elementwise half runs 1.43× faster:

```
H100 wall clock (normalised) = 0.70 + 0.30              = 1.000
H200 wall clock              = 0.70/1.43 + 0.30         = 0.790
Speedup                      = 1 / 0.790                = 1.27×
Price ratio (illustrative)   = 3.70 / 3.00              = 1.233
$/work ratio                 = 1.233 / 1.27             = 0.97  → ~3% cheaper
```

A 3% margin is inside the noise of spot pricing — so the honest recommendation is "not worth a migration on these numbers; revisit if the elementwise fraction rises or the price gap narrows." **The roofline's job is not always to justify a purchase. Sometimes it is to stop one.**

## Practice

**Task (rent one H100 or A100 by the hour; budget ~1–1.5 hr ≈ $3–6 — reuse the session from lesson 03.1 if possible).**

1. **Record your machine's roofs.** From the spec table (or `nvidia-smi -q` plus the datasheet for your exact SKU), write down peak dense BF16 TFLOP/s, peak FP8 TFLOP/s if supported, peak FP32 CUDA-core TFLOP/s, and HBM bandwidth. Compute all three ridge points by hand.
2. **Compute-bound point.** Run the large square BF16 `torch.matmul` above at N ≈ 8192 in a timed loop. Compute achieved TFLOP/s as `2·N³ × iters ÷ seconds`, and record it as a fraction of the dense BF16 roof. Compute AI by hand as `N/3`.
3. **Memory-bound point.** Run SAXPY/elementwise over a ≥1 GiB buffer. Compute achieved GB/s as `bytes_moved ÷ seconds` (count reads *and* writes), record it as a fraction of the HBM roof, and compute its achieved TFLOP/s and AI.
4. **Sweep the crossover.** Run the matmul at N = 256, 512, 1024, 2048, 4096, 8192. Plot achieved TFLOP/s against N. You should see performance climb and then flatten as AI (`N/3`) crosses the ridge (295) somewhere near N ≈ 1024, and then saturate. **Mark on your plot the N at which measured performance first reaches 50% of the compute roof, and compare it with where the theory says the ridge is.** Explain any gap.
5. **Draw the roofline** (hand-drawn is fine): x = AI on a log scale, y = achieved FLOP/s on a log scale. Plot the memory roof as a slanted line of slope 1 anchored at your measured bandwidth, the compute roof(s) as horizontal lines, and place every measured point.
6. **Cross-check placement against the DCGM fields from lesson 03.1.** The GEMM point should show high `PIPE_TENSOR_ACTIVE` and moderate `DRAM_ACTIVE`; the SAXPY point the reverse. If neither is high, you are overhead-bound and the placement is not meaningful — investigate before trusting it.
7. **(Stretch) Demonstrate fusion moving a point.** Time `y = ((x * 2) + 3).relu()` as three separate eager ops, then again wrapped in `torch.compile`. Compute AI for both. The fused version should move measurably right on the x-axis and up the memory roof, because it makes one HBM round trip instead of three.

**Acceptance:** a roofline plot carrying **both required measured points** (plus the N-sweep if you did step 4), with:

- AI and achieved FLOP/s labelled per point, computed by hand and shown;
- the ridge point(s) marked with the arithmetic that produced them;
- the DCGM cross-check values next to each point;
- a one-line verdict per point in the form *"<bound> at <x>% of the <which> roof → only <lever> helps."*

This figure is the centrepiece of the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md).

## Common pitfalls

1. **Treating low utilisation as inherently a problem to fix**, without first checking which roof applies. Google Cloud's B200 case rebuts it directly: 4.4% FLOPS-util was the *correct* outcome for that arithmetic intensity. *Mechanism:* attainable FLOP/s under the memory roof is `AI × B`, and at AI = 1 that is 0.34% of the compute roof by construction.
2. **Forgetting the ridge point moves with precision.** FP8 raises H100's ridge from 295 to 591, so a kernel that is compute-bound in BF16 can be memory-bound in FP8. Evaluating an FP8 kernel against the BF16 ridge silently misclassifies it — and then you "explain" the missing 2× speedup as a software problem when it is physics.
3. **Conflating overhead-bound with memory-bound.** Both show low achieved FLOP/s. The fixes are opposite (launch elimination vs bandwidth/AI). Check `DRAM_ACTIVE`: memory-bound saturates it, overhead-bound does not.
4. **Using the sparse compute peak to compute the ridge.** Using 1,979 instead of 989 on H100 doubles the ridge to 591 and makes every workload look more memory-bound than it is — the mirror image of pitfall 2, and it will make you buy bandwidth you did not need.
5. **Assuming roofline placement is a property of the algorithm alone.** It is a property of *(algorithm × implementation × precision × batch size)*. Unfused attention has AI ≈ 32; fused, ≈ 2,048. Decode at batch 1 has AI 1; at batch 295 on H100 it is at the ridge. "Is this compute-bound?" is not answerable without those qualifiers.
6. **Computing bytes as total memory traffic rather than DRAM traffic.** The model's x-axis is defined against off-chip DRAM specifically. Counting L2 hits inflates the denominator and drags every kernel leftward into a memory-bound classification it does not deserve.
7. **Comparing achieved FLOP/s against the compute roof when you are under the memory roof.** "We are at 15% of peak" is meaningless if the applicable ceiling is `AI × B`. Always divide by *the roof that applies*, and say which one.
8. **Ignoring the lower compute ceilings.** A kernel running FP32 on CUDA cores is capped at 67 TFLOP/s on H100 — 6.8% of the BF16 tensor roof — so it can look catastrophically far from "peak" while being at 95% of the peak *it can actually reach*. Establish which functional unit the kernel uses before dividing.

## Self-check

- **Arithmetic intensity of an FP16 GEMM vs an elementwise add — roughly, and why?** Square GEMM: `FLOPs = 2N³`, compulsory `Bytes = 2·(3N²)`, so `AI = N/3` — 2,731 FLOP/byte at N = 8192. Elementwise add: `FLOPs = N`, `Bytes = 6N`, so `AI = 1/6 ≈ 0.167`. About four orders of magnitude apart, because GEMM reuses each loaded element across the whole inner dimension (each element of A participates in N multiply-accumulates) while the add touches each byte once. Note the GEMM figure assumes blocked execution; a naive unblocked GEMM has AI ≈ 0.5 and is memory-bound.
- **You measure 15% of peak TFLOPS but HBM bandwidth is saturated. Will a faster-compute GPU help?** No. Saturated bandwidth with low FLOP/s means you are on the *memory* roof, where the ceiling is `AI × B`, not P. The 15% figure is measured against a roof you never touch, so raising that roof does nothing. Buy bandwidth (H100 → H200 is ×1.43 bandwidth for ×1.00 compute) or raise AI via fusion/batching. Paying for FLOPs is pure waste.
- **Ridge point for an H100, at BF16 and at FP8, with the arithmetic.** `989e12 FLOP/s ÷ 3.35e12 B/s = 295 FLOP/byte` dense BF16. FP8: `1,979e12 ÷ 3.35e12 = 591 FLOP/byte`. Lower precision doubles compute without touching bandwidth, so it *raises* the intensity a kernel needs to remain compute-bound. For contrast, H200 at BF16 is `989e12 ÷ 4.8e12 = 206` — the ridge moves *left*, meaning more workloads can reach the compute roof.
- **Derive the arithmetic intensity of attention decode and explain why batching does not change it.** For one new token against a KV cache of length S with head dim d: `FLOPs = 2Sd (q·Kᵀ) + 2Sd (p·V) = 4Sd`; `Bytes = 2·S·d elements × 2 B = 4Sd` (the K and V caches must both be read; q and the output are O(d)). `AI = 4Sd ÷ 4Sd = 1.0 FLOP/byte`, independent of S and d. Batching does not help because every sequence carries its *own* KV cache — the bytes scale with the batch exactly as the FLOPs do. Batching *does* help the weight-streaming GEMMs (AI ≈ 2B/e, i.e. ≈ the batch size at FP16), because there the weights are shared across the batch. Those are different terms of the same forward pass, and confusing them is the classic error.
- **Why did FlashAttention matter, in roofline terms?** Unfused attention writes the S×S score matrix to HBM and reads it back, so bytes are dominated by `≈16S²` and `AI ≈ d/4` ≈ 32 FLOP/byte at d = 128 — far below H100's 295 ridge, i.e. memory-bound. Fusing the whole attention into one kernel with tiling and an online softmax means the score matrix never leaves on-chip memory, so bytes drop to `≈8Sd` and `AI = S/2` ≈ 2,048 at S = 4,096 — firmly compute-bound. Identical FLOPs, 64× the arithmetic intensity, and a change of which roof applies.
- **What is the critical batch size for compute-bound decode on H100, and why is it the same at FP8?** The weight-streaming GEMM has `AI ≈ 2B/e` where e is bytes per element. At BF16 (e = 2), AI = B, and the ridge is 295, so B\* = 295. At FP8 (e = 1), AI = 2B, but the ridge also doubles to 591, so B\* = 591/2 = 295 again. Precision halves the bytes and doubles the compute roof by the same factor, leaving the crossover batch invariant. What FP8 buys is 2× throughput throughout the memory-bound region below B\*, which is where inference actually lives.
- **A GKE benchmark reports a B200 at 4.4% FLOPS utilisation while serving 1M tokens/sec. Bug?** No. Decode's AI ≈ 1 FLOP/byte sits at `1 ÷ 292` of B200's BF16 ridge (`2,250 TFLOP/s ÷ 7.7 TB/s = 292 FLOP/byte`), so the attainable ceiling is `1 × 7.7e12 = 7.7 TFLOP/s`, which is 0.34% of the compute roof — and they measured 4.4%, i.e. *above* the batch-1 figure, consistent with some batching already in play. The aggregate throughput comes from concurrency against a fixed bandwidth budget. The lever is architectural (batching, disaggregation — lesson 03.4), not a higher utilisation target.
- **When is the roofline the wrong tool?** When you are overhead-bound. Launch latency and host stalls are not a function of arithmetic intensity, so they do not appear anywhere on the chart; a kernel can be far below both roofs with neither `DRAM_ACTIVE` nor `PIPE_TENSOR_ACTIVE` elevated. Diagnose that first; the fix is fusion/CUDA graphs and costs nothing.

## Connections & what's next

This lesson is the module's anchor. The DCGM fields from 03.1 (`DRAM_ACTIVE`, `PIPE_TENSOR_ACTIVE`) are how you verify a roofline placement on real hardware rather than trusting an estimate. The ridge point's dependence on precision is exactly the mechanism lesson 03.5 exploits as a cost lever, and the reason FP8's realised speedup is workload-dependent. The critical-batch derivation in §5(g) is lesson 03.4's entire argument in embryo. The `$/unit-work` formula in §10 is the direct ancestor of lesson 03.7's capstone cost model. Every subsequent lesson in this module is, in some sense, "apply the roofline to a new specific case."

Next: **[03.3 · Memory hierarchy & HBM bottleneck](03-memory-hierarchy-hbm.md)** takes the memory *roof* — a single bandwidth number in this lesson — and opens it up: the full hierarchy from registers to host, why HBM has the bandwidth and capacity it does, what competes for that bandwidth (weights vs KV cache vs activations), and what decides whether a workload even *fits*.

## References & further reading

**Primary sources**

1. Williams, Waterman & Patterson, ["Roofline: An Insightful Visual Performance Model for Multicore Architectures"](https://dl.acm.org/doi/10.1145/1498765.1498785), *Communications of the ACM* 52(4), April 2009, pp. 65–76 — the original paper. Read for the formal definitions used here: operational intensity as FLOPs per byte of traffic *between processor and off-chip DRAM*; attainable performance as `min(peak FP, AI × peak BW)`; the ridge point as "the minimum operational intensity required to achieve maximum performance"; the ceilings idea; and the Opteron X2 (17.6 GFLOP/s, ~15 GB/s, ridge ≈1.0) vs X4 (ridge 4.4) worked comparison quoted in §3. Free mirror: [escholarship.org PDF](https://escholarship.org/content/qt78h8v7mr/qt78h8v7mr.pdf) (LBNL).
2. NVIDIA, [H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — source of truth for the H100 roofs used throughout (989 TFLOP/s dense BF16, 1,979 dense FP8, 495 dense TF32, 67 TFLOP/s FP32 CUDA-core, 3.35 TB/s HBM3, 50 MB L2). Read the fine print to separate dense from with-sparsity figures before computing any ridge point.
3. NVIDIA, [A100](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-nvidia-us-2188504-web.pdf), [H200](https://www.nvidia.com/en-us/data-center/h200/) and Blackwell/B200 datasheets — the per-SKU peaks and bandwidths behind the machine-balance table in §4. Note the shipping HGX/DGX B200 configuration is 180 GB at 7.7 TB/s per GPU even where marketing materials quote "up to" 192 GB / 8 TB/s.
4. Dao, Fu, Ermon, Rudra & Ré, ["FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness"](https://arxiv.org/abs/2205.14135) (NeurIPS 2022) — the IO-aware attention algorithm behind §5(d)–(e). Read specifically for the argument that attention's bottleneck is HBM traffic rather than FLOPs, and for the tiling/online-softmax mechanism that keeps the score matrix off HBM.
5. NVIDIA, [Nsight Compute documentation](https://docs.nvidia.com/nsight-compute/) — the `dram__bytes_*` and `sm__sass_thread_inst_executed_op_*` metric families used in §7 to measure AI directly, and the built-in roofline chart (`--set full`).

**Real-world engineering blogs**

6. Google Cloud, ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — a formal roofline calculation on live B200 production traffic; this lesson's headline real-world case and the source of the 292× figure re-derived in §4.
7. Horace He, ["Making Deep Learning Go Brrrr From First Principles"](https://horace.io/brrr_intro.html) — the module's conceptual anchor; the compute/memory/overhead trichotomy of §8 and the argument that operator fusion is the highest-leverage single technique. Read twice.
8. Google DeepMind, ["How to Scale Your Model" — "All About Rooflines"](https://jax-ml.github.io/scaling-book/roofline/) (jax-ml/scaling-book, 2025) — a modern, free, frontier-lab treatment of rooflines written for the ML-systems audience, including how GPU rooflines differ from TPU rooflines. The best complement to the 2009 paper if you want the same model in transformer vocabulary.
9. NERSC, ["Roofline Performance Model" docs](https://docs.nersc.gov/tools/performance/roofline/) — the origin lab applying the model operationally to HPC and GPU kernels, including hierarchical (L1/L2/HBM) rooflines that extend §7's "naive AI is an upper bound" point.

**Deeper dives**

10. Luo et al., ["Benchmarking and Dissecting the Nvidia Hopper GPU Architecture"](https://arxiv.org/abs/2402.13499) — independent microbenchmarks of the memory hierarchy and tensor-core pipelines; useful when you need a measured number rather than a datasheet one.
11. Kaplan et al., ["Scaling Laws for Neural Language Models"](https://arxiv.org/abs/2001.08361) — the `C ≈ 6ND` FLOP convention used to convert token rates into the FLOP/s numerator of a roofline placement.
12. Kirk & Hwu, *Programming Massively Parallel Processors*, ch. 5–6 — tiling and memory-coalescing as the mechanism behind the `N/3` versus `0.5` GEMM contrast in §5(a). Read for the reuse argument, skip the coding exercises.
13. NVIDIA, [CUDA C++ Best Practices Guide](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html) — the memory-optimisation chapter on coalescing and the arithmetic-instruction throughput tables; the practical source for why a kernel can move fewer *useful* bytes than it moves *actual* bytes.
