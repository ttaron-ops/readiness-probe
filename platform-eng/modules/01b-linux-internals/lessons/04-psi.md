---
lesson: "01b.4"
title: "Pressure Stall Information (PSI)"
module: "01b"
concept: "Pressure Stall Information (PSI)"
status: not-started
est_time: "5h"
prev: "03-cgroups-v2-and-k8s-enforcement.md"
next: "05-memory-and-oom.md"
artifacts: []
sources: 12
---

# 01b.4 · Pressure Stall Information (PSI)

> **Concept.** PSI measures the *time a workload was stalled waiting* for CPU, memory, or I/O — the saturation signal that utilization graphs hide.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

[Lesson 03](03-cgroups-v2-and-k8s-enforcement.md) built the whole cgroup-v2 enforcement picture — `cpu.max`, `cpu.weight`, `memory.max`, the `kubepods.slice` QoS tree — and every directory in that tree carried three files the lesson kept deferring: `cpu.pressure`, `memory.pressure`, `io.pressure`. This lesson is those files.

The distinction matters more than it sounds. Lesson 03 answered *"how much of a resource is this cgroup allowed, and did it hit the ceiling?"* — a question about **enforcement**, answered by `cpu.max` and proved by `cpu.stat`'s `nr_throttled`. This lesson answers a different question: *"is anything on this box being denied a resource right now, how much wall-clock time is that costing, and which cgroup is paying?"* — a question about **contention**, which can be true with no limit set anywhere. A cgroup with `cpu.max = max` will never throttle and will still starve if forty other threads want the same cores.

That is why PSI is the first file you read. `cpu.stat` tells you whether *your own configured cap* bit. `cpu.pressure` tells you whether *the machine* is failing you, cap or no cap. In the diagnostic order the rest of the module uses, PSI localizes the resource and the cgroup, and then the specific instrument — [lesson 05](05-memory-and-oom.md)'s `memory.events` and reclaim counters, [lesson 09](09-perf-ftrace-use.md)'s off-CPU flame graphs — finishes the job.

## Why this matters

Your GPU node shows 63% CPU utilization and the fleet dashboard is green. Yet training throughput has collapsed: `nvidia-smi` shows the GPUs at 31%, waiting. The data loader — the CPU-side processes that decode JPEGs, collate tensors, and hand batches to the GPU — is stalling, and every millisecond it stalls, expensive silicon idles. (Treat any per-GPU-hour figure as a dated snapshot rather than a constant; on-demand rates for a current training GPU have been of order **$2–3/GPU-hour** in 2026, so an 8-GPU node is roughly $16–24/hr. Recompute against your own contract.)

Utilization cannot see this, and not because your dashboard is badly built. Utilization measures *time the resource spent busy*. A CPU that is 63% busy is compatible with a perfectly healthy node **and** with a node where runnable threads spend most of their existence queued, because bursty demand leaves cores idle between bursts while still colliding within them. The two situations produce the same average and completely different p99s. **No amount of averaging a busy-time signal recovers a waiting-time signal; they are integrals of different quantities.**

PSI is the kernel's waiting-time signal, and it is accounted at the scheduling event rather than sampled afterwards. It is the mechanism behind:

- **`systemd-oomd`** and Meta's `oomd` (its production ancestor, written by PSI's own authors) and Android's LMKD — all of which kill or shed load based on memory pressure *before* the kernel OOM killer fires, avoiding the multi-minute thrash that precedes a hard OOM.
- **Kubernetes' own node metrics.** As of Kubernetes **v1.36**, kubelet-native PSI collection at node, pod, and container level is **stable** (KEP-4205, feature gate `KubeletPSI`; alpha v1.33, beta v1.34). This is no longer a metric you bolt on with a node exporter — it is in the Summary API and in the `container_pressure_*_seconds_total` counters.
- **Bin-packers and autoscalers** that want to pack GPU nodes densely without tipping them into thrash, which requires a signal that degrades continuously rather than a threshold that trips.

If your differentiator is cost and observability, PSI is the single highest-value saturation metric you can teach a fleet to emit. It converts "the node feels slow" into "`memory.pressure full avg10 = 42%` on `kubepods-burstable-pod7b4e_…slice` for the last 90 seconds," which is falsifiable, alertable, and citable in a postmortem.

## What's new here (calibration)

You already run `top`/`htop`/`kubectl top node`, read `%CPU` and `free -m`, and know the folklore that load average above core count is bad. [Lesson 01](01-processes-and-scheduling.md) gave you the precise mechanics of Linux load average — an EWMA over tasks in **R plus D** state, sampled on a timer — and established it as a *storage-health proxy*, not a saturation instrument. None of that is re-derived here.

What is genuinely new:

- **The exact task-state predicates the kernel evaluates** to decide that a domain is in a `some` or a `full` stall — four counters and one bit, straight out of `test_states()` in `kernel/sched/psi.c`. Once you have these, every "is it some or full?" question answers itself.
- **The accounting pipeline**: per-CPU time buckets updated at scheduler events, a 2-second aggregator, and a **non-idle-weighted** average across CPUs that makes PSI something other than "mean stall across all cores." This is why an idle core does not dilute pressure, and it is a fact almost nobody knows.
- **The correction on system-wide CPU `full`.** The widely-repeated claim that `/proc/pressure/cpu` has no `full` line is **out of date**: since Linux 5.13 the line is printed, hard-coded to zero, because CPU-full is undefined at the system level. Reading it as real pressure — or asserting the line is absent — are both wrong, and both come up in interviews.
- **The trigger/`poll()` API in full** — write format, window bounds, the privilege rule, the rate limit, and the working C program — because that is how `systemd-oomd` reacts in tens of milliseconds instead of on a scrape interval.
- **The Kubernetes surface as it actually exists in v1.36**: the Summary API structs, the six cAdvisor counter names, which of them is `some` and which is `full`, and — importantly — what PSI in Kubernetes still does *not* do (drive eviction).

## Core concepts

### 1. The measurement problem: three instruments, three blind spots

Before the mechanism, the gap it fills. There are three ways a Linux box has historically told you "things are bad," and each is blind to something the others see.

**Utilization** (`%CPU`, `%util`, `nvidia-smi`'s GPU-util) is *busy time over wall time*. It is bounded above by 100% by construction, which means it **saturates and then stops carrying information**. A node at 100% CPU with 2 runnable threads and a node at 100% CPU with 200 runnable threads report the same number; the second one is a hundred times worse for latency. Below 100% it is equally treacherous in the other direction, because the average smooths away the collisions that actually hurt.

**Load average** counts tasks in **R** (runnable) or **D** (uninterruptible sleep) and exponentially damps that count over 1, 5, and 15 minutes. Its problems are structural, not cosmetic: it is a **count**, not a time, so it cannot be compared against anything without knowing your core count; it merges CPU demand and I/O blocking into one number, so a value of 250 tells you nothing about *which* resource; and it is sampled on a timer, so brief stalls vanish. Lesson 01 covered this in detail.

**Pressure** is *time lost to waiting, over wall time, per resource*. It is a fraction like utilization, so it is comparable across machines of different sizes without normalizing. It is per-resource, so it names the culprit. And it is accumulated at the moment a task's state changes rather than sampled, so a 300 µs stall that a 1-second sampler would never see still lands in the counter.

```
                    WHAT EACH INSTRUMENT CAN AND CANNOT TELL YOU

  ┌────────────────────────────── one 64-core GPU node ───────────────────────────────┐
  │                                                                                    │
  │   kubepods.slice                          system.slice                             │
  │     ├── pod: trainer   (8 GPUs, 8 CPUs)     ├── containerd.service                  │
  │     ├── pod: loader    (16 threads)         └── kubelet.service                     │
  │     └── pod: metrics-agent                                                          │
  └────────────────────────────────────────────────────────────────────────────────────┘
        │                        │                                 │
        │                        │                                 │
  ┌─────▼──────────┐    ┌────────▼─────────────┐        ┌──────────▼────────────────────┐
  │ UTILIZATION    │    │ LOAD AVERAGE         │        │ PSI                           │
  │ mpstat, top,   │    │ /proc/loadavg        │        │ /proc/pressure/{cpu,memory,io}│
  │ %usr+%sys      │    │ EWMA of  R + D  count│        │ + per-cgroup *.pressure       │
  ├────────────────┤    ├──────────────────────┤        ├───────────────────────────────┤
  │ SEES           │    │ SEES                 │        │ SEES                          │
  │  how busy the  │    │  that a queue exists │        │  wall-clock time LOST to      │
  │  hardware was  │    │  somewhere, roughly  │        │  waiting, per RESOURCE,       │
  │                │    │                      │        │  per CGROUP, unsampled        │
  ├────────────────┤    ├──────────────────────┤        ├───────────────────────────────┤
  │ BLIND TO       │    │ BLIND TO             │        │ BLIND TO                      │
  │  queueing:     │    │  which resource      │        │  WHICH CODE PATH stalled      │
  │  63% busy can  │    │  (CPU? disk? NFS?)   │        │   → lesson 09 off-CPU stacks  │
  │  hide constant │    │  how BAD it is in    │        │  WHY the resource is short    │
  │  starvation.   │    │  time terms          │        │   → lesson 05 reclaim counters│
  │  Caps out at   │    │  brief stalls        │        │  anything about a resource    │
  │  100% and then │    │  (timer-sampled)     │        │   with no PSI accounting      │
  │  says nothing. │    │  per-cgroup blame    │        │   (network! see lesson 07)    │
  └────────────────┘    └──────────────────────┘        └───────────────────────────────┘

  The three are not competing versions of one metric. Utilization is a LEVEL
  (how full), load average is a COUNT (how many waiting), pressure is a RATE OF
  LOSS (how much time died). You need all three; you page on the third.
```

That last blind spot is worth stating plainly because it is a common overreach: **PSI covers CPU, memory, I/O, and (optionally) IRQ. It does not cover network.** There is no `/proc/pressure/net`. A node whose pods are timing out because the conntrack table is full ([lesson 07](07-networking-datapath-conntrack.md)) shows *nothing* in PSI. If your mental model is "PSI is the saturation signal," you will misdiagnose that incident.

### 2. What the kernel counts as a stall

PSI was written by **Johannes Weiner** at Facebook (documentation dated April 2018) and merged in **Linux 4.20**. Its central design decision is that stall accounting must happen where the kernel already knows a task cannot proceed — in the scheduler's enqueue/dequeue paths and in the memory reclaim path — rather than being reconstructed later from samples.

The kernel tracks, **per CPU, per cgroup (and once more for the system as a whole)**, four task counters and one flag. These are the entire input to PSI; everything else is arithmetic on them (`include/linux/psi_types.h`):

| Counter | Set when a task is… | Meaning |
|---|---|---|
| `NR_RUNNING` | runnable — `R` state, on-CPU or queued | wants a CPU |
| `NR_IOWAIT` | blocked waiting for I/O to complete | stalled on I/O |
| `NR_MEMSTALL` | inside a memory-stall region | stalled on memory |
| `NR_MEMSTALL_RUNNING` | *both* memstall **and** running | doing reclaim work, i.e. running but unproductive |
| `TSK_ONCPU` (a bit, not a count) | this CPU currently has a task of the group on it | there is a task actually executing here |

`NR_MEMSTALL` is not "using a lot of memory." The kernel brackets specific code regions with `psi_memstall_enter()` / `psi_memstall_leave()`, and those regions are the ones where a task is provably waiting on memory rather than making progress: **direct reclaim** (the allocator scanning LRU lists in the allocating task's own context), **refault waits** (blocking on a page that was evicted and is now being read back), **swap-in**, **compaction stalls**, and **thrashing page waits**. That is why a process with a large but stable resident set generates zero memory pressure: nothing is stalling. Pressure is about *motion*, not *size* — exactly the distinction [lesson 05](05-memory-and-oom.md) makes between `memory.current` (a level) and `workingset_refault_*` (a rate).

The fourth counter, `NR_MEMSTALL_RUNNING`, exists for one subtle reason spelled out in a comment in the header: for CPU and I/O, "somebody is running" is sufficient evidence that the domain still has productivity left. For memory it is not, because **page reclaimers are running while representing a stall**. A task in direct reclaim burns CPU cycles scanning page lists; by a naive "is anyone on-CPU?" test the domain looks productive when in fact 100% of its CPU time is being spent on overhead. So memory needs a separate count of "running but only because it is reclaiming."

From those five inputs the kernel derives the state of the domain on that CPU. This is `test_states()` in `kernel/sched/psi.c`, transcribed as a table:

| State | Predicate | In words |
|---|---|---|
| `IO_SOME` | `NR_IOWAIT > 0` | at least one task is blocked on I/O |
| `IO_FULL` | `NR_IOWAIT > 0 && NR_RUNNING == 0` | someone is blocked on I/O and **nothing** is runnable |
| `MEM_SOME` | `NR_MEMSTALL > 0` | at least one task is in a memory-stall region |
| `MEM_FULL` | `NR_MEMSTALL > 0 && NR_RUNNING == NR_MEMSTALL_RUNNING` | every runnable task is itself a reclaimer — no productive work at all |
| `CPU_SOME` | `NR_RUNNING > TSK_ONCPU` | more tasks want a CPU than are on one |
| `CPU_FULL` | `NR_RUNNING > 0 && !TSK_ONCPU` | tasks want a CPU and **none of them has one** |
| `NONIDLE` | `NR_IOWAIT \|\| NR_MEMSTALL \|\| NR_RUNNING` | the domain wanted *something*; used as the weight (§4) |

Read `CPU_SOME` carefully: `NR_RUNNING > oncpu` where `oncpu` is 0 or 1. With one task runnable and on the CPU, `1 > 1` is false — no pressure, correctly. With two runnable and one on-CPU, `2 > 1` is true — one task is queued, so `some`. This is why **`some` for CPU is exactly "the run queue for this group on this CPU had a waiter."**

And read `CPU_FULL` carefully, because it is the answer to the question everyone gets wrong. `NR_RUNNING > 0 && !oncpu` means *this group has work ready and is getting no CPU at all right now*. Inside a cgroup that is a completely ordinary situation: your throttled or out-weighted pod has 16 runnable threads and the CPU is executing somebody else's pod. At the **system** level the same predicate is meaningless — if the whole system has runnable tasks and none is on any CPU, the scheduler is broken, not contended. The kernel therefore declares CPU-full undefined system-wide (§5).

### 3. `some` and `full` on a timeline — the picture to memorize

Definitions in prose are slippery. Here is one cgroup with three tasks sharing one CPU's worth of a domain, drawn over 200 ms.

```
   ONE cgroup · three tasks (A, B, C) · one CPU in the domain · 200 ms of wall clock
   Each column is a 20 ms slice.

   R = running on-CPU, productive        Q = runnable but queued (stalled on CPU)
   M = blocked in reclaim (memstall)     S = sleeping / voluntarily idle
                                             — sleeping tasks are INVISIBLE to PSI

   slice        1    2    3    4    5    6    7    8    9   10
   ms         0-20 20-40 …                                  180-200
            ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
    task A  │ R  │ Q  │ R  │ M  │ M  │ S  │ R  │ Q  │ R  │ R  │
    task B  │ S  │ R  │ Q  │ R  │ M  │ S  │ Q  │ Q  │ Q  │ S  │
    task C  │ S  │ S  │ S  │ S  │ M  │ S  │ Q  │ Q  │ S  │ S  │
            └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
   on CPU?    A    B    A    B    –    –    A    –    A    A
                                  ▲    ▲         ▲
                        all three in    nothing  the CPU is running
                        reclaim: the    wants    ANOTHER cgroup's task:
                        CPU is busy     the CPU  this group has 3 ready
                        but 100% of     at all   threads and zero cores
                        it is overhead

   cpu  some  ····  ████ ████ ···· ···· ···· ████ ████ ████ ····   = 100 ms → 50%
   cpu  full  ····  ···· ···· ···· ···· ···· ···· ████ ···· ····   =  20 ms → 10%
   mem  some  ····  ···· ···· ████ ████ ···· ···· ···· ···· ····   =  40 ms → 20%
   mem  full  ····  ···· ···· ···· ████ ···· ···· ···· ···· ····   =  20 ms → 10%
   NONIDLE    ████  ████ ████ ████ ████ ···· ████ ████ ████ ████   = 180 ms

   slice 4: A is in reclaim (mem some) but B is productively running,
            so NR_RUNNING(1) ≠ NR_MEMSTALL_RUNNING(0) → NOT mem full.
   slice 5: all three are reclaimers, NR_RUNNING == NR_MEMSTALL_RUNNING → mem FULL.
   slice 6: everyone asleep. NONIDLE is 0, so this slice carries NO WEIGHT at all
            in the average (§4) — PSI never reports "we were fine" as evidence.
   slice 8: three tasks runnable, none on a CPU → cpu some AND cpu full.
```

**Now the arithmetic that makes the two lines mean different things.** Over the 200 ms window, this cgroup shows `cpu some = 50%` and `cpu full = 10%`. Suppose instead you saw `some = 10%` and `full = 10%` on two different cgroups and had to rank them. Work out lost task-time, with 3 tasks and a 200 ms window (600 task-ms of capacity):

```
  Case A — some = 10%, full = 0%
    "At least one task was waiting for 20 ms of the 200."
    Best case:  exactly one task waited, the other two ran.
                lost = 1 task × 20 ms = 20 task-ms  →  20/600 = 3.3% of capacity
    Worst case: all three waited for that 20 ms — but then full would also be
                nonzero, so by construction this case has someone productive.
    ⇒ some=10% bounds the damage at "somewhere between ~3% and just under 10%."
      It is a CONTENTION indicator. It tells you a queue formed. It does NOT
      tell you how much throughput you lost.

  Case B — full = 10%  (which forces some ≥ 10%)
    "For 20 ms of the 200, NOTHING in this cgroup made progress."
    lost = 3 tasks × 20 ms = 60 task-ms  →  60/600 = 10.0% of capacity
    …and it is 10% for ANY task count, because "full" means the whole domain
    was dead. Double the threads and you lose 10% of a bigger number.
    ⇒ full=10% is a THROUGHPUT MULTIPLIER. A workload that would have finished
      in 100 s now takes 100/(1−0.10) = 111 s. You can put that in a budget.
```

**That is the whole distinction.** `some` is the early-warning, latency-flavoured signal: queues are forming, tail latency is inflating, act before it gets worse. `full` is the loss accounting: this fraction of your wall clock produced nothing, and you can multiply it directly by what the node costs. `full ≤ some` always, mechanically, because every `full` predicate has its `some` predicate as a conjunct.

For a GPU node the distinction has a very concrete reading. Loader `cpu.pressure some = 35%, full = 2%` means the loader's threads are queueing — batches arrive late, the GPU sees jitter, p99 step time inflates, but throughput is roughly intact. Loader `cpu.pressure full = 22%` means that for 22% of every wall-clock second the loader produced **no batches at all**, and eight GPUs downstream waited for exactly that long. The first is a tuning problem. The second is money leaving the building at a rate you can compute.

### 4. How the number is actually produced

Knowing the predicates is not enough to read the file, because what lands in `total=` is not a simple sum of the per-CPU stall times. The pipeline has four stages, and the third one surprises people.

```
   ①  SCHEDULER / RECLAIM HOOKS                    ②  PER-CPU TIME BUCKETS
   ─────────────────────────────                   ──────────────────────
   psi_task_change()   ← enqueue/dequeue,          struct psi_group_cpu {
   psi_task_switch()   ← context switch              u32 tasks[4];   ← the counters
   psi_memstall_enter()← entering direct reclaim     u32 state_mask; ← test_states()
   psi_memstall_leave()← leaving it                  u32 times[NR_PSI_STATES];  (ns)
                                                     u64 state_start;
        each event:                                }
        1. update tasks[]                          On every state change, the time
        2. recompute state_mask = test_states()      since state_start is added to
        3. add (now − state_start) to times[old]     times[] for every state bit
                                                     that was set. Nothing is sampled.
                        │
                        ▼
   ③  AGGREGATOR, every PSI_FREQ = 2*HZ+1 jiffies (≈2 s), per group
   ────────────────────────────────────────────────────────────────
       for each CPU:
            nonidle_cpu = jiffies(times[PSI_NONIDLE] delta)
            for each state s:  deltas[s] += times[s]_delta × nonidle_cpu
            nonidle_total += nonidle_cpu

       sample[s] = deltas[s] / nonidle_total        ◀── NON-IDLE-WEIGHTED MEAN,
                                                        not a plain mean
       total[s]  += sample[s]                       ◀── this is what `total=` reports
       sample     = min(sample, period)             ◀── clamp at 100%; the overage is
                                                        carried into the next period
                        │
                        ▼
   ④  EWMA into the three averages          then:  seq_printf("%s avg10=… total=%llu")
   ──────────────────────────────────              with total converted ns → µs
       pct = sample × 100 / period
       avg10  = calc_load(avg10,  EXP_10s  = 1677, pct)
       avg60  = calc_load(avg60,  EXP_60s  = 1981, pct)
       avg300 = calc_load(avg300, EXP_300s = 2034, pct)
       (fixed point, FIXED_1 = 2048;  1677/2048 = 0.8187 = e^(−2/10))
```

**Stage ③ is the one to internalize: PSI is a non-idle-weighted average across CPUs, not a plain average.** The comment in `collect_percpu_times()` says why: weighting by non-idle time "eliminates artifacts from uneven loading, or even entirely idle CPUs." Work it through:

```
  8-CPU node, one 2.0 s aggregation period, one cgroup.
  CPUs 0–3:  non-idle for the full 2.0 s;  cpu_some stall time = 0.5 s each
  CPUs 4–7:  non-idle for only 0.1 s;      cpu_some stall time = 0.0 s

  PLAIN MEAN (what you might assume):
      (4 × 0.5 s + 4 × 0.0 s) / 8 CPUs = 0.25 s per 2.0 s  =  12.5%

  WHAT THE KERNEL ACTUALLY COMPUTES:
      deltas       = Σ (stall × nonidle)
                   = 4 × (0.5 × 2.0) + 4 × (0.0 × 0.1)     = 4.00  s²
      nonidle_total= 4 × 2.0 + 4 × 0.1                     = 8.40  s
      sample       = 4.00 / 8.40                           = 0.476 s
      pct          = 0.476 / 2.0                           = 23.8%

  The four idle cores barely dilute the number, because they contributed almost
  no weight. PSI reports pressure on the part of the machine that was actually
  doing something — which is what you want, because a pod pinned to 4 cores of a
  64-core node should not have its pressure divided by 16.
```

**Consequences you can act on:**

- `total=` is in **microseconds of wall-clock-equivalent stall**, already normalized across CPUs. It is *not* CPU-seconds and it is *not* summed across threads. Unlike `cpu.stat`'s `throttled_usec` (which [lesson 03](03-cgroups-v2-and-k8s-enforcement.md) showed can exceed one second per wall-second because it aggregates threads), **`total=` can never advance faster than wall clock**. If you sample it 1 s apart and the delta is 270,000 µs, that is 0.27 s of stall in that second — 27%, directly comparable to `avg10`.
- The averages update only every ~2 s. Reading the file more often than that gives you the same `avgN` values (the read path does force an aggregation pass first, but the EWMA only advances on a period boundary). **If you want sub-2-second resolution, you must diff `total=`, not poll `avg10`.**
- `avg10` is an EWMA with a 10-second time constant, not a 10-second sliding window. After a step change from 0% to 100%, `avg10` reads 63% after 10 s, 86% after 20 s, 95% after 30 s. That lag is exactly why the trigger API (§7) exists: `systemd-oomd` cannot wait 30 seconds to learn the box is thrashing.
- The `min(sample, period)` clamp means a reported value never exceeds 100%, and excess stall time recorded slightly late is "punted into the future until pressure subsides." A sustained 100% `full` reading may therefore persist for one extra period after the stall ends. Do not treat the trailing edge as precise.

### 5. The files, field by field

**System-wide:** `/proc/pressure/cpu`, `/proc/pressure/memory`, `/proc/pressure/io`, and — only when the kernel has `CONFIG_IRQ_TIME_ACCOUNTING` and IRQ time accounting is enabled — `/proc/pressure/irq`.

```
$ cat /proc/pressure/cpu
some avg10=8.42 avg60=6.13 avg300=4.20 total=1290034567
full avg10=0.00 avg60=0.00 avg300=0.00 total=0

$ cat /proc/pressure/memory
some avg10=0.00 avg60=0.12 avg300=0.31 total=8452119
full avg10=0.00 avg60=0.08 avg300=0.19 total=5109882

$ cat /proc/pressure/io
some avg10=88.20 avg60=84.11 avg300=71.03 total=182933944912
full avg10=71.50 avg60=68.02 avg300=57.44 total=140222110293

$ cat /proc/pressure/irq          # only if CONFIG_IRQ_TIME_ACCOUNTING
full avg10=0.03 avg60=0.02 avg300=0.02 total=6193044
```

| Field | Unit | Exactly what it is |
|---|---|---|
| `avg10` | percent of wall time, 2 decimals | EWMA of the 2-second samples with decay `e^(−2/10)` per period — a 10-second *time constant*, not a window |
| `avg60` | percent | same, decay `e^(−2/60)` |
| `avg300` | percent | same, decay `e^(−2/300)` |
| `total` | **microseconds**, monotonic | cumulative non-idle-weighted stall time since boot (system) or since cgroup creation (cgroup). The kernel accumulates in ns and divides by 1000 on output. |

Three details that decide whether you read the file correctly:

**1. `/proc/pressure/cpu` *does* have a `full` line, and it is always zero.** The kernel documentation is explicit: "CPU full is undefined at the system level, but has been reported since 5.13, so it is set to zero for backward compatibility." In `psi_show()` the guard is literally `if (!(group == &psi_system && res == PSI_CPU && full))` — for that one combination the code skips copying the real values and prints zeros. So:

- On kernels **≥ 5.13**, expect two lines; the second is a constant zero and carries no information.
- On kernels **4.20 – 5.12**, the `full` line for system CPU is genuinely absent, which is where the folklore comes from.
- **Per-cgroup `cpu.pressure` reports a real, meaningful `full`** on every version, because the predicate `NR_RUNNING > 0 && !TSK_ONCPU` is well-defined for a group: your tasks are ready and the CPU is running someone else's.

A parser that assumes exactly one line for system CPU breaks on 5.13+; a parser that assumes two lines breaks on older kernels; and a dashboard that graphs system CPU `full` shows a flat zero and will one day be presented as evidence that "CPU is fine."

**2. `/proc/pressure/irq` has only a `full` line.** IRQ time is, by definition, time no task could use, so there is no meaningful `some`. `psi_show()` sets `only_full = (res == PSI_IRQ)` and prints one line. It returns `-EOPNOTSUPP` if IRQ time accounting is off. On a GPU node with heavy NIC interrupt load this file is genuinely useful and almost never scraped.

**3. Per-cgroup files are the point.** In any cgroup-v2 directory:

```
$ CG=/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/\
kubepods-burstable-pod7b4e_2a91_4c03_8e15_6d9f0b2a7c34.slice
$ cat $CG/cpu.pressure
some avg10=41.70 avg60=38.20 avg300=22.50 total=99182340
full avg10=12.30 avg60=10.10 avg300=6.40  total=41029118

$ cat $CG/memory.pressure
some avg10=0.00 avg60=0.00 avg300=0.02 total=411882
full avg10=0.00 avg60=0.00 avg300=0.01 total=210044

$ cat $CG/io.pressure
some avg10=63.44 avg60=51.02 avg300=30.18 total=88421990233
full avg10=58.90 avg60=47.71 avg300=28.02 total=79021144108
```

Same format, same units, aggregated over **the tasks in that cgroup and its descendants**. Derive the path the way lesson 03 taught — `/sys/fs/cgroup$(cut -d: -f3 /proc/<pid>/cgroup)` — never by string-building, because of the pod-UID `-` → `_` escaping.

There is also **`cgroup.pressure`** (Linux ≥ 6.1), a read-write single-value file, default `1`. Writing `0` disables PSI accounting for that cgroup. It is deliberately **non-hierarchical**: turning it off in a parent does not affect children, and children do not need it enabled in ancestors. It exists because PSI accounts stalls separately at every level and aggregates upward, which costs real cycles in a deep hierarchy — so you can switch off the non-leaf levels you never read. Two operational consequences: (a) if a pod's `*.pressure` files read all-zero on a busy node, check `cgroup.pressure` before concluding the pod is healthy; (b) `runc`'s PSI reader treats a missing file as "kernel < 4.20, or `CONFIG_PSI` unset, or PSI turned off for this cgroup" and returns no stats at all — silently.

### 6. Enablement: why `/proc/pressure/` might not be there

| Requirement | Where it comes from | Symptom when missing |
|---|---|---|
| `CONFIG_PSI=y` | kernel build config | `/proc/pressure/` does not exist at all |
| `CONFIG_PSI_DEFAULT_DISABLED=n`, **or** `psi=1` on the kernel cmdline | build config + bootloader | `/proc/pressure/` missing, or reads return `-EOPNOTSUPP` |
| cgroup2 mounted | `stat -fc %T /sys/fs/cgroup` → `cgroup2fs` | no per-cgroup `*.pressure` files |
| `cgroup.pressure` = 1 for that cgroup (≥ 6.1) | runtime, per cgroup | that cgroup's files exist but never move |
| `CONFIG_IRQ_TIME_ACCOUNTING` + IRQ accounting on | build config | `/proc/pressure/irq` absent or `EOPNOTSUPP` |

The `CONFIG_PSI_DEFAULT_DISABLED` help text is worth quoting in substance because it is the honest answer to "what does PSI cost?": the feature adds code to the scheduler's task wakeup and sleep paths; the overhead is described as too low to affect common scheduling-intensive workloads such as web servers and memcache in practice, but it does show up in artificial scheduler stress tests like `hackbench`. Translation: enable it, and do not accept "it's too expensive" without a measurement on your workload.

Check it in one line, and note the failure mode where the file exists but reads fail:

```bash
$ stat -fc %T /sys/fs/cgroup && ls /proc/pressure/ && cat /proc/pressure/cpu
cgroup2fs
cpu  io  memory
some avg10=0.11 avg60=0.09 avg300=0.10 total=1200334
full avg10=0.00 avg60=0.00 avg300=0.00 total=0

$ grep -o 'psi=[01]' /proc/cmdline          # empty output is fine if
                                            # CONFIG_PSI_DEFAULT_DISABLED=n
$ zgrep -E 'CONFIG_PSI' /proc/config.gz 2>/dev/null || \
  grep -E 'CONFIG_PSI' /boot/config-$(uname -r)
CONFIG_PSI=y
# CONFIG_PSI_DEFAULT_DISABLED is not set
```

Some vendor kernels (the `runc` source notes CentOS Stream 9 as an example) return `ENOTSUP` on *read* rather than hiding the file, when `psi=1` was required and not passed. If your collector reports zeros for every cgroup on one node family and real numbers everywhere else, that is the shape of the bug.

### 7. Triggers: reacting in milliseconds instead of on a scrape interval

The averages have a floor on their reaction time — `avg10` is 63% of the way to a step change only after 10 seconds. For a killer daemon that is far too slow: by then the machine has been thrashing for ten seconds. So PSI exposes a second interface on the same files.

**The mechanism.** `open()` a pressure file `O_RDWR | O_NONBLOCK`, `write()` a threshold specification, then `poll()`/`select()`/`epoll()` the fd for `POLLPRI`. The kernel wakes you when the trigger condition is met.

The write format is exactly:

```
<some|full> <stall amount in µs> <time window in µs>
```

Semantics: *"wake me if cumulative stall time exceeds N µs within any window of W µs."* `some 150000 1000000` on `/proc/pressure/memory` means "150 ms of partial memory stall inside any 1-second window." `full 50000 1000000` on `/proc/pressure/io` means "50 ms of full I/O stall per second."

The rules, from `psi_trigger_create()` and the PSI documentation:

| Rule | Value | Why |
|---|---|---|
| Window bounds | documented as 500 ms – 10 s; current mainline validates `0 < window ≤ 10 s` (`WINDOW_MAX_US = 10000000`) | below the documented floor, polling cost dominates; above 10 s, use the averages instead |
| Update rate while stalled | window / `UPDATES_PER_WINDOW` (= 10) → 50 ms at a 500 ms window, 1 s at a 10 s window | bounded polling cost |
| Unprivileged writers | window must be a multiple of **2 s** unless the opener has `CAP_SYS_RESOURCE` | non-privileged triggers reuse the existing 2 s averaging work; no RT thread is spawned for them |
| One trigger per fd | second write to the same fd fails with **`EBUSY`** | each trigger needs its own pollable fd; `open()` the same file again for a second trigger |
| Notification rate | at most **one per tracking window** | prevents a wakeup storm during sustained pressure |
| Activation hysteresis | once active, a monitor stays active for at least one full window | stops flapping when the system bounces in and out of stall |
| Lifetime | trigger de-registers when the fd is closed | no cleanup API needed |

Monitors only run while the domain is actually in a stall state for the monitored metric, and stop when it leaves — so an idle node pays nothing.

Here is the complete monitor, in the shape the kernel documentation demonstrates, annotated:

```c
/* psi-watch.c — wake up when memory partial stall exceeds 150 ms in any 1 s window.
 * Build:  cc -Wall -o psi-watch psi-watch.c
 * Run:    ./psi-watch                     (system-wide)
 *         PSI_FILE=/sys/fs/cgroup/kubepods.slice/memory.pressure ./psi-watch
 */
#include <errno.h>
#include <fcntl.h>
#include <poll.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    const char *path = getenv("PSI_FILE");
    const char  trig[] = "some 150000 1000000";   /* 150 ms stall / 1 s window */
    struct pollfd fds;
    int n;

    if (!path)
        path = "/proc/pressure/memory";

    /* O_RDWR because we write the trigger; O_NONBLOCK so poll() drives us. */
    fds.fd = open(path, O_RDWR | O_NONBLOCK);
    if (fds.fd < 0) {
        fprintf(stderr, "open %s: %s\n", path, strerror(errno));
        return 1;                    /* ENOENT => CONFIG_PSI off or psi=0 */
    }
    fds.events = POLLPRI;            /* PSI signals with POLLPRI, not POLLIN */

    /* strlen()+1 sends the NUL; the kernel sscanf()s this buffer.
     * A second write on THIS fd would fail with EBUSY. */
    if (write(fds.fd, trig, strlen(trig) + 1) < 0) {
        fprintf(stderr, "write trigger: %s\n", strerror(errno));
        return 1;                    /* EINVAL => bad window, or unprivileged
                                        window not a multiple of 2 s */
    }

    fprintf(stderr, "watching %s for '%s'\n", path, trig);
    for (;;) {
        n = poll(&fds, 1, -1);       /* block indefinitely */
        if (n < 0) {
            if (errno == EINTR) continue;
            fprintf(stderr, "poll: %s\n", strerror(errno));
            return 1;
        }
        if (fds.revents & POLLERR) { /* cgroup was removed under us */
            fprintf(stderr, "POLLERR: event source is gone\n");
            return 0;
        }
        if (fds.revents & POLLPRI) {
            char buf[256];
            ssize_t r;
            /* Re-read the file to see WHERE it is now; the wakeup itself
             * carries no payload. Rate-limited to one per window. */
            lseek(fds.fd, 0, SEEK_SET);
            r = read(fds.fd, buf, sizeof(buf) - 1);
            if (r > 0) { buf[r] = '\0'; fputs(buf, stdout); }
            puts("--- threshold crossed ---");
            fflush(stdout);
        }
    }
}
```

**This is not an academic exercise — it is what `systemd-oomd` does.** Its documented defaults (`oomd.conf`) are:

| Setting | Default | Meaning |
|---|---|---|
| `DefaultMemoryPressureLimit=` | **60%** | the limit on a unit's cgroup `memory.pressure` **`full avg10`** — "the fraction of time in a 10 second window in which all tasks in the control group were delayed" |
| `DefaultMemoryPressureDurationSec=` | **30 s** | how long the limit must be continuously exceeded before acting |
| `SwapUsedLimit=` | **90%** | if both memory and swap usage exceed this, act on descendant cgroups using more than 5% of total swap, worst first |

When the pressure condition holds, `systemd-oomd` acts on eligible descendant cgroups **starting with the ones doing the most reclaim activity**, killing per unit policy (`ManagedOOMMemoryPressure=`, and `kill-all` / `kill-by-pgscan` / `kill-by-swap` strategies). Note what that gives you that the kernel OOM killer does not: it fires on a *rate* signal 30 seconds into sustained thrash, on a cgroup chosen by policy, while the machine is still responsive — instead of after an allocation has already failed, against a victim chosen by `oom_badness()`'s page count ([lesson 05](05-memory-and-oom.md)).

### 8. PSI in Kubernetes, exactly as it exists today

**Status (KEP-4205, "Expose PSI Metrics", SIG-Node, status `implemented`):** feature gate **`KubeletPSI`**, alpha in **v1.33**, beta in **v1.34**, **stable in v1.36**. The kubelet reads PSI from cgroup v2 via cAdvisor and the `runc` libcontainer cgroup library — no shelled-out binaries, no extra daemon.

**What you get.** Two surfaces.

*(a) The Summary API* (`/stats/summary` on the kubelet). The KEP adds these structs, mirroring the file format one-to-one:

```go
// PSI data for an individual resource.
type PSIData struct {
    Total  uint64  `json:"total"`   // cumulative stall time (see units note below)
    Avg10  float64 `json:"avg10"`   // % over a 10 s window
    Avg60  float64 `json:"avg60"`   // % over a 60 s window
    Avg300 float64 `json:"avg300"`  // % over a 300 s window
}

// PSI statistics for an individual resource.
type PSIStats struct {
    Some PSIData `json:"some,omitempty"`
    Full PSIData `json:"full,omitempty"`
}

type CPUStats    struct { /* … */ PSI *PSIStats `json:"psi,omitempty"` }
type MemoryStats struct { /* … */ PSI *PSIStats `json:"psi,omitempty"` }
type IOStats     struct {                       // new: node-level IO stats
    Time metav1.Time `json:"time"`
    PSI  *PSIStats   `json:"psi,omitempty"`
}
type NodeStats   struct { /* … */ IO *IOStats `json:"io,omitempty"` }
```

A units warning worth carrying: the KEP's Go comment labels `Total` as nanoseconds, but the value flowing through is the kernel's field verbatim. The `runc` cgroup library parses `total=` with a plain `ParseUint` and stores it unchanged, and cAdvisor converts it with a function named `asMicrosecondsToSeconds` when exporting. **Treat `Total` as microseconds and verify against `/sys/fs/cgroup/.../cpu.pressure` on your own node before you build an alert on it.**

*(b) Prometheus counters*, emitted by cAdvisor when the `PressureMetrics` set is enabled. There are exactly six, and the naming is the part people get wrong:

| Metric | Source field | Meaning |
|---|---|---|
| `container_pressure_cpu_waiting_seconds_total` | CPU **`some`** `total` | time tasks in the container **waited** due to CPU congestion |
| `container_pressure_cpu_stalled_seconds_total` | CPU **`full`** `total` | time **no task** in the container could make progress due to CPU congestion |
| `container_pressure_memory_waiting_seconds_total` | memory `some` `total` | as above, memory |
| `container_pressure_memory_stalled_seconds_total` | memory `full` `total` | as above, memory |
| `container_pressure_io_waiting_seconds_total` | I/O `some` `total` | as above, I/O |
| `container_pressure_io_stalled_seconds_total` | I/O `full` `total` | as above, I/O |

**`waiting` = `some`, `stalled` = `full`.** All six are `CounterValue`, converted from microseconds to **seconds**. Because they are counters of stall-seconds, `rate()` over them yields a dimensionless fraction — stall-seconds per second — which is directly the pressure percentage without the EWMA lag:

```promql
# Fraction of wall time the loader container made NO progress on CPU.
rate(container_pressure_cpu_stalled_seconds_total{pod=~"loader-.*"}[1m])

# Same for I/O, as a percentage, per pod, top 5.
topk(5, 100 * rate(container_pressure_io_stalled_seconds_total[5m]))

# The GPU-node alert that matters: sustained FULL memory pressure, which is the
# pre-OOM thrash signature from lesson 05.
100 * rate(container_pressure_memory_stalled_seconds_total[2m]) > 10
```

Two `rate()`-hygiene notes. First, these are cumulative counters that reset when the container restarts, which is exactly the case `rate()` handles. Second, use a range at least 4× your scrape interval; at a 15 s scrape, `[1m]` is the floor.

**What Kubernetes does *not* do with PSI, as of v1.36.** KEP-4205's Non-Goals are explicit: further uses of the PSI metric "for pod evictions, userspace OOM kills, and so on" are left to **future KEPs**. Node-pressure eviction still keys off level signals — `memory.available` computed from working set, plus `nodefs`/`imagefs` — not off pressure. So the correct framing is: **the kubelet now measures and exports PSI; it does not yet act on it.** Acting on it is your job, in an alerting rule or a custom controller, and that gap is precisely where a platform team adds value.

### 9. Reading PSI on a GPU node: starved versus merely busy

Here is the reasoning chain the module has been building toward. A GPU training step is a pipeline, and the expensive stage is downstream of the cheap ones:

```
   ┌──────────┐   ┌───────────┐   ┌───────────┐   ┌──────────┐   ┌──────────────┐
   │ shard    │──▶│ decode &  │──▶│ collate & │──▶│ H2D copy │──▶│ GPU kernels  │
   │ read     │   │ augment   │   │ pin       │   │ (PCIe)   │   │ fwd+bwd+opt  │
   │ (NFS/S3) │   │ (CPU)     │   │ (CPU+RAM) │   │          │   │              │
   └──────────┘   └───────────┘   └───────────┘   └──────────┘   └──────────────┘
        ▲               ▲               ▲                             ▲
   io.pressure     cpu.pressure    memory.pressure               nvidia-smi util
   on the loader   on the loader   on the loader                 (the SYMPTOM)
   cgroup          cgroup          cgroup

   The GPU being idle is never the cause. PSI on the loader's cgroup tells you
   WHICH upstream stage lost the time — and `full` tells you HOW MUCH.
```

The diagnostic rule, stated so you can apply it under pressure:

| What you see | What it means | First action |
|---|---|---|
| Node CPU high, loader `cpu.pressure` ~0 | The node is **busy**, not starved. Someone is using the CPU productively. | Nothing. Do not add capacity. |
| Node CPU moderate, loader `cpu.pressure some` high, `full` low | Loader threads are **queueing** — bursty collisions or a noisy neighbour. GPU sees jitter. | Check sibling cgroups' `cpu.weight`; check thread count vs `cpu.max` (lesson 03 §4) |
| Loader `cpu.pressure full` high | Loader is **getting no CPU at all** for that fraction of time. Direct GPU idle. | Is `cpu.max` capping it (`cpu.stat nr_throttled`)? Or is `cpu.weight` losing to a neighbour? |
| Loader `io.pressure full` high | Blocked reading shards. Storage or network FS, not CPU. | Prefetch depth, shard size, page-cache residency |
| Loader `memory.pressure full` climbing | Reclaim thrash — pre-OOM. | Lesson 05: `memory.events`, `workingset_refault_*`, right-size the limit |
| Trainer pod `cpu.pressure full` high, loader clean | The **trainer's own** host threads are starved (NCCL progress threads, CUDA runtime) | `cpuset.cpus` / NUMA placement (lesson 06) |

And the boundary of what PSI can do, stated just as plainly: **PSI localizes the resource and the cgroup. It does not identify the code path.** It will tell you `io.pressure full = 58%` on the loader; it will not tell you that the stall is in `read()` on a specific shard file over a specific NFS mount. That is [lesson 09](09-perf-ftrace-use.md)'s off-CPU flame graph, built from scheduler tracepoints. PSI is the wide-angle instrument; off-CPU analysis is the zoom. Running the zoom first is how people lose the first hour of an incident.

## Perspectives

**Kernel-mechanism view.** PSI is instrumentation placed exactly where the kernel already possesses the information — the scheduler's task-state transitions and the reclaim path's explicit `psi_memstall_enter()`/`leave()` brackets. Because it hooks state *changes* rather than sampling state, it captures stalls far shorter than any sampler's interval, at the cost of a small amount of work on the wakeup and sleep paths (the cost `CONFIG_PSI_DEFAULT_DISABLED` exists to let paranoid builders avoid). The per-CPU buckets with seqcount-protected reads exist so that the hot path never takes a lock and never touches another CPU's cache line; all the cross-CPU work happens in the 2-second aggregator. That is the same design shape as CFS bandwidth control's per-CPU quota silos from [lesson 03](03-cgroups-v2-and-k8s-enforcement.md): keep the hot path local, reconcile globally on a timer.

**Operator/SRE view.** The value is that pressure turns an adjective into a number. "The node feels slow" is not falsifiable and cannot be alerted on; `io.pressure full avg60 = 68%` is both. More importantly, pressure is *comparable*: 40% on a 4-core node and 40% on a 128-core node mean the same thing — 40% of wall time lost — whereas load average 40 means opposite things on those two machines. That comparability is what lets one alerting rule cover a heterogeneous fleet. The operational discipline is: page on `full`, investigate on `some`, and never page on system CPU `full` (it is a constant zero).

**GPU-fleet-specific view.** The economics of a GPU node invert the usual priorities. On a general web fleet, CPU pressure costs you latency on a cheap resource. On a GPU node, CPU or I/O pressure in the *loader* cgroup costs you idle time on the most expensive resource in the building, at a leverage ratio of roughly the GPU-to-CPU cost ratio. This is why PSI belongs on the loader's cgroup specifically, not just on the node: node-level pressure averages the starving loader together with seven healthy sidecars and reports something reassuring. Attribute to the leaf cgroup — the same discipline lesson 03 built for cost attribution, applied to loss attribution.

**Failure-mode/economics view.** PSI and level-based thresholds are different *kinds* of signal, not competing versions of one. `memory.available` is a fuel gauge: how much runway is left. `memory.pressure full` is a rate-of-climb indicator: how hard the engine is working to keep the gauge where it is. A node can sit at a comfortable `memory.available` for twenty minutes *because* reclaim is thrashing continuously to keep it there — the gauge is fine and the workload has lost 30% of its throughput. The rate signal leads the level signal, sometimes by minutes, sometimes (on a swapless node running mostly anonymous memory, which is every training node) by only seconds. Knowing which of those two regimes you are in determines whether PSI is an early warning or merely an obituary — see [lesson 05](05-memory-and-oom.md) §on unreclaimable footprints.

## Real-world use cases

- **Meta's `oomd`, and PSI's origin story.** PSI was written at Facebook by Johannes Weiner, and `oomd` — the userspace OOM daemon — is the reason it exists. The argument is structural: the in-kernel OOM killer runs only *after* an allocation has already failed, by which time the machine has typically spent minutes thrashing with every workload on it degraded, and it chooses a victim by `oom_badness()`, which is a page count plus a bias with no notion of which workload matters to the business. `oomd` watches memory PSI and cgroup state and acts on operator-written policy while the machine is still responsive. *What it shows:* the kernel's killer is a safety net, not a policy engine; PSI is the signal that lets a policy layer sit above it. Its descendant `systemd-oomd` ships in mainstream distributions with the defaults in §7 — `full avg10` over 60% for 30 s.
- **`systemd-oomd` and Android's LMKD as the same pattern, twice.** Both use the PSI trigger/`poll()` interface rather than polling the averages, and both act on `full` rather than `some`, for the reason §3 derives: `full` is the only one of the two that is a throughput multiplier you can threshold meaningfully. *What it shows:* when two independent production systems with very different constraints (a server init system and a phone's low-memory killer) converge on the same signal and the same API, that is a strong prior for your own alerting design.
- **Kubernetes KEP-4205, alpha v1.33 → beta v1.34 → stable v1.36.** The KEP's own motivation is that before it, "to identify disruptions caused by resource crunches, Kubernetes users need to install node exporter to read PSI metric"; the feature moves that into the kubelet's Summary API and metrics endpoint. Its Non-Goals leave eviction and userspace OOM kills to future KEPs. *What it shows:* two things. First, the metric this lesson teaches is now on the node by default on any v1.36+ cluster, so there is no bolt-on step. Second, the platform measures but does not yet act — the gap between "PSI is exported" and "the node responds to PSI" is exactly where a platform engineer's alerting rules and controllers earn their keep.

## Worked example

**The incident: eight GPUs at 31% utilization on a node whose dashboards are green.**

*(Transcripts below are representative reconstructions with internally consistent arithmetic, not a captured session. Run the Practice section to produce your own.)*

**Step 1 — establish that utilization is not the story.**

```bash
$ nproc
64
$ mpstat 1 3 | tail -2
Average:  all  58.90  0.00  4.10  1.20  0.00  0.90  0.00  34.90
#              %usr          %sys  %iowait                  %idle
```

63% busy, 35% idle. A capacity-planning dashboard reports this node as comfortable. Meanwhile:

```bash
$ nvidia-smi --query-gpu=index,utilization.gpu --format=csv,noheader
0, 31 %
1, 30 %
2, 33 %
...
```

**Step 2 — read pressure at the node level.**

```bash
$ cat /proc/pressure/cpu
some avg10=27.40 avg60=19.80 avg300=9.60 total=1990556331
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

27.4% of the last ten seconds had at least one task waiting for a core, on a node with 35% of its cores idle. **That contradiction is the finding**, and it is exactly the distribution effect §1 described: bursty threads wake in bunches and collide, then the machine idles between bursts. (Note the zero `full` line — system CPU `full` is hard-coded to zero; it is not evidence of health.)

**Step 3 — get a clean instantaneous rate from `total=`, not from `avg10`.**

`avg10` lags a step change by ~10 s. Diff the counter instead:

```bash
$ read_total() { awk '/^some/{sub("total=","",$5); print $5}' /proc/pressure/cpu; }
$ a=$(read_total); sleep 1; b=$(read_total); echo "stall µs in 1 s: $((b-a))"
stall µs in 1 s: 274118
```

274,118 µs of stall in 1,000,000 µs of wall clock = **27.4%** — agreeing with `avg10`, confirming the load is steady rather than a decaying spike. Remember from §4 that this counter is non-idle-weighted and wall-clock-bounded, so this division is legitimate; the same division on `cpu.stat`'s `throttled_usec` would not be, because that one aggregates across threads.

**Step 4 — attribute it to a cgroup.** Node-level pressure is useless for action. Walk the leaves:

```bash
$ for f in /sys/fs/cgroup/kubepods.slice/*/*/cpu.pressure \
           /sys/fs/cgroup/kubepods.slice/*/cpu.pressure; do
    printf '%-72s %s\n' "${f#/sys/fs/cgroup/kubepods.slice/}" \
      "$(awk '/^some/{print $2} /^full/{print "| " $2}' "$f" | tr '\n' ' ')"
  done | sort -t= -k2 -rn | head -5
kubepods-burstable.slice/kubepods-burstable-pod7b4e_…slice/cpu.pressure   avg10=64.20 | avg10=22.10
kubepods-pod3f2a_…slice/cpu.pressure                                      avg10=3.10  | avg10=0.40
kubepods-besteffort.slice/kubepods-besteffort-pod9a1c_…slice/cpu.pressure avg10=1.90  | avg10=0.00
```

One pod owns it. Resolve which:

```bash
$ crictl pods --name loader -o json | jq -r '.items[0].id' | \
    xargs -I{} crictl inspectp {} | jq -r '.info.config.metadata.name'
loader-7b4e2a91
```

**Step 5 — read the culprit's full picture, and separate contention from throttling.**

```bash
$ CG=/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod7b4e_2a91_4c03_8e15_6d9f0b2a7c34.slice
$ cat $CG/cpu.pressure
some avg10=64.20 avg60=58.90 avg300=41.30 total=884213991
full avg10=22.10 avg60=19.80 avg300=13.40 total=291002418

$ cat $CG/cpu.max
400000 100000                    # limits.cpu: 4
$ cat $CG/cpu.stat
usage_usec 118422000
user_usec 112010000
system_usec 6412000
nr_periods 4210
nr_throttled 61                  # 61/4210 = 1.4% of periods — NOT the story
throttled_usec 402118

$ cat $CG/io.pressure
some avg10=8.10 avg60=7.40 avg300=6.90 total=44192003
full avg10=2.20 avg60=2.10 avg300=1.90 total=11002118

$ cat $CG/memory.pressure
some avg10=0.00 avg60=0.00 avg300=0.01 total=310882
full avg10=0.00 avg60=0.00 avg300=0.00 total=108441
```

**Read this line by line.** `nr_throttled/nr_periods = 1.4%` — the pod is essentially *not* hitting its own `cpu.max`, so this is not the lesson-03 throttling paradox. `io.pressure` and `memory.pressure` are negligible, so it is not storage and not reclaim. But `cpu.pressure full avg10 = 22.1%`: for 22% of wall clock this pod has runnable threads and **zero** cores. Combined with a near-zero throttle rate, that leaves exactly one explanation — **the pod is losing the proportional-share competition on a contended node**. Confirm with weights ([lesson 03](03-cgroups-v2-and-k8s-enforcement.md) §3):

```bash
$ cat $CG/cpu.weight; cat ${CG%/*}/cpu.weight; cat /sys/fs/cgroup/kubepods.slice/cpu.weight
39            # this pod, from requests.cpu: 1
938           # the whole burstable tier
2383          # kubepods.slice
```

`requests.cpu: 1` against a 16-thread loader on a contended 64-core node. The request is a scheduling reservation *and* the runtime share; at weight 39 inside a busy tier, the loader gets a small slice of a contended core set. The 22% `full` is the price.

**Step 6 — price the finding.** This is what turns a graph into a decision:

```
  Given
    cpu.pressure full (loader)          = 22.1% of wall clock
    GPUs on the node                    = 8
    step time is loader-bound            (verified: GPU util 31%, no NCCL stalls)
    on-demand rate (dated snapshot)     = $2–3 per GPU-hour

  Lost GPU time per wall-clock hour
    = 8 GPUs × 1 h × 0.221
    = 1.77 GPU-hours per hour of operation

  Cost of the starvation, per node
    = 1.77 × $2   … $3         =  $3.54 – $5.31 per hour
    = × 24 × 365                =  $31,000 – $46,500 per node-year

  Cost of the fix
    = raising requests.cpu from 1 to 4 on ONE pod
    = 3 additional CPU-cores of scheduling reservation on a 64-core node
    ≈ 4.7% of the node's CPU allocatable

  Note WHY `full` is the right number to multiply by and `some` is not:
  `some avg10 = 64.2%` describes queueing, during much of which SOME loader
  thread was still producing batches. Only during `full` was the loader
  contributing nothing, so only `full` maps one-to-one onto GPU idle time.
  Multiplying by 64.2% would overstate the loss by roughly 3×.
```

**Step 7 — verify the fix with the same instrument.** After raising `requests.cpu` to `4` (weight 39 → 157 at the pod slice) and redeploying:

```bash
$ cat $CG/cpu.pressure
some avg10=11.30 avg60=14.20 avg300=33.10 total=901882440
full avg10=1.40 avg60=2.90 avg300=9.80 total=294118002
```

Read the *shape*, not just the value: `avg300` still carries the old regime (33% / 9.8%), `avg60` is mid-transition, `avg10` shows the new steady state. **The three windows disagreeing in that order is the signature of a recent change taking effect** — and it is why you keep all three rather than graphing only `avg10`.

**Step 8 — the memory analogue, which feeds the deliverable.** Reproduce the pre-OOM signature in a controlled cgroup:

```bash
$ sudo mkdir -p /sys/fs/cgroup/psi-demo
$ echo 256M | sudo tee /sys/fs/cgroup/psi-demo/memory.max
$ echo $$   | sudo tee /sys/fs/cgroup/psi-demo/cgroup.procs
$ stress-ng --vm 1 --vm-bytes 1G --vm-keep --timeout 60s &
$ watch -n1 cat /sys/fs/cgroup/psi-demo/memory.pressure
some avg10=64.20 avg60=41.10 avg300=15.30 total=44190882
full avg10=58.70 avg60=37.40 avg300=13.90 total=39882001
```

`full` near 59% means that for most of every second, **every** task in the cgroup was inside a memory-stall region — the workload is alive only because the kernel is frantically evicting and re-faulting pages. Note how close `full` is to `some` here (58.7 vs 64.2): with a single-process workload there is nobody left to be productive while the reclaimer runs, so the two converge. **A `full`/`some` ratio approaching 1 is itself diagnostic — it says the stalled tasks *are* the workload.** Cross-check against lesson 03's `memory.current` pinned at `memory.max` and lesson 05's rising `workingset_refault_anon`; all three are the same event seen by three subsystems.

## Practice

**Environment:** a Linux VM, laptop, or privileged container with a v2 cgroup tree (`stat -fc %T /sys/fs/cgroup` → `cgroup2fs`), `/proc/pressure/` present, `stress-ng`, `sysstat` (`mpstat`), a C compiler, and root. Verify PSI first — if `/proc/pressure/` is missing, fix that before anything else (§6).

1. **CPU pressure without CPU saturation.** Run `stress-ng --cpu <N> --cpu-load 50` with `N ≈ 1.5 ×` your core count. In parallel, capture `mpstat 1 10` and sample `/proc/pressure/cpu` every second. **Find a run where mean utilization stays below 80% while `cpu some avg10` exceeds 15%.** Record both. Tune `--cpu` and `--cpu-load` until you have it: the point is to produce the contradiction deliberately, not to stumble on it.

2. **Prove the `total=` rate matches `avg10`.** Sample the `some` line's `total=` twice, one second apart, and show that `Δtotal / 1e6` equals `avg10 / 100` within a few percent under steady load. Then create a *step* — start the hogs abruptly — and show the two disagreeing for ~10 seconds while the EWMA catches up. **This is the exercise that teaches you when to trust which field.**

3. **Per-cgroup attribution.** Create two sibling cgroups by hand, give them `cpu.weight` 100 and 1000, pin both to a single CPU via `cpuset.cpus`, and put an identical spinner in each. Read both `cpu.pressure` files. Show that the low-weight cgroup has high `some` *and* nonzero `full` while the high-weight one has neither, and that the **system-wide** file cannot distinguish them. This is the whole argument for leaf-cgroup attribution in one experiment.

4. **Memory pressure to thrash.** Create `/sys/fs/cgroup/psi-demo` with `memory.max=256M`, move a `stress-ng --vm 1 --vm-bytes 1G --vm-keep` into it, and record `memory.pressure` climbing. Note the moment `full` crosses 40%, and record the `full/some` ratio. Cross-check `memory.current`, `memory.events`, and (if available) `memory.stat`'s `workingset_refault_anon`.

5. **Build and run the trigger monitor.** Compile the C program from §7. Point it at your `psi-demo` cgroup's `memory.pressure` via `PSI_FILE`, start the memory hog, and measure the wall-clock delay between the hog starting and the first `POLLPRI` wakeup. Compare it to how long `avg10` takes to cross the same threshold. Then deliberately break it twice to see the error paths: write a second trigger to the same fd (expect `EBUSY`) and, as a non-root user, write a window that is not a multiple of 2 s (expect `EINVAL`).

6. **Latency probe under "idle" CPU.** While CPU pressure is high but utilization is under 80%, run a tiny probe — a loop that measures wall time to perform a fixed 1 ms of work, or `cyclictest` if available — and show that its latency inflates even though the node reports idle cores. This closes the loop from "pressure is high" to "a real workload felt it."

**Acceptance (feeds [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)):** a note of 8–15 lines presenting one cgroup or node that is **not** CPU-saturated by utilization (paste the `mpstat` average, under 80% busy) yet **is** pressured by PSI (paste the `/proc/pressure/cpu` or `*.pressure` line, `some` over 15%). It must state explicitly: **which resource** is pressured, **which cgroup** owns it, the **`total=` delta over 1 s** as rate evidence independent of the EWMA, whether `full` is nonzero and what that implies for lost throughput, and one sentence on the GPU-fleet failure mode it models (starved loader → idle GPU). If your `full` is zero, say so and explain what that rules out.

## Common pitfalls

1. **Treating `some` and `full` as the same signal with different sensitivities.** They answer different questions. `some` = "a queue formed" (contention, latency risk, no defined magnitude). `full` = "nothing ran" (a throughput multiplier you can put in a cost model). *Mechanism:* every `full` predicate contains its `some` predicate as a conjunct plus a "and nobody was productive" clause, so `full ≤ some` always. A dashboard with only one of them is blind to half the failure space — and, worse, multiplying `some` by your GPU cost overstates the loss, often by 3× or more.

2. **Asserting that `/proc/pressure/cpu` has no `full` line.** Since **Linux 5.13** it does, printed as a constant zero for backward compatibility because CPU-full is undefined system-wide. *Symptom of getting this wrong:* either a parser that breaks on modern kernels, or a dashboard graphing a flat zero that someone eventually cites as proof the CPU is healthy. The line you actually want is the **per-cgroup** `cpu.pressure` `full`, which is real and meaningful (`NR_RUNNING > 0 && !TSK_ONCPU`).

3. **Assuming PSI is present.** It needs `CONFIG_PSI=y`, and if the kernel was built with `CONFIG_PSI_DEFAULT_DISABLED=y` it also needs `psi=1` on the cmdline. Some vendor kernels return `ENOTSUP` on *read* rather than hiding the file, so a collector can report zeros forever without erroring. Since Linux 6.1, a cgroup can also have accounting switched off individually via `cgroup.pressure = 0`, non-hierarchically. **All-zero pressure on a busy node is a configuration hypothesis before it is a health finding.**

4. **Polling `avg10` when you need to react fast.** `avg10` is an EWMA with a 10-second time constant: 63% of a step change after 10 s, 95% after 30 s. If you need millisecond-scale response, use the trigger/`poll()` interface (§7), which is what `systemd-oomd` and LMKD use. If you need second-scale resolution for a graph, diff `total=`. Polling `avg10` on a 15-second scrape gives you a smoothed view of a smoothed view.

5. **Dividing `total=` by thread count, or comparing it to `throttled_usec`.** They are different quantities. `total=` is a **non-idle-weighted average across CPUs**, already normalized to wall clock, and cannot exceed 1 s per wall-second. `cpu.stat`'s `throttled_usec` is a **sum across threads** and routinely exceeds it (lesson 03 measured 3.5 s of stall per wall-second on a 4-thread container). Putting them on the same axis produces a graph that looks like a contradiction and is merely a units error.

6. **Reading node-level PSI and concluding "the node is overloaded, add capacity."** Node pressure averages a starving pod together with healthy neighbours and hides both. Always descend to the leaf cgroup. High pressure at a leaf can mean a noisy neighbour, a `cpu.weight` too low for the workload's thread count, a misconfigured `cpu.max`, or genuine capacity shortage — and those have four different fixes, only one of which is buying hardware.

7. **Expecting PSI to cover network.** There is no `/proc/pressure/net`. PSI covers CPU, memory, I/O, and optionally IRQ. A node dropping new connections because its conntrack table is full ([lesson 07](07-networking-datapath-conntrack.md)) shows a completely clean PSI. If PSI is your only saturation instrument, that class of incident is invisible to you.

8. **Assuming Kubernetes acts on PSI because it exports it.** As of v1.36, KEP-4205 delivers *metrics* — Summary API fields and the six `container_pressure_*_seconds_total` counters — and explicitly lists eviction and userspace OOM kills as **Non-Goals for future KEPs**. Node-pressure eviction still uses level signals (`memory.available`, `nodefs`, `imagefs`). Writing "the kubelet evicts on PSI" in a design doc is a checkable error.

9. **Getting the metric names backwards.** In cAdvisor's naming, `container_pressure_*_waiting_seconds_total` is **`some`** and `container_pressure_*_stalled_seconds_total` is **`full`**. "Stalled" sounds milder than "waiting" in ordinary English and means the opposite here. An alert built on the wrong one fires at the wrong severity.

## Self-check

**(a) What do `some` and `full` pressure mean, precisely — and what does each let you compute?**
**Answer:** `some` is the share of wall-clock time in which **at least one** task in the domain was stalled on the resource; `full` is the share in which **no task was productive** — for I/O, someone is blocked and `NR_RUNNING == 0`; for memory, every runnable task is itself a reclaimer (`NR_RUNNING == NR_MEMSTALL_RUNNING`); for CPU, tasks are runnable and none is on a CPU. `full ≤ some` always, because each `full` predicate contains its `some` predicate. Operationally, `some` is a contention/latency signal with **no defined magnitude of loss** — 10% `some` could be one thread of eight waiting (≈1.25% of capacity) or seven of eight waiting (nearly 9%). `full` is a **direct throughput multiplier**: 10% `full` means 10% of your wall clock produced nothing regardless of task count, so a job that would take 100 s takes 100/(1−0.10) ≈ 111 s, and on an 8-GPU node it means 0.8 GPU-hours lost per hour. Alert on `full`, investigate on `some`, and never multiply `some` by a cost.

**(b) How can CPU be under 100% utilized yet CPU-pressured?**
**Answer:** They measure different integrals. Utilization is busy-time over wall time; pressure is time tasks spent **runnable but off-CPU** (`NR_RUNNING > TSK_ONCPU`). Bursty threads wake in bunches: within a burst, twelve runnable threads collide on eight cores and eleven of them queue; between bursts the cores idle. Averaged over a second, utilization reports 63% and looks comfortable, while pressure reports 27% because a queue existed for 27% of the wall clock. It is a distribution effect that averaging destroys — the same class of error as lesson 03's throttling paradox, where a pod averaging 32% of its limit is frozen for 93.75 ms out of every 100 ms period. Utilization is bounded above by 100% and stops carrying information there; pressure keeps rising, which is why it is the saturation axis of the USE method ([lesson 09](09-perf-ftrace-use.md)).

**(c) Which PSI signal warns of memory thrash before an OOM kill, and what is the kernel actually counting?**
**Answer:** Rising **memory `full`** — per-cgroup `memory.pressure` or system `/proc/pressure/memory`. The kernel sets `MEM_SOME` whenever any task is inside a `psi_memstall_enter()`/`leave()` bracket: direct reclaim, refault wait, swap-in, compaction stall. It sets `MEM_FULL` when `NR_RUNNING == NR_MEMSTALL_RUNNING` — every runnable task is a reclaimer, so the CPU may be 100% busy while producing zero useful work. That fourth counter exists precisely because reclaimers *run*, so an "is anyone on-CPU?" test would call a thrashing cgroup productive. `systemd-oomd` thresholds exactly this signal: `DefaultMemoryPressureLimit=60%` on `full avg10`, sustained for `DefaultMemoryPressureDurationSec=30s`, then it kills the descendant cgroup with the most reclaim activity. Caveat for GPU nodes: with swap off and a footprint that is mostly anonymous and pinned memory, there is little to reclaim, so `full` may lead the kill by only seconds ([lesson 05](05-memory-and-oom.md)).

**(d) Why is CPU `full` undefined system-wide but meaningful per-cgroup — and what does the file actually show?**
**Answer:** The predicate is `NR_RUNNING > 0 && !TSK_ONCPU`: this domain has work ready and is getting no CPU. Inside a cgroup that is routine — your 16 loader threads are runnable while the core is executing another pod, because of `cpu.weight` competition, a `cpu.max` throttle, or `cpuset` pinning. System-wide the same condition would mean the machine has runnable tasks and every CPU is running nothing, which is a scheduler bug rather than contention; there is no coherent notion of "the entire system's CPU demand is being denied," because the system's CPU demand is by definition what is on the CPUs. As for the file: since **Linux 5.13**, `/proc/pressure/cpu` **prints a `full` line hard-coded to zero** for backward compatibility (`psi_show()` skips the real values for exactly the `psi_system` + `PSI_CPU` + `full` combination). On 4.20–5.12 the line is genuinely absent. Per-cgroup `cpu.pressure` reports a real `full` on all versions.

**(e) You read `total=1990556331` on `/proc/pressure/cpu`, and one second later `total=1990830449`. What do you know, and what would be wrong to conclude?**
**Answer:** The delta is 274,118 µs in 1,000,000 µs of wall clock, so **27.4% of that second had at least one task stalled on CPU**. This is a rate independent of the EWMA, so it is the right field for sub-10-second resolution and for detecting spikes too brief to move `avg10`. What would be wrong: (1) dividing by thread count or core count — `total=` is a **non-idle-weighted average across CPUs**, already normalized to wall clock, so it can never exceed 1 s per wall-second; (2) comparing it directly to `cpu.stat`'s `throttled_usec`, which is summed across threads and can exceed wall clock several-fold; (3) concluding anything about *magnitude of loss* — this is the `some` line, so during those 274 ms other tasks may well have been running productively. Read the `full` line for that. And (4) the derivation only holds for the system file or a cgroup file read consistently; the counter resets to zero when a cgroup is created, so a container restart looks like a counter reset — which is exactly why the Prometheus export is a counter and you use `rate()`.

**(f) On a Kubernetes v1.36 node, how do you get pod-level PSI into an alert, and what does Kubernetes itself do with it?**
**Answer:** PSI collection is stable (KEP-4205, gate `KubeletPSI`; alpha v1.33, beta v1.34, stable v1.36). The kubelet reads it from cgroup v2 through cAdvisor/`runc` and exposes it two ways: as `PSIStats{Some, Full}` with `Total/Avg10/Avg60/Avg300` inside `CPUStats`, `MemoryStats`, and a new node-level `IOStats` in the Summary API; and as six cAdvisor counters — `container_pressure_{cpu,memory,io}_{waiting,stalled}_seconds_total`, where **`waiting` is `some` and `stalled` is `full`**, converted from the kernel's microseconds to seconds. Because they are stall-second counters, `rate()` yields the pressure fraction directly, e.g. `100 * rate(container_pressure_memory_stalled_seconds_total[2m]) > 10`. What Kubernetes does *not* do: act on it. KEP-4205's Non-Goals defer pod eviction and userspace OOM kills to future KEPs, and node-pressure eviction still keys off level signals (`memory.available` from working set, `nodefs`, `imagefs`). So the platform measures; you supply the policy.

## Connections & what's next

PSI is the load-bearing diagnostic instrument for the rest of the module. It is the **saturation** axis of the USE method that [lesson 09](09-perf-ftrace-use.md) formalizes — the axis that utilization and error counters structurally cannot cover. It is the signal `systemd-oomd`-style tooling acts on *before* the kernel OOM killer of [lesson 05](05-memory-and-oom.md) ever fires. It is read out of the exact cgroup tree [lesson 03](03-cgroups-v2-and-k8s-enforcement.md) built, using the same leaf-attribution discipline you used there for cost. And it is the file you `cat` first — before `cpu.stat`, before `memory.events` — when a symptom appears and you do not yet know which resource is guilty.

Carry two things forward. The first is the diagnostic ordering: **PSI localizes the resource and the cgroup; the resource-specific counters explain the mechanism; off-CPU stacks name the code path.** The second is the boundary: PSI covers CPU, memory, I/O, and IRQ — and nothing else. When [lesson 07](07-networking-datapath-conntrack.md) shows you a node dropping new connections with a completely clean PSI readout, that gap is the point.

Next: **[05 · Memory management & the OOM killer](05-memory-and-oom.md)**, which follows the thread from "memory `full` is climbing" — this lesson's pre-thrash signature — to "here is the exact process the kernel killed and why": the reclaim path, `oom_badness()`, and reading a real `dmesg` OOM report cold.

## References & further reading

> **A note on verification.** This environment's egress proxy blocks `kernel.org`, `docs.kernel.org`, `kubernetes.io`, `lwn.net`, and most vendor blog domains. Everything marked **[verified against source]** below was read directly from the upstream Git repository via `raw.githubusercontent.com`. Entries marked **[not reachable]** are listed as further reading only — their substance is not relied on for any claim in this lesson.

**Primary sources**

1. **`Documentation/accounting/psi.rst`** (Linux kernel, authored by Johannes Weiner, dated April 2018) — https://docs.kernel.org/accounting/psi.html — **[verified against source]**, read from `torvalds/linux`. The authoritative spec: `some`/`full` definitions, the exact file format, the "CPU full is undefined at the system level, but has been reported since 5.13, so it is set to zero" statement, the trigger write format and window/privilege rules, and the reference `poll()` monitor program. **Correction recorded:** an earlier version of this lesson claimed `/proc/pressure/cpu` has no `full` line at all. That is true only for 4.20–5.12; from 5.13 the line is present and hard-coded to zero.
2. **`kernel/sched/psi.c` and `include/linux/psi_types.h`** — https://github.com/torvalds/linux/blob/master/kernel/sched/psi.c — **[verified against source]**. The ground truth for: the four task counters and `TSK_ONCPU`; `test_states()`'s six predicates; `collect_percpu_times()`'s non-idle weighting; `PSI_FREQ = 2*HZ+1`; the EWMA constants `EXP_10s = 1677`, `EXP_60s = 1981`, `EXP_300s = 2034` in 1/2048 fixed point; the `min(sample, period)` clamp; `psi_show()`'s system-CPU-full special case and the IRQ `only_full` path; and `psi_trigger_create()`'s validation (`WINDOW_MAX_US = 10000000`, `UPDATES_PER_WINDOW = 10`, the `CAP_SYS_RESOURCE` / 2 s rule, `EBUSY` on a second write).
3. **`init/Kconfig` — `CONFIG_PSI` and `CONFIG_PSI_DEFAULT_DISABLED`** — **[verified against source]**. The build-time dependency, the `psi=1` cmdline override, and the upstream statement of PSI's overhead (measurable in `hackbench`-style scheduler stress tests, not in ordinary server workloads).
4. **`Documentation/admin-guide/cgroup-v2.rst`** — https://docs.kernel.org/admin-guide/cgroup-v2.html — **[verified against source]**. `cgroup.pressure` (default `1`, non-hierarchical, exists to avoid PSI aggregation overhead in deep hierarchies) and `irq.pressure`. Also the companion reference for every other file [lesson 03](03-cgroups-v2-and-k8s-enforcement.md) uses.
5. **KEP-4205, "Expose PSI Metrics"** (kubernetes/enhancements, SIG-Node) — https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4205-psi-metric — **[verified against source]**, including `kep.yaml`. Feature gate `KubeletPSI`; alpha v1.33, beta v1.34, stable v1.36; status `implemented`. Source for the `PSIData`/`PSIStats` Go structs, the `CPUStats`/`MemoryStats`/`IOStats` extensions, the six metric names, and the explicit Non-Goal that eviction and userspace OOM kills are left to future KEPs.
6. **cAdvisor — `lib/metrics/prometheus.go`** — https://github.com/google/cadvisor — **[verified against source]**. The six `container_pressure_*_seconds_total` definitions with their help strings, proving `waiting` ← `Some.Total` and `stalled` ← `Full.Total`, and the `asMicrosecondsToSeconds` conversion that pins down the unit.
7. **`opencontainers/cgroups` — `fs2/psi.go` and `stats.go`** — https://github.com/opencontainers/cgroups — **[verified against source]**. How runtimes actually parse the file, and the three reasons a missing `*.pressure` file is treated as "no stats": kernel < 4.20, `CONFIG_PSI` unset, or `cgroup.pressure = 0` (kernel ≥ 6.1). Also the note that some vendor kernels return `ENOTSUP` on read when `psi=1` was required.
8. **`systemd` — `oomd.conf(5)` source** (`man/oomd.conf.xml`) — https://github.com/systemd/systemd — **[verified against source]**. `DefaultMemoryPressureLimit=60%` measured on `full avg10`, `DefaultMemoryPressureDurationSec=30s`, `SwapUsedLimit=90%`, the 5%-of-swap eligibility rule, and the `kill-all`/`kill-by-pgscan`/`kill-by-swap` strategies.

**Real-world engineering**

9. **Meta Engineering — "Open-sourcing oomd, a new approach to handling OOMs"** — https://engineering.fb.com/2018/07/19/production-engineering/oomd/ — **[not reachable from this environment]**. Listed for depth: the production daemon that motivated PSI, written by the same team. Its argument — the kernel killer runs only after an allocation has already failed, and picks victims by page count — is also summarized in [lesson 05](05-memory-and-oom.md) from the same reasoning.
10. **Kubernetes documentation and release notes for PSI metrics** — https://kubernetes.io/docs/ — **[not reachable from this environment]**; every Kubernetes claim in this lesson is sourced from KEP-4205 and the cAdvisor source instead, both verified above. Check the release notes for your own cluster version before relying on gate defaults.

**Deeper dives**

11. **Facebook PSI microsite** — https://facebookmicrosites.github.io/psi/ — **[not reachable from this environment]**. The origin team's explainer, with the "utilization lies, pressure doesn't" framing and the production motivation behind `oomd`.
12. **Brendan Gregg, *Systems Performance* (2nd ed.), methodology chapters** — https://www.brendangregg.com/systems-performance-2nd-edition-book.html — places PSI as the **saturation** axis of the USE method, which [lesson 09](09-perf-ftrace-use.md) formalizes into the grid you actually run on an unfamiliar node.
