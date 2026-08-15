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
sources: 7
---

# 01b.9 · perf, ftrace, and the USE Method

> **Concept.** The USE method is a systematic checklist that turns "the node is slow" into a bounded investigation; perf and flame graphs are how you see *where* the CPU actually goes, on-CPU and off-CPU — together they are how you answer every performance debugging question, in an incident or an interview.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 8 gave you eBPF as the observability substrate — programs you attach to kernel hooks (syscalls, tracepoints, `sched_switch`) without modifying the kernel, verified safe before they run. This lesson is where that substrate earns its keep as a *methodology*: perf's sampling profiler and the off-CPU tools you'll run here (`offcputime-bpfcc`) are themselves eBPF/kprobe-based instrumentation, and the USE method is the checklist that tells you *which* instrument to reach for and *when*. Where lesson 8 answered "how do I safely hook the kernel," this lesson answers "given a slow node and every tool at your disposal, what do I actually check, in what order, and how do I read what comes back." It unlocks the block-I/O and cgroup-pressure diagnosis in lesson 10 — io PSI is the USE-method saturation instrument for disk, and `biolatency` is the disk-specific cousin of the latency histograms you'll build here.

## Why this matters

"A node is slow, walk me through how you'd debug it" is the single most common systems interview question, and unstructured answers ("I'd check top, then maybe iostat, then...") read as guessing. The USE method is a named, complete methodology — the interviewer is listening for whether you cover Utilization, Saturation, and Errors across every resource *in order*, so you never miss the disk while staring at CPU. And when the problem *is* CPU, "profile it and read the flame graph" is the concrete next step that separates engineers who theorize from engineers who measure. On a GPU fleet the stakes are literal money: an idle-but-blocked training node burning $30/hr of reserved H100 because it's off-CPU waiting on a slow dataset mount is a cost incident, and the tool that reveals "this process spends 80% of wall-clock *off* CPU in `nfs` reads" is off-CPU analysis, not `top`. Being fluent here is both the interview filter and the day-job differentiator you're building toward (cost/observability).

## What's new here (calibration)

Per the [module README](../README.md)'s calibration, you already know shell pipelines, `top`/`iostat`/`sar` as commands, and general Linux administration — none of that is re-taught here. What's genuinely new:

- Treating "check the usual commands" as a **named, complete methodology** (USE) rather than an ad-hoc habit — the difference between "I checked some stuff" and "I can prove I didn't miss a resource class."
- Reading a **flame graph** correctly — width-not-depth, x-axis-is-population-not-time — which is a distinct, learnable skill most operators have never been taught explicitly even if they've glanced at one.
- **Off-CPU analysis** as a first-class technique with its own tooling, not just an intuition ("must be I/O-bound") — you'll produce the actual blocked-stack evidence.
- The GPU-fleet economics angle: translating "off-CPU wall-clock" directly into "wasted reserved-GPU dollars," which is the framing that makes this lesson a cost lesson, not just a debugging lesson.

## Core concepts

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

### Differential flame graphs
A single flame graph shows *where* time goes; a **differential** flame graph shows *what changed*. Take two folded-stack files (before/after a deploy, or the same workload on two different nodes) and diff them with `difffolded.pl` from the FlameGraph repo — it produces a folded file where each frame is colored by whether it grew (red) or shrank (blue) between the two profiles, then feed that into `flamegraph.pl` as usual. This is the tool for "did the last release regress CPU cost" or "why is node-12 slower than node-13 running the identical workload" — instead of eyeballing two SVGs side by side, you get one picture where the *delta* is the signal.

### Off-CPU analysis
On-CPU profiling is blind to a thread that's *blocked* — sleeping on I/O, a lock, or a condition variable contributes zero CPU samples yet may dominate wall-clock latency. **Off-CPU analysis** measures the time threads spend *off* the run queue and captures the stack at the moment they blocked, so you see *why* they went to sleep.

The mechanism: hook the scheduler. When `sched_switch` moves a thread off-CPU, record the timestamp and its stack; when it comes back, add the delta to that stack's total. This is cheap and precise with eBPF (lesson 01b.8) — the BCC/bpftrace `offcputime` tool does exactly this and emits an **off-CPU flame graph** (same tooling, blue by convention) where width = time *blocked* rather than time on-CPU.

```
# Off-CPU stacks aggregated by blocked duration, 10s, then fold to a flame graph.
sudo offcputime-bpfcc -f 10 > offcpu.folded    # or: bpftrace kstack on sched_switch
./FlameGraph/flamegraph.pl --color=io --title="Off-CPU" offcpu.folded > offcpu.svg
```

A wide tower ending in `nfs_readpage`/`io_schedule`/`futex_wait` tells you the latency is storage or lock contention, not compute. This is the tool that catches the idle-but-expensive GPU node. For a continuous, always-on version of this idea rather than a one-off profiling session — a scheduler-latency monitor that runs permanently and alerts on drift — see the Netflix eBPF noisy-neighbor case below; the mechanism (hook `sched_switch`) is identical, the difference is one-shot investigation vs standing telemetry.

### perf record → report → script, the full loop
The three-verb workflow, and what each verb is *for*:
- **`perf record`** captures samples to a `perf.data` file. Key flags: `-F <hz>` sample rate, `-a` all-CPUs (system-wide), `-g` call graphs, `-p <pid>` / `-t <tid>` to target, `-e <event>` to sample on a specific hardware/software/tracepoint event instead of the default cycles (`-e block:block_rq_issue`, `-e cache-misses`). `--` runs a command and records for its lifetime; `sleep N` is the idiom for "record everything for N seconds."
- **`perf report`** reads `perf.data` into an interactive (or `--stdio`) ranked view: functions by **self** time (samples *in* that function) vs **children** (self + everything it called). Expand a call graph to see callers/callees. This is your first look before committing to a flame graph.
- **`perf script`** dumps the raw per-sample records — one event with its full stack — as text. This is the *export* format: pipe it to `stackcollapse-perf.pl` for flame graphs, or grep it for a specific event sequence. `perf annotate` goes the other direction, down to hot *instructions* within a function.

`perf list` enumerates every event the box exposes (hardware PMU counters, cache events, tracepoints, kprobes you've added). `perf top` is the live, no-file version — a continuously-updating `perf report`, the profiling analogue of `top`.

### ftrace / trace-cmd (awareness)
ftrace is the kernel's built-in tracing framework (interface under `/sys/kernel/tracing`, or `/sys/kernel/debug/tracing`), predating eBPF and always present. You rarely poke the raw filesystem; you use **`trace-cmd`** (CLI) or **`perf-tools`** wrappers (`funccount`, `funcgraph`, `iolatency`). Its standout capability is the **function graph tracer** — it traces kernel *function entry and exit with timing and call nesting*, printing an indented call tree with per-function durations. Where perf samples statistically, ftrace's function tracer is *exhaustive* for the functions you select — invaluable for "which kernel function inside this syscall is slow." Know it exists, know `trace-cmd record -p function_graph -g <fn>` and `funcgraph` are how you'd get a timed kernel call tree; reach for eBPF/bpftrace first for most tasks. The authoritative reference for every tracer ftrace ships (`function`, `function_graph`, event tracers, the histogram/trigger machinery) is the kernel's own `Documentation/trace/ftrace.rst` — worth a skim even if you never touch the raw filesystem, because it's the ground truth `trace-cmd` and `perf-tools` wrap.

### `perf_event_paranoid` vs container seccomp — two different walls
These get confused because both manifest as "perf just doesn't work here." `/proc/sys/kernel/perf_event_paranoid` is a **host sysctl** that gates *who* may use `perf_event_open()` (values from -1, unrestricted, to 2+, kernel-and-user profiling disabled for non-root); it's a policy knob you can read and, with root, change. Container runtimes present a **separate** wall: most default seccomp profiles (Docker's, containerd's) block the `perf_event_open` syscall entirely inside the container regardless of what the sysctl says, because it's considered a host-introspection surface. Fixing the sysctl does nothing if the syscall itself is filtered out — you need `--security-opt seccomp=unconfined` (or a custom profile allowing it) *and* a permissive `perf_event_paranoid`, or you profile from the host instead. Knowing there are two independent gates, not one, is what stops "perf returns nothing in this pod" from turning into a wasted afternoon.

## Perspectives

**Kernel-mechanism view.** Every tool in this lesson reduces to one of two hook points. On-CPU profiling (`perf record`) is a periodic interrupt — a hardware or software timer fires thousands of times a second, and each interrupt reads the current instruction pointer and unwinds the call stack; the *distribution* of those samples across the run is a statistical estimate of where CPU cycles went. Off-CPU profiling hooks a single kernel event, `sched_switch`, and instead of sampling on a timer it records deterministically every time a thread leaves the run queue and every time it returns — no sampling error, but only useful for time spent *not* running. Two different instrumentation strategies (statistical sampling vs event hooking) for two different questions (where does CPU go vs where does wall-clock go).

**Operator/SRE view.** "A node is slow, walk me through how you'd debug it" tests structure, not tool trivia. Unstructured answers — "I'd check top, then maybe iostat, then..." — read as guessing, because they reveal no method for knowing when you've covered enough ground to rule resources *out*. The USE grid is the answer to "how do I know I'm done": when you've asked Utilization/Saturation/Errors of every resource and none of them show a problem, you've either found nothing (rare) or you're missing an instrument for one resource (a genuine finding — go build that instrument, e.g. add PSI where you only had `%util`). This is the difference between debugging and diagnosing.

**GPU-fleet-specific/economics view.** An H100 node reserved at roughly $2–3/GPU-hr (2024–2025 on-demand pricing snapshots; check your provider) that sits at 5% CPU utilization looks idle to a dashboard tuned for CPU-bound services — but if its training process is off-CPU 80% of the time blocked on a slow NFS mount, the GPU itself is stalled waiting for the next batch, and you are paying full reserved price for a device doing no useful work. `top` shows this as "not much happening." Off-CPU analysis shows it as a wide `nfs_readpage`/`io_schedule` tower — a named, provable root cause you can take to a postmortem or a capacity-planning conversation. This is the single clearest cost argument in the module: the tool that finds the bug is also the tool that quantifies the dollars.

**Interview/methodology view.** Two distinct skills get tested under the same "can you debug performance" heading, and conflating them is a common failure mode. The first is **enumerative**: can you recite the USE checklist completely, unprompted, for an unspecified "slow node" — this tests whether your method is complete-by-construction. The second is **interpretive**: handed one specific flame graph SVG, can you correctly identify the hot frame, distinguish it from a tall-but-cheap tower, and state the fix — this tests whether you can read a concrete artifact, which is a narrower and different skill than knowing the checklist exists. A candidate can ace one and fumble the other. Practice both separately: recite the grid cold, and separately, sit with real flame graphs (yours or a teammate's) until reading width-not-depth is reflexive.

## Real-world use cases

- **["Java in Flames" — Netflix TechBlog](https://netflixtechblog.com/java-in-flames-e763b3d32166)** (Brendan Gregg & Martin Spier). The origin story of mixed-mode Java flame graphs in production at Netflix: they hit exactly the "why are my flame-graph towers `[unknown]`" pitfall this lesson warns about — JVM- and gcc-omitted frame pointers breaking stack unwinding — and the fix that resulted, `-XX:+PreserveFramePointer` (prototyped by Gregg), became a standard JDK feature. First-person account of the broken-stacks gotcha, told from the team that had to solve it at scale.
- **["Noisy Neighbor Detection with eBPF" — Netflix TechBlog](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd)**. Extends the on-CPU/off-CPU dichotomy this lesson teaches into a third mode: *continuous* scheduler-latency monitoring built on the same `sched_switch`-hooking mechanism as `offcputime`, running standing rather than as a one-off `perf`/flame-graph session — the production evolution of "profile it once during an incident" into "always be measuring it."

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

**Acceptance (into the deliverable's diagnostic toolkit):** commit (a) **one flame graph SVG** (on-CPU or off-CPU) with a caption stating what it revealed, and (b) a **written USE walkthrough of one investigation** — the resource grid you ran, the command outputs that localized it, and the one-line verdict. Both must stand alone: a teammate should be able to re-run your checklist and read your flame graph without you. See the deliverable spec: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md).

## Common pitfalls

1. **Diagnosing "slow" without enumerating resources first.** Pattern-matching to a symptom you've seen before ("this smells like a memory leak") skips the step that makes USE valuable — walking the grid *before* forming a hypothesis. This is the central interview failure mode: an answer that starts with a guess instead of a checklist.
2. **Trusting `iostat` `%util` on NVMe as a saturation signal.** Multi-queue NVMe can show 100% `%util` while nowhere near saturated (many in-flight requests, none queued) or the reverse. Use `await`/`aqu-sz` and io PSI instead.
3. **Profiling with round sampling frequencies (100 Hz).** A round frequency risks phase-locking with other periodic kernel activity, systematically skewing which code gets sampled. Use an off-round frequency like 99 Hz to decorrelate.
4. **Chasing tall towers instead of wide frames in a flame graph.** Depth is call-chain length, not cost. A tall, narrow tower was sampled rarely; a wide, flat top frame is where the CPU time actually is. Optimize width.
5. **Running `perf` inside a container instead of from the host.** Symbol and debuginfo resolution breaks (perf can't reach the target's binaries), and `perf_event_open` is frequently blocked outright by the container runtime's default seccomp profile — a separate gate from the `perf_event_paranoid` sysctl, and fixing one without the other still leaves you stuck.

## Self-check
**(a) Walk the USE method for a node reported "slow" — what do you check, for which resources, in what order?**
**Answer:** For every resource, check Utilization, Saturation, and Errors. Enumerate resources — CPU, memory, disk (I/O and capacity), network (plus GPU/interconnect and cgroup limits on a real fleet) — and walk them: **CPU** — `mpstat -P ALL` for per-core %busy, `/proc/pressure/cpu` and run-queue latency for saturation, `dmesg`/MCE for errors; **memory** — `free`/`meminfo` for utilization, swap activity (`vmstat` si/so) and `/proc/pressure/memory` and OOM kills for saturation, ECC/EDAC for errors; **disk** — `iostat -xz` for %util but trust `await`/`aqu-sz` and `/proc/pressure/io` for saturation, `smartctl`/`dmesg` for errors; **network** — `sar -n DEV`/`ip -s link` for throughput, drops and `nstat` retransmits for saturation, interface error counters for errors. The order is resource-by-resource, and saturation (queuing) is the signal operators most often miss — 100% utilization without a queue is fine, moderate utilization with a deep queue is the problem. PSI is the cleanest single saturation instrument.

**(b) What does a wide flat frame at the top of a flame graph indicate versus a tall narrow tower?**
**Answer:** Width = fraction of samples (fraction of CPU time); the x-axis is population, not time, and the top edge is the function actually on-CPU. A **wide, flat top frame** is a leaf function burning CPU directly across many samples — the genuine hot spot, and your optimization target. A **tall, narrow tower** is a deep call chain that was rarely sampled — lots of stack depth but little CPU time; deep does not mean expensive, and narrow means it isn't where the time goes, so you don't chase it. You optimize width, not height.

**(c) On-CPU vs off-CPU analysis — which tool/method for which symptom?**
**Answer:** **On-CPU** analysis (perf record/report/script → flame graph, sampling the IP+stack on a timer) answers "where is the CPU spending cycles" — use it when the process is CPU-bound: high `%usr`/`%sys`, IPC and cache metrics implicating compute. **Off-CPU** analysis (scheduler `sched_switch` hooks, e.g. eBPF `offcputime` → off-CPU flame graph, measuring time blocked and the stack at block time) answers "why is a thread *not* running" — use it when wall-clock latency is high but CPU is idle: high `%iowait`, IO/memory PSI, threads in D-state or on locks. Symptom-to-tool: CPU pegged → on-CPU perf; slow but idle / blocked on I/O or locks → off-CPU. The USE grid tells you which one to reach for.

**(d) Why does an interviewer test "recite the USE checklist" and "read this flame graph" as two separate questions instead of one?**
**Answer:** They're different skills. Reciting the USE checklist tests whether your *method* is complete by construction — can you enumerate every resource and both signal types (utilization, saturation) without a hint, so you provably can't skip a resource class. Reading a specific flame graph tests whether you can *interpret a concrete artifact* — spot the wide top-edge frame, distinguish it from a tall-but-cheap tower, and state the fix. A candidate can have the checklist memorized and still misread a flame graph, or vice versa, so competent interviewers probe both independently rather than treating "knows the checklist" as proof they can also read the picture.

## Connections & what's next

This lesson is the module's methodology spine: lesson 1 (scheduling) explained *why* load can exceed core count, lesson 4 (PSI) gave you the saturation instrument USE was always asking for, and lesson 8 (eBPF) gave you the mechanism (`sched_switch` hooking) that off-CPU analysis and tools like `offcputime`/`biolatency` are built on. Everything in this lesson is the practiced *application* of those pieces under a single checklist. Next, **lesson 10** takes the block-I/O thread from the worked example above — the NFS-mount stall — and goes one layer deeper: how systemd owns the cgroup tree that both encloses and (via `io.weight`/`io.max`) can bound that I/O, and how `io.pressure` (the per-cgroup cousin of the PSI you just used) tells you *whose* I/O is stalling *whom* on a shared disk. The off-CPU flame graph that named `nfs_readpage` here is the same investigation `io.pressure` picks up in the next lesson from the cgroup-attribution side.

## References & further reading

**Primary sources**
- [Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html) — the canonical statement of the methodology; read it as the checklist you internalize.
- [Brendan Gregg — perf Examples](https://www.brendangregg.com/perf.html) — the perf command reference this lesson's `record`/`report`/`script` workflow is drawn from.
- [Kernel docs — Documentation/trace/ftrace.rst](https://docs.kernel.org/trace/ftrace.html) — the authoritative reference for every ftrace tracer (`function`, `function_graph`, event/histogram machinery) that `trace-cmd` and `perf-tools` wrap.

**Real-world engineering blogs**
- [Netflix TechBlog — "Java in Flames"](https://netflixtechblog.com/java-in-flames-e763b3d32166) — what it shows: the production origin of the broken-stacks/frame-pointer problem this lesson's gotchas warn about, and the fix that became standard JDK tooling.
- [Netflix TechBlog — "Noisy Neighbor Detection with eBPF"](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd) — what it shows: the same `sched_switch`-hooking mechanism as off-CPU profiling, evolved into continuous, standing scheduler-latency monitoring rather than a one-off session.

**Deeper dives**
- [FlameGraph repo (brendangregg/FlameGraph)](https://github.com/brendangregg/FlameGraph) — `stackcollapse-perf.pl`, `flamegraph.pl`, and `difffolded.pl`; read the README for the on-CPU, off-CPU (`--color=io`), and differential workflows.
- Brendan Gregg, *Systems Performance* (2nd ed.), methodology chapter — the deep treatment tying USE, drill-down analysis, perf, and off-CPU together; the book to own for this domain.
