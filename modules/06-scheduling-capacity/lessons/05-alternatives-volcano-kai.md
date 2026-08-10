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
sources: 9
---

# 06.5 · Scheduler alternatives — Volcano, NVIDIA KAI, and when to pick which

> **Concept.** Kueue, Volcano, and NVIDIA KAI solve overlapping but differently-shaped problems; pick by tenancy model and workload lineage, and know they can compose rather than compete.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

Lessons 3–4 built a complete mental model of Kueue: quota-first admission, cohorts, borrowing, and two flavors of fairness (`AdmissionFairSharing` within a queue, cohort `fairSharing` across queues). That model assumed one philosophy — cloud-native, quota-owns-fairness, gang-delegated-elsewhere. This lesson pressure-tests that assumption by putting two structurally different schedulers next to it: **Volcano**, which owns gang and fairness natively instead of delegating them, and **NVIDIA KAI**, which adds a capability Kueue has no concept of at all — sub-GPU sharing. What this unlocks: the ability to answer "why not just use Kueue for everything" with a real architectural argument instead of a preference, and to design a fleet where three schedulers coexist by tenancy shape rather than argue over one.

## Why this matters

In a GPU-heavy interview loop at a CoreWeave / NVIDIA / neocloud shop, "we use Kueue" is a starting position, not an answer. The follow-up is always: *why not Volcano? why not KAI? what happens when a tenant needs fractional GPUs, or an MPI job needs true gang semantics Kueue doesn't own?* CoreWeave's own Principal/Staff Cluster Orchestration JD asks for "a technical authority on scheduling, quota enforcement, fairness, pre-emption, and multi-tenant GPU isolation" across Kubernetes, Slurm, SUNK, *and* Kueue — the plural is the point. NVIDIA open-sourcing the Run:ai scheduler as KAI, and Volcano running as a CNCF *incubating* project with 300,000+ pods scheduled per day in production (Altoros' account of the Volcano deployment; see Real-world use cases), both signal that "which scheduler" is a live, staffed decision at these companies, not a solved problem.

The failure mode this lesson prevents is picking a scheduler by familiarity and discovering at scale that it structurally cannot express what a tenant needs — you cannot retrofit DRF fairness or fractional-GPU sharing onto a scheduler with no concept of either. The cost angle is sharper: the wrong scheduler leaves GPUs fragmented (stranded 3-GPU holes on 8-GPU nodes no job can use) or idle behind a queue that can't share sub-device. On a fleet of 40+ clusters at neocloud GPU prices, single-digit-percent fragmentation or a fleet of whole-GPU notebooks idling at single-digit utilization is a seven-figure line item. Your differentiator is being the person who defends a scheduler choice in FinOps terms, not feature terms.

## What's new here (calibration)

Everything through lesson 4 assumed one scheduling philosophy: quota-first, cloud-native, gang delegated. You already know Kueue's `Workload`/`ClusterQueue`/cohort model cold — this lesson does not re-teach it. New here:

- **Volcano** — a batch scheduler with HPC/MPI lineage (descended from kube-batch, now CNCF-incubating). Gang and **Dominant Resource Fairness (DRF)** are native and first-class, not delegated to a plugin underneath an admission layer. As of **v1.15**, Volcano also does **gang-granularity preemption** and **DRA queue quota** — genuinely new capability beyond what an older pass at this material would have covered.
- **NVIDIA KAI-Scheduler** — the scheduling engine extracted from Run:ai, now a **CNCF Sandbox project**. Its distinguishing feature is **fractional GPU sharing** (a capability neither Kueue nor Volcano has), plus active **consolidation** and, via **Grove**, hierarchical topology-aware gang scheduling purpose-built for **disaggregated LLM inference** (separate, ordered prefill/decode pod cliques).
- **Composition as the real skill** — the dissolving insight that "Kueue vs Volcano vs KAI" is usually the wrong frame; the right frame is which primitive (admission/quota, gang/fairness, fractional-sharing) each tenancy shape needs, and how they layer.
- **DRF's theoretical lineage** — Volcano's and Kueue's fairness math both descend from the same 2011 paper (Ghodsi et al.); tying that thread together is new depth this lesson adds on top of L4's fair-sharing mechanics.

## Core concepts

### The three schedulers, by primitive

| | **Kueue** | **Volcano** | **NVIDIA KAI** |
|---|---|---|---|
| Lineage | SIG-driven, cloud-native | kube-batch / HPC / MPI, CNCF incubating | Run:ai commercial engine, CNCF Sandbox |
| Core object | `Workload` + `ClusterQueue`/`LocalQueue` | `PodGroup` + `Queue` | `PodGroup` (auto via PodGrouper) + `Queue` |
| Fairness model | Cohort quota + borrowing + fair sharing (L4) | **DRF** (dominant resource fairness), native | **Hierarchical fair-share** (tree of queues) |
| Gang | **Delegated** (to scheduler/coscheduling) | **Native, strong**; v1.15+ gang-granularity preemption | Native (incl. hierarchical/topology-aware via Grove) |
| Fractional GPU | No | No (whole-device) | **Yes** (annotation-based sharing) |
| Bin-pack / consolidation | No (relies on kube-scheduler) | Plugin-level bin-packing; NVIDIA anti-fragmentation guide | **Yes — active consolidation** (evicts+reschedules to defrag) |
| Topology-aware | **Yes (TAS)** — see 06.6 | Via plugins / NUMA + network topology | Yes (topology-aware hierarchical gang, Grove) |
| Newest capability (this pass) | `AdmissionFairSharing` (v0.15+, L3/L4) | Gang-granularity preemption + DRA queue quota (v1.15) | Grove integration for disaggregated-inference gang placement |
| Operational weight | Light (admission controller) | Heavier (full batch stack) | Medium; still young — expect API churn |
| Best when | Multi-tenant quota + borrowing, cloud-native | HPC/MPI/Spark, hard gang, DRF fairness | GPU sharing, fragmentation-sensitive, disaggregated inference |

### Volcano — what it is really for

Volcano is what you pick when the *workload* is a batch/HPC job and the scheduling contract is "all-or-nothing, and share fairly across multiple resource dimensions." Its `PodGroup` carries `minMember` / `minResources` — the scheduler will not bind *any* pod of the group until the minimum can be satisfied, the textbook cure for the partial-gang deadlock from lesson 2 (default scheduler binds 6 of 8 workers, deadlocks, holds GPUs hostage). DRF is the differentiator over Kueue's quota model: instead of "you get N GPUs," DRF equalizes each tenant's *dominant* resource share, so a GPU-bound tenant and a CPU/memory-bound tenant get fair treatment along the axis that actually constrains each of them. DRF is not a Volcano invention — it is the formal algorithm from Ghodsi et al.'s 2011 NSDI paper, "Dominant Resource Fairness: Fair Allocation of Multiple Resource Types" (originally built for Mesos), and it is the same theoretical spine underneath Kueue's own cohort `fairSharing` from lesson 4 — Kueue computes a dominant-resource share too, it just layers that on top of a quota model instead of making DRF the primary allocator. Seeing both schedulers as two different UIs on the same fairness math is exactly the kind of connective answer that separates a staff-level explanation from a feature-list recital.

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

Pods opt in with `schedulerName: volcano` and the `scheduling.k8s.io/group-name` annotation. The `Queue` object holds `weight` (DRF share) and optional `capability` caps. Note the whole-device GPU request — Volcano schedules GPUs as indivisible units; it has no fractional-GPU concept.

**Fragmentation caveat.** Volcano's default bin-packing is not fragmentation-aware for GPUs out of the box. NVIDIA published a practical guide on configuring the `binpack` plugin and node-ordering so multi-GPU gangs don't strand partial nodes — required reading if you run Volcano on 8-GPU boxes (see References).

**What's new in v1.15 — gang-granularity preemption and DRA queue quota.** Before v1.15, Volcano's preemption logic could select victims **pod-by-pod**, which meant a preemption event could tear a single pod out of an otherwise-healthy gang, corrupting a running distributed job for no gain (you free one GPU but the rest of that victim's gang is now useless too, since it also can't proceed without its missing rank). v1.15 makes eviction **gang-aware on both sides**: the incoming preemptor is placed as a whole gang, and victim candidates are organized and evaluated at job/gang granularity — preferring to evict a job's *surplus* replicas over randomly puncturing multiple training jobs one pod at a time. This directly closes a correctness gap, not just an efficiency one: pod-granularity preemption under Volcano could previously produce the exact "3-of-4 admitted, N idle" deadlock shape lesson 1 taught you to fear — just via eviction instead of admission. v1.15 also brings **DRA (Dynamic Resource Allocation) resources into the same queue quota model** as CPU/memory/extended resources — previously a DRA `ResourceClaim` wasn't accounted against a queue's `capability`/`deserved`/`guarantee`, so a queue could not control DRA GPU-slice usage the way it already controlled ordinary GPU counts; this closes that gap and keeps heterogeneous resource types (including the newer DRA-managed GPU slices) inside one fairness accounting model instead of a bolt-on.

### NVIDIA KAI — what it does that the others don't

Two capabilities are genuinely distinct:

1. **Fractional GPU sharing.** A pod requests a *fraction* of a GPU via annotation (e.g. `gpu-fraction: "0.5"`) or a memory slice, and KAI time-/memory-shares the physical device across pods. This is not MIG (hardware partitioning, module 04) — it's a software-level share with soft isolation, aimed at notebooks, inference, and dev workloads that waste a whole A100/H100 each. For a FinOps story this is the highest-leverage feature in the lesson: it directly attacks the "one Jupyter notebook pinning a $30k GPU at 4% utilization" waste that neither Kueue nor Volcano (both whole-device schedulers) can touch at all.

2. **Consolidation (active bin-packing / defrag).** When a pending job can't fit because free GPUs are scattered across nodes, KAI will *move running pods* to compact the free space into a contiguous block, then place the pending job. Kueue does not do this — Kueue admits or waits, but never reshuffles already-running workloads to defragment. This is the difference between "3 GPUs free here, 3 free there, your 6-GPU job waits forever" and "KAI drains the fragments and your job runs" (the L7 fragmentation problem, attacked at the scheduler level instead of the reporting level).

KAI also brings **hierarchical fair-share** (a tree of queues with weights, so an org → team → project hierarchy shares fairly at each level) and, since being onboarded as a **CNCF Sandbox project**, integrates directly with **Grove** — NVIDIA's open-source Kubernetes API for orchestrating disaggregated, multinode inference. Grove introduces `PodClique`, `PodCliqueScalingGroup`, and `PodCliqueSet` CRDs that let you declare a *heterogeneous, ordered* gang: e.g. one prefill clique and one decode clique that must come up in a specific order, scale semi-independently, and still be placed with topology awareness (KV-cache transfer between prefill and decode is bandwidth-sensitive, so the two cliques benefit from being co-located just like a training gang does). KAI is the scheduling engine underneath Grove's placement decisions — this is genuinely new since disaggregated serving became a mainstream inference architecture, and it means KAI is no longer just "the fractional-GPU scheduler"; it is becoming the scheduling substrate for the emerging split-inference pattern.

Read a KAI-style manifest (field names are **version-dependent — verify against your tag**; the project's repo location and CRD group have shifted as it moved to CNCF Sandbox governance):

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

Gang is automatic: KAI's **PodGrouper** infers a PodGroup from the owning workload (Job, PyTorchJob, Deployment-set, or a Grove `PodCliqueSet`) so you rarely author `PodGroup` by hand. Contrast Volcano, where you (or the operator) declare it explicitly.

### The composition insight

The interview trap is treating this as a bake-off with one winner. The strong answer is layering:

- **Kueue for admission + quota + cohort borrowing + showback**, delegating placement/gang to the **coscheduling** plugin or to Volcano. Kueue's own docs support running with a gang-capable scheduler underneath; Kueue counts the quota and decides *when* a workload is admitted, the underlying scheduler decides *where* and enforces all-or-nothing.
- This gives you Kueue's clean multi-tenant quota model and FinOps-friendly Workload accounting **and** hard, now gang-granularity-aware (Volcano v1.15) preemption semantics, without forcing every tenant onto Volcano's batch worldview.
- You generally do **not** stack Kueue on top of KAI — KAI is itself a full quota+fairness+placement stack (the Run:ai model), so it substitutes for Kueue rather than layering under it. If your disaggregated-inference tenants need Grove-style hierarchical gang placement, that pool runs KAI directly.

Decision heuristic in one breath: **cloud-native multi-tenant quota → Kueue; HPC/MPI/Spark with hard gang and DRF → Volcano (optionally under Kueue); fractional GPUs, fragmentation pain, or disaggregated-inference topology gang placement → KAI.**

## Perspectives

**Developer / researcher.** An HPC-background researcher submitting an MPI job to Volcano gets a more familiar mental model (explicit `PodGroup`, explicit `Queue`, DRF) than a cloud-native engineer submitting to Kueue (implicit `Workload`, quota abstraction). The choice of scheduler shapes who feels at home on the platform — a genuine people/org consideration, not just a technical one, and one platform teams underweight.

**Operator.** Operational weight is asymmetric across the three. Kueue is "just" an admission webhook + controller layered on the default scheduler. Volcano is a full batch stack — its own scheduler binary, its own plugins, its own queue CRDs — that in effect *replaces* parts of the default scheduling path. KAI is younger still, and CNCF Sandbox governance plus repo/CRD-group churn (the `run.ai` API group predates the community handoff) means an operator adopting it today needs an explicit plan for API-surface drift, not a "set and forget" deployment.

**Hardware.** KAI's fractional-GPU sharing is a *software* time/memory-share with soft isolation — contrast MIG (module 04), which is hardware partitioning with hard isolation and fixed profile geometry. The choice is a hardware-vs-software tradeoff: MIG gives real isolation at coarse, fixed granularity; KAI gives fine, flexible granularity at the cost of noisy-neighbor risk between co-scheduled fractional tenants.

**Economics.** The FinOps case for KAI is the sharpest number in the whole lesson — a whole H100 pinned by a notebook at single-digit utilization is the single largest, most visible waste line item most platform teams can point to; fractional sharing directly reclaims it, while Kueue and Volcano (both whole-device schedulers) structurally cannot touch that waste at all, no matter how well their quota or fairness logic is tuned.

## Real-world use cases

- **Volcano at scale — Altoros, "Scheduling 300,000 Kubernetes Pods in Production Daily"** — https://www.altoros.com/blog/volcano-scheduling-300000-pods-in-production-daily/ (fetch blocked by this session's egress proxy; canonical URL search-confirmed, title and topic match). What it shows: Volcano running 300,000 pods/day in production, cited alongside adoption by 50+ enterprises — real evidence for the scale claims the comparison table above makes, not just an assertion.
- **NVIDIA — open-sourcing the Run:ai scheduler as KAI** — https://developer.nvidia.com/blog/nvidia-open-sources-runai-scheduler-to-foster-community-collaboration/ (fetch blocked by egress proxy; canonical URL, search-confirmed). What it shows: NVIDIA's own account of donating the scheduler and its subsequent path into CNCF Sandbox governance — the origin story for why KAI exists as an open project at all.
- **NVIDIA — "Deploying Disaggregated LLM Inference Workloads on Kubernetes"** — https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/ (fetch blocked by egress proxy; content independently confirmed via WebSearch snippets in this session). What it shows: Grove's `PodClique`/`PodCliqueScalingGroup`/`PodCliqueSet` CRDs and KAI's role as the scheduling engine satisfying gang, hierarchical-gang, and topology-aware placement constraints underneath them — the concrete mechanism behind "KAI is becoming the substrate for disaggregated inference" above.
- **AWS — "Maximizing GPU Utilization Using NVIDIA Run:ai in Amazon EKS"** — https://aws.amazon.com/blogs/containers/maximizing-gpu-utilization-using-nvidia-runai-in-amazon-eks/ (fetch blocked by egress proxy; canonical URL search-confirmed). What it shows: a hyperscaler's own reference architecture for Run:ai/KAI-style fractional GPU sharing on managed Kubernetes — a fourth use case for the fractional-sharing FinOps story, on infrastructure the learner's target companies also run workloads against.

## Worked example

**Profile.** A neocloud tenant platform: 40+ clusters, four tenant classes.

- *Class A — research training:* multi-node PyTorch/MPI, 8–64 GPU gangs, bursty, needs hard all-or-nothing and fair sharing across teams on the dimension that binds them.
- *Class B — interactive/dev:* hundreds of notebooks and small inference pods, each currently pinning a whole H100 at <10% utilization.
- *Class C — cost governance:* finance wants per-team showback and enforceable quota with controlled borrowing.
- *Class D — disaggregated LLM serving (new):* separate prefill and decode pod cliques with strict co-location and startup-ordering needs, and per-clique autoscaling.

**Reasoning.**
- Class C is unambiguously **Kueue**: Workload-level accounting is the cleanest showback source, and cohort borrowing gives finance the "guaranteed quota + burst into idle" model they want. This also anchors the module deliverable.
- Class A needs true gang + fairness. Kueue alone delegates gang; so either **Kueue + coscheduling/Volcano** (keep the quota story, add gang) or **Volcano** standalone with DRF. Given Class C already mandates Kueue, the composed **Kueue admission → Volcano/coscheduling gang** stack wins: one quota story, hard gang underneath, and as of Volcano v1.15 that gang enforcement extends into preemption too (a Class A job being reclaimed for cohort borrowing is evicted gang-atomically, not pod-by-pod).
- Class B is the money. Whole-GPU-per-notebook is the biggest waste line item. Only **KAI** expresses fractional sharing. But KAI is a full stack and still young — so you carve Class B onto its own node pool / cluster running KAI, rather than trying to unify all three classes under one scheduler.
- Class D is new and structurally different from Class A: it is not a homogeneous gang, it's a *heterogeneous, ordered* set of cliques (prefill must be reachable before decode routes to it) with its own topology sensitivity (KV-cache transfer bandwidth). Neither Kueue+coscheduling nor Volcano's `PodGroup` model expresses "two different pod shapes, one must precede the other, both should be topology-close" — that is exactly the gap **Grove** fills, with **KAI** as its scheduling engine. Class D therefore joins Class B's KAI pool rather than getting its own fourth scheduler.

**Defensible outcome:** *Kueue (with a gang scheduler beneath) as the fleet-wide quota/showback plane for training and governed workloads; a KAI-managed pool — running Grove on top for the disaggregated-serving tenants — for interactive/inference workloads that need fractional sharing or heterogeneous gang placement.* You did not pick one scheduler — you matched each tenancy shape to the primitive that expresses it, and you can put a dollar figure on the KAI pool (reclaimed idle-GPU hours × neocloud GPU rate) and a correctness argument on the Grove/Class-D placement (why coscheduling alone cannot express an ordered, heterogeneous gang).

## Practice

Light, reading-and-decision only. Feeds the deliverable's scheduler-selection artifact — see [Kueue setup + per-queue showback](../practice/kueue-showback/README.md).

1. **Read two manifests.** Pull one real Volcano `PodGroup` + `Queue` (Volcano docs) and one KAI `Queue` + fractional-GPU pod (KAI-Scheduler repo `docs/`). For each, annotate: where is gang expressed? where is fairness expressed? is the GPU whole or fractional?
2. **Write a one-page decision matrix** and commit it to the deliverable. Axes (rows) at minimum:
   - Tenancy model (single-team batch vs multi-tenant quota vs hierarchical org/team/project)
   - HPC/MPI vs cloud-native lineage
   - Fractional-GPU need (yes/no)
   - Heterogeneous/ordered gang need — e.g. disaggregated serving (yes/no)
   - Topology support (see 06.6)
   - Gang strength and preemption granularity (native-strong-gang-preemption / native / delegated)
   - Operational weight & maturity (flag KAI as still young)
   Columns: Kueue / Volcano / KAI. Each cell: a one-line verdict, not a checkmark.
3. **State a composition** at the bottom: one concrete "Kueue + X" stack for your fleet, and one sentence on why not KAI-under-Kueue.

**Acceptance:** a committed `scheduler-decision-matrix.md` in the deliverable directory that (a) fills every cell with a verdict, (b) names at least one workload where Volcano beats Kueue, (c) states the fractional-GPU FinOps case for KAI with the reclaimed-spend logic, (d) names a workload shape (heterogeneous/ordered gang) that needs KAI+Grove specifically, and (e) describes one valid composition. If a reader can pick a scheduler for a new tenant from your matrix without asking you a question, it passes.

## Common pitfalls

- **Picking Volcano purely because "it does gang scheduling."** Teams sometimes adopt the full batch-stack replacement for gang scheduling alone, when Kueue+coscheduling would have sufficed with far less operational surface. Weigh the operational cost, not just the feature checkbox.
- **Treating KAI's fractional GPU sharing as equivalent to MIG.** It is a *software* soft-isolation mechanism (time/memory-slicing at the scheduler level), not hardware partitioning; conflating the two leads to wrong isolation/security assumptions for multi-tenant workloads — a noisy-neighbor incident waiting to happen.
- **Assuming scheduler choice is permanent and fleet-wide.** A common mistake is trying to force one scheduler onto every tenant instead of segmenting by tenancy shape, as the worked example demonstrates. Different classes of work genuinely need different primitives.
- **Assuming pod-granularity preemption is "good enough" for gangs.** Before Volcano v1.15's gang-granularity preemption, a naive preemption policy could tear a single pod out of a healthy gang and strand the rest — the eviction-side mirror of the lesson-1 deadlock. Verify your Volcano version actually has gang-aware preemption before relying on it.
- **Forgetting DRA resources need their own quota accounting.** Pre-v1.15 Volcano queues could not see or bound DRA `ResourceClaim` usage the way they bounded ordinary `nvidia.com/gpu` requests — a queue "at capacity" by classic accounting could still be consuming unbounded DRA-managed GPU slices underneath it.

## Self-check

- Name one workload where Volcano beats Kueue, and why. **Answer:** A multi-node MPI all-reduce training job (or Spark/Ray batch job) with an 8–64-GPU gang. Volcano's `PodGroup` (`minMember`/`minResources`) enforces all-or-nothing placement natively and its DRF fairness equalizes tenants along their dominant resource, whereas Kueue delegates gang to an underlying scheduler and models fairness as quota/borrowing rather than DRF. For HPC-lineage jobs that need hard gang *and* dominant-resource fairness in one scheduler, Volcano is the better single tool.
- What does KAI's consolidation/bin-packing do that Kueue doesn't? **Answer:** KAI actively *defragments* by moving already-running pods to compact scattered free GPUs into a contiguous block, so a pending multi-GPU job that couldn't fit the fragments can now be placed. Kueue only admits-or-waits — it never reshuffles running workloads. On 8-GPU nodes this is the difference between stranded 3+3 GPU holes that no gang can use and reclaiming them into a runnable 6-GPU block. (KAI also uniquely does fractional-GPU sharing, which Kueue has no concept of.)
- Can Kueue and a gang scheduler compose — how? **Answer:** Yes. Kueue owns *admission and quota* (when a Workload is admitted, cohort borrowing, showback) while the underlying scheduler owns *placement and gang* (where pods land, all-or-nothing enforcement). You run Kueue with the coscheduling plugin or Volcano beneath it: Kueue counts quota and admits the Workload, the gang scheduler enforces minMember placement. You keep Kueue's multi-tenant quota model and get hard gang semantics. You would *not* stack Kueue on KAI, because KAI is itself a full quota+fairness+placement stack and substitutes for Kueue.
- What does Volcano v1.15's move to gang-granularity preemption fix that pod-granularity preemption couldn't? **Answer:** Pod-granularity preemption could evict a single pod out of a healthy, running gang to reclaim one GPU — but the rest of that gang still can't make progress without its missing rank, so you've stranded the *remaining* pods of the victim job while only "freeing" resources that are effectively still wasted. Gang-granularity preemption places the incoming preemptor as a whole gang and selects victims at job/gang granularity (preferring surplus replicas over random per-pod eviction), so a preemption event either evicts a coherent unit or doesn't happen — it doesn't leave a half-alive gang behind.
- Why does KAI's integration with Grove matter for disaggregated LLM inference specifically, and why can't Kueue+coscheduling express the same placement need? **Answer:** Disaggregated inference splits one logical service into heterogeneous, ordered pod cliques (prefill must be reachable before decode routes to it) with topology-sensitive KV-cache transfer between them — this is not one homogeneous gang, it's several interdependent gangs with startup ordering and independent scaling. Kueue+coscheduling's model is "one PodSet, admit or don't"; it has no concept of multiple distinct pod shapes within one workload that must come up in a specific order. Grove's `PodClique`/`PodCliqueScalingGroup`/`PodCliqueSet` CRDs express exactly that structure, and KAI is the scheduling engine that satisfies the resulting hierarchical, topology-aware gang placement underneath it.

## Connections & what's next

This lesson sits on top of everything Kueue-specific from lessons 3–4 (quota, cohorts, borrowing, both fairness layers) and reaches back to lesson 2's gang deadlock (Volcano's `PodGroup` is a second, native way to solve the same problem coscheduling solves as a plugin) and forward to lesson 7's fragmentation math (KAI's active consolidation is a scheduler-level attack on the same waste lesson 7 teaches you to measure and report). The DRF thread — Ghodsi et al.'s NSDI 2011 paper — is the shared theoretical spine under both Volcano's native fairness and Kueue's cohort `fairSharing` from lesson 4; you now have both implementations to compare against the one theory. Next: lesson 6 asks a question every scheduler surveyed here shares — admitting the right *count* of co-located GPUs is necessary but not sufficient if you don't also control *where* they land relative to the NVLink/network topology. Topology-aware placement is the layer that makes all three of these schedulers' gang guarantees actually fast, not just atomic.

## References & further reading

**Primary sources**
- Ghodsi, A. et al., "Dominant Resource Fairness: Fair Allocation of Multiple Resource Types" (NSDI 2011) — https://www.usenix.org/conference/nsdi11/dominant-resource-fairness-fair-allocation-multiple-resource-types (fetch blocked by egress proxy this session; canonical USENIX conference URL, search-confirmed) — the formal paper behind both Volcano's native DRF and Kueue's cohort fair-sharing math; read for the theory underneath both schedulers' fairness knobs.
- Volcano documentation — https://volcano.sh/en/docs/ — canonical reference for `PodGroup`, `Queue`, DRF, and plugin configuration; read the scheduling and plugin sections.
- Volcano v1.15.0 release notes — https://volcano.sh/blog/volcano-1.15.0-release/ (fetch blocked by egress proxy this session; canonical URL, search-confirmed and its content independently verified via WebSearch) — read for the gang-granularity preemption and DRA queue quota mechanics.
- NVIDIA KAI-Scheduler — repo https://github.com/NVIDIA/KAI-Scheduler (development has moved toward CNCF Sandbox governance — check the current canonical repo at deploy time) and the open-source announcement https://developer.nvidia.com/blog/nvidia-open-sources-runai-scheduler-to-foster-community-collaboration/ (fetch blocked; canonical URL). Flag: still young — verify CRD groups/field names against the exact tag you deploy.
- NVIDIA — "Deploying Disaggregated LLM Inference Workloads on Kubernetes" — https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/ (fetch blocked; content confirmed via WebSearch snippets) — read for Grove's `PodClique`/`PodCliqueScalingGroup`/`PodCliqueSet` CRDs and KAI's role underneath them.

**Real-world engineering blogs**
- Altoros — "Scheduling 300,000 Kubernetes Pods in Production Daily" — https://www.altoros.com/blog/volcano-scheduling-300000-pods-in-production-daily/ (fetch blocked; canonical URL search-confirmed) — what it shows: Volcano's real production scale and enterprise adoption.
- NVIDIA — "Practical Tips for Preventing GPU Fragmentation for Volcano Scheduler" — https://developer.nvidia.com/blog/practical-tips-for-preventing-gpu-fragmentation-for-volcano-scheduler/ — what it shows: the anti-fragmentation binpack configuration you need before running Volcano on multi-GPU nodes.
- AWS — "Maximizing GPU Utilization Using NVIDIA Run:ai in Amazon EKS" — https://aws.amazon.com/blogs/containers/maximizing-gpu-utilization-using-nvidia-runai-in-amazon-eks/ (fetch blocked; canonical URL search-confirmed) — what it shows: a hyperscaler's own reference architecture for fractional GPU sharing.

**Deeper dives**
- Kueue documentation on running with a gang-capable scheduler underneath (see lesson 6's references for the Kueue TAS/placement docs) — for how the composition described above is actually wired.
- The Grove project (linked from the NVIDIA disaggregated-inference blog above) — for the full `PodCliqueSet` API if you want to go past this lesson's summary.
