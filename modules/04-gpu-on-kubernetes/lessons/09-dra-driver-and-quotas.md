---
lesson: "04.9"
title: "The NVIDIA DRA driver and GPU quotas"
module: "04"
concept: "The NVIDIA DRA driver and GPU quotas"
status: not-started
est_time: "10h"
artifacts: []
---

# 04.9 · The NVIDIA DRA driver and GPU quotas

> **Concept.** Install a real DRA driver, schedule a GPU through a ResourceClaim, and fence tenants with namespace quotas.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Why this matters

The device plugin gives every pod an opaque integer: `nvidia.com/gpu: 1`. You cannot tell *which* physical
device a pod got, you cannot ask for "a GPU with ≥40GB and NVLink," and two pods time-slicing one card both
report `1` — so your cost model double-counts. Dynamic Resource Allocation (DRA) replaces that integer with a
first-class API object whose **status names the exact device UUID** that was allocated. That single fact is
the foundation of the per-pod attribution you build in the capstone (04.10). Get the driver installed and a
claim scheduled here, or the cost operator has nothing real to join against.

## What's new here

Lesson 02 (Kubernetes-controllers) taught the **object model** — ResourceClaim, ResourceClaimTemplate,
DeviceClass, ResourceSlice — and you wrote a controller against it. This lesson is **install and schedule**:
you deploy the NVIDIA driver that *publishes* ResourceSlices and *allocates* devices, then run a pod that
consumes one. You are the operator now, not the API designer.

Two things changed under you since 02, and both are version-sensitive — read the caveats:

- **DRA went GA in Kubernetes 1.34** (Sept 2025). The API group is now `resource.k8s.io/v1` (not the
  `v1beta1`/`v1alpha3` you saw in 02). The `DynamicResourceAllocation` feature gate is on by default at GA and
  **locks on (always-enabled) in 1.35**.
- **The NVIDIA driver was donated to CNCF.** It began life as `NVIDIA/k8s-dra-driver-gpu`; from **v0.4.0** it
  moved to **`kubernetes-sigs/dra-driver-nvidia-gpu`**, adopted semver, and the Helm chart was renamed
  `nvidia-dra-driver-gpu` → `dra-driver-nvidia-gpu`. NVIDIA's **GPU Operator 25.8** also folds the DRA driver
  in-tree, so on a GPU-Operator cluster you may not install the standalone chart at all.

> ⚠️ **VERSION CAVEAT — do not skim this.** **Kubernetes 1.34.0 and 1.34.1 ship DRA bugs.** The kubelet
> mishandles a pod with multiple ResourceClaims when one is already prepared (kubernetes/kubernetes#135901),
> and a goroutine race can double-allocate the same device to two claims. Both are fixed in **1.34.2**
> (released 2025-11-12). **Run Kubernetes 1.33.x, or ≥1.34.2 — never 1.34.0/1.34.1 for DRA.** Check with
> `kubectl version` before you do anything else.

### What DRA gives you structurally over the device plugin

| | Device plugin | DRA |
|---|---|---|
| Request shape | fixed integer `nvidia.com/gpu: 1` | structured claim: request devices by **DeviceClass + attribute selectors** (memory, product, MIG profile, NVLink) |
| What you learn after scheduling | nothing — just a count | `ResourceClaim.status.allocation` names the **exact device (UUID)** bound to the pod |
| Sharing config | cluster-wide ConfigMap (time-slicing/MPS applied to a whole node) | **per-claim** `opaque` config — this pod gets TimeSlicing, that one gets MPS |
| Multi-node fabric | not modelled | **ComputeDomain** orchestrates multi-node NVLink / IMEX channels across nodes |

The attribution win is the second row. With the device plugin you reconstruct the UUID→pod map only via the
pod-resources API (04.3). With DRA the answer is *in the claim status* — attributable by construction.

**ComputeDomain** (`compute-domain.nvidia.com`) is the piece that has no device-plugin equivalent: for
multi-node NVLink (GB200 NVL72 and friends), the driver sets up **IMEX** channels so GPUs across several nodes
share a memory-export fabric. You declare a `ComputeDomain` object; the driver co-schedules the members and
wires the channels. You will not exercise it on a single node, but name-drop it correctly in interviews.

## Core notes

**DeviceClasses the NVIDIA driver publishes.** After install, the driver creates cluster-scoped DeviceClasses
you select against:

- `gpu.nvidia.com` — whole GPUs (and the target for per-claim time-slicing/MPS config)
- `mig.nvidia.com` — MIG instances as individual devices
- `compute-domain.nvidia.com` — ComputeDomain channels for multi-node NVLink/IMEX

```bash
kubectl get deviceclasses
# gpu.nvidia.com, mig.nvidia.com, compute-domain.nvidia.com

# What the driver actually advertised on each node:
kubectl get resourceslices -o wide
```

**Install (standalone chart).** Prereqs: K8s 1.33.x or ≥1.34.2, NVIDIA driver ≥565 on the nodes, and **CDI
enabled** in the container runtime (containerd `enable_cdi = true`). Naming is mid-migration — the NGC chart is
CalVer, the kubernetes-sigs chart is semver:

```bash
# NGC chart (NVIDIA-hosted; use with GPU Operator-managed driver root):
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update
helm install nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
  --version="25.8.0" \
  --create-namespace --namespace nvidia-dra-driver-gpu \
  --set resources.gpus.enabled=true \
  --set nvidiaDriverRoot=/run/nvidia/driver   # omit / set to "/" for host-installed driver

# Verify the driver's node plugin + controller are up and slices exist:
kubectl -n nvidia-dra-driver-gpu get pods
kubectl get resourceslices
```

> ⚠️ On a GPU Operator ≥25.8 cluster, enable DRA through the operator (`--set dra.enabled=true` on the operator
> chart) instead of the standalone chart — running both fights over the driver root.

**Schedule a GPU via a claim.** The GA (`resource.k8s.io/v1`) request shape wraps the selector in `exactly:`.
A `ResourceClaimTemplate` mints a fresh, pod-scoped ResourceClaim per pod (the right default — a bare
`ResourceClaim` is shared and outlives pods):

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata: { namespace: gpu-test1, name: single-gpu }
spec:
  spec:
    devices:
      requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
---
apiVersion: v1
kind: Pod
metadata: { namespace: gpu-test1, name: pod1 }
spec:
  containers:
  - name: ctr
    image: ubuntu:22.04
    command: ["bash","-c","nvidia-smi -L; sleep 9999"]
    resources:
      claims:
      - name: gpu                    # container opts into the pod-level claim
  resourceClaims:
  - name: gpu
    resourceClaimTemplateName: single-gpu
  tolerations:
  - { key: nvidia.com/gpu, operator: Exists, effect: NoSchedule }
```

**Read back the allocated device** — this is the attribution payoff:

```bash
kubectl -n gpu-test1 get resourceclaims
# NAME              STATE                AGE
# pod1-gpu-xxxxx    allocated,reserved   10s

kubectl -n gpu-test1 get resourceclaim pod1-gpu-xxxxx -o yaml | yq '.status.allocation.devices.results'
# - device: gpu-0            driver: gpu.nvidia.com   pool: <node>
#   ...  the concrete device the scheduler picked
kubectl -n gpu-test1 logs pod1 | grep UUID    # nvidia-smi -L prints GPU UUID GPU-xxxx
```

You now have `namespace/pod → device UUID` **from the API**, cross-checked against `nvidia-smi -L` inside the
pod. Same fact the pod-resources API gives you (04.3) — but declarative and structured.

**Per-claim sharing config.** Time-slicing/MPS is set on the *claim*, not a node ConfigMap — so tenancy is
per-workload. The `config.opaque` block carries a `GpuConfig`:

```yaml
    devices:
      requests:
      - { name: ts-gpu, exactly: { deviceClassName: gpu.nvidia.com } }
      config:
      - requests: ["ts-gpu"]
        opaque:
          driver: gpu.nvidia.com
          parameters:
            apiVersion: resource.nvidia.com/v1beta1
            kind: GpuConfig
            sharing: { strategy: TimeSlicing, timeSlicingConfig: { interval: Long } }
```

### Multi-tenancy: quotas on top of claims

Scheduling is not isolation. Without a quota, one namespace drains every GPU. Two mechanisms:

**1. `ResourceQuota` for device-plugin GPUs** (still the common case). Cap the extended resource per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { namespace: team-a, name: gpu-quota }
spec:
  hard:
    requests.nvidia.com/gpu: "4"
    limits.nvidia.com/gpu: "4"
```

For **MIG under the mixed strategy** the resource names are per-profile, so quota them individually — a namespace
can be allowed slices but not whole GPUs:

```yaml
  hard:
    requests.nvidia.com/mig-1g.5gb: "8"
    requests.nvidia.com/mig-3g.20gb: "2"
```

**2. Quota for DRA claims.** ⚠️ Core `ResourceQuota` at GA does **not** sum allocated devices by attribute — it
counts objects. Fence DRA tenants by claim count:

```yaml
  hard:
    count/resourceclaims.resource.k8s.io: "8"
```

Finer "at most 4 GPUs of devices in this namespace" limits for DRA are still maturing upstream; today you pair
claim-count quota with a `ValidatingAdmissionPolicy` that rejects claims requesting more than N devices.

**`LimitRange` for defaults.** So a pod that forgets to request a GPU isn't silently CPU-only (or so a claimless
pod can't sneak onto a GPU node), set namespace defaults/max:

```yaml
apiVersion: v1
kind: LimitRange
metadata: { namespace: team-a, name: gpu-defaults }
spec:
  limits:
  - type: Container
    max: { nvidia.com/gpu: "1" }        # no single container grabs >1
    default: { nvidia.com/gpu: "0" }    # default to none unless asked
```

## Worked example

Cluster on **1.33.4** (deliberately dodging 1.34.0/.1). Goal: one claim-scheduled pod, one enforced quota.

```bash
kubectl version | grep Server                       # Server v1.33.4 — safe
helm install nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
  --version 25.8.0 --create-namespace -n nvidia-dra-driver-gpu \
  --set resources.gpus.enabled=true
kubectl get deviceclasses                            # gpu.nvidia.com present
kubectl apply -f gpu-test1.yaml                      # template + pod1 above
kubectl -n gpu-test1 wait --for=condition=Ready pod/pod1 --timeout=120s
kubectl -n gpu-test1 get resourceclaim -o jsonpath='{.items[0].status.allocation.devices.results[0].device}'
# -> gpu-0        the device is named in status. Attribution: solved.

# Now prove the quota. team-a may have 1 GPU:
kubectl create ns team-a
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata: { namespace: team-a, name: gpu-quota }
spec: { hard: { requests.nvidia.com/gpu: "1", limits.nvidia.com/gpu: "1" } }
EOF
# First GPU pod: admitted. Second, over quota: rejected at admission —
kubectl -n team-a apply -f two-gpu-pods.yaml
# Error from server (Forbidden): exceeded quota: gpu-quota,
#   requested: requests.nvidia.com/gpu=1, used: 1, limited: 1
kubectl -n team-a describe quota gpu-quota           # used 1/1
```

The rejection happens at **admission**, before scheduling — the second pod never appears as Pending; the API
server refuses it. That is the enforcement proof.

## Practice

**Feeds the deliverable** ([../practice/per-pod-attribution/README.md](../practice/per-pod-attribution/README.md)).
On a cluster running **K8s 1.33.x or ≥1.34.2** (verify with `kubectl version` and record it):

1. Install the NVIDIA DRA driver (standalone chart, or via GPU Operator ≥25.8). Confirm `kubectl get
   deviceclasses` shows `gpu.nvidia.com` and `kubectl get resourceslices` is non-empty.
2. Schedule a pod through a `ResourceClaimTemplate`. Capture the allocated device three ways: the claim's
   `status.allocation.devices.results`, `nvidia-smi -L` from inside the pod, and the pod-resources API client
   from 04.3. Confirm all three name the same UUID.
3. Apply a `ResourceQuota` capping `nvidia.com/gpu` in a tenant namespace. Submit a pod over the cap and
   **capture the `Forbidden … exceeded quota` error**.

**Acceptance:** a committed note under `practice/per-pod-attribution/` containing (a) the K8s version you ran
and why it's safe, (b) the claim YAML + the `status.allocation` device UUID, (c) the three-way UUID cross-check,
and (d) the verbatim quota-rejection error. That claim→UUID mapping is the seed the capstone joins to DCGM.

## Self-check

**(a) ResourceClaim vs a device-plugin `nvidia.com/gpu` request — what's structurally better for cost
attribution?**
**Answer:** The device plugin hands the pod an opaque integer; to learn which physical device it got you must
separately poll the pod-resources API. A DRA `ResourceClaim` records the concrete allocation in
`status.allocation.devices.results` — the **device (UUID) is named in the API object bound to the pod**, so the
namespace/pod→device mapping is attributable by construction. You also request by attribute (memory, MIG
profile, NVLink) and attach per-claim sharing config, so a shared device carries per-workload intent instead of
a node-wide ConfigMap.

**(b) The Kubernetes 1.34 DRA version caveat — what is it and what do you run?**
**Answer:** DRA went GA in 1.34, but **1.34.0 and 1.34.1 ship DRA bugs** — the kubelet mishandles a pod with
multiple ResourceClaims when one is already prepared (k/k#135901), and a goroutine race can double-allocate a
device to two claims. Both are fixed in **1.34.2** (2025-11-12). **Run 1.33.x, or ≥1.34.2 — never 1.34.0/1.34.1.**
The `DynamicResourceAllocation` gate is on by default at GA and locks on in 1.35.

**(c) How does a ResourceQuota interact with MIG-profile resources and DRA claims?**
**Answer:** For device-plugin GPUs, quota keys are the extended-resource names — `requests.nvidia.com/gpu`, and
under the MIG *mixed* strategy the per-profile names like `requests.nvidia.com/mig-1g.5gb` — so you can grant a
namespace slices but not whole cards. For **DRA**, core ResourceQuota at GA does **not** sum devices by
attribute; it counts objects, so you fence tenants with `count/resourceclaims.resource.k8s.io` and add a
`ValidatingAdmissionPolicy` for per-claim device caps. Enforcement is at admission, before scheduling.

## Resources

1. **NVIDIA DRA driver** — `github.com/NVIDIA/k8s-dra-driver-gpu` (now
   `kubernetes-sigs/dra-driver-nvidia-gpu`) and the install site
   `dra-driver-nvidia-gpu.sigs.k8s.io/docs/install/`. The demo `demo/specs/quickstart/v1/` YAMLs are the
   canonical claim/config examples.
2. **Kubernetes DRA docs** — https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/
   and the v1.34 GA blog https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/ (API group,
   gates, GA status).
3. **ResourceQuota** — https://kubernetes.io/docs/concepts/policy/resource-quotas/ · AKS "DRA, devices, and
   drivers" — https://blog.aks.azure.com/2025/11/17/dra-devices-and-drivers-on-kubernetes (managed-cluster
   version guidance).
