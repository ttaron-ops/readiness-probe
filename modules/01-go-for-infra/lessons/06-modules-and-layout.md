---
lesson: "01.6"
title: "Modules and Project Layout"
module: "01"
concept: "Modules and Project Layout"
status: not-started
est_time: "6h"
prev: "05-testing.md"
next: "07-stdlib-fluency.md"
artifacts: []
sources: 7
---

# 01.6 · Modules and Project Layout

> **Concept.** `go.mod`/`go.sum`, Minimal Version Selection, semantic import versioning,
> `replace`, workspaces, `internal/`, and the standard controller repo layout — plus why each of
> these is a supply-chain and org-boundary mechanism, not just file-organization trivia.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 05 closed the "write it, prove it" arc — interfaces, concurrency, and testing, the three
lessons the module README says never to shortcut. With correct, tested code in hand, this lesson
answers the next question a reviewer or CI pipeline asks: how is this code *packaged* — what's its
module path, what dependencies does it pin and why, what's public API versus internal
implementation, and does the repo look like the ones this ecosystem already trusts? It's
deliberately **light emphasis** (this is mechanical knowledge you apply once and rarely revisit)
but it's not optional: a controller repo is a module, and reviewers judge you on its shape before
they read a line of logic. This lesson is also where the `internal/` boundary you tested against
in lesson 05 becomes a compiler-enforced contract, and it sets up lesson 07, **Stdlib fluency**,
which assumes you already know how to add and pin a real third-party dependency like
`prometheus/client_golang` cleanly.

## Why this matters

A controller repo is a module, and reviewers judge you on its shape before they read a line of
logic. Knowing why `internal/` exists, when a `replace` directive is legitimate, and how `go.work`
unblocks cross-repo development separates someone who *uses* Go from someone who *ships* Go
services. The Kubernetes ecosystem is a mesh of interdependent modules with strict version
discipline — fumbling `go.mod` in an interview or a PR reads as junior. There's a supply-chain
dimension too, and it's not abstract: Google Cloud's own guidance on dependency management frames
pinned, hash-verified dependencies as a security control, not just a reproducibility nicety (see
Real-world use cases, below) — and a `go.sum` mismatch is exactly the mechanism that turns a
compromised or yanked upstream release into a build failure instead of a silent supply-chain
compromise.

## What's new here (calibration)

You already know the shape of a dependency manifest and lockfile from Python (`pyproject.toml` /
`requirements.txt` / a hash-pinned lockfile) — we skip re-explaining what a lockfile is for or why
pinning matters in the abstract. What's genuinely new, calibrated to a staff-level track:

- **Minimal Version Selection (MVS) mechanics** — the actual algorithm Go uses, and the specific
  misconception ("it always picks the oldest version") that's worth being able to correct on the
  spot.
- **`go.sum` as a tamper-evidence and transparency-log mechanism** (via `sum.golang.org`), not
  merely a version pin — the security reasoning behind it, not just the command to run.
- **`internal/` as a compiler-enforced org boundary**, not a Python `_private`-style convention —
  what that actually buys a team refactoring aggressively across repos.
- **The operational failure mode of a leaked `replace => ../local`** in a published release, and
  what a CI check to catch it looks like — a real incident class, not a hypothetical.

## Core concepts

### `go.mod` and `go.sum`

`go.mod` declares the module path, the Go version, and dependency requirements. `go.sum` records
cryptographic hashes of every module version in the build graph for tamper-proof, reproducible
builds. Commit both.

```
module github.com/tarhov/gpu-cost-exporter

go 1.22

require (
	github.com/prometheus/client_golang v1.19.1
)
```

The module path is also the import prefix: code here imports
`github.com/tarhov/gpu-cost-exporter/internal/exporter`. Core commands: `go mod init <path>`,
`go get <pkg>@<version>`, `go mod tidy` (add missing, drop unused, sync `go.sum`),
`go build ./...`. Skipping `go mod tidy` after a refactor is a common, easy-to-miss mistake: it
leaves stale `require` entries in `go.mod` that `go.sum` still carries hashes for, quietly
bloating the dependency graph with things nothing imports anymore.

### Minimal Version Selection — the actual algorithm

Go's module resolution is deliberately boring and deterministic — the opposite of npm/pip's "grab
the newest compatible" behavior. For every module in the build graph, Go computes, for each
dependency, the **maximum version required by any importer** in that graph, and builds against
that. This is worth stating precisely because it's commonly oversimplified in both directions: MVS
does **not** mean "always pick the latest version available" (it ignores versions nothing in the
graph asked for), and it does **not** mean "always pick the oldest version" either (a common
misconception baked into the very name "minimal"). "Minimal" refers to minimizing *unnecessary*
version bumps — Go never upgrades a dependency past what's actually required — not to defaulting
to old versions. The practical effect: two machines building the same commit always get the same
dependency graph, deterministically, with no separate lockfile-freeze step the way npm/pip need
one — `go.sum` is generated as a byproduct of resolution, not a separate freeze command.

### `go.sum` and the module checksum database

`go.sum` is a tamper-evidence mechanism, not just a version pin. Every hash in it must match on
every build; a mismatch fails the build rather than silently substituting in a different artifact.
By default (`GOSUMDB=sum.golang.org`), the first time your machine fetches a given module version,
the `go` command verifies its hash against the **Go module checksum database** — a public,
append-only transparency log — before trusting it. That means a compromised or yanked upstream
release fails verification even on a machine that has never fetched that dependency before; it's
not relying on "I've seen this hash before," it's checking against a public log every party can
audit. This directly extends the "pin dependencies, verify hashes, monitor for security fixes"
guidance Google Cloud publishes for exactly this reason (see Real-world use cases).

### Semantic import versioning

Go encodes the *major* version in the import path for v2+. `v0`/`v1` use the bare path; **v2 and
above must add a `/vN` suffix** to both the module path in `go.mod` and every import:

```go
import "github.com/foo/bar/v2"   // module github.com/foo/bar/v2
```

This is why you see `sigs.k8s.io/controller-runtime` (still v0.x, no suffix) but
`github.com/prometheus/common` deps sometimes carry `/v2`. The rule: a new major version is a
*different import path*, so two majors can coexist in one build. Pin a version with
`go get pkg@v1.19.1`; it lands in `require` and is hashed in `go.sum`.

### `replace`

A `replace` directive swaps one module source for another at build time — a local path or a fork:

```
replace github.com/foo/bar => ../bar               // local checkout you're editing
replace github.com/foo/bar => github.com/me/bar v1.2.0-fork  // your fork/patch
```

Legitimate uses: developing two modules in lockstep before publishing, testing a patch against an
upstream bug, or forcing a transitive dependency to a fixed fork. It's build-local and does **not**
propagate to consumers of your module — which is exactly why the *workspace* mechanism was added
for multi-module dev (below).

**The operational trap:** a dangling `replace => ../local` directive that leaks into a
published/tagged release is a real "works on my machine, breaks for every consumer" incident
class. It builds fine for you (the local path exists on your machine) and fails or silently
resolves differently for every downstream consumer who doesn't have `../local` at that path. This
isn't hypothetical — it's exactly the kind of thing a release pipeline should refuse to ship: CI
should fail a release build if `go.mod` contains a `replace` directive pointing at a local
filesystem path, checked as a matter of course before tagging, not caught by a confused user
filing an issue.

### Workspaces (`go.work`)

A workspace lets you develop **multiple modules together** without editing any `go.mod`.
`go work init ./exporter ./shared` writes a `go.work` that tells the toolchain "use these local
module directories as the source for these paths." Every `go build`/`go test` in the workspace
resolves those modules locally.

```
go 1.22

use (
	./gpu-cost-exporter
	./gpu-shared-lib
)
```

The problem it solves that vendoring/`replace` doesn't: `replace` mutates a committed `go.mod`
(easy to leak into a release, per above), and vendoring copies frozen deps into the repo.
`go.work` is a **local, uncommitted overlay** — you keep it out of git (`.gitignore`), so no
per-module file is polluted and there's nothing to accidentally publish. Its scope is easy to
misjudge: `go.work` affects only builds run from that directory on that machine — it is *not*
something that ever affects consumers of a published module, since it's never committed and never
shipped. Expecting it to influence how anyone downstream resolves your module is the common
mistake; it's a pure local development convenience.

### Vendoring — the third option

`go mod vendor` copies the exact dependency source tree into a `vendor/` directory in the repo,
and `go build -mod=vendor` builds against that copy instead of the module cache or proxy. It's
less common since modules matured, but teams still reach for it in specific situations: **air-gapped
builds** (no network access to a module proxy at build time), **strict license auditing** (legal
wants the literal dependency source checked into the repo they're reviewing), or **avoiding a
runtime dependency on the module proxy** entirely (a proxy outage shouldn't be able to break your
build). It trades repo size and a bit of `git diff` noise on dependency bumps for a hermetic build
that needs nothing but the checked-out repo.

### `internal/`

Any package under a directory named `internal/` is importable **only by code rooted at
`internal/`'s parent**. The compiler enforces this — an outside module importing
`.../internal/exporter` fails to build. It is real API surface control, not convention: it lets
you promise "nothing outside this repo depends on `internal/controller`" and *mean* it, letting
you refactor aggressively without a cross-team deprecation cycle. This is a different guarantee
than Python's honor-system `_private` prefix — there, "private" is a documentation convention a
determined caller can simply ignore; here, the import fails to compile. The common mistake with
`internal/` isn't misusing it, it's *under-trusting* it — adding defensive "do not import this
from outside" comments or runtime checks on top of a boundary the compiler already guarantees.
Trust the boundary; spend the comment budget elsewhere.

```
github.com/tarhov/gpu-cost-exporter/
  internal/exporter/   ← importable only within this module
  cmd/exporter/        ← can import internal/, external repos cannot
```

### Standard controller repo layout

Kubebuilder scaffolds — and hiring managers expect — this shape:

```
gpu-cost-exporter/
  go.mod
  cmd/
    main.go              ← wiring only: flags, manager setup, Start()
  api/
    v1alpha1/            ← CRD types (Go structs + kubebuilder markers)
  internal/
    controller/          ← reconcilers (the *Reconciler types + Reconcile)
    exporter/             ← business logic (aggregation, cost math)
  config/                ← kustomize manifests (CRDs, RBAC, deployment)
  Makefile
```

`cmd/main.go` stays thin — it constructs the manager and registers controllers. Types live in
`api/`, logic in `internal/controller/` (newer kubebuilder uses `internal/controller/`; older
`controllers/` at root — both are current in the wild). Keeping reconcilers in `internal/` is
deliberate: no one imports your controller as a library.

## Perspectives

**Developer view.** `go.mod`/`go.sum` plus Minimal Version Selection is a deliberately boring,
deterministic algorithm — the opposite of npm/pip's "grab the newest compatible." Two machines
building the same commit always get the same dependency graph without a separate lockfile-freeze
step; the "lockfile" falls out of resolution rather than being a distinct workflow you have to
remember to run.

**Security/supply-chain view.** `go.sum` is a tamper-evidence mechanism, not just a version pin —
every hash must match on every build, making a compromised or yanked upstream release **fail the
build** rather than silently substitute in a different artifact. Combined with verification against
the public module checksum database, this is a real, structural supply-chain control, not an
afterthought bolted onto the module system.

**Team/org view.** `internal/` is a compiler-enforced API boundary — you can promise "nothing
outside this repo depends on `internal/controller`" and mean it, letting you refactor aggressively
without a cross-team deprecation cycle. It converts a social contract ("please don't import this")
into a build-time guarantee.

**Operational/reproducibility view.** A dangling `replace => ../local` directive leaking into a
published/tagged release is a real "works on my machine, breaks for every consumer" incident
class. It's not a theoretical edge case — it's exactly the kind of thing that should be an explicit
CI gate: fail a release build if `go.mod` contains a `replace` pointing at a local path, rather
than discovering it from an angry downstream issue.

## Real-world use cases

- **Google Cloud — "Best practices for dependency management"**
  (<https://cloud.google.com/blog/topics/developers-practitioners/best-practices-dependency-management>)
  — official guidance: pin dependencies for reproducibility but automate monitoring/updates for
  security fixes; use lockfiles with hash/signature verification; separate private/public
  dependency installs to prevent dependency-confusion attacks; scan for vulnerabilities and prune
  unused deps. Extends the `go.sum`/`go mod tidy` coverage above with the *why* — supply-chain
  security, not just build reproducibility. (Guidance current as of the 2020s-era post; treat
  specific tooling references as a dated snapshot.)
- **Monzo — "Migrating our monorepo seamlessly from Dep to Go Modules"**
  (<https://monzo.com/blog/2022/09/29/migrating-our-monorepo-seamlessly-from-dep-to-go-modules>)
  — a real fintech engineering org's account of migrating a large, live production monorepo off
  pre-modules `dep` onto Go modules with zero downtime. What it shows: the module system's version
  and boundary discipline scales to a large, real production codebase, and migrating onto it is a
  tractable, well-understood project even for a monorepo under active development.

## Worked example

Turning the flat exporter script into a proper module with an enforced boundary and a pinned
third-party dep:

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

The result: `github.com/prometheus/client_golang v1.19.1` sits pinned in `go.mod`, hashed in
`go.sum` (and, on first fetch, checked against `sum.golang.org`); the aggregation logic under
`internal/exporter/` is unimportable by any other repo; and this is the exact skeleton the
capstone controller grows into by adding `api/` and `internal/controller/`. A one-line release-CI
check worth adding at this point: `grep -q '^replace' go.mod && exit 1 || true` (or equivalent) so
a stray local `replace` can never ship in a tag.

## Practice

Restructure `gpu-cost-exporter` into a real module:

1. `go mod init github.com/<you>/gpu-cost-exporter`.
2. Move aggregation/cost logic into `internal/exporter/`; keep only wiring in
   `cmd/exporter/main.go`.
3. Add and **pin** one third-party dependency — `github.com/prometheus/client_golang` at an
   explicit version — then `go mod tidy`.
4. Sketch the kubebuilder-style growth path by adding empty `api/v1alpha1/` and
   `internal/controller/` directories with a one-line doc comment each.
5. Add a one-line CI/local check that fails if `go.mod` contains a `replace` directive pointing at
   a local filesystem path (a simple `grep` against `go.mod` is enough for this deliverable).

**Acceptance:** `go build ./...` succeeds; the `internal/` boundary holds (a scratch external
import of the internal package fails to compile — verify by reasoning or a throwaway module);
`go.mod` shows exactly one pinned third-party `require` and `go.sum` has its hashes; the
`replace`-directive check passes on a clean tree and fails when you deliberately add a local
`replace` to test it.

Full deliverable spec: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md).

## Common pitfalls

1. **Believing MVS "always picks the oldest version."** It picks the *maximum* version required by
   any importer in the graph, often a recent one; "minimal" refers to minimizing unnecessary
   version bumps, not defaulting to old versions.
2. **A `replace => ../local` directive silently shipped in a tagged release.** Breaks every
   downstream consumer without that local path; needs an explicit CI check, not a manual
   pre-release memory check.
3. **Confusing `go.work`'s scope.** It's a local, uncommitted development-time overlay; a common
   mistake is expecting it to affect consumers of a published module — it never does, since it's
   never committed or shipped.
4. **Skipping `go mod tidy` after refactors**, leaving stale `require` entries in `go.mod` that
   `go.sum` still carries hashes for — quiet graph bloat that accumulates silently.
5. **Treating `internal/` as merely a naming convention** carried over from Python's `_private`
   habit. It's compiler-enforced; the mistake is under-trusting it — adding defensive comments or
   runtime checks — when the compiler already guarantees the boundary.

## Self-check

- **What does `internal/` actually enforce?**
  **Answer:** Compiler-enforced import visibility: a package under `.../internal/...` can be
  imported **only** by code whose path is rooted at the directory containing that `internal/`.
  Any other module (or a sibling subtree) that tries to import it fails to build. It's a hard
  boundary, not a naming convention, which is what makes everything under `internal/` safe to
  refactor without a cross-team deprecation cycle.
- **When do you need a `replace` directive, and what's the risk if you forget to remove it?**
  **Answer:** When you must build against a module source other than what the version graph
  resolves to — a local checkout you're editing in lockstep, a fork carrying a patch, or forcing a
  transitive dep to a fixed version during debugging. It's build-local and doesn't propagate to
  your consumers automatically, but if a `replace => ../local` is still present when you tag a
  release, every downstream consumer without that exact local path fails or silently resolves
  differently — a real incident class, which is why a release pipeline should check for it
  explicitly rather than relying on someone remembering to remove it.
- **What does Minimal Version Selection actually pick, and what's the common misconception?**
  **Answer:** For each dependency in the build graph, Go selects the *maximum* version required by
  any importer — not the newest version available anywhere, and not (the common misconception) the
  oldest version overall. "Minimal" means Go never bumps a dependency past what something in the
  graph actually requires, which is what makes the build deterministic and reproducible without a
  separate lockfile-freeze step.
- **What problem do workspaces (`go.work`) solve that vendoring doesn't, and what's their scope?**
  **Answer:** They provide a **local, uncommitted overlay** for developing several modules together
  without touching any module's `go.mod`. Vendoring copies frozen dependency snapshots into the
  repo (reproducibility, not active development), and a `replace` mutates a committed file that can
  leak into a release. `go.work` stays out of git and out of every `go.mod`, so you can iterate
  across the controller repo and a shared library simultaneously with nothing to accidentally
  publish — and, because it's never committed, it has zero effect on anyone consuming your
  published module.

## Connections & what's next

This lesson turns the `internal/` boundary you tested against in lesson 05 into a compiler-enforced
contract, and the pinned-dependency discipline here (`go.mod`/`go.sum`, MVS, the checksum database)
is exactly what makes lesson 09's `controller-runtime`/`envtest` dependencies reproducible across
machines. Next: lesson 07, **Stdlib fluency**, assumes you can already add and pin a real
third-party dependency cleanly — it spends its hours on `net/http`, `encoding/json`, `log/slog`,
and `prometheus/client_golang` idioms rather than on module mechanics, because this lesson already
covered how the dependency got there.

## References & further reading

**Primary sources**
- Go Modules Reference — <https://go.dev/ref/mod> — authoritative spec: MVS, semantic import
  versioning, `go.sum`, workspaces. Read as the definitive source when a version-resolution
  question gets subtle.
- Managing dependencies — <https://go.dev/doc/modules/managing-dependencies> — task-oriented guide
  to `go get`, `tidy`, versions, and `replace`. Read for the fastest path to correct day-to-day
  module hygiene.
- Organizing a Go module — <https://go.dev/doc/modules/layout> — official guidance on `cmd/`,
  `internal/`, and package layout. Read for how it maps directly to the controller repo shape
  reviewers expect.

**Real-world engineering blogs**
- Google Cloud — "Best practices for dependency management" —
  <https://cloud.google.com/blog/topics/developers-practitioners/best-practices-dependency-management>
  — what it shows: pinning + hash verification + vulnerability scanning framed explicitly as
  supply-chain security practice, not just build hygiene.
- Monzo — "Migrating our monorepo seamlessly from Dep to Go Modules" —
  <https://monzo.com/blog/2022/09/29/migrating-our-monorepo-seamlessly-from-dep-to-go-modules> —
  what it shows: a real production fintech monorepo migrating onto Go modules with zero downtime —
  proof the module system's discipline scales past a toy repo.

**Deeper dives**
- `go` command module downloading and verification —
  <https://pkg.go.dev/cmd/go#hdr-Module_downloading_and_verification> — the mechanics of how
  `GOSUMDB`/`GOPROXY` verify a fetched module against the checksum database.
- Go module checksum database — <https://sum.golang.org> — the public, append-only transparency
  log `go.sum` verification checks against; canonical reference for how the tamper-evidence
  mechanism actually works underneath `go.sum`.
