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
sources: 15
---
# 07.3 · PagedAttention and vLLM

> **Concept.** PagedAttention treats the KV cache like OS virtual memory — fixed-size non-contiguous blocks addressed through a per-sequence block table — which kills fragmentation, enables prefix sharing via copy-on-write, and is what lets vLLM's continuous-batching scheduler pack many sequences onto one GPU.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

07.2 established the concurrency cap as arithmetic — `KV_pool_bytes ÷ per-request KV footprint` — and then showed that a naive contiguous allocator strands most of that pool before you get to use it. You derived the waste yourself: with a realistic chat length distribution and a `max_model_len` reservation policy, only about 20 % of the reserved KV holds real token state. That lesson stopped at diagnosis.

This lesson is the fix, and it is the anchor of the module. Two mechanisms, stacked: **PagedAttention**, the allocator that makes non-contiguous KV storage possible, and **continuous batching**, the scheduler built on top of it that keeps the reclaimed capacity full. Everything in 07.4 (production tuning), 07.5 (the cost curve), and 07.6–07.10 assumes you hold blocks, block tables, reference counting and iteration-level scheduling as load-bearing mechanism rather than folklore.

What it unlocks immediately: 07.4 takes the knobs introduced here — `gpu-memory-utilization`, `max-num-seqs`, `max-num-batched-tokens`, `block-size` — and asks what happens when you push the block pool past its edge. That is preemption, which is this same allocator seen from its failure side.

Everything version-specific is checked against the **vLLM main branch cloned 2026-08-17 (commit `c1e4387`)**, specifically `vllm/v1/core/block_pool.py`, `vllm/v1/core/kv_cache_utils.py`, `vllm/v1/core/sched/scheduler.py`, `vllm/config/cache.py`, `vllm/config/scheduler.py`, and `docs/design/prefix_caching.md`. Where behaviour differs from the 2023 paper, the source wins and the difference is called out.

## Why this matters

Wasted KV memory is wasted GPU, one-for-one. From 07.1's chain: KV pool → concurrent sequences → resident batch → decode throughput → cost per million tokens. A contiguous allocator that holds 20 % useful state is not "somewhat inefficient" — it is a **5× multiplier on your cost per token**, applied to every hour of every GPU you rent, invisibly, with nothing in any dashboard flagging it. PagedAttention is the mechanism that turns "60 % of your H100's HBM is stranded" into ">96 % of it is usable KV," and that single change is the largest step-function improvement in LLM serving efficiency of the last several years.

It is also, concretely, the question that gates the interview loop this module targets. "Design an LLM inference platform" drills into KV-cache memory management within the first ten minutes, every time. A candidate who can only say "vLLM handles it" fails the immediate follow-up — *handles it how?* Being able to draw the block table, explain why the attention kernel needs to gather rather than stride, describe what a reference count buys you for a shared system prompt, and say precisely why internal fragmentation drops from ~80 % to under one block per sequence is the difference between a pass and a "strong platform background, weak on inference internals" write-up.

And the transferable knowledge is not vLLM trivia. It is that **an LLM serving engine is, at its core, a memory allocator with a scheduler attached**, and that both halves were solved by borrowing directly from operating systems: paging with page tables for the allocator, and preemptive iteration-level scheduling for the scheduler. If you already carry OS intuition from general platform work, you have most of this for free — the work is mapping it precisely, which is what §2 does.

## What's new here (calibration)

Per the module README, module 03 (prefill/decode, roofline, memory-bound decode, KV cache as a concept, FP8) and module 05 (TTFT/ITL/TPOT, `/metrics`) are referenced, not re-taught. This lesson is squarely in the "genuinely new" list.

- **07.1** gave you the HBM budget and the KV residual. **07.2** gave you the cap and the size of the fragmentation problem. **New here:** the KV cache is not one buffer per sequence — it is a *shared pool of fixed-size blocks* with an indirection layer, and the scheduler churns that pool every single forward pass.
- **New: the block table as a page table**, with the mapping made exact rather than analogical, including what the attention kernel has to do differently to make it work.
- **New: the allocation and eviction algorithms**, step by step — the free queue's structure, why blocks are returned in reverse order, what `touch` does, and why eviction is O(1).
- **New: content-addressed blocks.** How a block hash is computed, why only full blocks are cached, what `cache_salt` is for, and the difference between in-batch sharing (reference counting / copy-on-write) and across-time sharing (automatic prefix caching).
- **New: iteration-level scheduling**, described as the scheduler actually implements it — which is *not* "run prefill, then run decode," and which is why chunked prefill is a natural consequence rather than a bolted-on feature.
- **New: an honest account of what PagedAttention does not fix**, including the costs it introduces.

Out of scope, flagged so you know the boundary: production config tuning and preemption mechanics in depth (07.4), turning throughput into a cost curve (07.5), disaggregated prefill/decode (07.6), KV-cache quantisation (07.7).

## Core concepts

### 1. The problem, and why the obvious fixes fail

Restating 07.2 in one paragraph: a contiguous allocator gives each sequence one buffer sized to `max_model_len`, because output length is unknown at admission, because a contiguous buffer cannot grow in place when its neighbour is occupied, and because a naive attention kernel indexes K and V with a fixed stride. The result is three compounding wastes — internal fragmentation (early-finishing requests hold dead tails), over-reservation (even long requests hold their maximum from token 1), and external fragmentation (freed variable-sized buffers leave holes too small to admit anything) — leaving roughly 20–40 % of KV memory holding real token state.

Before reaching for paging, be sure the easy answers really do fail, because knowing *why* is what makes the fix inevitable rather than clever:

- **"Just allocate the actual length."** You do not know it. The model emits an end-of-sequence token when it decides to. You could cap it with `max_tokens` per request, but callers set that generously for exactly the same reason allocators over-reserve.
- **"Grow the buffer with `realloc`."** Growing means copying, and the thing you are copying is hundreds of megabytes of KV, on the GPU, in the middle of a decode step that has a 5 ms budget. A single 8k-context 70B sequence is 2.5 GiB; copying it at 3.35 TB/s is 1.5 ms round-trip, per growth, per sequence.
- **"Compact the pool periodically, like a moving GC."** Same objection at larger scale, plus you would have to stop the world across every in-flight sequence to update every kernel's base pointers.
- **"Just set `max_model_len` low."** This works, and 07.2 tells you to right-size it — but it caps the product, not the waste. At `max_model_len = 2048` with the same length distribution you still hold ~80 % dead reservation; you have merely made the dead part smaller in absolute terms while also refusing long requests.

Each failure points the same direction: the problem is the **requirement of contiguity**, not the size of the allocation. Break contiguity and all four objections dissolve. That is the entire idea.

### 2. PagedAttention: the block table is a page table

Partition each sequence's KV cache into **KV blocks**, each holding a fixed number of tokens (`block_size`, default **16**, from `CacheConfig.DEFAULT_BLOCK_SIZE`). Blocks live at arbitrary, non-contiguous physical locations in the pool. A per-sequence **block table** maps logical block index → physical block id. The attention kernel dereferences that table.

The mapping to virtual memory is exact, not metaphorical:

| OS virtual memory | PagedAttention | Where it lives in vLLM |
|---|---|---|
| Process virtual address space | One sequence's logical KV positions | `Request.num_computed_tokens` and its block list |
| Page (4 KiB) | KV block (16 tokens × every layer) | `KVCacheBlock` |
| Page table | Block table (array of physical block ids) | `req_to_blocks` in the KV cache manager |
| Physical frame | Physical KV block in HBM | index into the pre-allocated pool |
| Frame allocator / free list | Free block queue, LRU-ordered | `FreeKVCacheBlockQueue` |
| Page fault → allocate a frame | Sequence crosses a 16-token boundary → pop a block | `BlockPool.get_new_blocks` |
| Shared page + refcount | Shared KV block + `ref_cnt` | `KVCacheBlock.ref_cnt` |
| Copy-on-write | Copy-on-write on a partial-hit block | `kv_cache_block_copies` in `SchedulerOutput` |
| Page cache / LRU eviction | Automatic prefix caching with LRU eviction | `cached_block_hash_to_block` + free queue head |

```
  CONTIGUOUS ALLOCATION            │  PAGED ALLOCATION (PagedAttention)
  ═════════════════════════════════╪══════════════════════════════════════════
                                   │
  Sequence A needs max_model_len   │  Sequence A's BLOCK TABLE
  bytes, contiguous, up front.     │  (logical index → physical block id)
                                   │
  ┌──────────────────────────────┐ │    logical  0    1    2    3    4
  │ A: reserved 2048 tokens      │ │           ┌────┬────┬────┬────┬────┐
  │ ████░░░░░░░░░░░░░░░░░░░░░░░░ │ │           │4711│ 92 │5003│ 17 │ …  │
  │ ▲190 used   ▲1858 DEAD       │ │           └─┬──┴─┬──┴─┬──┴─┬──┴────┘
  └──────────────────────────────┘ │             │    │    │    │
  ┌──────────────────────────────┐ │             │    │    │    └──────┐
  │ B: reserved 2048 tokens      │ │             │    │    └────────┐  │
  │ ████████████████░░░░░░░░░░░░ │ │             │    └───────┐     │  │
  │ ▲1600 used        ▲448 DEAD  │ │             ▼            ▼     ▼  ▼
  └──────────────────────────────┘ │  PHYSICAL POOL (order irrelevant)
  ┌──────────────────────────────┐ │  ┌────┬────┬────┬────┬────┬────┬────┐
  │ C: reserved 2048 tokens      │ │  │ 17 │ 92 │ 93 │4711│4712│5003│ …  │
  │ █░░░░░░░░░░░░░░░░░░░░░░░░░░░ │ │  │ A3 │ A1 │ B0 │ A0 │ C0 │ A2 │free│
  │ ▲64 used  ▲1984 DEAD         │ │  └────┴────┴────┴────┴────┴────┴────┘
  └──────────────────────────────┘ │
  ┌──────────────────────────────┐ │  Sequence B's BLOCK TABLE: [93, …]
  │ ▒▒▒ 900-token hole ▒▒▒       │ │  Sequence C's BLOCK TABLE: [4712, …]
  │ too small for a 2048         │ │
  │ reservation → UNUSABLE       │ │  A 4th sequence needs ONE free block to
  └──────────────────────────────┘ │  be admitted, not 2048 contiguous
                                   │  tokens. Every free block is
  D arrives → REFUSED              │  interchangeable. D is ADMITTED.
                                   │
  ─────────────────────────────────┼──────────────────────────────────────────
  WASTE per sequence:              │  WASTE per sequence:
    max_model_len − actual_length  │    ≤ block_size − 1 = 15 tokens
    (scales with your context      │    (a CONSTANT, independent of context
     setting; ~80 % in practice)   │     length; ~1 % at realistic lengths)
  EXTERNAL fragmentation: yes      │  EXTERNAL fragmentation: structurally
                                   │    impossible — blocks are uniform
```

**Three properties fall out, and they are the whole payoff:**

1. **Near-zero fragmentation.** The only remaining waste is internal, in the *last, partially filled* block of each sequence, bounded by `block_size − 1` tokens **per sequence, forever**, independent of how long the sequence grows. The paper reports under 4 % waste versus the 60–80 % of contiguous systems. External fragmentation is not reduced — it is structurally eliminated, because uniform blocks means every free block is interchangeable and there is no such thing as a hole of the wrong shape.

2. **On-demand growth.** A sequence takes the blocks its prompt needs, then one more block every `block_size` generated tokens. No reservation to `max_model_len`. This is what turns 07.2's "worst-case concurrency 48×" into a measured 97 concurrent sequences on the same pool.

3. **Admission without contiguous holes.** The scheduler admits when *enough free blocks exist anywhere*, which is a counter comparison rather than a search. That is what physically permits a high `max-num-seqs`.

### 3. What the attention kernel has to do differently

This is the part most explanations skip, and it is the reason PagedAttention is a *paper* rather than a config change.

Standard attention for one query at position `t` computes `softmax(q_t · K^T / √d) · V` over all previous positions. In a contiguous layout, `K` for a sequence is one tensor with a fixed stride: the kernel walks `K[0], K[1], … K[t-1]` at regular address increments, which is exactly what a GPU's memory subsystem is optimised for — wide, coalesced, prefetchable reads.

Under paging, positions `0..15` live in one physical block, `16..31` in another, and those blocks are anywhere. So the kernel must, for each logical block `i`, look up `block_table[i]`, compute the physical base address, and read 16 positions from there. Mechanically:

```
  PAGED ATTENTION KERNEL, per query, per head (sketch)

    acc      = 0                     # running weighted sum of V
    m        = -inf                  # running max, for numerically stable softmax
    l        = 0                      # running sum of exp

    for i in range(num_logical_blocks):
        phys   = block_table[i]                      # ← the indirection
        K_blk  = K_cache[phys]        # [block_size, head_dim]
        V_blk  = V_cache[phys]
        s      = (q @ K_blk.T) * scale               # [block_size]
        s      = mask_out_positions_past_t(s)
        m_new  = max(m, s.max())
        alpha  = exp(m - m_new)                      # rescale prior accumulation
        acc    = acc * alpha + (exp(s - m_new) @ V_blk)
        l      = l * alpha + exp(s - m_new).sum()
        m      = m_new

    out = acc / l
```

Note two things. First, the loop is over *blocks*, and the block-table lookup is one extra load per block — 32 extra loads for a 512-token context, against reading 512 × head_dim × 2 bytes of K and the same of V. The indirection is genuinely cheap in relative terms. Second, the accumulation is FlashAttention's online-softmax rescaling: because the kernel already processes attention in tiles to keep the softmax denominator in registers rather than materialising the full `t × t` score matrix, **processing in blocks was already the shape of the computation**. PagedAttention's contribution is making the tile boundaries coincide with allocation boundaries and letting the tiles be scattered.

What you pay: the reads are less predictable, so the hardware prefetcher helps less, and the kernel carries block-table registers. In practice this costs a few percent of attention-kernel throughput on a workload where attention is a minority of the step time (07.1: weight reads dominate decode until context gets long). What you get: roughly 3–5× more sequences resident. The trade is not close.

### 4. The block pool, the free queue, and allocation step by step

The data structures are small enough to hold in your head, and holding them is what lets you reason about eviction behaviour you did not expect.

**The pool** (`vllm/v1/core/block_pool.py`) pre-allocates *every* `KVCacheBlock` object at initialisation — `self.blocks = [KVCacheBlock(idx) for idx in range(num_gpu_blocks)]`. Not the KV tensors (those are one big allocation), the small Python objects that track them. The design note in `docs/design/prefix_caching.md` is explicit about why: it "avoids Python object creation overheads and can easily track all blocks all the time." Per-step Python overhead is the thing V1 was rewritten to eliminate, and this is one of the places it shows.

Each block carries:

```python
class KVCacheBlock:
    block_id: int                          # immutable, indexes the KV tensors
    block_hash: BlockHash                  # assigned when the block becomes FULL;
                                           # cleared when the block is evicted
    ref_cnt: int                           # how many requests currently use it
    prev_free_block: "KVCacheBlock | None" # ← the free list is threaded THROUGH
    next_free_block: "KVCacheBlock | None" #   the blocks themselves
```

**The free queue** (`FreeKVCacheBlockQueue`) is a doubly linked list built from those `prev_free_block` / `next_free_block` pointers rather than a Python `deque`. The source explains the choice: a `deque` cannot remove from the middle in O(1), and any wrapper object would allocate. Removing from the middle is required constantly, because a cached-but-free block gets *revived* the moment a new request hits its hash — you must pull it out of wherever it sits in the eviction order without walking the list.

Its ordering is the eviction policy, and it has a subtlety worth knowing:

> The queue is ordered by block ID initially. When a block is allocated and then freed, it is appended back in eviction order: the least recently used block is at the front (LRU); if two blocks have the same last-access time (allocated by the same sequence), the one with **more hash tokens** — the tail of a block chain — is at the front.

That second clause is implemented by **freeing a request's blocks in reverse order**. When a request finishes, its last block goes back first, so it sits nearer the head and is evicted first. The reasoning: the last block of a sequence hashes the *most* tokens of context and is therefore the least likely prefix for any future request to match, while the first block (the system prompt's opening) is the most likely. **Eviction order is a bet about prefix reuse**, and the bet is encoded in a `reversed()` call.

Now the algorithms, from `docs/design/prefix_caching.md` and the scheduler source.

**Admitting a new request:**

```
  1. get_computed_blocks(request)
       Hash the prompt block by block. Look each hash up in
       cached_block_hash_to_block. Return the longest run of blocks that
       already exist. (Prefix cache hit — §6.)

  2. allocate_slots(request, num_new_tokens)
       a. Compute how many NEW blocks are needed.
          If insufficient free blocks exist → return None (do not admit).
       b. TOUCH the computed blocks: ref_cnt += 1 on each, and if a block's
          ref_cnt was 0 it was sitting in the free queue as an eviction
          candidate — remove it from the queue so it cannot be evicted
          out from under the request that just claimed it.
       c. Allocate new blocks by popping heads of the free queue. If the
          head block is currently cached, popping it EVICTS it: its hash is
          removed from the map so no future request can hit it.
       d. If an allocated block ends up full of tokens, cache it immediately,
          so a sibling request in the SAME batch can already reuse it.
```

**Advancing a running request:** same as (2) minus the cache lookup — compute new block need, pop from the free queue (evicting cached heads if necessary), append token ids into the tail block, and cache any block that just became full.

**Freeing:** decrement `ref_cnt` on every block; any that reach zero go to the *tail* of the free queue, in reverse block order, still carrying their hashes so they remain cache-hittable until actually evicted.

```
  BLOCK POOL OVER TIME — block_size = 4, pool of 10 blocks
  ══════════════════════════════════════════════════════════════════════════════
  ██ = in use (ref_cnt > 0)   ▓▓ = free but still CACHED (hittable)
  ░░ = free and empty

  T1  Request R0 arrives, prompt = 15 tokens → needs ceil(15/4) = 4 blocks.
      Blocks 0,1,2 become full and are cached. Block 3 holds 3 of 4 tokens.
      pool:  [██0][██1][██2][██3][░░4][░░5][░░6][░░7][░░8][░░9]
      R0 block table: [0,1,2,3]        cached hashes: {h0→0, h1→1, h2→2}
      free queue head → 4,5,6,7,8,9

  T2  R0 decodes one token → block 3 becomes FULL → cache it, and allocate
      block 4 for the next token.
      pool:  [██0][██1][██2][██3][██4][░░5][░░6][░░7][░░8][░░9]
      cached hashes: {h0→0, h1→1, h2→2, h3→3}

  T3  Request R1 arrives with 14 prompt tokens whose FIRST 10 match R0.
      Hash lookup: block 0 (tokens 0-3)   → HIT  h0
                   block 1 (tokens 4-7)   → HIT  h1
                   block 2 (tokens 8-11)  → MISS (only 8,9 match; 10,11 differ)
      ⇒ 2 blocks hit = 8 tokens of prefill SKIPPED, not 10. Cache hits land
        on BLOCK boundaries, so 2 of the 10 matching tokens are re-prefilled.
      touch(0), touch(1) → ref_cnt 0→1 becomes 2 for those blocks
      allocate 5,6 for the rest of R1's prompt.
      pool:  [██0][██1][██2][██3][██4][██5][██6][░░7][░░8][░░9]
      R1 block table: [0,1,5,6]     ← SHARES physical blocks 0 and 1 with R0

  T4  R0 finishes. Free its blocks in REVERSE order: 4,3,2 (0 and 1 still
      have ref_cnt 1 from R1, so they are NOT freed).
      pool:  [██0][██1][▓▓2][▓▓3][░░4][██5][██6][░░7][░░8][░░9]
      free queue (head→tail): 7,8,9, 4, 3, 2
                              ▲ empty blocks first (they were never used),
                                then R0's tail-first ordering: 4 (empty tail),
                                then 3, then 2 — so the DEEPEST-context
                                cached block is evicted before the shallowest.

  T5  A new request needs 5 blocks. Pop heads: 7,8,9 (empty), then 4 (empty),
      then 3 → block 3 was CACHED, so popping it EVICTS h3 from the hash map.
      Block 2 survives one more round.

  ⇒ THE INVARIANT: a block is hittable exactly while (ref_cnt > 0) OR
    (ref_cnt == 0 AND it has not yet been popped off the free queue).
    Prefix caching is therefore FREE MEMORY-WISE — cached blocks live in
    space that is already free and would otherwise sit empty. It costs
    nothing to hold them and it costs one hash-map delete to evict them.
```

That last observation is worth pausing on, because it is why prefix caching could be turned on by default. **The cache does not reserve memory.** It is opportunistic reuse of blocks that are free anyway. The only costs are the hashing (CPU) and the bookkeeping (CPU), which is exactly what V1's rewrite drove down.

### 5. Content-addressed blocks: how the hash works

A block can be shared only if two sequences agree that its contents are identical. vLLM establishes that by hashing, and the scheme has to handle the fact that identical *tokens* in a block do not imply identical *KV*, because K and V depend on everything before them through attention.

The fix is hash chaining. From `docs/design/prefix_caching.md`:

```
                    Block 1                  Block 2                  Block 3
         [A gentle breeze stirred] [the leaves as children] [laughed in the distance]
Block 1: |<--- block tokens ---->|
Block 2: |<------- prefix ------>| |<--- block tokens --->|
Block 3: |<------------------ prefix -------------------->| |<--- block tokens ---->|
```

and in code (`hash_block_tokens` in `vllm/v1/core/kv_cache_utils.py`):

```python
BlockHash(hash_function((parent_block_hash, curr_block_token_ids_tuple, extra_keys)))
```

Three components, each load-bearing:

- **`parent_block_hash`** — the hash of the preceding block, `NONE_HASH` for the first. This makes a block's identity depend on its entire prefix, so block 3 of "A gentle breeze stirred / the leaves as children / laughed in the distance" cannot collide with block 3 of a different document that happens to end with the same four words.
- **`curr_block_token_ids`** — the exact token ids in this block. Included to reduce collision risk even given a matching parent hash.
- **`extra_keys`** — everything else that makes the KV distinct: the LoRA adapter id (07.10 — the same tokens under a different adapter produce different KV), multimodal input hashes (the placeholder tokens for an image are identical across images, so the image's own hash must enter here), and the optional `cache_salt`.

Four rules that follow, and that you will hit in practice:

1. **Only full blocks are cached.** A partially filled tail block has no stable identity yet — its remaining slots will be written by tokens not yet decided. This is why the T3 example above hits 8 tokens rather than 10: **prefix cache hits land on `block_size` boundaries.** With `block_size = 16`, a 2,000-token shared system prompt yields hits on `floor(2000/16) × 16 = 1,984` tokens and re-prefills the last 16. (Current vLLM adds a `prefix_match_unit` / `hash_block_size` concept that lets hash granularity be finer than the physical block size, which softens this for hybrid models with large physical blocks — but the "full unit only" rule stands.)

2. **The hash algorithm is configurable, and the default changed.** `--prefix-caching-hash-algo` accepts `sha256` (default as of v0.11, using `pickle` for serialisation), `sha256_cbor` (reproducible across languages and Python versions — use this if you need deterministic caching across environments), `xxhash` (128-bit, non-cryptographic, faster, requires the `xxhash` package), and `xxhash_cbor`. The move to SHA-256 as default was a deliberate collision-safety decision: a hash collision in a multi-tenant deployment means one tenant's request reads another tenant's KV, which is a data-leak class of bug, not a performance bug.

3. **`cache_salt` isolates tenants.** Passing `"cache_salt": "<value>"` in a request injects that value into the first block's hash, so only requests carrying the same salt can share blocks. This closes a genuine timing side channel: without it, an adversary can infer whether a given prefix has been seen recently by measuring TTFT. The cost is that you fragment the cache along salt boundaries, which is exactly the trade you want — sharing inside a trust group, isolation across them.

4. **Duplicate cached blocks are tolerated in V1.** If two requests with the same prompt run concurrently and one allocates a fresh block for content that is already cached, V1 does not redirect it, because **the V1 block table is append-only** — rewriting an entry from `[0, 3]` to `[0, 1]` mid-flight is not allowed. The duplicate is cleaned up when the request frees. This is a deliberate simplification that trades a small amount of memory for a much simpler concurrency story, and it is a difference from V0 worth knowing if you are reading old code.

### 6. Sharing: reference counting, copy-on-write, and automatic prefix caching

Three related mechanisms, distinct lifetimes. Keeping them apart is the difference between being able to answer "why did request C get a cache hit when A and B had both already finished?" and not.

**Reference counting (in-batch sharing).** Two sequences whose prefixes hash identically point their block tables at the *same physical blocks*, and each block's `ref_cnt` counts how many requests hold it. If 500 concurrent requests all begin with the same 2,000-token system prompt, that prefix's KV is stored **once**. For Llama-3.3-70B at bf16 KV that is `2000 × 320 KiB = 625 MiB` versus `500 × 625 MiB = 305 GiB` — the difference between "the system prompt is free" and "the system prompt is the entire deployment." This is also how parallel sampling (`n > 1`) and beam search share prompt KV: the branches literally are sequences with an identical prefix.

**Copy-on-write.** Sharing works while contents agree. When a sequence must write into a block whose contents are shared — the classic case is a *partial* prefix-cache hit, where a block matches for its first `k` tokens and diverges after — the block cannot simply be mutated, because the other holders depend on it. vLLM copies that one block, decrements the original's `ref_cnt`, and lets the writer mutate its private copy. In current V1 this is explicit in the scheduler output as `kv_cache_block_copies`, applied in the model runner before the forward pass (`vllm/v1/worker/gpu/model_runner.py`: *"Apply copy-on-write block copies for partial prefix-cache hits"*). One block copied, not the whole sequence — which is precisely the property paging buys you.

**Automatic prefix caching (across-time sharing).** Reference counting only helps sequences that are *concurrently resident*. APC extends sharing across time: when a request finishes, its full blocks are not scrubbed. They keep their hashes, sit in the free queue as eviction candidates, and a later request whose prefix hashes the same finds them already populated and **skips prefill for the cached span entirely**. The blocks are reclaimed only when the free queue's head reaches them and something needs the space.

```
  THE THREE SHARING MECHANISMS, ON ONE TIMELINE

  t →   0        1        2        3        4        5        6        7
        │        │        │        │        │        │        │        │
  REQ A ├════════════════════════════════╡ (finishes at t=4)
  REQ B      ├════════════════════════════════════╡ (finishes at t=5)
  REQ C                                                 ├══════════════╡
        │        │        │        │        │        │        │
        │        └── A and B share the system-prompt blocks
        │            REF COUNTING: ref_cnt = 2, one physical copy.
        │            Lifetime: while BOTH are resident.
        │
        │                          └── B's prompt matched A's for 1,990 of
        │                              2,000 tokens. The block containing
        │                              the divergence is COPIED (COW).
        │                              Cost: one block, not one sequence.
        │
        └───────────────────────────────────────────── C starts at t=6, long
                                                       after A and B are gone.
                                                       Their blocks are free
                                                       but still hashed →
                                                       APC HIT. C skips
                                                       prefill for the shared
                                                       prefix.
                                                       Lifetime: until evicted
                                                       from the LRU free queue.

  ⇒ REF COUNTING is in-batch and ends when a sharer finishes.
    COW is the divergence handler for both.
    APC is across-time and ends when the blocks are evicted.
    "Why did C hit?" → APC. "Why is the system prompt free for A and B
    simultaneously?" → ref counting. Different questions, different answers.
```

**APC is on by default in current vLLM** (`enable_prefix_caching: bool = True` in `vllm/config/cache.py`). That default is a hard-won engineering result, not an obvious choice. In V0, prefix caching carried enough CPU overhead — Python object churn, non-constant-time eviction — that enabling it *hurt* throughput at low hit rates, so it was opt-in. V1's rewrite gave it constant-time eviction (the embedded doubly linked free list of §4) and minimal per-request object creation, and the vLLM team reported near-zero performance degradation **even at a 0 % cache hit rate**. That is the property that makes default-on safe: there is no workload where turning it on is clearly wrong.

Measure it with `vllm:prefix_cache_hits` ÷ `vllm:prefix_cache_queries`. Both counters are in **tokens**, not requests — a detail that trips people up, because it means the ratio is a token-weighted hit rate and a single very long shared prefix can dominate it.

### 7. Continuous batching: iteration-level scheduling

PagedAttention makes a large batch physically *fit*. Continuous batching is what keeps it *full*.

**Static batching** — the baseline every naive `generate()` loop implements — assembles B requests, runs them to completion together, and returns them together. The batch finishes when its slowest sequence does. Put a 20-token reply and a 2,000-token reply in the same batch and the short one's slot sits idle for 1,980 steps. Anyscale's benchmark shows exactly this collapse: their naive static baseline fell to **81 tokens/second** on OPT-13B once realistic output-length variance was introduced, and their headline result was that continuous batching plus batching-aware memory management reached up to **23× that throughput** on the same 40 GB A100. Notice the framing — with fixed-length outputs static batching is fine, and it is the *variance* that destroys it, which means the multiplier you get depends entirely on your traffic's length distribution.

**Continuous (iteration-level) batching**, introduced as a research idea by Orca (Yu et al., OSDI 2022, which reported 36.9× throughput at matched latency versus FasterTransformer on GPT-3 175B), schedules at the granularity of a single forward pass. After every iteration the scheduler re-plans: sequences that emitted an end-of-sequence token are retired and their blocks returned to the pool; waiting requests are admitted into the freed capacity; the batch composition changes mid-flight.

```
  STATIC BATCHING vs CONTINUOUS BATCHING — where GPU time is wasted
  ══════════════════════════════════════════════════════════════════════════════
  Five requests. Output lengths: A=8, B=2, C=12, D=3, E=6 tokens.
  █ = a slot doing useful work    · = a slot occupied but IDLE (waste)

  ── STATIC BATCHING (batch of 4, then the next batch) ──────────────────────
   iter:  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20
   A   │  █  █  █  █  █  █  █  █  ·  ·  ·  ·│
   B   │  █  █  ·  ·  ·  ·  ·  ·  ·  ·  ·  ·│         E waits for the ENTIRE
   C   │  █  █  █  █  █  █  █  █  █  █  █  █│         first batch to drain,
   D   │  █  █  █  ·  ·  ·  ·  ·  ·  ·  ·  ·│         even though slots freed
       └──────────── batch 1 ───────────────┘         at iteration 3.
   E   │                                     █  █  █  █  █  █│
                                             └─── batch 2 ───┘

   Useful slot-iterations : 8+2+12+3+6 = 31
   Total slot-iterations  : 4 × 12 + 1 × 6 = 54
   UTILISATION            : 31/54 = 57 %
   Wall clock             : 18 iterations
   E's TTFT               : 12 iterations of pure queueing

  ── CONTINUOUS BATCHING (max_num_seqs = 4, re-plan every iteration) ────────
   iter:  1  2  3  4  5  6  7  8  9 10 11 12
   A   │  █  █  █  █  █  █  █  █ ✓
   B   │  █  █ ✓
   C   │  █  █  █  █  █  █  █  █  █  █  █  █ ✓
   D   │  █  █  █ ✓
   E   │        █  █  █  █  █  █ ✓
              ▲
              └─ B finished at iteration 2. Its blocks returned to the pool
                 at the END of iteration 2, so E is admitted at iteration 3.
                 Nobody waits for a batch to drain.

   Useful slot-iterations : 31
   Total slot-iterations  : 31   (a slot exists only while it is working)
   UTILISATION            : 100 %
   Wall clock             : 12 iterations  (−33 %)
   E's TTFT               : 2 iterations   (−83 %)

  ┌────────────────────────────────────────────────────────────────────────┐
  │ WHY THIS NEEDS PAGEDATTENTION                                          │
  │ Admitting E at iteration 3 means finding KV space for E at iteration   │
  │ 3. Under contiguous allocation that requires a max_model_len-sized     │
  │ contiguous hole to have appeared exactly where B's buffer was — and    │
  │ E's reservation must fit it. Under paging it requires only that        │
  │ enough free BLOCKS exist anywhere, which is an integer comparison.     │
  │ Continuous batching without paging is a scheduler writing cheques the  │
  │ allocator cannot cash.                                                 │
  └────────────────────────────────────────────────────────────────────────┘
```

**The V1 scheduler does not have phases.** This is the single most important structural fact about it, and the source states it directly (`vllm/v1/core/sched/scheduler.py`):

> *"There's no 'decoding phase' nor 'prefill phase' in the scheduler. Each request just has the `num_computed_tokens` and `num_tokens_with_spec` … At each step, the scheduler tries to assign tokens to the requests so that each request's `num_computed_tokens` can catch up its `num_tokens_with_spec`. This is general enough to cover chunked prefills, prefix caching, speculative decoding, and the 'jump decoding' optimization in the future."*

Read that as a design principle: **there is one quantity, "tokens this request still owes," and one budget, "tokens this iteration may process."** A request with 8,000 unprocessed prompt tokens and a request with 1 unprocessed sampled token are the same kind of object to the scheduler; they differ only in how many tokens they want. Chunked prefill is not a feature bolted onto this design — it is what the design *is*, once you cap the per-iteration budget.

Two budgets bound each iteration:

- **`max_num_batched_tokens`** — the total tokens the iteration may process. Defaults are device- and usage-context-dependent in current vLLM (`EngineArgs.get_batch_defaults`): for the OpenAI API server, **8192** on H100/H200-class cards (≥70 GB, non-A100), **16384** on ≥160 GB cards, and **2048** otherwise — with a source comment noting that large values *reduce* throughput specifically on A100, which is why the device-name check exists.
- **`max_num_seqs`** — the maximum sequences in one iteration. Defaults to **1024** on ≥70 GB non-A100 cards and **256** otherwise.

Both are dials you will set explicitly in 07.4. The point here is that they are the *scheduler's* limits, distinct from the KV pool's physical limit from 07.2, and any of the three can bind first.

### 8. Chunked prefill, and why decode is protected

Under the "tokens owed" model, a request with a 8,192-token prompt owes 8,192 tokens. If the iteration budget is 8,192 and it takes all of it, then every other resident sequence gets zero tokens that iteration — and a decode iteration that normally takes ~5 ms becomes one ~40 ms iteration. Every streaming user sees an 8× inter-token-latency spike caused by someone else's prompt.

**Chunked prefill** splits that prompt across iterations so it competes for, rather than monopolises, the budget. It is enabled by default whenever possible in V1 (`enable_chunked_prefill: bool = True`), and vLLM's optimization docs are explicit about the resulting dial:

- **Smaller `max_num_batched_tokens` (e.g. 2048) improves ITL**, because fewer prefill tokens crowd out decode steps in any given iteration.
- **Larger (>8192) improves TTFT**, because prompts finish sooner.

It is one dial with two ends, not two independent knobs — a fact worth internalising before you "tune for throughput" and discover you have doubled your inter-token latency.

Two chunking subtleties visible in the scheduler source that matter for prefix caching:

- **Chunk ends are block-aligned where possible.** The scheduler explicitly rounds a chunk's end down to a `block_size` boundary (`aligned_end = end // block_size * block_size`), because a block's KV state is written at chunk ends and a block that is only half-written cannot be hashed and cached. Misaligned chunking would silently destroy prefix-cache hit rates on the very prompts that most need them.
- **The scheduler prioritises decode.** Running requests are scheduled first and prefill work is admitted from the remaining budget. Operationally this means **under load, vLLM automatically trades TTFT away to protect ITL.** If you do not know that, you will read a TTFT-only regression under load as a capacity failure when it is the scheduler working exactly as designed.

### 9. When the pool runs dry: preemption

The allocator's failure mode, previewed here and covered operationally in 07.4.

When `allocate_slots` cannot find free blocks for a running request that needs to grow, the scheduler evicts a running request to make room. The victim selection is in `vllm/v1/core/sched/scheduler.py` and is simpler than most people expect:

```python
if self.policy == SchedulingPolicy.PRIORITY:
    preempted_req = max(self.running, key=lambda r: (r.priority, r.arrival_time))
else:                                   # FCFS — the default (policy = "fcfs")
    preempted_req = self.running.pop()  # the LAST request in the running list
```

Under the default FCFS policy the victim is the **most recently admitted** running request — LIFO eviction, which preserves the progress of the oldest requests and is the standard anti-convoy choice. Under `policy = "priority"` it is the lowest-priority, latest-arriving request.

What preemption does to the victim (`_preempt_request`):

```python
self._free_request_blocks(request)     # ALL its blocks return to the pool
request.status = RequestStatus.PREEMPTED
request.num_computed_tokens = 0        # ← everything it computed is discarded
request.num_preemptions += 1
self.waiting.prepend_request(request)  # goes to the FRONT of the queue
```

Three consequences to carry:

1. **Preemption is recomputation, not suspension.** `num_computed_tokens = 0` means the request will re-run its entire prefill when resumed. A preempted 8k-context request pays a second full 8k prefill. That is why preemptions hit TTFT-like intervals so hard and why `vllm:num_preemptions_total` climbing alongside a latency spike is one problem, not two.
2. **Prefix caching softens it materially.** The recompute is a fresh prefill, which goes through `get_computed_blocks` like any other — and the request's own blocks were just freed, so unless they have been evicted they are still hashed and hittable. With APC on, a preempted request often recovers most of its prefill from cache. This is an under-appreciated second-order benefit of default-on prefix caching.
3. **The swap path is gone.** V0 could copy a preempted sequence's KV to CPU memory over PCIe and copy it back. V1 does not: vLLM's own metrics design doc states that `vllm:num_requests_swapped` and `vllm:cpu_cache_usage_perc` are legacy metrics for a mode "no longer relevant in v1," and that **`--swap-space` has been removed**. If a tutorial tells you to tune `--swap-space`, it predates V1. Any content still discussing "recompute vs swap preemption modes" as a live choice is describing an engine you are not running.

### 10. PagedAttention is an algorithm, not a vLLM feature

Worth being explicit, because it is a common interview and résumé-review trap. PagedAttention is the *paper's* contribution (Kwon et al., SOSP '23) and vLLM was the first system to productionise it — but it is not vLLM-exclusive. SGLang implements its own paged KV manager plus RadixAttention (a radix-tree structure for prefix sharing that generalises the hash-chain approach to arbitrary shared subtrees). TensorRT-LLM has paged KV cache support. Essentially every modern engine converged on some variant of block-based non-contiguous KV storage, because the alternative is a 5× cost penalty.

Naming vLLM specifically as "the" solution reads as brand familiarity rather than mechanism understanding. The better formulation: *"a paged, block-based KV allocator — vLLM popularised it, most engines have converged on some form of it, and the differentiation between engines today is underneath the scheduling loop: how eviction and admission are implemented, how prefixes are hashed and shared, how chunked prefill interacts with the token budget."*

The same applies to continuous batching. It is no longer differentiating terminology; it is table stakes, the way TCP congestion control is table stakes for a network stack. An answer that says "vLLM does continuous batching, that's why it's fast" is describing 2023.

### 11. What PagedAttention does not fix

Be able to state the limits precisely; it is the difference between understanding and enthusiasm.

- **It does nothing for compute-bound workloads.** Long prompts and short generations (RAG, classification, extraction) are prefill-dominated, which is a FLOPs problem sitting above the roofline's ridge point. No allocator changes that. If your traffic is 4,000-token prompts producing 50-token answers, your bottleneck is tensor throughput and your levers are quantisation, better kernels, or more compute — not memory management.
- **It does not reduce KV bytes per token.** Fragmentation waste and cache size are different axes. PagedAttention reclaims the *stranded* portion; `n_kv`, `head_dim`, layer count and KV dtype set the *intrinsic* portion, and only architecture (07.1 §10) or quantisation (07.7) move those.
- **"<4 % waste" is a benchmark-configuration result, not a guarantee.** Real deployments pay a small but nonzero tax the headline number does not surface: the attention kernel's gather through the block table versus a dense contiguous read, and CPU-side bookkeeping — hashing every full block, reference counting, free-queue manipulation, eviction — competing for the same event loop as scheduling. V1's rewrite drove that tax down to the point where prefix caching defaults on, which is strong evidence it is small, but "small" is not "zero."
- **It does not eliminate preemption.** It raises the ceiling; it does not remove it. Oversubscribe the pool and you still evict, and eviction still costs a full recompute.
- **Prefix-cache hits are block-granular.** A shared prefix of 2,000 tokens with `block_size = 16` yields at most 1,984 tokens of hit. For short shared prefixes — a 20-token system message — the hit rate is materially worse than the naive "identical prefix ⇒ free" model suggests.

The honest one-line summary: **PagedAttention reclaims the overwhelming majority of previously wasted KV memory, at a small, well-amortised bookkeeping cost, and changes nothing about workloads whose bottleneck was never memory.**

### 12. Version pin: what is current and what is stale

Everything above describes the **V1 engine**, which is the default and only engine in current vLLM — V0 was removed in the 0.11.x line. This module targets **vLLM 0.11.x or later, V1**. Pin it in every command and record it in the deliverable.

Signals that a document predates V1 and should be discarded:

| Stale signal | Reality in V1 |
|---|---|
| `--swap-space` as a preemption tuning knob | Flag removed; the CPU-swap preemption path does not exist |
| "recompute vs swap preemption modes" | Only recompute exists |
| `BlockSpaceManagerV1` / `BlockSpaceManagerV2` | Replaced by `BlockPool` + `KVCacheManager` in `vllm/v1/core/` |
| "enable prefix caching with `--enable-prefix-caching`" as an opt-in | On by default (`enable_prefix_caching = True`) |
| Separate prefill and decode scheduler phases | One unified loop with a token budget; no phases |
| `vllm:gpu_cache_usage_perc` | Renamed `vllm:kv_cache_usage_perc` |
| `vllm:time_per_output_token_seconds` | Split into `vllm:inter_token_latency_seconds` (ITL) and `vllm:request_time_per_output_token_seconds` (TPOT) |
| `--enforce-eager` described as changing scheduler behaviour | It only disables CUDA graph capture |

## Perspectives

**The systems / OS-analogy view.** This is the paper's own framing and the right conceptual spine: pages, page tables, physical frames, page faults, reference counting, copy-on-write, LRU page-cache eviction — every one has a direct, load-bearing PagedAttention analogue, and §2's table is a mapping rather than a metaphor. If you carry OS memory-management intuition from general platform engineering you already have most of the model; the work is checking that each analogue really holds, and noticing where it does not (there is no swapping in V1, and there is no demand paging from a backing store — a "fault" allocates a fresh block, it does not read one back).

**The maintainer / engine-internals view.** The most valuable thing about reading `vllm/v1/core/` directly rather than a blog post is seeing which decisions were made for *performance of the control plane* rather than the data plane. Pre-allocating every `KVCacheBlock` object at init, threading the free list through the block objects rather than using a `deque`, making the block table append-only so no mid-flight rewrites are needed, tolerating duplicate cached blocks rather than paying for deduplication — these are all Python-overhead decisions, and collectively they are why V1 reported up to 1.7× the throughput of V0 and why prefix caching became cheap enough to default on. **The bottleneck in a modern inference engine is often the scheduler's Python, not the GPU**, which is a genuinely counter-intuitive thing to learn about a system whose entire purpose is matrix multiplication.

**The production-adopter view.** Red Hat's write-up on enterprise vLLM adoption is useful precisely because the two named workloads have almost nothing in common. Roblox runs a real-time consumer gaming platform serving billions of tokens per week to an in-product assistant, reporting a 50 % latency reduction; LinkedIn runs 50+ internal generative-AI use cases across an enterprise SaaS surface, including a Hiring Assistant. Both report meaningful wins from the same underlying mechanism. That is evidence the PagedAttention payoff is **workload-agnostic** — it is not a trick that only helps one traffic shape, because stranded memory is stranded regardless of what the requests look like.

**The skeptical / failure-mode view.** §11 is the counterweight. The headline "<4 % waste" is a best case on the paper's benchmark configuration, and real deployments pay costs the number does not surface. More importantly, PagedAttention is a *memory* fix, and a large fraction of production inference pain is not a memory problem: prefill-heavy RAG workloads are compute-bound, long-context decode is bandwidth-bound on KV reads rather than capacity-bound, and cold-start latency is a storage problem (07.9). Being able to say what the mechanism *does not* address is what stops you from reaching for it as a universal answer — and it is what an interviewer is probing when they ask "when would vLLM not help you?"

**The economics view.** Trace the chain: fragmentation waste ~80 % → ~4 % means roughly 3–5× more resident sequences on the same pool; resident sequences map roughly linearly to decode throughput while below the ridge point (07.1 §2); throughput is the denominator of cost per million tokens. So the allocator, by itself, is a 3–5× cost lever, before any tuning. Layer continuous batching's utilisation win on top — Anyscale's 23× against static batching under realistic length variance — and you have the two mechanisms that took self-hosted LLM inference from "obviously more expensive than an API" to "competitive with budget-tier API pricing." Neither required new hardware.

## Real-world use cases

- **Kwon et al., "Efficient Memory Management for LLM Serving with PagedAttention" (SOSP '23).** The originating result: contiguous per-request KV reservation left roughly 20–40 % of KV memory holding real token state; block-based allocation with a per-sequence block table bounds internal waste at under one block per sequence and eliminates external fragmentation entirely, delivering 2–4× throughput over FasterTransformer and Orca at matched latency. The paper also reports memory savings from block sharing specifically: on OPT-13B with an Alpaca trace, roughly 6.1–9.8 % for parallel sampling and 37.6–55.2 % for beam search. **What it shows:** the largest single efficiency win in LLM serving came from applying a 1960s operating-systems idea to a 2023 workload — and the sharing numbers show the beam-search case, where many branches share a long prefix, is where block sharing pays off most dramatically.

- **Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models" (OSDI 2022).** Introduced **iteration-level scheduling** — return control to the scheduler after every token so finished sequences leave and new ones join mid-flight — plus **selective batching**, applying token-wise batching to non-attention operations while processing attention per sequence, which is necessary because sequences in a batch have different KV lengths and cannot be batched into one naive attention call. Reported **36.9× throughput at matched latency** versus FasterTransformer on GPT-3 175B. **What it shows:** the scheduling half of this lesson predates the memory half by a year, and the two are complementary: Orca solved "keep the batch full," vLLM solved "make a full batch fit."

- **Anyscale — "Achieve 23× LLM Inference Throughput & Reduce p50 Latency."** OPT-13B on a single 40 GB A100. Continuous batching plus batching-aware memory management reached up to 23× the throughput of naive static batching; roughly 8× over Ray Serve's and HuggingFace TGI's implementations and ~4× over FasterTransformer's. Critically, the naive static baseline collapsed to **81 tok/s** once realistic variance in output lengths was introduced. **What it shows:** the multiplier is a function of your traffic's length distribution, not a constant. With uniform output lengths static batching is nearly fine; the win comes entirely from not letting short requests hold slots hostage to long ones. Quote the number with the workload attached or not at all.

- **Roblox and LinkedIn via Red Hat's enterprise adoption write-up.** Roblox adopted vLLM as its primary inference engine, reporting a **50 % latency reduction** while serving **4 billion tokens per week** to an in-product AI assistant. LinkedIn runs vLLM across **50+ generative-AI use cases** including its Hiring Assistant. **What it shows:** the same mechanism paying off across two maximally different traffic shapes — one latency-critical consumer feature at enormous volume, one long tail of internal enterprise tooling — which is the argument that this is infrastructure rather than a workload-specific optimisation.

- **vLLM V1 engine rewrite.** V1 reported up to **1.7× the throughput of V0**, attributed to comprehensive CPU-overhead reduction across the stack rather than to any single algorithmic change, and made prefix caching cheap enough to enable by default by moving to constant-time cache eviction and minimising per-request Python object creation — reporting near-zero degradation even at a **0 % cache hit rate**. **What it shows:** in a mature inference engine, the control plane is a first-order performance concern. The specific data structures in §4 (pre-allocated block objects, embedded doubly linked free list) are that lesson made concrete.

## Worked example

**Demonstrate prefix-cache sharing end to end, with a control.**

The goal is not "observe a lower latency." It is to produce evidence, in the form of a measurement plus a control that isolates the mechanism, that a shared prefix is served from cached blocks. A single number with no control is an anecdote.

### Setup

```bash
pip install "vllm==0.11.0"

vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --enable-prefix-caching \
  --gpu-memory-utilization 0.90 \
  --max-model-len 8192 \
  --max-num-seqs 256 \
  --port 8000
```

`--enable-prefix-caching` is redundant in current vLLM (it is the default) and is written out to make the experiment self-documenting. Capture the startup line:

```
INFO ... GPU KV cache size: 447,776 tokens, Maximum concurrency for 8,192 tokens per request: 54.66x
```

**That line is PagedAttention talking.** 447,776 usable KV tokens ÷ 8,192 = 54.66 sequences at full context — from *one shared pool of interchangeable blocks*, not from 54 pre-reserved per-request buffers. Under contiguous allocation the equivalent figure would require 54 max-length holes to exist simultaneously.

### The measurement

Build a fixed ~2,000-token system prompt, then send two requests with the *same* system prompt and *different* user turns.

```bash
SYS=$(python3 -c "print('You are a meticulous SRE assistant. '*250)")   # ≈2,000 tokens

snap() { curl -s localhost:8000/metrics | grep -E \
  '^vllm:(prefix_cache_queries|prefix_cache_hits)_total' | awk '{print $1, $2}'; }

ask() {
  curl -s -o /dev/null -w '  total=%{time_total}s\n' \
    localhost:8000/v1/chat/completions -H 'content-type: application/json' \
    -d "$(python3 - "$1" <<'PY'
import json, sys, os
sys_prompt = "You are a meticulous SRE assistant. " * 250
print(json.dumps({
  "model": "meta-llama/Llama-3.1-8B-Instruct",
  "messages": [{"role": "system", "content": sys_prompt},
               {"role": "user",   "content": sys.argv[1]}],
  "max_tokens": 16, "temperature": 0, "stream": False}))
PY
)"
}

echo "--- before A ---"; snap
ask "Explain etcd quorum loss."
echo "--- after A ---";  snap
ask "Explain a split-brain scenario."
echo "--- after B ---";  snap
```

A representative transcript (your absolute numbers will differ; the *deltas* are the result):

```
--- before A ---
vllm:prefix_cache_queries_total 0
vllm:prefix_cache_hits_total    0
  total=0.412s
--- after A ---
vllm:prefix_cache_queries_total 2016
vllm:prefix_cache_hits_total    0
  total=0.121s
--- after B ---
vllm:prefix_cache_queries_total 4032
vllm:prefix_cache_hits_total    2000
```

**Read it line by line.**

- **A queries 2,016 tokens and hits 0.** Cold cache. Every token of the prompt was prefilled. The counters are in **tokens**, which is why they are in the thousands rather than being 1 and 0.
- **B queries another 2,016 and hits 2,000.** The shared system prompt is 2,000 tokens; the divergent user turn is the remaining 16. `2000 / 2016 = 99.2 %` hit rate on B's prompt.
- **Why 2,000 and not 2,016?** Because the user turn differs, so the block containing the transition from system prompt to user message cannot match. Hits land on block boundaries, and here the boundary happened to fall cleanly. On a prompt where the divergence falls mid-block you will see a hit count rounded *down* to the nearest multiple of 16 — which is exactly §5's "only full blocks are cached" rule showing up in a metric.
- **A's wall time is 0.412 s and B's is 0.121 s.** A 3.4× improvement — but wall time on a `curl` includes connection setup, tokenisation, scheduling and 16 decode steps, so **do not report this as TTFT**. Read the server-side histogram instead:

```bash
curl -s localhost:8000/metrics | grep '^vllm:time_to_first_token_seconds_bucket' | head -12
```

and compute the ratio of `_sum` deltas across the two requests, or use a streaming client that timestamps the first SSE chunk.

### The control — the part that makes it evidence

```bash
# Restart with the mechanism OFF.
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --no-enable-prefix-caching \
  --gpu-memory-utilization 0.90 --max-model-len 8192 --port 8000
```

Repeat the identical A-then-B sequence. Expected result:

```
--- after A ---   total=0.401s
--- after B ---   total=0.388s      ← barely different from A
(prefix_cache_* counters absent or flat at zero)
```

**B's TTFT no longer improves.** That is what closes the argument: the improvement in the first run was the block-cache hit, not warm CUDA kernels, not the OS page cache, not a JIT compile, not the allocator having settled. Always pair a "mechanism on" measurement with a "mechanism off" control when you are building evidence for a deliverable.

### The second experiment: across time, not across the batch

Demonstrate that APC is a *cross-time* mechanism, which the A-then-B test does not distinguish from in-batch reference counting because A and B ran close together.

```bash
# With prefix caching ON:
ask "Explain etcd quorum loss."           # request A
sleep 120                                  # A is long gone; its blocks are FREE
ask "Explain leader election."             # request C
snap
```

C should still hit ~2,000 tokens. **A had finished two minutes earlier and its blocks had been freed** — but freed blocks retain their hashes and sit in the LRU queue as eviction candidates until something needs the space. That is the distinction between reference counting (in-batch, ends when a sharer finishes) and automatic prefix caching (cross-time, ends at eviction) made empirically. If you then flood the server with 300 unrelated long-prompt requests and repeat, C's hit rate should collapse — because the free queue's head advanced past A's blocks and evicted them. That is LRU eviction, observed.

## Practice

Feeds the deliverable at [`../practice/cost-per-token/README.md`](../practice/cost-per-token/README.md). On a **rented GPU** — 1× A10 / L4 / L40S / A100 / H100 is plenty for an 8B model; budget roughly 45 minutes.

### 1. Capture the allocator's own numbers

```bash
pip install "vllm==0.11.0" && vllm --version   # record it in the workbook

vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --enable-prefix-caching --gpu-memory-utilization 0.90 \
  --max-model-len 8192 --port 8000 2>&1 | tee serve.log

grep -E 'KV cache|Maximum concurrency|Chunked prefill' serve.log
curl -s localhost:8000/metrics | grep '^vllm:cache_config_info'
```

**Acceptance:** the `GPU KV cache size` / `Maximum concurrency` line, and the `cache_config_info` series showing the *actual* `block_size`, `gpu_memory_utilization` and `enable_prefix_caching` values in effect — not the ones you think you set.

### 2. The prefix-cache experiment with its control

Run the worked example in full: cold request A, warm request B, counter deltas, server-side TTFT for each; then restart with `--no-enable-prefix-caching` and repeat.

**Acceptance:** a four-row table — {A, B} × {caching on, caching off} — with server-side TTFT and the `prefix_cache_hits_total` delta in each cell. The off-rows must show no TTFT improvement from A to B. Without the control this is not a result.

### 3. Demonstrate block-boundary granularity

Construct three system prompts of exactly 1,600, 1,608 and 1,616 tokens (verify with the model's tokenizer, not a word count). For each, send a cold request then a warm one, and record the hit count.

**Acceptance:** hit counts that are multiples of `block_size` and that do **not** increase between the 1,600 and 1,608 cases. This is §5's "only full blocks are cached" rule, measured, and it is the observation that stops you from promising a product owner that a 20-token shared system prefix will be free.

### 4. Demonstrate cross-time reuse and its eviction

Run the cold → sleep 120 s → warm sequence and confirm the hit persists. Then flood the server with unrelated long-prompt traffic sufficient to cycle the pool (`vllm bench serve --dataset-name random --random-input-len 4096 --num-prompts 500 --max-concurrency 64`), repeat the warm request, and confirm the hit rate collapses.

**Acceptance:** three hit counts — warm-after-sleep, warm-after-flood — plus one sentence naming the mechanism for each (APC retention; LRU eviction from the free-queue head).

### 5. Observe continuous batching directly

```bash
vllm bench serve --model meta-llama/Llama-3.1-8B-Instruct \
  --base-url http://localhost:8000 --dataset-name random \
  --random-input-len 512 --random-output-len 128 --random-range-ratio 0.8 \
  --request-rate inf --max-concurrency 64 --num-prompts 500 \
  --save-result --result-filename cb.json
```

`--random-range-ratio 0.8` introduces output-length variance, which is what static batching cannot handle. While it runs, sample `vllm:num_requests_running` every second.

**Acceptance:** a short time series of `num_requests_running` showing it **holding near its ceiling rather than sawtoothing down to zero and back**. A static batcher would produce a sawtooth (drain to zero, refill); continuous batching produces a plateau. That plateau is the mechanism, visible in one gauge.

**Overall acceptance (deliverable):** a reproducible **prefix-cache TTFT improvement with its control**, the `prefix_cache_hits_total` delta showing the cached token count, the block-granularity observation, the cross-time-and-eviction result, and the `num_requests_running` plateau — written into the cost-per-token workbook as evidence that shared-prefix traffic (system prompts, few-shot templates, RAG contexts) is served materially cheaper. Record the "Maximum concurrency" figure alongside them: both feed `tokens/hr`, hence cost per million tokens.

## Common pitfalls

- **Believing PagedAttention == vLLM.** It is an algorithm (Kwon et al., SOSP '23) that vLLM productionised first and that SGLang, TensorRT-LLM and others now implement in their own forms. *Mechanism-level consequence:* if you think of it as a product feature you will not transfer the reasoning to the next engine, and in an interview it reads as brand familiarity rather than understanding.

- **Assuming a shared prefix is free down to the token.** Hits land on `block_size` boundaries because only full blocks are hashed and cached. A 2,000-token shared prefix at `block_size 16` yields at most 1,984 tokens of hit; a 20-token shared system message yields at most 16, and often 0. *Mechanism:* a partially filled block has no stable identity — its remaining slots will be written by tokens not yet decided — so it cannot be hashed.

- **Conflating reference counting with automatic prefix caching.** Both rest on content-addressed blocks, but the lifetimes differ. Reference counting is *in-batch*: sequences resident together share blocks, and the sharing ends when they finish. APC is *cross-time*: blocks outlive their originating request in an LRU pool for a future request to hit. Getting this wrong makes "why did request C, which started after A and B both finished, still get a cache hit?" unanswerable instead of obvious.

- **Assuming prefix caching is free at every hit rate because it is on by default.** The "near-zero degradation even at 0 % hit rate" figure is a specific V1 engineering result — constant-time eviction, minimal per-request Python object churn — and it is *why* the default flipped. In V0 the same feature carried enough overhead to need an explicit flag. Do not generalise "smart caches default on for free" to other systems.

- **Treating the paper's "<4 % waste" as a deployment guarantee.** It is a benchmark-configuration result. Real deployments pay attention-kernel gather overhead and CPU-side bookkeeping the headline does not surface. The conclusion (massively better than contiguous) holds; the specific percentage is a best case.

- **Tuning `--block-size` early.** It moves internal waste by fractions of a percent (bounded at 15 tokens per sequence at the default 16) while `--max-model-len`, `--kv-cache-dtype` and the model's `n_kv` move the cap by 2–8× each. The one real reason to change it is prefix-hit granularity on short shared prefixes (smaller) or block-table length at 100k+ contexts (larger) — and both warrant a measurement, not a guess.

- **Reading V0-era material on preemption.** `--swap-space` has been removed and the CPU-swap preemption path does not exist in V1. Any discussion of "recompute vs swap modes" as a live tuning choice is describing an engine you are not running.

- **Expecting PagedAttention to help a prefill-heavy workload.** Long prompts with short outputs are compute-bound above the roofline ridge. The allocator is not the constraint and freeing memory will not move throughput. Diagnose with the two-walls test from module 03 lesson 04: throughput flat while HBM still has free capacity means you crossed into the compute-bound regime and the fix flips entirely.

## Self-check

- **Why non-contiguous fixed-size blocks — what does the block table buy over contiguous allocation?**
  **Answer:** Contiguous per-request KV forces reservation to `max_model_len` up front (output length is unknown, a contiguous buffer cannot grow in place, and a naive kernel wants a fixed stride) and forces the scheduler to find a contiguous hole to admit anything. That produces internal fragmentation (early-finishing requests hold dead tails — around 80 % of the reservation for realistic chat length distributions), over-reservation (even long requests hold their maximum from token 1), and external fragmentation (freed variable-size buffers leave holes too small to reuse) — leaving roughly 20–40 % of KV memory holding real state. Fixed-size blocks addressed through a per-sequence block table make every free block interchangeable, so external fragmentation is *structurally impossible*; they grow KV on demand one block at a time, so there is no over-reservation; and they bound internal waste at the last partial block, `≤ block_size − 1 = 15` tokens per sequence **independent of context length**. Admission becomes an integer comparison ("are there enough free blocks anywhere?") rather than a search for a contiguous hole. That reclaimed memory is what physically permits a high `max-num-seqs`, and since decode throughput is roughly linear in resident batch below the ridge point, it is a 3–5× cost-per-token lever on its own.

- **What does block sharing buy for a system prompt used by many concurrent requests, and what is copy-on-write for?**
  **Answer:** Identical prefixes hash identically (the hash chains `parent_block_hash`, the block's token ids, and extra keys such as LoRA id and `cache_salt`), so many sequences point their block tables at the *same physical blocks*, tracked by a per-block `ref_cnt`. The system prompt's KV is stored once regardless of concurrency — for Llama-3.3-70B at bf16, a 2,000-token prompt is 625 MiB stored once versus 305 GiB stored 500 times. Copy-on-write handles divergence: when a sequence must write into a block whose contents are shared (the standard case is a *partial* prefix hit where a block matches for its first `k` tokens), vLLM copies that single block, decrements the original's refcount, and lets the writer mutate its private copy. The cost is one block, not one sequence — which is exactly what paging buys. The same mechanism gives parallel sampling (`n > 1`) and beam search free prompt sharing, and the PagedAttention paper measured 37.6–55.2 % memory savings for beam search on OPT-13B specifically because beam branches share long prefixes.

- **What is continuous batching, how does it differ from static batching, and why does it need PagedAttention?**
  **Answer:** Static batching assembles B requests and runs them to completion together, so the batch is held hostage by its slowest sequence and short replies leave their slots idle for the remainder. Continuous (iteration-level) batching, from Orca (OSDI 2022), re-plans **every forward pass**: finished sequences are retired and their blocks freed immediately, waiting requests are admitted into the freed capacity within the `max_num_seqs` and `max_num_batched_tokens` budgets, and prefill chunks and decode steps mix in one iteration. On the five-request example in §7 that takes slot utilisation from 57 % to 100 %, wall clock down 33 %, and the last request's TTFT down 83 %. It needs PagedAttention because admitting a request mid-flight means finding KV space mid-flight: under contiguous allocation that requires a `max_model_len`-sized contiguous hole to have appeared exactly where the finished request's buffer was, whereas under paging it requires only that enough free blocks exist anywhere. Continuous batching without paging is a scheduler writing cheques the allocator cannot cash.

- **Is PagedAttention vLLM-exclusive, and why does the distinction matter?**
  **Answer:** No. It is an algorithm from the SOSP '23 paper that vLLM was first to productionise; SGLang implements its own paged KV manager plus RadixAttention (a radix tree for prefix sharing that generalises hash-chained blocks to arbitrary shared subtrees), and TensorRT-LLM has paged KV cache support. Essentially every modern engine converged on block-based non-contiguous KV storage, because the alternative is a ~5× cost penalty. The distinction matters because the transferable, hireable knowledge is the paging idea itself — which shows up across the whole serving landscape — rather than one project's flag names. The same is true of continuous batching, which is now table stakes rather than a differentiator; today's engine-to-engine differences are *underneath* the scheduling loop, in how eviction and admission are implemented, how prefixes are hashed and shared, and how chunked prefill interacts with the token budget.

- **What does PagedAttention *not* fix?**
  **Answer:** Four things. (1) Compute-bound workloads — long prompts with short generations sit above the roofline ridge and are limited by tensor throughput, so no allocator changes them; the fix is quantisation, better kernels, or more compute. (2) Intrinsic KV size — fragmentation waste and bytes-per-token are different axes, and `n_kv`, `head_dim`, layer count and KV dtype are set by architecture (07.1) and quantisation (07.7), not by the allocator. (3) Preemption — paging raises the ceiling but does not remove it; oversubscribe and you still evict, and eviction in V1 means `num_computed_tokens = 0` and a full prefill recomputation. (4) Its own costs — the attention kernel gathers through the block table instead of reading a fixed stride, and CPU-side hashing, reference counting and free-queue manipulation compete with scheduling on the same event loop. The "<4 % waste" headline is a benchmark best case. The honest summary: it reclaims the overwhelming majority of previously wasted KV at a small, well-amortised bookkeeping cost, and changes nothing about workloads whose bottleneck was never memory.

- **A request is preempted because the pool ran dry. What exactly happens to it, and how does prefix caching change the cost?**
  **Answer:** Under the default FCFS policy the scheduler evicts the **most recently admitted** running request (`self.running.pop()` — LIFO, which preserves the oldest requests' progress and avoids convoy effects); under `policy = "priority"` it picks the lowest-priority, latest-arriving one. The victim has **all** its blocks freed, `num_computed_tokens` reset to 0, its status set to `PREEMPTED`, and is **prepended** to the waiting queue so it resumes first. Because computed tokens are zeroed, resumption is a full prefill recomputation — a preempted 8k-context request pays a second 8k prefill, which is why preemptions hammer TTFT-like intervals and why a TTFT spike alongside rising `vllm:num_preemptions_total` is one problem, not two. Prefix caching materially softens this: the recompute goes through the normal `get_computed_blocks` path, and the request's just-freed blocks retain their hashes in the LRU queue, so unless they have already been evicted the request recovers most of its prefill from cache. Note also that V1 has no swap path — `--swap-space` was removed and `vllm:num_requests_swapped` / `vllm:cpu_cache_usage_perc` are documented as legacy; recompute is the only mode.

## Connections & what's next

This lesson is the mechanism that moves 07.2's concurrency cap upward, and it reuses module 05's `/metrics` endpoint as the acceptance instrument for both the worked example and the practice. It also underlies 07.5's cost curve directly: the "throughput up 50–100× from batch 1" claim there is continuous batching and PagedAttention's cheap admission working together, and neither alone produces it.

Backward: it closes the diagnosis 07.2 opened, and it explains why 07.1's `--gpu-memory-utilization` matters so much — the residual it sizes is the block pool this lesson spends.

**Next: [07.4 — vLLM in production](04-vllm-in-production.md)** takes the knobs introduced here — `gpu-memory-utilization`, `max-num-seqs`, `max-num-batched-tokens`, `block-size`, `tensor-parallel-size` — and asks the operational question: what happens when you drive them past the edge of what the block pool can hold? That is preemption, which is this same allocator seen from its failure side, and it is where the module's cost story stops being a mechanism and becomes a configuration you own.

## References & further reading

**Primary sources**

1. **Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention," SOSP '23** — https://arxiv.org/abs/2309.06180 (arXiv) / https://dl.acm.org/doi/10.1145/3600006.3613165 (ACM DL) — the originating paper: the 20–40 % KV utilisation measurement for contiguous systems, the block-table design, copy-on-write sharing, 2–4× throughput over FasterTransformer and Orca at matched latency, and the block-sharing savings (≈6.1–9.8 % parallel sampling, ≈37.6–55.2 % beam search on OPT-13B / Alpaca). *(Both arxiv.org and dl.acm.org are blocked by this environment's egress proxy. The mechanism described in §2–§6 is verified against the vLLM source tree rather than the PDF; the paper's headline figures are reported as widely cited and should be re-checked against the original before being quoted in a document that matters.)*
2. **vLLM engine source — `vllm/v1/core/block_pool.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/block_pool.py — `BlockPool`, `KVCacheBlock` with `ref_cnt` and the embedded free-list pointers, `get_cached_block`, `cache_full_blocks`, `touch`, `free_blocks`, `get_new_blocks` and `_maybe_evict_cached_block`. The source for §4's allocation and eviction walkthrough.
3. **vLLM engine source — `vllm/v1/core/kv_cache_utils.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/kv_cache_utils.py — `hash_block_tokens(hash_function, parent_block_hash, curr_block_token_ids, extra_keys)`, `FreeKVCacheBlockQueue` with its documented LRU ordering and reverse-order free semantics, and `get_max_concurrency_for_kv_cache_config`.
4. **vLLM engine source — `vllm/v1/core/sched/scheduler.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/sched/scheduler.py — the `schedule()` comment stating there are no prefill/decode phases; the block-aligned chunk-end logic; `_preempt_request` (frees all blocks, `num_computed_tokens = 0`, prepends to the waiting queue); and the FCFS-versus-priority preemption victim selection quoted in §9.
5. **vLLM — `docs/design/prefix_caching.md`** — https://github.com/vllm-project/vllm/blob/main/docs/design/prefix_caching.md — the hash-chaining diagram, the "we only cache full blocks" rule, the multimodal `extra_keys` example, `cache_salt` for tenant isolation, the `--prefix-caching-hash-algo` options (`sha256` default since v0.11, `sha256_cbor`, `xxhash`, `xxhash_cbor`), the block-allocation and LRU-eviction workflows, and the append-only-block-table rationale for tolerating duplicate cached blocks.
6. **vLLM — `docs/design/metrics.md`** — https://github.com/vllm-project/vllm/blob/main/docs/design/metrics.md — confirms `vllm:num_requests_swapped` and `vllm:cpu_cache_usage_perc` are legacy and that **`--swap-space` has been removed** as the swap preemption path is no longer used in V1. **Correction to earlier versions of this lesson:** V1 does not offer a swap-versus-recompute choice; recompute is the only mode.
7. **vLLM — `docs/configuration/optimization.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/optimization.md — chunked prefill enabled by default whenever possible in V1, the `max_num_batched_tokens` ITL-versus-TTFT tradeoff (smaller ≈2048 favours ITL, larger >8192 favours TTFT), and the preemption remedies.
8. **vLLM engine source — `vllm/config/cache.py`, `vllm/config/scheduler.py`, `vllm/engine/arg_utils.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/cache.py — `DEFAULT_BLOCK_SIZE = 16`, `enable_prefix_caching = True`, `enable_chunked_prefill = True`, `policy = "fcfs"`, and `EngineArgs.get_batch_defaults` showing the device-dependent `max_num_batched_tokens` / `max_num_seqs` defaults quoted in §7.
9. **Yu et al., "Orca: A Distributed Serving System for Transformer-Based Generative Models" (OSDI 2022)** — https://www.usenix.org/conference/osdi22/presentation/yu — iteration-level scheduling and selective batching; 36.9× throughput at matched latency versus FasterTransformer on GPT-3 175B. The origin of what the industry now calls continuous batching.

**Real-world engineering**

10. **Red Hat — "How vLLM accelerates AI inference: 3 enterprise use cases"** — https://www.redhat.com/en/topics/ai/how-vllm-accelerates-ai-inference-3-enterprise-use-cases — Roblox (50 % latency reduction, 4 billion tokens/week to an in-product assistant) and LinkedIn (50+ generative-AI use cases including the Hiring Assistant) as named production adopters. 2025 snapshot; figures will drift.
11. **Anyscale — "Achieve 23x LLM Inference Throughput & Reduce p50 Latency"** — https://www.anyscale.com/blog/continuous-batching-llm-inference — OPT-13B on one 40 GB A100; up to 23× over naive static batching, ~8× over Ray Serve and TGI, ~4× over FasterTransformer, with the static baseline collapsing to 81 tok/s under realistic output-length variance. *(anyscale.com is blocked by this environment's egress proxy; figures are as reported in secondary summaries and should be re-verified against the original before quoting.)*
12. **vLLM — "vLLM V1: A Major Upgrade to vLLM's Core Architecture"** — https://blog.vllm.ai/2025/01/27/v1-alpha-release.html — up to 1.7× the throughput of V0 from CPU-overhead reduction across the stack, and the zero-overhead prefix-caching result (constant-time eviction plus minimised Python object creation giving near-zero degradation even at 0 % hit rate) that made default-on possible. *(blog.vllm.ai is blocked by this environment's egress proxy; the engineering claims are corroborated by the data structures in references 2–3, which is where the constant-time eviction and pre-allocated block objects actually live.)*

**Deeper dives**

13. **"Inside vLLM: Anatomy of a High-Throughput LLM Inference System"** (vLLM Blog, Sept 2025) — https://blog.vllm.ai/2025/09/05/anatomy-of-vllm.html — the vLLM core team's own walkthrough of the V1 scheduler, block manager and prefix-cache implementation. Read this first among the deeper dives once the domain is reachable to you. *(Blocked here; the equivalent ground truth is the source tree in references 2–5, which is strictly more authoritative.)*
14. **Aleksa Gordić — "Inside vLLM: Anatomy of a High-Throughput LLM Inference System"** — https://www.aleksagordic.com/blog/vllm — an independent author's walkthrough of the same material with different framing; a useful second pass if the maintainer post moves too fast.
15. **vLLM — `docs/design/paged_attention.md`** — https://github.com/vllm-project/vllm/blob/main/docs/design/paged_attention.md — the kernel-level walkthrough of the paged attention CUDA kernel (thread-group layout, the block-table dereference, the online-softmax accumulation sketched in §3). Explicitly labelled by vLLM as a historical document based on the original paper, so read it for the kernel shape rather than for current scheduler behaviour.
