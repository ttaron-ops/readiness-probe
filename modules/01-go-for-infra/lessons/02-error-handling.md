---
lesson: "01.2"
title: "Error Handling"
module: "01"
concept: "Error Handling"
status: not-started
est_time: "9h"
prev: "01-syntax-and-types.md"
next: "03-interfaces-and-composition.md"
artifacts: []
sources: 7
---

# 01.2 · Error Handling

> **Concept.** Idiomatic Go errors for a reconcile loop — wrap with `%w`, `errors.Is/As/Join`, sentinel vs typed errors, retry classification, and when to panic vs return.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

[Lesson 1](01-syntax-and-types.md) built the type-system muscle memory — value vs. pointer semantics, zero values, slice/map aliasing, struct layout. That's the vocabulary; this lesson is where it starts doing real work. Every error you construct, wrap, and classify in Go is itself a value with the semantics lesson 1 covered — a sentinel error is a package-level value compared by identity, a typed error is a struct you build with a literal and read fields off of, and `Unwrap() error` is just a pointer-receiver method like any other. This lesson takes that vocabulary and builds the specific discipline a Kubernetes reconcile loop runs on: deciding, for every error, whether the right response is "skip," "retry," or "give up" — and making that decision survive being wrapped three call layers deep. It unlocks [Lesson 3 (Interfaces & Composition)](03-interfaces-and-composition.md), where `error` itself becomes the running example of Go's smallest, most consequential interface.

## Why this matters

A Kubernetes reconcile loop *is* an error-classification engine: "not found" means the object was deleted — return nil, stop. "Conflict" or a network blip means requeue with backoff. A terminal validation failure means give up and record a condition. controller-runtime decides whether to retry based on the error you return, so getting error semantics wrong causes hot-loop retries on unfixable errors or silent drops of transient ones — both are on-call incidents. This is the highest-signal Go skill for the controller work and a guaranteed interview topic.

The disposition you assign an error becomes production behavior directly. Get it wrong and you either hot-loop retrying an unfixable validation error — burning API server QPS — or silently drop a transient blip that should have retried. The most expensive version of this mistake at scale isn't a missing `if err != nil`; it's the opposite failure — over-classification collapse, where every non-nil error gets treated as "retry forever," turning one permanently-broken object (a CRD spec that will never validate) into an infinite, silent, CPU-burning retry loop that never pages anyone because from outside, "the controller is running fine."

## What's new here (calibration)

Per the module README's skip-list, this lesson skips programming 101 and reflection/runtime internals, and doesn't re-teach exception handling as a concept — you already know what "catch the specific error type, not everything" means from Python. What's genuinely new:

- **Go has no exception hierarchy** — the `except NotFoundError` mental model maps to `errors.Is`/`errors.As` over sentinel and typed values, built with composition and wrapping instead of `class` inheritance. That's a different *mechanism* for a familiar *goal*.
- **`errors.Join` and multi-error trees** (Go 1.20+) — a shape Python's exception model doesn't have a clean equivalent for at all.
- **Retry-classification as a distributed-systems discipline**, not just a language feature — jittered exponential backoff, sentinel-identity breakage across vendored module versions, and treating `context.DeadlineExceeded` as a caller-gave-up signal rather than an application error.
- **The visual/structural signal of `if err != nil` everywhere** — why staff reviewers read the *shape* of error checks (adjacent to the call vs. batched at the end) as a code-quality signal, something Python's late/aggregated exception handling has no equivalent for.

## Core concepts

### From Python to Go

Python raises exceptions and unwinds the stack; Go returns errors as ordinary values and you check them inline. The deltas that trip a Pythonista:

- **No `try/except`.** The `if err != nil { return ..., err }` you'll type constantly is not boilerplate to eliminate — it's explicit, local control flow. Every call site declares what it does on failure.
- **`panic` is not `raise`.** Panic is for *programmer bugs* (nil deref, impossible state), not for expected failures like "file missing" or "404." Don't reach for it as an exception substitute.
- **No exception hierarchy — but structure exists.** Python's `except NotFoundError` maps to Go's `errors.Is`/`errors.As` over sentinel and typed errors. You build the hierarchy with values and wrapping, not `class` inheritance.
- **Wrapping is manual and additive.** Python chains via `raise X from Y`. Go wraps with `fmt.Errorf("...: %w", err)`, and the chain is inspected explicitly with `Is`/`As`. Nothing propagates automatically.

`if err != nil` being everywhere is the feature: failure paths are visible in the code, not hidden in a stack unwinder. This has a structural consequence worth naming explicitly: because Go checks and handles errors *at the call site*, immediately, a reviewer can read the shape of a function and see where every failure path goes without tracing exception propagation elsewhere in the file. Staff-level reviewers read a wall of adjacent `if err != nil` blocks as healthy; they read errors batched, ignored, or handled far from the call as a smell — the Go equivalent of Python code that catches `Exception` at the top of `main()`.

### `error` is an interface

Anything with `Error() string` is an error. `nil` means success.

```go
type error interface{ Error() string }
```

### Return, don't raise

The idiom — handle or annotate-and-return, never swallow:

```go
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("read cost export %q: %w", path, err) // add context, keep chain
}
```

### `%w` vs `%v` — a design decision about trust

In `fmt.Errorf`, `%w` *wraps* — the original error stays reachable via `errors.Is/As`. `%v` only formats the text, severing the chain. This isn't just a formatting choice — it's a statement about what the caller is allowed to know. `%w` says "you may inspect my cause and react to it." `%v` says "you may not — I've decided this detail is mine to hide." A senior engineer can read a codebase's `%w` vs. `%v` choices as a map of its trust boundaries: where callers are trusted with implementation detail, and where a layer deliberately hides it. **Wrap with `%w` when a caller might need to inspect the cause; use `%v` when you're deliberately hiding internals.**

You may wrap multiple errors with multiple `%w` verbs in one `fmt.Errorf` call (Go 1.20+), or combine several independent errors with `errors.Join`.

### `errors.Join` and multi-error trees (Go 1.20+)

`fmt.Errorf` with one `%w` builds a linear *chain*. `errors.Join(err1, err2, ...)` builds a *tree* — and `errors.Is`/`errors.As` search all branches, not just one linear path.

```go
var errs []error
for _, node := range nodes {
    if err := scrape(node); err != nil {
        errs = append(errs, fmt.Errorf("scrape %s: %w", node, err))
    }
}
if len(errs) > 0 {
    return errors.Join(errs...) // one error value, all N failures preserved
}
```

This matters directly for a fan-out scrape (the pattern [lesson 4](04-concurrency-and-context.md) builds on): when you scrape 50 GPU nodes concurrently and 3 fail, you don't want to collapse that to "something failed" — `errors.Join` lets the caller still `errors.Is`/`errors.As` against any of the three original causes, and `err.Error()` prints all three, not just the last one that happened to overwrite a shared variable.

### Sentinel errors

Package-level `var`, compared by identity with `errors.Is`:

```go
var ErrNotFound = errors.New("resource not found")

func lookup(ns string) (float64, error) {
    v, ok := cache[ns]
    if !ok {
        return 0, ErrNotFound // return the sentinel
    }
    return v, nil
}
```

**Sentinel identity breaks across module versions.** `errors.Is` compares by *value* identity, not by string content. If two versions of the same dependency each define their own `ErrNotFound` — say v1.2.0 and v1.4.0 of a client library, pulled in transitively at different versions by two of your own dependencies — `errors.Is` will **not** match across them, even though both errors print the identical string. This is exactly the kind of thing that bites in the multi-version vendoring scenarios [lesson 6](06-modules-and-layout.md) covers, and it's a real trap: the code compiles, the error message looks right in the logs, and the `errors.Is` check silently returns `false` because you're comparing two different package-level variables that happen to share a name and message.

### Typed errors

A struct implementing `error`, carrying *data* the caller can extract:

```go
type TransientError struct {
    Op    string
    Cause error
}
func (e *TransientError) Error() string { return fmt.Sprintf("%s: transient: %v", e.Op, e.Cause) }
func (e *TransientError) Unwrap() error { return e.Cause } // makes errors.Is/As see through it
```

Rule of thumb: **sentinel** when the caller only needs to know *which* error ("is this NotFound?"); **typed** when the caller needs *fields* (retry-after, the offending key, an HTTP status).

### `errors.Is` vs `errors.As`

- `errors.Is(err, ErrNotFound)` — walks the `%w`/`Unwrap` chain (or tree, if `errors.Join` was used) testing *identity* (or a custom `Is` method). Use for sentinels: "is this specific error anywhere in the chain?"
- `errors.As(err, &target)` — walks the chain looking for an error whose *type* matches `target`, and assigns it so you can read its fields. Use for typed errors.

```go
if errors.Is(err, ErrNotFound) { return nil }        // skip: nothing to reconcile

var te *TransientError
if errors.As(err, &te) {                              // extract typed error
    log.Printf("retrying op=%s: %v", te.Op, te.Cause)
    return retryWithBackoff()
}
```

The **`Unwrap() error` method** (or `%w`) is what makes both `Is` and `As` see *through* your typed errors to the wrapped cause. Without it, the chain stops at your struct.

### Error as control flow, cleanly

Reconcile semantics reduce to classification:

```go
func classify(err error) Disposition {
    switch {
    case err == nil:
        return Done
    case errors.Is(err, ErrNotFound):
        return Skip            // object gone — success, stop
    case isRetryable(err):
        return Retry           // transient — requeue with backoff
    default:
        return Terminal        // unfixable — record condition, don't loop
    }
}
```

This mirrors controller-runtime: returning an error triggers a requeue; returning `nil` (or `ctrl.Result{}`) stops. You classify *once*, at the boundary.

### `context.DeadlineExceeded` / `context.Canceled` are errors you must classify too

These are ordinary `error` values (`context.DeadlineExceeded`, `context.Canceled`) returned when a `context.Context` times out or is cancelled — and they need their own slot in your classification, not a default fallthrough. Treating a deadline as "terminal" is usually wrong; treating it as "the caller gave up, don't log this as an application-level failure" is usually right. A common logging-noise mistake is logging every `context.Canceled` at error level from a request whose client simply disconnected — that's operational noise, not an incident. This bridges directly into [lesson 4](04-concurrency-and-context.md)'s treatment of context propagation and cancellation.

### Retry classification is a distributed-systems pattern, not a Go feature

The retryable-vs-terminal split generalizes far beyond controller-runtime: HTTP 429 (rate limited) and 503 (unavailable) are the textbook retryable statuses; 400 (bad request) and 404 (not found, on an otherwise well-formed request) are terminal — retrying them wastes calls and never succeeds. This is exactly the pattern used for classifying errors from *any* API client, GPU-cost billing APIs included. And when you retry, **jittered exponential backoff** matters for a reason specific to fleets: without jitter, a fleet of controllers that all fail at the same moment (a control-plane blip) will all retry in lockstep at the same intervals, turning one transient outage into a self-inflicted thundering-herd second outage against the same backend.

### When to panic vs return

Return an `error` for anything the caller could reasonably handle: I/O failure, bad input, missing resource, API conflict. **Panic only for invariant violations** — impossible states that indicate a bug (a nil dependency that must be wired at startup, an unreachable `default`). In a long-running controller, an *unrecovered* panic in a worker crashes the process; controller-runtime installs a top-level `recover` per reconcile so one bad object doesn't take down the manager, but you should not lean on that — it's a safety net, not a design pattern. Also acceptable: panic in `init`/startup when misconfiguration makes the program unable to run at all (fail fast, loudly). Never panic to signal an expected, per-item failure inside the loop.

### Don't over-wrap, don't double-log

Add context once per layer (`operation: %w`), and log the error at the top of the stack — not at every return, or you get the same failure printed five times. Wrap to *add information* (which path, which namespace), not to restate.

## Perspectives

**Developer view.** Error wrapping is a design decision about what the caller may know — `%w` says "you may inspect my cause," `%v` says "you may not." A senior engineer reads a codebase's `%w`/`%v` choices as a map of its trust boundaries: which layers expose their internals to callers, and which deliberately don't. Reviewing a diff's error-handling isn't just "did they check `err`" — it's "did they wrap correctly, given who's going to catch this."

**Operator view.** In a reconcile loop, the disposition assigned to an error becomes production behavior directly — get it wrong and you either hot-loop retrying an unfixable validation error, burning API server QPS, or silently drop a transient blip that should retry. The `classify()` pattern in this lesson is exactly what a real "why is this controller stuck at 100% CPU retrying" incident traces back to when you pull up the postmortem.

**Reliability / systems view.** Retry-with-backoff is a distributed-systems pattern independent of Go — the classification of retryable (429/503) vs. terminal (400/404 on a well-formed request) generalizes across every API client you'll ever write, GPU-cost billing APIs included. Jittered exponential backoff prevents a fleet of controllers that all fail at once from retrying in lockstep and causing a second, thundering-herd outage on top of the first.

**Failure-mode view.** The most expensive error-handling mistake at scale isn't a missing `if err != nil` — it's over-classification collapse: treating every non-nil error as "retry forever," which turns one permanently bad object (a CRD spec that will never validate) into an infinite, silent, CPU-burning retry loop that never pages anyone, because from the outside "the controller is running fine." The bug isn't a crash; it's a controller that looks healthy while doing useless work forever.

## Real-world use cases

- **Google Cloud — "Reduce 429 errors on Vertex AI"** — <https://cloud.google.com/blog/products/ai-machine-learning/reduce-429-errors-on-vertex-ai> — official guidance classifying 429/503 as transient (retry with exponential backoff + jitter) versus other errors as terminal, with concrete SDK-level retry configuration. What it shows: the exact `classify()` disposition pattern this lesson teaches, from the API-consumer side, demonstrating the pattern is general — not something Kubernetes invented.
- **uber-go/guide — error-handling section** — <https://github.com/uber-go/guide/blob/master/style.md> — Uber's production Go style guide states plainly that "panic/recover is not an error handling strategy," and recommends specific error types for matchable errors plus `fmt.Errorf` wrapping with `%w` for dynamic context. What it shows: a real, widely-adopted company standard independently arriving at the same panic/return rule and sentinel-vs-typed distinction taught here.

## Worked example

Classifier for the exporter's fetch step, distinguishing skip vs. retry vs. terminal — the reconcile pattern in miniature.

```go
package cost

import (
	"errors"
	"fmt"
)

// Sentinel: caller only needs identity.
var ErrNamespaceNotFound = errors.New("namespace not found")

// Typed: caller needs the reason and the underlying cause.
type TransientError struct {
	Op    string
	Cause error
}

func (e *TransientError) Error() string { return fmt.Sprintf("%s: transient: %v", e.Op, e.Cause) }
func (e *TransientError) Unwrap() error { return e.Cause }

// fetchCost is 3 wrap layers deep: transport -> parse -> fetch.
func fetchCost(ns string) (float64, error) {
	usd, err := queryBackend(ns)
	if err != nil {
		// wrap with %w so the sentinel/typed error survives to the caller
		return 0, fmt.Errorf("fetchCost %q: %w", ns, err)
	}
	return usd, nil
}

type Disposition int

const (
	Done Disposition = iota
	Skip
	Retry
	Terminal
)

func classify(err error) Disposition {
	switch {
	case err == nil:
		return Done
	case errors.Is(err, ErrNamespaceNotFound):
		return Skip // deleted namespace — not an error to retry
	}
	var te *TransientError
	if errors.As(err, &te) {
		return Retry // network/backend blip — requeue with backoff
	}
	return Terminal // e.g. malformed data — no point retrying
}

// queryBackend simulates the two failure modes, each wrapped once more.
func queryBackend(ns string) (float64, error) {
	if ns == "gone" {
		return 0, fmt.Errorf("query %q: %w", ns, ErrNamespaceNotFound)
	}
	if ns == "flaky" {
		return 0, &TransientError{Op: "query", Cause: errors.New("connection reset")}
	}
	return 41.0, nil
}
```

Caller usage:

```go
_, err := fetchCost("gone")
switch classify(err) {
case Skip:
	// nothing to do
case Retry:
	requeueWithBackoff()
case Terminal:
	recordFailureCondition(err)
}
```

`errors.Is(err, ErrNamespaceNotFound)` returns true even though the sentinel is *three* `fmt.Errorf(...%w)` / `Unwrap` layers below the surface. If any of those three layers had used `%v` instead of `%w`, this would silently return `false` and `classify` would fall through to `Terminal` — the caller would give up on a namespace deletion that should have been a clean `Skip`.

## Practice

Add real error semantics to the [lesson 1](01-syntax-and-types.md) cost tool, feeding directly into [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md).

- Define one **sentinel** (`ErrRecordNotFound` or `ErrNamespaceNotFound`) and one **typed** error (`ParseError{Line int, Cause error}` and/or `TransientError`).
- Wrap failures with `fmt.Errorf("...: %w", err)` through at least **three call layers** (e.g. `readFile` → `decode` → `aggregate`).
- Write a `classify(err) Disposition` that separates **not-found (skip)** from **transient (retry)** from **terminal**, using `errors.Is` for the sentinel and `errors.As` for the typed error.
- For a fan-out step (reading multiple cost-export files), collect per-file failures with `errors.Join` instead of returning only the first error, and prove in a test that `errors.Is`/`errors.As` still finds a specific failure buried in the joined tree.
- Add a table-driven test proving the caller detects the sentinel through the wraps and distinguishes retryable from terminal.

**Acceptance:** a caller can detect the specific sentinel through 3 wrap layers via `errors.Is`; retryable-vs-terminal is a *typed* distinction resolved by `errors.As`, not string matching; a joined multi-error still resolves individual sentinels/types correctly; `go vet ./...` clean; `go test ./...` passes.

## Common pitfalls

1. **String-matching on `err.Error()` instead of `errors.Is`/`errors.As`.** It works until someone rewords the message in a dependency update — brittle, un-typesafe, and invisible to `go vet`. There is never a good reason to `strings.Contains(err.Error(), "not found")` when a sentinel or typed error is available.
2. **Wrapping the same error at every layer**, producing `"read cost: fetch node: get pod: context deadline exceeded: context deadline exceeded: context deadline exceeded"`. Decide once, at the layer where it's genuinely useful, where context gets added — not reflexively at every return.
3. **Logging AND returning the same error up the stack**, producing the same root cause printed five times for one failure as it bubbles through five callers that each log it "just in case." Log once, at the boundary that stops propagating it (usually the reconcile loop's top level or the HTTP handler).
4. **Treating panic/recover as Go's try/except.** controller-runtime's per-reconcile `recover` is a safety net for programmer bugs, not a design pattern to lean on — panicking for an *expected* failure (missing object, bad input) is a code smell, and Uber's Go style guide states this outright: "panic/recover is not an error handling strategy."
5. **Breaking the `errors.Is`/`errors.As` chain with `%v` at one layer.** A single `fmt.Errorf("...: %v", err)` instead of `%w` silently severs the chain at that point — a caller three layers up who used to detect a sentinel with `errors.Is` now gets `false`, with no compiler warning and no obvious symptom beyond "the retry logic stopped working after that refactor."

## Self-check

- **`errors.Is` vs `errors.As` — when do you use each?**
  **Answer:** Use `errors.Is(err, target)` to test whether a specific *sentinel value* (or an error with a matching `Is` method) appears anywhere in the wrap chain or `errors.Join` tree — identity comparison. Use `errors.As(err, &target)` to find an error of a specific *type* in the chain and bind it to a variable so you can read its fields. Rule: `Is` for "which error is this?", `As` for "give me the error object so I can inspect its data."

- **An error wrapped 3 layers deep with `%w` — how does the caller detect a specific sentinel?**
  **Answer:** `errors.Is(err, ErrNotFound)`. Each `fmt.Errorf("...: %w", inner)` records an `Unwrap()` link, so `errors.Is` walks the chain from the outermost error down, comparing each against the sentinel, and returns true when it finds a match — depth is irrelevant. Had any layer used `%v` instead of `%w`, the chain would be broken at that point and `Is` would return false.

- **When is `panic` acceptable in a long-running controller?**
  **Answer:** Only for programmer-bug invariant violations and unrecoverable startup misconfiguration — a nil dependency that must have been wired, a truly unreachable code path, or a fatal config error at boot where failing fast is correct. Never for expected per-item failures inside the reconcile loop (missing object, API conflict, transient network error) — those must return an `error` so the controller can classify and requeue. controller-runtime recovers panics per reconcile, but relying on that is a code smell, not a design.

- **Why can `errors.Is` fail to match two sentinel errors that print the identical string?**
  **Answer:** `errors.Is` compares by value identity (or a custom `Is` method), not by message text. If two different versions of the same dependency each declare their own package-level `ErrNotFound` variable, those are two distinct values in memory even though `.Error()` returns the same string on both — `errors.Is` correctly reports no match. This is a real trap in multi-version vendoring, where Go's module system can resolve two different versions of a transitive dependency into the same build.

- **Why does `errors.Join` matter for a fan-out scrape across many nodes, versus wrapping with a single `%w`?**
  **Answer:** `fmt.Errorf` with one `%w` builds a linear chain that can only preserve one underlying cause; if you scrape 50 nodes and 3 fail, wrapping only the last failure silently discards the other two. `errors.Join` builds a tree of all N failures in one error value, and `errors.Is`/`errors.As` search every branch — so a caller can still detect and react to any one of the three original causes, and the printed message shows all three instead of just the last one written to a shared variable.

## Connections & what's next

This lesson's `error` interface is the smallest, most-used interface in the entire language — [Lesson 3 (Interfaces & Composition)](03-interfaces-and-composition.md) picks it up as the running example for what makes a good small interface, and extends the "accept interfaces, return structs" idiom this lesson's typed errors already model in miniature. The `context.DeadlineExceeded`/`context.Canceled` classification here is unfinished business that [Lesson 4 (Concurrency & Context)](04-concurrency-and-context.md) completes, once goroutines and cancellation propagation are on the table. The sentinel-identity-across-versions trap connects forward to [Lesson 6 (Modules & Layout)](06-modules-and-layout.md), where module versioning is covered directly. And the `classify()` disposition pattern built here *is* the shape of a real `Reconcile()` method — [Lesson 9 (Controller primer)](09-controller-primer.md) is where it stops being a toy and starts governing actual requeues.

Next: [03 · Interfaces & Composition](03-interfaces-and-composition.md) — the highest-emphasis lesson in the module, where `error`'s one-method shape becomes the template for designing every other interface you'll write for a controller.

## References & further reading

**Primary sources**
- The Go Blog — "Working with Errors in Go 1.13" — <https://go.dev/blog/go1.13-errors> — canonical rationale for `%w`, `errors.Is`, `errors.As`, and `Unwrap`; read for the design intent behind the API, not just its mechanics.
- `errors` package documentation — <https://pkg.go.dev/errors> — authoritative reference for `Is`, `As`, `Join`, `Unwrap`, and their exact matching semantics; read for the precise contract each function guarantees.

**Real-world engineering blogs**
- Google Cloud — "Reduce 429 errors on Vertex AI" — <https://cloud.google.com/blog/products/ai-machine-learning/reduce-429-errors-on-vertex-ai> — what it shows: official retryable-vs-terminal classification and exponential-backoff-with-jitter guidance from an API vendor, proving the pattern generalizes beyond Kubernetes.
- Uber Go Style Guide, error-handling section — <https://github.com/uber-go/guide/blob/master/style.md> — what it shows: a real, widely-adopted company standard independently landing on "panic/recover is not an error handling strategy" and typed-errors-plus-`%w` as the house style.

**Deeper dives**
- Dave Cheney — "Don't just check errors, handle them gracefully" — <https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully> — the canonical, widely-cited essay on the "handle an error once" principle behind the no-double-logging pitfall in this lesson.
- Learning Go, 2nd ed. (Jon Bodner) — Errors chapter — <https://www.oreilly.com/library/view/learning-go-2nd/9781098139285/> — the clearest end-to-end treatment of sentinels, wrapping, `Is`/`As`, custom error types, and panic/recover; this is the chapter your controller code leans on daily.
- 100 Go Mistakes and How to Avoid Them — error section (Teiva Harsanyi) — <https://100go.co> — concrete anti-patterns: over-wrapping, ignoring errors, `%w` vs `%v` misuse, checking error types by string; the free companion site is a fast, high-density way to pre-empt exactly the mistakes reviewers flag in Go PRs.
