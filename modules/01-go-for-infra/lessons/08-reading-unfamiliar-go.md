---
lesson: "01.8"
title: "Reading unfamiliar Go: orient in a large repo and trace a call path"
module: "01"
concept: "Reading unfamiliar Go: orient in a large repo and trace a call path"
status: not-started
est_time: "10h"
artifacts: []
---

# 01.8 · Reading unfamiliar Go: orient in a large repo and trace a call path

> **Concept.** Land in a large unknown Go repo and orient fast — `go doc`, jump-to-def, follow an interface to its implementations, read tests for intent, and trace a call path — so you can read the source of any tool you operate.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Why this matters

This is the differentiator that separates a Senior Platform Engineer from an operator. Anyone can `helm install dcgm-exporter`; the senior engineer reads its source when the metrics look wrong, traces exactly which NVML call feeds `DCGM_FI_DEV_GPU_UTIL`, and files a precise bug or patches it. On a GPU-heavy platform you operate node-exporter, dcgm-exporter, kube-state-metrics, and eventually your own controllers — all Go, all open source, all readable. When an on-call page says "the operator isn't reconciling," the ability to open `controller-runtime` and *know within minutes* where the watch→reconcile path is wired is the skill that resolves the incident. Your capstone is a controller; you cannot write one competently without being able to read the framework it's built on. Coming from Python, the good news is Go repos are *unusually* readable: static types, no metaclass magic, one obvious way to structure a package, and tests that read like specs.

## From Python to Go

| Reading Python | Reading Go | Why Go is easier to trace |
|---|---|---|
| `help(obj)`, `dir()` at runtime | `go doc pkg`, `go doc pkg.Symbol` — static, no import side effects | No need to instantiate anything; docs come from source. |
| "Who calls this?" via grep + hope | jump-to-def + **find-implementations** (gopls) | Types are explicit; the tool finds every concrete type satisfying an interface. |
| Duck typing — implementers are invisible | Interfaces are satisfied **implicitly but statically** | A type implements an interface if it has the methods; gopls enumerates them exactly. |
| `__init__.py` re-exports obscure origin | One package = one directory; symbol is defined where it's defined | `go doc` and jump-to-def land on the real definition. |
| Read the class to learn intent | **Read the `_test.go` first** | Tests are colocated (`foo_test.go` next to `foo.go`) and show the API's intended use. |
| Dynamic dispatch hides the call | Interface method → gopls "implementations" → the concrete method | The indirection is real, but the tooling resolves it deterministically. |

The core delta: in Python you often *run* code to understand it; in Go you *read* it, because the type system has already written down everything the runtime would tell you. Implicit interface satisfaction is the one thing that feels alien — nothing says `implements` — so "find implementations" becomes your most-used command.

## Core notes

**`go doc` — the fastest orientation tool.** No browser, works offline, works on any package in your module graph.

```
go doc net/http                 # package synopsis + exported symbols
go doc net/http.Client          # one type, its fields and methods
go doc net/http.Client.Do       # one method's signature + doc comment
go doc -src net/http.Client.Do  # the actual source of that method
go doc sigs.k8s.io/controller-runtime/pkg/reconcile   # a dependency's package
go doc sigs.k8s.io/controller-runtime/pkg/reconcile.Reconciler
```

`go doc pkg` in 5 minutes tells you what a package is *for*: read the package doc comment (the paragraph at the top), then scan the exported type and function names. `-src` drops you into the implementation without opening an editor.

**Editor / gopls moves.** `gopls` is the Go language server; these are the four you'll use constantly:
- **Jump-to-definition** (`gd`) — land on where a symbol is defined, across package and module boundaries (including into `controller-runtime`'s source in your module cache).
- **Find-implementations** — given an *interface*, list every concrete type that satisfies it; given a *concrete method*, list the interfaces it satisfies. This is how you cross the interface indirection.
- **Find-references** — "who calls this?"
- **Hover** — signature + doc without navigating away.

**Grep for interface method names.** When gopls isn't handy, an interface is just a set of method names. Grep the method name across the repo to find implementers:

```
grep -rn "func.*Reconcile(ctx context.Context" --include=*.go
grep -rn ") Start(ctx context.Context) error" --include=*.go   # find Runnable impls
```

Signatures are distinctive enough that method-name grep finds implementations fast, even without tooling.

**Read `_test.go` files first.** Tests encode intent. `reconcile/reconcile_test.go` shows exactly how a `Reconciler` is meant to be called and what `Result{Requeue: true}` means. Example-style tests (`func Example...`) double as runnable docs. When you don't understand a type, find its `*_test.go` and read the first table-driven test — it's the authoritative usage spec. Go's table-test idiom is instantly recognizable and worth reading fluently:

```go
func TestReconcile(t *testing.T) {
    cases := []struct {
        name string
        req  reconcile.Request
        want reconcile.Result
    }{
        {"missing object requeues nothing", reconcile.Request{/*...*/}, reconcile.Result{}},
    }
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) { /* arrange fake client, call Reconcile, assert */ })
    }
}
```

The `name` strings *are* documentation — they tell you the contract's edge cases (what happens on a missing object, on a conflict, on a requeue) faster than the prose docs do.

**Trace a call path conceptually.** You rarely single-step Go; you follow types. Start at the entry point (a `main`, a `Builder.Complete`), follow each call into the type it returns, and stop at interface boundaries to run "find-implementations." Keep a scratch list of `file:type.method` as you go — that list *is* the trace.

**The controller-runtime types you must name.** These are the vocabulary of every Kubernetes controller:

- `manager.Manager` — owns the shared cache, clients, and the run loop; you register controllers on it and call `mgr.Start(ctx)`.
- `client.Client` — the read/write Kubernetes API client, backed by the manager's cache for reads.
- `reconcile.Reconciler` — the interface *you* implement: `Reconcile(ctx, reconcile.Request) (reconcile.Result, error)`. `Request` carries just a `NamespacedName`; `Result` carries requeue instructions.
- `source.Source` — an event stream (typically a `Kind` source wrapping an informer for one object type).
- `handler.EventHandler` — translates a raw watch event (add/update/delete of an object) into zero or more `reconcile.Request`s enqueued onto the workqueue.
- `workqueue` (`client-go/util/workqueue`) — the rate-limited, deduplicating queue between the informer and your `Reconcile`. This is the load-bearing decoupler.

## Worked example

**Trace: how does `Reconcile` get invoked when a watched object changes?** Here's the path you'd reconstruct by reading `controller-runtime` with `go doc` + jump-to-def, naming real packages/types.

1. **Wiring** (`pkg/builder/controller.go`). `ctrl.NewControllerManagedBy(mgr).For(&v1.Pod{}).Complete(r)` builds a `controller.Controller` and, for each `For`/`Owns`/`Watches`, registers a **`source.Source`** paired with a **`handler.EventHandler`**. `For` uses a `source.Kind` (backed by an informer from the manager's shared cache) and the `handler.EnqueueRequestForObject` handler.

2. **The informer watches the API** (`pkg/cache` → `client-go` informers). The shared cache runs an informer per watched `Kind`. The informer LIST/WATCHes the API server; when a Pod is added/updated/deleted, the informer fires its event handlers. This is `client-go`'s `SharedIndexInformer` under the hood — controller-runtime wraps it as a `source.Kind`.

3. **Event → Request** (`pkg/handler`). The registered `handler.EventHandler` (e.g. `EnqueueRequestForObject`) receives the object event and calls `queue.Add(reconcile.Request{NamespacedName: {Namespace, Name}})`. `EnqueueRequestForOwner` (used by `Owns`) instead maps a child's event to its owner's Request. The handler is the layer that decides *which* key to enqueue.

4. **The workqueue** (`client-go/util/workqueue`). The `Request` lands on the controller's rate-limited workqueue. The queue **deduplicates** (many events for the same Pod collapse to one item) and **rate-limits** (backoff on repeated failures). This decoupling is why a controller stays stable under an event storm.

5. **The worker loop calls Reconcile** (`pkg/internal/controller/controller.go`). The controller runs N worker goroutines, each looping: `queue.Get()` → build `ctx` → call `reconciler.Reconcile(ctx, req)` → inspect the returned `reconcile.Result` and `error`. On `err != nil` or `Result.Requeue`/`RequeueAfter`, the item is re-added with backoff via `queue.AddRateLimited`; on success, `queue.Forget` + `queue.Done`. Your `Reconcile` then uses `client.Client` to GET the object by `req.NamespacedName` and drive actual state toward desired state.

So the full chain is: **API change → informer (shared cache) → `handler.EventHandler` → `reconcile.Request` on the workqueue → worker goroutine → your `Reconcile`**. The reconciler is *level-triggered*: it gets a key, not a diff — it re-reads current state every time, which is why controllers are idempotent and resilient to missed events. That single insight ("you get a key, not an event payload") is the thing most people miss, and reading the `handler` → `workqueue` boundary in the source is how you internalize it.

And here is the `Reconciler` end of that chain — the ~15 lines you'd implement, so you can see what the framework hands you (`req`, a key) and what you do with it (`Get`, then act):

```go
type GPUCostReconciler struct {
    client.Client // embedded: gives r.Get, r.Update, r.Status().Update, ...
}

func (r *GPUCostReconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
    var pod corev1.Pod
    if err := r.Get(ctx, req.NamespacedName, &pod); err != nil {
        // NotFound = object was deleted after the event was enqueued; nothing to do.
        return reconcile.Result{}, client.IgnoreNotFound(err)
    }
    // ... compute GPU cost, update a status or a metric ...
    return reconcile.Result{RequeueAfter: 5 * time.Minute}, nil // re-check periodically
}
```

Note `r.Get(ctx, req.NamespacedName, &pod)` — the handler enqueued only the *key*, and the reconciler re-reads live state through the cache-backed `client.Client`. That's the level-triggered contract made concrete.

The commands that build this trace:
```
go doc sigs.k8s.io/controller-runtime/pkg/reconcile.Reconciler
go doc sigs.k8s.io/controller-runtime/pkg/handler
go doc sigs.k8s.io/controller-runtime/pkg/source.Kind
# then jump-to-def from Complete() into pkg/internal/controller, and
grep -rn "reconcile.Reconciler" $(go env GOMODCACHE)/sigs.k8s.io/controller-runtime*/pkg
```

## Practice

**Task.** Write a one-page `trace.md` of a single real flow, citing concrete files and types.

1. Pick **one exporter you operate** — `node-exporter` or `dcgm-exporter` — and trace how one metric gets from source to `/metrics` (e.g. for dcgm-exporter: NVML/DCGM field → collector → `prometheus.Desc` → registry → HTTP handler). Name the real files and types.
2. **And** trace `controller-runtime`'s watch→reconcile path: how `Reconcile` gets called when a watched object changes, from `Builder.For`/`source.Kind` through `handler.EventHandler` and the `workqueue` to the worker calling `reconcile.Reconciler.Reconcile`. Name `manager.Manager`, `client.Client`, `source.Source`, `handler.EventHandler`, `workqueue`, `reconcile.Request`/`Result`.
3. Use `go doc`, jump-to-def, find-implementations, and reading `_test.go` files as your method — cite at least one test that clarified intent.

**Acceptance.** A committed `trace.md` under `../practice/` (e.g. `../practice/gpu-cost-exporter/trace.md`) that:
- Names **real** files/types/packages (not hand-waved), and
- Is **correct** — the reconcile path goes informer → handler → workqueue → worker → `Reconcile`, and the reconciler receives a `reconcile.Request` (a key), not an event object.
- Is one page, skimmable, with the command(s) you used to find each hop.

## Self-check

**(a) Given an interface, how do you find every implementation?**
**Answer:** Use gopls **find-implementations** on the interface type — it statically enumerates every concrete type in scope whose method set satisfies the interface (Go satisfaction is implicit, so there's no `implements` keyword to grep). Without an editor, grep for the interface's distinctive method signature across `*.go` (e.g. `grep -rn "Reconcile(ctx context.Context, req reconcile.Request)"`), since the method set is what defines an implementer.

**(b) How do you tell what a package is for in 5 minutes?**
**Answer:** Run `go doc <pkg>`: read the package doc comment (the top paragraph — authors put the purpose there), then scan the list of exported types and top-level functions to see the shape of its API. Open the package's most-central `_test.go` or any `Example` function to see intended usage. Five minutes of `go doc` + one test beats reading the whole source, and needs no running code.

**(c) Trace: what wakes a reconciler when a watched object changes?**
**Answer:** The manager's shared **cache runs an informer** that WATCHes the API server for that `Kind`. On a change, the informer fires the controller's registered **`handler.EventHandler`**, which enqueues a **`reconcile.Request`** (just the object's `NamespacedName`) onto the rate-limited, deduplicating **`workqueue`**. A **worker goroutine** pops the key and calls your **`reconcile.Reconciler.Reconcile(ctx, req)`**, which re-reads current state via `client.Client`. It's level-triggered: the reconciler gets a *key*, not the event payload, so it re-reads and reconciles idempotently.

## Resources

1. **controller-runtime** — [repo](https://github.com/kubernetes-sigs/controller-runtime) · [pkg docs](https://pkg.go.dev/sigs.k8s.io/controller-runtime) — the source you're tracing; read `pkg/reconcile`, `pkg/handler`, `pkg/source`, `pkg/builder`, and `pkg/internal/controller`. Start from the pkg docs' subpackage list, then jump into source. **Deep** — this is the framework your capstone is built on.
2. **kubernetes/sample-controller** — [repo](https://github.com/kubernetes/sample-controller) — the canonical, heavily commented informer→workqueue→syncHandler pattern in *raw* `client-go` (no framework), so you see the machinery controller-runtime hides. `controller.go` is the whole lesson. **Deep** — read it once end to end; it makes the abstraction click.
3. **`go doc`** — [command docs](https://pkg.go.dev/cmd/go#hdr-Show_documentation_for_package_or_symbol) — the exact flags (`-src`, `-all`, symbol/method addressing) for the orientation workflow above. **Skim** — reference it until the invocations are muscle memory.
