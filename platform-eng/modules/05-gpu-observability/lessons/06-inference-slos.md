---
lesson: "05.6"
title: "Inference SLOs: TTFT, TPOT, and why request-latency lies"
module: "05"
concept: "Inference SLOs: TTFT, TPOT, and why request-latency lies"
status: not-started
est_time: "7h"
prev: "05-health-and-xid.md"
next: "07-profiling-escalation.md"
artifacts: []
sources: 7
---
{% raw %}
# 05.6 · Inference SLOs: TTFT, TPOT, and why request-latency lies

> **Concept.** A streaming LLM endpoint has *two* latencies with different physics — time-to-first-token and per-token cadence — and putting your SLO on total request latency measures your users' verbosity, not your service's health.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

05.5 gave you the health gate: is this GPU even trustworthy to run on. This lesson assumes yes, and asks the next question — is the *service* running on it meeting the promise it made to a user? That's a different failure mode: a fleet of perfectly healthy GPUs can still deliver a bad experience if the SLO you're watching is the wrong metric. It's also the module's second act on the util-lie thesis: 05.1 taught you `GPU_UTIL` can read 100% while `SM_ACTIVE` is near zero on a decode workload; this lesson explains *why* that specific workload shape exists (memory-bandwidth-bound autoregressive decode) and gives you the SLOs that actually track user-visible health instead. What it unlocks: 05.7's profiling ladder needs a concrete "this metric looks wrong" trigger to escalate from, and the split TTFT/TPOT metrics here are exactly that trigger for inference workloads.

## Why this matters

You've written hundreds of `histogram_quantile(0.99, …)` request-latency SLOs for REST and gRPC services, and every one of them was correct: request latency is a clean health signal when the work per request is roughly fixed. For a streaming LLM it is *not* correct, and shipping the same SLO you'd write for an API gateway is the single most common way a GPU dashboard lies to you.

Here's the trap. A user asks for a 2000-token essay. The endpoint is perfectly healthy — sub-second to first token, smooth 30 tokens/sec after — and the request takes 67 seconds end-to-end. Your `e2e_request_latency p99` SLO screams. You page on-call, they find nothing wrong, they raise the threshold, and now the SLO is deaf to actual regressions. The latency was dominated by **output length**, a property of the *prompt*, not of your service. This lesson is about splitting the one lying number into the two honest ones — and understanding the batching knob that trades them against each other.

The stakes aren't hypothetical. Getting this split right is worth real money and real UX, not just cleaner dashboards: Anyscale's published benchmark of prefill-decode disaggregation on Ray + vLLM — separating the exact two phases this lesson teaches apart onto different GPU pools — reported **up to 2.7× better goodput and roughly 67% lower compute cost** on the same class of hardware (AMD MI325X, 2026 pricing), plus a measurably lower TTFT under load (355ms vs. 389ms prefill-heavy, 165ms vs. 190ms decode-heavy at concurrency 256) against a comparison router. That's the ceiling of what taking "prefill and decode are different bottlenecks" seriously can buy you. For a platform/FinOps engineer this is the SLO design that lets you run batches *hot* (high throughput, low $/token) while still holding a real user-experience guarantee — and it's the same "allocated vs. used" honesty thesis as the rest of the module, applied to latency instead of utilization.

## What's new here (calibration)

Nothing about histograms, quantiles, recording rules, or burn-rate alerting is new — reference your existing Prometheus muscle memory. What's new is the *shape of the workload*:

- **A request is not one unit of work.** It's a **prefill** phase (process the whole prompt in one forward pass — compute-bound, parallel over prompt tokens) followed by a **decode** phase (generate tokens one at a time, autoregressive — memory-bandwidth-bound). Two phases, two bottlenecks, two metrics.
- **Latency is unbounded by design.** Output length is chosen at inference time, so total latency has no fixed ceiling. Any percentile over it is a percentile over a distribution you don't control.
- **The KV cache is the real capacity limit**, not GPU-util%. Each in-flight sequence holds a growing key/value tensor in HBM; when the cache is full the scheduler *preempts* or *queues* requests. KV-cache utilisation and queue depth are your saturation signals, not the 100%-pinned GPU-util that (as the deliverable's title warns) tells you almost nothing here.
- **Batching couples tenants.** Continuous batching interleaves many users' decode steps into shared GPU iterations — great for throughput and cost, but one user's tokens now share a step budget with everyone else's, so batch size directly moves each user's per-token latency.

## Core concepts

### The four metrics that matter

| Metric | What it measures | Phase | Physics |
|--------|------------------|-------|---------|
| **TTFT** — time to first token | request arrival → first output token | queue + prefill | Compute-bound; grows with prompt length, queue depth, batch admission. The "is it responsive?" number. |
| **TPOT / ITL** — time per output token / inter-token latency | gap between successive output tokens (steady state) | decode | Memory-bandwidth-bound; grows with batch size and context length. The "does it feel smooth?" number. |
| **Queue depth** | requests waiting for admission (not yet running) | pre-prefill | Saturation signal; rises before TTFT does. |
| **KV-cache utilisation** | fraction of paged KV blocks in use | whole request | The true capacity ceiling; at ~100% the scheduler preempts/queues. |

**TTFT = queue wait + prefill compute.** A rising TTFT with flat prefill time means you're queueing (capacity problem); a rising TTFT with flat queue means prefill got heavier (longer prompts, or contention in the batch). Split them if you can (`request_queue_time` vs `request_prefill_time`).

**TPOT vs ITL — a precise nuance.** Per BentoML's inference-metrics reference, TPOT is *(total generation time) / (output tokens − 1)* — an average decode cadence per request. ITL is the *individual* gap between consecutive tokens, token-weighted, so longer responses carry more weight in the aggregate; for a single request, the mean of its ITLs equals its TPOT. ITL is the better lens for steady-state throughput and jitter; TPOT is the better lens for "how fast did this one user's response feel." SLO on TPOT-p99 for the average-smoothness guarantee; watch the ITL distribution for stall diagnosis. Many people use the terms interchangeably — say which you mean.

### Prefill vs decode: why there are two bottlenecks

The two phases stress different parts of the GPU, which is *why* one SLO can't cover both:

- **Prefill** processes all N prompt tokens in a single forward pass. The attention and MLP matmuls are large and parallel — it's **compute-bound**, saturating the tensor cores. Cost scales with prompt length. This is most of TTFT.
- **Decode** generates one token per forward pass, each pass re-reading the entire model weights and the growing KV cache from HBM to produce a single token. Arithmetic intensity is tiny, so it's **memory-bandwidth-bound** — the GPU's compute units sit mostly idle waiting on HBM. This is TPOT.

Consequence: a decode-heavy server can show **GPU-util at 100% while doing almost no useful FLOPs** — the SM is "busy" stalling on memory. This is the headline lie the deliverable is named for: `DCGM_FI_PROF_GR_ENGINE_ACTIVE` / util% near 100 tells you the GPU is *occupied*, not that it's *productive* and not whether users are being served well. The productivity signal is achieved tokens/sec against your latency budget; the health signal is TTFT/TPOT; util% is neither. This is precisely why architectures that physically separate the two phases exist: Cloudflare's Infire engine disaggregates prefill (compute-bound) from decode (memory-bound) onto different execution paths and reports cutting p90 inter-token latency from roughly 100ms to 20–30ms — a ~3× improvement — purely by no longer letting one phase's resource profile contend with the other's on the same step.

### Why p99 total request latency is the WRONG SLO

Total latency ≈ `TTFT + (output_tokens − 1) × TPOT`. The dominant term for anything but the shortest replies is `output_tokens × TPOT`, and **`output_tokens` is chosen by the caller / the model's stopping behaviour, not by your service's health.** So:

- **The p99 of total latency is mostly the p99 of output length.** A batch of users who happen to ask for long completions moves your SLO with zero change in service health. You're alerting on verbosity.
- **It hides real regressions.** If TPOT quietly doubles (a bad batch config, a noisy neighbour), short requests barely move the aggregate while long ones balloon — the signal is smeared across a distribution dominated by length.
- **It's not actionable.** "p99 latency is high" gives on-call no lever. "TTFT-p99 is high" → check queue depth / admission. "TPOT-p99 is high" → check batch size / KV pressure. The split metrics *are* the diagnosis.

**The correct SLO: two guarantees, separately.**
- `TTFT-p99 < X ms` — responsiveness (time to *start* streaming).
- `TPOT-p99 < Y ms/token` (or "ITL-p99") — smoothness (streaming cadence).

Optionally normalise long requests with a **per-output-token latency budget** rather than a raw total. Never a single `e2e_request_latency` percentile as *the* user-facing SLO for streaming.

If you must express a whole-request bound (some product contracts want one), use a **length-normalised** target — e.g. "total latency ≤ TTFT_budget + output_tokens × TPOT_budget," evaluated per request and then aggregated as a *compliance ratio* (fraction of requests inside budget), not a raw-latency percentile. That keeps the guarantee honest across short and long completions and folds directly into a burn-rate error budget.

### The batching trade-off (throughput vs individual latency)

**Continuous (a.k.a. in-flight / dynamic) batching** — the vLLM/TGI default — doesn't wait to assemble a fixed batch. It admits new requests into the running batch at each decode iteration and evicts finished ones, keeping the GPU's matrix units fed. The trade:

- **Improves: throughput** (tokens/sec across the fleet) and therefore **$/token** — the FinOps win. Higher `max_num_seqs` / larger effective batch = more concurrent sequences amortising each weight-load over more tokens.
- **Degrades: individual TTFT and TPOT.** A bigger batch means each decode iteration does more work → each user's *next token* comes slightly later (TPOT up). And a large in-flight prefill (someone's 8k-token prompt) stalls the decode step for everyone sharing that iteration → TTFT spikes for newcomers and an ITL glitch for those mid-stream.

So batch size is a **throughput ↔ per-user-latency dial**, and your two SLOs are the guardrails: crank the batch for cost until TTFT-p99 or TPOT-p99 approaches its budget, then stop. That's the whole game — run hot, but bounded by the two honest latencies.

**The exact lever, sourced from vLLM's own optimization guide:** chunked prefill (splitting a long prompt's prefill across several decode iterations instead of injecting it whole) is enabled by default whenever possible in the V1 engine, and the scheduler prioritizes decode requests, admitting prefill work only when the token budget allows. The size of that budget — `max_num_batched_tokens` — is the precise dial: a **smaller** value (e.g. 2048) improves **ITL**, because fewer prefill tokens compete with decode steps for the same iteration; a **larger** value improves **TTFT**, because more prefill tokens get processed per batch, finishing prompts faster. There is no free lunch here — you are explicitly trading TTFT against ITL by moving one number, and vLLM's docs separately note that pushing `max_num_batched_tokens` above roughly 8192 is the throughput-maximizing setting for smaller models, which pulls further away from a tight ITL budget. Pick the direction based on which of your two SLOs is closer to breach.

KV-cache pressure forces the scheduler to **preempt** running sequences (evict them back to the queue) when it runs out of room — a visible, countable event (`vllm:num_preemptions_total`), not a silent slowdown. vLLM's optimization guide gives four concrete levers to relieve it, in order of how directly they buy you headroom: raise `gpu_memory_utilization` (more HBM reserved for KV cache), lower `max_num_seqs` or `max_num_batched_tokens` (smaller concurrent batch, less cache pressure per step), or scale out with `tensor_parallel_size` / `pipeline_parallel_size` (spread the model and free HBM for cache on each GPU). Each trades something — the first eats into headroom for CUDA graphs and other allocations, the second is a direct throughput cut, the third costs more GPUs and adds NVLink-dependent collective latency (tying back to the XID 74 fault class from 05.5).

### Budgets are workload-specific

There's no universal TTFT/TPOT number — the budget comes from the product and, ultimately, from human perception. Spheron's 2026 SLO-engineering guide frames this precisely: for interactive chat, **300ms TTFT-p99** is roughly where users stop noticing a delay before text begins; by **500ms** most users perceive lag; by **800ms** session-abandonment rates measurably rise. On the decode side, **50ms ITL-p99** reads as near-continuous text for short responses, but the budget *tightens* as output grows — a 2,000-token response needs **ITL-p99 under ~30ms** to avoid visible stuttering, because more tokens means more chances for a spike to be noticed. Voice/real-time agents have the tightest end-to-end TTFT budget (on the order of 400ms) because a downstream TTS pipeline adds its own 100–200ms after the LLM's first sentence. Anchors to reason from, not to copy (all figures are 2026-era, perception-based, and workload-specific — recalibrate for your own product):

| Workload | TTFT budget (illustrative) | TPOT/ITL budget (illustrative) | Why |
|----------|-------------|-------------|-----|
| Interactive chat | ~300–500 ms | ~50 ms/token, tightening toward ~30ms for long replies | Human reads ~5–10 words/s; faster-than-reading is wasted, so run the batch hotter. |
| Voice / real-time agent | ~100–300 ms into an overall ~400ms budget | tight, low-jitter | Downstream TTS needs a steady stream; ITL *variance* matters as much as the mean. |
| Coding / autocomplete | very tight TTFT | high tok/s | Perceived as instant or not at all. |
| Batch / offline (eval, summarisation) | seconds OK | loose | No human waiting → maximise throughput/goodput, batch as hard as KV allows. |

The design move: pick the budgets from the use case (product research, not a table you copy), then push batch size / `max_num_seqs` up until TTFT-p99 or TPOT-p99 nears the budget. For batch workloads the budgets are so loose that you run essentially KV-cache-limited — pure throughput mode.

### Reading it in vLLM's `/metrics`

vLLM exposes Prometheus metrics (prefix `vllm:`), labelled by `model_name`:

- `vllm:time_to_first_token_seconds` — **histogram** → TTFT (`_bucket`/`_count`/`_sum`).
- `vllm:time_per_output_token_seconds` — **histogram** → TPOT/ITL cadence.
- `vllm:e2e_request_latency_seconds` — histogram (the one you do *not* SLO on for streaming).
- `vllm:request_queue_time_seconds`, `vllm:request_prefill_time_seconds`, `vllm:request_decode_time_seconds` — the TTFT decomposition.
- `vllm:num_requests_running` — sequences currently in the batch (gauge).
- `vllm:num_requests_waiting` — **queue depth** (gauge).
- `vllm:gpu_cache_usage_perc` — **KV-cache utilisation**, 0–1 (gauge).
- `vllm:num_preemptions_total` — KV pressure forced a running seq out (counter) — a smoking gun for cache exhaustion.

> Note: newer vLLM (V1 engine) has adjusted a few names/labels over releases; confirm against *your* build's `/metrics` output. The histograms above are stable and are what your panels query.

### Server-side vs client-side TTFT

vLLM's `time_to_first_token` is measured *inside the engine* — arrival at the scheduler to first token emitted. Your users experience **client-side TTFT**, which also includes network RTT, TLS, the gateway/router hop, and any auth. The gap between them is a routing/ingress problem, not a model problem. SLO the client-side number for the user contract; keep the server-side histogram to prove the engine is innocent when they diverge. If client TTFT-p99 is 1.2 s but `vllm:time_to_first_token_seconds` p99 is 150 ms, your problem is the load balancer or a cold autoscale, not the GPU. Cloudflare's Infire team make the cold-start dimension of this concrete: their engine can start serving even the largest models in under 20 seconds — a deliberate engineering target because a slow model load shows up to users indistinguishable from a TTFT regression during any rolling deploy or autoscale event.

### Things that move the metrics (and how)

- **Chunked prefill** — splits a long prompt's prefill across several iterations so it stops monopolising a decode step. Smooths **ITL** (fewer decode stalls) at a small TTFT cost for the chunked request, via the `max_num_batched_tokens` dial above.
- **Prefix / KV caching** — reuses KV blocks for shared prompt prefixes (system prompts, few-shot preambles). Cuts **prefill time → lower TTFT** on cache hit; watch a prefix-cache-hit-rate metric if your build exposes one.
- **Speculative decoding** — a draft model proposes several tokens verified in one target step. Lowers **TPOT** when acceptance is high, but *raises ITL variance* (accepted runs are fast, rejections stall) — so SLO on TPOT-p99, and expect a wider ITL distribution.
- **Tensor/pipeline parallelism** — spreads a big model across GPUs; adds per-step collective (NVLink) latency → slightly higher **TPOT**, and makes you sensitive to the XID 74 NVLink faults from the previous lesson.
- **Prefill-decode disaggregation** — the architectural move, not a tuning knob (see Worked example): run prefill and decode on separate GPU pools connected by a KV-cache transfer path, so the two phases stop competing for the same iteration's compute/memory budget at all.

### Goodput, not just throughput

Raw throughput counts every token/sec including requests that already blew their SLO. **Goodput** = throughput of requests that *met* both TTFT and TPOT budgets. Optimising raw throughput can *lower* goodput (you batched so hard that half the requests missed latency). For the FinOps framing, the honest efficiency number is **goodput per dollar**, not tokens per dollar — it's the batch setting that maximises SLO-compliant work, which is usually short of the throughput-maximising batch. Anyscale's benchmark makes goodput the headline metric for exactly this reason: they report "up to 2.7× better goodput," not raw throughput, because raw throughput alone can't tell you whether users were actually served within budget.

### Turning the two SLOs into alerts

You already run multi-window burn-rate alerting — apply it unchanged, just to the *right* metrics. Recording rules keep the quantiles cheap:

```yaml
groups:
- name: llm-slo
  rules:
  - record: slo:ttft_p99_5m
    expr: histogram_quantile(0.99, sum by (le,model_name) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
  - record: slo:tpot_p99_5m
    expr: histogram_quantile(0.99, sum by (le,model_name) (rate(vllm:time_per_output_token_seconds_bucket[5m])))
  - alert: TtftSloBurn
    expr: slo:ttft_p99_5m > 0.5           # 500 ms responsiveness budget
    for: 5m
    labels: { severity: warning }
    annotations: { summary: "TTFT-p99 over budget on {{ $labels.model_name }}" }
  - alert: TpotSloBurn
    expr: slo:tpot_p99_5m > 0.05          # 50 ms/token smoothness budget
    for: 5m
    labels: { severity: warning }
```

One caveat you know but must respect here: `histogram_quantile` over `rate()` gives an interpolated quantile bounded by the histogram's bucket boundaries — if your TTFT buckets top out at 1 s you cannot observe a p99 of 2.3 s, it'll clamp. Make sure the vLLM histogram bucket layout spans your real tail before trusting the panel.

## Perspectives

**Developer / product.** TTFT and TPOT map directly to *perceived* UX — does it feel instant, does it feel smooth — in a way total latency does not. A developer designing a chat UI needs the split metrics to reason about streaming behavior at all; "average request took 4 seconds" tells a frontend engineer nothing about whether to show a typing indicator sooner or stream faster.

**Operator / SRE.** The split enables actionable on-call: TTFT regression → capacity/admission lever (scale out, cap batch tokens, add a replica); TPOT regression → batch-size/decode lever (lower `max_num_seqs`, enable chunked prefill). A single blended SLO gives on-call nothing to turn — it's a symptom, not a diagnosis.

**Systems / hardware.** Prefill (compute-bound, tensor-core-saturating) and decode (memory-bandwidth-bound) are architecturally different regimes on the same chip. This is why one SLO covering both phases is a category error, not just a statistics problem — you're averaging two physically different workloads and calling the average meaningful.

**Economics.** Batch size is explicitly a throughput↔latency dial; "goodput" (SLO-compliant throughput) rather than raw throughput is the correct FinOps efficiency number, because you can batch so hard you generate revenue-losing SLO violations while your dashboard shows record tokens/sec. Anyscale's PD-disaggregation numbers (2.7× goodput, ~67% cost savings on the cited hardware) are the concrete payoff of taking this distinction seriously at the architecture level, not just the tuning level.

## Real-world use cases

- **Anyscale — "Achieving Up to 67% Cost Savings with Prefill-Decode Disaggregation Using Ray + vLLM on AMD MI325X"** — https://www.anyscale.com/blog/ray-vllm-prefill-decode-disaggregation-amd-mi325x-67-percent-savings. Reports up to 2.7× better goodput and roughly 67% lower compute cost from physically separating prefill and decode onto different GPU pools; also shows lower TTFT under load (355ms vs. 389ms prefill-heavy, 165ms vs. 190ms decode-heavy at concurrency 256) versus a comparison router. **What it shows:** production-grade evidence that treating "prefill and decode are different bottlenecks" as an architectural decision, not just a metric-naming exercise, produces large, measurable cost and goodput wins.
- **`vllm-project/vllm` — `docs/configuration/optimization.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/optimization.md. States precisely that chunked prefill is enabled by default whenever possible in the V1 engine, that the scheduler prioritizes decode over prefill, and that `max_num_batched_tokens` is the exact TTFT/ITL tradeoff dial (smaller favors ITL, larger favors TTFT), plus the four KV-cache-preemption remedies. **What it shows:** the authoritative, engine-level source for the exact tuning knob this lesson's core-concepts section is built around — cite verbatim rather than paraphrase.
- **Cloudflare — "How we built the most efficient inference engine for Cloudflare's network"** — https://blog.cloudflare.com/cloudflares-most-efficient-ai-inference-engine/. Cloudflare's Rust-based Infire engine disaggregates prefill and decode, cutting p90 inter-token latency from roughly 100ms to 20–30ms (about 3×) and delivering up to 20% higher throughput on unconstrained hardware; can start serving even the largest models in under 20 seconds. **What it shows:** a hyperscale edge network (180+ cities) engineering its own inference stack specifically around the latency-phase-separation this lesson teaches, with concrete before/after numbers.
- **BentoML — LLM Inference Handbook, "Key metrics for LLM inference"** — https://bentoml.com/llm/llm-inference-basics/llm-inference-metrics. Precise, vendor-neutral definitions: TTFT (queueing + prefill + network), ITL (token-weighted per-gap), TPOT (mean of ITLs for a single request). **What it shows:** the definitional reference worth citing whenever TTFT/TPOT/ITL terminology needs to be pinned down exactly.

## Worked example

**Endpoint:** vLLM serving Llama-3.1-8B on one L40S, `max_num_seqs=256`. You ramp concurrency from 1 → 200.

- At low load: `num_requests_waiting ≈ 0`, TTFT-p99 ≈ 120 ms, TPOT-p99 ≈ 22 ms/token, `gpu_cache_usage_perc ≈ 0.2`. Healthy. `e2e_request_latency p99` = 9 s — but that's just because someone asked for 400 tokens. Ignore it as an SLO.
- Ramp to 120 concurrent: throughput (tokens/sec) climbs nicely — this is the batching win. `gpu_cache_usage_perc ≈ 0.75`, `num_requests_waiting` still ~0. TPOT-p99 creeps to 30 ms (bigger batch, each decode step heavier). TTFT-p99 steady ~150 ms. You're trading a little per-token latency for a lot of throughput — exactly the intended dial.
- Ramp to 200: `gpu_cache_usage_perc` pins at ~0.98, `vllm:num_preemptions_total` starts incrementing, `num_requests_waiting` climbs to 40. **TTFT-p99 jumps to 2.3 s** (requests are now *queueing* before admission) while **TPOT-p99 stays ~32 ms** (once admitted, decode cadence is fine).

Read the split: TTFT blew up but TPOT didn't → the fault is **admission/queueing**, not decode. The lever is capacity or admission control (lower concurrency, add a replica, cap `max_num_batched_tokens`), *not* a decode-side change. The single `e2e_latency` number would have just drifted up murkily and told you none of this. This queue-depth-vs-TTFT coupling is the observation your deliverable must capture.

**The architectural contrast — same workload, disaggregated.** Now imagine the same ramp against a prefill-decode-disaggregated deployment (two GPU pools: one runs only prefill and hands off KV cache, the other runs only decode) instead of today's co-located single pool. Per Anyscale's benchmark pattern, TTFT stays materially flatter as concurrency climbs — a big incoming prompt on the prefill pool no longer steals a decode step's budget from users already streaming, because there is no shared step to steal from. The cost is architectural, not free: you're now running (and paying for) two pools of GPUs, and a KV-cache transfer path between them, instead of one pool doing both jobs. That's the real trade PD disaggregation makes explicit — it turns "tune the batch to bound the coupling" into "remove the coupling entirely, and pay for the extra hardware and transfer path instead." Whether that trade is worth it is a $/goodput calculation, exactly the one Anyscale published a number for (67% cost savings) on their specific hardware and workload — treat that figure as a 2026, hardware-specific data point, not a universal constant.

## Practice

Do this on a rented GPU (any L4/L40S/A10/A100 hour will do). Build the panels for the deliverable, linked from ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md).

1. **Deploy vLLM** with metrics on:
   ```bash
   vllm serve meta-llama/Llama-3.1-8B-Instruct --port 8000 --max-num-seqs 256
   # scrape http://<host>:8000/metrics
   ```
2. **Point Prometheus** at `/metrics` (a static scrape config is fine) and add Grafana.
3. **Load-test** with a tool that streams and reports TTFT/ITL — `vllm bench serve` (a.k.a. the benchmark_serving script) or `guidellm`. Sweep concurrency, e.g. `--max-concurrency 1,8,32,64,128,200`, with a realistic prompt/output length mix.
4. **Build two SLO panels:**
   ```promql
   # TTFT p99
   histogram_quantile(0.99, sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
   # TPOT / ITL p99
   histogram_quantile(0.99, sum by (le) (rate(vllm:time_per_output_token_seconds_bucket[5m])))
   ```
5. **Build the saturation overlay** — plot on one panel and watch the coupling:
   ```promql
   vllm:num_requests_waiting            # queue depth
   vllm:gpu_cache_usage_perc            # KV utilisation
   histogram_quantile(0.99, sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
   ```
6. **Sweep `max_num_batched_tokens`** at a fixed concurrency (try 2048 vs. 8192+) and record the TTFT-p99/ITL-p99 trade directly — this reproduces vLLM's documented tradeoff on your own hardware instead of taking it on faith.

**Acceptance:** you produce (1) a TTFT-p99 panel, (2) a TPOT-p99 panel, and (3) the written observation showing that as batch/concurrency grows, **queue depth (`num_requests_waiting`) and KV utilisation rise, and TTFT-p99 tracks them while TPOT-p99 stays comparatively flat** — with the inflection point (the concurrency where TTFT knees upward and preemptions begin) called out. Bonus: mark where you'd set each SLO threshold (justify it against real perception numbers, not a guess) and argue why `e2e_request_latency_seconds` p99 is *not* on the dashboard as an SLO.

## Common pitfalls

1. **Treating `max_num_batched_tokens` as a single throughput knob** rather than the TTFT/ITL tradeoff dial it actually is — smaller favors ITL, larger favors TTFT (sourced from vLLM's own optimization docs above). Tuning it in one direction without watching both SLOs will silently blow the other.
2. **Believing prefill-decode disaggregation is "just for hyperscalers."** Anyscale published a specific, replicable recipe (Ray + vLLM) with concrete cost numbers on commodity-available hardware (AMD MI325X), not a bespoke mega-scale system — it's an architecture decision available at moderate scale, not an exotic one.
3. **Optimizing for raw tokens/sec instead of goodput.** The same batching lever that raises throughput can silently raise SLO violations; the metric to actually optimize is SLO-compliant throughput per dollar, which is usually a smaller batch than the throughput-maximizing one.
4. **Alerting on total request latency because "it's what users complain about."** Users complain about *feel* — slow to start, or halting mid-stream — which is exactly TTFT and TPOT/ITL. A blended total-latency alert fires on verbose prompts and stays silent on real regressions buried in short ones.
5. **Copying someone else's TTFT/ITL budget numbers verbatim.** The Spheron perception thresholds above are a reasonable starting anchor, not a spec — voice, chat, and batch workloads have genuinely different human-perception ceilings, and the only correct source for your budget is your own product's users.

## Self-check

- Continuous batching improves which metric and degrades which? **Answer:** It improves throughput (tokens/sec across the server, and therefore $/token) by keeping the GPU's compute units saturated with many concurrent sequences. It degrades individual TTFT and TPOT/ITL — a larger batch makes each decode iteration heavier (later next-token for every user → higher TPOT), and a big in-flight prefill can stall the shared decode step (TTFT spike for new requests, ITL jitter for streaming ones). It's a throughput-vs-per-user-latency dial.
- Why is total request-latency p99 the wrong SLO for a streaming LLM endpoint? **Answer:** Because total latency ≈ `TTFT + (output_tokens − 1) × TPOT`, and the dominant term is `output_tokens × TPOT` — output length is chosen by the caller, not by your service. So the p99 of total latency mostly reflects the p99 of *how long people's answers are*, not service health: a batch of verbose prompts moves the SLO with zero degradation, and a real TPOT regression gets smeared across a length-dominated distribution. It's also not actionable. SLO on TTFT-p99 (responsiveness) and TPOT/ITL-p99 (smoothness) separately instead.
- Queue depth is rising while TPOT stays flat. What does that tell you? **Answer:** The bottleneck is admission/capacity, not decode. Requests are waiting to enter the running batch (rising `num_requests_waiting`, and you'll likely see KV-cache utilisation near 100% and `num_preemptions_total` climbing), which drives TTFT up — but once a request is admitted, per-token decode cadence is unaffected, so TPOT is flat. The fix is capacity/admission-side (add replicas, lower concurrency, cap `max_num_batched_tokens`/enable chunked prefill), not a decode-side change.
- Per vLLM's own optimization docs, does a smaller or larger `max_num_batched_tokens` favor ITL, and why? **Answer:** A smaller value (e.g. 2048) favors ITL, because fewer prefill tokens are admitted per iteration, so fewer prefills compete with and stall decode steps. A larger value favors TTFT instead, because more prefill tokens get processed per batch, finishing prompts (and reaching first token) faster — the two are a direct tradeoff on the same dial, not independently tunable.
- What did prefill-decode disaggregation buy Anyscale's benchmark, and what's the underlying mechanism? **Answer:** Up to 2.7× better goodput and roughly 67% lower compute cost (on AMD MI325X, 2026 pricing — a dated, hardware-specific snapshot), plus lower TTFT under load versus a co-located comparison router. The mechanism: prefill and decode run on separate GPU pools, so a large incoming prompt's compute-bound prefill work no longer competes with other users' memory-bound decode steps for the same iteration's budget — the coupling that causes TTFT spikes and ITL jitter in a co-located deployment is removed rather than just bounded.

## Connections & what's next

This lesson is the module's second demonstration of the same core thesis from 05.1 (`GPU_UTIL` lies) applied to a different layer: a decode-bound LLM server can pin `GPU_UTIL` near 100% while doing almost no useful work per cycle, which is why TTFT/TPOT — not utilization percentages — are the metrics that actually track whether users are being served well. It builds directly on 05.5's health gate (a wedged or remap-pending GPU corrupts both utilization and latency numbers at once) and feeds forward into 05.7: when TTFT or TPOT breaches its budget and the cause isn't queueing or KV pressure, profiling is the next rung — you now have a precise, metric-driven trigger ("TPOT regressed, decode-side, not capacity") to hand a profiler instead of guessing where to look.

## References & further reading

**Primary sources**
- `vllm-project/vllm` — `docs/configuration/optimization.md` — https://github.com/vllm-project/vllm/blob/main/docs/configuration/optimization.md — read for the authoritative chunked-prefill defaults, the `max_num_batched_tokens` TTFT/ITL tradeoff, and the four KV-cache-preemption remedies.
- vLLM — production metrics reference — https://docs.vllm.ai/en/latest/serving/metrics.html — the exact `vllm:` histogram and gauge names your panels query.
- BentoML — "Key metrics for LLM inference" — https://bentoml.com/llm/llm-inference-basics/llm-inference-metrics — precise, vendor-neutral definitions of TTFT, TPOT, ITL, and how they compose.

**Real-world engineering blogs**
- Anyscale — "Achieving Up to 67% Cost Savings with Prefill-Decode Disaggregation Using Ray + vLLM on AMD MI325X" — https://www.anyscale.com/blog/ray-vllm-prefill-decode-disaggregation-amd-mi325x-67-percent-savings — what it shows: production-grade goodput and cost numbers from physically separating prefill and decode.
- Cloudflare — "How we built the most efficient inference engine for Cloudflare's network" — https://blog.cloudflare.com/cloudflares-most-efficient-ai-inference-engine/ — what it shows: a hyperscale edge network's own inference engine, disaggregated for the same latency reasons this lesson teaches, with concrete p90 ITL numbers.

**Deeper dives**
- Spheron — "LLM Inference SLO Engineering: TTFT, ITL, and P99 Latency Budgets for Production AI (2026)" — https://www.spheron.network/blog/llm-inference-slo-ttft-itl-latency-budget-guide-2026/ — perception-grounded TTFT/ITL budget numbers by workload (chat, voice, long-response) to calibrate your own SLOs against.
- Ray — "Prefill/decode disaggregation" user guide — https://docs.ray.io/en/latest/serve/llm/user-guides/prefill-decode.html — the operational how-to behind the Anyscale benchmark above, for going from the concept to a deployable config.

{% endraw %}
