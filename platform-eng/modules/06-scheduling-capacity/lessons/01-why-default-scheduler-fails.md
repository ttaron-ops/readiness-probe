---
lesson: "06.1"
title: "Why the default scheduler fails distributed jobs"
module: "06"
concept: "default-scheduler-deadlock"
status: not-started
est_time: "6h"
prev: null
next: "02-gang-scheduling.md"
artifacts: []
sources: 11
---

# 06.1 · Why the default scheduler fails distributed jobs

> **Concept.** The default scheduler binds pods one at a time and never reconsiders, so a distributed job that needs all-or-nothing placement can strand GPUs in a partial-placement deadlock — held, idle, and billing.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

Module 02 made you fluent in the scheduling *cycle* itself: the ordered extension points a
plugin can hook, from `PreEnqueue` through `Bind`. That module's unit of analysis was always
**one pod**. It never had to ask what happens when the unit of *correctness* is a group of
pods — eight training replicas that only make progress if all eight start together. This
lesson is module 06's entry point precisely because it closes that gap: it shows,
mechanically, why a scheduler built around per-pod decisions produces a structural deadlock
for gang-shaped GPU jobs, and it sets up the fix (L2) as the obvious next move rather than a
black box you're told to trust.

Everything below about `kube-scheduler` internals was read out of the
`kubernetes/kubernetes` tree at the v1.37 development head (`pkg/scheduler/schedule_one.go`,
`pkg/scheduler/backend/queue/scheduling_queue.go`, `pkg/scheduler/framework/interface.go`,
`pkg/features/kube_features.go`, `pkg/apis/scheduling/types.go`), not out of a blog post.
Where a default has changed between minor versions, that is called out.

## Why this matters

"Why can't you just run distributed training on vanilla `kube-scheduler`?" is close to a
universal opener in GPU-platform interviews, and the JDs this course targets say so directly —
Anthropic's Sr Staff+ Kubernetes Platform posting names "gang scheduling" explicitly, and
CoreWeave's Principal/Staff Cluster Orchestration role wants a "technical authority on
scheduling, quota enforcement, fairness, pre-emption, and multi-tenant GPU isolation." The
answer they're listening for is a mechanism, not an opinion: the scheduler is a **greedy,
per-pod, incremental binder with no cross-pod rollback**. On CPU workloads that greed is
invisible; on gang-shaped GPU jobs it produces a deadlock where N−1 replicas pin GPUs while
the Nth pends forever.

The economic stakes are concrete and, worse, **silent** — nothing crashes, no alert fires, the
bill just keeps running. A single deadlocked hold on a handful of H100s burns real money every
hour with zero training steps produced, and if it strands a full node across two mutually
blocked jobs the number climbs into thousands of dollars a week (worked out below, with the
formula so you can substitute your own rate). Because nothing errors, this failure mode is
discovered by a human noticing low SM utilization or a job that "seems stuck" — not by a
scheduler complaint. That gap between *looks fine on a dashboard* and *is burning money* is
exactly why this is the first lesson of a module whose thesis is "every scheduling decision is
a cost decision."

## What's new here (calibration)

- **Module 02 (scheduler framework)** — you already know the scheduling cycle and the
  `Filter`/`Score`/`Reserve`/`Permit` extension points cold. We reference this vocabulary and
  re-state the exact ordering because the deadlock argument depends on it, but we don't
  re-derive what a plugin is.
- **Module 02b (NVLink/NVSwitch, Topology Manager)** — you already know why co-located
  replicas want the same interconnect domain. Not repeated here; it resurfaces in L6.
- **Module 04 (GPU quotas, MIG, time-slicing)** — you already know how a GPU becomes a
  countable `nvidia.com/gpu` resource and how `ResourceQuota` caps it. Assumed, not re-taught.
- **Genuinely new here:**
  - The scheduler's **three-queue state machine** (`activeQ`, `backoffQ`, `unschedulablePods`)
    with its real default timings, and why the retry loop it implements can never break this
    particular deadlock.
  - The precise reason `Unreserve` is not a rollback: it is scoped to one pod's own cycle, and
    the in-memory `assume` it undoes is a *cache* operation, not an API operation.
  - Why **no tuning knob** on `kube-scheduler` changes the outcome — enumerated knob by knob.
  - The **observability trap** where GPU-memory-allocated dashboards misreport a deadlocked
    job as healthy.
  - That **native Kubernetes is closing this gap upstream**: `Workload`/`PodGroup` in
    `scheduling.k8s.io`, alpha in v1.35 and **Beta (still off by default) in v1.37** behind the
    `GenericWorkload` gate — with a scheduling path that does something the per-pod path
    structurally cannot: simulate the whole group and *revert* on failure.

## Core concepts

### 1. What `kube-scheduler` actually is: two loops, three queues, one pod

Before you can explain the deadlock you have to be exact about the machine that produces it.
`kube-scheduler` is a control loop that repeatedly calls `ScheduleOne`. Each call pops **one
scheduling entity** off the head of a priority queue and drives it to a binding decision.
Everything else — the caches, the queues, the plugin framework — exists to make that single
decision fast and correct *for that one pod*.

```
                        kube-scheduler, one process
 ═══════════════════════════════════════════════════════════════════════════════════

  API server ──watch──▶ informers ──▶ ┌──────────────── SchedulingQueue ─────────────┐
   (pods, nodes,                      │                                              │
    PVs, ...)                         │  activeQ            heap ordered by the      │
       │                              │   (ready now)       QueueSort plugin         │
       │                              │      ▲   │                                   │
       │                              │      │   │ pop                               │
       │                              │      │   ▼                                   │
       │                              │  backoffQ ◀── failed, waiting out backoff    │
       │                              │      ▲       1s → 2s → 4s → 8s → 10s (cap)   │
       │                              │      │                                       │
       │                              │  unschedulablePods ◀── "no event could help" │
       │                              │        flushed to backoffQ/activeQ after 5m  │
       │                              └───────────────┬──────────────────────────────┘
       │                                              │ one pod
       ▼                                              ▼
  ┌──────────────┐   UpdateSnapshot()      ┌──────────────────────────────────────┐
  │ scheduler    │ ──────────────────────▶ │  SCHEDULING CYCLE  (synchronous,     │
  │ Cache        │  point-in-time copy of  │  one at a time, never concurrent)    │
  │ (nodes +     │  node/pod allocations   │                                      │
  │  assumed     │                         │  PreFilter → Filter → PostFilter     │
  │  pods)       │ ◀── assume(pod, node) ──│  → PreScore → Score → Reserve        │
  └──────────────┘   in-memory only!       │  → Permit                            │
                                           └────────────────┬─────────────────────┘
                                                            │ handoff
                                                            ▼
                                           ┌──────────────────────────────────────┐
                                           │  BINDING CYCLE (goroutine, parallel) │
                                           │  WaitOnPermit → PreBind → Bind       │
                                           │  → PostBind                          │
                                           │  Bind = POST /pods/<p>/binding       │
                                           └──────────────────────────────────────┘
```

Four properties of this machine matter for the rest of the lesson.

**(a) The scheduling cycle is serialized; the binding cycle is not.** Only one pod is in a
scheduling cycle at a time, which is what makes the cache snapshot coherent. Binding — the
actual `POST` to `/api/v1/namespaces/<ns>/pods/<name>/binding` — is handed off to a goroutine
so the next pod's scheduling cycle can start immediately. This is why replicas of a job get
placed in rapid succession: the expensive part of each decision is milliseconds, and the slow
API write happens off the critical path.

**(b) `assume` is a cache write, not an API write.** Before `Permit` runs, the scheduler calls
`assume()`, which sets `pod.Spec.NodeName` **in the in-memory cache only** and marks the pod
as consuming that node's resources. That is what stops the *next* pod's `Filter` from handing
out the same GPU. The pod is not yet bound in etcd, but from the scheduler's own point of view
the resource is gone. In `pkg/scheduler/schedule_one.go` this is `assumeAndReserve()`, called
from `prepareForBindingCycle()` immediately before `RunPermitPlugins()`.

**(c) The three queues implement retry, not reconsideration.** A pod that fails scheduling
goes to `backoffQ` or `unschedulablePods`. From
`pkg/scheduler/backend/queue/scheduling_queue.go`:

| Constant | Default | What it controls |
|---|---|---|
| `DefaultPodInitialBackoffDuration` | **1 second** | First retry delay after a failed scheduling attempt |
| `DefaultPodMaxBackoffDuration` | **10 seconds** | Ceiling on the exponential backoff |
| `DefaultPodMaxInUnschedulablePodsDuration` | **5 minutes** | How long a pod may sit in `unschedulablePods` with no relevant cluster event before being flushed back for another try |

Backoff doubles from 1s and saturates at 10s, so an unschedulable pod is retried roughly every
10 seconds forever, plus a guaranteed sweep every 5 minutes even if nothing in the cluster
changed. **This retry loop is the only self-healing mechanism the default scheduler has for a
pending pod, and it retries the same decision against a cluster state that only its own
siblings could change.** Hold that thought.

**(d) There is no group in the data model.** The entity popped from `activeQ` is a pod (or, on
v1.37 with the alpha/beta `GenericWorkload` gate on, a pod group — see §11). A `Job`'s
`spec.parallelism: 8` produces eight independent `Pod` objects with an owner reference to the
Job and a shared `batch.kubernetes.io/job-name` label. Nothing in the default scheduling path
reads that label. Replica 7 is, to the scheduler, an anonymous pod requesting
`nvidia.com/gpu: 1`.

### 2. The extension points, in order, and what each one can see

Module 02 gave you the list. Here it is again with the two columns that matter for this
lesson: what state the plugin has access to, and whether it could, in principle, act on the
*group*. This table is the map for L2.

| # | Extension point | Runs in | Sees | Can it act on siblings? |
|---|---|---|---|---|
| 1 | `PreEnqueue` | queue admission | the pod, plugin-managed state | Only to *hold* the pod out of `activeQ` |
| 2 | `QueueSort` | queue ordering | two pods, pairwise | Only ordering — it can make siblings adjacent |
| 3 | `PreFilter` | scheduling cycle | pod + snapshot of all nodes | Yes, if it queries siblings itself (lister) |
| 4 | `Filter` | scheduling cycle | pod + one node | No — per (pod, node) feasibility |
| 5 | `PostFilter` | scheduling cycle, on failure | pod + per-node failure statuses | Yes — this is where preemption lives |
| 6 | `PreScore` / `Score` / `NormalizeScore` | scheduling cycle | pod + feasible nodes | No — ranks nodes for *this* pod |
| 7 | `Reserve` | scheduling cycle | pod + chosen node | Reserves plugin-side state for this pod |
| 8 | `Permit` | scheduling cycle | pod + chosen node | **Yes** — can return `Wait` and park the pod |
| 9 | `PreBind` / `Bind` / `PostBind` | binding cycle | pod + node | No |
| — | `Unreserve` | on failure after `Reserve` | pod + node | **This pod only** |

Two entries deserve unpacking because they are the ones people get wrong.

**`Permit` is the only extension point that can stop a pod after the decision is made but
before it becomes real.** `RunPermitPlugins` returns one of three outcomes. `Success` lets the
pod flow into the binding cycle. `Reject` fails it. **`Wait`** puts the pod into the
framework's *waiting pods* map with a per-plugin timeout, and the pod's binding cycle blocks in
`WaitOnPermit()` until some other code path calls `Allow()` or `Reject()` on it. Crucially,
the pod is *already assumed in the cache* at this point — its GPU is already accounted as
consumed. A `Permit` wait therefore holds a soft reservation, not nothing. That property is
what makes gang scheduling possible (L2) and also what makes a naive gang implementation cause
head-of-line blocking (also L2).

**`Unreserve` is not a transaction rollback.** Read the actual call site in
`prepareForBindingCycle()`: if `RunPermitPlugins` returns anything other than `Success` or
`Wait`, the scheduler calls `unreserveAndForget(...)` for **that pod**, which runs
`RunReservePluginsUnreserve` and removes the assumed pod from the cache. The same thing happens
if the binding cycle fails. In both cases the scope is a single pod and the trigger is a
failure in *that pod's own cycle*. There is no code path anywhere in `pkg/scheduler` — with
the default plugin set — in which pod A's failure causes pod B's reservation or binding to be
undone. **Rollback exists, but only along the single-pod axis. That asymmetry is the entire
bug this module exists to fix.**

```
   WHERE A GANG PLUGIN HAS TO INTERVENE (preview of L2)

   PreEnqueue   QueueSort    PreFilter    Filter   PostFilter  PreScore/Score
      │            │             │           │         │            │
      │       keep group     reject the      │      (preemption)    │
      │       members       whole group      │                      │
      │       adjacent      if fewer than    │                      │
      │                     minMember can    │                      │
      │                     ever fit         │                      │
      ▼            ▼             ▼           ▼         ▼            ▼
   ─────────────────────────────────────────────────────────────────────────▶
                                                       │
                                     Reserve ──▶  ★ PERMIT ★  ──▶ Bind
                                        │              │
                                 soft claim on   return Wait; count
                                 the node's GPU  waiting+running members;
                                        │        Allow() ALL at once when
                                        │        count >= minMember
                                        ▼
                                   Unreserve ◀── release the soft claim
                                                 on timeout/reject
```

The default plugin set implements **none** of the starred behaviour. That is not a bug in the
plugins; it is a consequence of the framework's contract, which is defined per pod.

### 3. The reason a group is the unit of correctness: collective barriers

The scheduler's per-pod model would be harmless if distributed jobs tolerated partial starts.
They do not, and the reason is in the collective-communication runtime, not in Kubernetes.

A PyTorch DDP job starts each rank with `torch.distributed.init_process_group(backend="nccl")`.
Underneath, with the default `env://` rendezvous, every rank connects to a `TCPStore` hosted by
rank 0 at `MASTER_ADDR:MASTER_PORT` and publishes its NCCL unique ID / rank metadata. The call
does not return until the store reports that **`WORLD_SIZE` ranks have checked in**. Then
`ncclCommInitRank()` runs, which is itself a collective: every rank must call it, and the call
blocks until all participating ranks have. MPI is the same shape — `MPI_Init` followed by a
communicator construction that every rank must enter.

So a job with `WORLD_SIZE=8` and 6 running ranks is not "62.5% running". It is **0% running
and 100% billing**:

- Six processes are alive, have created a CUDA context, and are blocked in a socket read or a
  spin-wait inside the rendezvous.
- Two processes do not exist, because their pods were never bound.
- The barrier will be satisfied when the two missing ranks arrive, and by no other event.

`init_process_group` accepts a `timeout` (default 30 minutes for NCCL in recent PyTorch
releases; check `torch.distributed.init_process_group`'s signature for the version you run).
When it expires you get a rendezvous timeout, the job crashes, the six pods exit, the GPUs
free — and the Job controller creates fresh pods that walk into the identical wall. **The
application-level timeout converts a deadlock into a crash loop. It does not convert it into
progress.**

### 4. The partial-placement deadlock, mechanically

Now put the two halves together. Cluster: four nodes, two GPUs each, of which **6 GPUs are
free** (some earlier work holds 2). Submit one `Job` with `parallelism: 8`, each pod
requesting `nvidia.com/gpu: 1`.

```
  TIMELINE — 8-replica job meets 6 free GPUs on a 4×2-GPU cluster
  ══════════════════════════════════════════════════════════════════════════════════

  free GPUs   node-a [G G]   node-b [G G]   node-c [G .]   node-d [G .]     = 6 free
              (. = already occupied by unrelated work)

  t+0.00s  r0 popped from activeQ
             PreFilter ok → Filter: a,b,c,d feasible → Score → Reserve(node-a)
             Permit: no plugin objects → Success → assume(r0,node-a)   free: 6→5
  t+0.01s  r1 → node-a (second GPU)                                    free: 5→4
  t+0.02s  r2 → node-b                                                 free: 4→3
  t+0.03s  r3 → node-b                                                 free: 3→2
  t+0.04s  r4 → node-c                                                 free: 2→1
  t+0.05s  r5 → node-d                                                 free: 1→0
  t+0.06s  r6 → Filter fails on every node:
             NodeResourcesFit: "Insufficient nvidia.com/gpu"  (0/4 nodes)
             PostFilter (DefaultPreemption): no lower-priority victims → no help
             → unschedulablePods
  t+0.07s  r7 → same → unschedulablePods

  t+0.1s … kubelets pull images, containers start on r0..r5
  t+~20s   all six ranks call init_process_group(world_size=8)
             → each blocks in TCPStore rendezvous waiting for ranks 6,7

  t+10s, +20s, +30s, …  scheduler flushes r6,r7 back to activeQ (backoff caps at 10s)
             every attempt: Filter fails identically. Nothing has changed.
             The only thing that COULD change it is r0..r5 exiting.
             r0..r5 will not exit: they are blocked waiting for r6,r7.

  ┌───────────────────────────────────────────────────────────────────────────┐
  │  STEADY STATE:  6 GPUs Running · 0 GPUs computing · 2 pods Pending        │
  │                 SM utilization ≈ 0%   ·   GPU memory allocated > 0        │
  │                 job status: "healthy" to any allocation-based dashboard   │
  │                 duration: until the NCCL rendezvous timeout, or a human   │
  └───────────────────────────────────────────────────────────────────────────┘
```

The circularity is worth stating in one sentence, because it is the sentence an interviewer
wants: **the event that would unblock the pending pods can only be produced by the running
pods, and the running pods are blocked on the pending pods.**

Note also *where* the resource went. It is not "reserved" in any recoverable sense — replicas
0–5 are genuinely `Running`, with containers up and CUDA contexts allocated. Deleting the two
`Pending` pods does not help; the Job controller recreates them. Deleting the Job is the only
cheap fix, and that is a human action.

### 5. Why it cannot self-heal — the four escape hatches, each closed

New engineers reliably propose one of four fixes. Each is worth knowing cold because each is
an interview follow-up.

**(a) "The scheduler will retry."** It does, forever, on the schedule in §1(c). Retry is a
loop over an unchanged input. `Filter` re-evaluates `NodeResourcesFit` against a cluster where
the same six GPUs are still allocated. The scheduler is also event-driven: `EventsToRegister`
means a pod-delete or node-add event re-queues unschedulable pods immediately rather than
waiting out the backoff. But the event never comes. Retrying a pure function with the same
arguments produces the same answer, however many times you do it.

**(b) "`activeDeadlineSeconds` or the rendezvous timeout will clean it up."** Both do,
eventually, and both are *crashes*, not repairs. Set `activeDeadlineSeconds: 3600` and you have
declared that you are willing to burn one hour of six GPUs before anything happens. Then the
Job fails, and unless `backoffLimit` is exhausted the controller retries — into the same wall.
The cost model in the Worked example puts a number on the resulting retry storm.

**(c) "Priority and preemption will make room."** `PostFilter`'s `DefaultPreemption` plugin
runs when a pod is unschedulable, finds nodes where evicting lower-priority pods would make the
pod fit, picks the node with the lowest "victim cost", and deletes those victims. Three
reasons this does not save you:

- It is **per-pod**. It frees enough room for r6. Then r7 runs its own cycle and preempts
  someone else. There is no notion of "free enough room for the remaining two members".
- The victims are typically *other jobs' pods*, so you relocate the deadlock: victim job now
  has N−1 replicas running and one pending. You have converted your deadlock into someone
  else's deadlock while adding an eviction.
- If the six holders are the same priority as r6/r7 — which they are, they are siblings —
  preemption declines to act at all. `DefaultPreemption` only considers victims with **strictly
  lower** priority.

**(d) "Cluster autoscaler will add a node."** On an autoscaled CPU fleet this is often true.
On GPUs it usually is not, and L8 is the full argument, but the short version: GPU node pools
are frequently at their quota/reservation ceiling, GPU node provisioning is minutes not
seconds, and on many neoclouds the capacity simply is not there to buy on demand. Cluster
autoscaler also scales on *pending pods*, so it would (correctly) see two pending pods and try
to add capacity for two — which is right, but slow, and does nothing about the six GPU-hours
you burn while waiting.

### 6. The mutual deadlock: two gangs, one cluster, zero progress

The single-job case wastes capacity. The two-job case wastes **all** of it, and this is the
pathology gang scheduling was invented to prevent.

Two teams each submit a 4-replica job to a cluster with exactly 4 free GPUs. The scheduler
interleaves them — `QueueSort` orders by priority then creation timestamp, and two jobs
created a second apart with equal priority will interleave as their pods are created by two
independent Job controllers.

```
  BEFORE  (4 free GPUs, two 4-replica jobs arrive together)
  ┌─────────────────────────────────────────────────────────────────────┐
  │ GPU-0   GPU-1   GPU-2   GPU-3                                       │
  │  free    free    free    free          Job-A: needs 4   Job-B: needs 4│
  └─────────────────────────────────────────────────────────────────────┘

  AFTER   (scheduler interleaves; each job wins 2)
  ┌─────────────────────────────────────────────────────────────────────┐
  │ GPU-0   GPU-1   GPU-2   GPU-3                                       │
  │ [A-r0]  [B-r0]  [A-r1]  [B-r1]                                      │
  │  ▲ blocked in rendezvous, waiting for A-r2,A-r3 (Pending)           │
  │          ▲ blocked in rendezvous, waiting for B-r2,B-r3 (Pending)   │
  └─────────────────────────────────────────────────────────────────────┘
   allocated capacity : 4/4  = 100%
   productive capacity: 0/4  =   0%
   pending pods       : 4     (2 from A, 2 from B)
   who can unblock whom: nobody. Each job holds exactly what the other needs.
```

Neither job yields, because yielding is not a behaviour either job has. Neither scheduler
action helps, because every action is per-pod. The cluster is fully allocated and completely
idle, and it will stay that way until a timeout fires or a human intervenes. This is what
"stranded capacity" means in the most literal sense.

### 7. Why no amount of tuning fixes it

This is the part that turns a good answer into a strong one. Go through the knobs a
`KubeSchedulerConfiguration` actually exposes and show that each is orthogonal to the problem.

| Knob | What it changes | Why it doesn't help |
|---|---|---|
| `percentageOfNodesToScore` | How many feasible nodes get scored before the scheduler stops looking (adaptive by default; floors of 100 nodes / 5% are hard-coded as `minFeasibleNodesToFind` / `minFeasibleNodesPercentageToFind`) | A throughput/quality tradeoff *within* one pod's decision. Does not affect whether the decision is made per pod. |
| Plugin weights (`NodeResourcesFit`, `PodTopologySpread`, …) | Which node a pod prefers | Changes *where* replicas land, not *whether* partial placement is allowed. |
| `podInitialBackoffSeconds` / `podMaxBackoffSeconds` | Retry cadence for unschedulable pods | Retries the same failing decision more or less often. |
| `PriorityClass` on the job | Whether this pod can preempt others | Per-pod preemption; see §5(c). Also useless among siblings, which share a priority. |
| `PodDisruptionBudget` | How many pods of a set may be *voluntarily* evicted | Governs eviction of already-running pods. Says nothing about admission. |
| `podAffinity` / `PodTopologySpread` | Placement relative to other pods | Constrains *where*, not *whether*. A gang can still be partially placed, just partially placed differently. |
| `schedulerName` | Which scheduler process handles the pod | This is the actual escape hatch — but it means running a *different* scheduler (L2, L5), not tuning this one. |
| `NodeResourcesFit` scoring strategy (`LeastAllocated` / `MostAllocated` / `RequestedToCapacityRatio`) | Bin-packing behaviour | `MostAllocated` reduces *fragmentation*, which reduces the frequency of the deadlock. It does not remove the possibility, because the deadlock is about atomicity, not packing. |

**The one-line version: every knob is a preference over placements; the deadlock is a missing
constraint on admission. You cannot express a constraint by adjusting a preference.**

### 8. Why the GPU makes this worse than the CPU case

The same structural gap exists for CPU-only distributed jobs, and it mostly does not bite.
Three properties of GPUs as a Kubernetes resource turn a theoretical gap into a daily incident.

**Extended resources are integers with no overcommit.** `nvidia.com/gpu` is an extended
resource advertised by the device plugin in the node's `status.capacity` /
`status.allocatable`. Kubernetes requires that for extended resources, requests equal limits
and both are whole numbers — you cannot ask for `0.5`, and you cannot oversubscribe the way
you can with CPU shares. So a node with 8 GPUs and 8 allocated has *zero* headroom. There is no
equivalent of "the CPU is oversubscribed 3:1 and everything is a bit slower". The
`NodeResourcesFit` plugin's failure message you will see in `kubectl describe` — `Insufficient
nvidia.com/gpu` — is produced by exactly this integer comparison.

**GPU pods are large relative to node capacity.** A CPU pod asking for 500m on a 64-core node
has 128 potential slots per node; the odds that all of them are gone are low. A pod asking for
1 of 8 GPUs has 8. Coarse granularity means the transition from "plenty" to "none" happens in a
handful of placements.

**The waste is expensive and invisible.** A stranded CPU core costs cents. A stranded H100
costs dollars per hour, and — the important part — a *held* GPU bills identically whether it is
at 100% SM utilization or blocked in a socket read. The scheduler has converted provisioned
capacity into stranded capacity at full price.

### 9. The observability trap: memory-allocated is not utilization

This is where the failure hides from you operationally, and it is the direct bridge to
module 05.

The six "Running" pods are not idle at the OS level. `torch.cuda.init()` creates a CUDA
context, which allocates a few hundred MB of device memory before a single kernel launches;
NCCL allocates its own buffers during initialization; frameworks commonly pre-allocate a
caching allocator arena. So on a deadlocked rank you will typically observe:

| Signal | Deadlocked rank | Healthy training rank |
|---|---|---|
| Pod phase | `Running` | `Running` |
| Process count on the GPU | ≥ 1 | ≥ 1 |
| GPU memory allocated | non-zero, often GB | non-zero, often near capacity |
| **SM activity** (DCGM `DCGM_FI_PROF_SM_ACTIVE`, or the coarse `DCGM_FI_DEV_GPU_UTIL`) | **≈ 0** | high |
| Framework logs | last line is the `init_process_group` call | step/loss lines advancing |

A dashboard keyed on "GPU memory used" or "GPU allocated" reports this job as healthy. A
dashboard keyed on SM activity does not. **The fingerprint of this failure is: pod phase
`Running`, non-zero GPU memory allocated, SM activity near zero, sustained for minutes, with
one or more sibling pods `Pending`.** That conjunction is specific enough to alert on, and
wiring exactly that alert is one of the more valuable things module 05's stack buys you.

Note the asymmetry that makes this dangerous: the *cheap* metric (memory allocated, available
from NVML with no special privileges) is the misleading one, and the *costed* metric (SM
activity, from the hardware performance monitors) is the truthful one. Teams default to the
cheap one.

### 10. The cost model: stranded GPU-hours, with the arithmetic

The formula, stated once so you can substitute your own numbers:

```
  stranded_GPU_hours = held_GPUs × wall_clock_hours_held
  stranded_cost      = stranded_GPU_hours × rate_per_GPU_hour

  where held_GPUs = (replicas_bound)              for a single-job deadlock
                  = Σ over jobs (replicas_bound)  for a mutual deadlock
  and   wall_clock_hours_held = min(rendezvous_timeout,
                                    activeDeadlineSeconds,
                                    time_to_human_intervention)
```

Rates vary enormously by market segment — reserved capacity, neocloud on-demand, hyperscaler
on-demand, and spot can differ by 3–5× for the same silicon, and that spread is L8's whole
subject. Use a **$2.35/GPU-hr H100 on-demand snapshot** below purely as a worked figure;
substitute your own contract rate and the shape of the answer is unchanged.

**Case 1 — the §4 deadlock (6 GPUs held), caught by a human in 45 minutes.**

```
  6 GPUs × 0.75 h                = 4.5 GPU-hours
  4.5 GPU-hours × $2.35/GPU-hr   = $10.58
```

Small. That is the point: individually these are invisible, which is why they persist.

**Case 2 — the same deadlock, unnoticed over a weekend, with `activeDeadlineSeconds: 3600`
and `backoffLimit: 6` (the Job default).**

The Job times out at 1h, the controller recreates the pods, and the same deadlock forms. Seven
attempts total (initial + 6 retries):

```
  7 attempts × 1 h × 6 GPUs      = 42 GPU-hours
  42 × $2.35                     = $98.70   for zero training steps
```

**Case 3 — the §6 mutual deadlock on a full 8-GPU node, unnoticed for a weekend
(64 hours Friday 18:00 → Monday 10:00).**

```
  8 GPUs × 64 h                  = 512 GPU-hours
  512 × $2.35                    = $1,203.20
  per-day rate: 8 × 24 × 2.35    = $451.20/day
  per-week rate: 8 × 168 × 2.35  = $3,158.40/week
```

**Case 4 — fleet scale.** The number that actually gets a platform team funded is not one
incident, it is the rate. Suppose a 512-GPU fleet runs 40 distributed jobs a day, 3% of which
hit a partial-placement deadlock, each stranding on average 5 GPUs for 40 minutes before
someone kills it:

```
  deadlocks/day        = 40 × 0.03            = 1.2
  stranded GPU-h/day   = 1.2 × 5 × (40/60)    = 4.0 GPU-hours/day
  stranded GPU-h/year  = 4.0 × 365            = 1,460 GPU-hours/year
  cost/year            = 1,460 × $2.35        = $3,431/year
  as % of fleet spend  = 1,460 / (512 × 8760) = 0.033%
```

Read that honestly: at a 3% incidence with fast human response, the direct burn is **third of
a tenth of a percent** of fleet spend — real money but not a crisis. Now change the two
assumptions that a mature platform actually changes: response time goes from 40 minutes to 8
hours over a weekend, and incidence rises to 10% because the fleet is busier and more
fragmented:

```
  deadlocks/day        = 40 × 0.10            = 4.0
  stranded GPU-h/day   = 4.0 × 5 × 8          = 160 GPU-hours/day
  cost/year            = 160 × 365 × $2.35    = $137,240/year   (0.85% of fleet)
```

**The honest framing for an interview is therefore: the direct GPU burn is a real but
second-order cost, and it is dominated by two things this lesson's fix also solves — engineer
time spent debugging what looks like a networking bug, and the queue-latency cost of capacity
that is allocated but unusable by anyone else.** L7 quantifies the second properly. Do not
oversell the first; a strong candidate gives the number *and* its share of spend.

### 11. New since this course was scoped: native gang scheduling upstream

For most of Kubernetes' history, "gang scheduling requires a third-party plugin or an alternate
scheduler" was simply true. That stopped being unconditionally true in **Kubernetes v1.35**,
which shipped KEP-4671 as **alpha**, and the work has continued since. Verified against the
`kubernetes/kubernetes` tree at the v1.37 development head:

**The feature gates** (`pkg/features/kube_features.go`, `versioned_kube_features` table):

| Gate | Introduced | Current state on v1.37 | Default |
|---|---|---|---|
| `GenericWorkload` | v1.35 Alpha | **Beta** | **`false`** — beta but *off by default* |
| `TopologyAwareWorkloadScheduling` | v1.36 Alpha | Alpha | `false` |
| `CompositePodGroup` | v1.37 Alpha | Alpha (depends on `GenericWorkload` + `TopologyAwareWorkloadScheduling`) | `false` |
| `PodGroupPreemptionPolicy` | v1.37 Alpha | Alpha (depends on `GenericWorkload`) | `false` |

A beta gate defaulting to `false` is unusual and deliberate: SIG-Scheduling is shipping a
change to *how groups of pods get scheduled*, which has real blast radius on a busy cluster, so
adoption is opt-in rather than arriving with a routine minor upgrade.

**The API shape** (`pkg/apis/scheduling/types.go`, group `scheduling.k8s.io`, external version
`v1alpha3` at the time of reading):

- **`Workload`** — a static description of a group's scheduling policy. It holds
  `spec.podGroupTemplates` (max 8 templates, and templates cannot be added or removed after
  creation) and an optional `spec.controllerRef` back to the owning Job/Deployment. It does
  **not** manage pod lifecycles.
- **`PodGroup`** — the *runtime instance*. Workload controllers (Job, LWS, JobSet, …) create
  `PodGroup`s from a `Workload`'s templates. Splitting template from instance keeps live
  scheduling status off one shared, hot-contended object and keeps very large gangs from
  bumping into etcd's per-object size ceiling.
- **`PodGroupSpec.schedulingPolicy`** is a union: `Basic{}` (ordinary per-pod behaviour) or
  **`Gang{ minCount }`**. From the field's own doc comment: `minCount` is "the minimum number of
  pods that must be schedulable or scheduled at the same time for the scheduler to admit the
  entire group", it must be positive, and it is **mutable** to support workload scaling —
  though the scheduler is eventually consistent, so a mid-cycle change may not take effect
  until the next cycle, and it never affects already-scheduled pods.
- **Pods opt in** via `spec.schedulingGroup.podGroupName` (`PodSchedulingGroup`, gated on
  `GenericWorkload`). The field is immutable, and the referenced `PodGroup` need not exist when
  the pod is created. Note the history: an earlier `spec.workloadRef` field was replaced by
  `spec.schedulingGroup` in v1.36, and protobuf tag 42 is tombstoned in `core/v1/types.go` to
  record it.

**The mechanism, and why it is genuinely different.** The new code path lives in
`pkg/scheduler/schedule_one_podgroup.go`. `ScheduleOne` now pops a **scheduling entity** —
either a pod or a pod group. For a group, the scheduler *simulates* placing every member
against one cluster-state snapshot, and it maintains an explicit stack of `revertFns`:
functions that undo the in-memory changes made during the simulation, specifically "assuming
pods and calls to Reserve plugins". These are invoked when the group fails, to roll back
partial modifications, and after each candidate placement is evaluated.

**That is precisely the thing the per-pod path cannot do.** In the per-pod path, `Unreserve`
undoes one pod's own reservation on that pod's own failure. In the group path, one member's
infeasibility triggers revert of *every* member's assumed placement. All-or-nothing stops being
a plugin's timeout-driven approximation and becomes a property of the scheduling loop.

**What it does not do.** The `Workload`/`PodGroup` API solves *atomic admission*. It says
nothing about quota pools, cohorts, borrowing, fair-sharing, or per-team showback — the
cost/fairness economics layer this module spends most of its hours on, and where Kueue's value
concentrates. Read the arrival of this API as: **the gang-admission mechanism is standardizing
into core Kubernetes; the queueing and quota-economics layer built on top of it is not.**

## Perspectives

**Developer.** A researcher submitting a multi-worker `Job` sees `init_process_group` hang with
no error — just silence — because a scheduler-level failure (6/8 pods bound) produces no
application-level exception; the rendezvous simply blocks on the missing ranks. The developer's
mental model ("I asked for GPUs, Kubernetes gives me GPUs, my job runs") breaks the first time
this happens, and debugging *looks* like a networking problem: a rendezvous timeout, a DNS
issue, a firewall rule on `MASTER_PORT`. Without this lesson's mechanism the natural instinct
is to raise the timeout and add retries in the training script, which makes the economics
strictly worse — a longer timeout means a longer hold.

**Operator/platform.** The platform engineer's job is to make this failure *structurally
impossible* rather than reactively debugged: the fix is an admission-time invariant, not a
runbook someone follows at 2am. The operator also has to reason about interaction with other
pending work. Does the greedy binder's per-pod ordering actively starve gangs behind small jobs
that sneak into exactly the fragments a large gang needed? It does, and that tension — one
job's atomicity versus another job's latency — is the thread running through the rest of the
module.

**Hardware/kernel.** The GPU has no idea it is deadlocked. A CUDA context is allocated, memory
is reserved, and the SMs are idle because no kernel has been launched. From the device's
perspective this is indistinguishable from a job between steps. Only a time-differenced
hardware performance-monitor sample (SM active cycles over elapsed cycles) can tell the two
apart, which is why the distinction between allocation metrics and activity metrics is not
pedantry — it is the difference between seeing this incident and not seeing it.

**Economics/FinOps.** This is a silent failure economically: nothing crashes, nothing alerts,
the bill runs. It compounds through retries, because `activeDeadlineSeconds` + `backoffLimit`
multiply the waste by the number of attempts before a human looks. The right framing for a
capacity review is not "we lost $99 on a stuck job" but "**allocated capacity and usable
capacity are different numbers, and this is one of the mechanisms that separates them**" —
which is exactly the argument L7 formalises and the capstone's showback report has to make
visible.

## Real-world use cases

- **OpenAI — "Scaling Kubernetes to 7,500 Nodes"** — https://openai.com/index/scaling-kubernetes-to-7500-nodes/.
  What it shows: OpenAI built and ran a **gang scheduling plugin** at 7,500-node scale
  specifically because their largest jobs use MPI, where any missing participant halts the
  whole job — alongside team taints and CPU/GPU "balloon" deployments to hold capacity for
  fair distribution. A first-party account of exactly this deadlock being solved operationally
  at hyperscale, not theoretically. *(Search-verified this session across multiple independent
  citations; direct fetch of openai.com was blocked by this environment's egress proxy — cited
  by canonical URL per the SPEC's sourcing rules.)*

- **kubernetes-sigs/scheduler-plugins — the coscheduling plugin's own demo.** What it shows:
  the maintainers' README documents the failure *and* the fix with output. A `ReplicaSet` of 6
  nginx pods on a cluster that fits 3, with `minMember: 3`, produces 3 Running and 3 Pending —
  the partial-placement pattern. Raise `minMember` to 4 and **all six** pods go Pending, zero
  resources consumed. That before/after is the cleanest possible demonstration that the
  difference is a *constraint*, not a preference. **Read directly from the cloned repository
  this session** (`pkg/coscheduling/README.md`), since the rendered docs site was unreachable
  from this environment.

- **Kubernetes upstream — the `Workload`/`PodGroup` API and its feature gates.** What it shows:
  SIG-Scheduling's own admission that per-pod scheduling is structurally insufficient for
  workloads, significant enough that core Kubernetes is building a native primitive. The
  substance: `GenericWorkload` alpha in v1.35 → **beta in v1.37, still `Default: false`**;
  `Gang{minCount}` as the policy; `spec.schedulingGroup.podGroupName` as the pod-side opt-in;
  and a scheduling path built around simulate-and-revert. **Read directly from the cloned
  `kubernetes/kubernetes` tree this session** — `pkg/features/kube_features.go`,
  `pkg/apis/scheduling/types.go`, `pkg/scheduler/schedule_one_podgroup.go` — because
  kubernetes.io was unreachable from this environment.

- **Lambda — "Why your Kubernetes scheduler can't handle AI workloads"** —
  https://lambda.ai/blog/why-your-kubernetes-scheduler-cant-handle-ai-workloads. What it shows:
  a GPU-cloud vendor's own engineering explanation that by default `kube-scheduler` does not
  gang-schedule and places pods one at a time, with no concept of "all pods in this job must
  start together, or none of them start" — an independent restatement of this lesson's thesis
  in different words, plus their comparison of Kueue vs Volcano as fixes. *(Search-verified;
  fetch blocked by egress this session.)*

## Worked example

Reproduce the deadlock on a kind cluster with **fake** GPUs, then price it. No hardware needed:
extended resources are opaque integers the scheduler tracks, so a patched node status is
indistinguishable from a real device plugin as far as scheduling is concerned.

**Step 1 — three workers, one fake GPU each.**

```bash
$ kind create cluster --name sched-lab --config - <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
EOF

$ kubectl proxy --port=8001 >/dev/null 2>&1 & PROXY=$!
$ for n in sched-lab-worker sched-lab-worker2 sched-lab-worker3; do
    curl -s -H "Content-Type: application/json-patch+json" -XPATCH \
      "http://localhost:8001/api/v1/nodes/$n/status" \
      --data '[{"op":"add","path":"/status/capacity/nvidia.com~1gpu","value":"1"}]' >/dev/null
  done
$ kill $PROXY

$ kubectl get nodes -o custom-columns=\
NAME:.metadata.name,GPU:.status.allocatable.'nvidia\.com/gpu'
NAME                     GPU
sched-lab-control-plane  <none>
sched-lab-worker         1
sched-lab-worker2        1
sched-lab-worker3        1
```

`~1` is JSON-Pointer escaping for the `/` inside `nvidia.com/gpu`. Note the patch targets
`capacity`; kubelet derives `allocatable` from it for extended resources.

**Step 2 — a 4-replica job on 3 GPUs.**

```yaml
# gang4.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: gang-demo
spec:
  parallelism: 4          # 4 pods run concurrently
  completions: 4          # all 4 must succeed for the Job to succeed
  backoffLimit: 0         # do not retry — we want one clean observation
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: registry.k8s.io/e2e-test-images/agnhost:2.53
          args: ["pause"]                 # stand-in for a blocked rendezvous
          resources:
            limits:
              nvidia.com/gpu: "1"         # extended resources: request==limit, integer
```

```bash
$ kubectl apply -f gang4.yaml
$ kubectl get pods -l batch.kubernetes.io/job-name=gang-demo -o wide
NAME              READY   STATUS    NODE                NOMINATED NODE
gang-demo-abc01   1/1     Running   sched-lab-worker    <none>
gang-demo-abc02   1/1     Running   sched-lab-worker2   <none>
gang-demo-abc03   1/1     Running   sched-lab-worker3   <none>
gang-demo-abc04   0/1     Pending   <none>              <none>
```

**Step 3 — read the failure, line by line.**

```bash
$ kubectl describe pod gang-demo-abc04 | sed -n '/Events/,$p'
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  62s                default-scheduler  0/4 nodes are available:
           1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: },
           3 Insufficient nvidia.com/gpu.
           preemption: 0/4 nodes are available:
           1 Preemption is not helpful for scheduling,
           3 No preemption victims found for incoming pod.
  Warning  FailedScheduling  9s (x6 over 61s)   default-scheduler  (repeated)
```

*(Representative transcript — the exact wording of the aggregated message varies slightly
across minor versions; the field names and reasons do not.)*

Four things to read out of that block:

1. **`3 Insufficient nvidia.com/gpu`** — the `NodeResourcesFit` `Filter` verdict, an integer
   comparison against `allocatable` minus committed requests. This is the plugin and the
   phrase to name in an interview.
2. **`1 node(s) had untolerated taint …control-plane`** — unrelated; kind taints the control
   plane. Worth noticing so you don't misread the denominator.
3. **`No preemption victims found`** — `PostFilter`/`DefaultPreemption` ran and declined. The
   three GPU holders are same-priority siblings, and `DefaultPreemption` only evicts strictly
   lower priority. §5(c) in the flesh.
4. **`x6 over 61s`** — the retry cadence from §1(c): backoff saturating around 10s. Six
   identical failures in a minute, and it will keep going.

**Step 4 — confirm the burn.**

```bash
$ kubectl get nodes -o json | jq -r '
    .items[] | select(.metadata.name|test("worker"))
    | "\(.metadata.name)  cap=\(.status.capacity."nvidia.com/gpu")"'
sched-lab-worker   cap=1
sched-lab-worker2  cap=1
sched-lab-worker3  cap=1

# how much of that is committed?
$ kubectl get pods -A -o json | jq -r '
    [ .items[] | select(.spec.nodeName != null)
      | .spec.containers[].resources.limits["nvidia.com/gpu"] // "0" | tonumber ]
    | add | "committed GPUs: \(.)"'
committed GPUs: 3
```

Three of three fake GPUs committed, one pod pending, zero useful work. On a real cluster the
same three pods would be blocked in `init_process_group`, memory allocated, SMs at zero.

**Step 5 — price it.** Take the deadlock at the 3-GPU scale you just built, and apply the
Case-2 retry model from §10 with a *realistic* Job spec (`activeDeadlineSeconds: 3600`,
default `backoffLimit: 6` — note the demo above set `backoffLimit: 0` deliberately so you'd get
one clean observation):

```
  per-attempt hold   = 3 GPUs × 1 h                          = 3 GPU-hours
  attempts           = 1 initial + 6 retries                 = 7
  total stranded     = 7 × 3                                 = 21 GPU-hours
  cost @ $2.35/GPU-h = 21 × 2.35                             = $49.35
  wall-clock burned  = 7 h, zero training steps, no alert fired
```

Now the comparison that is the whole argument for the module. Under gang admission (L2's
`Permit`-based plugin, Kueue's quota-based admission in L3, or the native
`Gang{minCount}` path in §11), the same submission on the same cluster produces:

```
  GPUs bound        = 0
  stranded GPU-h    = 0
  time to signal    = seconds ("waiting for capacity", 4/4 pods Pending)
  the 3 free GPUs   = still available to any other job
```

**Fast-fail beats slow-fail before you even count the deadlock's own waste**, because the
timeout is the *only* self-healing path the default scheduler has and it is measured in hours,
while gang admission's signal is measured in seconds. And the second-order term is bigger than
the first: during those 7 hours, the three GPUs were not merely wasted, they were **withheld
from every other job in the cluster**.

Capture the `get pods -o wide` output and the `describe` Events block. That pair is your L1
artifact and the "before" half of the deliverable's gang demo.

## Practice

On a **kind cluster with fake GPUs** (no real hardware):

1. Create a 3-worker kind cluster and advertise `nvidia.com/gpu: 1` on each worker by patching
   node status, exactly as in the Worked example. Verify with
   `kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.'nvidia\.com/gpu'`.

2. Submit a `Job` with `spec.parallelism: 4`, `spec.completions: 4`, each pod requesting
   `nvidia.com/gpu: 1`, running `pause`/`sleep` (fake work — this exercise tests placement, not
   compute).

3. Watch placement settle with `kubectl get pods -w`. Confirm exactly **3 Running, 1 Pending**.

4. Capture the deadlock: save `kubectl get pods -o wide` and `kubectl describe pod <pending>`.
   In the Events block, identify by name: the plugin that rejected the pod, the reason string,
   the preemption verdict, and the retry cadence.

5. **Prove the retry loop is inert.** Leave it for two minutes and re-run `describe`. The
   `x<N> over <T>` counter climbs; nothing else changes. Then delete one *Running* pod and
   watch the pending pod bind within seconds — demonstrating that the scheduler is
   event-driven and healthy, and that the *only* missing input was a free GPU that only a
   sibling could release.

6. **Reproduce the mutual deadlock (§6).** With 4 fake GPUs available, submit two 2-replica
   jobs at the same time and check whether they interleave. Then scale to two 4-replica jobs on
   4 GPUs and confirm the fully-allocated/zero-productive steady state. Capture it — this is
   the more persuasive artifact of the two.

7. **Stretch (optional).** If your kind node image tracks a recent enough Kubernetes minor,
   check whether `GenericWorkload` is offered as a feature gate
   (`kube-apiserver --feature-gates=GenericWorkload=true`, plus the same on `kube-scheduler`)
   and whether `kubectl api-resources | grep -i podgroup` shows a built-in `PodGroup` under
   `scheduling.k8s.io`. Even without wiring it end to end, seeing the CRD-free, built-in shape
   of the object is worth five minutes.

**Acceptance:** a saved capture showing 3 pods bound and holding fake GPUs while the 4th pends
with `Insufficient nvidia.com/gpu`, the identified plugin/reason/preemption-verdict from the
Events block, and a one-line cost annotation using the §10 formula
(`3 GPUs × $2.35/GPU-hr = $7.05/hr stranded`, with your own rate substituted if you have one).
This is the reproduced-deadlock "before" capture for the deliverable's gang demo; L2 produces
the "after".

## Common pitfalls

- **Believing `PodDisruptionBudget` or `PriorityClass` alone fixes this.** Neither is
  group-aware. A PDB governs *voluntary evictions* of already-running pods — it is a
  constraint on the eviction API, not on admission. Priority preemption operates per-pod and,
  among same-priority siblings, does not fire at all. Neither prevents partial admission,
  because neither is evaluated at the moment partial admission happens.

- **Assuming a bigger cluster solves it.** More free GPUs lower the *probability* of hitting
  the wall but do not remove the structural gap — the deadlock is about atomicity, not
  capacity. On a busy fleet the free-GPU count hovers near zero by design (that is what high
  utilisation means), so "just add nodes" moves the incidence rate, not the mechanism. This
  ties directly into L7: allocated capacity and usable capacity are different numbers.

- **Reading `Pending` as "no resource impact".** For a pod that never got past `Filter`, that
  is true. But a pod that passed `Reserve` and is parked at `Permit` (which is exactly what
  L2's gang plugin does) is *assumed in the scheduler's cache* — its node's GPU is accounted as
  consumed even though nothing is bound in etcd. `kubectl get pods` shows `Pending`;
  `Filter` for the next pod behaves as if the GPU is gone. Do not assume the two agree.

- **Trusting a GPU-memory-allocated dashboard as a proxy for useful work.** A deadlocked job
  shows non-zero memory allocation from CUDA context and framework init while SM activity sits
  at zero. Cross-check with a compute-activity metric, not an allocation metric, and alert on
  the *conjunction* (Running + allocated + zero SM + sibling Pending) rather than any single
  signal.

- **Claiming this is fixed upstream today.** `GenericWorkload` is beta on v1.37 and **off by
  default**, and the dependent gates (`CompositePodGroup`, `PodGroupPreemptionPolicy`,
  `TopologyAwareWorkloadScheduling`) are alpha and off. Saying "Kubernetes does gang scheduling
  now" without the gate status is the kind of imprecision an interviewer will catch. Say
  instead: the primitive exists, it is beta-but-opt-in, and the quota/fairness layer on top of
  it is not part of it.

## Self-check

- **Why doesn't the default scheduler roll back the already-bound pods to unblock the
  cluster?** *Answer:* Because the scheduling cycle is per-pod and greedy with no cross-pod
  transaction. The scheduler `assume()`s a pod into its cache before `Permit` and hands the
  binding to a goroutine; the decision is final for that pod's lifetime. The only rollback
  hook, `Unreserve`, is invoked from `unreserveAndForget()` when a *later stage of that same
  pod's own cycle* fails — a `Permit` rejection or a bind error — and it has no visibility into
  or authority over sibling pods. Nothing in the default plugin set even knows that replicas
  0–5 and the pending replicas 6–7 belong to one job, so there is no way to trigger or scope a
  group rollback. Contrast this with the native pod-group path in v1.37, which maintains an
  explicit `revertFns` stack precisely so that one member's infeasibility can undo every
  member's assumed placement.

- **The scheduler retries the pending pod forever. Why doesn't that help, and what exactly is
  it waiting for?** *Answer:* Retry re-runs the same pure function over an unchanged input.
  The pending pod goes to `unschedulablePods` or `backoffQ` and is retried on an exponential
  backoff starting at 1s and capping at 10s (`DefaultPodInitialBackoffDuration`,
  `DefaultPodMaxBackoffDuration`), plus a forced flush after 5 minutes
  (`DefaultPodMaxInUnschedulablePodsDuration`), plus immediate requeue on any relevant cluster
  event. The event it needs is a GPU being freed. The only pods holding GPUs are its own
  siblings, and they will not exit because they are blocked in the collective rendezvous
  waiting for it. The loop is real and correct; the input never changes.

- **What does the deadlock cost, and how do you size it honestly?** *Answer:*
  `stranded_GPU_hours = held_GPUs × hours_held`, times your rate. Three GPUs held for an hour
  at a $2.35/GPU-hr snapshot is $7.05; the same job with `activeDeadlineSeconds: 3600` and the
  default `backoffLimit: 6` retries seven times for 21 GPU-hours ≈ $49; a mutual deadlock
  stranding a full 8-GPU node over a 64-hour weekend is 512 GPU-hours ≈ $1,203, or
  ≈ $3,158/week at that rate. But size it as a *share of fleet spend*: at 3% incidence with
  40-minute response on a 512-GPU fleet it is ~0.03% of spend, and only at 10% incidence with
  weekend-length response does it reach ~0.85%. The stronger argument is the second-order one —
  held GPUs are withheld from every other job, plus the engineer-hours lost debugging what
  presents as a networking bug.

- **Name two extension points where a gang plugin could intervene, and say which does the real
  work.** *Answer:* **`Permit`** is load-bearing: it is the only point that can hold a pod
  after its placement decision but before binding, by returning `Wait` and parking it in the
  framework's waiting-pods map with a timeout; a gang plugin counts running-plus-waiting
  members and calls `Allow()` on all of them at once when the count reaches `minMember`.
  **`PreFilter`** supports it by rejecting the whole group early when fewer than `minMember`
  pods even exist or could fit, avoiding wasted `Reserve` work. `QueueSort` keeps a group's
  pods adjacent in `activeQ` so members are attempted back to back, and `Reserve`/`Unreserve`
  hold and release the soft claim. Note the cost: a pod parked at `Permit` is already assumed
  in the cache, so a waiting gang holds resources — that is L2's head-of-line-blocking problem.

- **Why would tuning `kube-scheduler` never fix this, no matter how carefully?** *Answer:*
  Every exposed knob is a *preference over placements* — `percentageOfNodesToScore` trades
  scoring breadth for throughput, plugin weights and `NodeResourcesFit` strategies change which
  feasible node wins, backoff settings change retry cadence, priority changes who may evict
  whom, PDBs constrain voluntary eviction, spread and affinity constrain topology. The deadlock
  is a *missing constraint on admission*: "bind all `minMember` of these or none". You cannot
  express a hard constraint by adjusting a preference. `MostAllocated` bin-packing reduces
  fragmentation and therefore the *frequency* of the deadlock, which is worth knowing but is
  not a fix. The only real escape is `schedulerName` pointing at a different scheduler, or an
  admission layer above the scheduler (Kueue) — both of which are the rest of this module.

- **Why would a GPU-memory dashboard under-report this while an SM-activity metric catches
  it?** *Answer:* A deadlocked rank has already created a CUDA context and run framework/NCCL
  initialisation, both of which allocate device memory before any kernel launches, so
  memory-allocated is non-zero and looks "in use". SM activity is derived from hardware
  performance counters measuring cycles with work resident, so it reflects actual compute and
  reads near zero. The mismatch — allocated but not active, sustained, with a sibling pod
  `Pending` — is a fingerprint specific enough to alert on. The trap is that the cheap,
  always-available metric is the misleading one.

## Connections & what's next

This lesson is the diagnosis; L2 is the cure. The `Permit`-phase mechanism sketched in §2 is
exactly what L2 wires up end to end, on the same deadlocked cluster you just built — including
the cost it introduces, since a gang parked at `Permit` holds assumed resources and can block
the queue behind it. The "bigger cluster doesn't fix it" pitfall resurfaces with real numbers
in L7's fragmentation and effective-capacity math. The observability trap (allocated vs active)
is worth carrying all the way to L8, where checkpoint-survivable preemption depends on reading
GPU activity signals correctly. And the native `Workload`/`PodGroup` API from §11 returns in
L2 as the upstream successor to the plugin you install, and in L3 as the reason Kueue's value
is the *quota* layer rather than the *admission* layer.
**Next: [02 — Gang scheduling: all-or-nothing admission](02-gang-scheduling.md)**, which takes
this exact cluster and this exact job and turns "3 running, 1 stranded" into "4 pending
together, 0 GPUs wasted, then 4 running together."

## References & further reading

**Primary sources — read directly from cloned repositories this session**

Note on method: this environment's egress proxy blocks `kubernetes.io` and several vendor
documentation domains. Rather than cite pages that could not be reached, the version-specific
claims above were verified against upstream *source trees* cloned during this session. Where a
canonical doc URL is given below for convenience, its reachability status is stated honestly.

- **`kubernetes/kubernetes`, v1.37 development head** — https://github.com/kubernetes/kubernetes.
  Read `pkg/scheduler/schedule_one.go` for `ScheduleOne`, `schedulingCycle`,
  `prepareForBindingCycle`, `assumeAndReserve`, `unreserveAndForget`, and the
  `minFeasibleNodesToFind = 100` / `minFeasibleNodesPercentageToFind = 5` constants;
  `pkg/scheduler/backend/queue/scheduling_queue.go` for the 1s/10s/5m queue defaults;
  `pkg/scheduler/framework/interface.go` for the ordered extension-point contract and the
  `RunPermitPlugins` / `WaitOnPermit` split. **Cloned and read directly this session.**
- **`kubernetes/kubernetes` — `pkg/features/kube_features.go`.** The authoritative statement of
  gate lifecycle: `GenericWorkload` alpha v1.35 → **beta v1.37 with `Default: false`**;
  `TopologyAwareWorkloadScheduling` alpha v1.36; `CompositePodGroup` and
  `PodGroupPreemptionPolicy` alpha v1.37 with declared dependencies on `GenericWorkload`.
  **Cloned and read directly this session.** *(This corrects the previous version of this
  lesson, which described beta as "targeted for v1.37"; it landed as beta, still off by
  default.)*
- **`kubernetes/kubernetes` — `pkg/apis/scheduling/types.go` and
  `staging/src/k8s.io/api/core/v1/types.go`.** Read for the `Workload` / `PodGroup` split,
  `PodGroupSpec.schedulingPolicy` as a `Basic`/`Gang{minCount}` union with `minCount`'s
  mutability and eventual-consistency caveats, the 8-template limit, and the pod-side
  `spec.schedulingGroup.podGroupName` opt-in (with the tombstoned `workloadRef` protobuf tag 42
  recording the v1.36 rename). **Cloned and read directly this session.**
- **`kubernetes/kubernetes` — `pkg/scheduler/schedule_one_podgroup.go`.** Read for the
  group-scheduling algorithm and its `revertFns` stack — the concrete implementation of
  simulate-and-roll-back that the per-pod path structurally lacks. **Cloned and read directly
  this session.**
- **kubernetes/enhancements — KEP-4671, "Gang Scheduling via Workload API"** —
  https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/4671-gang-scheduling/README.md.
  Read for the design rationale behind the `Workload`/`PodGroup` split and the rollout plan.
  **Fetched and verified in an earlier session; the API details above were re-verified against
  the source tree this session because the KEP text and the merged API can drift.**
- **kubernetes-sigs/scheduler-plugins — `pkg/coscheduling/README.md`** —
  https://github.com/kubernetes-sigs/scheduler-plugins. The maintainers' own before/after demo
  (6 nginx replicas, 3 fit; `minMember: 3` → 3 Running + 3 Pending; `minMember: 4` → all 6
  Pending) plus the statement that `queueSort`, `permit` and `unreserve` are mandatory and
  `preFilter` is an optional early-exit optimisation. **Cloned and read directly this session.**
  L2 covers this in depth.

**Real-world engineering accounts**

- **OpenAI — "Scaling Kubernetes to 7,500 Nodes"** —
  https://openai.com/index/scaling-kubernetes-to-7500-nodes/ — a gang scheduling plugin built
  and run in production at 7,500-node scale specifically to stop MPI jobs deadlocking on
  partial placement, alongside team taints and balloon deployments for fair capacity
  distribution. *(Search-verified; direct fetch blocked by this environment's egress proxy.)*
- **Lambda — "Why your Kubernetes scheduler can't handle AI workloads"** —
  https://lambda.ai/blog/why-your-kubernetes-scheduler-cant-handle-ai-workloads — a GPU-cloud
  vendor's independent restatement of the same mechanism, with its own Kueue-vs-Volcano
  comparison. *(Search-verified; fetch blocked by egress this session.)*

**Deeper dives**

- **Kubernetes blog — "Introducing Workload Aware Scheduling" (v1.35) and "Advancing
  Workload-Aware Scheduling" (v1.36)** — https://kubernetes.io/blog/. SIG-Scheduling's own
  narrative framing of why per-pod scheduling is insufficient for workloads. *(Both
  search-verified; `kubernetes.io` is unreachable from this environment, which is why the
  technical claims in §11 are sourced from the code rather than from these posts.)*
- **Module 02 (this course)** — the scheduling cycle and the `Filter`/`Score`/`Reserve`/`Permit`
  extension points. Reread its cycle diagram alongside §1 and §2 above and the hole becomes
  structural rather than incidental.
