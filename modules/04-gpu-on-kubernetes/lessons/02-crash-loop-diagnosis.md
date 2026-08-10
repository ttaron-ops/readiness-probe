---
lesson: "04.2"
title: "Crash-loop diagnosis: driver, toolkit, and device-plugin failures from logs alone"
module: "04"
concept: "Crash-loop diagnosis: driver, toolkit, and device-plugin failures from logs alone"
status: not-started
est_time: "12h"
prev: "01-gpu-operator-components.md"
next: "03-device-plugin-recap-pod-resources.md"
artifacts: []
sources: 7
---

# 04.2 · Crash-loop diagnosis: driver, toolkit, and device-plugin failures from logs alone

> **Concept.** When the GPU Operator breaks, the fix is a disciplined walk *up* the dependency chain to the first unhealthy stage — read its logs, name the root cause, and record it so the next person is faster.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

04.1 gave you the healthy chain: every operand, what gates it, and the sentinel-file mechanism that enforces order. That map is necessary but not sufficient — a map you've only ever seen converge cleanly hasn't taught you anything about failure. This lesson takes the same chain and breaks it three separate ways, on purpose, so you build the actual skill this module exists to teach: given a broken GPU node and nothing but `kubectl` and logs, localize the fault in minutes. Lesson 3 then moves past the Operator's own health entirely and into the API you'll consume once the chain is reliably green — the pod-resources socket that powers per-pod attribution.

## Why this matters

This module's README names the interview probe verbatim: *"name the GPU Operator components + debug a crash-looping driver pod."* That is not a hypothetical exercise — it is close to a literal transcript of what CoreWeave and NVIDIA platform interviews ask, and it is the incident you will actually get paged for at a GPU-heavy shop. The differentiator between a CKA holder and a senior GPU platform engineer is not knowing *that* things break — it's having a repeatable diagnosis order and a growing failure-mode log, so the third occurrence of a driver/kernel mismatch takes ninety seconds instead of an hour of guessing. This lesson builds both, and the failure-mode log you start here becomes literal input to your capstone's alerting logic in lesson 10.

The stakes are not abstract, either: [NVIDIA/gpu-operator#1220](https://github.com/NVIDIA/gpu-operator/issues/1220) is a real, filed GitHub issue where a practitioner's GPU Operator broke on an EKS Kubernetes 1.29→1.30 upgrade — a routine, unavoidable maintenance event, not a misconfiguration they invented. That kind of break happens to real fleets on real managed clouds, and the difference between an engineer who has internalized this lesson's diagnosis order and one who hasn't is measured in hours of downtime on production GPU capacity.

## What's new here (calibration)

This module's README is explicit that 02/02b/03 already own the underlying theory and this module is the operational integration layer. This lesson in particular does **not** re-teach:

- The device-plugin gRPC API or DRA object model (Module 02) — you already know what a healthy device plugin *does*; here you learn what it looks like when it *doesn't*.
- The driver/kernel-module relationship at the silicon level (Module 03) — background knowledge you now apply as evidence under time pressure, not as new material.
- The healthy dependency chain itself (04.1) — that lesson's map is the prerequisite; this lesson assumes you can already name every operand and its gate.

What this lesson adds: a **diagnostic discipline**, not new components — how to read a kernel-module build failure, how to tell three different `CrashLoopBackOff` causes apart from their *logs* (not guesses), how to prove the container runtime was actually rewired (not just patched on disk), and how to ground a failure-family taxonomy in a real, filed incident rather than a synthetic example. The durable artifact is the **failure-mode log entry**: symptom → evidence → root cause → fix → prevention.

## Core concepts

### The universal diagnosis order

Because operands are gated (04.1), **a Pending GPU pod or `Init` operand is a victim, not a culprit.** Always walk up the chain to the first stage that is not `Running`:

```bash
# 0. Is the resource even there?
kubectl describe node <node> | grep -A6 Allocatable        # nvidia.com/gpu: 0 ?

# 1. Whole-system snapshot — find the first non-Running / CrashLoop / Init:0-of-N
kubectl get pods -n gpu-operator -o wide

# 2. Read that pod. If it's Init, name the init container:
kubectl describe pod <pod> -n gpu-operator                 # which init is pending + why
kubectl logs <pod> -n gpu-operator -c <container>           # the actual error

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
  or, for a precompiled image against the wrong kernel, the exact string that also appears in the real EKS-1.30 incident below: `Could not resolve Linux kernel version`.
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

**How you tell them apart:** 1a shows *compiler/make* errors or an explicit kernel-version-resolution error; 1b shows *insert/in-use* errors; 1c shows *Key was rejected*. Fix for 1a is aligning `driver.version` to a branch that supports the node kernel (or matching kernel headers / using a matching precompiled image); 1b is unloading the stale module or draining processes; 1c is enrolling the signing key or disabling Secure Boot.

### Failure family 2 — container-toolkit / container-runtime wiring broken

The toolkit DS rewires the runtime. Two failure shapes:
- **Toolkit pod itself crashes** → `toolkit-ready` never written → device-plugin/GFD/DCGM/validator all stuck `Init`. Logs show `nvidia-ctk` unable to find or parse the runtime config, or unable to restart containerd. The real EKS-1.30 incident below shows a variant of this family: the runtime wiring reports itself unable to find the configured GPU runtime at all, with the log line `failed to get sandbox runtime: no runtime for 'nvidia' is configured` — the runtime class the toolkit was supposed to register simply isn't there from containerd's point of view.
- **Toolkit "succeeds" but the runtime is wrong/corrupt** → toolkit goes Running, but *GPU pods* fail at container create:
  ```
  Error: failed to create containerd task: failed to create shim task:
  OCI runtime create failed: ... exec: "nvidia-container-runtime-hook":
  executable file not found in $PATH
  ```
  or the pod runs but `nvidia-smi` inside it reports `command not found` / no devices — meaning containerd used plain `runc`, not the nvidia runtime. (On GPU Operator 25.10+, remember from 04.1 that CDI is the default injection path rather than this legacy hook — before assuming this exact log line applies, run `nvidia-ctk cdi list` to check which mechanism your version is actually using.)

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

### A real incident: NVIDIA/gpu-operator#1220 — EKS upgrade breaks the chain in two places at once

Grounding the taxonomy above in a synthetic example is useful for practice, but the real world is messier, and it's worth seeing the messiness once. [NVIDIA/gpu-operator#1220](https://github.com/NVIDIA/gpu-operator/issues/1220), a real filed GitHub issue titled "gpu-operator breaks when upgrading EKS to K8s v1.30," reports a fleet that had been running fine on Kubernetes 1.29 and broke on the routine 1.30 upgrade. The reporter's environment: Kubernetes v1.30, GPU Operator v24.6.2, NVIDIA driver 535.183.01, Ubuntu 22.04 with kernel 6.5.0-1020-aws. Two symptom log lines appear together:

```
failed to get sandbox runtime: no runtime for 'nvidia' is configured
```
```
Could not resolve Linux kernel version
```

Read against this lesson's taxonomy: the first line is a family-2 symptom — the container runtime doesn't see the `nvidia` runtime class registered, which is exactly the "toolkit reports success but the wiring isn't actually live" shape, just surfaced as a *scheduling*-time error instead of a create-time one. The second line is a textbook family-1a symptom — the driver pod cannot resolve/match against the running kernel, the same failure category as the worked example below. What makes this incident worth studying isn't that it fits the taxonomy — it's that a routine, unavoidable, vendor-driven event (a managed Kubernetes version bump) triggered *two* failure families simultaneously on a fleet that hadn't changed anything else. That's the real-world case for walking the chain methodically rather than trying to intuit "the" cause: on a live incident, you may be looking at more than one broken stage at once.

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

### A note on version sensitivity, and where to look before assuming you're uniquely broken

This area drifts release to release, so anchor your evidence to the running version (`kubectl -n gpu-operator get clusterpolicy -o jsonpath='{.metadata.labels}'`, or the operator image tag) rather than to any lesson's snapshot of it. As of mid-2026 the public GPU Operator line has moved from 25.3.x through 25.10.x (CDI-by-default) to 26.3.x (the `NVIDIADriver` CRD, faster same-version driver-pod restarts) — see 04.1's version note for the full detail. When a log line doesn't match what you expect, your first move should be checking [NVIDIA/gpu-operator's GitHub issues](https://github.com/NVIDIA/gpu-operator/issues) filtered to your exact operator version and Kubernetes version, not assuming your cluster is uniquely broken. Issue #1220 above is a real example of exactly this: someone else, on a specific version combination, already hit and documented the failure you might be looking at. Init-container names and sentinel-file paths have been stable across the 24.x–26.x lines, but never hard-code them into runbooks without confirming against the deployed version.

## Perspectives

**On-call/SRE-under-time-pressure perspective.** "Walk up the chain, never start at the Pending pod" is a triage protocol, not a preference — it's what lets you localize a fault before you've fully woken up. The value of the failure-mode log is precisely that it converts a first-occurrence hour-long investigation into a second-occurrence lookup.

**Kernel/systems perspective.** `rmmod` returning `EBUSY`, DKMS build failures, and Secure Boot MOK enrollment are Linux kernel-module facts wearing a Kubernetes costume. Nothing about family 1 is Kubernetes-specific — it's exactly what you'd hit running `nvidia-smi` on bare metal after a kernel upgrade, just surfaced through a DaemonSet's exit code and a `CrashLoopBackOff` status instead of a shell prompt.

**Fleet-scale/statistics perspective.** A single node hitting a driver/kernel mismatch is an inconvenience. The same failure recurring across a 200-node fleet after a coordinated driver bump is exactly why the failure-mode log exists as a structured artifact rather than tribal memory — it's the difference between one engineer relearning the fix five times and a team learning it once.

**Cross-cloud portability perspective.** #1220's EKS-1.30 incident is not evidence that AWS's managed Kubernetes is uniquely fragile — it's evidence that this exact failure family recurs, differently shaped, across every managed Kubernetes vendor whenever the control-plane version moves out from under a driver/kernel pairing that was tuned for the old one. The taxonomy in this lesson is vendor-agnostic precisely because the underlying mechanism (kernel module vs. running kernel, runtime config vs. live containerd state) is vendor-agnostic.

## Real-world use cases

- **[NVIDIA/gpu-operator#1220 — "gpu-operator breaks when upgrading EKS to K8s v1.30"](https://github.com/NVIDIA/gpu-operator/issues/1220).** A real, filed practitioner incident (not a blog) with exact log lines and a specific version fingerprint (K8s 1.30, GPU Operator v24.6.2, driver 535.183.01, Ubuntu 22.04 kernel 6.5.0-1020-aws). What it shows: a routine, vendor-driven Kubernetes version bump can trip both the driver family and the runtime-wiring family at once — real evidence that the taxonomy above isn't academic, and a concrete demonstration of why "check the issue tracker for your exact version pair" belongs in your diagnosis order.
- **[NVIDIA/gpu-operator GitHub issues](https://github.com/NVIDIA/gpu-operator/issues) generally.** Not a single incident but a resource: a searchable archive of practitioners' break/fix threads across driver, toolkit, and plugin failures. What it shows: for almost any log line you don't recognize, someone with your exact operator/Kubernetes version combination has likely already hit it and posted the fix.
- A note on a gap, deliberately left honest: this research could not find or confirm a tier-1 engineering-blog postmortem (Meta/Netflix/Discord-style, with full incident timeline and prose) specifically narrating a GPU-Operator crash loop. The strongest available real-world evidence for this lesson is the filed GitHub issue above — arguably *better* evidence than a blog post for this purpose, since it's primary, dated, and exactly on-topic, but it's a different genre than a polished postmortem, and it's worth knowing that gap exists rather than assuming a blog you haven't seen backs this up.

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
Compiler/make errors → **family 1a**, kernel/driver-branch mismatch (not in-use, not signing). Downstream `Init` pods are victims. This is the same family as issue #1220's `Could not resolve Linux kernel version` line above — a different specific error string, but the same underlying mismatch between what the driver container expects and what the host kernel actually is.

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
The chain snapshot is all-Running, so this is **family 2, second shape** — toolkit reported success but containerd is using plain `runc`. This is the same family as #1220's `failed to get sandbox runtime: no runtime for 'nvidia' is configured` line, just caught at container-create time instead of at pod-scheduling time — both are "the runtime class the toolkit was supposed to register isn't actually live in containerd." Prove it on the node:
```
$ sudo crictl info | grep -i nvidia            # (empty) -> nvidia runtime not registered live
$ sudo grep -A3 runtimes.nvidia /etc/containerd/config.toml   # present on disk...
```
Config on disk but not in the live daemon ⇒ containerd wasn't restarted after the patch. Fix by deleting the toolkit pod so it re-runs `nvidia-ctk` and restarts containerd, then re-confirm with `crictl info`. The lesson: for family 2, the deciding evidence is **`crictl info`, not the config file** — disk state lies.

## Practice

On your single-node rented-GPU cluster from 04.1, **deliberately break and recover each**, capturing logs and writing a failure-mode-log entry for every one — these entries are the direct seed of the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable's required `failure-mode-log.md`:

1. **Driver/kernel mismatch (family 1).** `helm upgrade ... --set driver.version=<incompatible>`, delete the driver pod, and read the kmod **build/load** failure. Recover by pinning a compatible branch (or `driver.enabled=false` against the host driver).
2. **Toolkit/containerd config (family 2).** On the node, corrupt or delete the `runtimes.nvidia` block in `/etc/containerd/config.toml` (back it up first) and restart containerd — then deploy a GPU pod and capture the `nvidia-container-runtime-hook: executable file not found` create failure. Recover by re-running the toolkit (delete the toolkit pod so it re-patches) and confirming with `crictl info`.
3. **Won't register (family 3).** Label the node `nvidia.com/gpu.deploy.device-plugin=false` **or** add a `NoSchedule` taint the plugin won't tolerate; watch `nvidia.com/gpu` fall to `0` and the plugin pod disappear/stay off-node. Recover by restoring the label / removing the taint.
4. **Cross-reference against a real incident.** Read [NVIDIA/gpu-operator#1220](https://github.com/NVIDIA/gpu-operator/issues/1220) in full and write two sentences mapping its two log lines onto the failure families you just reproduced by hand.

Capture `kubectl get pods -n gpu-operator`, the relevant `kubectl logs`/`describe`, and the before/after `nvidia.com/gpu` value for each.

**Acceptance:** three documented break/fix entries in the deliverable's **failure-mode log**, each with all five fields (symptom, evidence, root cause, fix, prevention) and the real log line that was the deciding evidence, plus the #1220 cross-reference note. Restore the cluster to a healthy `cuda-vectoradd → Test PASSED` state at the end.

## Common pitfalls

1. **Debugging the Pending workload pod first.** It almost never has useful information — `kubectl describe` on it will say "Insufficient nvidia.com/gpu" or similar and stop there. The real evidence is upstream in `gpu-operator`.
2. **Treating the current container's logs as the crash's logs on a `CrashLoopBackOff` pod.** The current attempt may be seconds old and still initializing; always check `--previous` for the crash that actually happened.
3. **Trusting `/etc/containerd/config.toml` on disk as proof the runtime is wired.** A config on disk that containerd never reloaded is a common and genuinely confusing "looks fine but GPU pods still fail" trap — `crictl info` against the *live* daemon is the only evidence that counts.
4. **Assuming a failure is a novel mystery instead of checking whether someone else already hit it.** [NVIDIA/gpu-operator's issue tracker](https://github.com/NVIDIA/gpu-operator/issues), filtered to your exact operator version and Kubernetes version, is a legitimate first diagnostic step — #1220 is proof that specific, dated, version-pinned incidents get filed and documented there.
5. **Assuming every driver-pod restart means a multi-minute kernel-module rebuild.** True through the 25.x line, but as of GPU Operator 26.3.0, a driver-pod restart that doesn't actually change the driver version reuses already-loaded kernel modules — recovery can be seconds, not minutes. Check the operand's actual behavior on your version before estimating an incident's expected recovery time.

## Self-check

- GPU pods stuck Pending and `nvidia.com/gpu` shows 0 allocatable — walk your diagnosis order. **Answer:** Don't touch the Pending pod. (1) `kubectl describe node` — confirm allocatable is 0, not a scheduling/tolerations issue on the workload. (2) `kubectl get pods -n gpu-operator` — find the first stage that is not Running. If the **device plugin is `Init`**, walk up: it's waiting on `toolkit-ready`, so the real fault is the toolkit or the driver — read those. If the **plugin is absent or Running-but-not-registering**, check node labels (`nvidia.com/gpu.deploy.device-plugin`, `pci-10de.present`) and taints, then the plugin's own logs. Fix the *first* broken stage; the downstream `Init` pods recover on their own.
- The driver pod is `CrashLoopBackOff` — top 3 causes and how you tell them apart. **Answer:** (1) **Kernel/headers or precompiled-driver mismatch** — logs show *compiler/make* errors (`make: *** [__modpost] Error 2`, `error:` in `nv-*.c`) or `Could not resolve Linux kernel version`; fix by aligning `driver.version` to a branch supporting the node kernel. (2) **Conflicting/stale module in use** — logs show `could not insert 'nvidia': Module already in use / Resource busy` (nouveau not blacklisted or old module pinned); fix by unloading via `k8s-driver-manager` / draining the holder. (3) **Secure Boot rejection** — logs show `Key was rejected by service`; fix by enrolling the MOK signing key or disabling Secure Boot. The log line is the discriminator: *make/compile or kernel-version-resolution* vs *insert/in-use* vs *key rejected*.
- Where does the container-toolkit write its containerd/CRI config, and how do you verify it took effect? **Answer:** It patches the host file **`/etc/containerd/config.toml`**, adding a `runtimes.nvidia` block with `BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime"` and setting `default_runtime_name = "nvidia"` (driven by the daemonset's `CONTAINERD_CONFIG` / `CONTAINERD_SOCKET` / `CONTAINERD_SET_AS_DEFAULT` env). Verify **on disk** with `grep -A3 runtimes.nvidia /etc/containerd/config.toml`, and — crucially — verify it **took effect in the running daemon** with `sudo crictl info | grep -i nvidia`, since a config on disk that containerd wasn't restarted to reload is a common "looks fine but GPU pods still fail" trap.
- What does NVIDIA/gpu-operator#1220 show, and why is it useful evidence beyond just "another driver bug"? **Answer:** It's a real, filed incident where a fleet running fine on Kubernetes 1.29 broke on the routine, vendor-driven upgrade to 1.30 (GPU Operator v24.6.2, driver 535.183.01, Ubuntu 22.04 kernel 6.5.0-1020-aws), producing two log lines simultaneously: `failed to get sandbox runtime: no runtime for 'nvidia' is configured` (family 2 — runtime-wiring) and `Could not resolve Linux kernel version` (family 1a — driver/kernel mismatch). It's useful beyond "another driver bug" because it proves two things this lesson depends on: the taxonomy holds against a real, non-synthetic incident, and a single routine event (a managed-Kubernetes version bump) can trip more than one failure family at once — which is exactly why you walk the whole chain methodically rather than stopping at the first plausible-looking error.
- Before spending an hour debugging an unfamiliar log line, what's a legitimate first move, and why? **Answer:** Search [NVIDIA/gpu-operator's GitHub issues](https://github.com/NVIDIA/gpu-operator/issues) filtered to your exact GPU Operator version and Kubernetes version. It's legitimate, not a shortcut around understanding, because issue #1220 demonstrates that specific, dated, version-pinned incidents with exact log strings genuinely do get filed there by other practitioners — matching your log line to an existing issue can save the diagnostic work of re-deriving a root cause someone else already found, and still requires you to understand *why* the fix works before applying it to production.

## Connections & what's next

The diagnosis order here — walk up the chain, trust logs and live daemon state over disk state and appearances — is the habit every later break/fix lesson in this module reuses: lesson 5's driver-upgrade failures, lesson 6's MIG reconfiguration failures, and lesson 9's DRA scheduling failures are all "a stage in a gated chain broke" in different clothing. The failure-mode log format started here is the exact artifact lesson 5 and lesson 6 add entries to, and the one your capstone (lesson 10) assembles into `failure-mode-log.md` with at least five real break/fix entries.

Next: **[04.3 · Device-plugin recap and the pod-resources API](03-device-plugin-recap-pod-resources.md)** moves past the Operator's own health — which you can now diagnose cold — and into the API surface you'll actually build against: the kubelet pod-resources socket that tells you which pod holds which GPU device, the foundation of per-pod cost attribution.

## References & further reading

**Primary sources**
- [NVIDIA/gpu-operator issue #1220 — "gpu-operator breaks when upgrading EKS to K8s v1.30"](https://github.com/NVIDIA/gpu-operator/issues/1220) — the real, filed incident this lesson's worked examples are cross-referenced against; read for the exact log lines and version fingerprint.
- [NVIDIA/gpu-operator GitHub issues](https://github.com/NVIDIA/gpu-operator/issues) — read as a searchable diagnostic resource: filter to your exact operator + Kubernetes version before assuming a log line is unprecedented.
- [NVIDIA GPU Operator docs — Troubleshooting](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/troubleshooting.html) — the vendor's own symptom-to-cause table for driver, toolkit, and validator failures. *Not independently fetched this session* (proxy-blocked domain) — verify field names and paths against your resolved version at study time.
- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) — confirmed at v0.17.1 this session; the source of truth for device-plugin registration behavior and log strings when the plugin runs but advertises 0 GPUs.

**Real-world engineering blogs**
- [NVIDIA/gpu-operator issue #1220](https://github.com/NVIDIA/gpu-operator/issues/1220) — treated here as the flagship real-world evidence for this lesson: a primary, dated, exactly-on-topic practitioner incident, stronger than any blog summary because it's the unedited original report. See note above on why a tier-1 engineering-blog postmortem specifically about a GPU-Operator crash loop could not be found or confirmed this session — this issue is the honest best available evidence.

**Deeper dives**
- [NVIDIA GPU Operator docs — Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/release-notes.html) — cross-check which behaviors (CDI-default, faster same-version driver restarts) are active on the version you're debugging before assuming a fix from an older lesson or issue thread still applies verbatim. *Not independently fetched this session* (proxy-blocked domain).
- [NVIDIA GPU Operator docs — Getting Started](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html) — the install/chart-values reference from 04.1; useful to re-check exact env var and CR field names when a log references a setting you don't recognize. *Not independently fetched this session.*
