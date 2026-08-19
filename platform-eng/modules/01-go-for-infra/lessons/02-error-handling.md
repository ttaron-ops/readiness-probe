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
sources: 15
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

Transcripts below were captured on **go1.24.7 linux/amd64**. Library facts are cited to the exact versions inspected: **controller-runtime v0.24.1**, **client-go v0.36.0**, **apimachinery v0.36.0**.

## Core concepts

### The design choice: why errors are values

Python's model is a second control-flow channel. `raise` unwinds the stack, skipping every frame that doesn't have a matching `except`, and the frames it skips don't get a say. That's ergonomic — happy-path code stays clean — and it has two costs that matter more in a long-running control loop than in a script:

1. **Failure paths are invisible at the call site.** `total = fetch_cost(ns)` says nothing about what can go wrong or who handles it. To know, you read the callee, its callees, and every `except` block between here and `main`.
2. **The default is propagate.** Forgetting to handle something means it escapes upward, possibly out of the loop, possibly killing a worker.

Go inverts both. An error is an ordinary return value, and ignoring it requires writing code that visibly ignores it. The result is that the failure path is *lexically adjacent* to the operation that can fail:

```go
data, err := os.ReadFile(path)
if err != nil {
	return fmt.Errorf("read cost export %q: %w", path, err)
}
```

Three lines instead of zero. Whether that's "boilerplate" is a taste argument you don't need to win; what you do need is the reviewing consequence. **A reviewer can read a Go function top-to-bottom and see every failure path without leaving the function.** Errors handled far from the call, batched at the end, or swallowed read as a smell in Go the same way `except Exception:` at the top of `main()` reads as a smell in Python. That structural signal is why senior reviewers scan the *shape* of a diff's error handling before reading its logic.

The other half of the design: `panic` exists, but it is not `raise`. Panic is for *programmer bugs* — a nil dereference, an impossible switch case, a dependency that must have been wired at startup. There is a section on it below; the summary is that anything a caller could reasonably handle must be a returned `error`.

### `error` is an interface — and what that costs

```go
type error interface {
	Error() string
}
```

That's the entire definition (it's a predeclared type; the spec defines it exactly this way). Anything with an `Error() string` method is an error. From [lesson 1](01-syntax-and-types.md), an interface value is a 16-byte `{type pointer, data pointer}` pair, so an `error` variable holds a type descriptor plus a pointer to the concrete value. Two consequences you will trip on:

- **`err != nil` is true whenever the type half is set**, even if the data half is a nil pointer. That's the typed-nil trap; [Lesson 3](03-interfaces-and-composition.md) covers it in full, and it is the single most common way a Go program reports a failure that never happened.
- **`errors.New("x") != errors.New("x")`.** `errors.New` returns `*errorString` — a *pointer*, deliberately, so that two errors with identical text are distinct values. That's what makes sentinel comparison meaningful: identity, not text.

### Building errors: which constructor, when

There are exactly three ways to make an error, and the choice is driven by one question — *does a caller need to react differently to this specific failure?*

| Caller needs to match on it? | Message | Use | Example |
|---|---|---|---|
| No | static | `errors.New("…")` inline | `errors.New("empty cost export")` |
| No | dynamic | `fmt.Errorf("…%s…", x)` | `fmt.Errorf("unknown unit %q", u)` |
| Yes | static | package-level `var Err… = errors.New(…)` — a **sentinel** | `var ErrNamespaceNotFound = errors.New("namespace not found")` |
| Yes | dynamic, or carries data | a **custom type** implementing `error` | `type ParseError struct{ Line int; Cause error }` |

(This matrix is the one in the Uber Go style guide; the naming conventions there are worth adopting too: exported sentinels get an `Err` prefix, error *types* get an `Error` suffix — `ErrNotFound` the value, `NotFoundError` the type.)

The distinction that actually decides it in practice: **sentinel when the caller needs to know *which* error; typed when the caller needs *fields* from it.** "Is this a not-found?" is a sentinel question. "How many seconds does the server want me to wait, and which line of the CSV was malformed?" is a typed-error question.

```go
// Sentinel — identity is the whole payload.
var ErrNamespaceNotFound = errors.New("namespace not found")

// Typed — the caller reads Op, RetryAfter, and the cause.
type TransientError struct {
	Op         string
	RetryAfter time.Duration
	Cause      error
}

func (e *TransientError) Error() string {
	return fmt.Sprintf("%s: transient (retry after %s): %v", e.Op, e.RetryAfter, e.Cause)
}
func (e *TransientError) Unwrap() error { return e.Cause } // ← makes Is/As see past this node
```

Use a **pointer receiver** on error types (and return `&TransientError{…}`). Two reasons: mutation-free consistency with [lesson 1](01-syntax-and-types.md)'s receiver rules, and because `errors.As(err, &te)` with `te` of type `*TransientError` matches only pointer values. Mixing `TransientError` (value) and `*TransientError` in a codebase produces `errors.As` calls that silently return `false`.

### What `%w` actually builds

`fmt.Errorf` is not doing anything magic; it constructs a small struct. Reading `src/fmt/errors.go` settles every question about it:

- **Zero `%w` verbs** → `errors.New(formattedString)`. A plain error, chain ends here.
- **Exactly one `%w`** → `&wrapError{msg: formattedString, err: theOperand}`, and `wrapError` has `Error() string` and **`Unwrap() error`**.
- **Two or more `%w`** → `&wrapErrors{msg, errs}`, which has `Error() string` and **`Unwrap() []error`** — the plural form. (Two `%w` verbs in one `Errorf` has been legal since Go 1.20.)

So the "chain" is not a runtime registry or a magic annotation — it is literally a linked list of structs, each holding a formatted message and a pointer to the next error. `%v` formats the operand's text into the message and stores **nothing**, which is why it severs the chain.

```
  fmt.Errorf("scrape node %s: %w", node, err2)     ← three layers of wrapping

   ┌──────────────────────────────────────────┐
   │ *fmt.wrapError                           │
   │  msg: "scrape node gpu-node-7: decode …" │
   │  err: ────────────────┐                  │
   └───────────────────────┼──────────────────┘
                           ▼
           ┌──────────────────────────────────────┐
           │ *fmt.wrapError                       │
           │  msg: "decode cost export: query …"  │
           │  err: ────────────────┐              │
           └───────────────────────┼──────────────┘
                                   ▼
                   ┌──────────────────────────────────────┐
                   │ *fmt.wrapError                       │
                   │  msg: "query \"gone\": namespace …"  │
                   │  err: ────────────────┐              │
                   └───────────────────────┼──────────────┘
                                           ▼
                           ┌────────────────────────────────┐
                           │ *errors.errorString            │
                           │  s: "namespace not found"      │  ← the sentinel,
                           │  (no Unwrap → chain ends)      │    ErrNamespaceNotFound
                           └────────────────────────────────┘

  Each Error() returns only its own msg — which already CONTAINS the inner text,
  because fmt.Errorf formatted it in at construction time. That is why the top-level
  message reads as a full path, and why a %v layer looks identical while being broken.
```

Verify it by walking the structure with a type switch (the same switch `errors.Is` uses internally):

```go
func walk(err error, depth int) {
	for err != nil {
		fmt.Printf("%*s%T: %q\n", depth*2, "", err, err.Error())
		switch x := err.(type) {
		case interface{ Unwrap() error }:
			err = x.Unwrap()
			depth++
		case interface{ Unwrap() []error }:
			for _, e := range x.Unwrap() {
				walk(e, depth+1)
			}
			return
		default:
			return
		}
	}
}
```

```
--- linear chain, three layers ---
Error(): scrape node gpu-node-7: decode cost export: query "gone": namespace not found
*fmt.wrapError: "scrape node gpu-node-7: decode cost export: query \"gone\": namespace not found"
  *fmt.wrapError: "decode cost export: query \"gone\": namespace not found"
    *fmt.wrapError: "query \"gone\": namespace not found"
      *errors.errorString: "namespace not found"
errors.Is(l3, ErrNamespaceNotFound) = true
```

Now change exactly one verb — the middle layer's `%w` becomes `%v` — and change nothing else:

```
--- chain broken by %v at one layer ---
Error(): scrape node gpu-node-7: decode cost export: query "gone": namespace not found
errors.Is(b3, ErrNamespaceNotFound) = false
```

**The printed message is byte-for-byte identical. The behaviour is inverted.** This is the most important transcript in the lesson: a `%v` regression is invisible in logs, invisible in tests that assert on message text, and invisible in code review unless you are specifically looking at verbs. It shows up only as "the retry logic stopped working after that refactor."

That also gives you the actual design meaning of the choice: **`%w` says "callers may inspect and react to my cause"; `%v` says "this detail is mine, and I am not making it part of my API."** Wrapping with `%w` is an API commitment — once a caller depends on `errors.Is(err, ErrX)` through your layer, removing the `%w` is a breaking change with no compiler error. Deliberate `%v` is legitimate at trust boundaries: a public API that doesn't want callers coupled to which HTTP library it uses internally.

### `errors.Is`: the exact traversal

`errors.Is(err, target)` is 20 lines of `src/errors/wrap.go`, and knowing them exactly removes all the mystery:

1. If `err == nil || target == nil`, return `err == target`.
2. Compute once whether `target`'s dynamic type is comparable (uncomparable targets, e.g. a struct with a slice field, would panic on `==` — see [lesson 1](01-syntax-and-types.md)'s comparability rules, so the check is skipped for them).
3. Loop over the current node:
   a. If the target is comparable and `err == target` → **true**.
   b. If `err` has a method `Is(error) bool` and `err.Is(target)` → **true**.
   c. If `err` has `Unwrap() error` → set `err` to that and continue the loop. If it returned nil → **false**.
   d. If `err` has `Unwrap() []error` → recurse into each child **in order**, returning true on the first hit; otherwise → **false**.
   e. Otherwise → **false**.

Two things fall out that people get wrong:

- **Depth is irrelevant.** Three wraps or thirty, `Is` walks until it matches or runs out. Wrapping does not "hide" a sentinel; only `%v` does.
- **The traversal is pre-order, depth-first, left-to-right** over a tree, not a list. `errors.Join` is what makes it a tree.

```
  errors.Is(joined, ErrNamespaceNotFound)  — visit order

        ┌─────────────────────────┐
   (1)  │ *errors.joinError       │  Unwrap() []error
        └───┬──────────┬──────────┘
            │          │          └──────────────┐
            ▼          ▼                         ▼
   (2) wrapError   (4) wrapError            (6) wrapError
       "scrape         "scrape                  "scrape
        gpu-node-1"     gpu-node-2"              gpu-node-3"
            │              │                        │
            ▼              ▼                        ▼
   (3) errorString   (5) *ParseError           (7) errorString
       "namespace         │                        "connection reset"
        not found"        ▼
         ★ MATCH     (5a) errorString
         stops here       "invalid float"

  Order: joinError → child1 → its cause ★ → (never reaches 4-7)
  errors.As(joined, &pe) for *ParseError instead walks 1→2→3→4→5 and stops at (5).
```

Run it:

```
--- errors.Join tree ---
Error() is newline-joined:
scrape gpu-node-1: namespace not found
scrape gpu-node-2: parse line 42: invalid float
scrape gpu-node-3: connection reset
tree:
*errors.joinError: "scrape gpu-node-1: namespace not found\nscrape gpu-node-2: parse line 42: invalid float\nscrape gpu-node-3: connection reset"
  *fmt.wrapError: "scrape gpu-node-1: namespace not found"
    *errors.errorString: "namespace not found"
  *fmt.wrapError: "scrape gpu-node-2: parse line 42: invalid float"
    *main.ParseError: "parse line 42: invalid float"
      *errors.errorString: "invalid float"
  *fmt.wrapError: "scrape gpu-node-3: connection reset"
    *errors.errorString: "connection reset"
errors.Is(joined, ErrNamespaceNotFound) = true
errors.As(joined, &pe) = true -> Line = 42
```

### `errors.As`: the exact contract, and its panics

`errors.As(err, target)` performs the same traversal, but instead of comparing identity it asks, at each node: *is this error's concrete type assignable to what `target` points at?* If yes, it assigns and returns true.

The signature is `As(err error, target any) bool`, and `target` is `any` because it must accept a pointer to any error type. That flexibility is paid for with **runtime panics on misuse** (`src/errors/wrap.go`):

| Call | Result |
|---|---|
| `errors.As(err, &te)` where `te` is `*TransientError` | correct |
| `errors.As(err, te)` — forgot the `&` | **panics**: `errors.As: target must be a non-nil pointer` |
| `errors.As(err, nil)` | **panics**: `errors.As: target cannot be nil` |
| `errors.As(err, &s)` where `s` is `string` | **panics**: `errors.As: *target must be interface or implement error` |
| `errors.As(err, &apiStatus)` where `apiStatus` is an **interface** type | correct and idiomatic — see below |

Two rules follow. **Always pass `&` of a variable whose type is exactly the error type you want** (usually a pointer type, e.g. `var te *TransientError; errors.As(err, &te)`). And **`errors.As` can target an interface**, which is how you match "any error that behaves like X" rather than "any error of concrete type X."

Kubernetes uses precisely that. `apimachinery`'s `reasonAndCodeForError` — the function behind every `apierrors.IsNotFound`, `IsConflict`, `IsTooManyRequests` call you will ever write in a reconciler — is:

```go
func reasonAndCodeForError(err error) (metav1.StatusReason, int32) {
	if status, ok := err.(APIStatus); ok || errors.As(err, &status) {
		return status.Status().Reason, status.Status().Code
	}
	return metav1.StatusReasonUnknown, 0
}
```

`APIStatus` is a one-method interface (`Status() metav1.Status`). The fast path is a direct type assertion; the fallback walks the wrap chain for anything satisfying it. That is why `apierrors.IsNotFound(fmt.Errorf("get gpucost: %w", err))` still works — **as long as every layer used `%w`**. And `IsNotFound` itself is a two-clause decision, worth knowing because the second clause surprises people:

```go
func IsNotFound(err error) bool {
	reason, code := reasonAndCodeForError(err)
	if reason == metav1.StatusReasonNotFound {
		return true
	}
	if _, ok := knownReasons[reason]; !ok && code == http.StatusNotFound {
		return true
	}
	return false
}
```

— i.e. an explicit `NotFound` reason, *or* an HTTP 404 whose reason string the client doesn't recognise (apimachinery v0.36.0).

### Custom `Is` and `As` methods

Any error type may define `Is(error) bool` to declare equivalences the traversal cannot infer. The stdlib's canonical example is `syscall.Errno`, which maps raw errnos onto portable sentinels:

```go
func (e Errno) Is(target error) bool {
	switch target {
	case oserror.ErrPermission:
		return e == EACCES || e == EPERM
	case oserror.ErrExist:
		return e == EEXIST || e == ENOTEMPTY
	case oserror.ErrNotExist:
		return e == ENOENT
	case errorspkg.ErrUnsupported:
		return e == ENOSYS || e == ENOTSUP || e == EOPNOTSUPP
	}
	return false
}
```

That single method is why `errors.Is(err, fs.ErrNotExist)` works uniformly across platforms even though the underlying failure is `ENOENT` on Linux and a different code on Windows. **This is Go's replacement for an exception hierarchy**: instead of `ENOENT` inheriting from `FileNotFoundError`, the concrete error declares which abstract sentinels it answers to.

The rule from the `errors` docs: an `Is` method should compare *shallowly* and must not call `Unwrap` — the traversal already does that, and doing it yourself creates exponential walks.

controller-runtime uses the same trick in reverse, to make a *wrapper type* matchable without exporting it (v0.24.1, `pkg/reconcile/reconcile.go`):

```go
func TerminalError(wrapped error) error { return &terminalError{err: wrapped} }

func (te *terminalError) Is(target error) bool {
	tp := &terminalError{}
	return errors.As(target, &tp) // true if the TARGET is any *terminalError
}
```

So `errors.Is(err, reconcile.TerminalError(nil))` means "is there a terminal error anywhere in this chain?" — you construct a throwaway instance purely as a type token. It looks odd; it is idiomatic, and you will see it in the controller loop below.

### `errors.Join`: one value, N causes

`fmt.Errorf` with one `%w` can only carry one cause. A fan-out scrape across 50 nodes where 3 fail has three causes, and picking one is data loss. `errors.Join` (Go 1.20+) builds the multi-cause node (`src/errors/join.go`):

- It **discards nil errors** and returns `nil` if every argument is nil — so `return errors.Join(errs...)` is safe to write unconditionally, with no `if len(errs) > 0` guard.
- It returns `*joinError{errs []error}` with `Unwrap() []error`.
- `Error()` is the children's messages **joined by newlines** (one message per line, no trailing newline).

```go
func scrapeAll(namespaces []string) (map[string]float64, error) {
	out := make(map[string]float64, len(namespaces))
	var errs []error
	for _, ns := range namespaces {
		usd, err := fetchCost(ns)
		if err != nil {
			errs = append(errs, err) // keep going: partial results are useful
			continue
		}
		out[ns] = usd
	}
	return out, errors.Join(errs...) // nil when errs is empty
}
```

Note the shape: **partial success plus a joined error.** A cost exporter that drops all 50 namespaces because 3 failed is worse than one that reports 47 and tells you which 3 broke.

Two operational cautions. First, the newline-joined `Error()` string is hostile to log aggregators that split on newlines — if you ship structured logs, log the count and the first N causes as fields rather than dumping `err.Error()`. Second, `errors.Unwrap()` (the function) returns `nil` for a joined error: it only calls the singular `Unwrap() error` method, which `joinError` deliberately does not have. Use `errors.Is`/`errors.As`, or type-assert to `interface{ Unwrap() []error }` if you genuinely need the children.

The same asymmetry applies to multi-`%w`:

```
--- two %w verbs in one Errorf ---
Error(): reconcile failed: namespace not found; and status update failed: conflict
has Unwrap() error: false | has Unwrap() []error: true
errors.Unwrap(multi) = <nil>
errors.Is(multi, ErrNamespaceNotFound) = true
```

### Sentinel identity, and where it breaks

`errors.Is` compares by **value identity**, not by message. That is the feature — it's why a reworded message doesn't break your control flow — and it has one sharp edge:

> Two different builds of "the same" sentinel are different values. `errors.Is` correctly returns `false`, and the log line looks right.

This happens when two versions of a dependency end up in one build. Go's minimal version selection normally picks one version per module path, so the common case is safe — but it does **not** deduplicate across a **major-version path change** (`example.com/client/v2` and `example.com/client` are, to the module system, unrelated modules) or across a fork (`github.com/you/client` vendoring alongside `github.com/them/client`). Both can be in the build graph simultaneously, each with its own `var ErrNotFound = errors.New("not found")`, and `errors.Is(errFromV2, v1.ErrNotFound)` is `false` forever. `go mod graph | grep client` is how you find it. [Lesson 6](06-modules-and-layout.md) covers the module mechanics; the defensive habit is to prefer typed errors with an `Is` method (or a package-level predicate function like `apierrors.IsNotFound`) at API boundaries you don't own, because a *behavioural* check survives what an *identity* check cannot.

### Classification: turning errors into control flow

Everything above exists to support one decision, made once, at the boundary:

```go
type Disposition int

const (
	Done      Disposition = iota // success
	Skip                         // expected absence — success, stop
	Retry                        // transient — requeue with backoff
	Terminate                    // unfixable — record a condition, do NOT requeue
	Abandon                      // caller gave up (ctx cancelled) — not our failure
)

func classify(err error) (Disposition, time.Duration) {
	switch {
	case err == nil:
		return Done, 0
	case errors.Is(err, context.Canceled), errors.Is(err, context.DeadlineExceeded):
		return Abandon, 0
	case errors.Is(err, ErrNamespaceNotFound):
		return Skip, 0
	case errors.Is(err, Terminal(nil)):
		return Terminate, 0
	}
	var te *TransientError
	if errors.As(err, &te) {
		return Retry, te.RetryAfter // the typed error carries the delay
	}
	return Terminate, 0 // default: unknown errors are NOT retried forever
}
```

Two design decisions in that function are worth arguing about explicitly, because the default choice is usually wrong:

- **`Abandon` comes first.** A cancelled context is not an application failure; it means the caller stopped caring. Logging it at error level is how you get an incident channel full of noise from clients that simply disconnected.
- **The fallback is `Terminate`, not `Retry`.** Unknown errors retried forever is the "over-classification collapse" failure: one permanently invalid object consumes a worker slot and API-server QPS indefinitely, and nothing pages because the process is healthy. Retrying should be something you opt *into*, by producing a `TransientError` at the layer that knows the failure is transient.

Here is the real controller-runtime code your `classify` mirrors (v0.24.1, `pkg/internal/controller/controller.go`, `reconcileHandler`), lightly trimmed to the decision:

```go
result, err := c.Reconcile(ctx, req)
switch {
case err != nil:
	if errors.Is(err, reconcile.TerminalError(nil)) {
		ctrlmetrics.TerminalReconcileErrors.WithLabelValues(c.Name).Inc() // counted, NOT requeued
	} else {
		c.Queue.AddWithOpts(priorityqueue.AddOpts{RateLimited: true, Priority: new(priority)}, req)
	}
	ctrlmetrics.ReconcileErrors.WithLabelValues(c.Name).Inc()
	log.Error(err, "Reconciler error")
case result.RequeueAfter > 0:
	c.Queue.Forget(req)                                   // reset the backoff counter
	c.Queue.AddWithOpts(priorityqueue.AddOpts{After: result.RequeueAfter, …}, req)
default:
	c.Queue.Forget(req)                                   // success: stop retrying this key
}
```

```
   your Reconcile returns …          controller-runtime does …               effect
   ─────────────────────────────────────────────────────────────────────────────────────
   (Result{}, nil)             ──▶   Queue.Forget(req)                  stop; failure
                                                                        counter reset

   (Result{RequeueAfter: d},   ──▶   Queue.Forget(req)                  re-run in exactly d;
    nil)                             Queue.AddAfter(req, d)             backoff counter reset

   (_, err)  err is ordinary   ──▶   Queue.AddRateLimited(req)          re-run after
                                     failures[req]++                    5ms · 2^failures
                                                                        (cap 1000s)

   (_, err)  errors.Is(err,    ──▶   TerminalReconcileErrors++          NOT requeued;
    TerminalError(nil))              (no Add at all)                    only a watch event
                                                                        will bring it back

   panic inside Reconcile      ──▶   recover() → err = "panic: … [recovered]"
                                     → falls into the ordinary-error branch above
   ─────────────────────────────────────────────────────────────────────────────────────
   Note: if err != nil, Result is IGNORED — RequeueAfter alongside an error is dropped
   (controller-runtime logs a warning telling you so).
```

Three practical rules fall straight out of that table:

1. **`return ctrl.Result{}, err` and `return ctrl.Result{RequeueAfter: d}, nil` are different tools.** The first means "this failed, back off"; the second means "this succeeded, check again later." Returning both is a bug the library warns about.
2. **`client.IgnoreNotFound(err)` is the `Skip` disposition, spelled idiomatically.** Its whole body is `if apierrors.IsNotFound(err) { return nil }; return err` — the object was deleted, there is nothing to reconcile, do not requeue.
3. **A recovered panic becomes an ordinary error.** `RecoverPanic` defaults to true; the deferred recover sets `err = fmt.Errorf("panic: %v [recovered]", r)`, which then takes the rate-limited requeue path. So a reconciler that panics on one malformed object doesn't crash the manager — it re-runs, panics again, and hot-loops on that key forever. The safety net keeps the process up; it does not fix your bug.

### Backoff: the actual numbers

"Requeue with exponential backoff" is a phrase; here are the constants behind it.

controller-runtime's default queue rate limiter is client-go's `workqueue.DefaultTypedControllerRateLimiter` (client-go v0.36.0), which is the **max of two** limiters:

| Limiter | Parameters | Scope |
|---|---|---|
| `ItemExponentialFailureRateLimiter` | base **5 ms**, cap **1000 s** (16m40s) | per item (per object key) |
| `BucketRateLimiter` (`rate.Limiter`) | **10 qps**, burst **100** | whole queue |

The per-item delay is `base × 2^failures` with **no jitter**:

```go
exp := r.failures[item]
r.failures[item] = r.failures[item] + 1
backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
```

Computed schedule for a key that keeps failing:

```
controller-runtime default per-item backoff (5ms base, 1000s cap):
  requeue #1  delay=5ms        cumulative=5ms
  requeue #2  delay=10ms       cumulative=15ms
  requeue #3  delay=20ms       cumulative=35ms
  requeue #4  delay=40ms       cumulative=75ms
  requeue #5  delay=80ms       cumulative=155ms
  requeue #6  delay=160ms      cumulative=315ms
  ...
  requeue #18 delay=10m55.36s  cumulative=21m50.715s
  requeue #19 delay=16m40s     cumulative=38m30.715s
  requeue #20 delay=16m40s     cumulative=55m10.715s
```

Read that carefully, because it explains two very different incidents:

- **The first six retries happen inside 315 ms.** For a genuinely transient blip, that's excellent — you recover before anyone notices. For an object that will *never* succeed, you have just fired six pointless API calls, and you'll fire twelve more before the delay reaches minutes. Multiply by 5,000 broken objects and the API server feels it. This is exactly why `TerminalError` exists.
- **After ~22 minutes of failures the delay pins at 16m40s.** A key that broke and then silently healed can wait up to 16 minutes before anyone notices, *unless* a watch event arrives (which calls `Forget` and resets the counter). "The controller took 15 minutes to notice" is usually this, not a stuck cache.

The counter is per key and is reset by `Queue.Forget(req)` on any successful reconcile — so intermittent success genuinely resets the ladder.

**Jitter** is the piece the per-item limiter deliberately doesn't do, and understanding why matters at fleet scale. If 400 replicas of a controller all fail at the same instant (a control-plane blip), a jitter-free exponential schedule makes all 400 retry at the same instants forever — you convert one outage into a periodic self-inflicted thundering herd. Two things save you here: the queue-wide token bucket (10 qps, burst 100) smooths a single process's bursts, and Kubernetes' own retry helpers add explicit jitter. `k8s.io/apimachinery/pkg/util/wait.Jitter` is:

```go
wait := duration + time.Duration(rand.Float64()*maxFactor*float64(duration))
```

— always additive, never negative, up to `maxFactor × duration` extra. And client-go ships two named backoff profiles you should reuse rather than reinvent (client-go v0.36.0, `util/retry`):

| Profile | Steps | Base duration | Factor | Jitter | Intended for |
|---|---|---|---|---|---|
| `retry.DefaultRetry` | 5 | 10 ms | 1.0 (flat) | 0.1 (+0–10%) | optimistic-concurrency conflicts on a resource several clients edit |
| `retry.DefaultBackoff` | 4 | 10 ms | 5.0 | 0.1 (+0–10%) | an unrelated modification to a resource under active management by other controllers |

`retry.RetryOnConflict(retry.DefaultRetry, fn)` is literally `retry.OnError(backoff, apierrors.IsConflict, fn)` — a read-modify-write loop for HTTP 409s. Use it; do not hand-roll a conflict loop.

The general classification, which is where this stops being a Kubernetes topic:

| Signal | Disposition | Why |
|---|---|---|
| HTTP 429 / `IsTooManyRequests` | Retry, honour `Retry-After` | server is telling you when to come back; `apierrors.SuggestsClientDelay(err)` extracts `Details.RetryAfterSeconds` |
| HTTP 503, connection reset, EOF, DNS failure | Retry with backoff | infrastructure, likely to heal |
| HTTP 409 / `IsConflict` | Retry immediately-ish, after re-reading | optimistic concurrency: your copy was stale, refetch and reapply |
| HTTP 404 / `IsNotFound` | Skip | object is gone; nothing to reconcile |
| HTTP 400, 422, validation failure | Terminate | the same request will fail identically forever |
| `context.Canceled` | Abandon | caller gave up |
| `context.DeadlineExceeded` | Abandon, or Retry with a *longer* budget | ambiguous: the work may have partially happened |

### Context errors deserve their own branch

`context.Canceled` and `context.DeadlineExceeded` are package-level sentinels (`src/context/context.go`), and `DeadlineExceeded`'s concrete type also implements `Timeout() bool` and `Temporary() bool`, so it satisfies `net.Error` and code that checks `err.(interface{ Timeout() bool })` sees it as a timeout.

Two habits to build now, both completed in [Lesson 4](04-concurrency-and-context.md):

- **Never let them fall into your `Terminate` default.** A cancelled scrape is not a broken namespace.
- **`ctx.Err()` tells you *what*, `context.Cause(ctx)` tells you *why*.** `context.WithCancelCause` (Go 1.20) lets a canceller attach a reason, and `context.Cause` retrieves it; if there is no cause, it falls back to `ctx.Err()`. This is how you distinguish "shutting down" from "budget exhausted" from "the parent request was aborted" in a fan-out, where every goroutine otherwise reports the identical `context canceled`.

`DeadlineExceeded` is the ambiguous one: a request that timed out may still have been executed by the server. For anything non-idempotent (a `Create`), a deadline is not automatically a safe retry — this is the level-triggered reconcile design's actual payoff, since a re-read of live state answers "did it happen?" without you having to guess.

### Panic vs return

Mechanically, `panic(v)` stops normal execution of the current function, runs its deferred functions, then does the same for its caller, and so on up the goroutine's stack; if nothing calls `recover()`, the runtime prints the value and the goroutine stacks and exits with status 2. `recover()` only does anything when called **directly inside a deferred function** of a panicking frame; it returns the panic value and resumes normal unwinding from that point. Critically: **a panic in any goroutine crashes the whole process** unless that goroutine's own stack recovers it — a `recover` in `main` does not protect a worker.

The rule, and it is not a preference: **panic for programmer bugs and unrecoverable startup failure; return errors for everything a caller could act on.** The Uber Go style guide's "Don't Panic" section says it exactly: *"Panic/recover is not an error handling strategy. A program must panic only when something irrecoverable happens such as a nil dereference. An exception to this is program initialization: bad things at program startup that should abort the program may cause panic."*

Acceptable in a controller: a nil dependency that must have been wired in `main`; `MustRegister` on a Prometheus collector at startup; an unreachable `default:` in a switch over your own closed enum; `regexp.MustCompile` on a constant pattern. Not acceptable: a missing object, an API conflict, a malformed input record, a network failure — every one of those is a value a caller can classify.

And as shown above, controller-runtime's per-reconcile `recover` (default on) is a **containment** mechanism, not a design pattern: it converts your panic into a rate-limited requeue and a metric, then lets it happen again on the next attempt. `ReconcilePanics` is a metric worth alerting on precisely because it means "a code bug is now a retry loop."

### Handle each error exactly once

The last discipline is the one reviewers notice most and the easiest to get wrong at 2am. For any given failure, exactly one layer should *handle* it — log it, record a condition, decide the disposition. Every layer below should either return it unchanged or add context and return.

- **Don't log and return.** The caller will handle it, and now you have the same root cause printed once per frame. In a five-deep call stack, one failure becomes five log lines, five index entries, and five chances to mislead whoever greps.
- **Add context that the caller doesn't already have.** `fmt.Errorf("read cost export %q: %w", path, err)` adds the path. `fmt.Errorf("failed to read file: %w", err)` adds nothing — it just makes the message longer. Skip "failed to"; every error is a failure.
- **Wrap at layer boundaries, not at every return.** Ideal message: one clause per meaningful boundary, reading outside-in as a path: `scrape node gpu-node-7: decode cost export: query "gone": namespace not found`.
- **Log once, at the top — with the whole chain.** In a controller that's `reconcileHandler`'s `log.Error(err, "Reconciler error")`; in an HTTP server it's the handler; in a CLI it's `main`.

## Perspectives

**Developer view.** Error wrapping is a design decision about what the caller may know — `%w` says "you may inspect my cause," `%v` says "you may not." A senior engineer reads a codebase's `%w`/`%v` choices as a map of its trust boundaries: which layers expose their internals to callers, and which deliberately don't. Reviewing a diff's error-handling isn't just "did they check `err`" — it's "did they wrap correctly, given who's going to catch this," and "is this the layer that should be logging it." The `%v`-vs-`%w` transcript above is the reviewer's nightmare in one image: identical output, inverted behaviour, no compiler help.

**Operator view.** In a reconcile loop, the disposition assigned to an error becomes production behavior directly — get it wrong and you either hot-loop retrying an unfixable validation error, burning API server QPS, or silently drop a transient blip that should retry. The numbers make it concrete: six retries inside 315 ms, eighteen before the delay reaches minutes, then a 16m40s ceiling. A controller stuck at 100% CPU is usually an unfixable object being retried; a controller that "took 15 minutes to notice" is usually a key sitting at the backoff ceiling with no watch event to `Forget` it.

**Reliability / systems view.** Retry-with-backoff is a distributed-systems pattern independent of Go — the classification of retryable (429/503/conflict) vs. terminal (400/422/404-on-a-well-formed-request) generalizes across every API client you'll ever write, GPU-cost billing APIs included. Note what the Kubernetes stack actually does: the per-item limiter has **no jitter**, and the queue-wide token bucket (10 qps, burst 100) plus explicit `wait.Jitter` in the retry helpers is where herd-smoothing comes from. If you build your own retry loop, jitter is yours to add — `duration + rand()*factor*duration` — because a fleet retrying in lockstep turns one control-plane blip into a periodic self-inflicted second outage.

**Failure-mode view.** The most expensive error-handling mistake at scale isn't a missing `if err != nil` — it's over-classification collapse: treating every non-nil error as "retry forever," which turns one permanently bad object (a CRD spec that will never validate) into an infinite, silent, CPU-burning retry loop that never pages anyone, because from the outside "the controller is running fine." The structural defence is a classifier whose *default* is terminal and whose retry branch requires a positive typed signal. The observability defence is separate metrics for retryable and terminal errors — which is exactly why controller-runtime increments `ReconcileErrors` and `TerminalReconcileErrors` separately.

## Real-world use cases

- **controller-runtime's own reconcile loop — `pkg/internal/controller/controller.go` (v0.24.1)** — <https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/internal/controller/controller.go> — the production code that turns your returned `error` into queue behaviour: `errors.Is(err, reconcile.TerminalError(nil))` selects between "count it and stop" and "rate-limited requeue"; success paths call `Queue.Forget(req)` to reset the failure counter; a `RequeueAfter` returned *alongside* a non-nil error is dropped with a warning. What it shows: your `classify()` is not an analogy for the library's behaviour — it is the same switch, and the library's version is 20 lines you can read in full.

- **The recovered-panic hot loop — same file, `Controller.Reconcile`** — the deferred recover (enabled by default; `RecoverPanic` defaults to true) turns a panic into `err = fmt.Errorf("panic: %v [recovered]", r)` and increments `ReconcilePanics`. That error then follows the ordinary rate-limited requeue path. What it shows: the safety net keeps the manager alive but converts a code bug into an infinite retry on one object — and it is why "we recover panics, so a nil map write is fine" is wrong. The symptom is a `ReconcilePanics` counter climbing steadily with no crash and no restart.

- **`apimachinery`'s `IsNotFound` / `reasonAndCodeForError` (v0.36.0)** — <https://github.com/kubernetes/apimachinery/blob/master/pkg/api/errors/errors.go> — the whole Kubernetes "is this a 404/409/429?" family is built on `errors.As(err, &status)` against the one-method `APIStatus` **interface**, with an HTTP-code fallback for reasons the client doesn't know. What it shows: (a) `errors.As` targeting an interface is the idiomatic way to ask "does this error behave like X"; (b) those helpers keep working through your `fmt.Errorf("…: %w", err)` layers and stop working the moment one layer uses `%v` — which is the single most common cause of "why didn't `IgnoreNotFound` ignore it?"

- **Uber Go Style Guide, "Don't Panic" and "Error Types"** — <https://github.com/uber-go/guide/blob/master/style.md> — a widely adopted production standard that states outright: *"Panic/recover is not an error handling strategy. A program must panic only when something irrecoverable happens such as a nil dereference. An exception to this is program initialization…"* — plus the error-construction matrix reproduced above and the "handle errors once" rule. What it shows: a large Go shop independently arrived at the same panic/return boundary and the same sentinel-vs-typed split this lesson teaches, which is a decent signal it isn't a matter of taste.

## Worked example

A classifier for the exporter's fetch step, with three real wrap layers, a fan-out that joins failures, and a disposition for every outcome — the reconcile pattern in miniature. This program compiles and runs as-is.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"math"
	"time"
)

// ---- error vocabulary -------------------------------------------------------

// Sentinel: the caller only needs identity.
var ErrNamespaceNotFound = errors.New("namespace not found")

// Typed: the caller needs fields (which op, how long to wait).
type TransientError struct {
	Op         string
	RetryAfter time.Duration
	Cause      error
}

func (e *TransientError) Error() string {
	return fmt.Sprintf("%s: transient (retry after %s): %v", e.Op, e.RetryAfter, e.Cause)
}
func (e *TransientError) Unwrap() error { return e.Cause }

// Terminal marks an error as "never retry" — the same shape controller-runtime
// uses for reconcile.TerminalError, including the Is method used as a type token.
type terminalError struct{ err error }

func Terminal(err error) error { return &terminalError{err: err} }

func (e *terminalError) Error() string {
	if e.err == nil {
		return "nil terminal error"
	}
	return "terminal error: " + e.err.Error()
}
func (e *terminalError) Unwrap() error { return e.err }
func (e *terminalError) Is(target error) bool {
	t := &terminalError{}
	return errors.As(target, &t) // true when the TARGET is any *terminalError
}

// ---- classification ---------------------------------------------------------

type Disposition int

const (
	Done Disposition = iota
	Skip
	Retry
	Terminate
	Abandon
)

func (d Disposition) String() string {
	return [...]string{"Done", "Skip", "Retry", "Terminate", "Abandon"}[d]
}

func classify(err error) (Disposition, time.Duration) {
	switch {
	case err == nil:
		return Done, 0
	case errors.Is(err, context.Canceled), errors.Is(err, context.DeadlineExceeded):
		return Abandon, 0 // the caller gave up; not our failure to log loudly
	case errors.Is(err, ErrNamespaceNotFound):
		return Skip, 0 // deleted namespace — success, stop
	case errors.Is(err, Terminal(nil)):
		return Terminate, 0 // explicitly marked unfixable
	}
	var te *TransientError
	if errors.As(err, &te) {
		return Retry, te.RetryAfter // typed error carries the delay
	}
	return Terminate, 0 // DEFAULT IS TERMINAL: retry must be opted into
}

// ---- three wrap layers: queryBackend -> decodeCost -> fetchCost -------------

func queryBackend(ns string) (float64, error) {
	switch ns {
	case "gone":
		return 0, fmt.Errorf("GET /api/v1/namespaces/%s: %w", ns, ErrNamespaceNotFound)
	case "flaky":
		return 0, &TransientError{Op: "GET /metrics", RetryAfter: 2 * time.Second,
			Cause: errors.New("connection reset by peer")}
	case "malformed":
		return 0, Terminal(fmt.Errorf("field usd: %w", errors.New(`invalid float "n/a"`)))
	case "slow":
		return 0, fmt.Errorf("GET /metrics: %w", context.DeadlineExceeded)
	}
	return 41.0, nil
}

func decodeCost(ns string) (float64, error) {
	usd, err := queryBackend(ns)
	if err != nil {
		return 0, fmt.Errorf("decode %q: %w", ns, err) // %w, not %v
	}
	return usd, nil
}

func fetchCost(ns string) (float64, error) {
	usd, err := decodeCost(ns)
	if err != nil {
		return 0, fmt.Errorf("fetchCost %q: %w", ns, err)
	}
	return usd, nil
}

// ---- fan-out with errors.Join ----------------------------------------------

func scrapeAll(namespaces []string) (map[string]float64, error) {
	out := make(map[string]float64, len(namespaces))
	var errs []error
	for _, ns := range namespaces {
		usd, err := fetchCost(ns)
		if err != nil {
			errs = append(errs, err) // collect, keep going: partial data is useful
			continue
		}
		out[ns] = usd
	}
	return out, errors.Join(errs...) // nil if errs is empty — no guard needed
}

func backoffSchedule(base, max time.Duration, n int) {
	total := time.Duration(0)
	for i := 0; i < n; i++ {
		d := time.Duration(float64(base) * math.Pow(2, float64(i)))
		if d > max {
			d = max
		}
		total += d
		if i < 6 || i >= n-3 {
			fmt.Printf("  requeue #%-2d delay=%-10s cumulative=%s\n", i+1, d, total.Round(time.Millisecond))
		} else if i == 6 {
			fmt.Println("  ...")
		}
	}
}

func main() {
	for _, ns := range []string{"prod", "gone", "flaky", "malformed", "slow"} {
		usd, err := fetchCost(ns)
		d, after := classify(err)
		fmt.Printf("%-10s usd=%-6.2f disposition=%-10s retryAfter=%-3s err=%v\n", ns, usd, d, after, err)
	}

	fmt.Println()
	totals, err := scrapeAll([]string{"prod", "gone", "flaky", "malformed"})
	fmt.Printf("collected %d namespaces: %v\n", len(totals), totals)
	fmt.Println("joined error:")
	fmt.Println(err)
	var te *TransientError
	fmt.Println("errors.As(joined, &te) =", errors.As(err, &te))
	fmt.Println("  -> Op =", te.Op, "RetryAfter =", te.RetryAfter)
	fmt.Println("errors.Is(joined, ErrNamespaceNotFound) =", errors.Is(err, ErrNamespaceNotFound))
	fmt.Println("errors.Is(joined, Terminal(nil))        =", errors.Is(err, Terminal(nil)))

	fmt.Println()
	fmt.Println("controller-runtime default per-item backoff (5ms base, 1000s cap):")
	backoffSchedule(5*time.Millisecond, 1000*time.Second, 20)
}
```

Captured run (go1.24.7):

```
prod       usd=41.00  disposition=Done       retryAfter=0s  err=<nil>
gone       usd=0.00   disposition=Skip       retryAfter=0s  err=fetchCost "gone": decode "gone": GET /api/v1/namespaces/gone: namespace not found
flaky      usd=0.00   disposition=Retry      retryAfter=2s  err=fetchCost "flaky": decode "flaky": GET /metrics: transient (retry after 2s): connection reset by peer
malformed  usd=0.00   disposition=Terminate  retryAfter=0s  err=fetchCost "malformed": decode "malformed": terminal error: field usd: invalid float "n/a"
slow       usd=0.00   disposition=Abandon    retryAfter=0s  err=fetchCost "slow": decode "slow": GET /metrics: context deadline exceeded

collected 1 namespaces: map[prod:41]
joined error:
fetchCost "gone": decode "gone": GET /api/v1/namespaces/gone: namespace not found
fetchCost "flaky": decode "flaky": GET /metrics: transient (retry after 2s): connection reset by peer
fetchCost "malformed": decode "malformed": terminal error: field usd: invalid float "n/a"
errors.As(joined, &te) = true
  -> Op = GET /metrics RetryAfter = 2s
errors.Is(joined, ErrNamespaceNotFound) = true
errors.Is(joined, Terminal(nil))        = true
```

Reading it against the mechanisms:

- **Every sentinel and typed error was found through three `fmt.Errorf(…%w…)` layers.** `errors.Is(err, ErrNamespaceNotFound)` is true for `gone` even though the sentinel is the fourth node in the chain. Change any one of those three `%w` to `%v` and `gone` reclassifies from `Skip` to `Terminate` — the exporter would treat a deleted namespace as a permanent failure and record a false condition.
- **`flaky` produced a `Retry` *with a delay carried in the error itself*.** That is the entire argument for typed errors over sentinels: the layer that knew the backend said "come back in 2s" was able to say so, and the classifier three frames up read it without parsing a string.
- **`malformed` hit `Terminate` via the `Is`-method type token**, matching how controller-runtime detects its own `TerminalError`.
- **`slow` hit `Abandon` before any other branch**, because the context check is first. Had it fallen through, a client that hung up would have been recorded as a broken namespace.
- **`scrapeAll` returned one namespace of good data *and* three preserved causes.** `errors.As(joined, &te)` reached into the second branch of the tree and pulled out the `*TransientError` with its fields intact; `errors.Is` found the sentinel in the first branch and the terminal marker in the third. A single `%w` chain could have carried exactly one of those.
- **The backoff table is the cost of getting classification wrong**, in wall-clock: six retries in the first 315 ms, ~22 minutes to reach the ceiling, then one attempt every 16m40s.

## Practice

Add real error semantics to the [lesson 1](01-syntax-and-types.md) cost tool, feeding directly into [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md).

- Define one **sentinel** (`ErrRecordNotFound` or `ErrNamespaceNotFound`) and one **typed** error (`ParseError{Line int, Cause error}` and/or `TransientError{Op, RetryAfter, Cause}`). Give the typed errors a pointer receiver and an `Unwrap() error`.
- Wrap failures with `fmt.Errorf("...: %w", err)` through at least **three call layers** (e.g. `readFile` → `decode` → `aggregate`). Add context each layer doesn't already have (the path, the line, the namespace) — and no "failed to".
- Write a `classify(err) (Disposition, time.Duration)` that separates **not-found (skip)** from **transient (retry)** from **terminal**, plus a first branch for `context.Canceled`/`context.DeadlineExceeded`. Make the default `Terminal`, not `Retry`, and write a comment saying why.
- For the fan-out step (reading multiple cost-export files), collect per-file failures with `errors.Join` and return **partial results plus the joined error**. Prove in a test that `errors.Is` and `errors.As` still find a specific failure buried in the joined tree.
- Add a table-driven test that asserts the disposition for each error shape. Include one case that wraps the sentinel with `%v` instead of `%w` and asserts the classification *changes* — that test is the executable version of this lesson's key transcript.
- Add an `Is(target error) bool` method to one of your error types (e.g. a `NotFoundError` that answers to your sentinel) and a test showing `errors.Is` matches through it.

**Acceptance:** a caller detects the specific sentinel through 3 wrap layers via `errors.Is`; retryable-vs-terminal is a *typed* distinction resolved by `errors.As`, not string matching; a joined multi-error still resolves individual sentinels and types correctly, and callers still get partial results; the `%v`-breaks-the-chain test passes; no error is both logged and returned anywhere in the tool; `go vet ./...` clean; `go test ./...` passes.

## Common pitfalls

1. **String-matching on `err.Error()`.** Symptom: `strings.Contains(err.Error(), "not found")` works until a dependency rewords its message or adds a prefix, then silently stops matching. Mechanism: message text is not part of any API contract; identity and type are. Fix: sentinel + `errors.Is`, typed error + `errors.As`, or a published predicate like `apierrors.IsNotFound`.
2. **Breaking the chain with `%v` at one layer.** Symptom: `errors.Is` returns `false` and the log line looks completely correct — because it is. Mechanism: `%v` formats the text and stores no reference, so no `Unwrap` link is created at that node. Fix: `%w` by default; reserve `%v` for a deliberate trust boundary, and add a test that asserts the sentinel is reachable through your public API.
3. **Wrapping at every return.** Symptom: `read cost: fetch node: get pod: context deadline exceeded: context deadline exceeded: context deadline exceeded`. Mechanism: every layer added a clause that duplicated what the inner message already said. Fix: wrap once per meaningful boundary, adding information the caller doesn't have (path, key, namespace) — never restating the operation.
4. **Logging *and* returning.** Symptom: one failure produces five near-identical log lines with different prefixes. Mechanism: each frame "helpfully" logged before returning. Fix: handle once — return with context, and log at the single layer that stops propagation (the reconcile loop, the HTTP handler, `main`).
5. **`errors.As` misuse.** Symptom: a runtime panic (`errors.As: target must be a non-nil pointer`) or a silent `false`. Mechanism: `As` requires a non-nil pointer to a type implementing `error` or to an interface — passing the value instead of its address panics, and passing `&TransientError{}` when your errors are constructed as `&TransientError{}` but declared as value type mismatches. Fix: `var te *TransientError; errors.As(err, &te)`, and keep error types pointer-only.
6. **Treating panic/recover as try/except.** Symptom: `ReconcilePanics` climbing with no crashes; one object reprocessed forever. Mechanism: controller-runtime's per-reconcile `recover` (default on) converts the panic to `fmt.Errorf("panic: %v [recovered]", r)`, which takes the rate-limited requeue path. Fix: return errors for expected failures; treat any panic metric as a code bug to fix, not a handled case.
7. **Retry-everything by default.** Symptom: 100% CPU on a controller, climbing API-server QPS, no alerts. Mechanism: an unknown error class falls into a `Retry` default, and one permanently invalid object is requeued forever. Fix: default to terminal; require a positive typed signal to retry; mark known-unfixable failures with `reconcile.TerminalError` so they're counted and dropped.
8. **Assuming `errors.Unwrap` sees a joined error.** Symptom: `errors.Unwrap(joined) == nil` and a "there's nothing in here" conclusion. Mechanism: `errors.Unwrap` calls only the singular `Unwrap() error`; `joinError` (and multi-`%w`) implement `Unwrap() []error`. Fix: use `errors.Is`/`errors.As`, which handle both, or assert on `interface{ Unwrap() []error }`.

## Self-check

- **`errors.Is` vs `errors.As` — when do you use each?**
  **Answer:** `errors.Is(err, target)` walks the error tree testing *identity* — at each node, `err == target` (when target is comparable) or `err.Is(target)` if the node defines that method. Use it for sentinels and for type tokens like `reconcile.TerminalError(nil)`: "is this specific error anywhere in here?" `errors.As(err, &target)` walks the same tree testing *assignability to a type*, and on a match assigns the node into `target` so you can read its fields. Use it for typed errors carrying data, and for interface targets ("any error that behaves like `APIStatus`"). Rule of thumb: `Is` for "which error is this?", `As` for "give me the error object so I can read it." `As` panics if `target` isn't a non-nil pointer to an error type or interface.

- **An error wrapped 3 layers deep with `%w` — how does the caller detect a specific sentinel?**
  **Answer:** `errors.Is(err, ErrNotFound)`. Each `fmt.Errorf("…: %w", inner)` builds a `*fmt.wrapError{msg, err}` whose `Unwrap() error` returns the inner error, so the chain is a linked list of structs. `errors.Is` loops: compare, try the node's `Is` method, follow `Unwrap`, repeat — depth is irrelevant. If a node has `Unwrap() []error` (from `errors.Join` or two `%w` verbs), it recurses depth-first, left to right. Had any layer used `%v`, that node would store no reference, the walk would stop there, and `Is` would return `false` — with an identical printed message.

- **When is `panic` acceptable in a long-running controller?**
  **Answer:** Only for programmer-bug invariant violations and unrecoverable startup misconfiguration: a nil dependency that must have been wired in `main`, `MustRegister`/`MustCompile` on constants, a genuinely unreachable `default:`, or a fatal config error at boot where failing fast is correct. Never for expected per-item failures — missing object, API conflict, transient network error, malformed input — those must be returned so the loop can classify and requeue. controller-runtime does recover panics per reconcile (`RecoverPanic` defaults to true) and converts them to `fmt.Errorf("panic: %v [recovered]", r)`, but that means the panicking object is then requeued with backoff and panics again: containment, not a fix. Also note a panic in a goroutine you spawned yourself is *not* covered and takes down the process.

- **Why can `errors.Is` fail to match two sentinel errors that print the identical string?**
  **Answer:** `errors.Is` compares by value identity (or a custom `Is` method), never by message text, and `errors.New` returns a distinct `*errorString` pointer per call — that's deliberate, so that two errors with the same wording aren't accidentally equal. If two versions of a dependency are both in the build (a `/v2` major-version path alongside `/v1`, or a fork alongside the original — module resolution treats those as different modules), each defines its own package-level `ErrNotFound` variable, and `errors.Is` correctly reports no match between them. Diagnose with `go mod graph`; defend by preferring typed errors with an `Is` method, or published predicate functions, at boundaries you don't own.

- **Why does `errors.Join` matter for a fan-out scrape across many nodes, versus wrapping with a single `%w`?**
  **Answer:** `fmt.Errorf` with one `%w` builds a linear chain that carries exactly one cause; scraping 50 nodes where 3 fail and wrapping only the last silently discards two failures. `errors.Join` builds a `*joinError` with `Unwrap() []error`, and `errors.Is`/`errors.As` search every branch depth-first — so a caller can still detect any of the three original causes and extract typed fields from any of them, and `Error()` prints all three, newline-separated. It also discards nils and returns nil when everything succeeded, so `return results, errors.Join(errs...)` needs no guard and naturally expresses "partial success with a list of what broke." Caveat: `errors.Unwrap()` (the function) returns nil for it, because it only calls the singular `Unwrap() error`.

- **What exactly happens to a requeue when your `Reconcile` returns a non-nil error, and how fast does it come back?**
  **Answer:** controller-runtime's `reconcileHandler` checks `errors.Is(err, reconcile.TerminalError(nil))` first: if it matches, the error is counted in `TerminalReconcileErrors` and **not** requeued. Otherwise the request is re-added with `RateLimited: true`, which consults client-go's default limiter — the max of a per-item exponential (base 5 ms, doubling per failure, capped at 1000 s) and a queue-wide token bucket (10 qps, burst 100). So the first six retries land inside ~315 ms, the delay reaches minutes after about a dozen failures, and pins at 16m40s after ~22 minutes of continuous failure. Any successful reconcile calls `Queue.Forget(req)`, resetting the counter. Also: if `err != nil`, a `Result{RequeueAfter: …}` returned alongside it is ignored, and the library logs a warning saying so.

## Connections & what's next

This lesson's `error` interface is the smallest, most-used interface in the entire language — [Lesson 3 (Interfaces & Composition)](03-interfaces-and-composition.md) picks it up as the running example for what makes a good small interface, and it is where the typed-nil trap (an `error` variable holding a nil `*MyError`, so `err != nil` is true with no error present) gets its full mechanical explanation. The `context.DeadlineExceeded`/`context.Canceled` classification here is unfinished business that [Lesson 4 (Concurrency & Context)](04-concurrency-and-context.md) completes, once goroutines, cancellation propagation, `context.Cause`, and `errgroup`'s first-error semantics are on the table — and `errors.Join` is exactly the shape a bounded fan-out returns. The sentinel-identity-across-versions trap connects forward to [Lesson 6 (Modules & Layout)](06-modules-and-layout.md), where module versioning and `go mod graph` are covered directly. And the `classify()` disposition pattern built here *is* the shape of a real `Reconcile()` method — [Lesson 9 (Controller primer)](09-controller-primer.md) is where it stops being a toy and starts governing actual requeues, `TerminalError`s, and status conditions.

Next: [03 · Interfaces & Composition](03-interfaces-and-composition.md) — the highest-emphasis lesson in the module, where `error`'s one-method shape becomes the template for designing every other interface you'll write for a controller.

## References & further reading

**Primary sources**
- `errors` package documentation — <https://pkg.go.dev/errors> — the authoritative contract for `Is`, `As`, `Join`, `Unwrap`, including the exact tree-traversal wording ("err itself, followed by … a depth-first traversal of its children") and the conditions under which `As` panics.
- Go source, `src/errors/wrap.go` — <https://github.com/golang/go/blob/master/src/errors/wrap.go> — the ~40 lines implementing `Is`/`As`. Read it once; every question about matching order, custom `Is` methods, and `Unwrap() []error` recursion is answered there.
- Go source, `src/errors/join.go` — <https://github.com/golang/go/blob/master/src/errors/join.go> — `Join` discards nils, returns nil if all are nil, and formats children newline-separated; `joinError` has `Unwrap() []error` and no singular `Unwrap`.
- Go source, `src/fmt/errors.go` — <https://github.com/golang/go/blob/master/src/fmt/errors.go> — what `%w` actually builds: `errors.New` for zero verbs, `*wrapError` for one, `*wrapErrors` (with `Unwrap() []error`) for two or more.
- The Go Blog — "Working with Errors in Go 1.13" — <https://go.dev/blog/go1.13-errors> — the design rationale for `%w`, `Is`, `As`, and `Unwrap`, including why identity comparison rather than text matching. Read for intent; the mechanics are above.
- Go 1.20 release notes — <https://go.dev/doc/go1.20> — the release that added `errors.Join`, multiple `%w` verbs in one `fmt.Errorf`, and `context.WithCancelCause`/`context.Cause`.
- controller-runtime, `pkg/reconcile/reconcile.go` (v0.24.1) — <https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/reconcile/reconcile.go> — the `Reconciler` contract ("if the returned error is non-nil, the Result is ignored and the request will be requeued using exponential backoff… unless it is a `TerminalError`") and the `terminalError` type with its `Is` method used as a type token.
- controller-runtime, `pkg/internal/controller/controller.go` (v0.24.1) — <https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/internal/controller/controller.go> — `reconcileHandler`'s error switch and the deferred `recover` that converts a panic into `fmt.Errorf("panic: %v [recovered]", r)`.
- client-go, `util/workqueue/default_rate_limiters.go` (v0.36.0) — <https://github.com/kubernetes/client-go/blob/master/util/workqueue/default_rate_limiters.go> — the default limiter: per-item exponential with base 5 ms and cap 1000 s (no jitter), maxed with a 10 qps / burst 100 token bucket.
- client-go, `util/retry` (v0.36.0) — <https://github.com/kubernetes/client-go/blob/master/util/retry/util.go> — `RetryOnConflict`, and the `DefaultRetry` (5 steps, 10 ms, factor 1.0, jitter 0.1) and `DefaultBackoff` (4 steps, 10 ms, factor 5.0, jitter 0.1) profiles.
- apimachinery, `pkg/api/errors` (v0.36.0) — <https://github.com/kubernetes/apimachinery/blob/master/pkg/api/errors/errors.go> — `IsNotFound`/`IsConflict`/`IsTooManyRequests`/`SuggestsClientDelay` and the `reasonAndCodeForError` helper that uses `errors.As` against the `APIStatus` interface.

**Real-world engineering**
- Uber Go Style Guide — <https://github.com/uber-go/guide/blob/master/style.md> — what it shows: the error-construction matrix reproduced in this lesson, the `Err…`/`…Error` naming conventions, "handle errors once", and the verbatim rule *"Panic/recover is not an error handling strategy."*

**Deeper dives**
- Dave Cheney — "Don't just check errors, handle them gracefully" — <https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully> — the canonical essay behind the "handle an error exactly once" principle and the case against reflexive logging at every layer. Predates `%w` (2016), so read it for the principle, not the API.
- Learning Go, 2nd ed. (Jon Bodner) — Errors chapter — <https://www.oreilly.com/library/view/learning-go-2nd/9781098139285/> — the clearest end-to-end print treatment of sentinels, wrapping, `Is`/`As`, custom error types, and panic/recover.
- 100 Go Mistakes and How to Avoid Them (Teiva Harsanyi) — error section — <https://100go.co> — concrete anti-patterns: over-wrapping, ignoring errors, `%w` vs `%v` misuse, checking error types by string. High-density way to pre-empt exactly what reviewers flag in Go PRs.
