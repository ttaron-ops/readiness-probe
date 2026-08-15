---
id: "03"
title: "GPU hardware fundamentals"
notion: "https://app.notion.com/p/3b33abaeb82381abaccfd55ab24bc852"
phase: "Phase 2 · Months 5–8"
effort: "~51 hrs ≈ 5 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["02b"]
unlocks: ["04", "07", "11"]
started: null
completed: null
---

# 🔌 03 — GPU hardware fundamentals

> **Goal.** Enough GPU hardware literacy to reason about **utilisation vs useful
> work, throughput, and cost per unit of work** — you're not becoming a CUDA kernel
> developer; you need to know what the numbers mean.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381abaccfd55ab24bc852
- **Phase:** Phase 2 · requires 02b · **Est. effort:** ~51 hrs ≈ 5 weeks (~$25–35 rented GPU)
- **Deliverable:** [GPU Efficiency & Cost Report](practice/gpu-efficiency-report/) —
  achieved-vs-spec TFLOPS, FP16→FP8 delta, the util-lie demo, and a cost-per-useful-work SKU call.

## Why this module, and to what bar

For a *platform* engineer the bar isn't "write CUDA" — it's "reason about GPU
efficiency and cost from the telemetry":

- **CoreWeave** — *GPU Performance Engineer*: "performance tests + automation for **hardware validation across the fleet**… controllers to automate infra testing… visibility into metrics/performance/health."
- **NVIDIA** — *Solutions Architect*: "inference optimization via **FP16/INT8/FP8**… arithmetic intensity vs peak compute and memory bandwidth using the **roofline model** to classify bottlenecks."
- **Interview probes:** *"100% util but terrible throughput — why?"* · *"why is decode memory-bandwidth bound, prefill compute-bound?"* · *"FP8 vs FP16 cost lever"* · *"tokens/sec ceiling for a 70B on H100"* (≈24) · *"which SKU is cheaper per unit of useful work?"*

## Calibrated to your background — what we skip

Not a kernel dev, and 02b already did the topology/power layer. We **skip**: CUDA
kernel authoring, PTX/SASS, deep microarchitecture, occupancy-tuning-as-coding, and
**NVLink/PCIe topology, node layout, and power/thermal throttling (all 02b)** —
referenced, not re-taught. The execution model appears *only* as the vocabulary for
reading utilisation honestly.

## Lessons

Anchored on **the roofline** (L2); ends in the cost capstone (L7).

| # | Lesson | Hrs | Decision it drives |
|---|--------|-----|--------------------|
| 01 | [Execution model & **the utilisation lie**](lessons/01-execution-model-and-utilisation.md) | 7 | distrust `GPU-Util`; demand tensor-active/MFU |
| 02 | [Compute- vs memory-bound + **roofline**](lessons/02-compute-vs-memory-bound-roofline.md) (anchor) | 8 | will a bigger/newer SKU even help? |
| 03 | [Memory hierarchy & **HBM bottleneck**](lessons/03-memory-hierarchy-hbm.md) | 6 | what fits (weights+KV) + decode throughput |
| 04 | [**Decode bandwidth-bound** → batching](lessons/04-decode-bandwidth-batching.md) | 7 | batch sizing, prefill/decode disaggregation |
| 05 | [Precision & tensor cores (**the cost lever**)](lessons/05-precision-and-tensor-cores.md) | 7 | FP8 → ½ memory, ~2× throughput, $/token |
| 06 | [Generational SKUs + **software-stack hazard**](lessons/06-generational-and-software-stack.md) | 6 | purchasing; driver↔CUDA↔NCCL breakage |
| 07 | [**Capstone — cost per unit of useful work**](lessons/07-capstone-cost-per-useful-work.md) | 10 | the SKU recommendation, defended by numbers |

Total ≈ **51 hrs ≈ 5 weeks** · ~7–8 hrs rented-H100 time. Spine = L2 + L7.

## Resource spine

- **Horace He — "Making Deep Learning Go Brrrr From First Principles"** — the
  conceptual anchor (deep-read twice).
- **NVIDIA Hopper/H100 whitepaper** — the numbers + FP8/Transformer Engine (skim).
- **Programming Massively Parallel Processors** ch.1–5 — the model only (skim, no exercises).
- **Modal util guide** + **"Measuring GPU utilization one level deeper"** — the util-lie demo.
- **SemiAnalysis GPU index** — live $/hr (refresh at study time); **CUDA Compatibility docs** — the version hazard.

## Deliverable & checkpoint

- Build the **[GPU Efficiency & Cost Report](practice/gpu-efficiency-report/)** on one
  rented H100 weekend.
- The [**checkpoint**](checkpoint.md) is the gate — interpret every `nvidia-smi` field,
  place workloads on a roofline, estimate decode ceilings, and argue cheapest-per-useful-work.

## How to work this module

1. Lessons in order; batch the hands-on (L1–L6) into one rented-GPU session, then write L7.
2. Keep the raw benchmark logs — they *are* the deliverable.
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion
   when the report is done.
