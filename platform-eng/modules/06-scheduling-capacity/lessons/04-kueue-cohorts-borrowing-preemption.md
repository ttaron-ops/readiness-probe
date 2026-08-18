---
lesson: "06.4"
title: "Kueue II — cohorts: borrowing, lending, preemption, and fair sharing"
module: "06"
concept: "Kueue cohorts: borrowing, lending, preemption, and fair sharing"
status: not-started
est_time: "13h"
prev: "03-kueue-queueing-model.md"
next: "05-alternatives-volcano-kai.md"
artifacts: []
sources: 15
---
# 06.4 · Kueue II — cohorts: borrowing, lending, preemption, and fair sharing

> **Concept.** A cohort lets queues lend idle floors to each other and reclaim them by preemption — utilisation without giving up guarantees.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

[L3](03-kueue-queueing-model.md) built the queueing model and gave every team a hard, isolated floor: `nominalQuota` on a ClusterQueue with no `cohortName`, which is simultaneously a guaranteed minimum and an absolute ceiling. That is safe and legible, and it is wasteful the instant one team is idle while another queues. L3's own §11 arithmetic showed why: queue wait time explodes as utilisation approaches capacity, and a fleet partitioned into isolated floors runs every partition at its own utilisation rather than the fleet's.

This lesson removes the waste without removing the guarantee. It groups ClusterQueues into a **cohort**, lets a busy queue **borrow** an idle sibling's unused floor, and lets the owner **reclaim** it by preemption the moment it needs it back. It adds the fairness math — dominant resource share, weights, preemption strategies — that decides *who* gets evicted when the fleet is tight, and it distinguishes that from L3's `AdmissionFairSharing`, which is a different layer.

It also cashes in the mechanism from [L2](02-gang-scheduling.md). Preemption in Kueue is not a new primitive: it re-suspends the victim's Job, which deletes its pods and releases its GPUs, and puts the Workload back in the queue. Everything you know about `spec.suspend` from L3 §2 applies in reverse.

This is the deepest, most interview-tested Kueue surface in the module. After it you have the full anchor and can take [L5](05-alternatives-volcano-kai.md)'s scheduler comparison with real footing.

## Why this matters

If research owns 8 A100s and is idle over a weekend, those 8 GPUs sit dark while product's queue backs up. On a fixed fleet, idle *guaranteed* capacity is the purest form of waste there is: you paid for it, nobody is using it, and the reason nobody is using it is a policy you wrote.

Put a number on it. A 128-GPU fleet split into four isolated 32-GPU floors, where each team averages 60% utilisation of its own floor with bursty demand:

```
    Isolated floors:   4 × 32 GPUs × 60%  =  76.8 GPU-hours of work per hour
                       51.2 GPU-hours/hour idle, unreachable by anyone else

    At ~$2.50/GPU-hour (a 2026 on-demand snapshot — recompute for your contract):
      51.2 × $2.50 × 730 h/month  ≈  $93,400/month of capacity paid for and idled

    Pool the same 128 GPUs in one cohort with borrowing. Aggregate demand is
    unchanged, but peaks no longer coincide, so the fleet runs at (say) 85%:
      128 × 0.85 = 108.8 GPU-hours of work per hour, +42% throughput,
      from a YAML change and zero new hardware.
```

That is the entire FinOps thesis of the module in one calculation, and it is the argument that gets a platform team a seat at the capacity-planning table. The counter-argument — "but then my guaranteed capacity isn't guaranteed" — is what `reclaimWithinCohort` and `lendingLimit` exist to answer, and answering it precisely is the point of this lesson.

The interview surface is unusually concrete. "Explain `borrowingLimit` versus `lendingLimit`." "When does fair-sharing preemption pick a different victim than classic priority preemption?" "Why can't `borrowWithinCohort` run under fair sharing?" "Design queues for three research teams and one prod service on a fixed 128-GPU fleet, and defend the borrowing and preemption choices." These are real screens at GPU-heavy shops — CoreWeave's own Principal/Staff Cluster Orchestration description names "quota enforcement, fairness, pre-emption, and multi-tenant GPU isolation" as core to the role. Master the mechanics *and* the cost reasoning behind each knob; interviewers probe both.

## What's new here (calibration)

- **You have L3's object model**: ClusterQueue, LocalQueue, ResourceFlavor, Workload, `nominalQuota`, the two-phase admission cycle, the status conditions. None of that is re-taught. What changes is that `nominalQuota` stops being a ceiling and becomes **a floor you can exceed** (by borrowing) and **a floor you may lend**.
- **You have L3's API-version correction**: everything here is `kueue.x-k8s.io/v1beta2`, the storage version since Kueue v0.16. Note especially `ClusterQueue.spec.cohortName`, renamed from `spec.cohort`.
- **You have [L2](02-gang-scheduling.md)'s suspend mechanics**; eviction reuses them.

Genuinely new:

- **The `Cohort` object as a first-class API** with its own quota and a parent pointer, not just a string that ClusterQueues happen to share. Cohort-level `nominalQuota` is *additional* shared capacity on top of the members' own, which is a distinct design tool most write-ups skip.
- **The classic preemption algorithm as an ordered, readable procedure** — candidate compilation, a five-key candidate sort straight out of `CandidatesOrdering`, four target heuristics, and a fill-back minimisation pass.
- **The dominant-resource-share value function with its actual formula**, including the ×1000 scaling and the `ceil`, so `status.fairSharing.weightedShare` is a number you can predict rather than read.
- **The classic-versus-fair victim inversion**, worked with numbers rather than asserted — and the upstream proof that fair-sharing preemption cannot loop.
- **Hierarchical cohorts (`CohortTree`)** and the lowest-common-ancestor reasoning that decides whether a preemption is a *reclaim* (S1) or a *rebalance* (S2).

## Core concepts

### 1. The cohort: what it is, and the two ways to declare it

A **cohort** is a set of ClusterQueues that share their unused quota. Membership is declared on the ClusterQueue:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: cq-research
spec:
  cohortName: gpu-fleet        # v1beta2 name. v1beta1 called this spec.cohort.
```

Any ClusterQueues with the same `cohortName` are in the same cohort. If it is empty, the ClusterQueue borrows from nobody and lends to nobody — L3's isolated floor.

Since the Cohort API graduated, the cohort can also be a **real object**, which unlocks two things a bare string cannot express:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: Cohort
metadata:
  name: gpu-fleet
spec:
  # (a) The cohort can OWN quota. This is ADDITIONAL capacity on top of what its
  #     member ClusterQueues declare — a shared pool that belongs to nobody in
  #     particular and can be consumed by any member.
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 16        # 16 GPUs owned by the cohort itself
  # (b) The cohort can have a PARENT, forming a CohortTree (§10).
  parentName: org-root
  fairSharing:
    weight: "0.75"
```

**Cohort-owned quota is a genuinely useful design tool**, and the natural GPU-fleet use is a *buffer*: give each team a floor sized to its steady-state need, and put the burst headroom in the cohort where it belongs to nobody and is therefore never "taken" from anyone. Nobody's guarantee is violated when the buffer is consumed, because nobody owned it.

One rule that trips people up: **to borrow a `[flavor, resource]` pair, a ClusterQueue must itself declare quota for that pair — even if the value is zero.**

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: cq-besteffort
spec:
  cohortName: gpu-fleet
  namespaceSelector: {}
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 0        # owns nothing, but is now ELIGIBLE to borrow
        borrowingLimit: 24     # ...up to 24 GPUs of other people's idle capacity
```

Omit the `a100`/`nvidia.com/gpu` entry entirely and this queue can never touch A100 quota no matter how idle the cohort is. This is the canonical shape for a "scavenger" queue that runs only on capacity nobody else wants, and it is the first half of the best-effort tier in §12's design.

Two more constraints from the API: `borrowingLimit` and `lendingLimit` **must be null when `cohortName` is empty** (a CEL rule on `ClusterQueueSpec` rejects the manifest otherwise), and a ClusterQueue may belong to exactly one cohort.

### 2. Borrowing: the fit rule, with arithmetic

L3 §7 gave the flavor-fit rule. Here is the part that only matters once a cohort exists:

```
  A pod set's request for resource R fits flavor F in ClusterQueue C if:

    (1) request ≤ unused nominalQuota for (F, R) in C                    ← plain fit
   OR
    (2) request ≤ Σ unused nominalQuota for (F, R) across C's cohort     ← the cohort has it
   AND
    (3) request ≤ unused (nominalQuota + borrowingLimit) for (F, R) in C ← C is allowed it

  When (2) and (3) hold but (1) does not, C is BORROWING.
```

Two independent caps govern it, and confusing them is the single most common Kueue error:

```yaml
resourceGroups:
- coveredResources: ["nvidia.com/gpu"]
  flavors:
  - name: a100
    resources:
    - name: "nvidia.com/gpu"
      nominalQuota: 8
      borrowingLimit: 4    # I may take at most 4 MORE than my 8  → I can reach 12
      lendingLimit: 2      # I will expose at most 2 of my 8      → I keep ≥ 6 always
```

- **`borrowingLimit`** caps how much this queue may take **from** the cohort, *above* its own nominal. Null means unlimited — up to whatever the cohort actually has spare. It protects the **cohort** from one queue hoovering all the slack.
- **`lendingLimit`** caps how much of this queue's own nominal it will expose **to** the cohort. Null means the entire idle nominal is lendable. It protects the **owner**: with `lendingLimit: 2` on an 8-GPU floor, six GPUs are never lent out, so six are always instantly available without waiting for a preemption to complete. It is stable since **v0.17**.

Memory hook: **`borrowingLimit` = how much I can take. `lendingLimit` = how much I let others take.** One caps consumption above the floor; the other caps generosity below it.

The arithmetic worth internalising, for a two-queue cohort:

```
   cq-research : nominalQuota 8,  lendingLimit unset (lends all idle)
   cq-product  : nominalQuota 12, borrowingLimit unset (borrows all available)

   Research idle, product busy:
     product's ceiling = its nominal 12 + research's lendable idle 8 = 20 GPUs

   Now set research's lendingLimit: 2
     research reserves 8 − 2 = 6 for its exclusive use
     product's ceiling = 12 + 2 = 14 GPUs
     ⇒ research gave up 6 GPUs of potential utilisation to buy
       INSTANT access to 6 GPUs instead of preemption-latency access

   Now instead set product's borrowingLimit: 4
     product's ceiling = 12 + 4 = 16, even if research is entirely idle
     ⇒ the cohort keeps 4 of research's idle GPUs unreachable by product,
       which is only rational if a third queue also wants them
```

**`lendingLimit` is the deductible on your insurance policy.** How much of your floor do you self-insure (keep instantly available) versus pool with everyone else (higher fleet utilisation, but reclaim costs you a preemption)? The right value depends entirely on how fast you need capacity: an inference service that must scale in seconds should lend almost nothing; a research team whose jobs queue for hours anyway should lend everything.

### 3. Borrow and reclaim, drawn

This is the picture the rest of the lesson refers back to.

```
  A COHORT OF TWO, THROUGH ONE BORROW-AND-RECLAIM CYCLE
  ══════════════════════════════════════════════════════════════════════════════════════
  cohort: gpu-fleet    ·    flavor: a100    ·    24 physical GPUs
  cq-research : nominalQuota 8,  reclaimWithinCohort: Any,  lendingLimit unset
  cq-product  : nominalQuota 16, reclaimWithinCohort: Any,  borrowingLimit unset

  Each ▓/░/█ is one GPU.   │ marks a ClusterQueue's nominalQuota boundary.

  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │ T0 — BOTH IDLE.  Nothing admitted; every GPU is free.                              │
  │                                                                                     │
  │  cq-research  ░░░░░░░░│                          nominal 8   used 0   borrowed 0    │
  │  cq-product   ░░░░░░░░░░░░░░░░│                  nominal 16  used 0   borrowed 0    │
  │                                                                                     │
  │  cohort borrowable = 8 (research idle) + 16 (product idle) = 24                     │
  └────────────────────────────────────────────────────────────────────────────────────┘
                                    │  product submits 3 × 8-GPU training jobs (24 GPUs)
                                    ▼
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │ T1 — PRODUCT BORROWS.  Its own 16 fill first, then it borrows research's idle 8.    │
  │                                                                                     │
  │  cq-research  ░░░░░░░░│                          used 0   lent out 8                │
  │  cq-product   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│▒▒▒▒▒▒▒▒          used 24  borrowed 8  ◀── over floor│
  │               └── own nominal 16 ──┘└ borrowed 8 ┘                                   │
  │                                                                                     │
  │  Fleet utilisation 24/24 = 100%.  Without the cohort this would be 16/24 = 67%      │
  │  and 8 GPUs would be dark. THIS IS THE ENTIRE POINT OF A COHORT.                    │
  │                                                                                     │
  │  Workload status on the borrowing jobs: Admitted=True, and                          │
  │  cq-product.status.flavorsUsage[a100].borrowed = "8"                                │
  └────────────────────────────────────────────────────────────────────────────────────┘
                                    │  research submits an 8-GPU job.
                                    │  Its own nominal is 8 and its usage is 0, so the
                                    │  request fits UNDER ITS NOMINAL QUOTA — which is
                                    │  exactly the condition that licenses reclamation.
                                    ▼
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │ T2 — RECLAIM BY PREEMPTION.  cq-research has reclaimWithinCohort: Any, so it may    │
  │      preempt ANY workload in the cohort whose ClusterQueue is over its nominal.     │
  │                                                                                     │
  │  candidate set = product's workloads, because cq-product is BORROWING.              │
  │  cq-product's own 16 nominal GPUs are NOT candidates — an owner's floor is never    │
  │  reclaimable by a sibling; only the borrowed excess is fair game.                   │
  │                                                                                     │
  │  chosen victim: one 8-GPU job in cq-product                                         │
  │    → Workload gets Evicted=True (reason Preempted)                                  │
  │      and Preempted=True (reason InCohortReclamation)                                │
  │    → Kueue sets its Job.spec.suspend = true                                         │
  │    → the Job controller DELETES its 8 pods → 8 GPUs actually free                   │
  │    → the Workload is requeued; it did not disappear                                 │
  └────────────────────────────────────────────────────────────────────────────────────┘
                                    ▼
  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │ T3 — AFTER.  Research has its guaranteed floor back. Product is back at its own.     │
  │                                                                                     │
  │  cq-research  ████████│                          used 8   borrowed 0  ◀── guarantee │
  │  cq-product   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│                  used 16  borrowed 0     honoured   │
  │                                                  + 1 workload requeued, waiting     │
  │                                                                                     │
  │  Fleet utilisation still 24/24 = 100%.                                              │
  │  What was LOST: the preempted job's progress since its last checkpoint (§11).       │
  └────────────────────────────────────────────────────────────────────────────────────┘

  THE INVARIANT THIS PICTURE ENCODES:
    a ClusterQueue's nominalQuota is ALWAYS obtainable — by force if necessary —
    and everything above it is a loan that can be called.
```

### 4. Preemption: the three levers

Preemption is configured per ClusterQueue, in `spec.preemption`, with three **independent** fields. All default to `Never`, so **a fresh ClusterQueue never preempts anything** — a cohort without preemption config gives you borrowing with no reclaim, which means a lender's "guarantee" is only honoured when the borrower finishes voluntarily.

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: cq-research
spec:
  cohortName: gpu-fleet
  preemption:
    # (a) May a pending Workload here preempt workloads in OTHER ClusterQueues of
    #     the cohort that are using more than their nominal quota?
    #       Never (default) | LowerPriority | Any
    #     `Any` = reclaim regardless of the victim's priority. This is how an owner
    #     GUARANTEES its floor comes back.
    reclaimWithinCohort: Any

    # (b) May a pending Workload here preempt cohort siblings IN ORDER TO BORROW —
    #     i.e. to go ABOVE its own nominal quota?
    #       policy: Never (default) | LowerPriority
    #     Requires reclaimWithinCohort != Never (CEL-validated).
    #     CLASSIC PREEMPTION ONLY — invalid under Fair Sharing (§8).
    borrowWithinCohort:
      policy: LowerPriority
      maxPriorityThreshold: 100    # only victims with priority ≤ 100 are eligible

    # (c) May a pending Workload here preempt THIS queue's own admitted workloads?
    #       Never (default) | LowerPriority | LowerOrNewerEqualPriority
    withinClusterQueue: LowerPriority
```

The distinction between (a) and (b) is the one people blur, and it is a fairness distinction, not a mechanical one:

- **`reclaimWithinCohort` is about taking back what is yours.** It only licenses preemption when the pending Workload fits *within its own ClusterQueue's nominal quota*. You are not asking for more than you own; you are asking for what you own and someone else is borrowing. That is why `Any` is defensible: no priority argument can outweigh an owner's floor.
- **`borrowWithinCohort` is about taking more than yours, by force.** The pending Workload does *not* fit within its own nominal, and it wants to evict someone else's work in order to borrow. That is a much stronger claim, which is why it is priority-gated *and* bounded by `maxPriorityThreshold`, and why it defaults to `Never`.

**What eviction actually does**, mechanically — reusing L3 §2's lever in reverse:

```
   Kueue selects the victim Workload
     → sets Evicted=True with reason Preempted, plus a Preempted condition
       carrying the finer reason (InClusterQueue | InCohortReclamation |
       InCohortFairSharing | InCohortReclaimWhileBorrowing) and a message
       naming the preemptor's Workload UID and Job UID
     → sets the victim Job's spec.suspend = true
     → the Job controller DELETES the pods  ⇒  GPUs are actually released
     → quota is released back to the ClusterQueue
     → the Workload is requeued (Requeued=True), NOT deleted
     → it competes for admission again like any other pending Workload
```

The victim is not lost, and it is not marked failed. It goes back to the state L3 §8 calls PENDING, with `status.accumulatedPastExecutionTimeSeconds` remembering how long it had been admitted. **What *is* lost is the work it had done since its last checkpoint** — §11 puts a number on that.

You can find the preemptor from the victim, which is the single most useful forensic command here:

```bash
$ kubectl describe workload <victim> -n product | sed -n '/Conditions/,/Events/p'
    Message: 'Preempted to accommodate a workload (UID: 5c023c28-8533-4927-b266-56bca5e310c1,
              JobUID: 4548c8bd-c399-4027-bb02-6114f3a8cdeb) due to reclamation within the cohort'
    Reason:  InCohortReclamation
    Status:  "True"
    Type:    Preempted

# then resolve that JobUID to the actual preempting Workload:
$ kubectl get workloads.kueue.x-k8s.io --all-namespaces \
    --selector=kueue.x-k8s.io/job-uid=4548c8bd-c399-4027-bb02-6114f3a8cdeb
```

### 5. Workload priority is not pod priority

Preemption is priority-driven, and Kueue deliberately gives you a priority axis that is **separate** from pod priority:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: WorkloadPriorityClass
metadata:
  name: research-high
value: 1000
description: "Interactive research jobs that a human is waiting on."
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: WorkloadPriorityClass
metadata:
  name: research-batch
value: 100
```

Reference it with a label on the Job: `kueue.x-k8s.io/priority-class: research-high`. Kueue populates `Workload.spec.priority` from it and records the source in `spec.priorityClassRef` with `group: kueue.x-k8s.io, kind: WorkloadPriorityClass`. If you instead use a core `scheduling.k8s.io` `PriorityClass` on the pod template, Kueue picks that up too, recording `group: scheduling.k8s.io, kind: PriorityClass`.

Why the separation matters on a GPU fleet: **queueing priority and node-eviction priority are different decisions.** A long batch training run may deserve a low *queueing* priority (it can wait; it should yield to interactive work) while needing a high *pod* priority (once it is running, the kubelet must not evict it under node pressure ahead of a sidecar). Encoding both on one number forces you to choose. And note the API's own caveat: changing a `WorkloadPriorityClass`'s `value` does **not** re-prioritise Workloads that already exist.

### 6. Classic preemption, as an ordered algorithm

"Kueue preempts the lowest-priority thing" is the folk version, and it is wrong often enough to matter. The real procedure has four stages.

**Stage 1 — is preemption licensed at all?** An incoming Workload that does not fit in unused quota may issue preemptions only if:

- its requests are **below its flavor's nominal quota** in its own ClusterQueue (the reclaim case), **or**
- `borrowWithinCohort` is enabled for its ClusterQueue.

Otherwise it simply waits. This is the rule that stops a queue from evicting its way past its own floor by default.

**Stage 2 — compile the candidate list.** Candidates are admitted Workloads that either:

- belong to the **same** ClusterQueue as the preemptor and satisfy its `withinClusterQueue` policy; or
- belong to **other** ClusterQueues in the cohort that are **actively borrowing**, and satisfy the preemptor's `reclaimWithinCohort` and `borrowWithinCohort` policies.

The "actively borrowing" clause is the guarantee from §3 restated as code: a sibling operating **at or below** its nominal quota is never a candidate.

**Stage 3 — sort the candidates.** This is `CandidatesOrdering` in `pkg/scheduler/preemption/common/ordering.go`, and its five keys, in order, are:

```
  0. Workloads already marked for preemption first
       (do not double-charge: reuse an eviction already in flight)
  1. Workloads from OTHER ClusterQueues in the cohort, before ones in the
       preemptor's own ClusterQueue
       (prefer reclaiming a loan over cannibalising your own team's work)
  2. (AdmissionFairSharing only) Workloads whose LocalQueue has LOWER usage first
       — note the direction: within one ClusterQueue, this comparison is applied
       so that the heavier-consuming LocalQueue's work is preferred as a victim
  3. Workloads with LOWER priority first
  4. Workloads admitted MORE RECENTLY first
       (a job 10 minutes in has less to lose than one 10 hours in — this is the
        rule that makes preemption cheaper in GPU-hours, see §11)
  5. UID, purely for deterministic tie-breaking
```

Key 4 is the one worth remembering for interviews and for cost reasoning: **among equal-priority candidates, Kueue evicts the youngest.** That is a direct minimisation of stranded work.

**Stage 4 — qualify candidates as targets, then minimise.** Kueue walks the sorted list, provisionally removing each candidate from its snapshot, until the incoming Workload fits. Four heuristics govern whether borrowing is permitted while doing so:

```
  1. If ALL candidates belong to the preemptor's own ClusterQueue:
       greedily qualify until the Workload fits, allowing borrowing up to
       borrowingLimit.
  2. Else if borrowWithinCohort is enabled:
       greedily qualify (respecting maxPriorityThreshold), allowing borrowing.
  3. Else if the preemptor's ClusterQueue is currently BELOW its nominal quota:
       greedily qualify, DISALLOWING borrowing — a pure reclaim.
  4. If none of the above made it fit:
       greedily qualify only candidates from the preemptor's OWN ClusterQueue,
       allowing borrowing.

  THEN — the minimisation pass:
    walk the target list in REVERSE and try to add each victim back;
    if the incoming Workload still fits without it, drop it from the targets.
```

That final pass matters more than it looks. The greedy walk overshoots — it may evict three jobs when the third alone would have sufficed. `fillBackWorkloads` un-evicts everything that was not necessary, in reverse order, so the committed target set is minimal. **On a GPU fleet the difference between "preempt 3 jobs" and "preempt 1 job" is measured in tens of GPU-hours per incident.**

### 7. Fair sharing: the share value, exactly

Classic preemption is **priority-and-borrowing-state** driven. It answers "who is using quota they do not own, and who is cheapest to evict by priority." It has no memory: a queue that hogged the fleet all week gets no penalty today.

Fair sharing changes the *objective* of victim selection from priority to **share equalisation**. Enable it in the Kueue Configuration — and note this changed shape in `config.kueue.x-k8s.io/v1beta2`:

```yaml
apiVersion: config.kueue.x-k8s.io/v1beta2
kind: Configuration
fairSharing:
  preemptionStrategies: [LessThanOrEqualToFinalShare, LessThanInitialShare]
```

**There is no `enable: true` field in v1beta2.** Fair sharing is on if and only if the `fairSharing` block is present — literally `func Enabled(pfs *config.FairSharing) bool { return pfs != nil }`. (The v1beta1 Configuration did have `enable`, and its conversion drops the block entirely when `enable: false`, and fills in `[LessThanOrEqualToFinalShare, LessThanInitialShare]` when `enable: true` with no strategies given. If you are copying an old manifest, this is the line to change.)

Each ClusterQueue and Cohort may carry a weight:

```yaml
spec:
  fairSharing:
    weight: "2"     # default 1. Higher weight ⇒ entitled to a larger share.
                    # Weight 0 ⇒ INFINITE share while borrowing: always the first
                    # victim and the last admitted. Must be > 10⁻⁹ if non-zero.
```

**The share value, from the source.** For a ClusterQueue or Cohort node `c`:

```
   For each resource r that c provides:
       borrowed(r) = Σ over flavors ( usage above the node's quota for (flavor, r) )
       lendable(r) = Σ over flavors ( nominalQuota — or lendingLimit where set — )
                     for r across the hierarchy of c's PARENT

       ratio(r)    = borrowed(r) × 1000 / lendable(r)          ← note the ×1000

   unweightedRatio   = max over r of ratio(r)      ← this is the DOMINANT resource
   dominantResource  = the r that achieved that max (ties broken alphabetically)

   weightedShare     = ceil( unweightedRatio / weight )        ← published in status

   Special cases from the code:
     · not borrowing anything          →  share = 0
     · borrowing with weight = 0       →  share = 9223372036854775807 (MaxInt64)
     · a node with no parent (the root) →  share = 0
```

Because of the ×1000, **`weightedShare` is in per-mille of the lendable pool**: 1000 means "borrowing an amount equal to 100% of what the cohort can lend." That single fact makes the metric readable at a glance, and almost nobody knows it.

Work it through on the §3 cohort, extended to three queues:

```
   cohort gpu-fleet, flavor a100, 24 GPUs
     cq-research   nominal 8   weight 1
     cq-product    nominal 12  weight 1
     cq-besteffort nominal 4   weight 1

   lendable for nvidia.com/gpu across the cohort = 8 + 12 + 4 = 24

   State: product is using 20 (8 borrowed), besteffort is using 4 (0 borrowed),
          research is using 0.

     cq-product   : borrowed = 20 − 12 = 8
                    ratio    = 8 × 1000 / 24 = 333.3
                    share    = ceil(333.3 / 1) = 334
     cq-besteffort: borrowed = 4 − 4 = 0      → share = 0
     cq-research  : borrowed = 0              → share = 0

   Kueue ADMITS from the lowest share first and PREEMPTS from the highest share
   first. Product, at 334, is the preemption target and the last to be admitted.

   Now give besteffort weight 0.5 (half-entitled) and suppose it borrows 6:
     cq-besteffort: borrowed = 6, ratio = 6 × 1000 / 24 = 250
                    share    = ceil(250 / 0.5) = 500      ← worse than product's 334

   Same 6 borrowed GPUs, but at half weight it looks twice as greedy. That is
   what a weight IS: a divisor on your measured greediness.
```

Read the metric directly:

```bash
$ kubectl get clusterqueue cq-product -o jsonpath='{.status.fairSharing}'
{"weightedShare":334}

# Or, fleet-wide, the series to graph:
#   kueue_cluster_queue_weighted_share{cluster_queue, cohort}
```

**The multi-resource point — this is DRF.** With GPUs *and* CPU *and* memory in play, the share is the **maximum** over resources of each resource's borrowing ratio. A queue that is GPU-light but memory-heavy can have a high dominant share on memory while looking GPU-idle, and fair sharing will treat it as the greedy one. That is a direct instance of **Dominant Resource Fairness** (Ghodsi, Zaharia, Hindman, Konwinski, Shenker, Stoica, NSDI 2011), which the KEP cites explicitly as the basis of its value function. Naive "equalise GPU count" fairness would miss it. You meet DRF again in [L5](05-alternatives-volcano-kai.md), where Volcano implements it as its *primary, native* fairness model rather than layering it on a quota-first system — same theory, two architectural choices about where it lives.

### 8. Fair-sharing preemption: the strategies, the inversion, and why it cannot loop

**The strategies.** `preemptionStrategies` gates whether a *particular* eviction is allowed, by comparing shares before and after. The two rules correspond to S2-a and S2-b in KEP-1714:

| Strategy | Rule | Effect |
|---|---|---|
| `LessThanOrEqualToFinalShare` | preempt only if the preemptor's share **with** its new workload ≤ the target's share **without** the victim | conservative and self-limiting. Favours preempting *smaller* workloads in the target queue, regardless of priority or start time, because removing a small workload leaves the target's share high |
| `LessThanInitialShare` | preempt only if the preemptor's share **with** its new workload < the target's share **as it is now** | more aggressive; does not depend on the victim's size at all, so within the target queue it picks lowest priority and newest start time first |

Only three orderings are accepted, and any other combination **fails configuration validation and Kueue does not start**:

```
  [LessThanOrEqualToFinalShare]
  [LessThanInitialShare]
  [LessThanOrEqualToFinalShare, LessThanInitialShare]     ← the usual choice
```

The list is tried in order: the algorithm only falls through to the next strategy when no remaining candidate satisfies the current one and the preemptor still does not fit. The default, when `fairSharing` is present with no strategies specified, is the two-element list.

The algorithm itself:

```
  FindFairPreemptionTargets(X ClusterQueue, W Workload)
    For each preemption strategy, in order:
      While W does not fit and candidates remain:
        Find the ClusterQueue Y with the HIGHEST share value
        For each admitted Workload U in Y:
          If U satisfies the current strategy:
            add U to targets
    Then, in reverse order over targets:
      try to remove each from the target list while W still fits   ← minimisation
```

**The victim inversion — worked, not asserted.** This is the most-tested trap in this space, so do it with numbers.

```
   cohort gpu-fleet, 24 lendable GPUs, all weights 1.
     cq-alpha : nominal 6.  Using 18  → borrowed 12 → share = ceil(12×1000/24) = 500
                Its workloads are priority 1000 (high).
     cq-beta  : nominal 6.  Using 8   → borrowed 2  → share = ceil( 2×1000/24) = 84
                Its workloads are priority 1 (low).
     cq-gamma : nominal 12. Using 0. Submits an 8-GPU Workload at priority 500.

   CLASSIC PREEMPTION:
     candidates = borrowing queues' workloads = alpha's AND beta's.
     Sort by CandidatesOrdering key 3: LOWER PRIORITY FIRST.
     → beta's priority-1 workloads are evicted first.
     Result: the frugal queue (borrowed 2) pays, the greedy one (borrowed 12)
             is untouched, because greed is not a term in the classic ordering.

   FAIR SHARING:
     the algorithm picks "the ClusterQueue with the HIGHEST share value" — alpha,
     at 500 — and looks for workloads in ALPHA that satisfy the strategy.
     → an alpha workload is evicted, DESPITE being priority 1000 vs beta's 1.
     Result: the greedy queue pays.

   ⇒ The inversion: fair sharing evicts a HIGHER-PRIORITY workload because its
     QUEUE is over-share. Priority still orders victims WITHIN a queue; share
     orders ACROSS queues, and share is consulted first.
```

The cost framing follows directly. **Classic** says "important work wins, and a history of hogging is free." **Fair sharing** says "sustained hogging is penalised; no queue can camp on borrowed GPUs indefinitely." Choose fair sharing when many teams share one fleet and you must defend equitable long-run access; choose classic when a strict priority order (prod above batch) must always hold no matter who has been greedy.

**Why it cannot ping-pong.** An obvious worry: if A preempts B to equalise shares, does B then preempt A back forever? Kueue's docs carry a short proof that it cannot, and it is worth being able to reproduce because it is a genuine "do you understand the mechanism" question.

```
  Let DRS_X_pending  = ClusterQueue X's share BEFORE admitting its workload
      DRS_X_admitted = X's share AFTER admitting it
  Assume both workloads are borrowing after admission (non-borrowing queues are
  pruned from the candidate set — nextTarget excludes nodes with DRS = 0),
  so DRS_X_pending < DRS_X_admitted for both.

  For A to preempt B, a strategy must hold:
       DRS_A_admitted ≤ DRS_B_pending   (S2-a)   or   DRS_A_admitted < DRS_B_admitted (S2-b)
  Since DRS_B_pending < DRS_B_admitted, either case gives:
       LEMMA 1:  A preempts B  ⇒  DRS_A_admitted < DRS_B_admitted

  By the identical argument in the other direction:
       LEMMA 2:  B preempts A  ⇒  DRS_B_admitted < DRS_A_admitted

  Both cannot hold. Therefore if A preempts B, B cannot preempt A afterwards. ∎

  Corollaries the docs draw out:
    · if A preempts a whole set {B₁…B_k}, none of them may preempt A afterwards;
    · it generalises along a chain: A→B→C gives
      DRS_A_admitted < DRS_B_admitted < DRS_C_admitted, so C cannot preempt A.
  Stated limitation: an equivalent proof for the HIERARCHICAL case is still open.
```

**Why `borrowWithinCohort` is invalid under fair sharing.** `borrowWithinCohort` selects victims by a **priority threshold** (`maxPriorityThreshold`) — its entire logic is denominated in priority. Fair sharing **replaces** priority-based victim selection with share-based selection and its own strategies. There is no coherent way to honour "only victims below priority 100" while also equalising shares: the two speak different currencies, and honouring one necessarily violates the other. Kueue therefore forbids the combination — the API comment on `BorrowWithinCohort` states it "may only be configured with Classical Preemption, and __not__ with Fair Sharing." Under fair sharing, borrow-driven preemption is already governed by the share strategies, making the knob both redundant and contradictory. There is a second, related CEL rule you will hit sooner: `reclaimWithinCohort: Never` together with `borrowWithinCohort.policy != Never` is rejected outright, because you cannot license the stronger claim while forbidding the weaker one.

### 9. Fair sharing on the admission side, too

Fair sharing is not only about eviction. The scheduler's ordering iterator is swapped when it is enabled: `makeIterator` returns `makeFairSharingIterator` instead of `makeClassicalIterator`, so the four-key classic sort from L3 §7 is replaced by ordering on dominant resource share. **Kueue admits from the lowest-share ClusterQueue first.**

There is also a nominal-first refinement: under the `FairSharingPrioritizeNonBorrowing` feature gate, a Workload whose subtree is *not* borrowing is preferred over one whose subtree is, before the share comparison is consulted at all. The check is **per-flavor** — a subtree borrowing on flavor X but with ample quota on flavor Y does not penalise a Workload requesting Y — and in a hierarchy it is applied at every level from the ClusterQueue to the root.

The practical consequence for a GPU fleet: with fair sharing on, a team that has been under-using is *both* admitted sooner *and* protected from eviction, and a team that has been over-borrowing is *both* pushed back in line *and* first in the firing line. The two effects compound, which is why fair sharing converges rather than oscillating.

### 10. Hierarchical cohorts, and what "who owes whom" means in a tree

A Cohort may have a `parentName`, forming a **CohortTree**. This is how you express organisational structure in quota:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: Cohort
metadata: { name: org-root }
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: Cohort
metadata: { name: research-org }
spec:
  parentName: org-root
  fairSharing: { weight: "0.75" }     # research trends toward 75% of shared capacity
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: Cohort
metadata: { name: platform-org }
spec:
  parentName: org-root
  fairSharing: { weight: "0.25" }
```

```
  A COHORT TREE, AND WHERE BORROWING HAPPENS
  ═══════════════════════════════════════════════════════════════════════════

                          ┌───────────────────┐
                          │    org-root       │  cohort-level nominalQuota:
                          │  (Cohort)         │  16 GPUs of shared buffer
                          └─────────┬─────────┘
                    ┌───────────────┴───────────────┐
                    │                               │
          ┌─────────▼─────────┐           ┌─────────▼─────────┐
          │   research-org    │           │   platform-org    │
          │  weight 0.75      │           │  weight 0.25      │
          └────┬─────────┬────┘           └─────────┬─────────┘
               │         │                          │
      ┌────────▼──┐  ┌───▼───────┐          ┌───────▼────────┐
      │ cq-vision │  │ cq-nlp    │          │ cq-inference   │
      │ nominal 24│  │ nominal 24│          │ nominal 32     │
      └───────────┘  └───────────┘          └────────────────┘

  BORROWING ORDER: a Workload in cq-vision looks for capacity
    1. in cq-vision's own nominal quota
    2. from its siblings under research-org (cq-nlp's idle quota)
    3. from research-org's own cohort-level quota, if any
    4. only then across to platform-org's subtree, through org-root

  WHY THE ORDER MATTERS — the LCA reasoning from KEP-1714:
    For a preemptor x and a candidate y, let LCA(x,y) be their lowest common
    ancestor and AlmostLCA(x,y) the last node on the path from x before that
    ancestor. Kueue compares the subtrees rooted at AlmostLCA(x,y) and
    AlmostLCA(y,x), and classifies:

      [S1] AlmostLCA(x,y) is NOT borrowing, yet x still lacks capacity
           ⇒ someone else must be borrowing what x is entitled to
           ⇒ this is a RECLAIM. Priority is irrelevant; take it back.

      [S2] AlmostLCA(x,y) IS borrowing
           ⇒ x is asking for more than its subtree's entitlement too
           ⇒ this is a REBALANCE, and it is only allowed if y's subtree is
             borrowing MORE than x's, per the preemption strategies.

    Before either, Kueue checks that y and every cohort on the path up to
    AlmostLCA(y,x) are borrowing — if y is consuming quota that genuinely lies
    under its own subtree, x has no claim on it at all.
```

Fair Sharing became compatible with hierarchical cohorts in **Kueue v0.11** (both for preemption and for scheduling), so the share computation runs across a CohortTree, not only a flat cohort. A cycle in `parentName` disables every member of the cohort — ClusterQueues included — and blocks admission until the cycle is removed.

**Verify hierarchical borrowing behaviour against your version.** Upstream issue **#7016**, "Clusterqueues must prefer borrowing within a cohort before borrowing across cohorts," reported that a ClusterQueue with access to multiple flavors could reach across cohort boundaries before exhausting its own cohort, causing unnecessary cross-cohort preemptions even though the nearer subtree had capacity. It is now **closed**, and adjacent fixes have continued to land — v0.19.1 alone shipped a fix for a `flavorFungibility.preference: PreemptionOverBorrowing` case where a flavor requiring preemption could outrank a later flavor that fits "purely because its quota was sourceable at a shallower borrowing level in the cohort tree." Hierarchical borrowing order is an area where the semantics have been actively refined; do not assume last year's mental model.

### 11. Two fairness layers, and the cost of an eviction

**The layers.** L3 introduced `AdmissionFairSharing`. This lesson's `fairSharing` is a different, older, cross-ClusterQueue mechanism. Current Kueue runs both on a busy fleet:

| | `AdmissionFairSharing` ([L3](03-kueue-queueing-model.md) §10) | Cohort `fairSharing` (this lesson) |
|---|---|---|
| Scope | LocalQueues **within one** ClusterQueue | ClusterQueues (and Cohorts) **within a CohortTree** |
| Acts on | pending Workloads | admitted Workloads |
| Mechanism | admission **ordering** | **preemption** (and admission ordering, §9) |
| Currency | decayed historical consumption per LocalQueue | dominant resource share per ClusterQueue |
| Can it evict a running job? | **no** | **yes** |
| Turned on by | feature gate (default on since v0.15) **plus** `admissionScope` on the CQ | presence of the `fairSharing` block in the Kueue Configuration |
| Weight lives on | `LocalQueue.spec.fairSharing.weight` | `ClusterQueue`/`Cohort` `.spec.fairSharing.weight` |

They compose within a single Workload's lifecycle: it can be admitted *later* than a sibling because its LocalQueue ranked poorly on decayed usage, and — once admitted and borrowing — be *preempted* because its whole ClusterQueue's share grew too large. They even meet inside the preemption sort: key 2 of `CandidatesOrdering` consults LocalQueue fair-sharing usage, but only when the two candidates are in the same ClusterQueue and different LocalQueues and AdmissionFairSharing is active. First is a queueing-order penalty; second is an eviction. Treating them as one knob reads as surface-level knowledge.

**Now the cost.** Borrowing is not free, and the price is paid by the borrower at reclaim time. This is the arithmetic that turns "preemption is a trade-off" into a number.

```
  STRANDED GPU-HOURS FROM ONE RECLAIM
  ═══════════════════════════════════════════════════════════════════════════

  Scenario: product's 8-GPU job has been borrowing research's floor. It has been
  running 5.0 hours. Research reclaims. What did the fleet lose?

  Parameters:
    G  = 8            GPUs held by the victim
    T  = 5.0 h        time since the job was admitted
    C  = 1.0 h        checkpoint interval
    t_c = 0.0 h       (checkpoints are async here; assume negligible write cost)
    R  = 0.25 h       restart overhead: image pull, NCCL init, dataloader warm-up
    p  = $2.50/GPU-h  blended rate

  (a) WITH checkpointing every hour.
      Work lost = time since the LAST checkpoint. Uniformly distributed over
      the interval, so the EXPECTED loss is C/2:
        lost_compute = G × C/2      = 8 × 0.5  =  4.0 GPU-hours
        restart cost = G × R        = 8 × 0.25 =  2.0 GPU-hours
        ─────────────────────────────────────────────────────────
        expected stranded            =  6.0 GPU-hours  =  $15.00
        as a fraction of the 40 GPU-hours consumed: 15%

      Worst case (preempted just before a checkpoint): 8 × 1.0 + 2.0 = 10 GPU-h.

  (b) WITHOUT checkpointing.
      Work lost = EVERYTHING since admission.
        lost_compute = G × T        = 8 × 5.0  = 40.0 GPU-hours
        restart cost = G × R        =            2.0 GPU-hours
        ─────────────────────────────────────────────────────────
        stranded                     = 42.0 GPU-hours  =  $105.00
        as a fraction of the 40 GPU-hours consumed: 105%

      ⇒ A job with no checkpoints costs MORE than it accomplished when preempted.
        And it grows without bound: a 20-hour uncheckpointed job strands
        8 × 20 + 2 = 162 GPU-hours, $405, from a single reclaim event.

  THE POLICY THAT FALLS OUT — the break-even on checkpoint frequency.
    Let f = expected preemptions per hour of runtime, and ignore restart cost
    (it is incurred either way).
      expected loss per hour of runtime  =  f × G × C/2
      checkpoint overhead per hour       =  G × t_c / C
    Setting them equal and solving for the optimal interval:
      C* = sqrt( 2 · t_c / f )

    With a checkpoint that costs t_c = 0.05 h (3 minutes of stalled training)
    and one preemption expected every 20 hours (f = 0.05/h):
      C* = sqrt(2 × 0.05 / 0.05) = sqrt(2) ≈ 1.41 hours

    So: checkpoint roughly every 85 minutes. If preemptions get 4× more frequent
    (f = 0.2/h), C* = sqrt(0.5) ≈ 0.71 h — checkpoint about every 42 minutes.
    ⇒ Optimal checkpoint interval scales as 1/sqrt(preemption rate), which is why
      the platform team owes tenants a PREEMPTION RATE SLO, not just a warning
      that preemption exists. "Borrowed capacity is preempted at most once per
      20 GPU-hours" is a number a tenant can design a checkpoint policy against.
      "Your job may be evicted" is not.

  AND THE FLEET-LEVEL COMPARISON, which is the number that decides the design:
    128-GPU fleet, four teams.
      Isolated floors  : 60% utilisation → 76.8 GPU-h of work per hour.
      Cohort+borrowing : 85% utilisation → 108.8 GPU-h/hour GROSS.
        Cost of reclaims: say 3 reclaim events per day, 6.0 stranded GPU-hours
        each (case (a) above) = 18 GPU-h/day = 0.75 GPU-h/hour.
        NET = 108.8 − 0.75 = 108.05 GPU-h/hour.
      ⇒ +40.6% net throughput. Preemption overhead is 0.7% of the gain.

    That ratio is the whole argument, and it is entirely conditional on
    checkpointing. Redo it with case (b)'s 42 GPU-hours per event:
        3 × 42 = 126 GPU-h/day = 5.25 GPU-h/hour → NET 103.55, still +35%.
    Even uncheckpointed, borrowing wins on this fleet — but the margin is 7×
    thinner, and it inverts entirely if reclaim events are frequent
    (at ~20 events/day uncheckpointed, borrowing is a net LOSS).
```

**The conclusion to carry into an interview:** preemption is economically useless — sometimes negative — without checkpointing, and the platform's job is not just to enable borrowing but to publish a preemption-rate SLO that tenants can design against. [L8](08-priority-preemption-capacity-economics.md) develops this into the full preemption-economics picture.

### 12. Designing the knobs: a three-tier fleet

Pulling it together into the shape the checkpoint asks you to defend. Three tiers, one cohort:

| Tier | `nominalQuota` | `borrowingLimit` | `lendingLimit` | `reclaimWithinCohort` | Why |
|---|---|---|---|---|---|
| **prod inference** | sized to p99 demand | small or 0 | **0** | `Any` | needs capacity in seconds; must never wait on a preemption, so lends nothing |
| **research teams** | sized to steady-state | generous | unset (lend all) | `Any` | bursty; wants the fleet's slack and is willing to lend its own |
| **best-effort / scavenger** | **0** (but declared) | very large | n/a | `Never` | owns nothing, runs only on other people's idle capacity, always the first victim |

The ordering property people ask about — "guarantee that best-effort is evicted before product or research" — is **not** achieved by setting anything on best-effort. It falls out of two facts:

1. Best-effort's `nominalQuota` is 0, so **everything it runs is borrowed**, and borrowed capacity is the only capacity that is ever a preemption candidate.
2. The other tiers set `reclaimWithinCohort: Any` on *their own* ClusterQueues, so they can reclaim from best-effort regardless of its workloads' priority.

Best-effort's own `reclaimWithinCohort` is nearly irrelevant — with zero nominal quota it has nothing to reclaim *from* anyone. Give its workloads a low `WorkloadPriorityClass` as well, so that even in the classic candidate sort it comes first among borrowers.

Two refinements worth stating in a design review:

- **Set `lendingLimit: 0` on the prod tier, not `borrowingLimit: 0`.** Those sound similar and do opposite things. `lendingLimit: 0` means prod never gives its floor away, so it never has to wait for a preemption to reclaim. `borrowingLimit: 0` would mean prod cannot use the fleet's slack during a spike, which is the opposite of what you want.
- **Put burst headroom in the Cohort object rather than in a team's floor.** Capacity owned by the cohort is consumed without anyone being over their nominal, so it generates no preemption at all — the cheapest capacity in the system, because reclaiming it costs zero stranded GPU-hours.

## Perspectives

**Developer/tenant.** From inside a borrowing team, admission just happens faster when a sibling is idle — and the job can be evicted later when the owner reclaims. The tenant obligation that follows is checkpoint tolerance, and §11 shows it is not a nicety: an uncheckpointed 8-GPU job preempted at hour five strands 42 GPU-hours, more than it produced. A tenant who does not know this treats a mid-run eviction as a platform bug rather than the system working as designed, which is a documentation failure on the platform's side as much as a discipline failure on theirs.

**Operator/platform.** Choosing `reclaimWithinCohort: Any` versus a priority-gated reclaim is a fairness-versus-predictability call the platform owns, not a default to accept. `Any` gives owners a *hard* guarantee — the right answer for prod-adjacent teams — at the cost that borrowers are never safe regardless of how important their work looked a moment ago. The corresponding obligation is a **preemption-rate SLO**: publish how often borrowed capacity is reclaimed, so tenants can size their checkpoint interval (§11's `C* = sqrt(2·t_c/f)`) instead of guessing.

**Systems/algorithms.** Kueue's fair sharing is a practical instance of Dominant Resource Fairness, with two engineering additions worth noticing. First, the share is computed **per node in a tree**, independently of its children — the value for a subtree depends only on the sums of usage and quota within it, which is what makes the hierarchy tractable. Second, the two preemption strategies are not heuristics bolted on but the S2-a/S2-b rules of the design, chosen so that the no-loop proof in §8 goes through. [L5](05-alternatives-volcano-kai.md) shows Volcano treating DRF as its primary scheduling model instead — the same theory, a different architectural decision about where fairness lives.

**Economics/FinOps.** The cohort is a shared insurance pool. `nominalQuota` is the premium each team pays in dedicated floor; borrowing is the payout when someone else is idle; `lendingLimit` is the deductible — how much you self-insure versus pool. `reclaimWithinCohort: Any` is the policy term that says claims are honoured immediately and unconditionally. That framing is a strong interview answer to "how do you think about quota design," because it maps every knob onto a financial instrument the interviewer already understands, and because it makes the right question obvious: what is the claim latency, and who bears the loss when a claim is made?

## Real-world use cases

- **IBM Research — Vela and Blue Vela.** *What it shows:* cohorts and borrowing used explicitly to let idle GPU floors serve bursty demand across research teams on real multi-hundred-GPU clusters (A100 on Vela, H100 on Blue Vela). Their stated framing — the challenge is getting more out of the GPUs you have, not getting more GPUs — is the §Why-this-matters calculation restated by an operator of exactly this fleet shape. *Verification note:* arXiv 2407.05467 and the KubeCon EU 2025 "Build An AI Cluster" tutorial deck are the sources; neither is reachable from this environment's egress proxy.

- **kubernetes-sigs/kueue issue #7016 — "Clusterqueues must prefer borrowing within a cohort before borrowing across cohorts."** *What happened:* a ClusterQueue with access to multiple resource flavors could reach across cohort boundaries to borrow before exhausting the capacity available inside its own cohort, causing preemptions in the far subtree that were unnecessary because the near one had quota. *What it shows:* hierarchical borrowing order is a real, subtle, and actively-maintained part of the system — and that a "correct" borrowing decision depends on *distance in the cohort tree*, not just on whether quota exists somewhere. *Status:* **closed**. Related fixes have continued landing: v0.19.1 shipped one for a `flavorFungibility.preference: PreemptionOverBorrowing` case where a flavor needing preemption outranked a later flavor that fit, "purely because its quota was sourceable at a shallower borrowing level in the cohort tree." *Verification note:* the issue title and closed state were read from GitHub this session; the v0.19.1 entry from `CHANGELOG/CHANGELOG-0.19.md` in the cloned repo.

- **The upstream no-loop proof, as an artefact.** *What it shows:* Kueue's own `fair_sharing.md` carries a formal argument that two Workloads in different ClusterQueues cannot preempt each other in a cycle, with lemmas, corollaries for chains, and an explicitly stated open limitation for the hierarchical case. *What to take from it:* the maturity signal. A scheduler that ships a termination proof for its preemption rules, and names the case the proof does not yet cover, is a different class of artefact from one that ships a heuristic. When you are choosing between schedulers in [L5](05-alternatives-volcano-kai.md), "does the project reason about convergence" is a legitimate criterion. *Verification note:* read directly from `site/content/en/docs/concepts/fair_sharing.md` in the cloned repository.

- **Kueue v0.11 — Fair Sharing made compatible with Hierarchical Cohorts.** *What it shows:* two changelog entries, one for preemption and one for scheduling, marking the point at which share computation started working across a CohortTree rather than a flat cohort. *What to take from it:* if you are running an older Kueue and have built a cohort hierarchy, fair sharing may not behave the way this lesson describes. This is the concrete instance of the module README's "verify feature gates" warning. *Verification note:* `CHANGELOG/CHANGELOG-0.11.md`, verified in the clone.

## Worked example

Continue from [L3](03-kueue-queueing-model.md)'s cluster: 24 fake `nvidia.com/gpu`, one `a100` flavor, Kueue installed, `cq-research` and `cq-product` with LocalQueues. Extend to a three-queue cohort so you can see a *chain* of borrow and reclaim, not just one hand-off.

**Step 1 — the cohort object, with its own buffer.**

```yaml
# 10-cohort.yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: Cohort
metadata:
  name: gpu-fleet
spec:
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 2      # a 2-GPU buffer owned by nobody: consumable without
                             # putting any ClusterQueue over its nominal, hence
                             # reclaimable at ZERO stranded-GPU-hour cost
```

**Step 2 — three ClusterQueues in it.**

```yaml
# 11-cqs.yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata: { name: cq-research }
spec:
  cohortName: gpu-fleet
  namespaceSelector: {}
  preemption:
    reclaimWithinCohort: Any          # I can always take my floor back, by force
    withinClusterQueue: LowerPriority
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 8
        # lendingLimit unset ⇒ lend all idle capacity
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata: { name: cq-product }
spec:
  cohortName: gpu-fleet
  namespaceSelector: {}
  preemption:
    reclaimWithinCohort: Any
    withinClusterQueue: LowerPriority
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 12
        borrowingLimit: 12       # may reach 24 total
        lendingLimit: 4          # but always keeps 8 instantly reclaimable
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata: { name: cq-besteffort }
spec:
  cohortName: gpu-fleet
  namespaceSelector: {}
  preemption:
    reclaimWithinCohort: Never   # owns nothing; can never initiate a reclaim
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 0          # DECLARED but zero — required to be eligible
        borrowingLimit: 24
```

Plus a low priority class for the scavenger tier:

```yaml
# 12-priorities.yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: WorkloadPriorityClass
metadata: { name: scavenger }
value: 1
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: WorkloadPriorityClass
metadata: { name: normal }
value: 100
```

**Step 3 — fill the fleet from the bottom.** Best-effort submits three 8-GPU jobs (`kueue.x-k8s.io/priority-class: scavenger`) while everyone else is idle.

```bash
$ kubectl get clusterqueues -o custom-columns=\
NAME:.metadata.name,COHORT:.spec.cohortName,\
PENDING:.status.pendingWorkloads,ADMITTED:.status.admittedWorkloads
NAME            COHORT      PENDING   ADMITTED
cq-besteffort   gpu-fleet   1         2
cq-product      gpu-fleet   0         0
cq-research     gpu-fleet   0         0
```

*(Representative transcript.)* Two admitted, one pending. Why two and not three? Lendable capacity is research's 8 + product's `lendingLimit` 4 + the cohort's own 2 = **14**, so best-effort can borrow 14 of the 24 physical GPUs, which fits one 8-GPU job with 6 left over. `lendingLimit: 4` on product is doing exactly its job: 8 of product's 12 are untouchable.

```bash
$ kubectl get clusterqueue cq-besteffort -o jsonpath='{.status.flavorsUsage}' | jq
[{"name":"a100","resources":[{"name":"nvidia.com/gpu","total":"8","borrowed":"8"}]}]
```

`total` 8, `borrowed` 8 — every GPU it holds is somebody else's. **That is what a scavenger tier looks like in the status API.**

**Step 4 — product reclaims.** Product submits a 12-GPU job at priority `normal`, which fits within its own nominal quota of 12 — the condition that licenses reclamation.

```bash
$ kubectl create -f job-product-12gpu.yaml
$ kubectl get workloads -A -w
NAMESPACE     NAME                        QUEUE           RESERVED IN    ADMITTED
besteffort    job-scav-a-9f21c            lq-besteffort   cq-besteffort  True
product       job-prod-12g-4d10e          lq-product
besteffort    job-scav-a-9f21c            lq-besteffort                              # evicted
product       job-prod-12g-4d10e          lq-product      cq-product     True
```

Read the victim:

```bash
$ kubectl describe workload job-scav-a-9f21c -n besteffort | sed -n '/Conditions/,/Events/p'
Status:
  Conditions:
    Message: 'Preempted to accommodate a workload (UID: 8a1f..., JobUID: 33bc...)
              due to reclamation within the cohort'
    Reason:  Preempted
    Status:  "True"
    Type:    Evicted
    ---
    Message: 'Preempted to accommodate a workload (UID: 8a1f..., JobUID: 33bc...)
              due to reclamation within the cohort'
    Reason:  InCohortReclamation
    Status:  "True"
    Type:    Preempted
    ---
    Reason:  BackoffFinished
    Status:  "True"
    Type:    Requeued
Events:
  Type     Reason     Age   From             Message
  Normal   Preempted  3s    kueue-admission  Preempted to accommodate a workload ...
  Normal   Suspended  3s    kueue-job        Job suspended
```

Two conditions, not one: **`Evicted` says it was preempted; `Preempted` says why** — `InCohortReclamation` rather than `InClusterQueue`, `InCohortFairSharing`, or `InCohortReclaimWhileBorrowing`. The `Requeued` condition confirms it went back into the queue rather than being discarded. And `kubectl get pods -n besteffort` now returns nothing: the Job was re-suspended and its pods deleted, which is why the GPUs are actually free.

**Step 5 — research reclaims from a still-borrowing queue.** Research submits an 8-GPU job. Research's nominal is 8 and its usage is 0, so this is a pure reclaim. Best-effort's remaining borrowed capacity goes first (it is a borrower, and its priority is 1); if that is not enough, product's *borrowed excess* is eligible too — but product's own nominal 12 never is.

**Step 6 — turn on fair sharing and see the inversion.**

```yaml
# kueue Configuration (ConfigMap kueue-manager-config), then restart the controller
apiVersion: config.kueue.x-k8s.io/v1beta2
kind: Configuration
fairSharing:
  preemptionStrategies: [LessThanOrEqualToFinalShare, LessThanInitialShare]
```

Then give best-effort's workloads a *high* priority (say 5000, above product's 100) and re-run steps 3–4. Under classic preemption, product could not have reclaimed from best-effort with `reclaimWithinCohort: LowerPriority` — but with `Any`, priority is irrelevant. Under fair sharing, the selection reason changes to `InCohortFairSharing` and the victim is chosen from the queue with the **highest `weightedShare`**, which is best-effort (borrowing 14 of 14 lendable, weight 1 → share = ceil(14×1000/22) = 637) regardless of its priority 5000.

```bash
$ kubectl get clusterqueues -o custom-columns=\
NAME:.metadata.name,SHARE:.status.fairSharing.weightedShare
NAME            SHARE
cq-besteffort   637
cq-product      0
cq-research     0
```

**That table is the fair-sharing model made visible**: two queues at zero because they are within their nominal quota, one at 637 per-mille of the lendable pool because everything it has is borrowed.

**Step 7 — the showback line that borrowing adds.** Report nominal and borrowed GPU-hours separately, because borrowed hours are a cross-team subsidy you want visible:

```promql
# GPU-hours a queue consumed from its OWN nominal quota, last 24h
sum by (cluster_queue) (
  avg_over_time(kueue_cluster_queue_resource_usage{resource="nvidia.com/gpu"}[24h])
  - avg_over_time(kueue_cluster_queue_resource_usage{resource="nvidia.com/gpu"}[24h])
) * 24

# Simpler and honest: usage total, nominal quota, and current share, side by side
kueue_cluster_queue_resource_usage{resource="nvidia.com/gpu"}
kueue_cluster_queue_resource_nominal_quota{resource="nvidia.com/gpu"}
kueue_cluster_queue_weighted_share
```

The `borrowed` figure itself lives in `ClusterQueue.status.flavorsUsage[*].resources[*].borrowed`; scrape or export it alongside `total` so the report can say "cq-product ran 300 GPU-hours, of which 90 were borrowed from cq-research." **That sentence is the deliverable's whole point** — it makes the subsidy explicit and gives the lender a reason to keep lending.

## Practice

Continue from [L3](03-kueue-queueing-model.md)'s cluster (24 fake `nvidia.com/gpu`, `a100` flavor, Kueue v0.19.x installed, two ClusterQueues + LocalQueues). Everything below runs on kind.

**Feeds:** [Kueue setup + per-queue showback](../practice/kueue-showback/README.md).

```bash
# Put the existing queues in a cohort and give them preemption policies.
# NOTE the v1beta2 field name: cohortName, not cohort.
kubectl patch clusterqueue cq-research --type merge -p '{"spec":{
  "cohortName":"gpu-fleet",
  "preemption":{"reclaimWithinCohort":"Any","withinClusterQueue":"LowerPriority"}}}'
kubectl patch clusterqueue cq-product --type merge -p '{"spec":{
  "cohortName":"gpu-fleet",
  "preemption":{"reclaimWithinCohort":"Any","withinClusterQueue":"LowerPriority"}}}'

# Give product room to borrow research's floor.
kubectl patch clusterqueue cq-product --type json -p \
  '[{"op":"add","path":"/spec/resourceGroups/0/flavors/0/resources/0/borrowingLimit","value":"8"}]'
```

1. **Capture a borrow.** With research idle, submit enough work to `lq-product` to exceed its nominal quota. Show `cq-product.status.flavorsUsage[a100].resources[nvidia.com/gpu].borrowed` greater than zero, and show `cq-research.status.flavorsUsage` unchanged. Record the fleet utilisation you achieved versus what the isolated floors would have allowed.

2. **Capture a reclaim.** Submit an 8-GPU job to `lq-research`. Capture: the victim Workload's `Evicted` (reason `Preempted`) **and** `Preempted` (reason `InCohortReclamation`) conditions with their full messages; the `kubectl get events` line; proof that the victim's pods were deleted; and the victim's `Requeued` condition. Then use the `JobUID` from the message with `kubectl get workloads --selector=kueue.x-k8s.io/job-uid=<uid> -A` to identify the preemptor from the victim alone.

3. **Prove `lendingLimit` does what you think.** Set `lendingLimit: 4` on `cq-research`'s 8-GPU quota. Re-run experiment 1 and show product's ceiling dropped from 16 to 12. Then set it to `0` and show product cannot borrow at all. **Write one sentence on what research bought with each setting.**

4. **Prove `borrowingLimit` is the other direction.** Reset `lendingLimit`, set `borrowingLimit: 2` on `cq-product`, and show product now stops at 14 even though research is fully idle. State which of the two limits you would use to protect a latency-sensitive prod queue and why.

5. **Build the scavenger tier.** Add `cq-besteffort` with `nominalQuota: 0`, a large `borrowingLimit`, and a `scavenger` WorkloadPriorityClass. Reproduce the borrowing chain from the Worked example end to end: best-effort fills the fleet, product reclaims, research reclaims. **Then delete the `nominalQuota: 0` entry and show that best-effort can no longer borrow at all** — this is the "must declare, even zero" rule, and demonstrating it is worth more than reading it.

6. **Trigger the validation errors deliberately.** (a) Set `borrowingLimit` on a ClusterQueue with no `cohortName` → capture the rejection. (b) Set `reclaimWithinCohort: Never` together with `borrowWithinCohort.policy: LowerPriority` → capture the rejection. (c) Enable fair sharing in the Configuration *and* leave `borrowWithinCohort` configured → capture what happens. Write down which of the three fail at `kubectl apply` time and which at controller startup.

7. **See the fair-sharing inversion.** Enable `fairSharing` in the Configuration and restart the controller. Arrange two borrowing queues where the one with the **higher-priority** workloads also has the **higher `weightedShare`**, then have a third queue submit. Capture `kueue_cluster_queue_weighted_share` (or `.status.fairSharing.weightedShare`) for all three before and after, and show that the high-priority workload in the high-share queue was the victim, with `Preempted` reason `InCohortFairSharing`. **This is the single most-tested behaviour in this lesson.**

8. **Predict a share, then check it.** Before running experiment 7, compute by hand what `weightedShare` each ClusterQueue should have, using `ceil(borrowed × 1000 / lendable / weight)`. Compare with the reported value. If they disagree, find out why — the usual cause is `lendingLimit` changing the lendable denominator.

9. **Do the stranded-GPU-hours arithmetic.** For your own fleet or the §11 scenario, compute expected stranded GPU-hours per reclaim with and without checkpointing, the optimal checkpoint interval `C* = sqrt(2·t_c/f)`, and the net throughput gain of borrowing after subtracting preemption overhead. State the preemption-rate SLO you would publish to tenants.

**Acceptance (feeds the deliverable):**
- Both original ClusterQueues in `gpu-fleet` with `reclaimWithinCohort: Any`; `cq-product` with a `borrowingLimit`; a third `cq-besteffort` with `nominalQuota: 0`.
- A captured **borrow**: `status.flavorsUsage[*].borrowed > 0` with the queue over its nominal.
- A captured **reclaim-by-preemption**: both the `Evicted` and `Preempted` conditions with reasons and messages, the event, and evidence the pods were deleted.
- A captured **fair-sharing inversion**: `weightedShare` for every queue before and after, and a higher-priority workload evicted from the higher-share queue with reason `InCohortFairSharing`.
- Your hand-computed share prediction alongside the reported one.
- One paragraph in the deliverable README stating the utilisation-versus-guarantee trade-off in $/GPU-hour terms, using your own arithmetic from experiment 9, **and** the preemption-rate SLO you would publish.

## Common pitfalls

1. **Using `spec.cohort` instead of `spec.cohortName`.** *Symptom:* the ClusterQueues never borrow from each other and nothing complains. *Mechanism:* v1beta2 renamed the field. A `v1beta1` manifest is still accepted and converted, but a `spec.cohort` key inside a `v1beta2` document is simply an unknown field. Check `kubectl get clusterqueue <name> -o jsonpath='{.spec.cohortName}'` — if it is empty, you have an isolated floor wearing a cohort's name.

2. **Forgetting that a borrower must declare the `[flavor, resource]` pair.** *Symptom:* a scavenger queue with a huge `borrowingLimit` never admits anything, while the cohort is visibly idle. *Mechanism:* a ClusterQueue may only borrow quota for flavors *it defines*. `nominalQuota: 0` is the required declaration; omitting the entry makes the pair invisible to that queue entirely.

3. **Leaving preemption at its defaults and calling it a guarantee.** *Symptom:* research's floor is "guaranteed" but research waits hours for it. *Mechanism:* all three `preemption` fields default to `Never`. Borrowing without `reclaimWithinCohort` means a lender only gets its capacity back when the borrower finishes voluntarily. **Enabling borrowing and not enabling reclamation converts a guarantee into a hope.**

4. **Confusing `borrowingLimit` with `lendingLimit`.** *Symptom:* you set `borrowingLimit: 0` to protect a prod queue's floor and instead prevent it from bursting. *Mechanism:* `borrowingLimit` caps what you *take*; `lendingLimit` caps what you *give*. Protecting a latency-sensitive queue's instant access is `lendingLimit: 0`.

5. **Setting `lendingLimit: 0` everywhere "to be safe."** *Symptom:* you run a cohort's operational complexity and get isolated floors' utilisation. *Mechanism:* if nobody lends, nobody can borrow, and the cohort is decorative. The reasonable posture is `lendingLimit: 0` on the tier that genuinely cannot wait for a preemption, and unset everywhere else.

6. **Enabling fair sharing with `borrowWithinCohort` still configured.** *Symptom:* a config Kueue refuses. *Mechanism:* `borrowWithinCohort` selects victims by priority threshold; fair sharing selects by share. The two are mutually exclusive by design, and the API documents `borrowWithinCohort` as classical-preemption-only. Related and more commonly hit: `reclaimWithinCohort: Never` with `borrowWithinCohort.policy != Never` is CEL-rejected at apply time.

7. **Copying `fairSharing: {enable: true}` into a v1beta2 Configuration.** *Symptom:* fair sharing does not turn on, or the config is rejected. *Mechanism:* the `enable` field exists only in the v1beta1 Configuration. In v1beta2 the block's *presence* enables the feature; supply `preemptionStrategies` (or omit them and get the two-element default).

8. **Reasoning about preemption purely in priority terms.** *Symptom:* surprise when a priority-1000 workload is evicted while a priority-1 workload survives. *Mechanism:* under fair sharing, the **ClusterQueue** with the highest share is chosen first, and only then is a workload chosen within it. Priority orders victims *inside* a queue; share orders *across* queues, and share is consulted first.

9. **Assuming preemption picks exactly one victim, or the "obvious" one.** *Symptom:* three jobs evicted where one would have done, or a job evicted that seemed safe. *Mechanism:* the greedy walk qualifies candidates until the preemptor fits and then runs a **reverse-order fill-back pass** to drop unnecessary victims — so the final set is minimal but the intermediate set was not, and the ordering (`CandidatesOrdering`: other-queues first, then lower priority, then *most recently admitted*) is what actually decides. That "most recently admitted first" key surprises people who expect longest-running to be evicted.

10. **Enabling borrowing without checkpointing, then measuring only the throughput gain.** *Symptom:* utilisation looks great and tenants are furious. *Mechanism:* §11 — an uncheckpointed 8-GPU job preempted at hour five strands 42 GPU-hours, more than it produced. The throughput gain and the preemption loss are both real, and only reporting the first is how a platform team loses tenant trust.

11. **Assuming hierarchical borrowing order is stable across versions.** *Symptom:* a cohort tree that behaved one way on an older Kueue behaves differently after an upgrade. *Mechanism:* the semantics of *which* subtree is borrowed from first, and how borrowing distance in the tree interacts with flavor selection, have been actively refined (issue #7016; the v0.19.1 `PreemptionOverBorrowing` fix). Fair Sharing only became hierarchy-compatible in v0.11. Pin your assumptions to your version.

## Self-check

**(a) `borrowingLimit` versus `lendingLimit` — what does each cap, and what does each protect?**
**Answer:** `borrowingLimit` caps how much a ClusterQueue may take **from** the cohort *above* its own `nominalQuota`; null means unlimited, up to whatever the cohort actually has spare. It protects the **cohort** from one queue absorbing all the slack. `lendingLimit` caps how much of the queue's **own nominal** it exposes to siblings; null means all idle nominal is lendable. It protects the **owner**: a queue with `nominalQuota: 8, lendingLimit: 2` reserves `8 − 2 = 6` for its exclusive use, so six GPUs are always instantly available without waiting for a preemption to complete. One limits consumption above the floor, the other limits generosity below it. Both must be null when `cohortName` is empty — CEL-validated. Design-wise, `lendingLimit: 0` is how you protect a latency-sensitive prod queue (it never has to reclaim); `borrowingLimit: 0` would instead stop that queue from bursting, which is usually the opposite of what you want. `lendingLimit` is stable since v0.17.

**(b) Trace a borrow-and-reclaim cycle, naming the conditions and what physically happens to the GPUs.**
**Answer:** Research (nominal 8) is idle; product (nominal 16) submits 24 GPUs of work. Product's own 16 fill first, then it borrows research's idle 8 — admitted because the fit rule's clauses (2) and (3) hold while (1) does not. `cq-product.status.flavorsUsage[a100].borrowed = 8`. Research then submits an 8-GPU job. Because that request fits **within research's own nominal quota**, reclamation is licensed. With `reclaimWithinCohort: Any`, candidates are workloads in cohort queues that are *actively borrowing* — product's borrowed excess, never its own nominal 16. Kueue selects a victim, sets `Evicted=True` with reason `Preempted` and a companion `Preempted=True` condition with reason `InCohortReclamation` and a message naming the preemptor's Workload and Job UIDs, then sets the victim Job's `spec.suspend = true`. The Job controller **deletes the pods**, which is what actually frees the GPUs; quota is released; the Workload is requeued (`Requeued=True`) rather than deleted, keeping `status.accumulatedPastExecutionTimeSeconds`. Research's job is then admitted and both queues sit at their nominal quota. What was lost is the victim's progress since its last checkpoint plus its restart overhead.

**(c) When does fair-sharing preemption pick a different victim than classic priority preemption? Give a concrete case.**
**Answer:** Whenever the queue with the **larger dominant resource share** holds the **higher-priority** workloads. Concretely: in a 24-GPU-lendable cohort, `cq-alpha` (nominal 6) is using 18 → borrowed 12 → share `ceil(12×1000/24) = 500`, and all its workloads are priority 1000; `cq-beta` (nominal 6) is using 8 → borrowed 2 → share 84, and its workloads are priority 1. A third queue submits. **Classic** compiles candidates from all borrowing queues and sorts by `CandidatesOrdering`, whose priority key puts lowest first — so beta's priority-1 workloads are evicted, and alpha's greed is not a term in the ordering at all. **Fair sharing** first picks the ClusterQueue with the highest share (alpha, 500) and then a workload within it, so a **priority-1000** workload is evicted while beta's priority-1 workloads survive. The general rule: priority orders victims *within* a queue; share orders *across* queues, and share is consulted first. Cost framing: classic says "important work wins and history is free"; fair sharing says "sustained hogging is penalised."

**(d) Why can `borrowWithinCohort` not be combined with fair sharing?**
**Answer:** `borrowWithinCohort` is a **priority-threshold** mechanism: it permits preemption in service of *borrowing* (going above your own nominal quota) but only against victims at or below `maxPriorityThreshold`, and its whole selection logic is denominated in priority. Fair sharing **replaces** priority-based victim selection with **share**-based selection governed by `preemptionStrategies` (`LessThanOrEqualToFinalShare`, `LessThanInitialShare`). Honouring a priority ceiling while equalising shares is incoherent — the two speak different currencies, and satisfying one can require violating the other. Kueue therefore documents `borrowWithinCohort` as classical-preemption-only and rejects the combination. Under fair sharing the knob is also redundant: borrow-driven preemption is already governed by the strategies. Related validation you hit earlier: `reclaimWithinCohort: Never` plus `borrowWithinCohort.policy != Never` is CEL-rejected, because you cannot license the stronger claim (preempt to borrow) while forbidding the weaker one (preempt to reclaim).

**(e) Compute the `weightedShare` for a ClusterQueue, and say what the number means.**
**Answer:** For each resource the queue provides, `borrowed(r)` is its usage above its own quota summed over flavors, and `lendable(r)` is the sum of nominal quotas (or `lendingLimit` where set) for that resource across the hierarchy of the node's parent. Then `ratio(r) = borrowed(r) × 1000 / lendable(r)`, the **unweighted ratio is the maximum over resources** (that resource is the *dominant* one, ties broken alphabetically), and `weightedShare = ceil(unweightedRatio / weight)`. Example: a 24-GPU-lendable cohort where `cq-product` (nominal 12, weight 1) is using 20 → borrowed 8 → `8×1000/24 = 333.3` → share **334**. Because of the ×1000, the number is in **per-mille of the lendable pool**: 1000 means borrowing an amount equal to 100% of what the cohort can lend. Special cases from the source: a queue that is not borrowing has share **0**; a borrowing queue with `weight: 0` gets `MaxInt64` (9223372036854775807), i.e. always the first victim and last admitted; a root node with no parent is 0. Kueue admits from the lowest share and preempts from the highest. It is exposed at `.status.fairSharing.weightedShare` and as `kueue_cluster_queue_weighted_share`. The multi-resource maximum is what makes this Dominant Resource Fairness rather than "equalise GPU count": a queue that is GPU-light but memory-heavy can be the greedy one on memory.

**(f) Design a three-queue cohort so best-effort is always evicted before product or research. Which fields enforce it?**
**Answer:** Give best-effort `nominalQuota: 0` for the `[flavor, resource]` pairs it may use — **declared but zero**, because a ClusterQueue can only borrow for flavors it defines — plus a large `borrowingLimit` so it can actually consume idle capacity, and a low `WorkloadPriorityClass` (say value 1). Set `reclaimWithinCohort: Any` on **research's and product's** ClusterQueues, so they can reclaim regardless of the victim's priority. The ordering then falls out of two facts rather than any per-queue setting: everything best-effort runs is **borrowed**, and only borrowed capacity is ever a preemption candidate; and `Any` means best-effort's priority cannot protect it. Best-effort's own `reclaimWithinCohort` is nearly irrelevant — with zero nominal quota it has nothing to reclaim from anyone; set it `Never` for clarity. Two refinements worth saying out loud: put the prod tier's protection in `lendingLimit: 0` (never lend, so never wait for a reclaim) rather than `borrowingLimit: 0`; and put burst headroom in the **Cohort object's** own `nominalQuota`, because capacity owned by the cohort is consumed without putting anyone over their nominal, so reclaiming it costs zero stranded GPU-hours.

**(g) How does `AdmissionFairSharing` differ from cohort `fairSharing`, and can one Workload be affected by both?**
**Answer:** `AdmissionFairSharing` ([L3](03-kueue-queueing-model.md) §10) acts at **admission time**, **within one ClusterQueue**, ordering pending Workloads from different LocalQueues by decayed historical consumption; it **never evicts**, and it needs both the feature gate (default on since v0.15) and `admissionScope.admissionMode: UsageBasedAdmissionFairSharing` on the ClusterQueue, with weights on `LocalQueue.spec.fairSharing.weight`. Cohort `fairSharing` acts **across ClusterQueues and Cohorts in a CohortTree**, using dominant resource share, and it **does evict** admitted Workloads via preemption — plus, since it swaps the scheduler's ordering iterator, it also orders admission by lowest share. It is enabled by the presence of the `fairSharing` block in the Kueue Configuration, with weights on `ClusterQueue`/`Cohort` `.spec.fairSharing.weight`. Yes, both can touch one Workload: it can be admitted later than a sibling because its LocalQueue ranked poorly on decayed usage, and later be preempted because its ClusterQueue's share grew too large. They even meet inside the preemption sort — key 2 of `CandidatesOrdering` consults LocalQueue fair-sharing usage, but only for candidates in the same ClusterQueue and different LocalQueues with AdmissionFairSharing active.

**(h) Quantify what a reclaim costs, and derive the checkpoint policy that follows.**
**Answer:** An 8-GPU job preempted after 5 hours: **with** hourly checkpointing, expected lost compute is `G × C/2 = 8 × 0.5 = 4.0` GPU-hours (the time since the last checkpoint is uniform over the interval), plus restart overhead `G × R = 8 × 0.25 = 2.0`, so **≈6 GPU-hours ≈ $15** at $2.50/GPU-h — 15% of the 40 GPU-hours it had consumed. **Without** checkpointing, everything since admission is lost: `8 × 5.0 + 2.0 = 42` GPU-hours ≈ $105, i.e. **more than the job produced**, and it grows without bound with runtime. The policy that follows: balancing expected loss per hour (`f × G × C/2`, where `f` is preemptions per hour) against checkpoint overhead per hour (`G × t_c / C`) gives the optimal interval **`C* = sqrt(2·t_c/f)`** — with a 3-minute checkpoint and one preemption per 20 hours, `C* ≈ 1.41 h`; at four times the preemption rate, `C* ≈ 0.71 h`. Because `C*` scales as `1/sqrt(f)`, the platform owes tenants a **preemption-rate SLO**, not just a warning. At the fleet level, borrowing on a 128-GPU four-team fleet takes utilisation from ~60% to ~85% (+42% gross); at three reclaims a day with checkpointing, preemption overhead is 0.75 GPU-h/hour, so the net gain is ~+40.6% and preemption costs 0.7% of the gain. Uncheckpointed, the same three events cost 5.25 GPU-h/hour and the net gain falls to ~+35% — and inverts entirely at high preemption rates. **Preemption is economically useless, sometimes negative, without checkpointing** — the thread [L8](08-priority-preemption-capacity-economics.md) picks up.

## Connections & what's next

This completes the Kueue anchor. [L3](03-kueue-queueing-model.md) gave you the queueing and quota vocabulary; this lesson gave you the borrowing, preemption, and fairness mechanics that make a fixed fleet economically efficient rather than merely safe. The through-line is one invariant worth stating in any design review: **a ClusterQueue's `nominalQuota` is always obtainable, by force if necessary, and everything above it is a loan that can be called.** Every knob in this lesson tunes either how large the loan may be (`borrowingLimit`), how much you are willing to lend (`lendingLimit`), how forcefully it is called (`reclaimWithinCohort`), or who is chosen when several loans could be called (priority versus share).

Three threads carry forward. **DRF** reappears in [L5](05-alternatives-volcano-kai.md), where Volcano implements it as its primary, native fairness model rather than layering it on a quota-first controller — the same theory, a different architecture, and now a comparison you can make on mechanism rather than on marketing. **The checkpoint-tolerance requirement** of borrowed capacity is the direct setup for [L8](08-priority-preemption-capacity-economics.md), where the stranded-GPU-hours arithmetic from §11 becomes the full preemption-economics and commitment-ladder picture. And **the aggregate-quota blind spot** that L3 §9 named is untouched by everything here: a cohort makes more capacity reachable, and says nothing about whether the GPUs it found are on one NVLink domain — which is [L6](06-topology-aware-placement.md)'s problem, and [L7](07-fragmentation-effective-capacity.md)'s reason that usable capacity is below the numbers in your `nominalQuota` fields.

Next: **[05 — Alternatives: Volcano & KAI](05-alternatives-volcano-kai.md)**, where the same gang-scheduling and fairness problems get solved by different architectural choices, and you learn when Kueue is the wrong tool.

## References & further reading

> **A note on verification.** This environment's egress proxy blocks `kubernetes.io`, `kueue.sigs.k8s.io`, and most vendor and academic domains. Everything marked **[verified against source]** was read directly from a clone of `kubernetes-sigs/kueue` at commit `e5084fe` (2026-08-17). Entries marked **[not reachable]** are further reading only; no claim here depends on them.

**Primary sources — Kueue API and implementation**

1. **`apis/kueue/v1beta2/clusterqueue_types.go`** — https://github.com/kubernetes-sigs/kueue/blob/main/apis/kueue/v1beta2/clusterqueue_types.go — **[verified against source]**. `cohortName` (**renamed from `cohort`**), `ResourceQuota` with `nominalQuota`/`borrowingLimit`/`lendingLimit` and their exact semantics, the CEL rule forbidding `borrowingLimit` without a cohort, `ClusterQueuePreemption` with all three fields and their `Never` defaults, `BorrowWithinCohort` with `policy` and `maxPriorityThreshold` plus the comment that it works with Classical Preemption "and __not__ with Fair Sharing," the CEL rule rejecting `reclaimWithinCohort: Never` with a non-`Never` `borrowWithinCohort`, `FlavorFungibility` and its `preference`, and `status.flavorsUsage[*].borrowed`.
2. **`apis/kueue/v1beta2/cohort_types.go`** — **[verified against source]**. `parentName` and its three cases (unset = root, non-existent = default cohort, existent), the cycle-disables-all-members rule, cohort-level `resourceGroups` as **additional** shared quota that may also be lent to parents, the "borrowing and lending limits must only be set when the Cohort has a parent" rule, and `fairSharing`.
3. **`apis/kueue/v1beta2/fairsharing_types.go`** — **[verified against source]**. `FairSharing.weight` (default 1, must be > 10⁻⁹ if non-zero, zero implies infinite share), and `FairSharingStatus.weightedShare` with its full definition.
4. **`pkg/cache/scheduler/fair_sharing.go`** — **[verified against source]**. The `dominantResourceShare` computation reproduced in §7: `ratio = borrowed × 1000 / lendable`, the maximum-over-resources with alphabetical tie-break, `PreciseWeightedShare = unweightedRatio / fairWeight`, `roundedWeightedShare` with its `math.Ceil` and `MaxInt64`-for-zero-weight case, `calculateLendable` walking to the root, and `CompareDRS`.
5. **`pkg/scheduler/preemption/preemption.go` and `pkg/scheduler/preemption/common/ordering.go`** — **[verified against source]**. `classicalPreemptions` with its three candidate classes (hierarchy, priority, same-queue) and the borrowing-attempt options; `fillBackWorkloads`, the reverse-order minimisation pass; `CandidatesOrdering`'s five keys reproduced verbatim in §6; `parseStrategies` and its default of `[LessThanOrEqualToFinalShare, LessThanInitialShare]`; `runFirstFsStrategy` and the highest-share-first ClusterQueue ordering.
6. **`pkg/scheduler/preemption/fairsharing/strategy.go`** — **[verified against source]**. `LessThanOrEqualToFinalShare` (rule S2-a) and `LessThanInitialShare` (rule S2-b) as one-line share comparisons, and `func Enabled(pfs *config.FairSharing) bool { return pfs != nil }` — the proof that presence, not an `enable` field, turns fair sharing on in v1beta2.
7. **`apis/config/v1beta2/configuration_types.go` and `apis/config/v1beta1/configuration_conversion.go`** — **[verified against source]**. The v1beta2 `FairSharing` struct (`preemptionStrategies` only), the v1beta1 struct (which *does* have `Enable`), and the conversion that drops the block when `enable: false` and fills the two-element default when `enable: true` with no strategies.
8. **`apis/kueue/v1beta2/workload_types.go` and `workloadpriorityclass_types.go`** — **[verified against source]**. The `Evicted` reasons (`Preempted`, `PodsReadyTimeout`, `AdmissionCheck`, `ClusterQueueStopped`, `LocalQueueStopped`, `Deactivated`, `NodeFailures`, …), the `Preempted` reasons (`InClusterQueue`, `InCohortReclamation`, `InCohortFairSharing`, `InCohortReclaimWhileBorrowing`), `Requeued`, `accumulatedPastExecutionTimeSeconds`, and `WorkloadPriorityClass` with the caveat that changing its `value` does not re-prioritise existing Workloads.
9. **`pkg/scheduler/scheduler.go`** — **[verified against source]**. `makeIterator` swapping `makeFairSharingIterator` for `makeClassicalIterator` when fair sharing is enabled — the §9 admission-side effect.
10. **`pkg/metrics/metrics.go`** — **[verified against source]**. `kueue_cluster_queue_weighted_share{cluster_queue, cohort}` with its help text, plus `kueue_cluster_queue_resource_usage`, `_nominal_quota`, `_borrowing_limit`, and `_lending_limit`.
11. **KEP-1714, "Fair Sharing"** (`keps/1714-fair-sharing/README.md`) — **[verified against source]**. The share value function and its explicit citation of DRF; the S1 (reclaim) versus S2 (rebalance) scenarios; the LCA / AlmostLCA formulation for hierarchical cohorts; the S2-a and S2-b rules; the candidate ranking criteria C1 (biggest offenders first), C2 (least important workload), C3 (smallest workload); and the `FairSharingPrioritizeNonBorrowing` per-flavor, per-level nominal-first ordering.
12. **Kueue concept docs, in-repo** (`site/content/en/docs/concepts/{cohort,cluster_queue,preemption,fair_sharing}.md`) — **[verified against source]**; the published site at https://kueue.sigs.k8s.io/docs/concepts/preemption/ is **[not reachable]**. Source for the classic-preemption candidate and target heuristics, the "must declare nominalQuota even if 0 to borrow" rule, the borrowing/lending worked examples, `lendingLimit` stable at v0.17, Fair Sharing stable at v0.7, and the **no-loop proof** reproduced in §8 including its stated open limitation for the hierarchical case.
13. **`CHANGELOG/CHANGELOG-0.11.md` and `CHANGELOG-0.19.md`** — **[verified against source]**. v0.11: "[FSxHC] Make Fair Sharing compatible with Hierarchical Cohorts during preemption / during scheduling." v0.19.1: the `flavorFungibility.preference: PreemptionOverBorrowing` fix describing quota "sourceable at a shallower borrowing level in the cohort tree."

**Theory**

14. **Ghodsi, Zaharia, Hindman, Konwinski, Shenker, Stoica — "Dominant Resource Fairness: Fair Allocation of Multiple Resource Types," USENIX NSDI 2011** — https://www.usenix.org/conference/nsdi11/dominant-resource-fairness-fair-allocation-multiple-resource-types — **[not reachable]** from this environment; cited directly by KEP-1714 (entry 11) as the basis of Kueue's value function, and that citation *was* verified. Read for the formal treatment of what "fair" means when tenants want different resource *mixes*, not just different amounts of one resource. You meet it again as Volcano's native model in [L5](05-alternatives-volcano-kai.md).

**Real-world engineering**

15. **kubernetes-sigs/kueue issue #7016**, "Clusterqueues must prefer borrowing within a cohort before borrowing across cohorts" — https://github.com/kubernetes-sigs/kueue/issues/7016 — **[title and closed state verified via GitHub this session]**. **Correction recorded:** an earlier version of this lesson described this as a live, unresolved design debate; it is now closed. The durable lesson is the one in §10 — hierarchical borrowing order is actively refined, so verify against your version rather than a remembered mental model. IBM Research's Vela/Blue Vela material (arXiv 2407.05467; KubeCon EU 2025 tutorial deck) and Red Hat's "Improve GPU utilization with Kueue in OpenShift AI" are **[not reachable]** from this environment and are listed as narrative depth only.
