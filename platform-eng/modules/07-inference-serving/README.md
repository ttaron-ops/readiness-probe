---
id: "07"
title: "Inference serving"
notion: "https://app.notion.com/p/3b33abaeb823810aa06bf912b8380bb1"
phase: "Phase 3 · Months 8–12"
effort: "~67 hrs ≈ 5–6 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["03"]
unlocks: ["11"]
started: null
completed: null
---

# 🚀 07 — Inference serving

> **Goal.** Operate production inference and reason about its unit economics — **the
> workload type most companies are actually hiring for**, and a distinct senior job
> family (Senior/Staff SWE, Inference) at every neocloud and AI-native shop.

- **Notion page:** https://app.notion.com/p/3b33abaeb823810aa06bf912b8380bb1
- **Phase:** Phase 3 · requires 03 · unlocks 11 · **Est. effort:** ~67 hrs ≈ 5–6 weeks (~$30–50 rented GPU)
- **Deliverable:** [Cost-per-million-tokens characterization](practice/cost-per-token/) —
  the CPM-vs-batch curve, FP8-vs-FP16 saving, and cold-start, wired into `gpu-cost-operator`.

## Why this module, and to what bar

Inference is a well-defined senior job family with remarkably consistent JD language:

- **CoreWeave** — *Sr/Staff SWE, Inference*: "inference internals including **batching, caching, mixed precision (BF16/FP8), streaming**"; "contributions to **vLLM, Triton, TensorRT-LLM, Ray Serve**"; "meet strict **P99 SLAs at scale**."
- **NVIDIA** — *AI Inference Performance Engineer*: "**TensorRT-LLM, SGLang, vLLM**… quantization, scheduling, memory management, distributed inference."
- **Anthropic** — *Sr/Staff+ SWE, Inference*: "**accelerator-agnostic** inference… intelligent request routing to fleet-wide orchestration."
- **The canonical interview loop question is literally "Design an LLM Inference Platform"** — GPU scheduling, KV cache, continuous batching, streaming. Sub-questions: size a 70B at N QPS to a TTFT target · KV-cache → concurrency cap · why HPA-on-CPU is useless · cold-start mitigation · **cost per million tokens FP8 vs FP16**.

## Calibrated to your background — what we skip

You did 03 (prefill/decode, roofline, memory-bound decode, KV cache concept, FP8) and 05
(TTFT/TPOT/ITL/queue-depth), so we **reference, not re-teach** those. New here: KV cache as
a **memory-management + concurrency** problem, the **PagedAttention** mechanism, vLLM
production tuning, batching **as cost-per-token curves**, quantization ops, KEDA
autoscaling + cold-start, and multi-model/LoRA economics.

## Lessons

Anchored on **PagedAttention/vLLM** (L3); the **cost pivot** is L5; everything converges on the CPM deliverable.

| # | Lesson | Hrs | Cost decision |
|---|--------|-----|---------------|
| 01 | [Inference workload shape](lessons/01-inference-workload-shape.md) | 6 | manage the KV budget to an SLO at min CPM |
| 02 | [KV cache as a concurrency problem](lessons/02-kv-cache-concurrency.md) | 6 | KV capacity *is* the concurrency ceiling |
| 03 | [**PagedAttention & vLLM**](lessons/03-pagedattention-and-vllm.md) (anchor) | 8 | near-zero fragmentation → high `max-num-seqs` |
| 04 | [vLLM in production (config + preemption)](lessons/04-vllm-in-production.md) | 8 | how much of the GPU you monetize |
| 05 | [**Batching economics** (CPM curves)](lessons/05-batching-economics.md) (cost pivot) | 8 | the min-CPM operating point under the SLO |
| 06 | [Alternative servers + disaggregation](lessons/06-alternative-servers-disaggregation.md) | 7 | match engine to workload; independent prefill/decode scaling |
| 07 | [**Quantization ops** (FP8 lever)](lessons/07-quantization-ops.md) | 6 | ~½ CPM on Hopper+ — measured, not asserted |
| 08 | [Autoscaling inference](lessons/08-autoscaling-inference.md) | 6 | scale-to-zero vs cold-start |
| 09 | [Model loading & storage](lessons/09-model-loading-storage.md) | 6 | what *makes* scale-to-zero viable |
| 10 | [Multi-model / LoRA](lessons/10-multi-model-lora.md) | 6 | one base + N adapters vs N deployments |

Total ≈ **67 hrs ≈ 5–6 weeks**. Spine = L3 + L5 + the CPM deliverable.

## Resource spine

- **vLLM docs** (conserving-memory, optimization/tuning, `bench serve`) — **pin a version**
  (V1 engine, ~v0.11.x); the V0→V1 rewrite invalidated older tutorials.
- **"Inside vLLM: Anatomy…"** blog + **PagedAttention paper** — the anchor deep-reads.
- **Introl/GMI** cost-per-token methodology — the `effective_cpm` formula.
- **NVIDIA Dynamo** for disaggregation + KV-aware routing; **BentoML** quantization handbook;
  **vLLM production-stack / Red Hat KServe+KEDA** for autoscaling.

> ⚠️ vLLM is the fastest-moving dependency here — pin the version in every command; **TGI
> is deprecated/archived** (don't pick it).

## Deliverable & checkpoint

- Build the **[Cost-per-million-tokens characterization](practice/cost-per-token/)** on one
  rented GPU: the CPM-vs-batch curve with the SLO knee, the FP8-vs-FP16 saving (with a
  *measured* accuracy delta), a cold-start measurement, and a config tuned to a TTFT-p99
  target — wired to emit the CPM into `gpu-cost-operator` (feeds Module 11).
- The [**checkpoint**](checkpoint.md) is the gate — size a 70B deployment cold, produce the
  curves, defend the operating point and the autoscaling signal, make the quant call.

## How to work this module

1. Reading-heavy slots (1, 2, 6-survey, 10) are GPU-light; concentrate GPU rental into
   weeks 2–5 (batch runs, kill promptly). Use an 8B model to learn mechanisms; reserve 70B/TP
   runs for the capstone measurements.
2. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion when
   the CPM report exists and emits into the operator.
