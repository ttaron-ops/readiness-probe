---
lesson: "07.10"
title: "Multi-model serving with LoRA"
module: "07"
concept: "Multi-model serving with LoRA"
status: not-started
est_time: "5h"
artifacts: []
---
# 07.10 · Multi-model serving with LoRA

> **Concept.** One frozen base model plus many MB-sized LoRA adapters on a single GPU collapses N per-model deployments into one — an order-of-magnitude GPU saving for a multi-tenant platform.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Why this matters

Picture the multi-tenant reality of a GPU-heavy platform: 50 teams each fine-tuned
"their own" Llama-3.1-8B. The naive architecture is 50 deployments, each holding a
full 16 GB of fp16 weights, each pinning at least one GPU. That's 50 GPUs — call it
$50–$100k/month — mostly idle, because no single tenant saturates a card.

Here's the structural insight: those 50 fine-tunes are **not 50 different models**.
They're one shared base model plus 50 tiny **deltas**. A LoRA adapter is *megabytes*;
the base is *gigabytes*. If the base is shared and only the deltas swap per request,
**thousands of tenants can ride one base model's VRAM on one GPU.** 50 GPUs collapse
toward 1–2. This is the single biggest platform-economics multiplier in the module,
and it's exactly the "we cut serving cost 20× by consolidating fine-tunes onto
multi-LoRA" story a GPU-heavy big-tech interviewer wants to hear from a senior
platform engineer.

## What's new here

Everything so far served **one model per server**. This lesson breaks that 1:1:

- **A model is now base + adapter.** The expensive part (base weights, KV cache,
  attention layers) is shared; the per-tenant part (LoRA delta) is a rounding error
  in memory.
- **Routing moves into the request.** The `model` field of each request selects
  which adapter to apply — one endpoint, many logical models, per-request switching.
- **A new failure mode: adapters cost *throughput*, not just memory.** Memory stays
  flat as you add adapters; compute does not. Knowing where multi-LoRA *breaks* is
  the senior-level part.

## Core notes

### Why an adapter is MBs and a full model is GBs

LoRA freezes the base weights `W` and learns a low-rank update `ΔW = B·A`, where for
a `d×d` weight matrix `A` is `r×d` and `B` is `d×r` with rank `r ≪ d` (typically
`r ∈ {8, 16, 32, 64}`). You store only `A` and `B`, applied to a subset of layers
(attention projections). The size ratio is enormous:

| | Llama-3.1-8B, fp16 | Llama-3.1-70B, fp16 |
|---|---|---|
| Full model | ~16 GB | ~140 GB |
| Full fine-tune (a copy) | ~16 GB | ~140 GB |
| **LoRA adapter, r=16** | **~40–60 MB** | **~150–250 MB** |
| **LoRA adapter, r=64** | ~150–250 MB | ~0.6–1 GB |

That's a **~300–400× memory difference** between a full fine-tune and an r=16 adapter
on 8B. A full fine-tune is a whole second model to host; an adapter is a file you can
`scp`. This ratio is the entire economic argument.

### How adapters share one base's VRAM (vLLM multi-LoRA)

At load, vLLM holds **one** copy of the base weights and its PagedAttention KV-cache
pool. Adapters are loaded as small `A/B` tensor pairs alongside it. At inference,
vLLM batches requests targeting *different* adapters in the **same** forward pass:
the base matmul runs once for the whole batch, and each request's adapter delta is
applied by a specialized kernel — vLLM uses the **Punica** SGMV kernel (Segmented
Gather Matrix-Vector Multiply), which gathers the right `A/B` per row of the batch
and adds the delta. Punica reports up to **~12× higher throughput** than running
separate LoRA servers, at **~2 ms/token** added latency, precisely because it keeps
the base shared instead of replicating it. Adapters share the base's attention layers
and KV cache; only the delta weights are per-tenant.

The consequence you'll verify in the practice: **VRAM stays ~flat as you add
adapters.** Going from 1 → 3 → 20 adapters on an 8B base moves VRAM by tens of MB
each, not by 16 GB each. The base and the KV-cache pool dominate the footprint and
are paid once.

### Serving and routing: vLLM `--enable-lora` (v0.26.0)

Start one server with the base model and register adapters:

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --enable-lora \
  --max-loras 4 \            # max adapters *active in a single batch* (GPU-resident)
  --max-cpu-loras 32 \       # adapters parked in CPU RAM, swapped in on demand (>= max-loras)
  --max-lora-rank 32 \       # must be >= the largest r you serve (8/16/32/64/128/256/512)
  --lora-modules sql=/adapters/sql-lora support=/adapters/support-lora legal=/adapters/legal-lora
```

Per-request routing is just the `model` field — same endpoint, different logical model:

```bash
curl localhost:8000/v1/completions -d '{"model":"sql","prompt":"SELECT ...","max_tokens":64}'
curl localhost:8000/v1/completions -d '{"model":"support","prompt":"Refund policy?","max_tokens":64}'
curl localhost:8000/v1/completions -d '{"model":"meta-llama/Llama-3.1-8B-Instruct","prompt":"..."}'  # base, no adapter
```

Dynamic (no restart) load/unload — the key to a self-serve tenant platform — needs
`VLLM_ALLOW_RUNTIME_LORA_UPDATING=1`, then:

```bash
curl localhost:8000/v1/load_lora_adapter   -d '{"lora_name":"newteam","lora_path":"/adapters/newteam"}'
curl localhost:8000/v1/unload_lora_adapter -d '{"lora_name":"legal"}'
```

Flag semantics that matter for cost: **`--max-loras`** caps how many *distinct*
adapters can be live in one batch (GPU-resident); **`--max-cpu-loras`** is the larger
pool held in host RAM and paged onto the GPU when a request needs them. Adapters
beyond `max-loras` in a batch aren't rejected — they're **swapped**, which is the
first performance cliff (below).

### Where multi-LoRA breaks down

The saving is real but not free. Senior-level answer = knowing the limits:

1. **Throughput cost per *active* adapter.** Batching requests across many distinct
   adapters means the SGMV kernel does more scattered gather/matmul work than a
   single-adapter (or base-only) batch. Latency creeps (~2 ms/token in Punica's
   numbers, more as active-adapter diversity rises). A batch hitting one hot adapter
   is far more efficient than the same batch spread across 30 cold ones.
2. **Adapter-count / rank limits.** `--max-loras` bounds *concurrent* adapters per
   batch; pushing it up costs VRAM and kernel overhead. **Rank drives cost**: an r=64
   adapter is ~4× the memory and delta-compute of r=16, and every adapter you serve
   must fit under `--max-lora-rank` (set it to the max you serve, not higher — extra
   rank wastes memory). Thousands *stored* is fine; the constraint is thousands
   *concurrently active in one batch*.
3. **Cold-swapping adapters.** An adapter in the `max-cpu-loras` pool (or loaded at
   request time) must be paged CPU→GPU before its first use — an added
   PCIe-transfer latency on that request, the multi-LoRA echo of Lesson 09's
   cold-start problem, one adapter at a time. A workload that constantly cycles
   through many *different* cold adapters thrashes this swap and loses the throughput
   win. Multi-LoRA shines when a working set of hot adapters stays resident.

**Rule of thumb:** multi-LoRA is a massive win for *many adapters, skewed traffic*
(a hot working set + a long cold tail). It degrades toward per-model serving when
traffic is *uniformly spread across a huge active set*, because you lose the shared-batch
efficiency and pay swap thrash.

## Worked example

**Scenario.** Platform serving **40 tenant fine-tunes** of Llama-3.1-8B. fp16 base
= 16 GB; each adapter r=16 ≈ 50 MB. GPU: 1× A100-40GB at $1.80/GPU-hr.

**Naive: 40 full deployments.**
- 40 × 16 GB = 640 GB of weights → at ~1 model/40 GB GPU (leaving KV headroom),
  ~40 GPUs. Even bin-packed 1 model/GPU, that's **40 GPUs**.
- Cost: 40 × $1.80 × 24 × 30 ≈ **$51,840/month**, most idle.

**Multi-LoRA: 1 base + 40 adapters.**
- VRAM: 16 GB base + 40 × 50 MB = 16 GB + 2 GB = **18 GB** — fits one A100-40GB with
  ~20 GB left for KV cache / batching.
- Handle real QPS with, say, **2 GPUs** for throughput + HA: 2 × $1.80 × 24 × 30 ≈
  **$2,592/month**.
- **~20× cheaper** ($51.8k → $2.6k), and adding tenant #41 is a 50 MB file, not a
  new GPU.

**Feeding the deliverable.** In cost-per-million-tokens terms: fixed GPU cost is now
amortized across *all 40 tenants' combined* token volume instead of each tenant
paying for a lonely, mostly-idle GPU. If the fleet does 200M tokens/day across
2 GPUs, GPU cost/M ≈ ($2,592/30/200M)×1e6 ≈ **$0.43/M tokens**, versus the naive
design where a low-traffic tenant's dedicated GPU could imply $50+/M. The adapter's
own storage/load cost is negligible (MB-scale, per Lesson 09). Multi-LoRA is where
the *fixed*-cost term of your unit economics collapses.

## Practice

**Goal:** run one vLLM `0.26.0` server with 2–3 LoRA adapters, route per request,
and prove VRAM stays ~flat as adapters are added (shared base).

1. **Get adapters.** Grab or train 2–3 small LoRA adapters for one base (e.g.
   Llama-3.1-8B-Instruct) — any r=8/16 adapters from the Hub work; they just need to
   match the base.
2. **Serve.**
   ```bash
   VLLM_ALLOW_RUNTIME_LORA_UPDATING=1 vllm serve meta-llama/Llama-3.1-8B-Instruct \
     --enable-lora --max-loras 3 --max-lora-rank 16 \
     --lora-modules a=/adapters/a b=/adapters/b
   ```
3. **Route per request.** Send requests with `"model":"a"`, `"model":"b"`, and the
   base name; confirm the outputs differ (adapter actually applied).
4. **Prove shared VRAM.** Record `nvidia-smi --query-gpu=memory.used --format=csv` at:
   base only → +adapter a → +adapter b → +adapter c (load the 3rd at runtime via
   `/v1/load_lora_adapter`). VRAM should rise by only tens of MB per adapter, not by
   base-model-sized jumps.

**Acceptance:** a multi-LoRA serving demo committed under `practice/cost-per-token/`
that (1) shows ≥2 adapters answering on one server with per-request routing, and
(2) includes the VRAM-vs-adapter-count table proving the base is shared (flat VRAM).
State the resulting GPU-count collapse (N deployments → 1 base) and its effect on the
fixed-cost term of cost-per-million-tokens.

## Self-check

**(a)** Memory for a LoRA adapter vs a full fine-tuned model — order of magnitude?
**Answer:** ~2–3 orders of magnitude apart. A full fine-tune of Llama-3.1-8B is a
whole ~16 GB copy; an r=16 LoRA adapter is only the low-rank `A/B` deltas on the
attention projections — ~40–60 MB, roughly **300–400× smaller**. That's the entire
reason thousands of adapters fit beside one base's VRAM while thousands of full
fine-tunes would need thousands of GPUs.

**(b)** When does multi-LoRA serving break down (throughput / adapter-count)?
**Answer:** Memory scales fine; **compute and swapping** don't. Batching across many
*distinct active* adapters makes the SGMV kernel do scattered per-row delta work,
adding latency (~2 ms/token and up), so a batch spread over 30 cold adapters is far
slower than one hitting a single hot adapter. `--max-loras` bounds concurrent
adapters per batch; exceeding it forces CPU↔GPU **cold-swaps** (PCIe transfer
latency on first use), and a workload cycling through many cold adapters thrashes
that swap. Higher rank multiplies both memory and delta-compute. Net: it degrades
toward per-model serving under *uniform traffic across a huge active set*; it wins
under *skewed traffic with a hot working set*.

**(c)** Why is one-base-plus-N-adapters an order-of-magnitude GPU saving for a
multi-tenant platform?
**Answer:** Because the expensive resource — base weights + KV-cache pool + attention
compute — is paid **once and shared**, while the per-tenant part (the adapter) is
MB-scale and effectively free in VRAM. N full deployments each pin ≥1 GPU on a
16–140 GB model that no single low-traffic tenant saturates; consolidating them puts
one base + N tiny deltas on one GPU, so 40 GPUs collapse to 1–2 (~20× in the worked
example). It also turns onboarding tenant N+1 from "provision a GPU" into "drop a
50 MB file," collapsing the *fixed*-cost term of cost-per-million-tokens.

## Resources

1. **vLLM Blog — "Efficiently serve dozens of fine-tuned models with vLLM"
   (multi-LoRA, 2026-02-26)** — the canonical production write-up on shared-base
   multi-LoRA serving and its economics:
   https://vllm.ai/blog/2026-02-26-multi-lora
2. **Gordon — "Multi-LoRA in Production: Designing for vLLM and EKS"** — Kubernetes
   deployment patterns, dynamic adapter loading, and routing on EKS:
   https://antonrgordon.medium.com/multi-lora-in-production-designing-for-vllm-and-eks-e8bc6a8b4b92
3. **vLLM LoRA docs (v0.26.0)** — exact flags (`--enable-lora`, `--max-loras`,
   `--max-lora-rank`, `--max-cpu-loras`) and the runtime load/unload API:
   https://docs.vllm.ai/en/latest/features/lora/
