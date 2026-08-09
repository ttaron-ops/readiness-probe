---
id: "02b"
title: "Host hardware and topology"
notion: "https://app.notion.com/p/3b33abaeb82381b48bc9e928ef7cb695"
phase: "Phase 2 · Months 5–8 (precedes 03)"
effort: "3–4 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: []
unlocks: ["03", "04", "09", "10"]
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# 🧬 02b — Host hardware and topology

> **Goal.** Understand the machine underneath the accelerator, and catch host-side waste nothing in monitoring reports.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381b48bc9e928ef7cb695
- **Phase:** Phase 2 · Months 5–8 (precedes 03) · **Est. effort:** 3–4 weeks
- **Prerequisites:** — · **Unlocks:** `03`, `04`, `09`, `10`

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [CPU architecture at the level that matters](lessons/01-cpu-architecture.md) | `not-started` |
| 02 | [NUMA](lessons/02-numa.md) | `not-started` |
| 03 | [Memory subsystem](lessons/03-memory-subsystem.md) | `not-started` |
| 04 | [PCIe](lessons/04-pcie.md) | `not-started` |
| 05 | [Server architecture for AI](lessons/05-server-architecture-for-ai.md) | `not-started` |
| 06 | [Kubernetes topology alignment](lessons/06-topology-alignment-k8s.md) | `not-started` |
| 07 | [Storage hardware](lessons/07-storage-hardware.md) | `not-started` |
| 08 | [Power and thermals](lessons/08-power-and-thermals.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb82381b48bc9e928ef7cb695). Save any
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
