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
sources: 8
---
# 07.10 · Multi-model serving with LoRA

> **Concept.** One frozen base model plus many MB-sized LoRA adapters on a single GPU collapses N per-model deployments into one — an order-of-magnitude GPU saving for a multi-tenant platform.
>
> Module: [🚀 07 — Inference serving](../README.md) · Deliverable: [Cost-per-million-tokens](../practice/cost-per-token/README.md)

## Where this fits

Lesson 09 made the GPU wake up fast. This lesson makes each wake-up worth more —
instead of one GPU serving one model for one tenant, one loaded base serves dozens
to hundreds of tenants at once. It's the last lesson of the module and the final
multiplier on the cost-per-million-tokens deliverable you've been building since
Lesson 01: everything upstream (KV budgeting, PagedAttention, batching, quantization,
autoscaling, storage) made *one* model cheap to serve; this lesson makes *N* models
share that one cheap deployment.

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
platform engineer. It's not a niche technique either: an independent company
(Predibase) built its entire serving product around this exact thesis, and it now
shows up in AWS's own Trainium2 documentation — this is table-stakes serving
infrastructure, not one project's party trick.

## What's new here (calibration)

Everything so far served **one model per server**. This lesson breaks that 1:1:

- **A model is now base + adapter.** The expensive part (base weights, KV cache,
  attention layers) is shared; the per-tenant part (LoRA delta) is a rounding error
  in memory.
- **Routing moves into the request.** The `model` field of each request selects
  which adapter to apply — one endpoint, many logical models, per-request switching.
- **A new failure mode: adapters cost *throughput*, not just memory.** Memory stays
  flat as you add adapters; compute does not. Knowing where multi-LoRA *breaks* is
  the senior-level part.

## Core concepts

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

### The research lineage, in order

This didn't arrive fully formed as a vLLM flag — it's a clean two-year progression
from paper to production CLI, worth citing in that order because each step solved
the previous one's limitation:

1. **Punica (Oct 2023)** — introduced the SGMV kernel that lets one batched forward
   pass serve requests targeting *different* adapters, the foundational trick that
   makes shared-base serving fast at all.
2. **S-LoRA (Nov 2023)** — pushed further with **Unified Paging**: a PagedAttention-
   style unified memory pool that manages adapter weights *and* KV-cache tensors of
   varying rank/sequence-length together, plus heterogeneous batching across
   adapters. S-LoRA reports ~4x throughput over vLLM's then-naive LoRA support and
   claims it can serve "orders of magnitude" more adapters concurrently — this is
   the paper the module README is pointing at when it names "S-LoRA" for this lesson.
3. **vLLM `--enable-lora` (productionized)** — both ideas landed as one flag set in
   a serving engine you already run, turning a research technique into
   `--max-loras` / `--max-lora-rank` / `--max-cpu-loras`.

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
first performance cliff (below), and is the multi-LoRA analog of Lesson 09's
cold-start problem — just at adapter granularity instead of whole-model granularity.

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
   rank wastes memory, and S-LoRA's whole memory-management argument is about *not*
   wasting the pooled space this way).
3. **Cold-swapping adapters.** An adapter in the `max-cpu-loras` pool (or loaded at
   request time) must be paged CPU→GPU before its first use — an added
   PCIe-transfer latency on that request, the multi-LoRA echo of Lesson 09's
   cold-start problem, one adapter at a time. A workload that constantly cycles
   through many *different* cold adapters thrashes this swap and loses the throughput
   win. Multi-LoRA shines when a working set of hot adapters stays resident.
4. **"Stored" is not "concurrently active."** Predibase's headline "100+ fine-tunes
   on one GPU" and S-LoRA's "orders of magnitude more adapters" are both about the
   size of the *served set*, not how many are hot in a single batch at once — a
   working set stays resident while a long tail sits cold. Quoting the big number as
   if it means "100 simultaneous forward passes with zero overhead" overstates the
   claim.

**Rule of thumb:** multi-LoRA is a massive win for *many adapters, skewed traffic*
(a hot working set + a long cold tail). It degrades toward per-model serving when
traffic is *uniformly spread across a huge active set*, because you lose the shared-batch
efficiency and pay swap thrash.

### The boundary condition: this only works on one shared base

Everything above assumes all tenants fine-tuned the *same* base checkpoint. If 50
teams each fine-tuned 50 *different* base models (different sizes, different
pretraining runs, even different major versions of the same family), multi-LoRA
buys you nothing — there's no shared base weight to amortize, and you're back to N
full deployments. This is a common interview trap ("what if the fine-tunes aren't on
the same base?") and the honest answer is: multi-LoRA is a consolidation strategy
for organizational sprawl on top of *one* base, not a general fix for "many custom
models." Standardizing tenants onto a shared base is itself a platform decision this
lesson's economics should justify.

## Perspectives

**The independent-vendor validation view (Predibase / LoRAX).** Predibase built an
entire commercial product — LoRAX, "LoRA Exchange" — around exactly this lesson's
economic thesis, independent of vLLM's implementation, and reports packing 100+
fine-tuned models onto a single GPU with Continuous Multi-Adapter Batching. A
company betting its product on this pattern, using a different serving stack, is
strong outside evidence that "one base + N adapters" is now an industry category
rather than one project's feature.

**The research-lineage view.** Punica's SGMV kernel (Oct 2023) → S-LoRA's Unified
Paging and heterogeneous batching (Nov 2023) → vLLM's `--enable-lora` (productionized
both) is a clean, citable progression from research idea to a CLI flag in under two
years. Structuring your primary-source list in that chronological order (see Core
concepts above) is itself a small signal of depth in an interview — it shows you
know *why* the current flags exist, not just what they're called.

**The hardware-portability view (AWS Neuron).** AWS documents multi-LoRA serving for
Llama-3.1-8B on Trn2 (Trainium2) instances, including a path through vLLM with
NxD Inference. Multi-LoRA showing up in a non-NVIDIA accelerator's own docs is
evidence this is table-stakes serving infrastructure, not a CUDA-specific hack —
directly relevant to the "accelerator-agnostic inference" framing you'll see in
JDs like Anthropic's for this job family.

**The economic-model view.** This lesson's own worked example (~40 tenants, ~20x
GPU-cost collapse) isn't an outlier number — it gets corroborated at different
scales by two independent sources: Predibase's "100+ adapters from one GPU" (a
larger tenant count on similar hardware) and S-LoRA's "orders of magnitude more
adapters" (a research-system upper bound). Three different measurements converging
on the same shape of result is worth more than any one of them alone.

## Real-world use cases

- **vLLM Blog — "Efficiently serve dozens of fine-tuned models with vLLM on Amazon
  SageMaker AI and Amazon Bedrock"**
  ([blog.vllm.ai/2026/02/26/multi-lora.html](https://blog.vllm.ai/2026/02/26/multi-lora.html))
  — what it shows: the production write-up of shared-base multi-LoRA serving from
  the engine you're using, with a concrete framing — five customers each using 10%
  of a dedicated GPU consolidate onto one shared GPU — that's a direct restatement
  of this lesson's core economic argument, plus a companion AWS post covering the
  SageMaker/Bedrock deployment path.
- **Predibase — "LoRA Exchange (LoRAX): Serve 100s of Fine-Tuned LLMs for the Cost
  of One"**
  ([predibase.com/blog](https://predibase.com/blog/lora-exchange-lorax-serve-100s-of-fine-tuned-llms-for-the-cost-of-one))
  — what it shows: an independent (non-vLLM) open-source production system —
  Continuous Multi-Adapter Batching — validating the same economics at a larger
  claimed adapter count (100+), with a live GitHub repo (`predibase/lorax`) and
  third-party newsletter/press coverage.
- **AWS Neuron Documentation — "Tutorial: Multi-LoRA serving for Llama-3.1-8B on
  Trn2 instances"**
  ([awsdocs-neuron.readthedocs-hosted.com](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/libraries/nxd-inference/tutorials/trn2-llama3.1-8b-multi-lora-tutorial.html))
  — what it shows: multi-LoRA serving documented as a first-class pattern on AWS's
  own Trainium2 accelerators (including a path through vLLM), evidence the pattern
  generalizes beyond NVIDIA GPUs.

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

(As of this writing, $1.80–$2.50/GPU-hr are representative committed-cloud spot
rates for A100/H100-class cards — treat these as a dated snapshot to re-check
against current pricing, not a constant.)

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
5. **(Stretch)** Measure the throughput cost from Core concepts point 1: benchmark
   one adapter's throughput in isolation vs. the same adapter's throughput in a
   batch mixed with 2 other active adapters. Quantify the per-request latency creep.

**Acceptance:** a multi-LoRA serving demo committed under `practice/cost-per-token/`
that (1) shows ≥2 adapters answering on one server with per-request routing, and
(2) includes the VRAM-vs-adapter-count table proving the base is shared (flat VRAM).
State the resulting GPU-count collapse (N deployments → 1 base) and its effect on the
fixed-cost term of cost-per-million-tokens.

## Common pitfalls

1. **Assuming multi-LoRA "just works" at any adapter count or traffic mix.**
   Throughput cost scales with *active* adapter diversity per batch, not with
   adapters stored — S-LoRA's Unified Paging exists specifically because naive
   memory management chokes here.
2. **Treating `--max-lora-rank` as a free knob to set generously high.** It should
   match, not exceed, the largest rank you actually serve; over-provisioning wastes
   pooled GPU memory that could hold more adapters or more KV cache.
3. **Reading "100+ adapters on one GPU" as "100+ simultaneously active in every
   batch."** These marketing/paper numbers describe the served working set, not
   guaranteed zero-overhead concurrent execution — precision here matters in an
   interview setting.
4. **Missing the shared-base requirement.** Multi-LoRA buys nothing if tenants are
   on different base models. This is a common trap question — always check the
   premise before pitching the consolidation.
5. **Ignoring cold-swap thrash from a long, uniformly-hit adapter tail.** If
   traffic doesn't have a hot/cold skew, you pay PCIe swap latency on a large
   fraction of requests and lose much of the throughput advantage — profile your
   actual adapter access pattern before promising the multi-LoRA number.

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

**(d)** What has to be true about the tenants' models for multi-LoRA to help at all?
**Answer:** They all have to be fine-tunes of the *same* base checkpoint. The saving
comes entirely from sharing that one base's weights, attention layers, and KV-cache
pool across tenants; if 50 teams fine-tuned 50 different base models, there's nothing
to share and you're back to needing (up to) 50 separate deployments. This is the
standard interview trap-question boundary condition to name explicitly.

**(e)** What's the research lineage behind vLLM's `--enable-lora`, and why does the
order matter?
**Answer:** Punica (Oct 2023) introduced the SGMV kernel enabling one batched
forward pass across different active adapters. S-LoRA (Nov 2023) added Unified
Paging — extending PagedAttention-style pooled memory management to adapter weights
alongside KV cache — plus heterogeneous batching, reporting ~4x over naive LoRA
serving and orders-of-magnitude more servable adapters. vLLM's `--enable-lora` /
`--max-loras` / `--max-lora-rank` productionized both ideas into one flag set. The
order matters because each step solved a specific limitation of the one before it —
citing them in sequence shows you understand *why* the current design looks the way
it does, not just what the flags are called.

## Connections & what's next

This is the last lesson of Module 07, and it's worth naming explicitly how the whole
module converges here. L01–L02 gave you the KV-cache concurrency budget; L03–L04
gave you PagedAttention and a production-tuned vLLM; L05 turned batching into a
cost-per-token curve; L06 matched engine to workload; L07 halved cost again with
FP8; L08 made idle capacity free via scale-to-zero; L09 made the wake-up survivable;
and L10 (here) multiplies served tenants per loaded model without adding GPUs at
all. Every one of those levers is a term in the **cost-per-million-tokens
characterization** at `practice/cost-per-token/README.md` — that deliverable isn't a
separate assignment, it's the running total of everything this module taught. Passing
[`checkpoint.md`](../checkpoint.md) means being able to defend every term in that
total cold, on a whiteboard, including this lesson's fixed-cost collapse.

**What's next:** once the CPM characterization is wired into `gpu-cost-operator`,
**Module 11** picks up the economics thread at the fleet level: where this module
computed cost-per-million-tokens for *one* deployment, Module 11 uses that number
(and the multi-LoRA / scale-to-zero / quantization levers behind it) as an input to
fleet-wide GPU cost allocation, capacity planning, and utilization economics across
many deployments and tenants at once. The habit this lesson builds — never quote a
cost number without stating what's shared and what's per-tenant — is exactly the
discipline Module 11 assumes you already have.

## References & further reading

- **Primary sources**
  - Chen et al. — **"Punica: Multi-Tenant LoRA Serving"**, arXiv:2310.18547 (Oct
    2023) — the SGMV kernel that makes shared-base batched serving fast; ~12x
    throughput vs. separate LoRA servers at ~2ms/token added latency:
    https://arxiv.org/abs/2310.18547
  - Sheng et al. — **"S-LoRA: Serving Thousands of Concurrent LoRA Adapters"**,
    arXiv:2311.03285, MLSys 2024 (Nov 2023) — Unified Paging and heterogeneous
    batching; the paper named in this module's README for this lesson:
    https://arxiv.org/abs/2311.03285
  - vLLM LoRA docs (v0.26.0) — exact flags (`--enable-lora`, `--max-loras`,
    `--max-lora-rank`, `--max-cpu-loras`) and the runtime load/unload API:
    https://docs.vllm.ai/en/latest/features/lora/

- **Real-world engineering blogs**
  - vLLM Blog — "Efficiently serve dozens of fine-tuned models with vLLM on Amazon
    SageMaker AI and Amazon Bedrock" (Feb 2026) — production multi-LoRA write-up
    from the engine you're using: https://blog.vllm.ai/2026/02/26/multi-lora.html
  - Predibase — "LoRA Exchange (LoRAX): Serve 100s of Fine-Tuned LLMs for the Cost
    of One" — an independent open-source system validating the same economics:
    https://predibase.com/blog/lora-exchange-lorax-serve-100s-of-fine-tuned-llms-for-the-cost-of-one
  - AWS Neuron Documentation — "Tutorial: Multi-LoRA serving for Llama-3.1-8B on
    Trn2 instances" — the pattern on non-NVIDIA accelerators:
    https://awsdocs-neuron.readthedocs-hosted.com/en/latest/libraries/nxd-inference/tutorials/trn2-llama3.1-8b-multi-lora-tutorial.html

- **Deeper dives**
  - Anton Gordon — "Multi-LoRA in Production: Designing for vLLM and EKS" —
    Kubernetes deployment patterns, dynamic adapter loading, and routing on EKS:
    https://antonrgordon.medium.com/multi-lora-in-production-designing-for-vllm-and-eks-e8bc6a8b4b92
  - LMSYS Org — "Recipe for Serving Thousands of Concurrent LoRA Adapters" — the
    more accessible narrative companion to the S-LoRA paper:
    https://www.lmsys.org/blog/2023-11-15-slora/
