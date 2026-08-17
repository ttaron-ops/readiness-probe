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
sources: 14
---

# GPU cost attribution: four sharing regimes

> Module: [💰 11 — GPU cost and unit economics](../README.md) · Deliverable: [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md)

## Where this fits

This is the module's anchor lesson, and it sits directly on top of machinery you already have. Module 04 built the per-pod attribution mechanism — the kubelet pod-resources API, the GPU-UUID join, the ownership map that survives sharing, and the two reconciliation identities that make the output auditable. Module 05 built the honest utilisation signal — `DCGM_FI_PROF_SM_ACTIVE` rather than `DCGM_FI_DEV_GPU_UTIL` — and the `sum_over_time` integral that turns a 0–1 ratio into GPU-hours without inflating bursty workloads. Modules 07, 08 and 10 gave you cost-per-token, cost-per-run with MFU, and the capex-vs-cloud model.

This lesson does not re-teach any of that collection machinery. It takes it as given and asks the question the machinery cannot answer on its own: **given one GPU-hour of spend, whose is it, how much of it, and how wrong can that answer be?** The answer is not a property of your tooling. It is a property of the *sharing regime* the workload runs under, because the regime decides whether the hardware is physically capable of telling you whose work ran.

Everything downstream in module 11 inherits this taxonomy. Lesson 02's two ledgers are per-regime. Lesson 03's idle taxonomy means different things on a MIG instance than on a time-sliced replica. Lesson 04's fragmentation math consumes the unallocatable MIG remainder defined here. Lesson 05's unit economics divides by a numerator whose error bars are set here. Lesson 09's OpenCost teardown is, in one sentence, "OpenCost implements regime 1 and applies it to everything." Lesson 10's FOCUS schema needs an attribution-regime dimension precisely because this taxonomy exists.

## Why this matters

Every GPU cost story collapses to one question: *given a GPU-hour of spend, which tenant owns what fraction of it?* On a fleet of exclusively-held whole GPUs that question is bookkeeping — one pod, one device, 100%, done. The moment you share a device the answer stops being a lookup and becomes a question about what the silicon can measure. And you *will* share devices: whole-H100-per-tenant for a fleet of 7B-parameter inference replicas is how you set money on fire.

This is where FinOps stops being transferable. You already know rate optimisation, commitment coverage and tag hygiene from the CPU world. None of it tells you how to split one physical H100's utilisation across four pods time-slicing it, because on a CPU the kernel accounts every scheduler tick to a cgroup and the answer is exact by construction. GPUs have no equivalent: the device exposes performance-monitoring counters scoped to the *device*, not to the CUDA context that happened to be resident when the counter was sampled.

The failure mode is not "you get no number." It is worse: you get a confident number that is wrong by an unbounded factor, and someone makes a procurement decision on it. The worked example below shows the same pod billed **$0.075 or $0.75 for the same hour** — a 10× swing — depending purely on which defensible-sounding approximation you picked. If you cannot say which one you used and how wrong it can be, your chargeback is fiction with a decimal point.

It is also the module's favourite interview probe, for a specific reason: most candidates assume a tool exists that reads per-tenant counters off a time-sliced GPU. It does not, and being able to explain *why it cannot* — and then produce the menu of approximations with their error sources — is the signal that separates someone who has run a shared fleet from someone who has read about one.

## What's new here (calibration)

- **Already yours (skip):** the *mechanism* of per-pod attribution — pod-resources API device binding joined to DCGM on GPU UUID (04), the `exported_` label trap and the normalising recording rule (05), the `SM_ACTIVE`-vs-`GPU_UTIL` distinction (05), the `sum_over_time × Δ/3600` integral for GPU-hours (05). General FinOps: rates, commitments, tagging, allocation-vs-consumption as a concept.
- **New angle 1 — attribution is four problems, not one.** The sharing regime dictates both the attribution formula *and* whether the signal it needs physically exists. A single "GPU cost per namespace" number computed uniformly across a mixed fleet is four different quantities added together.
- **New angle 2 — the non-invertibility argument, stated precisely.** One device-scoped counter, N resident contexts, and the algebra that says why no exporter can recover the split. Including the important qualification the earlier version of this lesson got wrong: it is unrecoverable *from DCGM device counters*, not from every interface the driver exposes. NVML per-PID sampling is a different, lower-fidelity interface that gets you part of the way, and knowing exactly where the line falls is the point.
- **New angle 3 — the error is redistributive, and that is a load-bearing property.** A fair-share split gets every device's *total* exactly right and every tenant's *share* wrong. That single fact is what makes an approximate chargeback defensible instead of embarrassing, and it gives you the exposure-fraction number to put in front of a stakeholder.
- **New angle 4 — DRA as of Kubernetes 1.34/1.35.** DRA core graduated to GA in v1.34 and the feature gate is locked on in v1.35; partitionable devices (KEP-4815) went alpha in v1.35 and beta in v1.36. What that actually changes for cost is the *allocation record*, not the counter.

## Core concepts

### 1. What attribution is, and where it breaks

Attribution is a chain of joins that starts at an invoice line and ends at a team name. Each hop is a different system with a different notion of identity, and exactly one hop can be lossy.

```
  THE COST-ATTRIBUTION LADDER — and the one rung where accuracy is lost

  ┌──────────────────────────────────────────────────────────────────────────┐
  │ [1] INVOICE LINE                    source: cloud bill / FOCUS row       │
  │     "p5.48xlarge, us-east-1, 744 h, $X"        identity: ResourceId      │
  │     ▸ granularity: ONE INSTANCE. 8 GPUs are one line item.               │
  └──────────────────────────────┬───────────────────────────────────────────┘
                                 │  divide by PHYSICAL GPU count (NVML),
                                 │  never by nvidia.com/gpu.count
                                 ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │ [2] NODE → PER-GPU RATE             source: node labels / instance table │
  │     r = node_hourly_rate / G_physical          identity: GPU UUID        │
  │     ▸ EXACT. Arithmetic only. No measurement involved.                   │
  └──────────────────────────────┬───────────────────────────────────────────┘
                                 │  pod-resources API: ListPodResources()
                                 │  → {namespace, pod, container, [device_ids]}
                                 ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │ [3] GPU → POD (ALLOCATION)          source: kubelet pod-resources        │
  │     which pod HOLDS this device, right now     identity: device_id       │
  │     ▸ EXACT IN EVERY REGIME. It is a scheduler fact, not a measurement.  │
  │       Even under time-slicing the plugin fabricates GPU-<uuid>::<n>,     │
  │       so you always know the exact set of contending holders.            │
  └──────────────────────────────┬───────────────────────────────────────────┘
                                 │  ◀══════════════════════════════════════╗
                                 │  DCGM counters keyed on UUID (+GPU_I_ID)║
                                 ▼                                         ║
  ┌──────────────────────────────────────────────────────────────────────┐  ║
  │ [4] DEVICE WORK → POD WORK          source: DCGM / NVML              │  ║
  │     how much of the device's work was THIS pod's                     │  ║
  │                                                                      │  ║
  │     exclusive : 1 counter,  1 holder   →  EXACT                      │  ║
  │     MIG       : 1 counter PER INSTANCE, 1 holder  →  EXACT           │  ║
  │     time-slice: 1 counter,  N holders  →  ✗✗✗ LOSSY ✗✗✗             │  ║
  │     MPS       : 1 counter,  N holders  →  LOSSY, but BOUNDED         │  ║
  │     DRA       : inherits whichever of the above it resolved to       │  ║
  └──────────────────────────────┬───────────────────────────────────────┘  ║
                                 │                                          ║
          ╔══════════════════════╧══════════════════════════════════╗       ║
          ║  THIS IS THE ONLY RUNG WHERE INFORMATION IS DESTROYED.  ║═══════╝
          ║  Everything above it is exact arithmetic or a recorded  ║
          ║  fact. Everything below it inherits rung 4's error      ║
          ║  unchanged — no downstream aggregation repairs it.      ║
          ╚══════════════════════╤══════════════════════════════════╝
                                 │  pod → namespace → cost centre
                                 │  (label / annotation lookup)
                                 ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │ [5] NAMESPACE → TEAM → $            source: your labels + rate card      │
  │     ▸ EXACT, and irrelevant: aggregating wrong numbers correctly still   │
  │       yields wrong numbers. Aggregation HIDES the error by averaging     │
  │       it away visually; it never removes it.                             │
  └──────────────────────────────────────────────────────────────────────────┘
```

Read the ladder once more and notice what it implies. Rung 3 — *who holds the device* — is exact in all four regimes. This is the single most under-appreciated fact in GPU cost work, and it is why the honest answer under time-slicing is an approximation with a **known denominator** rather than a guess: you always know exactly how many pods are contending, because the device plugin hands out distinct annotated IDs (`GPU-<uuid>::0`, `::1`, …) and pod-resources reports them verbatim. What you do not know is how the device's work divided among them.

Two ledgers therefore fall out of the ladder, and lesson 02 formalises them:

- **Allocation cost** — GPU-time a workload *reserved*, billed regardless of use. Derived from rungs 1–3 only. Exact in every regime, in every sharing mode, forever. It never depends on a counter.
- **Utilisation cost** — the share of that reserved time that did work, derived from rung 4. Its accuracy is a property of the regime.

**Attribution difficulty is entirely on the utilisation side.** That asymmetry is the reason the defensible chargeback default is to bill allocated hours (lesson 08) and report utilised hours as an efficiency signal: one of them survives an audit unconditionally and the other does not.

### 2. The signal inventory — what each system can and cannot tell you

Before the regimes, know the sources. Four systems can say something about who owns GPU work, and they disagree in structured ways.

| Source | What it reports | Identity it keys on | Granularity floor | Fails when |
|---|---|---|---|---|
| **kubelet pod-resources API** (`/var/lib/kubelet/pod-resources/kubelet.sock`, `ListPodResources`) | which container was assigned which device IDs, *now* | `{namespace, pod, container}` → `device_ids[]` | the device ID the plugin invented | no history — it is a snapshot API with no watch; you must poll (04.3) |
| **DCGM** (`nv-hostengine`, scraped by `dcgm-exporter`) | hardware counters — `SM_ACTIVE`, `PIPE_TENSOR_ACTIVE`, `DRAM_ACTIVE`, power, framebuffer | GPU `UUID`, plus `GPU_I_ID`/`GPU_I_PROFILE` under MIG | **one physical device, or one MIG instance** | several CUDA contexts share one device — the counter cannot separate them |
| **NVML per-process** (`nvmlDeviceGetProcessUtilization`, `nvmlDeviceGetComputeRunningProcesses`) | coarse per-PID SM% and framebuffer bytes, sampled into a ring buffer | OS PID → cgroup → pod UID | one process, coarse and sampled | `NVML_ERROR_NOT_SUPPORTED` on some SKUs/drivers; misses short kernels; ring overruns if you poll slowly |
| **`ResourceClaim.status.allocation`** (DRA) | exactly which device, with which attributes and config, was granted to which claim | claim UID → pod | the device the driver published in a `ResourceSlice` | tells you nothing about work performed — it is an allocation record |

The load-bearing row is DCGM's. **DCGM's granularity floor is the measurement identity the hardware exposes**, and that floor is a physical device or a MIG instance — never a CUDA context. Everything that follows is a consequence of that one sentence.

### 3. Regime 1 — exclusive whole-GPU

One pod holds one physical device for a bound interval. Nothing else runs on it.

**Mechanics.** The device plugin advertises `nvidia.com/gpu: N` where N is the physical GPU count. The scheduler binds the pod; the kubelet's `Allocate()` call picks specific devices and injects them into the container (as `NVIDIA_VISIBLE_DEVICES` for the NVIDIA container runtime, or CDI device nodes). pod-resources then reports `device_ids: ["GPU-8f2c…"]` — a bare UUID, no annotation suffix. DCGM's series for that UUID belongs to exactly one container.

**Attribution.** Trivial and exact:

```
  allocated_gpu_hours(p) = 1.0 × hours_bound(p)
  allocated_cost(p)      = r × allocated_gpu_hours(p)

  utilised_gpu_hours(p)  = ∫ SM_ACTIVE_UUID(t) dt      over the bound interval
                         ≈ sum_over_time(SM_ACTIVE[W]) × Δ / 3600     (module 05)
  utilised_cost(p)       = r × utilised_gpu_hours(p)
```

where `r` is $/physical-GPU-hour and `Δ` the sample spacing in seconds. There is no error term. Every counter sample on that UUID is unambiguously this pod's.

**Why this matters more than it looks.** This is the only regime existing tools implement correctly, and they implement it by assuming it. OpenCost's allocation pipeline computes `GPUHours = <GPU request> × hours` (`applyGPUsAllocated` in `pkg/costmodel/allocation_helpers.go`) and then `GPUCost = GPUHours × node.CostPerGPUHr`. That is exactly regime 1's formula — correct here, and silently applied everywhere else. Lesson 09 reads that code line by line.

**The one trap.** `r` must be the node's hourly cost divided by the **physical** GPU count from NVML, not by `nvidia.com/gpu.count`. Under a MIG `single` strategy that label counts MIG devices (56 on an 8-GPU node partitioned 7×), and under time-slicing it counts replicas. Dividing by the label instead of the physical count is a silent 4× or 7× error in every dollar figure downstream, and — this is the nasty part — it does not violate any reconciliation identity, because the shares stay internally consistent. Only an external check against the invoice catches it.

### 4. Regime 2 — MIG fractional (hardware partition)

Multi-Instance GPU physically partitions one card into up to seven GPU instances, each with its own SM slices, its own L2 slices, its own memory slices and its own copies of the performance counters. The partition is enforced in silicon, not by the driver's scheduler.

**Mechanics — what "partition" actually means.** The card is divided along two independent axes that do not have the same number of units:

- **Compute slices** — the SMs are grouped into 7 slices. On an A100, the MIG-addressable pool is **98 multiprocessors**, so one compute slice is 14 SMs. (The A100 die has 108 SMs; the MIG-usable count is smaller and is what the profile tables report.)
- **Memory slices** — the HBM stacks and their L2 partitions are divided into **8 memory slices**, not 7.

A profile name `NgM.Xgb` reads as *N compute slices, X GB of framebuffer*. Because compute has 7 units and memory has 8, **the two axes cannot both tile perfectly**, and that is the origin of MIG's structural stranding.

The NVIDIA-authored partitionable-devices KEP (KEP-4815, `keps/sig-scheduling/4815-dra-partitionable-devices`) publishes the A100-40GB numbers as a counter model, which is the cleanest primary-source statement of the geometry:

```
  A100-40GB, as modelled in KEP-4815's CounterSet:

    full-card counters:   memory 40Gi (= 40,960 MiB)
                          multiprocessors 98
                          copy-engines 7
                          memorySlice0 … memorySlice7   (8 slices)

    1g.5gb profile consumes:  memory 4,864 MiB
                              multiprocessors 14
                              copy-engines 1
                              exactly one memorySlice
```

Do the arithmetic, because it is the whole point:

```
  COMPUTE, 7 × 1g.5gb :   7 × 14 MP            = 98 MP  of 98 MP   = 100.0 %
  MEMORY,  7 × 1g.5gb :   7 × 4,864 MiB        = 34,048 MiB
                          of 40,960 MiB                            =  83.1 %
                          ─────────────────────────────────────────────────
  STRUCTURALLY UNALLOCATABLE MEMORY            =  6,912 MiB = 16.9 % of HBM
```

**Compute tiles perfectly. Memory does not, and cannot.** That 16.9% is not fragmentation you can schedule around and not overhead you can tune away; it is the eighth memory slice plus per-instance reserved capacity, sitting there because the geometry says so. On a card costing `r` per hour it is `0.169 × r` per hour of capacity that has no possible tenant. Lesson 04 prices it; here the job is to know it exists and that **it must be booked somewhere** — a platform overhead cost centre — or your per-GPU shares will not sum to 1 and the identity-A check from module 04 will fail with no obvious cause.

**Attribution.** Each MIG instance is a distinct measurement identity. `dcgm-exporter` emits per-instance series carrying `GPU_I_ID` and `GPU_I_PROFILE` alongside the *parent* physical `UUID` (this is the detail that trips people: DCGM does **not** key on the `MIG-…` UUID that pod-resources hands you — getting from one to the other needs an NVML hop, module 04.6). One instance belongs to exactly one container. So rung 4 is exact, and the only open question is the *fraction of the card* an instance represents.

**The fractional basis, and why it is a choice you must defend.** Cost splits by fraction:

```
  cost(instance) = f × r × hours          f = the instance's fraction of one card
```

and `f` has three defensible definitions that disagree:

| Basis | Formula | A100-40GB `1g.5gb` | A100-40GB `3g.20gb` | Defensible when |
|---|---|---|---|---|
| **Compute (SM) share** | `MP_instance / MP_card` | 14/98 = **0.1429** | 42/98 = **0.4286** | compute is the binding resource — training, prefill-heavy serving |
| **Memory share** | `mem_instance / mem_card` | 4,864/40,960 = **0.1188** | ~20,480/40,960 = **0.5000** | VRAM is what caps tenancy — decode-bound serving, many-model hosting |
| **Blended** | `w·compute + (1−w)·memory`, `w` documented | at w=0.5: **0.1308** | at w=0.5: **0.4643** | neither resource is cleanly the constraint |

For `1g.5gb` the two bases differ by 14/98 ÷ 4864/40960 = **1.20×** — a 20% relative gap on the same slice, entirely from which axis you called scarce. That is real money at fleet scale, and it is the single most likely thing a disputing team lead will challenge. **Pick per fleet, emit the basis as a label on the metric, and write the paragraph defending it.** Do not average the two without saying so; an undocumented blend is a number with no meaning.

**Do not cost a MIG slice as a whole GPU.** Seven allocated `1g` slices on one card is one card's worth of cost. A tool that counts `nvidia.com/mig-1g.5gb` allocations and multiplies by the whole-GPU rate reports 7× the fleet's real spend, and the number looks plausible enough to ship.

### 5. Regime 3 — time-sliced, and the non-invertibility argument

Kubernetes GPU time-slicing advertises one physical device as N *replicas*. In the NVIDIA device plugin's config API (`api/config/v1/sharing.go`) this is `sharing.timeSlicing` with a `replicas` count per resource; `renameByDefault: true` surfaces the replicas as `nvidia.com/gpu.shared`, and `failRequestsGreaterThanOne` makes a request for more than one replica fail rather than silently hand a pod several slices of the same card. There is no hardware partition and no memory isolation: all N pods open CUDA contexts on the same device, and the driver's channel scheduler time-multiplexes them onto the same SMs.

**Why the counter cannot be un-mixed.** State it as algebra, because prose lets people believe a better exporter would fix it.

Let `u_i(t)` be the true instantaneous SM-activity attributable to context *i*, for *i* = 1…N. What DCGM samples is one scalar per device per sample:

```
        N
  U(t) = Σ  u_i(t)                     ← the ONE number you get
       i=1

  You have 1 equation and N unknowns per sample.
  For N ≥ 2 the system is underdetermined: infinitely many
  vectors (u_1 … u_N) produce the same U(t).

  Recovering the u_i needs N−1 additional INDEPENDENT equations.
  DCGM device counters supply none of them, because every field
  in the device's counter bank is aggregated over the same
  hardware, over the same sample window, across all contexts.
```

There is no counter-select register on the GPU that says "count only warps belonging to context 7." Contexts are a driver-level construct; the SM performance counters are physical and count warp-cycles regardless of whose warps they were. **That is why this is not a tooling gap: DCGM is reporting exactly what the silicon exposes.** A future architecture could tag counters by context (MPS approximates this at the process level, see below), but on the hardware in front of you, an aggregated scalar cannot be decomposed after the fact.

Draw the timeline, because the picture is what makes it stick:

```
  ONE PHYSICAL H100, timeSlicing.replicas: 4, four pods resident.
  The driver's channel scheduler round-robins contexts onto the SMs.

  t →   0ms      50      100     150     200     250     300     350   400
        ├────────┼───────┼───────┼───────┼───────┼───────┼───────┼──────┤
  ctx A ████████████████████░░░░░░░░░████████████████████████░░░░░░░░░░░
  ctx B ░░░░░░░░░░░░░░░░░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░████████░░░
  ctx C ░░░░░░░░░░░░░░░░░░░░░░░░░░░░████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░███
  ctx D ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                                                        (D queued nothing)

  ── what the SILICON counts ────────────────────────────────────────────
  SM warp-cycle counters increment for whichever warps are resident.
  They do not carry a context tag. There is nowhere to put one.

  ── what DCGM PUBLISHES, once per collect-interval ─────────────────────
  DCGM_FI_PROF_SM_ACTIVE{UUID="GPU-8f2c…"}  0.55        ← ONE scalar
              ▲
              └── for the whole 400 ms window, all four contexts summed

  ── what dcgm-exporter LABELS it with, in Kubernetes mode ──────────────
  the SAME 0.55, FANNED OUT to every holding pod:
      {…, pod="pod-a"} 0.55   {…, pod="pod-b"} 0.55
      {…, pod="pod-c"} 0.55   {…, pod="pod-d"} 0.55

  ⇒ A naive sum() over those four series reports 2.20 GPU-equivalents
    of work on a device that can produce at most 1.00.
    THE OVER-REPORT FACTOR IS EXACTLY THE HOLDER COUNT, N,
    independent of load. That signature is how you detect the bug.
```

That last box is the single most common defect in home-grown GPU cost dashboards, and it is detectable for free: if `Σ_pods utilised_share ÷ device_utilisation` comes out to an exact integer, you forgot to deduplicate a fanned-out device metric (module 04, identity B).

**The approximations, with their real error sources.** This is the interview answer: not "you cannot do it," but "here is the menu and here is each option's error."

| Approximation | Rule | Extra signal required | Error source | Error bound |
|---|---|---|---|---|
| **Equal split** | `u_i = U / N` | none | load skew between sharers | **unbounded** as a *ratio*: a pod doing 1% of the work is billed for 1/N. Bounded in absolute terms by `U` (see the theorem below) |
| **Request-weighted** | `u_i ∝ replicas held by i` | pod-resources only | identical to equal split unless pods hold different replica counts; bills *intent*, not use | as above |
| **Per-PID NVML** | `u_i = U × smUtil_i / Σ smUtil` | `nvmlDeviceGetProcessUtilization` + `/proc/<pid>/cgroup` → pod UID; `hostPID` or a shared PID namespace | coarse sampling; short kernels missed; ring-buffer overruns if polled slowly; `NOT_SUPPORTED` on some SKUs | typically a few percent to low tens of percent; **must be measured, not assumed** |
| **Per-PID memory** | `u_i = U × fbUsed_i / Σ fbUsed` | `nvmlDeviceGetComputeRunningProcesses` | framebuffer occupancy is a terrible proxy for compute; a pod holding a big idle KV cache looks busy | large and *biased*, not just noisy |
| **MPS with a thread cap** | `u_i = U / N`, but each tenant's SM share is capped in the daemon | MPS control daemon, `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` | still a split of one scalar | **bounded by the configured cap** — a tenant capped at 25% cannot have used more than 25% |

Note the important correction to the folklore version of this lesson: the claim is *not* that per-tenant utilisation is unknowable by any means. It is that **it is unrecoverable from DCGM's device-scoped counters**, which is the interface every cost tool actually consumes. NVML's per-process interface is a genuinely different, lower-fidelity path that recovers an estimate — and `dcgm-exporter` 4.6+ uses exactly that path for two fields (`DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_DEV_FB_USED`) when `--kubernetes-virtual-gpus` is enabled, emitting per-pod series alongside the device series. Every `DCGM_FI_PROF_*` field — including `SM_ACTIVE`, the one you actually want — remains device-level and fanned out. So the practical state of the world is: **a coarse per-pod split for the utilisation duty-cycle metric, and no split at all for the honest occupancy metric.**

**The theorem that saves the chargeback.** Fair-share splitting is wrong per pod but exactly right per device. Derive it once:

```
  Device at utilisation U, N holders, TRUE shares s_i with Σ s_i = 1.
  Fair-share estimator assigns  û_i = U/N.
  True value is                 u_i = U · s_i.

  Per-pod error:      err_i = û_i − u_i = U · (1/N − s_i)

  Sum over holders:   Σ err_i = U · ( Σ 1/N − Σ s_i )
                              = U · ( 1 − 1 )
                              = 0                       ← EXACTLY ZERO

  ⇒ THE ERROR IS PURELY REDISTRIBUTIVE.
    The device's total utilised cost is exactly right.
    Some of those dollars are simply on the wrong invoice.

  Worst-case single-pod error:
       |err_i| ≤ U · max(1/N, 1 − 1/N) ≤ U
  i.e. a pod can be mis-billed by at most that device's ENTIRE
  utilised cost, and only in the degenerate case where one tenant
  did all the work.
```

This is what lets you say, in a budget review: *"the fleet total is exact; a named fraction of the per-team split is an estimate, and here is that fraction."* Compute the fraction — the **exposure** — like this:

```
  G    = physical GPUs in the fleet
  G_s  = GPUs running a software sharing regime (time-slicing / MPS)
  ū_x  = mean utilisation on exclusively-held or MIG GPUs
  ū_s  = mean utilisation on shared GPUs

  exact utilised GPU-equivalents      = (G − G_s) × ū_x
  estimated utilised GPU-equivalents  = G_s × ū_s
  exposure fraction  E = G_s·ū_s / [ (G−G_s)·ū_x + G_s·ū_s ]

  E is the share of every utilisation-based chargeback dollar that
  rests on an approximation. Report it next to the total. Always.
```

If `E` comes back at 0.47, the correct next project is not a prettier dashboard — it is moving those workloads onto MIG, and you now have the sentence that funds it.

**MPS is not time-slicing, and the difference is the error bound.** The Multi-Process Service runs a control daemon that funnels several processes' work through one CUDA context, and `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE` (or the per-client `nvidia-cuda-mps-control` setting) caps the fraction of SMs any one client may occupy. That cap is what makes MPS's estimate *bounded*: a client capped at 25% provably did not consume more than 25% of the device's throughput, so `|err_i|` is bounded by the cap rather than by `U`. Conflating the two regimes because they both produce `::N` device IDs is a real mistake — read the node's `nvidia.com/gpu.sharing-strategy` label (`none` / `time-slicing` / `mps`), do not infer the mode from the shape of a device ID.

### 6. Regime 4 — DRA (Dynamic Resource Allocation)

DRA replaces "the pod asks for an opaque count of `nvidia.com/gpu`" with "the pod declares a **ResourceClaim** and the driver satisfies it with a specific device." The status as of this writing: **DRA core graduated to GA in Kubernetes v1.34, and v1.35 locks the feature gate on**; the partitionable-devices extension (KEP-4815) went alpha in v1.35 and beta in v1.36.

**The object model, in three pieces.**

- **`ResourceSlice`** — published *by the driver*, per node. It lists the devices the driver can hand out, each with typed attributes (model, memory, driver version, topology hints) and capacities. This is the driver telling the scheduler what exists.
- **`DeviceClass`** — published by the cluster admin. A named group of devices plus vendor selectors and default config, so users reference a class instead of restating selector CEL every time.
- **`ResourceClaim`** / **`ResourceClaimTemplate`** — the workload's request: device requests expressed as CEL selectors over the published attributes, plus config that is passed to the driver but does *not* influence allocation. `status.allocation` records exactly what was granted.

**What partitionable devices add** (KEP-4815) is the mechanism that matters for cost. The driver publishes a **`CounterSet`** — a named pool of counters for a physical device — and each advertisable partition declares, via **`ConsumesCounters`**, how much of each counter it takes. The scheduler subtracts consumed counters as it allocates, and a partition becomes unallocatable the moment any single counter is exhausted. That is how overlapping MIG geometries are prevented without the scheduler knowing what MIG is: the A100 example in the KEP models the card as `memory: 40Gi`, `multiprocessors: 98`, `copy-engines: 7` plus eight individual `memorySliceN` counters, and a `1g.5gb` partition consumes `4864Mi` / `14` / `1` / one memory slice. Two profiles that would need the same memory slice simply cannot both be allocated.

**What DRA fixes for cost accounting.**

1. **Allocation provenance becomes a first-class API object.** You no longer infer the binding from device-plugin counts; you read `ResourceClaim.status.allocation` and get the device identity plus the attributes it was selected on. This is auditable in a way a device-plugin count never was.
2. **The kubelet's pod-resources API reports DRA-allocated devices** as of v1.34, so node agents (including your exporter) can see DRA allocations through the interface they already use. `dcgm-exporter` gates this behind `--kubernetes-enable-dra` / `KUBERNETES_ENABLE_DRA`, **default `false`** — leave it off on a DRA cluster and those GPUs show up with no pod labels at all, i.e. they land in your unallocated bucket and understate tenant allocation.
3. **Partition geometry becomes schedule-time**, which is a fragmentation lever (lesson 04), not an attribution lever.

**What DRA does not fix.** Nothing at rung 4. A DRA claim that resolves to a whole GPU is regime 1. To a MIG instance, regime 2. To a shared allocation over one physical device — and DRA explicitly supports sharing one allocated device between pods — it is regime 3, with the same one-counter-N-contexts algebra as before. **DRA makes intent legible; it does not give the hardware a counter it never had.** Anyone who tells you DRA solves GPU cost attribution has solved the easy half and not noticed the hard half.

### 7. The regimes side by side

| Regime | Isolation | Measurement identity | Pods per identity | Utilisation attribution | Primary error source | Fleet policy implication |
|---|---|---|---|---|---|---|
| **1 · Exclusive** | total | GPU UUID | 1 : 1 | **exact** | none | cheapest to bill, most expensive to run |
| **2 · MIG** | hardware partition | UUID + `GPU_I_ID` | 1 : 1 | **exact**, once you fix the fractional basis | choice of basis (up to ~20% relative on small profiles); unallocatable memory remainder (~17% on A100-40GB at 7×1g) | the sweet spot when spend must be defensible per tenant |
| **3a · Time-sliced** | none | GPU UUID | N : 1 | **estimate** | load skew; unbounded per-pod ratio error | densest packing, least defensible bill |
| **3b · MPS** | thread-percentage cap | GPU UUID | N : 1 | **bounded estimate** | still a split of one scalar, but capped | use when you need density *and* a stated error bound |
| **4 · DRA** | inherits | inherits | inherits | inherits | inherits, plus the exporter's `--kubernetes-enable-dra` default `false` | cleans up the allocation record; changes nothing at rung 4 |

### 8. The takeaway you design fleets with

**Attribution accuracy is a property of the sharing regime, not of your tooling.** Whole-GPU and MIG are exactly attributable because the hardware hands you a counter owned by exactly one tenant. Time-slicing is not, because it does not. DRA cleans up who reserved what and cannot conjure counters that never existed.

So attribution stops being an observability problem and becomes a **platform design decision**, made — usually unknowingly — by whoever chose which resource name to expose to which team:

```
  resource name a team sees          →  regime  →  bill defensibility
  ─────────────────────────────────────────────────────────────────────
  nvidia.com/gpu (no sharing config) →   1      →  exact, forever
  nvidia.com/mig-3g.40gb             →   2      →  exact, given a stated basis
  nvidia.com/gpu.shared              →   3a     →  estimate, unbounded per pod
  nvidia.com/gpu (MPS-backed)        →   3b     →  estimate, bounded by the cap
  a ResourceClaim against a          →   4      →  whatever it resolved to
    DeviceClass
```

If a tenant's spend must be defensible to the dollar — because they are an external customer, or because a chargeback dispute would be expensive — put them on an exclusive device or a MIG instance. If density matters more than per-tenant precision, share and publish the exposure fraction. What you may not do is share *and* claim precision.

## Perspectives

**Kernel/driver.** DCGM reads hardware performance-monitoring registers that the GPU exposes per physical device, and — under MIG — per hardware instance, because a MIG instance owns its slice of those registers in silicon. Plain time-slicing shares one register set across N software contexts with no per-context tagging anywhere in the counter path. The counters increment on warp-cycles; a warp does not carry an accounting identity. That is the literal basis of the regime-3 argument: not a missing DCGM feature, but the absence of the wiring a device-scoped counter would need to become context-scoped. MPS approximates the missing wiring at the process level by funnelling clients through one context and capping each client's thread percentage — which is precisely why MPS gets an error *bound* and raw time-slicing does not.

**Platform/scheduler engineer.** The regime is a scheduling decision and it is visible in the resource name the plugin advertises. A scheduler engineer choosing to expose `nvidia.com/gpu.shared` to the research team instead of a MIG profile is choosing that team's attribution accuracy — and, downstream, whether their spend can be defended in a dispute. Make that choice explicitly and write it down, because it will be re-litigated the first time a big number lands in someone's inbox.

**FinOps/economics.** What is technically possible and what is defensible to a finance partner are different bars. Finance does not need proof-level rigour; it needs a number that survives a challenge. That means the *chosen* approximation plus its stated error band beats chasing an unattainable exact answer — and it means the documented basis for a MIG split is itself a deliverable, not an implementation detail. The redistributive-error result is the sentence that wins the room: *"the fleet total is exact to the cent; 47% of the per-team split is an estimate; here is what it would cost to make it 0%."*

**Workload owner.** The regimes are not neutral to the people running on them. A researcher measuring step time and MFU wants hard isolation, because a noisy neighbour on a shared SM pool corrupts their measurements as well as their throughput. An SRE optimising density wants time-slicing or MPS, because it packs more small replicas per card and lifts headline utilisation. Those are opposed incentives on the same hardware, and the attribution policy is the referee: it is what lets the platform say *"yes, share — and here is exactly what your bill will look like and how uncertain it is,"* instead of letting density win by default with cost discovered later.

**Auditor.** A number you cannot check is an assertion, not a measurement. What makes this pipeline auditable is that rungs 1–3 are exact and independently verifiable: hand someone the per-GPU allocated shares plus the unallocated remainder and they can confirm it sums to 1.0 per physical GPU without knowing what a compute slice is. Every claim of uncertainty should therefore be localised to rung 4 and labelled there — which is why the attribution regime belongs on the metric as a label and in the cost schema as a dimension (lesson 10), not in a footnote.

## Real-world use cases

- **NVIDIA — KEP-4815, DRA partitionable devices** (`keps/sig-scheduling/4815-dra-partitionable-devices`, alpha in Kubernetes v1.35, beta in v1.36). **What it shows:** an NVIDIA-authored, machine-readable model of an A100's partition geometry — `memory: 40Gi`, `multiprocessors: 98`, `copy-engines: 7`, eight `memorySliceN` counters, with a `1g.5gb` partition declared as `4864Mi` / `14` / `1` / one slice. This is the primary source behind §4's stranding arithmetic: 7 × 4,864 MiB = 34,048 MiB of 40,960 MiB, leaving **16.9% of the framebuffer structurally unallocatable** at the densest geometry. It also shows the scheduler-side mechanism — allocate a partition, decrement counters, refuse anything whose counters are exhausted — that makes overlapping MIG profiles unschedulable by construction.

- **Kubernetes v1.34 — DRA graduated to GA** (kubernetes.io blog, Sept 2025; kubelet pod-resources API extended to report DRA-allocated devices). **What it shows:** regime 4 is no longer speculative. The practical consequence for this lesson is the pod-resources change: node agents can see DRA allocations through the interface they already speak, which is why `dcgm-exporter`'s `--kubernetes-enable-dra` flag exists — and why its **default of `false`** is a live footgun that silently moves DRA-held GPUs into your unallocated bucket.

- **NVIDIA `k8s-device-plugin` — `api/config/v1/sharing.go` and `replicas.go`.** **What it shows:** the exact shape of the regime-3 configuration, in code rather than in prose. `Sharing` carries `timeSlicing` and `mps` as separate `ReplicatedResources`; `SharingStrategy()` returns `mps` if the MPS block replicates, else `time-slicing`, else `none`. `ReplicatedResources` carries `renameByDefault` (the `nvidia.com/gpu.shared` rename) and `failRequestsGreaterThanOne`. This is the file that proves time-slicing and MPS are two configurations of the same replication machinery with different isolation — hence identical `::N` device IDs and completely different error bounds.

- **OpenCost — `pkg/costmodel/allocation_helpers.go`.** **What it shows:** `applyGPUsAllocated` sets `GPUHours = <GPU request> × hours`, and `GPUCost = GPUHours × node.CostPerGPUHr`. Read literally, that is regime 1's formula with no regime check anywhere. The same code path also records `GPUUsageAverage` (from `DCGM_FI_PROF_GR_ENGINE_ACTIVE`) and an `IsGPUShared` flag — so the tool *knows* a device is shared and still costs it as though it were not. That gap, named from the source, is lesson 09's deliverable.

- **Microsoft Research — the Philly trace (USENIX ATC'19), 117,325 jobs, Aug–Dec 2017.** **What it shows:** a real multi-tenant production fleet in which GPUs are treated as monolithic, coarsely-shareable scheduling units, with per-minute utilisation recorded per job. It is the empirical floor under this taxonomy: the regime mix and the utilisation spread are not hypotheticals invented for a lesson, they are what a large operator's own data looks like. Useful as a source of realistic request shapes when you build your own regime-mix analysis.

- **Alibaba PAI GPU cluster traces (2020 and 2023).** **What it shows:** an independent second fleet — the 2020 trace covers roughly 6,500 GPUs over two months of production MLaaS traffic; the 2023 trace covers 1,523 nodes and 8,152 tasks with **fractional GPU requests recorded as `gpu_milli`** for single-GPU workloads and roughly a third of GPU tasks carrying a specific GPU-type constraint. The `gpu_milli` column is the interesting part for this lesson: it is a production system that had to invent a sub-GPU allocation unit, which is regime 3's allocation side made explicit in the trace schema.

## Worked example

**The setup.** One physical H100 80GB SXM, observed for exactly one hour, DCGM at a 30 s collect interval (120 samples). Let `r` be the all-in cost of one physical GPU-hour — **keep it symbolic and substitute your own**. For arithmetic below, `r = $3.00/GPU-hour`.

> **Rate provenance, stated properly.** As of **August 2026**, published on-demand rates for a single H100 SXM span a wide band — roughly **$2–$7 per GPU-hour** depending on provider class (specialist GPU clouds at the low end, hyperscaler on-demand list at the high end), with spot and marketplace capacity quoted below that band and long commitments pulling the effective rate down further. Rates move monthly and by region. **Every figure below is `r`-parameterised for exactly this reason**; the method is the durable content, the number is a snapshot. (Vendor pricing pages were not reachable from this build environment — see References.)

### (a) Exclusive whole-GPU

Pod `train-A` holds the device for the hour. `SM_ACTIVE` integrates to 0.40 GPU-hours.

```
  allocated_gpu_hours = 1.0 × 1 h                  = 1.000 GPU-h
  allocated_cost      = 1.000 × r                  = $3.00
  utilised_gpu_hours  = ∫ SM_ACTIVE dt             = 0.400 GPU-h
  utilised_cost       = 0.400 × r                  = $1.20
  gap (waste ledger)  = $3.00 − $1.20              = $1.80
  attribution error   = 0
```

The 40% is a *utilisation* finding (lesson 02 owns it). Attribution is still 100% of $3.00 to `train-A`, and no amount of low utilisation changes that.

### (b) MIG-partitioned, on the two bases

Partition the card `3g.40gb` (tenant-X) + `3g.40gb` (tenant-Y) + `1g.10gb` (tenant-Z). Compute slices: 3 + 3 + 1 = 7, fully tiled. Take the A100-shaped fractions as the worked model (`1g` = 1/7 of compute; run `nvidia-smi mig -lgip` for your exact SKU's memory figures):

```
  COMPUTE (SM) BASIS — compute slices tile exactly, so shares sum to 1.000
    X: 3/7 = 0.4286  →  0.4286 × r = $1.286
    Y: 3/7 = 0.4286  →                $1.286
    Z: 1/7 = 0.1429  →                $0.429
    ─────────────────────────────────────────
    Σ = 1.0000                        $3.000    ✓ reconciles to the card

  MEMORY BASIS — memory does NOT tile; the remainder is real
    X: 40/80 = 0.500 →                $1.500
    Y: 40/80 = 0.500 →                $1.500
    Z: 10/80 = 0.125 →                $0.375
    ─────────────────────────────────────────
    Σ = 1.125  ✗  OVER 1.0 — the nominal profile sizes do not sum to the
                  usable framebuffer, because profile names are rounded
                  and each instance carries reserved capacity.

    Use MEASURED framebuffer, not the profile name. On the KEP's
    A100-40GB numbers the same exercise gives 7 × 4,864 / 40,960 = 0.831
    for a 7×1g layout, leaving 0.169 unallocatable.

  ⇒ THE RULE: compute the memory basis from `nvidia-smi mig -lgip`
    output, never from the digits in the profile name, and BOOK THE
    REMAINDER to platform overhead so the shares sum to exactly 1.0.
```

Tenant Z's bill differs by 0.1429 vs ~0.125 — a **14% relative swing on the same slice** from the basis choice alone. Now scale it: 200 such slices across a fleet at `r = $3.00` for a month (730 h) is `200 × 0.018 × 3.00 × 730 ≈ $7,900/month` of money moving between cost centres because of an undocumented modelling choice. That is why the basis is a written, defended decision.

### (c) Time-sliced, four pods, three splits

`timeSlicing.replicas: 4`; pods P1–P4 each hold one replica for the hour. DCGM reports device `SM_ACTIVE` integrating to **0.55 GPU-hours** — the only work number that exists. Per-PID NVML sampling (rung 3 of module 04's ladder) reports context-resident fractions of P1 0.70, P2 0.25, P3 0.025, P4 0.025.

```
  ALLOCATED LEDGER — identical under all three splits, and EXACT:
    each pod holds 1 of 4 replicas → allocated_share = 0.25
    allocated_cost(each) = 0.25 × 1 h × r = $0.75      Σ = $3.00 ✓
    (Divide by REPLICAS, not by holder count. If only 3 pods held
     replicas, the 4th replica's 0.25 is unsold platform capacity —
     socialising it onto the three tenants converts a platform
     over-provisioning error into a tenant cost they cannot act on.)

  UTILISED LEDGER — three candidate splits of the SAME 0.55 GPU-h:

    split            P1        P2       P3        P4       Σ
    ───────────────────────────────────────────────────────────
    equal (U/N)     0.1375    0.1375   0.1375    0.1375   0.550
      × r           $0.413    $0.413   $0.413    $0.413   $1.650
    ───────────────────────────────────────────────────────────
    per-PID NVML    0.3850    0.1375   0.0138    0.0138   0.550
      × r           $1.155    $0.413   $0.041    $0.041   $1.650
    ───────────────────────────────────────────────────────────
    request-wt      0.1375    0.1375   0.1375    0.1375   0.550
      (identical to equal split: all four hold one replica each)
    ───────────────────────────────────────────────────────────

  READ THE COLUMNS:
    · Every split totals EXACTLY $1.650. The device total is right.
    · P3's utilised bill is $0.413 or $0.041 — a 10× spread — for the
      same hour, on the same hardware, with the same data.
    · P1's is $0.413 or $1.155 — 2.8×.

  THE DISCLOSED ERROR BAND, computed rather than asserted:
    err_i = U × (1/N − s_i)      with U = 0.55, N = 4
    P1: 0.55 × (0.25 − 0.700) = −0.2475 GPU-h  → −$0.743  (under-billed)
    P2: 0.55 × (0.25 − 0.250) =  0.0000        →  $0.000
    P3: 0.55 × (0.25 − 0.025) = +0.1238        → +$0.371  (over-billed)
    P4: same as P3            = +0.1238        → +$0.371
    ─────────────────────────────────────────────────────
    Σ err                     =  0.0000        →  $0.000   ✓ redistributive
    max |err_i| ≤ U = 0.55 GPU-h = $1.65 (the whole device's utilised cost)
```

**The one-sentence interview answer:** *"You can't do it exactly — the device gives one counter for four contexts, so you have one equation and four unknowns. I'd bill the allocated ledger, which is exact at 0.25 of the card each; for the utilised ledger I'd use per-PID NVML weighting as the least-wrong split, publish the redistributive error band — the device total is exact, up to 0.55 GPU-hours can be on the wrong invoice — and if the money matters, move these tenants to MIG, which restores per-tenant counters."*

### (d) The same hour under DRA

Now the four pods hold `ResourceClaim`s instead of counted devices.

```
  If each claim resolved to its own MIG instance:
      → regime 2. Per-instance DCGM counters exist. Exact split.
      → gain vs (b): the claim RECORDS the granted device and the
        attributes it was selected on, so the allocation side is
        auditable from an API object instead of inferred.

  If the four claims resolved to a SHARED allocation of one device:
      → regime 3. Same 0.55 GPU-h scalar. Same four unknowns.
      → gain vs (c): exactly the allocation-side provenance, nothing
        at rung 4. The utilised split is still an estimate with the
        same error band.

  ⇒ DRA improved the NUMERATOR'S PROVENANCE.
    It did not improve the numerator.
```

### (e) Roll it up — the fleet exposure number

Suppose the fleet is 64 physical GPUs: 40 exclusive-or-MIG at mean utilisation 0.62, and 24 time-sliced at mean utilisation 0.91 (shared devices run hotter — that is the point of sharing).

```
  exact utilised GPU-equivalents      = 40 × 0.62 = 24.80
  estimated utilised GPU-equivalents  = 24 × 0.91 = 21.84
                                                    ─────
  total                                             46.64

  exposure  E = 21.84 / 46.64 = 0.468  →  46.8 %

  At r = $3.00 and 730 h/month, the exposed slice is
      21.84 × $3.00 × 730 = $47,830 / month
  of utilisation-based chargeback resting on a fair-share estimate.

  If per-PID NVML sampling works on those 24 GPUs, exposure drops
  toward ~0 and the same $47,830 becomes a per-pid-labelled number.
  THAT is the business case for hostPID and an NVML support probe,
  stated in dollars instead of engineering preference.
```

Note the shape of the argument, because it is the shape that survives a hostile room: you are not claiming the total is wrong. You are quantifying what fraction of the *split* is an estimate and offering a specific change that reduces it.

## Practice

Feed these into the module deliverable at [gpu-cost synthesis](../practice/gpu-cost-synthesis/README.md).

1. **Build the regime decision table.** For your fleet's real (or a plausible) tenancy mix, tag every workload with its regime — read `nvidia.com/gpu.sharing-strategy`, `nvidia.com/mig.strategy` and the resource names actually requested, do not guess from device-ID shape. For each row record: the resource name, the regime, the exact allocation formula, the exact utilisation formula, the measurement identity, and whether the utilisation figure is `exact` / `bounded-estimate` / `estimate`.
   **Acceptance:** every workload has a regime, and no row claims `exact` on a device with more than one holder.

2. **Defend a MIG basis in writing.** Pick compute, memory, or a documented blend. One paragraph naming the binding resource for that fleet, what breaks under the other basis, and the relative gap between the two computed from *your* `nvidia-smi mig -lgip` output — not from the profile names. Include the treatment of the unallocatable remainder and where you book it.
   **Acceptance:** the paragraph contains a number for the basis gap and names a cost centre for the remainder.

3. **Compute a time-slice error band.** Take a 3–4 pod time-slicing scenario (real or synthesised from the Alibaba trace's `gpu_milli` shapes). Bill it three ways — equal, request-weighted, per-PID-weighted — tabulate per-pod dollars, then compute `err_i = U × (1/N − s_i)` for each and confirm `Σ err_i = 0`. Report the min/max per-pod spread as the disclosed uncertainty.
   **Acceptance:** the three splits' totals are identical to the cent, and the error column sums to zero.

4. **Compute the fleet exposure fraction `E`.** Using your regime table plus measured mean utilisation per regime class. Write the one-sentence version you would say in a budget review.
   **Acceptance:** `E` is a number between 0 and 1 with the inputs shown, and the sentence names both the exactness of the total and the size of the exposed split.

5. **DRA mapping.** Sketch a `ResourceClaim` for (a) one MIG allocation and (b) one shared allocation, and state for each which utilisation regime the cost attribution inherits and what `ResourceClaim.status.allocation` adds that a device-plugin count did not.
   **Acceptance:** (b) explicitly says "regime 3, same error band as time-slicing."

## Common pitfalls

1. **"OpenCost shows a GPU cost, so attribution is solved."** *Mechanism:* OpenCost's `GPUHours` is `GPU request × hours` and `GPUCost` is `GPUHours × CostPerGPUHr`. That is the allocated ledger computed under regime-1 assumptions and applied uniformly, including to devices its own code has already flagged as shared. It is a correct number for exclusive GPUs and a category error everywhere else. (Lesson 09 reads the code.)

2. **Assuming time-sliced pods all receive the same device ID.** *Mechanism:* they do not — the plugin fabricates annotated IDs `GPU-<uuid>::<replica>`, so pod-resources hands you a distinct string per pod. Believing the folklore version costs you the one thing you actually have under regime 3: an exact, measured denominator for the fair-share split. What collapses is one layer down, at the counter, not at the allocation record.

3. **`map[deviceID]Owner` where it should be `map[deviceID][]Owner`.** *Mechanism:* a shared join key has several holders; a scalar map silently keeps whichever was written last, so one pod is charged for the whole card and the rest for nothing. Symptom: per-GPU allocated shares summing to less than 1.0 with no unallocated bucket to account for the difference. This is the highest-impact single bug in the build, and identity A catches it immediately.

4. **Summing a fanned-out device metric across holders.** *Mechanism:* `dcgm-exporter` labels the same device-level `SM_ACTIVE` value with every holding pod, so `sum by (namespace)` over a time-sliced pool reports N× the real work. Diagnostic signature: `Σ shares ÷ device utilisation` equals the holder count **exactly**, independent of load. Deduplicate on the measurement identity before summing.

5. **Treating SM-slice fraction as the universally correct MIG basis.** *Mechanism:* MIG deliberately decouples compute and memory — 7 compute slices, 8 memory slices — so the two bases give different answers by construction (~20% relative on a `1g` profile). Compute is the right default for training; a memory-bound serving fleet where VRAM caps tenancy should bill on memory. There is no basis-free answer, only a documented one.

6. **Costing MIG instances as whole GPUs.** *Mechanism:* the resource name `nvidia.com/mig-1g.10gb` counts *instances*, and seven of them live on one card. Multiplying instance count by the whole-GPU rate reports 7× the fleet's spend. Same failure mode as dividing the node rate by `nvidia.com/gpu.count` under a MIG `single` strategy — divide by the physical count from NVML, always.

7. **Letting the MIG remainder vanish.** *Mechanism:* at a 7×1g geometry the memory shares sum to ~0.83, not 1.0. If you do not book the missing ~0.17 to platform overhead, per-GPU shares do not reconcile, and the identity check fails with a symptom (sum < 1) that looks identical to an unresolved device ID. Book it explicitly, with a `reason` label, so the two failure modes are distinguishable.

8. **"DRA solves GPU cost attribution."** *Mechanism:* DRA improves the allocation record — you read the granted device out of `ResourceClaim.status.allocation` instead of inferring it. Rung 4 is untouched. A DRA claim over a shared device is regime 3, counters and error band included. Related trap: `dcgm-exporter`'s `--kubernetes-enable-dra` defaults to `false`, so on a DRA cluster the default configuration reports those GPUs with no pod labels at all.

9. **Conflating MPS with plain time-slicing.** *Mechanism:* both produce `::N` device IDs, so they look identical in pod-resources. They are not: MPS funnels clients through one context and caps each client's SM share via `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE`, which converts an unbounded split error into one bounded by the cap. Read `nvidia.com/gpu.sharing-strategy` from the node; never infer the mode from a device ID.

10. **Reporting an approximate split without reporting the exposure.** *Mechanism:* the redistributive-error result makes an estimate defensible *only if you say it is one*. A single unlabelled number invites the challenge "is this right?", to which the only answer is "partly" — which sounds like an admission. The labelled version invites "how much of it is estimated?", to which the answer is a number and a remediation plan.

## Self-check

- **Why can DCGM attribute utilisation exactly under MIG but not under software time-slicing?**
  **Answer:** Because the measurement identity differs. MIG is a hardware partition: each GPU instance gets its own slice of the performance-monitoring registers, `dcgm-exporter` emits per-instance series (parent `UUID` plus `GPU_I_ID`/`GPU_I_PROFILE`), and an instance belongs to exactly one container — so every counter sample has exactly one owner. Under time-slicing all sharers run CUDA contexts on the same physical device; the SM counters increment on warp-cycles with no context tag anywhere in the counter path, so DCGM publishes one scalar `U(t)` for N contexts. That is one equation in N unknowns: for N ≥ 2 infinitely many per-tenant vectors produce the same reading, so the mapping is many-to-one and non-invertible. It is the counter's scope, not a gap in the exporter. The qualification that matters: it is unrecoverable *from DCGM device counters*; NVML's per-PID sampling interface is a different, coarser path that recovers an estimate, and `dcgm-exporter` uses it for `GPU_UTIL` and `FB_USED` only when `--kubernetes-virtual-gpus` is on — never for the `PROF` fields.

- **A `3g.40gb` MIG instance — is its cost fraction 3/7 or 40/80, and how do you decide?**
  **Answer:** Neither is automatically correct. 3/7 = 0.4286 is the compute-slice share (on an A100's 98 MIG-addressable multiprocessors, 3 slices = 42 MP); 40/80 = 0.5 is the nominal memory share. MIG deliberately decouples the axes — 7 compute slices, 8 memory slices — so they disagree by construction, up to roughly 20% relative on a `1g` profile. You decide by naming the binding resource for that fleet: compute-bound training → compute basis; memory-bound serving where VRAM caps replica count → memory basis; otherwise a documented blend `w·compute + (1−w)·memory` with `w` written down. Three obligations follow: emit the basis as a label on the metric, compute the memory share from measured framebuffer (`nvidia-smi mig -lgip`) rather than the digits in the profile name, and book the unallocatable remainder — 16.9% of the framebuffer at a 7×1g geometry on A100-40GB — to platform overhead so per-GPU shares still sum to exactly 1.0.

- **How do you bill four pods time-slicing one A100, and why is every answer an approximation?**
  **Answer:** Bill the **allocated** ledger exactly: each pod holds 1 of `replicas` replicas, so `allocated_share = 1/replicas` — divide by the configured replica count, not by the number of pods currently holding one, because unsold replicas are platform capacity, not a tenant cost. For the **utilised** ledger you must split one device-scoped scalar `U` among N contexts, which is underdetermined. The menu: equal split (`U/N`, error unbounded per pod), request-weighted (identical unless holders differ in replica count; bills intent), per-PID NVML weighting (`U × smUtil_i / Σ smUtil` — best available, error from coarse sampling and missed short kernels, unavailable on some SKUs), and MPS with a thread-percentage cap (error bounded by the cap). Whatever you pick, publish the band: `err_i = U × (1/N − s_i)`, which sums to exactly zero across holders — the device total is exact and only the split is uncertain — with `|err_i| ≤ U`. Then report the fleet exposure fraction `E = G_s·ū_s / (G_x·ū_x + G_s·ū_s)` so the estimated share of the chargeback is a stated number rather than a hidden one.

- **Does DRA solve GPU cost attribution?**
  **Answer:** Only the allocation half. DRA (core GA in Kubernetes v1.34, gate locked on in v1.35) replaces an opaque device count with a `ResourceClaim` whose `status.allocation` records exactly which device was granted and which attributes it was selected on, published against driver-authored `ResourceSlice`s and admin-authored `DeviceClass`es; v1.34 also extended the kubelet pod-resources API to report DRA-allocated devices, so node agents can consume it. Partitionable devices (KEP-4815, alpha v1.35 / beta v1.36) add a `CounterSet` per physical device and `ConsumesCounters` per partition, letting the scheduler refuse overlapping MIG geometries by counter exhaustion. All of that improves *provenance* and *fragmentation*. None of it touches rung 4: a claim resolving to a whole GPU is regime 1, to a MIG instance regime 2, to a shared device regime 3 with the identical one-counter-N-contexts problem. Operational footnote: `dcgm-exporter`'s `--kubernetes-enable-dra` defaults to `false`, so on a DRA cluster the default configuration shows those GPUs with no pod labels and quietly understates tenant allocation.

- **Why is the fair-share error "redistributive," and why does that make an approximate bill defensible?**
  **Answer:** For a device at utilisation `U` shared by `N` holders with true shares `s_i` (Σ`s_i` = 1), a fair-share estimator assigns `U/N` and errs by `err_i = U(1/N − s_i)`. Summing: `Σ err_i = U(Σ1/N − Σs_i) = U(1 − 1) = 0`. Every dollar is accounted for; some are on the wrong invoice. That means you can make two separate, differently-strong claims: the fleet and per-device totals are **exact**, and a quantified fraction of the per-tenant *split* is an estimate bounded by `|err_i| ≤ U`. Quantify that fraction as the exposure `E` and you have a defensible position in a chargeback dispute plus a costed remediation ("per-PID sampling on these 24 GPUs moves $47,830/month from estimate to measurement"). Emitting a single unlabelled number forfeits both claims at once.

- **Why does a fleet operator accept MIG's attribution overhead instead of time-slicing everywhere?**
  **Answer:** Because MIG buys back the thing time-slicing gives up — a per-tenant hardware counter — while keeping most of the density benefit. Sharing exists because a whole H100 per small-model replica wastes most of the card; MIG delivers a large share of that consolidation benefit while keeping every tenant on its own measurement identity, so the utilised ledger stays exact and the bill survives a dispute. Time-slicing may pack marginally denser and is genuinely the right call for workloads whose spend nobody will ever challenge (internal batch, best-effort), but every dollar attributed off it carries the `|err_i| ≤ U` band. The decision rule is not technical purity: it is *does this tenant's bill need to be defensible to the dollar?* If yes, exclusive or MIG. If no, share and publish the exposure.

## Connections & what's next

This lesson is the spine the rest of module 11 hangs off. Lesson 02 takes the two ledgers introduced here — allocation cost from rungs 1–3, utilisation cost from rung 4 — and turns them into the module's core accounting model, with the corrected `sum_over_time` integral carried forward from module 05. Lesson 03's idle taxonomy is regime-aware: "idle" on a MIG instance is a measured fact, "idle" on a time-sliced replica is an estimate with the band computed here. Lesson 04's fragmentation math consumes the unallocatable MIG remainder derived in §4, and picks up DRA's partitionable-device counters as the structural fix. Lesson 05's unit economics divides by `H_attr`, whose error bars are exactly the exposure fraction `E`. Lesson 09's teardown is the claim that OpenCost implements regime 1 and applies it universally — now checkable against the source lines named above. And lesson 10's FOCUS schema needs an attribution-regime dimension precisely because this taxonomy exists and FOCUS 1.3's split-cost-allocation columns (`AllocatedMethodId`, `AllocatedMethodDetails`, `AllocatedRatio`) give you the standard place to record which split you used.

Next: **lesson 02** formalises the two-ledger model — allocated versus utilised, always reported together, with the gap as a first-class number in dollars.

## References & further reading

**Primary sources**

1. **Kubernetes — KEP-4381, DRA structured parameters** — https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4381-dra-structured-parameters — the `ResourceSlice` / `ResourceClaim` / `DeviceClass` model, CEL selectors over device attributes, and the explicit note that the base KEP covers only *static* pre-partitioned devices, with dynamic partitioning deferred to KEP-4815. Verified against the KEP README in the `kubernetes/enhancements` repository.
2. **Kubernetes — KEP-4815, DRA partitionable devices** — https://github.com/kubernetes/enhancements/issues/4815 (`keps/sig-scheduling/4815-dra-partitionable-devices`) — `CounterSet` / `SharedCounters` / `ConsumesCounters`, and the A100 example this lesson's MIG stranding arithmetic is computed from (`memory: 40Gi`, `multiprocessors: 98`, `copy-engines: 7`, eight memory slices; `1g.5gb` = 4,864 MiB + 14 MP + 1 copy engine + 1 slice). Alpha in Kubernetes v1.35, beta in v1.36.
3. **Kubernetes v1.34 — "DRA has graduated to GA"** — https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/ — DRA core to GA, gate locked on in v1.35, and the kubelet pod-resources API extended to report DRA-allocated devices. (kubernetes.io was unreachable from this build environment; the GA status and version numbers were cross-checked against the KEP metadata in the `kubernetes/enhancements` repository.)
4. **NVIDIA `k8s-device-plugin` — `api/config/v1/sharing.go`, `api/config/v1/replicas.go`** — https://github.com/NVIDIA/k8s-device-plugin — the real config API for regime 3: `Sharing{TimeSlicing, MPS}`, `SharingStrategy()` returning `mps` / `time-slicing` / `none`, and `ReplicatedResources{RenameByDefault, FailRequestsGreaterThanOne, Resources[]{Name, Rename, Devices, Replicas}}`. Read from the repository source rather than the rendered docs.
5. **NVIDIA — DCGM field-ID reference** — https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-api/dcgm-api-field-ids.html — the definitions of field 203 (`GPU_UTIL`), 1002 (`SM_ACTIVE`), 1004 (`PIPE_TENSOR_ACTIVE`) and the MIG-instance fields, i.e. the counter scope the regime-3 argument rests on. (docs.nvidia.com was unreachable from this build environment; the field semantics used here are the ones established and verified in module 05.)
6. **NVIDIA — MIG User Guide** — https://docs.nvidia.com/datacenter/tesla/mig-user-guide/ — profile tables, per-instance device UUIDs, placement rules. (Unreachable from this build environment; the A100 geometry numbers used in this lesson come from KEP-4815's NVIDIA-authored counter model instead, and readers should confirm their own SKU with `nvidia-smi mig -lgip`.)
7. **NVIDIA — Multi-Process Service (MPS) documentation** — https://docs.nvidia.com/deploy/mps/index.html — the control daemon, client contexts, and `CUDA_MPS_ACTIVE_THREAD_PERCENTAGE`, which is the mechanism that turns regime 3b's error from unbounded into capped.
8. **Kubernetes — pod-resources API / third-party device metrics** — https://kubernetes.io/blog/2020/12/16/third-party-device-metrics-reaches-ga/ — the kubelet API that makes rung 3 of the ladder exact in every regime.

**Reading the tools' source**

9. **OpenCost — `pkg/costmodel/allocation_helpers.go`, `core/pkg/opencost/allocation.go`** — https://github.com/opencost/opencost — `applyGPUsAllocated` (`GPUHours = request × hours`), `alloc.GPUCost = alloc.GPUHours * node.CostPerGPUHr`, and the `GPUAllocation` struct carrying `GPURequestAverage`, `GPUUsageAverage` and `IsGPUShared`. Read at the `develop` branch; the exact line numbers move, the shape does not.
10. **NVIDIA `dcgm-exporter`** — https://github.com/NVIDIA/dcgm-exporter — `--kubernetes` (default `false`), `--kubernetes-enable-dra` (default `false`), `--kubernetes-virtual-gpus` (default `false`), and the per-process split applied to `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_DEV_FB_USED` only.

**Real-world fleets**

11. **Jeon et al., "Analysis of Large-Scale Multi-Tenant GPU Clusters for DNN Training Workloads" (USENIX ATC'19)** — https://www.usenix.org/conference/atc19/presentation/jeon — the Philly trace paper: 117,325 jobs, Aug–Dec 2017, a real production fleet treating GPUs as coarse monolithic scheduling units.
12. **Microsoft Research — Philly trace dataset** — https://github.com/msr-fiddle/philly-traces — the public data, with per-minute GPU/CPU/memory utilisation, for building your own regime-mix analysis.
13. **Alibaba — PAI GPU cluster trace 2020** — https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2020 — ~6,500 GPUs across ~1,800 machines, two months of production MLaaS traffic.
14. **Alibaba — PAI GPU cluster trace 2023** — https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2023 — 1,523 nodes, 8,152 tasks, heterogeneous GPU types, fractional single-GPU requests recorded as `gpu_milli`, and ~33% of GPU tasks carrying GPU-type constraints. The cleanest public example of a production system that needed a sub-GPU allocation unit — regime 3's allocation side, in a trace schema.

---
Module backlink: [💰 11 — GPU cost and unit economics](../README.md)
