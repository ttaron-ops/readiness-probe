---
lesson: "07.1"
title: "Inference workload shape: from roofline to the KV-cache budget"
module: "07"
concept: "Inference workload shape: from roofline to the KV-cache budget"
status: not-started
est_time: "5h"
artifacts: []
---

# 07.1 · Inference workload shape: from roofline to the KV-cache budget

> **Concept.** Prefill is compute-bound and decode is memory-bandwidth-bound, so the whole serving problem reduces to managing a VRAM budget — `weights + KV cache + activations` — to hit a latency SLO at minimum cost per token.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Why this matters

In a serving-system interview at a GPU-heavy shop, the fork in the road is whether you can connect hardware behavior to a dollar figure. "Decode is memory-bound" is a fact you already have from module 03; the senior-level move is to say *therefore* the KV cache is what caps concurrency, *therefore* batch size is bounded by free VRAM, *therefore* your cost per million tokens is set before you ever touch a scheduler flag. Getting the VRAM budget wrong is the single most common way teams either OOM in production or burn 3–5× on idle GPUs they rented to paper over the OOM. This lesson turns the physics you already know into the one equation that governs the rest of the module.

## What's new here

Module 03 gave you the **hardware physics**: prefill vs decode, compute- vs memory-bound, the roofline, KV cache *as a concept*, FP8/precision, and the tokens/sec ceiling that HBM bandwidth imposes on decode. Module 05 gave you the **SLO vocabulary**: TTFT, TPOT/ITL, queue depth, why p99 *request* latency is meaningless for a streaming response, and how to read vLLM's `/metrics`. This lesson does not re-derive any of that — it references it.

What is new is the **serving-system layer that sits on top**: treating VRAM as a fixed budget you allocate, and recognizing that the KV cache is not just "a concept" but the *residual* term that decides how many requests you can run at once. The thesis for the entire module: **you manage the KV-cache budget to a latency SLO at minimum cost per token.** Everything downstream — continuous batching, PagedAttention, chunked prefill, tensor parallelism, quantization — is a tactic for making that residual bigger or using it more efficiently.

## Core notes

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

**Where the cost per token comes from.** A GPU-hour is a fixed rent (an H100 is roughly \$2–4/hr on-demand at neoclouds; treat the exact number as a variable in your deliverable). Cost per million *output* tokens, for a replica of `N` GPUs:

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

**Acceptance:** a 3-row VRAM-budget table (weights / overhead / residual KV per GPU) plus a one-sentence \$/Mtok prediction, committed to the deliverable's working notes. This is the input row your later measurements validate.

## Self-check

**(a) Why does batching help decode a lot but barely help prefill?**

**Answer:** Decode is memory-bandwidth-bound (03): each step re-reads *all* model weights from HBM to emit one token for one sequence, so a batch of 1 wastes almost all the bandwidth. Batching N sequences amortizes that single weight read across N tokens produced in the same step, driving decode toward the roofline. Prefill is already compute-bound — one sequence's prompt saturates the tensor cores — so adding sequences mostly queues more compute rather than unlocking idle bandwidth; you batch prefill only to fill scheduling bubbles.

**(b) Write the `VRAM = weights + KV + activations` budget for 70B-FP16 on 1×H100-80GB. Does it fit, and what does it force?**

**Answer:** `weights = 70e9 × 2 B = 140 GB`; `VRAM_total = 80 GB`; residual for KV + activations = `80 − 140 = −60 GB`. It does **not** fit — negative before any KV or overhead. It forces either tensor parallelism (e.g. TP=2 → ~70 GB weights/GPU, TP=4 → ~35 GB/GPU, freeing real KV room) or lower-precision weights (FP8 → ~70 GB, INT4 → ~35 GB). For meaningful concurrency you typically combine them (FP8 + TP=2), because a bare fit leaves almost nothing for the KV residual.

**(c) Why is TTFT dominated by prefill and TPOT by decode — and which one does a bigger batch hurt?**

**Answer:** TTFT (time-to-first-token) is the latency to process the whole prompt through prefill before the first decode step emits a token, so it scales with prompt length and prefill compute. TPOT (time-per-output-token) is the steady-state decode-step time. A bigger batch *helps* decode throughput (amortizes the HBM weight read) but *hurts* TTFT: incoming requests wait for a batch slot and share prefill compute, so tail TTFT rises with batch size and queue depth. That's the core serving tension — batch for decode throughput, cap batch/admission to protect TTFT SLOs (05).

## Resources

1. **Inference Basics: KV Cache, Batching, Parallelism** — https://s09g.medium.com/inference-basics-kv-cache-batching-parallelism-0c04378d4067 — *skim.* Consolidates the exact bridge this lesson makes (why decode needs batching, how KV and parallelism interact) in one readable pass; use it to sanity-check your mental model, not for numbers.
2. **Module 03 — GPU hardware** (`../../03-gpu-hardware/README.md`) — *reference.* Your source of truth for prefill/decode, roofline, memory-bound decode, and FP8 numerics — re-read the roofline section before doing the practice math.
3. **Module 05 — GPU observability** (`../../05-gpu-observability/README.md`) — *reference.* TTFT/TPOT/queue-depth definitions and `/metrics` reading that 07.2 builds directly on when you measure the concurrency cap.
