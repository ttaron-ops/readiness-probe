---
lesson: "03.3"
title: "Memory hierarchy & HBM: what fits and how fast"
module: "03"
concept: "Memory hierarchy & HBM"
status: not-started
est_time: "6h"
prev: "02-compute-vs-memory-bound-roofline.md"
next: "04-decode-bandwidth-batching.md"
artifacts: []
sources: 10
---

# 03.3 · Memory hierarchy & HBM: what fits and how fast

> **Concept.** HBM capacity decides what *fits*; HBM bandwidth decides how *fast* you decode. Two independent buying levers, one spec sheet.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 03.2 gave you the roofline: every workload is compute-bound or memory-bound, decided by arithmetic intensity (AI) versus the GPU's ridge point (~295 FLOP/byte dense BF16 on H100). It told you LLM decode lands "deep memory-bound" — AI ≈ 1 — without fully explaining *why* the memory side of that equation behaves the way it does, or what the actual numbers on a real spec sheet mean for a real model. This lesson fills that gap: it opens up HBM itself — why it's stacked, what capacity and bandwidth actually buy you separately, and how to turn "does this model fit, and how fast will it run" into arithmetic you can do on a napkin before you rent anything. Lesson 03.4 then takes the bandwidth number built here and derives the exact tokens/sec ceiling for autoregressive decode — this lesson is the foundation that derivation stands on.

## Why this matters

Two questions dominate every GPU capacity-planning conversation a platform engineer has: **"can we serve model X on SKU Y?"** (a capacity question) and **"how many tokens/sec will we get?"** (a bandwidth question). Get the first wrong and you discover it in production as an out-of-memory crash mid-deployment. Get the second wrong and you either over-provision GPUs you didn't need or under-provision and miss an SLO — both are line items on a bill someone reviews. NVIDIA's own Solutions Architect postings (per this module's README) explicitly test for reasoning about arithmetic intensity versus peak bandwidth to "classify bottlenecks" — this lesson is where that classification becomes a number instead of an intuition.

It's also the concrete reason GPU SKUs that share a compute die still cost meaningfully different amounts. An H100 (80 GB HBM3, 3.35 TB/s) and an H200 (141 GB HBM3e, 4.8 TB/s) run identical FLOPS — same GH100 die, same tensor cores. If GPUs were priced on compute alone, they would cost the same. They don't, because for inference — the workload most GPU-fleet dollars go toward — the two numbers this lesson teaches you to compute (capacity headroom, bandwidth ceiling) are the entire value proposition of the more expensive card. A platform engineer who can't derive that gets outbid on reasoning by whoever can.

## What's new here (calibration)

Per the module README's calibration, you're not becoming a kernel developer — you already know what a GPU is and roughly how memory hierarchies work in general-purpose computing. This lesson does **not** re-teach cache theory, register allocation, or how a compiler schedules loads. What it adds:

- **The two independent buying levers named precisely** — HBM *capacity* (what fits) and HBM *bandwidth* (how fast) move separately across generations and SKUs, and conflating them is the single most common GPU-purchasing mistake.
- **A budget formula you can run on any model/SKU pair in under a minute** — weights + KV cache + overhead versus capacity, with the KV-cache term broken down to its true drivers (layers, heads, context, batch, precision).
- **The GQA/MQA and quantization levers that compress KV cache, with primary-source citations** — not just "compression exists" but which architectural choice buys which multiple, and evidence of how far real production systems push it.
- **The physical "why"** — HBM's stacked-die packaging is *why* capacity and bandwidth are independent knobs a vendor can turn separately, which is the mental model behind every "why does the Nth-generation card cost more" question you'll field.

## Core concepts

### The hierarchy as a ladder

On a Hopper GPU, memory tiers run from fastest/smallest to slowest/largest:

| Tier | Scope | Approx size | Approx bandwidth | Latency |
|------|-------|-------------|-------------------|---------|
| Registers | per-thread | ~256 KB/SM (register file) | tens of TB/s aggregate | ~1 cycle |
| Shared memory / L1 | per-SM (per thread-block) | up to 228 KB/SM (H100) | tens of TB/s aggregate | ~20–30 cycles |
| L2 cache | whole GPU | 50 MB (H100) | several TB/s | ~200 cycles |
| **HBM (global memory)** | whole GPU | **80 GB (H100) / 141 GB (H200)** | **3.35 / 4.8 TB/s** | ~400–600 cycles |

Two things to internalize:

1. **Each rung down is roughly an order of magnitude bigger and slower.** Kernel-writing is the art of keeping hot data on the top rungs. *Platform* engineering only needs to know that for LLM inference, the working set — tens of GB of weights, plus a growing KV cache — always spills to HBM. There is no realistic way to keep a 70B model's weights in a 50 MB L2 cache. HBM is where the real budget conversation happens.
2. **Only HBM is measured in "GB you can spend."** Registers, shared memory, and L2 are fixed, tiny, and not something you allocate model weights into. When someone says "the model doesn't fit," they mean HBM, full stop — this lesson only needs that one tier.

### HBM: what it physically is, and why that makes it a stackable lever

**HBM (High-Bandwidth Memory)** is DRAM dies stacked vertically and mounted on the same package as the GPU die — connected through a silicon interposer by an extremely wide bus (thousands of signal lines), rather than the narrow board-level traces that connect a CPU to its DIMMs. That physical arrangement is *why* HBM reaches TB/s-class bandwidth that off-package GDDR or DDR memory cannot approach, and it's also why HBM is expensive and capacity-limited: you can only stack so many DRAM dies per stack and place so many stacks around a given compute die's package footprint.

This packaging fact has a direct, verified consequence across GPU generations:

- **H100 SXM5:** 80 GB HBM3 across **5 stacks of 16 GB each**, 3.35 TB/s.
- **H200 SXM5:** 141 GB HBM3e across **6 stacks of 24 GB each**, 4.8 TB/s.

Same GH100 compute die in both cards — NVIDIA didn't touch the SMs or tensor cores between H100 and H200. What changed is the memory package: one more physical stack, and a denser/faster generation of HBM (HBM3 → HBM3e) per stack. That's the entire mechanism behind "same compute, more memory, more bandwidth, higher price." **Capacity and bandwidth are somewhat independent knobs a vendor turns by changing the memory package, not the compute die** — which is exactly why you cannot infer bandwidth from capacity, or vice versa, without checking the spec sheet.

### A generational memory snapshot (verify exact figures at build time)

| GPU | HBM generation | Capacity | Bandwidth | Dense BF16/FP16 TFLOPS | FP8 tensor-core HW? |
|---|---|---|---|---|---|
| A100 SXM | HBM2e | 80 GB | ~1.94–2.0 TB/s | 312 (624 sparse) | **No** — FP8 tensor cores debut in Hopper; A100 has accelerated INT8 but not FP8 |
| H100 SXM5 | HBM3 | 80 GB | 3.35 TB/s | 989 (1,979 sparse) | Yes — 4th-gen Transformer Engine |
| H200 SXM5 | HBM3e | 141 GB | 4.8 TB/s | 989 (1,979 sparse) — identical to H100 | Yes |

The A100 row is worth dwelling on: it is the same compute-tier card family lineage as H100/H200 but predates Hopper's FP8 tensor-core pipe entirely. On A100, "quantize to FP8" is not a hardware-accelerated path — you'd fall back to INT8 (which *is* tensor-core accelerated on Ampere) for a comparable capacity/throughput win. Lesson 03.5 goes deep on precision; the takeaway here is narrower and purely a capacity one: **which quantized formats even run at full speed is gated by which silicon generation you're renting**, not just a software flag.

### Capacity: what fits

HBM capacity is a hard wall — there is no "spill to slower memory" option for a GPU the way there is virtual memory on a CPU host (paging a model's weights to host RAM at inference time is possible but catastrophically slow; it's an emergency measure, not a plan). Your resident footprint for LLM inference is:

```
HBM used  ≈  weights  +  KV cache  +  activations/overhead
```

- **Weights** — fixed per model and precision. `bytes ≈ params × bytes/param`. FP16/BF16 = 2 bytes/param, so a 70B model at FP16 ≈ **140 GB**. INT8/FP8 halve it to ~70 GB (hardware-gated per the table above); INT4/FP4 (Blackwell-only, lesson 03.5) ~35 GB.
- **KV cache** — grows with concurrent sequences × their lengths. This is the *variable* term, and it's what caps how many requests you can batch (lesson 03.4).
- **Overhead** — CUDA context, activation buffers, memory fragmentation, framework reserve. Budget a few GB; serving frameworks like vLLM reserve a configurable fraction of free memory up front specifically so a later allocation doesn't blow past capacity mid-request.

If `HBM used > capacity`, you shard across GPUs (tensor parallelism), quantize, or move to a bigger-memory SKU. All three are cost decisions with different tradeoffs — sharding buys capacity at the cost of inter-GPU communication overhead per token; quantization buys capacity and often throughput at the cost of an accuracy delta you have to measure; a bigger SKU buys capacity at a higher hourly rate.

### KV cache: the sizing formula, precisely

During autoregressive generation, every output token's attention computation needs the Key and Value projections of every prior token, at every layer. Rather than recompute them from scratch each step, they're cached in HBM — the **KV cache**. Its size **per token**, for a model using plain multi-head attention (MHA) at FP16, is:

```
bytes/token ≈ 2 × num_layers × hidden_dim × 2 bytes
              │                              └ FP16 = 2 bytes/element
              └ one K projection + one V projection
```

Then per sequence multiply by `seq_len`, and across a batch multiply by concurrent sequence count:

```
KV bytes ≈ 2 × layers × hidden_dim × 2 × seq_len × batch
```

**The headline: KV cache scales linearly with (batch × context length).** Double the context window or double concurrency, and you double the KV footprint. On long-context, high-concurrency serving, KV cache can rival or exceed the weights — which is exactly why the H200's extra 61 GB over the H100 matters disproportionately for inference, not just for fitting bigger models.

**Real models compress this with GQA or MQA.** Grouped-Query Attention (GQA) and Multi-Query Attention (MQA) reduce the number of distinct K/V projections by sharing them across groups of query heads, which shrinks the KV cache by exactly the head-grouping ratio. The precise substitution: replace `hidden_dim` in the formula above with `num_kv_heads × head_dim`:

```
KV bytes ≈ 2 × layers × (num_kv_heads × head_dim) × bytes/elem × seq_len × batch
```

MQA (all query heads share a single K/V head) was introduced by Noam Shazeer, "Fast Transformer Decoding: One Write-Head is All You Need" (2019) — the original insight that decode's memory-bandwidth bottleneck (lesson 03.4) can be attacked at the *architecture* level, not just the serving-system level. GQA generalizes this to a tunable number of shared groups, formalized in Ainslie et al., ["GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints"](https://arxiv.org/abs/2305.13245) (EMNLP 2023) — this is the citable primary source behind the "GQA/MQA shrink KV by the head-grouping ratio" claim used throughout this module. The formula above without GQA/MQA compression is the conservative *upper bound*, good enough for a first-pass budget; the real figure for a modern model is usually 4–8× smaller.

### Compounding levers: GQA is the first lever, not the ceiling

GQA/MQA compress KV cache at the *architecture* level — a choice the model's training team made, which you as a platform engineer inherit and can only observe, not choose. But it is one of several **independently stackable** levers, and production systems compound them. Character.AI's engineering team reports combining native int8 quantization-aware training (QAT), MQA, and KV-cache sharing to cut their production KV-cache footprint **more than 20× without quality regression** — dramatically more than GQA's typical 4–8× alone, because they're stacking a precision lever (int8, ~2× on top of the architectural lever) with an additional cache-sharing technique on top of MQA itself. The lesson: GQA/MQA is where this module's KV-cache story starts, not where it ends — precision (lesson 03.5) and cache-sharing/paging strategies are separate, compounding multipliers on the same number.

### Bandwidth: how fast

HBM bandwidth is bytes/sec off the memory stacks, and it matters because **LLM decode is memory-bound**: to generate one token, single-stream, the GPU must stream essentially every weight out of HBM once. So the throughput ceiling is roughly:

```
tokens/sec  ≈  HBM_bandwidth  ÷  model_bytes
```

For a 70B FP16 model on an H100: 3.35 TB/s ÷ 140 GB ≈ **~24 tok/s**, regardless of how many FLOPS the die has. This is the single most important number-shape in GPU-inference economics — lesson 03.4 derives and stress-tests it in full, including how batching changes the picture. Here, hold the intuition: **compute (FLOPS) doesn't set decode speed — bandwidth does.**

### Why capacity and bandwidth are different buying levers

| Question | Governed by | Lever |
|---|---|---|
| Does the model + KV cache fit? | HBM **capacity** (GB) | bigger card / shard across GPUs / quantize |
| How fast does it decode? | HBM **bandwidth** (TB/s) | faster memory / smaller model / batch (03.4) |
| How much math per second? | **FLOPS** (compute die) | rarely the binding constraint for decode |

You can be capacity-bound (won't fit) with bandwidth to spare, or fit comfortably while bandwidth-starved. Diagnosing *which* wall you're hitting is the platform engineer's core skill here, because the fix — and its cost — differ completely.

## Perspectives

**Developer/ML view.** GQA versus full MHA, and how many KV heads to share, is an architectural choice made at *training* time — by the time you're serving the model, it's baked into the checkpoint. As a platform engineer you don't choose it, but you must read it out of the model config (`num_key_value_heads` in most HF configs) to get your KV-cache budget right; using the plain-MHA formula on a GQA model silently overestimates your footprint by 4–8×, which either wastes capacity you didn't need to reserve or, worse, causes you to under-provision batch headroom because you thought you had less room than you actually do.

**Operator/platform view.** The memory-budget reflex — weights + KV + overhead versus capacity, run in under a minute — is the single most common "will this fit" ticket a platform engineer answers, and it's the difference between a confident yes/no and a production OOM discovered the hard way. This is the calculation you should be able to do from memory, on a whiteboard, in an interview.

**Hardware view.** HBM stacking (5×16 GB HBM3 on H100 versus 6×24 GB HBM3e on H200) is a literal packaging decision — more physical DRAM dies, denser per-stack — not a magic bandwidth number pulled from a marketing slide. Grounding "why can't they just make HBM bigger or faster arbitrarily" in the physical stack-count/interposer picture is what separates "I memorized the spec sheet" from "I understand why the spec sheet says what it says."

**Economics view.** Character.AI's >20× KV-cache reduction is a direct $/token argument: every byte of KV cache freed is either more concurrent users served per GPU (batch headroom, lesson 03.4) or the ability to run a smaller, cheaper SKU for the same traffic. At production LLM-chat scale, a KV-cache compression project is not an academic optimization — it is a line item in a cloud bill, and it is why serving teams invest real engineering time in it.

## Real-world use cases

- **["vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention"](https://blog.vllm.ai/2023/06/20/vllm.html) — vLLM Project (UC Berkeley, June 2023).** The original production announcement of paged, non-contiguous KV-cache allocation (borrowing the OS-paging idea to eliminate memory fragmentation) — reports up to 24× higher throughput than HuggingFace Transformers by managing exactly the capacity budget this lesson teaches you to compute by hand. Shows what "fixing the KV-cache-fits problem at the systems level" looks like in a widely deployed real serving stack.
- **["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) — Character.AI Engineering.** Native int8 QAT + MQA + KV-cache sharing combine to cut KV-cache size more than 20× at real production LLM-chat scale, without a quality regression — the standout concrete production number for "GQA/MQA is one lever among several, and they compound."
- **["Optimizing AI Inference at Character.AI, Part Deux"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/) — Character.AI Engineering.** The follow-up post extending the same optimization program; together with Part 1 it's the clearest public case study of a team treating memory-hierarchy efficiency as a first-class, ongoing cost-reduction program rather than a one-time architecture choice.

## Worked example

**Memory budget: Llama-3-70B at FP16, on one H100 (80 GB) versus one H200 (141 GB).**

*Llama-3-70B config:* 80 layers, hidden_dim 8192, 64 query heads (head_dim 128), 8 KV heads (GQA).

**1. Weights**
```
70e9 params × 2 bytes = 140 GB
```

**2. KV cache, one sequence at 8k context — uncompressed upper bound (plain MHA formula):**
```
bytes/token = 2 × 80 × 8192 × 2 = 2,621,440 B ≈ 2.62 MB/token
8192 tokens × 2.62 MB ≈ 21.5 GB per sequence
```

**3. KV cache, real GQA figure.** Llama-3-70B actually uses 8 KV heads × 128 head_dim = 1024 effective KV width versus 8192 hidden_dim — an 8× reduction (per Ainslie et al.'s GQA formulation):
```
21.5 GB ÷ 8 ≈ 2.7 GB per sequence at 8k context
```

**4. Illustrative extension — compounding levers (not a Character.AI-specific number).** Character.AI's production stack combines int8 KV quantization (a further ~2× on top of FP16→int8) with additional cache-sharing techniques to reach >20× overall reduction versus an uncompressed baseline. Applying the *same order* of compounding to our 2.7 GB/seq GQA figure — an additional ~2× from int8 KV quantization, plus further reduction from cache-sharing across similar-context sequences — plausibly pushes toward roughly **1 GB/seq or lower** for a comparably-sized model. This is an order-of-magnitude illustration, not a claim about Llama-3-70B specifically (Character.AI's model and workload differ) — the point is that GQA's 8× is the *first* compounding lever, not the ceiling of what's achievable; lesson 03.5 picks up the precision lever in full.

**5. Overhead:** ~3–4 GB context and framework reserve.

**Verdict:**
```
H100 80GB:  140 GB weights ALONE > 80 GB  → DOES NOT FIT. Not even the weights.
            Requires ≥2× H100 (tensor-parallel shard) or INT8/FP8 quantization (~70 GB).
H200 141GB: 140 GB weights ≈ 141 GB capacity → weights *barely* fit,
            leaving ~1 GB — effectively NO room for KV cache. In practice you would
            still shard across 2× H200, or quantize, to leave room to serve.
```

A 70B FP16 model is a **≥2-GPU model** on Hopper regardless of SKU; the H200's capacity edge shows up more clearly for 30–40B-class FP16 models, or for giving a model that already fits much more KV-cache headroom (longer context, higher batch — lesson 03.4).

**6. Bandwidth sanity check (decode ceiling preview):**
```
H100:  3.35 TB/s ÷ 140 GB ≈ 23.9 tok/s (single stream, if it fit)
H200:  4.80 TB/s ÷ 140 GB ≈ 34.3 tok/s
```
Same compute die, ~43% faster decode — purely from bandwidth. That's the H200 inference value proposition in one line, and it's the number lesson 03.4 builds an entire cost model around.

## Practice

Rent one GPU (H100 80 GB if available; any single modern datacenter GPU works — adjust the spec numbers to your SKU). Target ~1 hour of GPU time. Results feed the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md).

1. **Load a model and inspect resident memory.** Pick a model that fits your SKU (e.g., an 8B or 13B model at FP16 on an 80 GB card). Load it in PyTorch/transformers and record `nvidia-smi` memory used before vs. after load, and `torch.cuda.memory_allocated()` after load. Confirm it matches `params × 2 bytes` within a few percent.
2. **Compute the weights footprint by hand** at FP16 and compare to your SKU's HBM budget. Note the headroom.
3. **Estimate KV-cache size per sequence** using the GQA-aware formula (`2 × layers × num_kv_heads × head_dim × 2 bytes × seq_len`) for two context lengths (e.g., 2k and 32k). Pull `num_key_value_heads` and `hidden_size` from your model's config — do not use the plain-MHA formula if the model uses GQA/MQA, since it will overstate your footprint. Show how KV scales.
4. **Run an HBM bandwidth microbenchmark.** Simplest reliable approach: a large `torch` elementwise copy/add on a multi-GB tensor, timed, computing `bytes_moved / seconds` (read + write counts as 2× the tensor size per element). Or use NVIDIA's `bandwidthTest` (CUDA samples) or a GEMM at large sizes. Record achieved GB/s.
5. **Compare achieved vs. spec** (3.35 TB/s for H100). Expect to reach ~70–85% of spec on a well-formed benchmark; note your efficiency and why real workloads see less.

**Acceptance:** a **memory-budget breakdown** for one model on one SKU — weights (measured + hand-computed), GQA-aware KV-cache estimate at one context length, overhead, total vs. HBM capacity, plus your measured-vs-spec HBM bandwidth (GB/s and % of spec). Drop it into the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md). One table plus three sentences of interpretation is enough.

## Common pitfalls

1. **Using the uncompressed (plain-MHA) KV formula as if it were the real footprint for a modern model.** Nearly every model shipped since 2023 uses GQA or MQA. Using `hidden_dim` instead of `num_kv_heads × head_dim` overstates your KV budget by 4–8×, which can make a servable model look like it needs sharding it doesn't need.
2. **Assuming HBM capacity and bandwidth move together generationally.** H100 → H200 is direct proof they don't: same compute die, both capacity *and* bandwidth changed, but compute FLOPS did not move at all. Never infer one from the other — check both numbers independently on the spec sheet.
3. **Forgetting activation/workspace overhead and framework reserve when computing "does it fit."** A model that mathematically fits weights + KV cache can still OOM in practice from CUDA context, activation buffers, and fragmentation — budget a few GB and treat serving-framework memory reserves (e.g., vLLM's `gpu_memory_utilization` fraction) as load-bearing, not cosmetic.
4. **Believing precision only affects weights.** The KV cache itself can be quantized (int8, or FP8 on Hopper+) as a lever *independent* of weight precision — Character.AI's stack quantizes the KV cache specifically, on top of the architectural MQA/GQA reduction, and that's where a large share of their >20× reduction comes from.

## Self-check

- Does a 70B model at FP16 fit on one H100 (80 GB)? On one H200 (141 GB)? Show the math. **Answer:** Weights = 70e9 × 2 bytes = 140 GB. H100 80GB: 140 GB > 80 GB → no, weights alone don't fit — you're ~60 GB over before any KV cache; needs ≥2× H100 or quantization to INT8/FP8 (~70 GB, which then fits with room for KV). H200 141GB: 140 GB < 141 GB → weights technically fit, but with only ~1 GB left there's no room for KV cache or overhead, so it's not servable in practice on a single card — realistically still a 2-GPU deployment, or quantize. A 70B FP16 model is a ≥2-GPU model on Hopper regardless of SKU.
- How does KV-cache size scale with batch × context length, and what changes if the model uses GQA? **Answer:** Linearly in both, independently: `KV bytes ≈ 2 × layers × (num_kv_heads × head_dim) × bytes/elem × seq_len × batch`. Doubling context length doubles KV; doubling concurrent sequences (batch) doubles KV — KV grows as the *product* of batch and context. GQA/MQA reduce the constant factor (by substituting `num_kv_heads × head_dim` for the full `hidden_dim`, typically a 4–8× reduction) but do not change the linear scaling law itself — long-context, high-concurrency serving is still KV-cache-bound, just at a smaller absolute footprint than the plain-MHA formula predicts.
- Why is the H200 (same compute die as the H100) often the better inference buy? **Answer:** Because inference — specifically decode — is memory-bound, not compute-bound (lesson 03.2). The H200 keeps the identical GH100 compute die but upgrades memory to 141 GB @ 4.8 TB/s (vs. 80 GB @ 3.35 TB/s). The extra *capacity* lets more of the model plus KV cache fit on fewer GPUs (fewer shards, more batch headroom); the extra *bandwidth* directly raises the decode throughput ceiling by ~43% (tok/s ≈ bandwidth ÷ model_bytes). You pay nothing for more FLOPS you couldn't use anyway — you pay for the two things that actually gate inference throughput and fit.
- Character.AI reports cutting KV-cache size more than 20× via combined techniques. Name the techniques and which lesson each belongs to. **Answer:** Native int8 quantization-aware training (precision — lesson 03.5), MQA (attention architecture / the KV-cache formula in this lesson), and additional KV-cache sharing across sequences (a systems-level dedup technique on top of both). The point is that this lesson's GQA/MQA lever is one of several *independently stackable* multipliers on KV-cache footprint — not the ceiling of what's achievable.

## Connections & what's next

This lesson turns lesson 03.2's abstract "decode is deep memory-bound" claim into concrete GB and TB/s numbers you can compute for any model/SKU pair. It sets up lesson 03.4 directly: the bandwidth ceiling derived here (`tok/s ≈ bandwidth ÷ model_bytes`) is exactly what 03.4 extends into a full batching and prefill/decode-disaggregation model. It also hands off to lesson 03.5 (precision) — every capacity number here assumes FP16; halving bytes/param via FP8 or INT8 is the next lever on top of GQA/MQA, and to lesson 03.6, where the H100-vs-H200 purchasing argument from this lesson's worked example becomes a full generational-SKU comparison. The economics thread — capacity and bandwidth as independent cost levers — is the same thread module 07 (inference serving) and module 11 (GPU cost economics) build production serving architecture and fleet cost models on top of.

Next: **[03.4 · Decode bandwidth ceilings, batching, and the prefill/decode split](04-decode-bandwidth-batching.md)** takes the bandwidth ceiling this lesson introduced and derives, in full, why batching is the only lever that raises aggregate decode throughput — and why KV-cache capacity, not compute, is almost always what stops you from batching further.

## References & further reading

**Primary sources**
- NVIDIA, [H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — authoritative source for HBM capacity/bandwidth, L2 size, and the on-die memory hierarchy on Hopper; read the memory-subsystem section.
- NVIDIA, [H200 Tensor Core GPU datasheet](https://www.nvidia.com/en-us/data-center/h200/) — canonical 141 GB / 4.8 TB/s source; confirms the HBM3e generational upgrade over H100.
- Ainslie et al., ["GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints"](https://arxiv.org/abs/2305.13245) (EMNLP 2023) — read for the formal derivation behind the "replace hidden_dim with num_kv_heads × head_dim" KV-cache substitution used above.
- Shazeer, "Fast Transformer Decoding: One Write-Head is All You Need" (2019) — the original MQA paper; read for the decode-bandwidth motivation behind sharing K/V projections across query heads. *(Canonical paper; a specific arXiv URL was not independently verified in this pass — cite by title/author/year.)*
- Kwon et al., ["Efficient Memory Management for Large Language Model Serving with PagedAttention"](https://arxiv.org/abs/2309.06180) (SOSP 2023) — the research paper behind the vLLM blog post below; read for the formal treatment of KV-cache fragmentation and paging.

**Real-world engineering blogs**
- vLLM Project, ["vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention"](https://blog.vllm.ai/2023/06/20/vllm.html) — production paged, non-contiguous KV-cache allocation; what it shows: fixing the capacity budget problem at the systems level, not just the architecture level.
- Character.AI Engineering, ["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/) — what it shows: >20× KV-cache reduction via combined int8 + MQA + sharing, at real production chat scale.
- Character.AI Engineering, ["Optimizing AI Inference at Character.AI, Part Deux"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/) — what it shows: KV-cache/memory efficiency treated as an ongoing engineering program, not a one-time fix.

**Deeper dives**
- Kirk & Hwu, *Programming Massively Parallel Processors*, Ch. 5 (Memory architecture) — skim for the register → shared → global mental model only; ignore the CUDA coding exercises, you want tier intuition, not tiling technique.
- [vLLM documentation](https://docs.vllm.ai/) — for the practice section and beyond: how PagedAttention and memory reservation work in a real serving stack you can run yourself.
