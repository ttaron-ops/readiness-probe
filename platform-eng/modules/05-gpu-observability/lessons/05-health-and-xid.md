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
sources: 10
---
{% raw %}
# 05.5 · GPU health and XID errors

> **Concept.** An XID is the GPU's own crash code; your job is to classify each one as "the silicon is dying — cordon and drain" versus "the tenant's kernel is buggy — log and move on," and wire that decision into node conditions automatically.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Where this fits

04 gave you the attribution join — per-namespace `SM_ACTIVE` GPU-hours, honest and named. That answers "who is wasting capacity." This lesson answers a different question the fleet asks every day: "is this GPU even trustworthy to schedule on?" Attribution assumes the hardware underneath is healthy; XID triage is what verifies that assumption, and it's the piece that keeps a *reliability* incident from masquerading as a *utilization* one — a GPU throwing double-bit ECC errors will show garbage `SM_ACTIVE` numbers too, and you need to know which problem you're looking at. What this unlocks: 05.6 needs healthy GPUs to reason about inference SLOs on, and the capstone (05.8) needs a fleet where "idle" means "genuinely idle," not "quietly dying."

## Why this matters

You already run node-level health on 40+ clusters. On CPU boxes a bad DIMM throws an MCE, `mcelog`/`rasdaemon` catches it, and Node Problem Detector taints the node. GPUs have the exact same failure surface — ECC, bus drops, interconnect faults — but the signal arrives as an **XID**: a numeric error the NVIDIA driver logs to `dmesg` and DCGM surfaces as `DCGM_FI_DEV_XID_ERRORS`. Get the classification wrong in either direction and it's expensive:

- **Over-cordon.** A tenant ships a CUDA kernel with an out-of-bounds write. XID 31 fires. You cordon a healthy $30k H100 and page the on-call. Multiply by every ML team on the cluster and you've turned a user bug into a fleet-availability incident.
- **Under-cordon.** A double-bit ECC error (XID 48) corrupts a training step's weights. You log it and keep scheduling. The GPU silently produces garbage — corrupted checkpoints, NaN losses, or an uncontained fault that takes down the co-tenant's job hours later. This is the "your dashboard is green while the GPU is lying" failure the deliverable is named for.

This isn't a rare edge case you're insuring against — it's an operational constant at real fleet scale. Meta's own published account of training Llama 3 gives the numbers: across a 54-day snapshot on **16,384 H100 GPUs**, the job hit **466 total interruptions** — 47 planned, **419 unexpected**. That's roughly **one unplanned failure every three hours**. Hardware issues (GPU faults including NVLink, plus HBM3 memory failures) accounted for the large majority of those — GPU-related problems alone were about **58.7%** of interruption causes. And yet Meta held **>90% effective training time** across the run. That gap — hundreds of hardware faults a month and still >90% uptime — is not luck; it's automated classification and remediation running continuously, which is exactly the skill this lesson builds. The differentiator for the role you're targeting is knowing *which* XIDs are hardware death sentences and which are application noise, and having the remediation wired so a human never has to remember the table at 3am.

## What's new here (calibration)

You know the Prometheus/NPD/taint machinery cold — none of that is new. What is new:

- **XID is a discriminated union, not a severity.** The same field (`DCGM_FI_DEV_XID_ERRORS`) carries "user made a mistake" and "RMA this board" with no built-in severity flag. The number *is* the classification. You must carry the catalogue.
- **Row-remapping is the GPU's spare-tire mechanism** (Ampere/Hopper replaced the older "page retirement"). Some XIDs mean "a remap was recorded, reset to activate," others mean "the spare rows are exhausted, RMA." This lifecycle has no CPU analogue.
- **The remediation is stateful and has two time horizons.** A single XID 63 means "schedule a reset window" — an event. A remap-count counter creeping toward its ceiling is a *trend* — a maintenance-scheduling signal distinct from reacting to any one XID. You're not just tainting on events — you're driving an RMA workflow keyed on both.
- **Reset ≠ reboot.** Many GPU faults clear with a `nvidia-smi --gpu-reset` (or a driver/GSP reload), not a full node reboot — but only after every process holding the GPU is drained. Getting the ordering wrong wedges the device.

## Core concepts

### The signal: `DCGM_FI_DEV_XID_ERRORS`

DCGM exposes the **last XID the GPU raised** as field `DCGM_FI_DEV_XID_ERRORS` (field ID 230). Scraped through `dcgm-exporter` it becomes a gauge whose *value is the XID number*, labelled by `gpu`, `UUID`, `modelName`, `Hostname`, and (with the k8s mapping) pod/namespace:

```
DCGM_FI_DEV_XID_ERRORS{gpu="3",UUID="GPU-a1b2…",modelName="NVIDIA H100 80GB HBM3",Hostname="gpu-node-17"} 48
```

Two consequences that trip people up:

1. **It's last-value, not a counter.** A `48` sitting there means "the most recent XID was 48," not "48 errors." You alert on the *value being in a set*, and you should treat any appearance as an edge (use `changes()` or pair with `dcgm-exporter`'s event stream / `dmesg`). Don't `rate()` it.
2. **The raw truth is in `dmesg`.** The driver logs `NVRM: Xid (PCI:0000:65:00): 48, …`. DCGM is the scrape-friendly mirror; keep the kernel log as the forensic source (channel, pointer, the offending process for engine-exception XIDs).

### The XID catalogue you must carry

The authoritative source is NVIDIA's XID errors guide, cross-referenced against the DCGM `dcgm_errors.h` enum, which gives each fault a named handler beyond just "the number." The load-bearing split for a scheduler:

**CORDON + DRAIN (hardware fault — do not schedule new work):**

| XID | Name | DCGM enum | What it means | Action |
|-----|------|-----------|---------------|--------|
| **48** | Double-Bit ECC (DBE) | `DCGM_FR_VOLATILE_DBE_DETECTED` (4) | Uncorrectable ECC error. Data is already wrong. | Cordon, drain, reset; if it recurs or remap fails → RMA. |
| **63** | ECC row-remap **recorded / pending** | `DCGM_FR_PENDING_ROW_REMAP` (85) | A row was marked for remapping; **needs a GPU reset** to activate. | Cordon, drain, schedule reset window. |
| **64** | ECC row-remap **failure** | `DCGM_FR_ROW_REMAP_FAILURE` (80) | The remap itself failed — spare rows exhausted or remapper broken. | Cordon, drain, **RMA the board**. |
| **74** | NVLink error | `DCGM_FR_NVLINK_ERROR_THRESHOLD` (13) | Fault on an NVLink connection (GPU↔GPU or GPU↔NVSwitch). Corrupts collective ops. | Cordon, drain; reseat/reset; recurring → RMA. |
| **79** | GPU has fallen off the bus | `DCGM_FR_XID_ERROR` (101, generic handler) | Device dropped off PCIe — fatal, GPU unreachable. Often power/thermal/HW. | Cordon, drain, node reset; recurring → RMA. |
| **94** | Contained ECC error | `DCGM_FR_XID_ERROR` | Uncorrectable but **contained** to the faulting app; row will be remapped. | Cordon (drain gracefully), reset to apply remap; watch remap counters. |
| **95** | Uncontained ECC error | `DCGM_FR_UNCONTAINED_ERROR` (81) | Uncorrectable and **not** contained — may have corrupted other contexts. | Cordon, drain **hard**, reset; treat co-tenant results as suspect. |
| **119** | GSP RPC timeout (Hopper+) | `DCGM_FR_XID_ERROR` | GPU System Processor stopped responding to an RPC. GPU effectively wedged. | Cordon, drain, reset; recurring → RMA/driver bug. |
| **120** | GSP error / RPC failure (Hopper+) | `DCGM_FR_XID_ERROR` | GSP firmware fault. | Cordon, drain, reset; collect logs for NVIDIA. |

On an NVSwitch fabric (HGX 8-GPU boards), switch-side faults arrive as a parallel **SXID** and map to `DCGM_FR_SXID_ERROR` (109) rather than a per-GPU XID — same triage discipline, different bus.

**LOG ONLY (almost always the *user's* bug — GPU is healthy):**

| XID | Name | What it means | Action |
|-----|------|---------------|--------|
| **13** | Graphics Engine Exception | Out-of-bounds / illegal instruction inside a kernel — bad app code. | Log, attribute to pod, notify tenant. **Do not cordon.** |
| **31** | GPU memory page fault (illegal access) | App read/wrote memory it didn't own — classic CUDA OOB / use-after-free. | Log, attribute to pod, notify tenant. **Do not cordon.** |
| **43** | GPU stopped processing | The app's work was aborted due to an error in that app. Non-fatal to the GPU. | Log, attribute to pod. **Do not cordon.** |
| **45** | Preemptive cleanup / channel teardown | The driver cleaned up after a killed process (Ctrl-C, OOM-kill, previous XID). Usually a *symptom*, not a cause. | Log; if it trails a hardware XID, act on the hardware XID. |

Caveat you should say out loud in an interview: XID 13/31/43 are *usually* application bugs, but a flood of them across *different* tenants on the *same* GPU flips the interpretation to suspect hardware — the discriminator is "one pod repeatedly" (user bug) vs "every pod that lands here" (bad board). Classification is the number *plus* the blast pattern. Meta's data corroborates this from the other direction: the overwhelming majority of their 419 unexpected interruptions were genuine hardware (GPU/HBM3/NVLink), which is exactly why the log-only bucket has to be narrow and specific — over-cordoning on 13/31/43 stacks false-hardware noise on top of a fleet that already has real hardware faults to chase.

### Row-remapping lifecycle (Ampere/Hopper)

Modern data-center GPUs don't retire pages; each DRAM bank keeps **spare rows**. On an uncorrectable error (or a threshold of correctable ones) the GPU marks the bad row for remapping to a spare — replacing the older pre-Ampere scheme of permanently retiring whole pages (capped at 64 retirements, which slowly shrank usable memory). NVIDIA's GPU Memory Error Management documentation describes A100/H100 as supporting a much larger remap budget (on the order of hundreds of spare rows per bank) before a remap attempt fails outright — a materially bigger safety margin than page retirement ever offered, but still finite. State you can read via `nvidia-smi -q -d ROW_REMAPPER` or the DCGM fields:

- `DCGM_FI_DEV_ROW_REMAP_PENDING` — a remap is recorded but **needs a reset to take effect** (this is the XID 63 world). Node is degraded until reset.
- `DCGM_FI_DEV_ROW_REMAP_FAILURE` — remapping failed (XID 64 world). **RMA signal**, non-recoverable by reset.
- `DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS` / `..._CORRECTABLE_REMAPPED_ROWS` — running counts. A climbing uncorrectable count is a board wearing out; enough of them and you're out of spares → future 64.

Mental model: **63 = spare tire is on but you haven't torqued the lugs (reset). 64 = you're out of spares. 94/95 = the tire blew (48-class ECC) and 94 kept it contained to one lane, 95 didn't.**

Two distinct signals live here, and conflating them is a real operational mistake: **event-based alerting** (a 63 or 64 fires, act now) versus **capacity trending** (the remapped-rows counter is climbing toward its ceiling, schedule maintenance before it becomes a 64). The first is an incident; the second is a fleet-hygiene report you run weekly, ranking boards by remap headroom the way you'd rank disks by SMART reallocated-sector counts.

### SBE vs DBE, and the codes that are *warnings* not faults

ECC comes in two flavours. **Single-bit errors (SBE)** are *corrected* in flight — no data loss — but a rising SBE rate is a leading indicator of a failing die. **XID 92** ("high single-bit ECC error rate") is that warning: don't cordon on a single 92, but *do* trend `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL` and pre-emptively schedule the board for maintenance before it graduates to a 48/94/95 double-bit (DBE) event. Treat 92 like SMART pre-fail on a disk: not an outage, a countdown.

A few adjacent codes worth recognising so you don't misclassify them: **62** (internal micro-controller halt), **68** (video processor exception), **69** (graphics engine class error) — these are firmware/engine faults that, like 79/119/120, warrant cordon-and-reset and escalation if recurring. When in doubt, the rule is: *ECC-uncorrectable, bus, NVLink, GSP, and remap-failure codes cordon; engine-exception and illegal-access codes that map to a single tenant log.*

### Getting the XID attributed to a pod

DCGM alone tells you *which GPU*; to tell a tenant "your kernel did this," you need the pod. Two sources: `dcgm-exporter` run with the Kubernetes device-plugin mapping adds `pod`/`namespace`/`container` labels to `DCGM_FI_DEV_XID_ERRORS` (the same pod-resources join from 04 you're already reusing); and the driver's `dmesg` line names the offending process/PID for engine-exception XIDs (13/31/43), which you correlate to the container via cgroup. Without this, every user-bug XID looks like an anonymous node event and you can't route the "fix your code" message — so the log-only path is only useful if it's *attributed*.

### Reset scope: MIG and NVLink caveats

`nvidia-smi -i N --gpu-reset` resets one physical GPU and only works once every context is gone. Two gotchas: (1) if the GPU is partitioned with **MIG**, you generally can't reset a single instance — you drain and reset the whole physical GPU (disable MIG or reset at the device level). (2) On **NVLink/NVSwitch** systems (HGX H100 8-GPU boards) an XID 74 or a wedged GSP can implicate the fabric, and a per-GPU reset may not clear it — you drain and reboot the node. Encode "can this be isolated to one GPU, or must the node cycle?" as a branch in your controller.

### Wiring: XID → node condition → cordon + drain

This is the automation the deliverable wants, and it's not a toy pipeline — it's how real fleets stay above 90% availability while eating a hardware fault every few hours. Two production reference architectures:

- **NVIDIA NVSentinel** — an open-source, Kubernetes-native GPU fault-management stack. Its pipeline is explicit: health monitors watch DCGM (thermal, ECC, **XID events**), syslog, and cloud-provider maintenance signals; events flow into a **Fault Quarantine** stage that cordons nodes matching CEL-based policy rules; a **Node Drainer** evicts workloads with configurable per-namespace strategies; and a **Fault Remediation** stage files maintenance CRDs to external break-fix systems (reboot, RMA, terminate). It's validated across Volta/Ampere/Hopper/Ada/Blackwell and integrates GCP/AWS maintenance-event APIs. This is the "reference architecture" answer for exactly the automation this lesson describes.
- **AKS GPU health monitoring** — Azure runs **Node Problem Detector** with a dedicated `GPUXIDErrors` check that watches kernel logs for XID errors and sets a node condition when the driver misprograms the GPU or the command stream is corrupted. Same NPD machinery you already use for kernel-deadlock/OOM, extended with a GPU-XID rule file — the "we bolted this onto tooling you already run" answer.

The generic pipeline (build this mentally, then in code):

```
dmesg / DCGM  ──▶  XID classifier (the table above)
                        │
              ┌─────────┴─────────┐
        cordon-set XID        log-only XID
              │                     │
   set NodeCondition=GpuUnhealthy   emit event + attribute to pod
   (NPD / NVSentinel)               (no scheduling change)
              │
   controller: cordon → drain (respect PDBs, checkpoint) → reset
              │
   ┌──────────┴──────────┐
 remap pending & reset OK   remap FAILURE / recurring
   → uncordon              → keep cordoned, open RMA ticket
```

### The cost lens (FinOps angle)

Every cordon has a dollar clock. A cordoned H100 idles at ~$2–4/GPU-hr (on-demand neocloud, 2026 range) doing nothing; an over-cordon on a healthy board from a mis-classified XID 31 is pure waste, and a *fleet* of them from a bad rule is a budget line item. But the asymmetry runs the other way for real faults: scheduling a training run onto a board with a pending remap (XID 63 un-reset) risks a corrupted multi-day checkpoint worth far more than the idle hours. So the classification table isn't just reliability hygiene — it's the control that keeps both false-cordon waste *and* corruption-loss risk low. Track "GPU-hours lost to cordon" split by XID class; a spike in the log-only classes means a tenant is burning your capacity with a buggy kernel and should be billed/paged, not the platform team.

Complement the passive XID watch with active health checks: `dcgmi diag -r <level>` (or the DCGM diagnostics DCGM_FI health fields) run on drain — before you uncordon — catches marginal boards that haven't yet thrown a fatal XID. It's the "run the treadmill test before clearing the patient" step. This is exactly the discipline described by Imbue's published account of building a 70B-parameter training cluster from bare metal: their node health-check suite explicitly scans `dmesg` for **hardware Xid or SXid errors** as a pass/fail gate before a node is even accepted into the cluster — proof that this isn't hyperscaler-only tooling; a small team building bare-metal H100s from scratch treated it as table stakes.

Key ordering rule: **cordon before drain, drain before reset.** Resetting a GPU with live CUDA contexts either fails (`GPU is in use`) or wedges the device. And uncordon *only after* verifying `ROW_REMAP_PENDING=0` and `ROW_REMAP_FAILURE=0` post-reset — otherwise you schedule onto a still-degraded board.

## Perspectives

**Operator.** The core skill is triage under uncertainty at 3am — is this a $30k board dying, or a grad student's buggy kernel? Getting it wrong in either direction has a real cost: RMA'ing healthy hardware wastes fleet capacity and vendor goodwill; under-cordoning corrupts multi-day runs. The classification table exists so this decision doesn't depend on who's on call.

**Hardware.** Row-remapping is a genuinely different failure-recovery model than CPU DIMM ECC. A100/H100's spare-row mechanism gives a much larger uncorrectable-error budget than the pre-Ampere page-retirement scheme it replaced, and — critically — the remap only *activates at the next GPU reset*. That's a hardware property with no CPU analogue: you can't "just keep running" on a pending remap the way a CPU silently retires a bad page and moves on.

**Automation/platform.** At real fleet scale, no human triages XIDs by hand. NVSentinel's own numbers make the case: production deployments at **1,100+ nodes and roughly 40,000 GPUs** across AWS/GCP/Azure/OCI, processing **tens of millions of health events per month**. The entire value of this lesson's classifier is that it's a policy a machine executes continuously, not a runbook a human reads under pressure.

**Economics.** Meta's published Llama 3 numbers turn "GPU health matters" from a platitude into arithmetic: 419 unexpected interruptions in 54 days on one job, the majority hardware, and still >90% effective training time. That ratio — frequent faults, high uptime — is the entire economic argument for automating this lesson's classification table instead of hand-triaging it.

## Real-world use cases

- **Meta — "The Llama 3 Herd of Models" (arXiv 2407.21783), §3.3.2 Infrastructure, Scaling, and Efficiency** — https://arxiv.org/abs/2407.21783. 16,384 H100 GPUs, 54-day snapshot, 466 total interruptions (419 unexpected), roughly one unplanned hardware-class failure every three hours, GPU-related causes ~58.7% of interruptions, yet >90% effective training time maintained. **What it shows:** the cordon/RMA classification taxonomy this lesson teaches is the operational reality behind the highest-profile open LLM training run of 2024 — hardware failure at this scale is routine, not exceptional, and automated triage is why the run still finished with high uptime.
- **Imbue — "From bare metal to a 70B model: infrastructure set-up and scripts"** — https://imbue.com/research/70b-infrastructure/. Their node health-check suite explicitly scans `dmesg` for hardware Xid or SXid errors as a pre-flight pass/fail gate before a node is accepted into the cluster. **What it shows:** a real, published, from-scratch bare-metal H100 cluster build (not a hyperscaler) treats XID/SXid dmesg-scanning as first-class infrastructure — this discipline is buildable by any team standing up GPUs, not exclusive tooling.
- **NVIDIA/NVSentinel — GitHub repository** — https://github.com/nvidia/nvsentinel. Open-source, Kubernetes-native fault remediation: DCGM/syslog/cloud-maintenance-event monitors feed a Fault Quarantine (CEL-rule cordon) → Node Drainer (per-namespace eviction) → Fault Remediation (maintenance CRDs) pipeline, validated Volta through Blackwell, running in production at 1,100+ nodes / ~40,000 GPUs across four clouds. **What it shows:** the reference architecture this lesson describes, running at real fleet scale with concrete numbers to cite.
- **Microsoft/Azure — GPU health monitoring in Node Problem Detector (AKS)** — https://learn.microsoft.com/en-us/azure/aks/gpu-health-monitoring. The `GPUXIDErrors` NPD check watches kernel logs for driver-misprogramming or command-stream-corruption XIDs and sets node conditions automatically. **What it shows:** a second, independent production reference architecture (AKS/NPD) for the exact XID→condition→cordon wiring this lesson teaches, confirmed with the precise check name.

## Worked example

**Scenario.** `gpu-node-17`, GPU 3 (an H100). At 02:14 `dmesg` shows `Xid … 94`, then at 02:14 `Xid … 63`. Your alert fires on `DCGM_FI_DEV_XID_ERRORS`. This is a smaller version of the same class of event that fired roughly every three hours across Meta's 16,384-GPU Llama 3 cluster — the difference at your scale is you're triaging one, not hundreds a week.

Walk the classification:
1. **94 = contained ECC.** Uncorrectable but the fault was contained to the faulting context; the GPU has recorded a row for remapping. Cordon-set → the node must stop taking new pods on GPU 3.
2. **63 = remap pending.** Confirms a remap was recorded and **needs a reset** to activate. Check `DCGM_FI_DEV_ROW_REMAP_PENDING{gpu="3"} == 1`.
3. Controller sets `NodeCondition GpuUnhealthy`, cordons the node, drains GPU-3 pods respecting PDBs (let the training job checkpoint if it can).
4. Reset just that GPU: `nvidia-smi -i 3 --gpu-reset` (or drain the whole node and reset if the GPU is shared via MIG/NVLink and can't be isolated).
5. Post-reset, verify `ROW_REMAP_PENDING == 0` **and** `ROW_REMAP_FAILURE == 0` and no new XID for a soak window. Clean → uncordon. If instead you'd seen **64** or `ROW_REMAP_FAILURE == 1`, you skip the uncordon entirely and open an RMA — the board is out of spare rows.

Contrast: had those two lines been `Xid … 31` from a single pod, you'd have attributed it to that pod's namespace, emitted a Kubernetes event ("illegal memory access — check your kernel"), and left the node schedulable. Same field, opposite action.

**Second, contrasting mini-example — capacity trending, not event response.** Suppose instead of a fresh 63/64, your weekly remap-headroom report shows a board whose `DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS` has been climbing steadily and is now well into its remap budget with no room to spare. No XID fired today — it's healthy *right now*. But it's a "future 64": trending toward the ceiling is a maintenance-scheduling signal (queue an RMA at the next planned window) distinct from reacting to a single event. Treat it like a disk with a rising SMART-reallocated-sector count: not an outage, a countdown you schedule around instead of react to.

## Practice

Build the three artifacts for the deliverable ("Your GPU dashboard is lying to you"):

**1. The XID alert rule.** Write a Prometheus rule that fires on the cordon-set XIDs and stays silent on 13/31/43 (and treats 45 as informational). Acceptance: firing on 48/63/64/74/79/94/95/119/120, silent on 13/31/43.

```yaml
groups:
- name: gpu-xid
  rules:
  - alert: GpuFatalXid
    # value of the gauge IS the XID number
    expr: |
      (DCGM_FI_DEV_XID_ERRORS == bool 48) + (DCGM_FI_DEV_XID_ERRORS == bool 63)
      + (DCGM_FI_DEV_XID_ERRORS == bool 64) + (DCGM_FI_DEV_XID_ERRORS == bool 74)
      + (DCGM_FI_DEV_XID_ERRORS == bool 79) + (DCGM_FI_DEV_XID_ERRORS == bool 94)
      + (DCGM_FI_DEV_XID_ERRORS == bool 95) + (DCGM_FI_DEV_XID_ERRORS == bool 119)
      + (DCGM_FI_DEV_XID_ERRORS == bool 120) > 0
    for: 0m
    labels: { severity: critical, action: cordon-drain }
    annotations:
      summary: "Fatal XID {{ $value }} on {{ $labels.Hostname }} GPU {{ $labels.gpu }}"
      runbook: "cordon → drain → reset; RMA if 64 or remap failure"
```

Add a **separate low-severity** rule for the user-bug XIDs so tenants get told, without paging the platform team:

```yaml
  - alert: GpuAppXid
    expr: DCGM_FI_DEV_XID_ERRORS == 13 or DCGM_FI_DEV_XID_ERRORS == 31
          or DCGM_FI_DEV_XID_ERRORS == 43
    labels: { severity: info, action: notify-tenant }
    annotations:
      summary: "App-level XID {{ $value }} (likely tenant kernel bug) on {{ $labels.Hostname }}"
```

Add a **capacity-trend** recording rule for the "future 64" signal, separate from event alerting:

```yaml
  - record: gpu:remap_headroom_ratio
    expr: DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS / on() group_left() 512
    # rank boards by this weekly; schedule maintenance before it hits 1.0
```

**2. The classified XID table.** Reproduce the two tables above (cordon-set vs log-only) as a reference card in the deliverable, one action per row, with the DCGM enum name alongside the number.

**3. The cordon runbook.** Write the ordered steps: (a) confirm XID + read `ROW_REMAP_PENDING`/`ROW_REMAP_FAILURE`; (b) `kubectl cordon gpu-node-17`; (c) `kubectl drain … --ignore-daemonsets --delete-emptydir-data` respecting PDBs; (d) reset (`nvidia-smi -i N --gpu-reset`, or node reboot if NVLink/MIG-shared); (e) verify remap pending/failure both 0 and soak; (f) uncordon **only if clean**, else open RMA. Acceptance: the runbook explicitly branches on 64 / remap-failure → RMA, and never uncordons a board with a pending remap.

## Common pitfalls

1. **Treating every XID appearance as equally urgent.** Meta's own data shows the overwhelming majority of *interruptions* were genuine GPU/HBM3/NVLink hardware — but that's precisely why the log-only bucket (13/31/43) has to exist and be narrow: over-cordoning on application-bug XIDs creates false hardware-failure noise layered on top of real faults you're already chasing.
2. **Confusing row-remap capacity trending with event-based alerting.** A rising remapped-rows counter is a maintenance-scheduling signal; a fresh 63/64 is an immediate-cordon signal. Both matter, but they drive different SLAs — don't page on-call for a trend, and don't wait for a weekly report to react to a fresh fault.
3. **Assuming XID/SXid dmesg-scanning is hyperscaler-only tooling.** Imbue's from-scratch cluster build shows it's a practical, buildable first-class health gate for any team standing up bare-metal GPUs — not something only Meta/NVIDIA-scale operators can afford to build.
4. **Resetting before the drain completes.** `nvidia-smi --gpu-reset` with a live CUDA context either fails outright or wedges the device. Cordon → drain → reset is a strict order, not a suggestion.

## Self-check

- You see XID 48 on GPU 3, immediately followed by XID 63 on the same GPU. What do you do? **Answer:** Both are cordon-set. 48 is a double-bit (uncorrectable) ECC error — the data is already corrupt — and 63 says the GPU recorded a row for remapping that needs a reset to activate. Cordon the node, drain GPU-3's pods (let jobs checkpoint), then reset that GPU (`nvidia-smi -i 3 --gpu-reset`) to apply the remap. Verify `ROW_REMAP_PENDING == 0` and `ROW_REMAP_FAILURE == 0` and soak clean before uncordoning. If the remap had failed (or you'd seen XID 64), don't uncordon — open an RMA.
- XID 31 fires. Do you cordon the node, or is it the user's bug? **Answer:** Log-only — it's almost certainly the user's bug. XID 31 is a GPU memory page fault: the application accessed memory it didn't own (out-of-bounds, use-after-free in a CUDA kernel). The GPU is healthy. Attribute it to the offending pod/namespace, emit an event telling the tenant to fix their kernel, and leave the node schedulable. The one exception: if 31 recurs across *different* tenants on the *same* GPU, suspect hardware and investigate.
- Which XID means "the GPU fell off the bus"? **Answer:** XID 79 — "GPU has fallen off the bus." The device dropped off PCIe and is unreachable; it's fatal. Cordon, drain, and node-reset; if it recurs, RMA (often power, thermal, or a seating/HW fault).
- What fraction of Meta's Llama 3 training interruptions were GPU-related, and what does that imply about staffing/automation at fleet scale? **Answer:** Roughly 58.7% of the 419 unexpected interruptions were GPU-related causes (including NVLink and HBM3 memory failures), out of one unplanned failure roughly every three hours across 16,384 GPUs. At that rate, no team could hand-triage every event — the only way to hold >90% effective training time is a policy engine (NVSentinel-class automation) that classifies and remediates without waiting on a human.
- What's the difference between row-remap **event** signals (XID 63/64) and row-remap **capacity** signals (the remapped-rows counters), and which drives an immediate cordon vs a maintenance-scheduling flag? **Answer:** Event signals (a fresh 63 or 64 in `dmesg`/DCGM) are immediate — cordon and act now. Capacity signals (`DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS` trending toward its budget) are a maintenance-scheduling flag — the board is healthy today but heading toward a future 64, so you queue it for the next planned maintenance window rather than paging on-call.

## Connections & what's next

This lesson sits between attribution (04/05.4) and inference SLOs (05.6): a namespace's `SM_ACTIVE` numbers are only meaningful if the GPU producing them is healthy, and a healthy fleet is the precondition for reasoning about latency SLOs at all — a wedged or remap-pending GPU will corrupt both your utilization story and your TTFT/TPOT numbers simultaneously. It also feeds forward into 05.7 (profiling escalation): if a GPU looks inefficient rather than outright faulty, XID triage is the first filter that rules out "it's dying" before you spend Nsight-Compute-grade effort asking "why is it slow." Next up, 05.6 turns from *is the GPU healthy* to *is the service meeting its latency promise* — same fleet, a different kind of honest metric.

## References & further reading

**Primary sources**
- NVIDIA — XID Errors reference — https://docs.nvidia.com/deploy/xid-errors/index.html — the canonical catalogue of every XID code and what it means; start here for any XID not covered above.
- NVIDIA — GPU Debug Guidelines — https://docs.nvidia.com/deploy/gpu-debug-guidelines/contents.html — the triage process flow (understanding Xid messages, running DCGM diagnostics, field diagnostics) that this lesson's runbook is built on.
- NVIDIA — GPU Memory Error Management — https://docs.nvidia.com/deploy/a100-gpu-mem-error-mgmt/index.html — read for the row-remapping mechanism, RMA policy thresholds, and error-recovery/response flags on A100/H100.
- NVIDIA DCGM — `dcgm_errors.h` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_errors.h — the exact enum DCGM uses to classify faults (`DCGM_FR_UNCONTAINED_ERROR`, `DCGM_FR_PENDING_ROW_REMAP`, `DCGM_FR_ROW_REMAP_FAILURE`, `DCGM_FR_SXID_ERROR`, etc.) — ground your automation in these exact names, not just the XID number.

**Real-world engineering blogs**
- Meta — "The Llama 3 Herd of Models" (arXiv 2407.21783), §3.3.2 — https://arxiv.org/abs/2407.21783 — the 16,384-GPU, 419-unexpected-interruption, >90%-effective-training-time numbers that anchor this lesson's stakes.
- Imbue — "From bare metal to a 70B model" — https://imbue.com/research/70b-infrastructure/ — a from-scratch cluster build treating XID/SXid dmesg-scanning as a pre-flight health gate.
- NVIDIA — NVSentinel (GitHub) — https://github.com/nvidia/nvsentinel — the open-source fault-management pipeline running at ~40,000-GPU production scale.
- Microsoft — AKS GPU health monitoring — https://learn.microsoft.com/en-us/azure/aks/gpu-health-monitoring — the `GPUXIDErrors` Node Problem Detector check, a second production reference architecture.

**Deeper dives**
- NVIDIA — GPU Memory Error Management (RMA Policy — Thresholds for Row Remapping) — https://docs.nvidia.com/deploy/a100-gpu-mem-error-mgmt/latest/rma-policy-thresholds-for-row-remapping.html — the precise thresholds that turn a remap-capacity trend into an RMA decision.
- NVIDIA — Error Recovery and Response Flags — https://docs.nvidia.com/deploy/a100-gpu-mem-error-mgmt/latest/error-recovery-and-response-flags.html — how row-remap and other flags interact with driver-level recovery behavior.

{% endraw %}
