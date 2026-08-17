---
lesson: "07.2"
title: "KV cache as a concurrency problem: capacity, fragmentation, and the cap"
module: "07"
concept: "KV cache as a concurrency problem: capacity, fragmentation, and the cap"
status: not-started
est_time: "6h"
prev: "01-inference-workload-shape.md"
next: "03-pagedattention-and-vllm.md"
artifacts: []
sources: 13
---

# 07.2 · KV cache as a concurrency problem: capacity, fragmentation, and the cap

> **Concept.** Your maximum concurrent requests is `free VRAM after weights ÷ per-request KV bytes` — and naive contiguous KV allocation throws away 60–80% of that budget to fragmentation, which is exactly the waste PagedAttention exists to reclaim.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

07.1 established that the KV cache is the **residual** of a fixed HBM budget and that it is the only term that scales with concurrency. It left the residual as an abstract leftover: "whatever is under the utilisation ceiling after weights and working memory." This lesson makes it a number, twice over.

First as **capacity**: how many tokens of KV the pool holds, how many blocks that is in the unit the scheduler actually reasons about, and therefore how many sequences can be simultaneously resident at a given context length. Second as **efficiency**: how much of that capacity a naive allocator would strand before you ever get to use it, and why — because the gap between "how much KV memory I bought" and "how much of it holds real token state" is the single largest inefficiency in pre-2023 LLM serving, and understanding its shape is what makes PagedAttention (07.3) obvious rather than magical.

The job of this lesson is to make you feel the size of the problem *before* you are handed the solution. If you skip to the mechanism you will remember "vLLM uses blocks"; if you work the fragmentation arithmetic first you will remember *why* blocks are the only sane answer, which is the thing that transfers to the next allocator you have to reason about.

Everything version-specific is checked against the **vLLM main branch cloned 2026-08-17 (commit `c1e4387`)** — `vllm/config/cache.py`, `vllm/config/scheduler.py`, `vllm/v1/core/block_pool.py`, `vllm/v1/core/kv_cache_utils.py`, `vllm/v1/kv_cache_interface.py`, `vllm/v1/metrics/loggers.py` — and against module 05 lesson 06's verified metric names.

## Why this matters

The concurrency cap is the number that sets your cost per million tokens, and it is a **memory-management fact, not a scheduler setting**. That distinction is the whole point. You can set `--max-num-seqs 1024` on a card that physically holds 30 sequences' worth of KV at your context length; the flag will be accepted, the startup log will not complain, and you will discover the real ceiling in production as a preemption storm.

Interviewers at serving-heavy shops probe this precisely, because it is a five-minute question that separates people who have shipped from people who have read: *"you have an H100, an 8B model, and an 8k context — how many requests run at once, and where does the memory go?"* If you can compute the cap from model geometry and then explain why a contiguous allocator would have wasted most of it, you have demonstrated the single insight that motivates PagedAttention, prefix caching, KV quantisation, and KV offload in one answer. Getting fragmentation wrong is how teams conclude they "need more GPUs" when what they needed was a better allocator — and that conclusion is expensive in a way that shows up on a quarterly bill rather than a dashboard.

Character.AI's public account makes it concrete at production scale. They describe caching attention KV on host memory between chat turns, and state plainly that KV cache size "determines the maximum batch size that can fit on a GPU" — then attack it at the architecture level, cutting KV size by more than 20× via multi-query attention, hybrid attention horizons and cross-layer KV sharing, without regressing quality. That is a production team treating KV-bytes-per-token as a first-class design target rather than a number they inherited. This lesson gives you the arithmetic to reason the same way, at the scale you actually control.

## What's new here (calibration)

- **Already yours (referenced, not re-derived):** the KV cache as a concept and why decode must re-read it every step (module 03); the three-term HBM budget, `kv_bytes_per_token = 2 · L · n_kv · d · e`, and the `GPU KV cache size` startup line (07.1); TTFT/ITL/TPOT and the current vLLM metric names (module 05).
- **New: the KV cache reframed as a *memory allocator*** — a fixed-byte pool, requests as variable-length allocations against it with unknown final size, and the two failure modes that follow: capacity (how many fit) and fragmentation (how much you waste fitting them).
- **New: blocks as the real unit.** vLLM never allocates KV per token. It allocates fixed-size blocks, and `num_gpu_blocks` — not gigabytes — is the quantity the scheduler admits against and the quantity `kv_cache_usage_perc` reports.
- **New: three different ceilings that all look like "concurrency"** — the KV pool, `max_num_seqs`, and `max_num_batched_tokens` — and how to tell from metrics which one you are actually hitting.
- **New: the fragmentation arithmetic, derived rather than quoted.** You will compute the waste from a workload's length distribution yourself, so the paper's 60–80 % figure becomes a result you can reproduce rather than a statistic you memorise.
- **New: a three-signal incident runbook** for "is this a memory cap or a throughput cap," with real PromQL against verified metric names.

## Core concepts

### 1. Restating the problem as an allocator problem

Strip the transformer away and describe what the serving engine is doing with memory:

- There is a **fixed-size pool** of bytes, sized once at process start (07.1's residual).
- **Allocations arrive** at unpredictable times. Each one is a sequence.
- Each allocation **grows monotonically** — one token's worth of K and V appended per decode step — and **its final size is unknown at allocation time**, because the model decides when to emit an end-of-sequence token.
- Allocations are **freed all at once** when the request finishes, and freeing is the only way capacity returns.
- The pool is **oversubscribed on purpose**: you admit as many sequences as you can, because idle pool space is idle GPU.

That set of properties is not novel. It is the exact profile of a heap allocator serving objects whose lifetime and final size are both unknown, and it has the exact same two failure modes:

- **Capacity failure** — the sum of live allocations exceeds the pool. No allocator design saves you; something must be evicted or refused.
- **Efficiency failure** — the sum of *useful bytes* is far below the pool size, but you still cannot admit another allocation, because the bytes you are holding are not in a usable shape.

The first is physics and you manage it with the arithmetic in §3–§5. The second is a design choice, and §6–§7 show that the obvious design gets it catastrophically wrong.

### 2. Per-token KV bytes, from geometry to a number

For one token of one sequence, the cache stores one key vector and one value vector, in every layer, for every KV head:

```
  kv_bytes_per_token  =  2 × L × n_kv × d × e_kv

     2      one K and one V
     L      num_hidden_layers          — every layer keeps its own cache
     n_kv   num_key_value_heads        — NOT num_attention_heads
     d      head_dim                   — usually hidden_size / num_attention_heads
     e_kv   bytes per element of the KV dtype (2 for bf16/fp16, 1 for fp8)
```

This is not a heuristic. In `vllm/v1/kv_cache_interface.py`, an attention layer's page (one block's worth of cache for one layer) is

```python
unpadded_page_size_bytes = num_heads * storage_block_size * state_content_size_bytes
#                          ^ n_kv      ^ block_size (16)   ^ (head_size + head_size_v) * dtype_size
```

and the pool allocates one such page per layer per block. Divide by `block_size` and multiply by `L` and you have the formula above, with `head_size + head_size_v` being the `2 × d` term for the usual case where K and V have the same head dimension. (They do not always: DeepSeek's MLA and some multimodal encoders differ, which is exactly why vLLM carries `head_size_v` separately.)

**Reference geometry.** Pull `num_hidden_layers`, `num_key_value_heads` and `head_dim` from the model's `config.json` — these values drift between releases and between community re-uploads, so verify rather than trust a table, including this one.

| Model | L | n_q | n_kv | d | KV B/token (bf16) | KV per 8k seat | KV per 32k seat |
|---|---|---|---|---|---|---|---|
| Llama-3.1-8B | 32 | 32 | 8 | 128 | 131,072 (128 KiB) | 1.00 GiB | 4.0 GiB |
| Mistral-7B (GQA) | 32 | 32 | 8 | 128 | 131,072 (128 KiB) | 1.00 GiB | 4.0 GiB |
| Llama-3.3-70B | 80 | 64 | 8 | 128 | 327,680 (320 KiB) | 2.50 GiB | 10.0 GiB |
| Qwen2.5-72B | 80 | 64 | 8 | 128 | 327,680 (320 KiB) | 2.50 GiB | 10.0 GiB |
| Llama-3.1-405B | 126 | 128 | 8 | 128 | 516,096 (504 KiB) | 3.94 GiB | 15.8 GiB |
| *Hypothetical 70B with MHA* | 80 | 64 | **64** | 128 | 2,621,440 (2.5 MiB) | **20.0 GiB** | 80.0 GiB |
| *Hypothetical 70B with MQA* | 80 | 64 | **1** | 128 | 40,960 (40 KiB) | 0.31 GiB | 1.25 GiB |

Three readings from that table:

**(a) The MHA row is the counterfactual that shows GQA carrying the economics.** Same parameter count, same quality class, 8× the KV — which is 8× fewer concurrent seats and, since decode throughput is roughly linear in resident batch well below the ridge point, roughly 8× the cost per million tokens at long context. When you evaluate a candidate model for production, `num_key_value_heads` is a first-class cost input, not a modelling detail you defer to.

**(b) The 405B row is not 16× the 8B row.** It is 3.9×, because `n_kv` stayed at 8 while `L` went from 32 to 126. Per-token KV scales with **depth**, not with parameter count — a wide, shallow model is cheap to cache and a deep, narrow one is not. This is why you cannot estimate KV from "model size" and must read the geometry.

**(c) The dtype term is a free 2×.** `--kv-cache-dtype fp8` stores K and V at one byte per element, halving every number in that table. It is independent of weight precision: bf16 weights with an FP8 cache is a perfectly ordinary configuration. The cost is a small accuracy delta you measure (07.7), not assume.

**Architectures that break the formula.** Be aware of the three that do, because they are increasingly common and each makes the naive formula an overestimate:

- **Sliding-window attention.** Layers with a window of `W` cache at most `W` tokens, not the whole history, so their contribution saturates. Character.AI's "hybrid attention horizons" is precisely this — roughly one global-attention layer in six, the rest windowed — which is where a large slice of their >20× KV reduction comes from.
- **Cross-layer KV sharing.** Adjacent layers share one KV tensor, so the effective `L` in the formula is `L / group_size`.
- **Multi-head latent attention (MLA).** DeepSeek-V2/V3 compress K and V into a single low-rank latent per token per layer and reconstruct per-head K/V on the fly. vLLM carries a dedicated `fp8_ds_mla` cache dtype for it.

vLLM models all of these as separate **KV-cache groups** with their own specs, and its printed concurrency figure sums the per-group block requirement (`get_max_concurrency_for_kv_cache_config` in `vllm/v1/core/kv_cache_utils.py`). **If your model is hybrid, trust the printed line over your formula.**

### 3. Blocks, not bytes — the unit the scheduler actually uses

vLLM never allocates KV per token. The pool is a fixed array of `num_gpu_blocks` blocks, each holding `block_size` tokens of KV for every layer:

```
  block_size            = 16 tokens          (CacheConfig.DEFAULT_BLOCK_SIZE)
  bytes per block       = block_size × kv_bytes_per_token
  num_gpu_blocks        = KV_pool_bytes ÷ bytes per block
  total token capacity  = num_gpu_blocks × block_size
```

For Llama-3.1-8B at bf16 KV: one block is `16 × 131,072 = 2,097,152 B = 2 MiB`. A 52 GB pool is therefore about 24,800 blocks, or ~397,000 tokens of KV.

Three things follow that matter operationally:

- **`vllm:kv_cache_usage_perc` is `blocks_in_use ÷ num_gpu_blocks`.** It is a block-level gauge, not a byte-level one, which is why it moves in visible steps under low load and why it is exact rather than sampled.
- **Every sequence rounds up to a whole block.** A 100-token sequence occupies `ceil(100/16) = 7` blocks = 112 tokens of capacity. That 12-token overshoot is the *only* internal fragmentation a paged allocator has, and it is bounded by `block_size − 1` **per sequence, forever** — independent of how long the sequence gets. Hold that; §6 contrasts it with the alternative.
- **Block size is a real tradeoff, just a small one.** Smaller blocks mean less rounding waste and finer-grained prefix-cache hits (a hit can only land on a block boundary); larger blocks mean shorter block tables, better memory locality in the attention kernel's gather, and fewer per-block bookkeeping operations. 16 is the default and is right for almost everything. Reach for 32 or 64 only for very long contexts where block-table length itself becomes a cost, and re-measure when you do.

### 4. The concurrency cap, three ways

There is not one cap; there are three ways to state it, and mixing them up is how people produce numbers that do not match the log.

```
  ── (a) WORST CASE — what vLLM prints at startup ──────────────────────────
     Every request is assumed to run to the full --max-model-len.

     blocks_per_request = ceil(max_model_len / block_size)
     max_concurrency    = num_gpu_blocks / blocks_per_request

     This is the "Maximum concurrency for N tokens per request: X.XXx" line.
     It is a floor on what you can serve, not a prediction.

  ── (b) EXPECTED — what you actually get ──────────────────────────────────
     Sequences occupy blocks proportional to their CURRENT length, which
     averages far below max_model_len.

     cap ≈ num_gpu_blocks / ceil(avg_live_context / block_size)

     "avg_live_context" is subtle: it is the time-averaged length of a
     resident sequence, which for a request of prompt P generating M tokens
     is roughly P + M/2, not P + M.

  ── (c) HARD SCHEDULER LIMIT — max_num_seqs ───────────────────────────────
     Independent of memory. The scheduler will not put more than this many
     sequences in one forward pass no matter how much pool is free.
```

The cap that binds is `min(a-or-b, c)`, and **which one binds is the most important diagnostic fact about your deployment**, because the fixes are completely different. §9 gives the metric test.

Worked, for the model you will actually rent:

```
  Llama-3.1-8B, bf16 weights, bf16 KV, one H100-80GB, --gpu-memory-utilization 0.90

  usable            = 80 GB × 0.90                          = 72.0 GB
  weights           = 8.03e9 × 2 B                          = 16.1 GB
  non-KV working    (CUDA ctx + allocator + activations + graphs)
                                                            ≈  3.5 GB
  ─────────────────────────────────────────────────────────────────────
  KV pool                                                   ≈ 52.4 GB

  kv_bytes_per_token = 2 × 32 × 8 × 128 × 2                 = 131,072 B
  bytes per block    = 16 × 131,072                         = 2.00 MiB
  num_gpu_blocks     = 52.4e9 / 2.097e6                     ≈ 24,990 blocks
  total token cap    = 24,990 × 16                          ≈ 399,840 tokens

  (a) WORST CASE at --max-model-len 8192:
      blocks/request = ceil(8192/16)                        = 512
      max_concurrency = 24,990 / 512                        = 48.8x
      ⇒ startup log reads roughly:
        "GPU KV cache size: 399,840 tokens, Maximum concurrency
         for 8,192 tokens per request: 48.81x"

  (b) EXPECTED for chat traffic averaging 1,200 live tokens per sequence:
      blocks/request = ceil(1200/16)                        = 75
      cap            = 24,990 / 75                          = 333 sequences
      ⇒ but max_num_seqs on an H100 defaults to 1024 for the API server,
        so memory still binds here — at 333, not 48.
```

**The 7× gap between (a) and (b) is not an error in either.** It is the entire reason paged allocation exists: (a) is what you would be forced to provision if every sequence had to reserve its maximum up front, and (b) is what you get when allocation tracks actual length. Lesson 07.3 is the mechanism that turns (a) into (b). This lesson is about why the gap is that large.

### 5. The picture: KV growing, HBM filling, the ceiling falling out

```
  KV CACHE GROWTH AND THE BATCH CEILING
  Llama-3.1-8B, bf16 KV (128 KiB/token), 52.4 GB pool = 24,990 blocks of 16 tokens
  ══════════════════════════════════════════════════════════════════════════════

  ONE SEQUENCE OVER TIME  (blocks allocated on demand, 1 new block per 16 tokens)

   step:      prefill        decode ────────────────────────────────────────▶
   tokens:    0 ...... 2048  2049 .. 2064  2065 .. 2080  ...
   blocks:   [B][B][B]...[B]  [B]           [B]           ...
             └── 128 blocks ┘  ▲             ▲
                 (2048/16)     │             │
                               └─ +1 block every 16 generated tokens ─┘

   KV held:  256 MiB ─────────────▶ 258 MiB ──▶ 260 MiB ──▶ …  monotonic,
                                                                never shrinks
                                                                until FREE

  THE POOL, AS SEQUENCES ARRIVE  (each ▓ = 1,000 blocks ≈ 2 GB)

   pool: ████████████████████████████ 24,990 blocks free
         │
    +S1  ▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░  S1 holds 128 blocks and growing
    +S2  ▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░
    +S3  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░
     ⋮
   +S45  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░  kv_cache_usage_perc ≈ 0.92
   +S48  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  kv_cache_usage_perc ≈ 1.00
   +S49  ✗ no free blocks
         │
         ├─▶ if a running sequence can be evicted → PREEMPT one, admit S49
         └─▶ else S49 stays in the WAITING queue (num_requests_waiting += 1)

  THE CEILING, ALGEBRAICALLY

              num_gpu_blocks              24,990 blocks
    cap  =  ──────────────────────  =  ────────────────────
            blocks per sequence        ceil(ctx / 16)

    ctx =  2,048 → 128 blocks → cap = 195     ← short chat
    ctx =  8,192 → 512 blocks → cap =  48     ← long chat / small RAG
    ctx = 32,768 → 2048 blocks → cap =  12    ← document / agent trace
    ctx = 131,072 → 8192 blocks → cap =   3   ← full Llama-3.1 context

    ┌──────────────────────────────────────────────────────────────────────┐
    │ cap ∝ 1 / context.  Doubling the context you support HALVES the      │
    │ number of users you can serve on the same card — at the same         │
    │ request rate, with no change in traffic volume. Your unit of         │
    │ capacity is TOKENS OF KV, not requests and not GPUs.                 │
    └──────────────────────────────────────────────────────────────────────┘
```

That last box is the sentence to carry out of this lesson. It is also why a service that is comfortable at 2k context falls over the week a customer starts sending 32k prompts — traffic did not change, capacity did, by 16×.

### 6. Why a naive contiguous allocator throws most of it away

Now the efficiency half. Suppose you did the obvious thing — the thing every pre-2023 serving stack did, and the thing a hand-rolled HuggingFace `generate()` loop still does: **give each sequence one contiguous KV buffer, sized to the maximum it could possibly need, at admission time.**

You have to size it to the maximum, and this is the part worth being precise about, because it is not laziness. Three constraints force it:

1. **The final length is unknown.** The model decides when to stop. You cannot ask.
2. **A contiguous buffer cannot grow in place** if its neighbour is occupied — the same reason `realloc` sometimes copies. Copying a multi-gigabyte KV tensor mid-decode is not an option.
3. **The attention kernel wants a contiguous stride.** A naive kernel indexes `K[layer][head][position]` with a fixed stride; non-contiguity requires the kernel to gather through an indirection, which is a kernel change, not a config change.

So the allocator reserves `max_model_len` per admitted request. Three wastes compound, and each has a textbook OS analogue:

| Waste | Mechanism | OS analogue |
|---|---|---|
| **Internal fragmentation** | You reserved `max_model_len`; the request stopped early. The tail is dead for the request's whole lifetime. | A process reserving a huge stack it never uses. |
| **Reservation for unknown growth** | Even a request that *will* grow long holds its whole reservation from token 1, when it is using 2 % of it. | Pre-allocating a slab for a growable array. |
| **External fragmentation** | Variable-sized buffers admitted and freed in different orders leave holes between live buffers that are individually too small for a new reservation, even though their sum is large. | Classic `malloc` heap fragmentation. |

**Derive the internal-fragmentation number rather than quoting it.** Take a realistic chat workload with `max_model_len = 2048` and this output-length distribution (roughly the shape of ShareGPT-style traffic):

```
  output length   share of requests   reserved   used    wasted
  ──────────────  ─────────────────   ────────   ─────   ──────
     ≤   64            30 %             2048       64     1984
     ≤  256            35 %             2048      256     1792
     ≤  512            20 %             2048      512     1536
     ≤ 1024            10 %             2048     1024     1024
     ≤ 2048             5 %             2048     2048        0

  Expected USED per request
    = 0.30·64 + 0.35·256 + 0.20·512 + 0.10·1024 + 0.05·2048
    = 19.2 + 89.6 + 102.4 + 102.4 + 102.4
    = 416 tokens

  Utilisation of the reservation
    = 416 / 2048
    = 20.3 %          ⇒  79.7 % INTERNAL FRAGMENTATION ALONE
```

**That is the 60–80 % figure, reconstructed from first principles.** The PagedAttention paper (Kwon et al., SOSP '23) measured the same thing on real traces and reported that only about 20–40 % of KV memory in the systems it compared against held actual token state. You have just derived the bottom of that range from a length distribution and one reservation policy, which is a far more useful thing to own than the citation — change the distribution and you can predict where in the range a given workload lands.

And that is *before* the other two wastes. Add reservation-for-growth (a request that will eventually use its full 2,048 still holds all of it from step 1, so time-averaged utilisation is worse than the steady-state figure above) and external fragmentation (holes between freed buffers), and 20 % utilisation is optimistic, not pessimistic.

### 7. The two memory maps, side by side

```
  CONTIGUOUS ALLOCATION vs PAGED ALLOCATION — the same four requests
  ══════════════════════════════════════════════════════════════════════════════
  max_model_len = 2048.  Pool drawn as a linear address space.
  █ = KV bytes holding real token state    ░ = reserved but empty (waste)
  ▒ = unusable hole (external fragmentation)

  ── (A) CONTIGUOUS: one max-sized buffer per sequence ──────────────────────

   addr 0                                                              pool end
   ├────────────────────────────────────────────────────────────────────────┤
   │ S1 (used 190/2048)   │ S2 (used 1600/2048) │ S3 (used 64/2048)   │      │
   │██░░░░░░░░░░░░░░░░░░░░│████████████████░░░░░│█░░░░░░░░░░░░░░░░░░░░│▒▒▒▒▒▒│
   └──────────────────────┴─────────────────────┴─────────────────────┴──────┘
      ▲ 9 % useful            ▲ 78 % useful         ▲ 3 % useful        ▲ tail
                                                                       too small
                                                                       for a 4th
                                                                       reservation

   S4 arrives needing 2048 contiguous → REFUSED, even though the pool holds
   far more than 2048 tokens' worth of free bytes. They are just in the
   wrong shape.

   Pool utilisation: (190 + 1600 + 64) / (3 × 2048) = 30 %
   Admitted sequences: 3

  ── (B) PAGED: fixed 16-token blocks + a per-sequence block table ──────────

   PHYSICAL BLOCKS (all interchangeable, order irrelevant)
   ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
   │ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │ 8  │ 9  │ 10 │ 11 │ 12 │ …  │
   │ S2 │ S1 │ S2 │ S4 │ S1 │ S2 │ S3 │ S2 │ S4 │ S1 │free│ S2 │ S4 │free│
   └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘

   BLOCK TABLES (logical position → physical block; this is a page table)
     S1: [1, 4, 9, …]                  190 tokens → ceil(190/16) = 12 blocks
     S2: [0, 2, 5, 7, 11, …]          1600 tokens → 100 blocks
     S3: [6]                            64 tokens →   4 blocks
     S4: [3, 8, 12, …]                 admitted! needs only as many blocks
                                       as it currently holds, and grows one
                                       block at a time thereafter

   Waste per sequence = at most 15 tokens in the last partial block,
   regardless of context length. Four sequences → at most 60 tokens of
   internal waste. External fragmentation = ZERO: every free block is
   interchangeable, so there is no such thing as a hole of the wrong shape.

   Pool utilisation: > 96 %
   Admitted sequences: 4 (and counting, until blocks actually run out)

  ┌────────────────────────────────────────────────────────────────────────┐
  │ THE SWAP, IN ONE LINE                                                  │
  │ Contiguous:  waste ∝ (max_model_len − actual_length)  PER SEQUENCE.    │
  │ Paged:       waste ≤ (block_size − 1) tokens          PER SEQUENCE.    │
  │ The first scales with your context-window setting. The second is 15.   │
  └────────────────────────────────────────────────────────────────────────┘
```

The price of (B) is that the attention kernel must dereference the block table on every step instead of walking a fixed stride. That is a real cost — a gather instead of a linear read — but it is small, well-amortised, and paid in exchange for roughly tripling the sequences you can hold. That trade is what 07.3 unpacks.

### 8. Levers on the cap, and which ones are yours

The cap is `KV_pool_bytes ÷ (kv_bytes_per_token × context)`. Every term is a lever, and they are independent, so they multiply.

| Lever | Which term it moves | Typical factor | Where the decision lives |
|---|---|---|---|
| **Quantise the KV cache** (`--kv-cache-dtype fp8`) | `e_kv`: 2 → 1 | **2×** | Serving engine. Hopper+ for FP8. Validate accuracy. |
| **Right-size `--max-model-len`** | worst-case `context` | 2–8× vs the model's architectural max | You. Set it to your measured p99, not the model card. |
| **Quantise the weights** (FP8/INT4) | `KV_pool_bytes` (bigger residual) | 8B: 52 → 60 GB pool (1.15×). 70B on H200: 2 → 67 GB (**33×**) | Serving engine + quality validation. |
| **GQA / MQA model** | `n_kv` | 4–8× (GQA), up to 64× (MQA) | Baked into the checkpoint. Your input is *before* the model is chosen. |
| **Sliding-window / hybrid attention** | effective `L` for windowed layers | Workload-dependent; large at long context | Model architecture. |
| **Cross-layer KV sharing** | effective `L` | 2–4× | Model architecture. |
| **Prefix caching** (`--enable-prefix-caching`, default **on**) | *effective* per-request bytes, by deduplicating shared prefixes | Workload-dependent; large for shared system prompts | Serving engine. Free — on by default in V1. |
| **PagedAttention** | reclaims fragmentation waste | ~3× vs contiguous, from §6's arithmetic | Serving engine. Not a knob; it is the design. |
| **Bigger-HBM SKU** | `KV_pool_bytes` | Non-linear — see below | Procurement. |
| **Raise `--gpu-memory-utilization`** | `KV_pool_bytes` | 1.05–1.10× at best | You. **The weakest lever with the worst downside.** |
| **Tensor parallelism** | `KV_pool_bytes` per group | ~N×, minus per-step collective latency | You. Also the only fix when weights do not fit. |

Two notes on that table that are worth more than the table.

**The SKU row is non-linear and everyone gets it wrong.** Because weights are a *fixed subtraction*, headroom grows much faster than capacity. A 70B FP8 model on an H100-80GB leaves roughly 2 GB for KV after weights and overhead; on an H200-141GB it leaves roughly 67 GB. That is 76 % more HBM producing a **~33× larger KV pool** and therefore a ~33× larger batch. Never reason about SKU upgrades with a capacity ratio; always recompute the residual.

**The utilisation row is the trap.** On an 80 GB card, going 0.90 → 0.95 buys 4 GB against a ~52 GB pool: about 8 % more concurrency, in exchange for the headroom that absorbs allocator fragmentation, CUDA-graph capture and a burst of long prefills. Small upside, catastrophic downside. If someone reaches for it to fix a preemption problem, they are attacking the supply side of a demand-side problem (07.4 covers this properly).

### 9. Three ceilings that all look like "concurrency" — and the metric test

You are queueing under load. Which ceiling did you hit? There are three, they produce similar-looking symptoms, and they have completely different fixes.

| Ceiling | What it is | Metric signature | Fix |
|---|---|---|---|
| **KV pool** | Not enough free blocks to admit or grow | `kv_cache_usage_perc` ≈ 1.0, `num_preemptions_total` rising | Shorter `max-model-len`, FP8 KV, more/bigger GPUs, lower `max-num-seqs` |
| **`max_num_seqs`** | Scheduler refuses to put more sequences in one pass | `kv_cache_usage_perc` **low**, `num_requests_running` pinned exactly at the flag value | Raise `max-num-seqs` |
| **`max_num_batched_tokens`** | Per-iteration token budget exhausted (usually by prefill) | `iteration_tokens_total` histogram piled at the budget, TTFT rising, ITL flat | Raise the budget (hurts ITL) or shed prefill load |

The runbook — **the three-signal test** — is what you run first when paged about rising latency on an inference fleet, before anyone says "add GPUs." You are genuinely KV-bound when *all three* hold at once:

1. `vllm:kv_cache_usage_perc` pinned at ≈ 1.0 — blocks exhausted;
2. `vllm:num_requests_running` flat at a ceiling **below** `max_num_seqs` — memory, not the flag, is binding;
3. `vllm:num_requests_waiting` climbing — arrivals cannot be admitted.

Against verified V1 metric names (module 05 lesson 06), as PromQL you can paste:

```promql
# 1. KV pressure. Pinned near 1.0 under load ⇒ the pool is the ceiling.
avg_over_time(vllm:kv_cache_usage_perc[5m])

# 2. Live concurrency vs the configured flag. If running << max_num_seqs
#    while (1) is pinned, memory binds. If running == max_num_seqs while
#    (1) is low, the FLAG binds and more memory changes nothing.
vllm:num_requests_running

# 3. Queue depth. Rising while (2) is flat is the admission-blocked signature.
vllm:num_requests_waiting

# 3b. And WHY they are waiting — 'capacity' means genuine scheduling/KV
#     pressure; 'deferred' means a transient constraint (LoRA budget, KV
#     transfer, blocked status). These have different fixes.
vllm:num_requests_waiting_by_reason

# 4. The smoking gun: eviction is happening, so admitted work is being
#    thrown away and recomputed.
rate(vllm:num_preemptions_total[5m])

# 5. Compact "am I KV-bound" expression: 1 when all three signals agree.
(avg_over_time(vllm:kv_cache_usage_perc[5m]) > 0.95)
  and (vllm:num_requests_waiting > 5)
  and (rate(vllm:num_preemptions_total[5m]) > 0)
```

**The false positive to guard against:** `num_requests_waiting` climbing while `kv_cache_usage_perc` sits at 0.3. That is not a memory cap. Check `max_num_seqs` (is `num_requests_running` sitting exactly on it?), then the per-iteration token budget, then your load generator — a client-side connection pool cap produces exactly this shape and no amount of GPU memory moves it.

Two current-vLLM details that change how you read these signals, both verified in `vllm/config/scheduler.py`:

- **`scheduler_reserve_full_isl` defaults to `True`.** The scheduler checks that the *full input sequence length* fits in the KV cache before admitting a request, rather than only checking the first prefill chunk. This deliberately trades admission aggressiveness for stability — it prevents over-admission and the KV thrashing that chunked prefill would otherwise cause. Operationally: you will see requests wait with the pool at, say, 0.85 rather than see them admitted and immediately preempted. That is the scheduler being conservative on purpose, not a bug.
- **`watermark` defaults to `0.0`** (disabled). When set to a positive fraction, the scheduler keeps that fraction of blocks free when admitting waiting or preempted requests. It is the explicit dial for "stop admitting before the pool is bone dry," and it exists because repeated preemption of the same request is far worse than making it wait once.

### 10. Two mechanisms that stretch the pool (preview)

Because KV lives in independently-allocatable, content-addressable blocks, two things become possible that a contiguous allocator cannot express at all. Both are covered mechanically in 07.3; hold their shape here, because they change the arithmetic above.

**Prefix caching** (`--enable-prefix-caching`, and in current vLLM `enable_prefix_caching` defaults to **`True`**). Identical prompt prefixes hash to the same block content, so many requests point at the *same* physical blocks, reference-counted. A 2,000-token system prompt shared by 500 concurrent requests costs KV **once**, not 500 times. For high-fan-out workloads — a shared system prompt, few-shot exemplars, a RAG context reused across a session — this is frequently the single largest available win on cost per token, and it is on by default. Observe it with `vllm:prefix_cache_hits` ÷ `vllm:prefix_cache_queries`, both counted in **tokens**, not requests.

**KV-cache quantisation** (`--kv-cache-dtype fp8`) halves `e_kv` and therefore doubles the token capacity of the same pool. It composes with everything else in §8 multiplicatively.

Both stretch the pool without buying hardware. Both cost something you must measure — cache-hit rate is workload-dependent, and FP8 KV has an accuracy delta. Neither is a substitute for getting `--max-model-len` right.

## Perspectives

**The allocator / OS-analogy view.** Everything in §6 is textbook operating-systems memory management with the serial numbers filed off: internal fragmentation, external fragmentation, and over-reservation for unknown future growth are the same three failure modes a first-year OS course covers for heap allocators and process address spaces. If you have debugged a fragmented Java heap, a `jemalloc` arena, or Kubernetes node bin-packing, you already carry the right mental model — the only new fact is that the heap is HBM and the objects are per-token K/V tensors that grow one element at a time and are freed all at once. That transferability is the point: the durable, hireable knowledge here is the allocator reasoning, not the vLLM flag names.

**The architecture-as-cost-control view.** Character.AI is explicit that KV-cache-per-token is a target they optimise *at the model architecture level* — multi-query attention, hybrid attention horizons that bound how far back most layers attend, and cross-layer KV sharing — not merely a serving knob. The most senior version of this lesson's skill is not "I can compute the concurrency cap," it is "I can influence the architecture upstream so the cap starts higher." That is a materially different level of leverage than tuning `--gpu-memory-utilization` after the checkpoint is frozen, and it is the difference between a 1.1× win and a 20× one.

**The observability / SRE view.** The three-signal test in §9 is written as an incident runbook deliberately. "Requests are queuing" is an ambiguous alert: it is consistent with a KV cap, a `max_num_seqs` cap, a token-budget cap, or a client-side connection-pool limit, and only one of those is fixed by adding GPUs. Checking `kv_cache_usage_perc` *before* reaching for scale-out is the difference between a two-minute diagnosis and an hour of adding capacity that does not move the metric. Note the shape of the diagnostic: it is a **conjunction** of three signals, not any one of them, because each individually has a common benign explanation.

**The long-context economics view.** As agent workloads push contexts toward 100k+ tokens — long conversation histories, retrieved documents, multi-step tool-call traces — the inverse relationship in §5 stops being napkin trivia and becomes the dominant cost driver. A pool that comfortably serves 195 concurrent 2k-token requests serves about 3 concurrent 128k-token ones on identical hardware, a 65× swing driven entirely by a product decision about context. This is exactly the pressure that KV offload and prefill/decode disaggregation (07.6) exist to relieve, either by moving KV to a cheaper memory tier or by scaling the two phases independently so a long-context prefill does not starve the decode pool.

**The FinOps view.** Every lever in §8 is a multiplier on the same denominator, and they compose. FP8 KV (2×) on a right-sized `max-model-len` (say 4×, from a 32k default down to a measured 8k p99) on a GQA-8 rather than MHA-64 model (8×) is a 64× swing in concurrent seats before a single serving optimisation runs. That is the compounding that turns a "33× cost reduction over two years" headline from an exotic achievement into a list of six sensible decisions taken in order.

## Real-world use cases

- **Character.AI — "Optimizing AI Inference at Character.AI."** They state directly that KV cache size "determines the maximum batch size that can fit on a GPU," and describe caching attention KV on host memory between chat turns to avoid recomputing conversation history. The architecture changes — multi-query attention (`n_kv = 1`), hybrid attention horizons (roughly one global-attention layer per six, the rest sliding-window), and cross-layer KV sharing — cut KV cache size **more than 20×** with no reported quality regression, and enable an inter-turn cache with a reported >95 % hit rate. Result: ~20,000 QPS at under a cent per hour of conversation, ≥33× serving-cost reduction since 2022. **What it shows:** this lesson's formula with every term attacked at once, by a team for whom the cap is the business. It is also the cleanest existence proof that `n_kv` and effective `L` are *design variables*, not constants.

- **PagedAttention / vLLM (Kwon et al., SOSP '23).** The paper's motivating measurement is that in the systems it compared against — which reserved contiguous, maximum-length KV buffers per request — only roughly 20–40 % of KV memory held actual token state, with the rest lost to internal fragmentation, over-reservation for unknown output length, and external fragmentation. Its fix bounds internal waste at under one block per sequence and eliminates external fragmentation entirely, reporting 2–4× throughput improvements over FasterTransformer and Orca at matched latency. **What it shows:** the gap between "memory I bought" and "memory doing work" was the dominant inefficiency in LLM serving, it was an *allocator design* problem rather than a hardware one, and the fix was borrowed wholesale from 1960s operating systems. §6 reconstructs the bottom of that utilisation range from a length distribution so you can predict it for your own traffic rather than quoting theirs.

- **vLLM engine source — `vllm/v1/core/block_pool.py` and `vllm/v1/core/kv_cache_utils.py`.** The block pool pre-allocates every `KVCacheBlock` object at init and threads a doubly linked free list through the block objects themselves, explicitly "to avoid Python object creation overheads" and to get O(1) removal from the middle of the queue. `ref_cnt` on each block is what makes prefix sharing possible; the free queue is ordered LRU-first, and blocks are returned in *reverse* order when a request frees so that the tail blocks (which hash more tokens and are least likely to be reused) are evicted first. **What it shows:** the concurrency cap is enforced by a specific, readable data structure you can go and look at, and the engineering that makes prefix caching cheap enough to default on is visible in about 200 lines.

- **vLLM scheduler config — `scheduler_reserve_full_isl` and `watermark`.** Current vLLM admits a request only if its *full* input length fits in the pool (`scheduler_reserve_full_isl = True` by default), rather than admitting on the first chunk and hoping; and it offers a `watermark` fraction of blocks to hold in reserve when admitting. **What it shows:** production engines converge on *conservative admission* because the cost asymmetry is severe — a request that waits 200 ms costs 200 ms, while a request admitted and then preempted costs a full prefill recomputation plus the queue wait. Read a "waiting at 85 % pool" observation as the scheduler doing this deliberately.

## Worked example

**Measure and predict the concurrency cap for Llama-3.1-8B on one H100-80GB, and prove the `1/context` relationship.**

### Part 1 — predict, from geometry alone

```
  Model geometry (config.json): L=32, n_q=32, n_kv=8, d=128, bf16
  Hardware: 1 × H100-80GB SXM
  Flags:    --gpu-memory-utilization 0.90 --max-model-len 8192

  weights                   = 8.03e9 × 2 B                = 16.1 GB
  usable                    = 80 × 0.90                   = 72.0 GB
  non-KV working (measured by vLLM's profiling pass)      ≈  3.5 GB
  KV pool                                                 ≈ 52.4 GB

  kv_bytes_per_token        = 2 × 32 × 8 × 128 × 2        = 131,072 B
  bytes/block (16 tokens)                                 = 2.097 MB
  num_gpu_blocks            = 52.4e9 ÷ 2.097e6            ≈ 24,990
  total KV token capacity   = 24,990 × 16                 ≈ 399,840 tokens

  PREDICTION (a): startup log "Maximum concurrency for 8,192 tokens
                  per request" ≈ 399,840 / 8,192          ≈ 48.8x

  PREDICTION (b): at --max-model-len 32768, the same pool gives
                  399,840 / 32,768                        ≈ 12.2x
                  ⇒ 4× the context, 4× fewer seats. EXACTLY inverse.
```

### Part 2 — what the server actually says

```bash
pip install "vllm==0.11.0"     # V1 engine; pin it, and record the version

vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 256 \
  --port 8000
```

The line to grep for (representative — your exact numbers will differ with driver, dtype and vLLM version):

```
INFO ... Chunked prefill is enabled with max_num_batched_tokens=8192.
INFO ... Memory profiling takes 6.42 seconds.
INFO ... the current vLLM instance can use total_gpu_memory (79.11GiB)
         x gpu_memory_utilization (0.90) = 71.20GiB
INFO ... model weights take 14.99GiB; non_torch_memory takes 0.32GiB;
         PyTorch activation peak memory takes 1.21GiB;
         the rest of the memory reserved for KV Cache is 54.68GiB.
INFO ... GPU KV cache size: 447,776 tokens, Maximum concurrency for 8,192
         tokens per request: 54.66x
```

**Read it line by line — this is the acceptance test for Part 1.**

- `79.11GiB`, not 80 — the card reports slightly less than its marketing capacity, and `total_gpu_memory` is what remains after the driver's own reservation.
- The breakdown is exactly this lesson's three terms: weights `14.99 GiB` (note: **GiB**, so `16.1 GB = 15.0 GiB` — the prediction was right, the units were different), non-Torch `0.32 GiB`, activation peak `1.21 GiB`. Working memory came in at ~1.5 GiB, well under the 3.5 GB assumed, so the real pool is bigger than predicted.
- `447,776 tokens` vs the predicted 399,840 — 12 % high, entirely explained by the smaller-than-assumed working-memory term. **The prediction was structurally right and quantitatively within 12 %, and the gap is attributable to one named term.** That is what a good napkin looks like.
- `54.66x` = `447,776 ÷ 8,192`. Confirms the identity.

### Part 3 — prove the inverse relationship by measurement

Restart with `--max-model-len 32768`, nothing else changed:

```
INFO ... GPU KV cache size: 447,776 tokens, Maximum concurrency for 32,768
         tokens per request: 13.66x
```

`54.66 / 13.66 = 4.00`. **The token capacity did not move — only how you chose to divide it.** That is the cleanest possible demonstration that `--max-model-len` is a *reservation ceiling*, not an allocation: raising it did not consume memory, it consumed permission to run concurrently.

### Part 4 — saturate and read the real cap

Now drive it and find where the running batch actually stops:

```bash
vllm bench serve \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --base-url http://localhost:8000 \
  --dataset-name random --random-input-len 4096 --random-output-len 512 \
  --request-rate inf --max-concurrency 300 --num-prompts 1500 \
  --percentile-metrics ttft,tpot,itl --metric-percentiles 50,99 \
  --save-result --result-filename cap_probe.json
```

While it runs, sample the three signals every couple of seconds:

```bash
while true; do
  curl -s localhost:8000/metrics | grep -E \
    '^vllm:(kv_cache_usage_perc|num_requests_running|num_requests_waiting|num_preemptions_total)' \
    | tr '\n' ' '; echo
  sleep 2
done
```

A representative saturated sample:

```
vllm:kv_cache_usage_perc{...}   0.98
vllm:num_requests_running{...}  97
vllm:num_requests_waiting{...}  203
vllm:num_preemptions_total{...} 41
```

**Read the diagnosis, not just the numbers.** Live context averages roughly `4096 + 512/2 = 4,352` tokens, so the predicted cap is `447,776 / 4,352 ≈ 103` — and measured running is 97, about 6 % low, which is block rounding plus the conservative full-input-length admission check. `num_requests_running` is 97 against `--max-num-seqs 256`, so **the flag is not binding; memory is.** Waiting is climbing and preemptions are non-zero: all three signals agree. This is a genuine, measured KV-bound operating point, and it is the number your cost-per-token calculation should be computed at.

Contrast: had you seen `kv_cache_usage_perc 0.31`, `num_requests_running 256`, `num_requests_waiting 203`, the diagnosis flips entirely — the pool is two-thirds empty and `max_num_seqs` is pinned at exactly its configured value. More GPU memory buys nothing; raising the flag buys everything.

## Practice

Hands-on, rented GPU, feeds the deliverable. One H100-80GB or A100-80GB from any neocloud; roughly 30–40 minutes of GPU time. **Pin the vLLM version** (`vllm==0.11.0` or later V1) and record it — every number below is version-sensitive.

### 1. Predict before you measure

Using §4's arithmetic and the geometry from `config.json`, predict for Llama-3.1-8B on your card: `KV_pool` bytes, `num_gpu_blocks`, total token capacity, and the "Maximum concurrency" figure at `--max-model-len` of 4096, 8192 and 16384.

**Acceptance:** three predicted concurrency figures, written down *before* the server starts.

### 2. Read the startup profile

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 8192 --gpu-memory-utilization 0.90 \
  --max-num-seqs 256 --port 8000 2>&1 | tee serve_8k.log

grep -E 'KV cache|memory profiling|model weights|Maximum concurrency' serve_8k.log
vllm --version   # record it
```

Repeat for `--max-model-len 4096` and `16384`.

**Acceptance:** a table with predicted vs actual "Maximum concurrency" for all three, and **the token-capacity column showing the same value in all three rows**. That invariance is the single most instructive observation in this lesson — `--max-model-len` divides the pool, it does not size it.

### 3. Measure the real cap under saturation

For the 8k configuration, run the `vllm bench serve` sweep from the worked example at `--max-concurrency 300`, sampling `/metrics` throughout. Record the peak `num_requests_running` at the moment `kv_cache_usage_perc` first pins above 0.95.

**Acceptance:** the peak running count, the three raw gauge values at that instant, and a one-line statement of which of the three ceilings from §9 was binding, with the evidence.

### 4. Move the cap two ways, and confirm they compose

Re-run the saturation probe under each of these, one at a time:

- `--max-model-len 4096` (halve the reservation)
- `--kv-cache-dtype fp8` at `--max-model-len 8192` (halve the bytes)
- both together

**Acceptance:** a four-row table (baseline + three) of measured peak `num_requests_running` and measured `output_throughput` from the bench JSON, with the observed ratios next to the predicted 2×/2×/4×. Note where reality falls short of prediction and give the reason (block rounding, admission conservatism, the working-memory term shifting when the cache dtype changes).

### 5. Reproduce the fragmentation argument on paper

Take your actual measured output-length distribution from the bench run (`vllm:request_generation_tokens` histogram, or the per-request lengths in the result JSON) and compute what fraction of a contiguous `max_model_len`-sized reservation would have been used. Compare to §6's 20 %.

**Acceptance:** one number and one sentence — "with my traffic, a contiguous allocator would have held X % useful KV, so PagedAttention is worth roughly 1/X × concurrency here." This is the number that makes 07.3 land.

**Overall acceptance:** predicted-vs-measured concurrency at three context lengths; the token-capacity invariance observation; a measured KV-bound operating point with all three signals; the four-row lever table; and the fragmentation estimate for your own traffic — committed to the [cost-per-token deliverable](../practice/cost-per-token/README.md). The measured cap is the load-bearing input to the cost calculation in 07.5, because tokens/sec/GPU is roughly proportional to it.

## Common pitfalls

- **Setting `--max-model-len` to the model's architectural maximum "to be safe."** A model advertising 128k context does not mean your traffic uses it. The setting is a *worst-case reservation ceiling* that divides your printed concurrency directly, so setting it to 128k when your p99 is 8k costs you 16× the concurrency for capability nobody uses. *Mechanism:* the printed concurrency is `num_blocks ÷ ceil(max_model_len/block_size)`; the numerator does not move. Set it from your measured `vllm:request_prompt_tokens` + `vllm:request_generation_tokens` p99, and revisit when traffic shape drifts.

- **Using `num_attention_heads` where the formula wants `num_key_value_heads`.** The single most common arithmetic bug in this material. On a GQA-8 model it inflates your estimate 4–8×, which cascades into a wrong cap, a wrong throughput prediction, and a wrong cost per token. *Mechanism:* only KV heads have K and V projections to cache; query heads share them. Read `num_key_value_heads` from `config.json`, every time.

- **Reading `num_requests_waiting` climbing as "add GPUs."** Check `kv_cache_usage_perc` first. Low pool usage with a growing queue means you are not KV-bound — it is `max_num_seqs`, the per-iteration token budget, or a client-side concurrency limit, none of which more HBM fixes. *Mechanism:* the three ceilings in §9 all manifest as queueing; only the conjunction of signals distinguishes them.

- **Quoting the startup "Maximum concurrency" as your real concurrency.** It assumes every sequence runs to the full `--max-model-len`. Real traffic averages far below that, and paged allocation tracks actual length, so measured concurrency is routinely 3–8× the printed figure. Use the printed number as a floor and a sanity check on your arithmetic, and the measured `num_requests_running` at saturation as the operating figure.

- **Treating `--block-size` as an early tuning knob.** It moves the cap by fractions of a percent (bounded internal waste of ≤15 tokens per sequence at the default 16) while `--max-model-len`, `--kv-cache-dtype` and the model's `n_kv` move it by 2–8× each. Leave it at 16 unless you have a specific long-context or prefix-granularity reason, and re-measure when you change it.

- **Assuming preemption means you were out of memory *right then*.** With conservative admission (`scheduler_reserve_full_isl`) and no watermark, the engine can preempt because a *running* sequence needed one more block, not because a new one arrived. That distinction changes the fix: it is the growth of resident sequences, not the arrival rate, that is oversubscribed — so lowering `max-num-seqs` helps and rate-limiting admissions may not.

- **Comparing GB and GiB.** vLLM's logs report GiB; parameter arithmetic naturally produces GB. `16.1 GB = 15.0 GiB` — a 7 % difference that will make a correct prediction look wrong. Carry the units.

## Self-check

**(a) Compute per-token KV bytes for a 70B model, stating every assumption.**

**Answer:** Assume Llama-3.3-70B geometry — `num_hidden_layers = 80`, `num_key_value_heads = 8` (GQA; 64 query heads), `head_dim = 128` — and a bf16 KV cache (`e_kv = 2`). Then `kv_bytes_per_token = 2 × 80 × 8 × 128 × 2 = 327,680 B = 320 KiB`. The leading `2` is one K vector and one V vector; the head count that matters is the *KV* head count, not the query head count, which is the GQA saving — MHA at 64 KV heads would give `2 × 80 × 64 × 128 × 2 = 2.62 MB/token`, 8× worse. At `--kv-cache-dtype fp8` the figure halves to 160 KiB. A single 8k-token request therefore holds `8192 × 320 KiB = 2.5 GiB` of HBM at bf16 for its entire lifetime.

**(b) You double `--max-model-len` at fixed `--gpu-memory-utilization`. What happens to the printed maximum concurrency, and what happens to actual memory consumption?**

**Answer:** Printed concurrency **halves**; actual memory consumption **does not change at all**. The pool is `HBM × utilisation − weights − working memory`, none of which depends on `max_model_len`, so `num_gpu_blocks` and total token capacity are identical before and after. The printed figure is `num_gpu_blocks ÷ ceil(max_model_len / block_size)` — the numerator is fixed and you doubled the denominator. Under paged allocation a sequence occupies blocks proportional to its *current* length, so raising the ceiling does not make any individual request cost more; it only lowers the worst case the engine plans for and reports. The practical consequence is that `--max-model-len` costs you *admission headroom*, which is why setting it to a model's architectural maximum is expensive despite allocating nothing.

**(c) Why does contiguous KV allocation waste most of the memory, and what bounds the waste under paging?**

**Answer:** A contiguous allocator must reserve `max_model_len` per sequence at admission, for three compounding reasons: output length is unknown at admission, a contiguous buffer cannot grow in place when its neighbour is occupied, and a naive attention kernel wants a fixed stride. Three wastes follow — internal fragmentation (the request stops early and the reserved tail is dead for its whole lifetime), over-reservation for growth (even a long request holds its full reservation from token 1), and external fragmentation (variable-sized buffers freed in different orders leave holes individually too small to admit anything). Derived from a realistic chat length distribution with `max_model_len = 2048`, expected used length is about 416 tokens, i.e. 20 % utilisation and ~80 % internal waste before the other two effects — matching the PagedAttention paper's measured 20–40 % utilisation range for such systems. Under paging, waste is bounded by the last partial block per sequence: at most `block_size − 1 = 15` tokens, **independent of context length**, and external fragmentation is zero because every free block is interchangeable. That is the difference between waste that scales with your context setting and waste that is a constant 15 tokens.

**(d) `num_requests_waiting` is climbing on your dashboard. How do you determine, without guessing, whether the fix is "add GPUs"?**

**Answer:** Run the three-signal test as a conjunction, not a disjunction. (1) `vllm:kv_cache_usage_perc` — pinned near 1.0 means blocks are exhausted. (2) `vllm:num_requests_running` — flat at a ceiling *below* your configured `max_num_seqs` means memory rather than the flag is binding; sitting exactly *on* `max_num_seqs` means the flag binds and more memory buys nothing. (3) `rate(vllm:num_preemptions_total[5m])` — non-zero confirms the engine is evicting admitted work, each eviction costing that request a full prefill recomputation. Also check `vllm:num_requests_waiting_by_reason`: `capacity` is genuine scheduling/KV pressure, `deferred` is a transient constraint (LoRA budget, KV transfer, blocked status) with a different fix. Only if (1), (2)-below-flag and (3) all hold are you genuinely KV-bound, and only then is more KV pool — bigger card, more GPUs, FP8 KV, shorter `max-model-len` — the right lever.

**(e) A team wants to switch to a model with 4× more KV heads because it scores slightly higher on a quality benchmark. What do you ask before signing off?**

**Answer:** Ask what the change does to `kv_bytes_per_token`, and therefore to the concurrency cap and cost per million tokens **at your real production context length**. A 4× increase in `num_key_value_heads` is a 4× increase in per-token KV, a ~4× reduction in concurrent seats on the same pool, and — because decode throughput is roughly linear in resident batch well below the ridge point — roughly a 4× increase in cost per token. Then ask two follow-ups: whether the quality delta survives on *your* eval rather than the public benchmark, and whether the same architecture is available as a GQA variant (the GQA paper showed MHA checkpoints can be uptrained to GQA for about 5 % of original pre-training compute). Frame it as a trade with a number attached — "this is a 4× serving-cost increase for N points on benchmark X" — rather than an objection, because sometimes 4× is worth it and the point is that someone decided rather than drifted.

**(f) Your pool is at 85 % and requests are waiting. Is that a bug?**

**Answer:** Probably not. Current vLLM defaults `scheduler_reserve_full_isl = True`, meaning a request is admitted only if its **full** input sequence length fits in the KV cache, not merely its first prefill chunk. That deliberately declines to admit work it might immediately have to preempt, because the cost asymmetry is severe: waiting costs the request its queue time, while admit-then-preempt costs a full prefill recomputation *plus* the queue time. There is also an optional `watermark` (default `0.0`, disabled) that reserves a fraction of blocks free during admission for the same reason. So "waiting at 85 %" is the scheduler being conservative on purpose. What *would* be a problem is waiting at 85 % **with preemptions also climbing**, which means the conservatism is not preventing eviction and you are genuinely oversubscribed on resident-sequence growth.

## Connections & what's next

This lesson turns 07.1's abstract residual into the concrete `max_concurrent_requests` figure that every later lesson in this module measures, tunes, or multiplies. The fragmentation arithmetic it derives is the exact gap 07.3's PagedAttention closes, and the three-signal runbook is the diagnostic you will reuse when 07.4 covers production tuning and preemption, and again when 07.8 picks an autoscaling signal — because `kv_cache_usage_perc` and `num_requests_waiting` are the two signals that actually track inference saturation, unlike GPU utilisation. The long-context/concurrency tension raised in Perspectives resurfaces directly in 07.6's disaggregation lesson.

**Next: [07.3 — PagedAttention and vLLM](03-pagedattention-and-vllm.md)** takes the ~80 % waste this lesson derives and shows the block-table mechanism — non-contiguous fixed-size KV blocks addressed exactly like OS page tables — that cuts it to under 4 %, plus the copy-on-write and hash-based prefix sharing that a contiguous allocator cannot express at all. That reclaimed memory is what physically permits the high concurrency this lesson's cap formula assumes is achievable.

## References & further reading

**Primary sources**

1. **Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention," SOSP '23** — https://arxiv.org/abs/2309.06180 — §2 is the fragmentation analysis this lesson's §6 reconstructs from first principles: contiguous, maximum-length per-request reservation leaving only ~20–40 % of KV memory holding real token state, and 2–4× throughput gains over FasterTransformer and Orca at matched latency once that waste is reclaimed. *(arxiv.org is blocked by this environment's egress proxy, so the paper's exact figures could not be re-read directly; the mechanism and the utilisation range are cross-checked against the vLLM source tree and the engine's own design documents, and §6 derives the bottom of the range independently so the lesson does not depend on the citation.)*
2. **vLLM engine source — `vllm/v1/kv_cache_interface.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/kv_cache_interface.py — `AttentionSpec.unpadded_page_size_bytes = num_kv_heads × block_size × (head_size + head_size_v) × dtype_size`, and `FullAttentionSpec.max_memory_usage_bytes = ceil(max_model_len / block_size) × page_size_bytes`. §2's formula and §4's worst-case cap fall straight out of these two lines.
3. **vLLM engine source — `vllm/v1/core/kv_cache_utils.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/kv_cache_utils.py — `get_max_concurrency_for_kv_cache_config` (`num_blocks ÷ Σ_groups ceil(per-request bytes ÷ page bytes)`), `update_kv_cache_capacity` (which prints the startup line), and `FreeKVCacheBlockQueue` (the O(1) LRU free list with its documented reverse-order free semantics).
4. **vLLM engine source — `vllm/v1/core/block_pool.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/block_pool.py — the block pool, `ref_cnt` per block, `cached_block_hash_to_block`, and `touch`/`free_blocks`. This is the data structure the cap is enforced by.
5. **vLLM engine source — `vllm/config/cache.py` and `vllm/config/scheduler.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/cache.py — `DEFAULT_BLOCK_SIZE = 16`, `enable_prefix_caching = True`, `gpu_memory_utilization = 0.92` on main (0.90 in the 0.11.x line), the `CacheDType` enum; and in the scheduler config, `scheduler_reserve_full_isl = True` and `watermark = 0.0`. **Correction to earlier versions of this lesson:** prefix caching is *on by default* in current vLLM, not opt-in, and the conservative full-input-length admission check is why you will see requests wait before the pool is full.
6. **vLLM — `docs/design/prefix_caching.md`** — https://github.com/vllm-project/vllm/blob/main/docs/design/prefix_caching.md — the hash-chaining scheme (`hash(parent_hash, block_token_ids, extra_keys)`), the "we only cache full blocks" rule, LRU eviction from the head of the free queue, and the design rationale for pre-allocating all block objects. Read before 07.3.
7. **vLLM engine source — `vllm/v1/metrics/loggers.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/metrics/loggers.py — the exact names, types and label sets used in §9: `vllm:kv_cache_usage_perc`, `vllm:num_requests_running`, `vllm:num_requests_waiting`, `vllm:num_requests_waiting_by_reason` (labels `capacity` / `deferred`), `vllm:num_preemptions_total`, `vllm:prefix_cache_queries` / `vllm:prefix_cache_hits` (counted in **tokens**).
8. **Ainslie et al., "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints" (EMNLP 2023)** — https://arxiv.org/abs/2305.13245 — the derivation behind the `num_kv_heads` saving, and the uptraining result (≈5 % of original pre-training compute to convert MHA → GQA) that makes the §8 pushback actionable rather than rhetorical. *(Also behind the blocked arxiv domain; the architectural claim is directly verifiable from any post-2023 `config.json`.)*

**Real-world engineering**

9. **Character.AI — "Optimizing AI Inference at Character.AI"** — https://blog.character.ai/optimizing-ai-inference-at-character-ai/ — KV cache size as the batch-size-determining resource, cut >20× via multi-query attention, hybrid attention horizons and cross-layer KV sharing without quality loss; inter-turn host-memory KV caching with a reported >95 % hit rate; ~20,000 QPS at under a cent per hour of conversation. The load-bearing case study.
10. **Character.AI — "Optimizing AI Inference at Character.AI (Part Deux)"** — https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/ — the follow-up, covering int8 training-and-serving and further caching detail; the source of the ≥33× cumulative cost-reduction figure.
11. **NetApp Community — "Engineering Inference: KV Cache, Shared Storage, and the Economics of AI"** — https://community.netapp.com/t5/Tech-ONTAP-Blogs/Engineering-Inference-KV-Cache-Shared-Storage-and-the-Economics-of-AI/ba-p/466018 — **vendor content**, read with that lens rather than as an operator postmortem, but a usable secondary reference for the "KV cache as an economic object" framing and the reuse-vs-recompute and tiered-KV-storage vocabulary now common in AI infrastructure.

**Deeper dives**

12. **Module 03 lesson 04 — "Decode throughput: bandwidth ceilings, batching, and the prefill/decode split"** — [../../03-gpu-hardware/lessons/04-decode-bandwidth-batching.md](../../03-gpu-hardware/lessons/04-decode-bandwidth-batching.md) — the two-term decode model, and the capacity-wall-vs-compute-wall diagnostic that tells you whether a throughput plateau is a KV-capacity problem (this lesson) or a compute problem (not fixable with memory).
13. **Sebastian Raschka — "LLM Architecture Gallery"** — https://sebastianraschka.com/llm-architecture-gallery/ — compact fact sheets on attention variants (MHA, GQA, MQA, MLA, sliding-window) across recent open models; useful for checking a new model's KV geometry quickly before you commit to serving it.
