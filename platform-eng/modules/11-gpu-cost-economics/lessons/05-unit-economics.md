---
lesson: 05
title: "Unit economics: joining infra dollars to application counters"
module: 11
concept: "infra-$ to business-unit join"
status: not-started
est_time: "6 hrs"
prev: "04-fragmentation-cost.md"
next: "06-commitment-strategy.md"
artifacts: ["a $/1M-token calculator that joins attributed GPU-hours × blended rate ÷ app-emitted token counts, emitting both direct and fully-loaded numbers"]
sources: 14
---

# Unit economics: joining infra dollars to application counters

[💰 11 — GPU cost and unit economics](../README.md) · ← [04 Fragmentation: unschedulable GPUs](04-fragmentation-cost.md) · → [06 Commitment & procurement strategy](06-commitment-strategy.md)

## Where this fits

Lessons 01–04 built one number and one ledger of what is wrong with it. Lesson 01 established which GPU-hours can be attributed exactly and which carry an error band. Lesson 02 split the result into the allocated and utilised ledgers and fixed the rate basis at FOCUS `EffectiveCost`. Lesson 03 split the gap between those ledgers into states with owners, and showed that most of it is not reclaimable. Lesson 04 priced the capacity nobody can schedule at all.

This lesson performs the division. Take attributed GPU-hours, multiply by a rate, divide by something the business counts — tokens, runs, experiments, customers — and you have a **unit cost**. That single operation is what turns an infrastructure artifact into a business one, and it is the most-probed competency in a GPU-platform interview because it is where FinOps meets the P&L.

The formula is trivial. Everything interesting is in three places: **what goes in the numerator** (which ledger, which rate, loaded with what), **what goes in the denominator** (which counter, over which window, counting what), and **what physically determines the ratio** — because a unit cost is not an accounting artifact, it is a hardware number wearing a dollar sign. Most of this lesson is spent deriving the denominator from memory bandwidth and FLOPs so you can predict a unit cost before you have measured it, and diagnose one that is worse than it should be.

Lesson 06 opens the one input treated as given here: where the blended rate comes from.

## Why this matters

If you cannot state your product's cost in dollars per million tokens **and defend the denominator and the loading**, you do not know your gross margin — you know your cloud bill, which is a different and much less useful number.

The naive answer — monthly GPU spend divided by monthly tokens — fails three follow-ups in a row. It has no per-tenant attribution, so it is a fleet average masquerading as a product cost. It uses whatever rate happened to be on the invoice rather than a stated basis, which on a committed fleet is wrong by tens of percent. And it silently either includes or excludes the idle and fragmentation tax without saying which, so the number is neither a direct cost nor a loaded one and cannot be compared to anything.

The stakes are that this number sets prices, decides build-versus-rent, and decides which workloads get killed in a capacity crunch. Quote a *direct* cost in a pricing meeting and you set a price that is underwater once overhead lands; quote a *fully-loaded* cost in an engineering review and a genuine batching win disappears under fleet-wide waste the team never controlled. Both mistakes are made constantly, and both are avoided by one discipline: **state the basis every time you say the number out loud.**

And there is a reason to derive rather than merely measure. Cost per token is falling fast — Character.AI reports reducing serving cost by roughly **33×** against its late-2022 baseline through KV-cache reduction of more than 20× and native int8 training and inference — so any snapshot ages within months. What does not age is the factor tree: which quantity moved, by how much, and what it multiplies. A team that can say "our decode throughput is 38% of the memory-bandwidth roofline, and closing half that gap takes $/1M tokens from $0.67 to $0.51" is doing engineering. A team that can only say "our cost went up" is doing accounting.

## What's new here (calibration)

- **Already yours (skip):** cost-per-token as a serving metric (module 07); cost-per-run and MFU as a training metric (module 08); per-pod GPU attribution (module 04); the `SM_ACTIVE` versus `GPU_UTIL` distinction and the `sum_over_time × Δ/3600` integral (module 05); the two ledgers and `EffectiveCost` (lesson 02).
- **New angle 1 — one operator, many units.** `$/token`, `$/run`, `$/experiment` and `$/customer` are the same division with different denominators, so the skill is the *join*, not either endpoint. §1 states it with units carried and §8 synthesises a new unit on demand.
- **New angle 2 — the factor tree, derived from hardware.** §3 predicts tokens per GPU-second from memory bandwidth, model bytes, KV-cache geometry and batch size, and §4 does the same for prefill from peak FLOPs. That turns unit cost from something you observe into something you can *forecast and diagnose*, and it explains why input tokens cost roughly a fifth of output tokens.
- **New angle 3 — the loading multiplier, built from this module's own buckets, with the double-count corrected.** The common formula `L = 1 / utilisation` is wrong under this module's charge-allocated policy, because a tenant's own idle is already inside its allocated hours. §6 derives the correct loading from gap A plus non-GPU overhead, shows what `paid ÷ utilised` actually means, and notes that OpenCost's weighted idle sharing implements exactly the cost-proportional split this requires.
- **New angle 4 — the denominator is a choice with consequences.** Prefill versus decode, cached prompt tokens that you never computed, preempted and restarted work, failed runs. §5 and §8 make each of those an explicit, defensible decision instead of an accident of which metric was handy.
- **New angle 5 — sanity-checking against published prices.** A market price is a price, not a cost, but it bounds the plausible range, and §9 does the comparison properly.

## Core concepts

### 1. The identity, with units carried

```
                    H_attr [GPU-hours]  ×  r [$ / GPU-hour]
  unit_cost [$/U]  = ───────────────────────────────────────
                              U [units]

  H_attr   attributed GPU-hours for the workload over window W
           (lesson 01's regime decides how exact this is;
            lesson 02 decides which LEDGER it comes from)
  r        $ per PHYSICAL GPU-hour on a stated basis and date
           (lesson 02: FOCUS EffectiveCost; lesson 06: the blend)
  U        units produced by that workload over THE SAME window W
           (tokens, requests, runs, samples, customers)

  DIMENSIONAL CHECK: GPU-h × $/GPU-h ÷ units = $ / unit.  ✓
  For tokens, report per MILLION: multiply by 1e6 and say so.
```

The whole discipline is that **all three terms must describe the same workload over the same window.** Three alignment failures account for most wrong unit costs in the wild:

1. **Window skew.** GPU-hours on a UTC billing-day boundary joined to tokens from a dashboard on local time is a several-hour offset. On a workload with a diurnal profile that is a double-digit percentage error in a number that still looks plausible.
2. **Identity skew.** GPU-hours by namespace joined to tokens by model name, when one namespace serves three models. The ratio is then an average over a mix you did not intend to average over, and it moves whenever the mix moves — which looks exactly like a cost regression.
3. **Ledger skew.** `H_attr` from the *allocated* ledger and a throughput figure measured only while the service was under load. The denominator then covers a shorter window than the numerator, and the unit cost is understated by the idle fraction.

Pin one window (an hour or a day), one workload key (namespace × model), and pull all three terms from that key. Everything else in this lesson assumes that has been done.

**Which ledger goes in the numerator?** Allocated, in almost every case. It is what you pay regardless of activity (lesson 02), it is exact in every sharing regime, and it is the number a tenant is billed. Using the *utilised* ledger produces a figure that answers a different and rarely useful question — "what would this have cost if the GPU were only paid for while computing" — and that GPU does not exist.

### 2. The factor tree: what a unit cost is actually made of

The identity has three terms; each decomposes. Drawing the full tree once is what lets you answer "why is our cost per token 3× the competition's" with a factor rather than a shrug.

```
                        $ per 1,000,000 OUTPUT TOKENS
                                     │
        ┌────────────────────────────┴─────────────────────────────┐
        │                                                          │
   NUMERATOR: $ per GPU-hour                    DENOMINATOR: tokens per GPU-hour
        │                                                          │
   ┌────┴──────┬─────────────┐                     tokens/GPU-hour = 3600 × tokens/GPU-s
   │           │             │                                      │
 r_list   commitment      loading L                 ┌───────────────┴────────────────┐
 (sticker) mix (L06)      (§6)                      │                                │
   │           │             │                  DECODE (memory-bound)      PREFILL (compute-bound)
   │      ┌────┴────┐   ┌────┴─────┐                │                                │
   │   on-demand  committed   gap A share      tokens/s = B × BW_eff        tokens/s = MFU × P_peak
   │      │        │  spot     (fragmentation,            ─────────────────           ─────────────
   │      │        │           cordoned, L04)             W_bytes + KV_bytes                2N
   │      │        │                │                        │        │                  │     │
   │      │        │           non-GPU overhead              │        │                  │     │
   │      │        │           (control plane,               │        │                  │     │
   │      │        │            storage, network)            │        │                  │     │
   │      │        │                                         │        │                  │     │
   │      │        └── term, coverage, flexibility           │        │                  │     │
   │      └─────────── spot interruption + goodput           │        │                  │     │
   └────────────────── vendor / region / tier (L07)          │        │                  │     │
                                                             │        │                  │     │
        ┌────────────────────────────────────────────────────┘        │                  │     │
        │                                                             │                  │     │
   BW_eff = MBU × BW_peak                                   KV_bytes = B × L_ctx ×        │     │
     │        │        │                                       2·n_kv·d_head·L·bytes      │     │
     │        │        └── HBM generation (H100 3.35 TB/s,     │      │        │           │     │
     │        │            H200 4.8 TB/s) — a HARDWARE term    │      │        │           │     │
     │        └─────────── kernel quality, graph capture,      │      │        └── GQA/MQA │     │
     │                     paged attention, scheduling         │      │            heads   │     │
     │                                                         │      └── context length   │     │
     └── W_bytes = params × bytes/param                        └── prefix-cache HIT RATE   │     │
             │           │                                         (tokens you BILL but    │     │
             │           └── QUANTISATION: fp16 2 B, fp8 1 B,       DO NOT COMPUTE)        │     │
             │               int4 0.5 B — a DIRECT divisor                                 │     │
             │                                                                             │     │
             └── model size N ──────────────────────────────────────────────────────── 2N ─┘     │
                                                                                                 │
                                            P_peak = per-GPU peak FLOP/s at the serving precision┘
                                            (H100 SXM: ~989 TFLOP/s dense BF16,
                                             ~1,979 TFLOP/s dense FP8 — datasheet
                                             figures are quoted WITH 2:4 sparsity,
                                             i.e. 2× these; use the dense ones)

  ── READ THE TREE AS A DIAGNOSTIC ────────────────────────────────────────
  Every lever anyone will propose lands on exactly one leaf:
    quantise fp16→fp8        halves W_bytes            → decode tokens/s up
    MQA/GQA instead of MHA   cuts KV_bytes 8×          → bigger B fits
    bigger batch B           amortises W_bytes         → tokens/s up, TPOT up
    prefix caching           cuts tokens COMPUTED      → same U, less H_attr
    speculative decoding     >1 token per step         → tokens/s up
    newer HBM generation     raises BW_peak            → tokens/s up
    commitment coverage      lowers r                  → numerator down
    fixing fragmentation     lowers L                  → numerator down
  ⇒ IF A PROPOSAL DOESN'T MAP TO A LEAF, IT DOESN'T MOVE UNIT COST.
```

### 3. Deriving the denominator: decode is a memory-bandwidth problem

The single most useful thing in this lesson is that you can compute the denominator from first principles, before writing any code, and be right to within the efficiency factor.

**The mechanism.** Autoregressive decoding generates one token per sequence per forward pass. In that pass every model weight must be read from HBM at least once — and, crucially, **the same read serves every sequence in the batch**, because they all multiply against the same weights. The other mandatory read is the KV cache: each sequence's stored keys and values for its whole context. So per decode step:

```
  bytes_per_step  =  W_bytes  +  B × L_ctx × kv_bytes_per_token

  W_bytes             = N_params × bytes_per_param
  kv_bytes_per_token  = 2 × n_kv_heads × d_head × n_layers × bytes_per_element
                        (the 2 is K and V)

  steps_per_second    =  BW_eff / bytes_per_step        BW_eff = MBU × BW_peak
  tokens_per_second   =  B × steps_per_second           (one token per sequence)
  time_per_output_tok =  1 / steps_per_second           ( = TPOT, the SLO)
```

`MBU` — memory-bandwidth utilisation — is the achieved fraction of peak HBM bandwidth, the decode-side analogue of MFU. It is measurable and typically lands well below 1 because of attention kernel efficiency, scheduling gaps and non-weight traffic. **Measure yours; do not assume the number below.**

Work it end to end on a concrete, checkable configuration:

```
  MODEL     70B parameters, served in FP8            W_bytes = 70 GB
            80 layers, 8 KV heads (GQA), d_head 128, KV in FP8
            kv_bytes_per_token = 2 × 8 × 128 × 80 × 1 B = 163,840 B
                               ≈ 0.164 MB per token of context
  HARDWARE  2 × H100 SXM 80GB, tensor-parallel
            BW_peak = 2 × 3.35 TB/s = 6.70e12 B/s   (datasheet)
  ASSUMED   MBU = 0.65                (MEASURE THIS — it is the one
                                       squishy input, and it is linear)
  TRAFFIC   B = 64 concurrent sequences, mean context L_ctx = 4,096

  ── STEP 1: the byte budget per decode step ──────────────────────────
    weights                                    70.00 GB
    KV cache  64 × 4,096 × 163,840 B =         42.95 GB
                                              ─────────
    bytes_per_step                            112.95 GB

  ── STEP 2: steps and tokens per second ──────────────────────────────
    BW_eff   = 0.65 × 6.70e12                = 4.355e12 B/s
    steps/s  = 4.355e12 / 1.1295e11          = 38.6 /s
    TPOT     = 1/38.6                        = 25.9 ms  ← check vs SLO
    tokens/s = 64 × 38.6                     = 2,468 /s   (system)
                                             = 1,234 /s   (per GPU)

  ── STEP 3: the money ────────────────────────────────────────────────
    GPU-hours per hour of service            = 2 GPU-h
    tokens per hour  = 2,468 × 3,600         = 8.885e6

    $/1M output tokens = (2 GPU-h × $2.99) / 8.885e6 × 1e6
                       = $5.98 / 8.885     = $0.673 / 1M tokens

    (r = $2.99/GPU-h, FOCUS EffectiveCost basis, 2026-08 snapshot,
     within an observed on-demand H100 SXM band of roughly $2–$7.
     EVERY FIGURE IS LINEAR IN r.)
```

**Now vary the one term operators actually control day to day.** Batch size does not change `W_bytes` at all, so it amortises the dominant read across more tokens — until KV traffic catches up with it:

```
   B     KV bytes/step   total/step   steps/s   tokens/s   $/1M tokens
  ─────────────────────────────────────────────────────────────────────
    1        0.67 GB       70.67 GB     61.6        62       $26.80
    8        5.37 GB       75.37 GB     57.8       462       $ 3.60
   16       10.74 GB       80.74 GB     53.9       863       $ 1.92
   32       21.47 GB       91.47 GB     47.6      1,523      $ 1.09
   64       42.95 GB      112.95 GB     38.6      2,468      $ 0.673
  128       85.90 GB      155.90 GB     27.9      3,576      $ 0.464
  ─────────────────────────────────────────────────────────────────────
  ⇒ 40× cheaper per token at B=64 than at B=1, from ONE parameter.
    And the returns bend: 1→8 is 7.4×, 8→64 is 5.4×, 64→128 is 1.45×,
    because KV traffic grows LINEARLY in B while the weight read is
    fixed. Once KV bytes dominate, batching stops paying.
    Meanwhile TPOT rises 16.2 ms → 35.8 ms across that range, so the
    batch ceiling is set by your LATENCY SLO, not by economics.

  THIS TABLE IS THE ENTIRE ARGUMENT for continuous batching, and the
  entire reason an under-loaded replica is expensive per token even
  though it is cheap per hour.
```

Draw where the time goes, because the shape of the step is what makes the batching argument obvious:

```
  ONE DECODE STEP'S HBM TRAFFIC — the bar is time, and time is bytes.
  ▓ = weight read (FIXED, shared by the whole batch)
  ░ = KV read     (GROWS linearly with B × context)

  B=1    ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░           1 token out
         └────────────── 70.0 GB ───────────────┘└0.67┘      → 70.7 GB/token

  B=16   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░      16 tokens out
         └────────────── 70.0 GB ───────────────┘└10.7┘      → 5.05 GB/token

  B=64   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░
         └────────────── 70.0 GB ───────────────┘└──── 42.9 GB ────┘
                                                             64 tokens out
                                                             → 1.77 GB/token

  ⇒ THE WEIGHT READ IS A FIXED COST PER STEP AND FREE PER EXTRA SEQUENCE.
    Cost per token falls as 1/B until the ░ region dominates, then flattens.
    An idle-ish replica at B=1 is not "cheap" — it is 40× the unit cost
    of the same hardware at B=64. THIS is why lesson 03 insists a quiet
    serving GPU is a THROUGHPUT problem, not an idle-reclaim problem.
```

### 4. Prefill is a different machine, and that is why input tokens are cheaper

Prefill processes the whole prompt in parallel, so it is limited by arithmetic rather than by bandwidth. The standard accounting is **2 FLOPs per parameter per token** for a forward pass (one multiply and one add per weight):

```
  prefill_tokens_per_second  =  MFU × P_peak / (2 × N_params)

  Same rig: 2 × H100 SXM, FP8 dense P_peak ≈ 1,979 TFLOP/s per GPU
  (datasheet figures are stated WITH 2:4 structured sparsity —
   3,958 TFLOPS for FP8 on H100 SXM — so halve them for dense work),
  assume MFU = 0.45 for prefill:

    P_eff       = 2 × 1.979e15 × 0.45          = 1.781e15 FLOP/s
    tokens/s    = 1.781e15 / (2 × 70e9)        = 12,721 /s
    $/1M prompt tokens = (2 × $2.99)/(12,721 × 3,600) × 1e6
                       = $0.131 / 1M

  ── THE ASYMMETRY, AND WHAT IT EXPLAINS ──────────────────────────────
    decode  $0.673 / 1M output tokens   (B = 64)
    prefill $0.131 / 1M input  tokens
    ratio   5.1×

    Published API price sheets carry a very similar ratio — Anthropic's
    first-party list prices as of 2026-06-24 are $3/1M in versus $15/1M
    out for Claude Sonnet 5, and $5 versus $25 for Claude Opus 5: 5×
    in both cases. THAT RATIO IS NOT A PRICING CONVENTION, IT IS THE
    HARDWARE. Prompt tokens are processed in a compute-bound parallel
    pass; output tokens each require a full bandwidth-bound pass over
    the weights.

  ⇒ CONSEQUENCE FOR YOUR DENOMINATOR: pricing on output tokens only
    (which matches most price sheets) hides the prefill cost inside the
    output number, and a workload whose prompt-to-output ratio shifts
    will look like it regressed. Either weight the two, or state
    "output tokens only" and publish the prompt:output ratio beside it.
```

**Chunked prefill and disaggregation change where these costs land, not what they are.** Interleaving prefill chunks into decode batches (or running prefill and decode on separate replica pools) moves work between the two regimes to protect TPOT; the FLOPs and the bytes are unchanged. What *does* change the arithmetic is **prefix caching**: a cache hit means those prompt tokens are billed to the customer and never computed by you. vLLM exposes `vllm:prefix_cache_queries` and `vllm:prefix_cache_hits`, plus `vllm:prompt_tokens_cached`, precisely so you can measure it. That is a wedge between the tokens in your denominator and the work in your numerator, and it is the single most profitable one in serving: **at a 60% prefix-cache hit rate your effective prefill cost per billed input token falls by 60%, with no hardware change at all.**

### 5. The denominator in practice: which counter, counting what

`U` comes from the application, and the application has opinions. From vLLM's current metrics (`vllm/v1/metrics/loggers.py`), the ones that matter for unit economics:

| Metric | Type | What it counts | Unit-cost role |
|---|---|---|---|
| `vllm:generation_tokens_total` | counter | output tokens produced | the standard denominator |
| `vllm:prompt_tokens_total` | counter | prefill tokens processed | the input side of a weighted unit |
| `vllm:prompt_tokens_cached` | counter | prompt tokens served from cache (local + external) | billed-but-not-computed — the wedge in §4 |
| `vllm:prefix_cache_queries` / `_hits` | counters | cache lookups and hits | the hit rate that drives that wedge |
| `vllm:num_requests_running` / `_waiting` | gauges | in-flight and queued requests | the batch size `B` in §3, and lesson 03's idle attestation |
| `vllm:num_preemptions` | counter | requests preempted by the engine | work done twice — inflates `H_attr` without raising `U` |
| `vllm:request_success` | counter (by finish reason) | completed requests | `$/request` denominators; separates `stop` from `length` |
| `vllm:kv_cache_usage_perc` | gauge | KV utilisation, 1.0 = full | why `B` stopped growing |
| `vllm:iteration_tokens_total` | histogram | tokens per engine step | measured tokens-per-step, i.e. §3's `B` in reality |

Note the naming detail that costs people an afternoon: these are defined as Prometheus `Counter`s named `vllm:generation_tokens`, and the client appends `_total` on exposition — query `vllm:generation_tokens_total`.

Four denominator decisions, each of which must be *stated*:

1. **Output-only, or prompt + output?** Output-only matches most price sheets and is the usual choice. Prompt + output matches the compute you actually did. Whichever you pick, publish the prompt:output ratio alongside, because the unit cost moves when the ratio moves and that is not a regression.
2. **Do cached prompt tokens count?** They are billed to the customer and cost you almost nothing. Counting them lowers your unit cost honestly; excluding them tells you what your *compute* costs. Both are legitimate; mixing them across periods is not.
3. **Do preempted-and-recomputed tokens count once or twice?** Once in `U` (the customer got one answer) and the recompute is already in `H_attr`. That is the correct asymmetry, and it is why `vllm:num_preemptions` belongs on the same dashboard: a rising preemption rate raises unit cost with no other visible symptom.
4. **What about failed requests?** They consumed GPU-hours and produced nothing. Excluding them from `U` is right and makes the failure rate a unit-cost lever — the serving analogue of `$/successful-run` in §8.

### 6. Direct versus fully-loaded, with the double-count removed

Some spend produces no units directly: the fleet's unallocatable and fragmented capacity (lesson 04), cordoned and unhealthy devices, control-plane nodes, fast storage, networking. That spend has to be recovered from the units you *do* produce, and the multiplier that does it is `L`.

The formula in wide circulation is `L = 1 / utilisation`. **Under this module's policy it is wrong, and it double counts.** Lesson 02 §6 established that you charge the *allocated* ledger: a tenant's own idle hours are already inside `H_attr` and already paid for. Loading the fleet's utilisation gap on top charges them for that idle a second time. Do it properly:

```
  ── DEFINE THE BUCKETS (all from lessons 02–04, one window) ───────────
    P   paid GPU-hours          (area ①: every GPU you rent or own)
    A   allocated GPU-hours     (area ②: Σ over tenants)
    Uz  utilised GPU-hours      (area ③)
    f   non-GPU overhead as a fraction of GPU spend
        (control plane, storage, network, egress, observability)

  ── THE LOADING MULTIPLIER, CORRECTLY ────────────────────────────────
    L = (P / A) × (1 + f)

    P/A   recovers gap A — capacity nobody could hold, which INCLUDES
          lesson 04's fragmentation, cordoned devices and MIG stranding.
          It is the platform's own inefficiency, not the tenant's.
    1+f   recovers non-GPU spend.
    The tenant's own gap B is ALREADY in A. Do not load it again.

  ── THE THREE NUMBERS AND WHAT EACH ANSWERS ──────────────────────────
    DIRECT           uses H_attr = the workload's allocated hours, no L.
                     ANSWERS: how efficient is this workload?
                     AUDIENCE: the team that owns it.

    FULLY LOADED     multiplies by L = (P/A)(1+f).
                     ANSWERS: what does a unit cost the company?
                     AUDIENCE: pricing, P&L, gross margin.

    COST PER
    PRODUCTIVE HOUR  P / Uz  — the whole fleet's cost divided by hours
                     that actually computed.
                     ANSWERS: what would we pay per hour of real work if
                     nothing were wasted anywhere?
                     AUDIENCE: nobody's invoice. It is a DIAGNOSTIC, and
                     quoting it as a unit cost overstates by (A/Uz),
                     which on a normal fleet is 2–3×.
```

Put the module's own fleet through it, so the number is traceable:

```
  From lesson 03's worked fleet (64 × H100, one week, r = $2.99):
    P  = 10,752 GPU-h     A = 8,736 GPU-h     Uz = 3,669 GPU-h
    f  = 0.09   (control plane + fast storage + networking, MEASURED
                 as a fraction of GPU spend — measure yours)

    P/A          = 10,752 / 8,736       = 1.2308
    L            = 1.2308 × 1.09        = 1.3416

    for contrast, the WRONG loadings people use:
      1/(Uz/A)   = 8,736 / 3,669        = 2.381   ← charges the tenant
                                                    for its OWN idle,
                                                    which it already
                                                    paid for in A
      P/Uz       = 10,752 / 3,669       = 2.931   ← the diagnostic, not
                                                    a chargeable loading

  ⇒ APPLIED TO §3's NUMBER:
      direct        $0.673 / 1M output tokens
      fully loaded  $0.673 × 1.3416   = $0.903 / 1M
      the tax                          = $0.230 / 1M  (34 %)
    versus the naive-loading answer $0.673 × 2.381 = $1.602 / 1M,
    which is 78 % too high AND is the number that makes a healthy
    service look unprofitable.
```

**The cost-proportional split is the standard one, and the reference implementation already does it.** OpenCost's `computeIdleCoeffs` distributes idle to each allocation in proportion to that allocation's `GPUTotalCost()` over the per-idle-key total, when `ShareIdle` is weighted — i.e. `L` applied uniformly across GPU spend, which is exactly the `(P/A)` factor above. Two alternatives exist and are worth knowing: split gap A **evenly** across tenants (penalises small tenants; rarely defensible), or **do not split it at all** and keep it as a platform cost centre (lesson 02's recommendation for chargeback). Use the unloaded number for tenant bills and the loaded number for pricing, and never quietly move between them.

**Rule of thumb: optimise on direct, price on fully loaded, and never quote one as the other.** The gap between them *is* the idle-and-fragmentation tax expressed per business unit, which is the most legible way to show an executive why lessons 03 and 04 are worth funding.

### 7. The rate: blended or marginal

`r` has two correct values and they answer different questions.

```
  BLENDED   r_b = Σ_i (share_i × rate_i)  over the commitment mix
            e.g. 70 % committed at $1.90 + 30 % on-demand at $3.20
                 r_b = 0.7(1.90) + 0.3(3.20) = $2.29 / GPU-h
            USE FOR: steady-state margin, pricing, published unit costs.
            It is what capacity costs you on average, and it is the
            basis a price sheet must clear.

  MARGINAL  r_m = the rate of the capacity that would serve ONE MORE
            unit. On a fleet with committed headroom sitting idle,
            r_m ≈ 0 (the capacity is sunk and otherwise wasted).
            Above the committed baseline, r_m = the on-demand or spot
            rate that tops it up.
            USE FOR: accept/reject decisions on incremental work —
            "should we take this overnight batch job", "does this
            customer's traffic clear its own cost tonight".

  ── THE TWO CLASSIC ERRORS ───────────────────────────────────────────
  Marginal used for pricing   → you price as though sunk capacity is
                                free forever, and the commitment renews
                                at a rate nobody budgeted for.
  Blended used for accept/    → you decline revenue-positive work that
  reject on idle capacity       would otherwise be pure waste.
```

Commitment coverage is the dial that sets `r_b`, and lesson 06 is where that dial is designed. For this lesson `r` is an input — but an input that must arrive with its **basis** (FOCUS `EffectiveCost`, not `BilledCost`, so a prepaid commitment is amortised into the hours it covers rather than spiking in the purchase month) and its **date**.

### 8. Other units from the same operator

**`$/run` and `$/successful-run`.** Same division, denominator is completed training runs:

```
  A run: 8 × H100 for 14 h wall clock.
    H_attr = 8 × 14 = 112 GPU-h
    $/run  = 112 × $2.99 = $334.88

  Now load the failure rate. Runs crash, OOM, diverge, get preempted.
  With success rate s:
    $/successful-run = $/run ÷ s        (naive form)
    at s = 0.55:  $334.88 / 0.55 = $608.87
    the $273.99 gap IS the price of iteration inefficiency.

  ── BUT THE NAIVE FORM IS PESSIMISTIC, AND THE FIX IS THE INSIGHT ────
  Failures do not all cost a full run. A job that OOMs in minute 3 costs
  minutes; one that diverges at hour 13 of 14 costs nearly everything.
  Let E[t_fail] be the mean time-to-failure of failed runs:

    GPU-hours per SUCCESSFUL run
      = t_run × G  +  (1−s)/s × E[t_fail] × G

    s = 0.55, t_run = 14 h, G = 8, E[t_fail] = 4.2 h
      = 112 + (0.45/0.55) × 4.2 × 8
      = 112 + 27.5 = 139.5 GPU-h  →  $417.11 per successful run

    versus the naive $608.87 — a 32 % overstatement, because the naive
    form assumes every failure burned a full run.

  ⇒ AND THE LEVER FALLS OUT: cutting E[t_fail] (fail fast — shape checks,
    a 60-second smoke run, OOM prediction from batch geometry) is worth
    as much as raising s, and is usually far cheaper to implement.
```

**`$/experiment`, `$/customer`, `$/feature`.** Any denominator the business already counts works, subject to one constraint: the denominator must be attributable to the same key as `H_attr`. If your billing system counts customers and your metrics count namespaces, the join needs a mapping table, and building that table is usually the real work.

### 9. Sanity-checking, and what a market price does and does not tell you

Once you have a number, check it against the world before you publish it.

```
  PLAUSIBILITY LADDER — run all four, in this order.

  1. ROOFLINE CHECK. Compute §3's theoretical tokens/s at MBU = 1.0
     and compare to measured. Achieving > 100 % means a bug (double
     counting, wrong window, cached tokens counted as generated).
     Achieving < 20 % means a throughput problem, not a cost problem.

  2. LEDGER CHECK. utilised ≤ allocated ≤ paid (lesson 02). If the
     unit cost fell, confirm it fell because U rose, not because
     H_attr lost series to a scrape gap.

  3. MARKET CHECK. Compare to published list prices for a comparable
     model class. As of 2026-06-24, Anthropic's first-party list
     prices are $1/$5 per 1M in/out for Claude Haiku 4.5, $3/$15 for
     Claude Sonnet 5, and $5/$25 for Claude Opus 5.
     ⚠ A LIST PRICE IS NOT A COST. It contains gross margin, R&D
       amortisation, safety and serving overhead, and a different
       model class than yours. Use it as an ORDER-OF-MAGNITUDE
       BOUND ONLY: if your computed cost to serve a small open-weight
       model exceeds a frontier vendor's retail price, you have a bug
       or a utilisation catastrophe, and either way it is not a
       pricing insight.

  4. TREND CHECK. Unit cost should fall over time with no action, as
     hardware and kernels improve, and fall faster with action.
     Character.AI reports ~33× lower serving cost than its late-2022
     baseline, via MQA (8× smaller KV cache than GQA), further
     cross-layer KV sharing and inter-turn caching to >20× total KV
     reduction, and native int8 training plus int8 inference kernels.
     ⇒ A unit cost that is FLAT for a year is a finding: it means
       nobody is working the factor tree.
```

### 10. The implementation: joining the two ledgers to a counter

The join lives in the same rule groups as lesson 02's ledgers, and inherits the integration constant.

```yaml
# queries/unit-economics.yaml
#
# Δ = 30 s (dcgm-exporter --collect-interval default) ⇒ ×30/3600.
# GPU-hours use sum_over_time — NEVER avg_over_time × window_hours,
# which extrapolates over time a series did not exist (module 05).
groups:
- name: gpu-unit-economics
  interval: 5m
  rules:

  # ── DENOMINATOR: tokens produced in the window, per namespace+model.
  #    increase() handles counter resets on pod restart; a plain delta
  #    would go NEGATIVE on every rollout and silently corrupt the day.
  - record: ns_model:tokens_generated:1d
    expr: |
      sum by (ns, model_name) (
        increase(vllm:generation_tokens_total[1d])
      )
  - record: ns_model:tokens_prompt:1d
    expr: |
      sum by (ns, model_name) (
        increase(vllm:prompt_tokens_total[1d])
      )
  # Billed-but-not-computed. The wedge between U and the work done.
  - record: ns_model:tokens_prompt_cached:1d
    expr: |
      sum by (ns, model_name) (
        increase(vllm:prompt_tokens_cached[1d])
      )

  # ── DIRECT unit cost. ns:gpu_cost_allocated:1d comes from lesson 02
  #    (allocated GPU-hours × per-model EffectiveCost rate).
  #    clamp_min on the denominator prevents a divide-by-zero producing
  #    +Inf on a namespace that served nothing — which then poisons
  #    every aggregate that touches it.
  - record: ns_model:cost_per_million_output_tokens_direct:1d
    expr: |
        ns:gpu_cost_allocated:1d
      / clamp_min(ns_model:tokens_generated:1d / 1e6, 1e-9)

  # ── THE LOADING MULTIPLIER, from this fleet's own buckets.
  #    (paid ÷ allocated) × (1 + non-GPU overhead fraction).
  #    NOT 1/utilisation — that double-charges the tenant's own idle
  #    hours, which are already inside the allocated ledger.
  - record: fleet:loading_multiplier:1d
    expr: |
      ( fleet:gpu_hours_present:1d / sum(ns:gpu_hours_allocated:1d) )
      * (1 + 0.09)          # f: MEASURE IT; do not inherit this number

  - record: ns_model:cost_per_million_output_tokens_loaded:1d
    expr: |
        ns_model:cost_per_million_output_tokens_direct:1d
      * on() group_left() fleet:loading_multiplier:1d

  # ── THE DIAGNOSTICS THAT KEEP THE NUMBER HONEST ─────────────────────
  - record: ns_model:prompt_to_output_ratio:1d
    expr: ns_model:tokens_prompt:1d / ns_model:tokens_generated:1d
  - record: ns_model:prefix_cache_hit_ratio:1d
    expr: |
        sum by (ns, model_name) (increase(vllm:prefix_cache_hits[1d]))
      / clamp_min(
          sum by (ns, model_name) (increase(vllm:prefix_cache_queries[1d])), 1)
  - record: ns_model:preemptions:1d
    expr: sum by (ns, model_name) (increase(vllm:num_preemptions[1d]))
  - record: ns_model:mean_batch_size:1d
    expr: avg by (ns, model_name) (avg_over_time(vllm:num_requests_running[1d]))
```

Four implementation traps, each of which produces a plausible-looking wrong number:

- **Label alignment.** GPU-hours carry `ns` from the pod-resources join (module 04); vLLM metrics carry `model_name` and whatever namespace label your scrape config attaches. If they disagree the join returns empty and the recording rule silently records nothing — alert on the *absence* of the unit-cost series, not just on its value.
- **Multiple models per namespace.** Key on `ns × model_name` or the ratio is an average over a mix, and a shift in traffic between models reads as a cost change.
- **Restarts.** `increase()` over a counter that resets is correct; `x - x offset 1d` is not.
- **Window skew with the app.** If the token counter and the GPU-hours are computed over different windows, the error is proportional to the traffic asymmetry between them — largest on exactly the bursty workloads whose unit cost you care about most.

## Perspectives

**Product and pricing.** A price sheet is a promise made in advance against demand you do not yet control, and the number it must clear is the **fully-loaded** unit cost — every customer's traffic implicitly shares the fleet's unallocatable capacity, control plane and storage whether or not that customer ever sees an idle GPU. Price below fully loaded and you are funding that customer out of margin earned elsewhere. This is why pricing teams push back when engineering quotes a direct number: it looks better and it is the wrong basis for a commitment made across a whole fleet's economics.

**Engineering optimisation.** An efficiency team must be judged on the **direct** number, because most of what inflates the loaded one — fleet-wide fragmentation, someone else's abandoned namespace, the control plane — is outside its control. Direct cost isolates what the team owns: tokens extracted per GPU-hour its own pods hold. And the factor tree tells that team where to look, in order: batch size (§3's table spans 40×), quantisation (a direct divisor on `W_bytes`), KV geometry, prefix-cache hit rate, then kernels. Judge the same team on the loaded number and a real win vanishes under noise they never caused.

**Finance and FP&A.** Blended versus marginal is an accounting decision with different homes. Blended feeds gross margin and steady-state pricing; marginal feeds accept/reject calls on incremental demand. Using marginal for pricing understates true cost (it treats sunk committed capacity as free forever, right up until renewal); using blended for an accept/reject call on otherwise-idle capacity overstates the cost of saying yes and declines free money. Knowing which model each belongs in is the job.

**Research velocity.** `$/successful-run` is unusual among cost metrics because it prices something that looks like an engineering-quality problem — job success rate and time-to-failure — as a hard dollar figure. An org with poor checkpoint hygiene, no preemption handling or flaky hardware looks fine on `$/run` and terrible on `$/successful-run`, and the gap is the exact ROI of investing in reliability. The refinement in §8 sharpens it further: cutting the *mean time to failure* of doomed runs is often cheaper than raising the success rate, and it shows up in the same number.

**Hardware.** Unit cost is a hardware number in disguise. Decode cost per token is proportional to `(W_bytes + KV_bytes) / (B × BW)`, so a generation that raises HBM bandwidth moves every serving workload's economics with no software change — and quantisation, which halves `W_bytes`, does the same for free. That is why unit cost falls even for teams doing nothing, and why a flat unit cost over a year is evidence that nobody is paying attention.

## Real-world use cases

- **vLLM — `vllm/v1/metrics/loggers.py` (fetched directly).** **What it shows:** the exact denominator surface — `vllm:generation_tokens`, `vllm:prompt_tokens`, `vllm:prompt_tokens_cached`, `vllm:prefix_cache_queries`/`_hits`, `vllm:num_preemptions`, `vllm:request_success` by finish reason, `vllm:iteration_tokens_total`, `vllm:kv_cache_usage_perc`, `vllm:num_requests_running`/`_waiting` — all Counters exposed with the `_total` suffix. **Why it matters:** every denominator decision in §5 is a choice between metrics that actually exist, and the cache counters are what let you measure the billed-versus-computed wedge rather than assume it.

- **Character.AI — "Optimizing AI Inference at Character.AI" and its follow-up.** **What it shows:** roughly **33×** lower serving cost than the company's late-2022 baseline, achieved by attacking the KV cache and the numeric format rather than by buying hardware: Multi-Query Attention in all attention layers (about **8×** smaller KV cache than the GQA most open models use), cross-layer KV sharing and inter-turn caching taking total KV reduction past **20×**, and native int8 training with custom int8 inference kernels so there is no train/serve mismatch. **Why it matters:** it is the factor tree walked end to end in production — every one of those levers is a leaf in §2's diagram, and the compounding is what produces an order of magnitude. **Provenance:** blog.character.ai and research.character.ai are blocked from this build environment; the figures above are from the posts as surfaced in search and are consistently reported across secondary sources. Read the originals before quoting them in an interview.

- **NVIDIA H100 datasheet figures.** **What it shows:** H100 SXM at 80 GB HBM3 with **3.35 TB/s** of memory bandwidth, 700 W TDP, and Tensor Core peaks quoted **with 2:4 structured sparsity** — 1,979 TFLOPS BF16 and 3,958 TFLOPS FP8, i.e. roughly 989 and 1,979 TFLOP/s dense. **Why it matters:** these are the two constants that set §3's and §4's denominators, and the sparsity footnote is the single most common source of a 2× error in a cost model. **Provenance:** resources.nvidia.com and the distributor PDF mirrors are blocked from this environment; the values are the widely reproduced datasheet figures, search-verified. Confirm against the datasheet for your exact SKU — the PCIe and NVL variants differ.

- **Anthropic — published first-party API list prices (cached 2026-06-24).** **What it shows:** $1/$5 per 1M input/output tokens for Claude Haiku 4.5, $3/$15 for Claude Sonnet 5, $5/$25 for Claude Opus 5 — an input:output ratio of 5× across the range. **Why it matters:** it is a checkable market anchor and, more interestingly, the 5× ratio matches the compute asymmetry derived in §4 from bandwidth-bound decode versus compute-bound prefill. It is a **price**, not a cost: it carries margin, R&D amortisation and a different model class, so use it as an order-of-magnitude bound only.

- **OpenCost — `computeIdleCoeffs` in `core/pkg/opencost/allocation.go` (fetched directly).** **What it shows:** with `ShareIdle` set to weighted, idle cost is distributed to each allocation in proportion to that allocation's `GPUTotalCost()` over the per-idle-key total; the alternative constants are `ShareEven` and `ShareNone`. **Why it matters:** the reference OSS implementation already performs the cost-proportional loading of §6 — which both validates the method and shows exactly what a tool can and cannot do for you, since it has no application counter to divide by and therefore stops one step short of a unit cost.

- **FinOps Foundation — the Unit Economics capability, and FOCUS's cost columns.** **What it shows:** unit economics is a named capability in the FinOps Framework rather than a spreadsheet habit, and FOCUS supplies the vocabulary for the numerator's rate — `EffectiveCost` (the accrual view including amortised commitment drawdown) versus `BilledCost` (the invoice, and **0** for usage covered by a prepaid commitment) versus `ListCost`/`ContractedCost`. **Why it matters:** it is why `r` in this lesson is specified on an `EffectiveCost` basis: price a unit on `BilledCost` and a committed fleet looks free for eleven months and catastrophic in the twelfth. **Provenance:** finops.org is blocked from this environment; the column definitions are verified against the FOCUS specification repository, as in lesson 02.

## Worked example

One serving namespace, one billing day, every number traceable to a source or a stated assumption.

```
  WORKLOAD   namespace `llm-chat-prod`, model `qwen-70b-fp8`
  HARDWARE   4 replicas × (2 × H100 SXM 80GB, TP=2) = 8 H100 total
  WINDOW     24 h, UTC, aligned across all three terms
  RATE       fleet commitment mix: 70 % committed @ $1.90/GPU-h,
             30 % on-demand @ $3.20/GPU-h
             r_b = 0.7(1.90) + 0.3(3.20) = $2.29 / GPU-h
             BASIS: FOCUS EffectiveCost. SNAPSHOT: 2026-08.
```

### Step 1 — predict the denominator before measuring it

```
  From §3, per replica (2 × H100, MBU 0.65, B = 48, L_ctx = 3,200):
    W_bytes    = 70 GB
    KV/step    = 48 × 3,200 × 163,840 B          = 25.17 GB
    bytes/step = 95.17 GB
    steps/s    = (0.65 × 6.70e12) / 9.517e10     = 45.8 /s
    TPOT       = 21.8 ms                         (SLO is 30 ms ✓)
    tokens/s   = 48 × 45.8                       = 2,198 /s per replica
    fleet      = 4 × 2,198                       = 8,792 tokens/s
    per day    = 8,792 × 86,400                  = 759.6e6 output tokens

  ⇒ PREDICTION: ~760 M output tokens/day at full load.
```

### Step 2 — measure, and reconcile the gap

```
  MEASURED (from the recording rules in §10):
    ns_model:tokens_generated:1d            = 412.0e6 output tokens
    ns_model:tokens_prompt:1d               = 906.4e6 prompt tokens
    ns_model:tokens_prompt_cached:1d        = 517.0e6  (57.0 % cached)
    ns_model:mean_batch_size:1d             = 26.1 requests running
    ns_model:preemptions:1d                 = 1,840
    prompt : output ratio                   = 2.20 : 1

  RECONCILE: 412.0 / 759.6 = 54.2 % of the roofline prediction.
  Where did the other 45.8 % go?
    · mean batch 26.1, not the assumed 48 → §3's table says the weight
      read is amortised over half as many sequences. Recomputing at
      B = 26: KV/step 13.63 GB, bytes/step 83.63 GB, steps/s 52.1,
      tokens/s = 1,355 per replica → 468e6/day. That alone explains
      most of the shortfall — it is a TRAFFIC/BATCHING finding, not a
      hardware one.
    · 1,840 preemptions of long generations = recomputed decode work
      inside H_attr that produced no incremental U.
    · diurnal trough: three hours below B = 8, where §3's table says
      unit cost is 5× the steady-state figure.
  ⇒ The gap is EXPLAINED, in factors, not hand-waved. That is the
    deliverable of a unit-cost review.
```

### Step 3 — the direct unit cost

```
  H_attr = 8 GPUs × 24 h                        = 192.0 GPU-h
           (ALLOCATED ledger — billed regardless of load, lesson 02)
  numerator = 192.0 × $2.29                     = $439.68

  DIRECT $/1M OUTPUT TOKENS
    = 439.68 / (412.0e6 / 1e6)
    = 439.68 / 412.0                            = $1.067 / 1M

  DIRECT $/1M "billable" TOKENS (output + prompt, the compute view)
    = 439.68 / ((412.0 + 906.4))                = $0.333 / 1M

  DIRECT $/1M COMPUTED tokens (excluding the 517.0e6 cached prompt
  tokens you billed but never processed)
    = 439.68 / ((412.0 + 906.4 − 517.0))        = $0.549 / 1M

  ⇒ THREE DEFENSIBLE NUMBERS, 3.2× APART, FROM ONE DAY OF DATA.
    None is wrong. Publishing one without naming which is.
```

### Step 4 — the loading

```
  Fleet buckets for the same day (lessons 02–04):
    P (paid)       = 10,752 GPU-h ÷ 7           = 1,536.0 GPU-h/day
    A (allocated)  =  8,736 GPU-h ÷ 7           = 1,248.0 GPU-h/day
    f              = 0.09 (control plane, fast storage, networking —
                           measured as a fraction of GPU spend)

    P/A = 1.2308        L = 1.2308 × 1.09       = 1.3416

  FULLY LOADED $/1M OUTPUT TOKENS
    = $1.067 × 1.3416                           = $1.432 / 1M
    the tax                                     = $0.365 / 1M (34 %)

  WHAT THE TAX IS MADE OF (per §6 and lesson 04):
    gap A — unallocatable + fragmented + cordoned   $0.245 / 1M
    non-GPU overhead                                $0.120 / 1M
  ⇒ Two-thirds of the loading is fragmentation and unallocatable
    capacity — i.e. lesson 04's number, arriving here as a per-token
    surcharge. THAT is how you make a scheduler config change look
    like a product-margin problem, which is what gets it funded.
```

### Step 5 — decide, and show the sensitivity

```
  If the product prices at $1.20 per 1M output tokens:
    direct       $1.067  → looks like 11 % margin
    fully loaded $1.432  → UNDERWATER by 19 %
  The fleet is subsidising this service out of capacity that is not free.

  ── WHAT MOVES IT, IN ORDER OF LEVERAGE (each computed, not guessed) ──
   lever                        mechanism (factor tree leaf)   loaded $/1M
   ────────────────────────────────────────────────────────────────────
   as measured                  B = 26.1, L = 1.342               $1.432
   raise mean batch 26 → 40     amortise W_bytes over more seqs   $1.005
     (route traffic to fewer replicas; consolidate the trough)
   + fix fragmentation,         L 1.342 → 1.150                   $0.861
     P/A 1.231 → 1.055 (L04)
   + int4 weights (W 70→35 GB)  halves the fixed read per step     $0.560
     — REQUIRES a quality evaluation; do not assume it is free
   + commitment 70 % → 90 %     r_b $2.29 → $2.03                 $0.497
     (L06)
   ────────────────────────────────────────────────────────────────────
   AND THE ONE THAT DOES NOTHING FOR UNIT COST:
   buy 8 more H100s             H_attr doubles, U doubles          $1.432
     ⇒ capacity is not efficiency. Adding hardware leaves the
       ratio EXACTLY where it was. Only the factor tree moves it.

  SENSITIVITY IN THE TWO CONTESTED INPUTS (loaded $/1M output tokens):

              MBU 0.50   MBU 0.65   MBU 0.80
   r = $1.90    $1.545     $1.188     $0.965
   r = $2.29    $1.862     $1.432     $1.163
   r = $2.99    $2.431     $1.870     $1.519

  READ IT AS: "at our measured MBU and our current commitment mix the
  loaded cost is $1.43/1M; disagree with the MBU or the rate and the
  table gives you the answer under your assumption." That is how you
  end an argument about the conclusion by having it about an input.
```

## Practice

Feeds the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md).

1. **Predict before you measure.** For one serving workload, compute the §3 roofline: `W_bytes`, `kv_bytes_per_token` from the model's layer/head geometry, `bytes_per_step` at your observed batch and context, and the implied tokens/s and TPOT. Then measure and reconcile.
   **Acceptance:** measured tokens/s expressed as a percentage of the roofline, with the shortfall attributed to named factors (batch, preemptions, trough hours, kernel efficiency) rather than to a residual.

2. **Build the calculator.** Ingest three inputs — attributed GPU-hours per namespace-day, a commitment-mix rate table, and app token deltas — and emit direct and fully-loaded `$/1M tokens`, the loading multiplier `L`, and the per-token tax.
   **Acceptance:** `L` is computed as `(P/A)(1+f)`, not `1/utilisation`, with a comment explaining the double-count that avoids; every dollar figure carries its rate basis and date.

3. **Publish the three denominators.** Output-only, output+prompt, and output+prompt-minus-cached, with the prompt:output ratio and prefix-cache hit rate beside them.
   **Acceptance:** a one-line policy statement saying which is your headline number and why.

4. **Extend to training.** From a runs table with a `status` column and durations, compute `$/run`, naive `$/successful-run`, and the refined form using `E[t_fail]`. Report the difference between the two and the GPU-hours burned on failures.
   **Acceptance:** the refined form is used for the headline, with `E[t_fail]` measured from your own data and the fail-fast lever quantified.

5. **Add a marginal-versus-blended toggle.** Recompute the unit cost of the top-decile traffic hour served from spot or on-demand top-up, and contrast with the blended figure.
   **Acceptance:** a two-sentence memo naming which basis you would quote for (a) a price sheet, (b) a model-A-versus-B efficiency review, (c) accepting an overnight batch job.

6. **Write the sensitivity table.** Vary the two inputs most likely to be challenged (`r` and `MBU`, or `r` and batch size) and publish the grid alongside the point estimate.
   **Acceptance:** the point estimate appears inside the grid, and the memo invites disagreement with an input rather than with the conclusion.

## Common pitfalls

1. **Dividing the monthly bill by monthly tokens.** *Mechanism:* it skips per-workload attribution (so it is a fleet average), uses whatever rate the invoice implies (so on a committed fleet it is wrong by tens of percent, and it mixes `BilledCost` with `EffectiveCost` across months), and neither includes nor excludes the overhead deliberately. *Symptom:* a number that cannot be compared across months, teams, or to anyone else's. *Fix:* one window, one key, three stated terms.

2. **Loading with `1 / utilisation`.** *Mechanism:* under charge-allocated policy the tenant's own idle hours are already inside `H_attr`; multiplying by `A/Uz` charges them a second time. On the worked fleet that is 2.381 instead of the correct 1.342 — a 78% overstatement that makes healthy services look unprofitable and drives the wrong pricing decision. *Fix:* `L = (P/A)(1+f)`; recover only what is genuinely unattributable.

3. **Quoting `paid ÷ utilised` as a unit cost.** *Mechanism:* it is a real and useful diagnostic ("what an hour of actual compute costs if nothing were wasted"), but it embeds every gap in the fleet and belongs on nobody's invoice. On the worked fleet it is 2.93× the allocated rate. *Fix:* label it a diagnostic and never divide a business counter by it.

4. **Using list or on-demand rates on a committed fleet.** *Mechanism:* the rate term is a weighted mean over the commitment mix; substituting the sticker rate overstates cost by the coverage-weighted discount. *Fix:* `r_b = Σ share_i × rate_i` on a stated `EffectiveCost` basis, refreshed when coverage changes (lesson 06).

5. **Quoting the wrong basis for the audience.** *Mechanism:* direct excludes overhead that the P&L must still pay; loaded includes waste an engineering team never controlled. *Symptom:* prices set underwater, or efficiency wins invisible under fleet noise. *Fix:* say "direct" or "fully loaded" out loud every single time, and put it in the metric name.

6. **Ignoring the prefill/decode asymmetry.** *Mechanism:* output tokens each require a full bandwidth-bound pass over the weights; prompt tokens are processed in a compute-bound parallel pass, roughly 5× cheaper per token on the worked configuration. A workload whose prompt:output ratio drifts will show a moving unit cost with no efficiency change at all. *Fix:* publish the ratio beside the unit cost, or price a weighted unit.

7. **Counting cached prompt tokens as work.** *Mechanism:* a prefix-cache hit is billed to the customer and costs almost nothing to serve, so including cached tokens in a *compute* denominator understates your true cost per computed token — and excluding them from a *revenue* denominator understates your efficiency. *Fix:* track `vllm:prompt_tokens_cached` and the hit ratio, and state which denominator you used.

8. **Ignoring failure rate and time-to-failure in training unit economics.** *Mechanism:* `$/run` hides the GPU-hours burned by crashed, OOM'd, diverged and preempted jobs. The naive correction `$/run ÷ s` overcorrects, because failures that die early cost little — the honest form adds `((1−s)/s) × E[t_fail] × G` GPU-hours per successful run. *Fix:* measure `E[t_fail]`, and treat fail-fast as a first-class cost lever alongside raising `s`.

9. **Treating capacity as efficiency.** *Mechanism:* doubling the fleet doubles both `H_attr` and `U`, leaving the ratio exactly where it was — but it feels like progress and it consumes the budget. *Fix:* before approving hardware, require the proposal to name which leaf of the factor tree it moves.

10. **Window and identity skew.** *Mechanism:* GPU-hours on a UTC billing boundary joined to tokens on local time, or GPU-hours by namespace joined to tokens by model when a namespace serves several models. The result stays plausible, which is why it survives. *Fix:* one window, one key (`ns × model_name`), one source of truth; alert on the *absence* of the joined series, because an empty join records nothing rather than failing loudly.

11. **Letting the number go stale.** *Mechanism:* unit cost falls with hardware generations, kernels and quantisation whether or not you act, so a figure computed once and quoted for a year is wrong in a direction that flatters you and misleads pricing. *Fix:* recompute on a schedule, publish the trend, and treat a flat unit cost as a finding.

## Self-check

- **State the unit-cost identity with units, and name the three ways the join goes wrong.**
  **Answer:** `unit_cost [$/U] = H_attr [GPU-h] × r [$/GPU-h] ÷ U [units]`, dimensionally `GPU-h × $/GPU-h ÷ units = $/unit`; report tokens per million and say so. `H_attr` normally comes from the **allocated** ledger, because that is what you pay regardless of activity and it is exact in every sharing regime; `r` is a blended rate on a stated FOCUS `EffectiveCost` basis with a date; `U` comes from an application counter over the *same* window. The three failure modes are **window skew** (GPU-hours on a UTC billing boundary against tokens on local time — a double-digit error on a diurnal workload, and it still looks plausible), **identity skew** (GPU-hours by namespace against tokens by model when one namespace serves several models, so the ratio silently averages over a traffic mix and moves when the mix moves), and **ledger skew** (allocated hours in the numerator against a throughput measured only under load, which understates cost by the idle fraction).

- **Derive tokens per second for a decode workload from hardware constants, and say what each term does.**
  **Answer:** Each decode step reads every weight from HBM once — shared across the whole batch — plus the KV cache of every active sequence. So `bytes_per_step = W_bytes + B × L_ctx × kv_bytes_per_token`, where `W_bytes = N_params × bytes_per_param` and `kv_bytes_per_token = 2 × n_kv_heads × d_head × n_layers × bytes_per_element`. Then `steps/s = MBU × BW_peak / bytes_per_step` and `tokens/s = B × steps/s`, with `1/steps/s` being TPOT. Worked on 70B in FP8 (70 GB of weights, 80 layers, 8 KV heads, head dim 128 → 163,840 B of KV per context token) on 2×H100 SXM (6.70 TB/s peak) at MBU 0.65, batch 64 and 4,096-token context: 42.95 GB of KV plus 70 GB of weights is 112.95 GB per step, giving 38.6 steps/s, TPOT 25.9 ms and 2,468 tokens/s; at `r = $2.99` that is `$5.98 / 8.885e6 tokens/h × 1e6 = $0.673` per million output tokens. The terms do different jobs: `W_bytes` is a fixed per-step cost that batching amortises, KV traffic grows linearly in `B × L_ctx` and eventually caps the benefit, `MBU` is the measurable efficiency factor, and `BW_peak` is the hardware generation.

- **Why does batch size change unit cost by more than an order of magnitude, and what stops you raising it?**
  **Answer:** Because the weight read — the dominant term — is per *step*, not per sequence, so every extra concurrent sequence rides along for the cost of its own KV traffic. On the worked configuration, unit cost falls from $26.80 per million tokens at `B = 1` to $0.673 at `B = 64` — about 40× — with sharply diminishing returns after KV bytes start to dominate ($0.464 at `B = 128`, only 1.45× better than 64). Two things stop you: **latency**, since TPOT rises from 16.2 ms to 35.8 ms across that range and the batch ceiling is set by the SLO rather than by economics; and **KV capacity**, since `B × L_ctx × kv_bytes_per_token` must fit in HBM alongside the weights (`vllm:kv_cache_usage_perc` is the metric that tells you which limit you hit). This is also why lesson 03 insists that a quiet serving replica is a throughput problem, not an idle-reclaim problem: at `B = 1` the hardware is cheap per hour and catastrophic per token.

- **Why are input tokens cheaper than output tokens, and roughly by how much?**
  **Answer:** They run on different bottlenecks. Prefill processes the whole prompt in one parallel pass and is compute-bound: `tokens/s = MFU × P_peak / (2N)`, using the standard 2 FLOPs per parameter per token. Decode produces one token per sequence per pass and is memory-bandwidth-bound, requiring a full read of the weights per step. On the worked rig — 2×H100 SXM, 70B in FP8, dense FP8 peak about 1,979 TFLOP/s per GPU (the datasheet's 3,958 TFLOPS is stated *with* 2:4 sparsity), MFU 0.45 — prefill runs at about 12,721 tokens/s versus decode's 2,468 at batch 64, so roughly **5×** cheaper per token: $0.131 versus $0.673 per million. That is the same order as published price sheets, where Anthropic's list prices are $3/$15 for Claude Sonnet 5 and $5/$25 for Claude Opus 5 — a 5× input:output ratio. The ratio is hardware, not pricing convention. Practical consequence: if you price on output tokens only, publish the prompt:output ratio too, or a drift in prompt length will read as a cost regression.

- **Give the correct loading multiplier and explain what the common version double counts.**
  **Answer:** `L = (P / A) × (1 + f)`, where `P` is paid GPU-hours, `A` is allocated GPU-hours and `f` is non-GPU overhead as a fraction of GPU spend. `P/A` recovers gap A — capacity nobody could hold, which includes fragmentation, cordoned devices and MIG stranding — and `(1+f)` recovers control plane, storage and networking. The common `L = 1/utilisation` (i.e. `A/Uz`) double counts, because under this module's charge-allocated policy the tenant's *own* idle hours are already inside `A` and already billed; loading them again charges the same waste twice. On the module's fleet, `P/A = 1.2308` gives `L = 1.342` with `f = 0.09`, whereas `A/Uz = 2.381` — a 78% overstatement that turns a healthy service into a paper loss. A third number, `P/Uz = 2.931`, is a legitimate *diagnostic* ("cost per hour of real compute") but belongs on no invoice. Optimise on direct, price on fully loaded, and label every number.

- **When do you use the marginal rate rather than the blended one?**
  **Answer:** For accept/reject decisions on incremental work. The blended rate `r_b = Σ share_i × rate_i` over the commitment mix is what capacity costs on average and is the right basis for pricing, gross margin and any published unit cost. The marginal rate is the rate of the capacity that would serve one more unit: near zero when committed headroom is sitting idle and would otherwise be wasted, and the on-demand or spot rate once you are above the committed baseline. Use marginal to decide whether to take an overnight batch job or a burst of customer traffic; use blended to set the price. Reversing them causes the two classic errors — pricing as though sunk capacity is free forever (until renewal), and declining revenue-positive work that would otherwise be pure waste.

- **A research fleet reports `$/run` of $334.88 at a 55% success rate. What is the honest cost per usable result?**
  **Answer:** Not `$334.88 / 0.55 = $608.87` — that assumes every failure burned a full run, which is false. Use the time-to-failure form: GPU-hours per successful run `= t_run × G + ((1−s)/s) × E[t_fail] × G`. With `t_run = 14 h`, `G = 8` GPUs, `s = 0.55` and a measured mean time-to-failure of 4.2 h, that is `112 + (0.818 × 4.2 × 8) = 139.5` GPU-h, or **$417.11** per successful run at $2.99 — a 32% smaller figure than the naive one, and the $82 gap over `$/run` is the real reliability tax. The lever falls out of the formula: cutting `E[t_fail]` (shape checks, a 60-second smoke run, OOM prediction from batch geometry) is worth as much as raising `s` and is usually much cheaper to build. Report both the number and `E[t_fail]`, because a fleet with a low success rate and fast failures is in far better shape than one with the same rate and failures at hour 13.

## Connections & what's next

Backward: this lesson is the module's hinge and it consumes everything before it. Lesson 01 sets the error band on `H_attr` — under time-slicing the numerator carries the exposure fraction `E`, so a unit cost inherits it and must say so. Lesson 02 supplies the ledger choice and the `EffectiveCost` basis for `r`. Lesson 03 supplies the idle states that explain why a replica at batch 1 costs 40× per token what the same hardware costs at batch 64. Lesson 04 supplies gap A, which arrives here as two-thirds of the loading multiplier and turns a scheduler configuration change into a product-margin argument.

Forward: lesson 06 opens the one input treated as given here — `r_b` is not free-floating, it is the output of a commitment strategy, and the coverage dial trades a lower blended rate against exposure to unused commitment. Lesson 07 decomposes the rate further across vendor classes, where the same silicon differs several-fold in sticker price. Lesson 08 turns a unit cost into something a tenant is actually shown, alongside the two ledgers. And lesson 10 gives the whole thing a schema: `x_AppTokens` and `x_AppRuns` joined to the FOCUS cost columns is precisely this lesson's division, made queryable.

Next: **lesson 06** — where the rate in the numerator comes from, and what it costs to lower it.

## References & further reading

**Primary sources — the counters**

1. **vLLM — `vllm/v1/metrics/loggers.py`** — https://github.com/vllm-project/vllm — fetched directly. `vllm:generation_tokens`, `vllm:prompt_tokens`, `vllm:prompt_tokens_cached`, `vllm:prompt_tokens_by_source`, `vllm:prefix_cache_queries`/`_hits`, `vllm:num_preemptions`, `vllm:request_success` (labelled by finish reason), `vllm:iteration_tokens_total`, `vllm:kv_cache_usage_perc`, `vllm:num_requests_running`/`_waiting`. All Counters, so exposed with the `_total` suffix — the naming detail behind a common empty-join bug.
2. **vLLM — production metrics documentation** — https://docs.vllm.ai/en/stable/design/metrics/ — the design rationale for the metric set and the V1 changes, useful when reconciling a version whose names differ from the source above.
3. **Prometheus — `increase()`, `rate()` and counter semantics** — https://prometheus.io/docs/prometheus/latest/querying/functions/ — why `increase()` is correct across pod restarts and a manual `x - x offset` is not, plus the subquery syntax used in the joined recording rules.

**Primary sources — the hardware constants**

4. **NVIDIA H100 Tensor Core GPU datasheet** — 80 GB HBM3, **3.35 TB/s** memory bandwidth, 700 W TDP (SXM), Tensor Core peaks quoted **with 2:4 structured sparsity** (1,979 TFLOPS BF16 and 3,958 TFLOPS FP8 → roughly 989 and 1,979 TFLOP/s dense). **Provenance:** resources.nvidia.com and the distributor PDF mirrors are blocked from this build environment; these are the widely reproduced datasheet values, search-verified, and the PCIe/NVL variants differ — confirm for your SKU. The sparsity footnote is the single most common source of a 2× error in a serving cost model.
5. **NVIDIA — Multi-Instance GPU and MIG user guide** — https://docs.nvidia.com/datacenter/tesla/mig-user-guide/ — relevant here because a MIG slice's fraction of a card is the `share` term that scales `H_attr` for a partitioned device (lesson 01's fractional basis).

**Primary sources — the cost vocabulary**

6. **FinOps Foundation — FOCUS specification, cost columns** — https://focus.finops.org/focus-specification/ — `EffectiveCost` (accrual view including amortised commitment drawdown), `BilledCost` (invoice; **0** for usage covered by a prepaid commitment), `ListCost` and `ContractedCost`, plus `PricingQuantity`/`ConsumedQuantity`. Verified against the column definition files in the `FOCUS_Spec` repository, as in lesson 02, because focus.finops.org was unreachable from this environment.
7. **FinOps Foundation — Unit Economics capability** — https://www.finops.org/framework/capabilities/unit-economics/ — the framework's treatment of unit cost as a named capability with maturity stages, which is the organisational context for §6's basis discipline. (finops.org unreachable from this environment; summarised from the capability's published scope rather than quoted.)
8. **Anthropic — published API pricing** — https://www.anthropic.com/pricing — first-party list prices as of **2026-06-24**: $1/$5 per 1M input/output tokens for Claude Haiku 4.5, $3/$15 for Claude Sonnet 5, $5/$25 for Claude Opus 5. Used in §9 strictly as an order-of-magnitude market bound and as the observation that the 5× input:output ratio matches the prefill/decode compute asymmetry.

**Reading the tools' source**

9. **OpenCost — `core/pkg/opencost/allocation.go`, `computeIdleCoeffs`** — https://github.com/opencost/opencost — fetched directly. Idle-sharing coefficients built from each allocation's `GPUTotalCost()` over the per-idle-key total under `ShareWeighted`, with `ShareEven` and `ShareNone` as the alternatives — the reference implementation of §6's cost-proportional loading.
10. **OpenCost — `pkg/costmodel/allocation_helpers.go`** — fetched directly. `GPUHours = <GPU request> × hours` and `GPUCost = GPUHours × node.CostPerGPUHr`: the numerator this lesson divides, computed by the leading OSS tool, which stops one step short of a unit cost because it has no application counter to divide by.

**Real-world engineering**

11. **Character.AI — "Optimizing AI Inference at Character.AI"** — https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/ — and its follow-up, "Part Deux" — https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/ — roughly **33×** lower serving cost than the late-2022 baseline: Multi-Query Attention in all attention layers (about 8× smaller KV cache than GQA), cross-layer KV sharing and inter-turn caching to more than 20× total KV reduction, and native int8 training with custom int8 inference kernels. **Provenance:** both hosts are blocked from this build environment; figures are as surfaced in search and consistently reported by secondary coverage. Read the originals before quoting.
12. **Anyscale — "GPU (In)efficiency in AI Workloads"** — https://www.anyscale.com/blog/gpu-in-efficiency-in-ai-workloads — the failure archetypes behind a denominator that is smaller than the roofline predicts: prefill and decode stalling each other, dataloaders starving training GPUs, CPU-heavy stages bottlenecking GPU-heavy ones. Treat specific percentages as dated vendor snapshots; the causal taxonomy is the durable part.
13. **a16z — "Welcome to LLMflation: LLM inference cost is going down fast"** — https://a16z.com/llmflation-llm-inference-cost/ — the longer-horizon argument that cost for equivalent-quality inference falls roughly an order of magnitude per year, which is the context for §9's trend check and for why any unit-cost snapshot needs a date attached.
14. **NVIDIA — "Leading Inference Providers Cut AI Costs with Open Source Models on NVIDIA Blackwell"** — https://blogs.nvidia.com/ — multi-vendor reports of large cost-per-token reductions moving between hardware generations. **Provenance:** blogs.nvidia.com is blocked from this environment; treat the vendor figures as directional and note that the mechanism — a higher `BW_peak` and `P_peak` in §2's tree — is the checkable part.

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
