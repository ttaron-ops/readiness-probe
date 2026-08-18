---
lesson: "A03.9"
title: "GPU and ML observability at fleet scale"
module: "A-03"
concept: "fleet GPU observability"
status: not-started
est_time: "6 hrs"
prev: "08-profiling-and-ebpf.md"
next: "10-telemetry-lakehouse.md"
artifacts: ["fleet GPU + NCCL signal plan", "goodput-regression burn-rate alert", "straggler-detection query"]
sources: 13
---

# A03.9 · GPU and ML observability at fleet scale

> **Concept.** Take single-node GPU truth (SM occupancy, MFU/goodput, XID) and make it survive thousands of GPUs: bounded cardinality, sharded scrape, per-tenant multitenancy, comms/straggler visibility, and alerts that page on burned GPU-hours instead of a moved gauge.
>
> Module: [🔭 Observability engineering](../README.md) · Track A — Platform excellence

## Where this fits

This is the module's synthesis lesson. It introduces no new signal *type*; it forces every discipline from L1–L8 to survive contact with a real fleet. Cardinality budgets (L1/L3) meet the highest-cardinality label set in the stack. PromQL correctness (L2) meets gauges that get `rate()`'d by habit. Mimir sharding and multitenancy (L3) meet a workload that can blow a tenant's series budget in one bad rollout. Collector-side enrichment (L4) meets the scheduler join that turns "GPU 3 is hot" into "team-b's job owns GPU 3." Exemplars (L5), log discipline (L6), burn-rate alerting (L7) and fleet-wide profiling (L8) all reappear, applied to the one workload where getting them wrong costs thousands of dollars an hour.

**Scope boundaries, stated up front, because two other modules own the neighbouring ground.**

- `modules/05-gpu-observability` owns the **single-node truth**: what DCGM is and how the host engine and profiling module work, why `DCGM_FI_DEV_GPU_UTIL` is a *presence* metric rather than an intensity one, the PROF field semantics and counter-bank co-residency, per-node MFU/goodput definitions, XID meaning, the pod-resources attribution join, inference SLOs (TTFT/ITL/TPOT and the exact current vLLM metric names), and the allocated-vs-utilised GPU-hours integral. **This lesson assumes all of it and never re-derives it.**
- `modules/01b-linux-internals` lessons 08–09 and this module's [lesson 08](08-profiling-and-ebpf.md) own profiling. Here, profiling is one input among several.

What this lesson adds is the layer above both: **the signal stack of a whole ML system**, from hardware counters up to model-quality signals; the layers that only exist once there is more than one node (collectives, ranks, schedulers, fleets of replicas); the join keys that make those layers correlatable; and the reliability arithmetic that decides where to spend observability effort when hardware failure is a certainty rather than an anomaly.

Facts below are checked against **NCCL** (`master`, August 2026 — `src/include/plugin/nccl_profiler.h`, `src/plugin/profiler.cc`), **PyTorch** (`main` — `torch/csrc/distributed/c10d/FlightRecorder.hpp`, `ProcessGroupNCCL.hpp`, `ProcessGroupNCCL.cpp`), **Meta GCM** (`main` — `README.md`, `slurmprocessor/README.md`), and the **OpenTelemetry GenAI semantic conventions** (`open-telemetry/semantic-conventions-genai`, `docs/gen-ai/gen-ai-metrics.md`), all read from upstream repositories. Where a figure comes from a paper this environment cannot reach, it is marked as such and its provenance stated.

## Why this matters

A single-node GPU dashboard that is *correct* still tells you almost nothing about a 2,000-GPU training run, because a synchronous job runs at the speed of its slowest rank and that rank is invisible in a fleet average. The failure mode at staff level is not "we don't have GPU metrics" — it is:

> we have DCGM on every node, Prometheus fell over from `gpu_uuid` cardinality, nobody can attribute a hot GPU to a team, and the run lost 18% goodput to one fail-slow NVLink for six hours before anyone paged.

Every clause in that sentence is a different layer of the stack failing, and no single layer's tooling would have caught more than one of them. That is the lesson's thesis: **fleet ML observability is a correlation problem across layers, not a collection problem within one.**

There is also an arithmetic reason it matters more here than anywhere else in the module. A synchronous training job has no partial degradation: if one rank in 512 is 30% slow, the *whole job* is 30% slow, and the other 511 GPUs bill at full price while producing nothing extra. Nowhere else in infrastructure does a single-component fault multiply its cost by 512.

## What's new here (calibration)

- **Assumed from module 05:** DCGM architecture, the util-lie, PROF field semantics, XID, per-node MFU/goodput, the pod-resources join, inference SLO metrics and the vLLM V1 renames, the allocated-vs-utilised integral.
- **New: the ML observability signal stack as a design object** — six layers, each with the question it answers, the question it structurally cannot answer, its characteristic detection latency, and its cardinality cost. This is the artefact you defend in a design review.
- **New: detection-to-recovery as a derived budget.** From a stated interruption rate and a stated effective-training-time target, derive the *minutes per interruption* you are allowed to spend on detect + restart + recompute — and show that the checkpoint interval usually consumes most of it before observability gets any.
- **New: the framework's own default timeouts as your MTTD floor.** `kProcessGroupNCCLDefaultTimeout = 10 min`, `TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC = 480 s`, `kWorkStatusUpdatePeriodMs = 30 s`, watchdog poll 100 ms — all read from PyTorch source. If your detection budget is under ten minutes and you have not changed these, the framework is your bottleneck.
- **New: the collective-level trace.** PyTorch's Flight Recorder (`TORCH_FR_BUFFER_SIZE` / `TORCH_NCCL_TRACE_BUFFER_SIZE`, default **2000** entries per rank) and NCCL's profiler plugin API (`NCCL_PROFILER_PLUGIN`, event mask `ncclProfileColl | ncclProfileProxyOp | ncclProfileProxyStep | ncclProfileKernelCh | …`, v7). Including the buffer-sizing arithmetic that decides whether a hang leaves usable evidence.
- **New: straggler detection as statistics, not a magic 1.3×** — robust thresholds from median and MAD, and the multiple-comparisons correction you need at 512 ranks.
- **New: the serving and model-quality layers**, and why the top of the stack cannot be burn-rate alerted at all (label latency), so it needs a different alerting shape.
- **New: the join-key spine** — one identity per layer, where each key is allowed to live (metric label / exemplar / log field / lakehouse column), and the misattribution window that a caching enrichment processor creates at job boundaries.

## Core concepts

### 1. The ML observability signal stack

Every fleet incident is a question at one layer whose answer lives at another. Drawing the layers, with what each can and cannot answer, is the single most useful artefact in this lesson.

```
   THE ML-SYSTEM SIGNAL STACK — WHAT EACH LAYER CAN AND CANNOT ANSWER
   ═══════════════════════════════════════════════════════════════════════════════

  ┌── L6 · MODEL QUALITY ────────────────────────────────────────────────────────┐
  │ signals  eval scores, output-distribution drift, input drift, refusal rate,  │
  │          human feedback, task success                                        │
  │ answers  "is the system doing its JOB?"                                      │
  │ cannot   say anything about cost, or locate a cause                          │
  │ latency  hours → days (label latency)      cardinality  low (model × slice)  │
  ├── L5 · SERVING / APPLICATION ────────────────────────────────────────────────┤
  │ signals  TTFT, ITL, TPOT, queue depth, KV-cache pressure, preemptions,       │
  │          tokens in/out            (OTel: gen_ai.server.*, gen_ai.client.*)   │
  │ answers  "is the user promise being kept, and where in the request?"         │
  │ cannot   distinguish a slow GPU from a slow tokenizer                        │
  │ latency  seconds                  cardinality  model × engine × route        │
  ├── L4 · JOB / SCHEDULER ──────────────────────────────────────────────────────┤
  │ signals  allocation, restarts, preemptions, queue wait, checkpoint I/O,      │
  │          job lifecycle events (K8s pod events / Slurm job states)            │
  │ answers  "was the work even RUNNING, and who owns it?"                       │
  │ cannot   see inside a step                                                   │
  │ latency  seconds → minutes        cardinality  jobs × tenants (churny!)      │
  ├── L3 · FRAMEWORK / STEP ─────────────────────────────────────────────────────┤
  │ signals  per-rank step time, loss, grad-norm, dataloader wait, optimizer     │
  │          time, checkpoint duration                                           │
  │ answers  "is the training loop progressing at the expected rate?"            │
  │ cannot   say WHY a step is slow                                              │
  │ latency  one step (0.1–5 s)       cardinality  job × rank  ← the trap        │
  ├── L2 · COLLECTIVES / INTERCONNECT ───────────────────────────────────────────┤
  │ signals  per-collective duration, proxy step states, NVLink/PCIe/IB counters,│
  │          flight-recorder entries, comm ranks and seq numbers                 │
  │ answers  "WHICH RANK is late, and in which phase of which collective?"       │
  │ cannot   say why that rank is late (host? device? link?)                     │
  │ latency  one collective (ms)      cardinality  job × comm × rank × op  ⚠     │
  ├── L1 · DEVICE COUNTERS ──────────────────────────────────────────────────────┤
  │ signals  SM_ACTIVE, PIPE_TENSOR_ACTIVE, memory, power, clocks, temperature,  │
  │          throttle reasons, ECC, XID, NVLink errors    (module 05's material) │
  │ answers  "what is this GPU physically doing / is it healthy?"                │
  │ cannot   attribute to a job, or tell useful work from spinning               │
  │ latency  1 scrape (15–30 s)       cardinality  node × gpu (× MIG)            │
  ├── L0 · HOST ─────────────────────────────────────────────────────────────────┤
  │ signals  CPU profiles (on/off-CPU, lesson 08), page cache, NIC, storage      │
  │ answers  "is the host starving the device?"                                  │
  │ latency  continuous                cardinality  node × process               │
  └──────────────────────────────────────────────────────────────────────────────┘

   THE RULE THAT FALLS OUT:
     · Every layer can DETECT a problem in the layer below it, as a symptom.
     · No layer can DIAGNOSE its own problem — that always needs the layer below.
     · So: ALERT high in the stack (L3–L6, where cost and user pain live),
           DIAGNOSE downward (L2 → L1 → L0),
           and pay for exactly enough cardinality at each layer to make the
           downward step possible.
```

Read the two `cardinality` warnings. **L2 and L3 are where fleet GPU observability actually dies**, not L1. A DCGM series set is large but bounded — `nodes × gpus × metrics`. A per-rank, per-collective series set is `jobs × ranks × comms × op-types`, and jobs churn. §2 does that arithmetic.

Read the L6 `latency` line too. **Model quality is the only layer whose signal arrives after the fact**, sometimes days after, which breaks the burn-rate alerting model from lesson 07 entirely. §9 deals with that.

### 2. Cardinality at the layers that matter

Lesson 01 gave the identity (`series = ∏ distinct label values`) and lesson 03 gave the fall-over sequence. Module 05 audited the per-node DCGM label set. What is left, and what is genuinely fleet-specific, is the arithmetic at L2–L4 — the layers that do not exist on one node.

```
   SERIES BUDGET FOR A 4,000-NODE / 32,000-GPU FLEET
   ═══════════════════════════════════════════════════════════════════════════
   L1 · DEVICE (bounded — the easy layer, despite its reputation)
        32,000 GPUs × ~27 exported metric names                = 864,000 series
        with MIG at up to 7 instances on the sliced subset,
        assume 10 % of GPUs sliced: +3,200 × 6 × 27           = +518,400
                                                              ──────────────
                                                              ≈ 1.38 M series
        ⇒ large, FLAT, and stable. Churns only on RMA and reimage.

   L2 · COLLECTIVES (unbounded if you are naive — the real bomb)
        20 concurrent jobs × 512 ranks × 3 comms (DP/TP/PP)
          × 6 op types × 3 metrics                            = 553,000 series
        …per job generation. Jobs restart, and `job_id` changes:
        at 8 restarts/day the DAILY series creation is        ≈ 4.4 M
        ⇒ head-block churn, not steady state, is what kills you here (L3 §3).

   L3 · PER-RANK STEP TIME (the one everybody adds without thinking)
        20 jobs × 512 ranks × 1 metric                        = 10,240 series
        …which looks harmless — and IS, until someone adds `step`, or
        `micro_batch`, or emits a histogram per rank (12 buckets → ×12).

   THE DESIGN THAT SURVIVES
     L1  keep: node, gpu, gpu_family, tenant        drop: UUID, pod, container_id
     L2  DO NOT emit per-(rank × op) series at all. Emit per-JOB aggregates
         (p50/p99/max of collective duration) plus EXEMPLARS carrying rank.
     L3  per-rank step time IS worth its 10 k series — it is the straggler
         signal — but ONLY as a gauge, per (job, rank), with no further labels,
         and with a hard rule that `job_id` is bounded by admission control.
     L4  job/pod identity lives in an INFO metric and in the lakehouse (L10),
         not multiplied onto every measurement.
```

**The general rule this fleet teaches**: at each layer, ask *what is the smallest label set that still lets me take one step down the stack?* For L2 the answer is not "every rank" — it is "the job, plus a pointer (exemplar) to the rank that was slow." The pointer costs one exemplar per series per scrape; the naive version costs 553,000 series.

The scrape-side enforcement is the same shape as lesson 03's, and belongs in the design doc as config rather than prose:

```yaml
# Prometheus agent shard N of 8 — hashmod over the GPU-node estate only.
scrape_configs:
  - job_name: dcgm
    scrape_interval: 30s          # = dcgm-exporter --collect-interval; keeps the
                                  # GPU-hours integration constant at 30/3600
    kubernetes_sd_configs: [{ role: pod }]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: dcgm-exporter
        action: keep
      - source_labels: [__address__]
        modulus: 8
        target_label: __tmp_shard
        action: hashmod
      - source_labels: [__tmp_shard]
        regex: 3                  # ← this shard's index
        action: keep
    metric_relabel_configs:
      # Unbounded identity: off the series, into an info metric / the lake.
      - regex: "UUID|uuid|container_id|pod_hash|DCGM_FI_DEV_SERIAL"
        action: labeldrop
      # modelName is few-valued but 20+ bytes on EVERY series (L1 §2's
      # label-text term). Collapse to a short token.
      - source_labels: [modelName]
        regex: "NVIDIA (A100|H100|H200|L40S).*"
        target_label: gpu_family
        replacement: "$1"
      - regex: "modelName"
        action: labeldrop
```

### 3. Detection-to-recovery is the objective — and it has a derivable budget

At single-node scale a GPU dying is an incident. At fleet scale it is a Tuesday, and the observability question changes from *prevent* to *how fast from fault to running again*.

The public anchor is Meta's Llama 3 infrastructure report: on a **16,384-H100** cluster over **54 days**, **466 job interruptions**, of which **419 were unexpected**; a hardware-dominated breakdown of those (GPU including NVLink ≈ 30%, HBM3 ≈ 17%, GPU SRAM ≈ 4.5%, GPU system processor ≈ 4%, network switch/cable ≈ 8.4%); **>90% effective training time**; and only **3 manual interventions** across the whole run.

> **Provenance note.** These figures are quoted from *The Llama 3 Herd of Models* (arXiv:2407.21783) and are carried forward from this course's earlier verified pass. **arxiv.org is blocked by the egress proxy in this environment, so they could not be re-fetched or re-checked here.** Treat them as inputs to the derivation below; the derivation itself is what you should carry away, because you can re-run it with your own fleet's numbers.

**Now derive what those numbers demand of you.**

```
   FROM AN INTERRUPTION RATE TO A DETECTION BUDGET
   ═══════════════════════════════════════════════════════════════════════════
   Inputs (substitute your own):
     unexpected interruptions      I = 419 over 54 days
     effective-training-time target        E ≥ 90 %

   Step 1 — interruption frequency
     419 / 54 d = 7.76 per day  =  one every 3.09 hours, continuously.

   Step 2 — the per-interruption time budget
     lost fraction = I_per_day × L_hours / 24        where L = lost hours each
     require  lost fraction ≤ 1 − E = 0.10
       ⇒ L ≤ 0.10 × 24 / 7.76 = 0.309 h = 18.6 MINUTES PER INTERRUPTION,
         averaged over detect + decide + restart + re-warm + RECOMPUTE.

   Step 3 — subtract what the checkpoint policy already spends
     On an unplanned stop you lose, in expectation, HALF the checkpoint
     interval of recomputation:
       checkpoint every 30 min ⇒ E[recompute] = 15 min
       leaving 18.6 − 15 = 3.6 min for EVERYTHING ELSE.
       checkpoint every 60 min ⇒ E[recompute] = 30 min > 18.6 → 90 % is
                                  UNREACHABLE regardless of observability.
       checkpoint every 10 min ⇒ E[recompute] =  5 min,
                                  leaving 13.6 min of detect+restart headroom.

   Step 4 — the conclusion that reframes the whole lesson
     ⇒ The checkpoint interval and the detection latency trade against each
       other inside ONE budget. "Improve observability" and "checkpoint more
       often" are the same lever pulled from two ends, and you cannot argue
       about either without this arithmetic.
     ⇒ At a 30-minute checkpoint interval you have ~3.6 minutes to detect a
       fault, decide, and restart. §4 shows that the DEFAULT framework
       timeouts alone are 8–10 minutes.
```

Two structural conclusions follow, and they decide where the money goes:

- **Fail-stop is largely solved; fail-slow is not.** Three manual interventions across 419 unexpected interruptions means automation handled essentially all of the *rank died outright* cases: liveness checks catch them, the scheduler restarts, the run resumes. What no health check catches is a rank that is still alive, still responding, still passing every probe — and 30% slow. That is invisible to L4 and L1 and shows up only as *variance* at L2/L3.
- **XID and hardware error events are a continuous stream, not an alertable event.** At one interruption every ~3 hours on a 16k-GPU cluster, per-event alerting on XIDs generates a page volume nobody can consume. They belong in a **rollup correlated with goodput impact** — the same "aggregate, don't page per event" discipline lesson 07 applies to any high-frequency signal.

### 4. The framework's default timeouts are your detection floor

This is the most actionable paragraph in the lesson, and it comes straight from PyTorch's source rather than from folklore.

| Constant / env var | Default | Where | What it bounds |
|---|---:|---|---|
| `kProcessGroupNCCLDefaultTimeout` | **10 min** | `ProcessGroupNCCL.hpp` | how long a collective blocks before the watchdog declares it timed out |
| `TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC` | **480 s** (8 min) | `ProcessGroupNCCL.cpp` | monitor thread's patience before it decides the watchdog itself is stuck and tears the process down |
| `kWorkStatusUpdatePeriodMs` | **30 s** | `ProcessGroupNCCL.hpp` | how often work-status is refreshed/logged |
| `kWatchdogThreadSleepMillis` | **100 ms** | `ProcessGroupNCCL.cpp` | watchdog poll interval over in-flight work |
| `TORCH_NCCL_ASYNC_ERROR_HANDLING` | **3** (`SkipCleanUp`) | `ProcessGroupNCCL.cpp` | on error, tear down the process *without* cleaning up NCCL communicators (because `ncclCommAbort` can itself hang) |
| `TORCH_NCCL_DESYNC_DEBUG` | **false** | `ProcessGroupNCCL.cpp` | on timeout, report which ranks were in which collective |
| `TORCH_FR_DUMP_ON_TIMEOUT` / `TORCH_NCCL_DUMP_ON_TIMEOUT` | **true** | `FlightRecorder.hpp` | dump the flight recorder on collective timeout/failure |
| `TORCH_FR_WAIT_TIMEOUT_DUMP_MILSEC` / `TORCH_NCCL_WAIT_TIMEOUT_DUMP_MILSEC` | **15 s** | `FlightRecorder.hpp` | how long a dump attempt may take before being abandoned |

Put those against §3's budget:

```
   A HANG, ON DEFAULTS — TIMELINE
   ═══════════════════════════════════════════════════════════════════════════
   t+0:00   rank 271's NIC wedges. It stops entering collectives.
            The other 511 ranks are now blocked inside ncclAllReduce.
            EVERY EXTERNAL SIGNAL LOOKS FINE:
              · pod is Running, liveness probe passes  (L4 ✓)
              · GPU_UTIL reads ~100 % — a kernel is resident, spinning  (L1 ✓)
              · SM_ACTIVE may read high — the collective kernel occupies SMs
              · loss/step metrics simply STOP being emitted  (L3 — absence,
                which is the only honest signal, and absence is exactly what
                naive dashboards render as "flat line", not "missing")
   t+0:30   kWorkStatusUpdatePeriodMs tick: work status refreshed/logged.
   t+8:00   TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC (480 s) may fire if the watchdog
            itself is stuck → monitor thread tears the process down.
   t+10:00  kProcessGroupNCCLDefaultTimeout: the collective times out.
            TORCH_NCCL_DUMP_ON_TIMEOUT (default true) dumps the flight
            recorder, bounded by 15 s.
   t+10:15  process exits (ASYNC_ERROR_HANDLING=3 → tear down, skip cleanup).
   t+10:15  ONLY NOW does the scheduler see a dead pod and start a restart.

   ⇒ 10+ minutes of 512 GPUs producing nothing, BEFORE recovery even begins,
     against a §3 budget of 3.6 minutes for detect+restart. Blown by ~3×.
     At $2.50/GPU-hour that hang costs 512 × (10.25/60) × 2.50 ≈ $219 before
     anything reacts — and this happens every few hours somewhere in a fleet.
```

**The fix is not "lower the timeout" alone.** A collective timeout that is too aggressive kills healthy jobs during legitimate long operations (a large all-gather at initialisation, a checkpoint barrier). The fix is a *different, faster signal* that does not wait for the timeout:

1. **Emit a step heartbeat from the training loop** — a counter incremented every step, scraped at 15 s. Then `time() - timestamp(last step increment) > 90s` is a 90-second detector for a hang, versus the framework's 10 minutes. This is the single highest-leverage instrumentation change in this lesson, and it is four lines of Python.
2. **Alert on absence, not on value.** The signature of a hang is a *missing* series, and lesson 02's staleness rules are what make `absent_over_time()` reliable here.
3. **Then** tune the timeouts deliberately, with the known-long operations excluded (per-process-group timeouts exist for exactly this).

### 5. Collective-level observability: the flight recorder and the profiler plugin

When the heartbeat fires, the question is *which rank*, and neither L1 nor L4 can answer it. Two mechanisms at L2 can.

**(a) PyTorch's Flight Recorder — a per-rank ring buffer of collective records.**

Every rank keeps a circular buffer of the last `TORCH_FR_BUFFER_SIZE` (alias `TORCH_NCCL_TRACE_BUFFER_SIZE`) collective entries; the default is **2000** and `0` disables it. Each `Entry` carries, per PyTorch's `FlightRecorder.hpp`: an incrementing `id_` and `reset_epoch_`, the process-group id and `(group_name, group_desc)`, a `collective_seq_id` that increments only for true collectives, a `p2p_seq_id` for point-to-point ops, an `op_id` for logical ops inside a coalesced group, the profiling name, input/output sizes, a `state` of `scheduled` → `started` → `completed`, `duration_ms` when both events could be timed, `time_discovered_started_ns` / `time_discovered_completed_ns`, and a `retired_` flag. `TORCH_FR_CPP_STACK` / `TORCH_NCCL_TRACE_CPP_STACK` (default `false`) additionally captures a C++ stack per entry.

**How you use it is a comparison across ranks, and it is almost mechanical:**

```
   DIAGNOSING A HANG FROM 512 FLIGHT-RECORDER DUMPS
   ═══════════════════════════════════════════════════════════════════════════
   For each rank, take the last entry per process group:

     rank    pg   collective_seq_id  profiling_name      state
     ────────────────────────────────────────────────────────────────
     0       0    184,203            nccl:all_reduce     started      ← waiting
     1       0    184,203            nccl:all_reduce     started      ← waiting
     …       …    …                  …                   …
     270     0    184,203            nccl:all_reduce     started      ← waiting
     271     0    184,202            nccl:all_reduce     completed    ← ★
     272     0    184,203            nccl:all_reduce     started      ← waiting

   READ IT LIKE THIS:
     · 511 ranks are `started` on seq 184,203  → they entered and are blocked.
     · rank 271 completed 184,202 and NEVER ENTERED 184,203.
       ⇒ rank 271 is the culprit: it is stuck BEFORE the collective —
         in the dataloader, in a host-side hang, or dead in a way the
         liveness probe does not see.
     · The opposite pattern — ranks split across TWO different seq_ids or
       different profiling_names — is a DESYNC: the ranks are executing
       different collectives, usually a control-flow divergence in the model
       code (a rank took a different branch). Different bug, different fix.
```

**Size the buffer, because the default is often too small to contain the evidence.** The buffer holds entries, not seconds:

```
   FLIGHT-RECORDER HISTORY DEPTH
   ═══════════════════════════════════════════════════════════════════════════
     collectives per step   C   (DP all-reduce + TP all-gather/reduce-scatter
                                 + PP sends/recvs; commonly 4–40)
     steps per second       S
     history (seconds)  =  TORCH_FR_BUFFER_SIZE / (C × S)

   Example: C = 16, S = 2  ⇒ 32 entries/s
     default 2000 entries  ⇒  62 SECONDS of history.
     A 10-minute collective timeout dumps a buffer whose oldest entry is
     one minute old — the moment the fault occurred has already been
     overwritten by 600 seconds of `started` entries.

   ⇒ Set TORCH_FR_BUFFER_SIZE ≥ (C × S) × (timeout_seconds + margin).
     For the example with a 600 s timeout:  32 × 660 ≈ 21,000 entries.
     Cost: entries are small structs; ~20 k entries per rank is tens of MB of
     host RAM per process — cheap against a $219-per-hang incident. Enable
     TORCH_FR_CPP_STACK only while investigating: stack capture per entry
     multiplies both the memory and the per-collective overhead.
```

**(b) NCCL's profiler plugin API — the layer below the flight recorder.**

NCCL loads a profiler plugin named by the **`NCCL_PROFILER_PLUGIN`** environment variable; the plugin's `init` returns an *event activation mask*, so you pay only for the event classes you ask for. The current interface is **v7** (`src/include/plugin/nccl_profiler.h`), and the event types are a bitmask:

| Event type | Bit | What it observes |
|---|---:|---|
| `ncclProfileGroup` | 1<<0 | a group of operations |
| `ncclProfileColl` | 1<<1 | one host-side collective call |
| `ncclProfileP2p` | 1<<2 | one host-side point-to-point call |
| `ncclProfileProxyOp` | 1<<3 | a proxy operation (the CPU-side network progress engine) |
| `ncclProfileProxyStep` | 1<<4 | one step within a proxy op — **the phase-level detail** |
| `ncclProfileProxyCtrl` | 1<<5 | proxy control state (idle/active/sleep/wakeup/append) |
| `ncclProfileKernelCh` | 1<<6 | per-channel kernel activity |
| `ncclProfileNetPlugin` | 1<<7 | events defined by the network plugin |
| `ncclProfileGroupApi` / `CollApi` / `P2pApi` | 1<<8..10 | API-level entry/exit events |
| `ncclProfileKernelLaunch` | 1<<11 | kernel launch |
| `ncclProfileCeColl` / `CeSync` / `CeBatch` | 1<<12..14 | copy-engine collectives (v6) |
| `ncclProfileKernelPhase` | 1<<15 | kernel barrier phase sub-events (v7) |

And the proxy-step states are where a slow link becomes *legible* rather than merely slow — `ncclProfilerProxyStepSendGPUWait`, `…SendPeerWait`, `…SendWait`, `…RecvWait`, `…RecvFlushWait`, `…RecvGPUWait`. **That state list is the diagnostic vocabulary:** a rank stuck in `RecvWait` is waiting for bytes that never arrived (peer or fabric), while a rank stuck in `SendGPUWait` is waiting for its *own* GPU to produce the data (local compute or memory), and those are opposite investigations. No aggregate duration metric distinguishes them; the phase state does.

**The cost/benefit posture to take.** This is a per-collective instrumentation hook in the hot path of the thing you care about. The event mask exists precisely so you can run `ncclProfileColl` only (one event per collective) fleet-wide, and enable `ProxyStep`/`KernelCh` on a suspect job. NVIDIA's own NCCL Inspector / NIXT work builds on this interface and reports overhead under 2% with validation published at around **2,048 GPUs**, which is below this module's 4,000–10,000-GPU design target — pilot it, do not assume it. *(That figure and its validation scale come from NVIDIA's NIXT material, arXiv:2608.01449 and the NVIDIA developer forum post; both hosts are blocked here and were not re-fetched — see References.)*

### 6. Straggler detection as statistics

"Flag any rank whose step time exceeds 1.3× the job median" is a good starting heuristic and a bad final answer, because 1.3 is not derived from anything and because at 512 ranks you are running 512 simultaneous tests.

**Build it properly in three steps.**

**Step 1 — use a robust dispersion estimate, not the mean and standard deviation.** Step times have a hard floor and a long right tail; a single 3× straggler inflates the standard deviation enough to hide itself. Use the median and the **median absolute deviation**:

```
   MAD  = median_i( | x_i − median(x) | )
   robust z_i = 0.6745 × (x_i − median(x)) / MAD
        (0.6745 makes MAD a consistent estimator of σ for normal data)
```

**Step 2 — set the threshold from the multiple-comparisons requirement, not by taste.** With `R` ranks tested every evaluation, the expected number of false flags per evaluation is `R × P(|z| > k)`. Pick the false-flag rate you will tolerate:

```
   THRESHOLD FROM FALSE-FLAG BUDGET (R = 512 ranks, evaluated every 1 min)
   ═══════════════════════════════════════════════════════════════════════════
     want ≤ 1 false flag per DAY  ⇒  per evaluation: 1/1440 = 6.9e-4
     per rank:  6.9e-4 / 512 = 1.36e-6  (one-sided)
     normal quantile:  z ≈ 4.7

     ⇒ flag when robust-z > 4.7, i.e. step_time > median + 4.7 × 1.4826 × MAD

   Convert to a ratio for intuition: if MAD/median = 2 % (a well-behaved
   synchronous job), the threshold is  1 + 4.7 × 1.4826 × 0.02  ≈ 1.14×.
   If MAD/median = 8 % (a job with ragged data loading), it is ≈ 1.56×.

   ⇒ THIS is what the folk-wisdom "1.3×" is approximating: a fleet whose
     step-time dispersion happens to be ~4–5 %. Measure YOUR dispersion and
     the constant falls out. Using 1.3 on a tight job misses real stragglers;
     using it on a ragged job pages every minute.
```

**Step 3 — require persistence, and require impact.** A single slow step is a GC pause or a checkpoint. Require the condition to hold for `N` consecutive evaluations (lesson 07's `for:` clause), and gate the page on the job's goodput actually being down — a straggler that does not move goodput is a curiosity, not an incident.

The PromQL, written against the bounded L3 series set from §2:

```yaml
groups:
  - name: gpu-straggler
    interval: 30s
    rules:
      # Robust centre and dispersion, per job. Both are cheap: they aggregate
      # the 10k per-rank series down to 2 series per job.
      - record: job:step_time:median
        expr: quantile by (job) (0.5, step_time_seconds)
      - record: job:step_time:mad
        expr: |
          quantile by (job) (0.5,
            abs(step_time_seconds - on (job) group_left job:step_time:median)
          )
      # Robust z per rank. THIS series is per-rank and is the one worth its
      # cardinality — everything else about the rank stays an exemplar.
      - record: rank:step_time:robust_z
        expr: |
          0.6745 * (step_time_seconds - on (job) group_left job:step_time:median)
          / on (job) group_left clamp_min(job:step_time:mad, 0.001)

      - alert: TrainingStragglerRank
        expr: |
          rank:step_time:robust_z > 4.7
          and on (job) job:goodput_ratio < 0.9
        for: 5m
        labels: { severity: ticket, alert_type: straggler }
        annotations:
          summary: "rank {{ $labels.rank }} of {{ $labels.job }} is a straggler (robust z={{ $value | humanize }})"
          next_step: >-
            Pull this rank's flight-recorder tail and its host off-CPU profile.
            If it is NOT waiting in the collective while its peers are, it is
            the cause; if it IS waiting, look one hop further.
```

Note the last annotation: it encodes lesson 08's inversion. **In a barrier-bound system the victims accumulate wait and the culprit does not**, and any straggler runbook that omits that sentence sends people to the wrong node.

### 7. What single-node dashboards structurally cannot see

Worth stating as a general principle rather than a GPU fact, because it is what makes this lesson transferable: **a synchronous, barrier-bound distributed system has a class of faults that are invisible in every per-node signal, because the fault manifests as an absence of progress that every node shares equally.**

The same shape appears in a MapReduce shuffle waiting on one slow reducer, in a consensus group waiting on a lagging replica, and in a synchronous SGD step waiting on one slow rank. The instrumentation differs (RPC timing, replica lag, CUDA events on collectives); the diagnostic structure is identical:

1. Measure the **per-participant** completion time of the barrier-bound operation.
2. Compare participants against each other, not against an absolute threshold (§6).
3. Distinguish *waiting* from *working* — the culprit is the one that is not waiting (§5, §6).
4. Then descend a layer to find out why that participant is slow.

Interconnect saturation is the other half, and it is a plain USE-method application to a resource people forget is a resource: for NVLink, PCIe and IB/RoCE, track utilisation against line rate, saturation (queueing/retries), and errors (CRC, replay, recovery counters — module 05's field list). A link that is *erroring and retrying* rather than *saturated* looks identical in a bandwidth gauge and completely different in the error counters, and the fix is a cable, not a scheduler change.

### 8. The serving layer, and why fleet scale needs vendor-neutral names

Module 05 lesson 06 owns the inference SLO material — the TTFT/ITL/TPOT distinctions, the continuous-batching mechanics, the KV-cache ceiling, and the current vLLM V1 metric names, including the renames that break stale dashboards (`vllm:inter_token_latency_seconds` and `vllm:request_time_per_output_token_seconds` replacing the old single `time_per_output_token` metric, and `vllm:kv_cache_usage_perc` replacing `gpu_cache_usage_perc`). None of that is repeated here.

**What is new at fleet scale is that you will not have one engine.** A real platform runs vLLM for most models, TGI for a legacy service, SGLang for a research team, and a managed API for something. Their metric names differ, their histogram bucket boundaries differ, and a fleet-level SLO cannot be defined across them without a normalisation layer.

Two ways to build it, and the second is now the standard:

- **Recording rules per engine** mapping engine-specific names into one `slo:ttft_seconds_bucket` family. Cheap, fully under your control, and one more thing to maintain per engine version.
- **Emit the OpenTelemetry GenAI conventions.** The semantic conventions define `gen_ai.server.time_to_first_token`, `gen_ai.server.time_per_output_token`, `gen_ai.server.request.duration`, and client-side `gen_ai.client.token.usage` / `gen_ai.client.operation.duration` / `gen_ai.client.operation.time_to_first_chunk`, with attributes `gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.token.type`.

The conventions also do something that matters directly for lesson 07's SLO math: they **specify explicit histogram bucket boundaries**. `gen_ai.server.time_to_first_token` is specified with `[0.001, 0.005, 0.01, 0.02, 0.04, 0.06, 0.08, 0.1, 0.25, 0.5, 0.75, 1.0, 2.5, 5.0, 7.5, 10.0]`. That means 100 ms, 250 ms, 500 ms and 1 s are *exact* SLO thresholds — `…_bucket{le="0.5"}` is a count, not an interpolation — and 300 ms is not. **Choose your SLO thresholds from that list, or change the buckets; do not interpolate.**

The fleet cardinality question for L5 is then straightforward: `model × engine × route × le` per histogram. With 40 models, 3 engines, and a 16-bucket histogram that is `40 × 3 × 16 = 1,920` series per metric per route — comfortable. What is *not* comfortable, and what people add without thinking, is `gen_ai.request.model` **plus** a per-tenant or per-API-key attribute; that is the L1 rule again, and the answer is again the exemplar.

### 9. The model-quality layer, and why it breaks burn-rate alerting

Everything below L6 is infrastructure: it tells you whether the machine is working. L6 asks whether the *system is doing its job*, and it behaves differently in three ways that change the alerting design.

**(a) The signal arrives late, or not at all.** Ground truth for "was this answer good" comes from a human, a downstream conversion event, or an offline eval — hours to days after the request. Lesson 07's burn-rate machinery assumes you can compute a bad-event ratio over a five-minute window; here you cannot. So:

| Signal class | Latency | Alertable how |
|---|---|---|
| Input distribution drift (prompt length, language, topic mix) | seconds | thresholds/burn-rate on the *proxy*, since it is available immediately |
| Output-shape signals (refusal rate, truncation rate, empty responses, tool-call failure rate) | seconds | **this is your fast proxy for quality** — treat as a normal SLI |
| Guardrail/safety classifier fires | seconds | rate-based alert, absolute threshold |
| Scheduled offline eval on a fixed suite | hours (per run) | alert on *step change between runs*, not on a rolling window |
| Human feedback / thumbs-down rate | hours–days | trend review, never a page |
| Business outcome (conversion, escalation, resolution) | days | weekly review |

**(b) The comparison is against a reference, not a threshold.** "The p95 output length rose from 180 to 340 tokens" is meaningless without knowing what it was; drift signals are inherently two-population comparisons — exactly like lesson 08's profile diffs, with the same statistical structure. Population Stability Index or a simple binned KL divergence against a pinned reference window is the usual mechanism, and the operationally important part is that **the reference window must be pinned and versioned**, or your drift detector silently re-baselines onto the drift.

**(c) The cause is usually a version change, so version must be a label.** `model_version`, `prompt_template_version`, `retrieval_index_version` and `guardrail_version` are the dimensions every L6 investigation slices on. They are low-cardinality and stable — they belong on the series, and the fleet-scale mistake is not putting them there until after the first incident.

**The design rule that ties L6 back to the rest of the stack:** page on the fast proxies (refusal rate, truncation rate, tool-call failure, empty-response rate), because they are available in seconds and correlate with real quality regressions; ticket on drift; review evals on a schedule. A team that tries to page on model quality directly will either page on noise or discover their signal is three days stale during the incident.

### 10. The join-key spine

Six layers are only useful if a single incident can be followed across them, and that requires one identity chain that every layer stamps. This is the design decision that is cheapest to make early and most expensive to retrofit.

```
   THE JOIN-KEY SPINE — ONE IDENTITY CHAIN, SIX LAYERS
   ═══════════════════════════════════════════════════════════════════════════

     L6  model quality      model_version ─┐
     L5  serving            model_version ─┤ ← same key, both layers
                            + request_id ──┼──────────────► exemplar/trace only
     L4  job / scheduler    job_id ────────┤  (K8s: namespace/pod ‖ Slurm: JobID)
     L3  framework / step   job_id + rank ─┤
     L2  collectives        job_id + rank + comm_id + seq_id
     L1  device             node + gpu  (+ gpu_uuid, in an INFO metric only)
     L0  host               node + pid

   WHERE EACH KEY IS ALLOWED TO LIVE
   ┌──────────────┬───────────────┬──────────────────────────────────────────┐
   │ key          │ distinct vals │ home                                     │
   ├──────────────┼───────────────┼──────────────────────────────────────────┤
   │ node         │ 4,000         │ metric label                             │
   │ gpu (0–7)    │ 8             │ metric label                             │
   │ tenant       │ ~200          │ metric label — the chargeback dimension  │
   │ gpu_family   │ ~5            │ metric label (collapsed from modelName)  │
   │ job_id       │ churny        │ metric label ONLY with admission control;│
   │              │               │ otherwise info metric + lakehouse column │
   │ rank         │ ≤ ranks/job   │ metric label at L3 only (step time)      │
   │ gpu_uuid     │ 32,000        │ INFO metric (join at query time) + lake   │
   │ pod / pod_id │ unbounded     │ info metric + lakehouse column           │
   │ comm_id/seq  │ unbounded     │ flight recorder + log field + exemplar   │
   │ request_id   │ unbounded     │ exemplar → trace                         │
   │ model_version│ ~50           │ metric label (L5, L6) — cheap and vital  │
   └──────────────┴───────────────┴──────────────────────────────────────────┘

   ⇒ THE TEST: from a firing goodput alert, can you get to a rank, from that
     rank to a node and GPU, and from that GPU to a host profile — using only
     keys that exist in the data? If any hop needs a key you dropped for
     cardinality, you have to add it back as an exemplar or an info metric.
     Do that on paper BEFORE the incident.
```

**The enrichment mechanism, and its misattribution window.** The join is performed at collection time by an enrichment step: on Kubernetes, dcgm-exporter's device-plugin/pod-resources mapping (module 05 lesson 04); on Slurm, an OpenTelemetry Collector processor. Meta's open-sourced **GCM** is the clearest public example of the latter — three components, of which the `slurmprocessor` is "an OpenTelemetry processor for enriching telemetry data with Slurm metadata," identifying the Slurm jobs associated with specific GPUs and stamping job IDs, user names and partitions onto metrics. GCM's README states it powers Meta FAIR's AI workloads "across over hundreds of thousands of GPUs."

Its configuration carries the operationally important detail, and it generalises to every scheduler join:

```yaml
processors:
  slurm:
    # Seconds to cache Slurm lookups in memory. The README's own warning:
    # this "affects the number of misattributed Slurm metadata at the
    # beginning and end of the job lifetime" — and low values "could
    # overwhelm slurmctld".
    cache_duration: 60
    cache_filepath: '/tmp/slurmprocessor_cache.json'
    query_slurmctld: false
```

**Read that trade-off carefully, because every attribution system has it.** A cached scheduler lookup means that for up to `cache_duration` seconds after a job starts or ends, GPU metrics are stamped with the *previous* owner. At a 60-second cache and 30-second scrape, that is up to two scrapes per job boundary attributed to the wrong tenant. For dashboards, irrelevant. For **chargeback**, it is a systematic bias proportional to job churn — short jobs are affected disproportionately — and it is the kind of error that surfaces in a billing dispute rather than in a dashboard review. Bound it, measure it (`unattributable_gpu_hours` as an explicit series), and state the bound in the chargeback methodology. Lesson 10 is where that becomes a reconcilable ledger rather than a caveat.

The transferable lesson from GCM is not "use Slurm": it is **join hardware telemetry to whatever the scheduler treats as the unit of job identity** — a Slurm JobID, a Kubernetes pod UID, a Ray job ID — and make that key the spine of every layer above it.

### 11. Alerting the fleet: rollups, absence, and the meta-alerts

Lesson 07 derived the burn-rate machinery and §12 there applied it to a wasted-GPU-hours budget; that is not repeated. What is fleet-specific is *what else* must exist alongside it.

```yaml
groups:
  - name: gpu-fleet-rollups
    interval: 30s          # ← the GPU-hours integration constant is 30/3600
    rules:
      # Fleet rollups so dashboards never fan out over 1.4 M raw series (L1 §2,
      # L3 §12): compute once in the ruler, read everywhere.
      - record: fleet:sm_active:avg_by_tenant
        expr: avg by (tenant) (DCGM_FI_PROF_SM_ACTIVE)
      - record: fleet:sm_active:p01_by_tenant      # the TAIL is the story:
        expr: quantile by (tenant) (0.01, DCGM_FI_PROF_SM_ACTIVE)
      # XID as a ROLLUP, never per-event (§3).
      - record: fleet:xid_errors:rate5m_by_tenant
        expr: sum by (tenant, xid) (rate(DCGM_FI_DEV_XID_ERRORS[5m]))

      # ── The absence detectors. A hang shows up as MISSING data (§4), and
      #    missing data is what every naive dashboard renders as "fine".
      - alert: TrainingStepHeartbeatStalled
        expr: |
          (time() - max by (job) (timestamp(training_steps_total))) > 90
        for: 1m
        labels: { severity: page, alert_type: training-liveness }
        annotations:
          summary: "{{ $labels.job }} has not completed a step in >90s"
          why: >-
            Detects a collective hang ~7x faster than PyTorch's 10-minute
            default collective timeout. Pull flight-recorder tails next.

      - alert: DcgmExporterMissing
        expr: |
          count by (node) (up{job="dcgm"} == 1) unless on (node) (kube_node_info)
        for: 10m
        labels: { severity: ticket, alert_type: meta }

      # ── Meta: the attribution join itself failing. Without this, a broken
      #    join looks exactly like "the fleet is idle" (§10).
      - alert: GpuAttributionJoinDegraded
        expr: |
          ( count(DCGM_FI_PROF_SM_ACTIVE unless on (node, gpu) gpu_job_info) )
          / count(DCGM_FI_PROF_SM_ACTIVE) > 0.05
        for: 15m
        labels: { severity: ticket, alert_type: meta }
        annotations:
          summary: ">5% of GPUs have no job attribution — chargeback is wrong"

      # ── Correlate hardware events with impact rather than paging per event.
      - alert: XidBurstWithGoodputImpact
        expr: |
          sum by (tenant) (increase(DCGM_FI_DEV_XID_ERRORS[30m])) > 5
          and on (tenant) (avg by (tenant) (job:goodput_ratio) < 0.9)
        for: 10m
        labels: { severity: page, alert_type: hardware-impact }
```

Three patterns there are the fleet-specific content, and each is a direct consequence of an earlier section:

- **Absence beats value** (§4). The highest-value alert in a training fleet is a heartbeat staleness check, not a threshold on any gauge.
- **Rollups, not per-event** (§3). At one hardware interruption every few hours, per-XID paging is unusable; XID matters when it is *correlated with goodput impact*.
- **Meta-alerts on the observability system itself** (§10). A failed attribution join, a missing exporter, and a skipped rule group all produce the same symptom — a dashboard that looks calm — and each needs its own detector.

### 12. Where to invest, in order

The arithmetic of §3 and §4 gives a defensible priority order, which is what a staff engineer is actually being asked for:

| Priority | Investment | Why, from the derivations |
|---|---|---|
| 1 | **Step heartbeat + absence alerting** | Cuts detection of a hang from 10 min (framework default) to ~90 s. Four lines of code against a 3.6-minute budget (§3, §4). |
| 2 | **Checkpoint interval review** | E[recompute] = interval/2 dominates the loss budget; halving the interval buys more than any monitoring change (§3). |
| 3 | **Flight recorder sized to the timeout** | Turns a hang from "restart and hope" into a named culprit rank; costs host RAM (§5). |
| 4 | **Per-rank step time + robust straggler detection** | The only detector for fail-slow, which is the unsolved half (§3, §6). |
| 5 | **Goodput SLO + burn-rate alert** | Converts everything above into a page with a cost attached (lesson 07 §12). |
| 6 | **Attribution join + meta-alerts on it** | Without it none of the above can be routed to an owner, and chargeback is wrong (§10). |
| 7 | **Collective-level profiling (NCCL plugin)** | Highest resolution, highest cost, least mature — pilot on one job, not fleet-wide (§5). |

## Perspectives

**Reliability engineering.** At 16k GPUs, hardware failure is a rate, not an event, and the observability objective is a *time budget* rather than a coverage checklist. The derivation that matters is `L ≤ (1−E) × 24 / I_per_day` and its immediate corollary that the checkpoint interval consumes most of `L`. This reframes arguments that otherwise go in circles: "we need better monitoring" and "we need faster checkpointing" are competing claims on one budget, and the budget is computable.

**Distributed systems.** Nothing about straggler detection is GPU-specific. Per-participant timing of a barrier-bound operation, robust comparison across participants, and the waiter/culprit inversion transfer directly to shuffles, consensus groups and any fan-out RPC. The GPU case is distinctive only in cost per unit time, which is what makes it worth instrumenting at this depth.

**Statistics.** Both fleet detectors in this lesson are comparisons within a population, not thresholds: robust z against the job's own median (§6), and drift against a pinned reference window (§9). Both need a multiple-comparisons correction because they are evaluated over hundreds of participants continuously, and both fail the same way when someone picks a round-number threshold instead of deriving it from the dispersion.

**Economics.** Every number in this lesson converts to dollars through one identity: a synchronous job's cost is `ranks × wall-time × rate`, and a fault at any rank multiplies by `ranks`. A ten-minute hang on 512 A100s at $2.50/GPU-hour is ≈$219; at one hang per few hours across a fleet, framework timeout defaults alone are a five-figure monthly line item. That is the argument that funds the step heartbeat.

**Tooling maturity.** NCCL's profiler plugin interface is at v7 and still evolving (v6 added copy-engine events, v7 added kernel-phase sub-events); NIXT/NCCL Inspector is a 2026-vintage effort with published validation around 2,048 GPUs; GCM was open-sourced recently. Fleet-scale comms observability is actively being built in public. Saying that plainly — and pairing it with a pilot plan rather than an assumption — is the credible position, not a gap in your knowledge.

## Real-world use cases

- **Meta — Llama 3 infrastructure report** (arXiv:2407.21783). 466 interruptions (419 unexpected) over 54 days on 16,384 H100s, hardware-dominated causes, >90% effective training time, 3 manual interventions. **What it shows:** at fleet scale reliability engineering *is* GPU observability, and the achievable target is a detection-to-recovery budget rather than a failure-free run. *(Blocked by the egress proxy here; figures carried forward from this course's earlier verified pass and used as stated inputs to §3's derivation, which is reproducible with any fleet's own numbers.)*

- **Meta — GPU Cluster Monitoring (GCM)**, `github.com/facebookresearch/gcm`. Three components: Slurm-based monitoring collectors, lifecycle health checks, and a `slurmprocessor` OpenTelemetry processor that enriches telemetry with Slurm job metadata; the README states it powers Meta FAIR workloads across hundreds of thousands of GPUs. **What it shows:** the attribution *pattern* — join hardware telemetry to the scheduler's unit of job identity — plus the concrete trade its config exposes: `cache_duration: 60` bounds load on `slurmctld` and simultaneously creates a misattribution window at job boundaries. That trade exists in every attribution system, including the Kubernetes one, and it is the difference between a dashboard and a defensible bill.

- **PyTorch's Flight Recorder and NCCL watchdog defaults**, `pytorch/pytorch` (`FlightRecorder.hpp`, `ProcessGroupNCCL.{hpp,cpp}`). A 2000-entry per-rank ring buffer of collective records with `scheduled/started/completed` states and sequence numbers; a 10-minute default collective timeout; an 8-minute heartbeat monitor; dump-on-timeout enabled by default with a 15-second dump budget. **What it shows:** the framework already collects the evidence you need to name a culprit rank — and simultaneously sets a detection floor an order of magnitude above the budget §3 derives. Both facts are in the same file, and most teams know neither.

- **NCCL's profiler plugin API**, `NVIDIA/nccl` (`src/include/plugin/nccl_profiler.h`, `src/plugin/profiler.cc`). A versioned (v7) plugin interface loaded via `NCCL_PROFILER_PLUGIN`, with a bitmask of event classes from whole collectives down to proxy steps and kernel phases, and named proxy-step states (`SendGPUWait`, `SendWait`, `RecvWait`, `RecvFlushWait`, `RecvGPUWait`). **What it shows:** comms observability is a supported extension point rather than a research hack, and its event mask is an explicit cost knob — the design lets you run cheap fleet-wide and expensive on one job.

- **NVIDIA NCCL Inspector / NIXT** (developer forum post; arXiv:2608.01449). CUDA-event instrumentation of NCCL calls exporting Prometheus-shaped per-collective metrics, reported under 2% overhead, with published validation around 2,048 GPUs. **What it shows:** the fail-slow gap is being closed with productised tooling — and that published validation below your fleet size is an engineering risk to pilot, not a solved dependency. *(Both hosts blocked here; not re-fetched — carried forward from this course's earlier pass.)*

- **OpenTelemetry GenAI semantic conventions**, `open-telemetry/semantic-conventions-genai`. Server- and client-side metric definitions with explicit histogram bucket boundaries for TTFT and per-output-token latency. **What it shows:** the serving layer is standardising, which is what makes a cross-engine fleet SLO possible at all — and the specified boundaries determine which SLO thresholds are exact rather than interpolated.

## Worked example

**Scenario.** A 512-GPU pretraining run (`job=pretrain-7b`, 64 nodes × 8 H100) on a shared 4,000-node fleet. At 04:12 the goodput burn-rate alert from lesson 07 fires at 6×. Every GPU dashboard is green. Walk the stack.

### Step 0 — what the alert actually said

```
   GoodputBudgetBurnSlow  fired 04:12
     job:goodput_ratio    = 0.71   (achieved / expected tokens per second)
     wasted GPU-h per hour = 512 × 0.29                       = 148 GPU-h/h
     against the sustainable 15.4 GPU-h/h (lesson 07 §12)     = 9.6× burn
     ⇒ crossed the 6× slow tier, below the 14.4× fast tier.
       Cost while it runs: 148 × $2.50                        = $370/hour
```

### Step 1 — L1 says nothing, and that is information

```
   avg by (job) (DCGM_FI_PROF_SM_ACTIVE)                   = 0.93   ← "healthy"
   quantile by (job) (0.01, DCGM_FI_PROF_SM_ACTIVE)        = 0.91   ← no outlier
   max by (job) (DCGM_FI_DEV_GPU_UTIL)                     = 100    ← meaningless
   sum by (job) (increase(DCGM_FI_DEV_XID_ERRORS[1h]))     = 0

   ⇒ Every device is busy. GPU_UTIL is a presence metric (module 05) and
     SM_ACTIVE is occupancy, not productivity — a collective kernel spinning
     while it waits occupies SMs. High L1 signals are CONSISTENT with a
     job that is entirely blocked. This is the util-lie at fleet scale, and
     the correct read is "L1 has excluded a dead or throttled device, and
     nothing more."
```

### Step 2 — L3 finds the shape: not a hang, a straggler

```
   job:step_time:median{job="pretrain-7b"}                  = 0.412 s
   job:step_time:mad{job="pretrain-7b"}                     = 0.009 s
   ⇒ MAD/median = 2.2 %  (a tight job; the folk 1.3× threshold would be
     1 + 4.7 × 1.4826 × 0.022 ≈ 1.15× — so 1.3× would have MISSED this)

   topk(3, rank:step_time:robust_z{job="pretrain-7b"}):
     rank=143   z = 27.4    step_time = 0.581 s
     rank=87    z =  1.1
     rank=402   z =  0.9

   Sanity: with R=512 and z>4.7 the expected false-flag rate is ~1/day.
   One rank at z=27 is not a false flag.

   Impact check: the whole job runs at the slowest rank, so
     expected step time ≈ 0.412 s, actual ≈ 0.581 s
     ratio 0.412/0.581 = 0.709  →  matches the observed goodput_ratio 0.71 ✓
   ⇒ ONE rank in 512 explains ALL of the goodput loss. That reconciliation
     is what turns a hypothesis into a diagnosis; do it every time.
```

### Step 3 — L2 says which side of the collective rank 143 is on

Rank 143's flight-recorder tail (pulled live, not from a timeout dump — no timeout has occurred; the job is slow, not hung):

```
   rank 143, pg 0, last entries:
     seq 41,882  nccl:all_reduce  completed  duration_ms = 118.4
     seq 41,883  nccl:all_reduce  completed  duration_ms = 121.9
     seq 41,884  nccl:all_reduce  started
   a healthy peer, rank 144:
     seq 41,882  nccl:all_reduce  completed  duration_ms = 289.7
     seq 41,883  nccl:all_reduce  completed  duration_ms = 291.1
     seq 41,884  nccl:all_reduce  started

   READ THE INVERSION: rank 143 spends ~120 ms inside the collective;
   its peers spend ~290 ms. THE FAST ONE IS THE CULPRIT — 143 arrives late
   and leaves quickly; everyone else arrives early and waits for it.
   ⇒ rank 143 is losing time BEFORE the collective — in compute, in the
     dataloader, or on the host. Not in the fabric.
```

### Step 4 — L0 names the cause

Query the fleet profile store (lesson 08, shape ②): rank 143's node against its 63 peers, same 30-minute window.

```
   frame                                   node-0417   pool (63 nodes)    Δ
   ─────────────────────────────────────────────────────────────────────────
   __memmove_avx_unaligned_erms              14.9 %        1.2 %       +13.7
     ← under: PIL.Image.resize ← dataloader worker
   torch::autograd::Engine::execute           7.9 %        8.1 %        −0.2
   ncclKernel_AllReduce                      11.1 %       26.4 %       −15.3
     ← consistent with Step 3: this node spends LESS time in the collective

   N_A ≈ 1.09e6 samples, N_B ≈ 6.9e7 → SE on the 14.9 % frame ≈ 0.23 %.
   The +13.7 pp delta is ~60σ. Real.

   Descend once more: node-0417's dataloader is decoding on the CPU, not
   using the GPU decode path its peers use. Check the pod spec →
   the node is missing the nvidia-dali runtime label after a reimage, so
   the job fell back to the PIL path on this node only.
```

### Step 5 — cost, and what each layer contributed

```
   TIME AND MONEY
     detected by:            goodput burn-rate alert (L3/L5 aggregate)  04:12
     shape identified:       robust-z straggler query (L3)              04:15
     side of collective:     flight recorder comparison (L2)            04:22
     root cause:             fleet profile diff (L0)                    04:29
     ⇒ 17 minutes, no SSH, no reproduction.

     Loss while it ran (fault began ~02:50 at the reimage):
       99 min × 148 GPU-h/h ÷ 60 × $2.50                    ≈ $610
     Had only a 14.4× fast tier existed, 9.6× never pages:
       at $370/hour, one week undetected                    ≈ $62,000

   WHAT EACH LAYER CONTRIBUTED — AND WHAT IT COULD NOT
     L1 device   excluded thermal/ECC/XID; could not see the problem at all
     L3 step     found the shape (one rank) and reconciled it to goodput
     L2 collect. found the SIDE (before vs during the collective)
     L0 host     found the cause (CPU decode path)
     ⇒ No single layer solves this. The correlation is the capability,
       and the join keys of §10 are what made each hop a query rather
       than an investigation.
```

## Practice

Feeds the module deliverable directly: **[fleet observability design](../practice/fleet-observability/README.md)**. This lesson is the source of its **DCGM + NCCL signal plan** and its **GPU goodput SLO + straggler alert**.

1. **Draw the signal stack for your fleet.** Six layers; for each, list the signals you actually collect, the question it answers, the question it cannot answer, its detection latency, and its series count. Mark the layers where you currently have nothing — that gap list is the deliverable's most valuable page.
2. **Do the cardinality arithmetic per layer** for 4,000 nodes / 32,000 GPUs, including L2 and L3 (which do not exist in module 05's single-node audit). Show the naive number and the designed number, and name what moved to exemplars, info metrics, or the lakehouse.
3. **Derive your detection budget.** Pick an interruption rate (yours, or Llama 3's as a stated stand-in) and an effective-training-time target; compute the minutes-per-interruption budget; subtract `checkpoint_interval / 2`; state what is left for detect + restart. If the remainder is negative, say so and recommend the checkpoint change rather than a monitoring change.
4. **Audit your framework defaults against that budget.** Record the actual values of `kProcessGroupNCCLDefaultTimeout`, `TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC` and `TORCH_NCCL_ASYNC_ERROR_HANDLING` in your stack, and write the step-heartbeat instrumentation plus its absence alert with the threshold you chose and why.
5. **Size the flight recorder.** Measure or estimate collectives-per-step and steps-per-second, compute the history depth at the default 2000 entries, and set `TORCH_FR_BUFFER_SIZE` to cover your collective timeout plus margin. State the host-RAM cost per rank.
6. **Derive your straggler threshold.** Measure MAD/median on a healthy run, choose a false-flag budget (e.g. ≤1/day at your rank count and evaluation interval), compute the robust-z threshold and the equivalent ratio-to-median, and compare it to the folk 1.3×. Write the recording rules and the alert, gated on goodput impact.
7. **Write the cross-engine serving normalisation.** Either recording rules mapping each engine's names into one family, or the OTel GenAI convention names — and pick SLO thresholds that land exactly on the specified bucket boundaries. State which engines you have and which metric names each currently exposes (verify by curling `/metrics`, do not trust a tutorial).
8. **Specify the L6 layer.** List your fast quality proxies (refusal, truncation, empty response, tool-call failure), your drift signals with a pinned reference window, and your scheduled evals — and state explicitly which of these may page, which ticket, and which are review-only, with the label-latency justification.
9. **Draw the join-key spine and test it.** For a firing goodput alert, write the exact sequence of queries that takes you from alert → rank → node/GPU → host profile, using only keys you actually keep. Any hop that fails is a design change, not a runbook note.
10. **Write the meta-alerts.** Attribution-join degradation, exporter absence, and rule-group misses — with the argument for why each failure looks like "the fleet is fine."
11. **Pilot plan for comms profiling.** Where NCCL Inspector / a `NCCL_PROFILER_PLUGIN` would sit, which event-mask bits you would enable fleet-wide versus per-investigation, the overhead you would measure before and after, and what you would validate above 2,048 GPUs.

**Acceptance criteria:** a per-layer series budget with numbers; a detection budget derived rather than asserted; a straggler threshold derived from measured dispersion; a flight-recorder size justified by collective rate; a join-key table where every unbounded key has a named home outside metric labels; and at least three meta-alerts on the observability system itself.

## Common pitfalls

- **"High fleet-average SM_ACTIVE means the fleet is healthy."** A synchronous job's throughput is gated by its slowest rank, and a collective kernel spinning while it waits keeps SMs occupied. Symptom: 93% average occupancy with 71% goodput. Mechanism: occupancy measures residency, not progress — pair every average with a tail query and a goodput reconciliation.
- **Alerting on a value when the failure is an absence.** A hung job stops emitting step metrics; most dashboards render the last value forever, and most alerts on `x > threshold` never fire on missing data. Symptom: a 10-minute hang discovered by the framework timeout rather than by monitoring. Fix: heartbeat counters plus `time() - timestamp(...)` staleness alerts.
- **Leaving the framework's default timeouts unexamined.** A 10-minute collective timeout and an 8-minute heartbeat monitor are the floor on how fast anything downstream can react. Symptom: recovery automation that is fast and irrelevant, because the 10 minutes happen first.
- **Flight recorder enabled but too small to contain the evidence.** 2000 entries at 32 collectives/second is 62 seconds of history against a 600-second timeout. Symptom: dumps full of identical `started` entries with no sign of the transition that mattered.
- **Reading the straggler backwards.** In a barrier-bound collective the victims accumulate wait and the culprit does not. Symptom: investigations that keep landing on healthy nodes. Fix: compare per-rank *time inside* the collective and suspect the *shortest*.
- **A magic 1.3× straggler threshold.** It approximates a robust-z of ~4.7 only when step-time dispersion happens to be ~4–5%. Symptom: on a tight job (MAD/median 2%) it misses real 15% stragglers; on a ragged job (8%) it pages continuously. Fix: derive from measured MAD and a false-flag budget that accounts for testing hundreds of ranks at once.
- **Per-XID alerting.** At fleet scale hardware events are a continuous stream. Symptom: a channel nobody reads. Fix: rollup by tenant and XID, and page only when correlated with goodput impact.
- **Assuming the attribution join is exact.** A cached scheduler lookup misattributes GPU-seconds at every job boundary, biased toward short jobs. Symptom: chargeback numbers that do not reconcile and cannot be defended in a dispute. Fix: bound it, publish an explicit `unattributable_gpu_hours` series, and move the authoritative accounting to the analytics path (lesson 10).
- **Trying to page on model quality.** Ground truth arrives hours to days late. Symptom: either a noisy detector on a proxy nobody validated, or an alert whose data is three days stale during the incident. Fix: page on fast output-shape proxies, ticket on drift against a pinned reference, review evals on a schedule.
- **Deploying comms profiling fleet-wide on day one.** The event mask exists because full-detail proxy-step and kernel-channel events are not free, and published validation for the productised tooling sits around 2,048 GPUs. Symptom: an instrumentation change that becomes the goodput regression it was meant to find.

## Self-check

- **Your fleet sees one unexpected interruption every three hours and you need ≥90% effective training time. Derive what that demands, and say what it implies about checkpointing.** — `I = 8/day`; lost fraction `= I × L/24 ≤ 0.10` ⇒ `L ≤ 0.10 × 24 / 8 = 0.30 h = 18 minutes` per interruption for detect + decide + restart + re-warm + recompute. Expected recomputation alone is `checkpoint_interval / 2`, so a 30-minute checkpoint interval consumes 15 of the 18 minutes and leaves 3 for everything else; a 60-minute interval makes the target unreachable no matter how good the monitoring is. Observability latency and checkpoint frequency are two ends of one budget, and the arithmetic tells you which to spend on first.

- **A 512-rank job stops making progress. What does each layer show in the first ten minutes on stock PyTorch defaults, and what should you have built instead?** — L4 shows a Running pod with a passing liveness probe; L1 shows ~100% `GPU_UTIL` and high `SM_ACTIVE` because the collective kernel is resident and spinning; L3 shows *absence* — step metrics simply stop; nothing pages. At ~8 minutes `TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC` may fire if the watchdog itself is stuck; at 10 minutes `kProcessGroupNCCLDefaultTimeout` expires, the flight recorder dumps (default on, 15 s budget), and the process tears down under `ASYNC_ERROR_HANDLING=3`. Only then does the scheduler see a dead pod. The fix is a step-heartbeat counter plus a `time() - timestamp(...) > 90s` absence alert — roughly 7× faster detection, four lines of code, and independent of every framework default.

- **Two ranks' flight-recorder tails show them on different `collective_seq_id`s; in another job all ranks are on the same seq but one shows a much *shorter* time inside the collective. What are these two situations?** — Different seq ids (or different `profiling_name`s) is a **desync**: ranks are executing different collectives, usually because model control flow diverged across ranks — a code bug, not a hardware fault. Same seq id with one rank spending much less time inside the collective is a **straggler**: that rank arrives late and leaves quickly while its peers block waiting for it, so the short-duration rank is the culprit and the long-duration ranks are victims. The first is fixed in the training script; the second is fixed on the culprit's host or device.

- **You set your straggler threshold at 1.3× the job median. Under what conditions does that miss real stragglers, and what should it be instead?** — It corresponds to a robust-z of ~4.7 only when `MAD/median ≈ 4–5%`. On a tight synchronous job with `MAD/median = 2%`, the equivalent threshold is about **1.15×**, so a rank running 20% slow — enough to cost 20% of the whole job's goodput — never trips 1.3×. On a ragged job at 8% it corresponds to ~1.56×, so 1.3× pages constantly. Derive instead: measure MAD on a healthy run, pick a false-flag budget (with `R` ranks × evaluations/day tests, so `z ≈ 4.7` for ~1/day at R=512 and minute-resolution), and set `threshold = median + z × 1.4826 × MAD`. Then require persistence and gate on goodput impact.

- **Why does your flight-recorder buffer need sizing, and what do you set it from?** — It stores a fixed number of *entries* (default 2000), not a fixed span of time, so its history depth is `size / (collectives_per_step × steps_per_second)`. At 16 collectives/step and 2 steps/s that is 32 entries/s, so the default holds only ~62 seconds — far less than a 10-minute collective timeout, meaning a timeout dump contains nothing but recent `started` entries and no sign of the transition that caused the hang. Set `TORCH_FR_BUFFER_SIZE ≥ collective_rate × (timeout + margin)`; for this example, ~21,000 entries. Enable `TORCH_FR_CPP_STACK` only during investigation — per-entry stack capture multiplies both memory and overhead.

- **Which keys may be metric labels on a 32,000-GPU fleet, and where do the rest live?** — Bounded and stable keys are labels: `node` (4,000), `gpu` (0–7), `tenant` (~200), `gpu_family` (~5), `model_version` (~50), and `rank` on the single L3 step-time series. `gpu_uuid` (32,000) goes into an info metric to join at query time; `pod`/`container_id` and `job_id` are churny and belong in an info metric and the analytics path; `comm_id`, `seq_id` and `request_id` are unbounded and belong in the flight recorder, log fields and exemplars. The test is whether you can still walk alert → rank → node/GPU → host profile using only what you kept; every hop that breaks must be restored as an exemplar or an info metric, decided on paper before the incident.

- **Why can't the model-quality layer use the burn-rate alerting from lesson 07, and what do you do instead?** — Burn-rate alerting needs a bad-event ratio computable over a short rolling window; the ground truth for quality (human feedback, downstream outcomes, offline evals) arrives hours to days later, so the numerator does not exist when the alert would need to evaluate. Instead, page on fast output-shape proxies available in seconds — refusal rate, truncation rate, empty responses, tool-call failure — treated as ordinary SLIs; ticket on distribution drift measured against a *pinned, versioned* reference window (an unpinned reference silently re-baselines onto the drift); and review scheduled evals as step changes between runs rather than as a rolling window. Keep `model_version`, `prompt_template_version` and index/guardrail versions as labels, because every L6 investigation slices on them.

## Connections & what's next

This closes the module's through-line on the **hot path**: **cardinality is the master constraint, and delivered work (goodput) is the master SLI.** L1's signal-fit matrix, L2's PromQL traps, L3's fall-over sequence and sizing, L4's collector-side enrichment, L5's exemplars, L6's log discipline, L7's burn-rate derivation and L8's fleet profiling all appear in this lesson's worked example, in that order, on one incident.

Turn the signal stack, the join-key spine, the goodput SLO and the straggler rules into the **[fleet observability design](../practice/fleet-observability/README.md)**, then work the **[checkpoint](../checkpoint.md)** — items 4 and 5 are this lesson's material stated as pass criteria.

One thread is deliberately left hanging. To keep this fleet inside its cardinality budget you pushed `pod`, `job_id` and `gpu_uuid` off the series — which are exactly the dimensions a *cost* question needs, and §10's misattribution window is exactly the kind of error a billing dispute surfaces. **[10 · The telemetry lakehouse](10-telemetry-lakehouse.md)** is the other half of that trade: a second path off the same telemetry, tuned for accounting rather than alerting, that answers what this one structurally cannot.

Two sibling modules bracket this one: **[modules/05-gpu-observability](../../../modules/05-gpu-observability/README.md)** is the single-node prerequisite (DCGM internals, the util-lie, per-node MFU/goodput/XID, attribution, inference SLOs) — go there if any of §1's L1/L5 rows felt unfamiliar rather than merely referenced. **[modules/11-gpu-cost-economics](../../../modules/11-gpu-cost-economics/README.md)** is the downstream consumer: every dollar figure in allocated-vs-utilised attribution sits on the goodput and join-key work defined here.

## References & further reading

**Primary sources (read from upstream repositories; rendered doc sites and paper hosts are unreachable from this environment)**
- PyTorch — `pytorch/pytorch`, `torch/csrc/distributed/c10d/FlightRecorder.hpp`. *Verified: `TORCH_FR_BUFFER_SIZE` / `TORCH_NCCL_TRACE_BUFFER_SIZE` default **2000** (0 disables); `TORCH_FR_CPP_STACK` / `TORCH_NCCL_TRACE_CPP_STACK` default false; `TORCH_FR_DUMP_ON_TIMEOUT` / `TORCH_NCCL_DUMP_ON_TIMEOUT`; `TORCH_FR_WAIT_TIMEOUT_DUMP_MILSEC` / `TORCH_NCCL_WAIT_TIMEOUT_DUMP_MILSEC` (15 s); the `Entry` fields (`collective_seq_id`, `p2p_seq_id`, `op_id`, `state`, `duration_ms`, `time_discovered_started_ns`, `time_discovered_completed_ns`, `retired_`) and the `scheduled`/`started`/`completed` states.*
- PyTorch — `torch/csrc/distributed/c10d/ProcessGroupNCCL.hpp` and `.cpp`. *Verified: `kProcessGroupNCCLDefaultTimeout = 10 min`; `kWorkStatusUpdatePeriodMs = 30 s`; `TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC` default `60 × 8` s; `kWatchdogThreadSleepMillis = 100`; `TORCH_NCCL_ASYNC_ERROR_HANDLING` default 3 (`SkipCleanUp`) with the documented rationale that `ncclCommAbort` can itself hang; `TORCH_NCCL_DESYNC_DEBUG` default false.*
- NCCL — `NVIDIA/nccl`, `src/include/plugin/nccl_profiler.h`. *Verified: the v7 profiler interface, the event-type bitmask (`ncclProfileGroup`, `Coll`, `P2p`, `ProxyOp`, `ProxyStep`, `ProxyCtrl`, `KernelCh`, `NetPlugin`, the API events, `KernelLaunch`, the v6 copy-engine events, and the v7 `ncclProfileKernelPhase`), and the proxy-step state names used in §5.*
- NCCL — `NVIDIA/nccl`, `src/plugin/profiler.cc`. *Verified: the plugin is loaded from the `NCCL_PROFILER_PLUGIN` environment variable and the plugin's `init` returns the event activation mask.*
- Meta GCM — `facebookresearch/gcm`, `README.md` and `slurmprocessor/README.md`. *Verified: the three components (Slurm monitoring, lifecycle health checks, the OTel `slurmprocessor`); the claim that it powers Meta FAIR workloads across hundreds of thousands of GPUs; and the processor's `cache_duration: 60` / `cache_filepath` / `query_slurmctld` options with the README's own warning about misattribution at job-lifetime boundaries and load on `slurmctld`.*
- OpenTelemetry — `open-telemetry/semantic-conventions-genai`, `docs/gen-ai/gen-ai-metrics.md`. *Verified: `gen_ai.server.time_to_first_token`, `gen_ai.server.time_per_output_token`, `gen_ai.server.request.duration`, `gen_ai.client.token.usage`, `gen_ai.client.operation.duration`, `gen_ai.client.operation.time_to_first_chunk`; attributes `gen_ai.operation.name`, `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.token.type`; and the explicit TTFT bucket boundaries quoted in §8.*
- NVIDIA dcgm-exporter — `NVIDIA/dcgm-exporter`, `README.md`. *Consulted for the DaemonSet/ServiceMonitor deployment shape referenced in §2; the field-level and label-level detail is module 05's material.*

**Sources this environment could not fetch (used as stated inputs, marked where relied upon)**
- Meta — *The Llama 3 Herd of Models*, arXiv:2407.21783 (https://arxiv.org/pdf/2407.21783). Source of the 16,384-H100 / 54-day / 466-interruption / 419-unexpected figures, the hardware-cause breakdown, the >90% effective-training-time result and the 3-manual-interventions figure. **arxiv.org is blocked by the egress proxy here; these figures are carried forward from this course's earlier verified pass and were not re-checked.** §3's derivation is written to take them as inputs so it remains valid with any fleet's own measurements.
- NVIDIA — NIXT / NCCL Inspector: https://arxiv.org/abs/2608.01449 and the developer-forum announcement https://forums.developer.nvidia.com/t/enhancing-communication-observability-of-ai-workloads-with-nccl-inspector/354225. Source of the "under 2% overhead" and "validated to ≈2,048 GPUs" claims. **Both hosts blocked here; carried forward from the earlier pass and treated as early-stage claims to pilot, not as a solved dependency.** The underlying plugin interface those tools build on *was* verified from NCCL source above.
- CoreWeave — "CoreWeave Drives 20% Higher GPU Cluster Performance": https://www.coreweave.com/blog/coreweave-leads-the-charge-in-ai-infrastructure-efficiency-with-up-to-20-higher-gpu-cluster-performance-than-alternative-solutions. Vendor marketing reporting goodput up to 96% and MFU above 50%. **Not fetched here and not relied upon for any figure in this lesson** — methodology, model size and measurement window are unpublished, so it is directional evidence that goodput has become a customer-facing metric, nothing more.

**Deeper dives**
- NVIDIA DCGM documentation — https://docs.nvidia.com/datacenter/dcgm/latest/ *(field semantics; module 05's territory — not fetched here, `docs.nvidia.com` is blocked).*
- NVIDIA NCCL documentation — https://docs.nvidia.com/deeplearning/nccl/ *(collectives, environment variables, debugging — not fetched here; the profiler-plugin facts above come from the NCCL source tree instead).*
- Grafana Mimir multitenancy and per-tenant limits — https://grafana.com/docs/mimir/latest/ *(the enforcement point for §2's per-tenant series budgets; lesson 03's territory).*
