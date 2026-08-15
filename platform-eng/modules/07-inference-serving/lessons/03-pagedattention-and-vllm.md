---
lesson: "07.3"
title: "PagedAttention and vLLM"
module: "07"
concept: "PagedAttention and vLLM"
status: not-started
est_time: "8h"
prev: "02-kv-cache-concurrency.md"
next: "04-vllm-in-production.md"
artifacts: []
sources: 6
---
# 07.3 · PagedAttention and vLLM

> **Concept.** PagedAttention treats the KV cache like OS virtual memory — fixed-size non-contiguous blocks addressed through a per-sequence block table — which kills fragmentation, enables prefix sharing via copy-on-write, and is what lets vLLM's continuous-batching scheduler pack many sequences onto one GPU.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

---

## Where this fits

07.2 established the concurrency cap as pure arithmetic: `KV_pool_bytes ÷ per-request KV
footprint`, and showed that a naïve contiguous allocator throws away 60–80% of that pool to
fragmentation and over-reservation. That lesson stopped at diagnosis — you could compute
the waste, and read `kv_cache_usage_perc` pinning at 1.0 as the KV-bound signature, but you
hadn't yet seen the fix. This lesson is the fix: the actual allocator design (PagedAttention)
and the scheduler built on top of it (vLLM's continuous batching) that reclaim that stranded
memory and turn it into admitted requests. It is the anchor lesson of the module — everything
in 07.4 (production tuning), 07.5 (the CPM curve), and 07.6–10 (disaggregation, quantization,
autoscaling, multi-LoRA) assumes you understand blocks, block tables, and iteration-level
scheduling as load-bearing mechanism, not folklore. What it unlocks next: 07.4 takes the
knobs this lesson introduces (`gpu-memory-utilization`, block pool sizing) and shows what
happens when you push them past the edge — preemption — which is the operational side of
the same allocator.

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

It is also, concretely, the question that gates the interview loop this module targets.
"Design an LLM inference platform" always drills into KV-cache memory management within
the first ten minutes, and a candidate who can only say "vLLM handles it" fails the
follow-up ("handles it *how*?"). Being able to draw the block table, explain copy-on-write,
and say precisely why fragmentation drops from ~70% to <4% is the difference between a
pass and a "strong platform background, weak on inference internals" writeup.

---

## What's new here (calibration)

Per the module README: you did 03 (prefill/decode, roofline, memory-bound decode, KV
cache *concept*, FP8) and 05 (TTFT/TPOT/ITL, queue-depth) — those are referenced, not
re-taught. This lesson is squarely in the "new" list: **the PagedAttention mechanism
itself** and how vLLM turns it into a production scheduler.

- **Module 03** gave you the *physics*: prefill vs decode, the roofline, why batch size
  is the throughput lever, KV cache as a concept, FP8. This lesson is the *memory
  allocator* that makes a large batch physically fit.
- **Module 05** gave you the *SLIs*: TTFT, TPOT, and the vLLM `/metrics` endpoint. Here
  you'll cause a measurable TTFT drop (prefix cache hit) and read it back through those
  same metrics — the observability you built is the acceptance instrument.
- **07.2** gave you KV-cache sizing. New here: the KV cache is not one buffer, it's a
  *pool of blocks* with an indirection layer, and the scheduler churns that pool every
  single decode step.

Out of scope here (later lessons): production config tuning and preemption mechanics
(07.4), turning throughput into a CPM curve (07.5), disaggregated prefill/decode serving
(07.6), and quantization of the KV cache itself (07.7) — this lesson gives you the
mechanism those all build on.

---

## Core concepts

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

As text: imagine a sequence's block table as a small array — index 0 → physical block
#4711, index 1 → physical block #92, index 2 → physical block #5003. The physical blocks
are scattered across HBM in whatever order they happened to be free; the sequence doesn't
care, because the attention kernel dereferences the table on every step. Two *different*
sequences' block tables can point index 0 at the *same* physical block #4711 — that's
copy-on-write prefix sharing (§3) — something a contiguous allocator has no way to express
at all.

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
which is why it's no longer opt-in like it was in V0. That "<1% even at 0% hit rate"
number is worth dwelling on: it means turning APC on is a strict win with essentially no
downside case, which is precisely why the vLLM team felt safe flipping the default — this
was a hard-won engineering result (constant-time LRU with minimal per-request Python
overhead), not a given.

COW and APC are related but have different lifetimes and are worth keeping distinct in
your head (see Pitfalls): COW is *in-batch*, alive only as long as the sharing requests
are concurrently resident; APC is *cross-time*, alive as long as the blocks survive
eviction in the LRU pool, independent of whether the original request is still running.

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

It's worth naming clearly: "continuous batching" is no longer vLLM-specific terminology.
Every serious inference engine (SGLang, TensorRT-LLM, TGI's successors) now does
iteration-level scheduling — it's table stakes, the same way TCP congestion control is
table stakes for a network stack. The real engine-to-engine differentiation today is
*underneath* that scheduling loop: how KV eviction and admission are implemented, how
prefix caching is hashed and evicted, how chunked prefill interacts with the token budget.
An interview answer that says "vLLM does continuous batching, that's why it's fast" is
outdated framing; the sharper answer names the *allocator* underneath.

### 6. PagedAttention is an algorithm, not a vLLM feature

It is worth being explicit about this because it is a common interview and resume-review
trap: PagedAttention is the *paper's* contribution (Kwon et al., SOSP'23), and vLLM was
the first production system to implement it — but it is not vLLM-exclusive. SGLang
implements its own paged KV-cache manager (plus RadixAttention for prefix sharing);
TensorRT-LLM has paged KV-cache support; most modern engines converged on some variant of
block-based, non-contiguous KV storage. Naming vLLM specifically as "the" solution in an
interview reads as brand familiarity rather than mechanism understanding — better to say
"a paged, block-based KV allocator — vLLM popularized it, most engines have converged on
some form of it now."

### 7. Version pin

Everything above is the **V1 engine**, the default and only engine in current vLLM (V0 was
removed in the 0.11.x line — this lesson targets **vLLM 0.11.x**, V1). If a doc mentions
`--enforce-eager` toggling scheduler behavior, a `BlockSpaceManagerV1`, or `swap` as the
default preemption mode, it predates V1; ignore it.

---

## Perspectives

**The production-adopter view.** Red Hat's write-up on enterprise vLLM adoption
(see Real-world use cases below) is useful precisely because it covers two workloads with
almost nothing in common: Roblox is a real-time consumer gaming platform serving billions
of tokens/week to a chat assistant embedded in gameplay; LinkedIn is running 50+ internal
generative-AI use cases across an enterprise SaaS/HR product surface (including a Hiring
Assistant). Both report meaningful latency/throughput wins from the same underlying
mechanism. That's evidence the PagedAttention win is workload-agnostic — it isn't a trick
that only pays off for one traffic shape, it pays off wherever KV was previously stranded.

**The systems / OS-analogy view.** This is the paper's own framing and the one to keep as
your conceptual spine: pages, page tables, physical frames, page faults, copy-on-write —
every one of these has a direct, load-bearing PagedAttention analogue (§2 above). If you
already carry OS memory-management intuition from general platform engineering, you
already have 80% of the model for free; the paper is mostly "here's the mapping," not
"here's a new idea from scratch."

**The maintainer / engine-internals view.** "Inside vLLM: Anatomy of a High-Throughput LLM
Inference System" (vLLM Blog, Sept 2025) is written by the vLLM core team itself — the
closest you'll get to hearing the design decisions from the people who built the V1
scheduler. It's valuable specifically for the *why-V1-not-V0* context: why the block
manager was rewritten, why APC's overhead had to drop before it could default-on, and how
chunked prefill and continuous batching interact inside one unified scheduling loop rather
than two separate phases. Treat it as the canonical current-state reference, over any
blog post predating the V0→V1 rewrite.

**The skeptical / failure-mode view.** PagedAttention does not fix everything, and it's
worth being able to say precisely what it *doesn't* do. It has nothing to say about a
compute-bound, prefill-heavy workload (long prompts, short generations) — that's a FLOPs
problem, not a memory-fragmentation problem, and no allocator changes the roofline. The
paper's headline "<4% waste" is a best case measured on its benchmark configuration; real
deployments see costs the paper's top-line number doesn't fully surface — the attention
kernel's gather-through-block-table has a real (small but nonzero) overhead versus a dense
contiguous read, and block-table bookkeeping (refcounts, hashing, eviction) is CPU/Python
work that competes for the same event loop as scheduling. None of this reverses the
conclusion, but "PagedAttention fixes memory, full stop" is a rounder claim than the
evidence supports — the honest version is "reclaims the overwhelming majority of
previously-wasted KV, at a small, well-amortized bookkeeping cost."

---

## Real-world use cases

- **Roblox** — adopted vLLM as its primary inference engine, reporting a **50% latency
  reduction** while serving **4 billion tokens/week** to its in-product AI Assistant
  feature. *What it shows:* PagedAttention/continuous batching pay off in a real-time,
  consumer-scale, latency-sensitive workload, not just a batch/offline one.
  ([Red Hat](https://www.redhat.com/en/topics/ai/how-vllm-accelerates-ai-inference-3-enterprise-use-cases))
- **LinkedIn** — runs vLLM across **50+ generative-AI use cases**, including the LinkedIn
  Hiring Assistant. *What it shows:* the same mechanism generalizes across a completely
  different traffic shape — internal enterprise tooling, many small models/use cases,
  not one flagship consumer feature.
  ([Red Hat](https://www.redhat.com/en/topics/ai/how-vllm-accelerates-ai-inference-3-enterprise-use-cases))
- **Anyscale's continuous-batching benchmark** — "Achieve 23x LLM Inference Throughput"
  is the canonical, widely-cited quantification of what iteration-level scheduling buys
  over static/naive batching. *What it shows:* the throughput number behind the "batching
  is the biggest lever" claim this module keeps returning to — useful to cite when a
  number, not just a mechanism, is asked for.
  ([Anyscale](https://www.anyscale.com/blog/continuous-batching-llm-inference))
- **vLLM core team's own account** — "Inside vLLM: Anatomy of a High-Throughput LLM
  Inference System" is the maintainers describing their own scheduler, block manager, and
  APC implementation in the current V1 engine. *What it shows:* ground truth on how the
  mechanism actually works today, versus how the 2023 paper originally described V0 — the
  gap between paper and current code is exactly what a senior engineer needs to have
  correct. ([vLLM Blog](https://blog.vllm.ai/2025/09/05/anatomy-of-vllm.html))

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

**Sanity-check the mechanism, not just the number.** If you rerun with
`--no-enable-prefix-caching`, B's TTFT should barely differ from A's — confirming the
improvement really is the block-cache hit, not something incidental (warm CUDA kernels,
OS page cache, etc.). Always pair a "mechanism on" measurement with a "mechanism off"
control when you're building evidence for a deliverable — a single number with no control
is not evidence, it's an anecdote.

---

## Practice

Feeds the deliverable at [`../practice/cost-per-token/README.md`](../practice/cost-per-token/README.md).
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
4. Required contrast: restart with `--no-enable-prefix-caching`, repeat, confirm B's TTFT
   no longer improves. This is your control — without it, step 3's result is unfalsifiable.

**Acceptance (deliverable):** a recorded, reproducible **prefix-cache TTFT improvement** —
B's TTFT measurably below A's (typically a large multiple for a 2k-token shared prefix) —
with the `prefix_cache_hits_total` delta showing the cached-token count, written into the
cost-per-token workbook as evidence that shared-prefix traffic (system prompts, few-shot
templates) is served cheaper. Note the concurrency figure from step 1 next to it: both are
inputs to `tokens/hr`, hence to $/1M tokens.

---

## Common pitfalls

- **Believing PagedAttention == vLLM.** It's an algorithm (Kwon et al., SOSP'23) that
  vLLM was first to productionize, now also implemented (in various forms) by SGLang,
  TensorRT-LLM, and others. Saying "vLLM invented paged memory for LLMs, and it's the only
  one that has it" is factually wrong and reads as brand loyalty rather than mechanism
  understanding in an interview.
- **Assuming Automatic Prefix Caching is free at all hit rates.** The "<1% overhead even
  at 0% hit rate" figure is a specific, hard-won V1 engineering result (constant-time LRU
  eviction, minimal per-request Python-object churn) — it's *why* APC could move from
  opt-in (V0) to default-on (V1). Don't assume every "smart cache" defaults to on for free;
  in V0 this exact feature carried enough overhead that it needed an explicit flag.
- **Conflating copy-on-write with Automatic Prefix Caching.** They're related (both rest
  on content-addressable blocks) but distinct mechanisms with different lifetimes: COW is
  *in-batch* — sequences currently resident together share blocks — while APC is
  *cross-time* — blocks survive after their originating request finishes, sitting in an
  LRU pool for a *future* request to hit. Getting this distinction wrong makes "why did
  request C, which started after A and B both finished, still get a cache hit?" an
  unanswerable question instead of an obvious one (APC, not COW).
- **Thinking "continuous batching" is vLLM-specific terminology.** It's industry-standard
  now (iteration-level scheduling) — SGLang, TensorRT-LLM, and others all do it. The real
  differentiation between engines today is in *how KV eviction and admission are
  implemented underneath* the scheduling loop, not whether the loop is iteration-level.
- **Treating the paper's "<4% waste" as a deployment guarantee.** It's the paper's
  benchmark-configuration result. Real deployments still pay a (much smaller, well-
  amortized) tax in attention-kernel gather overhead and block-table bookkeeping that the
  headline number doesn't fully surface. The conclusion ("massively better than
  contiguous") holds; the specific percentage is a best case, not a universal constant.

---

## Self-check

- **Why non-contiguous fixed-size blocks — what does the block table buy you over
  contiguous allocation?**
  **Answer:** Contiguous per-request KV forces you to reserve `max_model_len` up front and
  to find a contiguous hole to admit a request, producing internal fragmentation (early-
  finishing requests waste their reserved tail), external fragmentation (freed variable-size
  buffers leave unusable holes), and over-reservation (60–80% of KV stranded). Fixed-size
  blocks addressed through a block table make every free block interchangeable (no external
  fragmentation), grow KV on demand one block at a time (no over-reservation), and bound
  internal waste to the last partial block per sequence — under one block, i.e. < ~4% total.
  That reclaimed memory is what physically permits a high `max-num-seqs`.

- **What does copy-on-write buy you for a shared system prompt across many requests?**
  **Answer:** Identical prompt prefixes hash to the same content, so many sequences point
  their block tables at the *same physical KV blocks* (refcounted) — the shared system prompt
  is stored **once** in HBM instead of once per request. Copy-on-write means a block is only
  duplicated at the moment a sequence diverges from the shared prefix, so up to that point
  the prompt costs one copy regardless of concurrency. For 500 requests sharing a 2k-token
  system prompt this is the difference between 1× and ~500× the prefix's KV per batch, and
  it's the in-batch basis that Automatic Prefix Caching extends across time.

- **What is continuous batching and how does it differ from static batching?**
  **Answer:** Static batching picks B requests and runs them to completion together; the
  batch is held hostage by its slowest sequence, so short replies leave their slots idle for
  the rest of the batch. Continuous (iteration-level) batching re-plans **every decode
  step**: it retires any sequence that just finished (freeing its blocks immediately) and
  admits queued sequences into the freed slots, within a token/`max-num-seqs` budget, mixing
  prefill chunks and decodes in one step under V1. No sequence waits for its batch-mates, so
  the GPU stays saturated with useful decode work — which, with PagedAttention's cheap
  block-based admission, is what turns a high `max-num-seqs` into actual throughput.

- **Is PagedAttention a vLLM-exclusive feature? Why does the distinction matter?**
  **Answer:** No — it's an algorithm from the SOSP'23 paper that vLLM was first to
  productionize; SGLang, TensorRT-LLM, and other engines now implement their own paged/
  block-based KV managers. The distinction matters because conflating the two makes an
  interview answer sound like product familiarity rather than mechanism understanding —
  the transferable, hireable knowledge is the block-table/paging idea itself, which shows
  up (in varying implementations) across the whole serving-engine landscape.

- **What does PagedAttention *not* fix?**
  **Answer:** It's a memory-fragmentation fix, not a compute fix — it does nothing for a
  compute-bound, prefill-heavy workload (long prompts, short generations), because that's
  a FLOPs/roofline problem, not a KV-allocation problem. It also isn't literally free: the
  attention kernel's gather-through-block-table and the CPU-side block-table bookkeeping
  (refcounts, hashing, eviction) carry a real, if small and well-amortized, cost that the
  paper's headline "<4% waste" benchmark number doesn't fully surface.

---

## Connections & what's next

Builds directly on 07.2's concurrency-cap arithmetic and the KV-bound signature
(`kv_cache_usage_perc` pinned, `num_requests_waiting` climbing) — this lesson is the
mechanism that moves that cap up. It reuses module 05's `/metrics` endpoint as the
acceptance instrument for the worked example and practice. It also underlies 07.5's
batching-economics curve: the "throughput up 50–100×" claim there is continuous batching
plus PagedAttention's cheap admission, working together.

Next: **07.4 — vLLM in production** takes the knobs this lesson introduced
(`gpu-memory-utilization`, the KV block pool) and asks the operational question — what
happens when you push them past the edge of what the pool can hold? That's preemption,
and it's the flip side of everything covered here: the same block manager that makes
admission cheap also has to decide who gets evicted when the pool runs dry under load.

---

## References & further reading

**Primary sources**
1. **PagedAttention paper** — "Efficient Memory Management for Large Language Model
   Serving with PagedAttention," Kwon et al., SOSP '23 — https://arxiv.org/abs/2309.06180
   (arXiv) / https://dl.acm.org/doi/10.1145/3600006.3613165 (ACM DL, SOSP'23 proceedings) —
   the fragmentation measurements and the OS-paging analogy, first-hand.
2. **vLLM source** — https://github.com/vllm-project/vllm — skim `vllm/v1/core/` (the V1
   scheduler and KV-cache/block manager) to see the block table and refcounting for real.

**Real-world engineering blogs**
3. **Red Hat — "How vLLM accelerates AI inference: 3 enterprise use cases"** —
   https://www.redhat.com/en/topics/ai/how-vllm-accelerates-ai-inference-3-enterprise-use-cases
   — Roblox (50% latency reduction, 4B tokens/week) and LinkedIn (50+ gen-AI use cases)
   as named production adopters. 2025 snapshot; figures will drift.
4. **Anyscale — "Achieve 23x LLM Inference Throughput"** —
   https://www.anyscale.com/blog/continuous-batching-llm-inference — the canonical
   continuous-batching throughput benchmark.

**Deeper dives**
5. **"Inside vLLM: Anatomy of a High-Throughput LLM Inference System"** (vLLM Blog,
   Sept 2025) — https://blog.vllm.ai/2025/09/05/anatomy-of-vllm.html — the deep, current
   (V1) walkthrough of scheduler + block manager + APC, written by the vLLM core team.
   Read this first among the deeper dives.
6. **Aleksa Gordić — "Inside vLLM: Anatomy of a High-Throughput LLM Inference System"**
   — https://www.aleksagordic.com/blog/vllm — same subject matter, independent author's
   framing and commentary; a good second pass if the maintainer post moves too fast.
