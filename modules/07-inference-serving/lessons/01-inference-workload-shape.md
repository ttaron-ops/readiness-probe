---
lesson: "07.1"
title: "Inference workload shape: from roofline to the KV-cache budget"
module: "07"
concept: "Inference workload shape: from roofline to the KV-cache budget"
status: not-started
est_time: "6h"
prev: null
next: "02-kv-cache-concurrency.md"
artifacts: []
sources: 8
---

# 07.1 · Inference workload shape: from roofline to the KV-cache budget

> **Concept.** Prefill is compute-bound and decode is memory-bandwidth-bound, so the whole serving problem reduces to managing a VRAM budget — `weights + KV cache + activations` — to hit a latency SLO at minimum cost per token.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

This is the first lesson of module 07, and it exists to convert two things you already have — module 03's hardware physics and module 05's SLO vocabulary — into the one equation that governs everything downstream in this module: the VRAM budget. Nothing before this point told you *why* the KV cache is the object worth obsessing over; this lesson makes that case with arithmetic, not assertion. What it unlocks: 07.2 takes the residual term this lesson names ("whatever's left after weights and overhead") and turns it into a hard concurrency number, which 07.3 (PagedAttention) then shows you how to stop wasting 60–80% of.

## Why this matters

In a serving-system interview at a GPU-heavy shop, the fork in the road is whether you can connect hardware behavior to a dollar figure. "Decode is memory-bound" is a fact you already have from module 03; the senior-level move is to say *therefore* the KV cache is what caps concurrency, *therefore* batch size is bounded by free VRAM, *therefore* your cost per million tokens is set before you ever touch a scheduler flag. Getting the VRAM budget wrong is the single most common way teams either OOM in production or burn 3–5× on idle GPUs they rented to paper over the OOM.

This isn't hypothetical rigor for its own sake. CoreWeave's own public GPU-selection guidance for inference customers starts from exactly this budget — model size and context window (VRAM), concurrency and batching behavior (throughput), latency SLOs — before it ever gets to a spec sheet (see Real-world use cases below). If a fleet operator's customer-facing sizing guidance opens with this equation, it is a safe bet their internal interview loop does too. This lesson turns the physics you already know into the one equation that governs the rest of the module.

## What's new here (calibration)

- **Already yours (referenced, not re-taught):** prefill vs decode, compute- vs memory-bound, the roofline knee, KV cache *as a concept*, FP8/precision basics, the HBM-bandwidth tokens/sec ceiling on decode (module 03); TTFT/TPOT/queue-depth and reading vLLM's `/metrics` (module 05).
- **New here:** treating VRAM as a *fixed budget you allocate*, not a background fact — and recognizing the KV cache is not just "a concept" but the *residual* term that decides how many requests you can run at once.
- **New here:** the module's governing thesis — **you manage the KV-cache budget to a latency SLO at minimum cost per token** — stated as an equation you can compute against, not a slogan.
- **New here:** treating a modeling decision (GQA, covered in depth in 07.2) as a *cost lever a platform engineer should have an opinion on*, not something that happens upstream and is none of your business.

## Core concepts

**The three-term VRAM budget.** Every byte of a GPU's HBM at serving time is doing one of three jobs:

```
VRAM_total  =  model_weights  +  KV_cache  +  activations/overhead
```

- **`model_weights`** — fixed the instant you pick a model and a dtype. `bytes ≈ num_params × bytes_per_param`. FP16/BF16 = 2 B/param, FP8 = 1 B/param, INT4 ≈ 0.5 B/param. A 70B model is ~140 GB at FP16, ~70 GB at FP8, ~35 GB at INT4.
- **`activations/overhead`** — CUDA context, the framework, NCCL buffers, temporary activation tensors during a forward pass, CUDA graphs. Rule of thumb: reserve **2–5 GB** on top of weights before you count KV. vLLM does not let you fill 100% of HBM; `--gpu-memory-utilization` (default **0.9**) caps the fraction vLLM will touch, leaving headroom for exactly this.
- **`KV_cache`** — **the residual.** Whatever is left after weights and overhead is carved into KV blocks. This is the only term that scales with *concurrency × context length*, which is why it, and not raw FLOPs, is your concurrency governor.

**Why the workload shape forces this framing.** Recall from 03 (do not re-derive): prefill ingests the whole prompt in one shot — many tokens, one matmul-heavy pass, arithmetic intensity high, **compute-bound**. Decode emits one token per step and must re-read *all* the weights from HBM for each step, so it is **memory-bandwidth-bound**. The consequence for serving:

- **Prefill barely benefits from batching.** It is already saturating the tensor cores with a single sequence's worth of tokens; stacking more sequences mostly just queues more compute. You batch prefill to fill bubbles, not to escape a bottleneck.
- **Decode *lives or dies* on batching.** A single decode step reads ~140 GB of weights (for a 70B) to produce *one* token for *one* request — catastrophic bandwidth efficiency. Run 64 requests through that same weight read and you amortize the HBM traffic across 64 tokens. Decode throughput is roughly *"how many sequences can I keep resident and step together."*
- **What lets you keep sequences resident? KV-cache capacity.** Each in-flight sequence owns a slice of KV proportional to its current length. So the batch size that makes decode efficient is *capped by free VRAM ÷ per-request KV*. The memory-bound nature of decode and the KV budget are the same constraint viewed from two sides.

The shape of the throughput curve follows directly: aggregate decode tokens/sec climbs almost linearly with batch size while HBM bandwidth is underused, then **flattens** as the batch saturates bandwidth (the roofline knee from 03). You want to run at or just below that knee — the largest batch that stays bandwidth-efficient without pushing per-token latency past SLO. The KV pool decides whether you can *reach* the knee at all: if the pool caps you at batch 8 but the knee is at batch 48, you are leaving ~6× throughput (and ~6× \$/Mtok) on the table purely to memory, not compute.

**Where the cost per token comes from.** A GPU-hour is a fixed rent (an H100 is roughly \$2–4/hr on-demand at neoclouds, as of 2025 — treat the exact number as a variable in your deliverable). Cost per million *output* tokens, for a replica of `N` GPUs:

```
$/Mtok = (N × GPU_$/hr) ÷ (tokens_per_sec_per_replica × 3600) × 1e6
```

The lever you actually control is **tokens/sec per replica**, and that is `throughput ≈ running_batch_size × per_sequence_decode_rate`. Per-sequence decode rate has a hard ceiling you already met in 03 — the HBM-bandwidth tokens/sec limit — so once a single sequence is bandwidth-bound, **the only remaining knob is batch size**, and batch size is capped by the KV residual. Bigger KV residual → bigger batch → more tokens/sec → lower \$/Mtok — right up until the batch inflates TPOT past your SLO. **That tension — KV budget vs latency SLO — is the module.**

Worked instance to anchor the number: 1×H100 at \$3/hr sustaining 2,500 output tok/s ⇒ `3 ÷ (2500 × 3600) × 1e6 ≈ \$0.33/Mtok`. Halve the achievable batch (e.g. you doubled context and halved concurrency) and, decode being batch-limited, throughput roughly halves ⇒ ~\$0.66/Mtok on the *same rented hardware*. The GPU bill didn't move; your KV allocation decision did. This is the through-line into the `gpu-cost-operator` work in module 11.

**Prefill and decode want different things — and you serve them together.** Prefill is a latency spike (it sets TTFT, module 05) and is compute-heavy; decode is a sustained bandwidth drip that sets TPOT. On a shared replica these two phases contend: a long prompt's prefill can stall the decode batch (a "TTFT vs TPOT" fight you'll see directly in the metrics). Modern vLLM interleaves them with **chunked prefill** (07.3) so a big prompt is sliced and fed alongside ongoing decodes instead of monopolizing a step. You don't need the mechanism yet — just hold the shape: **one replica, two workloads with opposite bottlenecks, both drawing on the same KV pool.**

**When weights don't even fit.** If `model_weights > VRAM_total`, KV is negative and there is no serving to discuss. Your two escape hatches, both from 03:

- **Tensor parallelism (TP):** shard weights (and KV) across N GPUs. Weights-per-GPU drop ~1/N, freeing KV room, at the cost of per-step NCCL all-reduces (adds latency, needs NVLink to stay cheap). Set with `--tensor-parallel-size N`.
- **Quantization:** FP8 halves weight bytes vs FP16 (see 03 on FP8 numerics); INT4 quarters them. Frees KV room on the *same* GPU, but you validate quality separately.

You will almost always reach for one of these for 70B-class models on a single 80 GB card — the worked example shows why the arithmetic leaves you no choice.

**Sizing the overhead term honestly.** "2–5 GB" is not hand-waving; it decomposes into things you can point at:

| Overhead component | Rough size | Notes |
|---|---|---|
| CUDA context + driver | ~0.5–1 GB | Per process, before any model. |
| Framework + NCCL buffers | ~1–2 GB | Grows with TP degree (more comms buffers). |
| Peak activation tensors | 0.5–2 GB | Scales with batch × hidden × chunk size during a forward pass. |
| CUDA graphs (FULL_AND_PIECEWISE default in 0.11) | ~0.5–1 GB | Captured graphs trade memory for lower per-step launch latency. |

vLLM measures real peak usage during a profiling run at startup and sizes the KV pool from what's left under `--gpu-memory-utilization`. If you see "not enough KV cache" at boot, the fix is almost always lower `--max-model-len` or higher `--gpu-memory-utilization`, not a bigger GPU — you're being told the residual math failed, and 07.2 makes that residual the object of study.

**Model architecture is a cost lever too (preview).** The VRAM budget above treats `model_weights` and per-token KV bytes as external facts about a model you were handed. They aren't — they're design decisions made by a modeling team, and the biggest one is how many KV heads the attention layers use. Grouped-Query Attention (GQA) lets many query heads share a small number of KV heads, which shrinks the KV-cache term by 4–8× versus classic multi-head attention (MHA) at negligible quality cost — see the GQA paper in References. A senior platform engineer evaluating a model for production should be able to say "this uses `num_kv_heads = 8`, not 64, and that's why it's affordable to serve at long context" — and should be willing to push back on a model team proposing MHA for a high-QPS target, because that choice is paid for in your GPU bill, not theirs. The full numeric treatment (`kv_bytes_per_token`, the GQA-vs-MHA counterfactual) is 07.2's job; hold the shape here: **architecture choices made before deployment set the ceiling your serving-layer tricks operate under.**

## Perspectives

**The fleet-operator / SKU-selection view.** From CoreWeave's side of the table, this budget is the sizing conversation they have with every inference customer: model size and context window set VRAM, concurrency and batching set throughput, latency SLOs set the tail behavior they have to protect, and only then does GPU selection (H100 vs H200 vs B200, single-GPU vs multi-node) become a spec-sheet question. Getting the budget right *before* picking hardware is the difference between right-sizing a fleet and over-provisioning it "to be safe."

**The cost/FinOps view.** Character.AI's public 33× serving-cost reduction since late 2022 is not one trick — it's compounding wins across model architecture, caching, and infrastructure, with KV-cache-footprint reduction as a first-class lever alongside everything else. Contrast that with this lesson's single-GPU worked example: one config decision (TP degree vs FP8) can already move \$/Mtok by 2× on unchanged hardware. Multiply a handful of such decisions across a fleet and 33× stops looking exotic — it looks like disciplined budget management applied repeatedly.

**The benchmark-vs-production view.** Anyscale's widely-cited "23× throughput" result for continuous batching is a *throughput multiplier under benchmark conditions* — it says how much more work the same GPU can do once batching stops leaving idle bandwidth on the table. It is not automatically a 23× *cost* multiplier in production: real fleets run below peak batch to protect tail latency, deal with heterogeneous prompt lengths, and pay for headroom against traffic spikes. Module 05's utilization-vs-SLO reasoning is exactly what closes the gap between a benchmark number and a defensible \$/Mtok claim — hold that skepticism forward into 07.5's batching-economics lesson.

**The model-architecture-as-cost-lever view.** Most platform engineers treat the model as a black box handed down by a research team. The senior version of this job treats `num_kv_heads` as a serving-cost input you evaluate *before* a model goes into production, the same way you'd evaluate a database schema before it goes into a hot path. If two candidate models have comparable quality but one uses GQA with 8 KV heads and the other uses MHA with 64, that is an 8× difference in KV footprint per token — and therefore roughly an 8× difference in achievable concurrency at a given context length, before a single serving-layer optimization is applied.

## Real-world use cases

- **CoreWeave — "Choosing the Right NVIDIA GPU for Running Inference"** — https://www.coreweave.com/blog/choosing-the-right-nvidia-platform-for-running-inference-on-coreweave — walks the same weights + KV + overhead + concurrency budget as a GPU-selection decision, framed for customers sizing 70B-class deployments.
- **Character.AI — "Optimizing AI Inference at Character.AI"** — https://blog.character.ai/optimizing-ai-inference-at-character-ai/ — serving cost cut at least 33× since late 2022, serving ~20,000 QPS at under a cent per hour of conversation; shows why the VRAM/KV budget governs cost at real scale, not just on a napkin.
- **Anyscale — "Achieve 23x LLM Inference Throughput & Reduce p50 Latency"** — https://www.anyscale.com/blog/continuous-batching-llm-inference — the benchmark numbers behind the batch-1-vs-batched throughput gap this lesson's roofline argument predicts.

## Worked example

**Does Llama-3.1-70B at FP16 fit on one H100-80GB, and what's left for KV?**

Weights: `70e9 params × 2 B = 140 GB`.

```
VRAM_total (H100)      =  80 GB
model_weights (FP16)   = 140 GB
--------------------------------
residual for KV+overhead = 80 - 140 = -60 GB   ← negative
```

It does **not** fit — you are 60 GB underwater before a single KV byte or activation. This is not a tuning problem; it is arithmetic. It forces exactly one of:

- **TP=2 across 2×H100-80GB:** weights become ~70 GB/GPU. With `--gpu-memory-utilization 0.9` → ~72 GB usable/GPU, minus ~70 GB weights minus ~3 GB overhead ≈ **a razor-thin ~2–5 GB/GPU for KV** — technically serving but almost no concurrency headroom. TP=4 (4×H100) is the practical choice: ~35 GB weights/GPU leaves ~30 GB/GPU for KV.
- **FP8 on 1×H100-80GB:** weights ~70 GB. Usable ~72 GB − 70 GB − ~3 GB overhead ≈ **again only a couple GB for KV** — it "fits" but serves almost no concurrent requests, so most teams still pair FP8 with TP=2 to get real KV headroom.

Laid out as the decision you'd actually defend:

| Config | GPUs | Weights/GPU | ~Usable/GPU (0.9) | ~KV residual/GPU | Verdict |
|---|---|---|---|---|---|
| 70B FP16, 1×H100 | 1 | 140 GB | 72 GB | **−71 GB** | Impossible |
| 70B FP16, TP=2 | 2 | 70 GB | 72 GB | ~−1 to 2 GB | Fits but near-zero concurrency |
| 70B FP16, TP=4 | 4 | 35 GB | 72 GB | **~34 GB** | Comfortable KV headroom |
| 70B FP8, 1×H100 | 1 | 70 GB | 72 GB | ~−1 to 2 GB | Bare fit, tiny KV |
| 70B FP8, TP=2 | 2 | 35 GB | 72 GB | **~34 GB** | Comfortable, half the GPUs of FP16 TP=4 |

The takeaway to say out loud in an interview: **"140 > 80, so a single H100 is off the table; the real decision is TP degree vs FP8, and I pick it by how much KV residual each option leaves for my target concurrency."** Note the last two rows deliver the *same* KV headroom on 2 GPUs (FP8) vs 4 (FP16) — that halving is a direct \$/Mtok win if FP8 quality holds. You compute the concurrency each residual buys explicitly in [07.2](02-kv-cache-concurrency.md).

## Practice

Reading + napkin calculation. No GPU required for this lesson (you rent one starting in 07.2), so this is deliberately cheap.

1. **Re-derive the budget** for three configs and record residual-KV-per-GPU in a table your [cost-per-token deliverable](../practice/cost-per-token/README.md) will reuse:
   - 70B FP16 on 1×H100-80GB (show it's negative),
   - 70B FP16 on 4×H100-80GB with `--gpu-memory-utilization 0.9`,
   - 70B FP8 on 1×H100-80GB.
   Use overhead = 3 GB/GPU and `bytes_per_param` = 2 (FP16) / 1 (FP8).
2. **Predict the cheaper config** on \$/Mtok grounds alone (more usable KV → bigger batch → more tokens/sec/GPU, but divided across more GPUs). Write one sentence on which you'd rent and why. You will *measure* against this prediction in 07.2–07.4.
3. **Name one model-architecture lever** (from the Perspectives section) you'd ask about before signing off on a model for production, and what a bad answer would cost you in KV headroom.

**Acceptance:** a 3-row VRAM-budget table (weights / overhead / residual KV per GPU) plus a one-sentence \$/Mtok prediction and one architecture-lever note, committed to the deliverable's working notes. This is the input row your later measurements validate.

## Common pitfalls

- **Treating `--gpu-memory-utilization` as "how full I want the GPU."** It's a profiling *ceiling*, not a target — vLLM profiles real peak usage at startup and sizes the KV pool from what's under that ceiling. Setting it too low starves KV for no reason; setting it to 1.0 leaves no room for activation spikes and risks an OOM mid-request.
- **Assuming a bigger GPU fixes a negative KV residual.** `140 GB weights > 80 GB VRAM` is arithmetic, not a config problem. No flag makes that number positive — you need TP, quantization, or both.
- **Ignoring overhead as rounding error.** NCCL and framework buffers grow with TP degree; a 4-GPU TP setup's "2–5 GB" overhead assumption can be optimistic if you don't check it. Always validate against vLLM's actual startup profiling log, not the rule of thumb.
- **Believing prefill batching is "free" throughput the way decode batching is.** Prefill is already compute-bound on a single sequence; stacking sequences queues compute rather than unlocking idle bandwidth the way decode batching does. Don't expect the same near-linear throughput scaling from batching prefill that you get from batching decode.

## Self-check

**(a) Why does batching help decode a lot but barely help prefill?**

**Answer:** Decode is memory-bandwidth-bound (03): each step re-reads *all* model weights from HBM to emit one token for one sequence, so a batch of 1 wastes almost all the bandwidth. Batching N sequences amortizes that single weight read across N tokens produced in the same step, driving decode toward the roofline. Prefill is already compute-bound — one sequence's prompt saturates the tensor cores — so adding sequences mostly queues more compute rather than unlocking idle bandwidth; you batch prefill only to fill scheduling bubbles.

**(b) Write the `VRAM = weights + KV + activations` budget for 70B-FP16 on 1×H100-80GB. Does it fit, and what does it force?**

**Answer:** `weights = 70e9 × 2 B = 140 GB`; `VRAM_total = 80 GB`; residual for KV + activations = `80 − 140 = −60 GB`. It does **not** fit — negative before any KV or overhead. It forces either tensor parallelism (e.g. TP=2 → ~70 GB weights/GPU, TP=4 → ~35 GB/GPU, freeing real KV room) or lower-precision weights (FP8 → ~70 GB, INT4 → ~35 GB). For meaningful concurrency you typically combine them (FP8 + TP=2), because a bare fit leaves almost nothing for the KV residual.

**(c) Why is TTFT dominated by prefill and TPOT by decode — and which one does a bigger batch hurt?**

**Answer:** TTFT (time-to-first-token) is the latency to process the whole prompt through prefill before the first decode step emits a token, so it scales with prompt length and prefill compute. TPOT (time-per-output-token) is the steady-state decode-step time. A bigger batch *helps* decode throughput (amortizes the HBM weight read) but *hurts* TTFT: incoming requests wait for a batch slot and share prefill compute, so tail TTFT rises with batch size and queue depth. That's the core serving tension — batch for decode throughput, cap batch/admission to protect TTFT SLOs (05).

**(d) A model team proposes shipping a 70B model with classic multi-head attention (64 KV heads) instead of GQA (8 KV heads) for a high-QPS chat product. What's your pushback, in terms of this lesson's budget?**

**Answer:** KV bytes per token scale with `num_kv_heads`, so MHA's 64 KV heads vs GQA's 8 is an 8× larger KV footprint per token at the same context length — an 8× smaller KV residual buys 8× fewer concurrent requests, which is roughly an 8× hit to tokens/sec/GPU and therefore \$/Mtok, before any serving-layer optimization runs. For a high-QPS target this is not a rounding error; it's a modeling decision with serving-economics consequences baked in before deployment, and it's exactly the kind of thing a platform engineer should be in the room to flag.

## Connections & what's next

This lesson is the spine module 07 hangs everything else on: the VRAM budget it derives is the input to 07.2's concurrency-cap math, the thing PagedAttention (07.3) reclaims waste from, and the fixed cost base that 07.5's batching-economics curves and 07.7's quantization lesson both move by changing one term of the equation (KV residual, weight bytes, respectively). The GQA-as-cost-lever thread opened here in Perspectives is picked up numerically next.

**Next: [07.2 — KV cache as a concurrency problem](02-kv-cache-concurrency.md)** takes the KV residual this lesson names as "whatever's left" and turns it into a hard number — `max_concurrent_requests` — then shows why a naive contiguous allocator would throw away 60–80% of it, setting up PagedAttention's fix in 07.3.

## References & further reading

**Primary sources**
- vLLM — Conserving Memory — https://docs.vllm.ai/en/latest/configuration/conserving_memory/ — read for the authoritative `--gpu-memory-utilization` / `--max-model-len` semantics this lesson's budget depends on.
- PagedAttention paper (arXiv 2309.06180) — https://arxiv.org/abs/2309.06180 — read §1–2 for the framing of KV cache as the serving-cost bottleneck; the fragmentation numbers are 07.2's territory.
- GQA paper, "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints" (arXiv 2305.13245) — https://arxiv.org/abs/2305.13245 — read for why `num_kv_heads`, not `num_attention_heads`, is the cost-relevant number; the architectural-choice angle this lesson previews.

**Real-world engineering blogs**
- CoreWeave — "Choosing the Right NVIDIA GPU for Running Inference" — https://www.coreweave.com/blog/choosing-the-right-nvidia-platform-for-running-inference-on-coreweave — the same weights+KV+overhead budget as a GPU-selection decision for 70B-class models.
- Character.AI — "Optimizing AI Inference at Character.AI" — https://blog.character.ai/optimizing-ai-inference-at-character-ai/ — serving cost cut 33× since late 2022 (as of the 2024 post); ties into why the VRAM/KV budget governs cost at scale.
- Anyscale — "Achieve 23x LLM Inference Throughput & Reduce p50 Latency" — https://www.anyscale.com/blog/continuous-batching-llm-inference — benchmark numbers for the batch-1-vs-batched throughput gap.

**Deeper dives**
- Pierre Lienhart — "LLM Inference Series: 5. Dissecting model performance" — https://medium.com/@plienhar/llm-inference-series-5-dissecting-model-performance-6144aa93168f — a solid roofline treatment covering arithmetic intensity and the compute/bandwidth split this lesson leans on.
- "Inference Basics: KV Cache, Batching, Parallelism" — https://s09g.medium.com/inference-basics-kv-cache-batching-parallelism-0c04378d4067 — consolidates the exact bridge this lesson makes (why decode needs batching, how KV and parallelism interact) in one readable pass; use it to sanity-check your mental model, not for numbers.
