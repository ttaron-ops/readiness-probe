---
lesson: "01.5"
title: "Testing in Go"
module: "01"
concept: "Testing in Go"
status: not-started
est_time: "14h"
prev: "04-concurrency-and-context.md"
next: "06-modules-and-layout.md"
artifacts: []
sources: 17
---

# 01.5 · Testing in Go

> **Concept.** Table-driven tests, fakes over mocks, `-race`, and the controller testing story
> (envtest + fake client) — plus the fidelity ladder between them and the review culture that
> makes your test file the first thing a senior reviewer reads.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 04 gave you the concurrency primitives a controller runs on — goroutines, channels,
`context.Context` for cancellation, `errgroup` for bounded fan-out, and `-race` as the tool that
catches unsynchronised access. That raises an obvious question: how do you *prove* concurrent code
is correct, how do you test a five-second timeout without waiting five seconds, and how do you test
something that calls a live Kubernetes API without standing up a cluster for every `go test` run?
This lesson answers all three. It closes the "write it" arc — interfaces (03), concurrency (04),
testing (05) are the three lessons the module README says never to shortcut — before lesson 06
shifts from "does the code work" to "how do you package, version, and ship it."

## Why this matters

Take-home and on-site rounds for platform roles are won in the test file, not the source file. This
is not folklore: Google's public code-review standard lists **Tests** as one of ten things a
reviewer must examine, and its instruction is specifically adversarial — *"Will the tests actually
fail when the code is broken? If the code changes beneath them, will they start producing false
positives?"* and *"Tests do not test themselves… a human must ensure that tests are valid."* A
green suite is not evidence; a suite that would go red for the right reason is.

Operationally the stakes are higher than the interview. Every serious controller repo — Kubernetes
itself, cert-manager, Karpenter — ships table-driven tests, `envtest`, and `-race` in CI. The
failure mode when they do not is specific and expensive: a reconciler whose status update races a
stale `resourceVersion` passes every fake-client test and then live-locks in production under
concurrent writers; a scrape whose goroutines leak passes every functional test and OOMKills the
pod four days later. The tests you write here are the only place those bugs are cheap to find.

## What's new here (calibration)

You already know general software testing from Python — what a unit test is, what mocking is, what
coverage means, `pytest.mark.parametrize`, fixtures. We skip re-teaching those. What is genuinely
new, calibrated to a staff-level GPU-fleet-operator track:

- **The absence of magic is deliberate, and it has mechanics.** A Go table is an ordinary slice
  literal and `t.Run` is an ordinary method that starts a goroutine. Knowing exactly what `t.Run`
  and `t.Parallel` do to the test tree is what lets you debug a flaky suite instead of guessing.
- **`testing/synctest`** (generally available in Go 1.25) — a fake clock and a deterministic
  "all goroutines are blocked" barrier, built into the standard library. This is the answer to
  testing timeouts, retries, and backoff without `time.Sleep` in tests, and most material written
  before 2025 does not mention it.
- **The fake-vs-envtest split is a fidelity ladder**, and the popular account of where the fake
  client's fidelity runs out is out of date. Knowing what it *actually* models today — it does
  track `resourceVersion` and does return `Conflict` — and what it still does not is the
  staff-level version of this knowledge.
- **Coverage as a signal, not a target**, plus the mechanics that make the number mean something
  (statement coverage, `-covermode`, `-coverpkg`).
- **`t.Cleanup` vs `defer`**, `t.Context`, `t.Setenv`'s interaction with `t.Parallel`, and async
  assertion discipline — small mechanics, load-bearing in a controller repo.

## Core concepts

### What `go test` actually builds and runs

Understanding the rest of this lesson is much easier once you know the artefact. `go test ./...`
does not interpret your test files; it **generates a `main` package, compiles a binary per package,
and runs it**.

```
 WHAT `go test ./internal/exporter` PRODUCES
 ═══════════════════════════════════════════════════════════════════════

  SOURCE                              COMPILED INTO
  ──────                              ─────────────
  collect.go        package exporter  ─┐
  aggregate.go      package exporter   ├─▶ [ exporter ]           the package
  collect_test.go   package exporter  ─┘   (+ white-box tests
                                            compiled INTO it)

  export_test.go    package exporter_test ─▶ [ exporter_test ]    black-box tests:
                                                                  can only see the
                                                                  exported API

  (generated by cmd/go, never written to your repo)
  _testmain.go      package main ─────────▶ [ exporter.test ]     the actual binary
     tests    = []testing.InternalTest{{"TestCollect", TestCollect}, ...}
     benchmarks, fuzzTargets, examples likewise
     func main() { testing.Main(matchString, tests, benchmarks, ...) }

  RUN AS:  ./exporter.test -test.v -test.run=... -test.parallel=8 ...
           (the flags `go test` shows you are rewritten with a `test.` prefix)
```

Three consequences:

- **Two package flavours.** `package exporter` in a `_test.go` file is a **white-box** test: it
  compiles into the package and can touch unexported identifiers. `package exporter_test` is a
  **black-box** test: a separate package that imports `exporter` and sees only the exported API.
  Both live in the same directory; this is the one place Go allows two package names in one folder.
  Use black-box by default — it tests the API a caller actually has, and it cannot accidentally
  depend on an implementation detail. Drop to white-box when you need to reach a helper.
- **`TestMain(m *testing.M)`**, if present, replaces the default entry point. It runs on the main
  goroutine, before `flag.Parse()` has happened, and must call `m.Run()`. This is where you start
  an `envtest` control plane once for the whole package, or install `goleak.VerifyTestMain(m)`.
- **Results are cached.** In package-list mode `go test` reuses a previous successful result and
  prints `(cached)`. The cache key is the test binary plus a *restricted* set of "cacheable" flags:
  `-benchtime -coverprofile -cpu -failfast -fullpath -list -outputdir -parallel -run -short -skip
  -timeout -v`. Any other flag disables caching. Tests that read files inside the module or consult
  environment variables only hit the cache if those are unchanged. **The idiomatic way to force a
  re-run is `-count=1`** — not `-a`, not clearing `GOCACHE`.

### Table-driven tests and subtests

The dominant Go idiom: one test function, a slice of cases, `t.Run` per case.

```go
package exporter_test

import (
	"maps"
	"testing"

	"github.com/tarhov/gpu-cost-exporter/internal/exporter"
)

func TestAggregate(t *testing.T) {
	tests := []struct {
		name string
		in   []exporter.Sample
		want map[string]float64
	}{
		{"empty", nil, map[string]float64{}},
		{"single", []exporter.Sample{{Pool: "a100", Cost: 3.0}}, map[string]float64{"a100": 3.0}},
		{"dupes sum", []exporter.Sample{{Pool: "a100", Cost: 1}, {Pool: "a100", Cost: 2}}, map[string]float64{"a100": 3.0}},
		{"distinct pools", []exporter.Sample{{Pool: "a100", Cost: 1}, {Pool: "h100", Cost: 4}}, map[string]float64{"a100": 1, "h100": 4}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := exporter.Aggregate(tt.in)
			if !maps.Equal(got, tt.want) {
				t.Errorf("Aggregate(%v) = %v, want %v", tt.in, got, tt.want)
			}
		})
	}
}
```

`maps.Equal` (Go 1.21+) beats `reflect.DeepEqual` for maps of comparable values: it is typed, it is
faster, and it will not silently succeed on two `nil` maps you meant to distinguish.

**What `t.Run` actually does.** It is not a decorator and there is no framework layer. Reading
`src/testing/testing.go`:

1. Compute the full name — parent name, `/`, the subtest name, with spaces replaced by underscores
   and a `#01` suffix appended to disambiguate duplicates. That is why a case named `dupes sum`
   prints as `TestAggregate/dupes_sum`.
2. Match it against the `-run` regexp. `-run` is **slash-separated and unanchored**, matched
   element by element: `-run TestAggregate/dupes` runs one case; `-run /dupes` runs any subtest
   containing `dupes` under any top-level test.
3. Allocate a fresh `T` with `barrier: make(chan bool)`, `signal: make(chan bool, 1)`, a parent
   pointer, and — since Go 1.24 — its own `context.WithCancel(context.Background())` for
   `t.Context()`. Note that the subtest context is *not* derived from the parent test's.
4. **`go tRunner(t, f)`** — every subtest runs on its own goroutine, parallel or not.
5. Block on `<-t.signal` until the subtest finishes *or* calls `t.Parallel()`.

The practical payoffs of the naming rule: failures identify themselves
(`--- FAIL: TestAggregate/dupes_sum`), you can rerun exactly one case, and because the table is a
plain slice literal you can print it, diff it in review, and set a breakpoint on one row.

**The one table-driven anti-pattern worth naming.** Do not put logic in the table. A field called
`wantErrContains string` is fine; a field called `check func(t *testing.T, got X)` means the table
is now code with cases, and you have reinvented the very framework indirection Go's idiom avoids.
When cases genuinely need different logic, write two test functions.

### `t.Parallel`, precisely

`t.Parallel()` is where flaky suites are born, so it is worth knowing the exact mechanism rather
than the folklore.

```
 THE t.Parallel BARRIER  (src/testing/testing.go)
 ═══════════════════════════════════════════════════════════════════════

 PARENT goroutine                    SUBTEST goroutines
 ────────────────                    ──────────────────
 t.Run("a", f)
   ├─ go tRunner(a, f) ─────────────▶ a: runs f
   └─ block on <-a.signal                ├─ f calls t.Parallel()
                                         │    ├─ append a to parent.sub
                                         │    ├─ a.signal <- true ───┐
                                         │    └─ block on            │
                                         │       <-parent.barrier    │
   ◀──────────────────────────────────────────────────────────────────┘
 t.Run("b", g)   (same dance)   ────▶ b: ... blocks on parent.barrier
 t.Run("c", h)   (h never calls
                  Parallel)     ────▶ c: runs to completion, THEN signals
   └─ block on <-c.signal
 ...parent's own body returns

 tRunner(parent) sees len(parent.sub) > 0:
   ├─ close(parent.barrier) ───────▶ a and b BOTH resume, here
   │                                 (each first waits on tstate.waitParallel(),
   │                                  which caps concurrency at -test.parallel,
   │                                  default = GOMAXPROCS)
   ├─ for _, sub := range parent.sub { <-sub.signal }   ← wait for all
   └─ then run parent's t.Cleanup functions

 TIMELINE CONSEQUENCE
   t=0    a starts, immediately parks at the barrier
   t=0    b starts, immediately parks
   t=0    c runs serially, start to finish
   t=1    parent body returns
   t=1    barrier closes — a and b now run CONCURRENTLY with each other
   t=2    parent cleanup runs, after a and b have both finished
```

Read four things off that:

- **A parallel subtest does not start when you call `t.Run`.** It starts when the parent's body
  returns. Anything the parent's body mutates after launching subtests is visible to all of them.
- **Parallel siblings run concurrently with each other**, and only with each other. This is why the
  "group" idiom exists: wrap a set of parallel subtests in an outer `t.Run("group", ...)` so the
  parent can clean up after all of them.
- **Concurrency is capped at `-test.parallel`, default `runtime.GOMAXPROCS(0)`.** So on a 2-CPU CI
  runner, 40 parallel subtests run 2 at a time. If your suite "only fails in CI," a different
  `-parallel` value is a prime suspect.
- **`t.Parallel()` twice panics**, and it is a no-op when fuzzing.

**Shared mutable fixtures are the real trap.** The loop-variable problem is mostly historical:
pre-Go 1.22 the range variable was one variable reused across iterations, so a parallel closure
usually saw the *last* case and you wrote `tt := tt` to shadow it. **Go 1.22 gave each iteration a
fresh variable** when the module's `go` directive declares 1.22 or later, so the shadow line is
dead code in new work. Know both facts — interviewers ask, and older repos still carry the shadow.

The sharper, still-live trap is a fixture shared *across* cases: one `*bytes.Buffer`, one map
passed by reference, one package-level `*httptest.Server` whose handler counts calls. Mark those
subtests `t.Parallel()` and you have not just risked a wrong assertion — you have introduced a
**data race between subtests**, which `-race` catches only if CI runs with `-race` *and* the racing
writes happen to overlap in that run. Give every case its own fixture; construct it inside the
`t.Run` closure, not in the table.

Finally, three `testing` methods are **incompatible with `t.Parallel`** because they mutate
process-global state: `t.Setenv`, `t.Chdir`, and `goleak.VerifyNone`. The first two panic if the
test is parallel; `goleak` cannot attribute goroutines to a specific test when several run at once.

### `t.Cleanup`, `t.Context`, and the rest of the helper surface

`t.Cleanup(fn)` registers a function to run when the test *and all its subtests* complete. Reading
`runCleanup` in the testing package, it does four things `defer` cannot:

1. **LIFO order**, popping from the end of `c.cleanups` — same as `defer`, but registered from
   anywhere, including a helper function that has already returned.
2. **Cancels `t.Context()` first**, before any cleanup runs. So a `t.Context()`-scoped server or
   manager is already being told to stop by the time your cleanup waits for it.
3. **Runs the remaining cleanups even if one panics**, via a `defer` that recurses into
   `runCleanup` if the list is non-empty.
4. **Fires at the right point in the test tree**, which is the whole reason it exists. A `defer`
   inside a shared setup helper runs when *that helper* returns — long before the test that uses
   what it set up has finished. A `t.Cleanup` registered by the same helper runs when the *test*
   finishes.

```go
// newFakeKubelet starts a test server and registers its own teardown.
// The caller writes ONE line and cannot leak the server.
func newFakeKubelet(t *testing.T, resp string) string {
	t.Helper()
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		_, _ = io.WriteString(w, resp)
	}))
	t.Cleanup(srv.Close) // NOT `defer srv.Close()` — that would fire before the test runs
	return srv.URL
}
```

Calling `t.Run` from inside a cleanup panics (`testing: t.Run called during t.Cleanup`) — the tree
is already collapsing.

The rest of the surface worth having memorised:

| Method | Since | What it gives you |
|---|---|---|
| `t.Helper()` | 1.9 | Marks the function as a helper so failures report the *caller's* line. Put it first in every assert helper you write. |
| `t.TempDir()` | 1.15 | A unique directory, removed automatically after the test. |
| `t.Setenv(k, v)` | 1.17 | Sets an env var and restores it. **Panics if the test is parallel.** |
| `t.Chdir(dir)` | 1.24 | Same, for the working directory. **Panics if parallel.** |
| `t.Context()` | 1.24 | A context cancelled *before* cleanup functions run — thread it into everything the test starts. |
| `t.Attr(k, v)` | 1.25 | Emits a structured key/value into the test log and `-json` output. Useful for CI dashboards. |
| `t.Output()` | 1.25 | An `io.Writer` writing into the test's log, correctly interleaved. Point a `slog` handler at it. |
| `t.ArtifactDir()` | 1.26 | A directory for output files; with `go test -artifacts` they survive the run. |
| `t.Skip` / `testing.Short()` | 1.0 | `if testing.Short() { t.Skip(...) }` is the standard gate for slow tests; CI runs `-short` on PRs and the full suite nightly. |

### Fakes over mocks

Depend on a small interface (lesson 03's rule: define it on the consumer), and pass a hand-written
fake in tests. This is the single biggest delta from Python's `unittest.mock` habit, and the reason
is structural rather than stylistic: **there is nothing to monkeypatch in Go.** A package-level
function is not an attribute you can reassign, a method set is fixed at compile time, and there is
no `__getattr__`. Substitution has to happen through an interface, at a seam you designed.

```go
// The interface lives in the consuming package and is exactly as wide as needed.
type CostSource interface {
	Rates(ctx context.Context) (map[string]float64, error)
}

// The fake is a struct literal. No library, no code generation, no DSL.
type fakeSource struct {
	rates map[string]float64
	err   error

	// Recording fields — only add these when a test asserts on them.
	calls int
}

func (f *fakeSource) Rates(ctx context.Context) (map[string]float64, error) {
	f.calls++
	if f.err != nil {
		return nil, f.err
	}
	return f.rates, nil
}
```

Two refinements that make fakes carry more weight:

**Make the fake honour the contract you care about.** If the real source is cancellable, the fake
should be too — otherwise a test can pass while the production path ignores `ctx`:

```go
func (f *fakeSource) Rates(ctx context.Context) (map[string]float64, error) {
	select {
	case <-ctx.Done():
		return nil, ctx.Err()
	case <-f.ready: // nil channel by default → never fires → the ctx arm decides
		return f.rates, f.err
	}
}
```

**Assert the compile-time contract once**, so a later interface change breaks at build time rather
than in a confusing test failure:

```go
var _ CostSource = (*fakeSource)(nil)
```

**When to reach for a generated mock** (`go.uber.org/mock`, formerly `golang/mock`, or `mockery`):
when the assertion is genuinely about *interaction* — exact arguments, call counts, or ordering
across several methods, e.g. "the reconciler must call `Get`, then `Update`, then never `Delete`."
For return-value stubbing a fake is less code, survives refactors, and reads like Go. The rule:
**fakes verify state, mocks verify behaviour.** Pick the one that matches what you are asserting,
not the one your previous language defaulted to.

### testify, judiciously

`require`/`assert` cut boilerplate: `require.NoError(t, err)` stops the test (it calls `t.FailNow`),
`assert.Equal(t, want, got)` records a failure and continues. Use `require` for preconditions whose
failure would make the rest of the test panic or lie, and `assert` for the independent checks you
want all of.

Two cautions. First, `require.*` calls `t.FailNow()`, which calls `runtime.Goexit()` — so
**`require` must not be called from a goroutine other than the test's own**. Doing so kills that
goroutine without failing the test, and `go vet`'s `testinggoroutine` analyser exists for exactly
this. Second, plain `if got != want { t.Errorf(...) }` is fully idiomatic; the standard library
uses nothing else. Do not let testify become the thing a reviewer has to learn before reading your
tests.

For structured comparisons, `github.com/google/go-cmp/cmp` is the tool worth adding:
`cmp.Diff(want, got)` prints a readable diff rather than two dumped structs, and `cmpopts` lets you
ignore fields (timestamps, `resourceVersion`) explicitly rather than by writing a custom equality
function nobody will maintain.

### Golden files

For large expected outputs — rendered Prometheus exposition text, generated YAML, a CLI's help
output — store the expectation in `testdata/` and gate regeneration behind a flag:

```go
package exporter_test

import (
	"flag"
	"os"
	"path/filepath"
	"testing"

	"github.com/google/go-cmp/cmp"

	"github.com/tarhov/gpu-cost-exporter/internal/exporter"
)

var update = flag.Bool("update", false, "update golden files")

func TestRenderExposition(t *testing.T) {
	got := exporter.Render(sampleMetrics())
	golden := filepath.Join("testdata", "metrics.golden")

	if *update {
		if err := os.WriteFile(golden, got, 0o644); err != nil {
			t.Fatalf("write golden: %v", err)
		}
	}
	want, err := os.ReadFile(golden)
	if err != nil {
		t.Fatalf("read golden (run with -update to create): %v", err)
	}
	if diff := cmp.Diff(string(want), string(got)); diff != "" {
		t.Errorf("output differs from golden (-want +got):\n%s", diff)
	}
}
```

`testdata/` is special to the `go` tool: it is ignored for builds and package loading, so anything
in it — including files that are not valid Go — is inert. Regenerate with
`go test -run TestRenderExposition -update`, then **read the diff in review**. A golden test whose
update nobody reviews is a test that asserts whatever the code happened to produce.

### `-race` in tests, and the CI recipe

Lesson 04 covered what the race detector instruments. Two things change when it runs under `go test`:

- **`-race` implies `-covermode=atomic`.** From `go help test`: the default cover mode is `set`
  "unless `-race` is enabled, in which case it is `atomic`." That matters because `count` mode uses
  a plain `++` on the counter array, which is itself a data race under parallel tests — `atomic`
  mode makes each counter increment an atomic add, at real cost.
- **`t.Parallel` makes `-race` far more effective**, because it actually interleaves your subtests.
  A serial suite exercises very few interleavings.

The CI recipe worth standardising on:

```bash
go vet ./...
go test -race -covermode=atomic -coverprofile=cover.out ./...
go tool cover -func=cover.out | tail -1
go test -run '^$' -bench=. -benchmem ./internal/exporter   # compile-check benchmarks
```

`GORACE="halt_on_error=1"` is worth adding in CI so the first race fails fast with a clean report
instead of a wall of them. Note that `-race` requires cgo and a C toolchain, and is supported on
`linux/amd64`, `linux/arm64`, `linux/ppc64le`, `linux/s390x`, `linux/loong64`, `freebsd/amd64`,
`netbsd/amd64`, `darwin/amd64`, `darwin/arm64`, and `windows/amd64` — so a `CGO_ENABLED=0` build
image will silently not run it.

### Coverage: mechanics first, then judgement

`go test -cover` reports **statement coverage**: the tool rewrites your source, inserting a counter
increment at the top of every basic block, and reports the fraction of statements whose counter is
non-zero. That definition is doing a lot of work — it is not branch coverage, not path coverage,
and not condition coverage.

| `-covermode` | Counter type | Answers | Cost |
|---|---|---|---|
| `set` (default) | `bool` | Did this statement run? | Lowest |
| `count` | `int` | How many times? | Low, unsafe under parallelism |
| `atomic` | `int` via `sync/atomic` | How many times, correctly under concurrency | Highest; **implied by `-race`** |

Two flags change what the number means:

- **`-coverpkg`**. By default, a package's coverage counts only statements in *that* package
  executed by *its own* tests. `-coverpkg=./...` measures every listed package across the whole
  test run, which is what you want when integration tests in one package exercise code in another.
  It also lowers the number, because now you are dividing by everything.
- **`-coverprofile`** writes the profile; `go tool cover -func=cover.out` prints a per-function
  table and `go tool cover -html=cover.out` opens a line-by-line view.

Here is a real-shaped `-func` report and how to read it (representative):

```
$ go tool cover -func=cover.out
internal/exporter/aggregate.go:14:  Aggregate           100.0%
internal/exporter/collect.go:31:    Collect              86.7%
internal/exporter/collect.go:78:    scrapeOne            72.0%
internal/exporter/parse.go:22:      ParseCostExport      94.1%
internal/exporter/metrics.go:19:    MustRegister          0.0%
cmd/exporter/main.go:18:            main                  0.0%
total:                              (statements)         71.4%
```

The number to report is not 71.4%. The reading is:

- `main` and `MustRegister` at 0.0% — **fine, and say so.** `main` is wiring: flag parsing,
  `ListenAndServe`, `os.Exit`. Testing it tests the standard library. This is the boundary you drew
  deliberately in lesson 06's layout.
- `scrapeOne` at 72.0% — **this is the line to interrogate.** Open the HTML view. If the uncovered
  statements are the `resp.StatusCode != 200` branch and the JSON-decode error branch, those are
  exactly the branches that fire during an incident, and 72% is not fine.
- `Aggregate` at 100.0% — **not proof of anything**. Statement coverage of a function containing
  `if a && b` is satisfied by one input; you have covered the statement, not the four combinations.

**The interview form of this**: a staff-level interviewer shown a coverage report asks "what is
*not* covered, and is that OK?" — because a percentage is trivially gamed by testing getters while
skipping error paths. Practise the answer against your own suite: name the uncovered branch, and
give the reason (wiring, an `os.Exit` path, a genuinely unreachable default case) or admit it is a
gap. That answer demonstrates you know your code's risk surface; the number cannot.

### Benchmarks, and how `b.N` is chosen

```go
func BenchmarkAggregate(b *testing.B) {
	in := makeSamples(10_000) // setup, not measured
	b.ReportAllocs()
	for b.Loop() { // Go 1.24+: resets the timer on first call, stops it on last
		_ = exporter.Aggregate(in)
	}
}
```

`go test -bench=. -benchmem ./internal/exporter` runs it. Output:

```
goos: linux
goarch: amd64
pkg: github.com/tarhov/gpu-cost-exporter/internal/exporter
cpu: AMD EPYC 7B12
BenchmarkAggregate-8      4021    291043 ns/op    163840 B/op    2 allocs/op
                     │       │         │              │             └ allocations per iteration
                     │       │         │              └ bytes allocated per iteration
                     │       │         └ nanoseconds per iteration
                     │       └ iterations actually run (b.N)
                     └ GOMAXPROCS the benchmark ran with
```

**How `b.N` gets picked** (`src/testing/benchmark.go`). The framework runs the body once, then
loops: predict how many iterations would fill `-benchtime` (default **1 s**) via
`n = goalns * prevIters / prevns`, multiply by **1.2** for headroom, clamp growth to **100× the
previous n**, force at least `last+1`, and cap at **1e9** iterations total. It keeps running
increasing `n` until elapsed time reaches `benchtime`. So `4021` above means one `Aggregate` call
over 10,000 samples takes ~291 µs and 4021 of them fill a second.

**Prefer `b.Loop()` over `for i := 0; i < b.N; i++`** on Go 1.24+, for two reasons stated in the
package docs: the benchmark function executes exactly once per `-count`, so expensive setup and
cleanup run once instead of per ramp-up round; and the compiler keeps loop-body arguments and
results alive with a `runtime.KeepAlive` intrinsic, which prevents the classic "the optimiser
deleted my benchmark" result of a suspiciously fast `0.25 ns/op`. Do not mix the two forms.

`-benchmem` (or `b.ReportAllocs()`) reports `B/op` and `allocs/op`, and **allocs/op is the number
you tune** in a hot exporter path — it is stable across machines in a way `ns/op` is not. To
compare two runs meaningfully, run each with `-count=10` and feed both to
`golang.org/x/perf/cmd/benchstat`, which reports the delta with a confidence interval. A single
benchmark run on a laptop with a browser open is noise.

### Fuzzing, and where it earns its keep

Go 1.18 added native coverage-guided fuzzing.

```go
func FuzzParseCostExport(f *testing.F) {
	f.Add([]byte("pool,cost\na100,3.0\n"))   // seed corpus, in code
	f.Add([]byte(""))                        // degenerate seeds are the valuable ones
	f.Add([]byte("pool,cost\n"))
	f.Fuzz(func(t *testing.T, data []byte) {
		rates, err := exporter.ParseCostExport(data)
		if err != nil {
			return // a parse error is a fine outcome; a panic is not
		}
		// A property, not an expected value: whatever parses must round-trip.
		out := exporter.RenderCostExport(rates)
		again, err := exporter.ParseCostExport(out)
		if err != nil {
			t.Fatalf("re-parsing our own output failed: %v", err)
		}
		if !maps.Equal(rates, again) {
			t.Errorf("round trip changed the data: %v -> %v", rates, again)
		}
	})
}
```

The mechanics that are easy to get wrong:

- **Seeds come from two places**: `f.Add(...)` calls, and files under `testdata/fuzz/FuzzParseCostExport/`.
  The types passed to `f.Add` must exactly match the fuzz target's parameters after `*testing.T`.
- **Only these types can be fuzzed**: `[]byte`, `string`, `bool`, `byte`, `rune`, `float32`,
  `float64`, `int`, `int8/16/32/64`, `uint`, `uint8/16/32/64`. No structs, no slices of anything
  else — you decode structure from `[]byte` yourself.
- **Without `-fuzz`, a fuzz test is an ordinary test** that runs only the seed corpus. That means
  `go test ./...` in CI already runs your seeds as regression tests, for free.
- **With `-fuzz=FuzzParseCostExport`**, the engine mutates inputs, guided by coverage
  instrumentation, and **runs forever by default** — always pass `-fuzztime` (a duration, or `Nx`
  for a fixed count). On a failure it minimises the input (`-fuzzminimizetime`, default **60 s**)
  and writes the reproducer to `testdata/fuzz/FuzzParseCostExport/<hash>`, which you commit; it
  then becomes a permanent seed.
- `-fuzz` matches exactly one target in exactly one package. You cannot fuzz `./...`.

**Where it is worth it**: functions that parse bytes whose shape you do not control. In
`gpu-cost-exporter` that is exactly one function — whatever parses the cost CSV/JSON export. Pure
aggregation over already-typed Go values gets almost nothing from fuzzing, because the type system
has already constrained the input space. `t.Skip()` inside a fuzz target is the idiomatic way to
say "this input is invalid, but not a bug."

### `testing/synctest`: testing time without spending it

This is the newest and most under-known piece, and it directly solves the hardest problem left over
from lesson 04: **how do you test a five-second timeout, a backoff schedule, or a ticker without a
five-second test?**

`testing/synctest` shipped experimentally in Go 1.24 (`GOEXPERIMENT=synctest`) and became generally
available in **Go 1.25** with a revised API. `synctest.Test(t, f)` runs `f` in an isolated
**bubble**; every goroutine started inside the bubble belongs to it.

Inside a bubble:

- **The `time` package uses a fake clock**, starting at midnight UTC on 2000-01-01, private to that
  bubble.
- **Time advances only when every goroutine in the bubble is durably blocked** — and then it jumps
  instantly to the next moment that would unblock something.
- A goroutine is **durably blocked** when only another goroutine *in the same bubble* could unblock
  it: a send/receive on a bubbled channel, a select where every case is a bubbled channel,
  `sync.Cond.Wait`, `sync.WaitGroup.Wait`, and `time.Sleep`. Blocking on I/O, a syscall, or a mutex
  is **not** durable blocking — the bubble cannot know when it will end.
- **`synctest.Wait()`** blocks until every *other* goroutine in the bubble is durably blocked. That
  is the deterministic barrier: after it returns, all pending work has settled.
- If every goroutine is durably blocked and no timer can advance, `synctest.Test` **panics with a
  deadlock**, which is exactly the failure you want.

```
 A SYNCTEST BUBBLE RUNNING A 30s BACKOFF TEST IN ~0ms WALL TIME
 ═══════════════════════════════════════════════════════════════════════

 fake clock          goroutines in the bubble                wall clock
 ──────────          ────────────────────────                ──────────
 00:00:00.000  root: start retrier; synctest.Wait()             t=0.0ms
               retrier: attempt 1 → fail → time.Sleep(100ms)
               ── all goroutines durably blocked ──
               ▼ clock JUMPS to the next timer deadline
 00:00:00.100  retrier: attempt 2 → fail → time.Sleep(200ms)    t=0.1ms
               ── all durably blocked ── ▼
 00:00:00.300  retrier: attempt 3 → fail → time.Sleep(400ms)    t=0.2ms
               ── all durably blocked ── ▼
 00:00:00.700  retrier: attempt 4 → ok                          t=0.3ms
               root: synctest.Wait() returns; assert 4 attempts
                     and time.Since(start) == 700ms EXACTLY

 The assertion on elapsed time is EXACT, not approximate, because the
 clock is a data structure, not a measurement.
```

```go
package retry_test

import (
	"context"
	"errors"
	"testing"
	"testing/synctest"
	"time"

	"github.com/tarhov/gpu-cost-exporter/internal/retry"
)

func TestBackoffSchedule(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		start := time.Now() // always midnight UTC 2000-01-01 inside a bubble

		attempts := 0
		err := retry.Do(t.Context(), retry.Policy{Base: 100 * time.Millisecond, Max: 4}, func(context.Context) error {
			attempts++
			if attempts < 4 {
				return retry.ErrRetryable
			}
			return nil
		})

		if err != nil {
			t.Fatalf("Do() = %v, want nil", err)
		}
		if attempts != 4 {
			t.Errorf("attempts = %d, want 4", attempts)
		}
		// 100 + 200 + 400 = 700ms of backoff. Exact — no tolerance window needed.
		if elapsed := time.Since(start); elapsed != 700*time.Millisecond {
			t.Errorf("elapsed = %v, want 700ms", elapsed)
		}
	})
}

func TestScrapeRespectsDeadline(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		ctx, cancel := context.WithTimeout(t.Context(), 8*time.Second)
		defer cancel()

		done := make(chan error, 1)
		go func() { done <- neverReturns(ctx) }()

		synctest.Wait() // everything has settled; nothing has completed yet
		select {
		case err := <-done:
			t.Fatalf("returned early: %v", err)
		default:
		}

		// No sleeping: the bubble advances the clock to the deadline instantly.
		if err := <-done; !errors.Is(err, context.DeadlineExceeded) {
			t.Errorf("err = %v, want DeadlineExceeded", err)
		}
	})
}
```

What this replaces: `time.Sleep(1500 * time.Millisecond)` in tests, tolerance windows
(`if elapsed < 600*time.Millisecond || elapsed > 900*time.Millisecond`), injected `Clock` interfaces
threaded through production code purely for testability, and the flakiness all three produce on a
loaded CI runner. The limits are equally clear: **do not use the network inside a bubble** (a socket
read is not durable blocking, so the clock will not advance), and `t.Parallel` and `t.Run` panic
inside one.

### Leak checking as test infrastructure

Lesson 04 established that `-race` finds races and not leaks. `go.uber.org/goleak` is the CI half:

```go
func TestMain(m *testing.M) {
	goleak.VerifyTestMain(m) // one check for the whole package, after all tests
}
```

or, per test, `defer goleak.VerifyNone(t)`. Recall the mechanics from lesson 04: it snapshots
`runtime.Stack`, drops a default filter list (the `testing` package's own goroutines, syscall
stacks, the DNS resolver), and retries up to 20 times with a `1µs << i` sleep capped at 100 ms —
about 431 ms worst case — before failing. Two operational notes: it is incompatible with
`t.Parallel`, and by default it skips the check when a test has already failed, so add
`goleak.RunOnFailure()` if you want it either way.

On Go 1.26+ there is a second, sharper option: build with `GOEXPERIMENT=goroutineleakprofile`
(unconditional in Go 1.27) and read `runtime/pprof`'s `goroutineleak` profile, which uses GC
reachability to report goroutines that *provably* can never wake. `goleak` answers "did anything
outlive this test"; the runtime profile answers "is this goroutine permanently stuck." Use `goleak`
as the gate and the profile when you need to know which one and why.

### The fidelity ladder

There is no single "how you test a controller." There are four rungs, and picking the rung is a
design decision you should be able to defend.

```
 THE FIDELITY LADDER
 ═══════════════════════════════════════════════════════════════════════

 rung                       speed        models                    misses
 ────                       ─────        ──────                    ──────
 1  PURE UNIT               ~µs          your logic                everything
    Aggregate(samples)                   arithmetic, parsing       about k8s
    no client at all

 2  FAKE CLIENT             ~ms          CRUD, resourceVersion     OpenAPI/CRD schema
    fake.NewClientBuilder()              increments + Conflict,    validation, admission
    in-memory tracker                    status subresource,       webhooks, defaulting,
                                         field indexes, error      GC / ownerRef cascade,
                                         injection (interceptors)  finalizer semantics,
                                                                   real watch timing

 3  ENVTEST                 ~2-20s       a REAL kube-apiserver     kubelet (pods never
    real apiserver + etcd    (start)     + etcd: CRD schema        run), scheduler (pods
    no kubelet/scheduler/               validation, defaulting,    stay Pending), kube-
    controller-manager                   admission webhooks,       controller-manager
                                         true optimistic           (no GC, no ownerRef
                                         concurrency, real         cascade, no built-in
                                         watch → reconcile         controllers at all)

 4  REAL CLUSTER (kind/e2e) ~minutes     everything                nothing — but it is
                                                                   slow and flaky enough
                                                                   that you run few of them

 RULE OF THUMB
   Put the LOGIC on rung 1. Put reconcile branching on rung 2.
   Put anything touching schema, webhooks, status subresources,
   conflict retries, or watch-driven timing on rung 3.
   Keep rung 4 for a handful of smoke tests.
```

**What the fake client actually does today** — this is the part most write-ups get wrong, including
the previous version of this lesson. Reading
`sigs.k8s.io/controller-runtime/pkg/client/fake/versioned_tracker.go`:

- It **does track `resourceVersion`**. Objects seeded via `WithObjects` get `resourceVersion: "999"`
  (the constant `trackerAddResourceVersion`), and every update parses the stored RV, increments it,
  and writes it back.
- It **does enforce optimistic concurrency**: if the RV on the object you pass to `Update` differs
  from the stored one, it returns `apierrors.NewConflict(...)` with "object was modified" — the same
  error type your retry logic checks with `apierrors.IsConflict`.
- It **does model the status subresource**, if you opt in with `WithStatusSubresource(&v1.GPUNode{})`.
  Without that call, `Status().Update()` and `Update()` write the same object, which is why a test
  can pass while production fails on a spec/status split.
- It **does support error injection**, via `WithInterceptorFuncs(interceptor.Funcs{...})` — a struct
  of optional per-method functions (`Get`, `List`, `Create`, `Update`, `Patch`, `SubResourceUpdate`,
  …) called instead of the underlying client. The package's own `doc.go` still says "This client
  does not have a way to inject specific errors"; that warning predates the interceptor package and
  is stale.

```go
c := fake.NewClientBuilder().
	WithScheme(scheme).
	WithObjects(node).
	WithStatusSubresource(&gpuv1.GPUNode{}).
	WithInterceptorFuncs(interceptor.Funcs{
		// Make the first status update fail with a Conflict, to exercise retry logic.
		SubResourceUpdate: func(ctx context.Context, cl client.Client, sub string,
			obj client.Object, _ ...client.SubResourceUpdateOption) error {
			if firstCall.CompareAndSwap(false, true) {
				return apierrors.NewConflict(
					schema.GroupResource{Group: "gpu.example.com", Resource: "gpunodes"},
					obj.GetName(), errors.New("object was modified"))
			}
			return cl.Status().Update(ctx, obj)
		},
	}).
	Build()
```

**What it genuinely still misses**, per its own `doc.go` and the absence of the relevant machinery:
no OpenAPI/CRD schema validation (a field your CRD would reject is accepted silently),
`metadata.Generation` does not behave like the real thing, no admission or mutating webhooks, no
API-server defaulting, and — because there is no kube-controller-manager — no garbage collection,
so deleting an owner does not cascade to its owned objects.

**Why the maintainers still say "prefer envtest."** The controller-runtime FAQ is explicit: the
fake client exists, "but we generally recommend using `envtest.Environment` to test against a real
API server. In our experience, tests using fake clients gradually re-implement poorly-written
impressions of a real API server, which leads to hard-to-maintain, complex test code." The same FAQ
gives the two structural rules worth internalising: **spin up a real API server rather than mocking
one**, and **assert that the state of the world is what you expect, not that a particular sequence
of API calls was made** — so you can refactor the controller's internals without rewriting tests.

Note also the frequently-cited comment in `kubernetes/client-go`'s `examples/fake-client/main_test.go`
— "the fake client isn't designed to work with informer. It doesn't support resource version." That
is true, and it is about a **different type**: `k8s.io/client-go/kubernetes/fake`, the fake
*clientset*, not controller-runtime's fake *client*. Conflating them is a common error; the
consequence there is that writes between an informer's initial LIST and its watch being established
are silently lost.

### envtest: what it starts, and what is not there

```
 ENVTEST WIRING
 ═══════════════════════════════════════════════════════════════════════

  go test ./internal/controller/
       │
       │ TestMain / SynchronizedBeforeSuite
       ▼
  envtest.Environment{ CRDDirectoryPaths: []string{"config/crd/bases"} }
       │  .Start()  ──────────────────────────────────────────┐
       │                                                       │
       │  binaries resolved from KUBEBUILDER_ASSETS            │
       │  (default /usr/local/kubebuilder/bin), populated by:  │
       │     setup-envtest use 1.31.x -p path                  │
       ▼                                                       ▼
  ┌──────────────┐   --etcd-servers      ┌──────────┐   fork/exec, random ports
  │ kube-apiserver│◀────────────────────▶│   etcd   │   start timeout 20s default
  │  (real!)      │                       │ (real!)  │   (KUBEBUILDER_CONTROLPLANE_
  └───────┬───────┘                       └──────────┘    START_TIMEOUT)
          │ *rest.Config
          ▼
  ┌────────────────────────┐        ┌────────────────────────────┐
  │ k8sClient (real client)│        │ ctrl.NewManager(cfg, ...)  │
  │  → Create/Get/Update   │        │   + your Reconciler        │
  └────────────────────────┘        │   go mgr.Start(ctx)        │
                                    └────────────────────────────┘
                                          ▲ informer watch → workqueue → Reconcile
                                          │ (ASYNCHRONOUS — this is why you poll)

  WHAT IS *NOT* RUNNING
    ✗ kubelet                 → Pods are created but never run; no status from a node
    ✗ kube-scheduler          → Pods stay Pending forever, .spec.nodeName never set
    ✗ kube-controller-manager → no garbage collection (ownerReferences do NOT cascade),
                                no ReplicaSet/Deployment/Job controllers, no
                                ServiceAccount token or default-SA creation
    ✗ CNI, CSI, cloud provider
```

Environment variables that control it (from `pkg/envtest/server.go`): `KUBEBUILDER_ASSETS` (the
binary directory), `USE_EXISTING_CLUSTER=true` (point the same suite at a real cluster),
`KUBEBUILDER_CONTROLPLANE_START_TIMEOUT` / `_STOP_TIMEOUT` (both default **20 s**), and
`KUBEBUILDER_ATTACH_CONTROL_PLANE_OUTPUT=true` to see apiserver/etcd logs when a suite will not
start — which is the first thing to try when envtest hangs.

The two consequences that trip people up in a GPU-fleet controller specifically: **a Pod you create
in envtest never becomes Running**, so a reconciler that waits for `.status.phase == Running` will
time out; and **deleting a parent does not delete children**, so a finalizer/ownership test that
passes on a real cluster fails here. Both are absences of controllers, not bugs. Write the test
against what the API server itself does — validation, defaulting, conflict, watch delivery — and
push node-behaviour assertions to rung 4.

### Async assertions

Because envtest exercises a real, asynchronous loop — write → watch event → workqueue → `Reconcile`
→ write — an assertion placed immediately after `Create` races the reconciler. It usually passes on
a laptop and fails on a loaded CI runner. This is the single most common source of flaky controller
tests.

```go
gomega.Eventually(func(g gomega.Gomega) {
	var node gpuv1.GPUNode
	g.Expect(k8sClient.Get(ctx, key, &node)).To(gomega.Succeed())
	g.Expect(node.Status.Phase).To(gomega.Equal(gpuv1.PhaseReady))
	g.Expect(node.Status.AllocatableGPUs).To(gomega.Equal(int32(8)))
}, "10s", "250ms").Should(gomega.Succeed())
```

Know the defaults, because they are short: Gomega's `Eventually` defaults to a **1 s** timeout with
a **10 ms** polling interval, and `Consistently` to **100 ms** duration with **10 ms** polling
(`internal/duration_bundle.go`). One second is not enough for an envtest reconcile round trip, so
**always pass explicit durations**, or set `gomega.SetDefaultEventuallyTimeout(10*time.Second)`
once in the suite.

Use both matchers deliberately: `Eventually` proves the controller *converges*; `Consistently`
proves it *stops* — the classic catch for a reconciler that flip-flops a status field forever
because two branches disagree.

controller-runtime ships `pkg/envtest/komega` to remove the boilerplate:

```go
komega.SetClient(k8sClient)
komega.SetContext(ctx)

gomega.Eventually(komega.Object(&node)).Should(
	gomega.HaveField("Status.Phase", gpuv1.PhaseReady))
```

And note that inside envtest you should poll rather than reach for `synctest` — the bubble's clock
cannot advance while a goroutine is blocked on a socket, and every envtest client call is a socket
read.

## Perspectives

**Developer view.** Table-driven tests are Go's `@pytest.mark.parametrize`, minus the decorator —
and the absence is the feature. The table is a slice literal: inspectable in a debugger, diffable
in review, greppable, and constructible at runtime if you need it. There is no framework layer
translating it into test identities; `t.Run(tt.name, ...)` *is* the whole mechanism, in plain sight.
The cost of that honesty is that nothing is done for you: no automatic fixtures, no automatic
isolation, no automatic parallelism. Every one of those is a line you write, which is exactly why
`t.Cleanup` and per-case fixtures matter so much.

**Reviewer/hiring view.** An experienced Go reviewer reads the test file before the source file.
Google's public standard makes the expectation concrete: reviewers must ask whether the tests will
actually fail when the code breaks, whether they will start producing false positives as the code
changes, whether each test makes simple and useful assertions — and it adds the line most people
forget, that *"tests are also code that has to be maintained. Don't accept complexity in tests just
because they aren't part of the main binary."* A 300-line test helper with its own DSL is a review
finding, not a sign of rigour.

**Systems/fidelity view.** Choosing a rung on the fidelity ladder is a design decision with a
runtime cost. Rung 1 is microseconds and runs on every save. Rung 2 is milliseconds and runs on
every save. Rung 3 pays a one-time ~2–20 s control-plane start per package and runs in CI. If your
whole suite is on rung 3, your feedback loop is minutes and you will stop running it; if none of it
is, you will ship a CRD whose schema rejects the object your controller writes. The split is the
skill.

**Failure-mode view.** The tests you did not write have a shape. Fake-client-only suites miss
schema, webhook, and cascade behaviour. Serial suites miss races. Suites without `goleak` miss the
leak that OOMKills the pod on day four. Suites with `time.Sleep` are flaky, and flaky suites get
retried until green, which converts a real intermittent bug into invisible noise — the worst
outcome of all, because now the signal exists and is being deliberately discarded.

## Real-world use cases

- **[Google — "What to look for in a code review"](https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md)** —
  Google's public code-review standard lists ten review dimensions, of which **Tests** is one. Its
  instructions are specific: ask for the appropriate level of test in the same CL as the production
  code; verify the tests are "correct, sensible, and useful," since "tests do not test themselves…
  a human must ensure that tests are valid"; ask whether they will fail when the code is broken and
  whether they will produce false positives as the code changes; and refuse complexity in tests just
  because they are not part of the main binary. *What it shows:* the mental model your test file has
  to survive in any staff-level review, written down by the organisation that invented most of Go's
  testing idioms.

- **[kubernetes-sigs/controller-runtime `FAQ.md`](https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md)** —
  official maintainer guidance, and unusually blunt: prefer `envtest.Environment` over the fake
  client, because "tests using fake clients gradually re-implement poorly-written impressions of a
  real API server, which leads to hard-to-maintain, complex test code." It also prescribes asserting
  final state over API-call sequences, and warns that "any time you're interacting with the API
  server, changes may have some delay between write time and reconcile time" — the one-sentence
  justification for `Eventually`. *What it shows:* the fidelity-ladder call, from the people who
  maintain both rungs.

- **[kubernetes/client-go — fake-client example](https://github.com/kubernetes/client-go/blob/master/examples/fake-client/main_test.go)** —
  a maintainer-level admission in a code comment: "The fake client doesn't support resource version.
  Any writes to the client after the informer's initial LIST and before the informer establishing the
  watcher will be missed by the informer… it's encouraged to use a real client in an
  integration/E2E test if you need to test complex behavior with informer/controllers." The example
  works around it by blocking on a `watchReactor` until the watcher has started. *What it shows:*
  where a fake's fidelity runs out, documented in the source — and a caution, since this is
  client-go's fake *clientset*, not controller-runtime's fake *client*, which does track
  `resourceVersion`.

- **Go 1.24 → 1.25, `testing/synctest`.** Shipped experimentally in Go 1.24 behind
  `GOEXPERIMENT=synctest` with `synctest.Run`, then promoted to general availability in Go 1.25 with
  a revised API (`synctest.Test(t, f)`), the old form kept only under the experiment flag and slated
  for removal in Go 1.26. *What it shows:* the Go team treating "concurrent and time-dependent code
  is untestable without a fake clock" as a standard-library problem rather than a
  bring-your-own-`Clock`-interface problem — and a reminder to check the Go version before copying
  a `synctest` example off the internet, because the two APIs are not compatible.

## Worked example

A focused unit test for the exporter's collector using a fake `CostSource` — no network, no cluster,
deterministic, race-clean, leak-checked.

```go
package exporter_test

import (
	"context"
	"errors"
	"maps"
	"testing"

	"go.uber.org/goleak"

	"github.com/tarhov/gpu-cost-exporter/internal/exporter"
)

func TestMain(m *testing.M) {
	goleak.VerifyTestMain(m) // no goroutine may outlive this package's tests
}

// fakeSource is a hand-written double for exporter.CostSource. No mock library.
type fakeSource struct {
	rates map[string]float64
	err   error
}

var _ exporter.CostSource = (*fakeSource)(nil) // compile-time contract check

func (f *fakeSource) Rates(ctx context.Context) (map[string]float64, error) {
	if err := ctx.Err(); err != nil {
		return nil, err // honour cancellation, like the real source does
	}
	return f.rates, f.err
}

var errBoom = errors.New("boom")

func TestCollect(t *testing.T) {
	tests := []struct {
		name    string
		rates   map[string]float64
		srcErr  error
		usage   []exporter.GPUUsage
		want    map[string]float64
		wantErr error
	}{
		{
			name:  "two pools summed by rate",
			rates: map[string]float64{"a100": 2.5, "h100": 4.0},
			usage: []exporter.GPUUsage{
				{Pool: "a100", Hours: 10},
				{Pool: "h100", Hours: 5},
				{Pool: "a100", Hours: 2},
			},
			want: map[string]float64{"a100": 30.0, "h100": 20.0}, // (10+2)*2.5, 5*4.0
		},
		{
			name:  "empty usage yields empty cost, not nil",
			rates: map[string]float64{"a100": 2.5},
			usage: nil,
			want:  map[string]float64{},
		},
		{
			name:    "unknown pool is a terminal error",
			rates:   map[string]float64{},
			usage:   []exporter.GPUUsage{{Pool: "a100", Hours: 10}},
			wantErr: exporter.ErrUnknownPool,
		},
		{
			name:    "source error propagates wrapped",
			srcErr:  errBoom,
			usage:   []exporter.GPUUsage{{Pool: "a100", Hours: 1}},
			wantErr: errBoom,
		},
		{
			name:  "zero hours contributes zero, not a missing key",
			rates: map[string]float64{"a100": 2.5},
			usage: []exporter.GPUUsage{{Pool: "a100", Hours: 0}},
			want:  map[string]float64{"a100": 0},
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel() // safe: each case builds its OWN fake below, nothing is shared

			src := &fakeSource{rates: tt.rates, err: tt.srcErr}
			got, err := exporter.Collect(t.Context(), src, tt.usage)

			if tt.wantErr != nil {
				if !errors.Is(err, tt.wantErr) {
					t.Fatalf("Collect() error = %v, want errors.Is(..., %v)", err, tt.wantErr)
				}
				return
			}
			if err != nil {
				t.Fatalf("Collect() unexpected error: %v", err)
			}
			if !maps.Equal(got, tt.want) {
				t.Errorf("Collect() = %v, want %v", got, tt.want)
			}
		})
	}
}

// TestCollect_HonoursCancellation is a separate test, not a table row:
// it asserts a different property and needs different setup.
func TestCollect_HonoursCancellation(t *testing.T) {
	t.Parallel()

	ctx, cancel := context.WithCancel(t.Context())
	cancel() // already cancelled before the call

	src := &fakeSource{rates: map[string]float64{"a100": 1}}
	_, err := exporter.Collect(ctx, src, []exporter.GPUUsage{{Pool: "a100", Hours: 1}})
	if !errors.Is(err, context.Canceled) {
		t.Fatalf("Collect() error = %v, want context.Canceled", err)
	}
}
```

Read the shape:

- **Every case owns its fake.** The table stores *data* (`rates`, `srcErr`), and the fake is
  constructed inside the closure. That is what makes `t.Parallel()` safe. Storing
  `src: &fakeSource{...}` in the table would work here but establishes a habit that breaks the
  moment a fake becomes stateful.
- **Error assertions use `errors.Is`, not `err != nil`.** `wantErr bool` tells you a test failed;
  `errors.Is(err, exporter.ErrUnknownPool)` tells you it failed *for the right reason*. That is the
  Google criterion — will the test fail when the code is broken, and only then? — applied to a
  single line.
- **Edge cases are rows**: nil input, empty output-not-nil, zero-valued input, missing lookup,
  upstream error. The "empty yields empty map, not nil" and "zero hours yields a present key with
  value 0" rows exist because both distinctions have burned Prometheus exporters — a missing series
  and a zero-valued series mean different things to `rate()`.
- **Cancellation is its own test.** It asserts a different property with different setup; forcing it
  into the table would need a `ctxFunc func() context.Context` field, which is logic in the table.
- **`t.Context()`** gives each case a context cancelled before cleanup, so nothing the collector
  starts can outlive the test — which is what makes `goleak.VerifyTestMain` pass.

Run it:

```
$ go test -race -covermode=atomic -coverprofile=cover.out ./internal/exporter/
ok      github.com/tarhov/gpu-cost-exporter/internal/exporter    1.213s  coverage: 88.9% of statements

$ go tool cover -func=cover.out | grep -v 100.0%
internal/exporter/collect.go:31:   Collect          88.9%
internal/exporter/metrics.go:19:   MustRegister      0.0%
total:                             (statements)     88.9%
```

The honest read: `MustRegister` is Prometheus wiring exercised by the binary, not the logic; the
missing 11.1% of `Collect` is the `ctx.Err()` early-return path, which `TestCollect_HonoursCancellation`
does cover — so if it still shows as uncovered, the branch is unreachable and should be deleted.
That is the loop to run: uncovered branch → justify it or delete it.

## Practice

In `gpu-cost-exporter`, write the test suite for your aggregation and collection logic.

1. **Table-driven `TestCollect`** covering, at minimum: empty input, a `nil` usage slice, duplicate
   `Pool` labels that must sum, a pool with no matching rate (error branch), a source error, and a
   zero-hours row. Assert errors with `errors.Is` against exported sentinels, never `wantErr bool`.
2. **A `fakeSource`** implementing your `CostSource` interface — a hand-written struct, no mock
   library — with a `var _ CostSource = (*fakeSource)(nil)` assertion and a `ctx.Err()` check so it
   honours cancellation. Build it per case so `t.Parallel()` is safe.
3. **A concurrency test** for lesson 04's bounded collector: assert peak in-flight never exceeds
   `limit`, and that a cancelled context returns promptly with `context.Canceled`.
4. **`goleak.VerifyTestMain(m)`** in a `TestMain` for the package.
5. **One benchmark** `BenchmarkCollect` over ~10k synthetic usage rows using `for b.Loop()` and
   `b.ReportAllocs()`. Record the `allocs/op` figure in a comment; it is your regression baseline.
6. **One fuzz target** `FuzzParseCostExport` over whatever parses your cost CSV/JSON export, seeded
   with `f.Add` *and* from `testdata/fuzz/`, asserting a round-trip property rather than a fixed
   output.
7. **One `testing/synctest` test** (Go 1.25+) for your retry/backoff path, asserting the exact
   elapsed fake time — no `time.Sleep`, no tolerance window. If you are on an older toolchain, note
   in a comment what you would assert instead.

**Acceptance:**

- `go test -race -covermode=atomic -coverprofile=cover.out ./...` is green.
- `go tool cover -func=cover.out` shows ~70%+ on the business-logic package **and** you can name
  every uncovered function and say why it is acceptable.
- No test touches a real network or cluster.
- `go test -bench=. -benchmem ./internal/exporter` reports `allocs/op`.
- `go test -fuzz=FuzzParseCostExport -fuzztime=30s ./internal/exporter` finds no crash; if it does,
  commit the reproducer from `testdata/fuzz/` and fix it.
- `go vet ./...` is clean.

Full deliverable spec: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md).

## Common pitfalls

1. **`wantErr bool` instead of a sentinel.** *Symptom:* a test stays green while the code returns
   the wrong error — a parse failure where a not-found was expected. *Mechanism:* `(err != nil) !=
   tt.wantErr` only asserts the shape of the return, not its meaning. *Correction:* export
   sentinels, assert with `errors.Is`, or with `errors.As` for typed errors carrying fields.

2. **Sharing a mutable fixture across table cases and then calling `t.Parallel()`.** *Symptom:*
   nondeterministic failures that vanish on rerun and only appear in CI. *Mechanism:* parallel
   siblings resume together when the parent's body returns, so they genuinely race on the shared
   map, buffer, or counter — and `-race` only reports it when the writes happen to overlap in that
   run. *Correction:* construct the fixture inside the `t.Run` closure; keep only data in the table.

3. **Asserting immediately after `Create`/`Update` in an envtest suite.** *Symptom:* passes locally,
   flakes in CI. *Mechanism:* the reconcile path is write → watch event → workqueue → `Reconcile` →
   write, all asynchronous; your assertion can run before the first `Reconcile`. *Correction:*
   `gomega.Eventually` with explicit durations (the 1 s default is too short for envtest), plus
   `Consistently` where you need to prove the controller settles rather than oscillates.

4. **Trusting a fake-client suite for schema, webhook, or cascade behaviour.** *Symptom:* green
   tests, then `is invalid: spec.foo: Required value` in the cluster. *Mechanism:* the fake client
   performs no OpenAPI validation, runs no admission webhooks, and has no
   kube-controller-manager, so ownerReference cascades never happen. It *does* model
   `resourceVersion` and conflicts — that part of the folklore is out of date — but not these.
   *Correction:* push anything schema-, webhook-, or GC-dependent to envtest.

5. **`time.Sleep` in a test.** *Symptom:* a suite that is both slow and flaky, and gets "fixed" by
   raising the sleep. *Mechanism:* you have encoded an assumption about a loaded CI runner's
   scheduling into an assertion. *Correction:* `testing/synctest` for anything clock-driven,
   `Eventually` for anything genuinely asynchronous and external. Neither ever needs a sleep.

6. **Reaching for a generated mock out of habit.** *Symptom:* a 200-line generated file and a test
   that breaks whenever you add a method to the interface. *Mechanism:* generated mocks bind to the
   interface's whole shape and to call sequences. *Correction:* default to a fake for state-based
   stubbing; reserve generated mocks for genuine interaction assertions across multiple methods.

7. **Treating a coverage percentage as a goal.** *Symptom:* tests of getters and wiring, error paths
   untested. *Mechanism:* statement coverage counts statements, and trivial code has the most
   statements per unit of risk. *Correction:* read the `-func` report top to bottom, and for each
   entry below 100% either name the branch and justify it or write the test.

8. **Calling `require.NoError` from a spawned goroutine.** *Symptom:* a test that passes while the
   goroutine silently died. *Mechanism:* `require` calls `t.FailNow` → `runtime.Goexit`, which
   terminates *that* goroutine without failing the test. *Correction:* send the error back over a
   channel and assert on the test's own goroutine; `go vet`'s `testinggoroutine` analyser catches
   the common cases.

## Self-check

**(a)** When do you write a hand-written fake vs. a generated mock?

**Answer:** Default to a **fake** — a small hand-written struct satisfying a narrow,
consumer-defined interface — for stubbing return values and errors. It is less code, it reads like
Go, and it survives refactors because it only implements what the interface declares. Reach for a
**generated mock** (`go.uber.org/mock`, `mockery`) only when the assertion is genuinely about
*interaction*: exact arguments, call counts, or ordering across multiple methods, e.g. "Create, then
Update, and never Delete." **Fakes verify state; mocks verify behaviour.** Note that monkeypatching
— the Python default — is not available at all in Go: package functions are not reassignable
attributes and method sets are fixed at compile time, so substitution must happen through an
interface seam you designed.

**(b)** What does `t.Parallel` change about shared state and (pre-1.22) loop-variable capture?

**Answer:** `t.Parallel()` pauses the subtest: it appends itself to the parent's `sub` list,
signals the parent to continue, and blocks on the parent's `barrier` channel. When the parent's body
returns, `tRunner` closes that barrier and **all parallel siblings resume at once**, capped at
`-test.parallel` (default `GOMAXPROCS`). So parallel siblings run concurrently with each other, and
any state they share — a map, a counter, one fake instance — becomes a real data race that `-race`
will flag when the writes overlap. Parallel subtests must be independent, with their own fixtures.

Pre-Go 1.22, the range variable was a single variable reused across iterations, so a closure that
captured `tt` and then paused at `t.Parallel()` typically observed the *last* case's value once
the barrier opened; the fix was `tt := tt` to shadow it. **Go 1.22 changed `for` semantics so each
iteration gets a fresh variable**, when the module's `go` directive is 1.22 or later — the shadow
line is unnecessary in new code and harmless in old. Also worth knowing: `t.Setenv`, `t.Chdir`, and
`goleak.VerifyNone` are all incompatible with `t.Parallel`, the first two by panicking.

**(c)** How do you test a `Reconcile` without a real cluster, and when is that not enough?

**Answer:** Build a controller-runtime **fake client**
(`fake.NewClientBuilder().WithScheme(s).WithObjects(...).WithStatusSubresource(...).Build()`), inject
it into the reconciler, call `Reconcile(ctx, req)` directly, and assert on the resulting in-memory
object state. Microseconds to milliseconds, no cluster. It models more than its reputation suggests:
it increments `resourceVersion`, returns `apierrors.NewConflict` on a stale one, models the status
subresource when you opt in, supports field indexes via `WithIndex`, and supports error injection via
`WithInterceptorFuncs`.

It is not enough when the behaviour under test belongs to the API server rather than to your code:
**OpenAPI/CRD schema validation**, **API-server defaulting**, **admission or mutating webhooks**,
`metadata.Generation` semantics, and — because no kube-controller-manager runs — **garbage
collection via ownerReferences**. For those, use **envtest**, which forks a real `kube-apiserver`
and `etcd` (binaries fetched by `setup-envtest`, located via `KUBEBUILDER_ASSETS`) with no kubelet,
scheduler, or controller-manager. Run the manager against it and assert asynchronously with
`gomega.Eventually` and explicit durations — the default 1 s timeout is too short for a reconcile
round trip. Remember what envtest still lacks: Pods never become Running and owner deletions never
cascade, because the controllers that would do those things are not there.

**(d)** Why does a staff-level interviewer ask "what's not covered?" instead of "what's your coverage
number?"

**Answer:** Because `go test -cover` reports **statement** coverage, and statements are not risk. A
percentage cannot distinguish `main`'s wiring from the retry-on-conflict branch, and it is trivially
raised by testing getters while skipping error paths. It also over-credits: a function containing
`if a && b` reaches 100% statement coverage from a single input, having exercised one of four
condition combinations. Naming the specific uncovered branch and justifying it — "`main` is wiring";
"that `default:` case is unreachable and I should delete it"; "the decode-error path is a real gap,
here's the ticket" — demonstrates that you have read the report rather than the summary line, which
is the actual skill being probed.

**(e)** You need to test that a scrape retries three times with 100 ms/200 ms/400 ms backoff and then
succeeds. How do you test it in under a millisecond, deterministically?

**Answer:** `testing/synctest` (generally available in Go 1.25). Wrap the test body in
`synctest.Test(t, func(t *testing.T) { ... })`. Inside the bubble the `time` package uses a fake
clock starting at midnight UTC 2000-01-01, and **time advances only when every goroutine in the
bubble is durably blocked** — at which point it jumps instantly to the next timer deadline. So the
retrier's `time.Sleep(100ms)` costs no wall time; the clock simply moves. Because the clock is a
data structure rather than a measurement, you can assert `time.Since(start) == 700*time.Millisecond`
**exactly**, with no tolerance window, and `synctest.Wait()` gives you a deterministic barrier that
returns once all other bubble goroutines have settled.

The constraints: do not use the network inside a bubble (a socket read is not durable blocking, so
the clock never advances and the bubble deadlocks), do not call `t.Parallel` or `t.Run` inside one,
and remember that locking a `sync.Mutex` is *not* durable blocking either. Also check your Go
version — Go 1.24 shipped a different API (`synctest.Run`) behind `GOEXPERIMENT=synctest`, removed
in 1.26.

**(f)** Your CI reruns the suite and it goes green. What have you just done?

**Answer:** Converted a real intermittent bug into invisible noise. A flake is a test that is
sometimes right; retrying until green discards the run where it was right. In Go the three usual
causes are all diagnosable: a **data race** (run with `-race` and `GORACE=halt_on_error=1`, and add
`t.Parallel()` so interleavings actually happen), a **timing assumption** (a `time.Sleep` or a
too-short `Eventually` — replace with `synctest` or explicit durations), or **shared state between
tests** (a package-level variable, a shared fixture, a `t.Setenv` in a parallel test). Reproduce it
with `go test -count=50 -race -run TestFlaky ./...`, and add `-shuffle=on` to catch order
dependence between tests.

## Connections & what's next

This lesson leans directly on lesson 04's `-race` detector, `context` propagation, and goroutine-leak
detection — the concurrency bugs that testing exists to catch, now wired into a suite that catches
them automatically. It leans on lesson 03 harder still: every fake here exists because the dependency
was a small, consumer-defined interface. It is also a direct rehearsal for lesson 09, **Controller
primer (CRD · reconcile · envtest)**, which puts the fidelity ladder to work on a real reconciler:
nothing here is throwaway.

Next: lesson 06, **[Modules and Project Layout](06-modules-and-layout.md)**, moves from "is the code
correct" to "is the module shaped, versioned, and packaged the way the ecosystem expects" — where the
`internal/` boundary your black-box tests have been respecting becomes a compiler-enforced API
contract, and where the `testdata/` and `go.sum` conventions you have been relying on get their
explanation.

## References & further reading

**Primary sources**

1. [`testing` package documentation](https://pkg.go.dev/testing) — the authoritative reference for
   `T`, `B`, `F`, `M`, and every method in the table above, plus the package-level docs on fuzzing,
   subtests, and `TestMain`. Read for exact signatures and the version each was added in.
2. [`go help test` / `go help testflag`](https://pkg.go.dev/cmd/go#hdr-Testing_flags) — the flag
   reference: `-covermode` defaults (`set`, or `atomic` under `-race`), `-coverpkg`, `-fuzztime`,
   `-fuzzminimizetime` (default 60 s), `-shuffle`, and the exact list of cacheable flags plus the
   `-count=1` idiom.
3. [The Go Blog — "Using Subtests and Sub-benchmarks"](https://go.dev/blog/subtests) — the canonical
   explanation of `t.Run`, the naming/`-run` matching rules, and the parallel-subtest grouping idiom.
4. [`testing/synctest` package documentation](https://pkg.go.dev/testing/synctest) — the bubble, the
   fake clock's 2000-01-01 epoch, the exact list of operations that count as *durably blocked*, and
   the isolation rules. The single most useful new testing API for concurrent Go.
5. [Go 1.24 release notes](https://go.dev/doc/go1.24) — `testing.B.Loop`, `T.Context`, `T.Chdir`,
   and the original experimental `synctest.Run` API.
6. [Go 1.25 release notes](https://go.dev/doc/go1.25) — `testing/synctest` promoted to general
   availability with the `synctest.Test` API (and the old form scheduled for removal in 1.26), plus
   `T.Attr` and `T.Output`.
7. [Go Fuzzing documentation](https://go.dev/doc/fuzz/) — corpus layout (`testdata/fuzz/FuzzXxx`),
   the fuzzable type list, minimisation, and how found crashes become permanent seeds.
8. [`sigs.k8s.io/controller-runtime/pkg/client/fake`](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client/fake) —
   the builder API and the package's own limitations list. *Correction to the previous version of
   this lesson:* the fake client **does** track `resourceVersion` (seeded objects start at `"999"`,
   updates increment, mismatches return `Conflict`) and **does** support error injection via
   `WithInterceptorFuncs`; the `doc.go` warning about error injection predates that package.
9. [`sigs.k8s.io/controller-runtime/pkg/client/interceptor`](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client/interceptor) —
   `interceptor.Funcs`, the per-method override struct that makes fake-client error injection
   possible. Read the field list; it is the whole API.
10. [`sigs.k8s.io/controller-runtime/pkg/envtest`](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/envtest) —
    `Environment`, `CRDDirectoryPaths`, and the environment variables (`KUBEBUILDER_ASSETS`,
    `USE_EXISTING_CLUSTER`, the 20 s start/stop timeouts,
    `KUBEBUILDER_ATTACH_CONTROL_PLANE_OUTPUT`). Also `pkg/envtest/komega` for Gomega helpers.
11. [Kubebuilder — Configuring envtest](https://book.kubebuilder.io/reference/envtest.html) — how
    `setup-envtest` fetches the `kube-apiserver`/`etcd`/`kubectl` binaries and how to wire the suite.
    Read for the exact commands your Makefile will run.
12. [kubernetes-sigs/controller-runtime `FAQ.md`](https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md) —
    maintainer guidance on fake client vs envtest, asserting final state over API-call sequences, and
    the reminder that write-to-reconcile has real delay.
13. [Gomega documentation](https://onsi.github.io/gomega/) — `Eventually`, `Consistently`, and the
    matcher set. The defaults that matter (1 s / 10 ms and 100 ms / 10 ms) live in
    `internal/duration_bundle.go`.
14. [`go.uber.org/goleak`](https://github.com/uber-go/goleak) — `VerifyNone`, `VerifyTestMain`, the
    default filter list, `IgnoreTopFunction`/`IgnoreCurrent`, and `RunOnFailure`.

**Real-world engineering blogs**

15. [Google eng-practices — "What to look for in a code review"](https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md) —
    what it shows: tests are one of ten explicit review dimensions, with the reviewer told to check
    that they will fail when the code breaks, will not become false positives, make simple and useful
    assertions, and are not more complex than the production code they guard.
16. [kubernetes/client-go — fake-client example](https://github.com/kubernetes/client-go/blob/master/examples/fake-client/main_test.go) —
    what it shows: a maintainer documenting, in a comment, that the fake *clientset* does not support
    resource versions and therefore loses writes between an informer's LIST and its watch — and
    working around it by blocking on a watch reactor. A precise illustration of a fake's fidelity
    edge, and of why the controller-runtime fake client (a different type) must not be judged by it.

**Deeper dives**

17. **100 Go Mistakes and How to Avoid Them — Harsanyi** — <https://100go.co>. *What-for:* the
    testing chapter catalogues the traps here — not using table-driven tests, forgetting
    `t.Parallel`, not using `-race`, misusing `testing.Short`, benchmark micro-optimisation errors.
    *How:* skim the testing and benchmarking chapters now; return when a benchmark reports something
    implausible. *Why:* every entry is a real review comment you would rather receive from a book.
