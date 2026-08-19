---
lesson: "02.9"
title: "The scheduler framework and the GPU-scheduling frontier"
module: "02"
concept: "The scheduler framework and the GPU-scheduling frontier"
status: not-started
est_time: "16h"
prev: "08-admission-webhooks.md"
next: "10-capstone-build.md"
artifacts: []
sources: 21
---

# 02.9 · The scheduler framework and the GPU-scheduling frontier

> **Concept.** The default scheduler is a framework of pluggable extension points; GPU and distributed-training workloads push past its integer-count, one-Pod-at-a-time model into DRA (request devices by attribute) and Kueue (quota + gang admission).
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 08 closed the write path: admission webhooks decide whether an object may exist at all. This lesson moves to the other end of an object's life — once a Pod *does* exist, something has to decide where it runs. That "where" question has three separate owners once GPUs and distributed jobs are involved, not one, and the interview failure mode is collapsing them into a single mental "the scheduler does it."

This lesson gives you the mechanism behind each layer: the scheduler's own extension points and how a GPU Pod's fate is decided at each one; how `nvidia.com/gpu` physically reaches the scheduler as an integer through a gRPC conversation between kubelet and a DaemonSet; why that integer is structurally insufficient; and what DRA (GA in Kubernetes 1.34) and Kueue replace it with. It closes with a design doc rather than code, and sets up Lesson 10, where everything built in this module gets assembled and tested.

## Why this matters

You run 40+ clusters and know the scheduler as an operator: taints, tolerations, affinity, `nvidia.com/gpu: 1`, `kubectl describe pod` on something Pending. At a GPU-heavy shop that literacy is table stakes. What differentiates a senior platform engineer is being able to sit at a whiteboard and reason about *where a placement decision is actually made, which layer owns it, and what breaks when a 512-GPU training job needs all its Pods or none of them*.

There is real money in the answer. A distributed job that gets 30 of its 32 Pods placed is not 94% running — it is 0% running and 100% billing. Thirty H100s at roughly $5/GPU-hour, parked waiting for two more, cost about $150/hour to accomplish nothing. Get several such jobs interleaved and they can deadlock, each holding a fraction of what it needs, none able to start. That is not a hypothetical; it is the reason Kueue exists.

This lesson is deliberately **literacy plus design**, not "write a production scheduler plugin." An in-tree Score plugin is a rare, high-blast-radius artifact owned by a handful of people. Being able to *design against* the framework — and to place your `gpu-cost-operator` correctly in the stack relative to DRA and Kueue — is the skill that scales across 40 clusters and the one a panel probes. Your FinOps background is the wedge: almost nobody is fluent in both the scheduling internals and the cost model, and that intersection is exactly where a `costScore` signal lives.

## What's new here (calibration)

As an operator you *configure* scheduling with knobs the scheduler already exposes. As an extender you reason about the decision pipeline itself, and about the two systems that now sit *around* the scheduler for AI/ML.

- **Already know, skip:** `kubectl` scheduling knobs — taints/tolerations, node affinity/anti-affinity, `nvidia.com/gpu: N` requests, reading a Pending Pod's events.
- **Already know, skip:** that a DaemonSet advertises `nvidia.com/gpu` as an extended resource; you have run NVIDIA's device plugin for years.
- **New here:** the scheduler as a *framework* — the extension points in order, what a plugin can return at each, and the exact point in the cycle where a GPU Pod's fate is decided.
- **New here:** the **device-plugin gRPC surface** — `Register`, `ListAndWatch`, `GetPreferredAllocation`, `Allocate`, `PreStartContainer` — and the sequence between kubelet and plugin that turns physical cards into an integer in `node.status.allocatable`.
- **New here:** why the scheduler treats `nvidia.com/gpu` as a **non-divisible, non-overcommittable integer**, what that forecloses, and how time-slicing and MIG try (and fail) to work around it.
- **New here:** the **Topology Manager**, the kubelet-side NUMA alignment the scheduler cannot see, and the `TopologyAffinityError` that results.
- **New here:** DRA's attribute-based device model (GA in **1.34**) and the KEP history (3063 withdrawn → 4381 GA) that explains why the current design looks the way it does.
- **New here:** Kueue as an admission/quota layer *above* the scheduler, and why gang semantics cannot be bolted onto a one-Pod-at-a-time loop.

Three distinct layers. Confusing them is the classic design-review failure:

| Layer | Question it answers | Owns | Object it acts on |
|---|---|---|---|
| **Kueue** | "Should this whole *job* start at all, right now, within quota?" | Gate + quota, *before Pods exist* | `Workload` (wrapping a Job) |
| **Scheduler framework** | "Given a Pod and N feasible nodes, which node?" | Per-Pod node selection | `Pod` → `Binding` |
| **DRA** (`resource.k8s.io`) | "Which *device* on that node, by attributes?" | Device modelling + allocation | `ResourceClaim` |

Kueue gates the door, the scheduler seats the guest, DRA picks the exact chair. A distributed training job touches all three. Versions this lesson is written against: **Kubernetes 1.36** (DRA GA since 1.34; in-tree `PodGroup` gang scheduling is alpha), **Kueue** `kueue.x-k8s.io` (v1beta1/v1beta2 depending on kind — check `kubectl api-resources | grep kueue`), **NVIDIA k8s-device-plugin v0.17.x**, device-plugin API `v1beta1`.

## Core concepts

### 1. The scheduler's loop, before the extension points

kube-scheduler is a single control loop with **one Pod in the scheduling cycle at a time**. That serialization is not laziness; it is what makes resource accounting correct. Two Pods scored concurrently against the same node would both see the same free capacity and both get placed, so the cycle is serial and the *binding* cycle — the part that does I/O — is what runs concurrently.

Before a Pod reaches the cycle it sits in one of three queues:

- **activeQ** — a priority heap ordered by the single `QueueSort` plugin (default `PrioritySort`: by `.spec.priority`, then by creation timestamp). Pods here are ready to be attempted.
- **backoffQ** — Pods that were attempted and failed, waiting out exponential backoff. Defaults: `podInitialBackoffSeconds: 1`, `podMaxBackoffSeconds: 10`. Same shape as the workqueue rate limiter from lesson 04, different implementation.
- **unschedulableQ** — Pods that could not be placed and are waiting for a *cluster event* that might change that (a Node added, a Pod deleted, a PVC bound). Each plugin declares which event types can make its own rejection stale, so a Pod that failed on `Insufficient nvidia.com/gpu` is re-queued when a Node's allocatable changes rather than being retried blindly.

Then the cycle splits:

- **Scheduling cycle** — synchronous, serial, no I/O. Picks a node.
- **Binding cycle** — asynchronous, concurrent across Pods. Does the API write and any pre-binding I/O (volume provisioning, DRA claim finalisation).

Everything below happens inside those two cycles.

### 2. The extension points, in order, with a GPU Pod's fate at each

Trace one Pod: 1 replica of a training job requesting `nvidia.com/gpu: 4`, on a 200-node cluster of which 24 have GPUs.

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ SCHEDULING CYCLE — serial, one Pod at a time, no blocking I/O                  │
└───────────────────────────────────────────────────────────────────────────────┘

 PreEnqueue      Runs before the Pod is even admitted to activeQ. Default plugin:
   │             SchedulingGates — if .spec.schedulingGates is non-empty the Pod
   │             never enters the queue and gets NO Unschedulable condition.
   │             GPU FATE: this is where Kueue-style gating COULD live; Kueue
   │                       instead suspends the Job so no Pod exists at all.
   ▼
 QueueSort       Exactly ONE plugin cluster-wide (PrioritySort). Orders activeQ.
   │             GPU FATE: a high-priority training job jumps ahead of inference.
   ▼
 PreFilter       Compute Pod-level facts ONCE instead of per node. NodeResourcesFit
   │             sums the Pod's requests here — including ScalarResources, where
   │             {"nvidia.com/gpu": 4} lands — into a preFilterState shared by all
   │             Filter calls. May also return a PreFilterResult restricting the
   │             candidate node set, or fail the Pod outright as unschedulable.
   │             GPU FATE: the "4" becomes an int64 in a map. Nothing about the
   │                       cards' model, memory, or NVLink topology is present.
   ▼
 Filter          THE PREDICATE STAGE. Per node, boolean, run across
   │             `parallelism` goroutines (default 16). Any plugin returning
   │             Unschedulable removes the node; the rest are skipped for it.
   │             Default filters: NodeName, NodeUnschedulable, TaintToleration,
   │             NodeAffinity, NodePorts, NodeResourcesFit, VolumeRestrictions,
   │             NodeVolumeLimits, VolumeBinding, VolumeZone, PodTopologySpread,
   │             InterPodAffinity (+ the DRA plugin when claims are present).
   │             GPU FATE: 176 CPU-only nodes fail NodeResourcesFit with
   │                       "Insufficient nvidia.com/gpu". Of the 24 GPU nodes,
   │                       17 have ≥4 free. 7 fail. Feasible set = 17.
   │                       Status is Unschedulable (preemption might help) vs
   │                       UnschedulableAndUnresolvable (it can't).
   ▼
 PostFilter      ONLY runs when the feasible set is EMPTY. Default plugin:
   │             DefaultPreemption — pick a victim set of lower-priority Pods
   │             whose eviction makes one node feasible, set nominatedNodeName,
   │             delete the victims, and return the Pod to the queue.
   │             GPU FATE: skipped here (17 feasible). If it had run: DRA
   │                       resources are NOT preemptible — the scheduler will
   │                       not evict a Pod to free a ResourceClaim.
   ▼
 PreScore        Precompute shared state for scoring. Errors abort the cycle.
   ▼
 Score           THE PRIORITY STAGE. Every surviving node gets an int64 from every
   │             Score plugin. Convention: MinScore=0, MaxScore=100.
   │             GPU FATE: this is the ONLY place a soft preference lives —
   │                       "prefer the cheaper card, but take the expensive one
   │                       if that's what's free."
   ▼
 NormalizeScore  A plugin rescales ITS OWN raw scores into [0,100] once it can see
   │             all of them. Then each plugin's score is multiplied by its
   │             configured weight and summed. Highest total wins; ties broken
   │             by reservoir sampling so equal nodes are chosen uniformly.
   ▼
 Reserve         Mark the resources claimed on the chosen node in the scheduler's
   │  ▲          in-memory cache, BEFORE any API write, so the next Pod in the
   │  │          serial cycle sees them consumed. Paired with Unreserve, which
   │  │          runs in REVERSE plugin order if anything later fails.
   │  └──Unreserve──┐   GPU FATE: DRA's plugin tentatively allocates a specific
   │                │             device here. Without DRA there is no per-device
   ▼                │             state — only the integer count in the cache.
 Permit            │  approve | deny | WAIT(timeout).
   │                │  "wait" parks the Pod without releasing its Reserve.
   │                │  GPU FATE: this is the framework hook gang scheduling uses —
   │                │            hold Pod 1 of 32 until the other 31 also reach
   │                │            Permit, then release them together. The DEFAULT
   │                │            profile has no plugin here, which is exactly why
   │                │            partial placement happens out of the box.
   ▼                │
┌───────────────────┼───────────────────────────────────────────────────────────┐
│ BINDING CYCLE — asynchronous, CONCURRENT across Pods                          │
└───────────────────┼───────────────────────────────────────────────────────────┘
 PreBind            │  Work that must SUCCEED before binding, and may do I/O.
   │                │  VolumeBinding provisions/binds PVCs here.
   │                │  GPU FATE: the DRA plugin writes the finalized allocation
   │                │            into ResourceClaim.status here — the first time
   │                │            the device decision leaves the scheduler's memory.
   │                │  Failure → Unreserve everything and requeue. ────────────┘
   ▼
 Bind            Writes the Binding object (POST /api/v1/namespaces/N/pods/P/binding),
   │             which sets .spec.nodeName. First Bind plugin to handle it wins;
   │             default is DefaultBinder.
   ▼
 PostBind        Informational cleanup/notify. Cannot fail the scheduling.
   ▼
 kubelet on the chosen node notices a Pod with its nodeName, and NOW the
 device-plugin machinery of §5 decides WHICH PHYSICAL CARDS this Pod gets.
 The scheduler never knew and never will.
```

That last line is the single most important thing in this lesson. **The scheduler picks a node; the kubelet picks the devices.** Without DRA, no component that made the placement decision knows which physical GPU the container ended up with, which NUMA node it hangs off, or whether the four cards it got are NVLink-connected peers or four strangers on different PCIe roots.

### 3. Filter versus Score, and why cost is a Score

Filter is a hard gate and Score is a soft bias, and the difference is the most common design-review catch at this layer.

A node that fails Filter is *gone*. If a Pod fails Filter on every node, it goes Pending, PostFilter may attempt preemption, and the Pod ends up in `unschedulableQ` waiting for a cluster event. Score never removes anything; it only ranks the survivors.

So "prefer the cheaper A100 over the H100 when the job only needs 40 GB" is a **Score** decision. Encoding cost as a Filter turns "prefer cheap" into "only cheap," and the moment the cheap tier fills, every subsequent Pod goes Pending even though expensive capacity is sitting idle — a self-inflicted outage that looks like a capacity problem.

The scoring arithmetic you need to design against:

| Constant / default | Value | Source |
|---|---|---|
| `MinScore` | `0` | scheduler framework |
| `MaxScore` | `100` | scheduler framework |
| `MaxTotalScore` | `math.MaxInt64` (overflow guard) | scheduler framework |
| Default plugin weights | `TaintToleration: 3`, `NodeAffinity: 2`, `PodTopologySpread: 2`, `InterPodAffinity: 2`, `NodeResourcesFit: 1`, `NodeResourcesBalancedAllocation: 1`, `ImageLocality: 1` | `getDefaultPlugins()` |

Two consequences for a `costScore` design:

1. **Normalize to 0–100 or be ignored.** If your plugin emits raw dollars-per-hour (say 2.50 vs 5.12) and everything else emits 0–100, your signal contributes at most ~5 points against a `TaintToleration` contribution of up to 300. It will "work" in a unit test and be invisible in production. `NormalizeScore` exists precisely to rescale before weighting.
2. **Weight against the total, not against one peer.** The default weights sum to 12 across seven scoring plugins, so a maximum total of 1,200. A `costScore` at weight 1 can move a placement by at most 100 points out of 1,200 — about 8%. If cost is meant to dominate among otherwise-equivalent GPU nodes, it needs a weight of 5–10, and you should say so explicitly in the design doc rather than shipping weight 1 and wondering why nothing changed.

### 4. `percentageOfNodesToScore`, and why GPU Pods pay full price

The scheduler does not filter every node for every Pod. Once it has found "enough" feasible nodes it stops and moves to scoring. The threshold comes from `numFeasibleNodesToFind`, and the logic is short enough to reproduce exactly:

```go
const (
    minFeasibleNodesToFind           = 100
    minFeasibleNodesPercentageToFind = 5
)

func (sched *Scheduler) numFeasibleNodesToFind(percentageOfNodesToScore *int32, numAllNodes int32) int32 {
    if numAllNodes < minFeasibleNodesToFind {   // <100 nodes: always scan everything
        return numAllNodes
    }
    percentage := /* profile value, else global value */
    if percentage == 0 {                        // 0 == "auto"
        percentage = int32(50) - numAllNodes/125
        if percentage < minFeasibleNodesPercentageToFind {
            percentage = minFeasibleNodesPercentageToFind
        }
    }
    numNodes := numAllNodes * percentage / 100
    if numNodes < minFeasibleNodesToFind {
        return minFeasibleNodesToFind
    }
    return numNodes
}
```

Read the auto formula. `50 − N/125`, floored at 5:

| Cluster size | Auto percentage | Feasible-node target |
|---|---|---|
| 100 nodes | 50 − 0 = 50% | 100 (the floor) |
| 500 nodes | 50 − 4 = 46% | 230 |
| 1,000 nodes | 50 − 8 = 42% | 420 |
| 2,500 nodes | 50 − 20 = 30% | 750 |
| 5,000 nodes | 50 − 40 = 10% | 500 |
| 10,000 nodes | 50 − 80 → floor 5% | 500 |

The scheduler also does not restart from node 0 each time. It resumes from `nextStartNodeIndex` and wraps modulo the node count, so successive Pods sweep different parts of the fleet and placement stays reasonably spread even though scoring only sees a sample.

**Now the part that matters for GPUs.** The cap is on **feasible nodes found**, not on nodes examined. The loop stops early only when it has *accumulated* that many nodes that passed Filter. Work it through:

```
  Fleet: 5,000 nodes, of which 200 have GPUs. Auto percentage = 10%.
  Target feasible nodes = max(100, 5000 × 10%) = 500.

  A CPU-only Pod:
      almost every node passes Filter → 500 feasible found after ~510 examined
      → 10% of the fleet examined. Cheap.

  A `nvidia.com/gpu: 4` Pod:
      at most 200 nodes can EVER pass Filter → the target of 500 is unreachable
      → the loop runs to completion → ALL 5,000 nodes examined, every cycle.

  Cost, with a stated assumption of ~0.2 ms of Filter work per node across the
  default plugin set, at the default parallelism of 16 goroutines:

      5,000 nodes × 0.2 ms / 16  ≈  62.5 ms of Filter, per GPU Pod
      vs.
        510 nodes × 0.2 ms / 16  ≈   6.4 ms of Filter, per CPU Pod

      ≈ 10× more scheduler CPU per Pod, purely because the resource is scarce.
```

Scale that to a job. A 512-Pod distributed training job is 512 serial scheduling cycles:

```
  512 Pods × 62.5 ms  ≈  32 s of pure Filter time, serialized,
                          during which NOTHING ELSE in the cluster is scheduled
                          (the scheduling cycle is single-threaded per profile).
```

And if the job cannot fit, that 32 s is paid **again on every backoff retry**, with backoff capped at `podMaxBackoffSeconds: 10`. Five hundred and twelve Pods cycling through `unschedulableQ` → `backoffQ` → `activeQ` → fail → repeat is a scheduler pegged at 100% CPU while the cluster appears to be doing nothing.

**That is a scheduler-throughput argument for Kueue that has nothing to do with GPU parking.** If the Job stays suspended until quota exists, those 512 Pods are never created, never queued, and never filtered. The relief is not "we avoid idle GPUs" — it is "we avoid 512 × 5,000 futile Filter evaluations on a 10-second loop."

Two knobs, and their honest limits:

- Setting `percentageOfNodesToScore: 100` changes *nothing* for the GPU Pod above — it was already scanning everything. It only ever helps by raising quality for Pods that currently stop early.
- Setting it lower (say 5) helps CPU Pods and is irrelevant to GPU Pods. The real lever for a scarce-resource fleet is a **separate scheduler profile** (a distinct `schedulerName` that GPU Pods select via `.spec.schedulerName`) so GPU scheduling latency does not serialize behind general workload scheduling, or partitioning the fleet.

### 5. How `nvidia.com/gpu` actually reaches the scheduler

The scheduler reads an integer out of `node.status.allocatable`. Here is the full path from silicon to that integer, and it is a gRPC conversation over a Unix socket that has nothing to do with the API server.

Extended resources are advertised in one of two ways. Manually, by PATCHing the Node's status subresource — worth knowing because it shows the shape of what the device plugin automates:

```
PATCH /api/v1/nodes/gpu-node-07/status
Content-Type: application/json-patch+json

[{"op": "add",
  "path": "/status/capacity/example.com~1foo",     ← ~1 is the JSON-Pointer escape for "/"
  "value": "5"}]
```

Or, for real hardware, by a **device plugin**: a DaemonSet pod that serves gRPC on a Unix socket in `/var/lib/kubelet/device-plugins/` and registers itself with kubelet. That directory path is hardcoded and not affected by kubelet configuration.

```
 NODE gpu-node-07                                       CONTROL PLANE
 ═══════════════                                        ═════════════

 ┌─ nvidia-device-plugin pod (DaemonSet) ─┐   ┌─ kubelet ──────────┐
 │  hostPath: /var/lib/kubelet/           │   │ Device Manager     │
 │            device-plugins              │   │ + Topology Manager │
 └────────────────────────────────────────┘   └────────────────────┘
              │                                        │
   ① start serving gRPC FIRST, on its own socket:      │
      /var/lib/kubelet/device-plugins/nvidia.sock      │
      (a plugin MUST serve before it registers)        │
              │                                        │
   ② Register(RegisterRequest{                         │
        version:       "v1beta1",         ───────────▶ │  over
        endpoint:      "nvidia.sock",                  │  kubelet.sock
        resource_name: "nvidia.com/gpu",               │  (the well-known
        options: DevicePluginOptions{                  │   registration socket)
          pre_start_required: false,                   │
          get_preferred_allocation_available: true}})  │
              │                                     ── Empty ──▶ ok
              │                                        │
   ③ kubelet dials BACK to nvidia.sock and opens the long-lived stream:
              │ ◀──────── ListAndWatch(Empty) ──────── │
              │                                        │
      stream ListAndWatchResponse{ devices: [           │
        Device{ID:"GPU-fef8...", health:"Healthy",      │
               topology:{nodes:[{ID:0}]}},   ← NUMA hint, feeds Topology Manager
        Device{ID:"GPU-11a2...", health:"Healthy",      │
               topology:{nodes:[{ID:0}]}},              │
        Device{ID:"GPU-3c9d...", health:"Unhealthy",    │
               topology:{nodes:[{ID:1}]}},              │
        ... ] }                             ──────────▶ │
              │  (re-sent on ANY change: ECC error,     │
              │   card fell off the bus, driver reload) │
              │                                        │
              │                            kubelet counts HEALTHY devices
              │                            and writes:
              │                              status.capacity["nvidia.com/gpu"]    = 8
              │                              status.allocatable["nvidia.com/gpu"] = 7
              │                                        │  ── PATCH /nodes/X/status ──▶ apiserver
              │                                        │                                │
              │                                        │                                ▼
              │                                        │                        ┌──────────────┐
              │                                        │                        │  SCHEDULER   │
              │                                        │   NodeResourcesFit reads a single     │
              │                                        │   int64 out of ScalarResources:       │
              │                                        │     {"nvidia.com/gpu": 7}             │
              │                                        │   That is the ENTIRE model. No memory │
              │                                        │   size, no SKU, no NVLink, no NUMA.   │
              │                                        └──────────────┬───────────┘
              │                                        │              │ binds Pod → gpu-node-07
              │                                        │ ◀────────────┘
              │                                        │
   ④ kubelet's Device Manager picks WHICH device IDs to hand over.
      Optionally asks the plugin for a hint first:
              │ ◀── GetPreferredAllocation(available, must_include, size=4) ──
              │ ──▶ ContainerPreferredAllocationResponse{deviceIDs: [...]}
              │     (NVIDIA uses this to prefer NVLink-connected peers.
              │      Device Manager is NOT obliged to follow it.)
              │                                        │
   ⑤          │ ◀── Allocate(ContainerAllocateRequest{devices_ids:[4 IDs]}) ──
              │ ──▶ ContainerAllocateResponse{
              │       envs: {"NVIDIA_VISIBLE_DEVICES": "GPU-fef8,GPU-11a2,..."},
              │       devices: [DeviceSpec{host_path:"/dev/nvidiactl",
              │                            container_path:"/dev/nvidiactl",
              │                            permissions:"rw"}, ...],
              │       mounts:  [Mount{host_path:"/usr/lib/.../libnvidia-ml.so.1", ...}],
              │       annotations: {...},
              │       cdi_devices: [CDIDevice{name:"nvidia.com/gpu=0"}] }
              │                                        │
   ⑥ optionally  ◀── PreStartContainer(devices_ids) ── (only if pre_start_required;
              │      e.g. reset/clean the card first)
              │                                        │
              │                       kubelet merges envs/mounts/devices into the
              │                       CRI ContainerConfig and calls CreateContainer.
              ▼                                        ▼
        container starts with /dev/nvidia* present and CUDA able to see 4 cards.
```

The complete gRPC surface, which is small enough to memorise:

| Service | RPC | Direction | Purpose |
|---|---|---|---|
| `Registration` | `Register(RegisterRequest) → Empty` | plugin → kubelet | announce socket, API version, resource name, options |
| `DevicePlugin` | `GetDevicePluginOptions(Empty) → DevicePluginOptions` | kubelet → plugin | does it need `PreStartContainer`? does it implement `GetPreferredAllocation`? |
| `DevicePlugin` | `ListAndWatch(Empty) → stream ListAndWatchResponse` | kubelet → plugin | the device inventory, pushed on every change |
| `DevicePlugin` | `GetPreferredAllocation(PreferredAllocationRequest) → PreferredAllocationResponse` | kubelet → plugin | *advisory* hint on which IDs to pick |
| `DevicePlugin` | `Allocate(AllocateRequest) → AllocateResponse` | kubelet → plugin | the binding decision: envs, mounts, device nodes, annotations, CDI names |
| `DevicePlugin` | `PreStartContainer(PreStartContainerRequest) → PreStartContainerResponse` | kubelet → plugin | device-specific prep before the container runs |

Message shapes worth knowing by heart: `Device{string ID; string health; TopologyInfo topology}` where `TopologyInfo{repeated NUMANode nodes}` and `NUMANode{int64 ID}`; and `ContainerAllocateResponse{map<string,string> envs; repeated Mount mounts; repeated DeviceSpec devices; map<string,string> annotations; repeated CDIDevice cdi_devices}`.

Three operational facts that fall out of this design:

- **Health is a count, not a label.** When a card goes unhealthy, the plugin re-sends the whole device list and kubelet *decrements allocatable*. The scheduler learns "7 instead of 8", never "GPU-3c9d has ECC errors." Correlating a capacity drop with a specific card is a DCGM/`nvidia-smi` job, not a Kubernetes one.
- **The plugin must serve before it registers.** If it registers first, kubelet's dial-back to `nvidia.sock` fails and registration is rejected. This ordering is in the spec and is the usual cause of a plugin that "starts fine" but never advertises.
- **Kubelet restart re-runs the handshake.** Kubelet recreates `kubelet.sock` on start; plugins are expected to watch for that and re-register. A plugin that does not is why GPUs sometimes vanish from a node after a kubelet restart until the DaemonSet pod is deleted.

### 6. What the integer forecloses

The scheduler's model of a GPU is `map[v1.ResourceName]int64`. `NodeResourcesFit` sums the Pod's scalar requests in `PreFilter` and, in `Filter`, checks each against `allocatable − requested`. That is it. From that model, four hard rules follow, all of which are *enforced by the API server*, not conventions:

1. **Integers only.** `nvidia.com/gpu: 0.5` is rejected at admission. There is no millicore equivalent.
2. **Requests must equal limits.** If both are set for an extended resource they must match; setting only `limits` copies it to `requests`.
3. **No overcommit.** Unlike CPU, the scheduler will never place a Pod whose extended-resource request exceeds what remains.
4. **Not shareable between containers.** Two containers in the same Pod each requesting `nvidia.com/gpu: 1` consume two cards.

The real request YAML, exactly as it must be written:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: trainer
spec:
  restartPolicy: Never
  containers:
  - name: train
    image: nvcr.io/nvidia/pytorch:24.10-py3
    command: ["python", "train.py"]
    resources:
      limits:
        nvidia.com/gpu: 4        # requests is set to 4 implicitly. You may write
                                 # requests: {nvidia.com/gpu: 4} but it MUST match.
        cpu: "16"                # CPU and memory are normal, divisible, overcommittable
        memory: 128Gi            # resources — the asymmetry is the whole point.
      requests:
        cpu: "16"
        memory: 128Gi
  tolerations:
  - key: nvidia.com/gpu          # GPU nodes are usually tainted so CPU workloads
    operator: Exists             # cannot land on $30/hour hardware.
    effect: NoSchedule
```

And the node side that the scheduler reads:

```
$ kubectl get node gpu-node-07 -o jsonpath='{.status.allocatable}' | jq
{
  "cpu": "126",
  "ephemeral-storage": "3500Gi",
  "memory": "2113929216Ki",
  "nvidia.com/gpu": "7",          ← one card is unhealthy; capacity says 8
  "pods": "110"
}
```

**What that model cannot express**, and every item on this list is a real cost or performance decision:

- "a GPU with at least 40 GB of memory" — an A100-40GB and an H100-80GB are both `1`.
- "compute capability ≥ 9.0" — a T4 and an H100 are both `1`.
- "a MIG `3g.40gb` slice" — a slice and a whole card are both `1` unless you invent separate resource names.
- "four cards on the same NVLink domain" — adjacency is invisible.
- "the cheap card, not the expensive one" — price is invisible.
- "this card, on NUMA node 0, matching the NIC" — see §7.

The workarounds that exist today, and precisely where they stop:

**Time-slicing** (NVIDIA device plugin `sharing.timeSlicing`) advertises each physical GPU as N replicas. It is *oversubscription by lying to the scheduler*: the count goes up, the hardware does not.

```yaml
# ConfigMap consumed by the NVIDIA device plugin
version: v1
sharing:
  timeSlicing:
    renameByDefault: true            # advertise as nvidia.com/gpu.shared instead
    failRequestsGreaterThanOne: true # a container asking for >1 shared replica is
                                     # rejected with UnexpectedAdmissionError — because
                                     # 2 replicas of the same card is not 2 GPUs
    resources:
    - name: nvidia.com/gpu
      replicas: 4                    # one card now reports as 4
```

There is **no memory isolation and no fault isolation** between time-sliced tenants: one process OOMs the card and every co-tenant dies. It is right for bursty inference and development, wrong for anything with an SLO.

**MIG** (Multi-Instance GPU, A100/H100/H200 and later) is a real hardware partition — up to 7 instances per card, each with its own memory slice, L2 partition, and SM allocation, with fault isolation. The device plugin exposes it in three strategies:

| `migStrategy` | Advertises | When it works |
|---|---|---|
| `none` (default) | `nvidia.com/gpu` only; MIG devices are ignored | no MIG in the fleet |
| `single` | MIG instances as plain `nvidia.com/gpu` | every card on a node has an *identical* MIG profile |
| `mixed` | one resource name per profile: `nvidia.com/mig-1g.10gb`, `nvidia.com/mig-3g.40gb`, … | heterogeneous profiles |

`mixed` is the honest one, and it exposes the model's limit precisely: every distinct profile becomes a *separate, non-fungible resource name*. A Pod requesting `nvidia.com/mig-1g.10gb` cannot be satisfied by an idle `nvidia.com/mig-2g.20gb`, even though the hardware could serve it. On a fleet with three GPU generations and four MIG profiles you now have a dozen independent, mutually-unsubstitutable integer pools, each fragmenting on its own. **That combinatorial explosion is the concrete reason DRA exists.**

### 7. Topology Manager: what kubelet aligns and the scheduler cannot see

`Device.topology.nodes[].ID` in the `ListAndWatch` response is the NUMA node a device hangs off. The **Topology Manager** in kubelet is what uses it, and it is the piece most people miss.

The problem: on a dual-socket node, a GPU on NUMA node 1 talking to a container whose CPUs were pinned to NUMA node 0 pays a cross-socket hop on every host-to-device transfer. On real training workloads that shows up as measurably reduced effective PCIe bandwidth and jittery step times.

The mechanism: kubelet's *Hint Providers* — CPU Manager, Memory Manager, and Device Manager — each report bitmasks of which NUMA nodes could satisfy the container's request and whether that allocation is "preferred." Topology Manager intersects the masks and applies a policy:

| Policy | Behaviour on a non-preferred alignment |
|---|---|
| `none` (default) | no alignment attempted at all |
| `best-effort` | store the preferred hint, **admit the Pod anyway** |
| `restricted` | **reject the Pod** from the node with a `TopologyAffinityError` |
| `single-numa-node` | require all resources on one NUMA node; reject otherwise |

Scope: `container` (default, aligns each container independently) or `pod` (aligns all containers in the Pod to one NUMA node set — what you want for a latency-sensitive multi-container Pod).

Policy options: `prefer-closest-numa-nodes` biases `best-effort`/`restricted` toward NUMA sets with shorter inter-node distances; `max-allowable-numa-nodes` overrides the default limit of **8** NUMA nodes (raising it is explicitly documented as unsupported and risky, because hint enumeration is combinatorial in the number of nodes).

**The gap you must be able to name in an interview:** *the scheduler is not topology-aware.* It places the Pod on a node using the integer count; the Topology Manager then evaluates alignment on that node; and under `restricted` or `single-numa-node`, the Pod is **rejected after binding**. The symptom is a Pod that reaches the node and then fails admission:

```
$ kubectl describe pod trainer
...
Status:   Failed
Reason:   TopologyAffinityError
Message:  Resources cannot be allocated with Topology locality
Events:
  Warning  TopologyAffinityError  kubelet  Resources cannot be allocated with Topology locality
```

Because the Pod was already bound, this is not a retry with a different node — the Pod is terminal, and the workload controller must create a new one, which the scheduler may well send to the same node again. This is a genuine hole in the count model, and it is one of the specific things DRA is designed to close by moving device selection into the scheduler where a rejection is just another failed Filter.

### 8. DRA — requesting devices by attribute (GA in 1.34)

**Dynamic Resource Allocation** graduated to **GA in Kubernetes 1.34** (September 2025) on **KEP-4381, "DRA structured parameters."** The core API is `resource.k8s.io/v1`, with several newer sub-features still at `v1beta1`/`v1beta2` behind their own gates.

History worth knowing for interviews, because it explains the shape: the *original* DRA design (**KEP-3063**) had the scheduler call out to a vendor-supplied controller mid-scheduling to ask "can you satisfy this claim on this node?" That was withdrawn and superseded during the 1.32 cycle. The GA design instead has vendors **publish structured data** that the scheduler reads and reasons about in-process. It is the same lesson as watch-versus-poll from lesson 01, applied one layer up: **put the data where the decision-maker can read it directly, rather than making the decision-maker call out mid-decision.** A vendor controller in the scheduling cycle is a synchronous external dependency in a serial loop — the same class of hazard as the admission webhook in lesson 08.

The object model:

| Object | Scope | Written by | Role |
|---|---|---|---|
| `ResourceSlice` | cluster | the vendor's **DRA driver** (a DaemonSet, one per node) | the inventory: devices on this node *and their attributes and capacities* |
| `DeviceClass` | cluster | admin | a "kind of device" plus default selectors and config. `StorageClass`, but for devices |
| `ResourceClaim` | namespaced | user or generated | the request: device requests with CEL selectors over attributes |
| `ResourceClaimTemplate` | namespaced | user | stamps out a fresh claim per Pod, like a PVC template |
| `DeviceTaintRule` | cluster | admin | taint devices for maintenance so claims avoid them |

What that buys you, concretely:

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu.nvidia.com
spec:
  selectors:
  - cel:
      expression: device.driver == "gpu.nvidia.com"
---
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: big-memory-gpu
  namespace: team-a
spec:
  spec:
    devices:
      requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 4
          selectors:
          - cel:
              # THIS is the thing the integer model cannot say.
              expression: >-
                device.capacity["gpu.nvidia.com"]["memory"].compareTo(quantity("40Gi")) >= 0 &&
                device.attributes["gpu.nvidia.com"]["productName"].startsWith("NVIDIA A100")
---
apiVersion: v1
kind: Pod
metadata:
  name: trainer
  namespace: team-a
spec:
  containers:
  - name: train
    image: nvcr.io/nvidia/pytorch:24.10-py3
    resources:
      claims:
      - name: gpu           # ← binds this container to the claim below
  resourceClaims:
  - name: gpu
    resourceClaimTemplateName: big-memory-gpu
```

Where it hooks into §2's extension points — and note that it hooks in at exactly the places the count model had nothing:

- **PreFilter** — parse the claim, fail fast if no `ResourceSlice` anywhere could satisfy it.
- **Filter** — for each candidate node, evaluate the claim's CEL selectors against that node's published `ResourceSlice` devices. Nodes without a matching device are removed, exactly like an insufficient count would remove them — but on *attributes*.
- **Reserve** — tentatively allocate a *specific device* in the scheduler's cache, so the next Pod in the serial cycle does not double-book it. Devices are considered in lexicographic order by slice and pool name with a first-fit strategy.
- **PreBind** — write the finalized allocation into `ResourceClaim.status`, so kubelet and the driver on the node know exactly which device to wire up.

**MIG becomes an attribute, not a resource name.** Under DRA, a `1g.10gb` slice is a device with `attributes` describing its profile and `capacity` describing its memory, published in the same `ResourceSlice` as the whole cards. A claim selects it with CEL. The dozen mutually-unsubstitutable integer pools from §6 collapse into one queryable inventory, and "the cheapest device that satisfies ≥40 GB" becomes expressible for the first time. That is the substrate a cost-aware policy needs: **you cannot optimize spend across a fleet you can only count.**

Two GA-era limits to state honestly, because they will be asked:

- **DRA resources are not preemptible.** The scheduler will not evict a Pod to free a `ResourceClaim`, so a high-priority job waits for organic release rather than reclaiming.
- **DRA requires 1.34+ *and* a DRA-aware vendor driver.** On a fleet with mixed Kubernetes versions, the device-plugin path is still live and must be designed for. Both paths coexist; in recent releases the scheduler even delegates certain extended-resource names to DRA when a driver claims them (`shouldDelegateResourceToDRA` in `NodeResourcesFit`), so the two models are converging rather than one replacing the other overnight.

### 9. Kueue — quota, gang, and topology-aware admission

**Kueue** (`kueue.x-k8s.io`) is a Kubernetes-native job queueing layer that sits *in front of* the scheduler. It does not place Pods. It decides whether a whole workload is **admitted**, and until then keeps the Job **suspended** (`.spec.suspend: true`), which means **no Pods are created at all**.

The object model:

| Object | Scope | Role |
|---|---|---|
| `ResourceFlavor` | cluster | how heterogeneous nodes are modelled: `nodeLabels` select A100 vs H100, spot vs on-demand. Quota is assigned *per flavor*. |
| `ClusterQueue` | cluster | the quota pool, per flavor, with `nominalQuota`, `borrowingLimit`, `lendingLimit`, preemption policy, and a `namespaceSelector` controlling who may use it. |
| `Cohort` | cluster | a set of ClusterQueues that lend each other unused quota. |
| `LocalQueue` | namespaced | a tenant-facing handle pointing at a ClusterQueue. Jobs reference this. |
| `Workload` | namespaced | Kueue's internal wrapper around a Job (or RayJob, JobSet, MPIJob…), carrying its **pod sets**. |
| `AdmissionCheck` | cluster | lets an external component gate admission (provisioning requests, multi-cluster). |

A real configuration, with GPU classes as flavors — this is the piece that makes Kueue cost-policy-relevant rather than merely capacity-relevant:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata: {name: a100-40gb}
spec:
  nodeLabels: {cost.example.com/gpu-class: a100-40gb}
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata: {name: h100-80gb}
spec:
  nodeLabels: {cost.example.com/gpu-class: h100-80gb}
---
apiVersion: kueue.x-k8s.io/v1beta2
kind: ClusterQueue
metadata: {name: team-a-cq}
spec:
  namespaceSelector:
    matchLabels: {team: team-a}
  cohort: research                    # may borrow from team-b-cq's idle quota
  queueingStrategy: BestEffortFIFO    # or StrictFIFO — head-of-line blocking on/off
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu", "cpu", "memory"]
    flavors:
    - name: a100-40gb
      resources:
      - {name: "nvidia.com/gpu", nominalQuota: 32, borrowingLimit: 16}
      - {name: "cpu",            nominalQuota: 512}
      - {name: "memory",         nominalQuota: 4Ti}
    - name: h100-80gb
      resources:
      - {name: "nvidia.com/gpu", nominalQuota: 8, borrowingLimit: 0}   # the expensive
      - {name: "cpu",            nominalQuota: 128}                    # tier is rationed
      - {name: "memory",         nominalQuota: 1Ti}                    # and unborrowable
  preemption:
    reclaimWithinCohort: LowerPriority
    withinClusterQueue: LowerPriority
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata: {namespace: team-a, name: training}
spec: {clusterQueue: team-a-cq}
```

**Gang / all-or-nothing admission** is the direct answer to §4's throughput problem and to GPU parking. A `Workload` carries one or more `podSets`, each with a `count` and a `spec`; Kueue computes total demand as `Σ (requests × count)` and admits the Workload only when quota for **every** pod set can be reserved **at once**. Until then the Job stays suspended and zero Pods exist. Kueue also handles the "admitted but never became ready" case: the Workload is evicted and requeued with exponential backoff, tracked in `.status.requeueState.count`.

**Topology-Aware Scheduling (TAS)** lets Kueue admit with a placement intent over a `Topology` hierarchy (rack, block, node) referenced from a `ResourceFlavor`: `required` (all Pods in one domain — what tight all-reduce needs), `preferred` (same domain if possible, else spread), or `unconstrained`. On a fabric where inter-node bandwidth is 10–20× lower than intra-node NVLink, this is the difference between a training job hitting its expected step time and quietly running at half speed.

The mental model, one line: **Kueue gates the door, the scheduler seats the guests, DRA picks the exact chair.**

### 10. The frontier: gang scheduling moving in-tree

Worth knowing because it will come up as "is Kueue still necessary?": Kubernetes 1.36 added an alpha `PodGroup` API (`scheduling.k8s.io/v1alpha1`, feature gate `GenericWorkload`) that evaluates a group of Pods as a single unit with a `minCount`, plus two new scheduler extension points behind the `TopologyAwareWorkloadScheduling` gate:

- **PlacementGenerate** — generate candidate *placements* (subsets of nodes theoretically feasible for the whole group). The `TopologyPlacement` plugin groups nodes by the distinct values of a requested topology key.
- **PlacementScore** — score those placements. `NodeResourcesFit` scores with `MostAllocated` bin-packing; `PodGroupPodsCount` scores by how many of the group's Pods a placement can actually take.

This is the framework growing a native notion of "a group of Pods with a topology preference" — the thing §2 showed it structurally lacked. It is alpha, its documented limitation is that placement generation has deterministic ordering constraints and may miss a valid placement a different ordering would have found, and it does **not** replace Kueue's quota, fair-sharing, borrowing, or multi-tenant admission model. The honest 2026 answer in an interview: gang *placement* is coming in-tree; gang *admission with quota* is still Kueue's job.

### 11. Where a cost signal can actually attach

Three integration points, three different blast radii, and this is the substance of the Practice design doc:

```
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ ADMISSION LAYER — Kueue                                                      │
 │   ClusterQueue quota per ResourceFlavor (a100-40gb: 32, h100-80gb: 8)        │
 │   ▸ COST HOOK: ration the expensive flavor. Hard cap, job-granular.          │
 │   ▸ Cannot pick a node. Cannot express "prefer cheap among feasible."        │
 │   ▸ Build cost: ZERO code. Just YAML your operator can reconcile.            │
 └────────────────────────────────┬─────────────────────────────────────────────┘
                                  │  Job unsuspended → Pods created
                                  ▼
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ PLACEMENT LAYER — kube-scheduler                                             │
 │   Filter: hard feasibility (count / DRA selectors / taints / affinity)       │
 │   Score:  soft ranking, 0–100 × weight                                       │
 │   ▸ COST HOOK: a Score plugin reading a node annotation your operator writes │
 │   ▸ Soft bias only — CANNOT enforce a cap (Score never removes a node)       │
 │   ▸ Build cost: an out-of-tree plugin + a scheduler build + a rollout to     │
 │     40 clusters. Highest blast radius in the whole stack.                    │
 └────────────────────────────────┬─────────────────────────────────────────────┘
                                  │  node chosen
                                  ▼
 ┌──────────────────────────────────────────────────────────────────────────────┐
 │ DEVICE LAYER — DRA (1.34+) or device plugin                                  │
 │   DRA: DeviceClass selectors + ResourceClaim CEL over published attributes   │
 │   ▸ COST HOOK: a DeviceClass per price tier; claims select the cheap class   │
 │   ▸ Most precise — MIG-aware, per-device. Requires 1.34+ and a DRA driver.   │
 │   ▸ Policy lives in DeviceClass/claim definitions, i.e. FAR from a central   │
 │     operator — which is a governance problem, not a technical one.           │
 └──────────────────────────────────────────────────────────────────────────────┘
```

Note that these are not alternatives so much as a layered set with different enforcement semantics: **Kueue enforces, Score suggests, DRA selects.** A design that needs a hard monthly budget *and* a soft preference for cheap cards *and* MIG-aware right-sizing needs all three, and the design doc's job is to say which one ships first and why.

## Perspectives

**Scheduler-internals perspective.** Filter-versus-Score is a hard-gate-versus-soft-bias distinction that generalizes far past GPUs — topology spread, taints, affinity, and any future extension work all reduce to it. Internalizing it here means you will never again encode a preference as a predicate.

**Device-modelling perspective.** DRA is the same upgrade as moving from an untyped config blob to a typed API: from "an opaque integer that means whatever the operator remembers it means" to "a set of typed, queryable attributes with a schema." It is lesson 02's API-machinery discipline applied to hardware, and the KEP-3063 → KEP-4381 pivot is a design lesson in its own right: read published data, do not call out mid-decision.

**Admission/quota perspective.** Kueue answers "should this job start at all, right now" — a fundamentally different question from "which node for this Pod." Conflating them is the single most common design-review mistake at this layer, and a reviewer who hears "Kueue schedules the Pod" should push back immediately.

**Throughput perspective.** The §4 arithmetic reframes gang admission as a *scheduler-capacity* argument, not only an efficiency one. Five hundred and twelve Pods that cannot fit, retried on a 10-second backoff against 5,000 nodes each, is a self-inflicted denial of service on your own control plane. Suspending the Job is the cheapest fix available.

**Economics/FinOps perspective (the differentiator).** This is where `gpu-cost-operator`'s signal has somewhere to go. A `costScore` can bias Score (per-Pod, soft), shape ResourceFlavor quota (per-job, hard), or select DeviceClasses (per-device, precise). Three real integration points, each with a different blast radius and build cost. Being fluent in *both* the scheduling internals and the cost model is an intersection almost nobody in the interview pool occupies.

## Real-world use cases

- **CoreWeave, "Kueue: A Kubernetes-Native System for AI Training Workloads"** — <https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads>. **What it shows:** Kueue running in production on CoreWeave Kubernetes Service for customer training and batch-inference workloads — gang admission and multi-tenant quota at real GPU-fleet scale, at exactly the kind of company this module targets. Read it for how ClusterQueue/cohort structure maps onto customer tenancy.
- **Google Cloud, "Kubernetes device management with DRA"** — <https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation>. **What it shows:** the ResourceSlice → DeviceClass → ResourceClaim flow on GKE from the platform provider's side, including how the driver publishes attributes. Useful as a second, independent rendering of §8's object model.
- **NVIDIA's DRA driver for GPUs, donated to the Kubernetes community** — <https://blogs.nvidia.com/blog/nvidia-at-kubecon-2026/>. **What it shows:** the "DRA driver as a DaemonSet" concept made concrete by the vendor that matters most here — the component that walks a node's physical GPUs and MIG configuration and writes them out as `ResourceSlice` objects.
- **AKS Engineering, "Running more with less: MIG with DRA on AKS"** — <https://blog.aks.azure.com/2026/03/03/multi-instance-gpu-with-dra-on-aks>. **What it shows:** the MIG-plus-DRA pairing this lesson calls the concrete cost lever, in production on a third cloud — the clearest available demonstration that §6's "dozen unsubstitutable integer pools" problem collapses under an attribute model.
- **Uber Engineering, "Uber's Journey to Ray on Kubernetes: Resource Management"** — <https://www.uber.com/en-DE/blog/ubers-journey-to-ray-on-kubernetes-resource-management/>. **What it shows:** a hyperscaler solving the same GPU pooling problem with a custom layer on top of Kubernetes rather than adopting Kueue. A useful contrast case for "how would you decide build-versus-adopt at this layer" — and for the reality that at sufficient scale, people do build.

## Worked example

One 8-Pod, 8-GPU distributed training job on a heterogeneous cluster: 5,000 nodes total, of which 180 are A100-40GB and 20 are H100-80GB. Narrate all three layers, then quantify.

**1 · Submit.** The user creates a `Job` with `parallelism: 8`, each Pod claiming one GPU with at least 40 GB via a `ResourceClaimTemplate`, and a `kueue.x-k8s.io/queue-name: training` label pointing at a LocalQueue. Kueue's Job integration immediately sets `.spec.suspend: true`. **Zero Pods exist.** Nothing is in the scheduler's queue; nothing is being filtered.

**2 · Kueue admission.** Kueue wraps the Job as a `Workload` with one pod set of `count: 8`. The team's ClusterQueue has 32 `nvidia.com/gpu` of `a100-40gb` quota, of which 28 are in use — 4 free. All-or-nothing: 8 > 4, so the Workload waits with condition `QuotaReserved: False`. No GPUs are parked, no partial placement, and critically **no Pods are entering the scheduler's queue to be filtered against 5,000 nodes on a 10-second retry loop.** When 4 more free up, quota for all 8 is reservable at once; Kueue sets `suspend: false` and the Job controller creates the Pods.

**3 · Pods created → scheduler.** Eight Pods hit `activeQ`. For each, serially:

- **PreFilter** — the DRA plugin parses the claim; `NodeResourcesFit` builds the scalar request state.
- **Filter** — 4,800 nodes have no GPU device at all and fail immediately. Of the 200 GPU nodes, the DRA plugin evaluates each candidate's published `ResourceSlice` devices against the claim's CEL selector (`memory ≥ 40Gi`); both A100-40GB and H100-80GB devices satisfy it. Nodes with no free matching device fail. Feasible set: say 96 nodes.
- Because 96 < the 500-node feasible target from §4, the loop examined **all 5,000 nodes** — ≈62.5 ms of Filter per Pod, ≈0.5 s across the eight.

**4 · Score.** `gpu-cost-operator` has annotated each node with a normalized `costScore` (100 = cheapest tier). A Score plugin reads it and, after `NormalizeScore`, A100 nodes score 100 and H100 nodes score ~49 (2.50/5.12 inverted and rescaled). At weight 5 that is a 255-point spread, comfortably above the ~200-point swing the default plugins can produce between similar nodes — so the job lands on A100s. **But no node was removed**: if every A100 were busy, the H100s are still feasible and the job still runs.

**5 · Reserve → Permit → PreBind → Bind.** DRA's plugin reserves a specific device per Pod in the scheduler cache; the default profile has no Permit plugin so nothing waits (the gang guarantee was already provided upstream by Kueue); `PreBind` writes each allocation into its `ResourceClaim.status`; `Bind` posts the Binding. Kubelet on each node sees its Pod, the DRA driver wires up the exact allocated device, and the container starts.

**The cost delta, worked.** Using illustrative 2025–2026 GPU-cloud list rates — A100-40GB at **$2.50/GPU-hour**, H100-80GB at **$5.12/GPU-hour** (both vary by provider, region, and commitment; substitute your own):

```
  Job: 8 GPUs × 4 hours = 32 GPU-hours

  Score correctly biases onto A100s (the job asked for ≥40 GB and fits):
      32 GPU-h × $2.50  =  $ 80.00

  Score absent, miscalibrated, or drowned out by weight-1 dilution — the job
  lands wherever NodeResourcesFit's bin-packing happened to prefer, say all H100:
      32 GPU-h × $5.12  =  $163.84

  Delta per job                          =  $ 83.84   (2.05×)
  × 400 such jobs / month                =  $33,536 / month
  × 12                                   = $402,432 / year
```

Two honest caveats that belong in the design doc, not omitted from it. First, the H100 is not merely more expensive — it is roughly 2–3× faster on transformer training, so if the job would finish in 1.6 hours instead of 4, the H100 run costs `8 × 1.6 × $5.12 = $65.54` and is *cheaper*. **A cost signal that ignores throughput is a wrong signal.** The correct normalized metric is dollars per unit of work (per token, per step, per epoch), not dollars per GPU-hour, and deriving it requires a performance model your operator does not have in v0.1. Second, if the fleet's A100s are the *contended* tier, biasing everything onto them increases queueing delay, and idle-engineer time is also money.

State both caveats explicitly. A design doc that says "prefer cheaper $/GPU-hour" without them is the kind of thing a CoreWeave interviewer takes apart in ninety seconds.

**Where each decision lived** — the sentence to be able to say cold: admission and gang = **Kueue**; node = **scheduler Filter (hard) then Score (soft)**; which physical device = **DRA**; and NUMA alignment of that device to the container's CPUs = **kubelet's Topology Manager**, after binding, where a mismatch is a terminal `TopologyAffinityError` rather than a retry. Four layers, four owners.

## Practice

**Deliverable: a design doc, committed** at `../practice/gpu-cost-operator/docs/design.md`.

Title: **"How would `GPUCostPolicy` influence GPU placement?"** Your operator reconciles a `GPUCostPolicy` CR and emits a per-node (or per-device) **costScore** signal. Design how that signal reaches a placement decision, comparing **three** approaches with explicit tradeoffs:

1. **Score-plugin approach** — the operator writes `costScore` as a Node annotation or label; an out-of-tree scheduler Score plugin (or a `scheduler-plugins` configuration) reads it to bias placement. Cover: the plugin's `Score` and `NormalizeScore` implementation sketch; the **weight** you would assign and why, argued against the default weights (`TaintToleration: 3`, `NodeAffinity: 2`, `PodTopologySpread: 2`, `InterPodAffinity: 2`, `NodeResourcesFit: 1`, `NodeResourcesBalancedAllocation: 1`, `ImageLocality: 1`); how you would roll a custom scheduler binary to 40 clusters and what the blast radius is; and why this can bias but never cap.
2. **Kueue-quota approach** — encode cost policy as `ClusterQueue` quota per `ResourceFlavor`, with cohort borrowing so cheap capacity is shared and expensive capacity is rationed (`borrowingLimit: 0` on the H100 flavor). Cover: what a hard monthly budget maps onto; why this is admission-time and therefore coarser than node selection; and that it needs zero custom code, which is the strongest argument in its favour.
3. **DRA-attributes approach** — express cost preference through `DeviceClass` selection and `ResourceClaim` CEL selectors so claims target cheaper device classes or MIG profiles. Cover: what the driver must publish for this to work; the 1.34+ requirement and what your mixed-version fleet does meanwhile; and the governance problem that policy now lives in claim definitions written by tenants rather than in a central operator.

Then **propose the concrete signal your operator emits**: the exact annotation key (e.g. `cost.example.com/gpu-cost-score`), value semantics (normalized 0–100? higher = cheaper? what is the normalization window?), what it is attached to (Node versus something ResourceSlice-adjacent), how it is kept fresh and what happens when it is stale, and how it degrades when the annotation is missing entirely. Say which of the three you would ship first and why.

Include a short section on **what the signal must not be**: the `$/GPU-hour` versus `$/unit-of-work` caveat from the Worked example, in your own words, with the arithmetic.

**Acceptance:** `design.md` committed, covering all three approaches with tradeoffs, a concrete costScore proposal, an explicit "ship this first" decision with a reason, and the throughput caveat. **Design artifact — no plugin code.** This is acceptance item 8 in the [deliverable's checklist](../practice/gpu-cost-operator/README.md#acceptance-criteria-matches-the-checkpoint).

## Common pitfalls

1. **Encoding a soft cost preference as a Filter.** Symptom: Pods go Pending the instant the cheap tier fills, while expensive capacity sits idle. Mechanism: Filter removes a node from consideration entirely; a Pod that fails Filter everywhere goes to `unschedulableQ` and waits for a cluster event. Fix: Score, with `NormalizeScore` and a deliberate weight.
2. **Designing a Score signal without accounting for normalization and weight.** Symptom: the plugin demonstrably runs, the annotation is read, and placement does not change. Mechanism: raw values outside 0–100 are dwarfed by other plugins' weighted contributions — the default set can swing ~1,200 points in total. Fix: normalize to `[MinScore, MaxScore]` = `[0, 100]` and justify the weight against the default table.
3. **Believing `percentageOfNodesToScore` limits how many nodes are *examined*.** Symptom: a GPU fleet where scheduler CPU is pegged and nobody can explain why, since "we only score 10%." Mechanism: the cap is on *feasible nodes found*; a Pod requesting a scarce resource never accumulates that many and therefore scans the whole fleet every cycle. Fix: understand the number as a quality/latency knob for *common* Pods; use a separate scheduler profile or fleet partitioning for scarce-resource workloads.
4. **Believing Kueue schedules Pods.** Symptom: a design doc that says "Kueue places the job on the A100 nodes." Mechanism: Kueue admits `Workload` objects and unsuspends Jobs; it never writes `.spec.nodeName`. Its influence on *where* is indirect, through `ResourceFlavor` node labels. Fix: say "Kueue admits, the scheduler places" every time, until it is automatic.
5. **Treating `ResourceFlavor` as a node selector alias.** Symptom: a quota model that cannot express borrowing or fair sharing. Mechanism: a flavor carries `nominalQuota`, `borrowingLimit`, `lendingLimit`, and cohort membership in addition to `nodeLabels` — it is a quota dimension, not just a selector. Fix: model price tiers as flavors precisely so quota can be assigned per tier.
6. **Assuming DRA has replaced device plugins everywhere.** Symptom: a design that breaks on 30 of your 40 clusters. Mechanism: DRA is GA from 1.34 and needs a vendor DRA driver; device plugins remain the only path on older clusters and on hardware without a driver. Fix: design for both, and note that `NodeResourcesFit` now delegates specific resource names to DRA when a driver claims them, so the two coexist by design.
7. **Forgetting the Topology Manager exists.** Symptom: Pods that bind successfully and then fail with `TopologyAffinityError`, terminal, sometimes repeatedly on the same node. Mechanism: the scheduler is not topology-aware; kubelet evaluates NUMA alignment after binding, and under `restricted` or `single-numa-node` it rejects rather than degrades. Fix: know the node's policy before you promise placement guarantees, and treat `best-effort` as the default for mixed fleets.
8. **Treating time-slicing as capacity.** Symptom: a "16 GPU" node that thrashes and OOMs under four real tenants. Mechanism: `sharing.timeSlicing.replicas` multiplies the advertised integer without adding hardware; there is no memory or fault isolation between replicas of the same card. Fix: use MIG (or DRA-modelled MIG) where isolation matters; reserve time-slicing for bursty inference and dev.

## Self-check

- **What does DRA request-by-attribute enable for cost and MIG that device-plugin integer counts cannot?**
  **Answer:** The device-plugin model advertises GPUs as an opaque integer extended resource that the scheduler reads out of `node.status.allocatable` as a single `int64` in `ScalarResources` — all devices are fungible and the scheduler can only *count* them. It cannot express "≥40 GB of device memory", "compute capability ≥ 9.0", "a MIG `3g.40gb` slice", "four cards on one NVLink domain", or "the cheaper card." The only workaround for MIG is the plugin's `mixed` strategy, which advertises each profile as a *separate, mutually unsubstitutable* resource name (`nvidia.com/mig-1g.10gb`, `nvidia.com/mig-3g.40gb`, …), so a fleet with three GPU generations and four profiles ends up with a dozen independent integer pools that each fragment on their own and cannot substitute for one another. DRA (GA in 1.34, KEP-4381) replaces this: a vendor driver publishes each device's **attributes and capacities** in a `ResourceSlice`, and a `ResourceClaim` selects with CEL over them. MIG slices become ordinary devices carrying profile attributes in the same inventory as whole cards, so "the cheapest device meeting ≥40 GB" is expressible for the first time, and a small inference Pod can land on a `1g.10gb` slice instead of monopolizing an H100. You cannot optimize spend or MIG utilization across a fleet you can only count.

- **Which extension point biases placement toward cheaper GPUs, and why Score rather than Filter?**
  **Answer:** **Score**, with `NormalizeScore` and a deliberate weight. Filter is a hard boolean gate: a node that fails it is removed from consideration entirely, and a Pod that fails Filter on every node goes Pending, may trigger `PostFilter` preemption, and lands in `unschedulableQ` awaiting a cluster event. Encoding cost as a Filter therefore turns "prefer cheap" into "only cheap," and the moment the cheap tier fills you get Pending Pods while expensive capacity idles — a self-inflicted outage that presents as a capacity problem. Score never removes a node; it ranks the survivors, and the highest weighted total wins with ties broken uniformly. Two design consequences: the plugin must normalize into `[0, 100]` (`MinScore`/`MaxScore`) or its signal will be dwarfed, and the weight must be argued against the default set (`TaintToleration: 3`, `NodeAffinity: 2`, `PodTopologySpread: 2`, `InterPodAffinity: 2`, and three plugins at weight 1 — about 1,200 points of possible total), so a cost signal at weight 1 can move at most ~8% of the total and will look broken in production despite passing unit tests.

- **What does Kueue's gang / all-or-nothing admission solve that the default scheduler cannot — and name the second-order effect most people miss.**
  **Answer:** The default scheduler processes **one Pod at a time, serially, with no notion of a job as a unit.** A 32-Pod training job is 32 independent decisions, so it will happily place 30 Pods and leave 2 Pending — 30 expensive GPUs parked doing nothing, at roughly $150/hour for H100s — and several interleaved jobs can deadlock, each holding a fraction and none able to run. The framework's `Permit` extension point *enables* gang semantics (hold a Pod without releasing its `Reserve` until siblings arrive), but the default profile enables no plugin there, which is exactly why partial placement is the out-of-the-box behaviour. Kueue solves it upstream: a `Workload` wraps the Job's pod sets, and it is admitted only when quota for **all** of them is reservable at once; until then the Job stays `suspend: true` and **no Pods exist**. The second-order effect most people miss is scheduler *throughput*: without gang admission, those 512 unschedulable Pods each get filtered against the full node list on every backoff retry (capped at `podMaxBackoffSeconds: 10`), and because a Pod requesting a scarce resource never accumulates enough feasible nodes to hit the `percentageOfNodesToScore` early-exit, each retry scans every node. On a 5,000-node fleet at ~0.2 ms of Filter per node with parallelism 16, that is ~62.5 ms per Pod, ~32 s per full sweep of a 512-Pod job, repeated every 10 seconds, in a single-threaded scheduling cycle that nothing else can use. Suspending the Job costs zero.

- **Walk `nvidia.com/gpu: 4` from silicon to a running container, naming every component and protocol.**
  **Answer:** (1) A **device-plugin DaemonSet** pod starts and serves gRPC on `/var/lib/kubelet/device-plugins/nvidia.sock` — it must serve *before* registering, or kubelet's dial-back fails. (2) It calls `Register(RegisterRequest{version: "v1beta1", endpoint: "nvidia.sock", resource_name: "nvidia.com/gpu", options})` on kubelet's well-known `kubelet.sock`. (3) Kubelet dials back and opens `ListAndWatch(Empty)`, a long-lived stream of `ListAndWatchResponse{devices}`, where each `Device` carries an `ID`, a `health` string, and `TopologyInfo{nodes: [NUMANode{ID}]}`. The plugin re-sends the whole list on any change. (4) Kubelet counts *healthy* devices and PATCHes `node.status.capacity` and `node.status.allocatable` with `"nvidia.com/gpu": "7"` — an unhealthy card simply decrements the number, with no way for the scheduler to learn which card. (5) The **scheduler** reads that integer into `ScalarResources` during `PreFilter`, and `NodeResourcesFit`'s `Filter` rejects any node where `allocatable − requested < 4`. Extended resources are integer-only, requests must equal limits, cannot be overcommitted, and cannot be shared across containers. (6) Score/Reserve/Bind pick and record the node. (7) On that node, **kubelet's Device Manager** — not the scheduler — chooses *which* device IDs, optionally consulting the plugin's advisory `GetPreferredAllocation` (NVIDIA uses it to prefer NVLink-connected peers), then calls `Allocate(ContainerAllocateRequest{devices_ids})`. (8) The plugin returns `ContainerAllocateResponse{envs, mounts, devices, annotations, cdi_devices}` — `NVIDIA_VISIBLE_DEVICES`, `/dev/nvidia*` device nodes, driver library mounts, CDI names. (9) Optionally `PreStartContainer` runs device prep. (10) **Topology Manager** intersects hints from CPU Manager, Memory Manager, and Device Manager and, under `restricted` or `single-numa-node`, may reject the Pod here with a terminal `TopologyAffinityError` — *after* it was bound. (11) Kubelet merges the response into the CRI `ContainerConfig` and the container starts. The line to say out loud: **the scheduler picks the node, the kubelet picks the devices**, and without DRA nothing that made the placement decision knows which physical cards were used.

- **Your operator must bias placement toward cheaper GPUs, enforce a hard per-team monthly GPU-hour budget, and let a small inference Pod use one seventh of an H100. Map each to the layer that owns it, and say why the other layers cannot do it.**
  **Answer:** *Bias toward cheaper GPUs* → **scheduler Score**, because it is a soft preference among already-feasible nodes; Kueue cannot pick a node at all, and DRA selects a device only after the node is chosen. *Hard per-team monthly budget* → **Kueue quota** (`ClusterQueue` with per-`ResourceFlavor` `nominalQuota`, `borrowingLimit: 0` on the expensive tier, cohort borrowing on the cheap one), because a hard cap must be able to *reject or queue* a whole job before it starts; Score structurally cannot enforce anything (it only ranks, and never removes a node), and DRA does not gate job-level admission. *One seventh of an H100* → **DRA plus MIG**, because only an attribute model can express "a `1g.10gb` slice" as a request target — the device-plugin path can only do it by inventing a separate, unsubstitutable resource name per profile, and neither Score nor Kueue reasons about devices at all. The general shape to state: **Kueue enforces, Score suggests, DRA selects** — and a real design needs all three, layered, not one of them chosen.

## Connections & what's next

This lesson sits on top of lesson 01's apiserver/watch model (DRA drivers publish `ResourceSlice` objects the same way any client publishes any object — there is no back door), lesson 02's typed-versus-CEL API machinery (`ResourceClaim` selectors are CEL over structured attributes, the same tool lesson 05's CRD validation used), and lesson 03's reconcile discipline (Kueue's own controllers admit and suspend `Workload` objects through the same level-triggered loop this module teaches). It also completes a thread from lesson 08: the DRA design pivot from KEP-3063 to KEP-4381 — from "call a vendor controller mid-decision" to "read published data in-process" — is the *same* architectural lesson as preferring `ValidatingAdmissionPolicy` over a webhook. A synchronous external dependency inside a serial loop is a hazard wherever it appears.

**Next: lesson 10, the capstone build.** Scheduling stays a design artifact here on purpose. Lesson 10 is where every earlier lesson's piece — CRDs, reconcile, finalizer, webhook, RBAC — gets assembled into `gpu-cost-operator` v0.1 and has to actually run, together, green under envtest.

## References & further reading

**Primary sources**

- [kubernetes.io — Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/) — the authoritative extension-point list and cycle split; read for what each point may return and which are informational.
- [kubernetes.io — kube-scheduler Configuration (v1)](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/) — the `KubeSchedulerConfiguration` reference and every default quoted here: `parallelism: 16`, `podInitialBackoffSeconds: 1`, `podMaxBackoffSeconds: 10`, `percentageOfNodesToScore`.
- [kubernetes.io — Scheduling Policies / scheduler configuration](https://kubernetes.io/docs/reference/scheduling/config/) — profiles, `schedulerName`, `multiPoint`, and the default plugin set per extension point.
- [kubernetes.io — Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/) — `percentageOfNodesToScore`, the auto formula, and the round-robin node iteration described in §4.
- [kubernetes/kubernetes — `pkg/scheduler/schedule_one.go`](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/schedule_one.go) — `numFeasibleNodesToFind` with `minFeasibleNodesToFind = 100` and `minFeasibleNodesPercentageToFind = 5`, reproduced verbatim in §4, plus the `nextStartNodeIndex` wraparound.
- [kubernetes/kubernetes — `pkg/scheduler/apis/config/v1/default_plugins.go`](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/apis/config/v1/default_plugins.go) — the default plugin weights table in §3, straight from `getDefaultPlugins()`.
- [kubernetes.io — Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/) — the registration handshake, socket paths, and the requirement to serve before registering.
- [kubernetes/kubernetes — device-plugin `api.proto` (v1beta1)](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1/api.proto) — the exact service and message definitions reproduced in §5.
- [kubernetes.io — Managing Resources for Containers (Extended resources)](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) — the integer-only / requests-equal-limits / no-overcommit rules and the Node-status PATCH form.
- [kubernetes.io — Control Topology Management Policies on a Node](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/) — hint providers, the four policies, `container`/`pod` scopes, `prefer-closest-numa-nodes`, `max-allowable-numa-nodes` (default 8), and the explicit statement that the scheduler is not topology-aware.
- [kubernetes.io — Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) — the `ResourceSlice`/`DeviceClass`/`ResourceClaim` model, the allocation flow through Filter/Reserve/PreBind, feature gates, and the no-preemption limitation.
- [KEP-4381, "DRA: structured parameters"](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4381-dra-structured-parameters) — the GA design rationale and the KEP-3063 withdrawal history behind §8.
- [Kueue documentation — Concepts](https://kueue.sigs.k8s.io/docs/concepts/) — `ResourceFlavor`, `ClusterQueue`, `LocalQueue`, `Workload`, `Cohort`, `AdmissionCheck`, and Topology-Aware Scheduling.
- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) — `migStrategy` (`none`/`single`/`mixed`) and the resulting resource names, `deviceListStrategy`, `deviceIDStrategy`, the time-slicing `sharing.timeSlicing` config with `replicas`/`renameByDefault`/`failRequestsGreaterThanOne`, and prerequisites.

**Real-world engineering writeups**

- CoreWeave, ["Kueue: A Kubernetes-Native System for AI Training Workloads"](https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads) — Kueue in production for AI training and batch inference at a GPU cloud.
- Google Cloud, ["Kubernetes device management with DRA"](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation) — a second vendor's production DRA treatment.
- NVIDIA, ["NVIDIA donates its DRA driver for GPUs to the Kubernetes community"](https://blogs.nvidia.com/blog/nvidia-at-kubecon-2026/) — the GPU-vendor DRA driver, concretely.
- AKS Engineering, ["Running more with less: MIG with DRA on AKS"](https://blog.aks.azure.com/2026/03/03/multi-instance-gpu-with-dra-on-aks) — production MIG + DRA pairing.
- Uber Engineering, ["Uber's Journey to Ray on Kubernetes: Resource Management"](https://www.uber.com/en-DE/blog/ubers-journey-to-ray-on-kubernetes-resource-management/) — a hyperscaler's custom alternative to Kueue.

**Deeper dives**

- [`kubernetes-sigs/scheduler-plugins`](https://github.com/kubernetes-sigs/scheduler-plugins) — out-of-tree plugins including co-scheduling/gang and `NetworkAware`; the reference implementation to read before designing any Score plugin. Not required to build.
- [kubernetes.io — PodGroup Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/) and [Topology-Aware Workload Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/) — the alpha in-tree gang work described in §10, including `minCount` and the `PlacementGenerate`/`PlacementScore` extension points behind the `GenericWorkload` and `TopologyAwareWorkloadScheduling` gates.
