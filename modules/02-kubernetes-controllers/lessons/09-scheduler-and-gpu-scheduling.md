---
lesson: "02.9"
title: "The scheduler framework and the GPU-scheduling frontier"
module: "02"
concept: "The scheduler framework and the GPU-scheduling frontier"
status: not-started
est_time: "8h"
artifacts: []
---

# 02.9 · The scheduler framework and the GPU-scheduling frontier

> **Concept.** The default scheduler is a framework of pluggable extension points; GPU and distributed-training workloads push past its integer-count, one-Pod-at-a-time model into DRA (request devices by attribute) and Kueue (quota + gang admission).
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Why this matters

You run 40+ clusters and know the scheduler as an operator: taints, tolerations, affinity, `nvidia.com/gpu: 1`. At a GPU-heavy shop (CoreWeave, NVIDIA) that literacy is table stakes. What differentiates a *senior* platform engineer there is being able to sit at a whiteboard and reason about *where a placement decision is made, which layer owns it, and what breaks when a 512-GPU training job needs all its Pods or none of them*. That is a design conversation, not a `kubectl` conversation.

This lesson is **literacy + design**, explicitly **not** "write a production scheduler plugin." Writing an in-tree Score plugin is a rare, high-blast-radius task owned by a handful of people; being able to *design against* the framework — and to place your `gpu-cost-operator` correctly in the stack relative to DRA and Kueue — is the skill that scales across 40 clusters and the skill an interview panel probes. Your FinOps background is the wedge: almost nobody in this space is fluent in *both* the scheduling internals and the cost model. That intersection is where a costScore signal lives.

## From operating to extending

As an operator you *configure* scheduling with knobs the scheduler already exposes. As an extender you reason about the decision pipeline itself and about the two systems that now sit *around* the scheduler for AI/ML.

Three distinct layers, and confusing them is the classic interview failure:

| Layer | Question it answers | Owns |
|---|---|---|
| **Scheduler framework** | "Given a Pod and N feasible nodes, which node?" | Per-Pod node selection |
| **DRA** (`resource.k8s.io`) | "Which *device* on that node, by attributes?" | Device modeling + allocation |
| **Kueue** | "Should this whole *job* be admitted at all, right now, within quota?" | Gate/quota *before* Pods exist |

Kueue decides admission (a Job waits, suspended, until quota is free); DRA decides which physical/virtual device backs a claim; the scheduler framework decides the node. A distributed training job touches all three. Keeping them separate on the whiteboard is the whole game.

## Core notes

### 1. The scheduler framework: extension points, in order

kube-scheduler is a framework: a fixed scheduling cycle with named extension points, and plugins register at the points they care about. One Pod moves through the pipeline at a time. The cycle splits into a **scheduling cycle** (synchronous, picks a node) and a **binding cycle** (can be asynchronous).

Scheduling cycle:

- **QueueSort** — orders the pending queue (one plugin, cluster-wide).
- **PreFilter** — validate/precompute Pod-level facts once; can short-circuit as infeasible.
- **Filter** — the *predicate* stage. For each candidate node, feasible or not? Runs per node, often in parallel. Taints, node affinity, resource fit (`nvidia.com/gpu` count), DRA's `Filter` all live here. Filter answers a **boolean**.
- **PostFilter** — runs only when *no* node passed Filter. Default preemption plugin lives here (evict lower-priority Pods to make room).
- **PreScore** — precompute shared state for scoring.
- **Score** — the *priority* stage. Each surviving node gets an integer (default 0–100 after normalization); the highest total wins. This is where you *bias* rather than *exclude*. Bin-packing (`NodeResourcesFit` with `MostAllocated`), spreading, and any "prefer cheaper GPU" logic belong here.
- **NormalizeScore** — rescale a plugin's raw scores to the common range before weighting.
- **Reserve** — mark resources claimed on the chosen node *before* binding, so concurrent Pods don't double-book. Has an **Unreserve** on failure. Stateful plugins (including DRA) use this.
- **Permit** — can **approve, deny, or _wait_**. This is the framework's built-in hook for **gang semantics**: hold a Pod at Permit until its siblings are also ready, then release them together. (In-tree co-scheduling plugin uses this.)

Binding cycle:

- **PreBind** — do work that must succeed before binding (e.g., provision/attach a volume; DRA finalizes allocation here).
- **Bind** — write the `Pod.spec.nodeName` binding. Default plugin does the API call; a custom Bind plugin can delegate elsewhere.
- **PostBind** — cleanup/notify after a successful bind.

**Filter vs Score is the load-bearing distinction.** Filter is a hard gate: fail it and the node is *gone* from consideration — a Pod that fails every node's Filter goes Pending (or triggers preemption). Score never removes a node; it only ranks the survivors. So "prefer the cheaper GPU, but still run on an expensive one if that's all that's free" is a **Score** decision. Encoding cost as a Filter would make Pods go Pending the moment cheap capacity is exhausted — a self-inflicted outage. This is the single most common design-review catch, and self-check (b) below.

### 2. Why the default scheduler is insufficient for GPU / distributed jobs

Two structural limits:

**(a) It counts, it doesn't model.** The device-plugin API advertises GPUs as an opaque **integer extended resource**, `nvidia.com/gpu: 8`. All eight are treated as fungible. The scheduler cannot express "a GPU with ≥40 GB", "a MIG `3g.40gb` slice", "compute capability ≥ 9.0 (Hopper)", or "the cheap A100 not the expensive H100" — because to the scheduler they are all just the integer `1`. Heterogeneous fleets and MIG partitioning are exactly where cost/efficiency decisions live, and the count model is blind to them. **DRA** fixes this (§3).

**(b) It schedules one Pod at a time, greedily.** A 32-Pod distributed job is 32 independent scheduling decisions. The scheduler will happily place 30 Pods and leave 2 Pending — you've now **parked 30 expensive GPUs doing nothing** waiting for the last 2, and if several such jobs interleave they can **deadlock**, each holding a fraction of what it needs and none able to complete. The framework's Permit point *enables* a fix, but the default profile doesn't gang-schedule out of the box. **Kueue** (or a co-scheduling plugin) fixes this (§4). This partial-placement/deadlock story is the crux of self-check (c).

### 3. DRA — requesting devices by attribute (GA in 1.34)

**Dynamic Resource Allocation** graduated to **GA in Kubernetes 1.34** (Sept 2025), on **KEP-4381, "DRA structured parameters."** The APIs are in `resource.k8s.io/v1`. (History worth knowing for interviews: the *original* DRA design used opaque vendor "structured parameters CRDs"/controllers — **KEP-3063** — and was **withdrawn/superseded in the 1.32 cycle**; the GA design is the structured-parameters model of KEP-4381, where allocation logic lives in the scheduler using data it can actually read, not in an out-of-tree vendor controller.)

The core API objects and how they reach the scheduler:

- **ResourceSlice** — published by the vendor **DRA driver** (a DaemonSet on each node). It advertises the node's devices *and their attributes*: memory, MIG profile, product name, compute capability, driver version, etc. This is the fleet's ground truth the scheduler reads.
- **DeviceClass** — a cluster-scoped "kind of device" plus default selection/config. Think `StorageClass`, but for devices.
- **ResourceClaim** / **ResourceClaimTemplate** — the *request*. A claim carries `deviceRequests` with **CEL selectors over attributes** (e.g. `device.capacity['memory'] >= '40Gi'`, `device.attributes['...productName'] == 'A100'`). A ResourceClaimTemplate stamps out a fresh claim per Pod (like a PVC template). Pods reference claims via the new `pod.spec.resourceClaims` field.

**How it plugs into the scheduling cycle:** the DRA scheduler plugin reads ResourceSlices, and at **Filter** eliminates nodes that can't satisfy the claim's selectors, at **Reserve** tentatively allocates the specific device, and at **PreBind** writes the finalized allocation back into the ResourceClaim status — all using structured data the scheduler owns, so no round-trip to a vendor controller during scheduling.

**Why this matters for cost/MIG (self-check a):** integer counts force "a GPU"; DRA lets you request "the *cheapest device that meets my constraint*" — a MIG `1g.10gb` slice for a small inference Pod instead of monopolizing a whole H100, or "any GPU with ≥24 GB" so a job lands on a cheaper card when it fits. Attribute-based requests are the substrate a cost-aware policy needs; you can't optimize spend across a fleet you can only count.

### 4. Kueue — quota, gang, and topology-aware admission

**Kueue** (`kueue.sigs.k8s.io`) is a Kubernetes-native **job queueing** layer that sits *in front of* the scheduler. It doesn't place Pods; it decides **whether a whole workload is admitted** and, until then, keeps the Job **suspended** (`.spec.suspend=true`, no Pods created). **CoreWeave runs Kueue** on CoreWeave Kubernetes Service for customer training and batch-inference workloads — this is production, not a lab curiosity.

Key concepts:

- **ClusterQueue** — cluster-scoped quota pool: how much CPU/memory/`nvidia.com/gpu` a tenant may consume, with borrowing/lending across cohorts and fair-sharing.
- **LocalQueue** — namespaced; a tenant submits jobs to a LocalQueue, which points at a ClusterQueue. Namespace-scoped handle onto cluster-scoped quota.
- **Workload** — Kueue's internal object wrapping a Job (or RayJob/JobSet/etc.) and its **pod sets**.
- **All-or-nothing / gang admission** — a Workload is unsuspended only when quota for **all** its pod sets can be reserved **at once**. This is the direct answer to §2(b): no partial placement, no half-held deadlock. The job waits, whole, in the queue until it fits, whole.
- **Topology-Aware Scheduling (TAS)** — Kueue tracks free capacity per topology domain (rack/block/node) and admits with a placement intent: `required` (all pods in one domain — tight all-reduce communication), `preferred` (same domain if possible, else spread), or `unconstrained`. Critical when inter-GPU bandwidth dominates training throughput.

Mental model: **Kueue gates the door (admission + quota + gang), the scheduler seats the guests (per-Pod node choice), DRA picks the exact chair (device by attribute).**

## Worked example

Walk the lifecycle of one 8-Pod, 8-GPU distributed training job on a heterogeneous cluster (some A100 40 GB nodes, some H100 80 GB nodes), narrating the three layers:

1. **Submit.** User creates a `Job` (8 parallel Pods, each claiming 1 GPU via a ResourceClaimTemplate that selects `memory >= 40Gi`) referencing a LocalQueue. Kueue's Job integration sets `suspend=true` — **zero Pods exist yet.**
2. **Kueue admission.** Kueue wraps it as a Workload with one pod set of size 8. Its ClusterQueue has `nvidia.com/gpu` quota of 16, currently 12 in use → only 4 free. **All-or-nothing:** 8 > 4, so the Workload **waits in the queue.** No GPUs parked, no deadlock — the job simply hasn't started. When 4 more free up, quota for all 8 is reservable at once; Kueue admits and sets `suspend=false`.
3. **Pods created → scheduler.** Now 8 Pods hit the scheduling queue. For each: **Filter** drops nodes with no free GPU *and* (via the DRA plugin, reading ResourceSlices) nodes whose devices don't meet `memory >= 40Gi`. Both A100-40 and H100-80 nodes survive.
4. **Score.** Your `gpu-cost-operator` has annotated nodes with a costScore. A Score plugin (or a scheduler configured to weight it) ranks A100 nodes above H100 nodes for a job that only asked for 40 GB → **bias toward the cheaper card, without excluding H100** if that's all that's free.
5. **Reserve → PreBind → Bind.** DRA plugin tentatively allocates the specific device at **Reserve**, finalizes the ResourceClaim at **PreBind**, and the Pod is bound. If TAS was configured, Kueue's admission already constrained these 8 to a topology domain for fast all-reduce.

Note where each decision lived: **admission/gang = Kueue**, **node = scheduler Score/Filter**, **which device = DRA**. If you had tried to make cost a *Filter*, step 3 would have gone Pending the instant A100s filled — which is why cost is a *Score*.

## Practice

**Deliverable: a 1-page design doc, committed** at `../practice/gpu-cost-operator/docs/design.md`.

Title: **"How would `GPUCostPolicy` influence GPU placement?"** Your `gpu-cost-operator` reconciles a `GPUCostPolicy` CR and emits a per-node (or per-device) **costScore** signal. Design *how that signal reaches a placement decision*, comparing **three** approaches with explicit tradeoffs:

1. **Score-plugin approach** — operator writes costScore as a node annotation/label; a scheduler Score plugin (or `NodeResourcesFit`/`scheduler-plugins` config) reads it to bias placement. Tradeoffs: needs scheduler config or a custom plugin (build/maintain/blast-radius); operates per-Pod at node granularity; can't see whole-job or quota.
2. **Kueue-quota approach** — encode cost policy as ClusterQueue quotas / cohort borrowing so expensive GPUs are rationed at admission. Tradeoffs: no plugin to maintain; enforces *budget* and gang cleanly; but it gates jobs, it doesn't pick the cheaper *node* among feasible ones — coarser, admission-time.
3. **DRA-attributes approach** — express cost preference through DeviceClass selection / ResourceClaim selectors so claims target cheaper device classes. Tradeoffs: most precise (device-level, MIG-aware); requires DRA (1.34+) and driver-published attributes; policy lives in claim/class definitions, further from a central operator.

Then **propose the costScore annotation your operator emits**: exact key (e.g. `gpu-cost.io/cost-score`), value semantics (lower = cheaper? normalized 0–100?), what it's attached to (Node vs ResourceSlice-adjacent), and how it's kept fresh. State which of the three you'd ship first and why.

**Acceptance:** `design.md` committed, covering all three approaches with tradeoffs and a concrete costScore proposal. **Design artifact — no plugin code.**

## Self-check

**(a) What does DRA request-by-attribute enable for cost/MIG that device-plugin integer counts cannot?**

**Answer:** The device-plugin model advertises GPUs as an opaque integer extended resource (`nvidia.com/gpu: 8`) — all devices are fungible and the scheduler can only *count* them. DRA (GA in 1.34, KEP-4381) lets a ResourceClaim request devices by **attribute** via CEL selectors over ResourceSlice data: memory, MIG profile, product name, compute capability. That enables landing a small workload on a MIG `1g.10gb` slice instead of a whole H100, and requesting "cheapest device that meets ≥40 GB" so a job packs onto a cheaper A100 when it fits. You cannot optimize spend or MIG utilization across a fleet you can only count — attribute requests are the substrate cost-awareness needs.

**(b) Which framework extension point biases placement toward cheaper GPUs, and why Score, not Filter?**

**Answer:** **Score** (with NormalizeScore + weighting). Filter is a hard boolean gate — a node that fails Filter is removed from consideration entirely, so a Pod failing every node's Filter goes Pending or triggers preemption. Encoding cost as a Filter would make Pods go Pending the moment cheap capacity is exhausted — a self-inflicted outage. Score never removes a node; it ranks the survivors. "Prefer cheaper, but still run on an expensive GPU if that's all that's free" is exactly a *soft preference among feasible nodes* → Score.

**(c) What does Kueue's gang / all-or-nothing admission solve that the default scheduler cannot?**

**Answer:** The default scheduler places one Pod at a time, greedily, with no notion of a job as a unit. A 32-Pod training job can get 30 Pods placed and 2 left Pending — **30 expensive GPUs parked idle** waiting for the last 2 — and interleaved jobs can **deadlock**, each holding a fraction and none able to run. Kueue admits a Workload **only when quota for all its pod sets can be reserved at once**, keeping the Job suspended (no Pods created) until it fits whole. That eliminates partial placement and the associated GPU parking/deadlock — the framework's Permit point *enables* gang semantics, but Kueue delivers it as job-level admission plus quota.

## Resources

- Scheduling framework — extension points and cycle: <https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/>
- Dynamic Resource Allocation (concept docs): <https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/>
- **Version-sensitive:** DRA graduated to **GA in v1.34** — announcement: <https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/> — verify API stability (`resource.k8s.io/v1`) and driver support against your cluster version; the 1.30–1.33 alpha/beta APIs differ.
- KEP-4381, DRA structured parameters (the GA design; note KEP-3063 was withdrawn in the 1.32 cycle): <https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4381-dra-structured-parameters>
- Kueue concepts (ClusterQueue/LocalQueue/Workload): <https://kueue.sigs.k8s.io/docs/concepts/>
- Kueue all-or-nothing (gang) admission: <https://kueue.sigs.k8s.io/docs/concepts/all_or_nothing/>
- Kueue topology-aware scheduling: <https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/>
- CoreWeave on running Kueue for AI training/batch inference: <https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads>
- scheduler-plugins (out-of-tree plugins incl. co-scheduling/gang; reference for Score-plugin design, not required to build): <https://github.com/kubernetes-sigs/scheduler-plugins>
