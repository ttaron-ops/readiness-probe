---
lesson: "01b.5"
title: "Memory, Reclaim, and the OOM Killer"
module: "01b"
concept: "Memory, Reclaim, and the OOM Killer"
status: not-started
est_time: "7h"
prev: "04-psi.md"
next: "06-hugepages-thp-numa.md"
artifacts: []
sources: 9
---

# 01b.5 · Memory, Reclaim, and the OOM Killer

> **Concept.** Virtual vs resident vs working set, the page cache and the reclaim path, and how the kernel chooses a victim — cgroup-scoped or global — when memory runs out.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)
>
> **Advanced Learning** — [Four Numbers and Three Kills](../../../learning/05-memory-and-oom.html): the four quantities called “memory used” side by side, and the QoS bias arithmetic that decides a global OOM. Optional visual companion; this lesson stays the source of truth.

## Where this fits

Lesson 04 gave you PSI: the stall-time signal that shows a workload being *denied* a resource, including the specific pre-OOM thrash signature — memory `full` pressure climbing while a cgroup burns wall-clock time in direct reclaim instead of doing work. That lesson deliberately stopped short of the endgame: what happens when reclaim can't keep up at all. This lesson picks up exactly there. It covers the reclaim path in full, the scoring function the kernel runs when reclaim fails, and the artifact — the `dmesg` OOM report — that tells you precisely who died and why. Once you can read that report cold, the next lesson (06, Hugepages/THP/NUMA) moves to the other half of the CPU-side memory story: not "did we run out," but "is the memory we *do* have being served to the GPU data path efficiently."

## Why this matters

A model-serving pod on a GPU node gets `OOMKilled`. The dashboard shows the node had 40 GB "free." The team blames the node, cordons it, pages SRE. All of that is wrong, and knowing *why* is a senior-vs-staff line. The pod died because **its own cgroup** hit `memory.max`, not because the node ran out; the node's 40 GB of "free" was mostly reclaimable page cache the kernel would have handed over instantly. The kernel wrote a complete post-mortem to `dmesg` naming the victim, its anonymous RSS, its `oom_score_adj`, and whether the kill was cgroup-scoped or global — and nobody read it.

On a GPU fleet the failure modes are asymmetric and expensive:

- A **fat model process** (a training job that grows its resident set past its limit) gets OOM-killed mid-epoch — a lost checkpoint and hours of idle GPU.
- A **bad limit** set below the model's true working set turns every run into a guaranteed cgroup-OOM.
- A **global** OOM on a shared node can take down the kubelet or the GPU device plugin if critical daemons aren't protected — turning one bad pod into a node outage.
- **Pinned host buffers** (the `pin_memory=True` staging buffers every CUDA data path uses) are unreclaimable by construction, so a GPU node's headroom is structurally thinner than its `free -m` output suggests.

Reading the OOM report cold, and knowing the difference between cgroup and global OOM, is the difference between "restart the pod, raise the limit 20%" and a two-hour incident. It is also, as the economics perspective below quantifies, the difference between losing minutes and losing hours of GPU-hours on every kill.

## What's new here (calibration)

Per the module README's calibration, you already know `free -m`, `kubectl top pod`, "the container hit its memory limit," and `OOMKilled` in `kubectl describe` — that operator-level vocabulary is not re-taught here. What's genuinely new at this depth:

- The **four distinct meanings of "memory used"** (virtual, resident, cached, working set), the exact arithmetic Kubernetes uses for working set, and why it deliberately picks the least intuitive one for limits and eviction.
- The **reclaim path as a state machine** — watermarks, `kswapd`, direct reclaim, the LRU lists, refault detection, and the sixteen-retry budget the memcg charge path spends before it gives up — not just "it kills things when full."
- **`oom_badness()` as an actual scoring function** you can read out of `mm/oom_kill.c`, predict, and bias — including what its units really are, which is not what most write-ups claim.
- The **cgroup-vs-global OOM distinction and its exact dmesg tell**, plus `memory.oom.group`, which on a modern Kubernetes node is already on by default and which you therefore need to reason about rather than "consider enabling."
- The **exact `oom_score_adj` values Kubernetes assigns per QoS class**, straight from the kubelet source, and the formula for the Burstable middle tier.
- The **economics of a mid-epoch kill** — translating "the pod restarted" into a GPU-hours number a staff engineer can put in a design doc.

## Core concepts

### 1. The problem memory management exists to solve

A process asks for memory in units it invented: `malloc(8 GiB)`, `mmap` a 200 GiB file, reserve a CUDA unified-memory range. The machine has a fixed number of physical page frames — on a 512 GiB node with 4 KiB pages, exactly 134,217,728 of them. Three facts create everything that follows:

1. **Programs ask for far more address space than they will ever touch.** A Go runtime reserves large arenas up front; a CUDA process maps device apertures tens of gigabytes wide; a JVM reserves its max heap. Backing all of that with physical frames at request time would waste most of the machine.
2. **The kernel can reconstruct some pages and not others.** A clean page of a file on disk can be dropped and re-read later at the cost of one I/O. A page of your model's activations exists nowhere else in the universe — the only place to put it is swap, and if there is no swap, nowhere.
3. **Allocation failure has no good answer.** `malloc` returning NULL is a fiction on Linux by default: the kernel overcommits, so the failure surfaces later, at *fault* time, in a context (a page fault deep inside a memcpy) where there is nothing sensible to return to the caller.

The kernel's answers, in order: **lazy allocation** (hand out virtual address space, attach physical pages only on first touch), **reclaim** (when frames run short, take back the cheapest ones), and — when reclaim can no longer make progress — **the OOM killer** (pick a process and SIGKILL it so the machine keeps running). Everything in this lesson is one of those three mechanisms, or the accounting that lets you see them.

**Lazy allocation, concretely.** When a process calls `mmap(NULL, 8<<30, PROT_READ|PROT_WRITE, MAP_ANONYMOUS|MAP_PRIVATE, -1, 0)`, the kernel creates a **VMA** (`struct vm_area_struct`) — a record saying "addresses X through X+8 GiB are readable/writable anonymous memory" — and returns. Zero physical pages have been allocated. The page-table entries for that range are empty. The first time the process writes to an address inside it, the CPU's MMU finds no valid translation, raises a **page fault**, and the kernel's fault handler allocates one 4 KiB frame, zeroes it, and installs a PTE. This is why `VmSize` (the sum of VMA lengths) and `VmRSS` (frames actually installed) diverge so wildly, and why a limit set on the wrong one is meaningless.

You can watch this directly:

```
$ grep -E 'VmSize|VmRSS|RssAnon|RssFile|RssShmem' /proc/self/status
VmSize:	   12864 kB
VmRSS:	    4224 kB
RssAnon:	 512 kB
RssFile:	3712 kB
RssShmem:	   0 kB
```

`VmSize` is a promise; `VmRSS` is a bill. The three `Rss*` lines are the breakdown the OOM report will also print, so learn to read them here.

### 2. The four numbers people call "memory used"

| Name | Where you read it | What it counts | Why it misleads |
|---|---|---|---|
| **Virtual** (VSZ, `VmSize`) | `ps`, `/proc/<pid>/status`, `total-vm` in the OOM report | Sum of all VMA lengths — every mapping, backed or not | Includes reservations never touched. A CUDA process routinely shows tens of GB of VSZ against a few GB resident. **Never budget on this.** |
| **Resident** (RSS, `VmRSS`) | `ps`, `/proc/<pid>/status`, `rss` in the OOM task table | Frames currently mapped into this process: `RssAnon + RssFile + RssShmem` | Counts shared pages once per sharer (two processes mapping the same 1 GiB library both report it), and counts file pages the kernel would drop for free. |
| **Page cache** | `Cached` / `Buffers` in `/proc/meminfo`, `file` in `memory.stat` | Kernel-held copies of file contents | Shows as "used" in naive tooling but is mostly **reclaimable**. Clean pages drop instantly; dirty pages drop after writeback. |
| **Working set** | `container_memory_working_set_bytes`, kubelet summary API | `memory.current − inactive_file` (cgroup v2) | The one number that predicts an OOM kill, and the only one Kubernetes actually acts on. |

**The working-set formula is worth memorising exactly, because nearly every blog post gets it wrong.** cAdvisor — which the kubelet embeds — computes it as *total cgroup memory usage minus the `inactive_file` counter*, floored at zero. Not `anon + active_file`. The difference matters: usage includes kernel memory charged to the cgroup (`slab`, `kernel_stack`, `pagetables`, `sock`, `percpu`), and those are *in* the working set. Subtracting only `inactive_file` means the kernel's own "this file cache looks cold" verdict is the single thing excluded. So:

```
working_set = memory.current − memory.stat[inactive_file]
```

That is what `kubectl top pod` shows, what the kubelet compares against `memory.available` for node-pressure eviction, and what your dashboards are graphing when they say "memory usage."

**Reading `free -m` correctly.** The classic mistake is reading `used`:

```
$ free -m
              total        used        free      shared  buff/cache   available
Mem:          64000       12000        2100         300       49900       50800
Swap:             0           0           0
```

`free` is 2.1 GB and looks alarming; `available` is 50.8 GB and is the number that matters. `available` comes from `MemAvailable` in `/proc/meminfo`, which the kernel computes from `MemFree`, the size of the file LRU lists, and `SReclaimable`, minus the per-zone low watermarks and a reserve for page cache the system needs to function (`Documentation/filesystems/proc.rst`). It is the kernel's own estimate of how much a new allocation can get **without swapping**. `buff/cache` being large is not a problem; it is the kernel doing its job.

`Swap: 0` in that transcript is not incidental — it is the Kubernetes default, and it changes the entire shape of this lesson. Hold that thought for §5.

**Worked math: what page tables themselves cost.** Page tables are memory too, they are charged to your cgroup (`memory.stat`'s `pagetables` field), and `oom_badness()` counts them against you. On x86-64 each page-table entry is 8 bytes and each table page is 4 KiB, so one PTE page holds 512 entries and maps 512 × 4 KiB = **2 MiB** of address space. Therefore:

```
leaf PTE cost  = RSS / 512                      (one 4 KiB table page per 2 MiB mapped)
PMD cost       = RSS / 512 / 512  = RSS / 262144   (one table page per 1 GiB)
PUD cost       = RSS / 512^3      ≈ negligible

For a 40 GiB resident training process, 4 KiB pages:
  leaf   = 40 GiB / 512      = 80 MiB
  PMD    = 40 GiB / 262144   = 160 KiB
  PUD    ≈ 0
  total  ≈ 80.2 MiB of page tables
```

80 MiB you did not budget for, per process, entirely invisible in RSS-based dashboards but fully charged to `memory.max`. Fork a dataloader pool with 16 workers that each map the same shared arena and you pay it 16 times unless the mapping is shared with shared page tables. This is also the number that collapses when you use 2 MiB pages — the same 40 GiB needs only the PMD level, 160 KiB, a **500× reduction** — which is one of the two reasons lesson 06 cares about page size.

### 3. Where the kernel keeps the truth: the cgroup v2 memory controller

Every container's memory reality is six or seven files in one directory. On a Kubernetes node with the systemd cgroup driver the path looks like `/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<uid>.slice/cri-containerd-<id>.scope/`.

| File | Type | Default | Semantics (cgroup-v2 admin guide) |
|---|---|---|---|
| `memory.current` | read-only | — | Total memory charged to this cgroup **and its descendants**, in bytes. |
| `memory.min` | rw | `0` | **Hard protection.** Memory below this is never reclaimed under any conditions. If nothing unprotected is reclaimable, the OOM killer runs instead. |
| `memory.low` | rw | `0` | **Best-effort protection.** Reclaimed only when there is nothing reclaimable in unprotected cgroups. Above the boundary, reclaim pressure is proportional to the overage. |
| `memory.high` | rw | `max` | **Throttle.** Going over puts the cgroup's processes under heavy reclaim pressure *and sleeps them*. It **never** invokes the OOM killer; usage may exceed it. |
| `memory.max` | rw | `max` | **Hard limit.** If usage reaches it and reclaim can't bring it down, the OOM killer runs **inside the cgroup**. This is what a Kubernetes `limits.memory` becomes. |
| `memory.peak` | rw | — | High-water mark of `memory.current` since creation (or since you last reset it by writing to the same fd). The right-sizing tool. |
| `memory.oom.group` | rw | `0` | If `1`, the OOM killer treats the cgroup as indivisible: every task in it (and descendants) is killed together, except tasks with `oom_score_adj == -1000`. |
| `memory.events` | read-only | — | Hierarchical counters, below. |
| `memory.events.local` | read-only | — | Same fields, but non-hierarchical — events that happened *at this cgroup*, not in descendants. |
| `memory.stat` | read-only | — | ~60 flat keys breaking usage down by type and counting reclaim activity. |
| `memory.numa_stat` | read-only | — | The same keys, split per NUMA node: `type N0=<bytes> N1=<bytes>`. Lesson 06 lives here. |
| `memory.swap.max` | rw | `max` | Swap ceiling for the cgroup. Kubernetes sets this to `0` unless the `NodeSwap` feature gate is on. |
| `memory.reclaim` | write-only | — | Proactive reclaim: `echo "1G" > memory.reclaim`. Accepts a `swappiness=` nested key. |
| `memory.pressure` | rw | — | PSI for this cgroup (lesson 04). Rising `full` is the pre-OOM signal. |

**`memory.events` — the six fields and exactly what each counts.** These are the fingerprints you match against symptoms.

| Field | Increments when |
|---|---|
| `low` | The cgroup was reclaimed despite being under its `memory.low` boundary — i.e. the system was desperate enough to breach best-effort protection. Usually means `low` is over-committed across siblings. |
| `high` | Processes in the cgroup were **throttled and routed into direct reclaim** because usage exceeded `memory.high`. For a cgroup deliberately capped by `high`, this counting up is expected, not alarming. |
| `max` | Usage was **about to go over** `memory.max`. If direct reclaim then fails to bring it down, the cgroup goes to OOM. A large `max` with zero `oom_kill` means reclaim is saving you — every time, at a cost you can see in PSI. |
| `oom` | Usage reached the limit and an allocation was about to fail. Not raised for allocations where the OOM killer isn't an option (failed high-order allocations, callers that asked not to retry). |
| `oom_kill` | **Processes belonging to this cgroup killed by any OOM killer.** This is your cgroup-OOM fingerprint; it counts *processes*, so with `memory.oom.group=1` it jumps by the number of tasks. |
| `oom_group_kill` | Number of times a **group** OOM happened (once per event, regardless of how many tasks died). |
| `sock_throttled` | Network sockets belonging to this cgroup were throttled because socket memory pushed against the limit. |

**`memory.stat` — the fields that carry the story.** There are around sixty; these are the ones you actually read. All amounts are bytes.

| Key | What it is |
|---|---|
| `anon` | Anonymous mappings — `brk`, `mmap(MAP_ANONYMOUS)`, the heap, your tensors. |
| `file` | Filesystem cache, including tmpfs and shared memory. |
| `kernel` | Total kernel memory charged here, including the four below plus other kernel uses. |
| `kernel_stack` | Kernel stacks for this cgroup's threads (16 KiB per thread on x86-64 — a 4,000-thread dataloader is 64 MiB you didn't plan). |
| `pagetables` | The page tables computed in §2. |
| `sec_pagetables` | Secondary page tables: KVM MMU allocations and IOMMU page tables. Non-zero on nodes doing device passthrough. |
| `percpu` | Per-CPU kernel data structures. |
| `sock` | Network transmission buffers. |
| `shmem` | Swap-backed cached data: tmpfs, `shm` segments, shared anonymous mmaps. |
| `file_mapped` | Cached file data mapped with `mmap`. |
| `file_dirty` | Modified page cache not yet written back. |
| `file_writeback` | Modified page cache currently being written back. |
| `anon_thp` | Anonymous memory backed by transparent huge pages (lesson 06). |
| `inactive_anon`, `active_anon`, `inactive_file`, `active_file`, `unevictable` | The five reclaim LRU lists. `inactive_file` is the one Kubernetes subtracts for working set. `unevictable` is `mlock`ed and long-term-pinned memory. |
| `slab_reclaimable` / `slab_unreclaimable` | Dentries and inodes (droppable) vs kernel structures that are not. |
| `workingset_refault_anon` / `workingset_refault_file` | **Refaults**: pages that were evicted and then immediately needed again. This is the thrash counter. Rising refaults + rising PSI `full` = the pre-OOM state. |
| `workingset_activate_file` | Refaulted pages that were promoted straight to the active list — the kernel admitting it evicted something hot. |
| `pgscan_kswapd` / `pgscan_direct` | Pages scanned by the background reclaim thread vs scanned synchronously by a process that had to reclaim on its own allocation path. **`pgscan_direct` climbing means your workload is paying reclaim latency itself.** |
| `pgsteal_kswapd` / `pgsteal_direct` | Pages actually reclaimed by each. The ratio `pgsteal/pgscan` is reclaim efficiency — as it falls toward zero, the kernel is scanning hard and freeing nothing, which is the last stop before OOM. |
| `pgfault` / `pgmajfault` | All faults vs faults that required I/O. |
| `pswpin` / `pswpout` | Pages swapped in / out. Zero on a standard Kubernetes node, by design. |

An annotated read of a real container, line by line:

```
$ cd /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/\
kubepods-burstable-pod3f2a....slice/cri-containerd-9c1e....scope

$ cat memory.max
268435456                       # 256 MiB — this is the pod's limits.memory
$ cat memory.current
268435456                       # pinned exactly at the limit
$ cat memory.peak
268435456                       # and it has been there before

$ grep -E '^(anon|file|kernel|pagetables|inactive_file|active_file|slab|sock) ' memory.stat
anon 251658240                  # 240 MiB — the model. Unreclaimable with swap off.
file 6291456                    #   6 MiB of page cache total
kernel 8404992                  #   8 MiB of kernel memory charged here
pagetables 1310720              # 1.25 MiB of page tables (§2 arithmetic, small process)
inactive_file 2097152           #   2 MiB the kernel thinks is cold
active_file 4194304             #   4 MiB it thinks is hot
slab 5242880                    #   5 MiB dentries/inodes/etc
sock 262144                     # 256 KiB of socket buffers

$ grep -E 'workingset_refault_file|pgscan_direct|pgsteal_direct' memory.stat
workingset_refault_file 918273  # it evicts cache and immediately needs it back
pgscan_direct 41882901          # scanning synchronously, on the allocation path
pgsteal_direct 1204418          # ~2.9% efficiency — scanning 34 pages per page freed

$ cat memory.events
low 0
high 0                          # memory.high is "max": no soft throttle configured
max 88                          # 88 times it was about to breach memory.max
oom 3                           # 3 of those ended with an allocation failure
oom_kill 3                      # and 3 processes died
oom_group_kill 1                # one of those was a group kill (3 tasks, 1 event)
```

**Reading it:** working set is `memory.current − inactive_file` = 268435456 − 2097152 = **266,338,304 bytes ≈ 254 MiB against a 256 MiB limit — 99.2% full.** The footprint is 240 MiB of *anonymous* memory, which with swap off cannot be reclaimed at all; the kernel's only lever is the ~6 MiB of page cache, and the refault counter proves it is evicting cache it immediately needs back. Reclaim efficiency of 2.9% is the kernel scanning 34 pages for every one it frees. `high 0` tells you nobody configured a soft throttle, so there was no gentle landing available. **This pod's limit is below its working set. It is a landmine, and it has already gone off three times.**

### 4. The reclaim path, as a state machine

Reclaim is not one thing. It is a background thread, a synchronous fallback, and a per-cgroup variant, all working over the same LRU lists.

**The lists.** Every reclaimable page sits on exactly one of four LRU lists per memory node: `active_anon`, `inactive_anon`, `active_file`, `inactive_file`, plus an `unevictable` list for pages that can never be reclaimed (`mlock`ed, long-term DMA-pinned). New file pages land on the *inactive* file list. A page referenced again while inactive gets promoted to active. Reclaim scans from the tail of the inactive lists — the coldest end — and demotes from active to inactive to refill them. This is a "second chance" clock, not a true LRU, because maintaining true LRU order would require touching a list on every memory access.

**Refault detection is what makes it self-correcting.** When the kernel evicts a file page it records a compact "shadow entry" in the page cache tree noting how much reclaim activity had happened at eviction time. If that page is read again soon, the kernel compares the eviction-time distance against current activity and can tell "this page was still hot when I dropped it." That increments `workingset_refault_file` and the page is activated immediately. A rising refault count is the kernel telling you, in its own words, that it is thrashing.

**Watermarks and who reclaims.** Each memory zone has three watermarks — `min`, `low`, `high` — derived from `vm.min_free_kbytes`, with the gaps between them scaled by `vm.watermark_scale_factor` (default 10, meaning the distances are 0.1% of the node's memory; max 3000 = 30%).

- Free pages fall below **low** → `kswapd` wakes and reclaims in the background until free pages are back above **high**. Your workload does not stall.
- Free pages fall below **min** while an allocation is in flight → the allocating task performs **direct reclaim** itself, inside the allocation call. This is a synchronous stall on your critical path, and it is what `pgscan_direct` counts and what PSI `some` reports.
- Reclaim cannot make progress → `__alloc_pages_slowpath` retries, then invokes the OOM killer.

**What gets reclaimed, and what cannot.**

| Page type | Reclaim action | Cost |
|---|---|---|
| Clean file page | Drop it | Free — no I/O |
| Dirty file page | Write back, then drop | One write I/O; may block on the device |
| Reclaimable slab (dentries, inodes) | Shrinker callbacks free entries | Cheap; `vm.vfs_cache_pressure` (default 100) tunes eagerness |
| Anonymous page **with swap** | Write to swap, then drop | One write; `vm.swappiness` (default **60**, range 0–200) sets the relative cost of swap vs file I/O |
| Anonymous page **without swap** | **Nothing.** Not reclaimable. | — |
| `unevictable` (mlocked, long-term pinned) | **Nothing.** Never scanned. | — |

**The memcg charge path — where a cgroup limit actually bites.** When a page is charged to a cgroup, `try_charge()` runs this loop (`mm/memcontrol.c`):

1. Try to charge against `memory.max`. If it fits, done.
2. If it doesn't fit, run **memcg-scoped reclaim** on that cgroup's own LRU lists, then retry. This repeats up to `MAX_RECLAIM_RETRIES`, which is **16** (`mm/internal.h`).
3. If sixteen rounds of reclaim still cannot free enough, increment `memory.events`'s `max`, then `oom`, and call the **cgroup OOM killer** with `totalpages = memory.max` for that cgroup.
4. Separately, whenever usage sits above `memory.high`, the returning task is **penalised with a sleep** before it resumes userspace.

That last mechanism is worth seeing in numbers, because it is why `memory.high` is a "throttle" and not a "warning." The penalty grows superlinearly with the overage ratio, capped at 2 seconds per charge batch. From the kernel's own table for a cgroup with `memory.high = 100M` (`mm/memcontrol.c`):

| Usage | Sleep imposed per allocation batch |
|---|---|
| 100 M | 0 ms |
| 101 M | 6 ms |
| 102 M | 25 ms |
| 105 M | 159 ms |
| 110 M | 639 ms |
| 115 M | 1439 ms |
| 118 M and above | 2000 ms (capped) |

A process 18% over `memory.high` is being put to sleep for two seconds at a time. That is deliberately brutal: `memory.high` exists so an external supervisor has time to react while the workload is slowed to a crawl but **not killed**. Kubernetes only sets it when the `MemoryQoS` feature gate is enabled, using `memory.high = floor((requests + 0.9 × (limits − requests)) / pagesize) × pagesize` — the 0.9 is the kubelet's `memoryThrottlingFactor`. Without that gate, `memory.high` on your pods reads `max` and there is no soft landing at all.

Here is the whole picture in one diagram — where each page type lives, which way reclaim moves it, and exactly where the two cgroup thresholds sit on that path:

```
                      A CGROUP'S MEMORY, AND WHAT RECLAIM CAN DO WITH IT
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │  memory.current  =  anon + file + shmem + kernel(slab,pagetables,sock,...)    │
  └──────────────────────────────────────────────────────────────────────────────┘

     ANONYMOUS (memory.stat: anon)          FILE-BACKED (memory.stat: file)
     tensors, activations, heap             page cache, mmap'd datasets
            │                                         │
            ▼                                         ▼
     ┌──────────────┐  referenced       ┌──────────────┐  referenced
     │ inactive_anon│◀────────────────▶ │ inactive_file│◀──────────────┐
     └──────┬───────┘                   └──────┬───────┘               │
            │ scan tail                        │ scan tail             │
            ▼                                  ▼                  ┌────┴──────┐
      swap.max == 0 ?                     clean ? ──yes──▶ DROP   │ active_file│
      ┌────┴────┐                           │              (free) └────▲──────┘
     yes        no                          no                         │
      │          │                          │                     promote on
      ▼          ▼                          ▼                     2nd reference
  ╔════════╗  write to                 writeback ──▶ DROP              │
  ║ CANNOT ║   swap ──▶ DROP           (one write I/O)                 │
  ║ RECLAIM║                                                           │
  ╚════════╝                        ┌────────────────┐                 │
                                    │ evicted page   │ needed again ───┘
                                    │ leaves a shadow│ → workingset_refault_file++
                                    └────────────────┘   (this is thrash)

  UNEVICTABLE (memory.stat: unevictable) — mlock'd, FOLL_LONGTERM DMA pins.
  Never scanned. On a GPU node this is your cudaHostAlloc'd staging buffers.

  ═════ WHERE THE THRESHOLDS SIT ON THAT PATH ═════════════════════════════════

  usage ──▶ memory.high ──────────────▶ memory.max ─────────────▶ OOM
             │                            │                        │
     throttle: task sleeps        16 rounds of memcg          oom_badness()
     0→2000 ms, superlinear       reclaim, then give up       over tasks in
     in the overage ratio;        events: max++, oom++        THIS cgroup only
     events: high++;              (never killed by .high)     events: oom_kill++
     NEVER kills
```

**Consequence for GPU jobs, stated precisely.** A training process's footprint is almost entirely `anon` — model parameters, optimizer state, activations — plus an `unevictable` tail of pinned host staging buffers. With `memory.swap.max = 0` (the Kubernetes default), *both* of those categories are in the CANNOT RECLAIM box. The kernel's entire reclaim toolkit applies to the few MiB of page cache the process happens to hold. So the sixteen retries in `try_charge` are sixteen rounds of scanning almost nothing, and the transition from "running fine" to "SIGKILL" takes milliseconds, not minutes. There is no gradual degradation to alert on. **This is why right-sizing the limit and watching `memory.peak` matters more on a training pod than on any stateless service** — a web service with 3 GB of page cache has a genuine cushion; your trainer does not.

### 5. `oom_badness()` — how the victim is actually chosen

When reclaim gives up, `out_of_memory()` runs. It establishes a **scope**, walks candidate tasks, scores each, and kills the highest scorer. Here is the scoring function, unabridged, from `mm/oom_kill.c`:

```c
long oom_badness(struct task_struct *p, unsigned long totalpages)
{
    long points;
    long adj;

    if (oom_unkillable_task(p))
        return LONG_MIN;                     /* kthreads, init, already-reaped */

    adj = (long)p->signal->oom_score_adj;
    if (adj == OOM_SCORE_ADJ_MIN ||          /* == -1000: unconditionally immune */
        mm_flags_test(MMF_OOM_SKIP, p->mm) ||
        in_vfork(p))
        return LONG_MIN;

    /* The baseline for the badness score is the proportion of RAM that each
     * task's rss, pagetable and swap space use. */
    points = get_mm_rss_sum(p->mm)
           + get_mm_counter_sum(p->mm, MM_SWAPENTS)
           + mm_pgtables_bytes(p->mm) / PAGE_SIZE;

    adj *= totalpages / 1000;                /* normalize to oom_score_adj units */
    points += adj;

    return points;
}
```

Five things fall out of those fifteen lines, and they are the whole of victim selection:

1. **The score is a raw page count, not a 0–1000 percentage.** Most write-ups say "the score is a permille of available memory." That was the pre-2.6.36 heuristic. What the kernel computes today is *literally the number of pages the task is responsible for*: resident pages + swap entries + page-table pages. The 0–1000 value you can read at `/proc/<pid>/oom_score` is a normalized *view* of this, not the internal number. **Bigger footprint wins, and "kill the hog" is the entire policy.**
2. **Page tables count against you.** The 80 MiB you computed in §2 is 20,480 pages added to your score. Processes with huge sparse mappings are penalised for the tables, which is deliberate — killing them frees those tables too.
3. **`oom_score_adj` is scaled into the same units before being added.** `adj *= totalpages / 1000` converts the −1000…+1000 knob into pages: each point of `oom_score_adj` is worth one-thousandth of the scope's total memory. So `+100` on a 256 MiB cgroup adds 100 × 65536/1000 ≈ 6,553 pages ≈ 25 MiB to your effective footprint. `+1000` adds the *entire* scope, guaranteeing selection. **This is why `oom_score_adj` is meaningful only relative to the scope's size** — the same adjustment is worth 25 MiB in a small container and 51 GB on a 512 GiB node.
4. **`-1000` is not "very low," it is an early return of `LONG_MIN`.** The task is skipped before its footprint is even computed. It is genuinely immune — including immune to `memory.oom.group`, which explicitly exempts `oom_score_adj == -1000` tasks from the group kill.
5. **`totalpages` differs by scope.** For a memcg OOM, `constrained_alloc()` sets `oc->totalpages = mem_cgroup_get_max(oc->memcg)` — the cgroup's `memory.max` (plus its swap allowance, if any). For a global OOM it is the machine's total present RAM. Same function, wildly different denominators.

**The scan itself** is in `select_bad_process()`, and it is one `if` statement that decides everything about blast radius:

```c
if (is_memcg_oom(oc))
    mem_cgroup_scan_tasks(oc->memcg, oom_evaluate_task, oc);   /* THIS SUBTREE ONLY */
else
    for_each_process(p) oom_evaluate_task(p, oc);              /* EVERY TASK ON THE NODE */
```

Here is that walk as a picture, with the Kubernetes QoS bias applied so you can see why a global OOM on a GPU node kills what it kills:

```
            THE OOM SCORING WALK — SAME FUNCTION, TWO SCOPES

  ┌────────────────────────── / (root cgroup) ────────────────────────────┐
  │                                                                        │
  │  system.slice/                       kubepods.slice/                   │
  │  ├── kubelet.service                 ├── kubepods-besteffort.slice/    │
  │  │     oom_score_adj = -999          │   └── log-shipper               │
  │  │     rss 180 MiB                   │         adj = +1000             │
  │  ├── containerd.service              │         rss 300 MiB             │
  │  │     oom_score_adj = -999          ├── kubepods-burstable.slice/     │
  │  └── nvidia-dcgm-exporter            │   └── dataloader-cache          │
  │        adj = 0, rss 400 MiB          │         adj = +938              │
  │                                      │         rss 6 GiB               │
  │                                      └── pod-trainer.slice/  ◀── memory.max = 200 GiB
  │                                          └── cri-containerd-...scope   │
  │                                              ├── rank0  adj=-997 rss 96 GiB
  │                                              ├── rank1  adj=-997 rss 96 GiB
  │                                              └── loader adj=-997 rss  3 GiB
  └────────────────────────────────────────────────────────────────────────┘

  CASE A — cgroup OOM (trainer container hits its own memory.max = 200 GiB)
    scope     : mem_cgroup_scan_tasks(pod-trainer container cgroup)
    candidates: rank0, rank1, loader          ← nothing else on the node is looked at
    totalpages: 200 GiB / 4 KiB = 52,428,800
    adj bias  : -997 × 52428800/1000 = -52,271,513 pages  ≈  -199.4 GiB
    → every candidate's score goes deeply negative, but they are all biased
      identically, so RELATIVE ORDER IS UNCHANGED: rank0 (or rank1) still wins.
    → oom_score_adj DID NOT PROTECT ANYTHING HERE. It only reorders peers,
      and inside one container every peer carries the same value.
    → memory.oom.group = 1 (kubelet default) ⇒ all three die together.
    dmesg says: "Memory cgroup out of memory: Killed process ..."

  CASE B — global OOM (the node itself is out of memory)
    scope     : for_each_process()             ← EVERY task on the box
    totalpages: 512 GiB / 4 KiB = 134,217,728
    adj bias  : one point = 134,217 pages = 512 MiB of effective footprint
      log-shipper   : 300 MiB rss  +1000 × 512 MiB = +512 GiB → HIGHEST, dies first
      dataloader    :   6 GiB rss   +938 × 512 MiB = +480 GiB → dies second
      dcgm-exporter : 400 MiB rss      0           =   0      → mid
      rank0         :  96 GiB rss   -997 × 512 MiB = -510 GiB → net ≈ -414 GiB
      kubelet       : 180 MiB rss   -999 × 512 MiB = -511 GiB → effectively last
    → the QoS ladder is doing exactly what it was designed to do: BestEffort
      dies before Burstable dies before Guaranteed dies before node daemons.
    dmesg says: "Out of memory: Killed process ..." + a full node memory dump
```

**The one thing that diagram is there to teach:** `oom_score_adj` is a *tiebreaker among candidates in the same scope*. In a global OOM it is decisive, because the candidate set spans every QoS class. In a cgroup OOM it is usually irrelevant, because every candidate is inside the same container and therefore carries the same value. Engineers who "protect" a process with `oom_score_adj=-998` and then watch it die to its own `memory.max` are hitting exactly this.

### 6. Kubernetes: QoS class → `oom_score_adj`, exactly

The kubelet computes this in `pkg/kubelet/qos/policy.go`. The real constants:

| Process | `oom_score_adj` | Source |
|---|---|---|
| kubelet | **−999** | `KubeletOOMScoreAdj` |
| kube-proxy | **−999** | `KubeProxyOOMScoreAdj` |
| Node-critical pods (`system-node-critical`) | **−997** | `IsNodeCriticalPod` → `guaranteedOOMScoreAdj` |
| **Guaranteed** pod containers | **−997** | `guaranteedOOMScoreAdj` |
| **Burstable** pod containers | `1000 − (1000 × memoryRequest) / memoryCapacity`, clamped to **[3, 999]** | computed |
| **BestEffort** pod containers | **+1000** | `besteffortOOMScoreAdj` |

The two clamps matter. The lower clamp is `1000 + guaranteedOOMScoreAdj = 3`, so a Burstable container can never be scored better than a Guaranteed one no matter how large its request. The upper clamp turns a computed `1000` into `999`, so a Burstable container with a negligible request still dies *after* BestEffort.

**Worked example.** A 512 GiB GPU node. A Burstable dataloader sidecar requests `memory: 32Gi`:

```
oom_score_adj = 1000 − (1000 × 32Gi) / 512Gi
              = 1000 − (1000 × 34359738368) / 549755813888
              = 1000 − 62
              = 938
```

Effective footprint added at global-OOM time: `938 × (134217728 / 1000)` pages = 125,896,228 pages ≈ **480 GiB**. Its actual RSS of 6 GiB is a rounding error next to the bias. That is the design: in a global OOM, QoS class decides, not size. Bump the request to `256Gi` and you get `adj = 500`, halving the bias — **which is the real reason "set requests close to actual usage" is advice about survival, not just scheduling.**

### 7. `memory.oom.group` — and why it is already on

`memory.oom.group=1` tells the kernel to treat the cgroup as an indivisible workload: when OOM fires anywhere in it, every task in it and its descendants is killed together, except tasks with `oom_score_adj == -1000`. If the OOM killer fires *in* a cgroup, it will not kill anything outside that cgroup regardless of ancestors' `oom.group` values.

Why you want it for a distributed job: a training container runs rank-0 plus N workers under `torchrun`. Without group kill, the killer picks the single largest process. The survivors then block forever inside a collective — an all-reduce waiting on a rank that will never post its buffer — holding their GPUs allocated and idle until some outer timeout fires. With group kill, the container dies as a unit, the kubelet reports one unambiguous failure, and the job restarts from its last checkpoint.

**The current-state correction that matters:** on cgroup v2, **the kubelet already sets this for you.** In `pkg/kubelet/kuberuntime/kuberuntime_container_linux.go`:

```go
// runc requires cgroupv2 for unified mode
if isCgroup2UnifiedMode() && !ptr.Deref(m.singleProcessOOMKill, true) {
    resources.Unified = map[string]string{
        // Ask the kernel to kill all processes in the container cgroup in case of OOM.
        "memory.oom.group": "1",
    }
}
```

and the API documentation for the flag says: *"On cgroup v2 linux, null / absent, true and false are allowed. The default value is false."* So on a cgroup-v2 node, `singleProcessOOMKill` defaults to `false`, the condition is true, and **`memory.oom.group=1` is applied to every container cgroup by default**. If you need the old cgroup-v1 behaviour — one process at a time — you set `singleProcessOOMKill: true` in the kubelet config.

Two consequences people trip over:

- It is set on the **container** cgroup, not the pod cgroup. A multi-container pod does not lose its sidecar when the main container OOMs; the group boundary is the container.
- Debugging shells and sidecar agents inside the OOMing container die with it. If you have ever `kubectl exec`'d into a memory-hungry container and had your shell vanish "for no reason," this is why.

### 8. Reading the OOM report from `dmesg`, field by field

The report is emitted by `dump_header()`, `dump_tasks()` and `__oom_kill_process()`. Because the format strings are in the source, you can predict every field. Here is a cgroup OOM as it actually appears (timestamps trimmed, stack trace elided):

```
[12345.678901] python invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=-997
[12345.678910] CPU: 42 PID: 4711 Comm: python Not tainted 6.8.0-45-generic
[12345.678915] Call Trace:
[12345.678920]  dump_stack_lvl+0x48/0x70
[12345.678925]  dump_header+0x4f/0x240
[12345.678930]  oom_kill_process+0x10d/0x1c0
                ... (elided) ...
[12345.679001] memory: usage 268435456kB, limit 268435456kB, failcnt 88
[12345.679005] swap: usage 0kB, limit 0kB, failcnt 0
[12345.679010] Memory cgroup stats for /kubepods.slice/kubepods-burstable.slice/
                 kubepods-burstable-pod3f2a....slice/cri-containerd-9c1e....scope:
[12345.679012] anon 251658240 file 6291456 kernel 8404992 pagetables 1310720
                 inactive_file 2097152 active_file 4194304 ...
[12345.679100] Tasks state (memory values in pages):
[12345.679101] [  pid  ]   uid  tgid total_vm      rss rss_anon rss_file rss_shmem pgtables_bytes swapents oom_score_adj name
[12345.679110] [   4711 ]  1000  4711   918273    62234    61234     1000         0        1310720        0          -997 python
[12345.679115] [   4715 ]  1000  4715    41200     1802     1600      202         0         180224        0          -997 python
[12345.679130] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=cri-containerd-9c1e...,
                 mems_allowed=0-1,oom_memcg=/kubepods.slice/.../cri-containerd-9c1e....scope,
                 task_memcg=/kubepods.slice/.../cri-containerd-9c1e....scope,task=python,pid=4711,uid=1000
[12345.679140] Memory cgroup out of memory: Killed process 4711 (python) total-vm:3673092kB,
                 anon-rss:244936kB, file-rss:4000kB, shmem-rss:0kB, UID:1000 pgtables:1280kB oom_score_adj:-997
[12345.679900] oom_reaper: reaped process 4711 (python), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
```

Now the field-by-field read. **Every one of these is derived from a `printk` format string you can check.**

| Line | What it tells you |
|---|---|
| `python invoked oom-killer: gfp_mask=..., order=0, oom_score_adj=-997` | **Who triggered it** — the task whose allocation failed, which is usually but not always the victim. `order=0` means a single-page allocation (order-N = 2^N contiguous pages); a non-zero order points at fragmentation, not exhaustion. The `oom_score_adj` here is the *trigger's*, not the victim's. |
| `memory: usage 268435456kB, limit 268435456kB, failcnt 88` | **The cgroup-OOM signature.** `mem_cgroup_print_oom_meminfo()` only runs `if (is_memcg_oom(oc))`. Usage == limit. `failcnt` is the `memory.events` `max` counter. A global OOM prints a node-wide zone dump here instead (`Node 0 Normal free:... min:... low:...`) and no `memory:`/`swap:` pair. |
| `swap: usage 0kB, limit 0kB` | Confirms swap was not available, so anonymous memory was unreclaimable. On a Kubernetes node this is the normal state and the reason the kill was abrupt. |
| `Memory cgroup stats for /kubepods.slice/...` | The exact cgroup path — this is how you map the kill to a pod without any Kubernetes API access. The keys that follow are the same `memory.stat` keys from §3. |
| `Tasks state (memory values in pages)` + header | The candidate list, **printed only if `vm.oom_dump_tasks` is 1 (the default)**. In a memcg OOM only tasks in that cgroup are listed — the list *is* the scope. Columns: `total_vm` (VSZ), `rss` (total), then the `rss_anon`/`rss_file`/`rss_shmem` split, `pgtables_bytes` (**bytes**, unlike the neighbouring page counts), `swapents`, and each task's `oom_score_adj`. |
| `[ 4711 ] ... 918273 62234 61234 1000 0 1310720 0 -997 python` | Multiply page columns by 4 KiB: `total_vm` 918,273 pages = 3.50 GiB virtual; `rss` 62,234 pages = 243 MiB resident, of which `rss_anon` 61,234 pages = 239 MiB is anonymous. `pgtables_bytes` 1,310,720 = 1.25 MiB. Compare `rss` across rows and you can predict the winner before reading the kill line. |
| `oom-kill:constraint=CONSTRAINT_MEMCG,...` | The single most useful line. `constraint` is one of `CONSTRAINT_NONE` (true global exhaustion), `CONSTRAINT_CPUSET`, `CONSTRAINT_MEMORY_POLICY` (a mempolicy/NUMA-bound allocation failed while other nodes had memory — lesson 06 territory), or `CONSTRAINT_MEMCG`. `oom_memcg=` is the cgroup **that hit its limit**; `task_memcg=` is the cgroup **the victim lives in**. When they differ, a parent cgroup's limit killed a child's task. For a global OOM this line reads `,global_oom` instead of `,oom_memcg=...`. |
| `Memory cgroup out of memory: Killed process 4711 (python) ...` | The verdict. The prefix is the `message` argument to `oom_kill_process()`: **`"Memory cgroup out of memory"` for memcg scope, `"Out of memory"` for global.** That single word "cgroup" is the fastest scope test you have. `anon-rss` is the number that tells you whether the victim was reclaimable at all. |
| `oom_reaper: reaped process 4711 ...` | A kernel thread that tears down the victim's address space asynchronously so memory is returned even if the victim is stuck exiting. Seeing `now anon-rss:0kB` confirms the memory actually came back. If you instead see `oom_reaper: unable to reap pid:...`, the victim's memory is pinned (often by a device driver holding long-term DMA pins) and the OOM may repeat. |

**The five-facts drill.** Under incident pressure, extract exactly these, in this order, in under a minute:

1. **Scope** — `Memory cgroup out of memory` vs `Out of memory`; confirm with `constraint=`.
2. **Victim** — PID and comm from the `Killed process` line.
3. **Its footprint** — `anon-rss` vs `file-rss`. Mostly anon ⇒ nothing was reclaimable.
4. **Its bias** — `oom_score_adj` on the kill line; cross-check against the QoS table in §6 to infer the pod's QoS class without touching `kubectl`.
5. **Why it won** — scan the `rss` column of the task table; the victim should be the largest, and if it isn't, the difference is explained by `oom_score_adj` in the last column.

### 9. cgroup OOM vs global OOM vs kubelet eviction — three different events

These get conflated constantly, and they have different blast radii, different signals, and different fixes.

| | **cgroup OOM** | **global OOM** | **kubelet eviction** |
|---|---|---|---|
| Trigger | A cgroup's charge hits `memory.max` and 16 rounds of memcg reclaim fail | Node-wide allocation fails after global reclaim fails | `memory.available` on the node crosses the hard threshold (`100Mi` by default) |
| Who decides | Kernel, `mm/oom_kill.c` | Kernel, `mm/oom_kill.c` | Kubelet, userspace |
| Candidates | Tasks in that cgroup subtree only | Every task on the node | Pods, ranked by whether they exceed requests, then QoS/priority, then usage |
| Metric used | `memory.current` vs `memory.max` | Free pages vs zone watermarks | **working set** = `memory.current − inactive_file` |
| Signal | `memory.events` `oom_kill`++, `Memory cgroup out of memory` in dmesg | `Out of memory` + node zone dump | Pod status `Evicted`, `MemoryPressure` node condition |
| Container status | `OOMKilled`, exit code **137** (128 + SIGKILL) | `OOMKilled`, exit code 137 | `Evicted`, graceful termination attempted |
| Typical fix | Raise the limit, or fix the leak | Fix the node's allocatable / a pod that escaped its limit | Same, plus reserve more for the system |

**The eviction path is the one that is *supposed* to happen.** The kubelet polls the node's working set, and when `memory.available` drops below `evictionHard["memory.available"]` (default `100Mi`; the full default set is `memory.available<100Mi`, `nodefs.available<10%`, `nodefs.inodesFree<5%`, `imagefs.available<15%`, `imagefs.inodesFree<5%`), it starts terminating pods gracefully — before the kernel is forced to SIGKILL anything. Global OOM means the eviction path lost the race, which on a node with fast-growing anonymous allocations (a trainer that adds 20 GB in two seconds) it frequently does, because the kubelet's housekeeping interval is on the order of seconds.

**Why eviction uses working set and not RSS.** RSS includes cold file cache the kernel will hand back for free, and counts shared pages once per sharer. Ranking pods by RSS would evict a pod that is merely *holding a lot of page cache* — the very thing the kernel is about to reclaim by itself — while sparing a pod whose anonymous footprint is the actual problem. Working set (`usage − inactive_file`) removes exactly the pages the kernel has already judged cold, which makes it the closest cheap proxy for "memory that will still be needed a moment from now."

### 10. What all of this means on a GPU node

Three GPU-specific realities change the arithmetic:

**Pinned host memory is unevictable by construction.** Any CUDA host-to-device copy that is asynchronous, or that hits full PCIe bandwidth, must come from **page-locked** memory — `cudaHostAlloc`, `cudaHostRegister`, or in PyTorch `DataLoader(..., pin_memory=True)`. Copies from ordinary pageable memory are staged through a driver-owned bounce buffer, which is why they are slower and effectively synchronous. Page-locking takes a **long-term DMA pin** (`FOLL_PIN | FOLL_LONGTERM` in kernel terms, `Documentation/core-api/pin_user_pages.rst`), which means those pages are never scanned by reclaim, never migrated by compaction, and never moved by NUMA balancing. They appear in `memory.stat`'s `unevictable`. **A pod with 24 GiB of pinned staging buffers has 24 GiB that reclaim structurally cannot touch, on top of its anonymous model memory.** If you size the limit from a graph of "usage" without knowing how much of it is pinned, you will size it wrong in the direction that kills you.

**Swap is off, and should stay off.** Kubernetes sets `memory.swap.max = 0` unless `NodeSwap` is enabled. This is correct for GPU work: a swapped-out page in a dataloader's path stalls the H2D copy, which stalls the training step, which idles eight accelerators. Killing the job and restarting from a checkpoint is genuinely cheaper than swapping it. But it removes the only reclaim mechanism that applies to anonymous memory, which is why the kill is instantaneous.

**Every kill costs GPU-hours, not seconds.** A kill mid-epoch discards everything since the last checkpoint, across every rank. The restart also pays container start, CUDA context creation, and NCCL bootstrap — order of minutes on a large job before a single useful FLOP.

## Perspectives

**Kernel-mechanism view.** From the kernel's side, OOM is a failure handler, not a feature. `try_charge()` will spend sixteen full rounds of memcg reclaim before it even considers killing; the global path retries in `__alloc_pages_slowpath` and wakes `kswapd` and `kcompactd` first. `oom_badness()` is deliberately, almost crudely simple — a page count plus a scaled bias — because it has to execute correctly under `oom_lock` on a machine that cannot allocate memory. There is no room in that context for a smarter policy, and that constraint is the honest explanation for why the kernel sometimes kills the "wrong" thing.

**Operator/SRE view.** The highest-leverage skill here is not theory, it is the five-facts drill in §8 executed cold, from a `dmesg` buffer, in under a minute. Scope first (the word "cgroup"), then victim, then `anon-rss`, then `oom_score_adj`, then the task table to confirm why that victim won. Everything downstream — is this one pod or the node, do I cordon, do I raise a limit or hunt a leak — follows from those five facts, and getting the scope wrong sends the first twenty minutes of an incident in the wrong direction.

**GPU-fleet-specific view.** Training and inference processes are the pathological case for reclaim: anonymous tensors plus long-term-pinned staging buffers, with swap off, means the reclaimable fraction of a trainer's footprint is often under 2%. The practical consequences are that (a) `memory.peak` is a far better right-sizing input than any average, (b) PSI `full` on the pod's `memory.pressure` is your only real early warning and it may only lead by seconds, and (c) `memory.high` with the `MemoryQoS` gate is the one mechanism that converts a hard kill into a survivable slowdown — at the price of a workload that can be slept for 2 seconds per allocation batch, which for a latency-sensitive inference pod may be worse than dying.

**Economics/failure-mode view.** Price a kill instead of describing it. Take an 8-GPU job, checkpointing every 30 minutes, killed 25 minutes into an interval. Lost work is 25 min × 8 GPUs = 3.33 GPU-hours; add ~4 minutes of restart (container pull-through, CUDA init, NCCL bootstrap, checkpoint reload) × 8 = 0.53 GPU-hours. At an on-demand rate of order **$2–3/GPU-hour** (a dated 2026 snapshot; recompute against your own contract), that is **$7.7–11.6 per kill**, and it scales linearly in GPU count and rate. Trivial once. A fleet of 300 such jobs with limits set from averages rather than `memory.peak`, killing at even 2% per day, is 6 kills/day ≈ **$17k–25k/year of pure waste** — and that is before the on-call time. The two levers are checkpoint interval (halving it halves the lost-work term but adds its own I/O stall, itself PSI-visible from lesson 04) and limit sizing from observed peaks. Sizing checkpoint interval *against the observed OOM-kill rate* rather than a round number like "hourly" is the staff-level version of this trade-off.

## Real-world use cases

**LINE Engineering — "Who murdered my lovely Prometheus container in Kubernetes cluster?"** A Prometheus container is repeatedly `OOMKilled` while node-level dashboards show plenty of free memory. The investigation walks exactly the path this lesson formalises: confirm from the kernel log that the kill is cgroup-scoped rather than global, map the cgroup path in the report back to the pod, read the victim's resident breakdown to see that the footprint is dominated by memory the kernel cannot reclaim, and check `oom_score_adj` to understand why that process and not a neighbour. The takeaway that generalises: the node-level "free memory" graph and the container's own limit are answering different questions, and only the second one killed anything. It is the same investigation as this lesson's worked example, told as a real incident.

**Meta Engineering — "Open-sourcing oomd."** Meta's argument is that the in-kernel OOM killer is structurally too late and too blunt for a fleet. Too late, because by design it only runs *after* an allocation has already failed — by which point the machine has typically spent minutes thrashing, with every workload on it degraded. Too blunt, because `oom_badness()` is a page count plus a bias, with no notion of which workload is more important to the business, and no way to express "kill the batch job, never the storage daemon." `oomd` is a userspace daemon that watches PSI pressure (the same signal from lesson 04) and cgroup-level memory state, and acts on operator-written policy *before* the kernel's hand is forced — killing a chosen cgroup while the machine is still responsive. The durable lesson for a GPU platform is the framing: the kernel's killer is a safety net, and if it is firing regularly in your fleet, the missing component is a policy layer above it, not a bigger `memory.max`.

**Kubernetes SIG-Node, group OOM kill (issues #117070, #124253; PR #126096).** When Kubernetes moved to cgroup v2, containers gained `memory.oom.group=1` — an OOM anywhere in a container kills every process in it. This surfaced as a behaviour change in the field: workloads that ran a supervisor plus workers, or that deliberately let a child be sacrificed, suddenly lost the whole container; CI runners and exec'd debugging shells died alongside the offending process. SIG-Node's resolution was to keep group kill as the default (it is the correct semantic for a container as a unit of failure) but add a `singleProcessOOMKill` kubelet option to restore cgroup-v1 behaviour for workloads that genuinely depend on it. What this shows: the group-kill decision is not a tuning knob you reach for after an incident — it is already made on your nodes, and knowing which way it is set changes what you predict will happen.

## Worked example

**Goal: trigger a cgroup OOM, then explain the kill from `dmesg` alone — including the group-kill contrast and the `oom_score_adj` contrast.**

### Step 1 — build the cage

```
$ sudo mkdir /sys/fs/cgroup/oom-demo
$ echo "+memory" | sudo tee /sys/fs/cgroup/cgroup.subtree_control   # if not already enabled
$ echo 256M | sudo tee /sys/fs/cgroup/oom-demo/memory.max
$ echo 0    | sudo tee /sys/fs/cgroup/oom-demo/memory.swap.max      # no swap escape
$ cat /sys/fs/cgroup/oom-demo/memory.oom.group
0                                                                   # one victim at a time, for now
```

Baseline the counters so the deltas are unambiguous:

```
$ cat /sys/fs/cgroup/oom-demo/memory.events
low 0
high 0
max 0
oom 0
oom_kill 0
oom_group_kill 0
```

### Step 2 — run a hog that cannot be reclaimed

```
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/oom-demo/cgroup.procs; \
    exec stress-ng --vm 1 --vm-bytes 1G --vm-keep --timeout 30s'
```

`--vm-bytes 1G` inside a 256 MiB cap with swap off: the allocation is anonymous, there is nothing on the file LRU worth reclaiming, so `try_charge()` burns its sixteen retries against empty lists and calls the memcg OOM killer.

### Step 3 — confirm the scope from the cgroup's own counters

```
$ cat /sys/fs/cgroup/oom-demo/memory.events
low 0
high 0
max 173          # 173 times a charge was about to breach memory.max
oom 1            # one of those ended in an unavoidable allocation failure
oom_kill 1       # one process died
oom_group_kill 0 # not a group kill — oom.group is still 0

$ free -m | head -2
              total        used        free      shared  buff/cache   available
Mem:          64000       11400        2600         300       50000       51200
```

**That pairing is the whole point of the exercise.** The cgroup died with 51 GB available on the node. Anyone reading the node dashboard would conclude the node was healthy — and they would be right. The failure was entirely local to one `memory.max`.

### Step 4 — read the kernel's post-mortem

```
$ sudo dmesg | grep -A 20 'invoked oom-killer' | tail -25
```

```
stress-ng-vm invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=0
memory: usage 262144kB, limit 262144kB, failcnt 173
swap: usage 0kB, limit 0kB, failcnt 0
Memory cgroup stats for /oom-demo: anon 260046848 file 0 kernel 1929216
  pagetables 786432 inactive_file 0 active_file 0 unevictable 0 ...
Tasks state (memory values in pages):
[  pid  ]   uid  tgid total_vm      rss rss_anon rss_file rss_shmem pgtables_bytes swapents oom_score_adj name
[   5121 ]     0  5121     3212      412      108      304         0          61440        0             0 stress-ng
[   5123 ]     0  5123   266128    63447    63215      232         0         786432        0             0 stress-ng-vm
oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=/,mems_allowed=0,
  oom_memcg=/oom-demo,task_memcg=/oom-demo,task=stress-ng-vm,pid=5123,uid=0
Memory cgroup out of memory: Killed process 5123 (stress-ng-vm) total-vm:1064512kB,
  anon-rss:252860kB, file-rss:928kB, shmem-rss:0kB, UID:0 pgtables:768kB oom_score_adj:0
oom_reaper: reaped process 5123 (stress-ng-vm), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
```

**The five facts, extracted:**

1. **Scope: cgroup.** Two independent confirmations — `memory: usage 262144kB, limit 262144kB` (this block is printed only for memcg OOMs) and the literal phrase `Memory cgroup out of memory`. Third confirmation: `constraint=CONSTRAINT_MEMCG`. A global OOM would print a zone dump and say plain `Out of memory`.
2. **Victim: PID 5123, `stress-ng-vm`.** Named on both the `oom-kill:` context line and the kill line.
3. **Footprint: `anon-rss:252860kB`, `file-rss:928kB`.** 99.6% anonymous. Cross-check against the cgroup stats line: `anon 260046848` = 248 MiB against a 256 MiB cap, `file 0`, `inactive_file 0`. **There was literally nothing to reclaim.** Working set = 262144 KiB − 0 = the entire limit.
4. **Bias: `oom_score_adj:0`.** No thumb on the scale; raw footprint decided.
5. **Why it won:** the task table has two candidates. `stress-ng` (the parent) has `rss` 412 pages = 1.6 MiB. `stress-ng-vm` has `rss` 63,447 pages = 248 MiB. Both carry `oom_score_adj 0`, so `oom_badness()` reduces to `rss + swapents + pgtables/PAGE_SIZE`: 63,447 + 0 + 192 = **63,639** versus 412 + 0 + 15 = **427**. The scores differ by 149×. The outcome was never in doubt.

One more field worth the habit: `total-vm:1064512kB` = 1.01 GiB. That is what the process *asked* for and matches `--vm-bytes 1G`. Only 248 MiB of it ever became resident before the kill. **Virtual size told you the intent; resident size told you the cause.**

### Step 5 — contrast A: prove `oom_score_adj` reorders peers

Run two hogs in the cgroup, a big one protected and a smaller one not:

```
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/oom-demo/cgroup.procs; \
    stress-ng --vm 1 --vm-bytes 200M --vm-keep --timeout 60s & BIG=$!; \
    sleep 1; echo -998 > /proc/$BIG/oom_score_adj; \
    stress-ng --vm 1 --vm-bytes 120M --vm-keep --timeout 60s'
```

The 200 MB process now carries `adj = -998`. With `totalpages = memory.max = 65536 pages`, that bias is `-998 × 65536/1000 = -65,405` pages. Its raw score of ~51,200 pages becomes roughly **−14,200**; the unprotected 120 MB process scores ~30,700. The kernel kills the *smaller* process. The kill line will confirm it with `oom_score_adj:0` on the victim while the log's task table shows the 200 MB peer at `-998`.

**Then note what this does not prove.** If you had set `-998` on *every* process in the cgroup — which is exactly what Kubernetes does inside a Guaranteed pod's container — all candidates shift by the same amount and the ordering is unchanged. Someone still dies. `oom_score_adj` protects you from *neighbours*, never from your own limit.

### Step 6 — contrast B: group kill

```
$ echo 1 | sudo tee /sys/fs/cgroup/oom-demo/memory.oom.group
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/oom-demo/cgroup.procs; \
    stress-ng --vm 2 --vm-bytes 400M --vm-keep --timeout 30s'
$ cat /sys/fs/cgroup/oom-demo/memory.events | grep -E 'oom_kill|oom_group_kill'
oom_kill 3
oom_group_kill 1
```

**Read the two counters together:** one group-OOM *event*, three *processes* killed — the two workers and their parent, atomically. `dmesg` shows one `Memory cgroup out of memory: Killed process ...` line per task. This is the behaviour a multi-rank training container wants, and — per §7 — the behaviour your Kubernetes nodes already have by default on cgroup v2.

### Step 7 — the counterfactual that makes it a design argument

Rerun step 2 with a soft throttle in place:

```
$ echo 200M | sudo tee /sys/fs/cgroup/oom-demo/memory.high
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/oom-demo/cgroup.procs; \
    exec stress-ng --vm 1 --vm-bytes 240M --vm-keep --timeout 30s'
$ cat /sys/fs/cgroup/oom-demo/memory.events
low 0
high 4192        # throttled 4192 times
max 0
oom 0
oom_kill 0       # never killed
```

Same workload shape, no kill — it was slept into compliance instead. Cross-check the cost in PSI (lesson 04): `cat /sys/fs/cgroup/oom-demo/memory.pressure` will show `full` accumulating, which is the honest accounting of what the throttle cost you. That is the trade `memory.high` offers, and it is the argument for enabling the `MemoryQoS` feature gate on nodes running restartable batch work.

## Practice

**Environment:** a Linux VM or laptop with cgroup v2 (`stat -fc %T /sys/fs/cgroup` returns `cgroup2fs`), `stress-ng`, root, and `dmesg` access. A container works if you can create a delegated sub-cgroup and set `memory.max`. Do not do step 4 on a shared or production node.

1. **Map a real container's memory picture.** Pick any running container (or create the demo cgroup). Record `memory.current`, `memory.max`, `memory.peak`, and from `memory.stat` the values of `anon`, `file`, `inactive_file`, `active_file`, `pagetables`, `unevictable`, `slab`. **Compute its working set by hand** as `memory.current − inactive_file` and express it as a percentage of `memory.max`. State in one sentence whether this cgroup has a reclaimable cushion or not, and cite the two numbers that decide it.
2. **Cause a cgroup OOM.** Create `/sys/fs/cgroup/oom-demo`, set `memory.max=256M` and `memory.swap.max=0`, snapshot `memory.events`, then run `stress-ng --vm 1 --vm-bytes 1G --vm-keep --timeout 30s` inside it. Confirm the deltas in `max`, `oom`, and `oom_kill`.
3. **Capture and dissect the dmesg report.** Extract the OOM block and annotate the five facts: (a) triggering task, (b) victim PID/name, (c) victim `anon-rss` / `file-rss` / `total-vm`, (d) victim `oom_score_adj`, (e) scope, quoting the exact phrase (`Memory cgroup out of memory` vs `Out of memory`) *and* the `constraint=` value you used to decide. Convert at least one task-table row from pages to MiB by hand and show the multiplication.
4. **Prove the scope claim.** While the demo cgroup OOMs, capture `free -m` on the host showing large `available`. Write the one sentence that this pairing supports: the node was healthy and only the cgroup died.
5. **Predict, then verify, a victim.** Before running step 6, compute `oom_badness()` by hand for each task in the cgroup from its task-table row: `rss + swapents + pgtables_bytes/4096 + oom_score_adj × (memory.max/4096)/1000`. Write down your predicted victim. Then run it and check. If you were wrong, find the field you mis-read.
6. **(Stretch) Bias the choice.** Set `oom_score_adj=-998` on the larger of two hogs and show the smaller one is chosen. Then state, in one sentence, why doing the same thing to *every* process in a container buys nothing against that container's own `memory.max`.
7. **(Stretch) Group kill.** Set `memory.oom.group=1`, run two hogs, and show `oom_kill` jumping by 2 or more while `oom_group_kill` jumps by exactly 1. Explain the difference between what the two counters count.
8. **(Stretch) Soft landing.** Set `memory.high` below `memory.max` and rerun the hog. Show `high` climbing with `oom_kill` staying at 0, and capture `memory.pressure` to quantify what the throttle cost.

**Acceptance (feeds "Anatomy of a Container", [`../practice/anatomy-of-a-container/README.md`](../practice/anatomy-of-a-container/README.md)):** an **annotated OOM dmesg excerpt** — paste the real lines and annotate inline: (1) **who triggered** the OOM, (2) **who was killed** and its `anon-rss` / `total-vm`, converted from the task table's page counts with the arithmetic shown, (3) its **`oom_score_adj`**, (4) **cgroup-scoped or global**, with both the exact log phrase and the `constraint=` value you used to decide, and (5) one sentence on *why the victim couldn't be reclaimed* (anon + swap off, with the `swap: usage 0kB, limit 0kB` line as evidence). Include the hand-computed `oom_badness()` for at least two candidate tasks showing why the winner won. Tie it to the GPU failure mode: contrast a right-sized limit against a landmine using `memory.peak` vs `memory.max`, and state whether `memory.oom.group` (already `1` on a cgroup-v2 Kubernetes node) changes the blast radius for a multi-process training job.

## Common pitfalls

1. **Believing "40 GB free" on `free -m` means headroom.** Most of it is reclaimable page cache. Read `available` — the kernel's own estimate, computed from `MemFree` plus the file LRU plus `SReclaimable` minus watermarks — not `free` or `used`. *Mechanism:* `used` is defined as everything not free and not cache, so it systematically under-reports what the kernel can hand back and over-reports pressure.
2. **Assuming RSS predicts an OOM kill.** RSS counts reclaimable file cache and double-counts shared pages. The number Kubernetes acts on is **working set = `memory.current − inactive_file`**, not `anon + active_file` (a formula you will see repeated widely and which is wrong — it omits the kernel memory charged to the cgroup, which is real and does count against `memory.max`). *Mechanism:* only `inactive_file` is subtracted, because that is the only category the kernel has already judged cold.
3. **Treating every `OOMKilled` / exit 137 as identical.** cgroup-scoped, node-global, and kubelet eviction are three different events with different candidate sets and different fixes. *Fast tell:* `Memory cgroup out of memory` + a `memory:/swap:` usage pair (cgroup) vs `Out of memory` + a node zone dump (global) vs no kernel message at all and a pod status of `Evicted` (kubelet).
4. **Assuming `oom_score_adj` protects a process from its own limit.** It is added into the *same* score as footprint, then compared only against tasks in the same scope. Inside a cgroup OOM, every candidate typically carries the same value, so the bias cancels and someone still dies. *Only `-1000` is absolute* — it returns `LONG_MIN` before the footprint is computed, and is the one value `memory.oom.group` explicitly exempts.
5. **Expecting anonymous memory to be reclaimable when swap is off.** It isn't, and neither is long-term-pinned host memory. On a GPU node those two categories are most of the footprint. *Symptom:* PSI `full` barely leads the kill, `pgsteal_direct/pgscan_direct` collapses toward zero, and the transition from healthy to SIGKILL takes milliseconds.
6. **Sizing `memory.max` from an average or a `kubectl top` snapshot.** Both miss the peak that kills you. `memory.peak` is a high-water mark maintained by the kernel since cgroup creation and is the correct input. *Mechanism:* the kill is triggered by an instantaneous charge, not a moving average, so any smoothed metric will under-report the number that matters.
7. **Reading `pgtables_bytes` from the OOM task table as pages.** Every other memory column in that table is in pages; `pgtables_bytes` is in bytes, exactly as its name says. Multiplying it by 4096 gives you a number 4096× too large and a very confusing incident review.

## Self-check

**(a) Why does Kubernetes use working set (not RSS) for memory limits and eviction, and what is the exact formula?**

**Answer:** RSS includes cold, reclaimable file-cache pages the kernel will drop for free, and counts shared pages once per sharing process — so ranking pods by RSS would evict pods that merely hold page cache while sparing the pod whose anonymous footprint is the real problem. The number the kubelet uses is **`working_set = memory.current − memory.stat[inactive_file]`**, floored at zero, computed by the embedded cAdvisor and exported as `container_memory_working_set_bytes`. Note it is *not* `anon + active_file`: it retains all kernel memory charged to the cgroup (`slab`, `pagetables`, `kernel_stack`, `sock`, `percpu`), because that memory is real and does count against `memory.max`. Only `inactive_file` is subtracted, because that is precisely the set the kernel's own LRU has already judged cold and will reclaim on demand. This number drives both the node-pressure eviction decision (against `evictionHard["memory.available"]`, default `100Mi`) and what `kubectl top pod` shows you.

**(b) What exactly does `oom_score_adj = -997` on a Guaranteed pod's container achieve, and what does it not achieve?**

**Answer:** `oom_badness()` computes `points = rss + swapents + pgtables_pages`, then adds `oom_score_adj × (totalpages / 1000)` where `totalpages` is the scope's memory — the node's RAM for a global OOM, or that cgroup's `memory.max` for a cgroup OOM. So each point of `oom_score_adj` is worth one-thousandth of the scope. **Achieves:** in a *global* OOM on, say, a 512 GiB node, `-997` subtracts about 510 GiB of effective footprint, which places the container's processes below every BestEffort pod (`+1000`), every Burstable pod (`1000 − 1000×req/capacity`, clamped to `[3, 999]`), and every unbiased daemon — the kernel will kill workload pods before it touches Guaranteed ones. The kubelet and kube-proxy sit lower still at `-999`, so the node stays manageable. **Does not achieve:** any protection at all from that container's *own* `memory.max`. A cgroup OOM's candidate set is only the tasks inside that cgroup, and inside a Guaranteed container every task carries the identical `-997`, so the bias cancels exactly and the largest process still dies. Only `-1000` is absolute — it returns `LONG_MIN` before the footprint is even computed, and it is the sole exemption from `memory.oom.group`.

**(c) Given an OOM log line, who died and why — walk every field.**

**Answer:** Take `Memory cgroup out of memory: Killed process 5123 (stress-ng-vm) total-vm:1064512kB, anon-rss:252860kB, file-rss:928kB, shmem-rss:0kB, UID:0 pgtables:768kB oom_score_adj:0`.
**Scope:** the message prefix is the `message` argument to `oom_kill_process()`, and it is `"Memory cgroup out of memory"` for a memcg OOM versus `"Out of memory"` for a global one — so this is a **cgroup** OOM; the cgroup hit its own `memory.max` and the node was fine. Confirm with the `memory: usage … limit … failcnt` block above it (printed only for memcg OOMs) and with `constraint=CONSTRAINT_MEMCG` on the `oom-kill:` line.
**Who:** PID 5123, comm `stress-ng-vm`, running as UID 0.
**Why it was chosen:** its footprint dominated the scope. `anon-rss` 252,860 KiB ≈ 247 MiB against a 256 MiB cap; `oom_score_adj:0` means no bias, so `oom_badness()` reduced to raw pages: 63,215 anon + 232 file + 786432/4096 = 192 pagetable pages ≈ 63,639, against a parent process scoring ~427. 149× apart.
**Why it couldn't be saved:** the footprint is **anonymous**, and the companion `swap: usage 0kB, limit 0kB` line shows swap was unavailable, so reclaim had nothing to scan — `file 0`, `inactive_file 0` in the cgroup stats confirms it. Sixteen rounds of memcg reclaim freed nothing and OOM was the only exit.
**Intent vs reality:** `total-vm:1064512kB` = 1.01 GiB shows it asked for a gigabyte; `anon-rss` shows only 247 MiB ever became resident before the kill. `pgtables:768kB` is the page-table cost of that mapping and is in **bytes**, not pages, unlike the neighbouring RSS figures.

**(d) A multi-rank training job's container OOMs. Why does `memory.oom.group` matter more than which single process gets killed — and is it even something you need to turn on?**

**Answer:** Without group kill, the killer picks one process — typically the largest rank — and kills only it. The surviving ranks then block inside a collective operation (an all-reduce waiting on a peer that will never post its buffer), producing a hung job that holds every one of its GPUs allocated and idle until some outer timeout fires. Nothing crashes, so nothing alerts; you leak GPU-hours silently. With `memory.oom.group=1` the kernel kills every task in the cgroup atomically (exempting only `oom_score_adj == -1000`), the container exits once with 137, the orchestrator sees an unambiguous failure, and the job restarts from its last checkpoint. **And you do not need to turn it on:** on cgroup v2 the kubelet sets `memory.oom.group=1` on every container cgroup by default, because its `singleProcessOOMKill` option defaults to `false` on cgroup-v2 Linux. What you actually need to know is (i) it is scoped to the *container*, not the pod, so sidecars in other containers survive; (ii) it will also kill your `kubectl exec` shell inside that container; and (iii) `memory.events` reports it as `oom_kill` incrementing by the number of tasks while `oom_group_kill` increments by exactly one event.

**(e) A pod is being killed and you suspect the limit is too low. Which numbers do you collect, in what order, and what do they prove?**

**Answer:** (1) `memory.max` and `memory.peak` — if `peak == max`, the workload has repeatedly filled the limit and the limit is the binding constraint, not a leak. (2) `memory.current − inactive_file` as a fraction of `memory.max` — the working set; above ~95% sustained, there is no cushion. (3) `anon`, `unevictable`, and `file` from `memory.stat` — if `anon + unevictable` is most of `current` and `file` is small, reclaim has nothing to work with and no throttle will save it. (4) `memory.stat`'s `pgscan_direct` and `pgsteal_direct` — a collapsing steal/scan ratio proves the kernel is scanning hard and freeing nothing. (5) `memory.events` `max` versus `oom_kill` — a high `max` with zero `oom_kill` means reclaim is saving you every time and you are living on the edge; both climbing means it stopped saving you. (6) `memory.pressure` `full` — how much wall-clock time the workload has already lost to this. Together these distinguish *undersized limit* (peak pinned at max, working set ~100%, mostly anon) from *leak* (peak climbing monotonically across restarts) from *cache-heavy but healthy* (large `file`, large `inactive_file`, working set well under the limit).

## Connections & what's next

This lesson depends on lesson 03's cgroup v2 mechanics (`memory.max` *is* the file from that lesson; the Kubernetes `limits.memory` translation is identical) and on lesson 04's PSI (memory `full` pressure is the leading indicator that precedes the event covered here — PSI tells you it's coming, this lesson tells you what happens and how to read the aftermath). Two specific hand-offs go forward. To lesson 06, **Hugepages, THP, and NUMA locality**: the `pagetables` line in `memory.stat` and the 80 MiB you computed in §2 are the direct motivation for larger page sizes, and the `unevictable` pinned buffers from §10 are exactly the pages that cannot be migrated or compacted — which is the mechanism behind half of lesson 06's THP and NUMA behaviour. To lesson 09, **perf / ftrace / USE method**: `pgscan_direct` and `workingset_refault_*` are the saturation half of the USE method for memory, and this lesson gave you the utilization and error halves.

## References & further reading

**Primary sources**

- **cgroup v2 admin guide — Memory controller** — <https://docs.kernel.org/admin-guide/cgroup-v2.html> — the canonical semantics for every file in §3's table, the complete `memory.stat` key list, and the `memory.events` field definitions. Read the "Memory Interface Files" section end to end once; it is the only place where `memory.high`'s "never invokes the OOM killer" guarantee and `memory.min`'s "OOM killer is invoked instead" behaviour are stated normatively. *Correction applied from this source:* `memory.events` has six fields on current kernels (`low`, `high`, `max`, `oom`, `oom_kill`, `oom_group_kill`, plus `sock_throttled`), not four, and `max` counts times usage was *about to* exceed the limit rather than times reclaim ran hard.
- **`mm/oom_kill.c`** — <https://github.com/torvalds/linux/blob/master/mm/oom_kill.c> — `oom_badness()`, `select_bad_process()`, `dump_tasks()`, `__oom_kill_process()` and the OOM reaper. Every log format string quoted in §8 is a `pr_info`/`pr_err` in this file, which is why you can predict the report's fields exactly. *Correction applied:* the badness score is a raw page count (`rss + swapents + pgtables/PAGE_SIZE`) plus `adj × totalpages/1000`, not a 0–1000 permille; the permille framing describes the pre-2.6.36 heuristic and the normalized `/proc/<pid>/oom_score` view.
- **`mm/memcontrol.c`** — <https://github.com/torvalds/linux/blob/master/mm/memcontrol.c> — `try_charge()`, `calculate_high_delay()` (with the kernel's own overage→milliseconds table reproduced in §4), `mem_cgroup_print_oom_meminfo()` and `mem_cgroup_print_oom_context()`. Read it to see that `MAX_RECLAIM_RETRIES` is 16 and that memcg OOM sets `totalpages = mem_cgroup_get_max(memcg)`.
- **`Documentation/admin-guide/sysctl/vm.rst`** — <https://docs.kernel.org/admin-guide/sysctl/vm.html> — real defaults for `swappiness` (60, range 0–200), `min_free_kbytes`, `watermark_scale_factor` (10 = 0.1% of memory), `oom_dump_tasks` (1), `oom_kill_allocating_task` (0), `panic_on_oom` (0), `vfs_cache_pressure` (100). Use it to check any tuning advice you are given against what the knob actually does.
- **`Documentation/filesystems/proc.rst`, `/proc/meminfo` section** — <https://docs.kernel.org/filesystems/proc.html> — the authoritative definition of `MemAvailable` (derived from `MemFree`, the file LRU sizes and `SReclaimable`, minus zone low watermarks and a page-cache reserve) and of every field `free -m` summarises.
- **Kubernetes kubelet source — `pkg/kubelet/qos/policy.go` and `pkg/kubelet/kuberuntime/kuberuntime_container_linux.go`** — <https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/qos/policy.go> — the exact `oom_score_adj` constants (−999 kubelet/kube-proxy, −997 Guaranteed and node-critical, +1000 BestEffort, computed-and-clamped Burstable) and the `memory.oom.group` assignment. *Correction applied:* the widely-repeated "−998 for Guaranteed" is not a value the kubelet uses.

**Real-world engineering**

- **LINE Engineering — "Who murdered my lovely Prometheus container in Kubernetes cluster?"** — <https://engineering.linecorp.com/en/blog/prometheus-container-kubernetes-cluster> — a production OOM investigation that walks cgroup limits, `oom_score_adj` and `dmesg` evidence to a root cause. Read it as a worked instance of §8's five-facts drill on someone else's incident.
- **Meta Engineering — "Open-sourcing oomd"** — <https://engineering.fb.com/2018/07/19/production-engineering/oomd/> — why the in-kernel killer is structurally too late (it only runs after an allocation has already failed) and too blunt (a page count plus a bias, with no notion of workload importance), and what a PSI-driven userspace policy layer above it looks like. The argument for treating the kernel killer as a safety net rather than a policy engine.

**Deeper dives**

- **Brendan Gregg, *Systems Performance* (2nd ed.), Memory chapter** — <https://www.brendangregg.com/systems-performance-2nd-edition-book.html> — virtual vs resident vs working set, the page cache, reclaim and swap behaviour, with the observability tooling for each. The best single narrative treatment of the mental model this lesson formalises against the source.
