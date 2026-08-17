---
lesson: "05.5"
title: "GPU health and XID errors"
module: "05"
concept: "GPU health and XID errors"
status: not-started
est_time: "6h"
prev: "04-attribution.md"
next: "06-inference-slos.md"
artifacts: []
sources: 16
---

# 05.5 · GPU health and XID errors

> **Concept.** An XID is the GPU's own crash code; your job is to classify each one as "the silicon is dying — cordon and drain" versus "the tenant's kernel is buggy — log and move on," and wire that decision into node conditions automatically.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

04 gave you the attribution join — per-namespace `SM_ACTIVE` GPU-hours, honest and named. That answers "who is wasting capacity." This lesson answers a different question the fleet asks every day: **"is this GPU even trustworthy to schedule on?"** Attribution assumes the hardware underneath is healthy; XID triage is what verifies that assumption, and it is the piece that keeps a *reliability* incident from masquerading as a *utilisation* one. A GPU with a pending row remap will happily report `SM_ACTIVE = 0.7` while silently running with part of its framebuffer offlined; a GPU that has fallen off the PCIe bus will report nothing at all and leave a stale gauge sitting in Prometheus that looks exactly like a healthy idle GPU. You need to know which problem you are looking at before the utilisation numbers mean anything.

Everything below is checked against three primary sources you can read yourself: **`nverror.h` from NVIDIA's open GPU kernel modules** (the header that literally defines the XID numbers), **`dcgm_errors.h` and `DcgmHealthWatch.cpp` from NVIDIA/DCGM** (the health system's error catalogue and the fields each watch actually subscribes to), and **`dcgm-exporter`'s `default-counters.csv`** (what actually reaches Prometheus). Where a number, symbol or CLI flag appears below, it came out of one of those trees, not out of a blog post.

What this unlocks: 05.6 needs healthy GPUs to reason about inference SLOs on, 05.7's profiling ladder needs "it's dying" ruled out before you spend Nsight-grade effort on "it's slow," and the capstone (05.8) needs a fleet where "idle" means "genuinely idle," not "quietly degraded."

## Why this matters

You already run node-level health on 40+ clusters. On CPU boxes a bad DIMM throws a Machine Check Exception, `rasdaemon` catches it, and Node Problem Detector taints the node. GPUs have the same failure surface — ECC, bus drops, interconnect faults, firmware hangs — but the signal arrives as an **XID**: a numeric error the NVIDIA driver prints to the kernel log and DCGM surfaces as field 230. Get the classification wrong in either direction and it is expensive:

- **Over-cordon.** A tenant ships a CUDA kernel with an out-of-bounds write. XID 31 fires. You cordon a healthy 8×H100 node and page the on-call. On a $20–30/hr node (mid-tier GPU cloud, 2026 order of magnitude) a four-hour false cordon is $80–120 of capacity you paid for and refused to schedule — and multiplied across every ML team on the cluster, you have turned a user bug into a fleet-availability incident.
- **Under-cordon.** A double-bit ECC error (XID 48) corrupts a training step's weights. You log it and keep scheduling. The GPU silently produces garbage — a corrupted checkpoint, a NaN loss twelve hours later, or an *uncontained* fault (XID 95) that also poisons the co-tenant's context. A destroyed multi-day run on 64 GPUs at $2.50/GPU-hr is $3,840 of compute plus the wall-clock you cannot buy back.

This is not a rare edge case. It is an operational constant at fleet scale. Meta's published account of training Llama 3 (arXiv 2407.21783, §3.3.2) gives the numbers: across a 54-day snapshot on **16,384 H100 GPUs**, the job hit **466 total interruptions** — 47 planned and **419 unexpected**. That is roughly **one unplanned failure every three hours**. GPU-related causes (GPU faults including NVLink, plus HBM3 memory failures) were about **58.7%** of the unexpected interruptions. And Meta still held **>90% effective training time** across the run. That gap — hundreds of hardware faults a month and still >90% uptime — is not luck. It is automated classification and remediation running continuously, which is exactly the skill this lesson builds.

The differentiator for the role you are targeting is knowing *which* XIDs are hardware death sentences and which are application noise, knowing what "recoverable" actually means at the silicon level, and having the remediation wired so a human never has to remember the table at 3am.

## What's new here (calibration)

You know the Prometheus/NPD/taint machinery cold — none of that is new. What is new:

- **XID is a discriminated union, not a severity.** The same field (`DCGM_FI_DEV_XID_ERRORS`) carries "user made a mistake" and "RMA this board" with no built-in severity flag. The number *is* the classification. You must carry the catalogue — or, as §7 shows, get the driver to carry it for you.
- **Row-remapping is the GPU's spare-tire mechanism** (Ampere and later; it replaced page retirement). It has hard, small, countable limits — **8 spare rows per DRAM bank, 512 uncorrectable remaps per GPU** — and a remap does not take effect until the next GPU reset. This lifecycle has no CPU analogue.
- **Containment is a hardware property.** XID 94 vs 95 is not a severity gradation someone chose; it is whether the memory-error containment logic managed to confine the fault to the offending context or not. That single bit decides whether your co-tenant's results are trustworthy.
- **The driver now ships a recommendation.** Recent drivers expose `NVML_FI_DEV_GET_GPU_RECOVERY_ACTION` — a five-valued enum (`NONE`/`GPU_RESET`/`NODE_REBOOT`/`DRAIN_P2P`/`DRAIN_AND_RESET`) that says what to do, and XID 154 fires when it changes. DCGM's `DRIVER` health watch consumes it directly. This is the single biggest change to GPU health automation in the last two driver generations and most write-ups have not caught up.
- **DCGM's XID surfacing is lossy and sticky.** It is a *mirror*, and a partial one. Two open upstream issues (below) show XIDs missing from the metric that were in `dmesg`, and the metric staying at its last value long after recovery. Your automation must know this.
- **Reset ≠ reboot, and reset scope is not always one GPU.** MIG, NVLink/NVSwitch fabrics and P2P traffic all change what "reset this GPU" means.

## Core concepts

### 1. Where an XID actually comes from

Start from the hardware, because the whole classification problem is downstream of one design decision.

A GPU executes work from **channels** — per-context ring buffers of pushbuffer commands fed to the engines (graphics/compute, copy engines, video decode). When an engine hits a condition it cannot complete — a memory management unit (MMU) fault because a kernel dereferenced an address outside its page tables, an ECC error the hardware cannot correct, a timeout waiting on a firmware RPC — it raises an interrupt. The driver's **Robust Channel (RC) error handling** path catches that interrupt, decides which channel and which context is implicated, tears down or resets what it must, and *reports*. That report is the XID.

That is why the symbolic names in NVIDIA's own header are called `ROBUST_CHANNEL_*`. From `open-gpu-kernel-modules/src/common/sdk/nvidia/inc/nverror.h`:

```c
#define ROBUST_CHANNEL_GR_EXCEPTION                     (13)
#define ROBUST_CHANNEL_FIFO_ERROR_MMU_ERR_FLT           (31)
#define ROBUST_CHANNEL_PREEMPTIVE_REMOVAL               (45)
#define ROBUST_CHANNEL_GPU_ECC_DBE                      (48)
#define INFOROM_PAGE_RETIREMENT_EVENT                   (63)
#define INFOROM_PAGE_RETIREMENT_FAILURE                 (64)
#define NVLINK_ERROR                                    (74)
#define ROBUST_CHANNEL_GPU_HAS_FALLEN_OFF_THE_BUS       (79)
#define EXCESSIVE_SBE_INTERRUPTS                        (92)
#define ROBUST_CHANNEL_CONTAINED_ERROR                  (94)
#define ROBUST_CHANNEL_UNCONTAINED_ERROR                (95)
#define GSP_RPC_TIMEOUT                                (119)
#define GSP_ERROR                                      (120)
#define UNRECOVERABLE_ECC_ERROR_ESCAPE                 (140)
#define GPU_RECOVERY_ACTION_CHANGED                    (154)
```

**This header is the authoritative anchor for XID numbers**, and it is worth internalising that the list is a flat enumeration of *reporting reasons*, not a severity ladder. XID 13 (a graphics engine exception raised because an app did something illegal) and XID 79 (the device stopped responding on PCIe) are adjacent entries in the same table. Nothing in the number tells you which is which. That is the entire problem this lesson solves.

Here is the full path from the fault to a cordon decision:

```
  SILICON                    DRIVER (kernel)               USERSPACE              CLUSTER
  ───────                    ───────────────               ─────────              ───────

  ┌──────────────┐
  │ SM / MMU     │ illegal address, no PTE
  │ HBM ECC      │ uncorrectable bit flip        ┌───────────────────┐
  │ NVLink PHY   │ CRC / link down          ───▶ │ Robust Channel    │
  │ GSP firmware │ RPC timeout                   │ error handler     │
  │ PCIe root    │ device stopped responding     │ (RC recovery)     │
  └──────────────┘                               └─────────┬─────────┘
                                                           │
                              ┌────────────────────────────┼──────────────────────────┐
                              │                            │                          │
                              ▼                            ▼                          ▼
                   ┌────────────────────┐      ┌──────────────────────┐   ┌────────────────────┐
                   │ printk to kernel   │      │ NVML error state     │   │ channel/context    │
                   │ log:               │      │ + GPU RECOVERY       │   │ teardown; app gets │
                   │ "NVRM: Xid (PCI:   │      │ ACTION enum          │   │ CUDA "unspecified  │
                   │  0000:65:00): 94"  │      │ (Xid 154 on change)  │   │  launch failure"   │
                   └─────────┬──────────┘      └──────────┬───────────┘   └────────────────────┘
                             │                            │
              ┌──────────────┘                            │
              │  FORENSIC PATH                            │  TELEMETRY PATH
              ▼  (complete, unstructured)                 ▼  (structured, LOSSY)
   ┌──────────────────────┐                    ┌───────────────────────────┐
   │ journald / syslog    │                    │ nv-hostengine             │
   │ NPD kernel-monitor   │                    │  field 230 XID_ERROR      │
   │ NVSentinel syslog-hm │                    │  health watch systems     │
   └──────────┬───────────┘                    └────────────┬──────────────┘
              │                                             │
              │                                  dcgm-exporter :9400
              │                                  DCGM_FI_DEV_XID_ERRORS  (gauge = XID number)
              │                                             │
              └──────────────┬──────────────────────────────┘
                             ▼
                  ┌─────────────────────────┐
                  │ XID CLASSIFIER          │  ← the table in §3
                  │ (rule engine / CEL)     │
                  └──────────┬──────────────┘
                             ▼
                  cordon · drain · reset · uncordon · RMA
```

Two things to take from the diagram. First, **there are two independent paths out of the driver, and they do not carry the same information.** The kernel log line is complete and unstructured; the DCGM field is structured and incomplete. Second, **the app-visible symptom is a third thing entirely** — a CUDA error inside the container — which is why tenants file "my job crashed with `unspecified launch failure`" tickets that you have to correlate back to an XID yourself.

The kernel-log line looks like this (representative — the exact tail varies by XID class and driver branch):

```
[168423.771092] NVRM: Xid (PCI:0000:65:00): 31, pid=31245, name=python3,
                Ch 00000018, intr 00000000. MMU Fault: ENGINE GRAPHICS GPCCLIENT_T1_0
                faulted @ 0x7f2c_4a000000. Fault is of type FAULT_PDE ACCESS_TYPE_READ
[168429.114550] NVRM: Xid (PCI:0000:65:00): 94, pid='<unknown>', name=<unknown>,
                Contained: CE User Channel (0x9). RST: No, D-RST: No
```

Read it field by field: `PCI:0000:65:00` is the **PCI bus address**, which is how you map the message to a GPU index and then to a UUID (`nvidia-smi --query-gpu=index,pci.bus_id,uuid --format=csv`); the number after the colon is the XID; `pid=`/`name=` name the offending process **when the driver can attribute it** (engine-exception XIDs) and are `<unknown>` when it cannot (asynchronous hardware faults); `Ch` is the channel ID. For XID 94/95, `RST: No, D-RST: No` reports whether the driver performed a reset as part of containment.

**Attribution asymmetry is structural, not accidental.** A page fault is raised *by* a running context, so the driver knows the PID. A double-bit ECC error in a DRAM row is discovered when someone reads that row, which may be any context or the driver itself, so there is often nobody to blame. This is the first-order reason the log-only bucket is exactly the set of XIDs with reliable PID attribution.

### 2. What "recoverable" means, precisely

"Recoverable" gets used three different ways in GPU operations, and conflating them is how people build broken runbooks. Separate them:

| Level | Question | Who decides | Observable |
|---|---|---|---|
| **Context-recoverable** | Did the fault stay inside the faulting CUDA context? | Hardware containment logic + driver | XID 94 (contained) vs 95 (uncontained); the app dies, the GPU survives |
| **Reset-recoverable** | Can a GPU reset (or node reboot) return the device to full capacity? | Driver — exposed as the GPU Recovery Action enum | `GPU_RESET` / `NODE_REBOOT` / `DRAIN_AND_RESET` vs a persistent fault |
| **Field-recoverable** | Can the board be returned to service at all, or does it need replacement? | Row-remap budget, remap-failure flag, recurrence | `ROW_REMAP_FAILURE=1`, exhausted bank histogram, repeat XID 79 after reseat |

A single XID rarely answers all three. XID 94 answers the first (yes, contained). It says nothing about the third — the same board may be three remaps from exhaustion. This is why the operational answer is never "look up the XID"; it is "look up the XID, *then* read the state fields it points at."

### 3. The XID catalogue you must carry

Split by **what you do**, not by NVIDIA's cause columns. Symbols are from `nverror.h`; DCGM error codes are from `dcgm_errors.h`.

#### 3a. CORDON + DRAIN — hardware fault, stop scheduling

| XID | `nverror.h` symbol | What it means | Reset-recoverable? | Action |
|---|---|---|---|---|
| **48** | `ROBUST_CHANNEL_GPU_ECC_DBE` | Uncorrectable (double-bit) ECC error in framebuffer. Data is *already* wrong. | Usually — after the row is remapped | Cordon, drain, reset. Recurrence or remap failure → RMA. |
| **63** | `INFOROM_PAGE_RETIREMENT_EVENT` | A row-remap (or, pre-Ampere, page-retirement) entry was **successfully recorded to InfoROM**. Not yet active. | Yes — requires a reset to activate | Cordon, drain, schedule reset window, verify pending flag clears. |
| **64** | `INFOROM_PAGE_RETIREMENT_FAILURE` | The remap **could not be recorded**. The self-repair path itself failed. | **No** | Cordon, drain, **RMA**. Do not uncordon. |
| **74** | `NVLINK_ERROR` | Fault on an NVLink connection (GPU↔GPU or GPU↔NVSwitch). Corrupts collectives. | Sometimes, after link retrain | Cordon, drain; reset/reseat; recurring → RMA. Check partner GPU too. |
| **79** | `ROBUST_CHANNEL_GPU_HAS_FALLEN_OFF_THE_BUS` | Device stopped responding on PCIe. NVML can no longer read it at all. | Node reboot, sometimes | Cordon, drain, node reboot. Recurring after reseat/slot-swap → RMA. |
| **92** | `EXCESSIVE_SBE_INTERRUPTS` | Single-bit ECC error *rate* exceeded threshold. Errors were corrected. | N/A — nothing is broken yet | **Do not cordon on one.** Trend it; pre-emptively schedule maintenance. |
| **94** | `ROBUST_CHANNEL_CONTAINED_ERROR` | Uncorrectable error **contained** to the faulting context. A row will be remapped. | Yes | Cordon, drain gracefully, reset to apply remap, verify. |
| **95** | `ROBUST_CHANNEL_UNCONTAINED_ERROR` | Uncorrectable error **not contained** — other contexts may be poisoned. | Yes, but | Cordon, drain **hard**, reset; treat co-tenant output as suspect. |
| **119** | `GSP_RPC_TIMEOUT` | The GPU System Processor stopped answering an RPC. Device effectively wedged. | Usually, via reset/reboot | Cordon, drain, reset. Recurring → driver bug or RMA; collect logs. |
| **120** | `GSP_ERROR` | GSP firmware fault. | Usually | Same as 119. |
| **140** | `UNRECOVERABLE_ECC_ERROR_ESCAPE` | An uncorrectable ECC error escaped the containment/poison machinery. | Rarely | Cordon, drain hard, reset, escalate. Treat results as suspect. |
| **62** | `PMU_HALT_ERROR` | Internal micro-controller (PMU) halted. | Usually via reset | Cordon, drain, reset; recurring → RMA. |
| **69** | `GR_CLASS_ERROR` | Graphics engine class error — frequently firmware/driver-level, not app. | Usually | Cordon-and-observe; recurring across tenants → hardware. |
| **143** | `GPU_INIT_ERROR` | GPU failed to initialise. Node never reaches a schedulable state. | Node reboot | Cordon, reboot; recurring → RMA. |
| **171 / 172** | `UNCORRECTABLE_DRAM_ERROR` / `UNCORRECTABLE_SRAM_ERROR` | Newer explicit uncorrectable-memory reports (Hopper/Blackwell-era enumerations). | Depends | Treat as 48-class: cordon, drain, reset, watch counters. |

#### 3b. LOG ONLY — almost always the tenant's bug

| XID | `nverror.h` symbol | What it means | Action |
|---|---|---|---|
| **13** | `ROBUST_CHANNEL_GR_EXCEPTION` | Graphics/compute engine exception: illegal instruction, out-of-bounds shared-memory access, bad kernel launch config. | Log, attribute to pod, notify tenant. **Do not cordon.** |
| **31** | `ROBUST_CHANNEL_FIFO_ERROR_MMU_ERR_FLT` | MMU fault — the kernel touched an address with no valid page-table entry. Classic CUDA OOB / use-after-free / freed-then-used pointer. | Log, attribute to pod, notify tenant. **Do not cordon.** |
| **43** | `ROBUST_CHANNEL_RESETCHANNEL_VERIF_ERROR` | The driver reset a channel because *that application's* work hit an error. GPU is fine. | Log, attribute to pod. **Do not cordon.** |
| **32** | `ROBUST_CHANNEL_PBDMA_ERROR` | Invalid/corrupted pushbuffer stream. Usually a buggy or mismatched userspace/driver pair. | Log; if it follows a driver upgrade, suspect the driver, not the board. |
| **45** | `ROBUST_CHANNEL_PREEMPTIVE_REMOVAL` | The driver preemptively tore down a channel — after a Ctrl-C, an OOM-kill, or *a previous XID*. | Log. **Almost always a symptom.** If it trails a hardware XID, act on that one. |
| **31/13 in a flood, across tenants** | — | Same numbers, different blast pattern. | **Flips to hardware suspicion.** See below. |

**The discriminator is the blast pattern, not just the number.** One pod repeatedly throwing XID 31 is a user bug. *Every* pod that lands on GPU 3 throwing XID 31 within minutes of scheduling is a bad board or a bad driver install. Encode this as a rule: `count of distinct namespaces raising an app-class XID on the same UUID within 1h >= 3` → escalate to hardware review. Meta's data corroborates the need for a narrow log-only bucket from the other direction: the overwhelming majority of their 419 unexpected interruptions were genuine hardware, so any false-hardware noise you generate by cordoning on XID 31 lands on top of a queue that already has real faults in it.

#### 3c. Informational / newer codes worth recognising

| XID | Symbol | Why you care |
|---|---|---|
| **61** | `PMU_BREAKPOINT` | Micro-controller breakpoint/warning. Informational; watch for it preceding 62. |
| **80** | `PBDMA_PUSHBUFFER_CRC_MISMATCH` | Corrupted data reached the GPU — often the *host* side (bad RAM, bad PCIe). Check the host before blaming the GPU. |
| **93** | `INFOROM_ERASE_LIMIT_EXCEEDED` | The InfoROM's write-wear budget is exhausted. Not a compute fault; blocks future remap recording. Plan replacement. |
| **110** | `SEC_FAULT_ERROR` | Security fault. Escalate; do not silently retry. |
| **121** | `C2C_ERROR` | Grace–Hopper chip-to-chip link corrected error. GH200/GB200 only. |
| **122–125** | `SPI_PMU_RPC_*_FAIL`, `INFOROM_FS_ERROR` | Serial-flash / InfoROM filesystem faults. The board's own persistent state is unreliable — remap recording may silently fail next time. |
| **154** | `GPU_RECOVERY_ACTION_CHANGED` | **The driver changed its recommended recovery action.** See §7 — this is the one to wire up. |
| **156/157** | `RESOURCE_RETIREMENT_EVENT` / `_FAILURE` | Blackwell-era generalisation of 63/64 beyond DRAM rows. |
| **168** | `REDUCED_GPU_MEMORY_CAPACITY` | The GPU is running with less framebuffer than nameplate — memory was offlined. Your capacity model is now wrong. |
| **177** | `BANK_REMAPPING_EVENT` | Bank-level (not row-level) remapping occurred. |

On an NVSwitch fabric (HGX 8-GPU boards, NVL72 racks), switch-side faults arrive as a parallel **SXID** and surface through DCGM as `DCGM_FI_DEV_SXID_FATAL_ERROR` / `DCGM_FI_DEV_SXID_NON_FATAL_ERROR`, raising `DCGM_FR_SXID_ERROR` (109) rather than a per-GPU XID. Same triage discipline, different bus, and a fatal SXID typically implicates the *node*, not one GPU.

### 4. ECC, containment, and the row-remap lifecycle

This is the mechanism behind half the cordon table, so it is worth doing properly.

HBM on data-center parts is ECC-protected. The code corrects single-bit errors in flight and detects double-bit errors it cannot correct. Two consequences:

- **Single-bit errors (SBE)** cost you nothing at the time — the data delivered is correct — but the *rate* is a wear signal. When the rate crosses the driver's threshold you get **XID 92**. Treat it exactly like a rising SMART reallocated-sector count on a disk: not an outage, a countdown.
- **Double-bit errors (DBE)** cannot be corrected. Something must decide what happens to the data and to the contexts that might have read it. That decision is **containment**.

**Containment** is the hardware+driver machinery that tries to confine an uncorrectable error to the context that touched the bad memory. When it succeeds you get **XID 94** and only the offending process dies. When it fails you get **XID 95**, and every context that shared the GPU is potentially poisoned — which is why 95 forces you to invalidate co-tenant results, not merely restart a pod. On MIG-partitioned GPUs, containment interacts with partitioning: an error contained to one MIG instance can leave the others running, which is a feature until your automation assumes "GPU 3 is bad" means all seven instances.

Then comes repair. Pre-Ampere GPUs used **page retirement**: mark a 4 KB page bad, never use it again, capped at 64 retirements per GPU (DCGM still carries `DCGM_FI_DEV_PAGE_RETIRED_SBE_TOTAL` / `_DBE_TOTAL` / `_PENDING`, fields 390/391/392, for that generation). Ampere and later use **row remapping**, which is finer-grained and repairs at the DRAM level rather than the address-space level.

The mechanism, with the real numbers (NVIDIA GPU Memory Error Management, r575):

```
  ONE DRAM BANK (many banks per GPU; 640 on an 80 GB A100/H100-class part)

   ┌──────────────────────────────────────────────────────────────┐
   │ addressable rows                          │  8 SPARE ROWS    │
   │ ██████████████████████████████████████    │  ▒ ▒ ▒ ▒ ▒ ▒ ▒ ▒ │
   └──────────────────────────────────────────────────────────────┘
                        │                              ▲
       uncorrectable    │  row marked bad              │ remap target
       error in a row ──┘                              │
                                                       │
   RECORD  ──▶ InfoROM entry written        ──▶ Xid 63  (…or Xid 64 if the write fails)
   PENDING ──▶ DCGM_FI_DEV_ROW_REMAP_PENDING = 1        ← repair scheduled, NOT ACTIVE
   RESET   ──▶ GPU reset re-reads InfoROM, programs the remap
   ACTIVE  ──▶ DCGM_FI_DEV_ROW_REMAP_UNCORRECTABLE_TOTAL += 1, PENDING = 0

  BUDGET (hard limits — a remap FAILURE flag is raised when any is hit):
    • 8 uncorrectable-error rows already remapped in the SAME bank
    • a remap attempt on a row that was ALREADY remapped
    • 512 total uncorrectable remappings on the GPU

  BANK REMAP AVAILABILITY HISTOGRAM   (nvidia-smi -q -d ROW_REMAPPER)
    Max      : 8 spare rows left   ███████████████████████████████ 635 banks
    High     : 7                   █ 3 banks
    Partial  : 2–6                 · 0 banks
    Low      : 1                   █ 2 banks     ◀── these two banks are one
    None     : 0                   · 0 banks         error away from a failure
```

Three things follow that people get wrong:

1. **A recorded remap is not an applied remap.** Between XID 63 and the next GPU reset, the board is running with a known-bad row still in service. `DCGM_FI_DEV_ROW_REMAP_PENDING` (field 396) is the flag. Do not uncordon on a pending remap.
2. **The budget is small, not "hundreds per bank."** Eight per bank, 512 uncorrectable total per GPU. A board eating a handful of uncorrectable errors a week is on a visible clock, and the histogram is what shows you which bank is about to tip.
3. **The histogram is diagnostic, not programmatic.** NVIDIA documents the bank availability histogram as reference output only — it is not exposed through NVML or DCGM. Your automation trends the *counters* (`ROW_REMAP_UNCORRECTABLE_TOTAL`, field 393) and the *flags* (`ROW_REMAP_FAILED`, 395; `ROW_REMAP_PENDING`, 396); the histogram is what a human reads during triage.

Real output from a healthy board:

```console
$ nvidia-smi -i 3 -q -d ROW_REMAPPER

==============NVSMI LOG==============
GPU 00000000:65:00.0
    Remapped Rows
        Correctable Error                 : 0
        Uncorrectable Error               : 2
        Pending                           : Yes
        Remapping Failure Occurred        : No
        Bank Remap Availability Histogram
            Max                           : 638 bank(s)
            High                          : 2 bank(s)
            Partial                       : 0 bank(s)
            Low                           : 0 bank(s)
            None                          : 0 bank(s)
```

Read it: two uncorrectable-error rows have been recorded, `Pending : Yes` means at least one has **not** been applied — this board needs a reset before it should take work — and no bank is anywhere near exhaustion, so this is a reset-and-return case, not an RMA.

**Mental model.** 92 = the tyre is losing air slowly. 48/94/95 = it blew; 94 means it stayed in its own lane, 95 means it took out the neighbour. 63 = the spare is on but you have not torqued the lugs (reset). 64 = you are out of spares *and the jack broke*. 168 = you finished the journey on a space-saver and your capacity model still thinks you have five full wheels.

### 5. The two signals that live here — events vs capacity

Conflating these is a real operational mistake, and it is the thing that separates a runbook from a fleet-hygiene programme:

- **Event-based alerting.** A fresh 63/64/94/95 arrives. Act now: cordon, drain, reset. Latency target: seconds to minutes. Paging is appropriate.
- **Capacity trending.** `DCGM_FI_DEV_ROW_REMAP_UNCORRECTABLE_TOTAL` is climbing on a board that is healthy *today*. This is a "future 64." Latency target: the next planned maintenance window. Paging is **not** appropriate; a weekly ranked report is.

DCGM itself encodes both. Its memory health watch raises `DCGM_FR_ROW_REMAP_FAILURE` (80) as a **FAILURE** when the failure flag is set, and `DCGM_FR_UNCORRECTABLE_ROW_REMAP_LIMIT` (132) as a **WARNING** when the uncorrectable remap count crosses DCGM's internal Ampere+ limit. Two different result levels for two different response times, from the same subsystem.

### 6. DCGM as the surfacing layer — and its limits

DCGM exposes the **last XID the GPU raised** as field **230**, `DCGM_FI_DEV_XID_ERROR` in the header (`DCGM_FI_DEV_XID_ERRORS` as the exporter's metric name). Its documented semantics: *"XID errors. The value is the specific XID error."*

Scraped through `dcgm-exporter` it becomes a gauge whose **value is the XID number**:

```
DCGM_FI_DEV_XID_ERRORS{gpu="3",UUID="GPU-a1b2c3d4-...",device="nvidia3",
  modelName="NVIDIA H100 80GB HBM3",Hostname="gpu-node-17",
  exported_namespace="team-research",exported_pod="trainer-0",
  exported_container="main"} 94
```

Four properties you must design around:

1. **It is last-value, not a counter.** A `94` sitting there means "the most recent XID was 94," not "94 errors." Never `rate()` it. Alert on membership in a set, and treat any appearance as an edge — `changes(DCGM_FI_DEV_XID_ERRORS[5m]) > 0` combined with the value test, or the exporter's expression metrics.
2. **It is sticky.** dcgm-exporter issue **#500** (dcgm-exporter 3.3.8–3.6.0, DCGM 3.3.8, driver 550.127.08, H100 80GB HBM3) reports the metric holding a stale XID long after `dcgmi` and `nvidia-smi` both show the GPU healthy again, with restarting the exporter as the only reliable clear. **Consequence:** never build "the GPU is healthy again" on the *absence* of a value in this field. Build recovery verification on `ROW_REMAP_PENDING`, `ROW_REMAP_FAILED`, the recovery-action field, and a `dcgmi diag` pass.
3. **It is lossy.** DCGM issue **#235** (dcgm-exporter 3.3.7-3.5.0, driver 550.127.08) reports XID 62 appearing in `dmesg` but never in `DCGM_FI_DEV_XID_ERRORS`; the exporter only surfaced the trailing XID 45. **Consequence:** keep the kernel-log path as a first-class input, not a fallback. Node Problem Detector's kernel monitor or NVSentinel's syslog health monitor is not redundancy — it is coverage.
4. **It ships enabled, but the fields you need for follow-up do not.** In `dcgm-exporter`'s `default-counters.csv`, `DCGM_FI_DEV_XID_ERRORS` is uncommented, and so are `DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS`, `DCGM_FI_DEV_CORRECTABLE_REMAPPED_ROWS` and `DCGM_FI_DEV_ROW_REMAP_FAILURE`. **`DCGM_FI_DEV_ROW_REMAP_PENDING` is not in the default set at all**, and the ECC totals (`DCGM_FI_DEV_ECC_SBE_VOL_TOTAL`, `_DBE_VOL_TOTAL`, `_SBE_AGG_TOTAL`, `_DBE_AGG_TOTAL`) ship **commented out**. The single most important field for "is this board safe to uncordon" is the one you have to add by hand. Add it, along with the ECC totals, in your custom counters CSV — and remember from 05.3 that a custom CSV **replaces** the default set rather than extending it.

The exporter also offers opt-in expression metrics — `DCGM_EXP_XID_ERRORS_COUNT` (a gauge counting XID samples in a configured window, labelled by the numeric `xid`) and `DCGM_EXP_XID_ERRORS_TOTAL` (a cumulative counter since exporter start). These fix property 1 for you: a real count, per XID value, that you *can* rate. They ship commented out and depend on `DCGM_FI_DEV_XID_ERRORS` also being enabled. Cardinality warning: labelling by XID number multiplies your XID series by the number of distinct codes observed — bounded in practice, but check it against the budget you set in 05.3.

### 7. The driver now tells you what to do: GPU Recovery Action

This is the mechanism most XID write-ups predate, and it is the highest-leverage thing in this lesson.

Recent drivers expose an NVML field, `NVML_FI_DEV_GET_GPU_RECOVERY_ACTION`, whose value is the driver's own assessment of what the operator must do. The enum, and what each value obliges you to do:

| Value | Enum | The driver is saying | Your move |
|---|---|---|---|
| 0 | `NVML_GPU_RECOVERY_ACTION_NONE` | Nothing is wrong. | Schedule freely. |
| 1 | `NVML_GPU_RECOVERY_ACTION_GPU_RESET` | A fault requires a device reset to clear. | Kill all GPU processes on that device, reset it, resume. |
| 2 | `NVML_GPU_RECOVERY_ACTION_NODE_REBOOT` | The fault may have left the **OS** in an inconsistent state. | Reboot the node. A device reset is not sufficient. |
| 3 | `NVML_GPU_RECOVERY_ACTION_DRAIN_P2P` | Peer-to-peer traffic must be quiesced before anything else can be determined. | Stop all P2P/NVLink traffic, disable UVM persistence, **re-query** — it will then return one of the other values. |
| 4 | `NVML_GPU_RECOVERY_ACTION_DRAIN_AND_RESET` | The GPU is running at **reduced capacity** (framebuffer offlined, MIG instances down). Existing unaffected work is safe. | Stop *new* scheduling, let running work reach a checkpoint, then reset to regain full capacity. |

**XID 154 (`GPU_RECOVERY_ACTION_CHANGED`) fires when this value changes**, which turns "poll a field" into "react to an event."

DCGM consumes it directly. Its `DCGM_HEALTH_WATCH_DRIVER` watch subscribes to `DCGM_FI_DEV_GPU_RECOVERY_ACTION` and maps the enum onto health results:

| Recovery action | DCGM health result | DCGM error raised |
|---|---|---|
| `NONE` | PASS | — |
| `GPU_RESET` | **FAIL** | `DCGM_FR_GPU_RECOVERY_RESET` (125) |
| `NODE_REBOOT` | **FAIL** | `DCGM_FR_GPU_RECOVERY_REBOOT` (126) |
| `DRAIN_P2P` | **WARN** | `DCGM_FR_GPU_RECOVERY_DRAIN_P2P` (127) |
| `DRAIN_AND_RESET` | **FAIL** | `DCGM_FR_GPU_RECOVERY_DRAIN_RESET` (128) |

Note `DRAIN_P2P` is the only one that is a *warning*: it is an intermediate state, not a verdict. Your controller must handle it as "quiesce, then re-evaluate," and a controller that treats WARN as "ignore" will sit forever on a GPU that is waiting for P2P to drain before it can tell you the real answer.

**Why this matters for your design:** the XID table is a decoder for *what happened*; the recovery action is a decoder for *what to do*. Where both are available, prefer the recovery action as the primary trigger and the XID as the explanation you put in the ticket. Where the driver is too old to expose it (check with `nvidia-smi -q | grep -i "Recovery Action"`), the table in §3 is your fallback. Design the classifier so the XID path is the fallback branch, not the only branch — that is what keeps it working as your fleet's driver baseline moves.

### 8. DCGM health watches — the real field set

`DCGM_FI_DEV_XID_ERRORS` is one field. DCGM's **health monitor** is a whole subsystem that watches curated field groups and returns a graded verdict. It is the thing you run before uncordoning, and the thing a fault-management controller subscribes to.

The watch systems are a bitmask (`dcgmHealthSystems_t`, from `dcgm_structs.h`):

| Bit | System | `dcgmi -s` char | Key fields watched | Representative errors raised |
|---|---|---|---|---|
| `0x1` | `PCIE` | `p` | `PCIE_REPLAY_TOTAL`, `PCIE_LINK_GEN`, `PCIE_LINK_WIDTH` | `DCGM_FR_PCI_REPLAY_RATE` (3) — WARN |
| `0x2` | `NVLINK` | `n` | CRC FLIT/data, replay, recovery, symbol-BER, integrity, ECC totals; `FABRIC_HEALTH_MASK`, `FABRIC_MANAGER_STATUS`, IMEX domain/daemon status | `DCGM_FR_NVLINK_*`, `DCGM_FR_IMEX_UNHEALTHY` (122) |
| `0x4` | `PMU` | — | power-management unit | — |
| `0x8` | `MCU` | — | micro-controller | — |
| `0x10` | `MEM` | `m` | `ECC_DBE_VOL_TOTAL`, `PAGE_RETIRED_{SBE,DBE,PENDING}`, **`XID_ERROR`**, `ROW_REMAP_FAILED`, `ROW_REMAP_UNCORRECTABLE_TOTAL`, `MEMORY_UNREPAIRABLE` | `DCGM_FR_VOLATILE_DBE_DETECTED` (4) FAIL · `DCGM_FR_PENDING_PAGE_RETIREMENTS` (6) WARN · `DCGM_FR_RETIRED_PAGES_LIMIT` (7) FAIL · `DCGM_FR_ROW_REMAP_FAILURE` (80) FAIL · `DCGM_FR_UNCORRECTABLE_ROW_REMAP_LIMIT` (132) WARN · `DCGM_FR_FAULTY_MEMORY` (52) FAIL |
| `0x20` | `SM` | — | streaming multiprocessor | — |
| `0x40` | `INFOROM` | `i` | `INFOROM_VALID`, `XID_ERROR` | `DCGM_FR_CORRUPT_INFOROM` (9) — WARN |
| `0x80` | `THERMAL` | `t` | `THERMAL_VIOLATION`, CPU temp/warn/critical | `DCGM_FR_CLOCKS_EVENT_THERMAL` (10) — WARN |
| `0x100` | `POWER` | `t` | `POWER_VIOLATION`, `BOARD_POWER_WATTS`, CPU power | `DCGM_FR_POWER_UNREADABLE` (11) — WARN |
| `0x200` | `DRIVER` | `d` | **`GPU_RECOVERY_ACTION`** | `DCGM_FR_GPU_RECOVERY_{RESET,REBOOT,DRAIN_P2P,DRAIN_RESET}` (125–128) |
| `0x400` | `NVSWITCH_NONFATAL` | — | `SXID_NON_FATAL_ERROR` | `DCGM_FR_NVSWITCH_NON_FATAL_ERROR` (16) |
| `0x800` | `NVSWITCH_FATAL` | — | `SXID_FATAL_ERROR` | `DCGM_FR_NVSWITCH_FATAL_ERROR` (15) |
| `0x1000` | `CONNECTX` | `x` | ConnectX health / uncorrectable error status+severity | — |
| `0xFFFFFFFF` | `ALL` | `a` | everything above | — |

Two mechanism details worth having:

**The PCIe replay threshold is computed, not fixed.** DCGM derives an expected replay rate from the link generation and lane count — per-lane allowances of roughly **0.15 errors/min at Gen1 (2.5 GT/s), 0.3 at Gen2, 0.48 at Gen3, 0.96 at Gen4, 1.92 at Gen5, 3.84 at Gen6**, multiplied by the negotiated width. So on a Gen5 x16 link the health check tolerates about 1.92 × 16 ≈ **31 replays/min** before raising `DCGM_FR_PCI_REPLAY_RATE`. That is why "we see PCIe replays" is not by itself an incident, and why a link that has silently trained down to x8 has *halved* its own alarm threshold — one more reason the watch subscribes to `PCIE_LINK_WIDTH` alongside the replay counter.

**Some watches need warm-up.** The `dcgmi health -s` help is explicit that memory, NVLink, PCIe, thermal/power and ConnectX watches require roughly **60 seconds** before the first meaningful query, because they are rate/delta computations over samples that do not exist yet. The health API is also *stateful*: the first `CheckWatches()` call after enabling establishes a baseline and returns no errors; subsequent calls report what changed since. **An automation that enables watches and immediately checks will always get a clean PASS.** That is a genuinely nasty bug to have in a post-reset verification step, because it looks like success.

Results come back on a three-value scale (`dcgmHealthWatchResults_t`):

| Value | Symbol | `dcgmi` renders it as | What it means for a controller |
|---|---|---|---|
| 0 | `DCGM_HEALTH_RESULT_PASS` | `Healthy` | Safe to schedule. |
| 10 | `DCGM_HEALTH_RESULT_WARN` | `Warning` | Degraded or indeterminate. Do **not** treat as pass. Investigate; for `DRAIN_P2P`, quiesce and re-check. |
| 20 | `DCGM_HEALTH_RESULT_FAIL` | `Failure` | Do not schedule. Cordon path. |

The non-contiguous values (0/10/20) are deliberate — they leave room for intermediate severities and they make "greater than PASS" a safe comparison in policy code.

Using it:

```console
# 1. Enable the watches you care about on the default group (all GPUs).
#    a = all;  or compose: m (memory) + p (PCIe) + n (NVLink) + t (thermal/power)
#              + i (InfoROM) + d (driver) + x (ConnectX)
$ dcgmi health -g 0 -s a
Health monitor systems set successfully.

# 2. Confirm what is being watched.
$ dcgmi health -g 0 -f
Health monitor systems: PCIe, NVLINK, Memory, InfoROM, Thermal, Power, Driver

# 3. …wait at least 60s for the rate-based watches to have samples…

# 4. Check. The FIRST check after enabling establishes a baseline.
$ dcgmi health -g 0 -c
Health Monitor Report
+----------------------------------------------------------------------------+
| Group 0            | Overall Health: Healthy                               |
+====================+=======================================================+

# 5. After a fault, the same command grades it.
$ dcgmi health -g 0 -c
Health Monitor Report
+----------------------------------------------------------------------------+
| Group 0            | Overall Health: Failure                               |
+====================+=======================================================+
| GPU 3              | Memory:  Failure                                      |
|                    |   GPU 3 had memory errors and row remappings are      |
|                    |   pending                                             |
|                    | Driver:  Failure                                      |
|                    |   GPU 3 requires a reset to recover from a fault.     |
|                    |   Recovery action: 1 (GPU_RESET).                     |
+----------------------------------------------------------------------------+
```

*(Transcript is representative of the documented output structure and the exact `dcgm_errors.h` message strings — `DCGM_FR_PENDING_ROW_REMAP_MSG` is "GPU %u had memory errors and row remappings are pending" and `DCGM_FR_GPU_RECOVERY_RESET_MSG` is "GPU %u requires a reset to recover from a fault. Recovery action: %ld (GPU_RESET)." — rather than a byte-for-byte capture. Use `-j` for machine-readable output; never parse the table.)*

The `-j` flag is the one your automation uses:

```console
$ dcgmi health -g 0 -c -j
```

…which emits the same content as JSON with the numeric result levels intact, so your controller compares against 0/10/20 rather than string-matching `Healthy`.

### 9. Active health: `dcgmi diag`

Health watches are **passive** — they read counters the GPU is already maintaining. They cannot catch a board that is marginal but has not yet faulted. For that you run the **diagnostic**, which puts real load on the device.

Four run levels, from `dcgmi`'s own help text:

| Level | Name | Duration | What it does | When you run it |
|---|---|---|---|---|
| `-r 1` | Quick / System Validation | seconds | Deployment sanity: driver/NVML/persistence-mode/env checks, denylisted-driver check, device-count match, PCIe generation/width. | Pre-flight on every node join; cheap enough to run on every pod-level prologue. |
| `-r 2` | Medium / Extended System Validation | ~2 minutes | Adds memory test, PCIe/NVLink bandwidth+latency, and a short stress. | On drain, before uncordon. **This is the default gate.** |
| `-r 3` | Long / System HW Diagnostics | ~15 minutes | Adds targeted stress, targeted power, memory bandwidth, and longer diagnostics. | After a reset that followed an ECC event; on a board you suspect but cannot prove. |
| `-r 4` | Extended / Longer-running HW Diagnostics | much longer | The full battery, including extended EUD-class tests where supported. | Before filing an RMA — you want a reproducible failure in the ticket. |

Per-test results use a five-value enum (`dcgmDiagResult_t`): `PASS` (0), `SKIP` (1), `WARN` (2), `FAIL` (3), `NOT_RUN` (4). **`SKIP` and `NOT_RUN` are not passes.** A diag that skips the memory test because ECC is disabled (`DCGM_FR_ECC_DISABLED`, 55) returns a green-looking summary while having tested nothing that matters — and a controller that treats "no FAIL" as "healthy" will happily uncordon it. Gate on `PASS` explicitly.

```console
$ dcgmi diag -r 2 -i 3
Successfully ran diagnostic for group.
+---------------------------+------------------------------------------------+
| Diagnostic                | Result                                         |
+===========================+================================================+
|-----  Deployment  --------+------------------------------------------------|
| Denylist                  | Pass                                           |
| NVML Library              | Pass                                           |
| CUDA Main Library         | Pass                                           |
| Permissions and OS Blocks | Pass                                           |
| Persistence Mode          | Pass                                           |
| Environment Variables     | Pass                                           |
| Page Retirement/Row Remap | Fail                                           |
|                           | Error: GPU 3 had memory errors and row         |
|                           | remappings are pending                         |
| Graphics Processes        | Pass                                           |
| Inforom                   | Pass                                           |
+-----  Integration  -------+------------------------------------------------+
| PCIe                      | Pass - All                                     |
+-----  Hardware  ----------+------------------------------------------------+
| GPU Memory                | Pass - All                                     |
+-----  Stress  ------------+------------------------------------------------+
```

*(Representative of the documented `dcgmi diag` output structure and real `dcgm_errors.h` message strings; the exact test names present depend on your DCGM version and GPU.)*

Two operational cautions. **The diagnostic needs the GPU to itself** — it allocates memory and runs CUDA work, so it runs *after* the drain, not during it, and `DCGM_FR_GRAPHICS_PROCESSES` (25) will warn you if something else still holds a context. And **higher levels are not free**: 15 minutes of `-r 3` on a $3/GPU-hr H100 is $0.75 of compute per GPU, which is nothing on one board and $384 if you reflexively run it fleet-wide on 512 GPUs. Level selection is a cost decision, which is the same discipline 05.7 applies to profiling.

### 10. Attributing an XID to a tenant

DCGM alone tells you *which GPU*. To tell a tenant "your kernel did this," you need the pod. Three sources, in decreasing reliability:

1. **`dcgm-exporter` with the Kubernetes device-plugin mapping** (`DCGM_EXPORTER_KUBERNETES=true`) adds `exported_pod` / `exported_namespace` / `exported_container` to every series including `DCGM_FI_DEV_XID_ERRORS`, via the pod-resources join you built in 04. This is the label your alert routes on. **It attributes by allocation, not by fault** — it tells you who *held* the GPU when the XID landed.
2. **The kernel-log line's `pid=` / `name=` fields** for engine-exception XIDs (13/31/43). Map the PID to a container through its cgroup (`/proc/<pid>/cgroup` on the host, or `crictl inspect`). This attributes by *fault*, which is stronger — and it is the only path that works when several containers share a GPU.
3. **Under time-slicing or MPS, source 1 degrades to a guess.** This is the same hole 05.4 established for utilisation: the field is device-level, several pods share the device, and the exported label is whichever mapping the exporter resolved. For an XID that means you can say "one of these three namespaces did it," not "this one did." Say so in the alert annotation rather than naming an innocent team.

Without attribution the log-only path is useless — an anonymous node event nobody owns. Build it before you build the alert.

### 11. Reset scope: what "reset this GPU" actually means

`nvidia-smi -i <n> -r` (equivalently `--gpu-reset`) resets one physical GPU, and only when **every** CUDA context on it is gone. Four complications:

- **Ordering is strict: cordon → drain → reset.** A reset with a live context fails with "GPU is currently in use by another process," or worse, wedges the device into a state only a reboot clears. This is why the controller cordons first (stop new work), drains second (end existing work), and only then resets.
- **MIG.** You generally cannot reset a single MIG instance. You drain every instance and reset the physical GPU — which means the blast radius of one instance's fault is all seven of them. Encode that in the drain plan.
- **NVLink / NVSwitch.** On an HGX 8-GPU board, an XID 74 or a wedged GSP can implicate the fabric rather than one endpoint. A per-GPU reset may not clear it and, on some topologies, resetting one GPU disturbs peers. The recovery-action enum handles this explicitly with `DRAIN_P2P` → re-query. If the driver says `NODE_REBOOT`, do not try to be clever with a device reset.
- **Driver-branch drift.** On some recent branches and on non-data-center parts, `nvidia-smi -r` returns *"Requested functionality has been deprecated"* — the supported recovery path moves toward driver/GSP reload or a node cycle. Verify the behaviour on **your** driver version before writing it into a runbook, and have the node-reboot fallback wired.

### 12. The wiring — XID to node condition to remediation

Now put it together. This is a state machine, and drawing it as one is the difference between a runbook someone follows and one someone argues with at 3am.

```
                             ┌───────────────────┐
                             │   SCHEDULABLE     │◀──────────────────────┐
                             │ (node Ready,      │                       │
                             │  no GPU taint)    │                       │
                             └─────────┬─────────┘                       │
                                       │ XID observed (dmesg or DCGM)    │
                                       ▼                                 │
                             ┌───────────────────┐                       │
                             │    CLASSIFY       │                       │
                             │ recovery-action?  │ ── prefer this        │
                             │ else XID table    │ ── fallback           │
                             └────┬─────────┬────┘                       │
                  APP-CLASS       │         │       HW-CLASS             │
              (13/31/43, single   │         │  (48/63/64/74/79/94/95/    │
               namespace)         │         │   119/120/140, or          │
                                  ▼         │   recovery-action != NONE) │
                    ┌──────────────────┐    │                            │
                    │  ATTRIBUTE+LOG   │    │                            │
                    │  k8s Event on    │    ▼                            │
                    │  the pod;        │  ┌──────────────────┐           │
                    │  notify tenant;  │  │    CORDON        │           │
                    │  NO taint        │  │ NodeCondition=   │           │
                    └────────┬─────────┘  │ GpuUnhealthy;    │           │
                             │            │ kubectl cordon   │           │
              same XID from  │            └────────┬─────────┘           │
              ≥3 namespaces  │                     ▼                     │
              on same UUID   │            ┌──────────────────┐           │
              in 1h ─────────┘───────────▶│     DRAIN        │           │
                                          │ evict, respect   │           │
                                          │ PDBs, allow      │           │
                                          │ checkpoint       │           │
                                          └────────┬─────────┘           │
                                                   ▼                     │
                                    ┌──────────────────────────┐         │
                                    │ RECOVERY ACTION == ?     │         │
                                    └───┬──────┬────────┬──────┘         │
                        DRAIN_P2P       │      │        │  NODE_REBOOT   │
                        (WARN)          │      │        │                │
                 quiesce P2P, disable   │      │        ▼                │
                 UVM persistence, ──────┘      │   ┌──────────┐          │
                 RE-QUERY (loop)               │   │  REBOOT  │          │
                                               │   │   NODE   │          │
                        GPU_RESET /            │   └────┬─────┘          │
                        DRAIN_AND_RESET        │        │                │
                                               ▼        │                │
                                       ┌──────────────┐ │                │
                                       │ RESET DEVICE │ │                │
                                       │ nvidia-smi   │ │                │
                                       │  -i N -r     │ │                │
                                       └──────┬───────┘ │                │
                                              └────┬────┘                │
                                                   ▼                     │
                                    ┌──────────────────────────┐         │
                                    │        VERIFY            │         │
                                    │ ROW_REMAP_PENDING == 0   │         │
                                    │ ROW_REMAP_FAILED  == 0   │         │
                                    │ recovery action == NONE  │         │
                                    │ dcgmi diag -r 2 == PASS  │         │
                                    │ soak: no new XID for 30m │         │
                                    └────┬────────────────┬────┘         │
                                    all  │                │ any check    │
                                   clean │                │ fails, OR    │
                                         │                │ XID 64, OR   │
                                         │                │ remap budget │
                                         │                │ exhausted    │
                                         │                ▼              │
                                         │        ┌───────────────┐      │
                                         │        │  QUARANTINE   │      │
                                         │        │  + RMA ticket │      │
                                         │        │  stays        │      │
                                         │        │  cordoned     │      │
                                         │        └───────────────┘      │
                                         └───────────────────────────────┘
                                                    UNCORDON
```

**The recoverable branch is the loop back to SCHEDULABLE through VERIFY. The unrecoverable branch is the terminal QUARANTINE state.** Everything hard about this design lives in VERIFY: it is the only place that distinguishes them, and every check in it exists because some earlier version of this pipeline uncordoned a board that was not fixed. `ROW_REMAP_PENDING != 0` means the repair has not been applied. A stale `DCGM_FI_DEV_XID_ERRORS` (issue #500) means you cannot use XID absence as a check. `SKIP` on the diag's memory test means it did not test memory. The soak window exists because faults that recur do so on a timescale of minutes, not seconds.

Two production reference architectures implement exactly this shape:

- **NVIDIA NVSentinel** — open-source, Kubernetes-native GPU fault management. Its pipeline is explicit and maps 1:1 onto the diagram: **health monitors** (a GPU health monitor that runs DCGM health checks in one of three DCGM modes — `operator-service`, `external-hostengine`, or `embedded-mode` — plus a separate **syslog** health monitor for the kernel-log path, a NIC health monitor, and cloud-provider maintenance-event monitors) feed a datastore; **Fault Quarantine** cordons nodes according to CEL policy rules; **Node Drainer** evicts with configurable per-namespace strategies; **Fault Remediation** files maintenance CRDs to external break-fix systems. It ships an embedded XID error catalogue that maps codes to recommended actions, a `suppressedErrorCodes` list so you can silence specific `DCGM_FR_*` codes (its own example is `DCGM_FR_CLOCK_THROTTLE_POWER`) without disabling a whole watch, and a `connectivityFailureEscalationThreshold` for the case where DCGM itself goes unresponsive — a failure mode most home-grown pipelines forget entirely. It is validated Volta through Blackwell.
- **AKS GPU health monitoring** — Azure runs **Node Problem Detector** with a dedicated `GPUXIDErrors` check that watches kernel logs for XID errors and sets a node condition. Same NPD machinery you already run for kernel deadlock and OOM, extended with a GPU-XID rule file: the "we bolted this onto tooling you already have" answer, and evidence for why the kernel-log path (issue #235's lossy metric) deserves first-class treatment.

### 13. The cost lens

Every cordon starts a dollar clock, and the classifier is what keeps both error directions bounded. Track **GPU-hours lost to cordon, split by XID class** — it is a two-line panel that answers two different questions:

- Hours lost to the *hardware* classes are the cost of reliability. Compare against the corruption risk you avoided.
- Hours lost to the *app* classes should be **zero**. Anything above zero is a rule bug, and its dollar value is the argument for fixing the rule.

Worked, on a 500-GPU H100 fleet at $2.50/GPU-hr (a dated 2026 rate-card snapshot — verify yours):

```
  Fault rate assumption: 1 unplanned hardware event per 3 GPU-days
    (order-of-magnitude consistent with Meta's Llama 3 snapshot:
     419 unexpected interruptions / 54 days / 16,384 GPUs, mostly hardware)

  500 GPUs × 24h = 12,000 GPU-hours/day allocated
                 = 500 × $2.50 × 24 = $30,000/day

  Events/day        = 500 GPUs ÷ 3 GPU-days per event ≈ 167 events/day  ← fleet-wide
    of which HW-class (≈59%, per Meta's mix)          ≈  98 events/day
    of which APP-class (the rest)                     ≈  69 events/day

  MANUAL triage, HW-class:  1 GPU cordoned × 4h mean-time-to-remediate
       98 × 4  = 392 GPU-hours/day × $2.50 = $980/day  = $358k/yr

  AUTOMATED, HW-class:      1 GPU cordoned × 25 min (drain 10 + reset 2
                            + diag -r 2 ≈ 2 + soak 10 + slack)
       98 × 0.42 = 41 GPU-hours/day × $2.50 = $103/day = $37.5k/yr

  ── automation saves ≈ $320k/yr on remediation latency alone ──

  MIS-CLASSIFYING app XIDs as hardware (cordon on 13/31/43):
       69 × 4  = 276 GPU-hours/day × $2.50 = $690/day  = $252k/yr
       …of PURE waste, on hardware that was never broken.
```

The second number is the one to quote in an interview. **A wrong classifier costs about as much as no automation at all** — which is why the log-only bucket must be narrow, specific, and attributed, and why "when in doubt, cordon" is not the safe default people assume it is.

The asymmetry runs the other way for real faults, and you should be able to state both sides: scheduling a 64-GPU, three-day training run onto a board with an un-reset pending remap risks the whole run. At $2.50/GPU-hr that is 64 × 72 × $2.50 = **$11,520** of compute, plus the wall-clock. One un-reset board can cost more than a week of over-cautious cordoning. The classifier is not "reliability hygiene versus cost" — it is the control that bounds *both*.

## Perspectives

**Operator.** The core skill is triage under uncertainty at 3am: is this a $30k board dying, or a grad student's buggy kernel? The classification table exists so that decision does not depend on who is on call. The second-order skill is knowing the *verification* step is where correctness lives — anyone can cordon; the hard part is knowing when it is safe to stop.

**Developer / tenant.** From inside the container an XID looks like `CUDA error: unspecified launch failure` or `an illegal memory access was encountered` and nothing else. A tenant cannot tell "I wrote past the end of an array" from "the GPU under me has bad HBM." That asymmetry is why the platform owes them the attributed event: the Kubernetes event that says *"XID 31, MMU fault at 0x7f2c4a000000 from PID 31245 — this is an out-of-bounds access in your kernel, run under `compute-sanitizer`"* saves a tenant a day and saves you a ticket.

**Hardware.** Row remapping is a genuinely different recovery model from CPU DIMM ECC. Eight spare rows per bank, 512 uncorrectable remaps per GPU, and — critically — **the remap only takes effect at the next reset**. A CPU silently retires a page and moves on; a GPU records the intent and keeps running on the bad row until you cycle it. That is a hardware property with no CPU analogue, and it is the single fact that makes "cordon, drain, *reset*, verify" a four-step process instead of two.

**Automation / platform.** At fleet scale nobody hand-triages XIDs. The value of this lesson's table is that it is a policy a machine executes continuously, not a runbook a human reads under pressure — and increasingly the machine should be reading the driver's own recovery-action enum rather than the table, with the table as fallback for older drivers. Design the classifier with both branches from day one.

**Economics.** Meta's Llama 3 numbers turn "GPU health matters" into arithmetic: 419 unexpected interruptions in 54 days on one job, the majority hardware, and still >90% effective training time. The cost model above turns it into a budget line: roughly $320k/yr of remediation-latency savings on a 500-GPU fleet, and roughly $252k/yr of avoidable waste if the classifier over-cordons. Both numbers come from the same table.

## Real-world use cases

- **Meta — "The Llama 3 Herd of Models" (arXiv 2407.21783), §3.3.2** — https://arxiv.org/abs/2407.21783. 16,384 H100 GPUs, 54-day snapshot: 466 total interruptions, 419 of them unexpected — roughly one unplanned failure every three hours — with GPU-related causes (including NVLink and HBM3 failures) about 58.7% of them, and >90% effective training time held throughout. **What it shows:** the cordon/RMA taxonomy this lesson teaches is the operational reality behind the highest-profile open LLM training run of its year. Hardware failure at that scale is routine, and automated triage is why the run still finished with high uptime.

- **`NVIDIA/DCGM` issue #235 — "Some XID errors (e.g. XID 62) are not exported via `DCGM_FI_DEV_XID_ERRORS`"** — https://github.com/NVIDIA/DCGM/issues/235. On dcgm-exporter 3.3.7-3.5.0 with driver 550.127.08, XID 62 appeared in `dmesg` but never in the metric; the exporter surfaced only the trailing XID 45. **What it shows:** the telemetry path is a *lossy mirror* of the kernel log. If your only XID input is Prometheus, you have blind spots that look exactly like healthy GPUs — which is the concrete justification for running an NPD kernel-monitor or NVSentinel syslog monitor *alongside* DCGM rather than instead of it.

- **`NVIDIA/dcgm-exporter` issue #500 — "`xid_errors` metric does not reset after error recovery unless dcgm-exporter is restarted"** — https://github.com/NVIDIA/dcgm-exporter/issues/500. dcgm-exporter 3.3.8–3.6.0, DCGM 3.3.8, driver 550.127.08, H100 80GB HBM3: the reporter verified via `dcgmi` and `nvidia-smi` that the GPU had recovered, while the Prometheus endpoint kept serving the old XID. **What it shows:** you cannot use absence-of-XID as your uncordon criterion. Recovery must be verified against state fields (`ROW_REMAP_PENDING`, `ROW_REMAP_FAILED`, recovery action) and an active `dcgmi diag`, which is exactly why VERIFY in §12 has four independent checks instead of one.

- **NVIDIA NVSentinel** — https://github.com/NVIDIA/NVSentinel. Open-source Kubernetes-native fault remediation: GPU/syslog/NIC/cloud-maintenance health monitors → datastore → CEL-rule Fault Quarantine (cordon) → Node Drainer (per-namespace eviction strategies) → Fault Remediation (maintenance CRDs to break-fix systems). Ships an embedded XID catalogue with recommended actions, three DCGM connection modes, `suppressedErrorCodes` for silencing individual `DCGM_FR_*` codes, and an escalation threshold for DCGM itself becoming unresponsive. Validated Volta through Blackwell. **What it shows:** the reference architecture for §12's state machine, including the failure modes (a suppression list, and "what if the health system itself dies") that home-grown pipelines forget.

- **Microsoft/Azure — GPU health monitoring in AKS Node Problem Detector** — https://learn.microsoft.com/en-us/azure/aks/gpu-health-monitoring. The `GPUXIDErrors` NPD check watches kernel logs for XID errors and sets node conditions automatically. **What it shows:** a second, independent production architecture for the same wiring, built on the kernel-log path rather than DCGM — corroborating issue #235's finding that the log is the more complete source.

- **Imbue — "From bare metal to a 70B model: infrastructure set-up and scripts"** — https://imbue.com/research/70b-infrastructure/. Their node health-check suite scans `dmesg` for hardware Xid/SXid errors as a pass/fail gate before a node is accepted into the cluster. **What it shows:** XID/SXid log scanning is not hyperscaler-only tooling — a small team building bare-metal H100s from scratch treated it as table stakes, and put it at *admission* time rather than only at fault time.

## Worked example

**Scenario.** `gpu-node-17`, GPU 3 (H100 80GB HBM3, driver 570.x, DCGM 4.x, dcgm-exporter with `DCGM_EXPORTER_KUBERNETES=true`). At 02:14:07 the alert fires.

### Step 1 — read both sources, not one

Prometheus:

```
DCGM_FI_DEV_XID_ERRORS{gpu="3",UUID="GPU-a1b2c3d4-...",Hostname="gpu-node-17",
  exported_namespace="team-research",exported_pod="llama-sft-0"} 94
```

Kernel log on the node — always check, because the metric is last-value and lossy:

```
[168429.114550] NVRM: Xid (PCI:0000:65:00): 94, pid='<unknown>', name=<unknown>,
                Contained: CE User Channel (0x9). RST: No, D-RST: No
[168429.118002] NVRM: Xid (PCI:0000:65:00): 63, pid='<unknown>', name=<unknown>,
                Row Remapper: New row marked for remapping, reset gpu to activate.
```

**Two XIDs, four milliseconds apart, and the metric only ever shows one of them.** This is exactly the #235 failure mode in miniature: if you had alerted only on the gauge you would have seen `94` and missed that a remap is now pending. Classification requires both lines.

### Step 2 — classify

- **94 = contained ECC error.** Uncorrectable, but containment held: only `llama-sft-0`'s context died. Co-tenants are trustworthy. (Had this been **95**, every result produced on GPU 3 in the surrounding window would be suspect and you would tell those teams so.)
- **63 = a remap entry was recorded to InfoROM.** Repair is scheduled but **not active**. The board is currently running with a known-bad row still in service.

Cross-check the state fields rather than trusting the XID alone:

```console
$ nvidia-smi -i 3 -q -d ROW_REMAPPER | grep -E "Correctable|Uncorrectable|Pending|Failure"
        Correctable Error                 : 0
        Uncorrectable Error               : 3
        Pending                           : Yes
        Remapping Failure Occurred        : No

$ nvidia-smi -i 3 -q | grep -A1 "Recovery Action"
    GPU Recovery Action               : Reset
```

The driver has independently concluded `GPU_RESET`. DCGM's `DRIVER` watch will be reporting `DCGM_FR_GPU_RECOVERY_RESET` (125) as a **FAIL** for the same reason. **Two independent signals agree: reset required.**

### Step 3 — decide the blast radius before touching anything

- Is GPU 3 MIG-partitioned? `nvidia-smi -i 3 --query-gpu=mig.mode.current --format=csv,noheader` → `Disabled`. Good: the drain is one GPU's worth of pods.
- Is anything doing P2P/NVLink work on it? The recovery action is `Reset`, not `Drain P2P`, so the driver is not blocking on quiesce.
- Which pods hold it? Use the 04 join: `DCGM_FI_PROF_SM_ACTIVE{gpu="3",Hostname="gpu-node-17"}` carries `exported_pod`/`exported_namespace`. One pod: `team-research/llama-sft-0`.

### Step 4 — cordon, drain, reset

```console
$ kubectl cordon gpu-node-17
node/gpu-node-17 cordoned

# Give the training job a chance to checkpoint. If your jobs honour SIGTERM
# with a checkpoint handler, a generous grace period is cheaper than a restart
# from the last periodic checkpoint.
$ kubectl drain gpu-node-17 \
      --ignore-daemonsets --delete-emptydir-data \
      --grace-period=300 --timeout=10m
node/gpu-node-17 already cordoned
evicting pod team-research/llama-sft-0
pod/llama-sft-0 evicted

# Confirm no CUDA contexts remain before resetting.
$ nvidia-smi -i 3 --query-compute-apps=pid,used_memory --format=csv
pid, used_gpu_memory [MiB]

$ nvidia-smi -i 3 -r
GPU 00000000:65:00.0 was successfully reset.
All done.
```

### Step 5 — VERIFY (the step people skip)

```console
$ nvidia-smi -i 3 -q -d ROW_REMAPPER | grep -E "Pending|Failure"
        Pending                           : No
        Remapping Failure Occurred        : No

$ nvidia-smi -i 3 -q | grep -A1 "Recovery Action"
    GPU Recovery Action               : None

$ dcgmi health -g 0 -s a          # re-enable watches (baseline call)
$ sleep 90                        # rate-based watches need ~60s of samples
$ dcgmi health -g 0 -c
Health Monitor Report
| Group 0            | Overall Health: Healthy                               |

$ dcgmi diag -r 2 -i 3
| Page Retirement/Row Remap | Pass                                           |
| PCIe                      | Pass - All                                     |
| GPU Memory                | Pass - All                                     |
```

All four checks clean. Soak for 30 minutes with no new XID, then:

```console
$ kubectl uncordon gpu-node-17
node/gpu-node-17 uncordoned
```

**Cost of this incident:** one GPU out of service ≈ 30 minutes (drain 8 + reset 1 + verify 3 + soak 30 ≈ 42 min wall clock, of which ~0.7 GPU-hours billable) — call it **$1.75 at $2.50/GPU-hr**, versus the multi-thousand-dollar exposure of running a three-day job on an un-reset board.

### Step 6 — the branches you did not take

**If it had been XID 64 / `Remapping Failure Occurred : Yes`.** Stop at VERIFY. The board's self-repair path failed — either a bank already had its eight uncorrectable remaps, or the row was already remapped, or the 512-remap budget is spent. There is no reset that fixes this. Keep it cordoned, mark it `QUARANTINE`, run `dcgmi diag -r 4` to attach a reproducible failure to the RMA, and file the ticket. **Never uncordon on a remap failure.**

**If it had been XID 31 from `team-ml/notebook-7`.** Opposite action entirely, same field. Attribute, emit a Kubernetes event on the pod, notify the tenant, leave the node schedulable:

```console
$ kubectl -n team-ml get events --field-selector involvedObject.name=notebook-7
LAST SEEN   TYPE      REASON        OBJECT              MESSAGE
12s         Warning   GpuAppFault   pod/notebook-7      XID 31 (MMU fault) on GPU 3,
                                                        PID 31245 (python3), fault @
                                                        0x7f2c4a000000. Illegal memory
                                                        access in your CUDA kernel —
                                                        run under compute-sanitizer.
```

Escalate only if the *blast pattern* flips: three or more distinct namespaces raising app-class XIDs on the same UUID within an hour means the board, not the code.

### Step 7 — the capacity signal, which is not an incident at all

Separately from any of the above, your weekly report ranks boards by remap headroom:

```promql
# Weekly fleet-hygiene report: boards closest to the 512-remap ceiling.
topk(20,
  DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS
)

# Rate of accumulation — how fast is each board burning its budget?
topk(20,
  increase(DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS[30d])
)
```

A board at 3 remapped rows with none added in 30 days is fine. A board at 41 rows that added 18 in the last 30 days will exhaust its budget in roughly `(512 − 41) / (18/30) ≈ 785 days` at the *fleet* limit — but a single bank hitting its **eight-row** limit will fail long before that, and the bank histogram is what tells you which. Queue it for the next planned maintenance window; do not page anyone. That is the whole distinction between §5's two signals, made operational.

## Practice

Build three artifacts, all of which feed ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md). Develop them against the [fake GPU fleet lab](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md) (which supports injectable XIDs) before spending rented-GPU time.

### 1. The counters CSV that actually supports triage

The default `dcgm-exporter` set cannot answer "is this board safe to uncordon." Write a custom counters CSV — remembering it **replaces** the default set, so re-list everything you still want:

```csv
# --- identity & the util story (from 05.1–05.3) ---
DCGM_FI_DRIVER_VERSION,            label,   Driver Version
DCGM_FI_DEV_GPU_UTIL,              gauge,   GPU utilization (in %).
DCGM_FI_PROF_GR_ENGINE_ACTIVE,     gauge,   Ratio of time the graphics engine is active.
DCGM_FI_PROF_SM_ACTIVE,            gauge,   The ratio of cycles an SM has at least 1 warp assigned.
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE,   gauge,   Ratio of cycles the tensor (HMMA) pipe is active.
DCGM_FI_DEV_FB_USED,               gauge,   Framebuffer memory used (in MiB).

# --- health: events ---
DCGM_FI_DEV_XID_ERRORS,            gauge,   Value of the last XID error encountered.
# opt-in expression metrics: a real count you can rate(), labelled by xid
DCGM_EXP_XID_ERRORS_COUNT,         gauge,   XID errors observed in the configured window.

# --- health: repair state (THE uncordon gate) ---
DCGM_FI_DEV_ROW_REMAP_PENDING,     gauge,   Whether remapping of rows is pending.   # NOT in default CSV
DCGM_FI_DEV_ROW_REMAP_FAILURE,     gauge,   Whether remapping of rows has failed.
DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS, counter, Remapped rows for uncorrectable errors.
DCGM_FI_DEV_CORRECTABLE_REMAPPED_ROWS,   counter, Remapped rows for correctable errors.

# --- health: wear trend ---
DCGM_FI_DEV_ECC_SBE_VOL_TOTAL,     counter, Total single-bit volatile ECC errors.
DCGM_FI_DEV_ECC_DBE_VOL_TOTAL,     counter, Total double-bit volatile ECC errors.
DCGM_FI_DEV_ECC_SBE_AGG_TOTAL,     counter, Total single-bit persistent ECC errors.
DCGM_FI_DEV_ECC_DBE_AGG_TOTAL,     counter, Total double-bit persistent ECC errors.
DCGM_FI_DEV_PCIE_REPLAY_COUNTER,   counter, Total number of PCIe retries.
```

**Acceptance:** `curl :9400/metrics | grep ROW_REMAP_PENDING` returns a series. If it does not, the field is not being collected and your uncordon gate is fictional.

### 2. The three-tier alert pack

Three severities, because there are three response times.

```yaml
groups:
- name: gpu-xid
  rules:

  # ── TIER 1: hardware fault → cordon path. Page. ────────────────────────────
  - alert: GpuFatalXid
    # The gauge's VALUE is the XID number, so test membership in the cordon set.
    expr: |
      DCGM_FI_DEV_XID_ERRORS == 48 or DCGM_FI_DEV_XID_ERRORS == 63
      or DCGM_FI_DEV_XID_ERRORS == 64  or DCGM_FI_DEV_XID_ERRORS == 74
      or DCGM_FI_DEV_XID_ERRORS == 79  or DCGM_FI_DEV_XID_ERRORS == 94
      or DCGM_FI_DEV_XID_ERRORS == 95  or DCGM_FI_DEV_XID_ERRORS == 119
      or DCGM_FI_DEV_XID_ERRORS == 120 or DCGM_FI_DEV_XID_ERRORS == 140
    for: 0m
    labels: { severity: critical, action: cordon-drain }
    annotations:
      summary: "Fatal XID {{ $value }} on {{ $labels.Hostname }} GPU {{ $labels.gpu }}"
      runbook: "cordon → drain → check recovery action → reset → VERIFY → uncordon; RMA on 64 or remap failure"

  # The repair-state gate. Fires independently of any XID, and is the one that
  # must be CLEAR before uncordoning. Catches the #500 stale-metric case.
  - alert: GpuRemapPending
    expr: DCGM_FI_DEV_ROW_REMAP_PENDING == 1
    for: 2m
    labels: { severity: critical, action: cordon-reset }
    annotations:
      summary: "Row remap PENDING on {{ $labels.Hostname }} GPU {{ $labels.gpu }} — needs reset"

  - alert: GpuRemapFailure
    expr: DCGM_FI_DEV_ROW_REMAP_FAILURE == 1
    for: 0m
    labels: { severity: critical, action: rma }
    annotations:
      summary: "Row remap FAILED on {{ $labels.Hostname }} GPU {{ $labels.gpu }} — RMA, do not uncordon"

  # ── TIER 2: tenant bug → notify, never cordon. ─────────────────────────────
  - alert: GpuAppXid
    expr: |
      DCGM_FI_DEV_XID_ERRORS == 13 or DCGM_FI_DEV_XID_ERRORS == 31
      or DCGM_FI_DEV_XID_ERRORS == 43
    for: 0m
    labels: { severity: info, action: notify-tenant }
    annotations:
      summary: "App-level XID {{ $value }} in {{ $labels.exported_namespace }}/{{ $labels.exported_pod }}"
      detail: "Likely an illegal memory access in the tenant's CUDA kernel. Run under compute-sanitizer."

  # Blast-pattern escalation: the same app-class XID hitting ≥3 namespaces on
  # ONE GPU inside an hour is a bad board, not three bad tenants.
  - alert: GpuAppXidAcrossTenants
    expr: |
      count by (Hostname, gpu, UUID) (
        count by (Hostname, gpu, UUID, exported_namespace) (
          max_over_time(
            (DCGM_FI_DEV_XID_ERRORS == 13 or DCGM_FI_DEV_XID_ERRORS == 31
             or DCGM_FI_DEV_XID_ERRORS == 43)[1h:1m]
          )
        )
      ) >= 3
    for: 5m
    labels: { severity: warning, action: investigate-hardware }
    annotations:
      summary: "App-class XIDs from ≥3 namespaces on one GPU — suspect the board, not the code"

  # ── TIER 3: wear trend → weekly report, never page. ───────────────────────
  - record: gpu:remap_budget_used_ratio
    # 512 uncorrectable remappings is the documented per-GPU ceiling.
    expr: DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS / 512

  - record: gpu:remap_burn_rate_30d
    expr: increase(DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS[30d])

  - alert: GpuSbeRateRising
    # XID 92 is the driver's own version of this; this catches the ramp before it.
    expr: increase(DCGM_FI_DEV_ECC_SBE_VOL_TOTAL[24h]) > 100
    for: 1h
    labels: { severity: info, action: schedule-maintenance }
```

**Acceptance:** the pack fires critical on 48/63/64/74/79/94/95/119/120/140, fires *info only* on 13/31/43, escalates on the cross-tenant pattern, and never pages on a remap trend.

### 3. The runbook, as a state machine

Write §12's diagram as an ordered procedure your on-call can execute, with the branches explicit:

1. Confirm the XID **in both `dmesg` and DCGM** (they disagree — issue #235).
2. Read `nvidia-smi -q -d ROW_REMAPPER` and `nvidia-smi -q | grep "Recovery Action"`.
3. Determine blast radius: MIG? NVLink partner? Which pods (04 join)?
4. `kubectl cordon` → `kubectl drain` (respect PDBs, generous grace for checkpointing).
5. Branch on recovery action: `DRAIN_P2P` → quiesce and re-query; `NODE_REBOOT` → reboot; `GPU_RESET`/`DRAIN_AND_RESET` → `nvidia-smi -i N -r`.
6. **VERIFY, all four:** `ROW_REMAP_PENDING == 0`, `ROW_REMAP_FAILED == 0`, recovery action `None`, `dcgmi diag -r 2` **PASS** (not SKIP), plus a 30-minute soak.
7. Uncordon **only if all four pass**; otherwise QUARANTINE + `dcgmi diag -r 4` + RMA ticket.

**Acceptance:** the runbook branches explicitly on XID 64 / remap failure → RMA, never uncordons on a pending remap, treats diag `SKIP` as not-a-pass, and states why the XID metric's absence is not evidence of recovery.

## Common pitfalls

1. **Using the absence of `DCGM_FI_DEV_XID_ERRORS` as proof of recovery.** The metric is last-value and, per dcgm-exporter #500, can hold a stale XID indefinitely after the GPU recovers. *Mechanism:* the exporter caches the field's last sample; DCGM has no "the XID cleared" event to publish because there is no such thing. Verify against `ROW_REMAP_PENDING`, `ROW_REMAP_FAILED`, the recovery action, and an active diag.

2. **Trusting DCGM as your only XID source.** Per DCGM #235, some XIDs (62 was the reported case) reach `dmesg` and never reach field 230. *Mechanism:* not every driver-side RC report is plumbed into NVML's last-XID state that DCGM samples. Run a kernel-log monitor in parallel — NPD's `GPUXIDErrors` or NVSentinel's syslog monitor — and treat the log as authoritative when they disagree.

3. **`rate()`-ing the XID gauge.** It is a value, not a count. `rate(DCGM_FI_DEV_XID_ERRORS[5m])` produces a meaningless number that goes *negative* when the XID number decreases (e.g. 94 → 63). Use membership tests plus `changes()`, or enable `DCGM_EXP_XID_ERRORS_COUNT`.

4. **Uncordoning on a pending remap.** `ROW_REMAP_PENDING == 1` means the repair is recorded but the bad row is still in service. *Mechanism:* the remap is programmed out of InfoROM at GPU init, so it only takes effect at reset. Scheduling onto that board means running on memory the GPU already knows is bad.

5. **Treating `dcgmi diag`'s `SKIP` or `NOT_RUN` as a pass.** A diag that skipped the memory test (ECC disabled, `DCGM_FR_ECC_DISABLED`) tested nothing you cared about, but the summary has no `Fail` in it. Gate on explicit `PASS`.

6. **Checking DCGM health immediately after enabling watches.** The health API is stateful: the first `CheckWatches()` establishes a baseline and reports nothing. Your post-reset verification will pass on a broken board. Enable, wait ≥60s, then check — and check twice if you want to be sure.

7. **Confusing capacity trending with event alerting.** A climbing remapped-rows counter is a maintenance-scheduling signal with a horizon of weeks; a fresh 63/64 is an immediate cordon. Paging on the former burns your on-call's trust; batching the latter into a weekly report loses data.

8. **Resetting before the drain completes.** `nvidia-smi -r` with a live CUDA context fails outright or wedges the device into a state only a reboot clears. Cordon → drain → reset is a strict order.

9. **Assuming one GPU's reset is isolated.** Under MIG you reset the whole physical GPU (all instances). On an NVLink fabric a fault may implicate peers, which is precisely what `DRAIN_P2P` exists to tell you. Ask the recovery-action field before assuming scope.

10. **Believing "when in doubt, cordon" is the safe default.** The cost model in §13 puts mis-classifying app XIDs as hardware at roughly the same order of magnitude as having no automation at all. Being wrong in the cautious direction is not free.

## Self-check

- **You see XID 94 on GPU 3, immediately followed by XID 63 on the same GPU. What do you do?**
  **Answer:** Both are cordon-set. 94 is an uncorrectable ECC error that containment confined to the faulting context (only that pod's work is invalid; co-tenants are fine — had it been 95, they would not be). 63 says a row-remap entry was recorded to InfoROM and **needs a GPU reset to take effect**. So: cordon the node, drain GPU 3's pods with a grace period long enough to checkpoint, check `nvidia-smi -q | grep "Recovery Action"` (expect `Reset`), then `nvidia-smi -i 3 -r`. Then verify all four gates — `ROW_REMAP_PENDING == 0`, `ROW_REMAP_FAILED == 0`, recovery action `None`, `dcgmi diag -r 2` returning explicit PASS — and soak 30 minutes before uncordoning. If the remap had *failed* (XID 64, or `Remapping Failure Occurred : Yes`), skip the uncordon entirely and open an RMA: the board is out of spare rows in that bank, has hit the 512-remap ceiling, or its InfoROM write path is broken, and no reset fixes any of those.

- **XID 31 fires. Do you cordon the node, or is it the user's bug?**
  **Answer:** Log-only — almost certainly the user's bug. XID 31 is `ROBUST_CHANNEL_FIFO_ERROR_MMU_ERR_FLT`: the application's kernel touched an address with no valid page-table entry (out-of-bounds write, use-after-free, a freed device pointer). The GPU is healthy; the driver killed the offending channel. Attribute it to the pod — the kernel-log line carries `pid=`/`name=`, and the dcgm-exporter k8s labels carry `exported_pod`/`exported_namespace` — emit a Kubernetes event telling the tenant to run under `compute-sanitizer`, and leave the node schedulable. **The one escalation:** if app-class XIDs appear from three or more *different* namespaces on the same UUID within an hour, the discriminator flips to hardware and you investigate the board.

- **Which XID means "the GPU fell off the bus," and what does that imply about your other telemetry?**
  **Answer:** XID 79, `ROBUST_CHANNEL_GPU_HAS_FALLEN_OFF_THE_BUS`. The device stopped responding on PCIe, so NVML cannot read it at all. The important second-order implication: **every other metric for that GPU stops updating**, and a stale gauge in Prometheus looks exactly like a healthy idle GPU. That is why your dashboards need a staleness/`up` check alongside value thresholds. Action: cordon, drain, node reboot (a device reset generally cannot reach a device that is not on the bus); if it recurs after reseating and trying a different slot — which rules out mechanical and slot-specific causes — RMA.

- **What is the GPU Recovery Action field, what are its values, and why does it change how you build the classifier?**
  **Answer:** `NVML_FI_DEV_GET_GPU_RECOVERY_ACTION` is a driver-provided enum saying what the operator must do: `NONE` (0), `GPU_RESET` (1), `NODE_REBOOT` (2), `DRAIN_P2P` (3), `DRAIN_AND_RESET` (4). XID **154** fires when it changes. DCGM's `DCGM_HEALTH_WATCH_DRIVER` watch consumes it and maps it to health results — FAIL for reset/reboot/drain-and-reset, **WARN** for `DRAIN_P2P` because that one is an intermediate state meaning "quiesce peer-to-peer traffic and ask me again." It changes the design because the XID table decodes *what happened* while the recovery action decodes *what to do*: make the recovery action your primary trigger and the XID table the fallback for drivers too old to expose it. A controller that ignores WARN will stall forever on a GPU sitting in `DRAIN_P2P`.

- **What are the actual limits on row remapping, and how do they turn into an RMA decision?**
  **Answer:** Every DRAIN bank has **8 spare rows**. A remap-failure flag is raised when any of three conditions hits: a ninth uncorrectable-error remap is attempted in a bank that already has eight, a remap is attempted on a row that was already remapped, or the GPU passes **512 total uncorrectable remappings**. When the flag is set you get XID 64 / `Remapping Failure Occurred : Yes` / `DCGM_FR_ROW_REMAP_FAILURE` (a DCGM **FAIL**), and there is no reset that recovers it — that is the RMA trigger. Before that, DCGM raises `DCGM_FR_UNCORRECTABLE_ROW_REMAP_LIMIT` as a **WARN** when the uncorrectable count crosses its internal Ampere+ limit — that is the maintenance-scheduling signal, not a page. The per-bank picture only appears in `nvidia-smi -q -d ROW_REMAPPER`'s availability histogram (Max=8 / High=7 / Partial=2–6 / Low=1 / None=0 spare rows remaining), which is documented as reference output and is **not** exposed via NVML or DCGM — so automation trends the counters and flags, and humans read the histogram during triage.

- **Why is checking DCGM health immediately after enabling watches a bug, and where does it bite you?**
  **Answer:** The health API is stateful. The first `CheckWatches()` after enabling a watch set establishes a baseline of the current counter values and returns no errors by construction; only subsequent calls report deltas. On top of that, the memory, PCIe, NVLink, thermal/power and ConnectX watches are rate-based and need roughly 60 seconds of samples before they mean anything. It bites hardest in **post-reset verification**: you reset a GPU, re-enable watches, immediately check, get `Overall Health: Healthy`, and uncordon a board that was never actually examined. Enable → wait ≥60s → check, and prefer a second check after another interval.

- **What fraction of Meta's Llama 3 interruptions were GPU-related, and what does that imply about staffing?**
  **Answer:** Roughly 58.7% of the 419 unexpected interruptions were GPU-related (including NVLink and HBM3 memory failures), at a rate of about one unplanned failure every three hours across 16,384 GPUs, while still holding >90% effective training time. At that rate no team can hand-triage every event — the only way to hold that uptime is a policy engine that classifies and remediates without waiting on a human. The cost model makes the same point in dollars: on a 500-GPU fleet, automating remediation latency is worth roughly $320k/yr, and a classifier that over-cordons on app-class XIDs gives about $252k/yr of it straight back.

## Connections & what's next

This lesson sits between attribution (05.4) and inference SLOs (05.6): a namespace's `SM_ACTIVE` numbers are only meaningful if the GPU producing them is healthy and at full capacity — a board running with framebuffer offlined (XID 168) or on an un-reset pending remap will corrupt your utilisation story and your latency numbers at the same time. It reuses 05.3's counters-CSV discipline directly (the fields you need for the uncordon gate are not in the default set, and a custom CSV replaces rather than extends) and 05.4's pod-resources join (for attributing an app-class XID to a tenant, with the same time-slicing caveat). It feeds forward into 05.7: if a GPU looks *inefficient* rather than faulty, XID triage plus a `dcgmi diag -r 2` is the cheap first filter that rules out "it's dying" before you spend Nsight-grade effort on "it's slow." Next, 05.6 turns from *is the GPU healthy* to *is the service meeting its latency promise* — same fleet, a different kind of honest metric.

## References & further reading

**Primary sources — read these to check anything above**

1. **NVIDIA — `open-gpu-kernel-modules`, `src/common/sdk/nvidia/inc/nverror.h`** — https://github.com/NVIDIA/open-gpu-kernel-modules/blob/main/src/common/sdk/nvidia/inc/nverror.h — the authoritative definition of every XID number as a `ROBUST_CHANNEL_*` / named constant. This is where the symbols in §1 and §3 come from; if you ever need to check whether an XID number is real, check here first.
2. **NVIDIA — XID Errors reference** — https://docs.nvidia.com/deploy/xid-errors/index.html — the prose catalogue with NVIDIA's own cause columns (HW error / driver error / user app error / system memory corruption / bus error / thermal / FB corruption) and the Xid Catalog analysis section.
3. **NVIDIA — GPU Debug Guidelines** — https://docs.nvidia.com/deploy/gpu-debug-guidelines/index.html — the official triage process flow: reading Xid messages, running DCGM diagnostics, escalating to field diagnostics. §12's state machine follows its shape.
4. **NVIDIA — GPU Memory Error Management (r575)** — https://docs.nvidia.com/deploy/a100-gpu-mem-error-mgmt/index.html — the row-remapping mechanism, the bank availability histogram, and the RMA policy thresholds. **Correction to earlier versions of this lesson:** the budget is **8 spare rows per bank and 512 total uncorrectable remappings per GPU**, not "hundreds of spare rows per bank."
5. **NVIDIA DCGM — `dcgmlib/dcgm_errors.h`** — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_errors.h — every `DCGM_FR_*` code and its message string. The numbers used above (`DCGM_FR_VOLATILE_DBE_DETECTED` 4, `DCGM_FR_ROW_REMAP_FAILURE` 80, `DCGM_FR_UNCONTAINED_ERROR` 81 — whose message literally reads "GPU had an uncontained error (XID 95)", `DCGM_FR_XID_ERROR` 101, `DCGM_FR_SXID_ERROR` 109, `DCGM_FR_GPU_RECOVERY_*` 125–128, `DCGM_FR_UNCORRECTABLE_ROW_REMAP_LIMIT` 132) are from this file.
6. **NVIDIA DCGM — `modules/health/DcgmHealthWatch.cpp`** — https://github.com/NVIDIA/DCGM/blob/master/modules/health/DcgmHealthWatch.cpp — the ground truth for §8: exactly which fields each health system subscribes to, the per-generation PCIe replay-rate thresholds, and which `DCGM_FR_*` each watch can raise. The `DRIVER` watch's mapping of the recovery-action enum to PASS/WARN/FAIL is here.
7. **NVIDIA DCGM — `dcgmlib/dcgm_structs.h`** — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_structs.h — `dcgmHealthSystems_t` (the bitmask in §8), `dcgmHealthWatchResults_t` (PASS 0 / WARN 10 / FAIL 20), and `dcgmDiagResult_t` (PASS 0 / SKIP 1 / WARN 2 / FAIL 3 / NOT_RUN 4).
8. **NVIDIA DCGM — `dcgmi/CommandLineParser.cpp`** — https://github.com/NVIDIA/DCGM/blob/master/dcgmi/CommandLineParser.cpp — the literal `dcgmi health` and `dcgmi diag` help text: the `-s` watch characters (`a` all, `d` driver, `i` InfoROM, `m` memory, `n` NVLink, `p` PCIe, `t` thermal+power, `x` ConnectX), the ~60-second warm-up notes, and the four diag run levels with their documented durations.
9. **NVIDIA — DCGM user guide (feature overview / health monitoring)** — https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html — the narrative version of the health and diagnostics subsystems, including the statefulness of the health check.
10. **NVIDIA — `dcgm-exporter` `etc/default-counters.csv`** — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — what actually ships enabled. Audit it live: `DCGM_FI_DEV_ROW_REMAP_PENDING` is absent and the ECC totals are commented out.
11. **NVIDIA — DCGM Exporter Metrics reference** — https://docs.nvidia.com/datacenter/dcgm/latest/reference/dcgm-exporter-metrics.html — the `DCGM_EXP_XID_ERRORS_COUNT` / `_TOTAL` expression metrics, their window configuration, and the `xid` label.

**Real-world engineering**

12. **Meta — "The Llama 3 Herd of Models" (arXiv 2407.21783), §3.3.2** — https://arxiv.org/abs/2407.21783 — the 16,384-GPU, 419-unexpected-interruption, 58.7%-GPU-related, >90%-effective-training-time numbers that anchor this lesson's stakes and the cost model in §13.
13. **`NVIDIA/DCGM` issue #235** — https://github.com/NVIDIA/DCGM/issues/235 — XID 62 in `dmesg`, absent from `DCGM_FI_DEV_XID_ERRORS`. The evidence for running a kernel-log monitor alongside DCGM.
14. **`NVIDIA/dcgm-exporter` issue #500** — https://github.com/NVIDIA/dcgm-exporter/issues/500 — the XID metric holding a stale value after recovery on H100/DCGM 3.3.8/driver 550.127.08. The evidence for VERIFY not depending on XID absence.
15. **NVIDIA NVSentinel** — https://github.com/NVIDIA/NVSentinel — the open-source reference implementation of §12: GPU/syslog/NIC/CSP health monitors, CEL-based Fault Quarantine, Node Drainer, Fault Remediation, an embedded XID catalogue, `suppressedErrorCodes`, and DCGM-unresponsive escalation. Read `docs/configuration/gpu-health-monitor.md` and `docs/syslog-health-monitor.md` for the two monitor paths.
16. **Imbue — "From bare metal to a 70B model"** — https://imbue.com/research/70b-infrastructure/ — a from-scratch bare-metal H100 build treating `dmesg` Xid/SXid scanning as a node-admission gate, not just a fault-time check.
