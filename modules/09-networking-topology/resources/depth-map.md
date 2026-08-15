# Depth map — Module 09 · Networking & topology

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **Good on the Kubernetes network, silent on the fabric.** The source has 2,300–2,800-line
> chapters on CNI, Cilium/eBPF and kube-proxy, but nothing on InfiniBand, RoCE, GPUDirect or SHARP.
> Use it for the layer where RDMA meets Kubernetes, not for RDMA itself.
>
> Overlaps heavily with [`platform/02`'s depth map](../../../platform/02-platform-networking/resources/depth-map.md) —
> read that one first if you're working both.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Intra→inter handoff | [`kubernetes/15-cni-and-pod-networking`](https://github.com/harut8/system-design/blob/main/kubernetes/15-cni-and-pod-networking.md) | the veth/bridge/routing path a packet takes leaving a pod — where the handoff physically happens |
| 04 IB vs RoCE / lossless | [`sre-observability/24-network-observability`](https://github.com/harut8/system-design/blob/main/sre-observability/24-network-observability.md) | what to measure on a fabric: drops, PFC/ECN counters, and how congestion shows up as tail latency |
| 06 K8s multi-NIC | [`kubernetes/15-cni-and-pod-networking`](https://github.com/harut8/system-design/blob/main/kubernetes/15-cni-and-pod-networking.md) | **the key chapter** — the CNI spec, chaining, and how Multus/SR-IOV attach a second interface |
| 06 K8s multi-NIC | [`kubernetes/16-cilium-and-ebpf-deep-dive`](https://github.com/harut8/system-design/blob/main/kubernetes/16-cilium-and-ebpf-deep-dive.md) | eBPF datapath, XDP, and bypassing the kernel network stack — the "what comes after iptables" argument |
| 06 K8s multi-NIC | [`kubernetes/20-network-policy-and-segmentation`](https://github.com/harut8/system-design/blob/main/kubernetes/20-network-policy-and-segmentation.md) | what happens to policy enforcement when a pod has an interface the CNI doesn't manage — a real multi-NIC trap |
| 07 Bandwidth as cost | [`sre-observability/31-finops-for-observability`](https://github.com/harut8/system-design/blob/main/sre-observability/31-finops-for-observability.md) | the general "measure it, price it, attribute it" method, applied to a different resource |

## Also worth a pass

- [`kubernetes/35-performance-scaling-and-tuning`](https://github.com/harut8/system-design/blob/main/kubernetes/35-performance-scaling-and-tuning.md)
  — at 5k–15k nodes, control-plane network behaviour (watch fan-out, endpoint churn) becomes a
  scaling limit in its own right.
- [`sre-observability/22-service-mesh-observability`](https://github.com/harut8/system-design/blob/main/sre-observability/22-service-mesh-observability.md)
  — useful mainly as the argument for why you keep a mesh *off* the training data path.

## Nothing there for

InfiniBand vs RoCEv2, PFC/ECN lossless tuning, GPUDirect RDMA, NVSHMEM, SHARP in-network
reduction, rail-optimised fat trees. Entirely this course's own material.
