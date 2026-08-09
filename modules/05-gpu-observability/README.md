---
id: "05"
title: "GPU observability and telemetry"
notion: "https://app.notion.com/p/3b33abaeb82381c390d7ce54b1b87b6e"
phase: "Phase 2 · Months 5–8"
effort: "3–4 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["04"]
unlocks: ["11"]
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# 📊 05 — GPU observability and telemetry

> **Goal.** Measure what is actually happening on a GPU fleet; the direct input to the cost work.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381c390d7ce54b1b87b6e
- **Phase:** Phase 2 · Months 5–8 · **Est. effort:** 3–4 weeks
- **Prerequisites:** `04` · **Unlocks:** `11`

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [DCGM architecture](lessons/01-dcgm-architecture.md) | `not-started` |
| 02 | [The metrics that mislead](lessons/02-metrics-that-mislead.md) | `not-started` |
| 03 | [The metrics that tell the truth](lessons/03-metrics-that-tell-truth.md) | `not-started` |
| 04 | [dcgm-exporter](lessons/04-dcgm-exporter.md) | `not-started` |
| 05 | [Health and errors](lessons/05-health-and-errors.md) | `not-started` |
| 06 | [Attribution](lessons/06-attribution.md) | `not-started` |
| 07 | [Inference SLOs](lessons/07-inference-slos.md) | `not-started` |
| 08 | [Profiling](lessons/08-profiling.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb82381c390d7ce54b1b87b6e). Save any
local copies (PDFs, diagrams) under [`resources/`](resources/).

## Checkpoint

The [checkpoint](checkpoint.md) is the real completion gate — answer it from
memory. See [`checkpoint.md`](checkpoint.md).

## Directory layout

| Path | What goes here |
|------|----------------|
| [`lessons/`](lessons/) | One page per concept — notes, worked example, practice, self-check. |
| [`practice/`](practice/) | Code, benchmarks, commands — the buildable output. |
| [`resources/`](resources/) | Saved references, diagrams, papers, link index. |
| [`checkpoint.md`](checkpoint.md) | Checkpoint answers (the completion gate). |
