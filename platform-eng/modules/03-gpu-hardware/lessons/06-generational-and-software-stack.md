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
sources: 8
---

# 03.6 · Generational Differences & the Software Stack

> **Concept.** Pick the right generation from the memory/bandwidth/FP8/MIG table, then keep the driver↔CUDA↔cuDNN/NCCL↔container-toolkit stack pinned — because version drift is the silent failure mode, and a throttled GPU silently spends your budget.
>
> Module: [🔌 03 — GPU hardware fundamentals](../README.md) · Deliverable: [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md)

## Where this fits

Lesson 5 established that precision support is gated by hardware generation — FP8 needs Hopper or newer, FP4 needs Blackwell or newer. That fact was a footnote there; here it becomes the spine. This lesson turns "which generation" into two deliverables you'll actually write in a job: a **placement/purchasing argument** (which SKU removes this workload's specific bottleneck, and at what cost delta) and a **stack inventory** (the driver/CUDA/NCCL chain that has to be right for any of the previous five lessons' numbers to be real on the hardware you land). It closes the module's hardware-literacy arc — lesson 7's capstone assumes you can both pick a generation and keep it running without version drift silently eating your budget.

## Why this matters

Two things quietly waste GPU budget in production, and neither shows up as an error message.

The first is **buying or scheduling the wrong generation**. A 70B inference server pinned to H100s "because that's what we have" may be paying for tensor-parallel all-reduce overhead that an H200's larger memory would eliminate outright — or, in the other direction, paying Blackwell prices for a workload an Ampere fleet already serves fine. You make these arguments from a table of memory, bandwidth, FP8/FP4 support, and MIG partitioning, tied to the workload's actual bottleneck — not from "the newest one is always better."

The second is **version drift in the software stack**. The driver, CUDA toolkit, cuDNN, NCCL, and the container toolkit are a versioned chain, and getting it wrong rarely produces a clean failure. It produces a container that won't start, or worse, an NCCL mismatch that *runs* but hangs on collective ops or silently drops to a slow fallback path. On a multi-node training job burning hundreds of dollars an hour, a silent performance regression is money leaving the building in real time, with no alert firing. This is not a hypothetical: Imbue's public infrastructure write-up (below) describes exactly this class of failure on a real 70B training run, and the fix was counterintuitive — a driver **downgrade**, not an upgrade. Treating this chain as an operational surface — pinned, inventoried, canaried before any bump — is core platform-engineering hygiene, and it's precisely the "hardware validation across the fleet" and "controllers to automate infra testing" language in CoreWeave's GPU Performance Engineer posting that this module's README opens with.

## What's new here (calibration)

The module README skips deep microarchitecture and NVLink/PCIe topology, node layout, and power/thermal throttling mechanics — those are 02b's territory, referenced here, not re-taught. This lesson does not re-explain how throttling works at the hardware level; it only reminds you (below) why a throttled card matters to the cost numbers this module builds. What it adds instead:

- **A precise, generation-spanning CUDA/driver compatibility table** — not just "CUDA 12 needs driver 525," but the full 11.x/12.x/13.x progression, so you can reason about a fleet that spans generations of both hardware and drivers simultaneously.
- **MIG's actual partitioning mechanics** — "up to 7 instances" is the number everyone quotes; the precise mechanism (7 compute slices, 8 memory slices, independently combinable) is the depth a Staff-level answer needs.
- **A named, real production incident for the abstract "version drift" warning** — Imbue's driver-downgrade fix, which directly contradicts the naive "always update" instinct and gives you a war story to reason from instead of a rule to recite.
- **Explicit handling of a live spec disagreement** — the B200's memory/bandwidth numbers are genuinely contested across vendor-mirror sources at the time of writing; this lesson teaches you to flag that rather than fabricate false precision, which is itself a skill a fast-moving hardware market demands.

## Core concepts

### The cross-generation table

Dense tensor-core throughput (2:4 structured sparsity roughly doubles the marketed figures; the numbers below are **dense**). Every number here is a **2026 dated snapshot** — verify against the current datasheet at build time, especially for Blackwell, where sources disagree (see the flag below).

| GPU (SXM) | Arch | Memory | Bandwidth | BF16/FP16 dense | FP8 dense | FP4 dense | MIG |
|---|---|---|---|---|---|---|---|
| A100 80GB | Ampere | 80 GB HBM2e | ~2.0 TB/s | 312 TFLOPS | — (no FP8 hardware) | — | up to 7 |
| H100 | Hopper | 80 GB HBM3 | ~3.35 TB/s | ~990 TFLOPS | ~1,979 TFLOPS | — | up to 7 |
| H200 | Hopper | 141 GB HBM3e | ~4.8 TB/s | ~990 TFLOPS | ~1,979 TFLOPS | — | up to 7 |
| B200 | Blackwell | **180–192 GB HBM3e (verify SKU)** | **~7.7–8.0 TB/s (verify SKU)** | ~2,250 TFLOPS | ~4,500 TFLOPS | ~9,000 TFLOPS | up to 7 |

**On the B200 row specifically: do not treat those numbers as settled.** Credible vendor-mirror sources disagree on whether a "B200" is 180 GB or 192 GB of HBM3e, and whether bandwidth is ~7.7 or ~8.0 TB/s. The likely explanation is that different sources describe different configurations under the same marketing name — the bare B200 GPU module vs. the board-level HGX B200 vs. later B300 variants — but that couldn't be resolved to one authoritative figure through public documentation alone at the time this lesson was written. The correct professional move here is not to pick the number that looks more precise; it's to **state the range and explicitly flag "confirm exact SKU variant at build/purchase time"** in any report or purchasing argument you write. A false-precision single number that turns out wrong in a procurement conversation costs you credibility a stated range never will.

Read the rest of the table by *what changed generation to generation*:

- **A100 → H100:** same 80 GB capacity, but **+FP8 as a wholly new precision tier** (A100 has no equivalent), ~1.7x bandwidth, ~3x BF16 compute. The generational jump here is *precision + compute*, not capacity.
- **H100 → H200:** **same GH100 silicon, same compute** — the only changes are **memory (80→141 GB) and bandwidth (3.35→4.8 TB/s)**. H200 is a pure *capacity/bandwidth* upgrade, delivered by re-packaging: H100 uses 5 stacks of 16 GB HBM3 (80 GB), H200 uses 6 stacks of 24 GB HBM3e (144 GB installed, **141 GB usable** — the ~3 GB delta is reserved) — a literal packaging decision, not a compute redesign. It matters only for memory-bound or capacity-constrained workloads (large models, long context, big KV cache). Same FP8 TFLOPS as H100, because it's the same tensor cores.
- **H200 → B200:** Blackwell — bigger memory, roughly 1.6-1.7x bandwidth (pending the exact-SKU caveat above), ~2.3x FP8 compute, and a genuinely new precision tier, native **FP4** (~2x again for inference that tolerates it, per lesson 5's precision table). This is the full jump: capacity *and* compute *and* a new precision tier, which is why it's priced and positioned differently from the H100→H200 step.

### MIG — the precise partitioning mechanics

**MIG** (Multi-Instance GPU) partitions one physical GPU into up to 7 hardware-isolated instances. The number worth knowing precisely, not just "up to 7": each MIG-capable GPU exposes **7 compute slices and 8 memory slices**, and a MIG instance is built by combining some number of each (the extra memory slice exists because not every compute-slice combination maps to an equal memory split — the two dimensions are partitioned independently, then instances are assembled from a menu of valid combinations). This is **hardware-level partitioning**, not a software time-slicing or cgroups-style scheme — each instance gets dedicated streaming multiprocessors, dedicated L2 cache slices, and dedicated memory bandwidth, isolated at the silicon level. The practical consequence worth stating precisely in an interview: **a misbehaving or noisy tenant on one MIG instance cannot starve another instance's throughput**, because there is no shared scheduling queue between instances the way there is with GPU time-slicing or MPS (Multi-Process Service). This is the lever for packing many small inference jobs onto one card with a real isolation guarantee, instead of one job under-utilizing an entire GPU or multiple jobs contending unpredictably under software sharing. MIG is available across all four generations in the table above.

### The versioned software chain

From the metal up:

1. **NVIDIA driver** (kernel module + user-space, e.g. R535/R550/R570/R580). Talks to the hardware. Ships the CUDA *driver API* (`libcuda.so`).
2. **CUDA toolkit** (e.g. 12.4, 13.0). Provides `nvcc`, the *runtime API* (`libcudart`, statically or dynamically linked into your app), and math libs.
3. **cuDNN / NCCL / cuBLAS** — the DL primitive and collective-communication libraries. **NCCL** drives multi-GPU/multi-node all-reduce, all-gather, etc. — the backbone of distributed training.
4. **NVIDIA Container Toolkit** (`nvidia-ctk`, the runtime hook) — injects the host driver into containers so a CUDA image can see the GPU. The container carries its own CUDA/cuDNN/NCCL; the *driver* comes from the host.

The key insight: **the driver lives on the host; the toolkit and libraries live in the container/venv.** Compatibility is a relationship between them, governed by the CUDA compatibility rules below.

### The CUDA/driver compatibility table you must know

| CUDA toolkit family | Minimum driver | Notes |
|---|---|---|
| CUDA 11.x | ≥ R450 | superseded once driver ≥R525 is installed; backward-compat still works |
| CUDA 12.x | ≥ R525 | valid for "minor version compatibility" purposes up to before R580 |
| CUDA 13.x | ≥ R580 | current major family as of mid-2026 |

- **Minor-version compatibility (within a major family).** From CUDA 11 onward, an application built with any minor toolkit in a major family runs on any driver in that family that meets the **minimum driver version**. For **CUDA 12.x the minimum driver is R525**. So an app built with toolkit 12.4 runs on a host driver installed for 12.0 (≥525) **without** any extra package — as long as it doesn't call APIs newer than that driver provides.
- **Forward compatibility (across major families / frozen datacenter drivers).** When the host is stuck on an *older* driver from a *different* major family (common on locked-down datacenter fleets that can't touch the driver without a maintenance window), you install the **`cuda-compat`** package — a special user-space upgrade of `libcuda` — to run a newer-toolkit app on that old driver. This is the escape hatch when you can't touch the host driver.
- **Backward compatibility.** A newer driver always runs older CUDA apps. Upgrading the driver is the safe direction; downgrading it is what typically breaks apps — which is exactly what makes Imbue's real incident below worth internalizing: the safe-sounding direction isn't always the one that fixes your actual problem.
- **NCCL / cuDNN** ride along with the toolkit but have their **own** version constraints and, critically, **must match across every node** in a distributed job. NCCL is peer-to-peer at runtime: a version skew between ranks doesn't reliably error — it can **hang** on the first collective or fall back to a slower transport, showing up as a mysterious throughput cliff with no crash to point at. This isn't a theoretical warning: real, filed issues document exactly this behavior — `NCCL hangs during ncclSend and ncclRecv` ([NVIDIA/nccl#584](https://github.com/NVIDIA/nccl/issues/584)) and a PyTorch-side report of collective-mismatch hang risk ([pytorch/pytorch#43546](https://github.com/pytorch/pytorch/issues/43546)).

**Operational rule: pin every layer, never `latest`.** Pin the container image by digest, pin toolkit/cuDNN/NCCL versions, and treat a driver bump as a change that needs a canary — because the driver is the one component every container on a node shares, so a bad bump has fleet-wide blast radius, not single-job blast radius.

### 15-minute callback to 02b: throttling silently lowers your roofline

You already learned (02b) how to read power/thermal throttling with `nvidia-smi -q -d PERFORMANCE` and `clocks_throttle_reasons`. It belongs in a *cost/roofline* module because: **a throttled GPU silently lowers its achieved clocks, and therefore its achieved TFLOPS and bandwidth.** The spec-sheet roofline in lessons 2 and 5 assumes rated clocks; a card in `SW_Thermal_Slowdown` or `HW_Power_Brake` is running a *lower* roofline than you're paying for. So when your DCGM tensor-active is high but throughput is below model, the cause may not be your code — it may be a thermally throttled card delivering fewer real FLOPS per rented hour. Always check `clocks_throttle_reasons` before concluding a workload is compute-bound. (Mechanics: see 02b — not re-taught here.)

## Perspectives

**Purchasing/placement view.** The H100→H200 argument is a pure capacity/bandwidth call — same compute, same FP8 rate, different memory packaging (5×16GB HBM3 vs. 6×24GB HBM3e stacks). A purchasing decision that reasons "H200 costs more per hour, so it's worse value" without checking whether the workload is memory-bound gets the recommendation backwards; this is the single most common placement mistake at this level.

**Ops/reliability view.** The driver/CUDA/NCCL pin-everything discipline exists because the failure modes are silent, not loud. Imbue's team, running a real 70B training cluster, found training stalls traced to driver firmware that had drifted relative to their InfiniBand fabric — and the fix was rolling the driver *backward*, not forward. "Pin everything" doesn't mean "never touch it"; it means every change is canaried and reversible, in either direction.

**Hardware view.** MIG's isolation is a silicon fact, not a scheduling policy: 7 compute slices and 8 memory slices, hardware-partitioned, meaning one tenant's misbehavior cannot starve another's throughput the way software time-slicing or MPS sharing can. This is a materially stronger guarantee than "we set cgroup limits," and naming that precision in an interview is a genuine signal of depth.

**Economics view.** The H200-vs-H100 worked example below is the concrete version of "does the generation upgrade pay for itself" — and the B200 spec ambiguity flagged above is a live reminder that any such argument built today has a shelf life; a purchasing memo that hard-codes "192 GB, 8.0 TB/s" without a verification date is already a liability the moment a vendor updates a datasheet.

## Real-world use cases

- **Imbue, ["From bare metal to a 70B model: infrastructure set-up and scripts"](https://imbue.com/research/70b-infrastructure/).** A small team's real, warts-and-all account of training a 70B model from scratch on their own cluster. They describe health-checking for InfiniBand error rates and **outdated driver firmware**, and directly resolved a training-stalling issue by **downgrading to a previous NVIDIA driver version** — the single strongest, most concrete piece of evidence in this entire module for "version drift is a silent, real production failure mode," and a direct counter-example to "always update the driver." They also checked for GPU/NVSwitch **Xid/SXid errors** and GPUDirect RDMA failures as part of the same health-check discipline.
- **Together AI, ["A practitioner's guide to testing and running large GPU clusters for training generative AI models"](https://www.together.ai/blog/a-practitioners-guide-to-testing-and-running-large-gpu-clusters-for-training-generative-ai-models)** (Lucchese, Birkner, Hagai, Adams; Aug 2024). A systematic testing/validation playbook that complements Imbue's narrative war story — where Imbue tells you what broke, Together AI tells you what to check *before* it breaks, covering the same stack-hygiene territory (driver, fabric, NCCL health) from the checklist side.
- **NVIDIA/NCCL and PyTorch GitHub issue trackers — [NVIDIA/nccl#584](https://github.com/NVIDIA/nccl/issues/584) ("NCCL hangs during ncclSend and ncclRecv") and [pytorch/pytorch#43546](https://github.com/pytorch/pytorch/issues/43546).** Real, filed, publicly visible bug reports that ground the lesson's "NCCL doesn't error cleanly, it hangs" claim in actual incidents other engineers hit and documented, rather than a generic assertion — in both filed issues the trigger was a mismatched/out-of-order collective call across ranks (not a driver/toolkit version skew), the same "silent hang, not a clean error" symptom a version mismatch also produces.

## Worked example

**Decision 1: H200 vs H100 for a 70B inference server (purchasing argument).**

- **Capacity.** 70B at FP16 = 140 GB. H100 (80 GB) can't hold it → you need **2× H100 with tensor parallelism**. H200 (141 GB) holds the weights on **one GPU**, leaving little room for KV cache — so H200 is the clean single-GPU fit at FP8 (70 GB weights + generous cache, per lesson 5) and a tight fit at FP16.
- **Bandwidth.** Decode is memory-bound (one token at a time, re-reading weights + KV cache — lesson 4). H200's **4.8 TB/s vs H100's 3.35 TB/s (~1.43x)** directly raises decode throughput on that bandwidth-bound path — same compute silicon, faster memory.
- **Compute.** Identical (same GH100 die, ~1,979 FP8 TFLOPS both). Prefill/compute-bound sections don't improve — so if your workload is prefill-heavy and already fits, H200 buys less.
- **Cost argument.** 2× H100 ≈ $6/replica-hr (2026 snapshot rate) with cross-GPU all-reduce overhead per token; 1× H200 removes tensor parallelism *and* adds ~1.43x decode bandwidth. Unless the H200 hourly rate exceeds ~2x the H100 rate, the H200 wins on $/token for this memory-and-bandwidth-bound workload. **Write it up as: choose H200 — it collapses the replica to one GPU (no TP overhead) and its bandwidth lifts the memory-bound decode path; H100 only wins if H200 pricing is >2x or the workload is compute-bound and already fits.**

**Decision 2: the Imbue driver-downgrade incident (stack-hygiene narrative, worked as a diagnostic trace).**

Imbue's team, training a 70B model on their own bare-metal cluster, hit training stalls. Their diagnostic path — described in their public infrastructure write-up — included checking InfiniBand error rates, GPU/NVSwitch Xid/SXid error logs, and driver firmware version against what their InfiniBand fabric expected. The root cause traced to **driver firmware that had drifted out of a known-good pairing with the IB fabric** — and critically, the fix was **rolling the driver back**, not forward, to a previously verified-good version. Walk the causal chain the way an incident review would: symptom (stalls) → hypothesis space (compute? network? driver?) → the specific health checks that narrowed it (IB error counters, Xid logs, driver/firmware version audit) → root cause (driver regression relative to fabric firmware) → fix (downgrade, canaried before wide rollout).

**The lesson this teaches about "pin everything, never `latest`":** that rule does not mean "never move." It means every version — including the driver — is a deliberate, tested, reversible choice. "Pin to the newest" and "pin to the latest verified-good" are different policies, and Imbue's incident is the concrete case where they diverged and only one of them worked.

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
nvidia-smi -q | grep -i -A3 Xid    # any recent Xid errors — Imbue-style health check, cheap to run
```

Then:

1. Record every version in a small table (driver, driver-API CUDA, toolkit CUDA, cuDNN, NCCL, container-toolkit, GPU model).
2. Map the toolkit version against the **CUDA compatibility table** above: confirm the host driver meets the minimum for that CUDA major family (e.g. toolkit 12.x needs driver ≥ R525; toolkit 13.x needs driver ≥ R580).
3. Write the **blast-radius note**: for each component, state what breaks if you bump *only* that one. E.g. "Bump toolkit 12.4→13.x: requires driver ≥ R580 — current R5xx driver may be below it → app fails to init unless we also upgrade the driver or add `cuda-compat`." "Bump NCCL on one node only: risks a collective hang / slow-transport fallback across the job (see NVIDIA/nccl#584) — must upgrade all ranks together."
4. **Stretch (Imbue-style health check, no multi-node fabric required):** run the Xid-error grep above even on a single rented GPU with no active job, to establish what a clean baseline looks like — Xid codes logged by the driver flag hardware/driver-level errors independent of your workload, and knowing your baseline is empty is what makes a later non-empty result meaningful.

**Acceptance:** the [GPU Efficiency & Cost Report](../practice/gpu-efficiency-report/README.md) contains a **stack-inventory table** (all versions captured from the rented GPU) plus a **compatibility note** stating, for at least the driver and NCCL, exactly what would break if that single component were bumped out of step — and one line noting whether any Xid errors were present at baseline.

## Common pitfalls

1. **Assuming "newer driver = safer."** Imbue's real fix was a driver *downgrade* — a driver regression relative to a specific InfiniBand/fabric firmware pairing is a real failure mode. "Pin everything, never latest" means pin to a *known-good, tested* version, not necessarily the newest one.
2. **Bumping NCCL on a subset of nodes during a rolling upgrade.** NCCL must be upgraded in lockstep across every rank in a job — a skew doesn't reliably error, it hangs or silently falls back to a slower transport (see the real filed GitHub issues above).
3. **Treating B200 spec numbers as settled and single.** Real vendor-mirror documentation disagrees on exact capacity/bandwidth depending on SKU variant (B200 vs. HGX B200 vs. B300); always state a range and re-verify at purchase/build time rather than quoting one number with false precision.
4. **Treating MIG isolation as equivalent to software cgroups-style sharing.** It's hardware-partitioned — 7 compute slices and 8 memory slices, silicon-level isolation — a materially stronger guarantee worth naming precisely, since conflating it with time-slicing understates what it actually buys you.
5. **Skipping the cheap health checks because "the GPU shows up in `nvidia-smi`."** Imbue's diagnostic path (IB error rates, Xid/SXid logs, driver-firmware version audit) catches exactly the class of failure that looks fine at a glance and only surfaces as an unexplained stall under real multi-node load.

## Self-check

- Argue H200 over H100 for a 70B inference server, on capacity and bandwidth. **Answer:** H100's 80 GB can't hold 140 GB of FP16 weights, forcing 2× H100 + tensor parallelism (all-reduce overhead per token). H200's 141 GB holds it on one GPU (comfortably at FP8), removing TP entirely. And decode is memory-bandwidth-bound — H200's 4.8 TB/s vs H100's 3.35 TB/s (~1.43x) raises decode throughput on identical compute silicon. So H200 collapses the replica to one GPU and speeds the bound path; H100 only wins if H200 pricing is >2x or the job is compute-bound and already fits.
- A CUDA 12.x application on a host with driver R525 — will it run? What about a CUDA 13.x app on that same driver? **Answer:** The CUDA 12.x app runs — R525 is the minimum driver for the CUDA 12 major family, so minor-version compatibility lets any 12.x-built app run on it without a compat package, provided it doesn't call APIs newer than that driver exposes. The CUDA 13.x app will **not** run without intervention — CUDA 13.x requires driver ≥ R580; on an R525 host you'd need either a driver upgrade to ≥R580 or the `cuda-compat` forward-compatibility package as a stopgap.
- Imbue's engineering team fixed a multi-node training stall by *downgrading* their NVIDIA driver. What does this imply about the "always pin the latest driver" instinct? **Answer:** "Pin everything, never latest" means pin to a *known-good, tested* version — not necessarily the newest. A driver regression relative to your specific InfiniBand/fabric firmware is a real failure mode; Imbue's fix was rolling back to a previously-verified-good driver, the kind of change that should be canaried before wide rollout in either direction, up or down.
- What's the classic NCCL version-mismatch symptom, and is there real-world evidence for it beyond this lesson's claim? **Answer:** No clean error — the job **hangs on the first collective** (all-reduce/all-gather) because ranks can't negotiate a common protocol, or it silently **falls back to a slower transport**, producing a throughput cliff with no crash. Yes: this is documented in real, filed issues — NVIDIA/nccl#584 ("NCCL hangs during ncclSend and ncclRecv") and pytorch/pytorch#43546 — not just an assertion in this lesson. That's why NCCL (and cuDNN) must be pinned identically across every node in a job.

## Connections & what's next

This lesson closes the hardware-literacy arc of the module: lesson 1 taught you to distrust raw GPU-Util, lesson 2 gave you the roofline to place a workload honestly, lesson 3 and 4 built the memory/bandwidth model that decode throughput and batching depend on, lesson 5 added precision as a lever on both axes of the roofline, and this lesson adds the two remaining variables — which generation, and whether the software stack underneath it is actually correct — without which none of the previous five lessons' numbers are trustworthy on real rented hardware.

Next: **[03.7 · Capstone — cost per unit of useful work](07-capstone-cost-per-useful-work.md)** is where all six lessons converge into a single deliverable: a defended SKU recommendation that states achieved-vs-spec TFLOPS, the FP16→FP8 delta, the utilization-lie demo, *and* — carrying this lesson's thread forward directly — the generation choice and a stack-hygiene note, all priced against live 2026 GPU-rental data and explicitly dated as a snapshot.

## References & further reading

**Primary sources**
- [NVIDIA CUDA Compatibility documentation](https://docs.nvidia.com/deploy/cuda-compatibility/) (including the `minor-version-compatibility.html` and `forward-compatibility.html` sub-pages) — the authoritative minimum-driver tables behind this lesson's CUDA/driver compatibility table.
- [NVIDIA H200 datasheet / product page](https://www.nvidia.com/en-us/data-center/h200/) — canonical 141 GB / 4.8 TB/s source, and the 6×24GB HBM3e stack detail.
- [NVIDIA H100 Tensor Core GPU Architecture whitepaper](https://resources.nvidia.com/en-us-tensor-core/gtc22-whitepaper-hopper) — canonical dense/sparse TFLOPS and HBM3 figures used throughout this table.
- [NVIDIA data-center GPU product hub](https://www.nvidia.com/en-us/data-center/) — cross-check every spec-table number here before quoting it in a report, especially the B200 row flagged above as SKU-ambiguous.

**Real-world engineering blogs**
- Imbue, ["From bare metal to a 70B model: infrastructure set-up and scripts"](https://imbue.com/research/70b-infrastructure/) — the real driver-downgrade incident this lesson's worked example traces end to end.
- Together AI, ["A practitioner's guide to testing and running large GPU clusters for training generative AI models"](https://www.together.ai/blog/a-practitioners-guide-to-testing-and-running-large-gpu-clusters-for-training-generative-ai-models) — a systematic pre-failure testing checklist that complements Imbue's incident narrative.
- [NVIDIA/nccl#584](https://github.com/NVIDIA/nccl/issues/584) and [pytorch/pytorch#43546](https://github.com/pytorch/pytorch/issues/43546) — real, filed GitHub issues documenting NCCL's hang-not-error failure mode (mismatched/out-of-order collective calls across ranks in both cases), the same symptom class a driver/NCCL version skew also triggers.

**Deeper dives**
- NVIDIA MIG documentation (linked from the [NVIDIA data-center GPU product hub](https://www.nvidia.com/en-us/data-center/) above) — for the full compute-slice/memory-slice combination menu beyond the "7 and 8" headline this lesson states.
