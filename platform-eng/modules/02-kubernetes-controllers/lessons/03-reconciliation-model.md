---
lesson: "02.3"
title: "The reconciliation model"
module: "02"
concept: "The reconciliation model"
status: not-started
est_time: "16h"
prev: "02-api-machinery.md"
next: "04-informers-caches-workqueues.md"
artifacts: []
sources: 6
---

# 02.3 · The reconciliation model

> **Concept.** Reconcile is level-triggered and idempotent — driven by observed cluster state, never by which event fired — plus the requeue/backoff paths and the failure modes where a controller silently stops reconciling.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 02 gave you the API-machinery substrate: how a GVK resolves to a GVR through the RESTMapper, how typed and dynamic clients read the same objects, and how a watch delivers `ADDED/MODIFIED/DELETED` events keyed to a `resourceVersion`. That lesson answered *how you talk to the API*. This lesson answers the harder question underneath every controller you'll ever write: *given that objects arrive as an unreliable, reorderable, coalescible stream of "something changed," what discipline makes your response to that stream correct?* The answer — level-triggered, idempotent reconciliation — is the single piece of engineering discipline that separates "a script that reacts to webhooks" from a Kubernetes controller. It's also the direct prerequisite for lesson 04: informers, caches, and workqueues are the *machinery* that enforces this discipline at the plumbing layer, and you can't reason about that machinery's design choices (dedup, resync, rate limiting) until you understand what correctness property they exist to protect.

## Why this matters

This is the highest interview-ROI concept in the module: "why must reconcile be idempotent?" and "walk me through the requeue paths and backoff" are asked in essentially every controller-building loop, and shallow answers get caught immediately. It's also where real operators break — a reconciler that branches on "which event fired" works in the demo and silently corrupts state in production the first time two edits coalesce, a resync fires, or the process restarts mid-loop. Your GPU cost operator writes `WorkloadCost` status on every observed change across 40 clusters; if that loop isn't provably idempotent and level-triggered, you'll get flapping status, wedged objects, runaway backoff you can't explain — and, because this is a *cost* operator, a double-counted GPU-hour is not an abstract correctness bug. It's a wrong invoice someone gets billed for.

## What's new here (calibration)

A CKA holder has *watched* reconciliation from outside: edit a Deployment's replica count, the ReplicaSet controller makes pods appear; delete a pod, it comes back. You know controllers "drive actual state toward desired state" as a slogan, and we won't re-teach what that slogan means at the kubectl-observation level — no YAML, no `kubectl describe` walkthroughs, no "what is a Deployment." What you learn now is the discipline that makes the slogan true in code *you* write:

- **Desired vs observed** isn't just spec/status labels — it's the invariant that your function reads `.spec` (intent, user-owned) and writes `.status` (observation, controller-owned) and never confuses the two.
- Reconcile is **level-triggered**: handed an object *key*, told to make the world consistent — never told *what changed*. The operator's intuition "it re-ran because I edited it" is a bug waiting to happen in the extender's code.
- The workqueue's **requeue/backoff** machinery — invisible when you operate — is now your retry strategy, and misusing it is how controllers wedge.
- This is also a *design*, not an accident: Kubernetes' controller pattern descends from Borg/Omega control-loop thinking, and understanding it as a deliberate choice (not just "how client-go happens to work") is what turns rote rule-following into transferable judgment.

## Core concepts

### Level-triggered, not edge-triggered

Edge-triggered = react to *events* ("a pod was created"). Level-triggered = react to *state* ("here is the object; is the world consistent with it?"). `Reconcile(ctx, req)` receives a `req.NamespacedName` — a *key*, not an event, not a diff. It must LIST/GET current state and converge.

This distinction isn't Kubernetes-specific — it's the same split as level-triggered vs edge-triggered I/O readiness (the difference between `epoll` telling you "the fd is still readable" vs "a readable event just occurred"). Framing it that way helps the idea transfer: any system where a consumer can miss, coalesce, or reorder notifications needs the level-triggered discipline, or it silently loses correctness under load. Kubernetes' choice to make controllers level-triggered was deliberate, not incidental — Tim Hockin (a founding Kubernetes engineer at Google) has an explicit talk on exactly this tradeoff, arguing edge-triggered systems are brittle under the realities of distributed delivery (dropped connections, restarts, slow consumers), while level-triggered systems self-heal from *any* missed notification because they never depend on having seen all of them (see References).

The workqueue is explicitly allowed to **coalesce** (ten rapid edits → one reconcile), **drop and replace** (only the latest matters), **reorder** across keys, and **replay** (periodic resync fires reconcile with nothing changed, restart re-LISTs everything). So the *only* correct model is: compute desired state from current inputs, make the world match, return. Any code shaped like "if this fired because X…" is broken by construction.

### Desired (spec) vs observed (status)

The user owns `.spec`; you own `.status`. Reconcile reads spec, does work, writes status — never the reverse, never its own spec. Keeping `/status` a subresource means a status write can't clobber a concurrent spec edit and vice-versa. Track `status.observedGeneration = obj.Generation` so you (and users) can tell whether the status reflects the *current* spec or a stale one.

### Idempotency — a more precise definition than "does nothing twice"

Because the same state is reconciled an unknown number of times, every reconcile must be safe to repeat with no cumulative effect. It is easy to over-simplify idempotency as "the second call does nothing" — that's wrong. Idempotency means **the same input produces the same result**, and reaching that result can still involve a write: a create-or-update that legitimately writes a drifted field back to its desired value is idempotent even though it mutates the object every time it's called, because the *result* — the target state — is identical regardless of how many times you call it or in what order. What's disallowed is a result that depends on *how many times* or *in what order* reconcile ran: "increment a counter," "append to a slice," "emit a bill" are not idempotent because the result accumulates.

Compute the target from inputs and *converge* — use create-or-update (`controllerutil.CreateOrUpdate`, or Server-Side Apply) rather than blind creates, and set status fields to computed values rather than mutating them incrementally. Idempotency is also what makes conflict-retry safe: a `409` just replays you.

### Tolerate stale reads

Your `Get`/`List` read from the informer **cache**, which lags the API server by a beat (lesson 04 covers exactly why). So: an object you just created may not be in cache yet; a delete may not have propagated; `observedGeneration` may trail. Never assume your last write is visible on the next reconcile — re-read and re-derive. This is a feature (the cache is why reconciles are cheap), and correct level-triggered code is naturally robust to it: if the read is stale, the next reconcile (triggered by the watch that eventually delivers the change) converges.

### The three requeue paths and how backoff differs

After controller-runtime calls your `Reconcile`, the return steers the **rate-limiting workqueue**:

1. **Return a non-nil `error`** → the item is requeued with **exponential backoff** from the rate limiter (controller-runtime's default is an item-exponential limiter, ~5ms up to ~1000s, combined with an overall bucket limiter). This is your retry path for transient failures.
2. **Return `ctrl.Result{RequeueAfter: d}, nil`** → requeued after a **fixed delay `d`**, *bypassing* the exponential backoff — for periodic polling of an external system (a cloud billing API). Success, just "check again later."
3. **Watch-driven re-enqueue** → you return `ctrl.Result{}, nil` (done) and the item is only re-queued when a watch (`For`/`Owns`/`Watches`) fires again. This is the steady state; most reconciles end here.

> `ctrl.Result{Requeue: true}` (the bare bool) is **deprecated** — return an `error` to get backoff, or set `RequeueAfter` to poll. Don't set `Requeue: true` alongside a non-nil error either; it's redundant and muddies intent.

### Where `Queue.Forget()` belongs

In raw client-go you manage the queue yourself, and the canonical pattern (sample-controller's `handleErr`) is: on **success**, `queue.Forget(key)` **then** `queue.Done(key)` — `Forget` resets that key's rate-limiter/backoff counter to zero so the next failure starts fresh; on **failure under the retry cap**, `queue.AddRateLimited(key)` (which *keeps* incrementing backoff); past the cap, `Forget` + drop + emit an event. The rule: **`Forget` on success or on give-up, never on a retry** — forgetting mid-retry would erase the backoff you're trying to accumulate. In controller-runtime this is done *for you*: returning `nil` with no `RequeueAfter` effectively forgets the key (resets backoff); returning an error keeps the backoff climbing. Knowing where `Forget` sits is exactly what "explain backoff" is probing.

### A known sharp edge

`RequeueAfter` does **not** reset the error backoff counter (controller-runtime [#392](https://github.com/kubernetes-sigs/controller-runtime/issues/392)) in older behavior — a reconcile that alternates errors and `RequeueAfter` can carry stale backoff. The practical guidance: don't mix an error return with `RequeueAfter` for the same logical outcome; pick one.

## Perspectives

**Theory perspective.** Level-triggered vs edge-triggered is a general distributed-systems idea, not a Kubernetes invention — it's the same distinction as level-triggered vs edge-triggered I/O interrupts, or `epoll`'s two readiness modes. Recognizing it as a transferable pattern (not Kubernetes trivia) means the discipline you learn here applies the next time you build any system that consumes an unreliable notification stream — a message-queue consumer, a file-watcher, a webhook receiver.

**Practice perspective.** The workqueue's coalesce/drop/reorder/replay behavior is what *forces* the idempotency discipline; it's not optional style, it's a correctness requirement given what the queue is allowed to do to your keys. You don't get to opt out of handling a coalesced double-edit — the machinery guarantees you'll see one eventually, so your reconcile has to be correct under it by construction.

**Failure-mode perspective.** Nearly every "controller stopped reconciling" incident traces to one of: swallowed error, missing watch, RBAC-blocked cache sync, wedged worker never calling `Done`, or backoff climbing toward the 1000s cap unnoticed. None of these are exotic — they're the direct, mechanical consequence of misunderstanding one of the mechanisms above.

**Economics perspective (module-specific).** An idempotency bug in a cost operator isn't just a correctness bug, it's a wrong invoice: a double-counted GPU-hour is money someone is billed incorrectly for, which is a much higher-stakes failure mode than a typical "my ReplicaSet count is briefly wrong." On a 40-cluster fleet, a systematic (not random) idempotency bug compounds silently across every cluster running the operator until someone notices the bill is wrong — by which point the wrong number may already be in a customer invoice.

## Real-world use cases

- **"Edge vs. Level Triggered Logic"** — Tim Hockin, Kubernetes founding engineer at Google. https://speakerdeck.com/thockin/edge-vs-level-triggered-logic — the design rationale for level-triggering straight from one of the engineers who helped choose it: why edge-triggered systems are brittle under real distributed-delivery failure modes, and why level-triggered systems self-heal instead. This is close to a primary source for *why* Kubernetes controllers are shaped the way they are, not just *that* they are.
- **"Migrating Uber's Compute Platform to Kubernetes: A Technical Journey"** — Uber Engineering. https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/ — describes Uber's own job controller reading a CRD object from the head of an event queue, determining its current state, and reconciling by performing an action — a real production description of exactly this lesson's Get → observe → compute → write shape, at Uber's scale.

## Worked example

Trace `WorkloadCost` through four reconciles to see level-triggered idempotency, backoff, and crash-recovery.

*Setup:* spec selects Pods by label; status reports `observedGPUCostUSD` and a `Ready` condition. Cost = (GPU count from Pod requests) × (rate from a `GPUCostPolicy`).

1. **User creates the `WorkloadCost`.** Watch enqueues the key. Reconcile: `Get` the object, `List` matching Pods (cache read), sum `nvidia.com/gpu` requests, multiply by policy rate → `observedGPUCostUSD = 12.00`. `Status().Update`. Return `{}, nil` → **path 3**, key forgotten, backoff zero.

2. **A user edits the object twice within 50ms** (tweaks the selector, then reverts). The workqueue **coalesces**: one key, dedup'd. Reconcile runs **once** against the final state — recomputes, gets the *same* `12.00`, writes identical status. Because the code derives cost from current Pods (not "the selector changed, so add a pod"), the double-edit is invisible and harmless. *This is the idempotency payoff.* Return `{}, nil`.

3. **The billing-rate policy lives in an external API that times out.** Reconcile calls it, gets an error, `return ctrl.Result{}, err` → **path 1**: requeued with exponential backoff (~5ms, 10ms, 20ms…). The transient clears on the 3rd try; reconcile succeeds, `return {}, nil` → backoff **reset to zero** (the controller-runtime equivalent of `Forget`). Had we instead swallowed the error and returned `{}, nil`, the queue would forget the key and the stale `12.00` would persist silently — a classic failure mode below.

4. **The controller process is killed between step 2 and step 3** (a pod eviction, a rolling update, an OOM). On restart: the informer relists every `WorkloadCost` and every Pod fresh from the API server, the workqueue is rebuilt from a synthetic `Sync` delta for every object, and reconcile runs again with the *exact same logic* — no special "recover from crash" branch anywhere in the code. It re-derives `12.00` from current Pods just as it did in step 1 and converges identically. This is the concrete payoff of level-triggering: crash recovery is *free* because there was never any in-memory state to lose in the first place — everything the reconciler needs is either in the object itself or re-derivable from a fresh LIST.

Note the shape every time: **Get → List/observe → compute from inputs → write status → return.** No branch on "what changed," no accumulated mutation. Run it a thousand times on one cluster state — or kill the process mid-loop and restart it — and the object lands identically. That's the whole discipline in one sentence.

## Practice

Take module 01's reconciler into `../practice/gpu-cost-operator/` and make it *provably* level-triggered and idempotent, then prove it with a test.

1. **Introduce the bug, then fix it.** Add a branch that assumes an edge — e.g. `if isCreate { status.Count++ }` or reading an "event type" and only computing on `MODIFIED`. Confirm it's wrong by reasoning: a coalesced double-edit or a resync double-counts / skips.
2. **Refactor to converge.** Replace the branch with a pure recompute from current inputs: derive every status field from the object + listed dependents, set (not mutate) it, `Status().Update`. Set `status.observedGeneration = obj.Generation`.
3. **Table-driven idempotency test** (envtest or a fake client, `sigs.k8s.io/controller-runtime/pkg/client/fake`). For a set of cluster states (no pods, one GPU pod, several, selector-miss), call `Reconcile` on the same key **twice back-to-back** and assert the resulting status is **byte-identical** after the second call — and identical regardless of a pre-seeded stale status. Assert `observedGeneration` matches.
4. **Backoff assertion (optional, high-value):** inject a dependency that errors once then succeeds; assert the first `Reconcile` returns a non-nil error (→ backoff) and the second returns `nil` with converged status.

**Acceptance:**
- Idempotency test is **green**: two consecutive reconciles on identical state yield identical status; pre-seeded stale status is overwritten to the same value.
- The "branch on event" bug is present in git history (one commit) and removed in the next — a visible before/after.
- No status field is mutated incrementally; all are computed from inputs. `observedGeneration` is asserted.

## Common pitfalls

1. **Believing "idempotent" means "does nothing on the second call."** It means "produces the same *result* on the second call," which can still involve writes — e.g. a create-or-update that legitimately updates a drifted field back to desired every time it runs. What's disallowed is a result that accumulates or depends on call count.
2. **Swallowed error / eternal success.** You `log.Error(err)` but `return ctrl.Result{}, nil`. The queue thinks you succeeded, forgets the key, never retries — the object sits un-reconciled until an unrelated event pokes it. Correction: return the error.
3. **Missing watch.** Your reconcile depends on a secondary object (a ConfigMap, a Node) you never registered via `Owns`/`Watches`. Changes to it don't enqueue the primary, so status goes stale silently. Correction: wire every input as a watch.
4. **Cache never synced / RBAC gap.** The informer can't `list`/`watch` the type (missing RBAC verb), so the cache never fills, `WaitForCacheSync` blocks, and no reconciles start. Envtest won't catch it (no RBAC enforcement); prod will. Correction: audit RBAC against every type your controller reads or watches.
5. **Conflating resync with a health check.** A resync firing doesn't mean anything was wrong — it's a local safety-net replay (lesson 04), not a "something broke" signal. Treating resync-triggered reconciles as anomalies to investigate wastes on-call time chasing nothing.

## Self-check

- **Why must you never branch on which event fired? Give a concrete bug.**
  **Answer:** Because the workqueue gives you a *key*, not an event or a diff, and it may coalesce, drop, reorder, and replay — you are never reliably told what changed or how many times. Concrete bug: a reconciler that does `if createEvent { status.PodCount++ }`. Two rapid edits coalesce into one reconcile, so a create+update counts once instead of twice; a periodic resync fires with no change and increments again; a controller restart re-LISTs and increments from a fresh baseline. The count drifts arbitrarily. The fix is to *derive* `PodCount` by listing pods every reconcile and setting it — no branch on event, no accumulation.

- **The three requeue paths and how backoff differs; where does `Queue.Forget()` belong?**
  **Answer:** (1) Return a non-nil **error** → requeued with **exponential backoff** (the rate limiter climbs per key) — the transient-failure retry. (2) `Result{RequeueAfter: d}, nil` → requeued after a **fixed delay**, bypassing exponential backoff — for periodic polling. (3) `Result{}, nil` → **no requeue** except when a watch fires again — the steady state. `Queue.Forget(key)` belongs on **success** (reset backoff to zero) and on **permanent give-up** (past the retry cap, drop it) — *never* on a retry, since forgetting mid-retry erases the backoff you're accumulating. controller-runtime does this implicitly: `nil`/no-requeue resets backoff, an error keeps it climbing.

- **Reconcile called twice for the same state — what must be true for that to be harmless?**
  **Answer:** The reconcile must be **idempotent**: it derives all desired state (and status) purely from current inputs and *converges* to it — create-or-update rather than blind create, set-to-computed-value rather than increment, no side effect that accumulates across runs. It must also **tolerate stale cache reads** (re-read, don't assume the last write is visible) and handle `409 Conflict` by re-reading rather than treating it as fatal. Given those, the second call recomputes the identical target, finds the world already matches, writes the same status, and is a no-op in effect — exactly what resyncs, requeues, and restarts rely on.

- **Your controller's backoff has climbed toward the ~1000s cap for one object but every other object reconciles fine. What series of `AddRateLimited` calls got it there, and what's the controller-runtime-idiomatic way to reset it once the underlying cause is fixed?**
  **Answer:** Every reconcile of that specific key returned a non-nil error (or `Result{Requeue: true}`), so the workqueue called `AddRateLimited` for that key repeatedly, and the item-exponential limiter doubled the delay each time (`baseDelay × 2^numRequeues`, e.g. 5ms → 10ms → 20ms → … up toward the ~1000s cap) — only that key's counter climbs, because backoff state is per-key, not global. The idiomatic reset is simply to make one reconcile of that key **succeed**: return `ctrl.Result{}, nil` (or a valid `RequeueAfter`) with no error. controller-runtime calls the equivalent of `Forget` on that path, zeroing the key's backoff counter, so the very next failure (if any) starts again from `baseDelay`. There's no separate "reset backoff" API to call directly — a clean success is the only sanctioned way to clear it.

## Connections & what's next

This lesson is the correctness contract; lesson 02 (API machinery) gave you the wire format and client types it's written in, and lesson 06 (controller-runtime deep) will show you the same discipline extended to owned children and finalizers, where "idempotent" has to cover creating/updating/deleting *other* objects, not just writing your own status. The immediate next step is **lesson 04, Informers, caches, and workqueues** — the machinery that actually *implements* everything you took on faith here: how a key gets coalesced, how backoff is computed and stored per-key, why a cache read can be stale, and where `WaitForCacheSync` sits in the startup path. Everything in this lesson was a black-box contract ("the queue may coalesce, drop, reorder, replay"); lesson 04 opens the box.

## References & further reading

**Primary sources**
- Kubebuilder Book — Architecture / reconcile — https://book.kubebuilder.io/architecture.html — the canonical diagram of the manager → cache → workqueue → reconcile flow and the level-triggered contract in the framework you'll ship.
- sample-controller — `controller.go` — https://github.com/kubernetes/sample-controller/blob/master/controller.go — the reconcile loop written in raw client-go, showing `AddRateLimited` / `Forget` / `Done` explicitly; read `processNextWorkItem` and `handleErr` to see where `Forget` sits by hand.

**Real-world engineering blogs**
- Tim Hockin, "Edge vs. Level Triggered Logic" — https://speakerdeck.com/thockin/edge-vs-level-triggered-logic — the design rationale for level-triggering, from a founding Kubernetes engineer.
- Uber Engineering, "Migrating Uber's Compute Platform to Kubernetes: A Technical Journey" — https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/ — a real production job-controller reconcile loop at scale.

**Deeper dives**
- controller-runtime issue #392 — https://github.com/kubernetes-sigs/controller-runtime/issues/392 — the `RequeueAfter`-doesn't-reset-backoff gotcha, discussed by maintainers.
- Stuart Leeks, "Error back-off with controller-runtime" — https://stuartleeks.com/posts/error-back-off-with-controller-runtime/ — the same gotcha worked through with code; the difference between "I know the requeue paths" and "I've been bitten by how they interact."
