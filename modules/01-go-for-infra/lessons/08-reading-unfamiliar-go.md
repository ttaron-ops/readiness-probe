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
sources: 5
---

# 01.8 · Reading unfamiliar Go: orient in a large repo and trace a call path

> **Concept.** Land in a large unknown Go repo and orient fast — `go doc`, jump-to-def, follow an interface to its implementations, read tests for intent, read history for intent — so you can read the source of any tool you operate.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 7 handed you the exporter idiom — `promauto`, `promhttp.Handler()`, a `GaugeVec` — and pointed you at the *real* dcgm-exporter and node_exporter source it's modeled on. This lesson is the skill that makes going and actually reading that source productive rather than intimidating: orienting in an unfamiliar repo in minutes, not hours, and following a call path across package and module boundaries with confidence. It closes the loop on everything Module 01 has taught about types and interfaces (Lessons 2–3) by turning them into a *reading* superpower, not just a writing one. It unlocks Lesson 9 directly: you cannot write a controller competently against `controller-runtime` without being able to read the framework it's built on, and this lesson is where you build and rehearse that trace — the manager → informer → workqueue → `Reconcile` path — before you're relying on it under time pressure.

## Why this matters

This is the differentiator that separates a Senior Platform Engineer from an operator. Anyone can `helm install dcgm-exporter`; the senior engineer reads its source when the metrics look wrong, traces exactly which NVML call feeds `DCGM_FI_DEV_GPU_UTIL`, and files a precise bug or patches it. On a GPU-heavy platform you operate node-exporter, dcgm-exporter, kube-state-metrics, and eventually your own controllers — all Go, all open source, all readable. When an on-call page says "the operator isn't reconciling," the ability to open `controller-runtime` and *know within minutes* where the watch→reconcile path is wired is the skill that resolves the incident instead of prolonging it with a guess-and-restart cycle. Your capstone is a controller; you cannot write one competently without being able to read the framework it's built on. Coming from Python, the good news is Go repos are *unusually* readable: static types, no metaclass magic, one obvious way to structure a package, and tests that read like specs.

## What's new here (calibration)

Per the module README's skip-list, we skip cover-to-cover reading (you are never asked to read a dependency front to back) and any retread of OOP-in-Go or reflection/runtime internals — this lesson is about navigation and comprehension speed, not language mechanics you already have from Lessons 1–3.

What's genuinely new here, calibrated to staff-level depth:
- A **repeatable orientation workflow** (`go doc` → gopls → tests → history) you can run cold on any repo, timed to minutes, not an afternoon.
- **Version discipline while reading** — the behavior you're debugging is whatever's pinned in `go.mod`, not whatever GitHub's default branch shows; `go doc -src` reads your actual module cache, browsing does not.
- **`git blame`/`git log -p` as a reading tool**, not just a blame-assigning one — a fifth orientation move most engineers never think to reach for.
- Recognizing **exported-but-internal** symbols (capitalized names under an `internal/` path) as *not* stable API, cross-referencing Lesson 6's compiler-enforced `internal/` boundary.
- A named failure mode — **"the trace that stops too early"** — for the single most common way this skill fails in practice.

## Core concepts

### `go doc` — the fastest orientation tool

No browser, works offline, works on any package in your module graph.

```
go doc net/http                 # package synopsis + exported symbols
go doc net/http.Client          # one type, its fields and methods
go doc net/http.Client.Do       # one method's signature + doc comment
go doc -src net/http.Client.Do  # the actual source of that method
go doc sigs.k8s.io/controller-runtime/pkg/reconcile   # a dependency's package
go doc sigs.k8s.io/controller-runtime/pkg/reconcile.Reconciler
```

`go doc pkg` in 5 minutes tells you what a package is *for*: read the package doc comment (the paragraph at the top), then scan the exported type and function names. `-src` drops you into the implementation without opening an editor — and critically, it reads from **your module cache**, i.e. the exact version pinned in `go.mod`, not whatever is on GitHub's default branch today.

### Editor / gopls moves

`gopls` is the Go language server; these are the four you'll use constantly:
- **Jump-to-definition** (`gd`) — land on where a symbol is defined, across package and module boundaries (including into `controller-runtime`'s source in your module cache).
- **Find-implementations** — given an *interface*, list every concrete type that satisfies it; given a *concrete method*, list the interfaces it satisfies. This is how you cross the interface indirection.
- **Find-references** — "who calls this?"
- **Hover** — signature + doc without navigating away.

### Grep for interface method names — and its blind spot

When gopls isn't handy, an interface is just a set of method names. Grep the method name across the repo to find implementers:

```
grep -rn "func.*Reconcile(ctx context.Context" --include=*.go
grep -rn ") Start(ctx context.Context) error" --include=*.go   # find Runnable impls
```

Signatures are distinctive enough that method-name grep finds implementations fast, even without tooling. A key detail coming from Python: dependencies live *on disk*, in the module cache, so you grep and jump-to-def *into them* exactly as you would your own code — there's no pip black box:

```
go env GOMODCACHE                         # where dependency source lives, read-only
ls $(go env GOMODCACHE)/sigs.k8s.io/      # controller-runtime@vX.Y.Z, etc.
grep -rn "EnqueueRequestForObject" $(go env GOMODCACHE)/sigs.k8s.io/controller-runtime*/pkg/handler
```

This is the single biggest reading advantage over Python: the full, exact source of every tool you operate is already on your machine, versioned, and grep-able. Reading dcgm-exporter's source is `cd $(go env GOMODCACHE)/...` away, or a `git clone` of the tag you run in prod.

**The blind spot: interface embedding.** A flat grep for `func (r *Foo) Reconcile` can miss a real implementer if that type gets `Reconcile` through an **embedded** type instead of declaring the method itself — e.g. `type FooReconciler struct { base.Reconciler }` satisfies the interface via `base.Reconciler`'s method, and grepping `Foo` for the method signature finds nothing. When a type you expect to satisfy an interface doesn't turn up under grep, check its struct fields for an embedded type before concluding the implementation doesn't exist — or just use gopls find-implementations, which resolves embedding correctly because it works off the type system, not text.

### Read `_test.go` files first

Tests encode intent. `reconcile/reconcile_test.go` shows exactly how a `Reconciler` is meant to be called and what `Result{Requeue: true}` means. Example-style tests (`func Example...`) double as runnable docs. When you don't understand a type, find its `*_test.go` and read the first table-driven test — it's the authoritative usage spec, and unlike a blog post explaining the library, it's guaranteed to match the version you're actually running because it's co-located and tested in CI against it. Go's table-test idiom is instantly recognizable and worth reading fluently:

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

### `git blame` / `git log -p` — the fifth orientation tool

A file's history is often the fastest way to understand *intent*, not just current shape — and it's directly available via `git`/`gh` on any cloned dependency, not just your own code:

```
git log -p --follow -- pkg/internal/controller/controller.go   # how this file evolved
git blame pkg/reconcile/reconcile.go -L 40,60                  # who/why for a specific block
gh pr view <n> --repo kubernetes-sigs/controller-runtime        # the PR that introduced a change, with discussion
```

A confusing-looking guard clause or an odd-seeming default often has a one-line commit message or a linked issue explaining exactly why it's there — history answers "why is this like this?" in a way the current snapshot alone cannot. This is underused precisely because it's easy to forget it's available on read-only dependency source, not just your own repo's history.

### Read the pinned version, not `main`

`gopls` and GitHub browsing both default to showing you `main` (or the latest tag). The behavior you're debugging in production is whatever version is **pinned in `go.mod`**, which may be several releases behind. Two habits fix this:
- `go doc -src` reads from your actual module cache — version-correct by construction.
- Check `CHANGELOG.md` / release notes **for the pinned version specifically**, not latest, before concluding "the docs say X" — a behavior you're reading about on GitHub's default branch may not exist yet in the version you run.

Trusting main-branch browsing over your actual pinned version is a real, repeat source of "but the docs I read say X" confusion — always confirm the tag/commit you're reading against `go list -m sigs.k8s.io/controller-runtime` (or equivalent) before trusting an explanation.

### Exported does not mean stable API

Go's `internal/` import restriction (Lesson 6) is compiler-enforced: nothing outside the parent of an `internal/` directory can import it. But **within** an `internal/` package, symbols are still capitalized — still "exported" in the Go sense — because sibling packages inside that same internal tree need to see them. A learner reading `sigs.k8s.io/controller-runtime/pkg/internal/controller` needs to recognize that "exported identifier" does **not** mean "stable public API" the moment the path contains `internal/`: the compiler stops you from importing it from outside, but reading it *as if* it were a stable contract — say, memorizing its exact field layout — leads to a wrong mental model that breaks on the next patch release, with no deprecation notice owed to you.

### Trace a call path conceptually

You rarely single-step Go; you follow types. Start at the entry point (a `main`, a `Builder.Complete`), follow each call into the type it returns, and stop at interface boundaries to run "find-implementations." Keep a scratch list of `file:type.method` as you go — that list *is* the trace.

**The failure mode: the trace that stops too early.** The most common way this skill fails in practice is stopping at the first layer of indirection — e.g. reading `ctrl.NewControllerManagedBy(mgr).For(&v1.Pod{}).Complete(r)`, satisfied that you've "seen the wiring," without following `Complete` into what it actually *registers*. `Complete` is a builder method; the real behavior — which `source.Source`, which `handler.EventHandler`, how the workqueue gets built — lives one or two calls further in. Naming this failure mode explicitly is the point: when a trace feels finished after one hop, that's the moment to ask "what does this call actually construct?" and go one layer deeper.

### The controller-runtime types you must name

These are the vocabulary of every Kubernetes controller:

- `manager.Manager` — owns the shared cache, clients, and the run loop; you register controllers on it and call `mgr.Start(ctx)`.
- `client.Client` — the read/write Kubernetes API client, backed by the manager's cache for reads.
- `reconcile.Reconciler` — the interface *you* implement: `Reconcile(ctx, reconcile.Request) (reconcile.Result, error)`. `Request` carries just a `NamespacedName`; `Result` carries requeue instructions.
- `source.Source` — an event stream (typically a `Kind` source wrapping an informer for one object type).
- `handler.EventHandler` — translates a raw watch event (add/update/delete of an object) into zero or more `reconcile.Request`s enqueued onto the workqueue.
- `workqueue` (`client-go/util/workqueue`) — the rate-limited, deduplicating queue between the informer and your `Reconcile`. This is the load-bearing decoupler.

## Perspectives

**Developer / tooling view.** `go doc` plus gopls jump-to-definition and find-implementations is genuinely different from Python's `help()`/`dir()` — nothing needs to be imported or instantiated to inspect it, because the type system has already written down everything the runtime would tell you. You're reading a static artifact, not probing a live object.

**Operator / on-call view.** This skill is the literal difference between "restart the pod and hope" and "read the exact NVML call feeding a suspicious metric and file a precise upstream bug" during an incident. For a GPU-fleet operator running dcgm-exporter, node-exporter, and their own controllers, every tool's full source is already on disk in `$GOMODCACHE`, not behind an API you have to petition.

**Design / architecture view.** Reading client-go's informer → workqueue → syncHandler pattern *before* reading controller-runtime's abstraction over it (Lesson 9) is the "read the framework's own reference implementation" move — `kubernetes/sample-controller` exists specifically as a canonical, heavily-commented teaching example of the raw pattern, built by the Kubernetes project for exactly this purpose.

**Organizational / review view.** The same reading discipline — read the test first, read the doc comment first, understand before judging complexity — is literally what Google's own public code-review standard asks reviewers to do: understand what a change is doing and why before forming an opinion on how it's done.

## Real-world use cases

- **kubernetes/sample-controller — `controller.go`.** <https://github.com/kubernetes/sample-controller/blob/master/controller.go> — moderately commented, walking through "if it is owned by a Foo resource then enqueue that Foo resource." A real, official Kubernetes-project reference implementation built specifically to teach the informer/workqueue/syncHandler pattern without framework indirection — directly the file this lesson's practice task points you toward.
- **Google — eng-practices, code review guide.** <https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md> — "read the whole CL/file to understand what it does," "prefer simple over clever." The standard Google asks its own reviewers to apply; grounds the Organizational/review perspective above in a real, public engineering standard.
- **Kubernetes official blog — "How the controller-runtime Cache Actually Works, and Why Your Controller Does Not Crash the API Server."** <https://kubernetes.io/blog/2026/07/29/controller-runtime-cache-explained/> — walks through Reflector → delta queue → Indexer/Store → SharedIndexInformer chain inside controller-runtime's cache, explicitly framed as "build a coherent mental model of the internals." This is exactly the trace-a-call-path exercise this lesson teaches, applied to the library your own controller (Lesson 9) will depend on.

## Worked example

**Trace: how does `Reconcile` get invoked when a watched object changes?** Here's the path you'd reconstruct by reading `controller-runtime` with `go doc` + jump-to-def, naming real packages/types — and deliberately *not* stopping at the first layer of indirection.

1. **Wiring** (`pkg/builder/controller.go`). `ctrl.NewControllerManagedBy(mgr).For(&v1.Pod{}).Complete(r)` builds a `controller.Controller` and, for each `For`/`Owns`/`Watches`, registers a **`source.Source`** paired with a **`handler.EventHandler`**. `For` uses a `source.Kind` (backed by an informer from the manager's shared cache) and the `handler.EnqueueRequestForObject` handler. Stopping here is the "trace that stops too early" — `Complete` is a builder method; the real registration happens inside it.

2. **The informer watches the API** (`pkg/cache` → `client-go` informers). The shared cache runs an informer per watched `Kind`. The informer LIST/WATCHes the API server; when a Pod is added/updated/deleted, the informer fires its event handlers. This is `client-go`'s `SharedIndexInformer` under the hood — controller-runtime wraps it as a `source.Kind`.

3. **Event → Request** (`pkg/handler`). The registered `handler.EventHandler` (e.g. `EnqueueRequestForObject`) receives the object event and calls `queue.Add(reconcile.Request{NamespacedName: {Namespace, Name}})`. `EnqueueRequestForOwner` (used by `Owns`) instead maps a child's event to its owner's Request. The handler is the layer that decides *which* key to enqueue.

4. **The workqueue** (`client-go/util/workqueue`). The `Request` lands on the controller's rate-limited workqueue. The queue **deduplicates** (many events for the same Pod collapse to one item) and **rate-limits** (backoff on repeated failures). This decoupling is why a controller stays stable under an event storm.

5. **The worker loop calls Reconcile** (`pkg/internal/controller/controller.go` — note the `internal/` path: exported symbols here are not stable API, per the Core concepts section above). The controller runs N worker goroutines, each looping: `queue.Get()` → build `ctx` → call `reconciler.Reconcile(ctx, req)` → inspect the returned `reconcile.Result` and `error`. On `err != nil` or `Result.Requeue`/`RequeueAfter`, the item is re-added with backoff via `queue.AddRateLimited`; on success, `queue.Forget` + `queue.Done`. Your `Reconcile` then uses `client.Client` to GET the object by `req.NamespacedName` and drive actual state toward desired state.

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

Note `r.Get(ctx, req.NamespacedName, &pod)` — the handler enqueued only the *key*, and the reconciler re-reads live state through the cache-backed `client.Client`. That's the level-triggered contract made concrete. Note also this type satisfies `reconcile.Reconciler` through its own declared `Reconcile` method, not through the embedded `client.Client` — a reminder to check both directions when grepping for implementers.

The commands that build this trace:
```
go doc sigs.k8s.io/controller-runtime/pkg/reconcile.Reconciler
go doc sigs.k8s.io/controller-runtime/pkg/handler
go doc sigs.k8s.io/controller-runtime/pkg/source.Kind
# then jump-to-def from Complete() into pkg/internal/controller, and
grep -rn "reconcile.Reconciler" $(go env GOMODCACHE)/sigs.k8s.io/controller-runtime*/pkg
# and, to see *why* a piece of this is shaped the way it is:
git log -p --follow -- pkg/internal/controller/controller.go | less
```

## Practice

**Task.** Write a one-page `trace.md` of a single real flow, citing concrete files and types.

1. Pick **one exporter you operate** — `node-exporter` or `dcgm-exporter` — and trace how one metric gets from source to `/metrics` (e.g. for dcgm-exporter: NVML/DCGM field → collector → `prometheus.Desc` → registry → HTTP handler). Name the real files and types. (Lesson 7's `Desc`-based custom `Collector` section is your starting vocabulary.)
2. **And** trace `controller-runtime`'s watch→reconcile path: how `Reconcile` gets called when a watched object changes, from `Builder.For`/`source.Kind` through `handler.EventHandler` and the `workqueue` to the worker calling `reconcile.Reconciler.Reconcile`. Name `manager.Manager`, `client.Client`, `source.Source`, `handler.EventHandler`, `workqueue`, `reconcile.Request`/`Result`. Do not stop at `Complete()` — follow it in.
3. Use `go doc`, jump-to-def, find-implementations, reading `_test.go` files, and (at least once) `git log -p`/`git blame` as your method — cite at least one test that clarified intent and one piece of history (a commit message, a changelog entry) that clarified a "why."

**Acceptance.** A committed `trace.md` under `../practice/` (e.g. `../practice/gpu-cost-exporter/trace.md`) that:
- Names **real** files/types/packages (not hand-waved), and
- Is **correct** — the reconcile path goes informer → handler → workqueue → worker → `Reconcile`, and the reconciler receives a `reconcile.Request` (a key), not an event object.
- Is one page, skimmable, with the command(s) you used to find each hop.

## Common pitfalls

1. **Reading the newest/most-starred blog explanation of a library instead of the library's own tests first.** Blog posts go stale; the co-located `_test.go` is guaranteed to match the version you're actually running, because it's tested in CI against it.
2. **Assuming "exported" means "meant for you to depend on" when the symbol is under an `internal/` path.** The compiler stops you from *importing* it, but reading it as if it were a stable contract leads to wrong mental models that break on the next patch release.
3. **Grepping for a method name without accounting for interface embedding.** A `Reconcile` implementation might be satisfied via an embedded type, so a flat grep for `func (r *Foo) Reconcile` can miss real implementers — check embedded fields, or use gopls find-implementations, which resolves this correctly.
4. **Trusting main-branch GitHub browsing over your actual pinned version.** A real, repeat source of "but the docs I read say X" confusion — confirm the tag/commit before trusting an explanation, and prefer `go doc -src` (module-cache-backed) when precision matters.
5. **Stopping at the first layer of indirection** — "the trace that stops too early." Reading `ctrl.NewControllerManagedBy(mgr).For(...).Complete(r)` and calling the trace done, without following `Complete` into what it actually registers, leaves you with a plausible-sounding but incomplete mental model.

## Self-check

**(a) Given an interface, how do you find every implementation?**
**Answer:** Use gopls **find-implementations** on the interface type — it statically enumerates every concrete type in scope whose method set satisfies the interface (Go satisfaction is implicit, so there's no `implements` keyword to grep), and it correctly resolves implementations gained through embedding. Without an editor, grep for the interface's distinctive method signature across `*.go` (e.g. `grep -rn "Reconcile(ctx context.Context, req reconcile.Request)"`), but remember this flat-text approach misses implementers that get the method via an embedded type.

**(b) How do you tell what a package is for in 5 minutes?**
**Answer:** Run `go doc <pkg>`: read the package doc comment (the top paragraph — authors put the purpose there), then scan the list of exported types and top-level functions to see the shape of its API. Open the package's most-central `_test.go` or any `Example` function to see intended usage. Five minutes of `go doc` + one test beats reading the whole source, and needs no running code — and unlike a blog post, both are guaranteed to match the version you actually have pinned.

**(c) Trace: what wakes a reconciler when a watched object changes?**
**Answer:** The manager's shared **cache runs an informer** that WATCHes the API server for that `Kind`. On a change, the informer fires the controller's registered **`handler.EventHandler`**, which enqueues a **`reconcile.Request`** (just the object's `NamespacedName`) onto the rate-limited, deduplicating **`workqueue`**. A **worker goroutine** pops the key and calls your **`reconcile.Reconciler.Reconcile(ctx, req)`**, which re-reads current state via `client.Client`. It's level-triggered: the reconciler gets a *key*, not the event payload, so it re-reads and reconciles idempotently.

**(d) You're debugging behavior that doesn't match a GitHub README you just read. What's the first thing to check?**
**Answer:** Whether the README reflects the version you actually have pinned in `go.mod`, not `main`. Check `go list -m <module>` for the resolved version, then read that version's source with `go doc -src` (which reads your module cache, not GitHub) or check the `CHANGELOG.md`/release notes for that specific tag. GitHub's default branch view is a common, repeat source of "but the docs say X" confusion precisely because it silently shows you code ahead of what you're running.

## Connections & what's next

This lesson turns the interfaces-and-composition instincts from Lessons 2–3 into a reading skill, and it's the direct prerequisite for Lesson 9: you're about to build a controller on top of `controller-runtime`, and the manager → informer → workqueue → `Reconcile` trace you just rehearsed here *is* the mental model that lesson assumes on day one. It also closes the loop with Lesson 6's `internal/` boundary (now you know why "exported but internal" isn't a contract) and Lesson 7's exporter idiom (now you have the tool to go read the real dcgm-exporter/node_exporter source that idiom was modeled on).

**Next: Lesson 9, [Controller primer (CRD · reconcile · envtest)](09-controller-primer.md).** You now know how to read the framework; next you write against it — a CRD plus a reconcile loop, tested with envtest, that becomes the seed of your capstone GPU cost/efficiency operator.

## References & further reading

**Primary sources**
- **controller-runtime** — [repo](https://github.com/kubernetes-sigs/controller-runtime) · [pkg docs](https://pkg.go.dev/sigs.k8s.io/controller-runtime) — the source you're tracing; read `pkg/reconcile`, `pkg/handler`, `pkg/source`, `pkg/builder`, and `pkg/internal/controller`. Start from the pkg docs' subpackage list, then jump into source.
- **`go doc`** — [command docs](https://pkg.go.dev/cmd/go#hdr-Show_documentation_for_package_or_symbol) — the exact flags (`-src`, `-all`, symbol/method addressing) for the orientation workflow above.

**Real-world engineering blogs**
- **kubernetes/sample-controller — `controller.go`** — <https://github.com/kubernetes/sample-controller/blob/master/controller.go> — what it shows: the canonical, heavily-commented informer→workqueue→syncHandler pattern in raw `client-go`, no framework indirection — read it once end to end; it makes the controller-runtime abstraction click.
- **Google — eng-practices, code review guide** — <https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md> — what it shows: the reading-before-judging discipline this lesson teaches, as an actual public engineering standard.

**Deeper dives**
- **Kubernetes official blog — "How the controller-runtime Cache Actually Works, and Why Your Controller Does Not Crash the API Server"** — <https://kubernetes.io/blog/2026/07/29/controller-runtime-cache-explained/> — walks the Reflector → delta queue → Indexer/Store → SharedIndexInformer chain; the same trace-a-call-path exercise applied one layer deeper into the cache your Lesson 9 controller depends on.
