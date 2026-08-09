---
lesson: "04.5"
title: "Driver Lifecycle & Rolling Upgrades"
module: "04"
concept: "Driver Lifecycle & Rolling Upgrades"
status: not-started
est_time: "9h"
artifacts: []
---
# 04.5 · Driver Lifecycle & Rolling Upgrades
> **Concept.** The GPU driver is a kernel module with cluster-wide blast radius; upgrading it safely is a drain-orchestrated state machine, not a `kubectl set image`.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters

The NVIDIA kernel driver is the single most dangerous thing to upgrade on a GPU node. It is an out-of-tree kernel module (`nvidia.ko`, `nvidia_uvm.ko`, `nvidia_modeset.ko`, plus `nvidia-peermem` for RDMA) that is *held open* by every running CUDA process. You cannot unload and reload it while a training job holds a context — the module refcount is non-zero, `rmmod` returns `EBUSY`, and the new driver never loads. So a driver upgrade is fundamentally a *tenant-eviction* problem wearing a kernel-module costume.

Get this wrong on a 200-node fleet and you have three failure modes that all cost real money:

- **Version skew halts the fleet.** The userspace CUDA driver (`libcuda.so`, shipped in the driver container) and the kernel module must match. If the operator restarts the driver DaemonSet pod but the old module is still loaded, every `cudaInit` on that node returns `Error 803: system has unsupported display driver / cuda driver combination`. The node looks Ready to Kubernetes but every GPU pod crashloops.
- **Silent eviction of long jobs.** A naive rollout drains a node mid-epoch. A 40-hour distributed training run with no checkpointing loses everything, and if it's one rank of a 512-GPU job, *all 512 GPUs* sit idle waiting for the gang to reschedule.
- **Node stuck half-upgraded.** The kernel module fails to build against a new kernel (DKMS/precompiled mismatch), the node wedges in a non-terminal upgrade state, and if your `maxUnavailable` is generous you take out capacity in batches before anyone notices.

At a GPU-heavy shop the driver-upgrade runbook is a promotion artifact. "I upgraded 200 H100 nodes from 550.x to 570.x with zero training-job loss and a documented rollback gate" is a sentence that gets you the senior req. This lesson is how you write it.

## What's new here

Lesson 03 taught the driver as *silicon enablement* — what the kernel module does to make MIG, ECC, and the SM scheduler visible. Nothing there was operational. Here the driver is a **fleet lifecycle object** managed by the GPU Operator, and every new concept is about *change management under load*:

| Lesson 03 (silicon) | Lesson 04.5 (operations) |
|---|---|
| Driver exposes GPU to userspace | Operator *owns* the driver as a DaemonSet you version-bump declaratively |
| Kernel module loads at boot | Module is upgraded **rolling**, node-by-node, with cordon/drain |
| Open vs proprietary = a build flavor | `kernelModuleType` is a ClusterPolicy field with RDMA/GDS and support-matrix consequences |
| "The driver is loaded" | A per-node **upgrade state machine** (`nvidia.com/gpu-driver-upgrade-state`) you observe and gate on |

The operational layer is the GPU Operator's **upgrade controller**. When you change the driver version in the `ClusterPolicy` (or `NVIDIADriver` CR), the controller does not just roll the DaemonSet — it walks each node through a labeled state machine that cordons, evicts GPU workloads, restarts the driver pod, revalidates, and uncordons, honoring concurrency and drain policy you configure. Your job is to understand that machine well enough to design its parameters for a 200-node fleet.

## Core notes

### Operator-managed vs pre-installed drivers

Two supported topologies, and you must know which you're on before touching anything:

- **Operator-managed (`driver.enabled=true`)** — the GPU Operator runs an `nvidia-driver-daemonset` per node. The container ships the kernel module + userspace and loads the module into the host kernel via the driver-container pattern. Upgrades are declarative: change the image tag, the upgrade controller does the rest. This is the default and what the rolling state machine below applies to.
- **Pre-installed / host driver (`driver.enabled=false`)** — the driver is baked into the node image (golden AMI, bootc, or a config-management run) and the operator only deploys the toolkit, device plugin, DCGM, and validators. Upgrades are a *node-image* problem: you roll new images through your normal node-pool upgrade (managed node group rotation, Cluster API `MachineDeployment` rollout). CoreWeave-style neoclouds and many on-prem HPC shops do this because they want the driver in the image for reproducibility and faster boot, and they don't want a container holding the kernel module.

Rule of thumb: operator-managed gives you fast, declarative, in-place driver bumps without reimaging; pre-installed gives you immutable-infra guarantees and decouples driver changes from the operator's release cadence. Big fleets often run pre-installed for the base driver and use the operator purely for the plugin/DCGM/MIG stack. Know which one the interview scenario assumes.

### Open vs proprietary kernel modules (`kernelModuleType`)

NVIDIA ships the kernel module in two flavors from the same driver branch:

- **Open** (`nvidia-open`, aka the *open GPU kernel modules*) — GPL/MIT-licensed source, still with a closed userspace firmware/GSP blob. **Required** for Grace Hopper / Blackwell-class parts (GH200, GB200) and strongly preferred on Hopper (H100/H200). It's the substrate for DMA-BUF, which GPUDirect RDMA and GPUDirect Storage (GDS) increasingly depend on.
- **Proprietary** (`nvidia`, the legacy closed module) — the historical default. Still needed for older architectures that the open module doesn't support (pre-Turing, and some Maxwell/Pascal datacenter parts).

**Version flag:** with the **R570** driver branch the GPU Operator changed the *default to open*. `driver.kernelModuleType` in the `ClusterPolicy` (and the `NVIDIADriver` CR) accepts `auto`, `open`, or `proprietary`. `auto` lets the driver container pick the correct flavor for the detected GPU + driver branch — on R570+ that resolves to open for supported SKUs. If you pin an older branch (e.g. 535/550 LTS) the effective default is still proprietary. **GPU Operator 25.x** (e.g. 25.3) is the release line that assumes this new default. When you write a runbook, *state the branch and the resolved module type explicitly* — "570.x, open modules" — because a reader on a 535 LTS node gets different behavior from the same YAML.

When to pick which:
- **Open** — Hopper/Blackwell (mandatory on some), anything using GPUDirect RDMA/Storage or DMA-BUF (InfiniBand fabrics, GPUDirect-backed object stores), and any fleet standardizing on R570+ for forward compatibility.
- **Proprietary** — legacy datacenter GPUs the open module doesn't support, or a validated legacy stack you're not ready to requalify. Also the pragmatic pick if a third-party kernel module in your stack has only been tested against the closed driver.

### The rolling driver-upgrade state machine

When the driver version changes, the upgrade controller labels each GPU node with `nvidia.com/gpu-driver-upgrade-state` and advances it through these values (this is the canonical order — memorize it):

| State (`nvidia.com/gpu-driver-upgrade-state`) | What happens |
|---|---|
| `upgrade-done` | Steady state. Driver pod is current; nothing to do. |
| `upgrade-required` | Controller detected the driver pod is out of date; node is queued for upgrade (subject to concurrency limits). |
| `cordon-required` | Node is marked `Unschedulable` (cordoned) so no new GPU pods land during the upgrade. |
| `wait-for-jobs-required` | Optional gate: wait for `Job`-backed GPU pods to complete on their own before evicting (guards batch/training workloads). Controlled by drain-policy timeout. |
| `pod-deletion-required` | Optional: delete GPU pods directly (by selector) instead of a full node drain. Skipped unless pod-deletion is configured. |
| `drain-required` | Node is drained (Kubernetes eviction) to evict remaining GPU workloads and release the kernel module. **Skipped** if pod-deletion already cleared all GPU pods. |
| `pod-restart-required` | The `nvidia-driver-daemonset` pod is restarted with the new image; old module unloads (now that no CUDA context holds it) and the new module loads. |
| `validation-required` | The `nvidia-operator-validator` pod runs its plugin chain (driver, toolkit, CUDA, plus MIG/workload validations) to confirm the new driver actually works. |
| `uncordon-required` | Node is uncordoned back to `Schedulable`. |
| `upgrade-failed` | Terminal-failure state; a step errored (build failure, validation failure, drain timeout) and the node is parked for human attention. |

The DaemonSet update strategy is `OnDelete` — the controller, not the DaemonSet controller, decides *when* each pod is deleted, which is exactly how it enforces the drain-before-restart ordering.

### Upgrade policy configuration (`driver.upgradePolicy`)

You tune the machine's aggressiveness in the `ClusterPolicy` under `spec.driver.upgradePolicy`:

```yaml
spec:
  driver:
    version: "570.148.08"           # bump this to trigger the rollout
    kernelModuleType: "open"        # auto | open | proprietary  (R570 default: open)
    upgradePolicy:
      autoUpgrade: true             # false = park nodes in upgrade-required for manual gating
      maxParallelUpgrades: 3        # how many nodes upgrade concurrently
      maxUnavailable: "20%"         # cap on simultaneously unavailable (cordoned/draining) nodes; int or %
      waitForCompletion:
        timeoutSeconds: 0           # wait-for-jobs gate; 0 = wait indefinitely for matching pods
        podSelector: "training=long-running"   # only wait for these before draining
      drain:
        enable: true                # false = never drain (relies on pod-deletion / job-wait only)
        force: true                 # evict bare/standalone pods not backed by a controller
        podSelector: ""             # limit drain to matching pods
        timeoutSeconds: 300         # drain deadline; on expiry -> upgrade-failed unless force logic clears it
        deleteEmptyDir: true        # allow eviction of pods with emptyDir volumes
      podDeletion:
        force: true                 # delete GPU pods directly instead of full drain
        timeoutSeconds: 300
        deleteEmptyDir: true
```

Key levers for a large fleet:
- **`maxParallelUpgrades`** bounds *concurrency* (how many nodes actively move through the machine); **`maxUnavailable`** bounds *impact* (how much capacity can be simultaneously offline). The effective batch is the *min* of the two, so set both — `maxParallelUpgrades` for throughput, `maxUnavailable` as the safety ceiling.
- **`waitForCompletion` + `podSelector`** is your training-job protection: label long jobs and the controller waits (up to timeout) for them to finish before it will drain, so short jobs and services drain immediately while the 40-hour run is left alone.
- **`drain.timeoutSeconds`** vs stuck pods: if a `PodDisruptionBudget` blocks eviction past the timeout, the node goes `upgrade-failed` rather than hanging forever — a feature, because it surfaces the blockage instead of silently stalling the rollout.

## Worked example

Trigger a version bump and watch one node walk the machine.

**1. Confirm starting state.**
```bash
$ kubectl -n gpu-operator get clusterpolicy cluster-policy \
    -o jsonpath='{.spec.driver.version}{"\n"}'
550.127.08

$ kubectl get nodes -L nvidia.com/gpu-driver-upgrade-state
NAME        STATUS   ...   GPU-DRIVER-UPGRADE-STATE
gpu-node-1  Ready          upgrade-done
```

**2. Bump the driver version (and pin the module type).**
```bash
$ kubectl -n gpu-operator patch clusterpolicy cluster-policy --type merge -p \
  '{"spec":{"driver":{"version":"570.148.08","kernelModuleType":"open",
     "upgradePolicy":{"maxParallelUpgrades":1,"maxUnavailable":"1"}}}}'
clusterpolicy.nvidia.com/cluster-policy patched
```

**3. Watch the label transition on the node.** Poll the label; you'll see it advance:
```bash
$ kubectl get node gpu-node-1 \
    -L nvidia.com/gpu-driver-upgrade-state -w
NAME        STATUS                     GPU-DRIVER-UPGRADE-STATE
gpu-node-1  Ready                      upgrade-required
gpu-node-1  Ready,SchedulingDisabled   cordon-required
gpu-node-1  Ready,SchedulingDisabled   drain-required
gpu-node-1  Ready,SchedulingDisabled   pod-restart-required
gpu-node-1  Ready,SchedulingDisabled   validation-required
gpu-node-1  Ready                      uncordon-required
gpu-node-1  Ready                      upgrade-done
```

**4. Correlate with the driver pod restart.** During `pod-restart-required` the DaemonSet pod is deleted and recreated:
```bash
$ kubectl -n gpu-operator get pod -l app=nvidia-driver-daemonset \
    -o wide --field-selector spec.nodeName=gpu-node-1 -w
NAME                          READY   STATUS        RESTARTS   NODE
nvidia-driver-daemonset-abc   1/1     Terminating   0          gpu-node-1
nvidia-driver-daemonset-xyz   0/1     Init:0/1      0          gpu-node-1
nvidia-driver-daemonset-xyz   1/1     Running       0          gpu-node-1
```

**5. Confirm the module actually reloaded** (the check the operator's validator automates):
```bash
$ kubectl -n gpu-operator exec nvidia-driver-daemonset-xyz -- nvidia-smi \
    --query-gpu=driver_version --format=csv,noheader
570.148.08
$ kubectl -n gpu-operator logs nvidia-operator-validator-... -c driver-validation
driver is not ready... / all validations successful
```

If the node parks at `validation-required` or `upgrade-failed`, jump to Self-check (c).

## Practice

This task feeds the deliverable's **failure-mode log** and produces the **200-node runbook** artifact.

**Part A — observe the machine (single node).** On your lab cluster (a single GPU node is fine; kind + a cloud GPU VM, or one node of a managed GPU pool):
1. Record the current `spec.driver.version` and the node's `nvidia.com/gpu-driver-upgrade-state`.
2. Schedule a marker GPU pod (`nvidia-smi -l 5` in a loop) so the node is *in use* when you trigger the upgrade.
3. Patch the `ClusterPolicy` to a new driver version with `maxParallelUpgrades=1`, `maxUnavailable=1`.
4. Capture the full ordered sequence of `nvidia.com/gpu-driver-upgrade-state` values with timestamps (`kubectl get node -w` piped to a file), plus the cordon (`SchedulingDisabled`) transition and the driver-pod restart. Confirm your marker pod was evicted at `drain-required` and rescheduled after `uncordon-required`.

**Part B — write the 200-node rollout runbook.** Produce a runbook that covers, explicitly:
- **Batching:** chosen `maxParallelUpgrades` and `maxUnavailable` with the reasoning (e.g. `maxUnavailable: 5%` → ≤10 nodes offline, `maxParallelUpgrades: 4`), and the topology constraint (never drain two nodes of the same NVLink/rail domain or the same gang-scheduled job at once).
- **Cordon/drain policy:** `drain.enable`, `force`, `deleteEmptyDir`, `timeoutSeconds`, and the `waitForCompletion.podSelector` you use to *not evict long training jobs*.
- **Long-job protection:** how you label long runs, how the wait gate interacts with checkpointing, and what you tell job owners (checkpoint cadence expectation before the window).
- **Health gates:** what must be green before the controller advances a batch — `nvidia-operator-validator` success, DCGM health (`dcgmi diag -r 1` or the exporter's health metric), node `Ready`, and a canary GPU pod passing.
- **Rollback trigger:** the concrete condition that stops the rollout (e.g. >1 node hits `upgrade-failed`, or DCGM XID errors spike) and the rollback action (repin `spec.driver.version` to the old tag — the controller rolls the same machine back).
- **Canary → wave plan:** 1 node → 5% → 25% → 100%, with a soak time and the metric watched at each wave.

**Acceptance:**
- The observed, timestamped state-machine transitions from Part A (including the cordon and driver-pod restart), saved to the failure-mode log.
- A written 200-node upgrade runbook containing *all six* bullet categories above, each with concrete values (not "TBD"), a named rollback trigger, and an explicit statement of how long training jobs avoid eviction.

## Self-check

**(a) List the driver-upgrade state-machine steps and what each does.**

**Answer:** In order via `nvidia.com/gpu-driver-upgrade-state`:
`upgrade-required` (driver pod out of date, node queued) → `cordon-required` (mark node Unschedulable) → `wait-for-jobs-required` (optionally wait for Job pods to finish) → `pod-deletion-required` (optionally delete GPU pods directly) → `drain-required` (drain the node to release the kernel module; skipped if pod-deletion cleared all GPU pods) → `pod-restart-required` (restart the `nvidia-driver-daemonset` pod so the old module unloads and the new one loads) → `validation-required` (`nvidia-operator-validator` runs driver/toolkit/CUDA/MIG checks) → `uncordon-required` (mark node Schedulable) → `upgrade-done` (steady state). `upgrade-failed` is the terminal error state for any failed step.

**(b) Open vs proprietary kernel modules — when would you pick each?**

**Answer:** **Open** (`nvidia-open`) is the R570+ default and is required on Grace Hopper/Blackwell and preferred on Hopper; pick it whenever you need DMA-BUF, GPUDirect RDMA, or GPUDirect Storage, or want forward compatibility on a modern fleet. **Proprietary** (`nvidia`, legacy closed module) is for older datacenter GPUs the open module doesn't support, or a validated legacy stack (or third-party kernel module) you can't requalify yet. Set it via `driver.kernelModuleType: open|proprietary|auto`; `auto` resolves to open on R570+ for supported SKUs and proprietary on older branches — so always state the driver *branch* alongside the choice.

**(c) A node is stuck in `validation-required` after an upgrade — what do you check, in order?**

**Answer:** (1) `nvidia-operator-validator` pod logs on that node — which sub-validation failed (driver, toolkit, cuda, plugin, MIG, workload). (2) The driver DaemonSet pod: is it `Running`, and does `nvidia-smi` inside it report the *new* version? If it's crashlooping, read its logs for a kernel-module build/load failure (kernel-headers mismatch, DKMS failure, Secure Boot signing). (3) `dmesg`/host kernel log for `nvidia.ko` load errors or a still-loaded *old* module (refcount stuck because a CUDA process didn't die during drain — check for leftover GPU pods that escaped eviction). (4) MIG state: if the node is MIG-enabled, a bad `mig-manager` state or `mig-mode` mismatch fails the MIG validation — check `nvidia-smi mig -lgi`. (5) Toolkit/device-plugin pods for the toolkit-validation step. Fix root cause, delete the validator pod to re-run; if unrecoverable, roll `spec.driver.version` back to trigger the reverse transition.

## Resources

1. **NVIDIA GPU Operator — GPU Driver Upgrades** (the canonical state-machine + `upgradePolicy` reference): https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-driver-upgrades.html
2. **GPU Operator ClusterPolicy / `driver` spec reference** (fields: `version`, `kernelModuleType`, `upgradePolicy`, `drain`): https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html
3. **GPU Operator Release Notes (25.x)** — confirms the R570 open-module default change and version-specific behavior: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html
