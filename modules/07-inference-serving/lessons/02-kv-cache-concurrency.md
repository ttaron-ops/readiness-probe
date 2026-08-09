---
lesson: "07.2"
title: "KV cache as a concurrency problem: capacity, fragmentation, and the cap"
module: "07"
concept: "KV cache as a concurrency problem: capacity, fragmentation, and the cap"
status: not-started
est_time: "5h"
artifacts: []
---

# 07.2 · KV cache as a concurrency problem: capacity, fragmentation, and the cap

> **Concept.** Your maximum concurrent requests is `free VRAM after weights ÷ per-request KV bytes` — and naive contiguous KV allocation throws away 60–80% of that budget to fragmentation, which is exactly the waste PagedAttention exists to reclaim.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Why this matters

The concurrency cap is the number that sets your \$/Mtok, and it is a memory-management fact, not a scheduler setting. Interviewers at serving-heavy shops probe this precisely: "you have an H100, an 8B model, and an 8k context — how many requests run at once, and where does the memory go?" If you can compute the cap from KV geometry and then explain why a contiguous allocator would have wasted most of it, you have demonstrated the single insight that motivates PagedAttention, continuous batching, and every KV-offload trick that follows. Getting fragmentation wrong is how teams conclude they "need more GPUs" when they actually needed a better allocator.

## What's new here

Module 03 introduced the KV cache *as a concept* — the per-token K and V tensors you keep so you don't recompute attention over the whole history each decode step. This lesson reframes it as a **memory-management and concurrency problem**: KV is a pool of a fixed byte size, requests are variable-length allocations against that pool, and the interesting failure modes are *capacity* (how many fit) and *fragmentation* (how much you waste fitting them). Module 05's `/metrics` reading is the instrument; here you use it to *observe the cap move* as you change context length. Nothing about prefill/decode physics is re-derived — we take "decode is memory-bound, so batch size matters" as given and ask the next question: what sets the ceiling on batch size?

## Core notes

**Per-token KV bytes.** For one token, one sequence, the KV cache stores a key and a value vector in every layer, for every KV head:

```
kv_bytes_per_token = 2 × num_layers × num_kv_heads × head_dim × dtype_bytes
                     ↑
                     K and V
```

The `2` is K+V. `num_kv_heads` (not `num_attention_heads`) is what matters — with **Grouped-Query Attention (GQA)**, many query heads share a few KV heads, and modern models exploit this to shrink KV by 4–8×. This is a serving-economics decision baked into the model architecture: GQA is largely *why* long-context serving is affordable.

Worked geometry for **Llama-3.1-70B** (BF16 KV): `num_layers = 80`, `num_kv_heads = 8` (GQA; 64 query heads), `head_dim = 128`, `dtype_bytes = 2`.

```
kv_bytes_per_token = 2 × 80 × 8 × 128 × 2 = 327,680 B ≈ 320 KiB/token
```

So a single 8k-token request holds `320 KiB × 8192 ≈ 2.5 GiB` of KV. Counterfactual: without GQA (MHA, 64 KV heads) it would be `2 × 80 × 64 × 128 × 2 ≈ 2.5 MiB/token` — **8× worse**, ~20 GiB for one 8k request. GQA is doing enormous economic work.

For **Llama-3.1-8B** (the model you'll actually rent for practice): `num_layers = 32`, `num_kv_heads = 8`, `head_dim = 128`, BF16 → `2 × 32 × 8 × 128 × 2 = 131,072 B = 128 KiB/token`.

**Reference geometry** (BF16 KV; pull the exact values from each model's `config.json` — these drift):

| Model | layers | kv_heads | head_dim | KV B/token | KV per 8k req |
|---|---|---|---|---|---|
| Llama-3.1-8B | 32 | 8 | 128 | 128 KiB | ~1.0 GiB |
| Llama-3.1-70B | 80 | 8 | 128 | 320 KiB | ~2.5 GiB |
| Qwen2.5-72B | 80 | 8 | 128 | 320 KiB | ~2.5 GiB |
| Mistral-7B (GQA) | 32 | 8 | 128 | 128 KiB | ~1.0 GiB |
| *Hypothetical 70B MHA (no GQA)* | 80 | 64 | 128 | 2.5 MiB | ~20 GiB |

The MHA row is the counterfactual that shows GQA carrying the economics: same size class, 8× the KV, 8× fewer concurrent requests, ~8× the \$/Mtok on long context. When you evaluate a new model for serving, `num_kv_heads` is a first-class cost input, not a modeling detail.

**KV blocks, not KV bytes.** vLLM never allocates KV per-token; it allocates **blocks** of a fixed number of tokens (default `--block-size 16` on GPU). The KV pool is `num_gpu_blocks` slots, each holding `block_size × kv_bytes_per_token`. vLLM prints `# GPU blocks:` at startup — that number *is* your KV budget in the unit the scheduler actually reasons about, and `kv_cache_usage_perc` is just `blocks_in_use / num_gpu_blocks`. Converting: `num_gpu_blocks × block_size` = total tokens of KV you can hold across all sequences at once. Everything below is that identity viewed as a cap.

**The concurrency cap.** Free VRAM for KV divided by per-request KV footprint:

```
max_concurrent_requests ≈  KV_pool_bytes  ÷  (kv_bytes_per_token × avg_seq_len)

KV_pool_bytes ≈ VRAM_total × gpu_memory_utilization − weights − overhead
```

Two things to internalize:

- **The cap is inversely proportional to context length.** Double `avg_seq_len` (or double `--max-model-len`, which is the *worst-case* per-request reservation), and the number of requests that fit **halves**. Context length and concurrency trade off against each other on a fixed KV pool — this is the lever behind almost every capacity decision.
- **`--max-model-len` is a reservation ceiling, not the live footprint.** vLLM sizes and admits work against the maximum a request *could* grow to; with paged allocation the live blocks track actual length, but the admission control and profiling are bounded by `--max-model-len`. Set it to your real p99 context, not the model's theoretical max, or you strand KV pool on headroom you never use.

**Why naive contiguous KV wastes 60–80%.** The pre-PagedAttention approach reserved one **contiguous** buffer per sequence, sized to the maximum possible length, at admission time. Three compounding wastes (this is the core argument of the PagedAttention paper, §2):

- **Internal fragmentation (the big one):** you reserve `max_model_len` per request but most requests decode far fewer tokens. A request that stops at 300 tokens inside a 2048-token reservation wastes 85% of its buffer for its entire lifetime.
- **Reservation-for-unknown-length:** generation length is unknown up front, so a safe contiguous allocator *must* over-reserve — you cannot grow a contiguous buffer in place if the neighbor is occupied.
- **External fragmentation:** variable-sized contiguous buffers leave unusable gaps between them as requests arrive and finish, like classic malloc heap fragmentation.

The paper measured that only **20–40%** of KV memory held actual token state under this scheme — i.e. **60–80% wasted**. PagedAttention (next lesson, 07.3) fixes this by allocating KV in fixed-size **blocks** (paging, like virtual memory), so the only waste is at most one partially-filled block per sequence (a few %), and blocks can be shared (prefix caching) or freed independently. That reclaimed 60–80% is directly *more concurrency on the same GPU* → directly lower \$/Mtok, which is why this is the pivotal serving optimization and not a footnote.

**Two more levers the block model unlocks (preview, both from paging).** Because KV lives in shareable, independently-freeable blocks:

- **Prefix caching** (`--enable-prefix-caching`, on by default in recent V1): identical prompt prefixes (system prompts, few-shot exemplars, shared RAG context) map to the *same* physical blocks across requests. A 2k-token shared system prompt costs KV *once* instead of once-per-request — for high-fan-out workloads this can multiply effective concurrency several-fold and is often the single biggest \$/Mtok win available. Observe hits via `vllm:prefix_cache_hits` / `queries` (07.3).
- **KV-cache quantization** (`--kv-cache-dtype fp8`): store K/V at FP8 instead of BF16, halving `kv_bytes_per_token` → ~2× the token capacity in the same pool, at a small accuracy cost you validate. This is a *direct* multiplier on the concurrency cap and a common lever for long-context serving.

Both are things you tune against the cap you're about to measure — hold them as "the pool can be stretched," and quantify in later lessons.

**Continuous batching is what turns the cap into throughput.** The cap says how many sequences *can* be resident; **continuous (in-flight) batching** — vLLM's default scheduler — is what keeps the running batch full: as soon as any sequence emits its EOS and frees its blocks, a waiting request is admitted mid-step instead of waiting for the whole batch to drain (07.3). Without it you'd pay for the cap and never fill it, because short requests would idle blocks waiting on long ones. The cap is the ceiling; continuous batching is how you live at the ceiling.

**Instrumentation (vLLM V1, pin v0.11.0).** The `/metrics` endpoint (module 05) exposes the gauges that make the cap observable:

- `vllm:kv_cache_usage_perc` — fraction of KV blocks in use, 0–1 (this is the V1 name; older docs/builds call it `vllm:gpu_cache_usage_perc`). When it pins near 1.0 you are KV-bound — the allocator, not the GPU's FLOPs, is your ceiling.
- `vllm:num_requests_running` — sequences currently in the running batch = your live concurrency.
- `vllm:num_requests_waiting` — the queue behind the KV wall; rising while `running` is flat is the signature of a KV-capacity cap (contrast queue-depth SLO reasoning from 05).

**Reading the cap in practice — the three-signal test.** You are KV-bound (not compute-bound, not client-bound) when *all three* hold simultaneously under load:

1. `kv_cache_usage_perc` pinned ≈ 1.0 (blocks exhausted),
2. `num_requests_running` flat at its ceiling,
3. `num_requests_waiting` climbing (arrivals can't be admitted).

That triple is your measured concurrency cap and the exact condition your \$/Mtok is computed at — it's the most efficient point *if* TPOT still meets SLO. If instead `kv_cache_usage_perc` is low while `waiting` climbs, you are *not* KV-bound (check `--max-num-seqs`, client concurrency, or a compute ceiling). Distinguishing these is a common senior-screen question: "the queue is growing — is it a memory cap or a throughput cap?" The KV gauge answers it.

## Worked example

**Concurrency cap for Llama-3.1-8B on one H100-80GB at 8k context.**

```
weights (BF16)        = 8e9 × 2 B            ≈ 16 GB
usable VRAM           = 80 GB × 0.9          = 72 GB
overhead              ≈ 3 GB
KV_pool               = 72 − 16 − 3          ≈ 53 GB
kv_bytes_per_token    = 2×32×8×128×2         = 128 KiB
per-request KV @ 8192 = 128 KiB × 8192       ≈ 1.0 GiB
--------------------------------------------------------
max_concurrent (worst-case, full 8k)  ≈ 53 GiB / 1.0 GiB ≈ ~53 requests
```

Now **halve the context to 4k**: per-request KV → ~0.5 GiB, cap → **~106 requests**. Halving `--max-model-len` (roughly) doubled concurrency on the *same* GPU — no new hardware, pure KV arithmetic. And because decode throughput scales with batch size (03: amortizing the weight read), doubling the runnable batch roughly doubles tokens/sec/GPU and halves \$/Mtok — until TPOT inflates past your SLO. That trade is what you measure next.

*(Real vLLM caps run a bit lower than this napkin figure: block granularity, CUDA-graph capture, and per-request scheduler state consume a little more. The point is the ~2× *movement* with context length, which you will confirm on hardware.)*

## Practice

Hands-on, rented GPU, feeds the deliverable. One H100-80GB (or A100-80GB) from any neocloud; ~30 min of GPU time. **Pin vLLM 0.11.0** (V1 engine only).

```bash
pip install "vllm==0.11.0"

# Run A: short context
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 4096 --gpu-memory-utilization 0.9 --port 8000 &

# Saturate it, then read the KV + concurrency gauges
# (use vllm's bench, or fire ~200 concurrent completions of ~2k output)
curl -s http://localhost:8000/metrics | \
  grep -E 'vllm:(kv_cache_usage_perc|gpu_cache_usage_perc|num_requests_running|num_requests_waiting)'
```

Kill it, then **Run B** identically but `--max-model-len 16384`. Saturate and scrape again.

Record, for each run, `num_requests_running` at the moment `kv_cache_usage_perc` (or `gpu_cache_usage_perc`) pins near 1.0 — that peak `running` count is your measured concurrency cap. You should see the 4× larger `--max-model-len` cut the cap by roughly 3–4× (not exactly 4× — block rounding and overhead).

**Acceptance:** a two-row measured table — `max-model-len` → peak `num_requests_running` at KV-saturation — plus the computed cap from the worked-example formula alongside each, committed to the [cost-per-token deliverable](../practice/cost-per-token/README.md). This "concurrency-cap vs max-model-len" observation is the load-bearing input for the \$/Mtok calc: tokens/sec/GPU ∝ this cap.

## Self-check

**(a) Compute per-token KV bytes for a 70B model; state your assumptions.**

**Answer:** Assume Llama-3.1-70B geometry — `num_layers = 80`, `num_kv_heads = 8` (GQA), `head_dim = 128` — and BF16 KV (`dtype_bytes = 2`). `kv_bytes_per_token = 2 × 80 × 8 × 128 × 2 = 327,680 B ≈ 320 KiB/token`. The `2` is K+V; using `num_kv_heads = 8` (not the 64 query heads) is the GQA saving — MHA would be 8× larger (~2.5 MiB/token).

**(b) If you double `--max-model-len` at fixed `--gpu-memory-utilization`, what happens to max concurrent requests, and why?**

**Answer:** It roughly **halves**. The KV pool (`VRAM × util − weights − overhead`) is unchanged, but each request now reserves/can-grow-to twice the KV footprint, so `cap = KV_pool ÷ (kv_bytes_per_token × max_seq_len)` scales as `1/max_model_len`. Concurrency and context length trade off on a fixed pool — which is why you set `--max-model-len` to your real p99 context, not the model maximum.

**(c) Why does contiguous KV allocation waste most of the KV memory?**

**Answer:** A contiguous allocator reserves one max-length buffer per sequence at admission because generation length is unknown and a contiguous buffer can't grow in place. Most requests finish far short of that reservation → **internal fragmentation** holds the unused tail for the request's whole life; variable-sized buffers also leave unusable gaps → **external fragmentation**. The PagedAttention paper measured only ~20–40% of KV holding real token state (60–80% wasted). Fixed-size paged blocks cut this to at most one partial block per sequence (a few %), turning the reclaimed memory directly into concurrency.

## Resources

1. **PagedAttention (Efficient Memory Management for LLM Serving), SOSP'23 — §2–3** — https://arxiv.org/abs/2309.06180 — *deep.* The primary source for the fragmentation argument and the block-based fix; §2 quantifies the 60–80% waste, §3 defines the paged KV mechanism 07.3 builds on. Read the memory-waste figure closely.
2. **vLLM — Conserving Memory** — https://docs.vllm.ai/en/latest/configuration/conserving_memory/ — *skim, then apply.* Practical `--gpu-memory-utilization`, `--max-model-len`, KV-dtype, and swap knobs that change the concurrency cap on real hardware — your reference during the practice runs.
3. **vLLM — Metrics design** — https://docs.vllm.ai/en/stable/design/metrics/ — *skim.* Authoritative names/semantics for `kv_cache_usage_perc`, `num_requests_running`, `num_requests_waiting`; confirm the exact strings against your pinned 0.11.0 `/metrics` output before wiring dashboards.
