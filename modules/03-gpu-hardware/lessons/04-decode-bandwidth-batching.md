---
lesson: "03.4"
title: "Decode throughput: bandwidth ceilings, batching, and prefill/decode split"
module: "03"
concept: "Decode throughput: bandwidth ceilings, batching, and prefill/decode split"
status: not-started
est_time: "5h"
artifacts: []
---
# 03.4 · Decode throughput: bandwidth ceilings, batching, and prefill/decode split
> **Concept.** Decode tok/s ≈ HBM bandwidth ÷ model bytes; batching is the only lever, and KV cache caps it.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Why this matters

This is the lesson that turns "GPU literacy" into "I can predict the invoice."
If you can derive a model's decode throughput ceiling from one spec-sheet number
and one model size, you can estimate cost-per-token *before* renting anything,
size a fleet, and immediately smell whether a serving setup is leaving money on
the table.

The counter-intuitive part — and the part that trips up engineers who assume
"more FLOPS = faster" — is that a single-user token stream uses a rounding-error
fraction of a datacentre GPU's compute. An H100 can do ~1,979 TFLOPS of FP16
tensor math. Generating one token from a 70B model needs ~140 GFLOP of actual
math. If compute were the limit you'd get *thousands* of tokens/sec. You get
~24. The GPU spends virtually all its time **waiting on memory** — reading 140
GB of weights out of HBM to produce a single token. Decode is a memory-bandwidth
problem wearing a compute costume.

Understanding this tells you the two things that actually matter for inference
cost: (1) the hard per-stream ceiling set by bandwidth, and (2) that **batching**
is the *only* way to raise aggregate throughput — and that KV cache is what stops
you from batching infinitely.

## The platform-engineer's lens

**Extract for cost. Skip the kernel depth.**

- **What to extract:** the bandwidth-derived tok/s ceiling (you'll use it
  constantly), *why* batching amortises the weight read, why KV-cache footprint
  caps batch size (ties straight back to 03.3), and the prefill-vs-decode split
  that underpins modern serving architectures (disaggregation, continuous
  batching, chunked prefill). These are the levers you'll actually pull:
  batch size, max sequences, quantisation, SKU choice, and prefill/decode
  placement.
- **What to skip:** the FlashAttention kernel internals, how continuous batching
  is implemented in the scheduler, warp-level softmax tricks. You need to know
  these mechanisms *exist and what they cost*, not how to write them.

The mental model to walk away with: **arithmetic intensity** — FLOPs performed
per byte moved. Prefill has high intensity (compute-bound); decode has intensity
≈ 1 (memory-bound). Every serving optimisation is, at bottom, an attempt to push
decode's effective intensity up by batching.

## Core notes

### Roofline in one paragraph

Every operation is either **compute-bound** (limited by FLOPS) or **memory-bound**
(limited by bandwidth). Which one depends on **arithmetic intensity** =
FLOPs ÷ bytes moved. There's a hardware crossover, the *ridge point*: on an H100,
~1,979 TFLOPS ÷ 3.35 TB/s ≈ **~590 FLOP/byte**. Above that intensity you're
compute-bound; below it, memory-bound. Hold that number — decode lands *far*
below it.

### Deriving the single-stream decode ceiling

Autoregressive decode generates one token at a time. For each new token, the
model does a forward pass, and that pass must **read every weight from HBM
exactly once**. (The math per weight is tiny — one multiply-accumulate per
weight per token — so intensity ≈ 2 FLOP / 2 bytes ≈ 1 FLOP/byte. Deeply
memory-bound.)

So the time to produce one token is bounded below by the time to stream the
weights:

```
t_token ≥ model_bytes ÷ HBM_bandwidth
tokens/sec ≤ HBM_bandwidth ÷ model_bytes
```

That's it. That's the ceiling. FLOPS do not appear.

- **70B FP16 on H100:** 3.35 TB/s ÷ 140 GB ≈ **23.9 tok/s**.
- **70B FP16 on H200:** 4.80 TB/s ÷ 140 GB ≈ **34.3 tok/s** (same die, +43%).
- **70B at FP8/INT8 (~70 GB) on H100:** 3.35 TB/s ÷ 70 GB ≈ **~48 tok/s** —
  quantisation halves the bytes-per-token read, so it *doubles* the decode
  ceiling. Precision is a throughput lever, not just a capacity lever.

Real systems hit ~60–80% of this ideal (KV-cache reads, attention, imperfect
bandwidth utilisation, kernel launch overhead), so expect measured single-stream
of ~15–20 tok/s for a 70B on an H100. The ceiling is the ceiling — you'll never
beat it single-stream.

### Batching: the only decode lever

Here's the key insight. The weight read is a **fixed cost paid once per forward
pass**, and a forward pass can process *many sequences at once*. If you run a
batch of B sequences, you still read the weights once, but you produce B tokens.
The per-token weight-read cost is amortised across the batch:

```
aggregate tok/s  ≈  B × (single-stream ceiling)   — until you hit a wall
```

So batching is close to *free throughput* in the memory-bound regime: you were
wasting the compute anyway; batching fills it. A batch of 32 on that 70B/H100
can push aggregate throughput toward 32 × ~20 ≈ hundreds of tok/s, at nearly the
same per-token *latency* per stream. **This is why every serious inference server
batches** (continuous/inflight batching, e.g. vLLM), and why single-stream
serving is the most expensive way to run a GPU — you're paying for the whole card
to feed one user at bandwidth's mercy.

### Why KV cache caps the batch (and thus caps throughput)

Batching would be unbounded free lunch except for one thing: **each sequence in
the batch needs its own KV cache** in HBM (lesson 03.3). As you raise B, KV
footprint grows linearly:

```
KV bytes ≈ 2 × layers × hidden × bytes/elem × seq_len × B
```

You run out of HBM. The batch size ceiling is:

```
B_max ≈ (HBM_capacity − weights − overhead) ÷ KV_bytes_per_sequence
```

So the throughput-vs-batch curve **rises steeply, then flattens/stops** when KV
cache exhausts memory. This is the single most important operational curve in LLM
serving, and it's exactly why HBM *capacity* (03.3) and *bandwidth* (this lesson)
are joint constraints: bandwidth sets the per-batch throughput, capacity sets how
big the batch can be. **The two levers multiply.** More capacity → bigger batch →
more of the bandwidth ceiling actually harvested. (This is the second half of the
H200 inference case: more KV headroom → larger batches → higher GPU utilisation.)

Levers to raise B_max: quantise weights (frees capacity), quantise/compress the
KV cache (FP8 KV), use GQA/MQA models (smaller KV), page the KV cache to cut
fragmentation (PagedAttention), or move to a bigger-memory SKU.

### Prefill vs decode: two regimes, one request

A request has two phases with opposite hardware profiles:

| Phase | What it does | Tokens/pass | Arithmetic intensity | Bound by |
|-------|--------------|-------------|----------------------|----------|
| **Prefill** | process the whole prompt, build KV cache | many (all prompt tokens at once) | **high** (big matmuls, reuses weights across all prompt tokens) | **compute** (FLOPS) |
| **Decode** | generate output tokens one at a time | 1 per pass | **≈ 1** | **memory bandwidth** |

Prefill is a big dense matmul: one weight read feeds computation over hundreds or
thousands of prompt tokens simultaneously, so intensity is high and it saturates
the tensor cores — compute-bound. Decode reads the same weights to produce a
single token — intensity ≈ 1, memory-bound. **The same GPU is compute-bound for
a few milliseconds, then memory-bound for the rest of the request.**

### Why this drives serving architecture

Because the two phases stress different resources, running them on the same GPU
is wasteful — a long prefill blocks the decode queue (head-of-line latency), and
a GPU busy decoding wastes its compute:

- **Continuous / inflight batching:** don't wait for a batch to finish; inject
  new sequences into the running batch each step to keep the batch (and thus the
  amortised weight read) full. Standard in vLLM.
- **Chunked prefill:** slice long prefills into chunks interleaved with decode so
  prefill doesn't starve decode latency.
- **Prefill/decode disaggregation:** run prefill on one pool of GPUs
  (compute-optimised, keep tensor cores hot) and decode on another
  (bandwidth/capacity-optimised, big batches), streaming the KV cache between
  them. Lets you scale and buy for each regime independently — e.g. cheaper
  high-FLOPS cards for prefill, high-bandwidth/high-capacity cards (H200) for
  decode. This is where the 03.3 capacity/bandwidth buying levers pay off.

### When does decode become compute-bound?

Batching raises decode's *effective* arithmetic intensity: one weight read now
does B tokens' worth of math, so intensity ≈ B FLOP/byte-ish. Push B high enough
and you cross the ~590 FLOP/byte ridge point — decode flips to **compute-bound**,
and further batching stops adding throughput (you've saturated the tensor cores).
In practice you almost always hit the **KV-cache capacity** wall *before* the
compute wall, so decode stays memory-bound and the binding constraint is HBM, not
FLOPS. But the crossover is real and worth naming: if you ever see decode
throughput plateau while HBM still has free capacity, you've become compute-bound.

## Worked example

**Predict, then reason about, decode throughput for Llama-3-70B FP16 on an H100.**

**1. Single-stream ceiling.**
```
model_bytes = 70e9 × 2 = 140 GB
ceiling = 3.35 TB/s ÷ 140 GB = 3.35e12 / 1.40e11 ≈ 23.9 tok/s
```
Expect measured ~16–20 tok/s (70–80% of ideal). Note it doesn't fit on one H100
(03.3) — assume 2× H100 tensor-parallel; TP splits both the weight bytes *and*
the bandwidth across 2 cards, so the per-token weight-read time is roughly
preserved and the ~24 tok/s ceiling still describes the aggregate. Good enough
for planning.

**2. Batch ceiling from KV cache.** Suppose after weights + overhead you have
~40 GB of HBM free for KV across your 2×H100 (160 GB total − 140 weights − ~20
overhead ≈ 0... tight — in reality you'd quantise or use more GPUs; let's instead
do the cleaner FP8 case). Take **70B at FP8 (~70 GB) on one H100 80GB**:
```
free for KV ≈ 80 − 70 − 4 ≈ 6 GB
KV per seq @ 4k ctx, GQA (8 KV heads × 128) :
  bytes/token = 2 × 80 × (8×128) × 1 byte(FP8 KV) ≈ 163 KB/token
  4096 tokens × 163 KB ≈ 0.67 GB/seq
B_max ≈ 6 GB ÷ 0.67 GB ≈ ~9 concurrent sequences
```
**3. Aggregate throughput.** FP8 single-stream ceiling ≈ 3.35 TB/s ÷ 70 GB ≈
47.9 tok/s; measured ~35. With B≈9: aggregate ≈ 9 × 35 ≈ **~315 tok/s** off one
H100 — versus ~35 tok/s single-stream. **A ~9× throughput gain for the same
rented GPU-hour = ~9× lower cost-per-token.** That gap is the entire economic
argument for batched serving.

**4. Cost framing.** If the H100 rents at ~$3/hr and you sustain 315 tok/s:
```
315 tok/s × 3600 s = 1.13M tokens/hr  →  $3 / 1.13M ≈ $2.65 per 1M output tokens
```
Single-stream (35 tok/s) would be ~$24 per 1M tokens. Same hardware, ~9× the
bill. This is the number the deliverable is built to expose.

## Practice (hands-on — feeds the deliverable)

Rent a GPU (H100 80GB ideal; adjust spec numbers to your SKU) and serve with
**vLLM**. Budget ~1.5 GPU-hours.

1. **Serve a model** you can fit (e.g. an 8B–13B FP16, or a quantised larger
   model). `vllm serve <model>` exposes an OpenAI-compatible endpoint.
2. **Measure single-stream decode tok/s:** send one request, `max_tokens` ~256,
   temperature 0, and record output tokens ÷ decode time (exclude prefill/TTFT).
3. **Measure batched decode tok/s vs batch size:** drive concurrent requests at
   concurrency 1, 2, 4, 8, 16, 32, 64… (use vLLM's benchmark script
   `benchmarks/benchmark_serving.py`, or a simple async client). Record
   **aggregate output tok/s** at each level.
4. **Find the knee:** keep raising concurrency until aggregate throughput stops
   rising (KV-cache/memory cap) or vLLM starts queueing/preempting. Note the
   batch size where it flattens and check `nvidia-smi` / vLLM logs for the
   KV-cache-full signal.
5. **Compare to theory:** compute your bandwidth-derived single-stream ceiling
   (`spec_bandwidth ÷ model_bytes`) and compare to your *measured* single-stream
   number. Report the ratio (measured/ideal) and explain the gap.

**Acceptance:** A **decode-throughput-vs-batch-size curve** (table or plot:
concurrency on x, aggregate tok/s on y) for one model on one SKU, annotated with
(a) the measured single-stream number, (b) the bandwidth-derived ceiling and the
measured/ideal ratio, and (c) the batch size where throughput flattens and why
(KV-cache cap). Add it to the [GPU Efficiency & Cost
Report](../practice/gpu-efficiency-report/README.md) alongside a cost-per-1M-tokens
figure at the best sustained throughput.

## Self-check

**(a) Estimate the single-stream tok/s ceiling for a 70B model at FP16 on an H100. Show the arithmetic.**

**Answer:** Decode reads all weights per token, so tok/s ≤ HBM_bandwidth ÷
model_bytes. Weights = 70e9 × 2 bytes = 140 GB. H100 bandwidth = 3.35 TB/s.
`3.35e12 ÷ 1.40e11 ≈ 23.9 tok/s` — call it **~24 tok/s**. FLOPS never enter the
calculation because decode is memory-bound. Measured will be lower (~16–20 tok/s,
70–80% of ideal) due to KV reads, attention, and imperfect bandwidth use.

**(b) Why does batching help decode but not a single long prefill?**

**Answer:** Decode is memory-bound: the per-token cost is dominated by reading
the weights, and that read is a *fixed cost per forward pass* that can be shared
across many sequences. Batching B sequences reads the weights once and emits B
tokens, amortising the read → near-linear throughput gain, because the compute
was idle anyway. A single prefill is *already compute-bound*: one weight read
already feeds math over all the prompt's tokens at once, so arithmetic intensity
is high and the tensor cores are already saturated. There's no idle compute to
fill, so batching more work onto an already-compute-bound prefill doesn't raise
throughput — you're limited by FLOPS, not by an amortisable memory read.

**(c) At what point does decode itself become compute-bound?**

**Answer:** When batching raises decode's effective arithmetic intensity above
the hardware ridge point (~590 FLOP/byte on H100). Each sequence added to the
batch reuses the same weight read for more math, so effective intensity grows
roughly with batch size; push the batch large enough and you cross from
memory-bound to compute-bound, at which point the tensor cores saturate and
further batching stops adding throughput. In practice you usually hit the
**KV-cache capacity** wall first (HBM fills up long before compute saturates), so
decode stays memory-bound and HBM — capacity and bandwidth — remains the binding
constraint. If you ever see throughput plateau while HBM still has free room,
you've reached the compute-bound regime.

## Resources

1. **"Prefill is compute-bound, decode is memory-bound — why your GPU shouldn't
   do both."** The definitive walkthrough of the two-regime model and why it
   drives disaggregation. *Read in full.*
   https://towardsdatascience.com/prefill-is-compute-bound-decode-is-memory-bound-why-your-gpu-shouldnt-do-both/
2. **Horace He — "Making Deep Learning Go Brrrr From First Principles."** The
   clearest intuition for compute-bound vs memory-bound, arithmetic intensity,
   and roofline. Foundational; read before or alongside #1.
   https://horace.io/brrr_intro.html
3. **vLLM documentation** (continuous batching, PagedAttention, and the serving
   benchmarks). *Skim* for the mechanisms (inflight batching, KV paging) and use
   `benchmark_serving.py` for the practice. https://docs.vllm.ai/
