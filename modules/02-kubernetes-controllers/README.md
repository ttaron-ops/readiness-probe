---
id: "02"
title: "Kubernetes internals and controllers"
notion: "https://app.notion.com/p/3b33abaeb8238119965bd167fd8412a4"
phase: "Phase 1 · Months 1–5"
effort: "8–10 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["01"]
unlocks: ["04", "06"]
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# ⚙️ 02 — Kubernetes internals and controllers

> **Goal.** Move from operating Kubernetes to extending it — the layer underneath CKA.

- **Notion page:** https://app.notion.com/p/3b33abaeb8238119965bd167fd8412a4
- **Phase:** Phase 1 · Months 1–5 · **Est. effort:** 8–10 weeks
- **Prerequisites:** `01` · **Unlocks:** `04`, `06`

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [Control plane anatomy](lessons/01-control-plane-anatomy.md) | `not-started` |
| 02 | [API machinery](lessons/02-api-machinery.md) | `not-started` |
| 03 | [The reconciliation model](lessons/03-reconciliation-model.md) | `not-started` |
| 04 | [Informers and caches](lessons/04-informers-and-caches.md) | `not-started` |
| 05 | [CRDs](lessons/05-crds.md) | `not-started` |
| 06 | [controller-runtime](lessons/06-controller-runtime.md) | `not-started` |
| 07 | [Kubebuilder](lessons/07-kubebuilder.md) | `not-started` |
| 08 | [Admission webhooks](lessons/08-admission-webhooks.md) | `not-started` |
| 09 | [Scheduler internals](lessons/09-scheduler-internals.md) | `not-started` |
| 10 | [RBAC and identity for controllers](lessons/10-rbac-for-controllers.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb8238119965bd167fd8412a4). Save any
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
