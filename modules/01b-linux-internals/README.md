---
id: "01b"
title: "Linux systems internals"
notion: "https://app.notion.com/p/3b33abaeb823812a8e94cf07ea623410"
phase: "Phase 1 · Months 1–5 (parallel with 01)"
effort: "5–6 weeks"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: []
unlocks: []
started: null              # YYYY-MM-DD
completed: null            # set when the checkpoint passes
---

# 🐧 01b — Linux systems internals

> **Goal.** Move from using Linux to understanding the kernel mechanisms underneath, where fleet-scale failures live.

- **Notion page:** https://app.notion.com/p/3b33abaeb823812a8e94cf07ea623410
- **Phase:** Phase 1 · Months 1–5 (parallel with 01) · **Est. effort:** 5–6 weeks
- **Prerequisites:** — · **Unlocks:** —

## Objectives (must-know concepts)

Each objective maps to one lesson below. The module's objective is met only when
its lesson is `done` **and** the checkpoint question(s) for it pass.

## Lessons

| # | Lesson | Status |
|---|--------|--------|
| 01 | [Processes and scheduling](lessons/01-processes-and-scheduling.md) | `not-started` |
| 02 | [Memory management](lessons/02-memory-management.md) | `not-started` |
| 03 | [cgroups v2](lessons/03-cgroups-v2.md) | `not-started` |
| 04 | [Namespaces](lessons/04-namespaces.md) | `not-started` |
| 05 | [Filesystems and block I/O](lessons/05-filesystems-and-block-io.md) | `not-started` |
| 06 | [Networking stack](lessons/06-networking-stack.md) | `not-started` |
| 07 | [systemd](lessons/07-systemd.md) | `not-started` |
| 08 | [Kernel tunables](lessons/08-kernel-tunables.md) | `not-started` |
| 09 | [Observability tooling](lessons/09-observability-tooling.md) | `not-started` |
| 10 | [eBPF](lessons/10-ebpf.md) | `not-started` |

## Sources

Canonical reading lives on the [Notion page](https://app.notion.com/p/3b33abaeb823812a8e94cf07ea623410). Save any
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
