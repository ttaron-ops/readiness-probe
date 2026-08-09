---
lesson: "07.3"
title: "PagedAttention and vLLM"
module: "07"
concept: "PagedAttention and vLLM"
status: not-started
est_time: "6h"
artifacts: []
---
# 07.3 · PagedAttention and vLLM

> **Concept.** PagedAttention treats the KV cache like OS virtual memory — fixed-size non-contiguous blocks addressed through a per-sequence block table — which kills fragmentation, enables prefix sharing via copy-on-write, and is what lets vLLM's continuous-batching scheduler pack many sequences onto one GPU.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

---

## Why this matters

You already know from module 03 that decode is memory-bandwidth-bound and that the
KV cache — not the weights — is the thing that grows per request and per token. In
07.2 you saw the arithmetic: KV bytes ≈ `2 · layers · kv_heads · head_dim · seq_len ·
dtype_bytes`, and that this is what caps how many sequences you can run at once.

The naïve implementation (the one every pre-2023 server used, and the one a hand-rolled
HuggingFace `generate()` loop still uses) allocates one **contiguous** KV buffer per
request, sized to `max_model_len`. That is catastrophic for utilization: a request that
generates 200 tokens against a 32K context window reserves 32K worth of KV and uses 0.6%
of it. The vLLM paper measured **60–80% of KV memory wasted** to internal fragmentation,
external fragmentation, and over-reservation in those systems. Wasted KV memory is
directly wasted GPU — it caps `max-num-seqs`, which caps batch size, which (module 03
roofline) is the only lever that moves decode off the bandwidth wall toward compute.

For a platform engineer this is the whole cost story. The unit economics deliverable —
cost per million tokens — is `GPU_$/hr ÷ tokens/hr`, and `tokens/hr` is set by how many
sequences you keep resident and busy. PagedAttention is the mechanism that turns "60% of
your H100 HBM is stranded" into "≥96% of it is usable KV." Everything downstream —
throughput, the max batch you can admit, whether a shared system prompt is free or paid
for N times — flows from this one idea.

---

## What's new here vs 03 / 05

- **Module 03** gave you the *physics*: prefill vs decode, the roofline, why batch size
  is the throughput lever, KV cache as a concept, FP8. This lesson is the *memory
  allocator* that makes a large batch physically fit.
- **Module 05** gave you the *SLIs*: TTFT, TPOT, and the vLLM `/metrics` endpoint. Here
  you'll cause a measurable TTFT drop (prefix cache hit) and read it back through those
  same metrics — the observability you built is the acceptance instrument.
- **07.2** gave you KV-cache sizing. New here: the KV cache is not one buffer, it's a
  *pool of blocks* with an indirection layer, and the scheduler churns that pool every
  single decode step.

---

## Core notes

### 1. The problem: contiguous KV allocation fragments

Contiguous per-request KV has three losses, exactly mirroring the memory-management
problems an OS solves:

- **Internal fragmentation** — you reserve `max_model_len` but the request stops early;
  the tail is dead.
- **External fragmentation** — variable-length buffers freed at different times leave
  holes too small to reuse. The pool looks like Swiss cheese.
- **Over-reservation** — you cannot admit a new sequence unless a `max_model_len`-sized
  hole exists, even though the running sequences are nowhere near their limit.

### 2. PagedAttention: paging for the KV cache

The KV cache of each sequence is partitioned into **KV blocks**. Each block holds the
keys and values for a **fixed number of tokens** — vLLM's default `block_size` is **16**.
Blocks live in **non-contiguous** physical GPU memory. A per-sequence **block table** maps
logical block index → physical block number. This is textbook virtual memory:

| OS virtual memory | PagedAttention |
|---|---|
| Process address space | A sequence's KV cache |
| Page | KV block (16 tokens) |
| Page table | Block table |
| Physical frame | Physical KV block in HBM |
| Page fault → allocate frame | New block allocated on demand as decode advances |

The attention kernel is written to *gather* K/V through the block table, so physical
non-contiguity costs almost nothing. What the block table buys you:

- **Near-zero fragmentation.** The only waste is internal, in the *last* partially-filled
  block of each sequence — bounded by `block_size − 1` tokens per sequence, i.e. **< one
  block ever**, independent of context length. The paper reports **< 4% waste** vs the
  60–80% above. External fragmentation is gone entirely — every free block is
  interchangeable.
- **On-demand growth.** A sequence starts with the blocks its prompt needs and grabs one
  more block every 16 generated tokens. No pre-reservation to `max_model_len`.
- **Admission without contiguous holes.** The scheduler admits a new sequence whenever
  *enough free blocks exist anywhere*, not when a contiguous slab exists. This is what
  physically enables a high `max-num-seqs` — the concurrency win from 07.2.

Block-size trade-off: smaller blocks → less internal fragmentation and finer-grained
prefix sharing; larger blocks (32–64) → smaller block tables and better locality, useful
for 100K+ contexts. 16 is the default and rarely worth changing.

### 3. Copy-on-write and shared prefixes

Because a block is content-addressable, two sequences whose token prefixes are identical
can **point their block tables at the same physical blocks** — one copy in HBM, referenced
N times (a refcount per block). When a sequence needs to *diverge* (write token 17 of a
block whose first 16 tokens are shared), vLLM does **copy-on-write**: it copies that one
block, bumps the refcount down on the original, and the writer mutates its private copy.

What COW buys you for a shared system prompt: if 500 concurrent requests all begin with
the same 2,000-token system prompt, PagedAttention stores that prefix's KV **once**, not
500 times. At ~2 layers·kv_heads·head_dim·2 bytes per token that is the difference between
"the system prompt is free" and "the system prompt costs you 499× its KV every batch."
This is also how beam search and parallel sampling (`n>1`) share the prompt KV for free.

### 4. Automatic Prefix Caching (APC)

COW handles prefixes shared *within* a batch. **Automatic Prefix Caching** extends it
*across time*: after a request finishes, its prompt blocks aren't immediately freed —
they're hashed (each block keyed by its token content chained with the prefix before it)
and kept in a pool under **LRU eviction**. A later request with the same prefix looks up
the hash, finds the blocks already populated, and **skips prefill for the cached prefix
entirely** — reusing computed KV instead of recomputing attention over those tokens.

The effect you can measure: the *first* request pays full prefill (full TTFT); the
*second* request sharing the prefix has its prefill for the shared span served from cache,
so **TTFT drops sharply** — only the uncached suffix is prefilled. In **vLLM V1, APC is on
by default**: V1 restructured the KV-cache manager for constant-time eviction and minimal
Python-object churn, so prefix caching costs **< 1% throughput even at a 0% hit rate** —
which is why it's no longer opt-in like it was in V0.

### 5. Continuous (iteration-level) batching

Static batching (the transformers-style baseline): pick B requests, run them to
completion together, the batch finishes when the *slowest* sequence finishes. A 20-token
reply and a 2,000-token reply in the same batch means the short one's slot sits idle for
1,980 steps. Utilization tanks.

**Continuous batching** (a.k.a. iteration-level scheduling) is vLLM's answer: the scheduler
runs **one decode step for the whole running set, then re-plans** — every iteration it can
**retire** finished sequences (free their blocks) and **admit** waiting ones into the freed
slots. A sequence never waits for its batch-mates; the instant one emits EOS its blocks
return to the pool and a queued request takes its place. Combined with PagedAttention's
cheap admission, this keeps the GPU saturated with useful decode work.

Key scheduler concepts:

- **Token budget** — `max_num_batched_tokens` caps tokens processed per step; `max-num-seqs`
  caps concurrent sequences. The scheduler fills up to those bounds each iteration.
- **Chunked prefill** — a long prompt's prefill is split into chunks and interleaved with
  ongoing decodes, so one big prefill doesn't stall every decoding request's TPOT. **On by
  default in V1.**
- **Unified scheduling (V1)** — the V1 engine rewrite dropped V0's separate prefill/decode
  phases; a single step can mix prefill chunks and decodes. Tutorials written against V0's
  scheduler (and its default `swap` preemption) are stale — **pin to the V1 engine**.

### 6. Version pin

Everything above is the **V1 engine**, the default and only engine in current vLLM (V0 was
removed in the 0.11.x line — this lesson targets **vLLM 0.11.x**, V1). If a doc mentions
`--enforce-eager` toggling scheduler behavior, a `BlockSpaceManagerV1`, or `swap` as the
default preemption mode, it predates V1; ignore it.

---

## Worked example

Two requests sharing a long system prompt, on one GPU. Model: `meta-llama/Llama-3.1-8B-Instruct`.

Serve (APC is default-on in V1, shown explicitly for clarity):

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --enable-prefix-caching \
  --gpu-memory-utilization 0.90 \
  --max-model-len 8192 \
  --port 8000
# startup log will print the KV-cache profile, e.g.:
#   GPU KV cache size: 231,424 tokens
#   Maximum concurrency for 8192 tokens per request: 28.25x
```

That "Maximum concurrency … 28.25x" line *is* PagedAttention talking: 231,424 usable KV
tokens ÷ 8,192 = 28 sequences at full context, because blocks are shared from one pool
rather than reserved per request.

Fire request A (cold), then B (same 2,000-token system prompt, different question):

```bash
SYS=$(python -c "print('You are a meticulous SRE assistant. '*130)")  # ~2k tokens

curl -s localhost:8000/v1/chat/completions -H 'content-type: application/json' -d "{
  \"model\":\"meta-llama/Llama-3.1-8B-Instruct\",
  \"messages\":[{\"role\":\"system\",\"content\":\"$SYS\"},
                {\"role\":\"user\",\"content\":\"Explain etcd quorum loss.\"}],
  \"max_tokens\":16}" | python -c "import sys,json;print('A done')"

curl -s localhost:8000/v1/chat/completions -H 'content-type: application/json' -d "{
  \"model\":\"meta-llama/Llama-3.1-8B-Instruct\",
  \"messages\":[{\"role\":\"system\",\"content\":\"$SYS\"},
                {\"role\":\"user\",\"content\":\"Explain a split-brain scenario.\"}],
  \"max_tokens\":16}" | python -c "import sys,json;print('B done')"
```

Read the prefix-cache counters from `/metrics` (the module-05 endpoint):

```bash
curl -s localhost:8000/metrics | grep -E 'prefix_cache_(queries|hits)_total'
# vllm:prefix_cache_queries_total{...}  ~2040   (A: full prompt queried, 0 cached)
# vllm:prefix_cache_hits_total{...}     ~2000   (B: the 2k system prefix hit)
```

Hit rate ≈ hits/queries after B ≈ the shared prefix / total prompt tokens. In Prometheus
you'd track it rolling: `rate(vllm:prefix_cache_hits_total[1m]) /
rate(vllm:prefix_cache_queries_total[1m])`. B's server-side TTFT (module 05's
`vllm:time_to_first_token_seconds`) drops to a fraction of A's because ~2,000 prefill
tokens were served from cache and only the short user turn was prefilled.

---

## Practice — feeds the deliverable

On a **rented GPU** (1× A10/L4/A100 is plenty for 8B):

1. `vllm serve meta-llama/Llama-3.1-8B-Instruct --enable-prefix-caching
   --gpu-memory-utilization 0.90 --max-model-len 8192`. Capture the startup
   "Maximum concurrency …" line — that's your PagedAttention utilization number.
2. Build a fixed ~2,000-token system prompt. Send **request A** (cold). Record its TTFT
   from `vllm:time_to_first_token_seconds` (histogram — take the sum/count delta, or use a
   client that reports first-token latency). Snapshot `prefix_cache_queries_total` and
   `prefix_cache_hits_total`.
3. Send **request B** with the *same* system prompt, different user turn. Record B's TTFT
   and re-snapshot the counters. Compute Δhits (should ≈ the prefix length) and the TTFT
   delta A→B.
4. Optional contrast: restart with `--no-enable-prefix-caching`, repeat, confirm B's TTFT
   no longer improves.

**Acceptance (deliverable):** a recorded, reproducible **prefix-cache TTFT improvement** —
B's TTFT measurably below A's (typically a large multiple for a 2k-token shared prefix) —
with the `prefix_cache_hits_total` delta showing the cached-token count, written into the
cost-per-token workbook as evidence that shared-prefix traffic (system prompts, few-shot
templates) is served cheaper. Note the concurrency figure from step 1 next to it: both are
inputs to `tokens/hr`, hence to $/1M tokens.

---

## Self-check

**(a) Why non-contiguous fixed-size blocks — what does the block table buy you over
contiguous allocation?**

**Answer:** Contiguous per-request KV forces you to reserve `max_model_len` up front and
to find a contiguous hole to admit a request, producing internal fragmentation (early-
finishing requests waste their reserved tail), external fragmentation (freed variable-size
buffers leave unusable holes), and over-reservation (60–80% of KV stranded). Fixed-size
blocks addressed through a block table make every free block interchangeable (no external
fragmentation), grow KV on demand one block at a time (no over-reservation), and bound
internal waste to the last partial block per sequence — under one block, i.e. < ~4% total.
That reclaimed memory is what physically permits a high `max-num-seqs`.

**(b) What does copy-on-write buy you for a shared system prompt across many requests?**

**Answer:** Identical prompt prefixes hash to the same content, so many sequences point
their block tables at the *same physical KV blocks* (refcounted) — the shared system prompt
is stored **once** in HBM instead of once per request. Copy-on-write means a block is only
duplicated at the moment a sequence diverges from the shared prefix, so up to that point
the prompt costs one copy regardless of concurrency. For 500 requests sharing a 2k-token
system prompt this is the difference between 1× and ~500× the prefix's KV per batch, and
it's the in-batch basis that Automatic Prefix Caching extends across time.

**(c) What is continuous batching and how does it differ from static batching?**

**Answer:** Static batching picks B requests and runs them to completion together; the
batch is held hostage by its slowest sequence, so short replies leave their slots idle for
the rest of the batch. Continuous (iteration-level) batching re-plans **every decode
step**: it retires any sequence that just finished (freeing its blocks immediately) and
admits queued sequences into the freed slots, within a token/`max-num-seqs` budget, mixing
prefill chunks and decodes in one step under V1. No sequence waits for its batch-mates, so
the GPU stays saturated with useful decode work — which, with PagedAttention's cheap
block-based admission, is what turns a high `max-num-seqs` into actual throughput.

---

## Resources

1. **"Inside vLLM: Anatomy of a High-Throughput Inference System"** —
   https://vllm.ai/blog/2025-09-05-anatomy-of-vllm — the deep, current (V1) walkthrough of
   scheduler + block manager + APC. Read this first.
2. **PagedAttention paper** ("Efficient Memory Management for LLM Serving with
   PagedAttention") — https://arxiv.org/abs/2309.06180 — the fragmentation measurements and
   the OS-paging analogy, first-hand.
3. **vLLM source** — https://github.com/vllm-project/vllm — skim `vllm/v1/core/` (the V1
   scheduler and KV-cache/block manager) to see the block table and refcounting for real.
