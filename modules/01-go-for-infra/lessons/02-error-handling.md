---
lesson: "01.2"
title: "Error Handling"
module: "01"
concept: "Error Handling"
status: not-started
est_time: "8h"
artifacts: []
---

# 01.2 · Error Handling

> **Concept.** Idiomatic Go errors for a reconcile loop — wrap with `%w`, `errors.Is/As`, sentinel vs typed errors, and when to panic vs return.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Why this matters
A Kubernetes reconcile loop *is* an error-classification engine: "not found" means the object was deleted — return nil, stop. "Conflict" or a network blip means requeue with backoff. A terminal validation failure means give up and record a condition. controller-runtime decides whether to retry based on the error you return, so getting error semantics wrong causes hot-loop retries on unfixable errors or silent drops of transient ones — both are on-call incidents. This is the highest-signal Go skill for the controller work and a guaranteed interview topic.

## From Python to Go
Python raises exceptions and unwinds the stack; Go returns errors as ordinary values and you check them inline. The deltas that trip a Pythonista:

- **No `try/except`.** The `if err != nil { return ..., err }` you'll type constantly is not boilerplate to eliminate — it's explicit, local control flow. Every call site declares what it does on failure.
- **`panic` is not `raise`.** Panic is for *programmer bugs* (nil deref, impossible state), not for expected failures like "file missing" or "404." Don't reach for it as an exception substitute.
- **No exception hierarchy — but structure exists.** Python's `except NotFoundError` maps to Go's `errors.Is`/`errors.As` over sentinel and typed errors. You build the hierarchy with values and wrapping, not `class` inheritance.
- **Wrapping is manual and additive.** Python chains via `raise X from Y`. Go wraps with `fmt.Errorf("...: %w", err)`, and the chain is inspected explicitly with `Is`/`As`. Nothing propagates automatically.

`if err != nil` being everywhere is the feature: failure paths are visible in the code, not hidden in a stack unwinder.

## Core notes

**`error` is an interface.** Anything with `Error() string` is an error. `nil` means success.

```go
type error interface{ Error() string }
```

**Return, don't raise.** The idiom — handle or annotate-and-return, never swallow:

```go
data, err := os.ReadFile(path)
if err != nil {
    return fmt.Errorf("read cost export %q: %w", path, err) // add context, keep chain
}
```

**`%w` vs `%v`.** In `fmt.Errorf`, `%w` *wraps* — the original error stays reachable via `errors.Is/As`. `%v` only formats the text, severing the chain. **Wrap with `%w` when a caller might need to inspect the cause; use `%v` when you're deliberately hiding internals.** You may wrap multiple errors with multiple `%w` verbs (Go 1.20+), or `errors.Join` several into one.

**Sentinel errors** — package-level `var`, compared by identity with `errors.Is`:

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

**Typed errors** — a struct implementing `error`, carrying *data* the caller can extract:

```go
type TransientError struct {
    Op    string
    Cause error
}
func (e *TransientError) Error() string { return fmt.Sprintf("%s: transient: %v", e.Op, e.Cause) }
func (e *TransientError) Unwrap() error { return e.Cause } // makes errors.Is/As see through it
```

Rule of thumb: **sentinel** when the caller only needs to know *which* error ("is this NotFound?"); **typed** when the caller needs *fields* (retry-after, the offending key, an HTTP status).

**`errors.Is` vs `errors.As`.**

- `errors.Is(err, ErrNotFound)` — walks the `%w`/`Unwrap` chain testing *identity* (or a custom `Is` method). Use for sentinels: "is this specific error anywhere in the chain?"
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

**Error as control flow, cleanly.** Reconcile semantics reduce to classification:

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

**When to panic vs return.** Return an `error` for anything the caller could reasonably handle: I/O failure, bad input, missing resource, API conflict. **Panic only for invariant violations** — impossible states that indicate a bug (a nil dependency that must be wired at startup, an unreachable `default`). In a long-running controller, an *unrecovered* panic in a worker crashes the process; controller-runtime installs a top-level `recover` per reconcile so one bad object doesn't take down the manager, but you should not lean on that. Also acceptable: panic in `init`/startup when misconfiguration makes the program unable to run at all (fail fast, loudly). Never panic to signal an expected, per-item failure inside the loop.

**Don't over-wrap, don't double-log.** Add context once per layer (`operation: %w`), and log the error at the top of the stack — not at every return, or you get the same failure printed five times. Wrap to *add information* (which path, which namespace), not to restate.

## Worked example
Classifier for the exporter's fetch step, distinguishing skip vs retry vs terminal — the reconcile pattern in miniature.

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

`errors.Is(err, ErrNamespaceNotFound)` returns true even though the sentinel is *three* `fmt.Errorf(...%w)` / `Unwrap` layers below the surface.

## Practice
Add real error semantics to the Lesson 01.1 cost tool.

- Define one **sentinel** (`ErrRecordNotFound` or `ErrNamespaceNotFound`) and one **typed** error (`ParseError{Line int, Cause error}` and/or `TransientError`).
- Wrap failures with `fmt.Errorf("...: %w", err)` through at least **three call layers** (e.g. `readFile` → `decode` → `aggregate`).
- Write a `classify(err) Disposition` that separates **not-found (skip)** from **transient (retry)** from **terminal**, using `errors.Is` for the sentinel and `errors.As` for the typed error.
- Add a table-driven test proving the caller detects the sentinel through the wraps and distinguishes retryable from terminal.

**Acceptance:** a caller can detect the specific sentinel through 3 wrap layers via `errors.Is`; retryable-vs-terminal is a *typed* distinction resolved by `errors.As`, not string matching; `go vet ./...` clean; `go test ./...` passes.

## Self-check

**(a) `errors.Is` vs `errors.As` — when do you use each?**
**Answer:** Use `errors.Is(err, target)` to test whether a specific *sentinel value* (or an error with a matching `Is` method) appears anywhere in the wrap chain — identity comparison. Use `errors.As(err, &target)` to find an error of a specific *type* in the chain and bind it to a variable so you can read its fields. Rule: `Is` for "which error is this?", `As` for "give me the error object so I can inspect its data."

**(b) An error wrapped 3 layers deep with `%w` — how does the caller detect a specific sentinel?**
**Answer:** `errors.Is(err, ErrNotFound)`. Each `fmt.Errorf("...: %w", inner)` records an `Unwrap()` link, so `errors.Is` walks the chain from the outermost error down, comparing each against the sentinel, and returns true when it finds a match — depth is irrelevant. (Had any layer used `%v` instead of `%w`, the chain would be broken at that point and `Is` would return false.)

**(c) When is `panic` acceptable in a long-running controller?**
**Answer:** Only for programmer-bug invariant violations and unrecoverable startup misconfiguration — a nil dependency that must have been wired, a truly unreachable code path, or a fatal config error at boot where failing fast is correct. Never for expected per-item failures inside the reconcile loop (missing object, API conflict, transient network error) — those must return an `error` so the controller can classify and requeue. controller-runtime recovers panics per reconcile, but relying on that is a code smell, not a design.

## Resources
1. **Learning Go, 2nd ed. (Jon Bodner) — Errors chapter** — https://www.oreilly.com/library/view/learning-go-2nd/9781098139285/ — Sentinels, wrapping, `Is`/`As`, custom error types, panic/recover. *Deep.* The clearest end-to-end treatment; read it fully since this is the chapter your controller code leans on daily.
2. **100 Go Mistakes and How to Avoid Them — error section (Teiva Harsanyi)** — https://100go.co — Concrete anti-patterns: over-wrapping, ignoring errors, `%w` vs `%v` misuse, checking error types by string. *Deep.* The free companion site is a fast, high-density way to pre-empt exactly the mistakes reviewers flag in Go PRs.
3. **The Go Blog — "Working with Errors in Go 1.13"** — https://go.dev/blog/go1.13-errors — Canonical rationale for `%w`, `errors.Is`, `errors.As`, and `Unwrap`. *Skim.* Short and authoritative; read it to cite the design intent and understand why the API is shaped this way.
