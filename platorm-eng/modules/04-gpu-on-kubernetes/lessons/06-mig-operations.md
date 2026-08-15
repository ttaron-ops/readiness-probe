---
lesson: "04.6"
title: "MIG Operations & Per-Slice Attribution"
module: "04"
concept: "MIG operations & per-slice attribution"
status: not-started
est_time: "10h"
prev: "05-driver-lifecycle-upgrades.md"
next: "07-time-slicing-attribution.md"
artifacts: []
sources: 8
---

# 04.6 · MIG Operations & Per-Slice Attribution

> **Concept.** On Kubernetes, MIG is a resource-surfacing and drain-to-reconfigure problem — each slice becomes a distinct schedulable resource with its own UUID, which is what gives you clean per-slice cost attribution.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lesson 05 established the general pattern the GPU Operator uses for any disruptive per-node change: cordon, drain, mutate, validate, uncordon — applied there to swapping a kernel module. This lesson applies the *exact same* pattern one layer lower in the stack: instead of tearing down a loaded kernel module, you're tearing down and recreating GPU Instances and Compute Instances — the hardware-level partition objects a MIG-enabled GPU exposes. It's a narrower operation than a driver upgrade (one GPU, not a fleet), but higher stakes for correctness, because getting `mig.strategy` wrong doesn't just risk downtime — it silently breaks the cost-attribution story this whole module builds toward. This is also the first lesson in the module where "attribution" stops being aspirational: MIG is the one sharing mode where the Kubernetes-visible resource model gives you a genuinely clean answer to "who used this GPU," which sets the baseline the next two lessons (time-slicing, MPS) are measured against.

## Why this matters

You already know, from module 03, what MIG *is* at the silicon level: a GPU partitioned into isolated instances, each with a fenced slice of SMs, L2 cache, memory, and its own memory-bandwidth path. That isolation is the whole point for a platform team, but it's operationally worthless until Kubernetes can *see* the slices and *attribute* them — which is a Kubernetes-and-operator problem, not a silicon one, and it's the problem this lesson solves.

The reason this lands you a senior GPU-platform role is **cost/observability**, and the module README names both interview probes directly: *"MIG vs time-slicing vs MPS — mechanism/isolation/when"* and *"attribute cost to a time-sliced GPU"* — described as "your home turf." The competing way to share a GPU is **time-slicing** (lesson 07), where N pods all get injected the *same* physical GPU and the *same* `nvidia.com/gpu` device UUID; the scheduler thinks it handed out one card N times. When finance asks "which team burned that H100 last month," time-slicing has no answer — every pod shows the same UUID and the same fully-utilized device. That is the *attribution hole* the next lesson names explicitly.

MIG closes it. Each slice surfaces as its *own* resource name (`nvidia.com/mig-1g.10gb`) backed by its *own* MIG-device UUID (`MIG-GPU-xxxx/gi/ci`). The pod-resources API tells you exactly which slice UUID went to which pod, DCGM emits per-MIG-instance utilization and memory metrics, and you can join `pod → MIG-UUID → namespace/team → cost`. That join is the deliverable of this whole module. Time-slicing gives you density; MIG gives you density *plus* a clean invoice. Knowing when and how to run MIG on Kubernetes — and being able to explain the attribution difference in one breath, cold, in an interview — is the differentiator.

But MIG on Kubernetes has a sharp operational edge: **you cannot reconfigure a GPU's partition layout while anything is using it**. Changing profiles is a drain-and-tear-down operation, and the failure mode ("in use by another client") is one you must be able to trigger and recover on demand — exactly the kind of break/fix competence this module's failure-mode log is built to capture.

## What's new here (calibration)

Module 03 already owns MIG's hardware theory, and module 02 already owns the device-plugin gRPC mechanics — this lesson does not re-derive either:

- **A profile's SM/memory fraction and hardware isolation guarantees** (module 03) — assumed known; this lesson never re-derives why `2g.20gb` gets you two SM slices instead of a bandwidth number.
- **Device-plugin `Allocate()`/`ListAndWatch()` internals** (module 02) — assumed known; this lesson treats the device plugin as the thing that turns a MIG geometry into schedulable resources, not as a gRPC contract to re-explain.

What's genuinely new here: `mig-manager` as the DaemonSet that turns a node *label* into a hardware reconfiguration; `mig.strategy` (`single` vs `mixed`) as a **scheduler-visible** decision, not a hardware one — it changes what resource name a pod requests, which is the actual interview trap; the drain-to-reconfigure failure mode and its recovery, as a concrete break/fix skill; the pod-resources-API-to-UUID join that makes per-slice attribution possible; and the hard constraint that **MPS and MIG cannot be combined on the same device** — a real, documented limitation, not a hypothetical edge case.

## Core concepts

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

### `mig.strategy`: single vs mixed — the interview trap

This controls *how the node reports MIG capacity to the scheduler*, and it's the single most common place a MIG answer goes wrong under interview pressure:

- **`single`** — the node exposes MIG slices under the *generic* `nvidia.com/gpu` resource name, and **all GPUs on the node must have the identical MIG geometry**. A node with 8× A100 each split into 7× `1g.5gb` advertises `nvidia.com/gpu: 56`. Pros: workloads request plain `nvidia.com/gpu: 1` and don't need to know about MIG; scheduling is homogeneous. Cons: you cannot mix profiles on a node, and the *resource name hides the profile* — from the scheduler's point of view, a `1g.5gb` slice and a full 80GB GPU on a non-MIG node both look like `nvidia.com/gpu: 1`, which is exactly the kind of silent capacity misrepresentation that produces "why did my job OOM on a GPU the scheduler said had room" tickets.
- **`mixed`** — each distinct profile is advertised under its **own** resource name: `nvidia.com/mig-1g.5gb: 7`, `nvidia.com/mig-2g.10gb: 3`, etc. A single GPU can host *different-sized* slices, and a node can host different geometries per GPU. Pods request the exact slice: `nvidia.com/mig-1g.10gb: 1`. Pros: heterogeneous slices, explicit profile in the request, and the resource name itself is an attribution key. Cons: the scheduler needs the right resource names; capacity is fragmented across resource types.

For a multi-tenant platform with mixed workloads and per-team billing, **mixed** is almost always the answer — the per-profile resource name is half of your attribution story; the pod-resources-to-UUID join (below) is the other half.

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

`mig-manager`'s reconfiguration therefore drains GPU clients first — the same cordon/drain vocabulary from lesson 05, applied to hardware partition state instead of a kernel module. What gets torn down and why the drain is mandatory:
- **Running CUDA contexts** hold the GPU/MIG device open (non-zero refcount); the GI/CI can't be destroyed under them — this is the identical refcount fact from lesson 05's driver-restart story, one layer lower.
- **The GI/CI objects themselves** are deleted and recreated — every existing MIG-device UUID on that GPU becomes invalid. Any pod that was scheduled onto an old slice UUID now points at nothing.
- **Device-plugin advertisements** must be regenerated; the kubelet's device manager re-reads capacity.

So the sequence is: cordon → evict/complete GPU pods on the node (drain) → `mig-manager` destroys CIs then GIs, sets mode, creates new GIs/CIs → device plugin re-advertises → uncordon. The Operator gates this with a node label so you can watch it. Skipping the drain is exactly what produces the "in use by another client" error.

### Per-slice attribution: the pod-resources API + UUID mapping

This is the payoff. When a pod requests `nvidia.com/mig-1g.10gb: 1`, the device plugin allocates a specific MIG device and injects its UUID as `NVIDIA_VISIBLE_DEVICES`. You recover the pod→UUID mapping from the **kubelet pod-resources gRPC API** (`/var/lib/kubelet/pod-resources/kubelet.sock`), which lists, per container, the `resource_name` and the allocated `device_ids` (the MIG-device UUIDs). DCGM-exporter labels its metrics with the same GPU/GI/CI identity, so you can join:

```
pod (namespace/team) → container → MIG-device UUID → DCGM per-instance util/mem → cost
```

That join is impossible under time-slicing, where every co-scheduled pod gets the *same* parent GPU UUID and DCGM only sees one device — you can measure the card's total utilization but cannot split it per tenant. MIG's distinct-UUID-per-slice is precisely what makes the attribution clean, and it's the reason MIG is the baseline this module's capstone measures the time-slicing and MPS fallback paths against.

### The constraint that trips up multi-tenant designs: MPS and MIG do not combine

A design that looks appealing on paper — MIG for hard memory isolation between tenants, plus MPS *within* a MIG slice for finer-grained compute sharing among a tenant's own jobs — does not exist as a supported configuration. The NVIDIA k8s-device-plugin's own documentation states MPS sharing is **explicitly unsupported on MIG-enabled devices** (and MPS support itself is marked experimental as of device-plugin v0.15.0+). If your multi-tenancy design assumes you can layer MPS underneath a MIG slice for extra density, that assumption is wrong before you write a line of YAML — pick one sharing mode per physical device, and if you need both patterns, split it at the fleet level (some GPUs run MIG, others run MPS), not within one device.

## Perspectives

**Hardware/silicon (recap).** Module 03 already covered *why* a GI/CI is isolated — fenced SMs, a fenced L2 slice, a fenced memory-bandwidth path, enforced by the GPU's own partitioning hardware, not by software convention. Nothing here re-derives that. What's new is entirely about how Kubernetes *sees* and *reconfigures* that hardware fact: a profile becomes a ConfigMap key, an instance becomes a schedulable resource with a UUID, and reconfiguring means "ask mig-manager to tear it down and rebuild it," not "flip a silicon switch."

**Multi-tenant platform operator.** `mixed` strategy plus per-profile resource quota is the literal mechanism a GPU cloud uses to sell "1g.10gb slices" as a billable SKU: `ResourceQuota` objects scoped to `nvidia.com/mig-1g.10gb` give you per-namespace/per-team caps on a specific slice size, the same way CPU/memory quota works today. This is the operational payoff of `mixed` over `single` — you can't build a per-SKU quota story on top of a resource name (`nvidia.com/gpu`) that hides the profile.

**Cost/attribution.** The distinct-UUID-per-slice design choice is the single fact that makes MIG's attribution "solved" while every other sharing mode on this module's syllabus has to work around a shared-UUID limitation. It's the through-line connecting this lesson to lesson 07 (time-slicing's attribution hole), lesson 08 (MPS, same hole, different mechanism), and lesson 10 (the capstone, which treats MIG as the clean 1:1 case and everything else as needing a fallback join strategy).

**Managed-GPU-cloud reality.** Some GPU clouds let tenants configure their own MIG geometry inside a tenant-scoped slice of a shared cluster. CoreWeave has been described, in vendor blog material, as offering tenant-configurable MIG profiles on a vCluster-based multi-tenancy architecture — **this is not independently fetched or confirmed this session**; treat it as a directional data point about how a real neocloud might structure MIG multi-tenancy, not a verified mechanism, and re-check before quoting it as fact. The useful takeaway regardless of the specific vendor detail: MIG's clean per-slice resource model is exactly what makes it plausible to expose slice configuration to a tenant at all — you can't safely delegate "pick your own partition layout" to a tenant on a sharing mode where the scheduler can't even tell slices apart.

## Real-world use cases

No independently confirmed tier-1 company engineering blog narrating a MIG-in-production rollout with real utilization/cost numbers turned up in this session's research — worth stating plainly rather than reaching for a weak citation. In its place, this section leans on the primary vendor source and one directly confirmed operational constraint:

- **[NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)** — the canonical reference this whole lesson's profile tables and GI/CI model trace back to; not independently fetched this session (proxy-blocked), but it is the document the entire ecosystem (Operator docs, device-plugin behavior, third-party MIG tooling) is built against. What it shows: in the absence of a confirmed production case study, this is the actual source of truth practitioners use — treat it as this lesson's primary reference, not a substitute for one.
- **[NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)** — fetched this session, confirmed at v0.17.1. What it shows: a directly-verified, real operational constraint (MPS unsupported on MIG-enabled devices, MPS itself experimental since v0.15.0) that would otherwise be easy to design around incorrectly — the strongest piece of "real-world" ground truth this lesson has, even though it's a repository rather than a blog post.
- **CoreWeave's reported vCluster-based tenant MIG configuration** — mentioned in vendor blog material found via search (`vcluster.com` blog); **not independently fetched or pinned to a specific URL this session** — flagged here explicitly as unverified and directional only, not cited as a fact in the References section below.

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
In production, `mig-manager`'s cordon+drain does this eviction for you; the error appears when a pod escapes the drain (e.g. no controller, PDB block) or you force a geometry change out-of-band — the same class of failure as a stuck `drain-required` state in lesson 05's driver upgrade, just one layer lower.

## Practice

Feeds the deliverable's **per-pod attribution map** and **failure-mode log**, both part of [Per-pod GPU attribution](../practice/per-pod-attribution/README.md).

On a MIG-capable GPU (rent an A100/H100 for one session; otherwise read + demo on a shared cloud MIG node):
1. Set `mig.strategy=mixed` in the `ClusterPolicy`; confirm `mig-manager` is Running.
2. Apply a geometry via the `nvidia.com/mig.config` label (e.g. `all-1g.5gb` on A100, or a `1g.10gb` layout on H100). Record the `nvidia.com/mig.config.state` transition `pending → success`.
3. Verify the node advertises the per-profile resource (`nvidia.com/mig-1g.5gb` etc.) and list the MIG device UUIDs with `nvidia-smi -L`.
4. Schedule a labeled pod (`team=<x>`) requesting one slice; from its logs capture the MIG-UUID it received.
5. Query the **pod-resources API** and record the `{pod, container, resource_name, device_ids}` mapping — this is your attribution row.
6. With the pod still running, change the geometry label to force the **"In use by another client"** reconfiguration error; capture `mig.config.state=failed` and the mig-manager log line. Then evict the pod, re-apply, and capture the recovery to `success`.
7. Write one paragraph in your failure-mode log explaining why MPS could not be layered under this slice even if you wanted finer-grained sharing within the tenant — cite the device-plugin constraint, not a guess.

**Acceptance:**
- A scheduled MIG pod whose **MIG-device UUID is mapped to the pod** via the pod-resources API (the `{pod → resource_name → device_id(UUID)}` row, with the pod's team label), saved to the attribution map.
- The **reconfiguration-error recovery documented**: the exact failure state (`nvidia.com/mig.config.state=failed`) and the "in use by another client" log line, the root cause (a client held a CUDA context so the GI/CI couldn't be destroyed), and the recovery steps (drain/evict the client → re-apply the geometry label → `state=success`), saved to the failure-mode log.
- The MPS-on-MIG constraint paragraph, cited from the device-plugin source, saved alongside the other entries.

## Common pitfalls

1. **Assuming MIG reconfiguration can happen live while pods are running.** It always requires draining GPU clients first (GI/CI teardown, possibly a full GPU reset); the "in use by another client" error is the *expected* failure mode when you skip this, not a bug.
2. **Confusing `single` vs `mixed` strategy behavior.** Believing you can mix profiles per GPU under `single` (you can't — all GPUs on the node must match), or that plain `nvidia.com/gpu` requests still see the per-profile split under `mixed` (they don't — `mixed` requires requesting the exact `nvidia.com/mig-<profile>` name). Get this backward and either scheduling breaks or the profile is invisible to the resource name, which quietly kills your attribution story.
3. **Treating time-sliced and MIG UUIDs as equivalent for billing.** Only MIG issues a distinct UUID per slice. Time-sliced pods (lesson 07) share the parent GPU's UUID, so the pod-resources-to-DCGM join that works cleanly here does not work there at all — don't design a single attribution pipeline assuming both sharing modes behave the same.
4. **Assuming MPS can be layered under a MIG slice for extra density.** The NVIDIA k8s-device-plugin explicitly does not support MPS on MIG-enabled devices (confirmed in the device-plugin README, v0.15.0+). Pick one sharing mode per physical device — if a design needs both patterns, split it at the fleet level, not within one GPU.
5. **Not distinguishing first-time MIG mode enablement from a later profile change.** Turning MIG mode on for the first time on a GPU forces a full GPU reset regardless of geometry; a subsequent geometry change on an already-MIG-enabled GPU tears down and recreates GIs/CIs but doesn't necessarily require the same reset. Budget more disruption time for the first enablement than for routine reprofiling.

## Self-check

- Why must you drain the node before reconfiguring MIG — what state is torn down? **Answer:** Reconfiguration destroys and recreates the GPU Instances and Compute Instances (and, if toggling MIG mode, resets the GPU), which invalidates every existing MIG-device UUID on that card. The driver cannot destroy a GI/CI while a process still holds a CUDA context on it — the refcount is non-zero and it returns "In use by another client." Draining evicts the GPU pods so no context is open; then `mig-manager` can tear down the CIs/GIs, apply the new geometry, and let the device plugin re-advertise the new slices. Skipping the drain is exactly what produces the failure.
- Single vs mixed MIG strategy — how does each appear in node capacity, and which one supports per-team resource quota by slice size? **Answer:** **`single`** requires every GPU on the node to have the *same* geometry and reports all slices under the generic `nvidia.com/gpu` name (e.g. 8 GPUs × 7 slices → `nvidia.com/gpu: 56`); workloads request plain `nvidia.com/gpu` and the profile is hidden. **`mixed`** advertises each distinct profile under its own resource name (`nvidia.com/mig-1g.5gb: 21`, `nvidia.com/mig-2g.10gb: 6`, …), allows different slice sizes on one GPU/node, and requires pods to request the exact profile. Only `mixed` supports per-profile `ResourceQuota` (e.g. capping `nvidia.com/mig-1g.10gb` per namespace), because `single` collapses every slice size into the same indistinguishable resource name.
- How does a MIG slice surface to the scheduler, and why does it give clean per-slice attribution where time-slicing doesn't? **Answer:** In `mixed` strategy each slice is advertised as an extended resource (`nvidia.com/mig-<profile>`), and when a pod requests one the device plugin allocates a *specific MIG device* and injects its **unique MIG-device UUID**. The kubelet pod-resources API exposes that `{container → resource_name → device UUID}` mapping, and DCGM-exporter emits per-GI/CI metrics keyed by the same UUID — so you can join pod → slice UUID → team → utilization → cost with hard isolation guaranteeing the numbers are real. Time-slicing instead injects the *same physical GPU* (same parent UUID) into every co-scheduled pod; the scheduler over-commits one device and DCGM sees only that one device, so all tenants share an indistinguishable UUID and utilization figure — there is no per-tenant signal to attribute. MIG's distinct-UUID-per-slice closes that attribution hole.
- Can you run MPS inside a MIG slice to get finer-grained sharing within one tenant's workloads? **Answer:** No. The NVIDIA k8s-device-plugin explicitly documents MPS sharing as unsupported on MIG-enabled devices (and MPS itself is marked experimental as of device-plugin v0.15.0+). A design that assumes you can stack MPS underneath a MIG slice for extra density within a tenant is not a supported configuration — if a fleet needs both sharing patterns, it has to split them across different physical GPUs, some running MIG and some running MPS, rather than combining them on one device.

## Connections & what's next

This lesson reused lesson 05's cordon/drain/validate pattern one layer lower — a GPU Instance instead of a kernel module, but the same "cannot mutate while a client holds it open" constraint and the same break/fix discipline (trigger the failure on purpose, read the log line, recover). It sets the clean baseline — distinct UUID per unit of sharing — that lesson 07 (time-slicing) explicitly fails to meet and lesson 08 (MPS) fails to meet differently; the MPS-MIG mutual-exclusion fact learned here becomes a hard constraint again in lesson 08's sharing-mode decision table. Everything here — the pod-resources-to-UUID join, the per-profile resource name as an attribution key — is reused directly by the lesson 10 capstone, which extends this exact pattern to the harder time-sliced case where the clean 1:1 join isn't available.

Next: **[04.7 · Time-slicing and the attribution hole](07-time-slicing-attribution.md)** takes the shared-UUID problem named above and makes it the lesson's whole subject — what happens to cost attribution when Kubernetes and DCGM both see one device where there are really N tenants sharing it.

## References & further reading

**Primary sources**
- NVIDIA MIG User Guide — https://docs.nvidia.com/datacenter/tesla/mig-user-guide/ — not independently fetched this session (proxy-blocked); the canonical profile-table and GI/CI-model reference this lesson is built on — the primary source of record given the lack of a confirmed production case study.
- GPU Operator — GPU Sharing (MIG section) — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html — not independently fetched this session (proxy-blocked); read for `mig.strategy` and resource-name surfacing details.
- GPU Operator — MIG Support / `mig-parted` configuration — https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html — not independently fetched this session (proxy-blocked); read for the `nvidia.com/mig.config` label and named-geometry ConfigMap shape.
- NVIDIA/k8s-device-plugin (repo) — https://github.com/NVIDIA/k8s-device-plugin — fetched this session, confirmed v0.17.1; read for the MPS-on-MIG unsupported constraint and `mig.strategy` implementation details.

**Real-world engineering blogs**
- No independently confirmed tier-1 company blog on MIG-in-production turned up this session — stated explicitly above rather than substituted with a weak citation. If you find one at study time, it belongs here.

**Deeper dives**
- NVIDIA GPU Operator (repo) — https://github.com/NVIDIA/gpu-operator — fetched this session; read the `ClusterPolicy` `mig`/`migManager` field definitions directly from source, cross-referenced with lesson 01 and lesson 05.
- NVIDIA/dcgm-exporter (repo) — https://github.com/NVIDIA/dcgm-exporter — fetched this session; the reference implementation of the UUID-keyed per-instance metric join this lesson's attribution payoff depends on, and the same tool the lesson 10 capstone builds against.
