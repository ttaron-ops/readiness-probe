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
sources: 7
---

# 06.6 · Topology-aware placement — packing a gang into one NVLink domain

> **Concept.** A gang that spreads across NVLink domains or racks pays a 2–5× collective-communication tax; Kueue Topology-Aware Scheduling packs it into one domain via node labels so NCCL collectives never cross the slow link.
>
> Module: [🗓️ 06 — Scheduling, queueing and capacity](../README.md) · Deliverable: [Kueue setup + per-queue showback](../practice/kueue-showback/README.md)

## Where this fits

Lesson 5 surveyed three schedulers — Kueue, Volcano, KAI — and every one of them answers the question "does this gang get admitted, all at once?" None of them, on their own quota/admission logic, answers "*where*, relative to the network fabric, does this gang land?" That is the gap this lesson closes. You already have gang scheduling (lesson 2) guaranteeing atomicity and Kueue's quota/cohort model (lessons 3–4) guaranteeing fairness — this lesson adds the third axis, placement, and shows why a gang that is atomically admitted but topologically blind can still be a slow, expensive mistake. What it unlocks: the ability to reason about *why* two runs with identical GPU counts can differ 2–5× in wall-clock cost, and to fix it with a scheduling constraint instead of more hardware.

## Why this matters

Anthropic's own Sr Staff+ Kubernetes Platform JD names "gang scheduling; topology-aware placement" side by side, not as two separate skills but as one sentence — because gang without topology is an incomplete answer in that interview. The cost is not subtle and it is not a tail case: synchronous data-parallel training is gated on the all-reduce at every step, so if the collective's slowest link is 5–10× slower because part of the ring crosses a rack or spine boundary, the *entire* gang idles waiting on it — you are billed for N GPUs and getting the throughput of a badly-bottlenecked fraction. Nothing in a quota dashboard flags this; it reads as "8/8 GPUs admitted, job running," and the only symptom is that training is mysteriously, expensively slow.

Meta's own account of this problem at extreme scale is the sharpest real number available: their MAST paper (OSDI 2024) describes scheduling LLaMA3 training across **16,000 H100 GPUs** with explicit topological constraints — "allocating in multiples of two hosts per rack and 128 hosts per data center" — and using that discipline (among other techniques) to bring the most-overloaded region's GPU demand-to-supply ratio from **2.63 down to 0.98**. That is not a toy example; it is the company that trained LLaMA3 telling you, in a peer-reviewed systems paper, that topology-aware placement is load-bearing at the scale your target companies operate at. Being the engineer who ties a slow training run back to a split gang, and fixes it with a topology constraint instead of a purchase order, is squarely the FinOps differentiator this course is building toward.

## What's new here (calibration)

You already did the hard physics in module 02b: NVLink-domain bandwidth (hundreds of GB/s intra-domain) versus the order-of-magnitude drop once you cross the PCIe host boundary, a rack, or a network spine, and how the kubelet's Topology Manager aligns CPU/NUMA/device locality on a single node. This lesson does **not** re-teach that hierarchy or single-node NUMA alignment — it assumes you can already explain *why* crossing a domain is slow. What's genuinely new:

- **Multi-node topology as a scheduling constraint**, not just a physical fact — Kueue's `Topology` object, `required`/`preferred` PodSet annotations, and how they interact (conjunctively) with gang admission.
- **The placement *algorithm* Kueue uses when a constraint doesn't pin an exact domain** — as of **Kueue v0.15**, this changed from `BestFit` to `LeastFreeCapacity` for unconstrained placements, plus a new `TASBalancedPlacement` feature gate for spreading pods evenly across selected domains. This is genuinely new depth beyond what an earlier pass at this lesson would have covered, and it connects directly to lesson 7's fragmentation math.
- **Multi-level, real-world-scale constraints** (MAST's "two hosts per rack, 128 hosts per datacenter") as a preview of how far this problem goes past Kueue's simpler single-level `required`/`preferred` model.

## Core concepts

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

Levels are **ordered coarse→fine**. Every node that participates must carry every label in the hierarchy. The lowest level is conventionally `kubernetes.io/hostname` — one node — which lets you express "pack onto a single host" for a gang that fits in one box. In production, these labels map to a specific cloud provider's real fabric topology — for example AWS EC2 UltraCluster placement groups or GCP's compact placement policies with A3/A3-Mega topology labels — so TAS is only ever as accurate as the labels the provider (or your own DCIM tooling) actually publishes.

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

- Use **`required`** when the collective genuinely cannot tolerate a cross-domain link — large synchronous data-parallel / tensor-parallel training where all-reduce dominates step time. Accept the cost: if no single rack can hold the gang right now, the workload **waits** (or never admits) rather than running slow. A job that waits an hour for a clean rack usually beats a job that runs 2× slow for a week — but frame this as a break-even, not a rule of thumb: if `required` makes a job wait `W` hours for a placement that saves `S` hours of wall-clock, it's only worth it when `W < S`. For long training runs that's almost always true; for short, bursty jobs it may not be.
- Use **`preferred`** when the workload benefits from locality but correctness/throughput degrade gracefully — data-parallel with gradient compression, embarrassingly-parallel work, inference replicas, or anything where "packed if possible, spread if not" beats waiting. Also the safer default when your topology labels or capacity are not yet trustworthy.
- Choose the **level** to match the collective's reach: a gang that fits in one node → `kubernetes.io/hostname`; an 8–16-GPU gang spanning a couple of boxes in a rack → `topology-rack`; a super-pod-scale job → `topology-block`.

### How TAS and gang interact — both must hold

This is the subtle part and a favorite interview probe. TAS and gang are **conjunctive** constraints:

1. Kueue admits a Workload only when quota is available (lessons 3–4) **and** TAS can find a single domain at the requested level with enough free capacity for the **entire** PodSet.
2. Gang (delegated to the underlying scheduler / coscheduling, or handled natively by Volcano/KAI — lesson 5) then enforces all-or-nothing placement — but now confined to the domain TAS selected.
3. With `required`, if no single domain fits the whole gang, the Workload is **not admitted** — it does not partially place and then deadlock (the lesson-2 failure mode) and it does not spread. The gang's atomicity and the topology's locality are satisfied together or not at all.

The reason you need both: gang alone gives you "all 8 pods or none" but says nothing about *where* — it will cheerfully give you an all-8 placement split 4+4 across the spine. TAS alone (without gang) could place some pods in the good domain and leave the rest pending. Together they give the only correct outcome for tightly-coupled training: **all pods, in one fast domain, or wait.**

### The unconstrained placement algorithm — BestFit vs LeastFreeCapacity (Kueue v0.15+)

There is a case the `required`/`preferred` framing above doesn't fully cover: when a PodSet has *no* explicit topology annotation at all, or when multiple domains equally satisfy the requested level, Kueue still has to choose *which* domain(s) to use. Before Kueue v0.15, the default algorithm for this "unconstrained" case was **BestFit**: it selects as many domains as needed starting from the one with the most free capacity, and optimizes the *last* domain chosen to minimize leftover free resources — in other words, it tries to pack as tightly as possible into as few domains as possible.

As of **Kueue v0.15**, the default for unconstrained placement changed to **LeastFreeCapacity**: instead of starting from the domain with the most free capacity, it iterates starting from the domain with the *least* free capacity, generally biasing toward a tighter fit that leaves the roomier domains untouched. Kueue's own KEP for TAS states the intent plainly: the change exists to "prioritize minimizing fragmentation." Read that against lesson 7 before you move on: BestFit's tight-packing bias on the *last* domain can still leave many domains lightly, unusably touched; LeastFreeCapacity instead protects the domains that still have the most slack, preserving them for a *future* large gang that needs a whole clean domain — the same "protect the whole-node hole" instinct lesson 7 teaches you to reason about, just implemented as a scheduler default rather than something you compute after the fact.

Kueue v0.15 also introduced a `TASBalancedPlacement` feature gate, which enables a third strategy: it first finds the optimal *set* of domains that fit the request, then distributes the pods as evenly as possible *across* that set — avoiding a lopsided split like 10-and-2 in favor of something like 6-and-6. This matters specifically for **all-to-all communication patterns** (as opposed to a simple ring all-reduce), where an uneven split can create one badly-loaded link even when every pod nominally "fits."

**Practical takeaway:** if you're running Kueue v0.15+ and relying on the *default* unconstrained behavior rather than an explicit `required`/`preferred` annotation, verify which algorithm your version actually runs — the fragmentation consequences of BestFit vs LeastFreeCapacity are not cosmetic, they change which domains stay whole for the next big job.

### Failure modes to name

- **Label drift / missing labels.** A node missing one Topology level label is invisible to TAS at that level — capacity silently shrinks. Validate label coverage as a cluster invariant, and re-validate it whenever a new node pool or hardware generation is onboarded — this is continuous day-2 operational hygiene, not a one-time setup step.
- **`required` starvation.** Over-aggressive `required` at a fine level on a fragmented cluster can leave a job pending indefinitely while GPUs sit idle in scattered domains. This is the topology analog of the fragmentation problem KAI's consolidation attacks (lesson 5) and lesson 7 quantifies — TAS won't defrag for you; it waits.
- **Level too coarse.** `required: topology-block` on a job that needed rack-level locality "passes" but still lets the gang spread within the block across racks — you constrained the wrong level and get the slow link anyway.
- **Trusting cloud-provided topology labels blindly.** Not every cloud SKU exposes fine-grained topology labels at all. `required` at a level the provider doesn't actually label correctly either errors out or silently degrades to no real constraint. Verify label coverage as part of onboarding any new node pool — don't assume the label hierarchy is truthful just because it's documented.

## Perspectives

**Developer.** A researcher who annotates a job `required` at rack level and then sees it sit `Pending` needs to understand this is *working as intended* — the job is correctly refusing a slow placement rather than silently accepting one. Without that framing, `required` reads as a bug report ("my job won't schedule!") instead of the cost-protection feature it actually is. Part of a platform team's job is making that distinction legible in the pending-reason message, not just in a lesson like this one.

**Operator.** Topology label hygiene is a continuous operational responsibility, not a one-time setup step. New nodes added to a fleet — especially heterogeneous ones from a different hardware generation or a different physical build — must carry every level's label, or TAS silently shrinks its view of available capacity without raising an alarm. This is a day-2-operations discipline as much as it is a scheduling concept.

**Hardware / network.** TAS levels are only useful insofar as the labels map to physical truth. The hierarchy is a *declaration*, and cloud providers back it with real fabric topology — AWS EC2 UltraCluster placement groups, GCP's compact placement policies and A3/A3-Mega topology labels are concrete examples. TAS is exactly as good as whether the underlying datacenter actually exposes an honest hierarchy; a fabricated or stale label hierarchy makes TAS confidently wrong.

**Economics.** The `required`-vs-`preferred` choice is "pay in wait time vs pay in throughput," and it's worth stating as an explicit break-even: if `required` makes a job wait `W` hours to land in a placement that saves it `S` hours of wall-clock, it is worth it exactly when `W < S`. For long, comm-bound training runs `S` is usually large and the trade is easy. For short or deadline-bound jobs, the wait itself can cost more than the throughput it buys — which is exactly why `preferred` exists as a real second option, not just a fallback for "don't trust your labels yet."

## Real-world use cases

- **Meta — "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale"** (OSDI 2024) — https://www.usenix.org/system/files/osdi24-choudhury.pdf (fetch blocked by this session's egress proxy; canonical URL, content independently confirmed via WebSearch in this session, corroborated by the paper's USENIX listing at https://www.usenix.org/conference/osdi24/presentation/choudhury). What it shows: Meta scheduling LLaMA3 training across **16,000 H100 GPUs** using explicit topological constraints — "allocating in multiples of two hosts per rack and 128 hosts per data center" — and reducing the most-overloaded region's GPU demand-to-supply ratio from **2.63 to 0.98**. This is a **dated snapshot of a specific training run described in a 2024 paper**, not a general benchmark — but it is the best "topology-aware placement at extreme scale, with real numbers" citation available, from the company that trained the model.
- **Kueue upstream — Topology Aware Scheduling KEP** — https://github.com/kubernetes-sigs/kueue/blob/main/keps/2724-topology-aware-scheduling/README.md and tracking issue https://github.com/kubernetes-sigs/kueue/issues/2724 (both fetched directly in this session). What it shows: the original design proposal and the concrete motivating complaint from the field — "Running a workload with Pods scattered across a data center results in longer runtimes, and thus costs" — plus the BestFit→LeastFreeCapacity and `TASBalancedPlacement` details cited above, confirmed directly from the KEP text.
- **Google Cloud (community) — "Kueue for AI: The Power of Atomic Admission & Topology Awareness"** — https://medium.com/google-cloud/kueue-for-ai-the-power-of-atomic-admission-topology-awareness-59c2fd1f86ed (fetch blocked by egress proxy; canonical URL search-confirmed). What it shows: a GCP engineer's own walkthrough of TAS mechanics on GKE with real node-label examples — a second, cloud-vendor-specific account complementing the Kueue docs cited below.
- **NVIDIA — "Deploying Disaggregated LLM Inference Workloads on Kubernetes"** (same source as lesson 5) — https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/ (fetch blocked by egress proxy; content confirmed via WebSearch snippets in this session). What it shows: topology-aware placement extended to the *inference* side — prefill/decode-split serving needs specific co-location too, via Grove and KAI, not just training — a good bridge showing this lesson's mechanism isn't training-only.

## Worked example

**Scenario.** An 8-GPU synchronous data-parallel H100 training job (one PyTorchJob, 8 workers, gang via coscheduling under Kueue). The `training` ClusterQueue has 64 GPUs across 4 racks of 2× 8-GPU nodes.

**Topology-blind baseline.** Without TAS, the scheduler satisfies "8 GPUs" as 4 GPUs on a node in rack A + 4 on a node in rack C. Every all-reduce ring now includes a cross-rack (spine) hop. Per 02b's bandwidth hierarchy, the spine link is ~5–10× slower than NVLink; the collective is gated on its slowest link, so per-step all-reduce time balloons and **all 8 GPUs stall** on each step. Observed: step time up ~2–5×, GPU utilization (SM-active) collapses during the comm phase, wall-clock — and therefore GPU-hour cost — up 2×+. Nothing in the quota dashboard flags it; it reads as "8/8 GPUs admitted, job running."

**With TAS `required` at rack level.** Annotate the PodSet:

```yaml
kueue.x-k8s.io/podset-required-topology: "cloud.provider.com/topology-rack"
```

Now Kueue admits the Workload only when a single rack has 8 free GPUs (its two nodes), and the gang places all 8 workers within that rack. Every all-reduce link is now intra-rack (and intra-node NVLink where possible). Step time returns to compute-bound; the job finishes in half the wall-clock. If no rack currently has 8 free, the job **waits for one to drain** — correct: waiting an hour beats running the whole job at half speed.

**Relaxing to `preferred`.** Swap `required`→`preferred`. If a clean rack exists, identical packing. If the cluster is fragmented (every rack has one busy node), TAS falls back to a coarser fit and the gang spreads — the job runs *now*, slower, instead of waiting. That is the right call for a deadline-bound but comm-tolerant job, and the wrong call for a comm-bound one. The choice *is* the FinOps decision: pay in latency (wait) or pay in throughput (spread).

**Extending to the v0.15+ placement algorithm.** Now suppose the job carries no `required`/`preferred` annotation at all — an unconstrained request for 4 GPUs — and the fleet's four racks currently show `[2, 2, 6, 8]` free GPUs. Under the pre-v0.15 default, **BestFit** starts from the domain with the *most* free capacity and tries to minimize leftover slack on the last domain chosen — in practice this tends to pack the request tightly, e.g. into the rack with exactly `2+2=4` free (a perfect fit that fully consumes two small domains, potentially fragmenting them further if the job doesn't land exactly). Under **LeastFreeCapacity** (the v0.15+ default), Kueue instead iterates starting from the domain with the *least* free capacity — biasing the placement away from the rack with 8 free and toward using up the tighter domains first, which preserves the roomy 8-free rack intact for a future large gang that needs a whole clean domain. This is the fragmentation tradeoff from lesson 7, expressed as a scheduler placement policy instead of a post-hoc calculation: LeastFreeCapacity is, in effect, Kueue defaulting toward "don't strand your biggest hole" without you having to ask for it explicitly.

## Practice — TAS co-placement demo on kind

This feeds the deliverable's topology artifact. You do not need real GPUs — fake the labels; TAS reasons over labels, not hardware. See [Kueue setup + per-queue showback](../practice/kueue-showback/README.md).

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
6. **Stretch — the placement algorithm.** If your Kueue version supports it, check (or toggle) `TASBalancedPlacement` and submit an unconstrained (no `required`/`preferred`) multi-pod request against an asymmetric free-capacity layout across the 4 workers; observe whether the result matches LeastFreeCapacity's "protect the roomiest domain" bias described above.

**Acceptance:** a committed TAS co-placement demo in the deliverable containing (a) the `Topology` + TAS `ClusterQueue` + `ResourceFlavor` manifests, (b) the node-labeling commands, (c) `kubectl get pods -o wide` output for the `required` run showing every gang pod on one rack, and (d) the `preferred` run showing spread when no rack fits, with one sentence explaining why `required` waited/packed and `preferred` spread. If a reviewer can rerun your commands on a fresh kind cluster and reproduce both outcomes, it passes.

## Common pitfalls

- **Treating `required` as a bug when the job sits `Pending`.** It's the constraint working as designed — refusing a slow placement — not a scheduler failure. Fix by draining a domain or relaxing to `preferred`, not by assuming TAS is broken.
- **Letting topology labels drift as the fleet grows.** A new node pool or hardware generation that's missing even one level's label silently shrinks TAS's view of capacity — treat full label coverage as a continuous invariant to validate, not a one-time install step.
- **Constraining the wrong level.** `required: topology-block` on a job that actually needed rack-level locality "passes" validation but still lets the gang spread within the block — you get the slow link anyway while believing you fixed it.
- **`required` starving a job on a fragmented cluster.** Aggressive `required` at a fine level with no relief valve can leave a job pending indefinitely while GPUs sit idle in scattered domains — TAS does not defragment for you (that's KAI's consolidation, lesson 5, or a manual defrag, lesson 7).
- **Assuming cloud topology labels are always accurate.** Some SKUs don't expose fine-grained topology labels at all; verify label coverage before trusting a `required` constraint on a new node pool, rather than discovering the gap when a job unexpectedly can't admit — or worse, silently degrades because a label was simply absent.

## Self-check

- Why does a topology-spread all-reduce gang waste GPU-hours? (Tie to 02b interconnect bandwidth.) **Answer:** Synchronous training does an all-reduce every step and the collective is gated on its slowest link. Per 02b's bandwidth hierarchy, a cross-rack/spine hop is ~5–10× slower than intra-NVLink-domain, so a gang split across domains runs every all-reduce at fabric speed. Because the step can't proceed until the collective completes, *all* GPUs in the gang idle waiting on the slow link — you pay for N GPUs but get the throughput of a comm-bottlenecked fraction, and wall-clock (hence GPU-hour cost) roughly doubles or worse.
- `required` vs `preferred` topology annotation — behavior difference? **Answer:** `required` is a hard constraint: the entire PodSet must fit within one domain at the named level or the Workload is **not admitted** — it waits rather than spreading. `preferred` is soft: TAS tries to pack into the named level, but if no single domain fits it falls back to a coarser level and lets the gang spread so the job still runs now. `required` trades latency (waiting) for guaranteed locality; `preferred` trades locality for guaranteed progress. Pick `required` for comm-bound training, `preferred` for comm-tolerant or deadline-bound work.
- How does TAS interact with gang scheduling (both must be satisfied)? **Answer:** They are conjunctive. Kueue admits only when quota is free **and** TAS finds a single domain at the requested level with room for the *whole* PodSet; gang (delegated to the underlying scheduler, or native to Volcano/KAI) then enforces all-or-nothing placement confined to that domain. Gang alone guarantees "all pods or none" but not *where* (it can place all 8 split 4+4 across the spine); TAS alone could place some pods well and leave others pending. Together the only admissible outcome for tightly-coupled training is all pods, in one fast domain, or wait — no partial placement, no cross-domain spread.
- Meta's MAST reduced a region's GPU demand-to-supply ratio from 2.63 to 0.98 using topology-aware placement constraints. What does a ratio above 1.0 mean operationally, and why would a topology constraint (not just more GPUs) be part of the fix? **Answer:** A demand-to-supply ratio above 1.0 means more high-priority work wants to run in that region than the region has capacity for at that moment — jobs queue, wait, or get placed suboptimally. Simply adding more GPUs to that region is a capex fix that doesn't address *why* demand concentrated there; a topology/placement-aware global scheduler can instead route (and constrain) workloads across the *existing* geo-distributed footprint so no single region gets overloaded relative to its supply — the same "unlock capacity you already own via better placement, not more hardware" thesis this whole module has been building, just applied at datacenter-region scale instead of rack scale.
- What does Kueue v0.15's switch from BestFit to LeastFreeCapacity for "unconstrained" TAS placement optimize for, and how does it relate to the fragmentation math in L7? **Answer:** BestFit picks the domain with the most free capacity and tries to minimize leftover slack on the *last* domain used — a tight-packing bias that can still leave many domains lightly, unusably touched. LeastFreeCapacity instead starts from the domain with the *least* free capacity, using up already-tight domains first and leaving domains with more slack untouched — explicitly, per Kueue's own KEP, "to prioritize minimizing fragmentation." This is the same instinct lesson 7's `effective_capacity.py` calculator teaches you to reason about after the fact (protect whole-node holes for large jobs) — Kueue v0.15 bakes that instinct into the default placement algorithm itself.

## Connections & what's next

This lesson is the "where" that completes the "whether/when" (lessons 3–4, quota and admission) and the "atomic or not" (lesson 2, gang; lesson 5, which scheduler owns gang) of the module's scheduling story. It reaches directly back into module 02b for the physics that motivates all of it, and it hands off directly to lesson 7: TAS's `required` constraint can *fail to admit* a gang on a fragmented cluster even when the fleet-wide GPU count looks sufficient — that is precisely the effective-capacity gap lesson 7 teaches you to measure and price. Next: lesson 7 takes the "topology-blind placement wastes GPU-hours" thesis from this lesson and generalizes it — fragmentation isn't only a topology problem, it's the general shape of "allocated capacity isn't usable capacity," and you'll build the calculator that turns that gap into a defensible dollar figure.

## References & further reading

**Primary sources**
- Kueue — Topology-Aware Scheduling (concept) — https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/ (fetch blocked by this session's egress proxy; canonical URL, previously verified and consistent with the KEP text fetched directly below) — the authoritative model: `Topology` levels, `required`/`preferred` semantics, PodSet annotations, and gang interaction.
- Kueue — Setting up Topology-Aware Scheduling (task) — https://kueue.sigs.k8s.io/docs/tasks/manage/setup_topology_aware_scheduling/ (fetch blocked; canonical URL) — the feature gate, ResourceFlavor `topologyName` wiring, and node-label requirements you follow for the kind demo.
- Kueue upstream — Topology Aware Scheduling KEP — https://github.com/kubernetes-sigs/kueue/blob/main/keps/2724-topology-aware-scheduling/README.md (fetched directly in this session) — read for the BestFit vs LeastFreeCapacity design rationale and the `TASBalancedPlacement` feature gate, straight from the source.
- Kueue upstream — TAS tracking issue #2724 — https://github.com/kubernetes-sigs/kueue/issues/2724 (fetched directly in this session) — the original motivating field complaint and design-doc/API/documentation deliverables that shaped the feature.
- Meta — "MAST: Global Scheduling of ML Training across Geo-Distributed Datacenters at Hyperscale" (OSDI 2024) — https://www.usenix.org/system/files/osdi24-choudhury.pdf (fetch blocked; canonical URL, content confirmed via WebSearch and the paper's USENIX conference page https://www.usenix.org/conference/osdi24/presentation/choudhury) — read for the 16,000-H100, multi-level topology-constraint case study and the 2.63→0.98 demand/supply result (dated snapshot of a specific 2024 training run).
- Module 02b — Host topology (NVLink domains, Topology Manager) — the interconnect-bandwidth physics this lesson builds on; reference for *why* cross-domain links are slow, which TAS exists to avoid.

**Real-world engineering blogs**
- Google Cloud (community) — "Kueue for AI: The Power of Atomic Admission & Topology Awareness" — https://medium.com/google-cloud/kueue-for-ai-the-power-of-atomic-admission-topology-awareness-59c2fd1f86ed (fetch blocked; canonical URL search-confirmed) — what it shows: TAS mechanics walked through on real GKE node labels, a second cloud-vendor account beside the Kueue docs.
- NVIDIA — "Deploying Disaggregated LLM Inference Workloads on Kubernetes" — https://developer.nvidia.com/blog/deploying-disaggregated-llm-inference-workloads-on-kubernetes/ (fetch blocked; content confirmed via WebSearch snippets) — what it shows: topology-aware placement applied to disaggregated inference, not just training — this lesson's mechanism carried into serving.

**Deeper dives**
- The USENIX OSDI '24 session page and recorded talk for the MAST paper (linked from the USENIX conference page above) — for the talk-format version of the case study if you prefer video to the paper.
- AWS EC2 UltraCluster placement groups and GCP compact placement policy documentation (search current docs at read time — these evolve) — for what a real cloud provider's topology labels actually correspond to physically, underneath Kueue's abstraction.
