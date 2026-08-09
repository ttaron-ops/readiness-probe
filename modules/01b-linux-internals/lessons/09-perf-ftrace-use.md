---
lesson: "01b.9"
title: "perf, ftrace, and the USE Method"
module: "01b"
concept: "perf, ftrace, and the USE Method"
status: not-started
est_time: "6h"
artifacts: []
---

# 01b.9 · perf, ftrace, and the USE Method

> **Concept.** The USE method is a systematic checklist that turns "the node is slow" into a bounded investigation; perf and flame graphs are how you see *where* the CPU actually goes, on-CPU and off-CPU — together they are how you answer every performance debugging question, in an incident or an interview.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters
"A node is slow, walk me through how you'd debug it" is the single most common systems interview question, and unstructured answers ("I'd check top, then maybe iostat, then...") read as guessing. The USE method is a named, complete methodology — the interviewer is listening for whether you cover Utilization, Saturation, and Errors across every resource *in order*, so you never miss the disk while staring at CPU. And when the problem *is* CPU, "profile it and read the flame graph" is the concrete next step that separates engineers who theorize from engineers who measure. On a GPU fleet the stakes are literal money: an idle-but-blocked training node burning $30/hr of reserved H100 because it's off-CPU waiting on a slow dataset mount is a cost incident, and the tool that reveals "this process spends 80% of wall-clock *off* CPU in `nfs` reads" is off-CPU analysis, not `top`. Being fluent here is both the interview filter and the day-job differentiator you're building toward (cost/observability).

## From using to understanding
As an operator you read `top`, `htop`, `iostat`, `mpstat`, `sar` and form a hunch. You know `%us` vs `%sy` vs `%wa`, you've watched `iostat -x` `%util` climb, you've maybe run `perf top` once and seen a scrolling list of functions. Your diagnosis is pattern-matching against symptoms you've seen before.

What you're learning now is (1) a **methodology** that is complete by construction rather than pattern-matching, and (2) the tools to see *inside* CPU time instead of just measuring its total. The USE method is Brendan Gregg's resource-oriented checklist: for **every** resource, ask three questions and you cannot miss a class of problem. `perf` is the kernel's sampling profiler — it interrupts the CPU thousands of times a second, records the **stack trace** at each interrupt, and the distribution of those samples *is* where CPU time goes. A **flame graph** is the visualization that makes ten thousand stacks readable in one glance. And **off-CPU analysis** inverts the question: instead of "what is the CPU running," it asks "when a thread is *not* running, what is it blocked waiting on, and for how long" — which is the only way to explain latency that doesn't show up as CPU. `%wa` in `top` hints at it; off-CPU profiling names the exact stack.

## Core notes

### The USE method
For **every resource**, check three things:
- **Utilization** — the percent of time the resource was busy servicing work (or, for capacity resources like memory/disk space, the percent consumed).
- **Saturation** — the degree of *queued/waiting* work the resource could not service immediately. This is the one operators skip and it's usually the real signal: 100% utilization with no queue is fine; 70% utilization with a deep queue is a problem.
- **Errors** — error counts, which are often silent and precede failure (ECC memory errors, disk read errors, NIC drops/retransmits).

Enumerate resources first (a mental or written list): **CPU, memory, disk (I/O + capacity), network, and the interconnects/controllers** (plus, on your fleet, GPU and NVLink, and cgroup limits, which are per-tenant resource caps). Then walk the grid. The power is that it's **complete**: three questions × every resource means you physically cannot forget to check disk saturation while fixated on CPU. Where a metric isn't obvious, that gap is itself a finding ("I have no saturation metric for this resource" → go get one, e.g. PSI).

The modern refinement: **PSI (Pressure Stall Information)**, `/proc/pressure/{cpu,io,memory}`, gives you a *saturation* number directly — the `some` and `full` percentages are the fraction of time tasks stalled waiting on that resource. PSI is the cleanest single saturation instrument USE was always asking for.

### The USE grid you actually run
| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat -P ALL 1` (per-core %busy), `top` | `/proc/pressure/cpu`, run-queue latency (`runqlat`), load vs core count | `dmesg` (MCE), `perf stat` |
| Memory | `free -m`, `/proc/meminfo` | swap activity `vmstat 1` (si/so), `/proc/pressure/memory`, OOM kills in `dmesg` | EDAC/ECC `dmesg`, `/sys/devices/system/edac` |
| Disk | `iostat -xz 1` (`%util`) | `iostat` `aqu-sz`/`await`, `/proc/pressure/io` | `smartctl -a`, `dmesg` I/O errors |
| Network | `sar -n DEV 1`, `ip -s link` | `ss -s`, tx/rx queue drops, `nstat` retransmits | `ip -s link` (errors/drops), `ethtool -S` |

Note `iostat`'s `%util` is **not** true utilization on modern multi-queue NVMe (a device servicing many I/Os in parallel can show 100% `%util` while nowhere near saturated) — trust `await`/`aqu-sz` and PSI for saturation instead. This is exactly the kind of nuance an interviewer rewards.

### Reading the vitals: the fields that actually matter
Before profiling, the standard "USE first-pass" tools and the one number per tool that localizes the problem:
- **`vmstat 1`** — `r` (run-queue length: > core count means CPU saturation), `b` (blocked/uninterruptible: nonzero means I/O stall), `si`/`so` (swap in/out: nonzero means memory pressure spilling to disk — usually pathological), `us`/`sy`/`id`/`wa` (user/system/idle/iowait split). One line tells you which resource class to chase.
- **`mpstat -P ALL 1`** — the per-core breakdown `vmstat`'s average hides: one pegged core at 100% among 127 idle ones is a single-threaded bottleneck, invisible in aggregate.
- **`iostat -xz 1`** — per-device: `r/s`+`w/s` (IOPS), `rkB/s`+`wkB/s` (throughput), `await` (avg ms per I/O — the latency users feel), `aqu-sz` (avg queue depth — the saturation signal), `%util` (busy time, unreliable on NVMe — see above). Rising `await` with rising `aqu-sz` is a saturated disk.
- **`sar -n DEV 1` / `ss -s` / `nstat`** — network throughput, socket states, and retransmit counters (retransmits are network *saturation/error* — a rising `TcpRetransSegs` under load points at a lossy path or an overloaded peer).
- **`/proc/pressure/{cpu,io,memory}`** — PSI, the one-stop saturation number; watch `some avg10` for "is anything waiting."

### perf: on-CPU profiling
`perf` samples the instruction pointer + call stack on a timer (or on a hardware event) and tallies where the samples land.

```
# Record stacks system-wide at 99 Hz for 30s. -g = capture call graphs.
# 99 (not 100) Hz avoids lock-step sampling with periodic kernel activity.
sudo perf record -F 99 -a -g -- sleep 30

# Interactive summary: functions ranked by sample count (self + children).
sudo perf report --stdio

# Flatten each sample to one stack per line — the input a flame graph needs.
sudo perf script > out.stacks
```

`perf stat` is the complement — it counts hardware/software events for a command without sampling: `perf stat -d ./workload` gives IPC (instructions-per-cycle), cache misses, branch misses. Low IPC (<1) with high cache-miss rate is a memory-bound signature; high IPC that's still slow is genuinely compute-bound.

Requirements that bite in practice: you need **frame pointers** or **DWARF** unwinding for good stacks. Many distro binaries are built with `-fomit-frame-pointer`, giving broken/shallow stacks; use `perf record --call-graph dwarf` (heavier, copies stack memory) or rebuild with frame pointers (Fedora/Ubuntu now ship them by default on recent releases). For containers, run `perf` from the host with access to the target's symbols, or the stacks show as raw addresses.

### Flame graphs: reading them
A flame graph turns tens of thousands of sampled stacks into one picture. Generate it from `perf script` output with Gregg's FlameGraph tools:

```
git clone https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl out.stacks | ./FlameGraph/flamegraph.pl > cpu.svg
```

How to read it — this is the exam question:
- **x-axis is NOT time.** It's the *population of samples*, sorted alphabetically for merging. Width = **fraction of samples** a function (and its callees) was on-CPU = fraction of CPU time.
- **y-axis is stack depth.** The bottom frame is where sampling started (often the thread entry / kernel); each frame above is a caller→callee edge. The **top edge** of each tower is the function actually *on-CPU* when sampled.
- **A wide, flat frame at the top** = a leaf function consuming CPU directly across many samples. That's your hot spot — the code actually burning cycles. Optimize this.
- **A tall, narrow tower** = a deep call chain that's rarely sampled — lots of stack depth but little CPU. Deep ≠ expensive; narrow means it's not where the time goes. Don't chase it.
- Look for **unexpectedly wide plateaus**: a wide `malloc`/`memcpy`/lock-spin/`json.Unmarshal` frame names the fix.

Interactivity: the SVG is clickable (zoom into a subtree) and searchable (Ctrl-F highlights all frames matching a regex and sums their width — great for "how much total time is in `crypto`?").

### Off-CPU analysis
On-CPU profiling is blind to a thread that's *blocked* — sleeping on I/O, a lock, or a condition variable contributes zero CPU samples yet may dominate wall-clock latency. **Off-CPU analysis** measures the time threads spend *off* the run queue and captures the stack at the moment they blocked, so you see *why* they went to sleep.

The mechanism: hook the scheduler. When `sched_switch` moves a thread off-CPU, record the timestamp and its stack; when it comes back, add the delta to that stack's total. This is cheap and precise with eBPF (lesson 01b.8) — the BCC/bpftrace `offcputime` tool does exactly this and emits an **off-CPU flame graph** (same tooling, blue by convention) where width = time *blocked* rather than time on-CPU.

```
# Off-CPU stacks aggregated by blocked duration, 10s, then fold to a flame graph.
sudo offcputime-bpfcc -f 10 > offcpu.folded    # or: bpftrace kstack on sched_switch
./FlameGraph/flamegraph.pl --color=io --title="Off-CPU" offcpu.folded > offcpu.svg
```

A wide tower ending in `nfs_readpage`/`io_schedule`/`futex_wait` tells you the latency is storage or lock contention, not compute. This is the tool that catches the idle-but-expensive GPU node.

### perf record → report → script, the full loop
The three-verb workflow, and what each verb is *for*:
- **`perf record`** captures samples to a `perf.data` file. Key flags: `-F <hz>` sample rate, `-a` all-CPUs (system-wide), `-g` call graphs, `-p <pid>` / `-t <tid>` to target, `-e <event>` to sample on a specific hardware/software/tracepoint event instead of the default cycles (`-e block:block_rq_issue`, `-e cache-misses`). `--` runs a command and records for its lifetime; `sleep N` is the idiom for "record everything for N seconds."
- **`perf report`** reads `perf.data` into an interactive (or `--stdio`) ranked view: functions by **self** time (samples *in* that function) vs **children** (self + everything it called). Expand a call graph to see callers/callees. This is your first look before committing to a flame graph.
- **`perf script`** dumps the raw per-sample records — one event with its full stack — as text. This is the *export* format: pipe it to `stackcollapse-perf.pl` for flame graphs, or grep it for a specific event sequence. `perf annotate` goes the other direction, down to hot *instructions* within a function.

`perf list` enumerates every event the box exposes (hardware PMU counters, cache events, tracepoints, kprobes you've added). `perf top` is the live, no-file version — a continuously-updating `perf report`, the profiling analogue of `top`.

### ftrace / trace-cmd (awareness)
ftrace is the kernel's built-in tracing framework (interface under `/sys/kernel/tracing`, or `/sys/kernel/debug/tracing`), predating eBPF and always present. You rarely poke the raw filesystem; you use **`trace-cmd`** (CLI) or **`perf-tools`** wrappers (`funccount`, `funcgraph`, `iolatency`). Its standout capability is the **function graph tracer** — it traces kernel *function entry and exit with timing and call nesting*, printing an indented call tree with per-function durations. Where perf samples statistically, ftrace's function tracer is *exhaustive* for the functions you select — invaluable for "which kernel function inside this syscall is slow." Know it exists, know `trace-cmd record -p function_graph -g <fn>` and `funcgraph` are how you'd get a timed kernel call tree; reach for eBPF/bpftrace first for most tasks.

## Worked example: a "slow node," USE then flame graph
Ticket: "training pods on node-47 are slow." Run the USE grid top-down.

```
$ uptime
 load average: 250.11, 240.3, 190.0        # very high — but high load ≠ CPU (lesson 01b.1)
$ mpstat -P ALL 1 1
 all   12.3   0.0   4.1   61.2   ...        # %idle low, but %iowait (61%!) dominates, %usr only 12%
$ cat /proc/pressure/io
 some avg10=88.20 ...  full avg10=71.50     # IO pressure is the saturation signal — tasks stalling on I/O
$ iostat -xz 1
 nvme0n1  await 45.0  aqu-sz 32.0  %util 99  # deep queue, 45ms awaits — the disk is the bottleneck
```

USE has already localized it: **CPU is mostly idle in user code; the resource under saturation is disk I/O** (PSI `io full=71%`, `await` 45ms, queue depth 32). This is off-CPU territory — the threads aren't burning CPU, they're *blocked*. Confirm and name the blocking stack:

```
$ sudo offcputime-bpfcc -f 15 > offcpu.folded
$ ./FlameGraph/flamegraph.pl --color=io offcpu.folded > offcpu.svg
```

Reading `offcpu.svg`: one wide tower under the `trainer` threads bottoms out in `read → nfs_file_read → nfs_readpage → io_schedule`. Verdict, one line: *node-47's trainers spend ~70% of wall-clock off-CPU blocked on NFS reads of the dataset; this is a storage-path latency incident, not a compute problem — CPU flame graph would be near-empty.* For contrast, had `%usr` been 90% and PSI-io low, we'd instead run `perf record -F 99 -a -g` and read the **on-CPU** flame graph, expecting a wide leaf plateau (say `json.Unmarshal` or a tokenizer) as the fix. **Which flame graph you generate is decided by the USE grid**, not guessed.

### Why 99 Hz, frame pointers, and other gotchas that cost you in an interview
- **99 Hz, not 100.** Sampling at a round 100 Hz risks phase-locking with periodic kernel timers (also at round frequencies), systematically over- or under-counting periodic work. An off-by-one like 99 decorrelates the sampler. Small detail, but naming it signals you've actually profiled.
- **Broken stacks = broken flame graph.** If towers bottom out in `[unknown]` or a single frame, you're missing unwind info. Fixes, in order: rebuild the target with frame pointers (`-fno-omit-frame-pointer`); or `perf record --call-graph dwarf` (unwinds from copied stack memory — accurate but heavier and larger `perf.data`); or `--call-graph lbr` on Intel (uses the Last Branch Record, low overhead, limited depth). Interpreted/JIT runtimes (JVM, Node, Python) need a **symbol map** (`perf-<pid>.map`) or a runtime agent, or the frames show as raw addresses.
- **Containers hide symbols.** Run `perf` from the **host**; the target's binaries/debuginfo must be reachable (perf resolves via `/proc/<pid>/root`). Inside a container `perf` often lacks the perf_event syscall permission anyway.
- **`perf_event_paranoid`.** `/proc/sys/kernel/perf_event_paranoid` gates who can profile; on locked-down hosts you'll need root or a lowered value. Know it exists so "perf returns nothing" doesn't stump you.

## Practice (feeds the deliverable toolkit)
1. Pick a **CPU-bound** process (e.g. `openssl speed`, `stress-ng --cpu 4`, or a real service under load). Profile and flame-graph it:
   ```
   sudo perf record -F 99 -g -p <pid> -- sleep 20
   sudo perf script > cpu.stacks
   ./FlameGraph/stackcollapse-perf.pl cpu.stacks | ./FlameGraph/flamegraph.pl > cpu-flame.svg
   ```
   Open the SVG, identify the widest top-edge frame, and write one sentence naming the hot function.
2. Pick an **I/O-blocked** process (e.g. `dd` on a slow/network disk, or a `fio --rw=randread` job) and do an **off-CPU** analysis with `offcputime-bpfcc -f` → flame graph. Identify the blocking stack.
3. Write a **USE checklist** for an unknown-slow node: the exact commands per resource (CPU/mem/disk/net), in order, with the one metric per cell you read and its "healthy vs alarm" threshold.

**Acceptance (into the deliverable's diagnostic toolkit):** commit (a) **one flame graph SVG** (on-CPU or off-CPU) with a caption stating what it revealed, and (b) a **written USE walkthrough of one investigation** — the resource grid you ran, the command outputs that localized it, and the one-line verdict. Both must stand alone: a teammate should be able to re-run your checklist and read your flame graph without you.

## Self-check
**(a) Walk the USE method for a node reported "slow" — what do you check, for which resources, in what order?**
**Answer:** For every resource, check Utilization, Saturation, and Errors. Enumerate resources — CPU, memory, disk (I/O and capacity), network (plus GPU/interconnect and cgroup limits on a real fleet) — and walk them: **CPU** — `mpstat -P ALL` for per-core %busy, `/proc/pressure/cpu` and run-queue latency for saturation, `dmesg`/MCE for errors; **memory** — `free`/`meminfo` for utilization, swap activity (`vmstat` si/so) and `/proc/pressure/memory` and OOM kills for saturation, ECC/EDAC for errors; **disk** — `iostat -xz` for %util but trust `await`/`aqu-sz` and `/proc/pressure/io` for saturation, `smartctl`/`dmesg` for errors; **network** — `sar -n DEV`/`ip -s link` for throughput, drops and `nstat` retransmits for saturation, interface error counters for errors. The order is resource-by-resource, and saturation (queuing) is the signal operators most often miss — 100% utilization without a queue is fine, moderate utilization with a deep queue is the problem. PSI is the cleanest single saturation instrument.

**(b) What does a wide flat frame at the top of a flame graph indicate versus a tall narrow tower?**
**Answer:** Width = fraction of samples (fraction of CPU time); the x-axis is population, not time, and the top edge is the function actually on-CPU. A **wide, flat top frame** is a leaf function burning CPU directly across many samples — the genuine hot spot, and your optimization target. A **tall, narrow tower** is a deep call chain that was rarely sampled — lots of stack depth but little CPU time; deep does not mean expensive, and narrow means it isn't where the time goes, so you don't chase it. You optimize width, not height.

**(c) On-CPU vs off-CPU analysis — which tool/method for which symptom?**
**Answer:** **On-CPU** analysis (perf record/report/script → flame graph, sampling the IP+stack on a timer) answers "where is the CPU spending cycles" — use it when the process is CPU-bound: high `%usr`/`%sys`, IPC and cache metrics implicating compute. **Off-CPU** analysis (scheduler `sched_switch` hooks, e.g. eBPF `offcputime` → off-CPU flame graph, measuring time blocked and the stack at block time) answers "why is a thread *not* running" — use it when wall-clock latency is high but CPU is idle: high `%iowait`, IO/memory PSI, threads in D-state or on locks. Symptom-to-tool: CPU pegged → on-CPU perf; slow but idle / blocked on I/O or locks → off-CPU. The USE grid tells you which one to reach for.

## Resources
1. **Brendan Gregg — [The USE Method](https://www.brendangregg.com/usemethod.html)** and **[perf examples](https://www.brendangregg.com/perf.html)** — the methodology and the perf command reference; treat the USE page as the checklist you internalize.
2. **[FlameGraph repo](https://github.com/brendangregg/FlameGraph)** — `stackcollapse-perf.pl` + `flamegraph.pl`; read the README for the on-CPU and off-CPU (`--color=io`) workflows.
3. **Gregg, *Systems Performance* (2nd ed.), methodology chapter** — the deep treatment tying USE, drill-down analysis, perf, and off-CPU together; the book to own for this domain.
