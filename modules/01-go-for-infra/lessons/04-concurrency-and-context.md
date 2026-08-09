---
lesson: "01.4"
title: "Concurrency and context"
module: "01"
concept: "Concurrency and context"
status: not-started
est_time: "16h"
artifacts: []
---

# 01.4 · Concurrency and context

> **Concept.** Goroutines, channels, select, sync, errgroup, worker pools, leak/race detection — and context for cancellation, deadlines and propagation, pervasive in every k8s API call and reconcile.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Why this matters

A Kubernetes controller *is* a concurrency engine. controller-runtime runs a work queue and spins up N reconcile loops in parallel (`MaxConcurrentReconciles`). Every client call — `Get`, `List`, `Update`, `Patch` — takes a `context.Context` as its first argument, and the manager cancels that context on `SIGTERM` so in-flight work drains instead of wedging a pod mid-shutdown. An exporter scrapes dozens of nodes/GPUs concurrently but must cap concurrency so it doesn't hammer the kubelet or a cloud billing API, and must respect Prometheus's scrape timeout. Get this wrong and you ship the classic production failures: a goroutine leak that grows RSS until OOMKill, a data race that corrupts a shared cache under load (invisible until it's a 2am incident), or a reconcile that ignores cancellation and blocks graceful shutdown past the pod's `terminationGracePeriodSeconds`, turning a rolling update into an outage.

Interview stakes: for a senior platform/Go role you will be asked to bound concurrency, explain what `go test -race` catches, show a goroutine leak and fix it, and explain why `ctx` threads through every call. These are table stakes, not trivia — they separate "wrote a script" from "operates fleets."

## From Python to Go

The single biggest mental shift: **Go has no GIL.** In CPython, one interpreter lock means threads don't run Python bytecode truly in parallel, so a sloppy `dict` mutation across threads usually *happens* to be safe. In Go, goroutines run on multiple OS threads on multiple cores simultaneously — two goroutines writing the same map or `int` is a genuine data race with undefined behavior (torn reads, corrupted maps that panic, stale values). The tool that catches this is the race detector: **`go test -race` / `go run -race`**. Run it in CI. It is your `-Werror` for concurrency.

- **Goroutine ≠ thread ≠ coroutine.** A goroutine is a green thread: ~2 KB initial stack, multiplexed by the runtime onto a small pool of OS threads. You launch millions cheaply with `go f()`. Unlike `asyncio`, there is **no `async`/`await` coloring** — any function can be called in a goroutine, and blocking I/O is fine (the runtime parks the goroutine and reuses the OS thread). Unlike Python threads, they give real parallelism.
- **Channels vs queues.** A channel is a typed, synchronized pipe — think `queue.Queue`, but a first-class language primitive with `select` for multiplexing and *closing* to signal "no more data." Unbuffered channels also synchronize (a send blocks until a receive), which `queue.Queue` doesn't do the same way.
- **The motto:** *"Don't communicate by sharing memory; share memory by communicating."* Prefer handing a value to exactly one owner over a channel to protecting shared state with locks. But it's guidance, not dogma — a counter or a cache is simpler and faster behind a `sync.Mutex` than behind a channel-guarded goroutine.

## Core notes

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

### errgroup — the production concurrency primitive

`golang.org/x/sync/errgroup` is what you actually use for fan-out. It's a `WaitGroup` that also collects the first error and cancels a derived context.

```go
g, ctx := errgroup.WithContext(ctx)   // ctx is cancelled when any g.Go returns non-nil error
g.SetLimit(N)                          // cap to N goroutines running at once (Go 1.20+)
for _, item := range items {
    item := item
    g.Go(func() error { return scrape(ctx, item) }) // blocks here once N are in flight
}
err := g.Wait()   // waits for all; returns the first non-nil error
```

`SetLimit(N)` is the bounded worker pool — no manual semaphore needed. On first error the derived `ctx` is cancelled, so cancellation-aware in-flight work unwinds instead of running to completion.

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

### The three classic goroutine leaks — and detection

1. **Blocked send, no receiver.** A goroutine does `ch <- v` but the reader returned early (e.g. on error). Send blocks forever. **Fix:** buffer the channel, or `select { case ch<-v: case <-ctx.Done(): }`.
2. **Blocked receive, no sender.** A goroutine ranges over a channel that is never closed / never sent to. **Fix:** ensure the sender closes, add a `ctx.Done()` arm.
3. **Forgotten cancel / no exit path.** A helper goroutine loops forever with no cancellation signal. **Fix:** pass `ctx`, `select` on `ctx.Done()`; always `defer cancel()` after `WithCancel`/`WithTimeout`.

Detection: `go test -race` finds data races (not leaks). For leaks, `go.uber.org/goleak` in tests (`defer goleak.VerifyNone(t)`) fails if goroutines outlive the test; in production, watch the `go_goroutines` Prometheus metric — a monotonic climb is a leak. `runtime.NumGoroutine()` and `/debug/pprof/goroutine` confirm it.

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
- **Never store `ctx` in a struct**; pass it per-call. A stored ctx outlives the request it belonged to and silently defeats cancellation. (`context.Value` is for request-scoped data like trace IDs — never for optional parameters.)

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

Why it's race-free: each goroutine writes only `results[i]` for its own `i` — disjoint slots, no shared mutation, so no mutex. If aggregation needed a shared map, guard it with `sync.Mutex`. `defer cancel()` guarantees the timer is released even on the happy path. `SetLimit(limit)` means only `limit` requests are ever open at once — the acceptance property an exporter needs.

## Practice

Build the collection stage of [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md) as a **bounded concurrent scrape**.

- `Collect(ctx context.Context, nodes []Node, limit int) ([]Sample, error)` scrapes per-node GPU metrics concurrently, capped at `limit` in-flight (use `errgroup` + `SetLimit`).
- Derive a scrape deadline from `ctx` (`WithTimeout`) so a slow node can't blow past Prometheus's scrape timeout; on timeout the scrape returns `context.DeadlineExceeded` and no goroutine keeps running.
- First hard error cancels all in-flight scrapes; aggregate successful samples into a slice (disjoint indices — no lock) or a mutex-guarded map.
- Wire it behind a `prometheus.Collector`'s `Collect` method later; for now, a `func` + tests.

**Acceptance:**
- `go test -race ./...` passes (no data race).
- No goroutine leak — assert with `defer goleak.VerifyNone(t)` in the test.
- Cancelling the passed `ctx` (or deadline expiry) returns promptly with `ctx.Err()` and leaves zero goroutines running.
- Concurrency never exceeds `limit` (assert with an atomic in-flight counter that peaks at ≤ `limit`).

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

## Resources

- **Learn Concurrent Programming with Go — James Cutajar (Manning, 2024).** *What-for:* modern, from-first-principles treatment of goroutines, channels, `select`, and the sync primitives with the race detector front and center. *How:* **deep-read** chs. on memory model, channels, and the classic problems; skim the OS-threads intro since you know it. *Why:* it's current (Go 1.21+) and builds the mental model of "no GIL → real races" that a Python-native engineer needs. (Alternative: *Concurrency in Go*, Cox-Buday — excellent on patterns/pipelines but older; pick one, not both.)
- **The Go Blog — "Go Concurrency Patterns: Context" — https://go.dev/blog/context.** *What-for:* the canonical rationale for `context.Context` — first arg, `Done()`, cancellation propagation. *How:* **deep-read** once; it's short. *Why:* it's the source these rules trace back to, and directly maps to how controller-runtime threads ctx through reconciles.
- **go-concurrency-exercises — https://github.com/loong/go-concurrency-exercises.** *What-for:* test-verified drills (bounded pool, rate limiting, graceful shutdown, deadlock fixes). *How:* **do**, don't read — solve 4–5 with `go test -race`. *Why:* muscle memory for the exact patterns in the practice task; the failing tests catch races you'd otherwise ship.
- **100 Go Mistakes — concurrency section — https://100go.co.** *What-for:* the specific traps: leaking goroutines, closing channels wrong, ctx propagation, `sync` misuse, loop-variable capture. *How:* **skim** the concurrency chapter now, reread when a `-race` failure baffles you. *Why:* every entry is a real production bug you want to have already seen once.
