# `gpu-cost-operator` v0.1 — Module 02 deliverable

The real first version of your **capstone**: a Kubernetes operator that attributes GPU
spend to workloads and enforces budgets. Built with kubebuilder across this module's
lessons, on top of Module 01's [`gpu-cost-exporter`](../../../01-go-for-infra/practice/gpu-cost-exporter/)
core (the `CostSource` interface and aggregation logic drop straight in).

> **Why this is the deliverable.** It exercises every lesson (component internals →
> API machinery → informers → CRDs → controller-runtime → RBAC → webhooks → envtest),
> it's the exact JD artifact ("build controllers, operators, CRDs, webhooks"), and it
> becomes the thing you open-source and put in production. Later modules extend it
> (fractional attribution, DRA support, unit economics).

## Scope guardrails (so v0.1 finishes)

- **Single cluster**, `v1alpha1` CRDs only (no conversion webhook — that's later).
- Cost inputs = a **static price table** (`$/GPU-hour` by class) + Node allocatable.
  **No live cloud-billing integration yet.**
- **Validating** webhook only (no mutating webhook).
- Scheduler influence stays a **design doc** (Lesson 09), not code.
- No real Skyro data/internals until the open-source/licence conversation is done.

## CRDs (`v1alpha1`)

| Kind | Spec (selected) | Status (selected) |
|------|-----------------|-------------------|
| `GPUCostPolicy` | price model — `$/GPU-hour` by GPU class | `Ready` condition |
| `WorkloadCost` | target workload ref/selector | computed `allocatedGpuHours`, `cost`, `Ready` |
| `Budget` | `limit`, `window`, namespace/scope selector, `enforcement: report\|warn\|block` | `spend`, `BudgetExceeded` condition |

All with **CEL validation** (e.g. `limit >= 0`, currency enum), `/status` subresources,
and `additionalPrinterColumns`.

## Controller behavior

- Reconciles observed GPU usage (Node allocatable + price table) into `WorkloadCost.status`.
- **Idempotent + level-triggered** — reconciles from observed state, safe to run N times.
- Creates/owns `WorkloadCost` children via `SetControllerReference` (owner-ref GC).
- A **finalizer** records final spend before a `WorkloadCost` is deleted.
- Status **conditions** (`Ready`, `BudgetExceeded`).

## Admission webhook

- **Validating** webhook rejecting workloads whose projected cost would breach an active
  `Budget` set to `block`.
- Scoped with `namespaceSelector` (exclude `kube-system`); **`failurePolicy` chosen
  deliberately and documented** (avoid self-lockout).

## RBAC

Least-privilege, generated from `+kubebuilder:rbac` markers. **No wildcard verbs**; every
grant justified in a comment.

## Suggested layout (kubebuilder standard)

```
gpu-cost-operator/
├── api/v1alpha1/            # CRD types + markers (GPUCostPolicy, WorkloadCost, Budget)
├── internal/
│   ├── controller/         # reconcilers
│   ├── webhook/            # validating budget webhook
│   └── cost/               # reused core from gpu-cost-exporter (CostSource + aggregation)
├── config/                 # generated CRDs, RBAC, webhook manifests (make manifests)
├── cmd/main.go
├── Makefile                # make manifests generate test
└── PROJECT
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] `make manifests generate` reproducible; `make test` (envtest) green
- [ ] reconcile tests prove **idempotency** and **level-triggering**
- [ ] apply policy + workload → correct owner-referenced `WorkloadCost.status` + `Ready`
- [ ] parent delete cascades; finalizer records final spend
- [ ] budget-breaching workload **rejected by webhook**; `failurePolicy` behavior demonstrable
- [ ] CEL rejects invalid specs (negative budget)
- [ ] RBAC has **zero wildcard verbs**; every grant justified
- [ ] a `design.md` (Lesson 09) covering Score-plugin vs Kueue-quota vs DRA placement

## Guardrails

- Open-source-by-default, but confirm the licence/employer conversation first
  (see [ROADMAP Phase 0](../../../../../docs/ROADMAP.md)).
- No secrets, kubeconfigs, or real cost figures in git (repo `.gitignore` guards these).
- Keep business logic trivial and correct — the point is the machinery.
