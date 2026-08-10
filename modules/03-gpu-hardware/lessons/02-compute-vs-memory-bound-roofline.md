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
sources: 10
---

# 03.2 · Compute-bound vs memory-bound and the roofline

> **Concept.** Compute arithmetic intensity, place a workload under the compute roof or the memory roof, find the ridge point — the single lens for the whole cost story.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 03.1 gave you the vocabulary to distrust `GPU-Util` and to read `SM_ACTIVE`, `TENSOR_ACTIVE`, and `DRAM_ACTIVE` as three separate, honest questions about a running GPU. What it didn't give you is a way to *predict*, before you run anything, whether a workload even has room to get faster — or which of those DCGM fields you should even expect to be high. This lesson supplies that: the roofline model turns "is my GPU busy" into "which physical ceiling — compute or memory bandwidth — is this specific workload up against, and is a bigger GPU capable of helping at all." It is the module's anchor lesson; everything from lesson 03.3 (memory hierarchy) through the 03.7 capstone is this model applied to a specific workload shape.

## Why this matters

This is the module spine. Every GPU cost decision reduces to one question: **which roof am I under, and am I near it?** If a workload is *memory-bound*, a faster-compute GPU buys you nothing — you would pay more per hour for zero speedup. If it is *compute-bound* and far from the roof, the fix is software (batching, fusion, precision), not a bigger SKU. Being able to place a workload on a roofline and name the *one* thing that can possibly help is the difference between a platform engineer who guesses at capacity and one who can justify a SKU choice or an optimisation to a finance review with a number.

This is precisely the competence NVIDIA's own *Solutions Architect* job description names explicitly: "inference optimization via FP16/INT8/FP8… arithmetic intensity vs peak compute and memory bandwidth using the roofline model to classify bottlenecks." An interviewer asking "100% util but terrible throughput — why?" or "why is decode memory-bandwidth bound, prefill compute-bound?" is asking you to run this lesson's decision procedure live, on the spot. Get it wrong — recommend a compute upgrade for a memory-bound workload — and you've just proposed a real six- or seven-figure hardware spend that will change nothing.

## What's new here (calibration)

The module README is explicit: you are not here to hand-derive rooflines per kernel, build cache-level hierarchical rooflines, or squeeze the last 5% via kernel scheduling — that is CUDA-kernel-developer depth, out of scope. What this lesson adds:

- **The formal roofline model** (Williams, Waterman & Patterson, 2009) as a first-order placement tool: estimate arithmetic intensity, compare to a ridge point, done. Not a research-grade model.
- **The practitioner's simplification** (Horace He's compute/memory/overhead trichotomy) layered on top, because the pure two-roof chart can't show the third, extremely common failure mode: overhead-bound.
- **The economics translation** — turning "which roof" into "$/unit of useful work," the exact quantity a finance review or a SKU-purchase decision needs.
- **How the ridge point itself moves** with precision (FP8 roughly doubles it) — the load-bearing fact that ties this lesson to lesson 03.5 (precision) and lesson 03.4 (batching, which changes effective AI).

## Core concepts

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

This is the formal model from Samuel Williams, Andrew Waterman, and David Patterson's "Roofline: An Insightful Visual Performance Model for Multicore Architectures" (*Communications of the ACM*, Vol. 52, No. 4, 2009, pp. 65–76) — the original, citable source for everything in this section. Plot the model with AI on the x-axis (log) and attainable FLOP/s on the y-axis (log): a **slanted line** (the memory roof, slope = bandwidth) rising until it hits a **flat line** (the compute roof). Where they meet is the **ridge point**.

### The ridge point — machine balance

```
Ridge-point AI = peak_FLOP/s / peak_bandwidth   [FLOP/byte]
```

This is the GPU's *machine balance*: the arithmetic intensity a kernel needs just to stop being memory-bound. For an **H100 SXM in dense BF16**:

```
Ridge = 989e12 FLOP/s ÷ 3.35e12 B/s ≈ 295 FLOP/byte
```

Interpretation you can quote: **on an H100 you need ~295 FLOPs of work per byte of HBM traffic just to reach the compute roof.** Below that AI you are memory-bound no matter what; above it you *can* be compute-bound (if the kernel is efficient). Note the ridge *moves with precision*: FP8 dense (1,979 TFLOP/s) pushes it to ≈ **590 FLOP/byte** — lower precision doubles compute but not bandwidth, so it *raises* the bar to stay compute-bound. That is why memory-bound kernels get no benefit from FP8 — you'll see this exact mechanic again in lesson 03.5.

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

**A third real point, from lesson 03.1's Google Cloud benchmark:** B200 batch-1 LLM decode, BF16 weights (2 bytes/param, ~2 FLOP/param for the matmul) → arithmetic intensity ≈ **1 FLOP/byte** — even lower than SAXPY, and roughly **292× below the B200's own ridge point** (the article's own stated multiple). This is deep, deep in memory-bound territory — and it's not a toy kernel, it's a real production LLM-serving system.

**The general map:**

| Workload | Typical AI | Roof | What helps |
|---|---|---|---|
| Large GEMM / dense training | 10²–10³ | Compute | Faster-compute SKU, lower precision, tensor cores |
| Attention (fused, e.g. FlashAttention) | 10¹–10² | Near ridge | Fusion, bigger heads/batch |
| Elementwise, activations, norms, add | <1 | Memory | Fusion (fewer round-trips), faster-BW SKU, bigger dtype cuts AI |
| LLM inference decode (batch 1) | ~1 (bandwidth-bound) | Memory | Batching, KV-cache tricks, faster-BW SKU (H200) |

The last row is the one that surprises people and wins interviews: **autoregressive decode at small batch is memory-bound** — you stream the whole weight matrix from HBM to generate each token, doing very little arithmetic per byte. That is why H200 (141 GB HBM3e @ **4.8 TB/s**) beats H100 on decode throughput despite near-identical compute, and why "buy more FLOPs" is the wrong answer for inference serving.

### The third regime: overhead-bound

The roofline draws two roofs, but real workloads have a *third* way to be slow that the plot does not show: **overhead-bound** — the GPU spends its time on neither compute nor memory but on kernel-launch latency, Python dispatch, and CPU/host stalls (the flip side of lesson 03.1's launch-bound storms). A kernel that is *theoretically* compute-bound can measure at 10% of the roof simply because 90% of wall-clock is launch overhead and gaps. The tell: achieved FLOP/s is far below *both* roofs, and neither DRAM_ACTIVE nor TENSOR_ACTIVE is saturated.

Why it matters for placement: **before you conclude "compute-bound, buy a faster GPU," rule out overhead.** If you are overhead-bound, the fix is `torch.compile`/CUDA-graphs/fusion (eliminate launches) and it is *free* — a bigger SKU would run the *same* overhead and show the *same* low utilisation at higher cost. Horace He's framing, from "Making Deep Learning Go Brrrr From First Principles" (the module's conceptual anchor resource), is the clean version: every workload is compute-bound, memory-bound, or overhead-bound, and you must know which before spending a dollar. He argues operator fusion is "the single most impactful technique" for exactly this reason — it is the one lever that helps both the memory-bound and the overhead-bound case simultaneously.

A rigorous, free, and notably modern complement to the 2009 CACM paper is Google DeepMind's open textbook **"How to Scale Your Model,"** specifically its ["All About Rooflines" chapter](https://jax-ml.github.io/scaling-book/roofline/) (2025) — written by a frontier AI lab specifically for the ML-systems audience this module targets, including a bonus chapter on how GPU rooflines differ from TPU rooflines. Worth the read alongside Horace He's piece once the mechanics above are solid.

### From roofline to dollars: cost per unit of work

The roofline becomes a cost model the moment you divide by price. Cost per unit of useful work =

```
$/unit-work = (GPU $/hr) / (achieved useful throughput)
```

Worked comparison — serving LLM decode (memory-bound, per the table above) on H100 vs H200, using bandwidth as the throughput proxy since decode scales with HBM bandwidth:

| SKU | HBM BW | On-demand ≈ $/hr (2026 snapshot) | Relative decode throughput | Relative $/token |
|---|---|---|---|---|
| H100 SXM | 3.35 TB/s | ~$3.0 | 1.00× | 1.00 |
| H200 SXM | 4.8 TB/s | ~$3.7 | ~1.43× | **~0.86** |

Even at a ~23% higher hourly rate, the H200 is **~14% cheaper per token** for this memory-bound workload — *because* the roofline says decode rides the memory roof, and the H200 raises exactly that roof. Run the identical comparison for a compute-bound training GEMM and the ranking can invert (you would weigh FP8 TFLOP/s, not bandwidth). **The roof you are under decides which SKU minimises cost per unit of work** — that sentence is the whole reason this lesson exists. (Dollar figures here are a 2026-era illustrative snapshot, not a live quote — see lesson 03.7 for how to pull current, tier-aware pricing.)

### The decision procedure (the whole point)

1. Estimate AI = FLOPs ÷ HBM bytes for the workload.
2. Compare to ridge (≈295 FLOP/byte dense BF16 on H100).
3. **AI ≫ ridge → compute-bound.** Levers: lower precision (FP8/tensor cores), a higher-FLOP SKU, better GEMM tiling. A faster GPU *can* pay off.
4. **AI ≪ ridge → memory-bound.** Levers: **fusion** (do more per byte → raises AI), **batching**, a higher-*bandwidth* SKU (H200). A higher-FLOP-but-same-BW GPU pays off *nothing* — do not buy it.
5. **Near the roof already?** If achieved FLOP/s (or achieved bandwidth) is ~70–90% of the relevant ceiling, stop optimising that layer — you are done; the remaining win is a different SKU or a different algorithm.
6. **Rule out overhead first.** If achieved FLOP/s is far below *both* roofs and neither DRAM_ACTIVE nor TENSOR_ACTIVE is saturated, you are overhead-bound — fix launch/fusion before touching hardware at all.

## Perspectives

**Theory.** The formal roofline (Williams, Waterman & Patterson, 2009): `Attainable = min(peak_FLOPs, AI × peak_BW)`. This is the rigorous, citable version of the model — the one to reach for when you need to defend the framework itself, not just apply it.

**Practice.** Horace He's compute/memory/overhead trichotomy is the practitioner's simplification of the same model, plus the crucial addition — overhead-bound — that the pure roofline chart literally cannot depict, because it isn't a function of AI at all. Horace He is a PyTorch core team engineer diagnosing real model slowdowns with this exact framework in production, not an academic exercise.

**Hardware.** The ridge point is a hardware constant — `peak_FLOPs / peak_BW` — that shifts *with precision* (FP8 roughly doubles it, since compute scales but bandwidth doesn't). This single fact is the load-bearing bridge between this lesson and lesson 03.5: a kernel's *bound* is not fixed, it depends on what precision you run it at.

**Economics.** The roofline is dollars the instant you divide by $/hr. Google Cloud's B200 decode benchmark (4.4% FLOPS-util, 292× below ridge) turns into a real cost-per-token argument at production scale the moment you attach a rental rate to it — this is the exact move lesson 03.7's capstone asks you to make for your own SKU recommendation.

## Real-world use cases

- **Google Cloud, ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0).** Directly computes the B200 ridge point and shows decode sitting 292× below it — the single best "roofline in production" story available; anchors this lesson's real-world section. What it shows: a formal roofline calculation, done by a hyperscaler, on live production LLM-serving traffic.
- **Horace He, ["Making Deep Learning Go Brrrr From First Principles"](https://horace.io/brrr_intro.html).** The module's spine resource, and legitimately a real-world use case in its own right: a working PyTorch-core engineer's account of diagnosing real model slowdowns via this exact compute/memory/overhead framework. What it shows: the roofline model applied by a practitioner to production ML-framework performance work, not textbook exercises.
- **NERSC, ["Roofline Performance Model" docs](https://docs.nersc.gov/tools/performance/roofline/).** A US national lab's (Lawrence Berkeley/NERSC — the same lab that produced the original roofline paper) production HPC-facility guide to applying roofline analysis to real scientific-computing kernels on real supercomputers. What it shows: the roofline model's origin lab using it operationally, decades after publishing it — good evidence the model has held up. (Found via search; not independently fetched in this environment — cite as a deeper-dive reference, not a fully confirmed production case, per the sourcing note below.)

*Note on scope:* a dedicated CoreWeave- or Anthropic-authored blog specifically framed as "we placed workload X on a roofline in production" was not found beyond the Google Cloud piece above — flagged honestly rather than invented.

## Worked example

Rent one H100 SXM. Run two kernels and interpret them on the roofline.

**(1) Compute-bound: `torch.matmul`, N = 8192, BF16.**
- Theoretical FLOPs = `2 · 8192³` ≈ 1.10e12 FLOP per matmul.
- Suppose you measure ~1,200 matmuls/s → achieved ≈ 1.10e12 × 1,200 ≈ **1.32e15 FLOP/s ≈ 780 TFLOP/s ≈ 79% of the 989 dense roof.** AI ≈ 2,730 → plot lands on the **flat (compute) roof**, near it. Reading: near the ceiling; the only lever left is FP8 (→1,979 roof) or a newer SKU.

**(2) Memory-bound: SAXPY on a 1 GB FP32 buffer.**
- Bytes ≈ 3 GB traffic; measure ~1.0 ms → `3e9 / 1e-3` = **3.0 TB/s ≈ 90% of the 3.35 TB/s HBM roof.** FLOP/s ≈ `2N / t` ≈ 0.5 TFLOP/s. AI ≈ 0.167 → plot lands on the **slanted (memory) roof**, near it. Reading: bandwidth-saturated at 0.05% of compute peak — a bigger-compute GPU is worthless; only more bandwidth (H200) or an algorithm with higher AI helps.

**(3) The B200 decode point, plotted for scale (from lesson 03.1's real-world case, not rerun on your rented H100 — included to complete the picture):** AI ≈ 1 FLOP/byte, deep under the memory roof, ~292× below B200's ridge. This third point shows a *legitimately* memory-bound production workload sitting even further from the ridge than SAXPY — the fix there is architectural (batching, disaggregation — lesson 03.4), not a kernel rewrite.

**Roofline sketch (three points):**

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
      │● B200 decode /
      │  (AI≈1)    /
      └──────────────┴─────────────────────────  Arithmetic Intensity (log)
      1  0.17        295        2730
```

Three points, one ridge, opposite roofs — the entire cost thesis in one figure: *GEMM is near the compute roof (upgrade compute to go faster); SAXPY and decode are near the memory roof (upgrade bandwidth or batch/fuse, never compute)* — and the decode point in particular is the one that maps directly to real dollars at inference-serving scale.

## Practice

**Task (rent one H100 or A100 by the hour; budget ~1–1.5 hr ≈ $3–6 — reuse the session from lesson 03.1 if possible).**

1. **Compute-bound point:** run a large square `torch.matmul` (N ≈ 8192) in BF16 in a timed loop; compute achieved TFLOP/s from `2·N³ × iters ÷ seconds`; record it as a fraction of the 989 dense-BF16 roof.
2. **Memory-bound point:** run a SAXPY/elementwise op over a large buffer (≥1 GB); compute achieved GB/s = `bytes_moved ÷ seconds`; record it as a fraction of the 3.35 TB/s roof, and its achieved TFLOP/s.
3. For each, compute AI by hand and place it on a **hand-drawn roofline** (x = AI log-scale, y = achieved FLOP/s log-scale). Mark the **ridge point** at AI ≈ 295.
4. Cross-check each point's placement against the DCGM fields from lesson 03.1: the GEMM point should show high `TENSOR_ACTIVE`; the SAXPY point should show high `DRAM_ACTIVE` and low `TENSOR_ACTIVE`. If they don't match, you may be overhead-bound rather than where you think you are — investigate before trusting the roofline placement.

**Acceptance:** a roofline plot (hand-drawn is fine) carrying **both measured points** with their AI and achieved FLOP/s labelled, the ridge point marked, and a one-line verdict per point ("compute-bound, ~79% of roof → only precision/SKU helps" / "memory-bound, ~90% of BW → only bandwidth/fusion helps"). This figure is the centrepiece of the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md).

## Common pitfalls

1. **Treating "low utilization" as inherently a problem to fix**, rather than checking which roof you're under first. The Google Cloud B200 case directly rebuts this — 4.4% FLOPS-util was the correct outcome for that workload's arithmetic intensity, not a bug.
2. **Forgetting the ridge point moves with precision.** FP8 raises it to ~590 FLOP/byte on H100, so a kernel that's "compute-bound at FP16" can flip to memory-bound at FP8 — using the dense BF16 ridge to evaluate an FP8 kernel silently misclassifies it.
3. **Conflating overhead-bound with memory-bound.** Both show low achieved-FLOPS, but the fix is opposite: fusion/launch-elimination for overhead-bound, bandwidth/batching for memory-bound. Check DRAM_ACTIVE and TENSOR_ACTIVE (lesson 03.1) before diagnosing which one you're looking at.
4. **Using the sparse compute-roof number instead of dense** when computing the ridge point — silently doubles the AI threshold and makes workloads look more memory-bound than they are.
5. **Assuming roofline placement is static.** Batching changes a workload's *effective* AI (this is lesson 03.4's crossover point), so "compute-bound vs memory-bound" is a property of *(algorithm × batch size)*, not algorithm alone — a workload that's memory-bound at batch=1 can become compute-bound at batch=256.

## Self-check

- Arithmetic intensity of an FP16 matmul vs an elementwise add — roughly? **Answer:** Matmul (square N) has AI ≈ **N/3** — hundreds to thousands of FLOP/byte (e.g. ~2,730 at N=8192), so compute-bound. Elementwise add has AI ≈ **1/6 ≈ 0.17** FLOP/byte (one FLOP per three element accesses × 2 B), so memory-bound. Difference of roughly **four orders of magnitude** — matmul reuses each loaded byte across the whole inner dimension; add touches each byte once.
- You measure 15% of peak TFLOPS but HBM bandwidth is saturated — will a bigger/faster-compute GPU help? **Answer:** **No.** Saturated bandwidth at low FLOP/s means you are **memory-bound** — sitting on the slanted memory roof, not the compute roof. The 15% is irrelevant; the workload never reaches the compute ceiling, so raising that ceiling does nothing. Buy **bandwidth** (e.g. H200 at 4.8 TB/s) or raise arithmetic intensity via **fusion/batching**. Paying for more FLOPs would be pure waste.
- Approximate ridge-point AI for an H100? **Answer:** ridge = peak FLOP/s ÷ HBM bandwidth = `989e12 ÷ 3.35e12` ≈ **295 FLOP/byte** (dense BF16). In FP8 it roughly doubles to ~590 FLOP/byte, because compute doubles while bandwidth is unchanged — so lower precision *raises* the intensity a kernel needs to stay compute-bound.
- A GKE benchmark reports a B200 running LLM decode at 4.4% FLOPS utilization but serving 1M tokens/sec. Is this a performance bug? **Answer:** No — decode AI ≈ 1 FLOP/byte sits roughly 292× below B200's ridge point, so the workload is fundamentally memory-bound at the single-stream level; the low per-request utilization is the correct roofline ceiling for that workload, not waste. The 1M tok/s aggregate comes from batching across many concurrent decode streams to fill the same fixed bandwidth budget — the fix for "more throughput" is batching/disaggregation architecture (lesson 03.4), not chasing a higher single-stream utilization number.

## Connections & what's next

This lesson is the module's anchor: the DCGM fields from 03.1 (`DRAM_ACTIVE`, `TENSOR_ACTIVE`) are how you verify a roofline placement in practice; the ridge point's dependence on precision is exactly what lesson 03.5 exploits as a cost lever; the "batching shifts effective AI" pitfall above is lesson 03.4's entire argument, worked out in full; and the $/unit-work formula here is the direct ancestor of lesson 03.7's capstone cost model. Every subsequent lesson in this module is, in some sense, "apply the roofline to a new specific case."

Next: **[03.3 · Memory hierarchy & HBM bottleneck](03-memory-hierarchy-hbm.md)** takes the memory *roof* from this lesson — a single bandwidth number — and opens it up: what's actually competing for that bandwidth (weights vs. KV cache vs. activations), what determines whether a workload even *fits* in HBM, and why GQA/MQA and HBM generation both change the memory-bound picture this lesson only sketched.

## References & further reading

**Primary sources**
- Williams, Waterman & Patterson, ["Roofline: An Insightful Visual Performance Model for Multicore Architectures"](https://dl.acm.org/doi/10.1145/1498765.1498785), CACM 52(4), 2009, pp. 65–76 — the original roofline paper; read for the formal ridge-point/arithmetic-intensity definitions. Free mirror: [escholarship.org PDF](https://escholarship.org/content/qt78h8v7mr/qt78h8v7mr.pdf) (LBNL).
- Google DeepMind, ["How to Scale Your Model" — "All About Rooflines" chapter](https://jax-ml.github.io/scaling-book/roofline/), jax-ml/scaling-book (2025) — a modern, free, frontier-lab-authored complement to the 2009 paper, including a GPU-specific chapter (parent repo confirmed live at `github.com/jax-ml/scaling-book`).
- NVIDIA H100 Tensor Core GPU Architecture whitepaper — https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper — source of truth for the roofs (989/1,979 dense TFLOP/s, 3.35 TB/s HBM3); read the fine print to separate dense from with-sparsity figures for correct ridge-point math.

**Real-world engineering blogs**
- Google Cloud, ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — a formal roofline calculation on live B200 production traffic; this lesson's headline real-world case.
- Horace He, ["Making Deep Learning Go Brrrr From First Principles"](https://horace.io/brrr_intro.html) — the module's conceptual anchor; a PyTorch-core engineer's practitioner account of the compute/memory/overhead framework applied to real slowdowns. Read twice.

**Deeper dives**
- NERSC, ["Roofline Performance Model" docs](https://docs.nersc.gov/tools/performance/roofline/) — the roofline model's origin lab (Lawrence Berkeley/NERSC) applying it operationally to HPC workloads; found via search, not independently fetched in this environment — treat as a good deeper-dive pointer rather than a fully verified primary case.
- Programming Massively Parallel Processors (Kirk & Hwu) — vocabulary-level skim only, no exercises; keeps SM/warp/memory terms consistent with lesson 03.1.
