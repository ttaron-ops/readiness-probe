---
lesson: "01.3"
title: "Interfaces and composition"
module: "01"
concept: "Interfaces and composition"
status: not-started
est_time: "14h"
prev: "02-error-handling.md"
next: "04-concurrency-and-context.md"
artifacts: []
sources: 7
---

# 01.3 · Interfaces and composition

> **Concept.** Implicit interface satisfaction, small interfaces, accept-interfaces/return-structs, embedding — the design core of idiomatic Go and of controller-runtime.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

Lesson 02 gave you Go's error-handling spine: errors are values, `%w` wraps them, and a function's return path is the contract callers rely on. That last point turns out to matter more than it looked at the time — this lesson's central gotcha (the typed-nil trap) is a return-path bug that lives exactly where lesson 02 told you to look. Here you pick up the other load-bearing idea in Go's design: how types relate to each other without a class hierarchy. Interfaces are what let `CostSource` in your exporter, and `client.Client` in every controller you'll read, swap a live dependency for an in-memory fake with zero mocking framework. That capability is what unlocks lesson 04's concurrent fan-out (testable without spinning up N real HTTP endpoints) and lesson 05's whole testing strategy — you cannot write a fast, deterministic unit test for a Kubernetes controller without this lesson.

## Why this matters

This is the single load-bearing idea in Go design, and interviewers for platform roles probe it directly ("how do you test this without a real cluster?"). Interfaces are how you fake external dependencies with **zero** mocking library, and they are why `controller-runtime` is shaped the way it is: `client.Client`, `reconcile.Reconciler`, and `manager.Manager` are all interfaces, so your controller code depends on behavior, not on a live API server. Get this right and your Go reads like the code you already operate; get it wrong and you end up writing Java-in-Go with interface pollution everywhere — a producer-side `Service` interface with twelve methods that every fake must stub, half of them unused.

## What's new here (calibration)

Per the module README's calibration: you already know OOP, so we are not teaching "interfaces are like abstract classes." We skip: what an interface *is* as a general programming concept, class-hierarchy design, and any hello-world "here's how to declare an interface" tour. What's genuinely new and gets the depth: Go's *structural, implicit, compile-time-checked* satisfaction (no language you've used does this quite this way); the consumer-side-definition convention, which inverts where OOP habitually puts the abstraction; the typed-nil trap, which has no Python analog at all; and the interface-value internals (type pointer + data pointer) that mechanically explain both the nil trap and why type switches are cheap. This lesson also goes one level under the existing "how to write idiomatic interfaces" material into dispatch cost and comparability — the parts that matter once you're reading, not just writing, controller-runtime source.

## Core concepts

### The delta from Python

In Python you have *duck typing* (works at runtime, no declaration) or *ABCs* (`class Foo(SomeABC)` — explicit, checked at registration). Go has neither ceremony but keeps the safety: satisfaction is **implicit and structural**, checked at compile time. There is no `implements` keyword. A type satisfies an interface simply by having the right method set — the type never mentions the interface, and the interface never mentions the type.

```go
type Stringer interface{ String() string }

type Cluster struct{ Name string }
func (c Cluster) String() string { return c.Name } // now satisfies Stringer — nothing declared it
```

Deltas that bite Python engineers:

- **No registration, no inheritance tree.** Structural typing means a type can satisfy an interface defined in a package it has never imported. That is a feature — you write interfaces the *consumer* needs.
- **Compile-time, not runtime.** A missing method is a build error at the assignment site, not an `AttributeError` in prod.
- **Interfaces are tiny.** Idiomatic Go interfaces have one or two methods (`io.Reader` is one). Python ABCs tend to be fat base classes; do not port that habit.
- **The typed-nil trap** (below) has no Python analog and *will* surprise you — an interface value can be non-nil while the pointer inside it is nil. In Python, `None` is `None`; there is no "typed None."

### Implicit satisfaction, compile-time checked

No keyword. If the method set matches, it satisfies. To assert satisfaction at compile time (documentation + a guard against drift), use a blank-identifier check:

```go
var _ CostSource = (*PrometheusSource)(nil) // fails to compile if *PrometheusSource stops satisfying CostSource
```

This idiom is explicitly recommended by [Uber's Go style guide](https://github.com/uber-go/guide/blob/master/style.md) — worth adopting as a habit any time you export a type meant to satisfy a known interface. It costs nothing at runtime (the compiler evaluates and discards it) and turns a silent API drift into a build failure at the point where it happened, not three call sites downstream.

### Keep interfaces small — and define them on the consumer side

The stdlib is the reference: `io.Reader`, `io.Writer`, `fmt.Stringer` — each one method. Small interfaces are easy to satisfy, easy to fake, and compose freely. A one-method interface is named by the method plus `-er` (`Reader`, `Reconciler`). Resist the urge to declare a 12-method `Service` interface; that is **interface pollution** — a fat, speculative interface defined "in case someone needs it," where every unused method is dead weight a fake must implement for no test ever to exercise it.

The deeper rule, and the one that separates idiomatic Go from ported OOP: **define interfaces on the CONSUMER side, not the producer.** The package that *calls* the dependency declares the small interface describing what it needs; the package that *provides* the implementation just returns a concrete struct. This is a dependency-direction decision, not a style preference — consequences:

- The concrete package has no dependency on the interface — it does not even import it.
- Each consumer declares exactly the methods it uses, so interfaces stay small and honest.
- You can retrofit a fake around a type you don't own — a third-party Prometheus client, a cloud billing SDK — without forking it, because the interface lives in *your* package, not theirs.

```go
// package aggregate — the CONSUMER declares what it needs, nothing more
type CostSource interface {
    HourlyCost(ctx context.Context, node string) (float64, error)
}

func Total(ctx context.Context, src CostSource, nodes []string) (float64, error) {
    var sum float64
    for _, n := range nodes {
        c, err := src.HourlyCost(ctx, n)
        if err != nil {
            return 0, fmt.Errorf("cost for %s: %w", n, err)
        }
        sum += c
    }
    return sum, nil
}
```

### Accept interfaces, return structs

Functions take interface parameters (maximally flexible for the caller — any implementation works) but return concrete types (the caller gets the full API and can't be surprised by a shrunk method set). Returning an interface prematurely hides methods and forces guesswork.

```go
func NewPrometheusSource(addr string) *PrometheusSource { ... } // return the concrete struct
func Total(ctx context.Context, src CostSource, ...) ...        // accept the interface
```

This is a guideline, not an absolute — don't confuse it with "never return an interface." Returning `error` is universal and correct; a factory that legitimately hands back a family of implementations (an `io.Reader` chosen by file extension, say) is also a legitimate exception. The rule targets a specific failure: a premature interface return that hides a concrete API the caller actually needs, forcing them to type-assert back down to get at it.

### Embedding: composition, not inheritance

Go composes by *embedding*: put a type inside a struct (or an interface inside an interface) with no field name. The outer type promotes the inner type's methods, but this is composition, not inheritance — there is no `super`, no virtual dispatch, no overriding in the OOP sense. It is has-a wearing is-a's clothes.

```go
type Base struct{ Region string }
func (b Base) Where() string { return b.Region }

type Node struct {
    Base       // embedded — Node gets Where() promoted
    GPUCount int
}
n := Node{Base{"us-east-1"}, 8}
_ = n.Where() // "us-east-1", promoted from Base
```

**Struct embedding and interface embedding look identical syntactically but follow different promotion rules — worth seeing side by side once.** Struct embedding promotes both methods *and fields*, and can create genuine ambiguity: if two embedded types both define a method with the same name, the outer type's call to that method name is a compile error until you resolve it explicitly (`n.Base.Where()` vs `n.Other.Where()`).

```go
type A struct{}
func (A) Name() string { return "A" }

type B struct{}
func (B) Name() string { return "B" }

type C struct {
    A
    B
}
// c.Name() is a COMPILE ERROR: ambiguous selector c.Name
// must disambiguate: c.A.Name() or c.B.Name()
```

Interface embedding, by contrast, simply *unions method sets* — interfaces have no fields, so there is nothing to collide over. Two embedded interfaces can never produce an ambiguous selector; at worst you get a compile error if they declare the same method name with incompatible signatures, which is rare and immediately visible.

```go
type ReadWriter interface {
    io.Reader // embed — unions Read() into the method set
    io.Writer // embed — unions Write()
}
```

### Interface composition in practice: `client.Client`

This is exactly how `controller-runtime` builds its central type, and it's worth reading the real source rather than a paraphrase: [`pkg/client/interfaces.go`](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/client/interfaces.go) defines `Reader` (`Get`/`List`), `Writer` (`Create`/`Update`/`Delete`/`Patch`), and composes them — alongside `StatusClient` — into `Client`. Your reconciler is handed a `client.Client` interface — in prod it is a real API-server-backed client; in tests you pass `fake.NewClientBuilder().Build()`, a full in-memory implementation, and **none of your reconcile logic changes**. `manager.Manager` and `reconcile.Reconciler` (one method: `Reconcile(ctx, Request) (Result, error)`) are the same pattern. Recognizing this — small interfaces composing into exactly the capability surface a caller needs — is half of reading controller code fluently, in a project ([controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime)) with thousands of production deployments behind it.

### Interface internals: what's actually in an interface value

An interface value is not the underlying data — it's a **pair**: a pointer to a type descriptor, and a pointer to the actual data.

```
interface value
┌─────────────────┬─────────────────┐
│  type descriptor │   data pointer   │
│  (e.g. *Cluster) │  (points at the  │
│                  │   Cluster value) │
└─────────────────┴─────────────────┘
```

This one fact mechanically explains two things you'll hit constantly:

- **The nil-interface trap** (below): an interface equals `nil` only when *both* words are zero. Put a nil `*T` into the data-pointer slot and the type-descriptor slot is still set — non-nil interface, nil pointer inside it.
- **Type switches are effectively free.** `switch v := x.(type) { case *Cluster: ... }` compares the interface's type-descriptor pointer against a table of known types — a pointer comparison, not a reflective walk. This is why idiomatic Go code type-switches freely in hot paths without a second thought.

### Dispatch cost: static vs dynamic

Calling a method on a concrete type is a **static, direct call** — the compiler knows the exact function address, and it can often inline it. Calling a method through an interface value is a **dynamic, indirect call** — the runtime looks up the method in the type descriptor's method table (the itable) and jumps through that pointer. The itable lookup is small and usually irrelevant — don't rewrite working code to avoid it. But it is not zero, and it defeats inlining, so it's worth naming once: in a hot per-GPU scrape loop running at high frequency across a large fleet, going through an interface for every metric point is a measurable cost in a way that a single reconcile's `client.Get` call never is. "Just use interfaces everywhere" is a good default, not a free lunch.

### `any` vs a real interface

`any` (the Go 1.18+ alias for `interface{}`) says "accept literally anything; find out what it is with a type assertion or `reflect` at runtime." It is the **escape hatch**, not the idiom — it discards the compiler's ability to check anything about the argument until you unwrap it. A Python engineer's duck-typing instinct reaches for `any` where Go wants a small named interface instead: if you can name the one or two methods you actually call, do that — you keep compile-time checking and a self-documenting signature. Reserve `any` for genuinely unconstrained data: JSON blobs of unknown shape, a generic container, `fmt.Println`'s arguments.

### The nil-interface trap

An interface value equals `nil` only when **both** halves — type descriptor and data pointer — are nil. If you put a nil pointer into an interface, the type half is set, so the interface is non-nil — even though the pointer inside is nil. This is the single most Go-specific interface bug: it doesn't exist in Python (`None` is `None`, there's no "typed None"), and it silently inverts error-handling logic in exactly the pattern lesson 02 taught you to trust.

```go
func find() *CostError { return nil } // typed nil pointer

func do() error {
    var err error = find() // err now holds (*CostError, nil) — type half is set
    return err             // returns a NON-nil error!
}

if do() != nil {
    // this branch RUNS — classic bug: caller thinks an error occurred
}
```

It's most dangerous in exactly this shape — a concrete `*T` (nil) flowing through a named error/interface-typed **return value**. The code reads correct at every call site, which is why it survives code review. Fix: return the interface type directly (`func find() error`) and `return nil` for the nil case, or explicitly `return nil` at the call site rather than passing a typed-nil pointer up through an `error`.

A related, less-discussed trap: **don't assume two interface values are comparable in general.** `==` on interface values compares the dynamic type first, then the dynamic value — and that second comparison **panics at runtime** if the dynamic value's type is not comparable (a slice, a map, a function). It's not "returns false," it's a panic in production the first time that code path runs with the wrong concrete type behind the interface.

## Perspectives

**Developer/design view.** Consumer-side interface definition is fundamentally a dependency-direction decision. It makes the concrete package depend on nothing — no import of the interface, no coupling to how it's used — and puts the abstract boundary exactly where the need is: in the caller's package. That's the mechanical reason you can retrofit a fake around a type you don't own, like a third-party Prometheus client or a cloud billing SDK, without forking it or wrapping it in an adapter layer nobody asked for.

**Testing view.** Interfaces are Go's answer to "how do I unit test without a live cluster" — this isn't an incidental use of the feature, it's the reason `controller-runtime` is a stack of one- and two-method interfaces (`client.Reader`, `client.Writer`, `reconcile.Reconciler`). The hand-written fake you write in a `_test.go` file is production-grade testing infrastructure, not a shortcut you'll graduate out of once you "get a real mocking framework." Kubernetes itself ships and depends on this exact pattern.

**Systems/architecture view.** `client.Client`'s composition — `Reader` + `Writer` + `StatusClient` — is a real design pattern used by a project with thousands of production deployments, not a textbook example. Small interfaces compose into exactly the capability surface a caller needs, and that discipline keeps test doubles honest: a fake only needs to implement the slice of behavior its consumer actually declared, so a passing fake-backed test is evidence about a bounded, legible surface — not a leaky abstraction that happens to compile.

**Failure-mode view.** The typed-nil trap is the single most Go-specific interface bug you will encounter. It doesn't exist in Python — `None` is `None`, there is no "typed `None`." It silently inverts error-handling logic: `err != nil` evaluates `true` when the author's intent was "no error occurred." It passes code review because the code *reads* correct — `return err` after `err := helper()` looks unimpeachable. The bug is invisible until you know to look at the return type of `helper()`.

## Real-world use cases

- **[controller-runtime — `pkg/client/interfaces.go`](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/client/interfaces.go)** — the real, in-production interface definitions: `Reader` (`Get`/`List`), `Writer` (`Create`/`Update`/`Delete`/`Patch`), composed into `Client` alongside `StatusClient`. This is the exact "interface composition, not inheritance" pattern, verifiable in the actual dependency your controller will import — not a toy analog of it.
- **[client-go — fake client + informer example](https://github.com/kubernetes/client-go/blob/master/examples/fake-client/main_test.go)** — the official example wiring `fake.NewSimpleClientset` to a `SharedInformerFactory` for unit tests. Its own code comment admits the fake client "isn't designed to work with informer. It doesn't support resource version" — a useful, honest data point that even the reference fake has edges, and that a green fake-backed test doesn't prove behavior depending on resourceVersion or watch semantics (that needs `envtest` — see lessons 5 and 9).

## Worked example

Refactor a cost tool so its data source is an interface — the exact shape controller-runtime uses for its client. One real implementation hits Prometheus; a fake feeds fixed numbers in tests. The aggregation code depends only on `CostSource`.

```go
// package aggregate
package aggregate

import (
    "context"
    "fmt"
)

// CostSource is declared HERE, by the consumer — one method, exactly what Total needs.
type CostSource interface {
    HourlyCost(ctx context.Context, node string) (float64, error)
}

func Total(ctx context.Context, src CostSource, nodes []string) (float64, error) {
    var sum float64
    for _, n := range nodes {
        c, err := src.HourlyCost(ctx, n)
        if err != nil {
            return 0, fmt.Errorf("cost for %s: %w", n, err)
        }
        sum += c
    }
    return sum, nil
}
```

Real implementation — a concrete struct, returned concretely, that never imports `aggregate`:

```go
// package promsource
package promsource

import "context"

type PrometheusSource struct{ addr string }

func New(addr string) *PrometheusSource { return &PrometheusSource{addr: addr} } // return struct

func (s *PrometheusSource) HourlyCost(ctx context.Context, node string) (float64, error) {
    // ... query Prometheus, parse, return $/hr ...
    return 3.06, nil
}
```

Fake for tests — no mocking library, just a struct with the one method, backed by a map:

```go
// package aggregate (aggregate_test.go)
package aggregate

import (
    "context"
    "testing"
)

type fakeSource struct{ costs map[string]float64 }

func (f fakeSource) HourlyCost(_ context.Context, node string) (float64, error) {
    return f.costs[node], nil
}

var _ CostSource = fakeSource{} // compile-time proof the fake still satisfies the interface

func TestTotal(t *testing.T) {
    src := fakeSource{costs: map[string]float64{"a": 3.06, "b": 3.06}}
    got, err := Total(context.Background(), src, []string{"a", "b"})
    if err != nil {
        t.Fatal(err)
    }
    if got != 6.12 {
        t.Fatalf("got %v, want 6.12", got)
    }
}
```

`Total` never learns whether it is talking to Prometheus or a map. That is precisely what `fake.Client` buys a reconciler test — the same trick, one layer up.

## Practice

In [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md), make the data source a `CostSource` interface:

1. Declare `CostSource` in the package that *aggregates* costs (the consumer), with the minimal method set your aggregation loop actually calls.
2. Write one real implementation (pull node/instance prices from your live source) that returns a concrete `*struct` from its constructor.
3. Write one fake implementation, in the test file, backed by an in-memory map — no gomock, no testify mocks.
4. Add `var _ CostSource = (*RealSource)(nil)` and the same for the fake.

**Acceptance:** swapping the real implementation for the fake requires **no change to the aggregation code** — only the value passed in changes. If aggregation imports or names a concrete source type, the interface is in the wrong place; move it to the consumer.

## Common pitfalls

1. **Interface pollution.** Defining a fat, speculative interface on the producer side "in case someone needs it." Every unused method is dead weight a fake must implement for no test to ever exercise. Correction: let interfaces emerge on the consumer side, sized to what's actually called.
2. **The typed-nil trap in a return path.** Returning a concrete `*T` (nil) through a named error/interface-typed return is the most common shape of this bug — see Core concepts above. Correction: return the interface type from the function signature and a literal `nil`, never a typed-nil concrete pointer.
3. **Assuming interface values are always comparable.** Two interface values are `==` only if their dynamic types are comparable *and* equal, and their dynamic values are equal — an interface holding a slice or map as its dynamic value **panics** on `==`, it does not just return `false`. Correction: don't put slices/maps behind an interface you intend to compare; compare the underlying data explicitly if you need to.
4. **Confusing "accept interfaces, return structs" with "never return an interface."** Legitimate exceptions exist — `error`, a factory returning a family of implementations. Correction: the rule targets *premature* interface returns that hide a concrete API the caller needs, not every interface-typed return.
5. **Forgetting the fake-client-vs-real-client capability gap.** A green test against `fake.NewSimpleClientset` or `fake.NewClientBuilder` doesn't prove behavior that depends on resourceVersion or watch semantics — see the client-go example above, whose own comment admits this. Correction: for that class of behavior, use `envtest` (a real API server, no kubelet) — covered in lessons 5 and 9.

## Self-check

**(a) Why prefer defining interfaces on the consumer, not the producer?**
**Answer:** The consumer knows exactly which methods it uses, so the interface stays small and honest, and each caller can declare its own minimal view. The producer just returns a concrete struct and takes no dependency on the interface — so you can even retrofit an interface (and a fake) around a third-party type you don't own. Producer-side interfaces drift into fat, speculative, multi-method blobs (interface pollution).

**(b) How do you fake an external dependency with zero mocking library?**
**Answer:** Depend on a small interface, then write a plain struct whose methods return canned data (often from a map or struct field) and pass it in place of the real implementation. Implicit satisfaction means the fake only needs the right method set — no registration, no framework. This is exactly what `controller-runtime`'s `fake.Client` is: a full in-memory implementation of the `client.Client` interface.

**(c) Why can an interface holding a nil pointer not equal nil?**
**Answer:** An interface value is a (type descriptor pointer, data pointer) pair and equals `nil` only when both halves are nil. Assigning a typed nil pointer (e.g. `(*CostError)(nil)`) sets the type half, so the interface is non-nil even though the pointer inside is nil. Returning such a value through an `error` makes `err != nil` true when the caller expects success — return the interface type and a literal `nil` instead.

**(d) Why does comparing two interface values sometimes panic instead of just returning `false`?**
**Answer:** `==` on interface values first compares dynamic types, then dynamic values. If the dynamic type is uncomparable — a slice, a map, or a function — Go can't perform that second comparison and panics at runtime rather than silently returning `false`. This only surfaces when the concrete type behind the interface happens to be uncomparable, so it can pass code review and testing and then panic in production the first time that code path runs with the "wrong" concrete type.

**(e) Why is a type switch (`switch v := x.(type)`) cheap, and why is a method call through an interface not entirely free?**
**Answer:** An interface value is a (type descriptor pointer, data pointer) pair. A type switch compares the type-descriptor pointer against a small table of known types — a pointer comparison, not a reflective walk — so it's effectively free. A method call through an interface, by contrast, is a dynamic dispatch: the runtime looks up the method in the type descriptor's itable and jumps through that pointer, versus a static call's direct, often-inlined address. The cost is small and usually irrelevant, but it's real, and it's worth naming once before reaching for an interface in a hot per-GPU scrape loop.

## Connections & what's next

Interfaces are the hinge the rest of this module turns on. Lesson 02's error-handling discipline is what makes the typed-nil trap dangerous in the first place — a `return err` that reads correct but isn't. Lesson 04 leans on this lesson immediately: bounding concurrent work with `errgroup` and faking the things you fan out to (an HTTP client, a `CostSource`) both depend on small consumer-defined interfaces. Lesson 05 (testing) is where the fake-vs-real distinction from this lesson becomes a full strategy — table-driven tests against fakes, plus `envtest` for the resourceVersion/watch behavior fakes can't fake. Lesson 09 (controller primer) is where `client.Client`, `manager.Manager`, and `reconcile.Reconciler` — all interfaces you've now seen the shape of — stop being examples and become the actual machinery you write against.

Next: **[04 · Concurrency and context](04-concurrency-and-context.md)** — the thread that starts here (interfaces you can fake and bound) is what makes bounded, testable, cancellable fan-out possible at all.

## References & further reading

**Primary sources**
- [pkg.go.dev/sigs.k8s.io/controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime) — the authoritative package-doc surface for `client.Client`, `manager.Manager`, `reconcile.Reconciler`. Read for the composed method sets your controller code will actually call.
- [Effective Go — Interfaces](https://go.dev/doc/effective_go#interfaces) — the canonical source on interface conventions and embedding. Dated in places but authoritative; read to align your vocabulary with how the stdlib and k8s codebases actually talk about interfaces.
- [uber-go/guide — style.md](https://github.com/uber-go/guide/blob/master/style.md) — read for the compile-time interface-compliance guard (`var _ Interface = (*T)(nil)`) and the warning against embedding types in public structs.

**Real-world engineering blogs**
- [controller-runtime — `pkg/client/interfaces.go`](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/client/interfaces.go) — what it shows: the real `Reader`/`Writer`/`Client` composition your controller imports, not a paraphrase of it.
- [client-go — fake client + informer example](https://github.com/kubernetes/client-go/blob/master/examples/fake-client/main_test.go) — what it shows: the official fake-client pattern, plus an honest, documented limitation (no resourceVersion support) that tells you when to reach for `envtest` instead.

**Deeper dives**
- **Learning Go, 2nd ed. — "Types, Methods, and Interfaces" chapter** — Jon Bodner (O'Reilly, 2024), https://www.oreilly.com/library/view/learning-go-2nd/9781098139285/. Deep read: the clearest treatment of implicit satisfaction, accept-interfaces/return-structs, embedding, and the type-value pair behind the nil trap.
- **100 Go Mistakes — interface-pollution items** — https://100go.co. Skim the "interface on the producer side" and "interface pollution" entries — short, blunt corrections to fat-interface habits ported from Python/Java.
