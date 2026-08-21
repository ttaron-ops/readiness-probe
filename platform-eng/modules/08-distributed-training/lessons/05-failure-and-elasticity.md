---
lesson: "08.5"
title: "Failure and elasticity"
module: "08"
concept: "failure-and-elasticity"
status: not-started
est_time: "7h"
prev: "04-checkpointing.md"
next: "06-job-orchestration.md"
artifacts: []
sources: 14
---
# 08.5 · Failure and elasticity
> **Concept.** A large training run is a single distributed process that dies when any one rank dies; elasticity is the loop — health signal → drain → re-rendezvous → resume-from-checkpoint — that turns a fatal crash into a bounded pause instead of a restart from zero.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

08.4 gave you the checkpoint — the durable floor you fall back to — and the Young/Daly arithmetic that says how often to write it. It also left one term of the goodput expression unexplained:

```
    goodput  =  1  −  √(2C/M)  −  R/M
                      └───┬──┘     └┬┘
                     08.4 owns    THIS LESSON OWNS THIS
```

`R` is the restart cost per failure — detect, drain, re-form the world, reload, warm back up — and 08.4's own table showed why it cannot be ignored: async checkpointing alone reaches 89% goodput, fast restart alone reaches 83%, and only both together reach the >90% territory Llama 3 reports. This lesson is the mechanism behind `R`: what detects the failure, what evicts the node, what re-forms the process group, and what the alternatives are when a stop-the-world restart is itself too expensive.

Read in order, **08.4 → 08.5 → 08.6** is "how expensive is a restart" → "what triggers and executes a restart" → "what Kubernetes object that restart lives inside."

## Why this matters

A synchronous data-parallel run is one process with `N` ranks joined at the hip by a collective. 08.2 established the property that makes this lesson necessary: `all_reduce` is a barrier, every rank must contribute, and there is no partial answer. So the failure semantics are brutal and non-negotiable — **if one rank dies mid-collective, the other `N−1` do not keep going.** They block on an operation that can never complete and, at PyTorch's defaults, die on a timeout ten minutes later.

08.4 derived the number that makes this a first-order concern rather than an occasional nuisance: `M_job = m_gpu / N`. A perfectly ordinary per-accelerator MTBF of ~50,600 GPU-hours becomes **3.1 hours** at 16,384 GPUs and **about 30 minutes** at 100,000. At that rate, the question stops being "will it fail" and becomes "how many minutes does each failure cost, times how many failures per day."

Price it. On a 16,384-GPU H100 fleet at a nominal $3/GPU-hour — **$49,152 per hour** — each *minute* of restart dead time costs **$819**, and at one failure every 3.1 hours you get 7.7 failures per day. **Cutting restart time from 10 minutes to 3 minutes is worth $43,000 per day, or $2.3 M over a 54-day run.** That is one engineer's quarter, paid back in a fortnight, and it is exactly the kind of number a CoreWeave or Anthropic platform interview is probing for.

## What's new here (calibration)

This lesson wires together three things you already have and adds the missing loop:

- **08.4 gave you the checkpoint.** You know it is `~14Ψ` bytes, how the four-stage write path works, and that the last durable checkpoint is the floor you fall back to. This lesson is what *triggers* the fall-back and what re-forms the process group around it.
- **Module 05 gave you the health signal.** You know XID codes and DCGM health. Here that signal becomes an *actuator*: an XID 79 (GPU fell off the bus) or XID 48 (double-bit ECC) or a `DCGM_FI_DEV_XID_ERRORS` reading is the interrupt that fires the drain. **We do not re-teach XID codes — module 05 owns those.** 08.5 owns what you *do* with them.
- **Module 06 placed the gang.** Gang scheduling and topology-aware placement got `N` pods co-scheduled all-or-nothing. The clean division: **06 places the gang; 08 keeps it alive; 05's XID is the signal that triggers a restart.** When a node dies, 06's gang semantics are also what guarantee the *replacement* comes back as a whole group rather than a partial, deadlocked world.

Genuinely new here:

- **Detection as a measurable, tunable latency** rather than an event — with a table of every detector in the stack, what it watches, and how long it takes, because detection is usually the largest slice of `R`.
- **TorchElastic's actual architecture and defaults**, read out of the source: the per-node agent, its 0.1 s monitor loop, the rendezvous state machine, and the four timeouts that govern it. Including the two defaults that surprise everyone.
- **The rendezvous round and its generation number**, which is how a restarted world excludes stale workers.
- **What a membership change does to your training semantics** — global batch size, LR schedule, and gradient accumulation — which is the part of elasticity that quietly breaks correctness rather than availability.
- **`R` decomposed into five measurable line items**, so "make restarts faster" becomes a prioritised work list.
- **Elastic versus fault-tolerant** as a real architectural distinction with a measured 44% → 80% consequence at 100,000-GPU scale, not a vocabulary quibble.

## Core concepts

### 1. The fault domain: why one rank takes the job

08.2 gave the mechanism; here is the consequence, stated as a system property.

**A synchronous training job has a fault domain equal to the whole job.** Not because anyone designed it that way, but because the maths of an all-reduce makes every rank's output depend on every rank's input. There is no correct partial result, so a rank that has not arrived cannot be skipped. The other `N−1` ranks sit inside a spin-wait kernel at 100% reported utilisation, burning money, until something kills them.

That single property generates the entire design space of this lesson, and it is worth naming the three escape routes now because everything below is one of them:

1. **Shrink the time to notice** — detect a dead rank in seconds instead of the ten minutes a NCCL watchdog takes. This is §2, and it is usually the cheapest large win.
2. **Shrink the time to recover** — re-form the world and reload fast, instead of re-queueing the job from scratch. This is §3–§6, and it is what "elastic" means.
3. **Shrink the fault domain itself** — arrange the job so that a failure kills a *replica group* rather than the world. This is §8, it is what "fault-tolerant" means, and it is the only one of the three that changes the asymptotics at 100,000 GPUs.

**Nothing here reduces the failure *rate*.** `M_job = m_gpu / N` is set by hardware and scale. Everything in this lesson attacks the cost per failure.

### 2. Detection — the term that usually dominates `R`

Most engineers assume `R` is dominated by reloading the checkpoint. Decompose it and that is rarely true.

```
    R  =  T_detect  +  T_drain  +  T_rendezvous  +  T_init  +  T_load  +  T_warmup
```

Only `T_load` is about storage. Here is every detector in a normal stack, what it actually watches, and the latency it contributes — every default verified against the PyTorch source at HEAD or the NCCL 2.31 documentation:

| Detector | Watches | Detection latency | Notes |
|---|---|---|---|
| **TorchElastic agent** | its local worker processes' exit status | **~0.1 s** (`--monitor-interval`, default `0.1`) | the fastest thing in the stack, and it only sees *local* process exits |
| **Rendezvous keep-alive** | peer agents' heartbeats | **~15 s** (`keep_alive_interval` 5 s × `keep_alive_max_attempt` 3) | how a *node* disappearing is noticed, as opposed to a process exiting |
| **NCCL RAS** | per-process liveness over the out-of-band network | seconds to query; **60 s** to promote `INCOMPLETE` → `DEAD` | on-demand, not a trigger — you or a supervisor must poll it (08.2) |
| **PyTorch NCCL watchdog** | per-collective elapsed time | **600 s** (`kProcessGroupNCCLDefaultTimeout` = 10 min) | the backstop when nothing else notices; the single largest default in the stack |
| **Watchdog heartbeat monitor** | that the watchdog thread itself is alive | **480 s** (`TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC`, 8 min) | catches a watchdog wedged by a stuck CUDA call |
| **DCGM / node-problem-detector** | XID errors, ECC, thermals, NVLink state (module 05) | seconds to a minute, depending on poll interval | the only detector that fires *before* the job notices anything |
| **Application-level step watchdog** | "has the step counter advanced in T seconds?" | whatever you set | catches hangs with no NCCL involvement — a stuck NFS read in a data loader has no timeout at all |

**Read the table as an ordering problem.** A GPU throwing XID 79 can be caught by DCGM in seconds, or by the NCCL watchdog in ten minutes, depending entirely on whether you wired the first path. **The difference between those two outcomes is 10 minutes × `N` GPUs, on every failure** — at 16,384 GPUs and $3/GPU-hour, $8,200 each time.

Three practical consequences:

- **Fire the drain from the health signal, not from the timeout.** Module 05's DCGM watch or node-problem-detector turning an XID into a node condition and a taint is the fast path. The NCCL watchdog is the backstop for everything the health layer cannot see (host OOM, data-loader deadlock, application bug).
- **Add a step watchdog.** As `stas00/ml-engineering` puts it: a `torch.distributed` hang eventually times out and throws, but a hang inside another syscall — reading from a stuck mount, for instance — has no timeout at all and can sit for hours with nobody the wiser. Since almost every training loop logs periodically, checking that the log file has been touched within the expected window is a cheap, universal detector.
- **Lowering the NCCL timeout is a real, defensible trade.** A shorter `timeout=` on the process group means faster detection but risks killing healthy jobs whose collectives legitimately run long (a very large all-reduce, a slow first step, a checkpoint barrier). The right value depends on your restart cost: with `R = 3 min` a spurious timeout is cheap insurance; with `R = 25 min` it is not. **This is the first place in the module where 08.4's number changes an 08.5 decision.**

### 3. TorchElastic — the architecture

`torchrun` is the console entry point for `torch.distributed.run`, the successor to `torch.distributed.launch`. Its structure is one **elastic agent process per node**; the agent owns the local workers, and the agents collectively own the rendezvous.

```
  TORCHELASTIC — what runs where
  ═══════════════════════════════════════════════════════════════════════════

   node A                       node B                       node C
  ┌──────────────────────┐     ┌──────────────────────┐     ┌───────────────┐
  │ torchrun (AGENT)     │     │ torchrun (AGENT)     │     │ torchrun      │
  │  monitor loop, 0.1 s │     │  monitor loop, 0.1 s │     │  (AGENT)      │
  │  ┌────────────────┐  │     │  ┌────────────────┐  │     │               │
  │  │ LocalWorker    │  │     │  │ LocalWorker    │  │     │   ...         │
  │  │  Group         │  │     │  │  Group         │  │     │               │
  │  │  train.py ×8   │  │     │  │  train.py ×8   │  │     │               │
  │  └────────────────┘  │     │  └────────────────┘  │     │               │
  └──────────┬───────────┘     └──────────┬───────────┘     └───────┬───────┘
             │                            │                         │
             └────────────┬───────────────┴─────────────────────────┘
                          ▼
              ┌───────────────────────────┐
              │  RENDEZVOUS BACKEND       │   keyed by --rdzv-id ("the room")
              │  c10d TCPStore | etcd     │   holds: participants, wait_list,
              │  at --rdzv-endpoint       │          last_heartbeats, ROUND #
              └───────────────────────────┘

   ── the three units, and why the distinction matters ──────────────────────
     Worker           one process running train.py; owns one GPU
     LocalWorkerGroup all workers on one node; what ONE agent manages
     WorkerGroup      the union of all LocalWorkerGroups = the job
                      ▲ THE RESTART UNIT IS THE WORKER GROUP, NOT THE WORKER.
                        One worker dying restarts EVERY worker, everywhere.
```

A launch, with every flag that matters and its real default:

```bash
torchrun \
  --nnodes=2:8 \                    # MIN:MAX — this is what makes it elastic
  --nproc-per-node=8 \              # workers (GPUs) per node
  --max-restarts=3 \                # DEFAULT IS 0. See below.
  --rdzv-backend=c10d \             # DEFAULT IS "static" — not elastic at all
  --rdzv-endpoint=$HEAD_ADDR:29400 \
  --rdzv-id=job-4417 \              # the rendezvous "room" name
  --monitor-interval=0.1 \          # agent poll period (default 0.1 s)
  --tee 3 \                         # tee both streams to per-rank log files
  --log-line-prefix-template '${hostname}:${rank}: ' \
  train.py --resume-from=/ckpt/latest
```

**Two defaults that catch everybody, verified in `torch/distributed/run.py`:**

- **`--max-restarts` defaults to `0`.** Out of the box, torchrun is *not* fault tolerant: the first worker failure fails the job. (Note the trap for readers of the source: the `LaunchConfig` dataclass in `torch/distributed/launcher/api.py` defaults it to 3, but the CLI parser passes 0, and the CLI is what a user gets.)
- **`--rdzv-backend` defaults to `static`.** Static rendezvous means a fixed world assembled from `--nnodes`, `--node-rank` and `MASTER_ADDR` — no membership changes, no elasticity. **You must pass `--rdzv-backend=c10d`** (or `etcd-v2`) to get any of this lesson's behaviour.

The other backends, so you can read someone else's launch script: **`c10d`** is the built-in TCPStore hosted by one of the agents — zero external dependencies, and the current recommendation. **`etcd-v2`** uses an external etcd cluster (v2 API must be enabled on the server); **`etcd`** is the legacy implementation, in maintenance mode and slated for removal. **`--standalone`** spins up a local c10d store on a free port and auto-assigns the endpoint and id — the right thing for single-node development, and it silently ignores any `--rdzv-*` you also passed.

**The environment the agent injects** into every worker — you never set these by hand:

| Variable | Meaning |
|---|---|
| `RANK` | global rank of this worker in the WorkerGroup |
| `LOCAL_RANK` | rank within this node — **this is your CUDA device index** |
| `WORLD_SIZE` | total workers in the job |
| `LOCAL_WORLD_SIZE` | workers on this node; equals `--nproc-per-node` |
| `GROUP_RANK` | rank of this node's worker group, `0..max_nnodes` |
| `ROLE_RANK` / `ROLE_WORLD_SIZE` | rank and size within the same `--role` |
| `MASTER_ADDR` / `MASTER_PORT` | host and port of global rank 0's TCPStore, for `init_process_group` |
| `TORCHELASTIC_RESTART_COUNT` | how many worker-group restarts have happened so far |
| `TORCHELASTIC_MAX_RESTARTS` | the configured maximum |
| `TORCHELASTIC_RUN_ID` | equals the rendezvous `run_id` (`--rdzv-id`) |

**`TORCHELASTIC_RESTART_COUNT` is the underused one.** It lets your script know it is a restart rather than a cold start, which is exactly what you want for logging, for metrics, and for deciding whether to expect a checkpoint to exist.

**On elastic runs `RANK` and `WORLD_SIZE` are not stable across restarts.** The docs are explicit that a job using elastic launch should not have logic depending on `WORLD_SIZE`, on specific ranks, or on any correlation between `RANK` and `LOCAL_RANK`. Any code that pins behaviour to "I am rank 0 forever" — writing checkpoints, owning a metrics socket, holding a lock — is a real and common bug class. Read them fresh from the environment after every `init_process_group()`.

### 4. Rendezvous — how a world is formed and re-formed

The rendezvous backend holds a small piece of shared state keyed by `--rdzv-id`. Its fields, from `_RendezvousState` in `torch/distributed/elastic/rendezvous/dynamic_rendezvous.py`: the set of **participants** and their assigned ranks, a **wait_list** of nodes that arrived after the round closed, **`last_heartbeats`** per node, a **`complete`** flag, and a monotonically increasing **round** number.

```
  THE RENDEZVOUS ROUND — a state machine, with real defaults
  ═══════════════════════════════════════════════════════════════════════════

    ┌─────────────┐  agent starts / a membership change is detected
    │   JOINING   │◀───────────────────────────────────────────────┐
    └──────┬──────┘                                                │
           │  add self to participants; heartbeat every 5 s        │
           ▼                                                       │
    ┌─────────────┐  wait until len(participants) >= MIN           │
    │  ACCUMULATE │  ── if MAX reached, close immediately          │
    └──────┬──────┘  ── else wait `last_call` for stragglers       │
           │            last_call  = 30 s      ◀── grace period    │
           │            join       = 600 s     ◀── give up after   │
           ▼                                                       │
    ┌─────────────┐  assign RANKs 0..WORLD_SIZE-1, elect rank 0    │
    │   CLOSED    │  as MASTER_ADDR, bump ROUND number             │
    └──────┬──────┘                                                │
           │  agents start (or restart) their local workers        │
           ▼                                                       │
    ┌─────────────┐  agent monitor loop, every 0.1 s:              │
    │   RUNNING   │   • any local worker FAILED/UNHEALTHY? ────────┤ restart
    │             │   • num_nodes_waiting() > 0?  ────────────────┤ scale-up
    │             │   • peer heartbeat older than 5 s × 3 = 15 s?  │
    └──────┬──────┘     → evict that node from participants ───────┘
           │
           ▼  all workers SUCCEEDED
    ┌─────────────┐  store_util.barrier(), exit_barrier_timeout = 300 s
    │  EXIT BARRIER│  keeps agents alive until every agent finishes
    └─────────────┘

  ── the four timeouts, all from RendezvousTimeout._DEFAULT_TIMEOUTS ────────
     join       600 s   the whole rendezvous must complete within this
     last_call   30 s   extra wait after MIN is reached, for stragglers
     close       30 s   time allowed to shut a rendezvous down
     heartbeat    5 s   a single keep-alive must complete within this
     (override with --rdzv-conf timeout=...,last_call_timeout=...,
      keep_alive_interval=...,keep_alive_max_attempt=...)

  ── THE ROUND NUMBER IS THE FENCE ─────────────────────────────────────────
     Every completed rendezvous bumps `round`. A worker from an older round
     that wakes up (a paused process, a partitioned node) cannot rejoin the
     current world — its state is stale and gets discarded. Without this, a
     zombie rank could corrupt a freshly formed communicator.
```

**What `--nnodes=MIN:MAX` actually buys.** With `--nnodes=2:8` the job *starts* once 2 nodes have joined and *tolerates* losing nodes down to 2. With a fixed `--nnodes=8`, `MIN = MAX = 8` and any loss is fatal — the rendezvous can never re-close. Choosing `MIN` is a real trade: too low and the job silently limps along at a fraction of its throughput after a bad night; too high and it fails where it could have continued. A common production choice is `MIN` at 80–90% of `MAX`, plus alerting when the world size drops.

**Scale-up uses the same machinery.** The agent's monitor loop calls `rdzv_handler.num_nodes_waiting()`; if new nodes are in the wait list, it restarts the worker group to admit them. The docs are blunt about the cost: **node arrival stops all existing workers, forms a new WorkerGroup, and starts everyone with a new `RANK` and `WORLD_SIZE`** — exactly the same disruption as a departure. Elasticity is symmetric, and it is never free.

**`c10d`'s failure asymmetry, stated precisely.** The agent hosting the TCPStore is a soft single point of failure for *forming* a new rendezvous. Workers already running survive its loss, but a subsequent re-rendezvous has nowhere to register. Production either pins the store to a stable head node that is not also a training node, or uses etcd. This is a real design decision, not a footnote — it is the reason "c10d has no external dependency" is a benefit and a risk at the same time.

### 5. What actually happens when a node dies

Now put §2, §3 and §4 together on a timeline, with 08.4's checkpoint underneath. This is the diagram to be able to draw on a whiteboard.

```
  FAILURE TIMELINE — 4 nodes × 8 GPUs, step 1.2 s, checkpoint every 500 steps
  Failure at step 7,300; last checkpoint at step 7,000 (5 min earlier)
  ═══════════════════════════════════════════════════════════════════════════

  ┌─ (A) FIXED WORLD SIZE, NO CHECKPOINT ───────────────────────────────────┐
  │                                                                          │
  │ t=0      node C hits XID 79; its 8 ranks vanish mid-all-reduce           │
  │ t=0      the other 24 ranks block inside ncclAllReduce                   │
  │          ░░░░░░░░ 24 GPUs at 100% util, ~40% power, ZERO progress ░░░░░░ │
  │ t=600s   NCCL watchdog fires (default 600,000 ms). Processes torn down.  │
  │ t=~610s  agent sees the exits. --max-restarts=0 (default) → JOB FAILS.   │
  │ t=?      job re-queued by hand; waits for 4 healthy nodes.               │
  │ t=?+     training restarts FROM STEP 0.                                  │
  │                                                                          │
  │   LOST WORK  = 7,300 steps × 1.2 s = 2.43 HOURS of 32-GPU compute        │
  │   plus 10 min of full-fleet burn before anyone knew                      │
  │   plus queue wait to re-acquire 32 GPUs                                  │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌─ (B) ELASTIC + CHECKPOINTING (--nnodes=3:4 --max-restarts=3) ───────────┐
  │                                                                          │
  │ t=0.0s   node C hits XID 79                                              │
  │          ├─ FAST PATH: DCGM/node-problem-detector sees the XID           │
  │ t=~5s    │  → node condition → taint → cordon+drain → pod evicted        │
  │ t=~5.1s  ├─ node C's AGENT is gone; its heartbeat stops                  │
  │ t=~20s   ├─ surviving agents notice: 5 s interval × 3 attempts = 15 s    │
  │          │                                                              │
  │          │  ── T_detect ≈ 20 s (or 600 s if you rely on the watchdog) ── │
  │          ▼                                                              │
  │ t=20s    RE-RENDEZVOUS: round N → N+1                                    │
  │          ├─ participants = 3 nodes ≥ MIN(3)  ✔                           │
  │          ├─ wait `last_call` 30 s for stragglers                         │
  │ t=50s    └─ round closes. WORLD_SIZE 32 → 24. RANKs REASSIGNED.          │
  │                                                                          │
  │ t=50s    each agent SIGTERMs its local workers and re-launches train.py  │
  │ t=~55s   ├─ python + CUDA context init                                   │
  │ t=~70s   ├─ init_process_group(): NCCL bootstrap, topology detection,    │
  │          │   ring/tree construction, transport setup  (08.2 §2)          │
  │ t=~100s  ├─ dcp.load() reads the step-7,000 checkpoint IN PLACE          │
  │ t=~110s  └─ first step issued; a few steps to reach steady throughput    │
  │                                                                          │
  │   R = 110 s  ≈ 1.8 min   ┌ detect     20 s ┐                             │
  │                          │ rendezvous 30 s │ ◀── mostly `last_call`      │
  │                          │ proc+CUDA  20 s │                             │
  │                          │ NCCL init  15 s │                             │
  │                          │ ckpt load  30 s │                             │
  │                          │ warmup      5 s │                             │
  │                          └─────────────────┘                             │
  │   LOST WORK = 7,300 − 7,000 = 300 steps × 1.2 s = 6 min                  │
  │   TOTAL PAUSE ≈ 7.8 min   vs   2.43 h + queue                            │
  │                                                                          │
  │   ⚠ AND THE JOB IS NOW RUNNING ON 24 GPUs, NOT 32.                       │
  │     Throughput is 75%. If per-GPU batch is fixed, the GLOBAL BATCH       │
  │     just shrank by 25% — see §6. Elasticity traded availability for      │
  │     a change in training semantics, and nothing warned you.              │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌─ (C) FAULT-TOLERANT, replica-scoped (torchft / FT-HSDP) ────────────────┐
  │ t=0     node C fails                                                     │
  │ t=~0.1s per-STEP heartbeat misses; the manager excludes that replica     │
  │ t=~0.2s the remaining replicas re-form quorum and KEEP TRAINING          │
  │         ░ no world restart, no checkpoint reload, no rank reassignment ░ │
  │ t=later the repaired replica rejoins and pulls current weights from a    │
  │         healthy peer over a checkpoint transport — not from storage      │
  │   LOST WORK ≈ ONE STEP for the surviving replicas                        │
  └──────────────────────────────────────────────────────────────────────────┘
```

**Three properties to internalise from (B).** The restart is *not* surgery on the dead rank — torchrun kills and re-launches **every** worker in the group, so your script must be written to resume from checkpoint on start (idempotent boot). The unit restarted is the **worker group**, not the failed worker. And the detector→drain half of the loop is Kubernetes and infrastructure plumbing that you own; the re-rendezvous→resume half is torchrun's.

**And note what dominates `R` in that breakdown**: detection and the rendezvous `last_call` grace period together are 50 of 110 seconds, while the checkpoint load — the thing everyone optimises first — is 30. If you want a faster restart, tune `--rdzv-conf last_call_timeout` and wire the DCGM fast path *before* you buy faster storage.

### 6. The failure modes of the recovery mechanism itself

Elasticity is machinery, and machinery fails. These are the ones that show up in real postmortems.

**Crash-looping on a poisoned checkpoint.** If the last checkpoint contains a NaN, or the failure is deterministic (a bad batch, an OOM at a specific step), every restart reproduces it. `--max-restarts` is the backstop: it bounds worker-group restarts and fails the job when exhausted. Set it deliberately — high enough to ride out transient hardware, low enough that a deterministic bug does not consume a night. Note that **membership changes do not count against it**; only failures do.

**Rank-identity assumptions.** Covered in §3, and it deserves repeating because it is the most common real bug: after a re-rendezvous, `RANK` and `WORLD_SIZE` change. Anything keyed to rank 0 must re-read the environment.

**The global-batch problem, which is a correctness issue, not an availability one.** This is the part of elasticity most write-ups omit. Your effective global batch is

```
    global_batch  =  WORLD_SIZE  ×  micro_batch  ×  grad_accum_steps
```

When the world shrinks from 32 to 24 and everything else is held constant, the global batch shrinks by 25%. Learning-rate schedules, warmup lengths and token budgets are all calibrated against a *fixed* global batch, so the run silently changes its training recipe mid-flight. Three honest options, and you should be able to name all three:

1. **Hold the global batch constant by raising `grad_accum_steps`** on the surviving ranks. Correct, and the usual choice — but each rank now does more work per step, so step time rises and the throughput loss is larger than the GPU loss.
2. **Let the global batch shrink** and accept the recipe change. Defensible only if you re-derive the LR schedule, and it makes "tokens seen" bookkeeping harder.
3. **Refuse to run below full size** (`MIN = MAX`) and treat any loss as a hard stop. Correct, simplest, and what a lot of frontier pre-training actually does — which is why "we run torchrun elastic" often means "we run torchrun with `max_restarts` and a fixed world size."

**Not every failure is a clean exit.** A rank that is *slow* rather than dead — a thermally throttled GPU, a straggler from MoE routing imbalance, a node with a degraded NIC — never triggers the agent's exit-status check. It just makes every collective slower for everyone (08.2 §12 shows how RAS distinguishes lagging from missing). Elastic restart does nothing for this case. Handling it needs a separate detector: per-rank step-time outlier detection, and a policy that *proactively* drains a persistent straggler.

**The drain must be idempotent and bounded.** A node condition that flaps produces a drain, a restart, a rejoin, and another drain. Every one of those cycles costs `R`. Production draining logic needs hysteresis and a per-node cooldown, or a marginal node becomes a permanent tax.

**Double-counting restarts.** There are two independent restart layers — the pod-level one (Kubernetes `restartPolicy`, 08.6) and the worker-group one (`--max-restarts`). If both are enabled with generous limits, a job can spend a very long time restarting itself in two nested loops with no forward progress. 08.6 covers the interaction; for now, know that the two exist and must be reasoned about together.

### 7. The production trigger chain

In production the human is not watching. The loop is armed by module 05's telemetry:

```
  XID / DCGM (mod. 05)  →  node condition + taint (node-problem-detector)
        →  cordon + drain  →  pod evicted (mod. 06 gang semantics)
        →  agent sees worker exit  →  RE-RENDEZVOUS (torchrun)
        →  resume from checkpoint (08.4)
```

**Each module owns exactly one arrow, and the chain is the single most interview-probed sequence in this module.** Be able to recite it and name the owner of each step.

Two refinements worth carrying beyond the recitation:

- **Not every XID should drain.** Module 05's taxonomy matters here: XID 79 (GPU fell off the bus) and XID 48 (double-bit ECC) are unambiguous drains. A single correctable ECC error is not — draining on it turns a healthy fleet into a churning one. The policy question "which conditions actuate" is yours to own, and getting it wrong is expensive in both directions.
- **Hardware is not the whole story.** Llama 3's figure is ~78% hardware-caused, which means **~22% is not**: software bugs, data-loader stalls, host OOM-kills (08.2's worked example is exactly this), storage outages, and configuration errors. Pure hardware-health monitoring will never see that fifth, which is precisely why the application-level step watchdog from §2 earns its place.

### 8. Elastic versus fault-tolerant — and torchft

**The distinction, in one line each.** *Elastic* training means the world can change size and the job survives **via a full worker-group restart** — it is about **membership**. *Fault-tolerant* training means individual failures are absorbed **without a full-world restart** — it is about **continuity**. Elasticity bounds the cost of a restart; fault tolerance removes the restart.

That distinction is not academic. §5's timeline (C) shows why: at 100,000 GPUs with `M_job ≈ 30 minutes`, even a "cheap" 3-minute stop-the-world restart is 10% of wall-clock before any rework.

**torchft** (`meta-pytorch/torchft`) is the open-source implementation of the fault-tolerant approach. Its architecture, and the real defaults from the source:

- **Lighthouse** — a coordination server (Rust) that all replica groups connect to. It decides quorum. CLI defaults: `--bind [::]:29510`, `--join_timeout_ms 60000` (how long to wait for heartbeating stragglers before issuing quorum), `--quorum_tick_ms 100`, `--heartbeat_timeout_ms 5000` (how long before a replica is considered dead).
- **Manager** — one per replica group, running in the training process. Its `heartbeat_interval` defaults to **100 ms**, with `timeout`, `quorum_timeout` and `connect_timeout` each defaulting to 60 s. **Health is determined per training step, not by waiting for a collective timeout** — which is the entire point: detection latency drops from ten minutes to roughly one step.
- **Fault-tolerant `ProcessGroup`** — a wrapper that reports errors sanely and **reinitialises gracefully**, so a failed collective becomes a recoverable error rather than a dead process.
- **Checkpoint transports** — a recovering or newly added replica **pulls current weights from a healthy peer** rather than reloading from durable storage. Recovery at step granularity, no cold start, and — critically — no dependence on the storage path that 08.4 §7 showed is often the slowest thing in the stack.

Out of the box it covers **fault-tolerant DDP** and **fault-tolerant HSDP** (fault tolerance across the replicated dimension, with any mix of FSDP/TP on the other dimensions), plus **LocalSGD** and **DiLoCo**, the semi-synchronous algorithms where replica groups sync infrequently and per-group fault tolerance buys the most. torchtitan wires it into an end-to-end HSDP loop.

**The measured payoff, at the scale that motivates it.** Meta's FT-HSDP work reports that using data-parallel replicas as the unit of fault tolerance — taking offline and restarting only the replica containing the failed GPU or server, while the others keep training — **reduces stall time from failure recovery from ~10 minutes to ~3 minutes and raises effective training time from 44% to 80%** compared with fully synchronous training at O(100K) GPUs, with no meaningful degradation in final model accuracy. The design includes a Fault Tolerant All Reduce (FTAR) protocol for gradient exchange across replicas that drives the control logic from the CPU while the GPU does the data movement, plus a non-blocking catch-up protocol so a recovering replica rejoins with minimal stall.

**Frame it for an interview.** 44% is roughly what stop-the-world elastic restart delivers once failure frequency scales with GPU count; 80% comes from swapping the recovery *mechanism*, not from tuning its parameters. Without the membership-versus-continuity distinction the jump looks like a generic optimisation; with it, it is obviously a different architecture.

**And be honest about deployment reality.** torchrun elastic is what most shops run today. torchft is where hyperscale training loops are heading, and at frontier scale it is already load-bearing. Name it as *direction*, backed by a number.

### 9. Budgeting `R` — turning the mechanism into a work list

Bring it back to 08.4's goodput expression. `R` is five or six measurable line items, and they have very different costs to attack:

| Line item | Typical | Cheapest fix | Effort |
|---|---|---|---|
| `T_detect` | 20 s (health-signal path) to **600 s** (NCCL watchdog) | wire DCGM/NPD → taint; add a step watchdog; consider lowering the PG timeout | low, high value |
| `T_drain` | seconds to a minute | ensure the gang controller evicts as a group (06); add hysteresis | low |
| `T_rendezvous` | ~30–60 s, mostly `last_call` | tune `--rdzv-conf last_call_timeout`; use a stable store host | low |
| `T_init` | 10–60 s, grows with world size | NCCL init is topology discovery + graph search + transport setup (08.2 §2); little to tune, but measure it |  — |
| `T_load` | `checkpoint_bytes / read_bandwidth` | sharded parallel read (08.4 §3); keep the two latest checkpoints local | medium |
| `T_warmup` | a few steps | nothing, usually | — |

**The ordering is the point.** Almost everyone optimises `T_load` first because it is the one with an obvious knob, and in the §5 breakdown it is 30 s of 110. Detection plus rendezvous is 50 s of the same 110 and is mostly *configuration*. **Measure the split before you optimise it** — which is exactly what the Practice section asks you to do.

## Perspectives

**Systems / mechanism view.** The conceptual spine is the loop — detect → drain → re-rendezvous → resume — plus the crisp elastic-versus-fault-tolerant distinction: membership change with restart, versus continuity without restart. Everything else in this lesson is elaboration. The single most useful structural insight is that **the restart unit is the worker group**, which is why "one GPU died" and "the whole job restarted" are the same event, and why shrinking the *fault domain* (§8) is categorically different from shrinking the *recovery time* (§5).

**Statistical / fleet-operator view.** Meta's FAIR team analysed 11 months of data across two production ML clusters — 4 million jobs and over 150 million A100-GPU-hours — and found that while large jobs are individually the most failure-vulnerable, **small jobs dominate the population by count**. That reframes the problem: it is not enough to build an elaborate recovery loop for the one dramatic 16K-GPU run; the reliability system has to work uniformly across a much larger population of small, less-visible jobs. That is a genuinely different lens from the Llama-3 single-run narrative anchoring this module, and it is the one that matches what a platform team actually operates day to day.

**Frontier / research-lab view.** FT-HSDP shows *why* hyperscalers are moving past stop-the-world restart: at O(100K) GPUs, restarting the entire world for one failed GPU is prohibitive, so restart only the data-parallel replica containing the failure. The restart-cost-scales-with-`N` argument is precisely what motivates that architecture, and the reported 10 min → 3 min stall and 44% → 80% effective time is the payoff. Note the shape of the idea: it converts a *global* barrier into a *group-scoped* one, which is the same move that HSDP itself made for bandwidth (08.1) — keep the expensive coupling local.

**Economics view.** Every minute of restart time, multiplied across thousands of GPUs and by the failure rate, is directly priceable, and this lesson's mechanisms are the levers. But the more useful economic observation is about *ordering*: detection is mostly configuration and is often the largest term, so the cheapest large win is usually a Grafana panel and a taint rule rather than a storage purchase. That is the kind of finding that makes a platform engineer's first quarter look good.

**Correctness view.** Elasticity is usually pitched as an availability feature, and the availability story is genuinely good. The under-discussed cost is that a membership change **changes your training recipe**: global batch, and therefore effective learning rate, and therefore the schedule the run was calibrated against. A job that "survived" a node loss by shrinking from 32 to 24 GPUs and kept its per-GPU batch is not the run you configured. Either compensate with gradient accumulation or decide, in advance and in writing, that you will not run below full size.

## Real-world use cases

- **Meta — "Training LLMs with Fault Tolerant HSDP on 100,000 GPUs"** ([arxiv.org/abs/2602.00277](https://arxiv.org/abs/2602.00277), January 2026). FT-HSDP uses data-parallel replicas as the unit of fault tolerance: only the replica containing the failed GPU or server is taken offline and restarted while the others keep training, supported by a Fault Tolerant All Reduce protocol and a non-blocking catch-up protocol. **Stall time from failure recovery falls from ~10 min to ~3 min; effective training time rises from 44% to 80%** at O(100K) GPUs, with no meaningful accuracy degradation. *What it shows:* the quantified bridge from elastic stop-the-world recovery to replica-scoped fault tolerance, and the clearest evidence that at extreme scale the recovery *mechanism*, not its tuning, is the lever.

- **Meta — "Revisiting Reliability in Large-Scale Machine Learning Research Clusters"** ([arxiv.org/abs/2410.21680](https://arxiv.org/abs/2410.21680), HPCA 2025). 11 months of data, 4 M jobs, >150 M A100-GPU-hours across two production clusters. *What it shows:* a rigorous large-`N` failure taxonomy rather than a single-run anecdote — large jobs are most failure-vulnerable individually, but small jobs dominate by count. The right base rate for capacity and reliability planning, and a corrective to designing only for the mega-run.

- **PyTorch — "Fault Tolerant Llama: training with 2,000 synthetic failures every ~15 seconds and no checkpoints on Crusoe L40S"** (June 2025). The official torchft stress test: 2,000+ synthetic node failures injected into a 300-GPU L40S run, with **zero stop-the-run checkpoint restarts** and a converging loss curve. *What it shows:* the fault-tolerant claim exercised adversarially rather than demonstrated on a happy path — the failure rate is far beyond anything real hardware produces, which is the point.

- **Meta — OPT-175B training chronicles** ([github.com/facebookresearch/metaseq](https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles)). A day-by-day public logbook of incidents, restarts and manual interventions during the OPT-175B run: at least 35 manual restarts and 100+ hosts cycled over roughly two months. *What it shows:* the `R` term when it is a human. Every restart in that log is a person noticing, deciding, and re-launching — the cost baseline that everything in this lesson exists to remove.

- **`stas00/ml-engineering` — the fault-tolerance chapter.** Practitioner recipes from a BLOOM engineer: keep 5–10% spare nodes and validate that your scheduler auto-drains bad ones; queue a serial job array (`sbatch --array=1-10%1`) so a crash immediately starts the next job rather than waiting for a human; prefer fixed accelerator allocations because dynamic pools hand you other users' rejected nodes; and add an **is-job-hanging watchdog** that checks the log file's mtime, because a hang outside `torch.distributed` has no timeout at all. *What it shows:* the same loop this lesson describes, implemented in a Slurm shop without Kubernetes — which is what most academic and many industrial clusters actually are.

## Worked example

**The scenario.** A 4-node × 8-GPU H100 DDP run. Step time 1.2 s. Checkpointing every 500 steps (~10 min) synchronously at `C = 45 s`. At step 7,300 a node hits XID 79. Nobody is watching. What does this cost, what should it cost, and what do you change?

**Step 1 — what the current setup actually does.** The launch script is `torchrun --nnodes=4 --nproc-per-node=8 train.py`. Read it against §3: `--rdzv-backend` is unset, so it defaults to `static`; `--max-restarts` is unset, so it defaults to 0. **This job is not elastic and not restartable.** No health-signal path is wired, so nothing sees the XID.

```
   t=0      8 ranks vanish. 24 ranks block in ncclAllReduce.
   t=600s   NCCL watchdog fires — the only detector present.
   t=610s   agent sees exits; max_restarts=0 → the job exits FAILED.
   t=?      a human notices, re-queues, waits for capacity.

   burn while blocked  = 32 GPUs × 610 s          = 5.4 GPU-hours   = $16
   lost work           = 300 steps × 1.2 s        = 6 min           ← cheap!
   human latency       = whatever it is           ← the real cost
```

**The interesting finding is that the lost work is trivial and the human latency is everything.** Checkpointing every 10 minutes already bounds rework to ~6 minutes. If the failure happens at 02:00 and someone notices at 08:00, that is **6 hours × 32 GPUs = 192 GPU-hours = $576** — thirty-six times the cost of the failure itself. **Automation of restart, not checkpoint tuning, is the fix here**, and the Young/Daly arithmetic from 08.4 will not tell you that. Only decomposing `R` will.

**Step 2 — job MTBF, so you know how often you are paying it.**

```
   M_job = m_gpu / N = 50,600 / 32 = 1,581 h ≈ 66 days
```

At 32 GPUs failures are rare. **That is itself a design input:** it means a simple, robust automation is worth more than a sophisticated one, because you will exercise it roughly once every two months and it must work unattended when you do.

**Step 3 — the redesign, and its measured effect.**

```bash
torchrun \
  --nnodes=3:4 \                       # tolerate losing one node
  --nproc-per-node=8 \
  --max-restarts=3 \                   # was 0 by default
  --rdzv-backend=c10d \                # was "static" by default
  --rdzv-endpoint=$HEAD:29400 \
  --rdzv-id=$JOB_ID \
  --rdzv-conf=last_call_timeout=10 \   # was 30 s; we know MIN quickly
  --tee 3 --log-line-prefix-template '${hostname}:${rank}: ' \
  train.py --resume-from=/ckpt/latest
```

plus, on the infrastructure side, a DCGM watch that turns XID 79/48 into a node taint, and a step watchdog that alerts if the training log has not been written for 5 minutes.

```
   T_detect      DCGM sees XID, taints, pod evicted, agent heartbeat lost
                 5 s + 15 s (keep_alive 5 s × 3)                     = 20 s
   T_rendezvous  round closes after last_call (now 10 s)             = 12 s
   T_init        process + CUDA context + init_process_group          = 35 s
   T_load        98 GB checkpoint, sharded parallel read              = 25 s
   T_warmup      a few steps to steady state                          =  5 s
   ─────────────────────────────────────────────────────────────────────────
   R                                                                  = 97 s
   lost work (300 steps × 1.2 s)                                      = 360 s
   TOTAL PAUSE                                                        ≈ 7.6 min
```

**From "however long until a human notices" to 7.6 minutes, unattended.**

**Step 4 — check the goodput arithmetic (08.4's model), and notice what it does not capture.**

```
   M = 1,581 h = 94,860 min,  C = 45 s = 0.75 min,  R = 97 s = 1.62 min

   τ_opt      = √(2 × 0.75 × 94,860)  = √142,290 = 377 min ≈ 6.3 hours
   floor waste= √(2 × 0.75 / 94,860)  = 0.40%
   R/M        = 1.62 / 94,860         = 0.0017%
   total waste                        ≈ 0.40%   →  goodput 99.6%
```

Two readings, and the second is the important one. First: the current 10-minute checkpoint interval is **38× more frequent than optimal** at this scale, costing 7.5% of wall-clock in overhead (`0.75/10`) against the 0.4% the optimum would cost. Lengthening the interval to a few hours is free money. Second, and more subtly: **the model says restart cost is negligible here (0.0017%), yet fixing restart was obviously the right call.** The model prices the *steady-state* expectation; it does not price the tail event where a failure lands at 02:00 and nobody is awake. At small `N`, `R/M` is small but the *variance* is what hurts. Use the formula for interval selection, and use judgement for automation.

**Step 5 — what you would do differently at 16,384 GPUs.**

```
   M = 50,600 / 16,384 = 3.09 h = 185 min
   R = 97 s = 1.62 min  →  R/M = 0.87%
   C: 5.67 TB sync at 20 GB/s = 284 s → async ≈ 3 s = 0.05 min
   τ_opt = √(2 × 0.05 × 185) = 4.3 min
   floor waste = √(2 × 0.05 / 185) = 2.3%
   total ≈ 3.2%  →  goodput ~96.8% (model), reality nearer 90% (08.4 §Worked)
```

**Everything inverts.** At 32 GPUs the checkpoint interval was 38× too aggressive and restart cost was noise; at 16,384 the interval must be 4 minutes and `R` is now a percentage point of the entire run. **The same job, the same code, opposite conclusions — and the only thing that changed is `N` in `M = m/N`.** That is the sentence to take into an interview.

**The recommendation you deliver.** Two config lines (`--rdzv-backend=c10d`, `--max-restarts=3`) plus a DCGM taint rule take unattended recovery from "hours until a human notices" to 7.6 minutes, and cost nothing. Separately, lengthen the checkpoint interval from 10 minutes toward the Young/Daly optimum for this scale. Neither change touches the model. And write down explicitly whether you will run at 24 GPUs after a loss or halt: if you will, add gradient accumulation to hold the global batch constant (§6), and if you will not, set `MIN = MAX` and be honest that you are buying automatic *retry*, not elasticity.

## Practice — feeds the "Survive-a-failure" deliverable

See [`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md) for the full deliverable spec. This lesson's slice: run the same DDP toy job (the checkpointing job from 08.4) three ways and quantify the gap.

**Run A — no safety net (the default you will inherit).**
1. Launch with a *fixed* `--nnodes`, **no** `--rdzv-backend`, **no** `--max-restarts`, and no checkpoint load on boot (or delete the checkpoint directory). This is plain `torchrun --nproc-per-node=2 train.py`, i.e. what everyone actually types.
2. `kill -9` one worker mid-run at a known step.
3. Observe: the survivor blocks on the collective; note the wall-clock until the watchdog fires, and the exact message it prints. Confirm the job exits rather than restarting.
4. **Measure:** steps completed before the kill; time-to-detection; and that resuming means starting from step 0. Record "lost work = all of it."

**Run B — elastic + checkpointing.**
1. Relaunch with `--nnodes=1:2 --rdzv-backend=c10d --rdzv-id=$(uuidgen) --max-restarts=3`, and a boot path that loads the latest 08.4 checkpoint.
2. Kill one worker mid-run at a known step.
3. **Watch the agent logs** for the re-rendezvous: the new round, the new `WORLD_SIZE`, the reassigned `RANK`s. Confirm the script reloads the checkpoint and continues.
4. **Measure `R`, itemised** — not as one number. Instrument your script to log a timestamp at process start, after `init_process_group()`, after checkpoint load, and at the first completed step. Combined with the kill timestamp and the agent log, that gives you `T_detect`, `T_init`, `T_load` and `T_warmup` separately. Also record the resume step versus the kill step: that difference is your rework.

**Run C — make the world change size, and look at what else changed.**
1. With the elastic launch from B still running on 2 GPUs, kill one worker and let the job continue at `WORLD_SIZE=1`.
2. **Log the effective global batch before and after.** Confirm it halved (assuming fixed per-GPU batch and no accumulation).
3. Now add gradient accumulation that compensates — `grad_accum_steps` scaled by `initial_world_size / current_world_size`, read fresh from the environment after `init_process_group()` — and confirm the global batch is preserved. **This is the correctness half of elasticity and it is the part most people skip.**

**Run D — tune the biggest term.**
1. Take your itemised `R` from Run B and find the largest line item. It will usually be detection or rendezvous, not the checkpoint load.
2. Change one thing that targets it — `--rdzv-conf last_call_timeout=5`, or a lower process-group `timeout=`, or an application-level step watchdog that exits the process on stall — and re-measure.
3. **Report the before/after and say which term you moved.**

**Acceptance (deliverable):** a killed-worker recovery demo, *with versus without* elasticity and checkpointing, reporting: (i) lost work without checkpointing; (ii) `R` **itemised** into detection, rendezvous, init, load and warmup, with the method you used to measure each; (iii) resume-step versus kill-step, i.e. rework; (iv) the goodput difference stated plainly (e.g. "hours → 7.6 min"); (v) evidence the world actually re-formed — extract the new `WORLD_SIZE` and `RANK` from the worker logs; and (vi) the global-batch observation from Run C and how you handled it. Bonus: the before/after from Run D and which line item you attacked.

## Common pitfalls

- **"`torchrun` is elastic and fault-tolerant by default."** *Mechanism:* `--rdzv-backend` defaults to **`static`** and `--max-restarts` defaults to **0** in `torch/distributed/run.py`. A plain `torchrun --nnodes=4 --nproc-per-node=8 train.py` forms a fixed world and fails the job on the first worker failure. You get elasticity only by explicitly passing `--rdzv-backend=c10d` (or `etcd-v2`), `--nnodes=MIN:MAX`, and a non-zero `--max-restarts`. Reading the `LaunchConfig` dataclass instead of the CLI parser will mislead you, because the dataclass defaults differ from the flags.

- **"Elastic training and fault-tolerant training are the same thing."** *Mechanism:* elastic (torchrun) survives via a **full worker-group restart** — every survivor is SIGTERMed, re-rendezvous, and reloads a checkpoint. Fault-tolerant (torchft, FT-HSDP) survives **without one** — per-step heartbeats detect the failure, a wrapped process group reinitialises, and the affected replica pulls live state from a healthy peer. Elasticity is about membership; fault tolerance is about continuity. The measured 44% → 80% effective-time result only makes sense once that distinction is clear: it is a mechanism swap, not a tuning change.

- **"`RANK` and `WORLD_SIZE` are stable identifiers."** *Mechanism:* every completed rendezvous round assigns fresh global ranks and a new world size. Code that pins behaviour to "I am rank 0 forever" — checkpoint writing, metrics ownership, log file naming — breaks after the first membership change, and breaks *silently* if two ranks both think they are rank 0 in different rounds. Read them from the environment after every `init_process_group()`.

- **"c10d has no single point of failure since it is built in."** *Mechanism:* the agent hosting the TCPStore is a soft SPOF for *forming a new* rendezvous. Already-running workers survive its loss, but a subsequent re-rendezvous has nowhere to register and the job cannot recover. Production pins the store to a stable host that is not itself a training node, or uses etcd — which trades the dependency for the availability.

- **"Failure rate is roughly constant per GPU regardless of scale."** *Mechanism:* per-GPU rate is roughly constant; **job** MTBF is not. Because any rank's death kills the job, `M_job = m_gpu / N` (08.4 §6). An ordinary 5.8-year per-GPU MTBF gives 66 days at 32 GPUs, 3.1 hours at 16,384, and ~30 minutes at 100,000. That is exactly why partial-restart architectures are over-engineering at 100 GPUs and forced at 100,000 — the maths only bites at scale, and the same code base needs different answers at each end.

- **"Hardware failures are the whole story."** *Mechanism:* Llama 3's figure is ~78% hardware-caused, so ~22% is not — software bugs, data-loader stalls, host OOM-kills, storage outages, config errors. Pure hardware-health monitoring (XID/DCGM) will never fire on those, and a NCCL watchdog only catches them if the stall happens inside a collective. A stall inside a syscall with no timeout (a hung network filesystem read) can sit indefinitely. An application-level step watchdog is the only detector that covers the whole space.

- **"Elasticity is free availability."** *Mechanism:* a smaller world with the same per-GPU batch means a smaller **global batch**, which changes the effective learning rate and invalidates the schedule the run was calibrated against. The run keeps going and quietly trains a different recipe. Either compensate with gradient accumulation scaled to the current world size, or set `MIN = MAX` and accept that you have automatic retry rather than elasticity. Decide in advance and write it down.

- **"A restart fixes it."** *Mechanism:* if the cause is deterministic — a poisoned checkpoint, a bad batch at a specific step, an OOM that reproduces — every restart reproduces it, and with a generous `--max-restarts` plus a pod-level `restartPolicy` you can burn a whole night in two nested restart loops with no progress. Bound both layers, and alert on `TORCHELASTIC_RESTART_COUNT` climbing without the step counter advancing.

## Self-check

- **What happens to the surviving ranks when one rank dies mid-collective?**
  **Answer:** Nothing good on their own. A synchronous collective is a barrier and an all-reduce's output on any rank depends on every rank's input, so there is no correct partial result: the survivors block inside a spin-wait kernel that reports 100% GPU utilisation at well below normal power, with the step counter frozen, and no error. At PyTorch's defaults the NCCL watchdog fires after **600 s** (`kProcessGroupNCCLDefaultTimeout` = 10 minutes) with a message naming the *waiters*, not the rank that left, and `TORCH_NCCL_ASYNC_ERROR_HANDLING` (default 3) then tears the processes down. Elasticity exists precisely because the default outcome is total collapse: with a c10d rendezvous the agents notice the departed node's missing heartbeat in ~15 s (5 s interval × 3 attempts) and force a re-rendezvous into a smaller world instead of waiting for the timeout — cutting detection from ten minutes to twenty seconds, which on a 16,384-GPU fleet is worth about $8,000 per incident.

- **What triggers an automated restart in production (tie to module 05)?**
  **Answer:** A GPU health signal. Module 05's DCGM health check or the XID error counter (`DCGM_FI_DEV_XID_ERRORS`) surfaces a fault — XID 79 "GPU fell off the bus" or XID 48 double-bit ECC being the unambiguous ones. Node-problem-detector or a DCGM watcher turns that into a **node condition** and a **taint**; the taint cordons and drains the node; module 06's gang semantics evict and replace the group as a whole rather than leaving a partial, deadlocked world; torchrun's surviving agents observe the departure and force a re-rendezvous; each agent re-launches its workers, which reload 08.4's last checkpoint. The chain to recite: **XID/DCGM (mod. 05) → node taint → cordon+drain → gang evict (mod. 06) → re-rendezvous (torchrun) → resume from checkpoint (08.4)**, one arrow per module. Two refinements: not every XID should actuate a drain (a single correctable ECC error should not), and ~22% of Llama 3's interruptions were *not* hardware, so this chain needs an application-level step watchdog beside it to catch host OOMs, data-loader stalls and software bugs.

- **Elastic training versus fault-tolerant training — what is the difference, and what does it buy?**
  **Answer:** **Elastic** (torchrun): the world can change size and the job survives by **restarting the worker group** — every survivor is SIGTERMed, the agents re-rendezvous into a new round with a new `WORLD_SIZE` and reassigned `RANK`s, and each worker reloads a durable checkpoint. It is about **membership**, and it bounds the cost of a restart to `R + rework`. **Fault-tolerant** (torchft, Meta's FT-HSDP): individual failures are absorbed **without a full-world restart** — a Lighthouse/Manager pair heartbeats every training step (manager `heartbeat_interval` 100 ms, lighthouse `heartbeat_timeout_ms` 5000), a wrapped fault-tolerant `ProcessGroup` turns a failed collective into a recoverable error, and a recovering replica pulls current weights from a healthy peer over a checkpoint transport rather than from storage. It is about **continuity**, and it removes the restart. The measured difference at O(100K) GPUs: stall time from failure recovery **~10 min → ~3 min**, effective training time **44% → 80%**. The reason the gap is so large is `M_job = m_gpu/N`: at 100,000 GPUs failures arrive roughly every 30 minutes, so even a 3-minute stop-the-world restart is 10% of wall-clock before any rework.

- **Decompose restart cost `R`. Which term usually dominates, and which do people optimise first?**
  **Answer:** `R = T_detect + T_drain + T_rendezvous + T_init + T_load + T_warmup`. **Detection usually dominates**, and it varies by more than an order of magnitude depending purely on configuration: the TorchElastic agent notices a local process exit in ~0.1 s (`--monitor-interval`); a peer *node* disappearing takes ~15 s (`keep_alive_interval` 5 s × `keep_alive_max_attempt` 3); NCCL RAS answers a query in seconds but takes 60 s to promote a process to `DEAD`; the NCCL watchdog is **600 s**; and DCGM/node-problem-detector can fire in seconds if you wired it. Rendezvous adds ~30 s, most of it the `last_call` grace period (default 30 s), tunable via `--rdzv-conf`. `T_init` is NCCL bootstrap, topology discovery and transport setup, growing with world size. **`T_load` — the checkpoint read — is what almost everyone optimises first, and in a typical breakdown it is 30 s of 110.** Detection plus rendezvous is 50 s of the same 110 and is mostly configuration, so the cheapest large win is a taint rule and a timeout change, not faster storage. In 08.4's goodput expression `R` contributes a flat `R/M` and, unlike `C`, does **not** change the optimal checkpoint interval.

- **Why does a study across 4 M jobs and >150 M A100-GPU-hours matter more than the Llama-3 anecdote for capacity planning?**
  **Answer:** The Llama-3 numbers describe one mega-run and are excellent for calibrating the extreme case — 16,384 GPUs, 419 unexpected interruptions in 54 days, ~78% hardware. Meta's HPCA 2025 reliability study describes the *distribution* across two production clusters over 11 months: large jobs are individually the most failure-vulnerable, but **small jobs dominate the population by count**. A reliability system designed only for the dramatic 16K-GPU outage will under-serve the far larger population of smaller jobs a platform team actually operates, where the economics are different — at 32 GPUs `M_job` is ~66 days, so the failure is rare but the *human latency* to notice it dominates the cost, and simple robust automation beats sophisticated machinery. Planning from a single anecdote also gives you the wrong base rate and the wrong failure taxonomy.

- **You enable `--nnodes=3:4` and a node dies. The job continues on 24 GPUs. What did you just change about the training run, and what are your options?**
  **Answer:** You changed the **global batch size**, and therefore the training recipe. `global_batch = WORLD_SIZE × micro_batch × grad_accum_steps`; with per-GPU batch and accumulation held constant, dropping from 32 to 24 GPUs shrinks the global batch by 25%. Learning-rate schedules, warmup lengths and token budgets are calibrated against a fixed global batch, so the run silently continues under a different recipe with no error and no warning — and throughput has dropped 25% as well, which is the part you *will* notice. Three options: **(1)** hold the global batch constant by scaling `grad_accum_steps` by `initial_world_size / current_world_size`, read fresh from the environment after each `init_process_group()` — correct, and the usual choice, though step time rises so the throughput loss exceeds the GPU loss; **(2)** let it shrink and re-derive the schedule, defensible only if you actually do it; **(3)** set `MIN = MAX` and treat any loss as a hard stop, which is what a lot of frontier pre-training does — meaning "we run torchrun elastic" often really means "we run torchrun with `max_restarts` and a fixed world size." Decide in advance and write it down; discovering it from a loss curve three days later is the expensive path.

- **A job's logs show `TORCHELASTIC_RESTART_COUNT` climbing but the step counter never advances. What is happening and what do you do?**
  **Answer:** A **crash loop**. The worker group restarts, boots, loads the checkpoint, hits the same deterministic failure, and restarts again — burning `R` on every cycle with zero forward progress. Causes: a poisoned checkpoint (a NaN in the state), a deterministic data-dependent failure (a bad sample, an OOM at a specific sequence length), a persistently unhealthy node that keeps being re-scheduled into the gang, or a config error that only manifests after `init_process_group()`. `--max-restarts` is the intended backstop and bounds the damage — note that membership changes do not count against it, only failures do. The trap to check for is **double-counting**: if a pod-level `restartPolicy` (08.6) is also enabled with a generous limit, you have two nested restart loops and the effective bound is their product. Response: bound both layers, alert on restart count climbing while the step counter is flat (that conjunction is the signal, not either alone), roll back to an earlier checkpoint if a NaN is suspected, and check whether the same node keeps rejoining — a drain policy without hysteresis will happily re-admit a marginal node forever.

## Connections & what's next

This lesson is the seam between "how expensive is a restart" (08.4's `C` and the Young/Daly interval) and "what object on Kubernetes actually restarts" (08.6's PyTorchJob / TrainJob). You now own the `R/M` term of the goodput expression, decomposed into five measurable line items with a prioritised list of what to attack, plus the architectural distinction — membership versus continuity — that explains why 100,000-GPU fleets need a different mechanism rather than a better-tuned one.

**08.6** shows the CRD and controller that generate the rendezvous environment this lesson assumed existed — the headless Service that resolves `MASTER_ADDR`, the `restartPolicy` that interacts with `--max-restarts`, and the two restart layers you must not double-count. **08.7** covers the data pipeline, which is both a cause of low MFU (08.3) and, as this lesson noted, a source of hangs that no hardware-health detector will ever see. **08.8's capstone** takes `C`, `R`, `M`, `τ` and MFU and produces one cost-per-successful-run figure; the itemised `R` from this lesson's Practice is a direct input.

Backward: **08.2** supplied the barrier semantics that make one rank fatal, the watchdog defaults that set the slow detection path, and the RAS tooling that localises the dead rank; **08.4** supplied the checkpoint you reload and the `M = m/N` relation that makes all of this scale-dependent; **module 05** supplied the health signals that arm the loop; **module 06** placed the gang whose replacement must come back whole.

## References & further reading

**Primary sources**

1. **PyTorch — `torch/distributed/run.py`** — <https://github.com/pytorch/pytorch>. **Read the module docstring in full; it is the `torchrun` manual.** Source of every flag and default in §3: `--rdzv-backend` default **`static`**, `--max-restarts` default **`0`**, `--monitor-interval` default `0.1`, `--nnodes=MIN:MAX` semantics, `--standalone`, `--rdzv-conf`, `--tee` / `--redirects` / `--log-line-prefix-template`; the full injected-environment list (`RANK`, `LOCAL_RANK`, `WORLD_SIZE`, `LOCAL_WORLD_SIZE`, `GROUP_RANK`, `ROLE_RANK`, `ROLE_WORLD_SIZE`, `MASTER_ADDR`, `MASTER_PORT`, `TORCHELASTIC_RESTART_COUNT`, `TORCHELASTIC_MAX_RESTARTS`, `TORCHELASTIC_RUN_ID`); the backend list (`c10d` recommended, `etcd-v2`, `etcd` in maintenance mode); the failure-mode and membership-change semantics quoted in §4–§6; and the explicit warning not to write logic depending on `WORLD_SIZE` or on a `RANK`/`LOCAL_RANK` correlation. **Correction:** the previous version of this lesson implied elastic behaviour was the default; it is not — both `--rdzv-backend=c10d` and a non-zero `--max-restarts` must be passed explicitly. *`docs.pytorch.org` is blocked by this environment's egress proxy; verified against the in-repo Python at HEAD.*

2. **PyTorch — `torch/distributed/elastic/rendezvous/dynamic_rendezvous.py`** (same repository). Source of the rendezvous state machine in §4: `_RendezvousState` (participants, `wait_list`, `last_heartbeats`, `complete`, `round`), the `RendezvousTimeout._DEFAULT_TIMEOUTS` of **join 600 s, last_call 30 s, close 30 s, heartbeat 5 s**, and the `keep_alive_interval = 5` / `keep_alive_max_attempt = 3` defaults that give the ~15 s node-death detection window. Verified in-source.

3. **PyTorch — `torch/distributed/elastic/agent/server/api.py` and `local_elastic_agent.py`** (same repository). Source of the agent architecture and monitor loop in §3 and §5: the `WorkerSpec.monitor_interval` default of `0.1 s`, the `_invoke_run` loop that restarts the worker group on `UNHEALTHY`/`FAILED` and on `num_nodes_waiting() > 0`, the explicit note that **membership changes do not count as retries**, `_restart_workers` stopping *all* local workers before re-initialising, and the `exit_barrier_timeout` default of 300 s. Verified in-source.

4. **PyTorch — `torch/csrc/distributed/c10d/ProcessGroupNCCL.cpp` / `.hpp`** (same repository). Source of the detection-latency figures in §2: `kProcessGroupNCCLDefaultTimeout` = **600,000 ms**, `TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC` = 480, `TORCH_NCCL_ASYNC_ERROR_HANDLING` = 3 (already on). The full treatment is in 08.2; reproduced here because these defaults set the slow path of `T_detect`. Verified in-source.

5. **NVIDIA — NCCL user guide, `docs/userguide/source/troubleshooting/ras.rst`** — <https://github.com/NVIDIA/nccl>. Source of the RAS row in §2's detection table: `ncclras` answers a `STATUS` query in seconds and promotes an unresponsive process from `INCOMPLETE` to **`DEAD` after 60 seconds** of failed reconnection. RAS is a query tool, not a trigger — a supervisor must poll it. Verified against the in-repo reStructuredText at NCCL v2.31.2; *`docs.nvidia.com` is blocked by this environment's egress proxy.*

6. **Meta — `meta-pytorch/torchft`** — <https://github.com/meta-pytorch/torchft>. **Read the README and `src/lighthouse.rs`'s `LighthouseOpt`.** Source of the architecture and every default in §8: the Lighthouse coordination server (`--bind [::]:29510`, `--join_timeout_ms 60000`, `--quorum_tick_ms 100`, `--heartbeat_timeout_ms 5000`), the per-replica-group `Manager` with `heartbeat_interval` **100 ms** and 60 s `timeout`/`quorum_timeout`/`connect_timeout`, the fault-tolerant `ProcessGroup` that "reports errors sanely and can be reinitialized gracefully", the **checkpoint transports for live recovery from a healthy peer**, and the supported algorithms (fault-tolerant DDP, fault-tolerant HSDP, LocalSGD, DiLoCo). Verified by cloning at HEAD.

**Real-world engineering blogs**

7. **Meta — "Training LLMs with Fault Tolerant HSDP on 100,000 GPUs"** — <https://arxiv.org/abs/2602.00277> (Salpekar et al., January 2026). FT-HSDP's design — data-parallel replicas as the fault-tolerance unit, the Fault Tolerant All Reduce (FTAR) protocol driving control logic from the CPU while the GPU moves data, and a non-blocking catch-up protocol — and the headline result: **stall time from failure recovery ~10 min → ~3 min, effective training time 44% → 80%** at O(100K) GPUs, with no meaningful accuracy degradation. *`arxiv.org` is blocked by this environment's egress proxy; the authorship, design summary and both numbers were confirmed via search snippets of the paper's abstract, not by reading the PDF in this pass.*

8. **Meta — "Revisiting Reliability in Large-Scale Machine Learning Research Clusters"** — <https://arxiv.org/abs/2410.21680> (Kokolis et al., HPCA 2025). 11 months of data, 4 M jobs, >150 M A100-GPU-hours across two production clusters; large jobs are individually most failure-vulnerable while small jobs dominate by count. *`arxiv.org` blocked here; cited for its scope and headline finding as reported, **not** relied upon for any specific rate or percentage in this lesson.*

9. **PyTorch blog — "Fault Tolerant Llama: training with 2,000 synthetic failures every ~15 seconds and no checkpoints on Crusoe L40S"** (June 2025) — <https://pytorch.org/blog/fault-tolerant-llama-training-with-2000-synthetic-failures-every-15-seconds-and-no-checkpoints-on-crusoe-l40s/>. The official torchft validation run: 2,000+ synthetic node failures on 300 L40S GPUs with no stop-the-run checkpoint restarts and a converging loss. *`pytorch.org` is blocked by this environment's egress proxy; the run's existence and framing are corroborated by `stas00/ml-engineering`'s fault-tolerance chapter and by the torchft repository, and it is cited as a pointer rather than relied upon for a measured number.*

10. **`stas00/ml-engineering` — `training/fault-tolerance/README.md`** — <https://github.com/stas00/ml-engineering>. **Read in full; it is the best practitioner companion to this lesson.** Source of the operational recipes in Real-world use cases: keeping 5–10% spare nodes and verifying the scheduler auto-drains bad ones; `sbatch --array=1-10%1` to queue a serial job array so a crash immediately starts the next job; the `drained`/`alloc`/`idle` Slurm state model; preferring fixed accelerator allocations because dynamic pools return other users' rejected nodes; the kill-switch and save-switch patterns from BLOOM-176B; and the **is-job-hanging watchdog** argument that a hang outside `torch.distributed` has no timeout and must be caught by checking log-file mtime. Also its summary of torchft, Oobleck, Bamboo and Varuna as the multi-replica fault-tolerance family. Verified by cloning at HEAD.

11. **Meta — OPT-175B training chronicles** — <https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles>. A day-by-day public logbook of incidents, restarts and manual interventions on a live 992-GPU run — at least 35 manual restarts and 100+ hosts cycled over roughly two months. The human-in-the-loop baseline for `R`.

12. **The Llama 3 Herd of Models, §3.3** — <https://arxiv.org/abs/2407.21783>. Used here for the ~78% hardware-attributed share of interruptions (and therefore the ~22% that hardware-health monitoring cannot see), and for the per-GPU MTBF of ~50,600 GPU-hours implied by 419 unexpected interruptions over 54 days on 16,384 GPUs, which every scale calculation in this lesson uses. *`arxiv.org` blocked here; figures confirmed via search snippets of the paper and contemporaneous technical reporting, not by reading the PDF. Derivation in 08.4 §6.*

**Deeper dives**

13. **AMD ROCm blog — "Resilient Large-Scale Training: Integrating TorchFT with TorchTitan on AMD GPUs"** — <https://rocm.blogs.amd.com/artificial-intelligence/primus-torchft/README.html>. A hardware vendor's own integration of torchft with torchtitan on AMD's Primus-SaFE Kubernetes platform for checkpoint-less training — evidence the pattern has reach beyond its origin team. *Not fetched in this pass; listed as optional depth and **not relied upon** for any claim above.*

14. **Nebius — "Fault-tolerant training: how we build reliable clusters for distributed AI workloads"** — <https://nebius.com/blog/posts/how-we-build-reliable-clusters>. A neocloud's account of idle-GPU cost during repair and its mitigations, including a reported mean time to repair. *Not fetched in this pass; listed as optional depth and **not relied upon** — the previous version of this lesson quoted a ~12-minute average MTTR from it, which has been removed rather than repeated unverified.*

> **Snapshot (2026-08).** GPU counts, $/GPU-hour and per-GPU MTBF figures used in the arithmetic above are dated snapshots. The relations — `M_job = m_gpu/N`, `R = T_detect + … + T_warmup`, `goodput = 1 − √(2C/M) − R/M` — outlive any of them. Re-measure before quoting a number in a review.
