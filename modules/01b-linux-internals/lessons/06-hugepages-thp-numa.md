---
lesson: "01b.6"
title: "Huge pages, THP, and NUMA locality"
module: "01b"
concept: "Huge pages, THP, and NUMA locality"
status: not-started
est_time: "4h"
artifacts: []
---

# 01b.6 · Huge pages, THP, and NUMA locality

> **Concept.** The TLB, page size, and memory-node distance decide whether a GPU box feeds its accelerators at line rate or stalls on address translation and remote DRAM.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters

A GPU training node is a memory-bandwidth machine wearing a CPU. The GPUs do the math, but the CPU-side data pipeline — page cache, dataloader workers, pinned host buffers, RDMA/GPUDirect staging — has to move tens of GB/s of tensors through host DRAM before it ever hits the PCIe/NVLink fabric. Two kernel mechanisms silently tax that pipeline:

- **TLB pressure.** Every virtual address the CPU touches must be translated to physical via the page tables. The TLB caches those translations. With 4 KiB pages, a 40 GiB pinned dataloader buffer needs ~10 million translations; the TLB holds a few thousand. The result is a storm of TLB misses, each a multi-level page-table walk (up to 4–5 memory accesses on x86-64). Huge pages (2 MiB, 1 GiB) map 512× or 262144× more memory per TLB entry, collapsing that walk overhead.
- **NUMA distance.** A dual-socket GPU node has two (or more) memory nodes. DRAM attached to the *other* socket is 1.5–2.1× higher latency and shares a limited inter-socket link (UPI/Infinity Fabric). If your dataloader allocates on node 0 but the GPU and its NIC hang off node 1's PCIe root complex, every batch crosses the interconnect twice and competes with NCCL traffic.

When a fleet of 40+ nodes each loses 10–20% of dataloader throughput to remote memory and TLB churn, you are paying for GPUs that idle waiting on `H2D` copies. This lesson is the kernel-mechanism half of the "why is my GPU only 60% utilized" investigation.

## From using to understanding

**What you already know as an operator:**
- You've seen `hugepages` show up as a Kubernetes resource (`hugepages-2Mi`) and maybe set `vm.nr_hugepages` for a database or DPDK app.
- You know NUMA "is a thing" and that `numactl` exists; you may have seen `numad` or `--cpu-manager-policy=static` on kubelet.
- You know THP because someone told you to *disable* it for Redis/Mongo/Postgres.

**What the kernel is actually doing:**
- The MMU walks a **radix page table** (PGD→P4D→PUD→PMD→PTE on x86-64). A **huge page** is a mapping installed at the PMD level (2 MiB) or PUD level (1 GiB), skipping the lowest level entirely — fewer walk steps *and* one TLB entry covers the whole region.
- **THP** is the kernel opportunistically promoting anonymous memory to 2 MiB pages behind your back, either at fault time or via the background `khugepaged` thread that scans and *collapses* 512 contiguous 4 KiB pages into one. It also has to *split* them under memory pressure — that split-and-compact machinery is the latency hazard.
- **NUMA** is not a scheduler hint; it's physical topology. The kernel's default allocation policy is *first-touch*: a page is placed on the node of the CPU that first writes it. So *who touches the buffer first* determines where it lives, which is why thread pinning and allocation order matter more than any config knob.

## Core notes

### 1. TLB, page-table walks, and why page size dominates

On x86-64 a virtual address is translated through 4 levels (5 with LA57). A TLB miss triggers a hardware page walk of up to 4 dependent memory loads; the page-walk cache mitigates upper levels but not the leaf. Typical L1 dTLB: ~64 entries for 4K, plus a small number of 2M entries; L2 (STLB): ~1536–2048 entries shared. Coverage math:

- 4 KiB × 1536 STLB entries ≈ **6 MiB** reach before thrash.
- 2 MiB × (2M-capable entries) covers **gigabytes**.

For a streaming dataloader touching a huge working set with poor locality, TLB misses become a measurable fraction of cycles. Measure it, don't guess:

```
perf stat -e dTLB-load-misses,dTLB-loads,cycles ./dataloader
# also: cpu-cycles vs. cycle_activity.stalls_mem_any on Intel
```

A high `dTLB-load-misses` rate that drops when you switch to huge pages is the proof.

### 2. Transparent Huge Pages (THP): the knobs and the hazard

Control files (sysfs):

```
/sys/kernel/mm/transparent_hugepage/enabled      # always | madvise | never
/sys/kernel/mm/transparent_hugepage/defrag       # always | defer | defer+madvise | madvise | never
/sys/kernel/mm/transparent_hugepage/khugepaged/*  # scan interval, pages_to_scan, etc.
/sys/kernel/mm/transparent_hugepage/hpage_pmd_size # usually 2097152 (2 MiB)
```

Reading state — the active mode is shown in `[brackets]`:

```
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
```

Observe THP usage:

```
$ grep -E 'AnonHugePages|ShmemHugePages|Huge' /proc/meminfo
AnonHugePages:  12582912 kB     # anon memory currently backed by THP
ShmemHugePages:        0 kB
$ grep -i huge /proc/<pid>/smaps_rollup   # per-process THP totals
AnonHugePages:  10485760 kB
```

Tradeoffs:

- **`always`**: kernel promotes all eligible anon mappings. **Throughput win** for long-lived, large, statically-sized buffers with sequential/dense access — ML training arenas, tensor staging buffers, in-memory feature stores, JVM/Go heaps that stay hot. Fewer TLB misses, better prefetch.
- **`always` as a latency hazard**: two failure modes. (1) **Allocation-time stalls** — under fragmentation, faulting a 2 MiB page can trigger *synchronous compaction*, a multi-millisecond pause on the critical path. This is why latency-sensitive stores (Redis, Postgres) recommend `never`. (2) **Write amplification / RSS bloat** — a process that touches one byte in a 2 MiB region pulls the whole page resident; databases with sparse, random small writes waste memory and suffer split overhead.
- **`madvise`**: THP only where the app explicitly `madvise(MADV_HUGEPAGE)`s. This is the sane default for mixed nodes — training frameworks that want it opt in; latency-sensitive sidecars don't pay.
- **`never`**: off entirely.

The `defrag` knob decouples *how hard* the kernel works to get a huge page from *whether* it tries. `defer+madvise` (a common distro default) means: for `madvise` regions, do direct reclaim/compaction synchronously; elsewhere just wake `kswapd`/`kcompactd` and move on. That avoids the worst synchronous-compaction stalls while still delivering THP to opted-in workloads.

**Rule of thumb for a GPU node:** `madvise` at the host level; let the training runtime request THP for its arenas; keep it off the path of any latency-critical infra pod.

### 3. Explicit huge pages: reservation, hugetlbfs, and Kubernetes

THP is best-effort and can be split away. When you need *guaranteed*, non-swappable, non-splittable huge pages — DPDK, SPDK, RDMA/GPUDirect staging, some CUDA pinned-memory setups — you reserve them explicitly. These come from a separate pool (`HugeTLB`), distinct from THP.

Reserve at runtime (2 MiB pages):

```
$ sysctl -w vm.nr_hugepages=1024          # 1024 × 2 MiB = 2 GiB reserved
# or per-node, respecting NUMA:
$ echo 512 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
$ echo 512 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```

1 GiB pages must usually be reserved at boot (they need contiguous physical memory that's impossible to assemble later): kernel cmdline `default_hugepagesz=1G hugepagesz=1G hugepages=16`.

Inspect the pool:

```
$ grep Huge /proc/meminfo
HugePages_Total:    1024
HugePages_Free:      1024
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:         2097152 kB
$ cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages   # 1 GiB pool
```

Apps consume them either via `hugetlbfs` (`mount -t hugetlbfs none /dev/hugepages`, then `mmap` a file) or anonymous `mmap(..., MAP_HUGETLB | MAP_HUGE_2MB, ...)`.

**Kubernetes** surfaces the reserved pool as a schedulable resource. Kubelet discovers `HugePages_Total` per size and advertises `hugepages-2Mi` / `hugepages-1Gi`:

```yaml
resources:
  limits:
    hugepages-2Mi: "2Gi"
    memory: "8Gi"
  requests:
    cpu: "4"
volumes:
- name: hugepage
  emptyDir:
    medium: HugePages          # or HugePages-2Mi to pin the size
```

Key gotchas: huge pages are **pre-allocated capacity**, not overcommit — the node must have them reserved before the pod schedules; requests must equal limits; they do **not** count against the pod's `memory` limit; and pinning is per-page-size. This is exactly the resource a DPDK-based CNI or an RDMA data-plane pod asks for.

### 4. NUMA locality: first-touch, distance, and device affinity

Discover topology:

```
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0-31,64-95
node 0 size: 515000 MB
node 1 cpus: 32-63,96-127
node 1 size: 515000 MB
node distances:
node   0   1
  0:  10  21      # 21 = ~2.1× the cost of local (10) to reach node 1
  1:  21  10
```

The distance matrix is relative (local normalized to 10). A `21` means a remote access costs roughly twice a local one in the SLIT the firmware reports — real-world latency delta is often 1.5–2×, plus contention on the inter-socket link.

**First-touch policy**: `malloc` doesn't place pages; the first write does, on the writing CPU's node. So a dataloader that allocates a buffer in the main thread but fills it from worker threads scattered across sockets ends up with pages smeared across nodes. Fixes:

```
# Pin an entire process to node 1's CPUs AND memory:
numactl --cpunodebind=1 --membind=1 ./train

# Interleave (good for bandwidth-bound, node-agnostic workloads):
numactl --interleave=all ./bench
```

Check where a running process's pages actually landed:

```
$ numastat -p <pid>
                 Node 0   Node 1   Total
Private          40960.1  12288.0  53248.1   # MB — leaked onto node 0
$ cat /proc/<pid>/numa_maps | head
```

**Device-to-NUMA affinity** is the piece operators miss. A GPU, NIC, or NVMe drive sits behind a PCIe root complex that belongs to one NUMA node. The kernel exposes it:

```
$ cat /sys/class/net/eth0/device/numa_node
1
$ cat /sys/class/net/ib0/device/numa_node          # RDMA NIC for NCCL
1
$ cat /sys/bus/pci/devices/0000:81:00.0/numa_node   # a GPU
1
$ nvidia-smi topo -m       # full GPU/NIC/CPU affinity matrix, incl. NVLink/PXB/SYS
```

A `numa_node` of `-1` means the firmware didn't report affinity (common on some single-socket or misconfigured BIOS) — treat as "unknown, verify with `lstopo`". Use `hwloc`'s `lstopo` for the full picture including PCIe link types.

**The GPU data-path rule:** put the dataloader threads, their pinned host buffers, *and* the NIC/GPU on the *same* NUMA node. Then a batch goes local-DRAM → local-PCIe → GPU without ever crossing UPI, and NCCL's RDMA path to the local NIC doesn't contend with cross-socket dataloader traffic. In Kubernetes this is what the **Topology Manager** (`--topology-manager-policy=single-numa-node`) plus CPU Manager `static` and the NVIDIA device plugin coordinate: they co-locate the allocated CPUs, huge pages, GPUs, and SR-IOV NIC VFs on one node so the container's threads land next to its accelerators.

## Worked example

**Investigation: "training throughput dropped 18% after we rebalanced pods onto the bigger nodes."**

1. Baseline the node. `numactl --hardware` shows 2 nodes, distance 21. GPUs `0000:81:00.0`–`84` report `numa_node = 1`; the RDMA NIC `ib0` also `numa_node = 1`.

2. Look at the running trainer. It was scheduled with only a `memory` limit, no CPU pinning, `--topology-manager-policy=none`. `numastat -p <pid>`:

   ```
                    Node 0    Node 1
   Private          38000.0   9000.0     # ~80% of its RSS on node 0
   ```

   But the GPUs and NIC are on **node 1**. Every H2D copy and every NCCL send crosses UPI.

3. Confirm the mechanism. `perf stat -e dTLB-load-misses,cycles` shows a high miss rate, and `cat /sys/kernel/mm/transparent_hugepage/enabled` reads `always [madvise] never` — the framework never `madvise`d, so the big arena is 4 KiB pages: TLB thrash *on top of* remote access.

4. Two fixes, ranked. (a) Locality: pin the process to node 1 — `numactl --cpunodebind=1 --membind=1` for a bare run, or enable Topology Manager `single-numa-node` + CPU Manager `static` + the device plugin so kubelet co-locates CPUs/GPU/NIC. `numastat` after: >95% of pages on node 1. (b) TLB: switch the arena allocator to request `MADV_HUGEPAGE` (or run with `THP=always` for that pod only), confirmed by `AnonHugePages` in `/proc/<pid>/smaps_rollup` climbing and `dTLB-load-misses` dropping.

5. Result read: throughput recovers because H2D and NCCL now stay on-socket, and the arena maps with 2 MiB pages. The root cause wasn't "bigger nodes" — it was that bigger nodes are multi-socket, and nothing was pinning the data path to the GPU's node.

## Practice

**Environment:** any Linux box; ideally a 2-socket server, but a laptop/VM still shows the mechanisms (it'll report 1 node — note that and reason about what a 2-socket box would show).

1. **THP state.** `cat /sys/kernel/mm/transparent_hugepage/enabled` and `.../defrag`. Record the active `[bracketed]` mode. Then `grep -E 'AnonHugePages|Huge' /proc/meminfo`.
2. **Watch THP in action.** Run a memory-hungry process (e.g. `python -c "x=bytearray(4*1024**3); import time; time.sleep(120)"` or `stress-ng --vm 1 --vm-bytes 4G --vm-hang 120`). In another shell, `grep AnonHugePages /proc/<pid>/smaps_rollup` and `/proc/meminfo`. Note whether the kernel promoted pages.
3. **NUMA topology.** `numactl --hardware` (install `numactl`). Record node count, sizes, and the distance matrix. On a single-node box, state that explicitly.
4. **Pin and observe.** `numactl --cpunodebind=0 --membind=0 stress-ng --vm 1 --vm-bytes 2G --vm-hang 60 &` then `numastat -p <pid>` — confirm the pages landed on node 0. If you have 2 nodes, repeat with `--membind=1` and watch them move.
5. **Device affinity.** Find a device's node: `cat /sys/class/net/<iface>/device/numa_node` (pick your primary NIC). If you have a GPU, `cat /sys/bus/pci/devices/<BDF>/numa_node` and `nvidia-smi topo -m`.

**Acceptance:** a short note (≤1 page) that (1) maps this machine's NUMA topology — nodes, sizes, distance matrix; (2) records the THP `enabled`/`defrag` state and one observed `AnonHugePages` value under load; (3) states which NUMA node your primary NIC (and GPU, if present) lives on; and (4) contains one concrete sentence: *"For a GPU data path, this matters because ___"* naming the specific cross-node hop you'd avoid and the pinning you'd apply.

## Self-check

**(a) When is THP=always a throughput win, and when is it a latency hazard? Name workload types.**

**Answer:** It's a throughput win for workloads with large, long-lived, densely-accessed anonymous arenas and sequential/streaming access — ML training staging buffers, in-memory feature stores, big JVM/Go heaps, analytics scratch space. There THP cuts TLB misses and page-walk overhead with no downside. It's a latency hazard for latency-sensitive stores with sparse, random, small allocations — Redis, Postgres, MongoDB — for two reasons: faulting a 2 MiB page under fragmentation can trigger synchronous compaction (multi-ms stalls on the critical path), and touching one byte pulls a full 2 MiB resident, causing RSS bloat and split overhead. Hence the standard "disable THP for databases" advice; `madvise` mode is the mixed-fleet compromise.

**(b) Why does cross-NUMA memory access hurt a GPU data-loading path, and how do you pin to avoid it?**

**Answer:** A GPU and its NIC sit behind one socket's PCIe root complex (one NUMA node). If the dataloader's pinned host buffers live on the *other* node (default first-touch places them wherever the allocating thread ran), every host-to-device copy and every NCCL/RDMA transfer traverses the inter-socket link (UPI/Infinity Fabric) — 1.5–2× higher latency, shared bandwidth, and contention with other cross-socket traffic, so the GPU stalls on H2D copies. Avoid it by co-locating the dataloader threads, their buffers, and the device on the GPU's node: `numactl --cpunodebind=N --membind=N` for bare processes, or in Kubernetes enable Topology Manager `single-numa-node` + CPU Manager `static` + the NVIDIA device plugin so kubelet allocates CPUs, huge pages, GPU, and NIC VFs on the same node. Verify with `numastat -p <pid>` and `/sys/bus/pci/devices/<BDF>/numa_node`.

**(c) Explicit huge pages vs THP — when do you reserve hugepages, and how does Kubernetes expose them?**

**Answer:** THP is best-effort, transparent, splittable, and swappable — great for opportunistic TLB relief on ordinary anonymous memory. You reserve *explicit* huge pages (the separate HugeTLB pool via `vm.nr_hugepages` / per-node sysfs, or 1 GiB pages at boot) when you need guaranteed, non-splittable, non-swappable large pages with a stable physical backing — DPDK/SPDK, RDMA/GPUDirect staging, some CUDA pinned-memory setups. Kubernetes exposes the pre-reserved pool as a schedulable resource: kubelet advertises `hugepages-2Mi` / `hugepages-1Gi`, pods request them under `resources.limits` (request must equal limit), they don't count against the pod's `memory` limit, and pods back memory with an `emptyDir{medium: HugePages}` volume. Because they're pre-allocated capacity (not overcommit), the node must have them reserved before scheduling.

## Resources

1. **Transparent Hugepage Support — Linux kernel admin guide** — https://docs.kernel.org/admin-guide/mm/transhuge.html — the authoritative reference for the `enabled`/`defrag` modes, `khugepaged` tunables, and the exact sysfs paths. **Deep.** Read it so you can reason precisely about what `madvise` vs `defer+madvise` actually does under memory pressure rather than cargo-culting "disable THP." (Companion: `hugetlbpage.rst` in the same admin-guide tree for the explicit pool.)
2. **hwloc / `lstopo` and `numactl` documentation** — https://www.open-mpi.org/projects/hwloc/ and `man 8 numactl` — the tools that render and control NUMA + PCIe topology. **Skim** the man pages, **deep** on `lstopo` output once. Why: `numactl --hardware` gives you node distances, but `lstopo` shows *which PCIe link* (and NVLink/PXB/SYS in `nvidia-smi topo -m`) connects a GPU/NIC to which node — the affinity data that drives the pinning decision.
3. **"Squeezing more performance out of the CPU" / NUMA & huge pages tuning writeups** — Brendan Gregg's site https://www.brendangregg.com/ (see the perf and memory-analysis posts) — practical `perf stat` recipes for proving TLB and NUMA effects. **Skim.** Why: turns "I think it's NUMA" into measured `dTLB-load-misses` and `numastat` deltas you can defend in a design review.
