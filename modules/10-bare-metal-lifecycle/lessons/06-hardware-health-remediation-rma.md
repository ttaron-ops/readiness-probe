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
sources: 9
---
# 10.6 · Hardware health: closed-loop remediation and RMA

> **Concept.** Close the health loop — NPD custom plugins → node Conditions → automated cordon/drain → RMA ticket → per-SKU failure-rate tracking — so a bad GPU node is detected in minutes, not a month.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 05 built the pipeline that turns a powered-on server into a labeled,
`Ready`, driverless Kubernetes node — the netboot-to-Ready sequence, run once
per node, forward in time. This lesson is that pipeline's *mirror*: what
happens when a node that already made it through provisioning goes bad. A
faulty GPU doesn't announce itself with a clean error and an alert page — it
throttles, throws correctable ECC storms, drops an NVLink, or silently
falls off the PCIe bus, and the failure surfaces as a slow or crash-restarting
job long before anyone notices the node itself is sick. This lesson closes the
loop: detect the fault, isolate the node, decide whether it needs a reboot or a
physical swap, and — if it's a hardware failure — send the node back through
lesson 05's exact pipeline once the replacement part is in. Provisioning and
remediation are two ends of one lifecycle; this lesson is where that lifecycle
becomes circular instead of a dead end.

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
thing that gets you hired.** Manual triage does not scale past a rack. It's
also not a hypothetical org problem — CoreWeave describes a dedicated **Fleet
Engineering Team** that owns exactly this: Fleet and Node Lifecycle Controllers
that detect GPU/hardware faults, cordon unhealthy nodes automatically, and
drive the hardware-replacement process, because at fleet scale a human staring
at dashboards cannot keep up with the fault rate of tens of thousands of GPUs.

## What's new here (calibration)

Lesson 05 gave you the *concepts*: **XID errors** (the NVIDIA GPU error codes),
**Node Problem Detector** as an idea, and **automated cordon/drain** in the
abstract. You know *what* an XID is and *that* NPD exists. We do **not**
re-teach either.

What's new is **wiring them into a closed loop that runs without a human**:

- **NPD custom plugins → node Conditions** as concrete manifests, not concepts
  — and the full set of monitor types NPD actually ships, not just the two
  most commonly cited.
- **The remediation controller** (draino / Medik8s NHC + self-node-remediation /
  Cluster API `MachineHealthCheck` / a Cloudflare-style custom controller) that
  *acts* on those Conditions — cordon, then drain respecting PDBs.
- **The RMA workflow**: bad hardware → ticket → vendor turnaround → the node
  re-enters the provisioning pipeline from lesson 05.
- **Fleet feedback**: per-SKU failure-rate tracking so you know *which* hardware
  batch is killing your goodput, and can push it back on the vendor.

Detection (NPD/XID) was introduced in lesson 05's calibration. **This lesson is
the loop that closes around it.**

## Core concepts

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

NPD actually ships **four** monitor types, not just the two most commonly
cited. Knowing all four matters because they cover different layers of a
GPU node's health, and a mature health story usually uses more than one:

| Monitor | What it watches | Typical GPU-relevant use |
|---|---|---|
| **SystemLogMonitor** | Regex rules against the kernel log / journald | Match an **XID** line directly in `dmesg`/`journalctl` — turns lesson 05's XID knowledge into a signal without any external tooling |
| **CustomPluginMonitor** | Runs a script/binary on an interval, maps exit code → Condition/Event | Run `dcgmi health` or a custom check and map 0/1 to `GPUXidError` — the pattern this lesson's worked example builds by hand |
| **SystemStatsMonitor** | Host-level resource stats (disk, memory, CPU, so-called "OOM" stats) rolled into conditions | Catch host-level resource exhaustion that isn't GPU-specific but still degrades a training job (e.g. disk pressure on the node hosting NVMe scratch from lesson 07) |
| **HealthChecker** | systemd service health (is the unit active, restarting, crash-looping) | Watch `kubelet`, `containerd`/`docker`, and `ntp`/`chrony` — a GPU node with a flapping container runtime looks "healthy" to a naive XID check but is still unschedulable in practice |

NPD itself takes **no remediation action** on any of the four — it only
surfaces a permanent `NodeCondition` and/or an Event. That separation is the
point: detection and remediation are decoupled, so you can swap the
remediation controller (draino, NHC, a custom one) without touching how faults
are detected.

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
surface is identical. AKS's own GPU health monitoring builds essentially this
pattern for you: it runs NPD on GPU node pools with a `GPUXIDErrors` check that
watches kernel logs for XID lines and sets a Condition — the managed-cloud
version of the script above, plus a `GPUMissing` check that catches a
mismatch between expected and detected GPU count (a firmware/enumeration
failure from lesson 05, not a runtime one).

### Reading an XID: the codes that matter for triage

XID codes are numeric error IDs the NVIDIA driver writes to the kernel ring
buffer (`NVRM: Xid …`) when the GPU hits a fault the driver can't silently
recover from. NVIDIA's own XID Reporting reference documents the full list;
the ones a GPU-fleet operator triages on most are:

| XID | Meaning | Severity |
|---|---|---|
| **48** | Uncorrectable double-bit ECC error (DBE) | Hard — RMA if it recurs more than once a week |
| **63** | Row-remapping / dynamic page retirement event | Soft-to-hard — a retired page is normal wear; frequent recurrence signals a dying memory cell |
| **74** | NVLink error | Hard — a broken interconnect link degrades or breaks multi-GPU collectives |
| **79** | GPU has fallen off the PCIe bus | Fatal — the GPU is unreachable; immediate RMA |
| **94 / 95** | Contained / uncontained ECC error (Hopper) | 94 contained = job-survivable but track it; 95 uncontained = workload output may be corrupted, treat as hard |

The severity column is the input to stage 4's triage decision below —
this table is the thing your custom plugin's regex and your controller's
routing logic both key off.

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
  (XID 48/94/95), NVLink/NVSwitch down (XID 74), row-remapping exhausted (XID
  63 recurring). Action: the node is out; open an **RMA**.

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

## Perspectives

**The developer/scheduler view.** A training job doesn't see "hardware health"
— it sees a pod get evicted mid-run. What it needs from your loop is a clean
signal (SIGTERM with enough grace period to checkpoint) and a gang-aware
scheduler (Volcano/JobSet/Kubeflow) that requeues the whole job on healthy
nodes rather than limping along one GPU short. The remediation loop's job is
to make hardware failure look, from the workload's perspective, like a
planned, resumable interruption instead of a silent slowdown.

**The operator/SRE view.** Your job is to keep the loop's own moving parts
boring: NPD's `invoke_interval` tuned so detection latency is predictable, the
remediation controller's cordon/drain policy tested against real PDBs (a
misconfigured PDB can make a drain hang forever, which is worse than no
automation), and the RMA ticket payload complete enough that a vendor doesn't
bounce it back asking for the GPU UUID you forgot to attach.

**The hardware/vendor view.** From the vendor's side, an RMA is a warranty
claim, and warranty claims are adjudicated on evidence: XID code, timestamp,
serial numbers, and ideally a DCGM diagnostic dump. Per-SKU failure-rate
tracking is also a *negotiating position* — if HGX-H100-8G batch "2024-Q3" is
failing at 3x the fleet average, that data is what gets you an accelerated
replacement or a credit, not a hunch.

**The economics view.** Every stage of this loop has a dollar translation:
detection latency × GPU-hours/hour × $/GPU-hour is the goodput bled while a
node stays schedulable but sick; RMA turnaround time is capacity you've paid
for but can't use until the swap lands; and per-SKU tracking converts "we've
had some hardware problems" into a quantified, vendor-facing claim. This
framing — not the YAML — is what a staff-level interview is actually probing
for when it asks "how do you close the health loop."

## Real-world use cases

- **CoreWeave — "What Is Node Lifecycle Management and Why Does It Matter for
  ML Training and Inference?"**
  ([coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference](https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference))
  — the primary anchor for this lesson. Beyond the "up to a month" detection-lag
  quote, it describes CoreWeave's **Fleet and Node Lifecycle Controllers**,
  owned by a dedicated **Fleet Engineering Team**, that detect GPU/hardware
  faults, automatically cordon unhealthy nodes, and drive hardware replacement
  — the exact detect → isolate → decide → RMA closed loop this lesson diagrams,
  described by the company that coined the "up to a month" framing this whole
  lesson is built around.
- **Nscale — "Inside Fleet Operations: Automating the GPU lifecycle"**
  ([nscale.com/blog/fleet-operations](https://www.nscale.com/blog/fleet-operations))
  — a second, independent GPU-neocloud describing the same closed loop: a
  Control Center that automates the hardware lifecycle from rack enrollment
  through maintenance, an observability platform that traces node health in
  real time, and a fault-surfacing API, together validating hardware, running
  burn-in/firmware checks, and auto-remediating before workloads are affected.
  Useful as independent corroboration that detect → cordon → drain → RMA is an
  industry norm, not one company's idiosyncrasy.
- **Cloudflare — "Automatic Remediation of Kubernetes Nodes"**
  ([blog.cloudflare.com/automatic-remediation-of-kubernetes-nodes/](https://blog.cloudflare.com/automatic-remediation-of-kubernetes-nodes/))
  — the canonical, widely-cited engineering write-up of building exactly this
  loop: NPD-style detection feeding a custom controller that cordons, drains,
  and remediates automatically, without a human paging at 3am for every bad
  node. The direct template for this lesson's "custom controller" option in
  stage 2–3.

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
5. **Triage + RMA.** XID 79 is a hard fault (fallen off the bus, per the table
   above) → the controller opens an RMA ticket with node serial, GPU UUID,
   XID, and lost-goodput fields; emits `gpu_node_remediation_total{sku=...,xid="79"}`.
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
   XIDs from the severity table above), the auto-generated ticket payload
   fields, and a **per-SKU failure-rate dashboard** sketch (the metric + the
   PromQL/panel).

**Acceptance:** a working NPD→cordon/drain demo on a synthetic fault, plus an
RMA + per-SKU failure-rate runbook, both checked into the deliverable, with the
**time-to-detect target** and its goodput-cost justification stated explicitly.

## Common pitfalls

- **Treating a liveness probe as hardware health.** A liveness probe restarts
  a *container* on the *same* node — if the node itself is broken, the pod
  just lands right back on broken hardware. Node-level health needs a
  node-level signal (NPD), not a pod-level one (see self-check c).
- **Draining without respecting PodDisruptionBudgets.** A drain that ignores
  PDBs turns a hardware fault into a job-killing outage instead of a graceful,
  checkpointed pause. Always drain through the eviction API, never a raw pod
  delete.
- **Reacting to every correctable ECC event as if it were fatal.** Single
  correctable ECC events are common and often benign; treating every one as an
  RMA trigger will pull healthy nodes out of service constantly. Track
  recurrence (e.g. XID 63 recurring, or multiple XID 48s in a week) and let
  the triage table's frequency thresholds — not a single event — decide.
- **Only using SystemLogMonitor or only CustomPluginMonitor.** NPD ships four
  monitor types; relying on just the XID-matching log monitor misses
  systemd-level failures (a crash-looping containerd) or host resource
  exhaustion that HealthChecker and SystemStatsMonitor are built to catch. A
  node with a healthy GPU but a dead container runtime is still unschedulable.
- **Filing an RMA ticket without enough evidence.** A ticket missing the GPU
  UUID, XID code, or a DCGM diagnostic dump gets bounced back by the vendor,
  costing you the exact detection-speed advantage this lesson is built around.
  Automate the payload so it's always complete.

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
NPD watches **node-level** signals (kernel log/XID, DCGM, custom scripts,
systemd unit health, host resource stats) and publishes a **permanent
NodeCondition** the scheduler and remediation controllers act on — cordon,
drain, RMA. It marks the *node* bad (durably), not the pod, and decouples
detection from remediation so a controller can close the loop.

**(d) NPD ships four monitor types. Name them and what each is for.**
**Answer:** **SystemLogMonitor** matches regex rules against the kernel log /
journald (e.g. catching an XID line in dmesg). **CustomPluginMonitor** runs a
user script/binary on an interval and maps its exit code to a Condition/Event
(the `check_gpu_xid.sh` pattern in this lesson, and what AKS's `GPUXIDErrors`
check is built on). **SystemStatsMonitor** gathers host-level resource
statistics (disk, memory, CPU pressure) into conditions. **HealthChecker**
checks the operational status of specific systemd services — kubelet,
containerd/docker, ntp — so a node with a flapping container runtime is
flagged even if its GPUs look fine.

**(e) A node shows XID 48 twice this month, three weeks apart. Is it a
reboot-and-return or an RMA?**
**Answer:** By the triage table, XID 48 (uncorrectable double-bit ECC) is a
hard-fault code, and the standard guidance is RMA if it **recurs more than
once a week** — two occurrences three weeks apart is below that recurrence
threshold, so the correct call is reboot/reset, return to service, and
**track** it (per stage 4's rule that a node "recovering" repeatedly is really
an RMA). The decision is frequency-based, not single-event-based: one XID 48
isn't automatically a return, but a rising rate on the same node (or the same
SKU/batch, per stage 5) is the signal that converts it into one.

## Connections & what's next

This lesson closes the loop that lesson 05 opened: provisioning builds a node
forward, this lesson tears one down safely and — when the hardware is bad —
sends it back through lesson 05's exact pipeline for re-imaging. The
`MachineHealthCheck` remediation path ties directly back to lesson 04's CAPI
object model (an unhealthy `Machine` gets deleted and the infra provider
re-provisions it automatically), so all three lessons (04, 05, 06) describe one
continuous machine lifecycle viewed from three angles: declarative shape,
forward bring-up, and backward remediation. The eviction/PDB mechanics here
also foreshadow lesson 08's checkpointing story — a drain is only safe because
a job can checkpoint and resume, which is precisely what makes the "cordon
first, drain gracefully" policy possible instead of merely aspirational.

Next, lesson 07 moves from *keeping nodes healthy* to *keeping GPUs fed*:
storage sizing, tiering, and the aggregate bandwidth a fleet of healthy nodes
actually needs to avoid idling six-figure silicon on I/O waits — a different
kind of "goodput," same underlying economics.

## References & further reading

**Primary sources**
- **Node Problem Detector** — <https://github.com/kubernetes/node-problem-detector>
  — the four monitor types (SystemLogMonitor, CustomPluginMonitor,
  SystemStatsMonitor, HealthChecker), Condition/Event exporters, and config
  format; the deep reference for stage 1.
- **NVIDIA — "XID Reporting"** — <https://docs.nvidia.com/deploy/topics/topic_4_1.html>
  — the canonical, authoritative list of XID codes and their meanings; read
  this before trusting any third-party XID summary, including this lesson's
  table.
- **AKS GPU health monitoring** — <https://learn.microsoft.com/en-us/azure/aks/gpu-health-monitoring>
  — a production reference implementation: AKS runs NPD on GPU node pools with
  a `GPUXIDErrors` check (the custom-plugin pattern built by hand in this
  lesson) and a `GPUMissing` check for enumeration failures.
- **AKS GPU observability (DCGM metrics)** — <https://learn.microsoft.com/en-us/azure/aks/monitor-gpu-metrics>
  — companion doc on DCGM-based GPU metrics collection, the observability
  layer that should back a real (non-dmesg) health check in stage 1.

**Real-world engineering blogs**
- **CoreWeave — "What Is Node Lifecycle Management…"** —
  <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference>
  — the "up to a month" detection-lag quote and the Fleet/Node Lifecycle
  Controllers that automate detect → cordon → replace; the primary anchor for
  this lesson.
- **Nscale — "Inside Fleet Operations: Automating the GPU lifecycle"** —
  <https://www.nscale.com/blog/fleet-operations> — independent corroboration
  from a second GPU-neocloud: burn-in, firmware checks, and auto-remediation
  before workloads are affected.
- **Cloudflare — "Automatic Remediation of Kubernetes Nodes"** —
  <https://blog.cloudflare.com/automatic-remediation-of-kubernetes-nodes/> —
  the canonical NPD → controller → auto-remediation closed-loop write-up, and
  the direct template for the "custom controller" option in stages 2–3.

**Deeper dives**
- **Medik8s Node Healthcheck Operator + Self-Node-Remediation docs** — the
  CRD-driven, more controllable successor to draino referenced in stage 2–3;
  linked from the NPD ecosystem docs at
  <https://github.com/kubernetes/node-problem-detector>.
- **draino (`planetlabs/draino`)** — the original Cloudflare-adjacent
  cordon-and-drain-on-Condition controller referenced throughout stage 2–3;
  worth reading the source for how small the "watch Condition, cordon, drain"
  loop actually is in code.
