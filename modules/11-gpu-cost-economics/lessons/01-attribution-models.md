---
lesson: 01
title: "GPU cost attribution: four sharing regimes"
module: 11
concept: "attribution sharing regimes"
status: not-started
est_time: "4 hrs"
artifacts: ["a per-regime attribution decision table + a written defence of your MIG fractional basis, dropped into the module deliverable"]
---

# GPU cost attribution: four sharing regimes

## Why this matters
Every GPU cost story eventually collapses to one question: *given a GPU-hour of spend, which tenant, team, or workload owns what fraction of it?* On a fleet of exclusively-owned whole GPUs that question is boring — one pod, one device, 100%. The moment you share a device (and every neocloud and every serious platform team shares devices, because whole-A100/H100-per-tenant is how you set money on fire) the answer stops being an accounting lookup and becomes a *physics* question about what the hardware can and cannot tell you.

This is the lesson where FinOps stops being transferable. You already know rate optimisation, commitments, and tagging from the CPU world; none of it tells you how to split one physical H100's utilisation across four pods that are time-slicing it. That gap is exactly what a neocloud FinOps hire or a platform-eng "GPU cost owner" is expected to close, and it is a favourite interview probe precisely because most candidates answer it wrong — they assume a tool exists that reads per-tenant hardware counters off a time-sliced GPU. It does not, and knowing *why* it cannot is the signal.

Get the framing right and the rest of the module (idle detection, unit economics, fragmentation, chargeback) is downstream bookkeeping. Get it wrong and you will confidently produce showback numbers that are fiction, which is worse than no number at all because someone will make a procurement decision on them.

## What's new here
- **Already yours (skip):** the *mechanism* of per-pod GPU cost attribution — kubelet pod-resources API device binding + DCGM per-device metrics joined on GPU UUID (module 04/05). The SM_ACTIVE-vs-GPU_UTIL "utilisation lie" (module 05). General FinOps: rates, commitments, tagging, allocation vs consumption as a concept.
- **New angle 1:** attribution is not one problem — it is *four* problems, one per sharing regime, and the regime dictates both the attribution math and whether the required signal even physically exists.
- **New angle 2:** the proof that **time-sliced GPU attribution is unsolvable from driver/DCGM signals alone** — the hardware reports one utilisation number per *device*, not per CUDA context, so per-tenant utilisation cannot be recovered. This is the intellectual centre of the lesson.
- **New angle 3:** how **DRA (Dynamic Resource Allocation)** changes the *allocation-side* picture (intent becomes explicit and machine-readable) while leaving the *utilisation-side* physics identical to MIG/time-slicing underneath.

## Core notes

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

### Comparison table
| Regime | Isolation | Per-tenant HW counters? | Attribution basis | Primary error source |
|---|---|---|---|---|
| Exclusive whole-GPU | Total | Yes (1:1 UUID) | 100% to binding pod × bound hours | None (exact) |
| MIG fractional | Hardware partition | Yes (per-MIG-instance UUID) | Chosen profile fraction (SM / memory / blended) × GPU-hour | Choice of fractional basis; unallocated MIG remainder |
| Time-sliced | None | **No** (device-scoped only) | Approximation (equal / request-wt / context-runtime-wt) | **Fundamental** — one counter, N contexts; unbounded split error |
| DRA | Inherits underlying (whole / MIG / shared) | Inherits underlying | Allocation from ResourceClaim; utilisation per underlying regime | Utilisation error inherited from underlying regime |

### The load-bearing takeaway
Attribution accuracy is a property of the **sharing regime**, not of your tooling. Whole-GPU and MIG are attributable because the hardware gives you a per-tenant counter (1:1 UUID). Time-slicing is not, because it does not — full stop. DRA cleans up *who reserved what* but cannot conjure counters that never existed. Design the fleet's sharing model with attribution in mind: if a tenant's spend must be defensible to the dollar, put them on an exclusive device or a MIG instance, not a time-slice.

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

## Self-check
- Why can DCGM attribute utilisation exactly under MIG but not under software time-slicing? **Answer:** MIG makes each instance a separate device UUID with its own hardware counters (1:1 tenant↔counter), so DCGM samples a counter owned by exactly one tenant. Under time-slicing all sharers share one physical device UUID; DCGM's registers are device-scoped and report a single aggregate number across all CUDA contexts, so there is no per-tenant counter to read — the mapping is many-to-one and non-invertible.
- A `3g.40gb` MIG instance — is its cost fraction 3/7 or 40/80, and how do you decide? **Answer:** Neither is automatically correct; 3/7 is the SM-slice (compute) fraction and 40/80 the memory fraction, and MIG profiles deliberately decouple the two. You decide by which resource is the binding constraint for that fleet (compute-bound training → SM-slice basis; memory-bound serving → memory basis) or a documented blended weight — and you must state and defend the choice, because it changes who pays.
- Does DRA solve GPU cost attribution? **Answer:** Only the allocation half. DRA makes allocation *intent* explicit and auditable via ResourceClaims (you read exactly which device/partition/share each workload got), but the utilisation attribution still inherits the physics of the underlying regime — a DRA claim resolving to a shared physical device is still regime-3 unsolvable; one resolving to a MIG instance gets regime-2 exact counters.

## Resources
- NVIDIA MIG User Guide (profiles, per-instance device UUIDs): https://docs.nvidia.com/datacenter/tesla/mig-user-guide/
- DCGM documentation (device- and MIG-instance-level metrics): https://docs.nvidia.com/datacenter/dcgm/latest/
- Kubernetes device plugin GPU time-slicing config (NVIDIA k8s-device-plugin): https://github.com/NVIDIA/k8s-device-plugin#shared-access-to-gpus
- Kubernetes Dynamic Resource Allocation with structured parameters (KEP-4381): https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4381-dra-structured-parameters
- NVIDIA DRA driver for GPUs: https://github.com/NVIDIA/k8s-dra-driver
- OpenCost GPU allocation source (how whole-GPU attribution is actually implemented): https://github.com/opencost/opencost

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
