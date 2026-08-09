---
lesson: "05.5"
title: "GPU health and XID errors"
module: "05"
concept: "GPU health and XID errors"
status: not-started
est_time: "4h"
artifacts: []
---
# 05.5 · GPU health and XID errors

> **Concept.** An XID is the GPU's own crash code; your job is to classify each one as "the silicon is dying — cordon and drain" versus "the tenant's kernel is buggy — log and move on," and wire that decision into node conditions automatically.
>
> Module: [📊 05 — GPU observability and telemetry](../README.md) · Deliverable: ["Your GPU dashboard is lying to you"](../practice/gpu-dashboard-lie/README.md)

## Why this matters

You already run node-level health on 40+ clusters. On CPU boxes a bad DIMM throws an MCE, `mcelog`/`rasdaemon` catches it, and Node Problem Detector taints the node. GPUs have the exact same failure surface — ECC, bus drops, interconnect faults — but the signal arrives as an **XID**: a numeric error the NVIDIA driver logs to `dmesg` and DCGM surfaces as `DCGM_FI_DEV_XID_ERRORS`. If you treat all XIDs the same you get one of two expensive failures:

- **Over-cordon.** A tenant ships a CUDA kernel with an out-of-bounds write. XID 31 fires. You cordon a healthy $30k H100 and page the on-call. Multiply by every ML team on the cluster and you've turned a user bug into a fleet-availability incident.
- **Under-cordon.** A double-bit ECC error (XID 48) corrupts a training step's weights. You log it and keep scheduling. The GPU silently produces garbage — corrupted checkpoints, NaN losses, or an uncontained fault that takes down the co-tenant's job hours later. This is the "your dashboard is green while the GPU is lying" failure the deliverable is named for.

The differentiator for the role you're targeting is exactly this: knowing *which* XIDs are hardware death sentences and which are application noise, and having the remediation wired so a human never has to remember the table at 3am.

## What's new for you

You know the Prometheus/NPD/taint machinery cold — none of that is new. What is new:

- **XID is a discriminated union, not a severity.** The same field (`DCGM_FI_DEV_XID_ERRORS`) carries "user made a mistake" and "RMA this board" with no built-in severity flag. The number *is* the classification. You must carry the catalogue.
- **Row-remapping is the GPU's spare-tire mechanism** (Ampere/Hopper replaced the older "page retirement"). Some XIDs mean "a remap was recorded, reset to activate," others mean "the spare rows are exhausted, RMA." This lifecycle has no CPU analogue.
- **The remediation is stateful.** A single XID 63 means "schedule a reset window." Repeated 94/95 or any 64 means "pull the board." You're not just tainting — you're driving an RMA workflow keyed on remap counters.
- **Reset ≠ reboot.** Many GPU faults clear with a `nvidia-smi --gpu-reset` (or a driver/GSP reload), not a full node reboot — but only after every process holding the GPU is drained. Getting the ordering wrong wedges the device.

## Core notes

### The signal: `DCGM_FI_DEV_XID_ERRORS`

DCGM exposes the **last XID the GPU raised** as field `DCGM_FI_DEV_XID_ERRORS` (field ID 230). Scraped through `dcgm-exporter` it becomes a gauge whose *value is the XID number*, labelled by `gpu`, `UUID`, `modelName`, `Hostname`, and (with the k8s mapping) pod/namespace:

```
DCGM_FI_DEV_XID_ERRORS{gpu="3",UUID="GPU-a1b2…",modelName="NVIDIA H100 80GB HBM3",Hostname="gpu-node-17"} 48
```

Two consequences that trip people up:

1. **It's last-value, not a counter.** A `48` sitting there means "the most recent XID was 48," not "48 errors." You alert on the *value being in a set*, and you should treat any appearance as an edge (use `changes()` or pair with `dcgm-exporter`'s event stream / `dmesg`). Don't `rate()` it.
2. **The raw truth is in `dmesg`.** The driver logs `NVRM: Xid (PCI:0000:65:00): 48, …`. DCGM is the scrape-friendly mirror; keep the kernel log as the forensic source (channel, pointer, the offending process for engine-exception XIDs).

### The XID catalogue you must carry

The authoritative list is NVIDIA's XID errors guide and the DCGM `dcgm_errors.h` enum. The load-bearing split for a scheduler:

**CORDON + DRAIN (hardware fault — do not schedule new work):**

| XID | Name | What it means | Action |
|-----|------|---------------|--------|
| **48** | Double-Bit ECC (DBE) | Uncorrectable ECC error. Data is already wrong. | Cordon, drain, reset; if it recurs or remap fails → RMA. |
| **63** | ECC row-remap **recorded / pending** | A row was marked for remapping; **needs a GPU reset** to activate. | Cordon, drain, schedule reset window. |
| **64** | ECC row-remap **failure** | The remap itself failed — spare rows exhausted or remapper broken. | Cordon, drain, **RMA the board**. |
| **74** | NVLink error | Fault on an NVLink connection (GPU↔GPU or GPU↔NVSwitch). Corrupts collective ops. | Cordon, drain; reseat/reset; recurring → RMA. |
| **79** | GPU has fallen off the bus | Device dropped off PCIe — fatal, GPU unreachable. Often power/thermal/HW. | Cordon, drain, node reset; recurring → RMA. |
| **94** | Contained ECC error | Uncorrectable but **contained** to the faulting app; row will be remapped. | Cordon (drain gracefully), reset to apply remap; watch remap counters. |
| **95** | Uncontained ECC error | Uncorrectable and **not** contained — may have corrupted other contexts. | Cordon, drain **hard**, reset; treat co-tenant results as suspect. |
| **119** | GSP RPC timeout (Hopper+) | GPU System Processor stopped responding to an RPC. GPU effectively wedged. | Cordon, drain, reset; recurring → RMA/driver bug. |
| **120** | GSP error / RPC failure (Hopper+) | GSP firmware fault. | Cordon, drain, reset; collect logs for NVIDIA. |

**LOG ONLY (almost always the *user's* bug — GPU is healthy):**

| XID | Name | What it means | Action |
|-----|------|---------------|--------|
| **13** | Graphics Engine Exception | Out-of-bounds / illegal instruction inside a kernel — bad app code. | Log, attribute to pod, notify tenant. **Do not cordon.** |
| **31** | GPU memory page fault (illegal access) | App read/wrote memory it didn't own — classic CUDA OOB / use-after-free. | Log, attribute to pod, notify tenant. **Do not cordon.** |
| **43** | GPU stopped processing | The app's work was aborted due to an error in that app. Non-fatal to the GPU. | Log, attribute to pod. **Do not cordon.** |
| **45** | Preemptive cleanup / channel teardown | The driver cleaned up after a killed process (Ctrl-C, OOM-kill, previous XID). Usually a *symptom*, not a cause. | Log; if it trails a hardware XID, act on the hardware XID. |

Caveat you should say out loud in an interview: XID 13/31/43 are *usually* application bugs, but a flood of them across *different* tenants on the *same* GPU flips the interpretation to suspect hardware — the discriminator is "one pod repeatedly" (user bug) vs "every pod that lands here" (bad board). Classification is the number *plus* the blast pattern.

### Row-remapping lifecycle (Ampere/Hopper)

Modern data-center GPUs don't retire pages; each DRAM bank keeps **spare rows**. On an uncorrectable error (or a threshold of correctable ones) the GPU marks the bad row for remapping to a spare. State you can read via `nvidia-smi -q -d ROW_REMAPPER` or the DCGM fields:

- `DCGM_FI_DEV_ROW_REMAP_PENDING` — a remap is recorded but **needs a reset to take effect** (this is the XID 63 world). Node is degraded until reset.
- `DCGM_FI_DEV_ROW_REMAP_FAILURE` — remapping failed (XID 64 world). **RMA signal**, non-recoverable by reset.
- `DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS` / `..._CORRECTABLE_REMAPPED_ROWS` — running counts. A climbing uncorrectable count is a board wearing out; enough of them and you're out of spares → future 64.

Mental model: **63 = spare tire is on but you haven't torqued the lugs (reset). 64 = you're out of spares. 94/95 = the tire blew (48-class ECC) and 94 kept it contained to one lane, 95 didn't.**

### SBE vs DBE, and the codes that are *warnings* not faults

ECC comes in two flavours. **Single-bit errors (SBE)** are *corrected* in flight — no data loss — but a rising SBE rate is a leading indicator of a failing die. **XID 92** ("high single-bit ECC error rate") is that warning: don't cordon on a single 92, but *do* trend `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL` and pre-emptively schedule the board for maintenance before it graduates to a 48/94/95 double-bit (DBE) event. Treat 92 like SMART pre-fail on a disk: not an outage, a countdown.

A few adjacent codes worth recognising so you don't misclassify them: **62** (internal micro-controller halt), **68** (video processor exception), **69** (graphics engine class error) — these are firmware/engine faults that, like 79/119/120, warrant cordon-and-reset and escalation if recurring. When in doubt, the rule is: *ECC-uncorrectable, bus, NVLink, GSP, and remap-failure codes cordon; engine-exception and illegal-access codes that map to a single tenant log.*

### Getting the XID attributed to a pod

DCGM alone tells you *which GPU*; to tell a tenant "your kernel did this," you need the pod. Two sources: `dcgm-exporter` run with the Kubernetes device-plugin mapping adds `pod`/`namespace`/`container` labels to `DCGM_FI_DEV_XID_ERRORS`; and the driver's `dmesg` line names the offending process/PID for engine-exception XIDs (13/31/43), which you correlate to the container via cgroup. Without this, every user-bug XID looks like an anonymous node event and you can't route the "fix your code" message — so the log-only path is only useful if it's *attributed*.

### Reset scope: MIG and NVLink caveats

`nvidia-smi -i N --gpu-reset` resets one physical GPU and only works once every context is gone. Two gotchas: (1) if the GPU is partitioned with **MIG**, you generally can't reset a single instance — you drain and reset the whole physical GPU (disable MIG or reset at the device level). (2) On **NVLink/NVSwitch** systems (HGX H100 8-GPU boards) an XID 74 or a wedged GSP can implicate the fabric, and a per-GPU reset may not clear it — you drain and reboot the node. Encode "can this be isolated to one GPU, or must the node cycle?" as a branch in your controller.

### Wiring: XID → node condition → cordon + drain

This is the automation the deliverable wants. Two production patterns to reference:

- **NVIDIA NVSentinel** — a GPU fault-management stack: DCGM/`dmesg` watchers detect XIDs and remap state, classify them, set a **node condition / label**, and drive cordon-drain-reset-or-RMA with a policy engine (including scheduling reset windows and de-cordoning after a clean reset). This is the "reference architecture" answer.
- **AKS GPU health monitoring** — Azure runs **Node Problem Detector** with a GPU rule set; DCGM/`dmesg`-sourced XIDs become `NodeCondition`s and the node gets tainted/cordoned. Same NPD machinery you already use for kernel-deadlock/OOM, extended with a GPU-XID rule file.

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

Key ordering rule: **cordon before drain, drain before reset.** Resetting a GPU with live CUDA contexts either fails (`GPU is in use`) or wedges the device. And uncordon *only after* verifying `ROW_REMAP_PENDING=0` and `ROW_REMAP_FAILURE=0` post-reset — otherwise you schedule onto a still-degraded board.

## Worked example

**Scenario.** `gpu-node-17`, GPU 3 (an H100). At 02:14 `dmesg` shows `Xid … 94`, then at 02:14 `Xid … 63`. Your alert fires on `DCGM_FI_DEV_XID_ERRORS`.

Walk the classification:
1. **94 = contained ECC.** Uncorrectable but the fault was contained to the faulting context; the GPU has recorded a row for remapping. Cordon-set → the node must stop taking new pods on GPU 3.
2. **63 = remap pending.** Confirms a remap was recorded and **needs a reset** to activate. Check `DCGM_FI_DEV_ROW_REMAP_PENDING{gpu="3"} == 1`.
3. Controller sets `NodeCondition GpuUnhealthy`, cordons the node, drains GPU-3 pods respecting PDBs (let the training job checkpoint if it can).
4. Reset just that GPU: `nvidia-smi -i 3 --gpu-reset` (or drain the whole node and reset if the GPU is shared via MIG/NVLink and can't be isolated).
5. Post-reset, verify `ROW_REMAP_PENDING == 0` **and** `ROW_REMAP_FAILURE == 0` and no new XID for a soak window. Clean → uncordon. If instead you'd seen **64** or `ROW_REMAP_FAILURE == 1`, you skip the uncordon entirely and open an RMA — the board is out of spare rows.

Contrast: had those two lines been `Xid … 31` from a single pod, you'd have attributed it to that pod's namespace, emitted a Kubernetes event ("illegal memory access — check your kernel"), and left the node schedulable. Same field, opposite action.

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
      DCGM_FI_DEV_XID_ERRORS
        == on() group_left() 48 or DCGM_FI_DEV_XID_ERRORS == 63
        or DCGM_FI_DEV_XID_ERRORS == 64  or DCGM_FI_DEV_XID_ERRORS == 74
        or DCGM_FI_DEV_XID_ERRORS == 79  or DCGM_FI_DEV_XID_ERRORS == 94
        or DCGM_FI_DEV_XID_ERRORS == 95  or DCGM_FI_DEV_XID_ERRORS == 119
        or DCGM_FI_DEV_XID_ERRORS == 120
    for: 0m
    labels: { severity: critical, action: cordon-drain }
    annotations:
      summary: "Fatal XID {{ $value }} on {{ $labels.Hostname }} GPU {{ $labels.gpu }}"
      runbook: "cordon → drain → reset; RMA if 64 or remap failure"
```

Cleaner, less brittle than the `or`-chain — match against a label-set with a recording rule, or use a boolean:

```yaml
  - alert: GpuFatalXid
    expr: |
      (DCGM_FI_DEV_XID_ERRORS == bool 48) + (DCGM_FI_DEV_XID_ERRORS == bool 63)
      + (DCGM_FI_DEV_XID_ERRORS == bool 64) + (DCGM_FI_DEV_XID_ERRORS == bool 74)
      + (DCGM_FI_DEV_XID_ERRORS == bool 79) + (DCGM_FI_DEV_XID_ERRORS == bool 94)
      + (DCGM_FI_DEV_XID_ERRORS == bool 95) + (DCGM_FI_DEV_XID_ERRORS == bool 119)
      + (DCGM_FI_DEV_XID_ERRORS == bool 120) > 0
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

**2. The classified XID table.** Reproduce the two tables above (cordon-set vs log-only) as a reference card in the deliverable, one action per row.

**3. The cordon runbook.** Write the ordered steps: (a) confirm XID + read `ROW_REMAP_PENDING`/`ROW_REMAP_FAILURE`; (b) `kubectl cordon gpu-node-17`; (c) `kubectl drain … --ignore-daemonsets --delete-emptydir-data` respecting PDBs; (d) reset (`nvidia-smi -i N --gpu-reset`, or node reboot if NVLink/MIG-shared); (e) verify remap pending/failure both 0 and soak; (f) uncordon **only if clean**, else open RMA. Acceptance: the runbook explicitly branches on 64 / remap-failure → RMA, and never uncordons a board with a pending remap.

## Self-check

**(a)** You see XID 48 on GPU 3, immediately followed by XID 63 on the same GPU. What do you do?

**Answer:** Both are cordon-set. 48 is a double-bit (uncorrectable) ECC error — the data is already corrupt — and 63 says the GPU recorded a row for remapping that **needs a reset to activate**. Cordon the node, drain GPU-3's pods (let jobs checkpoint), then reset that GPU (`nvidia-smi -i 3 --gpu-reset`) to apply the remap. Verify `ROW_REMAP_PENDING == 0` and `ROW_REMAP_FAILURE == 0` and soak clean before uncordoning. If the remap had failed (or you'd seen XID 64), don't uncordon — open an RMA.

**(b)** XID 31 fires. Do you cordon the node, or is it the user's bug?

**Answer:** Log-only — it's almost certainly the user's bug. XID 31 is a GPU memory page fault: the application accessed memory it didn't own (out-of-bounds, use-after-free in a CUDA kernel). The GPU is healthy. Attribute it to the offending pod/namespace, emit an event telling the tenant to fix their kernel, and leave the node schedulable. The one exception: if 31 recurs across *different* tenants on the *same* GPU, suspect hardware and investigate.

**(c)** Which XID means "the GPU fell off the bus"?

**Answer:** XID 79 — "GPU has fallen off the bus." The device dropped off PCIe and is unreachable; it's fatal. Cordon, drain, and node-reset; if it recurs, RMA (often power, thermal, or a seating/HW fault).

## Resources

1. **NVIDIA XID Errors / GPU Debug Guidelines** — the canonical catalogue and remediation notes. Start here; it's the source of truth for every code above. https://docs.nvidia.com/deploy/gpu-debug-guidelines/contents.html (XID table: https://docs.nvidia.com/deploy/xid-errors/index.html)
2. **DCGM `dcgm_errors.h`** — the enum DCGM uses to classify errors, plus field IDs like `DCGM_FI_DEV_XID_ERRORS` and the row-remap fields. Ground your automation in these exact names. https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_errors.h
3. **NVSentinel overview** (reference architecture for XID→condition→cordon/drain/RMA) https://docs.nvidia.com/nvsentinel/getting-started/overview/ · **AKS GPU health monitoring** (the same via Node Problem Detector) https://learn.microsoft.com/en-us/azure/aks/gpu-health-monitoring
