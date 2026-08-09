---
lesson: "06.2"
title: "Gang scheduling: all-or-nothing admission"
module: "06"
concept: "Gang scheduling: all-or-nothing admission"
status: not-started
est_time: "5h"
artifacts: []
---

# 06.2 · Gang scheduling: all-or-nothing admission

> **Concept.** Gang (co-)scheduling makes a group of pods admit atomically — all `minMember` land together or none bind at all — by parking every member in the `Permit` phase until quorum is reachable, dissolving the L1 deadlock.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + showback](../practice/kueue-showback/README.md)

## Why this matters
The follow-up to the L1 interview question is "so how do you fix it?" — and "gang scheduling" is the two-word answer, but the signal is whether you can name the *mechanism*: a `PodGroup` with a `minMember`, an atomic admission at the `Permit` extension point, and a partial-scheduling guard that keeps the group **entirely pending** rather than partially bound. Getting this right is what converts stranded GPUs (L1's $7–19/hr of pure burn) back into productive capacity. Get `minMember` wrong and you either recreate the deadlock or block the queue for everyone behind you — so this is also where gang scheduling starts trading against *fairness*, the theme of the rest of the module.

## What's new here
Module 02 gave you the `Permit` extension point as an abstract capability: a plugin can return `Wait` to hold a pod in a waiting state before binding, with a timeout, and later `Allow` or `Reject` it. Module 04 gave you `nvidia.com/gpu` as a countable resource and `ResourceQuota` as a static cap. Neither taught you a construct that reasons about a **set of pods as one admission unit**.

That's the new layer: **the `PodGroup` and atomic group admission**. Where L1 showed the default scheduler binding replicas independently and stranding three of four, gang scheduling introduces (1) a first-class group object (`PodGroup` CRD, or Kueue's `Workload`), (2) a `minMember`/quorum threshold, and (3) coordinated use of `QueueSort` → `PreFilter` → `Permit` → `Reserve` so the group is treated as **all-or-nothing**. We are not re-teaching what `Permit` *is*; we're showing what a plugin *does with it* to turn per-pod greed into group atomicity — and what that atomicity costs the jobs waiting behind the gang.

## Core notes

### The PodGroup and minMember
The coscheduling plugin adds a `PodGroup` CRD and a label that ties pods to it:

```yaml
apiVersion: scheduling.x-k8s.io/v1alpha1
kind: PodGroup
metadata:
  name: train-gang
spec:
  minMember: 4                 # quorum: bind all 4 together or none
  scheduleTimeoutSeconds: 30   # how long a waiting member holds before the group gives up
```

Pods join by label; GPU requests are unchanged:

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: gang-demo }
spec:
  parallelism: 4
  completions: 4
  template:
    metadata:
      labels:
        scheduling.x-k8s.io/pod-group: train-gang   # ties each pod to the PodGroup
    spec:
      schedulerName: scheduler-plugins-scheduler     # use the gang-aware scheduler
      containers:
      - name: worker
        image: busybox
        command: ["sleep","3600"]
        resources: { limits: { nvidia.com/gpu: "1" } }
```

**`minMember`** is the quorum: the number of pods that must be simultaneously admissible before *any* member binds. Set it to the replica count of the gang (4 here). It's the load-bearing knob:

- **`minMember` too low** (e.g. 3 for a 4-replica NCCL job): the group admits 3 pods, they bind and block in `init_process_group` waiting for a 4th rank the group never guaranteed — you've **re-created the L1 deadlock**, now with a false sense of safety.
- **`minMember` too high** (e.g. 5 for 4 replicas): quorum is *never* reachable, the group **stays pending forever** and times out repeatedly. Deadlock-free but also progress-free.
- **`minMember` == replica count**: correct. All-or-nothing matches the job's actual barrier semantics.

`minResources` (aggregate CPU/mem/GPU the group needs) can also be set so `PreFilter` can reject early when the cluster provably can't host the whole gang.

### The mechanism: atomic admission at Permit
The plugin coordinates several extension points; the atomicity lives in `Permit`:

1. **`QueueSort`** — orders the queue so pods of the same `PodGroup` (and earlier-created groups) are adjacent, so members are attempted back-to-back rather than interleaved with strangers.
2. **`PreFilter`** — the **partial-scheduling guard**: before scheduling a member, count how many pods of the group exist and could fit. If fewer than `minMember` are even present/feasible, reject immediately so no member wastes a `Reserve`. This is what prevents the "bind 3, strand 1" pattern from ever starting.
3. **`Permit`** — the core. When a member passes Filter/Score, the plugin does **not** let it bind. It returns **`Wait`** (with `scheduleTimeoutSeconds`), parking the pod in the framework's *waiting* map with its node resources tentatively `Reserve`d. Each subsequent member does the same. When the count of waiting members of that group reaches **`minMember`**, the plugin calls **`Allow` on all of them at once** — they proceed to their binding cycles together, atomically. If the timeout fires first, the plugin `Reject`s the waiting members and **`Unreserve`** releases their tentative reservations, so the group falls back to fully pending instead of partially bound.
4. **`Reserve`/`Unreserve`** — hold the tentative resource claim during the wait; release it cleanly on timeout/rejection.

The net invariant: **the group is either fully admitted or fully pending — never partial.** That is the entire difference from L1.

### Where it lives: admission, not binding
Gang scheduling is an **admission-time** decision. The bind cycle still binds one pod at a time (binding is inherently per-pod — kubelet gets one Pod→Node assignment at a time). What's atomic is the **decision to admit the group**, gated in `Permit` *before* any member enters its binding cycle. So the correct one-liner: *gang scheduling is enforced at admission, at the `Permit` extension point; binding remains per-pod but only happens after the whole group is cleared.*

### Effect on the queue: the fairness cost
A pending gang doesn't just wait — it can **hold the head of line**. With `QueueSort` keeping the group together and `Permit` reserving resources for members already waiting, a large gang (say `minMember: 32`) that can't yet assemble may **block smaller jobs behind it** from using the fragments it's tentatively holding. Two behaviors matter:

- **Head-of-line blocking / convoy effect.** A big gang waiting for quorum can starve small jobs that *could* have run in the gaps. This is why raw coscheduling is usually paired with a **queueing layer** (Kueue, next lessons) that adds priorities, borrowing, and backfill so waiting gangs don't idle the cluster.
- **Reservation churn.** If a gang repeatedly reserves-then-times-out (`scheduleTimeoutSeconds` too short for a large `minMember`), it thrashes: grabbing fragments, releasing them, re-grabbing — burning scheduling throughput and delaying everyone. Size `scheduleTimeoutSeconds` to the realistic time for `minMember` GPUs to free up.

So gang scheduling **removes intra-group deadlock** but **introduces an inter-group fairness question**: your gang's atomicity is purchased with other jobs' latency. That trade is exactly what Kueue's cohorts, quotas, and preemption (upcoming) exist to manage.

### Kueue's take (the alternative you'll actually run in prod)
Kueue does gang admission at a higher level: it suspends a `Job` (`spec.suspend: true`) until its `Workload` — the whole set of pods with its aggregate resource ask — fits within a `ClusterQueue`'s quota, then unsuspends it so all pods become schedulable together. This is all-or-nothing at the **quota-admission** layer rather than the scheduler-`Permit` layer, and it composes cost/fairness (quotas, cohorts, fair-sharing, borrowing) natively — which is why the deliverable is Kueue-based. Coscheduling and Kueue can be layered (Kueue admits the Workload; coscheduling guarantees the pods land atomically), but for a homogeneous gang Kueue's Workload admission is often enough on its own.

## Worked example

Same 3-worker fake-GPU cluster and same 4-replica job from L1 — but now with the coscheduling scheduler and a `PodGroup(minMember: 4)`. Only 3 GPUs are free, so quorum is unreachable:

```bash
$ kubectl apply -f podgroup.yaml       # PodGroup train-gang, minMember: 4
$ kubectl apply -f gang4.yaml          # 4 pods, labelled, schedulerName: scheduler-plugins-scheduler
$ kubectl get pods -l job-name=gang-demo
NAME              READY   STATUS    NODE
gang-demo-p1w2x   0/1     Pending   <none>
gang-demo-p3q4r   0/1     Pending   <none>
gang-demo-p5t6y   0/1     Pending   <none>
gang-demo-p7u8i   0/1     Pending   <none>

$ kubectl describe pod gang-demo-p1w2x | grep -A2 Events
Events:
  Warning  FailedScheduling  cannot find enough sibling pods, wait for more pods
                              (0/4 nodes: gang scheduling minMember 4 not satisfied)
```

**Critical diff vs L1:** in L1, 3 pods were `Running` and holding GPUs while 1 pended — 3 GPUs stranded. Here, **all 4 pods are `Pending` and 0 GPUs are consumed.** The gang refuses to partially bind, so the 3 free GPUs stay *available* for other jobs instead of being deadlocked. Now free a 4th GPU (scale up / free capacity) and quorum flips:

```bash
$ kubectl label node fake-gpu-worker4 ...   # add a 4th fake-GPU node → 4 free GPUs
$ kubectl get pods -l job-name=gang-demo
NAME              READY   STATUS    NODE
gang-demo-p1w2x   1/1     Running   fake-gpu-worker
gang-demo-p3q4r   1/1     Running   fake-gpu-worker2
gang-demo-p5t6y   1/1     Running   fake-gpu-worker3
gang-demo-p7u8i   1/1     Running   fake-gpu-worker4   # all 4 admitted in one Permit batch
```

All four transition together once `minMember` waiting members can be `Allow`ed at once. Capture both states (all-pending, then all-running) side by side with the L1 capture (3-running/1-pending): that before/after pair is the deliverable's gang demo.

## Practice
Continue on the L1 **kind cluster with fake `nvidia.com/gpu`**:

1. Install the gang-aware scheduler. Either (a) deploy **scheduler-plugins** as a second scheduler (`schedulerName: scheduler-plugins-scheduler`, RBAC + the `PodGroup` CRD via its Helm chart / manifests), or (b) install **Kueue** and use its Workload-level gang admission. Pick one; scheduler-plugins most directly exercises the `Permit` mechanism above.
2. Create a `PodGroup` with `minMember: 4` (coscheduling) — or a `ClusterQueue` + `LocalQueue` with a 4-GPU nominal quota and submit a suspended `Job` (Kueue).
3. Re-submit the **exact L1 4-replica job**, now labelled into the group / queue, on the cluster with only **3 free GPUs**. Confirm **all 4 pods `Pending`, 0 GPUs consumed** — capture `get pods` + the `cannot find enough sibling pods` event.
4. Free a 4th GPU (add a worker or free one). Confirm all 4 flip to `Running` **together**. Capture.
5. Diff against L1: annotate that L1 stranded 3 GPUs (`3 × $2.35 = $7.05/hr` burn) while the gang stranded **zero** — the 3 free GPUs stayed usable by other work until quorum.

**Acceptance:** a before/after capture proving the L1 deadlock is fixed — 4-pending-0-consumed → 4-running-together — with the cost annotation showing stranded-GPU-hours driven to zero. Pairs with the L1 capture as the deliverable's complete gang-scheduling demo.

## Self-check
**(a) What is `minMember` and what breaks if it's set wrong?**
**Answer:** `minMember` is the quorum — the number of the group's pods that must be simultaneously admissible before *any* binds. Too low (below the job's real barrier size), the group admits a subset that binds and then blocks waiting for ranks the gang never guaranteed — you re-create the L1 partial-placement deadlock. Too high (above the pod count), quorum is never reachable and the group stays pending forever. Correct value is the gang's actual replica count so all-or-nothing matches the collective's barrier.

**(b) How does gang scheduling change queue behavior for OTHER jobs waiting behind the gang?**
**Answer:** A pending gang can cause head-of-line blocking / a convoy effect: while it waits for quorum (and tentatively reserves fragments in `Permit`), smaller jobs that could have used those gaps may be starved, and a mis-sized `scheduleTimeoutSeconds` causes reserve/timeout thrash that delays everyone. Gang scheduling removes *intra*-group deadlock but introduces an *inter*-group fairness cost — which is why it's paired with a queueing layer (Kueue: priorities, quotas, borrowing, backfill) to keep waiting gangs from idling the cluster.

**(c) Where does gang scheduling live — at admission or at binding — and which extension point?**
**Answer:** At **admission**, specifically the **`Permit`** extension point. Members that pass Filter/Score are parked in a `Wait` state with their resources tentatively reserved; only when `minMember` members are simultaneously waiting does the plugin `Allow` them all at once. Binding itself remains per-pod (kubelet takes one assignment at a time) but happens only *after* the whole group is admitted, so the atomic decision is at `Permit`, not at bind. (`PreFilter` provides the early partial-scheduling guard; `Reserve`/`Unreserve` hold and release the tentative claim.)

## Resources
1. **scheduler-plugins — coscheduling plugin docs** — https://scheduler-plugins.sigs.k8s.io/docs/plugins/coscheduling/ — install steps, `PodGroup` spec (`minMember`, `scheduleTimeoutSeconds`, `minResources`), and the pod-group label; **deep** read, it's the config reference for the practice. Authoritative and version-matched to the plugin you deploy.
2. **scheduler-plugins — coscheduling README** — https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/coscheduling/README.md — the design writeup mapping each behavior to `QueueSort`/`PreFilter`/`Permit`/`Reserve`; **deep** read to cement the mechanism for interviews. Best source for *why* the `Permit`-phase wait produces atomicity.
3. **Kueue — gang / all-or-nothing admission** — https://kueue.sigs.k8s.io/docs/ — the production alternative: Workload-level atomic admission with quotas, cohorts, and fair-sharing; **skim** now, **deep** in the deliverable. This is what the module's Kueue-showback practice is built on.
