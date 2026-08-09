---
lesson: "01.5"
title: "Testing in Go"
module: "01"
concept: "Testing in Go"
status: not-started
est_time: "12h"
artifacts: []
---

# 01.5 · Testing in Go

> **Concept.** Table-driven tests, fakes over mocks, `-race`, and the controller testing story (envtest + fake client).
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Why this matters
Take-home and on-site rounds for platform roles are won in the test file, not the source file. A reviewer scanning your PR reads the tests first — they reveal whether you understand determinism, concurrency, and boundaries. Every serious controller repo (Kubernetes, cert-manager, Karpenter) ships table-driven tests, `envtest`, and `-race` in CI; matching that idiom is table stakes for a CoreWeave/NVIDIA-class Go hire. Get this right and you look senior on day one.

## From Python to Go
No pytest, no fixtures, no magic collection. Testing is a first-class part of the toolchain and the standard library — `go test` compiles a real binary, `testing.T` is passed explicitly into every test function, and there is no assertion DSL baked in (you compare values and call `t.Errorf`). The `unittest.mock`/`MagicMock` habit does **not** carry over: idiomatic Go fakes a small interface with a hand-written struct, and monkeypatching is essentially impossible because there's no attribute to reassign. Parametrization isn't a decorator — it's a slice of structs you loop over. And `-race` is a real, cheap, always-on tool with no Python equivalent worth the name.

## Core notes

**Test file mechanics.** A test lives in `foo_test.go`, function `TestXxx(t *testing.T)`, in the same package (white-box) or `package foo_test` (black-box, only the exported API). Run with `go test ./...`.

**Table-driven tests + subtests.** The dominant Go idiom. One test, a slice of cases, `t.Run` per case so failures name themselves:

```go
func TestAggregate(t *testing.T) {
	tests := []struct {
		name string
		in   []Sample
		want map[string]float64
	}{
		{"empty", nil, map[string]float64{}},
		{"single", []Sample{{Pool: "a100", Cost: 3.0}}, map[string]float64{"a100": 3.0}},
		{"dupes sum", []Sample{{Pool: "a100", Cost: 1}, {Pool: "a100", Cost: 2}}, map[string]float64{"a100": 3.0}},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := Aggregate(tt.in)
			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("Aggregate(%v) = %v, want %v", tt.in, got, tt.want)
			}
		})
	}
}
```

`t.Run` gives you a named subtest node — failures print `TestAggregate/dupes_sum`, and you can target one with `go test -run TestAggregate/dupes_sum`.

**`t.Parallel` and the loop-variable change.** `t.Parallel()` signals a subtest can run concurrently with other parallel siblings; the parent waits for all of them at the end. This surfaces shared-state bugs — two parallel cases mutating a shared map will trip `-race`. Historically the classic trap was **loop-variable capture**: pre-Go 1.22, `tt` was one variable reused across iterations, so a parallel closure often saw the *last* case. You had to write `tt := tt` to shadow it. **Go 1.22 changed loop semantics so each iteration gets a fresh `tt`** — the `tt := tt` dance is no longer needed when your `go.mod` declares `go 1.22` or later. Know both facts: interviewers ask, and older code still has the shadow.

```go
for _, tt := range tests {
	t.Run(tt.name, func(t *testing.T) {
		t.Parallel() // Go 1.22+: tt is per-iteration, safe to capture
		got := Aggregate(tt.in)
		// ...
	})
}
```

**Fakes over mocks.** Depend on a small interface, pass a hand-written fake in tests. This is the single most important Go testing idiom and the biggest delta from Python.

```go
type CostSource interface {
	Rates(ctx context.Context) (map[string]float64, error)
}

type fakeSource struct {
	rates map[string]float64
	err   error
}

func (f fakeSource) Rates(ctx context.Context) (map[string]float64, error) {
	return f.rates, f.err
}
```

Reach for a **generated mock** (`gomock`/`mockery`) only when you must assert on *call order or exact arguments across many methods* — e.g. verifying a client calls `Create` then `Update` then `Delete`. For return-value stubbing, a fake is less code, refactor-proof, and far easier to read.

**testify, judiciously.** `require`/`assert` cut boilerplate: `require.NoError(t, err)` stops the test (like a fatal), `assert.Equal(t, want, got)` continues. Use `require` for preconditions (a nil-check that would panic downstream) and `assert` for the checks you want all of. Don't let it become a crutch — plain `if got != want { t.Errorf(...) }` is perfectly idiomatic and the stdlib uses nothing else.

**Golden files.** For large expected outputs (rendered Prometheus exposition text, generated YAML), store the expectation in `testdata/foo.golden` and gate regeneration behind a flag:

```go
var update = flag.Bool("update", false, "update golden files")

func TestRender(t *testing.T) {
	got := Render(metrics)
	golden := filepath.Join("testdata", "metrics.golden")
	if *update {
		os.WriteFile(golden, got, 0o644)
	}
	want, _ := os.ReadFile(golden)
	if !bytes.Equal(got, want) {
		t.Errorf("output differs from golden; run -update to refresh")
	}
}
```

`testdata/` is special: the Go tool ignores it for builds. Regenerate with `go test -run TestRender -update`, then diff the golden in review.

**`-race` and `-cover`.** `go test -race` instruments memory access and flags concurrent unsynchronized read/write — invaluable for exporters and controllers that run goroutines. `go test -cover` prints statement coverage; `-coverprofile=c.out` + `go tool cover -html=c.out` shows line-by-line. Aim for meaningful coverage of *business logic*, not 100% of glue. CI should run `go test -race -cover ./...`.

**Benchmarks.** `BenchmarkXxx(b *testing.B)` loops `b.N` times; the framework tunes `N`. Run with `go test -bench=. -benchmem`:

```go
func BenchmarkAggregate(b *testing.B) {
	in := makeSamples(10000)
	b.ResetTimer()
	for range b.N {
		_ = Aggregate(in)
	}
}
```

`b.ResetTimer()` excludes setup; `-benchmem` reports allocs/op, which is what you tune in hot exporter paths.

**Fuzzing (awareness).** Go 1.18+ has native fuzzing: `FuzzXxx(f *testing.F)`, seed with `f.Add`, run bodies via `f.Fuzz`. Reach for it on parsers and untrusted input (a label parser, a cost-CSV reader) to find panics and edge cases. `go test -fuzz=FuzzParse` runs it; a found crash is written to `testdata/fuzz/`. Know it exists and when to use it — you rarely need it for pure aggregation.

**How controllers are tested.** Two layers you must be able to name:

- **Unit — `controller-runtime` fake client.** `sigs.k8s.io/controller-runtime/pkg/client/fake` builds an in-memory client seeded with objects. You construct your `Reconciler` with it, call `Reconcile`, and assert on the resulting object state — no cluster, microsecond-fast. Great for reconcile logic and error branches.

```go
scheme := runtime.NewScheme()
_ = corev1.AddToScheme(scheme)
c := fake.NewClientBuilder().WithScheme(scheme).WithObjects(pod).Build()
r := &PodReconciler{Client: c}
_, err := r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
```

- **Integration — envtest.** `sigs.k8s.io/controller-runtime/pkg/envtest` starts a **real `kube-apiserver` and `etcd`** locally (binaries fetched via `setup-envtest`) with no kubelet/nodes. You get a genuine API server to run your manager against, exercising validation, defaulting, watches, and status subresources that the fake client fakes imperfectly. Slower; run in CI, not on every save. The fake client catches logic bugs; envtest catches "the API server actually rejects this" bugs.

## Worked example
A focused unit test for the exporter's collector using a fake `CostSource` — no network, deterministic, race-clean:

```go
package exporter

import (
	"context"
	"errors"
	"testing"
)

func TestCollect_Aggregation(t *testing.T) {
	tests := []struct {
		name    string
		src     CostSource
		usage   []GPUUsage
		want    map[string]float64
		wantErr bool
	}{
		{
			name:  "two pools summed by rate",
			src:   fakeSource{rates: map[string]float64{"a100": 2.5, "h100": 4.0}},
			usage: []GPUUsage{{Pool: "a100", Hours: 10}, {Pool: "h100", Hours: 5}, {Pool: "a100", Hours: 2}},
			want:  map[string]float64{"a100": 30.0, "h100": 20.0}, // (10+2)*2.5, 5*4.0
		},
		{
			name:    "unknown pool has no rate",
			src:     fakeSource{rates: map[string]float64{}},
			usage:   []GPUUsage{{Pool: "a100", Hours: 10}},
			wantErr: true,
		},
		{
			name:  "empty usage yields empty cost",
			src:   fakeSource{rates: map[string]float64{"a100": 2.5}},
			usage: nil,
			want:  map[string]float64{},
		},
		{
			name:    "source error propagates",
			src:     fakeSource{err: errors.New("boom")},
			usage:   []GPUUsage{{Pool: "a100", Hours: 1}},
			wantErr: true,
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			t.Parallel()
			got, err := Collect(context.Background(), tt.src, tt.usage)
			if (err != nil) != tt.wantErr {
				t.Fatalf("Collect() err = %v, wantErr %v", err, tt.wantErr)
			}
			if tt.wantErr {
				return
			}
			if len(got) != len(tt.want) {
				t.Fatalf("got %d pools, want %d", len(got), len(tt.want))
			}
			for pool, w := range tt.want {
				if got[pool] != w {
					t.Errorf("cost[%q] = %v, want %v", pool, got[pool], w)
				}
			}
		})
	}
}
```

Note the shape: fake injected per-case, edge cases (nil, empty, error, missing rate) are rows, `t.Parallel` proves no shared state, and every assertion names the pool that failed.

## Practice
In `gpu-cost-exporter`, write the test suite for your aggregation logic:

1. **Table-driven `TestCollect`** covering: empty input, `nil` usage slice, duplicate `Pool` labels that must sum, a pool with no matching rate (error branch), and a source error.
2. **A `fakeSource`** implementing your `CostSource` interface — hand-written struct, no mock library.
3. **One benchmark** `BenchmarkCollect` over ~10k synthetic usage rows with `b.ResetTimer()` and `-benchmem`.

**Acceptance:** `go test -race -cover ./...` is green, coverage of the business-logic package is ~70%+, no test touches a real network or cluster, and `go test -bench=. -benchmem` reports allocs/op.

## Self-check
1. **When do you write a hand-written fake vs. a generated mock?**
   **Answer:** Default to a **fake** — a small hand-written struct satisfying a narrow interface — for stubbing return values and errors; it's less code and survives refactors. Reach for a **generated mock** (gomock/mockery) only when you must assert on *interaction*: exact arguments, call counts, or ordering across multiple methods (e.g. Create-then-Update-then-Delete). Fakes verify *state*; mocks verify *behavior*.

2. **What does `t.Parallel` change about shared state and (pre-1.22) loop-variable capture?**
   **Answer:** It lets marked subtests run concurrently, so any shared mutable state (a shared map, counter, or the same fake instance mutated in place) becomes a data race that `-race` will flag — parallel tests must be independent. Pre-Go 1.22 the loop variable was reused across iterations, so a parallel closure captured a reference that all iterations shared and typically read the *last* case's value; the fix was `tt := tt` to shadow. Go 1.22 makes each iteration's variable fresh, removing the need for that shadow.

3. **How do you test a `Reconcile` without a real cluster?**
   **Answer:** For unit tests, build a `controller-runtime` **fake client** (`pkg/client/fake`) seeded with your input objects, inject it into the reconciler, call `Reconcile`, and assert on the resulting in-memory object state — microsecond-fast, no cluster. For higher fidelity, use **envtest**, which starts a real local `kube-apiserver` + `etcd` (no kubelet/nodes) so validation, defaulting, and status subresources behave like production; run that layer in CI.

## Resources
1. **Using Subtests and Sub-benchmarks** — https://go.dev/blog/subtests — the canonical explanation of `t.Run`, parallelism, and table structure. *Skim.* Why: it's the primary source for the idiom every reviewer expects.
2. **`testing` package docs** — https://pkg.go.dev/testing — authoritative reference for `T`, `B`, `F`, `-race`, `-cover`, `Parallel`, `Cleanup`. *Deep* (as reference). Why: you'll return to it for exact signatures and flags.
3. **Kubebuilder — Configuring envtest** — https://book.kubebuilder.io/reference/envtest.html — how the real apiserver+etcd harness is wired and fetched via `setup-envtest`. *Deep.* Why: directly maps to how you'll test the capstone controller.
4. **A Comprehensive Guide to Testing in Go (JetBrains)** — https://blog.jetbrains.com/go/2022/11/22/comprehensive-guide-to-testing-in-go/ — end-to-end tour: tables, fakes, coverage, fuzzing. *Skim.* Why: good consolidation with practical patterns.
