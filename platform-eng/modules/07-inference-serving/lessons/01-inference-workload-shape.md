---
lesson: "07.1"
title: "Inference workload shape: from roofline to the KV-cache budget"
module: "07"
concept: "Inference workload shape: from roofline to the KV-cache budget"
status: not-started
est_time: "6h"
prev: null
next: "02-kv-cache-concurrency.md"
artifacts: []
sources: 14
---

# 07.1 · Inference workload shape: from roofline to the KV-cache budget

> **Concept.** Prefill is compute-bound and decode is memory-bandwidth-bound, so the whole serving problem reduces to managing a VRAM budget — `weights + KV cache + activations` — to hit a latency SLO at minimum cost per token.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

This is the first lesson of module 07, and its job is to fuse two things you already carry — module 03's hardware physics and module 05's SLO vocabulary — into the single equation everything downstream in this module operates on: the HBM budget, and the residual inside it that becomes the KV cache.

Module 03 lesson 04 gave you the two-term decode model and showed that a decode step's cost is dominated by streaming weights out of HBM. Module 05 lesson 06 gave you the split latencies (TTFT, ITL, TPOT), the metric names that actually exist on a current vLLM build, and the roofline derivation showing decode's arithmetic intensity is roughly two orders of magnitude below the machine's ridge point. Neither told you *why the KV cache is the object worth obsessing over* — they treated it as a term that happens to grow. This lesson makes that case with arithmetic: the KV cache is the **residual** of a fixed budget, it is the only term that scales with concurrency, and therefore it — not FLOPs, not weights — is what sets the number of requests you can run at once and hence the denominator of your cost per token.

What it unlocks: 07.2 takes the residual this lesson names and turns it into a hard concurrency ceiling, then shows why a naive allocator would strand most of it. 07.3 is the allocator that reclaims the stranded part. 07.4 is the flag set that sizes the residual in production. 07.5 turns all of it into dollars.

Everything version-specific below is checked against the **vLLM main branch as cloned on 2026-08-17 (commit `c1e4387`)** — specifically `vllm/config/cache.py`, `vllm/config/scheduler.py`, `vllm/engine/arg_utils.py`, and `vllm/v1/core/kv_cache_utils.py`. Where a default differs between main and the `0.11.x` line this module pins, both are stated.

## Why this matters

In a serving-system interview at a GPU-heavy shop, the fork in the road is whether you can connect hardware behaviour to a dollar figure without hand-waving in the middle. "Decode is memory-bound" is table stakes; you had it in module 03. The senior move is the chain: *therefore* one weight read must be amortised across many sequences, *therefore* many sequences must be simultaneously resident, *therefore* resident-sequence count is capped by free HBM divided by per-sequence KV footprint, *therefore* cost per million tokens is largely decided by a memory-allocation decision you make at process start, before a single scheduler flag is touched.

Getting the budget wrong has two distinct failure signatures and both are expensive. Over-provision the KV residual and vLLM OOMs — at startup during CUDA-graph capture, or, worse, mid-request under a burst of long prefills, which is a crash-and-restart outage rather than a slow response. Under-provision it and nothing breaks: you simply run at a fraction of the throughput the card can deliver, pay the same hourly rate, and quietly hold a 2–3× worse cost per token than a correctly sized peer. The second failure is the common one precisely because it is silent.

The framing is not academic. CoreWeave's public GPU-selection guidance for inference customers opens on exactly this budget — model size and context window fix VRAM, concurrency and batching fix throughput, latency SLOs fix the tail — and only then reaches for a spec sheet. If a fleet operator's *customer-facing* sizing guidance starts here, its internal interview loop does too.

## What's new here (calibration)

- **Already yours (referenced, not re-derived):** prefill vs decode as phases; the roofline and ridge point; why decode is memory-bandwidth-bound; the two-term decode step model `t_step(B) = max(memory_term, compute_term)`; KV cache as a concept; FP8 basics (module 03). TTFT / ITL / TPOT definitions, the vLLM `/metrics` endpoint, current metric names, and why total request latency is not an SLO (module 05).
- **New: the budget as an object you allocate**, not a background fact. Three terms, each derivable to within a few hundred MB, with the third defined as whatever the first two leave behind.
- **New: what "activations and overhead" actually decomposes into** — CUDA context, framework allocator, CUDA-graph replay buffers, peak activation working set — and how vLLM *measures* it rather than guessing, which is what `--gpu-memory-utilization` really controls.
- **New: the governing thesis of the module**, stated as an equation you can compute against rather than a slogan: you manage the KV residual to a latency SLO at minimum cost per token.
- **New: model architecture as a serving-cost input.** `num_key_value_heads` is a line item in your GPU bill. A platform engineer should have an opinion on it before a model ships.

## Core concepts

### 1. One request, drawn once, with the physics annotated

Every argument in this module is downstream of this picture. Draw it properly once and the rest stops being vocabulary.

A request arrives with a prompt of `P` tokens and generates `M` output tokens. The server does two structurally different things:

- **Prefill** runs the whole prompt through the network in one (or a few chunked) forward passes, producing the key and value tensors for all `P` positions and one output token.
- **Decode** runs the network once per output token. Each step reads *all* the weights and *all* the KV accumulated so far, produces one token, and appends that token's K and V to the cache.

```
  ONE REQUEST ON ONE GPU — the two regimes, annotated with arithmetic intensity
  ══════════════════════════════════════════════════════════════════════════════
  Llama-3.1-8B, bf16 weights (16 GB), one H100 SXM (3.35 TB/s HBM, ~990 TFLOP/s
  dense bf16 → ridge point ≈ 296 FLOP/byte).  Prompt P = 2,048.  Output M = 512.

   t=0            ~11 ms                                             ~2.5 s
   │◀── PREFILL ──▶│◀──────────────── DECODE (512 steps) ──────────────────▶│
   │  2,048 tokens │   one token per ~4.9 ms, batch 1, forever
   │  in ONE pass  │

  WORK PER PASS
   prefill:  FLOPs = 2·P·N = 2·2048·8e9  = 3.28e13     bytes = 16 GB weights
             AI = 3.28e13 / 1.6e10       = 2,048 FLOP/byte   ← T tokens in, AI = T
   decode:   FLOPs = 2·1·N = 2·1·8e9     = 1.6e10      bytes = 16 GB weights
             AI = 1.6e10 / 1.6e10        = 1 FLOP/byte       ← B=1 token, AI = 1

  ARITHMETIC INTENSITY (log scale)
   2048 ┤███████████████│
        │ compute-bound │
    296 ┤─ ─ ─ ─ ─ ─ ─ ─┼─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  H100 ridge
        │               │
     10 ┤               │
      1 ┤               └██████████████████████████████████████████████
        │                 memory-bound, flat at 1 FLOP/byte forever
      0 └──────────────────────────────────────────────────────────────▶ t

  TENSOR PIPES BUSY
   100% ┤███████████████│
        │               └░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ~0.3 %
     0% └──────────────────────────────────────────────────────────────▶ t

  HBM BANDWIDTH USED
   100% ┤               ┌██████████████████████████████████████████████ saturated
        │░░░░░░░░░░░░░░░┘
     5% └──────────────────────────────────────────────────────────────▶ t

  KV CACHE HELD BY THIS ONE SEQUENCE  (+128 KiB per token, never shrinks
                                       until the request completes)
  320 MiB┤                                                    ╭──────────
         │                                          ╭─────────╯
  288 MiB┤                                ╭─────────╯
         │                      ╭─────────╯
  256 MiB┤            ┌─────────╯
         │            │ ← 2,048 prompt tokens × 128 KiB = 256 MiB in ONE shot
       0 └────────────┘─────────────────────────────────────────────────▶ t

  ┌────────────────────────────────────────────────────────────────────────┐
  │ WHAT THIS PICTURE STATES                                               │
  │ 1. The same silicon is compute-saturated with idle memory for 11 ms,   │
  │    then memory-saturated with idle tensor pipes for 2.5 seconds.       │
  │ 2. Batch-1 decode reaches ~1/296 of the card's peak FLOP/s. That is    │
  │    not a tuning failure; it is what AI = 1 against a ridge of 296      │
  │    means. Only more tokens per pass moves it.                          │
  │ 3. The KV bar is the only thing on this page that grows with           │
  │    CONCURRENCY. Weights are paid once for the whole server. Add a      │
  │    second sequence and you add a second KV bar — nothing else.         │
  └────────────────────────────────────────────────────────────────────────┘
```

Point 3 is the lesson. Everything else on the diagram is a fixed cost of running the model at all. The KV bar is the *variable* cost of running one more user, and it is denominated in HBM.

### 2. Why intensity equals tokens-per-pass — the four-line derivation

You met this in 03 and again in 05.6; it is reproduced here because the whole budget argument hangs off it and you should be able to write it from memory on a whiteboard.

Take a dense transformer with `N` parameters stored at `e` bytes each. For a forward pass over `T` tokens:

```
  FLOPs  ≈ 2 · N · T        two flops (one multiply, one add) per parameter
                            per token — that is what a matmul does
  Bytes  ≈ e · N            every weight must be read from HBM exactly once,
                            regardless of T, because every layer is used and
                            no weight is reused inside a single pass

           2 · N · T FLOPs        2T
  AI(T) =  ──────────────── =  ──────  FLOP/byte
              e · N bytes          e
```

At `e = 2` (bf16) that is simply `AI = T`. **The arithmetic intensity of a forward pass is the number of tokens in it.** Everything else follows:

- Prefill of a 2,048-token prompt: `T = 2048` → `AI ≈ 2048`.
- Decode at batch size `B`: each step emits one token per sequence, so `T = B` → `AI ≈ B`. At batch 1, `AI ≈ 1`.

Now the hardware side. A GPU's **ridge point** — the arithmetic intensity at which compute and memory are equally busy — is `peak FLOP/s ÷ peak HBM bytes/s`:

| GPU | Dense BF16 tensor | HBM bandwidth | Ridge point (BF16) | HBM capacity |
|---|---|---|---|---|
| A100 80GB SXM | ~312 TFLOP/s | 2.04 TB/s | ≈ 153 FLOP/byte | 80 GB |
| L40S | ~181 TFLOP/s | 0.864 TB/s | ≈ 210 FLOP/byte | 48 GB |
| H100 SXM | ~990 TFLOP/s | 3.35 TB/s | ≈ 296 FLOP/byte | 80 GB |
| H200 SXM | ~990 TFLOP/s | 4.8 TB/s | ≈ 206 FLOP/byte | 141 GB |

*(Dense figures. NVIDIA's marketing numbers — 624 / 362 / 1,979 TFLOPS — are quoted **with structured sparsity**, which LLM inference does not use, so halve them. If a source treats those as dense, every ridge point above doubles and none of the conclusions change, because decode's intensity is off by two orders of magnitude either way.)*

Put wall-clock on it for Llama-3.1-8B in bf16 (`N = 8.03×10⁹`, weights = 16 GB) on one H100:

```
  weight read per pass  = 16e9 B ÷ 3.35e12 B/s          = 4.78 ms   ← the floor
  arithmetic at B = 1   = 2 × 8e9 × 1 ÷ 990e12 FLOP/s   = 0.016 ms
                                                          ────────
  ratio                                                    ≈ 300 ×

  So a batch-1 decode step is 4.78 ms of HBM traffic wrapped around 16 µs of
  arithmetic. Ceiling ≈ 1 / 4.78 ms ≈ 209 tok/s for a single stream, and real
  systems land at 60–80 % of that once attention, KV reads, kernel launch and
  sampling are counted.

  Now batch it. Compute scales with B; the weight read does not:
    B =  32 → compute 0.52 ms  vs memory 4.78 ms  → still memory-bound
    B = 128 → compute 2.07 ms  vs memory 4.78 ms  → still memory-bound
    B = 296 → compute 4.78 ms  vs memory 4.78 ms  → RIDGE
```

**Read the middle rows.** Going from B=1 to B=32 costs 0.52 ms of extra compute on a step that already takes 4.78 ms — roughly **11 % more wall-clock for 32× the tokens.** That ratio is the entire economic case for batching, and it is why the rest of this module is about nothing except *what stops you from batching*.

### 3. What stops you: the fixed HBM budget

Batching needs sequences resident. Resident sequences need KV cache. KV cache lives in HBM, and HBM is a hard, fixed, un-negotiable number printed on the card. So:

```
  HBM PARTITION ON ONE GPU AT SERVING TIME
  ════════════════════════════════════════════════════════════════════════════
  Example: Llama-3.1-8B bf16 on one H100-80GB, --gpu-memory-utilization 0.90

  0 GB                                                                   80 GB
  ├──────────────────────────────────────────────────────────────────────────┤
  │██████████████│▓▓▓▓▓▓│░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│▒▒▒▒▒▒│
  │  MODEL       │ NON- │            KV CACHE POOL                     │ HEAD │
  │  WEIGHTS     │ KV   │            (THE RESIDUAL)                    │ ROOM │
  │  16.0 GB     │ WORK │            ≈ 52 GB                           │ 8 GB │
  │              │ ~4GB │                                              │      │
  ├──────────────┴──────┴──────────────────────────────────────────────┴──────┤
  │◀────────── vLLM may touch this: 80 × 0.90 = 72 GB ─────────────────▶│     │
  │                                                                     │     │
  │  FIXED the moment you    MEASURED by     WHATEVER IS LEFT.          RESERVED
  │  choose model + dtype    a profiling     Carved into fixed-size     for
  │                          forward pass    blocks of 16 tokens.       spikes
  │                          at startup                                 & other
  │                                                                     procs

  PROPERTIES THAT MATTER
  ──────────────────────
  · WEIGHTS are paid ONCE for the whole server, no matter how many users.
    They do not scale with concurrency. They scale with model size and dtype.
  · NON-KV WORK scales with the per-step token budget (max_num_batched_tokens)
    and with how many CUDA graphs you captured — not with concurrency directly.
  · KV POOL is the ONLY term that concurrency draws on. Every resident
    sequence rents a slice of it proportional to its current length.
  · HEAD ROOM is not slack you are wasting. It is the buffer that absorbs
    allocator fragmentation, a burst of long prefills, and any other process
    on the card. Spend it and you convert a throughput problem into a crash.

  ⇒ Everything a platform engineer does to inference throughput is one of:
      (a) shrink the first bar   (quantise weights, shard with TP)
      (b) shrink the second bar  (lower max_num_batched_tokens, fewer graphs)
      (c) shrink per-user KV     (GQA/MQA models, FP8 KV, shorter contexts)
      (d) stop wasting the third bar (PagedAttention — lesson 07.3)
```

Written as an equation:

```
  usable  =  HBM_total × gpu_memory_utilization
  KV_pool =  usable − model_weights − non_KV_working_memory
```

That subtraction is the whole game, and each term is computable to within a few hundred megabytes before you rent anything. Take them one at a time.

### 4. Term one: model weights, computed not guessed

`weights_bytes ≈ N_params × bytes_per_param`. The dtype table:

| Storage format | Bytes/param | 8B model | 70B model | 405B model |
|---|---|---|---|---|
| FP32 | 4 | 32 GB | 282 GB | 1.62 TB |
| BF16 / FP16 | 2 | 16 GB | 141 GB | 810 GB |
| FP8 (E4M3) | 1 | 8 GB | 70.5 GB | 405 GB |
| INT4 / NVFP4 (weight-only, with scales) | ~0.55 | ~4.4 GB | ~39 GB | ~223 GB |

The INT4 row is not 0.5 exactly because group-wise quantisation stores a scale (and often a zero-point) per group of typically 128 weights, which adds roughly 8–12 % on top of the 4-bit payload. Quote it as "~0.55 B/param" and check the actual checkpoint size on disk; that is the honest number and it is what `du -sh` on the downloaded weights will tell you.

**Do not take a model's marketing size as its parameter count.** "8B" and "70B" are rounded. Derive the real figure from the config, because a 0.5 GB error propagates straight into your KV residual. For a Llama-style dense decoder with `L` layers, hidden size `H`, intermediate size `I`, vocabulary `V`, `n_q` query heads, `n_kv` KV heads and head dimension `d`:

```
  per layer:
    q_proj   H × (n_q · d)          o_proj   (n_q · d) × H
    k_proj   H × (n_kv · d)         v_proj   H × (n_kv · d)
    gate/up/down  3 × (H × I)
  plus:
    token embedding  V × H     and  lm_head  V × H   (untied in Llama 3)

  Llama-3.1-8B:  L=32, H=4096, I=14336, V=128256, n_q=32, n_kv=8, d=128
    attn/layer = 4096·4096 + 4096·1024 + 4096·1024 + 4096·4096 = 41.9 M
    mlp/layer  = 3 × 4096 × 14336                              = 176.2 M
    per layer                                                  = 218.1 M
    × 32 layers                                                = 6.98 B
    + 2 × 128256 × 4096  (embed + lm_head)                     = 1.05 B
    ────────────────────────────────────────────────────────────────────
    TOTAL                                                      = 8.03 B
    bf16 weights = 8.03e9 × 2 B                                = 16.1 GB

  Llama-3.3-70B: L=80, H=8192, I=28672, V=128256, n_q=64, n_kv=8, d=128
    attn/layer = 8192·8192 + 8192·1024 + 8192·1024 + 8192·8192 = 151.0 M
    mlp/layer  = 3 × 8192 × 28672                              = 704.6 M
    per layer                                                  = 855.6 M
    × 80 layers                                                = 68.4 B
    + 2 × 128256 × 8192                                        =  2.10 B
    ────────────────────────────────────────────────────────────────────
    TOTAL                                                      = 70.5 B
    bf16 weights = 70.5e9 × 2 B                                = 141 GB
```

Notice the asymmetry that the k/v projections are 4× smaller than q/o. That is grouped-query attention showing up in the *weights* as well as the cache — with 8 KV heads instead of 64, `k_proj` and `v_proj` are `8192 × 1024` rather than `8192 × 8192`. §8 returns to this.

### 5. Term two: non-KV working memory, itemised

"Reserve 2–5 GB" is a rule of thumb, and rules of thumb are exactly what this course is trying to replace. Here is what the number decomposes into and why each piece is the size it is.

| Component | Typical size | What determines it |
|---|---|---|
| CUDA context + driver allocations | 0.4–1.0 GB | Per process, before any tensor exists. Grows slightly with driver version and number of contexts. |
| PyTorch caching allocator overhead + framework | 0.5–1.5 GB | Cached-but-unreturned blocks, cuBLAS/cuDNN workspaces, the Python heap of the worker. |
| NCCL / communicator buffers | 0.2–1.5 GB | Only with TP or PP > 1. Grows with world size and with `max_num_batched_tokens` (the all-reduce staging buffers are sized to the largest activation crossing them). |
| Peak activation working set | 0.5–3 GB | The transient tensors of one forward pass. Scales with `max_num_batched_tokens × hidden_size × dtype_bytes` and with the widest intermediate (the MLP's `I` dimension). |
| CUDA-graph replay buffers | 0.5–2 GB | One captured graph per batch-size bucket. Trading memory for lower per-step launch latency. `--enforce-eager` sets this to zero at the cost of several hundred microseconds per step. |

The important operational fact: **vLLM does not guess any of this.** At startup, before allocating the KV pool, it runs a profiling forward pass at the configured maximum batch shape, records peak device memory, subtracts that from `HBM_total × gpu_memory_utilization`, and hands the remainder to the block pool. Then it prints what it decided:

```
INFO ... GPU KV cache size: 425,392 tokens, Maximum concurrency for 8,192 tokens per request: 51.93x
```

That single line (`vllm/v1/core/kv_cache_utils.py`, `update_kv_cache_capacity`) is the most useful thing in the whole startup log, and you should treat it as the acceptance test for every capacity claim in this module. It reports:

- **`GPU KV cache size: N tokens`** — the total number of tokens of KV that the pool can hold *summed across all sequences*, computed as `max_concurrency × max_model_len`.
- **`Maximum concurrency … X.XXx`** — `num_blocks ÷ blocks_per_request_at_max_model_len`. This is how many sequences fit **if every one of them ran to the full `--max-model-len`**. It is a worst-case number, not a promise; with paged allocation, real concurrency at shorter lengths is higher.

**This is what `--gpu-memory-utilization` actually means, and almost everyone gets it wrong.** It is not "how full I want the GPU to be" and it is not a target the server tries to reach. It is a *ceiling on the profiling subtraction*: vLLM computes `budget = total × utilization`, measures what weights and working memory really consumed, and gives the difference to KV. Set it too low and you fence off HBM for nothing. Set it near 1.0 and the residual is computed against a budget with no slack, so the first allocation spike — a burst of long prefills, a graph capture, another process on the card — has nowhere to go and you get `torch.OutOfMemoryError`.

**Defaults, verified against source.** On the vLLM main branch (`vllm/config/cache.py`, commit `c1e4387`, 2026-08-17) `gpu_memory_utilization` defaults to **0.92**. The `0.11.x` line that this module pins defaults to **0.90**. Both are documented as per-instance limits: if two vLLM processes share a card, each needs its own fraction and they do not coordinate. If you take one number away from this section, take the habit of reading the value back off `vllm:cache_config_info` rather than assuming it.

There is also a bypass worth knowing: `--kv-cache-memory-bytes` sets the KV pool size *directly in bytes* and, when set, ignores `gpu_memory_utilization` entirely. It exists so you can skip the profiling run (faster startup) and pin an exact pool size across restarts. Use it once you have measured what the profiler chooses; do not use it to guess.

### 6. Term three: the KV cache, and where its bytes come from

For one token of one sequence, the cache stores a key vector and a value vector, in every layer, for every KV head:

```
  kv_bytes_per_token  =  2          ← one K and one V
                       × L          ← layers, each with its own cache
                       × n_kv       ← KV heads (NOT query heads)
                       × d          ← head dimension
                       × e_kv       ← bytes per element of the KV dtype
```

This is not a heuristic — it is what vLLM's own `AttentionSpec` computes. In `vllm/v1/kv_cache_interface.py` the per-layer page size is `num_kv_heads × block_size × (head_size + head_size_v) × dtype_size`, and the pool allocates one such page per layer per block, so per token per layer you get `n_kv × (d + d_v) × e_kv`, which for the usual case `d_v = d` is `2 · n_kv · d · e_kv`. Multiply by `L` and you have the formula above.

Real numbers, bf16 KV:

```
  Llama-3.1-8B   L=32, n_kv=8,  d=128, e=2  →  2·32·8·128·2  = 131,072 B  = 128 KiB/token
  Llama-3.3-70B  L=80, n_kv=8,  d=128, e=2  →  2·80·8·128·2  = 327,680 B  = 320 KiB/token
```

Two consequences to sit with:

**(a) The per-user variable cost is denominated in tokens of context.** An 8B model at 8k context costs `8192 × 128 KiB = 1.0 GiB` of HBM per resident sequence. A 70B at 8k costs `8192 × 320 KiB = 2.5 GiB`. That is the rent for one seat, and it is charged for as long as the request is alive — including while it sits there generating token 1,900 of 2,000.

**(b) `n_kv` is a multiplier on your GPU bill.** Classic multi-head attention sets `n_kv = n_q`. Llama-3.3-70B has 64 query heads; had it shipped with MHA, per-token KV would be `2·80·64·128·2 = 2.62 MB` instead of 320 KiB — **8× worse**, turning that 8k seat from 2.5 GiB into 20 GiB. Same parameter count, same quality class, eight times the serving cost at long context. 07.2 works the full counterfactual; hold the shape here.

The dtype term is a lever too, and an independent one. `--kv-cache-dtype fp8` stores K and V at one byte instead of two, halving `kv_bytes_per_token` and therefore doubling the token capacity of the same pool — with a small accuracy cost you validate separately (07.7). Note it is genuinely independent of weight precision: you can serve bf16 weights with an FP8 cache, or FP8 weights with a bf16 cache. The vLLM `CacheDType` enum on main lists `auto, float16, bfloat16, fp8, fp8_e4m3, fp8_e5m2` plus a set of newer per-token-head and NVFP4 variants; `fp8` is an alias for `fp8_e4m3`.

### 7. The chain from residual to dollars

Now assemble it. This is the governing thesis of module 07, and it is five substitutions long.

```
  ① usable HBM        =  HBM_total × gpu_memory_utilization
  ② KV pool           =  ① − weights − non_KV_working_memory
  ③ concurrency cap   =  ② ÷ (kv_bytes_per_token × avg_live_context)
  ④ throughput        ≈  min(③, max_num_seqs) × per_sequence_decode_rate
  ⑤ cost per M tokens =  (GPUs × $/GPU-hr) ÷ (④ × 3600) × 1e6
```

Every term in ③ and ④ is something you control, and the pipeline is multiplicative, so a 2× error anywhere is a 2× error in ⑤.

Anchor it with a number you can carry:

```
  One H100 at $2.89/hr (RunPod Secure Cloud on-demand, 2026 snapshot —
  the deliverable's reference rate; verify yours, the spread across
  providers in 2026 is roughly $1.99–$6.16 per GPU-hour)

  sustaining 2,500 output tok/s:
      2.89 ÷ (2500 × 3600) × 1e6  =  $0.321 per 1M output tokens

  Now double the average context length. Per-sequence KV doubles, ③ halves,
  and because decode throughput is essentially linear in resident batch
  well below the ridge, ④ roughly halves too:

      2.89 ÷ (1250 × 3600) × 1e6  =  $0.642 per 1M output tokens

  The GPU bill did not move. Your KV allocation did. 2× the cost per token
  from one decision about context length.
```

**And the honesty term.** Step ⑤ silently assumes the GPU sustains that throughput for the whole hour. It does not. A real fleet runs at 30–50 % duty cycle, and the effective cost is `$0.321 ÷ 0.40 = $0.80/M`. Lesson 07.5 makes utilisation a first-class factor in the equation; for now, note that any CPM you quote without a utilisation figure is a benchmark number wearing a production costume.

### 8. Why prefill and decode fight, and why you serve them together anyway

Prefill and decode want opposite things from the same silicon in the same process:

| | Prefill | Decode |
|---|---|---|
| Tokens per forward pass | the whole prompt (hundreds–thousands) | one per resident sequence |
| Arithmetic intensity | `≈ P`, far above the ridge | `≈ B`, far below it until B is in the hundreds |
| Bottleneck | tensor pipes | HBM bandwidth |
| Latency it sets | **TTFT** | **ITL / TPOT** |
| What batching buys | little — already compute-saturated | almost everything |
| KV effect | *writes* `P` tokens of KV in one shot | *reads* all of it, appends one token |

Running them in the same iteration is not a compromise, it is a free lunch — the memory-bound decode work rides in the shadow of the compute-bound prefill work, and vice versa. That is exactly why vLLM's V1 scheduler mixes them by default rather than alternating phases. The scheduler source says so in its own words (`vllm/v1/core/sched/scheduler.py`):

> *"There's no 'decoding phase' nor 'prefill phase' in the scheduler. Each request just has the `num_computed_tokens` and `num_tokens_with_spec` … At each step, the scheduler tries to assign tokens to the requests so that each request's `num_computed_tokens` can catch up its `num_tokens_with_spec`."*

The free lunch has a bill attached, and it is a latency bill. An unchunked 8,192-token prefill turns one ~5 ms decode iteration into one ~40 ms iteration, so *every* streaming user in that batch sees a 40 ms gap instead of 5 ms. **Chunked prefill** — slicing the prompt into pieces that fit inside the per-iteration token budget — is the fix, and it is on by default in V1 (`enable_chunked_prefill: bool = True` in `vllm/config/scheduler.py`). 07.3 and 07.4 cover the mechanism and the dial; hold the shape: **one replica, two workloads with opposite bottlenecks, drawing on one KV pool and one per-iteration token budget.**

The practical consequence for this lesson's budget: prompt-heavy workloads (RAG, long-document summarisation, agent traces) are expensive twice over. They write a lot of KV per request *and* they consume a lot of the per-iteration token budget, which is what `max_num_batched_tokens` caps. A workload with 4,000-token prompts and 200-token outputs has a completely different budget shape from one with 500-token prompts and 2,000-token outputs, even at identical request rates.

### 9. When the weights do not fit at all

If `weights_bytes > HBM_total`, the residual is negative and there is nothing to tune. Llama-3.3-70B at bf16 is 141 GB against an H100's 80 GB: you are 61 GB underwater before a single KV byte. Two escape hatches, both from module 03, with the arithmetic that decides between them.

**Tensor parallelism (`--tensor-parallel-size N`).** Every layer's weight matrices are sharded across N GPUs; the GPUs execute each token collectively, exchanging partial activations through an all-reduce at (roughly) two points per layer. Per-GPU weights drop to `weights/N` and per-GPU KV drops to `kv/N` as well, because the KV heads are sharded along with the attention heads. What you pay: an all-reduce of `batch × hidden × 2 B` per collective, twice per layer, per step. On an NVLink-connected node (900 GB/s per GPU on H100) that is a few hundred microseconds per step and worth it. Over PCIe it is not. Set `TP ≤ GPUs per NVLink domain`.

**Weight quantisation.** FP8 halves `bytes_per_param` on Hopper and later, where the tensor cores execute FP8 natively so you also get a throughput win rather than a dequantisation tax. INT4/AWQ quarters it but is usually weight-only, meaning the GEMM dequantises to bf16 before multiplying, so you get the memory win without the compute win. Quality validation is a separate exercise (07.7) and is not optional.

You will almost always need one for 70B-class models on 80 GB cards, and usually both. The worked example computes exactly why.

### 10. Model architecture as a serving-cost lever

The budget above treats `N_params` and `kv_bytes_per_token` as facts handed to you. They are not — they are decisions a modelling team made, and the biggest one for serving cost is `num_key_value_heads`.

**Multi-head attention (MHA)** gives each of `n_q` query heads its own K and V projections. **Multi-query attention (MQA)** collapses to a single shared KV head. **Grouped-query attention (GQA)** is the interpolation: `n_kv` groups, each shared by `n_q / n_kv` query heads. The GQA paper (Ainslie et al., 2023) showed you can *uptrain* an existing MHA checkpoint into a GQA one using about 5 % of the original pre-training compute and land close to MHA quality with MQA-like speed. That result is why essentially every model shipped since mid-2023 uses GQA — it is a pure serving-cost win that costs almost nothing to obtain.

The numbers you should be able to state in a design review:

| Variant | `n_kv` for a 70B-class model | KV bytes/token (bf16) | HBM for one 8k seat |
|---|---|---|---|
| MHA | 64 | 2.62 MB | 20.5 GiB |
| GQA-8 (Llama 3.x) | 8 | 320 KiB | 2.5 GiB |
| MQA | 1 | 40 KiB | 0.31 GiB |

And a fourth option that is architecturally different: **multi-head latent attention (MLA)**, used by the DeepSeek-V2/V3 line, compresses K and V into a single low-rank latent vector per token per layer and reconstructs the per-head K/V on the fly. That trades a little extra compute per step for a KV footprint far below even MQA. It is not a knob you turn — it is baked into the checkpoint — but it is the reason DeepSeek-class models serve long contexts at concurrency that a same-size GQA model cannot match.

The senior version of this skill is not "I can compute the concurrency cap." It is being able to say, in the room where a model is being chosen: *"this one is `num_key_value_heads: 8`, not 64, which is why we can afford to serve it at 32k context; the other candidate scores two points higher and costs us eight times the KV per seat, so at our target QPS it is the more expensive model by a wide margin."* Model quality is not the only axis; you own the other one.

### 11. What the budget does not capture

Be precise about the limits of a napkin, so you know when to stop trusting it:

- **Block granularity.** vLLM allocates KV in blocks of `block_size` tokens (default **16**, `CacheConfig.DEFAULT_BLOCK_SIZE`), so each sequence rounds up to a whole block. At 16 tokens per block and thousands of tokens per sequence this is under 1 %; at very short sequences it is not.
- **Non-uniform layers.** Sliding-window layers, cross-layer KV sharing, and hybrid Mamba/attention stacks all break the "every layer caches every token" assumption. vLLM models these with separate KV-cache *groups*, and the concurrency figure it prints sums the per-group block requirement (`get_max_concurrency_for_kv_cache_config`). If your model is hybrid, trust the printed line, not your formula.
- **Speculative decoding** changes the accounting: draft tokens occupy slots that may be rejected, so the effective tokens-per-step is a function of acceptance rate.
- **The profiling pass measures peak, not steady state.** A model whose activation peak occurs at an unusual shape can profile higher than it ever uses, quietly costing you KV pool. Compare the printed KV size against your hand calculation; a big gap is worth investigating.

## Perspectives

**The fleet operator / SKU-selection view.** From CoreWeave's side of the table this budget *is* the sizing conversation: model size and context window set VRAM, concurrency and batching set throughput, latency SLOs set the tail behaviour they must protect, and only then does H100-vs-H200-vs-B200 become a spec-sheet question. Note which term the SKU actually moves. Going H100 (80 GB) → H200 (141 GB) does not increase capacity by 76 % for serving purposes; because weights are a *fixed subtraction*, a 70B FP8 model's free-for-KV space goes from roughly 2 GB to roughly 67 GB. The headroom grows far faster than the capacity, and that non-linearity is the single most under-appreciated number in inference SKU selection.

**The cost / FinOps view.** Character.AI reports cutting serving cost by at least 33× since late 2022 while serving around 20,000 queries per second at under a cent per hour of conversation. That is not one trick; it is compounding — MQA, hybrid attention horizons (most layers attend only to a local window), cross-layer KV sharing, inter-turn caching, and int8 throughout. Their headline architectural claim is a **>20× reduction in KV cache size without quality regression**, which is precisely term three of this lesson's budget being attacked at the model level rather than the serving level. Contrast that with this lesson's single-GPU worked example, where one config decision (TP degree vs FP8) already moves cost per token by 2× on unchanged hardware. Stack a handful of such decisions across a fleet for two years and 33× stops looking exotic.

**The benchmark-vs-production view.** Anyscale's widely-cited 23× continuous-batching result was measured on OPT-13B on a single 40 GB A100, against naive static batching, with a realistic *variance* in output lengths — and the variance is the point: with fixed-length outputs static batching is fine, and it is the mixed-length distribution that collapses it (their naive static baseline fell to 81 tok/s once length variance rose). That is a throughput multiplier under benchmark conditions, not a cost multiplier in production, because real fleets run below peak batch to protect tail latency and pay for spike headroom. Module 05's utilisation-vs-SLO reasoning is exactly what closes that gap; carry the scepticism into 07.5.

**The architecture-as-cost-control view.** Most platform engineers treat the model as a black box handed down by research. The senior version treats `num_key_value_heads` the way you would treat a missing index on a hot query path: a decision made upstream that you pay for downstream, forever, and therefore one you have standing to review. If two candidates have comparable quality and one is GQA-8 while the other is MHA-64, that is an 8× difference in KV footprint per token — roughly an 8× difference in achievable concurrency at fixed context — before a single serving-layer optimisation runs.

## Real-world use cases

- **Character.AI — "Optimizing AI Inference at Character.AI."** They serve ~20,000 queries/second at under a cent per hour of conversation, and report ≥33× serving-cost reduction since 2022. The mechanism is explicitly this lesson's term three: multi-query attention (`n_kv = 1`), **hybrid attention horizons** (only one in every six layers uses global attention; the rest use a sliding window, so most layers cache only a bounded recent span rather than the whole history), and **cross-layer KV sharing** (adjacent layer groups share one KV tensor). Together those cut KV cache size by more than 20× with no reported quality regression, and they enable an inter-turn cache with a reported >95 % hit rate on their traffic. **What it shows:** the biggest available lever on serving cost is often not in the serving stack at all — it is in the attention design, and it is worth having an opinion on before the model ships.

- **CoreWeave — "Choosing the Right NVIDIA GPU for Running Inference."** A neocloud's own customer-facing sizing guidance, which walks the same weights + KV + overhead budget as a GPU-selection decision for 70B-class deployments before it reaches any spec sheet. **What it shows:** this is not a pedagogical framing invented for a course; it is how the people who sell GPU-hours reason about what you need, which means it is also how they interview for the role that reasons about it.

- **Anyscale — "Achieve 23× LLM Inference Throughput & Reduce p50 Latency."** OPT-13B on one 40 GB A100. Continuous batching plus batching-aware memory management reached up to 23× the throughput of naive static batching; roughly 8× over Ray Serve and HuggingFace TGI's implementations and ~4× over FasterTransformer's. The naive static baseline fell to 81 tok/s under realistic output-length variance. **What it shows:** the quantified size of the prize that the KV budget is guarding — and the fact that the number is heavily dependent on the length distribution of the traffic, which is why "23×" should never be quoted without the workload.

- **vLLM engine source — `vllm/v1/core/kv_cache_utils.py` and `vllm/config/cache.py`.** The startup line `GPU KV cache size: N tokens, Maximum concurrency for M tokens per request: X.XXx` is emitted by `update_kv_cache_capacity`, and the concurrency figure is literally `num_blocks ÷ Σ_groups ceil(per_request_bytes ÷ page_bytes)`. `gpu_memory_utilization` defaults to 0.92 on main (0.90 in the 0.11.x line), `block_size` to 16, `enable_prefix_caching` to True. **What it shows:** every number in this lesson's budget has a corresponding line in the engine that will tell you whether you got it right, and reading it back is a thirty-second acceptance test on any capacity claim.

## Worked example

**Question: does Llama-3.3-70B fit on one H100-80GB, and what is left for KV under each escape hatch?**

This is the interview question the module README names, worked to a defensible answer.

**Step 1 — the impossible baseline.**

```
  weights (bf16)          = 70.5e9 × 2 B                   = 141.0 GB
  HBM_total (H100 SXM)                                     =  80.0 GB
  usable at util 0.90     = 80 × 0.90                      =  72.0 GB
  ───────────────────────────────────────────────────────────────────
  residual for KV + work  = 72.0 − 141.0                   = −69.0 GB
```

Negative before a single KV byte. This is arithmetic, not a tuning problem, and no flag makes it positive. Say that out loud first — a candidate who starts tuning here has failed the question.

**Step 2 — enumerate the escape hatches and compute each.** Assume 3 GB of non-KV working memory per GPU at TP=1, rising to ~4 GB at TP=2 and ~5 GB at TP=4 (NCCL buffers grow with world size).

| Config | GPUs | Weights/GPU | Usable/GPU @0.90 | Non-KV | **KV residual/GPU** | Total KV pool | Verdict |
|---|---|---|---|---|---|---|---|
| bf16, TP=1 | 1 | 141 GB | 72 GB | — | **−69 GB** | — | Impossible |
| bf16, TP=2 | 2 | 70.5 GB | 72 GB | 4 GB | **−2.5 GB** | — | Still does not fit |
| bf16, TP=4 | 4 | 35.3 GB | 72 GB | 5 GB | **31.7 GB** | 127 GB | Works, 4 GPUs |
| FP8, TP=1 | 1 | 70.5 GB | 72 GB | 3 GB | **−1.5 GB** | — | Bare miss |
| FP8, TP=2 | 2 | 35.3 GB | 72 GB | 4 GB | **32.7 GB** | 65 GB | Works, 2 GPUs |
| FP8, TP=4 | 4 | 17.6 GB | 72 GB | 5 GB | **49.4 GB** | 198 GB | Works, most headroom |

**Two rows deserve a second look.** bf16/TP=2 and FP8/TP=1 both *look* like they should work — 70.5 GB of weights against 72 GB of budget — and both fail, because the 3–4 GB of working memory is not optional. This is the single most common sizing error: computing `usable − weights` and forgetting term two. A 1.5 GB miss reads as "it nearly fits, bump utilisation to 0.95" — which buys 4 GB, appears to work at startup, and OOMs on the first burst of long prefills. **Do not solve a negative residual with `--gpu-memory-utilization`.**

**Step 3 — turn residual into concurrency.**

```
  kv_bytes_per_token (bf16 KV, 70B) = 2 × 80 × 8 × 128 × 2  = 327,680 B = 320 KiB

  With TP=N, KV is sharded too, so each GPU holds 1/N of each sequence's KV.
  The pool that matters is the TOTAL across the TP group.

  FP8 weights, TP=2, bf16 KV:
     total KV pool          = 2 × 32.7 GB                = 65.4 GB
     per-seat cost at 8k    = 8192 × 327,680 B           =  2.68 GB
     ⇒ concurrency at 8k    = 65.4 ÷ 2.68                ≈ 24 sequences

  Same config with --kv-cache-dtype fp8 (KV at 1 B/element):
     kv_bytes_per_token                                  = 163,840 B = 160 KiB
     per-seat cost at 8k                                 =  1.34 GB
     ⇒ concurrency at 8k                                 ≈ 48 sequences

  Same config, bf16 KV, but max-model-len cut from 8k to 4k:
     per-seat cost at 4k                                 =  1.34 GB
     ⇒ concurrency at 4k                                 ≈ 48 sequences
```

Read the last two blocks together: **quantising the KV cache and halving the context length buy you exactly the same thing.** Both halve `kv_bytes_per_token × context`, both double concurrency, and they compose — FP8 KV at 4k context gives ~97 seats on the same two GPUs. One is a numerics decision you validate; the other is a product decision about what context you actually need to support. Neither is a hardware purchase.

**Step 4 — price it.**

```
  Assume the FP8/TP=2/bf16-KV config sustains 3,200 output tok/s at 24 resident
  sequences (a plausible figure for a 70B FP8 on 2×H100; MEASURE yours — this
  is the number the deliverable exists to establish, not to assume).

  cost = (2 GPUs × $2.89/hr) ÷ (3,200 × 3600) × 1e6  =  $0.502 per 1M tokens

  Now the bf16/TP=4 config. Twice the GPUs, and roughly the same throughput,
  because throughput is set by resident batch and bandwidth, not by GPU count
  once the model fits:

  cost = (4 GPUs × $2.89/hr) ÷ (3,400 × 3600) × 1e6  =  $0.944 per 1M tokens
```

**The sentence to be able to say in an interview:** *"141 > 80, so one H100 is off the table before any tuning. The real decision is TP degree versus weight precision, and I pick it by how much KV residual each option leaves at my target context length. FP8 on TP=2 and bf16 on TP=4 deliver comparable KV headroom, but one uses half the GPUs — so if FP8 quality holds on my eval set, that is a ~2× cost-per-token win, and validating that eval is the gating work, not the config."*

You compute the concurrency each residual buys explicitly in [07.2](02-kv-cache-concurrency.md), measure it in 07.4, and turn it into a curve in 07.5.

## Practice

Reading plus napkin calculation. **No GPU required for this lesson** — you start renting in 07.2 — so this is deliberately cheap, and its output is the prediction row that every later measurement in this module validates against.

### 1. Rebuild the budget from parameters, not from marketing sizes

For each of the three configs below, produce a five-column row: `weights`, `usable HBM`, `non-KV working memory`, `KV residual`, `concurrency at 8k context`. Derive the parameter count from the config geometry (§4) rather than trusting the model's name.

- Llama-3.1-8B bf16 on 1×H100-80GB
- Llama-3.3-70B bf16 on 4×H100-80GB
- Llama-3.3-70B FP8 on 2×H100-80GB

Use `--gpu-memory-utilization 0.90`, non-KV working memory of 3 GB at TP=1 / 4 GB at TP=2 / 5 GB at TP=4, and bf16 KV.

**Acceptance:** a three-row table. The 8B row should show a KV residual of roughly 52–53 GB and a concurrency around 50 at 8k; if yours is wildly different, the error is almost always in `n_kv` (8, not 32).

### 2. Sensitivity, one variable at a time

Take the 8B row and recompute concurrency under each single change, holding everything else fixed:

| Change | New concurrency | Ratio |
|---|---|---|
| `--max-model-len` 8192 → 4096 | | |
| `--kv-cache-dtype fp8` | | |
| `--gpu-memory-utilization` 0.90 → 0.95 | | |
| Model switched to a hypothetical MHA-32 variant (`n_kv` 8 → 32) | | |

**Acceptance:** four ratios, and one sentence identifying which lever is *not* like the others. (It is the third: raising utilisation buys `80 × 0.05 = 4 GB` — about 8 % more KV — while spending the headroom that keeps you out of an OOM. The first two roughly double concurrency. The fourth quarters it.)

### 3. Predict the cheaper 70B config

Using the equation chain in §7 and your own guess at throughput, predict which of `70B-FP8-TP2` and `70B-bf16-TP4` gives lower cost per million tokens, and by how much. Write the prediction down with the assumption you are least sure about flagged explicitly.

**Acceptance:** one sentence with a number and a named assumption. You will measure against this in 07.4–07.5, and a wrong prediction with a correctly identified uncertain assumption is a better outcome than a lucky right one.

### 4. Name the architecture lever

Pick any model you would plausibly deploy. State its `num_hidden_layers`, `num_key_value_heads`, `num_attention_heads`, and `head_dim` from its `config.json`, compute `kv_bytes_per_token`, and write the one-sentence pushback you would give if a modelling team proposed the MHA equivalent for a high-QPS product.

**Acceptance:** the four geometry numbers, the derived bytes/token, and the pushback sentence with the KV-footprint multiplier in it.

**Overall acceptance:** the three-row budget table, the four-row sensitivity table, one cost prediction with a flagged assumption, and one architecture note — committed to the [cost-per-token deliverable's](../practice/cost-per-token/README.md) working notes. These are the predicted values that the measured runs in 07.2 and 07.4 will be checked against.

## Common pitfalls

- **Treating `--gpu-memory-utilization` as "how full I want the GPU."** It is a ceiling on a *subtraction*, not a target. vLLM profiles real peak usage at startup and gives the KV pool whatever is left under that ceiling. Too low starves KV for no reason; too high leaves no room for allocation spikes and converts a recoverable throughput problem into a crash. *Mechanism:* the residual is computed once, at startup, against `HBM_total × utilization`; the runtime has no way to give memory back if it guessed wrong.

- **Forgetting term two and concluding a model "just fits."** `72 − 70.5 = 1.5 GB` is not a KV pool, it is a rounding error, and the 3 GB of CUDA context, allocator overhead and activation peak has to come from somewhere. This is the specific error that produces the "it started fine and OOMed at 2 a.m." incident.

- **Using `num_attention_heads` where the formula wants `num_key_value_heads`.** On a GQA model that inflates your KV estimate by 4–8× and cascades into a wrong concurrency cap and a wrong cost per token. Always read `num_key_value_heads` from `config.json`; it is the only head count that appears in the KV formula.

- **Assuming a bigger GPU fixes a negative residual proportionally.** It does not, and the error runs in the *helpful* direction, which is why people miss it: because weights are a fixed subtraction, H100 → H200 turns a ~2 GB residual into a ~67 GB one for a 70B FP8 model. 76 % more HBM, 30× more KV. Compute the residual, not the capacity ratio.

- **Believing prefill batching is "free throughput" the way decode batching is.** Prefill of a single 2k-token prompt already sits at `AI ≈ 2048` against a ridge of ~296 — the tensor pipes are saturated. Stacking more prefills queues compute rather than unlocking idle bandwidth. Batching prefill fills scheduling bubbles; batching decode escapes a bottleneck. Different mechanisms, different payoffs.

- **Quoting a cost per token without a utilisation figure.** The denominator in §7 assumes the GPU sustains that throughput for the whole hour. At a realistic 35–40 % fleet duty cycle the honest number is 2.5–3× higher. This is the single most common way self-hosted inference gets mis-priced against an API.

## Self-check

**(a) Why does batching help decode enormously and prefill barely at all?**

**Answer:** Because arithmetic intensity is the number of tokens in the forward pass, and the two phases start on opposite sides of the ridge. A forward pass costs `≈ 2NT` FLOPs and reads `≈ eN` bytes of weights, so `AI = 2T/e`, which at bf16 is just `T`. Decode has `T = B`, so batch-1 decode sits at `AI ≈ 1` against a ridge of ~296 on an H100 — roughly 300× below it, meaning the card can reach about 1/300th of its peak FLOP/s no matter what. Adding sequences moves that point rightward almost for free: going B=1 → B=32 for Llama-3.1-8B adds 0.52 ms of compute to a step whose 4.78 ms weight read is paid regardless, so 32× the tokens for ~11 % more wall clock. Prefill of a 2,048-token prompt already sits at `AI ≈ 2048`, ten times *above* the ridge — the tensor pipes are the bottleneck, so stacking more prompts queues compute rather than unlocking idle bandwidth. You batch prefill to fill scheduling bubbles, not to escape a bottleneck.

**(b) Write the `VRAM = weights + KV + activations` budget for Llama-3.3-70B at bf16 on 1×H100-80GB. Does it fit, and what does it force?**

**Answer:** Weights are `70.5e9 × 2 B = 141 GB` against 80 GB of HBM, of which `--gpu-memory-utilization 0.90` makes 72 GB addressable. Residual `= 72 − 141 = −69 GB`: negative before any KV or working memory, so it does not fit, and no flag changes that. It forces tensor parallelism, weight quantisation, or both. TP=2 gives 70.5 GB/GPU against 72 GB usable — *still* fails once you subtract ~4 GB of CUDA context, allocator overhead, NCCL buffers and activation peak. The configurations that actually work are bf16/TP=4 (35.3 GB weights/GPU, ~32 GB KV residual/GPU) or FP8/TP=2 (35.3 GB/GPU, ~33 GB residual/GPU). The second uses half the GPUs for comparable KV headroom, so it is roughly a 2× cost-per-token win conditional on FP8 quality holding on your eval.

**(c) Why is TTFT set by prefill and ITL by decode, and which does a bigger resident batch hurt?**

**Answer:** TTFT is the interval from arrival to the first emitted token, which is queue wait plus the prefill forward pass(es) over the prompt — so it scales with prompt length and with how much of the per-iteration token budget prefill is allowed to take. ITL is the gap between consecutive tokens, which is the duration of one decode iteration — so it scales with resident batch size and with total context in the batch. A bigger resident batch *helps* throughput (the weight read amortises across more tokens) and *hurts* ITL directly, because each iteration now serves more sequences and takes longer. It also hurts TTFT indirectly: with more sequences resident, KV pressure rises, new requests wait longer for admission, and preemptions become likelier. That is the central tension of the module: batch for decode throughput, cap admission to protect the latency SLOs.

**(d) A model team proposes shipping a 70B with classic MHA (64 KV heads) instead of GQA (8) for a high-QPS chat product. What is your pushback, in budget terms?**

**Answer:** KV bytes per token scale linearly with `num_key_value_heads`, so 64 versus 8 is an 8× larger cache footprint per token at identical context. Concretely: `2 × 80 × 64 × 128 × 2 = 2.62 MB/token` versus `320 KiB/token`, which turns one 8k-token seat from 2.5 GiB of HBM into 20.5 GiB. On a fixed KV pool that is 8× fewer concurrent sequences, and because decode throughput is roughly linear in resident batch well below the ridge, roughly 8× fewer tokens/sec per GPU and 8× the cost per million tokens. The GQA paper showed you can uptrain an MHA checkpoint into a GQA one for about 5 % of original pre-training compute at near-MHA quality — so the cheap fix exists and the burden of proof is on the MHA proposal. This is a modelling decision with serving economics baked in before deployment, which is exactly the kind of thing a platform engineer should be in the room to flag.

**(e) You raise `--gpu-memory-utilization` from 0.90 to 0.97 on an 80 GB card and concurrency barely moves. Why?**

**Answer:** Because the increment is small relative to the pool and the pool is not the only thing it feeds. `80 × 0.07 = 5.6 GB` more budget, against a KV pool that was already ~52 GB — about 11 % more KV, so about 11 % more concurrency at best. Meanwhile you have spent essentially all the headroom that absorbs allocator fragmentation, CUDA-graph capture and a burst of concurrent long prefills, so the next spike OOMs the process. The lever is badly asymmetric: small upside, catastrophic downside. If you need materially more concurrency the effective levers are the ones that change the *per-seat* cost — FP8 KV, a shorter `--max-model-len`, a GQA/MQA model — or the ones that change the weights term, not the utilisation fraction.

## Connections & what's next

This lesson is the spine the rest of module 07 hangs on. The KV residual it derives is the input to 07.2's concurrency arithmetic, the resource 07.3's PagedAttention stops wasting, the thing 07.4's flags size in production, and the denominator 07.5 converts into dollars. The `num_key_value_heads`-as-cost-lever thread opened here is picked up numerically in the next lesson and again in 07.7 when quantisation attacks the same term from the dtype side.

**Next: [07.2 — KV cache as a concurrency problem](02-kv-cache-concurrency.md)** takes the residual this lesson calls "whatever is left" and turns it into a hard number — `max_concurrent_requests` — then shows why a naive contiguous allocator would strand 60–80 % of it before you ever get to use it, which is the problem PagedAttention exists to solve in 07.3.

## References & further reading

**Primary sources**

1. **vLLM engine source — `vllm/config/cache.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/cache.py — the authoritative defaults quoted in §5 and §6: `gpu_memory_utilization = 0.92` on main, `DEFAULT_BLOCK_SIZE = 16`, `enable_prefix_caching = True`, the `CacheDType` enum, and `kv_cache_memory_bytes` (which bypasses `gpu_memory_utilization` when set). **Correction to earlier versions of this lesson:** the default is 0.92 on main, not 0.90; 0.90 is the `0.11.x` value this module pins, and both are stated in §5.
2. **vLLM engine source — `vllm/v1/core/kv_cache_utils.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/kv_cache_utils.py — `get_max_concurrency_for_kv_cache_config` and `update_kv_cache_capacity`, which emit the `GPU KV cache size: N tokens, Maximum concurrency for M tokens per request: X.XXx` line and define exactly what that concurrency figure means (`num_blocks ÷ Σ_groups ceil(per-request bytes ÷ page bytes)`).
3. **vLLM engine source — `vllm/v1/kv_cache_interface.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/kv_cache_interface.py — `AttentionSpec.unpadded_page_size_bytes = num_kv_heads × block_size × (head_size + head_size_v) × dtype_size`, the line from which §6's `2 · L · n_kv · d · e` formula falls out directly.
4. **vLLM — `docs/configuration/conserving_memory.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/conserving_memory.md — the memory-reduction playbook: tensor parallelism, quantisation, `max_model_len`, `max_num_seqs`, CUDA-graph tuning via `compilation_config` or `enforce_eager`.
5. **vLLM — `docs/configuration/optimization.md`** — https://github.com/vllm-project/vllm/blob/main/docs/configuration/optimization.md — chunked prefill enabled by default in V1, the `max_num_batched_tokens` ITL-vs-TTFT tradeoff, the four KV-preemption remedies, and the parallelism strategy summary referenced in §8–9.
6. **Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention," SOSP '23** — https://arxiv.org/abs/2309.06180 — read §1–2 for the framing of KV cache as *the* serving-cost bottleneck. The fragmentation measurements are 07.2's material. *(arxiv.org is blocked by this environment's egress proxy; the claims cited here are cross-checked against the vLLM source tree and the engine's own design docs rather than the PDF.)*
7. **Ainslie et al., "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints" (EMNLP 2023)** — https://arxiv.org/abs/2305.13245 — why `num_key_value_heads`, not `num_attention_heads`, is the cost-relevant number, and the uptraining result (≈5 % of original pre-training compute to convert an MHA checkpoint to GQA) that made GQA universal. *(Also behind the blocked arxiv domain; the architectural claim is verifiable directly from any post-2023 model's `config.json`, where `num_key_value_heads < num_attention_heads`.)*

**Real-world engineering**

8. **Character.AI — "Optimizing AI Inference at Character.AI"** — https://blog.character.ai/optimizing-ai-inference-at-character-ai/ — MQA, hybrid attention horizons (roughly one global-attention layer in six, the rest sliding-window), and cross-layer KV sharing, together cutting KV cache size >20× with no reported quality regression; ~20,000 QPS at under a cent per hour of conversation; ≥33× serving-cost reduction since 2022. The load-bearing case study for §10.
9. **Character.AI — "Optimizing AI Inference at Character.AI (Part Deux)"** — https://blog.character.ai/optimizing-ai-inference-at-character-ai-2/ — the follow-up, with the int8 training-and-serving story and further inter-turn caching detail.
10. **CoreWeave — "Choosing the Right NVIDIA GPU for Running Inference"** — https://www.coreweave.com/blog/choosing-the-right-nvidia-platform-for-running-inference-on-coreweave — the same weights + KV + overhead + concurrency budget presented as a GPU-selection decision for 70B-class deployments.
11. **Anyscale — "Achieve 23x LLM Inference Throughput & Reduce p50 Latency"** — https://www.anyscale.com/blog/continuous-batching-llm-inference — OPT-13B on one 40 GB A100; up to 23× over naive static batching, ~8× over Ray Serve and TGI, ~4× over FasterTransformer; the static baseline collapsing to 81 tok/s under realistic output-length variance. *(anyscale.com is blocked by this environment's egress proxy; figures here are as reported in secondary summaries of the post and should be re-checked against the original before being quoted in a document that matters.)*

**Deeper dives**

12. **Module 03 lesson 04 — "Decode throughput: bandwidth ceilings, batching, and the prefill/decode split"** — [../../03-gpu-hardware/lessons/04-decode-bandwidth-batching.md](../../03-gpu-hardware/lessons/04-decode-bandwidth-batching.md) — the two-term decode-step model `t_step(B) = max((W + B·S·k)/BW, B·(2N + a·S)/P)` that §2 compresses. Re-read it if the batch-size curve shape is not intuitive yet.
13. **Module 05 lesson 06 — "Inference SLOs: TTFT, TPOT, and why request-latency lies"** — [../../05-gpu-observability/lessons/06-inference-slos.md](../../05-gpu-observability/lessons/06-inference-slos.md) — the current, verified vLLM metric names, histogram bucket boundaries, and the ITL-versus-TPOT weighting distinction that this module's SLO arguments all assume.
14. **DeepSeek-V2 / V3 technical reports** — https://arxiv.org/abs/2405.04434 — multi-head latent attention, the low-rank KV compression referenced in §10 as the architectural step beyond MQA. *(arxiv blocked here; MLA's existence and its KV-footprint claim are corroborated by vLLM's own `fp8_ds_mla` cache dtype, which exists in `vllm/config/cache.py` specifically to store MLA's compressed latent at FP8.)*
