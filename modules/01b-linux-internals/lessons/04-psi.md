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
sources: 5
---

# 01b.4 · Pressure Stall Information (PSI)

> **Concept.** PSI measures the *time a workload was stalled waiting* for CPU, memory, or I/O — the saturation signal that utilization graphs hide.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 03 built the whole cgroup-v2 enforcement picture — `cpu.max`, `memory.max`, the QoS tree — and along the way introduced `cpu.pressure`/`memory.pressure`/`io.pressure` as a side note under "the observability lever." This lesson promotes that side note to the main event. Lesson 03 answered "how much of a resource is this cgroup *allowed*"; this lesson answers a different and, for on-call debugging, more urgent question: "is anything being *denied* a resource *right now*, and by how much wall-clock time." Where lesson 03 gave you the enforcement files, this lesson gives you the *diagnostic* file — the one you read first when a symptom shows up and you don't yet know which resource is guilty.

## Why this matters

Your GPU node shows 60% CPU utilization and the fleet dashboard is green. Yet training throughput has collapsed: `nvidia-smi` shows the GPUs sitting at 30% utilization, waiting. The data-loader — CPU-side processes that decode JPEGs and feed tensors to the GPU — is stalling, and every millisecond it stalls, an expensive GPU idles (2026 on-demand pricing for a single high-end training GPU commonly runs several dollars per hour; a multi-GPU node is easily $20–40+/hr — treat any such figure as a dated snapshot, not a fixed number). Utilization cannot see this, because utilization measures *time spent running*, not *time spent waiting to run*. A CPU that is 60% busy can still have runnable threads queued behind it 90% of the time if those threads are bursty.

PSI is the kernel's answer to the question "is anything being *denied* a resource right now?" It is the mechanism behind:

- **Kubernetes node-pressure eviction** — the kubelet's memory/disk signals, and (as of the 1.36 GA feature covered below) increasingly PSI-derived signals, decide which pods get evicted before the node dies.
- **`systemd-oomd`** (and Meta's own **oomd**, its production ancestor) and **Android's LMKD** — all kill processes based on memory PSI *before* the kernel OOM killer fires, avoiding the hard stall.
- **Autoscalers and bin-packers** that want to pack GPU nodes densely without tipping them into thrash.

If your differentiator is cost and observability, PSI is the single most important saturation metric you can teach a fleet to emit. It turns "the node feels slow" into "memory pressure some=42% for 60s, resource=memory, victim cgroup=data-loader."

## What's new here (calibration)

You already run `top`/`htop`/`kubectl top node`, watch `%CPU`, load average, and `free -m`, and know load average above core count is "bad" and utilization near 100% is "saturated" — the module README's skip-list assumes exactly this operator fluency, and lesson 01 already gave you the precise mechanics of load average (R+D states, EWMA) as a *storage-health proxy*, not a saturation instrument. This lesson does not re-derive load average.

What's genuinely new:

- The precise, kernel-accounted (not sampled) definitions of `some` and `full`, and why they measure something load average and utilization structurally cannot.
- The system-level CPU asymmetry (`full` doesn't exist for system-wide CPU, but does per-cgroup) — a detail that trips people up in interviews because it seems inconsistent until you see why.
- The trigger-FD/`poll()` mechanism that lets automated systems act on pressure in milliseconds, not scrape intervals.
- PSI's direct lineage into production tooling — Meta's oomd (PSI's origin story) and, as of a recent Kubernetes release, first-class GA kubelet support — which turns this from "a kernel curiosity" into "a metric already on your node today."

## Core concepts

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

**Per-cgroup (cgroup v2):** each cgroup directory exposes `cpu.pressure`, `memory.pressure`, `io.pressure`. Same format. This is what makes PSI operationally powerful — you can attribute pressure to *one pod's cgroup* instead of guessing at the node level, the same leaf-cgroup-reading discipline lesson 03 built.

```
$ cat /sys/fs/cgroup/kubepods.slice/.../cpu.pressure
some avg10=41.7 avg60=38.2 avg300=22.5 total=99182340
full avg10=12.3 avg60=10.1 avg300=6.4  total=41029118
```

### PSI trigger FDs (the advanced hook)

You can `write()` a threshold like `some 150000 1000000` (150ms of stall per 1s window) to a pressure file, then `poll()` the fd — the kernel wakes you when pressure crosses the threshold. `systemd-oomd` and LMKD use exactly this to act in single-digit milliseconds instead of polling. Worth knowing exists; you rarely hand-roll it.

Enablement: kernel must be built with `CONFIG_PSI=y`. Some distros default it off at runtime behind `psi=1` on the kernel cmdline; `CONFIG_PSI_DEFAULT_DISABLED` controls that. If `/proc/pressure/` is missing, that's why.

### How this maps to Kubernetes and the GPU fleet

- **Node-pressure eviction** historically keys off `memory.available` (from `memory.working_set`) and `nodefs`/`imagefs` signals. But those are *level* signals — "how much is left" — not *rate-of-pain* signals. PSI memory `full` rising is the leading indicator that reclaim is thrashing *before* `memory.available` crosses the hard threshold. As of Kubernetes v1.36, kubelet-native PSI collection at node/pod/container level graduated to **GA, on by default** (see Real-world use cases) — this is no longer a metric you have to bolt on yourself.
- **The stalled GPU step.** A training step is a pipeline: CPU loader → host-to-device copy → GPU kernel. If the loader's cgroup shows `cpu.pressure some` high while node CPU utilization is moderate, the loader is being starved of cores (too few, or noisy-neighbor contention) — the GPU stalls downstream. If the loader shows `io.pressure full` spikes, it's blocked reading shards from disk/network FS. PSI tells you *which* stage of the pipeline lost the time, and *which* resource — it does not, by itself, tell you the exact code path or call stack that was blocked. That's lesson 09's job: once PSI has localized the resource and the cgroup, off-CPU flame graphs (built from scheduler tracepoints) show you the precise stack that was waiting. PSI narrows the search; off-CPU analysis finishes it.

## Perspectives

**Kernel-mechanism view.** PSI (merged in Linux 4.20, written by Johannes Weiner at Facebook) instruments the scheduler and the memory/I/O reclaim paths directly, at the moment a task can't make progress — it's runnable but off-CPU waiting for a core, or blocked in direct reclaim, or blocked on I/O caused by memory pressure. The kernel accumulates that stall time as it happens and aggregates it into the `some`/`full` running averages. This is not sampled after the fact the way `top` samples `/proc` every interval; the accounting happens at the scheduling event itself, which is why PSI can catch stalls too brief for a sampling profiler to ever see.

**Operator/SRE view.** Contrast a dashboard-that-lies with a dashboard-that-tells-the-truth. Utilization answers "was the resource busy?" — a question that is compatible with *both* a healthy node and a badly starved one, because bursty demand can leave a CPU only 60% busy on average while still queuing tasks behind it most of the time. PSI answers a different, sharper question: "how much wall-clock time did real work lose to waiting?" It turns a vibe ("the node feels slow," "something's off") into a number you can alert on, graph, and cite in a postmortem — `memory some avg10=42%` is falsifiable in a way "feels slow" never was.

**GPU-fleet-specific view.** The lesson's headline story — a data loader stalling the GPU step while node CPU utilization looks fine — is PSI doing exactly the job it was built for: localizing *which resource, in which cgroup* is the bottleneck in a multi-stage pipeline where the expensive resource (the GPU) is downstream of the cheap one (CPU/IO) that's actually stalled. Once PSI has told you "this cgroup, this resource, this severity," lesson 09's off-CPU flame graphs tell you the exact stack — PSI is the wide-angle instrument, off-CPU tracing is the zoom lens.

**Failure-mode/economics view.** PSI and classic eviction thresholds are different *kinds* of signal, not competing versions of the same one. A level signal like `memory.available` tells you how much runway is left — a fuel gauge. A rate signal like PSI `full` tells you how fast you're burning it — a rate-of-climb indicator. The second pages you before the first one does: a node can have plenty of `memory.available` left by the gauge while `memory.pressure full` is already climbing because reclaim is thrashing to keep it that way. Waiting for the level signal alone means you find out about the fire after it's already spread; the rate signal is your early warning.

## Real-world use cases

- **Meta Engineering — "Open-sourcing oomd, a new approach to handling OOMs"** — https://engineering.fb.com/2018/07/19/production-engineering/oomd/ — Facebook's userspace OOM daemon watches `memory.pressure avg10` and proactively sheds load or kills low-priority jobs *before* the kernel's blunt-instrument OOM killer ever fires. This is PSI's origin story in production, written by the same team (Johannes Weiner et al.) that wrote PSI itself — the direct ancestor of `systemd-oomd`.
- **Kubernetes Blog — "Kubernetes v1.36: PSI Metrics for Kubernetes Graduates to GA"** — https://kubernetes.io/blog/2026/05/12/kubernetes-v1-36-psi-metrics-ga/ — PSI moved from kernel curiosity to a first-class, GA, on-by-default kubelet feature: node-, pod-, and container-level PSI collection built into the platform you operate daily. Concrete evidence that the metric this lesson teaches isn't a lab exercise — it's already on the node.

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

**Acceptance (feeds "[Anatomy of a Container](../practice/anatomy-of-a-container/README.md)"):** a short note — 8–15 lines — presenting one cgroup or node that is **not** CPU-saturated by utilization (paste the `mpstat` average, < 80% busy) yet **is** pressured by PSI (paste the `/proc/pressure/cpu` or `*.pressure` line, `some` > 15%). State explicitly **which resource** is pressured and **which cgroup** owns it, and one sentence on the GPU-fleet failure mode it models (starved loader → idle GPU). Include the raw `total=` delta over 1s as your rate evidence.

## Common pitfalls

1. **Reading `some` and `full` as interchangeable.** They measure different failure modes: `full` is the lost-throughput signal (nothing ran), `some` is the latency/queueing signal (something waited). A dashboard that only surfaces one is blind to the other — a node can have low `full` but climbing `some` (queueing, not yet total starvation) and that's still a real, worsening problem.
2. **Expecting system-wide `/proc/pressure/cpu` to report `full`.** It structurally can't — "every task stalled on CPU" at the system level is indistinguishable from an idle CPU with nothing runnable, which isn't contention. Only *per-cgroup* CPU pressure reports `full`, because from inside a constrained cgroup all of *its* tasks can be starved while sibling cgroups run freely.
3. **Assuming PSI is always available.** It requires `CONFIG_PSI=y` at build time and, on some distros, `psi=1` on the kernel cmdline (`CONFIG_PSI_DEFAULT_DISABLED` governs the runtime default). A missing `/proc/pressure/` directory is a configuration gap to fix, not evidence "PSI doesn't apply here."
4. **Polling PSI files on a fixed interval instead of using the trigger-FD/`poll()` mechanism** for latency-sensitive automated response. Polling on a scrape interval means seconds of detection latency; the trigger-FD path (what `systemd-oomd` and LMKD use) delivers a wakeup in single-digit milliseconds once a threshold is crossed.
5. **Conflating high PSI with "the box is overloaded, add capacity."** PSI localizes *which* resource and *which* cgroup is stalled — it's a diagnostic instrument, not automatically a capacity verdict. High pressure can just as easily mean a noisy neighbor, a misconfigured limit, or a single bursty tenant; read the cgroup breakdown before reaching for more hardware.

## Self-check

**(a) What do `some` vs `full` pressure mean, precisely?**
**Answer:** `some` = the fraction of the time window in which **at least one** task was stalled waiting for the resource (a latency/contention signal). `full` = the fraction in which **all** non-idle tasks were stalled simultaneously, so zero productive work occurred (a lost-throughput signal). `full ≤ some` always. System-wide `/proc/pressure/cpu` reports only `some` (all-tasks-stalled-on-CPU is just an idle CPU, not contention); memory and I/O report both, and per-cgroup CPU also reports `full`.

**(b) How can CPU be < 100% utilized yet CPU-pressured?**
**Answer:** Utilization is time-averaged busyness; pressure is time tasks spent *runnable but off-CPU*. With bursty threads that wake in bunches, many can queue behind the run-queue at the same instant while, averaged over the second, cores still go idle between bursts. So average utilization sits below 100% while `some` pressure — a task waited for a core — is high. It's a distribution/queueing effect utilization smooths away.

**(c) Which PSI signal warns of memory thrash before an OOM kill?**
**Answer:** Rising **memory `full`** pressure (per-cgroup `memory.pressure` `full`, or system `/proc/pressure/memory` `full`). When `full` climbs, every task in the scope is stuck in direct reclaim — the kernel is spending wall-clock time evicting and re-faulting pages instead of running the workload. This is the thrash plateau that precedes the OOM killer; `systemd-oomd`, its ancestor Meta's `oomd`, and LMKD act on exactly this signal to kill early and avoid the hard stall.

**(d) Why doesn't system-wide `/proc/pressure/cpu` have a `full` line, and why does the per-cgroup version have one?**
**Answer:** `full` means "every non-idle task was stalled simultaneously." At the whole-system level that condition is indistinguishable from the CPU simply being idle with nothing runnable — there's no meaningful notion of "the entire system's CPU demand is being denied" separate from "no one wants the CPU." Inside a single cgroup, though, `full` is meaningful: all of *that* cgroup's tasks can be stalled (none scheduled) while sibling cgroups elsewhere on the box are actively running on the CPU. So per-cgroup CPU pressure can report real lost throughput for that cgroup even while the system as a whole is busy.

## Connections & what's next

PSI is the load-bearing diagnostic instrument for the rest of the module: it's the pressure axis of the USE method (Utilization, Saturation, Errors) that lesson 09 formalizes, it's the signal `systemd-oomd`-style tooling and Meta's oomd act on before the kernel OOM killer in lesson 05 ever fires, and it's the file you'll `cat` first — before `cpu.stat` or `memory.events` — whenever a symptom shows up and you don't yet know which resource is guilty. Lesson 03's `kubepods.slice` tree is where you point PSI: attribute pressure to a leaf cgroup the same way you attributed cost there.

Next: **[05 · Memory management & the OOM killer](05-memory-and-oom.md)**, which follows the thread from "memory `full` is climbing" (this lesson's pre-thrash signature) all the way to "here is the exact process the kernel killed and why" — the reclaim path, `oom_badness()`, and reading a real `dmesg` OOM report cold.

## References & further reading

**Primary sources**
- **PSI kernel documentation** — https://docs.kernel.org/accounting/psi.html — the authoritative spec: `some`/`full` definitions, file format, the trigger/`poll()` API. Read first.

**Real-world engineering blogs**
- **Meta Engineering — "Open-sourcing oomd, a new approach to handling OOMs"** — https://engineering.fb.com/2018/07/19/production-engineering/oomd/ — what it shows: a production userspace daemon acting on `memory.pressure` to kill/shed load before the kernel OOM killer, written by PSI's own authors — PSI's origin story in production.
- **Kubernetes Blog — "Kubernetes v1.36: PSI Metrics for Kubernetes Graduates to GA"** — https://kubernetes.io/blog/2026/05/12/kubernetes-v1-36-psi-metrics-ga/ — what it shows: PSI collection at node/pod/container level is now a GA, on-by-default kubelet feature, plus the newer PSI-based node-pressure-eviction direction.

**Deeper dives**
- **Facebook PSI intro** — https://facebookmicrosites.github.io/psi/ — the origin-team explainer with the "utilization lies, pressure doesn't" framing and production motivation; cross-reference with the oomd post above (same team).
- **Brendan Gregg, *Systems Performance* (2nd ed.), methodology chapters** — https://www.brendangregg.com/systems-performance-2nd-edition-book.html — use the USE method (Utilization, Saturation, Errors) to place PSI as the *saturation* axis the other tools miss.
