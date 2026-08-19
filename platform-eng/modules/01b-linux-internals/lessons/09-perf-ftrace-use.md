---
lesson: "01b.9"
title: "perf, ftrace, and the USE Method"
module: "01b"
concept: "perf, ftrace, and the USE Method"
status: not-started
est_time: "7h"
prev: "08-ebpf.md"
next: "10-systemd-cgroups-and-block-io.md"
artifacts: []
sources: 12
---

# 01b.9 · perf, ftrace, and the USE Method

> **Concept.** The USE method is a systematic checklist that turns "the node is slow" into a bounded investigation; perf and flame graphs are how you see *where* the CPU actually goes, on-CPU and off-CPU — together they are how you answer every performance debugging question, in an incident or an interview.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 08 gave you eBPF: a way to attach verified programs to kernel hooks and aggregate what they see, cheaply, in production. That is an *instrument*. This lesson supplies the other two things you need to use it — a **method** that tells you which instrument to reach for and when, and the two remaining instrument families that eBPF does not replace: **`perf`**, the kernel's statistical sampling profiler built on the hardware PMU, and **ftrace**, the kernel's built-in exhaustive function tracer that is present on every kernel whether or not BPF is available to you.

Where lesson 08 answered "how do I safely hook the kernel," this lesson answers "given a slow node and every tool at your disposal, what do I check, in what order, and how do I read what comes back." It also unlocks lesson 10: io PSI is the USE-method saturation instrument for disk, and the block-layer latency histogram you will build here is the same measurement `biolatency` makes, one layer down.

## Why this matters

"A node is slow — walk me through how you'd debug it" is the single most common systems interview question at the companies in this module's target list, and unstructured answers ("I'd check top, then maybe iostat, then...") read as guessing. Guessing is not a style problem; it is a *coverage* problem. Without a method you have no way to know when you have ruled a resource out, so you cannot know when you are done, so you stare at CPU while the disk is the problem.

On a GPU fleet the stakes are literal money. An H100 node reserved at roughly **$2–3 per GPU-hour** (2024–2025 on-demand snapshots; check your provider and commit terms — this varies by an order of magnitude between spot, reserved and on-demand) that sits at 5% CPU looks idle to a dashboard tuned for CPU-bound services. If its training process is off-CPU 80% of the time blocked on a slow dataset mount, the GPUs are stalled waiting for batches and you are paying full price for silicon doing nothing. Eight A100s or H100s idling for a week is a five-figure number. The tool that reveals "this process spends 80% of wall-clock *off* CPU in `nfs` reads" is off-CPU analysis — and nothing in `top`, Grafana, or a GPU utilization dashboard will tell you that.

The third reason is durability. eBPF is the newest layer and needs a modern kernel, root-equivalent capability, and BTF. `perf` and ftrace work on the locked-down vendor kernel of a five-year-old node, in a restricted environment, on the box where the eBPF agent is exactly what you suspect. Knowing all three means you always have something.

## What's new here (calibration)

Per the [module README](../README.md)'s calibration, you already know shell pipelines, `top`/`iostat`/`sar` as commands, and general Linux administration — none of that is re-taught. What's genuinely new:

- **USE as a complete method**, with the queueing-theory reason saturation beats utilization as an early warning, and the interval-averaging trap that makes "utilization" lie.
- **How sampling actually works**: PMU counters, the overflow interrupt, `sample_freq` versus `sample_period`, and the arithmetic linking sample rate to overhead *and* to statistical confidence.
- **Reading a flame graph correctly** — width not depth, x-axis is population not time — plus how one is *constructed*, which is what makes the reading rules obvious rather than memorized.
- **Off-CPU analysis** as a first-class technique with its own instrumentation, producing the blocked-stack evidence that on-CPU profiling structurally cannot.
- **ftrace as an actual tool**, not a name: the tracefs files, what writing to each one does, the function-graph output format including its latency markers, and when exhaustive tracing beats sampling.
- **The two independent walls** (`perf_event_paranoid` and container seccomp) that make `perf` "just not work" in a pod, and why fixing one without the other leaves you stuck.
- The **GPU-fleet economics angle**: converting off-CPU wall-clock into wasted reserved-GPU dollars.

## Core concepts

### 1. Why you need a method at all

The failure mode this lesson exists to prevent looks like competence. An engineer gets "node-47 is slow," runs `top`, sees 30% CPU, runs `free`, sees memory fine, remembers a similar incident that turned out to be a memory leak, starts looking for a memory leak, and loses a day. Everything in that sequence is a reasonable action. The sequence as a whole has no completeness property: at no point can it answer *"which resources have I ruled out?"*

Brendan Gregg's **USE method** fixes exactly this by making the search space explicit and small. For **every resource**, check three things:

- **Utilization** — the proportion of time the resource was busy servicing work. For *capacity*-type resources (memory, disk space, file descriptors) it is instead the proportion of capacity consumed.
- **Saturation** — the degree to which the resource has **extra work queued** that it cannot service yet. Queue length, wait time, or a stall metric.
- **Errors** — the count of error events. Often silent, often the earliest signal of a failing device.

Three checks × every resource. The method's value is that it is **complete by construction**: you cannot forget to check disk saturation while fixated on CPU, because the grid has a cell for it, and an empty cell is itself a finding ("I have no saturation metric for this resource" → go get one).

**Why saturation is the signal operators miss.** Utilization is bounded at 100% and saturating a resource does not change it; the queue behind it grows without bound. Queueing theory gives the shape: for a simple M/M/1 queue, mean response time `R = S / (1 − U)` where `S` is service time and `U` is utilization. Put numbers on it, with a 1 ms service time:

| Utilization U | Response time R = S/(1−U) | What the dashboard shows |
|---|---|---|
| 50% | 2 ms | "half busy, plenty of headroom" |
| 70% | 3.3 ms | "fine" |
| 90% | 10 ms | "busy but not maxed" |
| 95% | 20 ms | "still not 100%!" |
| 99% | 100 ms | "utilization is 99%, that's just one percent from before" |

Between 90% and 99% utilization — a 10% change in the metric everyone watches — latency grows **10×**. This is why "the node is only at 85% CPU, that's not the problem" is wrong so often, and why a saturation metric (run-queue length, queue depth, PSI stall time) gives you warning that a utilization metric does not.

**The interval-averaging trap.** Utilization is always an average over a window. A device that is 100% busy for 500 ms and idle for 500 ms reports 50% utilization over a second — while every request arriving in that first half-second queues. At five-minute Prometheus resolution this hides a lot; a node can look 40% utilized and be saturated in bursts the whole time. Two defences: shorten the interval (`vmstat 1`, `mpstat -P ALL 1`), and prefer metrics that measure *stall* rather than *busy* — which is precisely what PSI does.

**Enumerate resources first, including the ones without obvious metrics.** The physical list: CPUs, memory capacity and bandwidth, storage devices (I/O and capacity), storage controllers, network interfaces, network controllers, and the interconnects between them (QPI/UPI between sockets, PCIe to devices). On a GPU node add: GPUs (SM utilization, memory), GPU memory bandwidth, NVLink/NVSwitch, the PCIe path to the NIC and to storage. Then the *software* resources, which are where surprising incidents live: process/thread capacity (`kernel.pid_max`, `threads-max`), file descriptors (`ulimit -n`), the conntrack table (lesson 07), cgroup limits (lesson 03 — a per-tenant cap is a resource with its own utilization and saturation: `cpu.max` usage and `nr_throttled`), and kernel/user mutexes.

```
   USE, mapped onto a real GPU node
   ────────────────────────────────
                                   ┌────────────── U: %busy per core (mpstat -P ALL 1)
   ┌───────────────────────────┐   │ S: runqueue len (vmstat r), /proc/pressure/cpu,
   │  CPU  socket0   socket1   │◀──┤    cgroup cpu.stat nr_throttled
   │  64c      +      64c      │   └ E: dmesg MCE, EDAC
   └─────┬─────────────┬───────┘
         │ UPI         │                ┌───────── U: free -h, /proc/meminfo
   ┌─────▼─────┐ ┌─────▼─────┐          │ S: vmstat si/so, /proc/pressure/memory,
   │ DRAM node0│ │ DRAM node1│◀─────────┤    direct reclaim, OOM kills in dmesg
   └─────┬─────┘ └─────┬─────┘          └ E: EDAC/ECC counters
         │ PCIe        │ PCIe
   ┌─────▼──────┐ ┌────▼───────┐        ┌───────── U: DCGM SM/mem util, nvidia-smi
   │ GPU0..3    │ │ GPU4..7    │◀───────┤ S: SM occupancy vs pending work, NVLink
   │  +NVLink   │ │  +NVLink   │        │    bandwidth vs link capacity
   └─────┬──────┘ └────┬───────┘        └ E: XID errors in dmesg, ECC/retired pages
         │              │
   ┌─────▼──────────────▼──────┐        ┌───────── U: iostat -xz 1 (%util — see caveat)
   │  NVMe (local scratch)     │◀───────┤ S: aqu-sz, await, /proc/pressure/io, io.pressure
   └───────────────────────────┘        └ E: smartctl, dmesg I/O errors
   ┌───────────────────────────┐        ┌───────── U: sar -n DEV 1 vs link speed
   │  NIC 2×200G (NCCL, NFS)   │◀───────┤ S: qdisc/ring drops, nstat retransmits
   └───────────────────────────┘        └ E: ip -s link errors, ethtool -S
   ┌───────────────────────────┐        ┌───────── U: conntrack count / max, fds / ulimit
   │  Software: conntrack, fds,│◀───────┤ S: allocation failures, EMFILE
   │  pids, cgroup caps        │        └ E: dmesg "table full", cpu.stat throttling
   └───────────────────────────┘
```

### 2. The USE grid you actually run

This is the checklist to internalize. Each cell names the tool *and* the single field to read.

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| **CPU** | `mpstat -P ALL 1` (`%usr`+`%sys` per core), `top` | `vmstat 1` `r` column vs core count; `/proc/pressure/cpu` `some avg10`; `runqlat` | `dmesg` (MCE), `/sys/devices/system/edac`, `perf stat` |
| **Memory (capacity)** | `free -m`, `/proc/meminfo` (`MemAvailable`) | `vmstat 1` `si`/`so`; `/proc/pressure/memory`; OOM kills in `dmesg`; `pgscan_*` in `/proc/vmstat` | EDAC/ECC counters, `dmesg` |
| **Memory (bandwidth)** | `perf stat -e uncore_imc/data_reads/,uncore_imc/data_writes/` (platform-specific) | Stalled cycles: `perf stat -e cycle_activity.stalls_mem_any` | — |
| **Disk (I/O)** | `iostat -xz 1` (`r/s`,`w/s`,`rkB/s`,`wkB/s`, `%util` with caveat) | `aqu-sz`, `r_await`/`w_await`; `/proc/pressure/io`; per-cgroup `io.pressure` | `smartctl -a`, `dmesg` I/O errors, `/sys/block/*/device/ioerr_cnt` |
| **Disk (capacity)** | `df -h`, inodes `df -i` | — (ENOSPC is the cliff) | `dmesg` ENOSPC, filesystem errors |
| **Network (interface)** | `sar -n DEV 1` (rx/tx kB/s vs link speed), `ip -s link` | `ss -s`, `nstat TcpRetransSegs`, qdisc drops (`tc -s qdisc`), NIC ring drops | `ip -s link` errors, `ethtool -S <dev>` |
| **GPU** | `nvidia-smi dmon`, DCGM `DCGM_FI_DEV_GPU_UTIL`, `..._SM_ACTIVE` | Kernel launch queue depth; host-side `cuStreamSynchronize` wait (lesson 08) | `nvidia-smi -q` XID errors, ECC/retired pages |
| **Interconnect (NVLink/PCIe)** | DCGM NVLink bandwidth counters vs link capacity | Saturation shows as NCCL collective time inflation | NVLink CRC error counters |
| **cgroup CPU cap** | `cpu.stat` `usage_usec` vs `cpu.max` quota | `cpu.stat` `nr_throttled` / `throttled_usec` | — |
| **Software (fds, pids, conntrack)** | `ls /proc/<pid>/fd \| wc -l` vs `ulimit -n`; `conntrack -C` vs `nf_conntrack_max` | Allocation failures, `EMFILE`, `EAGAIN` on `fork` | `dmesg` "table full", "nf_conntrack: table full, dropping packet" |

**The `%util` caveat, with its mechanism.** `iostat`'s `%util` is computed from the time the device had *at least one* request in flight. On a single-actuator spinning disk, that is a real utilization: one request at a time, so busy-time and capacity-consumed coincide. A modern NVMe drive has up to 64K queues with up to 64K entries each and services many requests concurrently — so a device happily serving one request at a time, at 2% of its throughput capability, still reports **`%util` = 100%**. The number is not lying; it is answering a different question than the one you have. What you want instead:

- **`aqu-sz`** (average queue size) — the actual outstanding-request depth. This is the saturation signal.
- **`await`** (average time per request, queue + service) — what the application feels.
- **Little's Law** ties them together: `aqu-sz = throughput × await`. A device doing 20,000 IOPS with 1 ms average latency has `20000 × 0.001 = 20` requests outstanding. If `await` climbs while IOPS stays flat, the queue is growing and you are saturated.
- **`/proc/pressure/io`** — time *lost* to waiting, which is the only one of these that is directly a cost.

### 3. Reading the vitals: field by field

The first 60 seconds on an unfamiliar slow node, and the specific field in each output that localizes the problem.

```
$ uptime
 11:42:07 up 43 days,  2:19,  1 user,  load average: 250.11, 240.30, 190.02
```

Load average on Linux counts running **plus uninterruptible-sleep (D-state)** tasks, so a high load with idle CPUs means tasks are blocked in the kernel — usually on I/O (lesson 01). Three numbers, 1/5/15-minute exponentially-damped averages: rising means the problem is arriving, falling means you may be looking at the aftermath.

```
$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2 61      0 3841216  22940 74213056  0    0  4832    24  9821 21044 12  4 23 61  0
 1 58      0 3840992  22940 74213120  0    0  5104    16 10233 21980 11  4 24 61  0
```

Field by field, and what each rules in or out:

| Field | Meaning | Read it as |
|---|---|---|
| `r` | Tasks running or runnable (waiting for CPU) | `r` > core count ⇒ **CPU saturation** |
| `b` | Tasks in uninterruptible sleep | `b` high ⇒ **blocked on I/O** — here, 58–61 blocked tasks |
| `si`/`so` | Swap in/out, KB/s | Non-zero ⇒ memory pressure spilling to disk; on a GPU node this is usually pathological |
| `bi`/`bo` | Blocks in/out from block devices | The device read/write rate |
| `in`/`cs` | Interrupts and context switches per second | Very high `cs` with low throughput ⇒ lock contention or thrashing |
| `us`/`sy` | User / system CPU % | High `sy` ⇒ the cost is in the kernel (syscalls — lesson 08's investigation) |
| `wa` | I/O wait % | Idle CPU with outstanding I/O. **61% here** — the headline |
| `st` | Steal (hypervisor took the CPU) | Non-zero on a VM ⇒ noisy neighbour on the *host* |

```
$ mpstat -P ALL 1 1
Average:  CPU   %usr  %nice  %sys %iowait  %irq  %soft %steal  %idle
Average:  all   12.3   0.00   4.1    61.2  0.00   0.31   0.00   22.1
Average:    0   11.8   0.00   3.9    62.0  0.00   0.20   0.00   22.1
Average:   17   99.0   0.00   1.0     0.0  0.00   0.00   0.00    0.0
```

`mpstat` shows the per-core breakdown that `vmstat`'s average hides. One core at 99% among 127 idle ones is a single-threaded bottleneck and is *invisible* in aggregate — the aggregate reads 0.8% busy. This is the single most common reason "the node has plenty of CPU" is wrong.

```
$ iostat -xz 1
Device   r/s    rkB/s  rrqm/s %rrqm r_await rareq-sz  w/s  wkB/s w_await aqu-sz %util
nvme0n1  19842 79368.0   12.0  0.06   45.02     4.00  310 12400   1.21  32.14  99.4
```

19,842 reads/s at an average request size of **4.00 KB** (`rareq-sz`), 45 ms average read latency, queue depth 32. Cross-check with Little's Law: `19842 × 0.045 ≈ 893`… which does *not* match `aqu-sz` 32 — because `r_await` includes time the request spent queued in the block layer *and* `aqu-sz` is an average over the interval; when they disagree by that much, you are looking at a bursty workload where the averages are over different populations. Take the disagreement itself as information: sample at 1-second resolution and watch whether both move together.

```
$ cat /proc/pressure/io
some avg10=88.20 avg60=84.11 avg300=71.03 total=182933944912
full avg10=71.50 avg60=68.02 avg300=57.44 total=140222110293
```

PSI (lesson 04) is the cleanest saturation instrument in the grid because it measures **lost time**, not busy time. `some` = the percentage of wall-clock in which *at least one* runnable task was stalled on I/O; `full` = the percentage in which *every* runnable task was stalled, i.e. the machine got no useful work done at all. `full avg10=71.5` means that over the last ten seconds, 71.5% of the time nothing could proceed because of I/O. That is not a warning sign; that is the incident.

```
$ nstat -az | grep -E 'TcpRetransSegs|TcpExtTCPTimeouts'
TcpRetransSegs      18422   0.0
TcpExtTCPTimeouts    1203   0.0

$ ss -s
Total: 4212 (kernel 0)
TCP:   3891 (estab 3402, closed 214, orphaned 0, timewait 210)
```

Retransmits are network saturation *or* error, depending on cause (lesson 07); a rising `TcpRetransSegs` under load with no interface errors points at a congested path or an overloaded peer, not a broken cable.

**The one-line decision rule after 60 seconds:** high `%usr` ⇒ on-CPU profile (§4–6). High `%sys` ⇒ syscall analysis (lesson 08) or on-CPU profile of kernel stacks. High `%iowait`, high `b`, high io PSI ⇒ **off-CPU** analysis (§8), because there is nothing to sample on-CPU. High `%steal` ⇒ the problem is not on this VM.

### 4. perf: how sampling actually works

`perf` answers "where does the CPU time go" by **statistical sampling**. The mechanism is worth understanding precisely, because every one of perf's flags and every one of its failure modes falls out of it.

Modern CPUs have a **Performance Monitoring Unit**: a handful of hardware counters (typically 4–8 general-purpose per core, plus fixed counters) that can be programmed to count events — retired instructions, unhalted core cycles, cache misses, branch mispredictions. `perf` programs a counter with a **period**: count down from N, and when the counter overflows, raise a **performance monitoring interrupt** (on x86 delivered as an NMI). The interrupt handler records a sample: the interrupted instruction pointer, the pid/tid, the CPU, a timestamp, and — if you asked for it — the call stack, unwound right there in the handler. Samples go into a per-CPU mmap ring buffer that the `perf` process drains into `perf.data`.

Two ways to specify the rate, and the difference matters:

- **`-F <hz>` (frequency mode, the default).** You ask for N samples per second; the kernel adjusts the counter period dynamically to hit that rate as the workload's event rate changes. `perf record` with neither `-F` nor `-c` defaults to **4000 Hz**.
- **`-c <count>` (period mode).** Sample once every N events exactly. Use this when you want "one sample per 1,000,000 cache misses" rather than a wall-clock rate.

Two kernel guardrails you will meet:

| Sysctl | Default | Effect |
|---|---|---|
| `kernel.perf_event_max_sample_rate` | 100000 | Ceiling on samples/sec; `-F` above this is clamped |
| `kernel.perf_cpu_time_max_percent` | 25 | Target ceiling on CPU spent in perf's own interrupt handler |

If the NMI handler starts taking too long — typically because DWARF unwinding is copying stack memory on every sample — the kernel prints `perf: interrupt took too long (2503 > 2500), lowering kernel.perf_event_max_sample_rate to 50000` and **halves your sample rate**. If you see that line in `dmesg`, your profile was silently taken at a lower rate than you asked for, and the sample counts you are reasoning about are not what you think.

**Why `-F 99` and not 100.** Round frequencies risk phase-locking with periodic kernel activity — timer ticks, a 100 Hz housekeeping thread, a 10 ms polling loop — so that samples systematically land in (or systematically miss) the same code. 99 Hz is co-prime with the common periods and decorrelates the sampling from them. It is a cheap superstition-free habit: nothing breaks at 100 Hz, but nothing is gained either.

**The overhead arithmetic.** Per sample you pay: the interrupt, the IP read, and the stack unwind. With frame-pointer unwinding this is roughly 1–2 µs. On a 64-core node at 99 Hz:

```
  samples/sec  = 99 Hz × 64 CPUs                     = 6,336 samples/s
  CPU cost     = 6,336 × 1.5 µs                      ≈ 9.5 ms/s across the node
               = 9.5 ms / 64 cores                   ≈ 0.015% per core
```

Negligible. Now the same run with `--call-graph dwarf`, which copies a chunk of user stack per sample for later unwinding — default **8192 bytes** per sample:

```
  data rate    = 6,336 samples/s × 8 KiB            ≈ 50 MB/s to perf.data
  30-second run                                     ≈ 1.5 GB
```

That is why DWARF profiling fills disks, and why long DWARF captures are what trigger the "interrupt took too long" auto-throttle. Use `--call-graph dwarf,4096` to halve it, or better, get frame pointers (§7).

**The statistical arithmetic — how long do you need to record?** Sampling is a Bernoulli process: a function occupying fraction `p` of CPU time gets, in expectation, `N·p` of your `N` samples, with relative standard error:

```
  relative SE = sqrt( (1 − p) / (N · p) )
```

Worked, for `-F 99 -a` on 64 CPUs for 30 s, with the node ~50% busy — so about `99 × 64 × 30 × 0.5 ≈ 95,000` samples land in running code:

| Frame's true share p | Expected samples | Relative SE | Reading |
|---|---|---|---|
| 20% | 19,000 | 0.7% | Rock solid |
| 5% | 4,750 | 1.4% | Solid |
| 1% | 950 | 3.2% | Trustworthy |
| 0.1% | 95 | 10% | Directionally true only |
| 0.01% | 9.5 | 32% | Noise — do not chase it |

**The rule that falls out: to resolve a frame at share `p` with ~10% relative error you need roughly `100/p` samples.** Chasing a 0.05%-wide sliver in a 30-second profile is chasing noise; either record longer, raise `-F`, or accept that it is not your problem. Conversely, if the hot frame is 30% wide, ten seconds at 99 Hz on one thread (990 samples) is already conclusive — resist the urge to record for ten minutes.

### 5. perf versus ftrace: two different bargains

These two tools cost what they cost for structural reasons. Hold this picture:

```
  Workload timeline (one CPU, 10 ms of a training step)
  ═══════════════════════════════════════════════════════════════════════

  actual work:  [tokenize][ memcpy ][   nn_forward   ][ sched ][  memcpy  ]
                0ms      1.4      2.9                7.1     7.6        10

  perf record -F 99 -a -g        ← STATISTICAL SAMPLING
                ↓        ↓        ↓        ↓        ↓        ↓        ↓
  samples:      ●        ●        ●        ●        ●        ●        ●
               (every ~10.1 ms; on THIS 10 ms window you get ~1 sample)
  cost:   O(sample_rate), independent of how busy the workload is
  gives:  a distribution — "nn_forward was on-CPU in 42% of samples"
  misses: anything short and rare; exact call sequence; exact durations
  scales: to a 128-core node, all day, at ~0.02% overhead

  ftrace function_graph on vfs_read subtree   ← EXHAUSTIVE TRACING
                ├─entry──────────────exit─┤  ├entry─exit┤   ├─entry──exit─┤
  events:       ▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮▮  ▮▮▮▮▮▮▮▮▮▮   ▮▮▮▮▮▮▮▮▮▮▮▮▮
                (one record per function entry AND exit, with timestamps)
  cost:   O(event_rate) — proportional to how much the kernel is doing
  gives:  the exact call tree, exact durations, every occurrence
  misses: nothing in the filtered set
  scales: to a narrow filter for seconds. Unfiltered `function` tracing
          on a busy node can consume double-digit % of CPU.

  ⇒ Sample to find WHERE the time goes. Trace to find WHY a specific
    path is slow, once you know which path.
```

That is the whole tool-selection rule. `perf record -a -g` is the wide-angle lens you point at an unknown problem; ftrace's `function_graph` is the macro lens you point at one function once you know its name. eBPF (lesson 08) sits between them: event-driven like ftrace, but aggregating in-kernel so it can stay attached to a high-frequency path.

### 6. perf record → report → script, the full loop

**`perf record`** captures samples into `perf.data`:

```
# System-wide, 99 Hz, with call graphs, for 30 seconds.
$ sudo perf record -F 99 -a -g -- sleep 30
[ perf record: Woken up 41 times to write data ]
[ perf record: Captured and wrote 12.183 MB perf.data (60418 samples) ]
```

The flags that matter:

| Flag | Effect |
|---|---|
| `-F 99` | Sample at 99 Hz (frequency mode) |
| `-c 1000000` | Sample every 1,000,000 events instead (period mode) |
| `-a` | All CPUs, system-wide |
| `-p <pid>` / `-t <tid>` | Only this process / thread |
| `-g` | Record call graphs (frame-pointer unwinding by default) |
| `--call-graph dwarf[,size]` | Unwind with DWARF CFI instead; default stack dump **8192 bytes** |
| `--call-graph lbr` | Use hardware Last Branch Records (cheap, shallow, Intel) |
| `-e <event>` | Sample on something other than cycles: `-e cache-misses`, `-e block:block_rq_issue`, `-e sched:sched_switch` |
| `-o out.data` | Output file |
| `--` `sleep 30` | Run a command and record for its lifetime — the idiom for "N seconds" |

`perf list` enumerates every event the box exposes: hardware PMU events, software events (`page-faults`, `context-switches`), and every tracepoint. `perf top` is the live version — a continuously updating ranked profile, `top` for functions.

**`perf report`** ranks the samples:

```
$ sudo perf report --stdio --no-children
# Samples: 60K of event 'cycles:P'
# Event count (approx.): 141822930193
#
# Overhead  Command      Shared Object          Symbol
# ........  ...........  .....................  ..............................
#
    31.42%  python3      libtorch_cpu.so        [.] at::native::copy_
    12.08%  python3      [kernel.kallsyms]      [k] copy_user_enhanced_fast_string
     9.77%  python3      libc.so.6              [.] __memmove_avx_unaligned_erms
     6.15%  python3      libjpeg.so.62          [.] jpeg_idct_islow
     4.02%  swapper      [kernel.kallsyms]      [k] intel_idle
```

Read the columns: **Overhead** is the share of samples; **Shared Object** is the DSO the instruction was in (`[kernel.kallsyms]` means kernel); the `[.]`/`[k]` marker distinguishes user from kernel. `--no-children` gives **self** time (samples *in* that function); the default `--children` gives inclusive time (self + everything it called), which is what you want when looking for an expensive subtree rather than an expensive leaf. Both views are useful and they answer different questions — an interviewer asking "self versus children" is checking whether you know that.

Here the story is already visible: a third of CPU in a tensor copy, another 12% in the kernel's copy-to-user, ~10% in `memmove`, 6% in JPEG decode. This is a data pipeline that is copying and decoding, not computing.

**`perf script`** dumps one record per sample with its full stack — the export format:

```
$ sudo perf script | head -12
python3 48213 [017] 918273.117402:   10101010 cycles:P:
        7f2c1a4b21e0 at::native::copy_+0x1e0 (/usr/lib/libtorch_cpu.so)
        7f2c1a3f0a12 at::_ops::copy_::call+0x92 (/usr/lib/libtorch_cpu.so)
        7f2c19d81b40 torch::autograd::THPVariable_copy_+0x110 (/usr/lib/libtorch_python.so)
              4f1a20 _PyObject_MakeTpCall+0x90 (/usr/bin/python3.11)
              4e9c31 _PyEval_EvalFrameDefault+0x4a1 (/usr/bin/python3.11)

python3 48213 [017] 918273.127511:   10101010 cycles:P:
        ffffffff81a4c210 copy_user_enhanced_fast_string+0x10 ([kernel.kallsyms])
        ffffffff8132a0f1 filemap_read+0x201 ([kernel.kallsyms])
        ffffffff81470c93 vfs_read+0x1a3 ([kernel.kallsyms])
```

Each record: command, pid, `[cpu]`, timestamp, event period, event name, then the stack leaf-first. `perf annotate` goes the other way — down into a single function's instructions with sample counts per instruction, which is how you find the specific load that is stalling.

**`perf stat`** is the complement to sampling: it *counts* events for a command without sampling at all.

```
$ perf stat -d ./train_step.py

 Performance counter stats for './train_step.py':

          8,412.31 msec task-clock                #    0.998 CPUs utilized
             1,204      context-switches          #  143.128 /sec
                18      cpu-migrations            #    2.140 /sec
           412,933      page-faults               #   49.086 K/sec
    31,102,884,201      cycles                    #    3.697 GHz
    12,884,201,003      instructions              #    0.41  insn per cycle
     2,102,884,552      branches                  #  249.977 M/sec
        31,203,441      branch-misses             #    1.48% of all branches
     1,884,201,338      L1-dcache-loads           #  224.000 M/sec
       402,884,102      L1-dcache-load-misses     #   21.38% of all L1-dcache accesses
        88,201,553      LLC-loads                 #   10.485 M/sec
        42,110,882      LLC-load-misses           #   47.74% of all LL-cache accesses

       8.430119831 seconds time elapsed
```

The number to read first is **IPC — instructions per cycle**, here 0.41. Modern x86 cores can retire ~4 instructions/cycle; anything below ~1.0 means the core is stalled most of the time, and the cache-miss lines say why: 21% L1 miss rate and a 48% LLC miss rate means the workload is streaming data that does not fit in cache and is waiting on DRAM. **That is a memory-bound signature, and no amount of CPU-time optimization in the hot function will fix it** — you fix it with data layout, batching, or NUMA placement (lesson 06). High IPC (>2) with slow wall-clock means you are genuinely compute-bound and the algorithm is the target.

### 7. Flame graphs: construction, then reading

A flame graph compresses tens of thousands of stacks into one picture. Understanding *how* it is built makes the reading rules obvious instead of memorized.

```
  STEP 1 — perf script emits one stack per sample (leaf first)
  ────────────────────────────────────────────────────────────
    sample 1: copy_    ← _ops::copy_  ← THPVariable_copy_ ← eval
    sample 2: copy_    ← _ops::copy_  ← THPVariable_copy_ ← eval
    sample 3: memmove  ← decode_jpeg  ← loader_worker     ← eval
    sample 4: copy_    ← _ops::copy_  ← THPVariable_copy_ ← eval
    sample 5: idct     ← decode_jpeg  ← loader_worker     ← eval

  STEP 2 — stackcollapse-perf.pl folds identical stacks, root first
  ─────────────────────────────────────────────────────────────────
    eval;THPVariable_copy_;_ops::copy_;copy_      3
    eval;loader_worker;decode_jpeg;memmove        1
    eval;loader_worker;decode_jpeg;idct           1

  STEP 3 — flamegraph.pl sorts the folded lines ALPHABETICALLY and
           draws each frame as a box whose WIDTH = its sample count
  ────────────────────────────────────────────────────────────────
     depth 4  │ copy_          │  │memmove│idct│   ← leaves: ON-CPU
     depth 3  │ _ops::copy_    │  │  decode_jpeg  │
     depth 2  │THPVariable_copy_│ │ loader_worker │
     depth 1  │              eval                 │   ← common root
              └──────3 samples───┘└───2 samples───┘
              ◀──────────── 5 samples total ──────────▶

  The x-axis is therefore NOT TIME. Sample 3 happened between samples
  2 and 4 in wall-clock, but merges with sample 5 in the picture. The
  x-axis is the sorted POPULATION of stacks; adjacency means "sorts
  next to," never "happened next."
```

The reading rules, and why each is true:

- **Width = fraction of samples = fraction of CPU time.** This is the only thing that means anything. A frame twice as wide consumed twice the CPU.
- **The top edge of each tower is the function that was actually on-CPU** when sampled. A wide, flat top frame is a leaf burning cycles directly — your optimization target.
- **Height is call-chain depth, not cost.** A tall, narrow tower is a deep call chain sampled rarely: lots of frames, little time. Do not chase it. This is the single most common misreading.
- **Left-to-right ordering carries no information.** It is alphabetical, chosen so identical subtrees merge into wide blocks.
- **Plateaus name the fix.** A wide `memcpy`, `malloc`, `__lock_text_start`, `json.Unmarshal`, or JPEG-decode plateau is a specific, actionable finding.

Generate one:

```
$ git clone https://github.com/brendangregg/FlameGraph
$ sudo perf record -F 99 -a -g -- sleep 30
$ sudo perf script > out.stacks
$ ./FlameGraph/stackcollapse-perf.pl out.stacks | ./FlameGraph/flamegraph.pl > cpu.svg
```

The SVG is interactive: click a frame to zoom into that subtree, Ctrl-F to search a regex — matched frames are highlighted and their **total width is summed and printed**, which answers "how much total time is in anything matching `crypto|ssl`?" without any re-processing.

**Differential flame graphs** answer a different question: not *where* time goes but *what changed*. Fold two profiles (before/after a deploy, or node-12 vs node-13 on the identical workload) and diff them:

```
$ ./FlameGraph/stackcollapse-perf.pl before.stacks > before.folded
$ ./FlameGraph/stackcollapse-perf.pl after.stacks  > after.folded
$ ./FlameGraph/difffolded.pl before.folded after.folded \
    | ./FlameGraph/flamegraph.pl > diff.svg
```

Frames are coloured by delta — red for grew, blue for shrank — so a regression appears as a red block instead of requiring you to eyeball two pictures side by side.

**Broken stacks — the gotcha that wastes afternoons.** If your flame graph is a field of one-frame-tall towers labelled `[unknown]`, stack unwinding failed. Causes, in order of likelihood:

1. **No frame pointers.** Binaries compiled with `-fomit-frame-pointer` (the default at `-O2` for years, and standard across distro packages) leave `perf -g` nothing to walk. Fixes: `--call-graph dwarf` (heavier, see §4's data-rate math), `--call-graph lbr` on Intel (cheap but limited depth), or rebuild with `-fno-omit-frame-pointer`. Fedora 38+ and Ubuntu 24.04+ now ship distro binaries *with* frame pointers by default precisely because of this problem.
2. **JIT-compiled code.** A JVM or V8 process's hot code has no ELF symbols at all, because it was generated at runtime. The convention is a **`/tmp/perf-<pid>.map`** file the runtime writes mapping address ranges to names; perf reads it automatically. For the JVM, `-XX:+PreserveFramePointer` (added to OpenJDK after exactly this problem was hit at Netflix, see Real-world use cases) plus a `perf-map-agent` is the standard combination.
3. **Missing symbols/debuginfo.** Stripped binaries give you addresses. Install the distro's `-dbg`/`-debuginfo` package, or profile a build that has symbols.
4. **Containers.** The process's binaries live in another mount namespace, so perf on the host cannot resolve them by path. Profile from the host with the container's filesystem reachable (`/proc/<pid>/root/...`), or run `perf` in the container's namespaces with `nsenter`.

### 8. Off-CPU analysis: the half of wall-clock perf cannot see

A CPU profiler samples what is *running*. A thread blocked in `read()` on a slow NFS mount is not running, contributes zero samples, and is therefore **invisible** in a flame graph — while consuming 100% of the user's wall-clock and, on a GPU node, 100% of the GPU's idle time. This is not a limitation to work around; it is the definition of what a CPU profiler measures.

**The mechanism.** Off-CPU analysis hooks the scheduler instead of a timer. When `sched_switch` moves a thread off the run queue, record `(tid → timestamp, stack)`. When the thread is switched back in, compute the delta and add it to that stack's running total. Deterministic, not statistical: every blocking event is captured with the stack that led into it. The cost is proportional to context-switch rate, which is why this is normally a short capture rather than a permanent one — and why the same mechanism, aggregated more aggressively, is what Netflix left running fleet-wide for noisy-neighbour detection.

```
# BCC's tool: aggregate off-CPU stacks for 15 seconds, folded output
$ sudo offcputime-bpfcc -f 15 > offcpu.folded

# Same idea in bpftrace (lesson 08), kernel stacks only:
$ sudo bpftrace -e '
  tracepoint:sched:sched_switch {
      @start[args.prev_pid] = nsecs; @stack[args.prev_pid] = kstack;
  }
  tracepoint:sched:sched_switch /@start[args.next_pid]/ {
      @off_us[@stack[args.next_pid]] = sum((nsecs - @start[args.next_pid]) / 1000);
      delete(@start[args.next_pid]); delete(@stack[args.next_pid]);
  }'

# Render as a flame graph — blue by convention, width = time BLOCKED
$ ./FlameGraph/flamegraph.pl --color=io --title="Off-CPU Time" \
      offcpu.folded > offcpu.svg
```

Reading an off-CPU flame graph is the same skill with a different axis meaning: **width is time spent blocked**, and the leaf tells you *what the thread was waiting for*:

| Leaf frame you will actually see | What it means |
|---|---|
| `io_schedule` / `folio_wait_bit` | Waiting for a block I/O to complete — storage latency |
| `nfs_wait_on_request`, `rpc_wait_bit_killable` | Waiting on the NFS client / server |
| `futex_wait` | Userspace lock contention (mutex, Python GIL) |
| `do_epoll_wait`, `poll_schedule_timeout` | Idle event loop — usually *not* a problem, just waiting for work |
| `pipe_read` / `sock_recvmsg` | Waiting on another process or peer |
| `schedule_timeout` from `msleep` | The application is literally sleeping |

That table names the trap too: **an idle thread parked in `epoll_wait` will dominate an off-CPU flame graph and mean nothing.** Off-CPU time is only interesting when a thread is blocked *on the critical path of work you care about*. Two ways to handle it: filter by task state so you only capture uninterruptible sleep (`offcputime -f --state 2` selects `TASK_UNINTERRUPTIBLE`, i.e. D-state — real I/O blocking rather than voluntary waiting), or restrict to the pid of the process under investigation and read the tower that corresponds to its work loop.

Two related measurements, so you keep them distinct:

- **Off-CPU time** = time not on the run queue at all (blocked). Tool: `offcputime`.
- **Run-queue latency** = time *runnable but waiting for a CPU* — scheduler delay, the signature of CPU saturation or cgroup throttling. Tool: `runqlat` (a histogram of wait time), or `/proc/pressure/cpu`. On a throttled pod (lesson 03) this is where the missing time shows up: the thread is neither blocked nor running.

Together with an on-CPU profile these three account for a thread's whole wall-clock: `wall = on-CPU + runqueue-wait + off-CPU-blocked`. When the wall-clock of a training step does not match its CPU time, one of the other two terms is your incident.

### 9. ftrace: the kernel's own tracer, hands-on

ftrace has been in the kernel since 2.6.27, needs no compiler, no BTF, and no capability beyond root, and it is the substrate that `perf`'s tracepoint support, kprobes and `trace_printk`/`bpf_printk` output all ride on. You interact with it by reading and writing files under **`/sys/kernel/tracing`** (tracefs; on older systems `/sys/kernel/debug/tracing`, which is the same filesystem mounted elsewhere). Everything is `echo`.

| File | Reading it gives | Writing to it does |
|---|---|---|
| `available_tracers` | `function function_graph blk wakeup wakeup_rt irqsoff hwlat nop …` | — |
| `current_tracer` | The active tracer | Selects it: `echo function_graph > current_tracer` |
| `tracing_on` | `0`/`1` | `0` pauses writing to the ring buffer without tearing anything down |
| `trace` | The captured buffer, formatted, non-destructive | `echo > trace` clears the buffer |
| `trace_pipe` | Blocking stream; **consumes** as you read | — |
| `available_filter_functions` | Every function that can be traced (~50k lines) | — |
| `set_ftrace_filter` | The current allow-list | Restricts tracing to these functions; accepts globs (`vfs_*`), `:mod:nvidia`, and triggers like `schedule:stacktrace` |
| `set_ftrace_notrace` | The deny-list | Excludes noisy functions |
| `set_ftrace_pid` | PIDs traced | Restrict to one process |
| `set_graph_function` | Graph roots | Trace only this call subtree with `function_graph` |
| `max_graph_depth` | Depth limit | `echo 3 >` keeps output readable |
| `tracing_thresh` | Latency floor, µs | With `function_graph`, record only calls slower than this |
| `events/<subsys>/<event>/enable` | `0`/`1` | Enable one tracepoint (`events/block/block_rq_issue/enable`) |
| `events/<subsys>/<event>/filter` | Filter expression | `echo 'bytes > 65536' >` — filter *in the kernel*, before the record is written |
| `set_event` | Enabled events | `echo 'sched:sched_switch' >` as a shorthand |
| `buffer_size_kb` | Per-CPU ring size | Grow it before a burst capture; on a 128-CPU box this is per-CPU, so ×128 |
| `trace_marker` | — | Userspace writes a string into the trace timeline — the way to correlate an application phase with kernel events |
| `options/*` | One file per boolean option | `echo 1 > options/func_stack_trace` adds a stack to every function event |
| `per_cpu/cpuN/trace` | That CPU's buffer alone | — |
| `instances/<name>/` | A **complete independent tracer** with its own buffer and settings | `mkdir instances/mine` creates one — this is how two tools trace at once without fighting |
| `kprobe_events` / `uprobe_events` | Dynamic probe definitions | Create ad-hoc probes without BPF |

**The function tracer** records every call to the selected functions:

```
# cd /sys/kernel/tracing
# echo nop > current_tracer          # reset
# echo 'vfs_read' > set_ftrace_filter
# echo function > current_tracer
# echo 1 > tracing_on ; sleep 1 ; echo 0 > tracing_on
# head -12 trace
# tracer: function
#
# entries-in-buffer/entries-written: 14009/14009   #P:128
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |
        python3-48213   [017] ....  918273.117402: vfs_read <-ksys_read
        python3-48213   [017] ....  918273.117661: vfs_read <-ksys_read
```

The four flag columns are the latency-format fields: interrupts off (`d`), reschedule needed (`N`), hardirq/softirq context (`h`/`s`), and preempt-depth. `.` means "none of the above" — normal process context. `FUNCTION <-parent` gives you the caller for free, which is often the whole answer ("who is calling this?").

**The function-graph tracer** is the one worth knowing well, because nothing else gives you a *timed, nested kernel call tree*:

```
# echo nop > current_tracer
# echo vfs_read > set_graph_function
# echo 3 > max_graph_depth
# echo function_graph > current_tracer
# echo 1 > tracing_on ; sleep 0.2 ; echo 0 > tracing_on
# head -14 trace
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 17)               |  vfs_read() {
 17)   0.412 us    |    rw_verify_area();
 17)               |    filemap_read() {
 17) ! 214.882 us  |      filemap_get_pages();
 17)   1.204 us    |      copy_page_to_iter();
 17) ! 219.331 us  |    }
 17) ! 220.108 us  |  }
 17)               |  vfs_read() {
 17)   0.388 us    |    rw_verify_area();
 17)   2.104 us    |    filemap_read();
 17)   3.002 us    |  }
```

Two `vfs_read` calls, 220 µs and 3 µs. The slow one spent 214.882 µs inside `filemap_get_pages` — the page-cache lookup that had to go to the device — while the fast one was a cache hit. **That is a cache-miss diagnosis with per-call timing, from three `echo`s and no tooling.** The duration markers are the kernel's own severity legend (from `kernel/trace/trace_output.c`):

| Marker | Duration |
|---|---|
| (space) | ≤ 10 µs |
| `+` | > 10 µs |
| `!` | > 100 µs |
| `#` | > 1,000 µs (1 ms) |
| `*` | > 10 ms |
| `@` | > 100 ms |
| `$` | > 1 s |

Scanning a function-graph trace for `#`, `*`, `@`, `$` is a fast visual grep for outliers. Better still, make the kernel do it: `echo 1000 > tracing_thresh` records only calls over 1 ms, turning an unusable firehose into a short list.

**Always clean up.** ftrace state is global and persistent: a forgotten `function` tracer on a busy node keeps costing CPU after you log out. The reset sequence is `echo nop > current_tracer; echo > set_ftrace_filter; echo > set_graph_function; echo 0 > tracing_thresh; echo > trace`. Better habit: work inside `mkdir /sys/kernel/tracing/instances/mine` and `rmdir` it when done — an instance has its own buffer and settings and takes everything with it.

**The wrappers.** You rarely need to type all that: `trace-cmd record -p function_graph -g vfs_read -- sleep 5` then `trace-cmd report` does the same thing with a saved `trace.dat`; `perf-tools`' `funcgraph vfs_read`, `funccount 'vfs_*'`, and `iolatency` are shell wrappers over exactly these files. Knowing the files matters anyway, because on a locked-down node the wrappers are what is missing, not the kernel feature.

**When ftrace beats eBPF:** the kernel is too old or too locked down for BPF; you need the *exhaustive* call tree rather than an aggregate; you need `trace_marker` to correlate application phases with kernel events; or you are debugging the BPF tooling itself. **When eBPF beats ftrace:** anything you want to leave attached, anything needing in-kernel aggregation or a histogram, anything where per-event records would overwhelm the buffer.

### 10. `perf_event_paranoid` versus container seccomp — two independent walls

Both present as "perf just doesn't work here," and fixing one while the other is in place leaves you exactly as stuck.

**Wall 1: the host sysctl.** `/proc/sys/kernel/perf_event_paranoid` gates *who* may call `perf_event_open()` and what they may observe. Default is 2.

| Value | Unprivileged users may |
|---|---|
| `-1` | Everything, no restrictions |
| `0` | Use per-process and per-CPU events; raw tracepoint access still restricted |
| `1` | Use per-process events; **no CPU-wide** (`-a`) events |
| `2` (default) | Per-process user-space profiling only; **no kernel profiling** |
| `3` (some distro kernels) | No unprivileged use at all |

Symptom of hitting it as a non-root user: `perf record` fails with `Access to performance monitoring and observability operations is limited` or returns a profile with no kernel frames. Fixes: run as root, grant `CAP_PERFMON` (5.8+) to the profiler, or `sysctl -w kernel.perf_event_paranoid=-1` on a lab box (do not do this casually on a shared production host — it exposes side-channel-relevant information to any user).

**Wall 2: the container's seccomp profile.** Docker's and containerd's default seccomp profiles **block the `perf_event_open` syscall entirely**, because it is a host-introspection surface. Inside such a container, `perf record` fails with `EPERM`/`Operation not permitted` no matter what the sysctl says and no matter that you are root *in the container*, because the syscall never reaches the kernel's permission check. Fixes: `--security-opt seccomp=unconfined` (or a custom profile allowing `perf_event_open`), plus the capability, plus `--privileged` or `CAP_SYS_ADMIN`/`CAP_PERFMON` — or, far more simply, **profile from the host**, which also fixes symbol resolution.

The practical rule on a Kubernetes node: **profile from the host, targeting the container's PID.** You get working symbols via `/proc/<pid>/root`, you avoid both walls, and you do not have to modify the workload's security context to debug it.

## Perspectives

**Kernel-mechanism view.** Every tool here reduces to one of two instrumentation strategies. **Statistical sampling** (`perf record`) programs a hardware counter to overflow at a chosen rate, takes an interrupt, and records the instruction pointer plus an unwound stack; the *distribution* of thousands of such samples estimates where cycles went, with a cost that depends only on the sample rate. **Event hooking** (ftrace, eBPF, off-CPU analysis via `sched_switch`) runs code at a specific kernel event, with a cost proportional to the event rate but with no sampling error and full context. Sampling answers "where does the time go" for time that is *being spent*; event hooking answers "what happened, exactly, and how long did each occurrence take," including for time that is being *waited*. Choosing wrongly is the most common tooling mistake: sampling a rare event finds nothing, and exhaustively tracing a hot path costs more than the problem.

**Operator/SRE view.** The USE grid is the answer to "how do I know I'm done." Walk Utilization / Saturation / Errors for every resource, and one of three things is true: you found the bottleneck; you found a cell you have no instrument for (a genuine finding — go build it, which is how PSI ended up in your stack); or everything is clean and the problem is above the node — in the application, the network path, or the scheduler. All three outcomes are progress. The complementary habit is the 60-second first pass — `uptime`, `dmesg | tail`, `vmstat 1`, `mpstat -P ALL 1`, `iostat -xz 1`, `free -m`, `sar -n DEV 1`, `ss -s`, PSI — which is not a substitute for the grid but reliably tells you which row of it to start with.

**GPU-fleet/economics view.** This is the clearest cost lesson in the module because the arithmetic is unambiguous. An 8×H100 node at roughly $2.50/GPU-hour is **$20/hour** for the accelerators alone; a week of that is ~$3,360. If off-CPU analysis shows the trainers are blocked 70% of wall-clock on dataset reads, then ~$2,350 of that week bought nothing, and the fix (a faster mount, a prefetch depth, a different dataset format) is usually a one-line change. The tool that finds the bug is the same tool that sizes the invoice, which is what makes "we should fix the data loader" fundable rather than a preference. `top` reports this situation as "not much happening."

**Interview/methodology view.** Two distinct skills are tested under one heading and conflating them is a common failure. The first is **enumerative**: recite the USE grid cold for an unspecified "slow node," which tests whether your method is complete by construction. The second is **interpretive**: handed one flame graph, identify the hot frame, distinguish it from a tall-but-cheap tower, and state the fix — which tests whether you can read a concrete artifact. A candidate can ace either and fumble the other. Practice them separately: recite the grid from memory, and separately sit with real flame graphs (yours, ideally) until width-not-depth is reflexive. A third, rarer probe: "your flame graph is all `[unknown]` — now what?" The answer (frame pointers, DWARF, `/tmp/perf-<pid>.map`, container namespaces) is the fastest way to signal you have actually done this rather than read about it.

## Real-world use cases

- **[Netflix — "Java in Flames"](https://netflixtechblog.com/java-in-flames-e763b3d32166)** (Brendan Gregg & Martin Spier). Netflix could not profile their JVM fleet: `perf` flame graphs were mostly `[unknown]` frames because the JVM (like most `-O2` builds) omits frame pointers, and JIT-compiled Java methods had no symbols at all. The fixes they built and upstreamed are still the standard recipe: **`-XX:+PreserveFramePointer`** (prototyped by Gregg, subsequently a shipping JDK option) to make stacks walkable, plus a `perf-map-agent` writing `/tmp/perf-<pid>.map` so perf can name JIT-compiled frames — producing mixed-mode flame graphs with Java and kernel frames in one picture. What it shows: §7's broken-stack section is not a footnote; it is the reason production profiling of managed runtimes was effectively impossible until someone fixed the toolchain.
- **[Netflix — "Linux Performance Analysis in 60,000 Milliseconds"](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)**. The published first-60-seconds checklist — `uptime`, `dmesg | tail`, `vmstat 1`, `mpstat -P ALL 1`, `pidstat 1`, `iostat -xz 1`, `free -m`, `sar -n DEV 1`, `sar -n TCP,ETCP 1`, `top` — with the specific field to read in each. What it shows: the USE grid compressed into a runnable sequence, and evidence that a standardized opening sequence is what a large SRE organization actually ships to its engineers instead of intuition.
- **[Netflix — "Noisy Neighbor Detection with eBPF"](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd)**. The problem: on a multi-tenant container host, a co-tenant can degrade your latency without ever appearing in your own metrics — your process is neither using more CPU nor blocking on I/O, it is simply waiting longer to be scheduled. Netflix instrumented the same `sched_switch` hook that powers `offcputime`, but left it running permanently and aggregated it into per-container **run-queue latency** rather than off-CPU stacks, then verified the added cost with their own `bpftop` tool at under 600 ns per hook before fleet rollout. What it shows: the on-CPU/off-CPU pair has a third member — scheduler delay — and it is the term that catches noisy neighbours; also that "continuous" and "one-off profiling" are the same mechanism at different aggregation levels.

- **The `%util` misdiagnosis, as a recurring class.** This one is not a single company's postmortem but a pattern worth naming, because it is the most common wrong turn in disk investigations: an operator sees `%util` at 100% on an NVMe device, concludes the disk is maxed out, and requests faster hardware — while `aqu-sz` is 1.2 and `await` is 200 µs, meaning the device is idle 98% of the time by capability and the real bottleneck is elsewhere (single-threaded submission, `fsync` serialization, or a filesystem lock). The mechanism is in §2: `%util` was defined for single-request-at-a-time devices and measures "time with ≥1 request in flight," which stopped meaning "utilization" the moment devices gained deep parallel queues. What it shows: a metric can be perfectly accurate and still answer a question you are not asking — which is why the USE grid names the *field* to read, not just the tool.

## Worked example: a "slow node," USE then flame graph

**Ticket:** *"Training pods on node-47 are slow. GPU utilization looks bad."* Node is 2× 64-core, 8× H100, one local NVMe, dataset on NFS.

**Step 1 — 60 seconds of vitals, following the grid top to bottom.**

```
$ uptime
 load average: 250.11, 240.30, 190.02          # 128 cores; load ≫ cores, but see below

$ mpstat -P ALL 1 1 | tail -3
Average:  all  12.3  0.00  4.1  61.2  0.00  0.31  0.00  22.1
                ^usr       ^sys ^iowait                  ^idle
# CPU is NOT the bottleneck: only 12% user, 4% system. 61% iowait.

$ vmstat 1 3 | tail -2
 r  b   swpd   free  buff   cache  si so   bi   bo   in    cs us sy id wa st
 2 61      0 3.8G   22M   74G     0  0  4832  24  9821 21044 12  4 23 61  0
# r=2 (no CPU queue), b=61 (61 tasks in uninterruptible sleep) ← the signal

$ cat /proc/pressure/io
some avg10=88.20 avg60=84.11 avg300=71.03 total=182933944912
full avg10=71.50 avg60=68.02 avg300=57.44 total=140222110293
# 71.5% of the last 10s, NOTHING could run because of I/O.

$ cat /proc/pressure/cpu
some avg10=2.11 avg60=1.98 avg300=1.74 total=9922011
# CPU pressure negligible — confirms it is not a CPU problem.

$ iostat -xz 1
Device   r/s    rkB/s r_await rareq-sz  w/s  wkB/s w_await aqu-sz %util
nvme0n1   102   6528    0.31    64.00   88   5632    0.42   0.04   3.1
# The LOCAL disk is idle. So the I/O is going somewhere else — the NFS mount.

$ nfsstat -c | head -6
Client rpc stats:
calls      retrans    authrefrsh
44821903   118422     44821903
# 0.26% retransmits — the NFS server or the path is struggling.

$ dmesg -T | tail -3
[Sun Aug 17 09:22:14 2026] nfs: server nfs-store.storage not responding, still trying
```

**USE verdict after 90 seconds:** CPU utilization low, CPU saturation low, no CPU errors. Memory fine. Local disk utilization low. **Network/NFS: saturation high** (io PSI `full` 71.5%, 61 tasks in D-state, NFS retransmits, server-not-responding in `dmesg`). The load average of 250 is 61 D-state tasks plus normal runnables — high load with idle CPUs, exactly the lesson-01 case.

**Step 2 — pick the right profiler.** `%usr` is 12%: an on-CPU flame graph would be nearly empty and would tell you nothing. This is off-CPU territory. **The grid chose the tool, you did not guess.**

```
$ sudo offcputime-bpfcc -f -p $(pgrep -f train.py | head -1) 15 > offcpu.folded
$ ./FlameGraph/flamegraph.pl --color=io --title="node-47 off-CPU, 15s" \
      offcpu.folded > offcpu.svg
$ sort -t' ' -k2 -nr offcpu.folded | head -3
python3;_PyEval_EvalFrameDefault;...;read;vfs_read;nfs_file_read;nfs_wait_on_request;io_schedule  10482113
python3;_PyEval_EvalFrameDefault;...;cuStreamSynchronize;futex_wait                                1204882
python3;...;epoll_wait                                                                              884201
```

The folded file is already the answer: **10.48 seconds of blocked time out of a 15-second capture, in one stack, ending in `nfs_wait_on_request → io_schedule`.** As a flame graph it is one dominant blue tower. The second entry (1.2 s in `futex_wait` under `cuStreamSynchronize`) is the *consequence* — the training thread waiting on the GPU stream, which is idle because no batch has arrived.

**Step 3 — quantify the cost.** Off-CPU 10.48 s of 15 s = **69.9% of wall-clock blocked on NFS**. Assuming GPU work is gated on those batches:

```
  GPU idle fraction        ≈ 0.70
  8 × H100 at ~$2.50/GPU-hr = $20.00/hr for the node's accelerators
  wasted per hour           = 0.70 × $20.00        = $14.00/hr
  per day                   = $14.00 × 24          = $336/day
  per 7-day training run    = $336 × 7             ≈ $2,352
```

*(GPU-hour price is the variable input — substitute your own committed rate. The structure of the calculation is the transferable part.)*

**Step 4 — confirm the mechanism before proposing a fix.** Is it latency per request or request volume?

```
$ sudo bpftrace -e '
  kprobe:nfs_file_read { @s[tid] = nsecs; }
  kretprobe:nfs_file_read /@s[tid]/ { @ms = hist((nsecs - @s[tid]) / 1000000);
                                      delete(@s[tid]); }'
@ms:
[0]                 1204 |@@@                                     |
[1]                 8210 |@@@@@@@@@@@@@@@@@@@@                    |
[2, 4)             20118 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[64, 128)            402 |@                                       |
[128, 256)           118 |                                        |
```

A bimodal distribution: a fat body at 2–4 ms plus a tail past 128 ms. And read size, from the syscall side, is 4 KiB. So this is **many small, individually-slow reads**, not a few big ones — the classic "dataset of millions of small files on NFS" shape.

**Verdict, one line:** *node-47's trainers spend ~70% of wall-clock off-CPU blocked on 4 KiB NFS reads averaging 2–4 ms with a 128 ms+ tail; ~$14/hour of GPU time is being burned waiting for the dataset. Fix: increase read size / prefetch depth in the loader, pack the dataset into large sequential shards (WebDataset/TFRecord-style), or stage it onto the idle local NVMe. This is a storage-path incident, not a compute problem — a CPU flame graph would have been empty.*

**The contrast case, for calibration.** Had `%usr` been 90% with io PSI near zero, the same grid would have sent you to `perf record -F 99 -a -g -- sleep 30` and an **on-CPU** flame graph, where you would expect a wide leaf plateau — a tokenizer, a JPEG decode, a `memcpy`, a Python interpreter loop — and the fix would be in that function. **Which flame graph you generate is decided by the grid, not guessed.**

## Practice (feeds the deliverable toolkit)

1. **On-CPU profile and flame graph.** Pick a CPU-bound process (`openssl speed`, `stress-ng --cpu 4`, or a real service under load):
   ```
   sudo perf record -F 99 -g -p <pid> -- sleep 20
   sudo perf script > cpu.stacks
   ./FlameGraph/stackcollapse-perf.pl cpu.stacks | ./FlameGraph/flamegraph.pl > cpu-flame.svg
   ```
   Open the SVG, find the widest top-edge frame, and write one sentence naming the hot function. If the towers are `[unknown]`, fix it (§7) and record what the fix was — that is the more valuable artifact.
2. **Off-CPU profile.** Pick an I/O-blocked process (`dd` on a slow or network-backed disk, or `fio --rw=randread --direct=1`) and produce an off-CPU flame graph with `offcputime-bpfcc -f` (or the bpftrace equivalent from §8) → `flamegraph.pl --color=io`. Identify the blocking stack and name what it was waiting for.
3. **ftrace by hand, no wrappers.** Using only `echo` and `cat` under `/sys/kernel/tracing`, produce a `function_graph` trace of `vfs_read` limited to depth 3, with `tracing_thresh` set so only calls over 1 ms appear. Capture ten lines of output and explain one duration marker. Then reset the tracer state (`echo nop > current_tracer`, clear the filters) and verify with `cat current_tracer`.
4. **Sampling arithmetic.** For your task-1 profile, compute how many samples landed in your hot frame and its relative standard error using `sqrt((1−p)/(N·p))`. State the smallest frame width your profile can resolve at ~10% error, and whether any conclusion you drew was below that threshold.
5. **Write the USE checklist for an unknown-slow node** — the exact commands per resource (CPU, memory, disk, network, plus GPU and cgroup caps if you have them), the one field to read in each, and a healthy-vs-alarm threshold for each field.

**Acceptance (into the deliverable's diagnostic toolkit):** commit (a) **one flame graph SVG** (on-CPU or off-CPU) with a caption stating what it revealed and how you know, (b) a **written USE walkthrough of one investigation** — the grid you ran, the outputs that localized it, the one-line verdict — and (c) the **ftrace transcript from task 3** with your reading of it. All three must stand alone: a teammate should be able to re-run the checklist and read the artifacts without you. See the deliverable spec: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md).

## Common pitfalls

1. **Diagnosing "slow" without enumerating resources first.** Pattern-matching to a symptom you have seen before skips the step that makes the method valuable — walking the grid *before* forming a hypothesis. The mechanism of the failure: a hypothesis formed early determines which tools you run, and every tool you run then confirms or fails to disconfirm that one hypothesis, so you never look at the resource that is actually saturated.
2. **Trusting `iostat` `%util` as saturation on NVMe.** `%util` measures the fraction of time at least one request was in flight — a real utilization for a single-request-at-a-time disk, and nearly meaningless for a device with thousands of concurrent queue entries, which can read 100% while at 2% of capability. Use `aqu-sz`, `await`, and io PSI, and cross-check with Little's Law.
3. **Reading a flame graph by height.** Depth is call-chain length. A tall narrow tower was sampled rarely and costs little; a wide flat top frame is where the CPU actually is. The reason is structural: the picture is built by *folding identical stacks and sizing by count* (§7), so width literally is the sample count and height is literally just how many frames were on the stack.
4. **Profiling on-CPU when the process is blocked.** If `%usr` is 5% and io PSI is 70%, an on-CPU flame graph is mostly the idle task and tells you nothing. The symptom of this mistake is a profile dominated by `intel_idle`/`swapper`, or a nearly empty one. Switch to off-CPU analysis, and use the grid to decide *before* recording.
5. **Reading an off-CPU flame graph without filtering idle waits.** Threads parked in `epoll_wait`, `futex_wait` on an idle worker pool, or `schedule_timeout` from a sleep loop will dominate by width and mean nothing. Filter to D-state (`--state 2`) or to the pid whose latency you care about, and always ask "was this thread blocked on the critical path?"
6. **Running `perf` inside a container.** Two independent walls (§10): the `perf_event_paranoid` sysctl and the runtime's default seccomp profile, which blocks `perf_event_open` outright. Fixing one leaves the other. Symbol resolution also breaks because the binaries are in another mount namespace. Profile from the host, targeting the container's PID.
7. **Using round sampling frequencies.** 100 Hz can phase-lock with periodic kernel or application activity, systematically over- or under-sampling the same code. Use 99 Hz. And check `dmesg` for `perf: interrupt took too long ... lowering kernel.perf_event_max_sample_rate` — if it fired, your profile was taken at a lower rate than you asked for.
8. **Leaving ftrace running.** Tracer state is global and survives your shell. A forgotten `function` tracer on a busy node costs real CPU indefinitely and looks like an unexplained regression to whoever finds it next. Use `instances/` and clean up.

## Self-check

**(a) Walk the USE method for a node reported "slow" — what do you check, for which resources, in what order?**
**Answer:** For every resource, check Utilization, Saturation, and Errors. Enumerate resources first — CPU, memory capacity and bandwidth, disk I/O and capacity, network interface and controller, interconnects, plus GPU/NVLink and cgroup caps on a real fleet, plus software resources (fds, pids, conntrack). Then walk them: **CPU** — `mpstat -P ALL 1` per-core `%usr`+`%sys` for utilization, `vmstat` `r` vs core count and `/proc/pressure/cpu` and `runqlat` for saturation, MCE/EDAC in `dmesg` for errors; **memory** — `free`/`MemAvailable` for utilization, `vmstat` `si`/`so`, `/proc/pressure/memory` and OOM kills for saturation, ECC counters for errors; **disk** — `iostat -xz 1` for throughput and (cautiously) `%util`, `aqu-sz`/`await`/`/proc/pressure/io` for saturation, `smartctl`/`dmesg` for errors; **network** — `sar -n DEV 1` against link speed, drops and `nstat` retransmits, interface error counters. Saturation is the signal most often missed: 100% utilization with no queue is fine, 70% with a deep queue is not, because response time goes as `S/(1−U)` and blows up between 90% and 99%. An empty cell in the grid is itself a finding. PSI is the cleanest single saturation instrument because it measures lost time rather than busy time.

**(b) What does a wide flat frame at the top of a flame graph indicate versus a tall narrow tower — and why?**
**Answer:** Width is the number of folded samples containing that frame, i.e. the fraction of CPU time; the x-axis is the sorted *population* of stacks, not time, and adjacency means "sorts next to," never "happened next." The top edge of a tower is the function that was actually on-CPU when the sample was taken. So a **wide flat top frame** is a leaf burning cycles across many samples — the genuine hot spot and the optimization target. A **tall narrow tower** is a deep call chain that was sampled rarely: many frames, little time. Height is call-chain depth and carries no cost information. You optimize width. The reason this is true rather than conventional is the construction: `stackcollapse` merges identical stacks and `flamegraph.pl` sizes each box by its merged count.

**(c) On-CPU vs off-CPU analysis — which for which symptom, and what is the third measurement?**
**Answer:** **On-CPU** (`perf record -F 99 -g` → flame graph; timer/PMU sampling of IP+stack) answers "where are cycles going" — use it when `%usr`/`%sys` is high. **Off-CPU** (hook `sched_switch`, record the stack and the duration a thread is off the run queue; `offcputime` → `flamegraph.pl --color=io`) answers "why is this thread not running" — use it when wall-clock is high but CPU is idle: high `%iowait`, high io/memory PSI, tasks in D-state. The **third** is run-queue latency (`runqlat`, `/proc/pressure/cpu`): time *runnable but waiting for a CPU*, which is what CPU saturation and cgroup throttling look like. Together they decompose wall-clock: `wall = on-CPU + runqueue-wait + off-CPU-blocked`. The USE grid tells you which term to go measure; guessing is how you end up with an empty flame graph.

**(d) Your flame graph is a field of `[unknown]` frames one level deep. Diagnose it.**
**Answer:** Stack unwinding failed. In order of likelihood: (1) **no frame pointers** — the binary was built with `-fomit-frame-pointer`, so `perf -g`'s default frame-pointer walk has nothing to follow; fix with `--call-graph dwarf` (heavier — 8 KiB of stack copied per sample by default, ~50 MB/s system-wide at 99 Hz on 64 CPUs), `--call-graph lbr` on Intel, or rebuild with `-fno-omit-frame-pointer` (Fedora 38+/Ubuntu 24.04+ ship distro binaries with them). (2) **JIT-compiled code** with no ELF symbols — the runtime must write `/tmp/perf-<pid>.map`; for the JVM add `-XX:+PreserveFramePointer` and `perf-map-agent`, the fix Netflix upstreamed. (3) **Stripped binaries** — install debuginfo. (4) **Containers** — the binaries are in another mount namespace, so profile from the host with `/proc/<pid>/root` reachable.

**(e) How do you choose a sampling frequency and duration, quantitatively?**
**Answer:** Sampling is Bernoulli, so a frame at true share `p` gets `N·p` of `N` samples with relative standard error `sqrt((1−p)/(N·p))` — meaning you need roughly `100/p` samples to resolve a frame of share `p` at ~10% error. At `-F 99 -a` on 64 CPUs for 30 s with the node half-busy you get ~95,000 in-workload samples: a 1% frame has ~950 samples (~3% error, trustworthy), a 0.1% frame has ~95 (~10%, directional), a 0.01% frame is noise. On cost, frame-pointer sampling is ~1–2 µs per sample, so 99 Hz × 64 CPUs ≈ 0.015% of the node; DWARF unwinding instead copies 8 KiB of stack per sample, ~50 MB/s, which fills disks and can trip the kernel's `perf_cpu_time_max_percent` guard (default 25%) into halving your `perf_event_max_sample_rate` (default 100000) with an "interrupt took too long" message. Use 99 rather than 100 Hz to avoid phase-locking with periodic activity.

**(f) When would you reach for ftrace instead of eBPF or perf, and what exactly would you type?**
**Answer:** When the kernel is too old or too locked down for BPF; when you need an *exhaustive, timed, nested* call tree rather than a sampled distribution or an aggregate; when you want `trace_marker` to correlate an application phase with kernel events; or when the BPF tooling itself is the suspect. Concretely, to see where time goes inside `vfs_read`: `cd /sys/kernel/tracing`, `echo vfs_read > set_graph_function`, `echo 3 > max_graph_depth`, `echo 1000 > tracing_thresh` (only calls over 1 ms), `echo function_graph > current_tracer`, `echo 1 > tracing_on`, then `cat trace`. Output is an indented call tree with per-function durations and severity markers (`+` >10 µs, `!` >100 µs, `#` >1 ms, `*` >10 ms, `@` >100 ms, `$` >1 s). Clean up with `echo nop > current_tracer` and clear the filters — ftrace state is global and persists after you log out. Prefer eBPF when you want something left attached or aggregated in-kernel; prefer perf when you do not yet know which function to look at.

## Connections & what's next

This lesson is the module's methodology spine. Lesson 01 explained why load average can exceed core count (D-state tasks — the exact signature in this lesson's worked example); lesson 03 gave you cgroup throttling, which shows up here as run-queue latency rather than blocked time; lesson 04 gave you PSI, the saturation instrument the USE grid was always asking for; lesson 08 gave you the `sched_switch` hooking that off-CPU analysis is built on. Everything here is the practiced application of those pieces under a single checklist.

Next: **[10 — systemd as cgroup manager, and the block-I/O path](10-systemd-cgroups-and-block-io.md)** takes the storage thread from the worked example one layer deeper — who owns the cgroup tree that both encloses and can bound that I/O, how `io.max`, `io.weight` and `io.latency` act on the block layer, and how per-cgroup `io.pressure` tells you *whose* I/O is stalling *whom* on a shared device. The off-CPU flame graph that named `nfs_wait_on_request` here is the same investigation picked up from the cgroup-attribution side.

## References & further reading

**Primary sources**
- [Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html) — the canonical statement: definitions of utilization/saturation/errors, the resource-enumeration procedure, and the flowchart. Read it as the checklist you internalize.
- [Brendan Gregg — USE Method: Linux Performance Checklist](https://www.brendangregg.com/USEmethod/use-linux.html) — the per-resource tool/field table this lesson's grid is drawn from, including the software-resource rows.
- [Brendan Gregg — perf Examples](https://www.brendangregg.com/perf.html) — the `record`/`report`/`script`/`stat` reference, with worked invocations for most event types.
- [Kernel docs — `Documentation/trace/ftrace.rst`](https://docs.kernel.org/trace/ftrace.html) — the authoritative reference for every tracefs file and tracer in §9, including the function-graph output format and duration markers.
- [`perf-record(1)` man page](https://man7.org/linux/man-pages/man1/perf-record.1.html) — the flags in §6, including `--call-graph dwarf`'s default 8192-byte stack dump.
- [`perf_event_open(2)` man page](https://man7.org/linux/man-pages/man2/perf_event_open.2.html) — the syscall underneath, and the `perf_event_paranoid` levels reproduced in §10.

**Real-world engineering blogs**
- [Netflix — "Java in Flames"](https://netflixtechblog.com/java-in-flames-e763b3d32166) — what it shows: the production origin of the broken-stacks problem, and the `-XX:+PreserveFramePointer` / `perf-map-agent` fixes that became standard.
- [Netflix — "Linux Performance Analysis in 60,000 Milliseconds"](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55) — what it shows: the ten-command opening sequence and the one field to read in each, as actually deployed to an SRE org.
- [Netflix — "Noisy Neighbor Detection with eBPF"](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd) — what it shows: `sched_switch` hooking evolved from one-off off-CPU captures into standing per-container scheduler-latency telemetry.

**Deeper dives**
- [FlameGraph repo (brendangregg/FlameGraph)](https://github.com/brendangregg/FlameGraph) — `stackcollapse-perf.pl`, `flamegraph.pl`, `difffolded.pl`; the README covers the on-CPU, off-CPU (`--color=io`) and differential workflows in §7.
- [bcc tools](https://github.com/iovisor/bcc) — `offcputime`, `runqlat`, `biolatency`, `execsnoop`; read `tools/offcputime_example.txt` for the `--state` filtering discussed in §8, and prefer the `libbpf-tools/` rewrites where they exist.
- Brendan Gregg, *Systems Performance* (2nd ed.), methodology and CPU/disk chapters — the deep treatment tying USE, drill-down analysis, perf, and off-CPU analysis together.
