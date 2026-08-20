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
sources: 13
---

# 01.1 · Syntax and Types

> **Concept.** Type idiomatic Go fluently — receivers, slice/map aliasing, zero values, `struct{}` sets, field layout, and escape analysis — with no OOP baggage.
>
> Module: [🐹 01 — Go for infrastructure engineers](../README.md) · Deliverable: [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md)
>
> **Warm-up** — new to Go from Python? [Python to Go](../../../learning/00-go-warm-up.html) covers the mental-model shifts this module's skip-list assumes you have already made — value vs reference, the four nils, errors as values, implicit interfaces — with both sides of every comparison run live. Skip it if Go is already familiar.

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

Everything in this lesson that shows a transcript was captured on **go1.24.7 linux/amd64** unless labelled otherwise. Sizes and layouts assume a 64-bit platform (8-byte words). Where a behaviour is an implementation detail rather than a language guarantee, it is called out as such — that distinction is itself part of the lesson.

## Core concepts

### The value model: what a Go variable actually is

Start with the single sentence the rest of the lesson unpacks: **a Go variable is a named region of memory whose size and layout are fixed by its type, and assignment copies those bytes.**

Python's model is different in a way that matters more than the syntax. In CPython, a name is a slot in a namespace dict holding a pointer to a heap-allocated `PyObject`. `b = a` copies the pointer and bumps a refcount; `a` and `b` now name the same object, so `b.field = 1` is visible through `a`. Every value is boxed, every value is on the heap, and mutability is a property of the object, not of how you passed it.

In Go, `b := a` copies `unsafe.Sizeof(a)` bytes into a new region of memory. If `a` is a struct with five fields, all five are copied. If `a` is an `int64`, eight bytes are copied. There is no box, no refcount, no shared identity — unless the bytes you copied *contain* a pointer, in which case the copy points at the same thing the original pointed at. That last clause is where every "wait, I thought this was a copy" bug lives, and it's why slices and maps get their own sections below: their bytes contain pointers.

This gives you a rule you can apply mechanically, without memorising special cases:

> To know whether a callee can affect the caller's data, ask what is inside the bytes being copied. If the copied bytes contain a pointer to shared memory, the callee reaches the caller's data through it. Otherwise it cannot.

| Type | What the copied bytes are (64-bit) | Size | Callee can mutate caller's data? |
|---|---|---|---|
| `int`, `float64`, `bool` | the value itself | 8 / 8 / 1 | No |
| `[4]int` (array) | all four elements | 32 | No |
| `struct{A,B int}` | both fields | 16 | No |
| `*T` | one pointer | 8 | Yes, via `*p` |
| `[]T` (slice) | `{ptr, len, cap}` | 24 | Yes, elements via the shared array |
| `map[K]V` | one pointer to the map header | 8 | Yes, entries |
| `string` | `{ptr, len}`, data immutable | 16 | No (strings are immutable) |
| `chan T` | one pointer | 8 | Yes, by sending/receiving |
| `interface` | `{type ptr, data ptr}` | 16 | Depends on the dynamic type |
| `func` | one pointer to a closure object | 8 | Via captured variables |

Those sizes are not folklore. `unsafe.Sizeof` prints them, and this transcript is captured, not remembered:

```
slice header size 24 string header 16 map 8 iface 16
```

The Go spec guarantees the *numeric* sizes (`int64`/`float64` are 8 bytes, `complex128` is 16) and the alignment rules; the header sizes above are implementation properties of the gc compiler on 64-bit platforms, stable for many releases but not language guarantees (Go spec, "Size and alignment guarantees").

The practical consequence, and the thing to internalise before reading any controller code: **value vs. pointer is an API contract, not a memory micro-decision.** A value parameter tells the caller "I will not mutate what you gave me — pass a copy of your record and stop worrying." A pointer parameter tells the caller "I might mutate this, and if you shared this pointer with anyone else, they'll see it too." Get that backwards in a struct passed through layers — an aggregator hands a `*Record` to an exporter, which hands it to a CRD-status writer — and you create a bug class where an inner layer's mutation surprises a caller who believed it held an immutable copy. In controller code this is not hypothetical: objects you `Get` from the controller-runtime cache are pointers into a shared informer cache, and mutating one is mutating everyone's copy.

### Zero values: why Go has no constructors

The problem constructors solve is "a variable exists but hasn't been initialised, and reading it is undefined." C solves it badly (garbage bytes), Java solves it with `null` plus a constructor call, Python solves it with `__init__` and an exception if you forget.

Go solves it by *definition*: every declaration initialises its memory to the type's zero value. There is no uninitialised state to trip over. Mechanically, the compiler either zeroes the stack slot or asks the allocator for zeroed memory; for composite types it zeroes recursively, field by field, element by element.

```go
var c Cost           // struct: every field zeroed, recursively
var n int            // 0
var f float64        // 0.0
var s string         // "" — a {nil, 0} header, not nil
var p *Cost          // nil
var xs []string      // nil slice — len 0, cap 0, safe to range and append
var m map[string]int // nil map — safe to READ, PANICS on write
var ch chan int      // nil channel — blocks forever on send and receive
var e error          // nil interface — both halves zero
```

The design intent is that the zero value should be *useful*, and the stdlib is built that way: `var buf bytes.Buffer` is a ready-to-use buffer, `var mu sync.Mutex` is an unlocked mutex, `var wg sync.WaitGroup` is ready to `Add`. When you design a struct for the exporter, aim for the same property — a zero `Config` should be a working default, not a landmine.

Three zero values *are* landmines, and they are the ones you meet daily:

- **nil map** — reads fine, panics on write. Section below.
- **nil channel** — blocks forever on both send and receive, so a forgotten `make(chan T)` looks like a deadlock, not a nil error. (Lesson 4 leans on this deliberately: a nil channel in a `select` is a permanently disabled case.)
- **nil pointer in an interface** — the typed-nil trap, which is [Lesson 3](03-interfaces-and-composition.md)'s central gotcha and the reason `err != nil` can be true when there is no error.

There is no `__init__`, so where Python would run initialisation logic, Go uses either a struct literal with field names or a `New…` constructor function that returns the ready value:

```go
c := GPUCost{Namespace: "prod", GPUHours: 12, USD: 41.0} // named fields: order-independent, survives field additions
c2 := GPUCost{"prod", "", 12, 41.0}                      // positional: breaks silently when a field is inserted
```

Use named fields. Positional literals compile until someone adds a field in the middle of the struct, at which point they either fail to compile (if types differ) or, worse, keep compiling with the values shifted into the wrong fields.

### Struct layout: alignment, padding, and why field order is not cosmetic

Here is the mechanism, because "Go doesn't reorder fields" is the fact, not the reason.

Every type has an **alignment** — the address multiple its value must start at. On amd64/arm64 a `float64`, an `int64`, and a pointer align to 8; an `int32` to 4; an `int16` to 2; a `bool` or `byte` to 1. The hardware wants this: a load that straddles a cache line or a natural boundary costs extra work, and some architectures fault outright. A struct's alignment is the maximum of its fields' alignments (Go spec, "Size and alignment guarantees").

The compiler lays out fields **in declaration order** — Go deliberately does not reorder them, because the layout is observable through `unsafe.Offsetof`, `encoding/binary`, and cgo. To honour each field's alignment while keeping declaration order, it inserts **padding** bytes. And because arrays of the struct must keep every element aligned, the total size is rounded up to a multiple of the struct's alignment — trailing padding.

Run it and look:

```go
package main

import (
	"fmt"
	"unsafe"
)

type Bad struct { // declared small-large-small
	Dirty bool
	USD   float64
	Stale bool
}

type Good struct { // declared large-to-small
	USD   float64
	Dirty bool
	Stale bool
}

func main() {
	fmt.Println("Bad size", unsafe.Sizeof(Bad{}), "align", unsafe.Alignof(Bad{}))
	fmt.Println("  offsets:", unsafe.Offsetof(Bad{}.Dirty), unsafe.Offsetof(Bad{}.USD), unsafe.Offsetof(Bad{}.Stale))
	fmt.Println("Good size", unsafe.Sizeof(Good{}), "align", unsafe.Alignof(Good{}))
	fmt.Println("  offsets:", unsafe.Offsetof(Good{}.USD), unsafe.Offsetof(Good{}.Dirty), unsafe.Offsetof(Good{}.Stale))
}
```

Captured output (go1.24.7, linux/amd64):

```
Bad size 24 align 8
  offsets: 0 8 16
Good size 16 align 8
  offsets: 0 8 9
```

The offsets tell the whole story, and the picture makes it obvious:

```
  Bad{Dirty bool; USD float64; Stale bool}      24 bytes, 14 wasted
  byte:  0    1                              8                    16   17            24
         ┌────┬───────────────────────────────┬───────────────────┬────┬─────────────┐
         │Dirt│ P P P P P P P  (7 pad bytes)  │  USD  (float64)   │Stal│ P P P P P P │
         └────┴───────────────────────────────┴───────────────────┴────┴─────────────┘
           ▲   └─ USD needs offset % 8 == 0 ──┘                     ▲   └ trailing pad:
           │                                                        │     size must be a
           └─ align 1, sits anywhere                                │     multiple of 8
                                                                    └─ align 1

  Good{USD float64; Dirty bool; Stale bool}     16 bytes, 6 wasted
  byte:  0                                    8    9    10                          16
         ┌─────────────────────────────────────┬────┬────┬─────────────────────────┐
         │            USD  (float64)           │Dirt│Stal│  P P P P P P (6 pad)    │
         └─────────────────────────────────────┴────┴────┴─────────────────────────┘
           ▲ offset 0, already 8-aligned        ▲    ▲    ▲
           │                                    │    │    └─ only trailing padding is left
           └────────────────────────────────────┴────┴────── the two bools pack adjacently
```

The heuristic that falls out: **order fields from largest alignment to smallest** (pointers/8-byte scalars → 4-byte → 2-byte → 1-byte → zero-size), and put related small fields next to each other so they share one padding run.

You do not have to eyeball this. The `fieldalignment` analyzer from `golang.org/x/tools` reports structs whose size would shrink under reordering, and prints the suggested order:

```sh
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest
fieldalignment ./...
# or, as a vet tool:
go vet -vettool=$(which fieldalignment) ./...
```

Run against the two structs above, it finds exactly one problem and quantifies it:

```
main.go:8:10: Bad has size 24 but the optimal size is 16 leading to a waste of 8 bytes (33%)
```

`Good` produces no diagnostic — it is already optimal.

Now the part that makes this worth minutes of your attention rather than none. **Worked cost, carried in units:** the exporter's `Record` in this lesson's worked example measures 56 bytes. Suppose a fleet of 10,000 GPUs, one record per GPU per 15-second scrape, and you hold the last hour in memory for rate calculations:

```
records held = 10,000 GPUs × (3600 s / 15 s) = 10,000 × 240 = 2,400,000 records
at 56 B/record      = 134.4 MB of live backing array
at 64 B/record (+8) = 153.6 MB          → +19.2 MB purely padding
at 72 B/record (+16)= 172.8 MB          → +38.4 MB purely padding
```

Eight bytes of avoidable padding on a hot record is ~19 MB of RSS and ~19 MB more for the GC to scan on every cycle, per replica. That is a real number on a memory-limited exporter pod, and it is free to reclaim: you change the declaration order and recompile. It is *not* worth doing for a config struct instantiated once at startup — the discipline is "hot, high-cardinality records get ordered; everything else gets written for readability."

One more layout fact you will meet in k8s code: **embedding does not change this.** An embedded struct is laid out inline at its field position, with its own alignment; embedding `metav1.ObjectMeta` into your CRD type puts all of its fields in your struct's memory, in its declaration order.

### Slices: the header, the backing array, and what aliasing really shares

A slice is not a container. It is a three-word **descriptor** of a window onto an array that lives somewhere else:

```go
type sliceHeader struct {
    Data uintptr // pointer to element 0 of the window
    Len  int     // number of elements you may index
    Cap  int     // elements from Data to the end of the backing array
}
```

24 bytes on 64-bit. Copying a slice — passing it to a function, assigning it, storing it in a struct — copies **those 24 bytes only**. The backing array is not copied, not refcounted, not marked. Two slice values that came from the same array are two windows onto the same memory.

```
  base := make([]int, 5, 8);  view := base[1:3]

  base   ┌──────────┬───────┬───────┐
  header │ Data ────┼ Len 5 │ Cap 8 │
         └────┬─────┴───────┴───────┘
              │
              ▼
   backing  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
   array    │  0  │  1  │  2  │  3  │  4  │  -  │  -  │  -  │   8 elements
   index      [0]   [1]   [2]   [3]   [4]   [5]   [6]   [7]
                     ▲                 ▲                    ▲
                     │                 │                    │
  view   ┌──────────┬┼──────┬───────┐  │                    │
  header │ Data ────┘│Len 2 │ Cap 7 │  │                    │
         └───────────┴──────┴───────┘  │                    │
                     └─ view[0] is base[1]                  │
                                       └─ view's len ends here, but its
                                          cap runs to the end of the array,
                                          so append(view, x) writes base[3]
```

Read the diagram again with one question in mind: *what does `view` have the right to write?* Indices `[0, len)` are what you may read; indices `[len, cap)` are memory `append` is allowed to write **in place**, and that memory belongs to `base` too. That single overlap is the source of essentially every slice bug in production Go.

Run it:

```go
base := make([]int, 5, 8)
for i := range base {
	base[i] = i
}
view := base[1:3]
fmt.Printf("base=%v len=%d cap=%d\n", base, len(base), cap(base))
fmt.Printf("view=%v len=%d cap=%d\n", view, len(view), cap(view))
view = append(view, 99)              // fits in cap → writes base[3] in place
fmt.Printf("after append: base=%v view=%v\n", base, view)
view2 := base[1:3:3]                 // three-index slice: cap limited to 2
view2 = append(view2, 77)            // cap exceeded → copies to a fresh array
fmt.Printf("after 3-index append: base=%v view2=%v (view2 cap=%d)\n", base, view2, cap(view2))
```

Captured output:

```
base=[0 1 2 3 4] len=5 cap=8
view=[1 2] len=2 cap=7
after append: base=[0 1 2 99 4] view=[1 2 99]
after 3-index append: base=[0 1 2 99 4] view2=[1 2 77] (view2 cap=4)
```

Line 3 is the bug in miniature: appending to `view` silently overwrote `base[3]`. Line 4 is the fix: `base[low:high:max]` — the **three-index slice expression** — sets the new slice's cap to `max-low`, so the very first `append` is forced to allocate rather than trespass. When you hand a sub-slice to code you don't control, `xs[a:b:b]` is the way to say "this window is yours, the rest is mine."

#### What `append` actually does

`append` is not a function call in the normal sense; the compiler expands it. The decision procedure is:

```
  append(s, v...)  with  need = len(s) + len(v)
  ────────────────────────────────────────────────────────────────────────
        need <= cap(s) ?
             │
      yes ───┴──────────────────────────────── no
       │                                        │
       ▼                                        ▼
  write v into the EXISTING backing         runtime.growslice(...)
  array at s[len(s):need]                        │
       │                                         ├─ 1. newcap = nextslicecap(need, cap(s))
       │                                         ├─ 2. bytes  = roundupsize(newcap * elemsize)
       │                                         ├─ 3. newcap = bytes / elemsize   ← re-derived!
       │                                         ├─ 4. p = mallocgc(bytes)
       │                                         └─ 5. memmove(p, old, len(s)*elemsize)
       ▼                                         ▼
  return header {same ptr, need, cap(s)}    return header {p, need, newcap}
       │                                         │
       └──── caller's OTHER aliases SEE the ─────┘──── caller's other aliases see
             new elements (same array)                 NOTHING (different array)
```

The growth policy in step 1 is `runtime.nextslicecap` (`src/runtime/slice.go`, Go 1.24):

- If the required length is more than double the old cap, use the required length exactly.
- Else if `oldCap < 256`, **double** it.
- Else grow by `newcap += (newcap + 3*256) >> 2` repeatedly until it is big enough — a smoothed ~1.25× factor, so large slices don't jump to twice the memory they need.

Step 2 and 3 are the part nobody remembers and that produces the "weird" capacities. The allocator does not serve arbitrary sizes; it serves **size classes** (`src/runtime/sizeclasses.go`: …4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192…). `roundupsize` rounds the request up to the next class, and then `growslice` *divides that rounded byte count back by the element size* and reports the result as the capacity — so you get the extra elements the allocator was going to give you anyway.

Watch it happen on a plain `[]int` (8-byte elements):

```
len=1 cap=1
len=2 cap=2
len=3 cap=4
len=5 cap=8
len=9 cap=16
len=17 cap=32
len=33 cap=64
len=129 cap=256
len=257 cap=512
len=513 cap=848      ← doubling has stopped
len=849 cap=1280
len=1281 cap=1792
len=1793 cap=2560
```

Check `512 → 848` against the algorithm by hand: `oldCap = 512 ≥ 256`, so `newcap = 512 + ((512 + 768) >> 2) = 512 + 320 = 832`. Bytes: `832 × 8 = 6656`. The next size class up from 6656 is **6784**. Back-divide: `6784 / 8 = 848`. Exactly the printed number, no hand-waving.

This is also the mechanism behind **golang/go#66093, "append() grows backing array to odd-sized, unexpected capacity"** (reported against Go 1.22.0; now closed, labelled `FrozenDueToAge`). The reporter appended to a slice of a struct larger than 128 bytes containing pointers and found capacity going to 37 instead of the expected 64, where Go 1.21.7 had doubled. Reproduce the same class of surprise today with a 136-byte struct:

```go
type Big struct {
	Name string   // 16 B, contains a pointer
	Tags []string // 24 B, contains a pointer
	Pad  [96]byte // 96 B
}                // → unsafe.Sizeof(Big{}) == 136
```

```
sizeof Big = 136
len=1 cap=1 bytes=136
len=2 cap=2 bytes=272
len=3 cap=4 bytes=544
len=5 cap=8 bytes=1088
len=9 cap=16 bytes=2176
len=17 cap=35 bytes=4760      ← not 32
len=36 cap=71 bytes=9656      ← not 70
```

The arithmetic: doubling 16 gives 32; `32 × 136 = 4352` bytes; the next size class is **4864**; `4864 / 136 = 35` (integer division); the slice reports cap 35 and the final allocation is `35 × 136 = 4760` bytes, with 104 bytes of the size class left over. Nothing is broken — the runtime is handing you the elements you already paid for. **The lesson to take is the one the issue's own reporter states: capacity growth is unspecified behaviour and must not be relied on.** Test-asserting on `cap()` is a test that breaks on a point release.

#### The rules that follow

1. **Always reassign: `xs = append(xs, v)`.** Not style — correctness. The returned header may point at different memory.
2. **Never rely on an append inside a callee being visible to the caller.** It is visible exactly when the append fit in cap, which the call site cannot see. Return the slice instead.
3. **Mutating existing elements by index is always visible** through every alias — that's the same array by definition.
4. **`copy(dst, src)` when you need independence.** `copy` moves `min(len(dst), len(src))` elements and returns the count; it never allocates.
5. **Cap a window you hand out: `xs[a:b:b]`.**
6. **A slice keeps its entire backing array alive.** Holding `records[len(records)-10:]` from a 2-million-element scrape pins all 2 million elements against the GC, because the header still points into that array. `slices.Clone` (or `copy` into a fresh slice) is how you drop the rest.

#### nil slice vs empty slice

`var s []T` is nil: header `{nil, 0, 0}`. `s := []T{}` is non-nil with a zero-size allocation. They behave identically for `len`, `cap`, `range`, and `append` — appending to a nil slice is the normal way to build one. They differ in exactly two places, and both bite in an exporter:

| | `var s []string` (nil) | `s := []string{}` (empty) |
|---|---|---|
| `s == nil` | `true` | `false` |
| `len(s)`, `range`, `append` | works | works |
| `json.Marshal` | `null` | `[]` |

If a consumer of your JSON branches on `null` vs `[]` — and Kubernetes API consumers frequently do — this is a wire-format difference you did not intend. For output structs, initialise slices to empty.

### Strings and `[]byte`

A `string` is a two-word header `{ptr, len}` (16 bytes) over **immutable** bytes. You cannot assign `s[i] = 'x'`. That immutability is what lets the runtime share string data freely: slicing a string (`s[3:8]`) allocates nothing, it just makes a new header into the same bytes.

The conversions do allocate, and this shows up in profiles:

- `[]byte(s)` copies, because the result is mutable and the source must not be.
- `string(b)` copies, for the same reason in reverse.

Two compiler optimisations you can rely on: `for i, r := range s` decodes UTF-8 without allocating, and the map lookup `m[string(b)]` where `b` is a `[]byte` is special-cased to avoid the copy. Note the two `range` forms differ: `s[i]` indexes a **byte**, while `for _, r := range s` yields a **rune** (`int32`) decoded from UTF-8, with `i` jumping by the encoded width. `len(s)` is bytes, not characters — for a namespace label that's the same thing, for anything user-supplied it is not.

### Maps: nil semantics, iteration order, and what you may not do

A map value is **one word** — a pointer to a runtime map header. Copying a map value copies the pointer, so a callee mutates the caller's map. There is no copy-on-write and no way to deep-copy without iterating.

The zero value is `nil`, and the asymmetry is deliberate:

```go
var m map[string]float64
fmt.Println(m["x"], len(m)) // reads fine: 0, 0 — a nil map is an empty map for reading
m["x"] = 1                  // panic: assignment to entry in nil map
```

Captured:

```
read nil map: 0 len: 0
recovered: assignment to entry in nil map
```

Reads work because the lookup path checks for a nil header and returns the zero value; writes cannot, because there are no buckets to write into and the runtime refuses to silently allocate one behind your back. Make it first:

```go
m := make(map[string]float64)        // or map[string]float64{}
m := make(map[string]float64, 5000)  // pre-size: skips incremental growth/rehash work
```

Pre-sizing is worth it when you know the cardinality — an exporter aggregating a known number of namespaces — because it avoids repeated table growth as the map fills.

**Comma-ok distinguishes absent from zero**, which matters enormously in a metrics context where `0.00` is a legitimate cost:

```go
v, ok := m["ns/dev"] // v == 0, ok == false → the key is absent
```

**Iteration order is randomised, on purpose, on every range.** Go 1.24 replaced the map implementation with Swiss Tables (go.dev, "Faster Go maps with Swiss Tables"; ~30% faster access/assignment on maps over 1024 entries, ~35% faster assignment into pre-sized maps, and iteration faster across the board). The randomisation survived the rewrite: the iterator picks a random starting group and a random slot offset within groups (`internal/runtime/maps`, `Iter.Init` seeds `entryOffset` and `dirOffset` with `rand()`), then wraps around. Three consecutive ranges over the same five-key map:

```
iter: d e a b c 
iter: b c d e a 
iter: a b c d e 
```

This is a feature: it prevents code (and tests) from depending on an order the runtime never promised. For deterministic output you sort keys explicitly — that's what the worked example does.

Two more map rules you will hit:

- **Map elements are not addressable.** `&m[k]` does not compile, and neither does `m[k].Field = v` for a struct-valued map. The reason is mechanical: the runtime may move entries during growth, so an address into a map cannot be kept safely. Store `*T` values, or read-modify-write the whole struct (`t := m[k]; t.USD += x; m[k] = t` — exactly what the worked example does).
- **Deleting during iteration is defined** (entries not yet reached may or may not be produced); adding during iteration is defined the same loose way. Neither is a good idea. Collect the keys, then act.

### Value vs pointer receivers, method sets, and addressability

A method is a function with a receiver. Go gives you two forms, and the choice is semantic:

```go
func (c Cost) Total() float64           { return c.GPUHours * c.Rate } // operates on a COPY
func (c *Cost) ApplyDiscount(p float64) { c.Rate *= (1 - p) }          // operates on the ORIGINAL
```

`ApplyDiscount` with a value receiver would compile, run, mutate the copy, and discard it — no error, no warning, no effect. That is the single most common receiver bug.

The rules worth memorising are the **method set** rules, because interface satisfaction is defined in terms of them (Go spec, "Method sets"):

| Type | Methods in its method set |
|---|---|
| `T` | all methods declared with receiver `T` |
| `*T` | all methods declared with receiver `T` **or** `*T` |

So `*T`'s method set is a superset of `T`'s. If any method of your type has a pointer receiver, **only `*T` satisfies interfaces containing that method**, and a `T` value will not.

Separately from method sets, the compiler inserts `&` and `*` for you when calling a method directly — but only if the operand is **addressable**. `x.Mutate()` is rewritten to `(&x).Mutate()` when `x` is a variable, a pointer dereference, a slice element, or a field of an addressable struct. It is *not* rewritten when `x` is a map element, a function's return value, or any other non-addressable expression.

Here is the whole thing as three real compiler errors — this file does not build, and the messages are captured verbatim:

```go
package main

type Cost struct {
	Rate  float64
	Notes []string
}

func (c *Cost) ApplyDiscount(p float64) { c.Rate *= 1 - p }

type Discounter interface{ ApplyDiscount(float64) }

func main() {
	m := map[string]Cost{"a": {Rate: 1}}
	m["a"].ApplyDiscount(0.1) // (1) map element is not addressable

	var d Discounter = Cost{} // (2) value type, pointer-receiver method
	_ = d

	a := Cost{}
	b := Cost{}
	_ = a == b // (3) struct contains a slice
}
```

```
./main.go:14:9: cannot call pointer method ApplyDiscount on Cost
./main.go:16:21: cannot use Cost{} (value of struct type Cost) as Discounter value in variable declaration: Cost does not implement Discounter (method ApplyDiscount has pointer receiver)
./main.go:21:6: invalid operation: a == b (struct containing []string cannot be compared)
```

Error (2) is the one you will hit reading controller code. `reconcile.Reconciler` has one method; if you write `func (r *GPUCostReconciler) Reconcile(...)`, then `&GPUCostReconciler{}` satisfies it and `GPUCostReconciler{}` does not — which is why every kubebuilder scaffold hands the manager a pointer.

Decision rules, in the order you should apply them:

1. **Any method mutates → pointer receiver.** Non-negotiable.
2. **Struct is large or copied in a hot path → pointer receiver.** "Large" is fuzzy; a useful threshold is "more than a few words", i.e. bigger than a slice header.
3. **Type contains a `sync.Mutex`, `sync.WaitGroup`, or anything with a `noCopy` marker → pointer receiver**, always. Copying a mutex copies its lock state; `go vet` catches the obvious cases with "passes lock by value" and misses the subtle ones.
4. **Be consistent per type.** If one method needs a pointer receiver, give them all pointer receivers. Mixed receivers make the method set hard to reason about and produce the confusing case where `T` satisfies an interface and `*T` satisfies a bigger one.
5. Value receivers are fine — and cheap — for small immutable value types: a `Duration`-like wrapper, a coordinate pair, an enum with a `String()` method.

### `struct{}` as a set

Go has no set type. The idiom is a map whose value type occupies no memory:

```go
seen := make(map[string]struct{}, 1024)
seen["gpu-node-1"] = struct{}{}
if _, ok := seen["gpu-node-1"]; ok {
	// present
}
```

`struct{}` has size 0 (Go spec: "a struct or array type has size zero if it contains no fields with size greater than zero"), so the map stores keys and control bytes only, with no value array at all. Compare with `map[string]bool`, which allocates one byte per entry plus alignment: on a 1-million-entry set that's roughly a megabyte of pure ceremony, and worse, `map[string]bool` invites the reader to wonder what `false` means — "absent", "present but disabled", or "someone forgot". `struct{}` cannot be misread: presence is the only bit.

The `struct{}{}` spelling is the type `struct{}` followed by an empty composite literal `{}`. It looks strange exactly once.

A related fact the spec calls out: distinct zero-size variables may share an address. The runtime returns a pointer to a single global `zerobase` for zero-size allocations, so `&struct{}{} == &struct{}{}` is not guaranteed to be either true or false. Don't build identity on zero-size values.

### Comparability: a compile-time property, sometimes a runtime panic

Coming from Python, `==` "always works." In Go, comparability is part of the type system, and the rules are exact (Go spec, "Comparison operators"):

| Type | `==` allowed? | Notes |
|---|---|---|
| bool, numeric, string, pointer, channel | yes | pointers equal if same variable or both nil |
| array | yes, if the element type is comparable | element-wise, in index order, short-circuits |
| struct | yes, if **every** field type is comparable | field-wise in source order, short-circuits; blank `_` fields skipped |
| interface | yes | equal if dynamic types are identical and dynamic values are equal |
| **slice, map, func** | **no** | may only be compared to `nil` |

Two consequences that feel unrelated but come from the same rule:

- A struct with a slice/map/func field **fails to compile** on `==` — error (3) above. This is good: it's caught at build time, not in production.
- Comparing two **interface** values whose dynamic type is not comparable **panics at runtime**. The compiler allowed the comparison because `any == any` is legal in the type system; the runtime discovers the dynamic type is a slice and panics. This also applies transitively — comparing two structs that contain interface fields, or two arrays of interfaces.

When `==` isn't available, in order of preference:

1. **A hand-written `Equal(other T) bool`** — explicit, fast, no reflection, and it lets you define equality (e.g. compare `USD` to the cent, ignore `Stale`).
2. **`slices.Equal` / `maps.Equal`** (stdlib, since Go 1.21) for the field that is actually a slice or map — no reflection, generic, fast.
3. **`reflect.DeepEqual`** — works on anything, uses reflection, slow, and has a footgun: `reflect.DeepEqual([]int{}, []int(nil))` is `false`. Fine in tests, not in hot paths.

Comparability also gates map keys: a map key type must be comparable, which is why `map[[]string]int` does not compile and `map[[2]string]int` does.

### Struct tags, JSON, and the `omitempty` trap

Struct tags are string literals attached to fields, read at runtime via reflection by packages like `encoding/json`. They are opaque to the compiler — a typo in a tag is not a compile error, it is a silently wrong wire format.

```go
type Row struct {
	Namespace string   `json:"namespace"`
	Team      string   `json:"team,omitempty"`
	USD       float64  `json:"usd,omitempty"`
	USD2      float64  `json:"usd_omitzero,omitzero"`
	USD3      float64  `json:"usd_plain"`
	Nodes     []string `json:"nodes"`
	Empty     []string `json:"empty_nonnil"`
}
```

Marshalling `Row{Namespace: "prod", Empty: []string{}}` — everything else at its zero value — produces:

```json
{
  "namespace": "prod",
  "usd_plain": 0,
  "nodes": null,
  "empty_nonnil": []
}
```

Four separate lessons in four lines of output:

- `team` and `usd` **vanished**. `omitempty` drops a field when it is "empty", and `encoding/json`'s definition of empty (`isEmptyValue` in `encoding/json/encode.go`) is: length 0 for arrays/maps/slices/strings, and the *zero value* for bools, all integers, floats, interfaces and pointers. **A real, measured cost of exactly $0.00 is indistinguishable on the wire from "no data collected."** In a billing exporter that is a correctness bug, not a cosmetic one.
- `usd_omitzero` also vanished — `omitzero` (added in Go 1.24) drops zero values too, but with a cleaner definition: it uses the field's `IsZero() bool` method if it has one, otherwise the type's zero value. Its real win is `time.Time`, which `omitempty` never omits (a zero `time.Time` is a non-empty struct) but `omitzero` does.
- `usd_plain` survived as `0` — no tag option, no omission. **For any numeric field where zero is a meaningful measurement, that is the correct choice.**
- `nodes` is `null` (nil slice) while `empty_nonnil` is `[]` (empty non-nil slice) — the nil-vs-empty distinction, now visible on the wire.

Decoding has its own asymmetry worth knowing before you debug it: unknown JSON fields are silently ignored by default (use `dec.DisallowUnknownFields()` to make them errors), and fields absent from the JSON are left at whatever the target already held — which for a fresh struct is the zero value. That means "field absent" and "field present with a zero value" are indistinguishable after decoding into a value type. If you must tell them apart, use a pointer field (`*float64`): `nil` means absent, `&0.0` means present-and-zero.

### Escape analysis: stack, heap, and how to see the decision

Python allocates every object on the heap. Go's compiler tries to allocate on the **stack**, which is dramatically cheaper: a stack allocation is a pointer bump, freed automatically when the frame returns, invisible to the GC. The analysis that decides is **escape analysis**, and its rule is one sentence: *if the compiler cannot prove a value's lifetime ends with the function, the value escapes to the heap.*

You do not have to guess. Ask the compiler:

```sh
go build -gcflags="-m" ./...
```

Here is a small package and the compiler's actual verdicts:

```go
package esc

import "fmt"

type Record struct {
	Namespace string
	GPUHours  float64
	USD       float64
}

func totalStack(recs []Record) float64 { // reads only, never stores the slice
	var sum float64
	for _, r := range recs {
		sum += r.USD
	}
	return sum
}

func newRecord(ns string) *Record { // returns the address of a local
	r := Record{Namespace: ns}
	return &r
}

func logRecord(r Record) { // boxes into an interface
	fmt.Println("record", r.Namespace)
}

func sumFixed() float64 { // fixed, compile-time-known size
	buf := make([]float64, 8)
	buf[0] = 1
	var s float64
	for _, v := range buf {
		s += v
	}
	return s
}

func sumDynamic(n int) float64 { // size known only at runtime
	buf := make([]float64, n)
	var s float64
	for _, v := range buf {
		s += v
	}
	return s
}
```

```
./esc.go:12:17: recs does not escape
./esc.go:21:16: leaking param: ns
./esc.go:22:2: moved to heap: r
./esc.go:27:16: leaking param: r
./esc.go:28:14: "record" escapes to heap
./esc.go:28:25: r.Namespace escapes to heap
./esc.go:33:13: make([]float64, 8) does not escape
./esc.go:44:13: make([]float64, n) escapes to heap
```

Line by line:

| Verdict | Why the compiler said it |
|---|---|
| `recs does not escape` | `totalStack` only reads elements; no reference outlives the call, so the caller's array can stay wherever it is. |
| `moved to heap: r` | `newRecord` returns `&r`. The caller holds that address after the frame is gone, so `r` must outlive the frame. |
| `leaking param: ns` / `leaking param: r` | the parameter (or something reachable from it) is stored somewhere that outlives the call — here, into the `fmt` call's argument slice. |
| `"record" escapes to heap` | converting a value to `any` for `fmt.Println` needs a pointer to the data; the compiler must materialise one on the heap. |
| `make([]float64, 8) does not escape` | constant size, used only locally → the backing array becomes stack space. |
| `make([]float64, n) escapes to heap` | the size is a runtime value; the compiler cannot bound the stack frame, so it allocates. (Go 1.26 widened the cases where slice backing stores can be stack-allocated, so re-run `-gcflags=-m` on your own toolchain rather than trusting this table's boundaries verbatim.) |

The two escape triggers you will meet constantly in exporter and controller code:

1. **Returning a pointer to a local.** `return &Record{...}` always heap-allocates. Often correct — you *want* the object to outlive the constructor — but know you're paying for it.
2. **Boxing a value into an interface.** Passing a concrete value to `fmt.Println`, `slog.Info`, `any`, or any interface parameter typically forces a heap allocation, because the interface's data word must point at something the caller can outlive.

Trigger 2 is the mechanical reason behind "accept interfaces, return structs" — the idiom [Lesson 3](03-interfaces-and-composition.md) teaches as an API rule. A function returning a concrete struct by value can have that value live in the caller's frame; returning it wrapped in an interface generally cannot. It is also why logging in a tight loop is not free: every `slog.Info("scraped", "node", n, "usd", usd)` boxes each argument.

The honest caveat: **do not optimise from this alone.** Escape analysis output tells you where allocations happen; whether they matter is a question for `go test -bench . -benchmem` and `pprof`, which Lesson 5 covers. Use `-gcflags=-m` to explain a profile, not to pre-emptively contort code.

### Generics, calibrated

Go 1.18 added type parameters. The syntax is small:

```go
package agg

// Number is a constraint: the ~ means "any type whose underlying type is this",
// so a `type USD float64` also satisfies it.
type Number interface{ ~int | ~int64 | ~float64 }

// SumBy groups values by a key extracted from each element and sums them.
// K must be usable as a map key; V must be summable.
func SumBy[K comparable, V Number](xs []V, key func(V) K) map[K]V {
	out := make(map[K]V, len(xs))
	for _, x := range xs {
		out[key(x)] += x
	}
	return out
}
```

`comparable` is a predeclared constraint meaning "usable with `==`" — the same comparability rule as the table above, which is why it is exactly the constraint a map key needs.

How it compiles matters for the calibration. Go does not do C++-style full monomorphisation (one copy per concrete type) or Java-style erasure (one copy, everything boxed). It uses **GC-shape stenciling**: the compiler generates one instantiation per *memory shape* — all pointer-shaped types share one instantiation — and passes a hidden **dictionary** argument carrying the per-type information (type descriptors, method tables) the shared code needs. The compiler's own type machinery names these "shape" types (`cmd/compile/internal/typecheck`). The practical consequence: generics are neither free nor catastrophic, and calls through a dictionary can be slower than a direct monomorphised call, so generics are not a performance technique. Use them for the type safety.

The judgment call the module cares about: **interface or type parameter?**

| Use a type parameter when | Use an interface when |
|---|---|
| The function's body never calls a method on `T` — it just stores, moves, compares, or arithmetics it | The body calls behaviour on the value: `t.Reconcile()`, `t.String()`, `src.HourlyCost()` |
| You want the *same* concrete type in and out (`func First[T any]([]T) T`) with no boxing | You want callers to substitute different implementations, including a fake |
| The abstraction is over a **data shape** (containers, numeric algorithms, `slices`/`maps` package style) | The abstraction is over **behaviour** (a data source, a client, a writer) |
| Avoiding `any` + type assertions is the point | The set of implementations is open-ended and not known to you |

For a GPU-cost exporter, the honest answer is that you will write one or two small generic helpers (a `keys(map)` or a typed `sum`) and a handful of interfaces (a `CostSource`, a `Clock`). If you find yourself writing a generic function whose type parameter is constrained by a method set, stop — you wanted an interface.

### Errors are values (pointer forward)

Go functions return `(T, error)`; you check `if err != nil`. There are no exceptions for ordinary failures, and `panic` is reserved for programmer bugs. Two type-level facts belong here rather than in the next lesson, because they are type-system facts:

- `error` is an interface with one method, so an error value is a 16-byte `{type, data}` pair like any other interface value — which is why a nil `*MyError` stored in an `error` is not `nil`.
- Sentinel errors are package-level *values*, compared by identity. That is the same value semantics as everything else in this lesson, applied to failure.

[Lesson 2](02-error-handling.md) takes it from there.

## Perspectives

**Developer / API-design view.** Value vs. pointer types are an API contract, not just a memory choice — a value type says "immutable record, take a copy safely," a pointer type says "identity that can be mutated." Getting it backwards in a shared struct passed through layers (an aggregator hands a value to an exporter, which hands it to a CRD-status writer) creates a bug class where an inner layer's mutation surprises a caller who assumed a copy. This is a code-review smell to develop an eye for: does this function's signature honestly describe whether it can change what you gave it? The same eye reads `func(xs []T)` as "may rewrite my elements" and `func(xs []T) []T` as "hand me the result back, I know nothing about your capacity."

**Operator / production view.** Nil-map-write panics and slice-aliasing bugs are exactly the shape of real on-call pages — a panic in a hot reconcile path from an unmade map, or a metrics exporter silently reporting the identical number for two namespaces because two `[]Sample` values shared a backing array after a struct-slice copy. Neither of these looks like a "types bug" from the outside; they look like "the exporter is wrong" or "the controller crashed," and the root cause is a slice header quietly aliasing memory three call frames away. The tell in a panic trace is `assignment to entry in nil map` — that message names the exact bug and the exact fix, which is more than most panics give you.

**Hardware / memory view.** Struct field layout isn't free — Go doesn't reorder fields, so field order determines padding and cache-line packing. At GPU-fleet scale (a `Record` scraped every 15 seconds across 10,000 GPUs and held for an hour: 2.4 M records), eight avoidable padding bytes per record is ~19 MB of RSS per replica and ~19 MB more for the GC to walk every cycle — not academic trivia, and free to reclaim. Escape analysis (`go build -gcflags="-m"`) decides stack vs. heap; returning a pointer to a local, or boxing a value into an interface, forces an escape that a Python developer wouldn't think to check because Python has no equivalent decision to make.

**Economics view.** A cost/efficiency exporter's entire job is to be numerically correct at scale — a slice-aliasing bug or an `omitempty`-on-a-float mismatch doesn't crash anything, it silently corrupts a number that becomes a line item on someone's GPU bill. `usd,omitempty` deleting a genuine $0.00 measurement is the exact shape of this: the consumer cannot distinguish "this namespace cost nothing" from "we failed to collect," so a month of free-tier usage and a month of broken collection look identical in the ledger. A types bug here is a finance bug: nobody pages on it, and it's wrong for months — exactly like the year-long Allegro bug below.

## Real-world use cases

- **Allegro Tech — "Golang slices gotcha"** (<https://blog.allegro.tech/2017/07/golang-slices-gotcha.html>) — a production bug in `allegro/marathon-consul` that lived silently for roughly a year: code built per-port tag lists by appending onto a shared base slice, the appends landed inside the shared backing array's spare capacity, and two service ports ended up with identical Consul tags — making the service undiscoverable by its own clients. **The mechanism is reproducible in ten lines**, which is the point: this is not an exotic failure, it is the default outcome of a very natural refactor.

  ```go
  func buildTags(base []string, svc string) []string {
      return append(base, "svc:"+svc) // writes into base's spare capacity when it fits
  }

  base := make([]string, 0, 4)
  base = append(base, "cluster:gpu-east")
  a := Service{Name: "inference", Tags: buildTags(base, "inference")}
  b := Service{Name: "training", Tags: buildTags(base, "training")}
  ```

  ```
  a.Tags = [cluster:gpu-east svc:training]
  b.Tags = [cluster:gpu-east svc:training]
  same backing array: true
  ```

  `a` was built first and never touched again, yet its tag says `training`: the second `buildTags` call wrote over the same slot. The fix is one character of intent — `buildTags(base[:len(base):len(base)], svc)` — or an explicit `slices.Clone(base)` before appending. What it shows: "never rely on append's side effects across a function boundary" is not a style rule, it is the difference between a working service registry and a year of silent misrouting.

- **golang/go#66093 — "append() grows backing array to odd-sized, unexpected capacity"** (<https://github.com/golang/go/issues/66093>) — reported against Go 1.22.0 and now **closed** (labelled `FrozenDueToAge`): a 32-element slice of a >128-byte pointer-containing struct grew to capacity 37 rather than the expected 64, behaviour that differed from Go 1.21.7. The reporter's own framing is the takeaway — this depends on unspecified behaviour and "no one should rely on this property." As shown above, the same class of result reproduces on Go 1.24.7 (16 → 35 for a 136-byte element) and falls straight out of `nextslicecap` + `roundupsize` + size classes. What it shows: capacity growth is an implementation detail that has changed between point releases, so any test or algorithm keyed on `cap()` is a latent breakage.

  *(Correction to earlier versions of this lesson: this issue was described as "currently-open." It is closed.)*

## Worked example

Aggregate a GPU cost export two ways — by namespace and by team — and print stable, sorted tables. This is the shape of the deliverable's first cut, and every mechanism from Core concepts is doing visible work in it.

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"sort"
	"text/tabwriter"
	"unsafe"
)

// Field order is largest-alignment-first: two 16-byte string headers, two
// 8-byte floats, then the 1-byte bool last so it shares the trailing padding.
// usd/gpu_hours are deliberately NOT `omitempty` — a real 0.00 must survive.
type Record struct {
	Namespace string  `json:"namespace"`
	Team      string  `json:"team,omitempty"`
	GPUHours  float64 `json:"gpu_hours"`
	USD       float64 `json:"usd"`
	Preempted bool    `json:"preempted"`
}

type Totals struct {
	GPUHours float64
	USD      float64
	Records  int
}

// aggregate groups records by a caller-supplied key function.
// The map is made (never nil) and pre-sized, and because map elements are not
// addressable we read-modify-write the whole Totals value.
func aggregate(recs []Record, key func(Record) string) map[string]Totals {
	out := make(map[string]Totals, len(recs))
	for _, r := range recs {
		k := key(r)
		t := out[k] // missing key → zero Totals, no comma-ok needed here
		t.GPUHours += r.GPUHours
		t.USD += r.USD
		t.Records++
		out[k] = t // write the whole value back: out[k].USD += x does not compile
	}
	return out
}

// render sorts keys before printing — map iteration order is randomised.
func render(w *tabwriter.Writer, title string, totals map[string]Totals) {
	keys := make([]string, 0, len(totals)) // pre-size: len is known, no regrowth
	for k := range totals {
		keys = append(keys, k) // reassign the result — always
	}
	sort.Strings(keys)
	fmt.Fprintf(w, "%s\tGPU-HOURS\tUSD\tRECORDS\n", title)
	for _, k := range keys {
		t := totals[k]
		fmt.Fprintf(w, "%s\t%.1f\t%.2f\t%d\n", k, t.GPUHours, t.USD, t.Records)
	}
}

func main() {
	var recs []Record // nil slice: Decode will allocate and assign the result
	if err := json.NewDecoder(os.Stdin).Decode(&recs); err != nil {
		fmt.Fprintln(os.Stderr, "decode cost export:", err)
		os.Exit(1)
	}

	byNS := aggregate(recs, func(r Record) string { return r.Namespace })
	byTeam := aggregate(recs, func(r Record) string {
		if r.Team == "" { // omitempty on the way in means "" is what absent looks like
			return "(untagged)"
		}
		return r.Team
	})

	w := tabwriter.NewWriter(os.Stdout, 0, 4, 2, ' ', 0)
	render(w, "NAMESPACE", byNS)
	fmt.Fprintln(w, "\t\t\t")
	render(w, "TEAM", byTeam)
	w.Flush() // tabwriter buffers to compute column widths; without Flush, no output

	fmt.Fprintf(os.Stderr, "records=%d sizeof(Record)=%d bytes backing=%d bytes\n",
		len(recs), unsafe.Sizeof(Record{}), uintptr(cap(recs))*unsafe.Sizeof(Record{}))
}
```

Input (`sample.json`):

```json
[
  {"namespace":"prod-inference","team":"serving","gpu_hours":168.0,"usd":514.08,"preempted":false},
  {"namespace":"prod-inference","team":"serving","gpu_hours":72.0,"usd":220.32,"preempted":false},
  {"namespace":"research","team":"pretrain","gpu_hours":512.0,"usd":1566.72,"preempted":true},
  {"namespace":"research","gpu_hours":24.0,"usd":73.44,"preempted":false},
  {"namespace":"platform","team":"infra","gpu_hours":8.0,"usd":24.48,"preempted":false}
]
```

Captured run (go1.24.7):

```
$ go run . < sample.json
NAMESPACE       GPU-HOURS  USD      RECORDS
platform        8.0        24.48    1
prod-inference  240.0      734.40   2
research        536.0      1640.16  2

TEAM            GPU-HOURS  USD      RECORDS
(untagged)      24.0       73.44    1
infra           8.0        24.48    1
pretrain        512.0      1566.72  1
serving         240.0      734.40   2
records=5 sizeof(Record)=56 bytes backing=448 bytes
```

Reading the output against the lesson:

- **`research` = 536.0 GPU-hours, $1640.16** — the second research record has no `team` key at all, so `Team` decoded to `""` and the by-team aggregation bucketed it as `(untagged)`. The by-namespace total still includes it. That asymmetry is `omitempty` on the producer side turning "absent" and "empty" into the same thing; the exporter has to make an explicit decision about which bucket unlabelled spend lands in, and here it makes it visibly.
- **Both tables are sorted** because `sort.Strings(keys)` runs before printing. Delete that line and the row order changes between runs — the randomised map iteration shown earlier. A Prometheus scrape or a golden-file test that depends on line order would flap.
- **`sizeof(Record)=56`** — 16 (Namespace) + 16 (Team) + 8 + 8 = 48, then the `bool` at offset 48 plus 7 bytes of trailing padding to reach the struct's 8-byte alignment. Declare `Preempted` first instead and you get 64 bytes for the same data: `bool` at 0, 7 pad, then everything else. Same fields, +14%.
- **`backing=448` for 5 records** — `cap(recs)` is 8, not 5, because `json.Decode` builds the slice with `append` and the growth sequence went 1 → 2 → 4 → 8. Three slots of the backing array are unused. Harmless here; at 2.4 M records it is the difference between provisioning for `len` and provisioning for `cap`.

## Practice

Port an existing Python GPU-cost script to Go as the first cut of [`gpu-cost-exporter`](../practice/gpu-cost-exporter/README.md).

- Read a JSON cost export (array of records: namespace, a label/team field, gpu_hours, usd) from a file or stdin.
- Decode into a `[]Record` of typed structs with JSON tags. Deliberately choose, per numeric field, whether `omitempty`/`omitzero` is correct — and write a one-line comment on each saying why. A cost field where `0` is a real measurement must not be omitted.
- Aggregate cost two ways: by `namespace` and by `label`, using a `map[string]…`. Pre-size the maps with `make(..., len(recs))`.
- Print each aggregation as a table sorted by key (use `sort` + `text/tabwriter`).
- Use `(T, error)` returns for decode/parse; check every `err`.
- Add a small function that takes a `[]Record` and returns the N most recent records. Make it return an **independent** copy (`slices.Clone` or `copy` into a fresh slice) and write a test that mutates the returned slice and asserts the original is unchanged — then delete the clone, watch the test fail, and put it back.
- Once `Record` is stable, run `go build -gcflags="-m" ./...` and write down which allocations escape and why; then print `unsafe.Sizeof(Record{})`, reorder fields largest-alignment-to-smallest, and record the before/after size.

**Acceptance:** `go build ./...` succeeds; `go vet ./...` is clean; running against a sample export produces correct per-namespace and per-label totals in deterministic sorted order; the clone test passes; you can state, from your own `-gcflags=-m` output, one allocation that escapes and one that doesn't, and why. No nil-map panics, no reliance on append side effects, no `cap()` assertions in tests.

## Common pitfalls

1. **Assuming append mutation is always — or never — visible to the caller.** It is neither: visibility depends on whether the new length fits inside the existing capacity, which the call site cannot see. Symptom: a function "works" in tests with small inputs and silently corrupts data at scale (or vice versa), because capacity crossed a growth boundary. Mechanism: in-cap appends write into the shared backing array; over-cap appends allocate a new one. Fix: always `xs = append(xs, v)`, return slices you grow, and hand out capped windows with `xs[a:b:b]`.
2. **`omitempty` deleting a real zero.** A `float64` tagged `json:"usd,omitempty"` disappears from the JSON when the cost is exactly `$0.00`, indistinguishable on the wire from "no data collected." Mechanism: `encoding/json`'s `isEmptyValue` treats the numeric zero as empty. Fix: drop the option for measurement fields; use `omitzero` (Go 1.24+) only where zero genuinely means absent; use `*float64` when you must distinguish absent from zero on both encode and decode.
3. **Comparing structs that contain slices/maps with `==`.** Symptom: a build error (`struct containing []string cannot be compared`) — jarring if you're used to Python's always-works `==`. Mechanism: comparability is a type property; slices, maps, and funcs are not comparable, so any struct containing one isn't either. Fix: a hand-written `Equal` method, `slices.Equal`/`maps.Equal` on the offending field, or `reflect.DeepEqual` in tests only. Related and worse: `==` on two **interface** values whose dynamic type is uncomparable **panics at runtime** instead of failing to build.
4. **Treating nil and empty slices as interchangeable.** True for `len`, `range`, and `append`; false for `== nil` and for JSON, where nil marshals to `null` and empty to `[]`. Symptom: an API consumer branches differently for two runs that produced "no items". Fix: initialise output slices to `[]T{}` when the empty case must serialise as `[]`.
5. **`:=` inside `if`/`for` shadowing an outer variable.** `if err := f(); err != nil { … }` inside a function that already has an `err` creates a *second*, block-scoped `err`; assignments to it never reach the outer one. Symptom: "why is the outer error still nil?" Mechanism: every `if`, `for`, and `switch` is its own implicit block (Go spec, "Blocks"), and `:=` declares in the innermost block. Fix: `go vet -vettool=$(which shadow)` or `golangci-lint`'s shadow check; use `=` when you mean to assign to the existing variable.
6. **Writing to a map you never made.** Symptom: `panic: assignment to entry in nil map`, often in the one code path that skipped the constructor. Mechanism: the zero value of a map is a nil header — readable, not writable. Fix: `make` in the constructor, and prefer a `New…` function over a bare struct literal for any type with a map field.
7. **Holding a small window onto a huge array.** `recent := all[len(all)-10:]` keeps the entire backing array alive because the header still points into it. Symptom: RSS that never comes down after a big scrape. Fix: `slices.Clone(all[len(all)-10:])`.

## Self-check

- **When does passing a slice let the callee mutate the caller's backing array, and when not?**
  **Answer:** Passing a slice copies its 24-byte header (`ptr`, `len`, `cap`) — the backing array is shared. So mutating existing elements by index (`xs[i] = v`) is *always* visible to the caller. `append` is visible only if the new length still fits within the original `cap`, in which case it writes into the shared array's spare capacity; if the append exceeds `cap`, `growslice` allocates a new array, copies, and the callee's writes land somewhere the caller cannot see. Because the call site cannot know the capacity, the discipline is: always `xs = append(xs, v)`, return the grown slice, and pass `xs[a:b:b]` when you want to guarantee a callee's append cannot trespass.

- **Value vs pointer receiver — what breaks on a mutating method if you pick wrong?**
  **Answer:** A value receiver operates on a copy of the receiver, so field mutations are discarded when the method returns — no error, no warning, the caller's value is unchanged. A mutating method must use a pointer receiver. Two secondary breakages: (a) method sets — a method declared on `*T` is in `*T`'s method set but not `T`'s, so a `T` value will not satisfy an interface containing it (`cannot use Cost{} … Cost does not implement Discounter (method ApplyDiscount has pointer receiver)`); (b) addressability — the compiler auto-inserts `&` only for addressable operands, so `m["a"].Mutate()` on a `map[string]T` fails to build with `cannot call pointer method … on Cost`.

- **What is the zero value of a map, and what happens writing to a nil map?**
  **Answer:** The zero value is `nil` — a nil map header. Reads are safe and return the element type's zero value with `ok == false`, because the lookup path checks for the nil header and returns zero. Writes **panic** with `assignment to entry in nil map`, because there are no buckets to write into and the runtime will not silently allocate a map you did not ask for. Fix by `make(map[K]V)` (optionally pre-sized) or a `map[K]V{}` literal before the first write. The same asymmetry does not apply to slices: a nil slice is fully appendable.

- **Why can two structs fail to compile with `==`, and what do you use instead?**
  **Answer:** Comparability is a type-system property: a struct is comparable only if *every* field type is comparable, and slices, maps, and funcs are not comparable (they may only be compared to `nil`). So a struct with a `[]string` field is a build error on `==`, not a runtime surprise. Alternatives, best first: a hand-written `Equal(other T) bool` (explicit, fast, lets you define what equality means); `slices.Equal`/`maps.Equal` for the uncomparable field; `reflect.DeepEqual` for tests only (reflection-based, slow, and `DeepEqual([]int{}, []int(nil))` is `false`). The nastier cousin: comparing two **interface** values whose dynamic type is uncomparable compiles fine and panics at runtime.

- **What does `go build -gcflags="-m"` tell you, and name two patterns that force a heap escape.**
  **Answer:** It prints the compiler's escape-analysis decision per allocation — `does not escape`, `escapes to heap`, `moved to heap: x`, `leaking param: x`. The rule it applies: if the compiler cannot prove a value's lifetime ends with the function, the value must be heap-allocated. Two reliable triggers: (1) returning a pointer to a local (`return &r` → `moved to heap: r`), because the caller holds the address after the frame is gone; (2) boxing a concrete value into an interface (`fmt.Println(x)`, `any`, `slog` arguments), because the interface's data word must point at memory that outlives the call. A third worth knowing: `make([]T, n)` with a runtime `n` escapes, while `make([]T, 8)` with a constant can stay on the stack.

- **Why is field order in a struct not a cosmetic choice, and how do you find the waste?**
  **Answer:** Go lays fields out in declaration order (the layout is observable via `unsafe.Offsetof`, `encoding/binary`, and cgo, so the compiler must not reorder), and inserts padding so each field starts at an address matching its alignment, plus trailing padding so the struct size is a multiple of its own alignment. Declaring `bool, float64, bool` yields 24 bytes (7 pad + 7 pad); declaring `float64, bool, bool` yields 16. Order fields largest-alignment-first and let small fields pack together. Find it with `unsafe.Sizeof` plus the `fieldalignment` analyzer (`go vet -vettool=$(which fieldalignment) ./...`), and only act on hot, high-cardinality records — 8 wasted bytes across 2.4 M live records is ~19 MB of RSS and GC-scan work per replica.

## Connections & what's next

Value/pointer semantics and struct layout aren't a one-off topic — they're load-bearing for the rest of the module. [Lesson 3 (Interfaces & Composition)](03-interfaces-and-composition.md) leans directly on the method-set rules here ("methods on `*T` mean only `*T` satisfies the interface"), on the interface value's `{type, data}` pair that makes a typed nil non-nil, and on the escape-analysis reasoning behind "accept interfaces, return structs." [Lesson 4 (Concurrency & Context)](04-concurrency-and-context.md) depends on knowing precisely when a value is shared (and therefore needs synchronisation) versus copied (and therefore safe to hand to a goroutine without a lock) — the slice-header diagram above is exactly the picture you need when the race detector fires. [Lesson 6 (Modules & Layout)](06-modules-and-layout.md) revisits comparability and versioning gotchas at package boundaries. And every controller you read in [Lesson 9](09-controller-primer.md) is, mechanically, exactly the struct/slice/map/pointer vocabulary built here, just wearing a Kubernetes client-go costume — including the detail that a `Get` from the controller-runtime cache hands you a pointer into shared memory that you must `DeepCopy` before touching.

Next: [02 · Error Handling](02-error-handling.md) — now that values, pointers, and zero values are automatic, the next lesson builds the reconcile loop's real skill: classifying an error as skip, retry, or terminal, and making that classification survive being wrapped three call layers deep.

## References & further reading

**Primary sources**
- The Go Programming Language Specification — <https://go.dev/ref/spec> — canonical rules for zero values, comparability (including which types may only be compared to `nil`, and when an interface comparison panics), method sets for `T` vs `*T`, addressability, blocks and shadowing, and the size/alignment guarantees. Read the "Comparison operators", "Method sets", and "Size and alignment guarantees" sections in full; they are short and they settle arguments.
- `runtime/slice.go` — <https://github.com/golang/go/blob/master/src/runtime/slice.go> — the actual `growslice` and `nextslicecap` implementation: the 256-element doubling threshold, the `newcap += (newcap + 3*threshold) >> 2` smoothing above it, and the `roundupsize` step that re-derives capacity from the allocated byte count. This is where the "odd" capacities in this lesson come from.
- `runtime/sizeclasses.go` — <https://github.com/golang/go/blob/master/src/runtime/sizeclasses.go> — the allocator's size-class table (…4096, 4864, 5376, 6144, 6528, 6784…). Needed to reproduce the capacity arithmetic by hand.
- `encoding/json` package documentation — <https://pkg.go.dev/encoding/json> — the exact semantics of `omitempty` (and `isEmptyValue`) versus `omitzero`, struct-tag syntax, and decode behaviour for absent fields.
- Go 1.24 release notes — <https://go.dev/doc/go1.24> — the Swiss Tables map implementation and the new `omitzero` JSON tag option, both used above.
- Go 1.26 release notes — <https://go.dev/doc/go1.26> — notes that the compiler can now stack-allocate slice backing stores in more situations; a reason to re-run `-gcflags=-m` on your own toolchain rather than trusting any table of escape outcomes verbatim.
- `golang.org/x/tools` — `fieldalignment` analyzer — <https://github.com/golang/tools/tree/master/go/analysis/passes/fieldalignment> — the analyzer that reports structs whose size shrinks under reordering, plus the suggested order. Runnable standalone or via `go vet -vettool=`.

**Real-world engineering**
- Allegro Tech — "Golang slices gotcha" — <https://blog.allegro.tech/2017/07/golang-slices-gotcha.html> — what it shows: a ~year-long production bug in `marathon-consul` where appends onto a shared base slice gave two services identical Consul tags and broke service discovery. The ten-line reproduction in this lesson is the same mechanism.
- golang/go issue #66093 — <https://github.com/golang/go/issues/66093> — what it shows: capacity growth changed between Go 1.21.7 and 1.22.0 for large pointer-containing elements (32 → 37 instead of 64). Closed; the reporter's own note that "no one should rely on this property" is the correct takeaway. *(Corrected from an earlier version of this lesson, which described the issue as open.)*

**Deeper dives**
- The Go Blog — "Arrays, slices (and strings): The mechanics of `append`" — <https://go.dev/blog/slices> — the canonical narrative explanation of slice headers, sharing, and append. Read it after this lesson's diagrams for the same material in a different order.
- The Go Blog — "Faster Go maps with Swiss Tables" — <https://go.dev/blog/swisstable> — how the Go 1.24 map is structured (groups, control words, SIMD-style probing) and where the measured speedups come from. Optional depth on why iteration is still randomised.
- Learning Go, 2nd ed. (Jon Bodner), ch. 1–6 — <https://www.oreilly.com/library/view/learning-go-2nd/9781098139285/> — types, composite types, functions, pointers, methods. Bodner is explicit about *why* Go differs from other languages; the slice-aliasing and pointer-receiver sections are the sharpest treatment in print.
- Go by Example — <https://gobyexample.com> — copy-paste-ready idioms for maps, slices, structs, sorting. Keep it open while porting the script; ideal when you know the concept and just need the spelling.
