---
lesson: "03.3"
title: "Memory hierarchy & HBM: what fits and how fast"
module: "03"
concept: "Memory hierarchy & HBM: what fits and how fast"
status: not-started
est_time: "4h"
artifacts: []
---
# 03.3 · Memory hierarchy & HBM: what fits and how fast
> **Concept.** HBM capacity decides what *fits*; HBM bandwidth decides how *fast* you decode.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Why this matters

Almost every GPU purchasing mistake, and almost every "why is inference so
expensive?" postmortem, traces back to one of two numbers on the spec sheet:
**how many bytes of HBM the card has**, and **how fast it can move them**. These
are different levers with different price tags, and vendors ship SKUs that move
one without the other.

Concretely: an H100 80GB card and an H200 141GB card use the *same compute die*
(GH100). Same FLOPS, same tensor cores, same silicon doing the math. The H200
just bolts on more and faster memory: 141GB of HBM3e at **4.8 TB/s** versus the
H100's 80GB of HBM3 at **3.35 TB/s**. If GPUs were priced by compute, these two
would cost the same. They don't — and the reason the H200 commands a premium is
almost entirely about the two numbers in this lesson.

For a platform engineer optimising cost-per-token, this is the whole ballgame.
You will be asked "can we serve model X on SKU Y?" (a *capacity* question) and
"how many tokens/sec will we get?" (a *bandwidth* question). Getting these wrong
means either OOM crashes in production or paying for compute you can never use
because you're memory-starved. Both are cost events.

## The platform-engineer's lens

**Extract for cost. Skip the kernel depth.**

You are not going to write a tiled matmul or hand-manage shared memory. So read
the memory hierarchy as a **latency/bandwidth ladder** you reason *about*, not a
thing you program *against*:

- **What to extract:** the two buying levers (capacity vs bandwidth), how to
  compute a memory budget for a given model on a given SKU, and why bandwidth —
  not FLOPS — sets the decode throughput ceiling (that derivation is lesson
  03.4; here we build the capacity/bandwidth foundation).
- **What to skip:** register allocation, bank-conflict avoidance, coalescing
  rules, `__shared__` tiling, cache-line eviction policy. That's kernel-author
  territory. You need to know the *tiers exist and roughly how fast each is*, so
  that when a kernel engineer or a profiler says "we're L2-bound" or "this is a
  bandwidth-bound op," you know what they mean and what it costs.

The one habit worth building: whenever you see a model + a SKU, reflexively do
the **memory budget** — weights + KV cache + activations/overhead vs the HBM
capacity. That single calculation answers most of the questions you'll be paid
to answer.

## Core notes

### The hierarchy as a ladder

On a Hopper GPU the memory tiers, fastest/smallest to slowest/largest:

| Tier | Scope | Approx size | Approx bandwidth | Latency |
|------|-------|-------------|------------------|---------|
| Registers | per-thread | ~256 KB/SM (regfile) | ~tens of TB/s aggregate | ~1 cycle |
| Shared mem / L1 | per-SM (per thread-block) | up to 228 KB/SM (H100) | ~tens of TB/s aggregate | ~20–30 cycles |
| L2 cache | whole GPU | 50 MB (H100) | ~several TB/s | ~200 cycles |
| HBM (global) | whole GPU | **80 GB (H100) / 141 GB (H200)** | **3.35 / 4.8 TB/s** | ~400–600 cycles |

Two things to internalise from this table:

1. **Each rung down is roughly an order of magnitude bigger and slower.** The
   art of kernel writing is keeping hot data on the top rungs. The art of
   *platform* work is knowing that once your working set spills to HBM — which
   for LLM inference it always does, because the weights are tens of GB — HBM
   bandwidth is your ceiling.
2. **Only HBM capacity is measured in "GB you can spend."** Registers, shared
   memory, and L2 are fixed, tiny, and not something you budget model weights
   into. When someone says "the model doesn't fit," they mean HBM.

### HBM: high-bandwidth memory, and why it's stacked

HBM is DRAM dies stacked vertically and mounted next to the GPU die on the same
package (on a silicon interposer), connected by a very wide bus (thousands of
bits). That physical arrangement is *why* it hits TB/s bandwidths that
board-level GDDR or DDR cannot. It's also why it's expensive and capacity-capped:
you can only stack so high and place so many stacks around the die.

- **H100 80GB:** 5 stacks of HBM3, 3.35 TB/s.
- **H200 141GB:** HBM3e, 4.8 TB/s — same die, denser/faster stacks.

The takeaway for buying: **capacity and bandwidth are somewhat independent knobs
the vendor turns.** A generational memory upgrade (HBM3 → HBM3e) can raise both
without touching the compute die at all.

### Capacity: what FITS

HBM capacity is a hard wall. Your resident footprint for LLM inference is:

```
HBM used  ≈  weights  +  KV cache  +  activations/overhead
```

- **Weights** — fixed per model + precision. Bytes ≈ params × bytes/param.
  FP16/BF16 = 2 bytes/param. So a 70B model at FP16 ≈ **140 GB**. (INT8 halves
  it to ~70 GB; FP8 likewise ~70 GB; INT4 ~35 GB — precision is lesson 03.4/04
  territory, but note it's a capacity lever too.)
- **KV cache** — grows with concurrent sequences × their lengths. This is the
  variable, and it's what caps how many requests you can batch (lesson 03.4).
- **Overhead** — CUDA context, activation buffers, fragmentation, framework
  reserve. Budget a few GB; vLLM et al. reserve a configurable fraction.

If `HBM used > capacity`, you either shard across GPUs, quantise, or pick a
bigger-memory SKU. All three are cost decisions.

### KV cache: the sizing formula

During generation, every token's attention needs the Keys and Values of every
prior token, at every layer. Rather than recompute them, they're cached in HBM.
Size **per token** (no GQA/MQA compression, FP16):

```
bytes/token ≈ 2 × num_layers × hidden_dim × 2 bytes
              │                              └ FP16 = 2 bytes/element
              └ one K + one V
```

Then per sequence multiply by `seq_len`, and across a batch multiply by the
number of concurrent sequences:

```
KV bytes ≈ 2 × layers × hidden × 2 × seq_len × batch
```

Worked below. The headline: **KV cache scales linearly with (batch × context
length).** Double the context window or double the concurrency and you double
the KV footprint. On long-context, high-concurrency serving, KV cache can rival
or exceed the weights — which is exactly why the extra 61 GB on an H200 matters.

> Note: real models use **GQA/MQA** (grouped/multi-query attention), which
> shares K/V across query heads and shrinks KV by the head-grouping ratio.
> Replace `hidden_dim` with `num_kv_heads × head_dim` for a precise figure. The
> formula above is the conservative upper bound — good enough for budgeting.

### Bandwidth: how FAST

HBM bandwidth is bytes/sec off the memory stacks. It matters because **LLM
decode is memory-bound**: to generate one token, single-stream, the GPU must
read *every weight* from HBM once. So the ceiling is roughly:

```
tokens/sec  ≈  HBM_bandwidth  ÷  model_bytes
```

For a 70B FP16 model on an H100: 3.35 TB/s ÷ 140 GB ≈ **~24 tok/s**, no matter
how many FLOPS the die has. This is the single most important number-shape in
GPU-inference economics, and lesson 03.4 derives and measures it in full. Here,
just hold the intuition: **compute (FLOPS) doesn't set decode speed — bandwidth
does.**

### Why capacity vs bandwidth are different buying levers

| Question | Governed by | Lever |
|----------|-------------|-------|
| Does the model + KV cache fit? | HBM **capacity** (GB) | bigger card / shard / quantise |
| How fast does it decode? | HBM **bandwidth** (TB/s) | faster memory / smaller model / batch |
| How much math per second? | **FLOPS** (compute die) | rarely the binding constraint for decode |

You can be capacity-bound (won't fit) but have bandwidth to spare, or fit
comfortably but be bandwidth-starved. Diagnosing *which* wall you're hitting is
the platform engineer's core skill, because the fix — and its cost — differ.

## Worked example

**Memory budget: Llama-3-70B at FP16, on one H100 80GB vs one H200 141GB.**

*Llama-3-70B config:* 80 layers, hidden_dim 8192.

**1. Weights**
```
70e9 params × 2 bytes = 140 GB
```

**2. KV cache, one sequence at 8k context** (upper bound, ignoring GQA):
```
bytes/token = 2 × 80 × 8192 × 2 = 2,621,440 B ≈ 2.62 MB/token
8192 tokens × 2.62 MB ≈ 21.5 GB per sequence
```
(Llama-3-70B actually uses GQA with 8 KV heads × 128 head_dim = 1024 effective,
so real KV ≈ 8192/1024 = 8× smaller ≈ 2.7 GB/seq. The uncompressed 21.5 GB is
the conservative planning figure; note the 8× swing GQA buys you.)

**3. Overhead:** ~3–4 GB context + framework reserve.

**Verdict:**
```
H100 80GB:  140 GB weights ALONE > 80 GB  → DOES NOT FIT. Not even the weights.
            Requires ≥2× H100 (tensor-parallel shard) or INT8/FP8 (~70 GB).
H200 141GB: 140 GB weights ≈ 141 GB capacity → weights *barely* fit,
            leaving ~1 GB — effectively NO room for KV cache. In practice you'd
            still shard across 2× H200, or quantise, to leave room to serve.
```

The lesson: a 70B FP16 model is a **≥2-GPU model** on Hopper regardless of SKU;
the H200's capacity edge shows up more for 30–40B-class FP16 models, or for
giving a fitted model much more KV-cache headroom (long context / high batch).

**4. Bandwidth sanity check (decode ceiling):**
```
H100:  3.35 TB/s ÷ 140 GB ≈ 23.9 tok/s (single stream, if it fit)
H200:  4.80 TB/s ÷ 140 GB ≈ 34.3 tok/s
```
Same die, ~43% faster decode — purely from bandwidth. That's the H200 inference
value proposition in one line.

## Practice (hands-on — feeds the deliverable)

Rent one GPU (H100 80GB if available; any single modern datacentre GPU works —
adjust the spec numbers to your SKU). Target ~1 hour of GPU time.

1. **Load a model and inspect resident memory.** Pick a model that fits your SKU
   (e.g. an 8B or 13B at FP16 on an 80GB card). Load it in PyTorch/transformers
   and record:
   - `nvidia-smi` memory used before vs after load.
   - `torch.cuda.memory_allocated()` after load = weights footprint.
   Confirm it matches `params × 2 bytes` within a few %.
2. **Compute the weights footprint by hand** at FP16 and compare to your 80 GB
   (or actual SKU) budget. Note the headroom.
3. **Estimate KV-cache size per sequence** using
   `2 × layers × hidden × 2 bytes × seq_len` for two context lengths (e.g. 2k
   and 32k). Show how it scales.
4. **Run an HBM bandwidth microbenchmark.** Simplest reliable approach: a large
   `torch` elementwise copy/add on a multi-GB tensor, timed, computing
   `bytes_moved / seconds`. (Read + write counts as 2× the tensor size per
   element.) Or use NVIDIA's `bandwidthTest` (CUDA samples) / a GEMM at large
   sizes. Record achieved GB/s.
5. **Compare achieved vs spec** (3.35 TB/s for H100). Expect to reach ~70–85% of
   spec on a well-formed benchmark; note your efficiency and why real workloads
   see less.

**Acceptance:** A **memory-budget breakdown** for *one model on one SKU* —
weights (measured + hand-computed), KV-cache estimate at one context length,
overhead, total vs HBM capacity, plus your measured-vs-spec HBM bandwidth (GB/s
and % of spec). Drop it into the [GPU Efficiency & Cost
Report](../practice/gpu-efficiency-report/README.md). One table + three sentences
of interpretation is enough.

## Self-check

**(a) Does a 70B model at FP16 fit on one H100 (80GB)? On one H200 (141GB)? Show the math.**

**Answer:** Weights = 70e9 × 2 bytes = **140 GB**.
- H100 80GB: 140 GB > 80 GB → **no**, the weights alone don't fit (you're ~60 GB
  over before any KV cache). Needs ≥2× H100, or quantisation to INT8/FP8 (~70 GB,
  which then fits with room for KV).
- H200 141GB: 140 GB < 141 GB → the weights *technically* fit, but with only ~1
  GB left there's no room for KV cache or overhead, so it's not servable in
  practice on a single card. Realistically still a 2-GPU deployment, or quantise.
  A 70B at FP16 is a ≥2-GPU model on Hopper.

**(b) How does KV-cache size scale with batch × context length?**

**Answer:** Linearly in both, and independently:
`KV bytes ≈ 2 × layers × hidden × bytes/elem × seq_len × batch`. Doubling the
context length doubles KV; doubling the number of concurrent sequences (batch)
doubles KV. So KV grows as the *product* (batch × context). This is why
long-context, high-concurrency serving is KV-cache-bound, and why KV footprint —
not weights — usually caps how large a batch you can run (lesson 03.4). GQA/MQA
reduce the constant factor but not the linear scaling.

**(c) Why is the H200 (same compute die as the H100) often the better inference buy?**

**Answer:** Because inference — specifically decode — is **memory-bound, not
compute-bound**. The H200 keeps the identical GH100 compute die but upgrades
memory to 141 GB @ 4.8 TB/s (vs 80 GB @ 3.35 TB/s). The extra *capacity* lets
more of the model + KV cache fit on fewer GPUs (fewer shards, more batch
headroom), and the extra *bandwidth* directly raises the decode throughput
ceiling by ~43% (tok/s ≈ bandwidth ÷ model_bytes). You pay nothing for more
FLOPS you couldn't use anyway; you pay for the two things that actually gate
inference throughput and fit. Higher tokens/sec on the same power/rack footprint
is lower cost-per-token.

## Resources

1. **NVIDIA H100 / Hopper architecture whitepaper — memory subsystem section.**
   Authoritative source for HBM capacity/bandwidth, L2 size, and the memory
   hierarchy on Hopper. https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper
2. **PMPP (Programming Massively Parallel Processors), Ch. 5 — Memory
   architecture.** *Skim, for the mental model only* — the register → shared →
   global ladder and why bandwidth-bound kernels exist. Ignore the CUDA coding
   exercises; you want the tier intuition, not the tiling technique.
3. **A KV-cache sizing writeup** (e.g. the vLLM PagedAttention paper/blog, or any
   "how big is the KV cache" explainer) to cross-check the sizing formula and see
   how GQA and paging change the real footprint. https://blog.vllm.ai/2023/06/20/vllm.html
