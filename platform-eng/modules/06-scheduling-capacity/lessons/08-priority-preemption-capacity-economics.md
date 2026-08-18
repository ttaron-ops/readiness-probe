---
lesson: "06.8"
title: "Priority, preemption, and capacity economics"
module: "06"
concept: "Priority, preemption, and capacity economics"
status: not-started
est_time: "12h"
prev: "07-fragmentation-effective-capacity.md"
next: null
artifacts: []
sources: 12
---

# 06.8 · Priority, preemption, and capacity economics

> **Concept.** Priority tiers with *survivable* (checkpoint-aware) preemption, and a reserved / on-demand / spot commitment ladder for a fleet you cannot autoscale.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

This is the module's last lesson, and it deliberately pulls every earlier thread into one
decision. Lesson 7 measured the gap between allocated and usable capacity and put a dollar figure
on it — and ended by identifying a fourth kind of stranding that placement policy cannot touch:
**deliberate reserved headroom**, idle by design because a team is right to want a safety margin.
The only way to recover that capacity is to run someone else's work in the gap and take it back
instantly. Lessons 3–4 gave you the quota machinery for "take it back". Lesson 2 established that
a preempted *gang* has to re-admit atomically, not pod by pod.

This lesson answers the three questions that remain: **who gets to reclaim capacity, what does the
reclaiming cost, and how much of the fleet should you have committed to buying in the first
place?** Fairness (who gets preempted), survivability (what preemption actually costs), and
commitment strategy (what you buy) are one connected decision, not three. It is the decision your
target companies pay Staff-level compensation for someone to own.

Everything below about `kube-scheduler` preemption was read out of the `kubernetes/kubernetes`
tree cloned during this session at the v1.37 development head —
`pkg/scheduler/framework/preemption/{preemption.go,executor.go,util.go}`,
`pkg/scheduler/framework/plugins/defaultpreemption/default_preemption.go`,
`pkg/scheduler/apis/config/v1/defaults.go`, `pkg/apis/scheduling/{types.go,v1/helpers.go}` — and
the Kueue claims from `kubernetes-sigs/kueue` at the post-v0.19.1 head.

## Why this matters

This is where a FinOps background out-answers a pure SWE, and the reason is that both halves are
quantitative and most engineers only carry one of them.

Preemption is how you sweat a fixed fleet toward high *useful* utilisation without starving
production — Alibaba's OSDI '26 numbers from lesson 7 put a figure on it: preemption-aware
harvesting of idle capacity moved their GPU allocation ratio from **68% to 93%**. But preemption
is only economically usable if the preempted work *survives*, which makes it a joint
scheduler-plus-training-loop property rather than a scheduler knob. And the survival cost is not
a vibe: it is `T_ckpt/2 + T_restart` per event, it has a closed-form optimum, and the optimum
moved by an order of magnitude when async sharded checkpointing became production-mature.

Capacity planning for accelerators then breaks every reflex a cloud engineer has. You cannot
autoscale your way out of scarcity, so the lever is **commitment strategy**, not elasticity. "Design
the reserved/on-demand/spot mix and defend the break-even" is a standard senior interview
question — and the folklore answer ("reserve to your P50") is *wrong*. The right answer is a
newsvendor problem with a closed-form solution, and §14 derives it.

One more thing separates a strong answer from a junior-sounding one: **naming the market your
numbers came from**. GPU pricing is at least three markets with 4–6× spreads between them, and
quoting a neocloud rate as if it were universal is the fastest way to lose credibility with
someone who trades this capacity for a living.

## What's new here (calibration)

- **Lessons 3–4** gave you Kueue's quota, cohorts, borrowing, and the shape of its preemption
  policies. Assumed. §9 restates only the fields the tier design uses, and adds the candidate
  ordering and target-minimisation heuristics.
- **Lesson 1 §5(c)** established that `DefaultPreemption` is per-pod and only evicts strictly
  lower priority. That is the premise; this lesson opens the box and walks the algorithm.
- **Lesson 5** gave you KAI's preempt-versus-reclaim split and its priority-100 preemptibility
  threshold, and Volcano's bundle-ordering. Referenced, not re-derived.
- **Lesson 7** gave you fragmentation, defrag payback, and the `T_ckpt/2` term. §10 picks that
  term up and §11 optimises it.
- **Genuinely new here:**
  - **The PriorityClass API in full** — value ranges, the reserved band above 1 billion, the two
    auto-created system classes and their exact values, `globalDefault` semantics, and
    `preemptionPolicy: Never` as a distinct thing from low priority.
  - **The victim-selection algorithm as executable steps** — the *reprieve* loop that minimises the
    victim set, the group-aware `MoreImportantVictim` ordering, and the five ordered node tiebreaks.
  - **Where the grace period comes from**: the scheduler deletes victims with bare `DeleteOptions`,
    so the victim keeps its own `terminationGracePeriodSeconds`.
  - **`kube-scheduler`'s anti-cascade guard** — a real mechanism in the code, and the modern echo of
    Borg's "production never preempts production" rule.
  - **The optimal checkpoint interval, derived** — `T* = √(2C/λ)`, the Young/Daly result applied to
    preemption rate, with the equal-cost property at the optimum and a sensitivity curve.
  - **The commitment ladder as a newsvendor problem** — critical ratio `P(demand > Q*) = R/D`, and a
    worked 128-GPU ladder that lands on **P40 of demand, not P50**.

## Core concepts

### 1. The problem: one fleet, many urgencies

A shared GPU fleet serves work with genuinely different urgency. Production inference must not
wait. A researcher's interactive session should not wait long. A hyperparameter sweep can wait
indefinitely and can be killed at any moment, provided it resumes.

Queueing alone (lessons 3–4) handles the *arrival* side: a workload that does not fit waits until
quota frees. What queueing cannot do is take capacity back from work that is **already running**,
and without that, two things you want are impossible. **A guaranteed floor for production** — if
prod's quota is fully consumed by research work that borrowed it, prod waits, which is exactly what
a floor exists to prevent. And **backfilling deliberate headroom** — lesson 7 §12's fourth
stranding mechanism: a team reserving 32 GPUs and running 20 has 12 idle by design, and filling
them with batch work is free money *if and only if* you can evict that work the instant the owner
returns.

Preemption is the mechanism for both, and done badly it is a way to set money on fire: every
eviction destroys work, and if the evicted work cannot resume you have converted a scheduling
problem into a compute-hours loss. The rest of this lesson is who gets evicted (§2–§9), what each
eviction costs (§10), how to minimise it (§11–§12), and how all of that changes what you buy
(§13–§15).

### 2. Priority as a number: the PriorityClass API

`PriorityClass` is a cluster-scoped object in `scheduling.k8s.io/v1` that maps a name to an
integer. A pod references it by name in `spec.priorityClassName`; an admission controller resolves
it and writes the integer into `spec.priority`, which is what the scheduler actually reads.

The value space is structured, and the structure is enforced by API validation
(`pkg/apis/scheduling/types.go`):

| Constant | Value | Meaning |
|---|---|---|
| `DefaultPriorityWhenNoDefaultClassExists` | **0** | priority of a pod with no class, when no `globalDefault` exists |
| `HighestUserDefinablePriority` | **1,000,000,000** | ceiling for user-defined classes; above this is reserved for Kubernetes |
| `SystemCriticalPriority` | **2,000,000,000** | start of the system band (`2 × HighestUserDefinablePriority`) |
| `system-cluster-critical` | **2,000,000,000** | auto-created; "must run in the cluster, but can be moved to another node" |
| `system-node-critical` | **2,000,001,000** | auto-created; "must not be moved from their current node" |

The `system-` name prefix is reserved: validation rejects any user-created class whose name starts
with `system-` or whose value exceeds `HighestUserDefinablePriority`, unless it is exactly one of
the two auto-created classes. So the whole of `[−2³¹, 1e9]` is yours, and nothing you create can
outrank a system component.

Four fields, and three of them have non-obvious semantics:

- **`value`** (required) — any valid int32, including negative. Negative values are legitimate and
  useful for "run only on scraps".
- **`globalDefault`** (optional, default false) — this class applies to pods that name no class.
  Only one *should* be marked default; if several are, **the smallest value among them wins**. Do
  not rely on that tiebreak — set exactly one.
- **`description`** (optional) — free text. Use it; `kubectl describe priorityclass` is where an
  on-call engineer will look at 3am to find out what a tier means.
- **`preemptionPolicy`** (optional, defaults to `PreemptLowerPriority`) — either
  `PreemptLowerPriority` or **`Never`**. This is the field people miss, and it is the key to the
  tier design.

**`preemptionPolicy: Never` decouples two things that intuition welds together.** A pod with a
high value and `Never` gets **queue priority** — it sorts ahead of lower-priority pods in the
scheduler's `activeQ`, so it is attempted first — but it will not evict anything. It waits for
capacity like everyone else, just at the front of the line. That is the correct setting for a
large, important, *restartable-but-expensive* training job: you want it scheduled first, and you
do not want it triggering an eviction storm.

A complete three-tier ladder:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: gpu-prod
value: 900000                        # well below HighestUserDefinablePriority (1e9)
globalDefault: false
preemptionPolicy: PreemptLowerPriority   # prod MAY evict lower tiers
description: >
  Production inference and on-call-critical serving. Protected floor. May preempt
  lower tiers; never preempted itself.
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: gpu-research
value: 500000
globalDefault: true                  # pods with no class land here
preemptionPolicy: Never              # <-- queue priority WITHOUT eviction power
description: >
  Interactive training and experiments. Scheduled ahead of best-effort but never
  evicts anything: it waits rather than triggering a cascade. Reclaimable by prod.
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: gpu-best-effort
value: -10                           # negative is legal and expressive
globalDefault: false
preemptionPolicy: Never
description: >
  Sweeps, backfill, evaluation harnesses. Runs only on capacity nobody else wants,
  including idle reserved headroom. Freely evictable. MUST checkpoint: workloads
  without a documented resume path are rejected from this tier.
```

Two mechanical notes. **A pod's `spec.priority` is immutable after admission** — you cannot
promote a running job by editing it, you resubmit, which matters during incident response. And
**deleting a PriorityClass does not change already-admitted pods**, because the integer was copied
into the pod spec at admission time; the class is a lookup table, not a live reference.

### 3. Two priority systems, one cluster

On a Kueue fleet there are **two** priority numbers in play, they govern different decisions, and
conflating them is a classic interview stumble.

| | Pod `PriorityClass` | Kueue `WorkloadPriorityClass` |
|---|---|---|
| API | `scheduling.k8s.io/v1`, cluster-scoped | `kueue.x-k8s.io/v1beta2`, cluster-scoped |
| Set via | `pod.spec.priorityClassName` | `kueue.x-k8s.io/priority-class` label on the Job |
| Read by | `kube-scheduler` | Kueue's admission scheduler |
| Governs | queue order in `activeQ`; who may evict whom **at bind time** | queue order for admission; who may be **evicted from quota** |
| Granularity | one pod | the whole `Workload` |
| Changing it | immutable on a running pod | changing the class does not affect Workloads already created |

The division of labour: **Kueue decides which Workload holds quota; `kube-scheduler` decides which
pod holds a node.** A Workload Kueue evicts has all its pods deleted and the Job re-suspended — a
clean whole-workload operation. A pod `kube-scheduler` preempts is one pod, and if it belongs to a
gang the survivors are now below quorum (§10 prices that). You want both systems configured, with
the *ranking* mirrored across them — the values need not match, but if the orderings disagree the
two layers disagree about who is expendable.

### 4. The victim-selection algorithm, step by step

This is the part the concept line calls "the actual victim-selection ordering the scheduler uses",
and it is a favourite deep probe. Here it is as executable steps, read from
`pkg/scheduler/framework/preemption/preemption.go` and
`pkg/scheduler/framework/plugins/defaultpreemption/default_preemption.go` at the v1.37 head.

Preemption runs in **`PostFilter`** — only after the pod has failed `Filter` on every node.

**Step 0 — is the preemptor even eligible?** `PodEligibleToPreemptOthers` returns false, with a
reason, when either:

- the pod has `spec.preemptionPolicy: Never`; or
- the pod already has a `status.nominatedNodeName`, that node is not
  `UnschedulableAndUnresolvable`, and **there is still a pod on it terminating due to a previous
  preemption by this pod.**

The second condition is the anti-cascade guard, and §8 unpacks why it matters.

**Step 1 — which nodes could preemption possibly help?** `nodesWherePreemptionMightHelp` filters
to nodes whose `Filter` failure was merely `Unschedulable` (fixable by removing pods) rather than
`UnschedulableAndUnresolvable` (a node selector mismatch, an untolerated taint — removing pods
changes nothing).

**Step 2 — sample, do not enumerate.** On a large fleet the scheduler does not dry-run every
candidate node. `DryRunPreemption` evaluates a subset sized by the `DefaultPreemptionArgs`
defaults:

| Arg | Default | Effect |
|---|---|---|
| `minCandidateNodesPercentage` | **10** | evaluate at least 10% of feasible nodes |
| `minCandidateNodesAbsolute` | **100** | …but never fewer than 100 nodes |

So on a 2,000-node fleet, roughly 200 nodes are simulated, starting from a rotating offset so the
same nodes are not always chosen. **Preemption is therefore approximate at scale** — it finds a
good node, not provably the best one.

**Step 3 — per candidate node, `SelectVictimsOnNode`.** This is the core, and it has a
counter-intuitive shape: it removes everything, then puts back as much as it can.

```
  SELECT VICTIMS ON ONE NODE — remove-all-then-reprieve
  ══════════════════════════════════════════════════════════════════════════════════════

  a) ELIGIBLE = pods on this node with priority STRICTLY LESS than the preemptor's
                (plus any plugin-specific IsEligiblePod filter)
     if ELIGIBLE is empty  →  "No preemption victims found for incoming pod"  ✗ node out

  b) remove ALL of ELIGIBLE from a copy of the node, then re-run Filter
     if the preemptor STILL does not fit  →  ✗ node out
     (this is the cheap early exit: no point optimising a node that can't work)

  c) sort ELIGIBLE by MoreImportantVictim, DESCENDING  (most important first)

  d) split into PDB-VIOLATING and NON-VIOLATING
     (violating = evicting it would breach a PodDisruptionBudget)

  e) REPRIEVE PASS — add pods back one at a time, most important first,
     PDB-violating group first, then non-violating:

        for v in violating + nonviolating:
            put v back
            does the preemptor still fit?
              YES → v is REPRIEVED (it survives; it was not needed as a victim)
              NO  → take v out again; v is a VICTIM

     ┌──────────────────────────────────────────────────────────────────────────┐
     │  node has 8 GPUs.  preemptor needs 2.  eligible victims each hold 1 GPU:  │
     │     v1 (prio 400, 1 GPU)  v2 (prio 200, 1 GPU)  v3 (prio 100, 1 GPU)     │
     │     v4 (prio 100, 1 GPU)  v5 (prio  -5, 1 GPU)   … 3 GPUs held by prod   │
     │                                                                          │
     │  (b) remove all five → 5 free + 0 = fits ✓                               │
     │  (c) order: v1, v2, v3, v4, v5      (descending importance)              │
     │  (e) put v1 back → 4 free ≥ 2 ✓ REPRIEVED                                │
     │      put v2 back → 3 free ≥ 2 ✓ REPRIEVED                                │
     │      put v3 back → 2 free ≥ 2 ✓ REPRIEVED                                │
     │      put v4 back → 1 free < 2 ✗ VICTIM                                   │
     │      put v5 back → 1 free < 2 ✗ VICTIM   (v4 stayed out, so still 1)     │
     │                                                                          │
     │  VICTIMS = {v4, v5}  — the two LEAST important, and only two.            │
     └──────────────────────────────────────────────────────────────────────────┘

  f) return (victims, number of PDB violations among them)
```

Why remove-all-then-reprieve rather than "add victims until it fits"? Because feasibility is not
monotone in a single resource: `Filter` runs the *whole* plugin chain, including topology spread
and inter-pod affinity, whose verdicts can change non-monotonically as pods come back. The
remove-all step establishes feasibility once; the reprieve pass then minimises the victim set under
the real predicate rather than a resource-counting approximation.

**Step 4 — choose among the candidate nodes.** `pickOneNodeForPreemption` applies five score
functions **in order**, each a tiebreak for the previous:

1. **Fewest PDB violations.**
2. **Lowest highest-priority victim** — prefer the node whose *most important* victim is least
   important.
3. **Smallest sum of victim priorities.** (It adds `MaxInt32 + 1` to every priority before summing,
   so a node with a few negative-priority victims is not preferred over one with fewer victims at
   the same negative priority.)
4. **Fewest victims.**
5. **Latest earliest-start-time among victims** — i.e. **destroy the youngest work**, which has the
   least accumulated progress to lose.
6. If still tied, the first node in the list.

Criterion 5 is the one to name in an interview, because it is the scheduler doing exactly the
cost reasoning §10 formalises: all else equal, evict the job that has run the least.

**Step 5 — execute.** In `Executor.PreemptPod`: a victim waiting at `Permit` (a gang member,
lesson 2) is rejected **in scheduler memory** with no API call and goes back to the backoff queue;
a victim in `PreBind` has its binding cancelled in memory; otherwise the victim's status is patched
with a `DisruptionTarget` condition, `Reason: PreemptionByScheduler`, message
`"<schedulerName>: preempting to accommodate a higher priority <type>"`, and the pod is **deleted**.

**Step 6 — the preemptor waits.** It gets `status.nominatedNodeName` set to the chosen node and is
requeued. It does *not* skip the queue next time; it re-runs a normal scheduling cycle, which by
then should succeed. Meanwhile Step 0's guard stops it preempting *again* on that node while the
victims are still terminating.

### 5. `MoreImportantVictim`: the ordering, including the group-aware part

Step 3(c) and Step 4 both need a total order over victims. `MoreImportantVictim`
(`pkg/scheduler/framework/preemption/util.go`) applies five rules in sequence. The middle ones are
new with the pod-group work and are the ones most write-ups miss:

1. **Priority.** Higher `spec.priority` is more important. This dominates everything else.
2. **Workload type.** `CompositePodGroup` (rank 3) > `PodGroup` (rank 2) > individual `Pod`
   (rank 1). *Rationale in the code's own comment: preserve group integrity.* At equal priority,
   the scheduler would rather evict a lone pod than break a gang.
3. **Runtime, for two individual pods.** The one that started earlier (longer runtime) is more
   important — first-come-first-served, and it protects accumulated work.
4. **Group size, for two groups of the same type.** More members is more important. *Rationale in
   the code: avoid the high cost of rescheduling massive jobs.*
5. **Start time, as the group tiebreak.** The group with the oldest pod wins.

Read rules 2 and 4 together and you have the scheduler encoding lesson 2's gang amplification
directly: breaking a big gang is expensive, so at equal priority it prefers not to. Rules 3 and 5
encode lesson 7's `T_ckpt/2` intuition: older work has more to lose.

### 6. The grace period, and where it actually comes from

The single most operationally important line in the preemption path is easy to miss. After
patching the `DisruptionTarget` condition, the executor calls `util.DeletePod`, which is:

```go
return cs.CoreV1().Pods(pod.Namespace).Delete(ctx, pod.Name, metav1.DeleteOptions{})
```

**Empty `DeleteOptions` — no `GracePeriodSeconds` override.** Therefore the victim gets its own
`spec.terminationGracePeriodSeconds`, whose default is 30 seconds. The scheduler never shortens
your checkpoint budget, and never lengthens it either.

That makes `terminationGracePeriodSeconds` a first-class capacity-economics knob, not a Kubernetes
trivia field:

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 300   # 5 minutes to catch SIGTERM and flush a
                                           # final checkpoint before SIGKILL
```

The tradeoff is symmetric and you must own both sides:

- **Too short** → the trainer cannot finish a final checkpoint, so every preemption costs the full
  `T_ckpt/2` instead of ~0. On a synchronous checkpointing job whose save takes 49 seconds, a
  30-second grace period guarantees the final save fails.
- **Too long** → the preemptor waits. The scheduler does not block on termination, but the *node's
  resources are not actually free* until the victim's containers exit, so a 600-second grace period
  on a hung victim is 10 minutes during which the high-priority pod cannot bind. Worse,
  `PodEligibleToPreemptOthers` will refuse to let the preemptor preempt elsewhere while that
  victim terminates.

**The rule that falls out: set `terminationGracePeriodSeconds` to just over your measured
checkpoint-flush time, and measure it rather than guessing.** With async sharded checkpointing
(§12) the flush is short and you can afford a short grace period; with synchronous saves you
cannot, and that is a real cost of the old mechanism.

### 7. The preemption cascade, as a timeline

Put §4–§6 together and this is what a preemption looks like on the wall clock, with the loss
window marked. This is the picture to be able to draw.

```
  PREEMPTION TIMELINE — prod pod preempts a best-effort training job
  ══════════════════════════════════════════════════════════════════════════════════════
  t=0        prod pod P (priority 900000) fails Filter on every node
             │  "0/64 nodes are available: 64 Insufficient nvidia.com/gpu"
             ▼
  t+0.00s    PostFilter → DefaultPreemption.Preempt()
             │  Step 0 eligible ✓   Step 1 candidate nodes   Step 2 sample
             │  Step 3 remove-all → re-Filter → sort → reprieve
             │  Step 4 five ordered tiebreaks → node gpu-node-41
             ▼
  t+0.02s    VICTIM CHOSEN: best-effort workload B, 2 pods on gpu-node-41
             │  patch: status.conditions += DisruptionTarget
             │         reason=PreemptionByScheduler
             │  DELETE pod  (metav1.DeleteOptions{} — B keeps its OWN grace period)
             │  P.status.nominatedNodeName = gpu-node-41
             ▼
  t+0.03s    ┌─── SIGTERM delivered to B's containers ─────────────────────────────┐
             │  ██████ CHECKPOINT LOSS WINDOW ██████                                │
             │  work destroyed = now − B's last checkpoint                          │
             │      E[loss] = T_ckpt / 2   for a uniformly-timed eviction           │
             │  · B catches SIGTERM and flushes in time  → loss ≈ 0                 │
             │  · no handler, or flush > grace period    → loss = T_ckpt / 2        │
             │  · B does not checkpoint at all           → loss = ENTIRE runtime    │
             └─────────────────────────────────────────────────────────────────────┘
             │  clock running: terminationGracePeriodSeconds (default 30 s)
             ▼
  t+30s      SIGKILL if still alive. containers exit. GPUs ACTUALLY released.
             │  ← the DELETE at t+0.02s was only a promise
             ▼
  t+31s      P re-attempts, Filter passes on gpu-node-41, P binds.
             │
             │  MEANWHILE t+0.03s → t+30s: P cannot preempt anywhere else — Step 0's
             │  guard blocks it. THIS IS THE ANTI-CASCADE MECHANISM. Without it P
             │  would see the node still full and preempt a SECOND victim elsewhere,
             │  then a third, until the terminations landed.
             ▼
  t+35s      B's controller re-creates it → Kueue re-queues Workload B, SUSPENDED,
             │  holding ZERO GPUs (lesson 2 §9) until quota frees
             ▼
  t+???      B re-admitted and RESTARTS: image (cached) + model/optimizer load +
             │  framework/NCCL init + warm-up  =  T_restart
             └── cost of this one preemption:  m_B × ( loss + T_restart )  GPU-hours
```

Two facts on that timeline are worth stating explicitly because they surprise people:

- **The DELETE does not free the GPU.** Capacity becomes real at container exit, up to
  `terminationGracePeriodSeconds` later. A preemption-driven capacity plan that assumes instant
  release is wrong by that interval, per event.
- **The preemptor is not privileged on its retry.** It goes back through a normal scheduling
  cycle. If something else grabs the node in between — possible, though `nominatedNodeName` makes
  the scheduler account for it — the preemption was wasted and the victim died for nothing.

### 8. Cascades, and why Borg forbade same-tier preemption

Google's Borg (Verma et al., EuroSys 2015) organised work into priority bands — monitoring,
production, batch, and best-effort/"gratis" — and imposed a rule that looks arbitrary until you
think about the dynamics: **production-tier tasks do not preempt each other**, even though they
are nominally comparable and preemption "should" be neutral between equals.

The reason is **preemption cascades**. Allow same-tier preemption and task A evicts task B; B is
now unscheduled and must be re-placed; to fit, B evicts C; C evicts D. One legitimate reclaim
becomes a chain reaction that destabilises the entire tier — and every link in the chain destroys
work. The fix is structural, not a tiebreak rule: same-tier tasks simply cannot preempt each
other, full stop.

`kube-scheduler` gets part of the way there by construction and part of the way there by an
explicit guard:

| Mechanism | What it prevents |
|---|---|
| Victims must be **strictly lower** priority (`victim.Priority() < preemptor.Priority()`) | same-tier preemption within one PriorityClass — A cannot evict B at the same value |
| Step 0's guard: not eligible while a previously nominated node still has victims terminating | the *rapid* cascade — one pod preempting on node after node while earlier victims drain |
| `preemptionPolicy: Never` on a tier | that tier initiating any chain at all |

What is **not** prevented is a chain *across* tiers: prod evicts research, and research's requeued
Workload may (with `reclaimWithinCohort: LowerPriority`) evict best-effort elsewhere. That is by
design — each link moves work strictly down the ladder, so the chain terminates at the bottom
tier. **A ladder is cascade-safe when every preemption strictly decreases the tier of the work
disturbed, bounding the chain by the number of tiers.** That is the generalisation of Borg's rule,
and the `preemptionPolicy: Never` on `gpu-research` in §2 is what buys it: queue priority over
best-effort without the ability to start a chain.

### 9. Kueue's layer: quota reclaim, not node reclaim

`kube-scheduler` preemption is about a node. Kueue preemption is about **quota**, it operates on
whole `Workload`s, and it is what the deliverable actually demonstrates. Three fields on
`ClusterQueue.spec.preemption`, all defaulting to the safe value:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: cq-prod
spec:
  cohort: fleet
  preemption:
    # Can a pending Workload here evict Workloads in OTHER ClusterQueues in the
    # cohort that are using more than their nominal quota?
    #   Never (default) | LowerPriority | Any
    reclaimWithinCohort: Any            # prod takes back its floor from anyone
                                        # who borrowed it, regardless of priority

    # Can a pending Workload here evict Workloads in THIS ClusterQueue?
    #   Never (default) | LowerPriority | LowerOrNewerEqualPriority
    withinClusterQueue: LowerPriority

    # Can a Workload that needs to BORROW evict Workloads in other queues?
    # Classic Preemption only — not compatible with Fair Sharing.
    borrowWithinCohort:
      policy: Never                     # borrowing must never cost someone else
  resourceGroups:
    - coveredResources: ["nvidia.com/gpu"]
      flavors:
        - name: gpu-h100
          resources:
            - name: "nvidia.com/gpu"
              nominalQuota: 32          # prod's protected floor
              borrowingLimit: 0         # prod never borrows; it is sized to its need
```

The distinction that matters for the tier design: **`reclaimWithinCohort: Any` lets prod take back
its own nominal quota from a borrower irrespective of the borrower's priority.** That is what makes
the floor a floor. `LowerPriority` would leave prod blocked by a high-priority research Workload
that happened to borrow prod's GPUs — which is precisely the failure the floor exists to prevent.
And the guarantee that falls out, worth memorising: **capacity a queue is using *within* its own
nominal quota is never reclaimable by another queue.** Only borrowed capacity — usage above
nominal quota — is at risk from a cohort reclaim.

Kueue's candidate ordering, for the classic (non-fair-sharing) algorithm, is a three-key sort:

1. Workloads from **borrowing** queues in the cohort, before non-borrowing ones.
2. **Lowest priority** first.
3. **Most recently admitted** first — same instinct as `kube-scheduler`'s "latest start time"
   tiebreak: destroy the youngest work.

And then a step that is easy to overlook and materially reduces damage: after greedily qualifying
enough targets to make the preemptor fit, **Kueue traverses the target list in reverse and removes
any target whose quota is not actually needed** — the whole-Workload analogue of `kube-scheduler`'s
reprieve pass. Both layers converge on the same principle: *find a set that works, then shrink it.*

One structural difference with a direct cost consequence: **Kueue evicts a whole Workload, never
part of one.** For a gang that is strictly correct — it avoids §10's amplification entirely — and
it is one of the reasons Kueue is the right layer for this in a GPU fleet. Lesson 5 covers how
Volcano's `gangpreempt`/`gangreclaim` bundles and KAI's preempt/reclaim split reach the same
property from a different direction.

### 10. What a preemption actually costs

Now price it. Define, per preemption event on a job holding `m` GPUs:

```
  loss        = time since the job's last durable checkpoint         [hours]
  T_restart   = re-queue wait + image/model/optimizer load + init + warm-up
  cost_event  = m × ( loss + T_restart )                            [GPU-hours]
```

If evictions arrive at times uncorrelated with the checkpoint cycle — which is the right default
assumption, since preemption is driven by *other people's* arrivals — then the eviction lands
uniformly within a checkpoint interval of length `T_ckpt`, and:

```
  E[loss] = T_ckpt / 2
  E[cost_event] = m × ( T_ckpt/2 + T_restart )   GPU-hours
```

That is the term lesson 7 §10 used for defrag payback, and it is the same term because a defrag
*is* a preemption you initiated yourself.

**Three corrections that separate a good answer from a complete one:**

**(a) No checkpointing collapses the model.** With no resume path, `loss` is not `T_ckpt/2` — it is
the job's **entire elapsed runtime**. A 40-hour, 8-GPU job preempted at hour 39 costs 312
GPU-hours; at a **$2.35/GPU-hr H100 on-demand snapshot** (specialized-neocloud segment, §15) that
is $733 destroyed by one scheduling decision. Hence the best-effort tier's admission criterion in
§2: no documented resume path, no admission.

**(b) Gang amplification.** `kube-scheduler` preempts *pods*. Evicting one pod from a healthy
8-pod gang does not free 1 GPU of useful capacity — it strands 7 more, because the remaining ranks
block in the collective barrier (lesson 1) and produce nothing until the gang is whole again:

```
  naive accounting:  freed = 1 GPU
  real accounting:   freed = 1 GPU,  stranded = 7 GPUs,  net = −6 GPU-equivalents

  cost_event(gang, pod-granular) = m_gang × ( loss + T_restart + T_requeue )
                                   ↑ the WHOLE gang pays, not the evicted pod
```

And `T_requeue` is not small: the gang must re-admit atomically (lesson 2), queueing behind
everything else, possibly under a `required` topology constraint (lesson 6) that is harder to
satisfy than when it first ran. **This is the argument for whole-workload eviction granularity —
Kueue's default, Volcano's `gangpreempt` bundles, KAI's whole-workload preempt — a correctness
argument with a dollar attached, not a preference.**

**(c) The benefit side must be named too.** A preemption is only worth its cost if the freed
capacity runs work that could not otherwise have run. The full inequality, mirroring lesson 7's
defrag condition:

```
  preemption is worth it  ⟺  value(preemptor's work) > Σ_victims m_v ( loss_v + T_restart_v ) × rate
```

For prod reclaiming its own floor, the left side is "the service stays up" and the comparison is
not close. For a research job evicting a sweep to shave 20 minutes off a queue wait, it may well
be.

**Fleet-level rate.** What you budget for is the steady-state overhead:

```
  wasted_GPU_hours_per_day = Σ over running preemptible jobs j of
                                m_j × λ_j × ( T_ckpt,j/2 + T_restart,j ) × 24

  where λ_j = preemptions per hour experienced by job j
```

Worked, on the best-effort tier: 20 concurrently-running jobs averaging 4 GPUs each, 4 evictions
per job per day, `T_ckpt = 30 min`, `T_restart = 5 min`:

```
  per event   = 4 GPUs × (0.25 h + 0.083 h)          = 1.33 GPU-hours
  per day     = 20 jobs × 4 events × 1.33            = 106.4 GPU-hours/day
  per year    = 106.4 × 365                          = 38,836 GPU-hours/year
  at $2.35/GPU-hr                                    = $91,265/year of pure overhead
```

That is the number §11 attacks.

### 11. The optimal checkpoint interval, derived

Checkpointing more often shrinks the rework term and grows the checkpointing term. There is an
optimum, it has a closed form, and it is the same result Young derived in 1974 for supercomputer
failures — with **preemption rate substituted for failure rate**.

**Setup.** Let:

- `C` = the cost of taking one checkpoint, in seconds of compute time lost or blocked
- `T` = the checkpoint interval, in seconds of useful compute between checkpoints
- `λ` = preemption rate, events per second (`MTBP = 1/λ`, mean time between preemptions)
- `R` = restart overhead per preemption, seconds

**Overhead per unit of wall-clock time** has three additive terms:

```
  checkpointing:  C / T          (one C every T seconds of work)
  rework:         λ · T/2        (λ events per second, each destroying T/2 on average)
  restart:        λ · R          (independent of T — you cannot optimise it away here)

  Ω(T) = C/T + λT/2 + λR
```

**Minimise.** Only the first two terms depend on `T`:

```
  dΩ/dT = −C/T² + λ/2 = 0
  ⇒  T² = 2C/λ
  ⇒  T* = √(2C/λ) = √(2·C·MTBP)                    ← the Young/Daly result
```

**Two properties worth carrying.** Substituting `T*` back gives `C/T* = λT*/2 = √(Cλ/2)` — the two
variable terms are **equal at the optimum**. So: *at the optimal interval you spend exactly as much
time checkpointing as you lose to rework.* That is a field diagnostic requiring no arithmetic — if
the two overheads are far apart, your interval is wrong, in the direction of the larger term. And
`Ω(T*) = √(2Cλ) + λR`: minimum overhead grows as the **square root** of both the checkpoint cost and
the preemption rate, so cutting `C` by 80× (which §12 does) reduces it by ~89%.

**Worked, with the §10 fleet.** Preemption rate 4 per day → `λ = 4/86400 = 4.63×10⁻⁵ s⁻¹`,
`MTBP = 6 h`. Restart `R = 300 s`.

*Synchronous full checkpoint.* A 7B-parameter model in mixed precision with Adam carries roughly
12–16 bytes/parameter of durable state (fp32 master weights + two fp32 moments + the bf16 copy) —
call it ~98 GB. Written to network storage at an aggregate 2 GB/s, `C ≈ 49 s`:

```
  T* = √(2 × 49 / 4.63e-5) = √(2,116,800) = 1,455 s = 24.2 minutes

  at T*:  checkpointing  3.37%
          rework         3.37%     ← equal, as predicted
          restart        1.39%
          ───────────────────────
          TOTAL          8.12% overhead
```

*Async sharded checkpoint* (§12), where only the device→host copy blocks, `C ≈ 0.6 s`:

```
  T* = √(2 × 0.6 / 4.63e-5) = √(25,920) = 161 s = 2.7 minutes

  at T*:  checkpointing  0.37%
          rework         0.37%
          restart        1.39%
          ───────────────────────
          TOTAL          2.13% overhead
```

**The curve, so you can see how forgiving the optimum is:**

```
  OVERHEAD Ω(T) vs CHECKPOINT INTERVAL     λ = 4/day, R = 300 s
  ══════════════════════════════════════════════════════════════════════════════════════
  scale: one # ≈ 1% overhead

  SYNCHRONOUS  (C = 49 s)                       ASYNC SHARDED  (C = 0.6 s)
   0.5 min │###############...(165%)             0.5 min │###  3.5%
   1   min │###############...( 83%)             1   min │##   2.5%
   2   min │###########################  42%     2   min │##   2.2%
   3   min │#############################  29%   3   min │##   2.1%  ◀ T* = 2.7 min
   5   min │##################  18%              5   min │##   2.3%
  10   min │###########  10.9%                  10   min │###  2.9%
  15   min │#########  8.9%                     15   min │####  3.5%
  24.2 min │########  8.1%   ◀ T* = 24.2 min    24.2 min │#####  4.8%
  40   min │#########  9.0%                     40   min │#######  7.0%
  60   min │###########  11.1%                  60   min │##########  9.7%
 120   min │###################  18.7%         120   min │##################  18.1%

  Note the ASYMMETRY: the penalty for checkpointing too OFTEN is much steeper than
  for too rarely (the C/T term blows up as T→0, the λT/2 term grows only linearly).
  When uncertain, err LONG of T*, not short.
```

**And the inversion that matters most for capacity planning.** Fix an overhead budget and ask how
much preemption you can tolerate. Solving `√(2Cλ) + λR ≤ budget` for `λ`:

```
  budget = 5% overhead,  R = 300 s

  synchronous (C = 49 s):   λ_max = 1.98e-5 /s  =  1.71 preemptions/day
  async       (C = 0.6 s):  λ_max = 1.26e-4 /s  = 10.86 preemptions/day
```

**Async sharded checkpointing buys roughly 6× more preemption headroom at the same overhead
budget.** That is not a micro-optimisation. It is the difference between a backfill tier you can
evict twice a day and one you can evict ten times a day — which is exactly how aggressively you
can harvest the idle reserved headroom §14 will show you paying for.

### 12. Why `C` collapsed: async, sharded distributed checkpointing

The classic framing treats `C` as roughly fixed — "checkpointing is slow, so do it every N minutes
and eat the risk in between." That folklore is dated, and the mechanism behind why is a hardware
argument, not a software-cleverness argument.

PyTorch's `torch.distributed.checkpoint` (DCP) — production-mature since PyTorch 2.1, with
`save`, `async_save` and `load` as documented public APIs — changes two things:

**Sharded, parallel saves.** Each rank writes **only its own shard** of the model and optimizer
state, in parallel, into per-rank files with a metadata index. The classic bottleneck was the
opposite — gather the full state onto rank 0 and serialise it from one process — which makes `C`
scale with total state size and never improve with world size. With sharding, `C` scales with
`state_size / world_size` and *improves* as the job grows.

**Async save.** The GPU-blocking portion shrinks to the **device→host memory copy**; the durable
write happens on background CPU threads, overlapped with the next training steps. The hardware
asymmetry is doing the work: the D2H copy runs at PCIe/NVLink-to-host bandwidth (tens of GB/s),
while the slow network write to object storage never touches the GPU's critical path.

Numerically, for the §11 example: 98 GB of state, 8 ranks, ~12.25 GB per rank, D2H at an effective
~20 GB/s gives **`C ≈ 0.6 s`** of GPU-blocking time versus **`C ≈ 49 s`** for the synchronous
gather-and-write path. Roughly **80× smaller**, which through `T* = √(2C/λ)` is a **9× shorter
optimal interval** and, through `Ω(T*) = √(2Cλ) + λR`, a **74% reduction in total overhead**
(8.12% → 2.13%) at the same preemption rate.

Two operational caveats to state, because an answer that only sells the upside is a weak one:

- **Async save consumes host RAM** for the staged copy — roughly the size of the shard, per rank,
  held until the background write completes. On a node with 8 ranks each staging 12 GB that is
  ~96 GB of pinned host memory. Budget for it or the job OOMs on the host side.
- **The final checkpoint on SIGTERM must complete inside `terminationGracePeriodSeconds`** (§6). An
  async save that is still flushing when SIGKILL arrives is not durable. A correct SIGTERM handler
  waits for the in-flight async save *and* takes a final synchronous one — which means your grace
  period must cover a synchronous flush, not just the D2H copy.

**The platform-engineering point:** checkpoint frequency is a decision the *training code* owner
makes, but its economic consequences are paid by the *platform*. A team checkpointing every 30
minutes "because it's slow", unaware that async sharded saves exist, is unknowingly capping how
much of the fleet's idle headroom you can harvest. Surfacing that with the §11 arithmetic and a
concrete pointer is a higher-leverage platform intervention than any scheduler tuning.

### 13. Why you cannot autoscale a GPU cluster

CPU autoscaling rests on an assumption that is simply false for accelerators: that supply is
elastic. Three independent mechanisms break it.

**(a) Supply is finite and booked forward.** In a shortage, on-demand H100/H200/B200 capacity is
*out of stock* in your region and zone for hours or days. Cluster Autoscaler's behaviour is
instructive: it issues a scale-up request, the cloud API returns a capacity error, and CA marks the
node group as *backoff* and stops asking. Your pods stay pending and the autoscaler is behaving
correctly — there is no price at which the capacity appears, because it is physically allocated to
someone else's contract.

**(b) Cold start is minutes, not seconds.** A GPU node has to complete a chain no CPU node does:

| Stage | Typical duration | Why |
|---|---|---|
| Cloud instance provision + boot | 2–5 min | GPU instance families are slower to allocate than general-purpose |
| GPU driver + container toolkit | 0–3 min | zero if baked into the image; minutes if installed by an operator DaemonSet |
| Device plugin registers `nvidia.com/gpu` | 10–60 s | the node is not schedulable for GPU pods until this completes |
| Container image pull | 30 s – 5 min | CUDA/PyTorch images are 10–20 GB; cold registry pulls dominate |
| Model weights | 15 s – several min | 7B bf16 ≈ 14 GB; 70B ≈ 140 GB, at object-storage throughput |

Total: **5–15 minutes** before the node serves a single step. Module 07 covers the cold-start
breakdown properly; the relevant conclusion here is that reactive autoscaling is structurally too
slow for spiky GPU demand.

**(c) The unit is lumpy and expensive.** You scale in 8-GPU nodes. At a $2.35/GPU-hr snapshot each
node is ~$19/hr, ~$165k/yr. A mis-sized scale-up is an immediate, large bill rather than a rounding
error, which changes the decision from "react and correct" to "plan and commit."

**The consequence: for GPUs you commit capacity ahead of demand and *schedule* within it — which is
what this whole module has been about — rather than autoscaling to meet it.** Elasticity is
replaced by a portfolio decision, and that decision has a right answer.

### 14. The commitment ladder, derived as a newsvendor problem

The folklore is "reserve to your P50 and burst the rest on-demand." That is arbitrary. The problem
has a standard form and a closed-form solution, and knowing it is a genuine differentiator.

**Setup.** You choose a reserved quantity `Q` (GPUs) before observing demand. Let:

- `R` = reserved rate, $/GPU-hr — paid on all `Q` GPUs **whether or not you use them**
- `D` = on-demand rate, $/GPU-hr — paid only on GPUs used above `Q`
- `X` = demand, a random variable with distribution from your own showback history

Expected hourly cost:

```
  E[cost(Q)] = Q·R  +  E[ max(0, X − Q) ]·D
```

Differentiate with respect to `Q`. Adding one more reserved GPU costs `R` always, and saves `D`
exactly when demand exceeds `Q`:

```
  dE/dQ = R − D · P(X > Q) = 0
  ⇒  P(X > Q*) = R / D                      ← the critical ratio
  ⇒  Q* = the (1 − R/D) quantile of demand
```

**That is the whole result.** The cheaper the commitment relative to on-demand, the higher the
quantile you reserve to. Note where P50 comes from: it is correct only when `R/D = 0.5` — reserved
at exactly half the on-demand price. It usually is not.

**Worked on the module's 128-GPU fleet.** Demand: production inference needs a firm **32 GPUs**;
three research teams' combined hourly demand, taken from a month of showback samples, is:

| Research demand `X` | fraction of hours | cumulative | `P(X > Q)` |
|---|---|---|---|
| 24 GPUs | 0.10 | 0.10 | 0.90 |
| 40 GPUs | 0.15 | 0.25 | 0.75 |
| 56 GPUs | 0.25 | 0.50 | **0.50** |
| 72 GPUs | 0.20 | 0.70 | 0.30 |
| 88 GPUs | 0.20 | 0.90 | 0.10 |
| 96 GPUs | 0.10 | 1.00 | 0.00 |

Rates (specialized-neocloud snapshot, §15): `R = $2.35`, `D = $3.90`.

```
  critical ratio  R/D = 2.35 / 3.90 = 0.6026

  find the largest Q with P(X > Q) ≥ 0.6026, then step up one:
     P(X > 40) = 0.75  >  0.6026   → 40 is too little
     P(X > 56) = 0.50  ≤  0.6026   → Q* = 56 research GPUs

  Q*_total = 32 (prod, firm) + 56 (research) = 88 GPUs reserved
```

**56 is the P50 of this particular distribution, but that is a coincidence** — the *rule* is
`P(X>Q) ≤ 0.6026`, the P39.7 threshold crossing. Change the rates and the answer moves: a 3-year
commitment at `R/D = 0.35` gives `Q* = 72`, because the cheaper commitment justifies reserving
further up the demand curve.

Verify by brute force over the grid, which also gives you the write-up numbers:

```
  reserve  32 GPUs total  reserved $  658,752 + on-demand $2,186,496 = $2,845,248
  reserve  56 GPUs total  reserved $1,152,816 + on-demand $1,366,560 = $2,519,376
  reserve  72 GPUs total  reserved $1,482,192 + on-demand $  874,598 = $2,356,790
  reserve  88 GPUs total  reserved $1,811,568 + on-demand $  464,630 = $2,276,198  ◀ MINIMUM
  reserve  96 GPUs total  reserved $1,976,256 + on-demand $  327,974 = $2,304,230
  reserve 128 GPUs total  reserved $2,635,008 + on-demand $        0 = $2,635,008

  all-on-demand baseline (mean served demand 96.0 GPUs)          = $3,279,744/yr
  optimum (88 reserved)                                          = $2,276,198/yr
  ──────────────────────────────────────────────────────────────────────────────
  saving                                       $1,003,546/yr     = 30.6%
  blended rate                                 $2.707/GPU-hr     vs $3.90 all-on-demand
```

The numerical minimum lands exactly where the critical-ratio rule predicted. That agreement is the
point: **you do not have to search, you can compute it.**

**The break-even everyone asks for, and how it relates:** a single reserved GPU beats on-demand
once its **sustained utilisation** exceeds `R/D` — here ≈ **60%**. The newsvendor result is the
*portfolio* version of that same inequality: reserve up to the point where the marginal reserved
GPU is utilised exactly 60% of the time, which is precisely `P(X > Q*) = R/D`.

**Now the third rung, and the link back to lesson 7.** At the optimum, the reserved block is idle
whenever demand is below 88 GPUs:

```
  E[idle reserved GPU-hours/yr] = Σ_x p_x · max(0, 88 − (32 + x)) · 8760
                               = 49,056 GPU-hours/year
```

**Those 49,056 GPU-hours are already paid for; their marginal cost is zero.** Backfilling them with
best-effort work is not a saving on the ladder — it is free compute, worth
`49,056 × $2.35 = $115,282` at the reserved rate, or `$191,318` valued against what that work would
otherwise have cost on demand. This is lesson 7 §12's deliberate-headroom stranding, recovered.
§11's arithmetic then decides how much of it you keep:

```
  harvested capacity          = 49,056 GPU-hours/year (already paid)
  useful work, sync ckpt      = 49,056 × (1 − 0.0812) = 45,073 GPU-hours
  useful work, async ckpt     = 49,056 × (1 − 0.0213) = 48,011 GPU-hours
  ────────────────────────────────────────────────────────────────────
  difference                  =  2,938 GPU-hours/year  ≈ $6,905 at $2.35
```

Modest on this one tier — say so honestly rather than inflating it. The *bigger* effect is §11's
inversion: async checkpointing raises the tolerable preemption rate from ~1.7 to ~10.9 evictions
per day, which is what lets you slice the harvested capacity finely enough to fill 49,056
*scattered* GPU-hours rather than only the long contiguous gaps. **The overhead saving is small;
the schedulability gain is what pays.**

The full ladder, then:

```
  THE COMMITMENT LADDER — 128-GPU fleet, demand distribution from showback
  ══════════════════════════════════════════════════════════════════════════════════════

  GPUs
  128 ┤ ░░░░░░░░░░░░░░░░░░░░░░░░░  spot / opportunistic  ($0.85, when available at all)
      │                             batch sweeps only; evictable without notice
  ────┼───────────────────────────────────────────────────────────────────────────
  112 ┤ ▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒  ON-DEMAND  ($3.90)
      │                             absorbs research spikes above 88
   88 ┤ ═════════════════════════  ◀ Q* = the (1 − R/D) quantile of demand
      │ ███████████████████████████ RESERVED 1-yr  ($2.35), paid whether used or not
      │ ███████████████████████████   ├─ 32 GPUs: prod floor, never preempted
      │ ███████████████████████████   └─ 56 GPUs: research nominal quota
      │ ███████████████████████████
      │ ███░░░░███░░░░░░███░░░░████ ◀ idle reserved GPU-hours = 49,056/yr
    0 ┤ ███████████████████████████    marginal cost ZERO → BACKFILL with best-effort,
      └───────────────────────────      reclaim instantly via Kueue reclaimWithinCohort
        time →

   RULES OF THE LADDER
     · size the reserved rung by the critical ratio, NOT by P50
     · reserved rung is preemption-protected up to each queue's NOMINAL quota
     · on-demand rung is NOT guaranteed available in a shortage — plan for it to fail
     · spot rung carries best-effort work ONLY, and only work that checkpoints
     · every idle GPU-hour on the reserved rung is free compute if you can backfill it
```

### 15. Market-segment discipline, and the price-inversion thesis

**This is the single most important correction to the naive "$X/GPU-hr" interview answer.** GPU
pricing is not one market. It is at least three, and they move independently — sometimes in
opposite directions at the same time.

| Segment | What it is | Rough 2026 snapshot* |
|---|---|---|
| **Hyperscaler retail** | AWS/Azure/GCP on-demand at the public rate card | high — roughly **$7–13/GPU-hr** for H100-class on-demand, depending on provider and instance shape |
| **Specialized neocloud** | CoreWeave-style GPU-native clouds, on-demand | roughly **$2–4/GPU-hr** on-demand; committed/reserved lower still |
| **Spot / preemptible** | evictable capacity at any provider, *when available at all* | as low as **~$0.42–$1/GPU-hr**; broad-market H100 spot fell sharply through 2024–2025 as supply caught demand |

*Snapshot as of **August 2026**. These figures move fast — re-pull a live tracker before quoting
one, and always name the segment when you do.

The spread between segments is **4–6×**, larger than most of the optimisations in this module, so
an unqualified number carries almost no information. Every rate in this lesson — `$2.35` reserved,
`$3.90` on-demand, `$0.85` spot — sits in the **specialized-neocloud** band, and the worked ladder
would produce a materially different `Q*` in the hyperscaler-retail band because `R/D` differs
there.

**The price-inversion thesis.** Through 2024–2026, on-demand and committed pricing in the
**neocloud rental segment specifically** have at times moved in opposite directions: on-demand fell
as new supply came online, while 1-year committed rates *rose* as buyers rushed to lock in
guaranteed capacity ahead of the next scarcity wave. By early-to-mid 2026 trackers describe renewed
tightness, with 1-year committed rental pricing pushing above $2/GPU-hr and on-demand capacity
effectively sold out at several providers.

The durable lesson is not any number — they are all snapshots — but the **structure**: committed and
on-demand pricing can diverge, and **the spread between them is the market's price on scarcity**.
Your ladder is a bet on where that spread goes, and §14 says how to size it: against your own
measured demand distribution and the current `R/D`. Re-run the critical-ratio calculation at every
renewal, because `R/D` moving is exactly what changes `Q*`.

One asymmetry to fold into the decision that pure cost math misses: **the on-demand rung is not
guaranteed to exist.** §13(a) said the capacity can simply be unavailable. The newsvendor model
assumes you can always buy the shortfall at price `D`; when you cannot, the true cost of being
short is not `D` per GPU-hour but the business value of the work that did not run. If that value is
high — a training run on a deadline, a customer-facing service — inflate `D` in the model to reflect
it, which pushes `Q*` up. **Reserving above the naive optimum is how you buy insurance against
supply risk, and framing it that way is how you get it funded.**

## Perspectives

**Developer / training loop.** Checkpoint frequency is decided by the training code owner and paid
for by the platform. A team checkpointing every 30 minutes "because it's slow", unaware that
`torch.distributed.checkpoint`'s async sharded path exists, is capping how much idle fleet headroom
the platform can harvest and does not know it. The fix is not to ask nicely: show the §11
arithmetic and point at the API. The other developer obligation is the SIGTERM handler — a job that
ignores SIGTERM throws away the entire grace-period budget the platform deliberately gave it.

**Operator.** Two knobs the arithmetic turns into money. `terminationGracePeriodSeconds` must be
*just over* the measured flush time — long enough for a clean final save, short enough that a hung
victim does not block reclaim for minutes (§6). And the ladder must be cascade-safe (§8):
`preemptionPolicy: Never` on the middle tier, `reclaimWithinCohort: Any` only on the protected
floor. You also own the honest observation that preemption *capacity* is bounded — §11's inversion
says a 5% overhead budget tolerates ~1.7 evictions/day per job on synchronous checkpointing.
Publish that number; it is why a team's sweep gets killed twice a day and not twenty times.

**Hardware / systems.** Async sharded checkpointing exploits a real hardware asymmetry, and knowing
*why* separates "I heard checkpointing got faster" from an answer that survives a follow-up. The
GPU-blocking part is only the device→host copy, at PCIe/NVLink-to-host bandwidth (tens of GB/s);
the slow durable write to network storage runs on CPU threads, overlapped with the next step, never
touching the GPU's critical path. Sharding then divides state by world size, so `C` *improves* as
the job grows. The cost is host RAM for the staged copy — a real budget line on a node running 8
ranks.

**Economics / market structure.** The price-inversion thesis is a **neocloud GPU-rental**
phenomenon and must be distinguished from **hyperscaler retail**, where absolute pricing is several
times higher and moves on a different cycle. The deeper point is §14's: commitment sizing is a
newsvendor problem whose answer is a quantile of *your own* demand distribution — so the showback
report this module builds is not a reporting nicety, it is the input to a seven-figure decision.

## Real-world use cases

- **Alibaba — "Heterogeneity at Hyperscale" (OSDI '26)** —
  https://www.usenix.org/conference/osdi26/presentation/li-suyi. The strongest single data point
  for this lesson: on a six-month trace of up to **155,410 GPUs across 37,707 servers**, idle GPUs
  frequently became unallocatable — partly through lesson 7's structural mechanisms, partly because
  **users reserve ample headroom for production safety**. The deployed answer was **SpotGPU**, a
  *preemption-cost-aware* harvesting framework, which raised the GPU allocation ratio from
  **68% to 93%**. Note the adjective: a hyperscaler solving this built the §10 cost model into the
  scheduler, because harvesting without pricing the eviction loses money faster.
  *(Search-verified; usenix.org blocked by this environment's egress proxy.)*

- **Google — "Large-scale cluster management at Google with Borg" (Verma et al., EuroSys 2015)** —
  https://research.google.com/pubs/archive/43438.pdf. The priority-band model this lesson's tiers
  descend from — monitoring, production, batch, best-effort/"gratis" — predating Kubernetes by a
  decade, plus the deliberately non-obvious rule that **production-tier tasks do not preempt each
  other**, structurally, even at equal priority, to avoid cascades (§8). Borg also reports the
  payoff: mixing production and non-production work on the same machines, the low tier absorbing
  what the high tier is not using — §14's backfill argument. *(Search-verified;
  research.google.com blocked by the egress proxy.)*

- **`kubernetes/kubernetes` preemption implementation, v1.37 development head.** Read from source
  rather than docs: the remove-all-then-reprieve minimisation in `SelectVictimsOnNode`; the five
  ordered node tiebreaks in `pickOneNodeForPreemption`; the group-aware `MoreImportantVictim`
  ordering; the `DisruptionTarget` / `PreemptionByScheduler` condition; the bare
  `metav1.DeleteOptions{}` that leaves the victim's own `terminationGracePeriodSeconds` in force;
  and `PodEligibleToPreemptOthers`' anti-cascade guard. **Cloned and read directly this session** —
  kubernetes.io was unreachable from this environment.

- **PyTorch — Distributed Checkpoint (DCP), and its production deployments** —
  https://docs.pytorch.org/docs/stable/distributed.checkpoint.html plus the PyTorch blog posts on
  efficient and performant distributed checkpointing. The `save`/`async_save`/`load` API, per-rank
  sharded parallel saves, save-plan caching, and a named production deployment (IBM) running it at
  scale — the mechanism that turns "checkpoint more often" from a request into an affordable
  default, and therefore what makes the §11 optimum move by 9×. *(Search-verified; pytorch.org
  fetches blocked by the egress proxy.)*

- **OpenAI — "Scaling Kubernetes to 7,500 Nodes"** —
  https://openai.com/index/scaling-kubernetes-to-7500-nodes/. What it shows, reused here from a
  capacity-economics angle rather than the gang-scheduling angle of lessons 1–2: **team taints**
  plus priority-weighted **"balloon" deployments** — low-priority placeholder pods that hold
  capacity and are evicted the moment a real workload arrives. That is the soft-reclaim pattern
  §14's backfill rung generalises, running in production at 7,500-node scale.
  *(Search-verified; fetch blocked by egress this session.)*

- **SemiAnalysis — "The Great GPU Shortage: Rental Capacity"** —
  https://newsletter.semianalysis.com/p/the-great-gpu-shortage-rental-capacity. The structure of
  the neocloud rental market and the scarcity dynamics behind §15's committed-versus-on-demand
  divergence. Read it for the argument's *structure* — the spread is the market's price on scarcity
  — not the numbers, which move weekly. *(Search-verified; fetch blocked by egress.)*

## Worked example

The full 128-GPU design: three research teams plus one production service. This is the artefact
the checkpoint asks you to defend, built end to end.

**Step 1 — the tier ladder.** The three pod PriorityClasses from §2, mirrored by three Kueue
`WorkloadPriorityClass` objects (§3) so both layers agree on who is expendable. Same `apiVersion:
kueue.x-k8s.io/v1beta2`, `kind: WorkloadPriorityClass`, one `value` field each:

| WorkloadPriorityClass | `value` | mirrors | meaning |
|---|---|---|---|
| `wpc-prod` | 900000 | `gpu-prod` | never evicted from quota |
| `wpc-research` | 500000 | `gpu-research` | reclaimable when borrowing above nominal quota |
| `wpc-best-effort` | -10 | `gpu-best-effort` | first evicted, always; admission requires a documented checkpoint/resume path |

**Step 2 — the quota split**, sized from §14's ladder (88 reserved: 32 prod + 56 research):

| ClusterQueue | `nominalQuota` | `borrowingLimit` | `lendingLimit` | Rationale |
|---|---|---|---|---|
| `cq-prod` | 32 | 0 | 0 | firm floor, never borrows, never lends — protected by construction |
| `cq-research-a` | 20 | 24 | 12 | may burst to 44 by borrowing; lends up to 12 when idle |
| `cq-research-b` | 20 | 24 | 12 | symmetric |
| `cq-research-c` | 16 | 24 | 10 | smaller team, same burst ceiling |
| `cq-best-effort` | 0 | 128 | 0 | owns nothing; runs entirely on borrowed idle capacity |

All five in one cohort. `cq-best-effort` with `nominalQuota: 0` is the load-bearing trick: it has no
protected floor at all, so **every GPU it holds is borrowed and therefore reclaimable by anyone**,
which is exactly the property that makes idle-headroom harvesting safe.

**Step 3 — the preemption policy, and its defence.**

```yaml
# cq-prod — the floor
preemption:
  reclaimWithinCohort: Any            # take back the 32 from ANY borrower, regardless
                                      # of the borrower's priority. This is what makes
                                      # a floor a floor.
  withinClusterQueue: LowerPriority
  borrowWithinCohort: {policy: Never}
---
# cq-research-* — burst without starting cascades
preemption:
  reclaimWithinCohort: LowerPriority  # research may reclaim from best-effort only
  withinClusterQueue: LowerOrNewerEqualPriority   # within a team, newer work yields
  borrowWithinCohort: {policy: Never} # borrowing must never cost another team
---
# cq-best-effort — never preempts anything
preemption:
  reclaimWithinCohort: Never
  withinClusterQueue: Never
```

**Defence, in the form an interviewer wants.** The ladder is cascade-safe: every preemption moves
work strictly down a tier (prod → research or best-effort; research → best-effort only;
best-effort → nothing), so the chain is bounded by three and terminates at a tier whose workloads
all checkpoint. `borrowWithinCohort: Never` everywhere means no team can take capacity from a peer
merely to burst above its own quota — borrowing is only ever from *genuinely idle* capacity, which
keeps the fairness story simple and stays compatible with Fair Sharing.
`withinClusterQueue: LowerOrNewerEqualPriority` implements "within your own team, the newest job
yields", matching `kube-scheduler`'s latest-start-time tiebreak and protecting accumulated work.

**Step 4 — the survivability contract.** For the best-effort tier, admission requires:

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 120     # measured: async flush + final sync save
      priorityClassName: gpu-best-effort
      containers:
        - name: trainer
          # the training loop must:
          #   1. call dcp.async_save(...) every T* seconds  (see step 5 for T*)
          #   2. install a SIGTERM handler that waits for the in-flight async save
          #      AND takes a final synchronous one, inside the grace period
          #   3. on start, dcp.load(...) from the latest checkpoint and resume the
          #      step counter, LR schedule, dataloader position and RNG state
```

Point 3's list is what people get wrong: resuming weights but not the optimizer state, LR schedule
or data position is not a resume — it is a subtly corrupted restart that costs more than it saves.

**Step 5 — size the checkpoint interval.** Measure `C` for a representative best-effort job, count
evictions per day from the Kueue eviction events, then apply §11:

```bash
# preemption rate, from the last 7 days of Kueue evictions
$ kubectl get events -A --field-selector reason=EvictedByPreemption \
    -o json | jq '[.items[] | select(.lastTimestamp > (now-604800|todate))] | length'
28

# 28 evictions / 7 days / 20 concurrently-running best-effort jobs
#   λ = 28 / (7 × 20) = 0.2 evictions per job per day = 2.31e-6 /s   … per job
# but the FLEET-level rate an individual job sees is what matters; measure per job:
#   here, 4 evictions/job/day → λ = 4.63e-5 /s, MTBP = 6 h
```

```
  measured C (async sharded DCP, 7B model, 8 ranks)     = 0.6 s
  measured R (restart: requeue + load + init + warmup)  = 300 s
  λ = 4 evictions/day                                   = 4.63e-5 /s

  T* = √(2C/λ) = √(2 × 0.6 / 4.63e-5) = 161 s ≈ 2.7 min   → set T_ckpt = 3 min

  overhead at T*:  ckpt 0.37% + rework 0.37% + restart 1.39%  =  2.13%
  (had the team stayed on synchronous saves: T* = 24.2 min, overhead 8.12%)
```

Sanity-check with the equal-cost property: at 3 minutes the checkpointing and rework terms should
be roughly equal. They are (0.33% and 0.42% at T = 180 s). If they were not, the interval is wrong.

**Step 6 — the ladder, priced.** From §14, with the demand distribution taken from the showback
report:

```
  critical ratio R/D          = 2.35 / 3.90            = 0.6026
  Q* (research)               = 56 GPUs                 (largest Q with P(X>Q) ≤ 0.6026)
  Q* (total reserved)         = 32 + 56                 = 88 GPUs
  on-demand headroom          = up to 40 more           (fleet ceiling 128)
  spot / backfill             = opportunistic, best-effort tier only

  annual cost at optimum                                = $2,276,198
  all-on-demand baseline                                = $3,279,744
  saving                                                = $1,003,546  (30.6%)
  blended rate                                          = $2.707/GPU-hr
  break-even utilisation for the reserved rung          = R/D = 60.3%
  idle reserved GPU-hours/yr (free compute if backfilled)= 49,056
```

**Step 7 — the showback line that closes the loop.** The per-ClusterQueue report from the
deliverable now carries a column that did not exist before:

| Queue | Reserved quota | Actual usage | Borrowed | $ owed | Idle-quota cost |
|---|---|---|---|---|---|
| `cq-prod` | 32 | 30.1 | 0 | $619,000 | $39,000 |
| `cq-research-a` | 20 | 17.4 | 3.2 | $424,000 | $53,500 |
| `cq-research-b` | 20 | 18.9 | 1.1 | $412,000 | $22,600 |
| `cq-research-c` | 16 | 11.2 | 0.4 | $239,000 | $98,900 |
| `cq-best-effort` | 0 | 5.6 | 5.6 | $115,000 | — |

*(Illustrative figures at the $2.35 reserved rate; the deliverable generates the real ones.)* Read
the last two rows together and the whole module is visible in one table: `cq-research-c` is paying
$98,900/yr for quota it does not use, and `cq-best-effort` — which owns nothing — recovered
$115,000 of that by running in the gaps. **The idle-quota column is the signal that tells you
whether next year's commitment should grow or shrink, and it is only meaningful because the
backfill tier makes the idle capacity productive rather than merely visible.**

## Practice

Three artifacts, all feeding the deliverable, all runnable on the kind cluster with fake
`nvidia.com/gpu` resources.

1. **The tier ladder, applied and proven.** Create the three `PriorityClass`es from §2 and the
   three `WorkloadPriorityClass`es from the Worked example, then prove `preemptionPolicy: Never`
   does what §2 claims. Fill the cluster with `gpu-best-effort` pods. Submit a `gpu-research` pod
   (`Never`) and confirm it **pends** rather than evicting anything, with no preemption attempt in
   `kubectl describe pod`. Then submit a `gpu-prod` pod (`PreemptLowerPriority`), confirm it **does**
   evict, and capture the victim's condition — you want `"reason": "PreemptionByScheduler"` and the
   message naming the scheduler:
   ```bash
   kubectl get pod <victim> -o jsonpath='{.status.conditions[?(@.type=="DisruptionTarget")]}' | jq
   ```
   Also capture the preemptor's `status.nominatedNodeName` in the window before it binds.

2. **Survivable preemption demo.** Configure `cq-best-effort` (nominalQuota 0) below `cq-prod` in
   one cohort with `reclaimWithinCohort: Any` on prod. Run a fake "training" pod that writes a
   step counter to a mounted volume every N seconds and, on start, resumes from it. Then:
   - Submit prod work that forces reclaim. Confirm the best-effort Workload is evicted and, when
     re-admitted, **resumes from its checkpoint** rather than step 0. Capture the Kueue eviction
     event and the resumed step number from the pod log.
   - **Measure the loss, do not assume it.** Vary N over {10 s, 30 s, 120 s} and record the wasted
     step count per eviction across ~10 evictions each. Plot mean wasted work against N and check
     it against the `T_ckpt/2` prediction. It will not match exactly — real evictions are not
     perfectly uniform — and understanding *why* your data deviates is the point of the exercise.
   - **Then test the grace period.** Add a SIGTERM handler that writes a final checkpoint, set
     `terminationGracePeriodSeconds: 60`, and re-run. Wasted work should collapse toward zero.
     Then set it to `1` and watch it come back. This pair of runs is the most persuasive artifact
     in the whole practice.

3. **Commitment-ladder model.** Write `capacity/commitment_ladder.py`. Inputs: a demand
   distribution (from the showback report, or §14's table to start), a firm baseline, and
   reserved/on-demand/spot rates **labelled by market segment**. Outputs: the critical ratio `R/D`
   and the resulting `Q*` as the `(1 − R/D)` quantile; a brute-force cost table over candidate `Q`
   values that *confirms* the analytic answer (do both — agreement is what makes the model
   trustworthy); the blended $/GPU-hr, break-even utilisation and saving versus all-on-demand;
   **expected idle reserved GPU-hours per year**, which is the harvestable capacity; and a
   sensitivity run over `R/D ∈ {0.35, 0.50, 0.60, 0.75}` showing how `Q*` moves — the answer to
   "what if we sign a 3-year instead?" Apply it to the 128-GPU fleet and put the result in the
   capstone doc.

**Acceptance:**
- The `gpu-research` pod demonstrably pends without preempting while `gpu-prod` demonstrably
  evicts — with the `DisruptionTarget` condition captured for the victim.
- The preempted best-effort Workload **resumes from checkpoint** (log shows resumed step > 0), with
  the Kueue eviction event captured, plus the wasted-work-versus-`T_ckpt` measurements and the
  grace-period before/after pair.
- The ladder model outputs `Q*`, blended rate, break-even utilisation, idle-reserved GPU-hours and
  an `R/D` sensitivity table, applied to the 128-GPU inventory, with **every $/hr flagged as a
  dated snapshot and labelled by market segment**.

## Common pitfalls

- **Quoting a single "$/GPU-hr" without naming the market segment.** Hyperscaler retail,
  specialized neocloud, and spot are three markets with 4–6× spreads. An unqualified number is not
  a precision error, it is a category error — and it changes `R/D`, which changes `Q*`, which
  changes what you buy. Always say which segment and which month.

- **"Reserve to your P50."** P50 is correct only when `R/D = 0.5`. The rule is
  `P(X > Q*) = R/D`, so at `R/D = 0.60` you reserve to roughly P40 and at `R/D = 0.35` to roughly
  P65. Mechanism: the marginal reserved GPU costs `R` always and saves `D` only when demand exceeds
  `Q`, so you keep buying until those balance.

- **Confusing high priority with preemption power.** `preemptionPolicy: Never` on a high-value
  class gives queue priority *without* eviction rights — the pod sorts first in `activeQ` and then
  waits like everyone else. Forgetting the field means a tier you intended as "important but
  patient" starts evicting things and can seed a cascade. Conversely, assuming a low-priority pod
  is safe because "we don't use preemption" ignores that `PreemptLowerPriority` is the **default**.

- **Assuming the DELETE frees the GPU.** The scheduler deletes victims with bare
  `metav1.DeleteOptions{}`, so the victim keeps its own `terminationGracePeriodSeconds` (default
  30 s). Capacity is real only at container exit. A capacity plan that assumes instant release is
  wrong by that interval per event, and a preemptor blocked behind a hung victim cannot preempt
  elsewhere either — Step 0's guard sees the terminating pod on its nominated node.

- **Setting `terminationGracePeriodSeconds` without measuring the flush time.** Too short and every
  preemption costs the full `T_ckpt/2` because the final save never lands; too long and reclaim
  stalls. Measure the async-flush-plus-final-sync time for your model size and set it just above.

- **Copying Borg's "production never preempts production" without the reason.** It exists to stop
  cascades, where evicting one same-tier task forces a chain of further evictions. State the rule
  *and* the mechanism, then say how you get it in Kubernetes: strictly-lower-priority victims,
  `preemptionPolicy: Never` on middle tiers, and a ladder where every preemption moves work
  strictly down a tier.

- **Pricing a gang preemption as one pod.** Evicting one pod from a healthy 8-pod gang does not
  free 1 GPU — it strands 7, because the survivors block in the collective barrier. Real cost is
  `m_gang × (loss + T_restart + T_requeue)`. This is the argument for whole-workload eviction
  granularity (Kueue's default, Volcano's bundles, KAI's whole-workload preempt), and it is why
  pod-granular preemption against gangs is a correctness bug, not an inefficiency.

- **Treating `T_ckpt` as fixed.** With async sharded checkpointing only the device→host copy blocks
  the GPU, so `C` falls by roughly two orders of magnitude, `T* = √(2C/λ)` falls by ~9×, and the
  tolerable preemption rate at a fixed overhead budget rises ~6×. A team still on synchronous saves
  is capping how much of the fleet you can harvest, and usually does not know it.

- **Believing "just autoscale the GPU cluster."** Supply is not elastic: in a shortage on-demand
  capacity is out of stock and Cluster Autoscaler backs off on capacity errors; cold start is 5–15
  minutes; and the unit is a lumpy ~$19/hr node. You commit ahead of demand and schedule within it.

## Self-check

- **Why is preemption economically useless without checkpointing?** *Answer:* Preemption's value is
  reclaiming capacity you already paid for and giving it to work that needs it more; its cost is
  the work destroyed in the victim. With checkpointing at interval `T_ckpt`, an eviction landing
  uniformly in the cycle destroys `T_ckpt/2` in expectation, so
  `E[cost] = m × (T_ckpt/2 + T_restart)` GPU-hours — bounded, small, tunable. Without it, `loss` is
  the job's **entire elapsed runtime**: a 40-hour 8-GPU job preempted at hour 39 costs 312
  GPU-hours ≈ $733 at a $2.35/GPU-hr neocloud snapshot, typically more than the reclaim bought. So
  preemption is a cost optimisation only when the workload resumes — a joint scheduler-and-
  training-loop property, not a scheduler knob, which is why the best-effort tier's admission
  criterion is a documented resume path.

- **Walk the victim-selection algorithm.** *Answer:* It runs in `PostFilter`, after the pod fails
  `Filter` everywhere. **(0)** `PodEligibleToPreemptOthers`: reject if `preemptionPolicy: Never`, or
  if the pod already has a nominated node with victims still terminating from its previous
  preemption — the anti-cascade guard. **(1)** Keep only nodes whose failure was `Unschedulable`,
  not `UnschedulableAndUnresolvable`. **(2)** Dry-run a *sample*:
  `minCandidateNodesPercentage: 10` with a floor of `minCandidateNodesAbsolute: 100` nodes, so
  preemption is approximate at scale. **(3)** Per node: take all pods with **strictly lower**
  priority; remove them all and re-run `Filter` — if it still does not fit, the node is out; sort by
  `MoreImportantVictim` descending; split into PDB-violating and non-violating; then **reprieve** —
  add pods back one at a time, most important first, violating group first, keeping each one the
  preemptor can still fit around. Survivors are reprieved; the rest are the minimal victim set.
  **(4)** Across nodes, five ordered tiebreaks: fewest PDB violations → lowest highest-priority
  victim → smallest priority sum → fewest victims → latest earliest-start-time (destroy the
  youngest work) → first node. **(5)** Execute: patch `DisruptionTarget`/`PreemptionByScheduler`,
  delete with bare `DeleteOptions{}`, set `nominatedNodeName`, requeue the preemptor.

- **What ordering does `MoreImportantVictim` use, and what does it protect?** *Answer:* Five rules
  in sequence. (1) **Priority** — higher is more important, and it dominates. (2) **Workload type**
  — `CompositePodGroup` > `PodGroup` > individual `Pod`, to preserve group integrity: at equal
  priority the scheduler would rather kill a lone pod than break a gang. (3) For two individual
  pods, **earlier start time** (longer runtime) wins — FCFS, protecting accumulated work. (4) For
  two groups of the same type, **more members** wins, because rescheduling a massive job is
  expensive. (5) Tie-break on the group with the oldest pod. Rules 2 and 4 encode gang
  amplification; 3 and 5 encode "older work has more to lose", the same instinct as node tiebreak 5.

- **Where does the victim's grace period come from, and why does it matter economically?**
  *Answer:* From the victim. After patching `DisruptionTarget`, the scheduler calls
  `Pods(ns).Delete(ctx, name, metav1.DeleteOptions{})` — **no `GracePeriodSeconds` override** — so
  the pod's own `terminationGracePeriodSeconds` applies, defaulting to 30 s. Economically it is the
  budget for a checkpoint-on-SIGTERM: long enough and the loss window collapses toward zero, too
  short and every preemption costs the full `T_ckpt/2`. It is also the interval during which the
  GPU is *not yet free* — the DELETE is a promise, capacity is real at container exit — and during
  which the preemptor cannot preempt elsewhere, because Step 0's guard sees the terminating victim
  on its nominated node. Set it just above your measured flush time, and measure it.

- **Derive the cost-minimising checkpoint interval and give the field diagnostic.** *Answer:*
  Overhead per unit wall-clock is `Ω(T) = C/T + λT/2 + λR`, with `C` the GPU-blocking cost of one
  checkpoint, `T` the interval, `λ` the preemption rate, `R` the restart overhead. `C/T` is
  checkpointing; `λT/2` is rework, since an eviction landing uniformly in the cycle destroys `T/2`
  on average; `λR` is independent of `T`. Setting `dΩ/dT = −C/T² + λ/2 = 0` gives
  **`T* = √(2C/λ) = √(2·C·MTBP)`** — Young/Daly, with preemption rate substituted for failure rate.
  Substituting back, both variable terms equal `√(Cλ/2)`: **at the optimum you spend exactly as
  much time checkpointing as you lose to rework**, which is the diagnostic — if they are far apart
  your interval is wrong, in the direction of the larger term. Minimum overhead is `√(2Cλ) + λR`,
  growing only as the *square root* of `C` and `λ`. At `λ = 4/day`, `R = 300 s`: synchronous
  `C = 49 s` → `T* = 24.2 min`, 8.12% overhead; async sharded `C = 0.6 s` → `T* = 2.7 min`, 2.13%.
  The curve is asymmetric — too-frequent is much worse than too-rare — so err long of `T*`.

- **How does async sharded checkpointing change the economics, and what is the *biggest* effect?**
  *Answer:* It shrinks `C`, not the formula. Each rank saves only its own shard in parallel, so `C`
  scales with `state/world_size` instead of total state; and with `async_save` only the
  device→host copy blocks the GPU while the durable network write overlaps the next steps on CPU
  threads. For a 7B model over 8 ranks that is roughly `C: 49 s → 0.6 s`, which through
  `T* = √(2C/λ)` shortens the optimal interval 9× and through `Ω(T*) = √(2Cλ) + λR` cuts overhead
  from 8.12% to 2.13%. **The bigger effect is the inversion:** at a fixed 5% overhead budget the
  tolerable preemption rate rises from ~1.71 to ~10.86 evictions/day, about 6× — which determines
  how finely you can slice a backfill tier and therefore how much idle reserved headroom you can
  harvest. Costs to name: host RAM for the staged copy, and a SIGTERM handler that must wait for
  the in-flight async save *and* take a final synchronous one inside the grace period.

- **At what sustained utilisation does a 1-year reserve beat on-demand, and how do you size the
  ladder?** *Answer:* A single reserved GPU beats on-demand once its sustained utilisation exceeds
  `R/D` — with $2.35 committed against $3.90 on-demand (specialized-neocloud snapshot), ≈ **60%**.
  The portfolio version is a newsvendor problem: `E[cost(Q)] = Q·R + E[max(0, X−Q)]·D`, and
  `dE/dQ = R − D·P(X>Q) = 0` gives **`P(X > Q*) = R/D`** — reserve to the `(1 − R/D)` quantile of
  *your own* demand distribution, **not P50**, unless `R/D` happens to be 0.5. Worked on the
  128-GPU fleet with 32 GPUs firm prod demand: `R/D = 0.6026` → `Q* = 56` research GPUs, 88 total
  reserved, $2.28M/yr against a $3.28M all-on-demand baseline — a **$1.00M (30.6%) saving** at a
  blended **$2.707/GPU-hr**. Two adjustments the pure math misses: the on-demand rung may not
  *exist* in a shortage, so inflate `D` by the business value of work that would not run, pushing
  `Q*` up; and the optimum leaves ~49,056 idle reserved GPU-hours a year at zero marginal cost —
  free compute if you can backfill it.

- **Why does Borg forbid same-tier production preemption, and how do you get that on Kubernetes?**
  *Answer:* To prevent cascades: A evicts B, B must be re-placed and evicts C, C evicts D — one
  legitimate reclaim becomes a chain reaction that destabilises the tier, and every link destroys
  work. Borg's fix is structural, not a tiebreak. On Kubernetes you get the same property three
  ways: victims must be **strictly lower** priority, so preemption within one PriorityClass is
  impossible by construction; `PodEligibleToPreemptOthers` refuses to let a pod preempt again while
  victims from its previous preemption are still terminating, killing the rapid cascade; and
  `preemptionPolicy: Never` stops a tier initiating a chain at all. Design the ladder so **every
  preemption moves work strictly down a tier**, and the chain is bounded by the tier count.

## Connections & what's next

This lesson closes the loop lesson 1 opened. The default scheduler's atomicity gap (01–02) is fixed
by gang admission; Kueue (03–04) turns admission into a queueable, quota-bearing, cohort-sharing
system; alternatives (05) and topology (06) refine how and where work is placed; fragmentation (07)
measures what is left over and prices it; and this lesson decides who may reclaim it, what that
costs, and how much of the fleet you should have bought. Three threads tie back explicitly: the
`T_ckpt/2` term first appeared in lesson 7's defrag payback and is optimised here; the
deliberate-headroom stranding lesson 7 could not fix with placement is fixed here with a
`nominalQuota: 0` backfill queue; and lesson 6's hot swap is one more instance of the same
survivability contract. The "commit versus rent" output feeds **Module 11 (GPU cost economics)**,
which consumes the commitment ladder, the critical-ratio sizing, the break-even utilisation and the
per-queue showback report as its raw data.

There is no next lesson in this module — the [**checkpoint**](../checkpoint.md) is next. Use it to
prove, unaided, that you can reproduce the deadlock and its fix, explain Kueue cold, compute
effective capacity, design the 128-GPU queues, build the commitment mix, and produce the showback
report. That checkpoint, plus the
[Kueue setup + per-queue showback](../practice/kueue-showback/README.md) deliverable, is the gate
for the whole module.

## References & further reading

**Primary sources — read directly from cloned repositories this session**

Note on method: this environment's egress proxy blocks `kubernetes.io`, `usenix.org`,
`pytorch.org`, `openai.com`, `research.google.com` and several other domains. Rather than cite
pages that could not be reached, the mechanism and default-value claims were verified against
upstream *source trees* cloned during this session; where a canonical URL is given for
convenience, its reachability is stated honestly.

1. **`kubernetes/kubernetes` — `pkg/apis/scheduling/types.go`, `pkg/apis/scheduling/v1/helpers.go`**
   — https://github.com/kubernetes/kubernetes. §2's value structure:
   `HighestUserDefinablePriority = 1,000,000,000`, `SystemCriticalPriority = 2×` that, the two
   auto-created classes (`system-cluster-critical` **2,000,000,000**, `system-node-critical`
   **2,000,001,000**), the `PreemptionPolicy` union, and the `globalDefault` "smallest wins" note.
   **Cloned and read directly this session.**

2. **`kubernetes/kubernetes` — `pkg/scheduler/framework/plugins/defaultpreemption/default_preemption.go`.**
   `SelectVictimsOnNode`'s remove-all-then-reprieve structure, the strictly-lower-priority
   eligibility test in `isPreemptionAllowed`, the PDB-violating-first reprieve ordering, and
   `PodEligibleToPreemptOthers`' two rejection reasons (`preemptionPolicy: Never`; a terminating
   preemption victim on the pod's nominated node). **Cloned and read directly this session.**

3. **`kubernetes/kubernetes` — `pkg/scheduler/framework/preemption/{preemption.go,util.go}`.**
   `pickOneNodeForPreemption`'s five ordered score functions, and `MoreImportantVictim`'s five-rule
   ordering with its `CompositePodGroup(3) > PodGroup(2) > Pod(1)` type ranking and the code's own
   rationale ("preserve group integrity", "avoid the high cost of rescheduling massive jobs").
   **Cloned and read directly this session.**

4. **`kubernetes/kubernetes` — `pkg/scheduler/framework/preemption/executor.go`,
   `pkg/scheduler/util/utils.go`.** The execution path: in-memory fast paths for waiting and
   pre-binding victims, the `DisruptionTarget` / `PodReasonPreemptionByScheduler` condition and its
   message format, and `util.DeletePod`'s bare `metav1.DeleteOptions{}` — the source of §6's claim
   that the victim keeps its own `terminationGracePeriodSeconds`. **Cloned and read directly this
   session.**

5. **`kubernetes/kubernetes` — `pkg/scheduler/apis/config/v1/defaults.go`.**
   `SetDefaults_DefaultPreemptionArgs`: `minCandidateNodesPercentage = 10`,
   `minCandidateNodesAbsolute = 100` — the sampling that makes preemption approximate on large
   fleets. **Cloned and read directly this session.**

6. **`kubernetes-sigs/kueue` — `apis/kueue/v1beta2/{clusterqueue_types.go,workloadpriorityclass_types.go}`**
   — https://github.com/kubernetes-sigs/kueue. The `ClusterQueuePreemption` struct used in the
   Worked example — `reclaimWithinCohort` (`Never`|`LowerPriority`|`Any`), `withinClusterQueue`
   (`Never`|`LowerPriority`|`LowerOrNewerEqualPriority`), `borrowWithinCohort` with its
   `policy`/`maxPriorityThreshold` and its Classic-Preemption-only constraint — plus
   `WorkloadPriorityClass` and the note that changing its value does not affect already-created
   Workloads. **Cloned and read directly this session.**

7. **`kubernetes-sigs/kueue` — `site/content/en/docs/concepts/preemption.md`.** Kueue's candidate
   ordering (borrowing queues first → lowest priority → most recently admitted), the four target-
   qualification heuristics, the reverse-traversal target minimisation, and the `Evicted`/`Preempted`
   condition pair written onto the victim Workload. **Read from the cloned repo this session;**
   https://kueue.sigs.k8s.io/docs/concepts/preemption/ was unreachable from this environment.

**Research and foundational systems**

8. **Verma, Pedrosa, Korupolu, Oppenheimer, Tune, Wilkes — "Large-scale cluster management at
   Google with Borg" (EuroSys 2015)** — https://research.google.com/pubs/archive/43438.pdf. The
   priority-band model (monitoring / production / batch / best-effort) this lesson's tiers descend
   from, the deliberate rule that production tasks do not preempt each other, the cascade-avoidance
   reasoning behind it, and the production-plus-batch mixing that is §14's backfill argument.
   *(Search-verified; fetch blocked by this environment's egress proxy.)*

9. **Young (1974) and Daly (2006) — the optimal checkpoint interval.** `T* = √(2·C·MTBF)` and its
   higher-order refinements, derived for supercomputer failure but structurally identical to the
   preemption problem in §11 once `MTBF` is replaced by mean time between preemptions. The
   derivation in §11 is done from first principles rather than cited, so you can re-derive it under
   different assumptions (non-uniform eviction timing, checkpoint cost varying with state size).

10. **Alibaba — "Heterogeneity at Hyperscale: Characterization and Scheduling of Large Production
    AI Clusters at Alibaba" (OSDI '26)** —
    https://www.usenix.org/conference/osdi26/presentation/li-suyi. Six-month trace, up to
    **155,410 GPUs / 37,707 servers**; user-reserved headroom as a first-class cause of
    unallocatable capacity; and **SpotGPU**, a preemption-cost-aware harvesting framework that
    raised the GPU allocation ratio from **68% to 93%**. *(Search-verified; usenix.org blocked by
    egress.)*

**Engineering accounts and tooling**

11. **PyTorch — `torch.distributed.checkpoint`** —
    https://docs.pytorch.org/docs/stable/distributed.checkpoint.html, plus
    https://pytorch.org/blog/distributed-checkpoint-efficient-checkpointing-in-large-scale-jobs/
    and https://pytorch.org/blog/performant-distributed-checkpointing/. The
    `save`/`async_save`/`load` API, sharded parallel saves, save-plan caching, and a named
    production deployment (IBM) — the mechanism behind §12's `C: 49 s → 0.6 s`.
    *(Search-verified; fetches blocked by egress.)*

12. **Market references — SemiAnalysis, "The Great GPU Shortage: Rental Capacity"**
    (https://newsletter.semianalysis.com/p/the-great-gpu-shortage-rental-capacity) **and a live
    cross-provider tracker such as getdeploying.com's H100 page**
    (https://getdeploying.com/gpus/nvidia-h100). The first for §15's market structure; the second to
    pull a fresh, segment-labelled snapshot the day you build your ladder. Read the first for
    structure, never for numbers. *(Both search-verified; fetches blocked by egress.)*
