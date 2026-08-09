---
lesson: 06
title: "Deep-dive / debugging (incident) round drills"
module: 12
concept: "GPU incident debugging"
status: not-started
est_time: "4 hrs"
artifacts: ["gpu-debug-decision-trees", "d2-live-walkthrough"]
---

# Deep-dive / debugging (incident) round drills

## Why this matters
The neoclouds lean on the debugging round harder than on anything else, because it's the round you can't fake. Lambda runs a **live SSH round** — you're dropped onto a real box and asked to reason aloud. CoreWeave runs incident RCA. Together probes ML-systems debugging until you either produce a mechanism or hand-wave. Generic "check logs, check metrics, escalate" gets you cut. What passes is a **memorized decision tree** with the GPU-specific first move at each branch, delivered while narrating your hypothesis and the command that would confirm it.

The core trap running through all three scenarios: **`nvidia-smi` utilization is a liar.** It reports "a kernel was resident on the SM," not "useful FLOPs happened." Half of GPU debugging is refusing to trust that number and pivoting to goodput/MFU. If you internalize one thing from this lesson, internalize the reflex to distrust util and bisect compute / comms / data.

## What's new here
You know how to debug distributed systems. The overlay is three GPU-specific decision trees and the commands that arm each node of the tree:
- `nvidia-smi dmon` — rolling per-second SM/mem/power/pcie counters (better than plain `nvidia-smi` for spotting oscillation).
- `dcgmi diag -r {1,2,3}` — DCGM diagnostics; level 3 is the load/stress test that catches lemon nodes idle checks miss.
- `nsys profile` — Nsight Systems timeline that splits CUDA compute kernels from NCCL communication, so you can *see* whether you're compute-bound or comms-bound.
- `nccl-tests` (e.g. `all_reduce_perf`) — measured collective bandwidth vs the link's theoretical max; the fast way to indict the network/fabric.

Memorize which command sits at which branch. In a live round, naming the exact command *before* you'd run it is the senior tell.

## Core notes

**D1 — "A training job is slow — debug end-to-end."** *Arms: 08, 09, 04.*
Decision tree (memorize the order — the order is the point):
1. **Data pipeline first.** GPU starvation from slow dataloaders/storage is the #1 cause. If GPUs are waiting on data, nothing downstream matters.
2. **Test with synthetic data.** Replace the real dataset with in-memory random tensors. If throughput jumps → the bottleneck is **upstream of compute** (loaders, storage, preprocessing). If it doesn't → the bottleneck is compute or comms.
3. **Profile comms.** If utilization drops <50% during gradient sync → suspect the network. Run `nsys profile` to split compute vs NCCL time on the timeline; confirm fabric with `all_reduce_perf` (nccl-tests) against theoretical link BW.
4. **Check sync stalls.** Debug prints, forced `.item()` / `.cpu()` / device transfers, or `torch.cuda.synchronize()` in the hot loop puncture the async graph and serialize the GPU. Hunt these.
5. **Tune.** Microbatch size + gradient accumulation, dataloader `num_workers`/prefetch, gradient bucketing/overlap (DDP bucket size), pinned memory.

**D2 — "GPUs show 100% util but throughput dropped."** *The flagship trap. Arms: 05 (built for exactly this), 03, 08.*
The insight to state out loud: **util = "a kernel ran," not useful FLOPs.** High util + low goodput has four suspects:
- **Memory-bound kernels** — the SM is busy waiting on HBM, not computing. Low arithmetic intensity.
- **Tiny / oscillating batch** — kernels launch constantly (util pegged) but each does little work; `dmon` shows sawtooth SM%.
- **Comms-bound sync** — time spent in NCCL collectives counts as "busy" but produces no forward progress.
- **Silently degrading lemon node** — one straggler throttling the collective.
Correct move: **pivot the metric.** util → SM occupancy → MFU/goodput → job throughput. Then bisect: `nsys` to split compute vs comms; `dmon` to catch batch oscillation; `dcgmi diag` to catch a degrading node; check batch/KV-cache config for the tiny-batch case.

**D3 — "A node keeps failing large jobs but passes health checks."** *Arms: 08, 04.*
Grey/lemon-node decision tree:
1. **Reframe:** "passes idle, fails under load" → idle health checks are structurally insufficient.
2. **Load-based diagnostics:** run `dcgmi diag -r 3` (the stress/load level) *under real workload conditions*, not the quick idle check. Look for thermal throttle, ECC under load, NVLink/PCIe errors that only appear at bandwidth.
3. **Correlate:** does the same node ID appear across multiple large-job failures? Build the failure-mode log (module 04) and look for the repeat offender.
4. **Tiered remediation:** reset → cordon+drain+reschedule (via gang scheduler) → **remove from scheduling** and RMA.
5. **Cite the Meta result:** proactive lemon-node removal cut large-job failure rate **14% → 4%.** This is the number that shows you know the literature.

**"What a strong answer sounds like."** Narrate hypothesis → command → expected signal → next branch. E.g. "My first hypothesis is data starvation, so I'd swap in synthetic data; if throughput jumps I've isolated it upstream of compute and I'd profile the loader; if not, I'd `nsys profile` one step to split compute from NCCL time." You state the branch *and what result sends you down which fork* — that's the tell.

**Live-terminal drill protocol (Lambda-style).** SSH into a box (or a mock). Set a 20-minute timer. Rules: (1) narrate every hypothesis before you type; (2) name the command and the signal you expect *before* running it; (3) never say "check the logs" without saying which log and what you're looking for; (4) when you get a signal, state which branch it sends you to. Record yourself; grade on whether an interviewer could follow your reasoning without asking "why did you run that?" Re-run until two clean reps per scenario.

## Worked example
**Full D2 walkthrough — "GPUs at 100% util, throughput dropped 30% overnight":**

1. **Reject the metric out loud:** "100% util just means a kernel was resident — it doesn't mean useful FLOPs. I don't trust it; I want goodput. What's MFU / tokens-per-second doing?" (Establishes the senior frame immediately.)
2. **Establish ground truth:** pull job throughput (tokens/s or samples/s) and MFU from the dashboard (module 05/03). Confirm the *real* regression, since util is uninformative.
3. **`nvidia-smi dmon -s pucm`** for ~30s: if SM% is a sawtooth, suspect tiny/oscillating batch or launch overhead. If SM% is flat-high but mem-BW is pegged, suspect memory-bound kernels.
4. **`nsys profile` one training step:** read the timeline. Large NCCL bars → comms-bound (something changed in the network/topology, a rail moved). Compute bars back-to-back but low FLOP → memory-bound kernel or bad batch shape.
5. **Rule out a degraded node:** `dcgmi diag -r 3` on the suspect hosts; check per-rank step times for a straggler. A single slow rank drags the whole all-reduce and reads as "everyone busy, no progress."
6. **Confirm fabric if comms-bound:** `all_reduce_perf` from nccl-tests vs theoretical NVLink/IB bandwidth; a big gap indicts the link.
7. **Land the diagnosis:** "util was pegged because we became comms-bound after node X started throttling — util counted the spin-wait as busy. Fix: cordon X, reschedule the gang, and add a goodput-regression alert so we catch this on the leading indicator, not the lagging one."

The whole walkthrough is one message: *distrust util → pivot to goodput → bisect compute/comms/data → name the command at each fork → close with a remediation and a prevention.*

## Practice
- Memorize all three decision trees cold; write them into `gpu-debug-decision-trees` as one-page flowcharts.
- Run the live-terminal protocol on each scenario (D1/D2/D3), recorded, 20-min timer, two clean reps each.
- Drill the command→signal pairing until instant: for `dmon`, `dcgmi diag -r 3`, `nsys profile`, `all_reduce_perf`, say in one breath what each tells you and which branch its result sends you to.
- Write your **D2 walkthrough** as an 8-minute spoken script; it's the flagship trap and the one most likely to appear. Feed all three trees into the [GPU platform capstone](../practice/gpu-platform-capstone/README.md) as your incident-round appendix.

## Self-check
- In D1, what is the first thing you check and what single test isolates it? **Answer:** Check the data pipeline first (GPU starvation from slow loaders/storage is the #1 cause); swap in synthetic in-memory data — if throughput jumps, the bottleneck is upstream of compute; if not, it's compute or comms. **Answer:** Synthetic-data test.
- In D2, why can utilization read 100% while throughput drops, and what metric do you pivot to? **Answer:** `nvidia-smi` util counts a kernel being resident (including memory-bound stalls and NCCL spin-waits) as "busy," not useful FLOPs. Pivot up the hierarchy to SM occupancy → MFU/goodput → job throughput, then bisect compute/comms/data.
- In D3, what makes a lemon node evade detection, what catches it, and what's the headline number? **Answer:** It passes idle health checks but fails under load, so you need load-based diagnostics (`dcgmi diag -r 3` under real workload) plus cross-job failure correlation; proactive lemon-node removal cut Meta's large-job failure rate from 14% to 4%.

## Resources
- GMI Cloud — GPU utilization myths (gmicloud.ai/en/blog/gpu-utilization-myths)
- Crusoe — The AI engineer's checklist for optimal GPU performance (crusoe.ai/resources/blog/the-ai-engineers-checklist-for-optimal-gpu-performance)
- Lyceum — GPU utilization too low: how to fix (lyceum.technology/magazine/gpu-utilization-too-low-how-to-fix)
- Introl — Troubleshooting GPU clusters
- Together — Multi-node training (guide)
- [🎓 12 — Capstone & interview preparation](../README.md)
