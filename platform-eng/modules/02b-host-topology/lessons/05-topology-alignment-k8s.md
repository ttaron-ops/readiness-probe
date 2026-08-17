---
lesson: "02b.5"
title: "Kubernetes topology alignment — Topology, CPU, and Memory Managers"
module: "02b"
concept: "Kubernetes topology alignment"
status: not-started
est_time: "10h"
prev: "04-server-architecture-8gpu.md"
next: "06-storage-nvme.md"
artifacts: []
sources: 13
---
# 02b.5 · Kubernetes topology alignment — Topology, CPU, and Memory Managers
> **Concept.** How Kubernetes aligns a pod's CPUs, memory, and GPU onto one NUMA node — Topology Manager policies (guarantee vs attempt), the required CPU/Memory Manager static policies, the device-plugin `TopologyInfo` trap that silently defeats it, and where Dynamic Resource Allocation (DRA) is taking this next.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 04 established the physical layout of an 8-GPU node: a fixed NVLink/NVSwitch domain, a 4/4 CPU-socket split, and GPU↔NIC pairs rail-aligned to the same PCIe root complex — all of it baked into the board at manufacturing time and unchangeable at runtime. None of that hardware correctness is optional, and none of it is negotiable by software. What's still open, and entirely software's job, is whether the orchestrator sitting on top of that hardware — Kubernetes — actually *respects* the fixed layout when it decides which CPUs, which memory pages, and which GPU a given pod gets. This lesson is the gap between "the node is wired correctly" and "the pod that lands on it runs correctly": the kubelet-level machinery that turns a hardware fact into an admission guarantee, and the one silent way that guarantee breaks. It's the module's anchor lesson because it's where topology fluency becomes a Kubernetes skill, not just a hardware-reading skill — and it directly sets up lesson 06, where the same "is this thing actually co-located with the GPU" question gets asked of storage instead of CPU and memory.

## Why this matters

You already know the earlier lessons' hardware truth: on a two-socket GPU box, a GPU hangs off one socket's PCIe root complex, and a thread pinned to the *wrong* socket pays the cross-socket interconnect tax (Intel UPI / AMD Infinity Fabric) on every host-to-device copy and every memory access the kernel didn't place locally. The arithmetic from lesson 02 says why that hurts: per-socket DDR5 bandwidth on a Sapphire Rapids box is ~307 GB/s (8 channels × DDR5-4800 × 8 B), while a single UPI 2.0 link carries ~48 GB/s per direction. Cross-socket traffic is squeezing through a pipe roughly an order of magnitude narrower than local memory, *and* sharing it with cache-coherence traffic and every other pod's remote accesses. A GPU that wants 63 GB/s of PCIe Gen5 ×16 host-to-device bandwidth cannot get it through that pipe while anything else is running.

The interview question that separates a Senior Platform Engineer from a CKA holder is exactly this: **"How do you guarantee a pod's CPU, memory, and GPU all land on one NUMA node?"** The wrong answer is "node affinity and hope." The right answer is the kubelet's NUMA-alignment layer: the **Topology Manager** coordinating three hint providers (CPU Manager, Memory Manager, Device Manager), gated on the *static* policies, and — the part almost everyone misses — dependent on the GPU device plugin publishing `TopologyInfo`. Get one piece wrong and the policy is enabled, the pods admit, dashboards look green, and every GPU workload is silently cross-socket.

That is exactly the job this module's README names as the differentiator at neocloud operators: CoreWeave's *Sr HW Engineer, GPU & PCIe* posting calls for turning a silent placement fault into a monitored alert, and the interview probe is verbatim "guarantee a pod's CPU+mem+GPU on one NUMA node in k8s." That's the fleet-scale waste story you get paid to prevent: the failure is invisible per-pod and only shows up as a utilization/cost gap across thousands of nodes. **This is the money skill of the module — own it cold.**

## What's new here (calibration)

You run Kubernetes daily, so we skip re-teaching what you already know:

- **Skipped:** requests/limits, QoS classes, `nodeSelector`/taints, and how `kube-scheduler` picks a node — that's the scheduler's job, and you already own it operationally.
- **Skipped:** kernel-level NUMA mechanics, `cgroups`/hugepages/THP fundamentals — covered in the 01b Linux module; this lesson assumes you can read `numactl --hardware` cold and know what `cpuset.cpus` and `cpuset.mems` do.
- **New:** the kubelet runs a *second*, separate control loop after the scheduler — Topology Manager coordinating CPU/Memory/Device Manager hint providers — with its own admission failure mode (`TopologyAffinityError`, not `Pending`).
- **New:** the exact bitmask arithmetic behind the hint-provider merge, read out of `pkg/kubelet/cm/topologymanager/policy.go`, including the two facts that most write-ups get wrong: a provider with no topology information returns a **nil** affinity (a don't-care that is skipped in the AND), and `Preferred` is only true when every non-nil mask in a permutation is *identical*, not merely non-empty after ANDing.
- **New:** the device-plugin `TopologyInfo` trap traced to its actual source lines — in the kubelet's Device Manager (`deviceHasTopologyAlignment`) and in the NVIDIA plugin (`BuildDevice`) — so you can say precisely *which* line makes GPU alignment a no-op.
- **New:** Dynamic Resource Allocation (DRA) — GA in Kubernetes **v1.34** (September 2025) — and its `ResourceSlice`/`ResourceClaim` model, which expresses GPU↔NIC co-location as a first-class scheduler constraint instead of an opt-in kubelet-side hint. Know the trajectory even before your fleet runs it.

## Core concepts

### 1. The problem: two allocators, one machine, no shared view

Start with why this component exists at all, because the design follows directly from the problem.

A Kubernetes node has several independent things that hand out physical hardware to containers:

- **CPU Manager** decides *which logical CPUs* a container is pinned to (it writes `cpuset.cpus`).
- **Memory Manager** decides *which NUMA nodes* a container's pages come from (it writes `cpuset.mems`).
- **Device Manager** decides *which device instances* — which GPU UUIDs, which SR-IOV VFs — a container is handed, based on what device plugins advertised.

Before Kubernetes v1.18 these three made their decisions **completely independently**. Each was locally correct and globally wrong: CPU Manager would pick the 8 free cores it liked best, Device Manager would pick the first free GPU, and nothing forced them to be the same socket. On a single-socket machine that's harmless. On a two-socket GPU node it is a coin flip that costs tens of percent of achievable host↔device bandwidth on every batch.

The Kubernetes project's fix was deliberately *not* "make one component own everything." Rewriting CPU Manager to also know about GPUs would couple them permanently and would not extend to the fourth or fifth resource type. Instead they introduced a **coordinator that owns no resources at all**: the Topology Manager. It asks each allocator "if you had to allocate this container's share of your resource, which NUMA nodes could you draw from, and which of those options would you consider clean?", merges the answers into one decision, publishes that decision, and *then* lets each allocator do its real allocation constrained by it.

That is why the central data type is not an allocation. It is a *hint*.

### 2. The `TopologyHint` — the entire protocol in two fields

From `pkg/kubelet/cm/topologymanager/topology_manager.go`, the type every hint provider speaks:

```go
type TopologyHint struct {
    NUMANodeAffinity bitmask.BitMask   // which NUMA nodes could satisfy this
    Preferred        bool              // is this the narrowest/cleanest option?
}
```

Two fields. That's the whole protocol.

- **`NUMANodeAffinity`** is a bitmask over NUMA node IDs. Bit *n* set means "node *n* participates in this candidate allocation." On a 2-node box, `0b01` = node 0 only, `0b10` = node 1 only, `0b11` = both nodes. A **nil** bitmask is not the same as zero: nil means **"I have no opinion"** — a don't-care.
- **`Preferred`** means "this candidate is the narrowest affinity this provider can achieve for this request." A provider that can satisfy a request entirely from one node marks the single-node candidates preferred and the two-node candidates not-preferred. A provider that *needs* two nodes (say, 40 CPUs on a machine with 32 per node) marks its two-node candidates preferred, because that is the best it can do.

**`Preferred` is a per-provider judgement about that provider's own options, not a global verdict.** This matters enormously: it is why `restricted` can reject a pod that *did* get a single-NUMA-node allocation, and why it can admit a pod that spans two nodes. The policy tests the merged hint's `Preferred` flag, and that flag is a statement about whether every provider got its own best case, not about the number of bits set.

Each provider returns not one hint but a `map[string][]TopologyHint` — a list of candidate hints *per resource name*. CPU Manager returns them under the key `cpu`; Memory Manager under `memory` and `hugepages-2Mi`/`hugepages-1Gi`; Device Manager under `nvidia.com/gpu`, `rdma/hca`, and so on. Each resource's list is an independent axis in the merge.

### 3. Where in the kubelet this runs, and why it is not the scheduler

Sequence matters more than anything else for debugging, so nail it down.

`kube-scheduler` picks a node using node-level `Allocatable`: total CPU, total memory, `nvidia.com/gpu: 8`. **It has no per-NUMA view of anything.** It will cheerfully bind a pod requesting 8 CPUs + 1 GPU to a node whose 8 free CPUs are split 5/3 across sockets. The pod binds, the kubelet picks it up, and only *then* does the kubelet's admission chain run.

Topology Manager implements `lifecycle.PodAdmitHandler`. Its `Admit()` delegates to a *scope* (container or pod), which for each unit:

1. calls `GetTopologyHints()` on every registered hint provider,
2. calls `policy.Merge(hints)` to get `(bestHint, admit)`,
3. if `admit == false`, returns a `TopologyAffinityError` and the pod is done,
4. if `admit == true`, stores `bestHint` for that (podUID, containerName) so providers can read it back via `GetAffinity()`,
5. calls `Allocate()` on every provider, which now does the real allocation constrained by the stored hint.

Because the kubelet already owns the pod by this point, a topology rejection does **not** look like a scheduling failure. It is not `Pending`/`Unschedulable`. The pod's `STATUS` column literally reads `TopologyAffinityError`:

```
$ kubectl get pods
NAME         READY   STATUS                  RESTARTS   AGE
guaranteed   0/1     TopologyAffinityError   0          113s

$ kubectl describe pod guaranteed
...
Warning  TopologyAffinityError  10m  kubelet, dell8  Resources cannot be allocated with Topology locality
```

That message string — `Resources cannot be allocated with Topology locality` — is the literal `Error()` return of the `TopologyAffinityError` type in the kubelet source. Grep for it in logs; it is stable.

The scheduler will **not** retry a `TopologyAffinityError` pod onto a better node — the pod is terminal, and it is the ReplicaSet/Deployment/Job controller that creates a replacement, which may well land on the same node pool and fail identically. That is how a systematic misconfiguration (an oversized pod, or a node whose free CPUs are fragmented across sockets) turns into a create-fail-create loop rather than a clean `Pending`. Kubernetes' own docs name this as a known limitation: *"The scheduler is not topology-aware, so it is possible to be scheduled on a node and then fail on the node due to the Topology Manager."* Out-of-tree NUMA-aware scheduler plugins exist to narrow the gap; DRA (§11) is the structural fix.

```
        WHERE THE ALIGNMENT DECISION ACTUALLY HAPPENS
        (and why a rejection is not a scheduling failure)

  ┌──────────────────────── CONTROL PLANE ────────────────────────┐
  │  kube-scheduler                                               │
  │    sees: node Allocatable = {cpu: 128, memory: 2Ti,           │
  │                              nvidia.com/gpu: 8}               │
  │    does NOT see: per-NUMA free CPUs, GPU→socket mapping       │
  │    verdict: "node gpu-07 has room"  ──▶ binds pod             │
  └───────────────────────────────┬───────────────────────────────┘
                                  │ pod.spec.nodeName = gpu-07
                                  ▼
  ┌──────────────────────── NODE gpu-07 ──────────────────────────┐
  │  kubelet admission chain (PodAdmitHandlers, in order)         │
  │    … eviction, sysctl, appArmor …                             │
  │    ┌──────────────────────────────────────────────────┐       │
  │    │ Topology Manager .Admit()                        │       │
  │    │   scope = container | pod                        │       │
  │    │     1. GetTopologyHints() × 3 providers          │       │
  │    │     2. policy.Merge(hints) → (bestHint, admit)   │       │
  │    │     3. admit==false → TopologyAffinityError ─────┼──▶ POD TERMINAL
  │    │     4. store bestHint for (podUID, container)    │       │  (STATUS column
  │    │     5. Allocate() × 3 providers, using bestHint  │       │   shows the
  │    └──────────────────────────────────────────────────┘       │   error name)
  │                          │                                     │
  │                          ▼                                     │
  │   cpuset.cpus ← CPU Mgr   cpuset.mems ← Memory Mgr             │
  │   /dev/nvidia3 ← Device Mgr (CDI/device node injection)        │
  └───────────────────────────────────────────────────────────────┘

  KEY: nothing above the dashed control-plane boundary knows about NUMA.
       Every alignment fact is discovered *after* the placement decision.
```

### 4. The merge, step by step, from the actual algorithm

This is the part worth being able to execute on paper. Four stages.

**Stage A — `filterProvidersHints`: normalise the inputs.**

Every provider's `map[string][]TopologyHint` is flattened into a single `[][]TopologyHint` — one inner slice per *resource*, not per provider. Three special cases, straight from the source:

| Provider returned | Substituted with | Meaning |
|---|---|---|
| `nil` map / empty map (provider has nothing to say at all) | `[]TopologyHint{{nil, true}}` | one don't-care candidate |
| `hints[resource] == nil` (this resource has no topology info) | `[]TopologyHint{{nil, true}}` | one don't-care candidate |
| `hints[resource] == []` (empty slice — resource *has* topology, but no allocation is possible) | `[]TopologyHint{{nil, false}}` | one candidate that is **not** preferred |

The distinction between the last two rows is the single most important asymmetry in the whole system. **`nil` means "I don't care" and is preferred. An empty slice means "I care and I cannot be satisfied" and is not preferred.** A resource with no topology metadata gets the first treatment; a resource that is genuinely out of capacity gets the second.

**Stage B — policy-specific pre-filter.** Only `single-numa-node` does anything here. `filterSingleNumaHints` walks every candidate and keeps only those that are either `{nil, Preferred: true}` (a don't-care) or `{mask with exactly one bit set, Preferred: true}`. Everything else is dropped. If that leaves a resource's candidate list *empty*, the cross-product in stage C produces zero permutations for that resource — and therefore zero permutations overall.

**Stage C — `iterateAllProviderTopologyHints`: the cross product.** A recursive walk over `[][]TopologyHint` producing every combination, one candidate per resource. If you have 3 resources with 3, 1 and 2 candidates, that's 3 × 1 × 2 = 6 permutations. This is why the cost is exponential in NUMA count (each provider's candidate list is roughly the power set of NUMA nodes it can use) and why `max-allowable-numa-nodes` exists as a guard.

**Stage D — `mergePermutation`: collapse each permutation to one hint.** For a permutation of candidate hints:

```go
preferred := true
var numaAffinities []bitmask.BitMask
for _, hint := range permutation {
    if hint.NUMANodeAffinity != nil {          // nil hints are SKIPPED entirely
        numaAffinities = append(numaAffinities, hint.NUMANodeAffinity)
        if !hint.NUMANodeAffinity.IsEqual(numaAffinities[0]) {
            preferred = false                  // masks must be IDENTICAL, not just overlapping
        }
    }
    if !hint.Preferred {
        preferred = false
    }
}
mergedAffinity := bitmask.And(defaultAffinity, numaAffinities...)
return TopologyHint{mergedAffinity, preferred}
```

Read that carefully, because two behaviours fall out of it that most explanations get wrong:

1. **A nil affinity is not an all-ones mask — it is simply absent from the AND.** The effect is the same (identity), but a nil hint also never contributes `Preferred: false`, so it cannot spoil the merged hint. A resource that publishes no topology is *completely invisible* to the merge in both directions.
2. **`Preferred` requires every non-nil mask to be equal to the first one.** Two providers offering `0b01` and `0b11` AND to `0b01` — a perfectly good single-node result — but the merged hint is marked **not preferred**, because the masks disagreed. Under `restricted`, that combination is rejected even though it produced a one-node affinity. This is why `restricted` is not "reject unless single-node"; it is "reject unless every provider independently offered its own best case and they agreed."

**Stage E — `HintMerger.compare`: pick the winner.** Iterating all permutations, keep a running best:

- A `Preferred` candidate always beats a non-preferred one.
- Two preferred candidates: pick the **narrowest** mask (fewest bits). With the `prefer-closest-numa-nodes` policy option and a policy other than `single-numa-node`, pick the one with the shortest NUMA distance instead.
- Two non-preferred candidates: aim for a mask whose bit count is as close to `BestNonPreferredAffinityCount` as possible without exceeding it, where that value is `maxOfMinAffinityCounts` — the largest "narrowest possible mask" across all providers. Intuition: if one provider genuinely needs 2 nodes, there's no point picking a 1-node merged hint that provider can't use, and no point picking 4 nodes either.
- Candidates whose merged affinity has **zero** bits set are never chosen.
- If *no* candidate was ever chosen (zero permutations, or all had empty masks), `Merge()` returns `TopologyHint{defaultAffinity, false}` — the all-nodes mask, explicitly **not preferred**.

That last line is the mechanism behind every rejection. `restricted` and `single-numa-node` both implement `canAdmitPodResult(hint) { return hint.Preferred }`. A merge that fell through to the default returns `Preferred: false`, and the pod is rejected.

**One extra step in `single-numa-node`:** after merging, if the winning affinity happens to equal the machine's full default mask (all NUMA nodes), it is rewritten to `{nil, <same preferred flag>}`. That is the "nothing constrained anything, so there is no affinity to honour" case — and the pod is admitted only if that flag was somehow still true, which happens exactly when every provider was a don't-care.

```
  THE HINT-PROVIDER HANDSHAKE — one container, 2-NUMA node,
  pod requests: cpu "8", memory 32Gi, nvidia.com/gpu 1
  GPU0..3 live on NUMA 0; GPU4..7 on NUMA 1. GPU4 is the only one free.

  time ──▶

  Topology Mgr          CPU Manager         Memory Manager      Device Manager
       │                     │                    │                   │
       │──GetTopologyHints()─▶                    │                   │
       │◀── "cpu": [                              │                   │
       │      {0b01, P=true},   ← 8 free cores on node 0               │
       │      {0b10, P=true},   ← 8 free cores on node 1               │
       │      {0b11, P=false} ] ← 4+4 split, works but ugly            │
       │                     │                    │                   │
       │──GetTopologyHints()──────────────────────▶                   │
       │◀── "memory": [ {0b01,P=true}, {0b10,P=true}, {0b11,P=false} ] │
       │                     │                    │                   │
       │──GetTopologyHints()──────────────────────────────────────────▶
       │◀── "nvidia.com/gpu": [ {0b10, P=true} ]   ← only GPU4 is free │
       │                     │                    │                   │
       │                                                              │
       │ filterProvidersHints → 3 resource axes: 3 × 3 × 1 = 9 perms  │
       │                                                              │
       │  perm  cpu    mem    gpu     AND    all masks equal? P?      │
       │  ────  ────   ────   ────   ─────   ─────────────────────    │
       │   1    0b01   0b01   0b10   0b00    no        → skip (0 bits)│
       │   2    0b01   0b10   0b10   0b00    no        → skip         │
       │   3    0b10   0b01   0b10   0b00    no        → skip         │
       │   4    0b10   0b10   0b10   0b10    YES  P=true  ◀── WINNER  │
       │   5    0b11   0b10   0b10   0b10    no   P=false             │
       │   …                                                          │
       │                                                              │
       │ bestHint = {0b10, Preferred=true}                            │
       │                                                              │
       │ policy.canAdmitPodResult(bestHint):                          │
       │    none         → true    (never even merges)                │
       │    best-effort  → true    (always)                           │
       │    restricted   → true    (hint.Preferred)                   │
       │    single-numa  → true    (hint.Preferred, 1 bit set)        │
       │                                                              │
       │──Allocate(bestHint)─▶ pins CPUs from node 1                  │
       │──Allocate(bestHint)──────▶ cpuset.mems = 1                   │
       │──Allocate(bestHint)───────────────────────▶ hands over GPU4  │
       ▼
  RESULT: CPUs, pages and GPU all on NUMA 1. Zero UPI crossings per batch.
```

Now change one thing — delete the GPU's topology information — and re-run the same handshake:

```
       │──GetTopologyHints()──────────────────────────────────────────▶
       │◀── "nvidia.com/gpu": nil     ← plugin published no TopologyInfo
       │
       │ filterProvidersHints substitutes: [ {nil, P=true} ]
       │
       │  perm  cpu    mem    gpu     AND    all non-nil equal? P?
       │   1    0b01   0b01   nil    0b01    YES  P=true   ◀── WINNER
       │   4    0b10   0b10   nil    0b10    YES  P=true   (tie, narrowest wins by order)
       │
       │ bestHint = {0b01, Preferred=true}   ← single-numa-node ADMITS
       │
       │──Allocate──▶ CPUs on node 0, memory on node 0,
       │              and Device Manager hands over GPU4 — which is on NUMA 1.
       ▼
  RESULT: policy says "aligned", hardware says cross-socket. Nothing errors.
```

### 5. The four policies — guarantee vs attempt, in one line of code each

The policy is a node-wide kubelet setting (`topologyManagerPolicy` in `KubeletConfiguration`, or `--topology-manager-policy`). All four implement the same `Merge()` shape and differ only in `canAdmitPodResult`:

| Policy | Merges hints? | `canAdmitPodResult` | Admission behaviour | What it actually guarantees |
|---|---|---|---|---|
| `none` (default) | **No** — `Merge()` returns an empty hint immediately | `return true` | Always admits. Providers allocate independently. | Nothing. This is pre-1.18 behaviour. |
| `best-effort` | Yes | `return true` | **Always admits**, whatever the merge produced. The hint is still stored, so providers *try* to honour it. | Nothing enforceable. Alignment is attempted and silently abandoned when impossible. |
| `restricted` | Yes | `return hint.Preferred` | Rejects with `TopologyAffinityError` when the merged hint is not `Preferred`. | Every provider got its own narrowest option **and all their masks agreed**. Could still be 2+ NUMA nodes if that is the narrowest possible. |
| `single-numa-node` | Yes, after filtering candidates to single-bit masks and don't-cares | `return hint.Preferred` | Rejects with `TopologyAffinityError` unless a single-node placement exists. | The allocation fits **entirely within one NUMA node**. This is the one the kubelet's own `IsAlignmentGuaranteed()` returns true for. |

Two sentences worth memorising for the interview:

> **`best-effort` prefers alignment and never rejects. `restricted` and `single-numa-node` admit only if the merged hint is `Preferred`, and reject otherwise.** `best-effort` is the "looks configured, isn't enforced" trap — it produces the identical merged hint as `single-numa-node` and then ignores the verdict.

And the sharp distinction between the strict two:

> **`restricted` asks "did everyone get their best case?" `single-numa-node` asks "does it fit on one node?"** On a machine where a pod genuinely needs 2 NUMA nodes' worth of CPU, `restricted` admits it (two nodes was everyone's best case) and `single-numa-node` rejects it.

**Policy options** (`topologyManagerPolicyOptions`, gated by the `TopologyManagerPolicyOptions` feature gate, enabled by default; option groups additionally gated by `TopologyManagerPolicyBetaOptions`, default on, and `TopologyManagerPolicyAlphaOptions`, default off):

| Option | Status | What it does |
|---|---|---|
| `prefer-closest-numa-nodes` | **GA since v1.32**, visible by default | When a merge needs more than one NUMA node, prefer the set with the shortest NUMA distance (the SLIT matrix from `numactl --hardware`) rather than just the smallest node count. Applies to `best-effort` and `restricted` only — `single-numa-node` never uses more than one node, so it ignores the option. |
| `max-allowable-numa-nodes=<int>` | **GA since v1.35**, visible by default | Raises the hard limit on how many NUMA nodes the Topology Manager will run with. **Default 8.** On a machine with more NUMA nodes than the limit, the kubelet errors out and Topology Manager is not loaded at all. The cap exists because the permutation walk is exponential in NUMA node count. Kubernetes explicitly says it has limited data above 8 and does not recommend raising it. |

### 6. Scope: `container` (default) vs `pod`

`topologyManagerScope` picks the unit the merge runs over:

- **`container`** — the default. Each container in the pod is merged and admitted independently. Container A can end up on NUMA 0 and container B on NUMA 1. Correct for a sidecar that genuinely doesn't care; wrong for a trainer plus its data-loader helper that must share the GPU's node.
- **`pod`** — all containers are treated as one allocation. The resource total is computed with the standard effective-request formula: `max( Σ app-container requests , max(init-container requests) )`, per resource. One merge, one verdict, one affinity applied to every container.

`pod` scope is strictly harder to satisfy, so admission fails more often — which is the point. You want a loud failure rather than a pod quietly split across sockets. `pod` + `single-numa-node` is the combination Kubernetes' own documentation calls out as valuable for latency-sensitive and IPC-heavy workloads, and it is the combination to cite for a GPU training pod.

Note the failure-mode difference: under `container` scope, a rejection produces `TopologyAffinityError` with the message `Resources cannot be allocated with Topology locality`. Under pod-level resources with a container scope you can additionally see `PodLevelTopologyAffinityError`, whose message is prefixed with `Pod Scope `. Both report `Type() == "TopologyAffinityError"`, so an alert on the reason string catches both.

### 7. Prerequisite #1 — CPU Manager `static`

Topology Manager is worthless if the providers have nothing real to say, and **by default they don't**.

`cpuManagerPolicy` has exactly two values: `none` (default) and `static`. Under `none`, the kubelet enforces CPU limits with CFS quota, no container is pinned, and CPU Manager's hint is a don't-care. There is nothing to align.

Under `static`, the node's CPUs are split into two pools:

- **Reserved** — taken from `kubeReserved.cpu` + `systemReserved.cpu`, or explicitly from `reservedSystemCPUs` (which takes precedence). Implicit reservations are carved off the shared pool **in integer quantity, in ascending order by physical core ID**. The kubelet refuses to start `static` with a zero CPU reservation, because that would allow the shared pool to empty.
- **Shared** — everything else, minus whatever is currently allocated exclusively. All `BestEffort` and `Burstable` containers run here, *and so do `Guaranteed` containers with fractional CPU requests*.

A container gets **exclusive pinned CPUs** only when both hold:

1. the pod's QoS class is `Guaranteed` (every container sets `requests == limits` for both cpu and memory), and
2. the container's CPU request is a **positive integer**.

"Integer" is tested in the source as `cpuQuantity.Value()*1000 == cpuQuantity.MilliValue()`, so `cpu: "4"` and `cpu: "4000m"` both qualify and `cpu: "3500m"` does not. When exclusive CPUs are granted, CFS quota is *not* applied to that container — its usage is bounded by the cpuset itself.

**Static policy options**, with the versions that matter (`cpuManagerPolicyOptions`, gated by `CPUManagerPolicyOptions`; groups gated by `CPUManagerPolicyBetaOptions` (on) and `CPUManagerPolicyAlphaOptions` (off)):

| Option | Status | What it does, and why you'd set it on a GPU node |
|---|---|---|
| `full-pcpus-only` | GA, visible by default (available since v1.22, GA v1.33) | Allocate whole **physical cores** only. Without it, on an SMT node a "CPU" is a hardware thread and two pods can end up sharing one physical core's execution units while both believe they have exclusive CPUs — classic noisy-neighbour jitter on your GPU feeder threads. With it, a pod whose CPU count isn't a multiple of threads-per-core is **put in `Failed` state with reason `SMTAlignmentError`**, message `SMT Alignment Error: requested N cpus not multiple cpus per core = M` (or `... not enough free physical CPUs: available physical CPUs = X, requested CPUs = N, CPUs per core = M`). |
| `distribute-cpus-across-numa` | beta, visible by default (since v1.23) | When an allocation *must* span NUMA nodes, spread evenly instead of packing node 0 full and spilling. Helps barrier-synchronised parallel code, which runs at the speed of its slowest worker. Additive with `full-pcpus-only` (distributes in whole-core chunks). |
| `align-by-socket` | alpha, **hidden by default** (since v1.25) | Align at the socket boundary rather than the NUMA boundary — relevant on sub-NUMA-clustered CPUs where one socket exposes several NUMA nodes. **Explicitly incompatible with Topology Manager `single-numa-node`**, and a no-op where sockets ≥ NUMA nodes. Do not reach for it on a standard 2-socket/2-node GPU box. |
| `distribute-cpus-across-cores` | alpha, hidden by default (since v1.31) | Spread the container's threads across as many physical cores as possible instead of packing. Cannot be combined with `full-pcpus-only` or `distribute-cpus-across-numa`. |
| `strict-cpu-reservation` | GA, visible by default (since v1.32, GA v1.35) | Without it, `reservedSystemCPUs` only excludes *exclusive* allocations — `BestEffort`/`Burstable` pods can still burst onto the reserved cores and starve the kubelet and container runtime. With it, no workload of any QoS class may run on the reserved set. Enabling it requires deleting `/var/lib/kubelet/cpu_manager_state` and restarting. |
| `prefer-align-cpus-by-uncorecache` | GA, visible by default (since v1.32) | Best-effort: keep a container's CPUs within one uncore/LLC block. Purely additive — if it can't align, the container is still admitted with the packed layout. |

**Changing the policy requires a specific dance**, and skipping it is one of the most common "my node won't come back" incidents:

1. `kubectl drain <node>`
2. stop the kubelet
3. delete `/var/lib/kubelet/cpu_manager_state`
4. edit the kubelet config
5. start the kubelet
6. `kubectl uncordon <node>`

Skip step 3 and the kubelet crash-loops with the exact message:

```
could not restore state from checkpoint: configured policy "static" differs from
state checkpoint policy "none", please drain this node and delete the CPU manager
checkpoint file "/var/lib/kubelet/cpu_manager_state" before restarting Kubelet
```

The same applies if the set of **online** CPUs changes on the node — CPU Manager does not support CPU hotplug, and the state file must be reset by hand.

### 8. Prerequisite #2 — Memory Manager `Static`, and the arithmetic that breaks nodes

`memoryManagerPolicy` takes `None` (default), `Static` (Linux only), or `BestEffort` (Windows only). Memory Manager reached **GA in Kubernetes v1.32**.

Under `None`, Memory Manager returns the default (don't-care) hint for everything and enforces nothing. Your pod can have CPUs and GPU perfectly aligned to NUMA 0 while the kernel serves its pages from NUMA 1 — the first-touch policy from module 01b decides, and nothing in Kubernetes corrects it.

Under `Static`, for `Guaranteed`-QoS pods Memory Manager returns real hints describing which NUMA nodes can satisfy the request (conventional memory *and* each hugepage size, tracked as separate resources), reserves that memory in its internal Node Map, and enforces the decision by writing **`cpuset.mems`** on the container's cgroup. For `Burstable`/`BestEffort` pods it returns the default hint and reserves nothing. It also explicitly *minimises the number of NUMA nodes* a pod's memory is drawn from — and when a request exceeds any single node's capacity, it forms a **NUMA group** spanning several nodes and treats their memory as one pool.

**`reservedMemory` is mandatory under `Static`, and its arithmetic is a global constraint distributed per node:**

```
  Σ over NUMA nodes i of reservedMemory[i]  =  kubeReserved.memory
                                             + systemReserved.memory
                                             + evictionHard["memory.available"]
```

The last term defaults to **`100Mi`** and is the piece everyone forgets. Get the sum wrong and the kubelet **fails at startup** with a memory-reservation validation error. Kubernetes' own worked example:

```yaml
memoryManagerPolicy: Static
kubeReserved:   { cpu: "4", memory: "4Gi" }
systemReserved: { cpu: "1", memory: "1Gi" }
# total to distribute = 4Gi + 1Gi + 100Mi = 5Gi + 100Mi
reservedMemory:
- numaNode: 0
  limits: { memory: "3Gi" }
- numaNode: 1
  limits: { memory: "2148Mi" }     # 2Gi + 100Mi, expressed in Mi
```

Reservations are per memory *type*, so hugepages get their own entries:

```yaml
reservedMemory:
- numaNode: 1
  limits:
    memory:        "512Mi"
    hugepages-1Gi: "2Gi"
```

Configurations the validator rejects outright: duplicate `(numaNode, type)` pairs with different values; a zero limit for any type; a NUMA node ID that doesn't exist on the machine; a type name other than `memory` or `hugepages-<size>` for a size that actually exists.

**Verifying it worked** — `/var/lib/kubelet/memory_manager_state` is the ground truth, and it is worth reading once so its shape is familiar:

```json
{
  "policyName": "Static",
  "machineState": {
    "0": { "numberOfAssignments": 1,
           "memoryMap": { "memory": { "total": 134987354112,
                                      "systemReserved": 3221225472,
                                      "allocatable": 131766128640,
                                      "reserved": 131766128640,
                                      "free": 0 } },
           "nodes": [0, 1] },
    "1": { "numberOfAssignments": 1,
           "memoryMap": { "memory": { "total": 135286722560,
                                      "systemReserved": 2252341248,
                                      "allocatable": 133034381312,
                                      "reserved": 29295144960,
                                      "free": 103739236352 } },
           "nodes": [0, 1] }
  },
  "entries": {
    "fa9bdd38-6df9-4cf9-aa67-8c4814da37a8": {
      "guaranteed": [ { "numaAffinity": [0, 1], "type": "memory", "size": 161061273600 } ]
    }
  },
  "checksum": 4142013182
}
```

Read it: this pod asked for 150 GiB, which did not fit either node alone, so Memory Manager built a **NUMA group** — note `"nodes": [0, 1]` on *both* entries — and pinned the pod's `cpuset.mems` to `0,1`. `"numaAffinity": [0, 1]` on the pod's entry is the confirmation. Under `single-numa-node` this pod would have been rejected instead; under `restricted` it would have been admitted, because two nodes was Memory Manager's own narrowest possible option.

The `memory_manager_state` file has the same checkpoint-mismatch behaviour as the CPU one: delete it as part of any policy change, with the node drained.

### 9. Prerequisite #3 — Device Manager, and THE TRAP

Device Manager is the only hint provider that does not compute topology itself. It reports whatever the **device plugin** told it at registration, over the device-plugin gRPC API. The relevant part of that API is tiny:

```protobuf
message Device {
    string ID = 1;
    string health = 2;
    TopologyInfo topology = 3;      // ← optional
}
message TopologyInfo {
    repeated NUMANode nodes = 1;
}
message NUMANode {
    int64 ID = 1;
}
```

Kubernetes' own documentation states the semantics plainly: *"Setting `TopologyInfo` to `nil` or providing an empty list of NUMA nodes for a given device indicates that the Device Plugin does not have a NUMA affinity preference for that device."* The field is **opt-in on the plugin side**. Nothing forces a plugin to fill it, and nothing warns you when it doesn't.

Here is the kubelet-side consequence, from `pkg/kubelet/cm/devicemanager/topology_hints.go`:

```go
func (m *ManagerImpl) deviceHasTopologyAlignment(resource string) bool {
    // If any device has Topology NUMANodes available, we assume they care about alignment.
    for _, device := range m.allDevices[resource] {
        if device.Topology != nil && len(device.Topology.Nodes) > 0 {
            return true
        }
    }
    return false
}
```

and in `GetTopologyHints`:

```go
if aligned := m.deviceHasTopologyAlignment(resource); !aligned {
    logger.Info("Resource does not have a topology preference", ...)
    deviceHints[resource] = nil        // ← nil, not an empty slice
    continue
}
```

`nil` — which, from §4 stage A, `filterProvidersHints` converts into `{nil, Preferred: true}`: a **preferred don't-care**. It is skipped in the bitwise AND and it cannot set `Preferred: false`. The GPU therefore contributes exactly nothing to the merge, in either direction. Topology Manager aligns CPU and memory to each other perfectly, marks the result preferred, and `single-numa-node` admits it — while the GPU sits on whichever node Device Manager's own allocator picked.

**Where the NVIDIA plugin decides this.** From `internal/rm/nvml_devices.go` in `NVIDIA/k8s-device-plugin`, the plugin does not ask NVML for a NUMA node; it reads sysfs:

```go
func (d nvmlDevice) GetNumaNode() (bool, int, error) {
    info, ret := d.GetPciInfo()
    ...
    busID := strings.ToLower(strings.TrimPrefix(int8Slice(info.BusId[:]).String(), "0000"))
    b, err := os.ReadFile(fmt.Sprintf("/sys/bus/pci/devices/%s/numa_node", busID))
    if err != nil {
        return false, 0, nil          // file missing → no NUMA info, no error raised
    }
    node, err := strconv.Atoi(string(bytes.TrimSpace(b)))
    ...
    if node < 0 {
        return false, 0, nil          // numa_node == -1 → no NUMA info, no error raised
    }
    return true, node, nil
}
```

and in `internal/rm/devices.go`:

```go
hasNuma, numa, err := d.GetNumaNode()
...
if hasNuma {
    dev.Topology = &pluginapi.TopologyInfo{
        Nodes: []*pluginapi.NUMANode{{ID: int64(numa)}},
    }
}
// else: dev.Topology stays nil — and nothing logs a warning
```

So the whole GPU-alignment guarantee, on a stock NVIDIA plugin, hangs on one sysfs file per GPU containing a non-negative integer. Three real situations make it `-1` or absent:

- **Virtual machines and many cloud instance types** that do not expose an ACPI SRAT/PXM affinity for passthrough PCIe devices. `numa_node` reads `-1`. Very common.
- **Tegra / Jetson platforms** — the plugin's `tegraDevice.GetNumaNode()` returns unsupported unconditionally.
- **WSL** — `wslAllGPUsDevice.GetNumaNode()` likewise returns no NUMA association.

MIG devices inherit their parent GPU's NUMA node, so a MIG-partitioned node is fine *if* the parent's sysfs value is good.

**The one-line pre-flight check**, per GPU, before you trust any policy:

```
$ for d in /sys/bus/pci/devices/*/; do
    if [ -e "$d/vendor" ] && [ "$(cat $d/vendor)" = "0x10de" ]; then
      echo "$(basename $d) numa_node=$(cat $d/numa_node) class=$(cat $d/class)"
    fi
  done
0000:1b:00.0 numa_node=0 class=0x030200
0000:43:00.0 numa_node=0 class=0x030200
0000:52:00.0 numa_node=1 class=0x030200
...
```

`numa_node=-1` on any row means alignment for that GPU is **impossible**, whatever `topologyManagerPolicy` says. That is a node-condition-worthy fact, not a debugging afterthought.

### 10. What the misalignment actually costs — the arithmetic

Put a number on it, because "it's slower" does not survive a capacity-planning review.

Take one H100 SXM5 fed by a host-staged data loader over PCIe Gen5 ×16 (63 GB/s per direction), on a 2-socket Sapphire Rapids box with UPI 2.0 links at ~48 GB/s per direction per link.

```
  ALIGNED (CPUs, pages, GPU all on NUMA 1)
    dataloader thread reads its pinned staging buffer   → local DDR5, 307 GB/s, ~75 ns
    cudaMemcpyAsync H2D                                 → node-1 root complex → GPU
    UPI crossings per batch: 0
    Ceiling on H2D: the PCIe link. 63 GB/s.

  MISALIGNED (CPUs+pages on NUMA 0, GPU on NUMA 1)
    GPU's DMA engine reads host memory that lives on the *other* socket:
      GPU → node-1 root complex → UPI → node-0 memory controller → DDR5
    UPI crossings per batch: 1 (per byte, in the read direction)
    Ceiling on H2D: min(63 GB/s PCIe, UPI share)
```

The UPI link is not yours alone. It carries cache-coherence traffic for every core on both sockets, every other pod's remote accesses, and any peer-to-peer device traffic that has to cross. Even at a generous 50% share of one link you are at ~24 GB/s — **roughly 38% of the PCIe ceiling**. Published measurements of exactly this configuration land in the same band: Ronak Nathani's production write-up on GPU NUMA locality in Kubernetes reports **>30% higher p99 tail latency** for inference pods whose CPUs spanned both sockets versus single-socket-pinned pods, and recommends `memoryManagerPolicy: Static` specifically so memory participates in alignment rather than being left to first-touch.

Turn that into money. At a 2026-era neocloud rate of roughly $2–3 per GPU-hour:

```
  8-GPU node, 8760 h/yr, $2.50/GPU-hr        = $175,200/yr per node
  20% of wall-clock lost to a fixable stall  = $35,040/yr per node
  a 64-node pool                             = $2.2 M/yr
```

Those are illustrative rates, not a quote — plug your own. The structural point is what matters: the fix costs one kubelet config change and zero capital, and **no utilization dashboard shows the loss**, because `GPU-Util` reports kernel residency, not delivered bandwidth.

```
  ALIGNED vs MISALIGNED — same pod spec, same node, different bytes path

  ┌────────── NUMA node 0 ──────────┐   ┌────────── NUMA node 1 ──────────┐
  │ cores 0-31,64-95                │   │ cores 32-63,96-127              │
  │ DDR5 ×8  ~307 GB/s  ~75 ns      │   │ DDR5 ×8  ~307 GB/s  ~75 ns      │
  │            │                    │   │            │                    │
  │      ┌─────┴──────┐             │   │      ┌─────┴──────┐             │
  │      │ root cplx  │             │   │      │ root cplx  │             │
  │      └──┬──────┬──┘             │   │      └──┬──────┬──┘             │
  │      [nvme0] [mlx5_0]           │   │    [GPU4..7] [mlx5_4..7]        │
  └────────────────┬────────────────┘   └────────────┬────────────────────┘
                   │                                 │
                   └──────── UPI 2.0 ────────────────┘
                     ~48 GB/s/dir/link · +50-70 ns
                     SLIT distance 21 vs 10 (numactl --hardware)

  ── PATH A: ALIGNED (single-numa-node admitted, GPU hint present) ──
     cpuset.cpus = 32-39,96-103   cpuset.mems = 1   device = GPU4
     staging buffer first-touched by a node-1 core     → node-1 DDR5
     H2D DMA: node-1 DRAM → node-1 root complex → GPU4
     UPI crossings/batch: 0        H2D ceiling: 63 GB/s (the PCIe link)

  ── PATH B: MISALIGNED (best-effort, or GPU hint missing) ──
     cpuset.cpus = 0-7,64-71      cpuset.mems = 0   device = GPU4
     staging buffer first-touched by a node-0 core     → node-0 DDR5
     H2D DMA: node-0 DRAM → UPI ─────────────▶ node-1 root complex → GPU4
     UPI crossings/batch: 1        H2D ceiling: contended UPI share (~24 GB/s @50%)

     Both pods report GPU-Util 100%.  Only PATH B's step-time histogram
     goes bimodal, and only DCGM PCIe/NVLink byte counters and per-NUMA
     memory-bandwidth counters can tell them apart.
```

### 11. Where this is heading: Dynamic Resource Allocation

The device-plugin API has a structural ceiling, and the trap in §9 is a symptom of it, not an accident. A device plugin can tell the kubelet exactly one topological fact — a list of NUMA node IDs — and only the *kubelet* ever sees it, at admission, after the scheduler already committed the pod to a node. There is no way for a workload author to *declare* the requirement "give me a GPU and a NIC on the same PCIe root complex"; the best they can do is hope the plugin published something and the policy caught it.

**Dynamic Resource Allocation reached GA in Kubernetes v1.34** (September 2025). It replaces "GPU as an opaque integer count" with a structured, scheduler-visible model:

- A **DRA driver** publishes each device with a set of typed **attributes** on a `ResourceSlice` object. The documented example attribute set includes architecture, driver version, **PCIe root complex**, and **NUMA node**.
- A workload author writes a `ResourceClaim` with multiple named device **requests** and **inter-device constraints** between them. The canonical documented example is exactly this module's problem: a GPU *and* a NIC, constrained to share the same PCIe root complex.
- Because `ResourceClaim` constraints are evaluated by **`kube-scheduler`**, not only by the kubelet, the scheduler can filter and score nodes on topology attributes *before* binding. That closes the scheduler-blindness gap named in §3.

```yaml
# DRA: the co-location requirement as a first-class, scheduler-visible object
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-with-local-nic
spec:
  devices:
    requests:
      - name: gpu
        deviceClassName: gpu.nvidia.com
      - name: nic
        deviceClassName: nic.example.com
    constraints:
      - requests: ["gpu", "nic"]
        matchAttribute: "resource.example.com/pcieRootComplex"
```

Contrast the two models honestly:

| | Device plugin + `TopologyInfo` | DRA `ResourceSlice`/`ResourceClaim` |
|---|---|---|
| Who states the requirement | Nobody — it is inferred from a policy setting | The workload author, explicitly, in the claim |
| Who evaluates it | kubelet, at admission, post-binding | `kube-scheduler`, pre-binding (plus kubelet) |
| Expressiveness | one list of NUMA node IDs per device | arbitrary typed attributes + inter-device match constraints |
| Silent-failure mode | plugin omits `TopologyInfo` → constraint vanishes | driver omits the attribute → claim is *unsatisfiable*, pod stays `Pending` |
| Failure visibility | pod runs, misaligned, no signal | pod does not schedule; the reason is in scheduler events |

That last row is the real upgrade: the device-plugin era fails *open* (misaligned but running), DRA fails *closed* (unscheduled and visible).

At KubeCon EU 2026 NVIDIA donated its DRA driver to the CNCF, so production NVIDIA-GPU DRA support is now a CNCF-governed artifact rather than a vendor-proprietary one. One caution to keep straight: **GA of the core DRA API is not the same claim as "every vendor's DRA driver is battle-tested."** API GA means the interface is stable and you can build against it; a specific driver may still be maturing. Interviewers who probe this are checking whether you track the field or memorised a mechanism.

And do not conflate the two GA dates, because they are one release apart and easy to swap: **Memory Manager (a kubelet hint provider) went GA in v1.32. DRA (a scheduler-level device-selection API) went GA in v1.34.** Getting that backwards reads as not really knowing either feature.

### 12. Failure-mode quick reference

| Symptom | Mechanism | Confirm with |
|---|---|---|
| Pod runs but CPUs unpinned (`cpuset.cpus` = whole node range) | Not `Guaranteed` QoS, or fractional CPU request, or `cpuManagerPolicy: none` | `kubectl get pod -o jsonpath='{.status.qosClass}'`; `kubectl exec -- cat /sys/fs/cgroup/cpuset.cpus.effective` |
| Policy "on" but nothing is ever rejected | `best-effort` — `canAdmitPodResult` returns `true` unconditionally | read `topologyManagerPolicy` from the running kubelet config (`kubectl get --raw /api/v1/nodes/<node>/proxy/configz`) |
| GPU pod is cross-socket yet admits under `single-numa-node` | Device plugin published no `TopologyInfo` → `deviceHints[resource] = nil` → preferred don't-care | `cat /sys/bus/pci/devices/<bdf>/numa_node`; kubelet log line `Resource does not have a topology preference` |
| Kubelet won't start after enabling Memory Manager | `Σ reservedMemory ≠ kubeReserved + systemReserved + evictionHard[memory.available]` | kubelet journal; recompute including the default `100Mi` |
| Kubelet crash-loops after a policy change | Stale `cpu_manager_state` / `memory_manager_state` checkpoint | the literal `configured policy "static" differs from state checkpoint policy "none"` message |
| Pods loop: create → `TopologyAffinityError` → recreate | Merge falls through to `{defaultAffinity, false}` — oversized pod, or free CPUs fragmented across sockets | `kubectl describe pod`; `numactl -H` free-core count per node; kubelet logs for the per-provider hints and the "Best TopologyHint" line |
| Pod `Failed` with `SMTAlignmentError` | `full-pcpus-only` + CPU count not a multiple of threads-per-core, or not enough free physical cores | `lscpu -e` sibling map; the error message states requested vs cpus-per-core |
| Alignment works on bare metal, silently fails on the same image in a VM | `numa_node == -1` for passthrough GPUs; plugin can't publish topology | the sysfs sweep in §9 |

**Kubelet metrics worth scraping** (they exist precisely so you don't have to grep logs at fleet scale):

- `topology_manager_admission_errors_total` — count of admission rejections.
- `topology_manager_admission_duration_ms` — merge cost; watch it on high-NUMA machines.
- `container_aligned_compute_resources_count{scope, boundary}` — containers that *did* get a guaranteed NUMA-aligned allocation. Only incremented under `single-numa-node`, because that is the only policy for which the kubelet's `IsAlignmentGuaranteed()` returns true.
- `container_aligned_compute_resources_failure_count{scope, boundary}` — the same, for failures.

A cluster where `container_aligned_compute_resources_count` stays at zero while `topologyManagerPolicy` reads `single-numa-node` is telling you the policy is not actually loaded on those nodes.

## Perspectives

**Developer.** From a workload-YAML author's seat none of this is visible until a pod goes `TopologyAffinityError` with a reason string they've never seen. Their available fixes — rightsize the request, drop to `best-effort` — are the wrong ones; the real fix (kubelet config, device-plugin rollout) is entirely platform-side. The app team cannot self-serve past this. Under DRA that changes: the requirement becomes something the workload author writes down and the scheduler enforces.

**Operator / platform team.** This is your surface end to end: kubelet configuration, the two static prerequisites, the `reservedMemory` arithmetic, verifying the device plugin actually publishes topology, sizing `reservedSystemCPUs` so the feeder cores you want exclusive aren't eaten by the shared pool, and the drain/delete-state/restart choreography on every policy change. Nobody else in the org owns this layer, and it is invisible to everyone else when it is wrong.

**Kernel / hardware.** Topology Manager invents no kernel mechanism. Everything it does is orchestration of primitives that already existed: `cpuset.cpus` and `cpuset.mems` cgroup files, `/sys/bus/pci/devices/*/numa_node` for device affinity, the ACPI SLIT distance matrix for NUMA distances. Its actual contribution is **atomicity across resource types at admission time** — making it impossible for CPUs to be aligned while memory quietly isn't, because all three allocations are gated on one merged decision.

**Economics / future-proofing.** The failure mode is uniquely expensive because it is invisible per-pod and only aggregates into a number at fleet scale — exactly the shape of waste a FinOps-literate platform engineer is hired to find. And the direction of travel is legible: a core-API GA in v1.34 plus a CNCF driver donation are both signals of where engineering effort is flowing. A candidate who can speak to *both* the current device-plugin trap and the DRA trajectory is tracking the field rather than reciting a mechanism that will be legacy within a few release cycles.

## Real-world use cases

- **Ronak Nathani — "Keeping GPU Workloads NUMA-Local in Kubernetes."** The field's best practitioner deep-dive on this exact machinery: it walks the hint-provider merge, the NVIDIA `TopologyInfo` trap, and end-to-end verification on real GPU nodes. The concrete production number to remember: **inference pods whose CPUs spanned both sockets showed >30% higher p99 tail latency** than pods pinned to a single socket, with an explicit recommendation to set `memoryManagerPolicy: Static` so memory participates in alignment alongside CPU and GPU. **What it shows:** the cost of skipping Memory Manager is a measured, double-digit-percent tail-latency regression, not a theoretical one. `https://ronaknathani.com/blog/2026/05/keeping-gpu-workloads-numa-local-in-kubernetes/`
- **kubernetes/kubernetes issue #128669 — "NUMA-aware memory manager and Topology Manager policy of `restricted` results in `TopologyAffinityError` when it shouldn't."** A real bug report against exactly the `Preferred`-semantics subtlety from §4: under `restricted`, a merged hint can be marked not-preferred because the providers' masks *differed*, even when the AND produced a workable allocation. **What it shows:** the distinction between "fits on N nodes" and "everyone got their own best case and agreed" is not pedantry — it is the source of production admission failures that look wrong until you know the merge rule.
- **Google Cloud Blog — "Kubernetes device management with DRA Dynamic Resource Allocation."** Confirms the `ResourceSlice`/`ResourceClaim` model in detail, including the PCIe-root-complex/NUMA-node attribute example, and explicitly contrasts DRA's granular topology-aware scheduling against the legacy device plugin's integer-count model. **What it shows:** a major cloud vendor's own framing of why DRA is a structural rather than cosmetic upgrade. `https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation`
- **Kubernetes Blog — "Kubernetes v1.34: DRA has graduated to GA."** The official GA announcement anchoring the version/timeline fact used throughout this lesson. **What it shows:** the primary-source date to cite when asked "when did DRA go GA." `https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/`

## Worked example

A single GPU node you control (a rented 2-socket box running k3s, or a kubelet you can restart). Two NUMA nodes; GPUs 0–3 on NUMA 0, GPUs 4–7 on NUMA 1. Every transcript below is **representative** — the exact CPU ranges and PCI addresses will differ on your hardware — but every command, field name and error string is real.

### Step 0 — establish the ground truth before touching Kubernetes

```
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0-31 64-95
node 0 size: 515655 MB
node 1 cpus: 32-63 96-127
node 1 size: 516086 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10

$ lscpu -e=CPU,CORE,SOCKET,NODE | head -5
CPU CORE SOCKET NODE
0   0    0      0
1   1    0      0
...
$ lscpu | grep -E 'Thread|Socket|NUMA node\(s\)'
Thread(s) per core:   2
Socket(s):            2
NUMA node(s):         2

$ for d in /sys/bus/pci/devices/*/; do
    [ "$(cat $d/vendor 2>/dev/null)" = "0x10de" ] &&
      echo "$(basename $d) numa_node=$(cat $d/numa_node)"
  done
0000:1b:00.0 numa_node=0
0000:43:00.0 numa_node=0
0000:52:00.0 numa_node=0
0000:61:00.0 numa_node=0
0000:9d:00.0 numa_node=1
0000:c3:00.0 numa_node=1
0000:d1:00.0 numa_node=1
0000:df:00.0 numa_node=1
```

Every GPU reports a non-negative `numa_node`. **Alignment is possible on this box.** If any row had shown `-1`, stop here — no kubelet setting can fix it, and you would go fix the hypervisor/firmware NUMA exposure instead.

### Step 1 — configure the kubelet

```yaml
# /var/lib/kubelet/config.yaml  (KubeletConfiguration, abridged to the relevant fields)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# --- CPU Manager: required, or CPU contributes a don't-care hint ---
cpuManagerPolicy: static
cpuManagerPolicyOptions:
  full-pcpus-only: "true"          # whole physical cores; SMTAlignmentError if not a multiple of 2
  strict-cpu-reservation: "true"   # keep BestEffort/Burstable off the reserved cores
reservedSystemCPUs: "0-3,64-67"    # 4 physical cores (8 threads) for kubelet+runtime+OS

# --- Memory Manager: required, or memory contributes a don't-care hint ---
memoryManagerPolicy: Static
reservedMemory:
  - numaNode: 0
    limits: { memory: "3Gi" }
  - numaNode: 1
    limits: { memory: "2148Mi" }   # 2Gi + 100Mi eviction threshold

# --- the coordinator ---
topologyManagerPolicy: single-numa-node
topologyManagerScope: pod
topologyManagerPolicyOptions:
  max-allowable-numa-nodes: "8"    # explicit; 8 is the default

# --- the arithmetic reservedMemory must satisfy ---
kubeReserved:   { cpu: "4", memory: "4Gi" }
systemReserved: { cpu: "1", memory: "1Gi" }
# Σ reservedMemory = 3Gi + 2148Mi = 5Gi + 100Mi
#                  = kubeReserved(4Gi) + systemReserved(1Gi) + evictionHard default(100Mi) ✓
```

Apply it correctly or the node does not come back:

```
$ kubectl drain gpu-07 --ignore-daemonsets --delete-emptydir-data
$ systemctl stop kubelet
$ rm -f /var/lib/kubelet/cpu_manager_state /var/lib/kubelet/memory_manager_state
$ systemctl start kubelet
$ journalctl -u kubelet -n 50 --no-pager | grep -iE 'topology|cpumanager|memorymanager'
... "Starting topology manager" policy="single-numa-node" scope="pod"
... "Starting CPU manager" policy="static"
... "Starting memory manager" policy="Static"
$ kubectl uncordon gpu-07
```

Skip the `rm` and you get the checkpoint mismatch verbatim:

```
E... kubelet.go:1567] "Failed to start ContainerManager" err="start cpu manager error:
could not restore state from checkpoint: configured policy \"static\" differs from state
checkpoint policy \"none\", please drain this node and delete the CPU manager checkpoint
file \"/var/lib/kubelet/cpu_manager_state\" before restarting Kubelet"
```

### Step 2 — the admit case

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aligned-trainer
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: nvcr.io/nvidia/pytorch:24.05-py3
    command: ["sleep", "infinity"]
    resources:
      limits:                        # requests == limits for cpu AND memory → Guaranteed
        cpu: "8"                     # integer → eligible for exclusive CPUs
        memory: "64Gi"
        nvidia.com/gpu: "1"
      requests:
        cpu: "8"
        memory: "64Gi"
        nvidia.com/gpu: "1"
```

```
$ kubectl apply -f aligned-trainer.yaml
$ kubectl get pod aligned-trainer -o jsonpath='{.status.qosClass}{"\n"}'
Guaranteed

$ kubectl exec aligned-trainer -- cat /sys/fs/cgroup/cpuset.cpus.effective
36-39,100-103
$ kubectl exec aligned-trainer -- cat /sys/fs/cgroup/cpuset.mems.effective
1
$ kubectl exec aligned-trainer -- nvidia-smi --query-gpu=index,pci.bus_id --format=csv,noheader
0, 00000000:9D:00.0
```

Read it line by line:

- `36-39,100-103` — 8 logical CPUs, and because `full-pcpus-only` is on, they are 4 *whole physical cores* with both SMT siblings each (36/100, 37/101, 38/102, 39/103). Cross-check against `numactl --hardware`: node 1 owns `32-63` and `96-127`, so all 8 are on **NUMA 1**.
- `cpuset.mems.effective = 1` — Memory Manager wrote it. Every page this container touches comes from node 1's controllers.
- The GPU's bus ID `9D:00.0` — from step 0's sweep, `numa_node=1`. **Match.**

Confirm the kubelet agrees with itself:

```
$ jq '.entries' /var/lib/kubelet/cpu_manager_state
{
  "1f2c8f6e-...": { "app": "36-39,100-103" }
}
$ jq '.entries["1f2c8f6e-..."]' /var/lib/kubelet/memory_manager_state
{
  "app": [ { "numaAffinity": [1], "type": "memory", "size": 68719476736 } ]
}
```

`"numaAffinity": [1]` — one element, one node. That is the merged hint, persisted.

And the log line that shows the whole merge, if you raise verbosity:

```
$ journalctl -u kubelet | grep -E 'TopologyHints|Best TopologyHint' | tail -4
... "TopologyHints" hints={"cpu":[{"NUMANodeAffinity":"01","Preferred":true},
                                  {"NUMANodeAffinity":"10","Preferred":true},
                                  {"NUMANodeAffinity":"11","Preferred":false}]} ...
... "TopologyHints" hints={"memory":[{"NUMANodeAffinity":"01","Preferred":true},
                                     {"NUMANodeAffinity":"10","Preferred":true}]} ...
... "TopologyHints" hints={"nvidia.com/gpu":[{"NUMANodeAffinity":"10","Preferred":true}]} ...
... "Best TopologyHint" bestHint={"NUMANodeAffinity":"10","Preferred":true} pod="default/aligned-trainer"
```

**That third line is the one you are checking for.** `"nvidia.com/gpu"` with a real `NUMANodeAffinity`. If it instead reads `Resource does not have a topology preference`, you are in the trap.

### Step 3 — the reject case, and the `best-effort` contrast on identical input

Ask for more exclusive CPUs than one node can supply. Node 1 has 32 threads total; reserve 8 and allocate 8 to the pod above; request 40:

```yaml
    resources:
      limits:   { cpu: "40", memory: "64Gi", nvidia.com/gpu: "1" }
      requests: { cpu: "40", memory: "64Gi", nvidia.com/gpu: "1" }
```

```
$ kubectl apply -f oversized.yaml
$ kubectl get pod oversized
NAME        READY   STATUS                  RESTARTS   AGE
oversized   0/1     TopologyAffinityError   0          4s

$ kubectl describe pod oversized | tail -6
Status:   Failed
Reason:   TopologyAffinityError
Message:  Resources cannot be allocated with Topology locality
Events:
  Type     Reason                 Age  From              Message
  Warning  TopologyAffinityError  4s   kubelet, gpu-07   Resources cannot be allocated with Topology locality
```

Trace *why*, mechanically: CPU Manager's only candidate for 40 exclusive CPUs is `0b11` (both nodes). `filterSingleNumaHints` drops it — it has 2 bits set — leaving CPU's candidate list **empty**. An empty inner slice makes the cross-product produce zero permutations, `bestHint` stays nil, and `Merge()` falls through to `{defaultAffinity, false}`. `canAdmitPodResult` reads `hint.Preferred` → `false` → reject.

Now change **one string** and nothing else:

```
$ kubectl drain gpu-07 --ignore-daemonsets --delete-emptydir-data
$ systemctl stop kubelet
$ sed -i 's/topologyManagerPolicy: single-numa-node/topologyManagerPolicy: best-effort/' \
    /var/lib/kubelet/config.yaml
$ rm -f /var/lib/kubelet/cpu_manager_state /var/lib/kubelet/memory_manager_state
$ systemctl start kubelet && kubectl uncordon gpu-07
$ kubectl apply -f oversized.yaml         # the SAME manifest
$ kubectl get pod oversized
NAME        READY   STATUS    RESTARTS   AGE
oversized   1/1     Running   0          9s
$ kubectl exec oversized -- cat /sys/fs/cgroup/cpuset.cpus.effective
4-31,68-79           # spans NUMA 0 and NUMA 1
$ kubectl exec oversized -- cat /sys/fs/cgroup/cpuset.mems.effective
0,1
```

Same node, same pod spec, same merged hint. `best-effort` computed `{0b11, Preferred: false}` and admitted it anyway. **That is the whole guarantee-vs-attempt distinction demonstrated on identical inputs with one string changed** — and it is the single most convincing thing you can put in the deliverable.

### Step 4 — reproduce the `TopologyInfo` trap

If you can run a device plugin with topology reporting disabled, or you have access to a VM instance type where `numa_node` reads `-1`, do it for real. Otherwise you can force the same state on a test box by making the plugin's read fail — the point is to see the kubelet's own log line and the resulting misalignment.

Pre-condition, showing the hardware *does* know:

```
$ cat /sys/bus/pci/devices/0000:9d:00.0/numa_node
1
```

With the plugin publishing no `TopologyInfo`, the kubelet says so explicitly:

```
$ journalctl -u kubelet | grep 'topology preference'
... "Resource does not have a topology preference" resourceName="nvidia.com/gpu"
    pod="default/trapped" containerName="app" request=1
```

Re-apply the step-2 pod, on a node where NUMA 0 has free capacity and NUMA 1 doesn't. Under `single-numa-node` it **admits**:

```
$ kubectl get pod trapped
NAME      READY   STATUS    RESTARTS   AGE
trapped   1/1     Running   0          6s
$ kubectl exec trapped -- cat /sys/fs/cgroup/cpuset.cpus.effective
8-11,72-75                       # NUMA 0
$ kubectl exec trapped -- cat /sys/fs/cgroup/cpuset.mems.effective
0                                # NUMA 0
$ kubectl exec trapped -- nvidia-smi --query-gpu=pci.bus_id --format=csv,noheader
00000000:9D:00.0                 # → numa_node 1.  MISMATCH.
```

Policy: satisfied. Hardware: cross-socket. Error count: zero. The merge did exactly what §4 says it does — the GPU's `nil` hint was skipped in the AND and could not set `Preferred: false`, so `single-numa-node` saw a clean one-node result and admitted.

### Step 5 — quantify the cost on this box

Run the same host↔device bandwidth test aligned and misaligned so the deliverable has a measured number rather than a citation:

```
$ numactl --cpunodebind=1 --membind=1 ./bandwidthTest --memory=pinned --device=4 --htod
 Host to Device Bandwidth, 1 Device(s), Pinned memory
   Transfer Size (Bytes)   Bandwidth(GB/s)
   33554432                55.3

$ numactl --cpunodebind=0 --membind=0 ./bandwidthTest --memory=pinned --device=4 --htod
 Host to Device Bandwidth, 1 Device(s), Pinned memory
   Transfer Size (Bytes)   Bandwidth(GB/s)
   31.7
```

Representative numbers on a Gen5 ×16 link; your box will differ. The shape is what generalises: aligned lands near the PCIe ceiling, misaligned lands wherever the contended inter-socket link puts it. Record both, compute the ratio, and state which production metric would have caught it (**DCGM PCIe byte counters and per-NUMA memory-bandwidth counters: yes. `GPU-Util`: no.**).

## Practice

On a test node you control (kind/k3s/a single kubelet), producing evidence for the [Topology Teardown](../practice/topology-teardown/README.md) deliverable:

1. **Pre-flight.** Capture `numactl --hardware`, `lscpu -e`, and the per-GPU `numa_node` sweep from Worked example step 0. Record whether alignment is even *possible* on this hardware, and say why.
2. **Configure.** Apply `cpuManagerPolicy: static` (+ `full-pcpus-only`), `memoryManagerPolicy: Static` (+ a `reservedMemory` block whose sum you show satisfies the equation, including the `100Mi` eviction default), `topologyManagerPolicy: single-numa-node`, `topologyManagerScope: pod`. Capture the exact config, the drain/stop/delete-state/start/uncordon sequence, and the kubelet log lines confirming all three managers started with the intended policies.
3. **Admit case.** Schedule a `Guaranteed` integer-CPU pod that fits one NUMA node. Verify **all four** of: `qosClass == Guaranteed`; `cpuset.cpus.effective` ∈ one node per `numactl -H`; `cpuset.mems.effective` = that node; and (with a GPU) the allocated GPU's `numa_node` = that node. Paste the `cpu_manager_state` and `memory_manager_state` entries.
4. **Reject case.** Schedule a pod that cannot fit one NUMA node. Capture the `TopologyAffinityError` status *and* the `describe` event with the message `Resources cannot be allocated with Topology locality`. Then flip **only** the policy string to `best-effort`, redeploy the identical manifest, and show it now admits with a cpuset spanning both nodes. This before/after pair is the core artifact.
5. **`TopologyInfo` trap.** Either reproduce it (device plugin without topology reporting, or `numa_node == -1` hardware) and show GPU↔CPU alignment silently not happening, **or** — if you can't get the hardware — write it up mechanically: quote `deviceHasTopologyAlignment`, show your GPU's `/sys/bus/pci/devices/<bdf>/numa_node`, and explain in your own words why a `nil` hint is skipped in the AND and cannot set `Preferred: false`, and therefore why `single-numa-node` admits a misaligned GPU pod.
6. **Fleet detection.** Write the check you would run on every GPU node at bring-up: read each NVIDIA-vendor PCI device's `numa_node`, fail the node if any is `-1`, and surface it as a node condition or label. Name the two kubelet metrics you would alert on (`topology_manager_admission_errors_total`, `container_aligned_compute_resources_count`) and what a zero value of the second one means when the policy claims to be `single-numa-node`.
7. **Bonus (no cluster required).** Sketch the DRA `ResourceClaim` equivalent for your node's GPU+NIC pairing and write one paragraph on why the DRA failure mode (unsatisfiable claim → `Pending`) is structurally safer than the device-plugin one (missing attribute → silently misaligned but running).

**Acceptance:** a note capturing (1) the pre-flight verdict on whether alignment is possible, (2) the working config with the `reservedMemory` arithmetic shown, (3) the admit-vs-reject behaviour with the exact `TopologyAffinityError` output and the `best-effort` contrast on an identical manifest, (4) the `TopologyInfo` trap explained or reproduced, and (5) the fleet detection design. That note is the deliverable artifact.

## Common pitfalls

1. **Enabling `topologyManagerPolicy` without both static prerequisites.** With `cpuManagerPolicy: none` and `memoryManagerPolicy: None`, both providers return don't-care hints, the merge is driven entirely by whatever the device plugin published, and the policy setting accomplishes nothing observable. Verify all three policies from the *running* kubelet config, not from your intended manifest.
2. **Treating `best-effort` as a guarantee.** It computes the same merged hint as `single-numa-node` and then unconditionally returns `true` from `canAdmitPodResult`. If the interview answer is "I enabled Topology Manager," the follow-up is always "which policy," and `best-effort` is the wrong answer to "guarantee."
3. **Assuming `restricted` means "single NUMA node."** It means "the merged hint is `Preferred`" — every provider offered its own narrowest option and all their non-nil masks were identical. A pod that legitimately needs two NUMA nodes is *admitted* by `restricted` and *rejected* by `single-numa-node`. Conversely, a two-provider disagreement (`0b01` vs `0b11`) that ANDs to a clean single node is *rejected* by `restricted` despite fitting on one node — see kubernetes/kubernetes #128669.
4. **Believing a missing `TopologyInfo` produces an all-ones bitmask.** It produces a **nil** affinity. The AND result is the same, but the second consequence is the one that bites: a nil hint is always `Preferred: true`, so it can never flip the merged hint to not-preferred and can never cause a rejection. The GPU is invisible to the merge in both directions.
5. **Forgetting the `100Mi` default eviction threshold in the `reservedMemory` sum.** The equation is `Σ reservedMemory = kubeReserved + systemReserved + evictionHard[memory.available]`, and that last term is `100Mi` unless you set it. Getting it wrong stops the kubelet at startup — a very effective way to take a GPU node out of the fleet during a routine config rollout.
6. **Changing a policy without draining and deleting the state checkpoints.** Both `cpu_manager_state` and `memory_manager_state` persist the policy name and are validated at startup. The crash-loop message names the exact file to delete; read it rather than guessing.
7. **Conflating Memory Manager's GA version with DRA's.** Memory Manager (a kubelet hint provider) went GA in **v1.32**. DRA (a scheduler-level device API) went GA in **v1.34**. One release apart, completely different layers.
8. **Reading "DRA is GA" as "every vendor's DRA driver is production-ready."** API GA means the interface is stable. A specific driver — including NVIDIA's, freshly donated to the CNCF — may still be maturing.
9. **Assuming `topologyManagerScope: pod` alone fixes multi-container alignment.** Scope changes the *unit* of the merge, nothing else. Without `Guaranteed` QoS, integer CPU, and both static policies, pod scope aligns a set of don't-cares to a don't-care.
10. **Setting `align-by-socket` together with `single-numa-node`.** They are explicitly incompatible; the kubelet fails to start and tells you so in the logs. On a standard 2-socket/2-NUMA-node box the option is a no-op anyway.

## Self-check

**(a) What does `single-numa-node` do when a pod's resources can't fit one NUMA node, and how does that differ from `best-effort`?**

**Answer:** `single-numa-node` pre-filters every provider's candidate hints down to those with exactly one bit set (plus nil don't-cares). If any resource's list becomes empty, the cross-product yields zero permutations, `Merge()` falls through to `TopologyHint{defaultAffinity, Preferred: false}`, and `canAdmitPodResult` — which is literally `return hint.Preferred` — rejects. The pod's `STATUS` column reads `TopologyAffinityError` and `describe` shows the event message `Resources cannot be allocated with Topology locality`. It does **not** go `Pending` or get rescheduled, because the kubelet already owns it; the owning controller creates a replacement, which can loop. `best-effort` runs the *same* merge and its `canAdmitPodResult` is `return true` — it stores the (non-preferred) hint so providers try to honour it, and admits regardless. On identical inputs: `single-numa-node` = "one node or dead," `best-effort` = "prefer one node, run cross-socket anyway." `best-effort` is the silent-waste policy.

**(b) Why can GPU↔CPU alignment silently fail even with `single-numa-node` enabled?**

**Answer:** Device Manager can only emit a NUMA hint for a resource if at least one device of that resource has `Topology != nil && len(Topology.Nodes) > 0` — that is `deviceHasTopologyAlignment()` in the kubelet. If not, it sets `deviceHints[resource] = nil`, which `filterProvidersHints` converts to `{nil, Preferred: true}`: a preferred don't-care. In `mergePermutation`, hints with a nil affinity are **skipped entirely** — excluded from the bitwise AND, and unable to contribute `Preferred: false`. The GPU therefore constrains nothing and objects to nothing. Topology Manager aligns CPU and memory to each other, marks the result preferred, and admits. `single-numa-node` looks like it worked because it perfectly aligned everything it was given a hint for. On the NVIDIA plugin the root cause is one sysfs read: `GetNumaNode()` reads `/sys/bus/pci/devices/<bdf>/numa_node` and returns "no NUMA info" both when the file is missing and when the value is negative — common on VMs, Tegra and WSL — and `BuildDevice` then leaves `dev.Topology` nil without logging a warning. Detect it with the per-GPU `numa_node` sweep, or by grepping the kubelet log for `Resource does not have a topology preference`.

**(c) What does `full-pcpus-only` add, and what happens to a pod that violates it?**

**Answer:** On an SMT node a Kubernetes "CPU" is a hardware thread, and two sibling threads share one physical core's execution units. Plain `static` can hand a pod threads that split a physical core with a *different* pod's thread — both pods believe they have exclusive CPUs while contending on the shared core, which shows up as jitter and lost throughput on GPU feeder threads. `full-pcpus-only` forces allocation in whole-physical-core units (both siblings together). A pod whose CPU count is not a multiple of threads-per-core, or for which there aren't enough *free physical* cores, is put in `Failed` state with reason `SMTAlignmentError` and one of two messages: `SMT Alignment Error: requested N cpus not multiple cpus per core = M`, or `SMT Alignment Error: not enough free physical CPUs: available physical CPUs = X, requested CPUs = N, CPUs per core = M`. The option has been available since v1.22 and went GA in v1.33.

**(d) Walk the merge for: CPU offers `{0b01,P}`,`{0b10,P}`,`{0b11,¬P}`; memory offers `{0b01,P}`,`{0b10,P}`; GPU offers `{0b10,P}`. What does each policy decide?**

**Answer:** Six permutations (3 × 2 × 1). Any permutation pairing a `0b01` with the GPU's `0b10` ANDs to `0b00` — zero bits — and `compare()` skips those outright. The survivors are `{cpu 0b10, mem 0b10, gpu 0b10}` → AND `0b10`, all three masks identical and all preferred → `{0b10, Preferred: true}`; and `{cpu 0b11, mem 0b10, gpu 0b10}` → AND `0b10`, but the masks are not all equal → `{0b10, Preferred: false}`. A preferred candidate always beats a non-preferred one, so `bestHint = {0b10, Preferred: true}`. `none` never merges and admits. `best-effort` admits. `restricted` admits (`Preferred` is true). `single-numa-node` first drops CPU's `0b11` candidate in `filterSingleNumaHints`, reaches the same `{0b10, true}`, and admits. All allocations land on NUMA 1. Now delete the GPU's hint: the surviving best becomes `{0b01, true}` or `{0b10, true}` depending on iteration order, all policies admit, and the GPU's actual node is decided by Device Manager's own allocator — with no relationship to the CPUs.

**(e) A pod is `Guaranteed` with `cpu: "3500m"`, `topologyManagerPolicy: single-numa-node`, and both static policies enabled. What alignment does it get?**

**Answer:** None from the CPU side. `static` grants exclusive CPUs only when the QoS is `Guaranteed` *and* the CPU request is a positive integer, tested as `cpuQuantity.Value()*1000 == cpuQuantity.MilliValue()`. `3500m` fails that (`3*1000 ≠ 3500`), so the container runs in the shared pool with CFS quota and CPU Manager returns the default don't-care hint. Memory Manager still returns real hints (it keys on `Guaranteed` QoS, not on integer CPU), and Device Manager still returns GPU hints, so memory and GPU *will* be aligned to each other — but the container's threads float across both sockets under the OS scheduler, and every host↔device copy issued from a node-0 core to a node-1 GPU crosses UPI. This is the sneakiest variant of the failure: two of three resources aligned, admission succeeded, and the one that determines where the copies are issued from is unpinned. Fix: `cpu: "4"`.

**(f) What Kubernetes version did DRA reach GA in, and what structurally changed about where a topology constraint lives?**

**Answer:** **v1.34**, September 2025. Structurally it moves the constraint from a flat, kubelet-only hint (`TopologyInfo.NUMANodes`, a list of integers, visible only to Device Manager at admission and inferred rather than declared) to a `ResourceSlice`/`ResourceClaim` model: DRA drivers publish rich typed attributes per device (architecture, driver version, PCIe root complex, NUMA node), and the workload author writes explicit inter-device `constraints` on a `ResourceClaim` — e.g. `matchAttribute: pcieRootComplex` across a `gpu` and a `nic` request. Those constraints are evaluated by `kube-scheduler` **before binding**, closing the scheduler-blindness gap that makes the device-plugin era's silent failures possible. The failure mode inverts too: a device plugin that omits topology fails *open* (misaligned pod runs), whereas a DRA driver that omits an attribute makes the claim unsatisfiable and the pod stays `Pending` with a visible scheduler event.

## Connections & what's next

This lesson turns lesson 04's fixed hardware layout into an enforceable Kubernetes guarantee — the "hardware is correct" fact from the 8-GPU server architecture lesson only pays off if the orchestrator respects it, and that is exactly what Topology Manager plus the two static policies do, or silently fail to do via the `TopologyInfo` trap. The bitmask arithmetic here is the software mirror of the NUMA distance matrix you read in lessons 01–02: `numactl --hardware`'s `10`/`21` is the same SLIT table that `prefer-closest-numa-nodes` consults.

The same underlying question — "is this resource actually co-located with the GPU it is supposed to feed?" — reappears immediately in **lesson 06**, except the resource is a storage device instead of CPU and memory. A pod can pass every check in this lesson's worked example and still stall badly because its dataset lives on an NVMe controller across the socket boundary; and unlike CPUs and memory, **there is no Topology Manager hint provider for block devices at all** — nothing in Kubernetes will align storage for you. Both lessons feed the **lesson 08 capstone**, where you reconcile four tools into one topology diagram on a real node: the negative test you ran here (force a rejection, relax the policy, watch it silently admit) is exactly the tool-disagreement discipline that capstone rewards.

## References & further reading

**Primary sources**

- Kubernetes — "Control Topology Management Policies on a node" — `https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/` — canonical reference for the four policies, the two scopes, the policy options and their GA versions, and the documented known limitation that the scheduler is not topology-aware. **Correction applied from this source:** the previous version of this lesson stated that Topology Manager runs only for `Guaranteed` pods with integer CPU. The docs are explicit that Topology Manager "aligns Pods of all QoS classes" and aligns whatever resources hint providers give hints for — the QoS gate lives in *CPU Manager* and *Memory Manager*, which return default (don't-care) hints for non-`Guaranteed` pods. A `BestEffort` pod requesting only devices is still aligned by the device hints.
- Kubernetes — "Control CPU Management Policies on the Node" — `https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/` — the `static` policy, the shared/reserved pool split, the six policy options with their maturity levels and versions, and the exact drain/delete-state/restart procedure with the checkpoint-mismatch error message.
- Kubernetes — "Control Memory Management Policies on a Node" — `https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/` — the `Static` policy, the `reservedMemory` equation including the default `100Mi` eviction threshold, the per-type/per-node syntax, NUMA groups, and the configurations the validator rejects.
- Kubernetes — "Resource managers" (concept) — `https://kubernetes.io/docs/concepts/workloads/resource-managers/` — the per-option behaviour descriptions for `full-pcpus-only`, `distribute-cpus-across-numa`, `align-by-socket` (including its incompatibility with `single-numa-node`), `distribute-cpus-across-cores`, `strict-cpu-reservation` and `prefer-align-cpus-by-uncorecache`, plus the pod-level resource managers feature.
- Kubernetes — "Troubleshooting Topology Management" — `https://kubernetes.io/docs/tasks/debug/debug-cluster/topology/` — where the `TopologyAffinityError` status and event message come from, and a real annotated `memory_manager_state` dump showing a two-node NUMA group. **Correction applied:** the previous version described a rejected pod as `Failed`/`Terminated`; the pod's `STATUS` column literally displays `TopologyAffinityError`.
- Kubernetes — "Device Plugins" § "Device plugin integration with the Topology Manager" — `https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/` — the `TopologyInfo`/`NUMANode` protobuf messages and the explicit statement that nil or an empty node list means "no NUMA affinity preference."
- `kubernetes/kubernetes` — `pkg/kubelet/cm/topologymanager/` — `policy.go` (`mergePermutation`, `filterProvidersHints`, `HintMerger`, `iterateAllProviderTopologyHints`), `policy_best_effort.go` / `policy_restricted.go` / `policy_single_numa_node.go` (the one-line `canAdmitPodResult` differences and `filterSingleNumaHints`), `topology_manager.go` (`TopologyHint`, `TopologyAffinityError`, the default `max-allowable-numa-nodes` of 8). The only place the merge semantics are stated exactly. `https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/cm/topologymanager`
- `kubernetes/kubernetes` — `pkg/kubelet/cm/devicemanager/topology_hints.go` and `pkg/kubelet/cm/cpumanager/policy_static.go` — `deviceHasTopologyAlignment()` (the trap's kubelet half), `isIntegralCPUAmount()`, and the `SMTAlignmentError` message strings. `https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/cm`
- `NVIDIA/k8s-device-plugin` — `internal/rm/nvml_devices.go` and `internal/rm/devices.go` — `GetNumaNode()` reading `/sys/bus/pci/devices/<bdf>/numa_node` and returning "no NUMA info" for both a missing file and a negative value, and `BuildDevice()` leaving `Topology` nil in that case. The plugin's half of the trap. `https://github.com/NVIDIA/k8s-device-plugin`
- Kubernetes Blog — "Kubernetes v1.34: DRA has graduated to GA" — `https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/` — the primary-source GA date and what graduated with it.

**Real-world engineering**

- Ronak Nathani — "Keeping GPU Workloads NUMA-Local in Kubernetes" — `https://ronaknathani.com/blog/2026/05/keeping-gpu-workloads-numa-local-in-kubernetes/` — **what it shows:** a measured >30% p99 tail-latency cost of cross-socket inference pods, plus the operational checklist (including `memoryManagerPolicy: Static`) for getting alignment right on real GPU nodes.
- `kubernetes/kubernetes` issue #128669 — "NUMA-aware memory manager and Topology Manager policy of `restricted` results in `TopologyAffinityError` when it shouldn't" — `https://github.com/kubernetes/kubernetes/issues/128669` — **what it shows:** the `Preferred`-requires-identical-masks rule producing rejections that look wrong until you have read `mergePermutation`.
- Google Cloud Blog — "Kubernetes device management with DRA Dynamic Resource Allocation" — `https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation` — **what it shows:** the `ResourceSlice`/`ResourceClaim` model with the GPU+NIC PCIe-root-complex constraint example, contrasted against the device plugin's integer-count model.
