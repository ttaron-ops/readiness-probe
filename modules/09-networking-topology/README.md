---
id: "09"
title: "Networking and topology"
notion: "https://app.notion.com/p/3b33abaeb82381f69438c603703f933f"
phase: "Phase 4 · Months 12–16"
effort: "3–4 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["02b"]
unlocks: []
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# 🔗 09 — Networking and topology

> **Goal.** Understand the interconnect well enough to make placement and procurement arguments.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381f69438c603703f933f
- **Phase:** Phase 4 · Months 12–16 · **Est. effort:** 3–4 weeks
- **Prerequisites:** `02b` · **Unlocks:** —

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [Intra-node interconnect](lessons/01-intra-node-interconnect.md) | `not-started` |
| 02 | [Inter-node fabric](lessons/02-inter-node-fabric.md) | `not-started` |
| 03 | [RDMA fundamentals](lessons/03-rdma-fundamentals.md) | `not-started` |
| 04 | [GPUDirect](lessons/04-gpudirect.md) | `not-started` |
| 05 | [Kubernetes networking for GPU clusters](lessons/05-k8s-networking-gpu.md) | `not-started` |
| 06 | [Topology-aware scheduling in practice](lessons/06-topology-aware-scheduling.md) | `not-started` |
| 07 | [Bandwidth as a cost input](lessons/07-bandwidth-as-cost.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb82381f69438c603703f933f). Save any
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
