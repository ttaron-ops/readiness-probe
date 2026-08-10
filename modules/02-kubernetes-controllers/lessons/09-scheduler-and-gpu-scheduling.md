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
sources: 13
---

# 02.9 · The scheduler framework and the GPU-scheduling frontier

> **Concept.** The default scheduler is a framework of pluggable extension points; GPU and distributed-training workloads push past its integer-count, one-Pod-at-a-time model into DRA (request devices by attribute) and Kueue (quota + gang admission).
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Lesson 08 closed out the "write path" of the API server: admission webhooks decide whether an object may exist at all. This lesson moves to the other end of an object's life — once a Pod *does* exist, something has to decide where it runs. That "where" question turns out to have three separate owners once GPUs and distributed jobs are in the picture, not one. This lesson gives you the vocabulary and mental model to place your `gpu-cost-operator`'s signal correctly among them — DRA GA (Kubernetes 1.34) and Kueue are exactly the depth probe your target companies use to separate CKA-level scheduling literacy from staff-level design fluency. It closes with a design doc, not code, and directly sets up Lesson 10, where every earlier lesson's pieces get assembled and tested together.

## Why this matters

You run 40+ clusters and know the scheduler as an operator: taints, tolerations, affinity, `nvidia.com/gpu: 1`. At a GPU-heavy shop (CoreWeave, NVIDIA) that literacy is table stakes. What differentiates a *senior* platform engineer there is being able to sit at a whiteboard and reason about *where a placement decision is made, which layer owns it, and what breaks when a 512-GPU training job needs all its Pods or none of them*. That is a design conversation, not a `kubectl` conversation.

This lesson is **literacy + design**, explicitly **not** "write a production scheduler plugin." Writing an in-tree Score plugin is a rare, high-blast-radius task owned by a handful of people; being able to *design against* the framework — and to place your `gpu-cost-operator` correctly in the stack relative to DRA and Kueue — is the skill that scales across 40 clusters and the skill an interview panel probes. Your FinOps background is the wedge: almost nobody in this space is fluent in *both* the scheduling internals and the cost model. That intersection is where a costScore signal lives, and it's the checkpoint's explicit "GPU scheduling (differentiator)" probe.

## What's new here (calibration)

As an operator you *configure* scheduling with knobs the scheduler already exposes. As an extender you reason about the decision pipeline itself and about the two systems that now sit *around* the scheduler for AI/ML. Already know / skip vs genuinely new:

- **Already know, skip:** `kubectl` scheduling knobs — taints/tolerations, node affinity/anti-affinity, `nvidia.com/gpu: N` requests, `kubectl describe` on a Pending Pod's events.
- **Already know, skip:** device-plugin basics — that a DaemonSet advertises `nvidia.com/gpu` as an extended resource; you've run NVIDIA's device plugin for years.
- **New here:** the scheduler as a *framework* — named extension points, the Filter-vs-Score distinction, and where Permit/Reserve/PreBind live in the cycle.
- **New here:** DRA's attribute-based device model (GA in 1.34) and how it structurally replaces "GPU as opaque integer" with "GPU as queryable attributes" — and the KEP history (3063 withdrawn → 4381 GA) that explains *why* the current design looks the way it does.
- **New here:** Kueue as a distinct admission/quota layer *above* the scheduler, and why gang/all-or-nothing semantics can't be bolted onto the default scheduler's one-Pod-at-a-time loop.

Three distinct layers, and confusing them is the classic interview failure:

| Layer | Question it answers | Owns |
|---|---|---|
| **Scheduler framework** | "Given a Pod and N feasible nodes, which node?" | Per-Pod node selection |
| **DRA** (`resource.k8s.io`) | "Which *device* on that node, by attributes?" | Device modeling + allocation |
| **Kueue** | "Should this whole *job* be admitted at all, right now, within quota?" | Gate/quota *before* Pods exist |

Kueue decides admission (a Job waits, suspended, until quota is free); DRA decides which physical/virtual device backs a claim; the scheduler framework decides the node. A distributed training job touches all three. Keeping them separate on the whiteboard is the whole game.

## Core concepts

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

**(b) It schedules one Pod at a time, greedily.** A 32-Pod distributed job is 32 independent scheduling decisions. The scheduler will happily place 30 Pods and leave 2 Pending — you've now **parked 30 expensive GPUs doing nothing** waiting for the last 2, and if several such jobs interleave they can **deadlock**, each holding a fraction of what it needs and none able to complete. The framework's Permit point *enables* a fix, but the default profile doesn't gang-schedule out of the box. **Kueue** (or a co-scheduling plugin) fixes this (§4). This partial-placement/deadlock story is the crux of self-check (c) and the checkpoint's Kueue probe.

### 3. DRA — requesting devices by attribute (GA in 1.34)

**Dynamic Resource Allocation** graduated to **GA in Kubernetes 1.34** (Sept 2025), on **KEP-4381, "DRA structured parameters."** The APIs are in `resource.k8s.io/v1`. (History worth knowing for interviews: the *original* DRA design used opaque vendor "structured parameters CRDs"/controllers — **KEP-3063** — and was **withdrawn/superseded in the 1.32 cycle**; the GA design is the structured-parameters model of KEP-4381, where allocation logic lives in the scheduler using data it can actually read, not in an out-of-tree vendor controller. This shift — from "call out to a vendor controller mid-schedule" to "read vendor-published structured data in-process" — is the same kind of design lesson as watch-vs-poll from Lesson 01: put the data where the decision-maker can read it directly.)

The core API objects and how they reach the scheduler:

- **ResourceSlice** — published by the vendor **DRA driver** (a DaemonSet on each node). It advertises the node's devices *and their attributes*: memory, MIG profile, product name, compute capability, driver version, etc. This is the fleet's ground truth the scheduler reads. NVIDIA's own DRA driver for GPUs — donated to the CNCF/Kubernetes community at KubeCon — is the concrete, production example: it's the DaemonSet that walks the node's physical GPUs and MIG configuration and writes them out as ResourceSlices.
- **DeviceClass** — a cluster-scoped "kind of device" plus default selection/config. Think `StorageClass`, but for devices.
- **ResourceClaim** / **ResourceClaimTemplate** — the *request*. A claim carries `deviceRequests` with **CEL selectors over attributes** (e.g. `device.capacity['memory'] >= '40Gi'`, `device.attributes['...productName'] == 'A100'`). A ResourceClaimTemplate stamps out a fresh claim per Pod (like a PVC template). Pods reference claims via the new `pod.spec.resourceClaims` field.

**How it plugs into the scheduling cycle:** the DRA scheduler plugin reads ResourceSlices, and at **Filter** eliminates nodes that can't satisfy the claim's selectors, at **Reserve** tentatively allocates the specific device, and at **PreBind** writes the finalized allocation back into the ResourceClaim status — all using structured data the scheduler owns, so no round-trip to a vendor controller during scheduling.

**MIG as the concrete cost lever.** Multi-Instance GPU (MIG) is a hardware-level partition of an A100/H100 into up to 7 isolated slices, each with its own memory and compute fraction. Combined with DRA's attribute-based claims, MIG is *the* mechanism that makes "run a small inference workload on 1/7th of a GPU instead of monopolizing the whole card" both schedulable (the claim selects a MIG profile attribute) and cost-attributable (your operator can price a `1g.10gb` claim differently from a whole-GPU claim). Without DRA, MIG slices have to be advertised as separate device-plugin resource names per profile — clunky and hard to reason about at fleet scale; with DRA, they're just another attribute on the ResourceSlice.

**Why this matters for cost/MIG (self-check a):** integer counts force "a GPU"; DRA lets you request "the *cheapest device that meets my constraint*" — a MIG `1g.10gb` slice for a small inference Pod instead of monopolizing a whole H100, or "any GPU with ≥24 GB" so a job lands on a cheaper card when it fits. Attribute-based requests are the substrate a cost-aware policy needs; you can't optimize spend across a fleet you can only count.

### 4. Kueue — quota, gang, and topology-aware admission

**Kueue** (`kueue.sigs.k8s.io`) is a Kubernetes-native **job queueing** layer that sits *in front of* the scheduler. It doesn't place Pods; it decides **whether a whole workload is admitted** and, until then, keeps the Job **suspended** (`.spec.suspend=true`, no Pods created). **CoreWeave runs Kueue** on CoreWeave Kubernetes Service for customer training and batch-inference workloads — this is production, not a lab curiosity.

Key concepts:

- **ClusterQueue** — cluster-scoped quota pool: how much CPU/memory/`nvidia.com/gpu` a tenant may consume, with borrowing/lending across cohorts and fair-sharing.
- **LocalQueue** — namespaced; a tenant submits jobs to a LocalQueue, which points at a ClusterQueue. Namespace-scoped handle onto cluster-scoped quota.
- **ResourceFlavor** — how Kueue models *heterogeneous* node types: A100 nodes vs H100 nodes vs spot vs on-demand are different flavors of the same logical resource (`nvidia.com/gpu`). Quota is assigned per flavor, so a ClusterQueue can say "20 A100-GPU-hours and 8 H100-GPU-hours" rather than one undifferentiated GPU count. This is the piece that makes Kueue **cost-policy-relevant**, not just capacity-relevant — a `GPUCostPolicy` maps naturally onto flavor-scoped quota.
- **Workload** — Kueue's internal object wrapping a Job (or RayJob/JobSet/etc.) and its **pod sets**.
- **All-or-nothing / gang admission** — a Workload is unsuspended only when quota for **all** its pod sets can be reserved **at once**. This is the direct answer to §2(b): no partial placement, no half-held deadlock. The job waits, whole, in the queue until it fits, whole.
- **Topology-Aware Scheduling (TAS)** — Kueue tracks free capacity per topology domain (rack/block/node) and admits with a placement intent: `required` (all pods in one domain — tight all-reduce communication), `preferred` (same domain if possible, else spread), or `unconstrained`. Critical when inter-GPU bandwidth dominates training throughput.

Mental model: **Kueue gates the door (admission + quota + gang), the scheduler seats the guests (per-Pod node choice), DRA picks the exact chair (device by attribute).**

## Perspectives

**Scheduler-internals perspective.** Filter vs Score is a hard-gate-vs-soft-bias distinction that generalizes far beyond GPUs; internalizing it here pays off anywhere you touch scheduling — topology spread, taints, node affinity, or any future extension point work.

**Device-modeling perspective.** The shift from "GPU as an opaque integer" to "GPU as a set of typed, queryable attributes" is the same kind of modeling upgrade as moving from untyped config to a typed API. DRA is API-machinery (Lesson 02's GVK/scheme/CEL world) applied to hardware — the same discipline, a different substrate.

**Admission/quota perspective.** Kueue answers a fundamentally different question ("should this job start at all, right now") than the scheduler ("which node for this Pod"). Conflating the two is the single most common design-review mistake at this layer — a reviewer who hears "Kueue schedules the Pod" should immediately push back.

**Economics/FinOps perspective (the differentiator).** This is where your `gpu-cost-operator`'s signal actually has somewhere to plug in: a `costScore` can bias Score (per-Pod), inform ResourceFlavor/quota weighting (per-job admission), or shape DeviceClass/claim selectors (per-device) — three real integration points, each with different blast radius and build cost. That's exactly the Practice deliverable's design-doc comparison, and it's the intersection almost nobody else in the interview pool is fluent in.

## Real-world use cases

- **CoreWeave, "Kueue: A Kubernetes-Native System for AI Training Workloads"** — <https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads> — CoreWeave runs Kueue in production for customer training/batch-inference workloads on CoreWeave Kubernetes Service; shows gang admission and quota management at real GPU-fleet scale, not a lab demo.
- **Google Cloud Blog, "Kubernetes device management with DRA Dynamic Resource Allocation"** — <https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation> — a second major cloud vendor's production treatment of DRA, showing the ResourceSlice/DeviceClass/ResourceClaim flow on GKE.
- **NVIDIA Blog, "NVIDIA Donates Dynamic Resource Allocation Driver for GPUs to Kubernetes Community"** — <https://blogs.nvidia.com/blog/nvidia-at-kubecon-2026/> — makes the "DRA driver as DaemonSet" concept concrete with the actual GPU vendor shipping it; directly names the kind of company (NVIDIA) most relevant to this learner's target roles.
- **AKS Engineering Blog, "Running more with less: Multi-instance GPU (MIG) with Dynamic Resource Allocation (DRA) on AKS"** — <https://blog.aks.azure.com/2026/03/03/multi-instance-gpu-with-dra-on-aks> — a third major cloud vendor documenting the exact MIG+DRA combination this lesson calls the concrete cost lever; the clearest available production pairing for self-check (a).
- **Uber Engineering, "Uber's Journey to Ray on Kubernetes: Resource Management"** — <https://www.uber.com/en-DE/blog/ubers-journey-to-ray-on-kubernetes-resource-management/> — shows a hyperscaler solving the same GPU/ML resource-pooling problem with a custom scheduling layer on top of Kubernetes rather than Kueue; a useful contrast case for "how would you decide build-vs-adopt at this layer."

## Worked example

Walk the lifecycle of one 8-Pod, 8-GPU distributed training job on a heterogeneous cluster (some A100 40 GB nodes, some H100 80 GB nodes), narrating the three layers, then quantify the cost delta.

1. **Submit.** User creates a `Job` (8 parallel Pods, each claiming 1 GPU via a ResourceClaimTemplate that selects `memory >= 40Gi`) referencing a LocalQueue. Kueue's Job integration sets `suspend=true` — **zero Pods exist yet.**
2. **Kueue admission.** Kueue wraps it as a Workload with one pod set of size 8. Its ClusterQueue has `nvidia.com/gpu` quota of 16, currently 12 in use → only 4 free. **All-or-nothing:** 8 > 4, so the Workload **waits in the queue.** No GPUs parked, no deadlock — the job simply hasn't started. When 4 more free up, quota for all 8 is reservable at once; Kueue admits and sets `suspend=false`.
3. **Pods created → scheduler.** Now 8 Pods hit the scheduling queue. For each: **Filter** drops nodes with no free GPU *and* (via the DRA plugin, reading ResourceSlices) nodes whose devices don't meet `memory >= 40Gi`. Both A100-40 and H100-80 nodes survive.
4. **Score.** Your `gpu-cost-operator` has annotated nodes with a costScore. A Score plugin (or a scheduler configured to weight it) ranks A100 nodes above H100 nodes for a job that only asked for 40 GB → **bias toward the cheaper card, without excluding H100** if that's all that's free.
5. **Reserve → PreBind → Bind.** DRA plugin tentatively allocates the specific device at **Reserve**, finalizes the ResourceClaim at **PreBind**, and the Pod is bound. If TAS was configured, Kueue's admission already constrained these 8 to a topology domain for fast all-reduce.

**Cost-quantified variant.** Using illustrative, dated (2025–2026) snapshot rates — A100-40GB at **$2.50/GPU-hour**, H100-80GB at **$5.00/GPU-hour** — an 8-GPU job running 4 hours costs **8 × 4 × $2.50 = $80** if Score correctly biases it entirely onto A100 nodes that fit its 40 GB requirement, versus **8 × 4 × $5.00 = $160** if Score is absent or miscalibrated and the job lands entirely on H100 nodes it didn't need. That $80 delta on a single 4-hour job, multiplied across thousands of jobs a month on a real fleet, is the dollar-denominated version of "why Score, not Filter" — the self-check answer with a number attached.

Note where each decision lived: **admission/gang = Kueue**, **node = scheduler Score/Filter**, **which device = DRA**. If you had tried to make cost a *Filter*, step 3 would have gone Pending the instant A100s filled — which is why cost is a *Score*.

## Practice

**Deliverable: a 1-page design doc, committed** at `../practice/gpu-cost-operator/docs/design.md`.

Title: **"How would `GPUCostPolicy` influence GPU placement?"** Your `gpu-cost-operator` reconciles a `GPUCostPolicy` CR and emits a per-node (or per-device) **costScore** signal. Design *how that signal reaches a placement decision*, comparing **three** approaches with explicit tradeoffs:

1. **Score-plugin approach** — operator writes costScore as a node annotation/label; a scheduler Score plugin (or `NodeResourcesFit`/`scheduler-plugins` config) reads it to bias placement. Tradeoffs: needs scheduler config or a custom plugin (build/maintain/blast-radius); operates per-Pod at node granularity; can't see whole-job or quota.
2. **Kueue-quota approach** — encode cost policy as ClusterQueue quotas / ResourceFlavor / cohort borrowing so expensive GPUs are rationed at admission. Tradeoffs: no plugin to maintain; enforces *budget* and gang cleanly; but it gates jobs, it doesn't pick the cheaper *node* among feasible ones — coarser, admission-time.
3. **DRA-attributes approach** — express cost preference through DeviceClass selection / ResourceClaim selectors so claims target cheaper device classes or MIG profiles. Tradeoffs: most precise (device-level, MIG-aware); requires DRA (1.34+) and driver-published attributes; policy lives in claim/class definitions, further from a central operator.

Then **propose the costScore annotation your operator emits**: exact key (e.g. `gpu-cost.io/cost-score`), value semantics (lower = cheaper? normalized 0–100?), what it's attached to (Node vs ResourceSlice-adjacent), and how it's kept fresh. State which of the three you'd ship first and why.

**Acceptance:** `design.md` committed, covering all three approaches with tradeoffs and a concrete costScore proposal. **Design artifact — no plugin code.** This design doc is also acceptance item 8 in the [deliverable's checklist](../practice/gpu-cost-operator/README.md#acceptance-criteria-matches-the-checkpoint).

## Common pitfalls

1. **Believing DRA has already replaced the device-plugin model everywhere.** It hasn't — device plugins remain the default/only path on many real clusters. DRA requires Kubernetes 1.34+ *and* a DRA-aware vendor driver, so a design for a 40-cluster fleet with mixed Kubernetes versions has to handle both paths, not assume DRA universally.
2. **Believing Kueue schedules Pods.** It doesn't; conflating "Kueue admits the Workload" with "the scheduler placed the Pod" is the most common design-doc mistake, and the fastest way to lose credibility in a design review at a Kueue-running shop like CoreWeave.
3. **Assuming ResourceFlavor is just a label selector.** It also carries quota/borrowing semantics (cohorts, lending) that a naive label-based mental model misses — this is what makes it cost-policy-relevant rather than just a node-selector alias.
4. **Designing a Score-plugin costScore signal without accounting for NormalizeScore and plugin weighting.** A raw, unnormalized costScore can be silently drowned out by other Score plugins' weights if it isn't calibrated to the same 0–100 range — a design that "should" bias placement but empirically doesn't.
5. **Encoding a soft cost preference as a Filter.** Covered above as the load-bearing distinction, but common enough to restate as a pitfall on its own: it turns "prefer cheap" into "only cheap," and a burst of demand for the cheap tier produces Pending Pods instead of a bias toward the pricier tier.

## Self-check

- What does DRA request-by-attribute enable for cost/MIG that device-plugin integer counts cannot? **Answer:** The device-plugin model advertises GPUs as an opaque integer extended resource (`nvidia.com/gpu: 8`) — all devices are fungible and the scheduler can only *count* them. DRA (GA in 1.34, KEP-4381) lets a ResourceClaim request devices by **attribute** via CEL selectors over ResourceSlice data: memory, MIG profile, product name, compute capability. That enables landing a small workload on a MIG `1g.10gb` slice instead of a whole H100, and requesting "cheapest device that meets ≥40 GB" so a job packs onto a cheaper A100 when it fits. You cannot optimize spend or MIG utilization across a fleet you can only count — attribute requests are the substrate cost-awareness needs.
- Which framework extension point biases placement toward cheaper GPUs, and why Score, not Filter? **Answer:** **Score** (with NormalizeScore + weighting). Filter is a hard boolean gate — a node that fails Filter is removed from consideration entirely, so a Pod failing every node's Filter goes Pending or triggers preemption. Encoding cost as a Filter would make Pods go Pending the moment cheap capacity is exhausted — a self-inflicted outage. Score never removes a node; it ranks the survivors. "Prefer cheaper, but still run on an expensive GPU if that's all that's free" is exactly a *soft preference among feasible nodes* → Score.
- What does Kueue's gang / all-or-nothing admission solve that the default scheduler cannot? **Answer:** The default scheduler places one Pod at a time, greedily, with no notion of a job as a unit. A 32-Pod training job can get 30 Pods placed and 2 left Pending — **30 expensive GPUs parked idle** waiting for the last 2 — and interleaved jobs can **deadlock**, each holding a fraction and none able to run. Kueue admits a Workload **only when quota for all its pod sets can be reserved at once**, keeping the Job suspended (no Pods created) until it fits whole. That eliminates partial placement and the associated GPU parking/deadlock — the framework's Permit point *enables* gang semantics, but Kueue delivers it as job-level admission plus quota.
- Your `gpu-cost-operator` needs to bias placement toward cheaper GPUs *and* enforce a hard per-team monthly GPU-hour budget *and* let a small inference Pod use 1/7th of an H100. Map each requirement to the layer that owns it. **Answer:** "Bias toward cheaper GPUs" → **scheduler Score** (a soft preference among feasible nodes; wrong layer for a hard cap because Score can't reject). "Hard per-team monthly GPU-hour budget" → **Kueue quota** (ClusterQueue/ResourceFlavor with cohort borrowing; it's an admission-time gate, which is exactly what "hard budget" requires — reject/queue the whole job rather than let it start and go over). "1/7th of an H100" → **DRA claim + MIG**, because only DRA's attribute model can express "a MIG `1g.10gb` slice" as a request target; device-plugin integers can't. Each answer can't be done at the other layers: Score can't enforce a hard cap (it only ranks), Kueue can't pick a specific node or device, and DRA doesn't gate whole-job admission.

## Connections & what's next

This lesson sits directly on top of Lesson 01's apiserver/watch model (DRA drivers publish ResourceSlices the same way any client publishes any object — no back door), Lesson 02's typed-vs-CEL API machinery (ResourceClaim selectors are CEL over structured attributes, the same tool Lesson 05's CRD validation used), and Lesson 03's reconcile discipline (Kueue's own controllers admit/suspend Workloads through the same level-triggered reconcile loop this whole module teaches). It also directly informs the `gpu-cost-operator`'s roadmap: the design doc you write here is the seed of real placement-influencing work in later modules.

Next: **Lesson 10 — the capstone build.** Every earlier lesson's piece (CRDs, reconcile, finalizer, webhook, RBAC) plus this lesson's design doc get assembled into `gpu-cost-operator` v0.1 and proven under envtest. Scheduling stays a design artifact here — Lesson 10 is where the machinery you've built actually has to run, together, and pass tests.

## References & further reading

**Primary sources**
- Kubernetes docs, Scheduling Framework (extension points and cycle) — <https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/> — read for the authoritative extension-point list and cycle diagram.
- Kubernetes docs, Dynamic Resource Allocation (concept docs) — <https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/> — read for the DeviceClass/ResourceClaim/ResourceSlice object model.
- Kubernetes blog, "Kubernetes v1.34: DRA updates" (GA announcement) — <https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/> — read for what graduated to GA and API stability (`resource.k8s.io/v1`); **version-sensitive**, verify against your cluster's version.
- KEP-4381, "DRA: structured parameters" — <https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4381-dra-structured-parameters> — read for the GA design rationale and the KEP-3063-withdrawal history.
- Kueue docs, Concepts (ClusterQueue/LocalQueue/ResourceFlavor/Workload) — <https://kueue.sigs.k8s.io/docs/concepts/> — read for the full object model.
- Kueue docs, All-or-nothing (gang) admission — <https://kueue.sigs.k8s.io/docs/concepts/all_or_nothing/> — read for the exact semantics behind self-check (c).
- Kueue docs, Topology-Aware Scheduling — <https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/> — read for the `required`/`preferred`/`unconstrained` placement intents.

**Real-world engineering blogs**
- CoreWeave, "Kueue: A Kubernetes-Native System for AI Training Workloads" — <https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads> — Kueue in production for AI training/batch inference at a GPU-cloud provider.
- Google Cloud Blog, "Kubernetes device management with DRA Dynamic Resource Allocation" — <https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation> — a second vendor's production DRA treatment.
- NVIDIA Blog, "NVIDIA Donates Dynamic Resource Allocation Driver for GPUs to Kubernetes Community" — <https://blogs.nvidia.com/blog/nvidia-at-kubecon-2026/> — the GPU-vendor DRA driver, concretely.
- AKS Engineering Blog, "Running more with less: Multi-instance GPU (MIG) with DRA on AKS" — <https://blog.aks.azure.com/2026/03/03/multi-instance-gpu-with-dra-on-aks> — production MIG+DRA pairing.
- Uber Engineering, "Uber's Journey to Ray on Kubernetes: Resource Management" — <https://www.uber.com/en-DE/blog/ubers-journey-to-ray-on-kubernetes-resource-management/> — a hyperscaler's custom-scheduler alternative to Kueue.

**Deeper dives**
- `kubernetes-sigs/scheduler-plugins` — <https://github.com/kubernetes-sigs/scheduler-plugins> — out-of-tree plugins including co-scheduling/gang; reference for Score-plugin design, not required to build.
