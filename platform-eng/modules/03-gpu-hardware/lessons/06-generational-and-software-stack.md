---
lesson: "03.6"
title: "Generational Differences & the Software Stack"
module: "03"
concept: "SKU generations & driver/CUDA/NCCL stack hygiene"
status: not-started
est_time: "6h"
prev: "05-precision-and-tensor-cores.md"
next: "07-capstone-cost-per-useful-work.md"
artifacts: []
sources: 14
---

# 03.6 · Generational Differences & the Software Stack

> **Concept.** Pick the right generation from the memory/bandwidth/FP8/MIG table, then keep the driver↔CUDA↔cuDNN/NCCL↔container-toolkit stack pinned — because version drift is the silent failure mode, and a throttled GPU silently spends your budget.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 5 established that precision support is gated by hardware generation — FP8 needs Hopper or newer, FP4 needs Blackwell or newer — and gave you the mechanism behind it: the per-SM tensor throughput (`FLOP/SM/clock`) that doubles per generation and again per format halving, but only on silicon that has the narrower datapath. That fact was a footnote there; here it becomes the spine. This lesson turns "which generation" into two deliverables you'll actually write in a job: a **placement/purchasing argument** (which SKU removes this workload's specific bottleneck, and at what cost delta) and a **stack inventory** (the driver/CUDA/NCCL chain that has to be right for any of the previous five lessons' numbers to be real on the hardware you land). It closes the module's hardware-literacy arc — lesson 7's capstone assumes you can both pick a generation and keep it running without version drift silently eating your budget.

## Why this matters

Two things quietly waste GPU budget in production, and neither shows up as an error message.

The first is **buying or scheduling the wrong generation**. A 70B inference server pinned to H100s "because that's what we have" may be paying for tensor-parallel all-reduce overhead that an H200's larger memory would eliminate outright — or, in the other direction, paying Blackwell prices for a workload an Ampere fleet already serves fine. You make these arguments from a table of memory, bandwidth, FP8/FP4 support, and MIG partitioning, tied to the workload's actual bottleneck — not from "the newest one is always better."

The second is **version drift in the software stack**. The driver, CUDA toolkit, cuDNN, NCCL, and the container toolkit are a versioned chain, and getting it wrong rarely produces a clean failure. It produces a container that won't start, or worse, an NCCL mismatch that *runs* but hangs on collective ops or silently drops to a slow fallback path. On a multi-node training job burning hundreds of dollars an hour, a silent performance regression is money leaving the building in real time, with no alert firing. This is not a hypothetical: Imbue's public infrastructure write-up (below) describes exactly this class of failure on a real 70B training run, and the fix was counterintuitive — a driver **downgrade**, not an upgrade.

There is a structural reason these failures are quiet. The GPU stack is unusual among Linux subsystems in that the *same logical library* is split across two administrative domains: `libcuda.so` ships with the **host driver** and is owned by whoever runs your nodes, while `libcudart.so`, cuBLAS, cuDNN and NCCL ship inside your **container image** and are owned by whoever builds it. Those two halves are versioned independently, released on different cadences, and are only guaranteed to work together within documented bands. Nothing in Kubernetes, Docker, or your CI checks the band for you. Treating this chain as an operational surface — pinned, inventoried, canaried before any bump — is core platform-engineering hygiene, and it's precisely the "hardware validation across the fleet" and "controllers to automate infra testing" language in CoreWeave's GPU Performance Engineer posting that this module's README opens with.

## What's new here (calibration)

The module README skips deep microarchitecture and NVLink/PCIe topology, node layout, and power/thermal throttling mechanics — those are 02b's territory, referenced here, not re-taught. This lesson does not re-explain how throttling works at the hardware level; it only reminds you (below) why a throttled card matters to the cost numbers this module builds. What it adds instead:

- **A precise, generation-spanning CUDA/driver compatibility table** — not just "CUDA 12 needs driver 525," but the exact minimum driver versions per major family, the *upper* bound on the minor-version-compatibility band, which driver branches are supported and for how long, and the specific error codes each violation produces.
- **MIG's actual partitioning mechanics** — "up to 7 instances" is the number everyone quotes; the 7-compute-slice / 8-memory-slice structure, the real profile menu for A100 and H100, the GPU-Instance vs Compute-Instance two-level hierarchy, and the `nvidia-smi mig` commands to create them are the depth a Staff-level answer needs.
- **The driver as a two-domain object** — why `libcuda.so` (host) and `libcudart.so` (container) are versioned separately, what forward compatibility actually installs on disk, and why forward compatibility is datacenter-GPU-only.
- **A named, real production incident for the abstract "version drift" warning** — Imbue's driver-downgrade fix, which directly contradicts the naive "always update" instinct and gives you a war story to reason from instead of a rule to recite.
- **Explicit handling of a live spec disagreement** — the B200's memory/bandwidth numbers are genuinely contested across vendor-mirror sources at the time of writing; this lesson teaches you to flag that rather than fabricate false precision, which is itself a skill a fast-moving hardware market demands.

## Core concepts

### 1. The cross-generation table

Dense tensor-core throughput throughout (2:4 structured sparsity roughly doubles the marketed figures; lesson 5 §7 explains why). Every number here is a **2026 dated snapshot** — verify against the current datasheet at build time, especially for Blackwell, where sources disagree (see the flag below).

| GPU (SXM) | Arch | Memory | Bandwidth | BF16/FP16 dense | FP8 dense | FP4 dense | FP64 TC | MIG |
|---|---|---|---|---|---|---|---|---|
| A100 80GB | Ampere | 80 GB HBM2e | 2,039 GB/s | 312 TFLOPS | — (no FP8 hardware) | — | 19.5 TFLOPS | up to 7 |
| H100 | Hopper | 80 GB HBM3 | 3,350 GB/s | 989.5 TFLOPS | 1,979 TFLOPS | — | 67 TFLOPS | up to 7 |
| H200 | Hopper | 141 GB HBM3e | 4,800 GB/s | 989.5 TFLOPS | 1,979 TFLOPS | — | 67 TFLOPS | up to 7 |
| B200 | Blackwell | **180–192 GB HBM3e (verify SKU)** | **~7.7–8.0 TB/s (verify SKU)** | ~2,250 TFLOPS | ~4,500 TFLOPS | ~9,000 TFLOPS | ~40 TFLOPS | up to 7 |

**On the B200 row specifically: do not treat those numbers as settled.** Credible sources disagree on whether a "B200" is 180, 186, or 192 GB of HBM3e, and whether bandwidth is ~7.7 or ~8.0 TB/s. The most likely explanation is that different sources describe different things under the same marketing name — the bare B200 GPU module (commonly 180 GB / ~7.7 TB/s in NVIDIA's own datasheet and the DGX B200 user guide) versus the board-level HGX B200 (commonly 186 GB / 8 TB/s) versus later B300 variants — but that could not be resolved to one authoritative figure through public documentation alone at the time this lesson was written. The correct professional move is not to pick the number that looks more precise; it's to **state the range and explicitly flag "confirm exact SKU variant at build/purchase time"** in any report or purchasing argument you write. A false-precision single number that turns out wrong in a procurement conversation costs you credibility a stated range never will.

Now read the rest of the table by *what changed generation to generation*, because that is the shape of a purchasing argument:

**A100 → H100: a precision and compute jump, not a capacity one.** Capacity is identical (80 GB). Bandwidth rises 1.64× (2,039 → 3,350 GB/s). BF16 compute rises 3.17× (312 → 989.5 TFLOPS), which decomposes as `(132/108 SMs) × (4096/2048 FLOP/SM/clk) × (1.83/1.41 GHz) = 1.22 × 2 × 1.30 = 3.17` — more SMs, a doubled tensor datapath, and a higher clock, in that order of importance. And FP8 appears as a wholly new precision tier that has no equivalent on Ampere. **If your workload is capacity-constrained, A100 → H100 does nothing for you.**

**H100 → H200: the same silicon with different memory attached.** This is the generational step people misread most often, so state it precisely: *H200 is the same GH100 compute die as H100.* Same 132 SMs, same 4th-gen tensor cores, same 989.5 BF16 / 1,979 FP8 dense TFLOPS. What changed is the HBM packaging. The GH100 package has six HBM sites; H100 populates five with 16 GB HBM3 stacks (80 GB, ~3.35 TB/s), while H200 populates six with 24 GB HBM3e stacks — **144 GB installed, 141 GB usable** (the ~3 GB delta is reserved), at ~4.8 TB/s. So:

```
  H100 → H200 deltas, exactly:
     compute      1.00×   (identical die, identical clocks, identical TFLOPS)
     bandwidth    1.43×   (3,350 → 4,800 GB/s)
     capacity     1.76×   (80 → 141 GB)

  Which workloads care?
     compute-bound prefill / training GEMMs ........ ZERO benefit
     bandwidth-bound decode ........................ up to 1.43× directly
     capacity-bound (model doesn't fit, or KV cache
       is the batch-size limiter) .................. the big one — see §2
```

**H200 → B200: capacity *and* compute *and* a new precision tier.** Roughly 1.6–1.7× bandwidth (pending the exact-SKU caveat), ~2.3× FP8 compute, plus native FP4 with block scaling (lesson 5 §9), which is another ~2× again for inference that tolerates it. This is the full jump, which is why it is priced and positioned differently from the H100→H200 step. Note the one regression in the table: FP64 tensor-core throughput *fell* from Hopper's 67 TFLOPS to roughly 40 TFLOPS on B200 — Blackwell reallocated die area toward low-precision AI math and away from double-precision HPC. If you run traditional scientific codes alongside AI, "newer is better" is straightforwardly false for that half of your fleet.

### 2. Turning the table into a placement decision

The table is only useful if you can map a workload's bottleneck onto a column. Do it in this order:

```
  WHICH GENERATION? — decide by bottleneck, not by newness

  ┌───────────────────────────────────────────────────────────┐
  │ 1. Does the model + KV cache + workspace FIT at your      │
  │    target precision on ONE GPU of the candidate SKU?      │
  └───────────────┬───────────────────────────┬───────────────┘
             NO   │                           │  YES
                  ▼                           ▼
   ┌────────────────────────────┐   ┌──────────────────────────────┐
   │ You are paying a TP tax:   │   │ 2. Is the hot path compute-  │
   │ every token crosses an     │   │    bound or memory-bound?    │
   │ all-reduce. Capacity is    │   │    (lesson 2's roofline)     │
   │ your binding constraint →  │   └──────┬──────────────┬────────┘
   │ buy CAPACITY (H100→H200,   │   memory │              │ compute
   │ or drop precision first —  │          ▼              ▼
   │ lesson 5 is cheaper than   │   ┌──────────────┐  ┌──────────────┐
   │ new silicon).              │   │ buy BANDWIDTH│  │ buy FLOPS or │
   └────────────────────────────┘   │ (H200, B200) │  │ a lower      │
                                    │ compute gen  │  │ PRECISION    │
                                    │ won't help   │  │ tier (H100→  │
                                    └──────────────┘  │ B200; FP8;   │
                                                      │ FP4)         │
                                                      └──────────────┘

  ┌───────────────────────────────────────────────────────────────────┐
  │ 3. Is the GPU too BIG for the job? (small model, low QPS, high    │
  │    idle) → don't buy a smaller GPU; partition the one you have    │
  │    with MIG (§3) and pack tenants onto it.                        │
  └───────────────────────────────────────────────────────────────────┘
```

The reason step 1 comes first is that capacity is a *cliff*, not a slope. Below the line the model simply does not load; just above it, you keep all the compute and pay nothing for communication. Bandwidth and FLOPS are continuous — 1.43× more bandwidth buys you up to 1.43× more decode throughput — but capacity buys you a discrete structural change: **tensor parallelism disappears.** Every token in a TP=2 deployment pays two all-reduces per transformer layer; on an 80-layer model that is 160 collective operations per token, each with its own latency floor. Removing them is worth more than the arithmetic suggests, and it also removes an entire class of failure (NCCL, §6) from your serving path.

### 3. MIG — the precise partitioning mechanics

**MIG** (Multi-Instance GPU) partitions one physical GPU into up to seven hardware-isolated instances. It was introduced with Ampere and is supported on Hopper (H100, H200) and Blackwell as well.

The number worth knowing precisely, not just "up to 7": each MIG-capable GPU exposes **7 compute slices and 8 memory slices**. A compute slice is a fixed fraction of the SMs; a memory slice is a fixed fraction of the HBM *and* of the L2 cache and memory-controller bandwidth that serves it. Instances are assembled from a menu of valid `(compute, memory)` combinations — which is why the two counts differ. The eighth memory slice exists because the memory system partitions into 8 equal pieces naturally (following the HBM stack/controller layout) while the SM array partitions into 7 usable groups; not every compute-slice count maps onto an equal share of memory.

The real profile menu, for an A100 80GB and an H100 80GB:

| Profile | Compute slices | Memory | SMs (A100 / H100) | Max concurrent instances |
|---|---|---|---|---|
| `1g.10gb` | 1 | 10 GB | 14 / ~16 | 7 |
| `1g.20gb` (H100) | 1 | 20 GB | ~16 | 4 |
| `2g.20gb` | 2 | 20 GB | 28 / 32 | 3 |
| `3g.40gb` | 3 | 40 GB | 42 / 60 | 2 |
| `4g.40gb` | 4 | 40 GB | 56 / 64 | 1 |
| `7g.80gb` | 7 | 80 GB | 98 / 132 | 1 (the whole GPU) |

Note `3g.40gb` and `4g.40gb` both get 40 GB — three compute slices paired with four memory slices, or four with four. That asymmetry is exactly what the 7-vs-8 split makes possible, and it is the detail that separates "I read the marketing page" from "I have configured this."

There is a second layer people forget. A MIG device is really **two nested objects**: a *GPU Instance* (GI) — memory slices plus compute slices plus dedicated engines (copy engines, NVDECs) — and inside it a *Compute Instance* (CI), which subdivides the GI's SMs for separate CUDA contexts. By default you get one CI spanning the whole GI, which is why most people never notice. You can subdivide a `3g.40gb` GI into three `1c.3g.40gb` CIs that share the GI's 40 GB but get isolated SM groups.

Creating them, on a GPU with no running processes:

```
$ sudo nvidia-smi -i 0 -mig 1              # enable MIG mode (needs a GPU reset)
$ nvidia-smi mig -lgip                     # list GPU-instance profiles + free slots
+-----------------------------------------------------------------------------+
| GPU  Name              ID    Instances   Memory     P2P    SM    DEC   ENC   |
|                              Free/Total   GiB              CE    JPEG  OFA   |
|=============================================================================|
|  0   MIG 1g.10gb       19     7/7        9.75       No     14    0     0     |
|  0   MIG 2g.20gb       14     3/3        19.62      No     28    1     0     |
|  0   MIG 3g.40gb        9     2/2        39.50      No     42    2     0     |
|  0   MIG 7g.80gb        0     1/1        79.25      No     98    5     0     |
+-----------------------------------------------------------------------------+
     (representative output — A100-SXM4-80GB; exact SM/DEC counts vary by SKU)

$ sudo nvidia-smi mig -cgi 9,14,19,19 -C   # create GIs by profile ID, -C also
                                           # creates the default compute instance
$ nvidia-smi -L
GPU 0: NVIDIA A100-SXM4-80GB (UUID: GPU-...)
  MIG 3g.40gb Device 0: (UUID: MIG-...)
  MIG 2g.20gb Device 1: (UUID: MIG-...)
  MIG 1g.10gb Device 2: (UUID: MIG-...)
  MIG 1g.10gb Device 3: (UUID: MIG-...)
```

**Why the isolation guarantee is stronger than it sounds.** This is *hardware* partitioning, not a software time-slicing or cgroups-style scheme. Each instance gets dedicated SMs, a dedicated slice of L2, and a dedicated share of memory-controller bandwidth. There is no shared scheduling queue between instances the way there is with plain GPU time-slicing or MPS (Multi-Process Service). The practical consequence, stated precisely: **a misbehaving or noisy tenant on one MIG instance cannot starve another instance's throughput, and a fault in one instance is contained to it.** Under MPS, by contrast, all clients share one scheduler and one memory system, so a client that floods the GPU with work does degrade its neighbours.

The costs, which you must also be able to name: MIG instances **cannot communicate over NVLink with each other** and cannot be aggregated, so no multi-GPU job can span them; enabling or reconfiguring MIG requires no processes on the GPU (in practice, draining the node); and the partition menu is fixed, so you cannot ask for "3.5 slices." MIG is the right lever for packing many small, independent inference tenants onto one card with a real isolation guarantee. It is the wrong lever for anything that wants the whole GPU some of the time.

### 4. The versioned software chain

From the metal up, with the ownership boundary marked — because that boundary is where the failures live.

```
  THE CUDA STACK, AND WHERE THE COMPATIBILITY BOUNDARIES SIT

  ┌───────────────────────────────────────────────────────────────────────┐
  │  YOUR APPLICATION                    (training script, vLLM, triton)  │
  └───────────────────────────────┬───────────────────────────────────────┘
                                  │  python / C++ API
  ┌───────────────────────────────▼───────────────────────────────────────┐
  │  FRAMEWORK        torch 2.x, JAX, TensorRT-LLM                        │
  │  pins: the CUDA minor it was BUILT against; the cuDNN & NCCL it ships │
  └───────────────────────────────┬───────────────────────────────────────┘
  ┌───────────────────────────────▼───────────────────────────────────────┐
  │  MATH / COLLECTIVE LIBS   cuBLAS · cuDNN · NCCL · cuFFT · CUTLASS     │
  │  cuDNN: own version line (9.x), own CUDA-major requirement            │
  │  NCCL:  own version line (2.x) — MUST MATCH ACROSS EVERY RANK  ◀══════╪══ (A)
  └───────────────────────────────┬───────────────────────────────────────┘
  ┌───────────────────────────────▼───────────────────────────────────────┐
  │  CUDA RUNTIME API      libcudart.so.<major>.<minor>                   │
  │  statically or dynamically linked INTO your app / image               │
  └───────────────────────────────┬───────────────────────────────────────┘
  ══════════════════ CONTAINER IMAGE ═══╪═══ HOST ══════════════════════════
                                        │           ◀═══════════════════════ (B)
  ┌─────────────────────────────────────▼─────────────────────────────────┐
  │  CUDA DRIVER API       libcuda.so.<driver-version>                    │
  │  SHIPS WITH THE DRIVER. Injected into the container at runtime by     │
  │  the NVIDIA Container Toolkit — the image does NOT contain it.        │
  └───────────────────────────────┬───────────────────────────────────────┘
  ┌───────────────────────────────▼───────────────────────────────────────┐
  │  KERNEL MODULES   nvidia.ko, nvidia-uvm.ko, nvidia-drm.ko             │
  │  version MUST equal the user-space driver exactly              ◀══════╪══ (C)
  └───────────────────────────────┬───────────────────────────────────────┘
  ┌───────────────────────────────▼───────────────────────────────────────┐
  │  GPU + VBIOS/firmware                                                 │
  └───────────────────────────────────────────────────────────────────────┘

  THE THREE BOUNDARIES THAT BREAK:
   (A) NCCL across ranks .......... MUST MATCH EXACTLY across all nodes.
                                    Skew → hang or silent slow transport.
   (B) runtime ↔ driver ........... FORWARD-COMPATIBLE within documented
                                    bands (§5). Driver may be NEWER than the
                                    toolkit, never older than the minimum.
   (C) kernel module ↔ user-space
       driver ..................... MUST MATCH EXACTLY. A partial package
                                    upgrade that replaces one and not the
                                    other bricks the GPU until reboot.
```

Layer by layer, with what each actually is:

1. **NVIDIA driver.** A kernel module (`nvidia.ko`, plus `nvidia-uvm.ko` for unified memory and `nvidia-drm.ko`) and a matching user-space half whose most important component is **`libcuda.so`** — the CUDA *driver API*. Versions look like `535.183.06`, `570.124.06`, `580.65.06`. The first three digits are the **branch** (R535, R570, R580).
2. **CUDA toolkit.** `nvcc`, the *runtime API* (`libcudart`), the math libraries, and the headers. Versions look like `12.4`, `13.0`. This is what your image contains.
3. **cuDNN / NCCL / cuBLAS.** cuBLAS ships inside the toolkit. cuDNN and NCCL have independent version lines (cuDNN 9.x, NCCL 2.x) and their own CUDA-major requirements. **NCCL** drives multi-GPU/multi-node all-reduce, all-gather, reduce-scatter — the backbone of distributed training.
4. **NVIDIA Container Toolkit.** The piece that makes the split work. Historically it installed an OCI *prestart hook* (`nvidia-container-runtime-hook`, backed by `libnvidia-container`) that, at container start, bind-mounts the host's driver libraries and device nodes into the container's filesystem. Modern versions can instead operate in **CDI mode** (Container Device Interface): `nvidia-ctk cdi generate` writes a spec file describing the device, and a CDI-aware runtime performs the injection with no hook at all. Either way, the invariant is the same: **the driver comes from the host, everything above it comes from the image.**

**Why this split exists at all** is worth one sentence, because it explains the whole compatibility model: the kernel module must match the running kernel and the physical hardware, so it cannot live in a portable image; but the toolkit and libraries are what your application was *compiled* against, so they must travel with it. NVIDIA's answer is to guarantee that a *newer* driver runs *older* toolkits, and to define bands within which the reverse is also allowed.

### 5. The CUDA/driver compatibility rules, with concrete version pairs

Three distinct rules, often conflated. Learn them separately.

**Rule 1 — Backward compatibility (always true).** A newer driver runs applications built with any older CUDA toolkit. `libcuda.so` maintains a stable ABI; a driver from 2026 still runs a binary compiled with CUDA 11.0. This is why "upgrade the driver" is the usual advice — and why it is not always right (§8).

**Rule 2 — Minor version compatibility (within a major family, CUDA 11 onward).** An application built with *any* minor toolkit in a major family runs on *any* driver that meets the family's **minimum driver version**, with no extra packages. This is the rule that matters day to day:

| CUDA toolkit family | Minimum Linux driver | Minimum Windows driver | Minor-compat band | Notes |
|---|---|---|---|---|
| CUDA 11.x | **≥ 450.80.02** (R450) | ≥ 452.39 | R450 ≤ driver < R525 | The family that introduced minor-version compatibility |
| CUDA 12.x | **≥ 525.60.13** (R525) | ≥ 527.41 | R525 ≤ driver < R580 | The long-lived family; most 2023–2025 images |
| CUDA 13.x | **≥ 580.65.06** (R580) | (see release notes) | R580 and newer | Current major family as of mid-2026 |

Read a concrete case: an application built with **toolkit 12.4** running on a host with driver **525.60.13** works, because 12.4 is in the CUDA 12 family and 525.60.13 meets that family's minimum. It works *without* installing anything extra. The caveat NVIDIA attaches — "with a limited feature set" — means only that APIs introduced after 12.0 will return `cudaErrorCallRequiresNewerDriver` if you call them; the rest of your program is fine. In practice, frameworks guard these calls.

Now read the case that fails: an application built with **toolkit 13.0** on that same 525 host. CUDA 13.x requires R580. You get:

```
  CUDA error 35 · cudaErrorInsufficientDriver
  "CUDA driver version is insufficient for CUDA runtime version"
```

at the first CUDA call — usually surfacing in PyTorch as `RuntimeError: CUDA error: ... insufficient driver` or, unhelpfully, as `torch.cuda.is_available() == False` with no explanation at all. **The symptom to recognise: the container starts, `nvidia-smi` inside it works fine (that binary talks to the driver directly), and only your framework claims there is no GPU.** `nvidia-smi` working is not evidence that CUDA works.

**Rule 3 — Forward compatibility (across major families, for frozen datacenter drivers).** When the host is stuck on an *older* driver from a *different* major family — common on locked-down fleets that cannot touch the driver outside a maintenance window — you install the **`cuda-compat-<major>-<minor>`** package. It is not magic: it installs a *newer* `libcuda.so` into `/usr/local/cuda-<X.Y>/compat/`, and you put that directory ahead of the system path via `LD_LIBRARY_PATH` (the NVIDIA CUDA container images do this for you). The kernel module stays at the host's version; the user-space driver library is upgraded in place for that one process tree.

Two constraints that catch people:

- **Forward compatibility is supported on datacenter-class GPUs only** (Tesla/data-center branded — A100, H100, L40S, and so on). Attempting it on a GeForce card produces:
  ```
  CUDA error 804 · cudaErrorCompatNotSupportedOnDevice
  "forward compatibility was attempted on non supported HW"
  ```
  This error message is diagnostic gold: it means the `cuda-compat` libraries *were* found and loaded, and the hardware refused them. If you see it in a container on a workstation, the fix is to remove the compat path from `LD_LIBRARY_PATH`, not to add more packages.
- Forward compatibility does not let you use hardware the old kernel driver doesn't know about. A CUDA 13 app forward-compatted onto an R535 host still cannot drive a GPU that R535 never supported.

**Rule 4 — the kernel/user-space match (absolute).** `nvidia.ko` and `libcuda.so` must be the *same* version. A partial package upgrade — the very common case where `apt upgrade` replaces the user-space libraries while the old module is still loaded — gives:

```
  Failed to initialize NVML: Driver/library version mismatch
  NVML library version: 570.124
```

with the running module reporting a different version. Nothing works until the module is reloaded (`rmmod nvidia_uvm nvidia_drm nvidia_modeset nvidia`, which requires every process holding the GPU to exit) or the node reboots. On a Kubernetes node, this means a cordon and drain. **This is the single most common self-inflicted GPU outage**, and it is why unattended-upgrades on GPU nodes should exclude the NVIDIA packages.

**Driver branch lifecycle — because "pin it" needs a target.** NVIDIA ships three kinds of datacenter driver branch: New Feature Branch (short-lived), Production Branch (~1 year of fixes), and Long Term Support Branch (~3 years). As of this 2026 snapshot:

| Branch | Type | CUDA family | End of life |
|---|---|---|---|
| R535 | LTSB | CUDA 12.x | June 2026 |
| R570 | Production | CUDA 12.x | February 2026 |
| R580 | LTSB | CUDA 13.x | August 2028 |

Everything older — R575, R565, R560, R550, R525, R470 and back — is end of life. **Pin to an LTSB unless you have a concrete reason not to**: it gives you a three-year window in which security fixes arrive without forcing a functional change. That, not "the latest," is what "pin everything" means.

### 6. NCCL: the layer that hangs instead of erroring

NCCL rides along with the toolkit but has its own version line, its own constraints, and — uniquely in this stack — a **cross-node** requirement. Every rank in a job negotiates a common set of algorithms, protocols, and transports at initialisation. That negotiation assumes all ranks are running the same library.

The failure mode is the one that costs the most money, because it is silent:

```
  WHAT A COLLECTIVE MISMATCH LOOKS LIKE, IN ORDER

   t=0     job launches, all ranks call ncclCommInitRank()
           bootstrap over TCP: exchange unique ID, ranks, topology
   t=+2s   ranks agree (or fail to agree) on: algorithm (ring / tree /
           NVLS), protocol (Simple / LL / LL128), transport (NVLink /
           PCIe P2P / SHM / IB / TCP sockets)
   t=+3s   first ncclAllReduce() is issued on every rank
              │
              ├── all ranks agree ────────▶ normal steady state
              │
              ├── ranks disagree on the
              │   COLLECTIVE (different op,
              │   different size, different
              │   order) ─────────────────▶ ██ HANG ██  every rank blocks
              │                              in a CUDA kernel waiting for
              │                              peers that will never arrive.
              │                              No error. No exit code. The
              │                              job just stops making progress.
              │
              └── ranks agree, but a
                  transport was rejected
                  (IB down, GDR disabled,
                  crossNic mismatch) ─────▶ ██ SLOW ██  falls back to
                                             sockets/SHM. Runs correctly at
                                             a fraction of the bandwidth.
                                             Nothing logs a warning at
                                             default NCCL_DEBUG.

   t=+30m  PyTorch's ProcessGroupNCCL watchdog fires
           (default TORCH_NCCL_BLOCKING_WAIT off; timeout 10 min in recent
           versions, 30 min historically) — "Watchdog caught collective
           operation timeout" — the FIRST signal you get, long after the
           money was spent.
```

The evidence that this is real and not a warning invented for a lesson: publicly filed issues [NVIDIA/nccl#584](https://github.com/NVIDIA/nccl/issues/584) ("NCCL hangs during ncclSend and ncclRecv") and [pytorch/pytorch#43546](https://github.com/pytorch/pytorch/issues/43546) both document exactly this hang-not-error behaviour. **In both of those filed issues the trigger was a mismatched or out-of-order collective call across ranks — not a driver/toolkit version skew** — but the symptom class is identical, which is why they are worth reading: a version skew that causes ranks to disagree on transport or protocol produces the same silent stall with the same absence of a clean error.

NCCL's own release notes carry the same lesson in fixed-bug form. **NCCL 2.21.5** fixed a mismatch crash during bootstrap caused by TCP packet reordering, and a hang on heterogeneous systems caused by the `crossNic` value differing between nodes — a configuration difference, not a code bug, producing a hang. That is the shape of every NCCL incident you will investigate: *something differed between ranks, and the result was a stall rather than a message.*

Your operational defences, in order of value:

```
export NCCL_DEBUG=INFO            # prints the chosen algo/proto/transport per rank
export NCCL_DEBUG_SUBSYS=INIT,GRAPH,ENV
export TORCH_NCCL_ASYNC_ERROR_HANDLING=1   # turn watchdog timeouts into a crash
export TORCH_NCCL_BLOCKING_WAIT=0          # (keep async; blocking wait is slower)
```

With `NCCL_DEBUG=INFO`, every rank prints its NCCL version and the transport it selected. **Diffing that output across ranks is the single highest-value five minutes you can spend on a distributed job that is "mysteriously slow"** — a rank that says `via SHM/direct/direct` where its peers say `via NET/IB/0` is your answer.

### 7. The stack-hygiene rules, and what each one prevents

| Rule | Prevents | The failure it prevents looks like |
|---|---|---|
| Pin the container image **by digest**, not by tag | A `:latest` or `:24.10` tag being re-pushed under you | Yesterday's job worked, today's identical job doesn't; `docker pull` is the only thing that changed |
| Pin toolkit / cuDNN / NCCL versions explicitly in the image | Transitive dependency drift on rebuild | A rebuilt image with no source changes performs 20% differently |
| Pin the **host driver** to an LTSB, exclude it from unattended upgrades | Partial upgrades and out-of-band branch jumps | `Failed to initialize NVML: Driver/library version mismatch`, node-wide |
| Upgrade NCCL on **all ranks simultaneously** | Cross-rank skew | Hang on the first collective, or silent fallback to sockets |
| Canary a driver bump on one node before the fleet | Fleet-wide blast radius | Every container on every node fails at once |
| Record a **stack inventory** per node and diff it | Undetected drift between "identical" nodes | One node in a 64-node job is 3× slower and nobody knows which |

**The driver has fleet-wide blast radius, and nothing else in the stack does.** Every container on a node shares the one host driver. A bad image affects one job; a bad driver affects every job on every node you rolled it to. That asymmetry is the whole argument for canarying driver changes and for treating them as a different class of change from an image bump.

### 8. Version drift is bidirectional: the Imbue incident

The naive rule is "keep the driver current." Imbue's team, training a 70B model from scratch on their own bare-metal cluster, published a warts-and-all infrastructure write-up in which the resolution of a training-stall problem was to **downgrade** the NVIDIA driver to a previously verified-good version. Their health-checking regime — which is the transferable part — covered:

- **InfiniBand error rates** on every link, because a fabric that is up but erroring degrades collectives without failing them.
- **Driver and firmware versions**, audited against the pairing they had validated, rather than against "the newest available."
- **Xid and SXid errors** from the GPUs and NVSwitches (see §9), read as the driver's own hardware-fault channel.
- **GPUDirect RDMA** functioning end to end, because a silent GDR failure is precisely the "runs, but at socket speed" case from §6.

The transferable conclusion is not "downgrade drivers." It is: **"pin everything, never `latest`" means pin to a known-good, *tested* version, and treat movement in either direction as a change that needs a canary and a rollback plan.** "Pin to the newest" and "pin to the latest verified-good" are different policies. Imbue's incident is the concrete case where they diverged and only one of them worked.

Together AI's cluster-testing guide is the complement worth reading alongside it: where Imbue documents what broke, Together AI documents what to check *before* it breaks — a systematic validation playbook across driver, fabric, and NCCL health.

### 9. Xid errors: the driver's hardware-fault channel

`Xid` messages are how the NVIDIA kernel driver reports GPU faults. They land in the kernel ring buffer, and they are the cheapest health signal you have — free, always on, and completely ignored by most teams.

```
$ sudo dmesg -T | grep -i xid
[Tue Aug 11 03:14:22 2026] NVRM: Xid (PCI:0000:1b:00): 94, pid=12833, Contained: CE...
[Tue Aug 11 09:41:07 2026] NVRM: Xid (PCI:0000:1b:00): 63, pid='<unknown>', Row Remapper...

$ nvidia-smi -q | grep -A 12 -i "ecc errors"      # the counters behind Xid 48/94/95
$ nvidia-smi -q | grep -i -A 4 "remapped rows"    # the counters behind Xid 63/64
```

The codes worth recognising on sight:

| Xid | Meaning | Severity / action |
|---|---|---|
| **13, 31** | Graphics/compute engine exception — an illegal memory access by a kernel | Usually a **software** bug in the workload (the GPU equivalent of a segfault). Not a hardware condemnation. |
| **43** | GPU stopped processing | Usually workload software |
| **48** | **Double-bit ECC error** — uncorrectable | **Fatal.** The GPU is dying. Drain the node, RMA the card. |
| **63, 64** | Row remapping event / failure | Not urgent alone, but a **rising count is a trend toward failure**. Track it; plan replacement. |
| **74** | NVLink / interconnect error | Investigate the link and the NVSwitch; commonly precedes wider fabric trouble |
| **79** | **GPU has fallen off the bus** — PCIe unreachable | **Fatal.** Often power, seating, or thermal. Node out of service. |
| **94** | Contained ECC error — corrected, contained to one process | Informational. Data was corrected; no other process affected. |
| **95** | **Uncontained ECC error** | **Workload outputs may be corrupt.** Kill and rerun the job; do not trust the checkpoint. |

The distinction between 94 and 95 is the one to internalise: 94 says "the ECC hardware did its job"; 95 says "the error escaped containment and your results may be wrong." A training run that saw an Xid 95 should have its checkpoint discarded, and that is a call you can only make if you were reading the channel.

Establishing what a *clean* baseline looks like is the point of §Practice's health check. An empty `dmesg | grep -i xid` on a freshly rented box is what makes a non-empty one later mean something.

### 10. 15-minute callback to 02b: throttling silently lowers your roofline

You already learned (02b) how to read power/thermal throttling with `nvidia-smi -q -d PERFORMANCE` and `clocks_throttle_reasons`. It belongs in a *cost/roofline* module because of one identity from lesson 5:

```
  peak FLOPS = SMs × (FLOP/SM/clock) × CLOCK
                                        ▲
                                        └── throttling reduces THIS term,
                                            and therefore your real roofline
```

A card in `SW_Thermal_Slowdown` or `HW_Power_Brake` is running a *lower* roofline than the spec sheet you divided by. If an H100 SXM5 drops from 1,830 MHz to 1,400 MHz under a power cap, its real BF16 dense peak is `132 × 4096 × 1.40e9 = 757 TFLOPS`, not 989.5 — so a workload achieving 600 TFLOPS is at 79% of its *actual* ceiling while your report says 61% of spec and you go looking for a kernel problem that doesn't exist.

So: when DCGM tensor-active is high but throughput is below model, check `clocks_throttle_reasons` before concluding anything about your code. You may simply be renting a card that delivers fewer real FLOPS per hour than the datasheet promises. (Mechanics: see 02b — not re-taught here.)

## Perspectives

**Purchasing/placement view.** The H100→H200 argument is a pure capacity/bandwidth call — same GH100 die, same 989.5/1,979 dense TFLOPS, different memory packaging (five 16 GB HBM3 stacks vs six 24 GB HBM3e). A purchasing decision that reasons "H200 costs more per hour, so it's worse value" without checking whether the workload is memory- or capacity-bound gets the recommendation backwards; this is the single most common placement mistake at this level. The decision tree in §2 exists so that "which bottleneck?" precedes "which SKU?", never the reverse.

**Ops/reliability view.** The driver/CUDA/NCCL pin-everything discipline exists because the failure modes are silent, not loud. Imbue's team found training stalls that traced to driver/firmware drift relative to their InfiniBand fabric, and the fix was rolling the driver *backward*. "Pin everything" doesn't mean "never touch it"; it means every change is canaried and reversible, in either direction. The corollary from §7: the driver is the only component with fleet-wide blast radius, so it deserves a different change process from everything above it.

**Hardware view.** MIG's isolation is a silicon fact, not a scheduling policy: 7 compute slices and 8 memory slices, with dedicated SMs, L2 slices, and memory-controller bandwidth per instance. That is a materially stronger guarantee than MPS or time-slicing, where all clients share one scheduler. It also has hard limits worth naming in the same breath — no NVLink between instances, no aggregation, and reconfiguration requires draining the GPU.

**Economics view.** The H200-vs-H100 worked example below is the concrete version of "does the generation upgrade pay for itself" — and note that the answer depends on a ratio, `($/hr ratio) vs (throughput ratio)`, not on either number alone. The B200 spec ambiguity flagged in §1 is a live reminder that any such argument built today has a shelf life: a purchasing memo that hard-codes "192 GB, 8.0 TB/s" without a verification date is already a liability the moment a vendor updates a datasheet. And §10's throttling point is the economics of the same thing at the hour scale — you pay for rated clocks and receive achieved clocks.

## Real-world use cases

- **Imbue, ["From bare metal to a 70B model: infrastructure set-up and scripts"](https://imbue.com/research/70b-infrastructure/).** A small team's real, warts-and-all account of training a 70B model from scratch on their own cluster. They describe health-checking for InfiniBand error rates and **outdated driver firmware**, and directly resolved a training-stalling issue by **downgrading to a previous NVIDIA driver version** — the single strongest, most concrete piece of evidence in this entire module for "version drift is a silent, real production failure mode," and a direct counter-example to "always update the driver." They also checked for GPU/NVSwitch **Xid/SXid errors** and GPUDirect RDMA failures as part of the same health-check discipline, which is where §9's Xid table earns its place in this lesson.
- **Together AI, ["A practitioner's guide to testing and running large GPU clusters for training generative AI models"](https://www.together.ai/blog/a-practitioners-guide-to-testing-and-running-large-gpu-clusters-for-training-generative-ai-models)** (Lucchese, Birkner, Hagai, Adams; Aug 2024). A systematic testing/validation playbook that complements Imbue's narrative war story — where Imbue tells you what broke, Together AI tells you what to check *before* it breaks, covering the same stack-hygiene territory (driver, fabric, NCCL health) from the checklist side.
- **NVIDIA/NCCL and PyTorch GitHub issue trackers — [NVIDIA/nccl#584](https://github.com/NVIDIA/nccl/issues/584) ("NCCL hangs during ncclSend and ncclRecv") and [pytorch/pytorch#43546](https://github.com/pytorch/pytorch/issues/43546).** Real, filed, publicly visible bug reports that ground the lesson's "NCCL doesn't error cleanly, it hangs" claim in actual incidents other engineers hit and documented, rather than a generic assertion — in both filed issues the trigger was a mismatched/out-of-order collective call across ranks (not a driver/toolkit version skew), the same "silent hang, not a clean error" symptom a version mismatch also produces.
- **NCCL 2.21.5 release notes.** Two fixed defects that read like a summary of this lesson: a **mismatch crash during bootstrap** caused by TCP packet reordering, and a **hang on heterogeneous systems** caused by the `crossNic` value differing between nodes. What it shows: the canonical NCCL failure is a stall produced by a difference between ranks — sometimes a bug, often a configuration divergence — and NVIDIA's own changelog is the evidence.

## Worked example

### Decision 1 — H200 vs H100 for a 70B inference server, worked as a purchasing argument

**The workload.** Llama-3-70B class, chat serving, decode-heavy, FP8 weights. Take the hardware facts from §1 and lesson 5's footprint arithmetic and build the case in the order of §2's decision tree.

**Step 1 — capacity.** At FP8, weights are `70e9 × 1 B = 70 GB`.

```
  H100 (80 GB):  80 − 70 weights − ~4 runtime/workspace = ~6 GB for KV cache
  H200 (141 GB): 141 − 70 − ~4                          = ~67 GB for KV cache

  Llama-3-70B KV per token at FP8 (80 layers, 8 GQA KV heads, head_dim 128):
      2 × 80 × 8 × 128 × 1 B = 163,840 B = 0.156 MiB/token

  H100:  6 GB   ÷ 0.156 MiB = ~39,000 tokens ≈ 19 concurrent @ 2k context
  H200: 67 GB   ÷ 0.156 MiB = ~440,000 tokens ≈ 214 concurrent @ 2k context

  → 11× the achievable batch, from capacity alone.
```

At FP16 the picture changes qualitatively: 140 GB of weights does not fit an 80 GB H100 at all, so the H100 path forces **2× H100 with tensor parallelism** — two all-reduces per layer per token, 160 collectives per token on an 80-layer model — while the H200 holds the weights on one GPU with essentially nothing left for cache. This is the cliff from §2.

**Step 2 — bandwidth.** Decode re-reads the weight set (and the resident KV cache) per step, so its ceiling is `bytes ÷ bandwidth`:

```
  H100: 70 GB ÷ 3,350 GB/s = 20.9 ms/step → 47.8 steps/s (weights only)
  H200: 70 GB ÷ 4,800 GB/s = 14.6 ms/step → 68.5 steps/s

  ratio = 4800/3350 = 1.43×   ← directly on the bound path
```

**Step 3 — compute.** Identical. Same GH100 die, 1,979 FP8 dense TFLOPS on both. Prefill and any compute-bound section see **zero** improvement. If your traffic is prefill-dominated (long prompts, short generations) and already fits on one H100, H200 buys you very little and you should say so.

**Step 4 — combine into a throughput estimate.** The two effects multiply only up to the point where one of them saturates. Bandwidth gives 1.43×; the larger batch raises *aggregate* throughput because each weight read is amortised over more sequences, until either the latency SLO or the compute ceiling bites. A realistic composite for this workload lands near **1.7× aggregate tok/s**, and you should measure it rather than assume it — that measurement is exactly what lesson 7 asks you to produce.

**Step 5 — the cost test.** The decision rule is a ratio comparison, not a price comparison:

```
  H200 is cheaper per token   ⇔   ($/hr ratio)  <  (tok/s ratio)

  tok/s ratio ≈ 1.7 (measured)
  So H200 wins on $/token as long as it costs less than 1.7× the H100 rate
  at a MATCHED pricing tier.
```

**Write it up as:** *choose H200 for this workload. It collapses the replica to one GPU (removing tensor parallelism and 160 collectives per token), its 1.43× bandwidth lifts the memory-bound decode path directly, and its 1.76× capacity buys ~11× the KV headroom and therefore a much larger batch. H100 wins only if H200 pricing exceeds ~1.7× the H100 rate at a matched tier, or if the workload is prefill-dominated and already fits in 80 GB.* (Lesson 7 supplies the tier-aware pricing that turns that conditional into a number.)

### Decision 2 — a stack-mismatch diagnosis, worked as a triage runbook

**Symptom:** a job that ran yesterday now fails at startup with `torch.cuda.is_available() == False`, but `nvidia-smi` inside the same container prints the GPU normally.

Walk it in this order — each step eliminates one boundary from the §4 diagram:

```
  1. Is the DRIVER alive and consistent with its kernel module?  (boundary C)
     $ nvidia-smi
       → "Failed to initialize NVML: Driver/library version mismatch"
         ⇒ user-space driver was upgraded under a running module.
           Fix: reboot, or unload/reload the modules after draining.
       → prints normally ⇒ boundary C is fine; continue.

  2. What CUDA version does the DRIVER support?                  (boundary B)
     $ nvidia-smi | head -4
       the top-right "CUDA Version: 12.4" is the MAXIMUM the driver exposes,
       NOT the installed toolkit. This field confuses everyone once.
     $ cat /proc/driver/nvidia/version        # kernel module's own view

  3. What toolkit is the APP built against?
     $ python -c "import torch; print(torch.version.cuda)"   # e.g. 13.0
     $ nvcc --version                                        # if present

  4. Compare against the table in §5.
       toolkit 13.0 needs driver ≥ 580.65.06.
       Host driver is 535.183.06 → BELOW the minimum → error 35.
       Confirm:
     $ python -c "import torch; torch.zeros(1).cuda()"
       RuntimeError: CUDA error: CUDA driver version is insufficient
                     for CUDA runtime version

  5. Choose a remedy, in increasing order of blast radius:
       a. Use an image built against CUDA 12.x           (no host change)
       b. Install cuda-compat-13-0 in the image and put
          /usr/local/cuda-13.0/compat first on
          LD_LIBRARY_PATH                                (no host change;
                                                          datacenter GPUs only)
       c. Upgrade the host driver to R580 LTSB           (fleet-wide; canary)

  6. If instead the job RUNS but is slow across nodes, the boundary is (A):
     $ NCCL_DEBUG=INFO <job>  2>&1 | grep -E "NCCL version|via"
       diff that across ranks. A rank on a different NCCL version, or one
       reporting "via SHM" where peers report "via NET/IB", is the answer.
```

**The lesson this teaches about "pin everything, never `latest`":** that rule does not mean "never move." It means every version — including the driver — is a deliberate, tested, reversible choice, and that the *order* in which you try remedies should follow blast radius, from image-only to node-wide. Option (a) affects one job; option (c) affects every container on every node you touch.

## Practice — feeds the deliverable

On the rented box, capture the **full stack**:

```
nvidia-smi                         # driver version + MAX CUDA the driver supports (top-right)
nvidia-smi --query-gpu=name,driver_version,memory.total,compute_cap --format=csv
nvidia-smi -q | grep -i "cuda version"     # the driver's CUDA driver-API version
nvcc --version                     # installed CUDA toolkit version (if present)
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.backends.cudnn.version())"
python -c "import torch; print('.'.join(map(str, torch.cuda.nccl.version())))"
nvidia-ctk --version               # container toolkit (if containerized)
cat /proc/driver/nvidia/version    # kernel-module driver, as a cross-check
sudo dmesg -T | grep -i xid || echo "no Xid errors — clean baseline"
nvidia-smi -q -d PERFORMANCE | grep -A 12 "Clocks Event Reasons"   # 02b throttling check
nvidia-smi -q -d ECC | grep -A 6 "Aggregate"                       # ECC counters
```

Then:

1. **Record every version in a table** (GPU model, compute capability, driver, driver-API CUDA max, toolkit CUDA, cuDNN, NCCL, container-toolkit, framework). This table *is* an artifact — it is the thing you would diff across a fleet.
2. **Map it against §5's compatibility table.** State explicitly: "toolkit `<X.Y>` is in the CUDA `<major>` family, whose minimum driver is `<version>`; the host driver is `<version>`, which is `above/below` it, therefore minor-version compatibility `does/does not` apply." Note whether the driver branch you landed on is an LTSB and when it goes end of life.
3. **Write the blast-radius note** — for each component, what breaks if you bump *only* that one:
   - *Bump toolkit 12.x → 13.x*: requires driver ≥ R580.65.06. On an R535/R570 host the app fails with `cudaErrorInsufficientDriver` (35) at the first CUDA call, while `nvidia-smi` still works — unless you add `cuda-compat-13-x` (datacenter GPUs only) or upgrade the host driver.
   - *Bump NCCL on one node only*: risks a collective hang or a silent fallback to a slower transport across the whole job (NVIDIA/nccl#584); must be upgraded on all ranks together.
   - *Bump the host driver*: fleet-wide blast radius — every container on the node shares it. Canary one node; verify with a real job, not just `nvidia-smi`.
   - *`apt upgrade` touching the driver while jobs run*: `Failed to initialize NVML: Driver/library version mismatch` until reboot. Exclude NVIDIA packages from unattended upgrades.
4. **Imbue-style health check (no multi-node fabric required):** run the `dmesg | grep -i xid` and the ECC-counter query above on an idle rented GPU to establish what a clean baseline looks like, and record it. Xid codes are logged by the driver independently of your workload; knowing your baseline is empty is what makes a later non-empty result meaningful. If you *do* see anything, classify it against §9's table — 94 is informational, 63 is a trend, 48/79/95 mean stop.
5. **Stretch — verify a real throttling state.** Run the lesson-5 GEMM sweep in a loop for two minutes while sampling `nvidia-smi --query-gpu=clocks.sm,temperature.gpu,power.draw --format=csv -l 1`. If the SM clock falls below the rated boost, recompute your effective peak with §10's identity and note the difference between "% of spec" and "% of achievable" in your report.

**Acceptance:** the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md) contains a **stack-inventory table** (all versions captured from the rented GPU, plus the driver branch and its EOL date) plus a **compatibility note** stating, for at least the driver and NCCL, exactly what would break if that single component were bumped out of step and what the symptom would look like — and one line noting whether any Xid errors or throttle reasons were present at baseline.

## Common pitfalls

1. **Assuming "newer driver = safer."** Imbue's real fix was a driver *downgrade* — a driver regression relative to a specific InfiniBand/fabric firmware pairing is a real failure mode. "Pin everything, never latest" means pin to a *known-good, tested* version — in practice an LTSB — not necessarily the newest.
2. **Reading `nvidia-smi`'s top-right "CUDA Version" as the installed toolkit.** It is the *maximum* CUDA version the driver can support, not what is installed. A box showing "CUDA Version: 12.4" may have no toolkit at all, or a 12.1 toolkit. Use `nvcc --version` or `torch.version.cuda` for the toolkit.
3. **Treating a working `nvidia-smi` as proof that CUDA works.** `nvidia-smi` talks to the driver directly and will happily print a healthy GPU while every CUDA application fails with error 35. The two facts are independent.
4. **Bumping NCCL on a subset of nodes during a rolling upgrade.** NCCL must move in lockstep across every rank in a job — a skew doesn't reliably error, it hangs on the first collective or silently falls back to a slower transport (see the real filed GitHub issues above, and NCCL 2.21.5's own fixed `crossNic`-mismatch hang).
5. **Letting `apt upgrade` / unattended-upgrades touch the NVIDIA packages on a GPU node.** Replacing the user-space driver while `nvidia.ko` is loaded yields `Failed to initialize NVML: Driver/library version mismatch` and nothing works until reboot. Hold the packages.
6. **Reaching for `cuda-compat` on a workstation GPU.** Forward compatibility is a datacenter-GPU feature; on GeForce silicon it produces error 804, `forward compatibility was attempted on non supported HW`. Recognise that message as "the compat libs loaded and the hardware refused," and remove them from `LD_LIBRARY_PATH` rather than installing more.
7. **Treating B200 spec numbers as settled and single.** Credible documentation disagrees on exact capacity and bandwidth depending on SKU variant (bare B200 vs HGX B200 vs B300); always state a range and re-verify at purchase/build time rather than quoting one number with false precision. The same discipline applies to FP64 on Blackwell, which went *down* relative to Hopper — "newer is better" is not uniformly true.
8. **Treating MIG isolation as equivalent to MPS or time-slicing.** It's hardware-partitioned — 7 compute slices, 8 memory slices, dedicated SMs and L2 and memory bandwidth — a materially stronger guarantee. But name its limits too: no NVLink between instances, no aggregation for multi-GPU jobs, and reconfiguration requires draining the GPU.
9. **Skipping the cheap health checks because "the GPU shows up in `nvidia-smi`."** Imbue's diagnostic path (IB error rates, Xid/SXid logs, driver-firmware audit) catches exactly the class of failure that looks fine at a glance and only surfaces as an unexplained stall under real multi-node load. An Xid 95 in the log means your last checkpoint may be corrupt — and you will only know if you looked.

## Self-check

- **Argue H200 over H100 for a 70B inference server, on capacity and bandwidth, with numbers.** *Answer:* At FP16, H100's 80 GB cannot hold 140 GB of weights, forcing 2× H100 with tensor parallelism — two all-reduces per layer per token, 160 collectives per token on an 80-layer model. H200's 141 GB removes that entirely. At FP8 (70 GB of weights) both fit on one GPU, but the KV-cache headroom differs enormously: ~6 GB on H100 vs ~67 GB on H200, which at 0.156 MiB/token for Llama-3-70B is ~39k vs ~440k cached tokens — roughly 11× the concurrent batch. And decode is bandwidth-bound: 4,800 vs 3,350 GB/s is 1.43× directly on the bound path. Compute is *identical* (same GH100 die, 1,979 FP8 dense TFLOPS), so prefill-dominated workloads that already fit gain nothing. H200 wins on $/token whenever its price premium at a matched tier is below the measured throughput ratio (~1.7×).
- **A CUDA 12.x application on a host with driver R525 — will it run? What about a CUDA 13.x app on that same driver?** *Answer:* The 12.x app runs. 525.60.13 is the minimum driver for the CUDA 12 family, and minor-version compatibility means any 12.x-built app runs on any driver in the R525–R579 band with no extra package, provided it doesn't call APIs newer than that driver exposes (those return `cudaErrorCallRequiresNewerDriver`). The 13.x app will **not** run: CUDA 13.x requires ≥ 580.65.06, so you get `cudaErrorInsufficientDriver` (error 35, "CUDA driver version is insufficient for CUDA runtime version") at the first CUDA call — while `nvidia-smi` keeps working, which is the confusing part. Remedies in blast-radius order: rebuild against 12.x; install `cuda-compat-13-x` in the image and prepend `/usr/local/cuda-13.0/compat` to `LD_LIBRARY_PATH` (datacenter GPUs only); or upgrade the host driver to R580.
- **What exactly does the `cuda-compat` package do, and when does it fail?** *Answer:* It installs a *newer user-space* `libcuda.so` into `/usr/local/cuda-<X.Y>/compat/`, which you put ahead of the system path so a newer-toolkit application can run against an older host kernel driver. The kernel module is unchanged. It fails in two ways: on non-datacenter GPUs it returns error 804, `cudaErrorCompatNotSupportedOnDevice` ("forward compatibility was attempted on non supported HW"); and it cannot make an old kernel driver drive hardware it never supported. It is the escape hatch for fleets that cannot take a driver maintenance window, not a general-purpose version solvent.
- **Imbue's engineering team fixed a multi-node training stall by *downgrading* their NVIDIA driver. What does this imply about the "always pin the latest driver" instinct?** *Answer:* "Pin everything, never latest" means pin to a *known-good, tested* version — in practice an LTSB branch, which gives ~3 years of security fixes without functional churn — not necessarily the newest. A driver regression relative to a specific InfiniBand/fabric firmware pairing is a real failure mode. Movement in *either* direction is a change that needs a canary and a rollback plan, and the driver deserves a stricter process than anything above it because it is the only component every container on a node shares.
- **What's the classic NCCL mismatch symptom, and how do you diagnose it in five minutes?** *Answer:* No clean error. Either the job **hangs on the first collective** — every rank blocked in a CUDA kernel waiting for peers, no exit code, until PyTorch's ProcessGroupNCCL watchdog eventually fires minutes later with a "collective operation timeout" — or it **silently falls back to a slower transport** (sockets/SHM instead of NVLink/IB), producing a throughput cliff with no crash. Diagnosis: run with `NCCL_DEBUG=INFO`, then diff the per-rank output for the NCCL version line and the selected transport. A rank reporting `via SHM` where its peers report `via NET/IB` is your answer. Real evidence: NVIDIA/nccl#584 and pytorch/pytorch#43546 (both triggered by mismatched/out-of-order collectives across ranks, the same symptom class a version skew produces), and NCCL 2.21.5's own release notes, which fixed a hang caused by `crossNic` differing between nodes.
- **What precisely is MIG partitioning, and what does it *not* give you?** *Answer:* Hardware partitioning of one GPU into up to seven instances, assembled from **7 compute slices and 8 memory slices** (the counts differ because the memory system splits into 8 equal pieces while the SM array yields 7 usable groups — which is what allows `3g.40gb` and `4g.40gb` to coexist as distinct profiles). Each instance gets dedicated SMs, a dedicated L2 slice, and dedicated memory-controller bandwidth, so a noisy tenant cannot starve a neighbour — materially stronger than MPS or time-slicing, where all clients share one scheduler. Underneath, a MIG device is a GPU Instance (slices + engines) containing one or more Compute Instances (SM subdivisions). What it does *not* give you: NVLink between instances, aggregation across instances for a multi-GPU job, arbitrary sizing outside the profile menu, or reconfiguration without draining the GPU.
- **Your GEMM benchmark reports 600 TFLOPS BF16 on an H100 SXM5 — 61% of the 989.5 dense spec. Before you tune the kernel, what do you check?** *Answer:* The clock. Peak is `SMs × FLOP/SM/clock × clock`, and throttling reduces the last term. Run `nvidia-smi -q -d PERFORMANCE` and look at `Clocks Event Reasons`; if the card is in `SW_Thermal_Slowdown` or `HW_Power_Brake` at, say, 1,400 MHz instead of the rated 1,830 MHz, its real ceiling is `132 × 4096 × 1.40e9 = 757 TFLOPS` and you are at 79% of *achievable*, not 61% of achievable. Report both numbers and say which denominator you used — a throttled rented card delivers fewer real FLOPS per dollar-hour, and that is a procurement fact, not a code problem.
- **You see `NVRM: Xid (PCI:0000:1b:00): 95` in dmesg partway through a 12-hour training run. What do you do?** *Answer:* Xid 95 is an **uncontained** ECC error — the fault escaped containment, so workload outputs may be corrupt. Kill the job, discard the checkpoint written after that timestamp, and rerun from the last known-good one. Then look at the ECC and row-remap counters (`nvidia-smi -q -d ECC`, "Remapped Rows"): a rising Xid 63/64 count on the same card means it is trending toward failure and should be scheduled out. Contrast with Xid 94, which is the *contained* case — corrected, no other process affected, informational only — and with Xid 48 (double-bit ECC) or 79 (fallen off the bus), which are terminal and mean drain the node.

## Connections & what's next

This lesson closes the hardware-literacy arc of the module: lesson 1 taught you to distrust raw GPU-Util, lesson 2 gave you the roofline to place a workload honestly, lessons 3 and 4 built the memory/bandwidth model that decode throughput and batching depend on, lesson 5 added precision as a lever on both axes of the roofline *and* the `SMs × FLOP/SM/clock × clock` identity that this lesson reused twice — once to decompose the A100→H100 jump, once to show what throttling actually costs you. This lesson adds the two remaining variables: which generation, and whether the software stack underneath it is actually correct. Without both, none of the previous five lessons' numbers are trustworthy on real rented hardware.

Next: **[03.7 · Capstone — cost per unit of useful work](07-capstone-cost-per-useful-work.md)** is where all six lessons converge into a single deliverable: a defended SKU recommendation that states achieved-vs-spec TFLOPS, the FP16→FP8 delta, the utilization-lie demo, *and* — carrying this lesson's thread forward directly — the generation choice (with the §2 decision tree behind it) and a stack-hygiene note, all priced against live 2026 GPU-rental data and explicitly dated as a snapshot.

## References & further reading

**Primary sources**
- [NVIDIA CUDA Compatibility documentation](https://docs.nvidia.com/deploy/cuda-compatibility/) — including the `minor-version-compatibility.html` and `forward-compatibility.html` sub-pages. The authoritative source for this lesson's minimum-driver table (CUDA 11.x ≥ 450.80.02, 12.x ≥ 525.60.13, 13.x ≥ 580.65.06), the minor-compat band bounds, and what `cuda-compat-<major>-<minor>` installs where.
- [NVIDIA Data Center GPU Driver — driver lifecycle and supported CUDA versions](https://docs.nvidia.com/datacenter/tesla/drivers/) — the branch taxonomy (New Feature / Production / Long Term Support), support durations (~1 year PB, ~3 years LTSB), and the current EOL dates used in §5's branch table. **Time-sensitive — re-check before pinning.**
- [NVIDIA H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — canonical dense/sparse TFLOPS, the 132-SM GH100 configuration, and the HBM3 subsystem behind §1's H100 row.
- [NVIDIA H200 datasheet / product page](https://www.nvidia.com/en-us/data-center/h200/) — canonical 141 GB / 4.8 TB/s source and the HBM3e stack configuration that explains why H200 is the same compute die as H100.
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/) — the full profile menu per SKU, the GPU-Instance / Compute-Instance hierarchy, and the `nvidia-smi mig -lgip / -cgi / -cci` syntax reproduced in §3.
- [NVIDIA Container Toolkit documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/) — the runtime hook vs. CDI injection modes, `nvidia-ctk cdi generate`, and exactly which host libraries are mounted into a container.
- [NVIDIA Xid error documentation](https://docs.nvidia.com/deploy/xid-errors/) — the full Xid code list behind §9's table, including the 94-vs-95 contained/uncontained ECC distinction.
- [NCCL release notes](https://docs.nvidia.com/deeplearning/nccl/release-notes/) — version-by-version fixed defects; 2.21.5 in particular records the bootstrap-mismatch crash and the `crossNic`-divergence hang cited in §6.
- [NVIDIA data-center GPU product hub](https://www.nvidia.com/en-us/data-center/) — cross-check every spec-table number before quoting it in a report, especially the B200 row flagged in §1 as SKU-ambiguous.

**Real-world engineering blogs**
- Imbue, ["From bare metal to a 70B model: infrastructure set-up and scripts"](https://imbue.com/research/70b-infrastructure/) — the real driver-downgrade incident this lesson's §8 traces end to end, plus the health-check regime (IB error rates, Xid/SXid, GPUDirect RDMA, firmware audit) that §9 and the Practice section reproduce.
- Together AI, ["A practitioner's guide to testing and running large GPU clusters for training generative AI models"](https://www.together.ai/blog/a-practitioners-guide-to-testing-and-running-large-gpu-clusters-for-training-generative-ai-models) — a systematic pre-failure testing checklist that complements Imbue's incident narrative.
- [NVIDIA/nccl#584](https://github.com/NVIDIA/nccl/issues/584) and [pytorch/pytorch#43546](https://github.com/pytorch/pytorch/issues/43546) — real, filed GitHub issues documenting NCCL's hang-not-error failure mode (mismatched/out-of-order collective calls across ranks in both cases), the same symptom class a driver/NCCL version skew also triggers.

**Deeper dives**
- [NVIDIA GPU Operator documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/) — how the whole chain in §4 gets managed declaratively on Kubernetes (driver container, toolkit, device plugin, DCGM exporter, MIG manager), which is where module 04 picks this thread up.
- [PyTorch distributed troubleshooting / `TORCH_NCCL_*` environment variables](https://docs.pytorch.org/docs/stable/distributed.html) — the watchdog and async-error-handling knobs referenced in §6, and what each one changes about how a hang surfaces.
