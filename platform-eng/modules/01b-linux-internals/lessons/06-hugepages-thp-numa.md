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
sources: 10
---

# 01b.6 · Huge pages, THP, and NUMA locality

> **Concept.** The TLB, page size, and memory-node distance decide whether a GPU box feeds its accelerators at line rate or stalls on address translation and remote DRAM.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 05 established that memory which *survives* — that doesn't get OOM-killed — is accounted in four different ways, and that the reclaim path decides what's cheap to give back. It left two loose threads: the `pagetables` line in `memory.stat` (80 MiB of pure overhead for a 40 GiB process at 4 KiB pages) and the `unevictable` line (long-term-pinned host buffers that reclaim, compaction and migration can never touch). This lesson picks up both. It assumes memory is present and not under OOM pressure, and asks the next question: is that memory being served to the CPU and GPU *efficiently*? Two kernel mechanisms answer that — page size (TLB coverage, THP, hugetlbfs) and NUMA placement (which socket a page physically lives on relative to the CPU and the device touching it). This is the most GPU-specific lesson in the module: the worked example below is a full "why did throughput drop 18%" investigation that only makes sense once you know both. It closes the CPU-side memory story before the module turns, in lesson 07, to the network datapath.

## Why this matters

A GPU training node is a memory-bandwidth machine wearing a CPU. The GPUs do the math, but the CPU-side data pipeline — page cache, dataloader workers, pinned host buffers, RDMA/GPUDirect staging — has to move tens of GB/s of tensors through host DRAM before it ever hits the PCIe/NVLink fabric. Two kernel mechanisms silently tax that pipeline:

- **TLB pressure.** Every virtual address the CPU touches must be translated to a physical one through the page tables. The TLB caches those translations, and it is tiny: a current Intel server core's second-level TLB holds 2,048 entries, an AMD Zen 4 core's holds 3,072. At 4 KiB per entry that is **8 MiB and 12 MiB of reach respectively** — against a dataloader arena of 40 GiB. Every miss is a hardware page-table walk of up to four dependent memory loads. Switching that arena to 2 MiB pages multiplies the reach by 512×; 1 GiB pages multiply it by 262,144×.
- **NUMA distance.** A dual-socket GPU node has two (or more) memory nodes. DRAM attached to the *other* socket costs roughly **1.5–1.8× the idle latency** (Intel Memory Latency Checker typically reports ~70–85 ns local against ~125–145 ns remote on two-socket Xeons) and shares a limited inter-socket link. If your dataloader's pinned buffers live on node 0 but the GPU and its RDMA NIC hang off node 1's PCIe root complex, every batch crosses that link — and NCCL will *notice*: on an Azure ND GB300-v6 node, NCCL opens 2 channels to a NIC it considers `SYS`-distant versus 8 to a NUMA-aligned one and 16 with two aligned NICs (AKS engineering blog, 2026).

When a fleet of 40+ nodes each loses 10–20% of dataloader throughput to remote memory and TLB churn, you are paying for GPUs that idle waiting on `H2D` copies. This lesson is the kernel-mechanism half of the "why is my GPU only 60% utilized" investigation.

## What's new here (calibration)

Per the module README's calibration, you already know that `hugepages` shows up as a schedulable Kubernetes resource, that `numactl` exists and NUMA "is a thing," and the folklore instruction to *disable THP* for Redis/Mongo/Postgres — none of that is re-taught. What's genuinely new at this depth:

- The **page-table-walk structure and TLB-reach arithmetic** that explains *why* huge pages help, with real per-microarchitecture entry counts and a cost model you can plug your own miss rate into.
- **THP as a state machine and a policy machine** — fault-time allocation, `khugepaged` collapse, deferred split, underused-THP shrinking, and the full `enabled` × `defrag` matrix with the kernel's own definitions (including which value is actually the kernel default, which is not the one most guides claim).
- The **hugetlbfs pool as a distinct subsystem**: the exact `/proc/meminfo` fields, the per-size and per-node sysfs tree, surplus/overcommit semantics, demotion, and why 1 GiB pages are a boot-time decision.
- **First-touch placement, the full mempolicy mode list, device-to-NUMA affinity**, and `numa_balancing` — including the kernel's own recommendation about when to switch it off.
- **Pinned host memory as the hinge between the two halves**: why `cudaHostAlloc` memory is unmigratable, and how that turns a NUMA misplacement into a permanent one.
- The **Kubernetes alignment stack** — CPU Manager `static`, Memory Manager `Static`, Topology Manager scopes/policies/options — and the **fleet-scale dollar cost** of skipping it.

## Core concepts

### 1. The problem: every memory access is two memory accesses

A program uses virtual addresses. DRAM uses physical addresses. Something has to translate, on **every single load and store**, and it has to be fast enough not to matter.

x86-64 solves this with a **radix page table**: a four-level tree (five with LA57 enabled, which extends the virtual address space to 57 bits). The CPU's `CR3` register holds the physical address of the top-level table for the current address space. A 48-bit virtual address is chopped into five fields, each indexing one level:

```
  63      48 47    39 38    30 29    21 20    12 11         0
  ┌─────────┬────────┬────────┬────────┬────────┬────────────┐
  │  sign   │  PGD   │  PUD   │  PMD   │  PTE   │   offset   │
  │ extend  │ 9 bits │ 9 bits │ 9 bits │ 9 bits │  12 bits   │
  └─────────┴────────┴────────┴────────┴────────┴────────────┘
```

Each table is one 4 KiB page holding 512 entries of 8 bytes. Walking it means: read the PGD entry (memory access #1), follow it to a PUD table, read the entry (#2), follow to a PMD table, read the entry (#3), follow to a PTE table, read the entry (#4), and *now* you have the physical frame number and can finally issue the actual load (#5). **Four dependent memory accesses before the one you wanted.** The dependency is what hurts: each load's address comes from the previous load's result, so there is no instruction-level parallelism to hide behind.

Two hardware mitigations exist. **Page-walk caches** hold recently used upper-level entries, so most walks only pay for the leaf load. And the **TLB** caches completed translations so most accesses skip the walk entirely.

**The huge-page trick is to stop the walk early.** A PMD entry can set its "page size" bit and point directly at a 2 MiB physical region instead of at a PTE table. A PUD entry can do the same for 1 GiB. That is *all* a huge page is at the hardware level: a mapping installed one level up the tree.

```
        THE WALK, AND WHERE HUGE PAGES CUT IT SHORT

  CR3 ──▶ ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌──────────────┐
          │ PGD │───▶│ PUD │───▶│ PMD │───▶│ PTE │───▶│ 4 KiB frame  │
          └─────┘    └──┬──┘    └──┬──┘    └─────┘    └──────────────┘
                        │          │        4 loads before the data;
                        │          │        1 TLB entry covers 4 KiB
                        │          │
                        │          └─ PS bit set ─▶ ┌───────────────┐
                        │                           │  2 MiB frame  │
                        │             3 loads;      └───────────────┘
                        │             1 TLB entry covers 2 MiB (512×)
                        │
                        └──── PS bit set ─────────▶ ┌───────────────┐
                                                    │  1 GiB frame  │
                                      2 loads;      └───────────────┘
                                      1 TLB entry covers 1 GiB (262144×)

  ── TLB HIERARCHY (per core; entry counts are microarchitecture-specific) ──

   L1 dTLB   ┌──────────────────────────────┐  hit  = ~0 extra cycles
             │ Ice Lake / SPR: 64 × 4K,     │
             │                 32 × 2M,     │
             │                  4 × 1G      │
             │ Zen 4:          72 entries   │
             └──────────────┬───────────────┘
                            │ miss
   L2 STLB   ┌──────────────▼───────────────┐  hit  = ~7-8 extra cycles
             │ Ice Lake / SPR: 2048 entries │
             │ Zen 3:          2048 entries │
             │ Zen 4:          3072 entries │
             └──────────────┬───────────────┘
                            │ miss
   page-walk ┌──────────────▼───────────────┐  hit  = leaf load only
    caches   │ caches upper-level entries   │
             └──────────────┬───────────────┘
                            │ miss
   full walk ┌──────────────▼───────────────┐  up to 4 dependent memory
             │ PGD→PUD→PMD→PTE, then data   │  loads (5 with LA57)
             └──────────────────────────────┘
```

### 2. TLB reach — the number that decides everything

**TLB reach** is the amount of memory a core can address without a page walk: `entries × page_size`. Compute it for the two mainstream server microarchitectures:

| Level | Entries | 4 KiB reach | 2 MiB reach | 1 GiB reach |
|---|---|---|---|---|
| Intel Ice Lake / Sapphire Rapids L1 dTLB | 64 (4K), 32 (2M), 4 (1G) | 256 KiB | 64 MiB | 4 GiB |
| Intel Ice Lake / Sapphire Rapids L2 STLB | 2048 (shared) | **8 MiB** | **4 GiB** | (gen-dependent) |
| AMD Zen 3 L2 TLB | 2048 | 8 MiB | 4 GiB | (gen-dependent) |
| AMD Zen 4 L1 DTLB | 72 | 288 KiB | 144 MiB | 72 GiB |
| AMD Zen 4 L2 TLB | 3072 | **12 MiB** | **6 GiB** | (gen-dependent) |

The Zen 4 numbers cross-check against AMD's own framing — Zen 4 "caches address translations for 288 KB of memory with no latency overhead, or 12 MB with a 7–8 cycle penalty," which is exactly `72 × 4 KiB` and `3072 × 4 KiB`. 1 GiB entry counts are small and vary sharply by generation; verify yours with `cpuid` rather than assuming.

**Worked example: a 40 GiB dataloader arena, randomly accessed.**

```
Arena:            40 GiB
Accesses to cost: 1e8 random 64-byte reads (one shuffled epoch's index gather)
Assume:           Sapphire Rapids, STLB = 2048 entries
Assume:           a TLB miss costs one extra dependent DRAM access ≈ 80 ns
                  (page-walk caches absorb the upper levels; the leaf load misses cache)

  4 KiB pages
    translations needed to cover the arena = 40 GiB / 4 KiB = 10,485,760
    STLB reach                             = 2048 × 4 KiB   = 8 MiB
    fraction of arena resident in STLB     = 8 MiB / 40 GiB = 0.02 %
    → essentially every random access misses.
    translation cost ≈ 1e8 × 80 ns          = 8.0 seconds

  2 MiB pages
    translations needed                    = 40 GiB / 2 MiB = 20,480
    STLB reach                             = 2048 × 2 MiB   = 4 GiB
    fraction resident                      = 4 GiB / 40 GiB = 10 %
    → ~90 % of accesses still miss, but the walk is one level shorter.
    translation cost ≈ 0.9e8 × 60 ns        ≈ 5.4 seconds

  1 GiB pages
    translations needed                    = 40 GiB / 1 GiB = 40
    → 40 entries fit in any modern TLB's 1 GiB capacity plus page-walk caches.
    translation cost ≈ 0
```

The gap between the first and last row is the entire argument for explicit huge pages in a staging buffer. Note what the middle row teaches too: **2 MiB pages are a big win but not a total one for a working set that far exceeds 4 GiB.** That is why DPDK, SPDK and RDMA staging paths reach for 1 GiB pages specifically rather than settling for THP. Re-run it with your own arena size, access pattern and STLB entry count — and measure rather than assume:

```
$ perf stat -e dTLB-loads,dTLB-load-misses,dtlb_load_misses.walk_active,cycles ./dataloader

 12,884,901,888      dTLB-loads
    412,316,860      dTLB-load-misses          #    3.20% of all dTLB cache accesses
 18,253,611,041      dtlb_load_misses.walk_active
 84,120,993,244      cycles
```

Read it: 3.2% miss rate, and `dtlb_load_misses.walk_active` says **21.7% of all cycles had a page walk in flight** (18.25e9 / 84.12e9). That is the number that turns "I think it's TLB" into a defensible claim. Event names differ across vendors and generations — `perf list | grep -i tlb` shows what your CPU actually exposes.

**Page tables as memory, at three page sizes.** From lesson 05: one 4 KiB table page holds 512 entries and maps 512 × page_size of address space. So for a 40 GiB resident arena:

| Page size | Leaf level | Table pages needed | Page-table memory |
|---|---|---|---|
| 4 KiB | PTE (each maps 2 MiB) | 40 GiB / 2 MiB = 20,480 | **80 MiB** |
| 2 MiB | PMD (each maps 1 GiB) | 40 GiB / 1 GiB = 40 | **160 KiB** |
| 1 GiB | PUD (each maps 512 GiB) | 1 | **4 KiB** |

A 500× reduction going from 4 KiB to 2 MiB, and that memory is charged to your cgroup's `memory.max` (`memory.stat`'s `pagetables`) and counted against you by `oom_badness()`. Huge pages are not only a speed optimisation; they are a memory optimisation too.

### 3. Transparent Huge Pages: the state machine

THP is the kernel opportunistically backing anonymous (and tmpfs/shmem) memory with PMD-sized folios without the application asking. Modern kernels also support **mTHP** — "multi-size THP" — which allocates blocks larger than a base page but smaller than PMD size (16 KiB, 32 KiB, 64 KiB…), still PTE-mapped, giving most of the fault-reduction benefit with much smaller latency spikes.

There are three ways a region becomes a THP and three ways it stops being one:

```
              THE LIFE OF A TRANSPARENT HUGE PAGE

   (1) FAULT-TIME ALLOCATION
       process touches an address in a ≥2 MiB aligned, eligible VMA
                     │
                     ▼
            is THP allowed here?
            (sysfs `enabled` == always,
             or == madvise AND the VMA
             has MADV_HUGEPAGE)
                     │
              yes ───┴─── no ──▶ install a 4 KiB page. done.
                     │
                     ▼
            is a 2 MiB free block available right now?
                     │
             yes ────┴──── no
              │             │
              │             ▼
              │      what does sysfs `defrag` say?
              │        always        → DIRECT RECLAIM + COMPACTION *now*
              │                        (the caller STALLS. milliseconds.)
              │        defer         → wake kswapd/kcompactd, use 4 KiB now,
              │                        let khugepaged fix it later
              │        defer+madvise → stall only if MADV_HUGEPAGE, else defer
              │        madvise       → stall only if MADV_HUGEPAGE  ← KERNEL DEFAULT
              │        never         → just use 4 KiB
              │             │
              ▼             ▼
        ┌──────────────────────────┐
        │  2 MiB folio installed   │  counters: thp_fault_alloc++
        │  at the PMD level        │            (or thp_fault_fallback++)
        └────────────┬─────────────┘
                     │
   (2) BACKGROUND COLLAPSE                (3) EXIT PATHS
       khugepaged scans `pages_to_scan`        ├─ partial munmap / mprotect
       pages every `scan_sleep_millisecs`      │    → thp_split_pmd
       ms, finds 512 contiguous 4 KiB pages    ├─ swap-out without contiguous
       (allowing up to `max_ptes_none` holes)  │    swap space → thp_swpout_fallback
       and collapses them.                     ├─ memory pressure + THP is
       counters: thp_collapse_alloc++          │    "underused" (more zero-filled
                 pages_collapsed++             │    pages than max_ptes_none) and
                     │                         │    shrink_underused == 1
                     └────────────────────────▶│    → thp_underused_split_page
                                               └─ CoW after fork: touching ONE
                                                  byte copies the WHOLE 2 MiB
                                                  (this is the Redis failure mode)
```

**The sysfs controls, with the kernel's own definitions.**

```
/sys/kernel/mm/transparent_hugepage/enabled                    # always | madvise | never
/sys/kernel/mm/transparent_hugepage/defrag                     # always | defer | defer+madvise | madvise | never
/sys/kernel/mm/transparent_hugepage/hpage_pmd_size             # 2097152 on x86-64
/sys/kernel/mm/transparent_hugepage/use_zero_page              # 0 | 1  (huge zero page on read faults)
/sys/kernel/mm/transparent_hugepage/shrink_underused           # 0 | 1  (split underused THPs under pressure)
/sys/kernel/mm/transparent_hugepage/hugepages-<size>kB/enabled # always | madvise | never | inherit
/sys/kernel/mm/transparent_hugepage/hugepages-<size>kB/stats/* # per-size counters
/sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan
/sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs
/sys/kernel/mm/transparent_hugepage/khugepaged/alloc_sleep_millisecs
/sys/kernel/mm/transparent_hugepage/khugepaged/max_ptes_none
/sys/kernel/mm/transparent_hugepage/khugepaged/max_ptes_swap
/sys/kernel/mm/transparent_hugepage/khugepaged/max_ptes_shared
/sys/kernel/mm/transparent_hugepage/khugepaged/pages_collapsed
/sys/kernel/mm/transparent_hugepage/khugepaged/full_scans
```

Reading state — the active mode is the one in `[brackets]`:

```
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
$ cat /sys/kernel/mm/transparent_hugepage/defrag
always defer defer+madvise [madvise] never
$ cat /sys/kernel/mm/transparent_hugepage/hpage_pmd_size
2097152
```

**`enabled` — what each value means:**

| Value | Behaviour |
|---|---|
| `always` | Every eligible anonymous VMA is a THP candidate. An application requesting THP will stall on allocation failure and directly reclaim pages. |
| `madvise` | THP only in regions the application marked with `madvise(MADV_HUGEPAGE)`. |
| `never` | No THP allocation — **but note `madvise(MADV_COLLAPSE)` ignores this setting entirely** and will still collapse a range to PMD-sized pages. "Never" is not a global off switch. |
| `inherit` | (per-size files only) Adopt the top-level value. PMD-sized THP defaults to `inherit`; all other sizes default to `never`. |

**`defrag` — how hard the kernel works to *get* a huge page.** This is the knob that owns the latency risk, and it is separate from whether THP is attempted at all.

| Value | Behaviour (kernel `transhuge.rst`) |
|---|---|
| `always` | The application stalls on allocation failure and directly reclaims *and compacts* memory to get a THP immediately. Good for VMs willing to trade startup latency for THP. |
| `defer` | Wake `kswapd` to reclaim and `kcompactd` to compact in the background; use small pages now and let `khugepaged` install the THP later. No caller stall. |
| `defer+madvise` | Direct reclaim + compaction (i.e. stall) **only** for `MADV_HUGEPAGE` regions; everything else defers. |
| `madvise` | Direct reclaim **only** for `MADV_HUGEPAGE` regions. **This is the kernel's documented default.** |
| `never` | No defragmentation effort; fall back to small pages immediately. |

**Correction worth internalising:** the kernel default for `defrag` is `madvise`, not `defer+madvise`. Some distributions ship `defer+madvise`, and the difference is real — under `madvise`, an `MADV_HUGEPAGE` region that cannot get a free 2 MiB block will trigger *synchronous* direct reclaim and compaction on the faulting thread. Check what your image actually ships; do not assume.

**Observing THP.** Three levels of granularity:

```
$ grep -E 'AnonHugePages|ShmemHugePages|ShmemPmdMapped|FileHugePages|FilePmdMapped' /proc/meminfo
AnonHugePages:   4149248 kB      # anon memory currently backed by PMD-sized THP
ShmemHugePages:        0 kB      # shmem/tmpfs allocated with huge pages
ShmemPmdMapped:        0 kB      # shmem mapped into userspace with huge pages
FileHugePages:         0 kB      # page cache allocated with huge pages
FilePmdMapped:         0 kB      # page cache mapped into userspace with huge pages

$ grep -E 'AnonHugePages|Private_Hugetlb|Shared_Hugetlb' /proc/<pid>/smaps_rollup
AnonHugePages:  10485760 kB      # this process's THP total
Shared_Hugetlb:        0 kB      # explicit hugetlb (a DIFFERENT pool — §4)
Private_Hugetlb:       0 kB

$ grep -E 'thp_fault_alloc|thp_fault_fallback|thp_collapse_alloc|thp_split|thp_deferred' /proc/vmstat
thp_fault_alloc 218842           # THPs installed at fault time
thp_fault_fallback 91238         # fault wanted a THP and had to settle for 4 KiB
thp_collapse_alloc 4412          # THPs khugepaged built out of existing small pages
thp_split_pmd 1204               # PMD mappings broken back into PTEs
thp_deferred_split_page 8871     # queued for split under pressure

$ grep -E 'thp_fault_alloc|thp_collapse_alloc|anon_thp' /sys/fs/cgroup/<path>/memory.stat
anon_thp 8589934592              # per-cgroup: bytes of anon backed by THP
thp_fault_alloc 12844
thp_collapse_alloc 331

$ grep -E 'compact_stall|compact_fail|compact_success' /proc/vmstat
compact_stall 4471               # times a process STALLED waiting for compaction
compact_success 3902
compact_fail 569
```

**`thp_fault_fallback` climbing relative to `thp_fault_alloc` means fragmentation**, not policy — the kernel wanted a THP and could not find a contiguous 2 MiB block. **`compact_stall` climbing means processes are paying for that in wall-clock time on their own fault path.** Those two counters are the entire "is THP hurting me?" diagnosis.

**When THP is a throughput win.** Large, long-lived, densely-accessed anonymous arenas: ML training staging buffers, tensor arenas, in-memory feature stores, big JVM/Go heaps that stay hot. Fewer faults (one per 2 MiB instead of 512), 512× the TLB reach, 500× less page-table memory.

**When THP is a latency hazard — two distinct mechanisms, not one.**

1. **Allocation-time stalls.** Under fragmentation, faulting a 2 MiB page with `defrag=always` (or `madvise` + `MADV_HUGEPAGE`) triggers synchronous direct reclaim and compaction on the faulting thread. That is a multi-millisecond pause in the middle of a request. `compact_stall` in `/proc/vmstat` counts exactly this.
2. **Copy-on-write amplification.** This is the Redis mechanism, and it is worth stating precisely because it is the origin of the entire "disable THP" folklore. Redis persists by calling `fork()`; parent and child then share every page copy-on-write. With 4 KiB pages, a write to one key copies 4 KiB. With THP, that same one-byte write copies the **entire 2 MiB huge page**. A busy instance touching a few thousand keys during a background save therefore copy-on-writes a large fraction of its whole dataset, producing both a large latency spike and a large memory spike. Redis's own latency guide instructs operators to `echo never > /sys/kernel/mm/transparent_hugepage/enabled` and restart the process for this reason.

The second mechanism is *fork-specific*. A training process that never forks after allocating its arena is structurally immune to it. That is why cargo-culting the database rule onto an ML host is a mistake: you take on the cost (giving up 512× TLB reach) to avoid a hazard the workload does not have. The correct stance is `madvise` at the host level plus explicit opt-in — see the operator perspective below.

**How an application opts in.** `madvise(addr, len, MADV_HUGEPAGE)` marks a range as wanting THP; `MADV_NOHUGEPAGE` marks it as not; `MADV_COLLAPSE` synchronously collapses an existing range *regardless of the sysfs setting*. A process can disable THP for itself with `prctl(PR_SET_THP_DISABLE, 1, 0, 0, 0)`, inherited across `fork` and `execve`; `/proc/<pid>/status` reports the result as `THP_enabled`.

### 4. Explicit huge pages: hugetlbfs is a different subsystem

THP is best-effort. It can fail to allocate (`thp_fault_fallback`), it can be split back to 4 KiB pages under pressure (`thp_deferred_split_page`, `thp_underused_split_page`), and it is normal reclaimable/swappable memory. When you need **guaranteed, non-splittable, non-swappable** huge pages with stable physical backing — DPDK/SPDK packet buffers, RDMA and GPUDirect staging regions, QEMU guest RAM — you use the **HugeTLB pool**, which is reserved up front and accounted entirely separately from THP.

**The `/proc/meminfo` fields, and exactly what each one means** (`hugetlbpage.rst`):

```
$ grep -i huge /proc/meminfo
AnonHugePages:   4149248 kB      # ← THP.   Not part of the hugetlb pool at all.
ShmemHugePages:        0 kB      # ← THP.
FileHugePages:         0 kB      # ← THP.
HugePages_Total:    1024         # size of the pool, in pages of Hugepagesize
HugePages_Free:      1024         # pool pages not yet allocated to anyone
HugePages_Rsvd:        0         # committed-but-not-yet-faulted: an app has a
                                 #   guarantee it can fault these in later
HugePages_Surp:        0         # "surplus": pages above nr_hugepages, borrowed
                                 #   from the normal allocator up to
                                 #   nr_overcommit_hugepages
Hugepagesize:       2048 kB      # the DEFAULT huge page size
Hugetlb:         2097152 kB      # total memory consumed by hugetlb of ALL sizes
```

Three traps in that block:

- **`AnonHugePages` and `HugePages_*` are different subsystems.** A node with `AnonHugePages: 4149248 kB` and `HugePages_Total: 0` has 4 GiB of THP and *no* hugetlb pool. Kubernetes advertises capacity from the second, never the first.
- **`HugePages_Free` is not "available to you."** `HugePages_Free − HugePages_Rsvd` is; reserved pages are already promised to a process that has mapped but not yet touched them.
- **`Hugetlb` can exceed `HugePages_Total × Hugepagesize`**, because `HugePages_*` describe only the default size while `Hugetlb` sums every size. With both 2 MiB and 1 GiB pools, read the per-size sysfs tree instead.

**The sysfs tree**, one directory per supported size, plus a per-NUMA-node copy:

```
/sys/kernel/mm/hugepages/hugepages-2048kB/{nr_hugepages,nr_hugepages_mempolicy,
                                           nr_overcommit_hugepages,free_hugepages,
                                           resv_hugepages,surplus_hugepages,
                                           demote,demote_size}
/sys/kernel/mm/hugepages/hugepages-1048576kB/...
/sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
/sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```

**Reserving at runtime**, NUMA-aware (which is the only way that makes sense on a two-socket GPU box):

```
# Global — the kernel spreads these across the nodes allowed by the calling
# task's memory policy, silently skipping nodes without contiguous memory:
$ sysctl -w vm.nr_hugepages=1024                 # 1024 × 2 MiB = 2 GiB

# Explicit per node — deterministic, and what you actually want:
$ echo 512 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
$ echo 512 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
$ cat /sys/devices/system/node/node*/meminfo | grep -i huge
Node 0 HugePages_Total:   512
Node 0 HugePages_Free:    512
Node 1 HugePages_Total:   512
Node 1 HugePages_Free:    512
```

**Why 1 GiB pages must be a boot-time decision.** A 1 GiB page needs one *physically contiguous* gigabyte. After minutes of uptime, normal allocation activity has interleaved kernel objects, page tables and unmovable slab across physical memory; compaction moves movable pages but not unmovable ones, so assembling a contiguous gigabyte becomes impossible in practice. Requesting them at runtime typically delivers far fewer pages than asked — and the write *silently succeeds* while `nr_hugepages` reads back lower. Reserve on the kernel command line instead:

```
default_hugepagesz=1G hugepagesz=1G hugepages=16 hugepagesz=2M hugepages=1024
```

The parameters are positional and pair up: `hugepagesz` selects a size, and the `hugepages` that follows sets the count for that size. `default_hugepagesz` sets which size `Hugepagesize` and the legacy `/proc/sys/vm/nr_hugepages` refer to. Node-specific syntax exists too — `hugepagesz=2M hugepages=0:512,1:512` allocates 512 on node 0 and 512 on node 1.

**Surplus, overcommit and demotion.** `nr_overcommit_hugepages` lets the pool temporarily grow past `nr_hugepages` by borrowing from the normal allocator; those pages show as `HugePages_Surp` and are returned when freed — the closest hugetlb comes to overcommit, and Kubernetes does not use it. `demote_size` and `demote` split larger pool pages into smaller ones (write `1` to `demote` in the 1 GiB directory to turn one 1 GiB page into 512 2 MiB pages), which is the escape hatch when you over-reserved 1 GiB pages at boot.

**How applications consume them.** Either through a `hugetlbfs` mount:

```
$ mount -t hugetlbfs -o pagesize=2M,size=2G none /dev/hugepages
# then open() a file there and mmap() it
```

or anonymously:

```c
void *p = mmap(NULL, 2UL<<30, PROT_READ|PROT_WRITE,
               MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB|MAP_HUGE_1GB, -1, 0);
```

SysV shared memory works too (`shmget` with `SHM_HUGETLB`), which requires the caller's supplementary groups to include `vm.hugetlb_shm_group`.

**Cgroup accounting.** hugetlb usage is tracked by its *own* controller — `hugetlb.2MB.max`, `hugetlb.1GB.current`, etc. It is charged to `memory.current` only if the cgroup hierarchy was mounted with `memory_hugetlb_accounting`, in which case a `hugetlb` key appears in `memory.stat`. **By default, a container's hugetlb consumption does not count against its `memory.max`** — which is exactly why Kubernetes treats hugepages as a separate resource dimension.

### 5. Kubernetes and hugepages

The kubelet discovers the pre-reserved pool per size and advertises it as a schedulable resource:

```
$ kubectl describe node gpu-node-07 | sed -n '/Capacity/,/System Info/p'
Capacity:
  cpu:                128
  hugepages-1Gi:      16Gi
  hugepages-2Mi:      2Gi
  memory:             527654912Ki
  nvidia.com/gpu:     8
  pods:               110
Allocatable:
  cpu:                127
  hugepages-1Gi:      16Gi
  hugepages-2Mi:      2Gi
  memory:             509827392Ki      # note: hugepages are subtracted from
  nvidia.com/gpu:     8                #       allocatable `memory`
  pods:               110
```

A complete pod spec consuming both sizes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rdma-staging
spec:
  containers:
  - name: app
    image: registry.example.com/trainer:2026.08
    command: ["/opt/trainer/run"]
    volumeMounts:
    - mountPath: /hugepages-2Mi          # hugetlbfs is mounted here by the kubelet
      name: hugepage-2mi
    - mountPath: /hugepages-1Gi
      name: hugepage-1gi
    resources:
      limits:
        hugepages-2Mi: 512Mi             # request is implied and MUST equal limit
        hugepages-1Gi: 8Gi
        memory: 64Gi                     # a memory request/limit is MANDATORY
        nvidia.com/gpu: 1
      requests:
        cpu: "8"
        memory: 64Gi
  volumes:
  - name: hugepage-2mi
    emptyDir:
      medium: HugePages-2Mi              # size-qualified: required when using >1 size
  - name: hugepage-1gi
    emptyDir:
      medium: HugePages-1Gi
```

The rules that catch people:

| Rule | Why |
|---|---|
| **Requests must equal limits** (defaulted if you set only limits). | Pre-allocated physical capacity. There is no page to burst into — the pool is a fixed count of frames. |
| **No overcommit.** The node must have the pages reserved *before* the pod schedules. | The scheduler allocates from a fixed pool, not a virtual budget. |
| **You must also request `memory` or `cpu`.** | Kubernetes requires a classic compute resource alongside a hugepage request. |
| **Hugepage memory does not count against the container's `memory` limit.** | The hugetlb cgroup controller accounts it separately; `memory.max` never sees it. Budget node RAM for both. |
| **`medium: HugePages` only works with a single size.** | With two sizes the kubelet cannot infer which pool an unqualified mount draws from; use `medium: HugePages-<size>`. |
| **Isolation is per container, not per pod.** | Each container gets its own hugetlb cgroup limit. |
| **Dynamically allocated pages need a kubelet restart to be noticed.** | Capacity is discovered at kubelet start, not polled. `ResourceQuota` supports `hugepages-<size>` tokens the same way as cpu/memory. |

### 6. NUMA: the physical layout under all of this

On a multi-socket server, memory controllers belong to sockets. A CPU core reaching DRAM attached to its own socket takes the short path; reaching the other socket's DRAM goes over the inter-socket link (Intel UPI, AMD Infinity Fabric) and back. Same for I/O: **a PCIe root complex belongs to a socket**, so a GPU, NIC or NVMe device is physically closer to one socket's cores and memory than the other's.

Concrete numbers for a current two-socket box, with provenance:

| Quantity | Typical value | Source / caveat |
|---|---|---|
| Local DRAM idle latency | ~70–85 ns | Intel MLC measurements on 2-socket Xeon; varies by generation and DIMM config |
| Remote DRAM idle latency | ~125–145 ns | Same; the ratio is **1.5–1.8×**, and it *worsens under load* |
| Per-socket memory bandwidth | 307.2 GB/s | 8 channels × DDR5-4800 × 8 B/transfer (Sapphire Rapids) |
| Inter-socket link | ~48 GB/s per direction per link, up to 4 links | UPI 2.0 at 16 GT/s over 24 lanes; nominal, before protocol overhead |
| PCIe Gen4 ×16 | 31.5 GB/s per direction | 16 GT/s × 16 lanes, 128b/130b encoding |
| PCIe Gen5 ×16 | 63 GB/s per direction | 32 GT/s × 16 lanes |

**The ratio that matters:** per-socket DRAM bandwidth (307 GB/s) is roughly **6× a single UPI link's bandwidth**. Cross-socket traffic is not just higher-latency, it is squeezing through a pipe an order of magnitude narrower than local memory — and it is *shared* with cache-coherence traffic, other processes' remote accesses, and any peer-to-peer device traffic that has to cross.

```
        A TWO-SOCKET GPU NODE, AND THE TWO PATHS A BATCH CAN TAKE

  ┌───────────────── NUMA node 0 ─────────────────┐   ┌───────────── NUMA node 1 ─────────────┐
  │  cores 0-31,64-95        DDR5 ×8  256 GiB     │   │ cores 32-63,96-127   DDR5 ×8  256 GiB │
  │       │                    │  307 GB/s        │   │      │                 │  307 GB/s    │
  │       └────────┬───────────┘  ~75 ns local    │   │      └───────┬─────────┘  ~75 ns      │
  │                │                              │   │              │                        │
  │        ┌───────┴────────┐                     │   │      ┌───────┴────────┐               │
  │        │ PCIe root cplx │                     │   │      │ PCIe root cplx │               │
  │        └───┬────────┬───┘                     │   │      └───┬────────┬───┘               │
  │            │        │                         │   │          │        │                   │
  │        [NVMe]   [mgmt NIC]                    │   │     [GPU 0-3]  [ib0 RDMA NIC]         │
  │                                               │   │      numa_node:1    numa_node:1       │
  └───────────────────────┬───────────────────────┘   └──────────┬────────────────────────────┘
                          │                                      │
                          └──────── UPI 2.0 ─────────────────────┘
                            ~48 GB/s/dir/link · adds ~50-70 ns
                            SLIT reports this as distance 21 vs 10

  PATH A — ALIGNED (what you want)
    dataloader thread pinned to a node-1 core
      → allocates + first-touches its pinned buffer  → node-1 DRAM   (~75 ns, 307 GB/s)
      → cudaMemcpyAsync H2D                          → node-1 PCIe   (31.5 GB/s Gen4 ×16)
      → NCCL posts an RDMA send                      → ib0 on node 1, GPUDirect RDMA direct
    UPI crossings: 0.   nvidia-smi topo -m GPU↔ib0: NODE or better.

  PATH B — MISALIGNED (the default if nothing pins anything)
    dataloader thread scheduled on a node-0 core
      → first-touch puts the buffer in node-0 DRAM
      → cudaMemcpyAsync H2D: DMA engine on node 1 reads node-0 memory  ← UPI crossing #1
      → NCCL sees GPU↔NIC as SYS, disables GPUDirect RDMA, stages the
        transfer through host memory                                   ← UPI crossing #2
    UPI crossings: 2 per batch, contending with every other pod doing the same.
    On an Azure ND GB300-v6, NCCL responds by opening 2 channels instead of 8.
```

**Discovering the topology.**

```
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0-31 64-95
node 0 size: 257862 MB
node 0 free: 201044 MB
node 1 cpus: 32-63 96-127
node 1 size: 257993 MB
node 1 free: 233871 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

The distance matrix is the firmware-provided **SLIT** (System Locality Information Table). Local is defined as `10` by convention, so `21` means the firmware asserts remote access costs about 2.1× local. **Treat it as an index, not a measurement.** Real measured latency deltas run 1.5–1.8×, and SLIT captures nothing about *contention* — under concurrent cross-socket load the effective penalty is worse than the static number implies. The same file is readable per node at `/sys/devices/system/node/node0/distance`.

**First-touch is the default placement policy, and it is the thing people miss.** `malloc` and `mmap` do not place pages. The **first write** does, on the node of the CPU executing that write. So a dataloader that allocates a buffer in the main thread but *fills* it from workers scattered across both sockets ends up smeared across both nodes; a process that allocates and initialises everything in `main()` before spawning workers puts *all* of it on whichever node `main()` happened to run on; and `calloc` changes nothing, because placement still happens on the first real write.

**The mempolicy modes** (`set_mempolicy(2)`, `mbind(2)`, exposed by `numactl`):

| Mode | Behaviour |
|---|---|
| `MPOL_DEFAULT` | Remove any policy; fall back to the next most specific policy in the hierarchy (task → VMA → system default), ultimately local-node allocation. |
| `MPOL_BIND` | Allocate **only** from the specified nodes. If they are full, reclaim there or OOM — do not fall back. This is the strict one. |
| `MPOL_PREFERRED` | Prefer one node; fall back to others when it is full. |
| `MPOL_PREFERRED_MANY` | Like `MPOL_PREFERRED` but with a set of preferred nodes rather than one. |
| `MPOL_INTERLEAVE` | Round-robin allocation across the node set. Good for bandwidth-bound, node-agnostic workloads that would otherwise saturate one memory controller. |
| `MPOL_WEIGHTED_INTERLEAVE` | Interleave with per-node weights — for heterogeneous memory (CXL tiers) where nodes have different bandwidth. |
| `MPOL_LOCAL` | Allocate on the node of the executing CPU. |

Applying them:

```
# Pin CPUs AND memory to node 1 — the GPU's node:
$ numactl --cpunodebind=1 --membind=1 ./train

# Strict memory binding, but let the scheduler place threads:
$ numactl --membind=1 ./train

# Interleave for a bandwidth-bound benchmark with no locality to exploit:
$ numactl --interleave=all ./bench

# Inspect what a running process actually got:
$ numactl --show
policy: default
preferred node: current
cpubind: 0 1
nodebind: 0 1
membind: 0 1
```

**Verifying placement.** Three views, increasingly precise:

```
$ numastat -p 41277
Per-node process memory usage (in MBs) for PID 41277 (python)
                          Node 0          Node 1           Total
                 --------------- --------------- ---------------
Huge                        0.00            0.00            0.00
Heap                       11.55            2.13           13.68
Stack                       0.09            0.00            0.09
Private                 38102.41         9004.72        47107.13    ← 81% on node 0
----------------  --------------- --------------- ---------------
Total                   38114.05         9006.85        47120.90

$ head -4 /proc/41277/numa_maps
7f2a40000000 default anon=9437184 dirty=9437184 N0=7549747 N1=1887437 kernelpagesize_kB=4
7f38c0000000 prefer:1 anon=524288 dirty=524288 N1=524288 kernelpagesize_kB=2048
                │                                  │                        │
                │                                  │                        └─ 2 MiB pages here
                │                                  └─ all of this VMA is on node 1
                └─ the mempolicy in effect for this VMA

$ grep -E '^(anon|file) ' /sys/fs/cgroup/<pod-path>/memory.numa_stat
anon N0=39942340608 N1=9439281152
file N0=1073741824 N1=402653184
```

`numa_maps` is the highest-resolution tool: per-VMA policy, per-node page counts, and `kernelpagesize_kB` telling you which VMAs got huge pages. `memory.numa_stat` gives you the same split per cgroup, which is the one that works when you cannot get a PID inside a container.

**Automatic NUMA balancing (AutoNUMA)** is the third lever, alongside first-touch and explicit binding. `/proc/sys/kernel/numa_balancing` takes a bitmask: `0` disabled, `1` `NUMA_BALANCING_NORMAL`, `2` `NUMA_BALANCING_MEMORY_TIERING`. In `NORMAL` mode the kernel periodically unmaps pages, catches the resulting minor faults, works out which node is actually touching each page, and migrates it there. `memory.stat` reports the activity as `numa_pte_updates`, `numa_hint_faults` and `numa_pages_migrated`.

The kernel's own documentation states the trade-off plainly: *"The unmapping of pages and trapping faults incur additional overhead that ideally is offset by improved memory locality but there is no universal guarantee. **If the target workload is already bound to NUMA nodes then this feature should be disabled.**"* On a GPU node running Topology Manager `single-numa-node` with static CPU and device pinning, the placement is already correct by construction — AutoNUMA has nothing to discover and only adds fault and migration overhead. Turn it off there. On a general-purpose, unpinned node, leave it on.

There is also a manual migration path for one-off correction: `migratepages <pid> <from-nodes> <to-nodes>` moves a running process's pages, and `move_pages(2)` does it programmatically. **Neither can move pinned pages** — which is the subject of §8.

### 7. Device-to-NUMA affinity — the piece operators miss

Every PCIe device has a NUMA node, and the kernel exposes it:

```
$ cat /sys/class/net/ib0/device/numa_node          # the RDMA NIC NCCL will use
1
$ cat /sys/class/net/ib0/device/local_cpulist      # which CPUs are local to it
32-63,96-127
$ cat /sys/bus/pci/devices/0000:81:00.0/numa_node  # a GPU
1
$ cat /sys/bus/pci/devices/0000:81:00.0/local_cpulist
32-63,96-127
```

**A `numa_node` of `-1` does not mean "no affinity" or "any node is fine." It means the firmware did not report affinity.** It is common on single-socket systems (where it is harmless) and on servers with incomplete ACPI `_PXM` data (where it silently defeats your pinning strategy, because tools that read it will conclude there is nothing to align). Treat `-1` as *unknown* and cross-check with `lstopo` (from `hwloc`), which builds the topology from the PCIe hierarchy rather than trusting a single sysfs value.

For GPUs specifically, `nvidia-smi topo -m` gives the full relationship matrix:

```
$ nvidia-smi topo -m
        GPU0  GPU1  GPU2  GPU3  NIC0  NIC1  CPU Affinity  NUMA Affinity
GPU0     X    NV18  NV18  NV18  PXB   SYS   32-63,96-127  1
GPU1    NV18   X    NV18  NV18  PXB   SYS   32-63,96-127  1
GPU2    NV18  NV18   X    NV18  SYS   PXB    0-31,64-95   0
GPU3    NV18  NV18  NV18   X    SYS   PXB    0-31,64-95   0
NIC0    PXB   PXB   SYS   SYS    X    SYS
NIC1    SYS   SYS   PXB   PXB   SYS    X
```

**The legend, in increasing cost:**

| Symbol | Meaning |
|---|---|
| `X` | Self |
| `NV#` | Traverses a bonded set of # NVLinks — the fastest path, GPU-to-GPU only |
| `PIX` | Traverses at most a single PCIe bridge |
| `PXB` | Traverses multiple PCIe bridges, without going through the PCIe host bridge |
| `PHB` | Traverses a PCIe host bridge (typically the CPU) |
| `NODE` | Traverses PCIe plus the interconnect between PCIe host bridges **within** one NUMA node |
| `SYS` | Traverses PCIe **plus the SMP interconnect between NUMA nodes** (UPI/Infinity Fabric) — the slowest, and the one that disables GPUDirect RDMA in typical configurations |

Read the matrix above: GPU0/GPU1 pair with NIC0 (`PXB`) and are on NUMA node 1; GPU2/GPU3 pair with NIC1 and are on node 0. Any job that puts GPU0's dataloader threads on CPUs 0–31 has already lost: it will drive GPU0 across UPI *and* be told by NCCL that its NIC is `SYS`-distant.

**Why the GPU↔NIC relationship is decisive.** GPUDirect RDMA lets a NIC DMA directly into GPU device memory, bypassing host memory entirely. That works when the NIC and GPU share a common PCIe root complex; when they do not, the transfer has to be staged through host memory instead, adding a full round trip through DRAM in each direction. NCCL exposes the cutoff as `NCCL_NET_GDR_LEVEL` (values from `LOC` through `PIX`, `PXB`, `PHB`, up to `SYS`), and it makes the decision at init time — run with `NCCL_DEBUG=INFO` and grep for `GDRDMA` / `via NET` lines to see what it chose for the box in front of you rather than assuming a default. The observable consequence, measured on an Azure ND GB300-v6 node (four GB300 GPUs and four ConnectX-8 InfiniBand NICs split across two NUMA domains, GPUs 0–1 and NICs 0–1 on node 0, GPUs 2–3 and NICs 2–3 on node 1): NCCL allocated **2 channels** for a `SYS`-distant NIC, **8** for a NUMA-aligned one, and **16** with two aligned NICs. That is a 4–8× difference in achievable collective bandwidth arising purely from placement.

**The GPU data-path rule, stated once:** put the dataloader threads, their pinned host buffers, the GPU, and the RDMA NIC on the **same NUMA node**. Then a batch goes local-DRAM → local-PCIe → GPU without ever crossing UPI, and NCCL's RDMA path uses GPUDirect directly instead of staging through host memory.

**Kubernetes enforces this with three cooperating components:**

| Component | Setting | What it does |
|---|---|---|
| CPU Manager | `--cpu-manager-policy=static` | Gives Guaranteed pods with integer CPU requests **exclusive** cores, and reports a NUMA hint for them. Without this, nothing pins threads and Topology Manager has nothing to align. |
| Memory Manager | `--memory-manager-policy=static` | Allocates the pod's memory (and hugepages) from specific NUMA nodes and reports a hint. |
| Device Manager / NVIDIA device plugin | — | Reports each GPU's NUMA node as a hint. |
| Topology Manager | `--topology-manager-policy=single-numa-node`, `--topology-manager-scope=pod` | Collects all the hints and **admits the pod only if a single NUMA node can satisfy all of them**; otherwise the pod fails admission with `TopologyAffinityError`. |

The policies, in increasing strictness: `none` (default — no alignment), `best-effort` (prefer aligned, admit anyway if impossible), `restricted` (reject if the preferred allocation is not aligned), `single-numa-node` (reject unless everything fits on exactly one node). Scope is `container` (default) or `pod`; **`pod` scope with `single-numa-node` is the combination that matters for multi-container GPU workloads**, because it groups every container onto one node instead of aligning each independently. Two policy options are worth knowing: `prefer-closest-numa-nodes` (GA 1.32) makes `best-effort`/`restricted` prefer the node set with the smallest SLIT distance when a single node will not do, and `max-allowable-numa-nodes` (GA 1.35, default **8**) caps how many nodes the hint algorithm considers. A pod stuck in `Failed` with reason `TopologyAffinityError` is this working correctly — no single node could satisfy CPUs + memory + GPU together — not a bug to work around by dropping to `best-effort`.

### 8. Pinned host memory: where the two halves meet

Everything above assumes pages can move. For a GPU data path, the most important pages cannot.

**Why pinning exists.** A DMA engine addresses physical memory. If the kernel could swap out or migrate a page mid-transfer, the DMA would write to the wrong frame. So any buffer the GPU DMAs into or out of asynchronously must be **page-locked**: `cudaHostAlloc()`, `cudaHostRegister()`, or `DataLoader(..., pin_memory=True)`. Copies from pageable memory still work, but the driver stages them through an internal pinned bounce buffer — an extra copy, effectively synchronous, which is why pageable H2D bandwidth is typically well under half of pinned.

**What pinning does to the kernel.** Page-locking takes a long-term DMA pin — `FOLL_PIN | FOLL_LONGTERM` in the `pin_user_pages()` API (`Documentation/core-api/pin_user_pages.rst`). Those pages:

- are never scanned by reclaim (they land in `memory.stat`'s `unevictable`, per lesson 05);
- **cannot be migrated** — which means compaction cannot move them, `migratepages` cannot move them, and AutoNUMA cannot move them;
- must be migrated *out* of `ZONE_MOVABLE`/CMA regions at pin time, since those regions exist precisely to guarantee movability.

**Three consequences that only show up on GPU nodes:**

1. **A NUMA misplacement of a pinned buffer is permanent.** If a dataloader first-touches its staging buffer on node 0 and then pins it, nothing in the kernel will ever fix it: AutoNUMA will not migrate it, `migratepages` will fail on it. The only remedy is to restart the process with correct affinity. **Place deliberately, then pin** — run under `numactl --cpunodebind=N --membind=N`, or let Topology Manager `single-numa-node` do it.
2. **Pinned pages fragment physical memory against THP and hugetlb.** An unmovable page in the middle of a 2 MiB block prevents compaction from ever assembling that block. On a long-running node with many pin/unpin cycles this shows up as `thp_fault_fallback` and `compact_fail` climbing — THP failing not because memory is scarce but because it is *shredded*. It is a strong argument for pre-reserving an explicit hugetlb pool at boot, before any pinning has happened.
3. **Pinning is a hard cap on reclaimable memory.** 24 GiB of pinned staging buffers is 24 GiB reclaim structurally cannot touch, and `RLIMIT_MEMLOCK` (`ulimit -l`) may need raising in the container for the pin to succeed at all. With anonymous model memory and swap off alongside it, this is why a GPU pod's real headroom is far below what `free -m` suggests.

**Bandwidth arithmetic for the H2D path**, so you can tell whether a slow data path is a placement problem or a bandwidth problem:

```
GPU on PCIe Gen4 ×16:
  theoretical    = 16 GT/s × 16 lanes × (128/130) / 8 bits per byte = 31.5 GB/s per direction
  achievable, pinned host memory                                    ≈ 24-26 GB/s (typical)
  achievable, pageable host memory (driver bounce buffer)           ≈ 8-12 GB/s (typical)

Feeding 8 GPUs at 24 GB/s each simultaneously = 192 GB/s of host memory read traffic.
  Local socket DRAM bandwidth (8×DDR5-4800)   = 307 GB/s   → 63 % utilised. Fits.
  One UPI 2.0 link (≈48 GB/s/direction)       → 400 % oversubscribed. Does not fit.
```

That last line is the whole argument in one number. Even ignoring latency entirely, a misplaced pinned buffer tries to push four times a UPI link's bandwidth across it. The GPUs will simply wait. Measure your own with `bandwidthTest` from the CUDA samples, run once under `numactl --cpunodebind=<gpu's node> --membind=<gpu's node>` and once with the wrong node, and the delta is the cost of misplacement on your specific hardware.

## Perspectives

**Kernel-mechanism view.** Everything here descends from two hardware facts: a page-table entry is a scarce line in a TLB of a couple of thousand entries, and physical DRAM is not equidistant from every core or every PCIe root complex. THP, hugetlbfs, first-touch, mempolicy and AutoNUMA are all policy layered on those two constraints. The kernel's instinct throughout is to be lazy and reversible — allocate small, promote opportunistically, split under pressure, migrate when evidence arrives — which is right for a general-purpose machine and wrong for one whose correct placement is knowable in advance. That mismatch is why GPU nodes end up disabling AutoNUMA, pre-reserving hugetlb pools, and pinning statically: you are overriding adaptive heuristics with knowledge they cannot have.

**Operator/SRE view.** "Disable THP" is folklore inherited from database operations without the reasoning that produced it. The reasoning is `fork()`-based copy-on-write amplification plus synchronous compaction stalls — two mechanisms, both real, neither universal. The correct stance is not a blanket toggle but **`enabled=madvise` plus explicit opt-in**: set the host default to `madvise`, let each workload decide via `MADV_HUGEPAGE`, and watch `thp_fault_fallback` and `compact_stall` to know whether the decision costs anything. A training arena opts in and gets 512× TLB reach and 500× less page-table memory; a latency-sensitive sidecar never asks and never pays the compaction tax. Check `defrag` separately from `enabled` — the kernel default is `madvise`, some distros ship `defer+madvise`, and that difference decides whether an opted-in allocation stalls the faulting thread.

**GPU-fleet-specific view.** Nowhere else in the stack does a purely CPU-side kernel detail — which socket a buffer's pages landed on — directly gate the bandwidth of a fabric costing orders of magnitude more than the DRAM it is arguing about. The build recipe: pre-reserve hugetlb at boot with per-node counts; set `enabled=madvise` and let the framework opt in; enable CPU Manager `static` + Memory Manager `Static` + Topology Manager `single-numa-node` at `pod` scope, then disable `numa_balancing` because placement is now static by construction. Verification is always the same two commands: `nvidia-smi topo -m` to confirm GPU↔NIC is `NODE` or better, and `memory.numa_stat` to confirm the pod's pages landed where you asked.

**Economics view.** Price the loss instead of describing it. Take 40 GPU nodes × 8 GPUs = 320 accelerators and a sustained 15% loss on the CPU-side feed, meaning those GPUs idle roughly 15% of the time waiting on data. At an on-demand rate of order **$2–3/GPU-hour** (a dated 2026 snapshot; recompute against your own contract), that is `320 × 0.15 × 730 h/month × $2.50 ≈ $87,600/month` of paid-for-but-idle accelerator time — a five-figure monthly line item created by unpinned buffers and default 4 KiB pages. The counter-cost is real but small: Topology Manager `single-numa-node` will reject some pods that would otherwise have scheduled, so you trade a few percent of bin-packing efficiency for the throughput. Compute both sides before you argue for it — that is what makes it a roadmap argument rather than a kernel-internals curiosity.

## Real-world use cases

**Redis — "Diagnosing latency issues" (official documentation).** Redis persists by `fork()`ing a child that writes the dataset to disk while the parent keeps serving. Parent and child share every page copy-on-write. With 4 KiB pages, a write to one key copies 4 KiB. With THP enabled, that same write copies the entire **2 MiB** huge page. In a busy instance, a few event-loop iterations touch enough distinct keys to copy-on-write a large fraction of the whole dataset, producing both a multi-millisecond-to-multi-second latency spike and a memory spike far larger than the data actually modified. Redis's guidance is to set `enabled=never` and restart the process (a restart is needed because existing mappings keep their THP backing). **What it shows, and what it doesn't:** it is the textbook production instance of THP-as-latency-hazard, and it is *fork-specific*. A training process that allocates its arena and never forks afterwards does not have this failure mode — which is why the correct generalisation is `madvise` and opt-in, not blanket `never`. This is the vendor advice that created the folklore, read for its mechanism rather than its conclusion.

**Microsoft Azure AKS Engineering — "Optimizing RDMA performance for AI workloads on AKS with DRANET" (2026).** On an Azure ND GB300-v6 node — four NVIDIA GB300 GPUs and four ConnectX-8 InfiniBand NICs spread across two NUMA domains, with GPUs 0–1 and NICs 0–1 on node 0 and GPUs 2–3 and NICs 2–3 on node 1 — the post measures what NUMA alignment is worth. A `NODE` relationship between GPU and NIC (shared PCIe root complex) enables GPUDirect RDMA; a `SYS` relationship forces the transfer across the inter-socket link and disables GDR. The observable consequence is in NCCL's own channel allocation: **2 channels for a `SYS`-distant NIC, 8 for a NUMA-aligned NIC, and 16 with two aligned NICs.** The post's broader argument is that Dynamic Resource Allocation (via DRANET) lets you express this placement declaratively instead of through privileged containers and manual device management. Dated-2026 snapshot: the specific channel counts are point-in-time NCCL/driver behaviour on that SKU, but the mechanism — topology decides whether GDR is available, and GDR availability decides collective bandwidth — is durable.

**Kubernetes SIG-Node — Topology Manager policy options (`prefer-closest-numa-nodes` GA 1.32, `max-allowable-numa-nodes` GA 1.35).** The Topology Manager's hint-merging algorithm enumerates candidate NUMA node sets, and its cost grows steeply with node count — enough that Kubernetes hard-capped it at **8 nodes** and made raising the cap an explicit, documented-as-not-recommended opt-in. Separately, `prefer-closest-numa-nodes` exists because "align to a single node or give up" is too binary for large machines: when a workload genuinely does not fit on one node, you still want the *closest* set rather than an arbitrary one. **What this shows:** NUMA alignment is not free and does not always succeed. On any node with sub-NUMA clustering or many CXL memory tiers, check `numactl --hardware` for the node count before assuming `single-numa-node` will admit anything at all.

## Worked example

**Investigation: "training throughput dropped 18% after we rebalanced pods onto the bigger nodes."**

### Step 1 — establish what "bigger" changed

```
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0-31 64-95
node 0 size: 257862 MB
node 1 cpus: 32-63 96-127
node 1 size: 257993 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

The old nodes were single-socket. **The new ones are two-socket, and nothing in the workload's config knew that.** That is the entire hypothesis; the rest is confirming it.

```
$ for d in /sys/bus/pci/devices/0000:8{1,2,3,4}:00.0; do echo -n "$d "; cat $d/numa_node; done
/sys/bus/pci/devices/0000:81:00.0 1
/sys/bus/pci/devices/0000:82:00.0 1
/sys/bus/pci/devices/0000:83:00.0 1
/sys/bus/pci/devices/0000:84:00.0 1
$ cat /sys/class/net/ib0/device/numa_node
1
```

All four GPUs and the RDMA NIC are on **node 1**.

### Step 2 — find where the workload's memory actually is

```
$ kubectl get pod trainer-0 -o jsonpath='{.spec.containers[0].resources}'
{"limits":{"memory":"200Gi","nvidia.com/gpu":"4"},"requests":{"cpu":"32","memory":"200Gi"}}
```

Note: `requests.cpu: "32"` with no CPU *limit* means this pod is **Burstable**, not Guaranteed — so CPU Manager `static` would not give it exclusive cores even if it were enabled. Check whether it is:

```
$ grep -E 'cpuManagerPolicy|memoryManagerPolicy|topologyManagerPolicy|topologyManagerScope' \
    /var/lib/kubelet/config.yaml
cpuManagerPolicy: none
memoryManagerPolicy: None
topologyManagerPolicy: none
topologyManagerScope: container
```

Nothing is pinning anything. Now look at the placement that resulted:

```
$ PID=$(pgrep -f 'train.py' | head -1)
$ numastat -p $PID
Per-node process memory usage (in MBs) for PID 41277 (python)
                          Node 0          Node 1           Total
                 --------------- --------------- ---------------
Private                155904.12        38976.03       194880.15
----------------  --------------- --------------- ---------------
Total                 155904.12        38976.03       194880.15

$ grep -E '^anon ' /sys/fs/cgroup/kubepods.slice/.../memory.numa_stat
anon N0=163461201920 N1=40870641664
```

**80% of a 190 GiB resident set is on node 0. The GPUs and the NIC are on node 1.** Every H2D copy and every NCCL staging buffer crosses UPI.

Why? First-touch. The trainer allocates and zero-fills its arenas in `main()` before spawning dataloader workers, and the scheduler happened to place `main()` on a node-0 core. Confirm with `numa_maps`:

```
$ head -3 /proc/$PID/numa_maps
7f1c00000000 default anon=39845888 dirty=39845888 N0=33141760 N1=6704128 kernelpagesize_kB=4
7f4400000000 default anon=8388608 dirty=8388608 N0=8388608 kernelpagesize_kB=4
7f6800000000 default anon=2097152 dirty=2097152 N1=2097152 kernelpagesize_kB=4
```

Two things in that output: `default` policy (nobody called `mbind`), and `kernelpagesize_kB=4` on the big arenas — **no huge pages either.**

### Step 3 — quantify the second problem

```
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
$ grep AnonHugePages /proc/$PID/smaps_rollup
AnonHugePages:         0 kB
```

Host policy is `madvise`, and the framework never calls `madvise(MADV_HUGEPAGE)` — so the 190 GiB arena is entirely 4 KiB pages. Apply §2's arithmetic:

```
translations to cover the arena = 190 GiB / 4 KiB = 49,807,360
STLB reach at 4 KiB             = 2048 × 4 KiB    = 8 MiB  (0.004 % of the arena)
page-table memory               = 190 GiB / 512   = 380 MiB
```

380 MiB of page tables charged against a 200 GiB `memory.max`, and a TLB that covers four thousandths of one percent of the working set. Measure it rather than infer it:

```
$ perf stat -e dTLB-loads,dTLB-load-misses,dtlb_load_misses.walk_active,cycles -p $PID -- sleep 30

 41,882,901,229      dTLB-loads
  1,338,251,884      dTLB-load-misses          #    3.19% of all dTLB cache accesses
 61,204,993,110      dtlb_load_misses.walk_active
264,120,993,244      cycles
```

**23.2% of cycles had a page walk in flight.** That is not a rounding error, and it stacks on top of the remote-access latency: a walk that misses cache and resolves against *remote* DRAM pays ~140 ns instead of ~75 ns, so the two problems multiply rather than add.

### Step 4 — confirm the fabric-side consequence

```
$ nvidia-smi topo -m | head -6
        GPU0  GPU1  GPU2  GPU3  NIC0  CPU Affinity  NUMA Affinity
GPU0     X    NV18  NV18  NV18  PXB   32-63,96-127  1
GPU1    NV18   X    NV18  NV18  PXB   32-63,96-127  1

$ taskset -cp $PID
pid 41277's current affinity list: 0-127        # unpinned: can run anywhere

$ NCCL_DEBUG=INFO ... 2>&1 | grep -E 'GDRDMA|Channels|via NET'
NCCL INFO Channel 00/02 :    0   1   2   3
NCCL INFO NET/IB : GPU Direct RDMA Disabled for GPU 0 [0000:81:00.0] / HCA 0 'mlx5_0'
```

Two channels, GDR disabled. The GPU↔NIC relationship is fine (`PXB`), but the *process* is unpinned and its memory is on the wrong node, so NCCL's staging path crosses sockets and it scales back accordingly.

### Step 5 — fix, in priority order, and measure each

**(a) Locality first — it is the larger effect and the cheaper change.** For a bare run:

```
$ numactl --cpunodebind=1 --membind=1 python train.py
```

For the platform, the durable version is to make the pod Guaranteed (set CPU limits equal to requests) and turn on the alignment stack:

```yaml
# /var/lib/kubelet/config.yaml
cpuManagerPolicy: static
memoryManagerPolicy: Static
reservedMemory:
- numaNode: 0
  limits: {memory: 4Gi}
- numaNode: 1
  limits: {memory: 4Gi}
topologyManagerPolicy: single-numa-node
topologyManagerScope: pod
```

Verify after restart:

```
$ grep -E '^anon ' /sys/fs/cgroup/kubepods.slice/.../memory.numa_stat
anon N0=1073741824 N1=203281203200        # 99.5 % on node 1
$ taskset -cp $PID
pid 51882's current affinity list: 32-63
```

**(b) TLB second.** Either have the framework `madvise(MADV_HUGEPAGE)` its arena allocator, or — for a staging buffer whose size you control — move it to explicit hugetlb, which also removes it from THP's fragmentation lottery:

```
$ grep AnonHugePages /proc/$PID/smaps_rollup
AnonHugePages:  199229440 kB              # 190 GiB now THP-backed
$ perf stat -e dtlb_load_misses.walk_active,cycles -p $PID -- sleep 30
  4,881,204,112      dtlb_load_misses.walk_active
264,004,881,290      cycles                #  1.8 % of cycles, down from 23.2 %
```

**(c) Turn off what is now redundant.** `sysctl -w kernel.numa_balancing=0` — the placement is static and enforced, so AutoNUMA can only add fault and migration overhead on top of a solved problem, exactly as the kernel documentation says for bound workloads. It could not have helped anyway once the staging buffers were pinned (§8).

### Step 6 — the root-cause read

The regression was never "bigger nodes are slower." It was that bigger nodes are **multi-socket**, the workload had no NUMA policy, first-touch put 80% of its memory on the socket without the GPUs, and the same absence of policy meant the arena never got huge pages either. Two independent taxes — cross-socket bandwidth and TLB walk cycles — compounding, plus a third-order effect where NCCL detected the misalignment and cut its channel count. The fix is a kubelet configuration change and one `madvise` call; at fleet scale (see the economics perspective) the same misconfiguration replicated across every pod scheduled the same way is the 10–20% dataloader loss that shows up as a five-figure monthly line item.

## Practice

**Environment:** any Linux box; ideally a two-socket server, but a laptop or VM still shows most mechanisms (it will report one NUMA node — record that and reason explicitly about what a two-socket box would show). You need root for the sysfs writes in steps 4 and 7.

1. **Read the TLB out of your own CPU and compute its reach.** `lscpu | grep -iE 'model name|numa'`, then read the L1 dTLB and L2 STLB entry counts (`cpuid -1 | grep -iA3 TLB`, or the vendor's optimization manual). Compute reach at 4 KiB and 2 MiB, pick a working-set size you care about, and state what fraction of it the STLB covers at each page size.
2. **THP state and policy.** `cat /sys/kernel/mm/transparent_hugepage/enabled` and `.../defrag`; record the `[bracketed]` value for both. State whether `defrag` matches the kernel default (`madvise`) or your distro overrode it, and say in one sentence what the difference means for a process that calls `MADV_HUGEPAGE` under fragmentation.
3. **Watch THP happen.** Snapshot `grep -E 'thp_fault_alloc|thp_fault_fallback|thp_collapse_alloc' /proc/vmstat` and `grep AnonHugePages /proc/meminfo`. Run a memory hog (`stress-ng --vm 1 --vm-bytes 4G --vm-hang 120`). From another shell read `grep AnonHugePages /proc/<pid>/smaps_rollup` and re-read the vmstat counters. **Record the deltas** and say whether the kernel promoted at fault time, fell back, or collapsed later.
4. **Reserve explicit hugepages.** `sysctl -w vm.nr_hugepages=64`, then `grep -i huge /proc/meminfo` and `cat /sys/devices/system/node/node*/meminfo | grep -i huge`. Note `HugePages_Total`/`Free`/`Rsvd`/`Hugetlb`, then set it back to 0. **Write one sentence on why `AnonHugePages` did not change.**
5. **NUMA topology.** `numactl --hardware`. Record node count, per-node sizes, and the full distance matrix. On a single-node box say so explicitly, and note what a `10 / 21` matrix would mean and why `21` is not a literal 2.1× latency.
6. **Pin, and prove placement three ways.** `numactl --cpunodebind=0 --membind=0 stress-ng --vm 1 --vm-bytes 2G --vm-hang 60 &`, then confirm with (a) `numastat -p <pid>`, (b) `head /proc/<pid>/numa_maps` — noting the policy field, the `N0=`/`N1=` counts and `kernelpagesize_kB` — and (c) `memory.numa_stat` if it is in a cgroup. With two nodes, repeat with `--membind=1`.
7. **Device affinity.** `cat /sys/class/net/<iface>/device/numa_node` and `.../local_cpulist`. With a GPU, add `cat /sys/bus/pci/devices/<BDF>/numa_node` and `nvidia-smi topo -m`, and name one GPU↔NIC relationship and what its symbol means. **Flag any `-1` as unverified and cross-check with `lstopo-no-graphics`.**
8. **(Stretch) AutoNUMA.** Record `cat /proc/sys/kernel/numa_balancing`, and if available `numa_pte_updates` / `numa_hint_faults` / `numa_pages_migrated` from a cgroup's `memory.stat` under load. One sentence, citing the kernel documentation's own wording, on why a `single-numa-node` GPU node wants this off.
9. **(Stretch) Measure the TLB effect.** `perf stat -e dTLB-loads,dTLB-load-misses,cycles` on a random-access benchmark over a multi-GiB buffer, once with `MADV_HUGEPAGE` (or `THP=always`) and once with `THP=never`. Report the miss-rate delta, and if your CPU exposes `dtlb_load_misses.walk_active`, walk cycles as a fraction of total cycles for both runs — that is the number that persuades people.

**Acceptance:** a short note (≤2 pages) that (1) maps this machine's NUMA topology — node count, per-node sizes, distance matrix — and states which node your primary NIC and GPU (if present) live on, flagging any `-1` as unverified; (2) records the THP `enabled` **and** `defrag` state plus an observed `AnonHugePages` value under load, with the `thp_fault_alloc` / `thp_fault_fallback` deltas showing *how* it was obtained; (3) shows the hugetlb pool before and after a reservation, with `HugePages_Total`/`Free`/`Rsvd`, and states explicitly that this pool is separate from `AnonHugePages`; (4) contains a **worked TLB-reach calculation for this CPU** at 4 KiB and 2 MiB against a working-set size you name; and (5) contains one concrete sentence — *"For a GPU data path, this matters because ___"* — naming the cross-node hop you would avoid, the pinning you would apply (`numactl` flags or the exact kubelet policy settings), and how you would verify it landed (`memory.numa_stat` or `numastat -p`). Feeds ["Anatomy of a Container"](../practice/anatomy-of-a-container/README.md) as the NUMA/hugepage half of the node-level diagnostic toolkit.

## Common pitfalls

1. **"Disable THP" as a blanket rule** inherited from database folklore and applied uncritically to ML training. *Mechanism:* the database hazard is `fork()` copy-on-write amplification (one byte written copies 2 MiB) plus synchronous compaction stalls. A training process that allocates its arena once and never forks has the first hazard structurally absent, and the second is controlled by `defrag`, not `enabled`. Blanket `never` throws away 512× TLB reach and 500× less page-table memory to avoid a problem the workload does not have.
2. **Confusing `AnonHugePages` with `HugePages_Total`.** *Mechanism:* they are different subsystems with different allocators, different lifetimes and different accounting. A node can have 40 GiB of THP and a zero-size hugetlb pool. Kubernetes advertises `hugepages-2Mi` capacity **only** from the hugetlb pool; no amount of THP will make a pod requesting `hugepages-2Mi` schedulable.
3. **Reading `HugePages_Free` as "available."** *Mechanism:* `HugePages_Rsvd` counts pages already committed to a process that has mapped but not yet faulted them. Available is `Free − Rsvd`. Provisioning against `Free` will produce mysterious `mmap` failures at fault time rather than at allocation time.
4. **Assuming `numactl --hardware`'s distance numbers are literal latency multipliers.** *Mechanism:* they are a normalised firmware-reported SLIT index with local defined as 10, not a measurement. Real deltas run 1.5–1.8×, and SLIT models no contention at all — under concurrent cross-socket load the effective penalty is worse than the static number implies.
5. **Trusting `numa_node` sysfs files unconditionally.** *Mechanism:* `-1` means the firmware did not report affinity (missing or incomplete ACPI `_PXM`), not "no affinity" or "any node is fine." Tools that read it will silently conclude there is nothing to align. Cross-check with `lstopo` or `nvidia-smi topo -m`.
6. **Reserving 1 GiB pages at runtime and expecting them to appear.** *Mechanism:* a 1 GiB page needs a physically contiguous gigabyte, and unmovable allocations (kernel slab, page tables, DMA pins) shred physical memory within minutes of uptime; compaction cannot move them. The write to `nr_hugepages` succeeds while delivering fewer pages than requested — **always read the value back** — and the fix is boot-time reservation via `hugepagesz=1G hugepages=N`.
7. **Assuming Kubernetes hugepage requests overcommit like memory.** *Mechanism:* they are pre-allocated frames, not a virtual budget. Request must equal limit, the node must have the pool reserved before the pod schedules, and the pages do not count against the container's `memory` limit — so budget node RAM for both dimensions independently.
8. **Expecting AutoNUMA or `migratepages` to fix a misplaced GPU staging buffer.** *Mechanism:* once page-locked for DMA it holds a `FOLL_LONGTERM` pin and is unmigratable by construction. Placement must be correct **before** the pin, which means the right `numactl`/Topology Manager settings from process start.
9. **Being surprised by `TopologyAffinityError` after enabling `single-numa-node`.** *Mechanism:* that is the policy working — no single node could satisfy CPUs + memory + devices together. It is information about fragmented node capacity, not a bug. Dropping to `best-effort` to silence it reintroduces exactly the misalignment you enabled it to prevent.

## Self-check

**(a) When is THP a throughput win, and when is it a latency hazard? Name the mechanisms, not just the workloads.**

**Answer:** It is a **throughput win** for large, long-lived, densely-accessed anonymous arenas — ML training staging buffers, tensor arenas, in-memory feature stores, big JVM/Go heaps, analytics scratch. Three compounding mechanisms: one page fault per 2 MiB instead of 512; one TLB entry covering 2 MiB instead of 4 KiB, which on a 2048-entry STLB raises reach from 8 MiB to 4 GiB; and page-table memory falling by ~500× (a 40 GiB arena needs 80 MiB of PTEs at 4 KiB versus 160 KiB of PMDs at 2 MiB) — memory that is charged to `memory.max` and counted by `oom_badness()`. It is a **latency hazard** through two separate mechanisms. (1) *Allocation-time stalls*: under fragmentation, faulting a 2 MiB page with `defrag=always`, or with `defrag=madvise` in an `MADV_HUGEPAGE` region, triggers synchronous direct reclaim and compaction on the faulting thread — multi-millisecond pauses, counted by `compact_stall` in `/proc/vmstat`. (2) *Copy-on-write amplification*: after `fork()`, a one-byte write to a shared THP copies the whole 2 MiB, which is why a Redis background save can copy-on-write most of the dataset and spike both latency and RSS. The second mechanism is fork-specific and therefore absent from most training workloads. The right stance is `enabled=madvise` with explicit `MADV_HUGEPAGE` opt-in, and watching `thp_fault_fallback` and `compact_stall` to know whether the choice is costing anything.

**(b) Why does cross-NUMA memory access hurt a GPU data-loading path, and how do you pin to avoid it?**

**Answer:** A GPU and its RDMA NIC sit behind one socket's PCIe root complex, i.e. one NUMA node. The kernel's default placement policy is **first-touch** — a page lands on the node of the CPU that first *writes* it, not the one that allocated it — so an unpinned dataloader that initialises its arenas on whichever core the scheduler picked will scatter them, typically onto the wrong node. Every host-to-device copy then makes the GPU's DMA engine read remote DRAM: ~140 ns instead of ~75 ns idle latency, and more importantly across an inter-socket link of roughly 48 GB/s per direction versus 307 GB/s of local DRAM bandwidth. Feeding eight GPUs at ~24 GB/s each needs 192 GB/s, which fits comfortably in local DRAM and oversubscribes a UPI link fourfold. NCCL compounds it: with the NIC judged `SYS`-distant it disables GPUDirect RDMA, stages through host memory, and scales its channel count down (2 versus 8 on a measured ND GB300-v6 node). **The fix** is to co-locate threads, buffers and devices: `numactl --cpunodebind=N --membind=N` for a bare process, or in Kubernetes make the pod Guaranteed and enable CPU Manager `static` + Memory Manager `Static` + Topology Manager `single-numa-node` at `pod` scope, so the kubelet allocates CPUs, memory, hugepages and the GPU from one node. **Verify** with `memory.numa_stat` or `numastat -p <pid>` for the memory, `taskset -cp` for the threads, and `nvidia-smi topo -m` plus `NCCL_DEBUG=INFO` for the fabric. And do it *before* any buffer is pinned — see (d).

**(c) Explicit huge pages vs THP — when do you reserve, and how does Kubernetes expose them?**

**Answer:** THP is best-effort, transparent, splittable and swappable. It can fail to allocate under fragmentation (`thp_fault_fallback`), be split back to 4 KiB under memory pressure (`thp_deferred_split_page`, `thp_underused_split_page`), and it lives in ordinary reclaimable memory. You reserve **explicit** huge pages from the separate HugeTLB pool when you need *guaranteed, non-splittable, non-swappable* pages with stable physical backing: DPDK/SPDK packet buffers, RDMA and GPUDirect staging regions, QEMU guest RAM. Reserve via `vm.nr_hugepages` or the per-size/per-node sysfs tree for 2 MiB; **1 GiB pages must be reserved at boot** (`default_hugepagesz=1G hugepagesz=1G hugepages=16`) because assembling a physically contiguous gigabyte becomes impossible once unmovable allocations have fragmented memory. Kubernetes exposes the pre-reserved pool as a schedulable resource: the kubelet advertises `hugepages-2Mi` / `hugepages-1Gi` in Capacity and Allocatable; pods request them under `resources.limits` with **requests forced to equal limits**; there is **no overcommit**, so the node must have the pool before the pod can schedule; the pages **do not count against the container's `memory` limit** (the hugetlb cgroup controller accounts them separately unless the hierarchy is mounted with `memory_hugetlb_accounting`); and the container backs them with an `emptyDir{medium: HugePages}` volume, or `medium: HugePages-<size>` when using more than one size. A `memory` or `cpu` request must accompany the hugepage request, and dynamically added pages require a kubelet restart to be discovered.

**(d) Why would you disable `numa_balancing` on a GPU training node — and why would it not have saved you anyway once buffers are pinned?**

**Answer:** AutoNUMA works by periodically unmapping pages, catching the resulting minor faults, inferring which node is actually touching each page, and migrating it there. It is reactive by construction — it corrects a misplacement only *after* that misplacement has already cost cycles — and the correction itself (unmap, fault, allocate, copy, remap) is expensive and adds jitter. The kernel's own documentation is explicit: *"If the target workload is already bound to NUMA nodes then this feature should be disabled."* On a GPU node running CPU Manager `static` plus Topology Manager `single-numa-node`, the correct placement is known and enforced at admission time, so AutoNUMA has nothing to discover and contributes only overhead. **The second, stronger reason:** the buffers that matter most on a GPU node are page-locked for DMA (`cudaHostAlloc`, `cudaHostRegister`, `pin_memory=True`), which takes a long-term pin (`FOLL_PIN | FOLL_LONGTERM`). Pinned pages are unmigratable — AutoNUMA cannot move them, `migratepages` cannot move them, and compaction cannot move them. So on exactly the memory whose placement determines your H2D bandwidth, AutoNUMA is not merely unhelpful, it is inapplicable. Placement has to be right before the pin is taken; nothing downstream can repair it. As a bonus, those same unmovable pins are why `thp_fault_fallback` and `compact_fail` climb on long-lived GPU nodes — pinned pages shred physical memory against 2 MiB block assembly, which is an argument for pre-reserving an explicit hugetlb pool at boot.

**(e) Your STLB has 2048 entries. Your model's arena is 96 GiB. What does that tell you, and what would you change?**

**Answer:** At 4 KiB pages the STLB covers `2048 × 4 KiB = 8 MiB` — **0.008%** of the arena — so a randomly-accessed working set misses on essentially every touch, and each miss costs a walk of up to four dependent memory loads. Page-table memory is `96 GiB / 512 = 192 MiB`, charged to the cgroup and counted by `oom_badness()`. At 2 MiB pages, reach becomes `2048 × 2 MiB = 4 GiB` (4.2% of the arena — still mostly missing, but the walk is one level shorter) and page tables collapse to `96 GiB / 512 / 512 = 384 KiB`. At 1 GiB pages you need 96 translations total, which fit in the TLB hierarchy, and translation cost effectively disappears. **What to change, in order:** confirm with `perf stat -e dTLB-load-misses,dtlb_load_misses.walk_active,cycles`, treating walk-active cycles as a fraction of total cycles as the real evidence; get the arena onto 2 MiB pages via `MADV_HUGEPAGE` (cheap, no node config); and if it is a fixed-size staging region rather than a general heap, move it to a boot-reserved 1 GiB hugetlb pool — which also removes it from THP's fragmentation lottery and from the unmovable-pin problem. Re-measure after each step; if walk-active cycles do not move, the bottleneck was never translation and you should be looking at NUMA placement or PCIe bandwidth.

## Connections & what's next

This lesson builds on lesson 05's memory accounting in two places: the `pagetables` field in `memory.stat` (whose 500× collapse under 2 MiB pages is one of the two arguments for huge pages), and the `unevictable` field (the long-term DMA pins that make placement irreversible and fragment memory against THP allocation). It also builds on lesson 03's cgroups v2 — the same `resources.limits`/`requests` machinery that enforces `memory.max` advertises `hugepages-2Mi`/`hugepages-1Gi`, though through a *separate* hugetlb controller that `memory.max` never sees. Sideways it connects to the Kubernetes alignment stack — CPU Manager `static`, Memory Manager `Static`, Topology Manager `single-numa-node` — the orchestration layer that turns "the GPU and NIC are on node 1" into an enforced admission decision. Next, lesson 07 (**Networking datapath & conntrack**) moves to the next hop in the pipeline: how packets actually move through the kernel's networking stack, and where a node-global connection-tracking table becomes the same kind of shared, silently-degrading resource for egress traffic that TLB reach and NUMA placement are for host memory.

## References & further reading

**Primary sources**

- **Transparent Hugepage Support — kernel admin guide** — <https://docs.kernel.org/admin-guide/mm/transhuge.html> — the authoritative reference for the `enabled` and `defrag` value sets, every `khugepaged` tunable, mTHP and the per-size sysfs tree, and the full `/proc/vmstat` THP counter list. *Correction applied from this source:* the kernel's documented default for `defrag` is **`madvise`**, not `defer+madvise` (which some distributions ship); and `enabled=never` does **not** globally disable THP, because `madvise(MADV_COLLAPSE)` ignores the setting.
- **HugeTLB Pages — kernel admin guide** — <https://docs.kernel.org/admin-guide/mm/hugetlbpage.html> — exact definitions of `HugePages_Total`/`Free`/`Rsvd`/`Surp`, `Hugepagesize` and `Hugetlb`; the per-size and per-node sysfs tree; boot-parameter semantics for `hugepagesz`/`hugepages`/`default_hugepagesz` including the node-list form; and the `demote`/`demote_size` interface. Read it once for the `Rsvd` semantics alone, which is the field that turns "we had free hugepages" into a fault-time failure.
- **`/proc/meminfo` field reference — `Documentation/filesystems/proc.rst`** — <https://docs.kernel.org/filesystems/proc.html> — definitions of `AnonHugePages`, `ShmemHugePages`, `ShmemPmdMapped`, `FileHugePages`, `FilePmdMapped` and `DirectMap4k/2M/1G`, plus the `/proc/<pid>/numa_maps` output format used in the worked example.
- **NUMA Memory Policy — kernel admin guide** — <https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html> — the complete mempolicy mode list with fallback semantics, the `MPOL_F_STATIC_NODES` / `MPOL_F_RELATIVE_NODES` flags, and how policies interact with cpusets.
- **`Documentation/admin-guide/sysctl/kernel.rst`, `numa_balancing`** — <https://docs.kernel.org/admin-guide/sysctl/kernel.html> — the bitmask values (`0` disabled, `1` NORMAL, `2` MEMORY_TIERING), how the unmap-and-fault sampling works, and the kernel's explicit recommendation that the feature be disabled for workloads already bound to NUMA nodes.
- **`Documentation/core-api/pin_user_pages.rst`** — <https://docs.kernel.org/core-api/pin_user_pages.html> — `FOLL_PIN` and `FOLL_LONGTERM` semantics: why DMA-pinned pages must not be migrated, and why long-term pins are the strictest case. The kernel-side explanation for why a misplaced pinned GPU staging buffer can never be repaired in place.
- **Kubernetes — Manage HugePages; Topology Manager; CPU Management Policies** — <https://kubernetes.io/docs/tasks/manage-hugepages/scheduling-hugepages/> and <https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/> — the `hugepages-<size>` resource rules (requests equal limits, no overcommit, separate from the `memory` limit, `emptyDir{medium: HugePages-<size>}`), and the Topology Manager scope/policy matrix plus the `prefer-closest-numa-nodes` (GA 1.32) and `max-allowable-numa-nodes` (GA 1.35, default 8) policy options.

**Real-world engineering**

- **Redis — "Diagnosing latency issues"** — <https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/> — the `fork()`/copy-on-write mechanism behind "disable THP for Redis": with THP, one byte written after fork copies a full 2 MiB page, so a background save copy-on-writes a large fraction of the dataset. Read it for the mechanism, then note that it is fork-specific and does not generalise to a training process that never forks after allocation.
- **Microsoft Azure AKS Engineering — "Optimizing RDMA performance for AI workloads on AKS with DRANET" (2026)** — <https://blog.aks.azure.com/2026/04/01/dranet-rdma-optimization-for-ai-on-aks> — measured NCCL channel counts as a function of GPU↔NIC NUMA alignment on an ND GB300-v6 node: 2 channels `SYS`-distant, 8 NUMA-aligned, 16 with two aligned NICs, with GPUDirect RDMA available only in the aligned cases. Dated 2026 snapshot — the counts are point-in-time NCCL/driver behaviour on that SKU; the topology→GDR→bandwidth mechanism is durable.

**Deeper dives**

- **hwloc / `lstopo` and `numactl(8)`** — <https://www.open-mpi.org/projects/hwloc/> — the tools that render and control NUMA plus PCIe topology. `numactl --hardware` gives node distances; `lstopo` shows *which PCIe link* connects a GPU or NIC to which node, which is the affinity data that actually drives a pinning decision — and the cross-check you need whenever a `numa_node` sysfs file reads `-1`.
