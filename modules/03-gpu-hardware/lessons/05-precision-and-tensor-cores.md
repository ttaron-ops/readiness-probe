---
lesson: "03.5"
title: "Precision Formats & Tensor Cores"
module: "03"
concept: "Precision & tensor-core throughput"
status: not-started
est_time: "7h"
prev: "04-decode-bandwidth-batching.md"
next: "06-generational-and-software-stack.md"
artifacts: []
sources: 8
---

# 03.5 · Precision Formats & Tensor Cores

> **Concept.** Numeric precision sets your memory footprint and your roofline ceiling; tensor cores turn low precision into throughput — but only for dense matmul, and FP8 is a direct $/token lever.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 4 established the decode bandwidth ceiling and showed batching and prefill/decode disaggregation as the ways to push a memory-bound workload back toward the roofline's ridge point. This lesson hands you a different lever entirely: instead of restructuring how a workload runs, you can move the ridge point itself. Numeric precision is the one decision that touches both axes of the roofline at once — it shrinks the bytes every memory-bound kernel from lessons 3 and 4 has to move, *and* it raises the FLOPS ceiling every compute-bound kernel from lesson 2 can reach. Lesson 6 will show this lever is also gated by which hardware generation you're renting — FP8 requires Hopper or newer, FP4 requires Blackwell — so the precision conclusion you reach here determines what SKU argument you're even allowed to make there.

## Why this matters

Precision is where "reads a spec sheet" and "reasons about cost" diverge. The NVIDIA Solutions Architect role this module calibrates against explicitly names "inference optimization via FP16/INT8/FP8" and "arithmetic intensity vs peak compute and memory bandwidth" as core competencies — and the standard interview probe is blunt: *"FP8 vs FP16 — what's the cost lever, and what do you actually get?"* Answering "roughly 2x" is the answer that gets you screened out at a GPU-fleet operator, because the real answer is "2x on the tensor-core-bound matmuls, and something meaningfully less end-to-end, and here's why."

The concrete cost of not knowing this: quoting the *sparse* FP8 TFLOPS figure as your denominator silently doubles your apparent efficiency in a report someone else will act on. Assuming a precision flag "just works" the same way on an A100 as an H100 is the kind of mistake that turns into a failed deploy at 2 a.m., because A100 (Ampere) has **no FP8 tensor-core hardware at all** — FP8 is a Hopper-generation feature. And treating a vendor's own 1.5x throughput claim as if it contradicts a textbook 2x number — instead of recognizing they're measuring different things — is the kind of confusion that makes a report look sloppy in front of a hiring panel that has actually shipped an FP8 training run.

## What's new here (calibration)

The module README skips CUDA kernel authoring, PTX/SASS, deep microarchitecture, and occupancy-tuning-as-coding — you are not writing the matmul kernel or hand-tuning register allocation. This lesson holds that line: it does not teach you to write a quantization calibration routine or a custom FP8 GEMM. What it adds instead:

- **Translating a precision decision into capacity, placement, and cost** — bytes/param arithmetic that decides "one GPU or two," not the quantization math itself.
- **Observability into whether the precision change is actually landing** — DCGM's tensor-core-active signal as the ground truth that a "we switched to FP8" claim is real, not aspirational.
- **The gap between the theoretical multiplier and the realized one, quantified** — Databricks' verified ~1.5x end-to-end throughput against the ~2x tensor-core multiple, and why both numbers are correct at once.
- **PTQ vs. QAT as a named operational trade-off** — you don't own the accuracy call, but a Staff-level platform engineer can name that Character.AI chose native int8 *training* over post-hoc quantization specifically to eliminate train/serve mismatch, and explain why that's a real, deliberate architectural choice and not a detail.

## Core concepts

### Bytes per element — the table to memorize

| Format | Bytes/elem | Bits (E/M) | Typical use | Tensor-core accelerated? |
|---|---|---|---|---|
| FP32 | 4 | 8 exp / 23 mant | Master weights, reductions | No (FP32 CUDA cores) |
| TF32 | 4 (storage) | 8 exp / 10 mant | Ampere+ default matmul math | Yes (reads FP32, computes reduced) |
| BF16 | 2 | 8 exp / 7 mant | Training + inference default | Yes |
| FP16 | 2 | 5 exp / 10 mant | Inference, mixed-precision train | Yes |
| FP8 (E4M3 / E5M2) | 1 | 4/3 or 5/2 | Hopper+ inference & training | Yes (Transformer Engine) |
| INT8 | 1 | integer | PTQ inference | Yes |
| FP4 (E2M1) | 0.5 | 2/1 | Blackwell inference | Yes (2nd-gen Transformer Engine) |

Two things to internalize. **TF32 is a compute mode, not a storage saving** — data still occupies 4 bytes; the tensor core just truncates the mantissa internally, so you get FP32-ish range at higher throughput with no memory win. **FP4 is half a byte** — two values packed per byte — which is why Blackwell inference TFLOPS numbers look enormous, and why it only exists on hardware new enough to unpack it (see lesson 6).

### The hardware gate: FP8 is not available everywhere

This is worth stating as its own fact rather than leaving it implicit in a generation table: **A100 (Ampere) has no FP8 tensor-core pipeline.** FP8 tensor cores were introduced in Hopper's 4th-generation tensor core. On an A100, "FP8" is only achievable via software emulation/casting — there is no hardware path, and no throughput win to claim. If a report or a colleague says "we're running this model in FP8 on our A100 fleet," that claim needs a follow-up question, because the accelerated path they're describing does not exist on that silicon. Precision is not purely a software configuration flag; it is gated by which generation you're renting, which is exactly the thread lesson 6 picks up.

### Memory footprint: the arithmetic you'll do constantly

Weights alone: `params × bytes/param`.

| Model | FP32 | FP16/BF16 | FP8 | FP4 |
|---|---|---|---|---|
| 7B | 28 GB | 14 GB | 7 GB | 3.5 GB |
| 70B | 280 GB | 140 GB | **70 GB** | 35 GB |
| 405B | 1,620 GB | 810 GB | 405 GB | 203 GB |

A 70B model at FP16 is 140 GB — it does **not** fit on one 80 GB H100 and barely fits an H200 (141 GB) with almost no room for KV cache. At FP8 it is 70 GB and fits a single H100 with headroom for cache and activations. That single fact — one GPU vs. two — is the most common precision-driven cost decision you will make. Real serving footprint also includes the KV cache, which scales with batch × sequence length × layers and *also* shrinks with lower precision (lesson 3 covered the KV-cache formula; precision and attention-head-grouping are two independent, stackable levers on the same number), plus activation/workspace memory that a "does it fit" calculation can't skip.

### The roofline connection

Every kernel is either **compute-bound** (limited by FLOPS) or **memory-bound** (limited by bytes moved from HBM). The crossover is *arithmetic intensity* = FLOPs per byte, and the ridge point is a hardware constant — `peak FLOPs / peak bandwidth` — that **moves with precision**. Lowering precision helps each side differently:

- **Compute-bound dense matmul** (large GEMMs, prefill, training steps): FP8 roughly doubles the FLOPS ceiling, so the *ridge point itself* moves right — a kernel that was comfortably compute-bound at FP16 needs a proportionally higher arithmetic intensity to stay compute-bound at FP8. Throughput on these kernels scales close to 2x. This is where tensor cores earn their keep.
- **Memory-bound ops** (elementwise, layernorm, attention softmax overhead, single-token decode, small batch): the limit is bytes/sec off HBM, not FLOPS. Halving the bytes helps *bandwidth-bound* ops somewhat (fewer bytes to move), but there is no matmul to accelerate, so you do **not** get the 2x tensor speedup. The tensor cores sit idle — literally 0% tensor-active for a pure elementwise kernel, however fast the memory system is.

This is why a real inference server sees *less* than the headline 2x when you flip FP16→FP8: the attention and softmax overhead, the KV-cache reads during decode, and the many small elementwise kernels don't scale with tensor throughput. You get the memory win (always) and a partial throughput win (workload-dependent) — the exact shape of the gap is the next section's subject, with a real number attached to it.

### FP8 on Hopper — the Transformer Engine

Hopper's 4th-gen tensor cores add native FP8 with two encodings, formalized in the joint NVIDIA/Arm/Intel paper **"FP8 Formats for Deep Learning"** (Micikevicius et al., arXiv 2209.05433, Sept 2022): **E4M3** (more mantissa bits, better precision — used for weights/activations in the forward pass) and **E5M2** (more exponent bits, wider range — used for gradients, which can span a much larger dynamic range during backprop). NVIDIA's **Transformer Engine** library manages this automatically: it keeps per-tensor scaling factors, watches the dynamic range at runtime, and casts layer-by-layer so most matmuls run in FP8 while sensitive reductions stay in higher precision. Net effect vs. FP16 on the same H100: **~½ the memory** for weights/activations and, at the tensor-core level, **~2x the raw throughput** (1,979 vs ~990 dense TFLOPS). Blackwell's 2nd-generation Transformer Engine extends this scheme to FP4 with finer-grained (micro-tensor, sub-block) scaling — a coarser per-tensor scale factor isn't precise enough to keep FP4 usable, so the hardware had to get smarter about *how* it scales, not just *how small* the format is.

### The realized multiple vs. the theoretical multiple — a verified gap

The tensor-core-level multiple (~2x FP8 vs FP16) is a hardware fact. What you'll actually observe end to end is smaller, and you should be able to quote a real number for *why*, not just assert "less than 2x." Databricks/MosaicML's published FP8 training results report **>50% MFU** (among the highest published at the time) and **~1.5x realized training throughput** from FP8 plus other stack optimizations, on real large-scale training runs. The gap between the hardware's ~2x and the observed ~1.5x is not a discrepancy to explain away — it's the same Amdahl's-law-style dilution the roofline section above predicts: end-to-end training time includes data loading, optimizer steps, cross-GPU communication, layernorm, and softmax — none of which run on the tensor cores and none of which get faster when you halve the matmul's precision. A realized 1.5x on a pipeline that's partially non-matmul is exactly what the theory predicts; a report that quotes a flat "2x from FP8" without this caveat will not survive scrutiny from someone who has actually run the numbers.

Distinguish this from a *second*, differently-shaped worked example later in this lesson: a footprint change (2 GPUs → 1 GPU) **compounds** with a throughput change, producing a bigger overall $/token improvement than the throughput multiple alone. Databricks' 1.5x is throughput-only on a fixed hardware footprint; this lesson's own worked example below compounds a footprint halving with a throughput gain. Both numbers are correct — they're measuring different things, and being explicit about which one you're citing is the discipline that separates a defensible cost argument from a sound-bite.

### BF16 vs FP16 — same size, different failure mode

Both are 2 bytes, but the bit split differs. **FP16** = 5 exponent / 10 mantissa: more precision, but narrow dynamic range — values overflow to inf or underflow to zero easily, so FP16 training needs loss scaling to survive. **BF16** = 8 exponent / 7 mantissa: same range as FP32, less precision. Because gradients and activations span a huge dynamic range, **BF16 trains more stably** and is the default for modern training — you rarely need loss scaling. FP16 remains common in inference where the range is controlled. Rule of thumb: *BF16 for range/stability, FP16 for precision within a known range.*

### The quantization trade — and the PTQ-vs-QAT choice you must be able to name

Going lower precision trades accuracy for throughput and footprint. FP8 with Transformer Engine is typically near-lossless for inference on well-behaved models; INT8 and FP4 push harder and can cost measurable quality unless you use good calibration. Two operational paths exist, and platform engineers should be able to name both even though the ML team makes the call:

- **PTQ (post-training quantization).** Train in higher precision, then quantize the finished checkpoint using a calibration dataset to set scale factors. Cheap, fast, the default assumption most engineers reach for.
- **QAT (quantization-aware training) / native low-precision training.** Train *in* the target low precision from the start, so the model learns to be robust to it rather than being quantized after the fact. Character.AI's engineering blog is explicit that they train natively in int8 rather than quantizing post-hoc, stating this "eliminates the risk of training/serving mismatch" — a mismatch that PTQ can introduce if the calibration distribution doesn't match production traffic.

Two levers matter operationally regardless of which path: **which tensors** you quantize (weights are easy; activations and KV cache are harder, and KV-cache quantization is an independent lever from weight quantization — you can compress one without the other) and **granularity** of scaling (per-tensor is cheap, per-channel/per-block preserves more accuracy, which is exactly why FP4 needed the finer-grained scaling mentioned above). Your report should state the throughput gain *and* the measured or vendor-cited accuracy delta, so the trade is explicit rather than assumed.

## Perspectives

**Developer/ML view.** The choice between PTQ and QAT is made at training time, by people who own the training recipe — but it fixes what a platform engineer can later promise. Character.AI's decision to train natively in int8 (rather than the more common PTQ-after-training path) is a real, deliberate engineering choice with a stated rationale (eliminating train/serve mismatch), not an implementation detail. Knowing this distinction — and which one your org uses — changes what accuracy guarantee you can respond with when someone asks "if we go lower precision, what breaks?"

**Operator/observability view.** `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (field ID 1004, the same field lesson 1 introduced for the utilization-lie discussion) is your direct evidence that a precision change actually landed on the tensor cores rather than staying a config flag nobody verified. A model reported as "running in FP8" at 5% tensor-active is memory-bound or launch-bound regardless of what the config says — the precision change bought you the memory win but not the compute win, and your dashboard should be able to say so.

**Hardware view.** Precision support is gated generationally, not a universal software toggle: FP8 requires Hopper's 4th-gen tensor cores (A100 has none), FP4 requires Blackwell's 2nd-gen Transformer Engine with its finer-grained scaling. "Which precisions can this workload use" is inseparable from "which generation is it running on" — a fact this lesson states and lesson 6 turns into a purchasing argument.

**Economics view.** Databricks' verified ~1.5x *realized* end-to-end throughput against the ~2x *theoretical* tensor-core multiple is the single cleanest illustration in this module of "you get the memory win always, a partial throughput win depending on workload mix." Quantifying that gap — rather than rounding it up to "2x" in a report — is the difference between a number that survives review and one that doesn't.

## Real-world use cases

- **[Databricks/MosaicML, "Turbocharged Training: Optimizing the Databricks Mosaic AI Stack With FP8"](https://www.databricks.com/blog/turbocharged-training-optimizing-databricks-mosaic-ai-stack-fp8).** Real large-scale training results: **>50% MFU** (claimed the highest among published numbers at the time) and **~1.5x realized throughput** from FP8 plus stack optimizations — the concrete "theoretical 2x vs. realized 1.5x" number this lesson's Core concepts section builds on.
- **Character.AI, ["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) and ["Part Deux"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/).** Native int8 QAT training (not PTQ) combined with MQA and KV-sharing, at real production LLM-chat scale (billions of messages/day) — the clearest real example of a team choosing train-time quantization over post-hoc quantization on purpose, and stating why.
- **[Google Cloud, "What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) (Medium, official Google Cloud publication).** A benchmark on 96×B200 GPUs serving ~1M tok/s at decode batch=1 reports **tensor cores active only 1.5% of the time** even though GPU-Util reads near 100% — a real, current (B200-generation) demonstration of exactly the "precision doesn't help if the kernel never reaches the tensor cores" point this lesson's roofline section makes, reused here through the tensor-core-observability lens rather than the raw-utilization lens lesson 1 used it for.

## Worked example

**Question: a 70B-parameter chat model, target one GPU per replica — what precision, and what does it buy?**

Weights: 70B × 2 B (FP16) = **140 GB** → does not fit an 80 GB H100; needs 2× H100 (tensor-parallel) or an H200.
Weights at FP8: 70B × 1 B = **70 GB** → fits a single 80 GB H100 with ~10 GB for KV cache + activations.

Cost framing (illustrative on-demand rate, $3/H100-hr — a **2026 snapshot figure**, refresh at study/build time against lesson 7's live pricing sources):

- FP16 path: 2× H100 per replica = **$6/replica-hr**. Tensor-parallel adds NVLink all-reduce overhead per token.
- FP8 path: 1× H100 per replica = **$3/replica-hr**, *and* ~2x tensor throughput on the compute-bound matmuls, *and* room for a larger batch because you freed ~70 GB.

Suppose FP16 on 2 GPUs serves 1,000 tok/s/replica. Cost = $6/hr ÷ (1,000 × 3,600) tok/hr = **$1.67 per million tokens**.
FP8 on 1 GPU: tensor-bound sections ~2x faster but decode is memory-bound, so realistic end-to-end is ~1.6x → ~1,600 tok/s/replica. Cost = $3/hr ÷ (1,600 × 3,600) = **$0.52 per million tokens** — roughly a **3x cost improvement**, before any accuracy check.

**Why this "3x" doesn't contradict Databricks' verified "1.5x."** This worked example's ~3x is a *compounded* result: halving the hardware footprint (2 GPUs → 1 GPU, a 2x on its own) **times** an effective ~1.6x throughput gain on the surviving GPU. Databricks' ~1.5x is throughput-only, measured on an *already-fixed* hardware footprint (same GPU count before and after, just FP8 vs. FP16). The two numbers are not in tension — they answer different questions ("what does FP8 buy me if I also right-size the fleet" vs. "what does FP8 buy me on the fleet I already have"). State explicitly which question your own report is answering; conflating them is the fastest way to produce a defensible-looking number that falls apart under a follow-up question.

The decision you write up: *FP8 halves the hardware and cuts $/M-token roughly 3x on this footprint-plus-throughput view (or ~1.5-1.6x on a throughput-only view at fixed hardware); the open question is the accuracy delta, which we measure against the FP16 baseline before rollout.*

## Practice — feeds the deliverable

On a rented single GPU (H100 or H200 ideal, since FP8 needs Hopper+):

1. **Matmul throughput sweep.** Run a large square GEMM (e.g. 8192³) at FP16 and at FP8 and record TFLOPS achieved for each. Any of: a short PyTorch script timing `torch.matmul` with `torch.float16` vs. FP8 via Transformer Engine / `torch._scaled_mm`, or the CUDA samples / `cublasLt` benchmarks. Record the **throughput multiple** (expect ~1.7–2x on the raw matmul — this is where the tensor-core-level number belongs, distinct from your realized end-to-end number).
2. **Memory halving.** For a small model or a weight tensor, load it FP16 then FP8 and capture `nvidia-smi --query-gpu=memory.used` (or `torch.cuda.memory_allocated()`) each way. Confirm the FP8 footprint is ~½.
3. **Tensor-core activity.** While each run executes, sample DCGM `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` (via `dcgmi dmon -e 1004` or the DCGM exporter). Confirm the dense-matmul run drives tensor-active high, and run a small elementwise-only kernel alongside it to confirm near-zero tensor-active — proving the "tensor cores idle on memory-bound ops" claim with your own telemetry, not just the lesson's assertion.
4. **Accuracy note.** If you ran a small inference both ways, record any output/quality difference (or note "not measured — flagged as open"). Note in your write-up whether your model/framework used PTQ-style post-hoc casting or trained natively in low precision — this determines how much you should trust the accuracy delta you observe.

**Acceptance:** the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md) contains a measured **FP16-vs-FP8 throughput multiple and memory delta** from your rented GPU, a DCGM tensor-active reading showing dense-matmul vs. elementwise, and a one-line accuracy caveat that states whether the multiple you're reporting is raw-kernel or end-to-end.

## Common pitfalls

1. **Assuming FP8 works identically on A100 and H100.** It doesn't — A100 has no FP8 tensor-core hardware at all; "FP8 on A100" means software emulation with no throughput win, not an accelerated path.
2. **Conflating the tensor-core throughput multiple (~2x, hardware-level) with realized end-to-end throughput (often ~1.5-1.6x, workload-level).** Databricks' own published numbers make this gap concrete and citable — use their number instead of rounding up to "2x."
3. **Quoting the sparse FP8 TFLOPS figure as your MFU or roofline denominator.** This silently doubles your apparent efficiency; always confirm you're citing the dense figure unless the workload genuinely exploits 2:4 structured sparsity.
4. **Assuming PTQ is the only path to low precision.** Character.AI's native int8 training shows train-time quantization (QAT) is a real, chosen-on-purpose alternative with different — and in their framing, better — accuracy trade-offs than post-hoc calibration.
5. **Trusting a "running in FP8" claim without checking tensor-active telemetry.** A config flag is not proof of an accelerated kernel path; `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` near zero on a supposedly-FP8 workload means the precision change bought memory savings but not the compute win.

## Self-check

- How much GPU memory does a 70B model's weights need at FP16 vs. FP8? **Answer:** FP16 = 70e9 × 2 B = **140 GB**; FP8 = 70e9 × 1 B = **70 GB**. FP16 needs two 80 GB H100s (or one H200 with almost no headroom); FP8 fits one 80 GB H100 with room for KV cache. (Real serving footprint adds KV cache + activations on top of both.)
- Why doesn't FP8 speed up a memory-bound elementwise kernel proportionally to its ~2x tensor-core throughput? **Answer:** An elementwise kernel has no matmul, so the tensor cores don't engage at all — its limit is HBM bandwidth (bytes/sec), not FLOPS. Halving the bytes gives a modest bandwidth-side win, but there is no 2x tensor speedup to capture. The 2x applies only to dense low-precision matmul/conv that actually runs on the tensor cores — verifiable directly via `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`.
- Databricks reports ~1.5x realized training throughput from FP8, while raw tensor-core FP8 throughput is ~2x FP16. Why the gap, and does it contradict this lesson's own ~3x worked-example figure? **Answer:** End-to-end training includes non-matmul, non-tensor-core-accelerated portions (data loading, optimizer steps, communication, layernorm/softmax) that don't get the FP8 speedup — the realized multiple is diluted the same way an elementwise kernel is unaffected by precision. It does not contradict this lesson's ~3x figure, because that figure additionally compounds a *footprint* halving (2 GPUs → 1 GPU) with the throughput gain; Databricks' 1.5x is throughput-only on a fixed footprint. Same physics, different question being answered.
- What does moving from FP16 to FP8 do to your $/token and your max batch size? **Answer:** It roughly halves the memory per parameter/activation, which frees HBM for a **larger batch** (more concurrent requests per GPU) and often lets you drop from two GPUs to one; combined with the realized throughput gain on compute-bound sections, end-to-end $/token typically falls somewhere in the 1.5-3x range depending on whether you're compounding a hardware-footprint reduction or measuring throughput alone. The caveat you must report every time is the accuracy delta vs. the FP16 (or native-precision) baseline.

## Connections & what's next

This lesson is the second half of the roofline-manipulation pair that started in lesson 2: lesson 2 taught you where a workload sits on the roofline, this one taught you how to move the roofline itself. It also closes the loop lesson 3 opened — GQA/MQA shrink the KV-cache *constant factor*; precision (including KV-cache quantization specifically, as Character.AI's stack shows) is an independent, stackable lever on the same number, not a competing one. The `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` telemetry here is the same field lesson 1 used to debunk the utilization lie — this lesson is where you learn to read it as a precision-verification signal specifically, not just a general business-vs-idle signal.

Next: **[03.6 · Generational SKUs & the software stack](06-generational-and-software-stack.md)** takes the hardware-gating fact from this lesson (FP8 needs Hopper+, FP4 needs Blackwell+) and turns it into a purchasing argument — which generation actually removes your workload's bottleneck, and what breaks in the driver/CUDA/NCCL stack if you get the pairing wrong.

## References & further reading

**Primary sources**
- Micikevicius et al. (NVIDIA, Arm, Intel), ["FP8 Formats for Deep Learning"](https://arxiv.org/abs/2209.05433), arXiv 2209.05433 (2022) — read for the E4M3/E5M2 rationale (precision vs. range trade-off) straight from the format's designers.
- [NVIDIA H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — read for the authoritative dense/sparse FP8 and FP16 TFLOPS figures and the Transformer Engine description.
- [NVIDIA DCGM documentation — feature overview](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html) — read for the canonical `DCGM_FI_PROF_*` field definitions, including field 1004.

**Real-world engineering blogs**
- Databricks/MosaicML, ["Turbocharged Training: Optimizing the Databricks Mosaic AI Stack With FP8"](https://www.databricks.com/blog/turbocharged-training-optimizing-databricks-mosaic-ai-stack-fp8) — what the theoretical-vs-realized FP8 gap looks like at real training scale (>50% MFU, ~1.5x throughput).
- Character.AI, ["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) and ["Part Deux"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/) — native int8 QAT training chosen over PTQ, at production chat scale.
- Google Cloud, ["What Does 4.4% GPU Utilization Actually Mean?"](https://medium.com/google-cloud/what-does-4-4-gpu-utilization-actually-mean-ee61fabebbf0) — a real B200 benchmark showing tensor cores active only 1.5% of the time despite near-100% GPU-Util, the clearest production evidence that a precision win and a utilization win are not the same claim.

**Deeper dives**
- Horace He, ["Making Deep Learning Go Brrrr From First Principles"](https://horace.io/brrr_intro.html) — the compute/memory/overhead-bound trichotomy that explains, in general form, why elementwise kernels never benefit from a precision-driven tensor-core speedup.
