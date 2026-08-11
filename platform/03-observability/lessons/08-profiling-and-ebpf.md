---
lesson: "A03.8"
title: "Continuous profiling and eBPF"
module: "A-03"
concept: "on-CPU vs off-CPU, fleet-wide eBPF"
status: not-started
est_time: "4 hrs"
prev: "07-slos-and-alerting.md"
next: "09-gpu-and-ml-observability.md"
artifacts: ["offcpu-diagnosis.md"]
sources: 8
---

# A03.8 · Continuous profiling and eBPF

> **Concept.** The staff distinction is on-CPU vs off-CPU: real latency usually hides in the off-CPU stacks a CPU profiler never samples — and eBPF makes capturing it cheap enough to run continuously across the whole fleet.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

[07 · SLOs and alerting](07-slos-and-alerting.md) ends with a burn-rate alert firing correctly. This lesson is the next rung on the escalation ladder: an alert tells you a symptom is happening, not why. Profiling — and specifically the on-CPU/off-CPU split, run continuously via eBPF — is how you go from "the goodput SLI is burning budget" to "this specific node is parked in `cudaStreamSynchronize` waiting on the slowest NCCL peer." It's also the last stop before [09 · GPU and ML observability at fleet scale](09-gpu-and-ml-observability.md), which applies the same off-CPU/eBPF machinery as one input to fleet-wide straggler detection alongside DCGM and NCCL signals.

## Why this matters

A senior engineer reaches for `perf`, gets a flame graph, and finds the hot function. At staff depth the failures are subtler: the flame graph is *flat* and the service is still slow, because the time is spent *waiting*, not computing — and no amount of on-CPU sampling will ever show it. Add a 4000-node GPU fleet where the bottleneck is a single straggler blocked on `cudaStreamSynchronize`, and the ability to reason about off-CPU time and to profile continuously without SSH-ing every box becomes a core platform capability. This lesson is that distinction plus the eBPF substrate that operationalizes it — and the second, less obvious payoff: continuous profiling data as an input to the compiler, not just the dashboard.

## What's new here (calibration)

- **Skip:** what a flame graph is, `perf`, "profiling finds hot code," the USE/RED acronyms.
- **Push depth on:** the on-CPU/off-CPU distinction as a kernel mechanism (not just a UI toggle); the stack-unwinding deployment prerequisite (frame pointers vs DWARF) that decides whether an eBPF profiler actually works on your fleet; the concrete overhead budget that makes "always-on, every node" credible; and profiling-as-compiler-input (FDO/BOLT) as a distinct value stream from profiling-as-observability.

## Core concepts

### Reading a flame graph correctly (the misreads staff must not make)

The x-axis is **population / sample count, not time** — frames are ordered alphabetically, and width = fraction of samples. The y-axis is stack depth. The CPU is spending its time in the **widest *leaf* frames** (the top plateaus), not the wide frames lower down (those are just ancestors). The classic misread is treating width as latency — it is not; a wide frame means "sampled often," which for an on-CPU graph means "burned cycles here," and says nothing about wall-clock wait. Width only approximates CPU time cleanly when the sampling rate is uniform and the sample count is large enough to avoid quantization noise at the tails.

### On-CPU vs off-CPU — the staff distinction

- **On-CPU** (what `perf`/normal sampling captures): where cycles burn. Answers "what code is hot."
- **Off-CPU**: where threads are *blocked* — I/O, lock contention, scheduler run-queue wait, GPU synchronization. This is invisible to CPU profiling because a blocked thread is not running to be sampled, yet it is usually where real latency lives.

Off-CPU profiling is a specific kernel mechanism, not a UI mode: it hooks `sched_switch` tracepoints, timestamps the block (schedule-out) and wake (schedule-in) events for a thread, and attributes the elapsed delta to the stack that was blocking at schedule-out time — summed **in-kernel** by stack (`offcputime`), so the flame graph's width becomes *time spent waiting* per stack, not samples burned.

The complete picture needs both: on-CPU shows compute hotspots, off-CPU shows wait hotspots. A flat on-CPU graph on a slow service is the tell to go straight to off-CPU.

### Stack unwinding: the deployment prerequisite nobody mentions

Off-CPU attribution depends entirely on accurate stack unwinding *at the moment of schedule-out* — and that's nontrivial for JIT'd or interpreted languages. Two unwinding strategies, with a real tradeoff:

- **Frame-pointer unwinding** — cheap and simple, but requires the binary to have been compiled with frame pointers retained (`-fno-omit-frame-pointer`). Many distro packages strip frame pointers by default for a small performance win, which silently breaks unwinding for anything built from them.
- **DWARF-based unwinding** — works without frame pointers, but is meaningfully more CPU-expensive per sample. Recent eBPF tooling (including Parca) has pushed DWARF unwinding further in-kernel to reduce that cost, but this is still an actively evolving area, not a fully settled one.

Practical consequence: rolling out a fleet-wide eBPF profiler is not just "install the DaemonSet." It's also an audit of whether your fleet's binaries (and interpreters — Python, JVM without frame-pointer support) were built in a way the profiler can actually unwind, or whether you need the heavier DWARF path (or runtime-specific symbolication) to avoid a flame graph full of `[unknown]` frames.

### eBPF as the whole-fleet substrate (and its overhead budget)

In-kernel aggregation means low overhead, which means you can run it *continuously* on *every node* with **no app changes and no restarts**. Concretely: eBPF sampling profilers (Parca, Pyroscope's eBPF agent) typically target well under 1% steady-state CPU overhead at sampling rates around 19–100 Hz, because stack aggregation happens in-kernel before anything reaches userspace. Contrast that with `perf record`, whose cost is dominated by userspace post-processing of a raw event stream — workable for a point-in-time capture on one box, not for always-on collection across thousands of nodes. That overhead gap is *the* reason eBPF is the substrate for "profile everywhere all the time" and `perf` remains the tool for "profile this one box right now."

eBPF programs are also verifier-checked before they're allowed to run in-kernel, which bounds what they can do — but continuous whole-fleet profiling still means code with `CAP_BPF`/`CAP_PERFMON` (or full privilege on older kernels) running on every node, and a heterogeneous fleet means testing that agent against every kernel version in the fleet, not just the newest one. That operational trust boundary is a genuine deployment concern, not just a performance one — worth naming explicitly in a design doc, not waved away as "it's eBPF, it's safe."

### Two value streams: observability and compiler feedback

Continuous profiling data has two distinct payoffs that are easy to conflate:

1. **Observability** — a queryable, time-indexed store of where every node spends its cycles and its waits, for humans doing triage.
2. **Compiler feedback (FDO/BOLT)** — the same profile data fed back into Profile-Guided Optimization at compile/link time, closing the loop from "where does the fleet spend cycles" to "recompile the binary to spend fewer." Meta's Strobelight explicitly ties fleet-wide eBPF profiling to this second stream, feeding BOLT and FDO for their largest services. This is a genuinely different stakeholder and a different ROI story than a dashboard — it's an input to the build system, not to an on-call rotation.

That is the leap from "profile one box when it's on fire" to **continuous fleet-wide profiling** as durable infrastructure with more than one consumer.

### Tools

- **Parca** (Polar Signals): eBPF whole-system agent per node, Prometheus-style storage, OSS, clean Prometheus integration. Good default for "profiles as another Prom-shaped signal."
- **Pyroscope** (Grafana): fed via Alloy's eBPF profiler and the OTel eBPF profiler; multi-tenancy, alerting, Grafana-native, "metrics from profiles." Both emit **pprof-compatible** profiles queryable over time.

### USE method as the systematic frame

For every resource, check **Utilization / Saturation / Errors**. This is what stops you missing a *saturated* resource — run-queue, PCIe, NVLink — that a hot flame graph alone would never reveal, because saturation shows up as off-CPU wait and queue depth, not as a wide leaf frame.

### GPU tie

Brendan Gregg's **AI Flame Graphs** extend the technique into the GPU: a mixed-mode flame graph spanning CPU launch stacks down into GPU code, showing where GPU stall/instruction time goes. His follow-up **Doom GPU Flame Graphs** post validates the same mixed-mode CPU→GPU technique on a non-AI (gaming) GPU workload — good evidence this is a general GPU-profiling technique, not an AI-specific trick that happens to also work elsewhere. Off-CPU profiling catches training threads blocked on `cudaStreamSynchronize` / NCCL waits — **stragglers surface as off-CPU time**. Fleet-wide continuous profiling then finds the regressed kernel or host bottleneck across the pool without SSH-ing 4000 boxes: you query the store, not the nodes. (The separate GPU-observability artifact covers the DCGM/util-lie/MFU/goodput side.)

## Perspectives

**Kernel/runtime-internals.** Off-CPU profiling's correctness rests entirely on accurate stack unwinding at the exact instant a thread schedules out. For compiled languages with frame pointers, that's cheap and reliable; for JIT'd or interpreted runtimes without them, it's a real engineering problem requiring either recompilation with `-fno-omit-frame-pointer` or the heavier DWARF-unwinding path. Anyone presenting eBPF profiling as "zero instrumentation, zero setup" hasn't hit this wall yet.

**Fleet-economics.** Meta's Strobelight is the clearest evidence that fleet-wide profiling infrastructure pays for itself twice: once as an observability tool, and once as an input to FDO/BOLT compiler optimization that measurably cuts CPU cycles across their largest services. Treating "continuous profiling" as purely a debugging aid undersells the investment case to leadership — the compiler feedback loop is a separate, quantifiable ROI line.

**Security/isolation.** The eBPF verifier bounds what an in-kernel program can do, but that's a different claim from "safe to run everywhere." Fleet-wide continuous profiling means privileged (or near-privileged) code on every node, and a heterogeneous fleet means validating agent behavior across every kernel version in production — a genuine operational trust boundary, not a solved problem just because the technology is eBPF.

**Cost-reduction (concrete, non-GPU).** Polar Signals used eBPF profiling to attribute and cut cross-zone network cost by half — a reminder that "profiling" fleet-wide via eBPF extends past CPU/off-CPU stacks into any per-process resource attribution problem that's expensive to instrument any other way. The mental model generalizes: eBPF is a cheap, always-on, per-process attribution substrate, and CPU cycles are just the first resource anyone points it at.

## Real-world use cases

- **Meta — "Strobelight: A profiling service built on open source technology"** (https://engineering.fb.com/2025/01/21/production-engineering/strobelight-a-profiling-service-built-on-open-source-technology/, open-sourced at https://github.com/facebookincubator/strobelight): Meta's fleet-wide eBPF profiler feeding FDO/BOLT compiler optimization across their top ~200 services, with reported CPU-cycle reductions up to 20% — the clearest real-world case of profiling-as-compiler-input at scale.
- **eBPF Foundation case study — "Polar Signals Uses eBPF to Monitor Internal Cross-Zone Network Traffic on Kubernetes, Reducing These Operating Costs by 50%"** (https://ebpf.foundation/case-study-polar-signals-uses-ebpf-to-monitor-internal-cross-zone-network-traffic-on-kubernetes-reducing-these-operating-costs-by-50/): a concrete, quantified example of eBPF-based always-on attribution solving a cost problem well outside classic CPU profiling.
- **Brendan Gregg — "AI Flame Graphs"** (https://www.brendangregg.com/blog/2024-10-29/ai-flame-graphs.html): the primary source for mixed-mode CPU→GPU flame graphs — the technique lesson 09 builds on for GPU straggler and stall attribution.
- **Brendan Gregg — "Doom GPU Flame Graphs"** (https://www.brendangregg.com/blog/2025-05-01/doom-gpu-flame-graphs.html): validates the same mixed-mode technique on a non-AI GPU workload, evidence the method is general-purpose GPU profiling, not AI-specific.

## Worked example

A serving+training node is slow; the on-CPU flame graph is nearly flat (no wide leaf). You pull the off-CPU flame graph and see a wide plateau over two stacks:

```
[training thread] ... -> libcudart -> cudaStreamSynchronize -> [futex/sched_switch]   (62% of off-CPU time)
[worker]          ... -> recvmsg -> sock wait                                          (18% of off-CPU time)
```

**Diagnosis:** the node is **GPU-bound waiting — a straggler, not CPU-bound.** The dominant off-CPU stack is the CPU thread parked in `cudaStreamSynchronize`, i.e. blocked waiting for the GPU (or for the slowest NCCL peer) to finish, not spinning on the CPU.

**Which signal revealed it and why on-CPU missed it:** the **off-CPU** profile revealed it, because the relevant thread spends its time *blocked* (descheduled) rather than running. On-CPU sampling only records threads that are executing on a core at sample time; a thread parked in a futex behind `cudaStreamSynchronize` is never on-core, so it contributes ~zero on-CPU samples — hence the flat, uninformative on-CPU graph. Off-CPU profiling hooks the scheduler-switch path and attributes the *wait duration* to the blocking stack, making the synchronization stall the widest frame.

**Fleet move:** because this is captured continuously via eBPF, you query the fleet store for nodes whose `cudaStreamSynchronize` off-CPU time regressed after the last kernel/driver rollout — locating the bad kernel across the pool without touching a single box.

**Overhead sanity check:** running this continuously across 4000 nodes at ~99 Hz sampling with in-kernel stack aggregation stays under ~1% steady-state CPU per node — the same query pattern via ad hoc `perf record` sessions, one node at a time, would require someone to notice the problem, SSH in, capture, and post-process *after* the fact, by which point a transient straggler has often already resolved itself.

**Unwinding check before any of this works:** if the training thread's stack were in a language runtime without frame pointers (stripped Python, a JVM without `-XX:+PreserveFramePointer`), the off-CPU flame graph above could instead show a stub of `[unknown]` frames at the point of interest — the unwinding prerequisite has to be satisfied *before* the diagnosis is trustworthy, not assumed.

## Practice

<feeds [fleet observability design](../practice/fleet-observability/README.md)>

Add the profiling layer to the fleet observability design: (1) choose Parca or Pyroscope and justify it against your existing Prometheus/Grafana stack; (2) specify what the per-node eBPF agent captures (on-CPU, off-CPU, mixed-mode GPU), the overhead budget that keeps it always-on (name a concrete sampling rate and expected CPU%), and which unwinding strategy (frame-pointer vs DWARF) you require fleet binaries to support and why; (3) write the USE checklist for a GPU node — name the resource, and the U/S/E signal for each of CPU, run-queue, PCIe, NVLink, GPU SM, HBM; (4) define a straggler-detection query over the continuous off-CPU store that flags nodes with anomalous `cudaStreamSynchronize`/NCCL wait relative to the pool, and state how it composes with the goodput burn-rate alert from [07 · SLOs and alerting](07-slos-and-alerting.md); (5) name one non-debugging consumer of the same profile data (compiler feedback, cost attribution) to justify the infrastructure investment beyond incident response.

## Common pitfalls

- **"An eBPF profiler needs no in-app changes, so it captures everything with zero blind spots."** Not true for language runtimes without frame pointers or with heavy inlining/JIT — without runtime-specific symbolication or DWARF unwinding, stacks can come back incomplete or `[unknown]`, silently degrading exactly the workloads (Python, JVM) where you most need the signal.
- **"Off-CPU time is always bad and should be minimized."** Legitimate I/O wait, intentional backoff, and cooperative yielding are off-CPU too. Off-CPU profiling identifies *where* time goes, not automatically whether it's wasteful — a `cudaStreamSynchronize` wait on a well-balanced training job is normal; only an outlier relative to its peers indicates a straggler.
- **"Flame graph width directly equals wall-clock time for on-CPU graphs."** Width = fraction of samples, which approximates CPU time only under uniform sampling and a large enough sample count to avoid quantization noise — it is never a direct wall-clock measurement, and it says nothing about off-CPU wait at all.
- **"Continuous profiling replaces the need for targeted deep-dive tools like Nsight or the PyTorch Profiler."** eBPF continuous profiling is the fleet-wide triage layer that tells you *which node or job* to look at. Kernel-level GPU tools remain necessary for the deep single-job root-cause step — they're complementary rungs on the same escalation ladder, not competing tools.

## Self-check

- A service is slow but its on-CPU flame graph is flat with no wide leaf frame. What do you do next and why? **Answer:** Switch to an off-CPU flame graph. A flat on-CPU graph means the threads are not burning cycles — the time is spent *blocked* (I/O, locks, run-queue, GPU sync), which on-CPU sampling cannot see because blocked threads are not running to be sampled. Off-CPU profiling (e.g. eBPF `offcputime`) sums wait time by stack in-kernel and exposes where the latency actually lives.
- In a flame graph, a frame two levels from the bottom is very wide. Does that frame consume most of the CPU? **Answer:** No. Width = fraction of samples, and a wide non-leaf frame is just a common *ancestor*; the CPU time is attributed to the widest *leaf* frames (top plateaus). The x-axis is population/sample count, not time, and it is ordered alphabetically — width is never latency and depth of a wide frame doesn't imply it's where cycles burn.
- Why does eBPF enable continuous *fleet-wide* profiling where `perf` runs did not? **Answer:** eBPF aggregates in-kernel (e.g. summing samples/off-CPU time by stack in the kernel), giving low enough overhead — typically under 1% CPU at ~19–100 Hz sampling — to run always-on on every node with no app changes or restarts, feeding a queryable time-indexed store (Parca/Pyroscope, pprof-compatible). `perf record`'s cost is dominated by userspace post-processing, which is fine for one box, one incident, but not for continuous fleet-wide collection. You query the store to find a regressed kernel or straggler across 4000 nodes instead of SSH-ing in and running ad-hoc `perf` on one box at a time.
- You deploy an eBPF profiling agent fleet-wide and a chunk of your Python services show mostly `[unknown]` frames in their off-CPU flame graphs. What's the most likely cause, and what are the two fixes? **Answer:** Missing stack-unwinding support — most likely the Python interpreter (or its native extensions) was built/packaged without frame pointers retained, so frame-pointer-based unwinding can't walk the stack at schedule-out time. Fixes: rebuild with `-fno-omit-frame-pointer` (cheap unwinding, needs a rebuild) or switch the agent to DWARF-based unwinding (works without rebuilding, costs more CPU per sample) — or add runtime-specific symbolication for the interpreter.
- Besides finding the slow node, what's a second, distinct payoff of running continuous eBPF profiling fleet-wide, and who is its consumer? **Answer:** Feeding the same profile data into Profile-Guided Optimization at compile/link time (FDO/BOLT) — Meta's Strobelight does exactly this, closing the loop from "where does the fleet spend cycles" to "recompile the binary to spend fewer." The consumer is the build/compiler pipeline, not an on-call engineer — a different ROI story than incident response, worth naming explicitly when justifying the infrastructure investment.

## Connections & what's next

This lesson is the diagnosis layer that a correctly-firing burn-rate alert from [07 · SLOs and alerting](07-slos-and-alerting.md) hands off to — the alert says *what* is burning budget, off-CPU/eBPF profiling says *why*. It also reuses the cardinality and cost discipline from [01 · The signal model](01-signal-model.md) and [03 · Metrics at scale](03-metrics-at-scale.md): a profile store is another high-volume, always-on signal that has to be sized and retained deliberately, not an exception to those rules.

Next: [09 · GPU and ML observability at fleet scale](09-gpu-and-ml-observability.md) — the module's synthesis lesson, which folds off-CPU/eBPF straggler detection in alongside DCGM, NCCL, and multitenant cardinality discipline to diagnose a fleet-wide training regression, not just a single node.

## References & further reading

**Primary sources**
- Brendan Gregg — Flame Graphs: https://www.brendangregg.com/flamegraphs.html
- Brendan Gregg — Off-CPU Flame Graphs: https://www.brendangregg.com/FlameGraphs/offcpuflamegraphs.html
- Brendan Gregg — AI Flame Graphs: https://www.brendangregg.com/blog/2024-10-29/ai-flame-graphs.html
- Brendan Gregg — Doom GPU Flame Graphs: https://www.brendangregg.com/blog/2025-05-01/doom-gpu-flame-graphs.html

**Real-world engineering blogs**
- Meta Engineering — Strobelight: A profiling service built on open source technology: https://engineering.fb.com/2025/01/21/production-engineering/strobelight-a-profiling-service-built-on-open-source-technology/ (repo: https://github.com/facebookincubator/strobelight)
- eBPF Foundation — Polar Signals case study (cross-zone network cost, -50%): https://ebpf.foundation/case-study-polar-signals-uses-ebpf-to-monitor-internal-cross-zone-network-traffic-on-kubernetes-reducing-these-operating-costs-by-50/

**Deeper dives**
- Grafana Pyroscope docs: https://grafana.com/docs/pyroscope/latest/
