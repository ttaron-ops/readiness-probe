---
lesson: "04.2"
title: "Crash-loop diagnosis: driver, toolkit, and device-plugin failures from logs alone"
module: "04"
concept: "Crash-loop diagnosis: driver, toolkit, and device-plugin failures from logs alone"
status: not-started
est_time: "10h"
artifacts: []
---

# 04.2 · Crash-loop diagnosis: driver, toolkit, and device-plugin failures from logs alone

> **Concept.** When the GPU Operator breaks, the fix is a disciplined walk *up* the dependency chain to the first unhealthy stage — read its logs, name the root cause, and record it so the next person is faster.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters
This is the highest-ROI interview and on-call skill in the module: "GPU pods are Pending, allocatable is 0 — go" is a question you will be asked verbatim, and it is the incident you will actually get paged for at a GPU-heavy shop. The differentiator between a CKA and a senior GPU platform engineer is not knowing *that* things break, but having a **repeatable diagnosis order** and a **failure-mode log** so the third occurrence takes ninety seconds. This lesson builds both, and it seeds the failure-mode log that your capstone cost operator's alerting will one day reference.

## What's new here
04.1 gave you the healthy chain and each operand's logs. This lesson operates on the **broken** chain. You are not learning new components — you are learning a **diagnostic discipline** on the components you already know: how to read a kernel-module build failure, how to tell three different `CrashLoopBackOff` causes apart from their *logs* (not guesses), and how to prove the container runtime was actually rewired. Module 02's device-plugin API and Module 03's driver/kmod material are the *background knowledge*; here they become *evidence you interpret under time pressure*. The durable artifact is the **failure-mode log entry**: symptom → evidence → root cause → fix → prevention.

## Core notes

### The universal diagnosis order
Because operands are gated (04.1), **a Pending GPU pod or `Init` operand is a victim, not a culprit.** Always walk up the chain to the first stage that is not `Running`:

```bash
# 0. Is the resource even there?
kubectl describe node <node> | grep -A6 Allocatable        # nvidia.com/gpu: 0 ?

# 1. Whole-system snapshot — find the first non-Running / CrashLoop / Init:0-of-N
kubectl get pods -n gpu-operator -o wide

# 2. Read that pod. If it's Init, name the init container:
kubectl describe pod <pod> -n gpu-operator                 # which init is pending + why
kubectl logs <pod> -n gpu-operator -c <container>          # the actual error

# 3. For placement problems, check labels/taints the operator keys off of:
kubectl get node <node> --show-labels | tr ',' '\n' | grep nvidia
kubectl describe node <node> | grep -i taint
```
The order is invariant: **describe node → snapshot → first broken stage → its logs → labels/taints.** Never start debugging at the Pending workload pod.

Two habits make this fast. First, prefer `logs --previous` on a `CrashLoopBackOff` pod — the *current* container may be a fresh, still-initializing attempt; the crash you care about is in the previous instance. Second, when a pod is in `Init`, `kubectl describe` tells you *which* init container is blocking and often prints its wait message; go straight to that container's logs (`-c <name>`) rather than the main container, which hasn't started.

### Failure family 1 — driver pod `CrashLoopBackOff`
The driver DaemonSet builds and loads kernel modules against the **running host kernel**. Almost all crashes are one of three, distinguishable by log:

- **(1a) Kernel/headers mismatch or precompiled-driver mismatch.** The container's driver sources or precompiled `.ko` don't match `uname -r`. Logs from `kubectl logs ds/nvidia-driver-daemonset`:
  ```
  Compiling NVIDIA driver kernel modules...
  /usr/src/nvidia-<ver>/nvidia/nv.c:XX: error: ...
  make: *** [__modpost] Error 2
  Unable to install the NVIDIA driver: kernel module failed to build
  ```
  or, for a precompiled image against the wrong kernel: `Could not resolve Linux kernel version` / `no precompiled module for kernel <ver>`.
- **(1b) Conflicting/stale module in use.** `nouveau` not blacklisted, or an old `nvidia.ko` still resident:
  ```
  modprobe: ERROR: could not insert 'nvidia': Module already in use / Resource busy
  NVIDIA driver is already loaded, cannot install
  ```
  The `k8s-driver-manager` init container is what *should* rmmod these on upgrade; if it can't (module pinned by a running process), the main container loops.
- **(1c) Secure Boot / unsigned module rejected.**
  ```
  modprobe: ERROR: could not insert 'nvidia': Key was rejected by service
  ```
  MOK enrollment missing. On-prem gotcha.

**How you tell them apart:** 1a shows *compiler/make* errors; 1b shows *insert/in-use* errors; 1c shows *Key was rejected*. Fix for 1a is aligning `driver.version` to a branch that supports the node kernel (or matching kernel headers / using a matching precompiled image); 1b is unloading the stale module or draining processes; 1c is enrolling the signing key or disabling Secure Boot.

### Failure family 2 — container-toolkit / containerd config broken
The toolkit DS rewrites the runtime. Two failure shapes:
- **Toolkit pod itself crashes** → `toolkit-ready` never written → device-plugin/GFD/DCGM/validator all stuck `Init`. Logs show `nvidia-ctk` unable to find or parse `/etc/containerd/config.toml`, or unable to restart containerd.
- **Toolkit "succeeds" but the runtime is wrong/corrupt** → toolkit goes Running, but *GPU pods* fail at container create:
  ```
  Error: failed to create containerd task: failed to create shim task:
  OCI runtime create failed: ... exec: "nvidia-container-runtime-hook":
  executable file not found in $PATH
  ```
  or the pod runs but `nvidia-smi` inside it reports `command not found` / no devices — meaning containerd used plain `runc`, not the nvidia runtime.

**Where the config lives / how to verify it took:** the toolkit patches the **host** file `/etc/containerd/config.toml`, adding a `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]` block whose `BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime"`, and sets `default_runtime_name = "nvidia"`. The daemonset is driven by env: `RUNTIME=containerd`, `CONTAINERD_CONFIG=/etc/containerd/config.toml`, `CONTAINERD_SOCKET=/run/containerd/containerd.sock`, `CONTAINERD_RUNTIME_CLASS=nvidia`, `CONTAINERD_SET_AS_DEFAULT=true`. Verify from the node:
```bash
sudo grep -A3 'runtimes.nvidia' /etc/containerd/config.toml
sudo crictl info | grep -A5 -i nvidia          # runtime registered in live containerd
```
If the file has the block but the live containerd doesn't, containerd wasn't restarted — the config didn't *take effect* even though it's on disk.

### Failure family 3 — device plugin won't register (`nvidia.com/gpu` = 0)
Resource shows `0` allocatable. Split into two very different roots:
- **Gated:** device-plugin pod is `Init:0/1` — its `toolkit-validation` is waiting on `toolkit-ready`. This is really a family-1 or family-2 failure upstream. Walk up.
- **Not scheduled / not registering:** the pod isn't there at all, or is Running but kubelet never registered it. Causes:
  - **NFD/deploy labels missing or set false:** `nvidia.com/gpu.deploy.device-plugin=false`, or the node lost `feature.node.kubernetes.io/pci-10de.present`. The operator then never schedules (or actively removes) the operand. Check `--show-labels`.
  - **Taint the plugin doesn't tolerate:** a `NoSchedule` taint you added (or a GPU taint the pod's tolerations don't match) keeps the DaemonSet pod off the node.
  - **Registration failure:** plugin Running but `kubectl logs ds/nvidia-device-plugin-daemonset` shows it couldn't reach `/var/lib/kubelet/device-plugins/kubelet.sock` or found no GPUs (driver actually broken). `nvidia.com/gpu` stays 0.

### Reading the device-plugin registration path
When the plugin is Running but `nvidia.com/gpu` is still 0, the fault is between the plugin and kubelet, and the evidence is specific. Healthy logs (`kubectl logs ds/nvidia-device-plugin-daemonset`) show it enumerating GPUs and registering:
```
Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
Registered device plugin for 'nvidia.com/gpu' with Kubelet
```
The two broken shapes: (1) **no GPUs found** — `could not load NVML library` / `no GPU devices found`, meaning the *driver* is actually broken even though the driver pod looks up (walk up the chain, run `nvidia-smi` in the driver pod); (2) **can't reach kubelet** — errors touching `/var/lib/kubelet/device-plugins/kubelet.sock`, usually a hostPath/mount or a kubelet that restarted without the plugin re-registering (delete the plugin pod to force re-registration). Distinguishing these from the logs is the difference between "fix the driver" and "restart the plugin."

### The failure-mode log (the durable artifact)
For every incident, record five fields. This is the format your capstone will consume:
```
SYMPTOM:     GPU pods Pending; nvidia.com/gpu allocatable = 0
EVIDENCE:    driver-daemonset CrashLoopBackOff; logs show "make: *** [__modpost] Error 2"
ROOT CAUSE:  driver.version 550 built against node kernel 6.8; headers/branch mismatch
FIX:         pinned driver.version to a branch supporting 6.8 (or set driver.enabled=false, use host driver); delete pod to rebuild
PREVENTION:  gate driver upgrades on a kernel-compatibility check in CI; alert on driver DS restarts
```
Prevention is what separates senior work from firefighting — it's also the hook for the cost/observability operator (alert on operand restarts, on `nvidia.com/gpu` dropping to 0).

### Confirming a fix actually landed (not just "pod is green")
"The driver pod is Running" is necessary, not sufficient. The node is only genuinely fixed when a real kernel executes. Three-tier confirmation, cheapest first:
```bash
# Tier 1: resource restored
kubectl get node <node> -o jsonpath='{.status.allocatable.nvidia\.com/gpu}'   # -> 1

# Tier 2: driver sees the hardware (exec into the driver pod)
kubectl -n gpu-operator exec ds/nvidia-driver-daemonset -- nvidia-smi         # lists the GPU

# Tier 3: end-to-end CUDA (the validator already does this, or run it yourself)
kubectl logs nvidia-cuda-validator-xxxxx -n gpu-operator                       # "Test PASSED"
```
Tiers 1 and 2 can both pass while the *runtime* is still mis-wired (family 2) — only tier 3, a container that actually got a device via the nvidia runtime, proves the full path. Make tier 3 the acceptance bar for every recovery.

### A note on version sensitivity
This area drifts release-to-release, so anchor your evidence to the running version (`kubectl -n gpu-operator get clusterpolicy -o jsonpath='{.metadata.labels}'`, or the operator image tag). On **GPU Operator 25.3.x** the toolkit installs to `/usr/local/nvidia/toolkit` and there are known 25.3-line quirks (e.g. issues honoring `CONTAINERD_SOCKET` overrides in some point releases) — when a log doesn't match the docs, check the operator's GitHub issues *filtered to your exact tag* before assuming your cluster is uniquely broken. Init-container names and sentinel-file paths have been stable across the 24.x–25.x lines, but never hard-code them into runbooks without confirming against the deployed version.

## Worked example
**Break:** pin an incompatible driver.
```bash
helm upgrade gpu-operator nvidia/gpu-operator -n gpu-operator \
  --reuse-values --set driver.version=525.60.13   # too old for this node's kernel
kubectl delete pod -n gpu-operator -l app=nvidia-driver-daemonset
```
**Observe:**
```
$ kubectl get pods -n gpu-operator
nvidia-driver-daemonset-abcde             0/1   CrashLoopBackOff   4     6m
nvidia-container-toolkit-daemonset-fghij  0/1   Init:0/1           0     6m
nvidia-device-plugin-daemonset-klmno      0/1   Init:0/1           0     6m
nvidia-operator-validator-pqrst           0/1   Init:0/4           0     6m
$ kubectl describe node | grep -A4 Allocatable
  nvidia.com/gpu:  0
```
**Diagnose (up the chain):** first non-Running is the driver DS. Read it:
```
$ kubectl -n gpu-operator logs nvidia-driver-daemonset-abcde --previous | tail -6
Compiling NVIDIA driver kernel modules...
/usr/src/nvidia-525.60.13/nvidia/nv-mmap.c:XXX: error: too many arguments to function
make[1]: *** [scripts/Makefile.build] Error 1
make: *** [__modpost] Error 2
Unable to load the kernel module 'nvidia.ko'.
```
Compiler/make errors → **family 1a**, kernel/driver-branch mismatch (not in-use, not signing). Downstream `Init` pods are victims.
**Fix:**
```bash
helm upgrade gpu-operator nvidia/gpu-operator -n gpu-operator \
  --reuse-values --set driver.version=<branch-supporting-this-kernel>
```
Driver rebuilds, writes `driver-ready`, the chain drains, `nvidia.com/gpu` returns to `1`, validator goes Running. **Log entry** written per the five-field format above.

**Second trace — the runtime that "succeeded" but didn't wire.** A subtler incident: every operand is green, `nvidia.com/gpu` is `1`, yet a fresh GPU pod fails at create.
```
$ kubectl describe pod cuda-vectoradd | grep -A2 Warning
  Warning  Failed  ...  Error: failed to create containerd task: OCI runtime create failed:
  ... exec: "nvidia-container-runtime-hook": executable file not found in $PATH
```
The chain snapshot is all-Running, so this is **family 2, second shape** — toolkit reported success but containerd is using plain `runc`. Prove it on the node:
```
$ sudo crictl info | grep -i nvidia            # (empty) -> nvidia runtime not registered live
$ sudo grep -A3 runtimes.nvidia /etc/containerd/config.toml   # present on disk...
```
Config on disk but not in the live daemon ⇒ containerd wasn't restarted after the patch. Fix by deleting the toolkit pod so it re-runs `nvidia-ctk` and restarts containerd, then re-confirm with `crictl info`. The lesson: for family 2, the deciding evidence is **`crictl info`, not the config file** — disk state lies.

## Practice
On your single-node rented-GPU cluster from 04.1, **deliberately break and recover each**, capturing logs and writing a failure-mode-log entry for every one:

1. **Driver/kernel mismatch (family 1).** `helm upgrade ... --set driver.version=<incompatible>`, delete the driver pod, and read the kmod **build/load** failure. Recover by pinning a compatible branch (or `driver.enabled=false` against the host driver).
2. **Toolkit/containerd config (family 2).** On the node, corrupt or delete the `runtimes.nvidia` block in `/etc/containerd/config.toml` (back it up first) and restart containerd — then deploy a GPU pod and capture the `nvidia-container-runtime-hook: executable file not found` create failure. Recover by re-running the toolkit (delete the toolkit pod so it re-patches) and confirming with `crictl info`.
3. **Won't register (family 3).** Label the node `nvidia.com/gpu.deploy.device-plugin=false` **or** add a `NoSchedule` taint the plugin won't tolerate; watch `nvidia.com/gpu` fall to `0` and the plugin pod disappear/stay off-node. Recover by restoring the label / removing the taint.

Capture `kubectl get pods -n gpu-operator`, the relevant `kubectl logs`/`describe`, and the before/after `nvidia.com/gpu` value for each.

**Acceptance:** three documented break/fix entries in the deliverable's **failure-mode log**, each with all five fields (symptom, evidence, root cause, fix, prevention) and the real log line that was the deciding evidence. Restore the cluster to a healthy `cuda-vectoradd → Test PASSED` state at the end.

## Self-check
**(a) GPU pods stuck Pending and `nvidia.com/gpu` shows 0 allocatable — walk your diagnosis order.**
**Answer:** Don't touch the Pending pod. (1) `kubectl describe node` — confirm allocatable is 0, not a scheduling/tolerations issue on the workload. (2) `kubectl get pods -n gpu-operator` — find the first stage that is not Running. If the **device plugin is `Init`**, walk up: it's waiting on `toolkit-ready`, so the real fault is the toolkit or the driver — read those. If the **plugin is absent or Running-but-not-registering**, check node labels (`nvidia.com/gpu.deploy.device-plugin`, `pci-10de.present`) and taints, then the plugin's own logs. Fix the *first* broken stage; the downstream `Init` pods recover on their own.

**(b) The driver pod is `CrashLoopBackOff` — top 3 causes and how you tell them apart.**
**Answer:** (1) **Kernel/headers or precompiled-driver mismatch** — logs show *compiler/make* errors (`make: *** [__modpost] Error 2`, `error:` in `nv-*.c`); fix by aligning `driver.version` to a branch supporting the node kernel. (2) **Conflicting/stale module in use** — logs show `could not insert 'nvidia': Module already in use / Resource busy` (nouveau not blacklisted or old module pinned); fix by unloading via `k8s-driver-manager` / draining the holder. (3) **Secure Boot rejection** — logs show `Key was rejected by service`; fix by enrolling the MOK signing key or disabling Secure Boot. The log line is the discriminator: *make/compile* vs *insert/in-use* vs *key rejected*.

**(c) Where does the container-toolkit write its containerd/CRI config, and how do you verify it took effect?**
**Answer:** It patches the host file **`/etc/containerd/config.toml`**, adding a `runtimes.nvidia` block with `BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime"` and setting `default_runtime_name = "nvidia"` (driven by the daemonset's `CONTAINERD_CONFIG` / `CONTAINERD_SOCKET` / `CONTAINERD_SET_AS_DEFAULT` env). Verify **on disk** with `grep -A3 runtimes.nvidia /etc/containerd/config.toml`, and — crucially — verify it **took effect in the running daemon** with `sudo crictl info | grep -i nvidia`, since a config on disk that containerd wasn't restarted to reload is a common "looks fine but GPU pods still fail" trap.

## Resources
1. **NVIDIA GPU Operator — Troubleshooting** — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/troubleshooting.html — *deep.* The canonical symptom→cause table for driver, toolkit, and validator failures; mirror its checks into your failure-mode log. Pin the `/25.3/` path for version-accurate steps.
2. **NVIDIA/k8s-device-plugin** — https://github.com/NVIDIA/k8s-device-plugin — *deep.* Source and README for the plugin whose registration you're debugging; the place to confirm `nvidia.com/gpu` registration behavior and env flags when the plugin runs but advertises 0.
3. **NVIDIA/gpu-operator GitHub issues (driver/kernel-mismatch threads)** — https://github.com/NVIDIA/gpu-operator/issues — *skim.* Real practitioners' break/fix threads (e.g. driver kmod build failures, `nvidia-container-runtime-hook` errors); the fastest way to match a log line you've never seen to a known root cause.
