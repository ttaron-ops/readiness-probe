---
lesson: "04.6"
title: "MIG Operations & Per-Slice Attribution"
module: "04"
concept: "MIG operations & per-slice attribution"
status: not-started
est_time: "10h"
prev: "05-driver-lifecycle-upgrades.md"
next: "07-time-slicing-attribution.md"
artifacts: []
sources: 11
---

# 04.6 · MIG Operations & Per-Slice Attribution

> **Concept.** On Kubernetes, MIG is a resource-surfacing and drain-to-reconfigure problem — each slice becomes a distinct schedulable resource with its own UUID, which is what gives you clean per-slice cost attribution.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)
>
> **Advanced Learning** — [MIG Geometry](../../../learning/06-mig-operations.html): the 7-compute/8-memory asymmetry as slice strips, with the stranded framebuffer priced. Optional visual companion; this lesson stays the source of truth.

## Where this fits

Lesson 04.5 established the pattern the GPU Operator uses for any disruptive per-node change: pause the dependent operands, mutate, validate, restore. There it was applied to swapping a kernel module. This lesson applies the same shape one layer lower — instead of tearing down a loaded module, you tear down and recreate **GPU Instances and Compute Instances**, the hardware partition objects a MIG-enabled GPU exposes.

It is a narrower operation than a driver upgrade (one GPU, not a fleet) and a much sharper one, because the partition layout is not just a performance choice. It is the *shape of your billable unit*. Get `mig.strategy` wrong and you do not just risk downtime — you silently delete the attribution story this module is built toward, because the resource name stops carrying the profile.

This is also the first lesson where "attribution" stops being aspirational. MIG is the one sharing mode where the Kubernetes-visible resource model and the hardware partition boundary coincide, which sets the baseline the next two lessons — time-slicing (04.7) and MPS (04.8) — are measured against and fail to meet.

## Why this matters

Module 03 already owns what MIG *is* at the silicon level: a GPU carved into isolated instances, each with a fenced set of SMs, a fenced L2 slice, and a fenced path to a fenced region of framebuffer. That isolation is the whole point for a platform team, and it is operationally worthless until Kubernetes can *see* the slices and *attribute* them. That is a Kubernetes-and-operator problem, and it is what this lesson solves.

Two of this module's named interview probes land here directly: *"MIG vs time-slicing vs MPS — mechanism, isolation, when"* and *"why must you drain a node before reconfiguring MIG?"* The second one is a trap in a specific way. The expected answer — "because you can't destroy a GPU Instance while a process holds a context on it" — is correct and incomplete, and the follow-up is where candidates fall over: **who does the draining?** The honest answer, which you can verify in `nvidia-mig-manager`'s own source and which contradicts a lot of published material including the previous version of this lesson, is: *nobody does, unless you do it.* The Operator's MIG manager pauses its own operands and waits for them; it does not cordon the node and it does not evict your workloads. Knowing that is the difference between an engineer who has read the docs and one who has run the reconfiguration.

The cost side is the other half. The competing way to share a GPU is **time-slicing** (lesson 04.7), where N pods each get injected the same physical GPU and DCGM measures one device for all of them. When finance asks "which team burned that H100 last month," time-slicing has no answer — every pod's series carries the same value. MIG closes the hole: each instance is a distinct hardware object, gets its own resource name (`nvidia.com/mig-1g.10gb`), its own device UUID, and its own DCGM entity. Time-slicing gives you density; MIG gives you density *plus* a defensible invoice.

And MIG is not free. A partitioned GPU is measurably smaller than an unpartitioned one — before you have run a single kernel. §4 puts real numbers on that, and "MIG costs you 15 % of the die and up to 16 % of the framebuffer depending on the geometry you pick" is a much more senior answer than "MIG partitions the GPU."

## What's new here (calibration)

Module 03 owns MIG's hardware theory and module 02 owns the device-plugin gRPC mechanics; lesson 04.3 already re-established the device-plugin contract and the pod-resources API in the depth this module needs. None of that is re-derived. What is genuinely new here:

- **The placement geometry, exactly.** Seven compute slices, eight memory slices, and a placement axis measured in memory slices — from the NVML struct comments, not from an analogy. This is what determines which profile *combinations* physically fit, and it is the fact that makes MIG capacity planning tractable instead of trial-and-error.
- **The complete profile tables for every SKU you are likely to touch**, with slice counts, instance counts, SM counts, engine counts and real (not nominal) framebuffer sizes, read out of NVIDIA's own profile definitions.
- **Worked partitioning-efficiency math** — how much of an 80 GB card is actually usable under each geometry, where the stranded remainder goes, and which mixed geometries have zero waste.
- **`nvidia-smi mig` end to end**: the exact create/destroy sequence, what each subcommand does to the hardware, and what the outputs mean.
- **`mig-manager`'s real reconfiguration state machine**, step by step from its Go source — including the parts that surprise people: it auto-generates a per-node config from discovered hardware, it pauses six operand DaemonSets by label, it waits five minutes per operand, and it does **not** touch your workloads.
- **Why reconfiguration cannot be live, mechanically** — the NVML contract that says the driver may unbind or reset the device, and the exact error path when it cannot.
- **The MIG attribution join in full**, including the NVML hop that lesson 04.3 flagged and that most homegrown exporters miss. DCGM does not label MIG metrics with the MIG UUID.
- **The hard constraint that MPS and MIG do not combine** — documented, not hypothetical.

## Core concepts

### 1 — Two objects, not one: GPU Instances and Compute Instances

A MIG-partitioned GPU exposes a two-level hierarchy, and almost every confusing thing about MIG follows from people collapsing it into one level.

A **GPU Instance (GI)** is the memory-and-engine partition. Creating one fences off a set of memory slices — which means a fenced region of framebuffer, a fenced portion of L2, and fenced memory-controller paths — plus a set of copy engines, decoders, JPEG and OFA engines. This is the isolation boundary. Two GIs cannot see each other's memory and cannot contend for each other's bandwidth.

A **Compute Instance (CI)** lives *inside* a GI and partitions its SMs. A GI's compute slices can be handed to one CI (the usual case) or subdivided. CIs inside the same GI **share** the GI's memory and engines — the NVML enum name says it outright, `NVML_COMPUTE_INSTANCE_ENGINE_PROFILE_SHARED`: "All the engines except multiprocessors would be shared."

```
   MIG OBJECT HIERARCHY ON ONE PHYSICAL GPU
   (H100 80GB, geometry: 1× 3g.40gb + 1× 4g.40gb)

   ┌─────────────────────────────────────────────────────────────────┐
   │ PHYSICAL GPU        UUID: GPU-8f2c1d90-3e5a-4b17-9c2e-5da9…     │
   │ MIG mode: Enabled   (an InfoROM setting; changing it may        │
   │                      unbind/reset the device — §6)              │
   │                                                                 │
   │  ┌───────────────────────────┐  ┌────────────────────────────┐  │
   │  │ GPU INSTANCE  id=1        │  │ GPU INSTANCE  id=5         │  │
   │  │ profile 3g.40gb           │  │ profile 4g.40gb            │  │
   │  │ compute slices: 3 (48 SM) │  │ compute slices: 4 (64 SM)  │  │
   │  │ memory slices:  4 (40 GB) │  │ memory slices:  4 (40 GB)  │  │
   │  │ CE 3 · DEC 2              │  │ CE 4 · DEC 2               │  │
   │  │  ┌──────────────────────┐ │  │  ┌──────────────────────┐  │  │
   │  │  │ COMPUTE INSTANCE id=0│ │  │  │ COMPUTE INSTANCE id=0│  │  │
   │  │  │ profile 3c.3g.40gb   │ │  │  │ profile 4c.4g.40gb   │  │  │
   │  │  │ 48 SM                │ │  │  │ 64 SM                │  │  │
   │  │  └──────────────────────┘ │  │  └──────────────────────┘  │  │
   │  │                           │  │                            │  │
   │  │  MIG DEVICE (what k8s     │  │  MIG DEVICE                │  │
   │  │   and CUDA actually see)  │  │                            │  │
   │  │  = (GI, CI) pair          │  │  = (GI, CI) pair           │  │
   │  │  UUID: MIG-b6ba6b2a-…     │  │  UUID: MIG-4e91cc07-…      │  │
   │  └───────────────────────────┘  └────────────────────────────┘  │
   │                                                                 │
   │  ISOLATION: across GIs — memory, L2, bandwidth, faults.         │
   │             across CIs in one GI — SMs only. Memory is SHARED.  │
   └─────────────────────────────────────────────────────────────────┘

   The thing you schedule, the thing CUDA opens, and the thing DCGM
   measures is the MIG DEVICE — a (GI, CI) pair. Its UUID is what
   pod-resources reports. §8.
```

The device plugin only ever creates a **single CI spanning the whole GI** — `nvidia-mig-parted apply` drives `-cgi … -C`, which is "create the GI and its default full-width CI". Sub-partitioning a GI into multiple CIs is a bare-metal capability that the Kubernetes path does not expose, because a CI is not independently schedulable as a distinct extended resource. When you see `1g.10gb` in Kubernetes, that is one GI with one CI covering it.

Two names you will see in `nvidia-smi` output and should be able to read:

- `3g.40gb` — a **GPU Instance profile**: 3 compute slices, 40 GB.
- `3c.3g.40gb` — a **Compute Instance profile**: 3 compute slices inside a `3g.40gb` GI.

### 2 — The placement geometry: 7 compute slices, 8 memory slices

This is the single most useful mechanical fact in the lesson, and it is stated unambiguously in `nvml.h`:

```c
typedef struct nvmlGpuInstancePlacement_st {
    unsigned int start;   //!< Index of first occupied MEMORY slice
    unsigned int size;    //!< Number of MEMORY slices occupied
} nvmlGpuInstancePlacement_t;

typedef struct nvmlComputeInstancePlacement_st {
    unsigned int start;   //!< Index of first occupied COMPUTE slice
    unsigned int size;    //!< Number of COMPUTE slices occupied
} nvmlComputeInstancePlacement_t;
```

A GI's placement is expressed on the **memory-slice axis**; a CI's placement is expressed on the **compute-slice axis** within its GI. On Ampere, Hopper and Blackwell datacenter parts that axis is **8 memory slices wide**, while there are only **7 compute slices**. That asymmetry is the source of every stranded-capacity surprise in MIG.

The valid placements are not "anywhere it fits" — they are a fixed table per profile, and NVIDIA's own profile definitions (`NVIDIA/go-nvml`, `pkg/nvml/mock/gpus/a100.go`, which mirrors what `nvmlDeviceGetGpuInstancePossiblePlacements` returns on real silicon) give them exactly:

| Profile (A100 family) | compute slices | memory slices (`size`) | valid `start` positions |
|---|---|---|---|
| `1g.5gb` / `1g.10gb`* | 1 | 1 | 0, 1, 2, 3, 4, 5, 6 |
| `1g.5gb+me` | 1 | 1 | 0, 1, 2, 3, 4, 5, 6 |
| `1g.10gb` / `1g.20gb`† | 1 | **2** | 0, 2, 4, 6 |
| `2g.10gb` / `2g.20gb` | 2 | 2 | 0, 2, 4 |
| `3g.20gb` / `3g.40gb` | 3 | **4** | 0, 4 |
| `4g.20gb` / `4g.40gb` | 4 | 4 | 0 |
| `7g.40gb` / `7g.80gb` | 7 | **8** | 0 |

\* the base single-slice profile — `1g.5gb` on a 40 GB card, `1g.10gb` on an 80 GB card.
† the memory-doubled single-slice profile (`NVML_GPU_INSTANCE_PROFILE_1_SLICE_REV2`) — `1g.10gb` on a 40 GB card, `1g.20gb` on an 80 GB card. Same 1 compute slice, twice the memory.

Read the last three rows carefully. A `3g` profile consumes **four** memory slices, not three — the allocator rounds up to a power-of-two-aligned block. That is why only two `3g` instances fit (slices 0–3 and 4–7) even though 7 ÷ 3 = 2 with a compute slice left over, and it is why a `3g` cannot start at position 1, 2 or 3.

Now the picture that makes profile combinations obvious:

```
  THE PLACEMENT AXIS — 8 MEMORY SLICES WIDE, 7 COMPUTE SLICES DEEP
  (A100/H100/H200/B200 datacenter parts. Each ▣ = one memory slice.)

  memory slice:   0    1    2    3    4    5    6    7
                ┌────┬────┬────┬────┬────┬────┬────┬────┐
  compute slice:│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ —  │  ← only SEVEN
                └────┴────┴────┴────┴────┴────┴────┴────┘     compute slices
                                                              exist. Slice 7
                                                              of the memory
                                                              axis has no
                                                              compute partner.

  WHERE EACH PROFILE MAY START (size in ▣):

   1g.10gb  ▣                                     start ∈ {0,1,2,3,4,5,6}
   1g.20gb  ▣▣                                    start ∈ {0,2,4,6}
   2g.20gb  ▣▣                                    start ∈ {0,2,4}
   3g.40gb  ▣▣▣▣                                  start ∈ {0,4}
   4g.40gb  ▣▣▣▣                                  start ∈ {0}      ← only ONE
   7g.80gb  ▣▣▣▣▣▣▣▣                              start ∈ {0}

  GEOMETRIES THAT EXACTLY FILL THE CARD (zero stranding):

   7g.80gb                        ████████        7C  8M   1 tenant
   4g.40gb + 3g.40gb              ████████        7C  8M   2 tenants
   2g.20gb ×2 + 3g.40gb           ██▕██▕████      7C  8M   3 tenants
   1g.10gb ×2 + 2g.20gb + 3g.40gb █▕█▕██▕████     7C  8M   4 tenants  ← "all-balanced"

  GEOMETRIES THAT STRAND SOMETHING:

   1g.10gb ×7                     █▕█▕█▕█▕█▕█▕█·  7C  7M   → 1 MEMORY slice idle
                                                            (10 GB you paid for)
   2g.20gb ×3                     ██▕██▕██··      6C  6M   → 1 compute + 2 memory
   1g.20gb ×4                     ██▕██▕██▕██     4C  8M   → 3 COMPUTE slices idle
                                                            (48 SM on an H100)
   4g.40gb ×1                     ████░░░░        4C  4M   → half the card idle
                                                            (a 4g CANNOT be paired
                                                             with another 4g: only
                                                             one valid start)

   ▕ = instance boundary   · = unusable remainder   ░ = free but unusable at 4g
```

Two conclusions fall straight out, and both are worth being able to state cold:

1. **`4g` is a trap.** There is exactly one valid placement for a 4-slice GI, so you can never have two. A node configured `all-4g.40gb` runs one tenant on half a card and wastes the other half. The only sensible use of `4g` is paired with a `3g`.
2. **Maximum density and maximum utilisation are different geometries.** `7 × 1g.10gb` maximises tenant count and strands 10 GB. `4g + 3g` wastes nothing and serves two tenants. There is no geometry that does both, and choosing between them is a business decision, not a technical one.

`nvidia-smi mig -lgipp` prints this table for the card in front of you. Run it before you design a geometry; it is the ground truth and it changes between SKUs.

### 3 — The complete profile tables

Every number below comes from NVIDIA's own GPU-instance profile definitions in `NVIDIA/go-nvml` (`pkg/nvml/mock/gpus/`), which reproduce what `nvmlDeviceGetGpuInstanceProfileInfo` returns per SKU. `SM` is the profile's `multiprocessorCount`; `FB` is `memorySizeMB`, the *real* reported framebuffer, which is below the nominal name because of ECC and driver reservations.

**A100 40 GB** (SXM4 / PCIe; 108 SMs on the die, 40960 MiB):

| Profile | slices | max instances | SM | CE | DEC | JPEG | OFA | FB (MiB) |
|---|---|---|---|---|---|---|---|---|
| `1g.5gb` | 1 | 7 | 14 | 1 | 0 | 0 | 0 | 4864 |
| `1g.5gb+me` | 1 | 1 | 14 | 1 | 1 | 1 | 1 | 4864 |
| `1g.10gb` | 1 | 4 | 14 | 1 | 1 | 0 | 0 | 9856 |
| `2g.10gb` | 2 | 3 | 28 | 2 | 1 | 0 | 0 | 9856 |
| `3g.20gb` | 3 | 2 | 42 | 3 | 2 | 0 | 0 | 19968 |
| `4g.20gb` | 4 | 1 | 56 | 4 | 2 | 0 | 0 | 19968 |
| `7g.40gb` | 7 | 1 | 98 | 7 | 5 | 1 | 1 | 40192 |

**A100 80 GB and H100 80 GB** share a geometry (A100: 108 SMs, 81920 MiB, 14 SM/slice; H100 SXM5: 132 SMs, 81920 MiB, 16 SM/slice):

| Profile | slices | max instances | SM (A100) | SM (H100) | CE | DEC | FB A100 (MiB) |
|---|---|---|---|---|---|---|---|
| `1g.10gb` | 1 | 7 | 14 | 16 | 1 | 1 | 9856 |
| `1g.10gb+me` | 1 | 1 | 14 | 16 | 1 | 1 | 9856 |
| `1g.20gb` | 1 | 4 | 14 | 16 | 1 | 1 | 19968 |
| `2g.20gb` | 2 | 3 | 28 | 32 | 2 | 1 | 19968 |
| `3g.40gb` | 3 | 2 | 42 | 48 | 3 | 2 | 40192 |
| `4g.40gb` | 4 | 1 | 56 | 64 | 4 | 2 | 40192 |
| `7g.80gb` | 7 | 1 | 98 | 112 | 7 | 5 | 80384 |

**H200 141 GB** (144384 MiB) and **B200 180 GB** (184320 MiB) keep the same 7/8 geometry with bigger slices:

| H200 profile | slices | max | SM | FB (MiB) | | B200 profile | slices | max | SM | FB (MiB) |
|---|---|---|---|---|---|---|---|---|---|---|
| `1g.18gb` | 1 | 7 | 16 | 18432 | | `1g.23gb` | 1 | 7 | 18 | 23552 |
| `1g.18gb+me` | 1 | 1 | 16 | 18432 | | `1g.23gb+me` | 1 | 1 | 18 | 23552 |
| `1g.35gb` | 1 | 4 | 16 | 35840 | | `1g.45gb` | 1 | 4 | 18 | 46080 |
| `2g.35gb` | 2 | 3 | 32 | 35840 | | `2g.45gb` | 2 | 3 | 36 | 46080 |
| `3g.71gb` | 3 | 2 | 48 | 72704 | | `3g.90gb` | 3 | 2 | 54 | 92160 |
| `4g.71gb` | 4 | 1 | 64 | 72704 | | `4g.90gb` | 4 | 1 | 72 | 92160 |
| `7g.141gb` | 7 | 1 | 112 | 144384 | | `7g.180gb` | 7 | 1 | 126 | 184320 |

Other SKUs in the same family, from the GPU Operator's own `default-mig-parted-config` ConfigMap (v26.3.3): **A30 24 GB** (`1g.6gb`×4, `2g.12gb`×2, `4g.24gb`×1 — note A30 has only 4 slices, not 7); **H100 NVL / H800 NVL** (`1g.12gb`×7, `2g.24gb`, `3g.47gb`, `4g.47gb`, `7g.94gb`); **H100 96 GB / GH200 / H20** (`1g.12gb`, `3g.48gb`, `4g.48gb`, `7g.96gb`); **GH200 144 GB HBM3e** (`1g.18gb`, `2g.36gb`, `3g.72gb`, `7g.144gb`); **B300** (`1g.34gb`×7, `1g.67gb`×4, `2g.67gb`, `3g.135gb`, `7g.269gb`); **GB300** (`1g.35gb`×7, `2g.70gb`, `3g.139gb`, `7g.278gb`); **RTX PRO 6000 96 GB** (`1g.24gb`×4 with `+gfx`, `-me` and `+me.all` variants — the graphics-capable profiles).

**The `+me` / `-me` / `+gfx` suffixes.** `me` is "media extensions": the profile carries the card's JPEG and OFA engines, and on most SKUs there is exactly **one** such engine set, so only **one** `+me` instance can exist per GPU. That is why every `all-1g.*.me` entry in the Operator's config has count `1`. `-me` (RTX PRO 6000) is the inverse — a profile with *none* of the media engines. `+gfx` marks a graphics-capable instance, which matters for rendering and VDI workloads and not for training. If your workload does video decode at scale, `+me` availability is a hard capacity constraint you must plan around; if it does not, `+me` is a profile to avoid, because taking it removes the option for anyone else.

### 4 — Worked math: what a partitioned 80 GB card is actually worth

Now put §2 and §3 together. Two independent efficiencies matter and they trade off against each other.

**Efficiency zero: turning MIG on costs you compute before you partition anything.** On an A100, the full-card `7g.40gb`/`7g.80gb` profile reports **98 SMs**. The die has **108**. On an H100 SXM5, `7g.80gb` reports **112 SMs** against **132** on the die.

```
  A100:  98 / 108 = 90.7 %  → MIG mode costs ~9.3 % of the SMs
  H100:  112 / 132 = 84.8 % → MIG mode costs ~15.2 % of the SMs
```

Framebuffer takes a smaller hit: `7g.80gb` on an A100 80 GB reports 80384 MiB of 81920, a 1.9 % reservation. **So a fully-MIG-enabled H100 running one 7g instance is already ~15 % smaller than the same card with MIG off.** If a tenant needs the whole card, do not give them a `7g` slice — turn MIG off for that GPU.

**Efficiency one: compute-slice utilisation.** How many of the 7 compute slices (98 SMs on A100-80GB) does the geometry actually hand out?

**Efficiency two: framebuffer utilisation.** How much of the 81920 MiB ends up inside an instance?

The full table for an **A100 80 GB**, using the verified per-profile FB values from §3:

| Geometry | instances | SMs handed out | of 98 | FB handed out (MiB) | of 81920 | stranded |
|---|---|---|---|---|---|---|
| `7 × 1g.10gb` | 7 | 7 × 14 = 98 | **100 %** | 7 × 9856 = 68992 | 84.2 % | 1 memory slice ≈ 12.9 GB |
| `4 × 1g.20gb` | 4 | 4 × 14 = 56 | 57.1 % | 4 × 19968 = 79872 | **97.5 %** | 3 compute slices (42 SM) |
| `3 × 2g.20gb` | 3 | 3 × 28 = 84 | 85.7 % | 3 × 19968 = 59904 | 73.1 % | 1 compute + 2 memory slices |
| `2 × 3g.40gb` | 2 | 2 × 42 = 84 | 85.7 % | 2 × 40192 = 80384 | **98.1 %** | 1 compute slice (14 SM) |
| `1 × 4g.40gb` | 1 | 56 | 57.1 % | 40192 | 49.1 % | **half the card** |
| `1 × 7g.80gb` | 1 | 98 | **100 %** | 80384 | **98.1 %** | nothing (but see efficiency zero) |
| `3g.40gb + 4g.40gb` | 2 | 42 + 56 = 98 | **100 %** | 80384 | **98.1 %** | nothing |
| `2 × 2g.20gb + 3g.40gb` | 3 | 28+28+42 = 98 | **100 %** | 80128 | 97.8 % | nothing |
| `2 × 1g.10gb + 2g.20gb + 3g.40gb` (`all-balanced`) | 4 | 14+14+28+42 = 98 | **100 %** | 79872 | 97.5 % | nothing |

Three things to take from that table:

1. **`7 × 1g.10gb` — the density-maximising geometry everyone reaches for first — strands 12.9 GB of an 80 GB card.** Seven instances consume seven of eight memory slices; the eighth has no compute partner and cannot be handed to anyone. On a $4.00/GPU-hour card that is **$0.51/hour, $373/month, per GPU**, paid for capacity that is physically unallocatable. On a 64-GPU fleet all sliced that way: about **$23,900/month** of framebuffer that exists, is billed, and cannot be sold.
2. **The zero-stranding geometries all use all 7 compute slices and all 8 memory slices**, and there are four of them at different tenant counts: 1, 2, 3 and 4. `all-balanced` is not a marketing name — it is the 4-tenant member of that family, and NVIDIA's own `values.yaml` example config uses `3g.40gb + 4g.40gb` and `2g.20gb ×2 + 3g.40gb` for the same reason.
3. **Each additional instance costs a little reserved memory.** Compare `2 × 3g.40gb` (80384 MiB) with `all-balanced` (79872 MiB): same slices, 512 MiB less usable, because per-instance overhead is charged per instance. It is small, but it means "same slices, therefore same capacity" is not quite true, and a capacity model that assumes it will drift.

**The tenant-count-vs-waste curve, stated as a decision.** If your workload mix is inference servers that fit in 10 GB, `7 × 1g.10gb` is right and the 12.9 GB is the price of density — you are selling 7 units instead of 4. If your mix is 20 GB fine-tunes, `2 × 2g.20gb + 3g.40gb` gives you three tenants and wastes nothing. The wrong move is to pick `7 × 1g` reflexively, discover your tenants need 15 GB, and end up with seven slices nobody can use.

### 5 — `nvidia-smi mig`, end to end

Everything the Operator does is this sequence, automated. Run it once by hand and the automation stops being magic.

**Step 1 — enable MIG mode.** This is a per-GPU setting persisted in the board's InfoROM, and it is the disruptive one:

```console
$ nvidia-smi -i 0 -q | grep -A2 "MIG Mode"
    MIG Mode
        Current                           : Disabled
        Pending                           : Disabled

$ sudo nvidia-smi -i 0 -mig 1
Enabled MIG Mode for GPU 00000000:1B:00.0
All done.

$ nvidia-smi -i 0 -q | grep -A2 "MIG Mode"
    MIG Mode
        Current                           : Enabled
        Pending                           : Enabled
```

The **Current / Pending** split is the mechanism. `nvmlDeviceSetMigMode` writes the *pending* mode and then attempts to activate it, and the NVML header is explicit about what activation involves:

> "This API may unbind or reset the device to activate the requested mode. Thus, the attributes associated with the device, such as minor number, might change."
> "…if device unbind fails because the device isn't idle, `NVML_ERROR_IN_USE` would be returned. The caller of this API is expected to idle the device and retry setting the mode."
> "On certain platforms like pass-through virtualization, where reset functionality may not be exposed directly, VM reboot is required. `activationStatus` would return `NVML_ERROR_RESET_REQUIRED` for such cases."

That is the whole "why you must drain" answer, from the primary source, in three sentences. If activation fails, `Current` stays `Disabled` while `Pending` reads `Enabled` — and the mode takes effect on the next reset or reboot. **A GPU showing `Current: Disabled, Pending: Enabled` is a node waiting for a reboot, and it is the exact state `mig-manager --with-reboot` exists to resolve.**

**Step 2 — list the profiles the card supports.**

```console
$ nvidia-smi mig -lgip
+-----------------------------------------------------------------------------+
| GPU instance profiles:                                                      |
| GPU   Name             ID    Instances   Memory     P2P    SM    DEC   ENC  |
|                              Free/Total   GiB              CE    JPEG  OFA  |
|=============================================================================|
|   0  MIG 1g.10gb       19     7/7        9.75       No     14     0     0   |
|                                                             1     0     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 1g.10gb+me    20     1/1        9.75       No     14     1     0   |
|                                                             1     1     1   |
+-----------------------------------------------------------------------------+
|   0  MIG 1g.20gb       15     4/4        19.50      No     14     1     0   |
|                                                             1     0     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 2g.20gb       14     3/3        19.50      No     28     1     0   |
|                                                             2     0     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 3g.40gb        9     2/2        39.25      No     42     2     0   |
|                                                             3     0     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 4g.40gb        5     1/1        39.25      No     56     2     0   |
|                                                             4     0     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 7g.80gb        0     1/1        78.50      No     98     5     0   |
|                                                             7     1     1   |
+-----------------------------------------------------------------------------+
```

*(Representative transcript for an A100-SXM4-80GB. The profile names, instance counts, SM/CE/DEC/JPEG/OFA counts and memory sizes are the verified values from §3 — 9856 MiB renders as 9.75 GiB, 40192 as 39.25, 80384 as 78.50. The **Profile ID** column is device- and driver-specific; read yours off your own card and prefer profile names in `-cgi` so nothing in your automation depends on those integers.)*

The `Instances Free/Total` column is the useful one operationally: after you create instances, `Free` drops, and the drop is *not* one-for-one across profiles because they share the placement axis. Creating one `3g.40gb` takes `3g.40gb` from 2/2 to 1/2 **and** takes `1g.10gb` from 7/7 to 3/7, because four memory slices just went away.

**Step 3 — see where instances may be placed.**

```console
$ nvidia-smi mig -lgipp
GPU  0 Profile ID 19 Placements: {0,1,2,3,4,5,6}:1
GPU  0 Profile ID 20 Placements: {0,1,2,3,4,5,6}:1
GPU  0 Profile ID 15 Placements: {0,2,4,6}:2
GPU  0 Profile ID 14 Placements: {0,2,4}:2
GPU  0 Profile ID  9 Placements: {0,4}:4
GPU  0 Profile ID  5 Placement : {0}:4
GPU  0 Profile ID  0 Placement : {0}:8
```

The `{starts}:size` notation is §2's table verbatim: valid start indices on the 8-wide memory axis, and the size consumed. This is the command that tells you whether the geometry you want is physically possible, and it is the one people skip.

**Step 4 — create GPU Instances and their Compute Instances.** `-cgi` creates GIs; `-C` also creates the default full-width CI in each. Profile names work as well as IDs:

```console
$ sudo nvidia-smi mig -cgi 3g.40gb,4g.40gb -C
Successfully created GPU instance ID  1 on GPU  0 using profile MIG 3g.40gb (ID  9)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  1 using profile MIG 3g.40gb (ID  2)
Successfully created GPU instance ID  5 on GPU  0 using profile MIG 4g.40gb (ID  5)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  5 using profile MIG 4g.40gb (ID  3)
```

Note the two different "profile ID" numbers on each pair of lines: the first is the *GPU instance* profile ID, the second is the *compute instance* profile ID from a different enum (`NVML_COMPUTE_INSTANCE_PROFILE_3_SLICE = 0x2`, `_4_SLICE = 0x3`). They are not the same numbering, which is one more reason to use names.

Without `-C` you get bare GIs and must create CIs separately with `-cci`, targeting a GI with `-gi`:

```console
$ sudo nvidia-smi mig -cgi 3g.40gb
Successfully created GPU instance ID  1 on GPU  0 using profile MIG 3g.40gb (ID  9)
$ sudo nvidia-smi mig -gi 1 -cci 3g.40gb
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  1 using profile MIG 3g.40gb (ID  2)
```

**A GI with no CI is invisible.** It consumes memory slices, it shows up in `-lgi`, and it produces no MIG device, no `/dev` node, and nothing for the device plugin to advertise. If a node's MIG capacity is lower than the geometry implies, `nvidia-smi mig -lgi` and `-lci` side by side is the first thing to check.

**Step 5 — list what exists.**

```console
$ nvidia-smi mig -lgi
+-------------------------------------------------------+
| GPU instances:                                        |
| GPU   Name          Profile  Instance   Placement     |
|                       ID       ID       Start:Size    |
|=======================================================|
|   0  MIG 3g.40gb       9        1          0:4        |
|   0  MIG 4g.40gb       5        5          4:4        |
+-------------------------------------------------------+

$ nvidia-smi mig -lci
+--------------------------------------------------------+
| Compute instances:                                     |
| GPU     GPU       Name         Profile  Instance       |
|       Instance                   ID       ID           |
|         ID                                             |
|========================================================|
|   0      1     MIG 3g.40gb        2        0           |
|   0      5     MIG 4g.40gb        3        0           |
+--------------------------------------------------------+
```

The `Placement Start:Size` column is the payoff of §2 made concrete: `0:4` and `4:4` are the two halves of the 8-slice memory axis, exactly filled.

**Step 6 — see the MIG devices, which is what Kubernetes and CUDA consume.**

```console
$ nvidia-smi -L
GPU 0: NVIDIA A100-SXM4-80GB (UUID: GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763)
  MIG 3g.40gb  Device  0: (UUID: MIG-b6ba6b2a-1cd4-5f1e-8a03-77d19e02c4b1)
  MIG 4g.40gb  Device  1: (UUID: MIG-4e91cc07-2b8a-51d3-9f76-0ac5518be7d2)
```

**Those `MIG-…` UUIDs are the strings the pod-resources API will report.** They are what lesson 04.3 §8 called the MIG row. On drivers older than R470 (470.42.01) the format is instead `MIG-<parent-GPU-UUID>/<giID>/<ciID>` — both forms exist in the field and both must be parsed by an attribution client.

**Step 7 — tear down, in the right order.** CIs first, then GIs. A GI with a live CI cannot be destroyed:

```console
$ sudo nvidia-smi mig -dci -ci 0 -gi 1
Successfully destroyed compute instance ID  0 from GPU  0 GPU instance ID  1
$ sudo nvidia-smi mig -dgi -gi 1
Successfully destroyed GPU instance ID  1 from GPU  0

# or everything at once:
$ sudo nvidia-smi mig -dci && sudo nvidia-smi mig -dgi
```

And the failure that this whole lesson is about:

```console
$ sudo nvidia-smi mig -dgi -gi 1
Unable to destroy GPU instance ID  1 from GPU  0: In use by another client
Failed to destroy GPU instances: In use by another client
```

`In use by another client` is `nvidia-smi`'s rendering of `NVML_ERROR_IN_USE` (value 19: *"An operation cannot be performed because the GPU is currently in use"*). It means some process still holds an open handle on the instance — a CUDA context, or in some cases just an open file descriptor on the MIG device node.

### 6 — Why reconfiguration cannot be live, and what actually breaks

Put the mechanism together. There are three distinct reasons a MIG change is disruptive, at three different levels, and a good answer names which one applies:

**(a) Enabling or disabling MIG mode.** `nvmlDeviceSetMigMode` may unbind and reset the device. That invalidates the device's minor number, which invalidates every open `/dev/nvidia*` handle on the node for that GPU. Any client holding one — a CUDA process, DCGM's hostengine, the device plugin's NVML handle — must be gone first, or the unbind fails with `NVML_ERROR_IN_USE`. On platforms where reset is not exposed, you get `NVML_ERROR_RESET_REQUIRED` and the only path is a reboot.

**(b) Destroying and recreating GIs/CIs on an already-MIG-enabled GPU.** No device reset is needed here, but each destroy fails with `NVML_ERROR_IN_USE` while anything holds the instance. This is the cheaper case and the one you hit on routine reprofiling.

**(c) The consequence for anything that survived.** Every existing MIG-device UUID on that card becomes invalid, and the `/dev/nvidia-caps/*` capability nodes that gate access to each GI/CI are torn down and re-created for the new geometry. A container that was injected with the old device is now holding paths that describe a partition that no longer exists. **Kubernetes does not notice.** The pod is not evicted, not restarted, and not marked unhealthy. It just stops being able to use its GPU, and — because cap-node minor numbers are allocated per instance and a new geometry re-allocates them — you do not want to be relying on the old handle simply erroring out. This is why the correct procedure evicts rather than gambling on the destroy failing safely.

**Now the part most write-ups get wrong, including the previous version of this lesson: `mig-manager` does not drain your node.** Read its `Run()` method (`NVIDIA/mig-parted`, `pkg/mig/reconfigure/reconfigure.go`, v0.14.x). In order, it:

1. Reads the current `nvidia.com/gpu.deploy.*` labels and `nvidia.com/mig.config.state`.
2. Validates the requested config against the config file; if it is already applied, sets `state=success` and stops.
3. Persists the selection to the host state file so it survives a reboot.
4. Checks whether a MIG *mode* change is required (case (a) vs (b) above).
5. Sets `nvidia.com/mig.config.state=pending`.
6. **`shutdownKubernetesGPUClients()`** — sets six node labels to `paused-for-mig-change`: `nvidia.com/gpu.deploy.device-plugin`, `.gpu-feature-discovery`, `.dcgm-exporter`, `.dcgm`, `.nvsm`, and `.client`. Those labels are the operands' `nodeSelector`s, so flipping them makes the DaemonSet controller delete those pods from this node. (A label already at `"false"` is left at `"false"` — the operand was deliberately off and stays off.)
7. **`waitForPodsToBeDeleted()`** — waits, with a **five-minute timeout each**, for `app=nvidia-device-plugin-daemonset`, `app=gpu-feature-discovery`, `app=nvidia-dcgm-exporter`, `app=nvidia-dcgm`, and any DaemonSet using the `gpu.deploy.client` selector to actually go away. Then it deletes the `nvidia-cuda-validator` and `nvidia-device-plugin-validator` pods outright.
8. Optionally (`WITH_SHUTDOWN_HOST_GPU_CLIENTS`, which the Operator sets when the driver is host-installed rather than containerised) stops host systemd services from `clients.yaml` — `nvsm*.service`, `nv_peer_mem.service`, `nvidia-dcgm.service`, `dcgm.service`, `dcgm-exporter.service` — **and the kubelet itself** if a mode change is required. Then sleeps 30 s.
9. `nvidia-mig-parted -d apply --mode-only`, then `assert --mode-only`. If that fails and `--with-reboot`/`WITH_REBOOT` is set, it sets `state=rebooting` and `chroot /host reboot`.
10. `nvidia-mig-parted -d apply` — the actual GI/CI creation.
11. If CDI is enabled: `nvidia-ctk system create-device-nodes --control-devices`, run `nvidia-smi` (needed to materialise the new `nvidia-cap*` nodes), then regenerate the CDI spec.
12. Restart host services, restart the validator pod, restore all six labels to `true`.
13. `nvidia.com/mig.config.state=success`.

**Nowhere in that list is `kubectl cordon`, `kubectl drain`, or an eviction of a workload pod.** The `clients.yaml` list contains monitoring and management daemons — the things that hold NVML handles — not tenants. If your training job is holding a MIG device, step 9 or 10 fails, `state=failed` is set, and step 12 still runs so the device plugin comes back.

Here is the timeline, with what is broken at each moment:

```
  MIG RECONFIGURATION TIMELINE — WHAT IS DOWN, WHEN
  (H = user workload holding a MIG device; ▮ = component running)

  t        action                                device   dcgm   sched-  running
                                                 plugin  export  ulable?  pods
  ──────────────────────────────────────────────────────────────────────────────
  t0       label nvidia.com/mig.config=<name>      ▮        ▮      yes      H ▮
  t0+0s    state=pending                           ▮        ▮      yes      H ▮
  t0+1s    gpu.deploy.* → paused-for-mig-change    ▮        ▮      yes      H ▮
  t0+5s    DaemonSet controller deletes pods       ·        ·      yes      H ▮
           └ kubelet loses the plugin socket
  t0+~30s  kubelet drops nvidia.com/mig-* from     ·        ·      NO       H ▮
           Allocatable (endpoint stopped)                                   │
           ── NEW GPU PODS PENDING FROM HERE ──                             │
                                                                            │
  t0+35s   nvidia-mig-parted apply --mode-only     ·        ·      no       H ▮
           ┌─────────────────────────────────────────────────────────┐     │
           │ IF H still holds a CUDA context:                        │     │
           │   NVML_ERROR_IN_USE → "In use by another client"        │     │
           │   state=failed; labels restored; plugin returns;        │     │
           │   OLD GEOMETRY INTACT; H unharmed.  ← the SAFE failure  │     │
           │ IF --with-reboot and this was a MODE change:            │     │
           │   state=rebooting → chroot /host reboot  ← H dies here  │     │
           └─────────────────────────────────────────────────────────┘     │
  t0+40s   nvidia-mig-parted apply (destroy CIs,   ·        ·      no       │
           destroy GIs, create new GIs+CIs)                                 │
           ┌─────────────────────────────────────────────────────────┐     │
           │ IF H holds the DEVICE NODE but no live context, the     │     │
           │ destroy SUCCEEDS. Every MIG-<uuid> on this card is now  │     │
           │ invalid. H is NOT evicted, NOT restarted, NOT marked    │     │
           │ unhealthy — it is holding a path to a partition that    │     │
           │ no longer exists.  ← the DANGEROUS success              │     ▼
           └─────────────────────────────────────────────────────────┘  H broken,
  t0+45s   nvidia-ctk create-device-nodes; nvidia-smi; CDI regen           status
  t0+50s   gpu.deploy.* → true; operands return    ▮        ▮      yes      Running
  t0+60s   plugin re-advertises NEW resource names ▮        ▮      yes
           state=success

  TOTAL WINDOW where the node cannot schedule GPU pods: ~30–90 s typical,
  up to 5 min per operand if a pod is slow to terminate (the wait timeout),
  and unbounded if --with-reboot fires.
```

**The operational rule that falls out:** the reconfiguration is safe against *contexts* and unsafe against *holders*. So the correct procedure is the one the Operator does not do for you:

```bash
# 1. Stop new work landing.
kubectl cordon gpu-a100

# 2. Evict everything holding a GPU on this node. --force covers bare pods;
#    without it, a pod with no controller blocks the drain forever.
kubectl drain gpu-a100 \
  --pod-selector='!app.kubernetes.io/name in (nvidia-device-plugin,gpu-feature-discovery)' \
  --ignore-daemonsets --delete-emptydir-data --force --timeout=10m

# 3. NOW change the geometry. mig-manager handles its own operands.
kubectl label node gpu-a100 nvidia.com/mig.config=all-balanced --overwrite
kubectl get node gpu-a100 -L nvidia.com/mig.config.state -w

# 4. Uncordon once state=success AND the new resource names are advertised.
kubectl uncordon gpu-a100
```

**One more thing the Operator now does that older material does not mention:** as of the v26.3 line, `mig-manager` **auto-generates** the mig-parted config from the hardware it discovers on each node, writing it to a per-node ConfigMap named `<nodeName>-mig-config`, and only uses `default-mig-parted-config` or your custom ConfigMap if one is explicitly named. The Helm values say so directly: *"MIG manager automatically generates configuration from hardware on each node. Only provide a custom config if you need settings that differ from hardware discovery."* If you are debugging why a geometry name is not recognised, look at the per-node ConfigMap, not the shared one.

### 7 — How Kubernetes sees the slices: `mig.strategy`

The device plugin turns MIG devices into extended resources, and *how* it names them is a scheduler-visible decision with direct consequences for quota and attribution. This is the most common place a MIG answer goes wrong under interview pressure.

**`strategy: none`** — MIG devices are ignored. On a MIG-enabled GPU the plugin advertises nothing for that card. Useful only as a transitional state.

**`strategy: single`** — every GPU on the node must have the *identical* geometry, and all MIG devices are advertised under the plain `nvidia.com/gpu` name. Eight A100s each split `7 × 1g.10gb` advertise `nvidia.com/gpu: 56`. GFD *overloads* the standard labels rather than adding new ones:

```
nvidia.com/mig.strategy        = single
nvidia.com/gpu.product         = A100-SXM4-40GB-MIG-1g.5gb   ← profile appended
nvidia.com/gpu.count           = 56                          ← MIG devices, not GPUs
nvidia.com/gpu.memory          = 5120                        ← per MIG device
nvidia.com/gpu.multiprocessors = 14
nvidia.com/gpu.slices.gi       = 1
nvidia.com/gpu.slices.ci       = 1
nvidia.com/gpu.engines.copy    = 1
nvidia.com/gpu.engines.decoder = 1
nvidia.com/gpu.engines.encoder = 1
nvidia.com/gpu.engines.jpeg    = 0
nvidia.com/gpu.engines.ofa     = 0
```

The upside is that workloads request plain `nvidia.com/gpu: 1` and need to know nothing about MIG. The downside is severe and specific: **the resource name hides the profile.** To the scheduler and to `ResourceQuota`, a `1g.10gb` slice and an entire un-partitioned 80 GB H100 on a neighbouring node are both `nvidia.com/gpu: 1`. That is exactly the silent capacity misrepresentation that produces "why did my job OOM on a GPU the scheduler said had room" tickets, and it makes per-slice-size quota impossible. The `gpu.product` label is your only defence, and it only works if every workload sets a `nodeSelector` on it — which is more discipline than "single strategy is simpler" implies.

**`strategy: mixed`** — each distinct profile gets its own resource name, and per-profile GFD labels are generated with the `MIG_TYPE` prefix:

```console
$ kubectl get node gpu-a100 -o jsonpath='{.status.capacity}' | tr ',' '\n' | grep -E 'nvidia|mig'
"nvidia.com/gpu":"0"
"nvidia.com/mig-1g.10gb":"14"
"nvidia.com/mig-2g.20gb":"6"
"nvidia.com/mig-3g.40gb":"4"

$ kubectl get node gpu-a100 -o json | jq '.metadata.labels | with_entries(select(.key|test("mig-1g")))'
{
  "nvidia.com/mig-1g.10gb.count":            "14",
  "nvidia.com/mig-1g.10gb.memory":           "10240",
  "nvidia.com/mig-1g.10gb.multiprocessors":  "14",
  "nvidia.com/mig-1g.10gb.slices.gi":        "1",
  "nvidia.com/mig-1g.10gb.slices.ci":        "1",
  "nvidia.com/mig-1g.10gb.engines.copy":     "1",
  "nvidia.com/mig-1g.10gb.engines.decoder":  "1",
  "nvidia.com/mig-1g.10gb.engines.jpeg":     "0",
  "nvidia.com/mig-1g.10gb.engines.ofa":      "0"
}
```

Note `nvidia.com/gpu: 0` alongside the MIG resources — under `mixed`, a MIG-enabled card contributes nothing to the generic name, so a pod requesting `nvidia.com/gpu: 1` will go `Pending` forever on that node with no obvious explanation. That is a real and common outage: someone enables MIG, existing manifests keep requesting `nvidia.com/gpu`, and nothing schedules.

**For a multi-tenant platform with per-team billing, `mixed` is almost always the answer**, for two reasons that are the same reason twice:

1. **Quota.** `ResourceQuota` fences on resource names, so `mixed` lets you cap a namespace's consumption of a *specific slice size*:

   ```yaml
   apiVersion: v1
   kind: ResourceQuota
   metadata: { name: gpu-slices, namespace: team-vision }
   spec:
     hard:
       requests.nvidia.com/mig-1g.10gb: "12"   # up to 12 inference slices
       requests.nvidia.com/mig-3g.40gb: "2"    # and at most 2 fine-tune slices
       requests.nvidia.com/gpu:         "0"    # explicitly: no whole cards
   ```

   Under `single` there is one bucket and no way to express any of that.
2. **Attribution.** The resource name *is* an attribution dimension. Before you have joined a single metric, `nvidia.com/mig-3g.40gb` tells you the pod holds 3/7 of a card's compute and 40 GB of its memory — which is exactly the `pod_share_of_device` term lesson 04.10's cost formula needs.

Configure it through the `ClusterPolicy` (the field is validated as an enum, `none|single|mixed`; note the Helm chart's default is `single`, not `none`):

```bash
kubectl -n gpu-operator patch clusterpolicy cluster-policy --type merge \
  -p '{"spec":{"mig":{"strategy":"mixed"},"migManager":{"enabled":true}}}'
```

### 8 — Per-slice attribution: the real join, including the hop

This is the payoff, and it is where a homegrown exporter most often breaks.

The naive model — the one asserted in most write-ups and in the previous version of this lesson — is: *pod-resources gives you `MIG-b6ba…`, DCGM emits metrics keyed by `MIG-b6ba…`, join on that string.* **That is wrong.** DCGM's exporter does not emit a MIG-UUID label at all. Its label builder (`NVIDIA/dcgm-exporter` 4.6.0, `internal/pkg/rendermetrics/render_metrics.go`) adds, for a MIG sample, the **parent GPU's** UUID plus two MIG identifiers:

```
DCGM_FI_PROF_GR_ENGINE_ACTIVE{gpu="0",UUID="GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763",
  pci_bus_id="00000000:1B:00.0",device="nvidia0",modelName="NVIDIA A100-SXM4-80GB",
  GPU_I_PROFILE="3g.40gb",GPU_I_ID="1",hostname="gpu-a100"} 0.81
```

`GPU_I_PROFILE` and `GPU_I_ID` are only present when the sample belongs to a GPU instance. The join key on the DCGM side is therefore `(gpu index, GPU_I_ID)` — dcgm-exporter builds it literally as `fmt.Sprintf("%s-%s", m.GPU, m.GPUInstanceID)`, e.g. `"0-1"`. The pod-resources side gives you a `MIG-…` UUID. **They do not match, and the bridge is NVML.**

```
  THE MIG ATTRIBUTION JOIN — FOUR SYSTEMS, ONE ROW

  ┌──────────────────────────┐
  │ kubelet pod-resources    │  List()
  │ /var/lib/kubelet/        │
  │  pod-resources/          │  PodResources{
  │  kubelet.sock            │    name: "infer-7b", namespace: "team-nlp",
  └────────────┬─────────────┘    containers:[{ name:"server",
               │                     devices:[{ resource_name:
               │                       "nvidia.com/mig-3g.40gb",
               │                       device_ids:["MIG-b6ba6b2a-…"] }]}]}
               ▼
      device_id = "MIG-b6ba6b2a-1cd4-5f1e-8a03-77d19e02c4b1"
               │
               │  ①  strip any "::N" suffix   (time-sliced MIG — yes, that
               │                                combination exists, 04.7)
               ▼
  ┌──────────────────────────┐
  │ NVML  — THE HOP          │  nvml.DeviceGetHandleByUUID("MIG-b6ba…")
  │ (needs the driver        │  → device.GetDeviceHandleFromMigDeviceHandle()
  │  visible in-process)     │  → parent.GetUUID()      = "GPU-8f2c1d90-…"
  │                          │  → device.GetGpuInstanceId()     = 1
  │ pre-R470 fallback: parse │  → device.GetComputeInstanceId() = 0
  │ "MIG-<parent>/<gi>/<ci>" │
  └────────────┬─────────────┘
               │  ②  (parentUUID, giID)  →  local GPU index
               ▼
        join key = "0-1"        ← "<gpu index>-<GPU instance id>"
               │
               ▼
  ┌──────────────────────────┐
  │ DCGM / dcgm-exporter     │  metric.GetIDOfType():
  │ nv-hostengine programs   │    if MigProfile != "" → "<gpu>-<GPU_I_ID>"
  │ counters per GPU_I       │
  │                          │  DCGM_FI_PROF_GR_ENGINE_ACTIVE{
  │  ✅ DISTINCT COUNTERS    │    gpu="0", UUID="GPU-8f2c1d90-…",
  │     PER GPU INSTANCE     │    GPU_I_PROFILE="3g.40gb", GPU_I_ID="1"} 0.81
  └────────────┬─────────────┘
               │  ③  attach owner labels
               ▼
   DCGM_FI_PROF_GR_ENGINE_ACTIVE{…,GPU_I_ID="1",
       namespace="team-nlp",pod="infer-7b",container="server"} 0.81
               │
               │  ④  × pod_share_of_device (3/7)  ×  $/GPU-hour
               ▼
        gpu_cost_utilised_per_hour{namespace="team-nlp",
            attribution="exact", device_type="mig"}  — lesson 04.10

  CONTRAST — time-slicing (04.7): steps ①–③ collapse, because DCGM has
  ONE counter for the whole device and N pods claim it. Here every GPU
  instance has its OWN counters, so sum() over instances is meaningful
  and no fallback estimator is needed.
```

Three consequences you should be able to state:

1. **Your exporter needs NVML in-process to do MIG attribution.** Not just DCGM, not just the pod-resources socket. That means the exporter pod must see the driver — either it requests a GPU itself, or it mounts the driver root. Plan for it; it is a deployment constraint, not a code detail.
2. **Both MIG UUID formats must be handled.** `MIG-<uuid>` on drivers ≥ R470 (470.42.01), `MIG-<GPU-UUID>/<gi>/<ci>` before that. `dcgm-exporter` implements both (`getMIGDeviceInfoForNewDriver` / `getMIGDeviceInfoForOldDriver`); if you write your own, so must you.
3. **This is why MIG attribution is "exact".** Each GPU instance has its own hardware counters, so `DCGM_FI_PROF_GR_ENGINE_ACTIVE{GPU_I_ID="1"}` genuinely describes only that instance's work. There is one number per billable unit. Under time-slicing there is one number per *seven* billable units, which is the entire subject of the next lesson.

**The `pod_share_of_device` term comes free from the profile name.** For cost, you need to know what fraction of the card a slice represents, and §3's table gives it exactly — but pick your denominator deliberately:

```
  compute share = profile SM count / 7g profile SM count
      3g.40gb on A100-80GB:  42 / 98  = 0.4286   (3/7, as expected)
      1g.10gb on A100-80GB:  14 / 98  = 0.1429   (1/7)

  memory share  = profile FB / (7g profile FB)
      3g.40gb:  40192 / 80384 = 0.5000
      1g.10gb:   9856 / 80384 = 0.1226

  ← these DISAGREE, and which one is right depends on what is scarce.
    For 1g.10gb: compute share 14.3 %, memory share 12.3 %. For a
    memory-bound inference tenant, billing on compute over-charges by
    16 %. State which basis you used in the metric, or in the docs.
```

And they must reconcile against the whole card. Under `7 × 1g.10gb`, the compute shares sum to 7 × (1/7) = 1.00 but the memory shares sum to 7 × 0.1226 = 0.858 — the missing 14 % is §4's stranded memory slice. **That gap is not a rounding error to hide; it is the platform's stranded capacity and it belongs in a platform-overhead bucket, not socialised onto tenants.** Lesson 04.10's reconciliation identity is exactly this check.

### 9 — The constraint that kills a popular multi-tenant design: MPS does not stack on MIG

The design that looks obviously right on a whiteboard — MIG for hard isolation *between* tenants, plus MPS *inside* a slice for finer-grained sharing among one tenant's own jobs — does not exist as a supported configuration.

The NVIDIA device-plugin README states it flatly, in a `[!NOTE]` block in the MPS sharing section: *"Sharing with MPS is currently not supported on devices with MIG enabled."* The same section carries a `[!WARNING]`: MPS support has been **experimental since device-plugin v0.15.0**, and it still is on the current v0.19.x line.

Time-slicing, by contrast, *is* supported on MIG resources. The README is explicit that the time-sliceable resources are `nvidia.com/gpu` **plus any resource type that emerges from `mixed` MIG strategy** — so `nvidia.com/mig-1g.10gb: 1` with `replicas: 4` advertises 4× that resource name and multiplexes four pods onto one MIG instance. That combination re-introduces the entire lesson-04.7 attribution hole *inside* a slice, which is worth knowing precisely because it looks like it should be safe: the instance has hardware isolation from its neighbours, and none at all from its own co-tenants.

So the design rule is: **one sharing mode per physical device.** If a fleet genuinely needs both patterns, split at the fleet level — some GPUs run MIG, others run MPS — not within one card. Lesson 04.8's decision matrix takes this constraint as an input.

## Perspectives

**Hardware/silicon (recap).** Module 03 covered *why* a GI is isolated: fenced SMs, a fenced L2 slice, a fenced memory path, enforced by partitioning hardware rather than software convention. Nothing here re-derives it. What is new is the shape of the constraint the silicon imposes on the *operator*: seven compute slices against eight memory slices, a fixed placement table, and the fact that a partition is a persistent hardware object whose creation and destruction is a privileged operation requiring exclusive access. Every operational awkwardness in this lesson traces back to that last property.

**Multi-tenant platform operator.** `mixed` strategy plus per-profile `ResourceQuota` is literally how a GPU cloud sells "a 1g.10gb slice" as a billable SKU. The quota key *is* the product name. What makes this workable is that the resource name is scheduler-visible, quota-visible, and attribution-visible all at once — one string carries the whole story. Under `single` you have one indistinguishable bucket and you are back to hoping tenants set the right `nodeSelector`.

**Capacity planner.** §4's table is a capacity-planning tool, not trivia. The question "how many `2g.20gb` tenants can this 64-GPU cluster hold?" has the answer 3 per card × 64 = 192, and the follow-up question — "and what is stranded?" — has the answer "one compute slice and 20 GB per card, i.e. 14 % of your compute and 27 % of your framebuffer, about $115,000/year at $4/GPU-hour". A planner who can produce both numbers is doing a different job from one who can produce the first.

**Cost/attribution.** MIG is the only mode in this module where the hardware boundary and the billing boundary coincide, which is why it is the baseline lesson 04.7 and 04.8 are measured against. But "exact" applies to the *utilisation* term, not the *share* term: §8 shows that compute-share and memory-share disagree by up to 16 % for the same slice, so even the clean case requires you to state a basis. Exact does not mean unambiguous.

**Managed-GPU-cloud reality.** Some GPU clouds expose tenant-configurable MIG geometry inside a tenant-scoped slice of a shared cluster. That is only plausible *because* MIG's per-slice resource model is scheduler-legible — you cannot safely delegate "pick your own partition layout" to a tenant on a sharing mode where the scheduler cannot tell the slices apart, and you cannot delegate it at all without also delegating the drain, which is the operationally hard part. Specific vendor claims in this area could not be independently verified from this environment (most vendor blog domains are blocked by the egress proxy), so treat any particular offering as something to check rather than a fact to repeat.

## Real-world use cases

- **[NVIDIA/mig-parted](https://github.com/NVIDIA/mig-parted) `pkg/mig/reconfigure/reconfigure.go`, v0.14.x — read this session.** The definitive answer to "why must you drain?", and the source of the §6 correction. The reconfiguration pauses six operand DaemonSets by label, waits five minutes each, and never touches a workload pod. Anyone who has debugged a stuck `mig.config.state=pending` has been looking at step 7 of that function timing out on a monitoring pod that would not terminate.
- **The GPU Operator's own `default-mig-parted-config` ConfigMap (v26.3.3) — read this session.** 625 lines covering every SKU from A30 to GB300, and a working example of a real production capacity decision embedded in a config file. Two of its `all-balanced` entries (`2g.20gb ×2 + 3g.40gb`, and the `3g.40gb + 4g.40gb` in the Helm values example) are exactly the zero-stranding geometries §2 derives from the placement table — which is a nice confirmation that the vendor picks them for the same reason.
- **The `+me` singleton constraint as a real capacity limit.** Every `all-*.me` entry in that ConfigMap has count `1`, because the card has one set of JPEG/OFA engines. A video-inference platform that needs media engines on more than one tenant per card cannot use MIG for that, full stop. This is the kind of hardware constraint that turns up three weeks into a migration if nobody read the profile table.
- **The `nvidia.com/gpu: 0` outage.** Under `mixed` strategy a MIG-enabled card contributes nothing to the generic `nvidia.com/gpu` resource. Every manifest in the cluster that requests `nvidia.com/gpu: 1` goes `Pending` with `0/1 nodes are available: 1 Insufficient nvidia.com/gpu`, on a node that visibly has GPUs. It is self-inflicted, it is instant, and it is the single most common thing to go wrong the first time a team enables MIG.
- **Honest gap.** No independently verified tier-1 company engineering blog narrating a MIG-in-production rollout with real utilisation and cost numbers could be fetched this session — `docs.nvidia.com`, `kubernetes.io` and essentially every blog domain are blocked by this environment's egress proxy. Rather than cite something unread, this lesson's numbers all come from sources that *were* read directly: NVIDIA's NVML header, NVIDIA's `go-nvml` profile definitions, NVIDIA's `mig-parted` source, and the GPU Operator's shipped ConfigMap. Those are stronger evidence than a blog post anyway.

## Worked example

Take one A100-SXM4-80GB node from unpartitioned to a `mixed`-strategy multi-tenant node with verified per-slice attribution, then break the reconfiguration on purpose and recover it. Rent an A100 or H100 for one session; everything below takes about ninety minutes.

### Step 1 — establish the starting state and the cost of MIG mode

```console
$ nvidia-smi -L
GPU 0: NVIDIA A100-SXM4-80GB (UUID: GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763)

$ nvidia-smi -q -i 0 | grep -E "MIG Mode|Current|Pending" | head -3
    MIG Mode
        Current                           : Disabled
        Pending                           : Disabled

$ nvidia-smi --query-gpu=name,memory.total --format=csv
name, memory.total [MiB]
NVIDIA A100-SXM4-80GB, 81920 MiB
```

Before enabling MIG, write down what you are about to give up. From §4:

```
  Un-partitioned A100 80GB:  108 SMs,  81920 MiB
  MIG-enabled, one 7g.80gb:   98 SMs,  80384 MiB
  ────────────────────────────────────────────────
  cost of MIG mode itself:  −10 SMs (−9.3 %),  −1536 MiB (−1.9 %)
```

That is the number to quote when someone proposes "just enable MIG everywhere, we can always use 7g".

### Step 2 — pick the geometry from the placement table, not from habit

Suppose the tenant mix is: one 40 GB fine-tuning job, and up to three small inference servers needing ~18 GB each. Candidate geometries:

```
  (a) 7 × 1g.10gb          → 7 tenants × 9.6 GB.   Inference needs 18 GB. ✗
  (b) 4 × 1g.20gb          → 4 tenants × 19.5 GB, but only 4 of 7 compute
                              slices used — 42 SM idle. Fine-tune gets 14 SM. ✗
  (c) 3 × 2g.20gb          → 3 tenants × 19.5 GB, 28 SM each. No 40 GB slot. ✗
  (d) 2 × 2g.20gb + 3g.40gb → 2 × 19.5 GB inference + 1 × 39.25 GB fine-tune,
                              98/98 SM, 80128/81920 MiB.  ZERO STRANDING.  ✓
```

Check (d) against the placement axis before trusting it: `2g` may start at {0,2,4} with size 2; `3g` may start at {0,4} with size 4. Place `2g`@0, `2g`@2, `3g`@4 → memory slices 0–7, all consumed; compute 2+2+3 = 7, all consumed. Valid.

```console
$ nvidia-smi mig -lgipp
GPU  0 Profile ID 19 Placements: {0,1,2,3,4,5,6}:1
GPU  0 Profile ID 15 Placements: {0,2,4,6}:2
GPU  0 Profile ID 14 Placements: {0,2,4}:2
GPU  0 Profile ID  9 Placements: {0,4}:4
GPU  0 Profile ID  5 Placement : {0}:4
GPU  0 Profile ID  0 Placement : {0}:8
```

### Step 3 — do it by hand once, to see the mechanism

```console
$ sudo nvidia-smi -i 0 -mig 1
Enabled MIG Mode for GPU 00000000:1B:00.0
All done.

$ sudo nvidia-smi mig -cgi 2g.20gb,2g.20gb,3g.40gb -C
Successfully created GPU instance ID  7 on GPU  0 using profile MIG 2g.20gb (ID 14)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  7 using profile MIG 2g.20gb (ID  1)
Successfully created GPU instance ID  8 on GPU  0 using profile MIG 2g.20gb (ID 14)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  8 using profile MIG 2g.20gb (ID  1)
Successfully created GPU instance ID  2 on GPU  0 using profile MIG 3g.40gb (ID  9)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  2 using profile MIG 3g.40gb (ID  2)

$ nvidia-smi mig -lgi
+-------------------------------------------------------+
| GPU instances:                                        |
| GPU   Name          Profile  Instance   Placement     |
|                       ID       ID       Start:Size    |
|=======================================================|
|   0  MIG 2g.20gb      14        7          0:2        |
|   0  MIG 2g.20gb      14        8          2:2        |
|   0  MIG 3g.40gb       9        2          4:4        |
+-------------------------------------------------------+

$ nvidia-smi -L
GPU 0: NVIDIA A100-SXM4-80GB (UUID: GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763)
  MIG 2g.20gb  Device  0: (UUID: MIG-1a7f3e02-9b44-5c8d-a1e6-3d2c9f60b481)
  MIG 2g.20gb  Device  1: (UUID: MIG-5c93b71d-4e28-5aa9-b0f7-8e1d24c73a95)
  MIG 3g.40gb  Device  2: (UUID: MIG-b6ba6b2a-1cd4-5f1e-8a03-77d19e02c4b1)
```

`Placement Start:Size` reads `0:2`, `2:2`, `4:4` — the memory axis exactly filled, as designed in step 2. **Instance IDs are not sequential and are not indices**; `7`, `8`, `2` are what the driver assigned. Never assume GI ids are `0..n`; `GPU_I_ID` in DCGM carries these values verbatim and your join must too.

Now tear it back down so the Operator can own it:

```console
$ sudo nvidia-smi mig -dci && sudo nvidia-smi mig -dgi
Successfully destroyed compute instance ID  0 from GPU  0 GPU instance ID  7
Successfully destroyed compute instance ID  0 from GPU  0 GPU instance ID  8
Successfully destroyed compute instance ID  0 from GPU  0 GPU instance ID  2
Successfully destroyed GPU instance ID  7 from GPU  0
Successfully destroyed GPU instance ID  8 from GPU  0
Successfully destroyed GPU instance ID  2 from GPU  0
```

### Step 4 — hand it to the Operator

```console
$ kubectl -n gpu-operator patch clusterpolicy cluster-policy --type merge \
    -p '{"spec":{"mig":{"strategy":"mixed"},"migManager":{"enabled":true}}}'
clusterpolicy.nvidia.com/cluster-policy patched

$ kubectl -n gpu-operator get pods -l app=nvidia-mig-manager -o wide
NAME                       READY   STATUS    RESTARTS   AGE   NODE
nvidia-mig-manager-8kx2v   1/1     Running   0          41s   gpu-a100
```

A custom geometry needs a ConfigMap, because `2g.20gb ×2 + 3g.40gb` is not one of the shipped named configs:

```yaml
# custom-mig-config.yaml — the key MUST be named config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-mig-parted-configs
  namespace: gpu-operator
data:
  config.yaml: |-
    version: v1
    mig-configs:
      all-disabled:
        - devices: all
          mig-enabled: false
      tenant-mix:
        - devices: all
          mig-enabled: true
          mig-devices:
            "2g.20gb": 2
            "3g.40gb": 1
```

```console
$ kubectl apply -f custom-mig-config.yaml
$ kubectl -n gpu-operator patch clusterpolicy cluster-policy --type merge \
    -p '{"spec":{"migManager":{"config":{"name":"custom-mig-parted-configs","default":"all-disabled"}}}}'
```

### Step 5 — drain first (the Operator will not), then apply

```console
$ kubectl cordon gpu-a100
node/gpu-a100 cordoned

$ kubectl drain gpu-a100 --ignore-daemonsets --delete-emptydir-data --force --timeout=10m
node/gpu-a100 already cordoned
evicting pod team-nlp/infer-7b-5c9f
pod/infer-7b-5c9f evicted
node/gpu-a100 drained

$ kubectl label node gpu-a100 nvidia.com/mig.config=tenant-mix --overwrite
node/gpu-a100 labeled

$ kubectl get node gpu-a100 -L nvidia.com/mig.config,nvidia.com/mig.config.state -w
NAME       STATUS                     MIG.CONFIG    MIG.CONFIG.STATE
gpu-a100   Ready,SchedulingDisabled   tenant-mix    
gpu-a100   Ready,SchedulingDisabled   tenant-mix    pending
gpu-a100   Ready,SchedulingDisabled   tenant-mix    success
```

Watch the operand labels move during the `pending` window — this is §6's step 6 happening live:

```console
$ kubectl get node gpu-a100 -o json | jq '.metadata.labels
    | with_entries(select(.key|test("gpu.deploy")))'
{
  "nvidia.com/gpu.deploy.dcgm":                  "paused-for-mig-change",
  "nvidia.com/gpu.deploy.dcgm-exporter":         "paused-for-mig-change",
  "nvidia.com/gpu.deploy.device-plugin":         "paused-for-mig-change",
  "nvidia.com/gpu.deploy.gpu-feature-discovery": "paused-for-mig-change",
  "nvidia.com/gpu.deploy.driver":                "true",
  "nvidia.com/gpu.deploy.container-toolkit":     "true"
}
```

Four operands paused; the driver and toolkit are untouched, because they are not GPU *clients* in the NVML sense. Once `state=success` those four go back to `"true"` and the plugin re-advertises.

```console
$ kubectl get node gpu-a100 -o jsonpath='{.status.capacity}' | tr ',' '\n' | grep nvidia
"nvidia.com/gpu":"0"
"nvidia.com/mig-2g.20gb":"2"
"nvidia.com/mig-3g.40gb":"1"

$ kubectl uncordon gpu-a100
node/gpu-a100 uncordoned
```

`nvidia.com/gpu: 0` is expected under `mixed` — and is the thing to warn your tenants about before you flip the switch.

### Step 6 — schedule onto slices and recover the attribution row

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: finetune-a
  namespace: team-vision
  labels: { team: vision, workload: finetune }
spec:
  restartPolicy: Never
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:12.8.1-base-ubuntu24.04
      command: ["sh","-c","nvidia-smi -L && sleep 3600"]
      resources:
        limits:
          nvidia.com/mig-3g.40gb: 1
```

```console
$ kubectl apply -f finetune-a.yaml
$ kubectl -n team-vision logs finetune-a
GPU 0: NVIDIA A100-SXM4-80GB (UUID: GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763)
  MIG 3g.40gb  Device  0: (UUID: MIG-b6ba6b2a-1cd4-5f1e-8a03-77d19e02c4b1)
```

The container sees **only** its own MIG device. That is the isolation, visible from inside.

Now the attribution row, using the lesson-04.3 client:

```console
$ kubectl -n gpu-operator logs ds/podres-probe --tail=5
team-vision/finetune-a       cuda       nvidia.com/mig-3g.40gb   mig  MIG-b6ba6b2a-1cd4-5f1e-8a03-77d19e02c4b1

--- node summary ---
allocatable(healthy)=3  allocated=1  free=2  dra-claims=0
```

And the hop from §8, which you can do by hand to convince yourself:

```console
$ nvidia-smi -L | grep -A0 "MIG-b6ba6b2a"
  MIG 3g.40gb  Device  2: (UUID: MIG-b6ba6b2a-1cd4-5f1e-8a03-77d19e02c4b1)
$ nvidia-smi mig -lgi | grep "3g.40gb"
|   0  MIG 3g.40gb       9        2          4:4        |
                                  ^ GPU instance ID = 2

$ curl -s localhost:9400/metrics | grep 'GPU_I_ID="2"' | head -2
DCGM_FI_PROF_GR_ENGINE_ACTIVE{gpu="0",UUID="GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763",device="nvidia0",modelName="NVIDIA A100-SXM4-80GB",GPU_I_PROFILE="3g.40gb",GPU_I_ID="2",hostname="gpu-a100",namespace="team-vision",pod="finetune-a",container="cuda"} 0.79
DCGM_FI_DEV_FB_USED{gpu="0",UUID="GPU-8f2c1d90-3e5a-4b17-9c2e-5da9abb51763",GPU_I_PROFILE="3g.40gb",GPU_I_ID="2",hostname="gpu-a100",namespace="team-vision",pod="finetune-a",container="cuda"} 31744
```

`MIG-b6ba…` (pod-resources) → GI id 2 (NVML/`-lgi`) → `GPU_I_ID="2"` (DCGM). **Three systems, one row, no estimation.**

### Step 7 — cost, with the share term made explicit

```
  Node rate $32.00/hr, 8 GPUs         → $4.00 per physical GPU-hour
  finetune-a holds 3g.40gb on GPU 0.

  compute share = 42 SM / 98 SM       = 0.4286
  memory share  = 40192 / 80384 MiB   = 0.5000
  ── pick one and SAY SO. This is a fine-tune; compute is scarce. ──

  allocated cost = $4.00 × 0.4286               = $1.714 /hr
  utilised cost  = $1.714 × 0.79 (GR_ENGINE_ACTIVE) = $1.354 /hr
  labels: attribution="exact", device_type="mig", share_basis="compute"

  Whole-card reconciliation for this geometry (tenant-mix):
    2 × 2g.20gb : 2 × 0.2857 = 0.5714
    1 × 3g.40gb :     0.4286 = 0.4286
                              ───────
                               1.0000  ✓ compute shares sum to the card
    Framebuffer: (2×19968 + 40192) / 81920 = 97.8 %
                 → 1792 MiB (2.2 %) = $0.088/hr to platform overhead
```

That last line is the reconciliation discipline lesson 04.10 formalises: **the shares of all tenants plus the platform's stranded remainder must equal exactly one card.** Here it does, and the remainder is small because the geometry was chosen from the placement table. Under `7 × 1g.10gb` the same check would show 14 % unattributable, and finding that in your own reconciliation is how you discover you picked the wrong geometry.

### Step 8 — break it, read the failure, recover

With `finetune-a` still running:

```console
$ kubectl label node gpu-a100 nvidia.com/mig.config=all-1g.10gb --overwrite
node/gpu-a100 labeled

$ kubectl get node gpu-a100 -L nvidia.com/mig.config.state -w
gpu-a100   pending
gpu-a100   failed

$ kubectl -n gpu-operator logs -l app=nvidia-mig-manager --tail=30
time="…" level=info msg="Updating to MIG config: all-1g.10gb"
time="…" level=info msg="Selected MIG config is not currently applied"
time="…" level=info msg="Shutting down all GPU clients in Kubernetes by disabling their component-specific nodeSelector labels"
time="…" level=info msg="Waiting for the device-plugin to shutdown"
time="…" level=info msg="Waiting for gpu-feature-discovery to shutdown"
time="…" level=info msg="Waiting for dcgm-exporter to shutdown"
time="…" level=info msg="Waiting for dcgm to shutdown"
time="…" level=info msg="Applying the MIG mode change from the selected config to the node (and double checking it took effect)"
time="…" level=info msg="Applying the selected MIG config to the node"
ERRO[0004] Unable to destroy GPU instance ID  2 from GPU  0: In use by another client
time="…" level=error msg="failed to apply MIG configuration: exit status 1"
time="…" level=info msg="Restarting any GPU clients previously shutdown in Kubernetes by reenabling their component-specific nodeSelector labels"
```

Read it against §6's step list. Steps 6 and 7 succeeded — the four operands went away. Step 10 failed at the first `nvidia-smi mig -dgi`, because `finetune-a` still holds a CUDA context on GI 2. `setState(failed)` then restored the operand labels, so the plugin came back and the old geometry is intact. **This is the safe failure.** Verify:

```console
$ kubectl get node gpu-a100 -o jsonpath='{.status.capacity}' | tr ',' '\n' | grep mig
"nvidia.com/mig-2g.20gb":"2"
"nvidia.com/mig-3g.40gb":"1"
$ kubectl -n team-vision get pod finetune-a
NAME         READY   STATUS    RESTARTS   AGE
finetune-a   1/1     Running   0          14m
```

Old geometry, pod unharmed. Recover the right way:

```console
$ kubectl cordon gpu-a100 && kubectl drain gpu-a100 --ignore-daemonsets --delete-emptydir-data --force
node/gpu-a100 drained
$ kubectl label node gpu-a100 nvidia.com/mig.config=all-1g.10gb --overwrite
$ kubectl get node gpu-a100 -L nvidia.com/mig.config.state -w
gpu-a100   pending
gpu-a100   success
$ kubectl get node gpu-a100 -o jsonpath='{.status.capacity}' | tr ',' '\n' | grep mig
"nvidia.com/mig-1g.10gb":"7"
$ kubectl uncordon gpu-a100
```

Seven slices, and — per §4 — 12.9 GB of that card is now unallocatable. Write that number into your failure-mode log next to the recovery, because it is the part that costs money after the incident is closed.

## Practice

Feeds the deliverable's **per-pod attribution map** and **failure-mode log**, both part of [Per-pod GPU attribution](../practice/per-pod-attribution/README.md). Rent an A100 or H100 for one session; if you have no MIG-capable card, steps 1, 2, 7 and 8 are still doable from the tables and the source.

1. **Read the geometry off the hardware.** Capture `nvidia-smi mig -lgip` and `-lgipp` for your card. Build the §2 placement table from *your* output and confirm it matches §3's profile table for that SKU. Note the driver version — profile availability moves between driver branches.
2. **Compute your card's efficiency table.** Reproduce §4's table for your GPU: for each shipped geometry, SMs handed out, framebuffer handed out, and stranded remainder in GB and in dollars per month at your rate card. Identify every zero-stranding geometry and how many tenants each supports. This is a spreadsheet, and it belongs in the deliverable.
3. **Create and destroy a geometry by hand** with `-cgi … -C`, `-lgi`, `-lci`, `-dci`, `-dgi`. Capture the `Placement Start:Size` column and confirm it matches the placements you predicted in step 1. Then create a GI **without** `-C` and confirm that no MIG device appears in `nvidia-smi -L` — the invisible-GI failure mode.
4. **Hand it to the Operator.** Set `mig.strategy: mixed`, confirm `nvidia-mig-manager` is `Running`, author a custom ConfigMap with a *zero-stranding* geometry from step 2 (key must be `config.yaml`), and apply it via the `nvidia.com/mig.config` label. Record the `mig.config.state` transitions and, during `pending`, capture the `nvidia.com/gpu.deploy.*` labels showing `paused-for-mig-change`.
5. **Record the `nvidia.com/gpu: 0` trap.** Before switching to `mixed`, deploy a pod requesting plain `nvidia.com/gpu: 1`. After the switch, capture its `Pending` status and the `Insufficient nvidia.com/gpu` scheduler event. This is failure-mode-log entry material and it is the most common self-inflicted MIG outage.
6. **Build the attribution row, all three hops.** Schedule a labelled pod onto a slice. Capture (a) the MIG UUID from the pod's own `nvidia-smi -L`, (b) the same UUID from the pod-resources API via your 04.3 client, (c) the GI id from `nvidia-smi mig -lgi`, and (d) a `dcgm-exporter` `/metrics` line carrying `GPU_I_ID` equal to that GI id plus the pod/namespace labels. Write the four-line chain down; that chain *is* the attribution proof.
7. **Trigger and recover the reconfiguration failure.** With the pod running, change the geometry label. Capture `mig.config.state=failed`, the `In use by another client` line from the mig-manager log, and proof that the old geometry and the pod both survived. Then cordon, drain, re-apply, and capture `success`. In the log entry, state explicitly **which component did the draining** — the answer is "you did", and that is the point.
8. **Write the MPS-on-MIG paragraph.** One paragraph in the failure-mode log explaining why MPS cannot be layered inside a MIG slice, citing the device-plugin README's own note rather than a guess, and stating what you would do instead if a design needed both patterns.

**Acceptance:**

- A **geometry-efficiency table** for your SKU: every shipped geometry with SMs used, framebuffer used, stranded remainder, tenant count, and monthly dollar cost of the stranding at your rate.
- A **verified attribution chain** for one MIG pod: `pod → resource_name → MIG-UUID → GI id → DCGM GPU_I_ID → metric value`, all four artefacts captured, saved to the attribution map with the pod's team label.
- A **reconfiguration-failure entry** in the failure-mode log: symptom (`mig.config.state=failed`), evidence (the `In use by another client` log line, plus the operand labels showing what mig-manager *did* pause), root cause (a workload held a CUDA context on the GI; mig-manager pauses operands, not tenants), fix (cordon + drain + re-apply), prevention (drain as a required pre-step in the runbook, or a `PreStop` hook that releases the context).
- The **`nvidia.com/gpu: 0` entry** and the **MPS-on-MIG paragraph**, both cited to the primary source.

## Common pitfalls

1. **Believing `mig-manager` drains the node.** It does not cordon, and it does not evict workloads. It pauses six operand DaemonSets via `nvidia.com/gpu.deploy.*=paused-for-mig-change` and waits five minutes each for those pods. Your training job is not on that list. If you skip the drain you get `In use by another client` — which is the *good* outcome, because the alternative is the destroy succeeding against a pod that held the device without an active context, leaving it pointed at a partition that no longer exists with nothing in Kubernetes noticing.
2. **Requesting `nvidia.com/gpu` on a `mixed`-strategy node.** A MIG-enabled card contributes `nvidia.com/gpu: 0`. Every legacy manifest goes `Pending` on a node full of GPUs. Migrate the manifests *before* you flip the strategy, or run `mixed` on a distinct node pool with a taint.
3. **Assuming any set of profiles that fits by memory will fit by placement.** `4g.40gb` has exactly one valid start position, so two of them can never coexist regardless of arithmetic. `3g` consumes four memory slices, not three. Check `nvidia-smi mig -lgipp` before designing a geometry; the failure is a confusing "insufficient resources" at create time.
4. **Choosing `7 × 1g` reflexively.** It maximises tenant count and strands one memory slice — 12.9 GB on an 80 GB card, ~$373/GPU/month at $4/GPU-hour. If your tenants need more than a single slice's memory, you have paid for maximum density and delivered unusable capacity.
5. **Joining pod-resources MIG UUIDs directly to DCGM series.** They do not match. DCGM labels a MIG sample with the *parent* `UUID` plus `GPU_I_PROFILE` and `GPU_I_ID`; the bridge is an NVML lookup (`DeviceGetHandleByUUID` → `GetDeviceHandleFromMigDeviceHandle` → `GetGpuInstanceId`). An exporter without NVML in-process cannot do MIG attribution at all.
6. **Assuming GPU-instance IDs are indices.** The driver assigns them; `7`, `8`, `2` is a perfectly normal set for three instances. Code that assumes `0..n-1` mis-attributes.
7. **Creating GIs without CIs.** `-cgi` alone gives you a GI that consumes memory slices, appears in `-lgi`, and produces no MIG device and no schedulable resource. Use `-C`, or follow with `-cci`.
8. **Treating MIG and time-sliced UUIDs as equivalent for billing.** Only MIG gives a distinct hardware entity with its own counters. Time-sliced pods share one physical device and one metric (lesson 04.7). And note that time-slicing *can* be layered on a MIG resource name — which re-opens the attribution hole inside a slice that looked safe.
9. **Assuming MPS stacks on MIG.** The device-plugin explicitly does not support MPS on MIG-enabled devices, and MPS itself has been experimental since v0.15.0. One sharing mode per physical device; split at the fleet level if you need both.
10. **Not distinguishing a first-time MIG-mode enable from a later geometry change.** Enabling or disabling MIG mode may unbind/reset the device and can require a reboot (`NVML_ERROR_RESET_REQUIRED` on platforms without exposed reset); a geometry change on an already-enabled GPU only destroys and recreates GIs/CIs. Budget a reboot window for the first one — that is what `--with-reboot`/`WITH_REBOOT` exists for, and it will reboot your node.

## Self-check

- **Why must you drain the node before reconfiguring MIG — what state is torn down, and who does the draining?** *Answer:* Two distinct disruptions. Changing MIG **mode** calls `nvmlDeviceSetMigMode`, which the NVML header says "may unbind or reset the device… attributes such as minor number might change"; if the device is not idle the unbind fails and NVML returns `NVML_ERROR_IN_USE` (19), rendered by `nvidia-smi` as `In use by another client`. On platforms without exposed reset you instead get `NVML_ERROR_RESET_REQUIRED` and need a reboot. Changing **geometry** on an already-enabled GPU destroys and recreates every Compute Instance and GPU Instance, which invalidates every MIG-device UUID on the card and re-creates the `/dev/nvidia-caps/*` nodes — and each destroy fails with `NVML_ERROR_IN_USE` while a client holds the instance. **Who drains:** not `mig-manager`. Its `Run()` sets `mig.config.state=pending`, flips six `nvidia.com/gpu.deploy.*` labels to `paused-for-mig-change` (device-plugin, GFD, dcgm-exporter, dcgm, nvsm, third-party clients), waits up to five minutes per operand for those pods to be deleted, optionally stops host systemd GPU services (and the kubelet, on a mode change), then applies. It never cordons the node and never evicts a workload pod. You must `kubectl cordon` and `kubectl drain` yourself; skipping it produces the failure, and — worse — a pod holding the device *without* a live context lets the destroy succeed, leaving that pod pointed at a partition that no longer exists with nothing in Kubernetes marking it unhealthy.

- **Single vs mixed MIG strategy — how does each appear in node capacity, and which supports per-slice quota?** *Answer:* **`single`** requires every GPU on the node to have the identical geometry and reports all MIG devices under the generic `nvidia.com/gpu` name (8 GPUs × 7 slices → `nvidia.com/gpu: 56`). GFD *overloads* the standard labels: `gpu.product` becomes e.g. `A100-SXM4-40GB-MIG-1g.5gb`, `gpu.count` becomes the MIG-device count, `gpu.memory` becomes per-slice, plus `gpu.multiprocessors`, `gpu.slices.gi`, `gpu.slices.ci` and `gpu.engines.*`. Workloads request plain `nvidia.com/gpu: 1` and the profile is invisible to the scheduler — a `1g.10gb` slice and a whole H100 are the same resource. **`mixed`** advertises each profile under its own name (`nvidia.com/mig-1g.10gb: 14`, `nvidia.com/mig-3g.40gb: 4`, …) with per-profile `MIG_TYPE.*` labels, allows different slice sizes on one card, and sets `nvidia.com/gpu: 0` on MIG-enabled cards — which is the trap that puts every legacy manifest into `Pending`. Only `mixed` supports per-profile `ResourceQuota` (`requests.nvidia.com/mig-1g.10gb: "12"`), because `single` collapses every slice size into one indistinguishable bucket. `mixed` is also the strategy where the resource name itself is an attribution key.

- **You have an 80 GB A100 and need three ~20 GB tenants. Which geometries work, which is best, and what is stranded?** *Answer:* Candidates are `3 × 2g.20gb` and `2 × 2g.20gb + 1 × 3g.40gb`. `3 × 2g.20gb` gives three 19.5 GB / 28 SM tenants but uses only 6 of 7 compute slices and 6 of 8 memory slices: **1 compute slice (14 SM) and 20 GB stranded**, framebuffer efficiency 73.1 %. `2 × 2g.20gb + 3g.40gb` gives two 19.5 GB / 28 SM tenants plus one 39.25 GB / 42 SM tenant, consuming 2+2+3 = 7 compute slices and 2+2+4 = 8 memory slices — **zero stranding**, 97.8 % framebuffer efficiency. Verify against the placement table before trusting it: `2g` may start at {0,2,4} with size 2, `3g` at {0,4} with size 4, so `2g`@0, `2g`@2, `3g`@4 exactly fills the axis. If all three tenants genuinely need only 20 GB, the honest answer is that the third one gets a bigger slice than it asked for and you should either right-size the tenant or accept the 73 % geometry — but you should make that choice knowingly, at roughly $0.79/hour per card of difference at a $4/GPU-hour rate.

- **How does a MIG slice surface to the scheduler, and why is its attribution "exact" where time-slicing's is not — and what is the extra step?** *Answer:* Under `mixed`, each MIG device is advertised as its own extended resource (`nvidia.com/mig-3g.40gb`). When a pod requests one, the device plugin allocates a specific MIG device and injects its UUID; the kubelet records that in `podDevices` and the pod-resources API reports `{resource_name, device_ids: ["MIG-b6ba…"]}`. Attribution is exact because a GPU Instance is a real hardware partition with **its own performance counters** — DCGM measures GI 2 and only GI 2, so there is one number per billable unit. Time-slicing multiplexes N pods onto one unpartitioned device, so DCGM has one number for N claimants and any split is an estimate. **The extra step people miss:** DCGM does not emit a MIG-UUID label. A MIG sample carries the *parent* GPU's `UUID` plus `GPU_I_PROFILE="3g.40gb"` and `GPU_I_ID="2"`, and the exporter's join key is `"<gpuIndex>-<GPU_I_ID>"`. Getting from `MIG-b6ba…` to that key requires NVML — `DeviceGetHandleByUUID` → `GetDeviceHandleFromMigDeviceHandle` → `GetUUID` on the parent → `GetGpuInstanceId` — with a string-parsing fallback for the pre-R470 `MIG-<parent>/<gi>/<ci>` format. So a MIG-capable exporter needs the driver visible in-process, which is a deployment constraint, not just a code detail.

- **What does enabling MIG cost you before you have partitioned anything, and how do you quantify a geometry's waste?** *Answer:* MIG mode itself reserves resources. The full-card profile reports 98 SMs on an A100 (die has 108, so −9.3 %) and 112 SMs on an H100 SXM5 (die has 132, so −15.2 %); framebuffer loses ~1.9 % (`7g.80gb` = 80384 MiB of 81920). So a tenant who needs a whole card should get MIG turned *off*, not a `7g` slice. For a geometry, compute two efficiencies against the MIG-mode maxima: **SMs handed out ÷ 98** (A100-80GB) and **framebuffer handed out ÷ 81920 MiB**. `7 × 1g.10gb`: 98/98 compute but 68992/81920 = 84.2 % memory — one memory slice, ~12.9 GB, stranded because there are eight memory slices and only seven compute slices. `4 × 1g.20gb`: 79872 MiB = 97.5 % memory but only 56/98 = 57.1 % compute. `1 × 4g.40gb`: half the card idle, because a 4-slice GI has exactly one valid placement so a second cannot exist. The zero-stranding geometries are `7g.80gb`, `4g+3g`, `2g×2+3g`, and `1g×2+2g+3g` (`all-balanced`) — all of which use 7 compute and 8 memory slices. Convert to money by multiplying the stranded fraction by your $/GPU-hour: 12.9 GB on a $4.00/hour card is $0.51/hour, $373/month, per GPU.

- **Can you run MPS inside a MIG slice, or time-slice a MIG slice?** *Answer:* MPS, no: the NVIDIA device-plugin README states that "Sharing with MPS is currently not supported on devices with MIG enabled", and MPS support itself has been flagged experimental since v0.15.0 and still is on v0.19.x. Time-slicing, **yes** — the README explicitly lists the time-sliceable resources as `nvidia.com/gpu` plus any resource type produced by `mixed` MIG strategy, so `nvidia.com/mig-1g.10gb` with `replicas: 4` is a supported configuration. That is worth flagging rather than celebrating: it re-creates lesson 04.7's attribution hole *inside* a slice, where the instance has hardware isolation from its neighbours and none at all from its own four co-tenants, and DCGM's per-GI counter becomes one number for four claimants. The design rule is one sharing mode per physical device; if a fleet needs both MIG and MPS patterns, split at the fleet level.

## Connections & what's next

This lesson reused lesson 04.5's pause/mutate/validate/restore pattern one layer lower — a GPU Instance instead of a kernel module, the same "cannot mutate while a client holds it open" constraint, and the same break/fix discipline of triggering the failure deliberately and reading the log line. The correction it adds is that at this layer the Operator's automation stops short of your workloads, so the drain is yours to run.

It also sets the clean baseline the rest of the module is measured against. Lesson 04.3's device-ID table listed the MIG row and flagged the NVML hop; §8 here is that hop worked out in full, and lesson 04.10's capstone consumes it as the `attribution="exact"` code path. Lesson 04.7 takes the same join and shows it collapsing under time-slicing — same map, one metric, N claimants — and quantifies exactly how wrong the fallback is. Lesson 04.8 adds MPS as a third point, with the MPS-on-MIG exclusion from §9 as a hard input to its decision matrix. Lesson 04.9's DRA model eventually offers a structurally cleaner way to express "a 3g.40gb slice for this workload" than an extended-resource name, but it does not repeal any of §2's placement geometry — the silicon constraints survive the API change.

Next: **[04.7 · Time-slicing and the attribution trap](07-time-slicing-attribution.md)** takes the shared-UUID problem this lesson avoids and makes it the whole subject — what happens to cost attribution when Kubernetes and DCGM both see one device where there are really N tenants sharing it.

## References & further reading

**Primary sources — read directly this session (August 2026)**

- [NVML API header (`nvml.h`)](https://docs.nvidia.com/deploy/nvml-api/) — read from the copy shipped in the DCGM source tree, because `docs.nvidia.com` is blocked by this environment's egress proxy. Source of: `nvmlGpuInstancePlacement_t{start, size}` documented as **memory** slices and `nvmlComputeInstancePlacement_t` as **compute** slices (§2); the `nvmlGpuInstanceProfileInfo_{,_v2,_v3}_t` field set; `NVML_ERROR_IN_USE = 19` and `NVML_ERROR_RESET_REQUIRED = 16`; and the `nvmlDeviceSetMigMode` contract quoted in §5 ("may unbind or reset the device… the caller is expected to idle the device and retry").
- [NVIDIA/go-nvml — `pkg/nvml/mock/gpus/{a100,h100,h200,b200,a30}.go`](https://github.com/NVIDIA/go-nvml) — NVIDIA's own per-SKU GPU-instance profile definitions, mirroring `nvmlDeviceGetGpuInstanceProfileInfo` and `nvmlDeviceGetGpuInstancePossiblePlacements`. Every slice count, instance count, SM count, engine count, framebuffer size and placement list in §2–§4 comes from here. **Correction to the previous version of this lesson:** it omitted the `4g.20gb` / `4g.40gb` profiles entirely and mis-stated the A100-80GB and H100 profile sets; the `4g` row is the one with a single valid placement and is exactly the trap worth knowing.
- [NVIDIA/mig-parted — `pkg/mig/reconfigure/reconfigure.go`, `cmd/nvidia-mig-manager/main.go`, `README.md`](https://github.com/NVIDIA/mig-parted) — read on `main` at the v0.14.x line (v0.14.5 is the newest CHANGELOG entry). The complete reconfiguration state machine in §6, the six `gpu.deploy.*` labels and the `paused-for-mig-change` value, the five-minute per-operand wait, the `WITH_REBOOT` / `WITH_SHUTDOWN_HOST_GPU_CLIENTS` flags, the `nvidia.com/mig.config` label watch, the per-node `<nodeName>-mig-config` ConfigMap, and the `apply` / `export` / `assert` CLI. **This is the source that corrects the "mig-manager cordons and drains the node" claim** in the previous version of this lesson — it does neither.
- [NVIDIA/gpu-operator — `assets/state-mig-manager/{0400,0410,0420}_configmap.yaml`, `0600_daemonset.yaml`](https://github.com/NVIDIA/gpu-operator) — read at tag `v26.3.3` and on `main`. The full 625-line `default-mig-parted-config` (every named geometry for A30 through GB300, the `+me` singleton counts, the PCI-device-filtered per-SKU variants, and the `all-balanced` definitions used in §2), the `default-gpu-clients` service list stopped during a mode change, and the mig-manager DaemonSet's mounts and environment.
- [NVIDIA/gpu-operator — `api/nvidia/v1/clusterpolicy_types.go`, `deployments/gpu-operator/values.yaml`](https://github.com/NVIDIA/gpu-operator) — `MIGSpec.Strategy` validated as `none|single|mixed`, the `MIGManagerSpec` fields including `config` and `gpuClientsConfig`, the chart default `mig.strategy: single`, `migManager.config.default: "all-disabled"`, the note that MIG manager auto-generates config from discovered hardware, and the shipped example geometries (`3g.40gb + 4g.40gb`, `2g.20gb ×2 + 3g.40gb`). Pinned versions at v26.3.3: device-plugin `v0.19.3`, mig-manager `v0.14.2`.
- [NVIDIA/k8s-device-plugin — `README.md`, `docs/gpu-feature-discovery/README.md`](https://github.com/NVIDIA/k8s-device-plugin) — read on `main` at the v0.19.x line (v0.19.3 newest in CHANGELOG). The `--mig-strategy` / `MIG_STRATEGY` option and the `nvidia.com/mig-<slice_count>g.<memory_size>gb` resource-name form; the complete GFD label catalogue for `single` and `mixed` reproduced in §7; the statement that **"Sharing with MPS is currently not supported on devices with MIG enabled"** and that MPS is experimental as of v0.15.0; and the confirmation that time-slicing *is* supported on `mixed`-strategy MIG resource names. Note the README still shows a stale `v0.17.1` in one install example — that is not the current version.
- [NVIDIA/dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter) — read at tag `4.6.0-4.8.3`. `internal/pkg/rendermetrics/render_metrics.go` (the MIG label set: parent `UUID` plus `GPU_I_PROFILE` and `GPU_I_ID`, and **no** MIG-UUID label), `internal/pkg/collector/types.go` (`GetIDOfType` producing the `"<gpu>-<GPU_I_ID>"` join key), `internal/pkg/transformation/kubernetes.go` (the pod-resources→NVML→GI-identifier path), `internal/pkg/nvmlprovider/provider.go` (`GetMIGDeviceInfoByID` and both MIG UUID formats, pre- and post-R470). **Correction to the previous version of this lesson**, which claimed DCGM keys MIG metrics by the MIG UUID and that the pod-resources join is therefore direct. It is not.
- [kubelet pod-resources proto — `k8s.io/kubelet/pkg/apis/podresources/v1`](https://github.com/kubernetes/kubelet/tree/master/pkg/apis/podresources) — the `ContainerDevices{resource_name, device_ids}` shape that carries the MIG UUID into your exporter. Covered in depth in lesson 04.3.

**Deeper dives**

- [Lesson 04.3 — Device-plugin recap & the kubelet pod-resources API](03-device-plugin-recap-pod-resources.md) — the client this lesson's attribution chain runs through, and the device-ID-shape table §8 extends.
- [Lesson 04.7 — Time-slicing and the attribution trap](07-time-slicing-attribution.md) — the same join with the metric identity removed; read §6 of that lesson directly against §8 of this one.
- [Lesson 04.10 — Capstone: per-pod GPU attribution](10-capstone-per-pod-attribution.md) — where the `attribution="exact"` MIG path and the stranded-capacity bucket from §8 become code.
