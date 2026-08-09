---
lesson: "05.6"
title: "Inference SLOs: TTFT, TPOT, and why request-latency lies"
module: "05"
concept: "Inference SLOs: TTFT, TPOT, and why request-latency lies"
status: not-started
est_time: "5h"
artifacts: []
---
# 05.6 · Inference SLOs: TTFT, TPOT, and why request-latency lies

> **Concept.** A streaming LLM endpoint has *two* latencies with different physics — time-to-first-token and per-token cadence — and putting your SLO on total request latency measures your users' verbosity, not your service's health.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Why this matters

You've written hundreds of `histogram_quantile(0.99, …)` request-latency SLOs for REST and gRPC services, and every one of them was correct: request latency is a clean health signal when the work per request is roughly fixed. For a streaming LLM it is *not* correct, and shipping the same SLO you'd write for an API gateway is the single most common way a GPU dashboard lies to you.

Here's the trap. A user asks for a 2000-token essay. The endpoint is perfectly healthy — sub-second to first token, smooth 30 tokens/sec after — and the request takes 67 seconds end-to-end. Your `e2e_request_latency p99` SLO screams. You page on-call, they find nothing wrong, they raise the threshold, and now the SLO is deaf to actual regressions. The latency was dominated by **output length**, a property of the *prompt*, not of your service. This lesson is about splitting the one lying number into the two honest ones — and understanding the batching knob that trades them against each other. For a platform/FinOps engineer this is the SLO design that lets you run batches *hot* (high throughput, low $/token) while still holding a real user-experience guarantee.

## What's new for you

Nothing about histograms, quantiles, recording rules, or burn-rate alerting is new — reference your existing Prometheus muscle memory. What's new is the *shape of the workload*:

- **A request is not one unit of work.** It's a **prefill** phase (process the whole prompt in one forward pass — compute-bound, parallel over prompt tokens) followed by a **decode** phase (generate tokens one at a time, autoregressive — memory-bandwidth-bound). Two phases, two bottlenecks, two metrics.
- **Latency is unbounded by design.** Output length is chosen at inference time, so total latency has no fixed ceiling. Any percentile over it is a percentile over a distribution you don't control.
- **The KV cache is the real capacity limit**, not GPU-util%. Each in-flight sequence holds a growing key/value tensor in HBM; when the cache is full the scheduler *preempts* or *queues* requests. KV-cache utilisation and queue depth are your saturation signals, not the 100%-pinned GPU-util that (as the deliverable's title warns) tells you almost nothing here.
- **Batching couples tenants.** Continuous batching interleaves many users' decode steps into shared GPU iterations — great for throughput and cost, but one user's tokens now share a step budget with everyone else's, so batch size directly moves each user's per-token latency.

## Core notes

### The four metrics that matter

| Metric | What it measures | Phase | Physics |
|--------|------------------|-------|---------|
| **TTFT** — time to first token | request arrival → first output token | queue + prefill | Compute-bound; grows with prompt length, queue depth, batch admission. The "is it responsive?" number. |
| **TPOT / ITL** — time per output token / inter-token latency | gap between successive output tokens (steady state) | decode | Memory-bandwidth-bound; grows with batch size and context length. The "does it feel smooth?" number. |
| **Queue depth** | requests waiting for admission (not yet running) | pre-prefill | Saturation signal; rises before TTFT does. |
| **KV-cache utilisation** | fraction of paged KV blocks in use | whole request | The true capacity ceiling; at ~100% the scheduler preempts/queues. |

**TTFT = queue wait + prefill compute.** A rising TTFT with flat prefill time means you're queueing (capacity problem); a rising TTFT with flat queue means prefill got heavier (longer prompts, or contention in the batch). Split them if you can (`request_queue_time` vs `request_prefill_time`).

**TPOT vs ITL — a precise nuance.** TPOT is usually defined as *(total generation time) / (output tokens − 1)* — an average decode cadence per request. ITL is the *individual* gap between consecutive tokens; its distribution shows jitter (e.g. a decode-step stall when a big prefill barges into the batch). SLO on TPOT-p99 for the average-smoothness guarantee; watch ITL distribution for stall diagnosis. Many people use the terms interchangeably — say which you mean.

### Prefill vs decode: why there are two bottlenecks

The two phases stress different parts of the GPU, which is *why* one SLO can't cover both:

- **Prefill** processes all N prompt tokens in a single forward pass. The attention and MLP matmuls are large and parallel — it's **compute-bound**, saturating the tensor cores. Cost scales with prompt length. This is most of TTFT.
- **Decode** generates one token per forward pass, each pass re-reading the entire model weights and the growing KV cache from HBM to produce a single token. Arithmetic intensity is tiny, so it's **memory-bandwidth-bound** — the GPU's compute units sit mostly idle waiting on HBM. This is TPOT.

Consequence: a decode-heavy server can show **GPU-util at 100% while doing almost no useful FLOPs** — the SM is "busy" stalling on memory. This is the headline lie the deliverable is named for: `DCGM_FI_PROF_GR_ENGINE_ACTIVE` / util% near 100 tells you the GPU is *occupied*, not that it's *productive* and not whether users are being served well. The productivity signal is achieved tokens/sec against your latency budget; the health signal is TTFT/TPOT; util% is neither.

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

So batch size is a **throughput ↔ per-user-latency dial**, and your two SLOs are the guardrails: crank the batch for cost until TTFT-p99 or TPOT-p99 approaches its budget, then stop. That's the whole game — run hot, but bounded by the two honest latencies. (Related knobs: chunked prefill to stop big prompts from stalling decode; `max_num_batched_tokens` to cap per-iteration work.)

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

vLLM's `time_to_first_token` is measured *inside the engine* — arrival at the scheduler to first token emitted. Your users experience **client-side TTFT**, which also includes network RTT, TLS, the gateway/router hop, and any auth. The gap between them is a routing/ingress problem, not a model problem. SLO the client-side number for the user contract; keep the server-side histogram to prove the engine is innocent when they diverge. If client TTFT-p99 is 1.2 s but `vllm:time_to_first_token_seconds` p99 is 150 ms, your problem is the load balancer or a cold autoscale, not the GPU.

### Things that move the metrics (and how)

- **Chunked prefill** — splits a long prompt's prefill across several iterations so it stops monopolising a decode step. Smooths **ITL** (fewer decode stalls) at a small TTFT cost for the chunked request. If you see ITL spikes correlated with long-prompt arrivals, this is the knob.
- **Prefix / KV caching** — reuses KV blocks for shared prompt prefixes (system prompts, few-shot preambles). Cuts **prefill time → lower TTFT** on cache hit; watch a prefix-cache-hit-rate metric if your build exposes one.
- **Speculative decoding** — a draft model proposes several tokens verified in one target step. Lowers **TPOT** when acceptance is high, but *raises ITL variance* (accepted runs are fast, rejections stall) — so SLO on TPOT-p99, and expect a wider ITL distribution.
- **Tensor/pipeline parallelism** — spreads a big model across GPUs; adds per-step collective (NVLink) latency → slightly higher **TPOT**, and makes you sensitive to the XID 74 NVLink faults from the previous lesson.

### Goodput, not just throughput

Raw throughput counts every token/sec including requests that already blew their SLO. **Goodput** = throughput of requests that *met* both TTFT and TPOT budgets. Optimising raw throughput can *lower* goodput (you batched so hard that half the requests missed latency). For the FinOps framing, the honest efficiency number is **goodput per dollar**, not tokens per dollar — it's the batch setting that maximises SLO-compliant work, which is usually short of the throughput-maximising batch.

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

## Worked example

**Endpoint:** vLLM serving Llama-3.1-8B on one L40S, `max_num_seqs=256`. You ramp concurrency from 1 → 200.

- At low load: `num_requests_waiting ≈ 0`, TTFT-p99 ≈ 120 ms, TPOT-p99 ≈ 22 ms/token, `gpu_cache_usage_perc ≈ 0.2`. Healthy. `e2e_request_latency p99` = 9 s — but that's just because someone asked for 400 tokens. Ignore it as an SLO.
- Ramp to 120 concurrent: throughput (tokens/sec) climbs nicely — this is the batching win. `gpu_cache_usage_perc ≈ 0.75`, `num_requests_waiting` still ~0. TPOT-p99 creeps to 30 ms (bigger batch, each decode step heavier). TTFT-p99 steady ~150 ms. You're trading a little per-token latency for a lot of throughput — exactly the intended dial.
- Ramp to 200: `gpu_cache_usage_perc` pins at ~0.98, `vllm:num_preemptions_total` starts incrementing, `num_requests_waiting` climbs to 40. **TTFT-p99 jumps to 2.3 s** (requests are now *queueing* before admission) while **TPOT-p99 stays ~32 ms** (once admitted, decode cadence is fine). 

Read the split: TTFT blew up but TPOT didn't → the fault is **admission/queueing**, not decode. The lever is capacity or admission control (lower concurrency, add a replica, cap `max_num_batched_tokens`), *not* a decode-side change. The single `e2e_latency` number would have just drifted up murkily and told you none of this. This queue-depth-vs-TTFT coupling is the observation your deliverable must capture.

## Practice

Do this on a rented GPU (any L4/L40S/A10/A100 hour will do). Build the panels for the deliverable.

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

**Acceptance:** you produce (1) a TTFT-p99 panel, (2) a TPOT-p99 panel, and (3) the written observation showing that as batch/concurrency grows, **queue depth (`num_requests_waiting`) and KV utilisation rise, and TTFT-p99 tracks them while TPOT-p99 stays comparatively flat** — with the inflection point (the concurrency where TTFT knees upward and preemptions begin) called out. Bonus: mark where you'd set each SLO threshold and argue why `e2e_request_latency_seconds` p99 is *not* on the dashboard as an SLO.

## Self-check

**(a)** Continuous batching improves which metric and degrades which?

**Answer:** It **improves throughput** (tokens/sec across the server, and therefore $/token) by keeping the GPU's compute units saturated with many concurrent sequences. It **degrades individual TTFT and TPOT/ITL** — a larger batch makes each decode iteration heavier (later next-token for every user → higher TPOT), and a big in-flight prefill can stall the shared decode step (TTFT spike for new requests, ITL jitter for streaming ones). It's a throughput-vs-per-user-latency dial.

**(b)** Why is total request-latency p99 the wrong SLO for a streaming LLM endpoint?

**Answer:** Because total latency ≈ `TTFT + (output_tokens − 1) × TPOT`, and the dominant term is `output_tokens × TPOT` — output length is chosen by the caller, not by your service. So the p99 of total latency mostly reflects the p99 of *how long people's answers are*, not service health: a batch of verbose prompts moves the SLO with zero degradation, and a real TPOT regression gets smeared across a length-dominated distribution. It's also not actionable. SLO on TTFT-p99 (responsiveness) and TPOT/ITL-p99 (smoothness) separately instead.

**(c)** Queue depth is rising while TPOT stays flat. What does that tell you?

**Answer:** The bottleneck is **admission/capacity, not decode.** Requests are waiting to enter the running batch (rising `num_requests_waiting`, and you'll likely see KV-cache utilisation near 100% and `num_preemptions_total` climbing), which drives **TTFT up** — but once a request is admitted, per-token decode cadence is unaffected, so TPOT is flat. The fix is capacity/admission-side (add replicas, lower concurrency, cap `max_num_batched_tokens`/enable chunked prefill), not a decode-side change.

## Resources

1. **BentoML — LLM inference metrics** (deep, precise definitions of TTFT, TPOT, ITL, throughput, and how they compose) https://bentoml.com/llm/llm-inference-basics/llm-inference-metrics
2. **Spheron — LLM inference SLO / latency-budget guide (2026)** (TTFT/ITL SLO design and per-token latency budgets) https://www.spheron.network/blog/llm-inference-slo-ttft-itl-latency-budget-guide-2026/
3. **vLLM production metrics** — the authoritative `/metrics` reference for the exact `vllm:` histogram and gauge names your panels query. https://docs.vllm.ai/en/latest/serving/metrics.html
