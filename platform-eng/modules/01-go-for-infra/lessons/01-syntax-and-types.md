---
lesson: "01.1"
title: "Syntax and Types"
module: "01"
concept: "Syntax and Types"
status: not-started
est_time: "7h"
prev: null
next: "02-error-handling.md"
artifacts: []
sources: 8
---

# 01.1 · Syntax and Types

> **Concept.** Type idiomatic Go fluently — receivers, slice/map aliasing, zero values, `struct{}` sets, field layout, and escape analysis — with no OOP baggage.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Where this fits

This is lesson one of the module — there's no prior lesson to recap, so start from the module's own framing: **Go for infrastructure engineers** gates every other module in this program, and it bends toward one thing — writing controllers. Every controller you'll touch (client-go, controller-runtime, kubebuilder scaffolding) is a wall of structs, pointer receivers, and slices of typed objects, so before anything about errors, interfaces, or concurrency can click, the type system has to become muscle memory. This lesson builds that muscle memory: value vs. pointer semantics, slice/map aliasing, zero values, and how Go lays structs out in memory. Once that's automatic, [Error Handling](02-error-handling.md) can build the reconcile loop's control-flow discipline on top of it without re-litigating "wait, is this a copy or not?" at every line.

## Why this matters

Every controller you touch — client-go, controller-runtime, kubebuilder scaffolding — is a wall of structs, pointer receivers, and slices of typed objects. If slice aliasing or nil maps surprise you, your reconcile loop mutates shared cache objects or panics on a nil write in production. Interviewers at GPU shops will hand you a `func (r *Reconciler) Reconcile(...)` and expect you to know instantly why the receiver is a pointer. This lesson makes the type system muscle memory so the k8s libraries read as obvious — and, at GPU-fleet scale, a types bug isn't cosmetic: a slice-aliasing bug or an `omitempty` mismatch in a cost exporter doesn't crash anything, it silently corrupts a number that becomes a line item on someone's GPU bill.

Company signals from the module README make the stakes concrete: CoreWeave hires for "backend services and APIs (gRPC/REST) in Go to interact with Kubernetes"; NVIDIA for "scalable Go programs deployed to Kubernetes that manage large datacenters"; Cisco for "controllers, operators, CRDs, and Golang services." None of that is reachable if value-vs-pointer semantics are still a source of surprise.

## What's new here (calibration)

Per the module README's skip-list, you already program (Python) and know systems/k8s, so this lesson skips: programming 101, pointers-as-concept (you know what a pointer *is* — this is about Go's specific rules for using one), IDE/hello-world tours, OOP-in-Go (there is no OOP-in-Go), and LeetCode/DSA grind.

What's genuinely new depth here, beyond "Go's spelling of what you already know":

- **Escape analysis** — the compiler decision of stack vs. heap allocation, and how to inspect it. This is invisible in Python (everything is heap-allocated and refcounted/GC'd) but directly shapes GC pressure in a hot Go path.
- **Struct field alignment and padding** — Go does not reorder your struct fields; declaration order determines memory layout, padding, and cache-line packing. This has no analogue in Python's object model at all.
- **Generics, calibrated** — Go 1.18+ type parameters, and specifically *when idiomatic Go still prefers an interface over a generic* — a judgment call, not a syntax lesson.
- **Comparability as a compile-time constraint** — why `==` on a struct sometimes doesn't compile, which is a genuinely Go-flavored trap for someone coming from Python's universal (if sometimes surprising) `==`.

## Core concepts

### From Python to Go: the mental shift

Coming from Python, four things bite:

- **No classes, no `self`, no inheritance.** You attach behavior to types via *methods with receivers*. Composition (embedding) replaces subclassing.
- **Assignment copies.** `b := a` on a struct copies the whole struct. Python rebinds a reference; Go copies the value. This is why you pass `*T` when you want mutation to stick.
- **`nil` is not `None`.** A nil map is *readable* (returns zero values) but *panics on write*. A nil slice is fully usable — you can `append` to it. Python's `None` does neither.
- **Static types, zero values, no `__init__`.** Every declared variable is immediately usable at its zero value. There is no "uninitialized" — but the zero value of a map or pointer is `nil`, which is a trap, not a convenience.

The mental shift: stop thinking "object references" and start thinking "values, and pointers to values." Value vs. pointer isn't just a memory-layout detail — it's an **API contract**. A value parameter tells a caller "I will not mutate what you gave me, you're safe passing a copy of your record." A pointer parameter tells a caller "I might mutate this, and if you shared this pointer with anyone else, they'll see it too." Getting that contract backwards in a struct passed through layers (an aggregator that hands a `*Record` to an exporter, which hands it to a CRD-status writer) creates a bug class where an inner layer's mutation surprises a caller who assumed it held an immutable copy.

### Zero values

Declaring a variable always yields a usable zero: `0`, `false`, `""`, `nil` for pointers/slices/maps/channels/interfaces/funcs, and *recursively zeroed fields* for structs. No constructors run.

```go
var c Cost           // struct: all fields zeroed
var s []string       // nil slice — len 0, safe to append
var m map[string]int // nil map — safe to READ, panics on WRITE
```

### Slices alias a backing array

A slice is a 3-word header `{ptr, len, cap}`. Copying the header (pass to a func, assign) shares the *same backing array*. Mutating an element through any alias is visible to all.

```go
func zeroOut(xs []float64) {
    for i := range xs {
        xs[i] = 0 // caller SEES this — same backing array
    }
}
```

But `append` may or may not be visible: if it fits in `cap`, it writes in place (caller's array mutated, caller's `len` unchanged); if it reallocates, the callee gets a fresh array and the caller sees nothing. **Never rely on append's side effects across a function boundary — return the slice.** This is the single most common slice bug, and it isn't a rule you can shortcut by "just remembering the capacity" — production code hits it because capacity is rarely visible at the call site.

```go
xs = append(xs, v) // idiom: always reassign the result
```

A subtler version of the same trap: slicing a struct slice (`recent := records[len(records)-10:]`) shares the backing array of the *whole* slice, including elements outside the visible window — a later `append` to `recent` can silently overwrite records the original slice still references. When you need an independent copy, `copy()` into a freshly made slice explicitly.

**`nil` vs empty slice.** `var s []T` (nil) and `s := []T{}` (empty, non-nil) behave identically for `len`, `range`, `append`. They differ in `s == nil` and in JSON: a nil slice marshals to `null`, an empty slice to `[]`. For API responses and exporter output, prefer initializing to empty when "no items" should serialize as `[]`.

### Maps

The zero value is `nil`. Reading a nil map is fine; writing panics. Always `make` before writing:

```go
m := make(map[string]float64)
m["ns/prod"] += 4.10          // ok
v, ok := m["ns/dev"]          // comma-ok: v=0, ok=false — distinguishes "absent" from "zero"
```

Map iteration order is **randomized** by design — never assume order; sort keys explicitly for stable table output. Maps are reference-like: passing a map to a func lets the callee mutate the caller's map (no copy of contents).

### Value vs pointer receivers

A method set is declared on a type:

```go
func (c Cost) Total() float64           { return c.GPUHours * c.Rate } // value: reads a COPY
func (c *Cost) ApplyDiscount(p float64) { c.Rate *= (1 - p) }          // pointer: mutates caller
```

Rules that actually matter:

- A **value receiver operates on a copy** — mutations are lost. If your method mutates fields, it *must* be a pointer receiver, or the change silently vanishes.
- Use pointer receivers to **avoid copying large structs** and when any method needs to mutate.
- **Be consistent per type**: if one method needs a pointer receiver, give them all pointer receivers. Mixing confuses the method set and interface satisfaction.
- Only **addressable** values can call pointer-receiver methods automatically. `map[k]Struct` values are not addressable — `m["x"].Mutate()` won't compile. Store `*Struct` in the map, or read-modify-write the whole value back.

**Interface satisfaction & the receiver trap.** If methods are on `*T`, then only `*T` satisfies the interface, not `T`. Passing a `T` value where the interface is expected fails to compile. This surprises Pythonistas constantly.

### `struct{}` as a set

Go has no set type. Idiom: `map[T]struct{}` — the empty struct occupies **zero bytes**, signaling "key presence only."

```go
seen := map[string]struct{}{}
seen["gpu-node-1"] = struct{}{}
if _, ok := seen["gpu-node-1"]; ok { /* present */ }
```

### Structs, tags, and construction

No constructors; use struct literals with field names. Tags drive JSON (de)serialization:

```go
type GPUCost struct {
    Namespace string  `json:"namespace"`
    Label     string  `json:"label,omitempty"` // omit if empty when marshaling
    GPUHours  float64 `json:"gpu_hours"`
    USD       float64 `json:"usd"`
}
c := GPUCost{Namespace: "prod", GPUHours: 12, USD: 41.0}
```

**Comparability.** Structs are comparable with `==` only if *every field* is comparable. A struct with a slice, map, or func field cannot use `==` — it's a compile error, not a runtime surprise, which is disorienting coming from Python's duck-typed equality (where `==` always "works," just sometimes wrongly). Alternatives: `reflect.DeepEqual` (slow, uses reflection, fine in tests) or an explicit field-by-field `Equal` method (fast, explicit, the idiomatic choice for hot paths).

### Struct field alignment and padding

Go does **not** reorder your struct's fields — they're laid out in memory in declared order, and the compiler inserts padding so each field starts at an address matching its own alignment requirement. Mixing small fields (`bool`, `int8`) with large ones (`int64`, pointers, `float64`) in a bad order wastes bytes — up to 7 padding bytes per boundary on a 64-bit platform:

```go
type Bad struct { // 24 bytes: bool(1) + pad(7) + float64(8) + bool(1) + pad(7)
    Dirty   bool
    USD     float64
    Stale   bool
}

type Good struct { // 16 bytes: float64(8) + bool(1) + bool(1) + pad(6)
    USD   float64
    Dirty bool
    Stale bool
}
```

`unsafe.Sizeof` shows the real size, and the `fieldalignment` analyzer (part of `golang.org/x/tools/go/analysis/passes/fieldalignment`, also runnable via `go vet -vettool=$(which fieldalignment)`) flags structs with reorderable savings. Rule of thumb: order fields largest-alignment-to-smallest. This isn't premature optimization for a one-off struct — it's real when a `GPUUsageRecord` gets scraped every 15 seconds across tens of thousands of GPUs: avoidable padding is measurable exporter RSS and GC pressure at that scale, not a rounding error.

### Escape analysis

The compiler decides, per allocation, whether a value can live on the stack (cheap, freed automatically on return) or must "escape" to the heap (GC-managed, more expensive). You can see its decisions directly:

```sh
go build -gcflags="-m" ./...
```

This prints `escapes to heap` or `does not escape` for each allocation. Two patterns force an escape that a Python developer wouldn't think to check for, because Python has no stack-allocation concept at all:

- **Returning a pointer to a local.** `return &localStruct` forces `localStruct` onto the heap — the compiler can't let it die with the stack frame if a caller now holds its address.
- **Boxing a value into an interface.** Assigning a concrete value to an `interface{}`/`any` (or passing it where an interface parameter is expected) generally forces a heap allocation to hold the value behind the interface's internal pointer.

This is the mechanical, performance-level reason behind the idiom "accept interfaces, return structs" that lesson 3 teaches as an API-design rule — returning a concrete struct by value lets the compiler keep it on the stack when possible; returning it wrapped in an interface usually can't.

### Generics, briefly

Go 1.18+ added type parameters:

```go
type Number interface{ ~int | ~int64 | ~float64 }

func SumBy[K comparable, V Number](xs []V, key func(V) K) map[K]V {
    out := make(map[K]V)
    for _, x := range xs {
        out[key(x)] += x
    }
    return out
}
```

This is genuinely useful when you want **one algorithm over many data shapes** — summing, filtering, or comparing across numeric or struct types without duplicating the function per type or falling back to `interface{}` and type assertions. But the calibration matters: idiomatic Go still prefers a small **interface** over a type parameter when the abstraction is about *behavior* (a method a type implements) rather than *data shape* (a type parameter constraining which concrete types are allowed in). If you find yourself writing a generic function whose body never actually calls a method on `T` — it just stores and returns `T`s — that's the generics-appropriate case. If the body calls `t.Reconcile()` or `t.String()`, that's an interface, not a type parameter.

### Errors are values

Functions return `(T, error)`; check `if err != nil`. No exceptions for ordinary failures — [lesson 2](02-error-handling.md) goes deep on this. `panic` is for programmer bugs, not control flow.

## Perspectives

**Developer / API-design view.** Value vs. pointer types are an API contract, not just a memory choice — a value type says "immutable record, take a copy safely," a pointer type says "identity that can be mutated." Getting it backwards in a shared struct passed through layers (an aggregator hands a value to an exporter, which hands it to a CRD-status writer) creates a bug class where an inner layer's mutation surprises a caller who assumed a copy. This is a code-review smell to develop an eye for: does this function's signature honestly describe whether it can change what you gave it?

**Operator / production view.** Nil-map-write panics and slice-aliasing bugs are exactly the shape of real on-call pages — a panic in a hot reconcile path from an unmade map, or a metrics exporter silently reporting the identical number for two namespaces because two `[]Sample` values shared a backing array after a struct-slice copy. Neither of these looks like a "types bug" from the outside; they look like "the exporter is wrong" or "the controller crashed," and the root cause is a slice header quietly aliasing memory three call frames away.

**Hardware / memory view.** Struct field layout isn't free — Go doesn't reorder fields, so field order determines padding and cache-line packing. At GPU-fleet scale (a `GPUUsageRecord` scraped every 15 seconds across tens of thousands of GPUs), avoidable padding is measurable exporter RSS and GC pressure, not academic trivia. Escape analysis (`go build -gcflags="-m"`) decides stack vs. heap; returning a pointer to a local, or boxing a value into an interface, forces an escape that a Python developer wouldn't think to check because Python has no equivalent decision to make.

**Economics view.** A cost/efficiency exporter's entire job is to be numerically correct at scale — a slice-aliasing bug or a `nil`-vs-`omitempty` mismatch doesn't crash anything, it silently corrupts a number that becomes a line item on someone's GPU bill. A types bug here is a finance bug: nobody pages on it, but it's wrong for months, exactly like the year-long Allegro bug below.

## Real-world use cases

- **Allegro Tech — "Golang slices gotcha"** — <https://blog.allegro.tech/2017/07/golang-slices-gotcha.html> — a real production bug in `allegro/marathon-consul` that lived silently for roughly a year: two service ports shared a backing array via `append`, so both ended up with identical Consul tags, making the service undiscoverable by its own clients. This is directly the failure mode behind this lesson's "never rely on append's side effects across a function boundary" rule.
- **golang/go issue #66093 — "append() grows backing array to odd-sized, unexpected capacity"** — <https://github.com/golang/go/issues/66093> — a real, currently-open Go core issue: `append`'s growth behavior for structs larger than 128 bytes containing pointers changed unexpectedly between point releases (a 32-element slice grew to capacity 37, not 64, going from Go 1.21.7 to 1.22.0). Evidence that "slice internals are simple, I've got the mental model" is a trap even for engineers who wrote the mental model — capacity growth is an implementation detail, not a contract.

## Worked example

Aggregate a GPU cost export by namespace and print a stable sorted table — the shape of the deliverable.

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"sort"
	"text/tabwriter"
)

type Record struct {
	Namespace string  `json:"namespace"`
	GPUHours  float64 `json:"gpu_hours"`
	USD       float64 `json:"usd"`
}

// aggregate sums cost per namespace. Returns an empty (non-nil) map for no input.
func aggregate(recs []Record) map[string]float64 {
	out := make(map[string]float64) // make BEFORE writing — nil map would panic
	for _, r := range recs {
		out[r.Namespace] += r.USD // missing key reads as 0.0, then adds
	}
	return out
}

func main() {
	var recs []Record
	if err := json.NewDecoder(os.Stdin).Decode(&recs); err != nil {
		fmt.Fprintln(os.Stderr, "decode:", err)
		os.Exit(1)
	}

	totals := aggregate(recs)

	// Map order is random — sort keys for deterministic output.
	keys := make([]string, 0, len(totals))
	for k := range totals {
		keys = append(keys, k) // reassign append result — idiom
	}
	sort.Strings(keys)

	w := tabwriter.NewWriter(os.Stdout, 0, 4, 2, ' ', 0)
	fmt.Fprintln(w, "NAMESPACE\tUSD")
	for _, k := range keys {
		fmt.Fprintf(w, "%s\t%.2f\n", k, totals[k])
	}
	w.Flush()
}
```

`out := make(map[string]float64)` is the whole zero-value/nil-map lesson in one line — skip it and every `out[r.Namespace] += r.USD` panics on the first write. `keys = append(keys, k)` is the whole append-reassignment lesson in one line — the idiom exists precisely because you cannot know, at the call site, whether that `append` grew in place or reallocated.

## Practice

Port an existing Python GPU-cost script to Go as the first cut of [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md).

- Read a JSON cost export (array of records: namespace, a label/team field, gpu_hours, usd) from a file or stdin.
- Decode into a `[]Record` of typed structs with JSON tags.
- Aggregate cost two ways: by `namespace` and by `label`, using `map[string]float64`.
- Print each aggregation as a table sorted by key (use `sort` + `text/tabwriter`).
- Use `(T, error)` returns for decode/parse; check every `err`.
- Once the `Record` struct is stable, run `go build -gcflags="-m" ./...` on the package and note which allocations escape to the heap; then check `unsafe.Sizeof(Record{})` and reorder fields largest-alignment-to-smallest if there's free padding to reclaim.

**Acceptance:** `go build ./...` succeeds; `go vet ./...` is clean; running against a sample export produces correct per-namespace and per-label totals in deterministic sorted order. No nil-map panics, no reliance on append side effects.

## Common pitfalls

1. **Assuming append mutation is always/never visible.** It's neither — visibility depends on whether the new length fits within the existing capacity. The fix isn't a memorizable rule about specific capacities; it's behavioral discipline: always reassign `xs = append(xs, v)` and never assume the caller sees (or doesn't see) an append.
2. **`omitempty` treating a real zero as "absent."** A USD `float64` field tagged `json:"usd,omitempty"` silently disappears from the JSON when the actual cost is exactly `$0.00` — indistinguishable on the wire from "no data was collected." In a billing exporter this is a genuine correctness bug, not a cosmetic one: a consumer can't tell "$0" from "missing."
3. **Comparing structs containing slices/maps with `==`.** This is a compile error, not a runtime surprise — confusing if you're used to Python's duck-typed equality, which always "just works" (if sometimes wrongly). Use `reflect.DeepEqual` or a hand-written `Equal` method instead.
4. **Treating nil slice and empty slice as always interchangeable.** True for `len`, `range`, and `append`; false for `== nil` and for JSON marshaling (`null` vs. `[]`). An API consumer that branches on `null` vs. `[]` will see a real difference you didn't intend.
5. **Forgetting `:=` inside `if`/`for` creates a new, shadowed variable.** `if err := f(); err != nil { ... }` inside a block that already has an outer `err` creates a second, block-scoped `err` — a common source of "why didn't my outer variable change" bugs for engineers used to Python's function-scoped assignment, where there's no equivalent shadowing trap.

## Self-check

- **When does passing a slice let the callee mutate the caller's backing array, and when not?**
  **Answer:** Mutating *existing elements* by index (`xs[i] = v`) is always visible to the caller — both share one backing array. `append` is visible only if the new length still fits within the original `cap` (in-place write); if `append` exceeds `cap` it allocates a new array and the callee's changes are invisible. Because you can't predict reallocation at the call site, always return the appended slice and reassign.

- **Value vs pointer receiver — what breaks on a mutating method if you pick wrong?**
  **Answer:** A value receiver operates on a copy, so any field mutation is discarded when the method returns — the caller's value is unchanged, silently. A mutating method must use a pointer receiver. Secondary breakage: if methods are on `*T`, only `*T` (not `T`) satisfies interfaces, and `T` values stored in a map aren't addressable so their pointer methods can't be called.

- **What is the zero value of a map, and what happens writing to a nil map?**
  **Answer:** The zero value of a map is `nil`. Reading from a nil map is safe and returns the element type's zero value (with `ok == false`), but *writing* to a nil map panics at runtime. You must `make(map[K]V)` (or a literal `map[K]V{}`) before assigning keys.

- **Why can two structs fail to compile with `==`, and what do you use instead?**
  **Answer:** A struct is comparable with `==` only if every field is comparable — slices, maps, and funcs are not comparable, so a struct containing any of them fails to compile on `==`. Use `reflect.DeepEqual` (works via reflection, slower, fine for tests) or write an explicit `Equal(other T) bool` method that compares fields individually (fast, explicit, idiomatic for hot paths).

- **What does `go build -gcflags="-m"` tell you, and name one pattern that forces a heap escape.**
  **Answer:** It prints the compiler's escape analysis decision for each allocation — `escapes to heap` or `does not escape`. Returning a pointer to a local variable (`return &localStruct`) forces that local onto the heap, because the caller now holds an address that must outlive the function's stack frame. Boxing a concrete value into an `interface{}`/`any` is the other common trigger.

## Connections & what's next

Value/pointer semantics and struct layout aren't a one-off topic — they're load-bearing for the rest of the module. [Lesson 3 (Interfaces & Composition)](03-interfaces-and-composition.md) leans directly on the receiver rules here ("methods on `*T` mean only `*T` satisfies the interface") and on the escape-analysis reasoning behind "accept interfaces, return structs." [Lesson 4 (Concurrency & Context)](04-concurrency-and-context.md) depends on knowing precisely when a value is shared (and therefore needs synchronization) versus copied (and therefore safe to hand to a goroutine without a lock). [Lesson 6 (Modules & Layout)](06-modules-and-layout.md) revisits comparability and versioning gotchas at package boundaries. And every controller you read in [Lesson 9](09-controller-primer.md) is, mechanically, exactly the struct/slice/map/pointer vocabulary built here, just wearing a Kubernetes client-go costume.

Next: [02 · Error Handling](02-error-handling.md) — now that values, pointers, and zero values are automatic, the next lesson builds the reconcile loop's real skill: classifying an error as skip, retry, or terminal, and making that classification survive being wrapped three call layers deep.

## References & further reading

**Primary sources**
- The Go Programming Language Specification — <https://go.dev/ref/spec> — canonical reference for zero values, comparability, method sets, and slice/map semantics; read for the exact rules, not the tutorial version.
- `golang.org/x/tools/go/analysis/passes/fieldalignment` — <https://github.com/golang/tools/tree/master/go/analysis/passes/fieldalignment> — the analyzer that flags reorderable struct-field savings; read for how to actually run it against your own structs.
- A Tour of Go — <https://go.dev/tour> — interactive syntax/types/methods walkthrough; skim Basics → Methods/Interfaces in an afternoon as an experienced dev, it's the fastest way to internalize receivers and slices hands-on.
- Learning Go, 2nd ed. (Jon Bodner), ch. 1–6 — <https://www.oreilly.com/library/view/learning-go-2nd/9781098139285/> — types, composite types, functions, pointers, methods; Bodner is explicit about *why* Go differs from other languages, and the slice-aliasing and pointer-receiver sections are the sharpest treatment in print.

**Real-world engineering blogs**
- Allegro Tech — "Golang slices gotcha" — <https://blog.allegro.tech/2017/07/golang-slices-gotcha.html> — a year-long production bug from slice aliasing via `append`; what it shows: the append-visibility rule isn't academic, it silently broke service discovery in production.
- golang/go issue #66093 — <https://github.com/golang/go/issues/66093> — an open core-Go issue where `append`'s growth behavior surprised engineers across a point release; what it shows: slice capacity growth is an implementation detail you cannot rely on, even release-to-release.

**Deeper dives**
- The Go Blog — "Arrays, slices (and strings): The mechanics of 'append'" — <https://go.dev/blog/slices> — the canonical deep explanation of slice headers, capacity growth, and why aliasing happens.
- Go by Example — <https://gobyexample.com> — copy-paste-ready snippets for maps, slices, structs, sorting; keep it open as a reference while porting the script, ideal when you know the concept and just need the idiom.
