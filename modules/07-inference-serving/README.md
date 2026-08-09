---
id: "07"
title: "Inference serving"
notion: "https://app.notion.com/p/3b33abaeb823810aa06bf912b8380bb1"
phase: "Phase 3 · Months 8–12"
effort: "5–6 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["03"]
unlocks: ["11"]
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# 🚀 07 — Inference serving

> **Goal.** Operate production inference and reason about its unit economics.

- **Notion page:** https://app.notion.com/p/3b33abaeb823810aa06bf912b8380bb1
- **Phase:** Phase 3 · Months 8–12 · **Est. effort:** 5–6 weeks
- **Prerequisites:** `03` · **Unlocks:** `11`

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [The inference workload shape](lessons/01-inference-workload-shape.md) | `not-started` |
| 02 | [KV cache](lessons/02-kv-cache.md) | `not-started` |
| 03 | [PagedAttention and vLLM](lessons/03-pagedattention-and-vllm.md) | `not-started` |
| 04 | [vLLM in production](lessons/04-vllm-in-production.md) | `not-started` |
| 05 | [Batching economics](lessons/05-batching-economics.md) | `not-started` |
| 06 | [Alternative servers](lessons/06-alternative-servers.md) | `not-started` |
| 07 | [Quantization for cost](lessons/07-quantization.md) | `not-started` |
| 08 | [Autoscaling inference](lessons/08-autoscaling-inference.md) | `not-started` |
| 09 | [Model loading and storage](lessons/09-model-loading-and-storage.md) | `not-started` |
| 10 | [Multi-model serving](lessons/10-multi-model-serving.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb823810aa06bf912b8380bb1). Save any
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
