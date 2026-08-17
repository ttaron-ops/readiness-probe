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
sources: 11
---

# 04.2 · Crash-loop diagnosis: driver, toolkit, and device-plugin failures from logs alone

> **Concept.** When the GPU Operator breaks, the fix is a disciplined walk *up* the dependency chain to the first unhealthy stage — read its logs, name the root cause, and record it so the next person is faster.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

04.1 gave you the healthy chain: every operand, its containers and mounts, the three-link label chain that places it, and the barrier files in `/run/nvidia/validations` that order it. That map is necessary but not sufficient — a map you've only ever seen converge cleanly has taught you nothing about failure. This lesson turns it into a **procedure**: an ordered checklist where every step is a command, an expected output, and a rule for what a deviation means. Then it catalogues the seven failure families that procedure resolves to, with the actual log lines each one produces.

Lesson 3 then leaves the Operator's own health behind and moves to the API you consume once the chain is reliably green — the kubelet pod-resources socket that tells you which pod holds which GPU.

Everything below is anchored to **GPU Operator v26.3.3**, **container-toolkit v1.19.1**, **k8s-device-plugin v0.19.3** — the pins in that chart — plus upstream kubelet, containerd and `libnvidia-container` sources. Log strings are quoted from those trees. Where a transcript is assembled rather than captured verbatim, it says so.

## Why this matters

This module's README names the interview probe almost verbatim: *"name the GPU Operator components + debug a crash-looping driver pod."* That is close to a literal transcript of what CoreWeave and NVIDIA platform interviews ask, and it is the page you will actually get at a GPU-heavy shop.

The differentiator between a CKA holder and a senior GPU platform engineer is not knowing *that* things break. It is having a **repeatable order** and a **growing failure-mode log**, so the third occurrence of a driver/kernel mismatch takes ninety seconds instead of an hour of guessing. Guessing has a specific shape and you have probably watched someone do it: restart the workload pod, restart the device plugin, delete the validator, `helm upgrade --reuse-values` for luck, reboot the node. Every one of those actions is plausible, four of them are useless for any given root cause, and one of them — deleting the validator — actively makes the cluster look *worse*, because its `preStop` hook removes every `*-ready` barrier file and sends every other operand back into `Init`.

Put the cost in units. Take an 8×H100 node at an illustrative $2.50/GPU-hour — $20/hr for the node, and rates vary a lot by provider and commitment, so treat that as an order of magnitude:

```
  Undirected diagnosis (guess-and-restart), single node:
      MTTR ≈ 75 min  →  1.25 h × 8 GPUs × $2.50  =  $25 idle

  Directed diagnosis (this lesson's checklist), single node:
      MTTR ≈  6 min  →  0.10 h × 8 GPUs × $2.50  =   $2 idle

  Savings per incident, one node ......................  $23
```

$23 is not the point. The point is what happens when the cause is a bad driver image rolled to a whole tier:

```
  Same failure across a 24-node inference tier:
      undirected  75 min × 24 nodes × 8 GPUs × $2.50/h  =  $600
      directed     6 min × 24 nodes × 8 GPUs × $2.50/h  =   $48

  And the fleet-wide version — someone bumped driver.version cluster-wide
  and every node is now crash-looping, 200 nodes:
      undirected  75 min → 200 × 8 × 1.25 × $2.50 =  $5,000
      directed     6 min → 200 × 8 × 0.10 × $2.50 =    $400
```

A four-thousand-six-hundred-dollar difference produced entirely by knowing which pod to read first. That is the argument for a checklist rather than intuition, and it is also the argument for the failure-mode log: the second occurrence should cost the *lookup* time, not the diagnosis time.

## What's new here (calibration)

02/02b/03 own the underlying theory and 04.1 owns the healthy map. This lesson does **not** re-teach:

- The device-plugin gRPC API or the DRA object model (Module 02) — you know what a healthy plugin *does*; here you learn what it looks like when it doesn't. The registration wire sequence itself is lesson 04.3's material.
- The driver/kernel-module relationship at the silicon level (Module 03) — background you now apply as evidence under time pressure.
- The operand inventory, the label chain, and the barrier-file mechanism (04.1) — prerequisites. If you cannot say which component writes `toolkit-ready`, go back.

What this lesson adds:

- **An ordered, repeatable procedure** — ten steps, each with the command, the expected output, and the interpretation rule for every plausible deviation. Not advice; a runbook you can hand to someone else.
- **A diagnostic decision tree** with the real command at each node.
- **Seven failure families**, each with verbatim-format log lines, the mechanism that produces that specific wording, and the fix.
- **The "disk state lies" principle**, generalised: a layered model of where the authoritative answer lives at each level of the stack, and which command asks the right layer.
- **The failure-mode log format** and the specific metrics and alerts that turn a repeat incident into a lookup.

## Core concepts

### 1. Why the order is invariant

From 04.1: operands are gated by barrier files, and every downstream operand's init container blocks on a file the *upstream* stage's validator writes. Two consequences make the diagnosis order forced rather than stylistic.

**A blocked pod carries no information about the cause.** A device-plugin pod in `Init:0/2` is printing `waiting for nvidia container stack to be setup` every five seconds. That is the same output whether the driver failed to compile, the toolkit failed to write containerd config, or the validator pod was never scheduled. The log is a *statement that the barrier is closed*, not a statement about why.

**The failure is upstream of the symptom, in one direction only.** Barriers are written by upstream stages and read by downstream ones; there is no reverse edge. So the set of possible causes for any symptom is exactly "the stages above it," and walking up is guaranteed to terminate at the first genuinely unhealthy stage.

That gives you a topological ordering to search, and the search is cheap because the chain is only five deep. What makes people slow is not the search — it's starting at the wrong end. **A Pending workload pod is the last place to look, not the first**, because `kubectl describe pod` on it will say `0/1 nodes are available: 1 Insufficient nvidia.com/gpu.` and stop there. That message is true, unhelpful, and identical for six different root causes.

One more structural fact that shapes the whole procedure: **the same symptom appears at two different layers, and they mean different things.** "GPU pods don't work" can be *the resource was never advertised* (a scheduling-layer failure: `nvidia.com/gpu` is 0 or absent, pods sit Pending) or *the resource was advertised but the runtime doesn't deliver a device* (a runtime-layer failure: pods get scheduled, then fail at container-create or run without a GPU). Different halves of the chain, different commands. Step 1 of the procedure exists purely to tell those two apart before you spend any effort.

### 2. The evidence stack — who to ask for the truth

Before the checklist, internalise this. At every level of the GPU stack there is a *record of intent* and a *record of reality*, and they can disagree. Debugging GPU nodes is largely the discipline of always asking the layer that holds reality.

```
  LAYER                     RECORD OF INTENT              RECORD OF REALITY
  ─────                     ────────────────              ─────────────────
                            (what someone asked for)      (what is actually true)

  Cluster config            Helm values file              ClusterPolicy object
                            (a file in git)               kubectl get clusterpolicy -o yaml

  Operand placement         ClusterPolicy .spec           node labels
                            (what should deploy)          kubectl get node --show-labels

  Scheduling resource       node .status.capacity         node .status.allocatable
                            (what the plugin said)        (what the scheduler may use)

  Barrier state             "the pods look Running"       the files themselves
                                                          ls /run/nvidia/validations/

  Runtime wiring            /etc/containerd/config.toml   the LIVE containerd daemon
                            /etc/containerd/conf.d/       crictl info
                              99-nvidia.toml              ctr --address … plugins ls

  Device injection          CDI spec files in /var/run/cdi  nvidia-ctk cdi list
                                                            (parses + validates them)

  Driver presence           the driver POD is Running      /sys/module/nvidia/refcnt
                                                          nvidia-smi in the driver ctr

  Devices in a container    the pod spec's resource limit  ls /dev/nvidia* in the ctr
                                                          nvidia-smi -L in the ctr

  Silicon health            "the GPU is allocated"         dmesg | grep -i xid
                                                          nvidia-smi -q -d PERFORMANCE
```

**Read the right-hand column.** The single most expensive mistake in this area is grepping `/etc/containerd/config.toml`, finding a correct `runtimes.nvidia` block, and concluding the toolkit worked — when containerd was never signalled to reload and is still running with the old config in memory. Config on disk is intent. `crictl info` is reality.

The same asymmetry shows up in a subtler place: `.status.capacity` versus `.status.allocatable`. The device plugin's `ListAndWatch` stream sets capacity. Allocatable is capacity minus reserved minus *unhealthy devices*. A GPU that NVML has flagged unhealthy (the plugin logs `'nvidia.com/gpu' device marked unhealthy: GPU-…` after an Xid critical event) is removed from allocatable while staying in capacity. **Capacity 8, allocatable 7 means one GPU died, not that your maths is wrong** — and the validator's `plugin-validation` step checks *capacity*, so it can pass on a node with a dead GPU.

### 3. The procedure

Ten steps. Run them in order. Each has a command, the output that means "move on," and a rule for interpreting anything else. Steps 0–2 take under a minute and localise the fault to a stage; 3–9 confirm which family it is.

---

#### Step 0 — Scope the incident and anchor the version

```bash
# Which nodes are affected? One, a tier, or everything?
kubectl get nodes -L nvidia.com/gpu.product -L nvidia.com/cuda.driver-version.full \
  -o custom-columns='NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'

# What version am I debugging?
kubectl -n gpu-operator get deploy gpu-operator \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl -n gpu-operator get ds nvidia-driver-daemonset \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

**Expect:** a clear blast radius and two concrete image tags, e.g. `nvcr.io/nvidia/gpu-operator:v26.3.3` and `nvcr.io/nvidia/driver:580.126.20-ubuntu24.04`.

**Interpretation.** *One node affected* → node-local: kernel, hardware, a stale label, a taint. *One tier, all the same GPU model or OS image* → something model- or image-specific: a driver branch that doesn't support that kernel, a MIG config. *Every GPU node at once* → something cluster-scoped changed: a `ClusterPolicy` edit, a `helm upgrade`, an RBAC or NFD failure. The blast radius eliminates two-thirds of the hypothesis space before you read a single log, and it costs one command.

The version anchor matters because behaviour genuinely moved: CDI became the default injection path in the 25.10 line, and the `NVIDIADriver` CRD arrived in 26.3.0. A runbook step that says "check the prestart hook" is wrong on a current install.

---

#### Step 1 — Is this a *scheduling* failure or a *runtime* failure?

```bash
kubectl describe node <node> | sed -n '/^Capacity:/,/^System Info/p'
```

**Expect,** on a healthy 1-GPU node:

```
Capacity:
  cpu:                8
  memory:             32601076Ki
  nvidia.com/gpu:     1
  pods:               110
Allocatable:
  cpu:                7910m
  memory:             31549300Ki
  nvidia.com/gpu:     1
  pods:               110
```

**Interpretation.**

| Observation | What it means | Go to |
|---|---|---|
| `nvidia.com/gpu` **absent entirely** from Capacity | The device plugin never registered with the kubelet on this node — or never ran. | Step 2 |
| `nvidia.com/gpu: 0` in Capacity | The plugin registered but `ListAndWatch` sent an empty device list — usually NVML found nothing, i.e. the driver is broken. | Step 2, then Step 4 |
| Capacity `8`, Allocatable `7` | One device is `Unhealthy`. The chain is fine; a GPU is not. | Step 9 (family F8) |
| Capacity and Allocatable both correct, yet pods fail | **Runtime-layer failure.** The scheduling half works; injection doesn't. | Skip to Step 5 |
| Capacity correct, pods still Pending | Not a GPU problem yet — taints, tolerations, quota, or a resource-name mismatch. | Step 9 |

Notice how much this one command decides. If capacity is right and pods still don't work, you can skip the entire driver/plugin half of the chain.

---

#### Step 2 — Whole-system snapshot: find the first unhealthy stage

```bash
kubectl get pods -n gpu-operator -o wide --field-selector spec.nodeName=<node>
```

**Expect,** on a healthy node: every operand `Running` with the right ready-counts — `nvidia-driver-daemonset` `1/1`, `nvidia-container-toolkit-daemonset` `1/1`, `nvidia-device-plugin-daemonset` `2/2`, `gpu-feature-discovery` `2/2`, `nvidia-dcgm-exporter` `1/1`, `nvidia-operator-validator` `1/1`, plus `nvidia-cuda-validator-…` `Completed`.

**Interpretation.** Scan **top-down in chain order — driver, toolkit, validator, plugin, GFD, DCGM, MIG-manager — and stop at the first one that is not `Running`.** Then classify what you see:

| Pod status | Meaning | Next |
|---|---|---|
| `Init:0/N`, restarts 0, age growing | Waiting on a barrier. **Victim, not cause.** Keep walking up. | Step 3 |
| `0/1 Running` on the **driver** pod, age < 20 min | The `startupProbe` hasn't passed yet. 60 s initial delay + 120 × 10 s = up to 21 min. **Carries no information yet.** | Wait, or Step 4 to see progress |
| `CrashLoopBackOff` | A container is exiting non-zero. Real cause. | Step 4 (driver) / 5 (toolkit) / 7 (plugin) |
| `Error` / `Init:Error` / `Init:CrashLoopBackOff` | An init container is failing. `WITH_WAIT=false` validations fail rather than loop. | Step 3, then the named family |
| `Pending` | The *operand* can't schedule: taint, resources, or a `nodeSelector` that doesn't match. | Step 9 |
| **Pod absent from the list entirely** | Its `nvidia.com/gpu.deploy.*` label is missing or `false`. Label-chain failure, not a pod failure. | Step 9 |
| `ImagePullBackOff` | Registry, credentials, or a `driver.version` tag that doesn't exist. Read `describe`. | — |

Two habits make this step reliable. First, **on a `CrashLoopBackOff` pod always read `--previous`** — the current container may be seconds old and still initialising; the crash you care about is the previous instance:

```bash
kubectl -n gpu-operator logs <pod> -c <container> --previous
```

Second, **for an `Init` pod, get the blocking init container's name from `describe`, not from guesswork:**

```bash
kubectl -n gpu-operator describe pod <pod> | sed -n '/Init Containers/,/^Conditions/p'
```

---

#### Step 3 — Barrier-file census: which gate is closed?

```bash
kubectl -n gpu-operator exec ds/nvidia-operator-validator -- ls -1 /run/nvidia/validations/
```

If the validator pod itself is not up, read the files from a debug pod or the node directly:

```bash
kubectl debug node/<node> -it --image=busybox -- ls -1 /host/run/nvidia/validations/
# or, with node access:
sudo ls -l /run/nvidia/validations/
```

**Expect,** fully converged:

```
.driver-ctr-ready
cuda-ready
driver-ready
plugin-ready
toolkit-ready
workload-type
```

**Interpretation** — this is the fastest single localisation in the whole procedure, because the *first missing file* names the broken stage:

| Highest file present | The broken stage is | Read this |
|---|---|---|
| *(directory empty)* | The driver container hasn't reached a healthy state. | Step 4 |
| `.driver-ctr-ready` only | Driver container is healthy, but `driver-validation` hasn't accepted it — usually it's still looping, occasionally the driver libraries aren't where it expects. | `logs <toolkit-pod> -c driver-validation` |
| `+ driver-ready` | The toolkit stage. `toolkit-ready` is written by the **validator's** `toolkit-validation`, which runs `nvidia-smi` inside its own container — so a missing `toolkit-ready` means the runtime wiring is broken, not that a file wasn't touched. | Steps 5–6 |
| `+ toolkit-ready` | CUDA validation. The `nvidia-cuda-validator` pod isn't completing. | `logs -l app=nvidia-cuda-validator -c cuda-validation` |
| `+ cuda-ready` | The device plugin. `plugin-validation` is polling node capacity and not finding GPUs. | Step 7 |
| all present, node still broken | The barriers are stale — they were opened before something regressed. Nothing re-checks them continuously. | Steps 5, 9 |

That last row is a real trap and worth stating plainly: **barrier files are latches, not live health checks.** Once `toolkit-ready` exists, nothing removes it if containerd is later reconfigured by a competing tool. The only things that clear them are the validator's `preStop` hook (`rm -f /run/nvidia/validations/*-ready`) and the driver container's own `preStop` (`rm -f …/.driver-ctr-ready`). So "all barriers open" means "this node was healthy at some point," not "this node is healthy now" — which is exactly why the procedure ends at Step 10 with a live end-to-end test.

---

#### Step 4 — Driver: are the modules loaded and does `nvidia-smi` work?

```bash
# Reality check, inside the driver container:
kubectl -n gpu-operator exec ds/nvidia-driver-daemonset -c nvidia-driver-ctr -- nvidia-smi -L
kubectl -n gpu-operator exec ds/nvidia-driver-daemonset -c nvidia-driver-ctr -- cat /sys/module/nvidia/refcnt

# If it isn't up, the log — and mind --previous:
kubectl -n gpu-operator logs ds/nvidia-driver-daemonset -c nvidia-driver-ctr --previous | tail -40
```

**Expect:** `GPU 0: NVIDIA L4 (UUID: GPU-3d1e2f4a-…)`, a refcnt that is a small integer, and a log whose last line is `Done, now waiting for signal`.

**Interpretation.** These are the two questions the driver container's own `startupProbe` asks, in the same order — `/sys/module/nvidia/refcnt` exists, then `nvidia-smi` succeeds — so you are reproducing the probe by hand. Failures fall into family **F1** (§4), discriminated by the log line:

| Log evidence | Family | Meaning |
|---|---|---|
| `Could not resolve Linux kernel version` | F1a | The container's apt repos have no `linux-headers-$(uname -r)`. Kernel drift. |
| `error:` in `nv-*.c`, `make: *** [__modpost] Error 2` | F1a | Modules don't compile against this kernel. Wrong driver branch. |
| `Could not unload NVIDIA driver kernel modules, driver is in use` | F1b | Old modules still referenced. Something holds the GPU. |
| `modprobe: ERROR: could not insert 'nvidia': Key was rejected by service` | F1c | Secure Boot rejected an unsigned module. |
| `NVIDIA kernel module not loaded` (from the probe) | F1d | Modules never loaded, or were unloaded under a running container. |
| `Failed to initialize NVML: Driver/library version mismatch` | F1e | Kernel module version ≠ user-space driver version. |

---

#### Step 5 — Toolkit: is the nvidia runtime live in containerd?

**Ask the live daemon first. Only then look at disk.**

```bash
# Reality — the running containerd, from the node:
sudo crictl info | jq '.config.containerd.runtimes | keys'
sudo crictl info | jq '.config.containerd.defaultRuntimeName'

# Intent — the files on disk:
sudo grep -n -A4 'runtimes.nvidia' /etc/containerd/config.toml
sudo cat /etc/containerd/conf.d/99-nvidia.toml 2>/dev/null

# Did the toolkit container think it succeeded?
kubectl -n gpu-operator logs ds/nvidia-container-toolkit-daemonset \
  -c nvidia-container-toolkit-ctr | tail -30
```

**Expect:** `["nvidia","nvidia-cdi","nvidia-legacy","runc"]` from the live daemon, and `"nvidia"` as the default runtime name.

**Interpretation.**

| Live daemon | On disk | Diagnosis |
|---|---|---|
| has `nvidia` | has the block | Toolkit is fine. Go to Step 6. |
| **no** `nvidia` | **has** the block | **Config never took effect.** containerd wasn't reloaded, or was restarted from a different config path. This is family **F2b** and the highest-frequency "everything looks fine but GPU pods fail" cause. |
| no `nvidia` | no block | Toolkit never ran or crashed. Family **F2a** — read its logs and check whether it's stuck on `driver-ready`. |
| has `nvidia`, `BinaryName` points at a path that doesn't exist | — | Install-dir mismatch (`toolkit.installDir` changed, or a partial upgrade). Container-create will fail with `executable file not found`. |

**Two traps here.** First, the CRI plugin key is version-dependent — `cri` at containerd config version 1, `io.containerd.grpc.v1.cri` at 2, `io.containerd.cri.v1.runtime` at 3 and later. Grepping for the v2 key on a v3 config finds nothing and proves nothing. **Grep `runtimes.nvidia`, not the plugin key.** Second, remember the drop-in: `/etc/containerd/conf.d/99-nvidia.toml` (CRI-O: `/etc/crio/crio.conf.d/99-nvidia.conf`). A config that looks untouched at the top level may be entirely correct via the drop-in.

Also worth knowing before you go looking for a restart in the journal: the toolkit's default restart mode is **`signal`** — it sends `SIGHUP` to containerd rather than restarting the unit. `systemctl status containerd` showing no recent restart is *expected*, not evidence of failure.

---

#### Step 6 — RuntimeClass and CDI: can a pod actually ask for a GPU runtime?

```bash
kubectl get runtimeclass
# NAME            HANDLER         AGE
# nvidia          nvidia          2d
# nvidia-cdi      nvidia-cdi      2d
# nvidia-legacy   nvidia-legacy   2d

# CDI specs (the default injection path since the 25.10 line):
sudo ls -l /var/run/cdi/
sudo nvidia-ctk cdi list
```

**Expect:** three RuntimeClasses whose `HANDLER` equals their `NAME`, spec files such as `nvidia.yaml` and `management.yaml` under `/var/run/cdi/`, and a `cdi list` that enumerates `nvidia.com/gpu=0`, `nvidia.com/gpu=GPU-<uuid>`, `nvidia.com/gpu=all`, `management.nvidia.com/gpu=all`.

**Interpretation.**

- **No RuntimeClasses at all** → the operator's `pre-requisites` state never applied. Usually an RBAC problem or the controller is wedged; check `kubectl -n gpu-operator logs deploy/gpu-operator` and `gpu_operator_reconciliation_failed_total`.
- **RuntimeClass exists, handler not registered in the live containerd** → family **F3**. A pod that sets `runtimeClassName: nvidia` (or *any* pod, when `nvidia` is the default runtime) fails to create its sandbox with:

  ```
  failed to get sandbox runtime: no runtime for "nvidia" is configured
  ```

  That exact wording is containerd's — `no runtime for %q is configured` in `internal/cri/config/config.go`, wrapped by `failed to get sandbox runtime: %w`. It is a *containerd* error, not an NVIDIA one, which is why grepping NVIDIA docs for it is unproductive. Because the toolkit sets `nvidia` as containerd's **default** runtime, this failure typically hits *every* pod on the node, GPU or not — a useful discriminator.
- **`nvidia-ctk cdi list` empty or erroring** while specs exist on disk → the specs are malformed or stale (generated against a driver version that's no longer there). Delete and regenerate by restarting the toolkit pod.

---

#### Step 7 — Device plugin: did it register with the kubelet?

```bash
kubectl -n gpu-operator logs ds/nvidia-device-plugin-daemonset -c nvidia-device-plugin | tail -30

# Is the plugin's socket actually there, next to the kubelet's?
sudo ls -l /var/lib/kubelet/device-plugins/
# kubelet.sock  nvidia-gpu.sock  kubelet_internal_checkpoint
```

**Expect** these three lines, which are the healthy registration signature:

```
I... Starting GRPC server for 'nvidia.com/gpu'
I... Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
I... Registered device plugin for 'nvidia.com/gpu' with Kubelet
```

and, on the kubelet side (`journalctl -u kubelet`), the matching:

```
Got registration request from device plugin with resource" resourceName="nvidia.com/gpu"
```

**Interpretation** — family **F4**:

| Log evidence | Meaning | Fix |
|---|---|---|
| `Failed to initialize NVML: ERROR_LIBRARY_NOT_FOUND.` followed by `If this is a GPU node, did you set the docker default runtime to 'nvidia'?` | The plugin can't find `libnvidia-ml.so.1`. The driver, or the injection of it into the plugin pod, is broken. | Walk up: Steps 4–5 |
| `nvml init failed: ERROR_DRIVER_NOT_LOADED` and container exit | Same, with `FAIL_ON_INIT_ERROR=true` (the Operator's default) turning it into a visible crash instead of a silent idle. | Steps 4–5 |
| `No devices found. Waiting indefinitely.` | NVML initialised but enumerated zero GPUs. Plugin stays `Running`, `nvidia.com/gpu` never appears. **The quietest failure in the whole system.** | Step 4; check MIG (Step 8) |
| Errors touching `/var/lib/kubelet/device-plugins/kubelet.sock` | Can't reach the kubelet: hostPath wrong, or the kubelet moved its root dir. | Check the mount; check `--root-dir` |
| `inotify: /var/lib/kubelet/device-plugins/kubelet.sock created, restarting.` | Normal and healthy — the kubelet restarted and the plugin re-registered on its own. Not a bug. | — |
| `invalid MIG configuration: …` and exit | MIG geometry doesn't satisfy the configured strategy. | Step 8 (family F6) |
| `the ResourceName "…" is invalid` in the **kubelet** log | The plugin registered a name that isn't a valid extended resource. Only happens with custom `resources` config. | Fix the plugin config |
| `requested API version "…" is not supported by kubelet. Supported version is ["v1beta1"]` | Version skew between a hand-rolled plugin and the kubelet. Not possible with the NVIDIA plugin, which pins `v1beta1`. | — |

---

#### Step 8 — MIG: does the geometry match the strategy?

Skip this on non-MIG hardware. On A100/H100/H200-class nodes it is the difference between a five-minute fix and an afternoon.

```bash
kubectl get node <node> \
  -L nvidia.com/mig.capable -L nvidia.com/mig.config -L nvidia.com/mig.config.state \
  -L nvidia.com/mig.strategy
kubectl -n gpu-operator logs ds/nvidia-mig-manager -c nvidia-mig-manager | tail -40
kubectl -n gpu-operator exec ds/nvidia-driver-daemonset -c nvidia-driver-ctr -- nvidia-smi -L
```

**Expect:** `mig.capable=true`, a `mig.config` you recognise (e.g. `all-1g.10gb`), `mig.config.state=success`, and `nvidia-smi -L` listing MIG devices under each GPU.

**Interpretation** — family **F6**. The device plugin's own validation is strict and its errors are precise:

| Plugin error | Cause |
|---|---|
| `invalid MIG configuration: at least one device with migEnabled=true was not configured correctly: device 0 has no MIG devices configured` | MIG mode is *on* for a GPU but no GPU instances were created. Half-applied config. |
| `invalid MIG configuration: … more than one MIG device type present on node` | `mig.strategy=single` but the GPUs have heterogeneous profiles. Use `mixed`, or make them uniform. |
| `all devices on the node must be configured with the same migEnabled value` | Under `single`, some GPUs are MIG-enabled and some are not. Not allowed. |

And `nvidia.com/mig.config.state` tells you where the MIG manager got to. Its `reconfigure-mig.sh` walks: read the current state → assert the desired config → set the label to `pending` → tear down GPU clients by flipping their `nvidia.com/gpu.deploy.*` labels to `paused-for-mig-change` → wait for the plugin/GFD/DCGM pods to be deleted (`kubectl wait --for=delete … --timeout=5m`) → apply the geometry with `nvidia-mig-parted` → restore the labels → set the state to `success`, or to `failed`. Two states are diagnostic:

- **Stuck at `pending`** → the teardown didn't finish. Usually a GPU pod won't terminate, so the `kubectl wait` timed out. **This is why you drain a node before reconfiguring MIG**: enabling or changing MIG mode requires a GPU reset, and a reset requires no processes holding the device.
- **`rebooting`** → a MIG *mode* change (not just geometry) needed a host reboot to take effect. If the script then finds the mode still unapplied it logs `MIG mode change did not take effect after rebooting` and goes to `failed`.

A subtle trap worth naming: under MIG, the resource name changes. With `mig.strategy=mixed` the node advertises `nvidia.com/mig-1g.10gb`, **not** `nvidia.com/gpu`. A pod requesting `nvidia.com/gpu: 1` on such a node sits Pending forever with a perfectly healthy chain — which is Step 9's territory.

---

#### Step 9 — Pod level: Pending on an apparently-healthy node

You get here when Steps 1–8 are all green and pods still don't run. Six causes, all cheap to check.

```bash
kubectl describe pod <workload-pod> | tail -25
kubectl get pod <workload-pod> -o jsonpath='{.status.reason}: {.status.message}{"\n"}'
kubectl describe node <node> | grep -i -A3 taint
kubectl get node <node> -o jsonpath='{.status.allocatable}{"\n"}' | jq .
kubectl get resourcequota -A | grep -i nvidia
```

| Symptom | Cause | Mechanism |
|---|---|---|
| `0/1 nodes are available: 1 Insufficient nvidia.com/gpu.` and allocatable is correct and non-zero | Every GPU is **already allocated** to other pods. Not a failure. | Extended resources are exclusive: one device, one container, no oversubscription without MIG/time-slicing/MPS. |
| Same message, and the node advertises `nvidia.com/mig-1g.10gb` instead | **Resource-name mismatch** under `mig.strategy=mixed`. | The scheduler is matching a literal resource key. `nvidia.com/gpu` genuinely does not exist on that node. |
| `1 node(s) had untolerated taint {nvidia.com/gpu: }` | The node is tainted and the *workload* doesn't tolerate it — even though every operand does (they all carry the `nvidia.com/gpu Exists NoSchedule` toleration). | A GPU taint you or your cloud added is deliberately keeping non-GPU work off; your GPU work must opt in. |
| Pod rejected at admission: `must be an integer` | A fractional request like `nvidia.com/gpu: 0.5`. | `IsIntegerResourceName()` returns true for every extended resource, so `ValidateResourceQuantityValue` rejects any quantity whose milli-value isn't a multiple of 1000. You **cannot** request half a GPU. Sharing works by advertising *more integer devices* (MIG slices, time-sliced replicas), never by fractional requests. |
| `status.reason: UnexpectedAdmissionError` with a device-manager message | The kubelet accepted the pod, then `Allocate` failed. Messages come from the device manager: `requested number of devices unavailable for nvidia.com/gpu. Requested: 2, Available: 1`; `no healthy devices present; cannot allocate unhealthy devices nvidia.com/gpu`; `previously allocated devices are no longer healthy…`; `cannot allocate unregistered device nvidia.com/gpu`. Under time-slicing with `failRequestsGreaterThanOne=true`, also `maximum request size for shared resources is 1; found 2`. | A scheduling/kubelet race or a device that went unhealthy between scheduling and admission. **Such pods do not retry** — they must be deleted and recreated. |
| Pod `Running`, but `nvidia-smi` inside says `command not found`, or `ls /dev/nvidia*` is empty | **Runtime-layer failure.** The resource was granted; injection didn't happen. | Back to Steps 5–6. This is the case where scheduling metrics look perfect and the workload is useless. |

---

#### Step 10 — Confirm the fix at the layer that matters

"The pod is Running" is not a fix. Three tiers, and **only tier 3 counts as done**:

```bash
# Tier 1 — resource restored
kubectl get node <node> -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{"\n"}'   # -> 1

# Tier 2 — driver sees the silicon
kubectl -n gpu-operator exec ds/nvidia-driver-daemonset -c nvidia-driver-ctr -- nvidia-smi -L

# Tier 3 — a real container actually got a device and ran a kernel
kubectl -n gpu-operator logs -l app=nvidia-cuda-validator -c cuda-validation
# [Vector addition of 50000 elements]
# Test PASSED
```

Tiers 1 and 2 can both pass while the runtime is still mis-wired — that is precisely family F2b. Make tier 3 the acceptance bar for every recovery, and note the container name: the cuda-validator's *main* container only echoes `cuda workload validation is successful`; the `vectorAdd` proof is in the `cuda-validation` init container.

---

### 4. The decision tree

The procedure above, compressed to the shape you can hold in your head at 3 a.m.

```
                    ┌──────────────────────────────────────────┐
                    │  SYMPTOM: "GPU pods don't work"          │
                    └───────────────────┬──────────────────────┘
                                        │
              kubectl describe node <node> | sed -n '/^Capacity:/,/^System Info/p'
                                        │
        ┌───────────────────────────────┼────────────────────────────────┐
        │                               │                                │
  nvidia.com/gpu                  nvidia.com/gpu                   capacity OK
  ABSENT or 0                     capacity 8 / alloc 7             but pods fail
        │                               │                                │
        ▼                               ▼                                ▼
 ┌──────────────┐              ┌─────────────────┐          ┌──────────────────────┐
 │ SCHEDULING   │              │ device unhealthy │          │ RUNTIME-LAYER        │
 │ LAYER broken │              │ (F8)             │          │ (F2b / F3)           │
 └──────┬───────┘              │ logs plugin:     │          │                      │
        │                      │ "'nvidia.com/gpu'│          │ crictl info | jq     │
        │                      │  device marked   │          │  '.config.containerd │
        │                      │  unhealthy: GPU-"│          │   .runtimes|keys'    │
        │                      │ dmesg | grep xid │          └──────┬───────────────┘
        │                      └─────────────────┘                  │
        │                                                  ┌────────┴─────────┐
        │                                            has "nvidia"?      no "nvidia"?
        │                                                  │                 │
        │                                                  ▼                 ▼
        │                                        ┌──────────────┐  ┌──────────────────┐
        │                                        │ Step 6:      │  │ grep runtimes.   │
        │                                        │ RuntimeClass │  │ nvidia in        │
        │                                        │ + cdi list   │  │ config.toml AND  │
        │                                        └──────────────┘  │ conf.d/99-nvidia │
        │                                                          └────┬─────────────┘
        │                                                        ┌──────┴───────┐
        │                                                    present?       absent?
        │                                                        │              │
        │                                                        ▼              ▼
        │                                                 ══ F2b ══      ══ F2a ══
        │                                                 containerd     toolkit never
        │                                                 not reloaded.  ran / crashed.
        │                                                 delete the     logs -c
        │                                                 toolkit pod    nvidia-container
        │                                                 → re-signal    -toolkit-ctr
        │
        ▼
  kubectl get pods -n gpu-operator -o wide --field-selector spec.nodeName=<node>
  ── scan TOP-DOWN in chain order, stop at first not-Running ──
        │
        ├── pod ABSENT ──────────▶ label-chain failure.  Check, in order:
        │                          feature.node.kubernetes.io/pci-10de.present
        │                          nvidia.com/gpu.present
        │                          nvidia.com/gpu.deploy.operands  (≠ false)
        │                          nvidia.com/gpu.deploy.<operand> (≠ false)
        │                          then taints vs the DS tolerations
        │
        ├── pod Pending ─────────▶ operand can't schedule: taint / resources
        │
        ├── Init:0/N, 0 restarts ▶ VICTIM. exec validator:
        │                          ls -1 /run/nvidia/validations/
        │                          ── first MISSING file names the broken stage ──
        │                            (empty)      → driver container      → F1
        │                            .driver-ctr- → driver-validation     → F1
        │                              ready only
        │                            + driver-    → toolkit / runtime     → F2/F3
        │                              ready
        │                            + toolkit-   → cuda validation       → F1/F2
        │                              ready
        │                            + cuda-ready → device plugin         → F4/F6
        │
        └── CrashLoopBackOff ────▶ logs <pod> -c <ctr> --previous | tail -40
                                   ── discriminate by the log line ──
             ┌───────────────────────────┬──────────────────────────────┐
             │ DRIVER pod                │ PLUGIN pod                    │
             ├───────────────────────────┼──────────────────────────────┤
             │ "Could not resolve Linux  │ "Failed to initialize NVML:   │
             │  kernel version"          │  ERROR_LIBRARY_NOT_FOUND"     │
             │  → F1a kernel drift       │  → driver/injection broken,   │
             │                           │    walk UP to F1/F2           │
             │ "make: *** [__modpost]    │                               │
             │  Error 2"                 │ "No devices found. Waiting    │
             │  → F1a wrong branch       │  indefinitely."               │
             │                           │  → NVML ok, 0 GPUs → F1/F6    │
             │ "Could not unload NVIDIA  │                               │
             │  driver kernel modules,   │ "invalid MIG configuration:"  │
             │  driver is in use"        │  → F6 geometry/strategy       │
             │  → F1b module busy        │                               │
             │                           │ kubelet.sock errors           │
             │ "Key was rejected by      │  → F4 socket/mount            │
             │  service"                 │                               │
             │  → F1c Secure Boot        │                               │
             │                           │                               │
             │ "Driver/library version   │                               │
             │  mismatch"                │                               │
             │  → F1e partial upgrade    │                               │
             └───────────────────────────┴──────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────────────────┐
                    │  FIX, then Tier-3 confirm:               │
                    │  logs -l app=nvidia-cuda-validator       │
                    │       -c cuda-validation → "Test PASSED" │
                    │  then write the failure-mode-log entry   │
                    └──────────────────────────────────────────┘
```

### 5. Failure family reference

The procedure localises; this section explains. For each family: what breaks, the log evidence with the wording you'll actually see, *why* the wording looks like that, and the fix.

#### F1 — Driver not loaded

The driver DaemonSet builds kernel modules against the host's running kernel and loads them into it. Five sub-shapes.

**F1a — kernel/headers or precompiled-driver mismatch.** The most common driver failure by a wide margin. The container's entrypoint calls `apt-cache show linux-headers-$(uname -r)`; if the repos don't have that kernel's headers it prints to stderr and returns 1:

```
Resolving Linux kernel version...
Could not resolve Linux kernel version
```

Or, if headers exist but the driver sources don't compile against that kernel's internal API:

```
Compiling NVIDIA driver kernel modules...
/usr/src/nvidia-580.126.20/nvidia/nv-mmap.c:1234:5: error: too many arguments to function 'vm_insert_page'
make[1]: *** [scripts/Makefile.build:244: /usr/src/nvidia-580.126.20/nvidia/nv-mmap.o] Error 1
make: *** [Makefile:1234: __modpost] Error 2
```

*Why this wording:* you're reading `make` and `gcc`, not NVIDIA tooling. Compiler errors mean "sources vs kernel API"; `Could not resolve Linux kernel version` means "package repo vs running kernel." Those are different fixes.

*Fix:* pin `driver.version` to a branch that supports the node's kernel; or match the node's kernel to what the image supports; or use `driver.usePrecompiled=true` **with an image built for that exact kernel** — precompiled images are stricter, not looser.

**F1b — stale or busy module.** The `k8s-driver-manager` init container's job is to unload the old modules before the new ones load. If a process still holds the GPU, `rmmod` can't proceed. The driver container's `_unload_driver()` reads the refcnt files and refuses:

```
lsmod | grep nvidia
nvidia_uvm           1519616  2
nvidia               56807424  43
Could not unload NVIDIA driver kernel modules, driver is in use
```

and the driver manager escalates:

```
Unable to cleanup driver modules, attempting again with node drain...
```

*Why this wording:* the check is arithmetic on `/sys/module/*/refcnt` — if `nvidia`'s refcount exceeds the number of its known dependent modules, something *else* is holding it. That's why the code dumps `lsmod` first: the extra reference is the evidence.

The same family covers `nouveau` not being blacklisted — the open-source driver claims the device and `modprobe nvidia` can't have it.

*Fix:* find and evict the holder. `driver.upgradePolicy.gpuPodDeletion` (force / timeout 300 s / deleteEmptyDir) and `driver.upgradePolicy.drain` (disabled by default, timeout 300 s) exist exactly for this. On-prem, also check for host-side holders: `nvidia-persistenced`, DCGM running outside Kubernetes, a stray `nvidia-smi -l`.

**F1c — Secure Boot rejects an unsigned module.**

```
modprobe: ERROR: could not insert 'nvidia': Key was rejected by service
```

*Why this wording:* it's the kernel's `-EKEYREJECTED`, surfaced by `modprobe`. Nothing NVIDIA-specific.

*Fix:* enrol the module signing key (MOK), use a distro-signed driver package, or disable Secure Boot. On-prem gotcha; rare on cloud images.

**F1d — modules never loaded / unloaded underneath.** The `startupProbe` prints its own diagnosis:

```
NVIDIA kernel module not loaded
```

or

```
nvidia-smi failed
```

*Fix:* read the container log above the probe output to find which of F1a–F1c actually happened. The probe reports the *symptom*; the entrypoint log has the cause.

**F1e — kernel/user-space version skew.**

```
Failed to initialize NVML: Driver/library version mismatch
NVML library version: 580.126
```

*Why this wording:* `nvidia.ko` and `libnvidia-ml.so`/`libcuda.so` must be the *same* version. This appears when user-space is replaced while the old module is still resident — classically an `apt upgrade` on the host, or a host driver colliding with the operator's containerized one. Nothing works until the modules are reloaded, which needs every GPU process gone, which on a Kubernetes node means cordon and drain (or reboot).

*Fix:* pick one owner for the driver. Either `driver.enabled=true` and exclude NVIDIA packages from host unattended-upgrades, or `driver.enabled=false` and let the host own it. Never both.

#### F2 — Container toolkit misconfigured

**F2a — the toolkit pod itself fails.** Either it's stuck in `Init` on `driver-validation` (which is really F1 wearing a different hat), or its main container is looping. Note that its main container *also* waits on `driver-ready` in shell, so a toolkit pod whose log is nothing but

```
waiting for the driver validations to be ready...
waiting for the driver validations to be ready...
```

is a driver problem, not a toolkit problem. Real toolkit failures show `nvidia-ctk`/`nvidia-toolkit` errors: unable to parse the existing runtime config (someone hand-edited invalid TOML), unable to write to the config path, or unable to signal containerd.

Consequence: `toolkit-ready` is never written, so the device plugin, GFD, DCGM-exporter and MIG-manager all sit in `Init`.

**F2b — the toolkit "succeeded" but the wiring isn't live.** The dangerous one, because every dashboard is green. The config block is on disk; the running containerd doesn't have it. GPU pods then fail at container create:

```
Error: failed to create containerd task: failed to create shim task: OCI runtime
create failed: runc create failed: unable to start container process: error during
container init: error running hook #0: error running hook: exit status 1, stdout: ,
stderr: Auto-detected mode as 'legacy'
nvidia-container-cli: initialization error: nvml error: driver not loaded: unknown
```

*(Representative transcript. The `initialization error: %s` prefix is `libnvidia-container`'s CLI; the suffix after `nvml error:` is NVML's own `nvmlErrorString()` for the returned code — here `NVML_ERROR_DRIVER_NOT_LOADED (9)`. Exact wording varies with toolkit and runc versions.)*

Or, when containerd fell back to plain `runc` because it has no `nvidia` runtime, the far quieter version: the pod runs, and inside it `nvidia-smi` is `command not found` and `/dev/nvidia*` doesn't exist. No error anywhere. The workload just can't see a GPU.

Or, if `BinaryName` points at a path that isn't there:

```
Error: failed to create containerd task: ... exec: "nvidia-container-runtime-hook":
executable file not found in $PATH
```

*Why "disk state lies" is the rule here:* the toolkit's job has two halves — write the file, and make the daemon reload it. The first half almost never fails; the second one does, because it depends on signalling a process the toolkit doesn't own. **`crictl info` is the only evidence that counts.**

*Fix:* delete the toolkit pod so it re-runs the install and re-signals containerd, then re-confirm with `crictl info`. If it recurs, look for a competing config manager — a cloud-init snippet, a Chef/Ansible run, or a node image build step that rewrites `/etc/containerd/config.toml` on boot.

#### F3 — RuntimeClass missing

```
failed to get sandbox runtime: no runtime for "nvidia" is configured
```

Two distinguishable situations produce this string, and they need different fixes:

- **The `nvidia` handler isn't in containerd's runtime table** (F2b in disguise). Fix the toolkit.
- **The RuntimeClass object was deleted** while pods still reference it by name. Fix: let the operator recreate it, or check the controller's logs for why `pre-requisites` isn't applying.

Because `CONTAINERD_SET_AS_DEFAULT=true` makes `nvidia` the default runtime, this frequently breaks *every* pod on the node, including ones with no GPU request. **A node where all pods fail sandbox creation is a strong signal for F3/F2b, not a GPU-specific problem** — and that's a useful thing to recognise, because it looks like a much scarier node-level outage than it is.

#### F4 — Device plugin not registered

Split it into three, because they need different fixes.

- **Gated.** The plugin pod is `Init:0/2` on `toolkit-validation`. Not a plugin problem. Walk up.
- **Not scheduled.** The pod isn't in the list at all. Label-chain failure: `nvidia.com/gpu.deploy.device-plugin` missing or `false`, the NFD label gone, `gpu.deploy.operands=false`, or a taint the DaemonSet doesn't tolerate (it tolerates `nvidia.com/gpu Exists NoSchedule` and nothing else by default).
- **Running but not registering.** The pod is up and `nvidia.com/gpu` is still absent or 0. Read the three-line registration signature from Step 7. If NVML failed you'll see the plugin's distinctive multi-line hint block:

  ```
  E... Failed to initialize NVML: ERROR_LIBRARY_NOT_FOUND.
  E... If this is a GPU node, did you set the docker default runtime to `nvidia`?
  E... You can check the prerequisites at: https://github.com/NVIDIA/k8s-device-plugin#prerequisites
  E... You can learn how to set the runtime at: https://github.com/NVIDIA/k8s-device-plugin#quick-start
  E... If this is not a GPU node, you should set up a toleration or nodeSelector to only deploy this plugin on GPU nodes
  ```

  Those five lines are hard-coded in the plugin's `factory.go` and are a reliable fingerprint: **NVML could not be loaded**, which means the driver isn't visible *inside the plugin pod*. With `FAIL_ON_INIT_ERROR=true` (the Operator's default) it then exits with `nvml init failed: <code>` and crash-loops; with `false` it would warn and idle.

If NVML loaded but found nothing, you get the single quietest line in the system:

```
I... No devices found. Waiting indefinitely.
```

The pod is `Running`, `Ready`, 0 restarts, and the node has no GPU resource. Nothing is red. Check the driver (Step 4) and MIG (Step 8).

One healthy-looking line worth *not* chasing:

```
I... inotify: /var/lib/kubelet/device-plugins/kubelet.sock created, restarting.
```

The plugin watches that directory precisely so it can re-register after a kubelet restart. Seeing it means the mechanism worked.

#### F5 — Capacity not advertised, or advertised wrong

This is F1/F4's downstream effect, but two variants are worth separating because their fixes are elsewhere.

**Capacity present, allocatable lower.** One or more devices are `Unhealthy`. The plugin marks a device unhealthy on an NVML **Xid critical error** event and logs:

```
I... XidCriticalError: Xid=79 on Device=GPU-3d1e2f4a-...; marking device as unhealthy.
I... 'nvidia.com/gpu' device marked unhealthy: GPU-3d1e2f4a-...
```

The plugin then re-sends its `ListAndWatch` stream with that device's health flipped. Two facts to carry: **there is no recovery path from `Unhealthy` in the plugin** (the source says so in a `FIXME`) — the pod must be restarted or the node fixed; and health-checking is tunable via `DP_DISABLE_HEALTHCHECKS` (set it to `all` or a comma-separated Xid list to ignore) and `DP_ENABLE_HEALTHCHECKS`, which is how teams suppress noisy application-level Xids. On startup the plugin tells you what it's ignoring: `Ignoring the following XIDs for health checks: …`.

Correlate with the kernel's own channel:

```bash
sudo dmesg -T | grep -i xid
# NVRM: Xid (PCI:0000:1b:00): 79, GPU has fallen off the bus.
```

Xid 79 (fallen off the bus) and 48 (double-bit ECC) are hardware — drain the node. Xid 13/31/43 are usually the *workload's* illegal memory access, not a dying card. Module 03 has the full table; the point here is that a capacity/allocatable gap is a hardware signal, and hardware signals don't get fixed by restarting operands.

**Capacity present under the wrong name.** Under `mig.strategy=mixed` the node advertises `nvidia.com/mig-1g.10gb` and friends. Every operand is green; pods requesting `nvidia.com/gpu` are Pending forever. Under time-slicing with `renameByDefault=true` the resource becomes `nvidia.com/gpu.shared`. **Always read the actual key, not the count** — `kubectl get node <node> -o jsonpath='{.status.allocatable}' | jq .` and look at the names.

#### F6 — MIG mode mismatch

Covered in Step 8. The essential mechanism to be able to explain: enabling or changing MIG *mode* requires a GPU reset, a GPU reset requires no processes attached to the device, and in Kubernetes "no processes attached" means every GPU pod and every GPU-touching operand off the node. That is why the MIG manager tears down the device plugin, GFD, DCGM and DCGM-exporter by flipping their deploy labels and then `kubectl wait --for=delete`s their pods, and why `nvidia.com/mig.config.state` stuck at `pending` is nearly always "something wouldn't terminate."

#### F7 — Pods Pending on an apparently-healthy node

Covered in Step 9. The one mechanism to internalise, because it is the most-asked interview question in this module: **you cannot request a fractional GPU.** `nvidia.com/gpu` is an *extended resource* — a resource name with a `/` that is not in a `*.kubernetes.io/` namespace. `IsIntegerResourceName()` returns true for every such name, and `ValidateResourceQuantityValue()` then rejects any quantity whose `MilliValue() % 1000 != 0` with `must be an integer`. The reason behind the validation is the device-plugin contract itself: `ListAndWatch` advertises a *set of discrete device IDs*, and `Allocate` hands specific IDs to a container. There is no wire representation for "40% of `GPU-a1b2…`". Sharing therefore works by **advertising more integer devices** — MIG exposes each slice as its own device ID, time-slicing advertises N replicas of one physical GPU as `GPU-<uuid>::0` … `::N-1` — so the request is still `1`, and the fraction lives behind the plugin. Lesson 04.7 is about what that costs you in attribution.

#### F8 — A running container loses its GPU

The failure that isn't in the chain at all, and the one that will confuse you most the first time. A container that has been happily training for hours suddenly reports:

```
Failed to initialize NVML: Unknown Error
```

Nothing in Kubernetes changed. The node is fine. Other containers are fine.

*Mechanism:* recent `runc` requires symlinks under `/dev/char` for any device node injected into a container. The NVIDIA driver doesn't create them. When systemd manages the container's cgroups and something triggers a Unit reload — `systemctl daemon-reload` is enough — the device cgroup is recomputed, and without the `/dev/char` symlinks the injected device nodes are dropped from the container's allowed list. The container keeps running with a device it can no longer open.

This is tracked as *"NOTICE: Containers losing access to GPUs with error: `Failed to initialize NVML: Unknown Error`"* — gpu-operator #485 and nvidia-container-toolkit #48. The GPU Operator's mitigation is built into the validator: its `driver-validation` step calls `createDevCharSymlinks()`, and if that fails it emits a multi-line error pointing at gpu-operator #430, explaining that the symlinks are required for runtimes with systemd cgroup management and telling you the escape hatch:

```
validator:
  driver:
    env:
    - name: DISABLE_DEV_CHAR_SYMLINK_CREATION
      value: "true"
```

*Fix / prevention:* on an Operator-managed cluster this is handled for you — which is a reason to check that `driver-validation` actually succeeded rather than assuming. Outside Kubernetes: `sudo nvidia-ctk system create-dev-char-symlinks --create-all`. Alternatively switch the container runtime off systemd cgroup management (`"exec-opts": ["native.cgroupdriver=cgroupfs"]` for Docker), though that has its own consequences.

### 6. The failure-mode log

Five fields, per incident. This is a deliverable, not a nice-to-have — lesson 04.10 assembles these into `failure-mode-log.md`, and the `PREVENTION` field is literal input to your cost operator's alerting.

```
SYMPTOM:     GPU pods Pending on 6 of 24 nodes; nvidia.com/gpu capacity absent
             on exactly those 6. Started 14:02, ~20 min after a node-pool image
             refresh. Blast radius = one node pool.

EVIDENCE:    Step 2: nvidia-driver-daemonset CrashLoopBackOff on all 6.
             Step 4: logs --previous | tail:
                 Resolving Linux kernel version...
                 Could not resolve Linux kernel version
             Step 3: /run/nvidia/validations/ empty on all 6.
             uname -r on a good node: 6.8.0-51-generic
             uname -r on a bad node:  6.11.0-9-generic

ROOT CAUSE:  Family F1a. The node-pool image refresh moved the kernel to 6.11;
             the driver container image for 580.126.20 has no matching
             linux-headers package in its apt repos, so header resolution
             failed before any compile was attempted.

FIX:         Pinned the node pool back to the previous image (immediate, 8 min),
             then moved to driver.usePrecompiled=false with a driver branch
             whose repos carry 6.11 headers. Verified at Tier 3:
             logs -l app=nvidia-cuda-validator -c cuda-validation → Test PASSED
             on all 6.

PREVENTION:  1. Alert: gpu_operator_node_driver_ready == 0 for > 15m, by node.
                (15m, not 5m — the driver startupProbe budget is 21m, so a
                 shorter window fires on every legitimate driver restart.)
             2. Alert: sum by (node) (kube_node_status_allocatable{
                       resource="nvidia_com_gpu"}) == 0 for > 15m
             3. CI gate: before any node-image or driver bump, run one canary
                node and require Tier-3 pass before the pool rolls.
             4. Record the (kernel, driver branch) pair that works in the
                fleet inventory; nvidia.com/cuda.driver-version.full and
                feature.node.kubernetes.io/kernel-version.full are both node
                labels, so this is a kubectl get nodes -L away.
```

Note what the `PREVENTION` field is doing: converting a one-off diagnosis into a *detector*. Three signals are cheap and specific enough to alert on, and all three come from things 04.1 already introduced:

| Signal | Source | Alert on |
|---|---|---|
| `gpu_operator_node_driver_ready` / `_toolkit_ready` / `_cuda_ready` / `_plugin_ready` | `nodeStatusExporter` (set `nodeStatusExporter.enabled=true`) — it watches the barrier files and exports each as a gauge | `== 0 for > 15m`, by node. This is *literally* "which barrier is closed on which node," fleet-wide. |
| `gpu_operator_reconciliation_status`, `gpu_operator_reconciliation_failed_total`, `gpu_operator_gpu_nodes_total` | the operator Deployment | reconcile failures rising; `gpu_nodes_total` dropping (NFD regression) |
| `kube_node_status_allocatable{resource="nvidia_com_gpu"}` and `..._capacity` | kube-state-metrics | allocatable `== 0`; and **allocatable `<` capacity**, which is the F8/F5 unhealthy-device detector and is otherwise nearly invisible |

Enabling `nodeStatusExporter` is the single highest-leverage thing in this lesson, and it is off by default. Four gauges per node turn the whole barrier chain into a dashboard.

### 7. Checking whether someone already hit it

Before an hour of first-principles work on an unfamiliar log line, spend two minutes on the [gpu-operator issue tracker](https://github.com/NVIDIA/gpu-operator/issues), filtered to your operator version and Kubernetes version. This is not a shortcut around understanding — you still have to know why the fix works before you apply it to production — but the base rate is genuinely high. `Could not resolve Linux kernel version` alone has issues filed against CentOS 7.9 + kernel 5.4 (#205), GKE 1.25 + operator 23.3.1 (#526), EKS 1.30 + Ubuntu 22.04 (#1220), and EKS-optimized Amazon Linux AMIs (#666). Four different platforms, one mechanism, four years apart.

The general shape of that mechanism is worth stating because it generalises past NVIDIA: **a vendor-driven control-plane or image upgrade moves the kernel out from under a driver image that was pinned for the old one.** You didn't change anything; your provider did. That is why the "blast radius" question in Step 0 is first, and why the failure-mode log's `PREVENTION` field for kernel-drift incidents is always some form of "canary the image, not just the driver."

## Perspectives

**On-call, under time pressure.** "Walk up the chain, never start at the Pending pod" is a triage protocol, not a preference — it's what lets you localise a fault before you're fully awake. The three commands worth muscle memory: `describe node | sed -n '/^Capacity:/,/^System Info/p'`, `get pods -n gpu-operator -o wide --field-selector spec.nodeName=…`, and `exec ds/nvidia-operator-validator -- ls -1 /run/nvidia/validations/`. Sixty seconds, and you have a stage.

**Kernel / systems.** Family F1 is Linux wearing a Kubernetes costume: `rmmod` returning `EBUSY`, DKMS-style builds against `/lib/modules/$(uname -r)`, Secure Boot MOK enrolment, `/sys/module/nvidia/refcnt`. Nothing about it is Kubernetes-specific — it is exactly what you'd hit running `nvidia-smi` on bare metal after a kernel upgrade, surfaced through a DaemonSet's exit code instead of a shell prompt. If you can debug it on bare metal you can debug it here; the only new skill is knowing which container to be inside.

**Fleet-scale / statistics.** One node hitting a driver/kernel mismatch is an inconvenience. The same failure recurring across 200 nodes after a coordinated bump is why the failure-mode log is a structured artifact rather than tribal memory — the difference between one engineer relearning a fix five times and a team learning it once. It's also why `maxParallelUpgrades: 1` is a *safety* default despite being slow (04.1 §11): a narrow wave means a bad driver takes out one node before you notice, not the fleet.

**Cross-cloud portability.** #1220's EKS incident is not evidence that AWS is uniquely fragile. It's evidence that this failure family recurs, differently shaped, on every managed Kubernetes whenever the control-plane version moves the node image out from under a driver/kernel pairing tuned for the old one. The taxonomy here is vendor-agnostic because the underlying mechanisms — kernel module vs running kernel, config on disk vs live daemon state, device cgroup vs injected device node — are vendor-agnostic.

**Economics.** The MTTR arithmetic in *Why this matters* is the whole business case. Directed diagnosis on a 200-node fleet-wide driver failure is a ~$400 incident; undirected is a ~$5,000 one. And the second occurrence should cost neither, because the failure-mode log turns it into a lookup — which is why the log is a deliverable and not a habit.

## Real-world use cases

- **[NVIDIA/gpu-operator #1220 — "gpu-operator breaks when upgrading EKS to K8s v1.30".](https://github.com/NVIDIA/gpu-operator/issues/1220)** A fleet running fine on Kubernetes 1.29 broke on the routine 1.30 upgrade. The Ubuntu 22.04 AMIs for 1.30 stopped shipping kernel 5.15 and moved to 6.5/6.8; the driver container could not resolve headers for the running kernel and produced `Could not resolve Linux kernel version`, while the runtime side reported `failed to get sandbox runtime: no runtime for 'nvidia' is configured`. *What it shows:* **one vendor-driven event can trip two families at once** (F1a and F2b/F3), which is exactly why you walk the whole chain instead of stopping at the first plausible error. *Verification note:* title and substance confirmed via search this session; `github.com` HTML is not fetchable through this environment's egress proxy, so the comment thread was not re-read. Both log strings were verified independently against primary source — `Could not resolve Linux kernel version` in `NVIDIA/gpu-driver-container`'s `nvidia-driver` script (`_resolve_kernel_version()`), and `no runtime for %q is configured` in containerd's `internal/cri/config/config.go`.
- **[gpu-operator #485](https://github.com/NVIDIA/gpu-operator/issues/485) / [nvidia-container-toolkit #48](https://github.com/NVIDIA/nvidia-container-toolkit/issues/48) — "NOTICE: Containers losing access to GPUs with error: `Failed to initialize NVML: Unknown Error`".** *What it shows:* a *running* container can lose its GPU with nothing in Kubernetes changing, because `runc` needs `/dev/char` symlinks for injected device nodes and a systemd cgroup reload recomputes the device cgroup without them. `systemctl daemon-reload` is a sufficient trigger. This is family F8, and it's the one failure in this lesson that the chain-walking procedure will *not* find, because the chain is healthy. *Corroboration:* the GPU Operator's validator source cites the companion issue [#430](https://github.com/NVIDIA/gpu-operator/issues/430) directly in its own error message and ships `createDevCharSymlinks()` plus a `DISABLE_DEV_CHAR_SYMLINK_CREATION` escape hatch — i.e. the product itself is primary-source evidence for the bug and the fix.
- **The `Could not resolve Linux kernel version` cluster of issues: [#205](https://github.com/NVIDIA/gpu-operator/issues/205) (CentOS 7.9, kernel 5.4), [#526](https://github.com/NVIDIA/gpu-operator/issues/526) (GKE 1.25, operator 23.3.1), [#666](https://github.com/NVIDIA/gpu-operator/issues/666) (EKS-optimized Amazon Linux AMIs), [#1220](https://github.com/NVIDIA/gpu-operator/issues/1220) (EKS 1.30, Ubuntu 22.04).** *What it shows:* **the same three words across four platforms and four years.** Kernel drift is not a bug in any one distro's integration; it is the structural consequence of shipping kernel modules in a container image. Treat it as an expected operating condition with a canary process, not as an incident to be surprised by. *Verified this session via search result titles; individual threads not fetched.*
- **The version-pin table in [`NVIDIA/gpu-operator`'s chart](https://github.com/NVIDIA/gpu-operator/blob/main/deployments/gpu-operator/values.yaml) as an incident tool.** Not an incident, a technique: when you're handed a broken cluster, `kubectl -n gpu-operator get ds -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image'` against the chart's pins for that operator version tells you instantly whether someone has overridden a component to a version the rest of the stack wasn't tested with. Mixed pins are a real and under-checked cause of "this cluster is weird."
- **An honest gap.** No tier-1 engineering-blog postmortem specifically narrating a GPU-Operator crash-loop incident (a full Meta/Netflix/Discord-style timeline) could be found or confirmed this session. The filed issues above are the strongest available evidence, and for this purpose they're arguably better: primary, dated, version-pinned, with the exact log strings. Don't assume a polished blog post exists behind these claims.

## Worked example

Two traces. The first is a loud failure with an obvious pod to read; the second is the quiet one that costs people afternoons.

### Trace 1 — driver pinned to a branch the kernel doesn't support

**Break it on purpose:**

```bash
helm upgrade gpu-operator nvidia/gpu-operator -n gpu-operator \
  --reuse-values --set driver.version=525.60.13
# The driver DaemonSet is OnDelete, so nothing happens until we force it:
kubectl delete pod -n gpu-operator -l app=nvidia-driver-daemonset
```

**Step 0 — scope and version.**

```
$ kubectl get nodes -o custom-columns='NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
NODE        GPU
gpu-node-1  <none>
$ kubectl -n gpu-operator get ds nvidia-driver-daemonset \
    -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
nvcr.io/nvidia/driver:525.60.13-ubuntu24.04
```

One node (it's a single-node cluster), and the driver image tag is the thing that just changed. Hypothesis space is already narrow.

**Step 1 — scheduling or runtime?**

```
$ kubectl describe node gpu-node-1 | sed -n '/^Capacity:/,/^System Info/p'
Capacity:
  cpu:                8
  memory:             32601076Ki
  pods:               110
Allocatable:
  cpu:                7910m
  memory:             31549300Ki
  pods:               110
```

`nvidia.com/gpu` is **absent entirely**, not zero. Scheduling layer. The plugin isn't registered — or isn't running.

**Step 2 — snapshot.**

```
$ kubectl get pods -n gpu-operator -o wide
NAME                                       READY   STATUS             RESTARTS   AGE
gpu-operator-7c9d8b6f5-2mzln               1/1     Running            0          2d
gpu-operator-node-feature-discovery-...    1/1     Running            0          2d
nvidia-driver-daemonset-2kx9w              0/1     CrashLoopBackOff   4          6m
nvidia-container-toolkit-daemonset-fghij   0/1     Init:0/1           0          6m
nvidia-device-plugin-daemonset-klmno       0/2     Init:0/2           0          6m
gpu-feature-discovery-uvwxy                0/2     Init:0/2           0          6m
nvidia-dcgm-exporter-zzzzz                 0/1     Init:0/1           0          6m
nvidia-operator-validator-pqrst            0/1     Init:0/4           0          6m
```

Top-down in chain order, the first not-`Running` is the **driver**, and it's `CrashLoopBackOff` — a real cause, not a wait. Everything below is `Init` with **0 restarts**: victims. Note the driver has 4 restarts in 6 minutes, so it's failing fast (not hitting the 21-minute probe budget) — the failure is early in the entrypoint, before module load.

**Step 3 — barrier census, to confirm.**

```
$ kubectl -n gpu-operator exec ds/nvidia-operator-validator -- ls -1 /run/nvidia/validations/
error: unable to upgrade connection: container not found ("nvidia-operator-validator")
$ kubectl debug node/gpu-node-1 -it --image=busybox -- ls -1 /host/run/nvidia/validations/
workload-type
```

The validator's main container doesn't exist yet (it's in `Init`), so read the host. Only `workload-type` — **not even `.driver-ctr-ready`**. Consistent with a driver that never got as far as loading modules.

**Step 4 — read the driver, with `--previous`.**

```
$ kubectl -n gpu-operator logs nvidia-driver-daemonset-2kx9w -c nvidia-driver-ctr --previous | tail -12
DRIVER_ARCH is x86_64
Creating directory NVIDIA-Linux-x86_64-525.60.13
Verifying archive integrity... OK
Uncompressing NVIDIA Accelerated Graphics Driver for Linux-x86_64 525.60.13
Resolving Linux kernel version...
Installing Linux kernel headers...
Installing Linux kernel module files...
Generating Linux kernel version string...
Compiling NVIDIA driver kernel modules...
/usr/src/nvidia-525.60.13/nvidia/nv-mmap.c:341:9: error: too many arguments to function 'vm_insert_page'
make[1]: *** [scripts/Makefile.build:250: /usr/src/nvidia-525.60.13/nvidia/nv-mmap.o] Error 1
make: *** [Makefile:1834: __modpost] Error 2
```

*(Representative: the exact source file, line and gcc diagnostic depend on the kernel/driver pair.)*

**Diagnosis.** Header resolution *succeeded* (no `Could not resolve Linux kernel version`), so the repos have this kernel — but the 525 sources don't compile against its API. **Family F1a, compile variant.** The discriminator was one word: `error:` from gcc, not `Could not resolve`. The two failures look identical at the pod level and have different fixes — one is "the image can't find headers," the other is "the driver branch is too old for this kernel."

**Fix and confirm.**

```bash
helm upgrade gpu-operator nvidia/gpu-operator -n gpu-operator \
  --reuse-values --set driver.version=580.126.20
kubectl delete pod -n gpu-operator -l app=nvidia-driver-daemonset
```

```
$ kubectl -n gpu-operator exec ds/nvidia-operator-validator -- ls -1 /run/nvidia/validations/
.driver-ctr-ready
cuda-ready
driver-ready
plugin-ready
toolkit-ready
workload-type

$ kubectl -n gpu-operator logs -l app=nvidia-cuda-validator -c cuda-validation
[Vector addition of 50000 elements]
Test PASSED
```

Tier 3. Done. Write the log entry.

### Trace 2 — the runtime that "succeeded" but didn't wire

The subtler one. Reproduce it by breaking the *live* state without touching the operands: on the node, remove the `runtimes.nvidia` block from `/etc/containerd/config.toml`, restart containerd, and then put the block back on disk **without** reloading. Now disk and daemon disagree, which is exactly what happens naturally when a config-management tool rewrites the file after the toolkit ran.

**Step 1.**

```
$ kubectl describe node gpu-node-1 | sed -n '/^Capacity:/,/^System Info/p'
Capacity:
  nvidia.com/gpu:     1
Allocatable:
  nvidia.com/gpu:     1
```

Capacity and allocatable both correct. **Runtime layer.** Per Step 1's table, skip the entire driver/plugin half of the chain.

**Step 2 — for completeness, and it's all green.**

```
$ kubectl get pods -n gpu-operator | grep -v Completed
nvidia-driver-daemonset-2kx9w              1/1   Running   0   3h
nvidia-container-toolkit-daemonset-fghij   1/1   Running   0   3h
nvidia-device-plugin-daemonset-klmno       2/2   Running   0   3h
gpu-feature-discovery-uvwxy                2/2   Running   0   3h
nvidia-dcgm-exporter-zzzzz                 1/1   Running   0   3h
nvidia-operator-validator-pqrst            1/1   Running   0   3h
```

Six green pods, correct capacity, `clusterpolicy` state `ready`. **Every dashboard says this node is fine.** But:

```
$ kubectl run cuda-check --restart=Never \
    --image=nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubi8 \
    --overrides='{"spec":{"containers":[{"name":"cuda-check",
      "image":"nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubi8",
      "resources":{"limits":{"nvidia.com/gpu":"1"}}}]}}'
$ kubectl describe pod cuda-check | tail -6
Events:
  Type     Reason   Age   From     Message
  ----     ------   ----  ----     -------
  Normal   Pulled   12s   kubelet  Container image already present on machine
  Warning  Failed   11s   kubelet  Error: failed to create containerd task: failed to
    create shim task: OCI runtime create failed: runc create failed: unable to start
    container process: error during container init: error running hook #0: error
    running hook: exit status 1, stdout: , stderr: Auto-detected mode as 'legacy'
    nvidia-container-cli: initialization error: nvml error: driver not loaded: unknown
```

**Step 5 — ask the live daemon, then disk.**

```
$ sudo crictl info | jq '.config.containerd.runtimes | keys'
[
  "runc"
]
$ sudo crictl info | jq '.config.containerd.defaultRuntimeName'
"runc"

$ sudo grep -n -A4 'runtimes.nvidia' /etc/containerd/config.toml
412:  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
413-    runtime_type = "io.containerd.runc.v2"
414-    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
415-      BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime"
416-      SystemdCgroup = true
```

**There it is.** The config on disk is correct and complete. The running containerd has only `runc`. **Family F2b: the config never took effect.** Nothing in the pod list, the barrier files, the node capacity or the `ClusterPolicy` status could have told you this — the only evidence is the gap between the two commands above.

And note *why* the hook error mentions `driver not loaded` even though the driver is demonstrably fine: containerd fell through to a path where the NVIDIA hook ran without the toolkit's environment, so `libnvidia-container` initialised NVML from a context that has no driver visible. **The error text names a symptom two layers away from the cause.** Chasing "driver not loaded" would have sent you to Step 4 to stare at a healthy driver.

**Fix and confirm.**

```
$ kubectl delete pod -n gpu-operator -l app=nvidia-container-toolkit-daemonset
$ sleep 60
$ sudo crictl info | jq '.config.containerd.runtimes | keys'
[
  "nvidia",
  "nvidia-cdi",
  "nvidia-legacy",
  "runc"
]
$ kubectl delete pod cuda-check && kubectl run cuda-check ...   # as above
$ kubectl logs cuda-check
[Vector addition of 50000 elements]
Test PASSED
```

**The transferable lesson: for family F2, the deciding evidence is `crictl info`, not the config file. Disk state lies.** And the corollary for your runbook: a "GPU node health check" that only asserts pods-Running and capacity-correct will pass on a node where no GPU workload can start. Tier 3 or it didn't happen.

## Practice

On your single-node rented-GPU cluster from 04.1, **deliberately break and recover each of the following**, capturing evidence and writing a five-field failure-mode-log entry for every one. These entries are the direct seed of the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md) deliverable's required `failure-mode-log.md`.

For every break, run the **whole procedure from Step 0**, not just the step you know will find it. The point is to build the reflex, and you'll be surprised how often an earlier step tells you something you'd have missed.

1. **F1a — driver/kernel mismatch.** `helm upgrade … --set driver.version=<an old branch>`, then `kubectl delete pod -n gpu-operator -l app=nvidia-driver-daemonset` (remember: `OnDelete`). Capture the discriminating log line and say which F1 sub-shape it is and why. Recover by pinning a compatible branch.
2. **F2b — runtime not live.** On the node, back up `/etc/containerd/config.toml`, remove the `runtimes.nvidia` block, restart containerd, then restore the file **without** reloading. Deploy a GPU pod and capture the create failure. **Capture both `crictl info` and the `grep` of the config file in the same log entry** — that pair is the evidence. Recover by deleting the toolkit pod; confirm with `crictl info`.
3. **F3 — RuntimeClass missing.** `kubectl delete runtimeclass nvidia` and deploy a pod with `runtimeClassName: nvidia`. Capture the error. Then note how long the operator takes to recreate the object, and what that tells you about relying on reconciliation as a safety net.
4. **F4 — plugin not scheduled.** `kubectl label node <node> nvidia.com/gpu.deploy.device-plugin=false --overwrite`. Watch the plugin pod disappear and `nvidia.com/gpu` leave capacity. Note precisely which step of the procedure catches this and what the *symptom* is at Step 2 (hint: it is "pod absent," which is easy to skim past). Recover by restoring the label to `true`.
5. **F4 — plugin present but can't register.** Add a `NoSchedule` taint the DaemonSet doesn't tolerate, or (harder, more instructive) scale the driver to zero via `nvidia.com/gpu.deploy.driver=false` and watch the plugin's NVML failure block appear. Capture the five hard-coded hint lines.
6. **F5 — capacity vs allocatable.** You can't easily fake an Xid, so instead **observe** the mechanism: read `kubectl -n gpu-operator logs ds/nvidia-device-plugin-daemonset -c nvidia-device-plugin | grep -i xid` for the `Ignoring the following XIDs for health checks:` line, then write down (a) what the plugin does when it sees a critical Xid, (b) which node field changes, (c) why the operator-validator can still pass. Add `sudo dmesg -T | grep -i xid` output (probably empty — that's your clean baseline).
7. **F7 — Pending on a healthy node.** Three sub-experiments: request `nvidia.com/gpu: 0.5` and capture the admission error verbatim; request `nvidia.com/gpu: 2` on a 1-GPU node and capture the scheduler message; add a taint your *workload* doesn't tolerate and capture that message. Explain in one sentence each why the message reads the way it does.
8. **F8 — read, don't reproduce.** Read [gpu-operator #485](https://github.com/NVIDIA/gpu-operator/issues/485) (or nvidia-container-toolkit #48) and verify on your own node that the mitigation is active: `ls -l /dev/char | grep -c nvidia` should be non-zero, and `kubectl -n gpu-operator logs ds/nvidia-operator-validator -c driver-validation` should mention creating symlinks. Write two sentences on why the chain-walking procedure would **not** find this failure.
9. **Cross-reference a real incident.** Read [#1220](https://github.com/NVIDIA/gpu-operator/issues/1220) and map its two log lines onto the families you reproduced by hand. Then write the `PREVENTION` field you'd have needed to catch it before your users did.
10. **Turn the log into detectors.** Set `nodeStatusExporter.enabled=true`, scrape `:8000` (or via the Service the operator creates), and confirm you see `gpu_operator_node_driver_ready`, `_toolkit_ready`, `_cuda_ready`, `_plugin_ready`. Write the four PromQL alert expressions from §6, with justified `for:` durations — and be able to explain why 5 minutes is the wrong window for the driver gauge.

**Acceptance:**
(a) **at least five** documented break/fix entries in the deliverable's `failure-mode-log.md`, each with all five fields and **the specific log line that was the deciding evidence**;
(b) for the F2b entry, both `crictl info` and the on-disk config in the same entry;
(c) the written procedure itself — your own one-page version of §3, with commands, that someone else on your team could run;
(d) the four PromQL alerts with justified thresholds;
(e) the cluster restored to a Tier-3-passing state (`cuda-validation` → `Test PASSED`).

## Common pitfalls

1. **Starting at the Pending workload pod.** `describe` on it yields `0/1 nodes are available: 1 Insufficient nvidia.com/gpu.` — true, and identical for six different root causes. *Mechanism:* the scheduler reports the resource it couldn't satisfy, not why the resource is absent; that information lives in the operand namespace.
2. **Reading the current container's logs on a `CrashLoopBackOff` pod.** The running attempt may be seconds old and still initialising. `--previous` gets the crash you care about. *Mechanism:* `CrashLoopBackOff` means the *last* attempt failed and the next is pending or newborn.
3. **Trusting `/etc/containerd/config.toml` as proof the runtime is wired.** Config on disk that containerd never reloaded is the highest-frequency "looks fine but GPU pods fail" trap. *Mechanism:* two-phase operation — write the file, signal the daemon — and only the second phase can fail silently. `crictl info` is the only evidence.
4. **Grepping for the wrong containerd CRI plugin key.** `cri` at config version 1, `io.containerd.grpc.v1.cri` at 2, `io.containerd.cri.v1.runtime` at 3+. Also check the drop-in `/etc/containerd/conf.d/99-nvidia.toml`. *Mechanism:* `criRuntimePluginName()` in the toolkit switches on the config's `version` field.
5. **Diagnosing a driver pod inside its first 20 minutes.** 60 s initial delay + 120 × 10 s failure budget = up to 21 min before the container is restarted. `0/1 Running` at 8 minutes carries no information. *Mechanism:* the `startupProbe` on `nvidia-driver-ctr`.
6. **Deleting the operator-validator pod to "reset" things.** Its `preStop` hook runs `rm -f /run/nvidia/validations/*-ready`, closing every barrier on the node and pushing every other operand back into `Init`. You've turned one broken stage into a whole-node reconvergence. *Mechanism:* barriers are files on a host path with a single writer.
7. **Assuming all barriers open means the node is healthy now.** Barrier files are latches, not live probes. Nothing removes `toolkit-ready` if containerd is later reconfigured behind the operator's back. *Mechanism:* the only removers are the two `preStop` hooks. Always finish with a Tier-3 test.
8. **Reading the resource *count* and not the resource *name*.** Under `mig.strategy=mixed` the node advertises `nvidia.com/mig-1g.10gb`; under time-slicing with `renameByDefault=true`, `nvidia.com/gpu.shared`. A green chain plus a Pending pod is often just a name mismatch. *Mechanism:* the scheduler matches literal resource keys.
9. **Assuming a driver-pod restart always means a multi-minute rebuild.** As of the 26.3 line, a driver-pod restart that doesn't change the driver version can reuse already-loaded kernel modules — the validator's `isNvidiaModuleLoaded()` checks `/sys/module/nvidia/refcnt` and logs `NVIDIA kernel module already loaded in kernel memory (refcnt=N)`, and the driver container skips the userspace-only reinstall path accordingly. Check your version's behaviour before quoting a recovery estimate.
10. **Not checking whether someone already filed it.** Four issues, four platforms, four years, one error string (`Could not resolve Linux kernel version`). Filtering the tracker to your operator + Kubernetes version is a legitimate diagnostic step, not a shortcut — you still have to understand the fix before applying it.

## Self-check

- **GPU pods are Pending and `nvidia.com/gpu` shows 0 allocatable. Walk your diagnosis.** *Answer:* Don't touch the Pending pod. **Step 0:** scope the blast radius (`kubectl get nodes` with the allocatable column) and record the operator and driver image tags — one node means node-local, a whole pool means an image or config change, everything means cluster-scoped. **Step 1:** `kubectl describe node <node> | sed -n '/^Capacity:/,/^System Info/p'`. `nvidia.com/gpu` *absent* means the plugin never registered; `0` means it registered and reported an empty device list (so NVML found nothing → the driver is broken); capacity 8 / allocatable 7 means a device is unhealthy; capacity correct means it's a runtime-layer problem instead. **Step 2:** `kubectl get pods -n gpu-operator -o wide --field-selector spec.nodeName=<node>`, scan top-down in chain order — driver, toolkit, validator, plugin, GFD, DCGM — and stop at the first that isn't `Running`. `Init:0/N` with 0 restarts is a victim; `CrashLoopBackOff` is a cause; **absent from the list is a label-chain failure**, not a pod failure. **Step 3:** `exec ds/nvidia-operator-validator -- ls -1 /run/nvidia/validations/` — the first *missing* file names the broken stage (empty → driver container; `.driver-ctr-ready` only → `driver-validation`; `+driver-ready` → toolkit/runtime; `+toolkit-ready` → CUDA validation; `+cuda-ready` → device plugin). **Then** read that stage's logs with `-c <container> --previous`. Fix the first broken stage; downstream `Init` pods recover on their own. Finish with Tier 3: `logs -l app=nvidia-cuda-validator -c cuda-validation` → `Test PASSED`.

- **The driver pod is `CrashLoopBackOff`. Name the sub-causes and the log line that discriminates each.** *Answer:* Five. **F1a kernel/headers or branch mismatch** — either `Could not resolve Linux kernel version` (the container's apt repos have no `linux-headers-$(uname -r)`; kernel drift) or gcc/`make` errors like `error: too many arguments to function` and `make: *** [__modpost] Error 2` (headers exist, sources don't compile against this kernel's API). Different fixes: fix the image/repo vs. change the driver branch. **F1b stale/busy module** — `Could not unload NVIDIA driver kernel modules, driver is in use`, usually preceded by an `lsmod | grep nvidia` dump showing a refcount higher than the number of dependent modules; the driver manager then logs `Unable to cleanup driver modules, attempting again with node drain...`. Also covers `nouveau` not blacklisted. **F1c Secure Boot** — `modprobe: ERROR: could not insert 'nvidia': Key was rejected by service`; enrol the MOK or use a signed package. **F1d never loaded** — the `startupProbe` prints `NVIDIA kernel module not loaded` or `nvidia-smi failed`; that's a symptom, so read the entrypoint log above it for the real cause. **F1e version skew** — `Failed to initialize NVML: Driver/library version mismatch`, i.e. `nvidia.ko` ≠ `libnvidia-ml.so`, typically a host package upgrade under a running module or a host driver colliding with the containerized one; needs a module reload, which needs a drain. The discriminator is always the log line, never the pod status.

- **Where does the toolkit write its runtime config, and how do you verify it actually took effect?** *Answer:* Two files — the top-level `/etc/containerd/config.toml` and the drop-in `/etc/containerd/conf.d/99-nvidia.toml` (CRI-O: `/etc/crio/crio.conf` and `/etc/crio/crio.conf.d/99-nvidia.conf`) — adding `runtimes.nvidia`, `runtimes.nvidia-cdi` and `runtimes.nvidia-legacy` blocks whose `options.BinaryName` points into `/usr/local/nvidia/toolkit/`, setting `default_runtime_name = "nvidia"` and `enable_cdi = true`. Verify *intent* with `sudo grep -A4 runtimes.nvidia /etc/containerd/config.toml` — and grep `runtimes.nvidia`, not the CRI plugin key, because that key is `cri` / `io.containerd.grpc.v1.cri` / `io.containerd.cri.v1.runtime` depending on the config version. Verify *reality* with `sudo crictl info | jq '.config.containerd.runtimes | keys'`, which must include `nvidia`. If disk has the block and the live daemon doesn't, containerd was never reloaded — family F2b, the most common "looks fine but GPU pods fail" cause. Note the toolkit's default restart mode is `signal` (SIGHUP), so the absence of a containerd unit restart in the journal is expected and proves nothing.

- **Why can't you request `nvidia.com/gpu: 0.5`, and how does GPU sharing work instead?** *Answer:* `nvidia.com/gpu` is an **extended resource** — a resource name containing `/` that isn't in a `*.kubernetes.io/` namespace, per `IsExtendedResourceName()`. `IsIntegerResourceName()` returns true for every extended resource, so `ValidateResourceQuantityValue()` rejects any quantity whose `MilliValue() % 1000 != 0` with the error `must be an integer`. The validation exists because of the device-plugin contract itself: `ListAndWatch` advertises a *set of discrete device IDs* and `Allocate` hands a container specific IDs — there is no wire representation for a fraction of a device, and the scheduler tracks only an integer count per node. Sharing therefore works by **advertising more integer devices**: MIG exposes each hardware slice as its own device ID (`MIG-<uuid>`), and time-slicing/MPS advertise N replicas of one physical GPU as annotated IDs `GPU-<uuid>::0` … `::N-1` (optionally renamed to `nvidia.com/gpu.shared` with `renameByDefault=true`). The request is always `1`; the fraction lives behind the plugin. Which is exactly why time-sliced attribution is hard — several pods hold IDs that resolve to the same physical UUID (lesson 04.7).

- **A container that's been training for six hours suddenly reports `Failed to initialize NVML: Unknown Error`. Nothing in Kubernetes changed. What happened?** *Answer:* Family F8, and the chain-walking procedure will not find it because the chain is healthy. Recent `runc` requires symlinks under `/dev/char` for any device node injected into a container. The NVIDIA driver doesn't create them. When systemd manages the container's cgroups and anything triggers a Unit reload — `systemctl daemon-reload` suffices — the device cgroup is recomputed and, without the `/dev/char` symlinks, the injected `/dev/nvidia*` nodes are dropped from the container's allowed devices. The process keeps running with a device it can no longer open. Tracked as gpu-operator #485 / nvidia-container-toolkit #48; the GPU Operator mitigates it in the validator's `driver-validation` step via `createDevCharSymlinks()`, whose failure message cites gpu-operator #430 and offers `DISABLE_DEV_CHAR_SYMLINK_CREATION=true` as an escape hatch. Outside Kubernetes the fix is `sudo nvidia-ctk system create-dev-char-symlinks --create-all`, or switching the runtime off systemd cgroup management.

- **MIG: your A100 node has a green chain but the device plugin is crash-looping. Where do you look, and why must a node be drained to reconfigure MIG?** *Answer:* Read the plugin's log for one of three precise errors: `invalid MIG configuration: at least one device with migEnabled=true was not configured correctly: device 0 has no MIG devices configured` (MIG mode on but no GPU instances created — half-applied config); `invalid MIG configuration: … more than one MIG device type present on node` (heterogeneous profiles under `mig.strategy=single` — switch to `mixed` or make them uniform); or `all devices on the node must be configured with the same migEnabled value` (some GPUs MIG-enabled, some not, under `single`). Then read `nvidia.com/mig.config.state`: `pending` means the MIG manager's teardown didn't finish, `rebooting` means a mode change needed a host reboot, `failed` means it gave up. **The drain requirement:** enabling or changing MIG *mode* requires a GPU reset, and a GPU reset requires that no process holds the device. So the MIG manager flips every GPU client's `nvidia.com/gpu.deploy.*` label to a paused value, `kubectl wait --for=delete`s the device-plugin/GFD/DCGM/DCGM-exporter pods with a 5-minute timeout, deletes the validator pods, and only then runs `nvidia-mig-parted`. If a GPU workload won't terminate, that wait times out and the state sticks at `pending` — which is why you drain first rather than hoping.

- **What are the three signals you'd alert on, and why is a 5-minute window wrong for the driver one?** *Answer:* (1) The barrier gauges from `nodeStatusExporter` (off by default; set `nodeStatusExporter.enabled=true`) — `gpu_operator_node_driver_ready`, `_toolkit_ready`, `_cuda_ready`, `_plugin_ready`, each `== 0` by node. This is literally "which barrier is closed on which node," fleet-wide, and it's the highest-leverage thing in this lesson. (2) The controller's own metrics — `gpu_operator_reconciliation_failed_total` rising, `gpu_operator_reconciliation_status` non-success, `gpu_operator_gpu_nodes_total` dropping (an NFD regression), plus `gpu_operator_nodes_upgrades_failed > 0` during a rollout. (3) From kube-state-metrics, `kube_node_status_allocatable{resource="nvidia_com_gpu"} == 0`, **and separately allocatable `<` capacity**, which is the unhealthy-device detector and is otherwise nearly invisible. The 5-minute window is wrong for the driver gauge because the driver container's `startupProbe` budget is 60 s + 120 × 10 s ≈ **21 minutes**: any legitimate driver restart — a version bump, a node reboot, a pod eviction — leaves `driver_ready` at 0 for minutes. A 5-minute `for:` would page on every normal driver lifecycle event and get the alert muted, which is worse than not having it. Use `for: 15m` and rely on the blast-radius question to distinguish a rollout from an outage.

## Connections & what's next

The diagnosis order here — walk up the chain, trust the layer that holds reality over the layer that holds intent, and never accept anything less than a Tier-3 confirmation — is the habit every later break/fix lesson in this module reuses. Lesson 04.4's "CUDA driver insufficient inside a container" is family F2 seen from the CDI side. Lesson 04.5's driver-upgrade failures are family F1 running under the `nvidia.com/<driver>-driver-upgrade-state` state machine, with the `maxParallelUpgrades`/`maxUnavailable` arithmetic from 04.1 §11 deciding how many nodes you break at once. Lesson 04.6's MIG reconfiguration failures are family F6 in detail. Lesson 04.9's DRA scheduling failures are "a stage in a gated chain broke" in `resource.k8s.io` clothing.

The failure-mode log format started here is the exact artifact 04.5 and 04.6 add entries to, and the one your capstone (04.10) assembles into `failure-mode-log.md` with at least five real break/fix entries. Its `PREVENTION` fields are the specification for the alerting your cost operator ships.

Next: **[04.3 · Device-plugin recap and the pod-resources API](03-device-plugin-recap-pod-resources.md)** moves past the Operator's own health — which you can now diagnose cold — and into the API surface you'll build against: the kubelet pod-resources socket that tells you which pod holds which GPU device, the foundation of per-pod cost attribution.

## References & further reading

**Primary sources**

1. [NVIDIA/gpu-operator](https://github.com/NVIDIA/gpu-operator), tag **v26.3.3** — `cmd/nvidia-validator/main.go` for the barrier file names, the four validation semantics, `validateHostDriver()` / `validateDriverContainer()`, `runCommandWithWait()`'s unbounded retry, `isNvidiaModuleLoaded()`'s `refcnt` check, and the `createDevCharSymlinks()` error message that cites issue #430. `assets/state-*/0500_daemonset.yaml` for init-container names, wait strings, the driver `startupProbe` budget and the validator's `preStop` hook. `controllers/operator_metrics.go` for the `gpu_operator_*` metric names; `cmd/nvidia-validator/metrics.go` for the node-status-exporter gauges.
2. [NVIDIA/gpu-driver-container](https://github.com/NVIDIA/gpu-driver-container) — the `nvidia-driver` entrypoint per base image. Source of `Resolving Linux kernel version...`, `Could not resolve Linux kernel version`, `Could not unload NVIDIA driver kernel modules, driver is in use`, `Loading NVIDIA driver kernel modules...`, `Done, now waiting for signal`, and the `nvidia-driver-startup-probe` script's `NVIDIA kernel module not loaded` / `nvidia-smi failed`.
3. [NVIDIA/k8s-driver-manager](https://github.com/NVIDIA/k8s-driver-manager) — the `k8s-driver-manager` init container. `Checking if the currently loaded NVIDIA driver version and configuration matches the desired state...`, `Unable to cleanup driver modules, attempting again with node drain...`, and the eviction/drain flags (`ENABLE_GPU_POD_EVICTION`, `ENABLE_AUTO_DRAIN`, `DRAIN_TIMEOUT_SECONDS`).
4. [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin), tag **v0.19.3** — `internal/plugin/server.go` for the registration signature (`Starting to serve …`, `Registered device plugin for … with Kubelet`) and the `'…' device marked unhealthy:` line; `internal/plugin/factory.go` for the five-line NVML failure hint block; `internal/rm/health.go` for `XidCriticalError: Xid=%d on Device=%s`, `DP_DISABLE_HEALTHCHECKS` / `DP_ENABLE_HEALTHCHECKS`, and the `FIXME` noting there is no recovery from `Unhealthy`; `internal/rm/device_map.go` for the three exact MIG-configuration errors; `cmd/nvidia-device-plugin/main.go` for `No devices found. Waiting indefinitely.` and the `inotify:` re-registration line.
5. [NVIDIA/nvidia-container-toolkit](https://github.com/NVIDIA/nvidia-container-toolkit), tag **v1.19.1** — `cmd/nvidia-ctk-installer/container/runtime/containerd/` for `DefaultConfig`, `DefaultDropInConfig = /etc/containerd/conf.d/99-nvidia.toml`, `DefaultRestartMode = "signal"`; `pkg/config/engine/containerd/containerd.go` for the config-version → CRI-plugin-key mapping that makes naive greps fail.
6. [NVIDIA/libnvidia-container](https://github.com/NVIDIA/libnvidia-container) — `src/error.c`'s `error_set_nvml()` and the CLI's `initialization error: %s` format, which together produce the `nvidia-container-cli: initialization error: nvml error: …` string; `src/nvml.h` for `NVML_ERROR_DRIVER_NOT_LOADED = 9`.
7. [kubernetes/kubernetes](https://github.com/kubernetes/kubernetes) — `pkg/kubelet/cm/devicemanager/manager.go` for the admission errors quoted in Step 9 (`requested number of devices unavailable for %s. Requested: %d, Available: %d`, `no healthy devices present…`, `cannot allocate unregistered device %s`) and the `kubelet_internal_checkpoint` file; `pkg/kubelet/cm/devicemanager/plugin/v1beta1/server.go` for the kubelet-side registration log and the `requested API version … is not supported` / `the ResourceName %q is invalid` errors; `pkg/apis/core/helper/helpers.go` + `pkg/apis/core/validation/validation.go` for `IsExtendedResourceName` / `IsIntegerResourceName` and the `must be an integer` rejection; `pkg/scheduler/framework/plugins/noderesources/fit.go` for `Insufficient %v`.
8. [containerd](https://github.com/containerd/containerd) — `internal/cri/config/config.go` (`no runtime for %q is configured`) and `internal/cri/server/podsandbox/sandbox_run.go` (`failed to get sandbox runtime: %w`). Read once so you recognise the error as containerd's, not NVIDIA's.
9. [NVIDIA/mig-parted](https://github.com/NVIDIA/mig-parted) — `deployments/container/reconfigure-mig.sh`, the MIG manager's actual reconfiguration script: the `nvidia.com/mig.config.state` transitions (`pending` → `success`/`failed`/`rebooting`), the GPU-client teardown by deploy label, the `kubectl wait --for=delete --timeout=5m` calls, and `MIG mode change did not take effect after rebooting`.

**Real-world incidents**

10. [gpu-operator #1220](https://github.com/NVIDIA/gpu-operator/issues/1220) (EKS 1.30 kernel drift; two families at once), plus the same-mechanism cluster [#205](https://github.com/NVIDIA/gpu-operator/issues/205), [#526](https://github.com/NVIDIA/gpu-operator/issues/526), [#666](https://github.com/NVIDIA/gpu-operator/issues/666). And [gpu-operator #485](https://github.com/NVIDIA/gpu-operator/issues/485) / [nvidia-container-toolkit #48](https://github.com/NVIDIA/nvidia-container-toolkit/issues/48) / [gpu-operator #430](https://github.com/NVIDIA/gpu-operator/issues/430) for the `/dev/char` / `Failed to initialize NVML: Unknown Error` family. *Titles and substance confirmed via search this session; `github.com` HTML is not fetchable through this environment's egress proxy, so individual threads were not re-read. Every log string attributed to these issues was independently verified against the primary source trees listed above.* The general practice stands regardless: filter the tracker to your operator + Kubernetes version before assuming a log line is unprecedented.

**Deeper dives**

11. [NVIDIA GPU Operator documentation — Troubleshooting and Release Notes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/) — the vendor's own symptom-to-cause tables, and the place to confirm which behaviours (CDI-by-default from the 25.10 line, the `NVIDIADriver` CRD from 26.3.0, module reuse on same-version restarts) are active on the version you're debugging. *Not fetchable this session — `docs.nvidia.com` is proxy-blocked in this environment — which is why every claim above is sourced from the repositories instead.* Re-check at study time before quoting field names or version-specific behaviour.
