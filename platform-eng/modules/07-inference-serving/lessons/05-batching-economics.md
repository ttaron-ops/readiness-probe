---
lesson: "07.5"
title: "Batching economics: the cost-per-token curve"
module: "07"
concept: "Batching economics"
status: not-started
est_time: "8h"
prev: "04-vllm-in-production.md"
next: "06-alternative-servers-disaggregation.md"
artifacts: []
sources: 14
---
# 07.5 · Batching economics: the cost-per-token curve

> **Concept.** Batch size is the single biggest lever on cost-per-token; the operating point is the batch where CPM is minimised subject to your TTFT p99 SLO.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

07.4 produced a tuned engine: a launch command whose every non-default flag is justified by
a measurement, running at zero preemptions with a known binding constraint. That lesson
answered *"how much of this GPU can I safely use."*

This lesson answers the question the module is named after: *"given that safe envelope, what
does a token actually cost, and where inside the envelope should I run?"* It is the cost
pivot — the point where the module stops being about mechanism and starts being about money,
and where the flagship deliverable, the CPM-vs-batch curve with its SLO knee, gets built.

The gap it closes is a specific one. You can already compute the concurrency cap (07.2),
explain why paging makes it achievable (07.3), and set the flags that reach it (07.4). What
you cannot yet do is turn "2,731 output tokens per second" into a number a finance partner
will act on, or defend *why* you chose to run at 96 concurrent sequences rather than 112
when 112 was measurably faster. Both of those are this lesson.

**What is assumed and pinned.** Numbers are for **vLLM v0.27.1** (cross-checked against
`main` @ `c1e4387`, 2026-08-17). GPU hardware figures are vendor specifications, cited
inline. GPU hourly rates use the module deliverable's reference points — **H100 ≈ $2.89/hr,
A100 ≈ $1.39/hr** — which are on-demand neocloud list prices at the time this module was
written and which you must replace with your own contracted rate. Metric names match the
verified set in [module 05 lesson 06](../../05-gpu-observability/lessons/06-inference-slos.md).

## Why this matters

You rent a GPU by the hour, and the bill is fixed the moment the pod schedules. An H100
burns the same $2.89 whether it serves one request or two hundred. So the entire economics
of self-hosted inference is a **denominator game**: cost per million tokens is the hourly
rate divided by how many tokens you extracted from that hour. Batching is how you fill the
denominator, and it is worth roughly two orders of magnitude — the single largest software
lever in this module.

The stakes are not abstract. A team that measures throughput at batch 1, or quotes a
saturation number with no SLO attached, or forgets that their fleet is 35 % busy, will be
wrong about their own cost by between 3× and 50×. Those errors do not cancel; they compound,
and they all point the same direction (optimistic). The result is a self-hosting business
case that survives the design review and dies in the quarterly bill review.

The career-shaped version: "Design an LLM inference platform" is the canonical loop question
for this job family, and its sharpest sub-question is some form of *"what is your cost per
million tokens, and how did you pick the operating point?"* Answering with a single
throughput-at-saturation figure is a tell. Answering with a curve, a named knee, a stated
SLO, an explicit utilisation assumption, and a date on the measurement is the difference
between describing a benchmark and describing an engineering decision.

## What's new here (calibration)

Referenced, not re-taught: the roofline and memory-bound decode (module 03); TTFT / ITL /
TPOT definitions and the verified vLLM metric names (module 05); the KV concurrency cap
(07.2); continuous batching and the unified token budget (07.3); the tuned config and
preemption (07.4).

Genuinely new here:

1. **A predictive throughput model, not just a shape.** The two-term decode-step equation —
   weights plus KV read, divided by achieved bandwidth — with a measured efficiency factor,
   which reproduces 07.4's measurements to within a percent and lets you *predict* the curve
   before renting anything.
2. **The critical batch size, derived from machine balance.** `peak_FLOP/s ÷ bandwidth` gives
   a hardware constant with units of FLOP per byte; the batch at which decode stops being
   bandwidth-bound is that constant. On an H100 it is about 295, which is *larger than the
   batch most deployments can hold*, and that single observation explains why almost every
   real LLM serving deployment is memory-bound rather than compute-bound.
3. **The SLO constraint solved in closed form.** Your ITL target caps the resident batch
   directly, and you can compute that cap from the model geometry and the GPU's bandwidth
   without measuring anything. So does your TTFT target, through the prefill token budget.
   The operating point is the smallest of three caps, and knowing which one binds tells you
   which lever to pull.
4. **Goodput**, the metric that makes the SLO part of the measurement instead of an
   annotation on it — including `vllm bench serve --goodput`'s exact semantics.
5. **Utilisation as a first-class term**, with the arithmetic for how it converts a benchmark
   number into a production number, and how to measure it rather than assume it.
6. **The sweep as tooling**, not a shell loop: `vllm bench sweep serve_workload` and
   `plot_pareto`, whose axes (tokens/s/user against tokens/s/GPU) *are* the latency–cost
   trade in the two units that matter.

## Core concepts

### 1. The equation, term by term

```
                       hourly_rate
  effective_cpm  =  ─────────────────────────────────────────  ×  1e6
                    output_tok_per_sec × 3600 × utilization
```

This is the module deliverable's formula. Each term hides a decision:

**`hourly_rate` — dollars per GPU-hour, all in.** The rented instance price is the floor,
not the number. Load it with everything you pay to keep that GPU serving:

```
  hourly_rate = instance_$/hr
              + attached storage amortised per GPU-hour
              + egress attributable to this endpoint
              + control-plane share (LB, Prometheus, the router)
              + the reserved/idle premium if you hold capacity you are not using
```

A common honest starting point is instance price × 1.15–1.3. If you are on committed-use
or reserved capacity, the correct rate is your *committed* rate divided by your *achieved*
utilisation of that commitment — which is the same utilisation term appearing twice, once
in procurement and once in serving.

**`output_tok_per_sec` — decode tokens only, summed across concurrent requests.** Two
disciplines here. First, **output only**: prefill tokens are dramatically cheaper per token
(one parallel pass over the whole prompt versus one full weight read per generated token),
so counting them inflates your throughput and flatters your CPM. `vllm bench serve` reports
both `Output token throughput (tok/s)` and `Total token throughput (tok/s)`; CPM uses the
first. Second, **sustained, not peak**: the harness also prints
`Peak output token throughput (tok/s)`, which is a single best interval and not a rate you
can bill against.

If your product prices input and output separately — as every commercial API does — you
want a **blended** figure instead, which needs its own arithmetic (§9).

**`3600`** — seconds per hour. Carries the units: `$/hr ÷ (tok/s × s/hr) = $/tok`.

**`utilization` — the honesty term.** The fraction of wall-clock during which the GPU is
actually running at `output_tok_per_sec`. At 1.0 you are quoting a benchmark. At 0.35 you
are quoting production. Nothing else in the formula is as commonly omitted or as large.

**`× 1e6`** — per million tokens, because that is the unit every price list uses.

**Worked, twice.** An H100 at $2.89/hr sustaining 3,088 output tok/s (07.4's measured
`--max-num-seqs 96` configuration):

```
  benchmark CPM  = 2.89 / (3088 × 3600 × 1.00) × 1e6
                 = 2.89 / 11,116,800 × 1e6
                 = $0.260 per million output tokens

  production CPM = 2.89 / (3088 × 3600 × 0.35) × 1e6
                 = $0.743 per million output tokens
```

**Same GPU, same model, same config, 2.86× the cost — purely because the fleet is idle
two-thirds of the time.** Quote the first number in an engineering review and the second one
to finance, and be explicit which is which. The gap between them is precisely the
opportunity that autoscaling (07.8) and multi-tenancy (07.10) exist to capture.

### 2. Where throughput comes from: the two-term decode model

Batching works because of one asymmetry: **the weights are read once per step regardless of
how many sequences are in the batch.** Everything else follows.

A single decode step must move, from HBM into the SMs:

```
  bytes_per_step  =  W_bytes                       weights, read ONCE
                  +  B × ctx × kv_bytes_per_token  KV history, read PER SEQUENCE

  t_step  =  bytes_per_step / (BW_peak × η)        while memory-bound
  throughput  =  B / t_step                        tokens per second
```

where `η` is the achieved fraction of peak bandwidth (kernel efficiency, ~0.6–0.75 for
well-optimised attention and GEMM kernels on current hardware — **measure yours**), `B` is
the resident batch, and `ctx` is the average live context length.

**Instantiate it** for Llama-3.1-8B, bf16 weights and bf16 KV, on one H100-80GB SXM:

```
  W_bytes             = 8.03e9 params × 2 B            = 1.606e10 B
  kv_bytes_per_token  = 2 × 32 layers × 8 kv_heads
                          × 128 head_dim × 2 B          = 131,072 B    (07.2 §2)
  BW_peak (H100 SXM5, HBM3, NVIDIA spec)                = 3.35e12 B/s
  workload: 4096-token prompt, 512-token generation
  ⇒ time-averaged live context ctx ≈ 4096 + 512/2       = 4,352 tokens
  ⇒ per-sequence KV read per step = 4352 × 131,072      = 5.704e8 B  (570 MB!)
```

Note what that last line says: **at 4k context, one sequence's KV history is 570 MB, and
sixteen of them out-weigh the entire model.** Long context does not merely consume capacity;
it consumes bandwidth on every single step.

Now solve for a few batch sizes. Calibrating `η` against 07.4's measured points gives
**η ≈ 0.67**, and with that single fitted constant the model reproduces both measurements:

| B | KV bytes/step | Total bytes/step | t_step | ideal tok/s | ×η=0.67 | 07.4 measured |
|---|---|---|---|---|---|---|
| 1 | 0.57 GB | 16.6 GB | 4.96 ms | 202 | 135 | — |
| 8 | 4.6 GB | 20.6 GB | 6.15 ms | 1,301 | 872 | — |
| 32 | 18.3 GB | 34.3 GB | 10.24 ms | 3,124 | 2,093 | — |
| **64** | 36.5 GB | 52.6 GB | 15.69 ms | 4,079 | **2,733** | **2,731** ✓ |
| **96** | 54.8 GB | 70.8 GB | 21.14 ms | 4,541 | **3,043** | **3,088** ✓ |
| 128 | 73.0 GB | 89.1 GB | 26.59 ms | 4,814 | 3,225 | (preempting) |
| 256 | 146.0 GB | 162.1 GB | 48.38 ms | 5,291 | 3,545 | (does not fit) |

**Two predictions worth more than the table.** First, the model is right to within 1.5 % at
both anchor points using one fitted constant, which means you can *predict* your CPM curve
from a `config.json` and a spec sheet before you rent anything, then use measurement to
confirm rather than to discover. Second — and this is the part people miss — **the returns
are already flattening hard by B = 64**. Going 64 → 128 doubles the batch and buys 18 % more
throughput, because the KV term now dominates the weight term and the KV term is *linear in
B*, so the amortisation that made small batches so profitable has already been spent.

That is the real shape of the batching win, and it is not the shape folklore describes. The
50–100× is real, but it is almost entirely earned between batch 1 and batch 32.

### 3. The critical batch size: when does decode stop being bandwidth-bound?

The roofline answer, in one line. A decode GEMM with a weight matrix of `P` parameters at 2
bytes each and batch `B` does `2·P·B` FLOPs while moving `2·P` bytes, so its arithmetic
intensity is exactly **`B` FLOP per byte**. The hardware's machine balance is
`peak_FLOP/s ÷ peak_bytes/s`. Setting them equal gives the batch at which decode crosses
the roofline ridge:

```
  B*  =  peak_dense_FLOP/s  ÷  peak_memory_bandwidth

  H100 SXM5  : 989.4e12 BF16 dense ÷ 3.35e12 B/s  =  295 FLOP/B  ⇒  B* ≈ 295
  H200 SXM   : 989.4e12            ÷ 4.80e12      =  206         ⇒  B* ≈ 206
  A100 80GB  : 312e12              ÷ 2.039e12     =  153         ⇒  B* ≈ 153
  L40S       : 181.05e12 (dense)   ÷ 0.864e12     =  210         ⇒  B* ≈ 210
  H100 @ FP8 : 1978.9e12           ÷ 3.35e12      =  591         ⇒  B* ≈ 591

  (FLOP/s figures are NVIDIA dense tensor-core specs, no sparsity;
   bandwidth is the published HBM figure. Verify for your exact SKU —
   SXM and PCIe variants differ, and clock/power caps move both terms.)
```

**Read `B* ≈ 295` against the table in §2, where B = 128 already exceeded the KV pool.** For
long-context serving on an 8B model, you will run out of KV capacity — and out of ITL budget
— long before you run out of bandwidth headroom. **Decode is essentially always
memory-bound in production, not because the ridge point is low, but because you cannot hold
a batch large enough to reach it.**

Three consequences that fall straight out:

- **The thing to buy is bandwidth, not FLOPs.** H100 → H200 is +43 % bandwidth on identical
  compute, and for decode that is close to +43 % throughput. The FLOP number on the spec
  sheet is prefill's currency, not decode's.
- **FP8 *raises* B\*** (591 on H100), which sounds bad but is not — it doubles compute while
  leaving bandwidth alone, so it pushes the ridge further away and keeps you comfortably in
  the regime where the *memory* saving (half the weight bytes, and with `--kv-cache-dtype
  fp8` half the KV bytes) is what pays. That is why 07.7's measured FP8 speedup is ~1.5–1.8×
  rather than the 2× the FLOP table suggests.
- **Prefill is on the other side of the ridge.** A 4,096-token prefill has arithmetic
  intensity in the thousands and is compute-bound. Prefill and decode therefore want
  opposite hardware and opposite batch policies — which is the seed of disaggregation (07.6)
  and the reason chunked prefill's dial has two ends (07.4 §5).

### 4. Static versus continuous batching: where the GPU time actually goes

The model in §2 assumes every slot in the batch is doing useful work every step. That
assumption is what continuous batching buys you, and it is worth seeing the counterfactual
because the gap *is* the multiplier.

```
  STATIC vs CONTINUOUS BATCHING — where GPU time is wasted
  ══════════════════════════════════════════════════════════════════════════════
  Six requests arriving at t=0. Output lengths: A=9, B=2, C=14, D=3, E=7, F=5.
  Engine capacity: 4 concurrent sequences.
  █ = slot doing useful work    · = slot occupied but IDLE (paid for, wasted)
  ✓ = request finishes          ▲ = a queued request is admitted

  ── STATIC BATCHING ────────────────────────────────────────────────────────
     Assemble 4, run to completion, return together, then take the next 2.

  iter  1  2  3  4  5  6  7  8  9 10 11 12 13 14 │15 16 17 18 19 20 21
   A  │ █  █  █  █  █  █  █  █  █✓ ·  ·  ·  ·  · │
   B  │ █  █✓ ·  ·  ·  ·  ·  ·  ·  ·  ·  ·  ·  · │   ← 12 idle iterations
   C  │ █  █  █  █  █  █  █  █  █  █  █  █  █  █✓│
   D  │ █  █  █✓ ·  ·  ·  ·  ·  ·  ·  ·  ·  ·  · │   ← 11 idle iterations
      └───────────── batch 1: 14 iterations ─────┘
   E  │                                           │ █  █  █  █  █  █  █✓
   F  │                                           │ █  █  █  █  █✓ ·  ·
                                                  └── batch 2: 7 iters ──┘

    useful slot-iterations : 9+2+14+3+7+5            =  40
    paid   slot-iterations : 4×14 + 2×7              =  70
    SLOT UTILISATION       : 40/70                   =  57 %
    wall clock             : 21 iterations
    E's TTFT               : 14 iterations of pure queueing for a free slot
                             — even though B's slot fell idle at iteration 2.

  ── CONTINUOUS (ITERATION-LEVEL) BATCHING ──────────────────────────────────
     Re-plan after EVERY forward pass. Finished ⇒ blocks freed ⇒ admit.

  iter  1  2  3  4  5  6  7  8  9 10 11 12 13 14
   A  │ █  █  █  █  █  █  █  █  █✓
   B  │ █  █✓
   C  │ █  █  █  █  █  █  █  █  █  █  █  █  █  █✓
   D  │ █  █  █✓
   E  │       ▲█  █  █  █  █  █  █✓          ← admitted the step after B freed
   F  │             ▲█  █  █  █  █✓          ← admitted the step after D freed

    useful slot-iterations : 40
    paid   slot-iterations : 40      (a slot exists only while it is working)
    SLOT UTILISATION       : 100 %
    wall clock             : 14 iterations        (−33 %)
    E's TTFT               : 2 iterations         (−86 %)

  ┌────────────────────────────────────────────────────────────────────────┐
  │ THE MULTIPLIER IS A PROPERTY OF YOUR TRAFFIC, NOT OF THE ENGINE.       │
  │ With fixed-length outputs, static batching wastes NOTHING and the two  │
  │ schemes are identical. All of the win comes from output-length         │
  │ VARIANCE. Chat traffic (σ/μ ≈ 1) is where the 10–20× numbers come      │
  │ from; a fixed-length classification workload sees almost none.         │
  │                                                                        │
  │ AND IT NEEDS PAGEDATTENTION: admitting E at iteration 3 means finding  │
  │ KV space at iteration 3. Contiguous allocation needs a max-sized hole  │
  │ exactly where B's buffer was. Paging needs only "are there N free      │
  │ blocks anywhere" — an integer comparison. (07.3 §7.)                   │
  └────────────────────────────────────────────────────────────────────────┘
```

Anyscale's published benchmark measured the same effect end to end: a naive static baseline
on OPT-13B collapsed to **81 output tokens/second** once realistic output-length variance was
introduced, while continuous batching with paged memory management reached up to **23× that**
on the same 40 GB A100. Orca (Yu et al., OSDI 2022), which introduced iteration-level
scheduling as a research idea, reported 36.9× throughput at matched latency versus
FasterTransformer on GPT-3 175B. Both multipliers are workload-shape-dependent by
construction; quote them with the caveat attached or not at all.

### 5. The other half of the curve: what batching costs you

Throughput is not free. Both latency SLIs degrade with batch, for mechanically different
reasons.

**ITL (and therefore TPOT) is a direct, computable function of batch.** Every resident
sequence's KV must be read every step, so from §2:

```
  ITL  =  t_step  =  (W_bytes + B × ctx × kv_bytes_per_token) / (BW × η)
```

This is linear in B with a positive intercept. At Llama-3.1-8B / 4,352-token context /
H100 / η = 0.67:

```
  ITL(B)  =  (1.606e10 + B × 5.704e8) / 2.245e12  seconds
          =  7.15 ms  +  B × 0.254 ms
```

so ITL is 7.4 ms at B=1, 23.4 ms at B=64, 31.5 ms at B=96, 72.2 ms at B=256. **Read the
intercept and the slope separately**: the 7.15 ms floor is the weight read, which no amount
of batching removes, and the 0.254 ms per additional sequence is the KV read, which is what
you are buying throughput with.

**TTFT degrades for a different reason — queueing, not step time.** With chunked prefill and
decode-priority scheduling (07.3 §8, 07.4 §5), each iteration spends `B` tokens on the
resident decodes and gives what remains of `max_num_batched_tokens` to prefill:

```
  prefill_tokens_per_iteration  =  max_num_batched_tokens − B
  prefill_token_rate            =  (max_num_batched_tokens − B) / t_step

  TTFT  ≈  queue_wait + own_prefill
        ≈  (Q × prompt_len + prompt_len) / prefill_token_rate
           where Q = requests queued ahead of this one
```

At `max_num_batched_tokens = 4096`, B = 96, t_step = 21.1 ms:

```
  prefill_token_rate = (4096 − 96) / 0.0211 s          = 189,573 tok/s
  own prefill (4096-token prompt)                      = 21.6 ms
  each queued request ahead adds                       = 21.6 ms
```

**Notice the coupling that makes this hard**: raising B raises `t_step` *and* lowers
`max_num_batched_tokens − B`, so TTFT degrades quadratically-ish while throughput improves
sub-linearly. That is the mechanism behind the knee.

### 6. The knee, and the constrained optimisation that locates it

Now put the pieces together. The problem is:

> **minimise** CPM(B) = rate / (throughput(B) × 3600 × u) × 1e6
> **subject to** TTFT_p99(B) ≤ T, ITL_p99(B) ≤ I, and B ≤ KV capacity

Since throughput is monotonically increasing in B, CPM is monotonically *decreasing* in B,
which means **the optimum is always the largest feasible B**. The whole problem is therefore
finding which constraint binds first. Three candidates, each computable in closed form.

**Cap 1 — the ITL SLO.** Invert `ITL(B)`:

```
  B  ≤  (I × BW × η  −  W_bytes) / (ctx × kv_bytes_per_token)

  With I = 50 ms:
    I × BW × η   = 0.050 × 3.35e12 × 0.67   = 1.122e11 B     (byte budget per step)
    − W_bytes                               = 1.606e10 B
    = 9.617e10 B available for KV reads
    ÷ (4352 × 131,072 = 5.704e8 B/sequence) = 168.6
  ⇒  B_ITL  =  168
```

**Cap 2 — KV capacity.** From 07.4's tuned boot, `GPU KV cache size: 460,800 tokens`:

```
  B_KV  =  460,800 / 4,352  =  105.9  ⇒  105
```

**Cap 3 — the TTFT SLO.** With `T = 500 ms` and the §5 model, at a candidate B:

```
  budget in queued requests  Q  ≤  T / 21.6 ms − 1  =  22.1  ⇒  22
  offered concurrency        C  =  B + Q
  ⇒ at B = 96:  C ≤ 118

  Then apply a p99 correction. Queue depth is bursty; a p99 that is
  ~1.5–2× the mean is typical for Poisson-ish arrivals, so hold the
  MEAN queue at Q/1.75 ≈ 12 to keep the p99 under 22.
  ⇒ C_operating ≈ 96 + 12 = 108
```

**The smallest cap binds: `min(168, 105, …) = 105`, and it is KV capacity.** Back off for
burst headroom and you land at **B = 96** — which is exactly the operating point 07.4's
measurement produced, arrived at here from geometry and a spec sheet.

That agreement is the point of the exercise. **Which cap binds tells you which lever to
pull**, and the answer is not the same for everyone:

| Binding cap | Symptom | The lever |
|---|---|---|
| **KV capacity** | preemptions at your target B; `kv_cache_usage_perc` ≈ 1.0 | `--kv-cache-dtype fp8` (2×), shorter `--max-model-len`, bigger-HBM SKU, TP |
| **ITL SLO** | ITL p99 over budget with the pool half empty | more bandwidth (H200), FP8 KV (halves the per-sequence read), shorter context, or **relax the SLO** |
| **TTFT SLO** | TTFT p99 over budget, ITL fine | raise `--max-num-batched-tokens`, add replicas, or disaggregate prefill (07.6) |
| **`max_num_seqs`** | `num_requests_running` pinned exactly at the flag | raise the flag — costs nothing |

Now the picture the deliverable is built around:

```
  THROUGHPUT, LATENCY AND COST AGAINST CONCURRENCY
  Llama-3.1-8B · 1×H100-80GB @ $2.89/hr · 4096-in / 512-out · util = 1.0
  Model of §2 at η = 0.67, anchored on 07.4's measurements at B = 64 and 96
  ══════════════════════════════════════════════════════════════════════════════

  output tok/s                                              CPM ($/1M output tok)
  4000┤                                    ▁▁▁▁▄▄▄▄▄▄▄▄▄▄  ┤ 0.05
      │                            ▁▄▄▄▀▀▀▀                │
  3000┤                    ▁▄▄▀▀▀▀▀                        ┤ 0.27  ← $0.260 @ B=96
      │             ▁▄▄▀▀▀▀    ╎                           │
  2000┤        ▄▄▀▀▀          ╎                            ┤ 0.40
      │     ▄▀▀               ╎                            │
  1000┤   ▄▀                  ╎                            ┤ 0.80
      │ ▄▀                    ╎                            │
   135┤▀                      ╎                            ┤ 5.95  ← $5.95 @ B=1
      └┬────┬────┬────┬────┬──╎─┬────┬────┬────┬────┬────  ┘
       1    8   16   32   64  ╎96  128  160  192  256
                              ╎
   TTFT p99 (ms)              ╎
   1200┤                      ╎                    ▁▄▄▀▀▀▀
    800┤                      ╎              ▁▄▄▀▀▀
    500┤─ ─ ─ ─ ─ SLO ─ ─ ─ ─ ╎▄▄▄▀▀▀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
    300┤              ▁▄▄▄▀▀▀▀╎
    150┤▄▄▄▄▄▄▄▄▄▄▄▄▀▀        ╎
        └──────────────────────╎──────────────────────────
                              ╎
   ITL p99 (ms)               ╎
     80┤                      ╎                       ▁▄▀▀
     50┤─ ─ ─ ─ ─ SLO ─ ─ ─ ─ ╎─ ─ ─ ─ ─ ─▁▄▄▀▀▀ ─ ─ ─ ─ ─
     30┤                    ▁▄╎▀▀
     10┤▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▀▀▀  ╎
        └──────────────────────╎──────────────────────────
                              ╎
                        THE KNEE — B ≈ 96
  ┌────────────────────────────────────────────────────────────────────────┐
  │ THREE THINGS TO READ OFF THIS PICTURE                                  │
  │                                                                        │
  │ 1. CPM falls 23× from B=1 to B=96, and 96 % of that fall happens       │
  │    before B=64. The tail of the curve is nearly free money you         │
  │    cannot spend without breaking something.                            │
  │                                                                        │
  │ 2. THE THROUGHPUT-OPTIMAL POINT AND THE SLO-OPTIMAL POINT DO NOT       │
  │    COINCIDE. Throughput keeps climbing to the right of the knee.       │
  │    From B=96 to B=192, CPM improves ~11 % while TTFT p99 doubles       │
  │    through the SLO. You are trading a rounding error on cost for       │
  │    the product.                                                        │
  │                                                                        │
  │ 3. The knee is where a CONSTRAINT crosses, not where the curve         │
  │    "bends". Change the SLO and the knee moves; change the model or     │
  │    the context length and it moves; change nothing and re-measure      │
  │    next quarter and it moves. It is not a property of the GPU.         │
  └────────────────────────────────────────────────────────────────────────┘
```

### 7. Goodput: making the SLO part of the measurement

Annotating a knee on a throughput plot is a manual step, and manual steps rot. **Goodput**
folds the SLO into the number itself: the rate of requests that completed *and* met every
stated SLO. Requests that finished slowly count as zero.

`vllm bench serve` implements it directly (`vllm/benchmarks/serve.py`):

```bash
vllm bench serve \
  --model meta-llama/Llama-3.1-8B-Instruct --base-url http://localhost:8000 \
  --dataset-name random --random-input-len 4096 --random-output-len 512 \
  --max-concurrency 128 --num-prompts 1500 \
  --goodput ttft:500 tpot:50 e2el:20000 \
  --percentile-metrics ttft,tpot,itl --metric-percentiles 50,99 \
  --save-result --result-filename sweep_c128.json
```

Exact semantics, from the source: keys are limited to **`ttft`**, **`tpot`** and **`e2el`**;
values are in **milliseconds**; and a request counts as good only if it satisfies **all**
specified SLOs — `is_good_req = all([slo >= observed for ...])`. The harness then prints
`Request goodput (req/s)` alongside `Request throughput (req/s)`. (The concept and the flag's
help text both point at the DistServe paper, which introduced goodput for exactly this
purpose.)

Why this matters for the deliverable: **plot CPM against goodput rather than against raw
throughput and the knee locates itself.** Past the SLO crossing, goodput *falls* while
throughput still rises, so the goodput-derived CPM curve has a genuine minimum instead of a
monotone slope you have to annotate by hand. That turns "here is where I decided to draw the
line" into "here is where the measurement says the line is."

### 8. Utilisation: the term that changes the answer by 3×

Everything so far measures a GPU under load. Production GPUs are not always under load.

```
                              measured tokens
  utilization  =  ───────────────────────────────────────────
                   tokens the GPU could have served in that
                   wall-clock at the operating point
```

Measured over a week, not a benchmark window. Two ways to compute it that do not require new
instrumentation:

```promql
# (a) Direct: actual output tokens ÷ capacity at your operating point.
#     Replace 3088 with YOUR measured sustained tok/s per replica.
  sum(rate(vllm:generation_tokens_total[7d]))
/ (3088 * sum(max_over_time(kube_deployment_status_replicas{deployment="vllm-chat-8b"}[7d])))

# (b) Proxy: mean resident batch ÷ operating batch. Cheaper, slightly optimistic
#     because a partially-full batch is more efficient per token than a full one.
  avg_over_time(vllm:num_requests_running[7d]) / 96
```

Typical values, and what they say:

| Utilisation | What produced it | What to do |
|---|---|---|
| 0.05–0.15 | Internal/dev endpoint, business-hours traffic, no autoscaling | Scale to zero (07.8), or multiplex tenants onto it (07.10) |
| 0.20–0.40 | Production endpoint sized for peak, diurnal traffic | Autoscale; consider committed-vs-on-demand mix |
| 0.50–0.70 | Autoscaled with a warm floor, batch backfill on the trough | Healthy. Further gains get expensive |
| 0.80+ | Queue-fed batch workload, or you are about to breach SLO | Verify TTFT p99 is still inside budget |

**The one-liner worth carrying into a review:** a GPU at 10 % utilisation turns a $0.26/M
benchmark number into $2.60/M — worse than several commercial APIs, on hardware you own.
This is the single most common way self-hosted inference is mis-priced, and it is why the
checkpoint's fail-signal list names "quoting CPM without utilisation" explicitly.

**Where utilisation actually comes from**, in rough order of leverage: autoscaling to match
diurnal demand (07.8); multi-tenancy so one deployment serves many workloads (07.10);
backfilling troughs with batch/offline work; and right-sizing the replica count to p95
rather than p99.9 traffic. Note that three of those four are *scheduling* decisions, not
serving decisions — which is why utilisation is usually a bigger lever than batch tuning
once your batch tuning is merely competent.

### 9. Prefill, decode, and blended pricing

CPM-on-output-tokens is the right internal engineering metric. It is not what your customers
pay, because input and output tokens have wildly different costs and every commercial API
prices them separately.

The cost split, for the same request:

```
  PREFILL  4,096-token prompt, 8B model
    FLOPs      = 2 × 8.03e9 × 4096                      = 6.58e13 FLOP
    at 989e12 FLOP/s × 0.5 achieved                     = 133 ms of GPU
    ⇒ 4096 tokens in 133 ms                             = 30,800 tok/s per GPU

  DECODE   512 tokens generated, at the B=96 operating point
    throughput per GPU                                  = 3,088 tok/s
    this request's share                                = 3088/96 = 32 tok/s
    ⇒ 512 tokens in 16.0 s

  ⇒ PREFILL IS ~10× CHEAPER PER TOKEN THAN DECODE at this operating point
    (30,800 vs 3,088 tokens per GPU-second)
```

That ratio is why every provider's price list charges 3–5× more for output than input, and
why your own blended cost must weight by your traffic's shape:

```
  cost_per_request  =  (P_in / 30,800  +  P_out / 3,088)  ×  (rate / 3600) / u

  Chat  (P_in=500,  P_out=500):  (0.0162 + 0.1619) s = 0.178 GPU-s ⇒ $0.000143 @ u=1.0
  RAG   (P_in=4000, P_out=200):  (0.1299 + 0.0648) s = 0.195 GPU-s ⇒ $0.000156
  Agent (P_in=8000, P_out=1500): (0.2597 + 0.4858) s = 0.746 GPU-s ⇒ $0.000599
                                                       (rate = $2.89/hr = $0.000803/GPU-s)
```

**The RAG row is the instructive one.** Its 4,000-token prompt costs about the same GPU-time
as chat's entire request, despite generating a quarter as many tokens — so a
per-output-token price list under-charges RAG traffic dramatically. If you publish internal
chargeback based on output tokens only, RAG teams are subsidised by chat teams, and the
subsidy grows as context windows do.

**And prefix caching changes the input term, not the output term.** A shared 2,000-token
system prompt hit from cache costs approximately zero prefill; `vllm:prefix_cache_hits_total
÷ vllm:prefix_cache_queries_total` (in tokens) is the discount factor to apply to `P_in`.
For a high-fan-out workload this is frequently the largest single line-item saving available,
and it is on by default.

### 10. Running the sweep properly

Do not write a bash loop. vLLM ships the sweep harness, and it handles the parts a loop gets
wrong — restarting the server per config, resetting caches between runs, repeating runs for
variance, and resuming after a failure.

```bash
# serve_hparams.json — the ENGINE configs to compare
cat > serve_hparams.json <<'JSON'
[
  {"_benchmark_name": "bf16-kv",  "max_num_seqs": 96, "max_num_batched_tokens": 4096},
  {"_benchmark_name": "fp8-kv",   "max_num_seqs": 96, "max_num_batched_tokens": 4096,
   "kv_cache_dtype": "fp8"}
]
JSON

# The workload explorer: it finds the latency/throughput frontier for you,
# rather than making you guess a concurrency ladder.
vllm bench sweep serve_workload \
  --serve-cmd 'vllm serve meta-llama/Llama-3.1-8B-Instruct --max-model-len 8192' \
  --bench-cmd 'vllm bench serve --model meta-llama/Llama-3.1-8B-Instruct \
     --backend vllm --endpoint /v1/completions \
     --dataset-name random --random-input-len 4096 --random-output-len 512 \
     --num-prompts 500 --goodput ttft:500 tpot:50' \
  --workload-var max_concurrency \
  --serve-params serve_hparams.json \
  --num-runs 3 \
  --output-dir results --experiment-name cpm-sweep
```

What `serve_workload` does, from the docs: run at concurrency 1 (lowest latency, lowest
throughput), run with everything at once (highest of both), estimate the workload variable
corresponding to that saturation point, then sample intermediate values uniformly with the
remaining iterations. `--workload-var max_concurrency` is recommended over `request_rate`
because it directly controls the load the engine sees.

Two properties that matter for honesty. The harness **starts the server once per
`--serve-params` entry** and reuses it across `--bench-params`, and **calls all
`/reset_*_cache` endpoints between benchmark runs** — which is how you avoid measuring your
prefix cache instead of your engine (07.4 §6). And `--num-runs` defaults to 3, because
single runs on a shared cloud GPU have more variance than people assume.

Then plot. The two figures the deliverable wants:

```bash
# (1) The latency/throughput trade with the SLO line — the classic knee plot.
vllm bench sweep plot results/cpm-sweep \
  --var-x total_token_throughput --var-y p99_ttft_ms \
  --col-by _benchmark_name --curve-by max_num_seqs \
  --fig-name latency_throughput

# (2) The Pareto frontier — the two axes that ARE the economics.
vllm bench sweep plot_pareto results/cpm-sweep \
  --label-by max_concurrency,tensor_parallel_size
```

`plot_pareto`'s axes deserve a moment, because they are the cost-vs-experience trade stated
in its natural units:

```
  x-axis:  tokens/s/USER  =  output_throughput ÷ concurrency     ← what a user FEELS
  y-axis:  tokens/s/GPU   =  output_throughput ÷ gpu_count       ← what you PAY FOR

  ┌─ tokens/s/GPU (∝ 1/CPM) ───────────────────────────────────────────────┐
  │                                                                        │
  │  3500┤                   ●B=192  ●B=256   ← cheap per token,           │
  │      │              ●B=128                  miserable per user         │
  │  3000┤          ●B=96  ← THE KNEE: on the frontier AND inside the SLO  │
  │      │      ●B=64                                                      │
  │  2000┤   ●B=32                                                         │
  │      │ ●B=8                                                            │
  │  1000┤                                                                 │
  │      │●B=1  ← lovely per user, catastrophic per GPU                    │
  │      └────────────────────────────────────────────────────────────────┤
  │        0    20    40    60    80   100   120   140  tokens/s/user      │
  │                                                                        │
  │  The Pareto frontier is the set of points where you cannot improve     │
  │  one axis without giving up the other. Your SLO is a VERTICAL LINE     │
  │  on this chart (a minimum tokens/s/user), and the operating point is   │
  │  the highest point on the frontier to the right of it.                 │
  └────────────────────────────────────────────────────────────────────────┘
```

### 11. Date every curve

A CPM curve is a measurement of a (model, engine version, GPU, kernel, workload) tuple at a
moment. Every one of those moves:

- **Hardware.** H100 → H200 is +43 % memory bandwidth at identical FLOPs — near-linear decode
  throughput on the same code.
- **Kernels and engine.** vLLM's own V0 → V1 rewrite reported up to 1.7× throughput on
  unchanged hardware and unchanged models, from scheduler and control-plane work.
- **Precision.** FP8 on Hopper (07.7) is a further ~1.5–1.8×.
- **Your own traffic.** A product change that doubles average prompt length halves your
  resident batch at fixed pool, and moves the knee left.

So: **put a date, a vLLM version, a GPU SKU, a model revision, and the workload shape on
every curve you publish**, and re-measure on a cadence — quarterly is reasonable for a
production fleet, and after every engine upgrade regardless. This is not pedantry; a
year-old CPM number used in a build-vs-buy decision is exactly as dangerous as a year-old
load test, and rather more expensive.

## Perspectives

**The FinOps view.** The sentence that lands in a director's meeting: *"our measured cost is
$0.26 per million output tokens at the SLO operating point, and $0.74 in production because
the fleet is 35 % utilised — the $0.48 gap is an autoscaling and multi-tenancy problem, not a
GPU problem."* That framing does two things at once: it gives an honest number, and it names
the owner of the delta. Utilisation is usually the biggest lever a platform team actually
controls day to day, because batch tuning is a one-time engineering task while utilisation is
a continuous operational one.

**The product view.** The knee is not an engineering constant; it is where a *product*
promise crosses a physics curve. Moving the TTFT SLO from 500 ms to 1 s moves the operating
point right and cuts CPM by roughly a tenth. Whether that trade is correct is a product
decision about what users will tolerate — for a streaming chat UI where the first token
starts a visible response, 500 ms is defensible; for an async summarisation job it is
theatre. The platform engineer's job is to put a *price* on each candidate SLO so the
decision is made with numbers, not with vibes. Bring the curve to that conversation, not an
opinion.

**The hardware-buyer view.** §3's derivation says decode is bandwidth-bound at every batch
you can realistically hold, which turns "which GPU should we buy" from a FLOPs comparison
into a bandwidth-per-dollar comparison. Compute `$/hr ÷ (GB/s)` across candidate SKUs and
you get a decode-cost proxy that is far more predictive than TFLOP/s. It also explains why an
inference fleet and a training fleet legitimately want different hardware, and why the
"we already have A100s" argument deserves an actual number rather than a shrug.

**The skeptic's view.** Every multiplier in this lesson has a workload dependency hiding in
it, and quoting them naked is a tell. Continuous batching's 23× is a function of
output-length *variance* and collapses to 1× for fixed-length outputs. The 50–100× CPM drop
is real but is nearly all earned by batch 32. FP8's 2× FLOPs becomes 1.5–1.8× throughput
because weights are only part of the bytes moved. The senior move is to state the multiplier
*and* the condition under which it holds — and to notice when someone else does not.

**The speculative-decoding counter-example (scoped out, worth knowing).** Folklore says
speculative decoding only helps at small batch, because at large batch the GPU is already
compute-saturated. §3 shows why that folklore is shakier than it sounds: at large batch
*and* long context the KV term dominates and decode is emphatically still bandwidth-bound,
which re-opens room for speculation. Together AI's long-context large-batch work
(MagicDec / Adaptive Sequoia Trees) measured up to ~2× throughput/latency improvement in
exactly that regime on 8×A100. This module scopes the technique out, but the lesson
generalises: "batch is large" does not automatically mean "no headroom left," and the
roofline argument that got you here is the same one that tells you where to look.

## Real-world use cases

- **Anyscale — continuous batching benchmarks on OPT-13B.** With realistic output-length
  variance, a naive static-batching baseline fell to **81 output tokens/second** on a 40 GB
  A100, while continuous batching with paged memory management reached up to **23×** that.
  **What it shows:** the multiplier is a property of *traffic shape*, not of the engine —
  with fixed-length outputs the two schemes are identical, and all of the win comes from
  variance. It also shows why "we do continuous batching" stopped being differentiating
  around 2023; the interesting question now is what your scheduler does under contention.

- **Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models"
  (OSDI 2022).** Introduced iteration-level scheduling — re-planning the batch after every
  forward pass rather than every request — and reported **36.9× throughput at matched
  latency** versus FasterTransformer on GPT-3 175B. **What it shows:** the idea that became
  "continuous batching" was a scheduling insight, published a year before PagedAttention, and
  the two are complementary rather than the same thing: Orca said *when* to change the batch,
  PagedAttention made it *possible* to change it cheaply. *(arxiv.org and usenix.org are
  blocked by this environment's egress proxy, so the paper could not be re-read directly; the
  mechanism and the headline figure are cross-checked against vLLM's in-tree documentation and
  the V1 scheduler source, which implements the same design.)*

- **vLLM's own sweep tooling — `vllm bench sweep serve_workload` and `plot_pareto`.** The
  project ships a workload explorer that walks the latency/throughput frontier automatically
  and a Pareto plotter whose axes are **tokens/s/user against tokens/s/GPU**, with GPU count
  derived as TP × PP × DP. **What it shows:** the two-axis framing in §10 is not this
  lesson's invention — it is the frame the engine's own maintainers chose for presenting the
  trade, which is a useful signal that "throughput" alone was never the right single number.
  It also means the deliverable's plots are a tool invocation rather than a plotting project.

- **The vLLM V0 → V1 engine rewrite.** V1 reported up to **1.7× the throughput of V0** on
  unchanged hardware and unchanged models, from control-plane work: pre-allocating block
  objects, threading the free list through them, an append-only block table, and a scheduler
  with no prefill/decode phases. **What it shows:** a CPM curve is a measurement of your
  *software* as much as your hardware, and a curve measured before an engine upgrade is a
  historical artefact. It is also the most concrete argument for §11's "date every curve"
  discipline: nobody changed a GPU, and the number moved 70 %.

## Worked example

**Build the CPM-vs-concurrency curve for Llama-3.1-8B on one H100, locate the SLO knee,
and defend the operating point.** This is the deliverable's component 1. Roughly 60 minutes
of GPU time on top of 07.4's tuned config.

SLOs: **TTFT p99 ≤ 500 ms**, **ITL p99 ≤ 50 ms**. Workload: 4,096-token prompts, 512-token
generations. Rate: $2.89/GPU-hr.

### Step 1 — predict the curve before renting anything

Using §2's model and §6's three caps, with `η = 0.67` as a starting guess:

```
  B_ITL  = (0.050 × 3.35e12 × 0.67 − 1.606e10) / (4352 × 131072)   = 168
  B_KV   = 460,800 tokens / 4,352 tokens per sequence               = 105
  B_TTFT = 96 + (500 ms / 21.6 ms − 1)/1.75                         ≈ 108 offered
  ⇒ binding cap: KV capacity at 105. Predicted operating point ≈ 96.
  ⇒ predicted throughput at B=96: 96 / 21.14 ms × 0.67              = 3,043 tok/s
  ⇒ predicted CPM: 2.89 / (3043 × 3600) × 1e6                       = $0.264 /1M
```

**Write these down before you boot.** A prediction you record and then miss teaches you
something; a prediction you make after seeing the answer teaches you nothing.

### Step 2 — sweep

```bash
pip install "vllm==0.27.1" && vllm --version

vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 8192 --max-num-seqs 256 --max-num-batched-tokens 4096 \
  --port 8000 &

for C in 1 8 16 32 64 96 128 192 256; do
  curl -s -X POST localhost:8000/reset_prefix_cache   # measure the ENGINE, not the cache
  vllm bench serve \
    --model meta-llama/Llama-3.1-8B-Instruct --base-url http://localhost:8000 \
    --dataset-name random --random-input-len 4096 --random-output-len 512 \
    --request-rate inf --max-concurrency "$C" --num-prompts $((C * 12)) \
    --goodput ttft:500 tpot:50 \
    --percentile-metrics ttft,tpot,itl --metric-percentiles 50,99 \
    --save-result --result-filename "sweep_c${C}.json"
done
```

`--max-num-seqs` is deliberately set high (256) so that `--max-concurrency` — the *offered*
load — is what varies. You are sweeping demand against a fixed engine, which is what a
production traffic ramp looks like.

One sample output block, at C = 96:

```
============ Serving Benchmark Result ============
Successful requests:                     1152
Maximum request concurrency:             96
Benchmark duration (s):                  191.24
Total input tokens:                      4718592
Total generated tokens:                  589824
Request throughput (req/s):              6.02
Request goodput (req/s):                 6.02
Output token throughput (tok/s):         3084.14
Peak output token throughput (tok/s):    3260.00
Peak concurrent requests:                96
Total token throughput (tok/s):          27751.26
---------------Time to First Token----------------
Median TTFT (ms):                        212.44
P99 TTFT (ms):                           441.09
-----Time per Output Token (excl. 1st token)------
Median TPOT (ms):                        30.91
P99 TPOT (ms):                           38.12
---------------Inter-token Latency----------------
Median ITL (ms):                         30.88
P99 ITL (ms):                            44.77
```

Note `Request goodput` equals `Request throughput` here: **every** request met both SLOs.
That equality is what "inside the SLO" looks like as a number rather than as an annotation.

### Step 3 — assemble the table

Extract with `jq` and compute CPM. Values below are representative for this configuration
and are the §2 model at `η = 0.67`, anchored on 07.4's two measured points — **replace every
row with your own run**:

```bash
for f in sweep_c*.json; do
  jq -r '[.max_concurrency, .output_throughput, .request_throughput, .request_goodput,
          .p99_ttft_ms, .p99_itl_ms] | @tsv' "$f"
done | sort -n | awk -v rate=2.89 'BEGIN{
  printf "%6s %10s %8s %8s %10s %9s %9s %9s\n",
    "conc","out tok/s","req/s","good/s","P99 TTFT","P99 ITL","CPM","goodCPM" }
{ cpm = rate/($2*3600)*1e6;
  gcpm = ($4>0) ? rate/($2*3600*($4/$3))*1e6 : 0;
  printf "%6d %10.0f %8.2f %8.2f %10.0f %9.1f %9.3f %9.3f\n",$1,$2,$3,$4,$5,$6,cpm,gcpm }'
```

| Concurrency | Output tok/s | req/s | goodput req/s | P99 TTFT | P99 ITL | CPM | Goodput-CPM |
|---|---|---|---|---|---|---|---|
| 1 | 135 | 0.26 | 0.26 | 158 ms | 8 ms | $5.947 | $5.947 |
| 8 | 872 | 1.70 | 1.70 | 178 ms | 10 ms | $0.921 | $0.921 |
| 16 | 1,320 | 2.58 | 2.58 | 201 ms | 12 ms | $0.608 | $0.608 |
| 32 | 2,093 | 4.09 | 4.09 | 264 ms | 16 ms | $0.384 | $0.384 |
| 64 | 2,731 | 5.33 | 5.33 | 348 ms | 24 ms | $0.294 | $0.294 |
| **96** | **3,084** | **6.02** | **6.02** | **441 ms** | **45 ms** | **$0.260** | **$0.260** |
| 128 | 3,225 | 6.30 | 4.41 | 662 ms | 51 ms | $0.249 | $0.356 |
| 192 | 3,410 | 6.66 | 1.13 | 1,105 ms | 63 ms | $0.235 | $1.385 |
| 256 | 3,545 | 6.92 | 0.21 | 1,588 ms | 72 ms | $0.226 | $7.451 |

### Step 4 — read it

**CPM falls 23× from concurrency 1 to 96** — $5.947 to $0.260. Of that fall, 95 % has
happened by concurrency 64. **Both SLOs are met up to 96 and both break at 128.**

Now look at the last two columns together, because that contrast is the whole lesson:

```
  Concurrency  96  →  256
    raw CPM:   $0.260 → $0.226      "15 % cheaper!"        ← the wrong reading
    goodput:   6.02   → 0.21 req/s   97 % of requests now MISS the SLO
    goodCPM:   $0.260 → $7.451       29× MORE EXPENSIVE per USEFUL token
```

**Raw CPM improves monotonically forever; goodput-CPM has a genuine minimum, and it is at the
knee.** That is why §7 argues for measuring goodput rather than annotating a throughput
plot: the raw curve invites you to keep pushing right, and the goodput curve tells you where
to stop.

### Step 5 — check the prediction, and account for the miss

| | Predicted (§1) | Measured | Error |
|---|---|---|---|
| Binding cap | KV capacity | KV capacity ✓ | — |
| Operating batch | 96 | 96 ✓ | — |
| Throughput at B=96 | 3,043 tok/s | 3,084 tok/s | +1.3 % |
| CPM at B=96 | $0.264 | $0.260 | −1.5 % |
| P99 ITL at B=96 | 31.5 ms (mean model) | 44.8 ms | model predicts the **mean**, not p99 |

The ITL row is the honest one: §5's equation predicts a *mean* step time, and p99 ITL runs
~40 % above it because of scheduling jitter, occasional prefill chunks landing in the same
iteration, and CUDA-graph size bucketing. **Use the closed forms to predict means and to size
the constraint; use measurement for percentiles.** A model that claims to predict a p99 from
a bandwidth number is lying.

### Step 6 — state the claim

The sentence the deliverable exists to support, with everything a reader needs to check it:

> On 1× H100-80GB at $2.89/hr, serving Llama-3.1-8B-Instruct (bf16 weights, bf16 KV) under
> vLLM 0.27.1 with `--max-num-seqs 96 --max-num-batched-tokens 4096 --max-model-len 8192`,
> at 4,096-in / 512-out, we sustain **3,084 output tok/s at 100 % goodput** against a
> 500 ms TTFT-p99 / 50 ms ITL-p99 SLO. That is **$0.260 per million output tokens at
> full utilisation** and **$0.743 at our measured 35 % fleet utilisation**. The binding
> constraint is **KV capacity, not latency** — the ITL SLO would permit a batch of 168 —
> so the next lever is `--kv-cache-dtype fp8`, which doubles the pool. Measured
> 2026-08-18; re-measure after any engine or model upgrade.

Every number in that paragraph is traceable to a command in this lesson. That is the
difference between a benchmark and an engineering claim.

## Practice

Rent one GPU (H100 or A100 80 GB ideal; an L40S or A10 works with a smaller model). Feeds
components 1 and 4 of the [cost-per-token deliverable](../practice/cost-per-token/README.md).
Pin the vLLM version in every command.

### 1. Predict before you measure

From `config.json` geometry, your GPU's published bandwidth, and your tuned config's
`GPU KV cache size`, compute all three caps from §6 (`B_ITL`, `B_KV`, `B_TTFT`) and predict
throughput and CPM at the binding one. Use `η = 0.67` as a first guess.

**Acceptance:** three cap values, the identification of which binds, and a predicted
throughput and CPM — **written down before the server starts.**

### 2. Sweep with goodput on

Run the concurrency ladder from the worked example with `--goodput ttft:<your SLO>
tpot:<your SLO>`, resetting the prefix cache between runs. At least seven points, spanning
from clearly-under to clearly-over saturation.

**Acceptance:** the sweep JSONs, plus the assembled table with `output_throughput`,
`request_throughput`, `request_goodput`, P99 TTFT, P99 ITL, CPM and goodput-CPM.

### 3. Plot it — the deliverable's flagship figure

Produce **CPM against concurrency with the goodput-CPM overlaid and the SLO crossing
marked.** Either `vllm bench sweep plot` or your own matplotlib; the requirement is that
someone who has never seen your run can read the operating point off the figure.

**Acceptance:** the figure, with axes labelled in units, the SLO stated in the caption, and
the knee annotated with its concurrency, throughput and CPM.

### 4. Calibrate η and check the model

Fit `η` from your measured throughput at two batch sizes, then use the §2 model to predict a
third point you have not yet run. Run it.

**Acceptance:** your fitted `η`, the out-of-sample prediction, the measurement, and the error.
If the error is above ~10 %, say what the model is missing for your setup (attention backend,
long-context kernels, CUDA-graph bucketing, TP collectives).

### 5. Move the knee, deliberately, twice

Re-run the sweep under each of:

- **`--kv-cache-dtype fp8`** — halves KV bytes, so it should move `B_KV` and `B_ITL` up by 2×.
- **A relaxed TTFT SLO** (1 s instead of 500 ms) — same run, recompute goodput from the saved
  per-request data or re-run with the new `--goodput`.

**Acceptance:** a three-row table (baseline, FP8-KV, relaxed-SLO) of operating concurrency,
throughput, CPM and goodput-CPM, with the predicted-versus-measured 2× for FP8 discussed —
including where reality falls short and why.

### 6. Measure your utilisation, do not assume it

Compute utilisation from Prometheus over your longest available window using either method
in §8, and produce the production CPM.

**Acceptance:** the PromQL, the utilisation figure with its window, and both CPM numbers —
benchmark and production — with the gap named as an autoscaling/multi-tenancy opportunity in
one sentence.

**Overall acceptance:** the CPM-vs-concurrency curve with the SLO knee identified from your
own GPU run, the operating point defended by the constraint analysis (which cap binds and
why), the utilisation-adjusted production CPM, and the six-line claim paragraph from Step 6
of the worked example — committed to the deliverable. This curve is the artefact the
[checkpoint](../checkpoint.md) asks you to hand over and defend.

## Common pitfalls

- **Quoting CPM without utilisation.** The single most common mis-pricing of self-hosted
  inference, and named explicitly in the module's fail signals. *Mechanism:* utilisation is a
  divisor in the denominator, so a 35 %-busy fleet costs 2.9× the benchmark number. Always
  state both figures and label which is which.

- **Using total token throughput instead of output token throughput.** *Mechanism:* prefill
  produces tokens roughly 10× more cheaply per token than decode (§9), so counting input
  tokens inflates throughput by the input:output ratio — for a 4000-in/512-out RAG workload
  that is 9×, and your CPM is 9× too good. `vllm bench serve` prints both; CPM uses
  `Output token throughput`.

- **Reporting peak instead of sustained throughput.** *Mechanism:* `Peak output token
  throughput` is a single best interval; you cannot bill against a maximum. Use
  `Output token throughput`, which is total generated tokens ÷ benchmark duration.

- **Sweeping with prefix caching on and no reset.** *Mechanism:* `enable_prefix_caching`
  defaults to `True`, so a sweep that replays prompts measures cache hits. Later runs look
  faster than earlier ones and the curve acquires a slope that is an artefact of ordering.
  `POST /reset_prefix_cache` between runs, or use `vllm bench sweep`, which does it for you.

- **Choosing the throughput-optimal point.** *Mechanism:* raw CPM decreases monotonically in
  batch, so it always argues for more load; the constraint that stops you is the SLO, which
  raw CPM cannot see. In the worked example, moving 96 → 256 saves 13 % of raw CPM while
  goodput collapses 97 % and goodput-CPM rises 29×. Optimise goodput-CPM.

- **Reporting a curve without the workload shape attached.** *Mechanism:* throughput is
  strongly a function of context length, because KV read per sequence per step is linear in
  it. The same GPU and model at 1k context versus 32k context produce curves that differ by
  more than a factor of five. A CPM number without an input/output length pair is not a
  number.

- **Assuming batching helps prefill.** *Mechanism:* prefill is already compute-bound at
  batch 1 — a 4,096-token prompt is a large GEMM with arithmetic intensity in the thousands,
  well right of the ridge point. Batching more prefills together adds latency without adding
  throughput. This is why "batching gives 50–100×" is a *decode* claim, and why
  prefill-dominated workloads (RAG, classification, extraction) see far less of it.

- **Comparing your CPM against an API price list without matching the terms.** *Mechanism:*
  API prices are per input *and* output token, include the provider's utilisation and margin,
  and often reflect a much larger model. Compare blended cost per *request* for your actual
  traffic shape (§9), not $/M against $/M.

- **Treating the curve as durable.** *Mechanism:* the V0 → V1 engine rewrite moved throughput
  1.7× with no hardware change. Date the curve, pin the versions, re-measure after upgrades.

## Self-check

**(a) Write the CPM equation, define every term, and compute it for an H100 at $2.89/hr
sustaining 3,084 output tok/s at 35 % utilisation.**

**Answer:** `effective_cpm = hourly_rate / (output_tok_per_sec × 3600 × utilization) × 1e6`.
`hourly_rate` is all-in $/GPU-hour (instance price plus storage, egress, control-plane share
— typically 1.15–1.3× the sticker). `output_tok_per_sec` is **sustained decode** tokens per
second summed across concurrent requests, excluding prefill tokens, because prefill is ~10×
cheaper per token and including it inflates the number by the input:output ratio. `3600`
carries the units. `utilization` is the fraction of wall-clock the fleet actually spends at
that throughput, measured over a week rather than a benchmark window. Computing:
`2.89 / (3084 × 3600 × 0.35) × 1e6 = 2.89 / 3,885,840 × 1e6 = $0.744` per million output
tokens. The full-utilisation figure is `2.89 / (3084 × 3600) × 1e6 = $0.260`; the 2.86× gap
between them is the autoscaling and multi-tenancy opportunity, and quoting the second without
the first is the module's named fail signal.

**(b) Why does CPM fall ~50–100× from batch 1 to a large batch, and why does the fall almost
entirely happen in the first part of that range?**

**Answer:** Decode is memory-bandwidth-bound: each step moves the *entire* weight set from
HBM regardless of batch size. At batch 1 you pay a full 16 GB weight read to produce one
token; at batch 32 the same read produces 32 tokens, so per-token bandwidth cost falls
roughly as `1/B`. That is the amortisation, and it is where the multiplier comes from. But
the step also reads every resident sequence's KV history, and *that* term is linear in B:
`bytes_per_step = W_bytes + B × ctx × kv_bytes_per_token`. Once the KV term exceeds the
weight term the amortisation is spent and throughput growth flattens. Concretely for
Llama-3.1-8B at 4,352-token context, the crossover is at `16.06 GB / 0.570 GB ≈ 28`
sequences — so past roughly batch 28 you are mostly paying for KV, and going 64 → 128 buys
only ~18 % more throughput. Hence 95 % of the CPM fall by batch 64. The headline 50–100× is
real but is earned early, which is exactly why the knee sits so far left of saturation.

**(c) Derive the largest batch your ITL SLO permits, for a model and GPU of your choosing,
and say what to do if that cap is the binding one.**

**Answer:** From `ITL = (W_bytes + B × ctx × kv_bytes_per_token) / (BW × η)`, invert:
`B ≤ (ITL_target × BW × η − W_bytes) / (ctx × kv_bytes_per_token)`. For Llama-3.1-8B on an
H100 (BW 3.35e12 B/s, η ≈ 0.67, W = 1.606e10 B, kv = 131,072 B/token) at ctx = 4,352 and a
50 ms target: byte budget `0.050 × 3.35e12 × 0.67 = 1.122e11`; minus weights `= 9.617e10`;
divided by `4352 × 131,072 = 5.704e8` gives **B ≤ 168**. If that is the *binding* cap — ITL
p99 over budget while the KV pool still has room — the levers are, in order: `--kv-cache-dtype
fp8` (halves the per-sequence KV read, roughly doubling the cap), a shorter effective context,
a higher-bandwidth SKU (H200 is +43 % bandwidth at identical FLOPs, so ~+43 % on the cap), or
renegotiating the SLO. Note what is *not* on that list: `--gpu-memory-utilization` and more
GPUs, neither of which changes a per-step bandwidth calculation.

**(d) Your CPM keeps improving as you raise concurrency. Why is that not a reason to keep
raising it, and what measurement makes the stopping point objective?**

**Answer:** Raw CPM is `rate / throughput`, and throughput is monotonically increasing in
batch, so raw CPM decreases forever — it will always argue for more load, right up to the
point where the service is unusable. The constraint that stops you is the latency SLO, which
raw CPM cannot see. **Goodput** makes it visible: the rate of requests that completed *and*
met every stated SLO, so a slow request counts as zero. `vllm bench serve --goodput
ttft:500 tpot:50` computes it, with keys limited to `ttft`/`tpot`/`e2el`, values in
milliseconds, and a request counting as good only if it satisfies *all* of them. Plot CPM
against goodput and the curve acquires a genuine minimum at the knee. In the worked example,
moving from concurrency 96 to 256 improved raw CPM 13 % ($0.260 → $0.226) while goodput fell
from 6.02 to 0.21 req/s and goodput-CPM rose 29× to $7.45 — the same run reading as a win on
one metric and a catastrophe on the other.

**(e) On an H100, at what batch size does decode stop being memory-bound, and why does the
answer matter more than the number?**

**Answer:** A decode GEMM does `2·P·B` FLOPs while moving `2·P` bytes of bf16 weights, so its
arithmetic intensity is exactly `B` FLOP/byte. The ridge is at the machine balance:
`989.4e12 FLOP/s ÷ 3.35e12 B/s ≈ 295 FLOP/byte`, so **B\* ≈ 295** on an H100 SXM at BF16
(≈206 on H200, ≈153 on A100, ≈591 on H100 at FP8, where compute doubles but bandwidth does
not). The number matters less than its comparison to the batches you can actually hold: with
a 460,800-token KV pool at 4,352-token contexts you can hold about 105 sequences, and your
ITL SLO permits 168. **Both are far below 295, so you never reach the ridge and decode is
memory-bound at every operating point you can realistically run.** Three consequences: buy
bandwidth rather than FLOPs for inference fleets ($/hr ÷ GB/s is a better decode-cost proxy
than TFLOP/s); FP8's throughput win is ~1.5–1.8× rather than 2× because only part of the
bytes moved are weights; and prefill sits on the *other* side of the ridge, which is why it
and decode want opposite policies and hardware.

**(f) Two teams share your platform: one runs chat (500-in / 500-out) and one runs RAG
(4,000-in / 200-out). You charge both per output token. What is wrong with that?**

**Answer:** Per-output-token pricing systematically under-charges the RAG team, because
prefill is not free — it is roughly 10× cheaper *per token* than decode, but RAG's prompts
are 8× longer and its generations are 2.5× shorter. Costing both in GPU-seconds at the B=96
operating point (prefill ≈ 30,800 tok/GPU-s, decode ≈ 3,088 tok/GPU-s): chat is
`500/30800 + 500/3088 = 0.0162 + 0.1619 = 0.178` GPU-s; RAG is
`4000/30800 + 200/3088 = 0.1299 + 0.0648 = 0.195` GPU-s. **RAG costs 10 % more per request
while generating 60 % fewer output tokens** — so per-output-token billing charges it about a
third of what chat pays for the same platform cost, and chat subsidises RAG. Fix it by
charging a blended input + output rate (as every commercial API does, typically 3–5× more for
output), and apply the prefix-cache discount to the input term where it applies:
`vllm:prefix_cache_hits_total ÷ vllm:prefix_cache_queries_total`, in tokens, is the fraction
of input tokens you did not actually compute. The subsidy grows as context windows do, so
this gets worse, not better.

## Connections & what's next

This lesson produced the module's flagship number and, more importantly, the method behind
it: predict the three caps from geometry and hardware, sweep to confirm, measure goodput so
the SLO is in the number rather than beside it, and divide by an honest utilisation. The
binding-cap analysis in §6 is the thread that ties the rest of the module together — if KV
capacity binds, 07.7's FP8 KV cache is your next lever and 07.6's disaggregation is the
structural version of the same fix; if utilisation is the gap, 07.8's autoscaling and 07.10's
multi-tenancy are where it closes; if TTFT binds while ITL is fine, 07.6's independent
prefill scaling is the answer. Backward, the curve is only meaningful because 07.4 produced a
preemption-free config to measure — a sweep against an oversubscribed engine measures the
misconfiguration, and its knee is an artefact.

**Next: [07.6 — Alternative servers and disaggregation](06-alternative-servers-disaggregation.md)**
asks what happens when one engine's single operating point is not enough: matching the engine
to the workload (vLLM / SGLang / TensorRT-LLM / KServe), and splitting prefill from decode
onto separately-scaled pools so the two phases — which §3 shows sit on opposite sides of the
roofline ridge — stop competing for the same budget.

## References & further reading

**Primary sources — vLLM (read at tag `v0.27.1`, cross-checked against `main` @ `c1e4387`, 2026-08-17)**

1. **`vllm/benchmarks/serve.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/benchmarks/serve.py — the harness this lesson's sweep is built on. Authoritative for: the printed result block (`Output token throughput` vs `Peak output token throughput` vs `Total token throughput`, `Request throughput` vs `Request goodput`, `Peak concurrent requests`); the `--goodput` semantics used in §7 (`VALID_NAMES = ["ttft", "tpot", "e2el"]`, values in ms, `is_good_req = all(...)` across all specified SLOs); and the full CLI surface including `--max-concurrency`, `--request-rate`, `--ramp-up-strategy`, `--percentile-metrics` and `--metric-percentiles`.
2. **`vllm/benchmarks/datasets/datasets.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/benchmarks/datasets/datasets.py — `--dataset-name` options (`random`, `sharegpt`, `burstgpt`, `sonnet`, `prefix_repetition`, `hf`, …) and the random-dataset controls used throughout: `--random-input-len` (default 1024), `--random-output-len` (default 128), `--random-range-ratio`, `--random-prefix-len`. The last one is how you construct a controlled prefix-cache-hit workload.
3. **`docs/benchmarking/sweeps.md`** — https://github.com/vllm-project/vllm/blob/main/docs/benchmarking/sweeps.md — `vllm bench sweep serve`, `serve_workload` (the four-step frontier-exploration algorithm quoted in §10, and the recommendation of `--workload-var max_concurrency` over `request_rate`), `plot`, and `plot_pareto` with its tokens/s/user × tokens/s/GPU axes and `gpu_count = TP × PP × DP`. Also documents `--num-runs` defaulting to 3 and the `/reset_*_cache` calls between runs. *(Read from `docs/` in the cloned repository; the rendered site at docs.vllm.ai is behind this environment's egress proxy.)*
4. **`docs/configuration/optimization.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/optimization.md — the chunked-prefill scheduling policy that §5's TTFT model depends on ("batches all pending decode requests before scheduling any prefill operations"), and the `max_num_batched_tokens` ITL-vs-TTFT guidance (smaller ≈2048 for ITL, >8192 for throughput).
5. **`vllm/v1/metrics/loggers.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/metrics/loggers.py — `vllm:generation_tokens_total`, `vllm:prompt_tokens_total`, `vllm:num_requests_running`, `vllm:prefix_cache_hits_total` / `vllm:prefix_cache_queries_total` (counted in **tokens**), used in §8's and §9's PromQL. Matches the verified set in [module 05 lesson 06](../../05-gpu-observability/lessons/06-inference-slos.md).

**Primary sources — hardware specifications**

6. **NVIDIA H100 datasheet (SXM5)** — https://resources.nvidia.com/en-us-tensor-core — 80 GB HBM3, **3.35 TB/s** memory bandwidth, **989.4 TFLOP/s** BF16 dense tensor (1,979 with 2:4 sparsity), **1,979 TFLOP/s** FP8 dense. These are the two numbers §3's ridge-point derivation divides. Verify against your exact SKU: PCIe variants have lower bandwidth and clocks, and power caps move both terms.
7. **NVIDIA H200 and A100 datasheets** — H200 SXM: 141 GB HBM3e, **4.8 TB/s**, same 989.4 TFLOP/s BF16 dense as H100 — the cleanest natural experiment for "decode is bandwidth-bound," since compute is held constant. A100 80 GB SXM: **2,039 GB/s**, 312 TFLOP/s BF16 dense. *(Vendor product pages are behind this environment's egress proxy; these are the published specification figures for these SKUs and are worth re-confirming on the datasheet PDF for the exact part you rent.)*

**Research and industry measurements**

8. **Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models," OSDI 2022** — https://www.usenix.org/conference/osdi22/presentation/yu — introduced iteration-level (continuous) batching and selective batching; reported 36.9× throughput at matched latency versus FasterTransformer on GPT-3 175B. §4's continuous-batching timeline is this design. *(usenix.org is blocked by this environment's egress proxy; the mechanism is cross-checked against vLLM's V1 scheduler source, which implements it, and against vLLM's in-tree documentation.)*
9. **Anyscale — "How continuous batching enables 23x throughput in LLM inference while reducing p50 latency"** — https://www.anyscale.com/blog/continuous-batching-llm-inference — the OPT-13B / 40 GB A100 measurements quoted in §4: a static baseline at 81 output tok/s under realistic length variance, and up to 23× from continuous batching plus paged memory management. Read it for the framing that the multiplier is a function of output-length variance, which is the caveat that must travel with the number. *(Blocked by this environment's egress proxy; the figures are as cited in vLLM's and the wider ecosystem's documentation of the result, and §4 derives the mechanism independently so the lesson does not depend on the citation.)*
10. **Zhong et al., "DistServe: Disaggregating Prefill and Decoding for Goodput-optimized LLM Serving"** — https://arxiv.org/pdf/2401.09670 — the source vLLM's own `--goodput` help text points at for the definition of goodput used in §7. Also the paper 07.6 builds on. *(arxiv.org is blocked by this environment's egress proxy; the goodput definition used here is taken from vLLM's implementation in `serve.py`, which is the artefact you will actually run.)*
11. **Introl — "Inference Unit Economics: True Cost Per Million Tokens"** — https://introl.com/blog/inference-unit-economics-true-cost-per-million-tokens-guide — the module README's named methodology source for `effective_cpm`, and the origin of the utilisation-trap framing in §8 (a GPU at 10 % load turning a cheap benchmark number into something worse than a commercial API). Vendor content; read it for the formula and the framing, not for benchmark figures.
12. **Character.AI — "Optimizing AI Inference at Character.AI"** and **"…Part Deux"** — https://blog.character.ai/optimizing-ai-inference-at-character-ai/ — ~20,000 QPS at under a cent per hour of conversation, and a **33×** serving-cost reduction since 2022 achieved by compounding architecture changes (MQA, hybrid attention horizons, cross-layer KV sharing), int8, caching and batching. **What it adds to this lesson:** batching alone is one lever measured once; a 33× sustained reduction is what happens when every lever in this module compounds continuously over years.

**Deeper dives**

13. **07.2 — KV cache as a concurrency problem** — [02-kv-cache-concurrency.md](02-kv-cache-concurrency.md) — the `kv_bytes_per_token = 2·L·n_kv·d·e` formula and the pool arithmetic that §2's KV term and §6's `B_KV` cap depend on, plus the levers table for moving that cap.
14. **Module 03 lesson 04 — Decode throughput: bandwidth ceilings, batching, and the prefill/decode split** — [../../03-gpu-hardware/lessons/04-decode-bandwidth-batching.md](../../03-gpu-hardware/lessons/04-decode-bandwidth-batching.md) — the roofline treatment behind §3, including the capacity-wall-versus-compute-wall diagnostic that tells you whether a throughput plateau is a KV-capacity problem (fixable with memory) or a compute problem (not).
