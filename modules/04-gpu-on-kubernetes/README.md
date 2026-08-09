---
id: "04"
title: "GPU on Kubernetes"
notion: "https://app.notion.com/p/3b33abaeb82381ff9762c82e3302fdbe"
phase: "Phase 2 · Months 5–8"
effort: "6–8 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["01", "02", "03"]
unlocks: ["05", "06"]
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# 📦 04 — GPU on Kubernetes

> **Goal.** Own the GPU integration layer end to end — the core operational module.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381ff9762c82e3302fdbe
- **Phase:** Phase 2 · Months 5–8 · **Est. effort:** 6–8 weeks
- **Prerequisites:** `01`, `02`, `03` · **Unlocks:** `05`, `06`

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [The device plugin API](lessons/01-device-plugin-api.md) | `not-started` |
| 02 | [NVIDIA GPU Operator](lessons/02-gpu-operator.md) | `not-started` |
| 03 | [Driver management](lessons/03-driver-management.md) | `not-started` |
| 04 | [MIG](lessons/04-mig.md) | `not-started` |
| 05 | [Time-slicing](lessons/05-time-slicing.md) | `not-started` |
| 06 | [MPS](lessons/06-mps.md) | `not-started` |
| 07 | [Dynamic Resource Allocation](lessons/07-dynamic-resource-allocation.md) | `not-started` |
| 08 | [Node labelling and feature discovery](lessons/08-node-labelling-nfd.md) | `not-started` |
| 09 | [Container runtime integration](lessons/09-container-runtime-integration.md) | `not-started` |
| 10 | [Multi-tenancy](lessons/10-multi-tenancy.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb82381ff9762c82e3302fdbe). Save any
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
