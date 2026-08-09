---
lesson: "07.5"
title: "Batching economics"
module: "07"
concept: "Batching economics"
status: not-started
est_time: "6h"
artifacts: []
---
# 07.5 · Batching economics

> **Concept.** Batch size is the single biggest lever on cost-per-token; the operating point is the batch where CPM is minimised subject to your TTFT p99 SLO.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Why this matters

You rent a GPU by the hour. The bill is fixed the moment the pod schedules — an
H100 at ~$2–4/hr on a neocloud (CoreWeave, Lambda, Baseten) burns the same
whether it serves 1 request or 200. So the entire economics of inference is a
**denominator game**: cost-per-million-tokens (CPM) is `hourly_rate` divided by
how many tokens you extract from that hour. Batching is how you fill the
denominator.

This is the lesson where the module's flagship artifact — the cost-per-token
curve — gets built. Everything before this (roofline in 03, TTFT/TPOT/queue-depth
in 05-observability) was instrumentation. Here you turn those signals into a
single dollar number a director will actually read, and you find the **knee**:
the point past which pushing more load stops saving money and starts breaking
your latency SLO. Being the person on the platform team who can draw that curve
and name the operating point — not "vLLM go brrr" — is the FinOps differentiator
that gets you hired at a GPU-heavy shop.

From 03 you already know decode is **memory-bandwidth-bound**: each token reads
the entire weight matrix from HBM, and at batch 1 you read all those weights to
produce one token for one user. That is the roofline's memory-bound floor, and
it is catastrophically wasteful. Batching is the fix, and this lesson quantifies
exactly how much it is worth.

## What's new here

Three ideas that were not in the hardware or observability lessons:

1. **Continuous batching (a.k.a. in-flight / iteration-level batching).** Classic
   static batching waits to assemble N requests, runs them lockstep, and every
   request in the batch waits for the slowest to finish. vLLM instead schedules
   at the **iteration** (per-token) level: a finished sequence leaves the batch
   and a queued one joins on the very next forward pass. This is why real vLLM
   throughput scales with load far better than a naive batch would suggest — the
   GPU never idles waiting for stragglers. PagedAttention (lesson 03) is what
   makes it possible: KV blocks are non-contiguous, so admitting/evicting a
   sequence mid-flight is cheap.

2. **The arithmetic-intensity flip.** At batch 1 the GEMM in each decode step is
   effectively a matrix-vector product — you move ~2 bytes of weight per FLOP,
   pinned to the memory roof. As batch grows the same weights are reused across
   B sequences, so bytes-moved-per-FLOP falls ~1/B until you hit the **compute
   roof**. Throughput (tokens/sec) climbs steeply, then flattens. CPM is the
   inverse of that curve, so it falls steeply then flattens.

3. **Cost as a first-class metric.** Observability gave you TTFT/TPOT/queue-depth.
   None of those is a dollar. CPM is the bridge. And CPM has a trap: it is
   **meaningless without a utilisation figure**, because the denominator assumes
   the GPU is actually busy at that throughput. A benchmark peak at 100%
   utilisation is a marketing number; your 40-cluster fleet runs at 30–50%.

## Core notes

### The equation

```
effective_cpm = hourly_rate / (peak_output_tokens_per_sec × 3600 × utilization) × 1e6
```

- `hourly_rate` — all-in $/GPU-hr. Use the *rented* rate you actually pay, and
  if you care about truth, load it: add storage, egress, and idle time. For a
  single-GPU pod at $2.50/hr, use 2.50.
- `peak_output_tokens_per_sec` — measured **output** (decode) tokens/sec at the
  operating batch, summed across all concurrent requests. Output only: prefill
  tokens are near-free per-token and double-counting them flatters CPM.
- `3600` — seconds per hour.
- `utilization` — fraction of wall-clock the GPU spends at that throughput.
  This is the honesty knob. At 1.0 you are quoting a benchmark; at 0.35 you are
  quoting production.
- `× 1e6` — per **million** tokens, the unit everyone prices in.

Worked instance: H100 at $2.50/hr, 4,000 output tok/s sustained, utilization 1.0:

```
2.50 / (4000 × 3600 × 1.0) × 1e6 = 2.50 / 14,400,000 × 1e6 = $0.174 / 1M tokens
```

Now the honest version at 35% utilization:

```
2.50 / (4000 × 3600 × 0.35) × 1e6 = $0.496 / 1M tokens
```

Same GPU, same model, **2.85× the cost** — purely because the fleet is half-idle.
This is why CPM without utilisation is a lie, and why autoscaling (lesson 08)
exists: it drags utilisation back up toward the benchmark number.

### Why batch 1 → 256 cuts CPM 50–100×

At batch 1, decode reads the full weight set from HBM to emit one token: you are
paying for ~14 TB/s of bandwidth to produce a trickle. Numerically a 7–8B model
on one H100 does maybe ~80–150 output tok/s at batch 1. Push concurrency to 256
and continuous batching amortises each weight read across ~256 sequences; the
same GPU sustains several thousand output tok/s before the compute roof and KV
cache pressure cap it. Throughput up ~50–100×, so CPM (its inverse) down 50–100×.

The ceiling is not infinite. Two walls stop you:

- **KV-cache capacity.** Each concurrent sequence holds a KV footprint that grows
  with context length (lesson 02). At long context you run out of HBM for KV
  before you run out of compute, so max feasible batch drops. This is why CPM
  curves are always quoted at a stated input/output length.
- **The compute roof.** Once GEMMs are compute-bound, more batch adds latency
  without adding throughput. The curve flattens; CPM stops improving.

### The knee: throughput vs. latency

Here is the tension the whole lesson turns on. Throughput keeps rising past the
point where latency has already blown your SLO. As batch grows:

- **TTFT** rises because a new request waits behind a fuller queue and larger
  prefill batches (queue-depth from lesson 05-observability is the leading
  indicator here).
- **TPOT** rises because each decode step now serves more sequences and takes
  longer per iteration.

So there are two operating points and they do not coincide:

- **Throughput-optimal** — the flat top of the tokens/sec curve. Cheapest CPM in
  isolation.
- **SLO-optimal (the knee)** — the largest batch where TTFT p99 still sits under
  your SLO (the brief uses **500 ms**). *This* is your operating point.

The knee is almost always to the **left** of throughput saturation. Running at
throughput-optimal means you have already violated latency for the sake of a CPM
number nobody can ship. The engineering judgement — and the thing you annotate
on the deliverable — is naming the knee and running just below it.

### Prefill vs. decode in the cost picture

Recall from 03/05 that a request has two phases: prefill (compute-bound, one big
parallel pass over the prompt, sets TTFT) and decode (memory-bound, one token at
a time, sets TPOT). They batch differently. Prefill saturates compute at small
batch; decode needs large batch to escape the memory roof. Mixing long prefills
into a decode batch spikes TTFT for everyone in it (head-of-line blocking) — this
is chunked-prefill's whole reason to exist, and it is the conceptual seed of
disaggregated serving in lesson 06. For now: know that a workload with long
prompts hits the TTFT knee *earlier*, so its cost-optimal batch is smaller.

## Worked example

Target: a single H100 (80 GB) at **$2.50/hr**, serving an 8B model, SLO **TTFT
p99 ≤ 500 ms**, input 512 / output 128 tokens. Sweep concurrency and build both
curves. Illustrative numbers (yours will differ — that's the point of measuring):

| Concurrency | Output tok/s | TTFT p50 | TTFT p99 | CPM @ util=1.0 |
|-------------|-------------|----------|----------|----------------|
| 1           | 110         | 90 ms    | 130 ms   | $6.31          |
| 8           | 780         | 110 ms   | 190 ms   | $0.89          |
| 32          | 2,400       | 180 ms   | 340 ms   | $0.29          |
| 64          | 3,600       | 260 ms   | **480 ms** | $0.19        |
| 128         | 4,300       | 520 ms   | **910 ms** | $0.16        |
| 256         | 4,600       | 980 ms   | 1,700 ms | $0.15          |

Read it: CPM falls from **$6.31 → $0.15**, ~42× across this range (a wider
model/GPU pairing hits 50–100×). But TTFT p99 crosses 500 ms **between
concurrency 64 and 128**. At 64, p99 is 480 ms — under SLO — and CPM is $0.19.
At 128, CPM improves only to $0.16 (a 16% saving) while p99 nearly **doubles to
910 ms**, blowing the SLO.

**The knee is concurrency ≈ 64.** You buy 97% of the available cost improvement
while staying inside latency. Pushing to 128–256 chases a rounding error on cost
and sacrifices the product. That is the annotation the deliverable exists to make.

CPM sample check at concurrency 64: `2.50 / (3600 × 3600 × 1.0) × 1e6 = $0.193`.
At a realistic 40% fleet utilisation the operating-point CPM is `$0.193 / 0.40 =
$0.48` — quote *that* to finance, and note the gap as the autoscaling opportunity.

## Practice

Rent one GPU (Lambda / RunPod / CoreWeave; an H100 or A100 80 GB is ideal, an
L40S/A10 works for a smaller model). Feed the deliverable at
`../practice/cost-per-token/`.

**1. Serve.**
```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8000 --max-num-seqs 256 --disable-log-requests
```

**2. Sweep with `vllm bench serve`.** Run one invocation per operating point,
holding input/output length fixed and stepping `--max-concurrency`. Using
`--request-rate inf` with a capped `--max-concurrency` gives a clean saturated
point at each batch:
```bash
for C in 1 8 16 32 64 96 128 192 256; do
  vllm bench serve \
    --backend vllm \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --base-url http://localhost:8000 \
    --dataset-name random \
    --random-input-len 512 --random-output-len 128 \
    --num-prompts 500 \
    --request-rate inf \
    --max-concurrency "$C" \
    --percentile-metrics ttft,tpot,itl \
    --metric-percentiles 50,99 \
    --save-result --result-filename "sweep_c${C}.json"
done
```
Key flags: `--max-concurrency` sets the batch/in-flight cap (the x-axis);
`--request-rate` sets arrival rate (`inf` = as fast as the server accepts);
`--random-input-len`/`--random-output-len` pin the token shape so CPM is
comparable across points; `--num-prompts` sets sample size; `--save-result`
writes JSON with `output_throughput`, `ttft` (p50/p99), and `tpot`.

**3. Compute CPM per point.** For each JSON, pull `output_throughput`
(output tok/s) and apply the equation with your real `hourly_rate`. Start at
`utilization=1.0` for the raw curve, then add a second series at your fleet's
real utilisation (0.3–0.5) to show the honest operating cost.

**4. Plot both curves** (tokens/sec-vs-concurrency and CPM-vs-concurrency),
overlay TTFT p99, and draw a horizontal line at the 500 ms SLO. Annotate the
concurrency where p99 crosses it — that vertical is the knee.

**Acceptance.** A `cpm_vs_batch.(png|svg)` with: CPM on the y-axis, batch/
concurrency on the x-axis, the SLO knee marked with the concurrency value and
its CPM, and a caption stating GPU, `hourly_rate`, model, input/output length,
and utilisation. This chart is the centrepiece of the cost-per-token deliverable.
Also commit the raw `sweep_c*.json` and a small `compute_cpm.py`.

## Self-check

**(a) Why does cost per million tokens fall 50–100× from batch 1 to batch 256?**

**Answer:** The GPU-hour cost is fixed, so CPM is the inverse of tokens
extracted per hour. At batch 1 decode is memory-bandwidth-bound (lesson 03):
the full weight set is read from HBM to emit a single token, wasting almost all
compute. Continuous batching amortises each weight read across up to ~256
concurrent sequences, so output tokens/sec climbs ~50–100× before the KV-cache
and compute roofs cap it. Throughput up 50–100× → CPM (its inverse) down 50–100×.

**(b) At what batch/concurrency does YOUR TTFT p99 cross 500 ms, and why is that
the operating point?**

**Answer:** Read it off your sweep — in the worked example it lands between
concurrency 64 and 128 (p99 480 ms → 910 ms). It is the operating point because
throughput-optimal (the flat top of the tok/s curve) sits well to the right of
it, so running there would violate the latency SLO for a marginal CPM gain. The
knee — the largest batch with TTFT p99 still under 500 ms — captures nearly all
the cost improvement while keeping the product shippable. Cost is minimised
*subject to* the SLO constraint, not unconstrained.

**(c) Why is CPM meaningless without stating utilisation?**

**Answer:** The denominator (`peak_tok/s × 3600 × utilization`) assumes the GPU
sustains that throughput for the whole hour. A benchmark quotes utilization=1.0
(100% busy); a real fleet runs at 30–50%. At 35% utilisation the same GPU and
model cost ~2.85× the benchmark CPM. Quoting CPM without utilisation compares a
saturated lab number against a half-idle production reality — off by 2–3×. The
gap between benchmark CPM and fleet CPM is precisely the autoscaling ROI
(lesson 08).

## Resources

1. **Introl — "Inference Unit Economics: True Cost per Million Tokens"** —
   the deep dive on building the CPM number honestly (utilisation, all-in rate,
   token accounting). <https://introl.com/blog/inference-unit-economics-true-cost-per-million-tokens-guide>
2. **vLLM — `vllm bench serve` CLI docs** — canonical flag reference for the
   sweep; confirm `--max-concurrency`, `--request-rate`, `--random-*-len`,
   `--percentile-metrics`, `--save-result`.
   <https://docs.vllm.ai/en/latest/cli/bench/serve/>
