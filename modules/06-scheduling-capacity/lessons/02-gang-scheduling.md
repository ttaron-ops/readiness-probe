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
sources: 8
---

# 06.2 · Gang scheduling: all-or-nothing admission

> **Concept.** Gang (co-)scheduling makes a group of pods admit atomically — all `minMember` land together or none bind at all — by parking every member in the `Permit` phase until quorum is reachable, dissolving the L1 deadlock.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits
L1 diagnosed the disease: the default scheduler is a per-pod greedy binder with no cross-pod rollback, so a 4-replica job on a 3-GPU-free cluster strands 3 GPUs holding nothing useful while a 4th pod pends forever. This lesson is the cure — it takes the exact deadlocked cluster from L1 and shows the mechanism that prevents the deadlock from ever forming: atomic, all-or-nothing admission at the `Permit` extension point. Getting this mechanism right is also where the module's real tension first appears — atomicity for one job's gang is purchased, in part, with latency for other jobs waiting behind it, which is the seed of everything L3 onward (queues, quotas, fair-sharing) exists to manage.

## Why this matters
The natural follow-up to L1's interview question is "so how do you fix it?" — and "gang scheduling" is the two-word answer, but the signal interviewers are actually listening for is whether you can name the *mechanism*: a `PodGroup` with a `minMember`, atomic admission at the `Permit` extension point, and a partial-scheduling guard that keeps the group **entirely pending** rather than partially bound. Getting this right is what converts stranded GPUs (L1's $7–19/hr of pure burn) back into productive capacity. Get `minMember` wrong in either direction and you either recreate the deadlock or you block the queue for everyone behind you — which is exactly why CoreWeave's Principal/Staff posting frames this as "quota enforcement, fairness, pre-emption" in the same breath as scheduling, and why Anthropic's JD names "gang scheduling; topology-aware placement" as one phrase. This is not a standalone party trick; it's the first domino in the module's whole fairness/cost chain.

## What's new here (calibration)
- **Module 02** gave you the `Permit` extension point as an abstract capability: a plugin can return `Wait` to hold a pod before binding, with a timeout, then later `Allow` or `Reject` it. Not re-derived here.
- **Module 04** gave you `nvidia.com/gpu` as a countable resource and `ResourceQuota` as a static, synchronous-reject cap. Assumed, not re-taught.
- **L1 (this module)** gave you the deadlock this lesson fixes and the specific extension points a gang plugin would need. That diagnosis is the premise; this lesson is the mechanism.
- **Genuinely new here:** the `PodGroup`/`minMember` construct as a first-class group-admission unit; the precise choreography across `QueueSort → PreFilter → Permit → Reserve/Unreserve` that makes atomicity real; the **inter-group fairness cost** atomic admission introduces (head-of-line blocking, reservation churn); and — new since this course was scoped — the fact that **native Kubernetes now ships an alpha `Workload`/`PodGroup` API (v1.35+)** that generalizes exactly this `Permit`-based mechanism into core Kubernetes, which changes how you should frame "third-party plugin vs. built-in" in an interview answer.

## Core concepts

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

- **`minMember` too low** (e.g. 3 for a 4-replica NCCL job): the group admits 3 pods, they bind and block in `init_process_group` waiting for a 4th rank the group never guaranteed — you've **re-created the L1 deadlock**, now with a false sense of safety, because "we have gang scheduling installed" doesn't mean "we configured it correctly for this job."
- **`minMember` too high** (e.g. 5 for 4 replicas): quorum is *never* reachable, the group **stays pending forever** and times out repeatedly. Deadlock-free but also progress-free — a different, equally useless failure mode.
- **`minMember` == replica count**: correct. All-or-nothing matches the job's actual barrier semantics.

`minResources` (aggregate CPU/mem/GPU the group needs) can also be set so `PreFilter` can reject early when the cluster provably can't host the whole gang, saving wasted scheduling cycles on a doomed group.

### The mechanism: atomic admission at Permit
The plugin coordinates several extension points; the atomicity lives in `Permit`. Per the coscheduling plugin's own design (its README states the `Permit` plugin is mandatory alongside `queueSort` and `unreserve`, with `preFilter` as an optional early-exit optimization):

1. **`QueueSort`** — orders the queue so pods of the same `PodGroup` (and earlier-created groups) are adjacent, so members are attempted back-to-back rather than interleaved with strangers.
2. **`PreFilter`** — the **partial-scheduling guard**: before scheduling a member, count how many pods of the group exist and could fit. If fewer than `minMember` are even present/feasible, reject immediately so no member wastes a `Reserve`. This is what prevents the "bind 3, strand 1" pattern from ever starting, and the plugin's own docs note it's "especially valuable for larger `minMember` configurations" — the bigger the gang, the more wasted work this guard avoids.
3. **`Permit`** — the core. When a member passes Filter/Score, the plugin does **not** let it bind. It returns **`Wait`** (with `scheduleTimeoutSeconds`), parking the pod in the framework's *waiting* map with its node resources tentatively `Reserve`d. Each subsequent member does the same. The scheduler counts running-plus-waiting pods for that group; when the total reaches **`minMember`**, the plugin calls **`Allow` on all waiting members at once** — they proceed to their binding cycles together, atomically. If the timeout fires first, the plugin `Reject`s the waiting members and **`Unreserve`** releases their tentative reservations, so the group falls back to fully pending instead of partially bound.
4. **`Reserve`/`Unreserve`** — hold the tentative resource claim during the wait; release it cleanly on timeout/rejection.

The net invariant: **the group is either fully admitted or fully pending — never partial.** That is the entire difference from L1. The coscheduling plugin also establishes precedence rules across competing groups: a higher-priority `PodGroup` is considered before a lower-priority one, and among equal-priority groups, the earlier-created group gets preference when resources are contended — a first taste of the fairness questions L3–L4 formalize.

### Where it lives: admission, not binding
Gang scheduling is an **admission-time** decision. The bind cycle still binds one pod at a time (binding is inherently per-pod — kubelet gets one Pod→Node assignment at a time). What's atomic is the **decision to admit the group**, gated in `Permit` *before* any member enters its binding cycle. So the correct one-liner: *gang scheduling is enforced at admission, at the `Permit` extension point; binding remains per-pod but only happens after the whole group is cleared.*

### Effect on the queue: the fairness cost
A pending gang doesn't just wait — it can **hold the head of line**. With `QueueSort` keeping the group together and `Permit` reserving resources for members already waiting, a large gang (say `minMember: 32`) that can't yet assemble may **block smaller jobs behind it** from using the fragments it's tentatively holding. Two behaviors matter:

- **Head-of-line blocking / convoy effect.** A big gang waiting for quorum can starve small jobs that *could* have run in the gaps. This is why raw coscheduling is usually paired with a **queueing layer** (Kueue, L3 onward) that adds priorities, borrowing, and backfill so waiting gangs don't idle the cluster. See the Worked example below for a quantified version of this effect.
- **Reservation churn.** If a gang repeatedly reserves-then-times-out (`scheduleTimeoutSeconds` too short for a large `minMember`), it thrashes: grabbing fragments, releasing them, re-grabbing — burning scheduling throughput and delaying everyone. Size `scheduleTimeoutSeconds` to the realistic time for `minMember` GPUs to free up.

So gang scheduling **removes intra-group deadlock** but **introduces an inter-group fairness question**: your gang's atomicity is purchased with other jobs' latency. That trade is exactly what Kueue's cohorts, quotas, and preemption (upcoming) exist to manage.

### Kueue's take (the alternative you'll actually run in prod)
Kueue does gang admission at a higher level: it suspends a `Job` (`spec.suspend: true`) until its `Workload` — the whole set of pods with its aggregate resource ask — fits within a `ClusterQueue`'s quota, then unsuspends it so all pods become schedulable together. This is all-or-nothing at the **quota-admission** layer rather than the scheduler-`Permit` layer, and it composes cost/fairness (quotas, cohorts, fair-sharing, borrowing) natively — which is why the deliverable is Kueue-based. Coscheduling and Kueue can be layered (Kueue admits the Workload; coscheduling guarantees the pods land atomically), but for a homogeneous gang Kueue's Workload admission is often enough on its own.

### Elasticity is a genuinely different axis from minMember
Not every distributed job has a fixed, rigid replica count. PyTorch elastic training supports a range — `torchrun --nnodes=4:8` means the job can *start* with 4 nodes and scale up to 8 as capacity frees, tolerating worker churn mid-run. Gang scheduling's `minMember` is a single fixed number, so an elastic job forces a design decision the operator has to make explicitly: set `minMember` to the *minimum viable* size (4) and let the job scale up opportunistically once running, or set it to the *target* size (8) and accept the job won't start until the larger quorum is available. There's no universally correct answer — it depends on whether the workload benefits more from starting sooner at reduced parallelism or from guaranteed full-scale throughput from step zero — but conflating "elastic range" with "gang minimum" is a common design mistake, because they answer different questions (how small can this job tolerate running vs. how many pods must be present simultaneously to admit at all).

### Historical grounding: this is not a Kubernetes invention
Gang (or "coscheduling") is a specific instance of the classical **all-or-nothing atomic admission control** problem from distributed systems and HPC batch scheduling, studied since the early 1990s for parallel supercomputers (Feitelson & Rudolph's gang-scheduling literature for distributed-memory machines is the canonical starting point). Slurm, PBS, and LSF have implemented some form of this for decades. Kubernetes catching up here with `scheduler-plugins`, Kueue/Volcano, and now a native `Workload` API is Kubernetes absorbing a solved HPC problem into a cloud-native substrate — worth naming explicitly so this doesn't read as a novel Kubernetes-specific trick in an interview.

### New since this course was scoped: the native successor to this mechanism
L1 introduced Kubernetes v1.35's alpha `Workload`/`PodGroup` API (KEP-4671). It's worth being precise about how it relates to *this specific lesson*, because it generalizes the exact `Permit`-based mechanism you just learned: the native scheduler runs a "Workload Scheduling Cycle" that evaluates all gang members against one cluster-state snapshot atomically, which is functionally the same all-or-nothing guarantee the coscheduling plugin's `Permit` wait produces — just moved from a plugin into the core scheduling loop, with `GangSchedulingPolicy.minCount` playing the role of `minMember`. It's alpha and disabled by default (`GenericWorkload` feature gate, beta targeted for v1.37), so `scheduler-plugins` and Kueue remain what you deploy today — but the direction of travel matters for how you answer "will this always require a third-party plugin?" in an interview: **no, but the fairness/quota/cohort economics on top of admission — Kueue's actual value-add — is not part of that native API and isn't going away.**

## Perspectives

**Developer.** From inside the training script, gang scheduling is invisible when it works — `init_process_group` either starts promptly (all ranks present) or the Job never creates running pods at all (a clean signal: "waiting for capacity," not "hung"). This is a genuine developer-experience win worth naming explicitly in an interview: it converts an undiagnosable hang into a legible, observable wait state.

**Operator.** Choosing `minMember` correctly requires knowing the job's *true* barrier size, which for elastic/fault-tolerant frameworks may not be a fixed number at all — the operator has to decide whether to gang-schedule the minimum viable size or the target size, a genuine design tension that doesn't exist for rigid MPI jobs where the replica count is simply fixed for the job's lifetime.

**Failure-mode/queueing.** `scheduleTimeoutSeconds` sizing is a queueing-theory problem in miniature — too short causes reserve/thrash churn as the gang repeatedly grabs and releases fragments; too long lets a doomed gang camp on tentatively-reserved fragments and starve everyone else behind it. This is the same head-of-line-blocking problem that motivates Kueue's queueing layer entirely — it's not a coincidence that L3 exists.

**Theory.** Gang scheduling is a specific instance of the classical all-or-nothing (atomic) admission control problem from distributed systems and HPC batch schedulers, going back decades before Kubernetes existed. Seeing the `Permit`-phase mechanism as "Kubernetes implementing a 1990s idea in a cloud-native idiom" — rather than as a novel invention — is both historically accurate and the more sophisticated answer in an interview.

## Real-world use cases

- **OpenAI — "Scaling Kubernetes to 7,500 Nodes"** — https://openai.com/index/scaling-kubernetes-to-7500-nodes/. What it shows: a concrete, named coscheduling plugin running in production at scale — the mechanism this lesson teaches, not just the problem L1 describes. OpenAI adopted it specifically because their MPI-based jobs halt entirely if any single participating pod dies or fails to start. *(Search-verified this session; fetch blocked by egress.)*
- **kubernetes-sigs/scheduler-plugins — coscheduling README** — https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/coscheduling/README.md. What it shows: the primary source for the `QueueSort`/`PreFilter`/`Permit`/`Unreserve` mechanism this lesson describes in detail — the plugin's own maintainers confirming `Permit` and `unreserve` are mandatory, `preFilter` is an optional-but-valuable early-exit optimization, and priority/creation-order precedence rules across competing groups. **Fetched and verified directly this session.**
- **Alibaba Cloud — "Use Gang scheduling to solve All-or-Nothing job scheduling issues"** — https://www.alibabacloud.com/help/en/ack/ack-managed-and-ack-dedicated/user-guide/work-with-gang-scheduling. What it shows: a major cloud vendor's managed Kubernetes product (ACK) documenting gang scheduling — implemented via a `PodGroup` resource with a `min-available` field and built directly on the same kube-scheduler framework extension points — as a first-class, generally-available feature for Pro-tier clusters running Kubernetes 1.16+. Evidence this is mainstream production practice, not a niche pattern, and a second real config example distinct from `scheduler-plugins`' own docs. *(Search-verified this session; fetch blocked by egress.)*
- **Kubernetes v1.35/v1.36 — Workload-Aware Scheduling blogs** — https://kubernetes.io/blog/2025/12/29/kubernetes-v1-35-introducing-workload-aware-scheduling/ and https://kubernetes.io/blog/2026/05/13/kubernetes-v1-36-advancing-workload-aware-scheduling/. What it shows: where this mechanism is heading — the vendor-neutral, native successor to plugin-based gang scheduling, worth citing here specifically because it generalizes the `Permit`-based wait-then-admit-atomically pattern this lesson teaches into the core scheduling loop. *(Search-verified this session; fetch blocked by egress.)*

## Worked example

**Part 1 — the before/after capture.** Same 3-worker fake-GPU cluster and same 4-replica job from L1 — but now with the coscheduling scheduler and a `PodGroup(minMember: 4)`. Only 3 GPUs are free, so quorum is unreachable:

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

**Part 2 — quantify the convoy effect.** Now put a number on the "fairness cost" from Core concepts. Suppose a large gang has `minMember: 32`, and the cluster frees exactly one GPU every 5 minutes (a plausible rate as other jobs complete). Under **naive coscheduling with no backfill**, each freed GPU is immediately claimed by one gang member entering the `Permit` wait state, tentatively reserved, holding it unusable by anyone else while the gang keeps waiting for the rest:

```
time to reach 32 GPUs at 1 per 5 min  = 160 minutes ≈ 2h 40m
GPUs held-but-not-running rises linearly from 1 → 32 over that window
average GPUs held during the ramp    ≈ 16.5
GPU-minutes denied to other jobs     = 16.5 × 160 ≈ 2,640 GPU-minutes ≈ 44 GPU-hours
```

That's **44 GPU-hours** of capacity that small 1–2 GPU jobs could have used during the ramp-up but couldn't, because the gang's `Permit`-held reservations locked each freed GPU out of the scheduling pool the moment it appeared. Under a **queueing layer with backfill** (Kueue-style), small jobs are allowed to use freed capacity that the gang hasn't yet been admitted to consume, and the gang's admission decision is made against aggregate quota rather than by individually reserving each physical GPU the instant it frees — driving that 44 GPU-hour figure toward zero. *(This is an illustrative, order-of-magnitude calculation to make the tradeoff numeric, not a claim about any specific scheduler implementation's exact internal accounting — the mechanism differs by system, but the shape of the tradeoff is real and is precisely why Kueue exists on top of raw gang scheduling.)*

## Practice

Continue on the L1 **kind cluster with fake `nvidia.com/gpu`**:

1. Install the gang-aware scheduler. Either (a) deploy **scheduler-plugins** as a second scheduler (`schedulerName: scheduler-plugins-scheduler`, RBAC + the `PodGroup` CRD via its Helm chart / manifests), or (b) install **Kueue** and use its Workload-level gang admission. Pick one; scheduler-plugins most directly exercises the `Permit` mechanism above.
2. Create a `PodGroup` with `minMember: 4` (coscheduling) — or a `ClusterQueue` + `LocalQueue` with a 4-GPU nominal quota and submit a suspended `Job` (Kueue).
3. Re-submit the **exact L1 4-replica job**, now labelled into the group / queue, on the cluster with only **3 free GPUs**. Confirm **all 4 pods `Pending`, 0 GPUs consumed** — capture `get pods` + the `cannot find enough sibling pods` event.
4. Free a 4th GPU (add a worker or free one). Confirm all 4 flip to `Running` **together**. Capture.
5. Diff against L1: annotate that L1 stranded 3 GPUs (`3 × $2.35 = $7.05/hr` burn) while the gang stranded **zero** — the 3 free GPUs stayed usable by other work until quorum.
6. **Stretch (optional):** deliberately misconfigure `minMember` (set it to 3 for the 4-replica job) and observe the L1 deadlock re-appear even with the gang scheduler installed — this drives home that gang scheduling is only as correct as its configuration, not a mechanism that's automatically safe once deployed.

**Acceptance:** a before/after capture proving the L1 deadlock is fixed — 4-pending-0-consumed → 4-running-together — with the cost annotation showing stranded-GPU-hours driven to zero. Pairs with the L1 capture as the deliverable's complete gang-scheduling demo.

## Common pitfalls

- **Setting `minMember` to the framework's default without checking the job's real elasticity.** PyTorch elastic jobs may tolerate `nnodes=4:8`, so hard-coding `minMember=8` needlessly serializes admission when 4 would let the job start and scale up — you pay the atomicity tax without needing the full quorum.
- **Forgetting `scheduleTimeoutSeconds` entirely** (relying on the plugin default). A default tuned for small gangs can be badly wrong for a 64-pod gang on a busy cluster, causing perpetual reserve/timeout thrash that burns scheduling throughput and delays other jobs.
- **Assuming gang scheduling implies topology co-location.** It does not — a `minMember`-satisfied gang can still land split across racks or NVLink domains, which is a completely separate problem L6 solves. "All ranks present" and "ranks are close together on the network" are independent guarantees.
- **Deploying the gang scheduler and assuming it's automatically safe.** As the stretch practice task shows, a misconfigured `minMember` can silently recreate the exact L1 deadlock — "we have gang scheduling" is not the same claim as "we have gang scheduling configured correctly for this job."

## Self-check

- What is `minMember` and what breaks if it's set wrong? **Answer:** `minMember` is the quorum — the number of the group's pods that must be simultaneously admissible before *any* binds. Too low (below the job's real barrier size), the group admits a subset that binds and then blocks waiting for ranks the gang never guaranteed — you re-create the L1 partial-placement deadlock. Too high (above the pod count), quorum is never reachable and the group stays pending forever. Correct value is the gang's actual replica count so all-or-nothing matches the collective's barrier.
- How does gang scheduling change queue behavior for OTHER jobs waiting behind the gang? **Answer:** A pending gang can cause head-of-line blocking / a convoy effect: while it waits for quorum (and tentatively reserves fragments in `Permit`), smaller jobs that could have used those gaps may be starved — the worked example quantifies this at roughly 44 GPU-hours denied during a 160-minute ramp for a `minMember: 32` gang under naive coscheduling. A mis-sized `scheduleTimeoutSeconds` also causes reserve/timeout thrash that delays everyone. Gang scheduling removes *intra*-group deadlock but introduces an *inter*-group fairness cost — which is why it's paired with a queueing layer (Kueue: priorities, quotas, borrowing, backfill) to keep waiting gangs from idling the cluster.
- Where does gang scheduling live — at admission or at binding — and which extension point? **Answer:** At **admission**, specifically the **`Permit`** extension point. Members that pass Filter/Score are parked in a `Wait` state with their resources tentatively reserved; only when `minMember` members are simultaneously waiting does the plugin `Allow` them all at once. Binding itself remains per-pod (kubelet takes one assignment at a time) but happens only *after* the whole group is admitted, so the atomic decision is at `Permit`, not at bind. (`PreFilter` provides the early partial-scheduling guard; `Reserve`/`Unreserve` hold and release the tentative claim.)
- What's the practical difference between job-level elasticity (PyTorch elastic `nnodes` range) and gang scheduling's `minMember`, and can they coexist? **Answer:** Elasticity describes the *range* a job can tolerate running at (e.g., 4 to 8 nodes, with the framework handling worker churn mid-run); `minMember` describes a *fixed* quorum the scheduler requires before admitting anything. They coexist by setting `minMember` to the elastic job's minimum viable size (e.g., 4) so the job can start early and scale opportunistically as more nodes become admissible — but this is an operator design choice, not something the scheduler infers automatically, and setting `minMember` to the target size instead trades faster start for guaranteed full-scale throughput.
- Why doesn't the arrival of a native Kubernetes `Workload`/`PodGroup` API (v1.35+) immediately obsolete Kueue's Workload-level admission model? **Answer:** The native API solves the same problem this lesson does — atomic group admission — by moving the `Permit`-style wait-then-admit mechanism into the core scheduling loop with `GangSchedulingPolicy.minCount`. It says nothing about quota pools, cohorts, borrowing, fair-sharing, or per-team showback reporting — the cost/fairness economics layer Kueue is built around and that L3–L5 spend most of the module teaching. Native gang admission is converging toward a standard; the queueing/quota economics on top of it is not part of that KEP and remains Kueue's (or Volcano's) value-add.

## Connections & what's next

This lesson closes the loop L1 opened: diagnosis (L1) → mechanism (L2). The fairness cost surfaced here — a pending gang holding fragments hostage from smaller jobs — is the exact motivation for L3's queueing model, which replaces raw `Permit`-level reservation with quota-aware, suspend-not-reject admission. The topology gap flagged in Common pitfalls (gang-admitted doesn't mean topology-co-located) is picked up directly in L6. And the historical grounding in classical HPC gang scheduling resurfaces conceptually in L5, where Volcano's native Dominant Resource Fairness sits on the same theoretical foundation as Kueue's cohort fair-sharing (L4). **Next: [03 — Kueue's queueing model: suspend, admit, quota pool](03-kueue-queueing-model.md)**, which moves gang admission from a scheduler plugin's `Permit` hook to a controller-driven, quota-aware `Workload` state machine — the anchor of the rest of this module.

## References & further reading

**Primary sources**
- [kubernetes-sigs/scheduler-plugins — coscheduling README](https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/coscheduling/README.md) — read for the exact `QueueSort`/`PreFilter`/`Permit`/`Unreserve` mechanism and the priority/creation-order precedence rules. **Fetched and verified directly this session.**
- [scheduler-plugins.sigs.k8s.io — coscheduling plugin docs](https://scheduler-plugins.sigs.k8s.io/docs/plugins/coscheduling/) — read for install steps and the `PodGroup` spec reference (`minMember`, `scheduleTimeoutSeconds`, `minResources`) used directly in the practice section. *(Cited in the prior version of this lesson; fetch blocked by egress this session — cite canonical URL per SPEC sourcing rules.)*
- [Kueue documentation](https://kueue.sigs.k8s.io/docs/) — read for Workload-level atomic admission via `spec.suspend`, the production alternative this lesson previews and L3 builds on. *(Fetch blocked by egress this session; cite canonical URL.)*
- [kubernetes/enhancements — KEP-4671, Gang Scheduling via Workload API](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/4671-gang-scheduling/README.md) — read for how `GangSchedulingPolicy.minCount` and `spec.schedulingGroup.podGroupName` generalize this lesson's `Permit`-based mechanism into core Kubernetes. **Fetched and verified directly this session.**

**Real-world engineering blogs**
- [OpenAI — "Scaling Kubernetes to 7,500 Nodes"](https://openai.com/index/scaling-kubernetes-to-7500-nodes/) — what it shows: a named coscheduling plugin running this exact mechanism in production at scale for MPI workloads. *(Search-verified; fetch blocked by egress this session.)*
- [Alibaba Cloud — "Use Gang scheduling to solve All-or-Nothing job scheduling issues"](https://www.alibabacloud.com/help/en/ack/ack-managed-and-ack-dedicated/user-guide/work-with-gang-scheduling) — what it shows: a second major cloud vendor shipping gang scheduling as a supported, general-availability feature — evidence of mainstream production adoption beyond the `scheduler-plugins` reference implementation. *(Search-verified; fetch blocked by egress this session.)*

**Deeper dives**
- [Kubernetes v1.35 — "Introducing Workload Aware Scheduling"](https://kubernetes.io/blog/2025/12/29/kubernetes-v1-35-introducing-workload-aware-scheduling/) and [v1.36 — "Advancing Workload-Aware Scheduling"](https://kubernetes.io/blog/2026/05/13/kubernetes-v1-36-advancing-workload-aware-scheduling/) — read for where this lesson's mechanism is heading upstream, in the scheduler team's own words. *(Search-verified; fetch blocked by egress this session.)*
