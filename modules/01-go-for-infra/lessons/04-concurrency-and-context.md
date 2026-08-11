---
lesson: "01.4"
title: "Concurrency and context"
module: "01"
concept: "Concurrency and context"
status: not-started
est_time: "18h"
prev: "03-interfaces-and-composition.md"
next: "05-testing.md"
artifacts: []
sources: 11
---

# 01.4 · Concurrency and context

> **Concept.** Goroutines, channels, select, sync, errgroup, worker pools, leak/race detection — and context for cancellation, deadlines and propagation, pervasive in every k8s API call and reconcile.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 03 gave you the tool this lesson depends on: small, consumer-defined interfaces you can fake without a mocking library. That's what makes everything here testable — you cannot write a fast, deterministic test for a bounded concurrent fan-out without a fake standing in for the thing being fanned out to. This lesson is the module's largest for a reason: a Kubernetes controller *is* a concurrency engine, and an exporter scraping a GPU fleet is a concurrency problem before it's anything else. What lesson 03 made fakeable, this lesson makes concurrent, bounded, and cancellable — and that combination is what lesson 05's tests will assert against (`goleak.VerifyNone`, an in-flight counter that never exceeds a limit).

## Why this matters

A Kubernetes controller *is* a concurrency engine. controller-runtime runs a work queue and spins up N reconcile loops in parallel (`MaxConcurrentReconciles`). Every client call — `Get`, `List`, `Update`, `Patch` — takes a `context.Context` as its first argument, and the manager cancels that context on `SIGTERM` so in-flight work drains instead of wedging a pod mid-shutdown. An exporter scrapes dozens of nodes/GPUs concurrently but must cap concurrency so it doesn't hammer the kubelet or a cloud billing API, and must respect Prometheus's scrape timeout. Get this wrong and you ship the classic production failures: a goroutine leak that grows RSS until OOMKill, a data race that corrupts a shared cache under load (invisible until it's a 2am incident), or a reconcile that ignores cancellation and blocks graceful shutdown past the pod's `terminationGracePeriodSeconds`, turning a rolling update into an outage.

Interview stakes: for a senior platform/Go role you will be asked to bound concurrency, explain what `go test -race` catches, show a goroutine leak and fix it, and explain why `ctx` threads through every call. These are table stakes, not trivia — they separate "wrote a script" from "operates fleets."

## What's new here (calibration)

Per the module README's calibration: you already understand concurrency as a concept from operating distributed systems, so we skip re-teaching what a thread or a race condition *is* in the abstract, and we skip runtime-internals rabbit holes (the scheduler's M:N implementation, work-stealing internals) — that's the "runtime internals" the README explicitly says to leave alone. What's new: Go's specific primitives (goroutines, channels, `select`) and their exact rules, `context.Context` as the pervasive cancellation/deadline vocabulary threaded through every k8s API call, `errgroup` as the production bounded-fan-out pattern, and the failure modes that are genuinely Go-shaped — the missing GIL turning sloppy shared state into real undefined behavior, GOMAXPROCS/cgroup mismatches, and goroutine leaks as a slow-burn production signature rather than a crash.

## Core concepts

### The delta from Python

The single biggest mental shift: **Go has no GIL.** In CPython, one interpreter lock means threads don't run Python bytecode truly in parallel, so a sloppy `dict` mutation across threads usually *happens* to be safe. In Go, goroutines run on multiple OS threads on multiple cores simultaneously — two goroutines writing the same map or `int` is a genuine data race with undefined behavior (torn reads, corrupted maps that panic, stale values). The tool that catches this is the race detector: **`go test -race` / `go run -race`**. Run it in CI. It is your `-Werror` for concurrency.

- **Goroutine ≠ thread ≠ coroutine.** A goroutine is a green thread: ~2 KB initial stack, multiplexed by the runtime onto a small pool of OS threads. You launch millions cheaply with `go f()`. Unlike `asyncio`, there is **no `async`/`await` coloring** — any function can be called in a goroutine, and blocking I/O is fine (the runtime parks the goroutine and reuses the OS thread). Unlike Python threads, they give real parallelism.
- **Channels vs queues.** A channel is a typed, synchronized pipe — think `queue.Queue`, but a first-class language primitive with `select` for multiplexing and *closing* to signal "no more data." Unbuffered channels also synchronize (a send blocks until a receive), which `queue.Queue` doesn't do the same way.
- **The motto:** *"Don't communicate by sharing memory; share memory by communicating."* Prefer handing a value to exactly one owner over a channel to protecting shared state with locks. But it's guidance, not dogma — a counter or a cache is simpler and faster behind a `sync.Mutex` than behind a channel-guarded goroutine.

### Goroutines and the leak they hide

```go
go func() { doWork(ctx) }()   // fire-and-forget: launches, main does NOT wait
```

`go` returns immediately. If `main` returns, the program exits and kills every goroutine — so you need explicit synchronization (`WaitGroup`, channel, or `errgroup`) to wait. Every goroutine you start must have a guaranteed path to exit, or it leaks.

### Channels: unbuffered vs buffered

```go
ch := make(chan int)      // unbuffered: send blocks until a receiver is ready (rendezvous)
ch := make(chan int, 8)   // buffered:   send blocks only when 8 items are unconsumed
close(ch)                 // signal "no more sends"; receivers get zero value + ok==false
v, ok := <-ch             // ok==false once drained after close
for v := range ch { ... } // loops until ch is closed
```

Rules that bite: send on a closed channel **panics**; closing twice **panics**. Only the *sender* closes, and only when it's the sole sender. A receive on a `nil` channel blocks forever (occasionally useful in `select`). Buffered channels are a bounded queue — and, sized to N, a **semaphore**.

### select — multiplex, with default for non-blocking

```go
select {
case v := <-in:      handle(v)
case out <- x:       // send if a receiver is ready
case <-ctx.Done():   return ctx.Err()   // cancellation arm — in almost every select
default:             // runs if no other case is ready → makes the select non-blocking
}
```

Without `default`, `select` blocks until one case is ready. With it, it's a non-blocking poll. The `<-ctx.Done()` arm is how blocking operations become cancellable.

### sync — when a lock is the right tool

```go
var mu sync.Mutex           // full mutual exclusion
mu.Lock(); count++; mu.Unlock()

var rw sync.RWMutex         // many readers OR one writer — for read-heavy caches
rw.RLock(); v := m[k]; rw.RUnlock()

var once sync.Once          // run exactly once (lazy init, singletons)
once.Do(func(){ initExpensive() })

var wg sync.WaitGroup       // wait for a fixed set of goroutines
for _, t := range tasks {
    wg.Add(1)
    go func() { defer wg.Done(); process(t) }()
}
wg.Wait()
```

`sync/atomic` covers single-word counters/flags without a lock. **When NOT to use a channel:** guarding one struct field, a counter, or a cache — reach for a `Mutex`; it's simpler, faster, and easier to reason about than a goroutine that owns the state behind a channel.

### The Go memory model: why unsynchronized access is undefined, not just "sometimes wrong"

Go's memory model (formalized for `sync`/`sync/atomic` as of Go 1.19, see the [package docs](https://pkg.go.dev/sync/atomic)) makes a stronger claim than "races are risky": **without a synchronization point, there is no guarantee one goroutine ever observes another's writes at all**, in any order, ever — not "usually consistent," genuinely undefined. A synchronization point is a channel operation, a mutex lock/unlock, a `sync/atomic` operation, or a `WaitGroup` completion. This is the formal backing for "share memory by communicating": a channel send/receive is a synchronization point with a defined happens-before relationship (everything before the send is visible to everything after the corresponding receive); a bare shared variable with no such point has no such guarantee, regardless of how reliably it "worked" in testing. This is also why `go test -race` matters more in Go than a linter-style nice-to-have: a race isn't a rare timing bug you might hit, it's a program with no defined behavior that happens to look fine on your laptop.

### errgroup — the production concurrency primitive

`golang.org/x/sync/errgroup` is what you actually use for fan-out. It's a `WaitGroup` that also collects the first error and cancels a derived context.

```go
g, ctx := errgroup.WithContext(ctx)   // ctx is cancelled when any g.Go returns non-nil error
g.SetLimit(N)                          // cap to N goroutines running at once (errgroup.SetLimit; x/sync ≥ v0.1.0)
for _, item := range items {
    item := item
    g.Go(func() error { return scrape(ctx, item) }) // blocks here once N are in flight
}
err := g.Wait()   // waits for all; returns the first non-nil error
```

`SetLimit(N)` is the bounded worker pool — no manual semaphore needed. On first error the derived `ctx` is cancelled, so cancellation-aware in-flight work unwinds instead of running to completion. This pattern's panic-propagation design was directly informed by a Go-team engineer's work on rethinking classical concurrency patterns (see Bryan Mills's GopherCon talk in Deeper dives) — `errgroup` isn't a random convenience wrapper, it's the distillation of hard lessons about how fan-out should fail.

**Important limit on what cancellation buys you:** `errgroup.WithContext`'s cancellation only aborts operations that actually *check* `ctx.Done()` or pass `ctx` into something that does (an HTTP call, a DB query). Pure CPU-bound computation with no `select` or `ctx.Err()` check inside it runs to completion regardless of cancellation — cancelling the context doesn't reach into a running loop and stop it. If a `g.Go` closure does heavy in-process math, it needs its own explicit `ctx.Done()` check to actually be cancellable.

### Bounded worker pool the manual way (semaphore channel)

When you need it without errgroup, a buffered channel is the semaphore:

```go
sem := make(chan struct{}, N)          // capacity N = max concurrency
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func() {
        defer wg.Done()
        sem <- struct{}{}              // acquire (blocks when N in flight)
        defer func() { <-sem }()      // release
        process(item)
    }()
}
wg.Wait()
```

Watch the off-by-one here: sizing `sem` to `N` bounds how many goroutines can *hold the semaphore* at once, but if you launch all `len(items)` goroutines unconditionally (as above) rather than gating the *launch* itself, you still spike memory at high fan-out — every launched goroutine exists, with its stack allocated, whether or not it has acquired the semaphore yet. For a bounded launch as well as bounded execution, gate the loop itself (a worker-pool-with-input-channel shape, or just use `errgroup.SetLimit`, which blocks inside `g.Go` before the closure's goroutine is even started).

### The three classic goroutine leaks — and detection

1. **Blocked send, no receiver.** A goroutine does `ch <- v` but the reader returned early (e.g. on error). Send blocks forever. **Fix:** buffer the channel, or `select { case ch<-v: case <-ctx.Done(): }`.
2. **Blocked receive, no sender.** A goroutine ranges over a channel that is never closed / never sent to. **Fix:** ensure the sender closes, add a `ctx.Done()` arm.
3. **Forgotten cancel / no exit path.** A helper goroutine loops forever with no cancellation signal. **Fix:** pass `ctx`, `select` on `ctx.Done()`; always `defer cancel()` after `WithCancel`/`WithTimeout`.

Detection: `go test -race` finds data races — **it does not find leaks**; those are two different bug classes and passing `-race` proves nothing about leaks. For leaks: `go.uber.org/goleak` in tests (`defer goleak.VerifyNone(t)`) fails if goroutines outlive the test; in production, watch the `go_goroutines` Prometheus metric — a monotonic climb over hours or days, not a crash, is the actual production signature of a leak. `runtime.NumGoroutine()` and `/debug/pprof/goroutine` confirm it. Uber's engineering blog describes exactly this two-pronged strategy at scale — `goleak` gating CI so a leak-introducing PR never merges, plus a lightweight production profiler (LeakProf) that samples goroutine stacks continuously and alerts on leak symptoms before RSS growth pages someone (see Real-world use cases below).

### context — cancellation, deadlines, propagation

`context.Context` carries a cancellation signal, an optional deadline, and request-scoped values across API boundaries and goroutines.

```go
ctx, cancel := context.WithCancel(parent)       // cancel() to stop
ctx, cancel := context.WithTimeout(parent, 5*time.Second)  // auto-cancel after 5s
ctx, cancel := context.WithDeadline(parent, t)  // auto-cancel at absolute time t
defer cancel()                                   // ALWAYS — releases the timer/resources
```

- **First argument, always:** `func Scrape(ctx context.Context, node string) (...)`. Never a struct field.
- **`ctx.Done()`** returns a channel closed on cancellation; **`ctx.Err()`** then returns `context.Canceled` or `context.DeadlineExceeded`.
- **Propagation:** deriving `child, _ := context.WithTimeout(parent, ...)` builds a tree. Cancelling a parent cancels all descendants. This is how one `SIGTERM` unwinds an entire reconcile.
- **Who honors it:** a `ctx` only cancels work if the **blocking call checks it.** `net/http`, database drivers, gRPC and the k8s client all take `ctx` and abort on cancellation. Your own CPU loop does not — you must `select { case <-ctx.Done(): return ctx.Err() default: }` or check `ctx.Err()` between iterations.
- **Never store `ctx` in a struct**; pass it per-call. A stored ctx outlives the request it belonged to and silently defeats cancellation — the request that ctx belonged to is long gone, but code holding the struct keeps using a ctx that will never itself be re-derived or re-cancelled for the new work.

**What `context.WithValue` actually costs, and why it's the wrong tool for parameters.** `context.Value(key)` isn't a map lookup — a context built from nested `WithValue` calls is a linked list of parent contexts, and `Value()` walks that chain looking for a match, an O(depth) operation on every call. That's fine for a handful of well-known keys threaded through deep call stacks — a trace ID, a deadline, an auth principal — where you're trading a small, bounded lookup cost for not having to thread an explicit parameter through every intermediate function signature. It's the wrong tool as a general parameter bag: passing business logic through `context.Value` hides the dependency from the function signature (you can't tell what a function needs by reading its parameters), loses type safety (`Value` returns `any`), and grows the walk depth with every layer that adds another key.

### Cooperative vs async preemption (Go 1.14+)

Before Go 1.14, the scheduler could only preempt a goroutine at specific points — function calls, primarily. A tight loop with no function calls (`for i := 0; i < 1e12; i++ { sum += i }`) could starve the scheduler and stall other goroutines on that OS thread indefinitely. Go 1.14 added **signal-based asynchronous preemption**, letting the runtime interrupt a running goroutine even mid-loop. Older "why did my goroutine never get scheduled" folklore from pre-1.14 codebases is now largely obsolete — worth knowing as history so you don't over-apply an outdated workaround (manually inserting `runtime.Gosched()` calls into hot loops) to a modern toolchain where it's no longer necessary.

### GOMAXPROCS and containers

Go's scheduler multiplexes goroutines onto `GOMAXPROCS` OS threads — by default, the number of logical CPUs Go detects. Before Go 1.25, that detection read the **host's** CPU count, which is wrong inside a cgroup-limited container: a pod with a `resources.limits.cpu: "2"` quota running on a 64-core node would see `GOMAXPROCS=64`, over-parallelize, get CPU-throttled by the cgroup, and show elevated latency despite "only" 2 CPUs of actual demand. This was a real, common source of over-parallel, throttled, high-latency containers, and `uber-go/automaxprocs` was a widely-adopted workaround library that read the cgroup limit and called `runtime.GOMAXPROCS()` explicitly at startup. **Go 1.25 (released Aug 2025)** is the version where the runtime itself became container-aware: on Linux it auto-detects the cgroup CPU bandwidth limit and defaults `GOMAXPROCS` to it (and periodically re-reads it if the limit changes). Because that landed only in 1.25, `automaxprocs` was necessary for essentially all production Go through **Go 1.24** — so the failure mode is still live wisdom on any 1.24-or-earlier toolchain or pinned base image. Check your Go version before assuming this is handled.

### Channel direction & where this lands in a controller

Type a channel's direction at API boundaries to encode intent and let the compiler enforce it: `func produce(out chan<- T)` (send-only), `func consume(in <-chan T)` (receive-only). It documents ownership — the producer closes `out`, consumers never do.

In controller-runtime this all shows up concretely:

```go
// setup: bound the parallel reconciles the manager runs
ctrl.NewControllerManagedBy(mgr).
    For(&v1.GPUNode{}).
    WithOptions(controller.Options{MaxConcurrentReconciles: 4}).
    Complete(r)

// reconcile: ctx is handed to you already; thread it into EVERY client call
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var node v1.GPUNode
    if err := r.Get(ctx, req.NamespacedName, &node); err != nil { // ctx-aware; aborts on shutdown
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    // ... never launch a bare `go` here without a ctx-linked exit path.
    return ctrl.Result{}, nil
}
```

`MaxConcurrentReconciles` is your `SetLimit` for the reconcile loop — the same bounded-concurrency lever. The manager cancels the `ctx` it passes on `SIGTERM`; your job is to not ignore it.

## Perspectives

**Developer view.** `errgroup.WithContext` + `SetLimit` is the one pattern to reach for by default for any fan-out. It's not a special-purpose tool — it *is* `MaxConcurrentReconciles` under a different name, and the same bounded-fan-out shape recurs at every layer of a controller and an exporter: bounded reconciles, bounded scrapes, bounded outbound API calls. Learn it once, apply it everywhere.

**Operator view.** A goroutine leak is invisible until it isn't. The actual production signature is `go_goroutines` climbing monotonically on a Grafana panel for days — not a crash, not an error log, just a slow slope upward. RSS grows because each leaked goroutine holds its stack and whatever it's referencing; GC pause times grow because pause time scales with live-heap size, not garbage-collection frequency; eventually the pod OOMKills. It's a uniquely hard on-call pattern precisely because nothing pages until the very end, and by then the postmortem has to reconstruct days of slow decay from a metric nobody was watching closely.

**Hardware/runtime view.** Go's scheduler multiplexes goroutines onto `GOMAXPROCS` OS threads, and before Go 1.25 `GOMAXPROCS` defaulted to the *host's* CPU count — wrong inside a cgroup-limited container. Pre-1.25 this was a real source of over-parallel, throttled, high-latency containers; Go 1.25 (Aug 2025) made the runtime cgroup-aware, but the failure mode existed for years — through Go 1.24 — and `automaxprocs`-style libraries were the widely-adopted workaround before the fix landed upstream. It's a good example of a bug class that's specifically about where your Go binary runs, not what it does.

**Economics/scale view.** Concurrency limits are literally a cost lever, not just a correctness one. An unbounded scrape fan-out against a billing API or the kubelet can trigger rate limiting or directly cost money per request at scale. A too-conservative limit under-utilizes GPUs sitting idle waiting on CPU-bound preprocessing — exactly the failure mode Google Cloud describes IceCube hitting (see Real-world use cases). Tuning concurrency is tuning a dial between "too slow, wasting expensive idle hardware" and "too fast, getting throttled or billed for it."

## Real-world use cases

- **[Google Cloud — "GKE GPU sharing helps scientists' quest for neutrinos"](https://cloud.google.com/blog/products/containers-kubernetes/gke-gpu-sharing-helps-scientists-quest-for-neutrinos)** — IceCube Observatory's ray-tracing simulation became CPU-bound, leaving GPUs (V100/A100) underutilized while CPU-bound preprocessing lagged behind; GKE GPU time-sharing and A100 MIG partitioning let multiple jobs share one GPU, delivering a reported 4.5x throughput increase on the most CPU-bound workload. A concrete, GPU-fleet-recognizable illustration of the economics perspective above. *Note: the reported throughput/hardware figures are a dated snapshot — treat them as illustrative, not current pricing or performance.*
- **[Google Cloud — "Testing Cloud Pub/Sub clients to maximize streaming performance"](https://cloud.google.com/blog/products/data-analytics/testing-cloud-pubsub-clients-to-maximize-streaming-performance)** — a rare vendor blog with an actual numeric concurrency-tuning data point: Go's Pub/Sub client throughput peaks around 8 goroutines per CPU core (roughly 128 workers on a 16-core machine), and the client's default setting is deliberately more conservative than that peak, as a safety mechanism against overload. *Note: the specific goroutines-per-core figure is workload- and hardware-dependent — treat it as a dated snapshot and a starting point for your own benchmarking, not a universal constant.*
- **[Uber Engineering — "LeakProf: Featherlight In-Production Goroutine Leak Detection"](https://www.uber.com/blog/leakprof-featherlight-in-production-goroutine-leak-detection)** — Uber's two-pronged strategy: `goleak` in CI blocks PRs that introduce leaks, and LeakProf, a lightweight production profiler, continuously samples goroutine stacks and alerts on leak symptoms before they page as an OOM. Shows what a mature org automates beyond "watch `go_goroutines` on a dashboard." The open-source CI half is [go.uber.org/goleak](https://github.com/uber-go/goleak).
- **[Discord — "Why Discord is switching from Go to Rust"](https://discord.com/blog/why-discord-is-switching-from-go-to-rust)** — a legitimate counterpoint, not a horror story: Discord hit GC-driven latency spikes in a service whose heap stayed full of live objects (a bad fit for a tracing garbage collector, which ran roughly every 2 minutes regardless of how much garbage had actually accumulated). A fair, specific example of a real limit of Go's concurrency/memory model, worth knowing so "just use Go" isn't treated as a universal answer.

## Worked example

Fan out N concurrent HTTP calls under a deadline, bounded to `limit` concurrency, aggregating results; cancel everything in flight on the first error. Runs clean under `-race`.

```go
package fetch

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"time"

	"golang.org/x/sync/errgroup"
)

type Result struct {
	URL  string
	Code int
}

// FetchAll fetches every url with at most `limit` concurrent requests,
// under an overall `budget` deadline. First error cancels all in-flight work.
func FetchAll(ctx context.Context, client *http.Client, urls []string, limit int, budget time.Duration) ([]Result, error) {
	ctx, cancel := context.WithTimeout(ctx, budget)
	defer cancel()

	g, ctx := errgroup.WithContext(ctx) // cancels ctx on first non-nil error
	g.SetLimit(limit)                   // bounded worker pool

	results := make([]Result, len(urls)) // index-per-goroutine: no shared write, no lock needed
	for i, u := range urls {
		i, u := i, u
		g.Go(func() error {
			req, err := http.NewRequestWithContext(ctx, http.MethodGet, u, nil)
			if err != nil {
				return fmt.Errorf("build %s: %w", u, err)
			}
			resp, err := client.Do(req) // aborts if ctx is cancelled/expired
			if err != nil {
				return fmt.Errorf("get %s: %w", u, err)
			}
			defer resp.Body.Close()
			_ = json.NewDecoder(resp.Body).Decode(&struct{}{})
			results[i] = Result{URL: u, Code: resp.StatusCode}
			return nil
		})
	}
	if err := g.Wait(); err != nil {
		return nil, err // in-flight requests already cancelled via ctx
	}
	return results, nil
}
```

Why it's race-free: each goroutine writes only `results[i]` for its own `i` — disjoint slots, no shared mutation, so no mutex. If aggregation needed a shared map, guard it with `sync.Mutex`. `defer cancel()` guarantees the timer is released even on the happy path. `SetLimit(limit)` means only `limit` requests are ever open at once — the acceptance property an exporter needs. Note also what this example does *not* protect against: if `client.Do` were replaced with a pure CPU-bound computation that never checks `ctx`, cancelling `ctx` here would stop *new* work from being launched but would not interrupt work already running — the pitfall named in Core concepts above.

## Practice

Build the collection stage of [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md) as a **bounded concurrent scrape**.

- `Collect(ctx context.Context, nodes []Node, limit int) ([]Sample, error)` scrapes per-node GPU metrics concurrently, capped at `limit` in-flight (use `errgroup` + `SetLimit`).
- Derive a scrape deadline from `ctx` (`WithTimeout`) so a slow node can't blow past Prometheus's scrape timeout; on timeout the scrape returns `context.DeadlineExceeded` and no goroutine keeps running.
- First hard error cancels all in-flight scrapes; aggregate successful samples into a slice (disjoint indices — no lock) or a mutex-guarded map.
- Wire it behind a `prometheus.Collector`'s `Collect` method later; for now, a `func` + tests.
- Use the `CostSource`-style fake from lesson 03 to drive the scrape in tests without a live target — no real HTTP endpoints needed to exercise the bounded-concurrency behavior.

**Acceptance:**
- `go test -race ./...` passes (no data race).
- No goroutine leak — assert with `defer goleak.VerifyNone(t)` in the test.
- Cancelling the passed `ctx` (or deadline expiry) returns promptly with `ctx.Err()` and leaves zero goroutines running.
- Concurrency never exceeds `limit` (assert with an atomic in-flight counter that peaks at ≤ `limit`).

## Common pitfalls

1. **Treating `-race` as a leak detector.** It only catches data races, not goroutine leaks. Correction: use `goleak.VerifyNone(t)` in tests, and watch `go_goroutines` (or `/debug/pprof/goroutine`) in production — these are two separate detection tools for two separate bug classes.
2. **Storing `ctx` in a struct field "for convenience."** This defeats cancellation semantics silently: the captured `ctx` outlives the request it belonged to, so code using it later has no relationship to the cancellation that should apply to the *new* work. Correction: pass `ctx` as the first argument to every function that needs it, never as stored state.
3. **Assuming `errgroup.WithContext`'s cancellation aborts CPU-bound work.** It only cancels operations that actually check `ctx.Done()`; pure computation with no `select`/context check runs to completion regardless of how many other goroutines errored out. Correction: CPU-bound work that should be cancellable needs its own explicit `ctx.Err()` check between chunks of work.
4. **Buffered-channel-as-semaphore off-by-one.** Sizing the semaphore channel to `N` but launching `N+1`-or-more unbounded goroutines that each try to acquire it (rather than gating the *launch* itself) still spikes memory at high fan-out — every launched goroutine's stack exists whether or not it holds the semaphore yet. Correction: gate the launch loop itself, or use `errgroup.SetLimit`, which blocks before starting the goroutine.
5. **GOMAXPROCS mismatch inside a container.** Real, and still relevant on pre-Go-1.25 toolchains (Go 1.24 or earlier) or older pinned base images: `GOMAXPROCS` defaults to the host's CPU count, over-parallelizing and getting throttled inside a cgroup-limited pod. Correction: confirm your Go version is cgroup-aware (1.25+), or explicitly set `GOMAXPROCS` (or use `automaxprocs`) on older toolchains.

## Self-check

**(a)** You must scrape 200 nodes but keep at most 8 requests in flight, and abort every in-flight request the instant one node returns a fatal error. How?

**Answer:** `g, ctx := errgroup.WithContext(ctx)` then `g.SetLimit(8)`; launch each scrape with `g.Go(func() error { return scrape(ctx, node) })` and `g.Wait()`. `SetLimit(8)` bounds concurrency to 8; the first non-nil error returned from any `g.Go` cancels the derived `ctx`, and every scrape uses that `ctx` (`http.NewRequestWithContext`), so all in-flight requests abort. `g.Wait()` returns that first error.

**(b)** Show a goroutine leak and fix it.

**Answer:** Leak — the receiver returns early, so the sender blocks forever:
```go
func leak() {
	ch := make(chan int)          // unbuffered
	go func() { ch <- expensive() }() // blocks forever; nobody receives
	return                         // goroutine stranded, stack never freed
}
```
Fix — give the send an escape via a cancellable context (or buffer the channel `make(chan int, 1)`):
```go
func fixed(ctx context.Context) {
	ch := make(chan int, 1)       // buffered: send never blocks
	go func() { ch <- expensive() }()
	select {
	case v := <-ch: use(v)
	case <-ctx.Done():            // caller cancels → goroutine's buffered send still completes, then GC's
	}
}
```

**(c)** Why is `ctx` always the first argument, and what must a blocking call do with it?

**Answer:** Convention makes cancellation/deadline propagation uniform and greppable — every layer threads the same `ctx` down the call tree so a single cancel unwinds the whole operation (e.g. `SIGTERM` → manager cancels root ctx → every reconcile and API call aborts). A blocking call must *honor* it: pass `ctx` into the underlying I/O (which selects on `ctx.Done()` internally), or in your own loops `select { case <-ctx.Done(): return ctx.Err() ...}` / check `ctx.Err()` between iterations, then return promptly. A ctx that isn't checked cancels nothing. And never store it in a struct — pass per call, so it can't outlive its request.

**(d)** Two goroutines write to the same `map[string]int` with no lock, no channel, no atomic. `go test -race` isn't run. Is this "usually fine" the way it might be in CPython?

**Answer:** No — and this is the sharpest Python-to-Go delta in this lesson. Go has no GIL, so both goroutines can genuinely run in parallel on different cores. Per the Go memory model, an unsynchronized write in one goroutine has no guaranteed visibility to a read in another without a synchronization point (channel op, mutex, `sync/atomic`, `WaitGroup`). This isn't "probably fine, rarely wrong" — it's undefined behavior, and for a map specifically the concrete failure mode is often a runtime panic ("concurrent map writes") rather than a silently wrong value. `go test -race` is how you catch this before production; skipping it in CI is not a minor gap.

**(e)** Your container has a `cpu: "2"` limit but the node has 64 cores. What Go-specific behavior can bite you here, and on which Go versions?

**Answer:** Pre-Go-1.25, `GOMAXPROCS` defaults to the *host's* CPU count (64), not the cgroup limit (2) — so the Go scheduler over-parallelizes onto far more OS threads than the container is actually entitled to run simultaneously, gets CPU-throttled by the cgroup, and shows elevated latency despite modest real demand. Go 1.25 (Aug 2025) made the runtime cgroup-aware — it auto-detects the cgroup CPU limit and sets `GOMAXPROCS` accordingly, closing the gap. On Go 1.24 or earlier, or a pinned toolchain, the fix is to set `GOMAXPROCS` explicitly or use a library like `automaxprocs` that reads the cgroup limit at startup.

## Connections & what's next

This lesson leans directly on lesson 03: bounding concurrent work with `errgroup` and testing it without live dependencies both require the small, consumer-defined, fakeable interfaces that lesson built. It also completes the picture lesson 02 started with `ctx`-aware error wrapping — cancellation errors (`context.Canceled`, `context.DeadlineExceeded`) flow through the same `%w`-wrapped error chains. Lesson 05 (testing) is where `goleak.VerifyNone` and the in-flight-counter assertions from this lesson's Practice task become first-class test infrastructure, and where you'll see `-race` and leak detection running together in CI as a standing requirement. Lesson 09 (controller primer) is where `MaxConcurrentReconciles` and the manager's `SIGTERM`-driven context cancellation stop being an analogy and become the literal reconcile loop you write against.

Next: **[05 · Testing](05-testing.md)** — the fakes from lesson 03 and the bounded, cancellable, leak-checked concurrency from this lesson are what a serious Go test suite is actually testing.

## References & further reading

**Primary sources**
- [The Go Blog — "Go Concurrency Patterns: Context"](https://go.dev/blog/context) — the canonical rationale for `context.Context`: first arg, `Done()`, cancellation propagation. Read for the source these rules trace back to.
- [`sync/atomic` package docs (Go Memory Model)](https://pkg.go.dev/sync/atomic) — read for the formal happens-before guarantees behind "share memory by communicating," and why unsynchronized access is undefined rather than merely risky.

**Real-world engineering blogs**
- [Google Cloud — "GKE GPU sharing helps scientists' quest for neutrinos"](https://cloud.google.com/blog/products/containers-kubernetes/gke-gpu-sharing-helps-scientists-quest-for-neutrinos) — what it shows: a CPU-bound preprocessing stage leaving GPUs idle, and concurrency/scheduling changes (GPU time-sharing, MIG) fixing the utilization gap. *Dated snapshot — treat throughput figures as illustrative.*
- [Google Cloud — "Testing Cloud Pub/Sub clients to maximize streaming performance"](https://cloud.google.com/blog/products/data-analytics/testing-cloud-pubsub-clients-to-maximize-streaming-performance) — what it shows: a rare concrete numeric concurrency-tuning benchmark (goroutines per core) from a vendor client library. *Dated snapshot — hardware- and workload-specific.*
- [Uber Engineering — "LeakProf: Featherlight In-Production Goroutine Leak Detection"](https://www.uber.com/blog/leakprof-featherlight-in-production-goroutine-leak-detection) — what it shows: what a mature org automates beyond a dashboard metric — CI-blocking `goleak` plus a lightweight production profiler. Open-source CI half: [go.uber.org/goleak](https://github.com/uber-go/goleak).
- [Discord — "Why Discord is switching from Go to Rust"](https://discord.com/blog/why-discord-is-switching-from-go-to-rust) — what it shows: a fair, specific counterpoint — GC-driven latency spikes as a real limit of Go's concurrency/memory model for one workload shape, not a universal indictment.

**Deeper dives**
- **GopherCon 2018 — Bryan C. Mills, "Rethinking Classical Concurrency Patterns"** — https://www.youtube.com/watch?v=5zXAHh5tJqQ. A Go-team engineer at Google whose thinking informed `errgroup`'s panic-propagation design; watch for the reasoning behind bounded fan-out, not just the API.
- **Learn Concurrent Programming with Go — James Cutajar (Manning, 2024).** *What-for:* modern, from-first-principles treatment of goroutines, channels, `select`, and the sync primitives with the race detector front and center. *How:* deep-read chs. on memory model, channels, and the classic problems; skim the OS-threads intro since you know it. *Why:* current (Go 1.21+) and builds the "no GIL → real races" mental model a Python-native engineer needs. (Alternative: *Concurrency in Go*, Cox-Buday — excellent on patterns/pipelines but older; pick one, not both.)
- **go-concurrency-exercises** — https://github.com/loong/go-concurrency-exercises. *What-for:* test-verified drills (bounded pool, rate limiting, graceful shutdown, deadlock fixes). *How:* do, don't read — solve 4–5 with `go test -race`. *Why:* muscle memory for the exact patterns in the practice task; the failing tests catch races you'd otherwise ship.
- **100 Go Mistakes — concurrency section** — https://100go.co. *What-for:* the specific traps: leaking goroutines, closing channels wrong, ctx propagation, `sync` misuse, loop-variable capture. *How:* skim the concurrency chapter now, reread when a `-race` failure baffles you. *Why:* every entry is a real production bug you want to have already seen once.
