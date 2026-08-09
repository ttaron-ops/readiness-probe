---
lesson: "01b.4"
title: "Pressure Stall Information (PSI)"
module: "01b"
concept: "Pressure Stall Information (PSI)"
status: not-started
est_time: "4h"
artifacts: []
---

# 01b.4 · Pressure Stall Information (PSI)

> **Concept.** PSI measures the *time a workload was stalled waiting* for CPU, memory, or I/O — the saturation signal that utilization graphs hide.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters

Your GPU node shows 60% CPU utilization and the fleet dashboard is green. Yet training throughput has collapsed: `nvidia-smi` shows the GPUs sitting at 30% utilization, waiting. The data-loader — CPU-side processes that decode JPEGs and feed tensors to the GPU — is stalling, and every millisecond it stalls, an $30k/hr GPU idles. Utilization cannot see this, because utilization measures *time spent running*, not *time spent waiting to run*. A CPU that is 60% busy can still have runnable threads queued behind it 90% of the time if those threads are bursty.

PSI is the kernel's answer to the question "is anything being *denied* a resource right now?" It is the mechanism behind:

- **Kubernetes node-pressure eviction** — the kubelet's memory/disk signals, and increasingly PSI-derived signals, decide which pods get evicted before the node dies.
- **`systemd-oomd`** and **Android's LMKD** — both kill processes based on memory PSI *before* the kernel OOM killer fires, avoiding the hard stall.
- **Autoscalers and bin-packers** that want to pack GPU nodes densely without tipping them into thrash.

If your differentiator is cost and observability, PSI is the single most important saturation metric you can teach a fleet to emit. It turns "the node feels slow" into "memory pressure some=42% for 60s, resource=memory, victim cgroup=data-loader."

## From using to understanding

**What the operator knows.** You read `top`, `htop`, `kubectl top node`. You watch `%CPU`, load average, `free -m`, `iostat`. You know load average > core count is "bad" and utilization near 100% is "saturated."

**Why that model is blind.** Utilization answers *"was the resource busy?"* Load average answers *"how many tasks were runnable or in uninterruptible sleep?"* — a 1/5/15-minute exponentially-decayed count that conflates CPU demand with D-state I/O waiters and gives you no per-resource breakdown. Neither tells you *how much wall-clock time real work lost to waiting*. A box at 55% CPU with a bursty, latency-sensitive loader is functionally saturated; a box at 95% CPU running one throughput batch job with nothing queued behind it is perfectly healthy. Utilization mislabels both.

**The kernel mechanism.** PSI (merged in Linux 4.20, by Johannes Weiner at Facebook) instruments the scheduler and the memory/I/O reclaim paths directly. Every time a task *cannot make progress because a resource is contended* — it's runnable but off-CPU waiting for a core, or blocked in direct reclaim, or blocked on I/O caused by memory pressure — the kernel accumulates stall time. It aggregates that into two productivity-loss ratios, **some** and **full**, per resource, and exposes running averages. This is not sampled after the fact; it is accounted at the scheduling events themselves.

## Core notes

### The two numbers: `some` vs `full`

For each resource PSI reports the fraction of a time window during which tasks were stalled:

- **`some`** — the percentage of time in which **at least one** task was stalled on the resource. This is a *latency / contention* signal: something, somewhere, waited.
- **`full`** — the percentage of time in which **every non-idle task** was stalled on the resource simultaneously, so *no work at all* got done. This is a *throughput-loss / lost-work* signal: the whole workload (or cgroup) was frozen.

`full` is always ≤ `some`. `full = 20%` means one fifth of wall-clock time was pure dead air — nothing productive ran because everyone was waiting.

**Important asymmetry:** `/proc/pressure/cpu` has **no `full` line at the system level** in mainline. Rationale: for CPU, "every task stalled" would mean the CPU is idle with nothing runnable, which by definition isn't CPU contention. (Per-cgroup CPU pressure *does* report `full`, because from a cgroup's constrained view all of *its* tasks can be starved while other cgroups run.) Memory and I/O report both `some` and `full` everywhere.

### The files

**System-wide:** `/proc/pressure/cpu`, `/proc/pressure/memory`, `/proc/pressure/io`.

```
$ cat /proc/pressure/cpu
some avg10=8.42 avg60=6.13 avg300=4.20 total=1290034567
$ cat /proc/pressure/memory
some avg10=0.00 avg60=0.12 avg300=0.31 total=8452119
full avg10=0.00 avg60=0.08 avg300=0.19 total=5109882
```

- `avg10 / avg60 / avg300` — the stall percentage averaged over the last 10, 60, and 300 seconds.
- `total` — a monotonic **microsecond** counter of cumulative stall time since boot. This is the field to scrape for rate() in Prometheus: `rate(psi_total[1m])` gives you stall-microseconds per second, i.e. instantaneous pressure, without the EWMA smearing of avg10.

**Per-cgroup (cgroup v2):** each cgroup directory exposes `cpu.pressure`, `memory.pressure`, `io.pressure`. Same format. This is what makes PSI operationally powerful — you can attribute pressure to *one pod's cgroup* instead of guessing at the node level.

```
$ cat /sys/fs/cgroup/kubepods.slice/.../cpu.pressure
some avg10=41.7 avg60=38.2 avg300=22.5 total=99182340
full avg10=12.3 avg60=10.1 avg300=6.4  total=41029118
```

### PSI trigger FDs (the advanced hook)

You can `write()` a threshold like `some 150000 1000000` (150ms of stall per 1s window) to a pressure file, then `poll()` the fd — the kernel wakes you when pressure crosses the threshold. `systemd-oomd` and LMKD use exactly this to act in single-digit milliseconds instead of polling. Worth knowing exists; you rarely hand-roll it.

Enablement: kernel must be built with `CONFIG_PSI=y`. Some distros default it off at runtime behind `psi=1` on the kernel cmdline; `CONFIG_PSI_DEFAULT_DISABLED` controls that. If `/proc/pressure/` is missing, that's why.

### How this maps to Kubernetes and the GPU fleet

- **Node-pressure eviction** today keys off `memory.available` (from `memory.working_set`) and `nodefs`/`imagefs` signals. But those are *level* signals — "how much is left" — not *rate-of-pain* signals. PSI memory `full` rising is the leading indicator that reclaim is thrashing *before* `memory.available` crosses the hard threshold. Fleets that scrape `memory.pressure` per pod evict the *right* pod earlier.
- **The stalled GPU step.** A training step is a pipeline: CPU loader → host-to-device copy → GPU kernel. If the loader's cgroup shows `cpu.pressure some` high while node CPU utilization is moderate, the loader is being starved of cores (too few, or noisy-neighbor contention) — the GPU stalls downstream. If the loader shows `io.pressure full` spikes, it's blocked reading shards from disk/network FS. PSI tells you *which* stage of the pipeline lost the time. Utilization tells you none of it.

## Worked example

**Goal: produce a node that is NOT CPU-saturated by utilization yet IS CPU-pressured by PSI.** We create contention by oversubscribing cores with more runnable threads than cores, each doing short bursts, so no single instant is 100% busy but tasks constantly queue.

Confirm PSI is on:

```
$ cat /proc/pressure/cpu
some avg10=0.11 avg60=0.09 avg300=0.10 total=1200334
```

Baseline utilization (say 8 cores). Now oversubscribe with intermittent CPU hogs — more workers than cores, with sleep gaps so average utilization stays below 100%:

```
$ nproc
8
$ stress-ng --cpu 24 --cpu-load 55 --timeout 120s &
```

`--cpu-load 55` makes each of 24 workers busy ~55% of the time. Naive math: 24 × 0.55 = 13.2 core-seconds of demand against 8 cores. Utilization will pin *near* 100% here, so to get the "sub-saturation but pressured" case, drop it: `--cpu 12 --cpu-load 50` → demand ≈ 6 core-seconds on 8 cores. Now watch both:

```
$ mpstat 1 3      # or: sar -u 1 3
Average: all  62.40  0.00  1.10  0.00  ...  36.50 (%idle)
```

Utilization ≈ 63% — a green dashboard. Meanwhile:

```
$ cat /proc/pressure/cpu
some avg10=27.4 avg60=19.8 avg300=9.6 total=1990556
```

**Read this:** ~37% of cores are idle, yet 27% of wall-clock time had a task stalled waiting for a CPU. That is the contradiction utilization can't express. The 12 bursty workers collide: when several wake at once they queue behind 8 cores even though the *average* leaves cores idle. A latency-sensitive data-loader living here would miss its deadline to feed the GPU one step in four.

**Trace the total for a clean rate:**

```
$ awk '/some/{print $5}' /proc/pressure/cpu   # total=... microseconds
# sample twice 1s apart; delta/1e6 = stalled CPU-seconds in that second
```

If the delta is 270,000 µs over 1 s, that's 0.27 s of aggregate stall per second — matching avg10≈27%.

**Now the memory analogue** (feeds Practice + the deliverable). Put a hog in a memory-capped cgroup and watch `memory.pressure`:

```
$ sudo mkdir /sys/fs/cgroup/psi-demo
$ echo 256M | sudo tee /sys/fs/cgroup/psi-demo/memory.max
$ echo $$ | sudo tee /sys/fs/cgroup/psi-demo/cgroup.procs   # move this shell in
$ stress-ng --vm 1 --vm-bytes 1G --vm-keep --timeout 60s &
$ watch -n1 cat /sys/fs/cgroup/psi-demo/memory.pressure
some avg10=64.20 avg60=41.10 avg300=15.30 total=44190882
full avg10=58.70 avg60=37.40 avg300=13.90 total=39882001
```

`full` near 60% means the cgroup spent most of its time with *every* task frozen in reclaim — the process is being kept alive only by frantically evicting and re-faulting pages. This is the pre-OOM thrash signature (next lesson). `memory.current` will be pinned at ~`memory.max` and `memory.events` `high`/`max` counters climb.

## Practice

**Environment:** a Linux VM or laptop with a v2 cgroup tree (`stat -fc %T /sys/fs/cgroup` → `cgroup2fs`), `stress-ng`, `sysstat` (`mpstat`/`sar`), root. A container works if the host exposes PSI and the cgroup is delegated.

1. **CPU pressure without CPU saturation.** Run `stress-ng --cpu <N> --cpu-load 50` with `N` ≈ 1.5× your core count. In parallel capture `mpstat 1 10` and sample `/proc/pressure/cpu` each second. Find a run where mean utilization stays **< 80%** while `cpu some avg10` stays **> 15%**.
2. **Memory pressure to thrash.** Create `/sys/fs/cgroup/psi-demo` with `memory.max=256M`, move a `stress-ng --vm 1 --vm-bytes 1G` into it, and record `memory.pressure` climbing. Note the moment `full` overtakes 40%.
3. **Correlate a stalled workload < 100% util.** While CPU pressure is high but util < 80%, run a tiny latency probe (a loop that measures wall-time to do a fixed 1ms of work, or `cyclictest` if available) and show its latency inflates even though "there's idle CPU."

**Acceptance (feeds "Anatomy of a Container"):** a short note — 8–15 lines — presenting one cgroup or node that is **not** CPU-saturated by utilization (paste the `mpstat` average, < 80% busy) yet **is** pressured by PSI (paste the `/proc/pressure/cpu` or `*.pressure` line, `some` > 15%). State explicitly **which resource** is pressured and **which cgroup** owns it, and one sentence on the GPU-fleet failure mode it models (starved loader → idle GPU). Include the raw `total=` delta over 1s as your rate evidence.

## Self-check

**(a) What do `some` vs `full` pressure mean, precisely?**
**Answer:** `some` = the fraction of the time window in which **at least one** task was stalled waiting for the resource (a latency/contention signal). `full` = the fraction in which **all** non-idle tasks were stalled simultaneously, so zero productive work occurred (a lost-throughput signal). `full ≤ some` always. System-wide `/proc/pressure/cpu` reports only `some` (all-tasks-stalled-on-CPU is just an idle CPU, not contention); memory and I/O report both, and per-cgroup CPU also reports `full`.

**(b) How can CPU be < 100% utilized yet CPU-pressured?**
**Answer:** Utilization is time-averaged busyness; pressure is time tasks spent *runnable but off-CPU*. With bursty threads that wake in bunches, many can queue behind the run-queue at the same instant while, averaged over the second, cores still go idle between bursts. So average utilization sits below 100% while `some` pressure — a task waited for a core — is high. It's a distribution/queueing effect utilization smooths away.

**(c) Which PSI signal warns of memory thrash before an OOM kill?**
**Answer:** Rising **memory `full`** pressure (per-cgroup `memory.pressure` `full`, or system `/proc/pressure/memory` `full`). When `full` climbs, every task in the scope is stuck in direct reclaim — the kernel is spending wall-clock time evicting and re-faulting pages instead of running the workload. This is the thrash plateau that precedes the OOM killer; `systemd-oomd` and LMKD act on exactly this signal to kill early and avoid the hard stall.

## Resources

1. **PSI kernel documentation** — <https://docs.kernel.org/accounting/psi.html>. The authoritative spec: `some`/`full` definitions, file format, trigger/poll API. Read first.
2. **Facebook PSI intro** — <https://facebookmicrosites.github.io/psi/>. The origin-team explainer with the "utilization lies, pressure doesn't" framing and production motivation.
3. **Brendan Gregg, *Systems Performance* (2nd ed.), methodology chapters** — <https://www.brendangregg.com/systems-performance-2nd-edition-book.html>. Use the USE method (Utilization, Saturation, Errors) to place PSI as the *saturation* axis the other tools miss.
