---
lesson: "03.5"
title: "Precision Formats & Tensor Cores"
module: "03"
concept: "Precision Formats & Tensor Cores"
status: not-started
est_time: "5h"
artifacts: []
---
# 03.5 · Precision Formats & Tensor Cores
> **Concept.** Numeric precision sets your memory footprint and your roofline ceiling; tensor cores turn low precision into throughput — but only for dense matmul, and FP8 is a direct $/token lever.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Why this matters

Precision is the one knob that moves *both* sides of the cost equation at once. Drop a model from FP16 to FP8 and you roughly **halve the bytes** every parameter and activation occupies (less HBM, higher effective bandwidth, bigger batch) **and** you roughly **double tensor-core throughput** (more tokens/sec on the same silicon). On a fleet where an H100 rents for a few dollars an hour, that is the difference between fitting a 70B model on one GPU or paying for two, and between serving 1× or 2× the tokens per dollar.

This is also where engineers who "know the hardware" separate from engineers who quote spec sheets. The spec sheet says an H100 does ~1,979 FP8 TFLOPS. Your workload will hit a fraction of that, because tensor cores accelerate *dense low-precision matmul and convolution* and sit **idle** on everything else — the softmax in attention, layernorm, activation functions, small batches, and anything memory-bound. Reasoning about cost per unit of work means knowing which of your FLOPs actually land on the tensor cores.

## The platform-engineer's lens

You are not choosing a quantization scheme — the ML team does that. Your job is to translate a precision decision into a **capacity, placement, and cost** statement, and to observe whether the tensor cores you are paying for are actually busy.

Three questions you own:

1. **Does it fit, and how much headroom?** Bytes/param × params + KV cache + activations vs. HBM capacity. Precision is the first lever before you reach for a second GPU or tensor parallelism.
2. **Are the tensor cores active?** DCGM exposes tensor-core utilization (`DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`). A model "on a GPU" at 5% tensor-active is memory-bound or launch-bound — you are paying for a matmul engine that is idle. This is your headline observability signal for GPU-efficiency.
3. **What did accuracy cost?** Lower precision can shift quality. You don't own the accuracy bar, but you own surfacing the *throughput/quality trade* so the org makes it on purpose, not by accident.

## Core notes

### Bytes per element — the table to memorize

| Format | Bytes/elem | Bits (E/M) | Typical use | Tensor-core accelerated? |
|---|---|---|---|---|
| FP32 | 4 | 8 exp / 23 mant | Master weights, reductions | No (FP32 CUDA cores) |
| TF32 | 4 (storage) | 8 exp / 10 mant | Ampere+ default matmul math | Yes (reads FP32, computes reduced) |
| BF16 | 2 | 8 exp / 7 mant | Training + inference default | Yes |
| FP16 | 2 | 5 exp / 10 mant | Inference, mixed-precision train | Yes |
| FP8 (E4M3 / E5M2) | 1 | 4/3 or 5/2 | Hopper+ inference & training | Yes (Transformer Engine) |
| INT8 | 1 | integer | PTQ inference | Yes |
| FP4 (E2M1) | 0.5 | 2/1 | Blackwell inference | Yes (2nd-gen TE) |

Two things to internalize. **TF32 is a compute mode, not a storage saving** — data still occupies 4 bytes; the tensor core just truncates the mantissa internally, so you get FP32-ish range at higher throughput with no memory win. **FP4 is half a byte** — two values packed per byte — which is why Blackwell inference numbers look enormous.

### Memory footprint: the arithmetic you'll do constantly

Weights alone: `params × bytes/param`.

| Model | FP32 | FP16/BF16 | FP8 | FP4 |
|---|---|---|---|---|
| 7B | 28 GB | 14 GB | 7 GB | 3.5 GB |
| 70B | 280 GB | 140 GB | **70 GB** | 35 GB |
| 405B | 1,620 GB | 810 GB | 405 GB | 203 GB |

A 70B model at FP16 is 140 GB — it does **not** fit on one 80 GB H100 and barely fits an H200 (141 GB) with almost no room for KV cache. At FP8 it is 70 GB and fits a single H100 with headroom for cache and activations. That single fact — one GPU vs. two — is the most common precision-driven cost decision you will make. (Real serving footprint also includes the KV cache, which scales with batch × sequence length × layers and *also* shrinks with lower precision, and activation/workspace memory.)

### The roofline connection

Every kernel is either **compute-bound** (limited by FLOPS) or **memory-bound** (limited by bytes moved from HBM). The crossover is *arithmetic intensity* = FLOPs per byte. Lowering precision helps each side differently:

- **Compute-bound dense matmul** (large GEMMs, prefill, training steps): FP8 roughly doubles the FLOPS ceiling → throughput scales close to 2×. This is where tensor cores earn their keep.
- **Memory-bound ops** (elementwise, layernorm, attention softmax overhead, single-token decode, small batch): the limit is bytes/sec off HBM, not FLOPS. Halving the bytes helps *bandwidth-bound* ops somewhat (fewer bytes to move), but there is no matmul to accelerate, so you do **not** get the 2× tensor speedup. The tensor cores sit idle.

This is why a real inference server sees *less* than the headline 2× when you flip FP16→FP8: the attention and softmax overhead, the KV-cache reads during decode, and the many small elementwise kernels don't scale with tensor throughput. You get the memory win (always) and a partial throughput win (workload-dependent).

### FP8 on Hopper — the Transformer Engine

Hopper's 4th-gen tensor cores add native FP8 with two encodings: **E4M3** (more mantissa, better precision — used for weights/activations in the forward pass) and **E5M2** (more range — used for gradients). NVIDIA's **Transformer Engine** library manages this automatically: it keeps per-tensor scaling factors, watches the dynamic range, and casts layer-by-layer so most matmuls run in FP8 while sensitive reductions stay higher precision. Net effect vs. FP16 on the same H100: **~½ the memory** for weights/activations and **~2× the tensor-core throughput** (1,979 vs ~990 dense TFLOPS). Blackwell's 2nd-gen Transformer Engine extends this to FP4 with finer-grained (micro-tensor) scaling.

### BF16 vs FP16 — same size, different failure mode

Both are 2 bytes, but the bit split differs. **FP16** = 5 exponent / 10 mantissa: more precision, but narrow dynamic range — values overflow to inf or underflow to zero easily, so FP16 training needs loss scaling to survive. **BF16** = 8 exponent / 7 mantissa: same range as FP32, less precision. Because gradients and activations span a huge dynamic range, **BF16 trains more stably** and is the default for modern training — you rarely need loss scaling. FP16 remains common in inference where range is controlled. Rule of thumb: *BF16 for range/stability, FP16 for precision within a known range.*

### The quantization trade

Going lower precision trades accuracy for throughput and footprint. FP8 with Transformer Engine is typically near-lossless for inference on well-behaved models; INT8 and FP4 push harder and can cost measurable quality unless you use good calibration (post-training quantization, PTQ) or quantization-aware training (QAT). Two levers matter operationally: **which tensors** you quantize (weights are easy; activations and KV cache are harder) and **granularity** of scaling (per-tensor is cheap, per-channel/per-block preserves more accuracy). Your report should state the throughput gain *and* the measured or vendor-cited accuracy delta, so the trade is explicit.

## Worked example

**Question: a 70B-parameter chat model, target one GPU per replica, what precision and what does it buy?**

Weights: 70B × 2 B (FP16) = **140 GB** → does not fit an 80 GB H100; needs 2× H100 (tensor-parallel) or an H200.
Weights at FP8: 70B × 1 B = **70 GB** → fits a single 80 GB H100 with ~10 GB for KV cache + activations.

Cost framing (illustrative on-demand rate $3/H100-hr):

- FP16 path: 2× H100 per replica = **$6/replica-hr**. Tensor-parallel adds NVLink all-reduce overhead per token.
- FP8 path: 1× H100 per replica = **$3/replica-hr**, *and* ~2× tensor throughput on the compute-bound matmuls, *and* room for a larger batch because you freed ~70 GB.

Suppose FP16 on 2 GPUs serves 1,000 tok/s/replica. Cost = $6/hr ÷ (1,000 × 3,600) tok/hr = **$1.67 per million tokens**.
FP8 on 1 GPU: tensor-bound sections ~2× faster but decode is memory-bound, so realistic end-to-end is ~1.6× → ~1,600 tok/s/replica. Cost = $3/hr ÷ (1,600 × 3,600) = **$0.52 per million tokens** — roughly a **3× cost improvement** (half the hardware × more throughput), before any accuracy check.

The decision you write up: *FP8 halves the hardware and cuts $/M-token ~3×; the open question is the accuracy delta, which we measure against the FP16 baseline before rollout.*

## Practice — feeds the deliverable

On a rented single GPU (H100 or H200 ideal, since FP8 needs Hopper+):

1. **Matmul throughput sweep.** Run a large square GEMM (e.g. 8192³) at FP16 and at FP8 and record TFLOPS achieved for each. Any of: a short PyTorch script timing `torch.matmul` with `torch.float16` vs. FP8 via Transformer Engine / `torch._scaled_mm`, or the CUDA samples / `cublasLt` benchmarks. Record the **throughput multiple** (expect ~1.7–2×).
2. **Memory halving.** For a small model or a weight tensor, load it FP16 then FP8 and capture `nvidia-smi --query-gpu=memory.used` (or `torch.cuda.memory_allocated()`) each way. Confirm the FP8 footprint is ~½.
3. **Tensor-core activity.** While each run executes, sample DCGM `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (via `dcgmi dmon -e 1004` or the DCGM exporter). Confirm the dense-matmul run drives tensor-active high, and note how it differs from an elementwise-only kernel (near-zero tensor-active) — proving the "tensor cores idle on memory-bound ops" claim.
4. **Accuracy note.** If you ran a small inference both ways, record any output/quality difference (or note "not measured — flagged as open").

**Acceptance:** the deliverable contains a measured **FP16-vs-FP8 throughput multiple and memory delta** from your rented GPU, plus a DCGM tensor-active reading showing dense-matmul vs. elementwise, and a one-line accuracy caveat.

## Self-check

**(a)** How much GPU memory to hold a 70B model's weights at FP16 vs. FP8?
**Answer:** FP16 = 70e9 × 2 B = **140 GB**; FP8 = 70e9 × 1 B = **70 GB**. FP16 needs two 80 GB H100s (or an H200); FP8 fits one 80 GB H100 with room for KV cache. (Serving adds KV cache + activations on top of both.)

**(b)** Why doesn't FP8 speed up a memory-bound elementwise kernel proportionally to its 2× tensor throughput?
**Answer:** An elementwise kernel has no matmul, so the tensor cores don't engage at all — its limit is HBM bandwidth (bytes/sec), not FLOPS. Halving the bytes gives a modest bandwidth-side win, but there is no 2× tensor speedup to capture. The 2× applies only to dense low-precision matmul/conv that actually runs on the tensor cores.

**(c)** What does moving from FP16 to FP8 do to your $/token and your max batch size?
**Answer:** It roughly halves the memory per parameter/activation, which frees HBM for a **larger batch** (more concurrent requests per GPU) and often lets you drop from two GPUs to one; combined with ~2× tensor throughput on compute-bound sections, end-to-end $/token typically falls ~2–3×. The caveat you must report is the accuracy delta vs. the FP16 baseline.

## Resources

1. **NVIDIA H100 / Hopper Architecture Whitepaper** — Transformer Engine and FP8 section; the authoritative source for FP8 encodings and the 1,979 FP8 TFLOPS figure. https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper
2. **NVIDIA TensorRT / inference quantization guide** — practical FP8 and INT8 quantization, calibration (PTQ), and accuracy-vs-throughput trade-offs. https://docs.nvidia.com/deeplearning/tensorrt/latest/inference-library/work-with-quantized-types.html
3. **NVIDIA Transformer Engine docs** — how per-tensor scaling and layer-wise casting actually manage FP8/FP4 in practice. https://docs.nvidia.com/deeplearning/transformer-engine/
