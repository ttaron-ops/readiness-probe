---
lesson: "08.1"
title: "Parallelism strategies: network and memory footprint"
module: "08"
concept: "Parallelism strategies: network and memory footprint"
status: not-started
est_time: "6h"
artifacts: []
---

# 08.1 · Parallelism strategies: network and memory footprint

> **Concept.** Every parallelism strategy is a trade between *which collective runs*, *over which link*, and *how much GPU memory it saves* — read a run and you can predict its network and memory footprint without reading the model.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Why this matters

When a training job pages you, the ticket says "OOM at step 3" or "step time doubled after we added nodes" — never "our tensor-parallel degree is wrong." You have to translate. If you can look at a launch config and say *"that's FSDP, so every layer does an all-gather over the fabric, and your NIC is the bottleneck"* or *"that's tensor parallel, it must stay inside one NVLink domain and it's spanning two,"* you diagnose in minutes what an ML engineer debugs in days. This is also the load-bearing skill for capacity and cost: the parallelism strategy dictates whether a run is bound by HBM (module 03), by NVLink, or by the InfiniBand fabric — which is the difference between a node you can pack densely and one you can't.

## What's new here

You already have the physical layer. **02b** gave you the NVLink/NVSwitch domain and rail-aligned fabric — the links. **03** gave you HBM capacity and the roofline — why a model does or doesn't fit and whether a kernel is compute- or bandwidth-bound. **06** gave you gang scheduling and topology-aware placement — how the pod lands on the right GPUs.

This lesson adds the layer *between* the model and those links: **the parallelism strategy is what decides which collective traverses which link, and how the model's memory is split across GPUs.** 06 places the gang on 8 rail-aligned GPUs; this lesson tells you *why* it had to be 8 and not 16, and what those 8 GPUs are shouting at each other every step. From the platform view you are not choosing the strategy (that's a modeling decision) — you are *reading* it off a config and predicting its footprint so you can explain a failure or size a cluster.

## Core notes

### The memory accounting every strategy starts from

Take a model with **Ψ** parameters trained in mixed precision with Adam. Per GPU, an un-sharded replica holds:

| Tensor | Precision | Bytes/param |
|---|---|---|
| fp16 parameters | 2 B | 2Ψ |
| fp16 gradients | 2 B | 2Ψ |
| fp32 master params (optimizer) | 4 B | 4Ψ |
| fp32 Adam momentum | 4 B | 4Ψ |
| fp32 Adam variance | 4 B | 4Ψ |

That is **16 bytes per parameter** — the number to memorize. A 7B model is ~112 GB of *state* before a single activation. It does not fit on an 80 GB H100. Everything below is a scheme for cutting that 16Ψ down, or for splitting the compute, and each scheme buys memory with network traffic. The optimizer/gradient/master-weight block (the 12Ψ of fp32) is where the fat is, which is why the ZeRO stages attack it first.

There is a *fourth* consumer the 16Ψ table ignores: **activations** — the intermediate tensors saved during forward for use in backward. Activation memory scales with batch size, sequence length, and depth, and on long-context runs it can rival or exceed the state. It's why you'll see **activation checkpointing** (recompute activations in backward instead of storing them — trades compute for memory) in nearly every large config, and why a run can OOM *mid-step* on activations even though its state fit at init. As a platform engineer you don't tune this, but you recognize its fingerprint: an OOM that appears only at large batch/sequence, not at startup.

### A one-glance summary

| Strategy | Per-GPU state | Collective(s) per step | Primary link |
|---|---|---|---|
| DDP | 16Ψ (full replica) | 1 × all-reduce (grads) | fabric (IB/RoCE) |
| ZeRO-1 | 4Ψ + 12Ψ/N | all-reduce + gather of opt state | fabric |
| ZeRO-2 | 2Ψ + 14Ψ/N | reduce-scatter + all-gather | fabric |
| FSDP / ZeRO-3 | 16Ψ/N | ~2 all-gather + 1 reduce-scatter **per unit** | fabric / NVLink |
| TP | weights ÷ TP degree | ~4 all-reduce **per layer** | **NVLink only** |
| PP | layers ÷ stages | point-to-point sends | inter-node, small |

Read a config against this table and you've predicted its footprint.

### DP / DDP — replicate the model, all-reduce the gradients

**Data parallel** puts a *full replica* of the model on every GPU; each processes a different micro-batch. Memory: the full **16Ψ on every GPU** — no saving, this is the baseline. Network: exactly **one all-reduce of the gradients per step** (2Ψ of data), issued at the end of backward. PyTorch DDP buckets gradients and overlaps the all-reduce with the backward pass so the wire time hides under compute.

- **Collective:** all-reduce, once per step.
- **Volume:** a ring all-reduce moves ~`2(N-1)/N · 2Ψ` bytes per GPU — roughly constant in N, which is why DDP scales well on bandwidth.
- **Link:** the gradient all-reduce spans *all* data-parallel replicas, so it rides the **inter-node fabric** (InfiniBand/RoCE) whenever replicas are on different nodes.
- **Fails when:** the model + optimizer state doesn't fit in one GPU's HBM. DDP does nothing for memory.

### FSDP / ZeRO — shard the state, gather it just-in-time

**Fully Sharded Data Parallel** (PyTorch's implementation of ZeRO-3) keeps DDP's data-parallel structure but **shards parameters, gradients, and optimizer state across the N GPUs**, so each holds 1/N of the 16Ψ. ZeRO defines three stages of increasing aggressiveness:

- **Stage 1** — shard optimizer state only: `4Ψ + 12Ψ/N` per GPU.
- **Stage 2** — also shard gradients: `2Ψ + 14Ψ/N`.
- **Stage 3 / FSDP** — also shard parameters: **`16Ψ/N`**, approaching zero per-GPU state as N grows.

The catch is that a GPU can't compute a layer it only owns 1/N of. So FSDP does this per layer (per "FSDP unit"), every step:

1. **all-gather** the full parameters for the unit right before its forward (materialize, compute, then free).
2. **all-gather** them *again* in the backward pass.
3. **reduce-scatter** the gradients so each GPU keeps only its shard.

So where DDP issues **one collective per step**, FSDP issues roughly **two all-gathers + one reduce-scatter per unit per step** — dozens to hundreds of collectives, tightly interleaved with compute. **This is the trade: FSDP spends network bandwidth to buy HBM.** On a fat NVLink/NVSwitch domain the all-gathers are cheap and the trade is a clear win; across a thin fabric they can dominate step time. FSDP overlaps the *next* unit's all-gather with the *current* unit's compute (prefetch) to hide it — when that overlap fails, you see the GPUs stall waiting on the gather.

### TP — shard the matmul, all-reduce inside the layer

**Tensor parallel** (Megatron-style) splits the *individual matrix multiplies* of a layer across GPUs — each GPU holds a slice of the weight matrix and computes a partial result. To reconstruct the correct output, the partials must be combined with an **all-reduce inside every layer**: in a transformer block that's ~2 all-reduces in the forward pass (after attention, after the MLP) and 2 more in backward. This is **enormous, latency-sensitive, on the critical path** — it happens many times per step and the layer *cannot proceed* until it completes.

Because of that, **TP must live inside a single NVLink/NVSwitch domain** (see 02b). Put a TP group across the InfiniBand fabric and every layer stalls on a fabric-latency all-reduce; step time collapses. This is the concrete reason TP degree is almost always ≤ 8 on an HGX box: it's the NVLink domain size. When you see a config with `tensor_parallel_size: 8`, that 8 *is* the NVLink domain, and 06's topology-aware placement exists precisely to keep that group rail-aligned on one node.

### PP — split the layers, pay the bubble

**Pipeline parallel** slices the model *by layer depth* across stages: GPU group 0 holds layers 1–8, group 1 holds 9–16, and so on. Communication is cheap — just **point-to-point sends** of activations between adjacent stages, not collectives. The cost is the **pipeline bubble**: while stage 0 processes the first micro-batch, stages 1..p-1 sit idle waiting for work to reach them, and the pipeline drains symmetrically at the end.

The bubble fraction is:

```
bubble ≈ (p - 1) / (m + p - 1)
```

where **p** = number of pipeline stages, **m** = number of micro-batches per step. More micro-batches amortize the fill/drain, shrinking the bubble; more stages enlarge it. This is why PP runs push `m` high, and why schedules like 1F1B and interleaved-1F1B exist — they reduce the bubble and the activation memory it costs. From the platform side: a PP run that "wastes" 20% of GPU-time may simply have too few micro-batches, and that is visible as low, *evenly distributed* utilization across stages, not a hang.

### Putting it together — how a real run stacks them

Big runs compose all four as **3D (or 4D) parallelism**: TP inside the node (NVLink), PP across a few nodes (point-to-point), FSDP/DP across the rest of the fleet (fabric all-reduce/all-gather). Reading a real launch config, the degrees multiply to the world size:

```yaml
# a launcher's parallelism block — read it top-down
tensor_parallel_size: 8      # -> 1 NVLink domain, all-reduce every layer
pipeline_parallel_size: 4    # -> 4 stages, point-to-point, watch the bubble
data_parallel_size: 16       # -> 16 replicas, 1 all-reduce/step over the fabric
# world size = 8 * 4 * 16 = 512 GPUs
```

The mental model to carry into an incident:

- **TP traffic → NVLink, per layer, all-reduce, latency-bound.** Spanning a node boundary = catastrophic.
- **PP traffic → point-to-point between stages, small; cost is idle bubble, not bandwidth.**
- **DP/FSDP traffic → the fabric, per step; all-reduce (DDP) or all-gather+reduce-scatter (FSDP).**

Name the strategy and you've named the link, the collective, and the failure mode.

### Identifying the strategy on a job you inherited

You'll often be handed a running job with no config in front of you. Fast tells:

- **`nvidia-smi topo -m`** on the node plus the NCCL log's transports: heavy per-*layer* NVLink traffic (many small all-reduces, GPUs in lockstep) ⇒ **TP**. Per-*step* fabric all-reduce only ⇒ **DDP**. Repeated all-gather/reduce-scatter per layer over the fabric ⇒ **FSDP**.
- **Utilization shape:** evenly *low* util across a contiguous group of GPUs, with no hang ⇒ likely a **PP bubble** (too few micro-batches). Sawtooth util synced across all GPUs ⇒ DDP/FSDP step boundaries.
- **Memory:** near-identical high HBM use on every GPU ⇒ replication (**DDP**). Much lower, evenly-sharded HBM ⇒ **FSDP/ZeRO-3**.

You're not reverse-engineering the model — you're reading the *footprint* the strategy leaves, which is exactly what you need to explain a failure or size capacity.

## Worked example

**Config lands on your desk:** a 70B-parameter run, `tp=8, pp=4, dp=16`, mixed-precision Adam, on 512 H100s. Someone asks: "will it fit, and where does the network pressure go?"

**Memory.** 70B params × 16 B = **1,120 GB** of state for a full replica — obviously won't fit an 80 GB GPU. TP=8 splits each layer's weights 8 ways; PP=4 splits the layers 4 ways; so each GPU holds ~`1120 / (8×4)` = **35 GB** of state, plus activations. That fits with headroom — the run is viable, and you can say so without touching the model code.

**Network, traced by strategy:**
- Inside each group of 8 GPUs on one HGX node: **TP all-reduces every layer over NVLink.** If placement (06) ever splits a TP group across two nodes, step time will jump 5–10× and the symptom will be "training slow after a reschedule."
- Across the 4 pipeline stages: **point-to-point activation sends.** Cheap. If `m` (micro-batches) is small, expect a visible bubble — evenly low util across all 512 GPUs, *not* a hang.
- Across the 16 data-parallel replicas: **one gradient all-reduce per step over InfiniBand** (this run uses DP, not FSDP, because 35 GB fits — so no per-layer all-gathers). That all-reduce is the fabric's main load; a slow/flapping NIC on any replica shows up here as a step-time spike.

**The predictive payoff:** without reading a line of model code, you've said it fits (35 GB/GPU), the NVLink domain is load-bearing (TP=8), and the fabric carries one all-reduce per step (DP=16). That's the diagnosis an ML engineer would take an afternoon to reconstruct.

## Practice

**Environment:** one node, **2 rented GPUs** (e.g. 2× A100/H100, or 2× L4/L40 for a cheap run), single-node so NVLink or PCIe is the only link. PyTorch ≥ 2.4 (FSDP2 available), `torchrun`.

Run the **same tiny model twice** — once under DDP, once under FSDP — and measure the difference.

1. Build a deliberately over-fat model so state dominates activations: a stack of large `nn.Linear` layers (e.g. 12 × 8192×8192 ≈ 800M params) with Adam. Wrap it once with `DistributedDataParallel`, once with `fully_shard` (FSDP2) / `FullyShardedDataParallel`.
2. Launch each with `torchrun --nproc_per_node=2 train.py`.
3. During the steady state of each, capture **peak memory per GPU**: `nvidia-smi --query-gpu=memory.used --format=csv -l 1`, or in-code `torch.cuda.max_memory_allocated()`. Record the peak for DDP vs FSDP.
4. Set `NCCL_DEBUG=INFO` (foreshadowing 08.2) and, per step, count the collectives each issues — DDP logs a single grouped all-reduce; FSDP logs repeated all-gather / reduce-scatter calls per layer. You do not need to parse every line; note the *pattern and count difference*.

**Expected result:** FSDP's peak memory is markedly lower than DDP's (state is sharded across the 2 GPUs), and FSDP issues many more collectives per step. That is the memory-for-network trade made visible on two GPUs.

**Acceptance (feeds "Survive-a-failure"):** a short note (10–15 lines) with a **DDP-vs-FSDP peak-memory comparison** — the two `max_memory_allocated` numbers (or `nvidia-smi` peaks), the model's parameter count, and one sentence stating the extra collectives FSDP issued per step and the link they rode. This is the memory-footprint baseline the deliverable builds its checkpoint-sizing on.

## Self-check

**(a) Why does FSDP trade network for memory versus DDP?**
**Answer:** DDP keeps a full replica (16Ψ) on every GPU and pays only one gradient all-reduce per step — cheap network, no memory saving. FSDP shards params, grads, and optimizer state to 1/N each (down to ~16Ψ/N), but a GPU can't compute a layer it only holds a shard of, so it must **all-gather the full parameters just-in-time before each unit's forward and backward, and reduce-scatter the gradients** — roughly two all-gathers + one reduce-scatter per unit per step instead of DDP's single collective. It spends bandwidth (many extra collectives) to buy HBM (sharded state).

**(b) Which parallelism strategy must stay inside one NVLink domain, and why (reference 02b)?**
**Answer:** **Tensor parallel.** TP shards each layer's matmuls and must **all-reduce the partial results inside every layer**, many times per step, on the critical path — the layer cannot proceed until it completes. That collective is bandwidth-heavy and latency-sensitive, so it needs the NVLink/NVSwitch domain from 02b. Span it across the InfiniBand fabric and every layer stalls on fabric latency, collapsing step time — which is why TP degree is usually ≤ the NVLink domain size (8 on an HGX node) and 06's topology-aware placement keeps the TP group rail-aligned.

**(c) What is the pipeline bubble and what determines its size?**
**Answer:** The bubble is the idle GPU-time in pipeline parallelism while the pipeline **fills** (downstream stages wait for the first micro-batch to reach them) and **drains** (upstream stages finish early). Its fraction is ≈ `(p − 1) / (m + p − 1)`, where **p** = number of pipeline stages and **m** = micro-batches per step. More micro-batches amortize the fill/drain and shrink it; more stages enlarge it. It shows up as low, *evenly distributed* utilization across stages — not a hang.

## Resources

1. **ZeRO paper — Rajbhandari et al.** — <https://arxiv.org/abs/1910.02054>. **Skim** the abstract and the stage-1/2/3 memory-consumption figure (the `4Ψ+12Ψ/N` → `16Ψ/N` progression). Why: it's the canonical source for the 16-bytes-per-param accounting and exactly what FSDP implements. Ignore the throughput/systems sections for now.
2. **PyTorch FSDP docs** — <https://docs.pytorch.org/docs/stable/fsdp.html>. **Skim** for the sharding API (`fully_shard` / `FullyShardedDataParallel`) and the wrapping/prefetch knobs. Why: this is the concrete config surface you'll be reading off real launch scripts to identify the strategy and predict its footprint.
