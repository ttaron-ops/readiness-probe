---
lesson: 06
title: "Deep-dive / debugging (incident) round drills"
module: 12
concept: "GPU incident debugging"
status: not-started
est_time: "5 hrs"
prev: "05-system-design-drills.md"
next: "07-narrate-artifacts.md"
artifacts: ["gpu-debug-decision-trees", "d2-live-walkthrough"]
sources: 10
---

# Deep-dive / debugging (incident) round drills

## Where this fits

Lesson 05 built fluency in the whiteboard design round — six prompts drilled until you volunteer scale, cost, failure modes and SLOs on reflex, and until every verdict comes with its reversal condition. This lesson drills a distinct and harder-to-fake round: the live debugging or incident interview, where you are handed a broken system — a described symptom, a screenshot of a dashboard, or an actual shell on an actual box — and have to reason out loud, on a clock, with nothing to hide behind. Design rounds reward architectural fluency. Debugging rounds reward disciplined diagnostic process under pressure, and they are weighted heavily in this market precisely because they are the round a candidate cannot prepare for by memorising an architecture.

## Why this matters

The debugging round is the one you cannot fake, and everyone running these loops knows it. You can read your way to a competent design answer. You cannot read your way to a competent diagnostic process, because a diagnostic process is a sequence of decisions made under uncertainty, and the only thing that produces it is having made those decisions before. Generic "check the logs, check the metrics, escalate" gets you cut, not because the words are wrong but because they contain no branch — there is no result that would send you anywhere in particular.

The stakes are not interview-theoretical. Distributed training at scale fails often enough that diagnostic speed is a direct cost lever: a published study of eleven months of operations across two large research clusters measured a mean-time-to-failure of **7.9 hours for 1,024-GPU jobs**, and *projects* shorter times at larger scale (arXiv:2410.21680). At that cadence, every minute spent guessing instead of bisecting is billed GPU-hours across a four- or five-figure fleet. Interviewers know this, which is why they score *process* rather than only the final answer — a candidate who guesses correctly and a candidate who reasons correctly look identical in the transcript unless the reasoning was narrated.

And one trap runs through every scenario in this lesson: **the flagship utilisation metric is a liar, and it lies hardest exactly when a job is stuck.** It reports that a kernel was resident, not that useful work happened — and a collective blocked on a straggler, a spin-wait, and a memory-starved decode loop all keep a kernel resident. Half of GPU debugging is refusing to trust that number and pivoting to something that measures work. If you internalise one reflex from this lesson, internalise that one.

## What's new here (calibration)

You know how to debug distributed systems. The overlay is three GPU-specific decision trees, the commands that arm each node of each tree, and the narration discipline that makes the reasoning visible.

- **New**: the three trees drawn *as* trees — with the branch condition on every edge, so the tree is a decision procedure rather than a checklist.
- **New**: each drill written as a **full run-through** — the prompt as asked, the narration in speaking register, the interviewer's interruptions with weak-versus-strong responses, and the failure modes.
- **New**: the **command→signal→branch** table, drilled to instant recall, because a command you cannot pair with an expected signal is a command you are running out of hope.
- **New**: the **live-terminal protocol** and how to run it against yourself with no interviewer.
- **New**: the tools, with what each actually gives you and what it costs to run:
  - `nvidia-smi dmon` — rolling per-second SM, memory, power and PCIe counters. Better than plain `nvidia-smi` for spotting oscillation, because a single-shot query samples once and a sawtooth averages away.
  - `dcgmi dmon -e <field-ids>` — the same idea in DCGM field terms, which lets you put the presence metric and the honest metric on adjacent columns.
  - `dcgmi diag -r {1,2,3}` — DCGM diagnostics. Level 1 is quick and idle-safe; level 3 is the load and stress level that catches faults idle checks miss, and it is a stress test, so it cannot run on a node serving a job.
  - `nsys profile` — an Nsight Systems timeline that separates CUDA compute kernels from NCCL communication, so you can *see* whether you are compute-bound or comms-bound instead of inferring it.
  - `all_reduce_perf` from nccl-tests — the collective benchmark that turns "the network feels slow" into a measured bandwidth against the link's theoretical maximum.

## Core concepts

### 1. What the round is scoring, and the one discipline that carries it

The interviewer is watching for a single behaviour, repeated: **hypothesis → command → expected signal → branch.** Say what you think is wrong, say what you will run, say what result would confirm it, and say where each possible result sends you. Then run it.

Why this and not correctness: a correct diagnosis reached silently is indistinguishable from a lucky guess, and the round exists to predict how you will perform on a failure mode you have not seen. Narrated reasoning generalises; a remembered answer does not. **In a round designed to be hard to fake, the narration is the primary signal being graded.**

The second discipline is **cheap checks first**. Order matters as much as content. A candidate who reaches for a profiler before ruling out the data pipeline is choosing an expensive, hard-to-interpret instrument over a cheap, high-yield one — and that ordering choice is scored independently of whether the eventual root cause is correct. The honest justification is a cost one: while you search, the fleet is still billing.

```
   WHAT THE INTERVIEWER HEARS — the same 90 seconds, two candidates
  ══════════════════════════════════════════════════════════════════════════════

   CANDIDATE A                             CANDIDATE B
   ─────────────                           ─────────────
   "Let me look at the GPUs."              "First hypothesis is data starvation,
        │                                   because it's the most common cause
        ▼                                   and the cheapest to rule out."
   [runs nvidia-smi]                             │
        │                                        ▼
        ▼                                   "I'll swap the dataset for
   "OK, they're at 100%."                    in-memory random tensors and
        │                                    re-measure step time.
        ▼                                    If throughput jumps, the
   [runs nsys profile]                       bottleneck is upstream of
        │                                    compute and I'll profile the
        ▼                                    loader. If it doesn't move,
   [reads timeline for 4 minutes,            it's compute or comms and I'll
    silently]                                split those with a profiler."
        │                                        │
        ▼                                        ▼
   "So it's NCCL."                          [runs the test]
                                                 │
   SCORED: correct answer, no visible            ▼
   process. Cannot be distinguished         "No change. So it's not the
   from a guess. Does not predict            input path. Next branch:
   performance on an unseen failure.         compute vs comms."

                                            SCORED: every step has a
                                            hypothesis, a test, and a
                                            branch. Generalises.

   THE ASYMMETRY: A can only be scored on the final answer.
                  B can be scored on the reasoning, and reasoning is
                  what the round is for.
```

### 2. The command→signal→branch table

Drill this until each row comes out in one breath. A command you cannot pair with an expected signal is a command you are running out of hope rather than out of a hypothesis.

| Command | What it gives you | The signal you are looking for | Where each result sends you |
|---|---|---|---|
| `dcgmi dmon -e 203,1002,1003,1004,1005,252 -d 1000` | presence, SM breadth, occupancy, tensor, DRAM, framebuffer — side by side, per second | the *shape* of the mismatch | presence high + breadth low → §4's four suspects · breadth high + tensor low → stall or wrong-pipe math · everything low + framebuffer high → loaded but idle replica |
| `nvidia-smi dmon -s pucm` | rolling per-second power, utilisation, clocks, memory | oscillation (a sawtooth) versus a flat line | sawtooth → tiny or oscillating batch, launch overhead · flat high with memory pegged → memory-bound kernels · clocks below boost → thermal or power cap |
| `nvidia-smi -q -d POWER,TEMPERATURE,CLOCK` | caps, current draw, throttle reasons | an active throttle reason, or a clock capped below spec | throttled → thermal/power path, check airflow and the power cap before blaming software |
| `dmesg -T \| grep -i xid` | driver-level hardware error codes with timestamps | any Xid, and *which* Xid | Xid present → hardware or driver fault, and the specific code tells you whether it is memory, a fallen-off-bus device, or an application fault |
| `dcgmi diag -r 1` | quick idle-safe checks | pass/fail | passes → tells you almost nothing about load behaviour. **This is the trap in §5.** |
| `dcgmi diag -r 3` | load and stress level, catches thermal/ECC/link faults under bandwidth | failures that only appear under sustained load | **cannot be run on a node serving a job** — it needs a drained node and a maintenance window |
| `nsys profile -o out ./one_step` | a timeline separating CUDA kernels from NCCL collectives | the ratio of compute bars to communication bars, and gaps | large NCCL bars → comms-bound · back-to-back compute with low FLOP → memory-bound or bad shapes · *gaps* → host-side stall, dataloader or a synchronisation point |
| `all_reduce_perf -b 8 -e 1G -f 2 -g N` | measured collective bandwidth across a size sweep | achieved bus bandwidth versus the link's theoretical maximum | far below theoretical → fabric: check link width, rail placement, a renegotiated link · at theoretical → the network is fine and the problem is above it |
| per-rank step-time series | which rank is slow | one rank consistently slower than its peers | one outlier → straggler, cordon candidate · all uniformly slower → workload or configuration, not a node |
| `nvidia-smi topo -m` | the device-to-device link matrix | whether ranks that talk are actually adjacent | ranks split across NUMA or rails → placement problem, not a hardware fault |

**Two facts about these tools that are worth volunteering**, because they demonstrate you have actually run them rather than read about them: `dcgmi diag -r 3` is a stress test and will fight with a running workload, so it belongs in a maintenance window on a drained node; and the profiling fields in `dcgmi dmon` require elevated privileges and are *dropped rather than zeroed* when unavailable, so a column of `N/A` where you expected breadth is a permissions problem, not an idle GPU.

### 3. Drill D1 — "A training job is slow. Debug it end to end."

**The prompt, as asked:** *"A team says their training job is running at about half the throughput it had last week. Nothing obvious changed. Go."*

**The tree.** The order is the content. Memorise the order.

```
   D1 · SLOW TRAINING JOB — the decision tree
  ══════════════════════════════════════════════════════════════════════════════

   START: "half throughput, nothing changed"
     │
     │  first, establish the ground truth. NOT utilisation.
     ▼
   ┌───────────────────────────────────────────────────────────┐
   │ Q0 · Is throughput actually down?                         │
   │      samples/s or tokens/s, and step time, week over week │
   └────────────┬──────────────────────────────┬───────────────┘
          no ───┘                              └─── yes
          │                                          │
          ▼                                          ▼
   the METRIC changed, not the job.       ┌────────────────────────────────┐
   Check what was deployed to the         │ Q1 · Is it DATA STARVATION?    │
   monitoring stack. This happens         │   THE CHEAPEST CHECK, SO FIRST │
   more often than people admit.          │   swap dataset → in-memory     │
                                          │   random tensors, re-measure   │
                                          └──────┬──────────────┬──────────┘
                            throughput JUMPS ────┘              └──── NO change
                                    │                                    │
                                    ▼                                    ▼
                    ┌───────────────────────────────┐   ┌────────────────────────────┐
                    │ BOTTLENECK IS UPSTREAM OF     │   │ Q2 · COMPUTE or COMMS?     │
                    │ COMPUTE                       │   │   nsys profile ONE step    │
                    │  · dataloader workers,        │   │   read the timeline        │
                    │    prefetch depth             │   └───┬──────────┬─────────┬───┘
                    │  · storage read bandwidth     │       │          │         │
                    │  · CPU-side preprocessing     │  big NCCL   compute    GAPS
                    │  · pinned memory / H2D copies │   bars      back-to-   between
                    └───────────────────────────────┘       │     back, low   kernels
                                                            │     FLOP        │
                                                            ▼        │        ▼
                                              ┌──────────────────┐   │   ┌──────────────┐
                                              │ Q3 · COMMS-BOUND │   │   │ Q4 · HOST    │
                                              │  all_reduce_perf │   │   │ STALL        │
                                              │  vs theoretical  │   │   │  a .item(),  │
                                              │       │          │   │   │  a .cpu(),   │
                                              │  ┌────┴─────┐    │   │   │  a debug     │
                                              │  ▼          ▼    │   │   │  print, or   │
                                              │ far      at spec │   │   │  an explicit │
                                              │ below      │     │   │   │  synchronize │
                                              │  │         ▼     │   │   │  in the hot  │
                                              │  ▼    it's not   │   │   │  loop        │
                                              │ FABRIC: link  the│   │   └──────────────┘
                                              │ width, rail   net│   │
                                              │ placement,       │   ▼
                                              │ a renegotiated   │  ┌──────────────────┐
                                              │ link, ONE slow   │  │ Q5 · COMPUTE     │
                                              │ rank dragging    │  │  memory-bound?   │
                                              │ the collective   │  │  wrong precision?│
                                              └──────────────────┘  │  bad GEMM shapes?│
                                                                    │  check tensor vs │
                                                                    │  DRAM active     │
                                                                    └──────────────────┘

   THE ORDERING RULE: cheapest and highest-yield first. The synthetic-data
   test costs one run and eliminates an entire half of the search space.
   A profiler costs setup, a privileged run, and interpretation time.
```

**The worked run-through**, in speaking register:

> *"Before I touch anything, I want ground truth, and it will not be the utilisation panel — that number tells me whether a kernel was resident, not whether work happened, and on a stalled job it reads high. So: samples per second, or tokens per second, and step time, this week against last week. If those are flat and only a dashboard changed, I'm debugging the monitoring stack, not the job, and that is worth ruling out in thirty seconds because it happens more often than people admit.*
>
> *Assume throughput really is down about half. First hypothesis is data starvation, and I lead with it for two reasons: it's the most common cause of a slow training job, and it's the cheapest thing to test. The test is to replace the dataset with in-memory random tensors of the same shape and re-measure step time. That's one run and it eliminates half the search space.*
>
> *If throughput jumps with synthetic data, the bottleneck is upstream of compute and I stop looking at GPUs entirely. Then it's dataloader worker count and prefetch depth, storage read bandwidth, CPU-side preprocessing, and whether host-to-device copies are using pinned memory. The signal I'd look for on the GPU side to corroborate is gaps in the timeline — the SMs going quiet between steps rather than being busy but unproductive.*
>
> *If throughput does not move, it's compute or comms, and I split those with a profiler on exactly one step. I'm reading three things off that timeline. Large communication bars relative to compute means comms-bound. Compute bars back to back but achieving low FLOP means memory-bound kernels or bad matrix shapes. And gaps between kernels mean a host-side stall — something punctured the async graph, which in practice is a `.item()`, a `.cpu()`, a debug print, or an explicit synchronise that someone added in the hot loop. That third case is the one that matches 'nothing obvious changed', because a one-line logging addition is exactly what people don't count as a change.*
>
> *If it's comms-bound, I don't jump to blaming the network — I measure it. Run the collective benchmark across a size sweep and compare achieved bus bandwidth against the link's theoretical maximum. If it's far below, the fabric is implicated and I'd look at link width, whether ranks landed on the same rail, and whether a link renegotiated to a lower width. If it's at spec, the network is fine and the problem is above it — which usually means one rank is slow and dragging the collective, so I'd pull per-rank step times and look for a single outlier rather than a uniform slowdown.*
>
> *Throughout, the thing I'd keep saying is which result sends me where, because I want you to be able to follow the reasoning even if my first hypothesis is wrong."*

**Interruption 1 — "Skip the synthetic-data test. I'll tell you the dataloader is fine."**

| Weak | Strong |
|---|---|
| "OK, then I'd profile." | "Happy to — but let me say what I'm giving up, because it changes what I trust later. That test doesn't only implicate the loader; it eliminates the whole input path, including storage bandwidth, preprocessing and the host-to-device copies. So if I skip it and later see gaps in the timeline, I won't be able to tell a host-side stall in the training loop from a slow input path without coming back to it. Given that, I'll go straight to a profile of one step, but I'll be reading specifically for gaps as well as for the compute/comms split." |

**Interruption 2 — "The profile shows 40% of step time in NCCL. Is that bad?"**

| Weak | Strong |
|---|---|
| "Yes, that's very high — it's comms-bound." | "It's high, but 'bad' depends on two things I'd want before saying so. First, what was it before? A 40% share is a problem if it used to be 15% and unremarkable if the model was always communication-heavy at this scale — and this job halved in throughput, so I want the delta, not the absolute. Second, is the collective *slow* or is compute *fast*? If compute got faster, the communication share rises without anything being wrong. So I'd measure the collective directly against theoretical bandwidth. And if the collective is genuinely slow, there's still a fork: the fabric is degraded, or one rank is late to the collective and everyone else is waiting — which shows up as communication time but is actually a straggler. Per-rank step times separate those." |

**Interruption 3 — "You have twenty minutes and one command. What is it?"**

| Weak | Strong |
|---|---|
| "`nsys profile`." | "`dcgmi dmon` with the presence field, SM breadth, tensor activity, DRAM activity and framebuffer, for about a minute, on one rank and one known-good rank if I can get both. It's the highest information per second available: the *shape* across those five columns distinguishes most of my hypotheses at once. Breadth low with framebuffer high and DRAM low is starvation or a stall. Breadth high with tensor low and DRAM low is a synchronisation stall. Breadth moderate with DRAM pegged is memory-bound. And a sawtooth in breadth is batch oscillation or launch overhead. A profiler gives me more detail but costs setup time and needs interpretation; that one command gives me the branch." |

**Failure modes on D1:** starting from the utilisation panel; skipping the synthetic-data test because the loader "obviously" isn't the problem — the value of the tree is the order, and skipping a cheap step even when you are right reads as impatience rather than efficiency; treating "40% in NCCL" as a verdict rather than a measurement needing a baseline; never mentioning that a one-line logging change can halve throughput by puncturing the async graph.

### 4. Drill D2 — "GPUs show 100% utilisation but throughput dropped"

**The prompt:** *"Overnight, job throughput fell about 30%. The GPU utilisation panel is unchanged at 100%. What's going on?"*

**This is the flagship trap** and the round most likely to appear, because it is the single cleanest test of whether a candidate has actually operated GPUs. The whole scenario is constructed around the metric that lies.

**The insight to state out loud, in the first fifteen seconds:** utilisation means a kernel was resident, not that useful work happened. It is a threshold at one — one kernel and ten thousand kernels evaluate the same predicate — and it has no notion of how many SMs exist. So the panel being unchanged is not evidence that nothing changed; it is evidence that the panel cannot see this class of change.

**The four suspects, and the pair of fields that separates them:**

```
   D2 · 100% PRESENCE, LOW THROUGHPUT — the four suspects
  ══════════════════════════════════════════════════════════════════════════════

   All four hold GPU_UTIL at ~100. The pair that separates them is
   SM_ACTIVE (breadth) × DRAM_ACTIVE (memory), with TENSOR as the tiebreak.

                              DRAM_ACTIVE (1005)
                        low  ◀──────────────────────▶  high
              ┌──────────────────────────┬──────────────────────────┐
              │  ③ SYNC / COMMS STALL    │  ④ MEMORY-BOUND AT SCALE │
         high │  SMs lit, tensor pipes    │  SMs lit, HBM saturated, │
              │  near zero, HBM quiet.    │  tensor pipes modest.    │
              │  A collective blocked on  │  Real work, wrong shape  │
   SM_ACTIVE  │  a straggler; a spin-     │  or wrong precision, or  │
     (1002)   │  wait; __syncthreads      │  genuinely bandwidth-    │
              │  contention.              │  limited.                │
              │  → per-rank step times    │  → check precision, GEMM │
              │    then nsys              │    shapes, batch size    │
              ├──────────────────────────┼──────────────────────────┤
              │  ① STARVED / TINY KERNELS │  ② SMALL-BATCH DECODE    │
          low │  Few SMs lit, nothing      │  Few SMs lit, HBM busy.  │
              │  moving. Dataloader stall  │  The classic batch-1     │
              │  between micro-steps, or   │  serving shape: every    │
              │  launch overhead dominating│  weight re-read per      │
              │  → dmon for a SAWTOOTH     │  token.                  │
              │    then the D1 tree        │  → BATCH IT. Do not buy  │
              │                            │    more GPUs.            │
              └──────────────────────────┴──────────────────────────┘

   AND A FIFTH, WHICH IS NOT ON THIS PLOT AT ALL:
     ⑤ THE HONEST METRIC STOPPED BEING EXPORTED.
       Profiling fields are DROPPED, not zeroed, when unavailable —
       a permissions change or an exporter restart removes them from
       the average rather than dragging it down. Symptom: a
       suspiciously stable fleet average. Check for absence FIRST if
       "nothing changed" but the numbers moved.
```

**The worked run-through:**

> *"First thing I'd say out loud: I don't trust that panel, and not because it's broken. `DCGM_FI_DEV_GPU_UTIL` is field 203, an unmodified passthrough of an NVML counter defined as the fraction of a short sample window during which at least one kernel was resident. It's a threshold at one. So 'unchanged at 100%' is exactly what I'd expect it to read whether the fleet is perfectly healthy or completely stalled — it cannot distinguish those. The fact that it didn't move is information about the metric, not about the job.*
>
> *So: ground truth first. Job throughput — tokens or samples per second — confirms the regression is real. Then I pivot up the hierarchy: SM breadth, occupancy, tensor-pipe activity, DRAM activity.*
>
> *Before the four suspects, one cheap check that catches an embarrassing case: did the honest metric stop being exported? The profiling fields are dropped rather than zeroed when they're unavailable, so a permissions change or a hardened deployment dropping a capability makes them silently absent, and absence is not zero in a query — an average over namespaces just skips those GPUs and looks unchanged. If 'nothing changed' but the numbers moved, I want to rule that out in one query before I go hunting.*
>
> *Assuming the honest metrics are present, `dcgmi dmon` with those fields for about a minute, and I'm reading the shape rather than any single value. Four suspects, and the pair of SM breadth against DRAM activity separates them.*
>
> *Breadth low, DRAM low: the GPUs are genuinely idle inside the window — starvation or launch overhead. I'd look for a sawtooth in the per-second series, because a single-shot query averages that away, and then I'm in the D1 tree at the data-pipeline branch.*
>
> *Breadth low, DRAM high: the classic small-batch decode shape — every weight re-read from memory per token. That's memory-bandwidth-bound by physics, and the fix is batching, not hardware. If this appeared overnight, something changed the batching configuration or the traffic mix.*
>
> *Breadth high, DRAM low, tensor near zero: SMs are lit but not computing. That's a stall — a collective blocked on a straggler, a spin-wait, synchronisation contention. This is the one that matches 'overnight, nothing changed', because a node degrading is not a change anyone made. I'd go to per-rank step times looking for one outlier, then a profile to confirm.*
>
> *Breadth high, DRAM high, tensor modest: real work, but on the wrong units or in the wrong shape — FP32 where lower precision would dispatch to the tensor path, or matrix shapes that don't tile well.*
>
> *For a 30% overnight drop with no deploy, my prior is suspect three: a node started degrading. So the sequence is — confirm throughput, check for metric absence, `dmon` for the shape, per-rank step times for an outlier, then load-level diagnostics on the suspect node once it's drained. And I'd close by naming the prevention, because that's the part that stops this recurring: an alert on the honest metric and on goodput, not on the presence metric, plus an alert on absence so that half the fleet going dark on the honest metric doesn't look like health."*

**Interruption 1 — "Per-rank step times show all ranks equally slow. Now what?"**

| Weak | Strong |
|---|---|
| "Then it must be the network." | "That rules out a straggler, which was my leading hypothesis, so I'd update rather than substitute a new guess. Uniform slowness across ranks means the cause is common to all of them: the fabric as a whole, a shared storage path, a configuration change that applied everywhere, or the workload itself. I'd split those cheaply. The collective benchmark against theoretical bandwidth tells me whether the fabric is degraded — and note that a degraded *shared* element like a switch shows up as uniform, which is consistent. If the fabric measures at spec, then it's above the network, and I'd look at what's common: a driver or container image rollout, a config change, or a shift in the input data's shape. The last one is easy to forget and it's real — longer sequences change the compute and memory profile without anyone deploying anything." |

**Interruption 2 — "You said the metric is a liar. So why does anyone ship it?"**

| Weak | Strong |
|---|---|
| "Legacy, mostly." | "Because it's cheap and it answers a real question — is anything scheduled on this device — which is genuinely useful for a liveness check. The driver can maintain it from scheduler state with no cost; the honest metrics require programming hardware performance counters, which needs elevated privileges and, on some silicon generations, a separate profiling module. So it's the default because it's the cheapest thing that is always available. What makes it dangerous isn't the metric, it's that it's presented under a name people read as 'efficiency'. I keep it on exactly one panel — beside the honest metric, as the foil that makes the gap visible — and I never alert on it, because a reclaim rule built on it will never fire on the most expensive waste, since that waste is precisely the case that keeps a kernel resident." |

**Interruption 3 — "Suppose you can't get the profiling metrics at all. This box is locked down."**

| Weak | Strong |
|---|---|
| "Then I'd be stuck without them." | "Then I'd work from what's still available, and I'd say clearly which conclusions get weaker. Without SM breadth I lose the four-suspect split, so I'd substitute proxies. Power draw is a decent proxy for real work — a GPU spinning on a wait kernel draws noticeably less than one doing dense math, so a fleet at 100% presence and low power is stalled. Clocks tell me about throttling. Per-rank step time is application-level and needs no privileges at all, so a straggler is still findable. Memory-copy utilisation and framebuffer usage are device-level and still there. And if the metrics vanished *because* the box was locked down recently, that's itself the answer to 'nothing changed' — a capability drop that removed the profiling fields would make the honest series disappear while everything else kept reporting. I'd check that first." |

**Failure modes on D2:** distrusting the metric correctly and then guessing at a fix without bisecting — pivoting away from utilisation is necessary and not sufficient, and jumping to "must be the network" reads as a lucky guess even when it's right; failing to check for metric *absence* before hunting a workload cause; conflating breadth with productivity, i.e. treating SM-active as the new truth when it too is an occupancy measure; not naming the prevention at the end.

### 5. Drill D3 — "A node keeps failing large jobs but passes health checks"

**The prompt:** *"One node in the fleet keeps showing up in failed 512-GPU runs. Every health check we run on it passes. What do you do?"*

**The reframe that opens the answer:** the fact that it passes idle checks is not exculpatory — it is the diagnostic clue. It tells you the failure is load-dependent, which immediately narrows the space to faults that only manifest under sustained stress: thermal, power delivery, marginal links, and memory errors that appear at bandwidth.

```
   D3 · THE NODE THAT PASSES AND STILL KILLS JOBS
  ══════════════════════════════════════════════════════════════════════════════

   "passes health checks, fails large jobs"
              │
              │  REFRAME: passing an IDLE check is evidence of a
              ▼  LOAD-DEPENDENT fault, not of health
   ┌────────────────────────────────────────────────────────────────┐
   │ STEP 1 · Is it actually this node? Establish the association.  │
   │   For each node: failure rate of jobs that touched it, vs the  │
   │   fleet baseline, over a window.                               │
   │   CONFOUND TO HANDLE: big jobs touch more nodes AND fail more  │
   │   often, so raw counts will implicate every node in the fleet. │
   └───────────┬──────────────────────────────┬─────────────────────┘
        not significant                  significantly above baseline
               │                                    │
               ▼                                    ▼
   look elsewhere: the job, the      ┌──────────────────────────────────┐
   fabric, a shared dependency.      │ STEP 2 · What KIND of fault?     │
   Say so rather than forcing        │   Pull, for this node, under     │
   the hypothesis.                   │   load and at failure time:      │
                                     └──┬────────┬────────┬────────┬────┘
                                        │        │        │        │
                     ┌──────────────────┘        │        │        └────────────┐
                     ▼                           ▼        ▼                     ▼
            ┌─────────────────┐      ┌────────────────┐ ┌──────────────┐ ┌──────────────┐
            │ THERMAL / POWER │      │ MEMORY (ECC)   │ │ LINK         │ │ NOTHING      │
            │ clocks below    │      │ correctable    │ │ width or     │ │ ANOMALOUS    │
            │ boost under     │      │ error rate     │ │ speed below  │ │ in counters  │
            │ load; throttle  │      │ climbing;      │ │ spec; retrain│ │      │       │
            │ reason set      │      │ row remaps     │ │ events       │ │      ▼       │
            └────────┬────────┘      └───────┬────────┘ └──────┬───────┘ │ STEP 3 ·    │
                     │                       │                 │         │ REPRODUCE   │
                     └───────────┬───────────┴─────────────────┘         │ drain, then │
                                 ▼                                       │ run the     │
                     ┌───────────────────────────┐                       │ LOAD-level  │
                     │ STEP 4 · TIERED RESPONSE  │◀──────────────────────│ diagnostic  │
                     │  soft: reset the device,  │                       │ (it is a    │
                     │        restart the agent  │                       │ STRESS test │
                     │  med : cordon, DRAIN, and │                       │ — cannot    │
                     │        re-place the WHOLE │                       │ run on a    │
                     │        gang (not one pod) │                       │ busy node)  │
                     │  hard: remove from the    │                       └──────────────┘
                     │        schedulable pool,  │
                     │        open an RMA        │
                     │  ── automate soft & med;  │
                     │     keep a human on hard  │
                     └───────────────────────────┘
```

**The worked run-through:**

> *"The first thing I'd do is refuse the framing. 'It passes health checks' is being offered as evidence the node is fine, and it's the opposite — it's evidence the fault is load-dependent. Idle checks are necessary and structurally insufficient, because the faults that kill large jobs are the ones that only appear under sustained stress: thermal throttling, power delivery, a marginal link that only errors at bandwidth, memory errors that only surface at real access rates. So passing the idle check narrows my search rather than ending it.*
>
> *Step one is to check whether it's actually this node, because 'keeps showing up' is exactly the shape of a confounded statistic. Large jobs touch many nodes and fail more often for reasons unrelated to any one of them, so if I naively count appearances in failed runs, every node in the fleet looks guilty and the ones in the most jobs look guiltiest. What I want is the failure rate of jobs that touched this node against the fleet baseline, over a window with enough job-touches to mean something. If that's not significant, I say so and look elsewhere rather than forcing the hypothesis — which is a thing I'd want to say explicitly in an interview, because being willing to abandon the offered lead is part of the process.*
>
> *If it is significant, step two is what kind of fault. I'd pull, for that node specifically, under load and around the failure timestamps: clocks against boost and any throttle reason set; correctable ECC rate and whether rows have been remapped; link width and speed against spec, plus any retraining events; and the driver-level error log for hardware error codes. Those four cover the load-dependent classes, and each one has a different remediation.*
>
> *If nothing in the counters looks anomalous, step three is to reproduce it deliberately — drain the node and run the load-level diagnostic, which is the stress level rather than the quick check. Worth stating that this is a stress test: it cannot run on a node serving a job, so it costs a maintenance window and the node's capacity for its duration. That cost is real and it's the reason you don't just run it on everything nightly without doing the arithmetic.*
>
> *Step four is remediation, tiered. Soft: reset the device, restart the agent, put it back. Medium: cordon, drain, re-place — and specifically re-place the whole gang, not one pod, because a distributed job draining one rank recreates exactly the partial-placement hang that gang scheduling exists to prevent. Hard: remove it from the schedulable pool and open a hardware replacement. I'd automate soft and medium and keep a human on hard, because an RMA decision is expensive to get wrong and rare enough to justify the review.*
>
> *And the reason this whole class of work is worth building rather than handling case by case: the published study of two large research clusters over eleven months found that proactive lemon-node detection took the failure rate of jobs at 512 GPUs and above from about 14% to about 4%, identifying roughly forty faulty nodes at over 85% detection accuracy. That's a measured, order-of-magnitude effect on large-job success, and it's the argument for spending engineering time on the association signal rather than on more health checks."*

**Interruption 1 — "What detection signals actually drove that 14-to-4 result?"**

| Weak | Strong |
|---|---|
| "Better health checks." | "Not a single silver-bullet check — that's the point of the result. It's the combination of association across job failures, which is what catches a node that passes every point-in-time test, with load-based diagnostics that reproduce the fault once a node is suspected. The association part is doing the detection and the diagnostic part is doing the confirmation. And I'd add the honest caveat: at 85%-ish detection accuracy there are false positives, so the design has to make a false positive cheap — send suspected nodes to a drain-and-diagnose queue rather than straight to replacement, so a wrong answer costs one diagnostic window instead of a node." |

**Interruption 2 — "Your detector flags a node. The team says the node is fine and wants it back. What do you say?"**

| Weak | Strong |
|---|---|
| "I'd trust the detector." | "I'd make the detector's output arguable rather than authoritative, because a binary verdict invites exactly this fight and loses it eventually. What I'd bring is evidence: this node was involved in six of the last nine large-job failures against a fleet baseline of about one in eight, here are the job IDs, and here's what the load diagnostic showed when we drained it. Then the conversation is about the data rather than about whose judgement wins. And I'd have a defined path to return it — passes the load diagnostic twice, runs clean for N large jobs while flagged — so 'give it back' has an answer that isn't 'no'. The failure mode I'm trying to avoid is operators learning to override the detector, because once they override the false positives they'll override the true ones too." |

**Interruption 3 — "Can't you just run the load diagnostic on every node every night?"**

| Weak | Strong |
|---|---|
| "That would be ideal but it's expensive." | "You can, and it's an arithmetic decision rather than a preference. The diagnostic takes a node out of service for its duration, so fleet-wide nightly coverage costs nodes times duration in GPU-hours — on a 125-node fleet at fifteen minutes a node that's about 31 node-hours a night, call it 1.3% of capacity. Against that, weigh what it prevents: if it moves large-job failure from 14% to 4%, and a failed 512-GPU run costs its elapsed time since the last checkpoint, the arithmetic usually favours running it. But I'd stage rather than blanket: full coverage on nodes returning from maintenance and on anything the association detector flags, sampled coverage on the rest, and tune the sample rate against the observed detection rate rather than against a calendar. That way the cost tracks the actual fault rate." |

**Failure modes on D3:** treating "passes health checks" as a dead end; counting appearances in failed jobs without handling the confound; automating the replacement decision; forgetting that draining one rank of a distributed job requires re-placing the whole gang; quoting the lemon-node number without being able to explain the mechanism behind it.

### 6. The live-terminal protocol

Some loops hand you an actual shell. Run this against yourself until it is comfortable.

```
   LIVE-TERMINAL DRILL — 20 minutes, self-run
  ══════════════════════════════════════════════════════════════════════════════

   t−1 min   Set a timer. Open a recording. Have the scenario written down
             by someone else, or pick one you have not rehearsed this week.

   t 0:00  ┌─ RULE 1 · Narrate the hypothesis BEFORE you type.
           │  "I think X. I'll run Y. I expect to see Z."
           │
           ├─ RULE 2 · Name the branch BEFORE you read the output.
           │  "If it's Z, I go to A. If it's not, I go to B."
           │
           ├─ RULE 3 · Never say 'check the logs'. Say WHICH log and
           │  WHAT you are looking for in it.
           │
           ├─ RULE 4 · When output arrives, say which branch it selects
           │  before interpreting the detail.
           │
           └─ RULE 5 · If you are surprised, SAY you are surprised, and
              say what it rules out. Surprise handled out loud is a
              strong signal; surprise absorbed silently reads as
              confusion.
   t 20:00
             ─────────────────────────────────────────────────────────
             GRADE THE RECORDING ON ONE QUESTION:
               could a listener follow the reasoning without ever
               asking "why did you run that?"
             If they'd have to ask even once, that's the rep to redo.
             ─────────────────────────────────────────────────────────

   THE FIVE-SECOND HABIT that fixes most of this: after every command,
   before reading its output, say the sentence "what I'm looking for is…".
   It forces the expectation to exist before the evidence does, which is
   the entire difference between diagnosing and browsing.
```

**Two more disciplines that show up in scores:**

**Say what you would do that you cannot do here.** "On a real fleet I'd pull per-rank step times from the training job's own metrics; I don't have those here, so I'll approximate with power draw across ranks." That sentence demonstrates you know the right instrument and are reasoning about its absence, which is strictly better than silently using the wrong one.

**Close with prevention.** Every scenario ends with "and here's the alert I'd add so this is caught on the leading indicator." Diagnosis is table stakes at this level; the thing that reads senior is turning one incident into a class of incidents that gets detected automatically next time.

## Perspectives

**The live-round view.** The interviewer is not only listening for the right diagnosis — they are scoring, second by second, whether you state a hypothesis *before* acting on it. A command run silently, with the correct conclusion announced once the output appears, scores lower than the same command preceded by "I expect X; if I see Y instead, that points at Z." In a round designed to be hard to fake, the narration is not decoration on the technical skill — it is the only window the interviewer has into whether your process would generalise to a failure mode you have not seen.

**The on-call view.** A dashboard curiosity and a page are different objects. At 03:00 nobody cares that utilisation reads 100 in the abstract; they care whether the job is making forward progress and what the fastest path to a fix is. These trees are the runbooks a real platform team would keep, compressed to interview pacing. Narrate them as the person who will actually be paged, and let that shape which branch you check first — cheapest and highest-yield, not most theoretically interesting.

**The cost view.** Every minute spent debugging is billed GPU-hours across however many nodes are stalled. With measured mean-time-to-failure around 7.9 hours for 1,024-GPU jobs, diagnostic speed compounds: disciplined bisection is a line item, not an aesthetic. When asked why you would run the cheap check first, the honest answer includes "because it's cheaper *and* it eliminates more of the search space per unit of billed time," not only "because that's the order."

**The tooling-evolution view.** Manual bisection of the kind drilled here is increasingly productised — rank-level straggler detection now ships as a feature in commercial fleet-management offerings rather than being assembled by hand. That is not a reason to skip the manual skill; it is the reason to know it cold, because the people who build, extend and debug that automation are the ones who could do the diagnosis by hand first. Naming the trend in a round — "this is now productised, but the underlying signal is per-rank step-time variance" — shows depth and currency at once.

**The honest-limits view.** Some of these questions do not have a clean answer, and saying so is a strong move rather than a weak one. Per-tenant attribution on a time-sliced device cannot be recovered from device counters; a node that fails intermittently may never reproduce under diagnosis; a 30% regression with no deploy may turn out to be a change in the input data's shape. A candidate who narrates the limit and proposes what they would do given it outperforms one who manufactures certainty.

## Real-world use cases

- **Meta, *Revisiting Reliability in Large-Scale Machine Learning Research Clusters*
  (arXiv:2410.21680, HPCA 2025).** Eleven months across two clusters (≈16K and ≈8K A100 GPUs).
  Measured MTTF of **7.9 hours for 1,024-GPU jobs**, with 1.8 h at 16,384 and 0.23 h at 131,072 as
  the paper's **projections**. Proactive lemon-node detection reduced the failure rate of jobs at
  512 GPUs and above from about **14% to about 4%**, identifying roughly forty faulty nodes at over
  85% detection accuracy. **What it shows:** D3's entire premise as a measured result, and the
  numbers to quote. It also shows the citation discipline: quoting the projected MTTFs as
  measurements is exactly the slip a careful interviewer catches, and it costs more credibility than
  the number was worth.

- **NVIDIA `dcgm-exporter`, `etc/default-counters.csv`.** The presence metrics ship enabled; the
  SM-breadth metrics ship commented out. **What it shows:** why D2's scenario is so common — the
  metric that cannot see the problem is the default, and the one that can is opt-in. It is also the
  answer to "why does anyone ship it": the default is the cheapest thing that is always available.

- **`NVIDIA/dcgm-exporter` issue #34 — profiling privileges.** An exporter failing to start on an
  A100 node, printing that it lacks sufficient privileges to expose profiling metrics and needs
  `SYS_ADMIN`. **What it shows:** D2's fifth suspect — the honest metric silently disappearing
  because of a capability change, producing absence rather than a loud failure, and absence in a
  query is not zero.

- **NVIDIA nccl-tests.** The canonical collective benchmark, used in D1's Q3 branch to turn "the
  network feels slow" into achieved bus bandwidth against the link's theoretical maximum.
  **What it shows:** the difference between suspecting the fabric and measuring it — the step that
  converts a guess into a bisection.

- **NVIDIA DCGM diagnostics levels.** Level 1 quick and idle-safe, level 3 a load and stress run.
  **What it shows:** the mechanism behind D3's central constraint — the diagnostic that would catch
  the fault is the one you cannot run while the node is serving, which is precisely why the
  association signal has to exist.

## Worked example

**A full D2 rep, transcript style, twelve minutes.** This is the shape to reproduce.

**0:00 — reject the metric, establish ground truth.**

> *"Before anything else: the utilisation panel being unchanged at 100% is not evidence that nothing changed. That field is a passthrough of a driver counter that reports whether at least one kernel was resident during a short sample window — it's a threshold at one, with no notion of how many SMs exist. It reads near 100 on a healthy fleet and on a completely stalled one. So I'll treat it as uninformative and go to throughput. What are tokens per second doing, week over week?"*

**1:30 — confirmed: throughput down 30%, no deploy in the window.**

> *"Good, so the regression is real and it's not a metric artefact. One cheap check before I hunt a workload cause: are the honest metrics still being exported? Profiling fields get dropped rather than zeroed when they're unavailable, so a capability change or an exporter restart makes them silently absent, and an average just skips those GPUs. `count by (instance, gpu)` on a device field, `unless` the same on the profiling field — if that returns anything, that's my answer and it's a config change, not a workload problem."*

**2:30 — the honest metrics are present.**

> *"Then I want the shape. `dcgmi dmon` with presence, SM breadth, occupancy, tensor and DRAM, plus framebuffer, for a minute. I'm not looking at any single value — I'm looking at which quadrant we're in. Breadth against DRAM splits four suspects and tensor breaks the tie."*

**3:30 — output: presence 99, breadth 0.71, tensor 0.02, DRAM 0.11, framebuffer high.**

> *"That's quadrant three: SMs broadly lit, tensor pipes essentially idle, HBM quiet. The SMs have warps assigned and those warps aren't doing math and aren't waiting on memory — so they're waiting on each other or on something off-device. That's a synchronisation or communication stall, and it's consistent with 'overnight, nothing changed', because a node degrading isn't a change anyone made. What it rules out: it's not memory-bound serving, and it's not starvation, because starvation would show low breadth."*

**5:00 — the branch.**

> *"Two candidates inside that quadrant: a collective blocked on a straggler, or a synchronisation point in the code. The code hypothesis is weak here because nothing was deployed, so I'd test the straggler hypothesis first — per-rank step times, looking for one consistent outlier rather than noise."*

**6:30 — output: rank 47 is 34% slower per step, consistently, for the whole window.**

> *"That's it, and it explains everything upstream. All the other ranks block in the collective waiting for 47; the wait is implemented as a resident kernel, so every one of them reports full presence and lit SMs while doing no math — which is exactly the shape on screen. Throughput falls to whatever rank 47 can sustain, which matches a 30% drop."*

**8:00 — the node, not the rank.**

> *"Now: is rank 47 slow because of its node or because of its data? I'd check the node's clocks against boost and any throttle reason, its link width, and its correctable error rate. If clocks are capped under load, that's thermal or power. If the link renegotiated, that's a fabric fault. If nothing shows, I'd drain it and run the load-level diagnostic, and I'd note that the diagnostic is a stress test so it needs the node drained first."*

**10:00 — close with prevention, which is the part that reads senior.**

> *"Two changes so this doesn't recur silently. First, alerting: page on sustained near-zero SM breadth gated on framebuffer, warn on tensor activity, and never alert on the presence metric — a rule built on that field will never fire on this class of failure, because the failure keeps a kernel resident by construction. Second, the straggler signal itself: per-rank step-time variance as a first-class metric, so 'the job is slow' becomes 'rank 47 on node-19 is 34% slower' automatically. And I'd add the absence alert from minute one as a standing rule, because the version of this incident where the honest metric is simply gone looks identical from the dashboard."*

**Self-scoring this rep:**

| Behaviour | Present? |
|---|---|
| Rejected the misleading metric with a *mechanism*, not an adjective | ✓ |
| Established ground truth before hunting | ✓ |
| Checked for metric absence before assuming a workload cause | ✓ |
| Stated hypothesis → command → expected signal → branch each time | ✓ |
| Said what each observation *ruled out*, not only what it suggested | ✓ |
| Separated "which rank" from "why that rank" | ✓ |
| Named the stress-test constraint on the diagnostic unprompted | ✓ |
| Closed on prevention with a specific alerting policy | ✓ |

That is a clean rep. The two most commonly missing rows are the absence check and the explicit
"what this rules out" — both are cheap to add and both are visible in a recording.

## Practice

1. **Draw all three trees from memory** onto one page each, with the branch condition written on
   every edge. Commit them to `gpu-debug-decision-trees`. A tree with unlabelled edges is a
   checklist, not a decision procedure — redo it.

2. **Drill the command→signal→branch table** until each row comes out in one breath. Have someone
   name a command and answer with the signal and both branches, and vice versa.

3. **Run the live-terminal protocol on each scenario**, recorded, twenty-minute timer, two clean
   reps each. Grade only on whether a listener could follow the reasoning without asking "why did
   you run that?"

4. **Write your D2 walkthrough as an eight-minute spoken script.** It is the flagship trap and the
   likeliest scenario. Rehearse the first fifteen seconds until the mechanism-level rejection of the
   metric is automatic.

5. **Drill the interruptions separately.** Have someone read you the interruption prompts from
   §3–§5 in random order, cold. The interruptions are where the round is decided, and rehearsing an
   answer without rehearsing its defences is half the work.

6. **Rehearse the citation precisely.** The lemon-node result and the MTTF figure, with the study
   named, the cluster scale, the window — and which numbers are measured versus projected. Then
   rehearse the follow-up: what detection signals produced that result.

7. **Practise abandoning a lead.** Run one rep where the offered hypothesis is wrong and the correct
   move is to say "this isn't significant, I'd look elsewhere." Being willing to drop the
   interviewer's framing is a scored behaviour and it feels unnatural under pressure.

8. **Feed all three trees** into the [GPU platform capstone](../practice/gpu-platform-capstone/README.md)
   as the incident-round appendix.

**Acceptance:** three trees drawn from memory with labelled edges · the command table at instant
recall · two clean recorded reps per scenario · a D2 script deliverable in eight minutes · every rep
closing on a prevention · the lemon-node citation accurate down to measured-versus-projected.

## Common pitfalls

1. **Running a command before stating the hypothesis it tests.** **Mechanism:** the round's purpose
   is to predict your performance on an unseen failure, and only narrated reasoning generalises — a
   silent command followed by a correct conclusion is indistinguishable from a guess. **Symptom:**
   the interviewer asks "why did you run that?" **Fix:** the five-second habit — after every command,
   before reading output, say "what I'm looking for is…".

2. **Skipping the cheap check because you are confident.** **Mechanism:** the tree's value is the
   ordering, and the ordering is a cost argument as much as a diagnostic one — the synthetic-data
   test eliminates half the search space for one run. **Symptom:** you reach for a profiler in the
   first two minutes. **Consequence:** even when you are right, it reads as impatience rather than
   efficiency.

3. **Distrusting the metric and then guessing anyway.** **Mechanism:** pivoting away from the
   presence metric is necessary and not sufficient; the bisection is what separates reasoning from a
   lucky guess. **Symptom:** "I don't trust util — it's probably the network."

4. **Treating SM breadth as the new truth.** **Mechanism:** it is an occupancy measure, not a
   productivity one — an SM with one stalled warp counts as active exactly as much as one with 64
   running warps. **Symptom:** you declare a job healthy at 0.9 breadth while its tensor pipes are at
   0.04, which is the signature of a stall.

5. **Not checking for metric absence.** **Mechanism:** profiling fields are dropped rather than
   zeroed, and absence is not zero in a query — an average silently skips the missing GPUs.
   **Symptom:** you spend twenty minutes hunting a workload cause for numbers that moved because a
   capability was dropped.

6. **Treating "passes idle health checks" as exculpatory.** **Mechanism:** it is the clue, not the
   dead end — it narrows the fault class to load-dependent ones. **Symptom:** "the node checks out,
   so it must be the job."

7. **Counting failures without handling the confound.** **Mechanism:** large jobs touch more nodes
   and fail more often, so raw appearance counts implicate the busiest nodes regardless of health.
   **Symptom:** your lemon detector flags exactly the nodes in the most jobs.

8. **Quoting the 14%-to-4% result without the mechanism.** **Mechanism:** the number's credibility
   comes from the detection method — association across job failures plus load-based confirmation —
   not from the number itself. **Symptom:** you drop the statistic and stall on "what drove it?",
   which costs more credibility than the citation gained.

9. **Ending on the diagnosis.** **Mechanism:** diagnosis is table stakes at this level; converting
   one incident into a detected class is the senior move. **Symptom:** you find the straggler and
   stop, without naming the alert that would have caught it.

## Self-check

- **What single discipline separates a strong live-debugging answer from a weak one, independent of
  whether the diagnosis is correct?** *Answer:* stating the hypothesis, the command that would test
  it, and the signal you expect — **before** running it — then naming which branch each possible
  result selects. The reason is that the round exists to predict your performance on a failure you
  have not seen, and only narrated reasoning generalises: a correct diagnosis reached silently is
  indistinguishable from a lucky guess. Two supporting habits: say what an observation *rules out*,
  not only what it suggests; and say out loud when you are surprised and what the surprise
  eliminates.

- **In D1, what is the first substantive check and why that one?** *Answer:* the synthetic-data test
  — replace the dataset with in-memory random tensors of the same shape and re-measure step time.
  It goes first because data starvation is the most common cause of a slow training job *and* it is
  the cheapest test available, and because it eliminates the entire input path in one run: the
  loader, storage bandwidth, CPU preprocessing and host-to-device copies. If throughput jumps, the
  bottleneck is upstream of compute and you never need to touch a profiler. If it does not move, you
  have halved the search space for the cost of one run. Before even that, confirm the regression is
  real using throughput or step time rather than the utilisation panel.

- **In D2, why can the presence metric read 100 while throughput drops, and what pair of fields
  separates the suspects?** *Answer:* because the field reports whether at least one kernel was
  resident during a short driver-chosen sample window — a threshold at one, with no notion of SM
  count — and every expensive failure mode keeps a kernel resident: a collective blocked on a
  straggler, a spin-wait, a memory-starved decode loop, a dataloader-starved training step. The pair
  that separates them is SM breadth against DRAM activity, with tensor activity as the tiebreak.
  Breadth low and DRAM low is starvation or launch overhead; breadth low and DRAM high is
  small-batch memory-bound serving; breadth high with DRAM and tensor low is a synchronisation or
  communication stall; breadth high with DRAM high and tensor modest is real work in the wrong shape
  or precision. And there is a fifth case off the plot entirely: the honest metric stopped being
  exported, because profiling fields are dropped rather than zeroed and absence is not zero in a
  query.

- **In D3, what makes a lemon node hard to detect, what catches it, and what is the confound?**
  *Answer:* by construction it passes point-in-time health checks, because the fault is
  load-dependent — thermal, power delivery, marginal links, memory errors at bandwidth — and the
  load-level diagnostic that would reproduce it is a stress test that cannot run on a node serving a
  job. What catches it is an *associative* signal: the failure rate of jobs that touched the node
  compared against the fleet baseline over a window, confirmed afterwards by draining and running
  the load diagnostic. The confound is that large jobs touch many nodes and fail more often for
  unrelated reasons, so naive appearance counts implicate the busiest nodes rather than the sick
  ones — you need a rate against a baseline, with enough job-touches for significance.

- **What is the published lemon-node result, from which study, and what is the caveat?** *Answer:*
  proactive lemon-node detection reduced the failure rate of jobs at 512 GPUs and above from about
  14% to about 4%, identifying roughly forty faulty nodes at over 85% detection accuracy — from
  Meta's *Revisiting Reliability in Large-Scale Machine Learning Research Clusters*
  (arXiv:2410.21680, HPCA 2025), measured across two clusters of roughly 16K and 8K A100 GPUs over
  eleven months. The same paper measures a mean-time-to-failure of 7.9 hours for 1,024-GPU jobs and
  *projects* 1.8 hours at 16,384 and 0.23 hours at 131,072 — quoting those projections as
  measurements is the common error. The caveat that matters operationally: at that detection
  accuracy there are false positives, so the design must make a false positive cheap — a
  drain-and-diagnose queue rather than a direct path to replacement.

- **Why does the remediation ladder stop automating before the last rung?** *Answer:* because the
  cost asymmetry flips. Soft remediation (device reset, agent restart) and medium remediation
  (cordon, drain, re-place) are cheap to get wrong — a spurious cordon costs one node's capacity for
  a window — and frequent enough that automation pays. Hardware replacement is expensive to get
  wrong, involves a supplier relationship, and is rare enough that a human review costs little in
  aggregate. The other constraint on the medium rung is that draining a node participating in a
  distributed job means re-placing the *whole gang*, not one pod, or you recreate the partial
  placement hang that gang scheduling exists to prevent — so the automation has to cooperate with
  the scheduler rather than act on pods directly.

- **How do you close a debugging answer?** *Answer:* with prevention, stated as a specific
  detection change rather than a resolution. Diagnosis is table stakes; converting one incident into
  a class that gets caught automatically is the senior signal. Concretely for D2: alert on sustained
  near-zero SM breadth gated on framebuffer so a warm loaded replica is not reclaimed; warn rather
  than page on tensor activity because efficiency is a conversation not an incident; never alert on
  the presence metric because a rule built on it cannot fire on this failure class by construction;
  add an absence alert because the version of the incident where the honest metric simply vanished
  looks identical from the dashboard; and promote per-rank step-time variance to a first-class
  metric so "the job is slow" becomes "rank 47 is 34% slower" without a human bisecting.

## Connections & what's next

This lesson is the incident mirror of lesson 05's design drills: where 05 trains volunteered
architectural reasoning, 06 trains narrated diagnostic reasoning under a clock, and together they
cover the two rounds that decide most GPU-infra loops. The trees lean directly on module 05's field
semantics (which is why a weak D2 usually traces back to that module rather than to this one), on
module 08's failure taxonomy, and on module 04's failure-mode log — which is the artifact that makes
"tell me about a GPU incident you actually debugged" answerable with a specific story instead of a
general one. The prevention close on each drill is also the bridge into lesson 07: an incident you
turned into a detection rule is an artifact narration, not just an anecdote.

Next: [07 — Narrate your artifacts](07-narrate-artifacts.md), where the concrete evidence behind
both the design skeletons and these debugging trees gets turned into a spoken decision rather than a
build log.

## References & further reading

**Primary sources**

- Meta — *Revisiting Reliability in Large-Scale Machine Learning Research Clusters* (arXiv:2410.21680, HPCA 2025) — https://arxiv.org/abs/2410.21680 — read for: the measured 7.9-hour MTTF at 1,024 GPUs, the projections at larger scale, and the lemon-node result (large-job failure rate from ~14% to ~4%, ~40 nodes at >85% accuracy) that D3 rests on. *Correction vs earlier versions of this lesson: the 1.8-hour figure is the paper's projection, not a measurement, and the clusters studied are A100-based research clusters.*
- NVIDIA DCGM user guide — diagnostics — https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html — read for: what each `dcgmi diag` level actually runs, and why level 3 is a stress test that needs a drained node.
- NVIDIA DCGM, `dcgmlib/dcgm_fields.h` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_fields.h — read for: the field IDs used in the `dcgmi dmon` invocations, and the exact definitions behind D2's four-suspect split.
- NVIDIA `dcgm-exporter`, `etc/default-counters.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — read for: why the metric that cannot see D2's failure is the default and the one that can is opt-in.
- `NVIDIA/dcgm-exporter` issue #34 — https://github.com/NVIDIA/dcgm-exporter/issues/34 — read for: the `SYS_ADMIN` requirement behind D2's fifth suspect, and the half-populated-dashboard failure mode.
- NVIDIA nccl-tests — https://github.com/NVIDIA/nccl-tests — read for: `all_reduce_perf` usage and the size sweep that turns a suspicion about the fabric into a measured bandwidth.
- NVIDIA Nsight Systems documentation — https://docs.nvidia.com/nsight-systems/ — read for: what the timeline separates, and what a gap between kernels actually indicates.

**Course-internal sources**

- `platform-eng/modules/05-gpu-observability/lessons/01-lie-and-truth.md` — the field-level basis of D2's entire premise, including the quadrant diagram and the alerting rules quoted in the prevention closes.
- `platform-eng/modules/04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md` — the failure-mode log that makes "an incident you actually debugged" a specific story, and the drain-and-re-place interaction with gang scheduling.

**Not relied upon**

- Vendor product pages and third-party incident-response playbooks describing straggler detection
  and tiered remediation were consulted for the tooling-evolution observation in Perspectives. The
  trend is stated as a general industry pattern rather than as a verified claim about any named
  product's current feature set, and no diagnostic step above depends on them.
