# 🐹 Checkpoint — 01 · Go for infrastructure engineers

This is the **completion gate**. Answer the probes from memory, and prove the rest
with the [`gpu-cost-exporter`](practice/gpu-cost-exporter/) deliverable. You've
**passed the Go gate** when items 1–5 are true; 6–7 mean you're ready to start the
flagship operator.

## Pass criteria

- [ ] **1 · Ships.** `gpu-cost-exporter` builds, runs, and serves a correct
      Prometheus gauge on `/metrics`; `go vet` and `golangci-lint` clean.
- [ ] **2 · Tested.** Table-driven tests on core logic, a fake data source (no mock
      library), ≥ ~70% coverage on business logic, one benchmark, passes `go test -race`.
- [ ] **3 · Idiomatic.** Data source behind an interface; errors wrapped with `%w`
      and classified (retryable vs terminal); `context` threaded through all I/O with
      a working deadline/cancellation; structured `slog` logging.
- [ ] **4 · Concurrency.** The scrape is a bounded worker pool that cancels cleanly on
      context cancellation and is demonstrably leak-free (`-race` clean).
- [ ] **5 · Reading proof.** A one-page written trace (committed under
      `practice/`) of how controller-runtime's `Reconcile` gets invoked when a watched
      object changes — citing specific files/types.
- [ ] **6 · Controller proof.** A kubebuilder-scaffolded CRD + reconciler that passes
      an `envtest` test driving `.status` toward `.spec` (Lesson 09 practice).
- [ ] **7 · Self-explain.** You can answer these **cold**:

## Depth probes (answer without looking anything up)

**Types & semantics**
- [ ] When does passing a slice let the callee mutate the caller's backing array — and when not?
- [ ] Pointer vs value receiver on a mutating method: what breaks if you pick wrong?
- [ ] Why can an interface holding a nil pointer not equal `nil`?

**Errors**
- [ ] `errors.Is` vs `errors.As` — when each?
- [ ] With `%w` wrapping three layers deep, how does a caller detect a specific sentinel?
- [ ] When is `panic` acceptable in a long-running controller?

**Concurrency & context**
- [ ] Cap concurrency to N and cancel all in-flight work on first error — how? (`errgroup`)
- [ ] Show a goroutine leak and its fix.
- [ ] Why is `ctx` always the first argument, and what must a blocking call do with it?

**Interfaces**
- [ ] Why define interfaces on the consumer, not the producer?
- [ ] How do you fake an external dependency with zero mocking library?

**Controllers**
- [ ] Why must `Reconcile` be idempotent and level-triggered, not edge-triggered?
- [ ] Spec vs status ownership — why is status a subresource?
- [ ] How does the controller's cache/informer avoid hammering the API server?

## Answers / notes

_Record your answers here as you close each lesson, and link the deliverable code
that demonstrates items 1–6._
