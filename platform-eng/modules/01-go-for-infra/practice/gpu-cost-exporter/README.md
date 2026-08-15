# `gpu-cost-exporter` — Module 01 deliverable

A small, real, tested Go program you build **incrementally across the lessons**. It's
a Prometheus exporter + CLI that reads GPU cost/usage data and exposes per-namespace
cost-efficiency gauges on `/metrics`.

> **Why this one.** It exercises every lesson (types, errors, interfaces,
> concurrency+context, testing, stdlib, module layout), and its core — the
> aggregation logic and the `CostSource` interface — **drops straight into your
> capstone GPU cost/efficiency controller** later. There, the controller becomes just
> another consumer that writes the same numbers to a CRD `.status` instead of to
> `/metrics`. One tested core, two front-ends.
>
> Portfolio-legible in exactly the JD language: *"backend services in Go interacting
> with Kubernetes."*

## Build order (each lesson adds a piece)

| From lesson | You add |
|-------------|---------|
| 01 Syntax & types | Parse a JSON cost export → structs; aggregate GPU cost by namespace/label; print a sorted table. |
| 02 Error handling | Typed + sentinel errors, `%w` wrapping; classify not-found (skip) vs transient (retry). |
| 03 Interfaces | Put the data source behind a `CostSource` interface; one real impl + one fake. |
| 04 Concurrency & context | Make collection a **bounded concurrent scrape** with a `context` deadline that cancels cleanly. |
| 05 Testing | Table-driven tests on aggregation, a fake `CostSource`, one benchmark; `go test -race -cover`. |
| 06 Modules & layout | Split into a module: `internal/` packages + `cmd/`; pin `prometheus/client_golang`. |
| 07 Stdlib fluency | Real CLI (cobra/flag) + `slog` logging + `/metrics` HTTP endpoint with a `GaugeVec`. |
| 08 Reading source | (Alongside) commit `trace.md` — how controller-runtime's `Reconcile` gets called. |
| 09 Controller primer | Wrap the same core in a tiny `CostBudget` CRD + reconciler (envtest) — the capstone seed. |

## Suggested layout (grows into a controller repo)

```
gpu-cost-exporter/
├── cmd/gpu-cost-exporter/main.go   # CLI entrypoint (cobra/flag)
├── internal/
│   ├── cost/                       # CostSource interface + aggregation (the reusable core)
│   ├── source/                     # real + fake implementations
│   └── metrics/                    # Prometheus GaugeVec registration
├── operator/                       # lesson 09: kubebuilder-scaffolded CostBudget CRD + reconciler
│   ├── api/v1alpha1/                #   CostBudget types
│   └── internal/controller/         #   Reconciler, wired to the same internal/cost core
├── trace.md                        # lesson 08 reading-source artifact
├── go.mod
└── README.md
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] `go build ./...` and runs; `curl localhost:PORT/metrics` shows a correct per-namespace gauge
- [ ] `go vet` + `golangci-lint` clean
- [ ] `go test -race -cover` green; ~70%+ coverage on `internal/cost`; a fake source; one benchmark
- [ ] `CostSource` is an interface; swapping impls needs no change to aggregation
- [ ] errors wrapped with `%w` and classified retryable vs terminal
- [ ] `context` threaded through all I/O; scrape is bounded and cancels cleanly (leak-free)
- [ ] structured `slog` logging
- [ ] `trace.md` committed and correct
- [ ] (stretch / lesson 09) `CostBudget` CRD + reconciler passing an `envtest` test

## Guardrails

- Keep the business logic **trivial and correct** — the point is the machinery, not clever math.
- Open-source-by-default, but confirm the licence/employer conversation before adding
  any real Skyro data or internals (see [ROADMAP Phase 0](../../../../../docs/ROADMAP.md)).
- No secrets, kubeconfigs, or real cost figures in git (the repo `.gitignore` guards these).
