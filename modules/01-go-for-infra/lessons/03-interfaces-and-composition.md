---
lesson: "01.3"
title: "Interfaces and composition"
module: "01"
concept: "Interfaces and composition"
status: not-started
est_time: "12h"
artifacts: []
---

# 01.3 · Interfaces and composition

> **Concept.** Implicit interface satisfaction, small interfaces, accept-interfaces/return-structs, embedding — the design core of idiomatic Go and of controller-runtime.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Why this matters

This is the single load-bearing idea in Go design, and interviewers for platform roles probe it directly ("how do you test this without a real cluster?"). Interfaces are how you fake external dependencies with **zero** mocking library, and they are why `controller-runtime` is shaped the way it is: `client.Client`, `reconcile.Reconciler`, and `manager.Manager` are all interfaces, so your controller code depends on behavior, not on a live API server. Get this right and your Go reads like the code you already operate; get it wrong and you end up writing Java-in-Go with interface pollution everywhere.

## From Python to Go

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
- **The typed-nil trap** (covered in Core notes) has no Python analog and *will* surprise you — an interface value can be non-nil while the pointer inside it is nil.

## Core notes

**Implicit satisfaction.** No keyword. If the method set matches, it satisfies. To assert satisfaction at compile time (documentation + a guard against drift), use a blank-identifier check:

```go
var _ CostSource = (*PrometheusSource)(nil) // fails to compile if *PrometheusSource stops satisfying CostSource
```

**Keep interfaces small.** The stdlib is the reference: `io.Reader`, `io.Writer`, `fmt.Stringer` — each one method. Small interfaces are easy to satisfy, easy to fake, and compose freely. A one-method interface is named by the method plus `-er` (`Reader`, `Reconciler`). Resist the urge to declare a 12-method `Service` interface; that is interface pollution.

**Define interfaces on the CONSUMER side, not the producer.** This is the rule that separates idiomatic Go from ported OOP. The package that *calls* the dependency declares the small interface describing what it needs; the package that *provides* the implementation just returns a concrete struct. Consequences:

- The concrete package has no dependency on the interface — it does not even import it.
- Each consumer declares exactly the methods it uses, so interfaces stay small and honest.
- You can retrofit a fake for a third-party type without touching it.

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

**Accept interfaces, return structs.** Functions take interface parameters (maximally flexible for the caller — any implementation works) but return concrete types (the caller gets the full API and can't be surprised by a shrunk method set). Returning an interface prematurely hides methods and forces guesswork.

```go
func NewPrometheusSource(addr string) *PrometheusSource { ... } // return the concrete struct
func Total(ctx context.Context, src CostSource, ...) ...        // accept the interface
```

**Embedding vs inheritance.** Go composes by *embedding*: put a type inside a struct (or an interface inside an interface) with no field name. The outer type promotes the inner type's methods, but this is composition, not inheritance — there is no `super`, no virtual dispatch, no overriding in the OOP sense. It is has-a wearing is-a's clothes.

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

**Interface composition** works the same way and is why `io.ReadWriter` exists:

```go
type ReadWriter interface {
    io.Reader // embed
    io.Writer
}
```

This is exactly how `controller-runtime` builds `client.Client`: it embeds `client.Reader` and `client.Writer` (which give `Get`/`List` and `Create`/`Update`/`Delete`/`Patch`). Your reconciler is handed a `client.Client` interface — in prod it is a real API-server-backed client; in tests you pass `fake.NewClientBuilder().Build()`, a full in-memory implementation, and **none of your reconcile logic changes**. `manager.Manager` and `reconcile.Reconciler` (one method: `Reconcile(ctx, Request) (Result, error)`) are the same pattern. Recognizing this is half of reading controller code fluently.

**The nil-interface gotcha.** An interface value is a *pair*: (concrete type, value). It equals `nil` only when **both** halves are nil. If you put a nil pointer into an interface, the type half is set, so the interface is non-nil — even though the pointer inside is nil.

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

Fix: return the interface type directly (`func find() error`) and `return nil` for the nil case, or explicitly `return nil` at the call site rather than passing a typed-nil pointer up through an `error`. This trap causes real "why is my err always non-nil" bugs in controllers.

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

`Total` never learns whether it is talking to Prometheus or a map. That is precisely what `fake.Client` buys a reconciler test.

## Practice

In [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md), make the data source a `CostSource` interface:

1. Declare `CostSource` in the package that *aggregates* costs (the consumer), with the minimal method set your aggregation loop actually calls.
2. Write one real implementation (pull node/instance prices from your live source) that returns a concrete `*struct` from its constructor.
3. Write one fake implementation, in the test file, backed by an in-memory map — no gomock, no testify mocks.
4. Add `var _ CostSource = (*RealSource)(nil)` and the same for the fake.

**Acceptance:** swapping the real implementation for the fake requires **no change to the aggregation code** — only the value passed in changes. If aggregation imports or names a concrete source type, the interface is in the wrong place; move it to the consumer.

## Self-check

**(a) Why prefer defining interfaces on the consumer, not the producer?**
**Answer:** The consumer knows exactly which methods it uses, so the interface stays small and honest, and each caller can declare its own minimal view. The producer just returns a concrete struct and takes no dependency on the interface — so you can even retrofit an interface (and a fake) around a third-party type you don't own. Producer-side interfaces drift into fat, speculative, multi-method blobs (interface pollution).

**(b) How do you fake an external dependency with zero mocking library?**
**Answer:** Depend on a small interface, then write a plain struct whose methods return canned data (often from a map or struct field) and pass it in place of the real implementation. Implicit satisfaction means the fake only needs the right method set — no registration, no framework. This is exactly what `controller-runtime`'s `fake.Client` is: a full in-memory implementation of the `client.Client` interface.

**(c) Why can an interface holding a nil pointer not equal nil?**
**Answer:** An interface value is a (type, value) pair and equals `nil` only when both halves are nil. Assigning a typed nil pointer (e.g. `(*CostError)(nil)`) sets the type half, so the interface is non-nil even though the pointer inside is nil. Returning such a value through an `error` makes `err != nil` true when the caller expects success — return the interface type and a literal `nil` instead.

## Resources

- **Learning Go, 2nd ed. — "Types, Methods, and Interfaces" chapter** — https://www.oreilly.com/library/view/learning-go-2nd/9781098139285/. **Deep read.** The clearest treatment of implicit satisfaction, accept-interfaces/return-structs, embedding, and the type-value pair behind the nil trap. For an experienced dev this is the fastest way to internalize *why* Go's rules differ from ABCs, not just *what* they are.
- **100 Go Mistakes — interface-pollution items** — https://100go.co. **Skim.** Read the "interface on the producer side" and "interface pollution" entries. Short, blunt, and directly correct the fat-interface / premature-abstraction habits ported from Python/Java — high signal for someone about to design controller packages.
- **Effective Go — Interfaces** — https://go.dev/doc/effective_go#interfaces. **Skim.** The canonical source on interface conventions and embedding. Dated in places but authoritative; read it to align your vocabulary with how the stdlib and k8s codebases actually talk about interfaces.
