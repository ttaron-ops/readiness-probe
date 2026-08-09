---
lesson: "10.6"
title: "Hardware health: closed-loop remediation and RMA"
module: "10"
concept: "Hardware health: closed-loop remediation and RMA"
status: not-started
est_time: "5h"
artifacts: []
---
# 10.6 · Hardware health: closed-loop remediation and RMA

> **Concept.** Close the health loop — NPD custom plugins → node Conditions → automated cordon/drain → RMA ticket → per-SKU failure-rate tracking — so a bad GPU node is detected in minutes, not a month.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Why this matters

CoreWeave has said the quiet part out loud: **a faulty node can take up to a
month to detect if you rely on humans.** At fleet scale that number is a
business, not a footnote. A single degraded H100 node inside a 500-GPU gang
doesn't fail cleanly — it throttles, throws correctable ECC storms, or drops an
NVLink, and the *entire* synchronous job runs slow or crash-restarts from its
last checkpoint. Every hour that node stays `schedulable` you burn GPU-hours
across the whole gang and you delay the **RMA clock**, which is where warranty
dollars actually live.

Do the arithmetic. H100 capacity runs on the order of a few dollars per
GPU-hour. A 500-GPU job stalled or repeatedly restarting because one node is
silently sick, undetected for weeks, is *hundreds of thousands of dollars* of
lost goodput — plus the missed warranty window if the RMA is filed late. This is
the differentiator for a senior platform role at a GPU neocloud: **automated,
closed-loop remediation is survival, and the cost/observability framing is the
thing that gets you hired.** Manual triage does not scale past a rack.

## What's new here

Lesson 05 gave you the *concepts*: **XID errors** (the NVIDIA GPU error codes),
**Node Problem Detector** as an idea, and **automated cordon/drain** in the
abstract. You know *what* an XID is and *that* NPD exists. We do **not**
re-teach either.

What's new is **wiring them into a closed loop that runs without a human**:

- **NPD custom plugins → node Conditions** as concrete manifests, not concepts.
- **The remediation controller** (draino / Medik8s NHC + self-node-remediation /
  Cluster API `MachineHealthCheck` / a Cloudflare-style custom controller) that
  *acts* on those Conditions — cordon, then drain respecting PDBs.
- **The RMA workflow**: bad hardware → ticket → vendor turnaround → the node
  re-enters the provisioning pipeline from lesson 05.
- **Fleet feedback**: per-SKU failure-rate tracking so you know *which* hardware
  batch is killing your goodput, and can push it back on the vendor.

Detection (NPD/XID) was lesson 05. **This lesson is the loop that closes around
it.**

## Core notes

### The closed loop

```
  ┌─────────────┐   condition    ┌──────────────┐   cordon+drain   ┌──────────┐
  │ NPD custom  │ ─────────────▶ │ Remediation  │ ───────────────▶ │  Node    │
  │ plugin      │  GPUXid=True   │ controller   │  (respect PDBs)  │ isolated │
  │ (XID/DCGM)  │                │ (draino/NHC) │                  └────┬─────┘
  └─────────────┘                └──────────────┘                       │
        ▲                                                               ▼
        │                                              ┌────────────────────────┐
        │  re-provision (lesson 05)                    │ Triage: soft vs hard   │
        │  ◀───────── good hardware ──────────────────┤ • reboot/reset → back   │
        │                                              │ • hard fault → RMA      │
        │                                              └───────────┬────────────┘
        │                                                          ▼
        │                                          ┌──────────────────────────┐
        └──────────────── RMA closed ◀─────────────│ RMA ticket + per-SKU     │
                                                   │ failure-rate dashboard    │
                                                   └──────────────────────────┘
```

Five stages: **detect → isolate → drain → decide (reboot vs RMA) → track**. The
only manual touch is physically swapping the RMA'd hardware; everything else is a
controller.

### Stage 1 — NPD custom plugin → node Condition

NPD ships two monitor types. A **system-log-monitor** matches regexes against
the kernel log / journald and can flag an **XID** line directly (this is how you
turn lesson 05's XID knowledge into a signal). A **custom-plugin-monitor** runs
a script and maps exit codes (0 = OK, 1 = problem) to a Condition or Event. NPD
itself takes **no remediation action** — it only surfaces a permanent
`NodeCondition` and/or an Event. That separation is the point: detection and
remediation are decoupled.

Custom-plugin monitor config (`gpu-xid-monitor.json`):

```json
{
  "plugin": "custom",
  "pluginConfig": { "invoke_interval": "30s", "timeout": "10s",
                    "concurrency": 1 },
  "source": "gpu-xid-custom-plugin-monitor",
  "conditions": [
    { "type": "GPUXidError", "reason": "GPUHealthy",
      "message": "No fatal GPU XID observed" }
  ],
  "rules": [
    { "type": "permanent", "condition": "GPUXidError",
      "reason": "FatalGPUXid",
      "path": "/config/plugin/check_gpu_xid.sh" }
  ]
}
```

The check script (grep dmesg for fatal XIDs, or query DCGM):

```bash
#!/bin/bash
# exit 1 => problem (sets GPUXidError=True); exit 0 => OK
# Fatal XIDs: 79 (GPU fell off bus), 48 (double-bit ECC), 74 (NVLink), ...
if dmesg | grep -Eq 'NVRM: Xid.*: (79|48|74|94|95),'; then
  echo "Fatal XID detected"; exit 1
fi
exit 0
```

Result: `kubectl describe node` shows `GPUXidError=True`. In production you'd
prefer DCGM/`dcgmi health` as the source over raw dmesg, but the Condition
surface is identical.

### Stage 2–3 — remediation controller: cordon then drain

A separate controller watches for the Condition and acts. Options, from simplest
to most integrated:

- **draino** (`planetlabs/draino`) — watches node Conditions; on match it
  cordons and drains. Pair with **cluster-autoscaler** so the drained,
  eventually-deleted node is replaced. The classic Cloudflare pattern.
- **Medik8s Node Healthcheck Operator (NHC) + Self-Node-Remediation** — CRD
  `NodeHealthCheck` selects nodes + unhealthy Conditions and triggers a
  remediation template (reboot/reprovision); the modern, more controllable
  successor pattern.
- **Cluster API `MachineHealthCheck`** — if nodes are CAPI `Machine`s
  (lesson 05, CAPM3/CAPT), an unhealthy Condition past a timeout deletes the
  `Machine`, and the infra provider **re-provisions** it — the loop closes into
  lesson 05 automatically.
- **A custom controller** — what Cloudflare built and described: react to node
  Conditions, cordon/drain, and file the ticket. ~200 lines of Go for full
  control of the policy.

draino wiring (flag form):

```
draino --node-label-expr='feature.node.kubernetes.io/pci-10de.present=true' \
       --evict-daemonset-pods=false \
       GPUXidError    # the NodeCondition type(s) to act on
```

**Drain must respect PodDisruptionBudgets.** The eviction API honors PDBs and
`terminationGracePeriodSeconds`; the training operator sets these so eviction is
graceful, not a `kill -9` (see self-check a).

### Stage 4 — triage: reboot vs RMA

Not every fault is a return. Classify:

- **Soft / transient** — single correctable ECC, a recoverable XID (e.g. a
  transient bus hiccup), thermal event that cleared. Action: GPU reset /
  node reboot; if it comes back clean, re-provision (lesson 05) and return to
  service. Track it — a node that "recovers" three times this week is really an
  RMA.
- **Hard fault** — GPU off the bus (XID 79), double-bit/uncorrectable ECC
  (XID 48/94/95), NVLink/NVSwitch down (XID 74), row-remapping exhausted. Action:
  the node is out; open an **RMA**.

An **RMA runbook** should capture, per event: node name + serial, GPU
UUID/board serial (`nvidia-smi -q`), the XID and dmesg excerpt, DCGM health
dump, the goodput lost (GPU-hours the node was in a job), and the warranty
identifiers the vendor needs. The ticket creation is automatable — the
controller that drained the node can `POST` to your ticketing API with that
payload attached.

### Stage 5 — per-SKU failure-rate tracking

The fleet signal that turns incidents into procurement leverage: **failures per
SKU per unit time.** Emit a metric on every remediation —
`gpu_node_remediation_total{sku="HGX-H100-8G", batch="2024-Q3", xid="79"}` — and
build a dashboard of failures-per-1000-node-hours by SKU/batch/DC. This tells you
which batch is a lemon (push it back on the vendor / recover warranty), feeds
capacity planning (over-provision the flaky SKU), and is exactly the
cost/observability story that differentiates a senior GPU-platform engineer.

### Time-to-detect is the whole game

Your **time-to-detect target is minutes** (an NPD `invoke_interval` of 30s plus
controller reaction is single-digit minutes), against CoreWeave's **one-month**
manual worst case. The gap is pure goodput: in a synchronous 500-GPU job, one
sick node degrades or crashes the *entire* gang. A month of a bad node silently
poisoning restarts is hundreds of thousands of GPU-hours of loss plus a blown
warranty window. Minutes-to-detect + auto-drain + auto-RMA converts that from a
recurring catastrophe into a logged, costed event.

## Worked example: synthetic XID → drain → RMA

1. **Deploy NPD** as a DaemonSet with the custom-plugin config above (mount the
   config + script, `hostPath` `/dev/kmsg` or run the check against dmesg).
2. **Inject a synthetic fault** on one node — write a fake fatal XID to the
   kernel ring buffer so the check trips without real broken hardware:
   ```bash
   echo "NVRM: Xid (PCI:0000:07:00): 79, GPU has fallen off the bus." \
     | sudo tee /dev/kmsg
   ```
3. **Observe the Condition.** `kubectl describe node gpu-node-07` now shows
   `GPUXidError=True reason=FatalGPUXid`. NPD also emits an Event.
4. **draino reacts** — cordons `gpu-node-07`, then drains it honoring PDBs and
   grace periods; the training pods receive SIGTERM and checkpoint (self-check a).
5. **Triage + RMA.** XID 79 is a hard fault → the controller opens an RMA ticket
   with node serial, GPU UUID, XID, and lost-goodput fields; emits
   `gpu_node_remediation_total{sku=...,xid="79"}`.
6. **Close the loop.** Good replacement hardware re-enters the lesson-05
   pipeline (`BareMetalHost` back to `available` → re-image → rejoin). The
   per-SKU dashboard ticks up one failure for that batch.

## Practice (feeds the deliverable)

**Build the NPD → cordon/drain demo + the RMA/failure-rate runbook.** Deliver
into `practice/capex-vs-cloud/`:

1. **NPD DaemonSet + custom plugin** that flags a **synthetic GPU fault** — the
   `check_gpu_xid.sh` script above, tripped by writing a fake XID to `/dev/kmsg`
   (or a custom monitor). Show `GPUXidError=True` on the node.
2. **A remediation controller** — draino configured on `GPUXidError`, *or* a
   ~100-line Go/Python controller that watches the Condition and cordons+drains
   with the eviction API (respecting PDBs). Capture the cordon + drain events.
3. **An RMA runbook** — the triage table (soft/reboot vs hard/RMA with example
   XIDs), the auto-generated ticket payload fields, and a **per-SKU
   failure-rate dashboard** sketch (the metric + the PromQL/panel).

**Acceptance:** a working NPD→cordon/drain demo on a synthetic fault, plus an
RMA + per-SKU failure-rate runbook, both checked into the deliverable, with the
**time-to-detect target** and its goodput-cost justification stated explicitly.

## Self-check

**(a) How do you drain a node running a 500-GPU distributed training job without
losing the job?**
**Answer:** You never `kill -9` it. Drain via the **eviction API**, which
respects the job's **PodDisruptionBudget** and `terminationGracePeriodSeconds`.
The grace period must be long enough for the training operator to catch SIGTERM
and **checkpoint** (lesson 08) — async checkpoint to node-local NVMe + remote
store — so the job **preempts and requeues** (lesson 06) and resumes from the
last checkpoint rather than from zero. Because the job is synchronous/gang-
scheduled, one node leaving pauses the whole gang, so the drain must be
coordinated with the scheduler (Volcano/JobSet/Kubeflow): signal-checkpoint,
then evict, then let the gang restart on a healthy node. Cordon first so nothing
new lands while it drains.

**(b) What's your time-to-detect target and why does a one-month detection lag
(CoreWeave's number) cost so much?**
**Answer:** Target is **minutes** — NPD `invoke_interval` ~30s plus controller
reaction is single-digit minutes. It costs so much because in a synchronous
500-GPU job a single sick node degrades or crash-restarts the **entire gang**;
every hour it stays schedulable burns GPU-hours across all 500 GPUs and delays
the RMA/warranty clock. A month undetected is hundreds of thousands of dollars of
lost goodput (H100 capacity at a few $/GPU-hr) plus a potentially blown warranty
window — the exact loss automated remediation exists to prevent.

**(c) What does NPD add over a plain liveness probe?**
**Answer:** A liveness probe is **per-pod** and only restarts a container — it
sees nothing at the node/hardware layer and takes no cluster action. If a GPU
falls off the bus, restarting the pod just re-lands it on the same broken node.
NPD watches **node-level** signals (kernel log/XID, DCGM, custom scripts) and
publishes a **permanent NodeCondition** the scheduler and remediation
controllers act on — cordon, drain, RMA. It marks the *node* bad (durably), not
the pod, and decouples detection from remediation so a controller can close the
loop.

## Resources

1. **Node Problem Detector** — <https://github.com/kubernetes/node-problem-detector>
   — system-log-monitor vs custom-plugin-monitor, Condition/Event exporters, and
   config format; the deep reference for stage 1.
2. **Cloudflare — "Automatic Remediation of Kubernetes Nodes"** —
   <https://blog.cloudflare.com/automatic-remediation-of-kubernetes-nodes/> —
   the canonical NPD → draino → auto-remediation closed-loop write-up.
3. **AKS GPU health monitoring** —
   <https://learn.microsoft.com/en-us/azure/aks/gpu-health-monitoring> — a
   production DCGM/XID-driven GPU-node health + remediation reference to compare
   your loop against.
