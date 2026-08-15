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
sources: 8
---
{% raw %}

# 01.5 · Testing in Go

> **Concept.** Table-driven tests, fakes over mocks, `-race`, and the controller testing story
> (envtest + fake client) — plus the fidelity ladder between them and the review culture that
> makes your test file the first thing a senior reviewer reads.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 04 gave you the concurrency primitives that make a controller work — goroutines, channels,
`context.Context` for cancellation, and `-race` as the tool that catches unsynchronized access.
That raises an obvious question: how do you *prove* concurrent code is correct, and how do you
test something that calls a live Kubernetes API without standing up a cluster for every `go test`
run? This lesson answers both. It closes the "write it" arc of the module — interfaces (03),
concurrency (04), testing (05) are the three lessons the module README says never to shortcut —
before lesson 06, **Modules and Project Layout**, shifts from "does the code work" to "how do you
package, version, and ship it as a real Go module."

## Why this matters

Take-home and on-site rounds for platform roles are won in the test file, not the source file. A
reviewer scanning your PR reads the tests first — they reveal whether you understand determinism,
concurrency, and boundaries. This isn't folklore: Google's own public code-review standard tells
reviewers to scrutinize tests explicitly — are they valid, will they catch real bugs, could they
produce false positives — as part of judging complexity in a change (see Real-world use cases,
below). Every serious controller repo (Kubernetes, cert-manager, Karpenter) ships table-driven
tests, `envtest`, and `-race` in CI; matching that idiom is table stakes for a CoreWeave/NVIDIA-class
Go hire. Get this right and you look senior on day one — get it wrong (a green suite that never
actually exercised the failure path) and you look senior for exactly as long as it takes someone
to ask "what happens if the API server rejects this?"

## What's new here (calibration)

You already know general software testing from Python — what a unit test is, what mocking is,
what coverage means, `pytest.mark.parametrize`. We skip re-teaching those concepts. What's
genuinely new here, calibrated to a staff-level GPU-fleet-operator track:

- **The absence of magic is deliberate**, not a missing feature — a Go test table is an ordinary
  slice literal, not a decorator-driven fixture system, and that has real consequences for how
  it's reviewed and debugged.
- **The fake-vs-envtest split is a fidelity ladder**, not a binary "unit vs integration" choice —
  knowing exactly where the fake client's fidelity runs out (and why) is a staff-level signal, not
  a beginner one.
- **Coverage as a signal, not a target** — chasing a percentage instead of asking "what branch is
  NOT covered and why is that OK" is a real interview tell, and we go deep on that framing here.
- **Async assertion discipline** (`Eventually`/polling against a real reconcile loop) and
  **`t.Cleanup` vs `defer`** are genuinely new mechanics beyond what table-driven tests alone teach
  you — skipped in most intro material, load-bearing in a controller repo.

## Core concepts

### Test file mechanics

A test lives in `foo_test.go`, function `TestXxx(t *testing.T)`, in the same package (white-box)
or `package foo_test` (black-box, only the exported API). Run with `go test ./...`.

### Table-driven tests + subtests

The dominant Go idiom. One test, a slice of cases, `t.Run` per case so failures name themselves:

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

`t.Run` gives you a named subtest node — failures print `TestAggregate/dupes_sum`, and you can
target one with `go test -run TestAggregate/dupes_sum`.

This is worth pausing on as its own idea: table-driven tests are Go's answer to
`@pytest.mark.parametrize`, but the absence of a magic decorator is deliberate. The table is just
a slice literal — a plain Go value. It's inspectable in a debugger, diffable in code review exactly
like any other data structure, and greppable with normal tools. There's no framework layer
translating the table into test identities behind the scenes; `t.Run(tt.name, ...)` *is* the whole
mechanism, in plain sight.

### `t.Parallel` and the loop-variable change

`t.Parallel()` signals a subtest can run concurrently with other parallel siblings; the parent
waits for all of them at the end. This surfaces shared-state bugs — two parallel cases mutating a
shared map will trip `-race`. Historically the classic trap was **loop-variable capture**:
pre-Go 1.22, `tt` was one variable reused across iterations, so a parallel closure often saw the
*last* case. You had to write `tt := tt` to shadow it. **Go 1.22 changed loop semantics so each
iteration gets a fresh `tt`** — the `tt := tt` dance is no longer needed when your `go.mod`
declares `go 1.22` or later. Know both facts: interviewers ask, and older code still has the
shadow.

```go
for _, tt := range tests {
	t.Run(tt.name, func(t *testing.T) {
		t.Parallel() // Go 1.22+: tt is per-iteration, safe to capture
		got := Aggregate(tt.in)
		// ...
	})
}
```

A related, sharper trap than the loop-variable issue: table-driven cases that share a **mutable
fixture** across `t.Run` cases — a shared `*bytes.Buffer`, a shared map passed by reference, a
package-level test server. Marking those subtests `t.Parallel()` doesn't just risk a wrong
assertion, it silently introduces a **data race between subtests** that `-race` will catch only if
CI actually runs with `-race` and the racing writes happen to overlap in that run. It's a
nondeterministic, hard-to-reproduce failure precisely because it depends on goroutine scheduling —
budget for it by giving each case its own fresh fixture, never a shared one.

### `t.Cleanup` vs manual `defer`

`t.Cleanup(fn)` registers a function to run when the test (and its subtests) complete, in LIFO
order — it's the testing-package-native alternative to `defer` inside a test body. The difference
that matters in a real suite: `t.Cleanup` runs even if a subtest **helper panics** or the test is
**skipped mid-way** via `t.Skip`, and it composes correctly with `t.Parallel()` and nested
subtests — a cleanup registered by a helper function fires at the right point in the tree even
though the helper itself has already returned. A `defer` written inside a shared helper only runs
when *that* function returns, not when the whole test tree finishes, which is the wrong lifetime
for things like a temp directory or an `envtest` environment that the entire subtest tree depends
on. Prefer `t.Cleanup` for anything a helper sets up on behalf of the caller.

### Fakes over mocks

Depend on a small interface, pass a hand-written fake in tests. This is the single most important
Go testing idiom and the biggest delta from Python's `unittest.mock`/`MagicMock` habit, which does
**not** carry over — idiomatic Go fakes a small interface with a hand-written struct, and
monkeypatching is essentially impossible because there's no attribute to reassign.

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

Reach for a **generated mock** (`gomock`/`mockery`) only when you must assert on *call order or
exact arguments across many methods* — e.g. verifying a client calls `Create` then `Update` then
`Delete`. For return-value stubbing, a fake is less code, refactor-proof, and far easier to read.
The rule of thumb: **fakes verify state, mocks verify behavior** — reach for the tool that matches
what you're actually asserting, not the one your last language's testing library defaulted to.

### testify, judiciously

`require`/`assert` cut boilerplate: `require.NoError(t, err)` stops the test (like a fatal),
`assert.Equal(t, want, got)` continues. Use `require` for preconditions (a nil-check that would
panic downstream) and `assert` for the checks you want all of. Don't let it become a crutch —
plain `if got != want { t.Errorf(...) }` is perfectly idiomatic and the stdlib uses nothing else.

### Golden files

For large expected outputs (rendered Prometheus exposition text, generated YAML), store the
expectation in `testdata/foo.golden` and gate regeneration behind a flag:

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

`testdata/` is special: the Go tool ignores it for builds. Regenerate with
`go test -run TestRender -update`, then diff the golden in review.

### `-race` and `-cover` — and coverage as a signal, not a target

`go test -race` instruments memory access and flags concurrent unsynchronized read/write —
invaluable for exporters and controllers that run goroutines. `go test -cover` prints statement
coverage; `-coverprofile=c.out` + `go tool cover -html=c.out` shows line-by-line. CI should run
`go test -race -cover ./...`.

Be explicit about what coverage *is not*: a percentage is a proxy, not a proof. The concrete
failure mode is chasing "70%" by testing getters, setters, and other trivial glue code while the
branches that actually matter — the error path, the empty-input path, the concurrent-write path —
stay untested. A staff-level interviewer, shown a coverage report, is far more likely to ask
"what's *not* covered here, and is that OK?" than to ask for the raw number. Practice answering
that question about your own suite: name the uncovered branch and justify it (dead code, an
`os.Exit` path you test manually, a truly unreachable default case) rather than padding coverage
to hit a threshold.

### Benchmarks

`BenchmarkXxx(b *testing.B)` loops `b.N` times; the framework tunes `N`. Run with
`go test -bench=. -benchmem`:

```go
func BenchmarkAggregate(b *testing.B) {
	in := makeSamples(10000)
	b.ResetTimer()
	for range b.N {
		_ = Aggregate(in)
	}
}
```

`b.ResetTimer()` excludes setup; `-benchmem` reports allocs/op, which is what you tune in hot
exporter paths.

### Fuzzing — and where it's actually worth it

Go 1.18+ has native fuzzing: `FuzzXxx(f *testing.F)`, seed with `f.Add`, run bodies via `f.Fuzz`.
Fuzzing is not a blanket recommendation — it earns its keep on code that parses input whose shape
you don't fully control. In `gpu-cost-exporter`, that's exactly one function: whatever parses the
cost CSV/JSON export. A two-line fuzz target seeded from `testdata/` is realistic and buildable:

```go
func FuzzParseCostExport(f *testing.F) {
	f.Add([]byte(`pool,cost\na100,3.0\n`)) // seed from testdata/
	f.Fuzz(func(t *testing.T, data []byte) {
		_, _ = ParseCostExport(data) // must not panic on any input
	})
}
```

`go test -fuzz=FuzzParseCostExport` runs it; a found crash is written to `testdata/fuzz/`. You
rarely need fuzzing for pure aggregation logic over already-typed Go values — reach for it
specifically where untrusted-shaped bytes cross into your program.

### How controllers are tested: the fidelity ladder

Two layers you must be able to name — and, more importantly, know why *both* exist rather than
picking one:

- **Unit — `controller-runtime` fake client.** `sigs.k8s.io/controller-runtime/pkg/client/fake`
  builds an in-memory client seeded with objects. You construct your `Reconciler` with it, call
  `Reconcile`, and assert on the resulting object state — no cluster, microsecond-fast. Great for
  reconcile logic and error branches.

  ```go
  scheme := runtime.NewScheme()
  _ = corev1.AddToScheme(scheme)
  c := fake.NewClientBuilder().WithScheme(scheme).WithObjects(pod).Build()
  r := &PodReconciler{Client: c}
  _, err := r.Reconcile(ctx, reconcile.Request{NamespacedName: key})
  ```

- **Integration — envtest.** `sigs.k8s.io/controller-runtime/pkg/envtest` starts a **real
  `kube-apiserver` and `etcd`** locally (binaries fetched via `setup-envtest`) with no
  kubelet/nodes. You get a genuine API server to run your manager against, exercising validation,
  defaulting, watches, and status subresources that the fake client fakes imperfectly. Slower; run
  in CI, not on every save.

Treat these as a **fidelity ladder, not a binary choice.** The fake client is fast but leaky: it
has no `resourceVersion`/optimistic-concurrency semantics worth trusting, no OpenAPI schema
validation, and no admission webhook execution. The `client-go` project's own fake-client example
documents this directly in a comment: the fake client "isn't designed to work with informer[s]. It
doesn't support resource version" (see Real-world use cases, below) — that's a maintainer-level
admission that the fake models CRUD, not the real object lifecycle. `envtest` runs an actual API
server, so it catches "the API server actually rejects this" bugs (a bad CRD schema, a webhook
that would reject the patch, a status update that races a stale `resourceVersion`) that a fake
client lets sail through green. The `controller-runtime` maintainers' own FAQ recommends
`envtest.Environment` with a real client "when in doubt," and to structure tests around asserting
**final state** rather than the exact sequence of API calls — because a fake has no
error-injection mechanism to simulate the API server rejecting a call the way a real one would.

**Async assertions against envtest.** Because envtest exercises a real, asynchronous control loop
— watch event → workqueue → `Reconcile` — a naive assertion placed immediately after `Create` can
race the reconciler; the object you just created may not have been reconciled yet. Tests need
polling, not a synchronous check:

```go
gomega.Eventually(func() error {
	var pod corev1.Pod
	if err := k8sClient.Get(ctx, key, &pod); err != nil {
		return err
	}
	if pod.Status.Phase != corev1.PodRunning {
		return fmt.Errorf("phase = %s, want Running", pod.Status.Phase)
	}
	return nil
}, "5s", "100ms").Should(gomega.Succeed())
```

This is a real, common source of flaky CI in controller repos when it's skipped — an assertion
placed right after `Create()` that "usually" passes because the reconciler is fast enough on a
laptop but occasionally races it in CI.

## Perspectives

**Developer view.** Table-driven tests are Go's answer to Python's `@pytest.mark.parametrize` —
but the absence of a magic decorator is deliberate: the table is just a slice literal,
inspectable, debuggable, and diffable in code review exactly like any other data structure. You
never have to ask "what does this decorator actually do at runtime" because there is no decorator.

**Reviewer/hiring view.** A PR's test file is read *before* its source file by an experienced Go
reviewer. This isn't a personal habit — it's near-verbatim Google's own official code-review
standard: reviewers are told to examine whether the tests are valid, whether they'll catch real
bugs, and whether they could produce false positives, as core evidence for judging the change's
complexity. Your test file is the artifact a senior reviewer (or interviewer) forms their opinion
of you from first.

**Systems/fidelity view.** The fake-client-vs-envtest split is a fidelity ladder, not a binary
choice. Fakes are fast but leaky — no `resourceVersion` semantics, no OpenAPI validation, no
admission webhooks. `envtest` is slower but runs a real `kube-apiserver` + `etcd`, catching "the
API server actually rejects this" bugs that a fake lets through untouched. Choosing where on that
ladder to test a given piece of logic is itself a design decision, not a default.

**Failure-mode view.** The fake client's leakiness isn't hypothetical. `controller-runtime`
maintainers themselves recommend `envtest.Environment` over the fake client "when in doubt,"
specifically because the fake has no error-injection mechanism and mismatched
`Generation`/`resourceVersion` behavior versus a real API server. Trusting a green fake-client
suite as proof of correctness for anything touching optimistic concurrency — status updates,
conflicting writers, retry-on-conflict logic — is a real staff-level trap: the suite passes, the
logic is wrong, and it only shows up under real concurrent load in production.

## Real-world use cases

- **Google — "What to look for in a code review"**
  (<https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md>) —
  Google's own official public code-review standard: reviewers must examine complexity, and tests
  are called out as requiring their own scrutiny — are they valid, will they catch real bugs,
  could they produce false positives. This is the reviewer mental model your tests need to survive
  in any staff-level Go review.
- **kubernetes-sigs/controller-runtime — `FAQ.md`**
  (<https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md>) — official maintainer
  guidance: prefer `envtest.Environment` with a real client and API server over the fake client;
  structure tests to verify final state rather than exact API call sequences. Directly corroborates
  the fake-vs-envtest section above, from the people who maintain both.
- **kubernetes/client-go — fake-client example**
  (<https://github.com/kubernetes/client-go/blob/master/examples/fake-client/main_test.go>) —
  concrete evidence of exactly where the fake client's fidelity runs out: a code comment states
  plainly that "the fake client isn't designed to work with informer[s]. It doesn't support
  resource version" — the leak named explicitly, in the source.

## Worked example

A focused unit test for the exporter's collector using a fake `CostSource` — no network,
deterministic, race-clean:

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

Note the shape: fake injected per-case (each case gets its own fresh `fakeSource`, so
`t.Parallel()` is safe — no shared mutable fixture), edge cases (nil, empty, error, missing rate)
are rows, and every assertion names the pool that failed. If this collector later reads from a
live cluster instead of a fake `CostSource`, the natural next step up the fidelity ladder is an
`envtest`-backed test that seeds real `Pod`/`Node` objects and polls with `Eventually` for the
aggregation to reflect them — exactly the pattern shown in Core concepts above.

## Practice

In `gpu-cost-exporter`, write the test suite for your aggregation logic:

1. **Table-driven `TestCollect`** covering: empty input, `nil` usage slice, duplicate `Pool`
   labels that must sum, a pool with no matching rate (error branch), and a source error.
2. **A `fakeSource`** implementing your `CostSource` interface — hand-written struct, no mock
   library. Give each table case its own instance (no shared fixture) so `t.Parallel()` is safe.
3. **One benchmark** `BenchmarkCollect` over ~10k synthetic usage rows with `b.ResetTimer()` and
   `-benchmem`.
4. **One fuzz target**, `FuzzParseCostExport`, over whatever function parses your cost
   CSV/JSON export, seeded from `testdata/`.

**Acceptance:** `go test -race -cover ./...` is green, coverage of the business-logic package is
~70%+ *and* you can name the uncovered branch and why it's OK, no test touches a real network or
cluster, `go test -bench=. -benchmem` reports allocs/op, and `go test -fuzz=FuzzParseCostExport
-fuzztime=30s` finds no crash.

Full deliverable spec: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md).

## Common pitfalls

1. **Trusting a green fake-client suite as proof against optimistic-concurrency bugs.** Status
   conflicts and stale `resourceVersion` writes aren't modeled by the fake client — only `envtest`
   exercises real API-server conflict semantics. A passing fake-client suite says nothing about
   these bugs.
2. **Asserting immediately after `Create`/`Update` in an `envtest` suite without polling.** A real,
   common source of flaky CI in controller repos, caused by the asynchronous reconcile loop —
   use `gomega.Eventually` (or an equivalent poll), not a synchronous check.
3. **Reaching for a generated mock (`gomock`/`mockery`) out of Python-testing-library habit**, for
   a case a hand-written fake would cover in fewer lines. Reserve generated mocks for asserting
   call order/arguments across multiple methods; default to a fake for state-based stubbing.
4. **Table-driven test cases sharing a mutable fixture across `t.Run` cases.** Silently breaks
   under `t.Parallel()`, producing a nondeterministic, hard-to-reproduce data race between
   subtests — give every case its own fixture.
5. **Treating 100% coverage as a goal.** Produces tests of trivial glue code while leaving actual
   branch conditions (the error path, the empty-input path) under-tested. Coverage is a signal to
   interrogate, not a number to maximize.

## Self-check

- **When do you write a hand-written fake vs. a generated mock?**
  **Answer:** Default to a **fake** — a small hand-written struct satisfying a narrow interface —
  for stubbing return values and errors; it's less code and survives refactors. Reach for a
  **generated mock** (gomock/mockery) only when you must assert on *interaction*: exact arguments,
  call counts, or ordering across multiple methods (e.g. Create-then-Update-then-Delete). Fakes
  verify *state*; mocks verify *behavior*.
- **What does `t.Parallel` change about shared state and (pre-1.22) loop-variable capture?**
  **Answer:** It lets marked subtests run concurrently, so any shared mutable state (a shared map,
  counter, or the same fake instance mutated in place) becomes a data race that `-race` will flag —
  parallel tests must be independent. Pre-Go 1.22 the loop variable was reused across iterations,
  so a parallel closure captured a reference that all iterations shared and typically read the
  *last* case's value; the fix was `tt := tt` to shadow. Go 1.22 makes each iteration's variable
  fresh, removing the need for that shadow.
- **How do you test a `Reconcile` without a real cluster, and when is that not enough?**
  **Answer:** For unit tests, build a `controller-runtime` **fake client** (`pkg/client/fake`)
  seeded with your input objects, inject it into the reconciler, call `Reconcile`, and assert on
  the resulting in-memory object state — microsecond-fast, no cluster. It's not enough when the
  logic depends on real API-server behavior the fake doesn't model: `resourceVersion`/optimistic
  concurrency, OpenAPI validation, admission webhooks, or watch-driven timing. For that, use
  **envtest**, which starts a real local `kube-apiserver` + `etcd` (no kubelet/nodes) so
  validation, defaulting, and status subresources behave like production; run that layer in CI and
  assert asynchronously with polling (`Eventually`), not immediately after `Create`.
- **Why does a staff-level interviewer ask "what's not covered?" instead of "what's your coverage
  number?"**
  **Answer:** Because a coverage percentage doesn't distinguish trivial glue (getters, setters,
  simple wiring) from the branches that actually matter (error paths, empty-input paths,
  concurrent-write paths). Chasing a target number is easy to game by testing the easy 70% and
  skipping the hard 30%; naming the specific uncovered branch and justifying why it's acceptable
  demonstrates you understand your own code's risk surface, which the number alone can't show.

## Connections & what's next

This lesson leans directly on lesson 04's `-race` detector and `context.Context` propagation —
the concurrency bugs testing exists to catch. It's also a direct rehearsal for lesson 09,
**Controller primer (CRD · reconcile · envtest)**, which puts the fake-client/envtest fidelity
ladder to work on a real reconciler; nothing here is throwaway. Next: lesson 06, **Modules and
Project Layout**, moves from "is the code correct" to "is the module shaped, versioned, and
packaged the way the ecosystem expects" — the same `internal/` boundary you've been testing
against becomes a compiler-enforced API contract there.

## References & further reading

**Primary sources**
- `testing` package docs — <https://pkg.go.dev/testing> — authoritative reference for `T`, `B`,
  `F`, `-race`, `-cover`, `Parallel`, `Cleanup`. Read for exact signatures and flags.
- Using Subtests and Sub-benchmarks — <https://go.dev/blog/subtests> — the canonical explanation
  of `t.Run`, parallelism, and table structure. Read for the idiom every reviewer expects.
- Kubebuilder — Configuring envtest — <https://book.kubebuilder.io/reference/envtest.html> — how
  the real apiserver+etcd harness is wired and fetched via `setup-envtest`. Read for how you'll
  test the capstone controller.
- kubernetes-sigs/controller-runtime `FAQ.md` —
  <https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md> — official maintainer
  guidance on fake client vs envtest and asserting final state. Read for the authoritative
  fidelity-ladder call.

**Real-world engineering blogs**
- Google eng-practices — "What to look for in a code review" —
  <https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md> — what it
  shows: tests are explicitly named as requiring their own review scrutiny, not assumed correct
  because they're green.
- kubernetes/client-go — fake-client example —
  <https://github.com/kubernetes/client-go/blob/master/examples/fake-client/main_test.go> — what
  it shows: the fake client's own maintainers document, in a code comment, exactly where its
  fidelity to a real API server runs out.

**Deeper dives**
- Google eng-practices — full code-review guide index —
  <https://github.com/google/eng-practices/blob/master/review/index.md> — broader context on the
  review culture your tests are written for.
- A Comprehensive Guide to Testing in Go (JetBrains) —
  <https://blog.jetbrains.com/go/2022/11/22/comprehensive-guide-to-testing-in-go/> — end-to-end
  tour: tables, fakes, coverage, fuzzing. Good consolidation with practical patterns.

{% endraw %}
