---
lesson: "06.6"
title: "Topology-aware placement — packing a gang into one NVLink domain"
module: "06"
concept: "Topology-aware placement — packing a gang into one NVLink domain"
status: not-started
est_time: "6h"
artifacts: []
---
# 06.6 · Topology-aware placement — packing a gang into one NVLink domain

> **Concept.** A gang that spreads across NVLink domains or racks pays a 2–5× collective-communication tax; Kueue Topology-Aware Scheduling packs it into one domain via node labels so NCCL collectives never cross the slow link.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + showback](../practice/kueue-showback/README.md)

## Why this matters

You already know from module 02b that GPUs inside one NVLink domain talk at hundreds of GB/s, and that stepping outside that domain — across the PCIe host boundary, across a rack, across a network spine — drops you by an order of magnitude to what the NIC and fabric can carry. That is interconnect physics; this lesson does **not** re-teach it (reference 02b for the NVLink-domain bandwidth cliff and Topology Manager). This lesson is about the *scheduling consequence*: a gang scheduler that satisfies your GPU count but ignores *where* those GPUs sit will happily give you 8 GPUs as 4-here-and-4-across-the-spine, and your distributed training job then runs every all-reduce at fabric speed instead of NVLink speed.

The cost is not subtle and it is not a tail case. Synchronous data-parallel training is gated on the all-reduce at every step. If the collective's slowest link is 5–10× slower because half the ring crosses the spine, the whole step stalls on it — and **every GPU in the gang idles while it waits.** You are billed for 8 GPUs and getting the throughput of a badly-bottlenecked 4. On a neocloud H100 gang at multi-dollar-per-GPU-hour, a job that runs 2× longer because of topology-blind placement is a 2× cost overrun that no dashboard attributes to scheduling — it just looks like "training is slow." Being the engineer who ties slow training back to a 4+4 split gang, and fixes it with a topology constraint, is squarely the cost/FinOps differentiator you are building.

## What's new here

Through lesson 5 the scheduling decision was *whether* and *when* a gang is admitted (quota, borrowing, gang all-or-nothing). New here is *where*: the gang must land **co-located within a topology domain**, and the platform needs a vocabulary to express "these N pods must share a rack / an NVLink block."

- **Kueue Topology-Aware Scheduling (TAS)** — a first-class Kueue feature (beta) that lets a ClusterQueue place a PodSet within a named topology level.
- **The `Topology` object** — declares an ordered hierarchy of levels (block → rack → host) mapped to **node labels**.
- **Per-PodSet annotations** — `required` (hard: fit in one domain at this level or don't admit) vs `preferred` (soft: try to pack, degrade gracefully to a wider level).
- **The interaction with gang** — TAS and gang are *both* constraints that must hold simultaneously; TAS chooses a topology domain that can fit the *whole* gang, gang enforces all-or-nothing within it.

## Core notes

### The physics-to-scheduling bridge (one line, then move on)

02b established the bandwidth hierarchy: intra-node NVLink/NVSwitch ≫ intra-rack fabric ≫ cross-rack/spine. TAS's whole job is to keep a collective's communication *inside the fastest level it can*. Everything below is how you express that in Kueue.

### The `Topology` object — naming the hierarchy

TAS reads a hierarchy of **node labels**, coarsest first. Cloud providers expose these (or you fabricate them from your DCIM/rack inventory):

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: Topology
metadata:
  name: gpu-fabric
spec:
  levels:
    - nodeLabel: "cloud.provider.com/topology-block"   # coarsest — NVLink block / super-pod
    - nodeLabel: "cloud.provider.com/topology-rack"     # rack
    - nodeLabel: "kubernetes.io/hostname"               # finest — single node (host)
```

Levels are **ordered coarse→fine**. Every node that participates must carry every label in the hierarchy. The lowest level is conventionally `kubernetes.io/hostname` — one node — which lets you express "pack onto a single host" for a gang that fits in one box.

### Wiring it to a ClusterQueue

A ResourceFlavor points at the Topology; the ClusterQueue uses that flavor. This is the same flavor mechanism from lessons 3–4, now carrying a topology reference:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: gpu-h100
spec:
  nodeLabels:
    accelerator: h100
  topologyName: gpu-fabric          # <-- binds this flavor to the hierarchy
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: training
spec:
  resourceGroups:
    - coveredResources: ["nvidia.com/gpu", "cpu", "memory"]
      flavors:
        - name: gpu-h100
          resources:
            - name: "nvidia.com/gpu"
              nominalQuota: 64
```

### Expressing the placement — `required` vs `preferred`

The workload asks for a topology level on its **PodSet**, via annotations on the pod template (the Job's `spec.template`):

```yaml
# HARD: the whole PodSet must fit inside ONE rack, or it is not admitted.
metadata:
  annotations:
    kueue.x-k8s.io/podset-required-topology: "cloud.provider.com/topology-rack"
```

```yaml
# SOFT: try to pack into one rack; if impossible, fall back to the next
# coarser level that fits (block), and only then spread. Job still runs.
metadata:
  annotations:
    kueue.x-k8s.io/podset-preferred-topology: "cloud.provider.com/topology-rack"
```

Decision criteria:

- Use **`required`** when the collective genuinely cannot tolerate a cross-domain link — large synchronous data-parallel / tensor-parallel training where all-reduce dominates step time. Accept the cost: if no single rack can hold the gang right now, the workload **waits** (or never admits) rather than running slow. That is usually the right trade for expensive training — a job that waits an hour for a clean rack beats a job that runs 2× slow for a week.
- Use **`preferred`** when the workload benefits from locality but correctness/throughput degrade gracefully — data-parallel with gradient compression, embarrassingly-parallel work, inference replicas, or anything where "packed if possible, spread if not" beats waiting. Also the safer default when your topology labels or capacity are not yet trustworthy.
- Choose the **level** to match the collective's reach: a gang that fits in one node → `kubernetes.io/hostname`; an 8–16-GPU gang spanning a couple of boxes in a rack → `topology-rack`; a super-pod-scale job → `topology-block`.

### How TAS and gang interact — both must hold

This is the subtle part and a favorite interview probe. TAS and gang are **conjunctive** constraints:

1. Kueue admits a Workload only when quota is available (lessons 3–4) **and** TAS can find a single domain at the requested level with enough free capacity for the **entire** PodSet.
2. Gang (delegated to the underlying scheduler / coscheduling, per lesson 5) then enforces all-or-nothing placement — but now confined to the domain TAS selected.
3. With `required`, if no single domain fits the whole gang, the Workload is **not admitted** — it does not partially place and then deadlock (the lesson-2 failure mode) and it does not spread. The gang's atomicity and the topology's locality are satisfied together or not at all.

The reason you need both: gang alone gives you "all 8 pods or none" but says nothing about *where* — it will cheerfully give you an all-8 placement split 4+4 across the spine. TAS alone (without gang) could place some pods in the good domain and leave the rest pending. Together they give the only correct outcome for tightly-coupled training: **all pods, in one fast domain, or wait.**

### Failure modes to name

- **Label drift / missing labels.** A node missing one Topology level label is invisible to TAS at that level — capacity silently shrinks. Validate label coverage as a cluster invariant.
- **`required` starvation.** Over-aggressive `required` at a fine level on a fragmented cluster can leave a job pending indefinitely while GPUs sit idle in scattered domains. This is the topology analog of the fragmentation problem KAI's consolidation attacks (lesson 5) — TAS won't defrag for you; it waits.
- **Level too coarse.** `required: topology-block` on a job that needed rack-level locality "passes" but still lets the gang spread within the block across racks — you constrained the wrong level and get the slow link anyway.

## Worked example

**Scenario.** An 8-GPU synchronous data-parallel H100 training job (one PyTorchJob, 8 workers, gang via coscheduling under Kueue). The `training` ClusterQueue has 64 GPUs across 4 racks of 2× 8-GPU nodes.

**Topology-blind baseline.** Without TAS, the scheduler satisfies "8 GPUs" as 4 GPUs on a node in rack A + 4 on a node in rack C. Every all-reduce ring now includes a cross-rack (spine) hop. Per 02b's bandwidth hierarchy, the spine link is ~5–10× slower than NVLink; the collective is gated on its slowest link, so per-step all-reduce time balloons and **all 8 GPUs stall** on each step. Observed: step time up ~2–5×, GPU utilization (SM-active) collapses during the comm phase, wall-clock — and therefore GPU-hour cost — up 2×+. Nothing in the quota dashboard flags it; it reads as "8/8 GPUs admitted, job running."

**With TAS `required` at rack level.** Annotate the PodSet:

```yaml
kueue.x-k8s.io/podset-required-topology: "cloud.provider.com/topology-rack"
```

Now Kueue admits the Workload only when a single rack has 8 free GPUs (its two nodes), and the gang places all 8 workers within that rack. Every all-reduce link is now intra-rack (and intra-node NVLink where possible). Step time returns to compute-bound; the job finishes in half the wall-clock. If no rack currently has 8 free, the job **waits for one to drain** — correct: waiting an hour beats running the whole job at half speed.

**Relaxing to `preferred`.** Swap `required`→`preferred`. If a clean rack exists, identical packing. If the cluster is fragmented (every rack has one busy node), TAS falls back to a coarser fit and the gang spreads — the job runs *now*, slower, instead of waiting. That is the right call for a deadline-bound but comm-tolerant job, and the wrong call for a comm-bound one. The choice *is* the FinOps decision: pay in latency (wait) or pay in throughput (spread).

## Practice — TAS co-placement demo on kind

This feeds the deliverable's topology artifact. You do not need real GPUs — fake the labels; TAS reasons over labels, not hardware.

1. **Spin up a multi-node kind cluster** (e.g. 1 control-plane + 4 workers). Install Kueue and enable the TAS feature gate (`TopologyAwareScheduling`) per the setup doc.
2. **Fake the fabric labels.** Label the 4 workers as two racks in one block:
   ```
   kubectl label node kind-worker  cloud.provider.com/topology-block=b1 cloud.provider.com/topology-rack=r1
   kubectl label node kind-worker2 cloud.provider.com/topology-block=b1 cloud.provider.com/topology-rack=r1
   kubectl label node kind-worker3 cloud.provider.com/topology-block=b1 cloud.provider.com/topology-rack=r2
   kubectl label node kind-worker4 cloud.provider.com/topology-block=b1 cloud.provider.com/topology-rack=r2
   ```
   (`kubernetes.io/hostname` already exists per node.)
3. **Apply** the `Topology` (block/rack/host), a `ResourceFlavor` referencing it, and a TAS-enabled `ClusterQueue` + `LocalQueue`. Use an extended/fake resource or CPU as the "GPU" stand-in so kind can schedule it — size requests so exactly one rack (two nodes) can hold the gang.
4. **Submit a gang** (a Job / JobSet with the gang plus `podset-required-topology: cloud.provider.com/topology-rack`). Confirm co-placement: `kubectl get pods -o wide` — **all pods land on nodes sharing one `topology-rack` value.**
5. **Relax to `preferred`,** shrink free capacity so no single rack fits (cordon or occupy one node per rack), resubmit, and **observe the gang spread across racks** instead of pending.

**Acceptance:** a committed TAS co-placement demo in the deliverable containing (a) the `Topology` + TAS `ClusterQueue` + `ResourceFlavor` manifests, (b) the node-labeling commands, (c) `kubectl get pods -o wide` output for the `required` run showing every gang pod on one rack, and (d) the `preferred` run showing spread when no rack fits, with one sentence explaining why `required` waited/packed and `preferred` spread. If a reviewer can rerun your commands on a fresh kind cluster and reproduce both outcomes, it passes.

## Self-check

**Q1. Why does a topology-spread all-reduce gang waste GPU-hours? (Tie to 02b interconnect bandwidth.)**
**Answer:** Synchronous training does an all-reduce every step and the collective is gated on its slowest link. Per 02b's bandwidth hierarchy, a cross-rack/spine hop is ~5–10× slower than intra-NVLink-domain, so a gang split across domains runs every all-reduce at fabric speed. Because the step can't proceed until the collective completes, *all* GPUs in the gang idle waiting on the slow link — you pay for N GPUs but get the throughput of a comm-bottlenecked fraction, and wall-clock (hence GPU-hour cost) roughly doubles or worse.

**Q2. `required` vs `preferred` topology annotation — behavior difference?**
**Answer:** `required` is a hard constraint: the entire PodSet must fit within one domain at the named level or the Workload is **not admitted** — it waits rather than spreading. `preferred` is soft: TAS tries to pack into the named level, but if no single domain fits it falls back to a coarser level and lets the gang spread so the job still runs now. `required` trades latency (waiting) for guaranteed locality; `preferred` trades locality for guaranteed progress. Pick `required` for comm-bound training, `preferred` for comm-tolerant or deadline-bound work.

**Q3. How does TAS interact with gang scheduling (both must be satisfied)?**
**Answer:** They are conjunctive. Kueue admits only when quota is free **and** TAS finds a single domain at the requested level with room for the *whole* PodSet; gang (delegated to the underlying scheduler) then enforces all-or-nothing placement confined to that domain. Gang alone guarantees "all pods or none" but not *where* (it can place all 8 split 4+4 across the spine); TAS alone could place some pods well and leave others pending. Together the only admissible outcome for tightly-coupled training is all pods, in one fast domain, or wait — no partial placement, no cross-domain spread.

## Resources

1. **Kueue — Topology-Aware Scheduling (concept)** — https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/ — the authoritative model: `Topology` levels, `required`/`preferred` semantics, PodSet annotations, and gang interaction.
2. **Kueue — Setting up Topology-Aware Scheduling (task)** — https://kueue.sigs.k8s.io/docs/tasks/manage/setup_topology_aware_scheduling/ — the feature gate, ResourceFlavor `topologyName` wiring, and node-label requirements you follow for the kind demo.
3. **Module 02b — Host topology (NVLink domains, Topology Manager)** — the interconnect-bandwidth physics this lesson builds on; reference for *why* cross-domain links are slow, which TAS exists to avoid.
