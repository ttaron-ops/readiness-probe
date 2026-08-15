---
id: "01"
title: "Go for infrastructure engineers"
notion: "https://app.notion.com/p/3b33abaeb82381e499c2faec468fb993"
phase: "Phase 0 · Months 1–3 (the gate)"
effort: "~110 hrs ≈ 10 weeks @ 11 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: []
unlocks: ["02", "04"]
started: null
completed: null
---

# 🐹 01 — Go for infrastructure engineers

> **Goal.** Write production Go and read the source of any tool you operate
> (Kubernetes, controllers, exporters). This module gates every other module —
> and it bends toward one thing: **writing controllers**.

- **Notion page:** https://app.notion.com/p/3b33abaeb82381e499c2faec468fb993
- **Phase:** Phase 0 · the gate · **Est. effort:** ~110 hrs ≈ 10 weeks @ 11 hrs/wk
- **Deliverable:** [`gpu-cost-exporter`](practice/gpu-cost-exporter/) — a tested Go
  exporter/CLI whose core drops straight into your capstone controller.

## Why this module, and to what bar

Go is **table stakes** for the roles you're targeting, and it's tied explicitly to
controllers/operators — which is your differentiator:

- **CoreWeave** — Sr GPU Infrastructure SW Eng: *"backend services and APIs (gRPC/REST) in **Go** to interact with Kubernetes."*
- **NVIDIA** — Sr Platform/AI-ML Infra: *"scalable **Go programs deployed to Kubernetes** that manage large datacenters."*
- **Cisco** — *Kubernetes Platform Engineer, AI Infrastructure (**Golang**/Python)*: *"controllers, operators, CRDs, and Golang services."*

Interview format for these roles favors you: mostly **take-home / practical** ("write
a small tested service or CLI") plus "walk me through your controller" — **not**
LeetCode-in-Go. So this module optimizes for *shipping a real, tested Go tool and
being able to read k8s source*, with only a thin "type idiomatic Go fast" layer.

## Calibrated to your background — what we skip

You already program (Python) and know systems/k8s, so we **skip**: programming 101,
pointers-as-concept, IDE/hello-world tours, OOP-in-Go, LeetCode/DSA grind,
web-framework/CRUD tutorials, reflection/cgo/runtime internals, and cover-to-cover
reading. You get Go's *spelling* of what you know, fast — then spend real hours only
where the senior signal is (interfaces, concurrency, testing, reading source,
controllers).

## Lessons

Order matters; emphasis is deliberate. **Never shortcut 3, 4, or 9.**

| # | Lesson | Emphasis | Hrs |
|---|--------|----------|-----|
| 01 | [Syntax & types](lessons/01-syntax-and-types.md) | light | 7 |
| 02 | [Error handling](lessons/02-error-handling.md) | high | 9 |
| 03 | [Interfaces & composition](lessons/03-interfaces-and-composition.md) | **highest** | 14 |
| 04 | [Concurrency & context](lessons/04-concurrency-and-context.md) | **highest** | 18 |
| 05 | [Testing](lessons/05-testing.md) | high | 14 |
| 06 | [Modules & layout](lessons/06-modules-and-layout.md) | light | 6 |
| 07 | [Stdlib fluency](lessons/07-stdlib-fluency.md) | medium | 12 |
| 08 | [Reading unfamiliar Go](lessons/08-reading-unfamiliar-go.md) | medium | 12 |
| 09 | [Controller primer (CRD · reconcile · envtest)](lessons/09-controller-primer.md) | high | 18 |

Each lesson is self-contained: *why it matters · the delta from Python · core notes ·
a worked example · a practice task (that builds the deliverable) · self-check Q&A ·
curated resources.*

## Resource spine (the few things worth buying/reading)

- **Learning Go, 2nd ed.** — Jon Bodner (O'Reilly, 2024) — idiomatic Go for people who
  already program. Deep-read interfaces/errors/concurrency/testing; skim early syntax.
- **100 Go Mistakes and How to Avoid Them** — Harsanyi (free companion: <https://100go.co>) —
  the "Effective Java" of Go; leapfrogs the "not really a Go dev" tells.
- **One** concurrency book — *Concurrency in Go* (Cox-Buday) or *Learn Concurrent
  Programming with Go* (Cutajar, 2024).
- **Kubebuilder Book** + **controller-runtime** + **sample-controller** source — for
  lessons 8–9, the controller machinery and the "read real source" gym.
- Tour of Go / Go by Example — skim-and-lookup only.

Per-lesson curation (what to skim vs deep-read) lives in each lesson's **Resources**.

## Deliverable & checkpoint

- Build **[`gpu-cost-exporter`](practice/gpu-cost-exporter/)** as you go — each lesson's
  practice task adds a piece.
- The [**checkpoint**](checkpoint.md) is the completion gate — the "you've passed the
  Go gate" bar. Answer it from memory; the deliverable is the proof.

## How to work this module

1. Do lessons in order; do every **Practice** task — they compound into the deliverable.
2. Keep the deliverable green: `go vet`, `golangci-lint`, `go test -race` after each addition.
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` to `checkpoint-passed`
   and update the Notion tracking row when the deliverable meets its acceptance criteria.
