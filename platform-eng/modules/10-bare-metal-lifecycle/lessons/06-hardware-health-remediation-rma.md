---
lesson: "10.6"
title: "Hardware health: closed-loop remediation and RMA"
module: "10"
concept: "Hardware health: closed-loop remediation and RMA"
status: not-started
est_time: "7h"
prev: "05-node-provisioning-pxe.md"
next: "07-storage-for-ai.md"
artifacts: []
sources: 15
---
# 10.6 · Hardware health: closed-loop remediation and RMA

> **Concept.** Close the health loop — NPD custom plugins → node Conditions → automated cordon/drain → RMA ticket → per-SKU failure-rate tracking — so a bad GPU node is detected in minutes, not a month.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 05 built the pipeline that turns a powered-on server into a labelled, `Ready`,
driverless Kubernetes node, and computed how long that takes for a rack. This lesson is
that pipeline's mirror image: what happens when a node that already made it through
provisioning goes bad, and how it gets back to the start of that pipeline.

The reason this needs its own lesson is that GPU hardware does not fail the way software
does. There is no exception, no non-zero exit code, no HTTP 500. There is a line in the
kernel ring buffer that a human would have to be watching for, a counter that ticks up,
a clock that quietly drops, or nothing at all until a collective operation times out
somewhere else in the cluster. By the time a person notices, the signal has usually been
sitting in `dmesg` for days. This lesson closes the loop: turn the physical signal into a
Kubernetes fact, turn that fact into an action, decide whether the action is a reset or a
return, and — when it is a return — put together the evidence package that makes the
vendor say yes.

## Why this matters

Start with a number that is public, specific and hard to argue with. Meta's Llama 3 405B
pre-training run used **16,384 H100 GPUs for 54 days** and recorded **466 job
interruptions: 47 planned and 419 unexpected**. Of the unexpected ones, **78% were
hardware**, with **GPU failures (including NVLink) at 30.1%** and **HBM3 memory at 17.2%**.
That is an unexpected interruption roughly **every 3 hours** across the cluster, and Meta
still reported over 90% effective training time — because they engineered for it, not
because the hardware was good.

Turn that into a rate you can reuse:

```
  GPU-and-HBM-attributable interruptions
      = 419 × (0.301 + 0.172)                    = 198 events in 54 days

  GPU-years observed
      = 16,384 GPUs × 54/365 yr                  = 2,423.6 GPU-years

  Interruption rate
      = 198 / 2,423.6                            = 0.0817 events / GPU-year
      → one job-killing GPU or HBM event per GPU roughly every 12.2 years

  Cross-check against the job-level number:
      MTBF_job = 1 / (0.0817 × 16,384) yr        = 7.5e-4 yr = 6.5 hours
      observed GPU/HBM-only interval = 54 × 24 / 198 = 6.5 hours   ✓
```

That per-GPU rate is the fact that makes fleet-scale remediation non-negotiable, because
**job-level MTBF divides by the number of nodes in the gang.** A single GPU that fails
once every twelve years is a fine component. Sixteen thousand of them, in one synchronous
job, is a failure every six and a half hours — and if your detect-and-recover loop takes
longer than that, the job never makes forward progress at all.

Now the other half. CoreWeave has said publicly that a faulty node can take **up to a
month** to detect if you rely on humans. Put the two numbers together: a fleet whose
component failure rate implies an event every few hours, and a detection process measured
in weeks. In between sits the actual cost — a degraded node that is still `schedulable`,
still accepting work, and still poisoning every gang it lands in.

The dollar translation is direct, and you will formalise it in lesson 08:

```
  goodput lost  =  detection_latency  ×  gang_size  ×  $/GPU-hr

  One sick node inside a 512-GPU synchronous job, at $2.50/GPU-hr:
      detected in 5 minutes    :  0.083 h × 512 × $2.50  =      $107
      detected in 8 hours      :  8      h × 512 × $2.50  =   $10,240
      detected in 30 days      :  720    h × 512 × $2.50  =  $921,600
```

Those are not three points on a smooth curve — they are three different businesses. The
whole of this lesson is machinery for staying on the first line.

## What's new here (calibration)

Lesson 05 mentioned XID errors and Node Problem Detector as concepts. You know *what* an
Xid is and *that* NPD exists. What is new here is everything underneath:

- **The signal layer, mapped.** Where a hardware fault physically surfaces — kernel ring
  buffer, NVML counters, DCGM health watches, systemd unit state, SMART, the BMC's system
  event log — and which of those a Kubernetes-native loop can actually consume.
- **Xid codes with NVIDIA's own recommended action**, not a folk-severity table. The
  catalog distinguishes `IGNORE`, `RESTART_APP`, `RESET_GPU`, `RESTART_BM` and
  `CONTACT_SUPPORT`, and several codes that get treated as "RMA now" in the wild are
  documented as *ignore*.
- **Memory error management as it actually works on Ampere and later** — row remapping,
  not page retirement; a 512-remap budget; the four availability buckets; and the single
  documented RMA criterion, which is not "N double-bit errors."
- **NPD's real configuration surface**: four monitor types, the exact JSON schema, the
  exit-code contract for custom plugins, and the defaults (including one that silently
  truncates your diagnostic message to 80 characters).
- **Remediation controllers as they stand in 2026**: what is maintained, what is not, the
  current Cluster API `MachineHealthCheck` API shape (which was restructured in v1beta2),
  and the `node.kubernetes.io/out-of-service` taint that makes eviction actually complete
  on a node that is gone.
- **Draining a gang-scheduled job without losing it**, including the checkpoint-interval
  arithmetic that decides whether a drain is cheap or catastrophic.
- **The RMA evidence package** — what a vendor actually requires before a claim is
  adjudicated, and why an incomplete first submission costs you weeks.
- **Failure-rate and spares arithmetic** for a fleet of `N` GPUs, so "how many spares do we
  hold" has an answer with a service level attached rather than a shrug.

## Core concepts

### 1. Why hardware failure is a detection problem, not a handling problem

Software failures announce themselves. A process crashes and the supervisor restarts it; a
request fails and the client retries. Hardware failure in a GPU node has three properties
that break that model:

1. **It is often partial.** A GPU with a degraded HBM row still computes. A NIC negotiating
   200G instead of 400G still passes traffic. An NVLink that dropped from 18 lanes to 16
   still carries collectives. Nothing is *down*; everything is *slow*, and slow in a
   synchronous job is indistinguishable from "the job is slow."
2. **It surfaces somewhere other than where it happened.** A sick GPU on node 37 shows up
   as an NCCL timeout on node 12, because node 12 is the rank that was waiting at the
   all-reduce barrier. Your first signal is almost always in the wrong place.
3. **The workload amplifies it.** In a data-parallel job every rank waits at every step
   boundary. One rank running 20% slow makes the *entire gang* run 20% slow — you are
   paying `N` GPUs for the throughput of the slowest one. This is why the goodput
   arithmetic above multiplies by gang size rather than by one.

So the engineering problem is not "how do we handle a failed GPU." It is **"how do we
convert a physical symptom into a durable, cluster-visible fact, fast enough that the
amplification does not have time to cost real money."** Everything else follows.

### 2. The signal layer: where faults physically surface

```
  ┌── PHYSICAL / FIRMWARE ────────────────────────────────────────────────────┐
  │  GPU die & HBM        NVSwitch/NVLink      NIC          NVMe      PSU/fan │
  │       │                     │               │             │          │     │
  │       │ NVIDIA kernel driver (nvidia.ko)    │ mlx5/ice    │ nvme     │ BMC │
  └───────┼─────────────────────┼───────────────┼─────────────┼──────────┼─────┘
          │                     │               │             │          │
  ┌───────▼─────────────────────▼───────────────▼─────────────▼──────────▼─────┐
  │ HOST-VISIBLE SIGNALS                                                       │
  │                                                                            │
  │  /dev/kmsg          "NVRM: Xid (PCI:0000:07:00): 79, GPU has fallen off…"  │
  │                     "NVRM: SXid (PCI:…): 20034, …"      ← NVSwitch variant │
  │  NVML / DCGM        ECC volatile+aggregate counters, remapped-row state,   │
  │                     clocks-event-reason bitmask, thermal margin,           │
  │                     NVLink error counters, XID field (DCGM_FI_DEV_XID_ERROR│
  │                     = field 230)                                           │
  │  systemd            kubelet / containerd / nvidia-persistenced unit state  │
  │  /sys, nvme-cli     SMART: media errors, critical warning, thermal counters│
  │  ethtool / ip       negotiated speed, link flaps, RDMA counters            │
  │  IPMI/Redfish SEL   PSU faults, fan faults, inlet temperature, chassis     │
  │                     intrusion — the only source that survives host death   │
  └───────┬────────────────────────────────────────────────────────────────────┘
          │
  ┌───────▼────────────────────────────────────────────────────────────────────┐
  │ COLLECTION                                                                 │
  │   node-problem-detector  ── SystemLogMonitor   (regex on kmsg/journald)     │
  │   (DaemonSet, one per     ── CustomPluginMonitor(exec a script, read exit)  │
  │    node, hostPath into    ── SystemStatsMonitor (host resource stats)       │
  │    /dev/kmsg, /var/log)   ── HealthChecker      (systemd unit health)       │
  │   dcgm-exporter          ── DCGM field values → Prometheus                  │
  │   BMC exporter           ── Redfish/IPMI SEL → Prometheus                    │
  └───────┬──────────────────────────────────┬─────────────────────────────────┘
          │ writes                           │ scraped by
  ┌───────▼──────────────────┐    ┌──────────▼──────────────────────────────────┐
  │ node.status.conditions   │    │ Prometheus + Alertmanager                   │
  │   GPUXidError=True       │    │   (thresholds, rates, per-SKU aggregation)  │
  │   GPURowRemapFailure=True│    └──────────┬──────────────────────────────────┘
  │   ContainerRuntimeUnhealthy                │
  └───────┬──────────────────┘                │
          │ watched by                        │ pages / opens tickets
  ┌───────▼───────────────────────────────────▼─────────────────────────────────┐
  │ REMEDIATION                                                                 │
  │   Medik8s NodeHealthCheck → SelfNodeRemediation / MachineDeletionRemediation│
  │   Cluster API MachineHealthCheck → provider re-provisions (lesson 05)       │
  │   a custom controller → cordon, evict, ticket, metric                       │
  └─────────────────────────────────────────────────────────────────────────────┘

  THE ARCHITECTURAL POINT: detection and remediation are deliberately decoupled by a
  DATA structure (a node Condition) rather than a call. NPD takes no action, ever. That
  is what lets you replace the remediation policy without touching detection, run two
  policies against one signal, or unit-test the policy by writing a Condition by hand.
```

Two consequences of that map are worth stating explicitly.

**The BMC is the only signal source that survives the host.** If a node hard-hangs, every
in-band source goes silent at once — and "no signal" is exactly what a healthy idle node
also looks like from a scrape. A remediation loop that only consumes in-band signals
cannot distinguish "node is fine and quiet" from "node is dead," which is why node-level
health always ends up needing either a heartbeat with a timeout (`Ready=Unknown` after the
kubelet stops reporting, which is what `node-monitor-grace-period` is for) or an
out-of-band check.

**A Condition is a fact, not a command.** Setting `GPUXidError=True` does not stop
scheduling by itself. The scheduler only avoids a node for conditions it knows about —
`Ready`, and the well-known `node.kubernetes.io/*` taints that the node-lifecycle
controller creates from `MemoryPressure`, `DiskPressure`, `PIDPressure` and
`NetworkUnavailable`. A custom condition like `GPUXidError` is invisible to the scheduler
until something translates it into a **taint** or a **cordon**. Forgetting that is the
single most common reason a "working" health loop still lets jobs land on broken nodes.

### 3. Reading an Xid

An Xid is an error identifier the NVIDIA kernel driver writes to the kernel ring buffer
when the GPU hits a fault it cannot silently recover from. The message format is fixed:

```
NVRM: Xid (PCI:0000:07:00): 79, pid=<pid>, name=<proc>, GPU has fallen off the bus.
      ───┬── ──────┬──────  ─┬─
         │         │         └── the Xid number: what class of fault
         │         └──────────── the PCI domain:bus:device of the faulting GPU
         └────────────────────── the driver's tag; SXid for NVSwitch faults
```

DCGM parses exactly this shape. Its kmsg reader uses the regular expression
`^\d+,\d+,(\d+),.*;NVRM: Xid \(PCI:(.*)\): (\d+),.*` against `/dev/kmsg` — capture group 1
is the monotonic timestamp, 2 the PCI address, 3 the Xid number — polling every 5 ms, and
by default it watches only **Xids 79, 119 and 120** unless you extend the set through the
`__DCGM_XID_KMSG__` environment variable (`NVIDIA/DCGM`,
`dcgmlib/src/DcgmKmsgReader.cpp`, read 2026-08-18). Those three defaults are informative:
79 is the GPU disappearing from the PCIe bus, and 119/120 are GSP firmware RPC failures —
the cases where the driver's own in-band telemetry becomes unreliable and a log line is the
only signal left.

**The catalog, with NVIDIA's own recommended action.** The table below reproduces entries
from NVIDIA's published Xid catalog. NVIDIA's documentation site is blocked by this
session's egress proxy, so these are verified against a machine-generated mirror of that
catalog in `leptonai/gpud` (`components/accelerator/nvidia/xid/catalog_generated.go`,
generated 2025-10-29 from `docs.nvidia.com/deploy/xid-errors/analyzing-xid-catalog.html`,
repository read 2026-08-18). The "immediate action" column is NVIDIA's, not mine.

| Xid | Mnemonic | Meaning | NVIDIA's immediate action |
|---|---|---|---|
| 13 | `GR_EXCEPTION` | Graphics engine exception | `RESTART_APP` |
| 31 | `FIFO_ERROR_MMU_ERR_FLT` | GPU memory page fault | `RESTART_APP` |
| 45 | `PREEMPTIVE_REMOVAL` | Preemptive cleanup after a previous error | Follow the *other* Xid; alone → restart Fabric Manager |
| **48** | `GPU_ECC_DBE` | Double-bit ECC error | `WORKFLOW_XID_48` — a defined procedure, **not** an automatic RMA |
| 63 | `INFOROM_DRAM_RETIREMENT_EVENT` | Row-remapping **event** (a repair succeeded) | **`IGNORE`** |
| **64** | `INFOROM_DRAM_RETIREMENT_FAILURE` | Row-remapping **failure** (repair could not be recorded) | `RESET_GPU`, then `CONTACT_SUPPORT` |
| 74 | `NVLINK_ERROR` | NVLink error | `WORKFLOW_NVLINK_ERR` |
| **79** | `GPU_HAS_FALLEN_OFF_THE_BUS` | Driver can no longer reach the device over PCIe | `RESTART_BM` (restart the bare-metal host), then `CONTACT_SUPPORT` |
| 92 | `EXCESSIVE_SBE_INTERRUPTS` | High single-bit ECC error rate | `IGNORE`, investigate if persistent |
| 94 | `CONTAINED_ERROR` | Contained memory error — blast radius is one process | `RESTART_APP` |
| 95 | `UNCONTAINED_ERROR` | Uncontained memory error — other work may be corrupted | `RESET_GPU` |
| 109 | `CTXSW_TIMEOUT_ERROR` | Context-switch timeout | `RESET_GPU` |
| 119 / 120 | `GSP_RPC_TIMEOUT` / `GSP_ERROR` | GPU System Processor firmware RPC failed | `RESET_GPU`, investigate software |
| 121 | `C2C_ERROR` | Chip-to-chip link corrected error | `IGNORE` |
| 140 | `UNRECOVERABLE_ECC_ERROR_ESCAPE` | ECC error escaped containment | `RESET_GPU` |
| 154 | `GPU_RECOVERY_ACTION_CHANGED` | Informational: the driver's recommended recovery action for this GPU changed | Informational only — read the accompanying Xid |

**Three corrections that matter for triage**, because the folk wisdom is wrong on all
three:

- **Xid 63 is not a warning.** It reports that a row remap *succeeded* — the hardware
  repaired itself. NVIDIA says `IGNORE`. What deserves attention is **Xid 64**, the
  remapping *failure*, and the remapped-row state you can read directly (§4).
- **Xid 79 is not an automatic RMA.** NVIDIA's immediate action is to restart the host.
  GPUs fall off the bus for reasons that are not the GPU: a marginal PCIe riser, a power
  event, thermal stress, a retimer. It becomes an RMA when it recurs on the same device
  after a host restart and the field diagnostic agrees.
- **94 and 95 are a pair with different blast radii.** *Contained* means the error was
  confined to one process's memory and only that application must restart. *Uncontained*
  means the driver could not guarantee containment — other work on that GPU may be
  silently corrupt, and the correct action is a GPU reset, not a retry. Treating 94 as
  fatal wastes capacity; treating 95 as recoverable ships corrupted results.

`SXid` is the NVSwitch equivalent, emitted by the fabric driver on NVSwitch-based systems
(HGX baseboards, NVL72 racks). It routes differently: an SXid usually implicates the
switch or a link, not a GPU, and its remediation frequently involves Fabric Manager rather
than a GPU reset.

### 4. Memory error management: remapping, not retirement

This is where most triage tables in the wild are a hardware generation out of date.

**Pre-Ampere: dynamic page retirement.** On uncorrectable errors the driver retired a
4 KB page of framebuffer, leaving a software-visible hole in the address space. Legacy
parts supported a maximum of **64** retirements.

**Ampere and later (A100, H100, H200, B200): row remapping.** The repair happens *in
hardware*. The memory controller swaps a degrading row for a spare row from a reserved
pool, so nothing is removed from the software-visible address space. The budget is
**up to 512 remappings** per GPU across the framebuffer, and the state is exposed as
counters you can read with `nvidia-smi -q -d ROW_REMAPPER` or, better, as DCGM fields
(verified against `NVIDIA/DCGM`, `dcgm_fields.h`, read 2026-08-18):

| Field ID | Name | What it tells you |
|---|---|---|
| 385 | `DCGM_FI_DEV_BANK_REMAP_AVAIL_MAX` | Banks with the **maximum** number of spare rows still available |
| 386 | `..._AVAIL_HIGH` | Banks with a high number of spares left |
| 387 | `..._AVAIL_PARTIAL` | Banks partially consumed |
| 388 | `..._AVAIL_LOW` | Banks with few spares left |
| 389 | `..._AVAIL_NONE` | **Banks with no spares left** — the next uncorrectable error in one of these cannot be repaired |
| 393 | `DCGM_FI_DEV_ROW_REMAP_UNCORRECTABLE_TOTAL` | Rows remapped due to uncorrectable errors |
| 394 | `DCGM_FI_DEV_ROW_REMAP_CORRECTABLE_TOTAL` | Rows remapped due to correctable errors |
| **395** | **`DCGM_FI_DEV_ROW_REMAP_FAILED`** | **The remapping-failure flag.** This is the RMA signal. |
| 396 | `DCGM_FI_DEV_ROW_REMAP_PENDING` | A remap is queued but needs a GPU reset or host reboot to take effect |
| 390–392 | `..._PAGE_RETIRED_{SBE,DBE,PENDING}` | The legacy page-retirement counters, for pre-Ampere parts |

**The RMA criterion, stated precisely.** NVIDIA's memory-error-management documentation
gives one rule: *the RMA criterion is met when the row-remapping failure flag is set and
validated by the field diagnostic.* The flag is raised by events including a remapping
attempt for an uncorrectable error on a bank that already has eight uncorrectable-error
rows remapped, or a remapping attempt on a row that was already remapped. Both `leptonai/gpud`
and NVIDIA's own DCGM validation suite implement it as *"remapping failed"* alone, without
also requiring an uncorrectable-error count threshold — gpud's source comments the reason:
the failure flag can be set with fewer than eight remaps to the same bank, so gating on the
count would miss real cases.

Two operationally distinct states fall out, and confusing them is expensive:

- **`ROW_REMAP_PENDING` (396) set** → the GPU has queued a repair it cannot apply while in
  use. Action: **reset the GPU or reboot the host**, then re-check. This is a scheduled
  maintenance item, not a return.
- **`ROW_REMAP_FAILED` (395) set** → the repair could not be scheduled at all. Action:
  **remove the GPU from service and start the RMA**, with a field-diagnostic run as the
  supporting evidence.

Notice what is *not* on that list: a count of double-bit errors. "RMA after N DBEs" is a
rule people repeat and NVIDIA does not publish. The measurable, documented trigger is the
failure flag, plus the field diagnostic as adjudication. **Where the exact recurrence
thresholds you want are genuinely not published, say so and instrument for the flag rather
than inventing a number** — an internal policy like "three uncontained errors in seven days
on the same device escalates to a support case" is a perfectly defensible *operational*
rule as long as it is labelled as yours, not as NVIDIA's.

### 5. Node Problem Detector, configured for real

NPD is a DaemonSet that runs *problem daemons*, each of which converts some host signal
into a Kubernetes Event (temporary) or a permanent node Condition. It ships **four**
monitor types, and a mature GPU-node health story uses at least three of them.

| Monitor | Watches | GPU-node use |
|---|---|---|
| **SystemLogMonitor** | Regex rules against a log source: the `kmsg` plugin reading `/dev/kmsg`, the `filelog` plugin tailing a file, or `journald` | Match `NVRM: Xid` lines directly. Streaming, so latency is sub-second. |
| **CustomPluginMonitor** | Runs a script or binary on an interval and maps its exit code to a Condition or Event | Anything you can express as a check: `dcgmi health`, a remapped-row query, an `nvidia-smi` GPU count assertion |
| **SystemStatsMonitor** | Host resource statistics (disk, memory, CPU, OS features) into conditions and metrics | Disk pressure on the NVMe scratch tier from lesson 07; host memory exhaustion |
| **HealthChecker** | systemd unit health for a named component | `kubelet`, `containerd`, `nvidia-persistenced` — a node with perfect GPUs and a crash-looping runtime is still unschedulable |

**The custom-plugin contract**, which is stricter than people expect
(`kubernetes/node-problem-detector`, `pkg/custompluginmonitor/types/`, read 2026-08-18):

- **Exit code 0 = OK, 1 = NonOK, 2 = Unknown.** Any other exit code is treated as
  `Unknown`. A plugin that exits 127 because your `dcgmi` binary is not on `$PATH` reports
  *unknown*, not *problem* — and a policy that only acts on `True` will silently ignore a
  broken checker forever. Alert on `Unknown` as well as on `True`.
- **stdout becomes the Condition message, truncated to `max_output_length`, which defaults
  to 80 characters.** Your carefully formatted diagnostic gets cut mid-word. Print the
  identifying facts first (device, Xid, count) and the prose second, or raise the limit.
- **Defaults:** `invoke_interval` 30 s, per-plugin `timeout` 5 s, global `concurrency` 3,
  `enable_message_change_based_condition_update` false, `skip_initial_status` false. The
  5-second global timeout is the one that bites: `dcgmi health --check` on a busy 8-GPU
  node can exceed it, and a timed-out plugin reports `Unknown`.

A real config, in the schema NPD actually parses:

```json
{
  "plugin": "custom",
  "pluginConfig": {
    "invoke_interval": "30s",
    "timeout": "20s",
    "max_output_length": 400,
    "concurrency": 2,
    "enable_message_change_based_condition_update": true
  },
  "source": "gpu-health-custom-plugin-monitor",
  "metricsReporting": true,
  "conditions": [
    { "type": "GPUUnhealthy",       "reason": "GPUHealthy",
      "message": "no GPU fault detected" },
    { "type": "GPURowRemapFailure", "reason": "NoRowRemapFailure",
      "message": "no row-remapping failure on any GPU" },
    { "type": "GPUCountMismatch",   "reason": "AllGPUsPresent",
      "message": "expected GPU count present" }
  ],
  "rules": [
    { "type": "permanent", "condition": "GPUUnhealthy",
      "reason": "DcgmHealthFailure",
      "path": "/config/plugin/check_dcgm_health.sh", "timeout": "20s" },
    { "type": "permanent", "condition": "GPURowRemapFailure",
      "reason": "RowRemapFailed",
      "path": "/config/plugin/check_row_remap.sh",   "timeout": "10s" },
    { "type": "permanent", "condition": "GPUCountMismatch",
      "reason": "GPUMissing",
      "path": "/config/plugin/check_gpu_count.sh",   "timeout": "10s",
      "args": ["8"] }
  ]
}
```

And the check that matters most, because it keys off the one documented RMA criterion:

```bash
#!/usr/bin/env bash
# check_row_remap.sh — exit 0 healthy, 1 problem, 2 unknown.
# Reads the remapping-FAILURE flag (DCGM field 395 equivalent) per GPU.
set -uo pipefail

command -v nvidia-smi >/dev/null || { echo "nvidia-smi not present"; exit 2; }

out=$(nvidia-smi --query-remapped-rows=gpu_uuid,remapped_rows.failure,remapped_rows.pending,remapped_rows.uncorrectable \
                 --format=csv,noheader,nounits 2>/dev/null) || { echo "query failed"; exit 2; }

bad=""
while IFS=, read -r uuid failure pending uncorr; do
  failure=${failure// /}; pending=${pending// /}
  [[ "$failure" == "Yes" || "$failure" == "1" ]] && bad+="${uuid}:FAILED "
done <<< "$out"

if [[ -n "$bad" ]]; then
  # identifying facts FIRST: max_output_length truncates the tail.
  echo "RowRemapFailure ${bad}"
  exit 1
fi
exit 0
```

Note the shape of that script: it exits **2** when it cannot tell, **1** only when it has
positive evidence, and it puts the GPU UUID at the front of the message because that UUID
is the primary key of the RMA (§9).

**Detection latency, end to end.** A `CustomPluginMonitor` contributes up to its
`invoke_interval` (30 s by default). NPD's condition manager checks for updates every
**1 second** and force-resyncs with the API server every **10 seconds**, with a
configurable heartbeat (`--k8s-exporter-heartbeat-period`, default **5 minutes**) for a
forced sync even when nothing changed (`pkg/exporters/k8sexporter/condition/manager.go`,
read 2026-08-18). So the pipeline from fault to visible Condition is: `invoke_interval` +
~1 s + one API write ≈ **31 seconds worst case**, or sub-second for a `SystemLogMonitor`
rule since `/dev/kmsg` is streamed. That is your detection budget, and it is a two-order-
of-magnitude improvement on a human reading dashboards — which is the entire argument.

### 6. The fault lifecycle, as a state machine

```
                                 ┌─────────────┐
                                 │   HEALTHY   │◀───────────────────────────────┐
                                 │ schedulable │                                │
                                 └──────┬──────┘                                │
                     first signal:      │                                       │
                     Xid / DCGM health / remap-pending / unit down               │
                                        ▼                                       │
                                 ┌─────────────┐                                │
                                 │  SUSPECT    │  Condition set. NOT yet cordoned.│
                                 │             │  Grace window: does it clear?   │
                                 └──┬───────┬──┘                                │
      clears within grace ──────────┘       └───── persists past `duration` ─────┐│
      (transient, e.g. Xid 92,                     (NHC unhealthyConditions[].    ││
       single correctable ECC)                      duration; CAPI               ││
              │                                     timeoutSeconds)              ││
              │                                              ▼                   ││
              │                                    ┌──────────────────┐          ││
              │                                    │    CORDONED      │          ││
              │                                    │ unschedulable;   │          ││
              │                                    │ running pods stay │         ││
              │                                    └────────┬─────────┘          ││
              │                                             │ policy decides when││
              │                                             │ (immediately, or at││
              │                                             │  next checkpoint)  ││
              │                                             ▼                    ││
              │                                    ┌──────────────────┐          ││
              │                                    │    DRAINING      │          ││
              │                                    │ eviction API,    │          ││
              │                                    │ respects PDBs +  │          ││
              │                                    │ grace period     │          ││
              │                                    └────────┬─────────┘          ││
              │                     drain blocked by a PDB ─┤                    ││
              │                     → escalate or force ────┘                    ││
              │                                             ▼                    ││
              │                                    ┌──────────────────┐          ││
              │                                    │     TRIAGE       │          ││
              │                                    │ which class of   │          ││
              │                                    │ fault is this?   │          ││
              │                                    └──┬────┬──────┬───┘          ││
              │   ┌───────────────────────────────────┘    │      └──────────┐   ││
              │   │ SOFT                            RESETTABLE            HARD│   ││
              │   │ Xid 92, 121, single             Xid 94/109/119/120,   Xid │   ││
              │   │ correctable ECC, a              remap PENDING (396)   64, │   ││
              │   │ crash-looped unit               → GPU reset or reboot  95,│   ││
              │   ▼                                        │              remap│  ││
              │ ┌──────────┐                        ┌──────▼───────┐    FAILED│  ││
              │ │ RESTART  │                        │  GPU RESET / │    (395) │  ││
              │ │ UNIT /   │                        │  HOST REBOOT │      │   │  ││
              │ │ APP ONLY │                        └──────┬───────┘      ▼   │  ││
              │ └────┬─────┘                               │        ┌─────────▼┐ ││
              │      │                              re-run health   │  RMA     │ ││
              │      │                              check + short   │ REQUIRED │ ││
              │      │                              diag (dcgmi     └────┬─────┘ ││
              │      │                              diag -r 1)           │       ││
              │      │                                     │             ▼       ││
              │      │                          ┌──────────┴──┐   ┌──────────────┴┐
              │      │                          │ clean → back│   │ EVIDENCE      │
              │      │                          │ to service  │   │ nvidia-bug-   │
              │      │                          └──────┬──────┘   │ report.sh,    │
              │      │                                 │          │ dcgmi diag -r3│
              │      │                       recurs ≥ K times     │ serials, Xid, │
              │      │                       in window W ──────┐  │ field diag    │
              │      │                                          │  └──────┬───────┘
              │      │                                          ▼         ▼
              │      │                                   ┌───────────────────────┐
              │      │                                   │  VENDOR RMA OPEN      │
              │      │                                   │  clock starts; node   │
              │      │                                   │  is capacity you own  │
              │      │                                   │  and cannot use       │
              │      │                                   └──────────┬────────────┘
              │      │                                              │ part arrives,
              │      │                                              │ swap performed
              │      │                                              ▼
              │      │                                   ┌───────────────────────┐
              │      │                                   │ RE-PROVISION          │
              │      │                                   │ BareMetalHost →       │
              │      │                                   │ available → clean →   │
              │      │                                   │ re-image → rejoin     │
              │      │                                   │        (LESSON 05)    │
              │      │                                   └──────────┬────────────┘
              │      │                                              │
              │      │                                              ▼
              │      │                                   ┌───────────────────────┐
              │      │                                   │ BURN-IN / VERIFY      │
              │      │                                   │ dcgmi diag -r 3,      │
              │      │                                   │ NCCL all-reduce,      │
              │      │                                   │ ~20 min full-GPU load │
              │      │                                   └──────────┬────────────┘
              └──────┴──────────────────────────────────────────────┴─────────────┘
                                  uncordon → HEALTHY

  THE DECISION AT EACH TRANSITION, in one line each:
    HEALTHY→SUSPECT     : did a watched signal fire at all?
    SUSPECT→HEALTHY     : did it clear inside the grace window? (transients must not
                          cost you a node — this window is why `duration` exists)
    SUSPECT→CORDONED    : has it persisted past `duration`, AND does the storm guard
                          allow it? (see §7 — never remediate when half the fleet is
                          "unhealthy"; that is your monitoring failing, not the fleet)
    CORDONED→DRAINING   : is the running job at a safe point, or is the fault severe
                          enough that corrupt output beats a lost checkpoint?
                          (Xid 95 uncontained → drain now; Xid 92 → wait for a
                          checkpoint boundary)
    DRAINING→TRIAGE     : which bucket does the signal fall in per §3/§4?
    TRIAGE→RMA          : is the row-remap FAILURE flag set, or has a resettable
                          fault recurred K times in window W after a clean reset?
    RMA→RE-PROVISION    : replacement part installed and inventory matches expected
    RE-PROVISION→VERIFY : node is Ready and labelled (lesson 05's handoff line)
    VERIFY→HEALTHY      : full-GPU burn-in AND a multi-node collective both pass
```

The two transitions people get wrong are the ones with grace windows. **SUSPECT→HEALTHY
exists so that transients do not consume nodes** — remove it and a single correctable ECC
event drains a node. **SUSPECT→CORDONED has a storm guard** because the most common cause
of "80% of nodes went unhealthy at once" is a monitoring or driver bug, and a loop without
a guard will happily cordon your entire fleet in response to it.

### 7. Remediation controllers, as of 2026

The Condition is written. Something has to act on it. The landscape, honestly:

- **draino (`planetlabs/draino`)** — the original "watch a Condition, cordon and drain"
  controller, and the one every blog post still names. **Its last commit is 2020-12-14**
  (verified by cloning the repository, 2026-08-18). It predates the current eviction
  semantics, the `out-of-service` taint, and every API change since Kubernetes 1.20.
  Read it — it is a genuinely small and clear implementation of the pattern, about the
  size of a long file — but do not deploy it and do not cite it as current tooling.
- **Medik8s Node Healthcheck Operator (NHC) + Self-Node-Remediation (SNR)** — actively
  maintained (last commit 2026-07-27). This is the maintained successor to the draino
  pattern and the one to reach for on a cluster that is not CAPI-managed.
- **Cluster API `MachineHealthCheck`** — the right answer when your nodes are CAPI
  `Machine`s backed by Metal3 or Tinkerbell (lesson 05), because remediation closes the
  loop into re-provisioning automatically.
- **A custom controller** — the "watch Condition, cordon, evict, file ticket, emit metric"
  loop is genuinely small. Cloudflare's published write-up of building exactly this is the
  canonical reference for the shape. Write your own when your triage policy is the product
  (which, at a neocloud, it is).

**NHC, with the fields that matter** (`medik8s/node-healthcheck-operator`,
`api/v1alpha1/nodehealthcheck_types.go`, read 2026-08-18):

```yaml
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: gpu-nodes
spec:
  selector:
    matchLabels:
      sku: hgx-h100-8g          # mandatory in practice: never mix control-plane
                                # and worker nodes in one NHC
  unhealthyConditions:
    # the API default is Ready=False/Unknown for 300s; these are additions
    - type: GPURowRemapFailure
      status: "True"
      duration: 60s             # hard fault: short grace window
    - type: GPUUnhealthy
      status: "True"
      duration: 600s            # DCGM health warnings: give transients time to clear
    - type: Ready
      status: "False"
      duration: 300s
  minHealthy: "80%"             # remediate only while ≥80% of selected nodes are healthy
                                # (mutually exclusive with maxUnhealthy — set one)
  stormCooldownDuration: 30m    # after a storm, pause: async status updates otherwise
                                # cause a second, unnecessary remediation round
  escalatingRemediations:
    - order: 0                  # try the cheap thing first
      timeout: 15m
      remediationTemplate:
        apiVersion: self-node-remediation.medik8s.io/v1alpha1
        kind: SelfNodeRemediationTemplate
        name: snr-reboot
        namespace: openshift-operators
    - order: 1                  # then the expensive one
      timeout: 30m
      remediationTemplate:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: Metal3RemediationTemplate
        name: reprovision
        namespace: metal3
```

Two design features there are worth stealing even if you write your own controller.
**`escalatingRemediations`** encodes "reboot first, re-provision if that did not work" as
declarative policy with per-step timeouts, rather than as tribal knowledge. And
**`stormCooldownDuration`** exists because node status updates are asynchronous: when the
root cause of a mass-unhealthy event is fixed, nodes recover in waves, the healthy count
crosses back over the threshold partway through, and a naive controller starts remediating
the nodes that were about to recover on their own.

SNR's `remediationStrategy` is an enum of `Automatic` (default), `ResourceDeletion` and
`OutOfServiceTaint`. The last one is the important one for GPU fleets: it applies the
well-known **`node.kubernetes.io/out-of-service`** taint, which tells the rest of the
control plane that the node is genuinely gone and it is safe to force-delete its pods and
detach its volumes. Without it, a node that has hard-hung leaves pods in `Terminating`
forever — the kubelet that is supposed to confirm deletion is not running — and a
StatefulSet or a volume-backed training job never reschedules.

**Cluster API `MachineHealthCheck` was restructured in v1beta2**, and manifests written
against the old shape will not apply (verified against `kubernetes-sigs/cluster-api`,
`api/core/v1beta2/machinehealthcheck_types.go`, commit `20e5ac6`, read 2026-08-18):

| v1beta1 (old) | v1beta2 (current) |
|---|---|
| `spec.unhealthyConditions[]` with `timeout: "5m"` | `spec.checks.unhealthyNodeConditions[]` with `timeoutSeconds: 300` |
| `spec.nodeStartupTimeout: "10m"` | `spec.checks.nodeStartupTimeoutSeconds: 600` (default 600; `0` disables) |
| `spec.maxUnhealthy: "40%"` | `spec.remediation.triggerIf.unhealthyLessThanOrEqualTo: "40%"` |
| `spec.unhealthyRange: "[3-5]"` | `spec.remediation.triggerIf.unhealthyInRange: "[3-5]"` |
| `spec.remediationTemplate` | `spec.remediation.templateRef` |
| — | `spec.checks.unhealthyMachineConditions[]` (new: conditions on the `Machine`, not the `Node`) |

```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: MachineHealthCheck
metadata:
  name: gpu-workers
  namespace: metal3
spec:
  clusterName: prod
  selector:
    matchLabels:
      nodepool: gpu
  checks:
    nodeStartupTimeoutSeconds: 1800     # bare metal POSTs slowly — lesson 05 §12
    unhealthyNodeConditions:
      - type: GPURowRemapFailure
        status: "True"
        timeoutSeconds: 60
      - type: Ready
        status: "Unknown"
        timeoutSeconds: 300
  remediation:
    triggerIf:
      unhealthyLessThanOrEqualTo: "20%"
    templateRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: Metal3RemediationTemplate
      name: gpu-remediation
```

Note `nodeStartupTimeoutSeconds: 1800`. The default of 600 seconds is tuned for cloud VMs.
A bare-metal GPU node spends five to eight minutes on firmware POST alone before it can
even PXE (lesson 05 §12), so a 10-minute startup timeout on metal will declare healthy
nodes unhealthy during a rack bring-up and put you in a re-provisioning loop. **Every
timeout in a bare-metal health check needs to be re-derived from the provisioning
arithmetic, not inherited from a cloud default.**

CAPI also always flags two things regardless of your configuration: a `Machine` carrying
the `cluster.x-k8s.io/remediate-machine` annotation, and a `Machine` whose `Node` has been
deleted. The annotation is your manual override — it is how an on-call engineer says
"remediate this one now" without editing policy.

### 8. Draining a gang-scheduled job without losing it

A drain is only safe if the job can resume. Four mechanisms have to line up.

**Cordon first, always.** `kubectl cordon` sets `spec.unschedulable`, which the
node-lifecycle controller turns into the `node.kubernetes.io/unschedulable:NoSchedule`
taint. Existing pods keep running; nothing new lands. Cordon costs nothing and buys you
time to decide.

**Evict, never delete.** The eviction API (`POST .../pods/<name>/eviction`) is the *only*
path that consults `PodDisruptionBudget`s. A raw `kubectl delete pod` bypasses PDBs
entirely. `kubectl drain` uses eviction by default; `--disable-eviction` turns it into
delete, which is a footgun with a polite name.

**Set a grace period long enough to checkpoint.** Eviction sends `SIGTERM` and waits
`terminationGracePeriodSeconds` (default **30**) before `SIGKILL`. Thirty seconds is not
enough to write a multi-hundred-gigabyte checkpoint over a shared filesystem — you will do
that arithmetic in lesson 07 — so the training job's pod spec must set a grace period that
covers a checkpoint write, and the framework must actually trap `SIGTERM` and write one.
`kubectl drain --timeout` bounds the whole operation and must exceed the grace period, or
the drain gives up while the checkpoint is still being written.

**The PDB has to be gang-aware, and this is where it goes wrong.** A synchronous training
job of `R` ranks cannot survive losing one rank; a PDB of `minAvailable: R-1` is a lie that
lets eviction proceed and kills the job. The honest expressions are:

```yaml
# A gang that must be whole: block eviction entirely, and let the drain FAIL loudly
# so the operator (or a controller) makes a deliberate decision.
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: llm-pretrain-gang }
spec:
  minAvailable: 64                # == the full gang. Any eviction is blocked.
  selector:
    matchLabels: { job-name: llm-pretrain }
  unhealthyPodEvictionPolicy: AlwaysAllow   # let already-broken pods go regardless
```

`unhealthyPodEvictionPolicy: AlwaysAllow` (GA since Kubernetes 1.27) is the field that
prevents the nastiest version of this: with the default `IfHealthyBudget`, pods that are
*already* not `Ready` still count against the budget, so a job that has partially crashed
can block its own node's drain forever. `AlwaysAllow` says an unready pod may always be
evicted.

**A blocked drain is a feature, not a bug** — it is the system refusing to silently destroy
a job. The correct response is a policy decision made by something that understands the
job: a gang-aware scheduler (Volcano, Kueue, JobSet) that can signal-checkpoint the whole
gang, mark it for requeue, and then release the PDB. What you must not do is reach for
`--force`, which deletes pods without eviction and takes the job with it.

**How much does the drain actually cost?** This is the arithmetic that decides whether you
drain immediately or wait for a checkpoint boundary. With a checkpoint interval `Δ`, a
checkpoint write time `δ`, and a restart/recovery time `ρ`, an unplanned interruption
costs on average `Δ/2 + ρ` of lost work. The optimal interval is Young's / Daly's classic
result, `Δ* = sqrt(2·δ·M)`, where `M` is the job-level MTBF — and `M` is where §1's
amplification returns, because `M_job = M_node / N`:

```
  A 64-node (512-GPU) job. Per-node MTBF from your own remediation metric: 8,000 h.

      M_job = 8,000 / 64                            = 125 h  = 4.5e5 s
      δ (checkpoint write, from lesson 07)          = 90 s
      ρ (restart, reload, re-warm)                  = 420 s

      Δ* = sqrt(2 × 90 × 4.5e5) = sqrt(8.1e7)       = 9,000 s = 2.5 h
      expected loss per interruption = Δ*/2 + ρ = 4,500 + 420 = 4,920 s = 1.37 h
      interruptions per 30-day run   = 720 / 125                        = 5.76
      lost time per run              = 5.76 × 1.37 h                    = 7.9 h  (1.1%)

  Now the same job on a fleet with a 2,000 h per-node MTBF (a poorly maintained fleet
  or a bad SKU batch):

      M_job = 2,000 / 64 = 31.25 h = 1.125e5 s
      Δ*     = sqrt(2 × 90 × 1.125e5) = 4,500 s = 1.25 h
      loss/interruption = 2,250 + 420 = 2,670 s = 0.74 h
      interruptions per 30-day run = 720 / 31.25 = 23.0
      lost time per run = 23.0 × 0.74 h = 17.0 h  (2.4%)
```

Two findings fall out, and the second one is the interesting one.

*First:* a 4× worse node MTBF only doubles lost time — **because checkpointing works**.
The optimal checkpoint interval adapts, and the loss grows roughly as `sqrt(1/M)`. This is
the same result lesson 11.07 derives from the other direction when it compares reliability
tiers, and it is why "reliability tier" is mostly *not* a goodput tax.

*Second, and this is the part that justifies this entire lesson:* **that model assumes the
fault is detected at the moment it occurs.** The Young/Daly result governs *crash* faults.
It says nothing about a node that silently degrades and keeps running. For a degradation
detected after `T_d`, the cost is not `Δ/2` — it is `T_d × N` GPU-hours of work running at
whatever fraction of speed the sick node imposes, plus every checkpoint written during that
window being a checkpoint of a slow run. **Fast detection is worth far more than frequent
checkpointing, because checkpointing only helps with the failure mode that announces
itself.**

### 9. The RMA evidence package

A vendor RMA is a warranty claim adjudicated on evidence. An incomplete first submission
does not get a polite request for more — it gets queued, bounced, and re-queued, and you
lose the two weeks that were the entire point of detecting the fault in minutes.

Assemble it automatically, at the moment of the drain, while the machine is still in the
state that produced the fault. NVIDIA's own guidance names the core artifacts:

| Artifact | How to get it | Why the vendor wants it |
|---|---|---|
| **`nvidia-bug-report.sh` output** | `sudo nvidia-bug-report.sh` → `nvidia-bug-report.log.gz` | The single most-requested file. Bundles `lspci`, kernel logs, `nvidia-smi -q`, driver state, X logs. NVIDIA's documentation states that omitting it may delay processing. **Run it immediately after the fault, before any reboot** — a reboot clears the ring buffer that contains the evidence. |
| **DCGM diagnostic** | `dcgmi diag -r 3` (the "long" run, `DCGM_DIAG_LVL_LONG` = 30; `-r 4` is `xlong` = 40) | NVIDIA's own tool failing is the strongest evidence short of the field diagnostic. Budget roughly 30 minutes. |
| **NVIDIA Field Diagnostic** | Vendor-supplied `fieldiag` binary | Authoritative and usually *required* before an RMA can be opened. This is what "validated by the field diagnostic" means in the row-remapping RMA criterion (§4). |
| **GPU identity** | `nvidia-smi --query-gpu=uuid,serial,pci.bus_id,vbios_version,inforom.img --format=csv` | The GPU UUID and board serial are the primary keys of the claim. A ticket without them cannot be matched to a warranty record. |
| **Chassis identity** | BMC/Redfish `SerialNumber`, `Model`, `PartNumber`; the node name and rack/slot | Which physical box a technician goes to. |
| **The fault evidence itself** | The `dmesg`/journald excerpt with the `NVRM: Xid` line, timestamped; the remapped-row query output; DCGM field values 385–396 | What actually happened, in the vendor's own vocabulary. |
| **Firmware and driver versions** | Driver version, VBIOS, BMC and BIOS versions, InfoROM image version | Rules out known-fixed firmware bugs and is the first thing support checks. |
| **Impact** | GPU-hours lost, jobs affected, time in service | Not required for adjudication; decisive in an escalation. |

```bash
#!/usr/bin/env bash
# collect_rma_evidence.sh — run ON the drained node, BEFORE any reboot.
set -euo pipefail
NODE=$(hostname); TS=$(date -u +%Y%m%dT%H%M%SZ); OUT="/var/log/rma/${NODE}-${TS}"
mkdir -p "$OUT"

# 1. Identity — the primary keys of the claim.
nvidia-smi --query-gpu=index,uuid,serial,pci.bus_id,name,vbios_version,inforom.img \
           --format=csv > "$OUT/gpu-identity.csv"

# 2. Memory-error state — the documented RMA criterion lives here.
nvidia-smi --query-remapped-rows=gpu_uuid,remapped_rows.failure,remapped_rows.pending,\
remapped_rows.correctable,remapped_rows.uncorrectable --format=csv > "$OUT/remapped-rows.csv"
nvidia-smi -q -d ECC,ROW_REMAPPER > "$OUT/nvidia-smi-ecc.txt" 2>&1 || true

# 3. The fault evidence, with timestamps, BEFORE the ring buffer is lost.
journalctl -k --since "-24h" --no-pager | grep -E 'NVRM: (S?)Xid' > "$OUT/xid.log" || true
dmesg -T > "$OUT/dmesg-full.log" 2>&1 || true

# 4. NVIDIA's bundle. Slow; run it anyway, it is the file support asks for first.
nvidia-bug-report.sh --output-file "$OUT/nvidia-bug-report" >/dev/null 2>&1 || true

# 5. Diagnostics. -r 3 is the long run (~30 min); -r 1 if you need the node back sooner.
dcgmi diag -r 3 -j > "$OUT/dcgmi-diag-r3.json" 2>&1 || true

# 6. Chassis identity from the BMC — survives even if the host later dies.
curl -sk -u "$BMC_USER:$BMC_PASS" "https://$BMC_HOST/redfish/v1/Systems/System.Embedded.1" \
  | jq '{Model, SerialNumber, PartNumber, PowerState, BiosVersion}' > "$OUT/chassis.json" || true

# 7. Firmware inventory.
nvidia-smi -q | grep -E 'Driver Version|VBIOS|Inforom' > "$OUT/versions.txt" || true

tar czf "${OUT}.tar.gz" -C "$(dirname "$OUT")" "$(basename "$OUT")"
echo "${OUT}.tar.gz"
```

The controller that drained the node should invoke this and attach the tarball to the
ticket automatically. **The step that people skip is ordering: bug-report and Xid capture
must happen before the reset or reboot**, because a reset is often the first thing an
operator tries and it destroys exactly the evidence the vendor will ask for.

### 10. Failure rates and spares: how many do you hold?

Now the capacity question. You have `N` GPUs. Components fail at some annualised rate. RMA
turnaround takes some number of days. How many spare nodes do you need on the shelf, and
what does that cost?

**Step 1 — establish your failure rate, and be honest about which one you are measuring.**
Two different rates get conflated:

- **Interruption rate** — events that stop a job. Meta's Llama 3 data gives
  **0.082 GPU-and-HBM-attributable interruptions per GPU-year** (§ "Why this matters").
  Most of these are recoverable by reset.
- **Replacement rate (AFR)** — devices that leave service permanently. This is strictly
  smaller and is the number that drives spares. Public, audited figures for datacenter GPU
  AFR are not something any vendor publishes; treat any single quoted figure with
  suspicion. **Measure your own** from the metric in §11 and parameterise until you have.

The model below uses `AFR` as an explicit parameter and shows the answer across a plausible
band, because that is the honest way to present it:

```
  PARAMETERS  (substitute your own; these are the shapes, not your numbers)
    N_gpu   fleet size, GPUs                        = 512     (64 nodes × 8)
    AFR     annualised replacement rate per GPU     = 2% / 3% / 5%     ← the sweep
    L       RMA lead time, order to installed, days = 21
    F       failure→replacement unit (the FRU)      = 8-GPU HGX baseboard
    SL      target service level (no stockout)      = 95%

  STEP 1 — expected replacement events per year
    events/yr = N_gpu × AFR
       AFR 2% :  512 × 0.02 = 10.2 GPU-replacement events/yr
       AFR 3% :  512 × 0.03 = 15.4
       AFR 5% :  512 × 0.05 = 25.6

  STEP 2 — the FRU multiplier, which is the thing people miss
    On SXM/HGX systems the GPU is not a socketed part. The field-replaceable unit is
    typically the whole 8-GPU baseboard (or, with some vendors, an individual SXM
    module — CONFIRM THIS WITH YOUR VENDOR, it changes the answer by 8×).
    If the FRU is the baseboard, one GPU failure removes 8 GPUs from service and
    consumes one baseboard from the spares pool.

      baseboard events/yr ≈ GPU events/yr   (each event consumes one baseboard)
      GPU-capacity removed per event        = 8 GPUs

  STEP 3 — demand during the lead time, and the spares count
    Replacement events are approximately independent, so demand during a lead time of
    L days is Poisson with mean λ:

      λ = events/yr × L/365
        AFR 3% :  15.4 × 21/365 = 0.884 events per lead-time window

    Choose the smallest s with P(demand ≤ s) ≥ SL.  For λ = 0.884:
      P(0)  = e^-0.884                       = 0.413
      P(≤1) = 0.413 × (1 + 0.884)            = 0.778
      P(≤2) = 0.778 + e^-0.884·0.884²/2      = 0.939
      P(≤3) = 0.939 + e^-0.884·0.884³/6      = 0.987   ← first ≥ 0.95

      → hold s = 3 spare baseboards for a 512-GPU fleet at AFR 3%, L = 21 d, SL = 95%

    Sweep:
      AFR 2%, λ = 0.587  →  P(≤2) = 0.977              →  s = 2
      AFR 3%, λ = 0.884  →  P(≤3) = 0.987              →  s = 3
      AFR 5%, λ = 1.473  →  P(≤3) = 0.941, P(≤4)=0.980 →  s = 4

  STEP 4 — what the spares cost, and what they buy
    Spare baseboard cost ≈ the GPU share of a node's capex. From lesson 08's model a
    node is ~$280K and the 8 GPUs are the dominant share; take $180K/baseboard as the
    parameter (state yours).

      holding cost (3 spares)              = 3 × $180K = $540K of idle capital
      annualised at a 12% cost of capital  = $64.8K/yr

    Against the alternative — no spares, wait out the lead time:
      capacity lost per event = 8 GPUs × 21 days × 24 h        = 4,032 GPU-hours
      at $2.50/GPU-hr                                          = $10,080 per event
      × 15.4 events/yr                                         = $155K/yr

      → spares pay for themselves at these parameters (155 > 65) with room to spare,
        and the margin widens with fleet size because λ grows linearly while the
        Poisson service-level requirement grows sub-linearly.

  STEP 5 — the sensitivity that actually decides it
    Lead time L is the lever, not AFR. Halving L from 21 to 10 days:
      λ (AFR 3%) = 15.4 × 10/365 = 0.422  →  P(≤2) = 0.992  →  s = 2 spares
    and it halves the capacity lost per unspared event. A vendor contract with a
    guaranteed 4-hour or next-business-day replacement is worth more than a lower
    sticker price on the hardware, and this calculation is how you say so with numbers
    in a procurement meeting.
```

Two structural notes on that model. **`λ` scaling sub-linearly in the spares count is why
big fleets are cheaper to run per GPU** — a 4,096-GPU fleet at the same AFR needs
`λ = 7.07` and `s = 11` for 95%, which is 11 spares for 8× the fleet rather than 24. That
is a genuine, quantified economy of scale and it belongs in lesson 08's model. And **the
FRU question is the highest-leverage thing to confirm before you buy**: if your vendor
replaces individual SXM modules rather than baseboards, both the capacity lost per event
and the spares capital fall by roughly 8×.

### 11. Per-SKU failure tracking: turning incidents into leverage

Every remediation should emit a metric. This is the cheapest thing in the lesson and the
one that changes vendor conversations.

```
# emitted by your remediation controller, once per remediation
gpu_node_remediation_total{
  sku="hgx-h100-8g",
  batch="2026-Q1",
  dc="iad1", rack="c14",
  node="gpu-node-07",
  gpu_uuid="GPU-4c2b…",
  signal="row_remap_failure",   # or xid_79, xid_95, dcgm_health, unit_down
  action="rma"                  # or reset, reboot, reprovision
} 1

# and a gauge for the population, so you can compute a rate
gpu_fleet_gpus{sku="hgx-h100-8g", batch="2026-Q1", dc="iad1"} 512
```

The queries that matter:

```promql
# failures per 1,000 GPU-hours, by SKU and batch, over 30 days
(
  sum by (sku, batch) (increase(gpu_node_remediation_total{action="rma"}[30d]))
)
/
(
  sum by (sku, batch) (avg_over_time(gpu_fleet_gpus[30d])) * 24 * 30 / 1000
)

# is one batch an outlier against the fleet? (ratio > ~2 is a conversation)
(
  sum by (batch) (increase(gpu_node_remediation_total{action="rma"}[90d]))
  / sum by (batch) (avg_over_time(gpu_fleet_gpus[90d]))
)
/ scalar(
  sum(increase(gpu_node_remediation_total{action="rma"}[90d]))
  / sum(avg_over_time(gpu_fleet_gpus[90d]))
)

# detection latency SLO: time from first Xid to node cordoned, p99
histogram_quantile(0.99, sum by (le) (rate(remediation_detect_seconds_bucket[7d])))
```

Three things this buys you that nothing else does:

1. **Procurement leverage.** "Batch 2026-Q1 is failing at 3.1× the fleet average across
   90 days and 512 GPUs" is a claim with a denominator, and it is what gets you an
   accelerated replacement, a credit, or a batch recall. "We've had some hardware
   problems" gets you a support engineer asking for logs.
2. **Capacity planning.** The AFR in §10 stops being a parameter you guessed and becomes a
   measurement, which tightens the spares number and feeds lesson 08's utilisation term.
3. **Loop self-monitoring.** The detection-latency histogram is how you know the health
   loop is still working. A loop that silently stops firing looks exactly like a healthy
   fleet — which is why the *absence* of remediations, sustained for longer than your
   expected inter-arrival time, deserves its own alert.

## Perspectives

**The developer / scheduler view.** A training job does not see "hardware health." It sees
a `SIGTERM` and then, some minutes later, a pod that has to come back somewhere else. What
it needs from your loop is a clean signal with enough grace period to checkpoint, and a
gang-aware scheduler that requeues the whole job rather than limping along one rank short.
Your loop's job is to make hardware failure look, from inside the job, like a planned,
resumable interruption rather than a mysterious slowdown.

**The operator / SRE view.** Keep the loop's own moving parts boring and instrumented.
`invoke_interval` tuned so detection latency is a number you can state; the cordon/drain
policy tested against the *real* PDBs your jobs ship with, because a drain that hangs
forever is worse than no automation; timeouts re-derived from your bare-metal provisioning
arithmetic rather than inherited from cloud defaults; and an alert on the loop's own
silence. The most dangerous state is a health loop everyone trusts and nobody has watched
fire in three months.

**The hardware and vendor view.** An RMA is a warranty claim, and claims are adjudicated on
evidence: Xid code, timestamp, GPU UUID and board serial, remapped-row state, and a field
diagnostic. Everything in §9 exists because someone lost weeks to a bounced ticket.
Per-SKU failure-rate tracking is separately a negotiating position — it converts anecdote
into a rate with a denominator.

**The economics view.** Every stage of this loop has a dollar translation, and lesson 08
consumes all of them. Detection latency × gang size × $/GPU-hr is the goodput bled while a
sick node stays schedulable. RMA turnaround is capacity you have paid capex for and cannot
use — which is exactly the `utilisation` term in the denominator of owned $/GPU-hr. Spares
are capital you hold specifically to shorten that window, and §10 is the calculation that
says how much is the right amount. The framing, not the YAML, is what a staff-level
interview is probing when it asks how you close the health loop.

## Real-world use cases

- **Meta — Llama 3 405B pre-training reliability data.** 16,384 H100 GPUs, 54 days, 466 job
  interruptions of which 419 were unexpected; 78% attributed to hardware, GPU failures
  including NVLink at 30.1% and HBM3 at 17.2%; an unexpected interruption roughly every
  three hours; over 90% effective training time maintained regardless. **What it shows:**
  the only large, public, per-cause failure breakdown for a GPU cluster of this size, and
  the source of the 0.082 events/GPU-year rate this lesson derives and reuses. It also
  shows the punchline: with a competent detect-and-recover loop, a cluster failing every
  three hours still delivers 90%+ effective training time. *(The Llama 3 paper is hosted on
  domains blocked by this session's egress proxy and was not fetched directly; the figures
  above are taken from multiple independent secondary reports of it —
  [Tom's Hardware](https://www.tomshardware.com/tech-industry/artificial-intelligence/faulty-nvidia-h100-gpus-and-hbm3-memory-caused-half-of-the-failures-during-llama-3-training-one-failure-every-three-hours-for-metas-16384-gpu-training-cluster)
  and [DataCenterDynamics](https://www.datacenterdynamics.com/en/news/meta-report-details-hundreds-of-gpu-and-hbm3-related-interruptions-to-llama-3-training-run/)
  — which agree on all of them.)*
- **CoreWeave — "What Is Node Lifecycle Management and Why Does It Matter for ML Training
  and Inference?"** — the source of the "up to a month" manual-detection figure, and a
  description of Fleet and Node Lifecycle Controllers owned by a dedicated Fleet
  Engineering Team that detect faults, automatically cordon, and drive hardware
  replacement. **What it shows:** that the loop in §6 is an org structure as much as a
  controller — someone owns the RMA queue, or it does not move.
  <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference>
  *(Domain blocked by this session's egress proxy; cited from the module's existing
  research notes rather than a fresh read.)*
- **Cloudflare — "Automatic Remediation of Kubernetes Nodes."** The canonical engineering
  write-up of building this loop: node-level detection feeding a custom controller that
  cordons, drains and remediates without paging a human for every bad node. **What it
  shows:** the "write your own controller" branch in §7 is a normal, small piece of
  software, not a research project.
  <https://blog.cloudflare.com/automatic-remediation-of-kubernetes-nodes/>
  *(Not fetched this session; domain blocked. Cited from the module's existing research
  notes.)*
- **`planetlabs/draino` as a maintenance cautionary tale.** The controller that every
  tutorial on this topic still recommends has a most-recent commit of **2020-12-14**
  (verified by cloning the repository on 2026-08-18). **What it shows:** in this corner of
  the ecosystem, the canonical blog-post answer and the maintained answer have diverged.
  Check `git log -1` on anything you are about to run in the remediation path — a
  controller with cordon and evict permissions on every node is not where you want
  six-year-old dependency versions. Read draino for the pattern; deploy Medik8s NHC + SNR
  (last commit 2026-07-27) or CAPI `MachineHealthCheck`.

## Worked example: one Xid, end to end

`gpu-node-07` (from lesson 05) is running rank 12 of a 64-node pre-training job.

**t+0 — the fault.** GPU 3 hits an uncontained ECC error.

```
$ journalctl -k --since -5m | grep NVRM
Aug 18 04:12:07 gpu-node-07 kernel: NVRM: Xid (PCI:0000:9d:00): 95, pid=31447,
    name=python3, Uncontained: ECC error in FBPA
Aug 18 04:12:07 gpu-node-07 kernel: NVRM: Xid (PCI:0000:9d:00): 154, GPU recovery
    action changed to: GPU Reset Required
```

Xid **95** is uncontained: per §3, other work on that GPU may be corrupt, and NVIDIA's
immediate action is `RESET_GPU`. Xid **154** is the driver telling you it has changed its
own recommended recovery action — informational, but it confirms the read.

**t+0.4 s — the Condition.** NPD's `SystemLogMonitor` is streaming `/dev/kmsg`, so this is
sub-second, not the 30 s a custom plugin would take:

```
$ kubectl describe node gpu-node-07 | sed -n '/Conditions:/,/Addresses:/p'
Conditions:
  Type                 Status  LastTransitionTime   Reason              Message
  GPUUnhealthy         True    …T04:12:07Z          UncontainedEccError PCI:0000:9d:00 Xid 95
  GPURowRemapFailure   False   …T09:02:11Z          NoRowRemapFailure   no row-remapping failure
  GPUCountMismatch     False   …T09:02:11Z          AllGPUsPresent      expected GPU count present
  Ready                True    …T09:02:14Z          KubeletReady        kubelet is posting ready
```

Note that `Ready` is still `True`. The kubelet is fine. Nothing in stock Kubernetes will
stop scheduling onto this node — that is entirely up to the loop you built.

**t+60 s — the controller acts.** `GPUUnhealthy=True` has persisted past the NHC
`duration`, and `minHealthy: 80%` is satisfied (63 of 64 nodes healthy = 98%). The
controller cordons, then evicts:

```
$ kubectl get events --field-selector involvedObject.name=gpu-node-07 --sort-by=.lastTimestamp
LAST SEEN  TYPE     REASON                OBJECT             MESSAGE
60s        Warning  NodeUnhealthy         node/gpu-node-07   GPUUnhealthy=True for 60s
58s        Normal   NodeCordoned          node/gpu-node-07   node marked unschedulable
57s        Normal   EvictionStarted       node/gpu-node-07   evicting 2 pods (1 protected by PDB)
57s        Warning  EvictionBlocked       node/gpu-node-07   pod llm-pretrain-r12: cannot evict,
                                                             would violate PDB llm-pretrain-gang
```

**The drain is blocked, and that is correct.** The PDB is `minAvailable: 64` on a 64-rank
gang. This is the system refusing to silently kill a job, exactly as §8 describes. The
gang-aware scheduler now takes over: it signals a checkpoint to all ranks, waits for the
write to complete (90 s in this fleet), marks the job for requeue, and releases the PDB.

```
120s       Normal   GangCheckpointRequested  job/llm-pretrain   checkpoint requested, 64 ranks
210s       Normal   GangCheckpointComplete   job/llm-pretrain   all ranks wrote in 88s
211s       Normal   EvictionSucceeded        node/gpu-node-07   2 pods evicted
215s       Normal   GangRequeued             job/llm-pretrain   requeued, 64 nodes requested
```

Total: 3m35s from fault to drained, of which 90 s was the checkpoint. Against §1's
arithmetic: `0.06 h × 512 GPUs × $2.50 = $76` of goodput. The same fault detected in eight
hours would have cost $10,240 — and a job restarting from a checkpoint written *before* the
uncontained error would have been the cheap outcome; the expensive one is 8 hours of
gradient updates computed on a GPU whose memory the driver could not vouch for.

**t+4 min — evidence, before anything is reset.**

```
$ ssh gpu-node-07 sudo /usr/local/bin/collect_rma_evidence.sh
/var/log/rma/gpu-node-07-20260818T041611Z.tar.gz

$ tar xzOf …tar.gz …/remapped-rows.csv
gpu_uuid, remapped_rows.failure, remapped_rows.pending, remapped_rows.correctable, remapped_rows.uncorrectable
GPU-4c2b…, No, No, 12, 1
GPU-9f31…, No, No, 3,  0
…
```

Read that carefully: **`remapped_rows.failure = No` on every GPU**, so by §4 this does
**not** meet the documented RMA criterion. One uncorrectable-error remap on GPU 3 is the
hardware doing its job. Combined with NVIDIA's `RESET_GPU` action for Xid 95, the triage
verdict is **resettable**, not RMA.

**t+5 min — reset and verify.**

```
$ ssh gpu-node-07 sudo nvidia-smi -i 3 -r
GPU 00000000:9D:00.0 was successfully reset.

$ ssh gpu-node-07 dcgmi diag -r 1
Successfully ran diagnostic for group.
+---------------------------+------------------------------------------------+
| Deployment                | Pass                                           |
| Integration               | Pass                                           |
| Hardware                  | Pass                                           |
+---------------------------+------------------------------------------------+
```

**t+35 min — burn-in and return.** The short diag is not enough to trust a node with a
64-node gang. Run the long one plus a multi-node collective, per CoreWeave's ~20-minute
full-GPU verification pattern:

```
$ ssh gpu-node-07 dcgmi diag -r 3          # ~30 min, all GPUs under load
$ mpirun -np 16 -H gpu-node-07,gpu-node-08 all_reduce_perf -b 8 -e 8G -f 2 -g 1
#      size    count   time(us)   algbw(GB/s)   busbw(GB/s)
#  8589934592  2147483648   19834      433.1        811.7
$ kubectl uncordon gpu-node-07
```

**t+35 min — record it, whatever the outcome.**

```
gpu_node_remediation_total{sku="hgx-h100-8g", batch="2026-Q1", node="gpu-node-07",
  gpu_uuid="GPU-4c2b…", signal="xid_95", action="reset"} 1
```

That single sample is what makes the *next* decision easy. If this GPU UUID appears with
`signal="xid_95"` three times in the next two weeks, the recurrence rule in §6 escalates it
from resettable to RMA — and because the evidence bundle was collected each time, the
ticket is already complete when you open it.

## Practice (feeds the deliverable)

**Build the detect → cordon → drain loop and the RMA runbook.** Deliver into
[`practice/capex-vs-cloud/`](../practice/capex-vs-cloud/README.md):

1. **NPD with at least two monitor types.** A `SystemLogMonitor` rule that matches
   `NVRM: Xid` lines in `/dev/kmsg`, and a `CustomPluginMonitor` whose script checks the
   remapped-row failure flag (or, on a GPU-less VM, a stub that reads a file so the
   contract is exercised). Trip the log rule with a synthetic Xid — `echo "NVRM: Xid
   (PCI:0000:07:00): 95, Uncontained: ECC error" | sudo tee /dev/kmsg` — and capture
   `kubectl describe node` showing the Condition. Deliberately break the custom plugin
   (rename the binary it calls) and capture the `Unknown` status, because handling
   `Unknown` is the part everyone skips.
2. **A remediation controller reacting to the Condition.** Either a Medik8s
   `NodeHealthCheck` with `escalatingRemediations`, a CAPI `MachineHealthCheck` in the
   **v1beta2** shape (`spec.checks.*`, `spec.remediation.triggerIf.*`), or ~100 lines of
   Go/Python that watch the Condition and cordon-then-evict through the eviction API.
   Whichever you choose, state the `duration`/`timeoutSeconds` you set and **derive it**
   from your provisioning arithmetic rather than copying a default.
3. **A blocked drain, on purpose.** Deploy a workload with a PDB that a drain cannot
   satisfy. Capture the `EvictionBlocked` event. Write one paragraph on why this is correct
   behaviour and what should resolve it — this is the checkpoint question the module's
   checkpoint asks in a different form.
4. **The RMA runbook.** The triage table from §3/§4 keyed on NVIDIA's own recommended
   actions (not folk severity), the evidence-collection script, and one worked example of
   the decision "reset or return" with the specific field you keyed on.
5. **The failure-rate and spares model.** Using §10's structure: your `N`, an AFR swept
   across at least three values, your assumed lead time, your FRU (baseboard or module —
   and say which, because it moves the answer by 8×), the Poisson spares count for a
   95% service level, and the lead-time sensitivity. **This model is an input to the
   lesson 08 capex model** — the spares holding is capital, and the unspared downtime is
   utilisation.

**Acceptance:** a working detect→cordon→drain demonstration on a synthetic fault, a
deliberately blocked drain with an explanation, an RMA runbook with an evidence bundle,
and a spares model with a stated service level — plus an explicit **time-to-detect target**
justified by the `detection_latency × gang_size × $/GPU-hr` arithmetic.

## Common pitfalls

- **Assuming a custom node Condition stops scheduling.** *Symptom:* the loop "works" —
  Conditions appear correctly — and jobs still land on broken nodes. *Mechanism:* the
  scheduler only reacts to `Ready` and to the well-known `node.kubernetes.io/*` taints.
  A custom Condition is inert until something translates it into a cordon or a taint.
- **Treating NPD's `Unknown` as healthy.** *Symptom:* a health check that has been broken
  for months and nobody noticed. *Mechanism:* exit code 2 (or any code that is not 0 or 1)
  is `Unknown`, and a policy that acts only on `True` treats a dead checker exactly like a
  healthy node. Alert on `Unknown` separately.
- **Losing the diagnostic message to `max_output_length`.** *Symptom:* Conditions whose
  message is cut off mid-word. *Mechanism:* the default is **80 characters**. Put the GPU
  UUID and the Xid at the front of the message, or raise the limit explicitly.
- **Rebooting before collecting evidence.** *Symptom:* an RMA that gets bounced, and the
  fault does not reproduce. *Mechanism:* a reboot clears the kernel ring buffer that holds
  the Xid line and resets volatile ECC counters. Run `nvidia-bug-report.sh` and capture the
  Xid *first*; the reset is the second step, always.
- **Draining with a PDB that lies about the gang.** *Symptom:* a 64-rank job dies during a
  routine remediation. *Mechanism:* `minAvailable: 63` on a synchronous 64-rank job permits
  exactly the eviction that kills it. Either express the true constraint
  (`minAvailable: 64`, so the drain blocks and something job-aware decides) or make the job
  genuinely elastic. Do not split the difference.
- **Forgetting `unhealthyPodEvictionPolicy: AlwaysAllow`.** *Symptom:* a drain that hangs
  forever on a node whose pods are already crashed. *Mechanism:* with the default
  `IfHealthyBudget`, not-ready pods still count against the budget, so a partially-crashed
  job blocks its own cleanup.
- **Inheriting cloud timeouts on bare metal.** *Symptom:* healthy nodes being remediated
  during a rack bring-up, producing a re-provisioning loop. *Mechanism:* CAPI's
  `nodeStartupTimeoutSeconds` defaults to 600, and a bare-metal GPU node spends five to
  eight minutes on firmware POST before it can even PXE. Re-derive every timeout from
  lesson 05's arithmetic.
- **Remediating during a storm.** *Symptom:* a driver bug or a monitoring outage takes out
  the whole fleet because the loop obediently cordoned every node. *Mechanism:* no
  `minHealthy` / `maxUnhealthy` guard, or no cooldown after the guard clears. Set both, and
  set the cooldown longer than your node-status update lag.
- **Treating Xid 63 as a warning and Xid 79 as an RMA.** *Symptom:* healthy GPUs pulled
  from service and genuinely dead ones left in. *Mechanism:* 63 reports a *successful*
  hardware repair (NVIDIA: `IGNORE`); the failure code is 64. And 79's documented immediate
  action is to restart the host, because GPUs fall off the bus for riser, power and thermal
  reasons that are not the GPU.
- **Deploying a six-year-old controller with cluster-wide evict permissions.** *Symptom:*
  discovered during an audit, or during an upgrade that breaks it. *Mechanism:* draino's
  last commit is 2020-12-14. `git log -1` before you deploy anything into the remediation
  path.

## Self-check

**(a) How do you drain a node running a 512-GPU synchronous training job without losing the
job, and what should happen if the PDB blocks you?**
**Answer:** Cordon first (`spec.unschedulable` → the
`node.kubernetes.io/unschedulable:NoSchedule` taint) so nothing new lands, then evict
through the **eviction API**, which is the only path that consults PodDisruptionBudgets;
a raw pod delete bypasses them. `terminationGracePeriodSeconds` must exceed the job's
checkpoint write time (30 s is the default and is far too short for a large model), and
`kubectl drain --timeout` must exceed the grace period. Set
`unhealthyPodEvictionPolicy: AlwaysAllow` on the PDB so already-crashed pods do not block
their own cleanup. If the gang's PDB blocks the eviction, **that is correct** — a
synchronous gang cannot survive losing a rank, and a PDB that permits the eviction is
lying. The resolution is a gang-aware scheduler (Volcano, Kueue, JobSet) that signals a
checkpoint to all ranks, waits for the write, marks the job for requeue and then releases
the budget. Never `--force`.

**(b) What is your time-to-detect target, and how do you justify it in dollars?**
**Answer:** Minutes, and specifically **≈31 seconds worst case** for a
`CustomPluginMonitor` (default `invoke_interval` 30 s + NPD's 1 s condition-manager tick +
one API write) or sub-second for a `SystemLogMonitor` streaming `/dev/kmsg`, plus the
remediation controller's `duration` grace window. The justification is
`goodput lost = detection_latency × gang_size × $/GPU-hr`, because a single sick node in a
synchronous job degrades the *entire* gang: at 512 GPUs and $2.50/GPU-hr, five minutes
costs $107, eight hours costs $10,240, and CoreWeave's "up to a month" manual worst case
costs $921,600. The second-order argument is stronger: Young/Daly checkpointing bounds the
cost of faults that *crash*, so a 4× worse node MTBF only doubles lost time — but it does
nothing for a node that silently degrades and keeps running. Fast detection is the only
defence against the failure mode checkpointing cannot see.

**(c) NPD ships four monitor types. Name them, and say what each catches that the others do
not.**
**Answer:** **SystemLogMonitor** applies regex rules to a log source (the `kmsg` plugin on
`/dev/kmsg`, `filelog` on a file, or journald) — it is the only one with sub-second latency,
and it is how you catch an `NVRM: Xid` line. **CustomPluginMonitor** runs a script or
binary on an interval and maps its exit code (0 OK, 1 NonOK, 2 Unknown) to a Condition or
Event — it is how you express any check that requires querying state rather than reading a
log, such as the remapped-row failure flag or an expected-GPU-count assertion.
**SystemStatsMonitor** gathers host resource statistics (disk, memory, CPU) into conditions
and metrics — it catches the non-GPU exhaustion that still stops a job, such as the NVMe
scratch tier filling. **HealthChecker** checks named systemd units — kubelet, containerd,
`nvidia-persistenced` — so a node with eight perfect GPUs and a crash-looping container
runtime is flagged rather than looking healthy. A real GPU-node config uses at least three;
relying only on log matching misses everything that does not print.

**(d) A GPU reports one uncorrectable-error row remap and `remapped_rows.failure = No`. RMA
or not? What if the flag were `Yes`?**
**Answer:** **Not an RMA.** A remap that succeeded is the hardware doing exactly what row
remapping exists for — Ampere-and-later parts carry a budget of up to 512 remappings, and
the corresponding Xid 63 is documented by NVIDIA as `IGNORE`. Track it (a device consuming
remaps quickly is worth watching, and DCGM fields 385–389 tell you how many banks still
have spares, with 389 counting banks with **none** left), but leave it in service. If the
**failure** flag (`remapped_rows.failure = Yes`, DCGM field 395
`DCGM_FI_DEV_ROW_REMAP_FAILED`) were set, that *is* the documented RMA criterion: NVIDIA's
memory-error-management guidance states the criterion is met when the row-remapping failure
flag is set and validated by the field diagnostic. The adjacent state to keep separate is
`remapped_rows.pending` (field 396), which means a repair is queued but needs a GPU reset
or host reboot to apply — scheduled maintenance, not a return. Note what is *not* a
criterion: a count of double-bit errors. "RMA after N DBEs" is folklore; the published,
measurable trigger is the failure flag.

**(e) What does NPD add over a liveness probe, and what does it deliberately not do?**
**Answer:** A liveness probe is **per-pod** and its only action is restarting a container —
if the node's hardware is broken, the restarted container lands on the same broken
hardware. NPD watches **node-level** signals (kernel log, custom scripts querying NVML or
DCGM, host resource statistics, systemd unit health) and publishes a durable
**NodeCondition** that survives pod churn. What it deliberately does *not* do is act:
NPD never cordons, drains or reboots anything. That separation is the architecture — the
Condition is a data structure, not a call, so you can swap the remediation controller
(Medik8s NHC, CAPI `MachineHealthCheck`, your own) without touching detection, run two
policies against one signal, and unit-test a policy by writing a Condition by hand. The
corollary in the other direction is the trap in pitfall one: because a Condition is only
data, nothing stops scheduling until something turns it into a cordon or a taint.

**(f) How many spare baseboards should a 512-GPU fleet hold, and what actually decides the
answer?**
**Answer:** Model demand during the RMA lead time as Poisson with
`λ = N × AFR × L/365`, and pick the smallest `s` with `P(demand ≤ s) ≥ SL`. At `N = 512`,
`AFR = 3%`, `L = 21 days`, `λ = 0.884`, and the cumulative Poisson gives 0.413 / 0.778 /
0.939 / 0.987 for `s = 0..3`, so **`s = 3`** for a 95% service level; sweeping AFR from 2%
to 5% moves it from 2 to 4. Justify it against the alternative: at AFR 3% that is ~15.4
events/year, each costing `8 GPUs × 21 days × 24 h = 4,032` GPU-hours (≈$10,080 at
$2.50/GPU-hr) if unspared, so ~$155K/year of lost capacity against ~$65K/year of holding
cost on three $180K baseboards at a 12% cost of capital. **What actually decides it is lead
time, not AFR** — halving `L` from 21 to 10 days drops `λ` to 0.422, drops the spares
requirement to 2, and halves the loss per unspared event, which is the numeric argument for
paying more for a guaranteed-turnaround support contract. And confirm the FRU first: if
your vendor replaces individual SXM modules rather than whole 8-GPU baseboards, both the
capacity lost per event and the spares capital fall by roughly 8×.

## Connections & what's next

This lesson closes the loop lesson 05 opened. Provisioning builds a node forward; this
lesson tears one down safely and, when the hardware is genuinely bad, sends it back through
lesson 05's exact pipeline — `BareMetalHost` to `available`, clean, re-image, rejoin — at
the makespan you computed there. The CAPI `MachineHealthCheck` path makes that literal: an
unhealthy `Machine` is deleted and the infrastructure provider re-provisions it, so lessons
04, 05 and 06 describe one continuous machine lifecycle from three angles — declarative
shape, forward bring-up, backward remediation.

Two threads run forward. The **eviction and checkpoint mechanics** in §8 depend entirely on
how fast a checkpoint can be written, which is a storage-bandwidth question and therefore
lesson 07's: `δ` in the Young/Daly formula is a number you size a filesystem for, and the
90 seconds used in the worked example is not a constant, it is a design choice with a price
tag. And the **spares model, the AFR measurement and the detection-latency SLO** in §10 and
§11 are three direct inputs to lesson 08's capex model — spares are held capital, RMA
turnaround is a subtraction from the utilisation term in the denominator of owned
$/GPU-hr, and a measured AFR replaces a guessed one.

Next, lesson 07 moves from *keeping nodes healthy* to *keeping healthy GPUs fed*: the
aggregate bandwidth a fleet needs so that six-figure silicon is not idling on I/O. A
different subsystem, the same underlying accounting — GPU-hours paid for and not used.

## References & further reading

**Primary sources (verified against upstream source this session)**

- **Node Problem Detector** — <https://github.com/kubernetes/node-problem-detector> —
  read 2026-08-18 at commit `7bff5a4`. Source for the four monitor types; the
  `CustomPluginConfig` JSON schema and its defaults in
  `pkg/custompluginmonitor/types/config.go` (`invoke_interval` 30 s, `timeout` 5 s,
  `max_output_length` **80**, `concurrency` 3,
  `enable_message_change_based_condition_update` false, `skip_initial_status` false); the
  exit-code contract (`OK` 0, `NonOK` 1, `Unknown` 2) in `pkg/custompluginmonitor/types/types.go`;
  the shipped configs in `config/` including the `kmsg`-plugin kernel monitor
  (`logPath: /dev/kmsg`, `lookback: 5m`, `bufferSize: 10`); and the condition-manager
  timings in `pkg/exporters/k8sexporter/condition/manager.go` (`updatePeriod` 1 s,
  `resyncPeriod` 10 s) with `--k8s-exporter-heartbeat-period` defaulting to 5 minutes.
- **NVIDIA DCGM** — <https://github.com/NVIDIA/DCGM> — read 2026-08-18. Source for the
  `/dev/kmsg` Xid parsing regex and the default watched-Xid set `{79, 119, 120}` with a
  5 ms poll interval (`dcgmlib/src/DcgmKmsgReader.cpp`); the row-remapping and page-
  retirement field IDs 385–396 and `DCGM_FI_DEV_XID_ERROR` = 230 (`dcgm_fields.h`); the
  health-watch system bitmask (`dcgmlib/dcgm_structs.h`); and the diagnostic levels
  `DCGM_DIAG_LVL_SHORT` 10 / `MED` 20 / `LONG` 30 / `XLONG` 40 that `dcgmi diag -r` selects.
- **NVIDIA Xid error catalog** — <https://docs.nvidia.com/deploy/xid-errors/> —
  *`docs.nvidia.com` is blocked by this session's egress proxy and was not fetched.* The Xid
  table in §3, including NVIDIA's own `ImmediateResolution` values, is verified against a
  machine-generated mirror of NVIDIA's catalog in `leptonai/gpud`
  (`components/accelerator/nvidia/xid/catalog_generated.go`, generated 2025-10-29 from
  `docs.nvidia.com/deploy/xid-errors/analyzing-xid-catalog.html`; repository read
  2026-08-18). **Corrections applied from this source:** Xid 63 is a remapping *event* with
  a documented action of `IGNORE`, not a warning sign — the failure code is 64; Xid 79's
  documented immediate action is `RESTART_BM` (restart the host), not an automatic RMA;
  Xid 94 (contained) is `RESTART_APP` while 95 (uncontained) is `RESET_GPU`; and the
  "RMA if XID 48 recurs more than once a week" rule that appeared in an earlier version of
  this lesson is not published by NVIDIA and has been removed.
- **NVIDIA GPU Memory Error Management (row remapping and RMA policy)** —
  <https://docs.nvidia.com/deploy/a100-gpu-mem-error-mgmt/> — *blocked; not fetched.* The
  512-remap budget, the four bank-availability buckets, and the RMA criterion ("met when the
  row-remapping failure flag is set and validated by the field diagnostic") are verified
  against `leptonai/gpud`'s `QualifiesForRMA()` implementation and its inline quotations of
  that document (`components/accelerator/nvidia/remapped-rows/remapped_rows.go`), which
  also cross-references NVIDIA's own DCGM validation implementation. Corroborated by search
  extracts of the NVIDIA page.
- **Medik8s Node Healthcheck Operator and Self-Node-Remediation** —
  <https://github.com/medik8s/node-healthcheck-operator> and
  <https://github.com/medik8s/self-node-remediation> — both read 2026-08-18, last commits
  2026-07-27. Source for the `NodeHealthCheck` spec: the default `unhealthyConditions` of
  `Ready=False/Unknown` for 300 s, the mutually exclusive `minHealthy`/`maxUnhealthy`,
  `stormCooldownDuration`, `escalatingRemediations` with `order` and `timeout`, and
  `healthyDelay`; and for SNR's `remediationStrategy` enum
  (`Automatic` default, `ResourceDeletion`, `OutOfServiceTaint`, the last applying the
  well-known `node.kubernetes.io/out-of-service` taint).
- **Cluster API** — <https://github.com/kubernetes-sigs/cluster-api> — read 2026-08-18 at
  commit `20e5ac6`, `api/core/v1beta2/machinehealthcheck_types.go`. **Correction applied:**
  the `MachineHealthCheck` API was restructured for v1beta2 — `unhealthyConditions` moved to
  `spec.checks.unhealthyNodeConditions` with `timeoutSeconds` instead of a duration string,
  `nodeStartupTimeout` became `spec.checks.nodeStartupTimeoutSeconds` (default 600, `0`
  disables), and `maxUnhealthy`/`unhealthyRange` became
  `spec.remediation.triggerIf.unhealthyLessThanOrEqualTo` / `unhealthyInRange`. Manifests
  written against the v1beta1 shape shown in most tutorials will not apply.
- **`planetlabs/draino`** — <https://github.com/planetlabs/draino> — cloned and inspected
  2026-08-18. **Correction applied:** last commit **2020-12-14**. It is the origin of the
  cordon-and-drain-on-Condition pattern and is worth reading for that, but it is not
  maintained tooling and should not be presented as a current option.
- **NVIDIA RMA Process and GPU Debug Guidelines** —
  <https://docs.nvidia.com/deploy/rma-process/> and
  <https://docs.nvidia.com/deploy/gpu-debug-guidelines/> — *blocked; not fetched and not
  relied upon for any specific figure.* The evidence-package contents in §9
  (`nvidia-bug-report.sh` as the primary requested artifact, DCGM diagnostic logs, and the
  field diagnostic as the authoritative pre-RMA tool) come from search extracts of these
  documents, corroborated by Dell's DCGM install-and-diagnose knowledge-base article.
- **AKS GPU health monitoring** —
  <https://learn.microsoft.com/en-us/azure/aks/gpu-health-monitoring> — a managed-cloud
  reference implementation of this lesson's stage 1: NPD on GPU node pools with a
  `GPUXIDErrors` log check and a `GPUMissing` enumeration check. *Not fetched this session
  (domain blocked); cited from the module's existing research notes as an existence proof
  of the pattern, not for any specific figure.*

**Real-world engineering**

- **Meta — Llama 3 405B pre-training reliability figures** — 16,384 H100s over 54 days;
  466 interruptions (47 planned, 419 unexpected); 78% hardware; GPU-with-NVLink 30.1%,
  HBM3 17.2%; >90% effective training time. *The paper itself is on a blocked domain and
  was not fetched; figures taken from two independent secondary reports that agree —*
  <https://www.tomshardware.com/tech-industry/artificial-intelligence/faulty-nvidia-h100-gpus-and-hbm3-memory-caused-half-of-the-failures-during-llama-3-training-one-failure-every-three-hours-for-metas-16384-gpu-training-cluster>
  *and*
  <https://www.datacenterdynamics.com/en/news/meta-report-details-hundreds-of-gpu-and-hbm3-related-interruptions-to-llama-3-training-run/>.
  **What it shows:** the derivation basis for 0.082 GPU/HBM interruptions per GPU-year and
  the demonstration that a cluster failing every three hours can still deliver 90%+
  effective training time.
- **CoreWeave — "What Is Node Lifecycle Management…"** —
  <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference>
  — the "up to a month" manual-detection figure and the Fleet/Node Lifecycle Controllers.
  *Domain blocked this session; cited from the module's existing research notes.*
- **Cloudflare — "Automatic Remediation of Kubernetes Nodes"** —
  <https://blog.cloudflare.com/automatic-remediation-of-kubernetes-nodes/> — the canonical
  build-your-own-controller write-up. *Domain blocked this session; cited from the module's
  existing research notes.*
- **Nscale — "Inside Fleet Operations: Automating the GPU lifecycle"** —
  <https://www.nscale.com/blog/fleet-operations> — independent corroboration from a second
  GPU neocloud that burn-in, firmware validation and auto-remediation before workload
  impact are an industry norm. *Domain blocked this session; cited from the module's
  existing research notes.*

**Deeper dives**

- **`leptonai/gpud`** — <https://github.com/leptonai/gpud> — an open-source GPU health
  daemon whose components map almost one-to-one onto §2's signal layer (xid, sxid, ecc,
  remapped-rows, hw-slowdown, clock-speed, nvlink, infiniband, fabric-manager). Read
  `components/accelerator/nvidia/remapped-rows/` for a production implementation of the
  RMA criterion, and the generated Xid catalog for the full code list.
- **Kubernetes eviction API and PodDisruptionBudget reference** —
  <https://kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/> — for the exact
  eviction semantics, `unhealthyPodEvictionPolicy`, and the interaction with
  `terminationGracePeriodSeconds`. *`kubernetes.io` is blocked by this session's egress
  proxy; listed as optional depth and not relied upon — the behaviours described in §8 are
  from the controller implementations cited above.*
