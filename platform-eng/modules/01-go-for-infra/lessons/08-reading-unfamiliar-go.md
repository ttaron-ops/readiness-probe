---
lesson: "01.8"
title: "Reading unfamiliar Go: orient in a large repo and trace a call path"
module: "01"
concept: "Reading unfamiliar Go: orient in a large repo and trace a call path"
status: not-started
est_time: "12h"
prev: "07-stdlib-fluency.md"
next: "09-controller-primer.md"
artifacts: []
sources: 10
---

# 01.8 · Reading unfamiliar Go: orient in a large repo and trace a call path

> **Concept.** Land in a large unknown Go repo and orient fast — `go doc`, jump-to-def, follow an interface to its implementations, read tests for intent, read history for intent — so you can read the source of any tool you operate.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 7 handed you the exporter idiom — `promauto`, a `Desc`-based `Collector`, `promhttp.HandlerFor`, four server timeouts — and pointed you at the *real* dcgm-exporter and node_exporter source it's modeled on. This lesson is the skill that makes going and actually reading that source productive rather than intimidating: orienting in an unfamiliar repo in minutes, not hours, and following a call path across package and module boundaries with confidence. It closes the loop on everything Module 01 has taught about types and interfaces (Lessons 2–3) by turning them into a *reading* superpower, not just a writing one. It unlocks Lesson 9 directly: you cannot write a controller competently against `controller-runtime` without being able to read the framework it's built on, and this lesson is where you build and rehearse that trace — the manager → informer → workqueue → `Reconcile` path — before you're relying on it under time pressure.

## Why this matters

This is the differentiator that separates a Senior Platform Engineer from an operator. Anyone can `helm install dcgm-exporter`; the senior engineer reads its source when the metrics look wrong, works out exactly which DCGM field feeds `DCGM_FI_DEV_GPU_UTIL`, and files a precise bug or patches it. On a GPU-heavy platform you operate node-exporter, dcgm-exporter, kube-state-metrics, and eventually your own controllers — all Go, all open source, all readable.

The cost of not having this skill is measured in incidents. When a page says "the operator isn't reconciling," there are exactly two responses. One is restart-and-hope, which resolves nothing and teaches nothing. The other is to open `controller-runtime` at the version you actually run and know, within minutes, where the watch→reconcile path is wired — at which point "the cache never synced because RBAC is missing a `list` verb on the CRD" is a five-minute diagnosis instead of a two-hour one, because you know that `source.Kind.Start` polls `Cache.GetInformer` every 10 s and logs a specific error when the kind isn't registered.

Coming from Python, the good news is that Go repos are *unusually* readable, and for structural reasons rather than cultural ones:

- **Dispatch is static.** In Python, `obj.collect()` could be any of a hundred `collect` methods, resolved at runtime by MRO, and `getattr(obj, name)()` could be anything at all. In Go, if a call site has type `*gpuCollector`, the compiler has already picked the method — and so can your tools. The only genuine indirection is an interface, and interfaces are enumerable.
- **The dependency source is on your disk.** `pip install` gives you a package; `go get` gives you the module's complete, read-only source tree in `$(go env GOMODCACHE)`, at the exact version your `go.mod` pins. You grep and jump into `controller-runtime` exactly as you would your own code.
- **There is roughly one way to lay out a package**, and tests live next to the code they test, in files you can find by name.

This lesson turns those three properties into a repeatable procedure, and then runs that procedure end to end on two real repositories so you can see what each step actually yields.

## What's new here (calibration)

Per the module README's skip-list, we skip cover-to-cover reading (you are never asked to read a dependency front to back) and any retread of OOP-in-Go or reflection/runtime internals — this lesson is about navigation and comprehension speed, not language mechanics you already have from Lessons 1–3.

What's genuinely new here, calibrated to staff-level depth:

- **A seven-move orientation procedure** with a stated order and a stated stop rule, timed to minutes rather than an afternoon — and then *demonstrated*, with real transcripts, on `sigs.k8s.io/controller-runtime@v0.24.1` and `NVIDIA/dcgm-exporter`.
- **Version discipline while reading** — the behaviour you're debugging is whatever's pinned in `go.mod`, not whatever GitHub's default branch shows. `go doc -src` reads your actual module cache; browsing does not.
- **`git log`/`git blame`/commit search as a reading tool**, not just a blame-assigning one — the fifth move most engineers never reach for, demonstrated on a real, checkable commit.
- **The interface-embedding blind spot** in text search, and why gopls's find-implementations doesn't have it.
- Recognizing **exported-but-internal** symbols (capitalized names under an `internal/` path) as *not* stable API, cross-referencing Lesson 6's compiler-enforced `internal/` boundary.
- A named failure mode — **"the trace that stops too early"** — plus a concrete stop rule that tells you when a trace is genuinely finished.

Every transcript below was produced against `sigs.k8s.io/controller-runtime v0.24.1` (the version that ships with kubebuilder v4.6 and targets Kubernetes 1.36) and `Go 1.24`. Your versions will differ; the *procedure* does not.

## Core concepts

### Move 0 — Pin the version before you read a single line

Everything else is worthless if you're reading the wrong code. Do this first, every time, and it takes four seconds:

```
$ go list -m sigs.k8s.io/controller-runtime
sigs.k8s.io/controller-runtime v0.24.1

$ go env GOMODCACHE
/root/go/pkg/mod

$ ls $(go env GOMODCACHE)/sigs.k8s.io/
controller-runtime@v0.24.1  controller-tools@v0.21.0  json@v0.0.0-20250730193827-2d320260d730
randfill@v1.0.0             structured-merge-diff     yaml@v1.6.0
```

Three things this buys you:

1. **The exact version string**, which you paste into every note you write. `go list -m` resolves through the full module graph including `replace` directives and MVS selection, so it is the answer to "what is actually linked into my binary," which `go.mod`'s `require` line is not (a transitive dependency can raise the selected version).
2. **A read-only source tree on disk.** The module cache directory is the real, complete source at that version. It is mode `0555` — Go makes it read-only deliberately, so you can grep and open it without any risk of editing a dependency by accident.
3. **A reason to distrust GitHub.** Browsing `main` shows you code that may be months ahead of what you run. This is the single most common source of "but the docs say X" confusion, and it is entirely avoidable.

A convenience worth putting in your shell profile:

```bash
# cdmod <module-path> — cd into the pinned source of a dependency
cdmod() { cd "$(go env GOMODCACHE)/$(go list -m -f '{{.Path}}@{{.Version}}' "$1")"; }
```

For a module whose name has capital letters, note that the cache path escapes them: `github.com/NVIDIA/dcgm-exporter` lives under `github.com/!n!v!i!d!i!a/dcgm-exporter@<version>`. Go does this because module paths are case-sensitive but many filesystems are not; `!x` is the escape for uppercase `X`. If a `ls` comes back empty, that's usually why — use `go list -m -f` to build the path rather than typing it.

### Move 1 — `go doc`, at three zoom levels

`go doc` is a compiler-backed documentation tool. It needs no browser, works offline, works on any package in your module graph, and reads from the module cache — so it is version-correct by construction. Use it at three zoom levels, in this order.

**Zoom level 1 — the package.** `go doc <pkg>` prints the package doc comment followed by every exported symbol, one line each. Five minutes here tells you what a package is *for*:

```
$ go doc sigs.k8s.io/controller-runtime/pkg/reconcile
package reconcile // import "sigs.k8s.io/controller-runtime/pkg/reconcile"

Package reconcile defines the Reconciler interface to implement Kubernetes
APIs. Reconciler is provided to Controllers at creation time as the API
implementation.

func TerminalError(wrapped error) error
type Func = TypedFunc[Request]
type ObjectReconciler[object client.Object] interface{ ... }
type Reconciler = TypedReconciler[Request]
    func AsReconciler[object client.Object](client client.Client, rec ObjectReconciler[object]) Reconciler
type Request struct{ ... }
type Result struct{ ... }
type TypedFunc[request comparable] func(context.Context, request) (Result, error)
type TypedReconciler[request comparable] interface{ ... }
```

Read that output the way a native reader does. The whole package is nine symbols. `Reconciler` is a **type alias** for `TypedReconciler[Request]` — the `=` is the giveaway, and it means the generic form is the real definition and `Reconciler` is the ergonomic name. `TerminalError` is a free function that wraps an error, which strongly implies "an error class the framework treats specially." `Request` and `Result` are the two data types crossing the boundary. That's a working map of the package in fifteen seconds.

**Zoom level 2 — one symbol.** `go doc <pkg>.<Symbol>` prints the declaration with its doc comment:

```
$ go doc sigs.k8s.io/controller-runtime/pkg/reconcile.Result
type Result struct {
	// Requeue tells the Controller to perform a ratelimited requeue
	// using the workqueues ratelimiter. Defaults to false.
	//
	// This setting is deprecated as it causes confusion and there is
	// no good reason to use it. When waiting for an external event to
	// happen, either the duration until it is supposed to happen or an
	// appropriate poll interval should be used, rather than an
	// interval emitted by a ratelimiter whose purpose it is to control
	// retry on error.
	//
	// Deprecated: Use `RequeueAfter` instead.
	Requeue bool

	// RequeueAfter if greater than 0, tells the Controller to requeue the reconcile key after the Duration.
	// Implies that Requeue is true, there is no need to set Requeue to true at the same time as RequeueAfter.
	RequeueAfter time.Duration

	// Priority is the priority that will be used if the item gets re-enqueued (also if an error is returned).
	// If Priority is not set the original Priority of the request is preserved.
	// Note: Priority is only respected if the controller is using a priorityqueue.PriorityQueue.
	Priority *int
}
    Result contains the result of a Reconciler invocation.

func (r *Result) IsZero() bool
```

Two facts fall out that no blog post would have given you at this version: `Requeue` is **deprecated**, with the maintainers' reasoning stated inline; and there is a third field, `Priority`, gated on the controller using a priority queue. If you had learned `Result` from a 2023 tutorial you'd know neither.

**Zoom level 3 — the implementation.** `go doc -src <pkg>.<Symbol>` prints the actual source, read from the module cache:

```
$ go doc -src sigs.k8s.io/controller-runtime/pkg/builder.TypedBuilder.Complete
// Complete builds the Application Controller.
func (blder *TypedBuilder[request]) Complete(r reconcile.TypedReconciler[request]) error {
	_, err := blder.Build(r)
	return err
}
```

Three lines. That is the entire body of the method most people believe does the wiring — and it is the concrete demonstration of "the trace that stops too early" (below). `-src` is the move that converts an assumption into a fact, and it costs one command.

Two more flags worth knowing: `go doc -all <pkg>` prints every exported symbol *with* its full doc comment (use it when the one-line listing isn't enough but you don't want to open files), and `go doc -u <pkg>.<Symbol>` includes unexported identifiers — which is how you read the private fields of a struct whose behaviour you're trying to explain.

### Move 2 — gopls: the four navigations that matter

`gopls` is the official Go language server; every editor with Go support drives it. Four operations carry almost all the value.

| Operation | Typical binding | What it answers | Why it's better than grep |
|---|---|---|---|
| **Jump to definition** | `gd` | "Where is this defined?" | Crosses package *and module* boundaries into the module cache. Resolves type aliases and generic instantiations to the real declaration. |
| **Find implementations** | `gi` | Given an interface: which concrete types satisfy it? Given a method: which interfaces does it satisfy? | Go satisfaction is implicit — there is no `implements` keyword to search for. gopls computes method sets from the type system, so it finds implementations gained through **embedding**, which text search cannot. |
| **Find references** | `gr` | "Who calls this?" | Distinguishes a call to *this* `Get` from the forty other `Get` methods in scope. |
| **Call hierarchy** | varies | The transitive incoming/outgoing call tree from a function. | This is the trace, computed for you — the one operation that turns a manual walk into a listing. |

The mental model shift from Python is worth stating explicitly: `help()` and `dir()` in Python *inspect a live object* — you must import the module, construct an instance, and hope the attribute exists at the moment you ask. gopls inspects a **static artifact**. Nothing runs, nothing needs to be constructible, and the answer is complete rather than a sample of one execution. The corollary is that anything genuinely dynamic in Go — a `map[string]func()` dispatch table, a `reflect`-driven decoder, a plugin loaded at runtime — is exactly where gopls goes quiet and you have to fall back to grep and reasoning. Fortunately those are rare in infrastructure Go.

**Find-implementations is the one to internalise.** It is the move that crosses the only real indirection Go has. You'll use it constantly in the walkthrough below.

### Move 3 — grep in `$GOMODCACHE`, and its one blind spot

When gopls isn't available (a stripped container, a quick check over SSH), an interface is just a set of method names, and Go's method signatures are distinctive enough to grep for directly:

```
$ CR="$(go env GOMODCACHE)/sigs.k8s.io/controller-runtime@v0.24.1"

$ grep -rn "Reconcile(ctx context.Context" --include="*.go" $CR/pkg | wc -l
7

$ grep -rln "reconcile.Reconciler" $CR/pkg
pkg/controller/controller.go
pkg/internal/controller/controller.go
```

That second result is already a finding: only two files in the entire library mention the `Reconciler` type by name. One is the public constructor package, one is the internal implementation. You now know where the worker loop lives without having read anything.

**The blind spot is embedding.** A type can satisfy an interface without declaring the method itself:

```go
type baseReconciler struct{ client.Client }

func (b *baseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) { /* ... */ }

// GPUCostReconciler satisfies reconcile.Reconciler — but grepping for
// "func (r *GPUCostReconciler) Reconcile" finds absolutely nothing.
type GPUCostReconciler struct{ baseReconciler }
```

`GPUCostReconciler`'s method set includes `Reconcile`, promoted from the embedded `baseReconciler`. Text search cannot see promotion, because promotion is a type-system fact with no textual representation at the outer type. When a type you're *certain* implements an interface doesn't turn up under grep, check its struct fields for an embedded type before concluding the implementation doesn't exist — or use find-implementations, which resolves it correctly because it works off method sets, not text.

The inverse case is also worth watching for: a type that embeds `client.Client` gains `Get`, `List`, `Update`, `Patch` and a dozen more methods it never declared. When you read `r.Get(ctx, req.NamespacedName, &obj)` in a reconciler and can't find `func (r *Reconciler) Get`, that's why — jump-to-definition lands you in `client.Client` in the module cache.

### Move 4 — read the tests first

Tests encode intent, and they are the one form of documentation that **cannot drift from the version you run**, because they are compiled and executed in CI against exactly that code. A blog post explaining a library was true on the day it was written; `pkg/reconcile/reconcile_test.go` is true right now.

Three kinds of test file, in decreasing order of usefulness for orientation:

- **`example_test.go`** — functions named `Example`, `ExampleType`, or `ExampleType_Method`. These are compiled, run, and their output verified by `go test`, so they are *guaranteed-correct usage snippets*. They also appear in `go doc` output and on pkg.go.dev. Start here.
- **Table-driven tests** — the Go idiom of a `[]struct{name string; ...}` slice iterated with `t.Run(tc.name, ...)`. **The `name` strings are the contract's edge-case list.** Reading only the names of the cases in a table tells you what the authors believed could go wrong, faster than reading the implementation.
- **Suite tests** — in the Kubernetes ecosystem, Ginkgo/Gomega (`Describe`/`It`/`Expect`) is pervasive. `controller-runtime`'s own `reconcile_test.go` is Ginkgo, and its `It(...)` strings play the same role as table-test names:

```go
// pkg/reconcile/reconcile_test.go, controller-runtime v0.24.1
It("IsZero should return true if empty", func() {
    var res *reconcile.Result
    Expect(res.IsZero()).To(BeTrue())
    res2 := &reconcile.Result{}
    Expect(res2.IsZero()).To(BeTrue())
    res3 := reconcile.Result{}
    Expect(res3.IsZero()).To(BeTrue())
})
```

Note what that one test establishes and the docs do not: `IsZero` is nil-safe on the pointer receiver. That's the kind of thing you'd otherwise discover with a panic.

One operational note: files ending `_suite_test.go` are usually just the Ginkgo bootstrap (`RunSpecs`) and occasionally an `envtest` setup — skip them when orienting, they contain no domain information.

### Move 5 — read the history

A file's history answers "*why* is it like this?", which the current snapshot structurally cannot. This is available on any dependency you've cloned, and via commit search on any public repo, and it is the move most engineers forget exists.

```
# In a clone of the dependency (git clone, then check out the tag you run):
$ git log --oneline -- pkg/reconcile/reconcile.go | head
$ git log -p --follow -- pkg/internal/controller/controller.go   # how this file evolved
$ git blame pkg/reconcile/reconcile.go -L 29,52                  # who/why for one block
$ git log -S "MaxConcurrentReconciles" --oneline                 # commits that added/removed the string
```

`git log -S` (the "pickaxe") is the one to remember: it finds commits where the *number of occurrences* of a string changed, which is how you find the commit that introduced a constant, a field, or a magic number — even if it has since moved between files.

Here is a real, checkable example. Reading `Result` above, you saw that `Requeue` is deprecated with reasoning in the doc comment. The history tells you when and by whom:

```
commit f15ff17054af279930e55d8f4eb6f0738a67d422
Author: Alvaro Aleman
Date:   2025-02-08

    :warning: Deprecate `reconcile.Result.Requeue`

    There is no good reason to use this setting, either an error or
    `RequeueAfter` should be used instead. Deprecate it to avoid confusion.
```

Merged as PR #3107 on 2025-02-24. That one commit message settles an argument that generates recurring StackOverflow noise: use an error return or `RequeueAfter`; `Requeue: true` exists only for compatibility.

The same technique explains the `Priority` field you saw on `Result`, which no tutorial covers. Searching the repo's history for "priorityqueue" finds three commits that tell a complete story:

- **PR #3014** (Dec 2024, Alvaro Aleman) added a priority workqueue, **opt-in**, whose stated purpose is to *de-prioritize events originating from the initial listwatch and from periodic resyncs*. The commit message includes the benchmark that justified a B-tree over a slice: `AddGetDone` went from 5.078 ms to 1.163 ms (−77%) with allocations dropping from ~3,000 to ~1,000 per operation.
- **PR #3111** (Mar 2025, Troy Connor) wired the priority queue through the `handler` package.
- **PR #3332** (merged 2025-10-06, Stefan Büringer) flipped it to **enabled by default**.

Now the `Priority *int` field makes sense, and so does `handler.LowPriority = -100`, and so does the behaviour you'd otherwise find baffling: after a controller restart, the flood of reconciles from the initial LIST is deliberately queued *behind* any genuine change events. That is three commits' worth of reading and it converts a mysterious field into a design you can explain in an interview.

### Move 6 — write the trace down, and know when to stop

Keep a scratch list as you go, one line per hop, in the form `file:line — Type.Method — what it does`. That list *is* the deliverable; it's also what stops you going in circles, because a repeated entry means you've looped.

**The stop rule.** A trace is finished when the next hop is one of:

1. **A syscall or the runtime** — `net.(*netFD).Read`, `runtime.selectgo`. Below this there is no more Go to read.
2. **A boundary you've decided to treat as opaque** — the API server's HTTP handler, the DCGM C library, etcd. Write down *which* boundary and what you're assuming about it.
3. **Code you already understand** — you've reached `client-go`'s `SharedIndexInformer` and you know what it does. Note the entry point and stop.

Anything else means you're not done. In particular, "I found a function whose name matches what I'm looking for" is **not** a stopping condition.

### The named failure mode: the trace that stops too early

The single most common way this skill fails is stopping at the first layer of indirection, satisfied by a plausible-sounding name.

```go
ctrl.NewControllerManagedBy(mgr).
    For(&costv1alpha1.CostBudget{}).
    Complete(r)
```

Read that and it's tempting to write in your notes: "`Complete` registers the controller with the manager and sets up the watch." That sentence is *approximately* true and completely useless — it names nothing you could act on during an incident. And you already have the disproof from Move 1: `Complete` is three lines that call `Build`.

The diagnostic question, asked at every hop: **"what does this call actually construct, and where does the thing it constructs get used?"** If you can't answer both halves with a file and a type name, go one layer deeper.

### Exported does not mean stable API

Go's `internal/` import restriction (Lesson 6) is compiler-enforced: nothing outside the parent of an `internal/` directory can import it. But **within** an `internal/` tree, symbols are still capitalized — still "exported" in the Go sense — because sibling packages inside that tree need to see them.

So `sigs.k8s.io/controller-runtime/pkg/internal/controller.Controller` is an exported type in an internal package. You can and should read it: it contains the worker loop, and understanding it is the whole point of this lesson. What you must not do is treat it as a contract. Its field layout, its method set, and its behaviour can change in a patch release with no deprecation notice owed to anyone, because the compiler already guarantees nobody outside the library depends on it.

Concretely, when you write your trace, mark those hops: `pkg/internal/controller/controller.go — Controller[request].reconcileHandler — (internal, v0.24.1 only)`. That annotation is the difference between a note that ages well and one that quietly becomes wrong.

### The navigation strategy, as a decision procedure

```
  Question: "how does X happen?"
        │
        ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │ MOVE 0  go list -m <module>   →  pin the version                │
  │         go env GOMODCACHE     →  locate the source on disk      │
  └─────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │ MOVE 1  go doc <pkg>          →  what is this package FOR?      │
  │         (read doc comment + scan exported symbol names)         │
  └─────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │ Find the ENTRY POINT. One of:                                   │
  │   · func main / cmd/*                    (a binary)             │
  │   · the interface YOU implement          (a framework)          │
  │   · the exported constructor you call    (a library)            │
  └─────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌──────────────── walk one hop ────────────────┐
  │                                              │
  │   What kind of thing is the next symbol?     │
  │                                              │
  │   ┌─ concrete func/method ─────────────┐     │
  │   │  gd (jump to def) / go doc -src    │     │
  │   │  → read it, note file:line, recurse│     │
  │   └────────────────────────────────────┘     │
  │                                              │
  │   ┌─ INTERFACE ────────────────────────┐     │
  │   │  gi (find implementations)         │     │
  │   │  → which concrete type is used     │     │
  │   │    HERE? follow the constructor    │     │
  │   │    that produced the value, not    │     │
  │   │    the whole implementation list   │     │
  │   └────────────────────────────────────┘     │
  │                                              │
  │   ┌─ unclear / too many impls ─────────┐     │
  │   │  read *_test.go for this symbol    │     │
  │   │  → the test names ARE the contract │     │
  │   └────────────────────────────────────┘     │
  │                                              │
  │   ┌─ "why is this here at all?" ───────┐     │
  │   │  git log -S / git blame / PR search│     │
  │   └────────────────────────────────────┘     │
  │                                              │
  └───────────────┬──────────────────────────────┘
                  │
                  ▼
        STOP RULE satisfied?  ──no──▶ walk another hop
                  │ yes
                  ▼
        Write the trace: file:line — Type.Method — effect
        Mark every hop under internal/ as "not stable API"
```

Note the branch that matters most: at an **interface**, do not enumerate every implementation and read them all. Ask which concrete type is in the variable *at this call site*, and answer it by finding the constructor that produced the value. Find-implementations tells you the universe of possibilities; the constructor tells you which one you actually have.

### The demonstrated walkthrough: `controller-runtime@v0.24.1`

The question: **how does `Reconcile` get invoked when a watched object changes?** This is checkpoint item 5 of this module, and it is the mental model Lesson 9 assumes on day one. Here is the procedure run end to end, with what each stage actually yields.

---

**Stage 0 — pin.** `go list -m sigs.k8s.io/controller-runtime` → `v0.24.1`. Everything below is that version. *Conclusion: 197 non-test Go files, ~33,000 lines, plus 134 test files. Too big to read; exactly the right size to trace.*

---

**Stage 1 — the package doc, which answers three questions before you open a file.**

```
$ go doc sigs.k8s.io/controller-runtime | head -60
```

The root package's doc comment is unusually good — it is a guided tour of the library's own layout. Four passages carry real information:

> "Every controller and webhook is ultimately run by a Manager (pkg/manager). A manager is responsible for running controllers and webhooks, and setting up common dependencies, like shared caches and clients…"

> "Controllers (pkg/controller) use events (pkg/event) to eventually trigger reconcile requests. They may be constructed manually, but are often constructed with a Builder (pkg/builder), which eases the wiring of event sources (pkg/source), like Kubernetes API object changes, to event handlers (pkg/handler)…"

> "Controller logic is implemented in terms of Reconcilers (pkg/reconcile). A Reconciler implements a function which takes a reconcile Request containing the name and namespace of the object to reconcile…"

> "Reconcilers use Clients (pkg/client) to access API objects. The default client provided by the manager reads from a local shared cache (pkg/cache) and writes directly to the API server… **The default split client does not promise to invalidate the cache during writes (nor does it promise sequential create/get coherence), and code should not assume a get immediately following a create/update will return the updated resource.**"

*Conclusions, in order:* (a) the package names are the map — `manager`, `builder`, `source`, `handler`, `reconcile`, `client`, `cache`; (b) `Builder` is a convenience over a manual path, so the *real* wiring is one layer under it; (c) `Request` carries a name and namespace, not an object — which is the level-triggered contract, stated in the first paragraph you read; (d) the last passage is a primary-source warning about stale reads that most operators learn the hard way. Four minutes of reading, and you have the shape of the entire library plus one production gotcha.

---

**Stage 2 — find the entry point: the interface *you* implement.** For a framework, that's the right entry point, because it's the one contract you cannot avoid.

```
$ go doc sigs.k8s.io/controller-runtime/pkg/reconcile.TypedReconciler
type TypedReconciler[request comparable] interface {
	// Reconcile performs a full reconciliation for the object referred to by the Request.
	//
	// If the returned error is non-nil, the Result is ignored and the request will be
	// requeued using exponential backoff. The only exception is if the error is a
	// TerminalError in which case no requeuing happens.
	//
	// If the error is nil and the returned Result has a non-zero result.RequeueAfter, the request
	// will be requeued after the specified duration.
	//
	// If the error is nil and result.RequeueAfter is zero and result.Requeue is true, the request
	// will be requeued using exponential backoff.
	Reconcile(context.Context, request) (Result, error)
}
```

*Conclusion:* the entire return-value contract is documented on the interface. Three outcomes, in strict precedence: error (unless `TerminalError`) → backoff requeue; `RequeueAfter > 0` → timed requeue; `Requeue` → backoff requeue. **Write this down — it is the whole of Lesson 9's `Result` semantics, from the primary source, before you've read any implementation.** Also note the precedence detail: on a non-nil error the `Result` is *ignored entirely*, so returning `(Result{RequeueAfter: 5*time.Minute}, err)` silently drops the five minutes.

The package doc also gives you the level-triggered statement verbatim:

> "Reconciliation is level-based, meaning action isn't driven off changes in individual Events, but instead is driven by actual cluster state read from the apiserver or a local cache. For example if responding to a Pod Delete Event, the Request won't contain that a Pod was deleted, instead the reconcile function observes this when reading the cluster state and seeing the Pod as missing."

---

**Stage 3 — who calls `Reconcile`?** Move 3, grep, because it's instant:

```
$ grep -rln "reconcile.Reconciler" $CR/pkg
pkg/controller/controller.go
pkg/internal/controller/controller.go
```

*Conclusion:* two files. `pkg/controller` is the public constructor; `pkg/internal/controller` is where the loop lives. Mark the second as **internal — not stable API**. This is the whole search; it took one command because Go's type names are distinctive.

---

**Stage 4 — the wiring, and the trap.** Start from what you'd write in `SetupWithManager`:

```go
ctrl.NewControllerManagedBy(mgr).For(&v1.Pod{}).Complete(r)
```

`ctrl.NewControllerManagedBy` is an alias — `alias.go` maps it to `builder.ControllerManagedBy`. Follow `Complete`:

```
$ go doc -src sigs.k8s.io/controller-runtime/pkg/builder.TypedBuilder.Complete
// Complete builds the Application Controller.
func (blder *TypedBuilder[request]) Complete(r reconcile.TypedReconciler[request]) error {
	_, err := blder.Build(r)
	return err
}
```

*Conclusion:* stopping here is the failure mode. `Complete` does nothing but delegate. Follow `Build` (`pkg/builder/controller.go:266`), which after validation does exactly two things:

```go
// Set the ControllerManagedBy
if err := blder.doController(r); err != nil { return nil, err }

// Set the Watch
if err := blder.doWatch(); err != nil { return nil, err }
```

*Conclusion:* the wiring is two independent halves — **construct the controller** (`doController`) and **register the watches** (`doWatch`). Trace both. This is the first hop where the trace becomes a tree rather than a line, which is exactly why writing it down matters.

---

**Stage 5 — `doWatch`: where a source and a handler get paired.** `pkg/builder/controller.go:307`. For the `For(...)` input:

```go
var hdler handler.TypedEventHandler[client.Object, request]
reflect.ValueOf(&hdler).Elem().Set(reflect.ValueOf(&handler.EnqueueRequestForObject{}))
allPredicates := append([]predicate.Predicate(nil), blder.globalPredicates...)
allPredicates = append(allPredicates, blder.forInput.predicates...)
src := source.TypedKind(blder.mgr.GetCache(), obj, hdler, allPredicates...)
if err := blder.ctrl.Watch(src); err != nil { return err }
```

*Conclusions, and there are several:*

- `For(...)` hard-wires `handler.EnqueueRequestForObject` — "enqueue the object the event is about." You never chose this; the builder chose it for you.
- The source is `source.TypedKind(mgr.GetCache(), obj, handler, predicates...)`. **The cache is passed in as an argument**, which is the mechanism behind "controllers don't hammer the API server": every watch in the process shares the manager's one cache, so N controllers watching Pods produce one informer, not N.
- Predicates are attached *at the source*, so filtering happens before anything reaches the queue.
- A few lines further down, `Owns(...)` uses `handler.EnqueueRequestForOwner(scheme, restMapper, forInput.object, handler.OnlyControllerOwner())` — it maps a *child's* event to the *parent's* key, and by default only when the owner reference has `controller: true`. That default (`matchEveryOwner` false) is the answer to "why doesn't my `Owns` fire for this child?"
- And the error message if you configure nothing: `"there are no watches configured, controller will never get triggered. Use For(), Owns(), Watches() or WatchesRawSource() to set them up"` — a real string you can grep for the day you hit it.

---

**Stage 6 — cross the interface boundary: what does `source.Kind` do at `Start`?** `Kind` is a constructor returning a `SyncingSource` interface, so this is the find-implementations hop. The concrete type is in `pkg/internal/source/kind.go` (internal — mark it). Its `Start(ctx, queue)`:

```go
if err := wait.PollUntilContextCancel(ctx, 10*time.Second, true, func(ctx context.Context) (bool, error) {
	i, lastErr = ks.Cache.GetInformer(ctx, ks.Type)
	if lastErr != nil {
		kindMatchErr := &meta.NoKindMatchError{}
		switch {
		case errors.As(lastErr, &kindMatchErr):
			logKind.Error(lastErr, "if kind is a CRD, it should be installed before calling Start", ...)
		case runtime.IsNotRegisteredError(lastErr):
			logKind.Error(lastErr, "kind must be registered to the Scheme")
		default:
			logKind.Error(lastErr, "failed to get informer from cache")
		}
		return false, nil // Retry.
	}
	return true, nil
}); err != nil { /* ... */ }

handlerRegistration, err := i.AddEventHandlerWithOptions(
	NewEventHandler(ctx, queue, ks.Handler, ks.Predicates), ...)

if !ks.Cache.WaitForCacheSync(ctx) {
	ks.startedErr <- errors.New("cache did not sync")
	...
}
```

*Conclusions:*

- A `source.Kind` is not itself a watch. It **asks the shared cache for an informer for this type**, retrying every 10 s until it gets one, then registers an event handler on it. The informer — and therefore the actual LIST/WATCH against the API server — belongs to the cache, not the controller.
- The three logged error messages are a diagnosis table you now own for free. `"if kind is a CRD, it should be installed before calling Start"` is the message when the CRD isn't applied yet; `"kind must be registered to the Scheme"` is the missing `AddToScheme`; the retry loop is why a manager with a missing CRD *hangs* rather than crashing, and why the symptom is silence plus a 10-second-cadence log line.
- `WaitForCacheSync` is the gate: no reconcile happens until the initial LIST has populated the cache. This is why a controller with insufficient RBAC never reconciles anything and never says why in an obvious place — the informer can't list, the cache never syncs, and the controller sits in `WaitForCacheSync`.

---

**Stage 7 — the handler: event → `Request`.** `pkg/handler/enqueue.go`. `TypedEnqueueRequestForObject.Create`:

```go
item := reconcile.Request{NamespacedName: types.NamespacedName{
	Name:      evt.Object.GetName(),
	Namespace: evt.Object.GetNamespace(),
}}
addToQueueCreate(q, evt, item)
```

*Conclusion:* **the object is thrown away here.** Only its name and namespace survive onto the queue. This is the single most load-bearing line in the whole trace, and it is three lines long: the level-triggered contract is not a design philosophy, it is a data structure — `reconcile.Request` has exactly one field, an embedded `types.NamespacedName`, and there is nowhere to put an event payload even if the framework wanted to.

One layer up, in `pkg/handler/eventhandler.go`, is the priority logic the history in Move 5 explained:

```go
if e.IsInInitialList {                                    // Create events
	wq.priority = new(LowPriority)                        // LowPriority = -100
}
// ...
if any(e.ObjectOld).(client.Object).GetResourceVersion() ==
   any(e.ObjectNew).(client.Object).GetResourceVersion() { // Update events
	wq.priority = new(LowPriority)                        // a resync, not a real change
}
```

*Conclusion:* an Update whose old and new `resourceVersion` are **identical** is a periodic resync, not a real change, and gets de-prioritised. That's how the framework distinguishes the two without being told, and it is the mechanism behind `GenerationChangedPredicate`-style filtering being unnecessary for most controllers on the priority queue.

---

**Stage 8 — the worker loop.** `pkg/internal/controller/controller.go` (internal). `Start` launches the workers:

```go
c.LogConstructor(nil).Info("Starting workers", "worker count", c.MaxConcurrentReconciles)
wg.Add(c.MaxConcurrentReconciles)
for i := 0; i < c.MaxConcurrentReconciles; i++ {
	go func() {
		defer wg.Done()
		// Run a worker thread that just dequeues items, processes them, and marks them done.
		// It enforces that the reconcileHandler is never invoked concurrently with the same object.
		for c.processNextWorkItem(ctx) {
		}
	}()
}
```

`processNextWorkItem` is `Queue.GetWithPriority()` → `defer Queue.Done(obj)` → `reconcileHandler`. And `reconcileHandler` is the payoff:

```go
result, err := c.Reconcile(ctx, req)
if result.Priority != nil { priority = *result.Priority }
switch {
case err != nil:
	if errors.Is(err, reconcile.TerminalError(nil)) {
		ctrlmetrics.TerminalReconcileErrors.WithLabelValues(c.Name).Inc()
	} else {
		c.Queue.AddWithOpts(priorityqueue.AddOpts{RateLimited: true, Priority: new(priority)}, req)
	}
	ctrlmetrics.ReconcileErrors.WithLabelValues(c.Name).Inc()
	ctrlmetrics.ReconcileTotal.WithLabelValues(c.Name, labelError).Inc()
	if result.RequeueAfter > 0 || result.Requeue {
		log.Info("Warning: Reconciler returned both a result with either RequeueAfter or Requeue set and a non-nil error. RequeueAfter and Requeue will always be ignored if the error is non-nil. ...")
	}
	log.Error(err, "Reconciler error")
case result.RequeueAfter > 0:
	c.Queue.Forget(req)
	c.Queue.AddWithOpts(priorityqueue.AddOpts{After: result.RequeueAfter, Priority: new(priority)}, req)
	ctrlmetrics.ReconcileTotal.WithLabelValues(c.Name, labelRequeueAfter).Inc()
case result.Requeue:
	c.Queue.AddWithOpts(priorityqueue.AddOpts{RateLimited: true, Priority: new(priority)}, req)
default:
	c.Queue.Forget(req)
	ctrlmetrics.ReconcileTotal.WithLabelValues(c.Name, labelSuccess).Inc()
}
```

*Conclusions:*

- The comment on the worker goroutine states the concurrency guarantee explicitly: **the same object key is never reconciled concurrently**, no matter how high `MaxConcurrentReconciles` is. That's a property of the workqueue, not of your code, and it's why you get parallelism across objects without per-object locking.
- The `switch` is the `Result` contract you read at Stage 2, now as executable code, in precedence order.
- `Queue.Forget(req)` on success resets the item's exponential-backoff counter. Forget is not "drop"; it's "reset the failure count." Missing that distinction is how people convince themselves the backoff is broken.
- There is a literal warning log for returning both an error and a `RequeueAfter`. Someone hit this often enough to add a log line — that alone is a lesson about which mistake is common.
- Every branch increments a metric labelled `error` / `requeue_after` / `requeue` / `success`. So `controller_runtime_reconcile_total{result="error"}` and `controller_runtime_reconcile_errors_total` exist for free, on every controller you write, and they are how you'd answer "is it reconciling?" during a page.

---

**Stage 9 — the rate limiter, one hop past where most traces stop.** `AddOpts{RateLimited: true}` uses the controller's rate limiter. `pkg/controller/controller.go:252`:

```go
if options.RateLimiter == nil {
	if ptr.Deref(options.UsePriorityQueue, true) {
		options.RateLimiter = workqueue.NewTypedItemExponentialFailureRateLimiter[request](5*time.Millisecond, 1000*time.Second)
	} else {
		options.RateLimiter = workqueue.DefaultTypedControllerRateLimiter[request]()
	}
}
```

*Conclusions:* per-item exponential backoff starting at **5 ms** and doubling to a cap of **1000 s** (≈16.7 minutes). Per *item* — so one flapping object backs off without starving the queue. And `ptr.Deref(options.UsePriorityQueue, true)` confirms the default from Move 5's history: priority queue on. In the non-priority path, `DefaultTypedControllerRateLimiter` additionally composes a global token bucket of 10 QPS with burst 100 (`client-go`'s `default_rate_limiters.go`).

Stop rule satisfied: the next hop is `client-go`'s workqueue and `SharedIndexInformer`, which is a boundary you've decided to treat as understood. Note the entry point (`cache.GetInformer` → `client-go` `SharedIndexInformer`) and stop.

---

**The trace, written down:**

```
API server                                    controller-runtime v0.24.1
──────────                                    ──────────────────────────

  object                       ┌──────────────────────────────────────────┐
  changes                      │ manager.Manager                          │
     │                         │   owns ONE shared cache for the process  │
     │  LIST + WATCH           │                                          │
     └────────────────────────▶│  pkg/cache → client-go SharedIndexInformer│
        (one per Kind,         │       │                                  │
         not one per           │       │ delta → in-memory Indexer/Store   │
         controller)           │       │                                  │
                               │       ▼                                  │
                               │  pkg/internal/source/kind.go             │
                               │    Kind.Start: Cache.GetInformer(...)    │
                               │      retries every 10s until it resolves │
                               │      then AddEventHandlerWithOptions     │
                               │      then WaitForCacheSync (a hard gate) │
                               │       │                                  │
                               │       ▼                                  │
                               │  pkg/handler — EnqueueRequestForObject   │
                               │    Request{NamespacedName{Name,Namespace}}│
                               │    ◀── THE OBJECT IS DISCARDED HERE ───  │
                               │    IsInInitialList → Priority = -100     │
                               │    old.RV == new.RV → Priority = -100    │
                               │       │                                  │
                               │       ▼                                  │
                               │  priorityqueue (default since v0.21)     │
                               │    dedupes by key · per-item backoff     │
                               │    5ms → ×2 → cap 1000s                  │
                               │       │                                  │
                               │       ▼  GetWithPriority()               │
                               │  N worker goroutines                     │
                               │    N = MaxConcurrentReconciles (default 1)│
                               │    same key NEVER concurrent             │
                               │       │                                  │
                               │       ▼                                  │
                               │  YOUR Reconcile(ctx, req)                │
                               │       │                                  │
                               │       ├─ err (not Terminal) ─▶ AddWithOpts│
                               │       │                       RateLimited ┼──┐
                               │       ├─ err is TerminalError ▶ no requeue│  │
                               │       ├─ RequeueAfter > 0 ──▶ Forget then │  │
                               │       │                       AddOpts{After}┼─┤
                               │       └─ Result{} , nil ────▶ Forget      │  │
                               └──────────────────────────────────────────┘  │
                                       ▲                                     │
                                       └─────────── requeue paths ───────────┘
```

Every box in that diagram is a file you have now read, and the two requeue arrows are the two branches of a `switch` you have seen in source. That is what a finished trace looks like.

## Perspectives

**Developer / tooling view.** `go doc` plus gopls's four navigations is genuinely different in kind from Python's `help()`/`dir()`. Nothing needs to be imported, constructed, or executed, because the type system has already written down everything a runtime probe would tell you — and more, since it covers branches your probe would never reach. You are reading a static artifact, not sampling a live object. The flip side: the moment a codebase does something genuinely dynamic — a `map[string]func()` dispatch table, a `reflect`-driven decoder — the tools go quiet and you're back to grep plus reasoning. Notice where that happened in the walkthrough: `doWatch` uses `reflect.ValueOf(&hdler).Elem().Set(...)` to assign a non-generic handler into a generic variable, and that assignment is invisible to find-references. Reflection is where a Go trace gets hard, and it is rare enough to be worth flagging when you hit it.

**Operator / on-call view.** This skill is the difference between "restart the pod and hope" and a named diagnosis. Concretely, from the walkthrough above: a controller that never reconciles is most likely stuck in `WaitForCacheSync` because the informer can't LIST — which is RBAC, or a CRD that isn't applied, and `source.Kind.Start` logs a distinguishing message for each at a 10-second cadence. You now know to look for that message, and you know why the pod is healthy, ready, and doing nothing. Every tool's full source is already on disk in `$GOMODCACHE`, at the version you run, not behind a registry you have to petition.

**Design / architecture view.** Reading `client-go`'s informer → workqueue → `syncHandler` pattern *before* reading controller-runtime's abstraction over it is the "read the framework's own reference implementation" move. `kubernetes/sample-controller` exists specifically as a canonical, heavily-commented teaching example of the raw pattern, built by the Kubernetes project for exactly this purpose. Reading it makes the abstraction click, because you see what controller-runtime removed: in `sample-controller` you write the informer registration, the queue, the worker loop, and the key-splitting yourself, and each of those is ten to thirty lines you can point at.

**Organizational / review view.** The same discipline — read the test first, read the doc comment first, understand before judging — is what Google's public code-review standard asks of reviewers: understand what a change does and why before forming an opinion on how it's done. Reading history has a second organizational payoff: a commit message like PR #3107's *"There is no good reason to use this setting, either an error or `RequeueAfter` should be used instead"* is a maintainer's stated intent, and citing it in a code review ends an argument that a Stack Overflow link would only prolong.

## Real-world use cases

- **kubernetes-sigs/controller-runtime — PR #3107, "Deprecate `reconcile.Result.Requeue`"** (commit `f15ff17`, authored 2025-02-08 by Alvaro Aleman, merged 2025-02-24). <https://github.com/kubernetes-sigs/controller-runtime/pull/3107> — a one-paragraph commit message that settles a recurring design question: *"There is no good reason to use this setting, either an error or `RequeueAfter` should be used instead. Deprecate it to avoid confusion."* **What it shows:** history as documentation. The deprecation notice in the doc comment tells you *what*; the commit tells you *why*, in the maintainer's own words, and it is a stronger citation in a review than any blog post.
- **kubernetes-sigs/controller-runtime — the priority-queue arc: PR #3014 → #3111 → #3332.** <https://github.com/kubernetes-sigs/controller-runtime/pull/3014> · <https://github.com/kubernetes-sigs/controller-runtime/pull/3332> — #3014 (Dec 2024) added an opt-in priority workqueue whose purpose is to de-prioritise events from the initial listwatch and from resyncs, with a benchmark in the commit message justifying a B-tree over a slice (`AddGetDone` 5.078 ms → 1.163 ms, −77%; allocations ~3,000 → ~1,000). #3332 (Oct 2025) enabled it by default. **What it shows:** three commits explain an entire field (`Result.Priority`), an entire constant (`handler.LowPriority = -100`), and a behaviour — post-restart LIST floods queued behind real changes — that no amount of reading the current source would have explained.
- **NVIDIA/dcgm-exporter.** <https://github.com/NVIDIA/dcgm-exporter> — the exporter you operate on every GPU node, and the subject of this lesson's worked example. **What it shows:** that a trace can overturn an assumption. Grepping its Go source for `DCGM_FI_DEV_GPU_UTIL` finds no metric definition, because the metric set is CSV configuration, not code — and its `/metrics` rendering does not use `prometheus/client_golang` at all. Both facts are invisible from the outside and obvious after fifteen minutes of tracing.
- **kubernetes/sample-controller — `controller.go`.** <https://github.com/kubernetes/sample-controller/blob/master/controller.go> — the official, heavily-commented reference implementation of the informer → workqueue → `syncHandler` pattern in raw `client-go`, with no framework indirection. **What it shows:** the same machinery you traced through controller-runtime, but written out by hand, so you can see exactly which lines the framework is generating for you.
- **Google — eng-practices, "What to look for in a code review."** <https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md> — "read the whole CL to understand what it does," "prefer simple over clever." **What it shows:** the reading-before-judging discipline this lesson teaches, as an actual public engineering standard rather than a personal preference.

## Worked example

**The scenario.** A GPU node reports `DCGM_FI_DEV_GPU_UTIL{gpu="3"} 0` while `nvidia-smi` on that host shows the GPU at 97%. You need to know where that number comes from before you can say whether it's a DCGM problem, an exporter problem, or a config problem. You have fifteen minutes.

Run the same seven moves on a repo you have never opened.

**Move 0 — pin and locate.**

```
$ go mod download github.com/NVIDIA/dcgm-exporter@latest
$ go list -m -f '{{.Dir}}' github.com/NVIDIA/dcgm-exporter
/root/go/pkg/mod/github.com/!n!v!i!d!i!a/dcgm-exporter@v0.0.0-20260715173009-181290c399d4
$ D=$(go list -m -f '{{.Dir}}' github.com/NVIDIA/dcgm-exporter)
```

Note the `!n!v!i!d!i!a` escaping — uppercase letters in a module path become `!` + lowercase in the cache. In production you'd pin the tag you actually run rather than `@latest`; the pseudo-version above is a commit, and it is what every conclusion below is true of.

**Move 1 — grep for the metric name. It fails, informatively.**

```
$ grep -rn "DCGM_FI_DEV_GPU_UTIL" $D --include="*.go" | grep -v _test
internal/pkg/transformation/const.go:44:	metricGPUUtil = "DCGM_FI_DEV_GPU_UTIL"

$ grep -rn "DCGM_FI_DEV_GPU_UTIL" $D --include="*.csv"
etc/default-counters.csv:23:DCGM_FI_DEV_GPU_UTIL,      gauge, GPU utilization (in %).
etc/dcp-metrics-included.csv:23:DCGM_FI_DEV_GPU_UTIL,      gauge, GPU utilization (in %).
```

*Conclusion, and it reframes the whole investigation:* the metric is **not defined in Go**. The single Go hit is a constant used by a transformation, not a definition. The metric set is **configuration** — a three-column CSV of `DCGM FIELD, prometheus type, help message`. So the question changes from "which code emits this?" to "how does a CSV row become a series?", and the first thing to check on the broken node is which CSV the DaemonSet actually mounted.

**Move 2 — follow the CSV into Go.**

```
$ grep -rn "csv.NewReader" $D/internal/pkg/counters/*.go | grep -v _test
internal/pkg/counters/counter_config.go:127:	r := csv.NewReader(file)
internal/pkg/counters/counter_config.go:229:	r := csv.NewReader(strings.NewReader(cm.Data[appconfig.DefaultConfigMapKey]))
```

Two readers: one from a file, one from a **ConfigMap**. *Conclusion:* the field list can be supplied either as a mounted file or as a Kubernetes ConfigMap — that's a deployment-shape fact you'd otherwise learn from the Helm chart.

`ExtractCounters` (`counter_config.go:135`) is where a CSV row becomes a Go value:

```go
if _, ok := promMetricType[record[1]]; !ok {
	return nil, fmt.Errorf("unsupported Prometheus metric type %q; supported types are counter, gauge, untyped, and label", record[1])
}

fieldID, ok := dcgm.GetFieldID(record[0])
isLegacyField := dcgm.IsLegacyField(record[0])
if !ok && !isLegacyField {
	expField, err := IdentifyMetricType(record[0])
	// ... exporter-internal metrics take a different path ...
}

if !fieldIsSupported(uint(fieldID), c) {
	slog.Warn(fmt.Sprintf("Skipping line %d ('%s'): metric not enabled", i, record[0]))
	continue
}

res.DCGMCounters = append(res.DCGMCounters,
	Counter{FieldID: fieldID, FieldName: record[0], PromType: record[1], Help: record[2]})
```

*Conclusions:*

- The CSV name is resolved to a numeric **DCGM field ID** by `dcgm.GetFieldID` in `github.com/NVIDIA/go-dcgm` — a different module. So the string `DCGM_FI_DEV_GPU_UTIL` is a DCGM library identifier, not an exporter one, which is why it isn't defined in this repo.
- There are exactly four legal values in column 2: `counter`, `gauge`, `untyped`, `label`. A typo is a startup error, not a silent skip.
- **`fieldIsSupported` can silently skip a line, logging at WARN**, with the message `Skipping line N ('FIELD'): metric not enabled`. Reading `fieldIsSupported`, the gate is `c.CollectDCP` for profiling-range field IDs. *This is a live hypothesis for the incident:* a missing metric can be a disabled feature flag, and the only evidence is one WARN line at startup. First thing to check in the pod logs.
- The resulting `Counter{FieldID, FieldName, PromType, Help}` is the unit that flows through the rest of the system.

**Move 3 — find the collection interface.**

```
$ grep -n "type Collector interface" -A 4 $D/internal/pkg/collector/types.go
34:type Collector interface {
35-	GetMetrics() (MetricsByCounter, error)
36-	Cleanup()
37-}
```

*Conclusion:* two methods, and — critically — **this is not `prometheus.Collector`.** It has no `Describe`, no `Collect(chan<- prometheus.Metric)`. dcgm-exporter defines its own collector abstraction. Note that discrepancy; it will pay off in two moves.

`collector_factory.go:180` shows `NewDCGMCollector(cf.counterSet.DCGMCounters, cf.hostname, cf.config, entityWatchList)` — the counters parsed from CSV are handed to the collector at construction. The chain holds.

**Move 4 — the aggregation layer.**

```
$ grep -n "func (r \*Registry) Gather" $D/internal/pkg/registry/registry.go
71:func (r *Registry) Gather() (MetricsByCounterGroup, error) {
```

Reading it: a `sync.RWMutex` (RLock for concurrent gathers), an `atomic.Bool` shutdown flag, an `atomic.Int32` in-flight counter, and then:

```go
g := new(errgroup.Group)
for group, collectors := range r.collectorGroups {
	for _, c := range collectors {
		c := c
		group := group
		g.Go(func() error {
			metrics, err := c.GetMetrics()
			if err != nil { return err }
			metricsByCounterGroupMtx.Lock()
			defer metricsByCounterGroupMtx.Unlock()
			// ... merge into metricsByCounterGroup ...
			return nil
		})
	}
}
if err := g.Wait(); err != nil { return nil, err }
```

*Conclusion:* this is **Lesson 7's concurrent-collection pattern, in the wild** — `errgroup` fan-out over sub-collectors inside one gather, exactly like node_exporter's `sync.WaitGroup` version. One difference worth noting: unlike node_exporter's `execute`, this one propagates the first error out of `g.Wait()` and returns `nil, err`, so **a single failing collector fails the whole gather**. That's a second live hypothesis for a missing metric, and it's the design tradeoff Lesson 7 flagged when it told you to `return nil` from each `g.Go`.

**Move 5 — the exposition path, and the surprise.**

```
$ grep -rn "prometheus/client_golang" $D --include="*.go" | grep -v _test | grep -v mocks
(no output)

$ grep -n "client_golang\|prometheus" $D/go.mod
20:	github.com/prometheus/client_model v0.6.2
21:	github.com/prometheus/common v0.69.0
22:	github.com/prometheus/exporter-toolkit v0.17.1
138:	github.com/prometheus/client_golang v1.23.2 // indirect
```

*Conclusion, and it is the payoff of the whole exercise:* **dcgm-exporter does not use `prometheus/client_golang` to expose metrics.** It appears in `go.mod` only as an `// indirect` dependency — pulled in by something else. There is no `prometheus.Desc`, no `MustNewConstMetric`, no `promhttp.Handler`. Everything you learned in Lesson 7 about the `Collector`/`Desc` API is *not* what this binary does.

What it does instead is in `internal/pkg/rendermetrics/render_metrics.go`:

```go
import (
	dto "github.com/prometheus/client_model/go"
	"github.com/prometheus/common/expfmt"
	"github.com/prometheus/common/model"
)

// Render writes all metric groups as Prometheus text exposition.
func Render(w io.Writer, metricGroups map[dcgm.Field_Entity_Group]collector.MetricsByCounter) error {
	// ... build one *dto.MetricFamily per metric name, sorted deterministically ...
	for _, name := range names {
		builder := builders[name]
		sort.SliceStable(builder.family.Metric, func(i, j int) bool { /* by label signature */ })
		if _, err := expfmt.MetricFamilyToText(w, builder.family); err != nil {
			return fmt.Errorf("render metric family %q: %w", name, err)
		}
	}
	return nil
}
```

It constructs `dto.MetricFamily` protobuf values by hand and serialises them with `expfmt.MetricFamilyToText` from `prometheus/common` — the layer *underneath* `client_golang`. The file even documents its target format in a comment:

```
* # HELP FIELD_ID HELP_MSG
* # TYPE FIELD_ID PROM_TYPE
* FIELD_ID{gpu="GPU_INDEX_0",uuid="GPU_UUID", attr...} VALUE
```

*Why this matters for the incident:* it tells you that a bug in the *shape* of the output (a mangled label, a duplicate series, a missing `# TYPE`) is dcgm-exporter's own code, not a well-tested library's. And it tells you which repo to file against.

**Move 6 — the HTTP entry point.**

```
$ grep -n "HandleFunc\|http.Server{" $D/internal/pkg/server/server.go | grep -v _test
65:		server: &http.Server{
113:	router.HandleFunc("/health", serverv1.Health)
114:	router.HandleFunc("/metrics", serverv1.Metrics)
117:		router.HandleFunc("/debug/pprof/", pprof.Index)
```

*Conclusion:* a plain `http.ServeMux`, `/metrics` bound to a handler that calls `Registry.Gather()` and then `rendermetrics.Render`. Also `/debug/pprof/*` behind a flag — which is an operational gift: you can profile a misbehaving dcgm-exporter in place without rebuilding it.

**The finished trace, and the answer to the question.**

```
etc/default-counters.csv  ── "DCGM_FI_DEV_GPU_UTIL, gauge, GPU utilization (in %)."
        │                     (or a ConfigMap — counter_config.go:229)
        │  counters.ExtractCounters            counter_config.go:135
        │    · validates col 2 ∈ {counter,gauge,untyped,label}
        │    · dcgm.GetFieldID(name)  → numeric DCGM field ID  [go-dcgm module]
        │    · fieldIsSupported()     → may SKIP with a WARN log   ◀── suspect #1
        ▼
counters.Counter{FieldID, FieldName, PromType, Help}
        │  collector_factory.go:180 → NewDCGMCollector(DCGMCounters, ...)
        ▼
collector.Collector (types.go:34) — GetMetrics() (MetricsByCounter, error)
        │    · reads the DCGM field group for the watched entities
        ▼
registry.Registry.Gather()  registry.go:71
        │    · errgroup fan-out across collectors  ── first error fails ALL  ◀── suspect #2
        ▼
rendermetrics.Render()  render_metrics.go:56
        │    · builds dto.MetricFamily by hand (NOT client_golang)
        │    · expfmt.MetricFamilyToText(w, family)
        ▼
server.go:114  GET /metrics  →  HTTP body
```

**What you'd do next, on the node.** The trace produced two testable hypotheses and one structural fact, in that order of cheapness:

1. Check the pod's startup logs for `Skipping line N ('DCGM_FI_DEV_GPU_UTIL'): metric not enabled` — a config/feature-flag problem, fixed by the CSV or `CollectDCP`, no code involved.
2. Check whether the whole `/metrics` response is missing other GPU metrics too. If it is, one collector is erroring and `errgroup` is failing the entire gather; if `DCGM_FI_DEV_GPU_UTIL` alone reads 0 while its siblings are fine, the field genuinely came back as 0 from DCGM and the bug is below the exporter — `dcgmi dmon -e 203` on the host settles it.
3. Either way, you now know that a *formatting* bug would be dcgm-exporter's own `rendermetrics` code, and that `/debug/pprof` is available if the exporter is slow rather than wrong.

Elapsed: seven commands and four files. That is the whole point of the method — you did not read dcgm-exporter, you *interrogated* it.

## Practice

**Task.** Write a one-page `trace.md` of two real flows, citing concrete files, types, and line numbers.

1. **Trace `controller-runtime`'s watch→reconcile path** at the version you have pinned. Start from `Builder.For`/`Complete` and follow it through `source.Kind`, `handler.EventHandler`, the workqueue, and the worker loop to `reconcile.Reconciler.Reconcile`. Name `manager.Manager`, `client.Client`, `source.Source`, `handler.EventHandler`, the queue type, and `reconcile.Request`/`Result`. **Do not stop at `Complete()` — follow it in.** Mark every hop that lives under an `internal/` path as "not stable API," with the version it's true of.
2. **Trace one metric in one exporter you operate** — `node-exporter` or `dcgm-exporter` — from its source of truth to `/metrics`. Name the real files and types. If, like dcgm-exporter, the metric turns out not to be defined in Go at all, that finding *is* the answer — write down what it is defined by instead, and what that means operationally.
3. **Use every move at least once, and say which move produced which fact.** In particular: cite at least one `_test.go` or `example_test.go` that clarified intent where the docs didn't, and at least one piece of **history** — a commit message, a PR, or a changelog entry — that explained a "why" the current source cannot.

**Acceptance.** A committed `trace.md` under `../practice/` (e.g. `../practice/gpu-cost-exporter/trace.md`) that:

- Names **real** files, types, and packages — no hand-waving. Every claim should be checkable by someone with the same version pinned.
- Is **correct**: the reconcile path goes shared cache/informer → `handler.EventHandler` → workqueue → worker goroutine → `Reconcile`, and the reconciler receives a `reconcile.Request` (a key), not an event payload.
- States the **version** every claim is true of, at the top, obtained from `go list -m`.
- Lists, per hop, the **command you used to find it** — so the trace is reproducible, not just assertable.
- Is one page and skimmable. If it's three pages you're transcribing, not tracing.

## Common pitfalls

1. **Reading the newest blog explanation of a library instead of the library's own tests.** *Symptom:* your mental model is confidently wrong about a specific behaviour, and you can't work out why the code doesn't do what you "know" it does. *Mechanism:* a blog post was true at the version it was written against and has no mechanism to notice it has gone stale. A co-located `_test.go` is compiled and executed in CI against exactly the code you have, so it cannot drift. `Example*` functions are even stronger: their output is asserted.

2. **Trusting main-branch GitHub browsing over your pinned version.** *Symptom:* "but the docs say X" — a field, method, or default that doesn't exist in your binary. *Mechanism:* GitHub's default view is `main`, which can be many releases ahead. `Result.Priority` did not exist before v0.21; `Requeue` was not deprecated before v0.20. Confirm with `go list -m` and read with `go doc -src`, which is module-cache-backed and therefore version-correct by construction.

3. **Grepping for a method name without accounting for embedding.** *Symptom:* a type you're certain implements an interface produces zero grep hits, and you conclude the implementation is elsewhere or doesn't exist. *Mechanism:* method promotion from an embedded type is a property of the method set, computed by the compiler, with no textual representation at the outer type. Check the struct's fields for an embedded type, or use gopls find-implementations, which works from the type system.

4. **Assuming "exported" means "meant for you to depend on" under an `internal/` path.** *Symptom:* a patch-release upgrade breaks a mental model or, worse, a vendored fork. *Mechanism:* capitalisation inside an `internal/` tree exists so sibling packages can see the symbol; the compiler already prevents external import, so the maintainers owe you no stability and no deprecation notice. Read it, cite it with a version, don't build on it.

5. **Stopping at the first layer of indirection — "the trace that stops too early."** *Symptom:* a trace that sounds right, uses the correct nouns, and cannot answer any specific question during an incident. *Mechanism:* builder and facade methods are named for their *effect*, not their *implementation*; `Complete` is three lines. Apply the stop rule: you are done only at a syscall, a boundary you've explicitly declared opaque, or code you already know — never at "I found a function with a matching name."

6. **Reading every implementation of an interface instead of the one in play.** *Symptom:* an afternoon lost in `client.Client`'s implementations — typed, unstructured, metadata, dry-run, namespaced, field-owner-wrapped. *Mechanism:* find-implementations answers "what could this be?", which is a much bigger set than "what is this?". The constructor that produced the value answers the second question. Follow values, not types.

## Self-check

**(a) Given an interface, how do you find every implementation — and what does text search miss?**
**Answer:** Use gopls **find-implementations** on the interface type. Go satisfaction is implicit (there is no `implements` keyword to grep for), so gopls computes method sets from the type system and enumerates every concrete type in scope that satisfies the interface. Without an editor, grep the interface's distinctive method signature across `*.go` — Go signatures are specific enough that this usually works (`grep -rn "Reconcile(ctx context.Context, req reconcile.Request)"`). What text search **misses is embedding**: a type that embeds another gains the embedded type's methods in its method set, with nothing textual at the outer type to match. `type GPUCostReconciler struct{ baseReconciler }` satisfies `reconcile.Reconciler` and matches no grep for `func (r *GPUCostReconciler) Reconcile`. And the practical refinement: when tracing a specific call path you usually don't want the full implementation list at all — you want the *one* concrete type at this call site, which you get by following the constructor that produced the value.

**(b) How do you tell what a package is for in five minutes?**
**Answer:** `go doc <pkg>` — read the package doc comment (authors put the purpose in the top paragraph) and scan the one-line listing of exported types and functions to see the shape of the API. Read the type aliases specially: `type Reconciler = TypedReconciler[Request]` tells you the generic form is the real definition. Then open `example_test.go` if there is one, or the most central `_test.go`, and read only the test *names* — they are the contract's edge-case list. Five minutes of `go doc` plus one test beats reading the source, needs no running code, and — unlike a blog post — is guaranteed to match the version you have pinned. Escalate to `go doc -all` for full comments, `go doc -src` for the implementation, and `go doc -u` when you need the unexported fields.

**(c) Trace: what wakes a reconciler when a watched object changes?**
**Answer:** The manager owns one **shared cache** per process. `Builder.For(...)` (via `Complete` → `Build` → `doWatch`) constructs a `source.Kind(mgr.GetCache(), obj, &handler.EnqueueRequestForObject{}, predicates...)` and hands it to `Controller.Watch`. At `Start`, the source asks the cache for an **informer** for that Kind — retrying every 10 s, and logging distinguishable errors for "CRD not installed" versus "kind not registered to the Scheme" — registers an event handler on it, and blocks on `WaitForCacheSync`. The informer does one LIST then a long-lived WATCH against the API server and streams deltas into an in-memory store. On a change, the registered **`handler.EventHandler`** builds a `reconcile.Request{NamespacedName{Name, Namespace}}` and enqueues it — **discarding the object**, because `Request` has nowhere to put it. Events from the initial LIST, and Updates whose old and new `resourceVersion` match (i.e. resyncs), are enqueued at `LowPriority = -100`. The **priority workqueue** dedupes by key and rate-limits retries with per-item exponential backoff (5 ms doubling to a 1000 s cap). One of `MaxConcurrentReconciles` **worker goroutines** pops the key and calls your `Reconcile(ctx, req)`; the workqueue guarantees the same key is never reconciled concurrently. It is level-triggered: you get a key, not a diff, so you re-read current state through the cache-backed `client.Client` and converge.

**(d) You're debugging behaviour that doesn't match a README you just read. What's the first thing to check?**
**Answer:** Whether the README reflects the version you have pinned, not `main`. Run `go list -m <module>` for the resolved version — note it resolves through the whole module graph including `replace` directives and MVS upgrades, so it can differ from your `require` line. Then read *that* version's source with `go doc -src`, which reads your module cache rather than the network, or check the `CHANGELOG.md`/release notes for that specific tag. GitHub's default-branch view is a repeat source of "but the docs say X" confusion precisely because it silently shows you code ahead of what you're running. If the difference is real and you want to know *why* it changed, `git log -S "<symbol>"` in a clone of the dependency finds the commit that introduced or removed it.

**(e) Why is history a reading tool, and what does it tell you that the current source cannot?**
**Answer:** The current snapshot tells you *what* the code does; only history tells you *why it is that way*, which is what you need to predict whether a behaviour is load-bearing or incidental. Two concrete examples from this lesson. `reconcile.Result.Requeue` is marked deprecated in the doc comment, but the commit (`f15ff17`, PR #3107) states the reasoning — *"There is no good reason to use this setting, either an error or `RequeueAfter` should be used instead"* — which is a citable maintainer position rather than an inference. And `Result.Priority`, `handler.LowPriority = -100`, and the post-restart behaviour where the initial LIST flood is deliberately queued behind real changes are all explained by three commits (PRs #3014, #3111, #3332) and by nothing in the current source. The tools: `git log -p --follow <file>`, `git blame -L m,n <file>`, and `git log -S "<string>"` (the pickaxe), which finds the commit where a string's occurrence count changed even if it has since moved files.

**(f) When is a trace finished?**
**Answer:** When the next hop is one of exactly three things: a **syscall or the runtime** (`net.(*netFD).Read`, `runtime.selectgo`) — there is no more Go below; a **boundary you have explicitly declared opaque** (the API server, the DCGM C library, etcd) — in which case write down which boundary and what you're assuming about it; or **code you already understand**, in which case note the entry point and stop. "I found a function whose name matches what I'm looking for" is not on the list, and neither is "this feels like enough." The written artifact of the stop rule is a trace where each line is `file:line — Type.Method — effect`, every `internal/` hop is annotated with the version it's true of, and the last line names the boundary rather than trailing off.

## Connections & what's next

This lesson turns the interfaces-and-composition instincts from Lessons 2–3 into a reading skill: find-implementations is only useful because you understand implicit satisfaction, and the embedding blind spot is only visible because you understand method promotion. It closes the loop with Lesson 6's `internal/` boundary — now you know why "exported but internal" isn't a contract — and with Lesson 7's exporter idiom, where the dcgm-exporter walkthrough both confirmed the concurrent-collection pattern and overturned the assumption that every Go exporter uses `client_golang`. It also feeds Lesson 5's testing instincts back the other way: "read the test first" only works if you can read a table-driven or Ginkgo test fluently.

It is the direct prerequisite for Lesson 9: you're about to build a controller on `controller-runtime`, and the trace you just rehearsed — cache → informer → handler → priority queue → worker → `Reconcile`, with the two requeue paths drawn — *is* the mental model that lesson assumes on day one. Checkpoint item 5 asks for that trace in writing, citing specific files and types; the Practice task above is exactly that artifact.

**Next: Lesson 9, [Controller primer (CRD · reconcile · envtest)](09-controller-primer.md).** You now know how to read the framework; next you write against it — a CRD plus a reconcile loop, tested with envtest, that becomes the seed of your capstone GPU cost/efficiency operator.

## References & further reading

**Primary sources**

- **controller-runtime** — [repo](https://github.com/kubernetes-sigs/controller-runtime) · [pkg docs](https://pkg.go.dev/sigs.k8s.io/controller-runtime) — the source traced throughout this lesson, at v0.24.1. Read `pkg/reconcile` (the interface and `Result` contract), `pkg/builder/controller.go` (`Build`/`doWatch`), `pkg/internal/source/kind.go` (`Kind.Start`), `pkg/handler` (`EnqueueRequestForObject`, `LowPriority`), and `pkg/internal/controller/controller.go` (the worker loop and `reconcileHandler`). The root package's own doc comment is a guided tour of the layout and is worth reading in full before anything else.
- **`cmd/go` documentation** — [go doc](https://pkg.go.dev/cmd/go#hdr-Show_documentation_for_package_or_symbol) · [go list](https://pkg.go.dev/cmd/go#hdr-List_packages_or_modules) — the exact flag semantics for the orientation workflow: `-src`, `-all`, `-u`, symbol/method addressing, and `go list -m -f` for building module-cache paths (including the `!x` uppercase escaping).
- **gopls documentation** — <https://github.com/golang/tools/blob/master/gopls/doc/features.md> — the authoritative list of navigations, including implementations, references, and call hierarchy, plus which editor commands map to them.
- **NVIDIA/dcgm-exporter** — <https://github.com/NVIDIA/dcgm-exporter> — the worked example's subject. The files that matter: `etc/default-counters.csv`, `internal/pkg/counters/counter_config.go`, `internal/pkg/collector/types.go`, `internal/pkg/registry/registry.go`, `internal/pkg/rendermetrics/render_metrics.go`, `internal/pkg/server/server.go`.

**Real-world engineering blogs and history**

- **controller-runtime PR #3107 — Deprecate `reconcile.Result.Requeue`** — <https://github.com/kubernetes-sigs/controller-runtime/pull/3107> — what it shows: a commit message as the citable primary source for a design decision. Commit `f15ff17`, authored 2025-02-08, merged 2025-02-24.
- **controller-runtime PR #3014 and #3332 — the priority queue** — <https://github.com/kubernetes-sigs/controller-runtime/pull/3014> · <https://github.com/kubernetes-sigs/controller-runtime/pull/3332> — what they show: how three commits explain a field, a constant, and a behaviour that the current source alone cannot. #3014 also carries the B-tree-vs-slice benchmark in its commit message.
- **kubernetes/sample-controller — `controller.go`** — <https://github.com/kubernetes/sample-controller/blob/master/controller.go> — what it shows: the informer → workqueue → `syncHandler` pattern in raw `client-go`, no framework indirection. Read it once end to end after the controller-runtime trace; the abstraction clicks when you see what it removed.
- **Google — eng-practices, "What to look for in a code review"** — <https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md> — what it shows: the reading-before-judging discipline this lesson teaches, as a public engineering standard.

**Deeper dives**

- **Kubernetes blog — "How the controller-runtime Cache Actually Works, and Why Your Controller Does Not Crash the API Server"** (29 July 2026, Andrei Kvapil and Timofei Larkin) — <https://kubernetes.io/blog/2026/07/29/controller-runtime-cache-explained/> — walks the Reflector → delta queue → Indexer/Store → `SharedIndexInformer` chain one layer below where this lesson's trace stops, and is blunt about the cost: a controller that "just reads" can consume gigabytes of memory, do hidden O(n) scans, and trip over stale reads. The same trace-a-call-path exercise applied to the cache your Lesson 9 controller depends on. (Note: this post has been revised since publication to correct technical inaccuracies — read the current version.)
- **prometheus/node_exporter — `collector/collector.go`** — <https://github.com/prometheus/node_exporter/blob/master/collector/collector.go> — what it shows: a second, independent implementation of the concurrent-collection pattern you found in dcgm-exporter's `Registry.Gather`, using `sync.WaitGroup` and a per-collector `execute` helper that records duration and success. Comparing the two is the cheapest way to see which parts of the pattern are essential and which are taste.
