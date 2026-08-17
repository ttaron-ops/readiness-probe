---
lesson: "06.5"
title: "Scheduler alternatives — Volcano, NVIDIA KAI, and when to pick which"
module: "06"
concept: "Scheduler alternatives — Volcano, NVIDIA KAI, and when to pick which"
status: not-started
est_time: "9h"
prev: "04-kueue-cohorts-borrowing-preemption.md"
next: "06-topology-aware-placement.md"
artifacts: []
sources: 14
---

# 06.5 · Scheduler alternatives — Volcano, NVIDIA KAI, and when to pick which

> **Concept.** Kueue, Volcano, and NVIDIA KAI solve overlapping but differently-shaped problems; pick by tenancy model and workload lineage, and know they can compose rather than compete.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

Lessons 3–4 built a complete mental model of Kueue: quota-first admission, cohorts,
borrowing, and two flavours of fairness (`AdmissionFairSharing` within a queue, cohort
`fairSharing` across queues). That model assumed one philosophy — cloud-native,
quota-owns-fairness, gang-delegated-elsewhere. This lesson pressure-tests the assumption by
putting two structurally different schedulers next to it. **Volcano** owns gang and
fairness natively inside its own scheduling loop instead of delegating them. **NVIDIA KAI**
(the open-sourced Run:ai engine) does the same and adds two things Kueue has no concept of
at all: sub-GPU sharing and active defragmentation of running work.

What this unlocks: the ability to answer *"why not just use Kueue for everything?"* with an
architectural argument rather than a preference, and to design a fleet where more than one
scheduler coexists because different tenancy shapes need different primitives.

Everything below is checked against source, not marketing pages. Volcano facts come from
the `volcano-sh/volcano` tree (the `staging/src/volcano.sh/apis` API types, the
`pkg/scheduler/actions` and `pkg/scheduler/plugins` trees, the shipped
`installer/helm/chart/volcano/config/volcano-scheduler.conf`, and `docs/design/`). KAI facts
come from the `NVIDIA/KAI-Scheduler` tree (`pkg/apis/scheduling/v2`, `pkg/scheduler/actions`,
and the in-repo `docs/`). Kueue facts come from `kubernetes-sigs/kueue` (`apis/kueue/v1beta2`,
`pkg/scheduler`, and `site/content/en/docs`). Where a field name or default is quoted, it is
the one in the tree, at the commit stated in References — **verify against the exact tag you
deploy**, because two of these three projects are moving fast.

## Why this matters

In a GPU-heavy interview loop at CoreWeave, NVIDIA, or a neocloud, "we use Kueue" is a
starting position, not an answer. The follow-up is always: *why not Volcano? why not KAI?
what happens when a tenant needs fractional GPUs, or when an MPI job needs gang semantics
Kueue does not own?* CoreWeave's Principal/Staff Cluster Orchestration JD asks for "a
technical authority on scheduling, quota enforcement, fairness, pre-emption, and
multi-tenant GPU isolation" across Kubernetes, Slurm, SUNK, *and* Kueue — the plural is the
point.

The failure mode this lesson prevents is picking a scheduler by familiarity and discovering
at scale that it structurally cannot express what a tenant needs. You cannot retrofit
dominant-resource fairness, or fractional-GPU sharing, or running-pod consolidation onto a
scheduler that has no concept of any of them; those are properties of the scheduling loop,
not plugins you bolt on.

The cost angle is sharper than the feature angle. Three concrete numbers you will derive or
defend later in this module:

- A whole H100 pinned by one Jupyter notebook at single-digit utilisation is roughly
  **$2–$4/GPU-hour of pure waste in the neocloud segment**, or $17k–$35k/year, *per
  notebook*. A fleet with 200 such notebooks is a seven-figure line item that a whole-device
  scheduler cannot touch no matter how well its quota logic is tuned.
- Stranded fragments — 3 free GPUs here, 3 there, and an 8-GPU job that cannot start — are
  free capacity you are already paying for (lesson 06.7 makes this arithmetic exact). A
  scheduler with a consolidation action attacks that stock directly; one without it can only
  wait for jobs to finish.
- A preemption that tears one pod out of a healthy 8-pod gang does not free 1 GPU. It
  *strands 7 more*, because the remaining ranks cannot make progress without their missing
  peer. Whether your scheduler evicts at pod granularity or gang granularity is therefore a
  correctness property with a direct dollar consequence.

## What's new here (calibration)

Everything through lesson 4 assumed one scheduling philosophy: quota-first, cloud-native,
gang delegated. You know Kueue's `Workload`/`ClusterQueue`/cohort model cold — it is not
re-taught. New here:

- **The session/action/plugin architecture.** Volcano and KAI are both built as a *loop* that
  snapshots cluster state, then runs an ordered list of **actions**, each of which consults
  **plugins** through registered callback functions. This is a genuinely different shape from
  Kueue's controller-plus-webhook admission layer, and once you can draw it, most of the
  behavioural differences fall out for free.
- **Dominant Resource Fairness**, derived rather than named: the formula, the worked
  allocation, and where it lives in Volcano's code.
- **Gang-granularity eviction** as implemented — the `gangpreempt` and `gangreclaim` actions,
  the "safe bundle" versus "whole bundle" split, and the ordering function that decides which
  bundle dies first.
- **Two distinct verbs that most people conflate**: KAI (and Volcano) draw a hard line between
  **preempt** (intra-queue, priority-driven) and **reclaim** (inter-queue, quota-driven). Getting
  this distinction right is the single fastest way to sound like you have run one of these.
- **Fractional GPU as three different mechanisms** — KAI's scheduler-level fraction, Volcano's
  `deviceshare` vGPU path, and MIG's hardware partitioning — with different isolation
  guarantees. The naive "only KAI does fractional GPUs" claim is **wrong**, and this lesson
  corrects it.

## Core concepts

### 1. Three answers to one question, and the question is not "how do I schedule pods"

All three projects exist because vanilla `kube-scheduler` answers exactly one question:
*given this single pod, which node should it bind to?* It is a per-pod, greedy, stateless-
between-pods loop. Every requirement in this module is a *set*-level requirement — all eight
pods or none; this team gets 40% of the GPUs; this job must land inside one rack; this
workload may take resources from that one — and none of them can be expressed one pod at a
time.

The three projects diverge on **where** the set-level logic lives:

- **Kueue** puts it *above* the scheduler. A `Workload` object is held suspended until the
  quota arithmetic says yes; then the job is unsuspended and ordinary `kube-scheduler` places
  the pods. Kueue never binds a pod. Its whole job is the admission decision, and its whole
  state is quota accounting.
- **Volcano** puts it *inside a replacement scheduler*. `volcano-scheduler` is a separate
  binary with its own cache, its own scheduling loop, and its own binder. Pods opt in with
  `schedulerName: volcano`. Because it does the binding, it can make placement decisions that
  are conditional on the whole set.
- **KAI** does the same as Volcano — a full replacement scheduler with `schedulerName:
  kai-scheduler` — but was built inside a commercial product (Run:ai) whose selling point was
  GPU utilisation, so its loop has actions Volcano's does not: consolidation, and a device
  model where a GPU is a divisible float rather than an integer.

That is the whole taxonomy. Draw it once:

```
   ═══════════════════════════════════════════════════════════════════════════════
    WHERE DOES THE SET-LEVEL DECISION LIVE?
   ═══════════════════════════════════════════════════════════════════════════════

    KUEUE — decision ABOVE the scheduler          VOLCANO / KAI — decision IS the scheduler
    ────────────────────────────────────          ─────────────────────────────────────────

    Job (suspend: true)                            Job / PyTorchJob / MPIJob / Pod
        │                                              │
        │ webhook creates                              │ webhook (Volcano) or PodGrouper (KAI)
        ▼                                              │ creates / infers
    ┌─────────────────────────┐                        ▼
    │ Workload                │                   ┌──────────────────────────┐
    │  podSets[]              │                   │ PodGroup                 │
    │  priority               │                   │  minMember / minTaskMember│
    │  queueName ──────────┐  │                   │  minResources            │
    └─────────────────────────┘                   │  queue ───────────────┐  │
        │                  │                      │  networkTopology      │  │
        ▼                  ▼                      └──────────────────────────┘
    ┌───────────┐   ┌──────────────┐                   │                   │
    │ LocalQueue│──▶│ ClusterQueue │                   ▼                   ▼
    └───────────┘   │  nominalQuota│              ┌─────────────────────────────┐
                    │  borrowing   │              │ Queue (cluster-scoped)      │
                    │  preemption  │              │  Volcano: weight/deserved/  │
                    └──────┬───────┘              │    guarantee/capability/    │
                           │ cohort               │    parent/priority          │
                    ┌──────▼───────┐              │  KAI:  parentQueue /        │
                    │   Cohort     │              │    resources{quota,limit,   │
                    │ (borrow/lend)│              │    overQuotaWeight}/priority│
                    └──────┬───────┘              └──────────────┬──────────────┘
                           │                                     │
              ADMIT: unsuspend the Job                   ┌───────▼───────────────┐
                           │                             │ SCHEDULING SESSION    │
                           ▼                             │  snapshot cache       │
                 ┌───────────────────┐                   │  run ACTIONS in order │
                 │  kube-scheduler   │                   │  each action asks     │
                 │  (+ coscheduling  │                   │  PLUGINS via callbacks│
                 │   plugin for gang)│                   └───────┬───────────────┘
                 └─────────┬─────────┘                           │
                           ▼                                     ▼
                     bind pod → node                       bind pod → node
                                                           evict pod (preempt/reclaim/
                                                                      consolidate)

    Kueue NEVER evicts a running pod to make room                Volcano/KAI DO — eviction
    for placement; it only evicts (deletes) a whole              is a first-class action in
    Workload and requeues it.                                    the loop.
   ═══════════════════════════════════════════════════════════════════════════════
```

Everything else in this lesson is detail hanging off that picture. Note especially the last
box: **Kueue's unit of eviction is a Workload; Volcano's and KAI's unit of eviction is a
pod**, chosen by an action that runs every scheduling cycle. That difference explains why
Kueue cannot defragment and why Volcano needed a *separate* gang-aware eviction path.

### 2. Volcano's scheduling session, mechanically

Volcano's scheduler runs a loop whose period defaults to one second. Each iteration opens a
**session**: it takes a consistent snapshot of jobs, queues, nodes and hyper-nodes from its
cache, instantiates the configured plugins, lets each plugin register callback functions on
the session, then executes the configured **actions** in order. At the end the session is
closed and plugin state is discarded.

The configuration is a single YAML document. This is the one that actually ships in the Helm
chart (`installer/helm/chart/volcano/config/volcano-scheduler.conf`) — reproduce it exactly,
because **the default is much less capable than people assume**:

```yaml
actions: "enqueue, allocate, backfill"
tiers:
- plugins:
  - name: priority
  - name: gang
    enablePreemptable: false
  - name: conformance
- plugins:
  - name: overcommit
  - name: drf
    enablePreemptable: false
  - name: predicates
  - name: proportion
  - name: nodeorder
  - name: binpack
```

Read the two load-bearing facts out of that listing:

1. **`preempt` and `reclaim` are not in the default action list.** A stock Volcano install
   does not preempt anything. If your mental model of Volcano is "the scheduler that
   preempts", you are describing a configuration you have to write. The repo's CI config
   (`volcano-scheduler-ci.conf`) is the one that turns them on:
   `actions: "enqueue, allocate, backfill, reclaim, preempt"`.
2. **Plugins are grouped into ordered `tiers`.** A tier is a veto group: for the callback
   families that are filters (is this victim evictable? may this job enqueue?), all plugins in
   tier 1 are consulted before tier 2, and a rejection in an earlier tier short-circuits. This
   is how `gang` and `priority` get to overrule `drf` and `binpack`.

#### The action pipeline

Volcano registers eight actions (`pkg/scheduler/actions/factory.go`). You compose the list you
want; order matters because each action mutates the session snapshot the next one sees.

| Action | What it does | Evicts? |
|---|---|---|
| `enqueue` | Moves `PodGroup`s from `Pending` to `Inqueue` when the queue has room for `minResources`. This is Volcano's admission gate — the analogue of Kueue's quota check. Job controllers are told not to create pods until the PodGroup is `Inqueue`. | no |
| `allocate` | The main placement pass. For each queue (ordered by `QueueOrderFn`), for each job (ordered by `JobOrderFn`), for each task: run predicates, score nodes with `NodeOrderFn`, and either **allocate** (bind now) or **pipeline** (reserve against resources that are being released). | no |
| `backfill` | A second pass that places small / best-effort tasks into holes the main pass left. | no |
| `preempt` | Intra-queue: a higher-priority job takes resources from lower-priority jobs **in the same queue**. Pod-granularity victims. | yes |
| `reclaim` | Inter-queue: a queue below its `deserved` takes resources back from a queue above its `deserved`. | yes |
| `gangpreempt` | Intra-queue preemption where victims are chosen as **bundles** at job granularity and the preemptor is placed as a whole gang (§2.4). | yes |
| `gangreclaim` | The same bundle logic applied to the inter-queue reclaim case. | yes |
| `shuffle` | Evicts tasks selected by plugin `VictimTasksFn`s — the hook the `rescheduling` plugin uses to rebalance. | yes |

Draw the cycle, because the *ordering* is the mechanism:

```
  ONE VOLCANO SCHEDULING SESSION  (default period 1s)
  ═══════════════════════════════════════════════════════════════════════════════

   t0  OpenSession
       ├─ snapshot: Jobs (from PodGroups)  Queues  Nodes  HyperNodes
       └─ for each tier, for each plugin: plugin.OnSessionOpen(ssn)
              registers callbacks:  JobValidFn      (gang: enough pods created?)
                                    JobEnqueueableFn(proportion/capacity: room?)
                                    JobOrderFn      (gang: unready first; priority: value)
                                    QueueOrderFn    (drf: lowest dominant share first)
                                    JobReadyFn      (gang: ReadyTaskNum >= minMember)
                                    JobPipelinedFn  (gang: allocated+pipelined >= minMember)
                                    JobStarvingFn   (gang: still short of minMember)
                                    PredicateFn     (predicates, deviceshare)
                                    NodeOrderFn     (nodeorder, binpack, NTA)
                                    PreemptableFn   (gang, priority, drf, conformance)
                                    ReclaimableFn   (proportion/capacity, gang)
   ─────────────────────────────────────────────────────────────────────────────
   ENQUEUE      PodGroup.Pending ──▶ Inqueue  when queue has room for minResources
                                     (job controller only now creates the pods)
   ─────────────────────────────────────────────────────────────────────────────
   ALLOCATE     for queue in queues sorted by QueueOrderFn:
                  for job in jobs sorted by JobOrderFn:
                    for task in tasks:
                       nodes := Predicate(task)              ← filter
                       node  := argmax NodeOrder(task, node) ← score (binpack, NTA…)
                       Allocate(task→node)  or  Pipeline(task→node)
                  ── statement is COMMITTED only if JobPipelinedFn(job) is true
                     ▲ THIS is gang scheduling: a job that cannot reach minMember
                       has its whole statement DISCARDED, so nothing is bound.
   ─────────────────────────────────────────────────────────────────────────────
   PREEMPT      (only if configured) same-queue, higher priority takes from lower
   RECLAIM      (only if configured) queue under `deserved` takes from queue over
   GANGPREEMPT  (only if configured) same, but victims chosen as job-level bundles
   GANGRECLAIM
   ─────────────────────────────────────────────────────────────────────────────
   BACKFILL     place best-effort / small tasks into whatever holes remain
   ─────────────────────────────────────────────────────────────────────────────
   t1  CloseSession   (plugin state discarded; next session re-snapshots)
  ═══════════════════════════════════════════════════════════════════════════════
```

**The commit-or-discard step in `allocate` is the entire gang mechanism, and it is worth
saying precisely.** Volcano does not place seven pods and then notice the eighth does not
fit. It accumulates the placements in a `Statement` — an in-memory transaction log of
allocate/pipeline operations against the snapshot — and only calls `Commit()` (which issues
the actual binds) if `JobPipelined(job)` is true, meaning the job's allocated-plus-pipelined
task count has reached `minMember`. Otherwise it calls `Discard()`, which rolls the snapshot
back and binds nothing. **This is the transactional property `kube-scheduler` lacks and the
direct cure for the lesson-01 deadlock.**

#### PodGroup: what gang actually means here

`PodGroup` is `scheduling.volcano.sh/v1beta1`. The spec fields that matter
(`staging/src/volcano.sh/apis/pkg/apis/scheduling/v1beta1/types.go`):

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: mpi-allreduce
  namespace: research
spec:
  minMember: 8                       # gang size: bind nothing until 8 can be placed
  minTaskMember:                     # per-role minimums, for heterogeneous jobs
    launcher: 1                      #   (superseded by subGroupPolicy, see below)
    worker: 8
  minResources:                      # what `enqueue` checks against the queue
    nvidia.com/gpu: "8"
    cpu: "64"
    memory: 512Gi
  queue: hpc-research                # cluster-scoped Queue object
  priorityClassName: research-high   # ordinary k8s PriorityClass
  networkTopology:                   # see 06.6 — native topology constraint
    mode: hard                       # hard | soft
    highestTierAllowed: 2            # may not cross above tier 2 of the HyperNode tree
```

Its status carries `phase` — one of `Pending`, `Inqueue`, `Running`, `Unknown`, `Completed` —
and the `Inqueue` phase is the interesting one: it is a *state that exists only because of
the enqueue gate*, letting the job controller hold off on creating pods that the scheduler
has already decided it cannot place. `kubectl get podgroup` prints `STATUS`, `minMember`,
`RUNNINGS`, `AGE`, and `QUEUE`.

Pods join a PodGroup by annotation. Two keys are accepted
(`.../scheduling/v1beta1/labels.go`):

```yaml
metadata:
  annotations:
    scheduling.k8s.io/group-name: mpi-allreduce        # KubeGroupNameAnnotationKey
    # or, equivalently:
    scheduling.volcano.sh/group-name: mpi-allreduce    # VolcanoGroupNameAnnotationKey
    scheduling.volcano.sh/queue-name: hpc-research     # QueueNameAnnotationKey
spec:
  schedulerName: volcano
```

Newer Volcano adds `spec.subGroupPolicy`, which the API comments describe as the long-term
replacement for `minTaskMember`: it splits a PodGroup's pods into sub-groups by label, gives
each sub-group a `subGroupSize`, sets `minSubGroups` (how many sub-groups must be
simultaneously satisfiable before anything schedules), and attaches a per-sub-group
`networkTopology`. That is the shape you need for "16 tensor-parallel ranks must share one
NVLink domain, and I need at least 4 such groups before the job is worth starting" — a
constraint neither Kueue's flat PodSet model nor a plain `minMember` can express.

#### Queue: Volcano's quota model is richer than Kueue's, in a specific way

`Queue` is cluster-scoped, `scheduling.volcano.sh/v1beta1`:

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: hpc-research
spec:
  weight: 4                       # proportion plugin: share of cluster ∝ weight
  reclaimable: true               # may this queue's over-deserved usage be taken back?
  parent: research-org            # hierarchical queues
  priority: 10                    # higher = allocated earlier, reclaimed later
  dequeueStrategy: traverse       # traverse | fifo  (fifo = head-of-line blocking)
  guarantee:
    resource:                     # NEVER lent out, even when the queue is idle
      nvidia.com/gpu: "8"
  deserved:                       # the queue's entitlement; lendable, reclaimable
    nvidia.com/gpu: "32"
    cpu: "256"
  capability:                     # hard ceiling; usage may never exceed this
    nvidia.com/gpu: "64"
```

Map that onto Kueue so the comparison is exact:

| Volcano `Queue` field | Kueue equivalent | Difference that matters |
|---|---|---|
| `deserved` | `nominalQuota` on a ClusterQueue flavor | Same idea: your entitlement, lendable when idle, reclaimable when you want it back. |
| `capability` | `borrowingLimit` + `nominalQuota` (the effective ceiling) | Volcano expresses the ceiling absolutely; Kueue expresses it as a delta above nominal. |
| `guarantee` | *no direct equivalent* | Kueue's `lendingLimit` caps how much you lend; Volcano's `guarantee` reserves capacity that is never lent **and can be node-locked** (`status.reservation.nodes`). |
| `weight` | fair-sharing weight | Volcano's `proportion` plugin divides the *cluster* by weight; Kueue's fair sharing divides *borrowable* capacity. |
| `parent` | Cohort hierarchy | Both support trees now. |
| `priority` | ClusterQueue is not itself prioritised | Volcano orders queues for both allocation and reclaim by `priority` before falling through to share. |
| `dequeueStrategy: fifo` | `queueingStrategy: StrictFIFO` | Same head-of-line-blocking semantics. |
| `reclaimable: false` | roughly `reclaimWithinCohort: Never` on *other* queues | Volcano puts the flag on the victim; Kueue puts the policy on the preemptor. |

The `deserved`/`guarantee`/`capability` triple lives in the `capacity` plugin, which the
design doc (`docs/design/capacity-scheduling.md`) introduced explicitly to decouple absolute,
per-resource-type entitlements from the older `proportion` plugin's single scalar `weight`.
The motivation given is exactly the heterogeneous-GPU case: *"the share ratio of A100 GPU
between ORG1 and ORG2 is 1:3, however, the share ratio of V100 GPU between ORG1 and ORG2 is
1:1."* A single weight cannot express that; a per-resource `deserved` can. **Use `capacity`,
not `proportion`, on a heterogeneous GPU fleet** — they are mutually exclusive choices in the
plugin list.

#### Dominant Resource Fairness, derived

DRF is the other half of Volcano's fairness story, and it is worth deriving rather than
naming, because Kueue's cohort fair-sharing rests on the same idea.

The problem: two tenants, one asks for GPU-heavy tasks and the other for CPU-heavy tasks.
"Equal share" is ambiguous — equal in what? Give each 50% of CPUs and 50% of GPUs and you
waste both. Ghodsi et al. (NSDI 2011) answer: for each tenant, find the resource of which it
is consuming the largest *fraction of the cluster's total* — its **dominant resource** — and
equalise the tenants' dominant shares.

Volcano's implementation is four lines
(`pkg/scheduler/plugins/drf/drf.go`, `calculateShare`):

```
  for each resource r in the cluster:
      share_r = allocated_r / total_r
  dominant_share = max over r of share_r
  dominant_resource = argmax over r of share_r
```

Worked, on a cluster of **64 GPUs and 1,024 CPUs**:

```
  Tenant A currently holds  16 GPU,  64 CPU
      share(GPU) = 16/64   = 0.250        ← dominant
      share(CPU) = 64/1024 = 0.0625
      dominant_share(A) = 0.250, dominant resource = nvidia.com/gpu

  Tenant B currently holds   4 GPU, 512 CPU
      share(GPU) =  4/64   = 0.0625
      share(CPU) = 512/1024= 0.500        ← dominant
      dominant_share(B) = 0.500, dominant resource = cpu

  QueueOrderFn / JobOrderFn: LOWER dominant share is served first.
      → A (0.250) is served before B (0.500), even though B holds
        one-quarter of A's GPUs. B is already consuming half the cluster
        along the axis that binds it.

  Give A one more 8-GPU task:
      share(GPU) = 24/64 = 0.375 → still below B's 0.500, A is served again.
  Give A another:
      share(GPU) = 32/64 = 0.500 → now tied; the tie-break falls through to
        the next JobOrderFn in the tier list (priority, then creation time).
```

The property that makes DRF worth the trouble: **a tenant cannot improve its allocation by
lying about what it needs.** Inflating a CPU request only raises your CPU share, and if CPU
is not your dominant resource it changes nothing; if the inflation makes CPU dominant, you
get served *less*. That strategy-proofness is why the algorithm survived from Mesos into
Volcano, YARN, and (in a quota-flavoured form) Kueue.

DRF also registers a `PreemptableFn`: a preemptor may only take from a preemptee if doing so
does not invert their dominant shares — i.e. if the preemptor's post-preemption share is
still ≤ the preemptee's. That is DRF acting as a *veto* on preemption, layered on whatever
priority says.

#### Gang-granularity eviction: the bundle model

Now the part that most write-ups get wrong. Volcano's older `preempt`/`reclaim` actions pick
victims **task by task**. On a fleet of gangs that is not merely inefficient, it is
incorrect:

```
  POD-GRANULARITY PREEMPTION AGAINST A GANG  — why it strands more than it frees
  ───────────────────────────────────────────────────────────────────────────────

  victim job V:  minMember = 8, running 8/8            preemptor P needs 2 GPUs
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │V0 │V1 │V2 │V3 │V4 │V5 │V6 │V7 │   all-reduce every step, all 8 required
  └───┴───┴───┴───┴───┴───┴───┴───┘

  evict V6, V7  ──▶
  ┌───┬───┬───┬───┬───┬───┬─ ─┬─ ─┐
  │V0 │V1 │V2 │V3 │V4 │V5 │   │   │   ReadyTaskNum = 6 < minMember = 8
  └───┴───┴───┴───┴───┴───┴───┴───┘   → V makes ZERO progress
       └──────── 6 GPUs still held, producing nothing ────────┘

  ACCOUNTING:  freed for P = 2 GPUs.   Stranded by the eviction = 6 GPUs.
               Net capacity change     = −4 GPU-equivalents of useful work.
```

The `gangpreempt` and `gangreclaim` actions
(`pkg/scheduler/actions/gangpreempt/gangpreempt.go`, `.../gangreclaim/`,
`pkg/scheduler/actions/utils/bundle.go`) fix this by never treating a task as an independent
victim. Instead, candidate victims are grouped into **bundles** of one of two types:

- **`BundleSafe`** — a set of a job's tasks whose removal leaves the job still ready. The code
  computes three surpluses to decide what is safe:
  `jobSurplus = ReadyTaskNum() − MinAvailable`,
  `roleSurplus[r] = allocated(r) − TaskMinAvailable[r]`, and
  `groupSurplus[g] = readySubJobs(g) − MinSubJobs[g]`. Only tasks covered by a positive
  surplus in *every* applicable dimension go into a safe bundle. If `jobSurplus < 0` the job
  is already broken and the whole thing becomes one bundle.
- **`BundleWhole`** — the entire job. Evicting it is coherent: the job dies cleanly and
  re-queues, rather than limping.

Bundles are then ordered by `SortBundlesForPreempt` (respectively `...ForReclaim`), which
compares, in order:

1. **Bundle type** — `BundleSafe` before `BundleWhole`. *Take surplus replicas before you
   kill a job.*
2. **Victim job order** — the inverse of the session's `JobOrderFn`, so the least-important
   job (lowest priority, or lowest DRF standing) is preferred as a victim. For `reclaim`, the
   inverse of the `QueueOrderFn` instead.
3. **Bundle ROI** — how much of the preemptor's *need* this bundle actually satisfies, highest
   first. This is what stops the scheduler from evicting five bundles when one would do.
4. A stable tie-break so the decision is deterministic across sessions.

Then, per candidate topology domain (up to `maxDomains`, default 8), it accumulates bundles
until `domainIdle + freed ≥ jobNeed`, builds a full nomination plan for the *whole* preemptor
gang inside that domain, and only commits if the plan validates. `allowWholeBundle` (default
`true`) can be set to `false` to forbid whole-job eviction entirely.

Two behavioural notes straight from the source that you will not find in a blog post:

- Both gang actions restrict victims to **the preemptor's own queue** for `gangpreempt`
  (`victimJob.Queue != preemptorJob.Queue → skip`) and require
  `preemptorJob.Priority > victimJob.Priority`. Cross-queue movement is `gangreclaim`'s job.
  This is the same preempt/reclaim split KAI draws explicitly.
- The legacy `preempt` action refused to evict non-BestEffort victims when the preemptor was
  BestEffort; `gangpreempt` deliberately drops that special case and delegates eligibility to
  the session's `UnifiedEvictable` plugin chain instead. If you are comparing behaviour across
  versions, that is a real difference, not a bug.

#### The `binpack` plugin, with its actual scoring function

Fragmentation control in Volcano is a `NodeOrderFn`. The `binpack` plugin
(`pkg/scheduler/plugins/binpack/binpack.go`) scores each candidate node by how *full* it would
become:

```
  For one resource r that the task requests:

      score_r = (requested_r + already_used_r) / allocatable_r  × weight_r
      (and the node is rejected outright if requested_r + used_r > allocatable_r)

  Node score = ( Σ_r score_r  /  Σ_r weight_r )  ×  MaxNodeScore(=100) × binpack.weight

  Defaults (all initialised to 1):
      binpack.weight = 1, binpack.cpu = 1, binpack.memory = 1
      per-resource weights via  binpack.resources / binpack.resources.<name>
```

The consequence: the node that ends up **closest to full** wins. That is the exact inverse of
`kube-scheduler`'s default `LeastAllocated` bias, and it is the setting that decides whether
your free GPUs coalesce into whole-node holes or smear across the fleet. On 8-GPU boxes you
want GPU weighted far above CPU/memory:

```yaml
- name: binpack
  arguments:
    binpack.weight: 10                      # how much binpack matters vs other NodeOrderFns
    binpack.cpu: 1
    binpack.memory: 1
    binpack.resources: nvidia.com/gpu
    binpack.resources.nvidia.com/gpu: 10    # GPU fullness dominates the score
```

Lesson 06.7 quantifies what that change is worth.

#### Volcano does have fractional GPUs — correcting a common claim

The widely repeated line "Volcano is whole-device only, KAI is the one with fractional GPUs"
is **false**. Volcano ships a `deviceshare` plugin with two independent NVIDIA sharing paths
(`pkg/scheduler/plugins/deviceshare/`, `pkg/scheduler/api/devices/nvidia/`):

| Path | Plugin argument | Resources a pod requests | Semantics |
|---|---|---|---|
| GPU-share | `deviceshare.GPUSharingEnable: true` | `volcano.sh/gpu-memory` (MiB) | Multiple pods per physical GPU, packed by memory. |
| GPU-number | `deviceshare.GPUNumberEnable: true` | `volcano.sh/gpu-number` | Whole-device counting with device-index awareness. |
| vGPU (HAMi-derived) | `deviceshare.VGPUEnable: true` | `volcano.sh/vgpu-number`, `volcano.sh/vgpu-memory`, `volcano.sh/vgpu-memory-percentage`, `volcano.sh/vgpu-cores` | Memory **and compute-core** limits, enforced by the HAMi core library; also handles MIG geometry. |

The plugin refuses contradictory configurations at startup —
`klog.Fatal("gpu-share and vgpu can't be used together")` and the same for gpu-share versus
gpu-number — so you pick exactly one model per cluster. The honest comparison is therefore
not "who has fractional GPUs" but **what isolation each fractional path gives you**, which is
§4's table.

### 3. NVIDIA KAI, mechanically

KAI-Scheduler is the Run:ai scheduling engine, open-sourced by NVIDIA and now developed in
the open at `NVIDIA/KAI-Scheduler`. Its API group is still `scheduling.run.ai` — a fossil of
its commercial origin, and a thing to check against your tag rather than assume.

#### The scheduling cycle

KAI's loop is the same session/action shape as Volcano's, with a different action list
(`docs/scheduling-deep-dive/README.md`, `pkg/scheduler/actions/`):

```
  KAI SCHEDULING CYCLE — non-disruptive actions first, disruptive last
  ═══════════════════════════════════════════════════════════════════════════

   1. ALLOCATE            place pending workloads on free capacity.   no eviction
   2. CONSOLIDATE         repack running workloads to defragment.     temporary
                          A pod is evicted ONLY if the scheduler has  eviction
                          already found it a new home.                 only
   3. RECLAIM             INTER-queue. Take back over-quota usage      eviction
                          from another queue for a queue under its
                          fair share / deserved quota.
   4. PREEMPT             INTRA-queue. Higher-priority workload evicts eviction
                          a lower-priority preemptible one in the
                          SAME queue.
   5. STALEGANGEVICTION   kill jobs stuck below their minMember so     eviction
                          they stop holding resources hostage.
  ═══════════════════════════════════════════════════════════════════════════
   Configurable per shard via the `actions` field on the SchedulingShard CRD.
```

Three things to take from that ordering:

- **Consolidate runs before any disruptive action.** KAI tries to solve "the job does not fit"
  by *moving* things before it tries to solve it by *killing* things. Kueue has no equivalent
  step at all; Volcano's nearest analogue is the `shuffle` action driven by the `rescheduling`
  plugin, which is a rebalancing hook rather than a fit-driven defrag.
- **`StaleGangEviction` is the answer to a failure mode this module keeps circling.** A gang
  that has fallen below `minMember` — because a node died, or because something evicted one
  of its pods — is holding GPUs and producing nothing. KAI has an explicit action whose whole
  job is to detect and destroy those. This is worth naming in an interview because it closes
  the loop on lesson 01's deadlock from the *runtime* side rather than the admission side.
- The whole cycle is scoped to a **shard**. Nodes, queues and pod groups carry a
  `kai.scheduler/node-pool` label; each shard runs its own scheduler instance with its own
  quotas, and there is no cross-shard preemption or reclaim.

#### Preempt versus reclaim — get this right

This is the crispest statement of a distinction that Kueue expresses more diffusely (through
`withinClusterQueue` versus `reclaimWithinCohort`) and that most candidates blur. From KAI's
own deep-dive doc, the rules are:

**Preemption** (intra-queue) requires *all* of:

1. preemptor and victim are in the **same queue**;
2. the victim is **preemptible** — by default, priority value **< 100**;
3. the victim has **strictly lower** priority than the preemptor;
4. the victim has at least one actively running pod.

**Reclaim** (inter-queue) requires *all* of:

1. reclaimer and victim are in **different queues**;
2. the victim is preemptible;
3. the reclaiming queue is **below** its fair share or deserved quota;
4. the victim's queue is **above** its fair share or deserved quota.

And the guarantee that falls out — the one worth memorising — is: **in-quota resources are
never reclaimable.** KAI enforces this with two named strategies, `MaintainFairShare` (the
victim's queue must be above its allocatable fair share) and `GuaranteeDeservedQuota` (the
reclaimer must be under its deserved quota *and* the victim's queue over its own). A queue
running entirely inside its own quota cannot have work taken from it, no matter how loudly
another queue is starving.

The canonical FAQ question — *"why can't my inference workload in Queue-A preempt training in
Queue-B?"* — has a three-part answer you should be able to give cold: preemption is
intra-queue so it does not apply; reclaim is the inter-queue mechanism but requires Queue-B
to be over quota; and if Queue-B is inside its quota it is protected by design, so Queue-A
waits.

#### Queue: a tree with three numbers per resource

```yaml
apiVersion: scheduling.run.ai/v2
kind: Queue
metadata:
  name: org-research                    # parent queues are organisational only —
spec:                                   # only LEAF queues can hold workloads
  displayName: "Research org"
  priority: 100
---
apiVersion: scheduling.run.ai/v2
kind: Queue
metadata:
  name: team-llm
spec:
  parentQueue: org-research
  priority: 100                         # default 100; higher = surplus first, reclaimed last
  preemptMinRuntime: 10m                # min runtime before a job here may be preempted
  reclaimMinRuntime: 30m                # min runtime before it may be reclaimed
  resources:
    gpu:
      quota: 16                         # GUARANTEED. -1 = unlimited, 0/unset = none
      overQuotaWeight: 2                # weight for splitting surplus at this priority
      limit: 24                         # hard ceiling. -1 = no limit, 0/unset = no surplus
    cpu:
      quota: 2000                       # millicores
      overQuotaWeight: 1
      limit: 4000
    memory:
      quota: 4096                       # megabytes (10^6 bytes)
      overQuotaWeight: 1
      limit: 8192
```

Units are worth pinning because they are unusual: **CPU in millicores, memory in MB (10⁶
bytes, not MiB), GPU in device units where 1 = one full physical GPU and fractions are legal.**
`Queue.status` reports `allocated`, `allocatedNonPreemptible` and `requested` — which is
exactly the join a showback tool wants.

Surplus is distributed in **two phases**, and the phase split is the thing people get wrong:

```
  KAI SURPLUS DISTRIBUTION  —  10 GPUs, three queues
  ═══════════════════════════════════════════════════════════════════════════

  PHASE 1 — guaranteed quota.  QUEUE PRIORITY IS IRRELEVANT HERE.
            every queue receives min(quota, requested)

     vision  quota 3, wants 6   ──▶ gets 3      ┌───┬───┬───┐
     llm     quota 3, wants 6   ──▶ gets 3      ├───┼───┼───┤   7 GPUs committed
     batch   quota 1, wants 8   ──▶ gets 1      └───┘           3 GPUs surplus
                                                 ▲
                             a queue at or below quota is now UNRECLAIMABLE

  PHASE 2 — surplus, by PRIORITY BUCKET then by overQuotaWeight

     priority 100 bucket:  vision (w=2), llm (w=1)   ← all 3 surplus GPUs go here
     priority  50 bucket:  batch  (w=100)            ← gets NOTHING; priority is
                                                        strictly hierarchical, so
                                                        weight 100 loses to a
                                                        higher-priority weight 1

     split 3 GPUs by weight 2:1  ──▶  vision +2,  llm +1

  FINAL:  vision 5   llm 4   batch 1
  ═══════════════════════════════════════════════════════════════════════════
   Non-preemptible workloads (priority ≥ 100) can ONLY consume in-quota
   resources — they never take surplus, and they never trigger reclaim.
```

#### Priority classes, and the number 100

KAI ships four PriorityClasses and treats **100 as the preemptibility threshold**:

| Class | Value | Preemptible? | Default for |
|---|---|---|---|
| `train` | 50 | yes | everything not otherwise matched |
| `build-preemptible` | 75 | yes | — |
| `build` | 100 | **no** | Kubeflow Notebook |
| `inference` | 125 | **no** | Deployment, Knative Service |

Any PriorityClass in the cluster works; the rule is purely numeric — **value ≥ 100 means
non-preemptible, and non-preemptible workloads may only use in-quota resources.** That single
rule is what stops an inference service from taking borrowed capacity it could be evicted
from a minute later, and it is a cleaner statement of the same instinct behind Kueue's
"don't let prod borrow" advice.

#### Fractional GPU, and exactly what isolation you get

A pod asks for a share of a device by annotation, and names its queue by **label** (not
annotation — a detail that bites):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-sharing
  labels:
    kai.scheduler/queue: default-queue      # LABEL, not annotation
  annotations:
    gpu-fraction: "0.5"                     # half of one device's memory
    # or, instead:  gpu-memory: "2000"      # absolute MiB
    # optional:     gpu-fraction-container-name: "gpu-workload"
spec:
  schedulerName: kai-scheduler
  containers:
    - name: gpu-workload
      image: nvidia/cuda:13.0.2-base-ubi8
      command: ["nvidia-smi"]
      args: ["-L"]
```

Mechanically: the pod requests **no** `nvidia.com/gpu` at all — the annotation drives
allocation. To hold the physical device against the device plugin, KAI's binder runs a
**reservation pod** in the `kai-scheduler` reservation namespace, on the `nvidia` RuntimeClass
so it has NVML access. Sharing is off by default and enabled at install with
`--set "global.gpuSharing=true"`.

And now the part the docs say plainly and every summary omits:

> **By default, KAI Scheduler does not enforce memory allocation limits or perform memory
> isolation between processes.** Pods sharing a GPU may consume more memory than requested,
> which can affect other workloads on that device. Isolation requires integrating HAMi (or
> MPS).

So the honest comparison of the three sharing mechanisms is:

| Mechanism | Granularity | Memory isolation | Compute isolation | Reconfiguration cost |
|---|---|---|---|---|
| **MIG** (hardware, module 04) | fixed profile menu (e.g. 1g.10gb … 7g.80gb) | **hard** — separate memory slices, separate L2/paths | **hard** — dedicated SM partitions | high: drain the whole GPU to change geometry |
| **KAI `gpu-fraction`** (scheduler) | any float | **none by default**; HAMi or MPS required | none — time-shared | zero: it is a scheduling decision |
| **Volcano `deviceshare` vGPU** | memory / cores | via HAMi core library | core-limit enforcement via HAMi | zero |

The FinOps read: fractional sharing is the highest-leverage lever on notebook/dev waste, and
it is also the one that will generate your first noisy-neighbour incident if you deploy it
without HAMi/MPS and tell tenants it is "isolated". Say both halves.

#### PodGrouper: gang without authoring PodGroups

KAI's `podgrouper` component watches workload objects — `Job`, `PyTorchJob`, `Deployment`,
Knative `Service`, Grove `PodCliqueSet`, and so on — and *infers* the PodGroup, including
`minMember`. Volcano expects the PodGroup to be authored (by you, by `vcjob`, or by an
operator's webhook). Same primitive; opposite ergonomics. In a fleet where researchers submit
raw `Job`s, the inference path removes an entire class of "I forgot the annotation and my job
deadlocked" tickets.

### 4. The comparison table that actually decides the choice

Rows are the axes that change a decision; cells are verdicts, not checkmarks.

| Axis | **Kueue** | **Volcano** | **NVIDIA KAI** |
|---|---|---|---|
| Lineage | Kubernetes SIG (`kubernetes-sigs/kueue`), cloud-native | kube-batch → CNCF, HPC/MPI/Spark lineage | Run:ai commercial engine, open-sourced by NVIDIA |
| Architecture | Controller + webhook **above** `kube-scheduler`; never binds a pod | **Replacement** scheduler binary; session → actions → plugins | **Replacement** scheduler binary; session → actions → plugins |
| Core objects | `Workload`, `ClusterQueue`, `LocalQueue`, `ResourceFlavor`, `Cohort` | `PodGroup`, `Queue`, `HyperNode` | `PodGroup` (auto-inferred), `Queue` (tree), `Topology`, `SchedulingShard` |
| Gang | **Delegated** — Kueue admits atomically but placement/gang is `kube-scheduler` + coscheduling, or a gang scheduler underneath | **Native**: transactional `Statement` committed only if `JobPipelined` ≥ `minMember`; plus `subGroupPolicy` for heterogeneous sub-gangs | **Native**, inferred by PodGrouper; plus `StaleGangEviction` to kill under-minMember zombies |
| Quota model | `nominalQuota` per flavor, `borrowingLimit`, `lendingLimit`, cohort tree | `deserved` / `guarantee` / `capability` per resource type, `weight`, `parent`, queue `priority` | `quota` / `limit` / `overQuotaWeight` per resource, queue tree, queue `priority` |
| Borrowing | Yes — cohort borrowing with explicit limits; the deepest model of the three | Yes — anything above `guarantee` up to `capability` is lendable; reclaim takes it back | Yes — "over-quota", distributed by priority bucket then weight |
| Fairness | Cohort `fairSharing` (DRS-based) + `AdmissionFairSharing` (usage-decayed, beta from v0.15) | **Native DRF** plugin, plus hierarchical DRF (`hdrf`) and `proportion`/`capacity` | Hierarchical fair share over the queue tree, with `MaintainFairShare` / `GuaranteeDeservedQuota` reclaim strategies |
| Preemption granularity | Whole `Workload` (evict + requeue). Never partial. | Pod-granularity in `preempt`/`reclaim`; **bundle/job granularity** in `gangpreempt`/`gangreclaim` | Whole workload; separate `preempt` (intra-queue) and `reclaim` (inter-queue) verbs |
| Topology awareness | **TAS** — `Topology` CRD of node-label levels, `required`/`preferred`/`unconstrained` PodSet annotations (06.6) | **HyperNode** CRD — an explicit tier tree of switches/domains, `networkTopology.mode: hard\|soft` + `highestTierAllowed` on the PodGroup, `network-topology-aware` plugin | `Topology` CRD (`kai.scheduler/v1alpha1`) of node-label levels + `kai.scheduler/topology-required-placement` / `-preferred-placement` annotations; required and preferred can be combined |
| Defragmentation of *running* work | **No.** Admits or waits. | Only via the `shuffle` action + `rescheduling` plugin (a rebalancing hook, not fit-driven) | **Yes** — `consolidate` is a first-class action that runs before any disruptive action, and evicts only when a new home already exists |
| Fractional GPU | **No concept.** | Yes — `deviceshare`: `volcano.sh/gpu-memory`, `gpu-number`, or HAMi vGPU (`vgpu-number/-memory/-cores`) | Yes — `gpu-fraction` / `gpu-memory` annotations; **no isolation by default**, needs HAMi or MPS |
| Bin-packing control | Indirect (via `kube-scheduler` scoring; TAS placement algorithms for topology) | `binpack` plugin with per-resource weights, and hyper-node-level binpack in the NTA plugin | Domain selection is bin-packing by construction (prefers the *fullest* domain that still fits) |
| Multi-cluster | **MultiKueue** (beta since v0.9) | No (single cluster; `extendClusters` is a stub) | Shards partition one cluster, not many clusters |
| Operational weight | **Lightest.** One controller + webhook; the default scheduler still does the binding, so `kubectl describe pod` reasoning is unchanged. | **Heaviest.** Separate scheduler, webhook manager, controller manager, its own CRDs; you now debug two schedulers. | Medium-heavy: scheduler, binder, podgrouper, queue controller, operator — plus reservation pods on your GPU nodes. |
| Maturity / churn risk | Highest. v1beta2 APIs, versioned feature gates with documented graduation, a release cadence you can plan against. | High. CNCF project, long history, stable v1beta1 API — but the *action/plugin config* surface changes between minors. | Lowest. API group still `scheduling.run.ai`; expect CRD-group and field churn. Pin a tag and read the changelog before every upgrade. |
| Showback ergonomics | **Best.** `Workload` objects carry admitted resources, queue, priority and timestamps in one place — a natural billing record. | Good: `Queue.status.allocated` + PodGroup phases. | Good: `Queue.status.allocated` / `allocatedNonPreemptible` / `requested`, and fractional GPU is already in the accounting. |
| Best when | Multi-tenant quota, borrowing, chargeback; you want the standard scheduler to stay the standard scheduler | HPC/MPI/Spark/Ray lineage; heterogeneous per-resource entitlements; you need topology as an explicit tier tree; you want gang-granularity eviction | GPU sharing for notebooks/inference; fragmentation is your dominant loss; you want defrag as policy rather than a runbook |

### 5. Composition: what actually layers, and what does not

The interview trap is treating this as a bake-off with one winner. The strong answer is
layering, and the layering rule is simple: **you can stack two systems only if they own
different decisions.**

```
  WHAT COMPOSES
  ═════════════════════════════════════════════════════════════════════════════

   ┌──────────────────────────────────────────────────────────────┐
   │  KUEUE                                                       │
   │  owns: WHEN a workload is admitted, against WHOSE quota,     │
   │        who may borrow from whom, and the billing record      │
   └────────────────────────────┬─────────────────────────────────┘
                                │ unsuspends the Job
                                ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  kube-scheduler + coscheduling   OR   volcano-scheduler      │
   │  owns: WHERE the pods land, and all-or-nothing binding       │
   └──────────────────────────────────────────────────────────────┘

   ✔ works, because "when + whose quota" and "where + atomically" are disjoint.
     You keep Kueue's Workload accounting (your showback source) and get a
     scheduler that will not bind 6 of 8 pods.


  WHAT DOES NOT COMPOSE
  ═════════════════════════════════════════════════════════════════════════════

   ┌──────────────────────────────────────────────────────────────┐
   │  KUEUE           quota, admission, borrowing, preemption     │
   └────────────────────────────┬─────────────────────────────────┘
                                ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  KAI             quota, admission, borrowing, preemption,    │
   │                  reclaim, consolidation, placement           │
   └──────────────────────────────────────────────────────────────┘
                    ▲▲▲ both layers own quota ▲▲▲

   ✘ two independent quota engines, two independent preemption engines, two
     independent notions of "admitted". They will fight: Kueue admits against
     ClusterQueue accounting, KAI reclaims against Queue accounting, and each
     sees the other's decision as an unexplained external event. KAI (like
     Run:ai before it) is a full stack — it SUBSTITUTES for Kueue.
  ═════════════════════════════════════════════════════════════════════════════
```

The same reasoning says a Volcano-under-Kueue stack must be configured so that **only one
layer preempts**. If Kueue is reclaiming borrowed quota by evicting Workloads *and* Volcano's
`gangreclaim` action is evicting bundles by queue accounting, you have two controllers
racing on the same pods with different models of who deserves what. In practice: let Kueue
own quota and preemption (leave Volcano's action list at `enqueue, allocate, backfill`), and
use Volcano purely for gang + topology + binpack. Or invert it — but pick one.

### 6. A decision procedure, not a preference

Run these in order; the first one that fires decides.

1. **Does any tenant need sub-GPU allocation?** If yes and MIG's fixed geometry does not fit
   the demand shape → you need `deviceshare` (Volcano) or `gpu-fraction` (KAI). Kueue is out
   for that pool. *Budget for HAMi or MPS in the same breath, or you are shipping
   noisy-neighbour risk.*
2. **Is fragmentation your dominant loss** (compute it — lesson 06.7) **and can your workloads
   tolerate being moved?** If yes → KAI's `consolidate` action is the only one of the three
   that attacks the stock directly.
3. **Do you need heterogeneous, per-resource-type entitlements** ("A100 1:3, V100 1:1")? →
   Volcano's `capacity` plugin with per-resource `deserved`. Kueue expresses this with
   separate ResourceFlavors, which works but multiplies the object count.
4. **Do you need topology expressed as a real switch tree** rather than a flat label
   hierarchy, or per-sub-group topology within one job? → Volcano's `HyperNode` +
   `subGroupPolicy`. See 06.6.
5. **Is chargeback / showback the primary deliverable?** → Kueue. `Workload` objects are the
   cleanest billing record of the three, and this module's deliverable depends on it.
6. **Otherwise** → Kueue, plus the coscheduling plugin for gang. It has the lowest operational
   weight, the most stable API, and it leaves `kubectl describe pod` meaning what your team
   already thinks it means.

Compressed to one breath for an interview: **cloud-native multi-tenant quota and chargeback →
Kueue; HPC/MPI lineage, per-resource entitlements, explicit switch-tree topology, or
gang-granularity eviction → Volcano; fractional GPUs, fragmentation-dominated loss, or
consolidation-as-policy → KAI. And they compose only when one owns quota and the other owns
placement.**

## Perspectives

**Developer / researcher.** The scheduler you pick decides whose mental model the platform
rewards. An HPC-background researcher gets Volcano for free: explicit `PodGroup`, explicit
`Queue`, DRF, `vcjob` — it reads like Slurm with YAML. A cloud-native engineer gets Kueue for
free: a Job that is suspended until quota exists, and nothing else to learn. KAI's PodGrouper
splits the difference by inferring the gang, which is the friendliest default but also the one
where "why did my Deployment get gang semantics?" becomes a support question. This is a
people/org consideration that platform teams systematically underweight.

**Operator.** Operational weight is asymmetric and it is the axis most likely to be wrong in
a feature-driven decision. Kueue leaves the scheduling path alone — when a pod is Pending you
still read `kube-scheduler` events. Volcano and KAI *replace* the scheduler for the pods that
opt in, so you now debug two schedulers, and a pod with the wrong `schedulerName` sits Pending
forever with no events from anyone. Budget for: a second set of scheduler metrics and
dashboards, a second upgrade cadence, and a runbook entry for "which scheduler owns this
pod?" Add reservation pods on every GPU node if you enable KAI's sharing.

**Hardware.** The three fractional-GPU mechanisms sit at different layers of the stack and
give different guarantees, and conflating them produces incidents. MIG partitions the silicon:
separate memory slices and dedicated SM partitions, hard isolation, fixed geometry, and a
drain to reconfigure. KAI's `gpu-fraction` is a *scheduler* bookkeeping decision — the device
is time-shared by the driver, and without HAMi or MPS there is no enforcement at all, so an
over-allocating tenant simply OOMs its neighbour. Volcano's HAMi-backed vGPU path sits in
between: memory and core limits enforced by an interposed library. Choose by the isolation
your tenancy needs, not by the granularity number in the annotation.

**Economics.** Three of this lesson's mechanisms have direct, separable dollar values, and
being able to attach a number to each is the differentiator. *Fractional sharing* reclaims
the whole-GPU-per-notebook waste — count your idle-but-allocated devices from module 05's
`SM_ACTIVE` data, multiply by your rate, and that is the ceiling of the prize.
*Consolidation* reclaims stranded fragments — lesson 06.7 gives you the exact
`sum(floor(free/k))` arithmetic. *Gang-granularity eviction* prevents the negative-value
preemption drawn in §2.4, where freeing 2 GPUs strands 6. The first is the biggest number,
the third is the easiest to defend, and the second is the one that most impresses a
capacity-planning interviewer because almost nobody quantifies it.

## Real-world use cases

- **Volcano — the shipped default config is not the config people describe.** Read
  `installer/helm/chart/volcano/config/volcano-scheduler.conf` in the repo:
  `actions: "enqueue, allocate, backfill"`. Neither `preempt` nor `reclaim` is enabled out of
  the box; the CI config (`volcano-scheduler-ci.conf`) adds them. What it shows: "we run
  Volcano so we have preemption" is a claim that has to be checked against a ConfigMap, not
  assumed. This is a first-party artefact, not a secondary account.
- **Volcano — the gang-aware eviction design.** `docs/design/gang-aware-eviction-design.md` and
  `docs/design/evictablefn-evolution-for-gang-eviction.md`, implemented as
  `pkg/scheduler/actions/gangpreempt` + `gangreclaim` + `pkg/scheduler/actions/utils/bundle.go`.
  What it shows: the surplus arithmetic (`jobSurplus`, `roleSurplus`, `groupSurplus`) that
  decides which tasks can be taken without breaking a gang, and the `BundleSafe`-before-
  `BundleWhole` ordering. This is the concrete mechanism behind "gang-granularity preemption"
  — a phrase that is otherwise just a slogan.
- **KAI — the documented absence of isolation in GPU sharing.** `docs/gpu-sharing/README.md`
  states that KAI "does not enforce memory allocation limits or perform memory isolation
  between processes" by default and points at HAMi. What it shows: the highest-value FinOps
  feature in this lesson ships with a caveat that changes its multi-tenancy story. A candidate
  who pitches fractional GPUs without mentioning it is describing a demo, not a platform.
- **KAI — the preempt/reclaim FAQ.** `docs/scheduling-deep-dive/README.md` walks the exact
  question *"why can't my inference workload in Queue-A preempt training in Queue-B?"* and
  answers it with the three-part rule in §3.2. What it shows: the intra/inter-queue split is
  load-bearing enough that the project wrote a FAQ entry for it — which is a strong hint that
  it is the thing operators actually get wrong.
- **Volcano at production scale — Altoros, "Scheduling 300,000 Kubernetes Pods in Production
  Daily"** — https://www.altoros.com/blog/volcano-scheduling-300000-pods-in-production-daily/
  (**not fetched: this session's egress proxy returns 403 for this host**; canonical URL,
  search-confirmed title and topic). What it reports: Volcano running on the order of 300,000
  pods/day in production, cited alongside adoption by dozens of enterprises. Treat the number
  as the blog's claim rather than as a verified measurement.
- **NVIDIA — open-sourcing the Run:ai scheduler as KAI** —
  https://developer.nvidia.com/blog/nvidia-open-sources-runai-scheduler-to-foster-community-collaboration/
  (**not fetched: proxy 403**; canonical URL, search-confirmed). What it shows: the origin
  story for why an API group called `scheduling.run.ai` is shipping under an `NVIDIA/` GitHub
  org — and therefore why field names in that group are the ones most likely to move.

## Worked example

**Profile.** A neocloud tenant platform: 40+ clusters, four tenant classes, one platform team.

- *Class A — research training:* multi-node PyTorch/MPI, 8–64-GPU gangs, bursty. Needs hard
  all-or-nothing and fair sharing across three teams whose jobs bind on different resources.
- *Class B — interactive/dev:* ~200 notebooks and small inference pods, each currently pinning
  a whole H100 at under 10% `SM_ACTIVE`.
- *Class C — cost governance:* finance wants per-team showback and enforceable quota with
  controlled borrowing.
- *Class D — disaggregated LLM serving:* separate prefill and decode pod cliques, with strict
  co-location and startup ordering, and per-clique autoscaling.

**Step 1 — put a number on Class B, because it decides everything.** 200 notebooks × 1 H100
each, at a mid-band specialized-neocloud rate of **~$3/GPU-hour** (a *dated snapshot* of one
market segment — see 06.8 for why the segment must be named):

```
  current burn      = 200 GPU × $3/GPU-hr × 24 × 365      ≈ $5.26 M/yr
  measured use      ≈ 8% mean SM_ACTIVE  (module 05 data)
  at gpu-fraction 0.25, four notebooks share one device:
      devices needed = 200 / 4                             = 50 GPU
      new burn       = 50 × $3 × 24 × 365                  ≈ $1.31 M/yr
  ─────────────────────────────────────────────────────────────────────
  recoverable        ≈ $3.94 M/yr,  minus the risk cost of soft isolation
```

Round it down hard for a real proposal — packing four notebooks per device assumes their
memory footprints actually fit, and without HAMi one greedy tenant can OOM three others. But
even at half that, no amount of Kueue quota tuning gets near it, because **Kueue has no
concept of half a GPU.** Class B therefore cannot be served by Kueue at any configuration.

**Step 2 — Class C is unambiguous.** Kueue. `Workload` objects are the cleanest showback
record of the three: admitted resources, queue, priority and timestamps in one object, which
is exactly what this module's deliverable joins to a rate. Cohort borrowing gives finance the
"guaranteed floor plus burst into idle" story they want.

**Step 3 — Class A needs gang, and Class C already mandates Kueue.** So compose: Kueue owns
admission and quota; the gang guarantee comes from the coscheduling plugin, or from Volcano if
the teams' jobs also need per-resource entitlements. Check whether they do:

```
  team-vision : mostly GPU-bound      dominant resource = nvidia.com/gpu
  team-nlp    : GPU-bound             dominant resource = nvidia.com/gpu
  team-data   : CPU+memory preprocessing, few GPUs
                on a 64-GPU / 1024-CPU cluster holding 4 GPU + 512 CPU:
                    share(GPU) = 4/64   = 0.0625
                    share(CPU) = 512/1024 = 0.500  ← dominant
  → team-data's dominant resource is CPU. Under equal GPU quota it would look
    "under-served" on the GPU axis while consuming half the cluster's CPU.
    DRF is the correct fairness model here; a flat GPU quota is not.
```

That is a real argument for Volcano-under-Kueue rather than a preference — and note what you
must then configure: **Volcano's action list stays `enqueue, allocate, backfill`** so that only
Kueue preempts. Volcano contributes gang, DRF ordering, and `binpack`; Kueue contributes
quota, borrowing, reclaim and the billing record.

**Step 4 — Class D is structurally different from Class A.** It is not one homogeneous gang;
it is several interdependent cliques with startup ordering and independent scaling, and with
KV-cache transfer between prefill and decode that is bandwidth-sensitive in the same way an
all-reduce is. Neither a flat Kueue PodSet nor a single `minMember` expresses "two different
pod shapes, one must be ready before the other, both should be topology-close, each scales
independently." Volcano's `subGroupPolicy` is the closest primitive in this lesson — sub-groups
with their own size, their own minimum count, and their own `networkTopology` — and NVIDIA's
Grove/`PodCliqueSet` API layered on KAI is the purpose-built one. Either way, **Class D does
not belong in the Class A queue**, and the reason is expressive, not political.

**Defensible outcome.**

> *Kueue is the fleet-wide quota, borrowing and showback plane, with a gang-capable scheduler
> beneath it for training. A separate KAI-managed node pool serves interactive and inference
> tenants that need fractional GPUs and consolidation; that pool is where disaggregated
> serving lands too. Volcano is used where per-resource-type entitlements or explicit
> switch-tree topology are required. Exactly one layer preempts in each pool.*

You did not pick a scheduler. You matched each tenancy shape to the primitive that expresses
it, put ~$2M/yr (risk-discounted) on the fractional-sharing pool, and can state the
correctness argument for why Class D cannot share Class A's queue.

## Practice

Reading-and-decision, feeding the deliverable's scheduler-selection artifact — see
[Kueue setup + per-queue showback](../practice/kueue-showback/README.md).

1. **Read three manifests against their source of truth.** Write out, by hand, (a) a Volcano
   `PodGroup` + `Queue`, (b) a KAI `Queue` + fractional-GPU pod, (c) a Kueue `ClusterQueue` +
   `LocalQueue`. For each, annotate three things in the margin: *where is gang expressed?
   where is fairness expressed? is the GPU whole or divisible?* Then check every field name
   against the API types cited in References — not against this lesson, and not against a
   blog. The point of the exercise is the habit of checking.
2. **Read one config and one action.** Open Volcano's shipped
   `installer/helm/chart/volcano/config/volcano-scheduler.conf` and write one sentence on what
   your cluster would *not* do with it. Then read
   `pkg/scheduler/actions/utils/bundle.go::SortBundlesForPreempt` and write down its four
   comparison keys in order.
3. **Write the decision matrix** and commit it as `scheduler-decision-matrix.md`. Rows, at
   minimum:
   - Tenancy model (single-team batch / multi-tenant quota / hierarchical org→team→project)
   - HPC-MPI vs cloud-native lineage
   - Fractional-GPU need, **and the isolation level required** (none / soft / hard)
   - Heterogeneous per-resource entitlements (yes/no)
   - Heterogeneous or ordered gang need (yes/no)
   - Topology model needed (flat label hierarchy / explicit tier tree / per-sub-group)
   - Preemption granularity required (workload / pod / gang-bundle)
   - Defragmentation of running work required (yes/no)
   - Showback as a primary deliverable (yes/no)
   - Operational weight and API-churn tolerance

   Columns: Kueue / Volcano / KAI. Each cell is a one-line verdict, not a checkmark.
4. **State a composition and a non-composition.** One concrete "Kueue + X" stack for your
   fleet, with an explicit sentence naming **which layer preempts**; and one sentence on why
   KAI-under-Kueue is not a stack.

**Acceptance:** a committed `scheduler-decision-matrix.md` that (a) fills every cell with a
verdict, (b) names at least one workload where Volcano beats Kueue *and gives the mechanism*,
(c) states the fractional-GPU FinOps case with its arithmetic **and its isolation caveat**,
(d) names a workload shape that needs sub-group or clique semantics, and (e) describes one
valid composition with the single-preemptor rule stated. If a reader can pick a scheduler for
a new tenant from your matrix without asking you a question, it passes.

## Common pitfalls

1. **"Volcano is whole-device only."** *Symptom:* you rule Volcano out for a sharing use case
   it handles. *Mechanism:* the `deviceshare` plugin supports `volcano.sh/gpu-memory`,
   `volcano.sh/gpu-number`, and a HAMi-derived vGPU path
   (`volcano.sh/vgpu-number/-memory/-memory-percentage/-cores`), gated by
   `deviceshare.GPUSharingEnable`, `deviceshare.GPUNumberEnable` and `deviceshare.VGPUEnable`.
   The plugin `klog.Fatal`s if you enable incompatible pairs, so a cluster picks one model.
2. **Assuming a stock Volcano install preempts.** *Symptom:* a high-priority job sits Pending
   while low-priority work runs, and you conclude priority is broken. *Mechanism:* the shipped
   `volcano-scheduler.conf` sets `actions: "enqueue, allocate, backfill"` — no `preempt`, no
   `reclaim`, no `gangpreempt`, no `gangreclaim`. Preemption is a configuration you write.
3. **Treating KAI's `gpu-fraction` as isolation.** *Symptom:* one tenant's OOM takes down three
   others on the same device, and the incident review blames the scheduler. *Mechanism:* KAI's
   fraction is a scheduler-side accounting decision; the docs state plainly that no memory
   limit or isolation is enforced by default, and point at HAMi (or MPS) to get it.
4. **Confusing preempt with reclaim.** *Symptom:* you configure priorities carefully and are
   baffled that a high-priority workload in queue A never touches queue B's low-priority work.
   *Mechanism:* in KAI (and in Volcano's action split) preemption is **intra-queue** and
   priority-driven; the only inter-queue mechanism is **reclaim**, which is quota-driven and
   cannot touch a queue that is inside its own quota. Priority does not cross a queue boundary.
5. **Assuming pod-granularity preemption is "good enough" for gangs.** *Symptom:* preemption
   is enabled, utilisation gets *worse*. *Mechanism:* evicting k pods from an n-pod gang frees
   k GPUs and strands n−k, because the survivors cannot reach `minMember`. Use
   `gangpreempt`/`gangreclaim`, whose victims are `BundleSafe` (surplus replicas) before
   `BundleWhole` (the entire job) — and verify your version actually registers those actions.
6. **Stacking two quota engines.** *Symptom:* Kueue admits a Workload, KAI reclaims its pods
   minutes later, Kueue does not understand why its admitted Workload lost pods, and it
   re-admits. *Mechanism:* KAI is a full quota+fairness+placement stack; it substitutes for
   Kueue rather than layering under it. When composing, exactly one layer owns quota and
   exactly one layer preempts.
7. **Copying `scheduling.run.ai/v2` field names from a blog post.** *Symptom:* CRD validation
   errors on install. *Mechanism:* KAI is the youngest of the three and its API group still
   carries its commercial origin. Read `pkg/apis/scheduling/v2/queue_types.go` and
   `resources.go` at the exact tag you deploy — the fields are `quota`, `overQuotaWeight`,
   `limit`, with CPU in millicores, memory in **MB (10⁶ bytes)**, GPU in device units.
8. **Putting the queue name in the wrong metadata field for KAI.** *Symptom:* the pod is
   ignored by the scheduler or lands in `default-queue`. *Mechanism:* `kai.scheduler/queue` is
   a **label**; `gpu-fraction` and `gpu-memory` are **annotations**. Volcano's
   `scheduling.k8s.io/group-name` and `scheduling.volcano.sh/queue-name` are annotations.
   They are not interchangeable.

## Self-check

- **Name one workload where Volcano beats Kueue, and give the mechanism, not the feature
  name.** *Answer:* a multi-node MPI all-reduce job sharing a cluster with a CPU-heavy
  preprocessing tenant. Two mechanisms do the work. First, gang: Volcano's `allocate` action
  accumulates placements in a `Statement` and calls `Commit()` only when `JobPipelined(job)`
  is true — i.e. allocated-plus-pipelined tasks have reached `minMember` — otherwise it calls
  `Discard()` and binds nothing, so a partial gang is structurally impossible. Kueue admits
  atomically but delegates the actual binding, so it needs coscheduling or a gang scheduler
  underneath to get the same property. Second, fairness: Volcano's `drf` plugin computes each
  tenant's `max_r(allocated_r / total_r)` and serves the lowest dominant share first, so a
  tenant consuming half the cluster's CPU is correctly deprioritised even though it holds few
  GPUs. Kueue's quota model would need a separate flavor/quota per resource axis to
  approximate this.

- **What does KAI's consolidation do that Kueue cannot, and where does it sit in the cycle?**
  *Answer:* `consolidate` is action 2 of 5, running after `allocate` and before every
  disruptive action. When a pending workload cannot fit because free capacity is scattered, it
  relocates already-running pods to compact that capacity — and it evicts a pod only if the
  scheduler has already found it a new home, which is what makes it "temporary eviction" rather
  than preemption. Kueue's only eviction verb is "evict this Workload and requeue it"; it has
  no mechanism for moving running work to change the shape of free capacity, so it can only
  admit or wait. On 8-GPU nodes this is the difference between two stranded 3-GPU holes and
  one runnable 6-GPU slot.

- **Can Kueue and a gang scheduler compose? State the rule that makes it safe.** *Answer:*
  yes, because they own disjoint decisions: Kueue owns *when* a workload is admitted and
  *against whose quota*, and the underlying scheduler owns *where* the pods land and
  all-or-nothing binding. The safety rule is **exactly one layer may preempt**. If Kueue is
  evicting Workloads to reclaim borrowed quota while Volcano's `gangreclaim` is evicting
  bundles by its own queue accounting, two controllers race on the same pods with different
  models of entitlement. In practice: leave Volcano at `enqueue, allocate, backfill`. You would
  not stack Kueue on KAI at all, because KAI is itself a complete quota+fairness+placement
  stack and both layers would own quota.

- **What does gang-granularity eviction fix that pod-granularity eviction cannot, and how does
  Volcano implement it?** *Answer:* pod-granularity eviction can free k GPUs from an n-pod
  gang while stranding the other n−k, because the survivors cannot reach `minMember` and make
  zero progress — a net-negative capacity change. The `gangpreempt`/`gangreclaim` actions group
  candidate victims into **bundles** per job: `BundleSafe` holds tasks covered by a positive
  surplus in every dimension (`jobSurplus = ReadyTaskNum − MinAvailable`, `roleSurplus[r]`,
  `groupSurplus[g]`), so removing them leaves the job ready; `BundleWhole` is the entire job,
  which dies cleanly and requeues. `SortBundlesForPreempt` orders by bundle type first (safe
  before whole), then inverse job order (least important victim first), then bundle ROI against
  the preemptor's need, then a stable tie-break. The preemptor is placed as a whole gang inside
  one candidate domain, and the statement is committed only if that plan validates.

- **A tenant says "we need fractional GPUs." What are the three follow-up questions?**
  *Answer:* (1) *What isolation do you need?* MIG gives hard memory and SM isolation at fixed
  profile geometry and costs a full GPU drain to reconfigure; KAI's `gpu-fraction` gives none
  by default; Volcano's HAMi-backed vGPU enforces memory and core limits via an interposed
  library. (2) *Is the demand shape stable?* MIG geometry has to match a stable profile mix or
  you pay drain costs continually (06.7); a scheduler-side fraction is free to change. (3)
  *Whose neighbours are these?* Soft isolation between two teams' notebooks is a risk you can
  accept; soft isolation between a customer's inference and another customer's training is not.
  Only after those three does the choice of mechanism follow.

- **Why is "in-quota resources are never reclaimable" the load-bearing guarantee in KAI's queue
  model?** *Answer:* because it is what makes a quota a *promise* rather than a suggestion.
  Surplus distribution is priority-ordered and can starve a low-priority queue of over-quota
  capacity entirely — a priority-2/weight-1 queue takes surplus before a priority-1/weight-100
  queue. That is tolerable only because Phase 1 hands every queue `min(quota, requested)`
  *before* priority is consulted at all, and both reclaim strategies (`MaintainFairShare`,
  `GuaranteeDeservedQuota`) refuse to take from a queue at or below its deserved quota. A team
  can therefore reason about its floor without reasoning about anyone else's priority. Kueue
  expresses the same guarantee through `nominalQuota` plus `lendingLimit`; Volcano expresses a
  stronger version through `guarantee`, which is never lent out at all and can be node-locked.

- **Your stock Volcano install is not preempting. Walk the diagnosis.** *Answer:* first check
  the action list in the scheduler ConfigMap — the shipped default is
  `actions: "enqueue, allocate, backfill"`, so no preemption action is registered at all. If
  `preempt` is present but nothing happens, check that the preemptor and victim are in the
  **same queue** (cross-queue movement is `reclaim`, a different action), that
  `preemptorJob.Priority > victimJob.Priority`, and that the victim is marked preemptable
  (`volcano.sh/preemptable`). If `reclaim` is present but nothing happens, check the victim
  queue's `reclaimable` flag and whether it is actually above its `deserved` — usage inside
  `guarantee` is never reclaimable. Finally, check the tier layout: the `gang` and `drf`
  plugins both register `PreemptableFn` vetoes, and `enablePreemptable: false` in the shipped
  config disables `gang`'s and `drf`'s contribution to that decision.

## Connections & what's next

This lesson sits on top of lessons 3–4 (Kueue's quota, cohorts, borrowing, both fairness
layers) and reaches back to lesson 2's gang deadlock — Volcano's `Statement`
commit-or-discard is a second, native solution to the problem coscheduling solves as a
plugin, and KAI's `StaleGangEviction` attacks the same failure from the runtime side. The DRF
thread (Ghodsi et al., NSDI 2011) is the shared theoretical spine under Volcano's `drf`
plugin and Kueue's cohort fair sharing; you now have both implementations to compare against
one theory. Two forward hooks are already open: KAI's `consolidate` action and Volcano's
`binpack` weights are scheduler-level attacks on the fragmentation that **lesson 7** teaches
you to measure and price, and the preempt/reclaim split, the priority-100 preemptibility
threshold and the bundle-ordering function are the raw material for **lesson 8**'s
priority-tier design.

Next: lesson 6 asks the question all three schedulers share. Admitting the right *count* of
co-located GPUs is necessary but not sufficient if you do not also control *where* they land
relative to the interconnect. You have now seen three different models of that — Kueue's
`Topology` label levels, Volcano's `HyperNode` tier tree, and KAI's `Topology` plus
required/preferred annotations — and lesson 6 takes the first of them apart in full.

## References & further reading

**Primary sources — read at the tag you deploy**

- `volcano-sh/volcano` — `staging/src/volcano.sh/apis/pkg/apis/scheduling/v1beta1/types.go` — https://github.com/volcano-sh/volcano/blob/master/staging/src/volcano.sh/apis/pkg/apis/scheduling/v1beta1/types.go — the authoritative `PodGroup` and `Queue` types: `minMember`, `minTaskMember`, `minResources`, `networkTopology`, `subGroupPolicy`, and the `Queue` triple `guarantee`/`deserved`/`capability` plus `weight`, `parent`, `priority`, `reclaimable`, `dequeueStrategy`. Read at commit `5a15213` (2026-08-17), the tree this lesson was checked against. *Fetched by cloning the repo — `volcano.sh` documentation pages are blocked by this session's egress proxy (403 at the CONNECT tunnel).*
- `volcano-sh/volcano` — `staging/src/.../scheduling/v1beta1/labels.go` — the exact annotation keys: `scheduling.k8s.io/group-name`, `scheduling.volcano.sh/group-name`, `scheduling.volcano.sh/queue-name`, `volcano.sh/preemptable`, `volcano.sh/hierarchy`.
- `volcano-sh/volcano` — `installer/helm/chart/volcano/config/volcano-scheduler.conf` and `volcano-scheduler-ci.conf` — the shipped default action list and tiered plugin list, reproduced verbatim in §2. *Correction vs. the previous version of this lesson: the default install enables neither `preempt` nor `reclaim`.*
- `volcano-sh/volcano` — `pkg/scheduler/actions/` (`enqueue`, `allocate`, `backfill`, `preempt`, `reclaim`, `gangpreempt`, `gangreclaim`, `shuffle`) and `pkg/scheduler/actions/utils/bundle.go` — the action pipeline and the bundle model: `BundleSafe` vs `BundleWhole`, the three surplus computations, and `SortBundlesForPreempt`'s four comparison keys. *Correction: gang-granularity eviction is implemented as two additional actions, not as a change to the existing `preempt` action.*
- `volcano-sh/volcano` — `pkg/scheduler/plugins/drf/drf.go` (`calculateShare`), `pkg/scheduler/plugins/binpack/binpack.go` (`BinPackingScore`, `ResourceBinPackingScore`, and the all-ones defaults), `pkg/scheduler/plugins/deviceshare/` and `pkg/scheduler/api/devices/nvidia/` — the DRF formula, the bin-pack scoring function, and the three GPU-sharing paths with their resource names. *Correction: Volcano does support fractional GPUs; the previous version of this lesson said it did not.*
- `volcano-sh/volcano` — `docs/design/capacity-scheduling.md` and `docs/design/Network Topology Aware Scheduling.md` — why `capacity` exists separately from `proportion` (heterogeneous per-GPU-type entitlements), and the `HyperNode`/tier model that lesson 06.6 compares against Kueue's TAS.
- `NVIDIA/KAI-Scheduler` — `pkg/apis/scheduling/v2/queue_types.go` and `resources.go` — the `Queue` API: `parentQueue`, `priority`, `preemptMinRuntime`, `reclaimMinRuntime`, and per-resource `quota`/`overQuotaWeight`/`limit`. Group is `scheduling.run.ai/v2`. Read at commit `2914d32` (2026-08-17). *Fetched by cloning; NVIDIA developer-blog pages are proxy-blocked in this session.*
- `NVIDIA/KAI-Scheduler` — `docs/scheduling-deep-dive/README.md` — the five-action cycle (allocate → consolidate → reclaim → preempt → StaleGangEviction), the two-phase surplus distribution, the four preemption rules and four reclaim rules, the `MaintainFairShare` / `GuaranteeDeservedQuota` strategies, and the "in-quota resources are always protected" guarantee.
- `NVIDIA/KAI-Scheduler` — `docs/queues/README.md`, `docs/priority/README.md`, `docs/gpu-sharing/README.md` (+ `gpu-sharing.yaml`) — queue units (millicores / MB / GPU units), the four shipped PriorityClasses (`train` 50, `build-preemptible` 75, `build` 100, `inference` 125) and the ≥100 non-preemptible rule, and the `gpu-fraction` / `gpu-memory` annotations with the explicit statement that **no memory isolation is enforced by default**. *Correction: `kai.scheduler/queue` is a label, not an annotation.*
- `NVIDIA/KAI-Scheduler` — `docs/topology/README.md` — the `kai.scheduler/v1alpha1` `Topology` CRD and the `kai.scheduler/topology-required-placement` / `-preferred-placement` annotations, including the combined required+preferred form. Compared against Kueue TAS in lesson 06.6.
- `kubernetes-sigs/kueue` — `apis/kueue/v1beta2/clusterqueue_types.go`, `topology_types.go`, `resourceflavor_types.go` — the Kueue side of every comparison row: `reclaimWithinCohort`, `borrowWithinCohort`, `withinClusterQueue`, and the TAS annotations. Read at commit `e5084fe` (2026-08-17). *Fetched by cloning; `kueue.sigs.k8s.io` is proxy-blocked.*
- Ghodsi, A. et al., "Dominant Resource Fairness: Fair Allocation of Multiple Resource Types" (NSDI 2011) — https://www.usenix.org/conference/nsdi11/dominant-resource-fairness-fair-allocation-multiple-resource-types (**not fetched: proxy 403**; canonical USENIX URL) — the formal paper behind Volcano's `drf` plugin and the fairness intuition under Kueue's cohort fair sharing. The share formula and the strategy-proofness argument in §2.5 are reproduced from the algorithm as implemented in `drf.go`, which matches the paper's definition.

**Real-world engineering accounts**

- Altoros — "Scheduling 300,000 Kubernetes Pods in Production Daily" — https://www.altoros.com/blog/volcano-scheduling-300000-pods-in-production-daily/ (**not fetched: proxy 403**; canonical URL, search-confirmed) — what it reports: Volcano's production scale and enterprise adoption. Treat the pod-count figure as the blog's claim.
- NVIDIA — "Practical Tips for Preventing GPU Fragmentation for Volcano Scheduler" — https://developer.nvidia.com/blog/practical-tips-for-preventing-gpu-fragmentation-for-volcano-scheduler/ (**not fetched: proxy 403**; canonical URL) — the anti-fragmentation `binpack` configuration for multi-GPU nodes. The scoring function it tunes is reproduced from source in §2.6; lesson 06.7 quantifies the effect.
- NVIDIA — "NVIDIA Open-Sources Run:ai Scheduler" — https://developer.nvidia.com/blog/nvidia-open-sources-runai-scheduler-to-foster-community-collaboration/ (**not fetched: proxy 403**; canonical URL) — the origin story behind the `scheduling.run.ai` API group.

**Deeper dives**

- `volcano-sh/volcano` — `docs/design/hdrf.md`, `fairshare.md`, `hierarchical-queue-on-capacity-plugin.md` — hierarchical DRF and hierarchical queues, if you need to defend an org→team→project tree in Volcano rather than KAI.
- `NVIDIA/KAI-Scheduler` — `docs/operator/scheduling-shards.md` and `docs/developer/action-framework.md` — how shards partition a cluster and how to add or reorder actions per shard.
- The Grove project (`PodClique` / `PodCliqueScalingGroup` / `PodCliqueSet`), reachable from NVIDIA's disaggregated-inference material — for the full API behind the Class D case in the worked example, past this lesson's summary.
