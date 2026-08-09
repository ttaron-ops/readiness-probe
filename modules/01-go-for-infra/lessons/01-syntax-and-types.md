---
lesson: "01.1"
title: "Syntax and Types"
module: "01"
concept: "Syntax and Types"
status: not-started
est_time: "6h"
artifacts: []
---

# 01.1 · Syntax and Types

> **Concept.** Type idiomatic Go fluently — receivers, slice/map aliasing, zero values, `struct{}` sets — with no OOP baggage.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)

## Why this matters
Every controller you touch — client-go, controller-runtime, kubebuilder scaffolding — is a wall of structs, pointer receivers, and slices of typed objects. If slice aliasing or nil maps surprise you, your reconcile loop mutates shared cache objects or panics on a nil write in production. Interviewers at GPU shops will hand you a `func (r *Reconciler) Reconcile(...)` and expect you to know instantly why the receiver is a pointer. This lesson makes the type system muscle memory so the k8s libraries read as obvious.

## From Python to Go
Coming from Python, four things bite:

- **No classes, no `self`, no inheritance.** You attach behavior to types via *methods with receivers*. Composition (embedding) replaces subclassing.
- **Assignment copies.** `b := a` on a struct copies the whole struct. Python rebinds a reference; Go copies the value. This is why you pass `*T` when you want mutation to stick.
- **`nil` is not `None`.** A nil map is *readable* (returns zero values) but *panics on write*. A nil slice is fully usable — you can `append` to it. Python's `None` does neither.
- **Static types, zero values, no `__init__`.** Every declared variable is immediately usable at its zero value. There is no "uninitialized" — but the zero value of a map or pointer is `nil`, which is a trap, not a convenience.

The mental shift: stop thinking "object references" and start thinking "values, and pointers to values."

## Core notes

**Zero values.** Declaring a variable always yields a usable zero: `0`, `false`, `""`, `nil` for pointers/slices/maps/channels/interfaces/funcs, and *recursively zeroed fields* for structs. No constructors run.

```go
var c Cost          // struct: all fields zeroed
var s []string      // nil slice — len 0, safe to append
var m map[string]int // nil map — safe to READ, panics on WRITE
```

**Slices alias a backing array.** A slice is a 3-word header `{ptr, len, cap}`. Copying the header (pass to a func, assign) shares the *same backing array*. Mutating an element through any alias is visible to all.

```go
func zeroOut(xs []float64) {
    for i := range xs {
        xs[i] = 0 // caller SEES this — same backing array
    }
}
```

But `append` may or may not be visible: if it fits in `cap`, it writes in place (caller's array mutated, caller's `len` unchanged); if it reallocates, the callee gets a fresh array and the caller sees nothing. **Never rely on append's side effects across a function boundary — return the slice.** This is the single most common slice bug.

```go
xs = append(xs, v) // idiom: always reassign the result
```

**`nil` vs empty slice.** `var s []T` (nil) and `s := []T{}` (empty, non-nil) behave identically for `len`, `range`, `append`. They differ in `s == nil` and in JSON: a nil slice marshals to `null`, an empty slice to `[]`. For API responses and exporter output, prefer initializing to empty when "no items" should serialize as `[]`.

**Maps.** The zero value is `nil`. Reading a nil map is fine; writing panics. Always `make` before writing:

```go
m := make(map[string]float64)
m["ns/prod"] += 4.10          // ok
v, ok := m["ns/dev"]          // comma-ok: v=0, ok=false — distinguishes "absent" from "zero"
```

Map iteration order is **randomized** by design — never assume order; sort keys explicitly for stable table output. Maps are reference-like: passing a map to a func lets the callee mutate the caller's map (no copy of contents).

**Value vs pointer receivers.** A method set is declared on a type:

```go
func (c Cost) Total() float64    { return c.GPUHours * c.Rate } // value: reads a COPY
func (c *Cost) ApplyDiscount(p float64) { c.Rate *= (1 - p) }   // pointer: mutates caller
```

Rules that actually matter:

- A **value receiver operates on a copy** — mutations are lost. If your method mutates fields, it *must* be a pointer receiver, or the change silently vanishes.
- Use pointer receivers to **avoid copying large structs** and when any method needs to mutate.
- **Be consistent per type**: if one method needs a pointer receiver, give them all pointer receivers. Mixing confuses the method set and interface satisfaction.
- Only **addressable** values can call pointer-receiver methods automatically. `map[k]Struct` values are not addressable — `m["x"].Mutate()` won't compile. Store `*Struct` in the map, or read-modify-write the whole value back.

**Interface satisfaction & the receiver trap.** If methods are on `*T`, then only `*T` satisfies the interface, not `T`. Passing a `T` value where the interface is expected fails to compile. This surprises Pythonistas constantly.

**`struct{}` as a set.** Go has no set type. Idiom: `map[T]struct{}` — the empty struct occupies **zero bytes**, signaling "key presence only."

```go
seen := map[string]struct{}{}
seen["gpu-node-1"] = struct{}{}
if _, ok := seen["gpu-node-1"]; ok { /* present */ }
```

**Structs, tags, and construction.** No constructors; use struct literals with field names. Tags drive JSON (de)serialization:

```go
type GPUCost struct {
    Namespace string  `json:"namespace"`
    Label     string  `json:"label,omitempty"` // omit if empty when marshaling
    GPUHours  float64 `json:"gpu_hours"`
    USD       float64 `json:"usd"`
}
c := GPUCost{Namespace: "prod", GPUHours: 12, USD: 41.0}
```

**Errors are values.** Functions return `(T, error)`; check `if err != nil`. No exceptions for ordinary failures (Lesson 01.2 goes deep). `panic` is for programmer bugs, not control flow.

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

## Practice
Port an existing Python GPU-cost script to Go as the first cut of `gpu-cost-exporter`.

- Read a JSON cost export (array of records: namespace, a label/team field, gpu_hours, usd) from a file or stdin.
- Decode into a `[]Record` of typed structs with JSON tags.
- Aggregate cost two ways: by `namespace` and by `label`, using `map[string]float64`.
- Print each aggregation as a table sorted by key (use `sort` + `text/tabwriter`).
- Use `(T, error)` returns for decode/parse; check every `err`.

**Acceptance:** `go build ./...` succeeds; `go vet ./...` is clean; running against a sample export produces correct per-namespace and per-label totals in deterministic sorted order. No nil-map panics, no reliance on append side effects.

## Self-check

**(a) When does passing a slice let the callee mutate the caller's backing array, and when not?**
**Answer:** Mutating *existing elements* by index (`xs[i] = v`) is always visible to the caller — both share one backing array. `append` is visible only if the new length still fits within the original `cap` (in-place write); if `append` exceeds `cap` it allocates a new array and the callee's changes are invisible. Because you can't predict reallocation, always return the appended slice and reassign.

**(b) Value vs pointer receiver — what breaks on a mutating method if you pick wrong?**
**Answer:** A value receiver operates on a copy, so any field mutation is discarded when the method returns — the caller's value is unchanged, silently. A mutating method must use a pointer receiver. Secondary breakage: if methods are on `*T`, only `*T` (not `T`) satisfies interfaces, and `T` values stored in a map aren't addressable so their pointer methods can't be called.

**(c) What is the zero value of a map, and what happens writing to a nil map?**
**Answer:** The zero value of a map is `nil`. Reading from a nil map is safe and returns the element type's zero value (with `ok == false`), but *writing* to a nil map panics at runtime. You must `make(map[K]V)` (or a literal `map[K]V{}`) before assigning keys.

## Resources
1. **A Tour of Go** — https://go.dev/tour — Interactive syntax/types/methods walkthrough. *Skim.* As an experienced dev, blast through Basics → Methods/Interfaces in an afternoon; it's the fastest way to internalize receivers and slices hands-on.
2. **Learning Go, 2nd ed. (Jon Bodner), ch. 1–6** — https://www.oreilly.com/library/view/learning-go-2nd/9781098139285/ — Types, composite types, functions, pointers, methods. *Skim.* Bodner is explicit about *why* Go differs from other languages; the slice-aliasing and pointer-receiver sections are the sharpest treatment in print.
3. **Go by Example** — https://gobyexample.com — Copy-paste-ready snippets for maps, slices, structs, sorting. *Lookup.* Keep it open as a reference while porting the script; ideal when you know the concept and just need the idiom.
