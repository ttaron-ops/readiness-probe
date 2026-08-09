---
lesson: "04.7"
title: "Time-slicing and the attribution trap"
module: "04"
concept: "Time-slicing and the attribution trap"
status: not-started
est_time: "8h"
artifacts: []
---
# 04.7 · Time-slicing and the attribution trap
> **Concept.** Time-slicing multiplies one GPU into N schedulable replicas that share it by taking turns — with no memory isolation, no fault isolation, and no per-pod allocation signal — so cost cannot be attributed from allocation counts and must fall back to DCGM per-PID utilisation.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters

Time-slicing is the cheapest, most common, most abused way to put more than one
pod on a GPU. It is one ConfigMap key — `replicas: N` — and suddenly a node that
advertised `nvidia.com/gpu: 1` advertises `nvidia.com/gpu: 4`. Utilisation on
idle inference fleets jumps, the scheduler stops wedging, and everyone is happy
until finance asks the question your whole differentiator is built on: **who owes
what for that GPU?**

Under time-slicing the honest answer is *you cannot tell from Kubernetes*. Every
one of the N replicas resolves to the **same physical GPU UUID**. The
allocation-count signal that works perfectly for whole-GPU and MIG — count the
`nvidia.com/gpu` requests per namespace, multiply by $/GPU-hr — silently produces
garbage: it either double-counts the physical GPU N times or, if you divide by N,
charges an idle tenant exactly as much as the tenant pinning the SMs. This is the
"attribute cost to a time-sliced GPU" interview trap, and it is *your* home turf.
The senior answer is not "use time-slicing" or "avoid it" — it is naming precisely
which signal died (per-pod allocation) and which signal you fall back to (DCGM
per-PID utilisation), and wiring the fallback into the cost operator.

It matters twice over because time-slicing also removes the two isolation
guarantees an operator quietly relies on. A neighbour pod can exhaust VRAM and
**OOM you** — a fault your own workload did nothing to cause. If your cost report
cannot see the neighbour, your incident review cannot either.

## What's new here

You have seen GPUs advertised as integer extended resources (`nvidia.com/gpu`)
since lesson 04.1, and you have seen MIG partition a GPU into hardware-isolated
slices with their own UUIDs. Time-slicing is the opposite design point:

- **Sharing by taking turns, not by building walls.** MIG carves silicon into
  fixed partitions each with dedicated SMs, L2, and framebuffer. Time-slicing
  carves *time*: the GPU's hardware context-switches between clients on the
  normal compute time-slice, exactly as it does for any two CUDA processes that
  happen to co-reside — the device plugin just stops enforcing exclusivity so
  the scheduler will place N pods where it used to place one.
- **The `replicas` count is a lie the scheduler believes.** `replicas: 4` does
  **not** create four GPUs, four contexts, or four memory partitions. It creates
  four identical advertisements of one device. Nothing at the hardware level
  changed; only the node's reported capacity did.
- **The pod-resources API becomes your ground truth, not the scheduler.** To
  prove attribution died you query the kubelet's pod-resources gRPC socket and
  read the `device_ids` each pod actually received — and watch them collide.
- **DCGM per-PID/per-pod utilisation is the fallback signal.** `dcgm-exporter`
  with the pod-association fields (or profiling metrics keyed by PID) is the only
  layer that can say "PID 4127, pod `tenant-a/infer-3`, used 61% of the SMs this
  minute." That sentence is your entire attribution story under time-slicing.

## Core notes — the meat

### 1 — Enabling time-slicing via the device-plugin ConfigMap

With the GPU Operator (the standard path), time-slicing is a sharing config the
`nvidia-device-plugin` consumes. You author a ConfigMap of named configs and tell
the plugin which one to apply, cluster-wide or per-node via a node label.

```yaml
# time-slicing-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        # failRequestsGreaterThanOne: true makes a request of >1 replica fail,
        # so a pod cannot silently ask for "2 slices" and imply isolation it
        # will not get. Keep this true — it is a guardrail against the trap.
        failRequestsGreaterThanOne: true
        resources:
          - name: nvidia.com/gpu
            replicas: 4
```

Apply it and point the operator's `ClusterPolicy` (or the plugin) at it:

```bash
kubectl create -f time-slicing-config.yaml

# Tell the operator which config to use by default:
kubectl patch clusterpolicy/cluster-policy \
  -n gpu-operator --type merge \
  -p '{"spec":{"devicePlugin":{"config":{"name":"time-slicing-config","default":"any"}}}}'
```

You can also opt in **per node** so only a labelled pool is sliced, leaving
training nodes exclusive:

```bash
kubectl label node <node> nvidia.com/device-plugin.config=any
```

After the plugin restarts, capacity multiplies. A single physical H100 node:

```bash
$ kubectl describe node <node> | grep -A3 Allocatable
Allocatable:
  nvidia.com/gpu:  4          # was 1 — one physical GPU, now 4 replicas
```

Confirm the labelled state and that this is *shared*, not partitioned, silicon:

```bash
$ kubectl get node <node> -o json \
    | jq '.metadata.labels | with_entries(select(.key|test("nvidia.com")))'
{
  "nvidia.com/gpu.count": "1",              # ONE physical GPU
  "nvidia.com/gpu.product": "NVIDIA-H100-80GB-HBM3",
  "nvidia.com/gpu.replicas": "4",           # advertised as 4
  "nvidia.com/gpu.sharing-strategy": "time-slicing"
}
```

`gpu.count: 1` next to `nvidia.com/gpu: 4` allocatable is the whole story in two
lines: **one device, four tickets.** (Without the GPU Operator, the raw
[k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin) takes the same
`sharing.timeSlicing` block via its `-config-file` flag; the schema is identical.)

### 2 — Property one: no memory or fault isolation

Time-slicing gives every client the **full 80 GB framebuffer** and the full SM
array on its turn. There is no VRAM cap, no MPS memory limit, no MIG wall. Two
consequences a cost/reliability operator must encode:

- **VRAM is first-come, uncapped.** If pod A allocates 78 GB, pod B's next
  `cudaMalloc` returns `out of memory` and the framework aborts — B is OOM-killed
  by CUDA, not by the kubelet, so it shows as a **CUDA error / non-zero exit**,
  not a Kubernetes `OOMKilled` (that flag only tracks host RAM cgroups). Your
  incident tooling that greps for `OOMKilled` will miss it entirely.
- **A fatal fault is not contained.** An ECC double-bit error, an XID 43/74, or a
  wedged kernel on the shared device degrades or halts *every* co-resident client.
  There is no blast-radius boundary — the boundary is the physical GPU.

This is why time-slicing is for **cooperative, trusted, bursty** workloads (dev
notebooks, low-QPS inference of your own services) and never for hostile
multi-tenancy.

### 3 — Property two: no per-pod allocation signal (the attribution trap)

Here is the mechanism that breaks cost attribution. Query the kubelet
**pod-resources API** — the authoritative record of which device IDs each pod was
handed — for two pods scheduled onto the sliced GPU:

```bash
# On the node, over the kubelet's pod-resources unix socket:
$ grpcurl -plaintext -unix \
    /var/lib/kubelet/pod-resources/kubelet.sock \
    v1.PodResourcesLister/List | jq '.pod_resources[]
      | select(.namespace=="tenants")
      | {pod: .name, dev: .containers[].devices[].device_ids}'
{ "pod": "infer-a", "dev": ["GPU-8f2c...-e91a"] }
{ "pod": "infer-b", "dev": ["GPU-8f2c...-e91a"] }   # SAME UUID
```

Both pods received the **identical physical GPU UUID** `GPU-8f2c…-e91a`. The
device plugin never minted per-replica identities — the replicas are indices into
one device, and only the underlying UUID is reported. Therefore:

```
allocation_count(tenant) × $per_gpu_hour   →  MEANINGLESS
```

- Sum the `nvidia.com/gpu` requests and you count the same physical GPU up to N
  times → you bill 4× the hardware that exists.
- Divide the GPU's cost evenly by N and an **idle** replica is charged the same as
  a replica saturating the SMs → you punish the frugal tenant and subsidise the
  greedy one. Allocation count carries **zero** information about who used the
  device, because allocation is uniform by construction.

The allocation signal is dead. Do not try to resurrect it with heuristics.

### 4 — The fallback: DCGM per-PID / per-pod utilisation

The only layer that can distinguish tenants on a shared device is **usage**, and
the only place usage is keyed to a process is DCGM. `dcgm-exporter` running as the
GPU Operator's DaemonSet enriches per-GPU metrics with the owning pod, and the
device-monitoring path can attribute to **PID → container → pod** even when the
GPU UUID is shared:

```bash
# Per-pod SM occupancy on the shared GPU (Prometheus form):
DCGM_FI_PROF_SM_ACTIVE{gpu="0", pod="infer-a", namespace="tenants"}  0.61
DCGM_FI_PROF_SM_ACTIVE{gpu="0", pod="infer-b", namespace="tenants"}  0.12
DCGM_FI_DEV_FB_USED{gpu="0", pod="infer-a"}                          74112   # MiB
DCGM_FI_DEV_FB_USED{gpu="0", pod="infer-b"}                          2048
```

Now attribution has a defensible basis: share the GPU's $/hr **in proportion to
integrated SM-active time (or FB-used, per your policy) per pod**, not in
proportion to allocation:

```
cost(pod) = gpu_hourly_cost
          × ∫ DCGM_FI_PROF_SM_ACTIVE[pod] dt  /  Σ_pods ∫ SM_ACTIVE dt
```

For your Go cost operator this is the time-sliced hard case: watch pods with the
`nvidia.com/gpu.sharing-strategy=time-slicing` node label, join pod-resources
(who is on which physical UUID) with DCGM profiling series (how much each used),
and emit a *utilisation-weighted* charge. The pod-resources join is what tells you
which pods contend for the *same* denominator; DCGM is what splits it fairly.

> **Note on the profiling metrics.** `DCGM_FI_PROF_*` (SM active/occupancy, tensor
> active) come from the profiling module and on some driver/GPU combinations are
> collected device-wide; verify on your SKU that they carry the pod label under
> time-slicing. If a metric only resolves per-GPU, fall back to
> `DCGM_FI_DEV_GPU_UTIL` with the accounting/PID association, and state the
> precision limit in the report. Naming the limit is the senior move.

## Worked example — bill two tenants on one sliced H100

Setup: one H100-80GB, `replicas: 4`, two tenant pods land on it for a 1-hour
window. Node $/hr = $2.40 (worked placeholder — refresh at study time).

Observed, integrated over the hour from DCGM:

| Pod       | ∫ SM_ACTIVE (SM-hours) | Peak FB used |
|-----------|------------------------|--------------|
| `infer-a` | 0.55                   | 74 GB        |
| `infer-b` | 0.10                   | 2 GB         |

**Naive allocation split** (the trap): each pod holds 1 replica of 4 →
`$2.40 / 4 = $0.60` each, and the other two idle replicas' $1.20 vanishes or gets
mis-socialised. `infer-b`, nearly idle, is billed the same as `infer-a`. Finance
cannot defend this and neither can you.

**Utilisation-weighted split** (the fallback, correct):

```
total SM-hours = 0.55 + 0.10 = 0.65
infer-a = $2.40 × 0.55/0.65 = $2.03
infer-b = $2.40 × 0.10/0.65 = $0.37
```

The greedy tenant carries the cost; the frugal one pays for its 15% of the work.
And note the **isolation footnote you must attach**: `infer-a` at 74 GB left
`infer-b` 6 GB of headroom — if `infer-b` had tried to load a 8 GB model it would
have hit `CUDA out of memory` and died with a non-zero exit, *not* `OOMKilled`.
The bill and the incident are the same physical fact: no isolation.

## Practice — the time-sliced hard case (feeds the deliverable)

Produce two artifacts for the [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)
deliverable: a **neighbour-OOM demonstration** and a **same-UUID proof**.

1. **Enable slicing.** Apply the `replicas: 4` ConfigMap above, patch the
   ClusterPolicy (or label a node), and confirm `nvidia.com/gpu: 4` allocatable
   with `nvidia.com/gpu.count: 1`. Capture the `describe node` and the label dump.
2. **Demonstrate no isolation (neighbour OOM).** Schedule two pods on the sliced
   GPU, each requesting `nvidia.com/gpu: 1`. In pod A, allocate ~78 GB
   (`torch.empty(int(78e9//2), dtype=torch.float16, device='cuda')` or a
   `cudaMalloc` loop). In pod B, attempt a normal model load. Capture pod B's
   `CUDA out of memory` / non-zero exit and show `kubectl get pod` — note the
   status is **not** `OOMKilled`. Document that CUDA, not the kubelet, killed it.
3. **Prove no per-pod allocation signal (same-UUID).** From the node, query the
   pod-resources socket (grpcurl form above) for both pods and show the
   `device_ids` are the **identical physical GPU UUID**. This is the proof that
   allocation counts cannot attribute cost.
4. **Show the fallback works.** Scrape `dcgm-exporter` and capture
   `DCGM_FI_PROF_SM_ACTIVE` (or the documented fallback) split per pod; compute
   the utilisation-weighted charge as in the worked example.

**Acceptance:** a documented neighbour-OOM (pod B's CUDA OOM with pod status
proving it was not a k8s `OOMKilled`) **and** the same-UUID proof (both pods'
pod-resources `device_ids` equal to one physical GPU UUID), plus a one-paragraph
written conclusion: allocation count is unusable under time-slicing; attribution
uses DCGM per-pod utilisation. These are the time-sliced hard case for the
operator.

## Self-check

**(a) Why can a neighbour pod OOM you under time-slicing — what isolation is
missing?**
**Answer:** Time-slicing shares the device by time-multiplexing only; it provides
**no memory isolation and no fault isolation**. Every client sees the full
framebuffer with no per-client VRAM cap, so a neighbour that allocates most of the
80 GB leaves your next `cudaMalloc` to fail with `CUDA out of memory`. It surfaces
as a CUDA error / non-zero exit, not a Kubernetes `OOMKilled` (that flag tracks
host-RAM cgroups, not VRAM). A fatal GPU fault (XID, double-bit ECC) is likewise
uncontained and hits all co-resident clients — the blast radius is the whole
physical GPU.

**(b) Why can't you attribute cost to a tenant from allocation counts under
time-slicing?**
**Answer:** All N replicas are advertisements of one device; the pod-resources API
hands every pod the **same physical GPU UUID**. Allocation is uniform by
construction, so counting `nvidia.com/gpu` requests either counts one GPU up to N
times (over-bills the hardware that exists) or, split evenly, charges an idle
replica the same as a saturating one. The allocation signal carries zero
information about who actually used the SMs, so it cannot form a defensible bill.

**(c) What signal CAN you attribute time-sliced usage with, and where does it come
from?**
**Answer:** Per-pod (per-PID) **utilisation** from DCGM — `dcgm-exporter`'s
profiling/device-monitoring metrics such as `DCGM_FI_PROF_SM_ACTIVE` and
`DCGM_FI_DEV_FB_USED`, associated to PID → container → pod. You split the GPU's
$/hr in proportion to each pod's integrated SM-active time (joined against
pod-resources to know which pods share the same physical device). That is the only
layer that distinguishes tenants once the allocation signal has collapsed.

## Resources

1. **NVIDIA GPU Operator — Time-Slicing GPUs.** Canonical config for the
   `sharing.timeSlicing.replicas` ConfigMap and ClusterPolicy wiring.
   https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html
2. **ScaleOps — Kubernetes GPU Sharing.** Practitioner comparison of time-slicing
   vs MPS vs MIG and their isolation/attribution trade-offs.
   https://scaleops.com/blog/kubernetes-gpu-sharing/
3. **NVIDIA k8s-device-plugin — Shared Access (time-slicing).** The raw
   `-config-file` schema behind the operator, plus `failRequestsGreaterThanOne`.
   https://github.com/NVIDIA/k8s-device-plugin#shared-access-to-gpus
