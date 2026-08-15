---
lesson: 01
title: "GPU cost attribution: four sharing regimes"
module: 11
concept: "attribution sharing regimes"
status: not-started
est_time: "5 hrs"
prev: null
next: "02-allocated-vs-utilised.md"
artifacts: ["a per-regime attribution decision table + a written defence of your MIG fractional basis, dropped into the module deliverable"]
sources: 12
---

# GPU cost attribution: four sharing regimes

## Where this fits
This is the module's anchor lesson, and it sits directly on top of what you already have from modules 04, 05, 07, 08, and 10: the per-pod attribution mechanism (04), the SM_ACTIVE-vs-GPU_UTIL utilisation signal (05), cost-per-token (07), cost-per-run and MFU (08), and the capex-vs-cloud model (10). This lesson doesn't add a new mechanism on top of those — it adds the *taxonomy* that decides which mechanism even applies. Every later lesson in this module — the two-ledger split (02), the idle taxonomy per regime (03), fragmentation cost (04), unit economics (05), and eventually the OpenCost gap (09) and the FOCUS schema (10) — inherits whichever of the four sharing regimes defined here a given workload falls into. Get this taxonomy wrong and every dollar figure downstream in the module inherits the error silently.

## Why this matters
Every GPU cost story eventually collapses to one question: *given a GPU-hour of spend, which tenant, team, or workload owns what fraction of it?* On a fleet of exclusively-owned whole GPUs that question is boring — one pod, one device, 100%. The moment you share a device (and every neocloud and every serious platform team shares devices, because whole-A100/H100-per-tenant is how you set money on fire) the answer stops being an accounting lookup and becomes a *physics* question about what the hardware can and cannot tell you.

This is the lesson where FinOps stops being transferable. You already know rate optimisation, commitments, and tagging from the CPU world; none of it tells you how to split one physical H100's utilisation across four pods that are time-slicing it. That gap is exactly what a neocloud FinOps hire or a platform-eng "GPU cost owner" is expected to close, and it is a favourite interview probe precisely because most candidates answer it wrong — they assume a tool exists that reads per-tenant hardware counters off a time-sliced GPU. It does not, and knowing *why* it cannot is the signal.

Get the framing right and the rest of the module (idle detection, unit economics, fragmentation, chargeback) is downstream bookkeeping. Get it wrong and you will confidently produce showback numbers that are fiction, which is worse than no number at all because someone will make a procurement decision on them.

## What's new here (calibration)
- **Already yours (skip):** the *mechanism* of per-pod GPU cost attribution — kubelet pod-resources API device binding + DCGM per-device metrics joined on GPU UUID (module 04/05). The SM_ACTIVE-vs-GPU_UTIL "utilisation lie" (module 05). General FinOps: rates, commitments, tagging, allocation vs consumption as a concept.
- **New angle 1:** attribution is not one problem — it is *four* problems, one per sharing regime, and the regime dictates both the attribution math and whether the required signal even physically exists.
- **New angle 2:** the proof that **time-sliced GPU attribution is unsolvable from driver/DCGM signals alone** — the hardware reports one utilisation number per *device*, not per CUDA context, so per-tenant utilisation cannot be recovered. This is the intellectual centre of the lesson.
- **New angle 3:** how **DRA (Dynamic Resource Allocation)** changes the *allocation-side* picture (intent becomes explicit and machine-readable) while leaving the *utilisation-side* physics identical to MIG/time-slicing underneath.

## Core concepts

### The two-sided cost equation (frame for everything below)
Every regime splits into two ledgers that must be attributed separately:

- **Allocation cost** — the GPU-time a workload *reserved* (device binding / request), billed at the rate regardless of use. Deterministic; you always know this from the allocation record.
- **Utilisation cost** — the fraction of that reserved time that did *useful* work, measured from hardware counters.

Attribution difficulty is almost entirely on the *utilisation* side. Allocation is a bookkeeping join (who held the claim, for how long); utilisation requires the hardware to be able to say *whose work ran*. The four regimes differ mainly in whether the hardware can say that. Lesson 02 develops the two-ledger model in full; lesson 04 (fragmentation) shows how allocation-side waste compounds.

### Regime 1 — Exclusive whole-GPU
One pod holds the entire physical device for a bound interval.

- **Isolation:** total (nothing else runs on the device).
- **Attribution:** trivial. 100% of the GPU-hour to the binding pod for the bound duration. `cost = rate × bound_hours`.
- **Signal:** kubelet **pod-resources API** gives the pod→GPU-UUID binding; **DCGM** gives that UUID's counters. One device, one owner, no ambiguity. Utilisation attribution is exact because every counter sample on that UUID belongs to exactly one pod.
- **This is the only regime existing tools (OpenCost, Kubecost, vendor dashboards) handle cleanly**, because it is the only one where the device↔tenant map is 1:1 and the hardware counter is unambiguous.

### Regime 2 — MIG fractional (hardware partition)
Multi-Instance GPU carves one physical GPU (A100/H100/H200, Hopper/Ampere+) into up to 7 hardware-isolated GPU instances, each with its own SM slices, dedicated L2 slices, and dedicated memory. Profiles like `1g.10gb`, `2g.20gb`, `3g.40gb`, `7g.80gb` (H100 80GB), and mixed layouts.

- **Isolation:** hardware. Each MIG instance is a **separate device UUID** (`MIG-GPU-…`). DCGM exposes **per-MIG-instance** metrics; the pod-resources API binds pods to the MIG UUID. So the device↔tenant map is again 1:1 — *per instance*.
- **Attribution basis:** by the instance's *fraction of the physical GPU*. `cost_instance = fraction × (rate_of_whole_GPU) × hours`.
- **The subtlety — MIG profiles are NOT equal-sized, and compute and memory scale differently.** A `3g.20gb` instance gets 3 of 7 compute slices but its memory fraction is set by the `.20gb` (20/80 on an 80GB card). SM-fraction and memory-fraction diverge. You must pick a *fractional basis* and defend it:
  - **SM-slice basis:** `fraction = compute_slices / 7`. Defensible when the GPU is compute-bound and you want cost to track FLOP capacity.
  - **Memory basis:** `fraction = instance_mem / total_mem`. Defensible when VRAM is the scarce/binding resource (common for inference serving where memory, not SMs, caps tenancy).
  - **Blended weight:** `fraction = w·(slices/7) + (1−w)·(mem/total)`, with `w` chosen and documented. Defensible as a pragmatic middle when neither resource is cleanly the constraint.
- The **sum of allocated fractions can be < 1** (profiles don't tile the whole GPU; some SMs/memory are reserved as MIG overhead and left unallocated). That un-allocated remainder is *stranded capacity* — it must land somewhere (a platform "overhead" cost centre), not silently vanish. Forward-ref lesson 04 (fragmentation).

> **2026 snapshot — why platforms pay the attribution cost of fractional sharing.** A joint NVIDIA + Nebius benchmark of MIG-based fractional GPU allocation under NVIDIA Run:ai found that a 0.5-GPU MIG fraction delivered **86% of the concurrent-user capacity of a full GPU**, and that co-locating mixed inference workloads on shared GPUs supported roughly **3x more total system users** than exclusive whole-GPU allocation. That's the economic argument for regime 2 in one data point: you give up the trivial attribution of regime 1 in exchange for a near-linear capacity return. Treat these percentages as a dated 2026 vendor benchmark, not a universal constant — verify against your own fleet's workload mix before you plan capacity around it.

### Regime 3 — Time-sliced (software time-multiplexing) — the unsolvable case
Kubernetes GPU time-slicing (advertising `nvidia.com/gpu` as N *replicas* of one physical device via the device plugin's `sharing.timeSlicing` config), or plain oversubscription where several pods' CUDA contexts land on one physical GPU with **no hardware isolation**. The GPU **context-switches** between the sharers on the driver's schedule; they interleave in time on the same SMs and memory.

- **Isolation:** none. No memory protection between tenants beyond the CUDA context; no per-tenant compute partition.
- **Per-tenant hardware counters:** **do not exist.** DCGM samples the *device*, and the device's utilisation registers (SM_ACTIVE, DRAM activity, etc.) report **one aggregate number for the physical GPU across all sharers**. DCGM does not — cannot, from these registers — say which CUDA context was resident when the sample was taken. The counters are device-scoped, not context-scoped.
- **Therefore per-tenant utilisation attribution is provably unrecoverable from DCGM/driver signals alone.** You have N pods and exactly one utilisation number. There is no function from "one device-level SM_ACTIVE time series" to "N per-pod SM_ACTIVE time series" without extra information the driver does not expose through DCGM. This is not a tooling gap that a better exporter fixes; it is the counter's scope.

**The approximations, and their failure modes** (this is the interview answer — *"you cannot do it accurately; here is the menu and its error bars"*):

| Approximation | Rule | Fails when |
|---|---|---|
| **Equal split** | device utilisation / N per sharer | sharers are wildly uneven — one pod hammers the GPU, three idle; equal split over-bills the idle three, under-bills the hog. Error unbounded. |
| **Request-weighted** | split ∝ each pod's requested share | requests rarely track actual burn; a pod that requested a big slice but sat idle still gets billed for it. Bills *intent*, not *use*. |
| **CUDA-context-runtime-weighted** | split ∝ each context's resident/active time, harvested from `nvidia-smi pmon` per-process SM% or **MPS** (Multi-Process Service) accounting | pmon per-process utilisation is a coarse sampled estimate and can miss short-lived kernels; MPS accounting only exists if you actually run MPS (which changes the isolation model and isn't plain time-slicing); still an *estimate*, not a hardware ground truth. Best available, but carry an explicit error band. |

The honest deliverable for this regime is a chosen approximation **plus a stated uncertainty**, and a recommendation to move heavy multi-tenant sharing to MIG (regime 2) precisely because MIG restores per-tenant counters. *"How do you bill 4 pods time-slicing one A100?"* → **"Accurately, you can't — the device gives one counter for four contexts. I'd use CUDA-context-runtime weighting via MPS accounting as the least-wrong split, publish the error band, and push the tenants onto MIG if the money matters."**

### Regime 4 — DRA (Dynamic Resource Allocation)
The emerging Kubernetes path (structured parameters; `ResourceClaim` / `ResourceClaimTemplate` / `DeviceClass`; GA-track in recent releases, NVIDIA DRA driver for GPUs). Instead of the pod requesting an opaque `nvidia.com/gpu: 1` count, it declares a **ResourceClaim** that the driver satisfies with a specific device allocation — which can *express partitions and sharing declaratively* (a MIG profile, a shared allocation, a specific device with attributes).

- **What DRA fixes — allocation-side attribution.** The allocation *intent* becomes explicit, structured, and machine-readable: the ResourceClaim records exactly which device (or partition, or shared slice) each workload was granted, with its parameters. You no longer infer the binding from device-plugin counts; you read it from the claim's allocation result. Allocation-cost attribution gets *cleaner and auditable*.
- **What DRA does NOT change — utilisation-side physics.** Underneath, a DRA-allocated device is still a whole GPU, a MIG instance, or a time-shared slice. If the claim resolves to a MIG instance, you get per-instance counters (regime 2). If it resolves to a *shared* allocation over one physical device, you are back to regime 3 and the device still reports one aggregate counter. **DRA makes the intent legible; it does not give the hardware a per-context counter it never had.**
- Net: DRA improves the numerator's provenance (who was allocated what) but the utilisation attribution inherits whichever of regimes 1–3 the allocation lands on.
- **2026 signal:** the NVIDIA DRA driver was donated to CNCF/Kubernetes and highlighted at KubeCon 2026 — a real production-adoption marker that regime 4 is moving from "emerging KEP" to community-owned infrastructure, not a vendor side-project. See the real-world use cases below.

### Comparison table
| Regime | Isolation | Per-tenant HW counters? | Attribution basis | Primary error source |
|---|---|---|---|---|
| Exclusive whole-GPU | Total | Yes (1:1 UUID) | 100% to binding pod × bound hours | None (exact) |
| MIG fractional | Hardware partition | Yes (per-MIG-instance UUID) | Chosen profile fraction (SM / memory / blended) × GPU-hour | Choice of fractional basis; unallocated MIG remainder |
| Time-sliced | None | **No** (device-scoped only) | Approximation (equal / request-wt / context-runtime-wt) | **Fundamental** — one counter, N contexts; unbounded split error |
| DRA | Inherits underlying (whole / MIG / shared) | Inherits underlying | Allocation from ResourceClaim; utilisation per underlying regime | Utilisation error inherited from underlying regime |

### The load-bearing takeaway
Attribution accuracy is a property of the **sharing regime**, not of your tooling. Whole-GPU and MIG are attributable because the hardware gives you a per-tenant counter (1:1 UUID). Time-slicing is not, because it does not — full stop. DRA cleans up *who reserved what* but cannot conjure counters that never existed. Design the fleet's sharing model with attribution in mind: if a tenant's spend must be defensible to the dollar, put them on an exclusive device or a MIG instance, not a time-slice.

## Perspectives

**Kernel/driver perspective.** DCGM reads hardware performance-monitoring registers that NVIDIA exposes per physical device, and — for MIG — per hardware instance, because MIG instances get their own dedicated slice of those registers at the silicon level. Plain time-slicing shares one set of registers across N software contexts with no per-context tagging in the counter path. That's the literal basis of the regime-3 impossibility proof: it isn't a missing feature in DCGM, it's an absence of the physical wiring a device-scoped counter would need to be context-scoped. A future driver could theoretically expose per-context sampling (something MPS partially approximates), but plain time-sliced hardware counters, as they exist today, cannot be un-aggregated after the fact.

**Platform/scheduler-engineer perspective.** The regime a workload lands in is a scheduling decision, not a fact of nature, and it's visible in how the device plugin advertises capacity. `nvidia.com/gpu` with no sharing config means regime 1 (exclusive). The same resource name with `sharing.timeSlicing.replicas` set (sometimes surfaced as `nvidia.com/gpu.shared`) means regime 3. A MIG-profile resource name (`nvidia.com/mig-3g.40gb`) means regime 2. A `ResourceClaim` against a `DeviceClass` means regime 4, resolved to whichever underlying regime the driver picks. A scheduler engineer choosing which resource name to expose to which team is, whether they realize it or not, also choosing that team's attribution accuracy.

**FinOps/economics perspective.** What's technically possible and what's defensible to a finance partner are different bars. A finance stakeholder doesn't need proof-level rigor; they need a number they can stand behind in a budget review. That means the *chosen* approximation for regime 3, plus its stated error band, matters more than chasing an unattainable exact answer — and it means documenting *why* a basis was chosen (SM-slice vs memory for MIG) is itself a deliverable, not an implementation detail, because an auditor or a disputing team lead will ask.

**Workload-owner perspective.** The four regimes aren't neutral to the people running workloads on them. An ML researcher training a large model wants hard isolation (MIG, or better, exclusive) because noisy-neighbor interference on a shared SM pool corrupts their step-time measurements and their MFU numbers (module 08). An SRE optimizing fleet density wants time-slicing or MPS because it packs more small inference workloads per physical GPU and raises headline utilisation. These are opposed incentives on the *same* hardware, and the attribution policy this lesson builds is the referee: it's what lets a platform team say "yes, you can share this GPU, but here's what your bill will look like and here's the error band," rather than letting the sharing decision get made purely on density with cost as an afterthought.

## Real-world use cases

- **NVIDIA + Nebius, fractional MIG allocation for production inference** — https://nebius.com/blog/posts/scaling-inference-with-runai-fractional-gpus (companion post: https://developer.nvidia.com/blog/unlock-massive-token-throughput-with-gpu-fractioning-in-nvidia-runai/). What it shows: a joint production benchmark of NVIDIA Run:ai fractional GPU allocation on Nebius infrastructure — a 0.5-GPU MIG fraction reached 86% of full-GPU concurrent-user capacity, and co-locating mixed workloads on shared GPUs supported roughly 3x more total system users than exclusive allocation. Concrete, dollar-adjacent evidence for why regime 2 exists despite its attribution overhead.
- **NVIDIA at KubeCon 2026** — https://blogs.nvidia.com/blog/nvidia-at-kubecon-2026/. What it shows: NVIDIA donated its DRA driver to CNCF/Kubernetes — a real 2026 production-adoption signal that regime 4 (DRA) is moving from an emerging KEP to community-owned, GA-track infrastructure rather than a single-vendor experiment.
- **Microsoft Research, "Analysis of Large-Scale Multi-Tenant GPU Clusters for DNN Training Workloads" (the Philly trace, ATC'19)** — paper: https://www.usenix.org/conference/atc19/presentation/jeon ; public trace data: https://github.com/msr-fiddle/philly-traces. What it shows: a real production multi-tenant GPU cluster trace (117,325 jobs, Aug–Dec 2017) that documents GPUs as a monolithic, coarsely-shareable scheduling resource — the empirical ground under this lesson's regime taxonomy, from a fleet operator's own data rather than vendor marketing.
- **Alibaba PAI GPU cluster trace 2020** — https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2020. What it shows: a second, independent real-world trace — over 6,500 GPUs across roughly 1,800 machines, two months of real MLaaS production traffic — useful for cross-checking regime-mix assumptions against a different operator's fleet than Microsoft's.

## Worked example
A single H100 80GB, on-demand at **$3.00/GPU-hour**, observed for **1 hour**. Compare how the same physical hour attributes under three regimes.

**(a) Exclusive whole-GPU.** Pod `train-A` bound the whole device for the hour.
`cost(train-A) = $3.00 × 1 h = $3.00`. Done. If SM_ACTIVE averaged 40%, that is a *utilisation* fact (lesson 02) — attribution is still 100% of $3.00 to train-A.

**(b) MIG partitioned** into `3g.40gb` (tenant-X) + `3g.40gb` (tenant-Y) + `1g.10gb` (tenant-Z). Slices: 3+3+1 = 7 (fully tiled on compute). Memory: 40+40+10 = 90 > 80 nominal — because MIG memory profiles include per-instance reserved capacity; assume the layout is valid on the SKU.
- SM-slice basis: X = 3/7 = 0.4286, Y = 0.4286, Z = 1/7 = 0.1429.
  `cost(X) = 0.4286 × $3.00 = $1.286`, `cost(Y) = $1.286`, `cost(Z) = $0.429`. Sum = $3.00. Clean, because slices tile fully.
- Memory basis would give X = 40/90, etc., shifting money toward the big-memory tenants. **You must state which basis you used.** For a training fleet on H100 where compute is the constraint, SM-slice basis is the defensible default.

**(c) Time-sliced** — 4 pods (P1..P4) share the whole H100 via `timeSlicing.replicas: 4`. DCGM reports device SM_ACTIVE = 55% averaged over the hour; that is the *only* per-work number available.
- Equal split → each pod attributed $0.75, "earning" 55%/4 ≈ 13.75% each. But `nvidia-smi pmon` shows P1's context resident ~70% of samples, P2 ~25%, P3/P4 ~2.5% each.
- Context-runtime-weighted → P1 = 0.70×$3.00 = $2.10, P2 = $0.75, P3 = P4 = $0.075. Same $3.00 total, radically different bills.
- **The gap between the equal-split bill ($0.75 for P3) and the context-weighted bill ($0.075 for P3) — a 10× difference for the same pod — is the error band you must disclose.** No signal in this regime resolves it to the dollar.

## Practice
Feed these into the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md):
1. **Regime decision table.** For your fleet's real (or a plausible) tenancy mix, tag each workload with its sharing regime and write the exact attribution formula you'd apply to each. Flag which ones are exact vs approximate.
2. **Defend a MIG basis.** Pick SM-slice, memory, or a blended weight for your MIG cost split. Write a 1-paragraph defence naming the binding resource and what breaks under the other basis. Include the treatment of the unallocated MIG remainder.
3. **The time-slice error band.** Take a 3–4-pod time-slicing scenario, compute the bill under equal-split, request-weighted, and context-runtime-weighted, and report the min/max spread per pod as an explicit uncertainty. Write the one-sentence interview answer.
4. **DRA mapping.** Sketch a `ResourceClaim` for one shared and one MIG allocation and state, for each, which utilisation regime the cost attribution inherits.

## Common pitfalls
1. **"OpenCost/Kubecost shows a GPU cost number, so attribution is solved."** That number is the *allocated* ledger only (lesson 02) — it is blind to fractional MIG splits and time-sliced sharing, and it silently assumes regime 1 for every device. Forward-ref lesson 09, which reads this gap straight from the OpenCost source.
2. **Treating SM-slice fraction as always the "correct" MIG basis.** It's the right default for compute-bound training, but a memory-bound serving fleet (where VRAM, not SMs, caps how many replicas fit) should use the memory basis instead. The choice must be stated and justified per fleet — there is no universally correct basis.
3. **"DRA solves GPU cost attribution."** DRA only cleans up allocation-side provenance — you can now read exactly what was granted from the ResourceClaim. The utilisation-side physics is fully inherited from whichever underlying regime the claim resolves to; a DRA claim over a shared device is still regime 3, counters and all.
4. **Assuming "equal split" is a neutral, fair default for time-slicing.** It's actually the worst-case error mode when sharers are uneven — the worked example above shows a 10x mis-bill (P3 billed $0.75 under equal split vs an actual $0.075 of context-runtime work) for exactly this reason. Equal split is a last resort, not a safe default.
5. **Conflating MPS (spatial partitioning, partial isolation) with plain time-slicing (pure temporal multiplexing, no isolation).** They have different attribution properties: MPS *can* expose per-process accounting if configured deliberately for it, while plain time-slicing structurally cannot, because its counters were never scoped below the device.

## Self-check
- Why can DCGM attribute utilisation exactly under MIG but not under software time-slicing? **Answer:** MIG makes each instance a separate device UUID with its own hardware counters (1:1 tenant↔counter), so DCGM samples a counter owned by exactly one tenant. Under time-slicing all sharers share one physical device UUID; DCGM's registers are device-scoped and report a single aggregate number across all CUDA contexts, so there is no per-tenant counter to read — the mapping is many-to-one and non-invertible.
- A `3g.40gb` MIG instance — is its cost fraction 3/7 or 40/80, and how do you decide? **Answer:** Neither is automatically correct; 3/7 is the SM-slice (compute) fraction and 40/80 the memory fraction, and MIG profiles deliberately decouple the two. You decide by which resource is the binding constraint for that fleet (compute-bound training → SM-slice basis; memory-bound serving → memory basis) or a documented blended weight — and you must state and defend the choice, because it changes who pays.
- Does DRA solve GPU cost attribution? **Answer:** Only the allocation half. DRA makes allocation *intent* explicit and auditable via ResourceClaims (you read exactly which device/partition/share each workload got), but the utilisation attribution still inherits the physics of the underlying regime — a DRA claim resolving to a shared physical device is still regime-3 unsolvable; one resolving to a MIG instance gets regime-2 exact counters.
- Why does a fleet operator running mixed inference workloads accept the attribution overhead of MIG (regime 2) instead of just using plain time-slicing (regime 3) everywhere? **Answer:** Because MIG buys back exactly what time-slicing gives up: per-tenant hardware counters. The NVIDIA+Nebius benchmark shows fractional MIG delivering most of the capacity upside of sharing (86% of full-GPU capacity at a 0.5 GPU fraction, ~3x more users when co-locating) while staying in regime 2's exact-attribution world — time-slicing might pack marginally denser, but every dollar attributed off it carries an unbounded error band.

## Connections & what's next
This lesson is the spine the rest of module 11 hangs off. Lesson 02 takes the two-ledger frame introduced here (allocation cost vs utilisation cost) and formalizes it into the allocated-vs-utilised split. Lesson 03's idle taxonomy is regime-aware — what "idle" even means, and how confidently you can detect it, differs between an exclusive device, a MIG instance, and a time-sliced pod. Lesson 04's fragmentation math depends on the unallocated MIG remainder introduced in regime 2. Lesson 09's "where OpenCost fails" writeup is, largely, the claim that OpenCost only ever implements regime 1. And lesson 10's FOCUS schema needs a `SharingRegime`-shaped field precisely because this taxonomy exists. Next: **lesson 02** picks up the "two ledgers" thread already running through this lesson's worked example — allocated cost (what you reserved) vs utilised cost (what you actually used) — and turns it into the module's core accounting model.

## References & further reading

**Primary sources**
- NVIDIA MIG User Guide — profiles, per-instance device UUIDs, how hardware partitioning works: https://docs.nvidia.com/datacenter/tesla/mig-user-guide/
- NVIDIA DCGM documentation — device- and MIG-instance-level metrics, the counter scope this lesson's impossibility proof rests on: https://docs.nvidia.com/datacenter/dcgm/latest/
- Kubernetes device plugin GPU time-slicing config (NVIDIA k8s-device-plugin) — how `sharing.timeSlicing` is configured and advertised: https://github.com/NVIDIA/k8s-device-plugin#shared-access-to-gpus
- Kubernetes Dynamic Resource Allocation with structured parameters (KEP-4381) — the design doc for regime 4's `ResourceClaim` model: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4381-dra-structured-parameters
- NVIDIA DRA driver for GPUs — the reference implementation resolving claims to whole/MIG/shared devices: https://github.com/NVIDIA/k8s-dra-driver
- OpenCost GPU allocation source — read for how whole-GPU (regime 1) attribution is actually implemented today: https://github.com/opencost/opencost

**Real-world engineering blogs**
- NVIDIA + Nebius, "Scaling efficient production-grade inference with NVIDIA Run:ai on Nebius": https://nebius.com/blog/posts/scaling-inference-with-runai-fractional-gpus — what it shows: fractional MIG allocation reaching 86% of full-GPU capacity at a 0.5 GPU fraction, 3x more users co-locating mixed workloads.
- NVIDIA Developer Blog, "Unlock Massive Token Throughput with GPU Fractioning in NVIDIA Run:ai": https://developer.nvidia.com/blog/unlock-massive-token-throughput-with-gpu-fractioning-in-nvidia-runai/ — what it shows: the companion technical deep-dive on the same fractional-GPU benchmark.
- NVIDIA, "NVIDIA at KubeCon 2026": https://blogs.nvidia.com/blog/nvidia-at-kubecon-2026/ — what it shows: the DRA driver's donation to CNCF/Kubernetes, a real production-adoption signal for regime 4.

**Deeper dives**
- Jeon et al., "Analysis of Large-Scale Multi-Tenant GPU Clusters for DNN Training Workloads" (USENIX ATC'19): https://www.usenix.org/conference/atc19/presentation/jeon — the Philly trace paper; grounds the regime taxonomy in a real multi-tenant fleet's data.
- Microsoft Research, Philly trace public dataset: https://github.com/msr-fiddle/philly-traces — 117,325 real jobs with per-minute GPU/CPU/memory utilisation, useful for building your own regime-mix analysis.
- Alibaba, PAI GPU cluster trace 2020: https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2020 — a second independent real trace (6,500+ GPUs, ~1,800 machines) for cross-checking assumptions.

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
