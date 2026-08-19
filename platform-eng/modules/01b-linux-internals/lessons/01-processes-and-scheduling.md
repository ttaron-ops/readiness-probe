---
lesson: "01b.1"
title: "Processes and Scheduling"
module: "01b"
concept: "Processes and Scheduling"
status: not-started
est_time: "6h"
prev: null
next: "02-namespaces.md"
artifacts: []
sources: 15
---

# 01b.1 · Processes and Scheduling

> **Concept.** A process is a set of kernel states, not a "running program" — and load average, the scheduler, and the D-state are what you actually reason about when a GPU node looks busy but the CPUs are idle.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

This is lesson 1 of module 01b, so there's no prior lesson to recap — but there is a prior *mode*. The module README frames the whole module as a move from **using** Linux to **understanding** the kernel mechanisms underneath, because at GPU-fleet scale the failures that actually cost money live below the container runtime, where `top` and `kubectl describe` stop explaining anything.

This lesson is where that shift starts. It replaces "a process is a running program" with "a process is a `task_struct` driven through a kernel state machine," and it replaces "load average means busy" with the exact sampled quantity, sampling interval, and decay constants the kernel maintains. Everything downstream depends on this substrate. The next lesson, **[02 — Namespaces](02-namespaces.md)**, takes the same `task_struct` and asks what happens when you virtualize the *views* a process has of PIDs, mounts, and the network — namespaces don't exist without processes to put inside them, so the state-machine model you build here is the foundation the container abstraction sits on. Lesson **[03 — cgroups v2](03-cgroups-v2-and-k8s-enforcement.md)** then takes the scheduler weights and bandwidth limits introduced here and shows exactly which Kubernetes YAML field writes each one.

## Why this matters

In an interview for a GPU-fleet role, "load average is 80 but CPU is 3%, what's wrong?" is a filter question. The wrong answer ("the CPUs are saturated") marks you as an operator who never looked below `top`. On real fleets the answer is usually storage: NFS-backed datasets, a hung Ceph mount, or a wedged NVMe controller parking training jobs in uninterruptible sleep, where they inflate load, resist `kill -9`, and block node drains.

The module README's company signals make this concrete: CoreWeave's Kernel Systems Engineer posting asks for "kernel-level observability … kernel readiness for production workloads"; Datadog's Sr SWE role sits "at the intersection of eBPF, the Linux kernel, and GPU infrastructure … investigating production incidents"; NVIDIA's Base OS/Kernel role expects debugging multiprocessor systems with GDB, kdump, and eBPF. All three assume you can read `/proc`, a state letter, and a `wchan` without reaching for a restart.

There is a second, more expensive reason. On a distributed-training job, a synchronous collective (an all-reduce) completes only when its slowest participant arrives. A single CPU-side thread — the data loader, the NCCL proxy thread — delayed on a run queue delays *every* GPU in the job, not just its own. At 2026 list prices a single high-end training GPU is commonly several dollars per hour and an 8-GPU node runs into the tens of dollars per hour (treat any such figure as a dated snapshot, not a constant; check your own contract). Run-queue latency measured in milliseconds, multiplied across a job's steps, is a directly measurable cost line. Knowing exactly what puts a task in D, what load average counts, and how the scheduler picks the next task is the difference between "the node is stuck" and "PID 24601 is blocked in `nfs_readpage`, the storage path is the incident."

## What's new here (calibration)

The module README's skip-list applies from lesson 1 onward: you already know shell/pipes/permissions, package managers, distro tours, beginner systemd units, vim, bash scripting, and general networking basics — none of that is re-taught here. What *is* new, even for an experienced operator:

- The exact kernel state machine behind the `ps` state letters — including the bitmask values, why `TASK_IDLE` exists, and why `TASK_KILLABLE` makes some NFS mounts killable and others not.
- Load average as a literal fixed-point EWMA over `nr_running + nr_uninterruptible`, with the real sampling interval (`5*HZ + 1` jiffies) and the real decay constants (1884 / 2014 / 2037 in 1/2048 fixed point), plus the exact kernel condition that decides whether a sleeping task is counted.
- **EEVDF** — the fair-class pick-next algorithm since Linux 6.6 — expressed the way the kernel expresses it: `lag_i = w_i · (V − v_i)`, eligibility as `v_i ≤ V`, and `vd_i = ve_i + r_i / w_i`. Most engineers still describe this with CFS-only vocabulary and get the mechanism wrong.
- **RT throttling.** The common claim that a runaway `SCHED_FIFO` thread starves a core *indefinitely* is wrong on a default kernel: `sched_rt_runtime_us`/`sched_rt_period_us` cap the RT classes at 95% of each 1-second window. Knowing the exception (and how to disable it) is the senior version of the answer.
- GPU-fleet-specific tuning most generalist SREs never touch: the `isolcpus=` flag list (`domain`, `nohz`, `managed_irq`), `nohz_full=`, the workqueue `cpumask` that people forget, and IRQ affinity.
- `sched_ext` (merged in Linux 6.12): scheduling policy as a loadable BPF program, and why a hyperscaler reached for it.

## Core concepts

### 1. What a process actually is

The kernel has no object called "a program." It has `struct task_struct` — one per **schedulable entity**, which means one per *thread*, not one per process. Everything you think of as a process is a group of `task_struct`s that share a thread group ID (TGID) and a memory descriptor (`mm_struct`).

```
                 struct task_struct  (one per THREAD)
  ┌──────────────────────────────────────────────────────────────┐
  │ pid          — kernel-wide unique thread id ("TID")          │
  │ tgid         — thread-group id; == pid for the group leader  │
  │              — this is what userspace calls "the PID"        │
  │ __state      — TASK_RUNNING / TASK_INTERRUPTIBLE / …         │
  │ se           — struct sched_entity: vruntime, deadline,      │
  │                slice, load.weight   ← the scheduler's view   │
  │ sched_class  — which scheduling class owns this task         │
  │ prio, static_prio, normal_prio, rt_priority                  │
  │ sched_contributes_to_load  ← decides load-average counting   │
  │ mm           — address space (shared across a thread group)  │
  │ files, fs    — fd table, cwd/root  (shared or copied)        │
  │ nsproxy      — pointer table to this task's namespaces  → L2 │
  │ cgroups      — css_set: which cgroup in each hierarchy  → L3 │
  │ signal, sighand, pending  — signal state                     │
  └──────────────────────────────────────────────────────────────┘
```

Two of those fields are the entire subject of the next two lessons. `nsproxy` is the namespace pointer table ([02](02-namespaces.md)); `cgroups` is the control-group membership ([03](03-cgroups-v2-and-k8s-enforcement.md)). **A "container" is not a new kind of `task_struct`. It is an ordinary `task_struct` whose `nsproxy` and `cgroups` pointers happen to point somewhere private.** Hold on to that; it is the module's whole thesis, and it is visible right here in lesson 1.

`/proc/<pid>/` is a rendering of one `task_struct`. `/proc/<pid>/task/<tid>/` is the per-thread view. `ps` shows thread groups by default; `ps -L` (or `ps -eLo`) shows threads.

### 2. The task state machine

`__state` is a bitmask, not an enum. The values below are from `include/linux/sched.h` in the current kernel tree:

| Constant | Value | Meaning |
|---|---|---|
| `TASK_RUNNING` | `0x0000` | Running on a CPU, or on a run queue waiting for one |
| `TASK_INTERRUPTIBLE` | `0x0001` | Sleeping; a signal will wake it |
| `TASK_UNINTERRUPTIBLE` | `0x0002` | Sleeping; signals are **not** delivered |
| `__TASK_STOPPED` | `0x0004` | Stopped by `SIGSTOP`/`SIGTSTP` |
| `__TASK_TRACED` | `0x0008` | Stopped by a ptracer |
| `EXIT_DEAD` | `0x0010` | Being reaped |
| `EXIT_ZOMBIE` | `0x0020` | Exited; parent has not `wait()`ed |
| `TASK_PARKED` | `0x0040` | Parked kthread |
| `TASK_DEAD` | `0x0080` | Final teardown |
| `TASK_WAKEKILL` | `0x0100` | Modifier: fatal signals may wake this sleep |
| `TASK_NOLOAD` | `0x0400` | Modifier: **do not count in load average** |
| `TASK_FROZEN` | `0x8000` | Frozen by the cgroup/system freezer |

Two of those are compositions, and they are the ones that matter operationally:

```c
#define TASK_KILLABLE   (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)   /* 0x0102 */
#define TASK_IDLE       (TASK_UNINTERRUPTIBLE | TASK_NOLOAD)     /* 0x0402 */
```

`fs/proc/array.c` maps the reportable subset to the letters you see in `ps`:

| Letter | Kernel state | Notes |
|---|---|---|
| `R` | `TASK_RUNNING` | On-CPU **or** runnable-and-queued — `ps` does not distinguish |
| `S` | `TASK_INTERRUPTIBLE` | Normal idle waiting; the vast majority of tasks |
| `D` | `TASK_UNINTERRUPTIBLE` | "disk sleep" — signals queued, not delivered |
| `T` | `TASK_STOPPED` | `SIGSTOP`/`SIGTSTP` |
| `t` | `TASK_TRACED` | ptrace stop |
| `X` | `EXIT_DEAD` | Transient; you will rarely catch it |
| `Z` | `EXIT_ZOMBIE` | Exited, unreaped — holds a PID and nothing else |
| `P` | `TASK_PARKED` | Parked kernel thread |
| `I` | `TASK_IDLE` | Idle kernel thread — **excluded from load average** |

Here is the state machine as the scheduler drives it:

```
                     ┌──────────────────────────────┐
       fork/clone    │                              │  exit()
   ──────────────▶  TASK_RUNNING (queued, "R")      ├──────────▶ EXIT_ZOMBIE ("Z")
                     │        ▲              │      │                 │
       schedule()    │        │              │      │                 │ parent wait()
       picks it ─────┼────▶ ON-CPU ("R")     │      │                 ▼
                     │        │   ▲          │      │            EXIT_DEAD ("X")
                     │ preempt│   │wake_up() │      │
                     │  (slice│   │          │      │
                     │  or VD)│   │          │      │
                     └────────┘   │          │      │
                                  │          │      │
        blocks on an event ───────┼──────────┘      │
        (socket, timer, futex)    │                 │
                                  │                 ▼
                        TASK_INTERRUPTIBLE ("S") ◀──┘
                          │  ▲          signal OR event → wake
                          │  └────────────────────────────────
                          │
        blocks in kernel I/O path (bio submit, NFS RPC, page fault
        on a slow device, lock held across I/O)
                          │
                          ▼
                TASK_UNINTERRUPTIBLE ("D")
                    │            ▲
                    │            └── signals are QUEUED in task->pending
                    │                but never DELIVERED here
                    │
                    └── I/O completion / timeout → wake_up() → TASK_RUNNING

        TASK_KILLABLE = TASK_UNINTERRUPTIBLE | TASK_WAKEKILL
              ▲ same "D" letter, but a FATAL signal *can* break it out
```

**Why D exists at all.** When a task submits I/O it typically holds kernel-side state that must be unwound in a specific order: a page locked for writeback, a filesystem transaction open, an RPC slot reserved, a device queue entry in flight. If a signal could yank the task out of that sleep at an arbitrary point, the kernel would have to teach every one of those paths how to safely abort and restart — an enormous, error-prone surface. `TASK_UNINTERRUPTIBLE` is the kernel saying "this sleep is short and bounded; do not make me handle interruption here." The pathology is that "short and bounded" stops being true when the device or the server on the other end of the wire stops answering.

**Why `kill -9` does nothing.** `SIGKILL` is not magic; it is a bit set in `task->pending`, followed by an attempt to wake the target. `signal_wake_up()` only actually wakes a sleeper whose state includes `TASK_WAKEKILL` (for fatal signals) or `TASK_INTERRUPTIBLE` (for any). A plain `TASK_UNINTERRUPTIBLE` sleeper matches neither, so the signal sits in `pending` and is delivered the instant the task next returns toward userspace — which happens only after the I/O completes or times out. `ps` still shows the process. `kill -9` "silently failed" is a misreading: it queued perfectly and is waiting its turn.

**`TASK_KILLABLE` is the compromise.** Introduced so that long-latency network filesystems could still be escaped, it is `TASK_UNINTERRUPTIBLE | TASK_WAKEKILL`: ordinary signals are still ignored, but a fatal one breaks the sleep. Modern NFS uses killable sleeps in most of its RPC wait paths, which is why `kill -9` on an NFS-hung process often *does* work on a current kernel while the folklore says it never does. Both experiences are real; they are different code paths. It still shows as `D` in `ps`, because `TASK_WAKEKILL` is a modifier bit outside `TASK_REPORT`.

**`TASK_IDLE` is why your box's load isn't 200.** A modern kernel runs a large number of worker kthreads (`kworker/*`, `nfsd`, `xfs-*`) that sit in uninterruptible sleep waiting for work — semantically idle, but historically counted as load. `TASK_IDLE = TASK_UNINTERRUPTIBLE | TASK_NOLOAD` gives them the safety of uninterruptible sleep with an explicit "don't count me" bit. They show as `I` in `ps` and contribute zero to load average.

### 3. Load average: the exact formula

Load average is **not** CPU utilization, and it is not a percentage of anything. It is a fixed-point exponentially-weighted moving average of the count of tasks that are runnable *or* in non-idle uninterruptible sleep.

The counting rule is one condition in `kernel/sched/core.c`, in the dequeue path:

```c
p->sched_contributes_to_load =
        (task_state & TASK_UNINTERRUPTIBLE) &&
        !(task_state & TASK_NOLOAD) &&
        !(task_state & TASK_FROZEN);
```

So a sleeping task counts toward load if and only if it is uninterruptible, not `TASK_IDLE`, and not frozen. Every runnable task counts via `rq->nr_running`. Read that literally: **load average is `nr_running + nr_uninterruptible`, smoothed.**

The smoothing constants live in `include/linux/sched/loadavg.h`:

| Symbol | Value | Meaning |
|---|---|---|
| `FSHIFT` | `11` | Fixed-point fractional bits |
| `FIXED_1` | `2048` (`1<<11`) | The fixed-point representation of 1.0 |
| `LOAD_FREQ` | `5*HZ + 1` | Sampling interval — 5 seconds plus one tick |
| `EXP_1` | `1884` | `1/exp(5s/60s)` in 1/2048 units ≈ 0.9199 |
| `EXP_5` | `2014` | `1/exp(5s/300s)` ≈ 0.9834 |
| `EXP_15` | `2037` | `1/exp(5s/900s)` ≈ 0.9946 |

and the update is the two-line `calc_load()`:

```
  new_avg = (old_avg × EXP_n + active × (FIXED_1 − EXP_n)) / FIXED_1
```

which is the standard `a₁ = a₀·e + a·(1−e)` EWMA in integer arithmetic. The extra `+1` in `LOAD_FREQ` is a deliberate anti-aliasing trick: sampling at exactly `5*HZ` would phase-lock with periodic workloads that also run on second boundaries.

Here is what that means as a timeline. Suppose 41 tasks pile into D-state at t=0 on an otherwise-idle box, and stay there:

```
  sample #     0     1     2     3     4     6     8    12    24    ∞
  wall time   0s    5s   10s   15s   20s   30s   40s   60s  120s
  active      0    41    41    41    41    41    41    41    41    41
             ───────────────────────────────────────────────────────
  1-min avg  0.00  3.28  6.30  9.08 11.63 16.10 19.79 25.24 34.34 41.0
  5-min avg  0.00  0.68  1.36  2.03  2.68  3.97  5.22  7.62 13.31 41.0
 15-min avg  0.00  0.23  0.46  0.68  0.91  1.35  1.79  2.65  4.98 41.0
```

Two consequences you can act on. **First, load average lags.** After the first 5-second sample the 1-minute figure has only moved to 3.28 — 8% of the way to the true value. A load-average alert is structurally 30–60 seconds late. **Second, the three numbers are a direction indicator.** `41.9 40.2 33.7` means 1-min > 5-min > 15-min: the condition is growing. `12.0 30.1 38.0` means it is receding. That ordering is often more informative than the absolute values.

`/proc/loadavg` renders `avenrun[]` back to decimal:

```
$ cat /proc/loadavg
0.52 0.58 0.59 2/1834 44127
 │    │    │   │ │     └── PID of the most recently created process
 │    │    │   │ └──────── total number of tasks (threads) that exist
 │    │    │   └────────── tasks currently in state R (runnable *right now*)
 └────┴────┴────────────── 1 / 5 / 15-minute EWMA of (nr_running + nr_uninterruptible)
```

**The single most useful diagnostic in this file is the mismatch between field 4's numerator and the three floats.** `41.9 40.2 33.7 1/2203` says: the smoothed count is ~42, but only 1 task is runnable *now*. The other ~41 are therefore in D. That one line has already localized the incident to the I/O path before you have run anything else.

Two failure modes fall straight out of the definition:

- **Load high, CPU ~0%.** Dozens of tasks in D on a hung mount. They burn no CPU and are counted anyway. This is the classic NFS/Ceph signature and the interview question.
- **Load low, box in pain.** One pegged thread on a 128-core node is load ≈ 1.0 — trivially under core count, easy to wave away, while that thread's owner is fully stalled. Load average has no notion of how many cores you have.

Because both hold, **load average is not a saturation metric.** For actual saturation use PSI (`/proc/pressure/{cpu,io,memory}`), which measures the wall-clock time work spent *waiting* for a resource. PSI gets its own lesson — [04 — PSI](04-psi.md) — and this lesson uses it only as a confirming instrument.

### 4. Scheduling classes: who gets asked first

`schedule()` does not consult one algorithm. It walks an ordered list of **scheduling classes**, asking each for a runnable task, and takes the first one that answers. The order is fixed at link time by the section layout in `include/asm-generic/vmlinux.lds.h`:

```
  __sched_class_highest
        │
        ├─ stop_sched_class    CPU hotplug, stop_machine(). Not user-reachable.
        ├─ dl_sched_class      SCHED_DEADLINE  (EDF + constant-bandwidth server)
        ├─ rt_sched_class      SCHED_FIFO, SCHED_RR   (prio 1..99)
        ├─ fair_sched_class    SCHED_NORMAL/OTHER, SCHED_BATCH  ← EEVDF lives here
        ├─ ext_sched_class     sched_ext — BPF-defined policy (Linux 6.12+)
        └─ idle_sched_class    SCHED_IDLE + the per-CPU swapper
  __sched_class_lowest
```

`pick_next_task()` walks this top-down. **A single runnable `SCHED_FIFO` task beats every fair-class task on that CPU, full stop** — no weighting, no fairness, no time-slicing against them. That is the entire point of the RT classes and it is also their hazard.

| Policy | Class | Selection rule | Set with |
|---|---|---|---|
| `SCHED_DEADLINE` | dl | Earliest absolute deadline first, admission-controlled | `sched_setattr(2)` |
| `SCHED_FIFO` | rt | Highest static prio (1–99); runs until it blocks or yields | `chrt -f <prio>` |
| `SCHED_RR` | rt | Same, but round-robins equal-prio peers per `sched_rr_timeslice_ms` | `chrt -r <prio>` |
| `SCHED_NORMAL` | fair | EEVDF (below) | default; `nice` |
| `SCHED_BATCH` | fair | EEVDF, but never treated as latency-sensitive | `chrt -b 0` |
| `SCHED_IDLE` | idle | Runs only when nothing else is runnable | `chrt -i 0` |

**RT throttling — the correction most people miss.** The folklore is "a spinning `SCHED_FIFO` thread wedges the core forever." On a default kernel that is false, and knowing why is the senior answer. The RT group scheduler enforces a global bandwidth cap:

```
/proc/sys/kernel/sched_rt_period_us    = 1000000    (1 second)
/proc/sys/kernel/sched_rt_runtime_us   =  950000    (0.95 second)
```

RT-class tasks may consume at most 950 ms of every 1 s window; the remaining 50 ms is reserved so that fair-class tasks — including your shell and your monitoring agent — still make progress. So a runaway `chrt -f 99` busy-loop degrades the node to 5% fair-class throughput rather than 0%. That is catastrophic for latency and *survivable* for recovery, which is exactly why the knob exists. Setting `sched_rt_runtime_us = -1` disables the cap entirely — some latency-critical audio/industrial setups do this deliberately, and on those nodes the folklore becomes true. Check it before you assert either behaviour:

```
$ cat /proc/sys/kernel/sched_rt_runtime_us
950000                    # capped at 95%  → runaway RT is survivable
$ cat /proc/sys/kernel/sched_rt_period_us
1000000
```

`SCHED_DEADLINE` is the safer real-time tool because it is admission-controlled: `sched_setattr()` rejects a new deadline task if total utilization `Σ(runtime/period)` would exceed what the CPUs can guarantee. You cannot accidentally oversubscribe it the way you can with FIFO.

### 5. EEVDF: how the fair class picks next

**The problem the fair scheduler solves.** Given N runnable tasks with weights, hand out CPU time in proportion to weight, over any reasonable window, while still giving short interactive tasks low wake-up-to-run latency. Those two goals fight: perfect proportional fairness wants long slices (low overhead, exact ratios); low latency wants short slices and frequent preemption.

**Weights come from nice.** `nice` is not a priority number the scheduler compares; it is an index into a multiplicative weight table in `kernel/sched/core.c`. Each nice level is a factor of ~1.25:

| nice | −20 | −15 | −10 | −5 | **0** | 5 | 10 | 15 | 19 |
|---|---|---|---|---|---|---|---|---|---|
| weight | 88761 | 29154 | 9548 | 3121 | **1024** | 335 | 110 | 36 | 15 |

The table is built so one nice level ≈ ±10% CPU relative to a peer. Two nice-0 tasks split a CPU 50/50 (`1024:1024`). A nice-0 and a nice-5 task split it `1024:335` ≈ 75/25. The cgroup `cpu.weight` knob you will meet in [lesson 03](03-cgroups-v2-and-k8s-enforcement.md) is the same mechanism applied to a *group* of tasks instead of one task.

**CFS, and what it got wrong.** Until Linux 6.6, the fair class was the Completely Fair Scheduler. It tracked per-task **vruntime** — wall time on CPU, scaled inversely by weight, so a nice-0 task accrues vruntime at 1× real time and a nice-−20 task at `1024/88761` ≈ 1/87× — and always ran the task with the smallest vruntime, from a red-black tree keyed on vruntime. That achieves proportional fairness well. It handles latency badly: CFS had no way for a task to say "I only need 200 µs, but I need it *soon*." The heuristics that approximated this (`sched_wakeup_granularity`, `GENTLE_FAIR_SLEEPERS`, wakeup buddies) were a pile of tunables that behaved differently on every workload.

**EEVDF replaced CFS's pick-next logic in Linux 6.6.** "Earliest Eligible Virtual Deadline First" comes from a 1995 paper; Peter Zijlstra's kernel implementation landed in 6.6 (kernel docs: `Documentation/scheduler/sched-eevdf.rst`). It keeps vruntime, keeps the weight table, keeps the red-black tree, keeps `nice`, keeps cgroup `cpu.weight`, and keeps CFS bandwidth control (`cpu.max`). What changes is the *selection rule*, and it adds two concepts.

**Concept 1: lag, and eligibility.** Define virtual time `V` as the **weighted average vruntime of all runnable entities**:

```
        Σ (v_i · w_i)
  V  =  ─────────────
            Σ w_i
```

`V` is the vruntime a task would have if service had been perfectly proportional up to now. Each task's **lag** is its signed deviation from that ideal, measured in real service units:

```
  lag_i  =  S − s_i  =  w_i · (V − v_i)
```

Positive lag means the task has received *less* than its fair share — it is owed CPU. Negative lag means it has received more. Fair schedulers conserve lag: `Σ lag_i = 0` by construction, because `V` is defined as the weighted mean.

A task is **eligible** exactly when `lag_i ≥ 0`, which (since weights are positive) is exactly `v_i ≤ V`. EEVDF only ever picks from eligible tasks. This is the structural fix for the problem CFS solved with heuristics: a task that just overran cannot immediately be picked again, no matter how attractive its deadline; it must wait for `V` to advance past its vruntime.

**Concept 2: the virtual deadline.** Every entity has a requested time slice `r_i` (`se->slice`). Its virtual deadline is:

```
  vd_i  =  ve_i + r_i / w_i
```

where `ve_i` is its eligible time. In the kernel this is `se->deadline = se->vruntime + calc_delta_fair(se->slice, se)` — the slice converted into virtual-time units by dividing by weight. Among all *eligible* entities, EEVDF runs the one with the **earliest** `vd_i`.

Read the formula for what it buys you: a **smaller requested slice yields an earlier deadline**, so a task that asks for less time at once gets picked sooner — *without changing its long-run CPU share*, because share is governed by weight and eligibility, not by deadline. Latency and throughput become independent dials. That was impossible under CFS.

The default slice is `sysctl_sched_base_slice`, 700,000 ns (0.70 ms) at the reference scale, scaled by `(1 + ilog2(ncpus))` on larger machines. A task can request its own via `sched_setattr(2)` with `sched_runtime` set on a `SCHED_NORMAL` policy — the kernel stores it in `se->slice` and flags `se->custom_slice`.

```
  EEVDF pick-next on one run queue
  ════════════════════════════════

  virtual time ──────────────────────────────────────────────▶

                          V = Σ(v·w)/Σw
                          (weighted mean vruntime)
                              ║
      ELIGIBLE (lag ≥ 0)      ║      INELIGIBLE (lag < 0)
      ◀─────────────────────  ║  ──────────────────────────▶
                              ║
   task  v_i    w_i    lag    ║    virtual deadline vd = v + slice/w
   ────  ────   ────   ────   ║    ────────────────────────────
    A    100    1024   +2048  ║ ├────────┤ vd=A+0.68  ◀── EARLIEST
    B    101    1024   +1024  ║   ├────────┤ vd=B+0.68     eligible
    C     99     335   +1005  ║ ├──────────────────┤ vd=C+2.09   deadline
    D    104    1024   −2048  ║        (not eligible — must wait for V)
                              ║
                     V = 102  ║
                              ║
   → pick A: eligible AND earliest virtual deadline.
   → D just overran; it stays parked until V advances past v_D = 104.
   → C has a light weight (nice 5), so the same slice buys it a LATER
     deadline: slice/w is bigger when w is smaller.
```

**Lag across sleeps: deferred dequeue.** If lag were simply discarded when a task sleeps, any task could reset a negative lag by sleeping for a microsecond — a trivial way to steal CPU. The current kernel handles this with **deferred dequeue**: a sleeping task stays on the run queue marked for delayed removal, so its lag continues to decay against advancing virtual time. Long sleeps therefore end with lag near zero (neither owed nor owing); short sleeps do not launder a negative lag. `Documentation/scheduler/sched-eevdf.rst` notes lag handling is still an area of active development, so treat the decay details as version-dependent while the `lag ≥ 0` eligibility rule is stable.

**What this changes for you as a platform engineer: essentially nothing in the interfaces, and a lot in the reasoning.** `nice`, `cpu.weight`, `cpu.max`, `chrt`, and `taskset` all mean exactly what they meant. But when you are asked "why did this latency-sensitive thread get scheduled late," the answer is now a specific two-part claim you can check — either it was **ineligible** (it had negative lag; it had recently overrun) or it was eligible but **outranked on deadline** by something with a shorter slice.

### 6. sched_ext: when the default scheduler is the problem

Merged in Linux 6.12, `sched_ext` adds a scheduling class (`ext_sched_class`, between fair and idle in the class order) whose policy is supplied by a **BPF program loaded from userspace**. You can swap the entire pick-next algorithm at runtime, and if the BPF scheduler misbehaves or stalls, a watchdog ejects it and the kernel falls back to the built-in fair class. The cgroup v2 docs already account for it: `cpu.weight` "affects processes under the fair-class scheduler *and* a BPF scheduler with the `cgroup_set_weight` callback" — meaning a `sched_ext` policy can honour your Kubernetes CPU requests, or ignore them, depending on how it is written.

This matters as a career-shaped fact, not just a trivium. Meta reported hitting a latency regression on their ads-ranking service after a kernel upgrade moved them onto EEVDF, and rather than fight the default's tuning knobs they built and shipped a workload-specific BPF scheduler on `sched_ext`, reporting a 28% cut in ads-retrieval p99 latency and 3.28 MW of fleet power saved. The staff-level move was to make the scheduler programmable rather than to keep tuning it.

### 7. Dedicating cores: `isolcpus=`, `nohz_full=`, IRQ affinity

**The problem.** A GPU node's NCCL proxy/polling thread busy-spins on completion queues to shave microseconds off collective latency. On an ordinary fair-scheduled core, that thread shares the run queue with everything else the kernel puts there: other tasks, softirqs, RCU callbacks, unbound workqueue items, and the periodic scheduler tick. Each of those is microseconds. Occasionally one is milliseconds. A millisecond of jitter in a proxy thread is a millisecond added to a synchronous all-reduce, which is a millisecond added to every rank in the job.

There are three independent sources of disturbance, and they need three different fixes. This is the part people get wrong — they set `isolcpus=` alone and are surprised the jitter is still there.

```
  A "dedicated" core needs all three lanes closed
  ═══════════════════════════════════════════════

  CPU 40  ──────────────────────────────────────────────────────▶ time

  (1) OTHER TASKS placed here by the load balancer
      fix: isolcpus=domain,40-47   (remove from sched domains)
           or cpuset partition     (dynamic; preferred, reversible)
      ────┬──────────┬──────────────┬──────────────────────────
          ▼          ▼              ▼
        [other]   [other]        [other]        ← eliminated

  (2) PERIODIC TICK — the scheduler tick fires even with 1 task
      fix: nohz_full=40-47  (or isolcpus=nohz,…)
      ─┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───
       ▼   ▼   ▼   ▼   ▼   ▼   ▼   ▼   ▼   ▼   ▼   ▼   ▼   ▼
      tick tick tick …  (CONFIG_HZ=1000 → every 1 ms)  ← reduced
      to a residual ~1 Hz tick, which is offloaded to workqueues
      → you MUST also pin the workqueue mask, see (3b)

  (3a) HARDWARE IRQs — NIC/NVMe interrupts land here
       fix: /proc/irq/<n>/smp_affinity, exclude from irqbalance,
            isolcpus=managed_irq,… for kernel-managed queue IRQs
       ──────────▲────────────────▲──────────────────────────
              NIC rx           NVMe cq                ← redirected

  (3b) UNBOUND WORKQUEUES — default to *all* CPUs
       fix: echo <housekeeping mask> > \
              /sys/devices/virtual/workqueue/cpumask
       ────────────────▲──────────────────────────────────────
                   kworker/u                            ← redirected

  Result: the NCCL proxy thread owns the core.
```

The `isolcpus=` parameter takes an optional **flag list** before the CPU list (`isolcpus=[flag-list,]<cpu-list>`, default flag `domain`), documented in `Documentation/admin-guide/kernel-parameters.txt`:

| Flag | Effect |
|---|---|
| `domain` | Remove the CPUs from general SMP balancing and scheduler domains. Tasks land there only via explicit affinity (`taskset`, `sched_setaffinity`, cgroup `cpuset`). **Irreversible at runtime** — the kernel docs point you at cpusets for a dynamic equivalent. |
| `nohz` | Equivalent to `nohz_full=` for those CPUs: stop the tick when a single task runs, offload RCU callbacks. Leaves a residual 1 Hz tick offloaded to workqueues. |
| `managed_irq` | Keep kernel-managed device-queue interrupts (whose affinity you cannot set via `/proc/irq/*`) off these CPUs, best-effort, when the queue mask also contains housekeeping CPUs. |

Two traps worth memorising because they are exactly what an interviewer probes:

1. **`nohz_full` alone does not silence the core.** It leaves a ~1 Hz residual tick that is offloaded to *unbound workqueues*, and the global workqueue mask defaults to all CPUs. You have to write a housekeeping mask to `/sys/devices/virtual/workqueue/cpumask` after boot, or use `isolcpus=domain,…` as well so the workqueue infrastructure knows to avoid them.
2. **`isolcpus=` is a boot parameter and is not reversible.** The kernel docs explicitly recommend **cpuset partitions** for anything you might want to change: `cpuset.cpus.partition` with an isolated partition gives the same domain isolation, at runtime, per-cgroup. On a Kubernetes node this is the same machinery the CPU Manager `static` policy drives — covered in [lesson 03](03-cgroups-v2-and-k8s-enforcement.md).

### 8. Where the numbers live

Everything above is readable from `/proc`. These are the files and the exact fields.

| File | Format | The fields that matter |
|---|---|---|
| `/proc/loadavg` | one line | `1m 5m 15m runnable/total lastpid` |
| `/proc/<pid>/stat` | space-separated, positional | **3** = state letter · **14** = utime (ticks) · **15** = stime · **19** = nice · **20** = num_threads · **22** = starttime · **39** = task_cpu · **40** = rt_priority · **41** = policy · **42** = blkio_ticks |
| `/proc/<pid>/status` | `Key:\tvalue` | `State:`, `Threads:`, `SigPnd:`/`SigBlk:`/`SigIgn:`/`SigCgt:` (signal bitmaps), `voluntary_ctxt_switches`, `nonvoluntary_ctxt_switches` |
| `/proc/<pid>/wchan` | one symbol, no newline | The kernel function the task is sleeping in |
| `/proc/<pid>/schedstat` | three integers | `sum_exec_runtime_ns  run_delay_ns  pcount` |
| `/proc/<pid>/sched` | key/value block | Richer version of the above, incl. `se.slice`, `se.vruntime` |
| `/proc/pressure/{cpu,io,memory}` | PSI | `some`/`full` × `avg10/avg60/avg300/total` → [lesson 04](04-psi.md) |

**`/proc/<pid>/schedstat` is the run-queue-latency evidence and almost nobody uses it.** Field 2, `run_delay`, is the cumulative nanoseconds this task spent **runnable but not on a CPU**. Field 3, `pcount`, is how many times it was scheduled. Divide them:

```
$ cat /proc/7712/schedstat
4182993214 918273645 39182
│          │         └── pcount: 39,182 times scheduled
│          └──────────── run_delay: 918,273,645 ns waiting on a run queue
└─────────────────────── sum_exec_runtime: 4,182,993,214 ns actually on CPU

  mean run-queue wait = 918,273,645 / 39,182 ≈ 23,436 ns ≈ 23 µs per wakeup
  on-CPU : waiting    = 4.18 s : 0.92 s → 18% of its "active" time was queueing
```

23 µs mean wake-up latency is fine for a web server and a disaster for a 10 µs-budget polling thread. Sampling this file before and after a change is the cheapest possible proof that CPU isolation worked.

**`voluntary` vs `nonvoluntary_ctxt_switches` in `/proc/<pid>/status`** separates "I chose to sleep" from "I was preempted." A thread with thousands of *nonvoluntary* switches per second is fighting for CPU. A thread parked in D shows almost none of either — it is not competing, it is waiting.

## Perspectives

**Kernel-mechanism view.** A process is a `task_struct` whose `__state` bitmask the kernel drives through a small state machine, and whose `sched_entity` carries the scheduler's sort keys. Nothing about "priority" or "fairness" is a human concept applied after the fact: `nice` is a table index producing a weight, `lag_i = w_i·(V − v_i)` is an arithmetic eligibility gate, and `vd_i = ve_i + r_i/w_i` is the literal comparison key of a red-black tree. Understanding the mechanism is what turns "the process is stuck" into a falsifiable claim about a specific state and a specific `wchan`.

**Operator/SRE view.** The recurring ticket is "the node feels wedged" — vague, urgent, and usually wrong about the cause. What resolves it fast is a fixed sequence, not a guess: read `/proc/loadavg` for the R/D split (the numerator of field 4 versus the three floats), `ps -eo stat` to separate runnable from uninterruptible, `wchan` on the D-state tasks to name the exact kernel path they are blocked in, `/proc/<pid>/schedstat` for run-queue latency on the tasks that *are* runnable, then `/proc/pressure/{cpu,io}` to confirm which resource is genuinely saturated. Rebooting skips all five and guarantees the same incident next week.

**GPU-fleet-specific view.** Noisy-neighbour CPU contention is a distributed-training tax that is easy to miss, because the GPU looks idle-adjacent rather than broken: a data-loader thread or the NCCL proxy thread gets bumped off-CPU by an unrelated fair-class task sharing the core, and the whole training step stalls waiting on that one thread. The GPU is fully capable of working; nothing is feeding it. This is exactly the case for core dedication — and, as the diagram above shows, for doing all three parts of it (scheduler domains, tick, interrupts and workqueues), because closing two of three lanes leaves the jitter roughly where it was.

**Economics/failure-mode view.** An idle-but-D-state GPU node is still burning its full reserved-instance or GPU-hour cost — the accelerators sit fully billed while every worker thread waits on a hung mount. And run-queue latency taxes p99 step time directly, because a synchronous collective completes only when its slowest participant arrives; one straggler delayed on a run queue delays the entire job. That is why `run_delay` in `schedstat` is a cost metric, not a curiosity: it converts, via step time and GPU-hour price, into dollars.

## Real-world use cases

- **Meta Engineering — "Modernizing the Meta Ads Service With an Open-Source Kernel Scheduler."** Meta hit a latency regression on their ads-ranking service after a kernel upgrade moved them from CFS to EEVDF. Rather than chase the default's tuning knobs, they built and shipped `sched_ext` — a BPF-programmable scheduler class, developed alongside the authors of Google's ghOSt — and ran a workload-specific policy, reporting a **28% reduction in ads-retrieval p99 latency and 3.28 MW of fleet power saved**. `sched_ext` was merged into mainline Linux in **6.12**. *What it shows:* when the default scheduler does not fit a workload, the staff-level response is to make the scheduler itself programmable, not to keep re-tuning the default — and that response is now available to you in an upstream kernel.
- **Netflix TechBlog — "Noisy Neighbor Detection with eBPF."** Netflix instruments scheduler tracepoints (`sched_switch`, `sched_wakeup`) with eBPF to measure per-container run-queue latency continuously in production, at a per-hook overhead they measured in the hundreds of nanoseconds, so it can run fleet-wide rather than as an incident-time tool. *What it shows:* "run-queue latency ≠ CPU utilization" is not academic — it is how a large fleet finds noisy neighbours before customers do. The quantity they are collecting is precisely `run_delay` from this lesson's `schedstat`, sampled continuously instead of by hand. Forward-links to [08 — eBPF](08-ebpf.md) and [09 — perf/ftrace/USE method](09-perf-ftrace-use.md).
- **Brendan Gregg — "Linux Load Averages: Solving the Mystery."** The canonical archaeology of why Linux load counts R+D while other Unixes count CPU-runnable only: the behaviour traces to a 1993 patch by Matthias Urlichs that added uninterruptible tasks so that load would reflect *demand for resources* rather than demand for CPU alone. *What it shows:* the "load high, CPU idle" surprise is a deliberate 30-year-old design decision, not a bug or an accident — which is why it keeps producing incidents and keeps appearing in interviews.

## Worked example

A node reports load 42 on 16 cores; `top` shows CPUs 96% idle. Trace it end to end. *(Transcript is representative — a reconstruction of the standard signature, not a captured session; the arithmetic is exact.)*

**Step 1 — the load file already localizes it.**

```
$ nproc
16
$ cat /proc/loadavg
41.9 40.2 33.7 1/2203 88510
```

Read the fields: 1-min 41.9 > 5-min 40.2 > 15-min 33.7, so this is **growing**, not receding. Field 4 numerator is **1** — exactly one task is runnable right now. Load average counts `nr_running + nr_uninterruptible`, so `41.9 ≈ 1 + nr_uninterruptible` ⟹ roughly **41 tasks are sitting in D**. No other command has been run yet and the incident is already narrowed to the I/O path.

**Step 2 — name the kernel path.**

```
$ ps -eo pid,stat,wchan:32,comm | awk '$2 ~ /^D/'
  7712 D    nfs_wait_on_request      dd
  7713 D    nfs_wait_on_request      dd
  7714 D    rpc_wait_bit_killable    dd
  ... (41 rows, all wchan nfs_* / rpc_*)
```

`rpc_wait_bit_killable` is the interesting one: that is a `TASK_KILLABLE` sleep, so those particular tasks *can* be killed with `SIGKILL` even though `ps` shows `D`. `nfs_wait_on_request` is a plain uninterruptible wait and cannot. Both letters are `D`; the behaviour differs.

**Step 3 — prove they are not computing.**

```
$ awk '{print "state="$3, "utime="$14, "stime="$15, "nonvol_blkio_ticks="$42}' /proc/7712/stat
state=D utime=1 stime=6 nonvol_blkio_ticks=41883
```

7 clock ticks of CPU total. At the usual `USER_HZ = 100`, that is **70 ms of CPU consumed over the task's entire life**, while field 42 (`blkio_ticks`, delay-accounted block-I/O wait) is 41,883 ticks ≈ **419 seconds spent waiting on I/O**. Ratio ≈ 6000:1. This task is a waiter, not a worker.

```
$ grep -E 'State|ctxt' /proc/7712/status
State:  D (disk sleep)
voluntary_ctxt_switches:        4
nonvoluntary_ctxt_switches:     0
```

Zero *nonvoluntary* switches: it was never preempted, because it never competed. Four voluntary ones: it went to sleep four times and has not come back from the fourth.

**Step 4 — arithmetic check that the story is consistent.** 41 tasks in D, 1 runnable, EWMA converged ⟹ steady-state load ≈ 42. On 16 cores, if those tasks were actually *running* you would see ~100% CPU. Instead:

```
$ mpstat 1 3 | tail -1
Average:  all  1.20  0.00  2.30  84.10  0.00  0.10  0.00  0.00  0.00  12.30
                │           │     │                                     └── %idle
                │           │     └── %iowait  = 84.1
                │           └── %sys = 2.3
                └── %usr = 1.2
```

96%+ of CPU time is not-user-not-system. `%iowait` is *idle time during which at least one task on that CPU was blocked on I/O* — it is a flavour of idle, not a flavour of busy, which is exactly why it can be 84% while the cores do nothing.

**Step 5 — confirm the resource with PSI, not with load.**

```
$ cat /proc/pressure/io
some avg10=99.20 avg60=98.71 avg300=71.40 total=884213991002
full avg10=97.55 avg60=96.02 avg300=68.31 total=861002418773
$ cat /proc/pressure/cpu
some avg10=0.31 avg60=0.28 avg300=0.30 total=1290034567
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

`io full avg10=97.55` means that for 97.5% of the last 10 seconds, **no productive work happened at all** — every non-idle task was blocked on I/O. Meanwhile CPU pressure is 0.31%: there is no CPU contention whatsoever. (`cpu full` reads zero at the system level by design; [lesson 04](04-psi.md) explains why.)

**Verdict.** An NFS server, or the network path to it, is slow or hung. The fix is on the storage path, not the node's CPU, and adding cores would change nothing. `kill -9 7712` will not clear the `nfs_wait_on_request` waiters until the I/O returns or the RPC times out; the `rpc_wait_bit_killable` ones will die immediately. Drains will hang because the kubelet cannot terminate D-state tasks either.

**The one-sentence version, which is what the interview wants:** *"Load 42 with one runnable task means 41 tasks in uninterruptible sleep; their `wchan` is in the NFS RPC path, they have consumed 70 ms of CPU between them, and `io.pressure full` is 97% — this is a storage incident presenting as a compute symptom."*

## Practice

On a laptop or VM, manufacture and read a D-state task, then measure run-queue latency. This directly feeds the module deliverable, [Anatomy of a Container](../practice/anatomy-of-a-container/README.md) — the diagnostic reflex you build here (state → wchan → schedstat → PSI) is what the toolkit's later steps assume you already have.

1. **Create sustained uninterruptible I/O.** Direct I/O against a large file is the most reliable path on a laptop:
   ```bash
   fallocate -l 4G /var/tmp/blob
   for i in 1 2 3 4 5 6; do
     ( while :; do dd if=/var/tmp/blob of=/dev/null bs=1M iflag=direct status=none; done ) &
   done
   ```
   On fast NVMe the D-windows are brief, which is why you run several readers — you need enough concurrent waiters to catch one. If you can point `dd` at a networked or deliberately throttled mount, the effect is far more pronounced.

2. **Catch a task in D and read its state four ways.**
   ```bash
   ps -eo pid,stat,wchan:32,comm | awk '$2 ~ /^D/'
   PID=<one of them>
   grep State /proc/$PID/status
   awk '{print $3}' /proc/$PID/stat
   cat /proc/$PID/wchan; echo
   cat /proc/$PID/schedstat
   ```
   Record the `wchan` symbol. It is the difference between "it's in D" and "it's in D *in the direct-I/O submission path*."

3. **Correlate load with states, and watch the lag.** Sample every 5 seconds for a minute:
   ```bash
   for i in $(seq 12); do
     printf '%s  R=%s D=%s\n' "$(cut -d' ' -f1-4 /proc/loadavg)" \
       "$(ps -eo stat --no-headers | grep -c '^R')" \
       "$(ps -eo stat --no-headers | grep -c '^D')"
     sleep 5
   done
   ```
   You should see the 1-minute figure climb toward `R + D` over roughly a minute rather than jumping — that is `EXP_1 = 1884/2048` doing its work. Compare your observed climb against the convergence table in Core concepts §3.

4. **Prove `kill -9` is impotent against a non-killable D.** `kill -9 $PID` a task currently in D and watch it survive until the I/O completes. Then check `SigPnd` in `/proc/$PID/status` — you will see the `SIGKILL` bit (`0x0000000000000100`) set and *pending*, which is the direct evidence that the signal was queued rather than lost.

5. **Measure run-queue latency, then remove it.** This is the `isolcpus=` idea without rebooting:
   ```bash
   # baseline: a spinner alone on core 0
   taskset -c 0 bash -c 'while :; do :; done' &
   SPIN=$!
   sleep 5; cat /proc/$SPIN/schedstat        # note run_delay and pcount

   # now add contention on the same core
   for i in 1 2 3 4; do taskset -c 0 bash -c 'while :; do :; done' & done
   sleep 5; cat /proc/$SPIN/schedstat        # run_delay should climb sharply
   ```
   Compute mean wait = Δrun_delay / Δpcount for each phase. The ratio between them is the jitter that `isolcpus=`/cpuset partitions exist to eliminate.

**Acceptance:** a 5–8 sentence note recording (i) an observed `/proc/loadavg` line with the R and D counts at that moment, (ii) the `wchan` of one D-state task, (iii) `/proc/pressure/{cpu,io}` readings at the same instant, (iv) the mean run-queue wait in nanoseconds for your spinner before and after contention, and (v) one sentence explaining why load ≠ CPU busy in your case — e.g. *"load 6.1 with 0 runnable and 6 in D on `blk_*`/`dio` wchan; io `full` 90%+, cpu `some` 0.4% → load is counting I/O waiters, and the mean run-queue wait went from 1.2 µs to 340 µs once four spinners shared core 0."* Clean up: `kill $(jobs -p); rm /var/tmp/blob`.

## Common pitfalls

1. **Reading load average as "percent CPU busy."** It is not a percentage of anything, and it has no knowledge of your core count. It is an EWMA-smoothed *count* of tasks in R plus non-idle D. Load 42 on a 16-core box is entirely consistent with 96% idle CPU when 41 of those tasks are parked in D. *Mechanism:* `sched_contributes_to_load` counts uninterruptible sleepers explicitly.
2. **Alerting on load average and expecting it to be timely.** The 1-minute figure reaches only ~8% of a step change after the first 5-second sample and takes about a minute to converge. *Mechanism:* `EXP_1 = 1884/2048 ≈ 0.92` per 5-second sample. If you need fast detection, alert on PSI (`avg10`, or better the `total` counter's rate) instead.
3. **Assuming `kill -9` always works — or that it never does on D.** Both halves are wrong. A plain `TASK_UNINTERRUPTIBLE` sleeper cannot be woken by any signal, but `TASK_KILLABLE` (`TASK_UNINTERRUPTIBLE | TASK_WAKEKILL`), which modern NFS uses in most RPC waits, breaks on a fatal signal. Both display as `D`. *Diagnostic:* read `wchan` — a `*_killable` symbol tells you which one you have.
4. **Claiming a runaway `SCHED_FIFO` thread starves a core indefinitely.** On a default kernel it does not: `sched_rt_runtime_us=950000` / `sched_rt_period_us=1000000` caps all RT classes at 95% of each second. The node becomes horrible, not dead. *Check first:* `cat /proc/sys/kernel/sched_rt_runtime_us` — a value of `-1` means the cap is off and the folklore applies.
5. **Setting `nohz_full=` and declaring the core dedicated.** It stops the periodic tick but leaves a residual ~1 Hz tick offloaded to unbound workqueues, whose global mask defaults to every CPU. Without also writing a housekeeping mask to `/sys/devices/virtual/workqueue/cpumask` (and handling IRQ affinity), the jitter you were chasing is still there.
6. **Confusing weight with a cap.** `nice` — and its cgroup counterpart `cpu.weight` — changes a task's *share* of CPU **under contention**. It never limits how much CPU a task can use when nothing else wants it. Only a bandwidth cap (`cpu.max`, quota/period) does that. This confusion resurfaces almost verbatim in [03 — cgroups v2](03-cgroups-v2-and-k8s-enforcement.md), where `requests` → `cpu.weight` (share) and `limits` → `cpu.max` (cap) are the exact same distinction.
7. **Assuming EEVDF changed the interfaces you tune.** It did not. `nice`, `cpu.weight`, `cpu.max` bandwidth control, `chrt`, and `taskset` all behave as before; only the pick-next rule changed (lowest-vruntime → eligible-and-earliest-virtual-deadline). What *is* new is `sched_setattr()`'s `sched_runtime` as a slice request for `SCHED_NORMAL` tasks.

## Self-check

- **Why can load average exceed core count while CPU is ~0%?**
  **Answer:** Load average counts tasks in state R *and* in non-idle uninterruptible sleep (D), sampled every `5*HZ + 1` jiffies and smoothed by a fixed-point EWMA with constants `EXP_1=1884`, `EXP_5=2014`, `EXP_15=2037` over `FIXED_1=2048`. The counting rule is literally `(state & TASK_UNINTERRUPTIBLE) && !(state & TASK_NOLOAD) && !(state & TASK_FROZEN)`. Tasks blocked in D on slow or hung I/O consume no CPU but are counted in every sample; enough of them (a hung NFS/Ceph mount parking dozens of readers) drives load far above the core count while the CPUs sit idle. Load average also has no knowledge of how many cores exist, so it is not a utilization ratio at all. The confirming reads are field 4 of `/proc/loadavg` (runnable *right now*, which will be tiny) and `/proc/pressure/io`.

- **What are vruntime, lag, and the virtual deadline — and what exactly did EEVDF change versus CFS?**
  **Answer:** *vruntime* (`v_i`) is per-task virtual runtime: wall time on CPU divided by the task's weight, where weight comes from the nice table (nice 0 = 1024, each level ≈ ×1.25). CFS (pre-6.6) always ran the runnable task with the lowest vruntime. EEVDF (6.6+) keeps vruntime and adds two things. **Virtual time** `V = Σ(v_i·w_i)/Σw_i` — the weighted mean vruntime — and **lag** `lag_i = w_i·(V − v_i)`, each task's signed deviation from perfect proportional service; `Σ lag_i = 0` by construction. A task is **eligible** only when `lag_i ≥ 0`, i.e. `v_i ≤ V`, so a task that just overran must wait for `V` to catch up. Among eligible tasks, EEVDF picks the earliest **virtual deadline** `vd_i = ve_i + r_i/w_i`, where `r_i` is the requested slice (`sysctl_sched_base_slice`, default 700,000 ns, or a custom value via `sched_setattr(2)`'s `sched_runtime`). Because a smaller slice yields an earlier deadline while share is still governed by weight, latency and throughput become independent dials — which was CFS's structural weakness. All external interfaces (nice, `cpu.weight`, `cpu.max`, `chrt`, `taskset`) are unchanged.

- **What puts a task in D versus S, and why can't you kill a D-state process — except when you can?**
  **Answer:** `S` (`TASK_INTERRUPTIBLE`) is a task blocked on an event the kernel allows a signal to interrupt — normal waiting. `D` (`TASK_UNINTERRUPTIBLE`) is a task blocked in a kernel path holding state that cannot be safely unwound mid-flight — typically I/O completion, an RPC in progress, or a lock held across I/O — so the kernel deliberately shields it from signal delivery. `SIGKILL` sent to such a task is set in `task->pending` and the wake attempt fails to match its state, so it is delivered only after the sleep ends; `ps` keeps showing the process and `SigPnd` in `/proc/<pid>/status` shows the pending bit. The exception is `TASK_KILLABLE = TASK_UNINTERRUPTIBLE | TASK_WAKEKILL`, used by modern NFS in most RPC waits, where a *fatal* signal does break the sleep. Both display as `D`; read `wchan` to tell them apart. Separately, `TASK_IDLE = TASK_UNINTERRUPTIBLE | TASK_NOLOAD` is an uninterruptible sleep deliberately excluded from load average, shown as `I`.

- **Why would a GPU-fleet operator dedicate CPU cores instead of just adding more cores — and what are the three things that must be done?**
  **Answer:** Adding cores does not remove jitter from a shared run queue; the scheduler can still place another task on the core between two polling iterations, and the periodic tick, interrupts, and unbound workqueue items fire on an otherwise-idle core regardless. Three independent disturbance sources need three fixes: (1) **other tasks** — `isolcpus=domain,<list>` removes the CPUs from SMP balancing and scheduler domains so tasks land there only via explicit affinity, or, preferably because it is runtime-changeable, an isolated `cpuset.cpus.partition`; (2) **the tick** — `nohz_full=<list>` (or `isolcpus=nohz,…`) stops the periodic scheduler tick when only one task is runnable, leaving a residual ~1 Hz tick that is offloaded to workqueues; (3) **interrupts and workqueues** — steer device IRQs away via `/proc/irq/<n>/smp_affinity` and exclusion from `irqbalance`, use `isolcpus=managed_irq,…` for kernel-managed queue interrupts, and write a housekeeping mask to `/sys/devices/virtual/workqueue/cpumask` so the offloaded work does not land back on the isolated cores. Doing only (1) and (2) is the common mistake, and it leaves the tail latency roughly where it was. Verify with `/proc/<pid>/schedstat` field 2 (`run_delay`) before and after.

- **Why does a misconfigured `SCHED_FIFO` thread threaten an entire node, when a runaway `SCHED_NORMAL` process doesn't — and what limits the damage?**
  **Answer:** `pick_next_task()` walks scheduling classes in a fixed link-time order — stop, deadline, rt, fair, ext, idle — and takes the first runnable task it finds. A single `SCHED_FIFO` task therefore preempts every fair-class task on that CPU unconditionally, with no weighting and no time-slicing against them; if it never blocks or yields, the fair class simply never gets asked. A runaway `SCHED_NORMAL` process, by contrast, still competes through weight, eligibility, and virtual deadlines, so it can slow its peers but not starve them. What limits the damage is **RT throttling**: `sched_rt_runtime_us` (default 950000) out of `sched_rt_period_us` (default 1000000) caps all RT-class execution at 95% of each second, reserving 5% for everything else so you can still SSH in and fix it. Setting `sched_rt_runtime_us = -1` removes the cap. `SCHED_DEADLINE` is the safer real-time option because `sched_setattr()` performs admission control and refuses to accept a task whose `runtime/period` would oversubscribe the CPUs.

- **What does `/proc/<pid>/schedstat` tell you, and how do you turn it into a number you can act on?**
  **Answer:** Three integers: `sum_exec_runtime` (nanoseconds actually on CPU), `run_delay` (nanoseconds spent runnable-but-queued), and `pcount` (number of times the task was scheduled onto a CPU). `run_delay / pcount` is the **mean run-queue latency per wakeup** — the direct measure of how long this thread waits between becoming runnable and actually running. `run_delay / (run_delay + sum_exec_runtime)` is the fraction of its active life spent queueing. This is the quantity CPU isolation exists to reduce and the quantity Netflix collects fleet-wide with eBPF scheduler tracepoints. Sample the file twice and difference it; a single reading is cumulative since task start and tells you about history rather than about now.

## Connections & what's next

This lesson's state machine and scheduler model are the substrate for the rest of the module. [03 — cgroups v2](03-cgroups-v2-and-k8s-enforcement.md) reuses `cpu.weight`/`cpu.max` — the same share-versus-cap distinction flagged in the pitfalls, now enforced per-cgroup rather than per-nice-value, and mapped field by field onto Kubernetes YAML. [04 — PSI](04-psi.md) formalises the `/proc/pressure` instrument this lesson used only as a confirming check, and explains the `full`-line asymmetry you saw in the worked example. [05 — memory & the OOM killer](05-memory-and-oom.md) picks up the D-state theme from the memory-reclaim side. [08 — eBPF](08-ebpf.md) and [09 — perf/ftrace/USE method](09-perf-ftrace-use.md) pick up the `sched_switch`/off-CPU instrumentation that the Netflix case previews — the automated, continuous version of hand-reading `schedstat`.

The immediate next step is **[02 — Namespaces](02-namespaces.md)**: it takes the same process — the `task_struct` you now know is a state machine with a scheduler entry — and follows one field, `nsproxy`, to ask what it means to give that process a private *view* of PIDs, mounts, and the network. That is the other half of "what a container really is."

## References & further reading

**Primary sources**

- **EEVDF Scheduler — kernel documentation** (`Documentation/scheduler/sched-eevdf.rst`) — https://docs.kernel.org/scheduler/sched-eevdf.html — the terse canonical description of lag, eligibility, virtual deadlines, deferred dequeue, and the `sched_setattr()` slice request. Confirms the transition began in 6.6 and that lag management is still evolving.
- **CFS Scheduler design — kernel documentation** (`Documentation/scheduler/sched-design-CFS.rst`) — https://docs.kernel.org/scheduler/sched-design-CFS.html — the vruntime model EEVDF inherited, plus the scheduling-class architecture and the group-scheduling extension that becomes `cpu.weight`.
- **CFS Bandwidth Control** (`Documentation/scheduler/sched-bwc.rst`) — https://docs.kernel.org/scheduler/sched-bwc.html — quota/period semantics, the 5 ms `sched_cfs_bandwidth_slice_us` transfer granularity, and the `cpu.stat` counters. Read before [lesson 03](03-cgroups-v2-and-k8s-enforcement.md).
- **Real-Time group scheduling** (`Documentation/scheduler/sched-rt-group.rst`) — https://docs.kernel.org/scheduler/sched-rt-group.html — source for `sched_rt_period_us = 1000000` and `sched_rt_runtime_us = 950000`, i.e. the 95% RT cap that corrects the "FIFO starves the node forever" folklore.
- **kernel-parameters.txt** — https://docs.kernel.org/admin-guide/kernel-parameters.html — authoritative text for `isolcpus=[flag-list,]<cpu-list>` with the `domain` / `nohz` / `managed_irq` flags, for `nohz_full=`, and for the workqueue-cpumask caveat.
- **CPU Isolation** (`Documentation/admin-guide/cpu-isolation.rst`) — https://docs.kernel.org/admin-guide/cpu-isolation.html — the housekeeping-CPU model behind all three isolation lanes, and why cpuset partitions are the recommended dynamic alternative to `isolcpus=`.
- **proc(5) filesystem documentation** (`Documentation/filesystems/proc.rst`) — https://docs.kernel.org/filesystems/proc.html — Table 1-4 is the positional field list for `/proc/<pid>/stat` used in this lesson (state = 3, utime = 14, stime = 15, nice = 19, starttime = 22, policy = 41, blkio_ticks = 42).
- **sched(7)** — https://man7.org/linux/man-pages/man7/sched.7.html — userspace-facing summary of the policies, priority ranges, and `sched_setattr(2)`; the companion to the kernel-side docs above.

**Real-world engineering blogs**

- **Meta Engineering — "Modernizing the Meta Ads Service With an Open-Source Kernel Scheduler"** — https://engineering.fb.com/2026/07/13/ml-applications/modernizing-the-meta-ads-service-with-an-open-source-kernel-scheduler/ — *what it shows:* the staff-level response to a post-EEVDF latency regression was to build `sched_ext` (BPF-programmable scheduling, merged in Linux 6.12) and run a workload-specific policy; 28% p99 reduction, 3.28 MW saved.
- **Netflix TechBlog — "Noisy Neighbor Detection with eBPF"** — https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd — *what it shows:* continuous, fleet-wide collection of exactly the run-queue-latency quantity that `/proc/<pid>/schedstat` field 2 exposes, via `sched_switch`/`sched_wakeup` tracepoints at sub-microsecond per-hook cost.
- **Brendan Gregg — "Linux Load Averages: Solving the Mystery"** — https://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html — *what it shows:* the 1993 patch that made Linux load count uninterruptible tasks, the reasoning behind it, and production war stories of the resulting "load high, CPU idle" confusion.

**Deeper dives**

- **Systems Performance, 2nd ed. — CPUs chapter (Brendan Gregg)** — https://www.brendangregg.com/systems-performance-2nd-edition-book.html — the authoritative treatment of run-queue latency, scheduler observability, and the USE method vocabulary senior interviewers expect.
- **LWN — "An EEVDF CPU scheduler for Linux"** — https://lwn.net/Articles/925371/ — the clearest narrative of why CFS's latency heuristics were replaced and how eligibility and deadlines interact; cited as reference [4] by the kernel's own EEVDF doc.
- **LWN — "Completing the EEVDF scheduler"** — https://lwn.net/Articles/969062/ — the follow-up work on lag handling and deferred dequeue; cited as reference [3] by the kernel's EEVDF doc, and the source for the "sleeping does not launder negative lag" behaviour described above.
- **LWN — "sched/fair: Fix low cpu usage with high throttling…"** — https://lwn.net/Articles/792268/ — the per-CPU quota-slice expiry bug (commit `512ac999`, fixed by `de53fd7a` in 5.4) seen from the scheduling-mechanism side; cross-links with [03 — cgroups v2](03-cgroups-v2-and-k8s-enforcement.md), which covers the same commit from the production-throttling angle.
