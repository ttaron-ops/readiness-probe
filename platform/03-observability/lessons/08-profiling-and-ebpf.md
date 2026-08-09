---
lesson: "A03.8"
title: "Profiling and eBPF"
module: "A-03"
concept: "on-CPU vs off-CPU, fleet-wide eBPF"
status: not-started
est_time: "3 hrs"
artifacts: ["offcpu-diagnosis.md"]
---

# A03.8 · Profiling and eBPF

> **Concept.** The staff distinction is on-CPU vs off-CPU: real latency usually hides in the off-CPU stacks a CPU profiler never samples — and eBPF makes capturing it cheap enough to run continuously across the whole fleet.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Why this matters

A senior engineer reaches for `perf`, gets a flame graph, and finds the hot function. At staff depth the failures are subtler: the flame graph is *flat* and the service is still slow, because the time is spent *waiting*, not computing — and no amount of on-CPU sampling will ever show it. Add a 4000-node GPU fleet where the bottleneck is a single straggler blocked on `cudaStreamSynchronize`, and the ability to reason about off-CPU time and to profile continuously without SSH-ing every box becomes a core platform capability. This lesson is that distinction plus the eBPF substrate that operationalizes it.

## Core notes

**Skip (you already know):** what a flame graph is; `perf`; that profiling finds hot code; the USE/RED acronyms.

**Reading a flame graph correctly (the misreads staff must not make).** The x-axis is **population / sample count, not time** — frames are ordered alphabetically, and width = fraction of samples. The y-axis is stack depth. The CPU is spending its time in the **widest *leaf* frames** (the top plateaus), not the wide frames lower down (those are just ancestors). The classic misread is treating width as latency — it is not; a wide frame means "sampled often," which for an on-CPU graph means "burned cycles here," and says nothing about wall-clock wait.

**On-CPU vs off-CPU — the staff distinction.**
- **On-CPU** (what `perf`/normal sampling captures): where cycles burn. Answers "what code is hot."
- **Off-CPU**: where threads are *blocked* — I/O, lock contention, scheduler run-queue wait, GPU synchronization. This is invisible to CPU profiling because a blocked thread is not running to be sampled, yet it is usually where real latency lives. eBPF makes off-CPU profiling cheap by hooking scheduler switch events and **summing off-CPU time by stack in-kernel** (`offcputime`), so you get a flame graph whose width = time *spent waiting* per stack.

The complete picture needs both: on-CPU shows compute hotspots, off-CPU shows wait hotspots. A flat on-CPU graph on a slow service is the tell to go straight to off-CPU.

**eBPF as the whole-fleet substrate.** In-kernel aggregation means low overhead, which means you can run it *continuously* on *every node* with **no app changes and no restarts**. That is the leap from "profile one box when it's on fire" to **continuous fleet-wide profiling** — a queryable, time-indexed store of where every node spends its cycles and its waits.

**Tools.**
- **Parca** (Polar Signals): eBPF whole-system agent per node, Prometheus-style storage, OSS, clean Prometheus integration. Good default for "profiles as another Prom-shaped signal."
- **Pyroscope** (Grafana): fed via Alloy's eBPF profiler and the OTel eBPF profiler; multi-tenancy, alerting, Grafana-native, "metrics from profiles." Both emit **pprof-compatible** profiles queryable over time.

**USE method as the systematic frame.** For every resource, check **Utilization / Saturation / Errors**. This is what stops you missing a *saturated* resource — run-queue, PCIe, NVLink — that a hot flame graph alone would never reveal, because saturation shows up as off-CPU wait and queue depth, not as a wide leaf frame.

**GPU tie.** Brendan Gregg's **AI Flame Graphs** extend the technique into the GPU: a mixed-mode flame graph spanning CPU launch stacks down into GPU code, showing where GPU stall/instruction time goes. Off-CPU profiling catches training threads blocked on `cudaStreamSynchronize` / NCCL waits — **stragglers surface as off-CPU time**. Fleet-wide continuous profiling then finds the regressed kernel or host bottleneck across the pool without SSH-ing 4000 boxes: you query the store, not the nodes. (The separate GPU-observability artifact covers the DCGM/util-lie/MFU/goodput side.)

## Worked example

A serving+training node is slow; the on-CPU flame graph is nearly flat (no wide leaf). You pull the off-CPU flame graph and see a wide plateau over two stacks:

```
[training thread] ... -> libcudart -> cudaStreamSynchronize -> [futex/sched_switch]   (62% of off-CPU time)
[worker]          ... -> recvmsg -> sock wait                                          (18% of off-CPU time)
```

**Diagnosis:** the node is **GPU-bound waiting — a straggler, not CPU-bound.** The dominant off-CPU stack is the CPU thread parked in `cudaStreamSynchronize`, i.e. blocked waiting for the GPU (or for the slowest NCCL peer) to finish, not spinning on the CPU.

**Which signal revealed it and why on-CPU missed it:** the **off-CPU** profile revealed it, because the relevant thread spends its time *blocked* (descheduled) rather than running. On-CPU sampling only records threads that are executing on a core at sample time; a thread parked in a futex behind `cudaStreamSynchronize` is never on-core, so it contributes ~zero on-CPU samples — hence the flat, uninformative on-CPU graph. Off-CPU profiling hooks the scheduler-switch path and attributes the *wait duration* to the blocking stack, making the synchronization stall the widest frame.

**Fleet move:** because this is captured continuously via eBPF, you query the fleet store for nodes whose `cudaStreamSynchronize` off-CPU time regressed after the last kernel/driver rollout — locating the bad kernel across the pool without touching a single box.

## Practice

<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Add the profiling layer to the fleet observability design: (1) choose Parca or Pyroscope and justify it against your existing Prometheus/Grafana stack; (2) specify what the per-node eBPF agent captures (on-CPU, off-CPU, mixed-mode GPU) and the overhead budget that keeps it always-on; (3) write the USE checklist for a GPU node — name the resource, and the U/S/E signal for each of CPU, run-queue, PCIe, NVLink, GPU SM, HBM; (4) define a straggler-detection query over the continuous off-CPU store that flags nodes with anomalous `cudaStreamSynchronize`/NCCL wait relative to the pool, and state how it composes with the goodput burn-rate alert from A03.7.

## Self-check

- A service is slow but its on-CPU flame graph is flat with no wide leaf frame. What do you do next and why? **Answer:** Switch to an off-CPU flame graph. A flat on-CPU graph means the threads are not burning cycles — the time is spent *blocked* (I/O, locks, run-queue, GPU sync), which on-CPU sampling cannot see because blocked threads are not running to be sampled. Off-CPU profiling (e.g. eBPF `offcputime`) sums wait time by stack in-kernel and exposes where the latency actually lives.
- In a flame graph, a frame two levels from the bottom is very wide. Does that frame consume most of the CPU? **Answer:** No. Width = fraction of samples, and a wide non-leaf frame is just a common *ancestor*; the CPU time is attributed to the widest *leaf* frames (top plateaus). The x-axis is population/sample count, not time, and it is ordered alphabetically — width is never latency and depth of a wide frame doesn't imply it's where cycles burn.
- Why does eBPF enable continuous *fleet-wide* profiling where `perf` runs did not? **Answer:** eBPF aggregates in-kernel (e.g. summing samples/off-CPU time by stack in the kernel), giving low enough overhead to run always-on on every node with no app changes or restarts, feeding a queryable time-indexed store (Parca/Pyroscope, pprof-compatible). You query the store to find a regressed kernel or straggler across 4000 nodes instead of SSH-ing in and running ad-hoc `perf` on one box at a time.

## References

- https://www.brendangregg.com/flamegraphs.html
- https://www.brendangregg.com/FlameGraphs/offcpuflamegraphs.html
- https://www.brendangregg.com/blog/2024-10-29/ai-flame-graphs.html
- https://grafana.com/docs/pyroscope/latest/
