# Depth map — Module 02b · Host topology

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **Partial match.** The source has no 8-GPU-server or PCIe-topology chapter — that content is
> this course's own. What it does have is the **memory-subsystem and kubelet-enforcement layers**
> underneath, which is where topology alignment actually gets decided.

| Lesson | Go deeper in | Why |
|---|---|---|
| 02 Memory subsystem | [`python-mastery/01-memory-hierarchy-and-caches`](https://github.com/harut8/system-design/blob/main/python-mastery/01-memory-hierarchy-and-caches.md) | cache lines, coherence protocols, and the cost of a cross-socket access — the mechanism NUMA alignment exists to avoid |
| 02 Memory subsystem | [`python-mastery/07-virtual-memory`](https://github.com/harut8/system-design/blob/main/python-mastery/07-virtual-memory.md) | page tables and TLB reach — why hugepages matter for large HBM-adjacent host buffers |
| 05 Topology alignment in K8s | [`kubernetes/10-kubelet-internals`](https://github.com/harut8/system-design/blob/main/kubernetes/10-kubelet-internals.md) | **the key chapter** — CPU Manager, Memory Manager, Device Manager and the Topology Manager that reconciles them; this is where alignment succeeds or silently doesn't |
| 05 Topology alignment in K8s | [`kubernetes/21-resource-management-and-qos`](https://github.com/harut8/system-design/blob/main/kubernetes/21-resource-management-and-qos.md) | 3,400 lines on QoS classes, `static` CPU policy, and why only Guaranteed pods get exclusive cores |
| 06 Storage & NVMe | [`python-mastery/09-syscalls-and-io`](https://github.com/harut8/system-design/blob/main/python-mastery/09-syscalls-and-io.md) | the I/O path, buffered vs direct, `io_uring` — what a dataloader is actually waiting on |
| 06 Storage & NVMe | [`kubernetes/19-storage-csi-pv-pvc`](https://github.com/harut8/system-design/blob/main/kubernetes/19-storage-csi-pv-pvc.md) | the three-phase volume lifecycle, local PVs, and how node-local NVMe gets exposed to pods |
| 08 Capstone topology teardown | [`python-mastery/12-observing-a-process`](https://github.com/harut8/system-design/blob/main/python-mastery/12-observing-a-process.md) | reading a process's CPU/memory placement from outside — the verification half of the teardown |

## Also worth a pass

- [`python-mastery/11-ipc-and-shared-memory`](https://github.com/harut8/system-design/blob/main/python-mastery/11-ipc-and-shared-memory.md)
  — shared-memory cost between address spaces. This is the `/dev/shm` sizing problem that breaks
  PyTorch DataLoader workers on GPU nodes, and it is topology-sensitive.
- [`kubernetes/29-pod-sandboxing`](https://github.com/harut8/system-design/blob/main/kubernetes/29-pod-sandboxing.md)
  — gVisor/Kata/RuntimeClass. Relevant if you ever need device passthrough into a sandboxed
  runtime, which interacts badly with topology alignment.

## Nothing there for

PCIe lane topology, NVLink/NVSwitch layout, the 8-GPU HGX board design, power and thermals. That
material is unique to this course — no import available, and Module 03 plus the vendor
architecture whitepapers remain the source.
