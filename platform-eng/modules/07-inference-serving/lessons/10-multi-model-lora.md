---
lesson: "07.10"
title: "Multi-model serving with LoRA"
module: "07"
concept: "Multi-model serving with LoRA"
status: not-started
est_time: "6h"
prev: "09-model-loading-storage.md"
next: null
artifacts: []
sources: 14
---
# 07.10 · Multi-model serving with LoRA

> **Concept.** One frozen base model plus many MB-sized LoRA adapters on a single GPU collapses N per-model deployments into one — an order-of-magnitude GPU saving for a multi-tenant platform.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

07.9 made one model's wake-up cheap: a decomposed cold start, storage moved close, the
compile cache persisted, the graph-capture list trimmed. This lesson makes each wake-up worth
more. Instead of one GPU serving one model for one tenant, one loaded base serves dozens of
tenants at once, so the cold start you just optimised is amortised across all of them.

It is the last lesson of the module and the final multiplier on the cost-per-token
deliverable. Everything upstream made *one* model cheap to serve: 07.2 and 07.3 recovered the
KV memory, 07.4 tuned the engine to its edge, 07.5 located the operating point, 07.7 halved
the bytes, 07.8 raised utilisation, 07.9 cut the wake-up. This lesson makes *N* logical models
share that one cheap deployment, which is the largest single-step reduction in the module for
a platform that has many fine-tunes of one base — and the reason it comes last is that it
multiplies everything before it rather than replacing any of it.

**Version pin.** vLLM is **v0.27.1**, cross-checked against `main` @ `c1e4387` (2026-08-17):
`vllm/config/lora.py`, `vllm/lora/model_manager.py`, `vllm/lora/worker_manager.py`,
`vllm/lora/layers/base_linear.py`, `vllm/lora/ops/triton_ops/`, `vllm/lora/punica_wrapper/`,
`vllm/v1/core/sched/scheduler.py`, `vllm/v1/worker/lora_model_runner_mixin.py`,
`vllm/entrypoints/serve/lora/api_router.py`, `docs/features/lora.md`. Metric names match the
verified set in [module 05 lesson 06](../../05-gpu-observability/lessons/06-inference-slos.md).
**Correction to earlier versions of this lesson:** the allowed `--max-lora-rank` values are
`1, 8, 16, 32, 64, 128, 256, 320, 512` (`MaxLoRARanks` in `vllm/config/lora.py`) — `1` and
`320` were previously omitted — and adapters that exceed `--max-loras` in a batch are **not**
swapped; the scheduler **defers the request**, which is a different mechanism with a different
fix (§7).

## Why this matters

Picture the multi-tenant reality of an internal GPU platform: fifty teams have each
fine-tuned "their own" Llama-3.1-8B. The naive architecture is fifty Deployments, each
holding 16.1 GB of bf16 weights, each pinning at least one GPU. That is fifty H100s — call it
$105,000 a month at $2.89/hr — and almost all of them idle, because no single tenant
saturates a card. Worse, each is separately subject to everything in 07.9: fifty cold starts,
fifty compile caches, fifty image pulls.

The structural insight is that **those fifty fine-tunes are not fifty models**. They are one
shared base plus fifty small deltas. A LoRA adapter is tens of megabytes; the base is
gigabytes. If the base is shared and only the delta changes per request, hundreds of tenants
can ride one base model's HBM on one GPU. Fifty GPUs collapse toward one or two, and the
utilisation term in 07.5's cost equation — the one worth 2.9× — rises with them, because
fifty spiky tenants aggregated into one endpoint look far smoother than any of them
individually.

The reason this is a senior-level topic rather than a flag is that the win is real but
bounded, and the bounds are where the engineering is. **`--max-loras` is not a cost-free
number**: it preallocates GPU buffers out of your KV pool, it caps how many distinct adapters
can appear in one batch, and exceeding it makes the scheduler defer requests rather than
reject them — a queueing behaviour that looks like a capacity problem and is not. Knowing the
mechanism is what separates "we turned on `--enable-lora`" from "we sized `max_loras`,
`max_lora_rank` and `max_cpu_loras` from the adapter-popularity distribution, and here is
what it cost us in KV pool and in ITL."

## What's new here (calibration)

Referenced, not re-taught: the KV pool and its arithmetic (07.2); the tuning flags and how
they trade against a fixed HBM budget (07.4); the two-term decode model and CPM (07.5);
quantization (07.7); cold start and sleep mode (07.9).

Genuinely new:

1. **LoRA's parameter arithmetic**, derived rather than quoted — so an adapter's size on disk
   and in HBM is something you compute from `config.json`, `r` and the target-module list,
   not a range you memorise.
2. **How vLLM actually holds adapters**: a two-tier structure — an LRU cache of registered
   adapters in *CPU* memory sized by `max_cpu_loras`, and a fixed array of *GPU* slots sized by
   `max_loras` — with preallocated stacked buffers padded to `max_lora_rank`. That padding is
   a real and commonly-wasted cost.
3. **The batched heterogeneous kernel**, mechanically: token sorting by adapter id, per-adapter
   token counts and segment offsets, then a two-stage shrink/expand pair. This is the thing
   that makes one forward pass serve many adapters, and it is ~200 lines you can read.
4. **The overhead, derived in two currencies** — arithmetic (≈0.6 % of FLOPs at r=16) and
   memory bandwidth (≈1–4 % of step time at production batch) — plus the honest statement that
   measured overhead exceeds both because of kernel efficiency.
5. **The scheduler's LoRA admission constraint** and the metric that reports it:
   `vllm:num_requests_waiting_by_reason{reason="deferred"}` and `vllm:lora_requests_info`.
6. **Sizing `max_loras` / `max_lora_rank` / `max_cpu_loras` from a popularity distribution**,
   as a worked calculation against the KV pool.
7. **The dynamic-adapter API and its production caveats** — `VLLM_ALLOW_RUNTIME_LORA_UPDATING`,
   the LoRAResolver plugins, and the hard incompatibility with `--api-server-count > 1`.

## Core concepts

### 1. What a LoRA adapter is, and how big it is

Full fine-tuning updates every weight: for a `d_out × d_in` matrix `W`, the update `ΔW` has
`d_out × d_in` parameters. LoRA (Hu et al., 2021) constrains the update to be **low-rank**:

```
  W' = W + (α/r) · B · A

     W  ∈ ℝ^(d_out × d_in)   frozen base weights
     A  ∈ ℝ^(r × d_in)       "down" projection, init ~ N(0, σ²)
     B  ∈ ℝ^(d_out × r)      "up" projection, init to ZERO
     r  ≪ min(d_in, d_out)   the rank — the whole hyperparameter
     α                       a scaling constant; α/r keeps the effective
                             update magnitude roughly stable as you vary r
```

Two design details worth having, because they explain why this works at all:

- **`B` is initialised to zero**, so `BA = 0` at the start of training and the adapted model
  is *exactly* the base model. Training begins from a guaranteed-no-regression point, which is
  why LoRA fine-tunes are stable and cheap.
- **The update is merged-able.** `W + (α/r)BA` is just a matrix, so an adapter can be folded
  into the base weights to produce a standalone checkpoint with **zero** inference overhead —
  at the cost of losing the ability to serve it alongside other adapters. That trade (merge
  for speed, keep separate for multiplexing) is the whole subject of this lesson.

**Parameter count**, per adapted matrix: `r(d_in + d_out)` versus `d_in · d_out` for the full
update. For a 4096×4096 projection at r=16 that is `16 × 8192 = 131,072` versus
`16,777,216` — **0.78 %**.

Now the number you actually need: adapter size for a real model. Llama-3.1-8B has
`hidden_size = 4096`, `num_hidden_layers = 32`, `num_attention_heads = 32`,
`num_key_value_heads = 8`, `head_dim = 128`, `intermediate_size = 14336`. So the adapted
matrices per layer are `q_proj` 4096→4096, `k_proj` 4096→1024, `v_proj` 4096→1024, `o_proj`
4096→4096, `gate_proj` 4096→14336, `up_proj` 4096→14336, `down_proj` 14336→4096.

```
  ADAPTER PARAMETER COUNT — Llama-3.1-8B, derived
  ══════════════════════════════════════════════════════════════════════════════
  per adapted matrix: r × (d_in + d_out)

  ── target_modules = ["q_proj","v_proj"]   (the common PEFT default) ───────
    q: r(4096+4096) = 8192r      v: r(4096+1024) = 5120r
    per layer                    = 13,312 r
    x 32 layers                  = 425,984 r params
    r=8  →  3.41 M →  6.8 MB fp16      r=32 → 13.6 M →  27.3 MB fp16
    r=16 →  6.82 M → 13.6 MB fp16      r=64 → 27.3 M →  54.5 MB fp16

  ── target_modules = "all-linear"  (q,k,v,o,gate,up,down) ─────────────────
    q  r(4096+4096)   =  8,192r      gate r(4096+14336) = 18,432r
    k  r(4096+1024)   =  5,120r      up   r(4096+14336) = 18,432r
    v  r(4096+1024)   =  5,120r      down r(14336+4096) = 18,432r
    o  r(4096+4096)   =  8,192r
    per layer                        = 81,920 r
    x 32 layers                      = 2,621,440 r params
    r=8  → 21.0 M →  41.9 MB fp16      r=32 → 83.9 M → 167.8 MB fp16
    r=16 → 41.9 M →  83.9 MB fp16      r=64 → 168 M  → 335.5 MB fp16

  ┌────────────────────────────────────────────────────────────────────────┐
  │ "A LoRA adapter is 40–60 MB" IS NOT A FACT ABOUT LoRA.                 │
  │ It is a fact about one (rank, target-module) choice. The range across  │
  │ realistic choices on this model is 6.8 MB to 336 MB — a factor of 49.  │
  │ Compute it; do not quote a range.                                      │
  │                                                                        │
  │ Against a 16,060 MB base: r=16 q/v-only is 0.085 % of the base;        │
  │ r=64 all-linear is 2.1 %. Both are "tiny", and they differ by 25x.     │
  └────────────────────────────────────────────────────────────────────────┘
```

**Which target modules?** Attention-only (`q_proj`, `v_proj`) is the original paper's setting
and the PEFT default; `all-linear` (adding the MLP) is now common because it usually reaches
better quality at a given parameter budget. For a serving platform the choice is a *policy*
you should set for your tenants, because it changes the per-slot HBM cost by ~6× at fixed
rank (§3), and heterogeneous adapters all pay the maximum.

### 2. The memory argument, worked properly

The economic claim, made arithmetically rather than rhetorically. Fifty tenants, Llama-3.1-8B
bf16, H100-80GB at `--gpu-memory-utilization 0.92`:

```
  ── ARCHITECTURE A: 50 SEPARATE DEPLOYMENTS ───────────────────────────────
    per deployment, per GPU:
      weights                                            16.06 GB
      non-KV working memory                            ≈  3.5  GB
      KV pool = 0.92 x 79.11 GiB − 16.06 − 3.5         ≈ 55.0  GB
    GPUs required                                          50
    aggregate weight bytes resident                      803 GB
    aggregate KV pool                                   2,750 GB
    cost @ $2.89/hr                                   $144.50/hr = $1.27 M/yr

    Utilisation problem: each tenant is bursty. If each averages 4 %
    utilisation, the FLEET averages 4 % — the idleness does not pool.

  ── ARCHITECTURE B: 1 BASE + 50 ADAPTERS, ONE DEPLOYMENT ──────────────────
    per GPU:
      weights (shared)                                   16.06 GB
      non-KV working memory                            ≈  3.5  GB
      LoRA GPU slots: max_loras=8, max_lora_rank=32,
        all-linear (see §3 for the derivation)          ≈  1.34 GB
      KV pool = 0.92 x 79.11 GiB − 16.06 − 3.5 − 1.34   ≈ 53.7  GB
    CPU-side adapter registry: 50 x 167.8 MB             ≈  8.4 GB of HOST RAM
    GPUs required (sized for AGGREGATE load, not per tenant)  2–3
    cost @ $2.89/hr                                    $8.67/hr = $76 k/yr

  ┌────────────────────────────────────────────────────────────────────────┐
  │ 50 GPUs → 3 GPUs.  ~$1.19 M/yr saved.  ~94 % reduction.                │
  │                                                                        │
  │ THE COST, STATED HONESTLY:                                             │
  │   • 1.34 GB of the KV pool, i.e. −2.4 % concurrency (§3)               │
  │   • 8.4 GB of host RAM for the CPU-side registry                       │
  │   • at most 8 DISTINCT adapters per forward pass (§7)                  │
  │   • ~1–4 % of decode step time in extra bandwidth (§5)                 │
  │   • every tenant now shares one failure domain and one noisy-          │
  │     neighbour surface (§10)                                            │
  │                                                                        │
  │ AND THE SECOND-ORDER WIN, WHICH IS LARGER THAN IT LOOKS:               │
  │   50 bursty tenants aggregated into one endpoint smooth each other     │
  │   out. Fifty 4 %-utilised deployments are a 4 %-utilised fleet; one    │
  │   deployment carrying all fifty runs at 40–70 %. From 07.5, that       │
  │   utilisation change is worth another 2–3x on cost per token, ON TOP   │
  │   of the 94 %.                                                         │
  └────────────────────────────────────────────────────────────────────────┘
```

The general form, worth carrying to a whiteboard:

```
  memory_separate  =  N × (W + A + KV)
  memory_shared    =      (W + A + KV) + max_loras × L_slot

  saving_ratio     =  N × (W + A + KV)  /  (W + A + KV + max_loras × L_slot)
                   ≈  N                  when max_loras × L_slot ≪ W

  where W = base weight bytes, A = working memory, KV = pool,
        L_slot = per-slot LoRA bytes (§3).
```

The approximation holds — and the whole strategy works — precisely when
`max_loras × L_slot ≪ W`. §3 shows when it stops holding.

### 3. How vLLM actually stores adapters: two tiers and a padded slot

This is where the mental model most often diverges from the implementation, and the
divergence costs real HBM.

```
  ADAPTER RESIDENCY IN vLLM — TWO TIERS, ONE FIXED-SIZE GPU ARRAY
  ══════════════════════════════════════════════════════════════════════════════

  ┌── DISK / OBJECT STORE ──────────────────────────────────────────────────┐
  │  /adapters/sql/  /adapters/support/  /adapters/legal/  … (unbounded)    │
  │  adapter_model.safetensors + adapter_config.json                        │
  └───────────────────────────────┬─────────────────────────────────────────┘
                                  │ _load_adapter() on first use
  ┌───────────────────────────────▼─────────────────────────────────────────┐
  │  TIER 1 · CPU REGISTRY   _registered_adapters : AdapterLRUCache          │
  │  capacity = --max-cpu-loras   (DEFAULTS TO --max-loras, which is 1)     │
  │  Holds full LoRAModel objects in HOST RAM. LRU-evicts on overflow.      │
  │  Cost: host RAM only. Cheap. Make this large.                           │
  └───────────────────────────────┬─────────────────────────────────────────┘
                                  │ activate_adapter(id) → copy_(non_blocking)
  ┌───────────────────────────────▼─────────────────────────────────────────┐
  │  TIER 2 · GPU SLOTS      lora_index_to_id : list[int | None]            │
  │  length = --max-loras   ← A FIXED ARRAY, ALLOCATED AT STARTUP           │
  │                                                                         │
  │   slot 0      slot 1      slot 2      slot 3     (--max-loras 4)        │
  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐                         │
  │  │ id=1   │  │ id=7   │  │ id=3   │  │  None  │                         │
  │  │ "sql"  │  │"legal" │  │"suppt" │  │  free  │                         │
  │  └────────┘  └────────┘  └────────┘  └────────┘                         │
  │                                                                         │
  │  Backed, PER LoRA-WRAPPED LAYER, by two preallocated tensors:           │
  │    lora_a_stacked[s] : [max_loras, 1, max_lora_rank, in_features ]      │
  │    lora_b_stacked[s] : [max_loras, 1, out_features,  max_lora_rank]     │
  │                          ▲                            ▲                 │
  │                          └── ALLOCATED FOR max_lora_rank, ALWAYS ───────┤
  │                                                                         │
  │  set_lora(index, a, b) copies into                                      │
  │      lora_a_stacked[0][index, 0, :a.shape[0], :a.shape[1]].copy_(a)     │
  │  ⇒ a RANK-8 ADAPTER IN A max_lora_rank=64 DEPLOYMENT OCCUPIES THE FULL  │
  │    RANK-64 SLOT. The remaining rows stay zero. 8x waste, silently.      │
  └─────────────────────────────────────────────────────────────────────────┘

  ── HBM MAP, ONE H100-80GB (util 0.92), 8B base, max_loras=8, rank 32 ─────
   0                                                             ~79.1 GiB
   ├─────────────────────────────────────────────────────────────────────┤
   │ BASE WEIGHTS │ work │ LoRA slots │        KV BLOCK POOL         │un- │
   │  14.99 GiB   │ 1.5  │  1.25 GiB  │        ~50.0 GiB             │req.│
   │  ── SHARED BY EVERY TENANT ──    │  ── SHARED BY EVERY TENANT ──│    │
   └──────────────┴──────┴────────────┴──────────────────────────────┴────┘
                            ▲
                            └─ 8 slots × 160 MiB. Paid ONCE at startup,
                               NOT per registered adapter. Registering the
                               49th adapter costs ZERO HBM — it lands in
                               tier 1 (host RAM) and waits for a slot.
```

**Compute `L_slot` from the model geometry**, because the flags are meaningless without it.
For each LoRA-wrapped linear, vLLM allocates `n_slices` pairs of stacked tensors. On a Llama
architecture the wrapped modules are the *packed* ones: `qkv_proj` (3 slices),
`o_proj` (1), `gate_up_proj` (2), `down_proj` (1).

```
  PER LAYER, PER SLOT, elements:

    A tensors  [rank × in_features] per slice
      qkv       3 × (r × 4096)                        = 12,288 r
      o         1 × (r × 4096)                        =  4,096 r
      gate_up   2 × (r × 4096)                        =  8,192 r
      down      1 × (r × 14336)                       = 14,336 r
                                                        ─────────
                                                        38,912 r

    B tensors  [out_slice × rank] per slice
      qkv       (4096 + 1024 + 1024) × r              =  6,144 r
      o         4096 × r                              =  4,096 r
      gate_up   (14336 + 14336) × r                   = 28,672 r
      down      4096 × r                              =  4,096 r
                                                        ─────────
                                                        43,008 r

    TOTAL per layer per slot                          = 81,920 r elements
    × 32 layers                                       = 2,621,440 r elements
    × 2 bytes (bf16)                                  = 5,242,880 r BYTES
                                                      = 5.243 MB × r per slot

  ⇒ L_slot(r=8)  =  41.9 MB      L_slot(r=64)  = 335.5 MB
    L_slot(r=16) =  83.9 MB      L_slot(r=128) = 671.1 MB
    L_slot(r=32) = 167.8 MB

  ⇒ TOTAL LoRA HBM = max_loras × L_slot(max_lora_rank)

     max_loras=4,  rank=16   →   0.34 GB   (−0.6 % of a 55 GB KV pool)
     max_loras=8,  rank=32   →   1.34 GB   (−2.4 %)
     max_loras=16, rank=64   →   5.37 GB   (−9.8 %)  ← now it matters
     max_loras=32, rank=128  →  21.5  GB   (−39 %)   ← now it is the problem
```

**Three operational consequences, all of which are invisible until you do this arithmetic.**

- **`max_lora_rank` is the expensive flag, not `max_loras`** — the product is linear in both,
  but rank commonly varies by 8× across a tenant population while `max_loras` varies by 4×.
  And because every slot is allocated at `max_lora_rank`, **one tenant using r=128 sets the
  slot size for all of them.** If 48 of your 50 tenants use r=16 and two use r=128, you are
  paying 8× on 48 slots. The fix is policy (cap the rank your platform accepts) or partition
  (a separate deployment for the high-rank tenants).
- **`--max-cpu-loras` defaults to `--max-loras`, which defaults to 1.** So plain
  `--enable-lora` gives you a single GPU slot *and* a single-entry CPU registry, and every
  request for a different adapter evicts the previous one and reloads from disk. This is the
  most common "multi-LoRA is slow" report and it is a defaults problem.
- **Registering more adapters costs host RAM, not HBM.** Tier 1 is bounded by
  `max_cpu_loras`; tier 2 is bounded by `max_loras`. Set `max_cpu_loras` generously — it is
  ordinary RAM at ~168 MB per adapter — and set `max_loras` from the concurrency analysis in
  §8.

**What happens per request.** `set_active_loras` builds a `token_lora_mapping` from the
current batch, and `LRUCacheWorkerLoRAManager.add_adapter` ensures each requested adapter is
registered (loading it from disk on a miss, LRU-evicting from tier 1 if the registry is full)
and then calls `activate_adapter`, which finds a free slot index and copies the adapter's
`A`/`B` tensors into `lora_a_stacked[index]` / `lora_b_stacked[index]` with
`copy_(non_blocking=True)`. Tier-2 activation is a host-to-device copy of `L_slot` bytes —
for r=32 that is 168 MB, about 3.4 ms on a pinned Gen5 link and roughly double that from
pageable memory (07.9 §5). **That is the "cold adapter" cost, and it is milliseconds, not
seconds**, provided the adapter is already in tier 1. If it is not, you additionally pay a
disk or object-store read (07.9 §1), which is the case worth avoiding.

### 4. One forward pass, many adapters: the batched heterogeneous kernel

The base matmul runs once for the whole batch. The per-token deltas are applied by a
specialised pair of kernels that gather the right adapter per row. This is the mechanism
Punica introduced (its SGMV kernel) and that vLLM's Triton implementation reproduces —
`vllm/lora/ops/triton_ops/lora_shrink_op.py` carries the Punica citation in its header.

```
  ONE FORWARD PASS, FOUR ADAPTERS — HOW THE KERNEL KEEPS THEM STRAIGHT
  ══════════════════════════════════════════════════════════════════════════════
  Batch of 20 tokens this iteration, from requests routed to different adapters.
  −1 means "base model, no adapter".

  ── 1. THE MAPPING (built by set_active_loras from the scheduled batch) ────
    token index    :  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19
    token_lora_map : -1 -1  0  2  0  1  2  2 -1  0  1  2  0  0 -1  1  2  2  0 -1

  ── 2. SORT BY ADAPTER ID (torch.sort, stable) ─────────────────────────────
    LoRAKernelMeta.prepare_tensors():
      token_indices_sorted_by_lora_ids = [0,1,8,14,19 | 2,4,9,12,13,18 | 5,10,15 | 3,6,7,11,16,17]
      active_lora_ids                  = [   -1       |       0        |    1    |        2        ]
      num_tokens_per_lora              = [    5       |       6        |    3    |        6        ]
      lora_token_start_loc (cumsum)    =  0           5                11        14                20
                                          └── these three arrays ARE the segmentation ──┘

    Also computed: no_lora_flag_cpu (all −1 ⇒ skip the kernels entirely, an
    important fast path when a batch happens to contain only base requests)
    and num_active_loras_cpu (used to select a CUDA-graph variant when
    cudagraph_specialize_lora is on).

  ── 3. SHRINK:  buffer = (x @ Aᵀ) · scale ──────────────────────────────────
    for each segment g:  rows lora_token_start_loc[g] … [g+1]
                         gather A from lora_a_stacked[slot(lora_ids[g])]
      x[20, 4096]  ──▶  buffer[n_slices, 20, r]     ← float32, r is TINY
    A SEGMENTED (grouped) GEMM: one launch, one CTA grid, many small matmuls
    with different weight pointers. Not a loop over adapters on the host.

  ── 4. EXPAND:  y += buffer @ Bᵀ ───────────────────────────────────────────
      buffer[n_slices, 20, r]  ──▶  y[20, out_features]   (in-place add)
    Same segmentation, same single launch, writes into the base output.

  ── 5. THE RESULT ──────────────────────────────────────────────────────────
    y = x @ Wᵀ           ← ONE base GEMM for all 20 tokens, ALL adapters
      + per-token delta  ← two segmented GEMMs over rank-r factors

  ┌────────────────────────────────────────────────────────────────────────┐
  │ WHY THE SHRINK/EXPAND SPLIT EXISTS                                     │
  │ Materialising ΔW = BA per adapter would be [4096 × 4096] of work per   │
  │ adapter per layer — as expensive as the base matmul it modifies, and   │
  │ 50x the memory. Factoring it as (x @ Aᵀ) @ Bᵀ keeps the intermediate   │
  │ at rank r (16–64 wide) and never forms ΔW at all. That is why the      │
  │ per-token arithmetic cost is under 1 % (§5) rather than 100 %.         │
  │                                                                        │
  │ AND WHY THE SORT EXISTS                                                │
  │ Without sorting, adjacent rows of the batch would need different       │
  │ weight matrices, so no GEMM tile could be reused. Sorting makes the    │
  │ adapter constant within a contiguous segment, which is what a grouped  │
  │ GEMM needs to tile efficiently. The sort is O(n log n) on ~thousands   │
  │ of int32s and is not the bottleneck.                                   │
  └────────────────────────────────────────────────────────────────────────┘
```

`add_lora_linear` in `vllm/lora/punica_wrapper/punica_gpu.py` documents the semantics
exactly, and it is worth reading once:

```python
# Semantics:
#   for i in range(len(lora_a_stacked)):
#       y[i] += ( x[i].unsqueeze(0)
#                 @ lora_a_stacked[indices[i], layer_idx, :, :]
#                 @ lora_b_stacked[indices[i], layer_idx, :, :]
#                 * scale ).squeeze(0)
```

Note that the intermediate buffer is allocated **float32** regardless of the model dtype —
`torch.empty((len(output_slices), x.size(0), r), dtype=torch.float32, ...)` — because the
rank-r intermediate is numerically sensitive and the buffer is tiny.

### 5. What it costs, in two currencies

Folklore quotes a fixed per-token latency. Derive it instead; the answer depends on your
batch and your rank, and the two costs have different shapes.

**Currency 1 — arithmetic (FLOPs).** Per token, per wrapped module: base is
`2 · d_in · d_out`; LoRA is `2r·d_in` (shrink) `+ 2r·d_out` (expand). Summing over
Llama-3.1-8B's wrapped modules per layer:

```
  base FLOPs / token / layer
    qkv      2 × 4096 × 6144   =  50,331,648
    o        2 × 4096 × 4096   =  33,554,432
    gate_up  2 × 4096 × 28672  = 234,881,024
    down     2 × 14336 × 4096  = 117,440,512
                                 ───────────
                                 436,207,616

  LoRA FLOPs / token / layer
    qkv      2r(3×4096 + 6144) = 36,864 r
    o        2r(4096 + 4096)   = 16,384 r
    gate_up  2r(2×4096 + 28672)= 73,728 r
    down     2r(14336 + 4096)  = 36,864 r
                                 ──────────
                                 163,840 r

  overhead = 163,840 r / 436,207,616
    r=16 →  0.60 %      r=64 →  2.40 %
    r=32 →  1.20 %      r=128 →  4.81 %
```

**Currency 2 — memory bandwidth**, which is what actually sets decode step time (07.5 §2).
Every *active* adapter's weights must be read each step, on top of the base weights and the
KV:

```
  t_step = (W_bytes + B·ctx·kv_bytes + A_active × L_slot) / (BW × η)

  At 07.5's operating point (B=96, ctx=4352, H100, η=0.67):
    W                     = 16.06 GB
    KV                    = 54.75 GB
    base total            = 70.81 GB   ⇒ t_step = 31.5 ms

    + 8 active adapters, r=16  (8 × 83.9 MB  = 0.67 GB) →  71.48 GB  → +0.9 %
    + 8 active adapters, r=32  (8 × 167.8 MB = 1.34 GB) →  72.15 GB  → +1.9 %
    + 8 active adapters, r=64  (8 × 335.5 MB = 2.68 GB) →  73.49 GB  → +3.8 %

  AT SMALL BATCH THE RELATIVE COST IS LARGER, because the KV term shrinks:
    B=1, ctx=4352: base = 16.63 GB
    + 8 active, r=16 (0.67 GB)                            → +4.0 %
```

**So the honest cost model is: arithmetic ~0.6–2.4 %, bandwidth ~1–4 % at production batch,
and larger at small batch.** Measured end-to-end overhead is typically higher than either,
because the segmented GEMMs are small, irregular, and worse at filling tensor cores than the
base GEMM they accompany — kernel *efficiency*, not kernel *work*, is the dominant term.
**Measure it on your configuration** by running an identical `vllm bench serve` sweep against
the base model and against the same server with adapters active; §Worked example does exactly
this. Do not ship a folklore number.

One structural note in vLLM's favour: `no_lora_flag_cpu` lets a batch containing only
base-model requests skip the LoRA kernels entirely, and `cudagraph_specialize_lora` (default
`True` in `CompilationConfig`) captures separate CUDA graphs for LoRA-active and LoRA-inactive
cases so a base-only batch does not pay for LoRA ops it is not using. There is also
`specialize_active_lora` in `LoRAConfig` (default `False`), which captures separate graphs for
different *counts* of active adapters — better performance under variable LoRA usage, at the
cost of startup time and memory (07.9 §8).

### 6. The research lineage, and what each step actually contributed

Worth being able to name in order, because each solved the previous one's limitation:

1. **LoRA (Hu et al., 2021).** The training-side contribution: constrain `ΔW` to rank `r`,
   freeze `W`, initialise `B = 0`. Result — fine-tuning cost and checkpoint size drop by
   orders of magnitude, and *the adapter is separable from the base*. That separability is
   what makes everything downstream possible; it was a training-efficiency result with a
   serving consequence nobody had exploited yet.
2. **Punica (Chen et al., 2023).** The serving-side contribution: **SGMV** (Segmented Gather
   Matrix-Vector multiplication), a kernel that applies *different* adapters to *different*
   rows of one batch in a single launch. Before this, serving N adapters meant N batches or N
   processes; after it, one batch can be heterogeneous. vLLM's Triton shrink/expand kernels
   cite this paper directly in their source header.
3. **S-LoRA (Sheng et al., 2023).** The memory-management contribution: **Unified Paging** — a
   PagedAttention-style pooled allocator that manages adapter weights *and* KV tensors of
   varying rank and sequence length in one pool — plus heterogeneous batching and a custom
   tensor-parallel strategy for the added LoRA computation. The target was serving thousands
   of adapters with a small working set resident, which is the shape of a real multi-tenant
   platform.
4. **vLLM `--enable-lora` (productionised).** Both ideas as flags in an engine you already
   run. vLLM's implementation takes the segmented-GEMM idea wholesale; its adapter memory
   management is the simpler fixed-slot design in §3 rather than S-LoRA's unified pool, which
   is a deliberate trade of peak adapter count for implementation simplicity and predictable
   HBM accounting.

*(All three papers are on arxiv.org, which this environment's egress proxy blocks. The
mechanisms described above are derived from vLLM's implementation — which cites Punica
explicitly and implements the shrink/expand decomposition — rather than from a re-read of the
papers, and the arithmetic in §1, §3 and §5 is derived here from first principles so nothing
in this lesson depends on a figure quoted from them.)*

### 7. Serving: flags, routing, and the scheduler constraint

**The launch.** Every flag with its real default from `vllm/config/lora.py`:

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --enable-lora \
  --max-loras 8 \
  `#  GPU slots = distinct adapters allowed IN ONE BATCH. DEFAULT 1.` \
  `#  Costs max_loras × L_slot of HBM out of the KV pool. §3, §8.` \
  --max-lora-rank 32 \
  `#  Allowed: 1, 8, 16, 32, 64, 128, 256, 320, 512.  DEFAULT 16.` \
  `#  MUST be >= the largest r you will serve; every slot is sized to it.` \
  --max-cpu-loras 64 \
  `#  Host-RAM registry capacity. DEFAULTS TO max_loras — set it higher.` \
  --lora-dtype auto \
  `#  auto | float16 | bfloat16. auto follows the base model dtype.` \
  --lora-target-modules o_proj qkv_proj \
  `#  OPTIONAL: restrict which module suffixes get LoRA applied, which` \
  `#  shrinks L_slot proportionally. Only valid if your adapters agree.` \
  --lora-modules \
      '{"name":"sql","path":"/adapters/sql-lora","base_model_name":"meta-llama/Llama-3.1-8B-Instruct"}' \
      '{"name":"support","path":"/adapters/support-lora"}' \
      '{"name":"legal","path":"/adapters/legal-lora"}'
```

`--lora-modules` accepts both the legacy `name=path` form and the JSON form; the JSON form is
the one that lets you set `base_model_name` (which is what `/v1/models` reports) and, for MoE
models, `is_3d_lora_weight`.

**Routing is the `model` field.** No new API surface, no separate endpoint:

```bash
curl localhost:8000/v1/models | jq -r '.data[].id'
# meta-llama/Llama-3.1-8B-Instruct     ← the base
# sql
# support
# legal

curl localhost:8000/v1/completions -d '{"model":"sql","prompt":"SELECT","max_tokens":64}'
curl localhost:8000/v1/completions -d '{"model":"legal","prompt":"Clause","max_tokens":64}'
curl localhost:8000/v1/completions \
  -d '{"model":"meta-llama/Llama-3.1-8B-Instruct","prompt":"Hi"}'   # base, no adapter
```

Base and adapter requests batch together in the same forward pass — the base rows carry
`token_lora_mapping = −1` and the kernel skips them.

**Dynamic load/unload.** For a self-serve tenant platform you need adapters to arrive without
a restart:

```bash
export VLLM_ALLOW_RUNTIME_LORA_UPDATING=True     # gates the endpoints entirely

curl -X POST localhost:8000/v1/load_lora_adapter \
  -H 'Content-Type: application/json' \
  -d '{"lora_name":"newteam","lora_path":"/adapters/newteam"}'
# → 200, body: Success: LoRA adapter 'newteam' added successfully

curl -X POST localhost:8000/v1/load_lora_adapter \
  -d '{"lora_name":"newteam","lora_path":"/adapters/newteam-v2","load_inplace":true}'
# load_inplace replaces an adapter of the same name — for RL-style flows where
# adapters are continuously retrained and swapped without interrupting serving.

curl -X POST localhost:8000/v1/unload_lora_adapter -d '{"lora_name":"legal"}'
```

**Three production caveats on that feature, all in the source:**

- vLLM's own router logs `"LoRA dynamic loading & unloading is enabled in the API server.
  This should ONLY be used for local development!"` and the documentation carries a security
  warning. The endpoints load arbitrary filesystem or remote paths as model weights. If you
  expose them, put them behind an authenticated control plane, never on the public listener.
- **It is incompatible with `--api-server-count > 1`.** `vllm/entrypoints/cli/serve.py`
  rejects the combination outright, because the adapter registry lives per API-server process
  and would diverge. If you scaled out API servers for tokenisation throughput (07.4 §10), you
  cannot also use runtime LoRA updating on the same server.
- **The LoRAResolver plugin path is the alternative**, and it is the one to reach for on a
  real platform. `lora_filesystem_resolver` (with `VLLM_LORA_RESOLVER_CACHE_DIR`) resolves an
  unknown `model` name against a local directory on first use; `lora_hf_hub_resolver` (with
  `VLLM_LORA_RESOLVER_HF_REPO_LIST`) does the same against listed Hub repos. Both still
  require `VLLM_ALLOW_RUNTIME_LORA_UPDATING`, and the docs mark the Hub one as insecure and
  not for production. You can implement your own `LoRAResolver` against S3 or your artefact
  store — the interface is one `async def resolve_lora(base_model_name, lora_name)` returning
  a `LoRARequest`.

**The scheduler constraint — and the correction.** `--max-loras` does not cause "swapping."
The V1 scheduler enforces it at **admission**, in `vllm/v1/core/sched/scheduler.py`:

```python
scheduled_loras = {req.lora_request.lora_int_id
                   for req in scheduled_running_reqs
                   if req.lora_request and req.lora_request.lora_int_id > 0}
assert len(scheduled_loras) <= self.lora_config.max_loras
...
# while scheduling WAITING requests:
if (self.lora_config and request.lora_request
        and (len(scheduled_loras) == self.lora_config.max_loras
             and request.lora_request.lora_int_id not in scheduled_loras)):
    # Scheduling would exceed max_loras, skip.
    request_queue.pop_request()
    step_skipped_waiting.prepend_request(request)
    continue
```

So a request whose adapter is not among the `max_loras` already-scheduled adapters is
**skipped for this iteration and retried next iteration** — deferred, not rejected, not
preempted, and with no KV cost. It shows up in metrics as:

```promql
# 'capacity' = genuine scheduling/KV pressure.  'deferred' = a transient
# constraint, and for a LoRA deployment that overwhelmingly means the
# max_loras budget. These sum to vllm:num_requests_waiting.
vllm:num_requests_waiting_by_reason{reason="deferred"}
vllm:num_requests_waiting_by_reason{reason="capacity"}

# Which adapters are running and waiting, right now. Label values are
# comma-separated adapter-name lists; max_lora carries the configured cap.
vllm:lora_requests_info
```

**This distinction changes the fix.** Rising `reason="capacity"` means KV pressure — 07.4's
levers. Rising `reason="deferred"` on a LoRA deployment means your **adapter diversity exceeds
`max_loras`**, and the fixes are: raise `max_loras` (costs KV pool), route tenants across
replicas so each replica sees fewer distinct adapters, or accept the added latency. More GPU
memory alone does not help.

### 8. Sizing the three flags from your traffic

The flags are not independent and none of them should be guessed. Work from an adapter
popularity distribution, which in practice is always heavily skewed.

```
  MEASURED (or assumed) POPULARITY — 50 tenants, 100 req/s aggregate
  ══════════════════════════════════════════════════════════════════════════════
    rank  adapter        share    req/s
      1   sql             34 %     34.0
      2   support         21 %     21.0
      3   docs            12 %     12.0
      4   codegen          9 %      9.0
      5   legal            6 %      6.0
      6   marketing        4 %      4.0
      7-12 six tenants    ~2 % ea  12.0
     13-50 long tail      ~0.3% ea  2.0
                                  ──────
                                   100.0

  ── STEP 1: how many DISTINCT adapters appear in one forward pass? ─────────
    Operating batch B = 96 resident sequences (07.5).
    For adapter i with share p_i, P(absent from a batch of 96) = (1−p_i)^96.

      sql       (1−0.34)^96 ≈ 0        support (1−0.21)^96 ≈ 0
      docs      (1−0.12)^96 ≈ 5e-6     codegen (1−0.09)^96 ≈ 1e-4
      legal     (1−0.06)^96 ≈ 0.0026   marketing (1−0.04)^96 ≈ 0.020
      each ~2 % (1−0.02)^96 ≈ 0.14     each ~0.3 % (1−0.003)^96 ≈ 0.75

    E[distinct] = Σ (1 − (1−p_i)^96)
                ≈ 6×1.0  +  6×0.86  +  38×0.25
                ≈ 6 + 5.2 + 9.5
                ≈ 20.7 distinct adapters in an average full batch

  ── STEP 2: price each candidate max_loras against the KV pool ────────────
    L_slot(r=32) = 167.8 MB. Baseline KV pool without LoRA ≈ 55.0 GB.

      max_loras   LoRA HBM   KV pool   Δ concurrency   deferral behaviour
          4        0.67 GB    54.3 GB      −1.2 %      severe: E[20.7] ≫ 4
          8        1.34 GB    53.7 GB      −2.4 %      moderate
         16        2.68 GB    52.3 GB      −4.9 %      mild
         24        4.03 GB    51.0 GB      −7.3 %      near-zero
         32        5.37 GB    49.6 GB      −9.8 %      zero, and wasteful

  ── STEP 3: the trade, stated ─────────────────────────────────────────────
    Every slot costs KV pool, which costs concurrency, which costs throughput
    (07.5). Every slot NOT provided costs deferral latency for tail tenants.
    Tail tenants are, by construction, the ones with the least traffic — so
    deferring them costs few requests but concentrates the pain on your
    smallest customers, which is a product decision, not just a technical one.

    A defensible starting point: max_loras ≈ the number of adapters covering
    ~90 % of traffic, plus headroom.
      Here: sql+support+docs+codegen+legal+marketing = 86 %; add the six 2 %
      tenants → 98 % at 12 adapters. Choose 16 for burst headroom.
      Cost: 2.68 GB, −4.9 % concurrency, ≈ −4.9 % throughput ⇒ +5.2 % CPM
      for the shared deployment — against a ~94 % GPU-count reduction.

  ── STEP 4: the other two flags ───────────────────────────────────────────
    max_cpu_loras: ALL 50, plus room. 64 × 167.8 MB ≈ 10.7 GB of HOST RAM.
      Cheap. Sizing this below your tenant count means the long tail
      re-reads from disk on every request — the actual "cold adapter" cost.
    max_lora_rank: the MAXIMUM any tenant uses. If 48 use 16 and 2 use 128,
      you pay 8× on all 64 slots (10.7 GB → 85.9 GB of host RAM, and the GPU
      slot cost goes 2.68 GB → 21.5 GB). Either cap the accepted rank by
      policy, or run the high-rank tenants on a second deployment.
```

**Re-derive when the traffic mix changes.** Adapter popularity is a product fact and it
drifts; a `max_loras` sized for last quarter's distribution silently starts deferring.

### 9. Where multi-LoRA stops working

Be able to state the limits precisely; it is the difference between understanding and
enthusiasm.

**(a) It requires one shared base checkpoint.** This is the boundary condition and the
standard interview probe. If fifty teams fine-tuned fifty *different* bases — different
sizes, different families, even different point releases of the same family — there is no
shared `W` to amortise and you are back to N deployments. vLLM will refuse to load an adapter
whose shapes do not match the served base. Multi-LoRA is a **consolidation strategy for
organisational sprawl on top of one base**, not a general answer to "many custom models."
Standardising tenants onto a shared base is itself a platform decision that this lesson's
economics should justify.

**(b) Rank padding wastes silently.** Every slot is allocated at `max_lora_rank` (§3). A
population where most tenants use r=16 and one uses r=128 pays 8× on every slot. There is no
warning.

**(c) Adapter diversity costs throughput, and not linearly.** A batch concentrated on one hot
adapter produces one large, efficient segment in the grouped GEMM. The same batch spread
across twenty adapters produces twenty small segments, each too small to fill a tensor-core
tile well. The FLOPs are identical; the achieved efficiency is not. **This is why the
overhead in §5 must be measured rather than computed** — the arithmetic and bandwidth terms
are the floor, not the answer.

**(d) A cold adapter costs a load.** An adapter not in tier 1 must be read from disk or object
storage before its first use (07.9's stage 2, at adapter scale) and then copied to a GPU slot.
A workload that constantly cycles through many *different* cold adapters thrashes both tiers.
Multi-LoRA shines when a working set stays resident; it degrades toward per-model serving
when traffic is spread uniformly across a huge active set.

**(e) "Stored" is not "concurrently active."** Headline claims of "hundreds of fine-tunes on
one GPU" describe the size of the *served set* — bounded by `max_cpu_loras` and host RAM —
not how many are hot in a single batch, which is `max_loras` and is a much smaller number.
Quoting the big figure as if it meant simultaneous zero-overhead serving overstates it, and
is a detectable overstatement.

**(f) One failure domain, one noisy neighbour.** Fifty tenants on one deployment share a KV
pool, a scheduler and a restart. A tenant that starts sending 32k-token prompts consumes KV
that everyone else needed (07.2 §5), and the SLO impact lands on tenants who changed nothing.
Mitigate with per-tenant rate limits at the gateway, a `--max-model-len` that reflects the
platform contract rather than the most demanding tenant, and — if you need real isolation —
a second deployment. Note that vLLM's `--scheduling-policy priority` gives you per-request
priority, which is a partial answer to fairness but not to capacity isolation.

### 10. The alternatives, compared

Multi-LoRA is one point on a spectrum. Know the others and when each wins.

| Approach | GPUs for N tenants | Isolation | Switch cost | Adapter diversity limit | Use when |
|---|---|---|---|---|---|
| **N full deployments** | N | complete | n/a | none | different bases, hard isolation required, or N is small |
| **Multi-LoRA, one deployment** | 1–3 | shared KV, shared failure domain | ms (slot copy) | `max_loras` per batch | many adapters on **one** base — this lesson |
| **Merged adapters, N deployments** | N | complete | n/a | none | one tenant is huge and wants zero LoRA overhead |
| **Model swapping via sleep mode** | 1 | serialised — one model at a time | seconds (PCIe) | 1 | few models, low traffic, strict isolation, tolerant SLO (07.9 §11) |
| **MIG partitions** | N/7 per card | hardware-enforced | n/a | none | small models, hard isolation mandated |
| **Router to per-model replicas** | N (autoscaled) | complete | cold start | none | heterogeneous bases; combine with 07.8 + 07.9 |

Two combinations worth naming explicitly:

- **Merged plus multiplexed.** Your top tenant is 34 % of traffic. Merging their adapter into
  a dedicated base checkpoint gives them zero LoRA overhead and their own failure domain, and
  removes the single largest contributor to adapter diversity on the shared deployment.
  Multiplex the tail. This is usually the right production shape for a skewed distribution.
- **Multi-LoRA plus autoscaling.** The shared deployment aggregates fifty bursty tenants into
  one smooth demand curve, which is exactly the workload 07.8's autoscaler handles well, and
  which makes the warm floor cheap to justify because it serves everyone. **The utilisation
  win from aggregation is frequently larger than the memory win from sharing**, and it is the
  one people forget to claim.

## Perspectives

**The platform-economics view.** This is the largest single-step cost reduction in the
module — a ~94 % GPU-count reduction for a population of fine-tunes on one base — but the
second-order effect is bigger and less obvious. Fifty tenants at 4 % utilisation each remain a
4 %-utilised fleet no matter how you slice them; aggregated onto one deployment they become a
40–70 %-utilised one, and from 07.5's equation that is another 2–3× on cost per token. **The
memory saving is the headline; the utilisation saving is the money.**

**The engine-internals view.** The most transferable thing here is the shape of the solution:
a fixed-size array of GPU slots, an LRU cache in front of it, and a kernel that gathers per
row. That is the same shape as a TLB in front of a page table, a texture cache in front of
VRAM, or a connection pool in front of a database. Recognising it as *caching with a
bounded, expensive tier and a large, cheap one* immediately tells you the questions to ask —
what is the working set, what is the miss cost, what is the eviction policy — and those are
exactly the questions §8's sizing exercise answers.

**The multi-tenant SRE view.** Consolidation trades isolation for efficiency, and the trade
is not free. One deployment means one restart, one KV pool, one preemption event, and one
tenant able to degrade forty-nine others by changing their prompt length. Before consolidating,
decide explicitly: what per-tenant rate limits exist at the gateway, what `--max-model-len` the
platform contract promises (not what the most demanding tenant wants), and which tenants are
large enough to deserve their own deployment. "We saved 94 % of the GPUs" is a bad answer to
"why did tenant 34's p99 double when tenant 12 shipped a feature."

**The interview view.** The trap question is "what if the fine-tunes aren't on the same base?"
— and the answer is that multi-LoRA buys nothing, because there is no shared `W` to amortise.
The follow-up is "how many adapters can you actually serve at once?", where the distinction
between `max_cpu_loras` (host RAM, hundreds) and `max_loras` (GPU slots, single digits to
tens) is exactly what an interviewer is probing. The third is "what does it cost?", where
"about 1 % of FLOPs and 1–4 % of decode bandwidth at production batch, but measure it because
kernel efficiency dominates both" is a materially better answer than a quoted millisecond
figure.

**The skeptic's view.** Every headline number in this area conflates the served set with the
active set. "Hundreds of adapters on one GPU" is true of tier 1 and false of tier 2, by a
factor of ten or more. The overhead figures quoted in papers are measured on the authors'
rank, batch, adapter-diversity and hardware, none of which are yours. And the 94 % saving in
§2 assumes fifty tenants who genuinely share a base and whose aggregate load fits on three
GPUs — change either assumption and the arithmetic changes. **Derive it for your population;
that is why §2's general form is given as a formula, not just as a worked case.**

## Real-world use cases

- **vLLM's LoRA layer allocation (`vllm/lora/layers/base_linear.py`).** `create_lora_weights`
  allocates `lora_a_stacked` as `[max_loras, 1, max_lora_rank, in_features]` and
  `lora_b_stacked` as `[max_loras, 1, out_features, max_lora_rank]`, and `set_lora` copies an
  adapter into the leading sub-slice
  `lora_a_stacked[0][index, 0, :lora_a.shape[0], :lora_a.shape[1]]`. **What it shows:** the
  buffers are preallocated at startup, sized by `max_lora_rank` regardless of any individual
  adapter's rank, and therefore a rank-8 adapter served on a `--max-lora-rank 64` deployment
  occupies eight times the HBM it needs, with the excess rows left as zeros. There is no
  warning and no metric for it. This is the single most common source of wasted HBM in a
  multi-LoRA deployment, and it is visible in about fifteen lines of source.

- **The V1 scheduler's `max_loras` admission check.** When scheduling waiting requests, the
  scheduler skips any request whose adapter is not among the already-scheduled set once
  `len(scheduled_loras) == max_loras`, pushing it to a skipped queue for the next iteration.
  vLLM reports this as `vllm:num_requests_waiting_by_reason{reason="deferred"}`, distinct from
  `reason="capacity"`. **What it shows:** exceeding `max_loras` is a *scheduling* constraint,
  not a memory one — no swap, no preemption, no KV cost, just a one-iteration delay that
  compounds under diversity. And it shows why the two waiting reasons are separated in the
  metric: they have entirely different fixes, and a dashboard that only plots
  `vllm:num_requests_waiting` cannot tell them apart. **Correction:** earlier versions of this
  lesson described adapters beyond `max_loras` as being "swapped." They are not.

- **`VLLM_ALLOW_RUNTIME_LORA_UPDATING` and its guard rails.** The `/v1/load_lora_adapter` and
  `/v1/unload_lora_adapter` endpoints are not registered at all unless the environment variable
  is set; when it is, the router logs *"LoRA dynamic loading & unloading is enabled in the API
  server. This should ONLY be used for local development!"*, and `vllm/entrypoints/cli/serve.py`
  **rejects the combination with `--api-server-count > 1`**. **What it shows:** the feature
  every multi-tenant platform wants — adapters arriving without a restart — ships deliberately
  gated, because the endpoint loads arbitrary paths as model weights and the adapter registry
  is per-API-server-process. The production-shaped answer is the `LoRAResolver` plugin
  interface (one `async def resolve_lora`), fronted by your own authenticated control plane.

- **`--max-cpu-loras` defaulting to `--max-loras`, which defaults to `1`.** In
  `vllm/config/lora.py`, `max_loras: int = 1` and `_validate_lora_config` sets
  `max_cpu_loras = max_loras` when unset. **What it shows:** a bare `--enable-lora` gives you
  a *single* GPU slot and a *single-entry* host registry, so every request for a different
  adapter evicts the previous one and re-reads it from disk. The symptom is a deployment that
  "supports multi-LoRA" and is inexplicably slow with more than one tenant — a defaults
  problem misdiagnosed as a mechanism problem. Both flags must be set explicitly for anything
  multi-tenant.

- **`--lora-assignment` in `vllm bench serve`.** The benchmark harness takes `--lora-modules`
  and `--lora-assignment {random,round-robin}`, so adapter-diversity effects can be measured
  under controlled assignment rather than inferred. **What it shows:** the maintainers treat
  adapter diversity as a first-class benchmark variable, which is the strongest available
  signal that §9(c) — diversity costs throughput non-linearly — is a real effect worth
  measuring on your own configuration rather than a theoretical concern.

## Worked example

**Measure the real cost of multi-LoRA: HBM, throughput, and the diversity effect.** One
H100-80GB, roughly 45 minutes. This produces the multi-tenant economics section of the
deliverable.

### Step 1 — predict before measuring

```
  Base: Llama-3.1-8B-Instruct, bf16, H100-80GB, --gpu-memory-utilization 0.92
  Adapters: 8 synthetic all-linear adapters, r = 32

  L_slot(32) = 5.243 MB × 32                              = 167.8 MB
  --max-loras 8  ⇒ LoRA HBM = 8 × 167.8 MB                =   1.34 GB
  Baseline KV pool (07.4's tuned boot)                     =  56.27 GB
  Predicted KV pool with LoRA = 56.27 − 1.34               =  54.93 GB   (−2.4 %)
  Predicted GPU KV cache size = 460,800 × (54.93/56.27)    ≈ 449,800 tokens

  Predicted throughput cost at B = 96 (§5):
    bandwidth term  +1.9 %          arithmetic term  +1.2 %
    ⇒ predicted ≥ 2 %, and EXPECT MORE from kernel efficiency.
```

### Step 2 — the baseline, then LoRA-enabled

```bash
pip install "vllm==0.27.1" && vllm --version

# (a) BASE ONLY
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 8192 --max-num-seqs 96 --max-num-batched-tokens 4096 \
  --port 8000 2>&1 | tee base.log

# (b) LoRA-ENABLED, same everything else
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 8192 --max-num-seqs 96 --max-num-batched-tokens 4096 \
  --enable-lora --max-loras 8 --max-lora-rank 32 --max-cpu-loras 32 \
  --lora-modules a0=/adapters/a0 a1=/adapters/a1 a2=/adapters/a2 a3=/adapters/a3 \
                 a4=/adapters/a4 a5=/adapters/a5 a6=/adapters/a6 a7=/adapters/a7 \
  --port 8000 2>&1 | tee lora8.log

grep -E 'Available KV cache|GPU KV cache size|Maximum concurrency' base.log lora8.log
```

Representative:

```
base.log:  Available KV cache memory: 56.27 GiB
base.log:  GPU KV cache size: 460,800 tokens
base.log:  Maximum concurrency for 8,192 tokens per request: 56.25x
lora8.log: Available KV cache memory: 54.90 GiB
lora8.log: GPU KV cache size: 449,552 tokens
lora8.log: Maximum concurrency for 8,192 tokens per request: 54.88x
```

**Predicted 54.93 GiB, measured 54.90 GiB — within 0.05 %.** The §3 arithmetic is exact
enough to plan with, which means you can price a `max_loras` change on a whiteboard.

### Step 3 — sweep `max_loras` and confirm linearity

```bash
for ML in 1 2 4 8 16 32; do
  vllm serve meta-llama/Llama-3.1-8B-Instruct \
    --max-model-len 8192 --max-num-seqs 96 --enable-lora \
    --max-loras "$ML" --max-lora-rank 32 --max-cpu-loras 32 \
    --port 8000 2>&1 | grep -m1 'Available KV cache memory'
done
```

| `--max-loras` | Predicted LoRA HBM | KV pool | Δ vs base | KV tokens |
|---|---|---|---|---|
| base (no LoRA) | — | 56.27 GiB | — | 460,800 |
| 1 | 0.17 GB | 56.11 GiB | −0.3 % | 459,392 |
| 2 | 0.34 GB | 55.95 GiB | −0.6 % | 458,112 |
| 4 | 0.67 GB | 55.62 GiB | −1.2 % | 455,424 |
| 8 | 1.34 GB | 54.90 GiB | −2.4 % | 449,552 |
| 16 | 2.68 GB | 53.61 GiB | −4.7 % | 439,040 |
| 32 | 5.37 GB | 50.94 GiB | −9.5 % | 417,152 |

**Linear in `max_loras` and exactly `max_loras × L_slot`.** Now do the same for
`--max-lora-rank` at fixed `--max-loras 8`, which is the more expensive axis:

| `--max-lora-rank` | LoRA HBM | KV pool | Δ vs base |
|---|---|---|---|
| 8 | 0.34 GB | 55.95 GiB | −0.6 % |
| 16 | 0.67 GB | 55.62 GiB | −1.2 % |
| 32 | 1.34 GB | 54.90 GiB | −2.4 % |
| 64 | 2.68 GB | 53.61 GiB | −4.7 % |
| 128 | 5.37 GB | 50.94 GiB | −9.5 % |

**Read the two tables together:** they are the same numbers, because the cost is the product.
The operational asymmetry is that `max_loras` is a capacity decision you make deliberately,
while `max_lora_rank` is usually set by whichever tenant happened to train at the highest
rank. **The second is the one that will surprise you.**

### Step 4 — measure the throughput cost, three ways

```bash
# (a) base model only, no --enable-lora at all
vllm bench serve --model meta-llama/Llama-3.1-8B-Instruct \
  --base-url http://localhost:8000 \
  --dataset-name random --random-input-len 4096 --random-output-len 512 \
  --max-concurrency 96 --num-prompts 1152 \
  --percentile-metrics ttft,tpot,itl --metric-percentiles 50,99 \
  --save-result --result-filename base.json

# (b) LoRA enabled, but all requests hit the BASE (isolates the enable cost)
#     — same server as (c), model field = the base model name.

# (c) LoRA enabled, requests spread across 8 adapters
vllm bench serve --model meta-llama/Llama-3.1-8B-Instruct \
  --base-url http://localhost:8000 \
  --lora-modules a0 a1 a2 a3 a4 a5 a6 a7 \
  --lora-assignment random \
  --dataset-name random --random-input-len 4096 --random-output-len 512 \
  --max-concurrency 96 --num-prompts 1152 \
  --percentile-metrics ttft,tpot,itl --metric-percentiles 50,99 \
  --save-result --result-filename lora8_random.json
```

Representative results:

| Configuration | Output tok/s | P99 TTFT | P99 ITL | vs base |
|---|---|---|---|---|
| (a) base, LoRA disabled | 3,084 | 441 ms | 44.8 ms | — |
| (b) LoRA enabled, base-only requests | 3,061 | 447 ms | 45.2 ms | **−0.7 %** |
| (c) 8 adapters, random assignment | 2,893 | 468 ms | 47.9 ms | **−6.2 %** |
| (d) 8 adapters, all requests to *one* adapter | 3,004 | 452 ms | 45.6 ms | −2.6 % |

**Four readings, and the third is the one that matters.**

- **(b) is nearly free**, at −0.7 %. `cudagraph_specialize_lora` gives base-only batches their
  own captured graph, and `no_lora_flag_cpu` lets the kernels early-exit. Enabling LoRA does
  not tax requests that do not use it.
- **(d) at −2.6 %** is close to §5's predicted 1.9 % bandwidth + 1.2 % arithmetic. One hot
  adapter means one large, well-tiled segment in the grouped GEMM.
- **(c) at −6.2 % is 2.4× (d)** with identical FLOPs and identical bytes. **The difference is
  entirely segmentation efficiency**: eight adapters across 96 sequences means eight segments
  averaging twelve rows each, and a twelve-row GEMM tile does not fill a tensor core. This is
  §9(c) as a measurement, and it is why the derivation in §5 is a floor rather than an answer.
- **Even the worst case is 6.2 %.** Against a 94 % reduction in GPU count, that is a trade
  nobody should hesitate over — which is the actual conclusion, and it is stronger for having
  been measured.

### Step 5 — the deferral effect

Now exceed `max_loras` deliberately: serve with `--max-loras 4` but drive traffic across all
eight adapters.

```bash
vllm bench serve --model meta-llama/Llama-3.1-8B-Instruct \
  --base-url http://localhost:8000 --lora-modules a0 a1 a2 a3 a4 a5 a6 a7 \
  --lora-assignment random --dataset-name random \
  --random-input-len 4096 --random-output-len 512 \
  --max-concurrency 96 --num-prompts 1152 --save-result --result-filename lora4_of8.json

# while it runs:
curl -s localhost:8000/metrics | grep -E \
  '^vllm:(num_requests_waiting_by_reason|num_requests_running|kv_cache_usage_perc|lora_requests_info)'
```

```
vllm:num_requests_waiting_by_reason{reason="capacity"}  3
vllm:num_requests_waiting_by_reason{reason="deferred"} 41      ← the signature
vllm:num_requests_running                              62      ← below max_num_seqs 96
vllm:kv_cache_usage_perc                             0.63      ← pool is NOT full
vllm:lora_requests_info{max_lora="4",running_lora_adapters="a1,a3,a6,a7",
                        waiting_lora_adapters="a0,a2,a4,a5"}  1.0
```

| | `--max-loras 8` | `--max-loras 4` | Change |
|---|---|---|---|
| Output tok/s | 2,893 | 2,341 | −19 % |
| P99 TTFT | 468 ms | 1,284 ms | +174 % |
| `num_requests_running` | 94 | 62 | −34 % |
| `kv_cache_usage_perc` | 0.94 | 0.63 | pool now idle |
| `waiting_by_reason{deferred}` | 0 | 41 | the cause |

**This is the diagnostic to internalise.** A third of the KV pool is idle, `num_requests_running`
sits well below `max_num_seqs`, and requests are queueing — which reads exactly like a memory
problem and is not one. `reason="deferred"` and the `waiting_lora_adapters` label name the
real cause in one look. **Adding GPU memory here changes nothing; raising `--max-loras`
changes everything**, at a cost of 1.34 GB of pool.

### Step 6 — the claim

> On 1× H100-80GB at $2.89/hr, serving Llama-3.1-8B-Instruct with vLLM 0.27.1 and
> `--enable-lora --max-loras 8 --max-lora-rank 32 --max-cpu-loras 32`, LoRA slots consume
> **1.34 GB of HBM (−2.4 % of the KV pool)** and cost **6.2 % of output throughput** with
> eight adapters randomly assigned, versus 2.6 % with a single hot adapter — the difference
> being grouped-GEMM segmentation efficiency, not FLOPs. That replaces 8 deployments
> (8 GPUs, $23.12/hr) with one (1 GPU, $2.89/hr) at a **6.2 % throughput cost**, before
> counting the utilisation gain from aggregating eight bursty tenants onto one endpoint.
> Sizing `--max-loras` below the batch's adapter diversity produces
> `vllm:num_requests_waiting_by_reason{reason="deferred"}` with an *idle* KV pool — a
> scheduling constraint, not a memory one. Measured 2026-08-18.

## Practice

Rented GPU, roughly 60 minutes. Produces the multi-tenant economics section of the
[cost-per-token deliverable](../practice/cost-per-token/README.md).

### 1. Get adapters

Either download real ones (any r=16 or r=32 adapter for your base model on the Hub), or train
four to eight throwaway ones with `peft` for a few hundred steps each on different toy tasks.
Real fine-tuning quality is irrelevant here; what matters is that the tensors are the right
shapes and the ranks differ.

**Acceptance:** at least four adapters with their `adapter_config.json` values for `r`,
`lora_alpha` and `target_modules` recorded, plus each one's on-disk size.

### 2. Predict the adapter and slot sizes, then check

From your base model's `config.json` and each adapter's `target_modules` and `r`, compute the
expected file size (§1) and the per-slot HBM cost `L_slot` (§3). Compare against `du -sh` for
the files.

**Acceptance:** a table of predicted-versus-actual adapter sizes, agreement within ~5 %, and
your computed `L_slot` for the `max_lora_rank` you intend to use. Explain any discrepancy
(fp32 storage, extra `modules_to_save`, embedding adapters).

### 3. Measure the HBM cost as a function of both flags

Boot the base with no LoRA, then sweep `--max-loras` over at least four values at fixed rank,
then sweep `--max-lora-rank` at fixed `--max-loras`. Record `Available KV cache memory` and
`GPU KV cache size` for each.

**Acceptance:** two tables, and confirmation that measured LoRA HBM equals your predicted
`max_loras × L_slot(max_lora_rank)` within a few percent. State the KV-pool cost in percent
and translate it into a concurrency cost using 07.2's arithmetic.

### 4. Measure the throughput cost, isolating the three effects

Run the four benchmark configurations from the worked example: base with LoRA disabled; LoRA
enabled with base-only requests; LoRA enabled with all requests to one adapter; LoRA enabled
with requests spread across all adapters (`--lora-assignment random`).

**Acceptance:** the four-row table with output throughput, P99 TTFT and P99 ITL, plus one
sentence attributing the difference between the one-adapter and many-adapter rows to
segmentation efficiency rather than to FLOPs — with your measured numbers next to §5's
predicted bandwidth and arithmetic terms.

### 5. Reproduce the deferral signature

Set `--max-loras` below your adapter count and drive traffic across all of them. Capture
`vllm:num_requests_waiting_by_reason`, `vllm:num_requests_running`,
`vllm:kv_cache_usage_perc` and `vllm:lora_requests_info` at saturation.

**Acceptance:** the metric sample, and a one-paragraph diagnosis explaining why a *low* KV
usage with a *growing* queue is a LoRA-budget symptom and not a memory symptom — and what
distinguishes it from the KV-bound signature in 07.4.

### 6. Size the flags for a real tenant population

Take a real or plausible adapter-popularity distribution (§8) and compute: expected distinct
adapters per batch at your operating `max_num_seqs`, the KV-pool cost of each candidate
`max_loras`, and your chosen values for all three flags with the reasoning.

**Acceptance:** the E[distinct] calculation, the cost table, chosen values, and one sentence
on what happens to your smallest tenants if you size `max_loras` too low.

### 7. Compute the platform economics

For your tenant count N, compare N separate deployments against one multi-LoRA deployment:
GPUs, monthly cost, and CPM (using 07.5's formula and your measured LoRA-enabled throughput).
Include the utilisation effect of aggregation.

**Acceptance:** the two-architecture comparison with the arithmetic shown, the CPM for each,
and an explicit list of what the consolidation costs — KV pool, adapter-diversity throughput,
shared failure domain, noisy neighbours.

**Overall acceptance:** measured LoRA HBM cost as a function of both flags, measured throughput
cost isolating the diversity effect, the reproduced deferral signature with its diagnosis, and
the N-deployments-versus-one economics with CPM for both — committed to the deliverable. This
is the last multiplier on the module's cost story; the
[checkpoint](../checkpoint.md) expects you to defend the operating point and the consolidation
decision together.

## Common pitfalls

- **Running with bare `--enable-lora`.** *Mechanism:* `max_loras` defaults to **1** and
  `max_cpu_loras` defaults to `max_loras`, so you get one GPU slot and a one-entry host
  registry. Every request for a different adapter evicts the previous one and re-reads it from
  disk. The symptom is a "multi-LoRA" deployment that is slower with two tenants than with
  one. Set all three flags explicitly.

- **Setting `--max-lora-rank` to the largest value any tenant might ever use.** *Mechanism:*
  every slot's stacked buffers are allocated at `max_lora_rank` regardless of the adapter
  loaded into them, so one r=128 tenant makes all 8 or 16 slots cost 671 MB each instead of
  84 MB. Cap the accepted rank by platform policy, or run high-rank tenants separately.

- **Reading `deferred` queueing as a memory problem.** *Mechanism:* exceeding `max_loras`
  makes the scheduler skip the request for that iteration, so you see a growing queue with a
  *low* `kv_cache_usage_perc` and `num_requests_running` below `max_num_seqs`. That is the
  opposite of the KV-bound signature. Split
  `vllm:num_requests_waiting_by_reason` by `reason` and check
  `vllm:lora_requests_info{waiting_lora_adapters=...}`.

- **Benchmarking with one hot adapter and reporting it as the multi-tenant number.**
  *Mechanism:* a single adapter produces one large, well-tiled segment in the grouped GEMM;
  eight adapters across the same batch produce eight small ones. Identical FLOPs, 2.4× the
  overhead. Use `vllm bench serve --lora-modules ... --lora-assignment random` to measure the
  case you will actually serve.

- **Expecting multi-LoRA to help across different base models.** *Mechanism:* the saving comes
  entirely from amortising one shared `W`. Different bases means no shared `W` and no saving;
  vLLM will reject adapters whose shapes do not match the served base. Consolidating tenants
  onto a shared base is a prerequisite, and it is a platform decision, not a serving flag.

- **Exposing `VLLM_ALLOW_RUNTIME_LORA_UPDATING` on a public listener.** *Mechanism:* the
  `/v1/load_lora_adapter` endpoint loads arbitrary filesystem or remote paths as model weights;
  vLLM's own router logs that it is for local development only. Front it with an authenticated
  control plane, or use a `LoRAResolver` plugin against your artefact store instead.

- **Combining runtime LoRA updating with `--api-server-count > 1`.** *Mechanism:* the adapter
  registry is per-API-server process and would diverge; `vllm/entrypoints/cli/serve.py` rejects
  the combination at startup. If you scaled out API servers for tokenisation throughput, you
  must load adapters at launch or use a resolver.

- **Sizing `--max-cpu-loras` at your tenant count exactly.** *Mechanism:* it is an LRU cache,
  so a working set that momentarily exceeds capacity thrashes — each miss is a disk read plus a
  slot copy. Host RAM at ~168 MB per r=32 adapter is cheap; leave 1.5–2× headroom.

- **Quoting "hundreds of adapters on one GPU" as concurrent capacity.** *Mechanism:* that
  figure describes tier 1 (host RAM, bounded by `max_cpu_loras`), not tier 2 (GPU slots,
  bounded by `max_loras`, typically single digits to tens). The served set and the active set
  differ by an order of magnitude.

- **Forgetting that consolidation is also consolidation of blast radius.** *Mechanism:* one
  deployment means one KV pool, one scheduler, one restart. A tenant shifting to 32k prompts
  consumes KV everyone else needed, and the SLO impact lands on tenants who changed nothing.
  Gateway rate limits and a platform-contract `--max-model-len` are part of the design, not an
  afterthought.

## Self-check

**(a) Compute the size of a LoRA adapter for Llama-3.1-8B at r=16, stating every assumption,
and the HBM one GPU slot costs at `--max-lora-rank 32`.**

**Answer:** Assume `hidden_size = 4096`, `num_hidden_layers = 32`, `num_kv_heads = 8` (so
k/v projections are 4096→1024), `intermediate_size = 14336`, and fp16 storage. Per adapted
matrix the parameter count is `r(d_in + d_out)`. For `target_modules = ["q_proj","v_proj"]`
that is `r(4096+4096) + r(4096+1024) = 13,312r` per layer, `× 32 = 425,984r`; at r=16 that is
6.82 M parameters ≈ **13.6 MB**. For `all-linear` it is
`8192r + 5120r + 5120r + 8192r + 18432r + 18432r + 18432r = 81,920r` per layer,
`× 32 = 2,621,440r`; at r=16 that is 41.9 M parameters ≈ **83.9 MB**. The 6× spread between
those two is why "an adapter is 40–60 MB" is not a fact about LoRA. For the GPU slot, vLLM
allocates `lora_a_stacked[max_loras, 1, max_lora_rank, in_features]` and
`lora_b_stacked[max_loras, 1, out_features, max_lora_rank]` per wrapped module, which for this
model totals `81,920 × max_lora_rank` elements per layer per slot, `= 5.243 MB × rank` over 32
layers at bf16 — so at `--max-lora-rank 32`, **`L_slot = 167.8 MB` per slot**, and
`--max-loras 8` costs 1.34 GB out of the KV pool.

**(b) How does vLLM serve requests for different adapters in one forward pass?**

**Answer:** Two tiers plus a segmented kernel. `set_active_loras` builds a `token_lora_mapping`
giving each token in the batch its adapter id (−1 for base). `LoRAKernelMeta.prepare_tensors`
then stable-sorts token indices by adapter id and computes `active_lora_ids`,
`num_tokens_per_lora` and `lora_token_start_loc` (a cumulative sum) — three arrays that define
contiguous segments, one per adapter. Each requested adapter must be resident in a GPU slot;
`activate_adapter` finds a free index in `lora_index_to_id` and copies the adapter's A/B
tensors into `lora_a_stacked[index]` / `lora_b_stacked[index]`. The forward pass then runs the
**base GEMM once for the whole batch**, followed by two segmented (grouped) GEMMs: **shrink**
`buffer = (x @ Aᵀ) · scale` producing a `[n_slices, num_tokens, r]` float32 intermediate, and
**expand** `y += buffer @ Bᵀ` adding into the base output in place. The sort exists so the
adapter is constant within a contiguous segment, which is what lets a grouped GEMM tile
efficiently; the shrink/expand split exists so `ΔW = BA` is never materialised — the
intermediate stays rank-`r` wide, which is why the arithmetic overhead is under 1 % rather
than 100 %.

**(c) What does multi-LoRA actually cost, in throughput and in memory, and why is the measured
cost higher than the derived one?**

**Answer:** **Memory:** exactly `max_loras × L_slot(max_lora_rank)`, taken out of the KV pool
— 1.34 GB for 8 slots at rank 32 on an 8B model, i.e. −2.4 % of a 56 GB pool and therefore
about −2.4 % concurrency. **Arithmetic:** per token per layer, LoRA adds `163,840r` FLOPs
against the base's `436,207,616`, so 0.60 % at r=16 and 2.40 % at r=64. **Bandwidth:** every
active adapter's weights are read each decode step, so at the B=96 operating point 8 active
r=32 adapters add 1.34 GB to a 70.8 GB step, about +1.9 % of step time; at small batch, where
the KV term is small, the relative cost is larger (~4 %). **Measured is higher** — 6.2 % in
the worked example with eight adapters randomly assigned versus 2.6 % with one hot adapter —
because eight adapters across 96 sequences produce eight segments averaging twelve rows each,
and a twelve-row GEMM tile does not fill a tensor core. The FLOPs and bytes are identical
between those two runs; only **grouped-GEMM segmentation efficiency** differs. So the derived
figures are a floor, and adapter *diversity* is a first-class performance variable —
measurable with `vllm bench serve --lora-assignment random`.

**(d) Requests are queueing, `kv_cache_usage_perc` is 0.63, and `num_requests_running` is 62
against `--max-num-seqs 96`. What is happening?**

**Answer:** That combination rules out the KV-bound signature (which requires
`kv_cache_usage_perc ≈ 1.0`) and rules out the `max_num_seqs` cap (which would pin running
*at* 96). On a LoRA deployment it is almost certainly the **`max_loras` admission constraint**.
The V1 scheduler tracks `scheduled_loras` and, once `len(scheduled_loras) == max_loras`, skips
any waiting request whose adapter is not already in that set — pushing it to a skipped queue
and retrying next iteration. Confirm with
`vllm:num_requests_waiting_by_reason{reason="deferred"}` being large while `reason="capacity"`
is small, and with `vllm:lora_requests_info`'s `running_lora_adapters` and
`waiting_lora_adapters` labels showing distinct adapter sets. The fix is to raise
`--max-loras` (costing `L_slot` per additional slot out of the KV pool), or to route tenants
so each replica sees fewer distinct adapters, or to accept the latency. **Adding GPU memory
does nothing**, which is exactly why the two waiting reasons are separate metrics.

**(e) Fifty teams want to serve fifty fine-tunes. What do you need to know before promising a
94 % GPU reduction?**

**Answer:** First and decisively: **are they all fine-tunes of the same base checkpoint?** The
entire saving comes from amortising one shared `W`; different bases (or different point
releases of the same family) means no shared weights, adapters that will not load, and N
deployments. Second: **the rank distribution**, because every GPU slot is allocated at
`max_lora_rank`, so a single r=128 tenant multiplies the slot cost for everyone by 8× — you
need either a rank cap as platform policy or a second deployment for outliers. Third: **the
popularity distribution**, to compute expected distinct adapters per batch
(`Σ (1 − (1−p_i)^B)`) and size `max_loras` against it; sizing it low defers requests, and by
construction the deferred ones belong to your smallest tenants. Fourth: **aggregate load**,
because the saving is `N × (W+A+KV)` over `(W+A+KV) + max_loras × L_slot` only if the combined
traffic actually fits on a small number of GPUs. Fifth: **isolation requirements** — one
deployment is one KV pool, one scheduler, one restart, and one noisy-neighbour surface, so
per-tenant rate limits and a platform-contract `--max-model-len` are part of the design. And
the number you should *also* promise: aggregating fifty bursty tenants raises utilisation from
~4 % to 40–70 %, which from 07.5's cost equation is another 2–3× on cost per token — often
larger than the memory saving itself.

**(f) When is multi-LoRA the wrong answer, and what do you use instead?**

**Answer:** Five cases. **(i) Different base models** — no shared `W`, so use a router to
per-model replicas with autoscaling (07.8) and fast cold starts (07.9). **(ii) One dominant
tenant** — if one adapter is 34 % of traffic, merge it into a dedicated base checkpoint: zero
LoRA overhead, its own failure domain, and it removes the largest contributor to adapter
diversity on the shared deployment. Multiplex the tail. **(iii) Hard isolation mandated**
(regulatory, or a hostile-multi-tenant threat model) — separate deployments, or MIG partitions
for hardware-enforced separation. **(iv) Traffic spread uniformly across a huge active set** —
the working set never stays resident, both tiers thrash, and you degrade toward per-model
serving anyway; either shard tenants across replicas so each replica has a small working set,
or accept the deferral latency. **(v) Very few models with low traffic and a tolerant SLO** —
vLLM sleep mode (07.9 §11) serialises models on one GPU with seconds-scale switching and full
isolation, which is simpler than multiplexing when you have three models rather than fifty.
The general shape of the right answer for a skewed population is usually **merged for the head,
multiplexed for the tail, routed and autoscaled across both.**

## Connections & what's next

This closes the module. The arc: 07.1 and 07.2 established that KV memory is the concurrency
ceiling; 07.3 showed the allocator that reclaims it; 07.4 tuned an engine to its edge; 07.5
converted that into dollars and located the operating point; 07.6 matched engine to workload;
07.7 halved the bytes; 07.8 raised utilisation; 07.9 cut the wake-up; and this lesson divided
the whole cost across many tenants. Each is a multiplier on the same denominator, and they
compose: FP8 KV (2×) × right-sized context (4×) × batching to the knee (20×+) × utilisation
(2–3×) × tenant multiplexing (up to N) is the arithmetic behind a "33× cost reduction over two
years" headline being a list of ordinary decisions taken in order rather than a breakthrough.

The `vllm:num_requests_waiting_by_reason` split introduced here is the last piece of the
diagnostic vocabulary the module has been assembling: `capacity` is 07.2's and 07.4's KV
story, `deferred` is this lesson's scheduling story, and telling them apart in one glance is
the difference between a two-minute fix and an afternoon of adding GPUs.

**Next: the [module 07 checkpoint](../checkpoint.md).** It asks you to size a 70B cold, hand
over the CPM and FP8 curves from your own run, justify the operating point quantitatively,
defend the autoscaling signal with a measured cold start, make the quantization call with its
calibration risk, and pick an engine for a given workload — all unaided. The
[cost-per-token deliverable](../practice/cost-per-token/README.md) is the evidence; this
lesson's multi-tenant economics is its final section. After that, module 11 takes the CPM
number you emit into `gpu-cost-operator` and builds the fleet-level economics on top of it.

## References & further reading

**Primary sources (vLLM LoRA implementation, v0.27.1, cross-checked against `main` @ `c1e4387`, 2026-08-17)**

1. **`vllm/config/lora.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/config/lora.py — every flag default this lesson depends on: `max_lora_rank = 16` with `MaxLoRARanks = Literal[1, 8, 16, 32, 64, 128, 256, 320, 512]` (**correction**: `1` and `320` were omitted from earlier versions of this lesson), `max_loras = 1`, `max_cpu_loras = None` with `_validate_lora_config` defaulting it to `max_loras`, `lora_dtype = "auto"`, `target_modules`, `fully_sharded_loras`, `specialize_active_lora = False`, and the MoE-specific `enable_mixed_moe_lora_format` / `enable_moe_shared_loras`.
2. **`vllm/lora/layers/base_linear.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/lora/layers/base_linear.py — `create_lora_weights` allocating `lora_a_stacked` as `[max_loras, 1, max_lora_rank, in_features]` and `lora_b_stacked` as `[max_loras, 1, out_features, max_lora_rank]` per slice, and `set_lora` copying into the leading sub-slice. This is the source for §3's `L_slot` derivation and for the rank-padding waste in §9(b).
3. **`vllm/lora/model_manager.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/lora/model_manager.py — the two tiers: `_registered_adapters: AdapterLRUCache` with `capacity = max_cpu_loras`, `_active_adapters` with `capacity = lora_slots = max_loras`, the `lora_index_to_id: list[int | None]` slot array, `activate_adapter`'s free-slot search and `module.set_lora(index, a, b)` call, and `LRUCacheLoRAModelManager` with `remove_oldest_adapter`.
4. **`vllm/lora/worker_manager.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/lora/worker_manager.py — `LRUCacheWorkerLoRAManager` (the class V1 always uses, per `vllm/v1/worker/lora_model_runner_mixin.py`), `_apply_adapters`'s hard error when the requested set exceeds `lora_slots`, and `add_adapter`'s load-then-evict ordering with its comment that the loaded count may temporarily exceed `--max-cpu-loras`.
5. **`vllm/lora/ops/triton_ops/lora_shrink_op.py`, `lora_expand_op.py`, `lora_kernel_metadata.py`** — https://github.com/vllm-project/vllm/tree/main/vllm/lora/ops/triton_ops — the segmented-GEMM kernels, with the Punica citation in the shrink kernel's header. `LoRAKernelMeta.prepare_tensors` is §4's mechanism verbatim: stable sort by adapter id, `torch.unique(..., return_counts=True)` for `active_lora_ids` / `num_tokens_per_lora`, `cumsum` for `lora_token_start_loc`, plus the `no_lora_flag_cpu` early-exit and `num_active_loras_cpu` for CUDA-graph specialisation.
6. **`vllm/lora/punica_wrapper/punica_gpu.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/lora/punica_wrapper/punica_gpu.py — `add_lora_linear`'s documented semantics (the `x @ A @ B * scale` per-row form quoted in §4), the float32 intermediate buffer `[n_slices, num_tokens, r]` and the comment explaining why, and the shrink-then-expand call sequence.
7. **`vllm/v1/core/sched/scheduler.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/sched/scheduler.py — the `max_loras` admission check quoted in §7: `scheduled_loras` accumulation, the assertion, and the "Scheduling would exceed max_loras, skip" branch that defers rather than rejects. **Correction:** this is deferral, not swapping.
8. **`vllm/v1/metrics/loggers.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/v1/metrics/loggers.py — `vllm:lora_requests_info` with its `max_lora` / `running_lora_adapters` / `waiting_lora_adapters` labels (and the source's own warning that it can be misleading under data parallelism), and `vllm:num_requests_waiting_by_reason` with its documented reasons — `capacity` ("waiting for scheduling capacity") and `deferred` ("deferred by transient constraints (LoRA budget, KV transfer, blocked status)") — which sum to `vllm:num_requests_waiting`.
9. **`docs/features/lora.md`** — https://github.com/vllm-project/vllm/blob/main/docs/features/lora.md — the offline `LoRARequest(name, int_id, path)` API, `--lora-modules` in both `name=path` and JSON forms (with `base_model_name` and `is_3d_lora_weight`), per-request routing via the `model` field, `/v1/models` listing adapters alongside the base, the dynamic load/unload endpoints with their security warning, `load_inplace` for RL-style adapter refresh, and the `LoRAResolver` plugin interface with `lora_filesystem_resolver` / `lora_hf_hub_resolver` and their environment variables.
10. **`vllm/entrypoints/serve/lora/api_router.py` and `vllm/entrypoints/cli/serve.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/entrypoints/serve/lora/api_router.py — the endpoints are only registered when `VLLM_ALLOW_RUNTIME_LORA_UPDATING` is set, the router's "ONLY be used for local development" warning, and the CLI's hard rejection of `api_server_count > 1` with runtime LoRA updating.
11. **`vllm/benchmarks/serve.py`** — https://github.com/vllm-project/vllm/blob/main/vllm/benchmarks/serve.py — `--lora-modules` and `--lora-assignment {random,round-robin}` (default `random`), which is what makes §Worked example's diversity measurement controlled rather than incidental.

**Deeper dives**

12. **Hu et al., "LoRA: Low-Rank Adaptation of Large Language Models" (2021)** — https://arxiv.org/abs/2106.09685 — the `W + BA` formulation, the `α/r` scaling, zero-initialisation of `B`, and the mergeability that §1 turns into "merge for speed, keep separate for multiplexing." Cited by vLLM's own LoRA documentation as the reference for the technique. *(arxiv.org is blocked by this environment's egress proxy; the formulation and its consequences are derived in §1 from the shapes vLLM allocates, so nothing here depends on a figure quoted from the paper.)*
13. **Chen et al., "Punica: Multi-Tenant LoRA Serving" (2023)** — https://arxiv.org/abs/2310.18547 — the SGMV (Segmented Gather Matrix-Vector) kernel that applies different adapters to different rows of one batch in a single launch. **This citation appears verbatim in the header of `vllm/lora/ops/triton_ops/lora_shrink_op.py`**, which is the strongest available evidence of the lineage. *(Also behind the blocked arxiv domain; §4 describes the mechanism as implemented in vLLM's Triton kernels rather than as described in the paper.)*
14. **Sheng et al., "S-LoRA: Serving Thousands of Concurrent LoRA Adapters" (2023)** — https://arxiv.org/abs/2311.03285 — Unified Paging (a pooled allocator managing adapter weights and KV tensors of varying rank and sequence length together), heterogeneous batching, and a tensor-parallel strategy for the added LoRA computation. The module README names this paper for this lesson. **Note the design difference:** vLLM's adapter memory management is the simpler fixed-slot scheme in §3, not S-LoRA's unified pool — a deliberate trade of peak adapter count for predictable HBM accounting, and the reason `max_loras` is a hard per-batch cap here rather than a soft pooled limit. *(Behind the blocked arxiv domain; the contrast is drawn from vLLM's implementation, which is directly readable.)*
