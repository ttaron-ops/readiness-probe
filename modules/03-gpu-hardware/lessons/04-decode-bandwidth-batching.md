---
lesson: "03.4"
title: "Decode throughput: bandwidth ceilings, batching, and the prefill/decode split"
module: "03"
concept: "Decode bandwidth & batching"
status: not-started
est_time: "7h"
prev: "03-memory-hierarchy-hbm.md"
next: "05-precision-and-tensor-cores.md"
artifacts: []
sources: 13
---

# 03.4 · Decode throughput: bandwidth ceilings, batching, and the prefill/decode split

> **Concept.** Decode tok/s ≈ HBM bandwidth ÷ model bytes; batching is the only lever that raises it, KV cache is what stops you batching forever, and production systems now split prefill and decode onto separate GPU pools to buy for each bottleneck independently.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 03.3 built the two-number budget — HBM capacity (what fits) and HBM bandwidth (how fast) — and ended with a bandwidth sanity check: a 70B FP16 model on an H100 has a theoretical decode ceiling of ~24 tokens/sec, set entirely by how fast the GPU can stream weights out of HBM. This lesson takes that single-stream number and asks the two questions a platform engineer actually gets paid to answer: how do you turn ~24 tok/s per stream into hundreds or thousands of tokens/sec of *aggregate* throughput off one GPU, and what stops that scaling from being free? The answer — batching, capped by the KV-cache footprint from lesson 03.3 — is the mechanism behind essentially every modern LLM serving architecture, including the prefill/decode disaggregation systems now running at production scale.

## Why this matters

This is the lesson that turns "GPU literacy" into "I can predict the invoice." If you can derive a model's decode throughput ceiling from one spec-sheet number and one model size, you can estimate cost-per-token *before* renting anything, size a fleet, and immediately smell whether a serving setup is leaving money on the table. This is squarely the CoreWeave/NVIDIA interview territory this module's README names directly: *"why is decode memory-bandwidth bound, prefill compute-bound?"* and *"tokens/sec ceiling for a 70B on H100"* are stock platform-engineering interview probes, and the honest answer to both requires exactly the arithmetic this lesson teaches.

The counter-intuitive part — and the part that trips up engineers who assume "more FLOPS means faster" — is that a single-user token stream uses a rounding-error fraction of a datacenter GPU's compute. An H100 can do ~1,979 TFLOPS of FP8 tensor math. Generating one token from a 70B model needs on the order of 140 GFLOP of actual math per forward pass. If compute were the limit, you'd get *thousands* of tokens/sec. You get ~24. The GPU spends virtually all its time waiting on memory — reading 140 GB of weights out of HBM to produce a single token. Decode is a memory-bandwidth problem wearing a compute costume, and production LLM-serving companies (Moonshot AI/Kimi, Microsoft, Databricks — see below) now build entire serving architectures specifically around that fact.

## What's new here (calibration)

Per the module README, you are not implementing a serving engine — you're learning to reason about its throughput and cost from the outside. This lesson does **not** re-teach FlashAttention kernel internals, how continuous batching is implemented inside a scheduler, or warp-level softmax tricks. What it adds:

- **A derivation you can run from memory** — the bandwidth-only decode ceiling, and why FLOPS never enters the calculation.
- **The batching mechanism named precisely** — why it's close to free throughput in the memory-bound regime, and the exact formula for the wall it eventually hits.
- **The prefill/decode split as the reason modern serving architecture looks the way it does** — continuous batching, chunked prefill, and full physical disaggregation are all responses to the same underlying fact: prefill and decode have opposite bottlenecks on the same chip.
- **Production evidence, not just theory** — real disaggregation systems (Mooncake/Kimi, Splitwise/Microsoft) and a documented failure-mode retrospective, so you know when disaggregation is worth the operational complexity and when it isn't.

## Core concepts

### Roofline, in one paragraph

Every operation is either compute-bound (limited by FLOPS) or memory-bound (limited by bandwidth), decided by **arithmetic intensity** (AI) = FLOPs ÷ bytes moved. The hardware crossover is the **ridge point**: on an H100, dense BF16 peak (989 TFLOP/s) ÷ 3.35 TB/s ≈ **~295 FLOP/byte**; in FP8 (1,979 TFLOP/s, same bandwidth) the ridge roughly doubles to **~590 FLOP/byte** — halving the bytes moved per useful FLOP raises the bar for staying compute-bound (lesson 03.2 derives this in full). Hold that number. Decode lands *far* below it.

### Deriving the single-stream decode ceiling

Autoregressive decode generates one token at a time. For each new token, the model runs a forward pass, and that pass must read essentially every weight from HBM exactly once. (The math per weight is tiny — one multiply-accumulate per weight per token — so arithmetic intensity ≈ 2 FLOP / 2 bytes ≈ 1 FLOP/byte: deeply memory-bound, hundreds of times below the ridge point.)

The time to produce one token is therefore bounded below by the time to stream the weights:

```
t_token ≥ model_bytes ÷ HBM_bandwidth
tokens/sec ≤ HBM_bandwidth ÷ model_bytes
```

That's the whole ceiling. FLOPS do not appear in the formula.

- **70B FP16 on H100:** 3.35 TB/s ÷ 140 GB ≈ **23.9 tok/s**.
- **70B FP16 on H200:** 4.80 TB/s ÷ 140 GB ≈ **34.3 tok/s** (same die, +43% — lesson 03.3's H200 case, restated as throughput).
- **70B at FP8/INT8 (~70 GB) on H100:** 3.35 TB/s ÷ 70 GB ≈ **~48 tok/s** — quantization halves the bytes read per token, which *doubles* the decode ceiling. Precision is a throughput lever, not just a capacity lever (lesson 03.5 goes deep on this).

Real systems hit ~60–80% of this ideal (KV-cache reads on top of weight reads, attention overhead, imperfect bandwidth utilization, kernel-launch overhead), so expect measured single-stream throughput of ~15–20 tok/s for a 70B model on an H100. The ceiling is the ceiling — no software optimization ever pushes single-stream decode past it, because it's a physical bandwidth limit, not an inefficiency.

### Batching: the only decode lever

Here's the key insight. The weight read is a **fixed cost paid once per forward pass**, and a forward pass can process many sequences at once. If you run a batch of B sequences, you still read the weights once, but you produce B tokens. The per-token weight-read cost is amortized across the batch:

```
aggregate tok/s  ≈  B × (single-stream ceiling)   — until you hit a wall
```

Batching is close to free throughput in the memory-bound regime: the compute was idle anyway (recall decode's AI ≈ 1, hundreds of times below the ridge point), so batching fills otherwise-wasted tensor-core cycles without meaningfully increasing the time per forward pass. A batch of 32 on a 70B/H100 can push aggregate throughput toward 32 × ~20 ≈ hundreds of tokens/sec, at nearly the same per-token *latency* per stream. **This is why every serious inference server batches** (continuous/inflight batching — vLLM's core mechanism), and why single-stream serving is the most expensive way to run a GPU: you're paying for the whole card to feed one user at bandwidth's mercy.

### Why KV cache caps the batch — and thus caps throughput

Batching would be an unbounded free lunch except for one thing: **each sequence in the batch needs its own KV cache** in HBM, sized per lesson 03.3's formula. As you raise B, KV footprint grows linearly:

```
KV bytes ≈ 2 × layers × (num_kv_heads × head_dim) × bytes/elem × seq_len × B
```

You run out of HBM. The batch-size ceiling is:

```
B_max ≈ (HBM_capacity − weights − overhead) ÷ KV_bytes_per_sequence
```

The throughput-vs-batch curve **rises steeply, then flattens** when KV cache exhausts available memory. This is the single most important operational curve in LLM serving, and it's exactly why HBM *capacity* (lesson 03.3) and *bandwidth* (this lesson) are joint constraints on the same problem: bandwidth sets the per-batch throughput ceiling, capacity sets how big the batch can grow to harvest it. **The two levers multiply.** More capacity → bigger batch → more of the bandwidth ceiling actually realized. This is the second half of the H200 inference case from lesson 03.3: more KV headroom directly buys larger batches, which buys higher realized GPU utilization, on top of the raw bandwidth advantage.

Levers to raise B_max: quantize weights (frees capacity for KV — lesson 03.5), quantize/compress the KV cache itself (FP8 or int8 KV, independent of weight precision — this is exactly what Character.AI's stack does per lesson 03.3), use GQA/MQA models (smaller KV per sequence), page the KV cache to eliminate fragmentation (PagedAttention, lesson 03.3), or move to a bigger-memory SKU.

### Prefill vs. decode: two regimes, one request

A request has two phases with opposite hardware profiles:

| Phase | What it does | Tokens/pass | Arithmetic intensity | Bound by |
|---|---|---|---|---|
| **Prefill** | process the whole prompt, build KV cache | many (all prompt tokens at once) | **high** (big matmuls, reuse weights across all prompt tokens) | **compute** (FLOPS) |
| **Decode** | generate output tokens one at a time | 1 per pass | **≈ 1** | **memory bandwidth** |

Prefill is a big dense matmul: one weight read feeds computation over hundreds or thousands of prompt tokens simultaneously, so intensity is high and it saturates the tensor cores — compute-bound, squarely above the ridge point. Decode reads the same weights to produce a single token — intensity ≈ 1, memory-bound, hundreds of times below the ridge point. **The same GPU is compute-bound for a few milliseconds per request, then memory-bound for the rest of it.**

### Why this drives serving architecture

Because the two phases stress different resources, running them on the same GPU is wasteful — a long prefill blocks the decode queue (head-of-line latency for every other in-flight user), and a GPU busy decoding leaves its compute idle. Modern serving systems attack this at increasing levels of aggressiveness:

- **Continuous / inflight batching.** Don't wait for a batch to finish; inject new sequences into the running batch each step to keep the batch — and thus the amortized weight read — full. Standard in vLLM.
- **Chunked prefill.** Slice long prefills into chunks interleaved with decode steps so a long prompt doesn't starve decode latency for everyone else sharing the GPU.
- **Prefill/decode disaggregation.** Run prefill on one pool of GPUs (compute-optimized, keep tensor cores hot) and decode on another (bandwidth/capacity-optimized, run large batches), streaming the KV cache between pools over the network. This lets you buy for each regime independently — cheaper high-FLOPS cards for prefill, high-bandwidth/high-capacity cards (H200) for decode. This is where the capacity/bandwidth buying levers from lesson 03.3 pay off at the architecture level, not just the single-GPU level.

Disaggregation is not a hypothetical — it's the subject of two influential 2024 systems papers, **DistServe** (Zhong et al., OSDI 2024) and **Splitwise** (Patel et al., Microsoft, ISCA 2024), and it's running in production today. Moonshot AI's **Mooncake** system, described in a FAST 2025 paper, is a KV-cache-centric disaggregated architecture serving Kimi (a live consumer LLM product) across **thousands of nodes, processing more than 100 billion tokens per day**. In trace-based tests against non-disaggregated baselines under real SLOs, Mooncake reports **59%–498% higher effective request capacity**, with gains up to 525% in long-context scenarios — the wide range reflects how much the benefit depends on context length and SLO tightness, with long-context, SLO-constrained traffic benefiting most. NVIDIA productizes the same idea in **Dynamo**, its open-source disaggregated-serving framework, reporting (in its own benchmarks) up to 7× higher throughput serving DeepSeek-R1 on Blackwell versus a non-disaggregated baseline, and up to 30× more requests served on a GB200 NVL72 — treat these specific multiples as **vendor-reported**, not independently verified against the same baseline as the academic numbers above.

### Disaggregation is not a free win — the honest failure-mode view

Disaggregation adds real costs: KV-cache has to be transferred over the network from the prefill pool to the decode pool for every request, which adds network latency and a new failure surface; the system is now two fleets to capacity-plan and operate instead of one; and the win only shows up past a certain scale or SLO tightness — for small deployments the added complexity can outweigh the throughput gain. This is precisely the finding of Hao AI Lab's retrospective, ["Disaggregated Inference: 18 Months Later"](https://haoailab.com/blogs/distserve-retro/) — written by the same lab that co-authored DistServe, reviewing how disaggregation actually played out operationally after the initial research result. Treat any single "disaggregation gives Nx" number (including Mooncake's own 59–498% range) as workload- and SLO-dependent, not a universal multiplier you get by flipping a switch.

### When does decode itself become compute-bound?

Batching raises decode's *effective* arithmetic intensity: one weight read now does B tokens' worth of math, so effective intensity grows roughly with batch size. Push B high enough and you cross the ~590 FLOP/byte FP8 ridge point (or ~295 FLOP/byte at BF16) — decode flips to **compute-bound**, and further batching stops adding throughput because the tensor cores are now saturated. In practice you almost always hit the **KV-cache capacity** wall first (HBM fills up long before compute saturates), so decode stays memory-bound and the binding constraint remains HBM — capacity and bandwidth together — not FLOPS. The crossover is real and worth naming precisely: if you ever see decode throughput plateau while HBM still has free capacity, you've crossed into the compute-bound regime, and the fix flips from "raise batch/capacity" to "raise FLOPS/precision."

## Perspectives

**Theory.** The roofline crossover argument is the whole mechanism: batching raises decode's effective arithmetic intensity toward the ridge point, and the KV-cache capacity formula from lesson 03.3 tells you exactly how far you can push B before HBM — not FLOPS — becomes the limiter.

**Practice.** Databricks/MosaicML's engineering blog, ["LLM Inference Performance Engineering: Best Practices"](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices), is written by a team that operated multiple production inference backends (vLLM, TensorRT-LLM, FasterTransformer) and explicitly frames prefill (parallel, compute-bound) versus decode (autoregressive, memory-bound) as the operating model for real serving-infrastructure decisions — the practitioner's confirmation that this lesson's two-phase model is how production teams actually reason, not a textbook simplification.

**Failure mode / hard-won lesson.** Hao AI Lab's 18-months-later retrospective is the necessary counterweight to a pure "disaggregation always wins" reading of the DistServe/Splitwise papers: real deployments pay a network-transfer cost for KV-cache migration and take on real operational complexity running two fleets instead of one, and the payoff is scale- and SLO-dependent. Pressure-test any disaggregation pitch against this before recommending the architecture change.

**Economics.** Mooncake's verified 59–498% capacity gain under real SLOs, at Kimi's production scale (>100B tokens/day across thousands of nodes), is the dollar argument in one number — and it's an independent sanity check on this lesson's own worked-example batching gain (below), confirming the order of magnitude is realistic, not a toy exaggeration.

## Real-world use cases

- **Moonshot AI / Kimi, ["Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving"](https://arxiv.org/abs/2407.00079) (FAST 2025).** Production KV-cache-centric disaggregation serving Kimi at >100 billion tokens/day across thousands of nodes; 59–498% higher effective request capacity under real SLOs versus non-disaggregated baselines. The best-in-class real-world evidence for this lesson's entire thesis — this is not a research toy, it's a live consumer product's serving stack.
- **Microsoft Research, ["Splitwise improves GPU usage by splitting LLM inference phases"](https://www.microsoft.com/en-us/research/blog/splitwise-improves-gpu-usage-by-splitting-llm-inference-phases/).** The engineering-blog companion to the ISCA 2024 Splitwise paper — shows the phase-splitting idea explained for a production-infrastructure audience, not just a research one.
- **Hao AI Lab, ["Disaggregated Inference: 18 Months Later"](https://haoailab.com/blogs/distserve-retro/).** The honest retrospective/failure-mode angle from the lab that originated DistServe — what disaggregation promised versus what actually held up operationally, including the network and complexity costs this lesson's "not a free win" section draws on.
- **Databricks/MosaicML, ["LLM Inference Performance Engineering: Best Practices"](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices).** Practitioner account from a team running multiple real serving backends, confirming the prefill-compute / decode-memory two-phase model as an operational reality, not just theory.

## Worked example

**Predict, then reason about, decode throughput for Llama-3-70B FP16/FP8 on an H100.**

**1. Single-stream ceiling.**
```
model_bytes = 70e9 × 2 = 140 GB
ceiling = 3.35 TB/s ÷ 140 GB = 3.35e12 / 1.40e11 ≈ 23.9 tok/s
```
Expect measured ~16–20 tok/s (70–80% of ideal). Note it doesn't fit on one H100 (lesson 03.3) — assume 2× H100 tensor-parallel; TP splits both the weight bytes *and* the effective bandwidth across 2 cards, so the per-token weight-read time is roughly preserved and the ~24 tok/s ceiling still describes the aggregate. Good enough for planning purposes.

**2. Batch ceiling from KV cache.** Take the cleaner case: **70B at FP8 (~70 GB) on one H100 80GB.**
```
free for KV ≈ 80 − 70 − 4 ≈ 6 GB
KV per seq @ 4k ctx, GQA (8 KV heads × 128 head_dim):
  bytes/token = 2 × 80 × (8×128) × 1 byte (FP8 KV) ≈ 163 KB/token
  4096 tokens × 163 KB ≈ 0.67 GB/seq
B_max ≈ 6 GB ÷ 0.67 GB ≈ ~9 concurrent sequences
```

**3. Aggregate throughput.** FP8 single-stream ceiling ≈ 3.35 TB/s ÷ 70 GB ≈ 47.9 tok/s; measured ~35. With B ≈ 9: aggregate ≈ 9 × 35 ≈ **~315 tok/s** off one H100 — versus ~35 tok/s single-stream. **A ~9× throughput gain (≈900%) for the same rented GPU-hour = ~9× lower cost-per-token.** That gap is the entire economic argument for batched serving.

**4. Sanity-check against production evidence.** Mooncake's independently measured production numbers — 59–498% higher effective capacity from disaggregation *alone*, before even counting the batching gain within each pool — confirm this lesson's ~9× (900%) batching gain sits in the right order of magnitude for real systems. The two numbers aren't measuring the same thing (Mooncake's is disaggregation's incremental gain on top of an already-batched baseline; this worked example is batching's gain over unbatched single-stream), but both point at the same underlying fact: the gap between naive single-stream serving and a properly batched/architected system is not a small optimization, it's close to an order of magnitude.

**5. Cost framing.** If the H100 rents at ~$3/hr *(2026 snapshot — always re-verify current rate; see lesson 03.7's capstone for live-pricing sourcing)* and you sustain 315 tok/s:
```
315 tok/s × 3600 s = 1.13M tokens/hr  →  $3 / 1.13M ≈ $2.65 per 1M output tokens
```
Single-stream (35 tok/s) would be ~$24 per 1M tokens. Same hardware, ~9× the bill. This is the number the module's deliverable is built to expose.

## Practice

Rent a GPU (H100 80GB ideal; adjust spec numbers to your SKU) and serve with **vLLM**. Budget ~1.5 GPU-hours. Results feed the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md).

1. **Serve a model** you can fit (e.g., an 8B–13B model at FP16, or a quantized larger model). `vllm serve <model>` exposes an OpenAI-compatible endpoint.
2. **Measure single-stream decode tok/s:** send one request, `max_tokens` ~256, temperature 0, and record output tokens ÷ decode time (exclude prefill/time-to-first-token).
3. **Measure batched decode tok/s vs. batch size:** drive concurrent requests at concurrency 1, 2, 4, 8, 16, 32, 64… (use vLLM's `benchmarks/benchmark_serving.py`, or a simple async client). Record aggregate output tok/s at each level.
4. **Find the knee:** keep raising concurrency until aggregate throughput stops rising (KV-cache/memory cap) or vLLM starts queueing/preempting. Note the batch size where it flattens and check `nvidia-smi` / vLLM logs for the KV-cache-full signal.
5. **Compare to theory:** compute your bandwidth-derived single-stream ceiling (`spec_bandwidth ÷ model_bytes`) and compare it to your *measured* single-stream number. Report the ratio (measured/ideal) and explain the gap.
6. **(Stretch)** If you have access to two GPUs, sketch — on paper, no need to implement — how you would split a prefill-heavy and a decode-heavy workload across them, and estimate the KV-cache-transfer bytes per request that a real disaggregated setup would have to move over the network.

**Acceptance:** a **decode-throughput-vs-batch-size curve** (table or plot: concurrency on x, aggregate tok/s on y) for one model on one SKU, annotated with (a) the measured single-stream number, (b) the bandwidth-derived ceiling and the measured/ideal ratio, and (c) the batch size where throughput flattens and why (KV-cache cap). Add it to the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md) alongside a cost-per-1M-tokens figure at the best sustained throughput.

## Common pitfalls

1. **Assuming disaggregation is a free win.** The Hao AI Lab retrospective is the source to cite for "it adds KV-transfer network cost and operational complexity; it only pays off past a certain scale or SLO tightness" — don't recommend physical prefill/decode splitting for a modest deployment without checking whether the added complexity is actually justified.
2. **Believing batching benefits are unbounded.** KV-cache capacity, not weights, is almost always the real wall — the throughput-vs-batch curve rises steeply then flattens hard, it does not keep climbing.
3. **Applying prefill-side optimizations (bigger batches) to decode and expecting the same linear gain, or vice versa.** The two phases have opposite bottlenecks — compute for prefill, bandwidth for decode — so a fix aimed at one regime can do nothing (or even hurt) in the other.
4. **Treating vendor-reported disaggregation multiples as directly comparable to independently-measured research numbers.** NVIDIA's Dynamo claims of 7× and 30× are the vendor's own benchmarks against a baseline they chose; DistServe's and Mooncake's numbers come from academic/production measurement against stated baselines. Always check what baseline a multiple was measured against before comparing two "Nx" claims.

## Self-check

- Estimate the single-stream tok/s ceiling for a 70B model at FP16 on an H100. Show the arithmetic. **Answer:** Decode reads all weights per token, so tok/s ≤ HBM_bandwidth ÷ model_bytes. Weights = 70e9 × 2 bytes = 140 GB. H100 bandwidth = 3.35 TB/s. `3.35e12 ÷ 1.40e11 ≈ 23.9 tok/s` — call it ~24 tok/s. FLOPS never enter the calculation because decode is memory-bound. Measured will be lower (~16–20 tok/s, 70–80% of ideal) due to KV reads, attention overhead, and imperfect bandwidth utilization.
- Why does batching help decode but not a single long prefill? **Answer:** Decode is memory-bound: the per-token cost is dominated by reading the weights, and that read is a fixed cost per forward pass that can be shared across many sequences. Batching B sequences reads the weights once and emits B tokens, amortizing the read into a near-linear throughput gain, because the compute was idle anyway. A single prefill is already compute-bound: one weight read already feeds math over all the prompt's tokens at once, so arithmetic intensity is high and the tensor cores are already saturated. There's no idle compute to fill, so batching more work onto an already-compute-bound prefill doesn't raise throughput — you're limited by FLOPS, not by an amortizable memory read.
- At what point does decode itself become compute-bound? **Answer:** When batching raises decode's effective arithmetic intensity above the hardware ridge point (~590 FLOP/byte FP8, ~295 FLOP/byte BF16, on H100). Each sequence added to the batch reuses the same weight read for more math, so effective intensity grows roughly with batch size; push the batch large enough and you cross from memory-bound to compute-bound, at which point the tensor cores saturate and further batching stops adding throughput. In practice you usually hit the KV-cache capacity wall first (HBM fills up long before compute saturates), so decode stays memory-bound and HBM — capacity and bandwidth together — remains the binding constraint.
- Mooncake (Kimi's production serving stack) reports 59–498% higher effective capacity from KV-cache-centric disaggregation. What's the mechanism, and why is the range so wide? **Answer:** The mechanism is disaggregating prefill and decode onto separate GPU pools while using otherwise-idle CPU/DRAM/SSD/network resources for a distributed KV cache, so neither phase blocks or wastes the other's resources. The range is wide because the gain depends heavily on context length and SLO tightness — long-context, SLO-constrained traffic (where KV-cache transfer and reuse matter most) benefits far more than short, loosely-SLO'd traffic, which is also why the Hao AI Lab retrospective stresses that disaggregation's payoff is workload-dependent, not a fixed multiplier.

## Connections & what's next

This lesson is the direct continuation of lesson 03.3: the bandwidth ceiling derived there becomes the single-stream throughput number here, and the KV-cache formula derived there becomes the batch-size cap here — capacity and bandwidth are literally the two inputs to every calculation in this lesson. It previews lesson 03.5 (precision), which is the next multiplier on both the decode ceiling (FP8 roughly doubles it) and the batch-size cap (quantized KV frees capacity for more concurrent sequences) — the same two-lever structure, one level deeper. It also sets up module 07 (inference serving), where prefill/decode disaggregation, continuous batching, and the serving-architecture decisions sketched here become full production system design, and module 11 (GPU cost economics), where the cost-per-token arithmetic in this lesson's worked example becomes a fleet-wide capacity-planning model.

Next: **[03.5 · Precision & tensor cores](05-precision-and-tensor-cores.md)** takes the FP8-halves-the-bytes, FP8-roughly-doubles-throughput observation used twice in this lesson and derives it properly — including exactly why the *realized* throughput gain (Databricks reports ~1.5×) is smaller than the *raw* tensor-core multiple (~2×), the same "theory vs. amortized reality" gap this lesson taught you to expect.

## References & further reading

**Primary sources**
- Horace He, ["Making Deep Learning Go Brrrr From First Principles"](https://horace.io/brrr_intro.html) — the module's conceptual anchor; the compute/memory/overhead-bound trichotomy that arithmetic intensity and the ridge point (used throughout this lesson) build on.
- Williams, Waterman, Patterson, ["Roofline: An Insightful Visual Performance Model for Multicore Architectures"](https://dl.acm.org/doi/10.1145/1498765.1498785) (CACM 52(4), 2009; free mirror at [escholarship.org](https://escholarship.org/content/qt78h8v7mr/qt78h8v7mr.pdf)) — the formal ridge-point / arithmetic-intensity definitions this lesson's crossover analysis relies on.
- NVIDIA, [H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — source of truth for the 989/1,979 dense TFLOPS and 3.35 TB/s bandwidth figures used in every calculation above.
- Zhong et al., ["DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving"](https://arxiv.org/abs/2401.09670) (OSDI 2024) — the research paper establishing prefill/decode disaggregation as a serving architecture.
- Patel et al., ["Splitwise: Efficient Generative LLM Inference Using Phase Splitting"](https://arxiv.org/abs/2311.18677) (Microsoft, ISCA 2024) — an independent disaggregation architecture and evaluation, pairs with the Microsoft Research blog below.
- Moonshot AI, ["Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving"](https://arxiv.org/abs/2407.00079) (FAST 2025) — the production-scale disaggregation paper behind this lesson's headline real-world numbers.

**Real-world engineering blogs**
- Microsoft Research, ["Splitwise improves GPU usage by splitting LLM inference phases"](https://www.microsoft.com/en-us/research/blog/splitwise-improves-gpu-usage-by-splitting-llm-inference-phases/) — what it shows: the Splitwise research made concrete for a production-infrastructure audience.
- Databricks/MosaicML, ["LLM Inference Performance Engineering: Best Practices"](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices) — what it shows: the prefill-compute/decode-memory model confirmed from real multi-backend serving experience.
- Hao AI Lab, ["Disaggregated Inference: 18 Months Later"](https://haoailab.com/blogs/distserve-retro/) — what it shows: the honest retrospective on disaggregation's real operational costs and when it's not worth it. *(Title and URL corroborated via search from the same lab that co-authored DistServe; not independently fetched in this pass — high confidence, but re-verify the live page at build time.)*
- ["Prefill is compute-bound, decode is memory-bound — why your GPU shouldn't do both"](https://towardsdatascience.com/prefill-is-compute-bound-decode-is-memory-bound-why-your-gpu-shouldnt-do-both/) — Towards Data Science — a full walkthrough of the two-regime model that motivates disaggregation; read in full alongside Horace He.

**Deeper dives**
- NVIDIA Dynamo [documentation](https://docs.nvidia.com/dynamo/) — the vendor's productized disaggregated-serving framework; treat its 7×/30× throughput claims as vendor-reported benchmarks, not independently verified against the same baselines as the academic papers above.
- Google DeepMind, ["How to Scale Your Model" — "All About Rooflines" chapter](https://jax-ml.github.io/scaling-book/roofline/) (jax-ml/scaling-book, 2025) — a modern, free, frontier-lab-authored treatment of rooflines including a GPU-specific bonus chapter; a strong complement to the 2009 CACM paper. *(The parent GitHub repo `jax-ml/scaling-book` is directly confirmed; this specific chapter page was corroborated via search, not independently fetched, in this research pass.)*
- [vLLM documentation](https://docs.vllm.ai/) — continuous/inflight batching and PagedAttention mechanisms in a real serving stack; use `benchmark_serving.py` for the practice section above.
