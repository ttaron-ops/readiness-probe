---
lesson: "01b.6"
title: "Huge pages, THP, and NUMA locality"
module: "01b"
concept: "Huge pages, THP, and NUMA locality"
status: not-started
est_time: "5h"
prev: "05-memory-and-oom.md"
next: "07-networking-datapath-conntrack.md"
artifacts: []
sources: 5
---

# 01b.6 · Huge pages, THP, and NUMA locality

> **Concept.** The TLB, page size, and memory-node distance decide whether a GPU box feeds its accelerators at line rate or stalls on address translation and remote DRAM.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 05 established that memory which *survives* — that doesn't get OOM-killed — is accounted in four different ways (virtual, resident, page cache, working set), and that the kernel's reclaim path decides what's cheap to give back. This lesson assumes memory is present and not under OOM pressure, and asks the next question: is that memory being served to the CPU and GPU *efficiently*? Two kernel mechanisms answer that — page size (TLB coverage, THP) and NUMA placement (which socket a page physically lives on relative to the CPU and device touching it). This is the most GPU-specific lesson in the module: the worked example below is a full "why did throughput drop 18%" investigation that only makes sense once you know both mechanisms. It closes the CPU-side memory story before the module turns, in lesson 07, to the network datapath — the next place data has to move efficiently to keep GPUs fed.

## Why this matters

A GPU training node is a memory-bandwidth machine wearing a CPU. The GPUs do the math, but the CPU-side data pipeline — page cache, dataloader workers, pinned host buffers, RDMA/GPUDirect staging — has to move tens of GB/s of tensors through host DRAM before it ever hits the PCIe/NVLink fabric. Two kernel mechanisms silently tax that pipeline:

- **TLB pressure.** Every virtual address the CPU touches must be translated to physical via the page tables. The TLB caches those translations. With 4 KiB pages, a 40 GiB pinned dataloader buffer needs ~10 million translations; the TLB holds a few thousand. The result is a storm of TLB misses, each a multi-level page-table walk (up to 4–5 memory accesses on x86-64). Huge pages (2 MiB, 1 GiB) map 512× or 262144× more memory per TLB entry, collapsing that walk overhead.
- **NUMA distance.** A dual-socket GPU node has two (or more) memory nodes. DRAM attached to the *other* socket is 1.5–2.1× higher latency and shares a limited inter-socket link (UPI/Infinity Fabric). If your dataloader allocates on node 0 but the GPU and its NIC hang off node 1's PCIe root complex, every batch crosses the interconnect twice and competes with NCCL traffic.

When a fleet of 40+ nodes each loses 10–20% of dataloader throughput to remote memory and TLB churn, you are paying for GPUs that idle waiting on `H2D` copies. This lesson is the kernel-mechanism half of the "why is my GPU only 60% utilized" investigation.

## What's new here (calibration)

Per the module README's calibration, you already know that `hugepages` shows up as a schedulable Kubernetes resource, that `numactl` exists and NUMA "is a thing," and the folklore instruction to *disable THP* for Redis/Mongo/Postgres — none of that is re-taught. What's genuinely new at this depth:

- The **page-table-walk and TLB-coverage math** that explains *why* huge pages help, not just that they do.
- **THP as a policy machine** (`always`/`madvise`/`never` × `defrag` modes) you can reason about per-workload, replacing "disable it" folklore with "opt in via `madvise`."
- The **hugetlbfs vs THP distinction** and exactly when explicit reservation is required (1 GiB pages, DPDK/RDMA staging).
- **First-touch NUMA placement and device-to-NUMA affinity** as the mechanism behind GPU/NIC co-location, plus `numa_balancing` as a third (usually-disabled-on-GPU-nodes) lever alongside first-touch and explicit pinning.
- The **fleet-scale dollar cost** of getting this wrong — translating a throughput percentage into a staff-level cost argument.

## Core concepts

### From using to understanding

**What you already know as an operator:**
- You've seen `hugepages` show up as a Kubernetes resource (`hugepages-2Mi`) and maybe set `vm.nr_hugepages` for a database or DPDK app.
- You know NUMA "is a thing" and that `numactl` exists; you may have seen `numad` or `--cpu-manager-policy=static` on kubelet.
- You know THP because someone told you to *disable* it for Redis/Mongo/Postgres.

**What the kernel is actually doing:**
- The MMU walks a **radix page table** (PGD→P4D→PUD→PMD→PTE on x86-64). A **huge page** is a mapping installed at the PMD level (2 MiB) or PUD level (1 GiB), skipping the lowest level entirely — fewer walk steps *and* one TLB entry covers the whole region.
- **THP** is the kernel opportunistically promoting anonymous memory to 2 MiB pages behind your back, either at fault time or via the background `khugepaged` thread that scans and *collapses* 512 contiguous 4 KiB pages into one. It also has to *split* them under memory pressure — that split-and-compact machinery is the latency hazard.
- **NUMA** is not a scheduler hint; it's physical topology. The kernel's default allocation policy is *first-touch*: a page is placed on the node of the CPU that first writes it. So *who touches the buffer first* determines where it lives, which is why thread pinning and allocation order matter more than any config knob.

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

1 GiB pages must usually be reserved at boot (they need contiguous physical memory that's impossible to assemble later, since uptime and normal allocation activity fragment physical RAM): kernel cmdline `default_hugepagesz=1G hugepagesz=1G hugepages=16`. Trying to reserve 1 GiB pages at runtime on a system that's been up for a while typically fails or returns far fewer pages than requested — plan the reservation into boot, not into a runtime script.

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

Key gotchas: huge pages are **pre-allocated capacity, not overcommit** — the node must have them reserved before the pod schedules; **requests must equal limits** (unlike CPU/memory, there is no burst above what was requested); they do **not** count against the pod's `memory` limit; and pinning is per-page-size. This is exactly the resource a DPDK-based CNI or an RDMA data-plane pod asks for.

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

The distance matrix is a normalized **SLIT** (System Locality Information Table) metric — firmware-reported, not a directly measured latency. Local is defined as 10 by convention; `21` means the firmware reports a remote access costing roughly twice a local one, but treat it as a rough index, not a literal multiplier: real-world latency delta is typically **1.5–2×**, plus contention effects on the inter-socket link that SLIT doesn't capture at all (concurrent cross-socket traffic makes remote access worse than the static number implies).

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

**A third lever: automatic NUMA balancing.** Besides first-touch placement and explicit `numactl` pinning, the kernel has a third mechanism — `numa_balancing` (a.k.a. AutoNUMA, `/proc/sys/kernel/numa_balancing`). It periodically unmaps pages, catches the resulting faults, and migrates hot pages toward the node of the CPU that's actually touching them. It's a reasonable default for general-purpose, unpinned workloads, but it's **usually disabled on latency-sensitive or GPU nodes** in favor of static Topology Manager pinning: migration is reactive (it only corrects placement after mis-locality is already costing cycles) and itself expensive (unmap, fault, copy, remap), so on a node where you can *know* the right placement up front — CPUs/GPU/NIC statically pinned per pod — letting AutoNUMA "help" just adds unpredictable pause-like overhead on top of a placement you already solved correctly.

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

A `numa_node` of `-1` means the firmware **didn't report affinity** — it does not mean "no affinity" or "any node is fine." This is common on some single-socket or misconfigured-BIOS systems, and trusting it at face value silently defeats your pinning strategy. Treat `-1` as "unknown, verify with `lstopo` or `nvidia-smi topo -m`" rather than as a real answer. Use `hwloc`'s `lstopo` for the full picture including PCIe link types.

**The GPU data-path rule:** put the dataloader threads, their pinned host buffers, *and* the NIC/GPU on the *same* NUMA node. Then a batch goes local-DRAM → local-PCIe → GPU without ever crossing UPI, and NCCL's RDMA path to the local NIC doesn't contend with cross-socket dataloader traffic. In Kubernetes this is what the **Topology Manager** (`--topology-manager-policy=single-numa-node`) plus CPU Manager `static` and the NVIDIA device plugin coordinate: they co-locate the allocated CPUs, huge pages, GPUs, and SR-IOV NIC VFs on one node so the container's threads land next to its accelerators.

The NCCL channel-count evidence makes the stakes concrete: when a GPU and its RDMA NIC are NUMA-aligned (same PCIe root — an `NV#`/`PXB`-class relationship in `nvidia-smi topo -m`), GPUDirect RDMA is used directly, and NCCL opens many parallel channels (commonly 8–16) to saturate the link. When they're misaligned (a `SYS` relationship — crossing the inter-socket link), NCCL falls back to staging transfers through host memory and typically allocates only 2 channels, a direct multi-x collapse in achievable collective bandwidth from topology alone, independent of anything else being wrong.

## Perspectives

**Kernel-mechanism view.** The spine of this lesson: the page-table walk and TLB-coverage math. Every mechanism above — THP, hugetlbfs, first-touch NUMA — exists because a 4 KiB page-table entry is a scarce, expensive-to-fill cache line in the TLB, and physical DRAM is not uniformly close to every CPU. Everything else is policy layered on that one hardware constraint.

**Operator/SRE view.** "Disable THP" is folklore inherited wholesale from database operations without the reasoning that produced it. The correct operator stance is not a blanket toggle but **`madvise` plus explicit opt-in**: set the host default to `madvise`, and let each workload decide via `MADV_HUGEPAGE` whether it wants THP. A training arena opts in and gets a real throughput win; a latency-sensitive sidecar never asks and never pays the compaction-stall tax. Cargo-culting the database rule onto an ML training host throws away a genuine performance win for a hazard that workload doesn't actually have.

**GPU-fleet-specific view.** This lesson's entire worked example lives here: dataloader buffers, pinned host memory, GPU/NIC PCIe-root NUMA affinity, and Topology Manager coordination are not abstract kernel trivia on a GPU fleet — they are the mechanism behind "why is my GPU only 60% utilized" investigations that recur constantly at this scale. Nowhere else in the stack does a purely CPU-side, seemingly unrelated kernel detail (which socket a buffer's pages landed on) directly gate the achievable bandwidth of an NVLink/InfiniBand fabric costing orders of magnitude more than the DRAM it's arguing about.

**Economics view.** A 10–20% dataloader throughput loss from TLB churn and cross-NUMA traffic is not a rounding error at fleet scale. Take a mid-size fleet of 40 GPU nodes, each running training jobs that would otherwise saturate their GPUs; a sustained 15% throughput loss on the CPU-side feed means those GPUs are idle roughly 15% of the time waiting on data, which — at a rough 2026 on-demand rate of order $2–3/GPU-hour per accelerator and, say, 8 GPUs/node — works out to real five-figure-per-month waste across the fleet purely from unpinned buffers and default 4 KiB pages (flagged as a dated 2026 snapshot; the arithmetic scales with fleet size, GPU count/node, and prevailing $/GPU-hour, so recompute it against your own numbers rather than trusting the figure itself). Framed this way, "we should tune THP and NUMA pinning" stops being a kernel-internals curiosity and becomes a cost-differentiator argument a staff engineer can put directly in a capacity-planning or platform-roadmap document.

## Real-world use cases

- **Redis official docs — "Diagnosing latency issues"** — <https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/> — the exact mechanism behind "disable THP for Redis": with THP enabled, `fork()`-based persistence (RDB/AOF background saves) means touching one byte in a shared 2 MiB huge page copy-on-writes the *whole* page, ballooning the CoW set and producing multi-millisecond to multi-second latency spikes during background saves. This is the textbook production instance of "THP as a latency hazard" — and it's from the vendor whose advice created the "always disable THP" folklore that this lesson's `madvise` nuance corrects rather than blindly inherits.
- **Microsoft Azure AKS Engineering Blog — "Optimizing RDMA performance for AI workloads on AKS with DRANET"** — <https://blog.aks.azure.com/2026/04/01/dranet-rdma-optimization-for-ai-on-aks> — concrete NCCL channel-count evidence: when a GPU and its RDMA NIC are NUMA-aligned, GPUDirect RDMA is used directly; when misaligned (`SYS` relationship in `nvidia-smi topo -m`), NCCL falls back to staging through host memory and allocates only 2 channels instead of 8–16. Dated 2026 snapshot — the specific channel counts are a point-in-time NCCL/driver behavior, but the topology-drives-bandwidth mechanism is durable.

## Worked example

**Investigation: "training throughput dropped 18% after we rebalanced pods onto the bigger nodes."**

1. Baseline the node. `numactl --hardware` shows 2 nodes, distance 21. GPUs `0000:81:00.0`–`84` report `numa_node = 1`; the RDMA NIC `ib0` also `numa_node = 1`.

2. Look at the running trainer. It was scheduled with only a `memory` limit, no CPU pinning, `--topology-manager-policy=none`. `numastat -p <pid>`:

   ```
                    Node 0    Node 1
   Private          38000.0   9000.0     # ~80% of its RSS on node 0
   ```

   But the GPUs and NIC are on **node 1**. Every H2D copy and every NCCL send crosses UPI.

3. Confirm the mechanism. `perf stat -e dTLB-load-misses,cycles` shows a high miss rate, and `cat /sys/kernel/mm/transparent_hugepage/enabled` reads `always [madvise] never` — the framework never `madvise`d, so the big arena is 4 KiB pages: TLB thrash *on top of* remote access. `nvidia-smi topo -m` confirms the CPU-GPU relationship for the pinned cores is `SYS`, not `NODE`/`PXB` — the NCCL-channel-collapse pattern described above.

4. Two fixes, ranked. (a) Locality: pin the process to node 1 — `numactl --cpunodebind=1 --membind=1` for a bare run, or enable Topology Manager `single-numa-node` + CPU Manager `static` + the device plugin so kubelet co-locates CPUs/GPU/NIC. `numastat` after: >95% of pages on node 1. (b) TLB: switch the arena allocator to request `MADV_HUGEPAGE` (or run with `THP=always` for that pod only), confirmed by `AnonHugePages` in `/proc/<pid>/smaps_rollup` climbing and `dTLB-load-misses` dropping.

5. Result read: throughput recovers because H2D and NCCL now stay on-socket, and the arena maps with 2 MiB pages. The root cause wasn't "bigger nodes" — it was that bigger nodes are multi-socket, and nothing was pinning the data path to the GPU's node. At fleet scale (see the economics perspective above), this single misconfiguration replicated across every pod scheduled the same way is the 10–20% dataloader loss that shows up as a real monthly cost line.

## Practice

**Environment:** any Linux box; ideally a 2-socket server, but a laptop/VM still shows the mechanisms (it'll report 1 node — note that and reason about what a 2-socket box would show).

1. **THP state.** `cat /sys/kernel/mm/transparent_hugepage/enabled` and `.../defrag`. Record the active `[bracketed]` mode. Then `grep -E 'AnonHugePages|Huge' /proc/meminfo`.
2. **Watch THP in action.** Run a memory-hungry process (e.g. `python -c "x=bytearray(4*1024**3); import time; time.sleep(120)"` or `stress-ng --vm 1 --vm-bytes 4G --vm-hang 120`). In another shell, `grep AnonHugePages /proc/<pid>/smaps_rollup` and `/proc/meminfo`. Note whether the kernel promoted pages.
3. **NUMA topology.** `numactl --hardware` (install `numactl`). Record node count, sizes, and the distance matrix. On a single-node box, state that explicitly.
4. **Pin and observe.** `numactl --cpunodebind=0 --membind=0 stress-ng --vm 1 --vm-bytes 2G --vm-hang 60 &` then `numastat -p <pid>` — confirm the pages landed on node 0. If you have 2 nodes, repeat with `--membind=1` and watch them move.
5. **Device affinity.** Find a device's node: `cat /sys/class/net/<iface>/device/numa_node` (pick your primary NIC). If you have a GPU, `cat /sys/bus/pci/devices/<BDF>/numa_node` and `nvidia-smi topo -m`. If any read `-1`, cross-check with `lstopo` rather than trusting it.
6. **(Stretch) AutoNUMA.** `cat /proc/sys/kernel/numa_balancing` — record whether it's on. If you have 2 nodes, note in one sentence why a GPU node running Topology Manager `single-numa-node` would typically want this off.

**Acceptance:** a short note (≤1 page) that (1) maps this machine's NUMA topology — nodes, sizes, distance matrix; (2) records the THP `enabled`/`defrag` state and one observed `AnonHugePages` value under load; (3) states which NUMA node your primary NIC (and GPU, if present) lives on, flagging any `-1` reads as unverified; and (4) contains one concrete sentence: *"For a GPU data path, this matters because ___"* naming the specific cross-node hop you'd avoid and the pinning you'd apply. Feeds ["Anatomy of a Container"](../practice/anatomy-of-a-container/README.md) as the NUMA/hugepage half of the node-level diagnostic toolkit.

## Common pitfalls

1. **"Disable THP" as a blanket rule** inherited from database folklore (Redis/Postgres/Mongo), applied uncritically to ML training — where THP is actually a throughput win because training arenas are large, long-lived, and densely accessed, the opposite profile of a database's sparse random-write set.
2. **Assuming `numactl --hardware`'s distance numbers (e.g. `21`) are literal latency multipliers.** They're a normalized SLIT metric reported by firmware, not a measured latency. Real-world latency delta is usually 1.5–2×, plus contention on the shared inter-socket link — not exactly "2.1×."
3. **Trusting `numa_node` sysfs files unconditionally.** A value of `-1` means the firmware didn't report affinity, not "no affinity" or "any node." Cross-check with `lstopo` or `nvidia-smi topo -m` before trusting a pinning decision built on it.
4. **Reserving explicit hugepages at runtime and expecting 1 GiB pages to appear.** 1 GiB pages generally require boot-time reservation via kernel cmdline — they need physically contiguous memory that's essentially impossible to assemble once the system has been up and fragmenting for a while.
5. **Assuming Kubernetes hugepage requests overcommit like memory.** They don't. Hugepages are pre-allocated capacity: the request must equal the limit, and the node must have the pages reserved *before* the pod can schedule — there is no bursting above what was requested the way there can be for CPU or (to a point) memory.

## Self-check

**(a) When is THP=always a throughput win, and when is it a latency hazard? Name workload types.**

**Answer:** It's a throughput win for workloads with large, long-lived, densely-accessed anonymous arenas and sequential/streaming access — ML training staging buffers, in-memory feature stores, big JVM/Go heaps, analytics scratch space. There THP cuts TLB misses and page-walk overhead with no downside. It's a latency hazard for latency-sensitive stores with sparse, random, small allocations — Redis, Postgres, MongoDB — for two reasons: faulting a 2 MiB page under fragmentation can trigger synchronous compaction (multi-ms stalls on the critical path), and touching one byte pulls a full 2 MiB resident, causing RSS bloat and split overhead. Hence the standard "disable THP for databases" advice; `madvise` mode is the mixed-fleet compromise.

**(b) Why does cross-NUMA memory access hurt a GPU data-loading path, and how do you pin to avoid it?**

**Answer:** A GPU and its NIC sit behind one socket's PCIe root complex (one NUMA node). If the dataloader's pinned host buffers live on the *other* node (default first-touch places them wherever the allocating thread ran), every host-to-device copy and every NCCL/RDMA transfer traverses the inter-socket link (UPI/Infinity Fabric) — 1.5–2× higher latency, shared bandwidth, and contention with other cross-socket traffic, so the GPU stalls on H2D copies. Avoid it by co-locating the dataloader threads, their buffers, and the device on the GPU's node: `numactl --cpunodebind=N --membind=N` for bare processes, or in Kubernetes enable Topology Manager `single-numa-node` + CPU Manager `static` + the NVIDIA device plugin so kubelet allocates CPUs, huge pages, GPU, and NIC VFs on the same node. Verify with `numastat -p <pid>` and `/sys/bus/pci/devices/<BDF>/numa_node`.

**(c) Explicit huge pages vs THP — when do you reserve hugepages, and how does Kubernetes expose them?**

**Answer:** THP is best-effort, transparent, splittable, and swappable — great for opportunistic TLB relief on ordinary anonymous memory. You reserve *explicit* huge pages (the separate HugeTLB pool via `vm.nr_hugepages` / per-node sysfs, or 1 GiB pages at boot) when you need guaranteed, non-splittable, non-swappable large pages with a stable physical backing — DPDK/SPDK, RDMA/GPUDirect staging, some CUDA pinned-memory setups. Kubernetes exposes the pre-reserved pool as a schedulable resource: kubelet advertises `hugepages-2Mi` / `hugepages-1Gi`, pods request them under `resources.limits` (request must equal limit), they don't count against the pod's `memory` limit, and pods back memory with an `emptyDir{medium: HugePages}` volume. Because they're pre-allocated capacity (not overcommit), the node must have them reserved before scheduling.

**(d) Why would you disable `numa_balancing` (AutoNUMA) on a GPU training node instead of leaving it on?**

**Answer:** `numa_balancing` reactively migrates hot pages toward the node that's touching them — it detects mis-placement *after* it's already costing cross-node access latency, and the migration itself (unmap, fault, copy, remap) is expensive and adds unpredictable overhead. On a GPU node using Topology Manager `single-numa-node` with static CPU/GPU/NIC pinning, you already know the correct placement up front — there's nothing for AutoNUMA to usefully discover, and its background scanning/migration only adds jitter on top of a placement problem that's already solved statically. It's a reasonable default for general-purpose, unpinned workloads, but actively counterproductive once you're doing deliberate topology-aware pinning.

## Connections & what's next

This lesson builds on lesson 05's memory accounting (a page that's resident and not reclaimed is the precondition for asking whether it's *placed* well) and on lesson 03's cgroups v2 (the same `resources.limits`/`requests` machinery that enforces `memory.max` is what advertises and enforces `hugepages-2Mi`/`hugepages-1Gi` as schedulable capacity). It also connects sideways to Kubernetes' Topology Manager and CPU Manager `static` policy — the orchestration layer that turns "the GPU and NIC are on node 1" into "the kubelet only ever schedules this pod's CPUs, hugepages, and devices on node 1." Next, lesson 07 (**Networking datapath & conntrack**) moves from CPU-to-GPU locality to the next hop in the pipeline — how packets actually move through the kernel's networking stack, and where conntrack table pressure becomes the equivalent bottleneck for NCCL/egress traffic that TLB and NUMA misconfiguration are for host memory.

## References & further reading

**Primary sources**
- **Transparent Hugepage Support — Linux kernel admin guide** — <https://docs.kernel.org/admin-guide/mm/transhuge.html> — the authoritative reference for the `enabled`/`defrag` modes, `khugepaged` tunables, and the exact sysfs paths. Read it so you can reason precisely about what `madvise` vs `defer+madvise` actually does under memory pressure rather than cargo-culting "disable THP." (Companion: `hugetlbpage.rst` in the same admin-guide tree for the explicit pool.)

**Real-world engineering blogs**
- **Redis official docs — "Diagnosing latency issues"** — <https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/> — what it shows: the exact fork()/CoW mechanism behind "disable THP for Redis," from the vendor whose advice created the folklore this lesson's `madvise` nuance corrects.
- **Microsoft Azure AKS Engineering Blog — "Optimizing RDMA performance for AI workloads on AKS with DRANET"** — <https://blog.aks.azure.com/2026/04/01/dranet-rdma-optimization-for-ai-on-aks> — what it shows: concrete NCCL channel-count evidence (8–16 channels aligned vs. 2 misaligned) tying NUMA/PCIe topology directly to achievable RDMA bandwidth. Dated 2026 snapshot.

**Deeper dives**
- **hwloc / `lstopo` and `numactl` documentation** — <https://www.open-mpi.org/projects/hwloc/> and `man 8 numactl` — the tools that render and control NUMA + PCIe topology. Skim the man pages, go deep on `lstopo` output once: `numactl --hardware` gives node distances, but `lstopo` shows *which PCIe link* (and NVLink/PXB/SYS in `nvidia-smi topo -m`) connects a GPU/NIC to which node — the affinity data that drives the pinning decision.
- **Brendan Gregg's site** — <https://www.brendangregg.com/> (see the perf and memory-analysis posts) — practical `perf stat` recipes for proving TLB and NUMA effects; turns "I think it's NUMA" into measured `dTLB-load-misses` and `numastat` deltas you can defend in a design review.
