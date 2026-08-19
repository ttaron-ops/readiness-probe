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
sources: 13
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

Transcripts and benchmarks below were captured on **go1.24.7 linux/amd64**, Intel Xeon @ 2.10 GHz, `GOMAXPROCS=4`. Library facts cite **controller-runtime v0.24.1** and **client-go v0.36.0**. Benchmark numbers are single-machine measurements — treat the ratios as informative and the absolute nanoseconds as machine-specific.

## Core concepts

### The delta from Python

Python gives you two ways to express "any object that can do X": **duck typing** (call the method and find out at runtime) and **ABCs** (`class Foo(SomeABC)` — declared, registered, checked at instantiation). Go gives you a third thing that is neither: satisfaction is **structural, implicit, and checked at compile time**. There is no `implements` keyword, the type never names the interface, and the interface never names the type.

```go
type Stringer interface{ String() string }

type Cluster struct{ Name string }

func (c Cluster) String() string { return c.Name } // now satisfies Stringer — nothing declared it
```

Four deltas that bite:

- **No registration, no inheritance tree.** A type can satisfy an interface declared in a package it has never heard of. That is the feature that makes consumer-side interfaces work, and it is the reason you can wrap a fake around a third-party client you don't own.
- **Compile-time, not runtime.** A missing method is a build error at the point of assignment or call — not an `AttributeError` in production at 3am.
- **Interfaces are tiny.** `io.Reader`, `io.Writer`, `fmt.Stringer`, `error`, `reconcile.Reconciler` — one method each. Python ABCs tend toward fat base classes; porting that habit produces interfaces nobody can fake.
- **The typed-nil trap has no Python analogue.** An interface value can be non-nil while the pointer inside it is nil. In Python, `None` is `None`; there is no "typed `None`."

There is one more difference in kind. In Python, the abstraction usually lives with the *provider*: the library defines the ABC, and you subclass it. In Go, the idiom is the reverse — the abstraction lives with the *consumer*. That inversion is the single highest-value habit in this lesson, and everything below builds toward why.

### Implicit satisfaction, and how to make the compiler check it

Satisfaction is checked wherever a concrete value is assigned to an interface-typed variable, parameter, return value, channel element, or map/slice element. Nowhere else. That means a type can drift out of satisfying an interface and nothing complains until some call site tries to use it — potentially in a different package, or in a test that runs only in CI.

The fix is a one-line compile-time assertion, and it costs nothing at runtime because the compiler evaluates and discards it:

```go
var _ CostSource = (*PrometheusSource)(nil) // fails to compile if *PrometheusSource drifts
var _ CostSource = fakeSource{}             // same guard for the test fake
```

`(*T)(nil)` is a typed nil pointer — no allocation, no initialisation, just a value of the right type for the compiler to check. The Uber Go style guide recommends this for exported types whose interface conformance is part of their contract, for families of types implementing one interface, and anywhere breaking conformance would break users. Put it directly above the type or its constructor, so the failure appears where the drift happened rather than at some distant call site.

What the compiler tells you when it fails is worth reading carefully, because two different messages mean two different bugs:

```
./main.go:26:17: cannot use Source{} (value of struct type Source) as Fetcher value in variable declaration: Source does not implement Fetcher (method Fetch has pointer receiver)
./main.go:31:16: cannot use e (variable of struct type Exporter) as Namer value in variable declaration: Exporter does not implement Namer (ambiguous selector Exporter.Name)
```

The first is a method-set problem (next section). The second is an embedding-ambiguity problem (further below). Go's error messages name the exact reason; you rarely need to guess.

### Method sets: why `*T` and `T` are different types to an interface

This is the rule that decides whether your reconciler compiles, and it comes straight from the spec ("Method sets"):

| Type | Method set |
|---|---|
| `T` | methods declared with receiver `T` |
| `*T` | methods declared with receiver `T` **or** `*T` |

`*T`'s method set is a superset. So:

- If **all** methods use value receivers, both `T` and `*T` satisfy the interface.
- If **any** required method uses a pointer receiver, **only `*T`** satisfies it.

The asymmetry has a reason, not just a rule. Calling a pointer-receiver method requires an address. When you have a `*T`, the compiler dereferences to call value-receiver methods — always possible. When you have a `T` *stored inside an interface*, there is no addressable variable to take the address of: the interface owns a copy, and letting a method mutate that copy would be silently useless. So the language refuses at compile time.

This is why every kubebuilder scaffold hands the manager a pointer:

```go
func (r *GPUCostReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) { … }

// &GPUCostReconciler{…} satisfies reconcile.Reconciler.
//  GPUCostReconciler{…} does not.
```

Related and separate: outside interfaces, the compiler *does* auto-insert `&` for you — `x.Mutate()` becomes `(&x).Mutate()` — but only when `x` is **addressable** (a variable, a slice element, a pointer dereference, a field of an addressable struct). Map elements and function results are not addressable, which is why `m["a"].Mutate()` fails to build. [Lesson 1](01-syntax-and-types.md) has the full addressability treatment; the interface-relevant half is: *interfaces never auto-address, because there is nothing to address.*

Practical rule: **pick one receiver style per type and stick to it.** If any method needs a pointer receiver, give them all pointer receivers, always construct with `&T{…}`, and add the `var _ I = (*T)(nil)` guard. Mixed receivers produce the confusing state where `T` satisfies one interface and `*T` satisfies a larger one, and reviewers cannot tell at a glance which you meant.

### What is actually inside an interface value

An interface value is **two machine words**, and essentially every surprising interface behaviour follows from that fact. The runtime has two shapes (`src/runtime/runtime2.go`):

```go
type iface struct {           // a non-empty interface: error, io.Reader, CostSource…
	tab  *itab
	data unsafe.Pointer
}

type eface struct {           // the empty interface: any / interface{}
	_type *_type
	data  unsafe.Pointer
}
```

The `itab` ("interface table") is the interesting one (`src/internal/abi/iface.go`):

```go
type ITab struct {
	Inter *InterfaceType // which interface this is (e.g. CostSource)
	Type  *Type          // which concrete type is inside (e.g. *PrometheusSource)
	Hash  uint32         // copy of Type.Hash, used by type switches
	Fun   [1]uintptr     // variable-length: the concrete method implementations,
	                     // in the interface's method order. Fun[0]==0 means "does not implement".
}
```

```
  var src CostSource = promsource.New("http://prom:9090")

   interface value (16 bytes on 64-bit)
   ┌─────────────────────────┬──────────────────────────┐
   │ tab  *itab              │ data unsafe.Pointer      │
   └───────────┬─────────────┴────────────┬─────────────┘
               │                          │
               ▼                          ▼
   ┌────────────────────────────┐   ┌──────────────────────────────┐
   │ itab                       │   │ PrometheusSource (heap)      │
   │  Inter → CostSource type   │   │   addr:  "http://prom:9090"  │
   │  Type  → *PrometheusSource │   │   rates: map[string]float64  │
   │  Hash  = <type hash>       │   └──────────────────────────────┘
   │  Fun[0]→ (*PrometheusSource).HourlyCost
   │  Fun[1]→ … (one slot per interface method, in the interface's
   │             own method order, so the call site can index it)
   └────────────────────────────┘

   For `any` (empty interface) there are no methods, so word 0 is the *_type
   directly and there is no itab at all.

   nil interface  = {tab: nil,       data: nil}      → == nil is TRUE
   typed nil      = {tab: <itab set>, data: nil}     → == nil is FALSE   ← the trap
```

**One important optimisation:** if the concrete type's representation is a single pointer, it is stored *directly* in the data word rather than boxed. The compiler calls this `IsDirectIface` (`cmd/compile/internal/types/type.go`) and it is true for pointers, channels, maps, funcs, `unsafe.Pointer`, and single-field structs/arrays wrapping one of those. Everything else — including a plain `int`, a `float64`, or any multi-field struct — must be stored *indirectly*, so putting one into an interface requires a heap allocation to point at. That is the mechanical reason `fmt.Println(x)` allocates, and the reason [lesson 1](01-syntax-and-types.md)'s escape analysis reported `escapes to heap` for interface boxing.

You can observe both words directly. This is not code to ship, but it makes the model concrete:

```go
type ifaceWords struct {
	tab  unsafe.Pointer
	data unsafe.Pointer
}

func dump(name string, err error) {
	w := (*ifaceWords)(unsafe.Pointer(&err))
	fmt.Printf("%-22s err==nil:%-6v type=%-16T tab=%v data=%v\n",
		name, err == nil, err, w.tab != nil, w.data != nil)
}
```

```
--- typed nil ---
doBad()                err==nil:false  type=*main.CostError  tab=true data=false
doGood()               err==nil:true   type=<nil>            tab=false data=false
errors.New("x")        err==nil:false  type=*errors.errorString tab=true data=true
var err error          err==nil:true   type=<nil>            tab=false data=false
```

Line 1 is the whole typed-nil bug, printed: the type word is set, the data word is nil, and `err == nil` is `false`.

### Where itabs come from, and what a dynamic call costs

For every (interface, concrete type) pair the compiler can see statically, it **builds the itab at compile time** and links it into the binary — no runtime work at all. For pairs it can't see statically (an interface-to-interface conversion, a type assertion on a value whose concrete type comes from somewhere else), the runtime builds one on demand in `getitab` (`src/runtime/iface.go`): look in a global `itabTable` hash table without a lock; on a miss, take `itabLock`, look again, then `persistentalloc` a new itab, fill `Fun` by matching method names/signatures, and add it. Itabs are never freed — they're small, permanent, and shared process-wide.

The call path, end to end:

```
  CONCRETE CALL                          INTERFACE CALL
  src := &PrometheusSource{…}            var src CostSource = &PrometheusSource{…}
  src.HourlyCost(ctx, n)                 src.HourlyCost(ctx, n)
  ───────────────────────────            ─────────────────────────────────────────
  compiler knows the target              compiler knows only the interface
        │                                      │
        ▼                                      ▼
  direct CALL to a fixed address        load  tab  ← src[0]
  (often INLINED away entirely;         load  fn   ← tab.Fun[i]     (i fixed at compile time)
   the arguments and body fuse into     CALL  fn                    (indirect, through a register)
   the caller)                                │
        │                                      ├─ cannot be inlined: the target
        ▼                                      │  isn't known until run time
  ~0.7 ns/element (measured)                   ▼
                                        ~3.2 ns/element (measured)

  TYPE ASSERTION  v, ok := x.(Linear)    TYPE SWITCH  switch v := x.(type)
  ────────────────────────────────       ───────────────────────────────────
  compare x's type word against a        same comparison, against a small
  known *_type (a pointer compare);      table; the compiler also emits an
  on match, copy the data out.           InterfaceSwitchCache the runtime
  The Go 1.24 runtime also keeps a       fills in lazily (runtime.interfaceSwitch)
  lazily-built TypeAssertCache           so repeated switches on the same
  (runtime.typeAssert).                  types skip the runtime call.
  ~0.8 ns (measured)                     ~1.0 ns (measured)
```

Now the honest numbers, because "interfaces are slow" is folklore and the truth is more useful. Two benchmarks, same machine, `-benchtime=300ms -count=3`:

**Benchmark A — a method whose body does real work (one float multiply), called in a dependency chain:**

```
BenchmarkConcreteCall-4     86702821    3.131 ns/op    0 B/op   0 allocs/op
BenchmarkConcreteCall-4     90996571    2.703 ns/op    0 B/op   0 allocs/op
BenchmarkConcreteCall-4     91630821    2.598 ns/op    0 B/op   0 allocs/op
BenchmarkInterfaceCall-4    89448938    3.004 ns/op    0 B/op   0 allocs/op
BenchmarkInterfaceCall-4    90733603    2.795 ns/op    0 B/op   0 allocs/op
BenchmarkInterfaceCall-4    93838890    2.595 ns/op    0 B/op   0 allocs/op
```

**Indistinguishable.** When the method body and the surrounding data dependencies dominate, the indirect call disappears into the noise.

**Benchmark B — a trivially inlinable accessor, summed over a 1,000-element slice:**

```
BenchmarkSumConcrete-4      528340     695.4 ns/op    0 B/op   0 allocs/op
BenchmarkSumConcrete-4      480370     702.6 ns/op    0 B/op   0 allocs/op
BenchmarkSumIface-4         105544    3416   ns/op    0 B/op   0 allocs/op
BenchmarkSumIface-4         118198    3079   ns/op    0 B/op   0 allocs/op
```

Per element: ~0.70 ns concrete vs ~3.2 ns through the interface — about **4.4×**, or roughly 2.5 ns of extra cost per call. Almost none of that is the indirect jump itself; the bulk is **lost inlining** and the optimisations inlining would have enabled (keeping the value in a register, eliminating the load, unrolling).

And the cost that is usually much bigger than either:

```
BenchmarkBoxAlloc-4       15404826     15.08 ns/op    8 B/op   1 allocs/op
```

Boxing a non-pointer-shaped value into an interface allocates. 15 ns and 8 bytes of GC-tracked garbage per boxing, in a loop that does nothing else.

**Conclusions to actually use:**

1. Interfaces at architectural boundaries — a `CostSource`, a `client.Client`, a `Reconciler` invoked once per object — cost nothing you can measure. Use them freely.
2. An interface call in the innermost loop of a per-sample transformation *is* measurable when the method is trivial, because you lose inlining. If profiling points there, take the concrete type in that one function.
3. **Boxing allocations usually dominate dispatch.** `[]Valuer` of 1 M non-pointer values is 1 M heap objects; `[]Sample` is one allocation. That's a GC-pressure decision, not a call-overhead decision.
4. Measure before you contort anything. `go test -bench . -benchmem` and `pprof` — [lesson 5](05-testing.md) — answer this in minutes.

### The typed-nil trap

Now the mechanism has a name and a picture, the bug becomes obvious rather than mystical.

```go
type CostError struct{ Namespace string }

func (e *CostError) Error() string { return "cost error: " + e.Namespace }

func findTyped() *CostError { return nil } // returns a TYPED nil pointer

func doBad() error {
	err := findTyped() // err is *CostError, value nil
	return err         // converting *CostError → error sets the TYPE word
}
```

`doBad()` returns an interface whose `tab` points at the `(error, *CostError)` itab and whose `data` is nil. `err != nil` is `true`. The caller runs its failure branch for an error that does not exist — and every line of that code reads correct.

It gets worse in the shape you will actually write it: a **named result** plus a helper.

```go
func reconcileOnce(ctx context.Context) (err error) { // named result of interface type
	var cerr *CostError                              // concrete typed pointer
	defer func() {
		err = cerr // ALWAYS non-nil after this, even when cerr is nil
	}()
	…
}
```

Three fixes, in order of preference:

1. **Never let a concrete error pointer be the thing you return.** Declare helpers as returning `error`, not `*MyError`:

   ```go
   func find() error { … return nil }   // nil literal → both words zero
   ```

2. **If you must have a concrete-typed variable, convert explicitly at the boundary:**

   ```go
   func doGood() error {
   	if e := findTyped(); e != nil {
   		return e
   	}
   	return nil // literal nil: tab and data both zero
   }
   ```

   Confirmed by the dump above: `doGood()` prints `err==nil:true  tab=false data=false`.

3. **Linters catch the common shapes.** `golangci-lint`'s `nilnil` and `nilerr` checks, and `staticcheck`'s SA4023 ("impossible comparison of interface value with untyped nil"), flag several variants. None catches all of them, which is why the rule matters more than the tool.

The general statement, worth memorising in this exact form: **an interface is nil only when it holds no type at all.** A nil pointer *of a known type* is a perfectly good value to put in an interface — the language is not confused, your expectation is.

### Interface comparison: sometimes a panic, not a `false`

Per the spec, two interface values are equal if their dynamic types are identical **and** their dynamic values are equal. So:

```
--- interface comparison ---
any(1) == any(1): true
any(int64(1)) == any(int(1)): false (different dynamic types)
comparing two Uncomparable behind any...
recovered: runtime error: comparing uncomparable type main.Uncomparable
```

Two consequences:

- **Different dynamic types are never equal**, even when the underlying numbers are. `any(int64(1)) == any(int(1))` is `false`. This bites when comparing values that round-tripped through `encoding/json` (every number decodes into `float64`) or through an untyped map.
- **If the dynamic type is uncomparable** — a slice, map, or func, or a struct containing one — the comparison **panics at runtime**. The compiler allowed it because `any == any` is legal in the type system; the runtime discovered a type it cannot compare. The same panic occurs transitively: comparing structs with interface fields, arrays of interfaces, or using such a value as a map key.

The operational danger is that this passes review and passes tests, then panics the first time production puts the "wrong" concrete type behind the interface. If you compare interface values at all, constrain what can be behind them, or compare something else (a key, an ID) instead.

### Keep interfaces small — and define them on the consumer side

The stdlib is the reference, and it is ruthless: `io.Reader`, `io.Writer`, `io.Closer`, `fmt.Stringer`, `error`, `sort.Interface` (three), `http.Handler` (one). The naming convention follows from the size: a one-method interface is the method name plus `-er` — `Reader`, `Reconciler`, `Scaler`.

Small interfaces are not an aesthetic. They are what makes the following three things possible at once: easy satisfaction (many types qualify without trying), easy faking (a fake implements exactly what's called), and free composition (embed two to get their union).

The deeper rule, and the one that separates idiomatic Go from ported OOP: **define interfaces in the package that CONSUMES them, not the package that implements them.**

```
   PRODUCER-SIDE (the Java/Python habit)         CONSUMER-SIDE (idiomatic Go)
   ───────────────────────────────────────       ────────────────────────────────────
   package promsource                            package promsource
     type CostSource interface { … 12 methods }    type PrometheusSource struct{…}
     type PrometheusSource struct{…}               func New(addr) *PrometheusSource
            ▲                                              ▲
            │ imports                                      │ imports
            │                                              │
   package aggregate                             package aggregate
     func Total(src promsource.CostSource, …)      type CostSource interface {        ← 1 method
                                                     HourlyCost(ctx, node) (float64, error)
   Dependency: aggregate → promsource            }
   Fake must implement all 12 methods.             func Total(src CostSource, …)
   Cannot fake a type you don't own.
                                                 Dependency: NOTHING points at aggregate's
                                                 interface. promsource doesn't import it and
                                                 doesn't know it exists. main wires them.

   ┌──────────┐        ┌───────────────────────┐        ┌──────────────┐
   │   main   │───────▶│ aggregate.Total(src)  │◀╌╌╌╌╌╌╌│ *PrometheusSource │
   └──────────┘  wires │  needs: CostSource    │ satisfies (structurally,
                       └───────────────────────┘         no import, no declaration)
                                  ▲
                                  ╎ also satisfied by
                       ┌──────────┴──────────┐
                       │ fakeSource (in the  │
                       │ consumer's _test.go)│
                       └─────────────────────┘
```

The consequences are concrete:

- **The producer takes no dependency on the abstraction.** `promsource` imports nothing from `aggregate`. You can delete `aggregate` and `promsource` still builds.
- **Each consumer declares exactly what it calls**, so interfaces stay small and honest by construction. A second consumer that needs two methods declares its own two-method interface; nobody has to agree on one "official" interface.
- **You can retrofit a fake around a type you don't own.** A third-party Prometheus client, a cloud billing SDK, `*sql.DB` — you never need the vendor to have declared an interface, because you declare the shape you use.
- **It documents the true dependency surface.** Reading `func Total(ctx context.Context, src CostSource, nodes []string)` tells you exactly what `Total` can do to the outside world: one call, per node.

The counter-rule, so this doesn't become dogma: **don't define an interface at all until you have a reason.** One implementation and no test double means the concrete type is the better parameter. The reasons that justify one are: a test needs a substitute, there genuinely are two-plus implementations, or the dependency crosses a module boundary you want to keep stable.

### Accept interfaces, return structs

Take interface parameters (any implementation works, and callers can substitute); return concrete types (callers get the full API and can't be surprised by a shrunken method set).

```go
func New(addr string) *PrometheusSource                                  // return the concrete struct
func Total(ctx context.Context, src CostSource, nodes []string) (Report, error) // accept the interface
```

The mechanism behind "return structs" is worth stating because it's not only style:

- **Methods survive.** `*PrometheusSource` has an `Addr()` method that isn't in `CostSource`. Return `CostSource` and every caller who needs `Addr()` must type-assert back down — an unchecked runtime operation to recover something the compiler had.
- **Allocation.** Returning a concrete value can leave it in the caller's frame; returning it as an interface generally forces the boxing allocation described above ([lesson 1](01-syntax-and-types.md)'s escape analysis).
- **Documentation.** `func New(addr string) *PrometheusSource` tells the reader what they got. `func New(addr string) CostSource` hides it.

The legitimate exceptions, so you can tell a violation from a design:

| Returning an interface is fine when | Example |
|---|---|
| It's `error` | every function in Go |
| A factory genuinely selects among implementations | `func openReader(path string) (io.Reader, error)` picking gzip vs plain |
| The concrete type is unexported and must stay that way | `rate.NewLimiter`-style constructors returning a sealed behaviour |
| The stdlib does it for compositional reasons | `io.MultiWriter`, `io.LimitReader` |

The failure the rule targets is the **premature** interface return that hides an API the caller actually needs.

### Optional capabilities via type assertion

Small interfaces plus type assertions give you optional behaviour without fat interfaces. The stdlib does this everywhere: `io.Copy` checks whether the destination implements `io.ReaderFrom` (and the source `io.WriterTo`) to skip the buffer; `net/http` handlers check `http.Flusher` and `http.Hijacker`. controller-runtime does it too — `client.WithWatch` is `Client` plus `Watch`, and code that needs watching asserts for it.

Your exporter can use the same pattern:

```go
// The base contract: one method.
type CostSource interface {
	HourlyCost(ctx context.Context, node string) (float64, error)
}

// An optional extra capability, declared separately.
type NodeLister interface {
	Nodes(ctx context.Context) ([]string, error)
}

func TotalAll(ctx context.Context, src CostSource) (Report, error) {
	lister, ok := src.(NodeLister) // comma-ok: never panics
	if !ok {
		return Report{}, fmt.Errorf("source %T cannot list nodes", src)
	}
	nodes, err := lister.Nodes(ctx)
	if err != nil {
		return Report{}, fmt.Errorf("list nodes: %w", err)
	}
	return Total(ctx, src, nodes)
}
```

Two rules for this pattern: **always use the comma-ok form** (a bare `src.(NodeLister)` panics on failure), and **always have a fallback path** — an optional capability that is really mandatory should just be in the interface.

### Embedding: composition that reads like inheritance and isn't

Go composes by **embedding**: declare a type inside a struct (or an interface inside an interface) with no field name. The unqualified type name becomes the field name (spec, "Struct types").

```go
type Base struct{ Region string }

func (b Base) Where() string { return b.Region }

type Node struct {
	Base          // embedded: Node gets Where() and .Region promoted
	GPUCount int
}

n := Node{Base{"us-east-1"}, 8}
_ = n.Where()  // "us-east-1"   — shorthand for n.Base.Where()
_ = n.Region   // "us-east-1"   — shorthand for n.Base.Region
```

**This is not inheritance.** There is no `super`, no virtual dispatch, no overriding. `Node.Where()` is literally rewritten by the compiler to `Node.Base.Where()`, and `Base`'s methods can never see `Node` — if `Base` had a method calling `b.Where()` and `Node` declared its own `Where()`, `Base`'s call would still reach `Base`'s. Anyone expecting template-method polymorphism gets silently wrong behaviour. **Embedding delegates; it does not dispatch.**

The promotion rules are precise (spec, "Selectors"):

- The **depth** of a field or method is the number of embedded fields traversed to reach it. Declared directly on `T`: depth 0. Declared on something embedded in `T`: depth 1. And so on.
- `x.f` resolves to the `f` at the **shallowest depth**. If there is not *exactly one* `f` at that depth, the selector is **illegal** — a compile error, not a silent pick.
- For method-set purposes: embedding `T` puts `T`'s value-receiver methods into both `S` and `*S`, and `*T`'s methods into `*S` only. Embedding `*T` puts both into `S` and `*S`.

Ambiguity is a real error you will hit when two embedded types share a method name:

```go
type Cache struct{ Hits int }
func (Cache) Name() string { return "cache" }

type Metrics struct{ Scrapes int }
func (Metrics) Name() string { return "metrics" }

type Exporter struct {
	Cache
	Metrics
}
```

```
./main.go:30:16: ambiguous selector e.Name
./main.go:31:16: cannot use e (variable of struct type Exporter) as Namer value in variable declaration: Exporter does not implement Namer (ambiguous selector Exporter.Name)
```

Note the second line: ambiguity doesn't just break the call, it removes the method from the type's method set, so the type stops satisfying interfaces. Resolve by qualifying (`e.Cache.Name()`) or by declaring `Name()` on `Exporter` itself — depth 0 beats depth 1, so an explicit method always wins and is the idiomatic way to "override".

**Interface embedding** is different and simpler: interfaces have no fields, so there's nothing to collide over, and embedding just **unions the method sets**.

```go
type ReadWriter interface {
	io.Reader // unions Read(p []byte) (int, error)
	io.Writer // unions Write(p []byte) (int, error)
}
```

Since Go 1.14, overlapping method sets in embedded interfaces are legal as long as the signatures match exactly — `interface { io.Reader; io.ReadCloser }` compiles. Mismatched signatures for the same name are a compile error.

One more form, with a sharp edge worth knowing: **embedding an interface in a struct.**

```go
type partialClient struct {
	client.Client // embedded INTERFACE, nil unless you set it
	getFn func(context.Context, client.ObjectKey, client.Object, ...client.GetOption) error
}

func (p *partialClient) Get(ctx context.Context, key client.ObjectKey, obj client.Object, opts ...client.GetOption) error {
	return p.getFn(ctx, key, obj, opts...)
}
```

This makes `partialClient` satisfy the whole 10+-method `client.Client` while implementing only `Get`. It's a genuinely useful trick for wrapping a big third-party interface — but **every method you didn't override panics with a nil pointer dereference** when called, because the embedded interface field is nil. Use it deliberately, in tests, when you know the code path; never ship it as production plumbing. And note the Uber style guide's separate warning about embedding *types* in **public** structs: it leaks implementation details and makes adding methods to the embedded type a breaking change for you. Prefer an explicit named field plus delegating methods in exported API.

### Interface composition in practice: `client.Client`

controller-runtime's central type is the canonical example, and reading the real definitions is worth more than any paraphrase (v0.24.1, `pkg/client/interfaces.go`):

```go
type Reader interface {
	Get(ctx context.Context, key ObjectKey, obj Object, opts ...GetOption) error
	List(ctx context.Context, list ObjectList, opts ...ListOption) error
}

type Writer interface {
	Apply(ctx context.Context, obj runtime.ApplyConfiguration, opts ...ApplyOption) error
	Create(ctx context.Context, obj Object, opts ...CreateOption) error
	Delete(ctx context.Context, obj Object, opts ...DeleteOption) error
	Update(ctx context.Context, obj Object, opts ...UpdateOption) error
	Patch(ctx context.Context, obj Object, patch Patch, opts ...PatchOption) error
	DeleteAllOf(ctx context.Context, obj Object, opts ...DeleteAllOfOption) error
}

type StatusClient interface {
	Status() SubResourceWriter
}

type SubResourceWriter interface {
	Create(ctx context.Context, obj Object, subResource Object, opts ...SubResourceCreateOption) error
	Update(ctx context.Context, obj Object, opts ...SubResourceUpdateOption) error
	Patch(ctx context.Context, obj Object, patch Patch, opts ...SubResourcePatchOption) error
	Apply(ctx context.Context, obj runtime.ApplyConfiguration, opts ...SubResourceApplyOption) error
}

type Client interface {
	Reader
	Writer
	StatusClient
	SubResourceClientConstructor

	Scheme() *runtime.Scheme
	RESTMapper() meta.RESTMapper
	GroupVersionKindFor(obj runtime.Object) (schema.GroupVersionKind, error)
	IsObjectNamespaced(obj runtime.Object) (bool, error)
}

type WithWatch interface {
	Client
	Watch(ctx context.Context, obj ObjectList, opts ...ListOption) (watch.Interface, error)
}
```

Read that as a design document, because it is one:

- **The pieces are usable alone.** A function that only reads takes `client.Reader` — two methods, trivially fakeable, and its signature documents that it cannot mutate the cluster. That is a security- and review-relevant property expressed in the type system.
- **`Client` is the union, not a new abstraction.** It adds only the four methods that genuinely have nowhere else to live.
- **`StatusClient` is a separate interface returning another interface**, mirroring the API server's status subresource — spec and status are written through different calls because they have different ownership. [Lesson 9](09-controller-primer.md) builds on that.
- **`WithWatch` is `Client` + one method** — the optional-capability pattern, at library scale.
- **`reconcile.Reconciler` is one method**: `Reconcile(context.Context, Request) (Result, error)`. Your entire controller is a type satisfying a one-method interface.

Recognising this shape — small interfaces composing into exactly the capability surface a caller needs — is half of reading controller code fluently.

### Faking without a mocking library — and what fakes can't do

A fake is a struct with the methods, backed by a map. That's it. No `gomock`, no `testify/mock`, no code generation, no `EXPECT().Times(1)` DSL.

```go
type fakeSource struct {
	costs map[string]float64
	fail  map[string]error
	calls []string // recording: assert on interaction when you need to
}

func (f *fakeSource) HourlyCost(_ context.Context, node string) (float64, error) {
	f.calls = append(f.calls, node)
	if err, ok := f.fail[node]; ok {
		return 0, err
	}
	return f.costs[node], nil
}

var _ CostSource = (*fakeSource)(nil)
```

Because satisfaction is structural, the fake needs no registration and the production code needs no change. Because the interface is one method, the fake is eight lines. Both properties come from decisions made earlier in this lesson — this is where they pay off.

**Fakes have limits, and knowing them is the senior signal.** controller-runtime's own fake client documents its limitations in the package doc (v0.24.1, `pkg/client/fake/doc.go`), and it opens by telling you not to use it:

> "When in doubt, it's almost always better not to use this package and instead use envtest.Environment with a real client and API server."
>
> ⚠️ Current Limitations / Known Issues with the fake Client ⚠️
> - no way to inject specific errors to test handled vs. unhandled errors
> - partial sub-resource support, which can break tests that update metadata and status in one reconcile
> - no OpenAPI validation on create/update
> - `Generation` and `ResourceVersion` "don't behave properly" — operations relying on them "will fail, or give false positives"

client-go's own fake-client example carries the same warning for informers: *"the fake client isn't designed to work with informer. It doesn't support resource version. It's encouraged to use a real client in an integration/E2E test if you need to test complex behavior with informer/controllers."*

So the testing ladder, which [lesson 5](05-testing.md) formalises:

| Level | What it proves | Cost |
|---|---|---|
| Hand-written fake behind your own small interface | your logic, given specified inputs | microseconds |
| `fake.NewClientBuilder()` client | your reconcile flow against an object store | milliseconds |
| `envtest` (real apiserver + etcd, no kubelet) | resourceVersion, conflicts, validation, watch semantics | seconds |
| A real cluster | scheduling, kubelet, CNI, device plugins | minutes |

The rule: **fake what you own, envtest what you don't.** A green fake-backed test says nothing about optimistic-concurrency conflicts, because the fake doesn't implement resourceVersion.

### Interface pollution: the failure mode in the other direction

The habit ported from Java/Python is to declare an interface for every service "in case someone needs it." In Go that produces:

- A 12-method `Service` interface with one implementation, so the abstraction buys nothing and every fake must stub 12 methods to exercise 2.
- Producer-side placement, so consumers import the provider package anyway — the dependency the interface was supposed to break is still there.
- Names like `ICostSource` or `CostSourceInterface`, which are a smell in Go: the interface gets the short behavioural name, and the implementation gets the specific one (`CostSource` the interface, `PrometheusSource` the struct).

The corrective, in one line: **interfaces should be discovered, not designed.** Write the concrete implementation. Write the consumer taking the concrete type. When a second implementation or a test double appears, extract the interface *in the consumer's package*, sized to the calls that are actually there.

## Perspectives

**Developer/design view.** Consumer-side interface definition is fundamentally a dependency-direction decision. It makes the concrete package depend on nothing — no import of the interface, no coupling to how it's used — and puts the abstract boundary exactly where the need is: in the caller's package. That's the mechanical reason you can retrofit a fake around a type you don't own, like a third-party Prometheus client or a cloud billing SDK, without forking it or wrapping it in an adapter layer nobody asked for. It also makes signatures honest: `func Total(ctx, src CostSource, nodes []string)` states its entire dependency surface in one line, and `client.Reader` in a signature is a machine-checked promise not to mutate the cluster.

**Testing view.** Interfaces are Go's answer to "how do I unit test without a live cluster" — this isn't an incidental use of the feature, it's the reason `controller-runtime` is a stack of one- and two-method interfaces (`client.Reader`, `client.Writer`, `reconcile.Reconciler`). The hand-written fake in a `_test.go` file is production-grade testing infrastructure, not a shortcut you'll graduate out of once you "get a real mocking framework" — Kubernetes itself ships and depends on this pattern. The senior half of the skill is knowing the ceiling: controller-runtime's fake client tells you in its own package doc that `ResourceVersion` and `Generation` "don't behave properly" and that you should probably use `envtest` instead. A green fake test is evidence about your logic, not about the API server's concurrency semantics.

**Systems/architecture view.** `client.Client`'s composition — `Reader` + `Writer` + `StatusClient` + `SubResourceClientConstructor` — is a real design used by a project with thousands of production deployments, not a textbook example. Small interfaces compose into exactly the capability surface a caller needs, and `WithWatch = Client + Watch` shows the optional-capability pattern at library scale. That discipline keeps test doubles honest: a fake only implements the slice of behaviour its consumer declared, so a passing fake-backed test is evidence about a bounded, legible surface rather than a leaky abstraction that happens to compile.

**Performance view.** The two-word interface value explains the cost model exactly, and the measurements refuse to be dramatic. A method call through an interface costs about 2.5 ns more than a concrete call on a trivially inlinable method (0.70 → 3.2 ns/element over a 1,000-element slice) and is *unmeasurable* when the method body does real work. The cost that dominates is not dispatch at all — it's **boxing**: putting a non-pointer-shaped value into an interface heap-allocates (8 B, ~15 ns per boxing measured here), so `[]Valuer` of a million values is a million GC-tracked objects while `[]Sample` is one. Optimise the allocation, not the call.

**Failure-mode view.** The typed-nil trap is the single most Go-specific interface bug you will encounter. It doesn't exist in Python — `None` is `None`, there is no "typed `None`." It silently inverts error-handling logic: `err != nil` evaluates `true` when the author's intent was "no error occurred." It passes code review because the code *reads* correct — `return err` after `err := helper()` looks unimpeachable. The bug is invisible until you know to look at the *return type of the helper*, and the mechanism is exactly the two-word layout: `{tab: set, data: nil}` is not the nil interface `{nil, nil}`.

## Real-world use cases

- **controller-runtime — `pkg/client/interfaces.go` (v0.24.1)** — <https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/client/interfaces.go> — the real in-production definitions reproduced above: `Reader` (2 methods), `Writer` (6, including `Apply`), `StatusClient` (1, returning `SubResourceWriter`), composed into `Client` with four extra methods, and `WithWatch = Client + Watch`. What it shows: interface composition, not inheritance, in the exact dependency your controller imports — and the practical payoff that a helper needing only reads can take `client.Reader` and be trivially fakeable.

- **controller-runtime — `pkg/client/fake` package doc (v0.24.1)** — <https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/client/fake/doc.go> — the maintainers' own ⚠️ limitations list: no error injection, partial sub-resource support, no OpenAPI validation, and `Generation`/`ResourceVersion` that "don't behave properly," so operations relying on them "will fail, or give false positives" — plus the opening advice that "when in doubt, it's almost always better not to use this package and instead use `envtest.Environment`." What it shows: fakes are a real testing tool with a documented ceiling; knowing where that ceiling is (and reaching for `envtest` past it) is the difference between a test suite that proves something and one that only passes.

- **client-go — fake client + informer example** — <https://github.com/kubernetes/client-go/blob/master/examples/fake-client/main_test.go> — the official example wires `fake.NewSimpleClientset` to a `SharedInformerFactory`, and its own comment admits: *"the fake client isn't designed to work with informer. It doesn't support resource version. It's encouraged to use a real client in an integration/E2E test if you need to test complex behavior with informer/controllers."* The test even has to wait for the watcher to start, because writes between the initial LIST and the watch would otherwise be lost. What it shows: even the reference fake has edges, and those edges are exactly the semantics (resourceVersion, watch ordering) that controller bugs live in.

- **The stdlib's optional-capability pattern — `io.Copy`** — `io.Copy` checks whether the source implements `io.WriterTo` or the destination implements `io.ReaderFrom` and delegates to them, skipping its 32 KB buffer entirely; `net/http` does the same with `http.Flusher` and `http.Hijacker`. What it shows: the way to add capability in Go is a *second small interface plus a comma-ok assertion*, not a wider first interface — the same move `client.WithWatch` makes, and the same move `NodeLister` makes in this lesson's worked example.

## Worked example

Refactor a cost tool so its data source is an interface — the exact shape controller-runtime uses for its client. One real implementation would hit Prometheus; a fake feeds fixed numbers in tests; a second, optional interface adds node discovery. The aggregation code depends only on the interfaces it declares.

**The consumer package declares what it needs and nothing more:**

```go
// package aggregate
package aggregate

import (
	"context"
	"fmt"
	"sort"
)

// CostSource is declared HERE, by the consumer. One method: exactly what Total calls.
type CostSource interface {
	HourlyCost(ctx context.Context, node string) (float64, error)
}

// NodeLister is a SECOND, separate consumer-side interface — an optional capability.
// A source that can do both satisfies both; one that can only price nodes satisfies
// only CostSource and is still perfectly usable with Total.
type NodeLister interface {
	Nodes(ctx context.Context) ([]string, error)
}

type Report struct {
	PerNode map[string]float64
	Total   float64
}

func Total(ctx context.Context, src CostSource, nodes []string) (Report, error) {
	rep := Report{PerNode: make(map[string]float64, len(nodes))}
	sorted := append([]string(nil), nodes...) // clone: never reorder the caller's slice
	sort.Strings(sorted)
	for _, n := range sorted {
		c, err := src.HourlyCost(ctx, n)
		if err != nil {
			return Report{}, fmt.Errorf("hourly cost for %s: %w", n, err) // %w: lesson 02
		}
		rep.PerNode[n] = c
		rep.Total += c
	}
	return rep, nil
}

// TotalAll uses the optional capability if the source has it.
func TotalAll(ctx context.Context, src CostSource) (Report, error) {
	lister, ok := src.(NodeLister) // comma-ok form: never panics
	if !ok {
		return Report{}, fmt.Errorf("source %T cannot list nodes", src)
	}
	nodes, err := lister.Nodes(ctx)
	if err != nil {
		return Report{}, fmt.Errorf("list nodes: %w", err)
	}
	return Total(ctx, src, nodes)
}
```

**The producer returns a concrete struct and never imports the consumer:**

```go
// package promsource
package promsource

import "context"

// PrometheusSource never imports aggregate and never mentions CostSource.
type PrometheusSource struct {
	addr  string
	rates map[string]float64
}

func New(addr string) *PrometheusSource { // return the concrete type, not an interface
	return &PrometheusSource{addr: addr, rates: map[string]float64{}}
}

func (s *PrometheusSource) HourlyCost(ctx context.Context, node string) (float64, error) {
	if err := ctx.Err(); err != nil { // honour cancellation — lesson 04
		return 0, err
	}
	return s.rates[node], nil
}

// Addr is a concrete-only method. Callers holding *PrometheusSource keep it;
// had New returned CostSource, they would have to type-assert to get it back.
func (s *PrometheusSource) Addr() string { return s.addr }
```

**The fake is the entire mocking framework:**

```go
// package aggregate (aggregate_test.go)
package aggregate

import (
	"context"
	"errors"
	"testing"
)

type fakeSource struct {
	costs map[string]float64
	fail  map[string]error
	calls []string // records interaction, when the test cares about it
}

func (f *fakeSource) HourlyCost(_ context.Context, node string) (float64, error) {
	f.calls = append(f.calls, node)
	if err, ok := f.fail[node]; ok {
		return 0, err
	}
	return f.costs[node], nil
}

func (f *fakeSource) Nodes(_ context.Context) ([]string, error) {
	return []string{"gpu-b", "gpu-a"}, nil // deliberately unsorted
}

// Compile-time guards: drift becomes a build failure, here, not at a call site.
var (
	_ CostSource = (*fakeSource)(nil)
	_ NodeLister = (*fakeSource)(nil)
)

func TestTotal(t *testing.T) {
	src := &fakeSource{costs: map[string]float64{"gpu-a": 3.06, "gpu-b": 3.06}}
	got, err := Total(context.Background(), src, []string{"gpu-b", "gpu-a"})
	if err != nil {
		t.Fatalf("Total: %v", err)
	}
	if got.Total != 6.12 {
		t.Errorf("total = %v, want 6.12", got.Total)
	}
	if len(got.PerNode) != 2 {
		t.Errorf("PerNode = %v, want 2 entries", got.PerNode)
	}
	t.Logf("call order observed by the fake: %v", src.calls)
}

func TestTotalPropagatesError(t *testing.T) {
	boom := errors.New("scrape timeout")
	src := &fakeSource{
		costs: map[string]float64{"gpu-a": 3.06},
		fail:  map[string]error{"gpu-b": boom},
	}
	_, err := Total(context.Background(), src, []string{"gpu-a", "gpu-b"})
	if !errors.Is(err, boom) { // lesson 02's chain, through this lesson's interface
		t.Fatalf("errors.Is(err, boom) = false; err = %v", err)
	}
	t.Logf("wrapped error: %v", err)
}

func TestTotalAllUsesOptionalCapability(t *testing.T) {
	src := &fakeSource{costs: map[string]float64{"gpu-a": 1, "gpu-b": 2}}
	rep, err := TotalAll(context.Background(), src)
	if err != nil {
		t.Fatalf("TotalAll: %v", err)
	}
	t.Logf("discovered nodes and totalled: %v total=%v", rep.PerNode, rep.Total)
}

// costOnly satisfies CostSource but NOT NodeLister — the assertion must fail cleanly.
type costOnly struct{}

func (costOnly) HourlyCost(context.Context, string) (float64, error) { return 0, nil }

func TestTotalAllRejectsSourceWithoutLister(t *testing.T) {
	_, err := TotalAll(context.Background(), costOnly{})
	if err == nil {
		t.Fatal("want error for a source that cannot list nodes")
	}
	t.Logf("got expected error: %v", err)
}
```

Captured run (go1.24.7):

```
$ go test -v ./aggregate/
=== RUN   TestTotal
    aggregate_test.go:45: call order observed by the fake: [gpu-a gpu-b]
--- PASS: TestTotal (0.00s)
=== RUN   TestTotalPropagatesError
    aggregate_test.go:58: wrapped error: hourly cost for gpu-b: scrape timeout
--- PASS: TestTotalPropagatesError (0.00s)
=== RUN   TestTotalAllUsesOptionalCapability
    aggregate_test.go:67: discovered nodes and totalled: map[gpu-a:1 gpu-b:2] total=3
--- PASS: TestTotalAllUsesOptionalCapability (0.00s)
=== RUN   TestTotalAllRejectsSourceWithoutLister
    aggregate_test.go:79: got expected error: source aggregate.costOnly cannot list nodes
--- PASS: TestTotalAllRejectsSourceWithoutLister (0.00s)
PASS
ok  	example.com/gpucost/aggregate	0.003s
```

Reading the output against the mechanisms:

- **`call order observed by the fake: [gpu-a gpu-b]`** — the fake recorded interaction, so the test proves `Total` sorts before iterating (deterministic output, [lesson 1](01-syntax-and-types.md)'s map-order discipline) without any mocking DSL. Recording is a struct field and an `append`.
- **`wrapped error: hourly cost for gpu-b: scrape timeout`** and `errors.Is(err, boom)` — the failure the fake injected travelled through the interface boundary, got wrapped with `%w`, and was still identifiable by identity at the top. Interfaces and lesson 02's error chain compose without either knowing about the other.
- **`source aggregate.costOnly cannot list nodes`** — the comma-ok assertion failed cleanly and produced a diagnosable error including the concrete type via `%T`. A bare `src.(NodeLister)` would have panicked here.
- **Nothing in `aggregate` mentions `promsource`**, and nothing in `promsource` mentions `aggregate`. Wiring happens in `main`. `Total` cannot tell whether it is talking to Prometheus or to a map — which is precisely what `fake.NewClientBuilder()` buys a reconciler test, one layer up.

## Practice

In [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md), make the data source a `CostSource` interface:

1. Declare `CostSource` in the package that *aggregates* costs (the consumer), with the minimal method set your aggregation loop actually calls. If it has more than two methods, you are declaring the producer's API, not the consumer's need — cut it down.
2. Write one real implementation (pull node/instance prices from your live source) that returns a concrete `*struct` from its constructor, and give it at least one method that is **not** in the interface — then verify a caller holding the concrete type can still use it.
3. Write one fake implementation, in the test file, backed by an in-memory map — no gomock, no testify mocks. Give it an injectable error map so a test can make a specific node fail, and a `calls []string` field so one test can assert interaction order.
4. Add `var _ CostSource = (*RealSource)(nil)` and the same for the fake. Then temporarily rename a method on the real source and confirm the build breaks at the guard, not somewhere else.
5. Add a second, optional interface (`NodeLister` or similar) and a function that uses it via the comma-ok type assertion, with a test for both the "has it" and "doesn't have it" paths.
6. Write one test that deliberately reproduces the typed-nil trap: a helper returning `*MyError` (nil), assigned to an `error`, asserted `!= nil`. Then fix the helper's signature to return `error` and watch the assertion flip. Keep the fixed version.

**Acceptance:** swapping the real implementation for the fake requires **no change to the aggregation code** — only the value passed in changes. If aggregation imports or names a concrete source type, the interface is in the wrong place; move it to the consumer. The compile-time guards exist for every implementation. `go vet ./...` and `go test ./...` are clean, and the optional-capability path is covered in both directions.

## Common pitfalls

1. **Interface pollution.** Symptom: a 12-method interface with one implementation, and every fake stubbing methods no test exercises. Mechanism: the interface was designed up front, producer-side, "in case someone needs it," so it describes the implementation rather than any caller's need. Fix: let interfaces be discovered — write concrete code, extract the interface in the consumer's package when a second implementation or a test double actually appears, and size it to the calls that exist.
2. **The typed-nil trap in a return path.** Symptom: `err != nil` is true and the error prints as `<nil>`, or a failure branch runs for an operation that succeeded. Mechanism: converting a nil `*T` into an interface sets the type word, and an interface is nil only when *both* words are zero. Fix: declare helpers as returning `error`, not `*MyError`; where you must hold a concrete pointer, convert explicitly (`if e != nil { return e }; return nil`). Enable `staticcheck` SA4023 and `golangci-lint`'s `nilnil`/`nilerr`.
3. **Assuming interface values are always comparable.** Symptom: `panic: runtime error: comparing uncomparable type main.Uncomparable` in production, on a code path that reviewed and tested fine. Mechanism: `==` on interfaces compares dynamic type then dynamic value; if the dynamic type is a slice, map, or func (or a struct containing one), the runtime cannot compare and panics rather than returning false. Fix: don't compare interface values whose dynamic type you don't control; compare an identifying field instead. Also remember `any(int64(1)) != any(int(1))` — different dynamic types are never equal.
4. **Confusing "accept interfaces, return structs" with "never return an interface."** Symptom: constructors returning `error`-like abstractions get flagged in review for no reason. Mechanism: the rule targets *premature* interface returns that hide a concrete API the caller needs, not every interface-typed return. Fix: return concrete types by default; return an interface when it's `error`, when a factory genuinely selects among implementations, or when the concrete type is intentionally unexported.
5. **Expecting embedding to behave like inheritance.** Symptom: a method on the outer type is "ignored" when the inner type calls it. Mechanism: promotion is compiler-rewritten delegation — `Node.Where()` becomes `Node.Base.Where()`, and `Base` has no reference to `Node`. There is no virtual dispatch. Fix: if you need the outer behaviour from inner code, pass it in explicitly (a field holding an interface, or a function parameter) — that's the composition Go actually offers.
6. **Ambiguous promotion silently removing a method from the method set.** Symptom: `ambiguous selector` plus a second error saying your type no longer implements an interface it obviously "has". Mechanism: `x.f` resolves to the shallowest depth, and if there isn't exactly one `f` at that depth the selector is illegal — so the method isn't in the method set at all. Fix: declare the method explicitly on the outer type (depth 0 wins), or qualify the call.
7. **Trusting a fake for behaviour it doesn't implement.** Symptom: a green test suite, then optimistic-concurrency conflicts or missed watch events in production. Mechanism: controller-runtime's fake client documents that `Generation`/`ResourceVersion` "don't behave properly," and client-go's fake has no resourceVersion at all. Fix: use fakes for your logic, `envtest` (real apiserver, no kubelet) for anything touching resourceVersion, conflicts, validation, or watch ordering — lessons 5 and 9.
8. **Embedding an interface in a struct and shipping it.** Symptom: a nil-pointer panic from a method you never implemented, the first time a new code path calls it. Mechanism: the embedded interface field is nil, so any non-overridden method dispatches through a nil interface. Fix: fine as a deliberate test-only shim for a large third-party interface; in production code use an explicit named field and write the delegating methods.

## Self-check

**(a) Why prefer defining interfaces on the consumer, not the producer?**
**Answer:** The consumer knows exactly which methods it calls, so the interface stays small and honest by construction, and each caller can declare its own minimal view without anyone agreeing on one "official" interface. The producer returns a concrete struct and takes **no dependency on the interface** — it doesn't import it and doesn't know it exists — which is only possible because Go's satisfaction is structural and implicit. Three payoffs: you can retrofit an interface (and a fake) around a third-party type you don't own; the signature documents the true dependency surface (`client.Reader` in a parameter is a machine-checked promise not to mutate); and producer-side interfaces drift into fat speculative blobs (interface pollution) because nothing constrains them to actual use.

**(b) How do you fake an external dependency with zero mocking library?**
**Answer:** Depend on a small interface declared in your own package, then write a plain struct whose methods return canned data — usually from a map field — and pass it where the real implementation goes. Implicit satisfaction means no registration and no framework: the fake just needs the right method set. Add `var _ CostSource = (*fakeSource)(nil)` so drift is a build error. Add a `calls []string` field if you need interaction assertions, and a `fail map[string]error` field to inject failures. This is exactly what controller-runtime's `fake.Client` is — an in-memory implementation of the `client.Client` interface — and exactly what its package doc warns about: no error injection, no working `ResourceVersion`/`Generation`, so anything depending on those needs `envtest` instead.

**(c) Why can an interface holding a nil pointer not equal nil?**
**Answer:** An interface value is two words — `{tab *itab, data unsafe.Pointer}` for a non-empty interface, `{_type, data}` for `any` — and it equals `nil` only when **both** words are zero. Assigning a typed nil pointer such as `(*CostError)(nil)` sets the type word (the itab for the `(error, *CostError)` pair) and leaves data nil, so `err != nil` is `true` while the pointer inside is nil. Dumping the words shows it directly: `err==nil:false tab=true data=false`. It surfaces most often as a helper returning `*MyError` whose result is returned through an `error`. Fix: give the helper the `error` return type and `return nil` literally, or convert explicitly with `if e != nil { return e }; return nil`.

**(d) Why does comparing two interface values sometimes panic instead of just returning `false`?**
**Answer:** `==` on interfaces first compares dynamic types; if they differ, the result is `false` and nothing else happens. If they're identical, it compares the dynamic **values** — and if that type is uncomparable (slice, map, func, or a struct/array containing one), there is no comparison operation to perform, so the runtime panics: `comparing uncomparable type main.Uncomparable`. The compiler can't reject it because `any == any` is legal in the type system; the offending type only shows up at run time. It therefore passes review and passes tests, and panics the first time that path runs with the "wrong" concrete type. Same hazard applies to using such values as map keys and to structs with interface fields.

**(e) Why is a type switch cheap, and what does a method call through an interface actually cost?**
**Answer:** A type switch or assertion compares the interface's type word against known `*_type` pointers — a pointer comparison, not a reflective walk — and the Go 1.24 runtime additionally keeps lazily-filled caches (`runtime.typeAssert`'s `TypeAssertCache`, `runtime.interfaceSwitch`'s `InterfaceSwitchCache`) so repeated assertions and switches on the same types skip the runtime call entirely. Measured here: ~0.8–1.1 ns. A method call through an interface loads the itab, loads `Fun[i]` (the index is fixed at compile time), and makes an indirect call. The jump itself is cheap; what costs is that the compiler **cannot inline** an unknown target. Measured: identical to a concrete call (~2.6–3.1 ns) when the method body does real work, but ~0.70 → ~3.2 ns per element (≈4.4×) for a trivially inlinable accessor over a 1,000-element slice. The bigger cost is usually **boxing**: storing a non-pointer-shaped value in an interface heap-allocates (8 B, ~15 ns measured), so `[]Valuer` of a million values is a million GC objects while `[]Sample` is one.

**(f) What exactly does struct embedding do, and how is it different from interface embedding?**
**Answer:** Struct embedding declares a field with no name; the type name becomes the field name, and the inner type's **fields and methods are promoted** — `n.Where()` is compiler shorthand for `n.Base.Where()`. It is delegation, not inheritance: no `super`, no virtual dispatch, and the inner type can never call the outer type's version of a method. Resolution uses depth (0 for declared on the type, +1 per embedded hop), always picks the shallowest, and is a **compile error** if there isn't exactly one at that depth — which also removes the method from the type's method set, so the type stops satisfying interfaces. Method-set detail: embedding `T` gives `S` and `*S` the `T`-receiver methods and `*S` the `*T`-receiver ones; embedding `*T` gives both to both. Interface embedding is simpler: interfaces have no fields, so it just **unions method sets**, and since Go 1.14 overlapping methods are legal when the signatures match exactly. Embedding an *interface* in a struct is a third thing — it satisfies the whole interface while implementing a few methods, and panics on every method you didn't override.

## Connections & what's next

Interfaces are the hinge the rest of this module turns on. [Lesson 01](01-syntax-and-types.md)'s method-set and addressability rules are what decide whether `T` or `*T` satisfies anything, and its escape analysis is the same boxing allocation measured here. [Lesson 02](02-error-handling.md)'s error-handling discipline is what makes the typed-nil trap dangerous in the first place — a `return err` that reads correct but isn't — and `error` itself is the one-method interface all of this generalises from. [Lesson 04 (Concurrency & context)](04-concurrency-and-context.md) leans on this lesson immediately: bounding concurrent work with `errgroup` and faking the things you fan out to (an HTTP client, a `CostSource`) both depend on small consumer-defined interfaces, and a fake is what lets you test cancellation without a real endpoint. [Lesson 05 (Testing)](05-testing.md) turns the fake-vs-real distinction into a full strategy — table-driven tests against fakes, plus `envtest` for the resourceVersion/watch behaviour fakes explicitly cannot provide. [Lesson 09 (Controller primer)](09-controller-primer.md) is where `client.Client`, `manager.Manager`, and `reconcile.Reconciler` stop being examples and become the machinery you write against.

Next: **[04 · Concurrency and context](04-concurrency-and-context.md)** — the thread that starts here (interfaces you can fake and bound) is what makes bounded, testable, cancellable fan-out possible at all.

## References & further reading

**Primary sources**
- The Go Programming Language Specification — <https://go.dev/ref/spec> — the exact rules used above: "Method sets" (`T` vs `*T`), "Selectors" (promotion depth and the "not exactly one at shallowest depth is illegal" rule), "Struct types" (embedded fields and promoted method sets), and "Comparison operators" (interface equality and the uncomparable-dynamic-type panic).
- Go source, `src/runtime/runtime2.go` and `src/internal/abi/iface.go` — <https://github.com/golang/go/blob/master/src/internal/abi/iface.go> — the actual `iface`/`eface` two-word layout and the `ITab{Inter, Type, Hash, Fun}` structure whose `Fun` array holds the concrete method addresses.
- Go source, `src/runtime/iface.go` — <https://github.com/golang/go/blob/master/src/runtime/iface.go> — `getitab` (lock-free lookup in the global `itabTable`, `persistentalloc` on miss), `typeAssert`/`interfaceSwitch` and their lazily-built caches.
- Go source, `cmd/compile/internal/types/type.go` — <https://github.com/golang/go/blob/master/src/cmd/compile/internal/types/type.go> — `IsDirectIface`: which types are stored directly in the interface's data word (pointers, chans, maps, funcs, `unsafe.Pointer`, single-field wrappers) and which therefore require a boxing allocation.
- Effective Go — Interfaces — <https://go.dev/doc/effective_go#interfaces> — canonical source on interface conventions, embedding, and the `-er` naming convention. Dated in places, still authoritative for vocabulary.
- controller-runtime, `pkg/client/interfaces.go` (v0.24.1) — <https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/client/interfaces.go> — the real `Reader`/`Writer`/`StatusClient`/`SubResourceWriter` definitions composed into `Client`, plus `WithWatch = Client + Watch`.
- controller-runtime, `pkg/client/fake` package doc (v0.24.1) — <https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/client/fake/doc.go> — the maintainers' limitations list (no error injection, partial sub-resources, no OpenAPI validation, broken `Generation`/`ResourceVersion`) and the advice to prefer `envtest.Environment`.
- `pkg.go.dev/sigs.k8s.io/controller-runtime` — <https://pkg.go.dev/sigs.k8s.io/controller-runtime> — the package-doc surface for `client.Client`, `manager.Manager`, and `reconcile.Reconciler`; useful for the composed method sets your controller will call.

**Real-world engineering**
- client-go — fake client + informer example — <https://github.com/kubernetes/client-go/blob/master/examples/fake-client/main_test.go> — what it shows: the official fake-client pattern *and* its documented limitation ("the fake client isn't designed to work with informer. It doesn't support resource version…"), which tells you exactly when to reach for `envtest` instead.
- uber-go/guide — `style.md` — <https://github.com/uber-go/guide/blob/master/style.md> — what it shows: the compile-time compliance guard `var _ http.Handler = (*Handler)(nil)` with its rationale, and the warning against embedding types in public structs ("these embedded types leak implementation details, inhibit type evolution, and obscure documentation") with the delegate-explicitly alternative.

**Deeper dives**
- Learning Go, 2nd ed. (Jon Bodner) — "Types, Methods, and Interfaces" — <https://www.oreilly.com/library/view/learning-go-2nd/9781098139285/> — the clearest print treatment of implicit satisfaction, accept-interfaces/return-structs, embedding, and the type-value pair behind the nil trap. Deep read.
- 100 Go Mistakes and How to Avoid Them (Teiva Harsanyi) — interface items — <https://100go.co> — the "interface on the producer side" and "interface pollution" entries are short, blunt corrections to fat-interface habits ported from Python/Java. Skim.
- Go FAQ — "Why is my nil error value not equal to nil?" — <https://go.dev/doc/faq#nil_error> — the canonical short statement of the typed-nil trap, straight from the language FAQ; useful to link in a code review when explaining the bug to someone else.
