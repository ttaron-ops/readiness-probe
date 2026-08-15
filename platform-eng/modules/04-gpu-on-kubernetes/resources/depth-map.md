# Depth map — Module 04 · GPU on Kubernetes

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **The module that gained the most.** The source's `k8s-learn/gpu-platform-tasks.md` seeded this
> module's [fake GPU fleet lab](../practice/fake-gpu-fleet/README.md) — the thing that makes
> Modules 04, 05, 06 and 11 buildable without renting hardware. Start there, not here.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 GPU Operator components | [`kubernetes/10-kubelet-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/10-kubelet-internals.md) | the **Device Manager** — the kubelet side of the device-plugin contract the Operator plugs into |
| 02 Crash-loop diagnosis | [`kubernetes/11-pod-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/11-pod-internals.md) | pod lifecycle, init containers, restart backoff — the state machine you're reading when a driver DaemonSet loops |
| 03 Device plugin & pod-resources API | [`kubernetes/10-kubelet-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/10-kubelet-internals.md) | registration, `ListAndWatch`, `Allocate`, and the pod-resources gRPC socket your attributor calls |
| 04 Container-runtime integration (CDI) | [`kubernetes/01-container-runtimes-cri-oci`](https://github.com/harut8/system-design/blob/main/kubernetes/01-container-runtimes-cri-oci.md) | CRI/OCI, containerd, runc hooks — where CDI device injection actually happens and why "CUDA driver insufficient" appears |
| 04 Container-runtime integration (CDI) | [`kubernetes/02-container-images-and-registries`](https://github.com/harut8/system-design/blob/main/kubernetes/02-container-images-and-registries.md) | layers and registries — relevant to multi-GB CUDA image pull time, which is Module 07's cold-start problem |
| 05 Driver lifecycle & fleet upgrades | [`kubernetes/32-cluster-lifecycle-and-day2`](https://github.com/harut8/system-design/blob/main/kubernetes/32-cluster-lifecycle-and-day2.md) | node drain, surge, and rolling-upgrade orchestration across a fleet |
| 06–08 MIG / time-slicing / MPS | [`kubernetes/21-resource-management-and-qos`](https://github.com/harut8/system-design/blob/main/kubernetes/21-resource-management-and-qos.md) | extended resources, QoS, and why a shared device breaks the accounting model |
| 06–08 MIG / time-slicing / MPS | [`gpu-observability/13-multi-tenant-gpu-observability`](https://github.com/harut8/system-design/blob/main/gpu-observability/13-multi-tenant-gpu-observability.md) | the observability consequence of each sharing mode — the other half of the attribution hole |
| 09 DRA driver & quotas | [`kubernetes/25-multi-tenancy`](https://github.com/harut8/system-design/blob/main/kubernetes/25-multi-tenancy.md) | namespace quota, hierarchical tenancy, and where the isolation boundaries actually are |
| 09 DRA driver & quotas | [`kubernetes/23-crds-operators-and-controller-runtime`](https://github.com/harut8/system-design/blob/main/kubernetes/23-crds-operators-and-controller-runtime.md) | ResourceClaim is a CRD-shaped API — the schema-design chapter applies directly |
| 10 Capstone — per-pod attribution | [`k8s-learn/gpu-platform-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/gpu-platform-tasks.md) | Projects 1–3 are the same deliverable with a different framing; steal the acceptance criteria |

## Practice worth stealing

[`k8s-learn/gpu-platform-tasks.md`](https://github.com/harut8/system-design/blob/main/k8s-learn/gpu-platform-tasks.md)
— five projects (telemetry → dashboard → capacity planner CRD → job queue → custom scheduler) that
map almost exactly onto Modules 04/05/06 plus the capstone. Its **Level 0** is the origin of this
module's [fake GPU fleet lab](../practice/fake-gpu-fleet/README.md); its per-project acceptance
checklists are sharper than most and worth folding into your own.

Also: [`k8s-learn/resources-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/resources-tasks.md)
and [`k8s-learn/scheduling-constraints-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/scheduling-constraints-tasks.md)
for requests/limits and placement, both runnable on the fake fleet.
