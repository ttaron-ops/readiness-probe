---
lesson: "03.2"
title: "Compute-bound vs memory-bound and the roofline"
module: "03"
concept: "Compute-bound vs memory-bound and the roofline"
status: not-started
est_time: "6h"
artifacts: []
---

# 03.2 · Compute-bound vs memory-bound and the roofline

> **Concept.** Compute arithmetic intensity, place a workload under the compute roof or the memory roof, find the ridge point — the single lens for the whole cost story.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Why this matters

This is the module spine. Every GPU cost decision reduces to one question: **which roof am I under, and am I near it?** If a workload is *memory-bound*, a faster-compute GPU buys you nothing — you would pay more per hour for zero speedup. If it is *compute-bound* and far from the roof, the fix is software (batching, fusion, precision), not a bigger SKU. Being able to place a workload on a roofline and name the *one* thing that can possibly help is the difference between a platform engineer who guesses at capacity and one who can justify a SKU choice or an optimisation to a finance review with a number.

## The platform-engineer's lens

**Extract this:** the ability to (1) estimate a workload's **arithmetic intensity** (FLOPs per byte of HBM traffic), (2) compare it to the GPU's **ridge point** (peak FLOP/s ÷ HBM bandwidth), (3) declare it compute- or memory-bound, and (4) from that, say which lever — bigger SKU, different precision, fusion/batching, or nothing — can actually move throughput and therefore cost per unit of work. This one framework decides "will a hardware upgrade pay for itself?" without running the upgrade.

**Do NOT master:** hand-deriving rooflines per kernel, cache-level hierarchical rooflines, or squeezing the last 5% via kernel scheduling. You need the *first-order* placement (which roof, roughly how far), not a research-grade model. Leave the sub-roof cache analysis to kernel developers.

## Core notes

### The two ceilings

Any GPU has two independent hard limits on how fast a kernel can run:

- **Compute roof** — peak arithmetic throughput. H100 SXM: **989 TFLOP/s dense BF16** (1,979 FP8 dense). Measured in FLOP/s.
- **Memory roof** — peak HBM bandwidth. H100 SXM: **3.35 TB/s** from 80 GB HBM3. Measured in bytes/s.

A kernel is bounded by whichever ceiling it hits first. Which one it hits is decided entirely by its **arithmetic intensity**.

### Arithmetic intensity (AI)

```
Arithmetic Intensity = FLOPs performed / bytes moved to-and-from HBM   [FLOP/byte]
```

AI is a property of the *algorithm*, not the hardware. It says: for every byte I drag across the memory bus, how many floating-point operations do I get out of it? High AI → the bus can keep the ALUs fed → compute-bound. Low AI → the ALUs starve waiting for data → memory-bound.

**Attainable performance (the roofline model):**

```
Attainable FLOP/s = min( peak_FLOP/s ,  AI × peak_bandwidth )
```

Plot this with AI on the x-axis (log) and attainable FLOP/s on the y-axis (log): a **slanted line** (the memory roof, slope = bandwidth) rising until it hits a **flat line** (the compute roof). Where they meet is the **ridge point**.

### The ridge point — machine balance

```
Ridge-point AI = peak_FLOP/s / peak_bandwidth   [FLOP/byte]
```

This is the GPU's *machine balance*: the arithmetic intensity a kernel needs just to stop being memory-bound. For an **H100 SXM in dense BF16**:

```
Ridge = 989e12 FLOP/s ÷ 3.35e12 B/s ≈ 295 FLOP/byte
```

Interpretation you can quote: **on an H100 you need ~295 FLOPs of work per byte of HBM traffic just to reach the compute roof.** Below that AI you are memory-bound no matter what; above it you *can* be compute-bound (if the kernel is efficient). Note the ridge *moves with precision*: FP8 dense (1,979 TFLOP/s) pushes it to ≈ **590 FLOP/byte** — lower precision doubles compute but not bandwidth, so it *raises* the bar to stay compute-bound. That is why memory-bound kernels get no benefit from FP8.

### Where real workloads land

**Large dense matmul (GEMM), C = A·B, square N, FP16:**
- FLOPs = `2·N³`
- Bytes (read A, B; write C, 2 B/elem) = `2·(3N²)` = `6N²`
- AI = `2N³ / 6N²` = **N/3**

For N = 8192: AI ≈ **2,730 FLOP/byte** — an order of magnitude above the 295 ridge → **firmly compute-bound.** This is why matmul-heavy training gets close to peak and *does* benefit from a faster-compute GPU. AI grows with N, so bigger matmuls are more compute-bound — the arithmetic reason batching helps.

**Elementwise add, C = A + B, FP16:**
- FLOPs = `N` (one add per element)
- Bytes = `3N` elements × 2 B = `6N`
- AI = `N / 6N` = **0.167 FLOP/byte**

**SAXPY, y = a·x + y, FP32:**
- FLOPs = `2N`; Bytes = `3N × 4` = `12N`; AI = **0.167 FLOP/byte**

Both sit ~1,800× *below* the ridge → **hopelessly memory-bound.** They run at `AI × bandwidth` ≈ `0.167 × 3.35e12` ≈ **0.56 TFLOP/s** — about **0.06% of the 989 TFLOP/s compute roof** — and there is nothing wrong with the kernel; that is the *ceiling* for that arithmetic intensity. A faster-compute GPU raises a roof this kernel never touches.

**The general map:**

| Workload | Typical AI | Roof | What helps |
|---|---|---|---|
| Large GEMM / dense training | 10²–10³ | Compute | Faster-compute SKU, lower precision, tensor cores |
| Attention (fused, e.g. FlashAttention) | 10¹–10² | Near ridge | Fusion, bigger heads/batch |
| Elementwise, activations, norms, add | <1 | Memory | Fusion (fewer round-trips), faster-BW SKU, bigger dtype cuts AI |
| LLM inference decode (batch 1) | <1 (bandwidth-bound) | Memory | Batching, KV-cache tricks, faster-BW SKU (H200) |

The last row is the one that surprises people and wins interviews: **autoregressive decode at small batch is memory-bound** — you stream the whole weight matrix from HBM to generate each token, doing very little arithmetic per byte. That is why H200 (141 GB HBM3e @ **4.8 TB/s**) beats H100 on decode throughput despite near-identical compute, and why "buy more FLOPs" is the wrong answer for inference serving.

### The third regime: overhead-bound

The roofline draws two roofs, but real workloads have a *third* way to be slow that the plot does not show: **overhead-bound** — the GPU spends its time on neither compute nor memory but on kernel-launch latency, Python dispatch, and CPU/host stalls (the flip side of lesson 03.1's launch-bound storms). A kernel that is *theoretically* compute-bound can measure at 10% of the roof simply because 90% of wall-clock is launch overhead and gaps. The tell: achieved FLOP/s is far below *both* roofs, and neither DRAM_ACTIVE nor TENSOR_ACTIVE is saturated.

Why it matters for placement: **before you conclude "compute-bound, buy a faster GPU," rule out overhead.** If you are overhead-bound, the fix is `torch.compile`/CUDA-graphs/fusion (eliminate launches) and it is *free* — a bigger SKU would run the *same* overhead and show the *same* low utilisation at higher cost. Horace He's framing is the clean version: every workload is compute-bound, memory-bound, or overhead-bound, and you must know which before spending a dollar.

### From roofline to dollars: cost per unit of work

The roofline becomes a cost model the moment you divide by price. Cost per unit of useful work =

```
$/unit-work = (GPU $/hr) / (achieved useful throughput)
```

Worked comparison — serving LLM decode (memory-bound, per the table above) on H100 vs H200, using bandwidth as the throughput proxy since decode scales with HBM bandwidth:

| SKU | HBM BW | On-demand ≈ $/hr | Relative decode throughput | Relative $/token |
|---|---|---|---|---|
| H100 SXM | 3.35 TB/s | ~$3.0 | 1.00× | 1.00 |
| H200 SXM | 4.8 TB/s | ~$3.7 | ~1.43× | **~0.86** |

Even at a ~23% higher hourly rate, the H200 is **~14% cheaper per token** for this memory-bound workload — *because* the roofline says decode rides the memory roof, and the H200 raises exactly that roof. Run the identical comparison for a compute-bound training GEMM and the ranking can invert (you would weigh FP8 TFLOP/s, not bandwidth). **The roof you are under decides which SKU minimises cost per unit of work** — that sentence is the whole reason this lesson exists.

### The decision procedure (the whole point)

1. Estimate AI = FLOPs ÷ HBM bytes for the workload.
2. Compare to ridge (≈295 FLOP/byte dense BF16 on H100).
3. **AI ≫ ridge → compute-bound.** Levers: lower precision (FP8/tensor cores), a higher-FLOP SKU, better GEMM tiling. A faster GPU *can* pay off.
4. **AI ≪ ridge → memory-bound.** Levers: **fusion** (do more per byte → raises AI), **batching**, a higher-*bandwidth* SKU (H200). A higher-FLOP-but-same-BW GPU pays off *nothing* — do not buy it.
5. **Near the roof already?** If achieved FLOP/s (or achieved bandwidth) is ~70–90% of the relevant ceiling, stop optimising that layer — you are done; the remaining win is a different SKU or a different algorithm.

## Worked example

Rent one H100 SXM. Run two kernels and interpret them on the roofline.

**(1) Compute-bound: `torch.matmul`, N = 8192, BF16.**
- Theoretical FLOPs = `2 · 8192³` ≈ 1.10e12 FLOP per matmul.
- Suppose you measure 1,400 matmuls/s → `1.54e15` FLOP/s = **~1.5 PFLOP/s? no** — sanity check: achieved ≈ 1.10e12 × 1400 = 1.54e15... that exceeds dense peak, so in practice you will see ~1,200/s → **≈ 780 TFLOP/s ≈ 79% of the 989 dense roof.** AI ≈ 2,730 → plot lands on the **flat (compute) roof**, near it. Reading: near the ceiling; the only lever left is FP8 (→1,979 roof) or a newer SKU.

**(2) Memory-bound: SAXPY on a 1 GB FP32 buffer.**
- Bytes ≈ 3 GB traffic; measure ~1.0 ms → `3e9 / 1e-3` = **3.0 TB/s ≈ 90% of the 3.35 TB/s HBM roof.** FLOP/s ≈ `2N / t` ≈ 0.5 TFLOP/s. AI ≈ 0.167 → plot lands on the **slanted (memory) roof**, near it. Reading: bandwidth-saturated at 0.05% of compute peak — a bigger-compute GPU is worthless; only more bandwidth (H200) or an algorithm with higher AI helps.

**Roofline sketch (both points):**

```
FLOP/s (log)
 989T ┤                         ┌──────────────  compute roof
      │                        /
      │                       /  ● GEMM (AI≈2730, ~780 TFLOP/s) — near compute roof
      │                      /
      │            memory   /
      │            roof   /
      │  ● SAXPY        /  ← ridge point (AI≈295)
      │  (AI≈0.17)    /
      └──────────────┴─────────────────────────  Arithmetic Intensity (log)
        0.17        295        2730
```

Two points, one ridge, opposite roofs — the entire cost thesis in one figure: *GEMM is near the compute roof (upgrade compute to go faster); SAXPY is near the memory roof (upgrade bandwidth or fuse, never compute).*

## Practice

**Task (rent one H100 or A100 by the hour; budget ~1–1.5 hr ≈ $3–6).**

1. **Compute-bound point:** run a large square `torch.matmul` (N ≈ 8192) in BF16 in a timed loop; compute achieved TFLOP/s from `2·N³ × iters ÷ seconds`; record it as a fraction of the 989 dense-BF16 roof.
2. **Memory-bound point:** run a SAXPY/elementwise op over a large buffer (≥1 GB); compute achieved GB/s = `bytes_moved ÷ seconds`; record it as a fraction of the 3.35 TB/s roof, and its achieved TFLOP/s.
3. For each, compute AI by hand and place it on a **hand-drawn roofline** (x = AI log-scale, y = achieved FLOP/s log-scale). Mark the **ridge point** at AI ≈ 295.

**Acceptance:** a roofline plot (hand-drawn is fine) carrying **both measured points** with their AI and achieved FLOP/s labelled, the ridge point marked, and a one-line verdict per point ("compute-bound, ~79% of roof → only precision/SKU helps" / "memory-bound, ~90% of BW → only bandwidth/fusion helps"). This figure is the centrepiece of the GPU Efficiency & Cost Report.

## Self-check

**(a) Arithmetic intensity of an FP16 matmul vs an elementwise add — roughly?**
**Answer:** Matmul (square N) has AI ≈ **N/3** — hundreds to thousands of FLOP/byte (e.g. ~2,730 at N=8192), so compute-bound. Elementwise add has AI ≈ **1/6 ≈ 0.17** FLOP/byte (one FLOP per three element accesses × 2 B), so memory-bound. Difference of roughly **four orders of magnitude** — matmul reuses each loaded byte across the whole inner dimension; add touches each byte once.

**(b) You measure 15% of peak TFLOPS but HBM bandwidth is saturated — will a bigger/faster-compute GPU help?**
**Answer:** **No.** Saturated bandwidth at low FLOP/s means you are **memory-bound** — sitting on the slanted memory roof, not the compute roof. The 15% is irrelevant; the workload never reaches the compute ceiling, so raising that ceiling does nothing. Buy **bandwidth** (e.g. H200 at 4.8 TB/s) or raise arithmetic intensity via **fusion/batching**. Paying for more FLOPs would be pure waste.

**(c) Approximate ridge-point AI for an H100?**
**Answer:** ridge = peak FLOP/s ÷ HBM bandwidth = `989e12 ÷ 3.35e12` ≈ **295 FLOP/byte** (dense BF16). In FP8 it roughly doubles to ~590 FLOP/byte, because compute doubles while bandwidth is unchanged — so lower precision *raises* the intensity a kernel needs to stay compute-bound.

## Resources

1. **Horace He — "Making Deep Learning Go Brrrr From First Principles"** — https://horace.io/brrr_intro.html — *(deep, read twice)* The module's conceptual anchor. Frames every workload as compute-bound, memory-bound (bandwidth), or overhead-bound, and shows why fusion is the memory-bound cure. Internalise this and the roofline becomes second nature.
2. **NVIDIA H100 Tensor Core GPU datasheet / architecture whitepaper** — https://resources.nvidia.com/en-us-tensor-core — *(skim)* Source of truth for the roofs (989/1,979 dense TFLOP/s, 3.35 TB/s HBM3, 132 SMs). Read the fine print to separate dense from with-sparsity figures for correct ridge-point math.
3. **Williams, Waterman & Patterson — "Roofline: An Insightful Visual Performance Model" (CACM 2009)** — https://dl.acm.org/doi/10.1145/1498765.1498785 — *(skim)* The original model; read for the formal ridge-point/arithmetic-intensity definitions so you can defend the framework rigorously.
