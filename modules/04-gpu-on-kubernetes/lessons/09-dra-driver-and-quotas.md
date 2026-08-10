---
lesson: "04.9"
title: "The NVIDIA DRA driver and GPU quotas"
module: "04"
concept: "DRA driver install, ResourceClaim scheduling, GPU quotas"
status: not-started
est_time: "12h"
prev: "08-mps-choosing-sharing.md"
next: "10-capstone-per-pod-attribution.md"
artifacts: []
sources: 11
---

# 04.9 · The NVIDIA DRA driver and GPU quotas

> **Concept.** Install a real DRA driver, schedule a GPU through a ResourceClaim, and fence tenants with namespace quotas.
>
> Module: [📦 04 — GPU on Kubernetes](../README.md) · Deliverable: [Per-pod GPU attribution](../practice/per-pod-attribution/README.md)

## Where this fits

Lessons 06–08 spent their whole budget on one problem: the device plugin hands the scheduler an opaque integer, `nvidia.com/gpu: 1`, and that integer is where attribution goes to die. MIG (06) dodges the problem because the hardware itself carves out a distinct UUID per slice — attribution is clean because the physical device *is* partitioned. Time-slicing (07) and MPS (08) don't dodge it — they multiplex several pods onto one UUID, and you found the actual failure in the wild: `dcgm-exporter#642`, a practitioner on Blackwell hardware watching every time-sliced pod report identical GPU utilization because DCGM has no idea which of them is doing the work. Three lessons, one conclusion: **the device-plugin model cannot represent "this specific device, or this fraction of it, went to this specific pod."** It was never designed to.

This lesson is where that gets structurally fixed. Dynamic Resource Allocation (DRA) is not a smarter device plugin — it's a different API family (`resource.k8s.io`) that replaces "count of an opaque resource" with "a claim whose status names the exact device allocated." You already met the object model in module 02 (ResourceClaim, ResourceClaimTemplate, DeviceClass, ResourceSlice) and wrote a controller against it in the abstract. Here you install the real NVIDIA driver that publishes those ResourceSlices and honors those claims, schedule a pod through one, and read back a UUID straight from the Kubernetes API — no pod-resources polling required. Then you fence tenants with quotas, because a working claim scheduler with no quota just lets one namespace drain every GPU in the cluster. What this unlocks: the capstone (04.10) can build its ownership map from claim status instead of only from pod-resources, and can name, precisely, why that's the structural fix for the attribution hole 04.7 exposed.

## Why this matters

DRA is the industry's actual answer to "why can't I bill a GPU precisely," not a research proposal — Google Cloud's DRA writeup for GKE frames it exactly this way: the API exists so device management stops being "an integer plus a pile of node labels and hope" and becomes a first-class scheduling primitive with structured attributes. The Kubernetes 1.35 "AI Conformance" program is expected to require DRA support, which means it stops being an opt-in curiosity and starts being table stakes for any platform claiming AI-workload readiness. For your résumé, "I ran DRA once" is a checkbox; "I can tell you which device UUID a claim allocated, why that's structurally better than pod-resources for attribution, and which Kubernetes patch versions you must avoid because they double-allocate devices" is a staff-level answer that survives a live follow-up question.

The stakes are also concrete and immediate for the capstone. `ResourceClaim.status.allocation.devices.results` puts the exact device in the API object bound to the pod — the same fact 04.3's pod-resources client works to reconstruct, but declarative and event-driven instead of polled. If you skip this lesson, 04.10's DRA-sourced ownership-map path (a nice-to-have optimization in the capstone) becomes a black box you can't reason about in an interview. And the GA-but-still-settling status of DRA is itself an interview-relevant fact: "GA on paper, still maturing operationally" is the honest answer to "should I run DRA in production today," and giving a confidently wrong answer here (either "it's rock solid" or "it's vaporware") reads as someone who hasn't actually run it.

## What's new here (calibration)

Module 02 already taught you the DRA *object model* — what a ResourceClaim, DeviceClass, and ResourceSlice are, and you wrote a controller that reasons about them. This module doesn't re-teach that. What's new here:

- **Installing and operating a real device driver that speaks DRA** — the NVIDIA driver that actually publishes ResourceSlices from real hardware and performs the allocation, not a synthetic object model exercise.
- **The exact version-sensitive facts as of today** — which Kubernetes patch versions are safe, which driver chart is current, and the CalVer/semver renaming history that trips people up (detailed below, precisely, because the module README you'll also touch had this partly wrong).
- **Quota mechanics for a claim-based API** — `ResourceQuota` was built for extended-resource integers; it does not sum DRA devices by attribute, so fencing tenants requires a different combination of primitives than the device-plugin case.
- **ComputeDomain/IMEX** — the multi-node-NVLink concept (GB200 NVL72-class fleets) that has no device-plugin equivalent at all, because the device-plugin model never had a notion of a fabric spanning multiple nodes.

## Core concepts

### The structural upgrade, concretely

| | Device plugin | DRA |
|---|---|---|
| Request shape | fixed integer, `nvidia.com/gpu: 1` | structured claim: request devices by **DeviceClass + attribute selectors** (memory, product, MIG profile, NVLink) |
| What you learn after scheduling | nothing — just a count | `ResourceClaim.status.allocation` names the **exact device (UUID)** bound to the pod |
| Sharing config | cluster-wide ConfigMap (time-slicing/MPS applied to a whole node) | **per-claim** `opaque` config — this pod gets TimeSlicing, that one gets MPS |
| Multi-node fabric | not modelled | **ComputeDomain** orchestrates multi-node NVLink/IMEX channels across nodes |

The row that matters most for your capstone is the second one. With the device plugin you *reconstruct* the UUID→pod map by polling the kubelet's pod-resources socket (04.3) — an indirect, out-of-band join. With DRA the answer is *in the claim object itself*: attributable by construction, and watchable as a normal Kubernetes object instead of polled off a node-local gRPC socket.

### Version-sensitive facts — read this section carefully, verify at study time

This lesson was flagged, correctly, as version-sensitive before it was written, so the facts below were checked against primary sources rather than carried over from an older draft. Some of what an earlier version of this material (and this module's README) claimed was wrong. Here is the corrected state:

**1. DRA's GA status and the 1.34.0/1.34.1 bugs — this part was already right, and it's worth keeping precisely.** DRA's core APIs (`resource.k8s.io/v1`) went **GA in Kubernetes 1.34**, and the `DynamicResourceAllocation` feature gate — on by default at GA — **locks on (cannot be disabled) in 1.35**. But 1.34.0 and 1.34.1 shipped two real, confirmed bugs:

  - **`kubernetes/kubernetes#135901`** — the kubelet mishandles a pod that has multiple ResourceClaims when one of them is already prepared; this was fixed by **PR #136480**.
  - A second, independent bug: a goroutine race that can **double-allocate the same device to two different claims** — fixed by **PR #136566**.

  Both fixes shipped in **Kubernetes 1.34.2, released 2025-11-12**. The rule stands exactly as before: **run 1.33.x, or ≥1.34.2 — never bare 1.34.0 or 1.34.1 for DRA workloads.** `kubectl version` before you do anything else on a new cluster.

**2. "GPU Operator 25.8 folds the DRA driver in-tree" is wrong — here's the actual mechanism.** This claim (which lived in an earlier version of this lesson and in the module README) conflates two *different* version schemes that happen to share a number:

  - The public **GPU Operator** release line never shipped a version called 25.8. It went `v25.3.x → v25.10.0/v25.10.1 → v26.3.0/.1/.2/.3`.
  - The **DRA driver**, before it adopted semantic versioning, used its *own* CalVer-style tags: `v25.3.0/.1/.2`, `v25.8.0/.1`, `v25.12.0`. "25.8" is a *DRA-driver* tag, not a GPU Operator release.

  These got merged into one false claim somewhere along the way — "GPU Operator 25.8" doesn't exist, and even the DRA driver's own `v25.8.x` tags didn't mean "now bundled into the operator." **As of the latest confirmed release, the DRA driver installs as a separate, companion Helm chart alongside the GPU Operator — it is not folded into the operator binary.** Full in-tree integration is described publicly as a *future direction*, not something that has shipped. Treat any claim that a specific Operator version makes the standalone chart unnecessary as suspect until you've checked the Operator's own release notes for that exact version.

**3. The driver repo and chart were renamed and switched to semver.** The DRA driver began life as `NVIDIA/k8s-dra-driver-gpu`. Starting at **v0.4.0**, it moved to **`kubernetes-sigs/dra-driver-nvidia-gpu`** (donated into the `kubernetes-sigs` org) and adopted **semantic versioning** — the CalVer tags above (`v25.3.x`, `v25.8.x`, `v25.12.0`) predate this switch. The Helm chart was renamed to match: `nvidia-dra-driver-gpu` (NGC) → `dra-driver-nvidia-gpu`. The **latest confirmed release at the time this lesson was written is `v0.4.1`** — check `helm search repo` / the repo's release page for what's actually current before you pin a version, because this line is moving.

**4. The driver-version prerequisite has moved and is still moving — don't hard-pin it.** Earlier guidance said the NVIDIA datacenter driver needed to be **≥565** for the DRA driver to work. More recent guidance says **≥580**. Both numbers have been correct at different points in this driver's life, which is itself the lesson: **treat the exact minimum as "≥565 or ≥580 depending on which DRA-driver version you're running — verify against the DRA driver's own release notes at study time,"** not as a number worth memorizing.

**5. ComputeDomain and IMEX — the piece with no device-plugin equivalent at all.** For fleets built around multi-node NVLink (NVIDIA's GB200 NVL72 and successors, where GPUs across *several physical nodes* share a coherent memory fabric), the DRA driver introduces a `compute-domain.nvidia.com` DeviceClass and a `ComputeDomain` object. Declaring one causes the driver to co-schedule the participating pods across nodes and wire up **IMEX** (Internode Memory Exchange) channels so those GPUs can address each other's memory across the NVLink fabric. There is nothing resembling this in the device-plugin model — it has no concept of a resource that spans nodes. NVIDIA's developer-blog writeup on enabling multi-node NVLink on Kubernetes for GB200 NVL72-class systems is the primary public description of this mechanism (cited below; not independently fetched this session, corroborated across multiple independent search results). You will not exercise this on the single node you're renting for this module, but you should be able to describe *why* it exists and *what problem it solves* in an interview — it's the strongest concrete answer to "when does DRA's extra complexity actually pay for itself?"

### DeviceClasses the NVIDIA driver publishes

After install, the driver creates cluster-scoped DeviceClasses you select against:

- `gpu.nvidia.com` — whole GPUs (and the target for per-claim time-slicing/MPS config)
- `mig.nvidia.com` — MIG instances as individual devices
- `compute-domain.nvidia.com` — ComputeDomain channels for multi-node NVLink/IMEX

```bash
kubectl get deviceclasses
# gpu.nvidia.com, mig.nvidia.com, compute-domain.nvidia.com

# What the driver actually advertised on each node:
kubectl get resourceslices -o wide
```

### Install (standalone chart)

Prereqs: Kubernetes **1.33.x, or ≥1.34.2** (never bare 1.34.0/1.34.1), a driver version that satisfies whatever the DRA driver release you're pinning actually requires (**≥565 or ≥580 — check the release notes**), and **CDI enabled** in the container runtime (containerd `enable_cdi = true`; see 04.4). The chart and repo are mid-rename — older docs and search results reference the NGC/CalVer name, current sources reference the semver kubernetes-sigs name:

```bash
# kubernetes-sigs chart (current name, semver — verify the actual latest tag before pinning):
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update
helm install dra-driver-nvidia-gpu nvidia/dra-driver-nvidia-gpu \
  --version="v0.4.1" \
  --create-namespace --namespace dra-driver-nvidia-gpu \
  --set resources.gpus.enabled=true \
  --set nvidiaDriverRoot=/run/nvidia/driver   # omit / set to "/" for host-installed driver

# Verify the driver's node plugin + controller are up and slices exist:
kubectl -n dra-driver-nvidia-gpu get pods
kubectl get resourceslices
```

> ⚠️ On a GPU Operator-managed cluster, check the Operator's own release notes for whether that specific version can enable DRA for you (`--set dra.enabled=true` on the operator chart) versus needing the standalone chart. **Do not assume any given Operator version folds DRA in — as of the latest confirmed release it's still a companion chart either way**; running both against the same driver root will fight over ownership.

### Schedule a GPU via a claim

The GA (`resource.k8s.io/v1`) request shape wraps the selector in `exactly:`. A `ResourceClaimTemplate` mints a fresh, pod-scoped ResourceClaim per pod (the right default — a bare `ResourceClaim` is shared and outlives pods):

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

You now have `namespace/pod → device UUID` **from the API**, cross-checked against `nvidia-smi -L` inside the pod. Same fact the pod-resources API gives you (04.3) — but declarative, structured, and watchable instead of polled.

### Per-claim sharing config

Time-slicing/MPS is set on the *claim*, not a node ConfigMap — so tenancy is per-workload rather than per-node:

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

Note that this config being *per-claim* doesn't remove the attribution hole from 04.7 — a time-sliced claim still means several pods sharing one physical UUID at the DCGM level. What DRA fixes is *who asked for what and got what*; it doesn't fix "what fraction of the shared device's utilization belongs to which pod" — that's still the per-PID fallback problem the capstone solves.

### Multi-tenancy: quotas on top of claims

Scheduling is not isolation. Without a quota, one namespace drains every GPU. Two mechanisms, because DRA and the device plugin need different tools:

**1. `ResourceQuota` for device-plugin GPUs** (still the common case on most fleets today). Cap the extended resource per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { namespace: team-a, name: gpu-quota }
spec:
  hard:
    requests.nvidia.com/gpu: "4"
    limits.nvidia.com/gpu: "4"
```

For **MIG under the mixed strategy** the resource names are per-profile, so quota them individually — a namespace can be allowed slices but not whole GPUs:

```yaml
  hard:
    requests.nvidia.com/mig-1g.5gb: "8"
    requests.nvidia.com/mig-3g.20gb: "2"
```

**2. Quota for DRA claims.** ⚠️ Core `ResourceQuota` at GA does **not** sum allocated devices by attribute — it counts *objects*. Fence DRA tenants by claim count:

```yaml
  hard:
    count/resourceclaims.resource.k8s.io: "8"
```

Finer "at most 4 GPUs of devices in this namespace" limits for DRA are still maturing upstream; today you pair claim-count quota with a `ValidatingAdmissionPolicy` that rejects claims requesting more than N devices. This gap — quota counting objects instead of the attribute you actually care about — is a direct symptom of "GA on paper, still maturing operationally": the scheduling primitive is solid, the surrounding governance tooling hasn't fully caught up yet.

**`LimitRange` for defaults.** So a pod that forgets to request a GPU isn't silently CPU-only (or a claimless pod can't sneak onto a GPU node), set namespace defaults/max:

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

## Perspectives

**API-design/scheduling theory.** The device plugin models "how many," DRA models "which, with what attributes, shared how." That's not an incremental improvement — it's a different abstraction: a claim/attribute model closer to how a cluster scheduler *should* think about heterogeneous, partitionable, shareable hardware. The Filter/Score/Reserve/Bind cycle you learned in module 02's component-internals lesson gains a DRA plugin that allocates specific devices against structured attributes instead of decrementing an integer — the scheduling *mechanism* barely changed, but what it's reasoning about got dramatically richer.

**Multi-tenancy/quota.** The claim-count-quota-plus-ValidatingAdmissionPolicy workaround is the tell that DRA's governance layer is younger than its scheduling layer. Device-plugin quotas are a solved problem — one extended-resource name, one number. DRA quotas require composing two separate primitives to approximate what device-plugin quota did natively, and even then you can't cap "at most 4 GPUs of *any* attribute combination" cleanly. If you're evaluating DRA for a multi-tenant platform today, this is the honest gap to name.

**Fleet/hardware-topology (GB200-class).** ComputeDomain/IMEX is where DRA stops being "a nicer API for the same problem" and starts solving a problem the device plugin literally cannot express: a resource that spans multiple physical nodes. On GB200 NVL72-class systems, the entire point of the hardware is that GPUs across nodes act like one coherent memory domain — a per-node integer resource model has no vocabulary for that. This is the strongest "why does the complexity earn its keep" argument for DRA, and it's worth having ready cold in an interview.

**Migration/adoption reality — "GA on paper, still maturing operationally."** DRA core is GA, the feature gate locks on in 1.35, and Google is building GKE's AI-workload story around it. None of that means most fleets have migrated. The device-plugin model still runs the overwhelming majority of production GPU scheduling today; DRA driver releases are still renaming repos and bumping minimum driver versions between minor releases (see the version-sensitive facts above); and quota tooling hasn't caught up. The honest senior answer to "should we run DRA in prod" in mid-2026 is: it's real, it's not going away, ecosystem maturity still varies enough that you pilot it deliberately rather than flip a fleet wholesale — and you re-verify every version number in this lesson before you do, because this is one of the fastest-moving corners of Kubernetes right now.

## Real-world use cases

- **["Kubernetes device management with DRA (Dynamic Resource Allocation)"](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation) — Google Cloud.** Directly fetched and read this session. The production-platform framing for *why* DRA exists — the claim/attribute model as the fix for device management that used to be "an integer plus node labels" — and ties DRA to GKE's AI-workload roadmap. Read this for the vendor's own justification of the design, not just the API mechanics.
- **[kubernetes/kubernetes#135901](https://github.com/kubernetes/kubernetes/issues/135901) — the real 1.34.0/1.34.1 kubelet multi-claim bug.** Directly fetched and read this session. This is not a hypothetical caveat — it's the actual issue thread describing the kubelet mishandling a pod with multiple ResourceClaims when one is already prepared, fixed by PR #136480 (with a related double-allocation race fixed separately by PR #136566). Read it for what a real, current DRA bug report looks like and how the fix landed.
- **["Enabling Multi-Node NVLink on Kubernetes for NVIDIA GB200 NVL72 and Beyond"](https://developer.nvidia.com/blog/enabling-multi-node-nvlink-on-kubernetes-for-gb200-and-beyond/) — NVIDIA Developer Blog.** *Not independently fetched this session* (proxy-blocked; found via search and corroborated across multiple independent results) — treat as high-confidence but spot-check if you have fetch access. The primary public description of ComputeDomain/IMEX in production-scale multi-node NVLink fleets — the concrete "why DRA's extra complexity earns its keep" case.
- **["Delve into Dynamic Resource Allocation, devices, and drivers on Kubernetes"](https://blog.aks.azure.com/2025/11/17/dra-devices-and-drivers-on-kubernetes) — AKS Engineering Blog.** *Not independently fetched this session* (proxy-blocked; found via search). A second managed-Kubernetes vendor's take on DRA version guidance and operational rollout — useful for triangulating against the Google Cloud post above.

## Worked example

Cluster on **1.33.4** (deliberately dodging 1.34.0/.1). Goal: one claim-scheduled pod, one enforced quota.

```bash
kubectl version | grep Server                       # Server v1.33.4 — safe
helm install dra-driver-nvidia-gpu nvidia/dra-driver-nvidia-gpu \
  --version v0.4.1 --create-namespace -n dra-driver-nvidia-gpu \
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

The rejection happens at **admission**, before scheduling — the second pod never appears as Pending; the API server refuses it. That is the enforcement proof.

Before you run any of this against a real cluster: re-check `kubectl version`, re-check the DRA driver's release notes for its actual current minimum NVIDIA driver version (**≥565 or ≥580 depending on which release you're pinning**), and re-check the chart name and latest tag (`v0.4.1` at the time this lesson was written, almost certainly not the latest by the time you read it).

## Practice

**Feeds the deliverable** ([../practice/per-pod-attribution/README.md](../practice/per-pod-attribution/README.md)). On a cluster running **K8s 1.33.x or ≥1.34.2** (verify with `kubectl version` and record it):

1. Install the NVIDIA DRA driver as a standalone chart (or via the GPU Operator, if you've confirmed *that specific Operator version* supports it in its own release notes — don't assume). Confirm `kubectl get deviceclasses` shows `gpu.nvidia.com` and `kubectl get resourceslices` is non-empty.
2. Schedule a pod through a `ResourceClaimTemplate`. Capture the allocated device three ways: the claim's `status.allocation.devices.results`, `nvidia-smi -L` from inside the pod, and the pod-resources API client from 04.3. Confirm all three name the same UUID.
3. Apply a `ResourceQuota` capping `nvidia.com/gpu` in a tenant namespace. Submit a pod over the cap and **capture the `Forbidden … exceeded quota` error**.

**Acceptance:** a committed note under `practice/per-pod-attribution/` containing (a) the K8s version you ran and why it's safe, (b) the claim YAML + the `status.allocation` device UUID, (c) the three-way UUID cross-check, and (d) the verbatim quota-rejection error. That claim→UUID mapping is the seed the capstone joins to DCGM.

## Common pitfalls

1. **Assuming "GPU Operator 25.8 folds DRA in-tree" means you never need a separate chart.** There is no public GPU Operator 25.8; "25.8" is the DRA driver's own old CalVer tag. As of the latest confirmed release, the DRA driver is a **companion Helm chart**, not part of the Operator binary — check the exact Operator version's release notes rather than assuming.
2. **Hard-pinning the driver-version prerequisite (≥565 or ≥580) as a fixed fact.** It has moved once already and will move again as the DRA driver evolves; check the release notes for the chart version you're actually installing.
3. **Running Kubernetes 1.34.0 or 1.34.1 for DRA workloads.** Both ship the kubelet multi-claim bug (`#135901`) and a device double-allocation race — silent correctness bugs, not crashes, which makes them worse to discover in production. Use 1.33.x or ≥1.34.2.
4. **Expecting `ResourceQuota` to cap DRA devices the way it caps `nvidia.com/gpu`.** It counts claim *objects*, not allocated devices by attribute. A namespace with a `count/resourceclaims.resource.k8s.io: "1"` quota can still request a claim for 8 devices in one object unless you add a `ValidatingAdmissionPolicy`.
5. **Treating DRA as "done" because it's GA.** GA means the API is stable, not that the ecosystem (quota tooling, driver release cadence, documentation naming) has caught up. Verify every version number in this lesson before you rely on it — this corner of Kubernetes moves fast.

## Self-check

- ResourceClaim vs a device-plugin `nvidia.com/gpu` request — what's structurally better for cost attribution? **Answer:** The device plugin hands the pod an opaque integer; to learn which physical device it got you must separately poll the pod-resources API. A DRA `ResourceClaim` records the concrete allocation in `status.allocation.devices.results` — the device (UUID) is named in the API object bound to the pod, so the namespace/pod→device mapping is attributable by construction. You also request by attribute (memory, MIG profile, NVLink) and attach per-claim sharing config, so a shared device carries per-workload intent instead of a node-wide ConfigMap.
- The Kubernetes 1.34 DRA version caveat — what is it, precisely, and what do you run? **Answer:** DRA went GA in 1.34, but 1.34.0 and 1.34.1 ship two real bugs: the kubelet mishandles a pod with multiple ResourceClaims when one is already prepared (`kubernetes/kubernetes#135901`, fixed by PR #136480), and a separate goroutine race can double-allocate the same device to two claims (fixed by PR #136566). Both fixes shipped in 1.34.2 (2025-11-12). Run 1.33.x, or ≥1.34.2 — never bare 1.34.0/1.34.1. The `DynamicResourceAllocation` gate is on by default at GA and locks on (can't be disabled) in 1.35.
- What's wrong with the claim "GPU Operator 25.8 folds the DRA driver in-tree," and what's actually true? **Answer:** There was never a public GPU Operator v25.8 release — the public line went 25.3.x → 25.10.x → 26.x. "25.8" belongs to the DRA driver's *own* old CalVer tag scheme (`v25.3.x`, `v25.8.x`, `v25.12.0`), which it used before switching to semver at v0.4.0 and moving from `NVIDIA/k8s-dra-driver-gpu` to `kubernetes-sigs/dra-driver-nvidia-gpu`. As of the latest confirmed release (v0.4.1), the DRA driver still installs as a separate companion Helm chart alongside the GPU Operator — full in-tree folding is a stated future direction, not something that has shipped.
- How does a ResourceQuota interact with MIG-profile resources and DRA claims? **Answer:** For device-plugin GPUs, quota keys are the extended-resource names — `requests.nvidia.com/gpu`, and under the MIG *mixed* strategy the per-profile names like `requests.nvidia.com/mig-1g.5gb` — so you can grant a namespace slices but not whole cards. For DRA, core ResourceQuota at GA does not sum devices by attribute; it counts objects, so you fence tenants with `count/resourceclaims.resource.k8s.io` and add a `ValidatingAdmissionPolicy` for per-claim device caps. Enforcement is at admission, before scheduling, in both cases.
- What does ComputeDomain/IMEX solve that no device-plugin mechanism can, and when would you actually reach for it? **Answer:** It models a resource that spans multiple physical nodes — GPUs across several machines sharing a coherent NVLink memory fabric (GB200 NVL72-class systems). The device plugin's per-node integer-resource model has no vocabulary for "these devices on different nodes are part of one fabric"; DRA's `compute-domain.nvidia.com` DeviceClass and `ComputeDomain` object let the driver co-schedule participating pods across nodes and wire up IMEX channels between them. You'd reach for it only on multi-node-NVLink hardware — it's the clearest case where DRA's added complexity solves a problem the old model literally cannot express, rather than just doing the old job more elegantly.

## Connections & what's next

This lesson closes the loop the module opened at 06/07/08: MIG's clean UUIDs, time-slicing's attribution hole, and MPS's throughput-vs-blast-radius tradeoff were all symptoms of the device plugin's integer model. DRA is the structural fix for that model, and its claim status becomes an alternate, event-driven source for the capstone's ownership map. Module 02's DRA object-model lesson is the theory this lesson operationalizes; module 02's component-internals lesson (the scheduler's Filter/Score/Reserve/Bind cycle) is the mechanism DRA's scheduling plugin actually runs inside.

Next: **[04.10 · Capstone — per-pod GPU attribution](10-capstone-per-pod-attribution.md)** assembles everything from L1 through this lesson into the module's deliverable — the pod-resources join from 04.3, MIG's clean 1:1 case from 04.6, the time-sliced fallback from 04.7, and this lesson's DRA claim as a second, structurally cleaner path to the same ownership map.

## References & further reading

**Primary sources**
- [Kubernetes documentation — Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) — *not independently fetched this session (proxy-blocked); canonical URL, spot-check if you have fetch access.* Read for the authoritative object-model and API-group reference.
- [Kubernetes blog — DRA updates for v1.34](https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/) — *not independently fetched this session (proxy-blocked).* Read for the official GA announcement and feature-gate detail.
- [kubernetes/kubernetes#135901](https://github.com/kubernetes/kubernetes/issues/135901) — **fetched and read this session.** The real 1.34.0/1.34.1 kubelet multi-claim bug; read for the exact failure description and the linked fix PRs.
- [kubernetes/kubernetes PR #136480](https://github.com/kubernetes/kubernetes/pull/136480) and [PR #136566](https://github.com/kubernetes/kubernetes/pull/136566) — the two fixes that shipped in 1.34.2; read for what actually changed in the kubelet/scheduler.
- [kubernetes-sigs/dra-driver-nvidia-gpu](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu) — **fetched and read this session (latest confirmed release v0.4.1).** The canonical driver source, chart, and release history — check this before pinning any version number in this lesson.

**Real-world engineering blogs**
- Google Cloud, ["Kubernetes device management with DRA (Dynamic Resource Allocation)"](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation) — **fetched and read this session.** The production-platform case for DRA's claim/attribute model, and its role in GKE's AI-workload/conformance story.
- NVIDIA Developer Blog, ["Enabling Multi-Node NVLink on Kubernetes for NVIDIA GB200 NVL72 and Beyond"](https://developer.nvidia.com/blog/enabling-multi-node-nvlink-on-kubernetes-for-gb200-and-beyond/) — *not independently fetched this session (proxy-blocked; corroborated across multiple independent searches).* The primary description of ComputeDomain/IMEX at production multi-node-NVLink scale.
- AKS Engineering Blog, ["Delve into Dynamic Resource Allocation, devices, and drivers on Kubernetes"](https://blog.aks.azure.com/2025/11/17/dra-devices-and-drivers-on-kubernetes) — *not independently fetched this session (proxy-blocked).* A second managed-Kubernetes vendor's DRA rollout and version guidance, useful to triangulate against the Google Cloud post.
- Red Hat Developer, ["Multitenant AI inference with dynamic resource allocation on OpenShift"](https://developers.redhat.com/articles/2026/08/03/multitenant-ai-inference-dynamic-resource-allocation-openshift) — *not independently fetched this session (proxy-blocked).* A third vendor's production multi-tenancy use case for DRA, directly relevant to the quota-gap discussion above.

**Deeper dives**
- [kubernetes/kubernetes CHANGELOG-1.34.md](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.34.md) — cross-check exact PR numbers and dates directly against the changelog before quoting them as fact in a live setting; this is the primary record.
