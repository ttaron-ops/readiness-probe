---
area: "Kubernetes operations"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 Kubernetes operations — interview refresh

> Day-2 ops, upgrades, multi-cluster, troubleshooting — the CKA-and-beyond operational layer.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

## Talking points to have ready

- **Upgrades & version skew.** The control-plane-first order (apiserver → controller-manager/
  scheduler → kubelets), the **N-2 kubelet skew** rule, drain/cordon with PodDisruptionBudgets,
  and surge vs in-place node pool upgrades. Say how you upgrade without dropping a long-running
  job (the GPU-fleet version of this: don't kill a 500-GPU training run — cordon, let it
  checkpoint, then drain).
- **Scheduling & resource management.** Requests/limits, QoS classes (Guaranteed/Burstable/
  BestEffort) and eviction order, taints/tolerations, affinity/topology-spread, PriorityClasses +
  preemption. The GPU angle: `nvidia.com/gpu` as an integer extended resource, **no overcommit**,
  and gang scheduling (Kueue/Volcano) for all-or-nothing jobs.
- **Day-2 troubleshooting decision tree.** Pending pod → (unschedulable? quota? taint? resource
  shape?); CrashLoop → (logs/exit code/OOMKilled=137); node NotReady → (kubelet/CNI/disk/PLEG);
  intermittent 5xx → (readiness, conntrack, DNS ndots, MTU). Name the **`kubectl` + node-level**
  moves, not just `describe`.
- **Multi-cluster & fleet.** Fleet management (Cluster API, Rancher/Fleet, Argo ApplicationSets);
  when multi-cluster beats multi-tenant-in-one (blast radius, hard isolation, regional); the cost
  of the extra control planes.
- **etcd & control-plane health.** etcd as the critical dependency — quorum math (3 vs 5), slow-
  disk sensitivity, defrag/compaction, backup/restore RTO. (Deep-dived in the bare-metal module.)

## Self-quiz

- In what order do you upgrade control-plane components and kubelets, and what's the skew limit?
- QoS classes — which pods get evicted first under node memory pressure, and why?
- A pod is `Pending` — walk the five checks in order.
- Why can't you overcommit GPUs the way you overcommit CPU, and what does that do to bin-packing?
- Intermittent pod-to-service 5xx with healthy pods — name three network-layer suspects.

## Refresh only if

- **Dynamic Resource Allocation (DRA)** — the GA-track replacement for device plugins for
  GPUs/accelerators; if your model is still "device plugin + extended resource," read the DRA
  KEP/docs (it's a live 2025-26 topic and directly relevant to GPU scheduling).
- **Gateway API** (superseding Ingress) and **sidecarless/ambient mesh** if those postdate your
  last hands-on.
