---
id: "03"
title: "GPU hardware fundamentals"
notion: "https://app.notion.com/p/3b33abaeb82381abaccfd55ab24bc852"
phase: "Phase 2 · Months 5–8"
effort: "3–4 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["02b"]
unlocks: ["04", "07", "11"]
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# 🔌 03 — GPU hardware fundamentals

> **Goal.** Enough GPU hardware literacy to reason about utilisation, throughput and cost.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381abaccfd55ab24bc852
- **Phase:** Phase 2 · Months 5–8 · **Est. effort:** 3–4 weeks
- **Prerequisites:** `02b` · **Unlocks:** `04`, `07`, `11`

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [Execution model](lessons/01-execution-model.md) | `not-started` |
| 02 | [Memory hierarchy](lessons/02-memory-hierarchy.md) | `not-started` |
| 03 | [Compute-bound vs memory-bound](lessons/03-compute-vs-memory-bound.md) | `not-started` |
| 04 | [Precision formats](lessons/04-precision-formats.md) | `not-started` |
| 05 | [Tensor cores](lessons/05-tensor-cores.md) | `not-started` |
| 06 | [Generational differences](lessons/06-generational-differences.md) | `not-started` |
| 07 | [The software stack](lessons/07-software-stack.md) | `not-started` |
| 08 | [Thermals and power](lessons/08-thermals-and-power.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb82381abaccfd55ab24bc852). Save any
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
