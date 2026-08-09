---
lesson: "06.5"
title: "Scheduler alternatives — Volcano, NVIDIA KAI, and when to pick which"
module: "06"
concept: "Scheduler alternatives — Volcano, NVIDIA KAI, and when to pick which"
status: not-started
est_time: "6h"
artifacts: []
---
# 06.5 · Scheduler alternatives — Volcano, NVIDIA KAI, and when to pick which

> **Concept.** Kueue, Volcano, and NVIDIA KAI solve overlapping but differently-shaped problems; pick by tenancy model and workload lineage, and know they can compose rather than compete.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + showback](../practice/kueue-showback/README.md)

## Why this matters

You have spent lessons 1–4 building a mental model of Kueue: a quota-and-queue admission layer that sits *above* the default scheduler, borrows across cohorts, and preempts by workload rather than by pod. That model is correct and it is what most cloud-native platform teams should reach for first. But in a GPU-heavy interview loop at a CoreWeave / NVIDIA / neocloud shop, "we use Kueue" is a starting position, not an answer. The follow-up is always: *why not Volcano? why not KAI? what happens when a tenant needs fractional GPUs, or an MPI job needs true gang semantics Kueue doesn't own?*

The failure mode this lesson prevents is picking a scheduler by familiarity and then discovering at scale that it structurally cannot express what a tenant needs — you can't retrofit DRF fairness or fractional-GPU sharing onto a scheduler that has no concept of them. The cost angle is sharper: the wrong scheduler leaves GPUs fragmented (stranded 3-GPU holes on 8-GPU nodes that no job can use) or idle behind a queue that can't borrow. On a fleet of 40+ clusters at neocloud GPU prices, single-digit-percent fragmentation is a seven-figure line item. Your differentiator here is being the person who can defend a scheduler choice in FinOps terms, not just feature terms.

This lesson is deliberately "know when, not how." You are not deploying Volcano or KAI today. You are learning to read their core manifests, place each on a decision matrix, and defend the choice.

## What's new here

Everything through lesson 4 assumed one scheduling philosophy: **quota-first, cloud-native, gang delegated.** Three things are new:

1. **Volcano** — a batch scheduler with HPC/MPI lineage (descended from kube-batch, a CNCF project). Its primitives are the `PodGroup` and **Dominant Resource Fairness (DRF)**. Gang scheduling is native and first-class, not delegated. It is heavier operationally and its worldview is "a batch system that happens to run on Kubernetes."

2. **NVIDIA KAI-Scheduler** — the scheduling engine extracted from Run:ai (NVIDIA acquired Run:ai late 2024; open-sourced the scheduler April 2025 under Apache 2.0; now a CNCF Sandbox project). Its distinguishing features are **fractional GPU sharing**, **workload consolidation / bin-packing** (active defragmentation), **hierarchical fair-share queues**, and topology-aware + hierarchical gang scheduling that integrates with **Grove** for disaggregated serving. **KAI is young and volatile** — APIs, the CRD surface, and even the repo location have shifted; treat any specific field name as version-dependent and verify against the tag you deploy.

3. **Composition** — the key insight that dissolves the false "vs." framing. Kueue governs *admission and quota*; a gang scheduler governs *placement*. You can run Kueue for cohort borrowing and showback while delegating the actual gang placement to Volcano or the coscheduling plugin. They stack.

## Core notes

### The three schedulers, by primitive

| | **Kueue** | **Volcano** | **NVIDIA KAI** |
|---|---|---|---|
| Lineage | SIG-driven, cloud-native | kube-batch / HPC / MPI | Run:ai commercial engine |
| Core object | `Workload` + `ClusterQueue`/`LocalQueue` | `PodGroup` + `Queue` | `PodGroup` (auto via PodGrouper) + `Queue` |
| Fairness model | Cohort quota + borrowing + fair sharing | **DRF** (dominant resource fairness) | **Hierarchical fair-share** (tree of queues) |
| Gang | **Delegated** (to scheduler/coscheduling) | **Native, strong** | Native (incl. hierarchical/topology-aware) |
| Fractional GPU | No | No (whole-device) | **Yes** (annotation-based sharing) |
| Bin-pack / consolidation | No (relies on kube-scheduler) | Plugin-level bin-packing; NVIDIA guide for anti-fragmentation | **Yes — active consolidation** (evicts+reschedules to defrag) |
| Topology-aware | **Yes (TAS)** — see 06.6 | Via plugins / NUMA + network topology | Yes (topology-aware gang, Grove) |
| Operational weight | Light (admission controller) | Heavier (full batch stack) | Medium; **volatile** |
| Best when | Multi-tenant quota + borrowing, cloud-native | HPC/MPI/Spark, hard gang, DRF fairness | GPU sharing, fragmentation-sensitive, Run:ai-style |

### Volcano — what it is really for

Volcano is what you pick when the *workload* is a batch/HPC job and the scheduling contract is "all-or-nothing, and share fairly across multiple resource dimensions." Its `PodGroup` carries `minMember` / `minResources` — the scheduler will not bind *any* pod of the group until the minimum can be satisfied, which is the textbook cure for the partial-gang deadlock you saw in lesson 2 (default scheduler binds 6 of 8 workers, deadlocks, holds GPUs hostage). DRF is the differentiator over Kueue's quota model: instead of "you get N GPUs," DRF equalizes each tenant's *dominant* resource share, so a GPU-bound tenant and a CPU/memory-bound tenant get fair treatment along the axis that actually constrains each of them. Volcano also carries the MPI/Spark/Ray/PyTorch operator integrations that HPC shops already speak.

Read a minimal Volcano `PodGroup`:

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: mpi-allreduce
spec:
  minMember: 8                      # gang: nothing schedules until 8 fit
  minResources:                     # DRF accounts against these
    nvidia.com/gpu: "8"
    cpu: "64"
    memory: 512Gi
  queue: hpc-research               # Volcano Queue, DRF-weighted
  priorityClassName: research-high
```

Pods opt in with `schedulerName: volcano` and the `scheduling.k8s.io/group-name` annotation. The `Queue` object holds `weight` (DRF share) and optional `capability` caps. Note the whole-device GPU request — Volcano schedules GPUs as indivisible units.

**Fragmentation caveat.** Volcano's default bin-packing is not fragmentation-aware for GPUs out of the box. NVIDIA published a practical guide on configuring the `binpack` plugin and node-ordering so multi-GPU gangs don't strand partial nodes — required reading if you run Volcano on 8-GPU boxes (see Resources).

### NVIDIA KAI — what it does that the others don't

Two capabilities are genuinely distinct:

1. **Fractional GPU sharing.** A pod requests a *fraction* of a GPU via annotation (e.g. `gpu-fraction: "0.5"`) or a memory slice, and KAI time-/memory-shares the physical device across pods. This is not MIG (hardware partitioning) — it's a software-level share with soft isolation, aimed at notebooks, inference, and dev workloads that waste a whole A100/H100 each. For a FinOps story this is the highest-leverage feature in the lesson: it directly attacks the "one Jupyter notebook pinning a $30k GPU at 4% utilization" waste.

2. **Consolidation (active bin-packing / defrag).** When a pending job can't fit because free GPUs are scattered across nodes, KAI will *move running pods* to compact the free space into a contiguous block, then place the pending job. Kueue does not do this — Kueue admits or waits, but never reshuffles already-running workloads to defragment. This is the difference between "3 GPUs free here, 3 free there, your 6-GPU job waits forever" and "KAI drains the fragments and your job runs."

KAI also brings **hierarchical fair-share** (a tree of queues with weights, so an org → team → project hierarchy shares fairly at each level) and, via **Grove**, topology-aware hierarchical gang scheduling for disaggregated inference (prefill/decode split, agentic pipelines) — the emerging shape of serving where one "workload" is several interdependent deployments with startup ordering.

Read a KAI-style manifest (field names are **version-dependent — verify against your tag**):

```yaml
# Queue: hierarchical fair-share
apiVersion: scheduling.run.ai/v2      # group has shifted across releases — check CRDs
kind: Queue
metadata:
  name: team-llm
spec:
  parentQueue: org-research           # tree: org-research > team-llm
  resources:
    gpu: { quota: 16, limit: 24, overQuotaWeight: 2 }   # deserved + borrow ceiling
---
# A fractional-GPU workload
apiVersion: v1
kind: Pod
metadata:
  annotations:
    gpu-fraction: "0.5"               # half a physical GPU
    kai.scheduler/queue: team-llm
spec:
  schedulerName: kai-scheduler
  # ... no whole nvidia.com/gpu request; the fraction annotation drives it
```

Gang is automatic: KAI's **PodGrouper** infers a PodGroup from the owning workload (Job, PyTorchJob, Deployment-set) so you rarely author `PodGroup` by hand. Contrast Volcano, where you (or the operator) declare it explicitly.

### The composition insight

The interview trap is treating this as a bake-off with one winner. The strong answer is layering:

- **Kueue for admission + quota + cohort borrowing + showback**, delegating placement/gang to the **coscheduling** plugin or to Volcano. Kueue's own docs support running with a gang-capable scheduler underneath; Kueue counts the quota and decides *when* a workload is admitted, the underlying scheduler decides *where* and enforces all-or-nothing.
- This gives you Kueue's clean multi-tenant quota model and FinOps-friendly Workload accounting **and** hard gang semantics, without forcing every tenant onto Volcano's batch worldview.
- You generally do **not** stack Kueue on top of KAI — KAI is itself a full quota+fairness+placement stack (the Run:ai model), so it substitutes for Kueue rather than layering under it.

Decision heuristic in one breath: **cloud-native multi-tenant quota → Kueue; HPC/MPI/Spark with hard gang and DRF → Volcano (optionally under Kueue); fractional GPUs, fragmentation pain, or Run:ai-style hierarchical GPU governance → KAI.**

## Worked example

**Profile.** A neocloud tenant platform: 40+ clusters, three tenant classes.
- *Class A — research training:* multi-node PyTorch/MPI, 8–64 GPU gangs, bursty, needs hard all-or-nothing and fair sharing across teams on the dimension that binds them.
- *Class B — interactive/dev:* hundreds of notebooks and small inference pods, each currently pinning a whole H100 at <10% utilization.
- *Class C — cost governance:* finance wants per-team showback and enforceable quota with controlled borrowing.

**Reasoning.**
- Class C is unambiguously **Kueue**: Workload-level accounting is the cleanest showback source, and cohort borrowing gives finance the "guaranteed quota + burst into idle" model they want. This also anchors the module deliverable.
- Class A needs true gang + fairness. Kueue alone delegates gang; so either **Kueue + coscheduling/Volcano** (keep the quota story, add gang) or **Volcano** standalone with DRF. Given Class C already mandates Kueue, the composed **Kueue admission → Volcano/coscheduling gang** stack wins: one quota story, hard gang underneath.
- Class B is the money. Whole-GPU-per-notebook is the biggest waste line item. Only **KAI** expresses fractional sharing. But KAI is a full stack and volatile — so you carve Class B onto its own node pool / cluster running KAI, rather than trying to unify all three under one scheduler.

**Defensible outcome:** *Kueue (with a gang scheduler beneath) as the fleet-wide quota/showback plane for training and governed workloads; a KAI-managed fractional-GPU pool for interactive/inference to reclaim stranded-GPU spend.* You did not pick one scheduler — you matched each tenancy shape to the primitive that expresses it, and you can put a dollar figure on the KAI pool (reclaimed idle-GPU hours × neocloud GPU rate).

## Practice

Light, reading-and-decision only. Feeds the deliverable's scheduler-selection artifact.

1. **Read two manifests.** Pull one real Volcano `PodGroup` + `Queue` (Volcano docs) and one KAI `Queue` + fractional-GPU pod (KAI-Scheduler repo `docs/`). For each, annotate: where is gang expressed? where is fairness expressed? is the GPU whole or fractional?
2. **Write a one-page decision matrix** and commit it to the deliverable. Axes (rows) at minimum:
   - Tenancy model (single-team batch vs multi-tenant quota vs hierarchical org/team/project)
   - HPC/MPI vs cloud-native lineage
   - Fractional-GPU need (yes/no)
   - Topology support (see 06.6)
   - Gang strength (native-strong / native / delegated)
   - Operational weight & maturity (flag KAI as volatile)
   Columns: Kueue / Volcano / KAI. Each cell: a one-line verdict, not a checkmark.
3. **State a composition** at the bottom: one concrete "Kueue + X" stack for your fleet and one sentence on why not KAI-under-Kueue.

**Acceptance:** a committed `scheduler-decision-matrix.md` in the deliverable directory that (a) fills every cell with a verdict, (b) names at least one workload where Volcano beats Kueue, (c) states the fractional-GPU FinOps case for KAI with the reclaimed-spend logic, and (d) describes one valid composition. If a reader can pick a scheduler for a new tenant from your matrix without asking you a question, it passes.

## Self-check

**Q1. Name one workload where Volcano beats Kueue, and why.**
**Answer:** A multi-node MPI all-reduce training job (or Spark/Ray batch job) with an 8–64-GPU gang. Volcano's `PodGroup` (`minMember`/`minResources`) enforces all-or-nothing placement natively and its DRF fairness equalizes tenants along their dominant resource, whereas Kueue delegates gang to an underlying scheduler and models fairness as quota/borrowing rather than DRF. For HPC-lineage jobs that need hard gang *and* dominant-resource fairness in one scheduler, Volcano is the better single tool.

**Q2. What does KAI's consolidation/bin-packing do that Kueue doesn't?**
**Answer:** KAI actively *defragments* by moving already-running pods to compact scattered free GPUs into a contiguous block, so a pending multi-GPU job that couldn't fit the fragments can now be placed. Kueue only admits-or-waits — it never reshuffles running workloads. On 8-GPU nodes this is the difference between stranded 3+3 GPU holes that no gang can use and reclaiming them into a runnable 6-GPU block. (KAI also uniquely does fractional-GPU sharing, which Kueue has no concept of.)

**Q3. Can Kueue and a gang scheduler compose — how?**
**Answer:** Yes. Kueue owns *admission and quota* (when a Workload is admitted, cohort borrowing, showback) while the underlying scheduler owns *placement and gang* (where pods land, all-or-nothing enforcement). You run Kueue with the coscheduling plugin or Volcano beneath it: Kueue counts quota and admits the Workload, the gang scheduler enforces minMember placement. You keep Kueue's multi-tenant quota model and get hard gang semantics. You would *not* stack Kueue on KAI, because KAI is itself a full quota+fairness+placement stack and substitutes for Kueue.

## Resources

1. **Volcano documentation** — https://volcano.sh/en/docs/ — canonical reference for `PodGroup`, `Queue`, DRF, and plugin configuration. Read the scheduling and plugin sections.
2. **NVIDIA — "Practical Tips for Preventing GPU Fragmentation for Volcano Scheduler"** — https://developer.nvidia.com/blog/practical-tips-for-preventing-gpu-fragmentation-for-volcano-scheduler/ — the anti-fragmentation binpack configuration you need before running Volcano on multi-GPU nodes; also the clearest statement of the FinOps stakes.
3. **NVIDIA KAI-Scheduler** — repo https://github.com/NVIDIA/KAI-Scheduler (**note:** development has moved to https://github.com/kai-scheduler/KAI-Scheduler as a CNCF Sandbox project — check both) and the open-source announcement https://developer.nvidia.com/blog/nvidia-open-sources-runai-scheduler-to-foster-community-collaboration/. **Flag:** young and volatile — verify CRD groups/field names against the exact tag you deploy.
