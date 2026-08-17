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
sources: 16
---

# 01.4 · Concurrency and context

> **Concept.** Goroutines, channels, select, sync, errgroup, worker pools, leak/race detection — and context for cancellation, deadlines and propagation, pervasive in every k8s API call and reconcile.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 03 gave you the tool this lesson depends on: small, consumer-defined interfaces you can
fake without a mocking library. That is what makes everything here testable — you cannot write a
fast, deterministic test for a bounded concurrent fan-out without a fake standing in for the thing
being fanned out to. This lesson is the module's largest for a reason: a Kubernetes controller *is*
a concurrency engine, and an exporter scraping a GPU fleet is a concurrency problem before it is
anything else. What lesson 03 made fakeable, this lesson makes concurrent, bounded, and
cancellable — and that combination is what lesson 05's tests will assert against
(`goleak.VerifyNone`, an in-flight counter that never exceeds a limit).

## Why this matters

Every Kubernetes client call — `Get`, `List`, `Update`, `Patch` — takes a `context.Context` as its
first argument, and controller-runtime's manager cancels that context on `SIGTERM` so in-flight
work drains instead of wedging a pod mid-shutdown. controller-runtime gives the manager 30 seconds
to finish after cancellation (`defaultGracefulShutdownPeriod = 30 * time.Second`, in
`pkg/manager/internal.go`); anything still blocked when that expires is killed. An exporter scrapes
dozens of nodes concurrently but must cap concurrency so it does not hammer the kubelet or a cloud
billing API, and must finish inside Prometheus's scrape timeout — which defaults to **10 s** on a
**60 s** scrape interval (`config.DefaultGlobalConfig` in prometheus/prometheus).

Get this wrong and you ship the classic production failures. A goroutine leak grows RSS until the
pod OOMKills, with no error log anywhere along the way. A data race corrupts a shared cache under
load and stays invisible until 2am. A reconcile that ignores cancellation blocks graceful shutdown
past `terminationGracePeriodSeconds`, turning a rolling update into an outage.

Interview stakes: for a senior platform/Go role you will be asked to bound concurrency, explain
what `go test -race` catches (and what it does not), show a goroutine leak and fix it, and explain
why `ctx` threads through every call. These are table stakes, not trivia.

## What's new here (calibration)

Per the module README's calibration, you already understand concurrency as a concept from operating
distributed systems, so we skip re-teaching what a thread or a race condition *is* in the abstract.
We also skip the scheduler's work-stealing internals — the README explicitly lists runtime
internals as out of scope. The one place the scheduler shows up here is where it changes what
"blocking" costs you, because that is the difference between a goroutine being cheap and a thread
being expensive.

What is genuinely new: Go's primitives and their exact rules, the **happens-before edges the Go
memory model actually guarantees** (a short, closed list you can memorise), `context.Context`'s
internal structure — because "the context tree" is a real data structure with a real allocation
cost and a real leak mode — `errgroup` as the production bounded-fan-out pattern, and the failure
modes that are genuinely Go-shaped: the missing GIL turning sloppy shared state into undefined
behaviour, GOMAXPROCS/cgroup mismatches, and goroutine leaks as a slow-burn signature rather than
a crash.

## Core concepts

### The delta from Python: no GIL, and what that actually changes

In CPython, the global interpreter lock means only one thread executes bytecode at a time. A
`dict` mutation from two threads usually *happens* to be atomic because the interpreter does not
preempt in the middle of the store. You can be sloppy and get away with it.

Go has no GIL. Goroutines run on multiple OS threads on multiple cores genuinely simultaneously.
Two goroutines writing the same `map` or the same `int` with no synchronisation is a data race,
and the Go memory model is explicit about what that means:

> A data race is a write to a memory location happening concurrently with another read or write to
> that same location, unless all the accesses involved are atomic data accesses as provided by the
> `sync/atomic` package. *(The Go Memory Model, version of June 6, 2022)*

Go's guarantee for programs *without* races is **DRF-SC**: data-race-free programs execute as if
all goroutines were multiplexed onto a single processor — sequentially consistent. That is a strong,
useful guarantee, and it evaporates the moment you have one race.

For programs *with* races, Go is deliberately less anarchic than C++ but not safe. The memory model
sets out implementation restrictions worth knowing precisely, because they explain the failure
symptoms you will actually see:

| Situation | What the memory model permits |
|---|---|
| Any detected race | The implementation may report the race and halt. `-race` does exactly this. |
| Racy read of a word-sized or smaller location | Must observe *some* value actually written by a preceding or concurrent write. No "out of thin air" values. |
| Racy read of anything larger than a machine word | May be implemented as several independent word-sized reads *in an unspecified order*. |

That last row is the dangerous one and it is why Go's race symptoms look the way they do. A slice
header is `(pointer, len, cap)`. An interface value is `(type pointer, data pointer)`. A string is
`(pointer, len)`. Race on any of those and you can observe the pointer from one write paired with
the length from another — a combination that no single writer ever produced. The memory model says
this outright: such races "can in turn lead to arbitrary memory corruption." For maps specifically,
the runtime ships an explicit check and you get `fatal error: concurrent map writes`, which is
*not* a recoverable panic — `recover()` does not catch it, the process dies.

**The tool that catches all of this is `go test -race`.** Run it in CI. It is your `-Werror` for
concurrency.

### The exact happens-before edges Go guarantees

Almost every concurrency bug is "I assumed goroutine B would see what goroutine A wrote." The
memory model answers that question with a closed list of *synchronising* operations. Everything
else gives you nothing. Memorise this list; it is short.

```
 EDGES THE GO MEMORY MODEL GUARANTEES  (→ means "synchronized before")
 ═══════════════════════════════════════════════════════════════════════

 GOROUTINE CREATION           GOROUTINE EXIT
   a = 1                        go func(){ a = 1 }()
   go f()   ──────┐             print(a)       ← NO EDGE. The exit of a
                  │                              goroutine is synchronized
        f() sees a=1 ✓                           before nothing. ✗

 CHANNEL, BUFFERED OR NOT     CHANNEL, UNBUFFERED ONLY
   a = 1                        a = 1
   ch <- x  ──────┐             <-ch     ──────┐   (receive completes first)
                  ▼                            ▼
            v := <-ch                     ch <- x
            print(a) ✓                    print(a) ✓   ← only unbuffered

 CLOSE                        BUFFERED CHANNEL, CAPACITY C
   a = 1                        kth receive ────▶ completion of the
   close(ch) ─────┐                                (k+C)th send
                  ▼             ← this is the rule that makes a
            <-ch (zero,false)     buffered channel a counting semaphore
            print(a) ✓

 MUTEX                        ONCE                    ATOMICS
   n-th Unlock() ──▶            completion of f()      all sync/atomic ops
   m-th Lock() returns          in once.Do(f) ──▶      appear in ONE
   for any n < m                return of ANY          sequentially
                                once.Do(f)             consistent order
```

Three of these are worth dwelling on.

**Goroutine exit gives you nothing.** `go func(){ a = 1 }(); print(a)` is a race, and the memory
model notes that "an aggressive compiler might delete the entire `go` statement." If you need the
result, you need a channel, a `WaitGroup`, or a mutex — the goroutine merely *finishing* is not an
event other goroutines can observe.

**The unbuffered send/receive rule runs in the direction people do not expect.** For an unbuffered
channel, *the receive is synchronized before the completion of the send.* That is what makes an
unbuffered send a rendezvous: the sender cannot proceed until a receiver has arrived. Swap in a
buffered channel of capacity 1 and that program is no longer guaranteed — the memory model's own
example says it "might print the empty string, crash, or do something else."

**The capacity-C rule is the semaphore.** "The kth receive from a channel with capacity C is
synchronized before the completion of the (k+C)th send." Read it as: send = acquire, receive =
release, capacity = the limit. This is not a clever trick someone invented; it is a documented
guarantee in the memory model, complete with a worked example of limiting concurrency to three.

Everything else — a bare `int` flag, a `bool` "done" variable, a shared map — has **no edge at
all**. Not "usually visible." No guarantee that the write is ever observed, in any order, ever.

### Goroutines: what actually gets allocated

A goroutine is not an OS thread and not an `asyncio` coroutine. When you write `go f()`, the
runtime calls `newproc1`, which either reuses a dead `g` from the P-local free list or calls
`malg(stackMin)`. In current Go, `stackMin = 2048` — **2 KB** (`runtime/stack.go`). The `g` struct
itself is a couple hundred bytes of bookkeeping: stack bounds, program counter, scheduling status,
the `waitReason` you see in a stack dump.

The stack grows by *copying*, not by guard pages: the compiler inserts a stack-bound check in the
prologue of every non-`//go:nosplit` function, and on overflow the runtime allocates a stack twice
the size, copies the frames, and rewrites pointers into the old stack. The ceiling is
`maxstacksize`, set to **1 GB** on 64-bit platforms and 250 MB on 32-bit (`runtime/proc.go`);
exceed it and you get `goroutine stack exceeds 1000000000-byte limit` followed by
`fatal error: stack overflow`. That is the message infinite recursion produces.

The practical consequences for you:

- **A million goroutines is a real design option.** A million threads is not. 1e6 × 2 KB = 2 GB of
  stack if none of them grow, which is why fan-out-per-item is idiomatic in Go and insane in Java.
- **There is no `async`/`await` colouring.** Any function can be called in a goroutine. Blocking
  I/O is fine — you do not need a separate async-flavoured HTTP client the way Python needs
  `aiohttp` alongside `requests`.
- **Leaked goroutines are memory leaks with a metric.** Each one holds at least 2 KB of stack plus
  everything its frames reference. `go_goroutines` from `prometheus/client_golang`'s default Go
  collector is the number that climbs.

### Why blocking is cheap: the one bit of scheduler you need

This is the only place the GMP scheduler earns its keep in this lesson. Go multiplexes goroutines
(G) onto logical processors (P) onto OS threads (M). `GOMAXPROCS` is the number of Ps — the number
of goroutines that can be *running Go code* at once.

What matters is what happens when a G blocks, because there are three different answers:

```
 WHAT HAPPENS WHEN A GOROUTINE BLOCKS
 ═══════════════════════════════════════════════════════════════════════

 (1) CHANNEL / MUTEX / SLEEP  ── pure runtime blocking
     G parks (gopark). The M keeps its P and immediately runs the next
     runnable G. Cost ≈ a few hundred ns. The OS never hears about it.

     P ─┬─ M ── G1 [running]                 P ─┬─ M ── G2 [running]
        └─ runq: G2 G3     ── G1 parks ──▶      └─ runq: G3
                                                    G1 → sudog on hchan.recvq

 (2) NETWORK I/O ── read(2) on a socket
     The fd is in the netpoller (epoll on Linux). G parks, M keeps its P,
     runs other work. When epoll reports readiness, G goes back on a runq.
     This is why net/http scales without you writing async code.

 (3) BLOCKING SYSCALL ── open(2), a cgo call, a disk read
     The M is stuck in the kernel. sysmon (a dedicated M with no P, waking
     on a 20µs→10ms backoff) notices and RETAKES the P, handing it to
     another M so the other Gs keep running. The blocked M rejoins later.

 AND, INDEPENDENTLY: PREEMPTION
     sysmon preempts any G that has held its P for more than
     forcePreemptNS = 10ms (runtime/proc.go). Since Go 1.14 this is
     signal-based (asynchronous), so a tight loop with no function calls
     is preemptible too.
```

Case 3 is why `GOMAXPROCS` is a floor, not a cap, on OS threads: a program doing many blocking
syscalls can accumulate hundreds of Ms. Case 2 is why "just use blocking I/O" is correct advice in
Go and wrong advice in Python.

The Go 1.14 async-preemption change (release notes: "Goroutines are now asynchronously
preemptible... loops without function calls no longer potentially deadlock the scheduler") retired
a whole genre of folklore. Sprinkling `runtime.Gosched()` into hot loops was a real pre-1.14
workaround. On any modern toolchain it is cargo cult — do not port it forward.

### Channels: the data structure, not the metaphor

A channel is a pointer to a runtime `hchan` (`runtime/chan.go`):

```
 hchan                          make(chan Sample, 4)
 ┌─────────────────────────┐
 │ qcount   uint    = 2    │  items currently buffered
 │ dataqsiz uint    = 4    │  capacity (0 for unbuffered)
 │ buf      unsafe.Pointer─┼──▶ ┌────┬────┬────┬────┐  circular queue of
 │ elemsize uint16         │    │ s0 │ s1 │    │    │  dataqsiz elements
 │ closed   uint32  = 0    │    └────┴────┴────┴────┘
 │ sendx    uint    = 2    │      ▲         ▲
 │ recvx    uint    = 0    │    recvx     sendx
 │ recvq    waitq ─────────┼──▶ [sudog G7] → [sudog G9]   parked receivers
 │ sendq    waitq          │    (empty — buffer has room)
 │ lock     mutex          │  ← a real runtime mutex; every op takes it
 └─────────────────────────┘
```

A `sudog` is "goroutine G, parked, waiting on this channel, with element pointer E." When a sender
finds a waiting receiver in `recvq`, it copies the value **directly into the receiver's stack
slot** and marks it runnable — the buffer is bypassed entirely. That is the fast path, and it is
why an unbuffered channel is a rendezvous rather than a zero-length queue.

The rules that bite, and why:

| Operation | Behaviour | Why |
|---|---|---|
| Send on closed channel | **panic**: `send on closed channel` | `closed == 1` is checked under the lock; there is no correct value to deliver. |
| `close` on closed channel | **panic**: `close of closed channel` | Same check. |
| `close` on nil channel | **panic**: `close of nil channel` | |
| Receive from closed channel | Returns the zero value immediately, `ok == false` | And it does so forever — closed channels are permanently ready. |
| Send or receive on a nil channel | Blocks **forever** | `nil` has no `hchan` to park on, so the G parks with no way to be woken. Useful in `select`. |

**Only the sender closes, and only when it is the sole sender.** There is no "close if not closed"
primitive because there could not be one — the check and the close would race. If you have N
senders, close from a coordinator after a `WaitGroup` says all N are done.

Closing is a *broadcast*: `close(ch)` wakes every goroutine parked in `recvq` at once. That is
exactly why `context.Context` uses a closed channel as its cancellation signal rather than sending
a value — one close, unbounded fan-out, and every subsequent receive returns immediately.

### select: the multiplexer, and its randomisation

```go
select {
case v := <-in:      handle(v)
case out <- x:       // send, if a receiver is ready
case <-ctx.Done():   return ctx.Err()   // the cancellation arm
case <-time.After(d):return errTimeout
default:             // if present, makes the whole select non-blocking
}
```

Without `default`, `select` blocks until some case is ready. With `default`, it never blocks — it
is a poll.

The mechanism worth knowing: when more than one case is ready, `select` picks **uniformly at
random**, not in source order. `runtime/selectgo` builds a `pollorder` array and shuffles it with
`cheaprandn` before evaluating (`runtime/select.go`). This is a deliberate fairness property. It
also means you cannot express priority with ordering — if you need "drain `work` unless `ctx` is
done," you need a nested select with a `default`, not a case order.

The `<-ctx.Done()` arm is the single most important idiom in this lesson. It is how a blocking
operation becomes cancellable, and its absence is how goroutines leak.

Two useful tricks:

- **`nil` channel to disable a case.** A `select` case on a nil channel blocks forever, so it is
  effectively removed. Set `ch = nil` after you have drained it to stop selecting on it, instead of
  restructuring the loop.
- **`time.After` in a loop allocates.** Each call allocates a new timer that lives until it fires —
  in a hot loop with a long duration, that is a real leak of timers. Use a `time.Timer` you
  `Reset`, or derive a context.

### sync: when a lock is the right tool

The motto "don't communicate by sharing memory; share memory by communicating" is guidance, not
dogma. Guarding a counter, a cache, or one struct field with a `sync.Mutex` is simpler and
measurably faster than owning it behind a goroutine and a channel.

```go
var mu sync.Mutex            // full mutual exclusion
mu.Lock(); count++; mu.Unlock()

var rw sync.RWMutex          // many readers OR one writer — read-heavy caches
rw.RLock(); v := m[k]; rw.RUnlock()

var once sync.Once           // run exactly once (lazy init, singletons)
once.Do(func() { initExpensive() })

var wg sync.WaitGroup        // wait for a set of goroutines
for _, t := range tasks {
    wg.Add(1)
    go func() { defer wg.Done(); process(t) }()
}
wg.Wait()
```

`sync.Mutex` is not a simple futex. It runs in two modes (`internal/sync/mutex.go`). In **normal
mode**, an arriving goroutine spins briefly (`active_spin = 4` iterations of `active_spin_cnt = 30`
`PAUSE` instructions, `runtime/lock_spinbit.go`) and may barge ahead of goroutines already queued — good throughput, bad tail latency. If a waiter has been
queued for longer than `starvationThresholdNs = 1e6` — **1 ms** — the mutex flips to **starvation
mode**, where unlock hands ownership directly to the head of the queue and arrivals do not spin.
This is a deliberate throughput-vs-tail-latency trade, and it is why mutex contention in Go shows
up as a bimodal latency histogram rather than a smooth one.

`sync/atomic` covers single-word counters and flags without a lock. Prefer the typed wrappers
(`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`) over the free functions — they cannot be
copied by accident and they cannot be mixed with non-atomic access to the same variable.

Go 1.25 added **`WaitGroup.Go`**, which collapses the `Add(1)` / `defer Done()` pair:

```go
var wg sync.WaitGroup
for _, t := range tasks {
    wg.Go(func() { process(t) })   // Go 1.25+
}
wg.Wait()
```

The classic `WaitGroup` bug it removes: calling `wg.Add(1)` *inside* the goroutine, so `Wait()` can
return before the goroutine has even incremented the counter.

### The race detector: what it actually instruments

`-race` is not a linter and not a static analysis. It compiles your program against
**ThreadSanitizer** (TSan), the same runtime used by C/C++ and Rust, and instruments every
non-atomic memory access with a call into the TSan runtime.

The data structure is **shadow memory**. For each aligned 8-byte word of application memory, TSan
keeps N shadow words (N is 4 in the standard configuration). Each 64-bit shadow word records one
access:

```
 APPLICATION MEMORY                    SHADOW MEMORY
 ─────────────────                     ─────────────
 addr 0x10a8  ┌──────────────┐         ┌───────────────────────────────────┐
              │  8 bytes of  │ ──────▶ │ shadow word 0: TID=3  clock=41  W │
              │  your struct │         │ shadow word 1: TID=3  clock=44  R │
              └──────────────┘         │ shadow word 2: TID=7  clock=12  W │◀── conflict?
                                       │ shadow word 3: (empty)            │
                                       └───────────────────────────────────┘
   each shadow word packs:  thread id (16b) │ clock (42b) │ W bit │ size (2b) │ offset (3b)

 ON EVERY NON-ATOMIC LOAD/STORE:
   1. compute the shadow slots for this address
   2. compare the new access against each stored shadow word:
        same thread?                     → no race
        ordered by happens-before?       → no race  (compare clocks)
        both reads?                      → no race
        otherwise                        → RACE. print report, exit 66.
   3. store the new access into a free slot; if all 4 are full,
      EVICT ONE AT RANDOM.
```

Three consequences fall directly out of that design:

1. **It is a dynamic detector.** It only sees accesses that actually execute. A race in a code path
   your tests never enter is invisible. Run `-race` in CI *and* under a realistic load, not just on
   unit tests.
2. **It can miss.** Random eviction of shadow words means a race whose two accesses are separated
   by many other accesses to the same word can be dropped. There are no false positives — a report
   is always a real race — but absence of a report is not proof.
3. **It is expensive, in known amounts.** The Go race detector documentation states memory usage
   may increase by **5–10×** and execution time by **2–20×**. It also allocates an extra **8 bytes
   per `defer` and `recover`**, not reclaimed until the goroutine exits — so a long-running
   goroutine that defers in a loop grows without bound under `-race` and *only* under `-race`.
   That is why you do not run `-race` binaries in production; you run them in CI and in load tests.

Tunables live in the `GORACE` environment variable:

| `GORACE` option | Default | What it does |
|---|---|---|
| `halt_on_error` | `0` | `1` exits on the first race instead of continuing. Use in CI for fast, clean failures. |
| `exitcode` | `66` | Process exit status after a detected race. |
| `history_size` | `1` | Per-goroutine access history is `32K * 2^history_size` entries. Raise it when reports say `failed to restore the stack`. |
| `log_path` | `stderr` | Writes to `log_path.pid` instead. |
| `strip_path_prefix` | `""` | Trims a prefix from reported paths. |

`-race` also sets the `race` build tag, so you can exclude a test with `//go:build !race`.

**Reading a report.** Here is the shape (representative, following the format in the Go race
detector documentation), annotated:

```
WARNING: DATA RACE                          ← always this header; exit code will be 66
Write at 0x00c0000b4020 by goroutine 12:    ← ① the SECOND access, the one that tripped it
  exporter.(*Collector).record()
      /src/internal/exporter/collect.go:78 +0x64
  exporter.Collect.func1()
      /src/internal/exporter/collect.go:52 +0x9c

Previous read at 0x00c0000b4020 by goroutine 9:   ← ② the FIRST access, from history
  exporter.(*Collector).total()
      /src/internal/exporter/collect.go:91 +0x3a

Goroutine 12 (running) created at:          ← ③ where each goroutine was STARTED
  exporter.Collect()
      /src/internal/exporter/collect.go:51 +0x188
Goroutine 9 (finished) created at:
  exporter.Collect()
      /src/internal/exporter/collect.go:44 +0xd0
```

Read it in this order:

- **The address** (`0x00c0000b4020`) is the same in both stanzas — that is the contended location.
  Both stanzas must name the same address or you are looking at two reports.
- **① and ② are a pair.** One is a write; the other is a read or a write. Two reads never race.
  The word *Previous* marks the access recovered from the shadow/history buffer.
- **③ tells you the goroutines' origins**, which is usually the actual fix site: both were created
  at `collect.go:51` and `collect.go:44`, i.e. inside the same fan-out, so the fix is to give each
  worker its own slot rather than sharing `Collector.total`.
- `(finished)` on goroutine 9 is normal. TSan remembers accesses from goroutines that have exited;
  the race was real when it happened.

If a stanza says `failed to restore the stack` instead of frames, raise `history_size`.

### context: the tree, the allocation, and the leak

`context.Context` is a four-method interface:

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

Everything else in the package builds *derived* contexts, and the word "derived" is literal: each
constructor allocates a new struct that embeds its parent. The result is a tree, and cancellation
is a traversal of that tree.

**What `WithCancel` allocates.** `context.WithCancel(parent)` calls `withCancel`, which allocates
a `cancelCtx` (`src/context/context.go`):

```go
type cancelCtx struct {
    Context                            // the parent, embedded

    mu       sync.Mutex                // protects the fields below
    done     atomic.Value              // of chan struct{}, created LAZILY
    children map[canceler]struct{}     // set to nil by the first cancel
    err      atomic.Value              // set non-nil by the first cancel
    cause    error                     // set non-nil by the first cancel
}
```

Note `done` is **lazily created**. `Done()` does an atomic load; only if that is nil does it take
the mutex, double-check, and `make(chan struct{})`. A context nobody ever calls `Done()` on never
allocates a channel at all. And `cancel()` on a context whose channel was never created stores the
package-level pre-closed `closedchan` instead of allocating one to immediately close.

**How the parent learns about the child.** `withCancel` calls `propagateCancel(parent, c)`, which
does four things in order:

1. If `parent.Done() == nil` (a `Background()` or `TODO()` root), the parent can never be
   cancelled — return immediately, wire nothing up. **This is why a chain rooted at `Background()`
   with no cancellable ancestor costs nothing.**
2. If the parent is *already* cancelled, cancel the child immediately and return.
3. Walk up with `parentCancelCtx(parent)` looking for the nearest `*cancelCtx` ancestor. If found,
   insert the child into that ancestor's `children` map — allocating the map on first use. **This
   is the edge in the tree.**
4. Only if no `*cancelCtx` ancestor exists — a custom `Context` implementation with its own `Done`
   channel — does the package fall back to spawning a **goroutine** that selects on
   `parent.Done()` and `child.Done()`. (Since Go 1.21 there is an intermediate case: if the parent
   implements `AfterFunc(func()) func() bool`, that is used instead of a goroutine.)

That step-4 fallback is the reason "write your own `Context` type" is bad advice: it silently turns
every derivation into a goroutine.

**What `cancel()` does.** One pass, under the lock:

```go
func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) {
    c.mu.Lock()
    if c.err.Load() != nil { c.mu.Unlock(); return }   // idempotent: first cancel wins
    c.err.Store(err)
    c.cause = cause
    d, _ := c.done.Load().(chan struct{})
    if d == nil { c.done.Store(closedchan) } else { close(d) }   // ← the broadcast
    for child := range c.children {
        child.cancel(false, err, cause)                          // ← depth-first recursion
    }
    c.children = nil
    c.mu.Unlock()
    if removeFromParent { removeChild(c.Context, c) }            // ← unlink from parent
}
```

Read the two important lines. `close(d)` is the broadcast that wakes every goroutine selecting on
`ctx.Done()`. The `for child := range c.children` loop recurses **synchronously, depth-first,
holding the parent's lock** — so by the time `cancel()` returns, every `Done()` channel in the
subtree is already closed. There is no propagation delay and no goroutine hop.

Note the argument asymmetry: the recursive calls pass `removeFromParent = false`, because
`c.children = nil` clears the whole set at once — unlinking each child individually would be O(n)
lock acquisitions for nothing.

```
 A CONTEXT TREE DURING A SIGTERM SHUTDOWN
 ═══════════════════════════════════════════════════════════════════════

  signals.SetupSignalHandler()  ── SIGTERM ──▶ cancel()
        │
        ▼
  rootCtx  (cancelCtx)  done: chan ──close──▶ ALL RECEIVERS WAKE
        │  children: {mgrCtx}
        ▼
  mgrCtx   (cancelCtx)                   ── manager.Start(ctx)
        │  children: {rec-1, rec-2, cacheCtx}
        ├──────────────┬──────────────────────┐
        ▼              ▼                      ▼
   rec-1 (timerCtx) rec-2 (timerCtx)     cacheCtx (cancelCtx)
   deadline +30s    deadline +30s        children: {watch-pods, watch-nodes}
   timer: *Timer    timer: *Timer             │
        │                │                    ├──▶ watch-pods  → http req ctx
        ▼                ▼                    └──▶ watch-nodes → http req ctx
   r.Get(ctx,...)   r.Update(ctx,...)
   (transport selects on ctx.Done())

  TIMELINE
  t+0.000s  SIGTERM delivered; handler goroutine calls cancel()
  t+0.000s  rootCtx.cancel(): close(done); recurse into mgrCtx
  t+0.000s  mgrCtx.cancel(): close(done); recurse into rec-1, rec-2, cacheCtx
  t+0.000s  ... entire subtree's Done() channels are closed. Depth-first,
            synchronous, under locks. No goroutines involved.
  t+0.000s+ each timerCtx.cancel() ALSO calls timer.Stop() — the pending
            30s timers are released now, not in 30s.
  t+~µs     every blocked http.RoundTrip returns context.Canceled
  t+≤30s    manager waits gracefulShutdownTimeout (30s default) then returns
```

**`WithTimeout` and `WithDeadline`.** Both funnel into `WithDeadlineCause`, which allocates a
`timerCtx` — a `cancelCtx` plus `timer *time.Timer` and `deadline time.Time`. Two behaviours are
worth knowing:

- **A child deadline can only shorten, never extend.** `WithDeadlineCause` starts with
  `if cur, ok := parent.Deadline(); ok && cur.Before(d) { return WithCancel(parent) }`. If the
  parent already expires sooner, you get a plain cancellable context with the *parent's* deadline.
  You cannot buy yourself more time by deriving a longer timeout from a shorter parent.
- **The timer is a real `time.AfterFunc`.** It lives in the runtime timer heap until it fires or is
  stopped, and it holds a reference to the `timerCtx`, which holds a reference to everything the
  context closes over.

**Why a missing `cancel()` leaks — precisely.** `go vet`'s `lostcancel` check flags this, and here
is what it is protecting you from:

```go
func scrapeNode(ctx context.Context, n Node) error {
    ctx, _ = context.WithTimeout(ctx, 5*time.Second)   // ← cancel discarded
    return doScrape(ctx, n)
}
```

If `doScrape` returns in 3 ms, two things are still true for the next **4.997 seconds**:

1. The `timerCtx` remains in the parent's `children` map. If the parent is a long-lived context
   (the manager's, say) and you call `scrapeNode` 200 times per scrape, that map grows to 200
   entries per scrape and never shrinks until each timer fires. At a 15 s scrape interval you have
   a steadily-growing map of dead contexts.
2. The `time.Timer` sits in the runtime timer heap holding the `timerCtx` alive, which holds the
   parent chain and every captured variable alive. The GC cannot collect any of it.

`defer cancel()` fixes both: `timerCtx.cancel` calls `removeChild` (unlink from the parent map)
*and* `c.timer.Stop()` (drop the timer). It is not "good hygiene" — it is the deallocation path.

**`WithValue` is a linked list, not a map.** `WithValue` allocates a `valueCtx{Context, key, val}`
— one key, one value, one parent pointer. `Value(k)` walks the chain comparing keys until it finds
one or reaches the root. That is **O(depth)** on every lookup, with an interface comparison per
level.

That cost is fine for a handful of well-known keys threaded through deep call stacks: a trace ID,
an auth principal, a request ID. It is the wrong tool as a general parameter bag, for three
reasons that compound: the walk gets longer with every layer; `Value` returns `any` so you lose
type safety and get a runtime type assertion instead of a compile error; and the dependency
disappears from the function signature, so you cannot tell what a function needs by reading it.
The package documentation is blunt about it: "Use context Values only for request-scoped data that
transits processes and APIs, not for passing optional parameters to functions."

Keys must be comparable — `WithValue` panics otherwise — and should be an unexported named type
(`type ctxKey struct{}`) so two packages cannot collide on `"user"`.

**Modern additions worth knowing** (they show up in current controller code):

| API | Since | What it is for |
|---|---|---|
| `context.WithCancelCause(parent)` | Go 1.20 | Cancel with an explanatory error. `ctx.Err()` still returns `Canceled`; `context.Cause(ctx)` returns your error. |
| `context.Cause(ctx)` | Go 1.20 | Retrieve that error. Returns `ctx.Err()` if no cause was set. |
| `context.WithoutCancel(parent)` | Go 1.21 | Keep the values, drop the cancellation. For cleanup/audit work that must complete *after* the request is cancelled. |
| `context.AfterFunc(ctx, f)` | Go 1.21 | Run `f` in its own goroutine when `ctx` is done; returns a `stop` func. Replaces the hand-rolled `go func(){ <-ctx.Done(); cleanup() }()`. |
| `context.WithDeadlineCause` / `WithTimeoutCause` | Go 1.21 | Same, but the cause is attached to the deadline firing. |

`WithCancelCause` matters here because **`errgroup` uses it**, which is how `context.Cause(ctx)`
inside a worker tells you *which sibling's error* killed the group.

### Timeout budgets: making the arithmetic add up

Deadlines compose by taking the minimum, which means a call chain has a budget and every hop
spends some of it. Lay it out explicitly — this is the arithmetic interviewers ask you to do out
loud.

Take the exporter. Prometheus's default `scrape_timeout` is 10 s and it will abandon the request
at exactly that point, marking the target `up=0`. Your budget is therefore **10 s minus a safety
margin**, and it is spent like this:

```
 SCRAPE TIMEOUT BUDGET  (Prometheus scrape_timeout = 10s default)
 ══════════════════════════════════════════════════════════════════════

  0s                          8s          9s        10s
  ├────────────────────────────┼───────────┼──────────┤
  │      collector budget      │  render   │  slack   │  ← Prometheus gives up
  │        ctx deadline 8s     │  ~200ms   │          │
  └────────────────────────────┘

  Inside the 8s collector budget, per node:
    per-node ctx = WithTimeout(collectorCtx, 2s)
      ├─ DNS + TCP + TLS handshake      ~150ms  (cold; ~0 with keepalive)
      ├─ HTTP request → kubelet          ~50ms  p50   ~800ms p99
      └─ JSON decode                     ~10ms
    retry budget: 2 attempts max, backoff below

  Why 2s per node and not 8s: with limit=8 workers and 200 nodes there are
  200/8 = 25 waves. If one node can burn the whole 8s, one bad node stalls
  a worker for the entire scrape and you lose 1/8 of your throughput.
  A 2s per-node cap means the worst case is 25 waves x 2s = 50s IF every
  node is slow — so the 8s parent deadline is what actually protects you,
  and the per-node cap is what stops ONE node from eating it.
```

The general rule: **the parent deadline is the guarantee; per-call deadlines are the isolation.**
Set both. And always leave the child deadline strictly less than the parent, because a child equal
to the parent gives the caller no room to do anything with the error.

**Backoff with jitter, computed out.** Retrying a failed scrape without jitter synchronises every
client into a thundering herd. Here is `base × 2^n` with full jitter (`sleep = rand(0, base×2^n)`),
`base = 100 ms`, capped at 3 s:

| Attempt | Nominal `base×2^n` | Capped | Full-jitter range | Worst-case cumulative |
|---|---|---|---|---|
| 1 (first retry) | 100 ms | 100 ms | 0–100 ms | 100 ms |
| 2 | 200 ms | 200 ms | 0–200 ms | 300 ms |
| 3 | 400 ms | 400 ms | 0–400 ms | 700 ms |
| 4 | 800 ms | 800 ms | 0–800 ms | 1.5 s |
| 5 | 1.6 s | 1.6 s | 0–1.6 s | 3.1 s |
| 6 | 3.2 s | **3.0 s** | 0–3.0 s | 6.1 s |

With a 2 s per-node budget, you get **at most 3 retries** before the per-node context expires
(0.7 s of worst-case backoff plus three ~0.4 s attempts). That is the number to configure, and it
falls out of the arithmetic rather than being guessed.

For comparison, this is exactly the shape controller-runtime uses for requeues.
`workqueue.DefaultTypedControllerRateLimiter` (client-go, `util/workqueue/default_rate_limiters.go`)
is the max of two limiters: a **per-item exponential** `5 ms × 2^failures` capped at **1000 s**, and
an **overall token bucket at 10 qps with burst 100**. So a permanently-failing object backs off
5 ms, 10 ms, 20 ms … up to ~16.7 minutes, while the bucket keeps the whole controller from
exceeding 10 requeues per second no matter how many objects are failing.

### Worker-pool sizing: choosing the number

For an I/O-bound scrape, the right concurrency is not "number of CPUs." Use Little's Law:
`concurrency = arrival_rate × latency`, rearranged to `concurrency = target_throughput × latency`.

Worked, for the exporter:

```
  GIVEN
    N          = 200 nodes to scrape
    L_p50      = 50ms  per-node latency   (kubelet /metrics/resource, warm)
    L_p99      = 800ms
    T_budget   = 8s    collector deadline

  REQUIRED THROUGHPUT
    200 nodes / 8s = 25 nodes/s

  CONCURRENCY BY LITTLE'S LAW
    at p50:  25/s x 0.05s = 1.25  →  2 workers is enough on a good day
    at p99:  25/s x 0.80s = 20    →  20 workers to survive a bad day

  BUT ALSO BOUND BY THE TARGET
    kubelet is not a load balancer. 20 concurrent scrapes against ONE
    kubelet is abusive; 20 spread across 200 kubelets is 0.1 each.
    → the bound that matters is per-target, and it is 1.

  AND BY THE CLIENT
    http.Transport default MaxIdleConnsPerHost = 2. With limit=20 and
    200 distinct hosts you get 20 in-flight, ~2 idle conns retained per
    host, so most scrapes pay a fresh handshake. Raise
    MaxIdleConnsPerHost to >= limit if the target set is small; leave it
    if the target set is large (you cannot cache 200 hosts' conns cheaply).

  CHOICE: limit = 8
    Covers p50 comfortably (needs 2), degrades gracefully at p99
    (8 workers x 1/0.8s = 10 nodes/s → 20s, exceeds budget → partial
    results + DeadlineExceeded, which is the CORRECT failure: report what
    you have, mark the scrape failed, let the next one retry).
```

The point is not the number 8. The point is that you can defend it: it comes from a throughput
requirement, a latency distribution, and a constraint from the thing being called. "I set it to
`runtime.NumCPU()`" is the wrong answer for I/O-bound work and a tell in an interview.

### errgroup: the production fan-out primitive

`golang.org/x/sync/errgroup` is what you actually use. It is small enough to read in full, and
reading it removes all the magic:

```go
type Group struct {
    cancel  func(error)     // the CancelCauseFunc from WithContext
    wg      sync.WaitGroup
    sem     chan token      // nil unless SetLimit was called
    errOnce sync.Once
    err     error
}

func WithContext(ctx context.Context) (*Group, context.Context) {
    ctx, cancel := context.WithCancelCause(ctx)
    return &Group{cancel: cancel}, ctx
}

func (g *Group) Go(f func() error) {
    if g.sem != nil { g.sem <- token{} }     // ← BLOCKS HERE when at limit
    g.wg.Add(1)
    go func() {
        defer g.done()                        // releases sem, then wg.Done()
        if err := f(); err != nil {
            g.errOnce.Do(func() {             // ← only the FIRST error is kept
                g.err = err
                if g.cancel != nil { g.cancel(g.err) }   // ← cancels the derived ctx
            })
        }
    }()
}

func (g *Group) SetLimit(n int) {
    if n < 0 { g.sem = nil; return }
    if active := len(g.sem); active != 0 { panic(...) }
    g.sem = make(chan token, n)               // ← the capacity-C semaphore rule
}
```

Four facts that follow directly from the source:

- **`SetLimit(n)` blocks in `Go`, before the goroutine is created.** That is the difference from a
  hand-rolled semaphore acquired *inside* the goroutine: with `errgroup`, at most `n` goroutines
  ever exist. With the naive semaphore, all `len(items)` goroutines exist, each holding 2 KB, most
  of them parked on the semaphore.
- **`errOnce` means only the first error survives.** Errors 2..N are dropped on the floor. If you
  need all of them, collect into a mutex-guarded slice yourself, or use `errors.Join`.
- **`Wait` also cancels.** `g.Wait()` calls `g.cancel(g.err)` after `wg.Wait()` returns — so the
  derived context is cancelled on the *success* path too. That is why you must not use the derived
  `ctx` for anything after `Wait` returns.
- **`SetLimit` panics if called while goroutines are active**, and `SetLimit(0)` deadlocks the next
  `Go` forever. Set it once, before the loop.

**`errgroup` does not propagate panics — and that is a decision, not an oversight.** This is worth
being precise about because the behaviour changed and changed back. Panic propagation was added in
January 2025 (CL 644575, shipped in x/sync v0.14.0) and **reverted in June 2025** (CL 682935,
shipped in v0.16.0) with the commit message "this caused more problems than it solved." The current
source carries a "tsunami stone" comment explaining why, citing golang/go issues
[#53757](https://github.com/golang/go/issues/53757),
[#74275](https://github.com/golang/go/issues/74275),
[#74304](https://github.com/golang/go/issues/74304), and
[#74306](https://github.com/golang/go/issues/74306): propagating a panic delays it arbitrarily,
turns the panic stack into a mere value and hides it from crash monitoring, and risks deadlocking
if the panic leaves the program unable to reach `Wait`. **A panic in a `g.Go` closure crashes the
process, with the original stack.** If you need per-worker panic recovery, write the
`defer recover()` yourself, inside the closure.

`SetLimit` and `TryGo` were added in May 2022 and first tagged in **x/sync v0.1.0**; anything older
than that needs a hand-rolled semaphore.

**What cancellation does and does not buy you.** `errgroup.WithContext`'s cancellation only aborts
operations that actually *check* `ctx.Done()` — directly, or by passing `ctx` into something that
does (`http.NewRequestWithContext`, a database driver, gRPC, the k8s client). A pure CPU-bound loop
with no `select` and no `ctx.Err()` check runs to completion no matter how many siblings have
failed. Cancellation is cooperative; there is no preemption of user code.

### The bounded worker pool without errgroup

When you need it by hand, a buffered channel *is* the semaphore — and the memory model's
capacity-C rule is the formal justification:

```go
sem := make(chan struct{}, limit)
var wg sync.WaitGroup
for _, item := range items {
    select {
    case sem <- struct{}{}:              // acquire BEFORE launching
    case <-ctx.Done():
        wg.Wait()
        return ctx.Err()
    }
    wg.Add(1)
    go func(it Item) {
        defer wg.Done()
        defer func() { <-sem }()         // release
        process(ctx, it)
    }(item)
}
wg.Wait()
```

The load-bearing detail is that the acquire is **outside** the goroutine. Move `sem <- struct{}{}`
inside the closure and you bound *execution* to `limit` but not *allocation*: all `len(items)`
goroutines get created and park, each holding its 2 KB stack. At 200 items that is invisible; at
200,000 it is 400 MB of stacks doing nothing. The `select` on `ctx.Done()` around the acquire is
what makes the launch loop itself cancellable.

The other shape — a fixed pool of `limit` long-lived workers ranging over a jobs channel — is
better when the work items are cheap and numerous, because you pay for `limit` goroutines total
instead of one per item:

```go
jobs := make(chan Item)
var wg sync.WaitGroup
for range limit {
    wg.Add(1)
    go func() { defer wg.Done(); for it := range jobs { process(ctx, it) } }()
}
for _, it := range items {
    select {
    case jobs <- it:
    case <-ctx.Done():
    }
}
close(jobs)   // ← the sole sender closes; workers' range loops end
wg.Wait()
```

### The three goroutine leaks, and how each is detected

A leak is a goroutine with no path to exit. Every one you start needs a guaranteed exit, and there
are exactly three ways to fail at that:

**1 · Blocked send, no receiver.** The canonical one, and it is in the Go 1.26 release notes as the
motivating example for the new leak profile:

```go
ch := make(chan result)              // unbuffered
for _, w := range work {
    go func() { ch <- process(w) }() // blocks until someone receives
}
for range len(work) {
    r := <-ch
    if r.err != nil { return r.err } // ← EARLY RETURN. Remaining senders block forever.
}
```

Fix: buffer the channel to `len(work)` so every send completes regardless, or `select` the send
against `ctx.Done()`.

**2 · Blocked receive, no sender.** A goroutine ranges over a channel that is never closed, or
waits on a channel whose producer already returned an error and never sent. Fix: make sure exactly
one owner closes, and add a `ctx.Done()` arm.

**3 · No exit path at all.** A helper loop with no cancellation signal — `for { poll(); time.Sleep(d) }`
launched at startup and never told to stop. Fix: pass `ctx`, `select` on `ctx.Done()`, and always
`defer cancel()`.

**Detection, in order of when you find out:**

| Tool | Finds | Does not find |
|---|---|---|
| `go vet` (`lostcancel`) | A `cancel` assigned to `_` or unused | Leaks generally |
| `go test -race` | **Data races only** | Goroutine leaks. Passing `-race` proves nothing about leaks. |
| `go.uber.org/goleak` | Goroutines outliving a test | Leaks in code the test does not run |
| `go_goroutines` metric | Production leaks, as a slope | Which line leaked |
| `/debug/pprof/goroutine?debug=2` | Full stacks of every goroutine, grouped | — |
| `/debug/pprof/goroutineleak` (Go 1.26 experiment; GA in 1.27) | Goroutines that provably can never wake | Leaks reachable from globals |

**`goleak` mechanics.** `goleak.VerifyNone(t)` snapshots `runtime.Stack` for all goroutines, drops
the current one and a built-in filter list (the `testing` package's own goroutines, syscall stacks,
`os/signal`, the DNS resolver), and fails if anything remains. Crucially it **retries**: up to
`_defaultRetries = 20` attempts, sleeping `1µs << i` capped at `maxSleep = 100ms`. Summing that
gives a worst case of about **431 ms** (131 ms across the first 17 doublings, then three 100 ms
sleeps) before it declares a leak — enough for a goroutine that is genuinely shutting down to
finish, short enough not to slow a suite noticeably. Two gotchas: `VerifyNone` is **incompatible
with `t.Parallel()`** (it cannot attribute goroutines to tests), and by default it does not run on
a failing test unless you pass `goleak.RunOnFailure()`.

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)   // one check for the whole package
}
// or, per test:
func TestCollect(t *testing.T) {
    defer goleak.VerifyNone(t)
    // ...
}
```

**The runtime's own leak profile (new, and the direction this is going).** Go 1.26 (February 2026,
the current stable release at the time of writing) shipped an experimental `goroutineleak` profile
behind `GOEXPERIMENT=goroutineleakprofile`, and Go 1.27 — in release-candidate as this is written,
due August 2026 — drops the experiment flag and makes it generally available, exposed as
`runtime/pprof` profile `goroutineleak` and HTTP endpoint `/debug/pprof/goroutineleak`. Check
`go version` before relying on either. The mechanism is elegant: **the garbage collector does
the detection.** If goroutine G is blocked on a concurrency primitive P (channel, `sync.Mutex`,
`sync.Cond`), and P is unreachable from any runnable goroutine — or from any goroutine those could
unblock — then nothing can ever touch P, so G can never wake. It is a leak, provably.

The known blind spot is stated in the release notes: leaks on primitives reachable through global
variables, or through locals of runnable goroutines, are missed, because reachability is the whole
test. The work was contributed by Vlad Saioc at Uber and builds on the same research as Uber's
LeakProf. It is designed to add no runtime overhead unless in use.

**The production signature.** None of this looks like a crash. `go_goroutines` climbs monotonically
over hours or days. RSS follows, because each goroutine holds its stack and whatever its frames
reference. GC pause time creeps up, because pause cost scales with live heap. Then the pod
OOMKills — often days after the deploy that introduced it, which is what makes the postmortem
hard. Alert on the *slope* of `go_goroutines`, not a threshold:
`deriv(go_goroutines[1h]) > 0` sustained for six hours catches it before the memory does.

### GOMAXPROCS and containers

`GOMAXPROCS` is the number of Ps — how many goroutines can run Go code simultaneously. Before
Go 1.25, it defaulted to `runtime.NumCPU()`, the number of logical CPUs *the host* has. Inside a
pod with `resources.limits.cpu: "2"` on a 64-core node, that means `GOMAXPROCS=64`: the runtime
happily schedules 64 goroutines in parallel, the cgroup CPU bandwidth controller throttles the
process at its 2-CPU quota, and you get elevated latency and `container_cpu_cfs_throttled_seconds_total`
climbing despite modest real demand. `uber-go/automaxprocs` existed precisely to read the cgroup
limit and call `runtime.GOMAXPROCS()` at init.

**Go 1.25 (August 2025) fixed this in the runtime.** From the release notes: on Linux the runtime
considers the CPU bandwidth limit of the cgroup containing the process, and if that is lower than
the logical CPU count, `GOMAXPROCS` defaults to the lower limit. On all OSes it now *periodically
re-reads* the limit and updates `GOMAXPROCS` if it changes. Two details matter operationally:

- It reads the **CPU limit**, not the CPU request. A pod with `requests.cpu: 500m` and no limit
  still gets the full node count.
- Both behaviours are disabled if you set the `GOMAXPROCS` environment variable or call
  `runtime.GOMAXPROCS()` yourself. They can also be turned off individually with
  `GODEBUG=containermaxprocs=0` and `GODEBUG=updatemaxprocs=0`. Go 1.25 added
  `runtime.SetDefaultGOMAXPROCS()` to opt back in.

So: on Go 1.25+ this is handled. On **Go 1.24 or earlier**, or a pinned base image, it is still
live and `automaxprocs` is still the fix. Check `go version` in the image before assuming.

### Where this lands in a controller

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

`MaxConcurrentReconciles` **defaults to 1** (`pkg/controller/controller.go`) — a single worker
goroutine pulling from the workqueue. That default is why "my reconciler is slow across many
objects" is usually a concurrency setting, not a code problem. It is your `SetLimit` for the
reconcile loop: the same bounded-fan-out lever, spelled differently.

The `ctx` you are handed descends from `ctrl.SetupSignalHandler()`, which registers `SIGTERM` and
`SIGINT`, cancels on the first signal, and calls `os.Exit(1)` on the second. So the tree in the
diagram above is literal: one signal, one `cancel()`, every derived context in the process closed
synchronously, every in-flight API call unwound. Your only job is to not ignore it.

## Perspectives

**Developer view.** `errgroup.WithContext` + `SetLimit` is the default shape for any fan-out. It is
not a special-purpose tool — it *is* `MaxConcurrentReconciles` under a different name, and the same
bounded-fan-out pattern recurs at every layer: bounded reconciles, bounded scrapes, bounded
outbound API calls. Learn it once, apply it everywhere. The corollary is that `go` with no
accompanying `errgroup`, `WaitGroup`, or channel-with-an-owner is a code smell in a review: it
means nobody is waiting, and nobody has thought about the exit path.

**Operator view.** A goroutine leak is invisible until it is not. The signature is
`go_goroutines` climbing monotonically on a Grafana panel for days — not a crash, not an error log,
just a slope. RSS grows because each leaked goroutine holds ≥2 KB of stack plus its captured
references; GC pause time grows because pause cost scales with live heap; eventually the pod
OOMKills. Nothing pages until the very end, which is why the postmortem has to reconstruct days of
slow decay from a metric nobody was watching.

**Hardware/runtime view.** The 2 KB starting stack is the whole reason Go's concurrency model
works: it makes a goroutine roughly 1/4000th the cost of an 8 MB pthread stack, which is what
makes goroutine-per-request and goroutine-per-item viable. The trade is that the compiler must
emit a stack-bound check in nearly every function prologue, and that stack growth means *copying*
frames and rewriting pointers — which is why Go can never hand a raw interior pointer to C without
pinning. The `GOMAXPROCS`/cgroup mismatch is the other side of the same coin: the runtime's model
of the machine has to match the kernel's, and until Go 1.25 it did not.

**Economics/scale view.** Concurrency limits are a cost lever, not only a correctness one. An
unbounded scrape fan-out against a billing API can trip rate limits or cost money per request. A
too-conservative limit leaves expensive GPUs idle waiting on CPU-bound preprocessing — precisely
the shape IceCube hit (below), where the GPU generation outran the CPU feeding it. Tuning
concurrency is tuning a dial between "too slow, wasting expensive idle hardware" and "too fast,
getting throttled or billed."

## Real-world use cases

- **Go 1.26/1.27 — the `goroutineleak` profile.** The Go release notes ship a real leaked-goroutine
  example: a fan-out sending results on an unbuffered channel, with an early `return` in the
  collection loop on the first error. Every remaining worker blocks on `ch <- result{...}` forever.
  The runtime detects it via GC reachability — once `ch` is unreachable from any runnable
  goroutine, the blocked senders provably cannot wake. Experimental in Go 1.26
  (`GOEXPERIMENT=goroutineleakprofile`); the flag is dropped in Go 1.27 (rc at the time of writing,
  due August 2026), exposed at `/debug/pprof/goroutineleak`. Contributed by Vlad Saioc (Uber) from the research behind LeakProf.
  *What it shows:* the single most common leak in production Go, and that the ecosystem considered
  it important enough to put detection in the runtime.

- **x/sync errgroup's panic-propagation reversal (2025).** Panic propagation through `Wait` was
  added in CL 644575 (January 2025, x/sync v0.14.0) and reverted in CL 682935 (June 2025, v0.16.0),
  with the maintainers leaving a permanent comment in the source. *What it shows:* the reasoning is
  the lesson — propagating a panic delays it arbitrarily, converts a stack trace into a value (so
  crash monitoring never sees it), and can deadlock if the panicking goroutine leaves the program
  unable to reach `Wait`. When a library declines to be helpful, there is usually a failure mode
  behind it.

- **[Google Cloud — "GKE GPU sharing helps scientists' quest for neutrinos"](https://cloud.google.com/blog/products/containers-kubernetes/gke-gpu-sharing-helps-scientists-quest-for-neutrinos)** —
  IceCube's photon-propagation simulation became **CPU-bound** on V100 and A100 hardware: the GPUs
  got faster than the CPU stage feeding them, so GPUs idled. GKE GPU time-sharing and A100 MIG
  partitioning (up to 7 partitions per A100) let several jobs share one GPU. Reported results: on
  the fastest workload configuration, ~40% more throughput on T4 and **~4.5×** on A100; on the most
  demanding configuration, no gain on T4/V100 and ~2× on A100. They preferred MIG over plain
  time-sharing for its stronger isolation. *What it shows:* concurrency sizing as a
  hardware-utilisation problem, in exactly your domain. *Figures are the blog's own dated snapshot.*

- **[Uber Engineering — "LeakProf: Featherlight In-Production Goroutine Leak Detection"](https://www.uber.com/blog/leakprof-featherlight-in-production-goroutine-leak-detection)** —
  Uber's two-pronged strategy: `goleak` in CI blocks PRs that introduce leaks, and LeakProf samples
  goroutine profiles in production, aggregates goroutines blocked at the *same source location*,
  and alerts the owning service when the count at one location exceeds a threshold. It suppresses
  known false positives by inspecting the AST of a reported `select` for ticker/timer arms. *What
  it shows:* what a mature org automates beyond a dashboard — and this work is the direct ancestor
  of the runtime leak profile above.

- **[Discord — "Why Discord is switching from Go to Rust"](https://discord.com/blog/why-discord-is-switching-from-go-to-rust)** —
  a fair counterpoint rather than a horror story. Discord's Read States service showed periodic
  latency spikes roughly every two minutes. The mechanism is verifiable in the runtime source:
  `forcegcperiod = 2 * 60 * 1e9` in `runtime/proc.go` forces a GC cycle every two minutes
  regardless of allocation rate, and their service held a large, mostly-live LRU cache — so every
  forced cycle scanned a big live heap that produced almost no garbage. *What it shows:* a real
  limit of a tracing GC for one workload shape (huge live set, low garbage), so "just use Go" is
  not a universal answer.

## Worked example

Fan out N concurrent HTTP scrapes under a total budget, bounded to `limit` concurrency, with a
per-node deadline and a peak-concurrency assertion hook. First hard error cancels everything in
flight. Runs clean under `-race` and under `goleak`.

```go
package scrape

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"sync/atomic"
	"time"

	"golang.org/x/sync/errgroup"
)

// Sample is one node's contribution to the scrape.
type Sample struct {
	Node    string
	GPUs    int
	Watts   float64
	Latency time.Duration
}

// Collector scrapes a fleet with bounded concurrency and a hard time budget.
type Collector struct {
	Client   *http.Client
	Limit    int           // max in-flight scrapes
	Budget   time.Duration // total deadline for the whole collection
	PerNode  time.Duration // per-node deadline (must be < Budget)

	// inFlight/peak exist so tests can assert the bound was respected.
	inFlight atomic.Int64
	peak     atomic.Int64
}

// Peak reports the highest observed concurrency. Test-only accessor.
func (c *Collector) Peak() int64 { return c.peak.Load() }

// Collect scrapes every node. It returns the samples that succeeded and the
// first error encountered; on deadline expiry it returns context.DeadlineExceeded.
func (c *Collector) Collect(ctx context.Context, nodes []string) ([]Sample, error) {
	ctx, cancel := context.WithTimeout(ctx, c.Budget)
	defer cancel() // releases the timer AND unlinks from the parent's children map

	g, ctx := errgroup.WithContext(ctx) // cancels ctx on the first non-nil error
	g.SetLimit(c.Limit)                 // blocks in g.Go, so only Limit goroutines exist

	// One slot per node: disjoint writes, so no mutex is needed and -race is happy.
	out := make([]Sample, len(nodes))

	for i, node := range nodes {
		g.Go(func() error {
			n := c.inFlight.Add(1)
			for { // lock-free max: only grows, so a CAS loop terminates
				p := c.peak.Load()
				if n <= p || c.peak.CompareAndSwap(p, n) {
					break
				}
			}
			defer c.inFlight.Add(-1)

			s, err := c.scrapeOne(ctx, node)
			if err != nil {
				return fmt.Errorf("scrape %s: %w", node, err)
			}
			out[i] = s
			return nil
		})
	}

	if err := g.Wait(); err != nil {
		// Every in-flight request was already aborted via the derived ctx.
		// context.Cause(ctx) returns the *originating* error, not just Canceled.
		return nil, err
	}
	return out, nil
}

func (c *Collector) scrapeOne(ctx context.Context, node string) (Sample, error) {
	// Per-node isolation: one slow node cannot eat the whole budget.
	// If the parent expires sooner, WithTimeout silently keeps the parent's deadline.
	ctx, cancel := context.WithTimeout(ctx, c.PerNode)
	defer cancel()

	start := time.Now()
	url := fmt.Sprintf("http://%s:10250/metrics/resource", node)
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return Sample{}, err
	}

	resp, err := c.Client.Do(req) // aborts when ctx is cancelled or expires
	if err != nil {
		return Sample{}, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return Sample{}, fmt.Errorf("status %d", resp.StatusCode)
	}

	var body struct {
		GPUs  int     `json:"gpus"`
		Watts float64 `json:"watts"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&body); err != nil {
		return Sample{}, fmt.Errorf("decode: %w", err)
	}

	return Sample{Node: node, GPUs: body.GPUs, Watts: body.Watts, Latency: time.Since(start)}, nil
}
```

Read the load-bearing lines:

- **`defer cancel()` twice.** In `Collect` it stops the budget timer and unlinks the `timerCtx`
  from the caller's children map. In `scrapeOne` it does the same per node — without it, 200 nodes
  × one dead `timerCtx` each accumulate in the collector's context for the full `PerNode` duration.
- **`out[i]` is race-free by construction.** Each goroutine writes exactly one distinct index. No
  two goroutines touch the same memory, so there is nothing to synchronise. (`-race` agrees: writes
  to disjoint elements of the same slice are not a race.) If you needed aggregation instead, a
  `sync.Mutex` around a map is the right answer, not a channel.
- **The CAS loop for `peak`** is the standard lock-free maximum. It terminates because `peak` only
  increases: either the load already exceeds `n`, or the swap wins, or someone else raised it and
  the next iteration sees a higher value.
- **`ctx` is reassigned by `errgroup.WithContext`.** Every subsequent use — including
  `http.NewRequestWithContext` — gets the *derived* context, so a sibling's error genuinely aborts
  in-flight HTTP.
- **What it does not protect against.** If `scrapeOne` were replaced with a CPU-bound computation
  that never checks `ctx`, cancellation would stop *new* work from launching but would not
  interrupt work already running. That is the cooperative-cancellation limit, restated.

Expected shape of a `-race` run against a fake server (representative):

```
$ go test -race -run TestCollect ./internal/scrape/
=== RUN   TestCollect_BoundedConcurrency
    collect_test.go:64: peak concurrency = 8 (limit 8)
--- PASS: TestCollect_BoundedConcurrency (0.09s)
=== RUN   TestCollect_DeadlineReturnsPromptly
    collect_test.go:88: elapsed 812ms, err = scrape node-17: Get "http://...": context deadline exceeded
--- PASS: TestCollect_DeadlineReturnsPromptly (0.81s)
ok      example.com/gpu-cost-exporter/internal/scrape   1.104s
```

## Practice

Build the collection stage of [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md) as a
bounded concurrent scrape.

1. `Collect(ctx context.Context, nodes []Node, limit int) ([]Sample, error)` scrapes per-node GPU
   metrics concurrently, capped at `limit` in-flight, using `errgroup` + `SetLimit`.
2. Derive a total scrape deadline from `ctx` with `WithTimeout` so a slow node cannot blow past
   Prometheus's 10 s `scrape_timeout`; pick the number by writing the budget arithmetic down (see
   the budget diagram) rather than guessing.
3. Give each node its own shorter `WithTimeout`, and `defer cancel()` on both.
4. First hard error cancels all in-flight scrapes; aggregate successful samples into disjoint slice
   indices (no lock) or a mutex-guarded map.
5. Use the `CostSource`-style fake from lesson 03 to drive the scrape in tests — no live endpoints
   needed to exercise bounded concurrency.
6. Add an `atomic.Int64` in-flight counter with a CAS peak, exposed via a test-only accessor.

**Acceptance:**

- `go test -race ./...` passes with no data race.
- `defer goleak.VerifyNone(t)` (or `goleak.VerifyTestMain`) passes — zero goroutines outlive each
  test.
- Cancelling the passed `ctx`, or letting the deadline expire, returns within ~100 ms with an error
  that satisfies `errors.Is(err, context.DeadlineExceeded)` or `context.Canceled`, and leaves zero
  goroutines running.
- Peak concurrency never exceeds `limit` — asserted, not assumed.
- `go vet ./...` is clean, which specifically means no `lostcancel` findings.

## Common pitfalls

1. **Treating `-race` as a leak detector.** It catches data races only. A `-race`-clean suite says
   nothing about goroutine leaks — different bug class, different tool. *Mechanism:* TSan
   instruments memory *accesses*; a leaked goroutine performs no conflicting access, it just never
   returns. *Correction:* `goleak` in tests, `go_goroutines` slope alerting in production, and the
   `goroutineleak` profile on Go 1.26+.

2. **Dropping the `cancel` from `WithTimeout`/`WithCancel`.** *Symptom:* slowly growing memory and
   a growing timer heap in a long-running process, with nothing in the logs. *Mechanism:* the
   `timerCtx` stays in the parent's `children` map and its `time.Timer` stays in the runtime timer
   heap holding the whole context chain alive, until the deadline fires. *Correction:*
   `defer cancel()` immediately after the constructor, always; `go vet`'s `lostcancel` catches the
   obvious cases.

3. **Storing `ctx` in a struct field "for convenience."** *Symptom:* cancellation silently stops
   working; shutdown hangs. *Mechanism:* the stored `ctx` belongs to the request that created the
   struct. Later work using it is unrelated to that request, so it will either never be cancelled
   or be cancelled at an unrelated moment. *Correction:* `ctx` is the first parameter of every
   function that does I/O. Never a field. (The one accepted exception is a struct that *is* a
   request, and even then the linters will complain.)

4. **Assuming cancellation aborts CPU-bound work.** *Mechanism:* cancellation is a closed channel;
   something has to read it. A loop with no `select` and no `ctx.Err()` check never reads it.
   *Correction:* check `ctx.Err()` between chunks of work, sized so the check runs at least every
   few milliseconds.

5. **Acquiring the semaphore inside the goroutine.** *Symptom:* memory spike proportional to input
   size even though "concurrency is bounded." *Mechanism:* `go` allocates the `g` and its 2 KB
   stack immediately; the semaphore only gates what runs *after* that. *Correction:* acquire before
   `go`, or use `errgroup.SetLimit`, which blocks inside `Go` before creating the goroutine.

6. **`GOMAXPROCS` mismatch inside a container.** *Symptom:* high latency and rising
   `container_cpu_cfs_throttled_seconds_total` at modest request rates. *Mechanism:* pre-Go-1.25
   the runtime sized itself to the host's CPU count, not the cgroup quota, so it over-parallelised
   into a throttle. *Correction:* Go 1.25+ handles it (CPU **limit**, not request); on 1.24 or
   earlier, set `GOMAXPROCS` explicitly or import `automaxprocs`.

7. **Closing a channel from the receiver, or from one of several senders.** *Symptom:*
   `panic: send on closed channel` under load, never in tests. *Mechanism:* there is no atomic
   "close if not closed" — the check and the close race. *Correction:* exactly one owner closes,
   and only when it is the sole remaining sender; with N senders, have a coordinator close after
   `wg.Wait()`.

## Self-check

**(a)** You must scrape 200 nodes but keep at most 8 requests in flight, and abort every in-flight
request the instant one node returns a fatal error. How?

**Answer:** `g, ctx := errgroup.WithContext(ctx)` then `g.SetLimit(8)`; launch each scrape with
`g.Go(func() error { return scrape(ctx, node) })` and finish with `g.Wait()`. `SetLimit(8)`
installs an 8-capacity `chan token` that `Go` sends into *before* starting the goroutine, so only 8
goroutines exist at a time. The first non-nil error is captured by `errOnce` and cancels the
derived context (via `context.WithCancelCause`), and every scrape passes that `ctx` into
`http.NewRequestWithContext`, so all in-flight requests abort. `g.Wait()` returns that first error;
later errors are discarded. Inside a worker, `context.Cause(ctx)` tells you which sibling's error
caused the cancellation.

**(b)** Show a goroutine leak and fix it.

**Answer:** The receiver returns early, so the sender blocks forever on an unbuffered channel:

```go
func leak() error {
	ch := make(chan result)                 // unbuffered
	for _, w := range work {
		go func() { ch <- process(w) }()    // blocks until someone receives
	}
	for range len(work) {
		r := <-ch
		if r.err != nil {
			return r.err                    // ← remaining senders stranded forever
		}
	}
	return nil
}
```

Each stranded goroutine holds ≥2 KB of stack plus everything `process` captured, forever. Two
fixes, and the choice matters:

```go
ch := make(chan result, len(work))          // fix 1: every send completes, no receiver needed
```

```go
select {                                     // fix 2: give the send an escape hatch
case ch <- process(w):
case <-ctx.Done():
}
```

Fix 1 costs `len(work)` slots of memory but is unconditionally correct. Fix 2 costs nothing but
requires a `ctx` that is actually cancelled on the early return — so pair it with
`defer cancel()`. In practice, use `errgroup`, which is fix 2 with the bookkeeping done for you.

**(c)** Why is `ctx` always the first argument, and what must a blocking call do with it?

**Answer:** Because cancellation propagates by *derivation*, and derivation only works if every
layer receives and forwards the same context. Making it position 0 of every signature makes the
chain uniform, greppable, and lint-checkable. One `cancel()` at the root then closes every `Done()`
channel in the subtree — synchronously, depth-first, in a single `cancelCtx.cancel` traversal —
which is how one `SIGTERM` unwinds an entire reconcile.

A blocking call must *honour* it. Either pass `ctx` into the underlying I/O (`net/http`, the k8s
client, gRPC, database drivers all select on `ctx.Done()` internally), or, in your own loops,
`select { case <-ctx.Done(): return ctx.Err(); default: }` / check `ctx.Err()` between iterations,
then return promptly. A context nobody reads cancels nothing — cancellation is cooperative, and
there is no preemption of user code. And never store it in a struct: a stored context belongs to
the request that created it, so it will be cancelled at a moment that has nothing to do with the
work now using it.

**(d)** Two goroutines write to the same `map[string]int` with no lock, channel, or atomic. `-race`
is not run. Is this "usually fine" the way it might be in CPython?

**Answer:** No, and this is the sharpest Python-to-Go delta in the lesson. Go has no GIL, so both
goroutines genuinely run in parallel on different cores. Per the memory model, an unsynchronised
write has *no* guaranteed visibility to a read in another goroutine — the guarantee list is closed
(channel ops, mutex lock/unlock, `sync/atomic`, `Once`, goroutine *creation*) and a bare shared
variable is on none of it. Worse, a map is a multiword structure, and the memory model explicitly
permits reads of multiword locations to be implemented as several independent word-sized reads in
unspecified order, which "can lead to arbitrary memory corruption."

In practice the concrete symptom is usually `fatal error: concurrent map writes` from the runtime's
own check — and note *fatal error*, not panic: `recover()` cannot catch it, the process dies
immediately. `go test -race` catches this before production. Skipping it in CI is not a minor gap.

**(e)** Your container has a `cpu: "2"` limit but the node has 64 cores. What Go-specific behaviour
can bite you, and on which versions?

**Answer:** Before Go 1.25, `GOMAXPROCS` defaults to `runtime.NumCPU()` — the *host's* 64, not the
cgroup's 2. The scheduler then runs up to 64 goroutines of Go code simultaneously, the cgroup CPU
bandwidth controller throttles the process to its 2-CPU quota, and you see elevated latency plus
rising `container_cpu_cfs_throttled_seconds_total` at modest load. Go 1.25 (August 2025) made the
runtime cgroup-aware: on Linux it defaults `GOMAXPROCS` to the cgroup CPU bandwidth limit when that
is lower than the CPU count, and on all platforms it periodically re-reads the limit. Two caveats:
it uses the CPU **limit**, not the request, so a pod with only `requests.cpu` set gets the host
count; and both behaviours switch off if you set the `GOMAXPROCS` env var or call
`runtime.GOMAXPROCS()`. On Go 1.24 or earlier, set `GOMAXPROCS` explicitly or import
`uber-go/automaxprocs`.

**(f)** Why is a `defer cancel()` you "know" is redundant still required?

**Answer:** Because `cancel` is the deallocation path, not just the signalling path.
`timerCtx.cancel` does three things: closes `Done()`, calls `removeChild` to unlink the context
from its parent's `children` map, and calls `timer.Stop()`. Skip it and the child stays in the
parent's map and the timer stays in the runtime timer heap — both holding the entire context chain
and its captured variables alive — until the deadline fires. In a long-lived parent context called
in a loop, that is a linearly growing map of dead contexts and a growing timer heap: a real leak
with no error message. `go vet`'s `lostcancel` analyser exists specifically for this.

## Connections & what's next

This lesson leans directly on lesson 03: bounding concurrent work with `errgroup` and testing it
without live dependencies both require the small, consumer-defined, fakeable interfaces that lesson
built. It completes the picture lesson 02 started with `ctx`-aware error wrapping — `context.Canceled`
and `context.DeadlineExceeded` flow through the same `%w` chains, and `context.Cause` is the
errgroup-shaped version of the same question ("what actually went wrong?"). Lesson 05 (testing) is
where `goleak.VerifyNone` and the peak-concurrency assertion become first-class test infrastructure,
and where `-race` becomes a standing CI requirement rather than a thing you remember to run. Lesson
09 (controller primer) is where `MaxConcurrentReconciles`, the workqueue's exponential rate limiter,
and the manager's `SIGTERM`-driven context cancellation stop being an analogy and become the
reconcile loop you write.

Next: **[05 · Testing](05-testing.md)** — the fakes from lesson 03 and the bounded, cancellable,
leak-checked concurrency from this lesson are what a serious Go test suite is actually testing.

## References & further reading

**Primary sources**

1. [The Go Memory Model](https://go.dev/ref/mem) — the closed list of happens-before edges
   reproduced above, the DRF-SC guarantee, and the implementation restrictions for racy programs
   (including the multiword-corruption clause). Version of June 6, 2022. Read for the exact wording
   of the buffered-channel capacity-C rule.
2. [`context` package documentation](https://pkg.go.dev/context) — signatures and semantics for
   `WithCancel`, `WithTimeout`, `WithDeadlineCause`, `WithCancelCause`, `Cause`, `WithoutCancel`,
   `AfterFunc`, including which Go release added each.
3. [`src/context/context.go`](https://github.com/golang/go/blob/master/src/context/context.go) —
   the ~800-line implementation this lesson walks: `cancelCtx`, `propagateCancel`, `parentCancelCtx`,
   the `closedchan` trick, `timerCtx`. Read it once; it is the fastest way to stop guessing about
   context behaviour.
4. [Data Race Detector](https://go.dev/doc/articles/race_detector) — report format, the full
   `GORACE` option table with defaults, supported platforms, and the 5–10× memory / 2–20× time
   overhead figures plus the 8-bytes-per-`defer` note.
5. [ThreadSanitizer algorithm](https://github.com/google/sanitizers/wiki/ThreadSanitizerAlgorithm) —
   the shadow-memory design behind `-race`: N shadow words per 8-byte application word, the bit
   layout of a shadow word, and random eviction when slots fill (which is why `-race` can miss).
6. [Go 1.25 release notes — container-aware `GOMAXPROCS`](https://go.dev/doc/go1.25) — the cgroup
   CPU-bandwidth default, the periodic re-read, the CPU-limit-not-request detail, and the
   `containermaxprocs` / `updatemaxprocs` GODEBUG escapes. Also `sync.WaitGroup.Go`.
7. [Go 1.26 release notes — experimental goroutine leak profile](https://go.dev/doc/go1.26) — the
   GC-reachability detection mechanism, the worked leak example, and the stated blind spot
   (primitives reachable from globals). The [Go 1.27 release notes](https://go.dev/doc/go1.27) mark it
   generally available; Go 1.27 was in release-candidate as this lesson was written.
8. [golang/go#74609 — goroutine leak profile proposal](https://github.com/golang/go/issues/74609) —
   the accepted proposal, its API (`runtime/pprof` profile `goroutineleak`,
   `/debug/pprof/goroutineleak`), and the validation data behind it.
9. [`golang.org/x/sync/errgroup` source](https://github.com/golang/sync/blob/master/errgroup/errgroup.go) —
   ~140 lines. The `sem chan token` semaphore, `errOnce`, `WithCancelCause`, and the "tsunami
   stone" comment explaining why panics are not propagated (issues #53757, #74275, #74304, #74306).
   *Correction to the previous version of this lesson:* errgroup does **not** propagate panics.
   That behaviour was added in v0.14.0 (Jan 2025) and reverted in v0.16.0 (Jun 2025).
10. [`go.uber.org/goleak`](https://github.com/uber-go/goleak) — the CI half of leak detection. Read
    `options.go` for the default filter list and the 20-retry / 100 ms-cap backoff that bounds how
    long `VerifyNone` waits.

**Real-world engineering blogs**

11. [Google Cloud — "GKE GPU sharing helps scientists' quest for neutrinos"](https://cloud.google.com/blog/products/containers-kubernetes/gke-gpu-sharing-helps-scientists-quest-for-neutrinos) —
    what it shows: a CPU-bound stage starving V100/A100 GPUs, and GPU time-sharing / MIG closing
    the utilisation gap (~40% on T4, ~4.5× on A100 for the fastest configuration). *Dated snapshot;
    treat the figures as illustrative.*
12. [Uber Engineering — "LeakProf: Featherlight In-Production Goroutine Leak Detection"](https://www.uber.com/blog/leakprof-featherlight-in-production-goroutine-leak-detection) —
    what it shows: goroutine-profile sampling, aggregation by blocking source location, a threshold
    alert to the owning service, and AST-based false-positive suppression. The direct ancestor of
    the runtime `goroutineleak` profile.
13. [Discord — "Why Discord is switching from Go to Rust"](https://discord.com/blog/why-discord-is-switching-from-go-to-rust) —
    what it shows: periodic latency spikes traced to Go's forced GC cycle (verifiable as
    `forcegcperiod = 2 * 60 * 1e9` in `runtime/proc.go`) over a large, mostly-live cache. A real
    limit of a tracing GC for one workload shape, not a general indictment.

**Deeper dives**

14. **GopherCon 2018 — Bryan C. Mills, "Rethinking Classical Concurrency Patterns"** —
    <https://www.youtube.com/watch?v=5zXAHh5tJqQ>. A Go-team engineer on why the textbook
    worker-pool and condition-variable patterns are usually the wrong shape in Go, and what to
    reach for instead. Watch for the reasoning behind bounded fan-out, not just the API.
15. **Learn Concurrent Programming with Go — James Cutajar (Manning, 2024).** *What-for:* a
    from-first-principles treatment of goroutines, channels, `select`, and the sync primitives with
    the race detector front and centre. *How:* deep-read the memory-model and channels chapters;
    skim the OS-threads intro. *Why:* current (Go 1.21+) and builds the "no GIL → real races"
    model a Python-native engineer needs. (Alternative: *Concurrency in Go*, Cox-Buday — excellent
    on pipelines but older. Pick one.)
16. **100 Go Mistakes and How to Avoid Them — Harsanyi** — <https://100go.co>. *What-for:* the
    concurrency chapter is a catalogue of the exact traps here: leaking goroutines, closing channels
    wrong, `ctx` propagation, `sync` misuse. *How:* skim now, reread when a `-race` failure baffles
    you. *Why:* every entry is a production bug you want to have already seen once.
