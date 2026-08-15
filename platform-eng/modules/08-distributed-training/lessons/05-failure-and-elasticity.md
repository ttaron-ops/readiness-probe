---
lesson: "08.5"
title: "Failure and elasticity"
module: "08"
concept: "Failure and elasticity"
status: not-started
est_time: "7h"
prev: "04-checkpointing.md"
next: "06-job-orchestration.md"
artifacts: []
sources: 13
---
# 08.5 · Failure and elasticity
> **Concept.** A large training run is a single distributed process that dies when any one rank dies; elasticity is the loop — health signal → drain → re-rendezvous → resume-from-checkpoint — that turns a fatal crash into a bounded pause instead of a restart from zero.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

04 gave you the checkpoint — the durable floor you fall back to, and the Young/Daly math
that says how often to write it. This lesson is what *triggers* the fall-back: the
detector, the drain, and the machinery that re-forms the process group around the last
checkpoint. 06, next, is the object that expresses this on Kubernetes — the CRD whose
`restartPolicy` and headless Service are the plumbing underneath the causal chain you
learn here. Read in order, 04 → 05 → 06 is "how expensive is a restart" → "what triggers
and executes a restart" → "what Kubernetes object that restart actually lives inside."

## Why this matters

A synchronous data-parallel run is one process with N ranks joined at the hip by a
collective. `all_reduce` is a barrier: every rank contributes its gradient shard and
every rank blocks until all shards arrive. So the failure semantics are brutal and
non-negotiable — **if one rank dies mid-collective, the other N−1 do not keep going.
They block on a NCCL operation that can never complete, and eventually die on a
timeout.** There is no partial progress, no "skip the dead node's gradient." The whole
world stops.

Now put a number on it. At 40+ clusters you already reason about MTBF; here the unit
that fails is a *GPU*, and a run holds thousands of them for weeks. If a single
accelerator's mean time to failure is, say, ~50,000 GPU-hours (optimistic — real fleets
see XID storms, ECC faults, NVLink flaps, and thermal trips well below that), then a
2,000-GPU run expects a hardware-caused interruption roughly **every ~25 hours**. Meta's
Llama 3 405B paper describes exactly this reality: the 405B run over 16k H100s saw a
failure roughly every ~3 hours, ~78% of them hardware. The consequence is the thesis of
this lesson: **on large runs, failure handling dominates effective throughput.** Your
"goodput" — fraction of wall-clock spent making forward progress — is set less by FLOPs
than by how fast you detect a dead node, evict it, re-form the world, and reload the
last checkpoint. That is a platform-engineering problem, not an ML problem, and it is
exactly the differentiator a CoreWeave/Anthropic platform team is hiring for.

## What's new here (building on 04, 05, 06)

This lesson wires together three things you already have and adds the missing loop:

- **04 gave you the checkpoint.** You know async/sharded checkpointing and that the last
  durable checkpoint is the *floor* you fall back to. This lesson is what *triggers* the
  fall-back and what re-forms the process group around it.
- **05 (module 05) gave you the health signal.** You know XID codes and DCGM health. Here
  that signal becomes an *actuator*: an XID 79 (GPU fell off the bus) or XID 48
  (double-bit ECC) or a `DCGM_FI_DEV_XID_ERRORS` reading is the interrupt that fires the
  drain. **We do not re-teach XID codes — module 05 owns those.** 08.5 owns what you *do*
  with them.
- **06 (module 06) placed the gang.** Gang scheduling and topology-aware placement got N
  pods co-scheduled all-or-nothing. The clean division of labor: **module 06 places the
  gang; 08 keeps it alive; module 05's XID is the signal that triggers a restart.** When
  a node dies, module 06's gang semantics are also what guarantee the *replacement* comes
  back as a whole group rather than a partial, deadlocked world.

New mechanism this lesson owns: **rendezvous** (how ranks discover each other and agree
on a world), **elastic restart** (torchrun re-forming that world after membership
changes), and **fault-tolerant training** (torchft removing the full-restart entirely).

## Core concepts

### The failure loop, end to end

```
  [mod.05] XID / DCGM health  ─────► detector (node-problem-detector / DCGM health watch)
        (the signal)                     │  taints/cordons the node
                                         ▼
                                   drain the pod  ─────►  [mod.06] gang controller evicts
                                         │                the group or the single pod
                                         ▼
                          torchrun agents detect worker exit
                                         │
                                         ▼
                              RE-RENDEZVOUS (c10d/etcd)  ── surviving agents re-form
                                         │                  the world at new WORLD_SIZE
                                         ▼
                       every rank restarts the *training script*  ─────►  [04] load
                                         │                                 last checkpoint
                                         ▼
                                   resume forward progress
```

Three properties to internalize. **(1)** The restart is *not* process-level surgery —
torchrun kills and re-launches every worker in the group, so your script must be written
to resume from checkpoint on start (idempotent boot). **(2)** The unit torchrun restarts
is the *worker group*, not the single dead rank; all survivors reboot too. **(3)** The
detector→drain half of the loop is Kubernetes/infra plumbing you own; the
re-rendezvous→resume half is torchrun's job.

### torchrun elastic — the mechanism

`torchrun` (the console entry point for `torch.distributed.run`, successor to
`torch.distributed.launch`) runs an **elastic agent** on every node. The agent owns the
local workers; the agents collectively own the rendezvous. The elastic knobs:

```bash
torchrun \
  --nnodes=2:8 \                    # MIN:MAX — elastic range, not a fixed count
  --nproc-per-node=8 \              # workers (GPUs) per node
  --max-restarts=3 \                # restarts of the WORKER GROUP before giving up
  --rdzv-backend=c10d \             # rendezvous backend: c10d (built-in) or etcd
  --rdzv-endpoint=$HEAD:29400 \     # host running the rendezvous store
  --rdzv-id=job-4417 \             # unique run id — the rendezvous "room" name
  train.py --resume-from=/ckpt/latest
```

- **`--nnodes=MIN:MAX`** is what makes it elastic. With `--nnodes=2:8`, the job *starts*
  once 2 nodes have joined and *tolerates* losing nodes down to 2. A fixed `--nnodes=8`
  is the non-elastic case: any loss is fatal.
- **`--rdzv-backend`**: `c10d` is the built-in TCP store — one node hosts it (the
  `--rdzv-endpoint`), zero external dependencies, and is the current default recommendation.
  `etcd` (v2 API, `etcd-v2`) is the older external-quorum backend; prefer `c10d` unless
  you specifically need etcd's HA store. **Note the failure asymmetry of c10d:** the node
  hosting the store is a soft SPOF for *forming* a new rendezvous — if it dies, in-flight
  workers survive but a subsequent re-rendezvous has no host. Production picks a stable
  head or uses etcd for that reason.
- **`--max-restarts`** bounds the worker-group restarts. Exhausting it fails the job —
  this is the backstop against crash-looping on a poisoned checkpoint or a persistently
  bad node.
- **Environment the agent injects** into every worker (you do *not* set these by hand):
  `RANK`, `LOCAL_RANK`, `WORLD_SIZE`, `LOCAL_WORLD_SIZE`, `MASTER_ADDR`, `MASTER_PORT`,
  `GROUP_RANK`. On elastic runs **`RANK` and `WORLD_SIZE` are not stable across restarts** —
  after a node leaves, the world shrinks and ranks are reassigned. Any code that pins
  behavior to "I am rank 0 forever" is a bug. Read them fresh from the env after each
  `dist.init_process_group()`, every restart.

### Re-rendezvous — re-forming the world after a node loss

When a worker exits abnormally, its local agent sees the exit and signals the rendezvous
that membership changed. This kicks all agents into a **new rendezvous round**:

1. **Barrier + membership.** Surviving agents rejoin the rendezvous "room" (`rdzv-id`).
   The backend gathers who is present within a timeout, then closes the round.
2. **World assignment.** The backend computes the new `WORLD_SIZE`, assigns fresh global
   `RANK`s, and elects the rank-0 / `MASTER_ADDR`. This is the "generation" bump — every
   rendezvous round has a monotonically increasing generation number so stale workers
   can't poison the new world.
3. **Restart workers.** Each agent SIGTERMs its local workers and re-launches
   `train.py` from scratch *with the new env*. Your script re-runs
   `init_process_group()`, which does the NCCL handshake and builds fresh communicators
   for the smaller world.
4. **Resume from checkpoint.** Because the script restarted, in-memory state (weights,
   optimizer, step) is gone. The script's boot path loads **04's last durable
   checkpoint** and continues. The work lost is exactly `(now − checkpoint_time)` of
   compute — which is why 04's checkpoint *interval* is the knob that bounds worst-case
   loss, and why async checkpointing (04) exists to make that interval short without
   stalling the GPUs.

The re-rendezvous *time* — barrier timeout + NCCL re-init + checkpoint reload — is dead
wall-clock, typically tens of seconds to a few minutes depending on world size and
checkpoint size. That number, times failure frequency, is your goodput tax. It is the
thing you measure and drive down.

### The trigger, tied to module 05

In production the human is not watching. The loop is armed by module 05's telemetry:

- **DCGM health checks** (`dcgm_health_check`) and the XID error counter surface a GPU
  fault. Node-problem-detector or a DCGM watcher translates that into a **node
  condition** and a **taint**.
- The taint triggers a **cordon + drain**; the training pod is evicted. Module 06's gang
  controller ensures the replacement rejoins as a whole group.
- torchrun's agent, seeing the worker gone, does the re-rendezvous above.

So the causal chain a review will ask you to recite: **XID/DCGM (mod. 05) → node
taint/drain (K8s) → pod evict (mod. 06 gang semantics) → re-rendezvous (torchrun) →
resume from checkpoint (04).** Each module owns one arrow.

### torchft — the frontier: fault tolerance without the full restart

torchrun elastic still does a **stop-the-world restart**: every survivor reboots and
reloads a checkpoint. `torchft` (Meta's `meta-pytorch/torchft`) removes that. Key ideas:

- **Per-step heartbeat.** A **Lighthouse** server + per-replica-group **Manager**
  determine which workers are healthy by heartbeating *every training step*, not by
  waiting for a NCCL timeout. Detection latency drops from a minutes-long collective
  timeout to one step.
- **Fault-tolerant `ProcessGroup`.** A wrapped process group that reports errors sanely
  and **reinitializes gracefully** — a failed collective becomes a recoverable error, not
  a dead process. Healthy replica groups keep training through the membership change.
- **Live recovery from a healthy peer.** Instead of every survivor reloading from durable
  storage, a recovering/scaling replica **pulls current weights over a checkpoint
  transport from a healthy peer** — recovery at step granularity, no cold restart.
- Targets **HSDP / DDP with replica groups**, and the LocalSGD/DiLoCo semi-synchronous
  algorithms where replica groups sync infrequently — the regime where per-group fault
  tolerance buys the most. The project is actively developed (see e.g. GitHub issue
  [#171](https://github.com/meta-pytorch/torchft/issues/171), broadcast-based fault
  tolerance for semi-synchronous training) — name it as *direction*, not the universally
  deployed default. torchrun elastic is what most shops run today; torchft is where
  hyperscaler training loops are going, and at frontier scale it is already load-bearing
  (see Meta's FT-HSDP below).

**Framing for interviews:** *elastic* training = the world can change size and the job
survives via restart (torchrun). *Fault-tolerant* training = individual failures are
absorbed without a full-world restart (torchft). Elasticity is about membership;
fault tolerance is about continuity. This distinction is not academic — it is the
difference between a 44%-effective and an 80%-effective training loop at 100K-GPU scale
(see Perspectives, below).

## Perspectives

**Systems/mechanism view.** The core of this lesson is the detector → drain →
re-rendezvous → resume loop and the crisp elastic-vs-fault-tolerant distinction —
membership change with restart, vs continuity without restart. This is the conceptual
spine; everything else in the lesson is elaboration on it.

**Statistical/fleet-operator view.** Meta's FAIR team analyzed 11 months of data across
two production ML clusters — 4 million jobs, >150M A100-GPU-hours — and found that while
large jobs are the *most vulnerable* to failure, small jobs *dominate the population by
count* (Kokolis et al., "Revisiting Reliability in Large-Scale Machine Learning Research
Clusters," HPCA 2025). That reframes the problem: it is not enough to build an elaborate
recovery loop for the one dramatic 16K-GPU mega-run; the reliability system has to work
uniformly across a much larger population of small, less-visible jobs too, which is a
genuinely different lens than the Llama-3 single-run narrative that anchors this module.

**Frontier/research-lab view.** Meta's FT-HSDP work ("Training LLMs with Fault Tolerant
HSDP on 100,000 GPUs," 2026) shows *why* hyperscalers are moving past torchrun-elastic's
stop-the-world restart toward partial-world fault tolerance: at O(100K) GPUs, restarting
the entire world for a single failed GPU is prohibitively expensive, so they restart only
the data-parallel replica containing the failure. Reported result: stall time from
failure recovery drops from ~10 minutes to ~3 minutes, and effective training time rises
from **44% to 80%**. The restart-cost-scales-with-N argument this lesson makes is exactly
what motivates that architecture.

**Economics view.** Every minute of re-rendezvous/restart time, multiplied across
thousands of GPUs, is a directly priceable cost. This lesson is where the *mechanism*
lives — detection latency, barrier time, NCCL re-init, checkpoint reload — that the
module's capstone (08.8, training economics) turns into a dollar figure. Drive down the
mechanism and you drive down the bill.

## Real-world use cases

- **Meta — "Training LLMs with Fault Tolerant HSDP on 100,000 GPUs"**
  ([arxiv.org/abs/2602.00277](https://arxiv.org/abs/2602.00277)). FT-HSDP restarts only
  the data-parallel replica containing a failed GPU, not the whole world — stall time
  from failure recovery drops from ~10 min to ~3 min, raising effective training time
  from **44% → 80%** at O(100K)-GPU scale. What it shows: the natural bridge from
  torchrun-elastic's stop-the-world recovery to torchft-style partial-world recovery,
  with a quantified, dated (2026) production number.
- **Meta — "Revisiting Reliability in Large-Scale Machine Learning Research Clusters"**
  ([arxiv.org/abs/2410.21680](https://arxiv.org/abs/2410.21680), HPCA 2025). 11 months of
  data, 4M jobs, >150M A100-GPU-hours across two production ML clusters. What it shows: a
  rigorous large-N empirical failure taxonomy — large jobs are most failure-vulnerable,
  but small jobs dominate by count — a much broader statistical base than the Llama-3
  single-run anecdote, good for capacity-planning framing.
- **Meta — OPT-175B training chronicles**
  ([github.com/facebookresearch/metaseq](https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles)).
  A day-by-day public logbook of the incidents, restarts, and manual interventions during
  the OPT-175B run. What it shows: what elastic/fault-tolerant recovery looked like
  before torchft existed — mostly human-triggered restart-from-checkpoint, which is
  exactly the "cost of the old way" baseline this lesson's mechanism replaces.
- **AMD ROCm blog — "Resilient Large-Scale Training: Integrating TorchFT with TorchTitan
  on AMD GPUs"** (rocm.blogs.amd.com, Feb 2026). Integrates torchft with TorchTitan on
  AMD's Primus-SaFE Kubernetes platform for checkpoint-less training. What it shows:
  torchft isn't a Meta/NVIDIA-only artifact — a hardware vendor's own integration
  writeup demonstrating the pattern's reach beyond its origin team.

## Worked example

**Scenario.** A 4-node × 8-GPU DDP run, step ~1.2 s, checkpoint every 500 steps
(~10 min). At step 7,300 a node hits XID 79 (GPU fell off the bus).

**Without elasticity / without checkpointing.** The dead rank's NCCL peers block on the
next `all_reduce`; after the ~10-min (default 600 s) `NCCL_TIMEOUT`/watchdog, they abort.
The job crashes. With no checkpoint, you restart from **step 0** — 7,300 steps × 1.2 s ≈
**2.4 hours of compute, gone**, plus the queue wait to re-acquire 32 GPUs.

**With torchrun elastic + 04 checkpointing (`--nnodes=3:4`, `--max-restarts=3`).** The
agent on the dead node reports the exit almost immediately. Survivors re-rendezvous into
a 3-node world (WORLD_SIZE 32→24, ranks reassigned), re-init NCCL (~20 s), and reload the
step-7,000 checkpoint (~30 s for a sharded async checkpoint). Lost work = steps
7,000→7,300 = **300 steps ≈ 6 minutes**, plus ~1 min re-rendezvous. Total pause **~7
min** vs **2.4 h + queue**. When a node is repaired and rejoins, the world scales back to
4 and the same re-rendezvous machinery grows it.

The delta between those two runs — **~7 minutes vs ~2.5 hours** — *is* the deliverable.
At 100K-GPU scale (see Perspectives), the FT-HSDP number shows the next rung of that same
ladder: **~7 min** (elastic restart) vs **~3 min** (fault-tolerant partial restart) is the
delta that matters once the world itself is huge enough that even a "cheap" 7-minute
stop-the-world restart burns tens of thousands of GPU-hours.

## Practice — feeds the "Survive-a-failure" deliverable

See [`../practice/survive-a-failure/README.md`](../practice/survive-a-failure/README.md)
for the full deliverable spec. This lesson's slice: run the same DDP toy job (the
checkpointing job from 04) two ways and quantify the gap.

**Run A — no safety net.**
1. Launch a multi-worker DDP job with a *fixed* `--nnodes` and **no** checkpoint load on
   boot (or delete the checkpoint dir).
2. `kill -9` one worker process mid-run (or `docker kill` / delete one pod).
3. Observe: survivors hang on the collective, then die on watchdog timeout. Restart.
4. **Measure:** steps completed before kill, and that you resume from **step 0**. Record
   "lost work = all of it."

**Run B — elastic + checkpointing.**
1. Relaunch with `--nnodes=MIN:MAX`, `--rdzv-backend=c10d`, `--max-restarts=3`, and a
   boot path that loads the latest 04 checkpoint.
2. Kill one worker mid-run at a known step.
3. Watch the agent logs re-rendezvous: new generation, new `WORLD_SIZE`, reassigned
   `RANK`s. Confirm the script reloads the last checkpoint and continues.
4. **Measure:** (i) re-rendezvous wall-time (kill → collective resumes), (ii) resume
   step vs kill step = work lost to checkpoint interval.

**Acceptance (deliverable):** a killed-worker recovery demo, *with vs without*, that
reports three numbers — lost-work without checkpointing, re-rendezvous time with
elasticity, and resume-from-checkpoint work-lost — and states the goodput difference in
plain terms (e.g. "2.4 h → 7 min"). Bonus: extract the new `WORLD_SIZE`/`RANK` from
worker logs to prove the world actually re-formed.

## Common pitfalls

- **"Elastic training and fault-tolerant training are the same thing."** They aren't.
  Elastic (torchrun) survives via a full worker-group restart; fault-tolerant (torchft)
  survives without one. The FT-HSDP 44%→80% number only makes sense once this distinction
  is clear — a full restart is a fundamentally more expensive recovery mode than
  membership-preserving recovery.
- **"c10d rendezvous has no single point of failure since it's built-in."** The node
  hosting the c10d store is a soft SPOF for *forming a new* rendezvous round — if it dies,
  a subsequent re-rendezvous has nowhere to go. Production deployments pick a stable head
  node or use etcd specifically because of this.
- **"RANK and WORLD_SIZE are stable identifiers you can build logic around."** On elastic
  runs they are reassigned after every re-rendezvous. Code that pins behavior to "I am
  rank 0 forever" is a real, common bug class — read them fresh from the environment
  after every `init_process_group()` call.
- **"Failure rate is roughly constant per GPU regardless of scale."** MTBF for the *job*
  scales inversely with GPU count (M ~ 1/N_GPUs). That's exactly why 100K-GPU labs (Meta's
  FT-HSDP work) invest in partial-restart architectures that would be over-engineering for
  a 100-GPU job — the math only bites at scale.
- **"Hardware failures are the whole story."** Llama-3's number is ~78% hardware-caused,
  meaning ~22% is *not* — software bugs, data-loader stalls, host OOMs (module 05's own
  worked example) are a meaningful non-hardware slice that pure hardware-health
  monitoring (XID/DCGM) won't catch on its own.

## Self-check

- **What happens to the surviving ranks when one rank dies mid-collective?**
  **Answer:** Nothing good on their own. A synchronous collective (`all_reduce`) is a
  barrier — the survivors block waiting for the dead rank's contribution, which never
  arrives, and after the NCCL watchdog/`NCCL_TIMEOUT` (default ~10 min) they abort too.
  There is no partial progress: the whole world stalls and then dies. Elasticity exists
  precisely because the default outcome is total collapse — torchrun's agents detect the
  exit and force a re-rendezvous into a smaller world instead of waiting for the timeout.

- **What triggers an automated restart in production (tie to module 05)?**
  **Answer:** A GPU health signal from module 05 — an XID error surfaced via
  `DCGM_FI_DEV_XID_ERRORS` / a DCGM health check (e.g. XID 79 fell-off-the-bus, XID 48
  double-bit ECC). Node-problem-detector or a DCGM watcher turns that into a node
  condition and taint; the taint cordons and drains the node; module 06's gang semantics
  evict/replace the group; torchrun's agent re-rendezvous the survivors. The chain:
  **XID/DCGM (mod. 05) → taint → drain → gang evict (mod. 06) → re-rendezvous (torchrun)
  → resume from checkpoint (04).**

- **Elastic training vs fault-tolerant training — what's the difference?**
  **Answer:** Elastic (torchrun): the world can change size and the job survives by
  **restarting the worker group** — every survivor reboots, re-rendezvous, and reloads a
  checkpoint. It's about *membership*. Fault-tolerant (torchft): individual failures are
  absorbed **without a full-world restart** — per-step heartbeats detect the failure, a
  fault-tolerant ProcessGroup reinitializes gracefully, and recovering replicas pull live
  state from a healthy peer. It's about *continuity*. Elastic bounds the cost of a
  restart; fault-tolerant removes the restart.

- **Why does a study across 4M jobs and >150M A100-GPU-hours (Meta's reliability paper)
  matter more than the single Llama-3 405B anecdote for capacity planning?**
  **Answer:** The Llama-3 number describes one mega-run; the Meta reliability study shows
  the fleet-wide distribution — large jobs are individually most failure-vulnerable, but
  small jobs vastly outnumber them, so a reliability system tuned only for the dramatic
  16K-GPU outage will under-serve the much larger population of smaller jobs a real
  platform team actually operates day to day.

- **Why does FT-HSDP's 44%→80% number only make sense once you understand the
  elastic-vs-fault-tolerant distinction?**
  **Answer:** 44% is roughly what stop-the-world elastic restart (torchrun-style, full
  checkpoint reload) delivers at O(100K)-GPU scale once failure frequency scales with
  GPU count; 80% comes from restarting only the failed replica (torchft/FT-HSDP-style)
  instead of the whole world. Without the membership-vs-continuity distinction, the jump
  looks like a generic "optimization" rather than what it is: swapping the recovery
  *mechanism* itself, not just tuning its parameters.

## Connections & what's next

This lesson is the seam between "how expensive is a restart" (04's checkpoint math) and
"what object on Kubernetes actually restarts" (06's PyTorchJob/TrainJob). The causal
chain — XID/DCGM (mod. 05) → taint/drain → gang evict (mod. 06) → re-rendezvous
(torchrun) → resume from checkpoint (04) — is the single most interview-probed sequence
in this module; be able to recite it and name which module owns each arrow. Next, 06
shows you the CRD and controller that generate the rendezvous env this lesson assumed
existed, and names the two restart layers (pod-level vs torchrun-level) you must not
double-count in an incident.

## References & further reading

**Primary sources**
- PyTorch — ["Fault-tolerant Distributed Training with torchrun"](https://docs.pytorch.org/tutorials/beginner/ddp_series_fault_tolerance.html) — the hands-on tutorial: snapshot-on-boot pattern and killing a worker to watch it resume.
- PyTorch — [`torchrun` (Elastic Launch) reference](https://docs.pytorch.org/docs/stable/elastic/run.html) — `--nnodes MIN:MAX`, `--rdzv-backend` (c10d/etcd), `--max-restarts`, injected env, rendezvous semantics.
- Meta — [`meta-pytorch/torchft`](https://github.com/meta-pytorch/torchft) — per-step heartbeat, fault-tolerant ProcessGroup, Lighthouse/Manager, live peer recovery; confirmed actively maintained (see [issue #171](https://github.com/meta-pytorch/torchft/issues/171)).
- PyTorch blog — ["Fault Tolerant Llama: training with 2000 synthetic failures every ~15 seconds and no checkpoints on Crusoe L40S"](https://pytorch.org/blog/fault-tolerant-llama-training-with-2000-synthetic-failures-every-15-seconds-and-no-checkpoints-on-crusoe-l40s/) — the official torchft stress-test/validation writeup (June 2025): 2,000+ synthetic node failures on 300 L40S GPUs, zero stop-the-run checkpoint restarts, converging loss.

**Real-world engineering blogs**
- Meta — ["Revisiting Reliability in Large-Scale Machine Learning Research Clusters"](https://arxiv.org/abs/2410.21680) (HPCA 2025) — the deepest, most rigorously-sourced failure-taxonomy document available; 4M jobs, >150M A100-GPU-hours.
- Meta — ["Training LLMs with Fault Tolerant HSDP on 100,000 GPUs"](https://arxiv.org/abs/2602.00277) — FT-HSDP design and the 44%→80% effective-time number; read in full.
- Nebius — ["Fault-tolerant training: How we build reliable clusters for distributed AI workloads"](https://nebius.com/blog/posts/how-we-build-reliable-clusters) — a neocloud's own account of idle-GPU cost during repair and their mitigation (reports ~12-min average MTTR); good second-source perspective alongside CoreWeave (06).
- AMD ROCm blog — ["Resilient Large-Scale Training: Integrating TorchFT with TorchTitan on AMD GPUs"](https://rocm.blogs.amd.com/artificial-intelligence/primus-torchft/README.html) — torchft beyond its origin team.
- Meta — [OPT-175B training chronicles](https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles) — the pre-torchft baseline: manual restart/recovery narrative from a real multi-month run.

**Deeper dives**
- > **Snapshot (2026-08).** Dollar/hardware figures above (Nebius MTTR, GPU counts) are dated snapshots — verify current numbers before quoting them in a review.
