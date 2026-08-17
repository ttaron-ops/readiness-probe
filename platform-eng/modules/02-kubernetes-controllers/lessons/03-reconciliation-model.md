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
sources: 13
---

# 02.3 · The reconciliation model

> **Concept.** Reconcile is level-triggered and idempotent — driven by observed cluster state, never by which event fired — plus the requeue/backoff paths and the failure modes where a controller silently stops reconciling.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 02 gave you the API-machinery substrate: how a GVK resolves to a GVR through the RESTMapper, how typed and dynamic clients read the same objects, how `generation` and `observedGeneration` track intent versus observation, and how a watch delivers `ADDED`/`MODIFIED`/`DELETED` events keyed to a `resourceVersion`. That lesson answered *how you talk to the API*. This lesson answers the harder question underneath every controller you will ever write: *given that objects arrive as an unreliable, reorderable, coalescible stream of "something changed," what discipline makes your response to that stream correct?*

The answer — level-triggered, idempotent reconciliation, plus an explicit deletion lifecycle — is the single piece of engineering discipline that separates "a script that reacts to webhooks" from a Kubernetes controller. It is also the direct prerequisite for lesson 04: informers, caches, and workqueues are the *machinery* that enforces this discipline at the plumbing layer, and you cannot reason about that machinery's design choices (dedup, resync, rate limiting) until you understand what correctness property they exist to protect.

## Why this matters

This is the highest interview-ROI concept in the module. "Why must reconcile be idempotent?" and "walk me through the requeue paths and backoff" are asked in essentially every controller-building loop, and shallow answers get caught immediately. It is also where real operators break — a reconciler that branches on "which event fired" works in the demo and silently corrupts state in production the first time two edits coalesce, a resync fires, or the process restarts mid-loop.

Your GPU cost operator writes `WorkloadCost` status on every observed change across 40 clusters. If that loop is not provably idempotent and level-triggered, you get flapping status, wedged objects, runaway backoff you cannot explain — and, because this is a *cost* operator, a double-counted GPU-hour is not an abstract correctness bug. It is a wrong invoice someone gets billed for. The finalizer half matters just as much: your operator records final spend on deletion, and a finalizer that can hang is a finalizer that leaves objects, namespaces, and eventually whole clusters wedged in `Terminating`.

## What's new here (calibration)

A CKA holder has *watched* reconciliation from the outside: edit a Deployment's replica count, the ReplicaSet controller makes pods appear; delete a pod, it comes back. You know controllers "drive actual state toward desired state" as a slogan, and this lesson does not re-teach that slogan at the kubectl-observation level — no YAML basics, no `kubectl describe` walkthroughs, no "what is a Deployment." What you learn now is the discipline that makes the slogan true in code *you* write:

- **The delivery contract.** Exactly what the queue is allowed to do to your events — coalesce, drop, reorder, replay, restart — and why every one of those permissions is load-bearing rather than a wart.
- **Level-triggered as a design choice.** Reconcile is handed an object *key* and told to make the world consistent; it is never told *what changed*. The operator's intuition "it re-ran because I edited it" is a bug waiting to happen in the extender's code.
- **Idempotency, precisely.** Not "does nothing the second time" but "produces the same result regardless of call count or order" — including the hard cases: accumulators, external billing calls, and list-valued status.
- **The requeue/backoff machinery** — invisible when you operate — is now your retry strategy, with exact numbers, exact branches in controller-runtime's source, and exact metrics.
- **The deletion lifecycle.** `deletionTimestamp`, finalizers, cascade propagation, and the stuck-finalizer failure mode that turns a five-second delete into a multi-hour incident.

Version note: controller-runtime behaviour below is verified against the `main` branch of `kubernetes-sigs/controller-runtime` and `kubernetes/client-go` as of August 2026. One behaviour documented in older material has changed and is corrected explicitly in the backoff section. Transcripts are **representative** — real in format, realistic in values — rather than literal captures.

## Core concepts

### The delivery contract: what the machinery is allowed to do to you

Start with the thing your reconciler must survive, because the discipline is derived from it and not the other way round. The path from "someone changed an object" to "your function runs" is: etcd → apiserver watch cache → your informer's reflector → DeltaFIFO → event handler → workqueue → worker goroutine → `Reconcile(ctx, req)`. Every hop in that chain is allowed to do things to your events:

| The machinery may… | Because | What that forbids in your code |
|---|---|---|
| **Coalesce** — ten edits in 50 ms become one call | The workqueue's `dirty` set holds a key at most once; `Add` on an item already dirty is a no-op | Any per-event accounting. You will not see all N edits. |
| **Drop and replace** — only the latest state matters | The queue holds keys, not object copies; the worker re-reads current state | Reading "the object as it was when the event fired." That object is gone. |
| **Reorder** across keys | Multiple workers (`MaxConcurrentReconciles`) run concurrently on different keys | Any cross-object ordering assumption. Object B may reconcile before object A even if A changed first. |
| **Replay** — reconcile with nothing changed | Periodic resync replays the *local cache* into the handlers; `RequeueAfter`; a `409` retry | Any side effect that is not safe to repeat. |
| **Restart from zero** | Process crash, eviction, rolling update, lost leader lease → new process relists everything and reconciles all objects | Any in-memory state that is not re-derivable from the API. |
| **Deliver late** | Cache lag, backoff, APF throttling | Any assumption that your last write is visible on the next read. |

It cannot, however, *lose* a change permanently: whatever the last state is, some future reconcile will observe it, because the informer's resync and relist guarantee that current state is eventually re-presented. That asymmetry — "may lose an event, may not lose the state" — is the whole basis of the design.

### Level-triggered, not edge-triggered

Edge-triggered means reacting to *transitions*: "a pod was created," "the replica count went from 3 to 5." Level-triggered means reacting to *state*: "here is the object; make the world consistent with it." `Reconcile(ctx, req)` receives a `reconcile.Request`, which is nothing but a `types.NamespacedName`. controller-runtime's own doc comment is unambiguous:

> *Reconciliation is level-based, meaning action isn't driven off changes in individual Events, but instead is driven by actual cluster state read from the apiserver or a local cache.* … The Request contains "the information to uniquely identify the object — its Name and Namespace" and does **not** include "information about any specific Event or the object contents itself."

The distinction is not Kubernetes-specific. It is the same split as edge- versus level-triggered hardware interrupts, and the same split as `epoll`'s `EPOLLET` versus its default mode. Tim Hockin — a founding Kubernetes engineer at Google — made exactly that analogy in his 2017 talk "Edge vs. Level triggered logic," arguing that level-driven software is about *asserting state* so that a client can check and re-check at any time, and that this is the heart of the Kubernetes controller model. Edge-triggered systems are brittle under the realities of distributed delivery: a dropped connection, a slow consumer, or a restart loses an edge permanently, and there is no way to recover the lost work because the signal that carried it is gone.

Here is the same event timeline delivered to both designs. This is the argument in one picture.

```
  WALL CLOCK ──────────────────────────────────────────────────────────────────────────▶
             t0        t1        t2        t3        t4        t5        t6        t7

  WHAT ACTUALLY HAPPENED IN THE CLUSTER
             pod-a     pod-b     pod-b     pod-c     [controller     pod-c     [resync
             created   created   deleted   created    RESTARTS]      deleted    fires]
  true GPU
  pod count:    1         2         1         2          2             1          1

  ── EDGE-TRIGGERED CONTROLLER ────────────────────────────────────────────────────────
     handler:  count++    count++   count--   count++   (in-memory     count--   (nothing:
                                                         count LOST,             no edge)
                                                         restarts at 0)
     internal
     count:       1         2         1         2      ──▶ 0    ──▶    -1        -1
                                                            ▲            ▲        ▲
                                                            │            │        │
                                              STATE DESTROYED│    NEGATIVE│  STAYS WRONG
                                              by the restart │    COUNT   │  FOREVER —
                                                             │            │  nothing ever
                                       ALSO: if t1 and t2 had arrived     │  re-asserts
                                       within one queue window they would │  the truth
                                       have COALESCED into one edge, and  │
                                       the count would be wrong even      │
                                       without a restart.                 │

  ── LEVEL-TRIGGERED CONTROLLER ───────────────────────────────────────────────────────
     handler:  LIST pods matching selector; count = len(gpu pods); write status
     computed
     count:       1         2         1         2      ──▶ 2    ──▶     1         1
                                                            ▲                     ▲
                                                            │                     │
                                            RESTART IS A NO-OP:      RESYNC IS A NO-OP:
                                            the new process LISTs    recomputes the same
                                            and recomputes 2.        answer, writes the
                                            No recovery code.        same value.

  ⇒ A MISSED EDGE IS FATAL TO ONE DESIGN AND INVISIBLE TO THE OTHER.
    The level-triggered controller never asks "what changed"; it asks "what is true now".
    That is why it needs no crash-recovery branch, no event log, no replay buffer,
    and no exactly-once delivery from the transport.
```

Notice what the level-triggered column did *not* need: no persistence, no idempotency token on the wire, no acknowledgement protocol, no "reconcile from checkpoint" branch. Crash recovery is free because there was never any in-memory state to lose. That is the payoff you are buying with the discipline, and it is why every mature Kubernetes controller — including the built-ins in kube-controller-manager — is written this way.

The corollary is a hard rule: **any code shaped like `if <this event was a create> { … }` is broken by construction**, because the input you are branching on does not exist in `reconcile.Request`. If you find yourself wanting it, you are trying to write an edge-triggered controller inside a level-triggered framework, and the framework will win.

### Desired (spec) versus observed (status)

The user owns `.spec`; your controller owns `.status`. Reconcile reads spec, does work, writes status — never the reverse, and never its own spec. The API conventions state the rule for the server side too: `PUT` and `POST` on the main resource must ignore `status` values, specifically to avoid clobbering status in read-modify-write flows.

Making `/status` a subresource (lesson 02's `subresources: {status: {}}` line in the CRD) buys three things at once:

1. A status write cannot clobber a concurrent spec edit, and vice versa — they are separate endpoints with separate optimistic-concurrency checks and separate `managedFields` entries.
2. `metadata.generation` stops incrementing on status writes, so `generation` counts *intent* revisions only.
3. RBAC can be split: a user may `update` the CR while only your controller may `update` its `/status`.

And it makes the `observedGeneration` protocol from lesson 02 work: set `status.observedGeneration = obj.Generation` at the end of a successful reconcile, and `generation == observedGeneration` becomes a machine-checkable statement that the status describes the current spec.

The practical rule that follows: **if a field can be derived from spec plus observed cluster state, it belongs in status and must be recomputed, not remembered.** Anything you "remember" in status is state you now have to reason about across restarts.

### Idempotency, precisely

The sloppy definition — "the second call does nothing" — is wrong and leads people to write reconcilers that check `if status already set { return }`, which is a *different* bug (drift is never corrected). The correct definition:

> **A reconcile is idempotent if the resulting state is a function of the inputs alone — not of how many times it ran, nor in what order relative to other reconciles.**

A create-or-update that writes a drifted field back to its desired value every single time it runs *is* idempotent, even though it mutates the object on every call, because the result is identical. What is disallowed is a result that *accumulates*.

#### The broken reconciler

Here is a plausible-looking `WorkloadCost` reconciler with four independent idempotency bugs. Every one of them is something people actually write.

```go
// ❌ BROKEN — do not ship this. Four bugs, marked.
func (r *WorkloadCostReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := logf.FromContext(ctx)

    var wc finopsv1alpha1.WorkloadCost
    if err := r.Get(ctx, req.NamespacedName, &wc); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    var pods corev1.PodList
    if err := r.List(ctx, &pods,
        client.InNamespace(wc.Namespace),
        client.MatchingLabels(wc.Spec.Selector)); err != nil {
        return ctrl.Result{}, err
    }

    // ── BUG 1: ACCUMULATION ────────────────────────────────────────────────
    // Adds to a running total on every call. The total is now a function of
    // how many times Reconcile ran, which is unbounded and unknowable.
    elapsedHours := time.Since(wc.Status.LastSampleTime.Time).Hours()
    for _, p := range pods.Items {
        wc.Status.AccumulatedGPUHours += gpuCount(&p) * elapsedHours
    }
    wc.Status.LastSampleTime = metav1.Now()
    wc.Status.SpendUSD = wc.Status.AccumulatedGPUHours * r.rateUSDPerGPUHour

    // ── BUG 2: BLIND CREATE ────────────────────────────────────────────────
    // Second call returns AlreadyExists → non-nil error → requeue → forever.
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{Name: wc.Name + "-report", Namespace: wc.Namespace},
        Data:       map[string]string{"spendUSD": fmt.Sprintf("%.2f", wc.Status.SpendUSD)},
    }
    if err := r.Create(ctx, cm); err != nil {
        return ctrl.Result{}, err
    }

    // ── BUG 3: APPEND TO A LIST ────────────────────────────────────────────
    // conditions grows without bound; the CRD schema does not dedupe by type.
    wc.Status.Conditions = append(wc.Status.Conditions, metav1.Condition{
        Type: "Ready", Status: metav1.ConditionTrue,
        Reason: "CostComputed", Message: "ok",
        LastTransitionTime: metav1.Now(),
    })

    // ── BUG 4: UNCONDITIONAL EXTERNAL SIDE EFFECT ──────────────────────────
    // Posts a charge to a billing API on every reconcile. The billing system
    // has no idea these are retries of the same logical event.
    if err := r.billing.PostCharge(ctx, wc.Name, wc.Status.SpendUSD); err != nil {
        return ctrl.Result{}, err
    }

    if err := r.Status().Update(ctx, &wc); err != nil {
        return ctrl.Result{}, err
    }
    log.Info("reconciled", "spendUSD", wc.Status.SpendUSD)
    return ctrl.Result{}, nil
}
```

#### Exactly what breaks when it runs twice

Take one concrete scenario: **4 GPUs held for one hour at $3.00/GPU-hour**. The correct answer is $12.00. Trace what this code produces.

```
  SETUP: 4 GPUs, rate $3.00/GPU-h, status starts empty (LastSampleTime = zero)

  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ CALL 1  — t = 10:00:00, triggered by the user creating the object                │
  ├──────────────────────────────────────────────────────────────────────────────────┤
  │ elapsedHours = time.Since(zero) ≈ 490,000 h   ← BUG 1 on the very first call:    │
  │                                                  zero-valued LastSampleTime       │
  │ AccumulatedGPUHours = 4 × 490,000 = 1,960,000                                    │
  │ SpendUSD            = $5,880,000.00           ← already catastrophically wrong    │
  │ Create(ConfigMap)   → OK                                                          │
  │ conditions          = [Ready]                 (len 1)                             │
  │ PostCharge($5.88M)  → billing row #1                                              │
  │ Status().Update     → OK                                                          │
  └──────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ CALL 2  — t = 10:00:00.030, the user's second edit 30 ms later                   │
  ├──────────────────────────────────────────────────────────────────────────────────┤
  │ elapsedHours ≈ 0.0000083                                                          │
  │ AccumulatedGPUHours = 1,960,000 + 0.000033 → basically unchanged                  │
  │ Create(ConfigMap)   → ❌ AlreadyExists                                             │
  │                        return err  ⇒ NOTHING BELOW RUNS                           │
  │                        ⇒ status is NEVER updated again                            │
  │                        ⇒ the key is requeued with rate-limited backoff            │
  └──────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ CALLS 3…N — the backoff ladder, every one failing at the same line               │
  ├──────────────────────────────────────────────────────────────────────────────────┤
  │ 5ms, 10ms, 20ms, 40ms, … doubling per failure (baseDelay × 2^failures)            │
  │ after 18 failures: 5ms × 2^18 ≈ 1,310 s → clamped to maxDelay 1000 s ≈ 16.7 min   │
  │ ⇒ within ~2 minutes of wall clock this object reconciles at most once every       │
  │   16.7 minutes, forever, and NOTHING in the logs says "AlreadyExists is fatal     │
  │   for this object" — it just looks like a slow controller.                        │
  └──────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │ NOW REMOVE BUG 2 (use CreateOrUpdate) AND RE-RUN. The other three are still there.│
  ├──────────────────────────────────────────────────────────────────────────────────┤
  │ 10-hour resync period, 10 reconciles/hour from ordinary pod churn:                │
  │   • AccumulatedGPUHours grows by (4 × Δt) on EVERY call, and the Δt windows        │
  │     OVERLAP because LastSampleTime is advanced per call, not per interval.         │
  │     Real elapsed: 1 h.  Reported: 1 h × (number of reconciles) in the worst case.  │
  │     With 10 reconciles in that hour → 40 GPU-hours reported, $120.00.              │
  │     TRUE ANSWER: $12.00. Overcharge factor: 10×, and it scales with event rate —   │
  │     a busier cluster produces a BIGGER bill for the SAME compute.                  │
  │   • conditions = [Ready, Ready, Ready, …] — 10 entries and climbing. At ~200 bytes │
  │     per condition, ~7,800 reconciles fills etcd's 1.5 MiB request limit and the    │
  │     object becomes UNWRITABLE: "etcdserver: request is too large".                 │
  │   • billing rows: 10 charges for 1 hour of compute. The customer is billed 10×.    │
  └──────────────────────────────────────────────────────────────────────────────────┘

  ⇒ NONE of these produce an error, an alert, or a stack trace.
    The controller reports Ready=True the entire time.
```

That last line is the point. An idempotency bug does not crash. It reports success while producing wrong numbers, and on a cost operator the wrong number ends up on an invoice. This is why the checkpoint for this module demands a *test* that proves idempotency, not a code review that asserts it.

#### The fix

Four bugs, four distinct techniques.

```go
// ✅ CORRECT — every status field is a pure function of observed inputs.
func (r *WorkloadCostReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := logf.FromContext(ctx)

    var wc finopsv1alpha1.WorkloadCost
    if err := r.Get(ctx, req.NamespacedName, &wc); err != nil {
        // NotFound is normal: the object was deleted between enqueue and reconcile.
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // (deletion handling goes here — see the finalizer section below)

    var pods corev1.PodList
    if err := r.List(ctx, &pods,
        client.InNamespace(wc.Namespace),
        client.MatchingLabels(wc.Spec.Selector)); err != nil {
        return ctrl.Result{}, err
    }

    // ── FIX 1: DERIVE, DO NOT ACCUMULATE ───────────────────────────────────
    // Instantaneous facts are pure functions of the current pod set. Compute
    // them from scratch every time; the answer cannot drift.
    var gpus int64
    for i := range pods.Items {
        gpus += gpuCount(&pods.Items[i])
    }
    rate := wc.Spec.RateUSDPerGPUHour            // resource.Quantity → *inf.Dec, no float drift
    hourlyBurn := computeHourlyBurn(gpus, rate)

    // ── FIX 1b: ACCUMULATORS NEED A WATERMARK, NOT A DELTA ─────────────────
    // A cumulative total genuinely cannot be derived from the current pod set,
    // so make the accumulation idempotent with a monotonic watermark: advance
    // in whole, aligned intervals, and only for intervals strictly after the
    // last one already accounted for. A second call in the same interval adds
    // exactly zero, which is what makes it safe to call any number of times.
    now := r.clock.Now().UTC()
    const bucket = time.Hour
    upTo := now.Truncate(bucket)                             // last CLOSED bucket
    from := wc.Status.AccountedThrough.Time.UTC().Truncate(bucket)
    if wc.Status.AccountedThrough.IsZero() {
        from = wc.CreationTimestamp.Time.UTC().Truncate(bucket)
    }
    if upTo.After(from) {
        // Authoritative series, not "time since I last ran": ask the metrics
        // backend for the exact closed window. Two callers asking for the same
        // window get the same number.
        added, err := r.usage.GPUHours(ctx, wc.Spec.Selector, from, upTo)
        if err != nil {
            return ctrl.Result{}, err          // requeue with backoff; nothing was mutated
        }
        wc.Status.AccumulatedGPUHours += added
        wc.Status.AccountedThrough = metav1.NewTime(upTo)   // watermark advances once
    }
    wc.Status.SpendUSD = round2(wc.Status.AccumulatedGPUHours * rateFloat(rate))
    wc.Status.CurrentGPUs = gpus
    wc.Status.HourlyBurnUSD = hourlyBurn

    // ── FIX 2: CREATE-OR-UPDATE, NOT CREATE ────────────────────────────────
    // Converge the child to its desired shape whether or not it exists.
    cm := &corev1.ConfigMap{ObjectMeta: metav1.ObjectMeta{
        Name: wc.Name + "-report", Namespace: wc.Namespace,
    }}
    op, err := controllerutil.CreateOrUpdate(ctx, r.Client, cm, func() error {
        cm.Data = map[string]string{
            "spendUSD":    fmt.Sprintf("%.2f", wc.Status.SpendUSD),
            "currentGPUs": strconv.FormatInt(gpus, 10),
        }
        // Owner reference ⇒ GC deletes the ConfigMap when the parent goes,
        // and Owns() maps ConfigMap events back to this WorkloadCost key.
        return controllerutil.SetControllerReference(&wc, cm, r.Scheme)
    })
    if err != nil {
        return ctrl.Result{}, err
    }
    if op != controllerutil.OperationResultNone {
        log.Info("report converged", "op", op)   // Created | Updated
    }

    // ── FIX 3: SET BY KEY, NOT APPEND ──────────────────────────────────────
    // meta.SetStatusCondition upserts by .type and only moves
    // lastTransitionTime when .status actually flips — no spurious writes,
    // no unbounded growth, no write-triggers-watch-triggers-write loop.
    meta.SetStatusCondition(&wc.Status.Conditions, metav1.Condition{
        Type:               "Ready",
        Status:             metav1.ConditionTrue,
        ObservedGeneration: wc.Generation,
        Reason:             "CostComputed",
        Message: fmt.Sprintf("%d GPUs across %d pods; $%.2f/h",
            gpus, len(pods.Items), hourlyBurn),
    })

    // ── FIX 4: MAKE THE EXTERNAL CALL IDEMPOTENT AT THE BOUNDARY ───────────
    // You cannot make someone else's API idempotent, but you can give it a
    // deterministic key so IT can dedupe. The key must be a function of the
    // billable fact, not of this process, this pod, or this attempt.
    idem := fmt.Sprintf("%s/%s@%s", wc.UID, upTo.Format(time.RFC3339), "v1")
    if err := r.billing.PostCharge(ctx, billing.Charge{
        IdempotencyKey: idem,             // server-side dedup: replay ⇒ same row
        Subject:        string(wc.UID),   // UID, not name: survives delete+recreate
        AmountUSD:      wc.Status.SpendUSD,
    }); err != nil {
        return ctrl.Result{}, err
    }

    // ── observedGeneration LAST, and only on the success path ──────────────
    wc.Status.ObservedGeneration = wc.Generation
    if err := r.Status().Update(ctx, &wc); err != nil {
        if apierrors.IsConflict(err) {
            return ctrl.Result{}, err     // requeue; re-read and recompute. Not an error to log.
        }
        return ctrl.Result{}, err
    }
    return ctrl.Result{}, nil
}
```

#### The taxonomy, so you can classify the next case yourself

| Shape of the operation | Idempotent? | How to make it safe |
|---|---|---|
| `x = f(observed inputs)` | ✅ yes | nothing to do — this is the target shape |
| `Create(child)` | ❌ `AlreadyExists` on retry | `CreateOrUpdate` / `CreateOrPatch`, or SSA `client.Apply` with a `FieldOwner` |
| `x += delta` | ❌ accumulates | derive from an authoritative series over a **closed, aligned window**, guarded by a monotonic watermark in status |
| `list = append(list, item)` | ❌ grows unbounded | upsert by key (`meta.SetStatusCondition`, or a map keyed by identity, or a `listType: map` field) |
| `Delete(child)` | ✅ effectively | ignore `NotFound` (`client.IgnoreNotFound`) |
| `POST /charges` to an external API | ❌ duplicates | deterministic idempotency key derived from the billable fact; dedupe server-side |
| `Update(obj)` read-modify-write | ⚠️ conflict-prone | return the `409` and let the queue replay you, or `retry.RetryOnConflict` with the `Get` **inside** the closure |
| Emitting a Kubernetes Event | ⚠️ noisy but bounded | the event recorder aggregates duplicates; still, only emit on state *transitions* |
| Rolling a random value (password, suffix) | ❌ different every call | generate once and persist it in status/a Secret; read it back on subsequent calls |
| `time.Now()` in a computed field | ❌ changes every call | either do not store it, or store it only on a transition (`lastTransitionTime`) |

That last row deserves emphasis because it produces the most confusing bug in the set: any field that changes on every reconcile makes your `Status().Update()` a real write, which produces a watch event, which enqueues your own key, which reconciles again. **A non-idempotent timestamp is an infinite loop with a CPU cost.** Symptom: `controller_runtime_reconcile_total` climbing at hundreds per second for a static cluster, and `resourceVersion` on one object incrementing forever.

### Tolerate stale reads

Your `Get` and `List` read from the informer **cache**, which lags the apiserver by a watch round-trip plus queue time — normally single-digit milliseconds, occasionally much more under load. So:

- An object you just created may not be in the cache on the very next reconcile. `CreateOrUpdate` may therefore try to `Create` again and get `AlreadyExists` — which is why you return that error and let the requeue sort it out rather than panicking.
- A delete you just issued may still be visible.
- `observedGeneration` may trail what you just wrote.
- Two consecutive reconciles may see the *same* stale state, so "it didn't change, therefore my write failed" is an invalid inference.

Correct level-triggered code is naturally robust to this: if a read was stale, the watch will eventually deliver the newer state and the next reconcile converges. The failure mode is code that treats a stale read as authoritative and takes a destructive action — e.g. "the child ConfigMap is not in my cache, therefore it was deleted, therefore recreate it" — which in a race recreates a resource that already exists, or worse, deletes something because it "should not exist."

When you genuinely need a live read (a leader-election-adjacent decision, or a pre-delete confirmation), controller-runtime lets you bypass the cache:

```go
// Manager-level: never cache this type at all.
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Client: client.Options{
        Cache: &client.CacheOptions{
            DisableFor: []client.Object{&corev1.Secret{}},   // read Secrets live
        },
    },
})

// Or build a one-off uncached client for a specific check.
live, err := client.New(mgr.GetConfig(), client.Options{Scheme: mgr.GetScheme()})
```

Use it sparingly. Every uncached read is a real apiserver request charged against APF; a controller that reads live on the hot path is exactly the controller that earns 429s and, at scale, the kind of apiserver pressure OpenAI documented at 7,500 nodes.

### The requeue paths, backoff, and where `Forget` belongs

After your `Reconcile` returns, controller-runtime steers the rate-limiting workqueue. Here are the *exact* branches from `pkg/internal/controller/controller.go` on `main`, in order:

```
                    Reconcile(ctx, req) returns (result, err)
                                     │
        ┌────────────────────────────┼────────────────────────────────┬──────────────┐
        ▼                            ▼                                ▼              ▼
  err != nil AND              err != nil AND              result.RequeueAfter > 0   default
  reconcile.TerminalError     ordinary error                 (err == nil)        (err == nil,
        │                            │                                │           no requeue)
        ▼                            ▼                                ▼              ▼
  TerminalReconcileErrors++    Queue.AddWithOpts(              Queue.Forget(req)  Queue.Forget(req)
  ── NOT REQUEUED ──            {RateLimited: true}, req)      Queue.AddWithOpts(  ── DONE ──
  metric:                       ── BACKOFF CLIMBS ──            {After: d}, req)   backoff reset
  controller_runtime_           backoff NOT reset               ── FIXED DELAY ──   to zero;
  terminal_reconcile_                                           backoff RESET       only a watch
  errors_total                                                                      re-enqueues
        │                            │                                │              │
        └────────────────────────────┴────────────────┬───────────────┴──────────────┘
                                                      ▼
                                          defer Queue.Done(req)
                              (releases the key; if it was re-Added while
                               you held it, Done() pushes it back now)

  BACKOFF ARITHMETIC — client-go DefaultTypedControllerRateLimiter, per key:
    MaxOf(
      ItemExponentialFailureRateLimiter(baseDelay = 5ms, maxDelay = 1000s),
      BucketRateLimiter(rate.NewLimiter(10 qps, burst 100))     ← global, all keys
    )
    delay(n) = min(5ms × 2^n, 1000s)        n = consecutive failures for THIS key

      n :   0     1     2     3     4      5      6      8      10      15      18+
  delay :  5ms  10ms  20ms  40ms  80ms  160ms  320ms  1.28s   5.1s   164s   1000s (cap)

    cumulative wall time to reach the cap ≈ 2× the cap ≈ 33 min of failing reconciles.
    Forget(key) deletes the key from the failures map ⇒ n returns to 0.
```

Three things to extract from that.

**1. `RequeueAfter` and error backoff are different mechanisms, and `RequeueAfter` now resets backoff.** This is a correction to widely-circulated older guidance. controller-runtime issue [#392](https://github.com/kubernetes-sigs/controller-runtime/issues/392), *"Reconcile loops using RequeueAfter never reset their failure count"* (opened 11 April 2019), reported that a controller which stabilises by returning `Result{RequeueAfter: d}, nil` never called `Forget`, so the per-key failure count accumulated over the object's lifetime and any subsequent error immediately hit the maximum backoff — roughly 16 minutes with the 1000-second cap. **That has been fixed.** The current code explicitly calls `c.Queue.Forget(req)` in the `RequeueAfter` branch before re-adding with a delay. If you are reading a blog post that tells you to avoid mixing `RequeueAfter` with error returns for this reason, the reason no longer applies — though the *advice* is still good, because mixing them muddies intent.

**2. `Result.Requeue` (the bare bool) is deprecated.** The source comment is explicit:

> *This setting is deprecated as it causes confusion and there is no good reason to use it. When waiting for an external event to happen, either the duration until it is supposed to happen or an appropriate poll interval should be used, rather than an interval emitted by a ratelimiter whose purpose it is to control retry on error.* **Deprecated: Use `RequeueAfter` instead.**

Note also from the branch table: `result.Requeue` does *not* call `Forget`, so it is genuinely a third, worse behaviour. Return an error to get backoff; set `RequeueAfter` to poll.

**3. `reconcile.TerminalError` exists for errors that retrying cannot fix.** Wrapping an error with `reconcile.TerminalError(err)` tells the controller "this will not be retried but still be logged and recorded in metrics" — it increments `controller_runtime_terminal_reconcile_errors_total` and does not re-enqueue. Use it for genuinely permanent conditions: a spec that fails a semantic check your CRD schema cannot express, a referenced object of a kind you cannot handle, an external API returning a hard 4xx. Do **not** use it for anything transient; the object then sits un-reconciled until an unrelated event pokes it.

The three paths in the order you will actually reach for them:

| Return | Effect | Use when |
|---|---|---|
| `ctrl.Result{}, nil` | `Forget`; only a watch re-enqueues | steady state — most reconciles end here |
| `ctrl.Result{}, err` | rate-limited requeue, backoff climbs | transient failure: conflict, API error, dependency not ready |
| `ctrl.Result{RequeueAfter: d}, nil` | `Forget`, then requeue after exactly `d` | polling something with no watch (a cloud billing API, a TTL expiry) |
| `ctrl.Result{}, reconcile.TerminalError(err)` | logged + metric, **no** requeue | permanently invalid input |

#### `Forget` in raw client-go

controller-runtime hides the queue, but the interview question is usually asked about raw client-go, where you manage it yourself. The canonical shape, from `kubernetes/sample-controller`'s `processNextWorkItem`/`handleErr`:

```go
func (c *Controller) processNextWorkItem(ctx context.Context) bool {
    key, shutdown := c.workqueue.Get()
    if shutdown {
        return false
    }
    // Done MUST be called for every Get, or the key is never re-processable.
    defer c.workqueue.Done(key)

    err := c.syncHandler(ctx, key)
    switch {
    case err == nil:
        c.workqueue.Forget(key)                     // ✅ success: reset this key's backoff
    case c.workqueue.NumRequeues(key) < maxRetries: // maxRetries typically 5–15
        c.workqueue.AddRateLimited(key)             // ✅ retry: backoff KEEPS climbing
    default:
        c.workqueue.Forget(key)                     // ✅ give up: reset so a future
        utilruntime.HandleError(err)                //    genuine change starts fresh
        c.recorder.Eventf(obj, corev1.EventTypeWarning,
            "SyncFailed", "dropping out of the queue after %d retries: %v",
            maxRetries, err)
    }
    return true
}
```

**The rule: `Forget` on success or on permanent give-up — never on a retry.** Forgetting mid-retry erases the backoff you are deliberately accumulating, which turns a failing dependency into a hot loop against the apiserver. And `Done` is unconditional and deferred: skipping it leaks the key in the `processing` set forever, and since `Add` on a processing key only marks it dirty, that key is never processed again. "One object stopped reconciling and nothing else did" is very often exactly this.

The workqueue metrics that make all of this observable: `workqueue_depth`, `workqueue_adds_total`, `workqueue_retries_total`, `workqueue_queue_duration_seconds` (time waiting), `workqueue_work_duration_seconds` (time in your handler), `workqueue_unfinished_work_seconds` and `workqueue_longest_running_processor_seconds` (a wedged worker shows up here and nowhere else). Plus controller-runtime's own `controller_runtime_reconcile_total`, `controller_runtime_reconcile_errors_total`, `controller_runtime_terminal_reconcile_errors_total`, `controller_runtime_reconcile_time_seconds`, and `controller_runtime_active_workers` against `controller_runtime_max_concurrent_reconciles`.

### The deletion lifecycle and finalizers

Reconcile is not just "converge toward spec." Half the lifecycle is teardown, and it is where controllers most often leave production wedged.

**The problem.** Kubernetes deletion is, by default, immediate: the object is gone from etcd and your controller may never see the deletion at all (the informer may deliver a `DELETED` event, but if your process was down it will just see the object missing on relist). That is fine when everything you created has an owner reference and the garbage collector handles it. It is not fine when your controller owns state *outside* the cluster: a cloud resource, a row in a billing ledger, a lease on a license server, a final spend record. You need a hook that runs *before* the object disappears, and it has to survive your controller being down when the delete arrives.

**The mechanism.** `metadata.finalizers` is a list of opaque string keys. The API server refuses to remove an object from storage while that list is non-empty. Instead:

```
   USER: kubectl delete workloadcost gpu-train-77
        │
        ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │ APISERVER (registry/generic/registry/store.go)                                │
  │   pendingFinalizers = len(obj.metadata.finalizers) != 0                       │
  │                                                                               │
  │   if pendingFinalizers:                                                       │
  │       markAsDeleting():  metadata.deletionTimestamp = now                     │
  │                          metadata.deletionGracePeriodSeconds = 0              │
  │                          metadata.generation++          ← intent changed       │
  │       ⇒ UPDATE the object; DO NOT delete it                                   │
  │       ⇒ respond 202 Accepted                                                  │
  │   else:                                                                       │
  │       ⇒ actually delete from etcd; respond 200                                │
  └───────────────────────────────┬───────────────────────────────────────────────┘
                                  │  a MODIFIED watch event (not DELETED!)
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │  OBJECT STATE: "Terminating"                                                   │
  │    • still fully readable via kubectl and your cache                           │
  │    • still reconciled — your Reconcile keeps being called                      │
  │    • spec is effectively frozen (no new finalizers may be ADDED)               │
  │    • existing finalizer entries may only be REMOVED                            │
  │    metadata:                                                                   │
  │      deletionTimestamp: "2026-08-17T09:41:03Z"                                 │
  │      finalizers:                                                               │
  │      - finops.example.com/record-final-spend      ← yours                      │
  │      - foregroundDeletion                          ← the GC's, if propagation  │
  │                                                       policy was Foreground    │
  └───────────────────────────────┬───────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────┴───────────────────────────┐
        ▼                                                     ▼
  YOUR CONTROLLER                                    GARBAGE COLLECTOR
  sees deletionTimestamp != nil                      (only if foregroundDeletion)
        │                                                     │
        ├─ run cleanup: post final spend record               ├─ delete dependents whose
        │  (idempotent! it may be called many times)          │  ownerRef has
        │                                                     │  blockOwnerDeletion: true
        ├─ cleanup succeeded?                                 │
        │     no  → return err → requeue with backoff         └─ all gone → remove
        │             (finalizer STAYS; object stays)            foregroundDeletion
        │     yes → controllerutil.RemoveFinalizer(&wc, key)
        │           r.Update(ctx, &wc)
        ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │ APISERVER: the update leaves metadata.finalizers EMPTY                        │
  │   ⇒ the object is now removed from etcd for real                              │
  │   ⇒ a DELETED watch event fires                                               │
  │   ⇒ background GC (the default policy) collects owned children                │
  └───────────────────────────────────────────────────────────────────────────────┘

  ── STUCK PATH ──────────────────────────────────────────────────────────────────
  cleanup NEVER succeeds, or the controller is gone entirely:
      finalizer is never removed → object stays in Terminating FOREVER
      • its namespace cannot finish deleting (the namespace controller waits on it)
      • its name cannot be reused
      • kubectl delete --grace-period=0 --force does NOT help:
        that shortens the GRACE PERIOD, it does not bypass finalizers
```

The two API-conventions rules that make this workable, and that people violate: **finalizers may only be removed once `deletionTimestamp` is set** (you cannot add one to an object already being deleted), and **order is explicitly not enforced** — the `ObjectMeta.Finalizers` doc comment says so directly, because ordering "introduces significant risk of stuck finalizers." Never write a finalizer that assumes another one ran first.

#### The reconcile skeleton, with the ordering that matters

```go
const finalizerKey = "finops.example.com/record-final-spend"

func (r *WorkloadCostReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var wc finopsv1alpha1.WorkloadCost
    if err := r.Get(ctx, req.NamespacedName, &wc); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // ── DELETING? ──────────────────────────────────────────────────────────
    if !wc.DeletionTimestamp.IsZero() {
        if !controllerutil.ContainsFinalizer(&wc, finalizerKey) {
            return ctrl.Result{}, nil        // someone else's finalizer; not our turn
        }

        // Cleanup MUST be idempotent: it will be retried, possibly many times,
        // and may have partially completed before a crash.
        if err := r.recordFinalSpend(ctx, &wc); err != nil {
            // Finalizer stays. Object stays. We get requeued with backoff.
            // THIS is the branch that becomes a stuck object if it never succeeds.
            return ctrl.Result{}, err
        }

        controllerutil.RemoveFinalizer(&wc, finalizerKey)
        // Note: Update, not Status().Update — finalizers live in metadata.
        if err := r.Update(ctx, &wc); err != nil {
            return ctrl.Result{}, client.IgnoreNotFound(err)
        }
        return ctrl.Result{}, nil            // object will now actually disappear
    }

    // ── NOT DELETING: register the finalizer BEFORE creating external state ─
    // Ordering is the whole game. If you create the external resource first and
    // crash before adding the finalizer, a delete arriving next removes the
    // object immediately and the external resource is orphaned with no record
    // of who owned it. Adding the finalizer first can at worst leave a
    // finalizer on an object with nothing to clean up — cheap and recoverable.
    if !controllerutil.ContainsFinalizer(&wc, finalizerKey) {
        controllerutil.AddFinalizer(&wc, finalizerKey)
        if err := r.Update(ctx, &wc); err != nil {
            return ctrl.Result{}, err
        }
        // The Update bumped resourceVersion; return and let the resulting watch
        // event re-enqueue us with a fresh object rather than reusing a stale one.
        return ctrl.Result{}, nil
    }

    // ── normal convergence (the idempotent body from earlier) ───────────────
    return r.converge(ctx, &wc)
}
```

Two subtleties worth internalising.

**`recordFinalSpend` must itself be idempotent.** It will be called on every reconcile of a terminating object, which — if anything else in the loop errors — can be hundreds of times. Give it the same treatment as any other external side effect: a deterministic idempotency key derived from `wc.UID` (not name — a delete-and-recreate reuses the name but not the UID) and the closing time window.

**Do not put the finalizer removal behind a "have I already cleaned up?" flag in status.** You cannot write status and remove the finalizer atomically, so a crash between the two puts you in a state where the flag says done and the finalizer says not-done — which is fine, because your idempotent cleanup will just run again. The flag buys nothing and adds a failure mode.

#### Cascade propagation, briefly

Deleting an owner offers three policies, selected by the client:

| Policy | `kubectl` | What happens | Finalizer used |
|---|---|---|---|
| **Background** (default) | `--cascade=background` | owner deleted immediately; GC deletes dependents asynchronously | none |
| **Foreground** | `--cascade=foreground` | `foregroundDeletion` added to the owner; owner stays visible in `Terminating` until dependents with `blockOwnerDeletion: true` are gone | `foregroundDeletion` |
| **Orphan** | `--cascade=orphan` | dependents survive; their owner reference to this owner is stripped | `orphan` |

For your operator: prefer background (the default) and rely on owner references, because foreground makes your object's deletion latency a function of every dependent's deletion latency — including any dependent with its *own* stuck finalizer. Foreground is for the rare case where something outside must not observe the owner as gone until the children really are.

#### The stuck finalizer: diagnosis, and why force-removal is dangerous

**Symptom.** An object sits in `Terminating` indefinitely. `kubectl delete` appears to hang or returns and changes nothing. Frequently the first thing anyone notices is a *namespace* stuck in `Terminating`, because the namespace controller cannot finish until every object in it is really gone.

**Diagnosis, in order.**

```
# 1. WHICH finalizer is holding it?
$ kubectl get workloadcost gpu-train-77 -n finops -o jsonpath='{.metadata.finalizers}{"\n"}'
["finops.example.com/record-final-spend"]

$ kubectl get workloadcost gpu-train-77 -n finops \
    -o jsonpath='{.metadata.deletionTimestamp}{"\n"}'
2026-08-17T09:41:03Z            # 6 hours ago — this is not a slow cleanup

# 2. Is the controller that owns that finalizer even running?
$ kubectl -n finops-system get deploy gpu-cost-operator-controller-manager
NAME                                   READY   UP-TO-DATE   AVAILABLE
gpu-cost-operator-controller-manager   0/1     1            0        # ← there it is

# 3. If it IS running: does it hold the leader lease, and is it erroring?
$ kubectl -n finops-system get lease gpu-cost-operator \
    -o jsonpath='{.spec.holderIdentity}{"  renewed="}{.spec.renewTime}{"\n"}'
gpu-cost-operator-7d9c-xk2p_5f3a…  renewed=2026-08-17T15:44:02Z

$ kubectl -n finops-system logs deploy/gpu-cost-operator-controller-manager --tail=20 \
  | grep gpu-train-77
ERROR  Reconciler error  {"controller":"workloadcost","name":"gpu-train-77",
       "error":"record final spend: Post \"https://billing.internal/v1/charges\":
        dial tcp 10.4.2.9:443: i/o timeout"}
# → the finalizer is doing its job. The dependency is down. Fix the dependency.

# 4. For a stuck NAMESPACE, read the conditions — they name the blocker exactly:
$ kubectl get namespace finops -o jsonpath='{range .status.conditions[*]}{.type}={.status}: {.message}{"\n"}{end}'
NamespaceDeletionDiscoveryFailure=False: All resources successfully discovered
NamespaceDeletionContentFailure=False:   All content successfully deleted
NamespaceContentRemaining=True:          Some resources are remaining: workloadcosts.finops.example.com has 1 resource instances
NamespaceFinalizersRemaining=True:       Some content in the namespace has finalizers remaining: record-final-spend in 1 resource instances
```

Those four namespace conditions are the fastest diagnosis in Kubernetes and almost nobody reads them. `NamespaceDeletionDiscoveryFailure=True` means an APIService or CRD is broken and the namespace controller cannot even enumerate what to delete — classically an aggregated API (`v1beta1.metrics.k8s.io`) whose backing pod is gone, which blocks *every* namespace deletion in the cluster. `NamespaceContentRemaining` without `NamespaceFinalizersRemaining` means the content has not been touched yet, i.e. the problem is upstream of any finalizer.

**The force-removal, and why it is a last resort.**

```
# The command everyone finds on Stack Overflow:
$ kubectl patch workloadcost gpu-train-77 -n finops \
    --type merge -p '{"metadata":{"finalizers":null}}'
workloadcost.finops.example.com/gpu-train-77 patched
# object vanishes immediately.
```

What you just did: you told the apiserver "nobody needs to clean up after this," and the apiserver believed you. The consequences, concretely:

1. **The cleanup never runs.** For this operator, the final spend record is never posted — a GPU workload's last billing period silently disappears. For a cloud-provisioning operator, the load balancer, disk, or IP address it created is now orphaned and billing forever with nothing in the cluster referencing it. This is the most expensive class of "I fixed it" in Kubernetes operations.
2. **You destroyed the evidence.** The object carried the only in-cluster record of what needed cleaning. After the patch there is no way to enumerate what was leaked.
3. **If it was a `foregroundDeletion` finalizer you removed,** the owner disappears while its dependents remain, and those dependents now have an owner reference to a UID that no longer exists. The GC will collect them — but if the UID were ever reused (it is not, but this is why UIDs exist), it would collect the wrong ones.
4. **For a Namespace, it is worse.** Force-clearing the namespace's `kubernetes` finalizer (through the `/finalize` subresource) removes the Namespace object while the objects *inside* it remain in etcd. They are now unreachable through normal listing, because listing is scoped by a namespace that no longer exists — invisible, undeleteable by normal means, consuming quota accounting and etcd space. Recreating a namespace with the same name resurrects them. Multiple vendor knowledge bases document this as the reason to diagnose rather than force.

The correct order of operations is always: **find which finalizer, find who owns it, find why its controller cannot make progress, fix that.** Force-removal is acceptable only when you have independently confirmed the cleanup is unnecessary — e.g. the external resource is already gone, or the controller has been permanently decommissioned and you have manually done what it would have done. And when you do it, record what you bypassed.

**Designing finalizers so this does not happen to you:**

- **Bound the work.** Put a timeout on every external call inside the cleanup path. An unbounded `http.Get` inside a finalizer is a stuck object waiting for a network partition.
- **Fail open on data you cannot get.** If the final spend cannot be computed because the metrics backend is gone, decide in advance whether "record zero and remove the finalizer" is better than "block deletion forever." For most operators it is. Encode that decision, do not leave it to whoever is on call at 3 a.m.
- **Emit an Event on every failed cleanup attempt** so `kubectl describe` explains the block without needing controller logs.
- **Consider a deadline.** After N minutes of failing cleanup, log loudly, emit a Warning event, and remove the finalizer anyway — a deliberate, observable fail-open beats an undocumented indefinite block. This is a policy decision; make it consciously.
- **Never add a finalizer you are not certain you will always be running to remove.** Uninstalling an operator whose CRs still carry its finalizer is how a "simple `helm uninstall`" becomes an afternoon.

## Perspectives

**Theory perspective.** Level- versus edge-triggered is a general distributed-systems idea, not a Kubernetes invention — the same distinction as level- versus edge-triggered interrupts, or `epoll`'s two readiness modes. Recognising it as transferable means the discipline applies the next time you build anything that consumes an unreliable notification stream: a message-queue consumer, a file watcher, a webhook receiver, a CDC pipeline. The general principle is: *if a consumer can miss, coalesce, or reorder notifications, make the notification a hint and the state the truth.*

**Practice perspective.** The queue's coalesce/drop/reorder/replay behaviour is what *forces* the idempotency discipline; it is not optional style, it is a correctness requirement given what the machinery is permitted to do. You do not get to opt out of handling a coalesced double-edit — the machinery guarantees you will see one eventually, so your reconcile has to be correct under it by construction rather than by testing.

**Failure-mode perspective.** Nearly every "my controller stopped reconciling" incident traces to one of six things, and all six are mechanical consequences of the material above: a swallowed error (returned `nil` after logging, so the queue forgot the key), a missing watch (a dependency changed and nothing enqueued the primary), an RBAC gap so the cache never synced and `WaitForCacheSync` blocked forever, a wedged worker that never called `Done`, backoff climbing to the 1000-second cap unnoticed, or a lost leader lease. The diagnosis order in the checkpoint is exactly that list.

**Economics perspective (module-specific).** An idempotency bug in a cost operator is a wrong invoice, not a wrong dashboard. The worked trace above showed a 10× overcharge from a single accumulator bug — and, worse, an overcharge that *scales with cluster event rate*, so a busier cluster is billed more for the same compute. Across 40 clusters a systematic bug compounds silently until someone reconciles the bill by hand. That is why the acceptance criterion for this lesson's practice is a green idempotency test, not a code review.

## Real-world use cases

- **Tim Hockin, "Edge vs. Level triggered logic" (June 2017).** The design rationale from a founding Kubernetes engineer, framed through the hardware-interrupt analogy: level-driven software is about *asserting state* so a client can check and re-check at any time, and that is the heart of the controller model. The load-bearing argument is that edge-triggered designs are brittle precisely where distributed systems are unreliable — dropped connections, restarts, slow consumers — because a lost edge is lost work with no recovery path, whereas a level-triggered consumer that misses every notification but one still converges. This is as close to a primary source as exists for *why* Kubernetes controllers are shaped this way.
- **controller-runtime issue [#392](https://github.com/kubernetes-sigs/controller-runtime/issues/392), "Reconcile loops using RequeueAfter never reset their failure count" (April 2019).** A real framework bug with a real production symptom: controllers that stabilise on `RequeueAfter` accumulated a per-key failure count that was never cleared, so the exponential limiter drove any later error straight to the ~16-minute cap and the controller merely *appeared* slow. Worth reading both because it is now fixed (the current code calls `Forget` on that branch) and because it is a case study in how invisible a backoff bug is — nothing errors, nothing alerts, the loop just gets slower.
- **`kubernetes/sample-controller`.** The reference implementation of the raw-client-go loop, and the source of the `Forget`/`AddRateLimited`/`Done` idiom quoted above. Read `processNextWorkItem` and `handleErr` end to end once; every controller framework is a wrapper around that twenty lines, and the interview question is usually about them specifically.
- **Namespaces stuck in `Terminating`.** The most common production manifestation of finalizer mechanics, documented in vendor knowledge bases across Red Hat, Google Cloud, and Platform9. The recurring root cause is not a controller bug at all but a broken aggregated APIService: the namespace controller must discover every resource type before it can delete namespace content, so one unavailable extension API server (`v1beta1.metrics.k8s.io` is the classic) sets `NamespaceDeletionDiscoveryFailure=True` and stalls deletion of *every* namespace in the cluster. The recurring wrong fix is force-clearing the finalizer, which deletes the Namespace object and strands its contents in etcd.
- **Uber Engineering, "Migrating Uber's Compute Platform to Kubernetes."** Describes a production job controller reading a CRD object, determining its current state, and reconciling by performing an action — the Get → observe → compute → write shape of this lesson, at a scale where "just retry" is not a strategy you can hand-wave.

## Worked example

Trace one `WorkloadCost` through six reconciles that exercise every mechanism in this lesson.

*Setup.* Spec selects Pods by label and carries `rateUSDPerGPUHour: "3.00"`. Status reports `currentGPUs`, `hourlyBurnUSD`, `accumulatedGPUHours`, `spendUSD`, `accountedThrough`, `observedGeneration`, and a `Ready` condition. Four GPU pods match.

**1 — Create.** The user applies the object. `generation: 1`. The watch enqueues the key. Reconcile: `Get` the object; `deletionTimestamp` is nil; the finalizer is absent, so `AddFinalizer` + `Update` and return `{}, nil`. That `Update` bumps `resourceVersion` and fires a watch event, re-enqueueing the key.

**2 — First real convergence.** `Get` (finalizer now present) → `List` matching pods from the cache → `gpus = 4` → `hourlyBurnUSD = 12.00`. `accountedThrough` is zero, so the watermark starts at `creationTimestamp` truncated to the hour; `upTo` is the last closed hour, which is not after `from` yet, so **zero GPU-hours are accumulated** — correct, because no full hour has closed. `CreateOrUpdate` the report ConfigMap → `OperationResultCreated`. `SetStatusCondition(Ready=True, observedGeneration=1)`. `observedGeneration = 1`. `Status().Update`. Return `{}, nil` → `Forget`, backoff zero.

**3 — Two rapid edits.** The user tweaks the selector and reverts it within 50 ms. `generation` goes 1 → 2 → 3. The workqueue **coalesces**: the key is already dirty, so the second `Add` is a no-op, and one reconcile runs against the *final* state. It recomputes `gpus = 4` from the current pod list, gets the same `12.00`, and writes `observedGeneration = 3`. `SetStatusCondition` finds an identical `Ready` condition and does **not** move `lastTransitionTime`, so the status is byte-identical apart from `observedGeneration` — one small write, no loop. *This is the idempotency payoff:* the code never asked what changed, so a coalesced double-edit is invisible.

**4 — The billing API times out.** An hour boundary has passed, so the watermark branch fires and `r.usage.GPUHours(ctx, sel, 10:00, 11:00)` is called; it returns `4.0`. `accumulatedGPUHours` becomes `4.0`, `spendUSD` `12.00`, `accountedThrough` `11:00`. Then `PostCharge` fails with `i/o timeout`. Reconcile returns `ctrl.Result{}, err`. **Nothing was persisted** — the mutations were on the in-memory copy and `Status().Update` never ran. The key is requeued rate-limited: 5 ms, 10 ms, 20 ms. On the third attempt the billing API recovers; the watermark logic recomputes the *same* closed window `[10:00, 11:00)` and gets the same `4.0` — not a fourth of an hour more, because the window is aligned and closed, not "time since I last ran." `PostCharge` is sent with the same idempotency key `<uid>/2026-08-17T11:00:00Z/v1`, so even if attempt 1 actually reached the server before the timeout, the server dedupes. `Status().Update` succeeds; return `{}, nil` → `Forget` → backoff resets to zero.

Had the code instead logged the error and returned `{}, nil`, the queue would have forgotten the key, the stale status would persist, and nothing would retry until an unrelated pod event happened to poke the object. That is the "swallowed error" pitfall, and it is invisible in metrics because `controller_runtime_reconcile_errors_total` never increments.

**5 — The controller process is killed** between reconciles (eviction during a node drain). On restart: the manager builds a new cache, the reflector LISTs every `WorkloadCost` and every Pod fresh from the apiserver, the informer fires synthetic `ADDED` handlers for all of them, and the workqueue fills with every key. Reconcile runs with the *exact same code path* — no `if recovering { … }` branch anywhere. It re-lists pods, recomputes `gpus = 4`, finds `accountedThrough = 11:00` already at the last closed hour and adds zero, and writes an identical status. **Crash recovery cost: zero lines of code.**

**6 — Delete.** `kubectl delete workloadcost gpu-train-77`. The apiserver sees a non-empty `finalizers`, so it sets `deletionTimestamp`, bumps `generation` to 4, and responds `202 Accepted` — the object is still there. A `MODIFIED` event enqueues the key. Reconcile takes the deletion branch: `ContainsFinalizer` is true, so `recordFinalSpend` runs, posting a final charge for the partial hour with idempotency key `<uid>/final/v1`. It succeeds. `RemoveFinalizer` + `Update` leaves `finalizers` empty; the apiserver deletes the object for real; a `DELETED` event fires; background GC collects the report ConfigMap via its owner reference. Total elapsed: one watch round trip plus one billing call.

Had `recordFinalSpend` failed, the finalizer would stay, the object would remain in `Terminating`, reconciles would retry on the backoff ladder, and — importantly — `kubectl delete namespace finops` would block behind it, with the namespace's `NamespaceFinalizersRemaining` condition naming `record-final-spend` explicitly.

The shape is identical every time: **Get → check deletionTimestamp → observe → compute from inputs → converge children → set status → return.** No branch on what changed, no accumulated mutation, no in-memory state. Run it a thousand times on one cluster state, or kill the process mid-loop and restart, and the object lands identically.

## Practice

Take module 01's reconciler into `../practice/gpu-cost-operator/` and make it *provably* level-triggered, idempotent, and safe to delete — then prove all three with tests.

1. **Introduce the bugs, then fix them (one commit each way).** Write the reconciler with at least two of the four bugs from the broken listing above — an accumulator and a blind `Create` are the highest-value pair. Do not just reason that they are wrong; write a failing test that demonstrates it, so the before/after is visible in `git log`.

2. **Refactor to converge.** Replace every accumulated or appended field with a value derived from current inputs:
   - status scalars computed from the listed pods;
   - the child ConfigMap via `controllerutil.CreateOrUpdate` (or `client.Apply` with `client.FieldOwner("gpu-cost-operator")`) plus `SetControllerReference`;
   - conditions via `meta.SetStatusCondition`, never `append`;
   - `status.observedGeneration = obj.Generation` on the success path only.

3. **Table-driven idempotency test.** Use `envtest`, or `sigs.k8s.io/controller-runtime/pkg/client/fake` for speed. For each cluster state — no pods, one GPU pod, several, selector-miss, and a **pre-seeded stale/garbage status** — call `Reconcile` on the same key **twice back to back** and assert:
   - the resulting `.status` is deep-equal after call 1 and call 2;
   - the pre-seeded stale status is overwritten to the same converged value (this is what catches an `if already set { return }` shortcut);
   - `status.conditions` has exactly one entry per condition type after N calls, for N = 1, 2, 10;
   - `lastTransitionTime` is unchanged between call 1 and call 2;
   - `observedGeneration == generation`.

   ```go
   for i := 0; i < 2; i++ {
       _, err := r.Reconcile(ctx, ctrl.Request{NamespacedName: key})
       g.Expect(err).NotTo(HaveOccurred())
   }
   var after finopsv1alpha1.WorkloadCost
   g.Expect(k8sClient.Get(ctx, key, &after)).To(Succeed())
   g.Expect(after.Status).To(Equal(expected))   // byte-identical, not "close enough"
   ```

4. **Order-independence (level-triggering) test.** Build the same final cluster state by two different event orders — (create WorkloadCost, then pods) and (create pods, then WorkloadCost) — and assert the converged status is identical. Then reconcile the same key three extra times with nothing changed and assert the status still matches. This is the test that proves you are not branching on an edge.

5. **Backoff and error-path test.** Inject a dependency that fails once then succeeds. Assert the first `Reconcile` returns a non-nil error (so the queue backs off) and that **nothing was persisted** on that call — that is the property that makes retry safe. Assert the second returns `nil` with converged status. Add a case where the dependency returns a permanent 400 and assert you return `reconcile.TerminalError(err)`.

6. **Finalizer lifecycle test.** Under `envtest`:
   - create the object, reconcile, assert the finalizer is present;
   - `Delete` it, assert the object is **still gettable** and `deletionTimestamp` is set;
   - reconcile with the cleanup dependency **failing**, assert the object still exists and the finalizer is still there;
   - fix the dependency, reconcile, assert the object is gone (`apierrors.IsNotFound` on `Get`);
   - assert the cleanup ran **exactly once** as observed by the fake billing client's idempotency-key set, even though `Reconcile` was called on the terminating object three times.

7. **Owner-reference / cascade test.** Assert the report ConfigMap has an owner reference with `controller: true` pointing at the `WorkloadCost` UID, and that under `envtest` (which runs no GC controller) you at least verify the reference is well-formed — cross-namespace references are invalid and silently break collection.

8. **Observe the backoff for real (optional, high value).** Run the controller against `kind` with a deliberately broken dependency and watch `workqueue_retries_total`, `workqueue_depth`, and `controller_runtime_reconcile_errors_total` climb; note how long it takes to reach the 1000-second cap and what the logs look like at that point (they look like nothing, which is the lesson).

**Acceptance:**

- Idempotency test green: two consecutive reconciles on identical state yield deep-equal status; pre-seeded stale status converges to the same value; conditions do not grow.
- Order-independence test green for two event orders plus three no-op reconciles.
- The "branch on event" / accumulator bug is present in git history (one commit) and removed in the next — a visible before/after with the failing test in between.
- No status field is mutated incrementally; every one is computed from inputs, and any genuine accumulator is guarded by an aligned, monotonic watermark.
- Finalizer test green, including the failing-cleanup case (object survives) and the exactly-once cleanup assertion.
- `observedGeneration` asserted equal to `generation` after convergence.
- A short note in the artifact naming the three requeue paths, which one your reconciler uses where, and where `Forget` would sit if you had written it in raw client-go.

## Common pitfalls

1. **Believing "idempotent" means "does nothing on the second call."** It means "produces the same *result*," which can absolutely involve writes — a create-or-update that pushes a drifted field back to desired every run is idempotent. The opposite error is just as bad: an `if status.Ready { return }` guard makes the second call a genuine no-op and therefore never corrects drift. Mechanism: idempotency is a property of the *result function*, not of the write count.

2. **Swallowed error / eternal success.** You `log.Error(err)` and `return ctrl.Result{}, nil`. The queue calls `Forget`, the key's backoff resets, and nothing retries — the object sits un-reconciled until an unrelated event pokes it. Symptom: `controller_runtime_reconcile_errors_total` flat at zero while objects have obviously wrong status. Correction: return the error. If you truly do not want a retry, return `reconcile.TerminalError(err)` so at least the metric records it.

3. **Missing watch.** Your reconcile depends on a secondary object — a `GPUCostPolicy`, a ConfigMap, a Node — that you never wired with `Owns()` or `Watches()`. Changes to it never enqueue the primary, so status goes stale silently and only the periodic resync (if any) eventually corrects it. Mechanism: the workqueue only receives keys someone enqueued. Correction: every input to your computation must be a watch, with an `EnqueueRequestsFromMapFunc` if the mapping is not owner-based.

4. **Cache never synced because of an RBAC gap.** The informer cannot `list`/`watch` a type, `WaitForCacheSync` blocks, and **no reconciles ever start** — the controller looks alive and does nothing. `envtest` will not catch it (it does not enforce RBAC); production will. Symptom: `controller_runtime_active_workers` stuck at 0 and a `failed to list …: forbidden` line at startup. Correction: audit the generated `ClusterRole` against every type you read, watch, or write, including subresources.

5. **A field that changes on every reconcile.** A timestamp, a duration, a re-marshalled map with non-deterministic ordering, a `resource.Quantity` re-serialised into a different canonical form. Every reconcile becomes a real write → a watch event → an enqueue → a reconcile. Symptom: one object's `resourceVersion` climbing continuously and `controller_runtime_reconcile_total` in the hundreds per second on an idle cluster. Correction: compare before writing (`equality.Semantic.DeepEqual(old.Status, new.Status)`) and skip the update when nothing changed.

6. **Conflating resync with a health check.** A resync firing means nothing was wrong; it is a local safety-net replay of the cache (lesson 04), not a signal. Treating resync-triggered reconciles as anomalies wastes on-call time chasing nothing.

7. **Adding the finalizer after creating external state.** Crash in that window and a subsequent delete removes the object immediately, orphaning whatever you created with no record of it. Correction: finalizer first, always. The worst case of finalizer-first is a finalizer on an object with nothing to clean up.

8. **`kubectl delete --grace-period=0 --force` as a fix for a stuck object.** That shortens the *termination grace period* (relevant to Pods) and does not touch finalizers at all. The object stays. Correction: diagnose which finalizer, and who owns it.

9. **Force-clearing a finalizer to "unstick" a namespace.** The cleanup never runs, external resources leak with no record of what leaked, and for a Namespace the contained objects are stranded in etcd, invisible to normal listing. Correction: read `namespace.status.conditions` — `NamespaceDeletionDiscoveryFailure`, `NamespaceDeletionContentFailure`, `NamespaceContentRemaining`, `NamespaceFinalizersRemaining` name the exact blocker, and the most common one is a broken aggregated APIService, not your controller at all.

10. **Assuming finalizers run in list order.** They do not, and the `ObjectMeta.Finalizers` doc comment says so explicitly, precisely because ordering "introduces significant risk of stuck finalizers." Correction: every finalizer must be independent.

## Self-check

- **Why must you never branch on which event fired? Give a concrete bug.**
  **Answer:** Because `reconcile.Request` contains only a `NamespacedName` — controller-runtime's own docs say it carries no information about any specific event or the object contents — and because the delivery path is explicitly allowed to coalesce, drop, reorder, replay, and restart. Concrete bug: `if createEvent { status.PodCount++ }`. Two rapid edits coalesce into one reconcile (the workqueue's `dirty` set holds a key once), so a create-plus-update counts once instead of twice; a periodic resync fires with nothing changed and increments again; a controller restart relists everything and increments from whatever baseline happened to be persisted. The count drifts arbitrarily and never self-corrects, and nothing errors. The fix is to *derive* `PodCount` by listing pods on every reconcile and setting it — no branch on event, no accumulation.

- **The three requeue paths and how backoff differs; where does `Queue.Forget()` belong?**
  **Answer:** (1) Return a non-nil **error** → the item is re-added with `RateLimited: true`, so the per-key exponential limiter climbs: `delay = min(5ms × 2^failures, 1000s)`, reaching the ~16.7-minute cap after about 18 consecutive failures. `Forget` is **not** called. (2) `ctrl.Result{RequeueAfter: d}, nil` → controller-runtime calls `Forget(req)` **and then** re-adds with `After: d`, so backoff is reset and the delay is exactly `d` — for polling something you cannot watch. (3) `ctrl.Result{}, nil` → `Forget(req)` and no requeue; only a watch re-enqueues. There is a fourth: `reconcile.TerminalError(err)` increments `controller_runtime_terminal_reconcile_errors_total` and does not requeue at all. `Queue.Forget(key)` belongs on **success** and on **permanent give-up past the retry cap** — never on a retry, because forgetting mid-retry erases the backoff you are deliberately accumulating and turns a failing dependency into a hot loop. And `Queue.Done(key)` must be called (deferred) for every `Get`, or the key stays in the `processing` set forever and that single object silently stops reconciling.

- **Reconcile is called twice for the same state — what must be true for that to be harmless?**
  **Answer:** The reconcile must be idempotent in the strict sense: the resulting state is a function of the inputs alone, not of call count or order. Concretely — derive every status field from current observations rather than mutating it incrementally; use create-or-update (or SSA) instead of blind `Create`; upsert list entries by key (`meta.SetStatusCondition`) instead of `append`; ignore `NotFound` on deletes; give any external side effect a deterministic idempotency key derived from the fact being recorded (and from `metadata.uid`, not name, so a delete-and-recreate does not collide); and guard any genuine accumulator with a monotonic watermark over closed, aligned windows so a repeat call adds exactly zero. It must also tolerate stale cache reads — re-read and re-derive rather than assuming your last write is visible — and treat `409 Conflict` as a normal requeue rather than a failure. Given those, the second call recomputes the identical target, finds the world already matching, writes the same status, and is a no-op in effect. That is precisely what resyncs, requeues, conflict retries, and process restarts depend on.

- **Your controller's backoff has climbed toward the cap for one object while every other object reconciles fine. How did it get there, and how do you reset it?**
  **Answer:** Every reconcile of that specific key returned a non-nil error, so `reconcileHandler` re-added it with `RateLimited: true` each time and the per-key `ItemExponentialFailureRateLimiter` doubled the delay: `baseDelay × 2^n` from 5 ms, clamped at `maxDelay` 1000 s. Backoff state is per key (a `failures` map in the limiter), which is exactly why one object can be at the cap while others are at zero. The idiomatic reset is to make one reconcile of that key **succeed**: return `ctrl.Result{}, nil` (or a valid `RequeueAfter`) with no error — both branches call `Queue.Forget(req)`, which deletes the key from the failures map. There is no "reset backoff" API to call directly; a clean success is the only sanctioned way. Diagnose it with `workqueue_retries_total` by name and `workqueue_queue_duration_seconds`, because from the outside a maxed-out backoff looks exactly like a controller that is merely slow.

- **What happens on delete if a finalizer never returns, and how do you avoid a stuck object?**
  **Answer:** The apiserver, seeing a non-empty `metadata.finalizers`, does not delete the object: it sets `deletionTimestamp`, sets `deletionGracePeriodSeconds`, bumps `generation`, responds `202 Accepted`, and leaves the object in storage. It is still readable, still reconciled, and its finalizer entries may now only be *removed*, never added. If the owning controller never removes its finalizer — because its cleanup keeps failing, because the controller is down, or because it was uninstalled — the object stays in `Terminating` forever, its name cannot be reused, and its namespace cannot finish deleting. `kubectl delete --grace-period=0 --force` does not help: it shortens the grace period, not the finalizer contract. Avoid it by: adding the finalizer *before* creating any external state; bounding every external call in the cleanup with a timeout; making the cleanup idempotent so retries are safe; deciding in advance whether an unreachable dependency should fail open (record zero, remove the finalizer, emit a Warning event) rather than block indefinitely; emitting Events so `kubectl describe` explains the block; and never shipping a finalizer whose controller might be uninstalled while CRs still exist. To diagnose one, read `metadata.finalizers` and `deletionTimestamp`, check whether the owning controller is running and holds its leader lease, and read its logs for that object; for a namespace, read `status.conditions` (`NamespaceDeletionDiscoveryFailure`, `NamespaceDeletionContentFailure`, `NamespaceContentRemaining`, `NamespaceFinalizersRemaining`).

- **Why is force-removing a finalizer dangerous, and when is it acceptable?**
  **Answer:** `kubectl patch … --type merge -p '{"metadata":{"finalizers":null}}'` tells the apiserver that no cleanup is needed, and the apiserver believes you and deletes the object. The cleanup then never runs: external resources the controller created — cloud load balancers, disks, IPs, license leases, or in this operator's case the final spend record — are orphaned, and the object that held the only in-cluster record of them is gone, so you cannot even enumerate what leaked. If the finalizer you removed was `foregroundDeletion`, dependents that were meant to be deleted first are left behind. For a Namespace it is worse: clearing the `kubernetes` finalizer removes the Namespace object while the objects inside it remain in etcd, unreachable through normal listing because their namespace no longer exists, and they resurface if a namespace with the same name is recreated. It is acceptable only when you have independently confirmed the cleanup is unnecessary — the external resource is already gone, or the controller has been decommissioned and you have manually performed its cleanup — and you record what was bypassed. The default response to a stuck finalizer is diagnosis, not force.

## Connections & what's next

This lesson is the correctness contract. Lesson 02 gave you the wire format, the metadata protocol (`generation`/`observedGeneration`, `finalizers`, `ownerReferences`), and the patch semantics it is written in; this lesson turns those into a discipline. Lesson 06 (controller-runtime in depth) extends the same discipline to owned children and garbage collection, where "idempotent" has to cover creating, updating, and deleting *other* objects rather than just writing your own status, and where the finalizer skeleton here becomes the operator's real spend-recording path. Lesson 08's webhooks are the other side of the same coin: admission is the edge-triggered, synchronous check; reconcile is the level-triggered, asynchronous convergence, and knowing which invariant belongs in which is a design skill.

The immediate next step is **lesson 04, Informers, caches, and workqueues** — the machinery that actually *implements* everything you took on faith here. How a key gets coalesced (the `dirty`/`processing` sets), how backoff is computed and stored per key, why a cache read can be stale and by how much, what resync really replays, and where `WaitForCacheSync` sits in the startup path. Everything in this lesson was a black-box contract; lesson 04 opens the box.

## References & further reading

**Primary sources**
- [controller-runtime `pkg/reconcile`](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile) — the `Reconciler` interface doc comment is the normative statement that "reconciliation is level-based," the `Result` struct carries the `Requeue`-is-deprecated note quoted above, and `TerminalError` is documented here.
- [controller-runtime `pkg/internal/controller/controller.go`](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/internal/controller/controller.go) — `reconcileHandler` is the source of the exact requeue branch table: which paths call `Forget`, which call `AddWithOpts({RateLimited: true})`, and where `Done` is deferred.
- [client-go `util/workqueue`](https://github.com/kubernetes/client-go/tree/master/util/workqueue) — `queue.go` for the `dirty`/`processing` dedup guarantee that makes coalescing possible, and `default_rate_limiters.go` for `DefaultTypedControllerRateLimiter` = `MaxOf(ItemExponentialFailure(5ms, 1000s), Bucket(10 qps, burst 100))` and the `baseDelay × 2^failures` formula.
- [Kubernetes API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) — spec/status ownership, the rule that `PUT`/`POST` must ignore `status`, `generation` vs `observedGeneration`, the standard condition fields, and the level-based statement that "the system will drive toward the most recent spec regardless of previous versions."
- [Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) — the deletion-blocking contract, the `202` response, the rule that you cannot add a finalizer after deletion is requested, and the explicit warning against force-removal.
- [Garbage collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) — owner references, `blockOwnerDeletion`, and the foreground/background/orphan propagation policies with their `foregroundDeletion` and `orphan` finalizers.
- [`kubernetes/sample-controller`](https://github.com/kubernetes/sample-controller/blob/master/controller.go) — the raw client-go loop; read `processNextWorkItem` and `handleErr` to see `Forget`/`AddRateLimited`/`Done` placed by hand.
- [Kubebuilder Book — Architecture](https://book.kubebuilder.io/architecture.html) — the manager → cache → workqueue → reconcile diagram in the framework you will ship.

**Real-world engineering**
- Tim Hockin, ["Edge vs. Level triggered logic"](https://speakerdeck.com/thockin/edge-vs-level-triggered-logic) (June 2017) — the design rationale from a founding Kubernetes engineer, via the hardware-interrupt analogy; the argument that level-driven software asserts state so clients can check and re-check at any time.
- controller-runtime issue [#392](https://github.com/kubernetes-sigs/controller-runtime/issues/392), "Reconcile loops using RequeueAfter never reset their failure count" (April 2019) — the historical backoff bug. **Note the correction:** current controller-runtime calls `Queue.Forget(req)` in the `RequeueAfter` branch, so this is fixed; read it as a case study in invisible backoff failures rather than as current behaviour.
- Stuart Leeks, ["Error back-off with controller-runtime"](https://stuartleeks.com/posts/error-back-off-with-controller-runtime/) — the same interaction worked through with code and measured delays.
- Uber Engineering, ["Migrating Uber's Compute Platform to Kubernetes"](https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/) — a production job-controller reconcile loop at scale.

**Deeper dives**
- [Google Cloud, "Troubleshooting namespaces stuck in the Terminating state"](https://cloud.google.com/kubernetes-engine/docs/troubleshooting/terminating-namespaces) — the vendor runbook for the finalizer failure mode, including reading `status.conditions` and the broken-APIService root cause; a good counterweight to the force-remove advice that dominates search results.
