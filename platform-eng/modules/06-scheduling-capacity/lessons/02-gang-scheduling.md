---
lesson: "06.2"
title: "Gang scheduling: all-or-nothing admission"
module: "06"
concept: "gang-scheduling-permit-mechanism"
status: not-started
est_time: "7h"
prev: "01-why-default-scheduler-fails.md"
next: "03-kueue-queueing-model.md"
artifacts: []
sources: 11
---

# 06.2 · Gang scheduling: all-or-nothing admission

> **Concept.** Gang (co-)scheduling makes a group of pods admit atomically — all `minMember` land together or none bind at all — by parking every member in the `Permit` phase until quorum is reachable, dissolving the L1 deadlock.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

L1 diagnosed the disease: `kube-scheduler` is a per-pod greedy binder with no cross-pod
rollback, so an 8-replica job on a 6-GPU-free cluster strands 6 GPUs holding nothing useful
while two pods pend forever. This lesson is the cure — it takes that exact deadlocked cluster
and shows the mechanism that prevents the deadlock from ever forming: atomic, all-or-nothing
admission built on the `Permit` extension point. Getting this mechanism right is also where
the module's real tension first appears: **atomicity for one job's gang is purchased, in part,
with latency for other jobs waiting behind it**, which is the seed of everything L3 onward
(queues, quotas, borrowing, fair-sharing) exists to manage.

Every mechanism claim below was read out of the `kubernetes-sigs/scheduler-plugins` tree
cloned during this session — `pkg/coscheduling/coscheduling.go`,
`pkg/coscheduling/core/core.go`, `pkg/util/podgroup.go`, `apis/scheduling/v1alpha1/types.go`,
`apis/config/types.go`, `apis/config/v1/defaults.go` — plus the `kubernetes/kubernetes`
framework contract from L1. Where a default is stated, it is the default in that code, not a
recollection.

## Why this matters

The natural follow-up to L1's interview question is "so how do you fix it?" — and "gang
scheduling" is the two-word answer, but the signal an interviewer is listening for is whether
you can name the *mechanism*: a `PodGroup` with a `minMember`, atomic admission at `Permit`, a
partial-scheduling guard at `PreFilter`, and a cross-pod rejection path at `Unreserve` that
does the thing L1 proved the default scheduler cannot do. Getting this right converts stranded
GPUs back into productive capacity.

Get `minMember` wrong in either direction and you either recreate the L1 deadlock — with a
false sense of safety, because "we run a gang scheduler" is not the same claim as "this job's
gang is configured correctly" — or you block the queue for everyone behind you. That is
exactly why CoreWeave's Principal/Staff posting frames "quota enforcement, fairness,
pre-emption" in the same breath as scheduling, and why Anthropic's JD names "gang scheduling;
topology-aware placement" as one phrase. This is not a standalone party trick; it is the first
domino in the module's whole fairness/cost chain, and the second half of this lesson is about
the bill that atomicity sends you.

## What's new here (calibration)

- **Module 02** gave you the `Permit` extension point as an abstract capability: a plugin may
  return `Wait` to hold a pod before binding, with a timeout, then later `Allow` or `Reject`
  it. Not re-derived here.
- **Module 04** gave you `nvidia.com/gpu` as a countable resource and `ResourceQuota` as a
  static, synchronous-reject cap. Assumed, not re-taught.
- **L1 (this module)** gave you the deadlock this lesson fixes, the exact ordering of the
  extension points, and the fact that `Unreserve` in the *default* plugin set is scoped to a
  single pod. That diagnosis is the premise; this lesson is the mechanism.
- **Genuinely new here:**
  - The `PodGroup` CRD in full, with every field that matters and what each one actually
    changes in the scheduler's behaviour.
  - The **five** extension points the coscheduling plugin implements — including two the
    common summary leaves out: `PostFilter`'s *optimistic rejection* with a tunable threshold,
    and the fact that the plugin's `Reserve` is a literal no-op whose only purpose is to earn
    it an `Unreserve` hook, which is where the real cross-pod rollback lives.
  - The plugin's real configuration surface (`permitWaitingTimeSeconds`,
    `podGroupBackoffSeconds`, `podGroupRejectPercentage`) with shipped defaults.
  - **The cost of atomicity, quantified**: head-of-line blocking, reservation churn, and the
    fragmentation tax gang scheduling introduces — with worked arithmetic on both sides of the
    ledger.
  - How Kueue's Workload-level admission solves the same problem one layer up, and why the two
    can be stacked.

## Core concepts

### 1. The problem restated as a constraint

L1's conclusion was that every `kube-scheduler` knob is a *preference over placements* while
the deadlock needs a *constraint on admission*. Write that constraint down precisely, because
everything else is an implementation of it:

> For a set of pods `G` with quorum `k`, **no** member of `G` may be bound unless at least `k`
> members of `G` can be placed against a single consistent view of cluster state.

Three details in that sentence do real work:

- **"at least `k`"**, not "all". The quorum is a parameter, not the group size. That is what
  lets elastic jobs express "I can start at 4 and grow to 8" (§8).
- **"against a single consistent view"**. The members are evaluated one at a time — the
  framework's scheduling cycle is serialized — so the mechanism has to *hold* earlier members'
  placements while later members are evaluated, and hold them against the same snapshot.
- **"may be bound"**. The constraint is at bind time, not at placement time. Members may be
  *placed* (a node chosen, resources assumed in the cache) and later un-placed. Only binding is
  irreversible.

The coscheduling plugin implements exactly this. So, one layer up, does Kueue (§9), and so does
the native `Gang{minCount}` path in Kubernetes v1.37 (§10). They differ in *where* they hold
the pending members and *what they hold* while waiting — and that difference is the whole
economics of the rest of this module.

### 2. The `PodGroup` object, field by field

The `scheduler-plugins` project ships a `PodGroup` CRD in group `scheduling.x-k8s.io`,
version `v1alpha1`. Here it is complete, with every field annotated:

```yaml
apiVersion: scheduling.x-k8s.io/v1alpha1
kind: PodGroup
metadata:
  name: train-gang                 # pods reference this by NAME, via a label
  namespace: research              # PodGroup is namespaced; members must be same-namespace
spec:
  # QUORUM. The number of the group's pods that must be simultaneously placeable
  # before ANY member is allowed to bind. Validated as Minimum=1 by the CRD schema.
  # This is the one field you must get right; §7 is entirely about getting it wrong.
  minMember: 8

  # OPTIONAL EARLY-EXIT GUARD. Aggregate resources the whole group needs.
  # Used ONLY by PreFilter, and only to answer "could this cluster host the gang at all,
  # ignoring who currently holds what belongs to THIS group?" — see §4 step 2.
  # If omitted, PreFilter still enforces the pod-count check but skips the resource check.
  minResources:
    nvidia.com/gpu: "8"
    cpu: "64"
    memory: 512Gi

  # HOW LONG A MEMBER WAITS AT PERMIT before the group gives up and everyone is rejected.
  # If unset, falls back to the plugin's permitWaitingTimeSeconds, which defaults to 60s
  # (pkg/util/podgroup.go: DefaultWaitTime = 60 * time.Second).
  # Sizing this is a queueing decision, not a formality — see §7(c).
  scheduleTimeoutSeconds: 300
status:                            # written by the plugin's controller, read-only for you
  phase: Pending                   # Pending | Scheduling | Running | Finished | Failed | Unknown
  running: 0
  succeeded: 0
  failed: 0
  scheduleStartTime: null
  occupiedBy: ""                   # UID of the Deployment/ReplicaSet/Job occupying the group
```

Members join by **label**, not by owner reference. The label key is
`scheduling.x-k8s.io/pod-group` (the constant `v1alpha1.PodGroupLabel`, formed as
`<group name>/pod-group`), and its value is the `PodGroup`'s `metadata.name`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: gang-demo
  namespace: research
spec:
  parallelism: 8
  completions: 8
  backoffLimit: 0
  template:
    metadata:
      labels:
        scheduling.x-k8s.io/pod-group: train-gang    # ← binds this pod to the PodGroup
    spec:
      # THE OTHER HALF OF THE WIRING. Without this the default scheduler handles the pod
      # and every guarantee below evaporates silently. This is the single most common
      # misconfiguration; see Common pitfalls.
      schedulerName: scheduler-plugins-scheduler
      restartPolicy: Never
      containers:
        - name: worker
          image: registry.k8s.io/e2e-test-images/agnhost:2.53
          args: ["pause"]
          resources:
            limits:
              nvidia.com/gpu: "1"
```

Two consequences of the label-based association that bite people:

- **The `PodGroup` and its members must be in the same namespace.** `PreFilter` and
  `PostFilter` both list pods with `podLister.Pods(pod.Namespace).List(selector)`.
- **A pod can carry the label with no `PodGroup` object present.** In that case
  `Permit` returns the status `PodGroupNotFound` and the plugin marks the pod
  `Unschedulable` — the pod does not silently fall through to normal scheduling. Conversely, a
  pod with *no* label yields `PodGroupNotSpecified` and `Permit` returns `Success` immediately,
  so unlabelled pods scheduled by this scheduler behave exactly as they would under the default
  one.

### 3. Wiring the scheduler: the config that actually matters

The plugin runs inside a `kube-scheduler` binary built from `scheduler-plugins`, usually
deployed as a **second scheduler** alongside the default one so you can opt jobs in by
`schedulerName`. Its `KubeSchedulerConfiguration`:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: true
profiles:
  - schedulerName: scheduler-plugins-scheduler
    plugins:
      # multiPoint enables the plugin at EVERY extension point it implements:
      # PreFilter, PostFilter, Permit, Reserve (and thus Unreserve).
      multiPoint:
        enabled:
          - name: Coscheduling
      # queueSort must be set explicitly and exclusively: a scheduler profile may have
      # exactly ONE queueSort plugin, so Coscheduling's Less() has to displace the default
      # PrioritySort. This is not optional — the plugin's own docs list queueSort, permit
      # and unreserve as mandatory.
      queueSort:
        enabled:
          - name: Coscheduling
        disabled:
          - name: "*"
    pluginConfig:
      - name: Coscheduling
        args:
          # Default timeout a member waits at Permit when the PodGroup doesn't set
          # scheduleTimeoutSeconds. Shipped default: 60.
          permitWaitingTimeSeconds: 60
          # After a group is rejected, refuse to even PreFilter its members for this long.
          # Shipped default: 0, i.e. the backoff is DISABLED unless you set it.
          podGroupBackoffSeconds: 0
          # PostFilter's optimistic-rejection threshold, as a percentage of minMember.
          # Shipped default: 10. See §4 step 5 — this one surprises people.
          podGroupRejectPercentage: 10
```

Those three defaults come from `apis/config/v1/defaults.go`
(`defaultPermitWaitingTimeSeconds = 60`, `defaultPodGroupBackoffSeconds = 0`,
`defaultPodGroupRejectPercentage = 10`). Committing them to memory is worth more than it
sounds: two of the three failure modes in §7 are "the default was wrong for my gang size and
nobody changed it."

The plugin also registers for cluster events so its pods are requeued promptly:
`EventsToRegister` returns `Pod: Add` and `podgroups.v1alpha1.scheduling.x-k8s.io: Add|Update`.
That is why creating the `PodGroup` *after* the pods still works — the update event wakes the
pending members.

### 4. The mechanism: what happens at each of the five extension points

This is the heart of the lesson. The plugin implements `QueueSort`, `PreFilter`, `PostFilter`,
`Permit`, and `Reserve` (the last purely to obtain `Unreserve`). Walk them in the order the
framework runs them.

```
   ONE MEMBER'S JOURNEY THROUGH THE COSCHEDULING PLUGIN
 ══════════════════════════════════════════════════════════════════════════════════════

   activeQ ──┐
             │  ① QueueSort.Less(a,b)
             │     priority DESC → PodGroup creationTimestamp ASC → ns/name ASC
             │     ⇒ members of one group land adjacent; older groups go first
             ▼
   ┌──────────────────────────────────────────────────────────────────────────────┐
   │ ② PreFilter                                                                  │
   │    a. group on the deny list (recent failure + podGroupBackoffSeconds)? ─▶ ✗ │
   │    b. count pods with this group's label.                                    │
   │       quorumGap = minMember − len(pods);  if > 0 ─▶ ✗                        │
   │       "cannot find enough sibling pods, current pods number: N,              │
   │        minMember of group: M"                                                │
   │    c. tolerate scheduling-gated siblings only while quorumGap stays ≤ 0      │
   │    d. if minResources set AND group not already in permittedPG cache:        │
   │         CheckClusterResource(all nodes, minResources + pods:minMember)       │
   │         — sums capacity across nodes, DISCOUNTING pods already belonging      │
   │           to this same group (so a partially-placed group isn't penalised)   │
   │         on success: add group to permittedPG with a scheduleTimeout TTL      │
   │    FAILURE MODE: returns UnschedulableAndUnresolvable — deliberately, so     │
   │    the framework does NOT attempt preemption for a doomed gang.              │
   └──────────────────────────────────────────────┬───────────────────────────────┘
                                                  ▼
              Filter / PostFilter / Score  (stock plugins pick a node)
                                                  │
                     ┌────────────────────────────┴──────────────────────────────┐
                     │ member fits                              member does NOT  │
                     ▼                                                     ▼
   ┌──────────────────────────────────────┐   ┌──────────────────────────────────┐
   │ ③ Reserve  →  returns nil. NO-OP.    │   │ ⑤ PostFilter (optimistic reject) │
   │    Implemented only so the plugin    │   │  assigned ≥ minMember?  → give up│
   │    gets an Unreserve callback.       │   │  gap/minMember ≤ 10%?   → give up│
   │    The actual resource hold is the   │   │  else: Reject() EVERY waiting    │
   │    FRAMEWORK's assume() from L1 §1(b)│   │  member of this group, backoff   │
   └──────────────────┬───────────────────┘   │  the group, drop it from         │
                      ▼                       │  permittedPG, mark failure       │
   ┌──────────────────────────────────────┐   └──────────────────────────────────┘
   │ ④ Permit                             │
   │   assignedPodsByPG[group].insert(pod)│
   │   n = |assignedPodsByPG[group]|      │
   │                                      │
   │   n  <  minMember  →  Wait(timeout)  │──▶ pod parks in the framework's
   │      if n == 1: ActivateSiblings()   │    waiting-pods map; its binding
   │      (push the group's other pods    │    cycle blocks in WaitOnPermit()
   │       back into activeQ so they get  │    with its node resources ALREADY
   │       attempted immediately)         │    assumed in the scheduler cache
   │                                      │
   │   n >= minMember  →  Success         │──▶ IterateOverWaitingPods:
   │                                      │    Allow() every member of the group
   └──────────────────────────────────────┘    ⇒ the whole gang enters binding
                      │
                      │ timeout fires, or bind fails, or PostFilter rejects
                      ▼
   ┌──────────────────────────────────────────────────────────────────────────────┐
   │ ⑥ Unreserve  ◀── THE PART THE SUMMARIES LEAVE OUT                            │
   │    pgMgr.Unreserve(pod)            drop this pod from assignedPodsByPG        │
   │    IterateOverWaitingPods:         Reject() EVERY OTHER waiting member of     │
   │                                    the same group — "rejection in Unreserve"  │
   │    DeletePermittedPodGroup()       clear the PreFilter resource-check cache   │
   │    MarkPodGroupScheduleFailure()   so PreFilter can deny-list the group       │
   │                                                                              │
   │  ⇒ this is the CROSS-POD rollback L1 proved the default plugin set lacks.    │
   │    One member's failure un-places every other member. Nothing is bound.      │
   └──────────────────────────────────────────────────────────────────────────────┘
```

Now the parts that deserve prose.

**① `QueueSort.Less` is not just "keep siblings together".** It compares, in order: pod
priority (descending), then a *group* timestamp, then `namespace/name` as a deterministic
tiebreak. The timestamp is the `PodGroup`'s `creationTimestamp` when the pod belongs to a
group, and the pod's *initial scheduling-attempt timestamp* otherwise. Two consequences worth
knowing: an older `PodGroup` outranks a newer one even if the newer group's pods were created
first, and a group whose pods are re-created (say, by a Job retry) does **not** lose its place,
because the group object's age is what counts. This is the plugin's entire fairness story, and
it is FIFO-by-group — which is precisely why L3 exists.

**② `PreFilter` is the partial-scheduling guard, and it refuses preemption on purpose.** The
pod-count check is cheap and catches the common case: you asked for `minMember: 8` but only 6
pods exist, so the group can never reach quorum and every scheduling cycle spent on it is
waste. The `minResources` check is the expensive one, and its implementation is worth
understanding: `CheckClusterResource` walks every node, subtracting each node's *free*
resources from the requested aggregate until the remainder is empty. Critically,
`getNodeResource` first removes from its node snapshot any pods **belonging to this same
group**, so a group that already has three members placed is measured against a cluster that
still counts those three members' resources as available to it. Without that, a
partially-placed group would fail its own feasibility check.

The result of a successful `minResources` check is cached in `permittedPG` with a TTL equal to
the schedule timeout, so the O(nodes) scan runs once per group per timeout window rather than
once per member.

And note the return code: `fwk.UnschedulableAndUnresolvable`, chosen explicitly — the source
comment says "to avoid any preemption attempts". A gang that cannot fit is not a gang whose
problem preemption can solve, so the plugin declines to let `PostFilter`'s preemption machinery
churn on it.

**③ `Reserve` returns `nil`.** This is the detail almost every summary of gang scheduling gets
wrong, including the previous version of this lesson. The plugin does **not** implement a
resource hold at `Reserve`. The hold is the framework's own `assume()` (L1 §1(b)), which marks
the pod as consuming its chosen node's resources in the scheduler cache before `Permit` runs.
The plugin implements the `ReservePlugin` interface solely because that is the only way to be
given the `Unreserve` callback. So when you say "the gang holds resources while waiting", be
precise: **the framework holds them, in its in-memory cache, because the pod was assumed; the
plugin's contribution is deciding when to let go.**

**④ `Permit` is the atomic gate.** `pgMgr.Permit` inserts the pod into a per-group set
`assignedPodsByPG` under a lock and compares the set's size to `minMember`. Below quorum it
returns `Wait`, and the plugin translates that into `fwk.NewStatus(fwk.Wait)` plus a wait
duration resolved by `GetWaitTimeDuration`: the `PodGroup`'s `scheduleTimeoutSeconds` if set,
else the plugin's configured `permitWaitingTimeSeconds`, else the hard-coded 60s
`DefaultWaitTime`. At quorum it returns `Success`, and the plugin then walks the framework's
waiting-pods map and calls `Allow()` on **every** member of the group at once. All of them
proceed into their binding cycles together.

There is one more subtlety here that explains a behaviour you will observe. When the *first*
member enters `Wait` (`len(assigned) == 1`), the plugin sets an `Activate` flag in cycle state,
and `ActivateSiblings` then pushes every other pod of the group into the framework's
`PodsToActivate` map — which the scheduler drains at the end of a successful scheduling cycle,
moving those pods from `backoffQ`/`unschedulablePods` straight back into `activeQ`. This is why
a gang assembles in a burst rather than dribbling in on 10-second backoffs: the first member to
park deliberately wakes its siblings. The plugin does this only for the first member, on
purpose — doing it unconditionally would be a requeue storm.

**⑤ `PostFilter` performs *optimistic rejection*, and this is where the tunable nobody knows
about lives.** When a member fails `Filter` on every node, `PostFilter` runs. Instead of blindly
tearing the group down, it applies two escapes:

- If `assigned >= minMember` already, do nothing — quorum has been met by other members and
  this straggler's failure does not matter.
- Compute `notAssignedPercentage = (minMember − assigned) / minMember`. If that is **≤
  `podGroupRejectPercentage / 100`** (default **10%**), do nothing either: the gap is small
  enough that it is worth letting subsequent pods try rather than throwing away the work
  already done.

Only if the gap is larger does it reject every waiting member of the group, optionally apply
`podGroupBackoffSeconds`, drop the group from `permittedPG`, and mark a schedule failure so
`PreFilter` deny-lists it.

Read that threshold carefully, because it has a real consequence: **with the default 10% and a
`minMember` of 8, a gap of 1 pod is 12.5% > 10%, so the group is rejected; with `minMember` of
16, a gap of 1 pod is 6.25% ≤ 10%, so it is not.** The plugin's willingness to hold a partially
assembled gang therefore depends on gang size. For large gangs this is a feature (avoid
throwing away 60 placements because the 61st is momentarily unlucky); if you want strict
behaviour, set `podGroupRejectPercentage: 0` and any failure tears the group down.

**⑥ `Unreserve` is the cross-pod rollback.** When a waiting member's timeout expires, or its
bind fails, or `PostFilter` rejected it, the framework runs `Unreserve` for that pod. The
plugin's implementation removes the pod from the group's assigned set and then — the key line —
iterates the waiting-pods map and calls `Reject()` on **every other waiting member of the same
group**. Rejection causes each of their blocked binding cycles to fail, which in turn triggers
*their* `Unreserve`, which forgets their assumed placements from the scheduler cache. The GPUs
those members were holding become available again, and nothing was ever bound.

That is the mechanism, and it is worth stating against L1's finding explicitly. L1: *"There is
no code path in `pkg/scheduler`, with the default plugin set, in which pod A's failure causes
pod B's reservation to be undone."* The coscheduling plugin adds exactly that code path, using
the one framework facility that permits it — `IterateOverWaitingPods` + `Reject` — which exists
precisely so that `Permit` plugins can coordinate across pods.

### 5. Where the guarantee lives: admission, not binding

Binding is inherently per-pod. The kubelet learns about one `Pod`→`Node` assignment at a time,
via one `POST` to the binding subresource. Nothing in gang scheduling changes that, and no
implementation could without changing the pod API.

What is atomic is the **decision to admit the group**, enforced at `Permit`, before any member
enters its binding cycle. So the correct one-liner, and the one to say in an interview:

> Gang scheduling is enforced at **admission**, at the `Permit` extension point. Binding
> remains per-pod, but no member binds until the whole group is cleared.

The observable consequence is the invariant that makes this whole lesson worth 7 hours:

> **The group is either fully admitted or fully pending — never partial.**

### 6. The before/after, with the exact observable difference

Same cluster as L1: 6 free GPUs, an 8-replica job. Now with `minMember: 8`.

```
  WITHOUT GANG (L1)                          WITH GANG (minMember: 8)
  ══════════════════════════════════         ══════════════════════════════════════════

  t+0.00  r0 Filter ok → assume → BIND       t+0.00  PreFilter: 8 pods exist ✓
  t+0.01  r1 → BIND                                  r0 Filter ok → assume
  t+0.02  r2 → BIND                                  Permit: n=1 < 8 → WAIT (60s)
  t+0.03  r3 → BIND                                  ActivateSiblings() → r1..r7 to activeQ
  t+0.04  r4 → BIND                          t+0.01  r1 → assume → Permit n=2 → WAIT
  t+0.05  r5 → BIND     free GPUs: 6→0        …
  t+0.06  r6 Filter FAILS → Pending          t+0.05  r5 → assume → Permit n=6 → WAIT
  t+0.07  r7 Filter FAILS → Pending          t+0.06  r6 Filter FAILS on all nodes
                                                     PostFilter: assigned=6, minMember=8
  STEADY STATE                                       gap = 2/8 = 25% > 10% → REJECT ALL
  ┌────────────────────────────────┐                 → Unreserve on each waiting member
  │ 6 pods Running, 0 computing    │                 → 6 assumed placements forgotten
  │ 2 pods Pending forever         │         t+0.07  free GPUs: back to 6
  │ 6 GPUs stranded, unusable      │
  │ by ANY other job               │         STEADY STATE
  │ resolves at: NCCL timeout      │         ┌──────────────────────────────────────┐
  └────────────────────────────────┘         │ 8 pods Pending                       │
                                             │ 0 GPUs consumed by this job          │
                                             │ 6 free GPUs available to OTHER jobs  │
                                             │ resolves at: 2 more GPUs free up     │
                                             └──────────────────────────────────────┘

  Two more GPUs free (a job finishes):
                                             t+T     PodGroup Update / Pod Add event
                                                     requeues all 8 members
                                                     r0..r6 → assume → Permit WAIT (n=1..7)
                                                     r7     → assume → Permit n=8 = minMember
                                                             → Success
                                                             → Allow() r0..r6 and r7
                                                     all 8 enter binding cycles together
```

The difference to point at in a capture is not "the pods are Pending instead of Running" — it
is **the free-GPU count**. Under L1, six GPUs were consumed and unusable by anyone. Under gang
admission, those six remained available to any other workload the whole time. That is the
entire economic argument, and it is also the setup for the argument against, which is next.

### 7. The failure modes, mechanically

Gang scheduling is only as correct as its configuration. Four ways to get it wrong, each with
its mechanism and its symptom.

**(a) `minMember` too low — the L1 deadlock, restored.** Set `minMember: 6` for an 8-replica
NCCL job. Quorum is reached at 6, `Permit` returns `Success`, six pods bind, and they block in
`init_process_group` waiting for ranks the gang never guaranteed. You have reproduced L1
exactly, with a gang scheduler installed and a false sense of safety.

The symptom is indistinguishable from L1: some pods `Running` with zero SM activity, some
`Pending`. The diagnostic that distinguishes it is
`kubectl get podgroup -o custom-columns=NAME:.metadata.name,MIN:.spec.minMember` compared
against `kubectl get job -o jsonpath='{.spec.parallelism}'`. If they differ and the job has a
hard collective barrier, that is your bug.

**(b) `minMember` too high — never admits.** Set `minMember: 9` for 8 replicas. `PreFilter`
computes `quorumGap = 9 − 8 = 1 > 0` and rejects every member on every attempt with `cannot
find enough sibling pods, current pods number: 8, minMember of group: 9`. Deadlock-free and
progress-free. The symptom is distinctive and easy to spot in `kubectl describe pod` — you get
a *specific* message naming both numbers, unlike the generic `Insufficient nvidia.com/gpu` of
case (a).

**(c) `scheduleTimeoutSeconds` too short — reservation churn.** The default is 60 seconds.
Consider a `minMember: 32` gang on a busy cluster where GPUs free up at a rate of roughly one
per five minutes. Assembling 32 GPUs takes on the order of 160 minutes; the members that park
first will time out at 60 seconds, get rejected via `Unreserve`, drop their assumed placements,
and be requeued. Then the cycle repeats. The gang grabs fragments, holds them for a minute,
releases them, and re-grabs — burning scheduling throughput and, worse, repeatedly making
capacity briefly unavailable to other jobs without ever making progress itself.

The fix is to size `scheduleTimeoutSeconds` to the realistic time for `minMember` units of
capacity to become free, which is a queueing question you should answer with data
(`kueue_quota_reserved_wait_time_seconds` in L3, or just the historical distribution of your
job durations). The symptom is a `PodGroup` whose members cycle between `Pending` and
briefly-assumed, with repeated `rejection in Unreserve` log lines from the scheduler.

**(d) Head-of-line blocking — the cost of atomicity.** This is not a misconfiguration; it is
the mechanism working as designed, and it is the most important of the four because it
motivates the entire rest of the module.

While a gang is assembling, each member that has passed `Permit` and is parked in `Wait` **has
already been assumed in the scheduler cache**. Its GPU is accounted as consumed. It is not
running anything. And no other job can use it, because `Filter` for any other pod sees the
resource as taken.

So a large gang that cannot yet reach quorum converts free capacity into *held-but-idle*
capacity for the duration of its wait — which is precisely the condition L1 called stranded
capacity. Gang scheduling has not eliminated stranding; it has **bounded** it, by
`scheduleTimeoutSeconds`, and made it recoverable rather than permanent. That is a genuine and
large improvement. It is not the same as zero.

```
  CONVOY EFFECT — a minMember:32 gang assembling while small jobs queue behind it

  GPUs freeing at 1 per 5 min.  Gang parks each one at Permit as it appears.

  t=0            [........................................]  0 held,  8 free  ← small jobs OK
  t=25m   gang:  [#####...................................]  5 held,  3 free
  t=50m   gang:  [##########..............................] 10 held,  0 free  ← small jobs BLOCKED
  t=100m  gang:  [####################....................] 20 held, (new capacity absorbed)
  t=160m  gang:  [################################........] 32 held → QUORUM → all Allow()
                                                             ↑
                        for 110 of those 160 minutes, 10–32 GPUs were held and idle,
                        and unavailable to the 1-GPU jobs sitting in the queue

  # = held at Permit, running nothing.   With scheduleTimeoutSeconds: 60 (the default),
  # this ramp never completes: members time out and release long before quorum.
```

§11 puts numbers on this. The fix is not to abandon gang scheduling; it is to add a queueing
layer that decides *whether the gang should be accumulating at all* against a quota model,
rather than letting it grab physical fragments on a first-come-first-served basis. That layer
is Kueue, and it is L3.

### 8. Elasticity is a different axis from `minMember`

Not every distributed job has a rigid replica count. PyTorch's elastic launcher accepts a
range — `torchrun --nnodes=4:8` means the job can start with 4 nodes, scale to 8 as capacity
frees, and tolerate worker churn mid-run via its own rendezvous and restart logic.

`minMember` is a single integer. So an elastic job forces an explicit operator decision:

| Choice | `minMember` | Starts | Throughput | Risk |
|---|---|---|---|---|
| Minimum viable | 4 | as soon as 4 GPUs are free | starts at half scale, may grow | may run the whole job at 4 if capacity never frees |
| Target size | 8 | only when 8 GPUs are free | full scale from step 0 | longer queue wait; larger held-but-idle window while assembling |

There is no universally correct answer — it depends on whether the workload benefits more from
starting sooner at reduced parallelism or from guaranteed full-scale throughput. What *is*
universally true is that conflating the two is a design error, because they answer different
questions: **elasticity asks "how small can this job tolerate running?", `minMember` asks "how
many pods must be present simultaneously for admission to be safe?"** For a rigid MPI job with
a fixed communicator, the two coincide and the decision disappears. For an elastic job they do
not, and the scheduler cannot infer your preference.

Kueue has a first-class answer to this shape (elastic workloads, where an admitted Workload's
pod-set count can change), which is one of the reasons it is the production choice — but the
underlying tradeoff is unchanged.

### 9. Kueue's take: the same guarantee, one layer up

Kueue (L3–L4) solves the same constraint from §1 at a different altitude, and understanding
the difference now makes L3 much easier.

```
  TWO PLACES TO ENFORCE "ALL OR NOTHING"
  ══════════════════════════════════════════════════════════════════════════════════

  COSCHEDULING PLUGIN                        KUEUE
  (inside kube-scheduler)                    (a controller ABOVE kube-scheduler)

  Job created                                Job created (labelled with a LocalQueue)
      │                                          │
      │ pods created immediately                 │ mutating webhook forces spec.suspend=true
      ▼                                          ▼   ⇒ ZERO pods created
  8 pods enter activeQ                       Kueue creates a Workload object summing
      │                                      all pod sets: 8 × 1 GPU = 8 GPUs
      ▼                                          │
  each runs a full scheduling cycle              ▼
  Filter/Score pick a REAL NODE              Kueue's scheduler checks the aggregate ask
      │                                      against the ClusterQueue's QUOTA
      ▼                                          │
  Permit parks the pod                           ├─ fits → reserve quota, inject the
  ⇒ HOLDS A PHYSICAL GPU on a                    │        flavor's nodeSelector/tolerations,
     specific node while waiting                 │        set spec.suspend=false
      │                                          │        ⇒ NOW the 8 pods are created and
      ▼                                          │          kube-scheduler binds them
  quorum → Allow() all → bind                    │
                                                 └─ doesn't fit → Workload waits with
  HELD WHILE WAITING:                                       QuotaReserved=False
    real GPUs on real nodes                       HELD WHILE WAITING:
                                                    nothing. Zero pods. Zero GPUs.
```

The difference is not cosmetic. **A gang waiting at `Permit` holds physical GPUs; a Workload
waiting for quota holds nothing.** That single property is why the head-of-line blocking in
§7(d) is a serious problem for raw coscheduling and a much smaller one for Kueue, and it is why
the deliverable for this module is Kueue-based.

Kueue's atomicity is a consequence of how it accounts: by default a Workload is unsuspended
only when quota for **all** of its pod sets can be reserved at once. Kueue does not admit a
fraction of a Workload unless you explicitly enable partial admission.

The two compose. Kueue's quota check is aggregate — it knows the cluster has 8 GPUs free, but
not that they are split 4+4 across two nodes when a pod needs 8 on one node. Kueue's own
documentation is explicit that aggregate quota is blind to node layout, and recommends
Topology-Aware Scheduling (L6) as the primary fix, with `waitForPodsReady` as a safety net that
evicts and requeues a workload whose pods do not all become ready in time. Layering
coscheduling underneath Kueue is another option: Kueue admits the Workload against quota,
coscheduling guarantees the pods land atomically. For a homogeneous gang on a well-shaped
cluster, Kueue's Workload admission is usually enough on its own.

### 10. The native successor: `Gang{minCount}` in core Kubernetes

L1 §11 introduced Kubernetes' `Workload`/`PodGroup` API. It is worth being precise about how it
relates to *this* lesson, because it generalises exactly the mechanism you just learned.

Verified against the `kubernetes/kubernetes` tree at the v1.37 development head:

- The policy field is `PodGroupSpec.schedulingPolicy`, a union of `Basic{}` and
  **`Gang{ minCount }`**. `minCount` is the direct analogue of `minMember` — "the minimum
  number of pods that must be schedulable or scheduled at the same time for the scheduler to
  admit the entire group". Unlike `minMember`, it is explicitly **mutable**, to support
  workload scaling, with the caveat that the scheduler is eventually consistent and a mid-cycle
  change may not apply until the next cycle.
- Pods opt in with `spec.schedulingGroup.podGroupName` instead of a label.
- The `GenericWorkload` feature gate is **Beta on v1.37 but still `Default: false`**.

The mechanism differs in one important way, and it is the way that matters for §7(d). The
plugin's approach is *place-then-wait*: each member gets a real node and a real assumed
resource hold, and the group waits with capacity in hand. The native path in
`pkg/scheduler/schedule_one_podgroup.go` is *simulate-then-commit-or-revert*: the scheduler
evaluates the whole group against one snapshot, maintaining an explicit stack of `revertFns`
that undo assumed pods and `Reserve` calls, invoked on failure or after each candidate
placement is evaluated. A group that cannot be placed is rolled back within the cycle rather
than parked holding resources for up to `scheduleTimeoutSeconds`.

So the direction of travel for an interview answer is: **no, gang scheduling will not always
require a third-party plugin — the primitive is standardising into core Kubernetes, and the
native implementation is architecturally better than the plugin at the specific point where the
plugin hurts. But the fairness/quota/cohort economics on top of admission — Kueue's actual
value-add — is not part of that KEP and is not going away.**

### 11. Historical grounding: this is a solved HPC problem

Gang scheduling is a specific instance of the classical **all-or-nothing atomic admission
control** problem from parallel and distributed systems, studied since the late 1980s and early
1990s for parallel supercomputers; Ousterhout's coscheduling work and the Feitelson & Rudolph
gang-scheduling literature are the canonical starting points. Slurm, PBS, and LSF have shipped
some form of it for decades — in Slurm, a job's node allocation is inherently all-or-nothing
because the allocation *is* the scheduling unit.

That framing is worth naming explicitly in an interview for two reasons. First, it is
historically accurate, and treating gang scheduling as a novel Kubernetes trick reads as
inexperience. Second, it explains *why* Kubernetes got here late: Kubernetes' scheduling unit
was designed as the pod, for services, where per-pod independence is a feature. Batch/HPC
workloads have the opposite requirement, and the last several years of Kubernetes scheduling
work — `scheduler-plugins`, Volcano, Kueue, KAI, and now the native `Workload` API — are
Kubernetes absorbing a solved HPC problem into a cloud-native substrate.

## Perspectives

**Developer.** From inside the training script, gang scheduling is invisible when it works:
`init_process_group` either starts promptly with every rank present, or the Job creates no
running pods at all. That second state is a *clean signal* — "waiting for capacity" instead of
"hung" — and converting an undiagnosable hang into a legible wait is a genuine
developer-experience win worth naming explicitly. The developer-visible cost is that
`kubectl get pods` showing 8/8 `Pending` looks alarming the first time, and the honest platform
answer is documentation plus a `kubectl describe podgroup` that says what it is waiting for.

**Operator.** Two decisions the operator owns and cannot delegate: `minMember` (which requires
knowing the job's true barrier size, and which for elastic frameworks is a policy choice, not a
fact) and `scheduleTimeoutSeconds` (which requires knowing the capacity-arrival rate on the
fleet). Both are per-workload, both have asymmetric failure modes, and neither has a safe
default that works across a mixed fleet. That asymmetry — a knob whose correct value depends on
data the platform has and the submitter does not — is a strong argument for the platform
providing job templates rather than letting each team hand-write `PodGroup`s.

**Failure-mode/queueing.** `scheduleTimeoutSeconds` sizing is a queueing-theory problem in
miniature. Too short and the gang thrashes: grab, hold 60s, release, regrab, never converge.
Too long and a doomed gang camps on assumed fragments and starves everything behind it. There
is no setting that is right for both a 4-pod gang on an idle cluster and a 64-pod gang on a
busy one, which means the mechanism needs a layer above it that reasons about the *queue*, not
just the group. That is not a coincidence; it is why L3 exists.

**Theory.** The `Permit`-based mechanism is a two-phase-commit-shaped protocol with a timeout
as the abort trigger: each participant votes by parking in `Wait`, the coordinator (the
plugin's shared `assignedPodsByPG` map) counts votes, and either commits all or aborts all via
`Unreserve`. Seeing it that way makes the failure modes predictable — the classic 2PC problems
are exactly the ones that appear here: a slow participant blocks the whole transaction
(head-of-line blocking), and resources are held during the voting window (the convoy effect).

## Real-world use cases

- **kubernetes-sigs/scheduler-plugins — the coscheduling plugin itself.** What it shows: the
  reference implementation of everything in §4, plus the maintainers' own before/after demo. A
  `ReplicaSet` of 6 nginx pods on a cluster that fits 3: with `minMember: 3` you get 3 Running
  and 3 Pending (partial placement, the L1 pattern); raise `minMember` to 4 and **all six** go
  Pending with zero resources consumed. The README also states plainly that `queueSort`,
  `permit` and `unreserve` are mandatory, that `preFilter` is an optional early-exit
  optimisation "especially valuable for larger `minMember` configurations", and that pods in
  one `PodGroup` should share a priority because mixed priorities within a group "might lead to
  unintended behavior" — a direct consequence of `Less()` sorting on priority first. Maturity
  is self-declared **Beta**. **Cloned and read directly this session.**

- **OpenAI — "Scaling Kubernetes to 7,500 Nodes"** —
  https://openai.com/index/scaling-kubernetes-to-7500-nodes/. What it shows: a named gang
  scheduling plugin running in production at 7,500-node scale, adopted specifically because
  their MPI-based jobs halt entirely if any single participating pod dies or fails to start.
  This is the mechanism this lesson teaches, deployed, not the problem L1 describes.
  *(Search-verified this session across multiple independent citations; direct fetch blocked by
  this environment's egress proxy.)*

- **Alibaba Cloud ACK — gang scheduling as a GA managed feature** —
  https://www.alibabacloud.com/help/en/ack/ack-managed-and-ack-dedicated/user-guide/work-with-gang-scheduling.
  What it shows: a major cloud vendor shipping gang scheduling as a first-class, generally
  available feature of its managed Kubernetes, implemented via a `PodGroup`-style resource with
  a minimum-member field built on the same kube-scheduler framework extension points. Evidence
  that this is mainstream production practice rather than a niche pattern, and a second config
  dialect to compare against `scheduler-plugins`. *(Search-verified; fetch blocked by egress
  this session.)*

- **Kueue's own "All-or-nothing Scheduling" documentation.** What it shows: the production
  alternative's explicit position, in the maintainers' words — quota-based admission is "the
  first line of defense" because a Workload is unsuspended only when quota for all its pod sets
  can be reserved at once; but "reserving aggregate quota alone does not guarantee that Pods can
  actually schedule", because quota tracks totals while scheduling depends on node layout and
  fragmentation. Their worked counterexample is exact: a cluster with 8 GPUs free split 4+4
  across two nodes will admit a Workload containing a single 8-GPU pod on aggregate quota, and
  that pod then sticks at the kube-scheduler layer forever. Their recommended ordering is
  quota → Topology-Aware Scheduling → ProvisioningRequest → `waitForPodsReady` as a safety net.
  **Read directly from the cloned `kubernetes-sigs/kueue` repository this session**
  (`site/content/en/docs/concepts/all_or_nothing.md`), since the docs site was unreachable from
  this environment.

## Worked example

**Part 1 — the before/after capture.** Same fake-GPU kind cluster as L1 (three workers, one
`nvidia.com/gpu` each), same 4-replica job, now with the coscheduling scheduler and a
`PodGroup` with `minMember: 4`. Only 3 GPUs are free, so quorum is unreachable.

```bash
$ kubectl apply -f podgroup.yaml
podgroup.scheduling.x-k8s.io/train-gang created

$ kubectl get podgroup train-gang \
    -o custom-columns=NAME:.metadata.name,MIN:.spec.minMember,PHASE:.status.phase
NAME         MIN   PHASE
train-gang   4     Pending

$ kubectl apply -f gang4.yaml     # 4 pods, labelled, schedulerName: scheduler-plugins-scheduler
$ kubectl get pods -l batch.kubernetes.io/job-name=gang-demo
NAME              READY   STATUS    RESTARTS   AGE
gang-demo-p1w2x   0/1     Pending   0          12s
gang-demo-p3q4r   0/1     Pending   0          12s
gang-demo-p5t6y   0/1     Pending   0          12s
gang-demo-p7u8i   0/1     Pending   0          12s
```

```bash
$ kubectl describe pod gang-demo-p7u8i | sed -n '/Events/,$p'
Events:
  Type     Reason            Age   From                          Message
  ----     ------            ----  ----                          -------
  Warning  FailedScheduling  11s   scheduler-plugins-scheduler   0/4 nodes are available:
           1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: },
           3 Insufficient nvidia.com/gpu.
  Warning  FailedScheduling  9s    scheduler-plugins-scheduler   PodGroup research/train-gang
           gets rejected due to Pod gang-demo-p7u8i is unschedulable even after PostFilter
```

*(Representative transcript; the wording of the `PostFilter` message is verbatim from the
plugin's source format string, the `Insufficient` line is the stock `NodeResourcesFit` verdict.)*

Read the two events together, because they are the mechanism:

1. The fourth member fails `Filter` on every node — three nodes are out of GPUs.
2. `PostFilter` computes `assigned = 3`, `minMember = 4`, so
   `notAssignedPercentage = 1/4 = 25% > 10%` (the default `podGroupRejectPercentage`) — above
   the optimistic-rejection threshold, so it **rejects the whole group**. Each rejection
   triggers `Unreserve`, which forgets the three assumed placements.

**The critical diff versus L1:** in L1, 3 pods were `Running` and holding GPUs while one
pended — 3 GPUs stranded. Here, all 4 pods are `Pending` and **0 GPUs are consumed**. Confirm
it, because this is the whole point:

```bash
$ kubectl get pods -A -o json | jq -r '
    [ .items[] | select(.spec.nodeName != null)
      | .spec.containers[].resources.limits["nvidia.com/gpu"] // "0" | tonumber ]
    | add | "committed GPUs: \(.)"'
committed GPUs: 0
```

Now free a fourth GPU — add a worker, or patch an existing node's capacity to 2 — and watch
quorum flip:

```bash
$ kubectl get pods -l batch.kubernetes.io/job-name=gang-demo -o wide
NAME              READY   STATUS    NODE
gang-demo-p1w2x   1/1     Running   sched-lab-worker
gang-demo-p3q4r   1/1     Running   sched-lab-worker2
gang-demo-p5t6y   1/1     Running   sched-lab-worker3
gang-demo-p7u8i   1/1     Running   sched-lab-worker4
```

All four transitioned together, in one `Permit` batch: the fourth member's `Permit` returned
`Success`, and the plugin's `IterateOverWaitingPods` loop called `Allow()` on the three parked
members plus itself. Capture both states — all-pending, then all-running — alongside the L1
capture. That trio is the deliverable's gang-scheduling demo.

**Part 2 — the convoy effect, priced.** Now quantify §7(d) so the tradeoff is a number rather
than an adjective.

*Setup.* A `minMember: 32` gang. The cluster frees GPUs at an average of one per 5 minutes as
other jobs complete. `scheduleTimeoutSeconds: 10800` (3 hours — deliberately long enough that
the gang does not thrash). Behind it, a steady stream of 1-GPU jobs that would each have used a
freed GPU immediately.

*Assumption to state plainly:* the plugin parks each member at `Permit` as soon as a GPU
becomes available to it, and that member's assumed placement makes the GPU unavailable to
anyone else. This is the mechanism from §4③–④, so it follows from the implementation — but the
*rate* of one GPU per 5 minutes is illustrative, chosen to make the arithmetic legible.

```
  time to reach 32 GPUs at 1 per 5 min = 32 × 5              = 160 minutes
  GPUs held rises linearly 0 → 32 over that window
  average GPUs held during the ramp    = (0 + 32) / 2        = 16
  GPU-minutes held-but-idle            = 16 × 160            = 2,560 GPU-minutes
                                                             = 42.7 GPU-hours
  cost of that idle time @ $2.35/GPU-hr = 42.7 × 2.35        = $100.30
```

That is **~43 GPU-hours** of capacity denied to the 1-GPU jobs during the ramp. Note carefully
what it is *not*: it is not waste in the L1 sense, because it is bounded and recoverable — if
the gang times out, every one of those GPUs is released within `scheduleTimeoutSeconds`. But it
is real opportunity cost, and it is charged to the jobs behind the gang, not to the gang.

*Now the other side of the ledger.* What did atomicity buy? Under L1's per-pod binder, the same
32-replica job on the same cluster would have bound members as GPUs appeared and deadlocked at
the first shortfall, holding whatever it had accumulated **until the NCCL rendezvous timeout**
(a PyTorch default on the order of 30 minutes) and then crash-looping under `backoffLimit`:

```
  DEADLOCK PATH (no gang)
    GPUs held at deadlock        ≈ 31   (all but the last)
    hold duration per attempt    = 30 min (rendezvous timeout)
    attempts (backoffLimit 6 + 1)= 7
    stranded GPU-hours           = 31 × 0.5 × 7            = 108.5 GPU-hours
    cost @ $2.35                                            = $254.98
    training steps produced                                 = 0
    and: the deadlock also blocked the same 1-GPU jobs, for the same reason

  GANG PATH
    held-but-idle during ramp                               = 42.7 GPU-hours
    cost @ $2.35                                            = $100.30
    training steps produced                                 = the whole job runs
```

**Gang scheduling is ~2.5× cheaper in stranded GPU-hours here *and* actually produces the
result.** State the comparison that way rather than claiming gang scheduling is free — the
honest claim is that it trades an unbounded, unproductive hold for a bounded, productive one.

**Part 3 — the fragmentation tax.** There is a second cost, distinct from the convoy effect,
that is easy to miss. Atomic admission means a gang cannot be split across time, so a cluster
serving gangs needs *contiguous* free capacity in a way a cluster serving independent pods does
not.

Model it simply. A 64-GPU fleet, average utilisation 85%, so ~9.6 GPUs free at any instant, but
scattered — say, as fragments of average size 2.4 GPUs across 4 nodes.

```
  free capacity                     = 64 × 0.15         = 9.6 GPUs
  a minMember:8 gang needs           8 GPUs simultaneously
  probability the fragments coalesce to ≥8 at a given instant: high (9.6 > 8)
  a minMember:16 gang needs          16 GPUs simultaneously
  ⇒ it must WAIT for utilisation to drop below 75%, no matter how many
    1-GPU jobs it could have displaced

  effective capacity for 16-GPU gangs  =  the fleet, only when util ≤ 75%
  effective capacity for 1-GPU jobs    =  the fleet, essentially always
```

The gang's *effective* fleet is smaller than the fleet. That is the fragmentation tax, and L7
turns this sketch into the real math. For now, hold two things: the tax is real, it grows with
`minMember`, and it is paid in queue latency rather than in dollars — which is exactly why it
is easy to under-manage and why the queueing layer in L3 is where a platform actually controls
it.

## Practice

Continue on the L1 **kind cluster with fake `nvidia.com/gpu`**.

1. **Install a gang-aware scheduler.** Deploy `scheduler-plugins` as a **second** scheduler
   (its Helm chart or manifests install the `PodGroup` CRD, the RBAC, and a deployment running
   `kube-scheduler` built with the plugin). Verify the profile: the pod's
   `KubeSchedulerConfiguration` must have `Coscheduling` enabled under `multiPoint` **and**
   as the sole `queueSort` plugin. `kubectl -n <ns> get cm <scheduler-config> -o yaml` and read
   it — do not assume the chart defaults match §3.

2. **Create the `PodGroup`** with `minMember: 4` and `scheduleTimeoutSeconds: 300`. Confirm the
   CRD is served: `kubectl get podgroups.scheduling.x-k8s.io -A`.

3. **Re-submit the exact L1 4-replica job**, now with the `scheduling.x-k8s.io/pod-group` label
   and `schedulerName: scheduler-plugins-scheduler`, on the cluster with only **3 free GPUs**.
   Confirm **all 4 pods `Pending`, 0 GPUs committed** — capture `get pods`, the committed-GPU
   `jq` count, and the `PostFilter` rejection event.

4. **Free a 4th GPU** (add a worker or patch a node's capacity to 2). Confirm all 4 flip to
   `Running` **together**, in a single transition rather than staggered. Capture.

5. **Diff against L1** and annotate the cost: L1 stranded 3 GPUs (`3 × $2.35 = $7.05/hr`,
   substituting your own rate); the gang stranded **zero**, and the 3 free GPUs stayed usable by
   other work until quorum was reachable.

6. **Break it deliberately — `minMember` too low.** Set `minMember: 3` for the 4-replica job
   and observe the L1 deadlock reappear *with the gang scheduler installed*: 3 Running, 1
   Pending. This is the single most valuable five minutes in the lesson, because it converts
   "we have gang scheduling" from a claim into a configuration you have to verify.

7. **Break it deliberately — `minMember` too high.** Set `minMember: 5`. Capture the distinct
   error text (`cannot find enough sibling pods, current pods number: 4, minMember of group: 5`)
   and note that it names both numbers — unlike case (6), which is diagnostically silent.

8. **Stretch — observe the optimistic-rejection threshold.** With `minMember: 4` and 3 GPUs
   free, the gap is 25% and the group is rejected. Set `podGroupRejectPercentage: 30` in the
   plugin args, restart the scheduler, and re-run: the gap is now *below* the threshold, so
   `PostFilter` declines to tear the group down and the three placed members stay parked at
   `Permit` until the timeout. Capture both behaviours and note which one you would want on a
   shared fleet, and why.

**Acceptance:** a before/after capture proving the L1 deadlock is fixed — 4-pending/0-committed
→ 4-running-together — with the committed-GPU count as evidence (not just pod phase), plus at
least one deliberately-broken configuration from (6) or (7) with its distinguishing error text.
This pairs with the L1 capture as the deliverable's complete gang-scheduling demo.

## Common pitfalls

- **Forgetting `schedulerName`.** The `PodGroup` exists, the label is right, and the default
  scheduler handles the pods anyway — because `schedulerName` was never set on the pod
  template. There is no error. The pods schedule normally, partially, and deadlock exactly as in
  L1. Always verify with
  `kubectl get pod <p> -o jsonpath='{.spec.schedulerName}'`, and check the scheduler that
  actually emitted the `FailedScheduling` event in `kubectl describe` — the `From` column names
  it.

- **Setting `minMember` from the framework's default rather than the job's real barrier.** For
  a rigid MPI/NCCL job, `minMember` must equal the world size. For an elastic job it is a
  policy choice (§8). Hard-coding the target size for an elastic job needlessly serialises
  admission; hard-coding the minimum for a rigid job recreates L1.

- **Leaving `scheduleTimeoutSeconds` unset for a large gang.** The fallback is 60 seconds, which
  is fine for a 4-pod gang on an idle cluster and badly wrong for a 64-pod gang on a busy one —
  members time out and release before quorum can ever form, producing perpetual grab/release
  thrash that burns scheduling throughput and makes capacity flicker for everyone else.

- **Assuming gang admission implies topology co-location.** It does not. A `minMember`-satisfied
  gang can land split across racks, NVLink domains, and availability zones. "All ranks present"
  and "ranks are close on the network" are independent guarantees, and the second one is L6's
  subject. A topology-blind gang that admits is still a gang whose all-reduce runs at a fraction
  of the bandwidth it could.

- **Mixing priorities within one `PodGroup`.** `Less()` sorts by pod priority first, so members
  with different priorities can be separated in `activeQ` by unrelated pods, defeating the
  adjacency `QueueSort` exists to provide. The plugin's own docs warn that this "might lead to
  unintended behavior". Keep one priority per group.

- **Believing the gang holds nothing while it waits.** Every member parked at `Permit` was
  assumed into the scheduler cache before `Permit` ran. Its GPU is unavailable to any other pod
  for the duration. Gang scheduling bounds and recovers stranded capacity; it does not eliminate
  it. That is precisely the gap Kueue closes by holding *quota* instead of *nodes*.

## Self-check

- **What is `minMember`, and what breaks if it is set wrong in each direction?** **Answer:** It is
  the quorum — the number of the group's pods that must be simultaneously placeable before any
  member is allowed to bind. Set **too low** (below the job's real collective barrier), quorum
  is reached early, `Permit` returns `Success`, that subset binds, and the processes block in
  `init_process_group` waiting for ranks the gang never guaranteed — the L1 deadlock, restored,
  with a gang scheduler installed. Set **too high** (above the pod count), `PreFilter` computes
  `quorumGap = minMember − len(pods) > 0` and rejects every member forever with the message
  `cannot find enough sibling pods, current pods number: N, minMember of group: M`. The correct
  value is the job's true barrier size: the world size for a rigid MPI/NCCL job, and an explicit
  operator policy choice between minimum-viable and target size for an elastic one.

- **Walk the mechanism: which extension points does the plugin implement, and what does each
  do?** **Answer:** Five. **`QueueSort`** orders by priority, then `PodGroup` creation timestamp,
  then `namespace/name` — so members are adjacent in `activeQ` and older groups go first.
  **`PreFilter`** is the partial-scheduling guard: it deny-lists recently failed groups, rejects
  when fewer than `minMember` pods exist, and (if `minResources` is set) checks whether the
  cluster could host the whole gang at all, discounting resources already held by this same
  group; it returns `UnschedulableAndUnresolvable` specifically to suppress preemption attempts
  on a doomed gang. **`Reserve`** returns `nil` — a deliberate no-op, implemented only so the
  plugin receives an `Unreserve` callback; the actual resource hold is the framework's own
  `assume()`. **`Permit`** is the atomic gate: it inserts the pod into a per-group set, returns
  `Wait` below quorum (activating siblings on the first member so the group assembles in a
  burst), and at quorum returns `Success` and calls `Allow()` on every waiting member at once.
  **`PostFilter`** performs optimistic rejection: if the gap to quorum is at or under
  `podGroupRejectPercentage` (default 10%) of `minMember` it declines to tear the group down;
  otherwise it rejects every waiting member. **`Unreserve`** is the cross-pod rollback — it
  rejects every *other* waiting member of the group, which cascades into their own `Unreserve`
  and releases every assumed placement. That last one is the code path L1 proved the default
  plugin set does not have.

- **How does gang scheduling change queue behaviour for OTHER jobs, and what does it cost?**
  **Answer:** A member parked at `Permit` has already been assumed into the scheduler cache, so
  its GPU is unavailable to any other pod while the gang waits. A large gang assembling slowly
  therefore converts free capacity into held-but-idle capacity — head-of-line blocking, or the
  convoy effect. The worked example prices a `minMember: 32` gang assembling at one GPU per five
  minutes at roughly **43 GPU-hours** denied to smaller jobs during a 160-minute ramp. Two
  qualifications matter: it is **bounded** by `scheduleTimeoutSeconds` and fully recoverable,
  unlike L1's unbounded deadlock (which on the same setup costs ~108 GPU-hours across a retry
  storm and produces nothing); and it is exactly the cost that a queueing layer removes, because
  Kueue holds *quota* rather than *nodes* while a Workload waits. There is also a second,
  quieter cost: the fragmentation tax, since a gang needs `minMember` units of capacity
  simultaneously and its effective fleet is therefore smaller than the fleet.

- **Where does gang scheduling live — admission or binding — and why does the distinction
  matter?** **Answer:** At **admission**, specifically the `Permit` extension point. Binding is
  inherently per-pod — the kubelet receives one `Pod`→`Node` assignment at a time via the
  binding subresource, and no scheduler can change that — but no member enters its binding cycle
  until the whole group is cleared. The distinction matters because it tells you *what is held
  while waiting*: since the members were assumed at placement time and only released at
  `Unreserve`, the gang holds real GPUs on real nodes for the duration of its wait. Contrast
  Kueue, which enforces the same all-or-nothing property one layer up, against aggregate quota,
  before any pod exists — so a waiting Workload holds nothing at all.

- **How do Kueue's Workload admission and the coscheduling plugin differ, and can they be
  combined?** **Answer:** They enforce the same constraint at different altitudes. Kueue's
  mutating webhook forces `spec.suspend: true` so **zero pods are created**, sums the Job's pod
  sets into a `Workload`, and unsuspends only when quota for *all* pod sets can be reserved at
  once — so a waiting Workload consumes nothing. The coscheduling plugin lets all pods be
  created, gives each a real node, and holds those nodes at `Permit`. The tradeoff is that
  Kueue's check is **aggregate and layout-blind**: Kueue's own docs give the counterexample of
  8 free GPUs split 4+4 across two nodes admitting a Workload with a single 8-GPU pod that then
  sticks at the kube-scheduler layer forever. Yes, they compose — Kueue admits against quota,
  coscheduling guarantees atomic placement — but Kueue's own recommended ordering is quota →
  Topology-Aware Scheduling → ProvisioningRequest → `waitForPodsReady` as a safety net, with TAS
  rather than coscheduling as the primary placement guarantee.

- **Why doesn't the native Kubernetes `Workload`/`PodGroup` API (beta, off by default, v1.37)
  make Kueue redundant — and where is it actually *better* than the plugin?** **Answer:** It
  solves the same problem this lesson does — atomic group admission, with
  `PodGroupSpec.schedulingPolicy = Gang{minCount}` playing the role of `minMember` and
  `spec.schedulingGroup.podGroupName` replacing the label. It is architecturally better than the
  plugin at exactly the point where the plugin hurts: the native path simulates the whole group
  against one snapshot with an explicit `revertFns` rollback stack, so a group that cannot be
  placed is reverted within the scheduling cycle instead of parking members that hold assumed
  resources for up to `scheduleTimeoutSeconds`. What it does *not* provide is quota pools,
  cohorts, borrowing, fair-sharing, or per-team showback — the cost/fairness economics layer
  that L3–L4 spend most of this module teaching and where Kueue's value concentrates. Native
  gang admission is converging toward a standard; the queueing and quota economics on top of it
  is not part of that KEP.

## Connections & what's next

This lesson closes the loop L1 opened: diagnosis (L1) → mechanism (L2). But it also opens the
one the rest of the module answers. The convoy effect and the fragmentation tax in the Worked
example are not implementation defects — they are what happens when atomicity is enforced
against *physical nodes* on a first-come-first-served basis, with no notion of whose turn it is
or how much of the fleet a team is entitled to. Replace "hold nodes at `Permit`" with "hold
quota in a pool" and both costs shrink dramatically, which is exactly the move L3 makes. The
topology gap flagged in Common pitfalls — gang-admitted does not mean topology-co-located — is
picked up directly in L6. And the historical grounding in HPC gang scheduling resurfaces in L5,
where Volcano's native Dominant Resource Fairness sits on the same theoretical foundation as
Kueue's cohort fair-sharing in L4.
**Next: [03 — Kueue's queueing model: suspend, admit, quota pool](03-kueue-queueing-model.md)**,
which moves all-or-nothing admission from a scheduler plugin's `Permit` hook to a
controller-driven, quota-aware `Workload` state machine — the anchor of the rest of this module.

## References & further reading

**Primary sources — read directly from cloned repositories this session**

Note on method: this environment's egress proxy blocks `kubernetes.io`,
`scheduler-plugins.sigs.k8s.io`, `kueue.sigs.k8s.io` and several vendor documentation domains.
Rather than cite pages that could not be reached, the mechanism and default-value claims above
were verified against upstream *source trees* cloned during this session. Canonical URLs are
given for convenience with their reachability stated honestly.

- **kubernetes-sigs/scheduler-plugins — `pkg/coscheduling/coscheduling.go` and
  `pkg/coscheduling/core/core.go`** — https://github.com/kubernetes-sigs/scheduler-plugins.
  The authoritative implementation of §4: `Less()`'s three-level ordering, `PreFilter`'s
  `quorumGap` check and `UnschedulableAndUnresolvable` return, `CheckClusterResource`'s
  same-group discounting, `Reserve`'s no-op, `Permit`'s `assignedPodsByPG` counting and
  `ActivateSiblings`, `PostFilter`'s `pgRejectThreshold` optimistic rejection, and `Unreserve`'s
  cross-pod `Reject` loop. **Cloned and read directly this session.** *(This corrects the
  previous version of this lesson, which described `Reserve`/`Unreserve` as "hold the tentative
  resource claim during the wait" — the plugin's `Reserve` is a literal no-op, the hold is the
  framework's `assume()`, and `Unreserve`'s real job is rejecting the group's other waiting
  members.)*
- **kubernetes-sigs/scheduler-plugins — `apis/scheduling/v1alpha1/types.go`.** The `PodGroup`
  CRD: `minMember` (validated `Minimum=1`), `minResources`, `scheduleTimeoutSeconds`, the
  `PodGroupPhase` enum, and the `PodGroupLabel` constant. **Cloned and read directly this
  session.**
- **kubernetes-sigs/scheduler-plugins — `apis/config/types.go`, `apis/config/v1/defaults.go`,
  `pkg/util/podgroup.go`.** The configuration surface and its shipped defaults:
  `permitWaitingTimeSeconds = 60`, `podGroupBackoffSeconds = 0`, `podGroupRejectPercentage = 10`,
  and `DefaultWaitTime = 60s` with `GetWaitTimeDuration`'s precedence order. **Cloned and read
  directly this session.**
- **kubernetes-sigs/scheduler-plugins — `pkg/coscheduling/README.md`.** The maintainers' own
  statement that `queueSort`, `permit` and `unreserve` are mandatory and `preFilter` is an
  optional early-exit optimisation valuable for large `minMember`; the same-priority-per-group
  warning; the self-declared **Beta** maturity; and the 6-nginx-on-3-slots before/after demo.
  **Cloned and read directly this session.**
- **kubernetes-sigs/kueue — `site/content/en/docs/concepts/all_or_nothing.md`** —
  https://kueue.sigs.k8s.io/docs/concepts/all_or_nothing/. Kueue's own account of quota-based
  admission as the first line of defense, the 4+4-GPU layout-blindness counterexample, and the
  recommended ordering of quota → TAS → ProvisioningRequest → `waitForPodsReady`. **Cloned and
  read directly from the repository this session; the rendered docs site was unreachable from
  this environment.**
- **kubernetes/kubernetes — `pkg/scheduler/schedule_one_podgroup.go`,
  `pkg/apis/scheduling/types.go`, `pkg/features/kube_features.go`** —
  https://github.com/kubernetes/kubernetes. The native successor: the `revertFns`
  simulate-and-rollback algorithm, `PodGroupSpec.schedulingPolicy` as a `Basic`/`Gang{minCount}`
  union with `minCount`'s mutability caveats, and `GenericWorkload` as **Beta with
  `Default: false`** on v1.37. **Cloned and read directly this session.**
- **kubernetes/enhancements — KEP-4671, "Gang Scheduling via Workload API"** —
  https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/4671-gang-scheduling/README.md.
  The design rationale behind the `Workload`/`PodGroup` split. **Fetched and verified in an
  earlier session; API specifics above were re-verified against the source tree, since KEP text
  and merged API can drift.**

**Real-world engineering blogs**

- **OpenAI — "Scaling Kubernetes to 7,500 Nodes"** —
  https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — a gang scheduling plugin running
  this exact mechanism in production at scale for MPI workloads that halt entirely if any
  participant fails to start. *(Search-verified; fetch blocked by egress this session.)*
- **Alibaba Cloud ACK — "Use Gang scheduling to solve All-or-Nothing job scheduling issues"** —
  https://www.alibabacloud.com/help/en/ack/ack-managed-and-ack-dedicated/user-guide/work-with-gang-scheduling
  — a second major cloud vendor shipping gang scheduling as a supported, generally available
  managed feature, with its own `PodGroup`-style API. Evidence of mainstream adoption beyond
  the reference implementation. *(Search-verified; fetch blocked by egress this session.)*

**Deeper dives**

- **scheduler-plugins docs site — coscheduling plugin page** —
  https://scheduler-plugins.sigs.k8s.io/docs/plugins/coscheduling/ — the rendered install guide
  and `PodGroup` reference. *(Unreachable from this environment; the same content is in the
  repository's `pkg/coscheduling/README.md`, which was read directly and is cited above.)*
- **Classical coscheduling literature** — Ousterhout's coscheduling work on the Medusa
  multiprocessor, and Feitelson & Rudolph's gang-scheduling papers for distributed-memory
  machines, are the canonical background for §11. Useful for framing this as Kubernetes
  absorbing a solved HPC problem rather than inventing one.
