---
lesson: "06.6"
title: "Topology-aware placement — packing a gang into one NVLink domain"
module: "06"
concept: "Topology-aware placement — packing a gang into one NVLink domain"
status: not-started
est_time: "9h"
prev: "05-alternatives-volcano-kai.md"
next: "07-fragmentation-effective-capacity.md"
artifacts: []
sources: 12
---

# 06.6 · Topology-aware placement — packing a gang into one NVLink domain

> **Concept.** A gang that spreads across NVLink domains or racks pays a 2–5× collective-communication tax; Kueue Topology-Aware Scheduling packs it into one domain via node labels so NCCL collectives never cross the slow link.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

Lesson 5 surveyed three schedulers — Kueue, Volcano, KAI — and every one of them answers the
question "does this gang get admitted, all at once?" None of them, on quota and admission logic
alone, answers "*where*, relative to the network fabric, does this gang land?" That is the gap
this lesson closes. You already have gang scheduling (lesson 2) guaranteeing atomicity and
Kueue's quota/cohort model (lessons 3–4) guaranteeing fairness. This lesson adds the third
axis, **placement**, and shows why a gang that is atomically admitted but topologically blind
can still be a slow, expensive mistake.

What it unlocks: the ability to derive — not assert — *why* two runs with identical GPU counts
differ by a factor of two to five in wall-clock cost, and to fix it with a scheduling constraint
instead of a purchase order.

Every Kueue mechanism claim below was read out of the `kubernetes-sigs/kueue` tree cloned during
this session at the post-v0.19.1 development head — `apis/kueue/v1beta2/topology_types.go`,
`pkg/features/kube_features.go`, `pkg/cache/scheduler/tas_flavor_snapshot.go`,
`pkg/cache/scheduler/tas_balanced_placement.go`, `pkg/controller/tas/topology_ungater.go`,
`keps/2724-topology-aware-scheduling/README.md`, and the in-repo docs under
`site/content/en/docs/`. Where a default or a version is stated, it is the one in that tree.

## Why this matters

Anthropic's Sr Staff+ Kubernetes Platform JD names "gang scheduling; topology-aware placement"
in one breath, not as two bullet points — because gang without topology is an incomplete answer
in that interview. Synchronous data-parallel training is gated on an all-reduce at every step,
and a collective is only as fast as the slowest link in it. If part of the communication pattern
crosses a node, rack, or spine boundary, the *entire* gang idles waiting on that link. You are
billed for N GPUs and getting the throughput of a badly bottlenecked fraction.

Nothing in a quota dashboard flags this. It reads as "8/8 GPUs admitted, job Running", every
GPU shows high memory allocation, and the only symptom is that training is mysteriously,
expensively slow. That is the same observability trap lesson 1 described for the deadlock, one
layer down: the cheap metric says healthy, and only a per-step timing breakdown says otherwise.

The economics are not subtle. §5 derives the exact multiplier from two ratios you can measure on
your own fleet, and §14 turns it into a break-even you can defend. Being the engineer who ties a
slow training run back to a split gang, and fixes it with a topology constraint rather than more
hardware, is squarely the FinOps differentiator this course is building toward.

## What's new here (calibration)

- **Module 02b (host topology)** gave you NVLink domains, NVSwitch, PCIe host boundaries, and
  the kubelet's Topology Manager aligning CPU/NUMA/device locality *on one node*. This lesson
  does not re-derive single-node NUMA alignment. It restates the bandwidth numbers compactly
  because the arithmetic in §4–§5 depends on them, and then leaves single-node behind: from §6
  onward the unit of analysis is a **multi-node topology domain**.
- **Lessons 2 and 5** gave you gang admission and the three schedulers' models of it, including
  the one-line comparison of Kueue's `Topology`, Volcano's `HyperNode` tier tree, and KAI's
  `Topology` plus placement annotations. Not re-derived; §12 says exactly how TAS composes with
  gang and points back.
- **Genuinely new here:**
  - **The collective-tax derivation.** Ring all-reduce from first principles — the ring's
    construction, the 2(N−1) steps, the bytes-per-step term, and the bus-bandwidth identity —
    then a cut-bandwidth lower bound that tells you the *minimum* time any all-reduce can spend
    crossing a boundary. This is where the "2–5×" in the concept line comes from, and you will
    be able to compute it for your own fabric instead of quoting it.
  - **Kueue's `Topology` API in full**, including the CEL validation rules that constrain what a
    valid level list looks like, and why `kubernetes.io/hostname` is special.
  - **All five placement annotations** — required, preferred, unconstrained, slice, and
    multi-layer slice constraints — with their exact semantics and version status.
  - **The TAS placement algorithm, step by step**: the two-phase tree walk, and the three
    domain-selection strategies (`BestFit`, `LeastFreeCapacity`, `BalancedPlacement`) with the
    feature-gate matrix that decides which one runs for which annotation.
  - **How the assignment is enforced**: the `kueue.x-k8s.io/topology` scheduling gate, node
    selector injection, the ungater controller, and the `topologyAssignment` field in the
    Workload status with its prefix-compressed encoding.
  - **What happens when a node dies mid-run** — TAS hot swap, its four interacting feature
    gates, and why a rack-filling workload can never be hot-swapped.

## Core concepts

### 1. The problem: admission answers "whether", not "where"

Lesson 2 wrote the gang constraint down precisely: for a set of pods `G` with quorum `k`, no
member of `G` may be bound unless at least `k` members can be placed against a single consistent
view of cluster state. That constraint is entirely about **counting**. It is satisfied by
placing all eight replicas of a job on eight different nodes in eight different racks.

For a tightly-coupled training job, that is a correct admission and a wrong placement. The
constraint you actually need is a second, independent one:

> For a set of pods `G` requesting topology level `L`, all members of `G` must be placed within a
> **single topology domain** at level `L` — one rack, one block, one NVLink domain — evaluated
> against the same consistent view of cluster state.

Two constraints, two failure modes, two mechanisms. Gang scheduling without topology gives you
"all 8 pods, anywhere". Topology without gang gives you "the pods that landed are in one rack,
and the rest are pending". You need both, and §12 shows they compose conjunctively.

The reason the second constraint exists is not a Kubernetes concern at all. It is a property of
the collective-communication runtime, and the next four sections derive it.

### 2. The bandwidth hierarchy, as numbers you can compute with

Module 02b established the shape of this hierarchy. Here are the figures the arithmetic below
uses, with their provenance, because unsourced bandwidth numbers are how bad capacity models get
built.

| Link | Generation / product | Per-GPU bandwidth (one direction) | Source |
|---|---|---|---|
| NVLink 3 | A100 SXM4, 12 links × 25 GB/s | **300 GB/s** (600 GB/s bidirectional) | NVIDIA A100 datasheet |
| NVLink 4 | H100 SXM5, 18 links × 25 GB/s | **450 GB/s** (900 GB/s bidirectional) | NVIDIA H100 datasheet |
| NVLink 5 | B200 / GB200 | **900 GB/s** (1.8 TB/s bidirectional) | NVIDIA GB200 NVL72 product page |
| NVSwitch (in-node) | HGX/DGX H100, 4× NVSwitch gen3 | 3.6 TB/s per direction across 8 GPUs (7.2 TB/s bidirectional aggregate) | NVIDIA DGX H100 datasheet |
| NVLink Switch (rack) | GB200 NVL72, 72 GPUs in one domain | 130 TB/s aggregate across the domain | NVIDIA GB200 NVL72 product page |
| PCIe Gen5 ×16 | host↔device, PCIe-form-factor GPUs | 64 GB/s | PCIe 5.0 base spec (32 GT/s × 16 lanes) |
| InfiniBand NDR | 400 Gb/s per port | **50 GB/s** raw, ≈45–48 GB/s payload | IBTA NDR signalling rate |
| DGX H100 compute fabric | 8× ConnectX-7 @ 400 Gb/s | **400 GB/s per node** = 50 GB/s per GPU | NVIDIA DGX H100 datasheet |
| Commodity GPU node | 1× or 2× 100 GbE | **12.5–25 GB/s per node** | — |

Read the last three rows together, because that is where the whole lesson lives. On a
**rail-optimised DGX H100** — one 400 Gb/s NIC per GPU — the step down from NVLink to the
network is 450 → 50 GB/s, a factor of **9**. On a cost-optimised node with a single 100 GbE
NIC shared by 8 GPUs, the step is 450 → 1.6 GB/s per GPU, a factor of **288**. The tax you pay
for a split gang is not a constant; it is a property of *your* fabric, and you must know which
of those two worlds you are in before you can size anything.

Above the node there is a second step down, and it is the one people forget. A leaf-spine fabric
is frequently **oversubscribed**: if a rack's 32 downlinks are served by 8 uplinks, the spine is
4:1 oversubscribed and cross-rack bandwidth per node is a quarter of intra-rack bandwidth *under
contention*. Non-blocking (1:1) fat-trees exist and are what dedicated AI clusters buy, but
"we're 1:1" is a claim to verify against the cabling, not assume.

```
  THE PHYSICAL TREE, AND TWO PLACEMENTS OF THE SAME 8-GPU JOB
  ══════════════════════════════════════════════════════════════════════════════════════

                              ┌──────────── SPINE ────────────┐
                              │   oversubscription 1:1..4:1   │
                              └───┬───────────────────────┬───┘
                       400 GB/s   │                       │   400 GB/s
                       per node   │                       │   per node
                    ┌─────────────┴──────┐         ┌──────┴─────────────┐
                    │   LEAF / rack r1   │         │   LEAF / rack r2   │
                    └──┬──────────────┬──┘         └──┬──────────────┬──┘
                       │              │               │              │
                   ┌───┴───┐      ┌───┴───┐       ┌───┴───┐      ┌───┴───┐
                   │node a │      │node b │       │node c │      │node d │
                   │NVSwitch      │NVSwitch       │NVSwitch      │NVSwitch
                   │3.6 TB/s      │3.6 TB/s       │3.6 TB/s      │3.6 TB/s
                   │[GPU×8]│      │[GPU×8]│       │[GPU×8]│      │[GPU×8]│
                   └───────┘      └───────┘       └───────┘      └───────┘

  PLACEMENT A — topology-aware, required at kubernetes.io/hostname
  ─────────────────────────────────────────────────────────────────
     node a: [J J J J J J J J]      all 8 ranks inside ONE NVSwitch domain
     every collective link  = 450 GB/s          bytes crossing any NIC = 0
     measured NCCL all-reduce bus bandwidth ≈ 370–480 GB/s (representative)

  PLACEMENT B — topology-blind, 4 + 4 across the spine
  ─────────────────────────────────────────────────────────────────
     node a: [J J J J . . . .]   node c: [J J J J . . . .]
     4 links intra-node @ 450 GB/s   ·   the a|c cut @ 4 NICs = 200 GB/s
     every step, S bytes must cross that cut in each direction
     → the collective is now bounded by the CUT, not by NVLink

  PLACEMENT C — the pathological one, 1 GPU per node across 8 nodes
  ─────────────────────────────────────────────────────────────────
     8 nodes × 1 rank, 7 of 8 ring links cross a NIC, some cross the spine
     nothing runs on NVLink at all; the job is a network benchmark with
     an expensive matrix multiply attached
```

### 3. Ring all-reduce, derived

You cannot reason about the tax without knowing what the collective actually does. Here is the
ring algorithm in full — it is the one NCCL uses by default for large messages, and every other
algorithm is a variation on the same accounting.

**Setup.** `N` ranks, each holding a gradient buffer of `S` bytes. Every rank must end holding
the element-wise sum of all `N` buffers. Arrange the ranks in a logical ring: rank *i* sends only
to rank *(i+1) mod N* and receives only from *(i−1) mod N*. Split each rank's buffer into `N`
equal chunks of `S/N` bytes.

**Phase 1 — reduce-scatter, N−1 steps.** In step *t*, rank *i* sends chunk *(i − t) mod N* to its
successor and receives chunk *(i − t − 1) mod N* from its predecessor, adding what it receives to
its own copy. After `N−1` steps, rank *i* holds the fully reduced chunk *(i + 1) mod N* and
nobody else does.

**Phase 2 — all-gather, N−1 steps.** The same ring, same chunk size, no addition: each rank
forwards the finished chunk it owns until every rank has all `N` finished chunks.

**The accounting that falls out.** Total steps = `2(N−1)`. Bytes sent per rank per step = `S/N`.
Therefore:

```
  bytes_sent_per_rank  = 2(N−1)/N × S
  time                 = 2(N−1)/N × S / B_link          [B_link = slowest ring link, one direction]

  algorithm bandwidth  algbw = S / time
  bus bandwidth        busbw = algbw × 2(N−1)/N         ← this is the number nccl-tests prints
```

Three consequences worth internalising:

1. **`busbw` is bounded by the slowest single link in the ring, not by the average.** A ring is a
   chain; one 50 GB/s hop in a chain of 450 GB/s hops sets the pace for every step.
2. **The `2(N−1)/N` factor saturates at 2.** At N=8 it is 1.75; at N=1024 it is 1.998. So
   doubling the world size does not double the communication time per rank — the *volume* per
   rank is essentially constant. What grows is the number of links the ring must traverse, and
   therefore the probability that one of them is slow.
3. **The measured `busbw` on the good placement is the ceiling you are giving up.** On 8×H100
   with NVSwitch, `all_reduce_perf` typically reports 370–480 GB/s for messages ≥1 GiB,
   depending on NCCL version and message size. That range — not the 450 GB/s paper figure — is
   what you should use as the "fast" term in a cost model.

NCCL does not always use a flat ring. For multi-node jobs it builds **multiple rings** to use
all NICs, offers a **tree** algorithm (better latency at small message sizes, `NCCL_ALGO=Tree`),
**NVLS**/`NVLSTree` on NVSwitch hardware where the switch performs the reduction in-network, and
**CollNet** on fabrics with SHARP. It also constructs **rail-optimised** rings when it detects
one NIC per GPU, so that each ring crosses the network on its own rail. All of these change
constants; none of them change the fact that the collective is bounded by the bandwidth across
the cut it has to traverse. That bound is next.

### 4. The cut bound: the honest lower limit on a split gang

Forget the algorithm for a moment and ask an information-theoretic question. Partition the `N`
ranks into two groups A and B. Every rank must end holding a sum that depends on data held in
both groups. Two facts follow immediately:

- Group A's contribution to the result — at minimum `S` bytes' worth of reduced partial sums —
  must reach group B, so at least `S` bytes cross the cut from A to B.
- Every rank in A must end with the part of the result that depends on B's data, so at least
  `S` bytes cross from B to A.

So for **any** all-reduce algorithm, on any topology:

```
  T_cross  ≥  S / B_cut          [B_cut = usable bandwidth across the A|B boundary, one direction]
```

That is a floor, not an estimate. It holds for rings, trees, NVLS, hand-rolled MPI, anything.
It is also the single most useful formula in this lesson, because `B_cut` is a number you can
look up from your NIC count and `S` is a number you can compute from your model size.

**Worked instance.** Take a 7-billion-parameter model trained in bf16 with plain DDP: the
gradient buffer is `7×10⁹ params × 2 bytes = 14 GB`, all-reduced once per optimizer step.

| Placement | `B_cut` (one direction) | `T_cross` floor | Same-node collective time |
|---|---|---|---|
| 8 GPUs, one HGX H100 node | — (no cut) | 0 | `1.75 × 14 GB / 400 GB/s` = **61 ms** |
| 4 + 4, two DGX H100 nodes, rail-optimised | 4 NICs × 50 GB/s = 200 GB/s | `14/200` = **70 ms** | — |
| 4 + 4, two nodes, 1× 100 GbE each | 12.5 GB/s | `14/12.5` = **1,120 ms** | — |
| 1 GPU × 8 nodes, 1× 100 GbE each | 12.5 GB/s at the worst cut | **1,120 ms** | — |
| 4 + 4 across a 4:1 oversubscribed spine, under contention | 200/4 = 50 GB/s | `14/50` = **280 ms** | — |

Read the second row carefully, because it is the row that corrects the folklore. On a properly
rail-optimised DGX-class fabric, splitting an 8-GPU job across two nodes costs you roughly
`70 ms` instead of `61 ms` on the collective — about **15%**, not 500%. The catastrophic numbers
come from cheap networking (row 3, an **18×** collective slowdown) or from spine contention
(row 5, **4.6×**). **The tax is a property of your fabric, and if you quote "2–5×" without
naming the fabric you are quoting someone else's cluster.**

### 5. From collective time to wall-clock cost: the tax formula

The collective is not the whole step. Let:

- `T_comp` = per-step compute time (forward + backward), seconds
- `T_comm_fast` = collective time in the good placement, seconds
- `T_comm_slow` = collective time in the bad placement, seconds
- `φ` = fraction of the collective that overlaps with compute, `0 ≤ φ ≤ 1`

With no overlap (`φ = 0`), step time is `T_comp + T_comm`, and the slowdown factor is:

```
  slowdown = (T_comp + T_comm_slow) / (T_comp + T_comm_fast)
```

With partial overlap — which is what PyTorch DDP's gradient bucketing actually gives you, by
launching the all-reduce for early buckets while the backward pass is still running:

```
  step_time = max( T_comp , (1−φ)·T_comm + φ·T_comm )   ... more usefully:
  step_time ≈ T_comp + max(0, T_comm − φ·T_comp)
```

**Worked instance, continued.** Same 7B model, 8×H100, `T_comp = 250 ms/step` (a plausible
figure for a large micro-batch; substitute your own from a profiler), `φ = 0.7`.

```
  GOOD placement (one node):
     T_comm_fast = 61 ms;  overlap budget = 0.7 × 250 ms = 175 ms
     exposed comm = max(0, 61 − 175) = 0 ms
     step = 250 ms                                       → 4.00 steps/s

  BAD placement, rail-optimised 4+4:
     T_comm_slow = 70 ms  → exposed comm = max(0, 70 − 175) = 0 ms
     step = 250 ms                                       → 4.00 steps/s
     slowdown = 1.00×      ← the fabric absorbed it entirely

  BAD placement, 1× 100 GbE 4+4:
     T_comm_slow = 1,120 ms → exposed = 1,120 − 175 = 945 ms
     step = 250 + 945 = 1,195 ms                         → 0.84 steps/s
     slowdown = 4.78×      ← THIS is where "2–5× slower" comes from

  BAD placement, 4:1 oversubscribed spine under contention:
     T_comm_slow = 280 ms  → exposed = 280 − 175 = 105 ms
     step = 355 ms                                       → 2.82 steps/s
     slowdown = 1.42×
```

**Now attach money.** A run that would take `H` GPU-hours in the good placement takes
`H × slowdown` in the bad one, and you pay for the difference at your rate:

```
  wasted_GPU_hours = H × (slowdown − 1)
  wasted_cost      = wasted_GPU_hours × rate_per_GPU_hour
```

For a 14-day, 8-GPU run at the 4.78× figure and a **$2.35/GPU-hr H100 on-demand snapshot**
(specialized-neocloud segment; see lesson 8's market-segment table before quoting any rate):

```
  H (good placement)  = 8 GPUs × 14 days × 24 h        = 2,688 GPU-hours
  wasted GPU-hours    = 2,688 × (4.78 − 1)             = 10,161 GPU-hours
  wasted cost         = 10,161 × $2.35                 = $23,878
  and the run finishes 53 days later than it should have
```

At the 1.42× spine-contention figure the same run wastes `2,688 × 0.42 = 1,129` GPU-hours
≈ **$2,653**. At the rail-optimised 1.00× figure it wastes nothing, and a `required` constraint
that makes the job *wait* for a clean node would be strictly harmful. **The correct topology
policy is a function of your fabric's cut bandwidth and your model's compute-to-comm ratio. Two
numbers. Measure them once and the policy writes itself.**

### 6. Kueue's `Topology` object — naming the hierarchy

TAS models the datacenter as a hierarchy of **node labels**, coarsest first. The object is
cluster-scoped, in API group `kueue.x-k8s.io`, storage version `v1beta2` at the current head
(`v1beta1` still served; the module README's "API is v1beta1" note predates the promotion).

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: Topology
metadata:
  name: default                                    # referenced by name from a ResourceFlavor
spec:
  levels:
    - nodeLabel: "cloud.provider.com/topology-block"   # coarsest: NVLink block / super-pod
    - nodeLabel: "cloud.provider.com/topology-rack"    # rack
    - nodeLabel: "kubernetes.io/hostname"              # finest: a single node
```

The API constrains this list more tightly than it looks, and the constraints are CEL rules on
the type, so the API server rejects violations at write time
(`apis/kueue/v1beta2/topology_types.go`):

| Rule | What it enforces | Why |
|---|---|---|
| `MinItems=1`, `MaxItems=16` | between 1 and 16 levels | bounds the tree depth the scheduler walks |
| levels must be unique | no repeated `nodeLabel` | a duplicated level would double-count capacity |
| `kubernetes.io/hostname`, if present, must be the **last** level | host is the finest granularity | a level below "one node" is meaningless to a pod-placement decision |
| `levels` is **immutable unless** `kubernetes.io/hostname` is the lowest level both before and after the edit | you cannot reshape a hierarchy under running workloads | admitted Workloads carry a `topologyAssignment` expressed in these levels |

Two operational facts that are easy to get wrong:

- **Every node that participates must carry every label in the hierarchy.** A node missing one
  level's label is not a partially-usable node; it is excluded from TAS's view at that level, and
  your capacity silently shrinks. §15 covers the failure mode.
- **`kubernetes.io/hostname` as the lowest level is not just a convention.** Kueue's own GPU
  example annotates it: node taints are only respected when hostname is the lowest level,
  because that is the only case where TAS resolves the assignment down to individual named
  nodes. If your lowest level is `rack`, TAS reasons about racks and leaves the node choice to
  `kube-scheduler`.

In production these labels come from the cloud provider's real fabric topology — AWS EC2
UltraCluster placement groups, GCP compact placement policies and the A3/A3-Mega topology labels,
CoreWeave's rack/leaf labels — or from your own DCIM export on-prem. **TAS is exactly as accurate
as those labels. It has no way to detect that a label lies.**

### 7. Wiring it to a ClusterQueue

A `ResourceFlavor` points at the `Topology` via `spec.topologyName`; the `ClusterQueue` uses that
flavor. This is the same flavor machinery from lessons 3–4, now carrying a topology reference.
Here is a complete, GPU-shaped setup — every field that matters, annotated:

```yaml
apiVersion: kueue.x-k8s.io/v1beta2
kind: Topology
metadata:
  name: default
spec:
  levels:
    - nodeLabel: "cloud.provider.com/topology-block"
    - nodeLabel: "cloud.provider.com/topology-rack"
    - nodeLabel: "kubernetes.io/hostname"   # taints are only respected when host is lowest
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ResourceFlavor
metadata:
  name: tas-h100
spec:
  nodeLabels:
    cloud.provider.com/node-group: "tas-group"   # which nodes this flavor covers
  topologyName: "default"                        # <-- binds the flavor to the hierarchy
  tolerations:
    # Most cloud providers inject GPU tolerations into pods via a webhook. TAS computes
    # capacity itself and is NOT aware of that webhook, so a tainted GPU node would be
    # excluded from TAS's capacity view unless the toleration is declared HERE.
    - key: "nvidia.com/gpu"
      operator: "Exists"
      effect: "NoSchedule"
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata:
  name: tas-cluster-queue
spec:
  namespaceSelector: {}
  resourceGroups:
    - coveredResources: ["cpu", "memory", "nvidia.com/gpu"]
      flavors:
        - name: tas-h100
          resources:
            - name: "cpu"
              nominalQuota: 100
            - name: "memory"
              nominalQuota: 100Gi
            - name: "nvidia.com/gpu"
              nominalQuota: 16
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata:
  namespace: default
  name: tas-user-queue
spec:
  clusterQueue: tas-cluster-queue
```

The `tolerations` block on the ResourceFlavor is the single most commonly missed field in a TAS
setup, and its failure mode is nasty: the Workload sits pending with a message saying the
topology cannot fit the pods, on a cluster where `kubectl get nodes` shows plenty of idle GPUs.
TAS excluded every tainted node from its own capacity computation.

### 8. How TAS computes free capacity

This is the part that decides whether a `required` constraint admits or waits, so it is worth
being exact. For each PodSet, per topology domain, TAS computes free capacity as:

```
  free(domain) =   Σ over nodes in domain, where the node is
                     Ready=True  AND  .spec.unschedulable == false
                   of  node.status.allocatable
                 − usage from all other ADMITTED TAS workloads assigned to that domain
                 − usage from all NON-TAS pods on those nodes
                     (DaemonSets, static pods, Deployments outside Kueue, …)
```

Three implications:

- **Cordoned and NotReady nodes vanish from capacity immediately.** A `kubectl cordon` on one
  node of a rack can flip a `required: rack` workload from admissible to pending, with no other
  change.
- **DaemonSets count against you.** A monitoring agent requesting 200m CPU on every node is 200m
  less that TAS believes is available. If your gang is CPU-tight as well as GPU-tight, DaemonSet
  growth is a silent capacity regression.
- **TAS tracks every pod and every node in the cluster.** Kueue's own docs list this under
  *Drawbacks*: enabling TAS raises Kueue's memory footprint and lengthens scheduling time,
  because the informer set and the per-domain accounting grow with cluster size. On a
  thousand-node fleet that is a real operational cost to budget for, not a footnote.

### 9. The five placement annotations

All of these go on the **Pod template's** `metadata.annotations` — `spec.template.metadata` for a
Job — not on the Job's own metadata. Setting them one level too high is the most common user
error and produces no error message at all; TAS simply never engages.

| Annotation | Value | Semantics |
|---|---|---|
| `kueue.x-k8s.io/podset-required-topology` | a level's node-label key | **Hard.** All pods of the PodSet must fit in one domain at that exact level. If none can, the Workload waits — no fallback, no spread. |
| `kueue.x-k8s.io/podset-preferred-topology` | a level's node-label key | **Soft.** Try that level; if no single domain fits, evaluate each **coarser** level in turn; if it does not fit even at the top level, admit it distributed across multiple domains. Never blocks. |
| `kueue.x-k8s.io/podset-unconstrained-topology` | the literal string `"true"` | **No topology requirement**, but TAS bookkeeping still applies — it packs pods into partially-used nodes to minimise fragmentation. Not the same as omitting the annotation. |
| `kueue.x-k8s.io/podset-slice-required-topology` + `kueue.x-k8s.io/podset-slice-size` | level key + integer | Split the PodSet into equal slices of `slice-size` pods; each **slice** must fit in one domain at the named level. Slice level must be at or below the main topology level. Size must be > 0 and is validated strictly since v0.19. |
| `kueue.x-k8s.io/podset-slice-required-topology-constraints` | JSON array, ≤3 entries | Multi-layer slices: a level and size per layer, ordered coarsest → finest; each inner size must evenly divide the outer. Mutually exclusive with the two single-layer slice annotations. Gate `TASMultiLayerTopology`: alpha v0.17, **beta and on by default v0.19**. |

A sixth, adjacent one: `kueue.x-k8s.io/podset-group-name` makes several PodSets of one Workload
— a leader PodSet and a workers PodSet, for example — a single unit of flavor assignment and
topology fitting, so they land in the same domain.

Choosing between required and preferred:

- **`required`** when the collective genuinely cannot tolerate a cross-domain link — large
  synchronous data-parallel or tensor-parallel training where the exposed comm term in §5 is
  large. Accept the consequence: if no single domain can hold the gang right now, the workload
  waits. §14 turns that into a break-even rather than a rule of thumb.
- **`preferred`** when locality helps but the workload degrades gracefully — gradient
  compression, embarrassingly-parallel sweeps, inference replicas, evaluation harnesses. Also
  the right default while your labels are still unproven.
- **`unconstrained`** when you do not care where pods land but you *do* want TAS's gap-filling.
  This is a fragmentation tool, and it is the case where the placement algorithm differs (§10).

And choosing the **level**: match it to the collective's reach. A gang that fits in one box →
`kubernetes.io/hostname`. A 16–32-GPU job spanning a few boxes → `topology-rack`. A super-pod
job, or anything on GB200 NVL72 where the NVLink domain is 72 GPUs and therefore a *rack* →
`topology-block`. Note what NVL72 does to this reasoning: the boundary that used to be "the
node" is now "the rack", so the level you constrain has to move up a step for the same physical
guarantee. Getting this wrong is §15's "constrained the wrong level" pitfall.

### 10. The placement algorithm, step by step

When more than one domain could satisfy the request, TAS still has to choose. The algorithm lives
in `pkg/cache/scheduler/tas_flavor_snapshot.go`, function `findTopologyAssignment`, and it is a
two-phase tree walk.

**Phase 1 — bottom-up capacity roll-up.** Starting at the leaf domains (usually hosts), compute
how many pods of this PodSet fit in each leaf, given the free-capacity rule from §8 and the
PodSet's per-pod resource requests. Then bubble those counts up the tree: a rack's `podCount` is
the sum of its hosts', a block's is the sum of its racks'. When slices are in play, the same
walk also computes `sliceCount = podCount / sliceSize` per domain.

**Phase 2 — top-down selection.** Start at the level the annotation named (for `required`), or
search upward from it (for `preferred`), and at each level:

1. **Sort the domains** at that level by the chosen strategy.
2. **Select** the domain, or the minimal consecutive set of domains, that fits the request.
3. **Descend** one level into the selected domain(s) and repeat, optimising the number of domains
   used at each level.
4. At the lowest level, emit the assignment: a list of `(domain value, pod count)`.

The interesting part is step 1, because there are three strategies and which one runs depends on
the annotation *and* a feature gate.

**`BestFit`** — sort domains **descending** by free capacity. If the largest domain alone can
hold the request, do not just take it: scan all domains that fit and take the **smallest one that
still fits** (`findBestFitDomainBy`, which tracks `bestDomainCount` as the minimum count ≥ needed).
That is the "tight fit" optimisation, and it applies to the *last* domain chosen at each level.
Intent: leave large domains intact.

**`LeastFreeCapacity`** — sort **ascending** by free capacity and take the first domain that fits.
Intent: consume the already-tight domains first, so the roomy ones stay whole.

**`BalancedPlacement`** — find the optimal *set* of domains that fits, then distribute pods as
evenly as possible across that set, maximising the minimum per-domain pod count one level below
the annotated level, and secondarily minimising the number of domains used at the annotated
level. Intent: avoid lopsided splits.

Which one runs (from KEP-2724's own matrix, cross-checked against `useBestFitAlgorithm` /
`useLeastFreeCapacityAlgorithm` in the source):

| Configuration | `preferred` | `required` | `unconstrained` |
|---|---|---|---|
| Kueue ≤ v0.14, no gates | BestFit | BestFit | BestFit |
| **Kueue ≥ v0.15, default** (`TASProfileMixed` beta, on) | BestFit | BestFit | **LeastFreeCapacity** |
| `TASProfileBestFit` (deprecated) | BestFit | BestFit | BestFit |
| `TASProfileLeastFreeCapacity` | LeastFreeCapacity | LeastFreeCapacity | LeastFreeCapacity | *(removed in v0.17)* |
| `TASBalancedPlacement` (alpha since v0.15, off by default) | **BalancedPlacement** | BestFit | LeastFreeCapacity |

`TASProfileMixed` went alpha in v0.10 and **beta, default-on, in v0.15**; the gate is now
formally deprecated because its behaviour is the default. The rationale in the KEP is one
sentence worth memorising: for `unconstrained` requests Kueue should "prioritize minimizing
fragmentation", which is what `LeastFreeCapacity` provides.

The KEP's own worked example makes the difference concrete. **One rack, four nodes with free
capacity for 3, 3, 2 and 1 pods. A PodSet of 7 pods:**

```
  DOMAIN INVENTORY (pods that fit):   n1=3   n2=3   n3=2   n4=1        request = 7 pods

  BestFit  — sort DESC [3,3,2,1], take greedily, optimise the LAST domain
     take n1 (3)  → remaining 4
     take n2 (3)  → remaining 1
     remaining 1 fits in n3(2) and n4(1); the TIGHTEST fit is n4 → take n4
     RESULT: n1=3, n2=3, n4=1        n3 left untouched with 2 free
             ┌───┬───┬───┬───┐
             │███│███│ . │█  │      n3 stays a usable 2-pod hole
             │ 3 │ 3 │ 2 │ 1 │
             └───┴───┴───┴───┘

  LeastFreeCapacity — sort ASC [1,2,3,3], take in that order
     take n4 (1) → 6 ;  take n3 (2) → 4 ;  take n2 (3) → 1 ;  take 1 of n1's 3
     RESULT: n4=1, n3=2, n2=3, n1=1   n1 left with 2 free
             ┌───┬───┬───┬───┐
             │█  │███│██ │█  │      the ROOMIEST domain keeps its slack
             │ 3 │ 3 │ 2 │ 1 │
             └───┴───┴───┴───┘

  BalancedPlacement (12 pods over two domains of capacity 10,10)
     greedy strategies produce   (10, 2)
     balanced produces           ( 6, 6)      ← matters for all-to-all patterns,
                                                where the lopsided split creates
                                                one badly loaded cross-domain link
```

The reason lesson 7 cares about this: **`LeastFreeCapacity` is Kueue baking "don't strand your
biggest hole" into the default placement policy**, which is exactly the instinct lesson 7 teaches
you to compute after the fact. If you rely on default unconstrained behaviour, know which
version you run — the fragmentation consequences are not cosmetic.

One more source-level detail with operational teeth. Kueue's regular scheduling has a
**nomination phase** (evaluate each Workload independently) followed by an **evaluation phase**
(process them one at a time). With ordinary quota, a Workload nominated as "fits" almost always
still fits when evaluated, because flavors are big pools. With TAS the assignment is effectively
per-node, so a Workload can be nominated Fit and then fail with reason `FailedAfterNomination`
because an earlier Workload in the same cycle took the domain. The KEP records that this caused
**outages** in environments with many ClusterQueues, where workloads stayed suspended for hours
through repeated conflicts. The fix — recompute the TAS assignment within the cycle when quota
still fits but placement does not — landed in **v0.19.0** behind
`TASRecomputeAssignmentWithinSchedulingCycle` (beta, on by default) and was cherry-picked to
v0.17.6 and v0.18.2. If you run TAS on an older patch release with many queues, that is a known
bug with a known fix, not a mystery.

### 11. How the assignment is enforced

Knowing *where* the pods should go is only half of it. Kueue does not bind pods — `kube-scheduler`
does — so TAS has to convert its decision into something the default scheduler will obey. The
mechanism is a **scheduling gate** plus **node selector injection**, and this is the temporal
picture you should be able to draw on a whiteboard.

```
  TAS ADMISSION AND ENFORCEMENT — one Workload, end to end
  ══════════════════════════════════════════════════════════════════════════════════════

  t0   user creates Job, label kueue.x-k8s.io/queue-name: tas-user-queue
         │
         ▼
  t1   Kueue mutating webhook sets spec.suspend=true          ⇒ ZERO pods exist
         │                                                        ZERO GPUs held
         ▼
  t2   Kueue creates the Workload object: PodSets + aggregate resource ask
         │
         ▼
  t3   QUOTA CHECK against the ClusterQueue / cohort  (lessons 3–4)
         │  ├─ doesn't fit → Workload waits, QuotaReserved=False
         │  └─ fits → quota reserved
         ▼
  t4   TAS PLACEMENT CHECK against the flavor's Topology  (§8, §10)
         │  phase 1: roll capacity up the tree
         │  phase 2: sort + select + descend
         │  ├─ required and no single domain fits →
         │  │     Workload stays pending. QuotaReserved condition message:
         │  │     topology "default" allows to fit only 4 out of 10 pod(s).
         │  │     Total nodes: 8; excluded: resource "cpu": 6
         │  │     ← quota was FINE. TAS is what refused.
         │  └─ fits → write status.admission.podSetAssignments[].topologyAssignment
         ▼
  t5   Kueue sets spec.suspend=false                          ⇒ pods are created NOW
         │  each pod carries the scheduling gate:
         │      spec.schedulingGates: [ {name: kueue.x-k8s.io/topology} ]
         │  a gated pod is NOT eligible for scheduling — PreEnqueue holds it
         ▼
  t6   topology_ungater controller reads the Workload's topologyAssignment,
         injects a nodeSelector matching the assigned domain(s) into each pod,
         then REMOVES the gate
         │
         ▼
  t7   kube-scheduler runs its normal cycle. Filter now only accepts nodes in
         the assigned domain, because the nodeSelector says so.
         │
         ▼
  t8   pods bind, inside the domain TAS chose.  Collective runs on NVLink.
```

The gate is the load-bearing piece. Without it there is a race: pods would be created and
scheduled by `kube-scheduler` before Kueue had written the node selector, and they would land
wherever the default scoring put them. `kueue.x-k8s.io/topology` closes that race by making the
pods ineligible for scheduling until the assignment exists. (For pod-based integrations the gate
is added by the webhook at pod creation time.)

The assignment itself is visible and is the authoritative signal that TAS was applied:

```bash
$ WL=$(kubectl get workload -o name --sort-by='.metadata.creationTimestamp' | tail -n1)
$ kubectl get $WL -o jsonpath='{.status.admission.podSetAssignments[0].topologyAssignment}' | jq
{
  "levels": [ "kubernetes.io/hostname" ],
  "slices": [
    {
      "domainCount": 2,
      "podCounts": { "individual": [ 7, 3 ] },
      "valuesPerLevel": [
        { "individual": { "prefix": "kind-worker", "roots": [ "", "2" ] } }
      ]
    }
  ]
}
```

Read that output line by line, because the encoding is not obvious:

- `levels` — which topology levels the assignment is expressed in. Here just hostname, so the
  assignment names individual nodes.
- `podCounts.individual: [7, 3]` — seven pods on the first named domain, three on the second.
- `valuesPerLevel[0].individual` — the domain names, **prefix-compressed**. `prefix: "kind-worker"`
  with `roots: ["", "2"]` expands to `kind-worker` and `kind-worker2`. The compression is
  controlled by `TASAssignmentsEncodingByHostnamePrefix` (beta, default-on since v0.19); it
  exists because a 1,000-node assignment written out in full is a large object in etcd.

So this Workload's ten pods went 7 + 3 across two nodes in one rack — a `required: rack`
constraint satisfied, with the pods spread inside the rack because a single host could not hold
ten.

### 12. TAS and gang: two constraints, conjunctive

This is a favourite interview probe, and the precise answer is short.

Kueue admits a Workload only when **both** hold:

1. quota is available in the ClusterQueue (plus any cohort borrowing / preemption, lessons 3–4);
2. TAS can find a domain assignment satisfying the PodSet's topology annotation for the **entire**
   PodSet.

Kueue does not admit a fraction of a Workload unless you explicitly enable partial admission, so
condition (2) is evaluated against the whole PodSet. That gives you gang-shaped atomicity at the
quota layer and topology at the placement layer simultaneously. Underneath, whatever enforces
gang at the pod layer — the coscheduling plugin, Volcano, or the native `Gang{minCount}` path
from lesson 2 §10 — now operates on pods that already carry a node selector confining them to
TAS's chosen domain.

Why you need both, stated as failure cases:

- **Gang alone**: "all 8 pods or none" is satisfied by 4 + 4 across the spine. Correct admission,
  §5's tax.
- **TAS alone (no gang, no Kueue-level atomicity)**: some pods land in the good domain, the rest
  pend. That is lesson 1's partial-placement deadlock with better-placed victims.
- **Both**: all pods, in one domain, or the Workload waits holding nothing. The only correct
  outcome for tightly-coupled training.

The three schedulers express this differently — Kueue's `Topology` label levels, Volcano's
`HyperNode` tier tree with `networkTopology.mode: hard|soft` and `highestTierAllowed`, KAI's
`Topology` CRD with `kai.scheduler/topology-required-placement` and `-preferred-placement`
annotations. Lesson 5 §4 has the comparison; the mechanism you take apart here is Kueue's,
because that is what the deliverable builds on.

### 13. When a node dies mid-run: hot swap

Because TAS pins a workload to named nodes via node selectors, a node failure is more disruptive
than it would be under plain `kube-scheduler`: the pod cannot simply be rescheduled anywhere.
Kueue's answer is **node hot swap** — find a replacement node for the failed one, keeping the
rest of the assignment intact, rather than evicting and re-admitting the whole workload.

Four feature gates interact here, and their current status in the tree is:

| Gate | Status | What it does |
|---|---|---|
| `TASFailedNodeReplacement` | alpha v0.12 → **beta, on, v0.14** | enables hot swap at all |
| `TASReplaceNodeOnPodTermination` | **beta, on, v0.14** | a NotReady node counts as failed only once the workload's pods there are terminating/terminated |
| `TASFailedNodeReplacementFailFast` | **beta, on, v0.14** | try the replacement search **once**; if it fails, evict and requeue instead of retrying |
| `TASReplaceNodeOnNodeTaints` | **beta, on, v0.17** | an untolerated `NoExecute`/`NoSchedule` taint also marks a node as failed, so drained capacity is released quickly |
| `TASReplaceNodeDueToNotReadyOverFixedTime` | on by default v0.17–v0.18, **off and deprecated since v0.19** | the old behaviour: NotReady for 30 s marks the node failed regardless of pod state |

The default-behaviour change at v0.19 is the one to know: a node that is `NotReady` but whose
pods are still running is **no longer** treated as failed after 30 seconds. The reasoning in the
docs is sound — nodes recover, and reassigning while the pods are alive makes the stored
assignment diverge from where the pods actually run, which corrupts per-node capacity
accounting.

And the limitation that matters most for this lesson: **hot swap can only work if there is a
spare node meeting the same constraints.** A workload that requires and fills an entire rack has,
by construction, no replacement inside that rack. Kueue's docs say so explicitly and recommend
running with `TASFailedNodeReplacementFailFast` and/or `waitForPodsReady.recoveryTimeout` so the
workload gets evicted and requeued promptly instead of waiting forever for a node that cannot
exist. Hot swap also only handles **one** node failure at a time; concurrent failures evict.

### 14. The `required`-vs-`preferred` decision, as a break-even

The trade is "pay in wait time or pay in throughput", and it has an exact form. Let:

- `W` = expected queue wait, in hours, to get a clean domain at the required level
- `H` = run length in GPU-hours in the good placement
- `slowdown` = the factor from §5 for the placement you would otherwise get
- `n` = GPUs in the gang

`required` is worth it when the wall-clock saved exceeds the wall-clock waited:

```
  wall-clock(required)  = W + H/n
  wall-clock(preferred) = H/n × slowdown

  required wins  ⟺  W + H/n  <  (H/n) × slowdown
                 ⟺  W        <  (H/n) × (slowdown − 1)
```

And in GPU-hours (the number your showback report bills):

```
  cost(required)  = n × W × idle_rate?   ← usually ZERO: a waiting Kueue Workload
                                            holds no pods and no GPUs (lesson 2 §9)
  cost(preferred) = H × slowdown
  ⇒ required is cheaper in GPU-hours whenever slowdown > 1, ALWAYS.
    The only thing `preferred` buys is earlier completion, and it only buys that
    when W > (H/n)(slowdown − 1).
```

That second block is the sharpest point in the lesson and it is easy to miss: **because a Kueue
Workload waiting for capacity holds nothing, waiting is free in GPU-hours.** Unlike a
coscheduling gang parked at `Permit` — which holds physical GPUs while it waits, lesson 2 §7(d) —
a pending Kueue Workload costs you only *latency*, not *money*. So the `required`-vs-`preferred`
decision is not a cost trade at all under Kueue; it is a deadline trade.

**Worked instance.** 8-GPU job, `H` = 2,688 GPU-hours (14 days), `slowdown` = 4.78 (the 100 GbE
case from §5):

```
  (H/n)(slowdown − 1) = (2688/8) × 3.78 = 336 h × 3.78 = 1,270 hours = 53 days
  ⇒ required wins on wall-clock unless the queue wait exceeds 53 DAYS.
```

For a 2-hour hyperparameter sweep trial with `slowdown` = 1.42 (spine contention):

```
  (H/n)(slowdown − 1) = 2 h × 0.42 = 0.84 hours = 50 minutes
  ⇒ required wins only if a clean rack appears within 50 minutes. On a busy fleet it
    often will not, and `preferred` is correct.
```

**The rule that falls out: long comm-bound runs → `required`; short or comm-tolerant jobs →
`preferred`.** And you can now say *why*, with the crossover point computed from two measurable
numbers rather than asserted.

### 15. Failure modes, mechanically

- **Label drift.** A node missing one level's label is invisible to TAS at that level. Capacity
  shrinks silently — no event, no condition, no alert. The symptom is a `required` workload
  pending on a cluster that looks empty. Diagnose with
  `kubectl get nodes -L cloud.provider.com/topology-block,cloud.provider.com/topology-rack` and
  look for blanks. Treat full label coverage as a cluster invariant you validate on every node-pool
  onboarding, not a one-time install step.
- **Missing flavor tolerations.** Covered in §7. Same symptom as label drift, different cause:
  TAS excluded tainted nodes from its capacity view. The `excluded:` clause in the not-fit message
  is your tell.
- **`required` starvation.** An aggressive `required` at a fine level on a fragmented fleet leaves
  a job pending indefinitely while GPUs sit idle in scattered domains. TAS will not defragment for
  you — it waits. That is the topology instance of the fragmentation problem lesson 7 quantifies
  and KAI's `consolidate` action (lesson 5) attacks.
- **Constrained the wrong level.** `required: topology-block` on a job that needed rack locality
  validates fine, admits, and still lets the gang spread across racks inside the block. You get
  §5's tax while believing you fixed it. On GB200 NVL72 the same error runs the other way: the
  NVLink domain is now the rack, so `required: hostname` over-constrains and starves you for no
  bandwidth benefit.
- **Annotation on the wrong object.** Put on the Job's `metadata.annotations` instead of
  `spec.template.metadata.annotations`, the annotation is simply ignored. No error. The job runs,
  scattered.
- **`FailedAfterNomination` churn.** On pre-v0.17.6/v0.18.2/v0.19 releases with many
  ClusterQueues, TAS workloads can bounce between nominated and failed for hours (§10). Known
  bug, known fix, check your patch version before debugging it as a mystery.
- **Trusting provider labels blindly.** Not every SKU exposes fine-grained topology labels, and
  those that do are not always accurate for every instance shape. Verify against a real
  measurement — run `all_reduce_perf` across two nodes the labels claim are in the same rack and
  see whether you get intra-rack bandwidth — before you build a `required` policy on it.

## Perspectives

**Developer.** A researcher who annotates `required` at rack level and watches the job sit
`Pending` will file it as a bug. It is not: the constraint is refusing a placement that would
have cost 53 days of wall-clock. The platform team's job is to make that legible where the
researcher looks — surface the Workload's `QuotaReserved` message
(`topology "default" allows to fit only 4 out of 10 pod(s)`) in whatever UI or `kubectl` wrapper
the team uses, so "waiting for a clean rack" is distinguishable from "the scheduler is broken".
The developer-side skill is knowing which of their jobs are comm-bound enough to want the
constraint at all, which is a profiler question, not a scheduler question.

**Operator.** Topology label hygiene is a continuous day-2 responsibility with no built-in alarm.
Two invariants are worth encoding as cluster checks: (a) every node carrying the flavor's
`nodeLabels` also carries every level of the referenced `Topology`, and (b) every taint present
on TAS nodes appears in the flavor's `tolerations`. Both fail silently and both present as
"capacity vanished". The operator also owns the feature-gate posture — the hot-swap quartet in
§13, and whether `TASBalancedPlacement` is worth enabling for all-to-all workloads.

**Hardware / network.** TAS levels are a *declaration*; the fabric is the truth. The declaration
is only useful if the labels map to real switch boundaries, and the value of the constraint is
set entirely by the ratio in §2 — 9× on rail-optimised NDR, 288× on a single 100 GbE NIC. This
is also where the ground is moving: GB200 NVL72 makes the NVLink domain a 72-GPU *rack*, which
relocates the boundary that matters from the node to the rack, and therefore changes which
`Topology` level a job should constrain for the same physical guarantee. A topology policy
written for HGX H100 and applied unchanged to NVL72 is wrong in a way that looks fine.

**Economics.** Under Kueue, waiting is free in GPU-hours and spreading is not (§14). That
asymmetry is the FinOps argument, and it is stronger than the usual "wait vs run slow" framing
because it is not a trade at all in cost terms — only in deadline terms. The showback report this
module builds should therefore carry topology as a dimension: GPU-hours billed to a queue that
ran split across domains are GPU-hours that bought less compute than the same hours run packed,
and that is a per-queue efficiency number worth publishing.

## Real-world use cases

- **Meta — "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at
  Hyperscale" (OSDI 2024)** — https://www.usenix.org/system/files/osdi24-choudhury.pdf. What it
  shows: scheduling LLaMA3 training across **16,000 H100 GPUs** with explicit topological
  constraints — the paper describes allocating in multiples of **two hosts per rack and 128 hosts
  per data center** — and, combined with their other techniques, bringing the most-overloaded
  region's GPU demand-to-supply ratio from **2.63 down to 0.98**. Note what the two-level
  constraint is doing: it is exactly the multi-layer slice shape Kueue expresses with
  `podset-slice-required-topology-constraints`, three years before Kueue shipped it. A ratio above
  1.0 means more high-priority work wants a region than the region can hold; the fix was placement
  policy across the existing footprint, not more hardware. *(Dated snapshot of one 2024 training
  run, not a general benchmark. Direct fetch of usenix.org was blocked by this environment's
  egress proxy; cited by canonical URL and cross-checked against the paper's USENIX listing.)*

- **Kueue KEP-2724, "Topology Aware Scheduling"** — read directly from
  `keps/2724-topology-aware-scheduling/README.md` in the cloned repo this session. Substance
  beyond the algorithm matrix: the motivating field complaint ("Running a workload with Pods
  scattered across a data center results in longer runtimes, and thus costs"), the 3/3/2/1
  worked example reproduced in §10, and the `FailedAfterNomination` incident report — TAS
  assignment conflicts within a scheduling cycle caused **outages** in clusters with many
  ClusterQueues, with workloads suspended for hours, fixed in v0.19.0 and cherry-picked back to
  v0.17.6 and v0.18.2. That last item is the kind of detail that separates "I read the docs" from
  "I run this".

- **Kueue's own TAS setup guide and example manifests** — `site/content/en/docs/tasks/manage/`
  and `site/static/examples/tas/` in the cloned repo. Substance: the 8-worker / 2-block / 4-rack
  kind topology the Practice section below reproduces, the exact not-fit condition message, the
  prefix-compressed `topologyAssignment` output decoded in §11, and the GPU flavor example whose
  comment records the toleration trap from §7 — *"Most cloud providers auto inject the toleration
  to the pods based on the nodeSelector via a webhook. However, TAS isn't aware of the webhook,
  so you need to manually add the toleration here."*

- **NVIDIA GB200 NVL72** — https://www.nvidia.com/en-us/data-center/gb200-nvl72/. What it shows:
  72 Blackwell GPUs in a **single NVLink domain**, 1.8 TB/s bidirectional NVLink per GPU, 130 TB/s
  aggregate across the domain. Why it belongs in a scheduling lesson: it moves the boundary that
  topology constraints exist to respect from the node to the rack, which changes which `Topology`
  level a given job should name. *(Search-verified this session; direct fetch of nvidia.com was
  blocked by the egress proxy.)*

- **NVIDIA — "Deploying Disaggregated LLM Inference Workloads on Kubernetes"** —
  https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/.
  What it shows: topology-aware placement applied to *inference*, not training — prefill/decode
  disaggregation needs specific co-location between the two roles, expressed through Grove and
  KAI. The bridge to note: this is the `podset-group-name` shape of the problem (two PodSets, one
  domain), which is why Kueue has that annotation at all. *(Search-verified; fetch blocked by
  egress this session.)*

## Worked example

An 8-worker synchronous data-parallel H100 training job. The `training` ClusterQueue has 64 GPUs
across 4 racks of 2× 8-GPU nodes. Fabric: DGX-class nodes, but the cluster's compute network is
**1× 100 GbE per node** — the cost-optimised case from §2, chosen because it is the one where
topology actually decides the outcome.

**Step 1 — establish the two numbers.** From a profiler run on a single node:

```
  model             7B params, bf16 gradients   ⇒ S = 14 GB all-reduced per step
  T_comp            250 ms/step  (fwd + bwd, measured)
  φ (overlap)       0.7          (DDP bucket overlap, measured as
                                  1 − exposed_comm/total_comm on the good placement)
  B_intra           400 GB/s busbw   (all_reduce_perf, 8 GPUs, ≥1 GiB msg — representative)
  B_cut (4+4)       12.5 GB/s        (one 100 GbE NIC per node)
```

**Step 2 — compute the tax.**

```
  packed, 8 GPUs on one node:
     T_comm = 2(N−1)/N × S / busbw = 1.75 × 14 GB / 400 GB/s   = 61 ms
     overlap budget = 0.7 × 250 ms                             = 175 ms
     exposed comm   = max(0, 61 − 175)                         = 0 ms
     step                                                      = 250 ms

  split 4+4 across two nodes:
     T_cross ≥ S / B_cut = 14 GB / 12.5 GB/s                   = 1,120 ms
     exposed comm = 1,120 − 175                                = 945 ms
     step                                                      = 1,195 ms

  slowdown = 1,195 / 250                                       = 4.78×
```

**Step 3 — price it over the run.**

```
  planned run: 8 GPUs × 14 days = 2,688 GPU-hours in the packed placement
  split placement: 2,688 × 4.78                    = 12,849 GPU-hours
  waste:           12,849 − 2,688                  = 10,161 GPU-hours
  cost @ $2.35/GPU-hr (neocloud on-demand snapshot) = $23,878
  schedule slip:   14 days → 67 days
```

**Step 4 — express the fix as a constraint.** Because the gang fits in one node, constrain at the
host level:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: train-7b-
  labels:
    kueue.x-k8s.io/queue-name: tas-user-queue
spec:
  parallelism: 8
  completions: 8
  completionMode: Indexed
  template:
    metadata:
      annotations:
        # NOTE: on the POD TEMPLATE, not on the Job's own metadata.
        kueue.x-k8s.io/podset-required-topology: "kubernetes.io/hostname"
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: registry.k8s.io/e2e-test-images/agnhost:2.53
          args: ["pause"]
          resources:
            limits:
              nvidia.com/gpu: "1"
```

**Step 5 — read the outcomes.** Two cases, two observable signals.

*Case A — a whole node is free.* Kueue admits, and the assignment names one host:

```bash
$ kubectl get workload -o name --sort-by='.metadata.creationTimestamp' | tail -1
workload.kueue.x-k8s.io/job-train-7b-x9k2p-4f81c

$ kubectl get workload.kueue.x-k8s.io/job-train-7b-x9k2p-4f81c \
    -o jsonpath='{.status.admission.podSetAssignments[0].topologyAssignment}' | jq -c
{"levels":["kubernetes.io/hostname"],
 "slices":[{"domainCount":1,"podCounts":{"individual":[8]},
            "valuesPerLevel":[{"individual":{"prefix":"gpu-node-","roots":["07"]}}]}]}

$ kubectl get pods -l batch.kubernetes.io/job-name=train-7b-x9k2p -o wide \
    | awk '{print $1, $3, $7}'
NAME                   STATUS   NODE
train-7b-x9k2p-0       Running  gpu-node-07
train-7b-x9k2p-1       Running  gpu-node-07
...                             (all eight on gpu-node-07)
```

`domainCount: 1` and a single value in `roots` is the proof. Every collective link is NVLink.

*Case B — no whole node is free.* The Workload stays pending, and the reason names TAS, not quota:

```bash
$ kubectl get workload.kueue.x-k8s.io/job-train-7b-b2m4q-9d13a \
    -o jsonpath='{.status.conditions[?(@.type=="QuotaReserved")].message}'
couldn't assign flavors to pod set main: topology "default" doesn't allow to fit any of 8 pod(s).
Total nodes: 8; excluded: resource "nvidia.com/gpu": 5
```

*(Representative transcript — the message is assembled by `notFitMessage` in
`pkg/cache/scheduler/tas_flavor_snapshot.go`; the exact counts depend on your fleet, the wording
does not.)* Read it: the topology could not fit **any** of the eight pods in a single host domain,
eight nodes were considered, five were excluded for lack of GPU. Quota was never the binding
constraint. Under §14's break-even this job should wait — the crossover is 53 days.

**Step 6 — the counterfactual, to keep yourself honest.** Re-run the same arithmetic for a
rail-optimised NDR fabric, `B_cut = 4 × 50 = 200 GB/s`:

```
  T_cross ≥ 14/200 = 70 ms ;  exposed = max(0, 70 − 175) = 0 ms ;  step = 250 ms
  slowdown = 1.00×   ⇒  the `required` constraint buys NOTHING and costs queue latency.
```

Same job, same annotation, opposite verdict — because the fabric changed. That is the sentence
to leave an interviewer with: **the topology policy is derived from the fabric's cut bandwidth
and the model's compute-to-comm ratio, not inherited from a blog post.**

## Practice

A TAS co-placement demo on kind. No real GPUs needed — TAS reasons over labels and resource
counts, not hardware. This feeds the deliverable's topology artifact; see
[Kueue setup + per-queue showback](../practice/kueue-showback/README.md).

1. **Create an 8-worker kind cluster labelled as 2 blocks × 2 racks each.** Put the labels in the
   kind config so they survive node recreation:

   ```yaml
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
     - role: control-plane
     - role: worker
       labels: {cloud.provider.com/node-group: tas-group,
                cloud.provider.com/topology-block: b1, cloud.provider.com/topology-rack: r1}
     - role: worker
       labels: {cloud.provider.com/node-group: tas-group,
                cloud.provider.com/topology-block: b1, cloud.provider.com/topology-rack: r1}
     - role: worker
       labels: {cloud.provider.com/node-group: tas-group,
                cloud.provider.com/topology-block: b1, cloud.provider.com/topology-rack: r2}
     # ... repeat to 8 workers: b1/r2, b2/r3, b2/r3, b2/r4, b2/r4
   ```

   Verify with
   `kubectl get nodes -L cloud.provider.com/topology-block,cloud.provider.com/topology-rack`.
   Every worker must show both labels — a blank is the §15 failure mode, injected on purpose
   later.

2. **Install Kueue** and confirm the gate is on. `TopologyAwareScheduling` has been beta and
   default-on since v0.14, so on a current release there is nothing to enable; confirm rather than
   assume, and record the version you ran.

3. **Apply the `Topology`, `ResourceFlavor` and TAS-enabled `ClusterQueue`/`LocalQueue`** from §7.
   Use CPU as the scarce resource (kind nodes have real CPU and no GPUs) or advertise fake
   `nvidia.com/gpu` via the node-status patch from lesson 1. Size the quota deliberately high so
   that **quota is never the binding constraint** — this is what makes any rejection provably a
   TAS decision.

4. **Submit a `required` job at rack level** sized so exactly one rack (two nodes) can hold it.
   Confirm co-placement two ways, not one:
   - `kubectl get pods -o wide` — every pod on nodes sharing one rack label;
   - the Workload's `status.admission.podSetAssignments[0].topologyAssignment` — decode
     `domainCount`, `podCounts`, and the prefix-compressed `valuesPerLevel` as in §11.

5. **Force the `required` refusal.** With the first job running, submit a second identical
   `required` job. It must stay pending. Capture the `QuotaReserved` condition message and confirm
   it names the topology and the excluded-node counts. **This capture is the artifact that proves
   TAS, not quota, is doing admission control** — keep it.

6. **Relax to `preferred` and resubmit.** Same cluster state. The job should now admit, distributed
   across racks. Diff the two `topologyAssignment` blocks: `domainCount` goes from 1 to >1. Write
   one sentence explaining why `required` waited and `preferred` spread.

7. **Try `unconstrained`.** Submit a job with `podset-unconstrained-topology: "true"` and watch it
   fill the scattered gaps left by the previous two. Note which nodes it chose and check the
   choice against §10's `LeastFreeCapacity` ordering — did it consume the tightest domains first
   and leave the roomiest one whole?

8. **Break it on purpose.** Remove the `topology-rack` label from one worker
   (`kubectl label node <n> cloud.provider.com/topology-rack-`) and resubmit the `required` job.
   Observe that capacity shrank with no event and no warning. Restore the label. This 60-second
   exercise is what makes §15's first pitfall stick.

9. **Stretch — placement algorithms.** If your version supports it, toggle `TASBalancedPlacement`
   and submit a `preferred` job against an asymmetric free-capacity layout. Compare the resulting
   `podCounts` against the greedy run: you are looking for `[6,6]` where the default gave
   `[10,2]`.

**Acceptance:** a committed TAS demo in the deliverable containing (a) the `Topology` +
`ResourceFlavor` + `ClusterQueue` manifests and the kind config with labels, (b) `get pods -o wide`
plus the decoded `topologyAssignment` for the `required` run showing one domain, (c) the pending
Workload's TAS refusal message from step 5, (d) the `preferred` run showing `domainCount > 1`,
and (e) two or three sentences applying §5's tax formula to your own numbers — your fabric's
`B_cut`, your model's `S` and `T_comp`, the resulting slowdown, and the §14 break-even. A reviewer
should be able to rerun the commands on a fresh kind cluster and reproduce every capture.

## Common pitfalls

- **Reading a pending `required` job as a bug.** It is the constraint refusing a placement that
  §5 says would cost multiples of the run. The mechanism: TAS evaluated every domain at the named
  level and none had capacity for the whole PodSet, so the Workload never got `QuotaReserved`.
  Fix by draining a domain, relaxing to `preferred`, or accepting the wait — after checking §14's
  break-even, not by assuming TAS is broken.

- **Putting the annotation on the Job instead of the Pod template.** The annotation lives at
  `spec.template.metadata.annotations`. On the Job's own `metadata.annotations` it is silently
  ignored: no validation error, no event, no condition. The job admits and runs scattered, and
  the only tell is that `topologyAssignment` is absent from the Workload status. Check that field
  before believing TAS is engaged.

- **Forgetting `tolerations` on the ResourceFlavor.** Cloud providers inject GPU tolerations into
  pods with a webhook; TAS computes capacity itself and never sees that webhook, so tainted GPU
  nodes are excluded from its view. Symptom: a not-fit message with a large `excluded:` count on a
  cluster full of idle GPUs. Mechanism: capacity is computed in §8's formula over nodes the flavor
  can actually tolerate.

- **Constraining the wrong level.** `required: topology-block` on a job that needed rack locality
  passes validation, admits, and still spreads across racks inside the block. You pay §5's tax
  while believing it is fixed. The inverse error appears on GB200 NVL72, where the NVLink domain
  is a 72-GPU rack and `required: hostname` over-constrains for no bandwidth gain. Match the level
  to the physical boundary that changes bandwidth, and re-check it every hardware generation.

- **Letting topology labels drift.** A new node pool missing one level's label silently shrinks
  TAS's capacity view — no alert, because a missing label is not an error, it is just a node TAS
  cannot place in the tree. Validate full label coverage on every node-pool onboarding as a
  cluster invariant.

- **Assuming `required` will defragment for you.** It will not. TAS waits; it never moves running
  work. Defragmentation is a separate capability (KAI's `consolidate`, lesson 5) or a manual
  operation you price (lesson 7). A `required` job on a fragmented fleet can pend indefinitely
  while the fleet-wide free-GPU count looks healthy — which is precisely the effective-capacity
  gap lesson 7 measures.

- **Quoting "2–5× slower" without naming the fabric.** §4's table shows the same 4+4 split costing
  1.15× on rail-optimised NDR and 18× on a single 100 GbE NIC, at the collective level. The
  wall-clock multiplier then depends on your compute-to-comm ratio. Give the formula and your two
  inputs; a bare multiplier is someone else's cluster.

## Self-check

- **Why does a topology-spread all-reduce gang waste GPU-hours, and how much?** *Answer:*
  Synchronous training does an all-reduce every step, and the collective cannot complete faster
  than its slowest path. The floor is information-theoretic: partition the ranks into two groups
  and at least `S` bytes must cross the boundary in each direction, so `T_cross ≥ S / B_cut`. On
  8×H100 in one NVSwitch domain, a 14 GB gradient all-reduce takes `1.75 × 14/400 ≈ 61 ms`; split
  4+4 over one 100 GbE NIC per node it takes at least `14/12.5 = 1,120 ms`, an 18× collective
  slowdown. Wall-clock is damped by overlap: with `T_comp = 250 ms` and 70% overlap the exposed
  comm goes from 0 ms to 945 ms, so step time goes 250 → 1,195 ms, a **4.78×** slowdown, and a
  2,688 GPU-hour run becomes 12,849 GPU-hours — 10,161 wasted, ≈$23.9k at a $2.35/GPU-hr
  snapshot. On a rail-optimised NDR fabric (`B_cut = 200 GB/s`) the same split costs 1.00×. **The
  multiplier is a property of the fabric, not a constant.**

- **`required` vs `preferred` — behaviour, and when is each right?** *Answer:* `required` is a
  hard constraint: the whole PodSet must fit in one domain at the named level or the Workload is
  not admitted — no fallback, no spread, it waits. `preferred` starts at the named level and, if
  no single domain fits, evaluates each coarser level in turn; if it does not fit even at the top
  level, the Workload is admitted distributed across multiple domains. Since a waiting Kueue
  Workload holds **zero pods and zero GPUs**, `required` is always cheaper in GPU-hours whenever
  the alternative placement is slower at all; the only thing `preferred` buys is earlier
  completion. The break-even is `W < (H/n)(slowdown − 1)`: for a 14-day 8-GPU run at 4.78× that
  is 53 days of tolerable queue wait; for a 2-hour sweep trial at 1.42× it is 50 minutes. Long
  comm-bound runs → `required`; short or comm-tolerant work → `preferred`; and
  `unconstrained` when you want TAS's gap-filling with no locality requirement at all.

- **How do TAS and gang scheduling interact?** *Answer:* Conjunctively, at different layers.
  Kueue admits a Workload only when quota is available **and** TAS can assign the *entire* PodSet
  to a domain satisfying its annotation; Kueue does not admit fractions of a Workload unless
  partial admission is explicitly enabled. Whatever enforces gang at the pod layer — coscheduling,
  Volcano, or the native `Gang{minCount}` path — then operates on pods that already carry a node
  selector confining them to TAS's chosen domain. Each alone fails: gang alone satisfies "all 8
  pods" with a 4+4 spine split; TAS alone (without atomic admission) places some pods well and
  leaves the rest pending, which is lesson 1's deadlock. Together the only admissible outcome for
  tightly-coupled training is all pods, one domain, or wait holding nothing.

- **Walk the TAS placement algorithm, and say which strategy runs when.** *Answer:* Two phases.
  Phase 1 is a bottom-up roll-up: compute how many pods of this PodSet fit in each leaf domain —
  allocatable of Ready and schedulable nodes, minus other admitted TAS workloads, minus all
  non-TAS pods — then sum those counts up the tree. Phase 2 is top-down: at the annotated level
  (searching upward for `preferred`), sort the domains, select the minimal set that fits, descend
  into it, repeat, and emit `(domain, podCount)` pairs at the lowest level. Three sort strategies:
  **BestFit** sorts descending by free capacity but replaces the last chosen domain with the
  *smallest* one that still fits; **LeastFreeCapacity** sorts ascending and takes the first that
  fits; **BalancedPlacement** picks the optimal domain set first, then distributes evenly.
  Since v0.15 the default (`TASProfileMixed`, beta and on) is BestFit for `required` and
  `preferred`, **LeastFreeCapacity for `unconstrained`** — the KEP's stated reason is to
  prioritise minimising fragmentation. `TASBalancedPlacement` (alpha, off) swaps `preferred` to
  BalancedPlacement, which matters for all-to-all patterns where `[6,6]` beats `[10,2]`.

- **How is a TAS assignment actually enforced, given that Kueue never binds a pod?** *Answer:*
  With a scheduling gate and node-selector injection. When Kueue unsuspends the Job, each pod is
  created carrying the `kueue.x-k8s.io/topology` scheduling gate, which makes it ineligible for
  scheduling — `kube-scheduler` will not even consider it. The `topology_ungater` controller then
  reads `status.admission.podSetAssignments[].topologyAssignment` from the Workload, injects a
  node selector matching the assigned domains into each pod, and removes the gate. Only then does
  `kube-scheduler` run its normal cycle, and its `Filter` now accepts only nodes in the assigned
  domain. The gate exists to close the race in which pods would otherwise be scheduled before the
  selector was written. The assignment is inspectable, with hostnames prefix-compressed
  (`prefix: "kind-worker"`, `roots: ["", "2"]` → `kind-worker`, `kind-worker2`) since v0.19 to keep
  large assignments small in etcd.

- **A node running a TAS workload goes NotReady. What happens, and when does it not work?**
  *Answer:* Because TAS pinned the workload to named nodes, it cannot just be rescheduled
  anywhere, so Kueue tries **hot swap**: find a replacement node meeting the same constraints and
  patch only that entry of the assignment, leaving the rest intact. Gated by
  `TASFailedNodeReplacement` (beta, on since v0.14). Since v0.19 the default trigger is *pod
  termination*, not elapsed time: a NotReady node whose pods are still running is not treated as
  failed, because nodes recover and reassigning under live pods corrupts capacity accounting
  (the old 30-second rule lives on behind the deprecated
  `TASReplaceNodeDueToNotReadyOverFixedTime`). Untolerated `NoExecute`/`NoSchedule` taints also
  count as failure with `TASReplaceNodeOnNodeTaints` (beta since v0.17). It does not work when
  there is no eligible spare — most obviously when the workload requires and fills a whole rack —
  and it handles only one failure at a time. For those cases run
  `TASFailedNodeReplacementFailFast` and/or `waitForPodsReady.recoveryTimeout` so the workload is
  evicted and requeued promptly instead of waiting for a node that cannot exist.

- **Your `required` workload is pending on a cluster with plenty of idle GPUs. Give three
  distinct causes and the signal that distinguishes them.** *Answer:* (1) **Missing topology
  labels** — a node lacking one level's label is not in the tree at that level; check
  `kubectl get nodes -L <block>,<rack>` for blanks. (2) **Missing flavor tolerations** — TAS
  computes capacity itself and does not see the provider's toleration-injecting webhook, so
  tainted GPU nodes are excluded; the tell is a large `excluded:` count in the not-fit message
  while `kubectl describe node` shows a taint the flavor does not list. (3) **Genuine
  fragmentation** — the free GPUs exist but no *single* domain at the required level holds the
  whole PodSet; the not-fit message reports a nonzero fit count below the request, e.g.
  `allows to fit only 4 out of 10 pod(s)`. A fourth, version-specific cause on older patch
  releases: `FailedAfterNomination` churn from intra-cycle TAS conflicts, fixed in v0.19.0 and
  backported to v0.17.6 / v0.18.2.

## Connections & what's next

This lesson supplies the "where" that completes the module's "whether" (lessons 3–4, quota and
admission) and "all-or-nothing" (lesson 2, gang; lesson 5, which scheduler owns it). It reaches
back into module 02b for the interconnect physics that motivates the whole thing, and forward in
two directions. First, `required` can refuse to admit a gang on a fragmented cluster even when
the fleet-wide GPU count looks ample — that is exactly the allocated-versus-usable gap lesson 7
teaches you to measure and price, and TAS's `LeastFreeCapacity` default is Kueue pre-empting that
gap with a placement policy. Second, hot swap and `waitForPodsReady.recoveryTimeout` are the
first appearance of a theme lesson 8 makes central: a workload can lose its placement mid-run, and
whether that is a shrug or a catastrophe is decided by whether it checkpoints.

**Next: [07 — Fragmentation & effective capacity](07-fragmentation-effective-capacity.md)**, which
generalises this lesson's thesis. Topology-blind placement wastes GPU-hours; fragmentation is the
broader statement that *allocated capacity is not usable capacity*, and you will build the
calculator that turns the gap into a defensible dollar figure on a 128-GPU fleet.

## References & further reading

**Primary sources — read directly from the cloned `kubernetes-sigs/kueue` tree this session**

Note on method: this environment's egress proxy blocks `kueue.sigs.k8s.io`, `kubernetes.io`,
`nvidia.com` and several other vendor documentation domains. Rather than cite pages that could
not be reached, the version-specific claims above were verified against the upstream *source
tree and its in-repo documentation*, cloned during this session at the post-v0.19.1 development
head. Where a canonical doc URL is given for convenience, its reachability is stated honestly.

1. **`kubernetes-sigs/kueue` — `apis/kueue/v1beta2/topology_types.go`** —
   https://github.com/kubernetes-sigs/kueue. The authoritative list of TAS annotations
   (`podset-required-topology`, `-preferred-topology`, `-unconstrained-topology`,
   `-slice-required-topology`, `-slice-size`, `-slice-required-topology-constraints`,
   `podset-group-name`), the `TopologySchedulingGate` constant, and the `TopologySpec` CEL rules
   reproduced in §6 — 1–16 levels, uniqueness, hostname-must-be-lowest, and the immutability
   condition. **Cloned and read directly this session.** *(This corrects the previous version of
   this lesson, which showed the API as `v1beta1`; `v1beta2` is the storage version at the current
   head, and the module README's `v1beta1` note is now dated.)*

2. **`kubernetes-sigs/kueue` — `pkg/features/kube_features.go`.** The gate lifecycle table behind
   every version claim in §10 and §13: `TopologyAwareScheduling` alpha v0.9 → **beta, default
   true, v0.14**; `TASProfileMixed` alpha v0.10 → **beta, default true, v0.15**;
   `TASBalancedPlacement` **alpha v0.15, default false**; `TASMultiLayerTopology` alpha v0.17 →
   **beta, default true, v0.19**; `TASFailedNodeReplacement` / `-FailFast` /
   `TASReplaceNodeOnPodTermination` beta-on since v0.14; `TASReplaceNodeOnNodeTaints` beta v0.17;
   `TASAssignmentsEncodingByHostnamePrefix` beta v0.19; plus the dependency map requiring
   `TopologyAwareScheduling` for all of them. **Cloned and read directly this session.**

3. **`kubernetes-sigs/kueue` — `pkg/cache/scheduler/tas_flavor_snapshot.go` and
   `tas_balanced_placement.go`.** The algorithm in §10: `findTopologyAssignment`'s two-phase
   walk, `findLevelWithFitDomains`, `sortedDomains` (ascending under `LeastFreeCapacity`,
   descending otherwise), `findBestFitDomainBy`'s tight-fit selection of the last domain,
   `useBestFitAlgorithm` / `useLeastFreeCapacityAlgorithm` (the latter true only when
   `unconstrained && TASProfileMixed`), and `notFitMessage`, which produces the exact refusal
   strings quoted in §11 and the Worked example. **Cloned and read directly this session.**

4. **Kueue KEP-2724, "Topology Aware Scheduling"** —
   https://github.com/kubernetes-sigs/kueue/blob/main/keps/2724-topology-aware-scheduling/README.md.
   Read for the algorithm-selection matrix reproduced in §10, the 3/3/2/1 worked example, the
   balanced-placement rationale, the two-level and multi-level slice designs, and the
   `FailedAfterNomination` incident write-up (TAS assignment conflicts within a scheduling cycle
   causing outages in many-ClusterQueue environments; fixed in v0.19.0, cherry-picked to v0.17.6
   and v0.18.2). **Read from the cloned repo this session.**

5. **Kueue in-repo docs — `site/content/en/docs/concepts/topology_aware_scheduling.md`.** The
   capacity-calculation rule in §8, the node-topology label model, the ClusterAutoscaler /
   ProvisioningRequest three-stage admission flow, the full hot-swap feature-gate interaction
   matrix in §13, the balanced-placement description, the multi-layer topology annotation format,
   and the *Drawbacks* section noting TAS's memory and scheduling-latency cost. **Read from the
   cloned repo this session;** the rendered page at
   https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/ was unreachable from this
   environment.

6. **Kueue in-repo docs and examples —
   `site/content/en/docs/tasks/manage/setup_topology_aware_scheduling.md`,
   `site/content/en/docs/tasks/run/topology_aware_scheduling.md`, and
   `site/static/examples/tas/*.yaml`.** The kind topology the Practice section reproduces, the
   `kubectl` recipes for reading `topologyAssignment`, the prefix-compression note decoded in §11,
   the real not-fit condition message, and `sample-gpu-queues.yaml` with the ResourceFlavor
   toleration comment that §7 turns into a pitfall. **Read from the cloned repo this session.**

**Hardware and collective-communication background**

7. **NVIDIA DGX H100 / HGX H100 platform specifications** —
   https://www.nvidia.com/en-us/data-center/dgx-h100/. Source for §2's node-level figures: 8×
   H100 SXM5 with fourth-generation NVLink at 900 GB/s bidirectional per GPU, four NVSwitches
   providing 7.2 TB/s bidirectional aggregate GPU-to-GPU bandwidth, and eight single-port
   ConnectX-7 400 Gb/s adapters for the compute fabric (plus two dual-port adapters for
   storage/management). *(Search-verified this session across multiple independent restatements;
   direct fetch of nvidia.com was blocked by the egress proxy.)*

8. **NVIDIA GB200 NVL72** — https://www.nvidia.com/en-us/data-center/gb200-nvl72/. 72 Blackwell
   GPUs in a single NVLink domain, fifth-generation NVLink at 1.8 TB/s bidirectional per GPU,
   130 TB/s aggregate across the domain. Read for why the topology *level* a job should constrain
   moves from node to rack on this generation. *(Search-verified; fetch blocked by egress.)*

9. **NVIDIA NCCL — `nccl-tests` (`all_reduce_perf`) and the NCCL documentation on algorithms and
   bandwidth reporting** — https://github.com/NVIDIA/nccl-tests and
   https://github.com/NVIDIA/nccl. Read for the `algbw` / `busbw` distinction used throughout §3
   (`busbw = algbw × 2(N−1)/N`), the ring construction, and the `Ring` / `Tree` / `NVLS` /
   `NVLSTree` / `CollNet` algorithm set. The 370–480 GB/s intra-node busbw range quoted in §2 and
   §5 is a representative measured range for 8×H100 NVSwitch at large message sizes, not a spec
   figure — **run `all_reduce_perf -b 1G -e 8G -f 2 -g 8` on your own hardware before using it in
   a cost model.**

**Real-world engineering accounts**

10. **Meta — "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at
    Hyperscale" (OSDI 2024)** — https://www.usenix.org/system/files/osdi24-choudhury.pdf,
    conference page https://www.usenix.org/conference/osdi24/presentation/choudhury. LLaMA3
    training across 16,000 H100 GPUs with two-level topological constraints (multiples of two
    hosts per rack, 128 hosts per data center), and a most-overloaded-region demand-to-supply
    ratio moving from 2.63 to 0.98. *(Dated snapshot of a specific 2024 run. Direct fetch blocked
    by this environment's egress proxy; content cross-checked via search against the USENIX
    listing.)*

11. **NVIDIA — "Deploying Disaggregated LLM Inference Workloads on Kubernetes"** —
    https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/.
    Topology-aware placement applied to prefill/decode-disaggregated inference via Grove and KAI —
    the co-locate-two-roles shape that Kueue expresses with `podset-group-name`.
    *(Search-verified; fetch blocked by egress this session.)*

**Deeper dives**

12. **Module 02b (this course) — host topology: NVLink domains, NVSwitch, Topology Manager.**
    The physics behind §2's table and the single-node alignment story TAS deliberately does not
    duplicate. Reread its bandwidth hierarchy next to §4's cut bound and the two halves click:
    02b tells you why a link is slow, this lesson tells you how to keep the collective off it.
