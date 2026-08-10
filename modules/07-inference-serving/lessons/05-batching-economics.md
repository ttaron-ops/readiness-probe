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
sources: 6
---
# 07.5 · Batching economics: the cost-per-token curve

> **Concept.** Batch size is the single biggest lever on cost-per-token; the operating point is the batch where CPM is minimised subject to your TTFT p99 SLO.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

Lesson 04 tuned a single vLLM instance to the edge of its HBM budget without falling off — `gpu-memory-utilization`, `max-num-seqs`, `max-model-len`, and what happens when you push past them (preemption). That lesson answered "how much of the GPU can I safely use." This lesson answers the question that pays your salary: "given that safe envelope, what is the actual dollar cost of a token, and where inside that envelope should I run?" The gap it fills is turning a tuned config into a defensible number — the CPM-vs-batch curve is the artifact a director, a finance partner, or an interview panel will actually look at, and naming the SLO knee on that curve is the single most senior-sounding thing you can do in this module.

## Why this matters

You rent a GPU by the hour. The bill is fixed the moment the pod schedules — an H100 at ~$2–4/hr on a neocloud (CoreWeave, Lambda, Baseten) burns the same whether it serves 1 request or 200. So the entire economics of inference is a **denominator game**: cost-per-million-tokens (CPM) is `hourly_rate` divided by how many tokens you extract from that hour. Batching is how you fill the denominator.

This is the lesson where the module's flagship artifact — the cost-per-token curve — gets built. Everything before this (roofline in module 03, TTFT/TPOT/queue-depth in module 05, preemption and HBM budgeting in lesson 04) was instrumentation and safety. Here you turn those signals into a single dollar number a director will actually read, and you find the **knee**: the point past which pushing more load stops saving money and starts breaking your latency SLO.

The stakes are concrete and career-shaped. "Design an LLM inference platform" is the canonical interview question for this job family at CoreWeave, NVIDIA, and Anthropic, and its sharpest sub-question is almost always some version of "what's your cost per million tokens, and how did you pick the operating point?" Answering with a single throughput-at-saturation number is a tell that you've never actually shipped this — it ignores the SLO, ignores utilization, and ignores that the curve you measured today decays as hardware and software improve. Being the person on the platform team who can draw the curve, name the knee, and state the utilization caveat — not "vLLM go brrr" — is the FinOps differentiator that gets you hired at a GPU-heavy shop, and it is exactly what the checkpoint at the end of this module will ask you to defend from memory.

From module 03 you already know decode is **memory-bandwidth-bound**: each token reads the entire weight matrix from HBM, and at batch 1 you read all those weights to produce one token for one user. That is the roofline's memory-bound floor, and it is catastrophically wasteful. Batching is the fix, and this lesson quantifies exactly how much it is worth — and exactly where it stops being worth it.

## What's new here (calibration)

You already have prefill/decode, the roofline, memory-bound decode, the KV cache concept, and FP8 from module 03; and TTFT/TPOT/ITL/queue-depth from module 05. Lesson 04 gave you the HBM-budget mechanics and preemption. None of that is re-taught here. Three ideas are genuinely new in this lesson:

1. **Continuous batching (a.k.a. in-flight / iteration-level batching).** Classic static batching waits to assemble N requests, runs them lockstep, and every request in the batch waits for the slowest to finish. vLLM instead schedules at the **iteration** (per-token) level: a finished sequence leaves the batch and a queued one joins on the very next forward pass. This is why real vLLM throughput scales with load far better than a naive batch would suggest — the GPU never idles waiting for stragglers. PagedAttention (lesson 03) is what makes it possible: KV blocks are non-contiguous, so admitting/evicting a sequence mid-flight is cheap.

2. **The arithmetic-intensity flip.** At batch 1 the GEMM in each decode step is effectively a matrix-vector product — you move ~2 bytes of weight per FLOP, pinned to the memory roof. As batch grows the same weights are reused across B sequences, so bytes-moved-per-FLOP falls ~1/B until you hit the **compute roof**. Throughput (tokens/sec) climbs steeply, then flattens. CPM is the inverse of that curve, so it falls steeply then flattens.

3. **Cost as a first-class metric, with a trap.** Observability gave you TTFT/TPOT/queue-depth. None of those is a dollar. CPM is the bridge. And CPM has a trap: it is **meaningless without a utilisation figure**, because the denominator assumes the GPU is actually busy at that throughput. A benchmark peak at 100% utilisation is a marketing number; your 40-cluster fleet runs at 30–50%.

Out of scope here, flagged so you know the boundary: speculative decoding (a further throughput lever layered on top of batching) is not covered in this module — see the forward pointer in Perspectives below.

## Core concepts

### The equation

```
effective_cpm = hourly_rate / (peak_output_tokens_per_sec × 3600 × utilization) × 1e6
```

- `hourly_rate` — all-in $/GPU-hr. Use the *rented* rate you actually pay, and if you care about truth, load it: add storage, egress, and idle time. For a single-GPU pod at $2.50/hr, use 2.50.
- `peak_output_tokens_per_sec` — measured **output** (decode) tokens/sec at the operating batch, summed across all concurrent requests. Output only: prefill tokens are near-free per-token and double-counting them flatters CPM.
- `3600` — seconds per hour.
- `utilization` — fraction of wall-clock the GPU spends at that throughput. This is the honesty knob. At 1.0 you are quoting a benchmark; at 0.35 you are quoting production.
- `× 1e6` — per **million** tokens, the unit everyone prices in.

Worked instance: H100 at $2.50/hr, 4,000 output tok/s sustained, utilization 1.0:

```
2.50 / (4000 × 3600 × 1.0) × 1e6 = 2.50 / 14,400,000 × 1e6 = $0.174 / 1M tokens
```

Now the honest version at 35% utilization:

```
2.50 / (4000 × 3600 × 0.35) × 1e6 = $0.496 / 1M tokens
```

Same GPU, same model, **2.85× the cost** — purely because the fleet is half-idle. This is why CPM without utilisation is a lie, and why autoscaling (lesson 08) exists: it drags utilisation back up toward the benchmark number.

### Why batch 1 → 256 cuts CPM 50–100×

At batch 1, decode reads the full weight set from HBM to emit one token: you are paying for ~14 TB/s of bandwidth to produce a trickle. Numerically a 7–8B model on one H100 does maybe ~80–150 output tok/s at batch 1. Push concurrency to 256 and continuous batching amortises each weight read across ~256 sequences; the same GPU sustains several thousand output tok/s before the compute roof and KV cache pressure cap it. Throughput up ~50–100×, so CPM (its inverse) down 50–100×.

The ceiling is not infinite. Two walls stop you:

- **KV-cache capacity.** Each concurrent sequence holds a KV footprint that grows with context length (module 03 / lesson 02). At long context you run out of HBM for KV before you run out of compute, so max feasible batch drops. This is why CPM curves are always quoted at a stated input/output length. Lesson 04's HBM-budget arithmetic (`gpu-memory-utilization`, `max-model-len`) is exactly what sets this wall's position.
- **The compute roof.** Once GEMMs are compute-bound, more batch adds latency without adding throughput. The curve flattens; CPM stops improving.

### The knee: throughput vs. latency

Here is the tension the whole lesson turns on. Throughput keeps rising past the point where latency has already blown your SLO. As batch grows:

- **TTFT** rises because a new request waits behind a fuller queue and larger prefill batches (queue-depth from module 05 is the leading indicator here).
- **TPOT** rises because each decode step now serves more sequences and takes longer per iteration.

So there are two operating points and they do not coincide:

- **Throughput-optimal** — the flat top of the tokens/sec curve. Cheapest CPM in isolation.
- **SLO-optimal (the knee)** — the largest batch where TTFT p99 still sits under your SLO (the brief uses **500 ms**). *This* is your operating point.

The knee is almost always to the **left** of throughput saturation. Running at throughput-optimal means you have already violated latency for the sake of a CPM number nobody can ship. The engineering judgement — and the thing you annotate on the deliverable — is naming the knee and running just below it.

### Prefill vs. decode in the cost picture

Recall from module 03 / module 05 that a request has two phases: prefill (compute-bound, one big parallel pass over the prompt, sets TTFT) and decode (memory-bound, one token at a time, sets TPOT). They batch differently. Prefill saturates compute at small batch; decode needs large batch to escape the memory roof. Mixing long prefills into a decode batch spikes TTFT for everyone in it (head-of-line blocking) — this is chunked-prefill's whole reason to exist, and it is the conceptual seed of disaggregated serving in the next lesson. For now: know that a workload with long prompts hits the TTFT knee *earlier*, so its cost-optimal batch is smaller.

### Why CPM curves decay over time — and why that's fine

A curve you measure today is a **snapshot**, not a permanent fact. New GPU generations (Hopper → Blackwell), better kernels (FlashAttention revisions, better GEMM scheduling), and better serving software (vLLM point releases) all shift the curve down and to the right — more throughput at the same batch, or the same throughput at a smaller batch. Vendors and cost-tracking write-ups have observed roughly an order-of-magnitude CPM improvement per year at the frontier. This is not a reason to skip measuring; it is a reason to **date every curve you publish** and re-measure on a cadence (quarterly is reasonable for a production fleet), the same way you would not trust a year-old load-test number for a web service.

## Perspectives

**The FinOps / economics view.** The one-liner that lands in a director's meeting or an interview: *"a GPU at 10% utilization turns $0.013 per thousand tokens into $0.13 — 10× the sticker cost, worse than most commercial APIs."* Utilization, not raw throughput, is usually the biggest lever a platform team actually controls day to day, because batch-size tuning is a one-time engineering task but utilization is a continuous operational one (traffic shaping, autoscaling, bin-packing). When someone asks "why is our self-hosted inference more expensive than the API," utilization is the first thing to check, not the model or the batch size.

**The API-provider pricing view.** Your self-hosted CPM doesn't exist in a vacuum — it competes against published API pricing. As a *dated 2025 snapshot*, published per-token API pricing roughly bands into budget/open-weight tiers (~$0.06–0.30 per million tokens), mid-tier proprietary models (~$0.55–15/M), and frontier models (~$15–75/M). These numbers move fast and should never be quoted without a date, but they give your self-hosted number market context: a self-hosted $0.19/M-token result (this lesson's worked example) is competitive with budget-tier API pricing *before* accounting for your own utilization discount — which is exactly the comparison a director will make, so know it before they ask.

**The production-scale compounding view.** Batching alone — going from batch 1 to a well-chosen operating batch — typically buys 50–100× off the batch-1 floor, as this lesson shows. That is a single lever, measured once. Character.AI's public account of cutting serving cost 33× over roughly two years is what happens when *every* lever compounds simultaneously and continuously: architecture changes (attention/KV cache innovations), quantization, batching, and utilization improvements stacked release over release. Treat this lesson's single CPM curve as one input to that larger, ongoing campaign — not as the whole campaign.

**The speculative-decoding-at-large-batch view (a scoped-out counter-intuition).** Folklore says speculative decoding — using a small draft model to propose tokens a large model verifies in parallel — only helps at *small* batch, because at large batch the GPU is already compute-saturated and there's no spare capacity to "spend" on speculation. Together AI's published work on long-context, large-batch serving found the opposite in a specific regime: at large batch **and** long context, decode becomes memory-bound again — the KV cache read dominates — which re-opens room for speculative decoding to help, up to ~2× throughput/latency improvement in their measurements on 8×A100s. This module scopes speculative decoding out, but the finding is worth carrying: "batch is large" does not automatically mean "no more free lunches," and the roofline argument that got you here (module 03) is the same one that explains why.

## Real-world use cases

- **Introl — "Inference Unit Economics: True Cost Per Million Tokens"** (2025). This module's own methodology source, and it holds up: it works through utilization-adjusted CPM and shows a GPU at 10% load turning $0.013/1K tokens into $0.13/1K — worse than several commercial APIs. What it shows: the utilization trap this lesson's equation is built to catch is not theoretical, it is the single most common way teams mis-price self-hosted inference. <https://introl.com/blog/inference-unit-economics-true-cost-per-million-tokens-guide>

- **Character.AI — "Optimizing AI Inference at Character.AI"** (2024) and the follow-up "Part Deux." Character.AI serves roughly 20,000 queries/sec — about a fifth of Google Search's request volume — at under a cent per hour of conversation, and reports a 33× serving-cost reduction since 2022 (13.5× cheaper than building on the most efficient commercial APIs). What it shows: batching is one lever in a much longer campaign; the compounding of architecture, KV-cache technique, and quantization changes over multiple years is what turns a 50–100× single-lever win into a 33× *sustained* one. <https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/>

- **Together AI — "Speculative decoding for high-throughput long-context inference"** (2024). Their MagicDec / Adaptive Sequoia Trees work measured up to 2× throughput/latency improvement from speculative decoding specifically in the large-batch, long-context regime — contradicting the standard assumption that speculative decoding only pays off at small batch. What it shows: the "batch size vs. lever value" relationship isn't monotonic once you introduce a second axis (context length); it's a genuine counter-example worth knowing exists, even though this module scopes the technique out. <https://www.together.ai/blog/speculative-decoding-for-high-throughput-long-context-inference>

## Worked example

Target: a single H100 (80 GB) at **$2.50/hr**, serving an 8B model, SLO **TTFT p99 ≤ 500 ms**, input 512 / output 128 tokens. Sweep concurrency and build both curves. Illustrative numbers (yours will differ — that's the point of measuring):

| Concurrency | Output tok/s | TTFT p50 | TTFT p99 | CPM @ util=1.0 |
|-------------|-------------|----------|----------|----------------|
| 1           | 110         | 90 ms    | 130 ms   | $6.31          |
| 8           | 780         | 110 ms   | 190 ms   | $0.89          |
| 32          | 2,400       | 180 ms   | 340 ms   | $0.29          |
| 64          | 3,600       | 260 ms   | **480 ms** | $0.19        |
| 128         | 4,300       | 520 ms   | **910 ms** | $0.16        |
| 256         | 4,600       | 980 ms   | 1,700 ms | $0.15          |

Read it: CPM falls from **$6.31 → $0.15**, ~42× across this range (a wider model/GPU pairing hits 50–100×). But TTFT p99 crosses 500 ms **between concurrency 64 and 128**. At 64, p99 is 480 ms — under SLO — and CPM is $0.19. At 128, CPM improves only to $0.16 (a 16% saving) while p99 nearly **doubles to 910 ms**, blowing the SLO.

**The knee is concurrency ≈ 64.** You buy 97% of the available cost improvement while staying inside latency. Pushing to 128–256 chases a rounding error on cost and sacrifices the product. That is the annotation the deliverable exists to make.

CPM sample check at concurrency 64: `2.50 / (3600 × 3600 × 1.0) × 1e6 = $0.193`. At a realistic 40% fleet utilisation the operating-point CPM is `$0.193 / 0.40 = $0.48` — quote *that* to finance, and note the gap as the autoscaling opportunity (lesson 08).

Sanity-check against the market snapshot from Perspectives above: $0.48/M at 40% utilization sits comfortably inside the budget-to-mid API band (~$0.06–15/M, 2025 pricing) — a legitimate "cheaper than the API" story, but only once utilization is included, and only for as long as that pricing snapshot holds.

## Practice

Rent one GPU (Lambda / RunPod / CoreWeave; an H100 or A100 80 GB is ideal, an L40S/A10 works for a smaller model). Feed the deliverable at `../practice/cost-per-token/`.

**1. Serve.**
```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8000 --max-num-seqs 256 --disable-log-requests
```

**2. Sweep with `vllm bench serve`.** Run one invocation per operating point, holding input/output length fixed and stepping `--max-concurrency`. Using `--request-rate inf` with a capped `--max-concurrency` gives a clean saturated point at each batch:
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
Key flags: `--max-concurrency` sets the batch/in-flight cap (the x-axis); `--request-rate` sets arrival rate (`inf` = as fast as the server accepts); `--random-input-len`/`--random-output-len` pin the token shape so CPM is comparable across points; `--num-prompts` sets sample size; `--save-result` writes JSON with `output_throughput`, `ttft` (p50/p99), and `tpot`.

**3. Compute CPM per point.** For each JSON, pull `output_throughput` (output tok/s) and apply the equation with your real `hourly_rate`. Start at `utilization=1.0` for the raw curve, then add a second series at your fleet's real utilisation (0.3–0.5) to show the honest operating cost.

**4. Plot both curves** (tokens/sec-vs-concurrency and CPM-vs-concurrency), overlay TTFT p99, and draw a horizontal line at the 500 ms SLO. Annotate the concurrency where p99 crosses it — that vertical is the knee.

**5. (Stretch) Re-run the sweep at a second input/output shape** — e.g. 4,000/200 to approximate a RAG workload — and confirm the knee moves. This is the fastest way to internalise the "curves don't transfer across workload shapes" pitfall below, and it's a strong deliverable addition if you have GPU budget left.

**Acceptance.** A `cpm_vs_batch.(png|svg)` with: CPM on the y-axis, batch/concurrency on the x-axis, the SLO knee marked with the concurrency value and its CPM, and a caption stating GPU, `hourly_rate`, model, input/output length, and utilisation. This chart is the centrepiece of the cost-per-token deliverable. Also commit the raw `sweep_c*.json` and a small `compute_cpm.py`.

## Common pitfalls

- **Quoting CPM without stating utilization.** The lesson's own headline point. `effective_cpm` at utilization=1.0 is a benchmark number; a production fleet runs 30–50%. Always publish both, or publish the fleet number and label the benchmark number "theoretical peak."
- **Comparing self-hosted CPM to API sticker price without normalizing for utilization.** A benchmark-peak self-hosted number will always look cheaper than an API — that's not a fair comparison. Introl's 10%-utilization example ($0.013/1K → $0.13/1K) is the canonical illustration of how badly this can mislead.
- **Running the batch sweep at one input/output length and generalising the knee.** CPM curves are shape-dependent: KV footprint per sequence scales with context length, so the concurrency where you run out of KV (and the concurrency where TTFT crosses your SLO) both shift with prompt/output length. A knee measured at 512-in/128-out does not transfer to a RAG workload at 4,000-in/200-out — re-measure per workload shape, don't extrapolate.
- **Treating throughput-optimal and SLO-optimal as the same point.** They are almost never the same point, and the gap between them is exactly where the cost story gets interesting. Running at throughput-optimal without checking the latency curve is the single most common mistake in a first CPM analysis.
- **Treating a CPM curve as a permanent fact.** Hardware generations, kernel improvements, and vLLM releases shift the curve; a number measured today can be materially stale within a year at the pace this space moves. Date every curve you publish and plan to re-measure, especially before a capacity-planning or pricing decision that depends on it.

## Self-check

**(a) Why does cost per million tokens fall 50–100× from batch 1 to batch 256?**

**Answer:** The GPU-hour cost is fixed, so CPM is the inverse of tokens extracted per hour. At batch 1 decode is memory-bandwidth-bound (module 03): the full weight set is read from HBM to emit a single token, wasting almost all compute. Continuous batching amortises each weight read across up to ~256 concurrent sequences, so output tokens/sec climbs ~50–100× before the KV-cache and compute roofs cap it. Throughput up 50–100× → CPM (its inverse) down 50–100×.

**(b) At what batch/concurrency does YOUR TTFT p99 cross 500 ms, and why is that the operating point?**

**Answer:** Read it off your sweep — in the worked example it lands between concurrency 64 and 128 (p99 480 ms → 910 ms). It is the operating point because throughput-optimal (the flat top of the tok/s curve) sits well to the right of it, so running there would violate the latency SLO for a marginal CPM gain. The knee — the largest batch with TTFT p99 still under 500 ms — captures nearly all the cost improvement while keeping the product shippable. Cost is minimised *subject to* the SLO constraint, not unconstrained.

**(c) Why is CPM meaningless without stating utilisation?**

**Answer:** The denominator (`peak_tok/s × 3600 × utilization`) assumes the GPU sustains that throughput for the whole hour. A benchmark quotes utilization=1.0 (100% busy); a real fleet runs at 30–50%. At 35% utilisation the same GPU and model cost ~2.85× the benchmark CPM. Quoting CPM without utilisation compares a saturated lab number against a half-idle production reality — off by 2–3×. The gap between benchmark CPM and fleet CPM is precisely the autoscaling ROI (lesson 08).

**(d) Why doesn't a CPM curve measured at 512-in/128-out transfer to a workload with 4,000-token prompts?**

**Answer:** KV-cache footprint per sequence scales with total context length, so a long-prompt workload hits the KV-capacity wall at a lower concurrency than a short-prompt one — the max feasible batch shrinks. Long prompts also mean bigger prefill passes, which spike TTFT for everyone sharing a batch (head-of-line blocking), so the TTFT-SLO knee arrives earlier too. Both walls that shape the curve — KV capacity and the TTFT knee — move with input/output length, so the curve (and its knee) has to be re-measured per workload shape, not assumed from a differently-shaped benchmark.

## Connections & what's next

This lesson is the module's cost pivot: lesson 03 (PagedAttention) made large batches physically fit, lesson 04 (production tuning) set the safe HBM envelope, and this lesson turned that envelope into a dollar curve and named the operating point inside it. The next lesson, [06 — Alternative servers and disaggregation](06-alternative-servers-disaggregation.md), asks what happens when a single fused engine instance is no longer the right unit of batching at all — splitting prefill and decode onto separate pools is, among other things, a way to give each phase its own batch-size knee instead of forcing one compromise batch on both. [Lesson 07 — Quantization ops](07-quantization-ops.md) stacks a second, largely independent lever (FP8) on top of the same CPM equation — expect to re-run this lesson's sweep at FP8 and compare curves directly. [Lesson 08 — Autoscaling](08-autoscaling-inference.md) is the answer to the utilisation gap this lesson keeps surfacing: it's the mechanism that drags a fleet's real utilisation toward the benchmark number this lesson computes at util=1.0.

## References & further reading

- **Primary sources**
  - Introl — "Inference Unit Economics: True Cost Per Million Tokens" — the deep dive on building the CPM number honestly (utilisation, all-in rate, token accounting), and the module's own cited methodology. <https://introl.com/blog/inference-unit-economics-true-cost-per-million-tokens-guide>
  - vLLM — `vllm bench serve` CLI docs — canonical flag reference for the sweep; confirm `--max-concurrency`, `--request-rate`, `--random-*-len`, `--percentile-metrics`, `--save-result`. <https://docs.vllm.ai/en/latest/cli/bench/serve/>

- **Real-world engineering blogs**
  - Character.AI — "Optimizing AI Inference at Character.AI" — the 33× multi-year cost-reduction case study; contrast a single sweep's snapshot with what compounds over years of stacked levers. <https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/>
  - Together AI — "Speculative decoding for high-throughput long-context inference" — the large-batch/long-context counter-example to "spec decoding only helps small batches," referenced in Perspectives. <https://www.together.ai/blog/speculative-decoding-for-high-throughput-long-context-inference>

- **Deeper dives**
  - Anyscale — "Achieve 23x LLM Inference Throughput & Reduce p50 Latency" — the mechanistic explanation of *why* the CPM curve has the shape it does: continuous batching vs. static batching, benchmarked. <https://www.anyscale.com/blog/continuous-batching-llm-inference>
  - NVIDIA — "An Introduction to Speculative Decoding for Reducing Latency in AI Inference" — a forward pointer to the next lever after batching, out of this module's scope but worth knowing exists. <https://developer.nvidia.com/blog/an-introduction-to-speculative-decoding-for-reducing-latency-in-ai-inference/>
