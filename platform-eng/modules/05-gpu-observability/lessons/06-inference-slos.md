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
sources: 14
---

# 05.6 · Inference SLOs: TTFT, TPOT, and why request-latency lies

> **Concept.** A streaming LLM endpoint has *two* latencies with different physics — time-to-first-token and per-token cadence — and putting your SLO on total request latency measures your users' verbosity, not your service's health.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

05.5 gave you the health gate: is this GPU even trustworthy to run on. This lesson assumes yes and asks the next question — is the *service* running on it meeting the promise it made to a user? That is a different failure mode: a fleet of perfectly healthy GPUs can still deliver a bad experience if the SLO you are watching is the wrong metric.

It is also the module's second act on the util-lie thesis. 05.1 taught you that `DCGM_FI_DEV_GPU_UTIL` can read 100% while `SM_ACTIVE` is near zero on a decode workload. This lesson explains *why that workload shape exists* — with the arithmetic, not the assertion — and gives you the SLOs that actually track user-visible health instead. The derivation in §3 is the missing "why" behind the exhibit your capstone is built on: a batch-1 LLM decode is memory-bandwidth-bound by roughly **two orders of magnitude**, so the GPU is genuinely occupied and genuinely doing almost no arithmetic, at the same time, for a reason you can compute from a datasheet.

What it unlocks: 05.7's profiling ladder needs a concrete "this metric looks wrong" trigger to escalate from, and the split TTFT/ITL metrics here are exactly that trigger for inference workloads. The capstone needs serving namespaces to be measurable on something other than utilisation.

Everything version-specific below is checked against the **vLLM main-branch metrics implementation** (`vllm/v1/metrics/loggers.py` and `docs/design/metrics.md`), **TGI's router** (`router/src/server.rs`), and **Prometheus 3.x documentation**. Several metric names in circulation are stale; §7 says which and what they became.

## Why this matters

You have written hundreds of `histogram_quantile(0.99, …)` request-latency SLOs for REST and gRPC services, and every one of them was correct: request latency is a clean health signal when the work per request is roughly fixed. For a streaming LLM it is *not* correct, and shipping the same SLO you would write for an API gateway is the single most common way a GPU dashboard lies to you.

Here is the trap, with numbers. A user asks for a 2,000-token essay. The endpoint is perfectly healthy — 120 ms to first token, a smooth 30 ms per token after. The request takes `0.12 + 1999 × 0.030 = 60.1 s` end to end. Another user asks for a 40-token answer on the same server in the same second: `0.12 + 39 × 0.030 = 1.29 s`. **Same service, same health, 47× difference in the number your SLO is watching.** Your `e2e_request_latency` p99 tracks the 99th percentile of *how long people's answers are*. On-call gets paged, finds nothing wrong, raises the threshold, and the SLO is now deaf to actual regressions.

Getting the split right is also worth real money, not just cleaner dashboards. Anyscale's published benchmark of prefill–decode disaggregation on Ray + vLLM — separating the exact two phases this lesson teaches apart onto different GPU pools — reported **up to 2.7× better goodput and roughly 67% lower compute cost** on AMD MI325X (2026 pricing), plus measurably lower TTFT under load (355 ms vs 389 ms prefill-heavy, 165 ms vs 190 ms decode-heavy at concurrency 256) against a comparison router. Cloudflare's Rust-based Infire engine, which makes the same architectural split, reports cutting p90 inter-token latency from roughly 100 ms to 20–30 ms — about 3×.

For a platform/FinOps engineer this is the SLO design that lets you run batches *hot* — high throughput, low $/token — while still holding a real user-experience guarantee. It is the same "allocated vs used" honesty thesis as the rest of the module, applied to latency instead of utilisation.

## What's new here (calibration)

Nothing about histograms, quantiles, recording rules, or burn-rate alerting is new — bring your existing Prometheus muscle memory. What is new is the *shape of the workload* and what that does to standard SLO practice:

- **A request is not one unit of work.** It is a **prefill** phase (process the whole prompt in one forward pass) followed by a **decode** phase (generate tokens one at a time, autoregressively). Two phases, two bottlenecks, two metrics.
- **The two phases sit on opposite sides of the roofline**, and you can prove it from two datasheet numbers. §3 does the arithmetic. This is not an analogy.
- **Latency is unbounded by design.** Output length is chosen at inference time, so total latency has no fixed ceiling. Any percentile over it is a percentile over a distribution you do not control.
- **The KV cache is the real capacity limit**, not GPU-util%. Each in-flight sequence holds a growing key/value tensor in HBM; when the cache is full the scheduler *preempts* or *queues*. KV-cache utilisation and queue depth are your saturation signals.
- **Batching couples tenants.** Continuous batching interleaves many users' decode steps into shared GPU iterations. One user's tokens now share a step budget with everyone else's.
- **ITL and TPOT are different metrics, and modern vLLM exposes both separately.** Most write-ups (including the previous version of this lesson) conflate them and cite a metric name that no longer exists.
- **Your SLO threshold must land on a histogram bucket boundary**, or your "99% under 30 ms" number is an interpolation artefact. §11 shows exactly where vLLM's boundaries are and which budgets are therefore measurable.

## Core concepts

### 1. The anatomy of one streaming request

Everything in this lesson comes from being precise about which interval you are naming. Draw the request once, properly, and the SLO definitions stop being vocabulary and start being obvious.

```
   CLIENT                    ROUTER/API                 ENGINE (vLLM scheduler + GPU)
      │                          │                              │
  t0  ├── HTTP POST ────────────▶│                              │
      │   (network + TLS)        ├── tokenize, validate ───────▶│  arrival_time
      │                          │                              │  ┌──────────────┐
      │                          │                              │  │   WAITING    │  ← queue
      │                          │                              │  │  (no GPU     │    depth
      │                          │                              │  │   work yet)  │    lives
      │                          │                              │  └──────┬───────┘    here
      │                          │                              │         │ SCHEDULED
      │                          │                              │  ┌──────▼───────┐
      │                          │                              │  │   PREFILL    │  ← one (or a
      │                          │                              │  │  N prompt    │    few, if
      │                          │                              │  │  tokens in   │    chunked)
      │                          │                              │  │  fwd pass(es)│    fwd passes
      │                          │                              │  └──────┬───────┘
      │                          │◀── token 1 ──────────────────┤  NEW_TOKENS #1
  t1  │◀── first SSE chunk ──────┤                              │  ┌──────▼───────┐
      │                          │◀── token 2 ──────────────────┤  │              │
      │                          │◀── token 3 ──────────────────┤  │    DECODE    │
      │                          │        ⋮                     │  │  1 token per │
      │                          │◀── token M ──────────────────┤  │  fwd pass    │
  t2  │◀── [DONE] ───────────────┤                              │  └──────────────┘
      │                          │                              │

  ├─────────────── TTFT ────────────────┤
  │  (client-side: t1 − t0)             │
  │                                     ├──ITL──┼──ITL──┼─ … ─┼──ITL──┤
  │           ┌──────────┬──────────────┤       (M−1 gaps between M tokens)
  │           │queue wait│ prefill compute
  │
  ├───────────────────────── e2e request latency (t2 − t0) ─────────────────────┤
  │                                                                             │
  │   ≈  TTFT  +  (M − 1) × TPOT          where TPOT = mean of this request's ITLs
  │                └── YOU control this ──┘   └── the CALLER controls M ──┘
```

The four intervals, named precisely:

| Interval | Definition | vLLM metric | Physics |
|---|---|---|---|
| **Queue wait** | arrival at scheduler → first time this request is `SCHEDULED` | `vllm:request_queue_time_seconds` | Pure saturation. Zero when there is capacity. |
| **Prefill** | most recent `SCHEDULED` → first `NEW_TOKENS` | `vllm:request_prefill_time_seconds` | **Compute-bound.** Scales with prompt length. |
| **TTFT** | frontend `arrival_time` (tokenisation start) → first output token | `vllm:time_to_first_token_seconds` | ≈ queue wait + prefill + input-processing overhead. |
| **ITL** | gap between consecutive `NEW_TOKENS` events | `vllm:inter_token_latency_seconds` | **Memory-bandwidth-bound.** Scales with batch size and context length. |
| **TPOT** | this request's total generation time ÷ (output tokens − 1) — i.e. the **mean of that request's ITLs** | `vllm:request_time_per_output_token_seconds` | Same physics as ITL, aggregated per request. |
| **e2e** | frontend `arrival_time` → final token | `vllm:e2e_request_latency_seconds` | Dominated by output length. **Not an SLO.** |

**ITL vs TPOT, precisely — and this is where most people are sloppy.** ITL is the population of *individual gaps*, so a 2,000-token response contributes 1,999 observations and a 40-token response contributes 39. The aggregate ITL distribution is therefore **token-weighted**: long responses dominate it. TPOT is one observation *per request* — the mean cadence that request experienced — so the aggregate TPOT distribution is **request-weighted**, and every user counts equally regardless of how much they asked for.

Which to SLO on follows directly from that:

- **SLO on TPOT-p99** when the guarantee is per-user ("99% of users get a smooth stream"). Request-weighted is what a user-facing contract means.
- **Watch the ITL distribution** for diagnosis. A stall — one 400 ms gap in an otherwise 25 ms stream — barely moves that request's TPOT (it is one of 2,000 gaps) but shows up loudly in ITL's tail. **ITL-p99 is your stall detector; TPOT-p99 is your promise.**

Both are exposed. Use both. And when you say "TPOT" in an interview, say which weighting you mean — that single sentence separates people who have read a blog post from people who have run the service.

vLLM computes all of these from monotonic timestamps inside the engine-core process (`time.monotonic()`), specifically to avoid wall-clock drift between the frontend and the engine. One consequence documented in vLLM's own design notes: **preemption distorts these intervals**. A request preempted during decode has its inter-token, decode and inference intervals stretched by the time it spent back in the waiting queue; a request preempted during prefill has its TTFT and prefill intervals stretched. So a TTFT spike with a simultaneous rise in `vllm:num_preemptions_total` is not two problems — it is one, and the preemption counter is the cause.

### 2. The identity that makes the whole argument

```
    e2e ≈ TTFT + (M − 1) × TPOT              M = output tokens
```

Take derivatives in your head. `∂e2e/∂M = TPOT ≈ 20–50 ms`. For any M beyond about 20 tokens, **the `M × TPOT` term dominates**, and M is a property of the prompt and the model's stopping behaviour — not of your service's health.

That single fact generates all three reasons total-latency p99 is the wrong SLO:

1. **The p99 of e2e is mostly the p99 of output length.** A shift in traffic mix toward long-form generation moves your SLO with zero change in service health. You are alerting on verbosity.
2. **It hides real regressions.** If TPOT doubles from 25 ms to 50 ms, a 40-token request goes from 1.1 s to 2.1 s while a 2,000-token request goes from 50 s to 100 s. In a mixed distribution the regression is smeared across a range dominated by length, and the p99 may move less than the day-to-day variance in M.
3. **It is not actionable.** "p99 latency is high" gives on-call no lever. "TTFT-p99 is high" → check queue depth and admission. "ITL-p99 is high" → check batch size and KV pressure. **The split metrics *are* the diagnosis**, which is the property a good SLO has and a blended one does not.

If a product contract demands a whole-request bound, express it **length-normalised**: `e2e ≤ TTFT_budget + M × TPOT_budget`, evaluated per request, then aggregated as a *compliance ratio* (fraction of requests inside budget) rather than a raw-latency percentile. That form folds directly into the error-budget arithmetic in §10 and stays honest across short and long completions.

### 3. Why there are two bottlenecks — the arithmetic

This is the part that is usually asserted. Derive it instead; it takes four lines and it is the thing that makes the util-lie exhibit make sense.

**The model.** A dense transformer with **P** parameters in **bf16** (2 bytes each). Ignore attention and the KV cache for the moment — for an 8B-class model at moderate context, weight traffic dominates.

**Per forward pass over T tokens:**

- **Compute:** roughly `2 × P × T` FLOPs. (Two FLOPs — a multiply and an add — per parameter per token, which is what a matmul does.)
- **Memory traffic:** you must read every weight from HBM once, regardless of T: `2 × P` bytes.

**Arithmetic intensity** — the roofline x-axis, FLOPs per byte of HBM traffic:

```
                2 × P × T  FLOPs
    I(T)  =  ─────────────────────  =  T   FLOP/byte
                  2 × P    bytes
```

**The intensity of a forward pass is just the number of tokens in it.** That is the whole result, and it is why the two phases are different:

- **Prefill** of a 2,000-token prompt: `T = 2000` → **I ≈ 2000 FLOP/byte**.
- **Decode** at batch size B: each step produces one token per sequence, so `T = B` → **I ≈ B FLOP/byte**. At batch 1, **I ≈ 1**.

Now the hardware. A GPU's **machine balance** is `peak FLOP/s ÷ peak HBM bytes/s` — the intensity at which compute and memory are equally busy (the roofline's ridge point):

| GPU | Dense BF16 tensor (TFLOP/s) | HBM bandwidth (TB/s) | Ridge point (FLOP/byte) |
|---|---|---|---|
| A100 80GB SXM | ~312 | 2.04 | ≈ 153 |
| L40S | ~181 | 0.864 | ≈ 210 |
| H100 SXM | ~990 | 3.35 | ≈ 296 |

*(Dense figures. NVIDIA's headline tensor numbers — 624 / 362 / 1,979 TFLOPS — are quoted **with structured sparsity**, which LLM inference does not use, so halve them. If your source treats those as dense, every ridge point below doubles and the conclusion is unchanged, because decode's intensity is off by orders of magnitude either way.)*

```
      FLOP/s
        ▲
  peak  ┤                    ╭───────────────────────────────  COMPUTE ROOF
        │                   ╱ │
        │                 ╱   │
        │               ╱     │        ●  PREFILL (T = 2000)
        │             ╱       │           I ≈ 2000 FLOP/byte
        │           ╱         │           → compute-bound, tensor cores hot
        │         ╱           │
        │       ╱  ← MEMORY   │
        │     ╱      ROOF     │
        │   ╱   (slope = HBM  │
        │ ╱      bandwidth)   │
        │╱                    │
      ──●─────────────────────┼──────────────────────────────▶  FLOP/byte
        ▲                   ridge
        │                   ≈296 (H100)
   DECODE, batch 1
   I ≈ 1 FLOP/byte
   → ~300× below the ridge: the GPU can only reach ~1/300 of its
     peak FLOP/s no matter what you do. The SMs have warps resident
     (SM_ACTIVE high, GPU_UTIL pinned at 100) and are stalled on HBM
     (PIPE_TENSOR_ACTIVE ≈ 0.01–0.1).
```

Put real time on it. Llama-3.1-8B in bf16: `P = 8×10⁹`, weights = **16 GB**.

```
  Per decode step on one H100 SXM (3.35 TB/s HBM, ~990 TFLOP/s dense bf16):

    weight read time   = 16 GB ÷ 3.35 TB/s          = 4.78 ms   ← the floor
    compute time (B=1) = 2 × 8e9 × 1 FLOP ÷ 990e12  = 0.016 ms
                                                      ────────
    ratio                                             ≈ 300 ×

  So a batch-1 decode step is ~4.8 ms of HBM traffic wrapped around
  16 µs of arithmetic.  Theoretical batch-1 ceiling ≈ 1/4.78 ms ≈ 209 tok/s
  (real systems land lower: attention, KV reads, kernel launch, sampling).

  Batch it:
    B =  32 →  compute 0.52 ms  vs memory 4.78 ms  → still memory-bound
    B = 128 →  compute 2.07 ms  vs memory 4.78 ms  → still memory-bound
    B = 296 →  compute 4.78 ms  vs memory 4.78 ms  → RIDGE
```

**Three conclusions fall straight out, and they are the whole lesson:**

1. **Batching is nearly free until you approach the ridge.** Going from B=1 to B=32 costs 0.5 ms of extra compute on a step that already takes 4.78 ms — roughly **11% more time for 32× the tokens.** That is why continuous batching is the single biggest $/token lever in inference, and why the throughput curve is so steep at low concurrency.
2. **`GPU_UTIL` is guaranteed to lie during decode.** A kernel *is* resident for essentially the whole 4.78 ms, so the occupancy duty cycle reads ~100%. The tensor pipes are idle for ~99.7% of it. The lie is not a measurement bug; it is the arithmetic above, and this is the mechanism behind the exhibit your capstone screenshots.
3. **Chunked prefill is not a hack, it is roofline arithmetic.** A 2,000-token prefill is compute-bound; a decode step is memory-bound. Run them in the *same* iteration and the memory-bound work rides along in the shadow of the compute-bound work at near-zero marginal cost. That is why vLLM's V1 scheduler mixes them by default rather than alternating phases.

*(Refinement for long contexts: the KV cache adds `2 × layers × kv_heads × head_dim × 2 bytes` of traffic per token of context, per sequence, per step. At 32k+ context this stops being negligible and can exceed weight traffic, which is why ITL degrades as conversations grow even at constant batch size. The direction of the conclusion does not change — it gets *more* memory-bound, not less.)*

### 4. Continuous batching, mechanically

Static batching waits to assemble N requests, runs them to completion together, and returns them together — so every request pays the *slowest* one's latency and the GPU idles while stragglers finish. **Continuous (in-flight, iteration-level) batching** — the vLLM/TGI/SGLang default — schedules at the granularity of a single forward pass:

```
  ONE DECODE ITERATION ≈ 5 ms on an H100.  The scheduler re-decides every iteration.

  iter k        iter k+1       iter k+2       iter k+3       iter k+4
  ┌─────────┐   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
  │ A  tok7 │   │ A  tok8 │    │ A  tok9 │    │ A  tok10│    │ A  tok11│
  │ B  tok2 │   │ B  tok3 │    │ B  DONE │    │         │    │         │
  │ C  tok41│   │ C  tok42│    │ C  tok43│    │ C  tok44│    │ C  tok45│
  │         │   │ D  PRE  │    │ D  PRE  │    │ D  tok1 │    │ D  tok2 │
  │         │   │  chunk1 │    │  chunk2 │    │         │    │ E  PRE  │
  │         │   │  (512t) │    │  (512t) │    │         │    │  chunk1 │
  └─────────┘   └─────────┘    └─────────┘    └─────────┘    └─────────┘
       │             │              │              │              │
       │        D arrives:     D's 1024-tok    B's KV blocks   E admitted
       │        admitted as    prompt done;    freed → E can   into the
       │        a PREFILL      D joins decode  be admitted     token budget
       │        chunk in the        │
       │        SAME iteration      │
       │             │              │
       └── each iteration's total token count ≤ max_num_batched_tokens ──┘
           each iteration's sequence count    ≤ max_num_seqs

  ⇒ B's completion frees a slot MID-STREAM. Nobody waits for the batch.
  ⇒ D's prefill is CHUNKED so it cannot monopolise one iteration and
     stall A and C's next token (which is what an unchunked 8k prefill does:
     one ~40 ms iteration instead of ~5 ms, i.e. an 8× ITL spike for
     everyone else in the batch).
```

Two scheduler knobs control this, and vLLM's optimization guide is explicit that they are a **tradeoff dial, not a throughput dial**:

- **`max_num_batched_tokens`** — the per-iteration token budget. **Smaller (e.g. 2048) improves ITL**, because fewer prefill tokens compete with decode steps in a given iteration. **Larger improves TTFT**, because more prefill tokens get processed per iteration so prompts finish sooner. vLLM recommends setting it **above 8192**, especially for smaller models on large GPUs, when you are maximising throughput — which explicitly pulls away from a tight ITL budget.
- **`max_num_seqs`** — the maximum concurrent sequences. This is the direct handle on decode batch size, and therefore on where you sit relative to the ridge point in §3.

Two more mechanics worth knowing because they show up in incidents:

- **Chunked prefill is enabled by default whenever possible in the V1 engine**, and **the scheduler prioritises decode requests**, admitting prefill work only when the token budget allows. So under load the system automatically protects ITL at the expense of TTFT. If you did not know that, you will misread a TTFT-only regression as a capacity problem when it is the scheduler doing exactly what it was designed to do.
- **The default preemption mode in V1 is `RECOMPUTE`, not `SWAP`** — when KV cache runs out, vLLM drops a sequence's cached KV and later recomputes its prefill rather than swapping blocks to host memory, because recomputation is cheaper than PCIe round-trips in the V1 architecture. Operationally: a preemption costs that request a **second full prefill**, which is why preemptions hit TTFT-like intervals so hard.

### 5. The KV cache is the capacity ceiling

Every in-flight sequence holds key/value tensors for every token it has seen, in HBM, in **paged blocks** (vLLM's PagedAttention borrows the OS virtual-memory idea: fixed-size blocks, a per-sequence block table, no requirement for contiguity, so fragmentation does not waste cache).

Size it, because "the KV cache is the limit" is meaningless until you can compute the limit:

```
  KV bytes per token  =  2 (K and V)
                       × num_layers
                       × num_kv_heads × head_dim
                       × bytes_per_element

  Llama-3.1-8B (32 layers, 8 KV heads [GQA], head_dim 128, bf16):
      = 2 × 32 × 8 × 128 × 2  =  131,072 bytes  =  128 KiB per token

  On one 80 GB H100:
      weights (bf16)                        16 GB
      activations / CUDA graphs / workspace  ~4 GB
      gpu_memory_utilization = 0.90       → 72 GB usable
      ⇒ KV cache budget                    ≈ 52 GB

      52 GB ÷ 128 KiB/token ≈ 425,000 tokens of KV

  Translate to concurrency:
      avg 2k context/request  → ~208 concurrent sequences
      avg 8k context/request  → ~52  concurrent sequences
      avg 32k context/request → ~13  concurrent sequences
```

**This is the number your `max_num_seqs` must respect**, and it is why a service that is comfortable at 2k context falls over when a customer starts sending 32k prompts *with no change in request rate*. Your capacity is measured in tokens of KV, not in requests.

When the cache fills, the scheduler **preempts** — a countable event, not a silent slowdown. vLLM's four documented remedies, in order of how directly they buy headroom:

| Remedy | What it buys | What it costs |
|---|---|---|
| Raise `gpu_memory_utilization` | More HBM reserved for KV | Less headroom for CUDA graphs, activations, fragmentation → OOM risk |
| Lower `max_num_seqs` | Less KV pressure per step | Direct throughput cut |
| Lower `max_num_batched_tokens` | Less prefill competing per iteration | Worse TTFT |
| Raise `tensor_parallel_size` / `pipeline_parallel_size` | Model sharded → more free HBM per GPU for KV | More GPUs; per-step collective latency (and sensitivity to the XID 74 NVLink faults from 05.5) |

### 6. Goodput, not throughput

Raw throughput counts every token/s, including tokens delivered to requests that already blew their SLO. **Goodput** is the throughput of requests that met *both* budgets. Optimising raw throughput can *lower* goodput: you batch so hard that half the requests miss latency, and your dashboard shows a record tokens/s while your users churn.

For the FinOps framing the honest efficiency number is **goodput per dollar**, and the batch setting that maximises it is usually short of the throughput-maximising one. Anyscale headlines goodput ("up to 2.7×") rather than throughput for exactly this reason: raw throughput cannot tell you whether users were served within budget.

Concretely:

```
  One H100 at $2.50/GPU-hr (dated 2026 rate-card snapshot; verify yours).

  Setting A — max_num_seqs=256, throughput-maximising
      6,400 tok/s total   ·   TTFT-p99 2.3 s   ·   ITL-p99 55 ms
      SLO (TTFT ≤ 500 ms, ITL ≤ 50 ms): only 46% of requests comply
      goodput      = 6,400 × 0.46      = 2,944 SLO-compliant tok/s
      $/M goodput  = 2.50 ÷ (2,944 × 3600 ÷ 1e6) = $0.236

  Setting B — max_num_seqs=96, SLO-aware
      4,900 tok/s total   ·   TTFT-p99 380 ms  ·   ITL-p99 38 ms
      98% of requests comply
      goodput      = 4,900 × 0.98      = 4,802 SLO-compliant tok/s
      $/M goodput  = 2.50 ÷ (4,802 × 3600 ÷ 1e6) = $0.145

  Setting A wins on the throughput dashboard by 31%.
  Setting B wins on cost-per-USEFUL-token by 39%.
```

*(Illustrative numbers with the arithmetic shown so you can re-run it with your own measurements — the point is the method and the sign of the result, not the specific throughputs.)*

### 7. The real metric names — and which ones you have wrong

Metric names in this ecosystem drift, and most tutorials are stale. Here is what current builds actually expose.

#### vLLM (V1 engine, main branch)

All series carry `model_name` and — new in V1 — an **`engine`** label, which matters because a multi-engine deployment will silently double-count if you forget to `sum by (le, model_name)` rather than `sum by (le)`.

| Metric | Type | What it is |
|---|---|---|
| `vllm:time_to_first_token_seconds` | Histogram | TTFT. **The responsiveness SLO.** |
| `vllm:inter_token_latency_seconds` | Histogram | **ITL** — individual token gaps, token-weighted. The stall detector. |
| `vllm:request_time_per_output_token_seconds` | Histogram | **TPOT** — per-request mean cadence, request-weighted. **The smoothness SLO.** |
| `vllm:e2e_request_latency_seconds` | Histogram | End-to-end. Keep on the dashboard, never as an SLO. |
| `vllm:request_queue_time_seconds` | Histogram | Time in WAITING. The TTFT decomposition's first half. |
| `vllm:request_prefill_time_seconds` | Histogram | Time in PREFILL. The second half. |
| `vllm:request_decode_time_seconds` | Histogram | Time in DECODE. |
| `vllm:request_inference_time_seconds` | Histogram | Time in RUNNING (prefill + decode). |
| `vllm:num_requests_running` | Gauge | Sequences in the current execution batch. |
| `vllm:num_requests_waiting` | Gauge | **Queue depth.** The leading saturation indicator. |
| `vllm:num_requests_waiting_by_reason` | Gauge | Queue depth split by `reason`: `capacity` (waiting for scheduling capacity) vs `deferred` (LoRA budget, KV transfer, blocked). **Sums to `num_requests_waiting`.** |
| `vllm:kv_cache_usage_perc` | Gauge | KV-cache usage, 1 = 100%. The capacity ceiling from §5. |
| `vllm:num_preemptions_total` | Counter | KV pressure forced a running sequence out. The smoking gun. |
| `vllm:prefix_cache_queries_total` / `vllm:prefix_cache_hits_total` | Counters | In **tokens**, not requests. Hit rate = hits ÷ queries. |
| `vllm:iteration_tokens_total` | Histogram | Tokens per engine step. Buckets `[1, 8, 16, …, 16384]`. Shows how full each iteration is against `max_num_batched_tokens`. |
| `vllm:prompt_tokens_total` / `vllm:generation_tokens_total` | Counters | Throughput numerators. |
| `vllm:request_success_total` | Counter | Labelled `finished_reason` (stop / length / abort). |
| `vllm:cache_config_info` | Gauge (Info) | `block_size`, `gpu_memory_utilization`, `enable_prefix_caching`, … at startup. |

**Corrections to names you will find in older material (including the previous version of this lesson):**

| Stale name | Current name | Why it matters |
|---|---|---|
| `vllm:time_per_output_token_seconds` | `vllm:inter_token_latency_seconds` (ITL) **and** `vllm:request_time_per_output_token_seconds` (TPOT) | The old single metric conflated the two weightings. **Panels built on the old name silently return no data.** |
| `vllm:gpu_cache_usage_perc` | `vllm:kv_cache_usage_perc` | Same. |
| `vllm:cpu_cache_usage_perc`, `vllm:num_requests_swapped` | *removed* | The CPU-swap path is gone in V1 (preemption is `RECOMPUTE`). |
| `vllm:prefix_cache_hit_rate` | *replaced* by the two counters | Deliberate: a pre-computed ratio cannot be re-aggregated correctly across replicas or windows. Compute it in PromQL. |
| `vllm:avg_prompt_throughput_toks_per_s`, `vllm:tokens_total`, `vllm:time_in_queue_requests` | *removed / never implemented* | Pre-averaged gauges are the same anti-pattern. |

**Verify against your own build before you ship a dashboard:** `curl -s localhost:8000/metrics | grep -E '^# HELP vllm:' | sort`. Pin the vLLM version in your dashboard's README. This is not pedantry — a renamed metric produces an empty panel, and an empty panel is indistinguishable from a healthy one at 3am.

#### TGI (Text Generation Inference)

Different names, same concepts, and one genuinely useful metric vLLM does not have — a batch-level forward-duration histogram labelled by method:

| Metric | Type | Maps to |
|---|---|---|
| `tgi_request_queue_duration` | Histogram | Queue wait |
| `tgi_request_inference_duration` | Histogram | Prefill + decode |
| `tgi_request_mean_time_per_token_duration` | Histogram | **TPOT** (per-request mean) |
| `tgi_request_duration` | Histogram | e2e |
| `tgi_request_validation_duration` | Histogram | Tokenise/validate — the bit vLLM folds into TTFT overhead |
| `tgi_queue_size` | Gauge | Queue depth |
| `tgi_batch_current_size` | Gauge | Running batch size |
| `tgi_batch_current_max_tokens` / `tgi_batch_total_tokens` | Gauges | KV/token budget occupancy — TGI's analogue of `kv_cache_usage_perc` |
| `tgi_batch_forward_duration` / `tgi_batch_decode_duration` / `tgi_batch_inference_duration` | Histograms | **Labelled by method (`prefill` / `decode`)** — the cleanest direct evidence of the two-phase split in any mainstream stack |
| `tgi_request_input_length` / `tgi_request_generated_tokens` | Histograms | The M distribution — plot this next to your e2e panel and the "we are alerting on verbosity" argument makes itself |
| `tgi_request_count`, `tgi_request_success`, `tgi_request_failure` (labelled `err`: overloaded / validation / template / incomplete) | Counters | Availability numerator/denominator; `err="overloaded"` is TGI's explicit shed signal |

Note **TGI has no direct TTFT histogram** — you reconstruct it as `queue_duration + prefill-side inference time`, or measure it client-side. That is a real gap and a reason to prefer client-side TTFT for the user-facing SLO regardless of stack (§8).

#### SGLang

Prefixed `sglang:`, enabled with `--enable-metrics`. Names deliberately mirror vLLM's: `sglang:time_to_first_token_seconds`, `sglang:inter_token_latency_seconds`, `sglang:num_running_reqs`, `sglang:num_queue_reqs`, `sglang:cache_hit_rate`, `sglang:token_usage`.

#### The portable mapping

Build your dashboards against this table, not against one engine's names, and a stack migration is a recording-rule change instead of a dashboard rewrite:

| Concept | vLLM | TGI | SGLang |
|---|---|---|---|
| TTFT | `vllm:time_to_first_token_seconds` | *(derive or client-side)* | `sglang:time_to_first_token_seconds` |
| ITL | `vllm:inter_token_latency_seconds` | *(derive from batch decode duration)* | `sglang:inter_token_latency_seconds` |
| TPOT | `vllm:request_time_per_output_token_seconds` | `tgi_request_mean_time_per_token_duration` | *(derive)* |
| Queue depth | `vllm:num_requests_waiting` | `tgi_queue_size` | `sglang:num_queue_reqs` |
| Running batch | `vllm:num_requests_running` | `tgi_batch_current_size` | `sglang:num_running_reqs` |
| KV pressure | `vllm:kv_cache_usage_perc` | `tgi_batch_current_max_tokens` / `tgi_batch_total_tokens` | `sglang:token_usage` |
| Shed / preempt | `vllm:num_preemptions_total` | `tgi_request_failure{err="overloaded"}` | — |

### 8. Server-side vs client-side TTFT

vLLM's `time_to_first_token_seconds` is measured *inside the engine* — frontend arrival to first token emitted. Your users experience **client-side TTFT**, which additionally includes DNS, TCP, TLS, the ingress/gateway hop, the router's model-selection logic, auth, and any queueing in front of the engine.

**SLO the client-side number** — that is the user contract. **Keep the server-side histogram to prove the engine is innocent** when they diverge. If client TTFT-p99 is 1.2 s and `vllm:time_to_first_token_seconds` p99 is 150 ms, your problem is the load balancer, a cold autoscale, or a router doing something expensive — not the GPU, and no amount of batch tuning will help.

The cold-start dimension is real and often missed: Cloudflare engineered Infire to start serving even their largest models in **under 20 seconds**, explicitly because a slow model load is indistinguishable from a TTFT regression to a user during any rolling deploy or scale-out. If your model takes 4 minutes to load and your HPA scales on queue depth, every scale-out event is a TTFT incident whose cause is invisible in every engine-side metric you have. Instrument model load time separately and exclude loading replicas from the SLO denominator (or accept them and size your error budget for it — but decide deliberately).

### 9. Choosing the budgets

There is no universal TTFT/ITL number; the budget comes from the product and ultimately from human perception. Useful anchors, all **2026-era, perception-based, workload-specific** — reason from them, do not copy them:

| Workload | TTFT budget | ITL/TPOT budget | Why |
|---|---|---|---|
| Interactive chat | ~300 ms is roughly where users stop noticing; by ~500 ms most perceive lag; by ~800 ms abandonment measurably rises | ~50 ms reads as near-continuous for short replies; tighten toward **~30 ms for 2,000-token responses** | A human reads 5–10 words/s (≈ 7–15 tok/s), so faster-than-reading is wasted — but *more tokens means more chances for a spike to be noticed*, which is why the budget tightens with length |
| Voice / real-time agent | ~100–300 ms inside an overall ~400 ms budget | tight, and **low-jitter** | A downstream TTS pipeline adds its own 100–200 ms after the first sentence; ITL *variance* matters as much as the mean |
| Coding autocomplete | very tight | high tok/s | Perceived as instant or not at all |
| Batch / offline eval | seconds fine | loose | No human waiting → run KV-cache-limited, pure throughput mode |

The design move: pick budgets from the use case (product research, not a copied table), then raise `max_num_seqs` until TTFT-p99 or TPOT-p99 approaches its budget, then stop. That is the whole game — run hot, bounded by the two honest latencies.

### 10. Error budgets and burn-rate alerting, worked

You know multi-window multi-burn-rate alerting. Apply it unchanged — the only novelty is that there are **two SLIs**, and each one needs its own budget.

**Define the SLI as a ratio, not a quantile.** This is the important move. A quantile cannot be aggregated across replicas, cannot be re-windowed, and cannot be turned into a budget. A ratio can:

```
    SLI_ttft = (requests with TTFT ≤ 0.5 s) / (all requests)
```

which, because histograms are cumulative, is *exactly* the `le="0.5"` bucket over the count:

```promql
# TTFT compliance ratio over 1h. EXACT, not interpolated — see §11.
  sum by (model_name) (rate(vllm:time_to_first_token_seconds_bucket{le="0.5"}[1h]))
/
  sum by (model_name) (rate(vllm:time_to_first_token_seconds_count[1h]))
```

**Now the budget arithmetic.** Service: 20,000,000 requests / 30 days. SLO: **99% of requests have TTFT ≤ 500 ms**.

```
  Error budget (fraction)      = 1 − 0.99          = 0.01
  Error budget (requests/30d)  = 20e6 × 0.01       = 200,000 bad requests
  Budget burn at exactly SLO   = 200,000 ÷ 720 h   ≈ 278 bad requests/hour

  BURN RATE = (bad-event ratio observed) ÷ (1 − SLO)
            = how many times faster than "budget lasts exactly 30 days"

  Choose alert windows by "how much budget may burn before I want to know":

    consume  2% of budget in   1 h  → burn rate = 0.02 ÷ (1/720)   = 14.4
    consume  5% of budget in   6 h  → burn rate = 0.05 ÷ (6/720)   =  6
    consume 10% of budget in  3 d   → burn rate = 0.10 ÷ (72/720)  =  1

  Convert each burn rate to a threshold on the observed bad-event ratio:

    burn 14.4 → bad ratio > 14.4 × 0.01 = 14.4 %   (page)
    burn  6   → bad ratio >  6   × 0.01 =  6.0 %   (page)
    burn  1   → bad ratio >  1   × 0.01 =  1.0 %   (ticket)

  Sanity-check the fast burn: 14.4% of 20e6/720 ≈ 27,778 req/h = 4,000 bad/h.
  4,000 ÷ 200,000 = 2% of the month's budget, in one hour. ✓ matches the design.

  Each alert uses a LONG window (to establish the burn is real) AND a SHORT
  window at 1/12th (to confirm it is still happening, so the alert resolves
  promptly instead of ringing for the rest of the long window).
```

As rules:

```yaml
groups:
- name: llm-slo-ttft
  interval: 30s
  rules:

  # ── SLI: the fraction of requests OUTSIDE the 500 ms TTFT budget. ─────────
  #    0.5 is a real vLLM TTFT bucket boundary, so this is exact (§11).
  - record: slo:ttft_bad_ratio:rate1h
    expr: |
      1 - (
        sum by (model_name) (rate(vllm:time_to_first_token_seconds_bucket{le="0.5"}[1h]))
        /
        sum by (model_name) (rate(vllm:time_to_first_token_seconds_count[1h]))
      )
  - record: slo:ttft_bad_ratio:rate5m
    expr: |
      1 - (
        sum by (model_name) (rate(vllm:time_to_first_token_seconds_bucket{le="0.5"}[5m]))
        /
        sum by (model_name) (rate(vllm:time_to_first_token_seconds_count[5m]))
      )
  - record: slo:ttft_bad_ratio:rate6h
    expr: |
      1 - (
        sum by (model_name) (rate(vllm:time_to_first_token_seconds_bucket{le="0.5"}[6h]))
        /
        sum by (model_name) (rate(vllm:time_to_first_token_seconds_count[6h]))
      )
  - record: slo:ttft_bad_ratio:rate30m
    expr: |
      1 - (
        sum by (model_name) (rate(vllm:time_to_first_token_seconds_bucket{le="0.5"}[30m]))
        /
        sum by (model_name) (rate(vllm:time_to_first_token_seconds_count[30m]))
      )

  # ── FAST BURN: 2% of a 30-day budget in 1h ⇒ burn rate 14.4. ─────────────
  - alert: TtftErrorBudgetFastBurn
    expr: |
      slo:ttft_bad_ratio:rate1h  > (14.4 * 0.01)
        and
      slo:ttft_bad_ratio:rate5m  > (14.4 * 0.01)
    for: 2m
    labels: { severity: page, slo: ttft }
    annotations:
      summary: "TTFT budget burning 14.4× on {{ $labels.model_name }}"
      diagnosis: "Check vllm:num_requests_waiting first (admission), then kv_cache_usage_perc and num_preemptions_total."

  # ── SLOW BURN: 5% in 6h ⇒ burn rate 6. ───────────────────────────────────
  - alert: TtftErrorBudgetSlowBurn
    expr: |
      slo:ttft_bad_ratio:rate6h  > (6 * 0.01)
        and
      slo:ttft_bad_ratio:rate30m > (6 * 0.01)
    for: 15m
    labels: { severity: page, slo: ttft }

  # ── Remaining budget, for the dashboard. ─────────────────────────────────
  - record: slo:ttft_budget_remaining_ratio
    expr: |
      1 - (
        (1 - (
          sum by (model_name) (rate(vllm:time_to_first_token_seconds_bucket{le="0.5"}[30d]))
          /
          sum by (model_name) (rate(vllm:time_to_first_token_seconds_count[30d]))
        )) / 0.01
      )
```

Do the same for TPOT with `vllm:request_time_per_output_token_seconds_bucket{le="0.05"}` (50 ms — also a real bucket boundary). **Two SLOs, two budgets, two burn-rate pairs.** Do not blend them into one composite: a service can be responsive and stuttery, or smooth and slow to start, and those have different fixes.

**And keep the diagnostic saturation alerts separate from the SLO alerts.** The SLO alert tells you the promise is breaking; these tell you why:

```yaml
- alert: LlmQueueBuilding
  expr: avg_over_time(vllm:num_requests_waiting[5m]) > 10
  for: 5m
  labels: { severity: warning, cause: admission }

- alert: LlmKvCachePressure
  expr: |
    avg_over_time(vllm:kv_cache_usage_perc[5m]) > 0.9
      or
    rate(vllm:num_preemptions_total[5m]) > 0
  for: 5m
  labels: { severity: warning, cause: kv-exhaustion }
```

### 11. Histogram mechanics — where SLO numbers go wrong

Three failure modes, all avoidable, all common.

**(a) Your threshold must be a bucket boundary.** `histogram_quantile` interpolates *linearly within a bucket*, assuming observations are uniformly distributed inside it. If they cluster near one edge — and latency distributions always do — the interpolated quantile can be badly wrong. But a **cumulative bucket count is exact**: `…_bucket{le="0.5"}` is precisely the number of observations ≤ 0.5 s, no assumption required.

So look at where vLLM's boundaries actually are:

```
  vllm:time_to_first_token_seconds buckets (seconds):
    0.001 0.005 0.01 0.02 0.04 0.06 0.08 0.1 0.25 0.5 0.75 1.0 2.5 5.0
    7.5 10.0 20.0 40.0 80.0 160.0 640.0 2560.0

  vllm:inter_token_latency_seconds
  and vllm:request_time_per_output_token_seconds buckets (seconds):
    0.01 0.025 0.05 0.075 0.1 0.15 0.2 0.3 0.4 0.5 0.75 1.0 2.5 5.0
    7.5 10.0 20.0 40.0 80.0

  vllm:e2e_request_latency_seconds / request_queue_time / prefill / decode:
    0.3 0.5 0.8 1.0 1.5 2.0 2.5 5.0 10.0 15.0 20.0 30.0 40.0 50.0
    60.0 120.0 240.0 480.0 960.0 1920.0 7680.0
```

Now the practical consequences:

- **A 500 ms TTFT SLO is exactly measurable** — `0.5` is a boundary. So is 250 ms, 100 ms, 1 s.
- **A 300 ms TTFT SLO is not.** It falls between 0.25 and 0.5, a bucket **250 ms wide**. Your "99% under 300 ms" is an interpolation across a quarter-second — which, for a distribution with most of its mass near 0.25, will systematically *understate* the violation rate. Either move the budget to 250 ms, or change the buckets.
- **A 30 ms ITL SLO is not measurable either.** The ITL buckets step 0.025 → 0.05, so 30 ms sits 20% into a 25 ms-wide bucket. **The §9 anchor of "~30 ms ITL for long responses" is not observable with default vLLM buckets** — a genuinely useful thing to notice before you promise it to a product owner. Use 25 ms (tighter, honest) or 50 ms (looser, honest), or configure custom buckets.
- **The e2e/queue/prefill histograms start at 0.3 s.** Everything faster lands in one bucket. Fine for e2e; it makes `request_queue_time_seconds` nearly useless for sub-300 ms queueing, which is exactly the regime you care about. Use `num_requests_waiting` as the queue signal instead, and treat the queue-time histogram as a coarse confirmation.

**(b) The top bucket clamps.** `histogram_quantile` cannot return a value above the highest finite boundary in a meaningful way — observations beyond it land in `+Inf` and the function returns the top boundary. vLLM's TTFT histogram tops out at 2,560 s and ITL at 80 s, so clamping is unlikely there; it is a real hazard with hand-rolled histograms and with client-side instrumentation you wrote yourself. Check your top bucket exceeds your worst realistic tail.

**(c) Aggregate before you quantile, and aggregate the right things.** Always:

```promql
histogram_quantile(0.99,
  sum by (le, model_name) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
```

Summing by `le` first merges replicas' buckets and *then* computes the quantile. Averaging per-replica p99s is a different (and wrong) number. And note the `by (le, model_name)` — with the V1 `engine` label present, dropping it is what you want for a service-level view, but *forgetting* it while also failing to `sum` will give you one series per engine and a panel that looks like noise.

**(d) Native histograms change the calculus.** Prometheus **native histograms became a stable feature in v3.8.0** and carry through the **v3.13.0 LTS** line. They use exponential bucketing with per-series schema, so resolution is dense everywhere rather than only where someone guessed — which eliminates failure mode (a) almost entirely and makes an arbitrary 30 ms ITL budget genuinely measurable. If you are on Prometheus 3.8+ and your client library supports it, prefer native histograms for latency SLIs. Until then, **pick budgets that land on bucket boundaries.**

### 12. What moves the metrics, and which one

A quick-reference table for on-call, because "TTFT is high" should map to a lever in one hop:

| Change | TTFT | ITL/TPOT | Throughput | Mechanism |
|---|---|---|---|---|
| ↑ `max_num_seqs` | ↑ slightly | **↑ (worse)** | **↑↑** | Bigger decode batch → heavier iteration → later next token for everyone (§3) |
| ↑ `max_num_batched_tokens` | **↓ (better)** | ↑ (worse) | ↑ | More prefill per iteration; prefill competes with decode |
| ↓ `max_num_batched_tokens` (e.g. 2048) | ↑ (worse) | **↓ (better)** | ↓ | Fewer prefill tokens stalling decode steps |
| Chunked prefill (default on) | ↑ slightly for the chunked request | **↓↓ (much better)** | ↑ | An 8k prefill no longer monopolises one iteration |
| Prefix / KV caching | **↓↓ on hit** | — | ↑ | Shared prefixes (system prompts, few-shot preambles) skip prefill entirely. Watch `prefix_cache_hits ÷ prefix_cache_queries` |
| Speculative decoding | — | **↓ mean, ↑ variance** | ↑ | A draft model proposes k tokens verified in one target step; accepted runs are fast, rejections stall. **SLO on TPOT-p99; expect a fatter ITL tail** |
| ↑ `tensor_parallel_size` | ↓ (more KV headroom) | ↑ slightly | ↑ | Per-step NVLink collectives add latency; ties to XID 74 from 05.5 |
| ↑ `gpu_memory_utilization` | ↓ | ↓ | ↑ | More KV cache → fewer preemptions → fewer recomputed prefills |
| Longer contexts (same rate) | ↑ | **↑** | ↓ | KV traffic per step grows; concurrency ceiling falls (§5) |
| Prefill–decode disaggregation | **↓↓ under load** | **↓↓** | ↑ goodput | Removes the coupling entirely — separate GPU pools, KV transferred between them. Costs two pools plus a transfer path |

## Perspectives

**Developer / product.** TTFT and TPOT map directly to *perceived* UX — does it feel instant, does it feel smooth — in a way total latency does not. "Average request took 4 seconds" tells a frontend engineer nothing about whether to show a typing indicator sooner or stream faster. TTFT-p99 = 380 ms and TPOT-p99 = 38 ms tells them both.

**Operator / SRE.** The split enables actionable on-call in one hop: TTFT regression with flat TPOT → admission/capacity (scale out, cap `max_num_batched_tokens`, shed); TPOT regression with flat queue → decode-side (lower `max_num_seqs`, check context growth, check speculative-decoding acceptance). A single blended SLO gives on-call a symptom, not a diagnosis.

**Systems / hardware.** Prefill and decode sit on opposite sides of the roofline — intensity `T` vs intensity `B`, against a ridge point around 150–300 FLOP/byte on current parts. One SLO covering both phases is a **category error, not just a statistics problem**: you are averaging two physically different regimes and calling the average meaningful. It is also why `GPU_UTIL` reads 100% during decode with tensor pipes near-idle — the same arithmetic, seen from the counter side.

**Economics.** Batch size is explicitly a throughput↔latency dial, and goodput per dollar — not tokens per dollar — is the correct efficiency number. §6's worked comparison shows a setting that wins the throughput dashboard by 31% while losing on cost-per-useful-token by 39%. Anyscale's disaggregation numbers (2.7× goodput, ~67% cost saving on their hardware) are what taking the distinction seriously at the *architecture* level buys, rather than just at the tuning level.

**Capacity planner.** Your unit of capacity is **tokens of KV cache**, not requests or even GPUs (§5). A 425,000-token KV budget is 208 concurrent 2k-context sessions or 13 concurrent 32k ones. Any capacity model that counts requests will be wrong the week your customers discover long context.

## Real-world use cases

- **vLLM — `docs/design/metrics.md` and `vllm/v1/metrics/loggers.py`** — https://github.com/vllm-project/vllm/blob/main/docs/design/metrics.md. The V1 metrics design records QUEUED / SCHEDULED / PREEMPTED / NEW_TOKENS engine events on a monotonic clock and derives every interval from them; it documents that preemption during decode distorts inter-token/decode/inference intervals and preemption during prefill distorts TTFT/prefill. It also lists an explicit deprecation set (`vllm:tokens_total` never implemented, `vllm:time_in_queue_requests` duplicated, `vllm:prefix_cache_hit_rate` replaced by raw counters, `vllm:num_requests_swapped` and `vllm:cpu_cache_usage_perc` removed with the V1 swap path). **What it shows:** metric names in this ecosystem are *not* stable, the engine deliberately publishes counters rather than pre-computed ratios so PromQL can aggregate them correctly, and the timestamps you are quantiling have documented distortions you should know about before you page someone.

- **vLLM — `docs/configuration/optimization.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/optimization.md. States that chunked prefill is enabled by default whenever possible in V1, that the scheduler prioritises decode requests, that **smaller `max_num_batched_tokens` favours ITL while larger favours TTFT**, that `> 8192` is the throughput-oriented recommendation especially for smaller models on large GPUs, that V1's default preemption mode is `RECOMPUTE` rather than `SWAP`, and gives the four KV-preemption remedies. **What it shows:** the authoritative, engine-level source for the exact dial this lesson is built around — and the fact that the default configuration already trades TTFT away to protect ITL, which changes how you read a TTFT-only regression.

- **Anyscale — "Achieving Up to 67% Cost Savings with Prefill-Decode Disaggregation Using Ray + vLLM on AMD MI325X"** — https://www.anyscale.com/blog/ray-vllm-prefill-decode-disaggregation-amd-mi325x-67-percent-savings. Up to 2.7× better **goodput** and roughly 67% lower compute cost from physically separating prefill and decode onto different GPU pools; lower TTFT under load (355 ms vs 389 ms prefill-heavy, 165 ms vs 190 ms decode-heavy at concurrency 256) versus a comparison router. **What it shows:** production-grade evidence that "prefill and decode are different bottlenecks" is an architectural decision with large measurable payoff — and note they headline *goodput*, because raw throughput cannot express whether users were served in budget.

- **Cloudflare — "How we built the most efficient inference engine for Cloudflare's network"** — https://blog.cloudflare.com/cloudflares-most-efficient-ai-inference-engine/. Their Rust engine Infire disaggregates prefill from decode, cutting p90 inter-token latency from roughly 100 ms to 20–30 ms (~3×) and delivering up to 20% higher throughput on unconstrained hardware; it can start serving even the largest models in under 20 seconds. **What it shows:** a network operating in 180+ cities engineering its own stack around exactly this phase separation — and treating **cold-start time as a first-class latency concern**, which is the §8 point about client-side TTFT during autoscale.

- **BentoML — LLM Inference Handbook, "Key metrics for LLM inference"** — https://bentoml.com/llm/llm-inference-basics/llm-inference-metrics. Vendor-neutral definitions: TTFT includes queueing + prefill + network; ITL is the individual token-weighted gap; TPOT is the mean of a single request's ITLs. **What it shows:** the definitional reference to cite when the ITL/TPOT terminology needs pinning down — and confirmation that the weighting distinction in §1 is the standard one, not a local convention.

- **Prometheus — native histograms** — https://prometheus.io/docs/specs/native_histograms/. Stable as of **v3.8.0**, carried through the **v3.13.0 LTS** (1 July 2026). Classic histograms (and native histograms with custom boundaries) use linear interpolation within a bucket, which over- or under-states quantiles when observations cluster near a bucket edge; native exponential histograms interpolate under a higher-resolution assumption instead. **What it shows:** the mechanism behind §11's "your SLO threshold must be a bucket boundary," and the upgrade that makes the problem go away.

## Worked example

**Endpoint.** vLLM (V1 engine) serving Llama-3.1-8B-Instruct on one H100 80GB, `max_num_seqs=256`, chunked prefill on by default, prefix caching on. Workload: 60% short chat (≈300-token prompt, ≈150-token output), 40% long-form (≈2,000-token prompt, ≈1,200-token output). SLO: **TTFT-p99 ≤ 500 ms** and **TPOT-p99 ≤ 50 ms**, both at 99% compliance over 30 days.

Ramp concurrency 1 → 200 with `vllm bench serve` and read the panels.

### Phase 1 — concurrency 8: healthy, and the trap in plain sight

```
vllm:num_requests_waiting                              0
vllm:kv_cache_usage_perc                               0.06
vllm:num_requests_running                              8
TTFT-p99                                             118 ms
TPOT-p99                                              21 ms
ITL-p99                                               24 ms
e2e p99                                             27.4 s     ← ignore
throughput                                           410 tok/s
DCGM_FI_DEV_GPU_UTIL                                    99 %   ← the lie
DCGM_FI_PROF_SM_ACTIVE                                0.71
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE                       0.04     ← the truth
```

**Read this carefully, because it is the whole module in one screenshot.** The service is healthy on both real SLOs and yet `GPU_UTIL` says 99% while the tensor pipes are 4% active. That is §3's arithmetic showing up on a dashboard: at batch 8 the decode step's intensity is ~8 FLOP/byte against a ridge of ~296, so the GPU spends ~97% of every step reading weights from HBM. And `e2e p99 = 27.4 s` is not a problem — it is `0.118 + 1199 × 0.021 ≈ 25.3 s` for the long-form tail, which is exactly what a healthy service producing 1,200 tokens looks like.

**If you had a p99-e2e SLO at, say, 10 s, you would be permanently in violation on a perfectly healthy service.** That is the lie this lesson exists to kill.

### Phase 2 — concurrency 120: the batching win, priced

```
vllm:num_requests_waiting                              1
vllm:kv_cache_usage_perc                               0.74
vllm:num_requests_running                            118
TTFT-p99                                             164 ms
TPOT-p99                                              31 ms
ITL-p99                                               36 ms
throughput                                          4,850 tok/s
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE                       0.19
```

Throughput up **11.8×** for a **48%** increase in TPOT. That is §3's "batching is nearly free below the ridge" in the field: you moved from intensity ~8 to ~118 FLOP/byte, still short of 296, so the extra compute rides in the shadow of memory traffic. Tensor activity rose from 0.04 to 0.19 — the honest efficiency metric moved, and it moved because you batched, not because you bought anything.

Both SLOs still comfortably inside budget. **This is the operating point you want** — and note you found it by watching two latencies, not by watching a utilisation number that was pinned at 99% in both phases.

### Phase 3 — concurrency 200: the knee

```
vllm:num_requests_waiting                             42
vllm:num_requests_waiting_by_reason{reason="capacity"} 42
vllm:kv_cache_usage_perc                             0.98
vllm:num_preemptions_total                    rate 3.2 /s      ← smoking gun
vllm:num_requests_running                            156
TTFT-p99                                           2,310 ms     ← BLOWN
TPOT-p99                                              34 ms     ← fine
ITL-p99                                              210 ms     ← spiky
throughput                                        5,020 tok/s   ← barely up
```

**Read the split.** TTFT blew up 14×; TPOT moved 10%. So the fault is **admission and queueing**, not decode. The chain is visible end to end:

1. KV cache hits 98% → the scheduler cannot admit new sequences.
2. `num_requests_waiting` climbs to 42, all with `reason="capacity"` (not `deferred` — so it is genuine KV/token-budget pressure, not a LoRA or KV-transfer stall).
3. Preemptions start at 3.2/s. Each preempted request will need a **full prefill recomputation** (V1's `RECOMPUTE` mode), which is why ITL-p99 jumped to 210 ms — those are the stalls of preempted-and-resumed sequences — while TPOT-p99 stayed at 34 ms, because a single 200 ms stall barely moves the *mean* cadence of a 1,200-token request. **That divergence between ITL-p99 and TPOT-p99 is the fingerprint of preemption**, and it is exactly why §1 says to keep both.
4. Throughput gained 3.5% for all that. You are past the useful end of the dial.

The lever is capacity/admission-side: lower `max_num_seqs`, raise `gpu_memory_utilization` if there is headroom, add a replica, or shed. **Not** a decode-side change. A single e2e number would have drifted up murkily and told you none of this.

### The error-budget consequence

Suppose phase 3 lasts 40 minutes before someone reacts, at 27,800 req/h, with 71% of requests exceeding the 500 ms TTFT budget during the incident.

```
  Requests during incident   = 27,800 × (40/60)         = 18,533
  Bad requests               = 18,533 × 0.71            = 13,158
  Monthly error budget       = 20e6 × 0.01              = 200,000
  Budget consumed            = 13,158 ÷ 200,000         = 6.6 %

  Burn rate during the incident = 0.71 ÷ 0.01           = 71 ×

  Time to exhaust the WHOLE month's budget at that rate:
      720 h ÷ 71 ≈ 10.1 hours
```

A 71× burn is 5× above the 14.4 fast-burn page threshold, so `TtftErrorBudgetFastBurn` fires within its 2-minute `for:` once the 5-minute window fills — roughly 7 minutes into the incident, not 40. **That is what the burn-rate framing buys you: the alert fires proportional to how fast you are spending, not at an arbitrary latency threshold.** And 6.6% of a month's budget for one 40-minute event is a number you can put in a postmortem and in a capacity request.

### The architectural contrast

Run the same ramp against a prefill–decode-disaggregated deployment: one pool runs only prefill and hands off KV cache, the other runs only decode. Per Anyscale's pattern, TTFT stays materially flatter as concurrency climbs, because a large incoming prompt on the prefill pool cannot steal a decode step's budget from users already streaming — there is no shared step to steal from. The cost is architectural, not free: two pools of GPUs plus a KV-transfer path instead of one pool doing both jobs.

That is the trade PD disaggregation makes explicit: it converts "tune the batch to *bound* the coupling" into "*remove* the coupling and pay for extra hardware and a transfer path." Whether it is worth it is a $/goodput calculation — exactly the one in §6, and exactly the one Anyscale published a number for (67% saving) on their specific hardware and workload. Treat that figure as a dated, hardware-specific data point, not a constant.

## Practice

Do this on a rented GPU (any L4/L40S/A10/A100/H100 hour will do). Build the panels for the deliverable, linked from ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md).

### 1. Deploy and verify the metric names *first*

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8000 --max-num-seqs 256 --gpu-memory-utilization 0.90

# BEFORE building any panel — confirm what THIS build actually exposes.
curl -s localhost:8000/metrics | grep -E '^# HELP vllm:' | sort
vllm --version   # record it in the dashboard README
```

**Acceptance:** you have a written list of the exact metric names your build exposes, and you have noted whether it is `inter_token_latency_seconds` / `kv_cache_usage_perc` (current) or `time_per_output_token_seconds` / `gpu_cache_usage_perc` (older). Every query below uses *your* names.

### 2. Scrape and load-test

```yaml
# prometheus.yml
scrape_configs:
  - job_name: vllm
    scrape_interval: 5s          # you are measuring seconds-scale phenomena
    static_configs:
      - targets: ['vllm-host:8000']
```

```bash
vllm bench serve \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --dataset-name random --random-input-len 1024 --random-output-len 256 \
  --max-concurrency 200 --num-prompts 2000
# sweep: 1, 8, 32, 64, 120, 200
```

### 3. The two SLO panels

```promql
# TTFT p99 — aggregate buckets across replicas FIRST, then quantile.
histogram_quantile(0.99,
  sum by (le, model_name) (rate(vllm:time_to_first_token_seconds_bucket[5m])))

# TPOT p99 (request-weighted — the user promise)
histogram_quantile(0.99,
  sum by (le, model_name) (rate(vllm:request_time_per_output_token_seconds_bucket[5m])))

# ITL p99 (token-weighted — the stall detector). Plot on the SAME panel as TPOT;
# a large gap between them IS the preemption/stall signature.
histogram_quantile(0.99,
  sum by (le, model_name) (rate(vllm:inter_token_latency_seconds_bucket[5m])))
```

### 4. The saturation overlay

One panel, four series, so the causal chain is visible in a single glance:

```promql
vllm:num_requests_waiting
vllm:kv_cache_usage_perc * 100
rate(vllm:num_preemptions_total[5m])
histogram_quantile(0.99, sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[5m]))) * 1000
```

### 5. The exact-ratio SLI and the burn-rate rules

Implement §10's recording rules and both burn-rate alerts against **your** budget. **Acceptance:** you can state, from the dashboard, (a) the current bad-event ratio, (b) the current burn rate, and (c) how many hours of budget remain at that rate.

### 6. Reproduce the documented tradeoff

Fix concurrency at 120 and sweep `--max-num-batched-tokens` at 2048, 4096, 8192, 16384. Record TTFT-p99 and ITL-p99 at each. **Acceptance:** a four-row table showing ITL improving and TTFT degrading as the value falls — reproducing vLLM's documented tradeoff on your own hardware instead of taking it on faith. Note which direction *your* workload should move and why.

### 7. The bucket-boundary audit

For each SLO threshold you chose, check it against the bucket list in §11. **Acceptance:** a one-line note per SLO saying either "threshold is a bucket boundary — ratio is exact" or "threshold falls inside bucket [a, b) — moved to `a`/`b`, or switched to native histograms."

### 8. Capture the exhibit

While the ramp runs, screenshot `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` for the same GPU alongside TTFT-p99 and TPOT-p99. **Acceptance:** a single image showing `GPU_UTIL` pinned near 100 across the entire ramp while tensor activity climbs from ~0.04 to ~0.19 and the two honest latencies tell the real story. This is the inference half of the deliverable's util-lie exhibit — and unlike the idle-GPU version, it shows the lie on a GPU that is *genuinely working*.

**Overall acceptance:** (1) a TTFT-p99 panel, (2) a TPOT-p99 panel with ITL-p99 overlaid, (3) the saturation overlay, (4) working burn-rate alerts on both SLOs, and (5) a written observation identifying the inflection concurrency where TTFT knees upward, queue depth rises, and preemptions begin — with an explicit argument for why `e2e_request_latency_seconds` p99 is *on the dashboard but not an SLO*.

## Common pitfalls

1. **Building panels on `vllm:time_per_output_token_seconds` or `vllm:gpu_cache_usage_perc`.** Both were renamed (`inter_token_latency_seconds` / `kv_cache_usage_perc`) and the old names simply return no data. *Mechanism:* Prometheus does not error on a non-existent metric — it returns an empty vector, which renders as a blank panel that looks exactly like "nothing is wrong." Grep `/metrics` before you build, and pin the engine version in the dashboard.

2. **Setting an SLO threshold that is not a bucket boundary.** A 30 ms ITL budget sits inside vLLM's 25–50 ms bucket, so every compliance number you report is a linear interpolation across a 25 ms range on a distribution that is not uniform in it. *Mechanism:* §11. Move the threshold, change the buckets, or move to native histograms (Prometheus ≥ 3.8).

3. **Averaging per-replica quantiles.** `avg(histogram_quantile(...))` is not the fleet p99 and can be off by a lot when replicas are unevenly loaded. Always `sum by (le, …) (rate(..._bucket[…]))` *then* `histogram_quantile`.

4. **Treating `max_num_batched_tokens` as a throughput knob.** It is the TTFT↔ITL dial: smaller favours ITL, larger favours TTFT. Moving it in one direction without watching both SLOs silently blows the other.

5. **Reading a TTFT regression as pure capacity without checking preemptions.** V1's default preemption mode is `RECOMPUTE`, so a preempted request pays a *second full prefill*. A TTFT spike whose real cause is KV exhaustion looks like a queueing problem and will be "fixed" by adding replicas when the cheaper fix was `gpu_memory_utilization` or a smaller `max_num_seqs`.

6. **Confusing ITL and TPOT, or SLOing on the wrong one.** ITL is token-weighted (long responses dominate); TPOT is request-weighted (every user counts once). A user-facing promise is request-weighted. Using ITL-p99 as the promise means a handful of very long responses set the number your entire user base is judged by.

7. **Optimising raw tokens/s instead of goodput.** The same lever that raises throughput raises SLO violations. §6's arithmetic shows a configuration winning the throughput dashboard by 31% while losing 39% on cost-per-useful-token.

8. **Alerting on e2e latency "because that's what users complain about."** Users complain about *feel* — slow to start, or halting mid-stream — which is TTFT and ITL. A blended alert fires on verbose prompts and stays silent on real regressions buried in short ones.

9. **Copying perception budgets verbatim.** §9's numbers are 2026-era anchors, not a spec. Voice, chat and batch have genuinely different ceilings, and the only correct source for your budget is your own product's users.

10. **SLOing server-side TTFT and calling it the user contract.** It excludes network, TLS, ingress, router and cold starts. If your model takes minutes to load and you scale on queue depth, every scale-out is a client-side TTFT incident that is invisible in every engine metric you have.

11. **Ignoring the `engine` label.** V1 adds it. A multi-engine deployment queried without `sum by (le, model_name)` gives one series per engine — a panel that looks like noise, or worse, a p99 that silently tracks only one engine.

## Self-check

- **Continuous batching improves which metric and degrades which, and why?**
  **Answer:** It improves **throughput** (and therefore $/token) by keeping the GPU's units fed with many concurrent sequences; it degrades individual **TTFT and TPOT/ITL**. The mechanism is the roofline: a decode step's cost is dominated by reading the model weights from HBM (≈4.8 ms for an 8B bf16 model on an H100 at 3.35 TB/s), a cost paid *once per step regardless of batch size*, while compute scales with B. So going from B=1 to B=32 adds ~0.5 ms of compute to a ~4.8 ms step — 32× the tokens for ~11% more time. Each user's next token comes marginally later (TPOT up); a large in-flight prefill sharing the iteration can stall it much more (ITL spike, TTFT spike for newcomers). It is a throughput-vs-per-user-latency dial, and it stays extremely favourable until batch size approaches the machine's ridge point (~296 FLOP/byte on H100, i.e. B ≈ 296).

- **Why is total request-latency p99 the wrong SLO for a streaming LLM endpoint?**
  **Answer:** Because `e2e ≈ TTFT + (M − 1) × TPOT`, and for any M beyond ~20 tokens the `M × TPOT` term dominates — with M chosen by the caller and the model's stopping behaviour, not by your service. Concretely, at a healthy TTFT of 120 ms and TPOT of 30 ms, a 40-token reply takes 1.3 s and a 2,000-token reply takes 60 s: a 47× spread with identical service health. So p99(e2e) is mostly p99(output length); a traffic-mix shift moves your SLO with zero degradation, a real TPOT regression gets smeared across a length-dominated distribution, and "p99 latency is high" gives on-call no lever. SLO on TTFT-p99 (responsiveness) and TPOT-p99 (smoothness) separately. If a contract needs a whole-request bound, use a length-normalised compliance ratio: `e2e ≤ TTFT_budget + M × TPOT_budget`, aggregated as a fraction of compliant requests.

- **Queue depth is rising while TPOT stays flat. What does that tell you, and what do you check next?**
  **Answer:** The bottleneck is **admission/capacity, not decode**. Requests are waiting to enter the running batch (`vllm:num_requests_waiting` climbing), which drives TTFT up; once admitted, per-token cadence is unaffected, so TPOT is flat. Check three things in order: (1) `vllm:num_requests_waiting_by_reason` — `capacity` means genuine KV/token-budget pressure, `deferred` means a transient constraint like LoRA budget or KV transfer, and the fixes are different; (2) `vllm:kv_cache_usage_perc` — near 1.0 confirms the KV ceiling; (3) `rate(vllm:num_preemptions_total[5m])` — non-zero means requests are being evicted and will pay a full prefill recomputation (V1's `RECOMPUTE` mode), which is also why ITL-p99 will have diverged upward from TPOT-p99. Fixes are capacity/admission-side: lower `max_num_seqs`, raise `gpu_memory_utilization` if there is headroom, add a replica, or shed — not a decode-side change.

- **Prove that decode is memory-bandwidth-bound, with numbers.**
  **Answer:** A forward pass over T tokens costs ≈ `2 × P × T` FLOPs and requires reading ≈ `2 × P` bytes of bf16 weights, so its **arithmetic intensity is just T FLOP/byte**. Decode produces one token per sequence per step, so T = batch size B; at B=1 the intensity is ~1 FLOP/byte. A GPU's ridge point — peak FLOP/s ÷ peak HBM bytes/s — is about **296 FLOP/byte on an H100 SXM** (~990 TFLOP/s dense bf16 ÷ 3.35 TB/s), ~153 on an A100 80GB, ~210 on an L40S. Batch-1 decode is therefore roughly **300× below the ridge**: it can reach about 1/300th of peak FLOP/s no matter what. Concretely for Llama-3.1-8B (16 GB of bf16 weights) on an H100: 4.78 ms to read the weights versus 0.016 ms of arithmetic per step. Prefill of a 2,000-token prompt has intensity ~2,000 FLOP/byte, far *above* the ridge — compute-bound. Same chip, opposite regimes, which is why one SLO cannot cover both and why `GPU_UTIL` reads ~100% during decode while `PIPE_TENSOR_ACTIVE` sits near 0.04.

- **Per vLLM's own optimization docs, does a smaller or larger `max_num_batched_tokens` favour ITL, and what is the default scheduling behaviour around it?**
  **Answer:** A **smaller** value (e.g. 2048) favours **ITL**, because fewer prefill tokens are admitted per iteration so fewer prefills compete with and stall decode steps. A **larger** value favours **TTFT**, because more prefill tokens are processed per iteration so prompts reach their first token sooner; vLLM recommends `> 8192` when maximising throughput, especially for smaller models on large GPUs. It is a direct tradeoff on one dial, not two independent knobs. The default behaviour matters too: **chunked prefill is enabled by default whenever possible in V1, and the scheduler prioritises decode requests**, admitting prefill only when the budget allows — so the system already trades TTFT away to protect ITL, and a TTFT-only regression under load may be the scheduler working as designed rather than a capacity failure.

- **You want a 99% / 30 ms ITL SLO. What is wrong with that, and what do you do?**
  **Answer:** 30 ms is not a bucket boundary. vLLM's inter-token-latency histogram steps `0.01, 0.025, 0.05, 0.075, …`, so 30 ms sits 20% into a **25 ms-wide** bucket, and any compliance figure you compute is a linear interpolation that assumes observations are uniformly distributed inside `[0.025, 0.05)` — which latency distributions never are, so the number will be systematically off. The exact quantity a histogram can give you is a cumulative bucket count: `…_bucket{le="0.025"}` or `{le="0.05"}` divided by `…_count`. Three options: (a) move the budget to 25 ms (tighter and exact) or 50 ms (looser and exact); (b) configure custom buckets that include 0.03; (c) switch to **Prometheus native histograms** — stable since v3.8.0 and carried through the v3.13.0 LTS — whose exponential bucketing gives dense resolution everywhere and makes an arbitrary threshold measurable.

- **Work out the error budget for a 99% / 500 ms TTFT SLO on 20M requests per 30 days, and derive the fast-burn alert threshold.**
  **Answer:** Budget = `1 − 0.99 = 1%` of 20M = **200,000 bad requests per 30 days**, i.e. ~278/hour sustained at exactly the SLO. Burn rate = observed bad-event ratio ÷ (1 − SLO). Pick "I want to know if 2% of the month's budget burns in 1 hour": burn rate = `0.02 ÷ (1/720) = 14.4`, so the threshold on the observed bad ratio is `14.4 × 0.01 = 14.4%`. The 6-hour/5% page is `0.05 ÷ (6/720) = 6` → 6%, and the 3-day/10% ticket is burn rate 1 → 1%. Pair each long window with a short window at 1/12th (5 m with 1 h, 30 m with 6 h) so the alert resolves promptly. Compute the SLI as an **exact ratio** off the `le="0.5"` bucket — `1 − (rate(..._bucket{le="0.5"}[1h]) / rate(..._count[1h]))` — not as a quantile, because quantiles cannot be aggregated or turned into budgets.

- **What did prefill–decode disaggregation buy Anyscale's benchmark, and what is the mechanism?**
  **Answer:** Up to **2.7× better goodput and roughly 67% lower compute cost** (AMD MI325X, 2026 pricing — a dated, hardware-specific snapshot), plus lower TTFT under load (355 ms vs 389 ms prefill-heavy, 165 ms vs 190 ms decode-heavy at concurrency 256) versus a co-located comparison router. **Mechanism:** prefill (compute-bound, intensity ≈ prompt length) and decode (memory-bound, intensity ≈ batch size) run on separate GPU pools, with KV cache transferred between them. A large incoming prompt's compute-bound work no longer shares an iteration budget with other users' memory-bound decode steps, so the coupling that causes TTFT spikes and ITL jitter is *removed* rather than merely bounded by chunked prefill. The cost is two GPU pools plus a KV-transfer path — the trade is a $/goodput calculation, not a free win.

## Connections & what's next

This lesson is the module's second demonstration of the 05.1 thesis at a different layer, and the first one that *derives* it: a decode-bound LLM server pins `GPU_UTIL` near 100% while `PIPE_TENSOR_ACTIVE` sits at 0.04 because a batch-1 decode step's arithmetic intensity is ~300× below the machine's ridge point — which is why TTFT and TPOT, not utilisation percentages, track whether users are being served well.

It builds on 05.5's health gate (a wedged or reduced-capacity GPU corrupts utilisation and latency simultaneously, and a fault on an NVLink partner — XID 74 — shows up as a TPOT regression on a tensor-parallel deployment). It builds on 05.3's cardinality discipline: `vllm:num_requests_waiting_by_reason` and prefix-cache counters add label dimensions, and the `engine` label is new in V1.

It feeds forward into 05.7: when TTFT or TPOT breaches its budget and the cause is *not* queueing or KV pressure, profiling is the next rung — and you now hand the profiler a precise, metric-driven trigger ("TPOT regressed, decode-side, queue flat, preemptions zero") instead of a shrug. And it feeds the capstone: serving namespaces get an honest efficiency number — goodput per dollar — that sits alongside the allocated-vs-utilised GPU-hours gap and explains *why* an inference namespace can hold GPUs at 99% `GPU_UTIL` and still be the biggest waste line on the dashboard.

## References & further reading

**Primary sources**

1. **vLLM — `vllm/v1/metrics/loggers.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/metrics/loggers.py — the definitive list of exposed metric names, types, docstrings, label sets, and **exact histogram bucket boundaries**. This is where §7's table and §11's bucket lists come from. **Correction to earlier versions of this lesson:** `vllm:time_per_output_token_seconds` no longer exists; it split into `vllm:inter_token_latency_seconds` (ITL) and `vllm:request_time_per_output_token_seconds` (TPOT), and `vllm:gpu_cache_usage_perc` became `vllm:kv_cache_usage_perc`.
2. **vLLM — `docs/design/metrics.md`** — https://github.com/vllm-project/vllm/blob/main/docs/design/metrics.md — how each interval is derived from QUEUED/SCHEDULED/PREEMPTED/NEW_TOKENS engine events on a monotonic clock, the documented distortion preemption introduces into each interval, and the deprecated/removed metric table.
3. **vLLM — `docs/configuration/optimization.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/optimization.md — chunked prefill on by default in V1, decode prioritised in scheduling, the `max_num_batched_tokens` TTFT/ITL tradeoff with the `> 8192` throughput recommendation, the four KV-preemption remedies, and V1's `RECOMPUTE` default preemption mode.
4. **vLLM — production metrics guide** — https://docs.vllm.ai/en/latest/design/metrics/ — the rendered version of (2), plus the reference Grafana dashboard.
5. **Hugging Face — Text Generation Inference, `router/src/server.rs`** — https://github.com/huggingface/text-generation-inference/blob/main/router/src/server.rs — the `tgi_*` metric names, types and units in §7, including the method-labelled `tgi_batch_forward_duration` / `tgi_batch_decode_duration` histograms and the `tgi_request_failure{err="overloaded"}` shed signal.
6. **SGLang — Prometheus metrics reference** — https://docs.sglang.io/ — the `sglang:` metric family (`time_to_first_token_seconds`, `inter_token_latency_seconds`, `num_running_reqs`, `num_queue_reqs`, `token_usage`, `cache_hit_rate`) and the `--enable-metrics` flag.
7. **Prometheus — Native histograms specification** — https://prometheus.io/docs/specs/native_histograms/ — stable as of v3.8.0, carried through the v3.13.0 LTS (1 July 2026); exponential bucketing and its interpolation model.
8. **Prometheus — Histograms and summaries (best practices)** — https://prometheus.io/docs/practices/histograms/ — why `histogram_quantile` interpolates linearly inside classic buckets, why that misleads when observations cluster at a bucket edge, and why you aggregate buckets before quantiling.
9. **Prometheus — Query functions** — https://prometheus.io/docs/prometheus/latest/querying/functions/ — `histogram_quantile`, `rate`, and the `le` label semantics behind every query in §10 and §11.
10. **BentoML — "Key metrics for LLM inference"** — https://bentoml.com/llm/llm-inference-basics/llm-inference-metrics — vendor-neutral definitions of TTFT (queueing + prefill + network), ITL (token-weighted individual gaps) and TPOT (mean of one request's ITLs).

**Real-world engineering**

11. **Anyscale — "Achieving Up to 67% Cost Savings with Prefill-Decode Disaggregation Using Ray + vLLM on AMD MI325X"** — https://www.anyscale.com/blog/ray-vllm-prefill-decode-disaggregation-amd-mi325x-67-percent-savings — 2.7× goodput, ~67% cost saving, and TTFT-under-load numbers from physically separating the two phases.
12. **Cloudflare — "How we built the most efficient inference engine for Cloudflare's network"** — https://blog.cloudflare.com/cloudflares-most-efficient-ai-inference-engine/ — Infire's prefill/decode separation, p90 ITL from ~100 ms to 20–30 ms, and sub-20-second cold starts as an explicit engineering target.

**Deeper dives**

13. **Ray — "Prefill/decode disaggregation" user guide** — https://docs.ray.io/en/latest/serve/llm/user-guides/prefill-decode.html — the deployable configuration behind the Anyscale benchmark.
14. **Spheron — "LLM Inference SLO Engineering: TTFT, ITL, and P99 Latency Budgets for Production AI (2026)"** — https://www.spheron.network/blog/llm-inference-slo-ttft-itl-latency-budget-guide-2026/ — the perception-grounded TTFT/ITL budget anchors by workload used in §9. Treat as dated, directional calibration, not a spec.
