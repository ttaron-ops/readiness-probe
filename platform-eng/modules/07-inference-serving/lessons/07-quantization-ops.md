---
lesson: "07.7"
title: "Quantization ops: the cost lever"
module: "07"
concept: "Quantization ops: the cost lever"
status: not-started
est_time: "6h"
prev: "06-alternative-servers-disaggregation.md"
next: "08-autoscaling-inference.md"
artifacts: []
sources: 8
---
# 07.7 · Quantization ops: the cost lever

> **Concept.** Quantization is the single largest lever on cost-per-million-tokens — but the saving is only honest once you have *measured* the accuracy delta, not asserted it.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

06 taught you to pick an **engine** for a workload shape (vLLM / SGLang / TensorRT-LLM / disaggregation) — a structural decision about where compute happens. This lesson is a **numerical-format** decision layered on top of whatever engine you picked: given the same GPUs and the same engine, which precision do you actually ship? It is lesson 07 of 10, sitting right after engine selection and right before autoscaling (08) — and the two are more connected than they look: a smaller quantized checkpoint loads faster from cold (09) and frees VRAM that autoscaling's batch-size headroom depends on. Quantization is also the second of the module's two explicit "cost decisions you defend at the checkpoint," after 05's batching curve.

## Why this matters

You already know FP8 as a *hardware capability* (03: Hopper/Blackwell tensor cores execute FP8 matmuls natively) and you already read `/metrics` for throughput and queue depth (05). This lesson is the **ops decision that sits on top of both**: given a model, a GPU fleet, and an SLO, which numeric format do you actually ship — and how do you prove the cost saving is real?

The stakes are money and trust. On an H100, switching a Llama-3.1-8B endpoint from FP16 to FP8 roughly **halves memory footprint and cuts cost-per-million-tokens (CPM) by ~40–50%** at the same GPU price. Ship that blindly and you can quietly degrade a code-generation product by several accuracy points because your calibration set was generic prose. The differentiator for a senior platform engineer is not "we turned on FP8" — it is "we cut CPM 43% and measured a +0.4pt / −0.1pt accuracy delta on MMLU and HumanEval before rollout." One is a slide; the other is an engineering decision.

## What's new here (calibration)

Three things you have *not* seen yet:

1. **Quantization as an OPS decision, not a hardware fact.** 03 told you FP8 tensor cores exist. Here you choose between FP8 / INT8 / 4-bit *as a fleet policy*, driven by GPU generation, VRAM budget, and workload shape.
2. **The calibration set as a first-class artifact.** Activation quantization derives scale factors from sample data. That sample data is now a thing you own, version, and can get *wrong*.
3. **"Measure the delta" as a gate.** No format ships without a paired throughput/CPM number *and* an accuracy number on a task slice that resembles production traffic.

## Core concepts

### The three formats you actually choose between

Notation: `WxAy` = x-bit weights, y-bit activations. "Weight-only" means activations stay FP16.

| Scheme | Bits (W/A) | Where it wins | Throughput vs FP16 (H100) | Accuracy posture | Needs calibration? |
|---|---|---|---|---|---|
| **FP8** (`W8A8-FP`, E4M3) | 8/8 | **Default on Hopper+/Blackwell.** Async, high-QPS, continuous batching | **~1.5–1.8× measured** (2× peak FLOPs) | Effectively **lossless** across model scales | Dynamic FP8 needs none; static needs a small set |
| **INT8** (`W8A8-INT`) | 8/8 | Cross-platform fallback (Ampere A100, L4, older) — no FP8 tensor cores there | ~1.3–1.7× | **1–3%** degradation when well-tuned | Yes (activation scales) |
| **AWQ / GPTQ 4-bit** (`W4A16`) | 4/16 | **VRAM-constrained fitting** — fit a bigger model on a smaller card, or a 70B on one H100. Synchronous / low-QPS | Weight-only: helps latency & memory, *not* compute-bound throughput | Competitive — rivals 8-bit on many tasks, but tail-sensitive | Yes (both are calibration-based) |

These bands are the headline result of the Neural Magic / IST Austria study **"Give Me BF16 or Give Me Death? Accuracy-Performance Trade-Offs in LLM Quantization"** (ACL 2025, arXiv:2411.02355 — Kurtic, Marques, Pandit, Kurtz, Alistarh; Neural Magic was acquired by Red Hat in 2025). Across **>500,000 evaluations** on the Llama-3.1 family they found: **8-bit quantization (FP8/INT8) recovers ~99.75% of the BF16 baseline on average**, and **W4A16-INT recovers ~99.36%**. Put plainly: "1–3% degradation" is a fair summary for well-tuned 8-bit *and* 4-bit, but the precise, citable numbers are 99.75% and 99.36% recovery — use those in a design doc, not a vague percentage. Their deployment rule of thumb matches the table: **W8A8 (FP8/INT8) dominates asynchronous continuous-batching serving; W4A16 is most cost-efficient for synchronous, latency- or VRAM-bound setups.** The >500k-eval scale is itself the methodological point — a single benchmark run "that looked fine" is not the same class of evidence, and it's the bar the deliverable's own accuracy-delta requirement is modeling at small scale.

### Why FP8 is the Hopper+/Blackwell default

- **Native tensor-core support** (03): the H100/H200/B200 execute E4M3 matmuls at ~2× the FP16 rate. You are not emulating.
- **KV cache halves too.** FP8 weights *and* an FP8 KV cache (`--kv-cache-dtype fp8`) roughly halve VRAM, which lets you raise `--max-num-seqs` / batch size. More concurrent sequences is where the *real* throughput comes from — often more than the raw FLOP win.
- **Near-zero accuracy loss.** E4M3's dynamic range covers LLM activation distributions well; with per-tensor or per-channel scales the error is in the noise for most tasks — consistent with the ~99.75% recovery figure above.
- **Dynamic FP8 needs no calibration file.** vLLM can quantize weights to FP8 at load and compute activation scales on the fly. Zero calibration risk, one flag.

The measured multiplier is **~1.5–1.8×**, not the "2×" peak FLOP number. Decode is memory-bandwidth-bound, not FLOP-bound; you capture the 2× only on the compute-heavy prefill and on the larger batches FP8 memory savings unlock. **Always quote the measured number, never the datasheet number.** NVIDIA's own TensorRT-LLM launch numbers for FP8 on H100 are datasheet/peak-FLOP framing (see References) — a useful contrast to keep on hand: it's a real number, just not the number you'll see on your own `bench serve` run once decode-bandwidth binding and real traffic shapes are in the mix.

### INT8 as the cross-platform fallback

If your fleet still has A100s, L4s, or T4s (no FP8 tensor cores), FP8 buys you nothing on those nodes — the ops on them fall back to a slow path. INT8 (`W8A8-INT`) runs fast on Ampere/Turing INT8 tensor cores and gives you a single quantized checkpoint that is fast *everywhere*. Cost: 1–3% accuracy and a mandatory calibration step (activations must be statically scaled). Choose INT8 when fleet heterogeneity matters more than squeezing the last accuracy point on your newest cards. This is standard practice, not a workaround — treat INT8-as-fallback as a fleet-heterogeneity *policy* you set once, not a one-off exception you reach for per model.

### AWQ / GPTQ 4-bit: a *fitting* tool, not a throughput tool

Reach for 4-bit weight-only when the problem is **"this model does not fit"** or **"I need bigger batches on a small card,"** not "I want more tok/s on an H100." Because activations stay FP16, W4A16 does not use the low-precision tensor cores for the matmul — the win is a 4× smaller weight footprint and less weight-load bandwidth, which helps *latency at low batch* and *VRAM headroom*, not compute-bound throughput at high batch. This is exactly why the study puts W4A16 in the **synchronous** column.

- **GPTQ**: layer-wise error-minimizing quantization (second-order, OBQ-style). Solid, widely supported.
- **AWQ** (activation-aware weight quantization — Lin, Tang, Tang, Yang, Chen, Wang, Xiao, Dang, Gan, Han; MLSys 2024 Best Paper, arXiv:2306.00978): protects the ~1% of weight channels that see the largest activations, using calibration data to find them. Usually edges out GPTQ on instruction/chat tasks.

Both are **calibration-driven** — which is the trap below.

### The calibration-set caveat (the part people get wrong)

Static activation quantization (INT8, static FP8) and AWQ/GPTQ all derive their scale factors — the numbers that map float ranges into the low-precision grid — from a small **calibration set** (typically 128–512 samples). Those scales are fit to the *distribution of that sample data*.

Feed a **generic** calibration set (C4/WikiText-style prose) to a model you serve for **code generation** and you mis-set the scales for code's very different token distribution: heavy indentation and bracket tokens, long rare-identifier tails, structured whitespace, and activation outliers that prose never produces. The scale factors then **clip or coarsely round exactly the activations code relies on**, and you lose several points of pass@1 on HumanEval while MMLU looks fine — so a prose-only eval *hides* the regression. Dynamic FP8 sidesteps this (scales computed per-inference, no calibration file), which is a further reason it is the low-risk default. **Rule: your calibration set must resemble your production traffic, and your eval must include a slice that resembles it too.**

This is not a "some models" edge case — it is first-order enough that the most sophisticated production answer (Character.AI, see Perspectives and Real-world use cases below) is to avoid the calibration step *entirely* by training natively in low precision, rather than trust a better calibration set.

### CPM: how the saving is actually computed

```
CPM ($/1M output tokens) = GPU_hourly_cost / (throughput_tok_per_s * 3600 / 1e6)
```

Same GPU, same price. If FP8 lifts sustained output throughput from 2,400 → 4,080 tok/s (1.7×), CPM drops by `1 − 1/1.7 = 41%`. That is the number for the deliverable — but it is only honest paired with the accuracy delta from the same run.

## Perspectives

**Inference-provider economics (Fireworks).** At a multi-tenant inference provider, a quantization decision is made *once* per model and amortized across every customer's traffic — the risk calculus is completely different from a single company quantizing its own model for its own product. Fireworks publishes a public methodology for evaluating quantization ("How Fireworks evaluates quantization precisely and interpretably") built on **divergence metrics**, not just task benchmarks — because a quantization error that flips a 0.51/0.49 token probability to 0.49/0.51 is invisible to an accuracy-based eval but real, and task benchmarks can even show *noise-driven improvement* from quantization that has nothing to do with quality. When you're serving one company's traffic, "close enough on MMLU" might be adequate; when you're serving thousands of customers' unknown workloads, you need a harder, distribution-level bar.

**Architecture-level (Character.AI).** The calibration-set trap assumes quantization happens *after* training, as a separate step with its own risk. Character.AI's answer is structural: they **train natively in int8** — weights, activations, and KV cache — specifically to "eliminate the risk of training/serving mismatch," rather than try to find a better calibration set for a post-hoc quantization step. This is the sophisticated end of the spectrum, well beyond what this module's deliverable expects (you are not retraining a model), but it's the kind of answer that separates a Staff-level response from a Senior one in an interview: "how would you eliminate calibration risk entirely?" has an answer beyond "use a better calibration set."

**Benchmark-vs-measured (NVIDIA vs. the fleet).** NVIDIA's own TensorRT-LLM materials quote FP8 in **peak-FLOP, datasheet terms** — a real, defensible number for the silicon. Fireworks' production numbers, and this lesson's own worked example, land at **~1.5–1.8× measured** sustained throughput. Neither number is wrong; they answer different questions. The practicing habit this builds: when someone hands you a vendor performance claim, ask "measured under what batch size, sequence length, and phase mix?" before you put it in a capacity plan.

**Academic rigor (Neural Magic/IST Austria).** The >500,000-eval scale behind the "Give Me BF16 or Give Me Death?" numbers is itself a methodological statement: a single benchmark run that "looked fine" is a different class of evidence from a systematic sweep across model family, size, and task. The deliverable's requirement to pair a CPM number with a *measured* accuracy delta on a knowledge slice and a code slice is this same discipline, scaled down to what one engineer can run on one rented GPU in an afternoon.

## Real-world use cases

- **Fireworks AI — quantization methodology** (dated snapshot, 2024–2025): [fireworks.ai/blog/fireworks-quantization](https://fireworks.ai/blog/fireworks-quantization). Shows the divergence-metrics critique of task-benchmark-only evaluation — a real inference provider explaining why "MMLU didn't move" is not sufficient evidence a quantized model is safe to ship.
- **Character.AI — "Optimizing AI Inference at Character.AI"**: [blog.character.ai/optimizing-ai-inference-at-character-ai-2](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/). Native int8 training (not post-training quantization) across weights, activations, and KV cache, explicitly to remove train/serve mismatch risk — reported as part of a >30× serving-cost reduction since late 2022 (dated figure, verify current before quoting externally). The advanced, architecture-level version of this lesson's calibration-set lesson.
- **NVIDIA Developer Blog — TensorRT-LLM on H100 FP8**: [developer.nvidia.com/blog/nvidia-tensorrt-llm-supercharges-large-language-model-inference-on-nvidia-h100-gpus](https://developer.nvidia.com/blog/nvidia-tensorrt-llm-supercharges-large-language-model-inference-on-nvidia-h100-gpus/). NVIDIA's own FP8/H100 throughput and memory numbers — useful specifically as the *datasheet* number to contrast against your own measured `bench serve` result.
- **Fireworks AI — FireAttention V4 (FP4, B200)**: [fireworks.ai/blog/fireattention-v4-fp4-b200](https://fireworks.ai/blog/fireattention-v4-fp4-b200) (dated snapshot, 2025). A one-line forward pointer to the frontier beyond this lesson's scope: NVFP4 on Blackwell as "what comes after FP8," the same way FP8 followed FP16 on Hopper.

## Worked example

Goal: produce, for the deliverable, an **honest FP8-vs-FP16 CPM saving + a measured accuracy delta** on one rented H100.

**1. Serve both variants.** FP16 baseline and dynamic FP8 (no calibration file needed):

```bash
# baseline
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8000 --max-num-seqs 256

# FP8: weights + activations dynamic, FP8 KV cache
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8001 --quantization fp8 --kv-cache-dtype fp8 --max-num-seqs 256
```

**2. Re-run the lesson-05 throughput benchmark against each** (identical dataset, concurrency, input/output lengths):

```bash
for PORT in 8000 8001; do
  vllm bench serve \
    --backend vllm --model meta-llama/Llama-3.1-8B-Instruct \
    --host localhost --port $PORT \
    --dataset-name sharegpt --num-prompts 1000 \
    --save-result --result-filename bench_$PORT.json
done
# read output_throughput (tok/s), TTFT p99, TPOT p99 from each JSON
```

Illustrative result (record *your* real numbers):

| Variant | Output tok/s | TTFT p99 | GPU $/hr | **CPM** |
|---|---|---|---|---|
| FP16 | 2,400 | 310 ms | $2.99 | **$0.346** |
| FP8 | 4,080 | 240 ms | $2.99 | **$0.204** |

Throughput multiplier **1.70×**, CPM cut **41%**. (`$2.99 / (4080*3600/1e6) = $0.204`.)

**3. Measure the accuracy delta on a small, relevant slice** — one general task and one code task, because the calibration caveat only shows up on the latter:

```bash
# knowledge slice
lm_eval --model vllm \
  --model_args pretrained=meta-llama/Llama-3.1-8B-Instruct,quantization=fp8,kv_cache_dtype=fp8 \
  --tasks mmlu --num_fewshot 5 --limit 500

# code slice (needs code execution enabled)
lm_eval --model vllm \
  --model_args pretrained=meta-llama/Llama-3.1-8B-Instruct,quantization=fp8 \
  --tasks humaneval --limit 100 --confirm_run_unsafe_code
# repeat both WITHOUT quantization=fp8 for the FP16 baseline
```

**4. Report the pair.** e.g. MMLU 68.1 → 68.3 (+0.2pt), HumanEval pass@1 63.4 → 63.0 (−0.4pt), within noise at `--limit`. Now the CPM claim is defensible: *"41% cheaper, no measurable accuracy loss on knowledge or code."*

## Practice

Feeds the **Cost-per-million-tokens** deliverable.

1. On a rented H100 (or H200), re-run the **lesson-05 benchmark** in FP8 vs FP16. Record the **throughput multiplier** (expect ~1.5–1.8×) and the **CPM reduction** using the formula above with your real GPU hourly rate.
2. Run a **small eval subset** — a `--limit 500` slice of MMLU *and* a `--limit 100` slice of HumanEval — at FP8 vs FP16, and record the **accuracy delta** for each.
3. (Stretch) Quantize once with a *generic* AWQ calibration set and once with a code-heavy one; show HumanEval moving while MMLU stays flat — proof of the calibration caveat.

**Acceptance:** the deliverable at `../practice/cost-per-token/README.md` contains the **FP8-vs-FP16 CPM saving** *and* a **measured accuracy delta** on a knowledge slice and a code slice — so the saving is honest, not asserted. A CPM number with no paired accuracy number does not pass.

## Common pitfalls

- **Quoting the "2×" datasheet number as the expected result.** NVIDIA's own peak-FLOP framing is real but is not what you'll measure. Report the ~1.5–1.8× *sustained* number from your own `bench serve` run, and say why it's lower (decode is bandwidth-bound; the full 2× shows up on prefill and on the larger batches FP8's memory savings unlock).
- **Treating the calibration-set risk as a "some models" problem.** Character.AI's decision to avoid post-training calibration entirely — by training natively in int8 — is evidence this is a first-order risk for production serving, not an edge case you can usually ignore.
- **Assuming accuracy loss is a flat, universal constant.** "1–3%" is a fair summary, but the actual >500k-eval numbers (99.75% recovery for 8-bit, 99.36% for W4A16) vary by model family, scale, and task — quote the precise, citable figures in a design doc rather than a rounded folk number.
- **Treating quantization as a one-time model decision instead of a fleet policy.** INT8-as-cross-platform-fallback is standard practice precisely because fleets are heterogeneous (Hopper+ alongside older Ampere/Turing cards) — decide the policy once, not per deployment.
- **Evaluating only on general benchmarks (MMLU) and missing domain-specific regressions.** This is the lesson's core pedagogical point: a prose-heavy calibration set and a prose-heavy eval can both look fine while a code (or tool-calling, or any structured-output) workload silently regresses. Always add a task slice that resembles your actual production traffic.

## Self-check

- **(a) What throughput multiplier should you expect from FP8 on an H100, and roughly what CPM reduction?**
  **Answer:** A *measured* ~**1.5–1.8×** sustained throughput (not the 2× peak-FLOP datasheet figure — decode is bandwidth-bound; you capture the full 2× only on prefill and on the larger batches FP8's halved memory unlocks). At constant GPU price, a 1.7× throughput gain is a CPM cut of `1 − 1/1.7 ≈ 41%`, i.e. roughly **40–50%**.

- **(b) Why can a generic calibration set hurt a code-generation workload specifically?**
  **Answer:** Static activation quant (and AWQ/GPTQ) fit their scale factors to the calibration data's distribution. Code's token/activation distribution — heavy indentation and bracket tokens, structured whitespace, long rare-identifier tails, larger activation outliers — differs sharply from generic prose. Scales fit on prose then clip/coarsely round exactly the activations code depends on, dropping HumanEval pass@1 while MMLU (prose-like) looks unchanged — so a prose-only eval hides it. Fix: calibrate on production-like data, and eval a code slice. Dynamic FP8 avoids the whole class (no calibration file); Character.AI's native int8 training avoids it at an even more fundamental level.

- **(c) When do you reach for AWQ/GPTQ 4-bit over FP8?**
  **Answer:** When the constraint is **fitting/VRAM**, not throughput: fit a larger model on a smaller card (e.g. a 70B on one H100), or free VRAM for bigger batches / longer context. W4A16 is weight-only, so it does not use low-precision tensor cores for the matmul — it wins on memory footprint and low-batch latency (synchronous/low-QPS), not compute-bound throughput. For high-QPS async continuous batching on Hopper+, FP8 (W8A8) wins.

- **(d) What is the precise, citable accuracy-recovery number for well-tuned 8-bit quantization, and why does the precision of that number matter?**
  **Answer:** ~**99.75%** of the BF16 baseline on average, across the >500,000-eval Neural Magic/IST Austria study (W4A16-INT is ~99.36%). It matters because "1–3% degradation" is directionally right but not a number you can defend in a design review the way "99.75% recovery, measured across >500k evals on the Llama-3.1 family" is — precision and evidence scale are themselves part of the engineering case for shipping the format.

- **(e) A multi-tenant inference provider and a single company serving its own model face different quantization risk. Why, and what does the provider do differently?**
  **Answer:** The provider's decision is made once and amortized across every customer's unknown, heterogeneous traffic, so a quantization error invisible on one benchmark could still hurt some customer's workload; a single company quantizing its own model only has to validate against its own traffic shape. Fireworks' response is methodological — divergence metrics rather than task-benchmark pass/fail, because task benchmarks can hide (or even show noise-driven improvement from) small distributional shifts that a probability-divergence metric catches.

## Connections & what's next

Quantization is the second explicit cost lever in this module (after 05's batching curve) and it compounds with what comes next: a quantized checkpoint is smaller, so it loads faster from cold storage (09) and it frees VRAM that autoscaling's `--max-num-seqs` headroom depends on (08). The calibration-set discipline here — measure on production-like data, don't trust a generic benchmark — is the same discipline the deliverable expects everywhere: a claimed saving is only real once it's measured on your workload, not asserted from a datasheet.

## References & further reading

**Primary sources**
1. **BentoML — "LLM Quantization" handbook** — the deepest practitioner walkthrough of FP8/INT8/AWQ/GPTQ tradeoffs and when to use each: https://bentoml.com/llm/getting-started/llm-quantization
2. **"Give Me BF16 or Give Me Death? Accuracy-Performance Trade-Offs in LLM Quantization"** (Kurtic, Marques, Pandit, Kurtz, Alistarh — Neural Magic/IST Austria, ACL 2025) — the >500k-eval study behind the FP8-lossless (~99.75% recovery) / W4A16 (~99.36% recovery) numbers and the sync-vs-async deployment rule: https://arxiv.org/abs/2411.02355
3. **NVIDIA TensorRT-LLM — Quantization docs** — the current FP8/FP4/INT4-AWQ/INT8-SmoothQuant hardware support matrix: https://nvidia.github.io/TensorRT-LLM/latest/features/quantization.html

**Real-world engineering blogs**
4. **Fireworks AI — "How Fireworks evaluates quantization precisely and interpretably"** — divergence-metrics methodology at a multi-tenant inference provider (dated snapshot): https://fireworks.ai/blog/fireworks-quantization
5. **Character.AI — "Optimizing AI Inference at Character.AI"** — native int8 training across weights/activations/KV cache to eliminate train/serve mismatch (dated cost figures, verify before external use): https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/
6. **NVIDIA Developer Blog — "NVIDIA TensorRT-LLM Supercharges Large Language Model Inference on NVIDIA H100 GPUs"** — the datasheet/peak-FLOP FP8 numbers to contrast against a measured result: https://developer.nvidia.com/blog/nvidia-tensorrt-llm-supercharges-large-language-model-inference-on-nvidia-h100-gpus/

**Deeper dives**
7. **AWQ — "Activation-aware Weight Quantization for LLM Compression and Acceleration"** (Lin et al., MLSys 2024 Best Paper) — the mechanism behind the ~1% salient-channel protection: https://arxiv.org/abs/2306.00978
8. **Fireworks AI — "FireAttention V4: Industry-Leading Latency and Cost Efficiency with FP4"** — the frontier beyond this lesson: NVFP4 on Blackwell/B200 (dated snapshot, 2025): https://fireworks.ai/blog/fireattention-v4-fp4-b200
