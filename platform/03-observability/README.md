---
id: "A-03"
track: "A — Platform excellence"
title: "Observability engineering"
notion: null                # repo-native module (added in the 12–15mo rebuild), not from the original Notion plan
phase: "Track A · deepen module"
effort: "6–8 weeks ≈ ~34 hrs @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: []
unlocks: []
started: null
completed: null
---

# 🔭 Observability engineering

> **Goal.** Deepen observability from your 40+ cluster operational base to **staff-level
> telemetry engineering**, and extend it to the GPU/ML signals that feed the cost work.
>
> **Track A — Platform excellence.** A deepen module: you cleared the senior bar here already;
> this takes it to staff depth and interview-ready fluency.

## Through-line

**Cardinality is the master constraint, and delivered work (goodput) is the master SLI.** Every
lesson is a variation on *"which signal, at what cost, answers this question — and does the metric
measure useful work or a lie?"* The module walks the signal-cost gradient (metrics → traces →
logs → profiles), teaches the correctness traps at each layer, and repeatedly reframes GPU
observability from *utilisation* to *goodput*.

## Calibrated to your background — what we skip

You run this stack daily, so we **skip the fundamentals** (what a metric/trace/log is, push vs
pull, RED/USE names, basic PromQL) and spend only on the **staff delta**: the traps that ship
wrong dashboards, how each system *falls over at fleet scale*, and the GPU/ML signals. Lesson 09
**references** your module-05 GPU-observability artifact (DCGM, the util-lie, MFU/goodput) and
scales it to thousands of GPUs rather than re-teaching it.

## Lessons

| # | Lesson | Staff delta |
|---|--------|-------------|
| 01 | [The signal model](lessons/01-signal-model.md) | cardinality as the master constraint; the signal-fit/cost matrix |
| 02 | [Prometheus and PromQL](lessons/02-prometheus-and-promql.md) | the 5 PromQL traps that ship wrong dashboards |
| 03 | [Metrics at scale](lessons/03-metrics-at-scale.md) | how a metrics system actually falls over; Thanos vs Mimir |
| 04 | [OpenTelemetry](lessons/04-opentelemetry.md) | the Collector as the integration point; two-tier + tail-sampling |
| 05 | [Distributed tracing](lessons/05-distributed-tracing.md) | head vs tail sampling; making tracing pay off via exemplars |
| 06 | [Logging pipelines](lessons/06-logging-pipelines.md) | Loki vs ELK; log cardinality as the sleeper failure |
| 07 | [SLOs and alerting](lessons/07-slos-and-alerting.md) | multi-window multi-burn-rate, from first principles |
| 08 | [Continuous profiling and eBPF](lessons/08-profiling-and-ebpf.md) | on-CPU vs off-CPU; fleet-wide eBPF profiling |
| 09 | [GPU and ML observability at fleet scale](lessons/09-gpu-and-ml-observability.md) | the synthesis: goodput alerts, NCCL, stragglers |

Total ≈ **34 hrs ≈ 6–8 weeks**. **Spine:** L1 (cardinality), L2 (PromQL traps), L7 (burn-rate),
L9 (fleet synthesis).

## Deliverable & checkpoint

- Build the **[fleet observability design](practice/fleet-observability/)** — a three-part
  portfolio artifact: (1) a 4,000-GPU **fleet observability design doc** (scrape-sharding + Mimir
  sizing, per-tenant cardinality budget with relabel configs, DCGM/NCCL signal plan, two-tier OTel
  collector); (2) a **burn-rate alert set** (a service SLO *and* a GPU goodput-regression SLO with
  a wasted-GPU-hours budget) as deployable rule YAML; (3) a **"PromQL traps that lie"** writeup.
- The [**checkpoint**](checkpoint.md) is the gate — diagnose and rewrite three broken PromQL panels
  *explaining why each is wrong*, derive an MWMBR alert set with the multiplier math, and defend a
  signal-choice tradeoff ("why is this a span attribute, not a label").

## Directory layout

| Path | What goes here |
|------|----------------|
| [`lessons/`](lessons/) | One page per concept — notes, worked example, practice, self-check. |
| [`practice/`](practice/) | Design write-ups, labs, diagrams — the buildable output. |
| [`resources/`](resources/) | Saved references, papers, link index. |
| [`checkpoint.md`](checkpoint.md) | Checkpoint answers (the completion gate). |
