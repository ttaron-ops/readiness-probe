---
lesson: "01b.3"
title: "cgroups v2 and Kubernetes resource enforcement"
module: "01b"
concept: "cgroups v2 and Kubernetes resource enforcement"
status: not-started
est_time: "9h"
prev: "02-namespaces.md"
next: "04-psi.md"
artifacts: []
sources: 14
---

# 01b.3 · cgroups v2 and Kubernetes resource enforcement

> **Concept.** The unified cgroup-v2 hierarchy and its controllers — and exactly how a pod's requests, limits, and QoS class become cpu.max, cpu.weight, memory.max, and memory.high on the node.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

[Lesson 02](02-namespaces.md) established that a container is not a kernel object — it is a process wrapped in namespaces, and namespaces only virtualize *views*: PID space, network stack, mount table, hostname. They isolate what a process can **see**. They do nothing to limit what it can **consume**. A perfectly namespaced process can still fork until the box falls over or burn every core on the node.

That is the gap this lesson closes. Lesson 01 gave you the second pointer on `task_struct` — `cgroups`, the control-group membership — and this lesson follows it. cgroups are the mechanism that fences *consumption*, not visibility, and together namespaces + cgroups + a rootfs are the complete, no-magic definition of "container" the module has been building toward.

This is also the anchor lesson. Everything downstream reads the same `/sys/fs/cgroup` tree from a different angle: [04 — PSI](04-psi.md) takes the `*.pressure` files, [05 — memory & OOM](05-memory-and-oom.md) takes `memory.events` and the reclaim path, [06 — hugepages/THP/NUMA](06-hugepages-thp-numa.md) takes `cpuset.mems` and `hugetlb.*`, and [10 — systemd as cgroup manager](10-systemd-cgroups-and-block-io.md) takes delegation and the io controller.

## Why this matters

This is the "how containers really work" story and the cost/efficiency story in one file. A container is not a lightweight VM; it is an ordinary Linux process whose consumption is fenced by cgroups. When you write `resources.limits.cpu: 500m` in YAML, nothing magical happens: a Go function multiplies by a period, the kubelet hands the result to the container runtime, and runc writes the ASCII string `50000 100000` into a file called `cpu.max`. Understanding that write — and being able to reproduce it by hand on a live node — is the difference between operating Kubernetes and understanding it.

It is also the single most common container-Linux interview question, usually posed as the **throttling paradox**: *"Your service has p99 latency spikes and is being CPU-throttled, but the dashboard shows the pod averaging well under its limit. How is that possible?"* The answer lives entirely in `cpu.max`, the 100 ms period, the 5 ms per-CPU quota slice, and `cpu.stat`'s `nr_throttled`. This lesson derives it with real arithmetic rather than asserting it, because the hand-wavy version ("bursty apps throttle") does not survive a follow-up question.

Finally, this is the foundation of cost attribution. Every dollar of GPU-node spend is charged through the cgroup tree — `kubepods.slice` and its QoS children are literally the accounting boundaries the kubelet creates. On a fleet where one GPU node runs into tens of dollars an hour, the walk `kubepods.slice → QoS slice → pod slice → container scope` *is* the chargeback report, with no gaps and no double-counting. If you cannot read the tree, you cannot attribute the cost.

## What's new here (calibration)

Operationally you already know that `requests` drive scheduling and act as a floor, `limits` are a ceiling, and that the request/limit relationship determines QoS class (Guaranteed/Burstable/BestEffort), which drives eviction and OOM order. That is enough to pass CKA. None of it is re-taught.

What is genuinely new:

- Opening the kernel files those YAML fields become, and reading their **evidence counters** (`cpu.stat`, `memory.events`) as proof rather than inferring from a dashboard.
- The v1→v2 rationale and the four structural rules — availability, top-down enablement, no-internal-process, delegation containment — that explain *why* the tree looks the way it does and why you cannot just `echo` a PID wherever you like.
- The CFS bandwidth machinery at the level of the global pool, the per-CPU silos, and the **5 ms `sched_cfs_bandwidth_slice_us` transfer granularity** — which is the actual mechanism behind throttling at low average utilization, and the thing the Indeed postmortem is really about.
- **The exact `requests.cpu → cpu.weight` conversion — and the fact that there are *two different* conversions on the same node.** The kubelet uses a linear formula for the pod and QoS slices; runc/crun use a completely different quadratic-in-log₂ formula for the container scope. The same 500m request produces `cpu.weight 20` at one level and `cpu.weight 59` at another. This is the single most checkable, most-often-wrong fact in the whole area.
- The precise systemd slice names the kubelet builds, including the `-` → `_` escaping of pod UIDs, so you can locate any pod's cgroup by hand.
- `cpuset`/NUMA pinning, `pids`, `io`, the **MemoryQoS** formula with its real 0.9 factor, and node swap — the fleet-specific layer CKA never covers.

## Core concepts

### 1. What a cgroup is, and why v2 exists

A **cgroup** (control group) is a set of processes plus a set of resource knobs applied to them as a unit. Two things make it more than a process group: **hierarchy** (a cgroup's limits bound its descendants, always) and **controllers** (per-resource kernel modules that account and enforce). It is exposed as a pseudo-filesystem — directories are cgroups, files are knobs.

**v1 gave each controller its own independent hierarchy.** `/sys/fs/cgroup/cpu`, `/sys/fs/cgroup/memory`, `/sys/fs/cgroup/blkio` were separate trees, and a process could sit at a different position in each. That flexibility turned out to be a mistake. Consider writeback: dirty page cache is charged to the memory controller, but flushing it is block I/O charged to the blkio controller. If the two controllers cannot agree on *which* cgroup a page belongs to — because the process is in `/memory/A` and `/blkio/B` — then writeback throttling is unimplementable. It genuinely was, for years.

**v2 collapses everything into one unified tree.** A cgroup is one directory; every controller you enable acts on the same set of processes at the same node. That is not a tidiness argument, it is what makes cross-controller features (writeback accounting, `io.latency`, memory-pressure-aware I/O) possible at all.

```
   cgroup v1 — one tree PER CONTROLLER            cgroup v2 — ONE tree
   ═══════════════════════════════════            ════════════════════

   /sys/fs/cgroup/cpu/                            /sys/fs/cgroup/
     ├── docker/                                    ├── cgroup.controllers
     │     └── abc123/  ← task X here               ├── cgroup.subtree_control
     └── system.slice/                              ├── cgroup.procs
                                                    │
   /sys/fs/cgroup/memory/                           ├── system.slice/
     ├── kubepods/                                  │     ├── cpu.max
     │     └── burstable/ ← task X ALSO here        │     ├── memory.max
     └── system.slice/                              │     └── sshd.service/
                                                    │
   /sys/fs/cgroup/blkio/                            └── kubepods.slice/
     └── ...            ← and here                        ├── cpu.weight
                                                          ├── memory.max
   Three different positions for one task.                ├── cpu.pressure
   The memory controller cannot know what                 └── …/…/…/*.scope
   the io controller is doing to the same                        └── task X
   page. Writeback accounting is impossible.
                                                    ONE position. Every
                                                    controller sees the same
                                                    node. Writeback works.
```

Detecting which you are on takes one command:

```
$ stat -fc %T /sys/fs/cgroup
cgroup2fs                     # v2 unified.  "tmpfs" here means v1 or hybrid.
```

Kubernetes cgroup v2 support went **GA in v1.25**, and every current mainstream distribution boots into the unified hierarchy by default. Assume v2 and check.

### 2. The four structural rules

These rules explain every "why can't I just…" question about the tree.

**Rule 1 — Availability.** `cgroup.controllers` lists which controllers are usable *in this cgroup*. It is read-only and is set by the parent.

```
$ cat /sys/fs/cgroup/kubepods.slice/cgroup.controllers
cpuset cpu io memory hugetlb pids rdma misc
```

**Rule 2 — Top-down enablement.** A controller acts on a cgroup's **children** only if the cgroup lists it in `cgroup.subtree_control`. You enable with `+`, disable with `-`:

```
$ cat /sys/fs/cgroup/kubepods.slice/cgroup.subtree_control
cpuset cpu io memory pids
$ echo "+cpu +memory -io" > cgroup.subtree_control
```

The consequence that catches people: **the interface files in a directory are owned by its parent, not by itself.** `cpu.max` appears in cgroup `C` because `C`'s parent enabled `cpu` in its `subtree_control`. Anything not prefixed `cgroup.` is a knob the parent is using to constrain you. This is exactly why delegation (Rule 4) can hand you a subtree without handing you the ability to raise your own limits.

**Rule 3 — No internal processes.** A non-root cgroup that has domain controllers enabled in `cgroup.subtree_control` **may not contain processes of its own**. Processes live only on leaves.

The reason is arithmetic, not aesthetics. If cgroup `P` had both a child cgroup `C` (weight 100) and its own resident processes, the CPU controller would have to decide how those bare processes compete against `C` — and there is no principled answer, because bare processes have no weight of their own. The rule rules the question out of existence. The root cgroup is exempt, because it has to hold kernel threads and unattributable consumption.

Practical consequence: to start controlling a populated cgroup, you must first create children and move every process into them, *then* enable controllers. This is precisely the dance the kubelet does at startup, and why `kubepods.slice` is empty of processes while its descendants are not.

**Rule 4 — Delegation containment.** A cgroup is delegated by granting a less-privileged user write access to the directory and to `cgroup.procs`, `cgroup.threads`, and `cgroup.subtree_control` — but *not* to the resource-control files, which belong to the parent. The delegatee can build a sub-hierarchy and shuffle processes around inside it, and cannot move processes in from outside or out to elsewhere: the kernel requires write access to `cgroup.procs` of the **common ancestor** of source and destination. Nothing in a delegated subtree can escape the parent's limits, ever. Every container runtime depends on this, and so does systemd's `Delegate=yes`.

There is also **threaded mode** (`cgroup.type` = `threaded`), which relaxes the no-internal-process rule for controllers that can meaningfully arbitrate between threads — currently `cpu`, `cpuset`, `perf_event`, and `pids`. Kubernetes does not use it; systemd does, for some units.

**The core interface files present in every cgroup:**

| File | Type | What it does |
|---|---|---|
| `cgroup.procs` | rw | PIDs in this cgroup. Writing a PID **moves the whole process** (all threads). |
| `cgroup.threads` | rw | Same at thread granularity; only meaningful in threaded mode. |
| `cgroup.controllers` | ro | Controllers available here. |
| `cgroup.subtree_control` | rw | Controllers pushed down to children. |
| `cgroup.type` | rw | `domain` / `domain threaded` / `domain invalid` / `threaded`. |
| `cgroup.events` | ro | `populated 0|1` — flips when the subtree empties. `poll()`able, which is how runtimes detect "container exited" without polling. |
| `cgroup.stat` | ro | `nr_descendants`, `nr_dying_descendants`. |
| `cgroup.pressure` | rw | `0` disables PSI accounting for this cgroup (Linux ≥ 6.1). Non-hierarchical. → [lesson 04](04-psi.md) |
| `cgroup.max.depth`, `cgroup.max.descendants` | rw | Structural limits on the subtree. |

Note `CLONE_INTO_CGROUP` from [lesson 02](02-namespaces.md): `clone3(2)` can place a child directly into a target cgroup at birth, avoiding the race window where a freshly forked process is briefly in its parent's cgroup. Modern runtimes use it.

### 3. The CPU controller

Four knobs and one evidence file. All time values are **microseconds**.

| File | Default | Type | Meaning |
|---|---|---|---|
| `cpu.weight` | `100` | rw, `[1, 10000]` | Proportional share **under contention**. Never a cap. |
| `cpu.weight.nice` | `0` | rw, `[-20, 19]` | Same knob expressed in nice units; coarser. |
| `cpu.max` | `max 100000` | rw, `$MAX $PERIOD` | Hard bandwidth cap. `max` = uncapped. |
| `cpu.max.burst` | `0` | rw, `[0, $MAX]` | Bankable unused quota (Linux ≥ 5.14). |
| `cpu.idle` | `0` | rw | `1` makes the whole cgroup `SCHED_IDLE`-class relative to peers. |
| `cpu.stat` | — | ro | The counters. Exists **whether or not** the controller is enabled. |
| `cpu.pressure` | — | rw | PSI → [lesson 04](04-psi.md) |

**`cpu.weight` is a share, not a cap.** A cgroup with weight 100 competing against one with weight 300 gets 25% of a contended CPU — and 100% of it when the other cgroup is idle. Weight only does anything when there is contention. Internally it feeds the same `sched_entity.load.weight` that `nice` feeds for individual tasks ([lesson 01](01-processes-and-scheduling.md) §5), applied at the group level.

**`cpu.max` is a cap, and it is absolute.** Two numbers:

```
$ cat /sys/fs/cgroup/.../cpu.max
50000 100000
│     └── PERIOD: the enforcement window, in µs. 100000 = 100 ms.
└──────── QUOTA:  CPU-microseconds the cgroup may consume per window.

  50000 µs of CPU time per 100000 µs of wall time  =  0.5 CPU
```

Read the units carefully, because this is where the intuition breaks: quota is **CPU-time summed across all cores**, and period is **wall time**. A cgroup with `200000 100000` on a 64-core node may use 2 CPU-seconds per second — as 2 threads for the whole period, or as 64 threads for 3.1 ms and then nothing.

**`cpu.stat` is the evidence.** Three fields always; five more once the controller is enabled:

```
$ cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/\
kubepods-burstable-pod3f2a_9c1d_4e77_b0a2_1f5c8d3e6b40.slice/\
cri-containerd-9c1d4e77b0a21f5c8d3e6b40aa11bb22cc33dd44ee55ff66.scope/cpu.stat
usage_usec 41230000        # total CPU time consumed (user + system), µs
user_usec 40010000         # of which, userspace
system_usec 1220000        # of which, kernel
nr_periods 612             # enforcement windows elapsed since the cap was set
nr_throttled 598           # windows in which this cgroup hit its quota
throttled_usec 55082140    # cumulative time tasks were runnable-but-frozen, µs
nr_bursts 0                # windows in which banked burst was spent
burst_usec 0               # cumulative time spent above quota using burst
```

Two derived numbers do all the work:

- **Throttle ratio** = `nr_throttled / nr_periods`. Above ~0.1 it is worth investigating; near 1.0 the cgroup is capped in essentially every window.
- **Throttle rate** = Δ`throttled_usec` / Δwall-time. This is aggregate stall across all threads, so on a 32-thread app it can legitimately exceed 1.0 seconds per wall-second. Divide by thread count for a per-thread figure.

`nr_periods` only advances while a quota is set. A cgroup with `cpu.max = max` shows `nr_periods 0` forever — which is the correct answer to "why are my throttle metrics zero," not evidence of health.

### 4. CFS bandwidth control: the mechanism, and why low average CPU still throttles

**The problem.** Charge CPU time against a per-cgroup budget on a machine with many cores, without taking a global lock on every context switch. A naive implementation would decrement one shared counter per scheduling decision — a cache line ping-ponging across 64 sockets thousands of times per second.

**The design.** The kernel keeps a **global pool** per cgroup, refilled to `quota` at every period boundary, and transfers budget to **per-CPU silos** in fixed-size slices on demand. The slice size is a system-wide sysctl:

```
$ cat /proc/sys/kernel/sched_cfs_bandwidth_slice_us
5000                          # 5 ms — the default
```

When a thread of the cgroup becomes runnable on CPU *n* and CPU *n*'s silo is empty, the scheduler pulls one 5 ms slice from the global pool. Threads run against their local silo with no global locking. When a CPU's silo runs dry and the global pool is empty, **every thread of that cgroup on that CPU is throttled** — dequeued and left off the run queue until the next period refill. Larger slices reduce global-lock traffic; smaller slices allow finer-grained consumption. That is the entire trade-off the sysctl exposes.

Here is a period, drawn:

```
  cpu.max = "200000 100000"   → quota 200 ms per 100 ms period  (2 CPUs)
  sched_cfs_bandwidth_slice_us = 5000  (5 ms)
  App: 32 runnable threads, spread over 32 CPUs on a 64-core node

  global pool ██████████████████████████████████████████  200 ms  (t=0, refill)

  t=0.0 ms   32 CPUs each pull one 5 ms slice   → 160 ms leaves the pool
             global pool ████████                40 ms remaining
  t=5.0 ms   silos empty; 8 more slices available → 8 CPUs continue,
             24 CPUs are THROTTLED (pool has nothing for them)
  t=6.25 ms  the last of the 200 ms is consumed  → ALL 32 threads throttled

  ┌──────────────────────── period 0 : 100 ms ─────────────────────────┐
  │████│░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
  │6.25│                       93.75 ms THROTTLED                      │
  │ ms │  every thread dequeued; the 62 idle cores CANNOT be used       │
  └────┴───────────────────────────────────────────────────────────────┘
   t=0                                                              t=100
                                                                      │
                                                          period boundary:
                                                          hrtimer fires,
                                                          pool refilled to
                                                          200 ms, all threads
                                                          re-enqueued
  ┌──────────────────────── period 1 : 100 ms ─────────────────────────┐
  │███│░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
  └───┴────────────────────────────────────────────────────────────────┘
   3.75 ms — the remaining work finishes here

  Observed: cpu.stat nr_throttled increments in period 0 only.
            Wall-clock latency for a 10 ms unit of work: 103.75 ms.
```

**Worked derivation — throttling at 32% of the limit.** Take a data-loader sidecar:

```
  Given
    limits.cpu: 2                    → cpu.max = "200000 100000"
    threads:    32                   (PyTorch DataLoader num_workers=32)
    work per batch:  10 ms of CPU per thread → 32 × 10 = 320 core-ms
    batch arrival:   every 500 ms

  Step 1 — how fast can the quota be spent?
    32 threads run in parallel, so quota drains at 32 core-ms per wall-ms.
    200 core-ms of quota ÷ 32 threads = 6.25 ms of wall time.

  Step 2 — period 0
    t = 0        batch arrives, 320 core-ms of demand
    t = 6.25 ms  quota exhausted → ALL threads throttled
    t = 100 ms   period boundary, quota refilled
    consumed: 200 core-ms.  remaining: 120 core-ms.

  Step 3 — period 1
    120 core-ms ÷ 32 threads = 3.75 ms
    t = 103.75 ms  batch complete.

  Step 4 — the two numbers that disagree
    LATENCY:  103.75 ms actual vs 10 ms if unthrottled  → 10.4× amplification
    AVERAGE:  320 core-ms of CPU per 500 ms of wall time
              = 0.64 CPU  ÷  2 CPU limit  =  32% of limit

  Step 5 — what cpu.stat shows over one second (2 batches)
    nr_periods    10
    nr_throttled   2        → throttle ratio 0.20
    throttled_usec ≈ 2 × 93.75 ms × 32 threads ≈ 6,000,000 µs
                              └── aggregate across threads: 6 s of stall
                                  in 1 s of wall time. This is correct,
                                  not a bug: 32 threads × 187.5 ms.
```

**The dashboard says 32%. The user says the p99 is 10× the p50. Both are true.** Average utilization is an *integral* over the period; throttling is a property of the *distribution* within it. They measure different things and neither implies the other.

**The second, subtler mechanism — slice hoarding on wide machines.** Quota moves to silos in 5 ms units, so at most `quota / 5000` CPUs can hold a slice at one instant:

```
  limits.cpu: 1   → quota = 100 ms/period
  slice           =   5 ms
  max CPUs holding a slice simultaneously = 100 / 5 = 20

  On an 88-core node with 40 runnable threads spread over 40 CPUs:
     20 CPUs get a slice and run
     20 CPUs find the global pool empty → THROTTLED from t≈0
  …even though the cgroup has consumed nothing yet.
```

This is exactly what Indeed described: *"If an application only has 100 ms of quota (1 CPU) and the kernel uses 5 ms slices, the application can only use 20 cores before running out of quota."* Widening the thread pool makes throttling **worse**, not better, which is deeply counter-intuitive until you have seen the silo model.

**The kernel-bug chapter.** Commit `512ac999` (Linux 4.18) fixed an unrelated bandwidth-timer clock-drift bug, and in doing so made per-CPU slices genuinely *expire* at period end rather than being usable later. Combined with slice hoarding, on an 88-core machine a cgroup could have up to 87 ms of its 100 ms period stranded in idle per-CPU silos and be throttled while its aggregate usage looked modest. The fix, commit `de53fd7aedb1` — *"sched/fair: Fix low cpu usage with high throttling by removing expiration of cpu-local slices"* — removed expiration entirely and landed in **Linux 5.4**. The current kernel documentation records the resulting behaviour: a slice assigned to a CPU does not expire, and all but 1 ms of it (`min_cfs_rq_runtime`) is returned to the global pool when that CPU's threads all become unrunnable.

**`cpu.max.burst`** (Linux ≥ 5.14) is the sanctioned way to smooth bursty workloads: unused quota accumulates, up to the burst budget, and can be spent in a later spike. `cpu.stat`'s `nr_bursts`/`burst_usec` account it. It defaults to `0`, and **Kubernetes exposes no pod field for it** — worth knowing exists when someone asks "can't the kernel just let it burst?"

**GOMAXPROCS and the same mismatch in userspace.** `/proc/cpuinfo` and `nproc` report the **host's** CPU count inside a container, because no namespace virtualizes them (see §9). A runtime that sizes its thread pool off `NumCPU()` on a 64-core node with a 2-CPU limit runs 64 worker threads against 200 ms of quota — the worst possible shape for the silo model. Historically the fix was `go.uber.org/automaxprocs`; **as of Go 1.25 (August 2025) the runtime reads the cgroup CPU limit and derives `GOMAXPROCS` from it by default.** The same mismatch bites the JVM (`-XX:ActiveProcessorCount`, or a container-aware JVM), Node.js worker pools, OpenMP (`OMP_NUM_THREADS`), and anything sized off `nproc` — including, notably, PyTorch's default intra-op thread count.

### 5. The memory controller

Memory is stateful — you cannot un-allocate it the way you stop granting CPU time — so it has a richer interface: two protection levels, two limit levels, and an event log. **All values are bytes.**

| File | Default | Semantics |
|---|---|---|
| `memory.min` | `0` | **Hard protection.** Memory below this is never reclaimed under any pressure. If nothing else is reclaimable, the OOM killer fires instead. |
| `memory.low` | `0` | **Best-effort protection.** Not reclaimed unless nothing unprotected is left. |
| `memory.high` | `max` | **Throttle.** Above it, tasks are put into synchronous direct reclaim and their allocations are slowed. **Never invokes the OOM killer**; the limit may be breached under extreme conditions. |
| `memory.max` | `max` | **Hard limit.** If usage would exceed it and reclaim cannot recover, the **cgroup OOM killer** kills a process inside this cgroup. |
| `memory.swap.max` | `max` | Per-cgroup swap ceiling. |
| `memory.current` | — | Current charged usage (this cgroup + descendants). |
| `memory.peak` | — | High-water mark; writable to reset. |
| `memory.oom.group` | `0` | `1` = kill **all** tasks in the cgroup together or none. |
| `memory.reclaim` | — | Write-only: `echo 1G > memory.reclaim` triggers proactive reclaim. |
| `memory.events` / `.local` | — | The counters (below). |
| `memory.stat` | — | ~60-key breakdown (below). |
| `memory.pressure` | — | PSI → [lesson 04](04-psi.md) |

Protection is proportional, not binary: above the effective min/low boundary, pages are reclaimed **in proportion to the overage**, so a cgroup slightly over its protection is reclaimed gently and one far over is reclaimed hard. Effective boundaries are also clamped by ancestors — you cannot protect more than your parent protects.

**The `high`/`max` ladder as a state machine:**

```
  memory.current climbing ───────────────────────────────────────────▶

  0 ─────────── memory.min ────── memory.low ────── memory.high ────── memory.max
  │                  │                 │                 │                  │
  │  NEVER           │  reclaimed      │  normal         │  DIRECT RECLAIM  │  OOM
  │  reclaimed       │  only if        │  reclaim        │  + ALLOCATION    │  KILL
  │                  │  nothing        │  eligible       │  THROTTLING      │
  │                  │  else left      │                 │                  │
  └──────────────────┴─────────────────┴─────────────────┴──────────────────┘
                                                          │                  │
                                       memory.events.high++│   memory.events.max++
                                       memory.pressure ↑↑  │   memory.events.oom++
                                                           │   memory.events.oom_kill++
                                                           │   → SIGKILL, exit code 137
                                       ← this is the graceful
                                         valve most clusters
                                         never turn on

  What "throttle" means concretely at memory.high:
    the ALLOCATING TASK itself runs the reclaim, synchronously, in its own
    context — scanning LRU lists, writing back dirty pages, swapping anon
    pages if swap exists — and its allocation does not return until reclaim
    succeeds or usage drops. That stall IS the throttle. It shows up as
    memory.pressure rising and as latency in the application, with no log
    line anywhere. If anon memory is unreclaimable and there is no swap,
    pressure just builds until memory.max is reached and the OOM path fires.
```

**`memory.events` is how you prove what happened.** Five counters that turn "the pod restarted" into a diagnosis:

| Key | Meaning |
|---|---|
| `low` | Times the cgroup was reclaimed despite being under `memory.low` — the low boundary is over-committed. |
| `high` | Times processes were throttled into direct reclaim because `memory.high` was exceeded. |
| `max` | Times usage was about to exceed `memory.max`. |
| `oom` | Times usage hit the limit and an allocation was about to fail. |
| `oom_kill` | **Processes actually killed by any OOM killer in this cgroup.** |
| `oom_group_kill` | Times a group-OOM (`memory.oom.group=1`) fired. |

The fields in `memory.events` are **hierarchical** — a value can change because of an event in a descendant. `memory.events.local` is the same set, non-hierarchical. That distinction is the whole diagnosis in one common case:

> A container restarts with **exit code 137**. If `memory.events` in its own cgroup shows `oom_kill 0`, it was **not** killed by its own limit — it was killed by the node-level OOM killer or evicted by the kubelet. Two completely different fixes.

**`memory.stat`** breaks the footprint down into ~60 keys. The ones that matter operationally:

| Key | Meaning |
|---|---|
| `anon` | Anonymous pages — heap, stacks, `MAP_ANONYMOUS`. **This is what gets you OOM-killed**: unreclaimable without swap. |
| `file` | Page cache, including tmpfs and shared memory. Reclaimable. |
| `shmem` | The tmpfs/shm subset of `file` — **includes `/dev/shm`**, which is *not* reclaimable while mapped. |
| `kernel`, `slab`, `kernel_stack`, `pagetables`, `sock` | Kernel-side charges. `pagetables` gets large with many processes or hugepages. |
| `inactive_anon`/`active_anon`/`inactive_file`/`active_file`/`unevictable` | LRU list sizes — the reclaim scanner's working set. |
| `workingset_refault_file` / `_anon` | **Pages evicted and then faulted back in.** A rising rate is the definition of thrashing. |
| `pgscan` / `pgsteal` | Pages scanned vs actually reclaimed. A low steal:scan ratio means reclaim is working hard for little return. |
| `pgmajfault` | Major faults — had to go to disk. |
| `pgscan_direct` vs `pgscan_kswapd` | Direct reclaim (synchronous, in the allocator's context — costs latency) vs background. |

The pair to internalise is **`workingset_refault_file` rising while `pgsteal/pgscan` falls**: the kernel is evicting pages the workload immediately needs back, and spending more and more scanning to find each one. That is the pre-OOM thrash signature, and [lesson 04](04-psi.md) shows the same event as `memory.pressure full` climbing.

### 6. pids, io, and cpuset

**`pids`** is the fork-bomb defence, and it fails in a way that is easy to misdiagnose.

| File | Meaning |
|---|---|
| `pids.max` | Hard ceiling (default `max`). Counts **TIDs**, i.e. threads, not processes. |
| `pids.current` | Current count including descendants. |
| `pids.peak` | High-water mark. |
| `pids.events` / `.local` | `max` — times the limit was hit. |

A `fork()`/`clone()` past the limit fails with **`EAGAIN`**, which surfaces in applications as "resource temporarily unavailable", `pthread_create` failures, or a Go runtime panic — *not* as an OOM. Kubernetes exposes it as the kubelet's `podPidsLimit` (per-pod, default `-1` = unlimited). Note that organisational moves are not blocked, so `pids.current > pids.max` is possible if you lower the limit or move processes in; only `fork`/`clone` are refused.

**`io`** — block-device bandwidth and IOPS.

```
$ cat io.max
8:16 rbps=2097152 wbps=max riops=max wiops=120
│    │             │        │         └── max write IOPS
│    │             │        └──────────── max read IOPS
│    │             └───────────────────── max write bytes/sec
│    └─────────────────────────────────── max read bytes/sec
└──────────────────────────────────────── device MAJ:MIN

$ cat io.stat
8:16 rbytes=1459200 wbytes=314773504 rios=192 wios=353 dbytes=0 dios=0

$ cat io.weight
default 100
8:16 300              # per-device override, range [1, 10000]
```

There is also `io.cost.qos`/`io.cost.model` (the **iocost** controller, which builds a cost model of the device and enforces latency targets rather than raw bandwidth) and `io.latency`. Kubernetes has **no first-class pod field for block I/O limits**, but the plumbing is here and systemd exposes it as `IOReadBandwidthMax=` etc. — [lesson 10](10-systemd-cgroups-and-block-io.md). The io controller only accounts writeback correctly when the memory controller is co-enabled on the same cgroup — the exact cross-controller dependency that motivated the unified hierarchy in §1.

**`cpuset`** — *which* CPUs and *which* NUMA memory nodes, as opposed to *how much*.

| File | Meaning |
|---|---|
| `cpuset.cpus` | Requested CPU list, e.g. `0-4,6,8-10`. Empty = inherit from nearest ancestor. |
| `cpuset.cpus.effective` | What the parent actually granted (ro). Affected by CPU hotplug; `cpuset.cpus` is not. |
| `cpuset.mems` / `.effective` | NUMA memory nodes, same semantics. |
| `cpuset.cpus.partition` | `member` / `root` / `isolated` — creates a scheduling-domain partition. `isolated` gives the runtime equivalent of `isolcpus=` from [lesson 01](01-processes-and-scheduling.md). |

The controller is strictly hierarchical: a cgroup can never use a CPU or memory node its parent does not have. On a GPU node this is the lever that matters most for tail latency — a Guaranteed pod pinned to cores on the wrong NUMA node relative to its GPU pays a cross-socket cost on every host-to-device copy and every NIC interrupt.

### 7. Where the kubelet puts everything

This is the section to memorise. With the **systemd cgroup driver**, the kubelet builds this tree:

```
/sys/fs/cgroup/                                    ← cgroup2fs root
│
├── system.slice/                                  ← sshd, containerd, kubelet
│   ├── containerd.service/                          (charged to --kube-reserved
│   └── kubelet.service/                              or --system-reserved)
│
└── kubepods.slice/                                ← NODE ALLOCATABLE boundary
    │                                                 = capacity − kube-reserved
    │                                                   − system-reserved − eviction
    │   cpu.weight   ← from allocatable CPU
    │   memory.max   ← = node allocatable memory (enforceNodeAllocatable: ["pods"])
    │   cpu.pressure, memory.pressure, io.pressure
    │
    ├── kubepods-pod3f2a_9c1d_4e77_b0a2_1f5c8d3e6b40.slice/     ◀── GUARANTEED
    │   │      ▲                                                    pods sit DIRECTLY
    │   │      └── pod UID, with every '-' rewritten as '_'          under kubepods
    │   │   cpu.max     = "8000000 100000"    (limits.cpu: 8)
    │   │   cpu.weight  = 313                 (kubelet formula, requests.cpu: 8)
    │   │   memory.max  = 68719476736         (limits.memory: 64Gi)
    │   │   cpuset.cpus = "8-15"              (CPU Manager static policy)
    │   │   cpuset.mems = "1"                 (Topology Manager, GPU-local node)
    │   │   pids.max    = 4096                (podPidsLimit)
    │   │
    │   ├── cri-containerd-9c1d…ff66.scope/          ← the trainer container
    │   │      cpu.max    = "8000000 100000"
    │   │      cpu.weight = 532               ◀── runc formula. NOT 313.
    │   │      memory.max = 68719476736
    │   │      cpu.stat, memory.events, *.pressure   ← WHAT YOU READ
    │   │
    │   └── cri-containerd-aa11…dd44.scope/          ← the pause container
    │
    ├── kubepods-burstable.slice/                               ◀── BURSTABLE tier
    │   │   cpu.weight = 938   (from Σ burstable pod CPU requests, recomputed
    │   │                       by the kubelet whenever the pod set changes)
    │   │
    │   └── kubepods-burstable-pod7b4e_2a91_….slice/
    │       │   cpu.max    = "500000 100000"   (limits.cpu: 5)
    │       │   cpu.weight = 39                (requests.cpu: 1)
    │       │   memory.max = 8589934592        (limits.memory: 8Gi)
    │       │
    │       └── cri-containerd-….scope/
    │              cpu.weight = 100            ◀── again, different formula
    │
    └── kubepods-besteffort.slice/                              ◀── BESTEFFORT tier
        │   cpu.weight = 1        (CPUShares pinned to MinShares = 2)
        │
        └── kubepods-besteffort-pod….slice/
            │   cpu.max    = "max 100000"      (no limit → uncapped)
            │   cpu.weight = 1
            │   memory.max = max               (no limit)
            │
            └── cri-containerd-….scope/
```

Three naming facts you need to navigate this by hand:

1. **systemd slice names are cumulative and dash-delimited.** `kubepods-burstable-pod<uid>.slice` *expands* to the path `kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<uid>.slice`. The name encodes its full ancestry; the directory nesting mirrors it.
2. **Pod UIDs have their dashes rewritten to underscores.** The kubelet escapes `-` → `_` before building the slice name, because `-` is systemd's path separator. A pod UID of `3f2a9c1d-4e77-b0a2-1f5c-8d3e6b40aa11` becomes `pod3f2a9c1d_4e77_b0a2_1f5c_8d3e6b40aa11.slice`. **This is why naive `find`s for a pod UID come back empty.**
3. **The container leaf is a scope, and its prefix names the runtime.** `cri-containerd-<full-container-id>.scope` for containerd; `crio-<id>.scope` for CRI-O; `docker-<id>.scope` for Docker. With the older **cgroupfs** driver the same shape appears as plain nested directories: `kubepods/burstable/pod<uid>/<containerid>/`, with no escaping and no `.slice`/`.scope` suffixes.

**Guaranteed pods have no QoS sub-slice.** They sit directly under `kubepods.slice`, because in the kubelet's QoS manager the "Guaranteed" parent *is* the root container. That is not cosmetic: it means a Guaranteed pod's CPU weight competes directly against the two QoS tier slices rather than being diluted inside one.

**Never guess the path — derive it.** For any running container:

```
$ PID=$(crictl inspect $CID | jq -r '.info.pid')
$ cat /proc/$PID/cgroup
0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod7b4e_2a91_4c03_8e15_6d9f0b2a7c34.slice/cri-containerd-4f8a2b….scope
│  └── v2 is always ONE line, always starts with "0::"
└───── hierarchy ID 0 = the unified hierarchy
$ CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)
```

(If you run that *inside* the container you will see `0::/` instead — that is the cgroup **namespace** from [lesson 02](02-namespaces.md) hiding the path, not a different cgroup.)

### 8. The mapping: every YAML field to its kernel file

This is the highest-value content in the module. Get it exact.

```
  pod spec                kubelet                     kernel file
  ════════                ═══════                     ═══════════

  requests.cpu: 500m
       │
       ├─▶ kube-scheduler: bin-packing reservation. Never re-read after
       │   placement. Does NOT cap anything at runtime.
       │
       └─▶ MilliCPUToShares(500) = 500 × 1024 / 1000 = 512 shares
                │
                ├─▶ POD + QoS SLICES (kubelet writes these itself)
                │      getCPUWeight(512) = 1 + (512−2)×9999/262142 = 20
                │      ───────────────────────────────────▶  cpu.weight = 20
                │
                └─▶ CONTAINER SCOPE (via CRI → runc/crun)
                       ConvertCPUSharesToCgroupV2Value(512):
                         l = log₂(512) = 9
                         exponent = (l² + 125l)/612 − 7/34
                                  = (81 + 1125)/612 − 0.20588 = 1.76471
                         weight = ⌈10^1.76471⌉ = ⌈58.16⌉ = 59
                       ───────────────────────────────────▶  cpu.weight = 59

  limits.cpu: 500m
       └─▶ MilliCPUToQuota(500, 100000) = 500 × 100000 / 1000 = 50000
           (floor of 1000 µs; period from --cpu-cfs-quota-period, default 100 ms)
           ───────────────────────────────────────────▶  cpu.max = "50000 100000"

  requests.memory: 4Gi
       ├─▶ scheduler reservation
       └─▶ MemoryQoS only:  Guaranteed → memory.min = 4294967296
                            Burstable  → memory.low = 4294967296

  limits.memory: 8Gi
       └─▶ ─────────────────────────────────────────▶  memory.max = 8589934592

  (MemoryQoS, factor 0.9)
       └─▶ ⌊(4Gi + 0.9×(8Gi − 4Gi))/4096⌋ × 4096
           ─────────────────────────────────────────▶  memory.high = 8160436224

  QoS class (derived)
       └─▶ ─────────────────────────────────────────▶  TREE POSITION

  kubelet --pod-pids-limit
       └─▶ ─────────────────────────────────────────▶  pids.max  (pod slice)

  CPU Manager static + Topology Manager
       └─▶ ─────────────────────────────────────────▶  cpuset.cpus / cpuset.mems
```

**The two-conversion fact deserves its own statement, because it is where people get caught.** The kubelet and the container runtime convert `shares` to `cpu.weight` with *different formulas*, so the same request produces different weights at different levels of the same tree:

| `requests.cpu` | shares | **pod / QoS slice**<br>`1 + (s−2)·9999/262142` | **container scope**<br>`⌈10^((l²+125l)/612 − 7/34)⌉` |
|---|---|---|---|
| *(none / BestEffort)* | 2 | **1** | **1** |
| `100m` | 102 | 4 | 17 |
| `250m` | 256 | 10 | 35 |
| `500m` | 512 | **20** | **59** |
| `1` | 1024 | **39** | **100** |
| `2` | 2048 | 79 | 174 |
| `4` | 4096 | 157 | 303 |
| `8` | 8192 | 313 | 532 |

Both are legitimate. The kubelet's is a straight linear rescale of the v1 range `[2, 262144]` onto the v2 range `[1, 10000]`. runc's is a quadratic in `log₂(shares)` chosen so that the three anchor points map exactly: `2 → 1`, `1024 → 100` (the defaults line up), `262144 → 10000`. Since ratios between siblings are all that matter, and siblings at the same level always use the same formula, neither choice is wrong — but if you read `cpu.weight` at the container scope and expect the kubelet's number, you will conclude your cluster is misconfigured when it is not.

**QoS class → placement and values:**

| | **Guaranteed** | **Burstable** | **BestEffort** |
|---|---|---|---|
| Condition | every container has `requests == limits` for **both** CPU and memory | at least one request or limit set, not full parity | no requests or limits anywhere |
| Slice path | directly under `kubepods.slice` | `kubepods-burstable.slice/` | `kubepods-besteffort.slice/` |
| `cpu.weight` | from `requests.cpu` | from `requests.cpu` | **1** (shares pinned to 2) |
| `cpu.max` | from `limits.cpu` | from `limits.cpu` **if set**, else `max` | `max` |
| `memory.max` | from `limits.memory` | from `limits.memory` **if set**, else `max` | `max` |
| MemoryQoS | `memory.min` = request; **no** `memory.high` | `memory.low` = request; `memory.high` per formula | `memory.high` from node allocatable |
| `cpuset.cpus` | exclusive cores if CPU Manager `static` **and** integer CPU request | shared pool | shared pool |
| `oom_score_adj` | **−997** | `1000 − 1000×memReq/memCapacity`, clamped to `[3, 999]` | **1000** |
| Eviction order | last | middle | **first** |
| Swap (`LimitedSwap`) | none (request == limit ⟹ proportion 0) | proportional share | none |

**Why the tree shape *is* the priority.** Because BestEffort and Burstable live in their own slices, an entire QoS tier can be reclaimed as a unit, and CPU share flows down the slices: under contention a Guaranteed pod keeps its weight while the whole BestEffort tier collapses to weight 1 against a `kubepods.slice` weight in the thousands. The `oom_score_adj` values reinforce the same ordering per-process. Guaranteed's `−997` is the kubelet's own constant (`guaranteedOOMScoreAdj`), deliberately just above the kubelet's and kube-proxy's own `−999` so that system components outlive workloads. [Lesson 05](05-memory-and-oom.md) covers `oom_badness()` scoring in full.

**One request, two jobs — keep them apart.** At **schedule time**, `requests.cpu` is a bin-packing reservation: the scheduler places the pod only on a node whose unrequested allocatable CPU covers it, and never re-reads it afterwards. At **run time**, the same number becomes `cpu.weight`, which does nothing until the node is contended and then divides spare CPU proportionally. So a request is simultaneously a *scheduling promise* and a *runtime priority* — and it is **not** a runtime cap. Only `limits.cpu → cpu.max` caps. "I requested 2 cores, why is it only getting 0.5?" is almost always this confusion: the node is contended and weight is doing exactly what weight does.

### 9. Node allocatable — where the cost boundary is drawn

`kubepods.slice` is not the machine. It is **node allocatable**:

```
  allocatable = capacity − kube-reserved − system-reserved − eviction-hard threshold

  Worked, for a 64-core / 256 GiB GPU node:
    capacity          64000m            274877906944 B  (256 GiB)
    − kube-reserved    2000m           −  8589934592 B  (8 GiB)
    − system-reserved  1000m           −  4294967296 B  (4 GiB)
    − eviction-hard        —           −   104857600 B  (memory.available<100Mi)
    ───────────────────────────────────────────────────
    allocatable       61000m            261888147456 B  (243.90 GiB)

  → kubepods.slice/memory.max  = 261888147456
  → kubepods.slice/cpu.weight  = 2383
       shares(61000m) = 61000 × 1024 / 1000 = 62464
       1 + (62464 − 2) × 9999 / 262142 = 2383
```

The kubelet writes `memory.max` on `kubepods.slice` when `enforceNodeAllocatable` includes `pods` — which is the **default** (`["pods"]`). So there is a genuine kernel-enforced ceiling on the sum of all pods, separate from every pod's own limit. Adding `kube-reserved` and/or `system-reserved` to `enforceNodeAllocatable` additionally caps `system.slice` (or whichever cgroups you name), which is how you stop a runaway containerd from starving pods.

**This is why cgroup accounting is cost accounting.** Walk `kubepods.slice → QoS slice → pod slice → container scope`, read `cpu.stat usage_usec` and `memory.current` at each level, and you have a complete, kernel-authoritative attribution of the node's spend with no gaps and no double-counting — because the hierarchy guarantees children sum into parents. `system.slice` is the overhead line. Nothing escapes.

### 10. What the container sees versus what the kernel enforces

A process inside a container reads `nproc`, `/proc/cpuinfo`, and `/proc/meminfo` and sees the **host's** hardware. No namespace virtualizes them: the cgroup namespace from [lesson 02](02-namespaces.md) hides the cgroup *path*, not the hardware numbers, and there is no "hardware namespace."

```
  Inside a container with cpu.max = "2000000 100000", memory.max = 8Gi,
  on a 64-core / 256 GiB node:

    $ nproc                                    64      ← host truth
    $ grep -c ^processor /proc/cpuinfo         64      ← host truth
    $ grep MemTotal /proc/meminfo         256 GiB      ← host truth
    $ free -h                        total 251Gi       ← host truth
    $ cat /sys/fs/cgroup/cpu.max        2000000 100000 ← ACTUAL LIMIT
    $ cat /sys/fs/cgroup/memory.max        8589934592  ← ACTUAL LIMIT
```

Every "why is my JVM heap sized for the whole node" and "why does my thread pool have 64 workers" question is this gap. The correct fix is always the same shape: make the runtime read `/sys/fs/cgroup/cpu.max` and `memory.max` (or use a cgroup-aware runtime version). Tools like `lxcfs` paper over it by bind-mounting a synthesised `/proc/meminfo`, which helps naive tools but is not the mechanism.

### 11. MemoryQoS and swap

**MemoryQoS** (feature gate `MemoryQoS`, alpha since v1.27, graduating to beta in v1.37) is what makes the kubelet write `memory.min`/`memory.low`/`memory.high` at all. **Without it, a Kubernetes memory limit sets `memory.max` and nothing else** — there is no soft throttle, only the cliff. That is the default state of the overwhelming majority of clusters.

With it enabled, the kubelet writes, per container:

```
  memory.min  = requests.memory     (Guaranteed pods — hard protection)
  memory.low  = requests.memory     (Burstable pods  — soft protection)

  memory.high = ⌊(requests.memory
                  + factor × ((limits.memory or node allocatable) − requests.memory))
                 / pageSize⌋ × pageSize

     where factor = kubelet's memoryThrottlingFactor, default 0.9
```

The formula was revised in v1.27 precisely because the original (`factor × limits.memory`) throttled early workloads that legitimately run near their limit — an 0.8 factor started throttling a JVM at 80% of its limit, which for a heap sized at 85% meant permanent throttling. The current form interpolates between the request and the limit, so `memory.high` is always above the request. Worked:

```
  requests.memory: 4Gi = 4294967296 B      limits.memory: 8Gi = 8589934592 B
  factor 0.9, page size 4096

  4294967296 + 0.9 × (8589934592 − 4294967296) = 4294967296 + 3865470566.4
                                                = 8160437862.4
  ⌊8160437862.4 / 4096⌋ × 4096                 = 8160436224 B  (7.60 GiB)

  → memory.high = 8160436224      throttle starts here
  → memory.max  = 8589934592      OOM kill here
    a 409,498,368 B (390 MiB) reclaim runway between throttle and kill
```

Guaranteed pods (request == limit) get **no** `memory.high` at all — there is no room to interpolate. When MemoryQoS is off, the kubelet explicitly writes `memory.high = max` so that a stale value from a previously-enabled state cannot linger.

**Swap** (`NodeSwap` gate; alpha v1.22, beta v1.30, **stable v1.34**) uses `memory.swap.max`. Default `swapBehavior` is `NoSwap`. Under `LimitedSwap`, only **Burstable** pods get swap, and the per-container allocation is proportional:

```
  containerSwapLimit = (containerMemoryRequest / totalNodePhysicalMemory)
                       × totalPodsSwapAvailable

  Containers with request == limit get proportion 0, i.e. no swap.

  Worked (from the KEP): node 40 GB RAM, 40 GB swap, 2 GB system-reserved
    Container A, request 20 GB → 20/40 = 0.50 → 0.50 × 38 GB = 19 GB
    Container B, request 10 GB → 10/40 = 0.25 → 0.25 × 38 GB = 9.5 GB
```

For memory-bound cost efficiency this is a real lever: a Burstable pod that would have been OOM-killed can instead page out cold anonymous memory. For a GPU training job it is usually the wrong tool — swapping a pinned host buffer used for host-to-device copies converts a memory problem into a latency catastrophe.

## Perspectives

**Kernel-mechanism view.** The unified hierarchy is the spine. One tree; the no-internal-process rule keeping processes only on leaves so controllers never have to arbitrate between a cgroup's children and its own bare tasks; top-down enablement so a controller touches only subtrees its parent explicitly pushed it into; delegation containment so a subtree can be handed to an unprivileged manager without letting anything escape. Every other perspective below is the same tree read from a different vantage point.

**Operator/SRE view.** You do not start at `cpu.max` — you start at a page: latency spikes, a restart, a neighbour complaining. The discipline is a fixed mapping from symptom to the file that *proves* the cause, and it replaces the argument about what the dashboard means. p99 spikes with a green CPU graph → `cpu.stat nr_throttled` and `cpu.pressure`. Exit 137 → `memory.events oom_kill` in that container's own cgroup, and if it reads 0 the kill came from outside. "Resource temporarily unavailable" → `pids.current` versus `pids.max`, not memory. An SRE who has internalised that table reads evidence instead of interviewing suspects.

**GPU-fleet-specific view.** On a NUMA GPU box, cgroups are not only "how much CPU/memory" — they are "*which* physical cores, next to *which* GPU." `cpuset.cpus`/`cpuset.mems`, driven by the CPU Manager `static` policy and the Topology Manager, is how a Guaranteed pod with an integer CPU request gets exclusive cores on the NUMA node local to its allocated GPU, so that host-to-device copies and NIC interrupts do not cross a socket. That is invisible if you have only run general-purpose fleets, and it is exactly the layer GPU-operator interviews probe. Note also what cgroups do **not** do here: GPU count is accounted by the device-plugin framework in userspace, and GPU access is a device-node bind mount plus a BPF cgroup-device program ([lesson 02](02-namespaces.md) §8) — the `misc` controller registers only SEV/TDX key slots and has nothing to do with GPUs.

**Economics view.** `kubepods.slice` is node allocatable, and every pod's usage is a leaf underneath it. Walking `kubepods.slice → QoS slice → pod slice → container scope` gives a complete attribution of a node's spend, guaranteed gap-free because the hierarchy makes children sum into parents. On a fleet where one GPU node costs tens of dollars an hour, that walk *is* the chargeback report. It also tells you the uncomfortable number: `kubepods.slice/cpu.stat usage_usec` versus the node's total CPU-seconds is your actual utilization, and the difference between that and the sum of `requests` is the money you are paying for reservations nobody is using.

## Real-world use cases

- **Indeed Engineering — "Unthrottled: How a Valid Fix Becomes a Regression."** The canonical production account of throttling despite low average utilization. Kernel commit `512ac999` (4.18), intended to fix an unrelated bandwidth-timer clock-drift bug, made per-CPU quota slices genuinely expire at period end. Combined with the 5 ms slice granularity, this meant — in Indeed's own framing — *an application with 100 ms of quota (1 CPU) and 5 ms slices can only occupy 20 cores before the global pool is empty; threads on the remaining 68 cores of an 88-core machine are throttled waiting for slack to be returned.* Worst-case request latency fell from over 2 seconds to about 30 ms once the fix landed. That fix, `de53fd7aedb1` — *"sched/fair: Fix low cpu usage with high throttling by removing expiration of cpu-local slices"* — removed slice expiry entirely and shipped in **Linux 5.4**. *What it shows:* this is the production version of the "throttled at low CPU" interview question, and the mechanism is the per-CPU silo model, not vague "burstiness." Read it end to end.
- **Omio Engineering — "CPU limits and aggressive throttling in Kubernetes."** An independent rediscovery of the same mechanism at a different company: random stalls and readiness-probe failures on multi-threaded services, traced to CFS quota throttling, with the "quota consumed in a fraction of the period" arithmetic worked out concretely and an honest internal debate about mitigations — raise the limits, remove limits entirely, or patch the kernel. *What it shows:* the failure is structural to quota/period bandwidth control, not a one-off bug. It also shows that "remove CPU limits" is a real position serious teams take, with the real cost being loss of isolation against noisy neighbours.
- **Go 1.25's cgroup-aware `GOMAXPROCS`.** After a decade in which `go.uber.org/automaxprocs` was effectively mandatory in every Kubernetes Go service, the Go runtime itself now reads the cgroup CPU limit and derives `GOMAXPROCS` from it by default (Go 1.25, August 2025). *What it shows:* the container-sees-host-hardware gap (§10) was severe and widespread enough that a language runtime absorbed the fix. The same gap is still open for the JVM's older container support, for OpenMP, and for anything sized off `nproc` — which is most numerical and data-loading code on a GPU node.

## Worked example

Take a container with a half-CPU cap and a 256 MiB memory cap, read its cgroup by hand, force it to throttle, and prove it. *(Transcripts are representative — reconstructions with exact arithmetic, not a captured session.)*

**Step 1 — start it and locate the leaf cgroup without guessing.**

```bash
docker run -d --name cg-demo --cpus=0.5 --memory=256m \
  alpine sh -c 'apk add -q stress-ng && stress-ng --cpu 4 --timeout 600s'

PID=$(docker inspect -f '{{.State.Pid}}' cg-demo)
CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)
echo "$CG"
# /sys/fs/cgroup/system.slice/docker-9c1d4e77b0a21f5c8d3e6b40aa11bb22.scope
```

**Step 2 — read the enforcement files and reconcile each with the flag that produced it.**

```bash
$ cat $CG/cpu.max
50000 100000
#      └── period 100 ms (default)
#  └────── quota 50 ms  → 50000/100000 = 0.5 CPU   ✔ matches --cpus=0.5

$ cat $CG/cpu.weight
100
#  └── unchanged. --cpus sets only the CAP. Weight is untouched, which is
#      the whole share-vs-cap distinction in one line.

$ cat $CG/memory.max
268435456
#  └── 256 × 1024 × 1024 = 268435456   ✔ matches --memory=256m

$ cat $CG/memory.high
max
#  └── nothing set it. Docker doesn't; Kubernetes doesn't either unless
#      MemoryQoS is on. There is NO soft throttle here — only the cliff.

$ cat $CG/memory.current
5849088
#  └── ~5.58 MiB in use right now

$ cat $CG/pids.max
max
```

**Step 3 — predict the throttling before you measure it.**

```
  Given: quota 50 ms per 100 ms period; 4 CPU-burning threads; ≥4 cores free.

  Drain rate  = 4 threads × 1 core-ms per wall-ms = 4 core-ms/ms
  Time to burn 50 ms of quota = 50 / 4 = 12.5 ms
  Throttled for the remaining 100 − 12.5 = 87.5 ms of every period

  Predictions:
    duty cycle           = 12.5 / 100                     = 12.5%
    nr_throttled/nr_periods → ≈ 1.0 (throttled EVERY period)
    throttled_usec rate  = 87.5 ms × 4 threads per 100 ms
                         = 350 ms of aggregate stall per 100 ms wall
                         = 3.5 s of stall per second of wall time
    usage_usec rate      = 50 ms per 100 ms = 0.5 CPU-s per wall-s  ✔ the cap
```

**Step 4 — measure and compare.**

```bash
$ cat $CG/cpu.stat
usage_usec 41230000
user_usec 40010000
system_usec 1220000
nr_periods 612
nr_throttled 598          # 598/612 = 97.7% of periods throttled ✔ predicted ≈1.0
throttled_usec 55082140
nr_bursts 0
burst_usec 0

$ a=$(awk '/throttled_usec/{print $2}' $CG/cpu.stat); \
  b=$(awk '/usage_usec/{print $2}' $CG/cpu.stat);     \
  sleep 10;                                            \
  a2=$(awk '/throttled_usec/{print $2}' $CG/cpu.stat); \
  b2=$(awk '/usage_usec/{print $2}' $CG/cpu.stat);     \
  echo "stall/s = $(( (a2-a)/10000000 )).$(( ((a2-a)/1000000)%10 ))  cpu/s = 0.$(( (b2-b)/1000000 ))"
stall/s = 3.4  cpu/s = 0.5
#          └── ≈3.5 s of aggregate stall per second ✔    └── exactly the cap ✔
```

The prediction and the measurement agree. **That agreement is the deliverable** — it is what turns "I think it's throttling" into "it is throttling, and here is the arithmetic."

**Step 5 — cross-check with PSI, which measures the same event differently.**

```bash
$ cat $CG/cpu.pressure
some avg10=91.44 avg60=89.02 avg300=71.15 total=488213991
full avg10=87.10 avg60=85.33 avg300=68.02 total=461002418
```

`full avg10=87.1` — for 87% of the last 10 seconds, **every** task in this cgroup was stalled and nothing productive ran. Compare that number against the 87.5 ms/100 ms predicted in step 3. They are the same fact, measured by two independent kernel subsystems. ([Lesson 04](04-psi.md) explains why per-cgroup CPU pressure has a `full` line at all.)

**Step 6 — change the cap live and watch the counters respond.**

```bash
before=$(awk '/nr_throttled/{print $2}' $CG/cpu.stat)
echo "400000 100000" > $CG/cpu.max          # raise to 4.0 CPUs
sleep 10
after=$(awk '/nr_throttled/{print $2}' $CG/cpu.stat)
echo "throttled periods in 10s: $((after - before))"
# throttled periods in 10s: 0
```

Four threads, four CPUs of quota: demand now fits, and `nr_throttled` stops advancing entirely. At `100000 100000` (1.0 CPU) it would still climb, because four threads want four cores. **Reading `cpu.stat` across a `cpu.max` change is the definitive test of a throttling hypothesis**, and it is the same evidence Indeed and Omio built their postmortems on.

**Step 7 — the memory cliff, and its proof.**

```bash
docker exec cg-demo sh -c 'stress-ng --vm 1 --vm-bytes 400M --vm-keep --timeout 30s'
# stress-ng: FAIL: [42] stress-ng-vm: [43] terminated on signal 9

$ cat $CG/memory.events
low 0
high 0                    # ← zero, because memory.high was never set
max 4127                  # ← 4127 times usage was about to exceed memory.max
oom 3
oom_kill 1                # ← ONE process killed by THIS cgroup's OOM killer
oom_group_kill 0

$ docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' cg-demo
137 true
```

`oom_kill 1` in *this* cgroup is the attribution. Had it read `oom_kill 0` while the container still exited 137, the kill came from outside — the node-level OOM killer or a kubelet eviction — and the fix would be node capacity or eviction thresholds, not this container's limit. `high 0` confirms there was no soft-throttle stage at all: usage went straight from fine to killed, which is the default Kubernetes behaviour and the reason MemoryQoS exists.

**Step 8 — the same walk for a real pod, plus the node-allocatable context.**

```bash
CID=$(crictl ps --name trainer -q)
PID=$(crictl inspect $CID | jq -r '.info.pid')
CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$PID/cgroup)

# climb the tree, printing the enforcement at every level
D=$CG
while [ "$D" != "/sys/fs/cgroup" ]; do
  printf '%s\n  cpu.max=%s  cpu.weight=%s  memory.max=%s  memory.current=%s\n' \
    "${D#/sys/fs/cgroup}" "$(cat $D/cpu.max 2>/dev/null|tr '\n' ' ')" \
    "$(cat $D/cpu.weight 2>/dev/null)" "$(cat $D/memory.max 2>/dev/null)" \
    "$(cat $D/memory.current 2>/dev/null)"
  D=$(dirname $D)
done
```

```
/kubepods.slice/kubepods-pod3f2a_9c1d_4e77_b0a2_1f5c8d3e6b40.slice/cri-containerd-9c1d….scope
  cpu.max=8000000 100000  cpu.weight=532  memory.max=68719476736  memory.current=41203662848
/kubepods.slice/kubepods-pod3f2a_9c1d_4e77_b0a2_1f5c8d3e6b40.slice
  cpu.max=8000000 100000  cpu.weight=313  memory.max=68719476736  memory.current=41205760000
/kubepods.slice
  cpu.max=max 100000      cpu.weight=2383 memory.max=261888147456 memory.current=198374129664
```

Read the three lines as one story. The pod is **Guaranteed** (it sits directly under `kubepods.slice`, no QoS sub-slice) with `limits.cpu: 8` and `limits.memory: 64Gi`, currently using 38.4 GiB. `cpu.weight` is 313 at the pod slice and 532 at the container scope — **the two different conversions of the same 8-CPU request**, exactly as §8 predicts. `kubepods.slice` is uncapped on CPU but capped at 243.90 GiB of memory: node allocatable, kernel-enforced. All pods together are currently at 184.7 GiB of that.

## Practice

For one running container — a `docker run` you control, or a real pod's container — reconstruct its **full cgroup resource picture by hand**, map every value back to the spec that produced it, and prove each enforcement with a counter.

1. **Locate the leaf cgroup from first principles.** Get the host PID (`docker inspect -f '{{.State.Pid}}' <name>` or `crictl inspect <id> | jq -r '.info.pid'`), then `cut -d: -f3 /proc/<pid>/cgroup` and prefix `/sys/fs/cgroup`. Confirm you are on v2 with `stat -fc %T /sys/fs/cgroup`. **Do not construct the path by hand** — derive it, then read the path back and identify the QoS tier and the pod UID from the slice names (remembering the `-` → `_` escaping).

2. **Record the full picture.** `cat` and record: `cpu.max`, `cpu.weight`, `cpu.stat`, `memory.max`, `memory.high`, `memory.current`, `memory.events`, `pids.max`, `pids.current`, `cpuset.cpus`, `cpuset.mems`, and `cpu.pressure`. Do the same at the pod slice and at `kubepods.slice` (or `system.slice` for a plain Docker container).

3. **Predict, then induce, throttling.** Before you run anything: from `cpu.max` and your intended thread count, compute (a) how many wall-milliseconds it takes to drain the quota, (b) the throttled fraction of each period, and (c) the expected `throttled_usec` rate per wall-second. Then run `stress-ng --cpu N` with `N` greater than your quota in cores, sample `cpu.stat` twice ten seconds apart, and compare measurement to prediction. **A prediction that matches within ~20% is the acceptance bar.**

4. **Demonstrate share-vs-cap.** Create two sibling cgroups by hand, give them `cpu.weight` 100 and 300, put a spinner in each, and pin both to a single CPU with `cpuset.cpus`. Confirm from `cpu.stat usage_usec` deltas that they split 25/75. Then stop one and confirm the other takes 100% — proving weight is a floor under contention, never a ceiling.
   ```bash
   cd /sys/fs/cgroup
   mkdir -p demo && echo "+cpu +cpuset" > cgroup.subtree_control
   mkdir -p demo/a demo/b
   echo "+cpu" > demo/cgroup.subtree_control
   echo 0 > demo/cpuset.cpus            # both children share ONE cpu
   echo 100 > demo/a/cpu.weight; echo 300 > demo/b/cpu.weight
   # start a spinner in each, then move it in:
   #   bash -c 'while :; do :; done' & echo $! > demo/a/cgroup.procs
   ```
   Note what happens if you try to put a process directly in `demo` after enabling controllers in its `subtree_control` — that error message *is* the no-internal-process rule.

5. **Force and attribute an OOM.** Set a small `memory.max`, run a memory hog, and capture `memory.events` before and after. Record which counters moved (`max`, `oom`, `oom_kill`) and confirm the container's exit code is 137. Then reason about the counterfactual: what would `memory.events` look like if the *node* had killed it instead?

6. **Build the acceptance artifact — a mapping table for your one real container:**

| Spec value | cgroup file | Level | Observed value | Enforcement proven by |
|---|---|---|---|---|
| `limits.cpu: 500m` | `cpu.max` | container scope | `50000 100000` | `nr_throttled` 598/612 under load |
| `requests.cpu: 500m` | `cpu.weight` | pod slice | `20` | 25/75 split vs a weight-300 sibling |
| `requests.cpu: 500m` | `cpu.weight` | container scope | `59` | (different formula — note it) |
| `limits.memory: 256Mi` | `memory.max` | container scope | `268435456` | `memory.events oom_kill 1`, exit 137 |
| *(MemoryQoS off)* | `memory.high` | container scope | `max` | `memory.events high 0` |
| QoS class | tree position | — | `kubepods-burstable-pod…` | eviction tier |
| node allocatable | `memory.max` | `kubepods.slice` | `261888147456` | = capacity − reserved − eviction |

7. **Write one paragraph per row** mapping the observed value back to the spec field and to the enforcement you saw with your own eyes. *"The spec says X, the kernel file says Y, and under load I observed Z"* — said fluently, that sentence is what the interview is testing.

Feed this table straight into the **[Anatomy of a Container](../practice/anatomy-of-a-container/README.md)** deliverable; it is the resource-enforcement section of that write-up.

### Failure-signature quick reference

Keep this. It is the fast path from a symptom to the exact file that proves the cause.

| Symptom | Read this | Confirms |
|---|---|---|
| p99 latency spikes, CPU dashboard looks fine | `cpu.stat` `nr_throttled`/`throttled_usec`, `cpu.pressure` | Bandwidth throttling against `cpu.max` |
| Throttled *and* thread count > quota-in-cores × 20 | `cpu.max` vs `sched_cfs_bandwidth_slice_us` | Per-CPU slice hoarding — reduce parallelism |
| Container restarts, exit 137 | `memory.events` `oom_kill` **in its own cgroup** | cgroup OOM at its own `memory.max` |
| Exit 137 but `oom_kill 0` here | `dmesg`, kubelet eviction events, parent cgroups | Node-level OOM or eviction, not this limit |
| App slow, no throttling, memory near limit | `memory.pressure`, `memory.events high`, `memory.stat workingset_refault_*` | Reclaim thrash at `memory.high` or under global pressure |
| "resource temporarily unavailable", thread-create failures | `pids.current` vs `pids.max` | PID limit (`EAGAIN`), not memory |
| One pod starves neighbours under load | `cpu.weight` across sibling slices, `cpu.pressure` per slice | Proportional-share imbalance |
| Latency varies by which node the pod lands on | `cpuset.cpus`, `cpuset.mems`, GPU's NUMA node | Cross-NUMA placement |
| `nr_periods 0` and you expected throttling | `cpu.max` | It is `max` — nothing is capped, so nothing is counted |

## Common pitfalls

1. **Believing `requests.cpu` is a runtime cap.** It is a *scheduling reservation* (bin-packing, evaluated once at placement) and, separately, `cpu.weight` (a proportional share that only matters under contention). Only `limits.cpu → cpu.max` caps CPU time. *Mechanism:* weight feeds the same `load.weight` the scheduler uses for `nice`; it divides a contended CPU and does nothing on an idle one.
2. **Expecting one `cpu.weight` value for one request.** There are **two different shares→weight conversions on the same node**: the kubelet's linear `1 + (s−2)·9999/262142` for pod and QoS slices, and runc's quadratic `⌈10^((l²+125l)/612 − 7/34)⌉` for the container scope. `requests.cpu: 500m` reads as `20` at the pod slice and `59` at the container scope. Both are correct. Comparing across levels and concluding "misconfiguration" is the trap.
3. **Assuming a Kubernetes memory limit sets `memory.high`.** By default it sets **`memory.max` only** — the hard kill threshold, with no soft-throttle stage. `memory.high` is written solely by the **MemoryQoS** feature gate (alpha since v1.27). Verify with `memory.events`: `high 0` on a busy container means there is no throttle stage at all.
4. **Assuming low average CPU rules out throttling.** It is close to the opposite. Quota is spent at *thread-count* × real time, so a 32-thread app drains a 200 ms quota in 6.25 ms and sits frozen for 93.75 ms — 32% of its limit on average, 10× amplified p99. *Diagnostic:* `nr_throttled/nr_periods`, never the CPU graph.
5. **Fixing throttling by adding threads.** Because quota moves to per-CPU silos in 5 ms slices, at most `quota_us / 5000` CPUs can hold a slice at once — 20 CPUs for a 1-CPU limit. Beyond that, extra threads are throttled from the start of the period. Widening the pool makes it *worse*. *Fixes:* raise the limit, reduce parallelism (`GOMAXPROCS`, `OMP_NUM_THREADS`, `num_workers`), or lower `sched_cfs_bandwidth_slice_us` node-wide (at the cost of global-lock traffic).
6. **Mismatching the kubelet's and the runtime's cgroup driver** (`cgroupfs` vs `systemd`). Two managers create two parallel trees for the same processes; limits get written to one and read from the other, and you get "pods OOM-killed for no reason" that no amount of dashboard-reading explains. Match them on both sides. *Note:* the KubeletConfiguration API default is `cgroupfs`, while kubeadm sets `systemd` — so "the default" depends on how the node was built. [Lesson 10](10-systemd-cgroups-and-block-io.md) covers this in full.
7. **Constructing cgroup paths by hand and finding nothing.** Pod UIDs have `-` rewritten to `_` in systemd slice names, slice names are cumulative (`kubepods-burstable-pod<uid>.slice` lives at `kubepods.slice/kubepods-burstable.slice/…`), and Guaranteed pods have **no** QoS sub-slice. Always derive the path from `/proc/<pid>/cgroup`.
8. **Reading `/proc/self/cgroup` inside a container and concluding the cgroup is `/`.** That is the **cgroup namespace** ([lesson 02](02-namespaces.md)) hiding the path. The limits are unchanged; you are looking at the same cgroup through a different lens. Read it from the host, or from a `hostPID`-enabled debug pod.
9. **Believing the `misc` controller accounts GPUs.** It registers AMD SEV/SEV-ES ASIDs and Intel TDX HKIDs. GPU count is accounted by the Kubernetes device-plugin framework in userspace; GPU *access* is a device-node bind mount plus a BPF cgroup-device program. The `dmem` controller does account device memory, but is driven by DRM drivers.
10. **Debugging blind from dashboards that only show averages.** `cpu.stat`, `memory.events`, and `pids.events` are the difference between a hypothesis and a proof. If your metrics pipeline does not scrape them per container, you are diagnosing throttling and OOMs by vibes.

## Self-check

**Q1. A pod sets `limits.cpu: 500m`. What exact value appears in `cpu.max`, and how does bandwidth control throttle it while average CPU stays low?**
**Answer:** `cpu.max` becomes the two-value string `50000 100000` — quota 50,000 µs per 100,000 µs (100 ms) period, i.e. 0.5 CPU. The kubelet computes it as `MilliCPUToQuota(500, 100000) = 500 × 100000 / 1000 = 50000`, with a floor of 1,000 µs and the period taken from `--cpu-cfs-quota-period` (default 100 ms). Quota is CPU-time **summed across all cores**, while the period is wall time, so an *n*-thread application drains it at *n* core-ms per wall-ms. Four threads drain a 50 ms quota in 12.5 ms and are then dequeued for the remaining 87.5 ms of every period. Averaged over the window that reads as 0.5 CPU used out of a 0.5 CPU limit — or, with intermittent load, well *under* the limit — while p99 latency is amplified by the throttled tail. The second, compounding mechanism is per-CPU slice hoarding: quota moves from the cgroup's global pool to per-CPU silos in `sched_cfs_bandwidth_slice_us` (default 5,000 µs) units, so at most `quota/5000` CPUs can hold a slice at once — 10 CPUs at 500m, 20 at 1 CPU — and threads on any further CPU are throttled from the start of the period regardless of aggregate usage. The proof is `cpu.stat`: `nr_throttled/nr_periods` near 1.0 and a large `throttled_usec` (which is aggregated across threads, so it can exceed one second per wall-second).

**Q2. What is the difference between `memory.high` and `memory.max`, and which one does a Kubernetes memory *limit* set?**
**Answer:** `memory.max` is the hard limit: when usage would exceed it and reclaim cannot recover, the **cgroup OOM killer** kills a process inside that cgroup (exit code 137, `memory.events oom_kill` increments). `memory.high` is a throttle: crossing it puts the allocating tasks into **synchronous direct reclaim in their own context** — they scan LRU lists, write back dirty pages, and swap anon pages if swap exists, and the allocation does not return until reclaim succeeds or usage drops. It **never** invokes the OOM killer and the limit may be breached under extreme pressure. A Kubernetes memory limit sets **`memory.max` only**. `memory.high` is written solely by the **MemoryQoS** feature (gate `MemoryQoS`, alpha since v1.27), using `memory.high = ⌊(requests.memory + factor × ((limits.memory or node allocatable) − requests.memory)) / pageSize⌋ × pageSize` with a default `memoryThrottlingFactor` of 0.9 — and Guaranteed pods (request == limit) get none, since there is nothing to interpolate. Without the gate there is no soft-throttle stage at all, only the cliff, which you can confirm from `memory.events` showing `high 0` on a container that is nonetheless being killed.

**Q3. How does Guaranteed QoS shape the cgroup tree differently from Burstable, and why does that matter for eviction and OOM order?**
**Answer:** A Guaranteed pod — every container with `requests == limits` for **both** CPU and memory — gets its cgroup **directly under `kubepods.slice`** as `kubepods-pod<uid>.slice`, with no QoS sub-slice, because in the kubelet's QoS manager the Guaranteed parent *is* the root container. Burstable pods live under `kubepods-burstable.slice`, BestEffort under `kubepods-besteffort.slice`. That placement is load-bearing three ways. (1) **CPU share flows down the slices**: the BestEffort tier slice has its shares pinned to `MinShares = 2`, giving `cpu.weight = 1`, while `kubepods.slice` carries a weight in the thousands — so under contention the whole BestEffort tier collapses as a unit while a Guaranteed pod's weight competes at the top level undiluted. (2) **A whole tier can be reclaimed as a unit** — the kubelet evicts BestEffort first, Burstable next, Guaranteed last, and the slices make that a structural property rather than a per-pod hint. (3) **`oom_score_adj` reinforces the same ordering per process**: Guaranteed containers get `−997`, BestEffort `1000`, and Burstable `1000 − 1000×memRequest/memCapacity` clamped to `[3, 999]` — so a Burstable container with a large request relative to node memory is protected almost like a Guaranteed one, and one that requested almost nothing is nearly as exposed as BestEffort.

**Q4. On a NUMA GPU node, how does Kubernetes give a Guaranteed pod exclusive integer cores co-located with its GPU, and what carries it out?**
**Answer:** The CPU Manager's `static` policy, coordinated with the Topology Manager, allocates exclusive whole cores from the NUMA node local to the pod's assigned GPU (as reported by the device plugin's topology hints), and writes them into the pod's cgroup as **`cpuset.cpus`** (the CPU set) and **`cpuset.mems`** (the NUMA memory nodes). The kernel then enforces both directly: no other cgroup's threads can be scheduled onto those CPUs, and allocations from that cgroup are steered to the specified memory nodes. Eligibility is narrow: the pod must be **Guaranteed** *and* request whole-integer CPUs — `1500m` gets you the shared pool. `cpuset.cpus.effective` is what was actually granted (bounded by the parent and affected by CPU hotplug), and `cpuset.cpus.partition` set to `isolated` gives the runtime, reversible equivalent of the boot-time `isolcpus=` from [lesson 01](01-processes-and-scheduling.md). The payoff is avoiding a cross-socket penalty on every host-to-device copy and every NIC interrupt in the training path.

**Q5. You are handed a PID on a Kubernetes node. Walk from it to a complete resource picture, and say what each level tells you.**
**Answer:** `cat /proc/<pid>/cgroup` returns exactly one line on v2, beginning `0::` (hierarchy 0 = the unified hierarchy); take field 3 and prefix `/sys/fs/cgroup` to get the container's leaf cgroup. Do **not** construct the path by hand — pod UIDs have `-` rewritten to `_`, slice names are cumulative, and Guaranteed pods have no QoS sub-slice. Then read four levels. **Container scope** (`cri-containerd-<id>.scope`): `cpu.max`, `cpu.weight` (runc's conversion), `cpu.stat`, `memory.max`, `memory.current`, `memory.events`, and `*.pressure` — this is where you diagnose. **Pod slice**: the pod's aggregate `cpu.max`/`memory.max`, `cpu.weight` (kubelet's conversion), `pids.max` from `podPidsLimit`, and `cpuset.cpus`/`cpuset.mems` if the static CPU Manager policy pinned it. **QoS slice** (absent for Guaranteed): the tier's share, and the eviction unit. **`kubepods.slice`**: node allocatable — `memory.max` equal to capacity minus kube-reserved, system-reserved, and the eviction threshold, and `cpu.weight` from allocatable CPU. The path itself tells you the QoS class and the pod UID; `memory.current` at each level tells you what that scope is consuming; and the difference between `kubepods.slice/memory.current` and the machine's total is the system overhead in `system.slice`. Run the same walk inside the container and you will see `0::/` instead — that is the cgroup namespace hiding the path, not a different cgroup.

**Q6. Why did the Indeed incident produce throttling at low CPU, and what changed in the kernel?**
**Answer:** Bandwidth control keeps a per-cgroup global quota pool refilled each period and hands budget to **per-CPU silos** in `sched_cfs_bandwidth_slice_us` (5 ms) units, so that threads charge a local counter instead of a globally contended one. Two consequences follow. First, at most `quota/5000` CPUs can hold a slice simultaneously — with `limits.cpu: 1` that is 20 CPUs, so on an 88-core machine threads on the other 68 cores are throttled from the start of a period even though the cgroup has used almost nothing. Second, before Linux 5.4, commit `512ac999` (4.18) — itself a legitimate fix for a bandwidth-timer clock-drift bug — made unused per-CPU slices **expire** at period end rather than returning to the global pool, so up to 87 ms of a 100 ms period could be stranded in idle silos. Indeed measured worst-case request latency above 2 seconds, falling to about 30 ms after the fix. The fix, commit `de53fd7aedb1` (*"sched/fair: Fix low cpu usage with high throttling by removing expiration of cpu-local slices"*), removed slice expiry entirely and shipped in **5.4**. The behaviour documented today is that an assigned slice does not expire and all but 1 ms of it (`min_cfs_rq_runtime`) returns to the global pool when a CPU's threads all become unrunnable. The slice-hoarding half of the problem was not removed and still applies on current kernels — which is why reducing thread parallelism remains a real mitigation.

## Connections & what's next

This lesson is the anchor. [Lesson 02](02-namespaces.md)'s namespaces (visibility) plus this lesson's cgroups (consumption) plus a rootfs is the complete container thesis, and everything after this reads the same tree from a different angle.

**[04 — PSI](04-psi.md)** takes the `cpu.pressure`/`memory.pressure`/`io.pressure` files that appeared at every level above and makes them the whole lesson — including why per-cgroup CPU pressure has a `full` line when the system-wide file does not, and why `full` on a throttled cgroup matches the arithmetic you did in the worked example. **[05 — memory & the OOM killer](05-memory-and-oom.md)** picks up `memory.events`, `memory.stat`'s `workingset_refault_*`, and `oom_score_adj`, and walks the full reclaim-to-kill path plus the `dmesg` OOM report. **[06 — hugepages/THP/NUMA](06-hugepages-thp-numa.md)** goes deep on the `cpuset.mems` NUMA story sketched here, plus the `hugetlb` controller. **[10 — systemd as cgroup manager](10-systemd-cgroups-and-block-io.md)** returns to delegation, the driver-mismatch pitfall, and the io controller and iocost in full.

Carry forward the map — **spec field → conversion → cgroup file → observed counter**. It is the lens for the rest of the module and the thing an interviewer is actually testing when they ask about throttling.

Next: **[04 · PSI — saturation the right way](04-psi.md)**, which takes the pressure files this lesson introduced at every level of the tree and turns them into the primary saturation instrument for the fleet.

## References & further reading

**Primary sources**

- **Control Group v2 — kernel admin guide** (`Documentation/admin-guide/cgroup-v2.rst`) — https://docs.kernel.org/admin-guide/cgroup-v2.html — the authoritative reference for every interface file quoted above: `cpu.weight`/`cpu.max`/`cpu.max.burst`/`cpu.stat`'s eight fields, `memory.min/low/high/max` semantics, all `memory.events` keys and the hierarchical-vs-`.local` distinction, `memory.stat`'s key list, `io.max`/`io.stat`/`io.weight` formats, `pids.*`, `cpuset.*` including partitions, and the four structural rules (availability, top-down, no-internal-process, delegation containment). Also the source for the statement that the v2 **device controller has no interface files** and the `misc` controller's real resource types.
- **CFS Bandwidth Control** (`Documentation/scheduler/sched-bwc.rst`) — https://docs.kernel.org/scheduler/sched-bwc.html — the global-pool/per-CPU-silo model, `sched_cfs_bandwidth_slice_us` (default 5 ms), the `min_cfs_rq_runtime` 1 ms retention, the post-5.4 non-expiry behaviour, and the hierarchical throttling rules (a child can be throttled by its parent's exhausted quota).
- **cgroups(7)** — https://man7.org/linux/man-pages/man7/cgroups.7.html — the userspace-facing overview of both versions, the v1→v2 migration rationale, and delegation from an administrator's perspective.
- **Kubernetes: Resource Management for Pods and Containers** — https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/ — how requests/limits/QoS are declared and what they mean operationally; the spec side of the mapping table.
- **Kubernetes: About cgroup v2** — https://kubernetes.io/docs/concepts/architecture/cgroups/ — the kubelet's use of the unified hierarchy, the `systemd` vs `cgroupfs` driver decision, and MemoryQoS; the bridge between the two references above.
- **kubelet source — `pkg/kubelet/cm/helpers_linux.go`** — https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/cm/helpers_linux.go — `MilliCPUToShares`, `MilliCPUToQuota`, `MinShares = 2`, `MaxShares = 262144`, `QuotaPeriod = 100000`, `MinQuotaPeriod = 1000`, `MinMilliCPULimit = 10`, and `ResourceConfigForPod`'s per-QoS branching. The ground truth for every number in the mapping section.
- **kubelet source — `pkg/kubelet/cm/cgroup_manager_linux.go`** — https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/cm/cgroup_manager_linux.go — `getCPUWeight()`, the **linear** shares→weight conversion used for pod and QoS slices, and `CgroupName.ToSystemd()`, which is where the `-` → `_` pod-UID escaping and cumulative slice naming live.
- **opencontainers/cgroups — `utils.go` and `fs2/cpu.go`** — https://github.com/opencontainers/cgroups — `ConvertCPUSharesToCgroupV2Value()`, the **quadratic-in-log₂** conversion runc/crun use for the container scope, and the code that actually writes `cpu.weight`/`cpu.max`. Read alongside the kubelet source to see why one node has two conversions.
- **KEP-2570: Memory QoS** — https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2570-memory-qos — the `memory.high` formula, the v1.27 revision from `factor × limit` to the request-interpolated form, the 0.9 default `memoryThrottlingFactor`, and the worked reason the original 0.8 form throttled JVMs too early.
- **KEP-2400: Node system swap support** — https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2400-node-swap — `NoSwap`/`LimitedSwap`, the Burstable-only policy, the proportional per-container `memory.swap.max` calculation, and its worked example. Alpha v1.22, beta v1.30, stable v1.34.

**Real-world engineering blogs**

- **Indeed Engineering — "Unthrottled: How a Valid Fix Becomes a Regression"** — https://engineering.indeedblog.com/blog/2019/12/cpu-throttling-regression-fix/ (mirror: https://medium.com/indeed-engineering/unthrottled-how-a-valid-fix-becomes-a-regression-f61eabb2fbd9) — *what it shows:* commit `512ac999` made per-CPU quota slices expire unused; combined with 5 ms slice granularity, a 1-CPU limit can occupy at most 20 cores before the pool empties, so on an 88-core machine most threads are throttled at near-zero aggregate usage. Worst-case latency 2 s → 30 ms after the fix (`de53fd7aedb1`, mainline 5.4).
- **Omio Engineering — "CPU limits and aggressive throttling in Kubernetes"** — https://engineering.omio.com/cpu-limits-and-aggressive-throttling-in-kubernetes-c5b20bd8a718 — *what it shows:* an independent rediscovery of the same mechanism, with the quota-consumed-in-a-fraction-of-the-period arithmetic and a real mitigation debate (raise limits vs remove limits entirely vs patch the kernel).

**Deeper dives**

- **LWN — "sched/fair: Fix low cpu usage with high throttling by removing expiration of cpu-local slices"** — https://lwn.net/Articles/792268/ — the technical dissection of the exact kernel bug behind the Indeed postmortem; cross-reference with [lesson 01](01-processes-and-scheduling.md), which covers the same commit from the run-queue/fairness side rather than the throttling-consequence side.
- **CPU throttling and the GOMAXPROCS-vs-quota mismatch** — https://victoriametrics.com/blog/kubernetes-cpu-go-gomaxprocs/ (and the `go.uber.org/automaxprocs` README) — *what it shows:* the container-sees-host-hardware gap end to end, and why Go 1.25 absorbed the fix into the runtime. The same reasoning applies to the JVM, OpenMP, and PyTorch's default thread counts.
