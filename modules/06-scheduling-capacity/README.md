---
id: "06"
title: "Scheduling, queueing and capacity"
notion: "https://app.notion.com/p/3b33abaeb82381d9b1baea460826f00e"
phase: "Phase 3 · Months 8–12"
effort: "5–6 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["02"]
unlocks: []
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# 🗓️ 06 — Scheduling, queueing and capacity

> **Goal.** Make a fixed, scarce GPU fleet serve many teams fairly and efficiently.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381d9b1baea460826f00e
- **Phase:** Phase 3 · Months 8–12 · **Est. effort:** 5–6 weeks
- **Prerequisites:** `02` · **Unlocks:** —

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [Why the default scheduler fails for GPUs](lessons/01-why-default-scheduler-fails.md) | `not-started` |
| 02 | [Gang scheduling / coscheduling](lessons/02-gang-scheduling.md) | `not-started` |
| 03 | [Kueue](lessons/03-kueue.md) | `not-started` |
| 04 | [Volcano](lessons/04-volcano.md) | `not-started` |
| 05 | [Topology-aware placement](lessons/05-topology-aware-placement.md) | `not-started` |
| 06 | [Fragmentation](lessons/06-fragmentation.md) | `not-started` |
| 07 | [Priority and preemption](lessons/07-priority-and-preemption.md) | `not-started` |
| 08 | [Quota models](lessons/08-quota-models.md) | `not-started` |
| 09 | [Capacity planning under scarcity](lessons/09-capacity-planning.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb82381d9b1baea460826f00e). Save any
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
