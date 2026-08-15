# Depth map — Module 01b · Linux internals

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **One of the two densest matches in the repo.** `python-mastery/00–12` is really an OS-and-hardware
> internals course that happens to use Python for its examples — the mechanisms (page tables, page
> faults, allocators, syscall costs, signals, IPC) are language-neutral and go several times deeper
> than this module's lessons. Chapters are 1,000–2,000 lines each.

| Lesson | Go deeper in | Why |
|---|---|---|
| 01 Processes & scheduling | [`python-mastery/06-processes-threads-scheduling`](https://github.com/harut8/system-design/blob/main/python-mastery/06-processes-threads-scheduling.md) | CFS/EEVDF, run queues, context-switch cost — the layer under `cgroup` CPU shares |
| 02 Namespaces | [`kubernetes/00-linux-primitives-for-containers`](https://github.com/harut8/system-design/blob/main/kubernetes/00-linux-primitives-for-containers.md) | all seven namespaces, `clone`/`setns`/`unshare`, and how runc assembles them |
| 03 cgroups v2 | [`kubernetes/00-linux-primitives-for-containers`](https://github.com/harut8/system-design/blob/main/kubernetes/00-linux-primitives-for-containers.md) · [`kubernetes/21-resource-management-and-qos`](https://github.com/harut8/system-design/blob/main/kubernetes/21-resource-management-and-qos.md) | the controller files themselves, then how the kubelet writes them per QoS class |
| 05 Memory & OOM | [`python-mastery/07-virtual-memory`](https://github.com/harut8/system-design/blob/main/python-mastery/07-virtual-memory.md) | page tables, faults, and **why RSS is not what you allocated** — the single best chapter for reasoning about container memory limits |
| 05 Memory & OOM | [`python-mastery/08-allocators`](https://github.com/harut8/system-design/blob/main/python-mastery/08-allocators.md) | `brk` vs `mmap` and the four allocators between you and the kernel — why freeing memory doesn't return it |
| 06 Hugepages / THP / NUMA | [`python-mastery/01-memory-hierarchy-and-caches`](https://github.com/harut8/system-design/blob/main/python-mastery/01-memory-hierarchy-and-caches.md) | cache lines, coherence, TLB pressure — the mechanism hugepages exist to fix |
| 07 Networking datapath | [`kubernetes/14-services-and-kube-proxy`](https://github.com/harut8/system-design/blob/main/kubernetes/14-services-and-kube-proxy.md) · [`kubernetes/15-cni-and-pod-networking`](https://github.com/harut8/system-design/blob/main/kubernetes/15-cni-and-pod-networking.md) | conntrack and the iptables/IPVS packet path in Kubernetes terms |
| 08 eBPF | [`kubernetes/16-cilium-and-ebpf-deep-dive`](https://github.com/harut8/system-design/blob/main/kubernetes/16-cilium-and-ebpf-deep-dive.md) | 2,800 lines on the verifier, maps, program types, and what replaces iptables |
| 09 perf / ftrace / USE | [`python-mastery/12-observing-a-process`](https://github.com/harut8/system-design/blob/main/python-mastery/12-observing-a-process.md) | `/proc`, `strace`, `perf`, and reading a process's state from outside — the tool-by-tool companion to the USE method |
| 09 perf / ftrace / USE | [`python-mastery/31-measurement-methodology`](https://github.com/harut8/system-design/blob/main/python-mastery/31-measurement-methodology.md) | **how to know whether you actually made it faster** — variance, warm-up, and statistically honest benchmarking. Read this before Module 07's cost-per-token benchmark. |
| 09 perf / ftrace / USE | [`python-mastery/32-profiling`](https://github.com/harut8/system-design/blob/main/python-mastery/32-profiling.md) | "the instrument changes the measurement" — sampling vs instrumenting overhead |
| 10 systemd, cgroups & block I/O | [`python-mastery/09-syscalls-and-io`](https://github.com/harut8/system-design/blob/main/python-mastery/09-syscalls-and-io.md) | syscall cost, buffered vs direct I/O, `io_uring` — what a block-I/O limit actually throttles |

## Also worth a pass

- [`python-mastery/10-signals-fork-exec`](https://github.com/harut8/system-design/blob/main/python-mastery/10-signals-fork-exec.md)
  — signal delivery and process lifecycle. This is the mechanism behind `terminationGracePeriodSeconds`,
  PID 1 in containers, and why a training job may not checkpoint on SIGTERM.
- [`python-mastery/11-ipc-and-shared-memory`](https://github.com/harut8/system-design/blob/main/python-mastery/11-ipc-and-shared-memory.md)
  — what a message between address spaces costs. Directly relevant to `/dev/shm` sizing for
  PyTorch DataLoader workers, a classic GPU-pod failure.
- [`python-mastery/00-cpu-execution-model`](https://github.com/harut8/system-design/blob/main/python-mastery/00-cpu-execution-model.md)
  — pipelines and speculation. Pairs with Module 03's GPU execution model as the CPU contrast.
