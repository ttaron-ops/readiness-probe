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
sources: 11
---

# 04.5 · Driver Lifecycle & Rolling Upgrades

> **Concept.** The GPU driver is a kernel module with cluster-wide blast radius; upgrading it safely is a drain-orchestrated state machine, not a `kubectl set image`.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 04.4 dissected the injection layer and kept arriving at the same dependency: the `libcuda.so.<version>` bind-mounted into every GPU container, the `/dev/nvidia*` majors read from `/proc/devices`, the driver root at `/run/nvidia/driver`, and the versioned filenames baked into a persistent CDI spec all trace back to **one kernel module on the host**. That lesson gave you the per-container fixes — enable the forward-compat hook, pin the image. This lesson is the expensive fix: changing the module itself, under a fleet of nodes that are all currently running jobs.

Lessons 04.1–04.2 taught the GPU Operator's steady-state component chain (NFD → driver → toolkit → device-plugin/GFD/DCGM → validator). This lesson is what the Operator does when steady state has to *move*: a controller-driven per-node state machine that cordons, evicts, unloads, installs, reloads, validates, and uncordons — with a specific blocking condition at every step. It also introduces the vocabulary lesson 04.6 reuses for a different disruptive change: repartitioning silicon instead of swapping a kernel module.

## Why this matters

The NVIDIA kernel driver is the single most dangerous thing to change on a GPU node, and the reason is a two-line fact about Linux kernel modules.

A module has a reference count. `delete_module(2)` — what `rmmod` calls — refuses with `EBUSY` if that count is non-zero. Every open file descriptor on `/dev/nvidia*` holds a reference, which means **every running CUDA process pins the driver.** The count is visible at `/sys/module/nvidia/refcnt`, and the module list at `/proc/modules` names who is holding it. There is no forcing it: no flag, no `--force`, no privileged escape. The only way to reach refcount zero is for every process holding a GPU to exit.

So a driver upgrade is a **tenant-eviction problem wearing a kernel-module costume**, and every mechanism in this lesson exists to make that eviction happen deliberately, on your schedule, rather than as a surprise.

Get it wrong at fleet scale and three failure modes cost real money:

- **Version skew halts the node while Kubernetes calls it healthy.** The userspace half (`libcuda.so`, `libnvidia-ml.so`) and the loaded module carry a version handshake that must match exactly (lesson 04.4 §8.1). If the driver container is replaced but the old module is still resident — because eviction did not fully clear, or because a stale rootfs is still mounted at `/run/nvidia/driver` — every new GPU container fails at NVML init. The node reports `Ready`, the scheduler keeps placing GPU pods on it, and they all crash-loop.
- **Silent destruction of long jobs.** A rollout that drains a node mid-epoch destroys the work of an uncheckpointed run. Worse, if that node holds one rank of a gang-scheduled 512-GPU job, all 512 GPUs sit idle waiting for the whole gang to reschedule — you paid for 512 and evicted 8.
- **A fleet that stalls in the dark.** The concurrency limits are computed from *cluster-wide unavailability*, not just from nodes this rollout touched. A handful of nodes cordoned for unrelated maintenance can drive the available-upgrade budget to zero, and the rollout simply stops making progress with no error anywhere.

This module's README names the probe explicitly: *"design a driver upgrade for 200 nodes."* That question separates someone who has *watched* a GPU Operator upgrade from someone who has *designed* a rollout with batching, blast-radius limits, long-job protection, health gates, and a rollback trigger — and who can answer "what if a PodDisruptionBudget blocks eviction on node 47?" with a state name and a timeout value rather than a shrug.

## What's new here (calibration)

Modules 02, 02b and 03 already own the theory this lesson would otherwise re-derive, and lesson 03.6 in particular already covers the CUDA/driver compatibility bands, the driver-branch lifecycle (NFB / Production / LTSB), the kernel-module-must-match-userspace rule, and the Xid fault channel. **None of that is re-taught here.** What is genuinely new:

- **The upgrade controller as a per-node state machine you can read live** off `kubectl get node -L`, including the two newer maintenance-operator states and the exact ordering the reconcile loop enforces.
- **The blocking condition and the timeout at every step**, with real defaults from the API types — including the hard-coded 600-second validation timeout that is not configurable.
- **The `upgradesAvailable` arithmetic, exactly.** `maxParallelUpgrades` and `maxUnavailable` do not compose the way people assume, and the formula counts nodes that have nothing to do with your rollout.
- **The safe-driver-load handshake** — an init container that blocks on a node annotation until the controller removes it. This is the mechanism that lets the driver container itself demand eviction before it loads, and it inverts the usual ordering.
- **The `DRIVER_CONFIG_DIGEST` fast path** — the real mechanism behind "not every driver-pod restart unloads the module", which is a config-digest comparison and not a version comparison.
- **Worked GPU-hour arithmetic** for a 200-node rollout, including the counter-intuitive result that the drained GPU-hour bill is independent of your parallelism setting, and the calculation that shows why long-job protection — not the driver — is what makes a fleet rollout take days.

## Core concepts

### 1. The physical operation you are orchestrating

Before any Kubernetes object, be clear about what must happen on one machine.

The NVIDIA driver on a Linux host is a set of out-of-tree kernel modules plus a matched userspace. The modules, and the order they must be removed in, are not a matter of taste — they have a dependency graph, and `k8s-driver-manager` encodes it (`cmd/driver-manager/main.go`):

```
  UNLOAD ORDER — dependents first, core last
    nvidia_modeset        display/modeset support
    nvidia_uvm            Unified Virtual Memory — every CUDA context uses this
    nvidia_peermem        GPUDirect RDMA peer-memory registration
    nvidia_fs             GPUDirect Storage
    nvidia_vgpu_vfio      vGPU passthrough
    gdrdrv                GDRCopy
    nvidia                the core module — everything above depends on it
```

For each of those, the manager checks `/sys/module/<name>/refcnt` exists, then calls `delete_module(2)`. If any call fails it logs `Could not unload NVIDIA driver kernel modules, driver is in use` and dumps the relevant lines of `/proc/modules` with a `Module / Size / Ref Count / Used by` header — which is exactly the evidence you need, because it names the holder.

That is worth internalising as the shape of the whole lesson: **the interesting failures are all "refcount is not zero", and the diagnostic is always `/proc/modules` plus "which process still has a GPU open".**

Two more physical steps beyond the modules, both of which cause real incidents when skipped:

- **Unmount the driver rootfs.** In the containerized-driver model the driver container bind-mounts its own filesystem at `/run/nvidia/driver`, and the toolkit's `root = "/run/nvidia/driver"` setting points every library lookup there. If the old mount survives a driver-container replacement, the toolkit discovers the *old* libraries against the *new* module. `k8s-driver-manager` therefore does a `RecursiveUnmount` of `/run/nvidia/driver` and removes `/run/nvidia/nvidia-driver.pid`, and it does this even on the "nothing was loaded" path precisely to clean up after a crashed predecessor.
- **Unload `nouveau`.** The open-source Nouveau driver claims the same PCI devices and blocks the NVIDIA module from loading. The manager checks `/sys/module/nouveau/refcnt` and unloads it if present.

### 2. Operator-managed vs pre-installed drivers

Two supported topologies. Know which one you are on before touching anything, because the playbooks share almost nothing.

**Operator-managed (`driver.enabled=true`, the default).** The GPU Operator runs an `nvidia-driver-daemonset`. Each pod has a `k8s-driver-manager` init container (image `nvcr.io/nvidia/cloud-native/k8s-driver-manager`, `v0.11.0` in GPU Operator v26.3.3) that performs the eviction-and-unload work of §1, followed by an `nvidia-driver-ctr` container that loads the new module and keeps the rootfs mounted. Upgrades are declarative: change `spec.driver.version`, and the controller in §3 does the rest. The DaemonSet update strategy is `OnDelete`, which is the linchpin — the *upgrade controller*, not the DaemonSet controller, decides when each pod is deleted, and that is how eviction is guaranteed to happen before the restart.

**Pre-installed / host driver (`driver.enabled=false`).** The driver is baked into the node image — a golden AMI, `bootc`, an image-mode RHEL build, a config-management run — and the Operator deploys only the toolkit, device plugin, GFD, DCGM and validators. Upgrades become a *node-image* problem: you roll new images through your normal node-pool rotation (a managed node-group version bump, a Cluster API `MachineDeployment` rollout). `k8s-driver-manager` detects this case explicitly: if it finds a host driver already installed it logs `NVIDIA GPU driver is already pre-installed on the node, disabling the containerized driver`, flips the deploy label, sleeps 60 seconds, and exits with an error so the driver pod does not fight the host.

The trade, stated plainly: operator-managed gives you fast declarative in-place bumps without reimaging; pre-installed gives immutable-infrastructure guarantees and a faster boot (no in-cluster module build or load step) at the cost of coupling driver changes to your image pipeline and node replacement. Large fleets frequently run pre-installed for the base driver and use the Operator purely for the toolkit/plugin/DCGM/MIG stack. **State which one a scenario assumes before answering an interview question about it.**

### 3. The upgrade state machine

When the driver version — or, more precisely, the driver *configuration digest*, §6 — changes, the upgrade controller labels every affected GPU node with `nvidia.com/gpu-driver-upgrade-state` and advances it. The label key is built from a format string, `nvidia.com/%s-driver-upgrade-state`, where `%s` is the driver type (`gpu`, or `vgpu` for the vGPU flavour).

| State | What happens | Blocks on |
|---|---|---|
| *(empty)* | Node not yet processed, or the upgrade flow is disabled | — |
| `upgrade-done` | Steady state: driver pod current and Ready, node schedulable | — |
| `upgrade-required` | Controller detected the running driver pod is out of date; node is queued | an available upgrade slot (§4) |
| `cordon-required` | Node marked `Unschedulable` so no new GPU pods land | nothing; a single API patch |
| `wait-for-jobs-required` | Optional gate: wait for pods matching `waitForCompletion.podSelector` to finish on their own | those pods reaching a terminal phase, or `timeoutSeconds` |
| `pod-deletion-required` | Optional: delete GPU pods directly (a filtered drain) rather than draining the node | `podDeletion.timeoutSeconds`; PDBs, unless `force` |
| `drain-required` | Full node drain via the `kubectl` drain helper | `drain.timeoutSeconds`; PDBs, unless `force`. **Skipped if pod-deletion already cleared every GPU pod** |
| `node-maintenance-required` | Cordon/drain delegated to an external maintenance operator | that operator completing. Only valid when `UseMaintenanceOperator` is on |
| `post-maintenance-required` | External maintenance finished; requestor performs post-maintenance actions | as above |
| `pod-restart-required` | The driver DaemonSet pod is deleted so a new one starts — **or**, if the node is waiting for a safe driver load, the blocking annotation is removed instead of restarting anything (§5) | module refcount reaching zero; module build/load |
| `validation-required` | Wait for the pod matching `app=nvidia-operator-validator` on this node to become Ready | **600 s hard-coded**, then `upgrade-failed` |
| `uncordon-required` | Node marked `Schedulable` again | nothing |
| `upgrade-failed` | Terminal failure; node parked for a human | — |

The controller's reconcile loop processes these in a fixed order every pass — `ApplyState` in `upgrade_state.go` calls the handlers in sequence: unknown → done → upgrade-required → cordon → wait-for-jobs → pod-deletion → drain → node-maintenance → pod-restart → upgrade-failed → validation → uncordon. It requeues itself every two minutes (`plannedRequeueInterval = time.Minute * 2`) in addition to reacting to watch events, so a stuck node will be re-evaluated on a predictable cadence rather than only on change.

Three details that matter operationally:

- **"GPU pod" has a precise definition.** The pod-deletion filter matches a pod in `Running` or `Pending` phase whose containers request any resource whose name starts with `nvidia.com/gpu` or `nvidia.com/mig-`, **or** which holds an NVIDIA GPU `ResourceClaim` (the DRA case). Everything else is left alone. So a pod that touches GPUs through some path the Operator cannot see — a hostPath-mounted `/dev/nvidia0` in a privileged pod, say — will not be evicted, and will hold the module open. That is a real way to wedge a rollout.
- **The drain helper is `kubectl`'s own.** `drain.Helper` with `IgnoreAllDaemonSets: true` (necessary — the driver itself is a DaemonSet), `GracePeriodSeconds: -1` (honour each pod's own `terminationGracePeriodSeconds`), and `Force`/`DeleteEmptyDirData`/`Timeout`/`PodSelector` from your spec. It uses the Eviction API where available, which is why PodDisruptionBudgets apply.
- **Drain skips pods labelled to opt out.** The controller uses the selector `nvidia.com/gpu-driver-upgrade-drain.skip!=true`, so labelling a pod `nvidia.com/gpu-driver-upgrade-drain.skip=true` exempts it from drain. Useful for infrastructure pods; dangerous if you exempt something that holds a GPU.

### 4. `maxParallelUpgrades` and `maxUnavailable` do not compose the way you think

You configure the machine's aggressiveness under `spec.driver.upgradePolicy`. Here are the **real defaults**, from the `DriverUpgradePolicySpec` kubebuilder markers, alongside what the GPU Operator Helm chart sets:

| Field | API default | Helm chart default | Meaning |
|---|---|---|---|
| `autoUpgrade` | `false` | `true` | Master switch; when false every other field is ignored and nodes park in `upgrade-required` |
| `maxParallelUpgrades` | `1` | `1` | Concurrency. **`0` means no limit** |
| `maxUnavailable` | `"25%"` | `25%` | Absolute number or percentage of total managed nodes; percentages round **up** |
| `waitForCompletion.timeoutSeconds` | `0` | `0` | `0` = wait forever |
| `waitForCompletion.podSelector` | `""` | `""` | Empty = no job-wait gate |
| `drain.enable` | **`false`** | **`false`** | Draining is **off by default** |
| `drain.force` | `false` | `false` | Evict pods not backed by a controller |
| `drain.timeoutSeconds` | `300` | `300` | Drain deadline |
| `drain.deleteEmptyDir` | `false` | `false` | Allow eviction of pods with `emptyDir` volumes |
| `drain.podSelector` | `""` | `""` | Limit drain to matching pods |
| `podDeletion.force` | `false` | `false` | (Helm key is **`gpuPodDeletion`**) delete GPU pods directly |
| `podDeletion.timeoutSeconds` | `300` | `300` | |
| `podDeletion.deleteEmptyDir` | `false` | `false` | |

**`drain.enable` defaulting to `false` is the single most commonly mis-stated fact about this API.** The Operator's own values.yaml comment explains the intent: *"options for node drain (`kubectl drain`) before the driver reload — this is required only if default GPU pod deletions done by the operator are not sufficient to re-install the driver."* The normal path is targeted GPU-pod deletion; a full node drain is the escalation. If your runbook says "the Operator drains the node", verify that `drain.enable: true` is actually set, because out of the box it is not.

Note also the Helm-vs-CR key mismatch: the CR field is `podDeletion`, the chart value is `gpuPodDeletion`. Writing `podDeletion` in a `values.yaml` produces a silently ignored block.

#### 4.1 The `upgradesAvailable` formula, exactly

This is the arithmetic that decides whether your rollout moves. It is short enough to read in full (`GetUpgradesAvailable` in `common_manager.go`, plus its caller):

```
  maxUnavailable = scaled(policy.maxUnavailable, totalManagedNodes, roundUp=true)

  upgradesInProgress = totalNodes − ( |unknown| + |upgrade-done| + |upgrade-required| )

  if maxParallelUpgrades == 0:
      upgradesAvailable = |upgrade-required|            # no limit
  else:
      upgradesAvailable = maxParallelUpgrades − upgradesInProgress

  currentUnavailableNodes = (# managed GPU nodes that are CORDONED or NOT-READY)
                            + |cordon-required|

  if upgradesAvailable > maxUnavailable:
      upgradesAvailable = maxUnavailable

  if currentUnavailableNodes >= maxUnavailable:
      upgradesAvailable = 0                             # ◀── the stall
  elif maxUnavailable < totalNodes and
       currentUnavailableNodes + upgradesAvailable > maxUnavailable:
      upgradesAvailable = maxUnavailable − currentUnavailableNodes
```

Read the consequences carefully, because two of them are counter-intuitive and both cause real "the rollout is stuck and nothing is logging an error" incidents.

**`currentUnavailableNodes` counts every managed GPU node that is cordoned or NotReady, for any reason whatsoever.** It is computed by walking all node states and testing `node.Spec.Unschedulable` and the `Ready` condition. A node you cordoned by hand last week for a hardware investigation counts. A node that is `NotReady` because its kubelet is wedged counts. So on a 200-node fleet with `maxUnavailable: 5%` (= 10 nodes), if 10 nodes are already cordoned or NotReady for unrelated reasons, `upgradesAvailable` is **zero** and the rollout never starts. There is one debug-level log line — `Node upgrade limit reached, pausing further upgrades` — and no event, no condition, no error.

**`upgradesInProgress` is "everything that is not idle".** It is derived by subtraction: total minus unknown minus done minus upgrade-required. So a node parked in `wait-for-jobs-required` for four hours waiting on a long training job counts as an upgrade in progress and consumes one of your `maxParallelUpgrades` slots for the whole four hours. Long-job protection and rollout throughput are in direct tension by construction.

**A node that is already unschedulable proceeds even when there are no slots.** `ProcessUpgradeRequiredNodes` has an explicit carve-out: when `upgradesAvailable <= 0`, a node that is *already* cordoned still advances, with the log line `Node is already cordoned, progressing for driver upgrade`. The reasoning is sound — the capacity is already lost, so finishing the upgrade is strictly better than leaving it half-done — but it means manual cordoning is a way to push a node through a saturated rollout.

**Two nodes can also be skipped deliberately.** `nvidia.com/gpu-driver-upgrade.skip=true` on a node makes the controller log `Node is marked for skipping upgrades` and leave it alone. `nvidia.com/gpu-driver-upgrade-requested` as an annotation forces a node into `upgrade-required` — the manual trigger, also used to recover orphaned driver pods.

#### 4.2 What the two knobs actually bound

Say it precisely, because interviewers probe exactly this:

- **`maxParallelUpgrades` bounds concurrency** — how many nodes are simultaneously somewhere between `cordon-required` and `uncordon-required`. It sets **wall-clock duration**.
- **`maxUnavailable` bounds impact** — how much of the fleet may be simultaneously cordoned or NotReady. It sets **peak capacity dip**, and it is a ceiling the first knob is clamped to.

The effective batch size is the smaller of the two, further reduced by unavailability you did not cause. Set both: the first for throughput, the second as the safety ceiling. And note what neither knob knows about: **NVLink/rail domains and gang-scheduled jobs.** A numerically safe 5% batch can still be five ranks of one 512-GPU job. Nothing in this API can express that constraint; it has to live in your rollout tooling or in your node labelling.

### 5. Safe driver load: the handshake that inverts the ordering

There is a chicken-and-egg problem in the containerized-driver model. The controller wants to evict workloads *before* the module is replaced. But the thing that knows the module needs replacing is the driver container, and by the time it is running the pod has already been scheduled — nothing evicted anything.

The safe-driver-load mechanism solves it with a two-step handshake using a node annotation, `nvidia.com/gpu-driver-upgrade.driver-wait-for-safe-load`:

```
  SAFE DRIVER LOAD — the driver asks to be let in

  driver pod starts on node N
      │
      ├─ init container: set annotation
      │     nvidia.com/gpu-driver-upgrade.driver-wait-for-safe-load = "<value>"
      │     …then BLOCK, polling until the annotation disappears
      │
      ▼
  upgrade controller sees the annotation
      │  IsWaitingForSafeDriverLoad(node) == true
      │
      ├─ unconditionally moves the node to  upgrade-required
      ├─ walks cordon → wait-for-jobs → pod-deletion → drain
      │      (per the configured upgradePolicy — the SAME gates as a version bump)
      │
      ▼
  node reaches  pod-restart-required
      │
      ├─ node is waiting for safe load, so instead of deleting the driver pod:
      │      UnblockLoading(node) → remove the annotation
      │
      ▼
  init container's poll succeeds → exits 0
      │
      ▼
  nvidia-driver-ctr loads the kernel module into a node with NO GPU clients
      │
      ▼
  validation-required → uncordon-required → upgrade-done
```

Two things fall out of this that you should be able to state cold.

First, **`pod-restart-required` is not always a pod restart.** On a safe-load node it is an annotation removal. The state name is historical; the state's job is "do the post-eviction action, whatever that is for this node". The upstream code even carries a TODO noting that `pod-restart-required` will eventually be replaced by the more general `post-maintenance-required`.

Second, **safe-load nodes always take the full flow.** `shouldRestartOnly` explicitly returns false when `IsWaitingForSafeDriverLoad` is true, with the comment that such nodes "must take the full flow so workloads are evicted before the load is unblocked at pod-restart-required". A node asking to load a driver safely can never be fast-pathed.

### 6. The fast path: `DRIVER_CONFIG_DIGEST`, not a version comparison

Not every out-of-date driver pod needs a full eviction cycle. If the DaemonSet template changed in a way that does not affect *how the driver is installed* — a `helm.sh/chart` label bump, a tolerations tweak, an unrelated `ClusterPolicy` field — then restarting the pod in place, without evicting a single workload, is correct and takes seconds instead of minutes.

The mechanism is a content digest, and it is worth knowing precisely because "same version means fast restart" is a near-miss description of it.

On the Operator side, `TransformDriver` runs after every other transformation, extracts the install-relevant fields from the finished pod spec via `extractDriverInstallConfig`, hashes them, and injects the result as the environment variable **`DRIVER_CONFIG_DIGEST`** into both the `k8s-driver-manager` init container and the `nvidia-driver-ctr` container. Fields such as `kernelModuleType` and proxy settings are captured implicitly, because they appear as container environment variables inside the hashed spec.

That digest is then consumed in two places:

**In the controller**, as a routing predicate. `DriverPodRestartOnly` compares the digest on the *running* pod's spec against the digest on the *desired* DaemonSet template. If both are present and equal, the node takes the restart-only path: it is **still cordoned** (so it stays unschedulable if the restart fails, exactly as in the full flow), then goes straight to `pod-restart-required`, skipping pod-deletion and drain. The log line is `Restart-only change detected; cordoning node and restarting driver pod in place, skipping pod-deletion and drain`. If either digest is missing, it logs `driver config digest missing; taking full upgrade flow` and does the safe thing.

**In `k8s-driver-manager`**, as a skip-uninstall check. The manager reads `DRIVER_CONFIG_DIGEST` from its environment and compares it against a digest persisted on the host at `/run/nvidia/nvidia-driver.state`. If they match and `FORCE_REINSTALL` is not set, it logs `The NVIDIA driver is already loaded with the desired version and configuration, skipping the uninstallation of the driver in an attempt to not disrupt running workloads` and leaves the module alone — but it still unmounts the stale rootfs, removes the stale PID file, and (on a DRA cluster) restarts the kubelet plugin, because those artefacts belong to the *previous container* and would otherwise pin the plugin to a filesystem that is about to disappear.

Two corrections to the folklore this replaces:

- It is a **digest comparison, not a version comparison.** Re-applying the same `driver.version` with a *different* `kernelModuleType` produces a different digest and therefore the full flow — correctly, because the module genuinely has to be rebuilt.
- The fast path **does not skip the whole pipeline.** The node is still cordoned, still validated, still uncordoned. What it skips is exactly two states: `pod-deletion-required` and `drain-required`. So the saving is "no workload eviction", which is the expensive part, but the node still goes briefly unschedulable and still consumes an upgrade slot.

The practical rule: **do not estimate downtime from a driver pod's `Restarting` status.** Diff the digest, or read the controller log line, and know which of the two paths you are watching.

### 7. Kernel module flavour: `kernelModuleType`

NVIDIA ships the kernel module in two flavours built from the same driver branch, selected by `spec.driver.kernelModuleType`, which is `+kubebuilder:validation:Enum=auto;open;proprietary` with `+kubebuilder:default=auto`.

- **`open`** — the *open GPU kernel modules*, GPL/MIT-licensed source, still paired with a closed GSP (GPU System Processor) firmware blob. This is the flavour required on Grace Hopper / Blackwell-class parts, and it is the substrate for DMA-BUF, which GPUDirect RDMA and GPUDirect Storage increasingly depend on.
- **`proprietary`** — the historical closed module. Still needed for older architectures the open module does not cover.
- **`auto`** — resolve based on the GPUs present *and the driver branch in use*. The API comment is explicit: *"If auto is chosen, it means that the recommended kernel module type is chosen based on the GPU devices on the host and the driver branch used."*

**`auto` is therefore not version-independent, and that is the caveat to state in any runbook alongside the setting.** The same `kernelModuleType: auto` next to a current branch resolves to `open`; next to an older pinned LTSB that the open module never covered, it resolves to `proprietary`. Two `ClusterPolicy` files with identical `kernelModuleType` lines can install different modules. Always write the branch beside the module type: "R580 LTSB, open modules", not "open modules".

A related deprecation to know: `useOpenKernelModules` still exists in the API but is marked *"Deprecated: This field is no longer honored by the gpu-operator. Please use KernelModuleType instead."* Setting it does nothing. `usePrecompiled` (pre-built modules for specific kernels, avoiding an in-cluster build) is separate, defaults to `false`, and — in the `NVIDIADriver` CRD — is marked **immutable** with the CEL validation message *"usePrecompiled is an immutable field. Please create a new NvidiaDriver resource instead when you want to change this setting."* You cannot flip a fleet between built-in-cluster and precompiled in place.

For the CUDA-family minimum driver versions, the minor-version-compatibility band, and the NFB/Production/LTSB branch lifecycle, see lesson 03.6 — those are its subject and are not repeated here. The one branch fact worth restating because it anchors a runbook: **R580 is an LTSB with support through August 2028**, whereas an NFB such as R590 reaches end of life inside about eighteen months (R590 Linux is dated 22 December 2026). Pin to an LTSB unless you have a concrete reason not to.

### 8. `NVIDIADriver`: heterogeneous fleets

`ClusterPolicy` assumes **one** driver configuration for the whole cluster. That breaks the moment your fleet is mixed — some nodes are Blackwell parts that require a current branch and open modules, some are older parts pinned to a validated LTSB for a workload you cannot requalify, some run Ubuntu 22.04 and some 24.04.

The `NVIDIADriver` custom resource (API group `nvidia.com`, version **`v1alpha1`**) is the answer. You create one or more `NVIDIADriver` objects, each with its own `version`, `kernelModuleType`, `usePrecompiled`, `driverType` (`gpu` | `vgpu` | `vgpu-host-manager`) and `upgradePolicy`, and each scoped by a `nodeSelector`. The Operator then runs one driver DaemonSet per matching object, and the upgrade controller reconciles upgrades **per `NVIDIADriver` instance** rather than cluster-wide.

Facts to get right, because the old story about this feature is wrong in two places:

- **It is not new in v26.x.** The CRD was introduced in GPU Operator **v23.9.0** as a technology preview, explicitly to support multiple driver types/versions and multiple OS versions on one cluster.
- **It is still not GA.** The API version is `v1alpha1`, `driver.nvidiaDriverCRD.enabled` defaults to `false`, and *"Promote the NVIDIADriver CRD to General Availability (GA)"* is a **roadmap item** in the GPU Operator's own README as of v26.3.3. Treat it as a real feature you can build on, but pin the Operator version and read that version's release notes.
- **Some fields are immutable.** `driverType` and `usePrecompiled` both carry `self == oldSelf` CEL validation with the message telling you to create a new resource instead. Changing them means a new object and a migration, not an edit.
- **Node selectors are validated.** `ValidateNodeSelector` rejects a `nodeSelector` on the object marked `default: true`, and rejects any selector that uses the Operator's own routing label. The Operator labels nodes with an owner label to route them to exactly one driver object; hand-writing that label breaks the routing.

One incompatibility worth knowing: the upgrade controller **refuses to run the advanced upgrade policy when `sandboxWorkloads.enabled=true`**, logging *"Advanced driver upgrade policy is not supported when 'sandboxWorkloads.enabled=true' in ClusterPolicy, cleaning up upgrade state and skipping reconciliation"* and removing the upgrade-state labels. If you run KubeVirt/vGPU workloads alongside containers, the state machine in §3 is simply not active, and every fact about it stops applying.

### 9. The full node sequence, with the blocking condition at each step

```
  ONE NODE THROUGH A REAL DRIVER UPGRADE
  left column: state label   ·   right column: WHAT BLOCKS, and for how long

  spec.driver.version 580.65.06 → 595.71.05   (digest changes)
        │
        ▼
  ┌───────────────────────┐
  │ upgrade-required      │◀── BLOCKS on: an upgrade slot.
  └───────────┬───────────┘    upgradesAvailable = min(maxParallelUpgrades −
              │                  inProgress, maxUnavailable − unavailableNodes)
              │                ⚠ unavailableNodes counts nodes cordoned/NotReady
              │                  for ANY reason, fleet-wide.  If it is already
              │                  ≥ maxUnavailable → 0 slots → silent stall.
              │                Escape hatch: an already-cordoned node proceeds.
              ▼
  ┌───────────────────────┐
  │ cordon-required       │──▶ node.spec.unschedulable = true
  └───────────┬───────────┘    BLOCKS on: nothing. One API patch.
              │                (prior schedulability is remembered in the
              │                 …node-initial-state.unschedulable annotation
              │                 so an already-cordoned node is not uncordoned)
              ▼
  ┌───────────────────────┐
  │ wait-for-jobs-required│    BLOCKS on: every pod matching
  │  (only if podSelector │      waitForCompletion.podSelector leaving
  │   is set)             │      Running/Pending — i.e. the JOB FINISHING.
  └───────────┬───────────┘    timeoutSeconds = 0 (default) → WAIT FOREVER.
              │                Start time is stamped in the
              │                  …-wait-for-pod-completion-start-time annotation.
              │                💰 the node is CORDONED for this entire wait and
              │                   consumes one slot.  A 40-hour job = 40 hours
              │                   of one slot. This is what makes rollouts take
              │                   days. GPUs are NOT idle though — the job runs.
              ▼
  ┌───────────────────────┐
  │ pod-deletion-required │    BLOCKS on: eviction of pods that request
  │  (only if podDeletion │      nvidia.com/gpu* or nvidia.com/mig-* or hold an
  │   is configured)      │      NVIDIA ResourceClaim.  PodDisruptionBudgets
  └───────────┬───────────┘      apply unless force=true.
              │                timeoutSeconds = 300.  On expiry → drain-required
              │                  if drain.enable, else → upgrade-failed.
              │                💰 GPUs go IDLE from here on.  Clock starts.
              ▼
  ┌───────────────────────┐
  │ drain-required        │    SKIPPED if pod-deletion cleared every GPU pod.
  │  (drain.enable=false  │    BLOCKS on: kubectl drain helper completing.
  │   BY DEFAULT!)        │      IgnoreAllDaemonSets=true, GracePeriod=-1
  └───────────┬───────────┘      (each pod's own terminationGracePeriodSeconds),
              │                  selector nvidia.com/gpu-driver-upgrade-drain.skip!=true
              │                timeoutSeconds = 300 → upgrade-failed.
              │                ⚠ THIS is where a stuck PodDisruptionBudget lands.
              ▼
  ┌───────────────────────┐
  │ pod-restart-required  │    Two behaviours:
  └───────────┬───────────┘     (a) normal: delete the driver DaemonSet pod
              │                     (strategy OnDelete, so only the controller
              │                      ever deletes it)
              │                 (b) safe-load node: remove the
              │                     …driver-wait-for-safe-load annotation
              │                     instead — the blocked init container proceeds
              │
              │   ┌─────────── INSIDE THE NEW DRIVER POD ──────────────────────┐
              │   │ k8s-driver-manager (init):                                  │
              │   │   /sys/module/nvidia/refcnt exists?  → driver is loaded     │
              │   │   DRIVER_CONFIG_DIGEST == /run/nvidia/nvidia-driver.state?  │
              │   │        yes → skip uninstall, just clean stale mounts        │
              │   │        no  → continue:                                      │
              │   │   delete_module(nvidia_modeset, nvidia_uvm, nvidia_peermem, │
              │   │                 nvidia_fs, nvidia_vgpu_vfio, gdrdrv,        │
              │   │                 nvidia)                                     │
              │   │        ⚠ BLOCKS HERE if refcount ≠ 0 → EBUSY.               │
              │   │          logs "driver is in use" + /proc/modules dump.      │
              │   │          fallback: full node drain, then retry once.        │
              │   │   RecursiveUnmount /run/nvidia/driver                       │
              │   │   rm /run/nvidia/nvidia-driver.pid                          │
              │   │   unload nouveau if /sys/module/nouveau/refcnt exists        │
              │   │ nvidia-driver-ctr:                                          │
              │   │   build or unpack modules, insmod, mount new rootfs         │
              │   │        ⚠ BLOCKS/FAILS on kernel-header mismatch, DKMS build │
              │   │          failure, Secure Boot signing, or the wrong          │
              │   │          precompiled tag for the running kernel.            │
              │   └─────────────────────────────────────────────────────────────┘
              ▼
  ┌───────────────────────┐
  │ validation-required   │    BLOCKS on: the pod matching app=nvidia-operator-
  └───────────┬───────────┘      validator on this node being Running with ALL
              │                  containers Ready.  Its init containers run in
              │                  order and each writes /run/nvidia/validations/
              │                  <component>-ready:
              │                     driver → toolkit → cuda → plugin
              │                  then the main container prints
              │                  "all validations are successful".
              │                TIMEOUT = 600 s, HARD-CODED, not configurable
              │                  (validationTimeoutSeconds in validation_manager.go)
              │                  → upgrade-failed.
              ▼
  ┌───────────────────────┐
  │ uncordon-required     │──▶ node.spec.unschedulable = false
  └───────────┬───────────┘    unless the initial-state annotation says the node
              │                was already unschedulable before the upgrade.
              ▼
  ┌───────────────────────┐          ┌──────────────────┐
  │ upgrade-done          │          │  upgrade-failed  │◀── any timeout above
  └───────────────────────┘          └──────────────────┘    terminal; needs a
                                                             human, or a rollback
                                                             of spec.driver.version
  💰 IDLE GPU-HOUR WINDOW = pod-deletion-required → uncordon-required
     everything before that is cordoned-but-productive.
```

### 10. Health gates you can actually query

"Wait for the validator" is the Operator's own gate. For a runbook you want independent signals. The Operator exports these Prometheus gauges under the `gpu_operator` namespace — real metric names, read from `controllers/operator_metrics.go`:

| Metric | Meaning |
|---|---|
| `gpu_operator_gpu_nodes_total` | number of nodes with GPUs |
| `gpu_operator_driver_auto_upgrade_enabled` | 1 if auto upgrade is enabled, 0 if not |
| `gpu_operator_nodes_upgrades_in_progress` | nodes currently mid-upgrade |
| `gpu_operator_nodes_upgrades_done` | nodes successfully upgraded |
| `gpu_operator_nodes_upgrades_failed` | nodes whose upgrade failed |
| `gpu_operator_nodes_upgrades_available` | nodes on which an upgrade *can* start right now |
| `gpu_operator_nodes_upgrades_pending` | nodes queued |
| `gpu_operator_reconciliation_status` | 0 = healthy; non-zero encodes operands-not-ready / policy-unavailable / operator error |
| `gpu_operator_reconciliation_has_nfd_labels` | 1 if NFD's mandatory kernel labels were found |

Three PromQL expressions worth putting in a rollout dashboard:

```promql
# 1. Rollback trigger: more than one node has failed.
gpu_operator_nodes_upgrades_failed > 1

# 2. Silent stall detector: work remains, but no slots are available and
#    nothing is moving. This is the §4.1 maxUnavailable stall.
(gpu_operator_nodes_upgrades_pending > 0)
  and (gpu_operator_nodes_upgrades_available == 0)
  and (gpu_operator_nodes_upgrades_in_progress == 0)

# 3. Progress rate, for an ETA. Nodes completed per hour.
rate(gpu_operator_nodes_upgrades_done[30m]) * 3600
```

Expression 2 is the one that earns its keep. The stall in §4.1 produces no error, no event and no Kubernetes condition — only a debug log line. A gauge combination is the only cheap way to see it.

Add, from outside the Operator: node `Ready`, `nvidia-smi --query-gpu=driver_version` matching the target on the node's driver pod, a DCGM health check (`dcgmi diag -r 1` is the quick one), and a canary GPU pod that actually runs a CUDA kernel — because the validator's `cuda` stage proves initialisation, not throughput.

### 11. Worked math: what a 200-node rollout actually costs

Fix the fleet and the per-node timings, then compute. Every input is stated so you can re-run it with your own numbers.

```
  FLEET
    nodes                     N   = 200
    GPUs per node             g   = 8         (H100 SXM)
    total GPUs                    = 1,600
    blended rental price      p   = $2.50 / GPU-hour
                                    (H100 on-demand ranges roughly $2–$4/GPU-hr
                                     depending on provider and commitment; use
                                     your own contracted rate)

  PER-NODE TIMINGS  (measure these on one node before trusting them)
    cordon                        ≈   1 s     one API patch
    pod-deletion (60 s grace)     =  60 s     ← idle clock starts
    driver-pod delete + schedule  =  20 s
    module unload + unmount       =  15 s
    module install (precompiled)  = 150 s
    validator chain Ready         =  75 s
    uncordon                      ≈   1 s
    ────────────────────────────────────────
    T_idle (precompiled)          = 321 s ≈ 5.35 min
    T_idle (in-cluster DKMS build)= 321 + 360 = 681 s ≈ 11.35 min
```

**Step 1 — drained GPU-hours per node.**

```
  precompiled: 8 GPUs × 5.35 min ÷ 60 = 0.713 GPU-hours per node
  DKMS build : 8 GPUs × 11.35 min ÷ 60 = 1.513 GPU-hours per node
```

**Step 2 — fleet total.**

```
  precompiled: 200 × 0.713 = 142.7 GPU-hours   → × $2.50 = $357
  DKMS build : 200 × 1.513 = 302.7 GPU-hours   → × $2.50 = $757
```

**The result to internalise: the drained GPU-hour bill does not depend on `maxParallelUpgrades` at all.** It is `N × g × T_idle`, full stop. Parallelism moves the *when*, not the *how much*. This is the single most useful thing to say out loud in an interview when someone asks you to justify a batch size, because it reframes the trade: you are not choosing between cheap and expensive, you are choosing between *a long shallow dip* and *a short deep one*, at the same total cost.

**Step 3 — wall clock and peak dip, as a function of `maxParallelUpgrades` (P).** Take `maxUnavailable: 5%` = 10 nodes as the ceiling, so P is only meaningful up to 10.

| P | batches = ⌈200/P⌉ | wall clock (precompiled) | peak GPUs offline | peak dip |
|---|---|---|---|---|
| 1 | 200 | 200 × 5.35 min = **17.8 h** | 8 | 0.5% |
| 4 | 50 | 50 × 5.35 min = **4.5 h** | 32 | 2.0% |
| 8 | 25 | 25 × 5.35 min = **2.2 h** | 64 | 4.0% |
| 10 | 20 | 20 × 5.35 min = **1.8 h** | 80 | 5.0% |
| 20 | (clamped to 10) | **1.8 h** | 80 | 5.0% |

The last row is the point of the `min()` in §4.2: raising `maxParallelUpgrades` past `maxUnavailable` buys nothing. If you want the rollout faster you must raise the impact ceiling, deliberately.

**Step 4 — the cost that actually dominates: long-job protection.** Now add the realistic complication. Suppose 30% of nodes (60 nodes) host a job labelled `training=long-running`, with a mean remaining runtime of 4 hours, and you set `waitForCompletion.podSelector: training=long-running` with `timeoutSeconds: 0` (wait forever). Run with P = 4.

Each waiting node occupies a slot for its wait *plus* its upgrade:

```
  slot-time per waiting node  = 4 h  +  5.35 min  ≈ 4.09 h
  slot-time per fast node     =        5.35 min   ≈ 0.089 h

  total slot-time = 60 × 4.09 + 140 × 0.089
                  = 245.4 + 12.5 = 257.9 slot-hours

  wall clock with P = 4  =  257.9 ÷ 4  ≈  64.5 h  ≈  2.7 days
```

Compare that with the 4.5 hours from step 3. **The driver takes 4.5 hours; protecting the long jobs takes 2.7 days.** That is the honest answer to "why does a 200-node driver rollout take a week", and it is the number to bring to a conversation with job owners.

Note carefully what those 245 slot-hours are *not*: they are not idle GPU-hours. A node in `wait-for-jobs-required` is cordoned, but the job on it is still running and still producing work. The idle bill is unchanged at 142.7 GPU-hours. What you are spending is **rollout throughput**, and the currency is calendar time and operational risk (a fleet in a mixed driver state for three days).

**Step 5 — the interaction that stalls you.** With `maxUnavailable: 5%` = 10 and P = 4, how many long-job waiters can you tolerate? Every waiter is cordoned, so it counts in `currentUnavailableNodes`. The moment 10 nodes are simultaneously cordoned — waiters plus any nodes cordoned for unrelated reasons — `upgradesAvailable` goes to zero and *nothing* advances, including the fast nodes that would have finished in five minutes.

```
  waiters concurrently cordoned : bounded by P = 4
  unrelated cordoned/NotReady   : call it u
  stall condition               : 4 + u ≥ 10   →   u ≥ 6
```

So on this fleet, six nodes cordoned for hardware investigations are enough to freeze a rollout that is otherwise running at its configured concurrency. **The fix is to size `maxUnavailable` against the fleet's baseline unavailability, not against zero** — audit `kubectl get nodes | grep SchedulingDisabled` before you start, and add that count to your ceiling.

**Step 6 — three levers, priced.**

```
  A. Raise maxUnavailable 5% → 10%:
       wall clock (no waiters) 4.5 h → 1.8 h at P=10 (P becomes the binding limit)
       stall headroom u: 6 → 16 nodes
       cost: peak dip 5% → 10% of fleet capacity
       idle GPU-hours: UNCHANGED (142.7)

  B. Set waitForCompletion.timeoutSeconds = 7200 (2 h) instead of 0:
       waiters that finish inside 2 h are protected; the rest are evicted
       wall clock: bounded by 2 h per waiter instead of 4 h mean
         → 60 × 2.09 + 12.5 = 138 slot-hours ÷ 4 ≈ 34.5 h  (2.7 d → 1.4 d)
       cost: jobs still running at 2 h lose their work → checkpointing becomes
             a hard prerequisite you must communicate before the window

  C. Move to precompiled modules (usePrecompiled), or bake the driver into the
     node image (driver.enabled=false):
       T_idle 11.35 min → 5.35 min in the DKMS case
       idle GPU-hours 302.7 → 142.7  →  saves ~$400 per fleet-wide rollout
       cost: usePrecompiled is IMMUTABLE on an NVIDIADriver object, and a
             precompiled image must exist for your exact running kernel;
             the node-image route couples driver changes to image rotation
```

Lever C is the only one that reduces the idle bill; A and B trade calendar time against risk. **Say which of the three you are pulling, and why, and you have answered the 200-node question.**

## Perspectives

**Fleet SRE at 200-node scale.** The two knobs turn a scary question into arithmetic: pick the blast-radius ceiling first (`maxUnavailable`, sized against your fleet's *baseline* cordoned count, not against zero), then pick throughput (`maxParallelUpgrades`) up to that ceiling. The constraint that appears nowhere in the YAML — never drain two nodes of the same NVLink/rail domain or the same gang-scheduled job simultaneously — is what separates a senior answer from a junior one, because it cannot be expressed in this API at all and therefore has to live in your labelling or your own orchestration.

**Kernel / systems.** The refcount fact from module 03 becomes a *planned* operation here rather than a debugging nuisance. Everything from `cordon-required` to `pod-restart-required` exists to drive `/sys/module/nvidia/refcnt` to zero deliberately. And note where that abstraction leaks: the module unload is a `delete_module(2)` syscall with no force flag, so anything holding a GPU that the Operator's pod filter cannot recognise — a privileged pod with a hostPath `/dev/nvidia0`, a leaked process outside a container, a systemd service on a host-driver node — will block the upgrade with `EBUSY` and no amount of Kubernetes-level configuration will help. `/proc/modules` is the only thing that will tell you who.

**Cost / economics.** §11's headline is worth repeating because it is genuinely counter-intuitive: `maxParallelUpgrades` does not change the GPU-hour bill. `N × g × T_idle` does. The only levers that reduce spend are the ones that shrink per-node idle time — precompiled modules, a baked node image, tighter grace periods — and the only levers that shrink calendar time trade against capacity or against job safety. Module 01's thread continues: every minute a node spends between pod-deletion and uncordon is a fully-billed idle GPU-hour, so `T_idle` deserves to be measured on your fleet, not assumed.

**Managed-cloud operator.** Not every fleet owns this layer. Hyperscalers and neoclouds increasingly productise it: GKE, for example, exposes automated driver installation as a node-pool setting (`gpu-driver-version=latest|default|disabled`), which mirrors the operator-managed-vs-pre-installed decision as a one-flag choice. On such a platform the honest answer to "design a driver upgrade for 200 nodes" may be *"the platform owns the install; my job is the per-nodepool policy, the long-job protection, and the canary plan"* — and knowing where the vendor boundary sits is itself a seniority signal. Also know the failure mode of that convenience: automated driver installation is a distributed system that can fail, and a rollout design that trusts it blindly with no canary and no rollback trigger is not a design.

**Vendor-boundary / incident reality.** Whichever layer you own, the class of failure that hurts most is the one that changes the driver *without anyone changing `spec.driver.version`*. A managed control-plane upgrade that replaces the node image swaps the kernel underneath you, which invalidates a precompiled module tag and can wipe `/etc/containerd/config.toml` along with the `nvidia` runtime handler (lesson 04.4 §6.2). Treat "we also bumped Kubernetes" as a variable in the rollout plan, not a constant.

## Real-world use cases

- **`k8s-driver-manager`'s own eviction-and-unload implementation** — read directly this session at `v0.11.0`. Its README lists the sequence in six steps (check installed modules → drain ignoring DaemonSets → evict GPU Operator components → unload kernel modules → unmount `/run/nvidia/driver` → uncordon), and the code shows the parts the README omits: the exact module unload order, the `/sys/module/<name>/refcnt` probe, the `delete_module(2)` calls, the `/proc/modules` dump on failure, and the DRA-specific ordering constraint. What it shows: the "drain the node" step in every runbook you have read is one line of a much longer procedure, and the interesting failures live in the lines that are usually omitted.

- **The DRA kubelet-plugin ordering constraint**, documented in comments in `k8s-driver-manager`'s `uninstallDriver()`. On a DRA cluster the plugin must **outlive** the pods holding GPU `ResourceClaim`s (it services `NodeUnprepareResources` for them) but must be drained **before** the module is unloaded and before the old driver rootfs is unmounted — because the plugin bind-mounts that rootfs and would otherwise be left building CDI specs against a filesystem whose submounts have vanished. The manager also explicitly checks `GetGPUResourceClaimHolders` and refuses with `cannot drain the DRA kubelet-plugin: pod(s) still hold GPU resource claims: …` rather than proceeding. What it shows: adding DRA to a cluster adds a genuine ordering constraint to driver upgrades, and it is the kind of constraint that only appears once someone has hit the deadlock in production. Relevant forward-looking context for lesson 04.9.

- **The `sandboxWorkloads.enabled=true` incompatibility** — read from `controllers/upgrade_controller.go`. With sandbox (KubeVirt/vGPU) workloads enabled, the Operator logs *"Advanced driver upgrade policy is not supported when 'sandboxWorkloads.enabled=true' in ClusterPolicy, cleaning up upgrade state and skipping reconciliation"* and **removes** the `nvidia.com/gpu-driver-upgrade-state` labels. What it shows: a whole-feature switch-off with a single info-level log line and no condition on the CR. If you run mixed VM and container GPU workloads and wonder why your nodes have no upgrade-state label, this is why — and every mechanism in §3 is inactive for you.

- **The GKE automated-driver-install feature and its own failure history.** Google shipped `gpu-driver-version=latest|default|disabled` as a node-pool setting, productising exactly this lesson's problem; Google Cloud has also had a real multi-hour GKE incident traced to a GPU driver *fetch* failure affecting GPU node pools. *(The feature is documented publicly; the specific incident's date and duration could not be re-verified this session because the status-page domain is not reachable from this environment — do not quote a duration without checking the incident page yourself.)* What it shows, regardless of the exact numbers: even a hyperscaler's own driver-install automation is a distributed system with an outage history, which is the argument for canary waves and a named rollback trigger rather than trust.

- **The upgrade-stall class, filed repeatedly.** Practitioners have reported nodes wedged mid-upgrade — [NVIDIA/gpu-operator#626 "Nodes stuck in upgrade"](https://github.com/NVIDIA/gpu-operator/issues/626) (nodes stuck in `validation-required` and `pod-restart-required`), [#542 "GPU-operator not applying driver version changes on EKS"](https://github.com/NVIDIA/gpu-operator/issues/542) (zero nodes reported as needing an upgrade despite an out-of-date driver), [#901](https://github.com/NVIDIA/gpu-operator/issues/901) (eviction and drain reported as *"disabled by the upgrade policy"* despite auto-upgrade being enabled), and [#416 "Driver won't fully start without manually draining node"](https://github.com/NVIDIA/gpu-operator/issues/416). *(Titles and numbers found via search this session; bodies not independently fetched, as GitHub API access is restricted in this environment.)* What it shows: the two states that trap nodes are `validation-required` (600-second hard timeout, then failed) and `pod-restart-required` (module refcount), and the two ways a rollout silently does nothing are the `upgradesAvailable == 0` stall of §4.1 and `drain.enable` being `false` when the runbook assumed `true`. The mechanisms behind all four are derived from source in §3–§6, not from the issue bodies.

## Worked example

Trigger a version bump and walk one node through the machine, then read the fleet-level arithmetic off the same cluster. Transcripts are representative — the shapes and field names are real; run it yourself for your own timings.

### 1. Establish the starting state, including the things that will bite you

```bash
$ kubectl -n gpu-operator get clusterpolicy cluster-policy \
    -o jsonpath='{.spec.driver.version}{"  kmt="}{.spec.driver.kernelModuleType}{"\n"}'
580.65.06  kmt=auto

$ kubectl -n gpu-operator get clusterpolicy cluster-policy \
    -o jsonpath='{.spec.driver.upgradePolicy}' | jq
{
  "autoUpgrade": true,
  "maxParallelUpgrades": 1,
  "maxUnavailable": "25%",
  "drain":       { "enable": false, "force": false, "timeoutSeconds": 300 },
  "podDeletion": { "force": false, "timeoutSeconds": 300 },
  "waitForCompletion": { "timeoutSeconds": 0, "podSelector": "" }
}
```

Read what is actually configured, not what you assume. `drain.enable` is `false` — the default — so this cluster relies on targeted GPU-pod deletion and will jump straight to `upgrade-failed` if that does not clear the node. `waitForCompletion.podSelector` is empty, so there is **no long-job protection at all**; a 40-hour run on this cluster gets deleted. Both of those are findings, not background.

Now audit the baseline unavailability that §4.1 will count against you:

```bash
$ kubectl get nodes -l nvidia.com/gpu.present=true \
    -L nvidia.com/gpu-driver-upgrade-state -o wide | head
NAME         STATUS                     ...  GPU-DRIVER-UPGRADE-STATE
gpu-node-01  Ready                           upgrade-done
gpu-node-02  Ready                           upgrade-done
gpu-node-03  Ready,SchedulingDisabled        upgrade-done      ◀── cordoned by hand
gpu-node-04  NotReady                        upgrade-done      ◀── kubelet wedged
...

$ kubectl get nodes -l nvidia.com/gpu.present=true --no-headers | wc -l
8
$ kubectl get nodes -l nvidia.com/gpu.present=true --no-headers \
    | grep -cE 'SchedulingDisabled|NotReady'
2
```

Eight managed GPU nodes, two already unavailable. `maxUnavailable: 25%` of 8 rounds up to 2. So `currentUnavailableNodes (2) >= maxUnavailable (2)` and **`upgradesAvailable` will be zero from the first reconcile**. On this cluster, as configured, the rollout would not start at all — and the only trace would be a debug log line. Fix that before touching the version:

```bash
$ kubectl uncordon gpu-node-03
node/gpu-node-03 uncordoned
# and either repair or cordon-and-skip gpu-node-04:
$ kubectl label node gpu-node-04 nvidia.com/gpu-driver-upgrade.skip=true
node/gpu-node-04 labeled
```

### 2. Schedule a marker workload so the node is genuinely in use

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: marker
  labels: { training: long-running }
spec:
  nodeName: gpu-node-01
  terminationGracePeriodSeconds: 30
  containers:
  - name: busy
    image: nvidia/cuda:13.0.0-base-ubuntu24.04
    command: ["sh","-c","nvidia-smi -l 5"]
    resources: { limits: { nvidia.com/gpu: 1 } }
```

Confirm it is holding the module:

```bash
$ kubectl -n gpu-operator exec ds/nvidia-driver-daemonset -- \
    sh -c 'cat /sys/module/nvidia/refcnt; grep ^nvidia /proc/modules'
6
nvidia_uvm 1892352 2 - Live 0x...
nvidia 62783488 4 nvidia_uvm,nvidia_modeset Live 0x...
```

Refcount 6 on `nvidia`, and `/proc/modules` shows `nvidia_uvm` holding it. **`rmmod nvidia` cannot succeed until that reaches zero.** This is the fact the whole state machine is built around; look at it once with your own eyes.

### 3. Set a policy you can defend, then bump the version

```bash
$ kubectl -n gpu-operator patch clusterpolicy cluster-policy --type merge -p '{
  "spec": { "driver": {
    "version": "595.71.05",
    "kernelModuleType": "open",
    "upgradePolicy": {
      "autoUpgrade": true,
      "maxParallelUpgrades": 1,
      "maxUnavailable": "1",
      "waitForCompletion": { "podSelector": "training=long-running",
                             "timeoutSeconds": 600 },
      "podDeletion": { "force": true, "timeoutSeconds": 300,
                       "deleteEmptyDir": true },
      "drain": { "enable": true, "force": true, "timeoutSeconds": 300,
                 "deleteEmptyDir": true }
    }
  }}}'
clusterpolicy.nvidia.com/cluster-policy patched
```

> **Which path is this?** `580.65.06 → 595.71.05` changes the driver install config, so the digest changes and this is the **full** flow: unload, rebuild/unpack, load. The `DRIVER_CONFIG_DIGEST` fast path of §6 does *not* apply. If you had re-applied the same version and the same install-relevant fields, the digest would match, the node would still be cordoned and validated, but `pod-deletion-required` and `drain-required` would be skipped. Diff the digest to know which you are watching:
>
> ```bash
> $ kubectl -n gpu-operator get ds nvidia-driver-daemonset \
>     -o jsonpath='{.spec.template.spec.containers[?(@.name=="nvidia-driver-ctr")].env[?(@.name=="DRIVER_CONFIG_DIGEST")].value}{"\n"}'
> 7f2a9c1e...
> $ kubectl -n gpu-operator get pod -l app=nvidia-driver-daemonset \
>     --field-selector spec.nodeName=gpu-node-01 \
>     -o jsonpath='{.items[0].spec.containers[?(@.name=="nvidia-driver-ctr")].env[?(@.name=="DRIVER_CONFIG_DIGEST")].value}{"\n"}'
> c3b81d40...
> ```
>
> Different digests ⇒ full flow.

### 4. Watch the label walk, with timestamps

```bash
$ kubectl get node gpu-node-01 -L nvidia.com/gpu-driver-upgrade-state -w \
    | ts '%H:%M:%S' | tee /tmp/upgrade-gpu-node-01.log
09:14:02 gpu-node-01  Ready                      upgrade-required
09:14:04 gpu-node-01  Ready,SchedulingDisabled   cordon-required
09:14:06 gpu-node-01  Ready,SchedulingDisabled   wait-for-jobs-required
09:24:07 gpu-node-01  Ready,SchedulingDisabled   pod-deletion-required
09:24:41 gpu-node-01  Ready,SchedulingDisabled   pod-restart-required
09:28:12 gpu-node-01  Ready,SchedulingDisabled   validation-required
09:29:30 gpu-node-01  Ready                      uncordon-required
09:29:32 gpu-node-01  Ready                      upgrade-done
```

Read the intervals, not just the states:

- `09:14:06 → 09:24:07` — exactly **10 minutes** in `wait-for-jobs-required`. The marker pod matched `training=long-running` and never terminates, so the `timeoutSeconds: 600` gate expired and the controller moved on. Had you left the default `timeoutSeconds: 0`, that node would still be sitting there, cordoned, consuming the single upgrade slot, indefinitely. The wait's start time is recorded on the node:

  ```bash
  $ kubectl get node gpu-node-01 -o json \
      | jq -r '.metadata.annotations
               | to_entries[] | select(.key|test("upgrade")) | "\(.key)=\(.value)"'
  nvidia.com/gpu-driver-upgrade-wait-for-pod-completion-start-time=1755418446
  nvidia.com/gpu-driver-upgrade.node-initial-state.unschedulable=false
  ```

  The second annotation is why the node gets uncordoned at the end: it recorded that the node *was* schedulable before the upgrade started.

- `09:24:07 → 09:24:41` — 34 s of pod deletion. The marker's `terminationGracePeriodSeconds: 30` is most of it, because the drain helper uses `GracePeriodSeconds: -1` and honours each pod's own value. **Your grace periods are directly on the critical path of every node**, which is a cheap thing to audit before a rollout.

- No `drain-required` state appears. Pod-deletion cleared every GPU pod, so drain was skipped — exactly as designed. If you *do* see `drain-required`, it means the targeted deletion did not clear the node, which is itself a finding.

- `09:24:41 → 09:28:12` — 3 m 31 s of `pod-restart-required`: the driver pod terminating, the new one scheduling, `k8s-driver-manager` unloading the modules and unmounting the rootfs, and `nvidia-driver-ctr` loading the new module.

- `09:28:12 → 09:29:30` — 78 s of validation, well inside the 600-second hard timeout.

**Idle GPU-hour window = 09:24:07 → 09:29:32 = 5 m 25 s.** The ten minutes before that were cordoned but productive: the marker pod was still running. That distinction is the whole basis of §11's arithmetic, and this transcript is where you see it directly.

### 5. Correlate with the driver pod and the module

```bash
$ kubectl -n gpu-operator get pod -l app=nvidia-driver-daemonset -o wide \
    --field-selector spec.nodeName=gpu-node-01 -w
NAME                          READY   STATUS            RESTARTS   NODE
nvidia-driver-daemonset-abc   1/1     Terminating       0          gpu-node-01
nvidia-driver-daemonset-xyz   0/1     Init:0/1          0          gpu-node-01
nvidia-driver-daemonset-xyz   0/1     PodInitializing   0          gpu-node-01
nvidia-driver-daemonset-xyz   1/1     Running           0          gpu-node-01

$ kubectl -n gpu-operator logs nvidia-driver-daemonset-xyz -c k8s-driver-manager
Starting driver uninstallation process
Checking if the currently loaded NVIDIA driver version and configuration matches the desired state...
Shutting down all GPU clients in Kubernetes by disabling their component-specific nodeSelector labels
Waiting for nvidia-operator-validator to shutdown
Waiting for nvidia-container-toolkit-daemonset to shutdown
Waiting for nvidia-device-plugin-daemonset to shutdown
Waiting for gpu-feature-discovery to shutdown
Waiting for nvidia-dcgm-exporter to shutdown
Waiting for nvidia-dcgm to shutdown
Unloading NVIDIA driver kernel modules
Unmounting NVIDIA driver rootfs
Successfully unmounted /run/nvidia/driver and all its submounts
Successfully uninstalled nvidia driver components
Driver uninstallation completed successfully

$ kubectl -n gpu-operator exec nvidia-driver-daemonset-xyz -- nvidia-smi \
    --query-gpu=driver_version --format=csv,noheader | head -1
595.71.05

$ kubectl -n gpu-operator logs -l app=nvidia-operator-validator \
    --all-containers --tail=5 --field-selector spec.nodeName=gpu-node-01
all validations are successful
```

Note the shape of the driver-manager log: it tears down the GPU Operator's *own* operands first, by flipping their `nvidia.com/gpu.deploy.*` node labels to `paused-for-driver-upgrade` (the DaemonSets use those labels as node selectors, so flipping the label deletes the pod), then waits for each to terminate with a five-minute grace period each. It is the same label-flipping pattern lesson 04.6 will show `mig-manager` using with `paused-for-mig-change`.

### 6. Break it deliberately, and read the block

Reproduce the `EBUSY` case so it is not theoretical. Start a pod that holds a GPU through a path the pod filter cannot see:

```bash
$ kubectl run rogue --image=nvidia/cuda:13.0.0-base-ubuntu24.04 --restart=Never \
    --overrides='{"spec":{"nodeName":"gpu-node-01",
      "containers":[{"name":"c","image":"nvidia/cuda:13.0.0-base-ubuntu24.04",
        "command":["sh","-c","nvidia-smi -l 5"],
        "securityContext":{"privileged":true},
        "env":[{"name":"NVIDIA_VISIBLE_DEVICES","value":"all"}]}]}}'
```

This pod requests **no** `nvidia.com/gpu` resource, so the Operator's GPU-pod filter (`nvidia.com/gpu*` / `nvidia.com/mig-*` / an NVIDIA `ResourceClaim`) does not match it and pod-deletion leaves it alone. Trigger another upgrade and watch:

```bash
$ kubectl -n gpu-operator logs nvidia-driver-daemonset-new -c k8s-driver-manager | tail
Unloading NVIDIA driver kernel modules
Failed to unload kernel module nvidia_uvm: device or resource busy
Failed to unload kernel module nvidia: device or resource busy
Could not unload NVIDIA driver kernel modules, driver is in use
Module               Size       Ref Count       Used by
nvidia_uvm           1892352    2               -
nvidia_modeset       1355776    1               -
nvidia               62783488   4               nvidia_uvm,nvidia_modeset
Unable to cleanup driver modules, attempting again with node drain...
```

There is the whole mechanism in one log block: two `EBUSY` failures, the refcount table, and the fallback to a full node drain — which *will* catch the rogue pod, because `drain` is not resource-filtered the way pod-deletion is. That is the reason to enable `drain.enable: true` as a backstop even though the normal path never uses it.

The recovery, and the lesson for the failure-mode log:

```bash
$ kubectl delete pod rogue --grace-period=10
$ kubectl -n gpu-operator delete pod nvidia-driver-daemonset-new   # retry
$ kubectl get node gpu-node-01 -L nvidia.com/gpu-driver-upgrade-state -w
gpu-node-01  Ready,SchedulingDisabled   pod-restart-required
gpu-node-01  Ready,SchedulingDisabled   validation-required
gpu-node-01  Ready                      upgrade-done
```

**Write this in the log:** a pod that reaches GPUs without requesting a GPU resource is invisible to the Operator's eviction filter and will block a driver upgrade at the module-unload step with `EBUSY`. Detect it with an admission policy that forbids setting `NVIDIA_VISIBLE_DEVICES` and forbids privileged pods with hostPath `/dev`, and keep `drain.enable: true` as the backstop.

## Practice

This task feeds the deliverable's **failure-mode log** and produces the **200-node runbook** artifact, both landing in [Per-pod GPU attribution](../practice/per-pod-attribution/README.md).

**Part A — observe the machine and measure your own timings (single node).** On a lab cluster (one GPU node is fine — a cloud GPU VM, or one node of a managed GPU pool):

1. Record the *actual* configured policy: `spec.driver.version`, `kernelModuleType`, and the whole `upgradePolicy` object. Note explicitly whether `drain.enable` is `true` or `false` and whether `waitForCompletion.podSelector` is set — do not assume either.
2. Audit baseline unavailability: count managed GPU nodes, count how many are `SchedulingDisabled` or `NotReady`, compute `maxUnavailable` from the configured value, and state whether `upgradesAvailable` can be non-zero at all. This is the §4.1 stall check and it takes two commands.
3. Look at the refcount directly: `cat /sys/module/nvidia/refcnt` and `grep ^nvidia /proc/modules` with a GPU pod running, and again with none running.
4. Schedule a marker GPU pod with a deliberate `terminationGracePeriodSeconds` you choose, then trigger a version bump with `maxParallelUpgrades=1`, `maxUnavailable=1`.
5. Capture the full ordered sequence of `nvidia.com/gpu-driver-upgrade-state` values **with timestamps** (`kubectl get node -w | ts`), plus the cordon transition and the driver-pod restart. From that transcript, extract and record: the duration of each state, whether `drain-required` appeared at all (and why), and — the number that matters — the **idle GPU-hour window** from `pod-deletion-required` to `uncordon-required`.
6. Record whether this was a real config change or a same-digest reconcile, by diffing `DRIVER_CONFIG_DIGEST` between the running pod and the DaemonSet template. If you can, do both: bump the version once, then re-apply the identical spec, and record the difference in which states appear.
7. Read the node's upgrade annotations (`kubectl get node -o json | jq '.metadata.annotations'`) and identify the wait-start, validation-start, and initial-schedulability annotations.

**Part B — trigger and recover the `EBUSY` block.** Reproduce §6 of the worked example: run something that holds a GPU without requesting a `nvidia.com/gpu` resource, attempt an upgrade, and capture the `Could not unload NVIDIA driver kernel modules, driver is in use` log line together with the `/proc/modules` refcount table. Then recover. Write the root cause in terms of the Operator's GPU-pod filter, not in terms of "a pod was still running".

**Part C — write the 200-node rollout runbook.** Produce a runbook covering, explicitly, all eight of:

- **Batching, with arithmetic.** Chosen `maxParallelUpgrades` and `maxUnavailable`, with the reasoning shown — including your fleet's **baseline** cordoned/NotReady count added into the ceiling, and the resulting stall headroom.
- **The GPU-hour bill.** `N × g × T_idle` using your measured `T_idle` from Part A, converted to dollars at your rate, plus the wall-clock table as a function of `maxParallelUpgrades`. State explicitly that the bill is independent of parallelism and what that means for the choice.
- **Cordon/eviction policy.** `podDeletion` vs `drain`: which is the primary path, which is the backstop, and the `force` / `deleteEmptyDir` / `timeoutSeconds` values with reasons. Include the audit of pod `terminationGracePeriodSeconds` values, since they are on the critical path.
- **Long-job protection, priced.** How you label long runs, the `waitForCompletion.podSelector` and a **non-zero** `timeoutSeconds` with the calendar-time consequence computed (§11 step 4), what you tell job owners about checkpoint cadence before the window, and the slot-consumption interaction with `maxUnavailable`.
- **Topology constraints the YAML cannot express.** The rule that no batch may contain two nodes of the same NVLink/rail domain or two ranks of the same gang-scheduled job, and the labelling or tooling that enforces it.
- **Health gates.** What must be green before a batch advances: `nvidia-operator-validator` Ready, node `Ready`, `nvidia-smi --query-gpu=driver_version` equal to the target, a DCGM health check, and a canary GPU pod that runs a real kernel. Include the three PromQL expressions from §10 — especially the silent-stall detector.
- **Rollback trigger and action.** The concrete condition that stops the rollout (`gpu_operator_nodes_upgrades_failed > 1`, or a spike in Xid errors) and the action: re-pin `spec.driver.version` to the previous tag, which drives the same state machine in reverse. Note what rollback does *not* undo — evicted jobs are gone.
- **Canary → wave plan and the vendor boundary.** 1 node → 5% → 25% → 100%, with a soak time and the metric watched at each wave; plus one line stating whether this fleet runs its own upgrade controller or a managed layer owns part of it, and what changes if so.

**Acceptance:**

- Timestamped state-machine transitions from Part A, with per-state durations, the measured `T_idle`, and the digest-diff note, saved to the failure-mode log.
- The Part B `EBUSY` reproduction: the log line, the `/proc/modules` table, the root cause expressed in terms of the GPU-pod filter, and the recovery steps.
- The 200-node runbook containing all **eight** bullet categories with concrete values (not "TBD"), the GPU-hour arithmetic worked through with your own measured numbers, a named rollback trigger expressed as a query, and an explicit statement of how long a labelled training job is protected before it is evicted.

## Common pitfalls

1. **Assuming `drain.enable` is `true`.** It defaults to `false` in both the API and the Helm chart. The Operator's normal path is targeted GPU-pod deletion; a full node drain is the escalation. A runbook that says "the Operator drains the node" is describing a configuration you may not have — and worse, without drain as a backstop, a pod that escapes the GPU-pod filter takes the node to `upgrade-failed` instead of being force-drained.

2. **Sizing `maxUnavailable` against zero baseline unavailability.** `currentUnavailableNodes` counts every managed GPU node that is cordoned or `NotReady` for *any* reason, plus everything in `cordon-required`. If that count already meets your ceiling, `upgradesAvailable` is zero and the rollout never starts, with only a debug-level log line to show it. Audit `kubectl get nodes | grep -cE 'SchedulingDisabled|NotReady'` first and add it to the ceiling.

3. **Confusing "cordoned" with "idle".** A node in `wait-for-jobs-required` is unschedulable but its GPUs are fully busy running the job you are waiting for. It costs you a rollout slot and calendar time, not GPU-hours. Conflating the two leads to the wrong optimisation: people raise `maxParallelUpgrades` to "save money" when the money is set by per-node idle time and only calendar time moves.

4. **Leaving `waitForCompletion.timeoutSeconds` at its default of 0.** Zero means *wait forever*. One node with a permanently-running pod that matches your `podSelector` sits cordoned in `wait-for-jobs-required` indefinitely, holding a slot. With `maxParallelUpgrades: 1` that is a rollout that never completes and never errors. Always set a finite timeout, and tell job owners what it is.

5. **Assuming every driver-pod restart unloads the module.** The fast path is keyed on `DRIVER_CONFIG_DIGEST` — a hash of install-relevant pod-spec fields — not on the version string. Matching digests skip `pod-deletion-required` and `drain-required` and reuse the loaded module; a differing digest (including a same-version change to `kernelModuleType`) takes the full flow. Diff the digest before estimating downtime, and remember that even the fast path still cordons, validates and uncordons.

6. **Treating `kernelModuleType: auto` as version-independent.** `auto` resolves from the GPUs present *and the driver branch*. The same line means `open` beside a current branch and `proprietary` beside an older pinned LTSB the open module never covered. Always write the branch next to the module type in a runbook. And do not reach for `useOpenKernelModules` — it is deprecated and no longer honoured.

7. **Sizing batches without a topology rule.** `maxParallelUpgrades` and `maxUnavailable` bound blast radius numerically and know nothing about NVLink or rail domains or gang scheduling. A numerically safe 5% batch can be five ranks of one 512-GPU job, taking out 512 GPUs to upgrade 40. Nothing in this API can express the constraint; it has to come from your labelling or your own orchestration.

8. **Forgetting that `sandboxWorkloads.enabled=true` switches the whole machine off.** With sandbox workloads enabled, the controller removes the upgrade-state labels and skips reconciliation entirely, with a single info log line. Every state, knob and metric in this lesson becomes inert. Check for the label's presence before you start debugging a rollout that "isn't doing anything".

## Self-check

- **Walk the driver-upgrade state machine end to end. What does each state do, what blocks it, and where does a stuck PodDisruptionBudget land?** **Answer:** Via `nvidia.com/gpu-driver-upgrade-state`: `upgrade-required` (queued; blocks on an available upgrade slot, computed from `maxParallelUpgrades` minus in-progress, clamped to `maxUnavailable` minus fleet-wide cordoned/NotReady nodes) → `cordon-required` (marks the node `Unschedulable`; blocks on nothing, and records the node's prior schedulability in a `…node-initial-state.unschedulable` annotation) → `wait-for-jobs-required` (optional; blocks until every pod matching `waitForCompletion.podSelector` leaves Running/Pending, or `timeoutSeconds` expires — default `0` means wait forever) → `pod-deletion-required` (optional; deletes pods requesting `nvidia.com/gpu*`, `nvidia.com/mig-*`, or holding an NVIDIA `ResourceClaim`; bounded by `podDeletion.timeoutSeconds`, default 300) → `drain-required` (full `kubectl` drain, **skipped** if pod-deletion cleared every GPU pod, and **disabled by default** since `drain.enable` defaults to `false`; bounded by `drain.timeoutSeconds`, default 300) → optionally `node-maintenance-required` / `post-maintenance-required` when an external maintenance operator owns cordon and drain → `pod-restart-required` (delete the driver DaemonSet pod, whose update strategy is `OnDelete` so only the controller ever deletes it; or, on a safe-load node, remove the `…driver-wait-for-safe-load` annotation instead; blocks on the module refcount reaching zero and on the module build/load succeeding) → `validation-required` (blocks until the `app=nvidia-operator-validator` pod on this node is Running with all containers Ready; **600-second hard-coded timeout**, not configurable) → `uncordon-required` → `upgrade-done`. `upgrade-failed` is terminal. **A stuck PodDisruptionBudget lands at `drain-required`** (or at `pod-deletion-required`, since both use the Eviction API): eviction cannot complete, the timeout expires, and the node goes to `upgrade-failed` rather than hanging forever — which is a feature, because it surfaces the blockage instead of silently stalling the whole rollout. `force: true` on either spec is what lets it evict pods not backed by a controller.

- **Why does unloading the module fail while a process holds the device, and what exactly do you look at?** **Answer:** Linux kernel modules carry a reference count, and `delete_module(2)` — what `rmmod` calls — returns `EBUSY` when it is non-zero. Every open file descriptor on `/dev/nvidia*` holds a reference, so every live CUDA context pins `nvidia.ko`. There is no force flag and no privileged escape; the only path to zero is for every holder to exit. `k8s-driver-manager` unloads in dependency order — `nvidia_modeset`, `nvidia_uvm`, `nvidia_peermem`, `nvidia_fs`, `nvidia_vgpu_vfio`, `gdrdrv`, then `nvidia` — checking `/sys/module/<name>/refcnt` exists before each attempt, and on failure logs `Could not unload NVIDIA driver kernel modules, driver is in use` followed by the matching lines of `/proc/modules` with a `Module / Size / Ref Count / Used by` header. **That table is the diagnostic**: it names the count and it names the dependent modules. The nastiest version of this failure is a pod that reaches GPUs *without* requesting `nvidia.com/gpu` — a privileged pod with `NVIDIA_VISIBLE_DEVICES=all` or a hostPath `/dev/nvidia0` — because the Operator's GPU-pod filter does not match it, pod-deletion leaves it alone, and only a full node drain will clear it. Beyond the modules, two more things must be cleaned: the previous driver container's rootfs at `/run/nvidia/driver` (recursively unmounted, or the toolkit discovers old libraries against a new module) and `nouveau`, which claims the same PCI devices.

- **Why does a running container keep a stale driver ABI after an upgrade, and what does that look like?** **Answer:** Because the injection in lesson 04.4 happened **once, at container create time**. The container's mount namespace holds bind mounts to specific versioned files — `/usr/lib/x86_64-linux-gnu/libcuda.so.580.65.06` — and its device nodes were `mknod`'d with the majors and minors that were valid then. Nothing revisits that when the host changes. A container that was running before the upgrade keeps a mount that now points at a file the host has replaced or removed, and a `libcuda` whose version no longer matches the loaded module. The first thing it does after the module swap fails the NVML version handshake: `Failed to initialize NVML: Driver/library version mismatch`. This is why the state machine evicts rather than tries to be clever — **there is no in-place fix for a running container's injected driver**, so the only correct action is to make the container exit and let a new one be created against the new driver. The symptom to recognise at fleet level: the node is `Ready`, the scheduler happily places GPU pods on it, and every one of them crash-loops. Cross-check with `nvidia-smi --query-gpu=driver_version` inside the driver pod against `spec.driver.version`. A related, subtler version: a persistent CDI spec in `/etc/cdi/nvidia.yaml` naming the old driver's exact filenames will fail *new* containers too, until it is regenerated — which is one reason the default `jit-cdi` mode, which synthesises the spec per container, is the safer default across upgrades.

- **`maxParallelUpgrades: 4` and `maxUnavailable: 5%` on a 200-node fleet. How many nodes upgrade at once, what does the rollout cost in GPU-hours, and what would stall it?** **Answer:** `maxUnavailable: 5%` of 200 rounds up to 10 nodes. `upgradesAvailable` = `maxParallelUpgrades − upgradesInProgress`, clamped to `maxUnavailable`, then further reduced by `maxUnavailable − currentUnavailableNodes`. So at steady state **4 nodes** move at once — `maxParallelUpgrades` is binding, and raising it above 10 would buy nothing because `maxUnavailable` caps it. **GPU-hours** = `N × g × T_idle`, where `T_idle` is the window from `pod-deletion-required` to `uncordon-required`. With 8 H100s per node and a measured `T_idle` of 5.35 min, that is `200 × 8 × 5.35/60 ≈ 143 GPU-hours`, about $357 at $2.50/GPU-hour. **Crucially that figure does not depend on `maxParallelUpgrades`** — parallelism sets wall clock (⌈200/4⌉ × 5.35 min ≈ 4.5 h) and peak dip (4 × 8 = 32 GPUs = 2% of the fleet), not total spend. **Two things stall it.** First, `currentUnavailableNodes` counts every managed GPU node cordoned or `NotReady` for any reason: with `maxParallelUpgrades: 4` occupying four cordoned nodes, six unrelated cordoned nodes bring the total to 10 and drive `upgradesAvailable` to zero, freezing everything with only a debug log line. Second, nodes parked in `wait-for-jobs-required` count as in-progress and hold slots for the entire wait; with 60 nodes hosting 4-hour jobs and `timeoutSeconds: 0`, total slot-time becomes `60 × 4.09 + 140 × 0.089 ≈ 258` slot-hours, so at P=4 the wall clock goes from 4.5 hours to about 65 hours. Detect the first with the PromQL `pending > 0 and available == 0 and in_progress == 0`; mitigate the second with a finite `waitForCompletion.timeoutSeconds`.

- **A node is stuck in `validation-required`. What do you check, in order, and how long before it fails on its own?** **Answer:** It will go to `upgrade-failed` after **600 seconds**, hard-coded in the validation manager and not configurable; the countdown start is stamped on the node as a `…-driver-upgrade-validation-start-time` annotation. Within that window: (1) `kubectl logs` the `nvidia-operator-validator` pod on that node **per init container** — they run in order `driver-validation` → `toolkit-validation` → `cuda-validation` → `plugin-validation`, each writing `/run/nvidia/validations/<component>-ready`, so the first one that has not written its file names the failing layer. (2) If `driver-validation` is the blocker, check the driver DaemonSet pod: is it `Running`, and does `nvidia-smi --query-gpu=driver_version` inside it report the *new* version? A crash-looping driver pod means a module build or load failure — read `k8s-driver-manager` and `nvidia-driver-ctr` logs for kernel-header mismatch, DKMS failure, Secure Boot signing, or a precompiled image that has no build for the running kernel. (3) `dmesg | grep -i -E 'nvrm|nvidia'` on the host for `nvidia.ko` load errors, and `grep ^nvidia /proc/modules` to check whether the *old* module is still resident with a non-zero refcount because something escaped eviction. (4) If `toolkit-validation` is the blocker, the container toolkit DaemonSet is the suspect — and that is lesson 04.4's territory: the runtime handler, the CDI spec, the driver root. (5) If `plugin-validation` is the blocker, look at the device plugin, and on a MIG node at `mig-manager` state (lesson 04.6). Note the validator's `preStop` hook removes `/run/nvidia/validations/*-ready`, so deleting the validator pod is a clean way to re-run the whole chain after fixing the root cause. If it is unrecoverable, re-pin `spec.driver.version` to the previous tag, which drives the same state machine back the other way.

- **What does CUDA forward compatibility buy you in the context of a fleet driver upgrade, and what does it not?** **Answer:** It buys you the ability to **not do the upgrade yet.** The `cuda-compat` package puts a newer `libcuda.so` inside the image, and the `enable-cuda-compat` CDI hook prepends its directory to the container's linker path, so a newer CUDA runtime works against the older resident kernel module — per container, with no cordon, no eviction, no reboot and no GPU-hours lost. That makes it the correct first response to `cudaErrorInsufficientDriver` on a frozen fleet. What it does not buy: (1) freedom from the branch allowlist — the compat library's `.note.cuda.fwd_compatibility` ELF note enumerates the exact host driver branches it supports (a CUDA 13.1 build listing `[535, 550, 570, 575, 580, 590]` will refuse an R565 host), so it is a published matrix, not a general guarantee; (2) support for silicon the resident module never knew about, since the module is what talks to the hardware; (3) anything on consumer-class GPUs, where it produces CUDA error 804 `cudaErrorCompatNotSupportedOnDevice`; (4) any relief from an NVML `Driver/library version mismatch`, which is a module-versus-userspace *build* mismatch rather than an API-level gap. And it does not buy you new driver features, bug fixes, Xid handling improvements, or security patches — all of which are reasons the upgrade still has to happen eventually. Treat it as a way to **decouple** the CUDA-version deadline from the driver-upgrade window, not as a way to avoid the window. Lesson 04.4 §8.3 has the mechanism in full.

- **Your fleet moved from Kubernetes 1.29 to 1.30 and GPU pods start crash-looping with no driver-version change. First thing you check, and why?** **Answer:** Whether the container-runtime and toolkit wiring survived the node-image change that came with the control-plane upgrade. Two specific signatures: `failed to get sandbox runtime: no runtime for 'nvidia' is configured`, which is containerd's CRI plugin saying the pod asked for a `RuntimeClass` handler that does not exist in the config it actually loaded — a replaced node image can wipe `/etc/containerd/config.toml` and take the `nvidia` handler with it; and a kernel-version-resolution failure in the driver container, because a managed upgrade frequently ships a new kernel and a precompiled driver image built for the old one has nothing to install. The reason to check this *first* is that the mental model "we only bumped Kubernetes" is wrong: a managed control-plane upgrade can change the node image, the kernel, and the containerd config underneath every assumption the GPU Operator makes, and none of it is visible in `spec.driver.version`. Practical mitigations: prefer a containerd drop-in at `/etc/containerd/conf.d/99-nvidia.toml` over editing the top-level file (it survives more image changes), treat "we also bumped Kubernetes" as a variable in the rollout plan, and canary a node-pool upgrade with a GPU workload before rolling the fleet.

## Connections & what's next

This lesson's cordon → evict → mutate → validate → uncordon pattern is the general shape the GPU Operator uses for *any* disruptive per-node change, and lesson 04.6 reuses it one layer lower: tearing down GPU Instances instead of a kernel module, for exactly the same reason — `nvmlGpuInstanceDestroy` returns `NVML_ERROR_IN_USE` when a process holds the instance, just as `delete_module(2)` returns `EBUSY` when a process holds the module. You will also see the same label-flipping eviction trick, with `paused-for-mig-change` in place of `paused-for-driver-upgrade`.

The `NVIDIADriver` CRD introduced here for heterogeneous fleets resurfaces in lesson 04.9, where multi-driver-version awareness matters again for mixed-generation clusters running the DRA driver — and where the DRA kubelet-plugin ordering constraint from the Real-world section becomes a first-class concern rather than a footnote. The economics thread is module 01's: every minute between `pod-deletion-required` and `uncordon-required` is a fully-billed idle GPU-hour, which is why `maxParallelUpgrades` and `maxUnavailable` are risk knobs and `T_idle` is the cost knob.

Next: **[04.6 · MIG operations](06-mig-operations.md)** applies the same "you cannot mutate this while a client holds it open" constraint to a GPU's *partition layout*. Reconfiguring MIG geometry is a drain-and-tear-down operation for structurally identical reasons, one layer below the kernel module — GPU Instances and Compute Instances rather than `nvidia.ko` — and it is where each unit of sharing finally gets its own UUID, which is what makes clean per-slice attribution possible.

## References & further reading

**Primary sources**
- [NVIDIA GPU Operator repository](https://github.com/NVIDIA/gpu-operator) — cloned and read this session at **v26.3.3**. Behind specific claims: `deployments/gpu-operator/values.yaml` (all Helm defaults in §4, including `drain.enable: false`, `maxUnavailable: 25%`, the `gpuPodDeletion` key name, `driver.version: "595.71.05"`, `k8s-driver-manager v0.11.0`, `toolkit v1.20.0`, `k8s-mig-manager v0.14.5`, `k8s-device-plugin v0.19.3`); `api/nvidia/v1/clusterpolicy_types.go` (`kernelModuleType` enum and default, the deprecated `useOpenKernelModules`); `api/nvidia/v1alpha1/nvidiadriver_types.go` (`driverType`/`usePrecompiled` immutability, `nodeSelector`, per-object `upgradePolicy`); `controllers/upgrade_controller.go` (the two-minute requeue, the drain-skip label selector, the `sandboxWorkloads` incompatibility); `cmd/gpu-operator/main.go` (the GPU-pod filter, the validator pod selector, the restart-only predicate registration); `controllers/object_controls.go` and `internal/config/driver_config_digest.go` and `internal/predicates/restart_only.go` (the `DRIVER_CONFIG_DIGEST` fast path); `controllers/operator_metrics.go` (the §10 metric names); `assets/state-operator-validation/0500_daemonset.yaml` (the driver→toolkit→cuda→plugin init-container chain and the `/run/nvidia/validations/*-ready` files).
- [`NVIDIA/k8s-operator-libs` — `pkg/upgrade`](https://github.com/NVIDIA/k8s-operator-libs/tree/main/pkg/upgrade) — read this session via the Operator's vendor tree; the authoritative source for the state machine. `consts.go` (every state name and annotation key, including the two maintenance-operator states); `upgrade_state.go` (`ApplyState`'s fixed handler order); `upgrade_inplace.go` (the `upgradesAvailable` gate, the already-cordoned carve-out, the restart-only routing); `common_manager.go` (`GetUpgradesAvailable`, `GetUpgradesInProgress`, `GetCurrentUnavailableNodes` — the formula in §4.1); `drain_manager.go` (the `kubectl` drain helper settings); `pod_manager.go` (pod-deletion filtering, job-completion waiting); `validation_manager.go` (**`validationTimeoutSeconds = 600`**, hard-coded); `safe_driver_load_manager.go` (the §5 handshake, described in full in that file's package comment).
- [`NVIDIA/k8s-operator-libs` — `api/upgrade/v1alpha1/upgrade_spec.go`](https://github.com/NVIDIA/k8s-operator-libs/blob/main/api/upgrade/v1alpha1/upgrade_spec.go) — read this session; the source of every API default in §4's table. *Correction: an earlier pass of this lesson presented `drain.enable: true` and `podDeletion` as the field names and defaults; the API defaults `drain.enable` to `false` and the Helm key is `gpuPodDeletion`.*
- [`NVIDIA/k8s-driver-manager`](https://github.com/NVIDIA/k8s-driver-manager) — cloned and read this session. `cmd/driver-manager/main.go` for the §1 module unload order, the `delete_module(2)` calls, `/sys/module/<name>/refcnt` probing, the `paused-for-driver-upgrade` label flipping and the per-component termination waits, the `DRIVER_CONFIG_DIGEST` vs `/run/nvidia/nvidia-driver.state` skip check, the host-driver detection path, and the DRA kubelet-plugin ordering constraints; `internal/linuxutils/kmod.go` for the `/proc/modules` refcount dump.
- [NVIDIA GPU Operator — GPU Driver Upgrades documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-driver-upgrades.html) — the canonical prose reference for this state machine and the `upgradePolicy` fields. *Not fetchable from this environment (the domain is blocked by the network egress proxy), so every field name, default and timeout in this lesson was taken from the source repositories above rather than from the docs. Re-verify against the docs for your exact Operator version before writing a production runbook.*
- [NVIDIA GPU Operator release notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html) — the source to check before pinning any version number from this lesson. *Not fetchable this session (blocked domain); the release line was instead read from the repository's git tags, which show `v24.6.x → v24.9.x → v25.3.x → v25.10.x → v26.3.0/.1/.2/.3`.*
- [NVIDIA GPU Operator — NVIDIA GPU Driver Custom Resource Definition (v23.9.0 documentation)](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/23.9.0/gpu-driver-configuration.html) — the `NVIDIADriver` CRD's introduction as a **technology preview in v23.9.0**, supporting multiple driver types/versions and multiple OS versions with node-selector routing, and noting that it does not support in-place upgrade from an earlier Operator version. *Confirmed via search snippets this session; the page itself is on a blocked domain. Correction: an earlier pass of this lesson attributed the CRD to v26.3.0 — it is four release lines older, and per the Operator's own README GA is still a roadmap item as of v26.3.3.*

**Real-world engineering evidence**
- [Google Cloud — "GKE can now automatically install NVIDIA GPU drivers"](https://cloud.google.com/blog/products/containers-kubernetes/gke-can-now-automatically-install-nvidia-gpu-drivers) — a hyperscaler productising automated, versioned driver installation as a node-pool flag (`gpu-driver-version=latest|default|disabled`), whose three-way choice mirrors the operator-managed-vs-pre-installed decision in §2. Google Cloud has also had a GPU-driver-fetch incident affecting GPU node pools. *The blog post was fetched in an earlier pass of this course; the incident's status page is not reachable from this environment — do not quote its date or duration without checking it yourself.*
- [NVIDIA/gpu-operator#626](https://github.com/NVIDIA/gpu-operator/issues/626), [#542](https://github.com/NVIDIA/gpu-operator/issues/542), [#901](https://github.com/NVIDIA/gpu-operator/issues/901), [#416](https://github.com/NVIDIA/gpu-operator/issues/416) — the upgrade-stall class: nodes stuck in `validation-required`/`pod-restart-required`; a version change producing zero nodes needing an upgrade on EKS; eviction and drain reported as "disabled by the upgrade policy"; a driver that would not start without a manual drain. *Titles and numbers found via search this session; issue bodies not independently fetched (GitHub API access is restricted in this environment). Use them as evidence that these failure modes are common; the mechanisms are derived from source above.*
- [NVIDIA/gpu-operator#1220 — "gpu-operator breaks when upgrading EKS to K8s v1.30"](https://github.com/NVIDIA/gpu-operator/issues/1220) — `failed to get sandbox runtime: no runtime for 'nvidia' is configured` plus a kernel-version-resolution failure after a managed control-plane upgrade, with no driver-version change involved. *Recorded in an earlier pass of this course; not independently re-fetched this session.*

**Deeper dives**
- [NVIDIA data-center driver branch lifecycle](https://docs.nvidia.com/datacenter/tesla/drivers/supported-drivers-and-cuda-toolkit-versions.html) — the NFB / Production Branch / LTSB distinction and per-branch end-of-life dates that anchor a "pin to what" decision. *Blocked domain; the two anchor facts used here — R580 is an LTSB supported through August 2028, and R590 Linux is an NFB ending 22 December 2026 — were confirmed via search snippets this session and are consistent with lesson 03.6's table. Re-verify at study time; branch dates move.*
- [`k8s.io/kubectl/pkg/drain`](https://github.com/kubernetes/kubectl/tree/master/pkg/drain) — the drain helper the upgrade controller uses directly. Worth reading if you want to know exactly what `IgnoreAllDaemonSets`, `GracePeriodSeconds: -1`, `Force` and `AdditionalFilters` do, since those four settings determine what a "drain" actually evicts on your cluster.
