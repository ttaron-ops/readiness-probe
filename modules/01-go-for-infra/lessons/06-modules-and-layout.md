---
lesson: "01.6"
title: "Modules and Project Layout"
module: "01"
concept: "Modules and Project Layout"
status: not-started
est_time: "5h"
artifacts: []
---

# 01.6 · Modules and Project Layout

> **Concept.** `go.mod`/`go.sum`, semantic import versioning, `replace`, workspaces, `internal/`, and the standard controller repo layout.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Why this matters
A controller repo is a module, and reviewers judge you on its shape before they read a line of logic. Knowing why `internal/` exists, when a `replace` directive is legitimate, and how `go.work` unblocks cross-repo development separates someone who *uses* Go from someone who *ships* Go services. The Kubernetes ecosystem is a mesh of interdependent modules with strict version discipline — fumbling `go.mod` in an interview or a PR reads as junior.

## From Python to Go
`go.mod` is `pyproject.toml`, `go.sum` is the hash-pinned lockfile — but there's no virtualenv and no active environment to "be in." The module graph is resolved per-build from `go.mod`, globally cached under `$GOPATH/pkg/mod`, and version selection is deterministic via **Minimal Version Selection**: Go picks the *lowest* version that satisfies all requirements, the opposite of pip/npm grabbing the newest compatible. There's no `requirements.txt` freeze step — `go.sum` is maintained automatically and `go mod tidy` prunes it. And unlike Python's honor-system `_private`, Go's `internal/` is a **compiler-enforced** visibility boundary.

## Core notes

**`go.mod` and `go.sum`.** `go.mod` declares the module path, the Go version, and dependency requirements. `go.sum` records cryptographic hashes of every module version in the build graph for tamper-proof, reproducible builds. Commit both.

```
module github.com/tarhov/gpu-cost-exporter

go 1.22

require (
	github.com/prometheus/client_golang v1.19.1
)
```

The module path is also the import prefix: code here imports `github.com/tarhov/gpu-cost-exporter/internal/exporter`. Core commands: `go mod init <path>`, `go get <pkg>@<version>`, `go mod tidy` (add missing, drop unused, sync `go.sum`), `go build ./...`.

**Semantic import versioning.** Go encodes the *major* version in the import path for v2+. `v0`/`v1` use the bare path; **v2 and above must add a `/vN` suffix** to both the module path in `go.mod` and every import:

```go
import "github.com/foo/bar/v2"   // module github.com/foo/bar/v2
```

This is why you see `sigs.k8s.io/controller-runtime` (still v0.x, no suffix) but `github.com/prometheus/common` deps sometimes carry `/v2`. The rule: a new major version is a *different import path*, so two majors can coexist in one build. Pin a version with `go get pkg@v1.19.1`; it lands in `require` and is hashed in `go.sum`.

**`replace`.** A `replace` directive swaps one module source for another at build time — a local path or a fork:

```
replace github.com/foo/bar => ../bar               // local checkout you're editing
replace github.com/foo/bar => github.com/me/bar v1.2.0-fork  // your fork/patch
```

Legitimate uses: developing two modules in lockstep before publishing, testing a patch against an upstream bug, or forcing a transitive dependency to a fixed fork. It's build-local and does **not** propagate to consumers of your module — which is exactly why the *workspace* mechanism was added for multi-module dev (below). A lingering `replace => ../local` in a published module breaks everyone downstream; remove it before release.

**Workspaces (`go.work`).** A workspace lets you develop **multiple modules together** without editing any `go.mod`. `go work init ./exporter ./shared` writes a `go.work` that tells the toolchain "use these local module directories as the source for these paths." Every `go build`/`go test` in the workspace resolves those modules locally.

```
go 1.22

use (
	./gpu-cost-exporter
	./gpu-shared-lib
)
```

The problem it solves that vendoring/`replace` doesn't: `replace` mutates a committed `go.mod` (easy to leak into a release), and vendoring copies frozen deps into the repo. `go.work` is a **local, uncommitted overlay** — you keep it out of git (`.gitignore`), so no per-module file is polluted and there's nothing to accidentally publish. It's the clean way to iterate across the controller repo and a shared library at once.

**`internal/`.** Any package under a directory named `internal/` is importable **only by code rooted at `internal/`'s parent**. The compiler enforces this — an outside module importing `.../internal/exporter` fails to build. It is real API surface control, not convention: it lets you refactor everything under `internal/` freely because nothing external can depend on it. Put everything that isn't a deliberate public API there.

```
github.com/tarhov/gpu-cost-exporter/
  internal/exporter/   ← importable only within this module
  cmd/exporter/        ← can import internal/, external repos cannot
```

**Standard controller repo layout.** Kubebuilder scaffolds — and hiring managers expect — this shape:

```
gpu-cost-exporter/
  go.mod
  cmd/
    main.go              ← wiring only: flags, manager setup, Start()
  api/
    v1alpha1/            ← CRD types (Go structs + kubebuilder markers)
  internal/
    controller/          ← reconcilers (the *Reconciler types + Reconcile)
    exporter/            ← business logic (aggregation, cost math)
  config/                ← kustomize manifests (CRDs, RBAC, deployment)
  Makefile
```

`cmd/main.go` stays thin — it constructs the manager and registers controllers. Types live in `api/`, logic in `internal/controller/` (newer kubebuilder uses `internal/controller/`; older `controllers/` at root — both are current in the wild). Keeping reconcilers in `internal/` is deliberate: no one imports your controller as a library.

## Worked example
Turning the flat exporter script into a proper module with an enforced boundary and a pinned third-party dep:

```bash
# 1. Initialize the module (path == repo, becomes the import prefix)
go mod init github.com/tarhov/gpu-cost-exporter

# 2. Move logic behind internal/, keep main thin
#    internal/exporter/collect.go   -> package exporter (business logic)
#    cmd/exporter/main.go           -> package main (wiring)

# 3. Add and pin a real dependency
go get github.com/prometheus/client_golang@v1.19.1

# 4. Sync and verify the whole tree builds
go mod tidy
go build ./...
```

`cmd/exporter/main.go` — wiring only, importing the internal package it's allowed to see:

```go
package main

import (
	"log"
	"net/http"

	"github.com/prometheus/client_golang/prometheus/promhttp"
	"github.com/tarhov/gpu-cost-exporter/internal/exporter"
)

func main() {
	reg := exporter.MustRegister() // sets up collectors, returns *prometheus.Registry
	http.Handle("/metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{}))
	log.Println("serving :9090/metrics")
	log.Fatal(http.ListenAndServe(":9090", nil))
}
```

The result: `github.com/prometheus/client_golang v1.19.1` sits pinned in `go.mod`, hashed in `go.sum`; the aggregation logic under `internal/exporter/` is unimportable by any other repo; and this is the exact skeleton the capstone controller grows into by adding `api/` and `internal/controller/`.

## Practice
Restructure `gpu-cost-exporter` into a real module:

1. `go mod init github.com/<you>/gpu-cost-exporter`.
2. Move aggregation/cost logic into `internal/exporter/`; keep only wiring in `cmd/exporter/main.go`.
3. Add and **pin** one third-party dependency — `github.com/prometheus/client_golang` at an explicit version — then `go mod tidy`.
4. Sketch the kubebuilder-style growth path by adding empty `api/v1alpha1/` and `internal/controller/` directories with a one-line doc comment each.

**Acceptance:** `go build ./...` succeeds; the `internal/` boundary holds (a scratch external import of the internal package fails to compile — verify by reasoning or a throwaway module); `go.mod` shows exactly one pinned third-party `require` and `go.sum` has its hashes.

## Self-check
1. **What does `internal/` actually enforce?**
   **Answer:** Compiler-enforced import visibility: a package under `.../internal/...` can be imported **only** by code whose path is rooted at the directory containing that `internal/`. Any other module (or a sibling subtree) that tries to import it fails to build. It's a hard boundary, not a naming convention, which is what makes everything under `internal/` safe to refactor.

2. **When do you need a `replace` directive?**
   **Answer:** When you must build against a module source other than what the version graph resolves to — a local checkout you're editing in lockstep, a fork carrying a patch, or forcing a transitive dep to a fixed version during debugging. It's build-local and doesn't propagate to your consumers, so it's fine in application repos and internal iteration but must be removed before publishing a library. For pure multi-module local dev, prefer `go.work` over committing a `replace`.

3. **What problem do workspaces (`go.work`) solve that vendoring doesn't?**
   **Answer:** They provide a **local, uncommitted overlay** for developing several modules together without touching any module's `go.mod`. Vendoring copies frozen dependency snapshots into the repo (reproducibility, not active development), and a `replace` mutates a committed file that can leak into a release. `go.work` stays out of git and out of every `go.mod`, so you can iterate across the controller repo and a shared library simultaneously with nothing to accidentally publish.

## Resources
1. **Managing dependencies** — https://go.dev/doc/modules/managing-dependencies — task-oriented guide to `go get`, `tidy`, versions, and `replace`. *Skim.* Why: the fastest path to correct day-to-day module hygiene.
2. **Organizing a Go module** — https://go.dev/doc/modules/layout — official guidance on `cmd/`, `internal/`, and package layout. *Skim.* Why: maps directly to the controller repo shape reviewers expect.
3. **Go Modules Reference** — https://go.dev/ref/mod — authoritative spec: MVS, semantic import versioning, `go.sum`, workspaces. *Deep* (as reference). Why: the definitive source when a version-resolution question gets subtle.
