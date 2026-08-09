---
lesson: "03.6"
title: "Generational Differences & the Software Stack"
module: "03"
concept: "Generational Differences & the Software Stack"
status: not-started
est_time: "4h"
artifacts: []
---
# 03.6 · Generational Differences & the Software Stack
> **Concept.** Pick the right generation from the memory/bandwidth/FP8/MIG table, then keep the driver↔CUDA↔cuDNN/NCCL↔container-toolkit stack pinned — because version drift is the silent failure mode, and a throttled GPU silently spends your budget.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Why this matters

Two things quietly waste GPU budget in production, and neither shows up as an error.

The first is **buying or scheduling the wrong generation**. A 70B inference server pinned to H100s because "that's what we have" may be paying for tensor-parallel all-reduce overhead that an H200's larger memory would eliminate — or conversely, paying B200 prices for a workload that an A100 fleet serves fine. You make these arguments from a table of memory, bandwidth, FP8 support, and MIG partitioning, tied to the actual bottleneck.

The second is **version drift in the software stack**. The driver, CUDA toolkit, cuDNN, NCCL, and the container toolkit are a versioned chain. Get it wrong and you don't get a clean failure — you get a container that won't start, or worse, an NCCL mismatch that *runs* but hangs on collective ops or silently drops to a slow fallback path. On a multi-node training job at hundreds of dollars an hour, a silent perf regression is money leaving the building. Treating this chain as an operational surface — pinned, inventoried, tested before bumps — is core platform-engineering hygiene.

## The platform-engineer's lens

You own two artifacts here: a **placement/purchasing argument** and a **stack inventory with a blast-radius note**.

- **Placement/purchasing:** given a workload's bottleneck (memory capacity? bandwidth? compute? isolation?), name the generation that removes it and the cost delta. "H200 over H100 for this 70B server: fits on one GPU, so we drop tensor parallelism and its NVLink overhead."
- **Stack inventory:** capture every version in the chain, map it to the CUDA compatibility matrix, and state exactly what breaks if you bump one component. This is the artifact that prevents a 3 a.m. "the training job hangs after the driver upgrade" incident.

The governing rule for the stack is **pin everything, never `latest`.** Reproducibility and blast-radius control beat being current.

## Core notes

### The cross-generation table

Dense tensor-core throughput (sparsity roughly doubles the marketed figures; the numbers below are **dense**). Verify against the current datasheet at build time — these are version-sensitive.

| GPU (SXM) | Arch | Memory | Bandwidth | BF16/FP16 dense | FP8 dense | FP4 dense | MIG |
|---|---|---|---|---|---|---|---|
| A100 80GB | Ampere | 80 GB HBM2e | ~2.0 TB/s | 312 TFLOPS | — (no FP8) | — | up to 7 |
| H100 | Hopper | 80 GB HBM3 | ~3.35 TB/s | ~990 TFLOPS | ~1,979 TFLOPS | — | up to 7 |
| H200 | Hopper | 141 GB HBM3e | ~4.8 TB/s | ~990 TFLOPS | ~1,979 TFLOPS | — | up to 7 |
| B200 | Blackwell | 192 GB HBM3e | ~8.0 TB/s | ~2,250 TFLOPS | ~4,500 TFLOPS | ~9,000 TFLOPS | up to 7 |

Read this table by *what changed*:

- **A100 → H100:** same 80 GB, but +FP8 (new precision tier, no A100 equivalent), ~1.7× bandwidth, ~3× BF16 compute. The generational jump is *precision + compute*, not capacity.
- **H100 → H200:** **same GH100 silicon, same compute** — the only changes are **memory (80→141 GB) and bandwidth (3.35→4.8 TB/s)**. H200 is a pure *capacity/bandwidth* upgrade. It matters only for memory-bound or capacity-constrained workloads (large models, long context, big KV cache). Same FP8 TFLOPS as H100.
- **H200 → B200:** Blackwell — bigger memory (192 GB), ~1.7× bandwidth, ~2.3× FP8 compute, and native **FP4** (~2× again for inference that tolerates it). This is the full jump: capacity *and* compute *and* a new precision tier.

**MIG** (Multi-Instance GPU) partitions one physical GPU into up to 7 hardware-isolated instances with dedicated memory slices and compute — the lever for packing many small inference jobs onto one card with strong isolation, instead of one job under-utilizing it. Available across all four generations here.

### The versioned software chain

From the metal up:

1. **NVIDIA driver** (kernel module + user-space, e.g. R535/R550/R570). Talks to the hardware. Ships the CUDA *driver API* (`libcuda.so`).
2. **CUDA toolkit** (e.g. 12.4). Provides `nvcc`, the *runtime API* (`libcudart`, statically or dynamically linked into your app), and math libs.
3. **cuDNN / NCCL / cuBLAS** — the DL primitive and collective-communication libraries. **NCCL** drives multi-GPU/multi-node all-reduce, all-gather, etc. — the backbone of distributed training.
4. **NVIDIA Container Toolkit** (`nvidia-ctk`, the runtime hook) — injects the host driver into containers so a CUDA image can see the GPU. The container carries its own CUDA/cuDNN/NCCL; the *driver* comes from the host.

The key insight: **the driver lives on the host; the toolkit and libraries live in the container/venv.** Compatibility is a relationship between them, governed by the CUDA compatibility rules.

### The CUDA compatibility rules you must know

- **Minor-version compatibility (within a major family).** From CUDA 11 onward, an application built with any minor toolkit in a major family runs on any driver in that family that meets the **minimum driver version**. For **CUDA 12.x the minimum driver is R525** (Linux ≥ 525.60.13). So an app built with toolkit 12.4 runs on a host driver installed for 12.0 (≥525) **without** any extra package — as long as it doesn't call APIs newer than that driver provides.
- **Forward compatibility (across major families / frozen datacenter drivers).** When the host is stuck on an *older* driver from a *different* major family (common on locked-down datacenter fleets), you install the **`cuda-compat`** package — a special user-space upgrade of `libcuda` — to run a newer-toolkit app on that old driver. This is the escape hatch when you can't touch the host driver.
- **Backward compatibility.** A newer driver always runs older CUDA apps. Upgrading the driver is the safe direction; downgrading it is what breaks apps.
- **NCCL / cuDNN** ride along with the toolkit but have their **own** version constraints and, critically, **must match across every node** in a distributed job. NCCL is peer-to-peer at runtime: a version skew between ranks doesn't reliably error — it can **hang** on the first collective or fall back to a slower transport, showing up as a mysterious throughput cliff.

**Operational rule: pin every layer, never `latest`.** Pin the container image by digest, pin toolkit/cuDNN/NCCL versions, and treat a driver bump as a change that needs a canary — because it changes the one component every container shares.

### 15-minute callback to 02b: throttling silently lowers your roofline

You already learned (02b) how to read power/thermal throttling with `nvidia-smi -q -d PERFORMANCE` and `clocks_throttle_reasons`. Here's why it belongs in a *cost/roofline* module: **a throttled GPU silently lowers its achieved clocks, and therefore its achieved TFLOPS and bandwidth.** The spec-sheet roofline in 03.4/03.5 assumes rated clocks; a card in `SW_Thermal_Slowdown` or `HW_Power_Brake` is running a *lower* roofline than you're paying for. So when your DCGM tensor-active is high but throughput is below model, the cause may not be your code — it may be a thermally throttled card delivering fewer real FLOPS per rented hour. Always check `clocks_throttle_reasons` before concluding a workload is compute-bound. (Mechanics: see 02b — not re-taught here.)

## Worked example

**Decision: H200 vs H100 for a 70B inference server.**

- **Capacity.** 70B at FP16 = 140 GB. H100 (80 GB) can't hold it → you need **2× H100 with tensor parallelism**. H200 (141 GB) holds the weights on **one GPU**, leaving little room for KV cache — so H200 is the clean single-GPU fit at FP8 (70 GB weights + generous cache) and a tight fit at FP16.
- **Bandwidth.** Decode is memory-bound (one token at a time, re-reading weights + KV cache). H200's **4.8 TB/s vs H100's 3.35 TB/s (~1.43×)** directly raises decode throughput on that bandwidth-bound path — same compute silicon, faster memory.
- **Compute.** Identical (same GH100 die, ~1,979 FP8 TFLOPS both). Prefill/compute-bound sections don't improve — so if your workload is prefill-heavy and already fits, H200 buys less.
- **Cost argument.** 2× H100 ≈ $6/replica-hr with cross-GPU all-reduce overhead per token; 1× H200 removes tensor parallelism *and* adds ~1.43× decode bandwidth. Unless the H200 hourly rate exceeds ~2× the H100 rate, the H200 wins on $/token for this memory-and-bandwidth-bound workload. **Write it up as: choose H200 — it collapses the replica to one GPU (no TP overhead) and its bandwidth lifts the memory-bound decode path; H100 only wins if H200 pricing is >2× or the workload is compute-bound and already fits.**

## Practice — feeds the deliverable

On the rented box, capture the **full stack**:

```
nvidia-smi                         # driver version + max CUDA supported by driver (top-right)
nvidia-smi -q | grep -i cuda       # driver's CUDA driver-API version
nvcc --version                     # installed CUDA toolkit version
python -c "import torch; print(torch.version.cuda, torch.backends.cudnn.version())"   # toolkit + cuDNN the app links
python -c "import torch; print(torch.cuda.nccl.version())"   # NCCL version
nvidia-ctk --version               # container toolkit (if containerized)
cat /proc/driver/nvidia/version    # kernel-module driver, as a cross-check
```

Then:

1. Record every version in a small table (driver, driver-API CUDA, toolkit CUDA, cuDNN, NCCL, container-toolkit, GPU model).
2. Map the toolkit version against the **CUDA compatibility matrix**: confirm the host driver meets the minimum for that CUDA major family (e.g. toolkit 12.x needs driver ≥ R525).
3. Write the **blast-radius note**: for each component, state what breaks if you bump *only* that one. E.g. "Bump toolkit 12.4→13.x: requires driver ≥ the CUDA 13 minimum — current R5xx driver may be below it → app fails to init unless we also upgrade the driver or add `cuda-compat`." "Bump NCCL on one node only: risks a collective hang / slow-transport fallback across the job — must upgrade all ranks together."

**Acceptance:** the deliverable contains a **stack-inventory table** (all versions captured from the rented GPU) plus a **compatibility note** stating, for at least the driver and NCCL, exactly what would break if that single component were bumped out of step.

## Self-check

**(a)** Argue H200 over H100 for a 70B inference server, on capacity and bandwidth.
**Answer:** H100's 80 GB can't hold 140 GB of FP16 weights, forcing 2× H100 + tensor parallelism (all-reduce overhead per token). H200's 141 GB holds it on one GPU (comfortably at FP8), removing TP entirely. And decode is memory-bandwidth-bound — H200's 4.8 TB/s vs H100's 3.35 TB/s (~1.43×) raises decode throughput on identical compute silicon. So H200 collapses the replica to one GPU and speeds the bound path; H100 only wins if H200 pricing is >2× or the job is compute-bound and already fits.

**(b)** A CUDA 12.x application on a host with driver 525 — will it run?
**Answer:** Yes. R525 is the minimum driver for the CUDA 12 major family, so minor-version compatibility lets any 12.x-built app run on it **without** a compat package — provided the app doesn't call APIs newer than that driver exposes (which would fail at load). If the host were on an *older* driver from a different major family, you'd need the `cuda-compat` forward-compatibility package instead.

**(c)** What's the classic NCCL version-mismatch symptom?
**Answer:** No clean error — the job **hangs on the first collective** (all-reduce/all-gather) because ranks can't negotiate a common protocol, or it silently **falls back to a slower transport** and you see a throughput cliff with no crash. That's why NCCL (and cuDNN) must be pinned identically across every node.

## Resources

1. **NVIDIA CUDA Compatibility docs** — the authoritative minor-version and forward-compatibility pages, including minimum-driver tables. https://docs.nvidia.com/deploy/cuda-compatibility/
2. **NVIDIA data-center GPU datasheets** (H100, H200, B200) — cross-check every spec-table number to the primary source before quoting it. https://www.nvidia.com/en-us/data-center/
3. **Cross-generation SKU comparison** (cross-check to NVIDIA datasheets, not a primary source) — convenient side-by-side of memory/bandwidth/FP8/MIG across generations. https://intuitionlabs.ai/articles/nvidia-data-center-gpu-specs
