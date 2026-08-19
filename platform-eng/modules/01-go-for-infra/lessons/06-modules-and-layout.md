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
sources: 14
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
answers the next question a reviewer or CI pipeline asks: how is this code *packaged*? What is its
module path, what does it pin and why, what is public API versus internal implementation, and does
the repo look like the ones this ecosystem already trusts? It is deliberately **light emphasis** —
mechanical knowledge you apply once and rarely revisit — but it is not optional. A controller repo
*is* a module. This lesson also turns the `internal/` boundary your black-box tests in lesson 05
were respecting into a compiler-enforced contract, and it sets up lesson 07, **Stdlib fluency**,
which assumes you can already add and pin a real third-party dependency like
`prometheus/client_golang` cleanly.

## Why this matters

Three concrete failure modes, each of which this lesson prevents.

**A bad module path is unfixable later.** The module path is the import prefix for every package
inside it, and it is baked into every downstream `go.mod` and every import statement in every file
that ever imports you. Get it wrong — a typo, a personal account you later move off, a missing
`/v2` — and fixing it is a breaking change for everyone.

**A leaked `replace` breaks every consumer.** `replace github.com/foo/bar => ../bar` builds
perfectly on your laptop and fails for everyone who does not have `../bar`. It is a one-line
mistake that ships in a tag and is only discovered by a confused downstream user.

**The checksum database is a real supply-chain control, and it only works if you keep `go.sum`.**
A mismatch is not a warning. Here is what the `go` command actually does, captured from a real run
after tampering with a single hash:

```
$ go build ./...
go: downloading github.com/beorn7/perks v1.0.1
verifying github.com/beorn7/perks@v1.0.1: checksum mismatch
	downloaded: h1:VlbKKnNfV8bJzeqoa4cOKqO6bYr3WgKZxO8Z16+hsOM=
	go.sum:     h1:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=

SECURITY ERROR
This download does NOT match an earlier download recorded in go.sum.
The bits may have been replaced on the origin server, or an attacker may
have intercepted the download attempt.

For more information, see 'go help module-auth'.
```

That is the mechanism that turns a compromised upstream release into a build failure instead of a
silent compromise. Beyond that, the Kubernetes ecosystem is a mesh of interdependent modules with
strict version discipline — `k8s.io/api`, `k8s.io/apimachinery`, `k8s.io/client-go`, and
`sigs.k8s.io/controller-runtime` all move in lockstep, and one mismatched pin produces a wall of
type errors. Fumbling `go.mod` in an interview or a PR reads as junior; being able to explain what
MVS picked and why reads as senior.

## What's new here (calibration)

You already know the shape of a dependency manifest and a lockfile from Python
(`pyproject.toml` / `requirements.txt` / a hash-pinned lock). We skip re-explaining what a lockfile
is for or why pinning matters in the abstract. What is genuinely new, calibrated to a staff-level
track:

- **Minimal Version Selection as an algorithm you can execute by hand** — over a real version graph
  from a real `go mod graph`, not a slogan — plus the specific misconception ("it always picks the
  oldest") that is worth being able to correct on the spot.
- **Why there is no lockfile at all.** This is the structural difference from `pip`/`npm`, and it
  follows from MVS being deterministic rather than from Go having a different tool.
- **Module graph pruning (Go 1.17+)** — why modern `go.mod` files list far more `// indirect`
  entries than old ones, and why that is a performance feature rather than clutter.
- **`go.sum` as tamper evidence backed by a public transparency log**, not merely a version pin —
  the security reasoning, the Merkle-tree mechanism, and the exact escape hatches (`GOPRIVATE`,
  `GONOSUMDB`, `GOSUMDB=off`) and when using them is legitimate.
- **`internal/` as a compiler-enforced org boundary** with an exactly-specified rule — not Python's
  honour-system `_private`.
- **The operational failure mode of a leaked `replace => ../local`**, and the one-line CI check that
  makes it impossible.

## Core concepts

### Modules, packages, and paths

Three nested concepts, and confusing them is the root of most module questions.

- A **package** is a directory of `.go` files sharing a `package` clause. It is the unit of
  compilation and of import.
- A **module** is a tree of packages with a `go.mod` at its root. It is the unit of *versioning*
  and *distribution*.
- The **module path** declared in `go.mod` is the import prefix for every package in the module.

```
 FROM REPOSITORY TO IMPORT PATH
 ═══════════════════════════════════════════════════════════════════════

  github.com/tarhov/gpu-cost-exporter          ← the repo, and the module path
  └── go.mod        module github.com/tarhov/gpu-cost-exporter
      ├── cmd/
      │   └── exporter/  main.go     package main
      ├── internal/
      │   └── exporter/  collect.go  package exporter
      │                              ▲
      │                              └─ imported as
      │                                 "github.com/tarhov/gpu-cost-exporter/internal/exporter"
      │                                  └────────── module path ─────────┘└─ dir path ─┘
      └── api/
          └── v1alpha1/  types.go    package v1alpha1
                                     imported as ".../api/v1alpha1"

  RESOLVING AN IMPORT — how `go` finds the module for a package path
  ──────────────────────────────────────────────────────────────────
  import "golang.org/x/net/html"
     candidates, longest prefix wins:
       golang.org/x/net/html   ← is there a module at this path?  no
       golang.org/x/net        ← yes. module found; package is ./html inside it
       golang.org/x            ← (not consulted; longer match already won)
```

The module path is a *URL-shaped identifier*, and the `go` command really will fetch it: given
`example.com/foo/bar`, it requests `https://example.com/foo/bar?go-get=1` and reads a
`<meta name="go-import">` tag to find the repository. `github.com/...` short-circuits that with a
built-in rule. This is why the path must be something you control: it is simultaneously a name and
an address.

Two practical rules that follow:

- **Use the repo URL as the module path.** `github.com/<you>/gpu-cost-exporter`. Not
  `gpu-cost-exporter`, not `main`. A bare path works locally and breaks the moment anyone imports it.
- **The path is public API.** Renaming it is a breaking change for every consumer, so pick it once.

### `go.mod`: every directive that matters

Here is a real `go.mod`, produced by `go mod init` + `go get` + `go mod tidy` for the exporter:

```
module github.com/tarhov/gpu-cost-exporter

go 1.24.7

require github.com/prometheus/client_golang v1.20.5

require (
	github.com/beorn7/perks v1.0.1 // indirect
	github.com/cespare/xxhash/v2 v2.3.0 // indirect
	github.com/klauspost/compress v1.17.9 // indirect
	github.com/munnerz/goautoneg v0.0.0-20191010083416-a7dc8b61c822 // indirect
	github.com/prometheus/client_model v0.6.1 // indirect
	github.com/prometheus/common v0.55.0 // indirect
	github.com/prometheus/procfs v0.15.1 // indirect
	golang.org/x/sys v0.22.0 // indirect
	google.golang.org/protobuf v1.34.2 // indirect
)
```

Read every part of it:

| Directive | Since | What it does |
|---|---|---|
| `module` | 1.11 | Declares the module path / import prefix. Exactly one per file. |
| `go` | 1.11 | The **minimum** Go version required to build this module. Since Go 1.21 it is a hard requirement, not advice: older toolchains refuse to build a module declaring a newer version. It also gates language features and `go`-command behaviour (see below). If absent, `go 1.16` is assumed. |
| `toolchain` | 1.21 | A *suggested* toolchain, e.g. `toolchain go1.24.0`. Only takes effect when this is the main module and the local default toolchain is older — in which case the `go` command downloads and re-execs the named toolchain. |
| `godebug` | 1.23 | Pins a [GODEBUG setting](https://go.dev/doc/godebug) for the whole main module, e.g. `godebug containermaxprocs=0`. Equivalent to a `//go:debug` line in every main package. |
| `require` | 1.11 | A **minimum** version of a dependency. `// indirect` marks a module not imported directly by this module's own packages. |
| `exclude` | 1.11 | Removes a specific version from the graph; requirements on it are redirected to the *next higher* version. Rare, and only honoured in the main module. |
| `replace` | 1.11 | Swaps a module's source. Only the main module's (and `go.work`'s) `replace` directives apply. See below. |
| `retract` | 1.16 | *You* declaring, in a later version of your own module, that an earlier version should not be used. The mechanism for un-publishing a bad release without deleting the tag. |
| `tool` | 1.24 | Records an executable dependency (`tool golang.org/x/tools/cmd/stringer`) so `go tool stringer` works. Replaces the old `tools.go` blank-import hack. Added with `go get -tool`. |
| `ignore` | 1.25 | Excludes directory paths from package-pattern matching, e.g. `ignore ./node_modules`. |

Two subtleties on the `go` directive that trip people up:

- **It is not the toolchain you are running.** `go 1.24.7` in the file above is a floor. Building
  with Go 1.26 is fine; building with Go 1.23 is refused.
- **It changes the `go` command's own behaviour**, not just the compiler's. At `go 1.14`+, a
  `vendor/` directory is used automatically. At `go 1.16`+, the `all` pattern narrows. At
  `go 1.17`+, module graph pruning turns on and every transitively-imported module gets an explicit
  `require`. At `go 1.21`+, the `go` line must be ≥ the `go` line of every dependency.

The core commands:

```bash
go mod init github.com/tarhov/gpu-cost-exporter   # create go.mod
go get github.com/prometheus/client_golang@v1.20.5 # add/pin a requirement
go get -u ./...                                    # upgrade deps of packages in the main module
go mod tidy                                        # add missing, drop unused, sync go.sum
go mod why -m golang.org/x/sys                     # WHY is this in my graph?
go list -m all                                     # the build list (the "lockfile" view)
go mod graph                                       # every edge of the requirement graph
go mod verify                                      # re-hash the module cache against what was downloaded
```

`go mod why` is the underused one. Real output from the exporter:

```
$ go mod why -m golang.org/x/sys
# golang.org/x/sys
github.com/tarhov/gpu-cost-exporter/internal/exporter
github.com/prometheus/client_golang/prometheus
golang.org/x/sys/windows
```

That is a shortest path through the *import* graph: your package imports `client_golang/prometheus`,
which imports `x/sys/windows`. When a reviewer asks "why does a Linux exporter depend on
`x/sys/windows`," this is the two-second answer — the Prometheus process collector imports it
behind build tags, and module requirements ignore build tags.

**Skipping `go mod tidy` after a refactor** leaves stale `require` entries that `go.sum` still
carries hashes for. It is quiet graph bloat: more modules to download, more to audit, more surface
for a vulnerability scanner to flag. Run it, and commit both files.

### Minimal Version Selection, executed by hand

This is the algorithm. It is short enough to state completely.

> **MVS.** Build a directed graph whose vertices are module versions and whose edges are `require`
> directives. Start at the main module (a vertex with no version). Traverse the graph, loading each
> reached module's `go.mod`. For each module path, keep the **highest version required by anything
> reached in the traversal**. That set of highest versions is the **build list**.

That is it. Three properties fall out immediately:

- **It never picks a version nothing asked for.** If `v1.9.0` exists on the proxy but the highest
  requirement in your graph is `v1.5.0`, you build against `v1.5.0`. Publishing a new version of a
  dependency cannot change your build.
- **It is deterministic and reproducible with no lockfile.** Because the answer depends only on the
  `go.mod` files reachable from your commit, two machines building the same commit compute the same
  build list. `go.sum` records *hashes*, not version choices — it is tamper evidence, not a lock.
- **"Minimal" is about not over-upgrading, not about picking old versions.** MVS takes the *maximum*
  of the requirements; the minimality is that it never goes higher than something in the graph
  actually asked for.

Now run it on a real graph. This is captured output from the exporter — `go mod graph` prints one
edge per line, `path@version path@version`, with the main module unversioned:

```
$ go mod graph
github.com/tarhov/gpu-cost-exporter github.com/prometheus/client_golang@v1.20.5
github.com/tarhov/gpu-cost-exporter golang.org/x/sys@v0.22.0
...
github.com/prometheus/client_golang@v1.20.5 github.com/cespare/xxhash/v2@v2.3.0
github.com/prometheus/client_golang@v1.20.5 github.com/prometheus/common@v0.55.0
github.com/prometheus/client_golang@v1.20.5 github.com/prometheus/procfs@v0.15.1
github.com/prometheus/client_golang@v1.20.5 golang.org/x/sys@v0.22.0
github.com/prometheus/client_golang@v1.20.5 google.golang.org/protobuf@v1.34.2
github.com/prometheus/client_model@v0.6.1 google.golang.org/protobuf@v1.33.0
github.com/prometheus/common@v0.55.0 github.com/cespare/xxhash/v2@v2.2.0
github.com/prometheus/common@v0.55.0 github.com/prometheus/client_golang@v1.19.1
github.com/prometheus/common@v0.55.0 golang.org/x/sys@v0.21.0
github.com/prometheus/procfs@v0.15.1 golang.org/x/sys@v0.20.0
github.com/prometheus/procfs@v0.15.1 github.com/google/go-cmp@v0.6.0
google.golang.org/protobuf@v1.34.2 github.com/google/go-cmp@v0.5.5
```

Trace `golang.org/x/sys` through it:

```
 MVS ON golang.org/x/sys — A REAL THREE-WAY RESOLUTION
 ═══════════════════════════════════════════════════════════════════════

  main (gpu-cost-exporter)
    │
    ├─ requires ──▶ prometheus/client_golang@v1.20.5
    │                   ├─ requires ──▶ golang.org/x/sys@v0.22.0   ◀── candidate
    │                   ├─ requires ──▶ prometheus/common@v0.55.0
    │                   │                   └─ requires ──▶ x/sys@v0.21.0   ◀── candidate
    │                   └─ requires ──▶ prometheus/procfs@v0.15.1
    │                                       └─ requires ──▶ x/sys@v0.20.0   ◀── candidate
    │
    └─ requires ──▶ golang.org/x/sys@v0.22.0        (the // indirect line in go.mod)

  MAX( v0.22.0, v0.21.0, v0.20.0, v0.22.0 )  =  v0.22.0
                                                 ▲
  VERIFY:                                        │
  $ go list -m golang.org/x/sys                  │
  golang.org/x/sys v0.22.0  ─────────────────────┘

  NOTE WHAT DID *NOT* HAPPEN:
    • x/sys v0.36.0 exists upstream. Nothing in the graph requires it,
      so MVS does not select it. Publishing it cannot change this build.
    • procfs asked for v0.20.0 and got v0.22.0. MVS gives every requirer
      AT LEAST what it asked for — a require is a floor, never a ceiling.
```

Two more resolutions from the same real graph, worth walking because they show different shapes:

| Module | Requirements found in the graph | Selected | Why |
|---|---|---|---|
| `google.golang.org/protobuf` | `client_model@v0.6.1` → v1.33.0; `client_golang@v1.20.5` and `common@v0.55.0` → v1.34.2 | **v1.34.2** | Max wins. `client_model` gets a newer protobuf than it asked for, which is safe because v1 promises compatibility. |
| `github.com/google/go-cmp` | `protobuf@v1.34.2` → v0.5.5; `client_golang`, `common`, `procfs` → v0.6.0 | **v0.6.0** | Max wins again — and note the winner came from a *shallower* node. Depth is irrelevant; only the maximum matters. |
| `github.com/cespare/xxhash/v2` | `common@v0.55.0` → v2.2.0; `client_golang@v1.20.5` → v2.3.0 | **v2.3.0** | Max. Also note the `/v2` suffix: this is a v2+ module, so the major version is part of the path. |

And note this edge, which surprises people:

```
github.com/prometheus/common@v0.55.0 github.com/prometheus/client_golang@v1.19.1
```

`common` requires `client_golang`, which requires `common`. **The module graph has cycles and that
is fine** — MVS is a traversal keeping a running maximum, not a topological sort. Our main module
requires `client_golang@v1.20.5`, which is higher than the v1.19.1 that `common` asks for, so
v1.20.5 is selected.

**Upgrades and downgrades, precisely.** `go get pkg@vX` does not "just edit go.mod." For an
**upgrade**, the `go` command adds edges from the visited versions to the upgraded ones and re-runs
MVS; requirements that would not otherwise be selected are written back with `// indirect`
comments. For a **downgrade**, it *removes* the too-new versions from the graph, and also removes
any module version that required them — because such a module may not work with the older
dependency — then reruns MVS.

Here is that observed, real:

```
$ go get github.com/prometheus/client_golang@v1.19.1
go: downgraded github.com/prometheus/client_golang v1.20.5 => v1.19.1
$ go mod tidy
```

`go.mod` afterwards has **eight** indirect requirements instead of nine — `github.com/klauspost/compress`
is gone entirely, because only `client_golang` v1.20.x pulled it in (for zstd response encoding).
Re-running `go get github.com/prometheus/client_golang@v1.20.5` brings it back. That is MVS: the
build list is a pure function of the graph, so a version change adds and removes transitive
dependencies deterministically rather than accumulating them.

**The `@none` and `@latest` queries** round out the vocabulary: `go get example.com/m@none` removes
a module from the graph entirely (a downgrade to nothing), and `@latest`, `@upgrade`, `@patch`, a
branch name, or a commit hash all resolve to a concrete version before MVS runs.

### Module graph pruning, and why `go.mod` got long

If you have seen an old `go.mod` with three requires and a modern one with forty, this is why.

At `go 1.16` and below, `go.mod` listed only direct dependencies plus whatever indirect versions
MVS would not otherwise pick. To be sure it had every indirect requirement, the `go` command had to
load the **full transitive closure** of every dependency's `go.mod` — potentially thousands of
files, over the network, before it could build anything.

At **`go 1.17` and above**, the trade flips. `go.mod` gains an explicit `require` for *every module
providing a package transitively imported by a package or test in the main module* — which is why
there is a second, longer `require` block full of `// indirect` lines. In exchange, the module
graph is **pruned**: for a dependency that itself declares `go 1.17` or higher, only its
*immediate* requirements are loaded, not its full transitive closure. Those pruned-out modules
cannot affect your packages' behaviour, because if they could, they would be needed to build
something you import, and would therefore be in your own `go.mod`.

The companion optimisation is **lazy module loading**: the `go` command first tries to resolve all
imports using only the main module's `go.mod`, and only loads the rest of the graph if something is
missing. In a large repo this is the difference between an instant `go build` and a multi-second
one.

One consequence to know for CI: `go mod tidy` records `go.sum` entries needed by the Go version
**one below** your `go` directive, so a `go 1.17` module carries the checksums Go 1.16 would need.
The `-compat` flag overrides that. If a CI job on an older toolchain fails with "missing go.sum
entry," this is usually why.

### `go.sum` and the checksum database

`go.sum` is not a lockfile. MVS already made the build reproducible; `go.sum` makes it
**tamper-evident**.

Its format is exactly three space-separated fields — and note there are normally **two lines per
module version**:

```
github.com/beorn7/perks v1.0.1 h1:VlbKKnNfV8bJzeqoa4cOKqO6bYr3WgKZxO8Z16+hsOM=
github.com/beorn7/perks v1.0.1/go.mod h1:G2ZrVWU2WbWT9wwq4/hrbKbnv/1ERSJQ0ibhJ6rlkpw=
github.com/alecthomas/kingpin/v2 v2.4.0/go.mod h1:0gyi0zQnjuFk8xrkNKamJoyUo382HRL7ATRpFZCw6tE=
```

| Field | Meaning |
|---|---|
| `github.com/beorn7/perks` | Module path. |
| `v1.0.1` | The version. A `/go.mod` suffix means the hash covers **only that module's `go.mod` file**; without it, the hash covers the module's whole `.zip`. |
| `h1:...` | Algorithm name and base64 hash. `h1` is SHA-256 over a deterministic listing of file names and contents (`x/mod/sumdb/dirhash`), so it is unaffected by compression, file order, or timestamps. `h2` etc. are reserved for a future algorithm change. |

Why two lines: MVS needs to read the `go.mod` files of module versions it never actually builds
against (to compute the maximum), so those get a `/go.mod`-only entry. `kingpin/v2` above has only
the `/go.mod` line — it was traversed but its source was never downloaded.

**The verification flow**, end to end:

```
 HOW A DEPENDENCY GETS TRUSTED
 ═══════════════════════════════════════════════════════════════════════

  go build
     │
     │ need github.com/beorn7/perks@v1.0.1
     ▼
  ┌──────────────────────┐  hit   ┌──────────────────────────────────┐
  │  module cache        │───────▶│ verify against MAIN MODULE go.sum│
  │  $GOMODCACHE         │        │  mismatch → SECURITY ERROR, stop │
  └──────────┬───────────┘        └──────────────────────────────────┘
             │ miss
             ▼
  ┌──────────────────────┐
  │  GOPROXY             │  default: "https://proxy.golang.org,direct"
  │  fetch .mod and .zip │  ("direct" = clone the VCS repo yourself)
  └──────────┬───────────┘
             │  compute h1: hash of what arrived
             ▼
      ┌──────────────────────────────┐
      │ is the hash already in       │  yes ──▶ compare. mismatch → SECURITY ERROR
      │ the main module's go.sum?    │
      └──────────┬───────────────────┘
                 │ no
                 ▼
      ┌──────────────────────────────────────────────┐
      │ GOSUMDB (default sum.golang.org)              │
      │  a TRANSPARENT LOG — a Merkle tree of go.sum  │
      │  lines, run by Google on Trillian.            │
      │  Ask for the record; verify the inclusion     │
      │  proof against the signed tree head.          │
      └──────────┬───────────────────────────────────┘
                 │ verified
                 ▼
       write the line into go.sum, store in the cache, build.

  BYPASSES (each narrows the check, so use the narrowest one):
    GOPRIVATE=github.com/mycorp/*     sets GONOPROXY *and* GONOSUMDB
    GONOSUMDB=github.com/mycorp/*     skip the sumdb for these paths only
    GONOPROXY=github.com/mycorp/*     fetch these direct from VCS, not the proxy
    GOSUMDB=off                       disable the checksum database entirely  ← blunt
    GOFLAGS=-mod=mod                  allow go.mod/go.sum to be edited by a build
```

The property that makes the checksum database more than "a hash server" is that it is a **Merkle
tree** whose signed tree heads clients gossip about. A server that served you a different hash than
it served everyone else would have to produce an inconsistent tree, which independent auditors can
detect. That is what "transparent log" buys over "a database Google promises is correct."

Two things follow for a real controller repo:

- **Commit `go.sum`.** A repo without it re-verifies against the sumdb on every fresh machine, which
  is *safer than nothing* but means you have no record of what you previously accepted.
- **Set `GOFLAGS=-mod=readonly` in CI** (it is the default since Go 1.16, but be explicit) so a
  build that would need to modify `go.mod` fails instead of silently changing your dependency graph.
  `go mod tidy` is a human action, not a build step.

For a private module, the right setting is `GOPRIVATE=github.com/mycorp/*` — narrow, and it
correctly turns off *both* the public proxy and the public checksum database for exactly those
paths, without weakening verification for anything else. `GOSUMDB=off` globally is the answer to a
different, larger question and should be rare.

### Semantic import versioning

Go encodes the **major** version in the import path from v2 onward. This implements the *import
compatibility rule*, stated in the module reference:

> If an old package and a new package have the same import path, the new package must be backwards
> compatible with the old package.

Since a new major version is by definition not backwards compatible, it must have a different import
path. So:

```go
import "github.com/foo/bar"      // module github.com/foo/bar      — v0.x or v1.x
import "github.com/foo/bar/v2"   // module github.com/foo/bar/v2   — v2.x
```

Both the `module` line in the dependency's `go.mod` and every import of it carry the suffix. Rules:

- **No suffix at v0 or v1.** v0 has no compatibility promise, and v1 is the commitment to
  compatibility rather than a break from v0.
- **`gopkg.in/` is a special case**: it always carries a version, with a dot instead of a slash —
  `gopkg.in/yaml.v2`, `gopkg.in/yaml.v3`.
- **`+incompatible`** marks a v2-or-higher tag published *before* the repo adopted modules, e.g.
  `v3.2.0+incompatible`. The `go` command tolerates it for legacy repos; you should never create one.

The payoff is that **two major versions can coexist in a single build**, because they are different
module paths with different package paths. That is not theoretical — the real build list for the
exporter contains both:

```
$ go list -m all | grep yaml
gopkg.in/yaml.v2 v2.4.0
gopkg.in/yaml.v3 v3.0.1
```

`prometheus/common` uses yaml.v2 while `stretchr/testify` uses yaml.v3, and both are present, both
compile, and neither knows about the other. In a dependency manager without this rule, that is an
unresolvable conflict; here it is a non-event.

The corresponding obligation when *you* publish a v2: you must move your own module to
`github.com/you/thing/v2` (edit the `module` line, either at the repo root or in a `v2/`
subdirectory) *and* update every internal import. Forgetting the internal imports is the classic
error — your v2 module silently compiles against your own v1 packages.

### `replace`: what it is for, and the incident it causes

A `replace` directive swaps a module's contents for something else at build time:

```
replace github.com/foo/bar => ../bar                          // a local checkout you are editing
replace github.com/foo/bar => github.com/me/bar v1.2.0-fork   // your fork, carrying a patch
replace github.com/foo/bar v1.2.3 => github.com/me/bar v1.2.4 // only that one version
```

Semantics worth knowing exactly:

- **Only the main module's `replace` directives apply** (plus a workspace's). A dependency's
  `replace` lines are ignored entirely. This is deliberate: it means a transitive dependency cannot
  redirect your build.
- **`replace` alone does not add anything to the graph.** You still need a `require` for the module
  being replaced, from somewhere. A `replace` for a module nothing requires is a no-op.
- **With a local path, the version on the right is omitted**, and the target directory must contain
  its own `go.mod` whose `module` line matches the path being replaced.
- Replacement changes the graph, because the replacement may have different requirements than the
  version it replaced. MVS then runs over the modified graph.

Legitimate uses: developing two modules in lockstep before publishing; testing a patch against an
upstream bug; forcing a transitive dependency to a fixed fork.

**The incident.** A `replace => ../local` that survives into a tagged release builds fine for you
(the path exists on your machine) and fails for every consumer who does not have that directory. It
is a "works on my machine" bug with a blast radius of your entire downstream. Consumers do not even
get a clear error — they get whatever their own resolution produces, or a confusing "cannot find
module" pointing at a path on someone else's laptop.

The fix is not vigilance, it is a gate. One line in the release job:

```bash
# Fail the release if go.mod carries a filesystem replace.
if go mod edit -json | jq -e '.Replace[]? | select(.New.Path | test("^\\.{1,2}/"))' >/dev/null; then
  echo "ERROR: go.mod contains a local-path replace directive; refusing to release" >&2
  exit 1
fi
```

`go mod edit -json` prints the parsed `go.mod` as structured JSON, which is far more robust than
grepping — it survives block syntax, comments, and formatting. If `jq` is unavailable, a
`grep -E '^[[:space:]]*replace .*=> +\.{1,2}/'` against `go mod edit -print` is an acceptable
approximation.

The *ecosystem-level* reason this trap exists at all is that `replace` predates workspaces. Which
brings us to the right tool.

### Workspaces (`go.work`)

A workspace lets you develop several modules together **without editing any of their `go.mod`
files**.

```bash
go work init ./gpu-cost-exporter ./gpu-shared-lib
```

produces:

```
go 1.24.7

use (
	./gpu-cost-exporter
	./gpu-shared-lib
)
```

Mechanically: the modules listed under `use` all become **main modules** for MVS. Any import of
`github.com/tarhov/gpu-shared-lib` resolves to the local directory, not the published version, for
every `go build` and `go test` run inside the workspace.

Scope rules, which are the part people get wrong:

- The `go` command finds the workspace by checking `GOWORK` first: `off` disables workspace mode
  entirely; an empty value means "search the current directory and its parents for `go.work`"; a
  path ending in `.work` names one explicitly. `go env GOWORK` prints which file is in effect —
  the first thing to run when a build resolves a dependency you did not expect.
- `go.work` supports `use`, `replace`, `go`, `toolchain`, and `godebug`. A `replace` in `go.work`
  **overrides** conflicting `replace` directives in the member modules' `go.mod` files.
- Some commands are deliberately single-module even inside a workspace: `go mod init`,
  `go mod tidy`, `go mod vendor`, `go mod why`, `go mod edit`, and `go get`.
- `go work sync` pushes the workspace's resolved build list back into each member's `go.mod` when
  you are ready to make it real.

```
 THREE WAYS TO POINT A BUILD AT DIFFERENT SOURCE
 ═══════════════════════════════════════════════════════════════════════

                 replace              go.work              vendor/
                 ───────              ───────              ───────
 lives in        go.mod               go.work              vendor/ tree
                 (COMMITTED)          (gitignored)         (COMMITTED)

 scope           this module's        every module         this module's
                 builds, and it       under the            builds
                 SHIPS in the tag     workspace root

 affects
 consumers?      no (ignored in       never — it is        no
                 dependencies)        never published

 use it for      a fork with a        editing 2+ modules   air-gapped or
                 patch; pinning a     in lockstep before   proxy-free builds;
                 transitive dep       publishing either    license audit

 failure mode    leaks into a         a forgotten go.work  stale vendor/ that
                 release and breaks   silently overrides   disagrees with
                 every consumer       the pinned version   go.mod → build error
```

The one-line summary: **`replace` is a property of your module; `go.work` is a property of your
laptop.** Keep `go.work` and `go.work.sum` in `.gitignore` and the leaked-replace incident class
disappears, because you stop needing `replace` for local development at all.

### Vendoring

`go mod vendor` copies the source of every package needed to build and test the main module into a
`vendor/` directory, plus a `vendor/modules.txt` manifest recording which module version each
package came from.

Behaviour worth knowing:

- If `vendor/` exists and the `go` directive is **1.14 or higher**, vendoring is used
  **automatically** — no `-mod=vendor` flag needed. Override with `-mod=mod` or `-mod=readonly`.
- The `go` command checks `vendor/modules.txt` against `go.mod` on every build and **errors if they
  disagree**. So a dependency bump without re-running `go mod vendor` fails loudly rather than
  building stale code.
- Vendoring changes *builds*, not *module commands*: `go mod tidy` and `go mod download` still hit
  the network and the module cache.
- At `go 1.17`+, `go mod vendor` omits the dependencies' own `go.mod`/`go.sum` files and records
  each dependency's `go` version in `modules.txt`.

When it is the right call, in order of how often you will actually see it: **air-gapped builds**
where no proxy is reachable; **legal/licence review** that wants the literal dependency source in
the repo under review; and **removing the module proxy from your build's critical path** so an
upstream outage cannot break a release. The costs are repo size and a large, noisy diff on every
dependency bump. For a controller repo that builds in normal CI, skip it — `GOPROXY` plus a warm
module cache gets you the same hermeticity with none of the diff noise.

### `internal/`: a compiler-enforced boundary

The rule, stated exactly (from `go help packages`):

> Code in or below a directory named `internal` is importable only by code that shares the same
> import path above the `internal` directory.

Note "in or below," and note that the boundary is the **parent of the `internal` directory**, not
the module root. `internal/` can appear at any depth, and each occurrence creates its own boundary.

```
 THE internal/ VISIBILITY RULE
 ═══════════════════════════════════════════════════════════════════════

  module example.com/m

  example.com/m/
   ├── crash/
   │    └── bang/           package bang
   └── foo/                 package foo          ◀── the boundary is HERE:
        ├── bar/            package bar              "example.com/m/foo"
        ├── quux/           package quux
        └── internal/
             └── baz/       package baz

  MAY import example.com/m/foo/internal/baz:
     ✓ example.com/m/foo          (the parent itself)
     ✓ example.com/m/foo/bar      (below the parent)
     ✓ example.com/m/foo/quux

  MAY NOT:
     ✗ example.com/m/crash/bang   (same module, wrong subtree)
     ✗ example.com/m              (ABOVE the parent — this surprises people)
     ✗ any other module, ever

  A CONTROLLER REPO PUTS internal/ AT THE ROOT, so the boundary is the
  module itself:

  github.com/tarhov/gpu-cost-exporter/
   ├── internal/exporter/     ◀── importable only within this module
   └── cmd/exporter/          ◀── may import internal/; nobody outside can
```

And the enforcement is a build error, captured from a real attempt by a second module that
`replace`s the exporter to a local path:

```
$ go build ./...
package example.com/outsider
	main.go:3:8: use of internal package github.com/tarhov/gpu-cost-exporter/internal/exporter not allowed
```

This is a categorically different guarantee from Python's `_private` convention, where "private"
is documentation that a determined caller ignores with no consequence. Here the import does not
compile. What that buys a team is the ability to promise "nothing outside this repo depends on
`internal/controller`" and *mean* it — which means you can rename types, change signatures, and
restructure packages without a cross-team deprecation cycle, because there is provably no
cross-team caller.

The common mistake is not misusing `internal/`; it is **under-trusting it** — adding "do not import
this from outside" doc comments, or runtime checks, on top of a boundary the compiler already
guarantees absolutely. Spend that comment budget on why the code does what it does.

(`vendor/` has an analogous rule in GOPATH mode, which is where the mechanism came from; in module
mode only the main module's `vendor/` is consulted.)

### Standard layout

Go's official layout guidance
([`go.dev/doc/modules/layout`](https://go.dev/doc/modules/layout)) is explicit about servers, which
is what you are building: server projects usually export nothing, so **keep the implementation
under `internal/` and put every binary under `cmd/`**. Kubebuilder scaffolds exactly that shape,
plus the CRD-specific directories:

```
gpu-cost-exporter/
  go.mod
  go.sum
  Makefile
  cmd/
    main.go                ← wiring ONLY: flags, manager setup, Start()
  api/
    v1alpha1/              ← CRD types: Go structs + kubebuilder markers.
                             PUBLIC on purpose — other repos import these to
                             build clients against your CRD.
  internal/
    controller/            ← reconcilers: the *Reconciler types and Reconcile
    exporter/              ← business logic: aggregation, cost math
  config/                  ← kustomize manifests: CRDs, RBAC, manager Deployment
  test/
    e2e/                   ← tests that need a real cluster
```

The design decisions encoded in that tree, each of which a reviewer will read:

- **`cmd/main.go` is thin.** It constructs the manager, registers controllers, and calls `Start`.
  Anything else in `main` is untestable by construction — it is the one package your test suite
  cannot easily exercise, so it should contain nothing worth testing. This is also why `main`
  showing 0% coverage in lesson 05's report was the correct answer rather than a gap.
- **`api/` is deliberately *not* internal.** Your CRD's Go types are the API other people generate
  clients from. Making them importable is the point.
- **`internal/controller/` is deliberately internal.** Nobody imports your reconciler as a library;
  keeping it internal is what lets you restructure it freely.
- **Newer kubebuilder scaffolds `internal/controller/`; older versions put `controllers/` at the
  repo root.** Both are common in the wild — recognise both when reading source in lesson 08.

Two things *not* to do. Do not create a top-level `src/` — that is GOPATH-era muscle memory and the
module system has no use for it. And be wary of the widely-copied `golang-standards/project-layout`
repo: it is not official, it is not what the Go team recommends, and its `pkg/` convention adds a
directory level that buys nothing now that `internal/` exists. Follow the official layout doc and
whatever `kubebuilder init` generates.

## Perspectives

**Developer view.** MVS is deliberately boring, and boring is the feature. There is no resolver
backtracking, no SAT solver, no "resolution took 40 seconds" like npm, and no separate lockfile
step to remember, because the build list is a pure function of the `go.mod` files reachable from
your commit. The cost of that determinism is that a dependency's `require` is a *floor* you cannot
lower except by explicit `replace` or `exclude` — Go trades expressiveness for predictability, and
for infrastructure code that is the right trade.

**Security/supply-chain view.** `go.sum` plus `sum.golang.org` is one of the strongest
supply-chain stories in any mainstream language ecosystem, and it is on by default rather than
being an opt-in tool you bolt on. The transparency-log design is the part that matters: it is not
"trust Google's hash server," it is "Google's hash server cannot lie to you selectively without
producing a Merkle tree that independent auditors can prove is inconsistent." The failure mode is
loud (`SECURITY ERROR`, build stops) rather than silent, which is the correct default for a control
whose job is to detect substitution.

**Team/org view.** `internal/` converts a social contract into a build-time guarantee, and that
changes how fast a team can move. The expensive part of refactoring in a large org is not the code
change, it is discovering and coordinating with unknown callers. `internal/` makes the set of
possible callers finite and enumerable — it is a *grep away*, not a Slack thread away.

**Operational/reproducibility view.** Every mechanism here has an operational tell. A leaked
`replace` shows up as a downstream build failure. A missing `go mod tidy` shows up as a
vulnerability scanner flagging a module nothing imports. A stale `vendor/` shows up as a `go`
command error comparing `modules.txt` to `go.mod`. A forgotten `go.work` shows up as a build that
mysteriously uses your uncommitted local changes. Learn each tell once, and module problems stop
being mysterious.

## Real-world use cases

- **Go 1.17's module graph pruning (2021).** Before it, resolving dependencies meant loading the
  full transitive closure of every dependency's `go.mod` — for a Kubernetes-adjacent repo, thousands
  of files over the network before a single package compiled. Go 1.17 traded a longer `go.mod` (an
  explicit `require` for every transitively-imported module) for a pruned graph and lazy loading:
  the `go` command now resolves imports from the main module's `go.mod` alone and only loads the
  rest on demand. *What it shows:* the long `// indirect` block in a modern `go.mod` is not clutter
  from a sloppy `go mod tidy` — it is the data structure that makes the build fast, and deleting
  those lines by hand makes things worse, not tidier.

- **`+incompatible` and the v2 migration debt.** A large number of Go projects tagged v2, v3, or
  higher before modules existed and without adding a `/vN` path suffix. The `go` command tolerates
  these as `v3.2.0+incompatible`, treating them as a legacy dialect. *What it shows:* the cost of
  retrofitting a versioning rule onto an ecosystem, and why you should never create a new
  `+incompatible` version yourself — if you are going to v2, take the path suffix, including in
  your own internal imports.

- **The Kubernetes module mesh.** `k8s.io/api`, `k8s.io/apimachinery`, `k8s.io/client-go`, and
  `sigs.k8s.io/controller-runtime` release in lockstep, and controller-runtime's `go.mod` pins a
  matched set. Because MVS takes the maximum, a stray `go get k8s.io/client-go@latest` in a repo
  pinned to an older controller-runtime raises client-go above what controller-runtime was built
  against and produces a wall of type errors — the classic symptom being a mismatched
  `k8s.io/apimachinery` type in a function signature. *What it shows:* MVS's "max wins" rule is
  exactly right for compatible dependencies and exactly wrong for a family of modules that must
  move together, which is why the fix is to bump `controller-runtime` and re-tidy rather than to
  pin the leaf.

- **[Google Cloud — "Best practices for dependency management"](https://cloud.google.com/blog/topics/developers-practitioners/best-practices-dependency-management)** —
  vendor-neutral guidance that lines up with everything above: pin dependencies for
  reproducibility but automate monitoring so security fixes are not blocked by the pin; use
  lockfiles with hash or signature verification; keep private and public dependency resolution
  separate to prevent dependency-confusion attacks; scan for vulnerabilities and prune unused
  dependencies. In Go those map onto `go.sum` + `GOSUMDB`, `GOPRIVATE`/`GONOPROXY`,
  `govulncheck`, and `go mod tidy` respectively. *What it shows:* the *why* behind the mechanics —
  supply-chain security, not just build hygiene. *(Treat tooling references as a dated snapshot.)*

- **[Monzo — "Migrating our monorepo seamlessly from Dep to Go Modules"](https://monzo.com/blog/2022/09/29/migrating-our-monorepo-seamlessly-from-dep-to-go-modules)** —
  a fintech engineering org moving a large, live production monorepo off pre-modules `dep` onto Go
  modules without downtime. *What it shows:* the module system's version and boundary discipline
  scales past a toy repo, and migrating onto it is a tractable, well-understood project even for a
  codebase under continuous development.

## Worked example

Turn the flat exporter script into a proper module with an enforced boundary and a pinned
dependency. Every command and output below is real.

```bash
# 1. Initialize the module. The path IS the import prefix — use the repo URL.
$ go mod init github.com/tarhov/gpu-cost-exporter
go: creating new go.mod: module github.com/tarhov/gpu-cost-exporter
go: to add module requirements and sums:
	go mod tidy

# 2. Move logic behind internal/, keep main thin:
#      internal/exporter/collect.go   -> package exporter (business logic)
#      cmd/exporter/main.go           -> package main     (wiring only)

# 3. Add and PIN a real dependency at an explicit version.
$ go get github.com/prometheus/client_golang@v1.20.5
go: downloading github.com/prometheus/client_golang v1.20.5
go: downloading github.com/prometheus/procfs v0.15.1
go: downloading golang.org/x/sys v0.22.0
go: downloading google.golang.org/protobuf v1.34.2
go: downloading github.com/klauspost/compress v1.17.9
go: added github.com/prometheus/client_golang v1.20.5

# 4. Sync go.mod/go.sum and verify the tree builds.
$ go mod tidy
$ go build ./...
```

The resulting `go.mod` — one direct requirement, nine indirect, and the pruned-graph block that
Go 1.17+ produces:

```
module github.com/tarhov/gpu-cost-exporter

go 1.24.7

require github.com/prometheus/client_golang v1.20.5

require (
	github.com/beorn7/perks v1.0.1 // indirect
	github.com/cespare/xxhash/v2 v2.3.0 // indirect
	github.com/klauspost/compress v1.17.9 // indirect
	github.com/munnerz/goautoneg v0.0.0-20191010083416-a7dc8b61c822 // indirect
	github.com/prometheus/client_model v0.6.1 // indirect
	github.com/prometheus/common v0.55.0 // indirect
	github.com/prometheus/procfs v0.15.1 // indirect
	golang.org/x/sys v0.22.0 // indirect
	google.golang.org/protobuf v1.34.2 // indirect
)
```

`cmd/exporter/main.go` — wiring only, importing the internal package it is allowed to see:

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

Now verify each property, rather than assuming it.

**The boundary holds.** Create a throwaway second module that `replace`s this one to a local path
and tries to import `internal/`:

```bash
$ cat go.mod
module example.com/outsider
go 1.24.7
require github.com/tarhov/gpu-cost-exporter v0.0.0
replace github.com/tarhov/gpu-cost-exporter => ../gpu-cost-exporter

$ cat main.go
package main
import "github.com/tarhov/gpu-cost-exporter/internal/exporter"
func main() { _ = exporter.MustRegister }

$ go build ./...
package example.com/outsider
	main.go:3:8: use of internal package github.com/tarhov/gpu-cost-exporter/internal/exporter not allowed
```

Note that even a `replace` pointing straight at the source directory cannot get past it — the rule
is on the *import path*, not on where the bytes came from.

**The dependency is pinned and hashed.** `go.sum` has 49 lines for this one direct dependency —
roughly two per module version in the graph, one for the `.zip` and one for the `go.mod`:

```
$ head -4 go.sum
github.com/alecthomas/kingpin/v2 v2.4.0/go.mod h1:0gyi0zQnjuFk8xrkNKamJoyUo382HRL7ATRpFZCw6tE=
github.com/alecthomas/units v0.0.0-20211218093645-b94a6e3cc137/go.mod h1:OMCwj8VM1Kc9e19TLln2VL61YJF0x1XFtfdL4JdbSyE=
github.com/beorn7/perks v1.0.1 h1:VlbKKnNfV8bJzeqoa4cOKqO6bYr3WgKZxO8Z16+hsOM=
github.com/beorn7/perks v1.0.1/go.mod h1:G2ZrVWU2WbWT9wwq4/hrbKbnv/1ERSJQ0ibhJ6rlkpw=
```

`kingpin/v2` has only a `/go.mod` line: MVS read its `go.mod` while computing the maximum, but no
package from it is in the build, so the source zip was never downloaded. That distinction is
exactly what the `/go.mod` suffix encodes.

**MVS is doing real work.** The build list contains 37 modules resolved from one direct
requirement:

```
$ go list -m all | wc -l
38
$ go list -m -f '{{.Path}} {{.Version}}' golang.org/x/sys google.golang.org/protobuf
golang.org/x/sys v0.22.0
google.golang.org/protobuf v1.34.2
```

Both are maxima over multiple conflicting requirements, worked out earlier in this lesson.

**The release gate.** Add this to the release job so a stray local `replace` can never ship:

```bash
if go mod edit -json | jq -e '.Replace[]? | select(.New.Path | test("^\\.{1,2}/"))' >/dev/null; then
  echo "ERROR: go.mod contains a local-path replace directive; refusing to release" >&2
  exit 1
fi
```

Test the gate by adding `replace github.com/foo/bar => ../bar` to `go.mod` and re-running it. A
check you have never seen fail is a check you have not verified.

## Practice

Restructure `gpu-cost-exporter` into a real module.

1. `go mod init github.com/<you>/gpu-cost-exporter` — use your actual repo URL, not a bare name.
2. Move aggregation/cost logic into `internal/exporter/`; keep only wiring in
   `cmd/exporter/main.go`. Confirm `main` contains no logic worth testing.
3. Add and **pin** one third-party dependency — `github.com/prometheus/client_golang` at an explicit
   version — then `go mod tidy`. Commit both `go.mod` and `go.sum`.
4. **Prove the `internal/` boundary** rather than asserting it: create a throwaway module in a
   sibling directory that `replace`s yours to a local path and imports `internal/exporter`, run
   `go build`, and paste the `use of internal package ... not allowed` error into your notes.
5. **Read your own dependency graph.** Run `go mod graph` and `go list -m all`, pick one module that
   appears with two different required versions, and write one sentence explaining which version
   MVS selected and why. Then run `go mod why -m <that module>` and explain the import path it
   prints.
6. Sketch the kubebuilder growth path by adding empty `api/v1alpha1/` and `internal/controller/`
   directories, each with a one-line `doc.go` package comment.
7. Add the `replace`-directive release check above, and **verify it fires**: add a local `replace`,
   watch it fail, remove it, watch it pass.

**Acceptance:**

- `go build ./...` and `go vet ./...` succeed.
- The `internal/` boundary demonstrably holds — you have the compiler error, not an assumption.
- `go.mod` shows exactly one direct third-party `require` at an explicit version, and `go.sum` has
  its hashes.
- You can name, from your own graph, one module whose selected version differs from what some
  dependency asked for, and say why.
- The `replace` check passes on a clean tree and fails on a deliberately dirtied one.
- `go.work` (if you created one) is in `.gitignore`.

Full deliverable spec: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md).

## Common pitfalls

1. **Believing MVS "always picks the oldest version."** *Mechanism:* MVS keeps the **maximum**
   version required by anything in the graph — in the real example above, `x/sys` resolved to
   v0.22.0 even though one dependency asked for v0.20.0. "Minimal" means it never goes *higher*
   than something asked for, not that it prefers old versions. *Correction:* read `go mod graph`
   and take the max per path; that is the whole algorithm.

2. **A `replace => ../local` shipped in a tagged release.** *Symptom:* downstream users report
   "cannot find module" or, worse, get a silently different resolution. *Mechanism:* the directive
   is committed in `go.mod` and the local path exists only on your machine. *Correction:* use
   `go.work` for local multi-module development, and gate releases on a `go mod edit -json` check.

3. **Confusing `go.work`'s scope.** *Symptom:* "it builds here but not in CI," or a build that
   uses uncommitted local changes you forgot about. *Mechanism:* `go.work` is discovered by walking
   up from the working directory and is never committed or published, so it silently overrides
   pinned versions for you and nobody else. *Correction:* `go env GOWORK` to see which file is in
   effect; `GOWORK=off` to build as CI would.

4. **Skipping `go mod tidy` after a refactor.** *Symptom:* a vulnerability scanner flags a module
   nothing imports; `go.sum` grows. *Mechanism:* removing an import does not remove the `require`;
   the graph keeps the module and its hashes. *Correction:* `go mod tidy` after any import change,
   and check that `go.mod`/`go.sum` diffs in a PR are explained by that PR.

5. **Treating `internal/` as a naming convention.** *Mechanism:* it is enforced by the compiler
   against the *import path*, keyed on the parent of the `internal` directory — not on the module
   root, and not on where the source is on disk. *Correction:* trust it. The failure mode here is
   over-defending a boundary that is already absolute; spend the effort elsewhere.

6. **Putting `internal/` above the code that needs it.** *Symptom:* "use of internal package not
   allowed" *within your own module*. *Mechanism:* the boundary is the parent of `internal/`, and
   packages **above** that parent cannot import it either. `foo/internal/baz` is invisible to the
   module root. *Correction:* put `internal/` at the level whose subtree should have access —
   usually the repo root for a server.

7. **`GOSUMDB=off` or `GOFLAGS=-mod=mod` in CI to make an error go away.** *Mechanism:* the first
   disables checksum-database verification globally; the second lets a build silently rewrite your
   dependency graph. Both convert a loud failure into a silent one. *Correction:* `GOPRIVATE` for
   private module paths (it narrows both proxy and sumdb to exactly those paths), and
   `-mod=readonly` in CI so `go.mod` changes must be a deliberate human commit.

8. **Publishing v2 without moving your own imports.** *Symptom:* your v2 module compiles but
   behaves like v1. *Mechanism:* the module path gained `/v2`, but internal imports still name the
   v1 path, so the build resolves them to the published v1 module. *Correction:* when you take the
   `/v2` suffix, rewrite every internal import too, and check with `go list -deps ./... | grep
   yourmodule` that no v1 path survives.

## Self-check

**(a)** What does `internal/` actually enforce?

**Answer:** Compiler-enforced import visibility. Code in or below a directory named `internal` is
importable only by code whose import path shares the prefix **above** that `internal` directory. So
`example.com/m/foo/internal/baz` can be imported by `example.com/m/foo`, `example.com/m/foo/bar`,
and `example.com/m/foo/quux`, but not by `example.com/m/crash/bang` and not by `example.com/m`
itself — nor by any other module, ever, even one that `replace`s yours to a local path. The
violation is a build error: `use of internal package ... not allowed`. That hard guarantee is what
makes everything under `internal/` safe to refactor without a cross-team deprecation cycle, because
the set of possible callers is finite and inside your repo.

**(b)** When do you need a `replace` directive, and what is the risk if you forget to remove it?

**Answer:** When the build must use a module source other than what the version graph resolves to:
a local checkout you are editing in lockstep, a fork carrying a patch, or forcing a transitive
dependency to a fixed version while debugging. Note two semantics: only the **main module's**
`replace` directives apply (a dependency's are ignored, so nobody can redirect your build), and a
`replace` needs a matching `require` from somewhere or it is a no-op.

The risk is that `replace github.com/foo/bar => ../bar` is committed in `go.mod` and therefore
**ships in your tag**. It builds for you and fails — or silently resolves differently — for every
consumer without that path. The right prevention is structural, not procedural: use `go.work` for
local multi-module work (it is never committed, so nothing can leak) and gate the release job on
`go mod edit -json` finding no filesystem `replace`.

**(c)** What does Minimal Version Selection actually pick, and what is the common misconception?

**Answer:** For each module path in the graph, MVS selects the **maximum version required by
anything reachable from the main module**. Concretely, in the exporter's real graph
`client_golang@v1.20.5` requires `golang.org/x/sys@v0.22.0`, `common@v0.55.0` requires v0.21.0, and
`procfs@v0.15.1` requires v0.20.0 — MVS selects **v0.22.0**. It does *not* pick the newest version
that exists upstream (v0.36.0 was available and was not selected, because nothing asked for it), and
it does *not* — the common misconception — pick the oldest. "Minimal" means it never upgrades past
what something in the graph actually requires.

That is what makes the build reproducible with **no lockfile at all**: the build list is a pure
function of the `go.mod` files reachable from your commit, so it cannot change when a maintainer
publishes a new release. `go.sum` is not a lockfile — it records hashes for tamper evidence, which
is a different job.

**(d)** What problem do workspaces (`go.work`) solve that vendoring does not, and what is their
scope?

**Answer:** They provide a **local, uncommitted overlay** for developing several modules together
without touching any module's `go.mod`. The modules listed under `use` all become main modules for
MVS, so imports between them resolve to the local directories. Vendoring solves a different problem
— it freezes dependency *snapshots* into the repo for hermetic or air-gapped builds — and `replace`
solves it by mutating a committed file that can leak into a release.

Scope: `go.work` is found by checking the `GOWORK` environment variable, then by walking up from
the working directory. It affects builds run inside that tree on that machine and nothing else. It
is never committed and never published, so it has **zero** effect on anyone consuming your module —
expecting otherwise is the standard misunderstanding. `go env GOWORK` tells you which file is
active, and `GOWORK=off` gives you a CI-equivalent build. When you are ready to make the local
versions real, `go work sync` writes the resolved build list back into each member's `go.mod`.

**(e)** Your `go.sum` and the module you are downloading disagree. What happens, and what is the
mechanism that would have caught a compromised upstream you had *never* downloaded before?

**Answer:** The build stops with `SECURITY ERROR: This download does NOT match an earlier download
recorded in go.sum`, printing both the computed and the expected `h1:` hash, and the downloaded file
is discarded rather than entering the module cache. The hash is SHA-256 over a deterministic listing
of the module zip's file names and contents, so it is immune to recompression or reordering.

For a module you have never fetched, `go.sum` has nothing to compare against — so the `go` command
consults the **checksum database** (`GOSUMDB`, default `sum.golang.org`) before trusting the
download, and only then writes the line into `go.sum`. That database is a **transparent log**: a
Merkle tree of `go.sum` lines, run on Trillian, whose signed tree heads let independent auditors
detect a server that served different hashes to different clients. So the guarantee is not "trust
Google," it is "Google cannot lie selectively without producing provable inconsistency." The
narrow, legitimate bypass for private code is `GOPRIVATE=github.com/mycorp/*`, which turns off both
the public proxy and the public checksum database for exactly those paths; `GOSUMDB=off` is the
blunt instrument and should be rare.

**(f)** Why does a modern `go.mod` have a long block of `// indirect` requirements, and should you
delete them?

**Answer:** No — deleting them makes builds slower and can break resolution. Since Go 1.17, a
module's `go.mod` records an explicit `require` for **every** module providing a package
transitively imported by a package or test in the main module. That extra information enables
**module graph pruning**: when loading a dependency that itself declares `go 1.17` or higher, the
`go` command loads only that dependency's *immediate* requirements rather than its full transitive
closure, because anything that could affect your packages is already listed in your own `go.mod`.
It also enables **lazy module loading**, where the `go` command tries to resolve all imports from
the main module's `go.mod` alone and only loads the wider graph if something is missing. Before
1.17, `go.mod` listed only direct dependencies and the toolchain had to walk the entire closure to
be safe. So the long block is the index that makes the fast path possible. `go mod tidy` maintains
it; hand-editing it does not.

## Connections & what's next

This lesson turns the `internal/` boundary your black-box tests in lesson 05 were already
respecting into a compiler-enforced contract, and it explains the `testdata/` convention those
golden-file tests relied on. The pinned-dependency discipline here — `go.mod`, `go.sum`, MVS, the
checksum database — is exactly what makes lesson 09's `controller-runtime` and `envtest`
dependencies resolve identically on your laptop and in CI, which is a precondition for envtest
being useful at all. It also sets up lesson 08's "read unfamiliar Go": `go mod why`, `go list -m
all`, and the module-path-to-import-path mapping are how you navigate from an import statement in
Kubernetes source to the file that defines it.

Next: lesson 07, **Stdlib fluency**, assumes you can already add and pin a real third-party
dependency cleanly. It spends its hours on `net/http`, `encoding/json`, `log/slog`, and
`prometheus/client_golang` idioms rather than on module mechanics, because this lesson already
covered how the dependency got there.

## References & further reading

**Primary sources**

1. [Go Modules Reference](https://go.dev/ref/mod) — the authoritative spec, and the source for
   nearly everything above: the MVS definition and its replacement/exclusion/upgrade/downgrade
   variants, every `go.mod` directive with its grammar, the `go.sum` three-field format, module
   graph pruning and lazy loading, workspaces, vendoring, and the full environment-variable table.
   Read the MVS and Authenticating-modules sections in full at least once.
2. [Russ Cox — "Minimal Version Selection"](https://research.swtch.com/vgo-mvs) — the original
   design essay the reference cites, with the algorithm's rationale and the worked graphs. Read for
   *why* the maximum-of-requirements rule produces reproducibility without a lockfile.
3. [Russ Cox — "Semantic Import Versioning"](https://research.swtch.com/vgo-import) — the import
   compatibility rule ("if an old package and a new package have the same import path, the new
   package must be backwards compatible") and why it forces the `/vN` suffix. Read for the diamond
   dependency problem it solves.
4. [`go help packages` — Internal packages](https://pkg.go.dev/cmd/go#hdr-Internal_Directories) —
   the exact `internal/` rule and the canonical directory example reproduced in this lesson.
5. [Managing dependencies](https://go.dev/doc/modules/managing-dependencies) — the task-oriented
   guide to `go get`, `go mod tidy`, version queries, and `replace`. Read for the fastest path to
   correct day-to-day hygiene.
6. [Organizing a Go module](https://go.dev/doc/modules/layout) — official layout guidance, including
   the explicit recommendation that server projects keep implementation under `internal/` and
   binaries under `cmd/`. This is the doc to cite when someone proposes `src/` or `pkg/`.
7. [Module graph pruning](https://go.dev/ref/mod#graph-pruning) and the
   [lazy module loading design document](https://go.googlesource.com/proposal/+/master/design/36460-lazy-module-loading.md) —
   why `go.mod` grew a long `// indirect` block in Go 1.17 and what it buys.
8. [`go help module-auth`](https://pkg.go.dev/cmd/go#hdr-Module_authentication) and
   [Module downloading and verification](https://pkg.go.dev/cmd/go#hdr-Module_downloading_and_verification) —
   the mechanics of `GOPROXY`, `GOSUMDB`, `GOPRIVATE`, `GONOPROXY`, `GONOSUMDB`, and the error text
   you get on a mismatch.
9. [Proposal: Secure the Public Go Module Ecosystem](https://go.googlesource.com/proposal/+/master/design/25530-sumdb.md) —
   the checksum-database design: the Merkle-tree/transparent-log construction, the endpoint
   protocol, and the threat model it addresses. Pairs with Russ Cox's
   [Transparent Logs for Skeptical Clients](https://research.swtch.com/tlog).
10. [Go 1.24 release notes — `tool` directive](https://go.dev/doc/go1.24) — the `tool` directive and
    `go get -tool`, which replace the `tools.go` blank-import workaround for pinning code-generation
    tools like `controller-gen`.
11. [Go toolchains](https://go.dev/doc/toolchain) — how the `go` and `toolchain` directives select
    and, since Go 1.21, automatically download the toolchain that builds your module.

**Real-world engineering blogs**

12. [Google Cloud — "Best practices for dependency management"](https://cloud.google.com/blog/topics/developers-practitioners/best-practices-dependency-management) —
    what it shows: pinning plus hash verification plus vulnerability scanning framed as
    supply-chain security rather than build hygiene, including dependency-confusion avoidance —
    which in Go is exactly what `GOPRIVATE`/`GONOPROXY` are for.
13. [Monzo — "Migrating our monorepo seamlessly from Dep to Go Modules"](https://monzo.com/blog/2022/09/29/migrating-our-monorepo-seamlessly-from-dep-to-go-modules) —
    what it shows: a live production monorepo migrating onto Go modules with zero downtime; proof
    the module system's discipline scales past a toy repo.

**Deeper dives**

14. [`govulncheck`](https://go.dev/blog/govulncheck) — the official vulnerability scanner. It uses
    the module graph *and* static reachability analysis, so it reports only vulnerabilities in code
    your build actually calls, rather than every CVE in your dependency tree. Add
    `go tool govulncheck ./...` to CI once the module in this lesson exists; it is the piece that
    turns pinned dependencies from a reproducibility feature into a security one.
