# Depth map — Module 10 · Bare metal & lifecycle

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **The best match for the etcd and control-plane lessons.** `kubernetes/04-etcd-internals` is 2,700
> lines and goes well past what any operations guide covers; the `databases/` track supplies the
> replication and leader-election theory underneath it.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Cluster provisioning | [`kubernetes/32-cluster-lifecycle-and-day2`](https://github.com/harut8/system-design/blob/main/kubernetes/32-cluster-lifecycle-and-day2.md) | bootstrap, upgrade ordering, certificate rotation, and the day-2 work that actually consumes the year |
| 02 etcd operations | [`kubernetes/04-etcd-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/04-etcd-internals.md) | **the chapter.** MVCC revisions, compaction vs defrag, the watch implementation, quota-backend-bytes, and why `fsync` latency is the metric that decides your cluster's health |
| 02 etcd operations | [`databases/14-write-ahead-log-internals`](https://github.com/harut8/system-design/blob/main/databases/14-write-ahead-log-internals.md) | the WAL mechanics that make etcd fsync-bound in the first place — group commit, durability vs latency |
| 02 etcd operations | [`databases/13-lsm-trees-and-compaction`](https://github.com/harut8/system-design/blob/main/databases/13-lsm-trees-and-compaction.md) | compaction as a general storage-engine problem — clarifies why etcd compaction and defrag are two different operations people constantly conflate |
| 03 Control-plane HA | [`databases/16-failure-detection-and-leader-election`](https://github.com/harut8/system-design/blob/main/databases/16-failure-detection-and-leader-election.md) | failure detectors, split-brain, fencing — the theory under Raft leadership and lease-based HA |
| 03 Control-plane HA | [`databases/12-replication-and-distributed-storage`](https://github.com/harut8/system-design/blob/main/databases/12-replication-and-distributed-storage.md) | quorum arithmetic and replication topologies; pairs with `platform/01` L2–L3 |
| 03 Control-plane HA | [`kubernetes/05-kube-apiserver-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/05-kube-apiserver-internals.md) | the watch cache and API Priority & Fairness — what actually saturates first under fleet load |
| 04 Declarative fleets (CAPI, Talos) | [`kubernetes/26-multi-cluster-and-fleet`](https://github.com/harut8/system-design/blob/main/kubernetes/26-multi-cluster-and-fleet.md) | fleet management patterns, hub-and-spoke, and the failure modes of cluster-of-clusters |
| 04 Declarative fleets (CAPI, Talos) | [`kubernetes/31-gitops-helm-kustomize`](https://github.com/harut8/system-design/blob/main/kubernetes/31-gitops-helm-kustomize.md) | 3,400 lines on the reconciliation loop that drives declarative fleet state |
| 05 Node provisioning / PXE | [`kubernetes/33-edge-and-special-distributions`](https://github.com/harut8/system-design/blob/main/kubernetes/33-edge-and-special-distributions.md) | immutable and special-purpose distributions — the design space Talos sits in |
| 06 Hardware health & RMA | [`gpu-observability/07-hardware-health-and-failure-detection`](https://github.com/harut8/system-design/blob/main/gpu-observability/07-hardware-health-and-failure-detection.md) | the detection half of the remediation loop; the flowcharts in `appendix-c` are the triage tree |
| 07 Storage for AI | [`kubernetes/19-storage-csi-pv-pvc`](https://github.com/harut8/system-design/blob/main/kubernetes/19-storage-csi-pv-pvc.md) | CSI, the three-phase volume lifecycle, local PVs and topology constraints |
| 07 Storage for AI | [`databases/00-os-and-hardware-internals`](https://github.com/harut8/system-design/blob/main/databases/00-os-and-hardware-internals.md) | 3,000 lines on the storage stack from the device up — the layer under any "which filesystem for checkpoints" argument |
| 08 Capex vs cloud | [`sre-observability/39-build-vs-buy-framework`](https://github.com/harut8/system-design/blob/main/sre-observability/39-build-vs-buy-framework.md) | a transferable decision framework — the same shape as your TCO argument, in a different domain |

## Also worth a pass

- [`kubernetes/37-cloud-provider-integration`](https://github.com/harut8/system-design/blob/main/kubernetes/37-cloud-provider-integration.md)
  — the cloud-controller-manager, and what stops working on bare metal. A good checklist of what
  you inherit when you leave a managed control plane.
- [`kubernetes/36-garbage-collection-and-object-lifecycle`](https://github.com/harut8/system-design/blob/main/kubernetes/36-garbage-collection-and-object-lifecycle.md)
  — finalizers and orphaned objects, a recurring cause of nodes that won't drain.
