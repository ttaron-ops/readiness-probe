---
lesson: "02.3"
title: "The reconciliation model"
module: "02"
concept: "The reconciliation model"
status: not-started
est_time: "8h"
artifacts: []
---

# 02.3 · The reconciliation model

> **Concept.** Reconcile is level-triggered and idempotent — driven by observed cluster state, never by which event fired — plus the requeue/backoff paths and the failure modes where a controller silently stops reconciling.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Why this matters

This is the highest interview-ROI concept in the module: "why must reconcile be idempotent?" and "walk me through the requeue paths and backoff" are asked in essentially every controller-building loop, and shallow answers get caught immediately. It's also where real operators break — a reconciler that branches on "which event fired" works in the demo and silently corrupts state in production the first time two edits coalesce. Your GPU cost operator writes `WorkloadCost` status on every observed change across 40 clusters; if that loop isn't provably idempotent and level-triggered, you'll get flapping status, wedged objects, and runaway backoff you can't explain.

## From operating to extending

A CKA holder has *watched* reconciliation from outside: edit a Deployment's replica count, the ReplicaSet controller makes pods appear; delete a pod, it comes back. You know controllers "drive actual state toward desired state" as a slogan. What you learn now is the discipline that makes that slogan true in code you write:

- **Desired vs observed** isn't just spec/status labels — it's the invariant that your function reads `.spec` (intent, user-owned) and writes `.status` (observation, controller-owned) and never confuses the two.
- Reconcile is **level-triggered**: handed an object *key*, told to make the world consistent — never told *what changed*. The operator's intuition "it re-ran because I edited it" is a bug waiting to happen in the extender's code.
- The workqueue's **requeue/backoff** machinery — invisible when you operate — is now your retry strategy, and misusing it is how controllers wedge.

## Core notes

**Level-triggered, not edge-triggered.** Edge-triggered = react to *events* ("a pod was created"). Level-triggered = react to *state* ("here is the object; is the world consistent with it?"). `Reconcile(ctx, req)` receives a `req.NamespacedName` — a *key*, not an event, not a diff. It must LIST/GET current state and converge. The workqueue is explicitly allowed to **coalesce** (ten rapid edits → one reconcile), **drop and replace** (only the latest matters), **reorder** across keys, and **replay** (periodic resync fires reconcile with nothing changed, restart re-LISTs everything). So the *only* correct model is: compute desired state from current inputs, make the world match, return. Any code shaped like "if this fired because X…" is broken by construction.

**Desired (spec) vs observed (status).** The user owns `.spec`; you own `.status`. Reconcile reads spec, does work, writes status — never the reverse, never its own spec. Keeping `/status` a subresource means a status write can't clobber a concurrent spec edit and vice-versa. Track `status.observedGeneration = obj.Generation` so you (and users) can tell whether the status reflects the *current* spec or a stale one.

**Idempotency.** Because the same state is reconciled an unknown number of times, every reconcile must be safe to repeat with no cumulative effect. "Ensure X exists; if present, leave it" is idempotent. "Increment a counter," "append to a slice," "emit a bill" is not. Compute the target from inputs and *converge* — use create-or-update (`controllerutil.CreateOrUpdate`, or Server-Side Apply) rather than blind creates, and set status fields to computed values rather than mutating them incrementally. Idempotency is also what makes conflict-retry safe: a `409` just replays you.

**Tolerate stale reads.** Your `Get`/`List` read from the informer **cache**, which lags the API server by a beat. So: an object you just created may not be in cache yet; a delete may not have propagated; `observedGeneration` may trail. Never assume your last write is visible on the next reconcile — re-read and re-derive. This is a feature (the cache is why reconciles are cheap), and correct level-triggered code is naturally robust to it: if the read is stale, the next reconcile (triggered by the watch that eventually delivers the change) converges.

**The three requeue paths and how backoff differs.** After controller-runtime calls your `Reconcile`, the return steers the **rate-limiting workqueue**:

1. **Return a non-nil `error`** → the item is requeued with **exponential backoff** from the rate limiter (controller-runtime's default is an item-exponential limiter, ~5ms up to ~1000s, combined with an overall bucket limiter). This is your retry path for transient failures.
2. **Return `ctrl.Result{RequeueAfter: d}, nil`** → requeued after a **fixed delay `d`**, *bypassing* the exponential backoff — for periodic polling of an external system (a cloud billing API). Success, just "check again later."
3. **Watch-driven re-enqueue** → you return `ctrl.Result{}, nil` (done) and the item is only re-queued when a watch (`For`/`Owns`/`Watches`) fires again. This is the steady state; most reconciles end here.

> `ctrl.Result{Requeue: true}` (the bare bool) is **deprecated** — return an `error` to get backoff, or set `RequeueAfter` to poll. Don't set `Requeue: true` alongside a non-nil error either; it's redundant and muddies intent.

**Where `Queue.Forget()` belongs.** In raw client-go you manage the queue yourself, and the canonical pattern (sample-controller's `handleErr`) is: on **success**, `queue.Forget(key)` **then** `queue.Done(key)` — `Forget` resets that key's rate-limiter/backoff counter to zero so the next failure starts fresh; on **failure under the retry cap**, `queue.AddRateLimited(key)` (which *keeps* incrementing backoff); past the cap, `Forget` + drop + emit an event. The rule: **`Forget` on success or on give-up, never on a retry** — forgetting mid-retry would erase the backoff you're trying to accumulate. In controller-runtime this is done *for you*: returning `nil` with no `RequeueAfter` effectively forgets the key (resets backoff); returning an error keeps the backoff climbing. Knowing where `Forget` sits is exactly what "explain backoff" is probing.

**A known sharp edge:** `RequeueAfter` does **not** reset the error backoff counter (controller-runtime [#392](https://github.com/kubernetes-sigs/controller-runtime/issues/392)) in older behavior — a reconcile that alternates errors and `RequeueAfter` can carry stale backoff. The practical guidance: don't mix an error return with `RequeueAfter` for the same logical outcome; pick one.

**Classic "controller stopped reconciling" failure modes:**

- **Swallowed error / eternal success.** You `log.Error(err)` but `return ctrl.Result{}, nil`. The queue thinks you succeeded, forgets the key, never retries — the object sits un-reconciled until an unrelated event pokes it. Return the error.
- **Missing watch.** Your reconcile depends on a secondary object (a ConfigMap, a Node) you never registered via `Owns`/`Watches`. Changes to it don't enqueue the primary, so status goes stale silently. Wire every input as a watch.
- **Cache never synced / RBAC gap.** The informer can't `list`/`watch` the type (missing RBAC verb), so the cache never fills, `WaitForCacheSync` blocks, and no reconciles start. Envtest won't catch it (no RBAC enforcement); prod will.
- **Wedged on conflict.** You treat `409 Conflict` as fatal and hot-loop, or you never re-`Get` before `Update` so every write conflicts. Re-read on conflict; let backoff space the retries.
- **Panic in reconcile.** An unrecovered panic can take the worker down; controller-runtime recovers per-reconcile by default, but a panic still means that reconcile did no work — guard nil map/pointer access on partial objects.
- **Blocking the worker.** A long synchronous call (no `ctx` deadline) holds a worker slot; with `MaxConcurrentReconciles` small, throughput collapses. Thread `ctx`, set timeouts.

## Worked example

Trace `WorkloadCost` through three reconciles to see level-triggered idempotency and backoff.

*Setup:* spec selects Pods by label; status reports `observedGPUCostUSD` and a `Ready` condition. Cost = (GPU count from Pod requests) × (rate from a `GPUCostPolicy`).

1. **User creates the `WorkloadCost`.** Watch enqueues the key. Reconcile: `Get` the object, `List` matching Pods (cache read), sum `nvidia.com/gpu` requests, multiply by policy rate → `observedGPUCostUSD = 12.00`. `Status().Update`. Return `{}, nil` → **path 3**, key forgotten, backoff zero.

2. **A user edits the object twice within 50ms** (tweaks the selector, then reverts). The workqueue **coalesces**: one key, dedup'd. Reconcile runs **once** against the final state — recomputes, gets the *same* `12.00`, writes identical status. Because the code derives cost from current Pods (not "the selector changed, so add a pod"), the double-edit is invisible and harmless. *This is the idempotency payoff.* Return `{}, nil`.

3. **The billing-rate policy lives in an external API that times out.** Reconcile calls it, gets an error, `return ctrl.Result{}, err` → **path 1**: requeued with exponential backoff (~5ms, 10ms, 20ms…). The transient clears on the 3rd try; reconcile succeeds, `return {}, nil` → backoff **reset to zero** (the controller-runtime equivalent of `Forget`). Had we instead swallowed the error and returned `{}, nil`, the queue would forget the key and the stale `12.00` would persist silently — failure mode #1.

Note the shape every time: **Get → List/observe → compute from inputs → write status → return.** No branch on "what changed," no accumulated mutation. Run it a thousand times on one cluster state and the object lands identically — the whole discipline in one sentence.

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

## Self-check

**(a) Why must you never branch on which event fired? Give a concrete bug.**
**Answer:** Because the workqueue gives you a *key*, not an event or a diff, and it may coalesce, drop, reorder, and replay — you are never reliably told what changed or how many times. Concrete bug: a reconciler that does `if createEvent { status.PodCount++ }`. Two rapid edits coalesce into one reconcile, so a create+update counts once instead of twice; a periodic resync fires with no change and increments again; a controller restart re-LISTs and increments from a fresh baseline. The count drifts arbitrarily. The fix is to *derive* `PodCount` by listing pods every reconcile and setting it — no branch on event, no accumulation.

**(b) The three requeue paths and how backoff differs; where does `Queue.Forget()` belong?**
**Answer:** (1) Return a non-nil **error** → requeued with **exponential backoff** (the rate limiter climbs per key) — the transient-failure retry. (2) `Result{RequeueAfter: d}, nil` → requeued after a **fixed delay**, bypassing exponential backoff — for periodic polling. (3) `Result{}, nil` → **no requeue** except when a watch fires again — the steady state. `Queue.Forget(key)` belongs on **success** (reset backoff to zero) and on **permanent give-up** (past the retry cap, drop it) — *never* on a retry, since forgetting mid-retry erases the backoff you're accumulating. controller-runtime does this implicitly: `nil`/no-requeue resets backoff, an error keeps it climbing.

**(c) Reconcile called twice for the same state — what must be true for that to be harmless?**
**Answer:** The reconcile must be **idempotent**: it derives all desired state (and status) purely from current inputs and *converges* to it — create-or-update rather than blind create, set-to-computed-value rather than increment, no side effect that accumulates across runs. It must also **tolerate stale cache reads** (re-read, don't assume the last write is visible) and handle `409 Conflict` by re-reading rather than treating it as fatal. Given those, the second call recomputes the identical target, finds the world already matches, writes the same status, and is a no-op in effect — exactly what resyncs, requeues, and restarts rely on.

## Resources

- **Kubebuilder Book — Architecture / reconcile** (https://book.kubebuilder.io/architecture.html). *What-for:* the canonical diagram of the manager → cache → workqueue → reconcile flow and the level-triggered contract in the framework you'll ship. **Skim now**, re-read once you've written the idempotency test. *Why:* it's the mental model your capstone is built on, stated authoritatively.
- **sample-controller — `controller.go`** (https://github.com/kubernetes/sample-controller/blob/master/controller.go). *What-for:* the reconcile loop written in raw client-go, showing `AddRateLimited` / `Forget` / `Done` explicitly — the machinery controller-runtime hides. **Deep-read** `processNextWorkItem` and `handleErr`. *Why:* seeing where `Forget` sits by hand is exactly what "explain backoff" interviews probe, and it demystifies controller-runtime's implicit version.
- **controller-runtime issue #392 + Stuart Leeks' backoff post** (https://github.com/kubernetes-sigs/controller-runtime/issues/392, https://stuartleeks.com/posts/error-back-off-with-controller-runtime/). *What-for:* the real-world gotcha that `RequeueAfter` doesn't reset error backoff, worked through with code. **Skim** the issue, **read** the post. *Why:* it's the difference between "I know the requeue paths" and "I've been bitten by how they interact" — a senior-level distinction.
