---
id: "02"
title: "Kubernetes internals and controllers"
notion: "https://app.notion.com/p/3b33abaeb8238119965bd167fd8412a4"
phase: "Phase 0–1 · Months 1–6"
effort: "~104 hrs ≈ 9–10 weeks @ 10–12 hrs/wk"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["01"]
unlocks: ["04", "06"]
started: null
completed: null
---

# ⚙️ 02 — Kubernetes internals and controllers

> **Goal.** Move from **operating** Kubernetes (you have CKA) to **extending** it —
> the layer underneath. This module forges the controller-building competence and
> produces the first real version of your capstone: `gpu-cost-operator`.

- **Notion page:** https://app.notion.com/p/3b33abaeb8238119965bd167fd8412a4
- **Phase:** Phase 0–1 · **Est. effort:** ~104 hrs ≈ 9–10 weeks @ 10–12 hrs/wk
- **Prerequisites:** `01` (Go) · **Unlocks:** `04`, `06`
- **Deliverable:** [`gpu-cost-operator` v0.1](practice/gpu-cost-operator/) — a real
  kubebuilder operator built on Module 01's `gpu-cost-exporter` core.

## Why this module, and to what bar

For your target roles, controller/operator/CRD/webhook work **is the job**:

- **NVIDIA** — Sr SWE, Kubernetes/KubeVirt: *"extend Kubernetes components — **operators/controllers, CRDs, device plugins** … automate install, upgrade, rollback with API-driven workflows."*
- **CoreWeave** — SWE, Kubernetes Core Interfaces: *"builds the control plane — **custom controllers, CRDs, gateways, API endpoints**"*; senior roles own *"scheduling, quota enforcement, hyperscale scaling"* (Kueue on GPU clusters).
- Recurring JD language: *"Go with experience building Kubernetes **controllers, operators, CRDs, and webhooks**"* and *"internals including CRDs, RBAC, custom controllers and **scheduler extensions**."*

**Interview probes:** "design an operator for X" · "explain informers/listers/workqueues" ·
"why must reconcile be idempotent" · "your controller stopped reconciling — debug it" ·
GPU depth: **DRA** (GA in k8s 1.34) and **Kueue**.

## Calibrated to your background — what we skip

You have CKA and run 40+ clusters, so we **skip** everything at the user/operator
altitude: kubectl/YAML basics, what-is-a-pod, Helm/Kustomize 101, CNI/CSI/Ingress
*usage*, cluster bring-up as a user, all CKA prep. We **start at the internals /
API-machinery / controller-runtime layer** — the machinery, not the org chart. (A
`kind` cluster is used only as a 10-minute test harness.)

## Lessons

Internals-first, ending in a real build. Spine = **L4, L6, L10**.

| # | Lesson | Hrs | Advances the operator |
|---|--------|-----|-----------------------|
| 01 | [Component internals — how each works](lessons/01-component-internals.md) | 12 | Node-watch client-go for the cost signal |
| 02 | [API machinery](lessons/02-api-machinery.md) | 6 | Typed vs unstructured GPU-capacity reads |
| 03 | [Reconciliation model](lessons/03-reconciliation-model.md) | 8 | Idempotency test on reconcile |
| 04 | [Informers, caches, workqueues](lessons/04-informers-caches-workqueues.md) | 12 | Cache-vs-live instrumentation |
| 05 | [CRD design (versioning · CEL · subresources)](lessons/05-crd-design.md) | 10 | The three `v1alpha1` CRD schemas |
| 06 | [controller-runtime deep + finalizers/GC](lessons/06-controller-runtime-deep.md) | 14 | Owned children + finalizer recording spend |
| 07 | [Kubebuilder + RBAC least-privilege](lessons/07-kubebuilder-and-rbac.md) | 8 | Minimal, justified ClusterRole |
| 08 | [Admission webhooks + failurePolicy](lessons/08-admission-webhooks.md) | 10 | Budget-enforcement validating webhook |
| 09 | [Scheduler + GPU-scheduling literacy (DRA/Kueue)](lessons/09-scheduler-and-gpu-scheduling.md) | 8 | Placement design doc |
| 10 | [Capstone build + envtest](lessons/10-capstone-build.md) | 16 | Assemble & test v0.1 |

**Scheduler (L9) is deliberately literacy/design, not a plugin build** — DRA + Kueue
fluency is the interview-visible payoff; a production scheduler plugin would blow the
budget.

## Resource spine

- **Kubebuilder Book** + **controller-runtime / client-go godoc** + **sample-controller**
  (`controller.go` read end-to-end) — the build spine.
- **Kubernetes Programming with Go** (Martin, 2023) — API machinery & client-go, current.
- **Programming Kubernetes** (Hausenblas/Schimanski) — the machinery *concepts* (dated
  code — read, don't copy).
- **DRA** KEP-4381 + kubernetes.io DRA docs, **Kueue** docs + CoreWeave's writeup — for L9.

Per-lesson skim/deep guidance lives in each lesson's **Resources**.

## Deliverable & checkpoint

- Build **[`gpu-cost-operator` v0.1](practice/gpu-cost-operator/)** across the lessons.
- The [**checkpoint**](checkpoint.md) is the completion gate — answer the probes cold
  and prove the rest with the operator running green under envtest.

## How to work this module

1. Lessons in order; do every **Practice** — they compound into the operator.
2. Keep it green: `make manifests generate`, `make test` (envtest), `golangci-lint`.
3. Answer the [checkpoint](checkpoint.md) from memory; flip `status` and update the
   Notion tracking row when the operator meets its acceptance criteria.
