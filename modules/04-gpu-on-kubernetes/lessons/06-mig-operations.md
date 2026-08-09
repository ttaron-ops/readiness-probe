---
lesson: "04.6"
title: "MIG Operations & Per-Slice Attribution"
module: "04"
concept: "MIG Operations & Per-Slice Attribution"
status: not-started
est_time: "9h"
artifacts: []
---
# 04.6 · MIG Operations & Per-Slice Attribution
> **Concept.** On Kubernetes, MIG is a resource-surfacing and drain-to-reconfigure problem — each slice becomes a distinct schedulable resource with its own UUID, which is what gives you clean per-slice cost attribution.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters

You already know (from 03) what MIG *is* at the silicon level: a GPU partitioned into isolated instances, each with a fenced slice of SMs, L2, memory, and its own memory-bandwidth path. That isolation is the whole point for a platform team, but it's worthless until Kubernetes can *see* the slices and *attribute* them.

The reason this lands you a senior GPU-platform role is **cost/observability**. The competing way to share a GPU is **time-slicing**, where N pods all get injected the *same* physical GPU and the same `nvidia.com/gpu` device UUID; the scheduler thinks it handed out one card N times. When finance asks "which team burned that H100 last month," time-slicing has no answer — every pod shows the same UUID and the same 100%-utilized device. That is the *attribution hole*.

MIG closes it. Each slice surfaces as its *own* resource name (`nvidia.com/mig-1g.10gb`) backed by its *own* MIG-device UUID (`MIG-GPU-xxxx/gi/ci`). The pod-resources API tells you exactly which slice UUID went to which pod, DCGM emits per-MIG-instance utilization and memory metrics, and you can join `pod → MIG-UUID → namespace/team → cost`. That join is the deliverable of this whole module. Time-slicing gives you density; MIG gives you density *plus* a clean invoice. Knowing when and how to run MIG on Kubernetes — and being able to explain the attribution difference in one breath — is the differentiator.

But MIG on Kubernetes has a sharp operational edge: **you cannot reconfigure a GPU's partition layout while anything is using it**. Changing profiles is a drain-and-tear-down operation, and the failure mode ("in use by another client") is one you must be able to trigger and recover on demand.

## What's new here

Lesson 03 was silicon: profiles, SM/memory fractions, isolation guarantees. This lesson never re-derives a profile's SM count. Everything here is the **Kubernetes operations layer** on top:

| Lesson 03 (silicon) | Lesson 04.6 (operations) |
|---|---|
| A profile carves SMs + memory | A profile becomes a *ConfigMap label* the operator applies via `mig-manager` |
| An instance is isolated in HW | An instance is a *schedulable resource* `nvidia.com/mig-<profile>` with a UUID |
| Reconfiguring re-partitions silicon | Reconfiguring requires *draining GPU clients first* and can fail mid-flight |
| MIG "exists" on the card | `mig.strategy` decides how the node's capacity is *reported to the scheduler* |
| The GPU has a UUID | Each MIG instance has its *own* UUID → per-slice attribution via pod-resources API + DCGM |

The tools that make this operational: the **GPU Operator** (turns MIG on), **`mig-manager`** (a DaemonSet that reads a `mig.config` label and applies the requested geometry), the **device plugin** (advertises the slices), the **pod-resources API** (maps pod → device UUID), and **DCGM-exporter** (per-instance telemetry).

## Core notes

### Enabling MIG via the Operator

MIG mode is a per-GPU hardware setting (persisted in the GPU's InfoROM) that requires a **GPU reset** to change — which is why it's disruptive. The Operator manages this for you when MIG is enabled in the `ClusterPolicy`:

```yaml
spec:
  mig:
    strategy: mixed          # none | single | mixed
  migManager:
    enabled: true
    config:
      name: default-mig-parted-config   # ConfigMap of named geometries
```

`mig-manager` runs as a DaemonSet on MIG-capable nodes. It watches the node label **`nvidia.com/mig.config`**; when that label changes it: cordons/prepares the node, ensures no clients hold the GPU, sets MIG mode on (resetting the GPU if needed), creates the GPU Instances (GIs) and Compute Instances (CIs) per the named geometry, and writes back a status label **`nvidia.com/mig.config.state`** (`pending` → `success`, or `failed`). Under the hood it drives `nvidia-smi mig -cgi/-cci` (or `nvidia-mig-parted`).

### Picking profiles for the SKU

Profiles are fixed per silicon; on Kubernetes you pick a *named geometry* in the ConfigMap. The memory tag is per-slice framebuffer, and the `Ng` tag is the SM/compute fraction:

- **A100 40GB:** `1g.5gb` (×7), `2g.10gb` (×3), `3g.20gb` (×2), `7g.40gb` (×1); also `1g.5gb+me` and mixed layouts.
- **A100 80GB:** `1g.10gb` (×7), `1g.20gb`, `2g.20gb`, `3g.40gb`, `7g.80gb`.
- **H100 80GB / H200 141GB:** `1g` variants scale to the bigger framebuffer — H100 80GB gives `1g.10gb` (×7), `2g.20gb`, `3g.40gb`, `7g.80gb`, plus `1g.10gb+me` and `1g.20gb` (the memory-doubled single-slice for memory-bound inference); H200 mirrors the geometry at ~141GB so slices are proportionally larger (e.g. `1g.18gb`).
- **Choosing:** inference / notebooks / CI → smallest slice (`1g`) for max density and cheapest per-tenant unit; small fine-tunes → `2g`/`3g`; anything needing NVLink, the full memory bandwidth, or multi-GPU → **don't MIG it**, hand out the whole GPU. Right-sizing the slice to the workload is the cost lever: a `1g.10gb` inference pod on an H100 costs ~1/7th of the card instead of stranding 90% of an undivided GPU.

`me` = "media extensions" (extra NVDEC/NVJPEG on the slice) — relevant for video/vision inference.

### `mig.strategy`: single vs mixed

This controls *how the node reports MIG capacity to the scheduler*, and it's a frequent interview trap:

- **`single`** — the node exposes MIG slices under the *generic* `nvidia.com/gpu` resource name, and **all GPUs on the node must have the identical MIG geometry**. A node with 8× A100 each split into 7× `1g.5gb` advertises `nvidia.com/gpu: 56`. Pros: workloads request plain `nvidia.com/gpu: 1` and don't need to know about MIG; scheduling is homogeneous. Cons: you cannot mix profiles on a node, and the *resource name hides the profile*.
- **`mixed`** — each distinct profile is advertised under its **own** resource name: `nvidia.com/mig-1g.5gb: 7`, `nvidia.com/mig-2g.10gb: 3`, etc. A single GPU can host *different-sized* slices, and a node can host different geometries per GPU. Pods request the exact slice: `nvidia.com/mig-1g.10gb: 1`. Pros: heterogeneous slices, explicit profile in the request, and the resource name itself is an attribution key. Cons: the scheduler needs the right resource names; capacity is fragmented across resource types.

For a multi-tenant platform with mixed workloads and per-team billing, **mixed** is almost always the answer — the per-profile resource name is half of your attribution story.

`kubectl describe node` shows the difference immediately:
```
# strategy=single
Capacity:
  nvidia.com/gpu: 56
# strategy=mixed
Capacity:
  nvidia.com/mig-1g.5gb:  21
  nvidia.com/mig-2g.10gb: 6
  nvidia.com/mig-3g.20gb: 4
```

### Why reconfiguration requires a drain — and what it tears down

Repartitioning destroys the current GIs/CIs and (if toggling MIG mode) resets the GPU. You **cannot** do that while a process holds a CUDA context on the card — the driver refuses with:

```
Unable to destroy GPU instance ... In use by another client
```

`mig-manager`'s reconfiguration therefore drains GPU clients first. What gets torn down and why the drain is mandatory:
- **Running CUDA contexts** hold the GPU/MIG device open (non-zero refcount); the GI/CI can't be destroyed under them.
- **The GI/CI objects themselves** are deleted and recreated — every existing MIG-device UUID on that GPU becomes invalid. Any pod that was scheduled onto an old slice UUID now points at nothing.
- **Device-plugin advertisements** must be regenerated; the kubelet's device manager re-reads capacity.

So the sequence is: cordon → evict/complete GPU pods on the node (drain) → `mig-manager` destroys CIs then GIs, sets mode, creates new GIs/CIs → device plugin re-advertises → uncordon. The operator gates this with a node label so you can watch it. Skipping the drain is exactly what produces the "in use by another client" error.

### Per-slice attribution: the pod-resources API + UUID mapping

This is the payoff. When a pod requests `nvidia.com/mig-1g.10gb: 1`, the device plugin allocates a specific MIG device and injects its UUID as `NVIDIA_VISIBLE_DEVICES`. You recover the pod→UUID mapping from the **kubelet pod-resources gRPC API** (`/var/lib/kubelet/pod-resources/kubelet.sock`), which lists, per container, the `resource_name` and the allocated `device_ids` (the MIG-device UUIDs). DCGM-exporter labels its metrics with the same GPU/GI/CI identity, so you can join:

```
pod (namespace/team) → container → MIG-device UUID → DCGM per-instance util/mem → cost
```

That join is impossible under time-slicing, where every co-scheduled pod gets the *same* parent GPU UUID and DCGM only sees one device — you can measure the card's total utilization but cannot split it per tenant. MIG's distinct-UUID-per-slice is precisely what makes the attribution clean.

## Worked example

Set `mixed`, apply a profile, schedule a slice, map its UUID, then trigger and recover the reconfig error. (Rent an A100/H100 for the session, or use a shared cloud MIG node.)

**1. Enable MIG mixed + confirm mig-manager.**
```bash
$ kubectl -n gpu-operator patch clusterpolicy cluster-policy --type merge \
    -p '{"spec":{"mig":{"strategy":"mixed"}}}'
$ kubectl -n gpu-operator get pods -l app=nvidia-mig-manager
NAME                       READY   STATUS    NODE
nvidia-mig-manager-abcde   1/1     Running   gpu-a100
```

**2. Apply a geometry by labeling the node.** The label value names a key in the `mig-parted` ConfigMap:
```bash
$ kubectl label node gpu-a100 nvidia.com/mig.config=all-1g.5gb --overwrite
node/gpu-a100 labeled
# watch mig-manager reconfigure (it drains GPU clients first):
$ kubectl get node gpu-a100 -L nvidia.com/mig.config.state -w
gpu-a100   pending
gpu-a100   success
```

**3. Confirm the slices are advertised as distinct resources.**
```bash
$ kubectl get node gpu-a100 -o jsonpath='{.status.capacity}' | tr ',' '\n' | grep mig
"nvidia.com/mig-1g.5gb":"7"
$ kubectl -n gpu-operator exec ds/nvidia-driver-daemonset -- nvidia-smi -L
GPU 0: NVIDIA A100-SXM4-40GB (UUID: GPU-abc...)
  MIG 1g.5gb  Device  0: (UUID: MIG-3e1d...)
  MIG 1g.5gb  Device  1: (UUID: MIG-7a09...)
  ...
```

**4. Schedule a pod onto a slice.**
```yaml
apiVersion: v1
kind: Pod
metadata: { name: mig-consumer, labels: { team: search } }
spec:
  containers:
  - name: cuda
    image: nvidia/cuda:12.4.1-base-ubuntu22.04
    command: ["sh","-c","nvidia-smi -L && sleep 3600"]
    resources:
      limits: { nvidia.com/mig-1g.5gb: 1 }
```
```bash
$ kubectl apply -f mig-consumer.yaml
$ kubectl logs mig-consumer | grep MIG
  MIG 1g.5gb  Device 0: (UUID: MIG-3e1d...)   # the slice this pod owns
```

**5. Map pod → MIG-UUID via the pod-resources API.** From a debug pod mounting the kubelet socket (or a node shell), query the gRPC endpoint (e.g. with a small Go/Python client or `crictl`-adjacent tooling):
```
$ podresources List | jq '.pod_resources[]
    | select(.name=="mig-consumer") | .containers[].devices'
[{ "resource_name": "nvidia.com/mig-1g.5gb",
   "device_ids": ["MIG-3e1d..."] }]
```
Now you have `mig-consumer (team=search) → MIG-3e1d...`, joinable to DCGM's per-instance metrics for cost.

**6. Deliberately trigger the reconfig error, then recover.** With the pod still holding the slice, try to change geometry *without* draining:
```bash
$ kubectl label node gpu-a100 nvidia.com/mig.config=all-2g.10gb --overwrite
$ kubectl get node gpu-a100 -L nvidia.com/mig.config.state
gpu-a100   failed
$ kubectl -n gpu-operator logs -l app=nvidia-mig-manager --tail=20
... Unable to destroy GPU instance ... In use by another client
```
**Recover:** evict the client, then re-apply.
```bash
$ kubectl delete pod mig-consumer          # release the CUDA context / slice
$ kubectl label node gpu-a100 nvidia.com/mig.config=all-2g.10gb --overwrite
$ kubectl get node gpu-a100 -L nvidia.com/mig.config.state -w
gpu-a100   pending
gpu-a100   success                          # new geometry applied
$ kubectl get node gpu-a100 -o jsonpath='{.status.capacity}' | tr ',' '\n' | grep mig
"nvidia.com/mig-2g.10gb":"3"
```
In production, `mig-manager`'s cordon+drain does this eviction for you; the error appears when a pod escapes the drain (e.g. no controller, PDB block) or you force a geometry change out-of-band.

## Practice

Feeds the deliverable's **per-pod attribution map** and **failure-mode log**.

On a MIG-capable GPU (rent an A100/H100 for one session; otherwise read + demo on a shared cloud MIG node):
1. Set `mig.strategy=mixed` in the `ClusterPolicy`; confirm `mig-manager` is Running.
2. Apply a geometry via the `nvidia.com/mig.config` label (e.g. `all-1g.5gb` on A100, or a `1g.10gb` layout on H100). Record the `nvidia.com/mig.config.state` transition `pending → success`.
3. Verify the node advertises the per-profile resource (`nvidia.com/mig-1g.5gb` etc.) and list the MIG device UUIDs with `nvidia-smi -L`.
4. Schedule a labeled pod (`team=<x>`) requesting one slice; from its logs capture the MIG-UUID it received.
5. Query the **pod-resources API** and record the `{pod, container, resource_name, device_ids}` mapping — this is your attribution row.
6. With the pod still running, change the geometry label to force the **"In use by another client"** reconfiguration error; capture `mig.config.state=failed` and the mig-manager log line. Then evict the pod, re-apply, and capture the recovery to `success`.

**Acceptance:**
- A scheduled MIG pod whose **MIG-device UUID is mapped to the pod** via the pod-resources API (the `{pod → resource_name → device_id(UUID)}` row, with the pod's team label), saved to the attribution map.
- The **reconfiguration-error recovery documented**: the exact failure state (`nvidia.com/mig.config.state=failed`) and the "in use by another client" log line, the root cause (a client held a CUDA context so the GI/CI couldn't be destroyed), and the recovery steps (drain/evict the client → re-apply the geometry label → `state=success`), saved to the failure-mode log.

## Self-check

**(a) Why must you drain the node before reconfiguring MIG — what state is torn down?**

**Answer:** Reconfiguration destroys and recreates the GPU Instances and Compute Instances (and, if toggling MIG mode, resets the GPU), which invalidates every existing MIG-device UUID on that card. The driver cannot destroy a GI/CI while a process still holds a CUDA context on it — the refcount is non-zero and it returns "In use by another client." Draining evicts the GPU pods so no context is open; then `mig-manager` can tear down the CIs/GIs, apply the new geometry, and let the device plugin re-advertise the new slices. Skipping the drain is exactly what produces the failure.

**(b) Single vs mixed MIG strategy — how does each appear in node capacity?**

**Answer:** **`single`** requires every GPU on the node to have the *same* geometry and reports all slices under the generic `nvidia.com/gpu` name (e.g. 8 GPUs × 7 slices → `nvidia.com/gpu: 56`); workloads request plain `nvidia.com/gpu` and the profile is hidden. **`mixed`** advertises each distinct profile under its own resource name (`nvidia.com/mig-1g.5gb: 21`, `nvidia.com/mig-2g.10gb: 6`, …), allows different slice sizes on one GPU/node, and requires pods to request the exact profile. Mixed is preferred for multi-tenant/billing because the resource name itself encodes the profile.

**(c) How does a MIG slice surface to the scheduler, and why does it give clean per-slice attribution where time-slicing doesn't?**

**Answer:** In `mixed` strategy each slice is advertised as an extended resource (`nvidia.com/mig-<profile>`), and when a pod requests one the device plugin allocates a *specific MIG device* and injects its **unique MIG-device UUID**. The kubelet pod-resources API exposes that `{container → resource_name → device UUID}` mapping, and DCGM-exporter emits per-GI/CI metrics keyed by the same UUID — so you can join pod → slice UUID → team → utilization → cost with hard isolation guaranteeing the numbers are real. Time-slicing instead injects the *same physical GPU* (same parent UUID) into every co-scheduled pod; the scheduler over-commits one device and DCGM sees only that one device, so all tenants share an indistinguishable UUID and utilization figure — there is no per-tenant signal to attribute. MIG's distinct-UUID-per-slice closes that attribution hole.

## Resources

1. **NVIDIA MIG User Guide** (profiles per SKU, GI/CI model, `nvidia-smi mig` reconfiguration, the "in use by another client" behavior): https://docs.nvidia.com/datacenter/tesla/mig-user-guide/
2. **GPU Operator — GPU Sharing (MIG section)** (`mig.strategy` single vs mixed, resource-name surfacing, mig-manager): https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html
3. **GPU Operator — MIG Support / `mig-parted` configuration** (`nvidia.com/mig.config` label, named geometries, `mig.config.state`): https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html
