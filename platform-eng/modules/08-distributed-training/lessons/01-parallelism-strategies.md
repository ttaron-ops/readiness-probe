---
lesson: "08.1"
title: "Parallelism strategies: network and memory footprint"
module: "08"
concept: "Parallelism strategies: network and memory footprint"
status: not-started
est_time: "7h"
prev: null
next: "02-nccl-collectives.md"
artifacts: []
sources: 14
---

# 08.1 · Parallelism strategies: network and memory footprint

> **Concept.** Every parallelism strategy is a trade between *which collective runs*, *over which link*, and *how much GPU memory it saves* — read a run and you can predict its network and memory footprint without reading the model.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

This is lesson 1 of 8 in module 08. It is the module's foundation lesson: everything downstream — NCCL triage (08.2), the comms-bound MFU math (08.3), checkpoint sizing (08.4), failure/elasticity (08.5) — assumes you can look at a launch config and name the strategy, the collective it issues, and the link that collective rides. Nothing here is new physics; it's the vocabulary the rest of the module speaks.

What is new is the *arithmetic*. By the end of this lesson you should be able to take a parameter count, a parallelism block from a YAML file, and the HBM capacity from module 03, and answer "does this fit, and what is on the wire every step" on paper. That calculation is the input to every other lesson in this module: 08.3's MFU model needs the per-step byte count this lesson derives, and 08.4's checkpoint cost needs the optimizer-state accounting.

## Why this matters

When a training job pages you, the ticket says "OOM at step 3" or "step time doubled after we added nodes" — never "our tensor-parallel degree is wrong." You have to translate. If you can look at a launch config and say *"that's FSDP, so every layer does an all-gather over the fabric, and your NIC is the bottleneck"* or *"that's tensor parallel, it must stay inside one NVLink domain and it's spanning two,"* you diagnose in minutes what an ML engineer debugs in days.

This is also the load-bearing skill for capacity and cost. The parallelism strategy decides whether a run is bound by HBM (module 03), by NVLink (02b), or by the InfiniBand fabric — which is the difference between a node you can pack densely and one you can't. And the numbers are not small: a 405B-parameter model in mixed-precision Adam carries **6.5 terabytes of optimizer and parameter state**. Whether that lands as 50 GB per GPU (fits, plus room for activations) or 90 GB per GPU (instant OOM) is decided entirely by three integers in a YAML file. Reading those three integers correctly is a five-minute skill that saves a five-hour incident.

## What's new here (calibration)

You already have the physical layer. **02b** gave you the NVLink/NVSwitch domain and the rail-aligned fabric — the links, and the per-generation bandwidth table (NVLink 4 on Hopper: 18 links × 25 GB/s per direction = **450 GB/s per GPU per direction**, marketed as 900 GB/s bidirectional; a 400 Gb/s NDR rail NIC is **50 GB/s per direction**). **03** gave you HBM capacity and the roofline — 80 GB on an H100 SXM at 3.35 TB/s, 141 GB usable on an H200 at 4.8 TB/s, 180 GB usable on a B200 at 7.7 TB/s. **06** gave you gang scheduling and topology-aware placement — how the pod lands on the right GPUs.

This lesson adds the layer *between* the model and those links. Specifically new here:

- **The full memory accounting**, derived from the mixed-precision update rule rather than asserted — why the number is 16, 18 or 20 bytes per parameter depending on the framework's gradient dtype, and which of those bytes each ZeRO stage removes.
- **The activation term**, with the Megatron formula that predicts it from sequence length, hidden size and head count, so you can tell a "won't fit at init" OOM from a "won't fit at step 3" OOM before it happens.
- **What tensor parallelism actually splits** — the column-parallel/row-parallel matmul pair, and why the count of all-reduces per transformer block is exactly two forward and two backward, not "some."
- **The pipeline bubble derived**, not quoted, plus the two schedules (1F1B and interleaved-1F1B) that shrink it and what each costs.
- **Expert and context parallelism** as the fifth and sixth axes, because Llama 3 405B used context parallelism and every MoE run uses expert parallelism, and neither issues an all-reduce.

From the platform view you are not choosing the strategy — that is a modelling decision. You are *reading* it off a config and predicting its footprint so you can explain a failure or size a cluster. We skip ML-eng entirely: no model architecture design, no optimizer theory, no kernel authoring.

## Core concepts

### 1. The problem: what actually occupies a training GPU

Inference holds weights and a KV cache (module 03). Training holds four things, and three of them are invisible on a model card.

Start from the update rule. Mixed-precision training with Adam does not keep one copy of each parameter — it keeps several, at different precisions, because the arithmetic demands it. Walk one step:

1. The **forward and backward passes run in bf16 or fp16** — that is the whole point of mixed precision, since the tensor cores are 2× faster at 16-bit than at 32-bit (module 03.5). So you need a **16-bit copy of every parameter**: 2 bytes each.
2. The backward pass produces a **gradient for every parameter**. Frameworks differ here: PyTorch DDP accumulates in the parameter dtype (16-bit, 2 bytes); Megatron-LM's distributed optimizer accumulates into an **fp32 main-gradient buffer** (4 bytes) because summing thousands of small bf16 gradients loses bits to rounding.
3. The optimizer step *cannot* run in 16-bit. A typical update at step 100,000 changes a weight by a relative amount of order 1e-7; bf16 has ~3 decimal digits of mantissa, so `w + Δ` rounds straight back to `w` and training silently stops learning. So the optimizer keeps an **fp32 master copy** of every parameter: 4 bytes.
4. Adam itself is stateful. It keeps an exponential moving average of the gradient (**first moment, `m`**) and of the squared gradient (**second moment, `v`**), both fp32: 4 + 4 bytes.

Add them up. Using **Ψ** for parameter count:

```
                                        bytes/param
  bf16 parameters (compute copy)             2
  gradients (16-bit)                         2
  fp32 master parameters                     4
  Adam first moment  m                       4
  Adam second moment v                       4
                                        ───────────
                                            16 Ψ
```

**16 bytes per parameter is the ZeRO paper's accounting** and the number to carry as a default. The last 12 bytes — master weights plus the two Adam moments — are what ZeRO calls "optimizer states," `K = 12` in the paper's notation. DeepSpeed's own ZeRO tutorial pins this with a measurement: a 1.5B-parameter GPT-2 "consumes 18 GB" of Adam optimizer state, and 18 GB ÷ 1.5e9 = **12 bytes per parameter**, exactly. Partitioned across 8 data-parallel ranks it becomes 2.25 GB per GPU, which is what makes the model trainable on 32 GB V100s at all.

Real implementations land a little higher, and Megatron-LM documents its own table (`docs/user-guide/features/dist_optimizer.md`), where *d* is the data-parallel degree:

| Parameter / gradient dtypes | Non-distributed optimizer | Distributed optimizer (ZeRO-1) |
|---|---|---|
| fp16 params, fp16 grads | **20** bytes/param | 4 + 16/d |
| bf16 params, fp32 grads | **18** bytes/param | 6 + 12/d |
| fp32 params, fp32 grads | **16** bytes/param | 8 + 8/d |

Read the middle row, which is the modern default: 2 (bf16 param) + 4 (fp32 main grad) + 4 (fp32 master) + 4 (`m`) + 4 (`v`) = 18. The distributed column shows what stays replicated (6 = the bf16 param plus the fp32 grad) and what gets sharded (12 = master + `m` + `v`). The fp16/fp16 row is 20 because that implementation carries *both* a 2-byte gradient and a separate 4-byte fp32 main-gradient accumulation buffer.

**Use 16 Ψ for napkin math and cite the framework's table when the number has to be right.** The difference between 16 and 18 is 12% of your memory budget, which is the difference between fitting and not.

The fourth consumer is **activations** — the intermediate tensors saved during forward because backward needs them to compute gradients. Unlike the first three, activation memory does not scale with parameter count. It scales with **batch size, sequence length and depth**, which is why a job can pass its startup allocation and then OOM at step 3 when it hits its first long sequence. §5 gives the formula.

```
   WHAT SITS IN 80 GB OF HBM DURING TRAINING  —  one GPU, no parallelism
   ═══════════════════════════════════════════════════════════════════════

   ┌────┬────┬────────┬────────┬────────┐  ┌───────────────────┐  ┌──────┐
   │ P  │ G  │ master │   m    │   v    │  │   ACTIVATIONS     │  │ ctx  │
   │2B/p│2B/p│  4B/p  │  4B/p  │  4B/p  │  │  f(batch, seqlen, │  │ +NCCL│
   └────┴────┴────────┴────────┴────────┘  │     depth)        │  │ bufs │
     └──────────┬───────────────────────┘  └───────────────────┘  └──────┘
       MODEL STATE = 16 Ψ bytes                grows with the         3–5 GB
       fixed once Ψ is fixed                   *data*, not the        fixed
                                               model
     ↑ ZeRO / FSDP shards THIS               ↑ activation checkpointing,
       (÷ data-parallel degree)                sequence & context parallel
                                               shrink THIS
     ↑ TP and PP also shard this
       (÷ tp_size, ÷ pp_size)

   7B model  : 16 × 7e9   =   112 GB  ← already exceeds one H100 (80 GB)
   70B model : 16 × 70e9  = 1,120 GB  ← 14 H100s of state, before activations
   405B model: 16 × 405e9 = 6,480 GB  ← 81 H100s of state, before activations
```

Read the bottom three lines and the entire field makes sense. **A 7B model does not fit on an 80 GB GPU for training**, even though it fits four times over for inference. Everything below is a scheme for cutting `16 Ψ` down, or for splitting the compute, and each scheme buys memory with network traffic.

### 2. The four axes on one device grid

There are four ways to cut a training job across devices, plus two more that matter at frontier scale. They are orthogonal: you can apply all of them at once, and the degrees multiply to the world size.

The single most useful mental picture is all of them drawn on the same set of GPUs, because it makes clear **what each axis splits and what it replicates**.

```
  THE PARALLELISM AXES ON ONE 32-GPU GRID   (tp=4, pp=2, dp=4 → 4×2×4 = 32)
  ═════════════════════════════════════════════════════════════════════════

                    ── TENSOR PARALLEL (tp=4) ──▶  splits each WEIGHT MATRIX
                       stays inside one NVLink domain
                       all-reduce INSIDE every layer
        ┌───────────┬───────────┬───────────┬───────────┐
   │    │  GPU 0    │  GPU 1    │  GPU 2    │  GPU 3    │  ← layers 1..40,
   │ P  │ L1-40     │ L1-40     │ L1-40     │ L1-40     │    weight cols 0..3
   │ I  │ cols 0    │ cols 1    │ cols 2    │ cols 3    │
   │ P  ├───────────┼───────────┼───────────┼───────────┤
   │ E  │  GPU 4    │  GPU 5    │  GPU 6    │  GPU 7    │  ← layers 41..80
   │    │ L41-80    │ L41-80    │ L41-80    │ L41-80    │
   │ pp=2│ cols 0    │ cols 1    │ cols 2    │ cols 3   │
   ▼    └───────────┴───────────┴───────────┴───────────┘
        splits LAYERS by depth ▲   point-to-point activation sends between
                                   stage boundaries — NOT a collective

        ══════════ that whole 8-GPU block is ONE model replica ══════════

   DATA PARALLEL (dp=4): four IDENTICAL copies of the block above,
   each fed a different slice of the global batch.

     replica 0        replica 1        replica 2        replica 3
   ┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
   │ GPU 0..7  │    │ GPU 8..15 │    │GPU 16..23 │    │GPU 24..31 │
   └─────┬─────┘    └─────┬─────┘    └─────┬─────┘    └─────┬─────┘
         └────────────────┴────────────────┴────────────────┘
              ALL-REDUCE the gradients once per step, over the
              inter-node fabric.  DDP: state is REPLICATED (16Ψ each).
              FSDP/ZeRO-3: state is SHARDED (16Ψ/dp each) and
              re-gathered per layer.

   EXPERT PARALLEL (ep, MoE only): replaces the DP/TP split *inside* the
   MoE feed-forward block only. Each GPU owns a disjoint set of experts.
   Communication is ALL-TO-ALL (route tokens to their expert, route the
   results back) — twice per MoE layer, and NOT an all-reduce.

   CONTEXT PARALLEL (cp): splits the SEQUENCE dimension of the activations.
   Weights are replicated; only activations and the attention computation
   are split. Communication is point-to-point ring exchange of K/V blocks.
```

The one-glance table that falls out of the picture:

| Axis | What it **splits** | What it **replicates** | Collective per step | Primary link | Memory saved |
|---|---|---|---|---|---|
| **DP (DDP)** | the batch | *everything* (16Ψ per GPU) | 1 × all-reduce of grads | inter-node fabric | none |
| **ZeRO-1** | optimizer states | params, grads | reduce-scatter + all-gather | fabric | 12Ψ·(1−1/N) |
| **ZeRO-2** | + gradients | params | reduce-scatter + all-gather | fabric | 14Ψ·(1−1/N) |
| **ZeRO-3 / FSDP** | + parameters | nothing | ~2 all-gather + 1 reduce-scatter **per unit** | fabric or NVLink | 16Ψ·(1−1/N) |
| **TP** | every weight matrix | the batch, the layer list | ~2 all-reduce **per block** fwd + 2 bwd | **NVLink domain only** | state ÷ tp |
| **PP** | the layer list | the batch, each layer's weights | point-to-point sends only | inter-node, small | state ÷ pp |
| **EP** | MoE experts | attention + dense layers | 2 × all-to-all **per MoE layer** | NVLink then fabric | expert state ÷ ep |
| **CP** | the sequence | all weights | ring P2P exchange of K/V | NVLink preferred | activations ÷ cp |

Read a config against this table and you have predicted its footprint. The rest of this section explains each row so you can reason about cases the table does not cover.

### 3. DDP — replicate the model, all-reduce the gradients

**The problem it solves:** you have more data than one GPU can chew through, and the model fits. Nothing else.

**How it works internally.** Every GPU holds a complete replica: all 16Ψ bytes. Each processes a different micro-batch. Because they all started from identical weights and each applies the *average* of all replicas' gradients, they stay bit-identical forever — no parameter communication is ever needed, only gradient communication.

The naive implementation would be: run the whole backward pass, then all-reduce one giant gradient tensor. That is correct and slow, because the network sits idle for the entire backward pass and the GPU sits idle for the entire all-reduce. PyTorch DDP instead **buckets**: it groups gradients into buffers of a fixed size and fires the all-reduce for a bucket as soon as every gradient in it has been produced. Since backward produces gradients last-layer-first, the last layer's bucket is ready while the first layer is still computing.

```
  DDP GRADIENT BUCKETING — why the all-reduce is (mostly) free
  ═══════════════════════════════════════════════════════════

  backward compute (GPU) ──▶ time
  ├─ L80 ─┼─ L79 ─┼─ L78 ─┼─ ... ─┼─ L2 ─┼─ L1 ─┤
  │       │       │       │       │      │      │
  ▼       ▼       ▼       ▼       ▼      ▼      ▼
  bucket3 full    bucket2 full    bucket1 full  bucket0 full
     │               │               │             │
  NCCL stream:  [AR b3]────▶  [AR b2]────▶  [AR b1]──▶ [AR b0]
                                                          ↑
                            only THIS one is exposed — the rest
                            ran underneath backward compute

  Bucket size: PyTorch DDP default `bucket_cap_mb = 25` MiB
  (torch/nn/parallel/distributed.py: "If None, uses default of 25 MiB").
  Too small → many tiny collectives, latency-bound, poor bus bandwidth.
  Too large → the last bucket is huge and fully exposed.
```

**The collective:** one logical all-reduce per step, physically split into `ceil(2Ψ / 25 MiB)` bucket-sized all-reduces. For a 7B model in bf16 that is 14 GB of gradient, or about 560 buckets.

**The volume, and why DDP scales better on bandwidth than people assume.** A ring all-reduce (derived fully in 08.2) moves `2·(N−1)/N · M` bytes per GPU, where `M` is the message size. As `N` grows, `(N−1)/N → 1`, so the per-GPU byte count converges to **2M and stops growing**. Adding data-parallel replicas does not multiply your per-GPU bandwidth bill. What *does* grow with N is the number of hops and therefore the latency and the synchronisation tail. **The reason to leave DDP is memory pressure, not bandwidth scaling.**

**The link:** the gradient all-reduce spans all data-parallel replicas, so it rides the inter-node fabric (InfiniBand or RoCE) whenever replicas sit on different nodes.

**The failure mode:** DDP does nothing for memory. `16Ψ > HBM` and you get a `torch.OutOfMemoryError` during the first optimizer step — not at model construction, because the Adam moment buffers are lazily allocated on the first `.step()`. An OOM that reproducibly appears at step 1 rather than step 0 is the signature of optimizer-state allocation.

### 4. ZeRO and FSDP — shard the state, gather it just in time

**The problem it solves:** the model state does not fit. The observation ZeRO starts from is that DDP is *maximally redundant*: N GPUs each hold an identical copy of 16Ψ bytes, so `(N−1)/N` of all state memory in the cluster is a duplicate. If you already have N GPUs, you already have somewhere to put a sharded copy.

**The three stages, and what each removes.** ZeRO shards along the data-parallel dimension, in order of "how rarely is this needed":

- **Stage 1 — shard optimizer states.** Master weights and the Adam moments (12Ψ, the `K` term) are touched only inside `optimizer.step()`. Each rank keeps only its 1/N slice, updates only that slice, and then all-gathers the updated 16-bit parameters. Per-GPU state: **4Ψ + 12Ψ/N**. This is exactly Megatron's "distributed optimizer."
- **Stage 2 — also shard gradients.** A rank only needs the gradients for the parameters it will update, so instead of an all-reduce (everyone ends up with every gradient) the run does a **reduce-scatter** (everyone ends up with the summed gradients for their slice only). Per-GPU state: **2Ψ + 14Ψ/N**.
- **Stage 3 / FSDP — also shard parameters.** Now a rank holds 1/N of every weight matrix and must materialise the full weights momentarily to compute. Per-GPU state: **16Ψ/N**, which tends to zero as N grows.

**Worked, for a 7B model with N = 64 data-parallel ranks (16Ψ accounting):**

| Stage | Formula | Per-GPU state | Fits in 80 GB? |
|---|---|---|---|
| DDP | 16Ψ | 112.0 GB | no |
| ZeRO-1 | 4Ψ + 12Ψ/N | 28.0 + 1.3 = **29.3 GB** | yes |
| ZeRO-2 | 2Ψ + 14Ψ/N | 14.0 + 1.5 = **15.5 GB** | yes |
| ZeRO-3 / FSDP | 16Ψ/N | **1.75 GB** | yes, with enormous headroom |

**The mechanism, and where the network cost comes from.** A GPU cannot run a matmul against a weight matrix it only owns a quarter of. So FSDP does this per **communication group** (FSDP2's term; FSDP1 called it a "unit" or `FlatParameter`), every step:

1. Before the group's **forward**, `all_gather` the sharded parameters so every rank momentarily has the full weights. Compute. Then **free** the unsharded copy immediately — no communication needed to free.
2. Before the group's **backward**, `all_gather` the same parameters again (they were freed after forward, so they must be re-materialised). Compute the gradients.
3. After backward, `reduce_scatter` the gradients so each rank keeps only the summed slice it owns.

So where DDP issues one logical collective per step, **FSDP issues roughly two all-gathers plus one reduce-scatter per group per step** — dozens to hundreds of collectives, tightly interleaved with compute. That is the trade in one sentence: **FSDP spends network bandwidth to buy HBM.**

**How PyTorch FSDP2 actually hides that cost.** From the PyTorch docs for `torch.distributed.fsdp.fully_shard`: each call to `fully_shard` creates exactly one communication group containing every parameter in that module not already claimed by a nested call. There is **no automatic bucketing and no `bucket_cap_mb`** — the group boundaries are entirely determined by which modules you wrap, which is why the documented recommendation is to apply `fully_shard` bottom-up to every transformer layer and then to the root. If you only wrap the root, you get exactly two enormous blocking collectives per step and zero overlap, which the docs describe as "almost never what you want."

The overlap itself comes from running the collectives on separate CUDA streams:

```
  FSDP2 STEP TIMELINE — model[ m1 → m2 → m3 → m4 ], fully_shard on m2, m3, root
  ═══════════════════════════════════════════════════════════════════════════════
                time ──────────────────────────────────────────────────────▶

  FORWARD
  compute:      [wait]  [ fwd(m1)  │ fwd(m2)   │ fwd(m3,m4)          ]
  AG stream:  [AG(a,d)]   [ AG(b)  │   AG(c)   ]
                            ▲          ▲
                            │          └─ issued while fwd(m2) runs
                            └─ issued while fwd(m1) runs: the CPU runs ahead
                               of the GPU, so m2's pre-forward hook fires early

  BACKWARD  (FSDP2 additionally prefetches explicitly, and reduce-scatters
             on a third stream — no configuration required)
  compute:      [ bwd(m4,m3)   │ bwd(m2)       │ bwd(m1)             ]
  AG stream:  [AG(c)]  [ AG(b) │   AG(a,d)     ]
  RS stream:                   │[RS(c)]  [RS(b)│      RS(a,d)        ]

  WHEN THE OVERLAP FAILS you see: GPU utilisation sawtoothing to near zero
  between layers, step time ≈ compute + comms instead of max(compute, comms),
  and NCCL kernels dominating a profile. Causes: too few groups (wrapped only
  the root), a per-group message too small to reach peak bus bandwidth, or a
  fabric slow enough that AG(next) cannot finish inside fwd(current).
```

Two knobs worth naming because they appear in real configs. `reshard_after_forward=False` keeps the unsharded parameters resident between forward and backward — it skips the second all-gather entirely, trading memory back for network (this is "hybrid" / ZeRO-2-like behaviour at the group level). And `set_modules_to_forward_prefetch` issues the next group's all-gather from the *current* group's pre-forward hook rather than waiting, which rescues overlap when CPU-side overhead has eaten the run-ahead.

**FSDP2 vs FSDP1, in one paragraph, because you will see both.** FSDP1 (`FullyShardedDataParallel`) flattens and concatenates a group of parameters into a single `FlatParameter` and shards that. FSDP2 (`fully_shard`) shards each parameter individually on dim-0 using `torch.chunk(dim=0)` and represents the shards as `DTensor`s. The practical consequences: FSDP2 gives you communication-free sharded state dicts (FSDP1 needed all-gathers to produce a state dict, which is a checkpointing cost — see 08.4), deterministic memory without `limit_all_gathers`, and per-parameter behaviour like freezing or fp8 casting. torchtitan reports FSDP2 matching FSDP1's loss curve with **7% lower peak memory and higher MFU** on 8×H100 Llama-7B runs. **Teach FSDP2 as the API in new configs; recognise FSDP1 in older ones.**

**FSDP and DeepSpeed ZeRO are the same idea, not competitors.** FSDP is PyTorch's native implementation of ZeRO-3-equivalent sharding. Megatron-FSDP exposes the same three stages under `--data-parallel-sharding-strategy` with the values `optim` (ZeRO-1), `optim_grads` (ZeRO-2) and `optim_grads_params` (ZeRO-3). Debugging them as different techniques wastes time.

**HSDP — the variant you will meet at scale.** Fully sharding across 16,384 GPUs would mean an all-gather spanning the entire fabric for every layer, which is absurd. **Hybrid Sharded Data Parallel** shards within a group (typically one node's 8 GPUs, or a small number of nodes) and *replicates* across groups, so the expensive all-gather stays on NVLink and only a cheap gradient all-reduce crosses the fabric. In Megatron this is `--num-distributed-optimizer-instances > 1` plus `--outer-dp-sharding-strategy no_shard`. **When someone says "we run FSDP on 16K GPUs," they almost always mean HSDP.**

### 5. Activations — the term that OOMs you at step 3

Model state is fixed once you fix Ψ and the parallelism degrees. Activations are not: they scale with the data.

For a standard transformer layer, storing every intermediate needed by backward costs, per layer, per micro-batch, in 16-bit (Korthikanti et al., *Reducing Activation Recomputation in Large Transformer Models*, MLSys 2023):

```
  activation bytes per layer  =  s·b·h · (34 + 5·a·s/h)

    s = sequence length          b = micro-batch size
    h = hidden size              a = number of attention heads

  The 34 term  : the ~11 saved tensors of a transformer block, in bytes/element
                 (layernorm inputs, QKV inputs and outputs, the attention output
                 projection input, the two FFN activations, dropout masks, …)
  The 5·a·s/h  : the ATTENTION SCORE MATRIX — a·s·s elements per micro-batch.
                 QUADRATIC in sequence length. This is the term that explodes.
```

Substituting Llama-3-70B's shape (`h = 8192`, `a = 64`, `L = 80` layers) at `s = 8192`, `b = 1`:

```
  per layer = 8192 × 1 × 8192 × (34 + 5·64·8192/8192)
            = 67,108,864 × (34 + 320)
            = 67,108,864 × 354  =  23.8 GB   ← PER LAYER, per micro-batch
  × 80 layers                    = 1,900 GB  ← for ONE micro-batch of ONE sequence
```

1.9 TB of activations against 1.1 TB of model state. This is why *every* real config carries at least one activation-memory mitigation:

| Mitigation | What it does | Cost | Result on the formula |
|---|---|---|---|
| **Tensor parallel (t)** | splits the FFN/attention tensors | per-layer all-reduce | `sbh(10 + 24/t + 5as/(ht))` |
| **+ Sequence parallel** | also splits the layernorm/dropout regions TP left replicated | extra all-gather/reduce-scatter, cheap | `sbh(34/t + 5as/(ht))` |
| **+ Selective recompute** | recomputes only the attention-score block | ~4% extra FLOPs | **`34sbh/t`** — quadratic term gone |
| **Full activation checkpointing** | stores only layer boundaries, recomputes the rest in backward | ~30% extra FLOPs | `2sbh` per layer |
| **Context parallel (cp)** | splits the sequence dimension | ring P2P of K/V blocks | everything ÷ cp |

With TP=8, sequence parallelism and selective recompute, Llama-3-70B's per-layer activation cost becomes `34 × 8192 × 1 × 8192 / 8 = 285 MB`, and 80 layers is **22.8 GB** — from 1,900 GB to 22.8 GB, an 83× reduction, mostly from killing the quadratic term. That is the entire reason `--sequence-parallel` is documented in Megatron as "always enable when using TP."

**The platform-side tell.** You do not tune this, but you recognise its fingerprint: an OOM that appears **only at large batch or long sequence, several steps in, never at startup**, and that moves when you change `micro_batch_size` or `seq_length` but not when you change the model. That is activations. An OOM at step 0 or step 1 that is insensitive to batch size is model state, and the fix is a different sharding degree.

### 6. Tensor parallel — shard the matmul, all-reduce inside the layer

**The problem it solves:** a single weight matrix is too large for one GPU, or the per-layer compute is too slow. Splitting by layer (PP) does not help if *one layer* does not fit.

**How it works internally.** Megatron-style TP splits each of the two matmul pairs in a transformer block, and the split is chosen so that exactly one collective is needed per pair.

```
  MEGATRON TENSOR PARALLELISM — the column/row pair, tp = 2
  ═════════════════════════════════════════════════════════

  MLP block:   Y = GeLU(X · A) ,   Z = Y · B

  ── COLUMN-PARALLEL on A ──────────────────────────────────
     A is split by COLUMNS:  A = [A₀ | A₁]
     GPU0: Y₀ = GeLU(X·A₀)      GPU1: Y₁ = GeLU(X·A₁)
     ✔ No collective needed. GeLU is elementwise, so
       GeLU([XA₀ | XA₁]) = [GeLU(XA₀) | GeLU(XA₁)] exactly.
       Splitting by ROWS instead would require an all-reduce
       BEFORE the nonlinearity — that is the whole design trick.

  ── ROW-PARALLEL on B ─────────────────────────────────────
                              ┌ B₀ ┐
     B is split by ROWS:  B = │ ── │
                              └ B₁ ┘
     GPU0: Z₀ = Y₀·B₀           GPU1: Z₁ = Y₁·B₁
     Z = Z₀ + Z₁  ← PARTIAL SUMS. Neither GPU has the answer.
     ✖ ALL-REDUCE REQUIRED, right here, before the residual add.

  ── the two conjugate operators ───────────────────────────
     forward:   f = identity            g = all-reduce
     backward:  f = all-reduce          g = identity

  PER TRANSFORMER BLOCK, per micro-batch:
     attention block : 1 all-reduce fwd + 1 all-reduce bwd
     MLP block       : 1 all-reduce fwd + 1 all-reduce bwd
     ═════════════════════════════════════════════════════
     TOTAL: 2 forward + 2 backward all-reduces PER BLOCK
     × 80 blocks × micro-batches per step = thousands per step
```

**The message size, worked.** Each of those all-reduces carries the block's activation tensor: `s × b × h` elements × 2 bytes. For Llama-3-70B at `s = 8192`, `b = 1`, `h = 8192`: `8192 × 8192 × 2 = 134 MB` per all-reduce. Four per block × 80 blocks = **43 GB of all-reduce traffic per micro-batch, per step**. On NVLink at an effective 450 GB/s per GPU per direction, that is ~95 ms of wire time; on a 50 GB/s rail NIC it is ~860 ms — and it sits *on the critical path*, because the block cannot proceed until the partial sums are combined.

**Hence the rule.** TP must live inside a single NVLink/NVSwitch domain. The rule to carry is **"TP degree ≤ the NVLink domain size," not "TP degree ≤ 8."** Eight is correct on a classic HGX H100 baseboard, where four NVSwitch-3 chips give eight GPUs a non-blocking 450 GB/s-per-direction fabric (02b). On a GB200 NVL72 rack the NVLink domain spans 72 GPUs at 900 GB/s per direction, so the ceiling moves with the hardware. When you see `tensor_parallel_size: 8` in a config, that 8 *is* the NVLink domain on that fleet, and 06's topology-aware placement exists precisely to keep the group inside it.

Put a TP group across the fabric and you do not get a 10% slowdown; you get the 9× ratio above applied to a term that used to hide under compute, on every one of thousands of collectives per step. The symptom is "training got 5–10× slower after a reschedule," and the cause is placement, not code.

**Sequence parallelism is TP's mandatory companion.** TP leaves the layernorm and dropout regions replicated across the TP group — they are elementwise, so there is nothing to split by the column/row trick. Sequence parallelism splits *those* regions along the sequence dimension instead, converting the two all-reduces per block into an all-gather plus a reduce-scatter of the same total volume (an all-reduce *is* a reduce-scatter followed by an all-gather, so the wire cost is unchanged) while removing the replicated activation memory. Same network, less HBM: that is why `--sequence-parallel` is free and always on.

### 7. Pipeline parallel — split the layers, pay the bubble

**The problem it solves:** you have filled the NVLink domain with TP and the model still does not fit, or you need to span more nodes than TP can. PP splits by depth, and its communication is a **point-to-point send of one activation tensor between adjacent stages** — no collective at all, and the tensor is `s × b × h × 2` bytes, the same 134 MB as one TP all-reduce but sent *once per micro-batch per stage boundary* instead of four times per block.

**The cost is idle time, not bandwidth.** Naively, stage 0 computes micro-batch 1 and hands it to stage 1; stages 1..p−1 have nothing to do until work reaches them, and stage 0 has nothing to do once it has fed the last micro-batch. That fill-and-drain idle is the **bubble**.

**Derive its size.** Let `p` = number of stages, `m` = micro-batches per step, and let one stage's forward+backward on one micro-batch take unit time. The pipeline needs `p−1` unit-times to fill and `p−1` to drain, but fill and drain overlap in the standard schedules, so:

```
   ideal time (perfect pipeline) = m                (each stage does m units)
   actual time                   = m + (p − 1)      (plus the fill/drain tail)

                     (m + p − 1) − m        p − 1
   bubble fraction = ───────────────── = ───────────
                        m + p − 1         m + p − 1
```

| p | m | bubble | reading |
|---|---|---|---|
| 4 | 4 | 3/7 = **43%** | catastrophic — never run m ≈ p |
| 4 | 16 | 3/19 = **16%** | tolerable |
| 4 | 64 | 3/67 = **4.5%** | good |
| 16 | 64 | 15/79 = **19%** | deep pipelines need m ≫ p |
| 16 | 256 | 15/271 = **5.5%** | Llama-3-405B territory |

**The rule of thumb that falls out: keep `m ≥ 4p`, and preferably `m ≥ 8p`.**

**The three schedules, drawn.**

```
  PIPELINE SCHEDULES, p = 4 stages, m = 8 micro-batches
  F = forward on a micro-batch   B = backward   · = IDLE (the bubble)
  ══════════════════════════════════════════════════════════════════════════

  (a) GPipe — all forwards, then all backwards
      stage0: F1 F2 F3 F4 F5 F6 F7 F8 ·  ·  ·  B8 B7 B6 B5 B4 B3 B2 B1
      stage1: ·  F1 F2 F3 F4 F5 F6 F7 F8 ·  B8 B7 B6 B5 B4 B3 B2 B1 ·
      stage2: ·  ·  F1 F2 F3 F4 F5 F6 F7 B8 B7 B6 B5 B4 B3 B2 B1 ·  ·
      stage3: ·  ·  ·  F1 F2 F3 F4 F5 F6 B6 B5 B4 B3 B2 B1 ·  ·  ·  ·
      bubble = (p−1)/(m+p−1) = 3/11 = 27%
      PEAK ACTIVATION MEMORY: stage0 must hold ALL m micro-batches'
      activations until their backwards run.  m × per-microbatch. Brutal.

  (b) 1F1B — steady state alternates one forward, one backward
      stage0: F1 F2 F3 F4 B1 F5 B2 F6 B3 F7 B4 F8 B5 B6 B7 B8
      stage1: ·  F1 F2 F3 B1 F4 B2 F5 B3 F6 B4 F7 B5 F8 B6 B7 B8
      stage2: ·  ·  F1 F2 B1 F3 B2 F4 B3 F5 B4 F6 B5 F7 B6 F8 B7 B8
      stage3: ·  ·  ·  F1 B1 F2 B2 F3 B3 F4 B4 F5 B5 F6 B6 F7 B7 F8 B8
                      └── steady state: F,B,F,B ... ──┘
      SAME bubble as GPipe — 3/11.  The win is MEMORY: a stage holds at
      most (p − stage_id) micro-batches in flight, not m.  Stage 0 holds 4,
      stage 3 holds 1.  This is why 1F1B, not GPipe, is the production default.

  (c) INTERLEAVED 1F1B (virtual pipeline, v=2 chunks per device)
      Each device owns TWO non-contiguous chunks: device 0 holds
      layers 1-10 AND layers 41-50, device 1 holds 11-20 and 51-60, ...
      The pipeline is now effectively p·v = 8 stages deep over 4 devices,
      so a micro-batch traverses it twice — twice the P2P sends —
      but each traversal is half as long, so the fill/drain shrinks:

            bubble = (p − 1) / (v·m + p − 1) = 3/19 = 16%     (v = 2)

      TRADE: bubble ÷ v,  point-to-point message COUNT × v.
      Worth it when the fabric is fast (the sends are small) and p is large.
      Megatron: --num-layers-per-virtual-pipeline-stage
      (schedules.py: total_num_microbatches = num_microbatches × num_model_chunks)
```

**The platform-side tell.** A PP run that "wastes" 20% of GPU-time is not hung and not broken — it has too few micro-batches. The signature is **low, evenly distributed utilisation across a contiguous group of GPUs, with the step counter still advancing**, and it is *periodic* at the step boundary. A hang, by contrast, pins utilisation at 100% and freezes the step counter (08.2). Do not page the ML team for a bubble; tell them `m` is too small relative to `p`.

**PP is not a strategy to avoid.** When a model does not fit even after TP has filled the NVLink domain and FSDP has sharded everything else, PP is the only remaining axis. The bubble is a tunable cost, not a reason to skip PP.

### 8. Expert parallel — the all-to-all axis

Mixture-of-Experts models replace the dense FFN in some layers with `E` expert FFNs plus a router that sends each token to its top-`k` experts. Only `k/E` of the expert weights are touched per token, so the model has far more parameters than FLOPs — and far more parameters than one GPU can hold.

**Expert parallelism** gives each GPU a disjoint subset of the experts. The communication is neither an all-reduce nor an all-gather:

1. The router decides, per token, which expert it goes to. Those experts live on other GPUs.
2. **all-to-all #1 (dispatch):** every rank sends each of its tokens to the rank owning that token's expert. Every rank sends a *different* payload to every other rank.
3. Each rank runs its local experts on whatever arrived.
4. **all-to-all #2 (combine):** the results are routed back to the ranks that own those tokens.

Two all-to-alls per MoE layer, per micro-batch. Two properties matter operationally:

- **All-to-all is the most topology-sensitive collective.** Every rank talks to every other rank simultaneously, so it saturates bisection bandwidth rather than per-link bandwidth. Within an NVSwitch domain that is fine (the fabric is non-blocking by design). Across a fabric it is the pattern most likely to congest.
- **All-to-all is load-imbalanced by construction.** If the router happens to send 30% of tokens to one expert, that expert's GPU does 30% of the work while others idle, and *every* rank waits at the second all-to-all. The symptom is a persistent straggler that is not a hardware fault — the fix is an auxiliary load-balancing loss or expert capacity limits, i.e. a modelling change, not an infrastructure one. **Recognising "straggler, but the hardware is healthy" as MoE imbalance rather than a sick node is a real on-call skill** (08.2 shows how NCCL's RAS distinguishes a lagging rank from a missing one).

Megatron requires `--sequence-parallel` whenever TP and EP are combined, and exposes `--expert-model-parallel-size` alongside `--num-experts`. Reference configurations from Megatron's own parallelism guide: Mixtral 8×7B on 64 GPUs at `TP=1, PP=4, EP=8`; DeepSeek-V3 671B on 1024 GPUs at `TP=2, PP=16, EP=64`.

### 9. Context parallel — the sequence axis

The activation formula in §5 has a term quadratic in `s`. At 128K context that term dominates everything else, and no amount of TP fixes it, because TP divides by `t` while the term grows as `s²`.

**Context parallelism** splits the sequence dimension across GPUs: each rank holds `s/cp` tokens' worth of activations for *every* layer, with the weights fully replicated. Attention is the only operation that needs tokens it does not own, so the ranks exchange K/V blocks in a ring (each rank passes its K/V to the next while computing on what it received) — point-to-point, overlappable, and `O(cp)` steps rather than a collective.

Llama 3 405B used `CP = 2` alongside `TP = 8` and `PP = 16`, which is what let the same infrastructure train at both 8K and 128K+ sequence length. Megatron's own guidance is blunt about when to reach for it: **"Use CP instead of larger TP for long sequences."** The reason is the collective count — TP costs four all-reduces per block, CP costs a ring exchange only inside attention.

### 10. Composing them — the mesh, the ordering, and the arithmetic

Real frontier runs stack all of these. The degrees multiply to the world size:

```
   world_size  =  TP × CP × PP × EP × DP
```

and the **order in which ranks are assigned to axes is a placement decision that matters more than the degrees themselves.** The rank ordering is chosen so the most communication-intensive axis is the most local:

```
  RANK-TO-AXIS MAPPING on a 512-GPU fleet, tp=8, cp=1, pp=4, dp=16
  ════════════════════════════════════════════════════════════════

  global rank = (((dp_rank × pp_size) + pp_rank) × tp_size) + tp_rank
                                                    └────┬────┘
                                        TP is the FASTEST-VARYING index
                                        → tp ranks 0..7 are consecutive
                                        → consecutive ranks land on the
                                          SAME NODE (8 GPUs per node)
                                        → the per-layer all-reduce never
                                          leaves the NVSwitch

  node 0  : ranks 0..7    = dp0/pp0/tp0..7   ← one TP group, one NVLink domain
  node 1  : ranks 8..15   = dp0/pp1/tp0..7   ← next PIPELINE stage, adjacent node
  node 2  : ranks 16..23  = dp0/pp2/tp0..7
  node 3  : ranks 24..31  = dp0/pp3/tp0..7   ← one full model replica = 4 nodes
  node 4  : ranks 32..39  = dp1/pp0/tp0..7   ← second replica starts here
   ...
  DP groups are ranks {0, 32, 64, 96, ...} — one GPU per replica, spread
  across the whole fleet. Their all-reduce is the only traffic that must
  cross the spine, and it happens ONCE per step. It is also the collective
  that RAIL ALIGNMENT (02b) exists to make fast: rank 0 on every node talks
  to rank 0 on every other node, so if GPU0 egresses on NIC0 on every node,
  that entire collective stays on one leaf switch.
```

**Why more axes is not automatically better.** Each additional split adds collectives per step. Composing TP+PP+DP+FSDP is worth it only when memory genuinely does not fit a simpler layout. Over-sharding does not produce a correctness bug — it produces low MFU (08.3), which is easy to miss until someone checks the number.

**The composition worked, for Llama-3-70B on 512 H100s at `tp=8, pp=4, dp=16`, 16Ψ accounting:**

```
  MODEL STATE
    total state          = 70e9 × 16 B            = 1,120 GB
    ÷ tp (8)             → each GPU holds 1/8 of every weight matrix
    ÷ pp (4)             → each GPU holds 1/4 of the layers
    per-GPU state        = 1,120 / (8 × 4)        =    35.0 GB
    (dp does NOT divide state under plain DDP — it replicates)

  ACTIVATIONS  (s=8192, b=1, h=8192, a=64, L=80, TP+SP+selective recompute)
    per layer            = 34 · s · b · h / t
                         = 34 × 8192 × 1 × 8192 / 8   =  0.285 GB
    layers per stage     = 80 / 4                     =  20
    micro-batches in flight on stage 0 under 1F1B     =  p = 4
    activation memory    = 0.285 × 20 × 4             =  22.8 GB

  FRAMEWORK + NCCL BUFFERS                            ≈   4.0 GB
    (CUDA context, allocator reserve, NCCL_BUFFSIZE default 4 MiB
     per channel per peer, fragmentation)

  ────────────────────────────────────────────────────────────────
  TOTAL per GPU          = 35.0 + 22.8 + 4.0        =  61.8 GB
  H100 SXM capacity                                 =  80.0 GB
  HEADROOM                                          =  18.2 GB  ✔ fits
  ────────────────────────────────────────────────────────────────

  SENSITIVITY — the two numbers that move it:
    micro-batch 1 → 2   : activations ×2 → 84.6 GB total → OOM at step 3
    switch dp to FSDP   : state 35.0 → 2.2 GB → total 29 GB, but adds
                          ~2 all-gather + 1 reduce-scatter per group per
                          step across 16 replicas, on the fabric
```

**That last block is the whole lesson.** You did not read a line of model code, and you can now say: it fits with 18 GB of headroom, the headroom is entirely consumed by doubling the micro-batch, the NVLink domain is load-bearing because TP=8, and the fabric carries exactly one all-reduce per step.

### 11. Reading the strategy off a job you inherited

You will often be handed a running job with no config in front of you. The reliable tells, in order of how fast you can get them:

| Signal | Reading | Strategy |
|---|---|---|
| `nvidia-smi --query-gpu=memory.used` across all GPUs | near-identical, *high*, > half of HBM | replication → **DDP** |
| same | near-identical, *low* and falling as world size grows | sharding → **FSDP/ZeRO-3** |
| same | differs systematically between contiguous groups of 8 | **PP** (stage 0 holds more in-flight activations than stage p−1 under 1F1B) |
| `NCCL_DEBUG=INFO` + `NCCL_DEBUG_SUBSYS=COLL,TUNING` | many small `AllReduce` per step, GPUs in lockstep, all on `via P2P` | **TP** |
| same | one large `AllReduce` burst at the step boundary, `via NET/IB` | **DDP** |
| same | repeated `AllGather` + `ReduceScatter` pairs per layer | **FSDP** |
| same | `AllToAll` entries | **EP / MoE** |
| GPU utilisation shape | evenly *low* across a contiguous group, step counter advancing | **PP bubble**, `m` too small |
| same | sawtooth synchronised across all GPUs at step boundaries | DDP/FSDP step boundary |
| same | pinned at 100%, step counter frozen | **not a parallelism issue — a hang (08.2)** |

You are not reverse-engineering the model. You are reading the *footprint* the strategy leaves, which is exactly what you need to explain a failure or size capacity.

## Perspectives

**Developer / ML-eng view.** The choice between "just wrap it in FSDP" and hand-tuned 5D parallelism is a choice between convenience and minimising communication at extreme scale. FSDP is the default answer when a model does not fit and you want it to work: shard everything, let prefetch hide the cost, accept that you are issuing hundreds of collectives per step. Hand-tuned TP+CP+PP+DP is what teams reach for when FSDP alone would make the fabric the bottleneck — Llama 3, BLOOM and OPT all needed the extra dimensions. The ML engineer's mental model is "which knob makes the loss curve converge fastest per dollar"; yours is "which knob puts which bytes on which wire."

**Platform / operator view.** The parallelism degrees in a launch config are not just numbers — they are a **placement constraint you must honour**. TP degree dictates how large an NVLink domain the scheduler must carve out and keep intact; PP degree dictates how many nodes must be co-scheduled contiguously so activation sends stay cheap; DP rank ordering dictates whether the once-per-step gradient all-reduce is rail-aligned or spine-crossing. Get placement wrong — a TP group split across a node boundary, a PP stage scattered non-contiguously — and the strategy collapses regardless of how correct the model code is. This is exactly where 02b's rail alignment and 06's topology-aware placement stop being background material and become the thing standing between you and a working run.

**Hardware / topology view.** Every "rule" in this lesson is really a hardware fact wearing a software costume. "TP ≤ 8" is "TP ≤ the NVSwitch domain," which is 8 on an HGX H100 baseboard and 72 on a GB200 NVL72 rack. "FSDP is fine intra-node, expensive inter-node" is the 450 GB/s vs 50 GB/s ratio from 02b. "Keep `m ≥ 4p`" is a statement about how many activation tensors will fit in HBM. Teaching the numbers rather than the ratios will mislead you the first time you touch a rack built after the generation you learned on; teaching the ratios generalises.

**Economics view.** Every strategy is a memory-for-network trade, and network is the scarcer, more failure-prone resource at scale. The strategy that looks cheapest on paper — maximal FSDP sharding, minimal replication — issues the most collectives per step and is therefore the most exposed to a flaky fabric. A run that is "optimally" memory-efficient but comms-fragile can lose more wall-clock to stalls and hangs (08.2, 08.5) than a less elegant, more replicated layout would have. Cheapest-in-theory and cheapest-in-practice diverge exactly at the point where the network stops being reliable — which, at 16K GPUs, is roughly every three hours.

## Real-world use cases

- **Meta — Llama 3 405B, 4D parallelism on 16K H100s.** The published configuration is **TP=8, CP=2, PP=16**, with data parallelism (FSDP-style sharding) filling the remaining dimension; reported throughput is ~400 TFLOP/s per GPU at 8K sequence length and 380 TFLOP/s at 131K, for **38–43% BF16 MFU**. The paper notes MFU dips from 43% at 8K GPUs (DP=64) to 41% at 16K GPUs (DP=128) purely because the per-DP-group batch shrinks to hold the global token count constant. *What it shows:* the exact composition this lesson teaches you to read, at the largest published scale — and that CP exists specifically to break the `s²` activation term rather than to save weights.

- **Hugging Face / BigScience — BLOOM-176B on 384 A100s.** Megatron-DeepSpeed: Megatron-LM tensor parallelism composed with DeepSpeed ZeRO sharding and pipeline parallelism, documented in a public day-by-day chronicle. *What it shows:* a clean, fully written-up example of composing three strategies from two different codebases, including the reasoning for each degree — the closest thing to a worked answer key for the config-reading skill above.

- **Meta — OPT-175B, 992 A100s, FSDP + Megatron TP.** The chronicles record at least 35 manual restarts and 100+ hosts cycled over two months of training. *What it shows:* the memory/network trade made in the launch config directly shaped the failure surface the team lived with — more sharding meant more collectives meant more opportunities for one rank's problem to become everyone's hang. This is the bridge into 08.5.

- **Imbue — 4,088 H100s / 511 nodes for a 70B model.** A from-scratch cluster build-out with fully non-blocking 3-tier InfiniBand and, by their own account, more than 12,000 network connections to keep healthy. *What it shows:* the placement-constraint view is not theoretical — at this scale one flaky IB link degrades the entire run, and the parallelism layout determines *which* link being flaky matters.

- **Megatron-LM's own reference configurations.** From `docs/user-guide/parallelism-guide.md`: Llama-3 8B on 8 GPUs at `TP=1, PP=1, CP=2`; Llama-3 70B on 64 GPUs at `TP=4, PP=4, CP=2`; Llama-3.1 405B on 1024 GPUs at `TP=8, PP=8, CP=2`; GPT-3 175B on 128–512 GPUs at `TP=4, PP=8`; DeepSeek-V3 671B on 1024 GPUs at `TP=2, PP=16, EP=64`. *What it shows:* TP tracks the NVLink domain, PP grows with depth, CP appears wherever the context is long, and EP appears only for MoE — the table is the rules of this lesson, applied.

## Worked example

**The ticket:** "We want to train Llama-3.1-405B. Ops says we have 1,024 H100 SXM (80 GB). The ML team proposed `tp=8, pp=8, cp=2, dp=8`. Will it fit, where does the network pressure go, and what breaks first?"

**Step 1 — check the arithmetic.** `8 × 8 × 2 × 8 = 1,024`. ✔ The degrees multiply to the world size. If they had not, the launcher would either hang at rendezvous or silently under-allocate; catching this before submission is free.

**Step 2 — model state per GPU.**

```
  total state (16 Ψ)     = 405e9 × 16 B              = 6,480 GB
  sharded by TP × PP     = 8 × 8 = 64 ways
  (CP replicates weights; DP under plain DDP replicates too)
  per-GPU state          = 6,480 / 64                =   101.3 GB
                                                        ▲
                                    EXCEEDS 80 GB. Does not fit. ✖
```

**Flag it before launch.** Two ways out, and they have different costs:

```
  Option A — shard the optimizer across DP (ZeRO-1 / distributed optimizer)
    replicated part  = 4Ψ / 64      = 405e9 × 4 / 64      = 25.3 GB
    sharded part     = 12Ψ / (64×8) = 405e9 × 12 / 512    =  9.5 GB
    per-GPU state                                          = 34.8 GB  ✔
    network cost: the DP gradient all-reduce becomes a reduce-scatter
    plus an all-gather of the same total volume — essentially free.

  Option B — full FSDP/ZeRO-3 across DP
    per-GPU state    = 16Ψ / (64×8) = 405e9 × 16 / 512    = 12.7 GB  ✔
    network cost: ~2 all-gathers + 1 reduce-scatter PER GROUP PER STEP
    across the 8 DP replicas, on top of everything else.

  Take Option A. It buys 66 GB of headroom for the cost of a
  collective the run was already paying for in a different shape.
  Option B buys another 22 GB you do not need and adds hundreds of
  collectives per step. Do not over-shard.
```

**Step 3 — activations, at `s = 8192`, `b = 1`, with TP+SP+selective recompute.** Llama-3.1-405B's shape is `h = 16384`, `L = 126`, `a = 128`.

```
  per layer  = 34 · s · b · h / t
             = 34 × 8192 × 1 × 16384 / 8            = 0.570 GB
  layers per pipeline stage = 126 / 8               = 15.75 → 16
  in-flight micro-batches on stage 0 under 1F1B     = p = 8
  ÷ cp (context parallel halves the sequence)       = 2

  activation memory = 0.570 × 16 × 8 / 2            = 36.5 GB
```

**Step 4 — the budget, and what breaks first.**

```
                                          per GPU
  model state (Option A, ZeRO-1)          34.8 GB
  activations (b=1, s=8192)               36.5 GB
  framework + CUDA ctx + NCCL buffers       4.0 GB
  ──────────────────────────────────────────────────
  total                                   75.3 GB
  H100 SXM capacity                       80.0 GB
  headroom                                 4.7 GB   ← thin

  WHAT BREAKS FIRST: micro-batch size. b=2 adds another 36.5 GB
  and OOMs immediately. So this configuration is pinned at b=1,
  which means the global batch is 8 (dp) × 1 = 8 sequences per
  step unless gradient accumulation raises the micro-batch count.
```

**Step 5 — where the network pressure goes.**

```
  ── TP (inside each node's 8 GPUs, over NVLink) ────────────────────
     message  = s × b × h × 2 B / (cp) = 8192 × 16384 × 2 / 2 = 134 MB
     count    = 4 per block × 126 blocks             = 504 all-reduces
     volume   = 504 × 134 MB                         = 67.6 GB per
                                                       micro-batch
     at 450 GB/s per GPU per direction, algbw ≈ 450 / 1.75 = 257 GB/s
     time     = 67.6 / 257                           = 263 ms
     ⚠ If placement splits ANY TP group across two nodes, that
       257 GB/s becomes 50/1.75 = 28.6 GB/s and the term jumps to
       2.4 s — a 9× step-time regression that looks like "slow after
       a reschedule." This is the single highest-value thing to verify.

  ── PP (between the 8 stages, point-to-point) ──────────────────────
     message  = 134 MB per micro-batch per boundary, 7 boundaries
     cheap in bandwidth; the cost is the BUBBLE.
     with m micro-batches: bubble = (8−1)/(m+7)
       m = 8   → 47%  ✖    m = 32  → 18%      m = 64  → 10%
       m = 128 → 5.2% ✔    ← this is the number to ask the ML team for
     Or enable interleaving (v=2): bubble = 7/(2m+7), halving it at
     the cost of doubling the P2P send count.

  ── DP (across the 8 replicas, over the fabric) ────────────────────
     gradient volume  = 405e9 / 64 shards × 4 B (fp32 main grad)
                      = 25.3 GB per GPU
     ring reduce-scatter + all-gather ≈ 2 × (7/8) × 25.3 = 44.3 GB
     at 50 GB/s per rail NIC, algbw ≈ 50 / 1.75 = 28.6 GB/s
     time     = 44.3 / 28.6                          = 1.55 s per step
     ⚠ THIS is why rail alignment matters: DP rank k on every node
       must reach DP rank k elsewhere. Rail-aligned, it stays on one
       leaf switch. Mis-aligned, it crosses the spine and contends.
```

**The verdict you deliver, without having read the model code:** it fits with ZeRO-1 and 4.7 GB of headroom, it is pinned at micro-batch 1, it needs at least 128 micro-batches per step or the pipeline bubble eats 20%+, the TP groups must not cross a node boundary under any circumstance, and the DP all-reduce is ~1.5 s per step so rail alignment is worth real money. That is a full capacity and placement review from a parameter count and four integers.

## Practice

**Environment:** one node, **2 rented GPUs** (2× A100/H100 ideal; 2× L4/L40S works and is cheap), single node so NVLink or PCIe is the only link. PyTorch ≥ 2.4 for FSDP2 (`fully_shard`), `torchrun`.

The goal is to make the memory-for-network trade **visible in numbers you measured yourself**, because that is the baseline every later lesson's cost model builds on.

1. **Build a deliberately state-heavy model** so the 16Ψ term dominates activations and the comparison is clean: a stack of large `nn.Linear` layers — e.g. 12 × `Linear(8192, 8192)` ≈ 806M parameters — with `torch.optim.AdamW`. Print `sum(p.numel() for p in model.parameters())` and record it.

2. **Predict before you measure.** Write down, from §1, what you expect per-GPU peak memory to be for DDP (`16Ψ`) and for FSDP2 with 2 ranks (`16Ψ/2` plus the transient full-parameter buffer for the largest communication group). Getting this wrong by more than ~20% means you have missed a term — find it before continuing.

3. **Run it twice**, same model, same batch, once wrapped in `DistributedDataParallel` and once with `fully_shard` applied per layer *and then* to the root (per the FSDP2 contract in §4 — wrapping only the root gives you no overlap and a misleading memory number). Launch each with `torchrun --nproc_per_node=2 train.py`.

4. **Capture peak memory per GPU** in the steady state — `torch.cuda.max_memory_allocated()` in-code is more reliable than `nvidia-smi`, which includes the allocator's reserve. Reset with `torch.cuda.reset_peak_memory_stats()` after warmup so you measure steady state, not initialisation.

5. **Count the collectives.** Set `NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=COLL` (foreshadowing 08.2) and count the `AllReduce` / `AllGather` / `ReduceScatter` lines per step for each run. You do not need to parse every line — note the *pattern and count difference*: DDP shows a burst of bucket-sized all-reduces at the step boundary; FSDP shows all-gather/reduce-scatter pairs interleaved through forward and backward.

6. **Time both.** Median step time over ≥100 steady-state steps, discarding warmup. On two NVLinked GPUs FSDP's extra collectives are nearly free; on two PCIe-only GPUs they are not, and the delta is the lesson.

**Expected result:** FSDP2's peak memory is markedly lower than DDP's (roughly half the state, plus one group's unsharded transient), and FSDP2 issues many more collectives per step. That is the memory-for-network trade made visible on two GPUs.

**Acceptance (feeds "Survive-a-failure"):** a short note (10–15 lines) recording the **parameter count**, your **predicted** and **measured** peak memory for DDP and FSDP2, the **collective count per step** for each, the **median step time** for each, and one sentence naming the link those collectives rode (NVLink, PCIe, or SHM — read it from the NCCL log). Commit it under [`../practice/survive-a-failure/`](../practice/survive-a-failure/README.md) — this is the memory-footprint baseline the deliverable's checkpoint sizing builds on, since checkpoint bytes are exactly the model-state bytes you just measured.

## Common pitfalls

- **"FSDP and DeepSpeed ZeRO are different things."** They are not. FSDP is PyTorch's native implementation of ZeRO-3-equivalent sharding; Megatron-FSDP exposes the same three stages under `--data-parallel-sharding-strategy optim / optim_grads / optim_grads_params`. *Mechanism:* all three shard the same 16Ψ along the data-parallel dimension and re-materialise on demand. Debugging them as competing approaches sends you looking for differences that are naming, not behaviour.

- **"16 bytes per parameter, always."** It is 16 in the ZeRO paper's fp16/fp16 accounting, 18 in Megatron's bf16-params/fp32-grads default, and 20 in fp16-params with a separate fp32 main-gradient buffer. *Mechanism:* the variation is entirely in how many *gradient* copies the framework keeps and at what precision. A 12% error in bytes-per-parameter is a 12% error in your fit decision, which is often the whole margin.

- **"More parallelism dimensions is always better."** Each additional split issues more collectives per step. *Mechanism:* the collectives are barriers, so their cost is `max` over ranks, not `mean` — every extra collective is another opportunity for the slowest rank to set the pace. Over-sharding shows up as low MFU (08.3), not as a correctness bug, so it survives code review and dies in the invoice.

- **"TP degree is capped at 8 because that's standard."** It is capped by the NVLink/NVSwitch domain size — 8 on a classic HGX baseboard, 72 on a GB200 NVL72. *Mechanism:* TP's per-block all-reduce is on the critical path and 9× more expensive over a 50 GB/s rail NIC than over 450 GB/s of NVLink, so the ceiling is wherever NVLink stops. Treating "8" as a constant will mislead you on the first rack-scale system you touch.

- **"Pipeline bubble is wasted GPU-time, so avoid PP."** *Mechanism:* the bubble is `(p−1)/(m+p−1)` and shrinks with micro-batch count; at `m = 8p` it is under 12%, and interleaving divides it by the virtual-chunk count. Meanwhile PP is often the *only* remaining axis once TP has filled the NVLink domain. Refusing PP because of the bubble means refusing to train the model.

- **"DDP doesn't scale as well as FSDP."** *Mechanism:* the ring all-reduce's per-GPU byte count is `2(N−1)/N · M`, which converges to `2M` and stops growing with N. DDP's bandwidth cost per GPU is essentially flat in world size. The reason to leave DDP is that `16Ψ` does not fit — a memory reason, not a scaling reason.

- **"An OOM means we need more sharding."** *Mechanism:* an OOM at step 0–1 that is insensitive to batch size is model state, and sharding fixes it. An OOM at step 3+ that moves with `micro_batch_size` or `seq_length` is activations, and sharding the *state* will not touch it — you need activation checkpointing, sequence parallelism, or context parallelism. Applying the wrong fix costs a day and a rebuild.

## Self-check

- **Why does FSDP trade network for memory versus DDP?**
  **Answer:** DDP keeps a full replica — all 16Ψ bytes of parameters, gradients, master weights and Adam moments — on every GPU, and pays exactly one logical gradient all-reduce per step (bucketed at 25 MiB by default so it overlaps backward). Cheap network, zero memory saving. FSDP shards parameters, gradients and optimizer states along the data-parallel dimension so each rank holds ~16Ψ/N, but a rank cannot run a matmul against a weight matrix it owns a slice of. So before every communication group's forward it must `all_gather` the full parameters, free them after, `all_gather` them *again* before backward, and `reduce_scatter` the gradients afterwards — roughly two all-gathers plus one reduce-scatter per group per step instead of DDP's single collective. It spends bandwidth (hundreds of extra collectives) to buy HBM (state divided by N). On NVLink that trade is nearly free; across a 50 GB/s fabric it can dominate step time, which is why HSDP (shard within a node, replicate across nodes) is what large fleets actually run.

- **Which parallelism strategy must stay inside one NVLink domain, and why (tie to 02b)?**
  **Answer:** **Tensor parallel.** Megatron TP splits each weight matrix column-wise then row-wise; the row-parallel half leaves every GPU holding a *partial sum*, so an all-reduce is mandatory before the residual add. That is 2 all-reduces forward and 2 backward per transformer block, each carrying `s × b × h × 2` bytes — 134 MB for a 70B model at 8K sequence — on the critical path, thousands of times per step. On NVLink 4 the per-GPU ceiling is 450 GB/s per direction; on a 400 Gb/s rail NIC it is 50 GB/s. Splitting a TP group across a node boundary applies that 9× ratio to a term that previously hid under compute, and step time collapses 5–10×. The rule is **TP degree ≤ NVLink domain size** — 8 on a classic HGX H100 baseboard (four NVSwitch-3 chips, non-blocking across 8 GPUs), 72 on a GB200 NVL72 rack — and 06's topology-aware placement exists to enforce it.

- **What is the pipeline bubble, what determines its size, and how do the schedules differ?**
  **Answer:** The bubble is the idle GPU-time while the pipeline **fills** (downstream stages wait for the first micro-batch to arrive) and **drains** (upstream stages finish early). With `p` stages and `m` micro-batches, ideal time is `m` unit-steps and actual time is `m + p − 1`, so the bubble fraction is `(p − 1)/(m + p − 1)`. At `p=4, m=4` that is 43%; at `p=4, m=64` it is 4.5%. **GPipe** runs all forwards then all backwards — same bubble, but stage 0 must hold all `m` micro-batches' activations, which is memory-prohibitive. **1F1B** alternates one forward and one backward in steady state: identical bubble, but a stage holds at most `p − stage_id` micro-batches in flight, which is why it is the production default. **Interleaved 1F1B** gives each device `v` non-contiguous chunks, making the pipeline `p·v` stages deep so the bubble becomes `(p − 1)/(v·m + p − 1)` — divided by `v`, at the cost of `v`× as many point-to-point sends. The operational signature of a bubble is low, *evenly distributed* utilisation with the step counter still advancing — never a hang.

- **A launch config shows `tensor_parallel_size: 16` on a fleet of classic 8-GPU HGX nodes. What do you flag before this job runs?**
  **Answer:** TP=16 exceeds the NVLink domain size on this hardware (8 GPUs per HGX baseboard), so the TP group cannot fit inside one NVSwitch domain and *must* span two nodes. Half of every per-block all-reduce will therefore traverse InfiniBand at ~50 GB/s per direction instead of NVLink at 450 GB/s, on a collective that runs 4 times per transformer block on the critical path. Expect step time to collapse 5–10× the moment the scheduler places it — and note that it will *run*, correctly, just catastrophically slowly, so nothing will alert. The fixes: drop TP to 8 and absorb the remaining split into PP or into optimizer sharding; or target hardware with a larger NVLink domain (GB200 NVL72 gives you 72). Also check whether the config really needs 16-way TP at all — if it was chosen to fit memory, ZeRO-1 across the data-parallel dimension usually buys the same headroom for a collective the run already pays.

- **A 405B model, `tp=8, pp=8`, on 80 GB H100s. Plain DDP across 8 replicas. Does it fit?**
  **Answer:** No. Model state is `405e9 × 16 B = 6,480 GB`, sharded by `tp × pp = 64` ways to **101.3 GB per GPU** — over capacity before a single activation byte. Data parallelism under plain DDP *replicates*, so `dp=8` divides nothing. The cheapest fix is ZeRO-1 (Megatron's distributed optimizer): the 12Ψ of master weights and Adam moments shard across the 8 DP ranks, giving `4Ψ/64 + 12Ψ/(64×8) = 25.3 + 9.5 = 34.8 GB`, which fits with room for ~36 GB of activations. Full ZeRO-3 would take it to 12.7 GB but adds two all-gathers and a reduce-scatter per communication group per step for headroom you do not need. **The general lesson: check whether the DP dimension is replicating or sharding before you conclude anything about fit.**

- **A job's per-GPU memory is near-identical and high on every GPU, and it OOMs at step 3 but not step 0. What is it and what do you recommend?**
  **Answer:** Near-identical high memory across all GPUs means replication — DDP or a TP/PP layout with no data-parallel sharding. OOM at step 3 rather than step 0 means the growing term is **activations**, not model state: state is allocated once (with Adam's moments appearing on the first `.step()`, which is why step-1 OOMs are optimizer state), while activation peaks track batch size and sequence length. Confirm by halving `micro_batch_size` — if the OOM moves, it is activations. The fix ladder, cheapest first: enable sequence parallelism if TP is already on (free, removes replicated layernorm/dropout activations), then selective activation recomputation (~4% extra FLOPs, removes the `5as²b/h` quadratic term), then full activation checkpointing (~30% extra FLOPs), then context parallelism if the sequence is long. Sharding the model state further will not help, because model state is not what is growing.

## Connections & what's next

You now have the vocabulary and the arithmetic the rest of module 08 assumes: name the strategy, name the collective, name the link, and compute the bytes. **08.2** takes the collectives this lesson introduced — all-reduce, all-gather, reduce-scatter, all-to-all — and goes one layer down into NCCL itself: how it constructs the ring, how it chooses between ring and tree, how it picks a transport, and where the module's anchor skill (silent-hang debugging) lives. Every "at 450 GB/s that takes 263 ms" estimate in this lesson rests on the bus-bandwidth model 08.2 derives from first principles.

**08.3** takes the same numbers and turns them into MFU: given the per-step collective volume you can now compute, how much of your purchased FLOPs does a real step actually deliver, and at what batch size does the collective term overtake the compute term. **08.4**'s checkpoint sizing is the model-state accounting from §1, applied to bytes-on-disk: the same `16Ψ` that decides whether a run fits decides how long it takes to save.

Backward: **02b** supplied the 450 GB/s NVLink and 50 GB/s rail-NIC figures every trade in this lesson turns on; **03** supplied the HBM capacities the fit calculations run against; **06** places the gang so the TP groups this lesson says must stay NVLink-local actually do.

## References & further reading

**Primary sources**

1. **ZeRO: Memory Optimizations Toward Training Trillion Parameter Models** — Rajbhandari et al., <https://arxiv.org/abs/1910.02054>. The canonical source for the `16Ψ` accounting and the stage-1/2/3 progression (`4Ψ+12Ψ/N` → `2Ψ+14Ψ/N` → `16Ψ/N`). **Skim** the abstract and the memory-consumption figure. *arxiv.org is unreachable from this environment's egress proxy; the `K = 12` optimizer-state figure used above was verified against DeepSpeed's own ZeRO tutorial in the upstream repo (`docs/_tutorials/zero.md`), which states that a 1.5B-parameter GPT-2's Adam states occupy 18 GB — exactly 12 bytes/parameter — and 2.25 GB after partitioning across 8 ranks.*

2. **DeepSpeed repository — `docs/_tutorials/zero.md` and `docs/_pages/training.md`** — <https://github.com/deepspeedai/DeepSpeed>. **Read the ZeRO tutorial in full.** The stage definitions quoted in §4 are DeepSpeed's own wording of what each stage partitions, and the tutorial carries the measured 18 GB → 2.25 GB example. Verified by cloning the repository at HEAD in this pass.

3. **Megatron-LM repository — `docs/user-guide/parallelism-guide.md` and `docs/user-guide/features/dist_optimizer.md`** — <https://github.com/NVIDIA/Megatron-LM>. **Read both in full.** Source of the bytes-per-parameter table (20 / 18 / 16 non-distributed vs `4+16/d` / `6+12/d` / `8+8/d` distributed), the `Total GPUs = TP × PP × CP × EP × DP` identity, the reference configurations for Llama-3 / GPT-3 / Mixtral / DeepSeek-V3, and the "always enable `--sequence-parallel` with TP" guidance. Verified by cloning at HEAD; NVIDIA's hosted docs site was unreachable from this environment.

4. **PyTorch — `torch.distributed.fsdp.fully_shard` documentation** (`docs/source/distributed.fsdp.fully_shard.md` in the pytorch repository) — <https://github.com/pytorch/pytorch>. **Read the "Communication Grouping and Scheduling" section in full.** Source of the FSDP2 user contract, the one-group-per-`fully_shard`-call rule, the explicit "no `bucket_cap_mb`, no automatic bucketing" statement, and the forward/backward stream timelines reproduced in §4. *docs.pytorch.org is blocked by this environment's egress proxy; verified against the in-repo Markdown at HEAD.*

5. **PyTorch — `torch/nn/parallel/distributed.py`** (same repository). Source of the DDP `bucket_cap_mb` default of **25 MiB**, quoted directly from the parameter documentation. Verified in-source.

6. **Megatron-LM — `megatron/core/pipeline_parallel/schedules.py`** (same repository). Source of the three schedule implementations (`forward_backward_no_pipelining`, `forward_backward_pipelining_without_interleaving` = 1F1B, `forward_backward_pipelining_with_interleaving`) and the interleaving identity `total_num_microbatches = num_microbatches × num_model_chunks`, which is where the `v` in the interleaved bubble formula comes from. Verified in-source.

7. **Reducing Activation Recomputation in Large Transformer Models** — Korthikanti et al., MLSys 2023, <https://arxiv.org/abs/2205.05198>. Source of the activation-memory formulas: `sbh(34 + 5as/h)` per layer with no parallelism, `sbhL(10 + 24/t + 5as/(ht))` with tensor parallelism, and `34sbh/t` with sequence parallelism plus selective recomputation. *arxiv.org unreachable from this environment; the three formulas were confirmed via search against the paper's own text and NVIDIA's Megatron-Bridge activation-recomputation documentation rather than by fetching the PDF.*

**Real-world engineering blogs and reports**

8. **The Llama 3 Herd of Models, §3.3** — <https://arxiv.org/abs/2407.21783>. The module's anchor. 16,384 H100s, 4D parallelism at TP=8 / CP=2 / PP=16 with DP filling the remainder, ~400 TFLOP/s per GPU at 8K sequence, 38–43% BF16 MFU with the documented 43%→41% dip going from DP=64 to DP=128. *arxiv.org and ar5iv are both blocked here; the parallelism degrees and MFU figures were confirmed via search snippets of the paper and of the ISCA'25 follow-up "Scaling Llama 3 Training with Efficient Parallelism Strategies," not by reading the PDF in this pass.*

9. **Hugging Face — "The Technology Behind BLOOM Training"** — <https://huggingface.co/blog/bloom-megatron-deepspeed>, with the day-by-day chronicle at <https://github.com/bigscience-workshop/bigscience/blob/master/train/tr11-176B-ml/chronicles.md>. **Skim the blog, read the chronicle's first week.** BLOOM-176B's Megatron-DeepSpeed 3D-parallelism stack on 384 A100s — the cleanest public worked example of composing TP + PP + ZeRO, including why each degree was chosen. *huggingface.co is blocked by this environment's egress proxy; the chronicle is reachable on GitHub.*

10. **Meta — OPT-175B chronicles** — <https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles>. **Skim the README and logbook.** 992 A100s, FSDP combined with Megatron TP, at least 35 manual restarts and 100+ hosts cycled over two months — the failure surface a given parallelism layout produces, in primary-source form. Bridges into 08.5.

11. **Imbue — "From bare metal to a 70B model: infrastructure set-up and scripts"** — <https://imbue.com/research/70b-infrastructure/>. **Skim.** A 4,088-H100 / 511-node build-out with fully non-blocking 3-tier InfiniBand and 12,000+ network connections to keep healthy — what "the parallelism layout is a placement constraint" looks like at cluster-build scale.

12. **torchtitan — `docs/fsdp.md`** — <https://github.com/pytorch/torchtitan>. **Read in full; it is short.** The FSDP1→FSDP2 migration guide, the API difference table, and the measured claim that FSDP2 matches FSDP1's loss curve with 7% lower peak memory and higher MFU on 8×H100 Llama-7B. Verified by cloning at HEAD.

**Deeper dives**

13. **stas00/ml-engineering** — <https://github.com/stas00/ml-engineering>. An open book on parallelism, NCCL, checkpointing and debugging written by a BLOOM engineer. The performance and network chapters are the best practitioner companion to 08.1–08.3.

14. **"Efficient Parallelization Layouts for Large-Scale Distributed Model Training"** — <https://arxiv.org/abs/2311.05610>. A systematic study of TP/PP/DP layout choices beyond rule-of-thumb heuristics — useful once you are past reading configs and into choosing between them. *Not fetched in this pass (arxiv.org blocked); listed as optional depth, not relied on for any claim above.*
