---
lesson: "01b.1"
title: "Processes and Scheduling"
module: "01b"
concept: "Processes and Scheduling"
status: not-started
est_time: "4h"
artifacts: []
---

# 01b.1 · Processes and Scheduling

> **Concept.** A process is a set of kernel states, not a "running program" — and load average, the scheduler, and the D-state are what you actually reason about when a GPU node looks busy but the CPUs are idle.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters
In an interview for a GPU-fleet role, "load average is 80 but CPU is 3%, what's wrong?" is a filter question — the wrong answer (CPUs are saturated) marks you as an operator who never looked below `top`. On real fleets the answer is usually storage: NFS-backed datasets, a hung Ceph mount, or a wedged NVMe controller parking training jobs in uninterruptible sleep, where they inflate load, resist `kill -9`, and block drains. Knowing exactly what puts a task in D, what load average counts, and how the scheduler picks the next task is the difference between "the node is stuck" and "PID 24601 is blocked in nfs_readpage, the storage path is the incident."

## From using to understanding
As an operator you read `uptime`, `top`, and `ps` and trust the numbers: load is "how busy," a process is "running or not," `nice` makes things "lower priority." You `kill -9` stuck processes and mostly it works.

What you're learning now is the mechanism under those numbers. A "process" is a `task_struct` with a **state** field the kernel drives through a state machine. "Load" is a specific sampled count the kernel maintains, not a CPU utilization percentage. "Priority" is an input to a scheduler (EEVDF since 6.6, CFS before) that orders tasks by a virtual clock, not a simple ranking. And `kill -9` is a signal — which the kernel can only deliver at moments a task is in the right state to receive it. Once you see the state machine, the pathological cases (D-state, load-vs-CPU divergence, priority inversion) stop being mysteries.

## Core notes

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

Because D-state inflates load, load average on a GPU node is often a **storage-health proxy**, not a compute-pressure signal. For actual saturation, PSI (`/proc/pressure/{cpu,io,memory}`) is the better instrument — `some`/`full` stall percentages tell you whether tasks are *waiting* on a resource, decoupled from the load EWMA.

### How the scheduler picks the next task
The default scheduling class is the fair class (`SCHED_NORMAL`/`SCHED_OTHER`). Above it sit real-time classes (`SCHED_FIFO`, `SCHED_RR`) and `SCHED_DEADLINE`, which always win over fair tasks; below it is `SCHED_IDLE`. `chrt` shows/sets class and priority.

**CFS (pre-6.6):** Completely Fair Scheduler ordered runnable tasks by **vruntime** — virtual runtime, wall-time-on-CPU scaled by the task's weight (derived from nice). Lower vruntime = more owed CPU = picked next, via a red-black tree keyed on vruntime. `nice` changed the weight, so a niced task accumulated vruntime faster and got less CPU. Fairness was "everyone's vruntime stays close."

**EEVDF (6.6+):** Earliest Eligible Virtual Deadline First replaced CFS's picking logic (merged 6.6, 2023) while **keeping the same interfaces** — nice, cgroup `cpu.weight`, and CFS bandwidth control (`cpu.max`) all still work. Two new ideas:

- **lag** — each task's signed deviation from its perfectly-fair share. Positive lag = the task is owed CPU (ran less than fair); negative = it got more than fair. EEVDF only considers a task **eligible** when its lag is ≥ 0, so a task that just overran must wait until virtual time catches up before it's eligible again.
- **virtual deadline** — among *eligible* tasks, EEVDF picks the one with the earliest virtual deadline, computed from the task's requested **time slice** (`sched_runtime`, tunable via `sched_setattr`). A shorter requested slice yields an earlier deadline, so latency-sensitive tasks can ask for smaller slices and get scheduled sooner without changing their overall CPU share.

The practical upshot for a platform engineer: EEVDF gives better tail latency for interruptible, latency-sensitive tasks and makes slice length a first-class request, but from cgroup/nice's perspective the throttling and weighting knobs you tune in Kubernetes are unchanged. `vruntime` still exists internally; `lag` is the new eligibility gate layered on top.

### Where the numbers live
- `/proc/<pid>/stat` — space-separated, positional. Field 3 = state letter; field 14/15 = utime/stime (clock ticks on CPU); field 19 = nice; field 22 = starttime.
- `/proc/<pid>/status` — human-readable: `State:`, `Threads:`, `voluntary_ctxt_switches`, `nonvoluntary_ctxt_switches` (high nonvoluntary = getting preempted a lot).
- `/proc/<pid>/wchan` — the kernel symbol the task is sleeping in (e.g. `nfs_wait_on_request`). This is how you turn "it's in D" into "it's in D *on NFS*."
- `/proc/<pid>/schedstat`, `/proc/<pid>/sched` — time-on-CPU vs time-waiting-on-runqueue; the second is your run-queue-latency evidence.

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
On a laptop or VM, manufacture and read a D-state task.

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

**Acceptance:** a 3–5 sentence note stating an observed load value, the R and D counts at that moment, `/proc/pressure/{cpu,io}` readings, and a one-line explanation of why load ≠ CPU busy for your case (e.g. "load 6 with 0 runnable and 6 in D on `blk_*`/`dio` wchan; io pressure 90%+, cpu pressure ~0 → load is counting I/O waiters"). Clean up: `kill %1; rm /var/tmp/blob`.

## Self-check
**(a) Why can load average exceed core count while CPU is ~0%?**
**Answer:** Load average counts tasks in state R *and* D — runnable plus uninterruptible sleep — sampled and EWMA-smoothed. Tasks blocked in D on slow or hung I/O consume no CPU but are counted in every load sample. Enough of them (a hung NFS/Ceph mount parking dozens of readers) drives load far above the core count while the CPUs sit idle. Load is a "how many tasks want to make progress but one of them is blocked on a resource" signal, not a utilization percentage; use PSI (`/proc/pressure`) for actual saturation.

**(b) What are vruntime and lag, and what does EEVDF change vs CFS?**
**Answer:** vruntime is CFS's per-task virtual runtime — wall time spent on CPU scaled by the task's nice-derived weight; CFS always ran the runnable task with the lowest vruntime, keeping everyone's vruntime close. EEVDF (6.6+) adds **lag**, each task's signed deviation from its perfectly-fair share: a task is only **eligible** when lag ≥ 0, so a task that just overran must wait for virtual time to catch up. Among eligible tasks EEVDF picks the earliest **virtual deadline**, derived from the task's requested time slice, so latency-sensitive tasks can request shorter slices for quicker scheduling. Interfaces (nice, `cpu.weight`, `cpu.max` bandwidth) are unchanged — only the pick-next algorithm changed.

**(c) What puts a task in D vs S, and why can't you kill a D-state process?**
**Answer:** S (interruptible sleep) is a task blocked on an event that the kernel allows a signal to wake — normal waiting. D (uninterruptible sleep) is a task blocked in a kernel path — typically waiting for I/O completion or a lock held across I/O — that the kernel deliberately shields from signal delivery to protect in-flight state. Signals to a D task are queued but not delivered until it leaves D, so `kill -9` has no effect until the underlying I/O completes or times out. `TASK_KILLABLE` (used by soft NFS and some paths) is the compromise that lets fatal signals through; hard/legacy paths give you true unkillable D.

## Resources
1. **Systems Performance, 2nd ed. — CPUs chapter (Brendan Gregg)** — https://www.brendangregg.com/systems-performance-2nd-edition-book.html — the authoritative treatment of load average, run-queue latency, and scheduler observability, with the load-average-is-not-CPU explanation. *Deep.* Why: it gives you the vocabulary (run-queue latency, PSI, USE method) that senior interviewers expect.
2. **EEVDF Scheduler (LWN)** — https://lwn.net/Articles/925371/ — the clearest narrative of why CFS was replaced, what lag and eligibility mean, and how deadlines are computed. *Deep.* Why: EEVDF is recent enough that most engineers still say "CFS" — knowing lag/eligibility is a differentiator.
3. **sched-eevdf kernel docs** — https://docs.kernel.org/scheduler/sched-eevdf.html — canonical, terse spec of the current algorithm and interfaces. *Skim.* Why: confirms which knobs (nice, slice via `sched_setattr`, bandwidth) survived the switch.
