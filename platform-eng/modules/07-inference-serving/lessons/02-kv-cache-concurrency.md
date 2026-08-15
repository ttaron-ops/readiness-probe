---
lesson: "07.2"
title: "KV cache as a concurrency problem: capacity, fragmentation, and the cap"
module: "07"
concept: "KV cache as a concurrency problem: capacity, fragmentation, and the cap"
status: not-started
est_time: "6h"
prev: "01-inference-workload-shape.md"
next: "03-pagedattention-and-vllm.md"
artifacts: []
sources: 7
---

# 07.2 · KV cache as a concurrency problem: capacity, fragmentation, and the cap

> **Concept.** Your maximum concurrent requests is `free VRAM after weights ÷ per-request KV bytes` — and naive contiguous KV allocation throws away 60–80% of that budget to fragmentation, which is exactly the waste PagedAttention exists to reclaim.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

07.1 established that the KV cache is the *residual* term in the VRAM budget and that it, not raw FLOPs, governs how many requests you can run concurrently — but it left "the residual" as an abstract leftover. This lesson turns that residual into an exact number by working out per-token KV bytes from model geometry, and then exposes the uncomfortable fact that a naive allocator wastes most of that number to fragmentation before you ever get to use it. That waste — and the concurrency you're leaving on the table because of it — is precisely what 07.3's PagedAttention mechanism exists to reclaim, so this lesson's job is to make you feel the size of the problem before you're handed the solution.

## Why this matters

The concurrency cap is the number that sets your \$/Mtok, and it is a memory-management fact, not a scheduler setting. Interviewers at serving-heavy shops probe this precisely: "you have an H100, an 8B model, and an 8k context — how many requests run at once, and where does the memory go?" If you can compute the cap from KV geometry and then explain why a contiguous allocator would have wasted most of it, you have demonstrated the single insight that motivates PagedAttention, continuous batching, and every KV-offload trick that follows. Getting fragmentation wrong is how teams conclude they "need more GPUs" when they actually needed a better allocator.

Character.AI's public account of their serving stack makes this concrete at production scale: they describe an "efficient system for caching attention KV on host memory between chat turns" that directly "determines the maximum batch size that can fit on a GPU" — and they cut KV cache size by more than 20× through architecture choices (multi-query attention, hybrid attention horizons) without regressing quality. That is a team treating KV-cache-per-token as a first-class design target, not an afterthought tuned after the fact. This lesson gives you the arithmetic to reason about that the same way.

## What's new here (calibration)

- **Already yours (referenced, not re-taught):** the KV cache *as a concept* — the per-token K/V tensors you keep so you don't recompute attention over the whole history each decode step (module 03); reading TTFT/TPOT and vLLM's `/metrics` (module 05).
- **New here:** the KV cache reframed as a **memory-management and concurrency problem** — a fixed-byte pool, requests as variable-length allocations against it, and the two failure modes that matter: *capacity* (how many fit) and *fragmentation* (how much you waste fitting them).
- **New here:** the exact per-token KV-bytes formula, worked against real model geometry, and the concurrency-cap formula it feeds.
- **New here:** the three-signal observability runbook (`kv_cache_usage_perc` + `num_requests_running` + `num_requests_waiting`) for diagnosing *whether* you're KV-bound at all — module 05 gave you the instrument, this lesson gives you the diagnostic procedure.
- We take "decode is memory-bound, so batch size matters" as given (03) and ask the next question this module is built around: **what sets the ceiling on batch size?**

## Core concepts

**Per-token KV bytes.** For one token, one sequence, the KV cache stores a key and a value vector in every layer, for every KV head:

```
kv_bytes_per_token = 2 × num_layers × num_kv_heads × head_dim × dtype_bytes
                     ↑
                     K and V
```

The `2` is K+V. `num_kv_heads` (not `num_attention_heads`) is what matters — with **Grouped-Query Attention (GQA)**, many query heads share a few KV heads, and modern models exploit this to shrink KV by 4–8×. This is the serving-economics decision 07.1 flagged as an architecture-level cost lever — GQA is largely *why* long-context serving is affordable at all.

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

- **The cap is inversely proportional to context length.** Double `avg_seq_len` (or double `--max-model-len`, which is the *worst-case* per-request reservation), and the number of requests that fit **halves**. Context length and concurrency trade off against each other on a fixed KV pool — this is the lever behind almost every capacity decision, and it is the exact tension that agent workloads with 100K+ token contexts push to its limit (see Perspectives below).
- **`--max-model-len` is a reservation ceiling, not the live footprint.** vLLM sizes and admits work against the maximum a request *could* grow to; with paged allocation the live blocks track actual length, but the admission control and profiling are bounded by `--max-model-len`. Set it to your real p99 context, not the model's theoretical max, or you strand KV pool on headroom you never use.

**Why naive contiguous KV wastes 60–80%.** The pre-PagedAttention approach reserved one **contiguous** buffer per sequence, sized to the maximum possible length, at admission time. Three compounding wastes (this is the core argument of the PagedAttention paper, §2), each with a direct OS-memory-management analogue:

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

That triple is your measured concurrency cap and the exact condition your \$/Mtok is computed at — it's the most efficient point *if* TPOT still meets SLO. If instead `kv_cache_usage_perc` is low while `waiting` climbs, you are *not* KV-bound (check `--max-num-seqs`, client concurrency, or a compute ceiling). Distinguishing these is a common senior-screen question: "the queue is growing — is it a memory cap or a throughput cap?" The KV gauge answers it. Treat this three-signal test as a small incident-response runbook: it's the first thing you check when someone pages you about rising latency on an inference fleet, before you reach for "add more GPUs."

## Perspectives

**The architecture-as-cost-control view.** Character.AI's engineering blog is explicit that KV-cache-per-token is a design target they optimize *at the model architecture level* — multi-query attention variants and hybrid attention horizons that bound how far back a layer needs to attend — not just a serving-layer tuning knob. The most senior version of the skill this lesson teaches is not "I can compute the concurrency cap" but "I can influence the model architecture upstream so the cap is bigger to begin with." That's a materially different level of leverage than tuning `--gpu-memory-utilization` after the model is already fixed.

**The allocator/OS-analogy view.** Everything in the "why naive contiguous KV wastes 60–80%" section is textbook OS memory management with the serial numbers filed off: internal fragmentation, external fragmentation, and over-reservation for unknown future growth are the same three failure modes a first-year OS course covers for heap allocators and process address spaces. If you've debugged fragmented Java heaps or Kubernetes node bin-packing, you already have the right mental model — the only new fact is that here the "heap" is HBM and the "objects" are per-token K/V tensors.

**The observability/SRE view.** The three-signal test above is written as an incident-response runbook on purpose. In production, "requests are queuing" is an ambiguous alert — it could mean you're KV-bound, compute-bound, or hitting a client-side concurrency cap that has nothing to do with the GPU. Checking `kv_cache_usage_perc` first, before reaching for "scale out," is the difference between a two-minute diagnosis and an hour of adding GPUs that don't move the metric because the real constraint was a `--max-num-seqs` setting or a caller-side connection pool.

**The long-context economics view.** As agent workloads push context windows toward 100K+ tokens (long conversation histories, large retrieved documents, multi-step tool-call traces), the inverse relationship between context length and concurrency cap stops being a napkin-math curiosity and becomes the dominant cost driver: a pool that comfortably serves 100 concurrent 8k-token requests serves roughly 8 concurrent 100k-token requests on the same hardware. This is exactly the tension that later lessons' disaggregation and KV-offload techniques (07.6) exist to relieve — by moving KV off the GPU's scarce HBM onto cheaper tiers, or by scaling prefill and decode independently so a long-context prefill doesn't starve the decode pool.

## Real-world use cases

- **Character.AI — "Optimizing AI Inference at Character.AI"** — https://blog.character.ai/optimizing-ai-inference-at-character-ai/ — describes an "efficient system for caching attention KV on host memory between chat turns" that "determines the maximum batch size that can fit on a GPU," and cuts KV cache size by more than 20× via architecture (multi-query attention, hybrid attention horizons) without quality loss. The load-bearing case study for this lesson: a production team treating KV footprint as the primary lever on serving cost.
- **NetApp Community — "Engineering Inference: KV Cache, Shared Storage, and the Economics of AI"** — https://community.netapp.com/t5/Tech-ONTAP-Blogs/Engineering-Inference-KV-Cache-Shared-Storage-and-the-Economics-of-AI/ba-p/466018 — vendor content (flag it as such, not an independent operator postmortem), but a usable secondary reference for the "KV cache as an economic object, not just a data structure" framing, including why reuse vs. recompute and tiered storage for KV are becoming standard vocabulary in AI infrastructure.

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

While both runs are saturated, walk the three-signal test explicitly: confirm `kv_cache_usage_perc` is pinned, `num_requests_running` is flat, and `num_requests_waiting` is climbing — write down the values, since this triple is what you'll cite as "measured KV-bound" evidence in the deliverable, not just an inferred cap.

**Acceptance:** a two-row measured table — `max-model-len` → peak `num_requests_running` at KV-saturation — plus the computed cap from the worked-example formula alongside each, and the three raw gauge values confirming the KV-bound signature, committed to the [cost-per-token deliverable](../practice/cost-per-token/README.md). This "concurrency-cap vs max-model-len" observation is the load-bearing input for the \$/Mtok calc: tokens/sec/GPU ∝ this cap.

## Common pitfalls

- **Setting `--max-model-len` to the architectural max "to be safe."** This directly strands KV pool you'll never use — the reservation ceiling scales your worst-case per-request footprint, not your actual usage. Character.AI's counterexample is instructive: elite serving teams actively engineer KV footprint *down* (via architecture and via right-sized `--max-model-len`), they don't pad it up for safety margin.
- **Confusing `num_kv_heads` with `num_attention_heads`.** A common interview bug — using the query-head count instead of the KV-head count silently inflates your KV-bytes estimate by up to 8×, which cascades into a wrong concurrency cap and a wrong \$/Mtok. Always pull `num_key_value_heads` from the model's `config.json`, not the attention-head count from the architecture diagram.
- **Believing block-size 16 is a serving-quality knob to tune first.** It rarely matters compared to `--max-model-len` and `--gpu-memory-utilization`, which move the cap by orders of magnitude more. Leave `--block-size` at its default unless you have a specific long-context or prefix-sharing reason to change it (07.3 covers when it does matter).
- **Reading `num_requests_waiting` climbing as automatically "add GPUs."** Check `kv_cache_usage_perc` first. If it's low while the queue grows, you're not KV-bound — you may be hitting a compute ceiling, a `--max-num-seqs` cap, or a client-side concurrency limit, none of which more GPU memory or more replicas necessarily fixes without also checking what's actually saturated.

## Self-check

**(a) Compute per-token KV bytes for a 70B model; state your assumptions.**

**Answer:** Assume Llama-3.1-70B geometry — `num_layers = 80`, `num_kv_heads = 8` (GQA), `head_dim = 128` — and BF16 KV (`dtype_bytes = 2`). `kv_bytes_per_token = 2 × 80 × 8 × 128 × 2 = 327,680 B ≈ 320 KiB/token`. The `2` is K+V; using `num_kv_heads = 8` (not the 64 query heads) is the GQA saving — MHA would be 8× larger (~2.5 MiB/token).

**(b) If you double `--max-model-len` at fixed `--gpu-memory-utilization`, what happens to max concurrent requests, and why?**

**Answer:** It roughly **halves**. The KV pool (`VRAM × util − weights − overhead`) is unchanged, but each request now reserves/can-grow-to twice the KV footprint, so `cap = KV_pool ÷ (kv_bytes_per_token × max_seq_len)` scales as `1/max_model_len`. Concurrency and context length trade off on a fixed pool — which is why you set `--max-model-len` to your real p99 context, not the model maximum.

**(c) Why does contiguous KV allocation waste most of the KV memory?**

**Answer:** A contiguous allocator reserves one max-length buffer per sequence at admission because generation length is unknown and a contiguous buffer can't grow in place. Most requests finish far short of that reservation → **internal fragmentation** holds the unused tail for the request's whole life; variable-sized buffers also leave unusable gaps → **external fragmentation**. The PagedAttention paper measured only ~20–40% of KV holding real token state (60–80% wasted). Fixed-size paged blocks cut this to at most one partial block per sequence (a few %), turning the reclaimed memory directly into concurrency.

**(d) Your dashboard shows `num_requests_waiting` climbing steadily. How do you determine, without guessing, whether the fix is "add GPUs"?**

**Answer:** Run the three-signal test: check `kv_cache_usage_perc`, `num_requests_running`, and `num_requests_waiting` together. If `kv_cache_usage_perc` is pinned near 1.0 and `num_requests_running` is flat at its ceiling while `waiting` climbs, you are genuinely KV-bound — more KV pool (bigger GPU, more GPUs, quantized KV, or a shorter `--max-model-len`) is the right fix. If `kv_cache_usage_perc` is low while `waiting` still climbs, the bottleneck is elsewhere — a `--max-num-seqs` cap, a compute ceiling, or a client-side concurrency limit — and adding GPU memory won't move the metric.

**(e) A team proposes cutting \$/Mtok by moving to a model with 4× more KV heads because it scored slightly higher on a quality benchmark. What do you ask before signing off?**

**Answer:** Ask what the KV-heads change does to `kv_bytes_per_token` and, therefore, to the concurrency cap and \$/Mtok at your real production context length — a 4× increase in `num_kv_heads` is roughly a 4× cut in max concurrent requests on the same GPU, which likely dwarfs a marginal quality gain in cost terms. This is the GQA-vs-MHA counterfactual from this lesson applied as a go/no-go gate, not just a napkin exercise.

## Connections & what's next

This lesson turns 07.1's abstract "KV is the residual" into the concrete `max_concurrent_requests` number that every later lesson in this module measures, tunes, or multiplies. The fragmentation waste it quantifies (60–80%) is the exact gap 07.3's PagedAttention closes; the three-signal observability runbook is the diagnostic you'll reuse when 07.4 covers production tuning and preemption, and when 07.8 covers autoscaling signals. The long-context/concurrency tension raised in Perspectives resurfaces directly in 07.6's disaggregation lesson.

**Next: [07.3 — PagedAttention and vLLM](03-pagedattention-and-vllm.md)** takes the fragmentation number this lesson names (60–80% wasted) and shows the block-table mechanism — non-contiguous fixed-size KV blocks addressed like OS virtual memory — that cuts that waste to under 4%, which is what physically permits the high `max-num-seqs` this lesson's cap formula assumes is achievable.

## References & further reading

**Primary sources**
- PagedAttention paper, §2 ("Efficient Memory Management for LLM Serving with PagedAttention," SOSP'23) — https://arxiv.org/abs/2309.06180 — read for the fragmentation measurements (20–40% real utilization) that this lesson's core argument is built on.
- vLLM — Metrics design — https://docs.vllm.ai/en/stable/design/metrics/ — read for the authoritative names/semantics of `kv_cache_usage_perc`, `num_requests_running`, `num_requests_waiting` before wiring the practice's dashboard.
- vLLM — Conserving Memory — https://docs.vllm.ai/en/latest/configuration/conserving_memory/ — the practical `--gpu-memory-utilization`, `--max-model-len`, and KV-dtype knobs that change the concurrency cap on real hardware.

**Real-world engineering blogs**
- Character.AI — "Optimizing AI Inference at Character.AI" — https://blog.character.ai/optimizing-ai-inference-at-character-ai/ — KV cache as the batch-size-determining resource, cut >20× via architecture without quality loss; the load-bearing case study for this lesson.
- NetApp Community — "Engineering Inference: KV Cache, Shared Storage, and the Economics of AI" — https://community.netapp.com/t5/Tech-ONTAP-Blogs/Engineering-Inference-KV-Cache-Shared-Storage-and-the-Economics-of-AI/ba-p/466018 — vendor content on KV-cache-as-economic-object framing; read with that lens, not as an operator postmortem.

**Deeper dives**
- GQA paper, "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints" (arXiv 2305.13245) — https://arxiv.org/abs/2305.13245 — the full derivation behind the `num_kv_heads` savings this lesson's worked example uses; cross-ref 07.1 rather than re-deriving.
- Sebastian Raschka — "LLM Architecture Gallery" — https://sebastianraschka.com/llm-architecture-gallery/ — includes a compact GQA explainer alongside other attention-variant fact sheets, useful for quickly checking a new model's KV-head geometry.
