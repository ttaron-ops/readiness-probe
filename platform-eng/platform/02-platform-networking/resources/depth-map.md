# Depth map — platform/02 · Platform networking

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **One of the strongest matches in the repo.** Chapters 14–20 of the `kubernetes/` track are
> 2,300–2,800 lines each and cover this module's middle five lessons at a depth that would take a
> week each to work through properly. Read them when the packet-path artifact stalls, not before.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 TCP/IP & the packet path | [`kubernetes/00-linux-primitives-for-containers`](https://github.com/harut8/system-design/blob/main/kubernetes/00-linux-primitives-for-containers.md) | netns, veth pairs, and the kernel plumbing the path is drawn on |
| 01 TCP/IP & the packet path | [`python-mastery/09-syscalls-and-io`](https://github.com/harut8/system-design/blob/main/python-mastery/09-syscalls-and-io.md) | socket syscall costs and the copies between user and kernel space — the per-packet overhead RDMA exists to remove |
| 02 DNS & service discovery | [`kubernetes/18-dns-and-coredns`](https://github.com/harut8/system-design/blob/main/kubernetes/18-dns-and-coredns.md) | CoreDNS plugin chain, `ndots:5` and the search-path amplification that is the classic latency bug |
| 03 Load balancing | [`kubernetes/14-services-and-kube-proxy`](https://github.com/harut8/system-design/blob/main/kubernetes/14-services-and-kube-proxy.md) | iptables vs IPVS vs eBPF datapaths, EndpointSlices, and conntrack behaviour under churn |
| 03 Load balancing | [`kubernetes/17-ingress-gateway-and-service-mesh`](https://github.com/harut8/system-design/blob/main/kubernetes/17-ingress-gateway-and-service-mesh.md) | Ingress → Gateway API migration, and L7 LB placement |
| 04 Cloud networking | [`kubernetes/37-cloud-provider-integration`](https://github.com/harut8/system-design/blob/main/kubernetes/37-cloud-provider-integration.md) | the cloud-controller-manager, LoadBalancer provisioning, and what you inherit on bare metal |
| 05 Kubernetes networking | [`kubernetes/15-cni-and-pod-networking`](https://github.com/harut8/system-design/blob/main/kubernetes/15-cni-and-pod-networking.md) | **the core chapter** — the CNI spec, IPAM, chaining, overlay vs routed |
| 05 Kubernetes networking | [`kubernetes/16-cilium-and-ebpf-deep-dive`](https://github.com/harut8/system-design/blob/main/kubernetes/16-cilium-and-ebpf-deep-dive.md) | 2,800 lines on the eBPF datapath, XDP, and kube-proxy replacement — "what comes after iptables" |
| 05 Kubernetes networking | [`kubernetes/20-network-policy-and-segmentation`](https://github.com/harut8/system-design/blob/main/kubernetes/20-network-policy-and-segmentation.md) | default-deny, policy tiers, and zero-trust east-west enforcement |
| 06 Service mesh | [`kubernetes/17-ingress-gateway-and-service-mesh`](https://github.com/harut8/system-design/blob/main/kubernetes/17-ingress-gateway-and-service-mesh.md) | sidecar vs ambient/sidecarless, mTLS, and the latency tax — which is the argument for keeping a mesh off the GPU data path |
| 06 Service mesh | [`sre-observability/22-service-mesh-observability`](https://github.com/harut8/system-design/blob/main/sre-observability/22-service-mesh-observability.md) | what a mesh actually gives you in signal, so you can price the tax against the benefit |
| 07 GPU & RDMA networking | *(no match)* | the source has nothing on IB/RoCE/GPUDirect — see [Module 09's depth map](../../../modules/09-networking-topology/resources/depth-map.md) for the closest adjacent chapters |
| 08 Network observability & debugging | [`sre-observability/24-network-observability`](https://github.com/harut8/system-design/blob/main/sre-observability/24-network-observability.md) | flow logs, drops, PFC/ECN counters, and the metrics that make congestion visible |
| 08 Network observability & debugging | [`kubernetes/16-cilium-and-ebpf-deep-dive`](https://github.com/harut8/system-design/blob/main/kubernetes/16-cilium-and-ebpf-deep-dive.md) | Hubble — flow-level visibility as a debugging tool, not just a security one |

## Practice worth stealing

[`k8s-learn/service-networking-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/service-networking-tasks.md)
— Services, Endpoints, DNS and Ingress as a graded task ladder. A good way to make the
[packet-path artifact](../practice/packet-path-and-debug/README.md) concrete on a kind cluster.

## Scale note

[`kubernetes/35-performance-scaling-and-tuning`](https://github.com/harut8/system-design/blob/main/kubernetes/35-performance-scaling-and-tuning.md)
covers what breaks at 5,000–15,000 nodes, and a surprising amount of it is networking: EndpointSlice
churn, conntrack table pressure, and DNS QPS. Worth a pass before any "design networking for a large
fleet" interview question.
