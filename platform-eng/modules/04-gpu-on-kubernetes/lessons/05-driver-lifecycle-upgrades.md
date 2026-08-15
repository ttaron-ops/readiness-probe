---
lesson: "04.5"
title: "Driver Lifecycle & Rolling Upgrades"
module: "04"
concept: "Driver lifecycle & fleet upgrades"
status: not-started
est_time: "11h"
prev: "04-container-runtime-integration.md"
next: "06-mig-operations.md"
artifacts: []
sources: 9
---

# 04.5 · Driver Lifecycle & Rolling Upgrades

> **Concept.** The GPU driver is a kernel module with cluster-wide blast radius; upgrading it safely is a drain-orchestrated state machine, not a `kubectl set image`.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 04 established CDI (Container Device Interface) as the declarative, per-pod-start mechanism a container runtime uses to inject `/dev/nvidia*` into a container — a read-mostly integration you debug when a single pod can't see its GPU. This lesson is about the thing CDI injects: the kernel driver itself, and what happens when you have to *change* it under a fleet of nodes that are all currently running jobs. Lessons 01–04 taught you the GPU Operator's steady-state component chain (NFD → driver → toolkit → device-plugin/GFD/DCGM → validator); this lesson is what the Operator does when steady state has to move — a controller-driven, per-node state machine that cordons, drains, restarts, and validates. It also introduces the cordon/drain/validate vocabulary that lesson 06 (MIG operations) reuses for a different kind of disruptive change: repartitioning silicon instead of swapping a kernel module.

## Why this matters

The NVIDIA kernel driver is the single most dangerous thing to upgrade on a GPU node. It ships as a set of out-of-tree kernel modules — `nvidia.ko` (core), `nvidia_uvm.ko` (Unified Virtual Memory), `nvidia_modeset.ko`, and `nvidia-peermem` for RDMA — and each is *held open* by every running CUDA process. You cannot unload and reload a kernel module while a training job holds a context on it: the module's reference count is non-zero, `rmmod` returns `EBUSY`, and the new driver never loads. A driver upgrade is therefore fundamentally a *tenant-eviction* problem wearing a kernel-module costume, and every mechanism in this lesson exists to manage that eviction safely.

Get it wrong on a large fleet and three failure modes all cost real money:

- **Version skew halts the fleet.** The userspace CUDA driver (`libcuda.so`, shipped inside the driver container) and the loaded kernel module must match. If a driver pod restarts but the old module is still resident, every `cudaInit()` on that node returns a driver/library version-mismatch error. The node looks `Ready` to Kubernetes; every GPU pod on it crash-loops.
- **Silent eviction of long jobs.** A naive rollout drains a node mid-epoch. A 40-hour distributed training run with no checkpointing loses its work — and if that node holds one rank of a 512-GPU gang-scheduled job, all 512 GPUs sit idle waiting for the whole gang to reschedule, not just the one node's worth.
- **A node stuck half-upgraded.** The kernel module fails to build against a new kernel (a DKMS/precompiled mismatch), the node wedges in a non-terminal upgrade state, and a generous `maxUnavailable` lets several nodes fail the same way before anyone notices — you've taken out real capacity in the dark.

This module's own README names the probe explicitly: *"design a driver upgrade for 200 nodes."* That's not a hypothetical — it's the exact interview question senior GPU-platform roles ask to separate someone who has *operated* a GPU Operator install from someone who has *designed* a rollout with batching, blast-radius limits, long-job protection, and a rollback trigger. "I upgraded 200 H100 nodes from one driver line to the next with zero training-job loss and a documented rollback gate" is the sentence that gets the senior req. This lesson is how you write it, and how you defend it live when the interviewer asks "what if a PodDisruptionBudget blocks eviction on node 47?"

## What's new here (calibration)

Module 02/02b/03 already own the theory this lesson would otherwise have to re-derive, so it isn't re-taught here:

- **Device-plugin gRPC mechanics and the DRA object model** (module 02) — you know how a device gets advertised and allocated; this lesson doesn't re-explain `Allocate()`.
- **What the kernel module *does* at the silicon level** — SM scheduling, ECC, MIG enablement (module 03) — this lesson treats the driver as a fleet lifecycle object, not a hardware-enablement mechanism.
- **Topology Manager / NUMA pinning** (module 02b) — orthogonal to a driver upgrade; not revisited.

What's genuinely new: the GPU Operator's **upgrade controller** as a per-node state machine you must be able to read live off `kubectl get node -L`; the **`upgradePolicy`** knobs that trade throughput against blast radius at fleet scale; the fact that **not every driver-pod restart is the same operation anymore** — GPU Operator 26.3.0 changed same-version restarts to reuse the loaded kernel module, and you need to know which restart you're looking at before you reason about its risk; and the reality that on a managed GPU cloud, this entire layer may not be yours to operate at all.

## Core concepts

### The GPU Operator release line — why you should never hard-pin a version in a runbook

> **Version-sensitive — verify at study time.** The facts below were confirmed via GitHub releases as of mid-2026; the Operator ships frequently, so check `helm search repo nvidia/gpu-operator -l` (after `helm repo update`) or the [release notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html) before designing a real rollout.

The public release line moved **v25.3.x → v25.10.0/v25.10.1 → v26.3.0/.1/.2/.3**. Each jump changed behavior this lesson's runbook cares about:

| Line | What changed for driver lifecycle |
|---|---|
| 25.3.x | The "classic" behavior most existing docs and blog posts describe: legacy NVIDIA-Container-Toolkit hook injection is still common, driver-pod restarts always rebuild/reload the kernel module. |
| 25.10.0 / .1 | **CDI becomes the default** injection path (containerd/CRI-O use CDI out of the box instead of the legacy prestart hook — the mechanism lesson 04 covered). Run `nvidia-ctk cdi list` before assuming which injection path you're debugging on a node you didn't provision yourself. |
| 26.3.0+ | Adds the **`NVIDIADriver` custom resource** (below) for multi-driver-version fleets, and changes the driver-pod restart behavior for the *same-version* case (below). |

The practical rule for a runbook: **never write `--version=v25.3.2`** as if it's evergreen. State the behavior you're depending on ("CDI is the default injection path," "same-version restarts are fast") and tell the reader to confirm which release line resolves that behavior on the cluster in front of them.

### Operator-managed vs pre-installed drivers

Two supported topologies, and you must know which you're on before touching anything:

- **Operator-managed (`driver.enabled=true`)** — the GPU Operator runs an `nvidia-driver-daemonset` per node. The container ships the kernel module plus userspace and loads the module into the host kernel via the driver-container pattern. Upgrades are declarative: change the image tag, the upgrade controller does the rest. This is the default, and the rolling state machine below applies to it.
- **Pre-installed / host driver (`driver.enabled=false`)** — the driver is baked into the node image (a golden AMI, `bootc`, or a config-management run), and the Operator only deploys the toolkit, device plugin, DCGM, and validators. Upgrades are a *node-image* problem: you roll new images through your normal node-pool rotation (managed node group replacement, a Cluster API `MachineDeployment` rollout). Fleets that want immutable-infrastructure guarantees, faster boot (no in-cluster kernel build/load step), or to decouple the driver's release cadence from the Operator's often choose this path.

Rule of thumb: operator-managed gives you fast, declarative, in-place driver bumps without reimaging nodes; pre-installed gives you immutable-infra guarantees at the cost of coupling driver changes to your node-image pipeline. Many large fleets run pre-installed for the base driver and use the Operator purely for the plugin/DCGM/MIG stack. State which one an interview scenario assumes before answering — the two upgrade playbooks are almost entirely different.

### Open vs proprietary kernel modules (`kernelModuleType`)

NVIDIA ships the kernel module in two flavors from the same driver branch:

- **Open** (`nvidia-open`, the *open GPU kernel modules*) — GPL/MIT-licensed source, still paired with a closed userspace firmware/GSP (GPU System Processor) blob. **Required** for Grace Hopper/Blackwell-class parts (GH200, GB200) and the substrate for DMA-BUF, which GPUDirect RDMA and GPUDirect Storage increasingly depend on.
- **Proprietary** (`nvidia`, the legacy closed module) — the historical default, still needed for older architectures the open module doesn't support (pre-Turing, and some Maxwell/Pascal datacenter parts).

> **Version-sensitive — verify at study time.** The commonly cited milestone is "open became the default at R570," but the evidence found this session points earlier: **the open kernel module reportedly became the full default starting at R560** — R570 just continued that default. The current datacenter driver branch has moved to **R580.x** (e.g. 580.159.04, 580.167.08, 580.173.02 — a February 2026 snapshot; treat as dated), which adds **CDMM** (a memory-management feature for GB200-class multi-node NVLink) and an `nvidia-driver-assistant` tool that auto-detects the correct module flavor for the installed GPU. Whichever branch you're on, `driver.kernelModuleType` accepts `auto | open | proprietary`; `auto` resolves correctly for the detected GPU **on a current branch**, but resolves to `proprietary` if you've pinned an older LTS line the open module never covered. **Always state the driver branch alongside the module-type choice in a runbook** — the same `kernelModuleType: auto` YAML behaves differently depending on what `driver.version` it sits next to.

When to pick which:
- **Open** — Hopper/Blackwell (mandatory on some parts), anything using GPUDirect RDMA/Storage or DMA-BUF (InfiniBand fabrics, GPUDirect-backed object stores), and any fleet standardizing on a current branch for forward compatibility.
- **Proprietary** — legacy datacenter GPUs the open module doesn't support, or a validated legacy stack you're not ready to requalify. Also the pragmatic pick if a third-party kernel module in your stack has only been tested against the closed driver.

### The `NVIDIADriver` CRD — managing a heterogeneous fleet

Older `ClusterPolicy`-only configuration assumes **one** driver version for the whole cluster. That breaks down the moment your fleet is heterogeneous — some nodes are H100s that need a current branch for GPUDirect, some are older T4s pinned to a validated LTS line for a workload you can't requalify yet. GPU Operator **v26.3.0** added the **`NVIDIADriver` custom resource** specifically for this: instead of one global `driver.version` in `ClusterPolicy`, you define multiple `NVIDIADriver` objects, each scoped to a node selector (by GPU product, OS, or label), and each carries its own version/module-type/upgrade policy. This is the modern answer to "how do I run two driver versions on the same cluster without two clusters" — a real 200-node-fleet design constraint, not a toy feature. If your interview scenario mentions mixed GPU generations in one cluster, this CRD is the answer to reach for.

### The rolling driver-upgrade state machine

When the driver version changes (via `ClusterPolicy` or an `NVIDIADriver` object), the upgrade controller labels each affected GPU node with `nvidia.com/gpu-driver-upgrade-state` and advances it through these values — memorize the order, it's the backbone of every self-check and every incident on this topic:

| State (`nvidia.com/gpu-driver-upgrade-state`) | What happens |
|---|---|
| `upgrade-done` | Steady state. Driver pod is current; nothing to do. |
| `upgrade-required` | Controller detected the driver pod is out of date; node is queued for upgrade (subject to concurrency limits). |
| `cordon-required` | Node is marked `Unschedulable` (cordoned) so no new GPU pods land during the upgrade. |
| `wait-for-jobs-required` | Optional gate: wait for `Job`-backed GPU pods to complete on their own before evicting (guards batch/training workloads). Controlled by a drain-policy timeout and pod selector. |
| `pod-deletion-required` | Optional: delete GPU pods directly (by selector) instead of a full node drain. Skipped unless pod-deletion is configured. |
| `drain-required` | Node is drained (Kubernetes eviction) to evict remaining GPU workloads and release the kernel module. **Skipped** if pod-deletion already cleared all GPU pods. |
| `pod-restart-required` | The `nvidia-driver-daemonset` pod is restarted with the new image. See the callout below — this step's actual cost now depends on whether the version changed. |
| `validation-required` | The `nvidia-operator-validator` pod runs its plugin chain (driver, toolkit, CUDA, plus MIG/workload validations) to confirm the new driver actually works. |
| `uncordon-required` | Node is uncordoned back to `Schedulable`. |
| `upgrade-failed` | Terminal-failure state; a step errored (build failure, validation failure, drain timeout) and the node is parked for human attention. |

The DaemonSet update strategy is `OnDelete` — the *upgrade controller*, not the DaemonSet controller, decides *when* each pod is deleted, which is exactly how it enforces the drain-before-restart ordering above.

> **Explicit version caveat — `pod-restart-required` is no longer one operation.** As of **GPU Operator v26.3.0**, if the target driver version *equals* the version already running on the node (for example, you're re-applying the same tag to pick up an unrelated `ClusterPolicy` field, or reconciling drift), the driver-pod restart **reuses the already-loaded kernel module instead of unloading and rebuilding it** — recovery drops from **minutes to seconds**. This does **not** change the state machine's structure for a real version bump (550→570, or any actual driver upgrade still walks every state above, full unload/rebuild included) — it only changes how expensive `pod-restart-required` is for the same-version case. Practically: don't assume every driver-pod `Restarting` you see in `kubectl get pods -w` is a multi-minute kernel rebuild — check whether `spec.driver.version` actually changed before you estimate downtime.

### Upgrade policy configuration (`driver.upgradePolicy`)

You tune the machine's aggressiveness in the `ClusterPolicy` under `spec.driver.upgradePolicy` (the same shape applies per-object under an `NVIDIADriver` CR):

```yaml
spec:
  driver:
    version: "570.148.08"           # bump this to trigger the rollout — never hard-pin in a runbook without a "verify current" note
    kernelModuleType: "open"        # auto | open | proprietary — state the driver branch alongside this
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
- **`drain.timeoutSeconds`** vs. stuck pods: if a `PodDisruptionBudget` blocks eviction past the timeout, the node goes `upgrade-failed` rather than hanging forever — a feature, because it surfaces the blockage instead of silently stalling the rollout.

## Perspectives

**Fleet SRE at 200-node scale.** `maxParallelUpgrades` and `maxUnavailable` are the two knobs that turn "design a driver upgrade for 200 nodes" from a scary open question into an arithmetic answer: pick a blast-radius ceiling first (`maxUnavailable: 5%` → at most 10 nodes offline at once on a 200-node fleet), then pick throughput (`maxParallelUpgrades`) up to that ceiling. The topology constraint that doesn't show up in the YAML at all — never draining two nodes of the same NVLink/rail domain or the same gang-scheduled job simultaneously — is the thing that actually separates a senior answer from a junior one in this interview.

**Kernel/systems.** The `rmmod`/refcount fact from module 03 — a CUDA context holds `nvidia.ko` open, so you can't unload it out from under a live process — is the same fact here, just turned from a debugging nuisance into a *planned* operation. The entire drain-before-restart ordering in the state machine exists solely to make that refcount hit zero on purpose, on your schedule, instead of finding out about it via a crash.

**Managed-cloud operator.** Not every GPU fleet operator manages this layer themselves. CoreWeave's Kubernetes service (CKS) offers **per-nodepool GPU driver version selection** with queued reconfigure-and-reboot semantics, plus a feature CoreWeave calls **"Protected Rolling Updates"** that is described as workload-scheduler-state-aware (i.e., it won't roll a node out from under an active Slurm/batch job) — see the reference below; this citation is search-only and unfetched this session, corroborated across two independent search results, and should be spot-checked before you rely on exact mechanics. The point for an interview answer: on a managed GPU cloud, "design a driver upgrade for 200 nodes" may have the honest answer "the platform does this for me — my job is to configure the per-nodepool policy and protect long jobs," and knowing that distinction is itself a signal of seniority.

**Cloud-portability / incident reality.** This isn't hypothetical risk theater — Google Cloud's own GKE has had a real driver-fetch failure incident (see Real-world use cases below). If a hyperscaler's own automated driver-install pipeline can produce a multi-hour outage, a hand-rolled fleet runbook without canary waves, health gates, and a rollback trigger is not a safe design, regardless of how good the individual `ClusterPolicy` YAML looks.

## Real-world use cases

- **[GKE can now automatically install NVIDIA GPU drivers](https://cloud.google.com/blog/products/containers-kubernetes/gke-can-now-automatically-install-nvidia-gpu-drivers) — Google Cloud.** GA'd March 2024; introduces the `gpu-driver-version=latest|default|disabled` node-pool flag. What it shows: a hyperscaler productizing exactly this lesson's problem — automated, versioned driver installation as a first-class GKE feature rather than a DIY DaemonSet, and the three-way choice (`latest`/`default`/`disabled`) mirrors the operator-managed vs. pre-installed decision above.
- **GKE GPU driver fetch failure (Google Cloud incident, reported ~March 12, 2024, ~6h55m, affecting T4/L4/A100/H100-80GB GPU node pools).** Referenced via `status.cloud.google.com/incidents/aRSt8sTQLKMTVgdbbK6P` — **not independently fetched this session** (proxy-blocked); found via an independent search snippet citing the specific date, duration, and affected GPU types. Treat the exact duration as unverified until you can re-check the incident page yourself. What it shows: even Google's own automated driver-install pipeline (the feature in the post above) had a real multi-hour failure mode — a concrete argument for canary waves and a rollback trigger rather than trusting automation blindly.
- **[CoreWeave — GPU driver management for CKS node pools](https://docs.coreweave.com/docs/products/cks/nodes/gpu-driver-management/update-gpu-driver)** — **not independently fetched this session** (proxy-blocked); corroborated across two independent search results, with a CoreWeave changelog entry dated August 15, 2025 introducing these driver-management features. What it shows: a real GPU-cloud vendor's per-nodepool driver-version-selection and job-aware rolling-update design — the managed-cloud alternative to running your own upgrade controller.
- **[NVIDIA/gpu-operator#1220 — "gpu-operator breaks when upgrading EKS to K8s v1.30"](https://github.com/NVIDIA/gpu-operator/issues/1220)** — fetched directly. A real practitioner incident: GPU Operator v24.6.2, driver 535.183.01, Ubuntu 22.04 kernel 6.5.0-1020-aws, worked fine on K8s 1.29, broke on 1.30 with `failed to get sandbox runtime: no runtime for 'nvidia' is configured` (runtime-wiring failure) and `Could not resolve Linux kernel version` (driver kernel-mismatch failure). What it shows: a driver/runtime upgrade risk that isn't about the GPU Operator's own version at all — a managed-Kubernetes control-plane upgrade can silently break the exact toolkit/driver wiring this lesson depends on, which is why a fleet upgrade runbook should treat "we also bumped K8s" as a variable, not a constant.

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

> **Which restart are we about to see?** `550.127.08 → 570.148.08` is a real version bump, so `pod-restart-required` below is the full unload/rebuild/load path — the v26.3+ same-version fast path does **not** apply here. If instead you re-applied `570.148.08 → 570.148.08` (e.g., to pick up a config-only field), on GPU Operator ≥26.3.0 this same step would complete in seconds by reusing the loaded module, but every other state (cordon, drain, validate, uncordon) would still run — the fast path shortens one step's cost, not the pipeline.

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

**5. Confirm the module actually reloaded** (the check the Operator's validator automates):
```bash
$ kubectl -n gpu-operator exec nvidia-driver-daemonset-xyz -- nvidia-smi \
    --query-gpu=driver_version --format=csv,noheader
570.148.08
$ kubectl -n gpu-operator logs nvidia-operator-validator-... -c driver-validation
driver is not ready... / all validations successful
```

If the node parks at `validation-required` or `upgrade-failed`, jump to Self-check (c).

## Practice

This task feeds the deliverable's **failure-mode log** and produces the **200-node runbook** artifact, both landing in [Per-pod GPU attribution](../practice/per-pod-attribution/README.md).

**Part A — observe the machine (single node).** On your lab cluster (a single GPU node is fine; kind + a cloud GPU VM, or one node of a managed GPU pool):
1. Record the current `spec.driver.version` and the node's `nvidia.com/gpu-driver-upgrade-state`.
2. Schedule a marker GPU pod (`nvidia-smi -l 5` in a loop) so the node is *in use* when you trigger the upgrade.
3. Patch the `ClusterPolicy` to a new driver version with `maxParallelUpgrades=1`, `maxUnavailable=1`.
4. Capture the full ordered sequence of `nvidia.com/gpu-driver-upgrade-state` values with timestamps (`kubectl get node -w` piped to a file), plus the cordon (`SchedulingDisabled`) transition and the driver-pod restart. Confirm your marker pod was evicted at `drain-required` and rescheduled after `uncordon-required`. Note in your log whether this was a real version bump or a same-version reconcile, and how long `pod-restart-required` actually took — this is your first-hand data point on the v26.3+ behavior split.

**Part B — write the 200-node rollout runbook.** Produce a runbook that covers, explicitly:
- **Batching:** chosen `maxParallelUpgrades` and `maxUnavailable` with the reasoning (e.g. `maxUnavailable: 5%` → ≤10 nodes offline, `maxParallelUpgrades: 4`), and the topology constraint (never drain two nodes of the same NVLink/rail domain or the same gang-scheduled job at once).
- **Cordon/drain policy:** `drain.enable`, `force`, `deleteEmptyDir`, `timeoutSeconds`, and the `waitForCompletion.podSelector` you use to *not evict long training jobs*.
- **Long-job protection:** how you label long runs, how the wait gate interacts with checkpointing, and what you tell job owners (checkpoint cadence expectation before the window).
- **Health gates:** what must be green before the controller advances a batch — `nvidia-operator-validator` success, DCGM health (`dcgmi diag -r 1` or the exporter's health metric), node `Ready`, and a canary GPU pod passing.
- **Rollback trigger:** the concrete condition that stops the rollout (e.g. >1 node hits `upgrade-failed`, or DCGM XID errors spike) and the rollback action (repin `spec.driver.version` to the old tag — the controller rolls the same machine back).
- **Canary → wave plan:** 1 node → 5% → 25% → 100%, with a soak time and the metric watched at each wave.
- **Vendor-boundary note:** one line stating whether this fleet runs its own upgrade controller or whether a managed-cloud layer (e.g. CKS-style per-nodepool driver management) already owns part of this — and what you'd change about the runbook if it does.

**Acceptance:**
- The observed, timestamped state-machine transitions from Part A (including the cordon and driver-pod restart, and the real-bump-vs-same-version note), saved to the failure-mode log.
- A written 200-node upgrade runbook containing *all seven* bullet categories above, each with concrete values (not "TBD"), a named rollback trigger, and an explicit statement of how long training jobs avoid eviction.

## Common pitfalls

1. **Assuming every driver-pod restart is a multi-minute kernel rebuild.** False on GPU Operator ≥26.3.0 for the *same-version* case — the module is reused and the restart completes in seconds. Real version bumps still walk the full unload/rebuild/load path. Check `spec.driver.version` old-vs-new before estimating downtime, don't assume from the pod's `Restarting` status alone.
2. **Not knowing your GPU-cloud vendor may already own this layer.** On a managed GPU cloud (CoreWeave CKS and similar), driver lifecycle may be a platform-owned feature with per-nodepool version selection, not something you're expected to run your own `ClusterPolicy` upgrade controller against. Running a self-managed Operator install redundantly against a walled managed layer is wasted engineering and a real source of drift.
3. **Confusing version skew with a crash.** A driver-pod restart that leaves the *old* kernel module loaded (because a container held it open and drain didn't fully clear) produces a userspace/kernel version mismatch on every `cudaInit()` — the node is `Ready`, every GPU pod crash-loops, and it looks like an application bug until you check `nvidia-smi --query-gpu=driver_version` against `spec.driver.version`.
4. **Treating `kernelModuleType: auto` as version-independent.** `auto` resolves differently depending on the driver branch it's paired with — open on a current branch, proprietary on an older pinned LTS line the open module doesn't cover. State the branch explicitly in any runbook; the same YAML means different things on different `driver.version` values.
5. **Sizing `maxUnavailable`/`maxParallelUpgrades` without a topology constraint.** The two knobs bound blast radius numerically, but neither one knows about NVLink/rail domains or gang-scheduled jobs — a batch that's numerically safe (5% of nodes) can still take out an entire multi-node training job if those 5% happen to be the same job's ranks. The runbook needs an explicit topology rule the YAML can't express on its own.

## Self-check

- Walk the driver-upgrade state machine end to end — what does each state do, and which state is where a stuck `PodDisruptionBudget` shows up? **Answer:** In order via `nvidia.com/gpu-driver-upgrade-state`: `upgrade-required` (driver pod out of date, node queued) → `cordon-required` (mark node Unschedulable) → `wait-for-jobs-required` (optionally wait for Job pods to finish) → `pod-deletion-required` (optionally delete GPU pods directly) → `drain-required` (drain the node to release the kernel module; skipped if pod-deletion cleared all GPU pods) → `pod-restart-required` (restart the `nvidia-driver-daemonset` pod so the old module unloads and the new one loads — fast/module-reuse if same-version on ≥26.3.0, full rebuild otherwise) → `validation-required` (`nvidia-operator-validator` runs driver/toolkit/CUDA/MIG checks) → `uncordon-required` (mark node Schedulable) → `upgrade-done` (steady state). A stuck PDB shows up at `drain-required`: if eviction can't complete before `drain.timeoutSeconds` expires, the node goes straight to `upgrade-failed` instead of hanging indefinitely.
- Open vs. proprietary kernel modules — when would you pick each, and what's the one caveat you must always state alongside the choice? **Answer:** **Open** (`nvidia-open`) is required on Grace Hopper/Blackwell and preferred on Hopper; pick it whenever you need DMA-BUF, GPUDirect RDMA, or GPUDirect Storage, or want forward compatibility on a current-branch fleet. **Proprietary** (`nvidia`, legacy closed module) is for older datacenter GPUs the open module doesn't support, or a validated legacy stack (or third-party kernel module) you can't requalify yet. The caveat: `driver.kernelModuleType: auto` resolves differently depending on the driver *branch* it's paired with — always state the branch (e.g. "R580, open modules") alongside the module-type choice, because the same `auto` setting means something different on an older pinned LTS line.
- A node is stuck in `validation-required` after an upgrade — what do you check, in order? **Answer:** (1) `nvidia-operator-validator` pod logs on that node — which sub-validation failed (driver, toolkit, cuda, plugin, MIG, workload). (2) The driver DaemonSet pod: is it `Running`, and does `nvidia-smi` inside it report the *new* version? If it's crash-looping, read its logs for a kernel-module build/load failure (kernel-headers mismatch, DKMS failure, Secure Boot signing). (3) `dmesg`/host kernel log for `nvidia.ko` load errors or a still-loaded *old* module (refcount stuck because a CUDA process didn't die during drain — check for leftover GPU pods that escaped eviction). (4) MIG state: if the node is MIG-enabled, a bad `mig-manager` state fails the MIG validation — check `nvidia-smi mig -lgi`. (5) Toolkit/device-plugin pods for the toolkit-validation step. Fix root cause, delete the validator pod to re-run; if unrecoverable, roll `spec.driver.version` back to trigger the reverse transition.
- Your fleet just moved from K8s 1.29 to 1.30 and GPU pods start crash-looping with no driver-version change at all — what's the first thing you check, and why? **Answer:** Whether the container-runtime/toolkit wiring survived the control-plane upgrade — specifically `failed to get sandbox runtime: no runtime for 'nvidia' is configured` (RuntimeClass/containerd config drift) or a kernel-version-resolution failure, both real symptoms from the fetched `NVIDIA/gpu-operator#1220` incident on an EKS 1.29→1.30 upgrade. The reason to check this first: a "we only bumped Kubernetes" mental model is wrong — a managed-K8s control-plane upgrade can change the node image, kernel, or containerd config underneath the GPU Operator's assumptions, breaking driver/toolkit wiring without anyone touching `spec.driver.version`.

## Connections & what's next

This lesson's cordon/drain/validate vocabulary is the general pattern the GPU Operator uses for *any* disruptive per-node change — lesson 06 reuses it verbatim for MIG reconfiguration, just tearing down GPU Instances instead of a kernel module. The `NVIDIADriver` CRD introduced here for heterogeneous fleets resurfaces in lesson 09 (DRA), where multi-driver-version awareness matters again for mixed-generation clusters running the DRA driver. The economics thread is the same one module 01 started: every minute a node sits cordoned or draining is a fully-billed idle GPU-hour, which is exactly why `maxParallelUpgrades`/`maxUnavailable` are cost knobs, not just safety knobs.

Next: **[04.6 · MIG operations](06-mig-operations.md)** takes the same "you cannot mutate this while a client holds it open" constraint from this lesson and applies it to a GPU's *partition layout* — reconfiguring MIG geometry is a drain-and-tear-down operation for exactly the same reason a driver restart is, just one layer lower in the stack (GPU Instances instead of a kernel module).

## References & further reading

**Primary sources**
- NVIDIA GPU Operator (repo) — https://github.com/NVIDIA/gpu-operator — fetched this session; read for the exact component chain, `ClusterPolicy` shape, and current release line.
- GPU Operator — GPU Driver Upgrades docs — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-driver-upgrades.html — not independently fetched this session (proxy-blocked); canonical state-machine and `upgradePolicy` field reference — re-verify field names at study time.
- GPU Operator — ClusterPolicy / `driver` spec reference — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html — not independently fetched this session (proxy-blocked); read for `version`, `kernelModuleType`, `upgradePolicy`, `drain` field definitions.
- GPU Operator Release Notes — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html — not independently fetched this session (proxy-blocked); the source to check before pinning any version number from this lesson into a real runbook.

**Real-world engineering blogs**
- Google Cloud, ["GKE can now automatically install NVIDIA GPU drivers"](https://cloud.google.com/blog/products/containers-kubernetes/gke-can-now-automatically-install-nvidia-gpu-drivers) — fetched this session. GA March 2024; a hyperscaler productizing automated driver installation.
- Google Cloud incident status page — `https://status.cloud.google.com/incidents/aRSt8sTQLKMTVgdbbK6P` — not independently fetched this session (proxy-blocked; found via search snippet). GKE GPU driver fetch failure, reported ~6h55m, March 12 2024 — re-verify date/duration before citing as fact.
- CoreWeave, ["GPU driver management for CKS node pools"](https://docs.coreweave.com/docs/products/cks/nodes/gpu-driver-management/update-gpu-driver) — not independently fetched this session (proxy-blocked; corroborated across two independent searches, changelog dated Aug 15 2025). A managed GPU cloud's per-nodepool driver-version and "Protected Rolling Updates" design.
- NVIDIA/gpu-operator issue #1220, ["gpu-operator breaks when upgrading EKS to K8s v1.30"](https://github.com/NVIDIA/gpu-operator/issues/1220) — fetched this session. Real practitioner incident with exact log lines for the runtime-wiring and kernel-mismatch failure families.

**Deeper dives**
- [NVIDIA/gpu-operator GitHub Issues](https://github.com/NVIDIA/gpu-operator/issues) — not individually fetched (general issue tracker); the practical "search your exact log line" resource before assuming a driver-upgrade failure is novel — filter to your operator + Kubernetes version.
