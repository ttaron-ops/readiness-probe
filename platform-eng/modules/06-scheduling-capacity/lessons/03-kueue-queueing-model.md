---
lesson: "06.3"
title: "Kueue I — the queueing model: suspend, admit, and the quota pool"
module: "06"
concept: "kueue-queueing-suspend-admit"
status: not-started
est_time: "10h"
prev: "02-gang-scheduling.md"
next: "04-kueue-cohorts-borrowing-preemption.md"
artifacts: []
sources: 16
---
# 06.3 · Kueue I — the queueing model: suspend, admit, and the quota pool

> **Concept.** Kueue admits work by *suspending* Jobs and *unsuspending* them when quota is free — a queue, not a wall.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

[L1](01-why-default-scheduler-fails.md) showed *why* the default scheduler deadlocks a distributed job whose pods cannot all be bound at once: the scheduling unit is the pod, placement is incremental, and there is no rollback. [L2](02-gang-scheduling.md) fixed atomicity for one job — all pods or none — and ended by contrasting two places to enforce it: the coscheduling plugin's `Permit` hook, which parks pods **holding real GPUs on real nodes**, versus Kueue, which holds a `Workload` object and **zero pods**.

This lesson is the inside of that second box. L2 gave you the one-paragraph version: a webhook suspends the Job, a controller sums the pod sets into a `Workload`, and quota is checked in aggregate. That paragraph hides the entire system: four API objects and their composition rules, a two-phase admission cycle with a documented sort order, a flavor-assignment algorithm that walks a list, and a status surface of about a dozen conditions and thirty reason strings that is the only thing standing between you and "the job is just sitting there."

Gang scheduling answers *"can this one job start safely?"* Kueue answers *"which job should start next, out of the two hundred that ten teams have submitted against a fleet that fits eight of them?"* That is a queueing and quota-accounting problem, and it is where the module's FinOps thesis lands: the quota pool is the unit you do showback against. Master this lesson and [L4](04-kueue-cohorts-borrowing-preemption.md) and you have the deepest interview surface in the module.

## Why this matters

You run a fixed GPU fleet that many teams want more of than exists. The stock Kubernetes answer to "team X wants more than its share" is `ResourceQuota`: an **admission plugin inside kube-apiserver** that intercepts the pod create, compares committed usage in that namespace against the quota object, and returns `403 Forbidden: exceeded quota` **synchronously**. The workload is then gone. Nothing remembers that it wanted to run. There is no ordering, no fairness, no "start it when capacity frees." Every client is forced into a submit-and-retry loop, which on a busy fleet means every team writes the same bad retry script and your apiserver absorbs the aggregate.

For an expensive, permanently oversubscribed GPU fleet that is the wrong primitive in a specific, quantifiable way. Consider two teams, each capped at 4 GPUs of an 8-GPU fleet by `ResourceQuota`. Team A is idle for the weekend; team B has a 6-GPU job queued. `ResourceQuota` rejects team B's job for 48 hours while four GPUs sit dark. At a mid-range on-demand rate of order **$2–3/GPU-hour in 2026** — treat that as a dated snapshot and recompute against your own contract — that is 4 GPUs × 48 h × ~$2.50 ≈ **$480 burned to enforce a fairness rule nobody wanted enforced that way**. Scale it to 128 GPUs and a 30% average idle-behind-quota rate and you are looking at roughly 128 × 0.30 × 730 h × $2.50 ≈ **$70k/month**, which is the number that gets a platform team headcount.

What you want instead is a **queue**: work is held, ordered, and admitted as capacity frees, so the fleet stays hot and teams get fairness instead of HTTP 403s. That is Kueue, and it is the batch-admission layer for GPU Kubernetes in production rather than a lab curiosity — CoreWeave (named in this module's target JDs) runs it on CoreWeave Kubernetes Service for AI-lab customers, Netflix runs it across millions of batch workloads, and IBM Research's Vela/Blue Vela GPU clusters use its queueing and cohort model explicitly because, in their framing, the problem is not getting more GPUs but getting more out of the ones they have.

An interviewer at any GPU-fleet operator can reasonably expect you to define ClusterQueue, LocalQueue, ResourceFlavor, and Workload cold, and to explain *why* Kueue queues instead of rejects. Fumbling that is a strong negative signal for a role whose description says "quota enforcement, fairness, pre-emption."

## What's new here (calibration)

- **You already know controllers, informers, webhooks, and CRDs (module 02).** Kueue is not a new paradigm — it is a well-factored Go controller applying a pattern you have already studied to one new domain object and one native-Kubernetes lever. Reconcile loops are not re-taught.
- **You already know `ResourceQuota` (module 04).** What it *is* is not re-taught; what is new is the precise mechanical reason it cannot queue, and where it still belongs alongside Kueue.
- **[L2](02-gang-scheduling.md) gave you the one-paragraph version of Kueue's suspend model.** This lesson replaces every hand-wave in it with the actual field, the actual default, and the actual code path.

Genuinely new:

- **The API version has moved.** Kueue **v0.16 switched the storage version to `kueue.x-k8s.io/v1beta2`**; `v1beta1` is still *served* but is marked `deprecated: true` in the CRDs with the warning "This version is deprecated. Use v1beta2 instead." Several field names changed with it — most consequentially `ClusterQueue.spec.cohort` became **`spec.cohortName`**. Every manifest in this lesson and in [L4](04-kueue-cohorts-borrowing-preemption.md) is v1beta2.
- **The admission cycle as an ordered algorithm**, not a black box: six numbered phases in `Scheduler.schedule()`, a flavor-assignment walk with documented tie-breaks, and a four-key sort that decides who goes first.
- **The status surface as a debugging instrument** — the exact `QuotaReserved` / `Admitted` / `Evicted` / `Preempted` reason strings the controller writes, and what each one tells you to do next.
- **`AdmissionFairSharing`**, beta-and-default since **v0.15**, which is *within-*ClusterQueue fairness and is easy to confuse with the *cohort-level* `fairSharing` in L4. They are different knobs solving different problems, and it requires a per-ClusterQueue opt-in that most write-ups omit.

## Core concepts

### 1. Why `ResourceQuota` cannot queue — the mechanism, not the slogan

`ResourceQuota` is a **built-in admission plugin** in kube-apiserver. Its entire life happens inside one API request:

```
   client: kubectl create -f job.yaml
      │
      ▼
   kube-apiserver
      ├─ authn / authz
      ├─ mutating admission webhooks
      ├─ schema validation
      ├─ validating admission plugins ── ResourceQuota plugin
      │     · find ResourceQuota objects in this namespace
      │     · sum the COMMITTED usage already recorded in their status
      │     · add this object's requests
      │     · over the hard limit?  →  reject the whole request
      │                                403 Forbidden: exceeded quota:
      │                                team-a-quota, requested: requests.nvidia.com/gpu=8,
      │                                used: requests.nvidia.com/gpu=4, limited: requests.nvidia.com/gpu=4
      ▼
   etcd write
```

Four properties fall out of that placement, and each one is a reason it cannot be a queue:

1. **It is synchronous and terminal.** The decision must be made before the API call returns. There is nowhere to put a "not yet." The only outcomes are *stored* and *rejected*.
2. **It has no memory of rejected work.** Nothing is persisted, so nothing can be ordered, prioritised, or retried by the platform. Ordering becomes an emergent property of whose retry loop is most aggressive.
3. **It is per-namespace.** The quota object lives in a namespace and counts that namespace's usage. There is no object that represents "the fleet," so there is no place to express "team A may use team B's idle capacity."
4. **It counts committed usage.** By the time usage is counted, pods exist. There is no notion of an intent that has not yet consumed anything.

Note the second-order effect on a *gang* job, which is the failure L1 and L2 spent two lessons on: `ResourceQuota` counts pods one at a time. A 16-pod job against a 12-pod-remaining quota does not fail cleanly — the Job controller creates pods until the 13th is rejected, leaving 12 pods holding GPUs and the job unable to make progress. **`ResourceQuota` can produce exactly the partial-placement deadlock of L1, at the quota layer instead of the node layer.**

Kueue moves the decision out of the request path entirely:

| | `ResourceQuota` | Kueue |
|---|---|---|
| Where it runs | kube-apiserver admission plugin | controller reconcile loop (`kueue-controller-manager`) |
| When it decides | synchronously, inside the create request | asynchronously, on every scheduling cycle |
| Scope of the pool | one namespace | one **ClusterQueue**, spanning many namespaces, groupable into a cohort |
| What it counts | committed pod usage | the `Workload`'s declared aggregate ask, before any pod exists |
| Outcome when over | `403 Forbidden`, work discarded | `QuotaReserved=False`, work **retained and ordered** |
| Granularity | per-pod | per-Workload (all pod sets together — the L2 atomicity guarantee) |
| Ordering / priority | none | `StrictFIFO` / `BestEffortFIFO`, priority classes, fair sharing |
| Borrowing across tenants | impossible | cohorts ([L4](04-kueue-cohorts-borrowing-preemption.md)) |
| Cost when the fleet is idle | idle GPUs stay idle | idle GPUs get lent out |

**They are complementary, and the right posture is to keep both.** Keep a coarse `ResourceQuota` per namespace as a blast-radius ceiling — it stops a runaway controller from creating 10 000 objects and is enforced even for work that bypasses Kueue. Use Kueue for the scheduling economics. If you set `ResourceQuota` tightly *and* use Kueue, you get a confusing hybrid where Kueue admits a Workload, unsuspends the Job, and the Job controller then gets 403s creating the pods — an admitted Workload with no pods and a very unhelpful error trail. Set namespace quotas well above any ClusterQueue's nominal quota, or omit them for Kueue-managed resources.

### 2. The lever: `Job.spec.suspend`, and the webhook that pulls it

Kueue does not invent an admission path. It uses a field the Job API already has.

`batch/v1.Job.spec.suspend` is a boolean, **stable since Kubernetes v1.24** (KEP-2232 "Suspend Job": alpha v1.21, beta v1.22, stable v1.24). Its semantics in the Job controller:

- While `suspend: true`, the controller creates **no pods**, and **deletes** any pods that already exist for the Job, resetting the Job's active count.
- The Job's `.status.startTime` is cleared while suspended, so the `activeDeadlineSeconds` clock does not run.
- Flipping `suspend: false` makes the controller create pods normally — including recreating pods it deleted on the way in.

That last property is what makes suspension usable as an *eviction* primitive as well as an admission gate, which is how preemption works in [L4](04-kueue-cohorts-borrowing-preemption.md): re-suspending a running Job deletes its pods and releases the GPUs, and the Workload goes back into the queue.

Kueue's mutating webhook (`BaseWebhook.Default`, `pkg/controller/jobframework/base_webhook.go`) runs three defaulting steps on every create of a supported job type:

```go
// pkg/controller/jobframework/base_webhook.go — the defaulting path, abridged
func (w *BaseWebhook[T]) Default(ctx context.Context, obj T) error {
    job := w.FromObject(obj)
    if w.IntegrationManager != nil {
        // 1. If the namespace has a LocalQueue literally named "default" and the
        //    object carries no queue-name label, stamp the label on.
        if err := w.IntegrationManager.ApplyDefaultLocalQueue(ctx, w.Client, job.Object(),
            w.Queues.DefaultLocalQueueExist, w.ManagedJobsNamespaceSelector); err != nil {
            return err
        }
        // 2. Apply a default WorkloadPriorityClass, if that feature gate is on.
        w.IntegrationManager.ApplyDefaultWorkloadPriorityClass(ctx, w.Client, job.Object())
        // 3. Decide whether this job is Kueue-managed and, if so, force suspend=true.
        if err := w.IntegrationManager.ApplyDefaultForSuspend(ctx, job, w.Client,
            w.ManageJobsWithoutQueueName, w.ManagedJobsNamespaceSelector); err != nil {
            return err
        }
    }
    ApplyDefaultForManagedBy(job, w.Queues, w.Cache, log)
    return nil
}
```

Step 3 is the lever. `ApplyDefaultForSuspend` asks `WorkloadShouldBeSuspended(...)` and, if the answer is yes and the job is not already suspended, calls `job.Suspend()` — **overriding whatever `spec.suspend` the user submitted.** You cannot opt out by setting `suspend: false`; the webhook wins.

Three configuration knobs decide "is this job Kueue-managed," and getting them wrong is the most consequential misconfiguration in this whole lesson:

| Knob | Where | Effect |
|---|---|---|
| `kueue.x-k8s.io/queue-name` label | on the Job | names the LocalQueue. Present ⇒ managed. |
| `manageJobsWithoutQueueName` | Kueue Configuration | if `true`, jobs **without** the label are managed too |
| `managedJobsNamespaceSelector` | Kueue Configuration | restricts the above to namespaces matching a label selector |
| a LocalQueue named `default` | in the namespace | the webhook stamps `queue-name: default` onto unlabelled jobs (`constants.DefaultLocalQueueName = "default"`) |

**The default posture — `manageJobsWithoutQueueName: false`, no `default` LocalQueue — means an unlabelled Job bypasses Kueue silently.** It is never suspended, never gets a Workload, and is scheduled straight through by kube-scheduler as if Kueue did not exist. No error, no event, no rejected admission: just a job quietly consuming GPUs outside the accounting system your showback report is built on. On a shared fleet this is the difference between a quota system and the appearance of one. The two robust fixes are creating a LocalQueue named `default` in every tenant namespace, or setting `manageJobsWithoutQueueName: true` scoped by `managedJobsNamespaceSelector` so system namespaces are exempt.

### 3. The object model, and one Workload's path through it

Five object kinds, three of which the platform team owns, one the teams touch, and one that Kueue creates for you.

| Object | Scope | Owner | Role |
|---|---|---|---|
| **ResourceFlavor** | cluster | platform | names a *class of capacity* and binds it to real nodes via labels/taints |
| **ClusterQueue** | cluster | platform | **the quota pool** — how much of each resource exists, per flavor |
| **Cohort** | cluster | platform | groups ClusterQueues so they can lend to each other ([L4](04-kueue-cohorts-borrowing-preemption.md)) |
| **LocalQueue** | namespace | platform (used by teams) | the team-facing handle pointing at one ClusterQueue |
| **Workload** | namespace | **Kueue** | the queued unit: one per Job, carrying the aggregate ask |

```
  THE KUEUE OBJECT MODEL, WITH ONE WORKLOAD'S ADMISSION PATH TRACED THROUGH IT
  ═══════════════════════════════════════════════════════════════════════════════════

   NAMESPACE: research                          CLUSTER SCOPE
   ─────────────────────                        ─────────────
                                                                    ┌──────────────────┐
   ┌────────────────────────┐                                       │     Cohort       │
   │ Job  llm-pretrain-3b   │                                       │  gpu-fleet       │
   │ labels:                │                                       │ (shared pool +   │
   │  kueue.x-k8s.io/       │                                       │  lend/borrow —   │
   │   queue-name: lq-res ──┼──┐                                    │  see L4)         │
   │ parallelism: 8         │  │                                    └────────▲─────────┘
   │ requests: 1 GPU/pod    │  │                                              │
   └────────────────────────┘  │                                   spec.cohortName
        │  ① mutating webhook  │                                              │
        │     forces           │                                    ┌─────────┴─────────┐
        │     spec.suspend=true│                                    │   ClusterQueue    │
        │     ⇒ ZERO PODS      │                                    │   cq-research     │
        ▼                      │                                    ├───────────────────┤
   ┌────────────────────────┐  │  ③ spec.queueName                  │ namespaceSelector │
   │ Workload               │  │     ┌──────────────────┐           │ queueingStrategy  │
   │  llm-pretrain-3b-a1b2  │◀─┘     │   LocalQueue     │           │ preemption        │
   ├────────────────────────┤        │   lq-research    │──────────▶│ admissionScope    │
   │ spec.queueName:        │───────▶│  ns: research    │ ④ spec.   │ resourceGroups:   │
   │   lq-research          │        ├──────────────────┤ clusterQ. │  coveredResources:│
   │ spec.podSets:          │        │ status:          │           │   [nvidia.com/gpu]│
   │  - name: main          │        │  pendingWorkloads│           │  flavors:         │
   │    count: 8            │        │  admittedWorkl.  │           │   ┌─────────────┐ │
   │    template: {...}     │        │  flavorsUsage    │           │   │ h100        │ │
   │ spec.priority: 100     │        └──────────────────┘           │   │ nominal: 16 │ │
   │ spec.active: true      │                                       │   ├─────────────┤ │
   ├────────────────────────┤                                       │   │ a100        │ │
   │ status.conditions:     │                                       │   │ nominal: 32 │ │
   │  QuotaReserved         │                                       │   └──────┬──────┘ │
   │  Admitted              │                                       └──────────┼────────┘
   │ status.admission:      │                                                  │
   │  clusterQueue:         │        ⑤ walk flavors IN ORDER,                  │
   │   cq-research          │           first one that fits wins    ┌──────────▼────────┐
   │  podSetAssignments:    │                                       │  ResourceFlavor   │
   │   - name: main         │◀──────────── ⑥ flavor chosen ─────────│  a100             │
   │     flavors:           │                                       ├───────────────────┤
   │      nvidia.com/gpu:   │                                       │ nodeLabels:       │
   │        a100            │                                       │  gpu-type: a100   │
   │     resourceUsage:     │                                       │ nodeTaints: [...] │
   │      nvidia.com/gpu: 8 │                                       │ tolerations: [...]│
   │     count: 8           │                                       │ topologyName: ... │
   └───────────┬────────────┘                                       └─────────┬─────────┘
               │                                                              │
               │ ⑦ Kueue sets Job.spec.suspend=false AND injects              │
               │    the flavor's nodeLabels as a nodeSelector +               │
               │    its tolerations onto the pod template  ◀──────────────────┘
               ▼
   ┌────────────────────────┐        ⑧ NOW 8 pods exist. kube-scheduler binds them
   │ Job  ... suspend:false │───────▶   to nodes carrying gpu-type=a100.
   └────────────────────────┘           (Which nodes, and whether they are on one
                                        NVLink domain, is L6's problem — Kueue's
                                        quota check is aggregate, not topological.)

   WHAT IS HELD WHILE WAITING, AT EACH LAYER:
     Job suspended            → nothing (no pods, no GPUs)
     Workload QuotaReserved   → quota accounting only (still no pods)
     Workload Admitted        → pods exist; kube-scheduler owns placement
```

Read the trace once more from the resource-accounting angle, because this is the property that distinguishes Kueue from L2's coscheduling plugin: **between ① and ⑦ the job consumes exactly zero cluster resources.** A thousand queued Workloads cost a thousand small etcd objects and nothing else. A thousand gangs parked at the coscheduling plugin's `Permit` hook would be holding a thousand gangs' worth of GPUs.

### 4. ResourceFlavor — binding abstract quota to real hardware

**The problem.** "This queue owns 8 GPUs" is meaningless on a heterogeneous fleet. Eight H100s and eight T4s are not interchangeable, and a job that assumed the former and got the latter will run 10× slower or OOM on device memory. Quota must therefore be denominated *per class of hardware*, and the class must be attached to real nodes.

A ResourceFlavor is that class. It is a cluster-scoped object with exactly four spec fields (`apis/kueue/v1beta2/resourceflavor_types.go`):

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: a100-80gb-ondemand
spec:
  # 1. nodeLabels — up to 8 entries. TWO jobs at once:
  #    (a) at admission, a PodSet may only be assigned this flavor if the pod
  #        template's nodeSelector / requiredDuringSchedulingIgnoredDuringExecution
  #        node affinity does not CONTRADICT these labels;
  #    (b) on admission, Kueue INJECTS these as a nodeSelector on the pod template,
  #        which is what actually steers pods onto the right machines.
  nodeLabels:
    nvidia.com/gpu.product: NVIDIA-A100-SXM4-80GB
    cloud.provider.com/lifecycle: on-demand

  # 2. nodeTaints — up to 8. Taints the matching nodes CARRY. A PodSet must already
  #    tolerate these to be eligible for this flavor. Only NoSchedule and NoExecute
  #    are evaluated; PreferNoSchedule is ignored. If the same taint also appears in
  #    .spec.tolerations below, it is NOT considered at admission.
  nodeTaints:
  - key: nvidia.com/gpu
    value: "present"
    effect: NoSchedule

  # 3. tolerations — up to 8. Extra tolerations Kueue ADDS to admitted pods.
  #    Use this to keep tenants from having to know about your taint scheme.
  tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "present"
    effect: NoSchedule

  # 4. topologyName — links this flavor to a Topology object for Topology-Aware
  #    Scheduling (L6). Immutable once set. Requires at least one nodeLabel.
  topologyName: gpu-rack-topology
```

Then, for spot capacity of the same hardware:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: a100-80gb-spot
spec:
  nodeLabels:
    nvidia.com/gpu.product: NVIDIA-A100-SXM4-80GB
    cloud.provider.com/lifecycle: spot
  nodeTaints:
  - key: cloud.provider.com/spot
    value: "true"
    effect: NoSchedule
  tolerations:
  - key: cloud.provider.com/spot
    operator: Equal
    value: "true"
    effect: NoSchedule
```

**Flavor design is the real operator decision in this lesson**, and it has one governing rule: *a flavor should name a class of capacity you would write a different quota number for.* Two dimensions almost always qualify:

- **Hardware model** — H100 vs A100 vs L40S. Different performance, different price, different quota.
- **Pricing/lifecycle** — on-demand vs spot vs reserved on the *same* model. Same silicon, different eviction risk and different $/hour, so different quota and different flavor ordering. This is the hook for L8's commitment ladder.

Dimensions that usually do *not* qualify: individual nodes (a flavor per node makes the quota arithmetic unreadable and every showback report noise), and topology zones (that is what `topologyName` and L6's TAS are for — do not encode racks as flavors).

**Flavor order inside a ClusterQueue is a policy, not a formality.** Kueue assigns the *first* flavor in `.spec.resourceGroups[*].flavors` that fits (§7). Listing `a100-80gb-spot` before `a100-80gb-ondemand` means "prefer the cheap capacity, fall through to expensive only when spot is exhausted" — the single highest-leverage cost line in a Kueue config, and it is expressed purely as list order.

### 5. ClusterQueue — the quota pool, field by field

The pool. Cluster-scoped, platform-owned, and the object your showback report is keyed on.

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: cq-research
spec:
  # Which namespaces may target this pool through a LocalQueue.
  # CAREFUL: null (field absent) is a NOTHING selector — no namespace is eligible
  # and every Workload is Inadmissible. The empty selector {} means ALL namespaces.
  namespaceSelector: {}

  # Ordering of pending Workloads. Default: BestEffortFIFO.
  #   StrictFIFO      — ordered by priority, then creationTimestamp. A head-of-line
  #                     Workload that does not fit BLOCKS newer ones that would.
  #   BestEffortFIFO  — same order, but a blocked head does not stop newer,
  #                     fitting Workloads from being admitted.
  queueingStrategy: BestEffortFIFO

  # Temporarily freeze this pool without deleting it.
  #   None (default) | Hold (stop admitting, let running finish)
  #   | HoldAndDrain (stop admitting AND evict what is running)
  stopPolicy: None

  # How the flavor walk behaves when a flavor requires borrowing or preemption.
  # Defaults shown; see §7.
  flavorFungibility:
    whenCanBorrow: MayStopSearch     # default
    whenCanPreempt: TryNextFlavor    # default

  # Opt in to within-pool fairness across LocalQueues (§10). Without this block,
  # AdmissionFairSharing does nothing for this ClusterQueue even though the
  # feature gate is on by default.
  admissionScope:
    admissionMode: UsageBasedAdmissionFairSharing

  # Cohort membership — L4. NOTE THE FIELD NAME: v1beta2 renamed
  # spec.cohort → spec.cohortName.
  cohortName: gpu-fleet

  resourceGroups:
  # A resource group ties resources that must come from the SAME flavor.
  # GPUs, the CPU that feeds them, and the memory they need all live on the same
  # node, so they belong in one group. Up to 16 groups; up to 64 flavors per group
  # and 256 flavors total; up to 64 resources per group, 256 covered total.
  - coveredResources: ["nvidia.com/gpu", "cpu", "memory"]
    flavors:
    # ORDER IS PREFERENCE. Cheap capacity first.
    - name: a100-80gb-spot
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 16          # the guaranteed floor this pool owns
      - name: "cpu"
        nominalQuota: 192
      - name: "memory"
        nominalQuota: 1500Gi
    - name: a100-80gb-ondemand
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 8
        borrowingLimit: 8         # L4: at most 8 MORE than nominal, from the cohort
        lendingLimit: 4           # L4: lend at most 4 out; always keep 4 reclaimable
      - name: "cpu"
        nominalQuota: 96
      - name: "memory"
        nominalQuota: 750Gi
  # A separate group for something not tied to the nodes — e.g. a licence server.
  - coveredResources: ["example.com/license"]
    flavors:
    - name: shared-license-pool
      resources:
      - name: "example.com/license"
        nominalQuota: 10
```

The fields worth dwelling on:

**`nominalQuota`** is "the quantity of this resource that is available for Workloads admitted by this ClusterQueue at a point in time." Two things it is *not*. It is not a description of the cluster — the API docs are explicit that you should set it to what is actually available after discounting system components and pods Kueue does not manage, and that in an autoscaled cluster it should account for capacity the autoscaler can add. And it is not automatically a ceiling: without a cohort it is both floor and ceiling, but inside a cohort it is a **floor you can exceed by borrowing** ([L4](04-kueue-cohorts-borrowing-preemption.md)). A `borrowingLimit` may only be set when `cohortName` is non-empty; the CRD enforces this with a CEL validation rule.

**Resource groups** are the mechanism for "these resources must come from the same machine." Every flavor in a group must list every covered resource in the group, in the same order — the CRD validates the count with `self.flavors.all(x, size(x.resources) == size(self.coveredResources))`. A flavor may belong to at most one group. Put GPU + CPU + memory in one group so a Workload cannot be assigned A100 quota for its GPUs and spot-CPU quota for its CPUs; put network licences, or anything genuinely detached from the nodes, in another.

**The `pods` pseudo-resource.** You can put `pods` in `coveredResources` and give it a `nominalQuota` to cap the number of pods admitted. `pods` is reserved: you cannot request it in a pod spec, and Kueue computes the count itself. On a GPU fleet this is a useful second axis — it stops a thousand-replica CPU-only sweep from filling the queue with objects even though it needs no GPUs.

**`namespaceSelector` is the field that bites.** Its documented default is `null`, which is "a nothing selector (no namespaces eligible)." Omit it and every Workload lands `Inadmissible`. `{}` means all namespaces. To scope it to one team, use the label the control plane puts on every namespace:

```yaml
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: research
```

**`queueingStrategy` is a fairness choice, not a performance one.** `BestEffortFIFO` (the default) keeps the fleet full: a 64-GPU job that does not fit does not stop 4-GPU jobs behind it from running. `StrictFIFO` prevents *starvation*: without it, a large job on a busy fleet can be perpetually skipped while a stream of small jobs is admitted around it. On a GPU fleet where the big jobs are the important ones, the starvation risk is real, and the honest answer is that `StrictFIFO` trades utilisation for predictability. §11 puts numbers on both.

The status side is what you graph:

```yaml
status:
  conditions:
  - type: Active            # false ⇒ this pool cannot admit at all; check `reason`
    status: "True"
    reason: Ready
  pendingWorkloads: 7
  reservingWorkloads: 3
  admittedWorkloads: 3
  flavorsReservation:       # quota RESERVED (phase 1 of admission)
  - name: a100-80gb-spot
    resources:
    - name: nvidia.com/gpu
      total: "16"
      borrowed: "0"
  flavorsUsage:             # quota USED by fully admitted Workloads (phase 2 done)
  - name: a100-80gb-spot
    resources:
    - name: nvidia.com/gpu
      total: "16"
      borrowed: "0"
```

`reservingWorkloads` versus `admittedWorkloads`, and `flavorsReservation` versus `flavorsUsage`, are the same two-phase split as the Workload conditions in §8: reservation happens first, admission completes after any AdmissionChecks pass. A persistent gap between the two means checks are pending, not that quota is short.

The `Active` condition is the first thing to read when *nothing* is being admitted from a pool. Its reasons are enumerated in the API and each is a specific misconfiguration: `FlavorNotFound` (a flavor named in `resourceGroups` has no ResourceFlavor object), `AdmissionCheckNotFound`, `AdmissionCheckInactive`, `TopologyNotFound`, `Stopped` (your own `stopPolicy`), `Terminating`, `Unknown`, and `Ready` when it is healthy.

### 6. LocalQueue — the tenancy seam, and the Workload

**LocalQueue** is a namespaced pointer at one ClusterQueue. It looks trivial and is load-bearing:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata:
  namespace: research
  name: lq-research
spec:
  clusterQueue: cq-research    # IMMUTABLE — CEL rule "self == oldSelf".
                               # Repointing a team at a different pool means
                               # delete and recreate, which is deliberate.
  stopPolicy: None             # same semantics as ClusterQueue.stopPolicy
  fairSharing:
    weight: "2"                # §10 — only meaningful under AdmissionFairSharing
```

It exists for three reasons:

1. **RBAC.** Teams get `create` on Jobs and `get`/`list` on LocalQueues in their own namespace. Only the platform team touches ClusterQueues and ResourceFlavors. The cluster-scoped quota model is therefore invisible and untouchable from a tenant namespace.
2. **Indirection.** Repointing a whole namespace at a different pool is a platform-side change; tenants keep writing the same label.
3. **Attribution.** Many LocalQueues can point at one ClusterQueue. That is the structure that lets several teams share one pool — and it is exactly the case `AdmissionFairSharing` exists to arbitrate (§10). Its status carries per-LocalQueue `pendingWorkloads`, `admittedWorkloads`, and `flavorsUsage`, which is your per-team usage series inside a shared pool.

**Workload** is the object Kueue creates and owns — one per Job, named after it with a hash suffix. This is what actually queues:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: Workload
metadata:
  name: job-llm-pretrain-3b-a1b2c
  namespace: research
  labels:
    kueue.x-k8s.io/job-uid: 4548c8bd-c399-4027-bb02-6114f3a8cdeb
  ownerReferences:
  - apiVersion: batch/v1
    kind: Job
    name: llm-pretrain-3b          # deleting the Job garbage-collects the Workload
spec:
  active: true                     # flip to false to evict + stop requeueing
  queueName: lq-research
  priority: 100                    # populated FROM the priority class, not set directly
  priorityClassRef:
    group: kueue.x-k8s.io          # or scheduling.k8s.io for a pod PriorityClass
    kind: WorkloadPriorityClass
    name: research-high
  maximumExecutionTimeSeconds: 86400   # auto-deactivate after 24 h of admitted time
  podSets:                         # 1..18 entries; IMMUTABLE after creation
  - name: main
    count: 8
    template:
      spec:
        containers:
        - name: trainer
          image: ghcr.io/example/trainer:v3
          resources:
            requests:
              cpu: "12"
              memory: 96Gi
              nvidia.com/gpu: "1"
        restartPolicy: Never
```

**How the ask is computed** — this is the arithmetic the whole quota system rests on, and it has three adjustments most people miss:

```
   Workload total for resource R  =  Σ over podSets ( count × per-pod request for R )

   where the per-pod request is taken AFTER:
     · LimitRange defaults are applied if the container omitted requests;
     · limits are treated as requests when only limits were specified
       (including pod-level `pod.spec.resources` on Kubernetes 1.32+);
     · RuntimeClass overhead is added.
   If the adjusted values violate a LimitRange, the Workload is marked Inadmissible.

   For the Workload above:
     nvidia.com/gpu :  8 × 1     =  8
     cpu            :  8 × 12    =  96
     memory         :  8 × 96Gi  =  768Gi
     pods           :  8                 ← computed by Kueue, not requested
```

A multi-role job — a Ray cluster, a PyTorchJob with a master and workers — produces several pod sets, and the totals sum across all of them. **All pod sets are admitted together or not at all**: this is where L2's atomicity guarantee actually lives in Kueue. (Partial admission exists as an alpha opt-in via `podSets[*].minCount` and the `PartialAdmission` feature gate, and only one pod set in a Workload may use it.)

Two spec fields are underused and worth knowing:

- **`spec.active`** — setting it to `false` evicts a running Workload and stops it from being requeued. This is the supported "stop this job now, keep the object" lever, and it is how an operator drains a specific tenant's work without deleting anything.
- **`spec.maximumExecutionTimeSeconds`** — a wall-clock budget on *admitted* time, accumulated across admit/evict cycles in `status.accumulatedPastExecutionTimeSeconds`. On a shared GPU fleet this is the cheapest possible guard against a hung job holding 8 GPUs for a week: set it and the Workload is automatically deactivated.

### 7. The admission cycle, as an algorithm

Kueue's scheduler runs a loop. One iteration is `Scheduler.schedule()` (`pkg/scheduler/scheduler.go`), whose six phases are numbered in the source:

```
  ONE SCHEDULING CYCLE
  ═════════════════════════════════════════════════════════════════════════════

  1. HEADS          Take the head Workload of every ClusterQueue's ordered queue.
                    (Blocks while all queues are empty — no busy-wait.)

  2. SNAPSHOT       Build an immutable snapshot of the cache: every ClusterQueue's
                    quota, current usage, cohort tree, and (if AdmissionFairSharing
                    is on) the usage ledger. The whole cycle reasons about THIS
                    snapshot, so decisions inside a cycle are mutually consistent.

  3. NOMINATE       For each head Workload, compute an ASSIGNMENT: which flavor for
                    each resource of each pod set, whether it requires borrowing,
                    and — if it cannot fit — what would have to be preempted (L4).
                    Workloads with no possible assignment become "inadmissible
                    entries" and are requeued with a reason.

  4. ORDER          Sort the nominated entries. Four keys, in order:
                       a. entries that ALREADY have quota reserved (second pass)
                       b. entries that fit under NOMINAL quota, before borrowers
                       c. higher priority first  (feature gate PrioritySortingWithinCohort,
                          on by default; disable it and this key becomes creation time)
                       d. FIFO on eviction-or-creation timestamp
                    Under Fair Sharing (L4) this iterator is replaced by one that
                    orders on dominant resource share instead.

  5. ADMIT          Walk the ordered entries and process each: re-check the
                    assignment against the snapshot as it mutates, issue any
                    preemptions, then reserve quota and mark the entry assumed.
                    At most ONE borrowing admission per cohort per cycle, so a
                    single cohort cannot be drained by one queue's head in one pass.

  6. REQUEUE        Every entry that was not admitted goes back on its queue with
                    an updated status reason.
```

**Phase 3 in detail — the flavor walk.** For each resource of each pod set, Kueue goes through the flavors of the relevant resource group *in the order they are listed in the ClusterQueue*, and asks whether the request fits. The documented fit rule is:

```
  A pod set's request for resource R fits flavor F in ClusterQueue C if:

    (1) request ≤ unused nominalQuota for (F, R) in C                     ← plain fit
   OR
    (2) request ≤ Σ unused nominalQuota for (F, R) across C's cohort      ← borrowing
   AND
    (3) request ≤ unused (nominalQuota + borrowingLimit) for (F, R) in C

  When (2) and (3) hold but (1) does not, that is BORROWING (L4).
  A ClusterQueue may only borrow for flavors it itself defines, and for each
  pod-set resource it may borrow for only ONE flavor.
```

Two additional filters apply before a flavor is even considered: the pod template's `nodeSelector` / required node affinity must not contradict the flavor's `nodeLabels`, and the pod template must tolerate the flavor's `nodeTaints` (unless the flavor's own `tolerations` already cover them).

`flavorFungibility` controls what happens when a flavor *could* work but only by borrowing or preempting:

| Field | Values | Default | Meaning |
|---|---|---|---|
| `whenCanBorrow` | `MayStopSearch`, `TryNextFlavor` | `MayStopSearch` | if this flavor fits only by borrowing, stop here (default) or keep looking |
| `whenCanPreempt` | `MayStopSearch`, `TryNextFlavor` | `TryNextFlavor` | if this flavor fits only by preempting, stop here or keep looking (default) |
| `preference` | `BorrowingOverPreemption`, `PreemptionOverBorrowing` | `BorrowingOverPreemption` | only settable when **both** of the above are `TryNextFlavor`; breaks ties among feasible assignments |

(`Borrow` and `Preempt` are deprecated aliases for `MayStopSearch`.) A flavor that fits with **no** borrowing and **no** preemption is always selected immediately, regardless of these settings. Otherwise the default preference orders candidates `(Fit, NoBorrow) → (Fit, Borrow) → (Preempt, NoBorrow) → (Preempt, Borrow)`; `PreemptionOverBorrowing` swaps the middle two.

The practical reading for a GPU fleet: with the defaults, a Workload that fits on spot only by borrowing will take it rather than checking whether on-demand fits outright. If you would rather it fall through to the next (more expensive but uncontended) flavor, set `whenCanBorrow: TryNextFlavor`.

**Admission is two-phase.** Reserving quota is not the same as being admitted:

1. **Quota reservation** — the ClusterQueue's quota and flavors can accommodate the ask (and, with Topology-Aware Scheduling on, the physical topology can too). Condition `QuotaReserved=True`. Other Workloads can no longer use that quota.
2. **AdmissionChecks** — wait for every configured `AdmissionCheck` to reach `Ready`. Checks can be built-in (MultiKueue, ProvisioningRequest) or your own controller. Only when all `AdmissionCheckStates` are `Ready` does the Workload become `Admitted=True` and the Job get unsuspended.

A check that fails *temporarily* (say, a cloud capacity shortage behind a ProvisioningRequest) causes Kueue to **release the reserved quota immediately** and requeue with exponential backoff — so a Workload waiting on infrastructure does not sit on GPUs nobody can use. A check that is `Rejected` evicts the Workload, releases quota, and **deactivates** it; a human must set it active again.

### 8. The Workload state machine, and reading it to debug

Everything above produces one object whose status you read under pressure. Here is the machine, with what drives each edge.

```
  WORKLOAD LIFECYCLE — every edge labelled with what causes it
  ══════════════════════════════════════════════════════════════════════════════════════

        Job created with kueue.x-k8s.io/queue-name
                     │
                     │  mutating webhook: spec.suspend := true
                     ▼
          ┌──────────────────────┐
          │      (no object)      │
          └──────────┬───────────┘
                     │  Kueue's Job reconciler creates the Workload
                     ▼
   ╔═════════════════════════════════════════════════════════════════════════════════╗
   ║  PENDING            QuotaReserved=False   Admitted=False   Job.suspend=true      ║
   ║  ────────           pods: 0   GPUs held: 0                                       ║
   ║                                                                                  ║
   ║  Why it is here — read reason on the QuotaReserved condition:                     ║
   ║    WaitingForQuota ......... the pool is full; you are in line                    ║
   ║    PendingEvaluation ....... not yet reached in a scheduling cycle                ║
   ║    NoMatchingFlavor ........ no flavor's nodeLabels/taints match this pod spec    ║
   ║    ExceedsMaxQuota ......... asks for more than the pool can EVER provide         ║
   ║    TopologyPlacementFailed . TAS could not place it (L6)                          ║
   ║    WaitingForPreemptedWorkloads . preemption issued; waiting for victims to go    ║
   ║    Misconfigured ........... the config is wrong, not the capacity                ║
   ║    Inadmissible ............ LocalQueue/ClusterQueue missing or inactive          ║
   ║    AdmissionGated .......... held by an admission gate annotation                 ║
   ╚═══════════════════════════════╤═════════════════════════════════════════════════╝
                                   │  scheduler reserves quota (phase 1)
                                   │  status.admission is written:
                                   │    clusterQueue + podSetAssignments
                                   │    (flavors, resourceUsage, count)
                                   ▼
   ╔═════════════════════════════════════════════════════════════════════════════════╗
   ║  QUOTA RESERVED     QuotaReserved=True    Admitted=False   Job.suspend=true      ║
   ║  ──────────────     pods: 0   quota held: YES                                    ║
   ║  Waiting on AdmissionChecks. status.admissionChecks[] shows each check's state.  ║
   ║  Admitted=False reasons: UnsatisfiedAdmissionChecks | NoReservation              ║
   ║                          | PendingDelayedTopologyRequests                         ║
   ╚═══════════════════════════════╤═════════════════════════════════════════════════╝
                                   │  all AdmissionCheckStates == Ready
                                   │  Kueue sets Job.spec.suspend := false and injects
                                   │  the flavor's nodeSelector + tolerations
                                   ▼
   ╔═════════════════════════════════════════════════════════════════════════════════╗
   ║  ADMITTED           Admitted=True                          Job.suspend=false     ║
   ║  ────────           pods: created   GPUs: actually consumed                      ║
   ║  PodsReady=True once ≥ Σ podSets[*].count pods are Ready or Succeeded.           ║
   ║  waitForPodsReady (ON BY DEFAULT since v0.19.0): if PodsReady is not reached      ║
   ║  within timeout (default 30m), the Workload is EVICTED with reason                ║
   ║  PodsReadyTimeout and requeued with backoff (base 60s, capped at 3600s).          ║
   ╚═══╤════════════════════════════════════╤════════════════════════════════════════╝
       │                                    │
       │ Job completes/fails                │ something evicts it
       ▼                                    ▼
   ╔═══════════════════╗   ╔══════════════════════════════════════════════════════════╗
   ║  FINISHED         ║   ║  EVICTED   Evicted=True, reason ∈ {                        ║
   ║  Finished=True    ║   ║    Preempted ......... a Preempted condition gives detail  ║
   ║  quota released,  ║   ║                        (InClusterQueue | InCohortReclamation║
   ║  next Workload    ║   ║                         | InCohortFairSharing              ║
   ║  evaluated        ║   ║                         | InCohortReclaimWhileBorrowing)   ║
   ╚═══════════════════╝   ║    PodsReadyTimeout .. pods never became ready             ║
                           ║    AdmissionCheck .... a check flipped to False            ║
                           ║    ClusterQueueStopped / LocalQueueStopped                 ║
                           ║    Deactivated ....... spec.active=false                   ║
                           ║    NodeFailures ...... non-recoverable node failures       ║
                           ║  }                                                          ║
                           ║  Job re-suspended ⇒ pods DELETED ⇒ GPUs released           ║
                           ╚═══════════════════╤══════════════════════════════════════╝
                                               │ Requeued=True (reason BackoffFinished,
                                               │ Reactivated, …) unless deactivated
                                               ▼
                                        back to PENDING
```

**The debugging drill.** When someone says "my job isn't running," this is the sequence, and it takes about ninety seconds:

```bash
# 1. Does a Workload even exist? If not, the Job was never Kueue-managed (§2).
$ kubectl get workloads -n research
NAME                          QUEUE         RESERVED IN   ADMITTED   FINISHED   AGE
job-llm-pretrain-3b-a1b2c     lq-research   cq-research   True                  14m
job-hpo-sweep-7f3d1           lq-research                                       9m

# 2. Read the conditions. This is the single most informative command.
$ kubectl describe workload job-hpo-sweep-7f3d1 -n research
...
Status:
  Conditions:
    Type:     QuotaReserved
    Status:   False
    Reason:   WaitingForQuota
    Message:  couldn't assign flavors to pod set main: insufficient unused quota
              for nvidia.com/gpu in flavor a100-80gb-ondemand, 4 more needed
  Resource Requests:
    Name:  main
    Resources:
      Cpu:             96
      Memory:          768Gi
      nvidia.com/gpu:  8
Events:
  Type    Reason    Age   From             Message
  Normal  Pending   9m    kueue-admission  couldn't assign flavors to pod set main: ...

# 3. Is the pool healthy, or is this a misconfiguration wearing a capacity costume?
$ kubectl get clusterqueue cq-research -o yaml | yq '.status'
conditions:
  - type: Active
    status: "True"
    reason: Ready
pendingWorkloads: 7
reservingWorkloads: 3
admittedWorkloads: 3
flavorsUsage:
  - name: a100-80gb-ondemand
    resources:
      - name: nvidia.com/gpu
        total: "8"
        borrowed: "0"

# 4. Where in line is it? (StrictFIFO especially — the head may be blocking you.)
$ kubectl get workloads -n research \
    -o custom-columns=NAME:.metadata.name,PRIO:.spec.priority,\
CREATED:.metadata.creationTimestamp,RESERVED:'.status.conditions[?(@.type=="QuotaReserved")].status' \
    --sort-by=.metadata.creationTimestamp
```

**Reason-to-action mapping**, which is the thing worth memorising:

| Condition + reason | What is actually wrong | What to do |
|---|---|---|
| `QuotaReserved=False` / `WaitingForQuota` | nothing; the pool is full | wait, raise quota, or enable borrowing (L4) |
| `QuotaReserved=False` / `ExceedsMaxQuota` | the ask exceeds what the pool can *ever* provide | the job will never run — shrink it or raise `nominalQuota` |
| `QuotaReserved=False` / `NoMatchingFlavor` | pod's nodeSelector/affinity/tolerations do not match any flavor | fix the pod spec or the ResourceFlavor labels |
| `QuotaReserved=False` / `Inadmissible` | LocalQueue or ClusterQueue missing or inactive | check `ClusterQueue.status.conditions[Active].reason` |
| `QuotaReserved=True`, `Admitted=False` / `UnsatisfiedAdmissionChecks` | quota is yours; a check is pending | inspect `status.admissionChecks[]` |
| `Evicted=True` / `PodsReadyTimeout` | admitted, but pods never became ready in 30 min | image pulls, device-plugin failures, unschedulable pods |
| `Evicted=True` / `Preempted` | something took the quota back | read the `Preempted` condition's reason (L4) |
| `Evicted=True` / `Deactivated` | `spec.active=false`, or a rejected check, or exec-time budget | set `spec.active=true` after fixing the cause |
| No Workload object at all | the Job was never Kueue-managed | missing `queue-name` label (§2) |

### 9. What Kueue does *not* do, and why L6 exists

Kueue's quota check is **aggregate arithmetic**. It knows the pool has 8 unused GPUs of flavor `a100-80gb-ondemand`. It does **not** know, at reservation time, whether those 8 are one node with 8 free GPUs or eight nodes with one each. Admission therefore succeeds and kube-scheduler is then asked to place a pod that needs 8 GPUs on one node — which it cannot.

Three mitigations, in increasing order of correctness:

1. **`waitForPodsReady`** — a safety net, not a fix. Since **v0.19.0 it is on by default** for new installs and for installs that do not configure it explicitly, with a 30-minute `timeout` and a 30-minute `recoveryTimeout`, `blockAdmission: false`, and requeue backoff starting at 60 s and capped at 3600 s. If pods do not all become ready in time, the Workload is evicted with `PodsReadyTimeout` and requeued. That converts a silent stuck job into a loud, bounded retry — but it burns up to 30 minutes of a GPU node's time per attempt.
2. **Layer coscheduling underneath** ([L2](02-gang-scheduling.md)) — Kueue admits against quota, the plugin guarantees the pods land atomically.
3. **Topology-Aware Scheduling** ([L6](06-topology-aware-placement.md)) — Kueue's own answer, where the ResourceFlavor's `topologyName` and the pod set's `topologyRequest` make placement part of the admission decision rather than a downstream hope. This is the correct fix and the reason L6 exists.

Being able to state this limitation crisply is a genuine seniority signal: it is the difference between "we use Kueue" and "we use Kueue, and here is the class of failure it structurally cannot prevent."

### 10. `AdmissionFairSharing` — fairness *inside* one pool

Everything so far governs *whether* a Workload is admitted against a fixed pool. It says nothing about *order* when several Workloads in the **same** ClusterQueue, submitted through **different LocalQueues** — i.e. different teams sharing one pool — are all eligible at once. Priority and FIFO answer that badly: a team that submits early and often wins forever, and history is free.

`AdmissionFairSharing` closes that gap. It is **beta and enabled by default since Kueue v0.15** (feature gate `AdmissionFairSharing`). The mechanism:

- Kueue maintains a **decaying weighted average of each LocalQueue's resource consumption**, exposed at `LocalQueue.status.fairSharing.admissionFairSharingStatus.consumedResources` alongside a `lastUpdate` timestamp.
- When multiple pending Workloads in a ClusterQueue could all fit, Kueue admits the one whose **LocalQueue has consumed less** first.
- An **entry penalty** (since v0.13.0) is added to a LocalQueue's usage the moment one of its Workloads is admitted, rather than waiting for the next sampling interval. Without it, a tenant could submit 100 Workloads in one burst and have all of them admitted before the statistics caught up. With it, each admission immediately deprioritises the next.

**It requires a per-ClusterQueue opt-in that is easy to miss.** The feature gate being on by default does *not* turn it on for your pool:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: cq-research
spec:
  admissionScope:
    admissionMode: UsageBasedAdmissionFairSharing   # or NoAdmissionFairSharing
  # ...
```

and it is tuned cluster-wide in the Kueue Configuration:

```yaml
apiVersion: config.kueue.x-k8s.io/v1beta2
kind: Configuration
admissionFairSharing:
  usageHalfLifeTime: "168h"      # usage decays by half every 7 days
  usageSamplingInterval: "5m"    # how often consumedResources is refreshed (default 5m)
  resourceWeights:
    nvidia.com/gpu: 10.0         # a GPU-second counts 10× a CPU-second
    cpu: 1.0
```

Per-LocalQueue weights adjust entitlement:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata: { name: lq-research-interactive, namespace: research }
spec:
  clusterQueue: cq-research
  fairSharing:
    weight: "2"     # treated as if it used HALF as many resources → favoured
```

The decay is a standard exponential half-life. Concretely, with `usageHalfLifeTime: 168h`:

```
    usage(t) = usage(t₀) × 2^(−(t − t₀)/168h)   +  new consumption in the window

    A team that burned 5 000 GPU-hours this week carries, as effective usage:
      after  7 days with zero new usage : 2 500 GPU-hours
      after 14 days                     : 1 250
      after 21 days                     :   625
      after 28 days                     :   313

    So a team that hogged the fleet for a week is disadvantaged for roughly a
    month, tapering — not punished forever, and not forgiven overnight. Choosing
    usageHalfLifeTime IS choosing how long your fleet's memory is. Set it to a
    fraction of your planning cycle: 168h (a week) is a sane default for research
    teams on a sprint cadence; a few hours if your tenants are CI pipelines.
```

Observe it directly:

```bash
$ kubectl get localqueue lq-research -n research -o jsonpath='{.status.fairSharing}' | jq
{
  "admissionFairSharingStatus": {
    "consumedResources": { "nvidia.com/gpu": "1284", "cpu": "15408" },
    "lastUpdate": "2026-08-18T09:12:04Z"
  },
  "weightedShare": 0
}
```

**Do not confuse this with the cohort-level `fairSharing` in [L4](04-kueue-cohorts-borrowing-preemption.md).** They are different layers and both can touch the same Workload:

| | `AdmissionFairSharing` (this lesson) | Cohort `fairSharing` ([L4](04-kueue-cohorts-borrowing-preemption.md)) |
|---|---|---|
| Scope | LocalQueues **within one** ClusterQueue | ClusterQueues **within a cohort** |
| Acts on | pending Workloads | already-admitted Workloads |
| Mechanism | **admission ordering** | **preemption** |
| Currency | decayed historical consumption per LocalQueue | dominant resource share per ClusterQueue |
| Can it evict a running job? | **no** | **yes** |
| Enabled by | feature gate (default on) **+** `admissionScope` on the CQ | presence of `fairSharing` in the Kueue Configuration |

A single Workload can lose its place in the *admission* order because its LocalQueue has been consuming more than its siblings inside the ClusterQueue, and separately be a preemption *target* later because its ClusterQueue is over-share within the cohort. Conflating them is the fastest way to look like you skimmed the release notes.

### 11. Queue wait time: the arithmetic

A quota pool is a queueing system, and queueing systems have arithmetic. You should be able to answer "how long will my job wait?" with a number and an assumption list, not a shrug.

**Setup.** One ClusterQueue with `nominalQuota: 64` GPUs of one flavor. Jobs arrive with a mix of sizes:

| Class | GPUs/job | Mean runtime | Arrival rate λ | GPU-hours/hour demanded |
|---|---|---|---|---|
| small (HPO trial) | 1 | 0.5 h | 40/h | 40 × 1 × 0.5 = 20 |
| medium (fine-tune) | 8 | 4 h | 1.2/h | 1.2 × 8 × 4 = 38.4 |
| large (pretrain) | 32 | 12 h | 0.02/h | 0.02 × 32 × 12 = 7.68 |
| | | | **total** | **66.08 GPU-h/h** |

**Step 1 — utilisation.** Offered load divided by capacity:

```
    ρ = 66.08 GPU-h per hour ÷ 64 GPU-h available per hour = 1.033
```

ρ > 1. **The queue is unstable**: demand exceeds capacity, so the backlog grows without bound and average wait time is not a finite number. This is the first and most useful check, and it is the one people skip before arguing about scheduling policy. *No scheduler fixes ρ > 1.* Cut demand, add capacity, or accept an ever-growing queue.

**Step 2 — a stable version.** Drop the small-job rate to 24/h (12 GPU-h/h), giving 58.08 GPU-h/h and ρ = 58.08/64 = **0.9075**.

Treat the pool as a single server processing GPU-hours (a crude but standard first cut — it ignores packing, which makes it *optimistic*; §12 corrects for that). For an M/G/1 queue the Pollaczek–Khinchine formula gives mean waiting time in queue:

```
    W_q = ( λ · E[S²] ) / ( 2 · (1 − ρ) )

  where S is the service time of a job measured in GPU-hours of pool capacity,
  i.e. S = (GPUs × runtime) / 64.

    small : S = (1 × 0.5)/64  = 0.0078 h    λ = 24  /h
    medium: S = (8 × 4)/64    = 0.5    h    λ = 1.2 /h
    large : S = (32 × 12)/64  = 6.0    h    λ = 0.02/h
    total λ = 25.22 /h

    E[S]  = Σ (λᵢ/λ) · Sᵢ
          = (24/25.22)(0.0078) + (1.2/25.22)(0.5) + (0.02/25.22)(6.0)
          = 0.00742 + 0.02379 + 0.00476  =  0.03597 h     ✓ (λ·E[S] = 0.907 = ρ)

    E[S²] = (24/25.22)(0.0078²) + (1.2/25.22)(0.5²) + (0.02/25.22)(6.0²)
          = 0.0000579 + 0.011895 + 0.028549  =  0.040502 h²

    W_q   = (25.22 × 0.040502) / (2 × (1 − 0.9075))
          = 1.02146 / 0.185
          ≈ 5.52 hours of mean wait
```

**Three things to take from that number, none of which is the number itself:**

1. **Variance dominates.** Look at the `E[S²]` decomposition: large jobs are 0.08% of arrivals and contribute **70%** of `E[S²]` (0.0285 of 0.0405). Waiting time in a shared queue is driven by the *second* moment of the job-size distribution, so a handful of enormous jobs sets the wait time for everyone. This is the quantitative case for giving large jobs their own ClusterQueue rather than mixing sizes in one pool — and it is a far better interview answer than "big jobs are disruptive."

2. **The (1 − ρ) term is a cliff, not a slope.** Same mix, varying ρ:

   | ρ | 1 − ρ | W_q | multiple of 0.90 case |
   |---|---|---|---|
   | 0.70 | 0.30 | ≈ 1.70 h | 0.31× |
   | 0.80 | 0.20 | ≈ 2.55 h | 0.46× |
   | 0.90 | 0.10 | ≈ 5.11 h | 0.93× |
   | 0.95 | 0.05 | ≈ 10.2 h | 1.85× |
   | 0.98 | 0.02 | ≈ 25.5 h | 4.62× |

   Going from 90% to 98% utilisation buys 8% more throughput and costs **5× the wait**. That is the fundamental tension of this whole module: FinOps wants ρ near 1, researchers want W_q near 0, and the only ways out are reducing variance (separate pools by size), adding capacity, or borrowing idle capacity from a neighbour — which is exactly [L4](04-kueue-cohorts-borrowing-preemption.md).

3. **`StrictFIFO` versus `BestEffortFIFO` is visible in this model.** `BestEffortFIFO` lets the 24/h stream of 1-GPU jobs flow past a blocked 32-GPU job. Their mean service time is 0.0078 h, so they see almost no wait — but the 32-GPU job can be skipped indefinitely while the pool never has 32 free at once. `StrictFIFO` reserves the head position, so the large job's wait is bounded, at the cost of idling GPUs while the pool drains to 32 free. The honest framing: **`BestEffortFIFO` optimises the mean, `StrictFIFO` bounds the tail for large jobs.** On a fleet whose purpose is large training runs, bounding the tail is usually what you actually want, and the utilisation you give up is the price of that guarantee.

## Perspectives

**Developer/tenant.** From a researcher's seat, Kueue is nearly transparent: submit a normal `Job` with one extra label and either it starts or it shows as pending with a legible reason. That is a world away from `ResourceQuota`'s hard `403 exceeded quota` at `kubectl apply` time, which reads like a permissions bug rather than a capacity one and sends people to the wrong Slack channel. The one thing a tenant must internalise is that **`kubectl get jobs` is now the wrong instrument** — a suspended Job with zero pods looks identical whether it is second in line or structurally inadmissible. `kubectl get workloads` and the `QuotaReserved` reason are the tenant-facing truth.

**Operator/platform.** The day-to-day job is not writing YAML, it is **flavor and pool design**: how many flavors (per GPU SKU? per pricing model?), how many ClusterQueues, and how they map to teams. Too few flavors and quota cannot distinguish A100 capacity from H100 capacity; too many and both the admission logic and the showback report fragment into noise. The second discipline is guarding the boundary — `manageJobsWithoutQueueName` and a `default` LocalQueue per namespace, so a forgotten label is not a silent quota bypass.

**Controller/systems (module 02 callback).** Kueue is a textbook reconciler and one of the better "controller pattern in the wild" reading exercises available: a mutating webhook that forces `spec.suspend=true`, a Job reconciler that maintains a mirror object, and a scheduler loop that snapshots a cache and drives a state machine. The snapshot-per-cycle design is worth noting specifically — it is the same shape as kube-scheduler's own snapshot from [L1](01-why-default-scheduler-fails.md), for the same reason: decisions inside one cycle must be mutually consistent, and consistency is cheaper to get from an immutable copy than from locks.

**Economics/FinOps.** The suspend-not-reject model is what makes an oversubscribed fleet *safe* rather than merely tolerated. Without a queue, teams either over-provision idle floors defensively (waste) or get rejected and retry-storm (waste plus alert noise). With one, the platform can sell 100% of nominal capacity on paper and let the queue absorb the variance. And the ClusterQueue is the natural showback unit — `kueue_cluster_queue_resource_usage{cohort, cluster_queue, flavor, resource}` joined to a $/GPU-hour rate per flavor is a per-team bill, with the flavor dimension carrying the on-demand-versus-spot price difference for free.

## Real-world use cases

- **CoreWeave — Kueue on CoreWeave Kubernetes Service.** *What it shows:* the company named in this module's README, running the tool this lesson teaches, for AI-lab customers doing both training and batch inference. Their framing is the same as L1's: GPU training has all-at-once semantics that a pod-at-a-time scheduler cannot express, so you need an admission layer above the scheduler rather than a better scoring plugin. *Verification note:* `coreweave.com` is blocked by this environment's egress proxy; the claim is corroborated by the JD language quoted in the module README and by Kueue's own adopter listings, not by a page read this session.

- **Netflix — Kueue for batch compute.** *What it shows:* Kueue fully rolled out in production managing millions of batch workloads, replacing a homegrown scheduler, with a migration strategy of moving the largest and most complex tenant first to stress the system before the easy ones. *What to take from it:* the migration sequencing, not the scale. Moving the hardest tenant first is the opposite of most rollout instincts and is right for this class of system, because the failure modes that matter (quota bypass, head-of-line blocking, eviction storms) only appear under the hardest workload. *Verification note:* `netflixtechblog.com` is blocked from this environment; search-confirmed only.

- **IBM Research — Vela and Blue Vela.** *What it shows:* Kueue's queueing and cohort model on real multi-hundred-GPU research clusters (A100 on Vela, H100 on Blue Vela), adopted explicitly to raise *utilisation* rather than to add capacity — "the challenge is not about getting more GPUs, but getting more out of the GPUs they already have." That is this module's thesis stated by an operator of the exact fleet shape you are training for. *Verification note:* arXiv 2407.05467 and the KubeCon EU 2025 tutorial deck are the sources; neither was fetchable from this environment.

- **The upstream project's own defaults, as evidence.** *What it shows:* Kueue **v0.19.0 turned `waitForPodsReady` on by default** (30-minute timeout, 30-minute recovery timeout) for new installs and for existing installs that never configured it. That is a maintainer group deciding, on production evidence, that the *default* posture should be "evict and requeue a Workload whose pods never became ready" rather than "let it sit." *What to take from it:* §9's limitation — aggregate quota is blind to node layout — is common enough in the field that upstream made the safety net mandatory. *Verification note:* read directly from `CHANGELOG/CHANGELOG-0.19.md` in the cloned repository.

## Worked example

**Scenario.** 24 A100-80GB GPUs: 16 spot, 8 on-demand. Two teams — `research` and `product` — each owning half the on-demand capacity, both allowed to use spot. No borrowing yet; that is [L4](04-kueue-cohorts-borrowing-preemption.md). One namespace per team.

**Step 1 — flavors.**

```yaml
# 00-flavors.yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: a100-spot
spec:
  nodeLabels:
    nvidia.com/gpu.product: NVIDIA-A100-SXM4-80GB
    lifecycle: spot
  nodeTaints:
  - { key: lifecycle, value: spot, effect: NoSchedule }
  tolerations:
  - { key: lifecycle, operator: Equal, value: spot, effect: NoSchedule }
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: a100-ondemand
spec:
  nodeLabels:
    nvidia.com/gpu.product: NVIDIA-A100-SXM4-80GB
    lifecycle: on-demand
```

Note the asymmetry: spot nodes are tainted so nothing lands on them accidentally, and the flavor carries the matching toleration so tenants never have to know the taint exists. That is the flavor doing its job as an abstraction boundary.

**Step 2 — two pools, spot first.**

```yaml
# 01-clusterqueues.yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: cq-research
spec:
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: research
  queueingStrategy: BestEffortFIFO
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu", "cpu", "memory", "pods"]
    flavors:
    - name: a100-spot                      # CHEAP FIRST — this ordering is the cost policy
      resources:
      - { name: "nvidia.com/gpu", nominalQuota: 8 }
      - { name: "cpu",            nominalQuota: 96 }
      - { name: "memory",         nominalQuota: 750Gi }
      - { name: "pods",           nominalQuota: 100 }
    - name: a100-ondemand
      resources:
      - { name: "nvidia.com/gpu", nominalQuota: 4 }
      - { name: "cpu",            nominalQuota: 48 }
      - { name: "memory",         nominalQuota: 375Gi }
      - { name: "pods",           nominalQuota: 50 }
---
# cq-product: identical, namespaceSelector on product, same quotas
```

**Step 3 — LocalQueues, one per team, plus a `default` alias so a forgotten label is not a bypass.**

```yaml
# 02-localqueues.yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata: { namespace: research, name: lq-research }
spec: { clusterQueue: cq-research }
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata: { namespace: research, name: default }     # webhook stamps this on unlabelled Jobs
spec: { clusterQueue: cq-research }
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata: { namespace: product, name: lq-product }
spec: { clusterQueue: cq-product }
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata: { namespace: product, name: default }
spec: { clusterQueue: cq-product }
```

**Step 4 — a Job, and what Kueue makes of it.**

```yaml
# job-finetune.yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: finetune-
  namespace: research
  labels:
    kueue.x-k8s.io/queue-name: lq-research
    kueue.x-k8s.io/priority-class: research-normal
spec:
  parallelism: 8
  completions: 8
  suspend: true          # the webhook forces this anyway; setting it is honest, not required
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: trainer
        image: ghcr.io/example/trainer:v3
        resources:
          requests:
            cpu: "12"
            memory: 96Gi
            nvidia.com/gpu: "1"
```

Submit three of these back to back. The arithmetic: each asks 8 GPUs, 96 CPU, 768 Gi. `cq-research` has 8 spot GPUs and 4 on-demand.

```bash
$ for i in 1 2 3; do kubectl create -f job-finetune.yaml; done
job.batch/finetune-9v2kd created
job.batch/finetune-lqx7m created
job.batch/finetune-tt4hb created

$ kubectl get workloads -n research
NAME                       QUEUE         RESERVED IN   ADMITTED   FINISHED   AGE
job-finetune-9v2kd-6f21a   lq-research   cq-research   True                  6s
job-finetune-lqx7m-b03c8   lq-research                                       5s
job-finetune-tt4hb-11e4f   lq-research                                       4s
```

*(Representative transcript.)* Job 1 got all 8 spot GPUs. Job 2 needs 8 and only 4 on-demand remain — it cannot be split across flavors, because a pod set's resource is assigned exactly one flavor. Read why:

```bash
$ kubectl describe workload job-finetune-lqx7m-b03c8 -n research | sed -n '/Conditions/,/Events/p'
Status:
  Conditions:
    Last Transition Time:  2026-08-18T09:04:11Z
    Message:               couldn't assign flavors to pod set main: insufficient unused quota
                           for nvidia.com/gpu in flavor a100-spot, 8 more needed;
                           insufficient unused quota for nvidia.com/gpu in flavor
                           a100-ondemand, 4 more needed
    Reason:                WaitingForQuota
    Status:                False
    Type:                  QuotaReserved
```

Both flavors were tried, in list order, and both were reported. `WaitingForQuota` (not `ExceedsMaxQuota`) tells you this job *will* run — the pool is big enough, it is just busy right now. Had someone asked for 16 GPUs, the reason would be `ExceedsMaxQuota` and the job would never run without a config change. **That single reason string is the difference between "wait" and "escalate."**

Now finish job 1 and watch the queue drain:

```bash
$ kubectl delete job finetune-9v2kd -n research
$ kubectl get workloads -n research -w
job-finetune-lqx7m-b03c8   lq-research                              1m
job-finetune-lqx7m-b03c8   lq-research   cq-research   True         1m21s
```

**You just watched a queue drain — the thing `ResourceQuota` structurally cannot do.**

**Step 5 — the showback join.** Sum admitted GPU-seconds per ClusterQueue and per flavor, and multiply by the flavor's rate. Because flavor is a metric label, the on-demand/spot price split falls out for free:

```promql
# GPU-hours consumed per ClusterQueue per flavor over the last 24h
sum by (cluster_queue, flavor) (
  avg_over_time(
    kueue_cluster_queue_resource_usage{resource="nvidia.com/gpu"}[24h]
  )
) * 24
```

With, say, $2.50/GPU-h on-demand and $0.90/GPU-h spot, a day where `cq-research` averaged 8 spot and 3 on-demand GPUs bills as:

```
    spot      : 8 × 24 h × $0.90  =  $172.80
    on-demand : 3 × 24 h × $2.50  =  $180.00
    ─────────────────────────────────────────
    cq-research daily             =  $352.80

  And the counterfactual that justifies the flavor ordering: if the same work
  had run entirely on-demand, 11 × 24 × $2.50 = $660.00 — so "spot listed first"
  saved $307.20/day, ≈ $9.2k/month, from one line of YAML ordering.
  (Rates are illustrative; substitute your own contract.)
```

That table, keyed on ClusterQueue and split by flavor, is the deliverable's showback report. Other useful series for the same report: `kueue_cluster_queue_nominal_quota` (what they own), `kueue_pending_workloads` and `kueue_admitted_active_workloads` (queue depth and concurrency), and `kueue_quota_reserved_wait_time_seconds` / `kueue_admission_wait_time_seconds` (the §11 wait time, measured rather than modelled).

## Practice

You do not need real GPUs. Advertise a fake `nvidia.com/gpu` extended resource on kind nodes so the quota arithmetic is real while pods schedule on CPU.

**Feeds:** [Kueue setup + per-queue showback](../practice/kueue-showback/README.md).

```bash
# 1. Cluster. Kueue v0.19 recommends Kubernetes 1.34+.
kind create cluster --name kueue-lab --image kindest/node:v1.34.0

# 2. Fake GPU capacity: patch the node status with an extended resource.
#    Extended resources are opaque integers the scheduler tracks; nothing has to exist.
NODE=$(kubectl get nodes -o name | head -1 | cut -d/ -f2)
kubectl proxy --port=8001 &
curl -s --header "Content-Type: application/json-patch+json" -XPATCH \
  http://localhost:8001/api/v1/nodes/$NODE/status \
  --data '[{"op":"add","path":"/status/capacity/nvidia.com~1gpu","value":"24"}]'
kubectl label node $NODE nvidia.com/gpu.product=NVIDIA-A100-SXM4-80GB lifecycle=on-demand --overwrite

# 3. Install Kueue. Check the releases page for the newest tag;
#    v0.19.1 is the latest at time of writing.
KUEUE_VERSION=v0.19.1
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/kueue/releases/download/${KUEUE_VERSION}/manifests.yaml
kubectl -n kueue-system wait --for=condition=Available \
  deploy/kueue-controller-manager --timeout=300s

# 4. Confirm which API version you are on — v1beta2 since Kueue v0.16.
kubectl get crd clusterqueues.kueue.x-k8s.io \
  -o jsonpath='{range .spec.versions[*]}{.name}{" served="}{.served}{" storage="}{.storage}{"\n"}{end}'

# 5. Namespaces and the manifests from the Worked example.
kubectl create ns research; kubectl create ns product
kubectl apply -f 00-flavors.yaml -f 01-clusterqueues.yaml -f 02-localqueues.yaml
```

1. **Trace the suspend lever.** Submit `job-finetune.yaml` with `suspend: false` explicitly set. Show with `kubectl get job -o jsonpath='{.spec.suspend}'` that the webhook flipped it to `true` anyway, and that `kubectl get pods -n research` returns nothing. **This is the mechanism of the whole lesson in one observation.**

2. **Watch a queue drain.** Submit three 8-GPU Jobs. Capture `kubectl get workloads -n research` showing one admitted and two pending, the `describe` output with the `QuotaReserved=False` / `WaitingForQuota` condition and its per-flavor message, then delete the admitted Job and capture the second Workload flipping to admitted with its Job unsuspending.

3. **Produce three different failure reasons deliberately.** (a) Submit a Job asking for more GPUs than the pool's total → confirm `ExceedsMaxQuota`. (b) Add a `nodeSelector` to the pod template that contradicts every flavor's `nodeLabels` → confirm `NoMatchingFlavor`. (c) Delete a ResourceFlavor that a ClusterQueue references → confirm `ClusterQueue.status.conditions[Active]` goes `False` with reason `FlavorNotFound`, and that Workloads then report `Inadmissible`. **Recreating each of these from memory is the actual skill.**

4. **Demonstrate the silent bypass, then close it.** Submit a Job with **no** `kueue.x-k8s.io/queue-name` label into a namespace that has no LocalQueue named `default`. Show that no Workload exists and the pods ran anyway. Then create the `default` LocalQueue, resubmit, and show the Job is now suspended and queued. Write down which of the two fixes (`default` LocalQueue vs `manageJobsWithoutQueueName`) you would choose for a real fleet and why.

5. **Prove the flavor walk order.** Give the ClusterQueue both flavors (label a second kind node `lifecycle=spot`, or fake it on the same node with two label sets and two nodes). Submit a job that fits in either. Show from `status.admission.podSetAssignments[0].flavors` which flavor was chosen, then reverse the flavor order in the ClusterQueue and show the choice flips. **This is your evidence that list order is the cost policy.**

6. **Turn on `AdmissionFairSharing` and observe it.** Add `admissionScope.admissionMode: UsageBasedAdmissionFairSharing` to `cq-research`, add a second LocalQueue `lq-research-batch` in the same namespace pointing at the same ClusterQueue, and drive uneven consumption through them for a few minutes. Read `kubectl get lq -n research -o jsonpath='{.items[*].status.fairSharing}'` and show the two `consumedResources` values diverging. Then submit one job from each simultaneously with a single slot free and record which is admitted first, with timestamps.

7. **Do the queueing arithmetic for your own numbers.** Using §11's method, compute ρ, `E[S]`, `E[S²]`, and `W_q` for a job mix you pick. State the assumption you are most uncomfortable with (packing efficiency is the right answer) and say which direction it biases the estimate.

**Acceptance (feeds the deliverable):**
- Kueue controller `Available`; the CRD version output from step 4 pasted, showing `v1beta2 storage=true`.
- Two ClusterQueues, two ResourceFlavors, and four LocalQueues applied cleanly.
- A captured **suspend → admit** transition: the `QuotaReserved=False` condition with its message, then `True` plus the `status.admission` block naming the ClusterQueue and the assigned flavor.
- **Three distinct failure reasons** captured verbatim (`ExceedsMaxQuota`, `NoMatchingFlavor`, and a `ClusterQueue` `Active=False` reason), each with one sentence on what you would do about it.
- The bypass demonstration and its fix.
- Your queue-wait arithmetic: ρ, `W_q`, and the ρ-sensitivity table for your own mix.

Keep these manifests — [L4](04-kueue-cohorts-borrowing-preemption.md) puts both ClusterQueues in a cohort and turns these hard floors into borrowable quota.

## Common pitfalls

1. **Writing `v1beta1` manifests against a current Kueue.** *Symptom:* deprecation warnings, or a `spec.cohort` field that silently does nothing. *Mechanism:* Kueue **v0.16 moved the storage version to `v1beta2`** and the CRD marks `v1beta1` `deprecated: true`. Some fields were renamed in the move — most importantly `ClusterQueue.spec.cohort` → **`spec.cohortName`**. `v1beta1` is still served and converted, so old manifests appear to work while the field you meant to set is not the field you set. Check with the CRD version query in Practice step 4.

2. **Omitting `namespaceSelector`.** *Symptom:* every Workload is `Inadmissible` and the ClusterQueue looks perfectly healthy. *Mechanism:* the documented default is `null`, which is a **nothing** selector — no namespace is eligible. `{}` means all namespaces. This is inverted from most people's intuition about empty selectors in Kubernetes.

3. **Treating `nominalQuota` as if it were `ResourceQuota`.** *Symptom:* a wave of "why didn't my job run" tickets on a freshly Kueue-enabled cluster. *Mechanism:* both are numbers, but one rejects and the other queues. Teams migrating expect a synchronous error and are confused by a Job that simply sits with zero pods. The fix is documentation and `kubectl get workloads`, not a config change.

4. **Letting unlabelled Jobs bypass Kueue.** *Symptom:* the showback report does not add up, and the fleet is busier than the ClusterQueue status says. *Mechanism:* with `manageJobsWithoutQueueName: false` (the default) and no `default` LocalQueue, an unlabelled Job is never suspended, never gets a Workload, and is scheduled by kube-scheduler directly. **It fails silently — no error, no event.** On a shared fleet this quietly converts your quota system into a suggestion.

5. **One ResourceFlavor per node.** *Symptom:* unreadable quota arithmetic and a showback report nobody can act on. *Mechanism:* a flavor is a *class* of capacity, and quota is denominated per flavor. Per-node flavors multiply the flavor count by your node count, fragment every quota number, and add a dimension to every metric. Flavor on hardware model and pricing model; leave topology to L6.

6. **Setting a tight namespace `ResourceQuota` alongside Kueue.** *Symptom:* an `Admitted=True` Workload whose Job has no pods, with `exceeded quota` errors in the Job controller's events. *Mechanism:* Kueue reserves quota and unsuspends; the Job controller then creates pods and kube-apiserver's `ResourceQuota` plugin rejects them. Two independent quota systems disagreeing, with the failure surfacing in the least obvious of the two. Keep `ResourceQuota` as a loose blast-radius ceiling well above any ClusterQueue's nominal quota.

7. **Assuming aggregate quota implies a feasible placement.** *Symptom:* an admitted Workload whose pods sit `Pending` on the scheduler, then get evicted 30 minutes later with `PodsReadyTimeout`. *Mechanism:* Kueue counts GPUs, not node layouts (§9). Eight free GPUs spread one-per-node satisfy a quota check for an 8-GPU job that needs them on one node. `waitForPodsReady` (on by default since v0.19.0) turns this into a bounded retry rather than a silent hang, but the real fix is TAS (L6) or coscheduling underneath (L2).

8. **Expecting `AdmissionFairSharing` to work because the feature gate is on.** *Symptom:* two LocalQueues in one pool, admission order still pure FIFO, `consumedResources` diverging with no effect. *Mechanism:* the gate is beta-and-default since v0.15, but the ClusterQueue must additionally set `admissionScope.admissionMode: UsageBasedAdmissionFairSharing`. Without that block the pool uses standard queue ordering.

9. **Conflating `AdmissionFairSharing` with cohort `fairSharing`.** *Symptom:* claiming, in an interview or a design doc, that fair sharing will evict a running job in a single-ClusterQueue setup. *Mechanism:* different scopes (LocalQueues within a pool vs ClusterQueues within a cohort) and different mechanisms (admission ordering vs preemption). `AdmissionFairSharing` **never** evicts anything.

10. **Reasoning about wait times without checking ρ.** *Symptom:* arguing about `StrictFIFO` versus `BestEffortFIFO` on a fleet where demand exceeds capacity. *Mechanism:* when ρ ≥ 1 the queue is unstable and mean wait is unbounded; no ordering policy changes that. Compute offered load ÷ capacity first. Scheduling policy redistributes wait; it does not create throughput.

## Self-check

**(a) Why does Kueue suspend jobs instead of rejecting them like `ResourceQuota`, and what does that buy?**
**Answer:** Because `ResourceQuota` runs as a kube-apiserver **admission plugin**, inside the create request, and must return a terminal answer before the request completes — so its only options are *store* and *reject with 403*. Nothing is persisted about rejected work, so there is no ordering, no priority, no fairness, and no retry the platform controls; it is also per-namespace and counts committed pod usage, so it cannot express a fleet-wide pool. Kueue moves the decision into a controller reconcile loop and holds the intent as a `Workload` object, using the native `batch/v1.Job.spec.suspend` field (stable since Kubernetes **v1.24**, KEP-2232) to keep the Job inert — the Job controller creates no pods while suspended and deletes any that exist. A queued Workload therefore consumes **zero** GPUs and zero pods. That buys ordering (`StrictFIFO`/`BestEffortFIFO`, priority, fair sharing), safe oversubscription, cross-namespace pooling, and borrowing (L4). Secondary but important: `ResourceQuota` counts pods one at a time, so a 16-pod job against a 12-pod-remaining quota gets 12 pods created and the 13th rejected — reproducing L1's partial-placement deadlock at the quota layer. Kueue evaluates all pod sets together.

**(b) What does a ResourceFlavor map to, and what happens if you get it wrong?**
**Answer:** It maps abstract quota to a concrete class of capacity via `nodeLabels` (up to 8), plus `nodeTaints` the nodes carry, `tolerations` Kueue adds to admitted pods, and optionally a `topologyName` for TAS. It does two jobs: at admission it *filters* — a pod set can only be assigned a flavor whose `nodeLabels` do not contradict its `nodeSelector`/required node affinity, and whose `nodeTaints` it tolerates — and on admission Kueue *injects* the flavor's `nodeLabels` as a `nodeSelector` and its `tolerations` onto the pod template, which is what actually steers pods onto the right machines. You need it because "8 GPUs" is ambiguous on heterogeneous hardware; quota is denominated per flavor so A100 and H100 capacity are counted separately. Getting it wrong in the two common directions: too few flavors and a job asking for A100s can be admitted against H100 quota; one flavor per node and the quota arithmetic and showback report fragment into unreadable noise. Flavor on things you would write a different quota number for — hardware model and pricing/lifecycle (spot vs on-demand) — and remember that **flavor order in the ClusterQueue is the cost policy**, since Kueue takes the first flavor that fits.

**(c) Walk one Workload from `kubectl create` to running pods, naming every object and every state transition.**
**Answer:** (1) The Job is created with `kueue.x-k8s.io/queue-name: lq-research`. (2) Kueue's mutating webhook runs `ApplyDefaultLocalQueue` (stamping `queue-name: default` if the namespace has a LocalQueue by that name and the label is absent), a default WorkloadPriorityClass if that gate is on, and `ApplyDefaultForSuspend`, which **forces `spec.suspend=true`** regardless of what you submitted — so zero pods are created. (3) Kueue's Job reconciler creates a namespaced `Workload` owned by the Job, with `spec.queueName`, `spec.podSets` (count plus pod template, immutable), and a priority populated from the priority class. The ask is `Σ over podSets (count × per-pod request)` **after** LimitRange defaults, limits-as-requests, and RuntimeClass overhead. (4) The Workload enters the LocalQueue, which points at a ClusterQueue via an immutable `spec.clusterQueue`. (5) On a scheduling cycle, Kueue snapshots the cache, nominates an assignment by walking the ClusterQueue's flavors in list order for the relevant resource group, orders the candidates (quota-reserved first, non-borrowing before borrowing, higher priority, then FIFO), and reserves quota — writing `status.admission` with the ClusterQueue and per-pod-set flavor assignments, and setting `QuotaReserved=True`. (6) Any AdmissionChecks must reach `Ready`; then `Admitted=True`. (7) Kueue sets `Job.spec.suspend=false` and injects the flavor's `nodeSelector` and tolerations. (8) The Job controller creates the pods and kube-scheduler binds them. (9) On completion, `Finished=True` and quota is released for the next Workload. Between (2) and (7) the job holds nothing but an etcd object.

**(d) A Workload has sat pending for an hour. Give the exact commands and fields you read, and what each possible answer means.**
**Answer:** First `kubectl get workloads -n <ns>` — **if no Workload exists at all, the Job was never Kueue-managed** (missing `queue-name` label with `manageJobsWithoutQueueName: false` and no `default` LocalQueue), and it is running outside your quota system. If it exists, `kubectl describe workload <name> -n <ns>` and read the `QuotaReserved` condition's `reason`: `WaitingForQuota` means the pool is genuinely full — wait, raise quota, or enable borrowing; `PendingEvaluation` means it has not been reached in a cycle yet; `ExceedsMaxQuota` means the ask exceeds what the pool can *ever* provide, so it will never run without a config change; `NoMatchingFlavor` means the pod's nodeSelector/affinity/tolerations match no flavor; `TopologyPlacementFailed` is TAS; `WaitingForPreemptedWorkloads` means preemption was issued and victims are still terminating; `Inadmissible` means the LocalQueue or ClusterQueue is missing or inactive; `Misconfigured` means the config, not the capacity, is wrong. If `QuotaReserved=True` but `Admitted=False`, read `Admitted`'s reason — `UnsatisfiedAdmissionChecks` — and then `status.admissionChecks[]` for the specific check. Then check the pool: `ClusterQueue.status.conditions[Active]`, whose `reason` may be `FlavorNotFound`, `AdmissionCheckNotFound`, `AdmissionCheckInactive`, `TopologyNotFound`, `Stopped`, or `Terminating`; plus `pendingWorkloads`, `reservingWorkloads`, `admittedWorkloads`, and `flavorsUsage`. Finally, under `StrictFIFO`, sort Workloads by priority and creation timestamp — the head of the line may be blocking you even though your job would fit.

**(e) Explain the two-phase admission cycle and why quota reservation is separate from admission.**
**Answer:** Phase 1, **quota reservation**: the scheduler snapshots the cache, computes a flavor assignment for each pod set, and — if the ClusterQueue's quota can cover the ask — writes `status.admission` and sets `QuotaReserved=True`. That quota is now unavailable to other Workloads. Phase 2, **AdmissionChecks**: every configured `AdmissionCheck` must reach `Ready`. Checks may be built-in (MultiKueue, ProvisioningRequest) or a custom controller. Only when all `AdmissionCheckStates` are `Ready` does `Admitted=True` and the Job get unsuspended. The separation exists because some admission criteria are slow and external — provisioning nodes from a cloud autoscaler, choosing a target cluster in a multi-cluster setup — and you want the quota decision made and recorded *before* paying that latency, while still not starting pods until the external precondition holds. The failure handling shows the design intent: a **temporary** check failure releases the reserved quota immediately and requeues with exponential backoff, so a Workload waiting on infrastructure does not sit on GPUs; a **rejected** check evicts, releases quota, and deactivates the Workload until a human sets `spec.active=true`. It also shows up in the ClusterQueue status as the `reservingWorkloads`/`admittedWorkloads` and `flavorsReservation`/`flavorsUsage` pairs — a persistent gap between them means checks are pending, not that quota is short.

**(f) A pool has `nominalQuota: 64` GPUs and offered load of 58 GPU-hours per hour. Estimate the mean queue wait, and say what you would change.**
**Answer:** ρ = 58/64 = **0.906** — stable, but only just. Using Pollaczek–Khinchine, `W_q = λ·E[S²] / (2(1−ρ))` where each job's service time is `(GPUs × runtime)/64`. With the §11 mix (24/h × 1 GPU × 0.5 h, 1.2/h × 8 GPU × 4 h, 0.02/h × 32 GPU × 12 h), `E[S²] ≈ 0.0405 h²` and λ = 25.22/h, giving **W_q ≈ 5.5 hours**. Two observations matter more than the number. First, **the 32-GPU jobs are 0.08% of arrivals and 70% of `E[S²]`** — wait time is driven by the second moment of the size distribution, so a few enormous jobs set everyone's wait. Second, `(1−ρ)` is a cliff: the same mix gives ≈1.7 h at ρ=0.70, ≈5.1 h at 0.90, ≈10.2 h at 0.95, and ≈25.5 h at 0.98, so the last 8% of utilisation costs 5× the wait. What I would change, in order: split large jobs into their own ClusterQueue to cut the variance the small jobs inherit; put the pools in a cohort so idle capacity is lent rather than reserved (L4); and only then consider adding capacity. And a caveat to state out loud: this single-server model ignores packing, so it is *optimistic* — real usable capacity is below 64 because of fragmentation (L7).

**(g) `AdmissionFairSharing` versus cohort `fairSharing` — scope, mechanism, and can one Workload be hit by both?**
**Answer:** `AdmissionFairSharing` (this lesson; beta-and-default since **v0.15**) acts **within one ClusterQueue**, across the LocalQueues that target it, and works by **admission ordering**: Kueue keeps a decaying weighted average of each LocalQueue's consumption (`usageHalfLifeTime`, sampled every `usageSamplingInterval`, default 5 m, weighted per resource by `resourceWeights`) in `LocalQueue.status.fairSharing.admissionFairSharingStatus.consumedResources`, and admits from the least-consuming LocalQueue first. An **entry penalty** (since v0.13.0) is charged at admission time so a burst of 100 submissions cannot outrun the sampling interval. It **never evicts anything**, and it requires a per-ClusterQueue opt-in: `admissionScope.admissionMode: UsageBasedAdmissionFairSharing`. Cohort `fairSharing` ([L4](04-kueue-cohorts-borrowing-preemption.md)) acts **across ClusterQueues in a cohort** and works by **preemption**, using each queue's dominant resource share; it is enabled by the presence of a `fairSharing` block in the Kueue Configuration. Yes, one Workload can be hit by both in a single lifecycle: admitted later than a sibling because its LocalQueue ranked poorly on decayed usage, and then — once admitted and borrowing — preempted because its whole ClusterQueue's share grew too large relative to cohort siblings. One is a queueing-order penalty, the other is an eviction.

## Connections & what's next

This lesson turns [L2](02-gang-scheduling.md)'s all-or-nothing guarantee into an **economic** mechanism. Coscheduling answered "can this job start safely"; the quota pool answers "should it start now, given everyone else." The `Workload` is the unit of both admission and accounting, which is why the deliverable's showback report is keyed on the ClusterQueue and why `kueue_cluster_queue_resource_usage{cluster_queue, flavor, resource}` is the series everything else in this module joins to.

Three threads carry forward. **The flavor is a price tag**: listing spot before on-demand is a one-line cost policy, and L8's commitment ladder is that idea made explicit. **The quota check is aggregate, not topological** — L6's Topology-Aware Scheduling is the fix for the gap §9 named, and L7's fragmentation math is the reason usable capacity is below the number in `nominalQuota`. **And the floors you just built are wasteful the moment one team is idle**, which is precisely the "held-but-idle GPUs = pure burn" line from the module README.

Next: **[04 · Kueue II — cohorts: borrowing, lending, preemption, and fair sharing](04-kueue-cohorts-borrowing-preemption.md)**, which puts these ClusterQueues into a cohort so `nominalQuota` becomes a floor you can *exceed* by borrowing an idle sibling's capacity, and a floor you can *reclaim* by preemption when you need it back. The `AdmissionFairSharing` mechanism introduced here resurfaces there alongside its cohort-level cousin, and the fairness math behind both lands on the same theoretical foundation: dominant resource share.

## References & further reading

> **A note on verification.** This environment's egress proxy blocks `kubernetes.io`, `kueue.sigs.k8s.io`, and most vendor blog domains. Everything marked **[verified against source]** was read directly from a clone of `kubernetes-sigs/kueue` at commit `e5084fe` (2026-08-17), or from `kubernetes/enhancements` via `raw.githubusercontent.com`. Entries marked **[not reachable]** are further reading only; no claim in this lesson depends on them.

**Primary sources**

1. **`apis/kueue/v1beta2/clusterqueue_types.go`** — https://github.com/kubernetes-sigs/kueue/blob/main/apis/kueue/v1beta2/clusterqueue_types.go — **[verified against source]**. Ground truth for every ClusterQueue field used here: `cohortName` (**renamed from `cohort` in v1beta2**), `namespaceSelector` and its documented `null` = nothing-selector default, `queueingStrategy` with `BestEffortFIFO` as the default, `stopPolicy`, `admissionScope`, `flavorFungibility` (`MayStopSearch`/`TryNextFlavor`, defaults, `preference`, and the deprecated `Borrow`/`Preempt` aliases), `ResourceQuota` (`nominalQuota`, `borrowingLimit`, `lendingLimit`), the ResourceGroup limits (16 groups, 64 flavors/resources per group, 256 total), the CEL rule forbidding `borrowingLimit` without a cohort, the `Active` condition reason strings, and the status fields.
2. **`apis/kueue/v1beta2/workload_types.go`** — **[verified against source]**. `WorkloadSpec` (`podSets` 1–18 and immutable, `queueName`, `priority`/`priorityClassRef`, `active`, `maximumExecutionTimeSeconds`, `preemptionGates`), `PodSet` (`count`, `minCount` for alpha partial admission, `topologyRequest`), `Admission`/`PodSetAssignment` (`flavors`, `resourceUsage`, `count`), `WorkloadStatus`, and the complete condition/reason vocabulary reproduced in §8 — `QuotaReserved` reasons (`WaitingForQuota`, `PendingEvaluation`, `NoMatchingFlavor`, `ExceedsMaxQuota`, `TopologyPlacementFailed`, `WaitingForPreemptedWorkloads`, `Misconfigured`, `Suspended`, `WaitingForPodsReady`), `Admitted` reasons, and `Evicted`/`Preempted` reasons.
3. **`apis/kueue/v1beta2/localqueue_types.go` and `resourceflavor_types.go`** — **[verified against source]**. The immutable `LocalQueue.spec.clusterQueue` CEL rule, LocalQueue status including `fairSharing.admissionFairSharingStatus.consumedResources`, and ResourceFlavor's four spec fields with their limits (8 nodeLabels / 8 nodeTaints / 8 tolerations), the "only `NoSchedule` and `NoExecute` are evaluated" rule, and the immutability of `topologyName`.
4. **`pkg/scheduler/scheduler.go`** — **[verified against source]**. The six numbered phases of `Scheduler.schedule()` and `makeClassicalIterator`'s four-key sort (quota-reserved first, non-borrowing before borrowing, priority under the `PrioritySortingWithinCohort` gate, then FIFO on eviction-or-creation timestamp).
5. **`pkg/controller/jobframework/base_webhook.go`, `defaults.go`, and `pkg/controller/constants/constants.go`** — **[verified against source]**. The `Default()` chain, `ApplyDefaultForSuspend`'s override of `spec.suspend`, `setDefaultLocalQueue` and `DefaultLocalQueueName = "default"`, and the label keys `kueue.x-k8s.io/queue-name`, `kueue.x-k8s.io/priority-class`, `kueue.x-k8s.io/job-uid`.
6. **`apis/config/v1beta2/configuration_types.go` and `defaults.go`** — **[verified against source]**. `AdmissionFairSharing` (`usageHalfLifeTime`, `usageSamplingInterval` default 5 m, `resourceWeights`), and the `waitForPodsReady` defaults: timeout 30 m, `recoveryTimeout` defaulting to the timeout, `blockAdmission: false`, requeuing backoff base 60 s capped at 3600 s.
7. **`pkg/metrics/metrics.go`** — **[verified against source]**. The exact metric names and label sets used in the showback query: `kueue_cluster_queue_resource_usage{cohort, cluster_queue, flavor, resource}`, `kueue_cluster_queue_resource_reservation`, `kueue_cluster_queue_nominal_quota`, `kueue_cluster_queue_borrowing_limit`, `kueue_pending_workloads`, `kueue_admitted_active_workloads`, `kueue_quota_reserved_wait_time_seconds`, `kueue_admission_wait_time_seconds`.
8. **`config/components/crd/bases/kueue.x-k8s.io_*.yaml`** — **[verified against source]**. Proof of the API-version state: `v1beta1` is `served: true, storage: false` and carries `deprecated: true` with `deprecationWarning: "This version is deprecated. Use v1beta2 instead."`; `v1beta2` is `served: true, storage: true`. **Correction recorded:** earlier versions of this lesson (and the module README's "API is v1beta1" note) predate this change; all manifests here are v1beta2.
9. **`CHANGELOG/CHANGELOG-0.16.md` and `CHANGELOG-0.19.md`** — **[verified against source]**. v0.16: "Kueue v0.16 starts using `v1beta2` API version for storage," with the `hack/migrate-to-v1beta2.sh` migration script and the stated intent to discontinue `v1beta1`. v0.19.0: "WaitForPodsReady is now enabled by default … (30 minute timeout, 30 minute recovery timeout)." Latest release at time of writing: **v0.19.1**.
10. **Kueue concept docs, in-repo** (`site/content/en/docs/concepts/{cluster_queue,local_queue,resource_flavor,workload,admission,admission_fair_sharing,cohort}.md`) — **[verified against source]**; the published site at https://kueue.sigs.k8s.io/docs/concepts/ is **[not reachable]** from this environment. Source for the flavor-fit rule (1)/(2)/(3), the two-phase admission description and its failure handling, the `pods` reserved resource name, the request-adjustment rules (LimitRange defaults, limits-as-requests, RuntimeClass overhead, pod-level resources on Kubernetes 1.32+), and the AdmissionFairSharing configuration and entry-penalty rationale.
11. **`site/content/en/docs/getting-started/installation.md`** — **[verified against source]**. The install command and the "Kubernetes 1.34 or newer is recommended" prerequisite for this Kueue version.

12. **KEP-2232, "Suspend Job"** (kubernetes/enhancements, SIG-Apps, `@adtac`) — https://github.com/kubernetes/enhancements/tree/master/keps/sig-apps/2232-suspend-jobs — **[verified against source]**, including `kep.yaml`: alpha **v1.21**, beta **v1.22**, stable **v1.24**, status `implemented`, feature gate `SuspendJob`. The field Kueue's entire admission model is built on.

**Real-world engineering blogs**

13. **CoreWeave — Kueue on CoreWeave Kubernetes Service** — https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads — **[not reachable]**. Listed for depth: a named target employer running Kueue in production for AI-lab customers.
14. **Netflix TechBlog — "How Netflix Simplified Batch Compute with Kueue"** — https://netflixtechblog.com/how-netflix-simplified-batch-compute-with-kueue-87860682629c — **[not reachable]**. Kueue at millions-of-workloads scale, plus the hardest-tenant-first migration strategy.
15. **IBM Research — Vela / Blue Vela** — arXiv 2407.05467, "The infrastructure powering IBM's Gen AI model development," and the KubeCon EU 2025 tutorial deck — **[not reachable]**. A research supercomputer using Kueue's queueing and cohort model to attack utilisation rather than capacity.

**Deeper dives**

16. **Kueue releases and the source tree** — https://github.com/kubernetes-sigs/kueue/releases and https://github.com/kubernetes-sigs/kueue. Check the release notes for your installed version before relying on any default stated here; `pkg/controller/jobframework` (the webhook and reconciler) and `pkg/scheduler` are the two packages worth reading end to end as the module-02 controller exercise.
