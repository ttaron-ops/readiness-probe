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
sources: 8
---

# 01b.1 · Processes and Scheduling

> **Concept.** A process is a set of kernel states, not a "running program" — and load average, the scheduler, and the D-state are what you actually reason about when a GPU node looks busy but the CPUs are idle.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits
This is lesson 1 of module 01b, so there's no prior lesson to recap — but there is a prior *mode*. The module README frames the whole module as a move from **using** Linux to **understanding** the kernel mechanisms underneath, because at GPU-fleet scale the failures that actually cost money live below the container runtime, where `top` and `kubectl describe` stop explaining anything. This lesson is where that shift starts: it replaces "a process is a running program" with "a process is a `task_struct` driven through a kernel state machine," and replaces "load average means busy" with the exact sampled quantity the kernel maintains. Everything downstream depends on this substrate. The next lesson, **[02 — Namespaces](02-namespaces.md)**, takes the same `task_struct` and asks what happens when you virtualize the *views* a process has of PIDs, mounts, and the network — namespaces don't exist without processes to put inside them, so the state-machine model you build here is the foundation the container abstraction sits on.

## Why this matters
In an interview for a GPU-fleet role, "load average is 80 but CPU is 3%, what's wrong?" is a filter question — the wrong answer (CPUs are saturated) marks you as an operator who never looked below `top`. On real fleets the answer is usually storage: NFS-backed datasets, a hung Ceph mount, or a wedged NVMe controller parking training jobs in uninterruptible sleep, where they inflate load, resist `kill -9`, and block drains. The module README's own company signals make this concrete: CoreWeave's Kernel Systems Engineer posting asks for "kernel-level observability … kernel readiness for production workloads"; Datadog's Sr SWE role sits "at the intersection of eBPF, the Linux kernel, and GPU infrastructure … investigating production incidents"; NVIDIA's Base OS/Kernel role expects debugging multiprocessor systems with GDB, kdump, and eBPF. All three assume you can read `/proc`, a state letter, and a wchan without reaching for a restart. Knowing exactly what puts a task in D, what load average counts, and how the scheduler picks the next task is the difference between "the node is stuck" and "PID 24601 is blocked in `nfs_readpage`, the storage path is the incident."

## What's new here (calibration)
The module README's skip-list applies from lesson 1 onward: you already know shell/pipes/permissions, package managers, distro tours, beginner systemd units, vim, bash scripting, and general networking basics — none of that is re-taught here. What *is* new, even for an experienced operator:

- The exact kernel state machine behind the `ps` state letters, including `TASK_KILLABLE` as the reason some NFS mounts are killable and others aren't.
- Load average as a literal EWMA formula over `nr_running + nr_uninterruptible`, not a vibe.
- The EEVDF scheduler (kernel 6.6+, replacing CFS's pick-next logic) — lag, eligibility, and virtual deadline — which most engineers still describe using CFS-only vocabulary.
- GPU-fleet-specific tuning most generalist SREs never touch: `isolcpus=`, `nohz_full=`, and IRQ affinity to keep NCCL polling threads off the shared fair-scheduler run queue.
- Why a misbehaving `SCHED_FIFO`/`SCHED_DEADLINE` thread can starve an entire node — a real-time-class failure mode that "percent CPU" monitoring never surfaces.

## Core concepts

### Process states
Every task has a state, exposed as a single letter in `ps` and in `/proc/<pid>/stat` (field 3) and spelled out in `/proc/<pid>/status` (`State:` line). The ones that matter:

- **R — running or runnable.** On a CPU right now, or on a run queue waiting for one. `ps` does not distinguish "on-CPU" from "wants-CPU"; both show R.
- **S — interruptible sleep.** Blocked waiting for an event (a socket, a timer, a keypress), and *will* wake for a signal. The overwhelming majority of processes on any box are S. This is the normal idle-waiting state.
- **D — uninterruptible sleep.** Blocked in the kernel on something the kernel refuses to let a signal interrupt — almost always I/O completion or a lock held across an I/O. Signals are queued but **not delivered** until the task leaves D. This is why `kill -9` does nothing to a D-state task.
- **T — stopped** (by `SIGSTOP`/`SIGTSTP`, or ptrace/`t`).
- **Z — zombie.** Exited, but its parent hasn't `wait()`ed to reap the exit status. Holds a PID and a slot, nothing else.
- **I — idle kernel thread** (a kthread parked so it doesn't inflate load — introduced precisely so idle worker threads stop counting).

The kernel constants behind the letters: `TASK_RUNNING`, `TASK_INTERRUPTIBLE`, `TASK_UNINTERRUPTIBLE`, `TASK_STOPPED`, `EXIT_ZOMBIE`, `TASK_IDLE`. `TASK_KILLABLE` is a middle ground — uninterruptible except for fatal signals — which modern NFS and some paths use so a hung mount is at least `kill`-able; older/`hard` NFS mounts are full D.

### What load average actually measures
Load average is **not** CPU utilization. It is an exponentially-damped moving average (1/5/15-minute) of the number of tasks in state **R or D** — runnable *plus* uninterruptible. The kernel samples `nr_running + nr_uninterruptible` roughly every 5 seconds and feeds it through the EWMA. You can read the instantaneous inputs:

```
$ cat /proc/loadavg
0.52 0.58 0.59 2/1834 44127
      # 1m   5m   15m  running/total  last-pid
```

The `2/1834` is runnable-now / total-threads. The three floats include D-state tasks. So:

- **Load can be high while CPU is ~0%** when many tasks sit in D waiting on slow/hung I/O. They aren't burning CPU; they're counted anyway. This is the classic NFS/storage signature.
- **Load can be lower than you'd expect on a saturated box** if work is single-threaded — one pegged core is load ~1 on a 128-core node.

Because D-state inflates load, load average on a GPU node is often a **storage-health proxy**, not a compute-pressure signal. For actual saturation, PSI (`/proc/pressure/{cpu,io,memory}`) is the better instrument — `some`/`full` stall percentages tell you whether tasks are *waiting* on a resource, decoupled from the load EWMA. (PSI gets its own full lesson — [04 — PSI](04-psi.md) — this lesson only uses it as a confirming instrument.)

### How the scheduler picks the next task
The default scheduling class is the fair class (`SCHED_NORMAL`/`SCHED_OTHER`). Above it sit the real-time classes (`SCHED_FIFO`, `SCHED_RR`) and `SCHED_DEADLINE`, which **always** preempt fair-class tasks — a `SCHED_FIFO` thread that never blocks or yields can starve every fair-class task on that core indefinitely. This is a real fleet failure mode, not a theoretical one: a misconfigured RT thread (a driver's polling thread pinned wrong, a monitoring agent accidentally launched with `chrt -f`) can silently wedge a node's fair-scheduled workloads while `top` shows "one process using 100% of a core" and everything else just... stops making progress. Below the fair class sits `SCHED_IDLE`. `chrt` shows/sets class and priority.

**CFS (pre-6.6):** Completely Fair Scheduler ordered runnable tasks by **vruntime** — virtual runtime, wall-time-on-CPU scaled by the task's weight (derived from nice). Lower vruntime = more owed CPU = picked next, via a red-black tree keyed on vruntime. `nice` changed the weight, so a niced task accumulated vruntime faster and got less CPU. Fairness was "everyone's vruntime stays close."

**EEVDF (6.6+):** Earliest Eligible Virtual Deadline First replaced CFS's picking logic (merged 6.6, 2023) while **keeping the same interfaces** — nice, cgroup `cpu.weight`, and CFS bandwidth control (`cpu.max`) all still work. Two new ideas:

- **lag** — each task's signed deviation from its perfectly-fair share. Positive lag = the task is owed CPU (ran less than fair); negative = it got more than fair. EEVDF only considers a task **eligible** when its lag is ≥ 0, so a task that just overran must wait until virtual time catches up before it's eligible again.
- **virtual deadline** — among *eligible* tasks, EEVDF picks the one with the earliest virtual deadline, computed from the task's requested **time slice** (`sched_runtime`, tunable via `sched_setattr`). A shorter requested slice yields an earlier deadline, so latency-sensitive tasks can ask for smaller slices and get scheduled sooner without changing their overall CPU share.

The practical upshot for a platform engineer: EEVDF gives better tail latency for interruptible, latency-sensitive tasks and makes slice length a first-class request, but from cgroup/nice's perspective the throttling and weighting knobs you tune in Kubernetes are unchanged. `vruntime` still exists internally; `lag` is the new eligibility gate layered on top. See kernel docs: [sched-eevdf](https://docs.kernel.org/scheduler/sched-eevdf.html).

### Dedicating cores on GPU nodes: isolcpus=, nohz_full=, IRQ affinity
On a generic fair-scheduled core, your task competes for run-queue slots with everything else the kernel schedules there — timer ticks, softirqs, other processes. For most workloads that's invisible noise. For a **GPU node's NCCL proxy/polling thread**, which busy-spins waiting on collective-communication completions to shave microseconds off an all-reduce, sharing a core with the fair-class run queue means occasional multi-millisecond scheduling delays — which show up as stalls in distributed training steps, not as anything `top` calls out.

Three kernel boot parameters address this (documented in the [kernel-parameters.txt admin guide](https://docs.kernel.org/admin-guide/kernel-parameters.html)):

- **`isolcpus=`** — removes listed CPUs from the general SMP balancing and scheduler domains, so ordinary tasks are never scheduled onto them unless explicitly pinned there (via `taskset`/`sched_setaffinity`/cgroup `cpuset`). This gives a latency-sensitive thread a core with (almost) nothing else on the fair-class run queue.
- **`nohz_full=`** — stops the periodic scheduler timer tick on the listed CPUs when only one runnable task is present, eliminating a recurring source of jitter for a busy-spinning thread.
- **IRQ affinity** (`/proc/irq/<n>/smp_affinity`, or `irqbalance` exclusion) — steers hardware interrupts (NIC, NVMe) away from the isolated cores, so an interrupt storm on another device doesn't preempt the pinned thread.

Together these three turn "a core the fair scheduler occasionally hands to something else" into "a core that is functionally dedicated" — the standard recipe for pinning NCCL/RDMA polling threads and other latency-critical GPU-fleet workloads off the shared run queue.

### Where the numbers live
- `/proc/<pid>/stat` — space-separated, positional. Field 3 = state letter; field 14/15 = utime/stime (clock ticks on CPU); field 19 = nice; field 22 = starttime.
- `/proc/<pid>/status` — human-readable: `State:`, `Threads:`, `voluntary_ctxt_switches`, `nonvoluntary_ctxt_switches` (high nonvoluntary = getting preempted a lot).
- `/proc/<pid>/wchan` — the kernel symbol the task is sleeping in (e.g. `nfs_wait_on_request`). This is how you turn "it's in D" into "it's in D *on NFS*."
- `/proc/<pid>/schedstat`, `/proc/<pid>/sched` — time-on-CPU vs time-waiting-on-runqueue; the second is your run-queue-latency evidence.

## Perspectives

**Kernel-mechanism view.** A process is a `task_struct` with a state field the kernel drives through `TASK_RUNNING → TASK_INTERRUPTIBLE/UNINTERRUPTIBLE → …`, and the fair-class scheduler picks among runnable tasks using vruntime (CFS) or lag + virtual deadline (EEVDF). Nothing about "priority" or "fairness" is a human concept applied after the fact — it is the literal sort key a red-black-tree-like structure uses to answer "who runs next." Understanding this mechanism is what turns "the process is stuck" into a specific, falsifiable claim about which state and which wchan.

**Operator/SRE view.** The recurring ticket is "the node feels wedged" — vague, urgent, and usually wrong about the cause. The discipline that resolves it fast is a fixed sequence, not a guess: read `/proc/loadavg` for the R/D split, `ps -eo stat` to separate runnable from uninterruptible, `wchan` on the D-state tasks to name the exact kernel path they're blocked in, then `/proc/pressure/{cpu,io}` to confirm which resource is actually saturated. Skipping straight to "let's reboot it" is how the same storage-path incident recurs weekly instead of getting root-caused once.

**GPU-fleet-specific view.** Noisy-neighbor CPU contention is a distributed-training tax that's easy to miss because the GPU itself looks idle-adjacent: a data-loader thread or the NCCL proxy thread gets bumped off-CPU by an unrelated fair-class task sharing the core, and the whole training step stalls waiting on that one thread — the GPU is fully capable of working, but nothing is feeding it. This is exactly the case for CPU pinning and isolation (`isolcpus=`, `nohz_full=`, IRQ affinity) on GPU nodes: dedicating cores to the polling/communication threads removes them from the shared fair-scheduler run queue entirely, trading some fleet-wide CPU flexibility for predictable, low-jitter step times.

**Economics/failure-mode view.** An idle-but-D-state GPU node is still burning its full reserved-instance or GPU-hour cost — the accelerator sits there fully billed while every worker thread waits on a hung NFS mount, contributing nothing. And scheduling latency — the time a runnable task spends waiting on the run queue before it actually gets a CPU — directly taxes p99 step time in distributed training, because a synchronous collective (all-reduce) is only as fast as its slowest participant; one straggler thread delayed by run-queue contention delays the entire job, not just its own progress.

## Real-world use cases

- **Meta Engineering — "Modernizing the Meta Ads Service With an Open-Source Kernel Scheduler"** — https://engineering.fb.com/2026/07/13/ml-applications/modernizing-the-meta-ads-service-with-an-open-source-kernel-scheduler/ — Meta found a latency regression from EEVDF on their ads-ranking service after a kernel upgrade, then built and shipped `sched_ext` (a BPF-programmable scheduler class, developed with Google's ghOSt authors) to run a workload-specific scheduler — cutting ads-retrieval p99 latency 28% and saving 3.28 MW of fleet power. *What it shows:* when the default scheduler doesn't fit a workload, a staff-level response is to make the scheduler itself programmable rather than fight the default's tuning knobs.
- **Netflix TechBlog — "Noisy Neighbor Detection with eBPF"** — https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd — Netflix instruments scheduler `sched_switch` and run-queue events with eBPF (under 600ns overhead per hook) to self-serve-detect CPU contention between co-located containers in production. *What it shows:* "load / run-queue latency ≠ CPU utilization" isn't academic — it's how a large fleet actually finds noisy neighbors before customers notice. Forward-links to [08 — eBPF](08-ebpf.md) and [09 — perf/ftrace/USE method](09-perf-ftrace-use.md), where this instrumentation technique is covered directly.
- **Brendan Gregg — "Linux Load Averages: Solving the Mystery"** — https://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html — the canonical deep-dive on why Linux load counts R+D state (unlike other Unixes' CPU-only load), told through production war stories of exactly this "load high, CPU idle" confusion. *What it shows:* this isn't a one-off gotcha — it's a decades-old, still-surprising property of the kernel that keeps producing incidents.

## Worked example
A node reports load 42 on 16 cores; `top` shows CPUs 96% idle. Trace it:

```
$ cat /proc/loadavg
41.9 40.2 33.7 1/2203 88510          # ~42 load, only 1 runnable now → the rest are D

$ ps -eo pid,stat,wchan:32,comm | awk '$2 ~ /D/'
  7712 D    nfs_wait_on_request   dd
  7713 D    nfs_wait_on_request   dd
  ... (dozens more, all wchan nfs_*)

$ cat /proc/7712/status | grep -E 'State|ctxt'
State:  D (disk sleep)
voluntary_ctxt_switches:   4
nonvoluntary_ctxt_switches: 0     # not preempted — it's parked, not competing for CPU

$ cat /proc/7712/stat | awk '{print "state="$3" utime="$14" stime="$15}'
state=D utime=1 stime=6           # ~7 ticks of CPU total → it has done almost no computing
```

Reading it: ~41 tasks are in D, all with `wchan` in the NFS read path, each having consumed a handful of CPU ticks. They contribute ~41 to load while burning no CPU — hence load 42 with idle cores. Confirm the resource is I/O, not CPU:

```
$ cat /proc/pressure/io
some avg10=99.20 avg60=98.71 avg300=71.40 total=...
full avg10=97.55 ...              # tasks are almost continuously stalled on I/O
$ cat /proc/pressure/cpu
some avg10=0.31 ...               # essentially no CPU contention
```

Verdict: an NFS server (or the network to it) is slow/hung. The fix is on the storage path, not the node's CPU. `kill -9 7712` will not clear it — the task is uninterruptible until the I/O returns or times out.

## Practice
On a laptop or VM, manufacture and read a D-state task. This directly feeds the module deliverable, [Anatomy of a Container](../practice/anatomy-of-a-container/README.md) — the diagnostic reflex you build here (state → wchan → PSI) is exactly what the toolkit's later steps assume you already have.

1. Create sustained uninterruptible I/O. Easiest reliable path: a loop device on a file, or heavy direct I/O.
   ```
   # generate a large file and read it with O_DIRECT in a loop to force real I/O waits
   fallocate -l 4G /var/tmp/blob
   while :; do dd if=/var/tmp/blob of=/dev/null bs=1M iflag=direct; done &
   ```
   (Even better if you can point `dd` at a slow/networked mount. On fast NVMe D-windows are brief — run several readers so you can catch one.)
2. Catch a task in D and read its state three ways:
   ```
   ps -eo pid,stat,wchan:32,comm | awk '$2 ~ /D/'
   PID=<one of them>
   grep State /proc/$PID/status
   awk '{print $3}' /proc/$PID/stat
   cat /proc/$PID/wchan; echo
   ```
3. Correlate load with states. Run `uptime`, then count:
   ```
   uptime
   ps -eo stat | grep -c '^R'      # runnable
   ps -eo stat | grep -c '^D'      # uninterruptible
   cat /proc/loadavg
   ```
4. Prove `kill -9` is impotent against D: `kill -9 $PID` a task that is currently in D and watch it survive until the I/O completes.
5. Optional (GPU-fleet stretch goal): if you have `taskset` and a multi-core box, pin a busy-spin loop to one core with `taskset -c 0`, then start several other CPU-bound processes and watch `/proc/<pid>/schedstat`'s run-queue-wait field grow — the same mechanism `isolcpus=` is designed to prevent.

**Acceptance:** a 3–5 sentence note stating an observed load value, the R and D counts at that moment, `/proc/pressure/{cpu,io}` readings, and a one-line explanation of why load ≠ CPU busy for your case (e.g. "load 6 with 0 runnable and 6 in D on `blk_*`/`dio` wchan; io pressure 90%+, cpu pressure ~0 → load is counting I/O waiters"). Clean up: `kill %1; rm /var/tmp/blob`.

## Common pitfalls
1. **Reading load average as "percent CPU busy."** It isn't a percentage of anything — it's an EWMA-smoothed count of tasks in R+D. A load of 42 on a 16-core box can mean 96% idle CPU if most of those tasks are parked in D.
2. **Assuming `kill -9` always works.** D-state defeats it — signals are queued, not delivered, until the task leaves uninterruptible sleep. `TASK_KILLABLE` (used by soft NFS mounts and some other paths) is the deliberate exception that lets fatal signals through; hard/legacy paths give you true unkillable D.
3. **Treating "load < core count" as automatically healthy.** A single pegged thread on a 128-core box is load ~1 — trivially under the core count, and easy to wave away, while that one thread's owner is fully stalled.
4. **Confusing nice/weight with a cap.** Nice (and cgroup `cpu.weight`) changes a task's *share* of CPU under contention — it never limits how much CPU a task can use when nothing else wants it. That confusion resurfaces almost identically in [03 — cgroups v2](03-cgroups-v2-and-k8s-enforcement.md), where `cpu.weight` (a share) and `cpu.max` (a hard cap) are easy to mix up.
5. **Assuming EEVDF changed the interfaces you tune.** It didn't. Nice, `cpu.weight`, and `cpu.max` bandwidth control all mean exactly what they meant under CFS — only the internal pick-next algorithm (vruntime-only → lag + virtual deadline) changed.

## Self-check
- **Why can load average exceed core count while CPU is ~0%?**
  **Answer:** Load average counts tasks in state R *and* D — runnable plus uninterruptible sleep — sampled and EWMA-smoothed. Tasks blocked in D on slow or hung I/O consume no CPU but are counted in every load sample. Enough of them (a hung NFS/Ceph mount parking dozens of readers) drives load far above the core count while the CPUs sit idle. Load is a "how many tasks want to make progress but are blocked on a resource" signal, not a utilization percentage; use PSI (`/proc/pressure`) for actual saturation.
- **What are vruntime and lag, and what does EEVDF change vs CFS?**
  **Answer:** vruntime is CFS's per-task virtual runtime — wall time spent on CPU scaled by the task's nice-derived weight; CFS always ran the runnable task with the lowest vruntime, keeping everyone's vruntime close. EEVDF (6.6+) adds **lag**, each task's signed deviation from its perfectly-fair share: a task is only **eligible** when lag ≥ 0, so a task that just overran must wait for virtual time to catch up. Among eligible tasks EEVDF picks the earliest **virtual deadline**, derived from the task's requested time slice, so latency-sensitive tasks can request shorter slices for quicker scheduling. Interfaces (nice, `cpu.weight`, `cpu.max` bandwidth) are unchanged — only the pick-next algorithm changed.
- **What puts a task in D vs S, and why can't you kill a D-state process?**
  **Answer:** S (interruptible sleep) is a task blocked on an event that the kernel allows a signal to wake — normal waiting. D (uninterruptible sleep) is a task blocked in a kernel path — typically waiting for I/O completion or a lock held across I/O — that the kernel deliberately shields from signal delivery to protect in-flight state. Signals to a D task are queued but not delivered until it leaves D, so `kill -9` has no effect until the underlying I/O completes or times out. `TASK_KILLABLE` (used by soft NFS and some paths) is the compromise that lets fatal signals through; hard/legacy paths give you true unkillable D.
- **Why would a GPU-fleet operator dedicate CPU cores with `isolcpus=` instead of just adding more cores?**
  **Answer:** Adding cores doesn't remove jitter from a shared fair-class run queue — the scheduler can still hand a core to another task between the NCCL proxy thread's polling iterations, and periodic timer ticks add scheduling noise even on an otherwise-idle core. `isolcpus=` removes listed CPUs from general SMP scheduling so ordinary tasks aren't placed there; paired with `nohz_full=` (stopping the periodic tick when only one task is runnable) and IRQ affinity (steering hardware interrupts elsewhere), the result is a core that behaves close to dedicated hardware for the latency-sensitive thread — which more cores alone can't guarantee.
- **Why does a misconfigured `SCHED_FIFO` thread threaten an entire node, when a runaway `SCHED_NORMAL` process doesn't?**
  **Answer:** Real-time classes (`SCHED_FIFO`, `SCHED_RR`) and `SCHED_DEADLINE` always preempt fair-class (`SCHED_NORMAL`) tasks on the kernel's priority ordering — that's the point of the RT classes. A `SCHED_FIFO` thread that busy-loops without blocking or yielding will hold a core indefinitely, since the fair scheduler has no mechanism to reclaim it; a runaway `SCHED_NORMAL` process, by contrast, still time-slices against its peers via vruntime/lag and never starves the whole run queue. This is why RT priority should be reserved for threads that briefly need guaranteed low latency and reliably yield, not applied casually to "make something faster."

## Connections & what's next
This lesson's state machine and scheduler model are the substrate the rest of the module builds on: [03 — cgroups v2](03-cgroups-v2-and-k8s-enforcement.md) reuses `cpu.weight`/`cpu.max` — the same share-vs-cap distinction flagged in the pitfalls above, now enforced per-cgroup rather than per-nice-value; [04 — PSI](04-psi.md) formalizes the `/proc/pressure` instrument this lesson used only as a confirming check; and [08 — eBPF](08-ebpf.md) / [09 — perf/ftrace/USE method](09-perf-ftrace-use.md) pick up the `sched_switch`/off-CPU instrumentation that the Netflix use case above previews. The immediate next step, though, is **[02 — Namespaces](02-namespaces.md)**: it takes the same process — the `task_struct` you now know is just a state machine with a scheduler entry — and asks what it means to give that process a private *view* of PIDs, mounts, and the network, which is the other half of "what a container really is."

## References & further reading

**Primary sources**
- sched-eevdf kernel docs — https://docs.kernel.org/scheduler/sched-eevdf.html — canonical, terse spec of the current pick-next algorithm and which interfaces (nice, `sched_setattr` slice, bandwidth) survived the switch from CFS.
- kernel-parameters.txt admin guide — https://docs.kernel.org/admin-guide/kernel-parameters.html — authoritative reference for `isolcpus=` and `nohz_full=` boot parameters used for CPU isolation on GPU nodes.

**Real-world engineering blogs**
- Meta Engineering — "Modernizing the Meta Ads Service With an Open-Source Kernel Scheduler" — https://engineering.fb.com/2026/07/13/ml-applications/modernizing-the-meta-ads-service-with-an-open-source-kernel-scheduler/ — what it shows: a staff-level response (build a BPF-programmable scheduler, `sched_ext`) when the default scheduler doesn't fit a large-scale workload; 28% p99 latency cut, 3.28 MW power saved.
- Netflix TechBlog — "Noisy Neighbor Detection with eBPF" — https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd — what it shows: production-scale scheduler instrumentation (`sched_switch`, run-queue events) used to self-serve-detect CPU contention between co-located containers.
- Brendan Gregg — "Linux Load Averages: Solving the Mystery" — https://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html — what it shows: why Linux load counts R+D (not just CPU-runnable), with the production war stories behind the design.

**Deeper dives**
- Systems Performance, 2nd ed. — CPUs chapter (Brendan Gregg) — https://www.brendangregg.com/systems-performance-2nd-edition-book.html — the authoritative treatment of load average, run-queue latency, and scheduler observability; gives the vocabulary (run-queue latency, PSI, USE method) senior interviewers expect.
- EEVDF Scheduler (LWN) — https://lwn.net/Articles/925371/ — the clearest narrative of why CFS was replaced and how lag/eligibility/deadlines work.
- LWN — "sched/fair: Fix low cpu usage with high throttling…" — https://lwn.net/Articles/792268/ — a real CFS-bandwidth-control bug from the scheduling-mechanism side; cross-links with [03 — cgroups v2](03-cgroups-v2-and-k8s-enforcement.md)'s Indeed use case, which hit the same bug from the cgroup-throttling angle.
