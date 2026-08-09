---
id: "05"
title: "GPU observability and telemetry"
notion: "https://app.notion.com/p/3b33abaeb82381c390d7ce54b1b87b6e"
phase: "Phase 2 · Months 5–8"
effort: "~37 hrs ≈ 3.5 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["04"]
unlocks: ["11"]
started: null
completed: null
---

# 📊 05 — GPU observability and telemetry

> **Goal.** Measure what is **actually** happening on a GPU fleet. Your observability
> depth makes this the fastest module — and it's the **direct input to your cost work**
> and the home of the flagship "your GPU dashboard is lying to you" artifact.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381c390d7ce54b1b87b6e
- **Phase:** Phase 2 · requires 04 · unlocks 11 · **Est. effort:** ~37 hrs ≈ 3.5 weeks
- **Deliverable:** ["Your GPU dashboard is lying to you"](practice/gpu-dashboard-lie/) —
  allocated-vs-utilised dashboard + the util-lie exhibit + a PromQL pack (also a blog post).

## Why this module, and to what bar

GPU telemetry is now a **named senior competency**, not a monitoring sub-bullet:

- **NVIDIA** — *Senior AI & HPC Observability Engineer*, *Senior Platform Telemetry Engineer*, *SRE — Observability & Telemetry Platform*.
- **CoreWeave** — *Sr SWE, Observability*: "visibility into complex AI workloads… metrics/tracing at massive scale" (a neocloud whose *product is GPU fleet transparency*).
- **Datadog** shipped a first-class DCGM GPU-monitoring product in 2025 — its taxonomy *is* this syllabus (`sm_active`, `tensor_active`, `process.sm_active`, `errors.xid`).
- **Interview probes:** *"what does `DCGM_FI_DEV_GPU_UTIL` actually measure?"* · *"`SM_ACTIVE` vs `PIPE_TENSOR_ACTIVE` — which to alert on?"* · *"which XID cordons a node?"* · *"TTFT vs TPOT, and why p99 request latency is wrong for streaming"* · *"explain the util trap to a CFO."*

## Calibrated to your background — what we skip

You run observability at 40+ clusters and did 03/04, so we **reference, not re-teach**:
Prometheus/PromQL/Grafana mechanics, the pod-resources→UUID join (04 — you *consume* it
here), the hardware util-lie *concept* (03 — here it becomes a **metric name and an
alert**), and generic FinOps framing (you're certified). Hours go to **DCGM internals**.

## Lessons

Anchored on **the lie vs the truth**; spine = **allocated-vs-utilised GPU-hours** (introduced L1, paid off L8).

| # | Lesson | Hrs | The payoff |
|---|--------|-----|-----------|
| 01 | [**The lie and the truth**](lessons/01-lie-and-truth.md) (metrics, merged anchor) | 5 | `GPU_UTIL` semantics + the four `PROF_*` metrics + which to alert on |
| 02 | [DCGM architecture](lessons/02-dcgm-architecture.md) | 4 | hostengine, field groups, the profiling sampler + its cost |
| 03 | [dcgm-exporter at fleet scale](lessons/03-dcgm-exporter-at-scale.md) | 4 | `SM_ACTIVE` ships **commented out**; cardinality on 500+ GPUs |
| 04 | [Attribution (consumes 04)](lessons/04-attribution.md) | 3 | per-namespace truthful metrics; the time-slicing hole |
| 05 | [**Health & errors / XID**](lessons/05-health-and-xid.md) | 4 | which XID cordons (48/63/64/79/94/95) vs logs (13/31/43) |
| 06 | [Inference SLOs](lessons/06-inference-slos.md) | 5 | TTFT/TPOT/ITL/queue-depth; the batching trade |
| 07 | [Profiling escalation](lessons/07-profiling-escalation.md) | 4 | metrics → PyTorch Profiler → Nsight ladder |
| 08 | [**Capstone — allocated-vs-utilised dashboard + util-lie artifact**](lessons/08-capstone-allocated-vs-utilised.md) | 8 | the flagship |

Total ≈ **37 hrs ≈ 3.5 weeks** — your fluency banks time for the capstone/blog polish. Spine = L1 → L8.

## Resource spine

- **DCGM field-ID reference** + **profiling API** (the multiplexing / sampling-cost section).
- **dcgm-exporter's `default-counters.csv`** — audit it live; the trap ships in the box.
- **NVIDIA XID guide** + `dcgm_errors.h` + **NVSentinel / AKS NPD** for XID→cordon wiring.
- **Datadog's DCGM taxonomy** as the "how a paid product does it" reference.
- **BentoML / Spheron** for inference SLOs; community **Grafana DCGM dashboards** as the "here's the lie everyone ships" exhibit.

> ⚠️ Version-sensitive: use DCGM `/latest/` docs; the dcgm-exporter default CSV drifts across releases; pin your vLLM version (metric names changed 0.5→0.8).

## Deliverable & checkpoint

- Build **["Your GPU dashboard is lying to you"](practice/gpu-dashboard-lie/)** from your
  own cluster — the per-namespace allocated-vs-utilised (in dollars) dashboard, the
  `GPU_UTIL=100%` vs `SM_ACTIVE≈0.1` exhibit, and the PromQL pack. It doubles as the blog post.
- The [**checkpoint**](checkpoint.md) is the gate — state what `GPU_UTIL` measures without
  the word "utilisation," alert on the right metric, classify XIDs, and pass the CFO test.

## How to work this module

1. Front-load L1–L3 (they enable everything); L4–L5 next; L6–L7; capstone spills into week 3–4.
2. Batch the hands-on onto one rented GPU weekend (dcgm-exporter + Prometheus/Grafana + vLLM).
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update Notion when
   the dashboard renders the gap and the exhibit screenshot exists.
