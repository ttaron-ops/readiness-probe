---
id: "10"
title: "Bare metal and cluster lifecycle"
notion: "https://app.notion.com/p/3b33abaeb823817fb539ecac9645878f"
phase: "Phase 4 · Months 12–16"
effort: "4–5 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["02b"]
unlocks: []
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# 🖥️ 10 — Bare metal and cluster lifecycle

> **Goal.** Close the bare-metal gap: provision and operate clusters where the control plane is yours.

- **Notion page:** https://app.notion.com/p/3b33abaeb823817fb539ecac9645878f
- **Phase:** Phase 4 · Months 12–16 · **Est. effort:** 4–5 weeks
- **Prerequisites:** `02b` · **Unlocks:** —

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [Cluster provisioning](lessons/01-cluster-provisioning.md) | `not-started` |
| 02 | [etcd operations](lessons/02-etcd-operations.md) | `not-started` |
| 03 | [Control plane HA](lessons/03-control-plane-ha.md) | `not-started` |
| 04 | [Node lifecycle](lessons/04-node-lifecycle.md) | `not-started` |
| 05 | [Hardware health](lessons/05-hardware-health.md) | `not-started` |
| 06 | [Storage for AI](lessons/06-storage-for-ai.md) | `not-started` |
| 07 | [On-prem vs cloud economics](lessons/07-onprem-vs-cloud-economics.md) | `not-started` |
| 08 | [Load balancing and ingress without a cloud provider](lessons/08-load-balancing-ingress.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb823817fb539ecac9645878f). Save any
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
