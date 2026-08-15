---
lesson: "08.2"
title: "NCCL collectives: topology, transport, and the silent hang"
module: "08"
concept: "NCCL collectives: topology, transport, and the silent hang"
status: not-started
est_time: "9h"
prev: "01-parallelism-strategies.md"
next: "03-communication-bottleneck.md"
artifacts: []
sources: 10
---

# 08.2 · NCCL collectives: topology, transport, and the silent hang

> **Concept.** NCCL is the layer that turns "which collective over which link" into real wire traffic — and its signature failure is a job that hangs at 100% GPU utilization with *no error*, which you debug by reading its topology/transport logs and, on 2.24+, querying the RAS subsystem.
>
> Module: [🧮 08 — Distributed training infrastructure](../README.md) · Deliverable: [Survive-a-failure lab](../practice/survive-a-failure/README.md)

## Where this fits

This is the module's **anchor lesson** — lesson 2 of 8, and the one the module README names as its centerpiece ("06 places the gang; 08 keeps it alive; 05's XID is the signal that triggers a restart"). Everything in 08.1 (which collective a strategy issues) becomes concrete here: NCCL is the library that actually runs those collectives, picks the transport, and — when something goes wrong — produces the module's defining incident shape, the silent hang. Module 08.3's MFU math, 08.4's checkpoint sizing, and 08.5's failure/elasticity design all assume you can already do the triage this lesson teaches.

## Why this matters

This is the single most valuable debugging skill in distributed training, and the one that most cleanly separates a senior platform engineer from everyone else on the incident bridge. A NCCL collective is a **synchronization barrier**: every rank must arrive or none proceed. So when one GPU on one node out of thousands dies, straggles, or loses its NIC, the collective doesn't error — it *waits*. Every other GPU spins at 100% utilization inside the collective, burning money, emitting no error, until a watchdog timeout fires (often 10–30 minutes later) with a stack trace that points at the *victims*, not the *cause*. In the Llama-3 fleet story this class of event is a large share of the ~1 interruption/3hrs; the teams that keep >90% effective training time are the ones who can localize a silent hang in minutes instead of hours. If you can read NCCL's topology and transport selection and run the standard hang triage, you own that number.

## What's new here (calibration)

**05** taught you GPU *health*: XID errors, DCGM, when a GPU is physically sick. **02b** taught you the *links*: NVLink domains, rail alignment, GPUDirect. **08.1** (previous lesson) told you *which collective* each parallelism strategy issues.

This lesson is the piece that connects them: **NCCL is the software that, at job startup, discovers the topology from 02b, decides which transport (NVLink / PCIe / shared-memory / InfiniBand / TCP) and which algorithm (ring / tree) to use for each collective from 08.1, and then runs them.** The new platform skill is *reading that decision from the logs* and *triaging a hang the health tools can't see* — because a silent NCCL hang produces no XID (05 is quiet), the GPU is electrically fine, and gang scheduling (06) has done its job placing the pod. 06 places the gang; **08 keeps it alive**. When it hangs, the diagnosis lives entirely in NCCL's own logs and its RAS subsystem, not in DCGM. We skip ML-eng here too — you are reading NCCL's *behavior*, never modifying kernel code or writing custom collectives.

## Core concepts

### The collectives and who issues them

NCCL implements a handful of collectives; from 08.1 you already know which strategy drives each:

| Collective | What it does | Issued by |
|---|---|---|
| **all-reduce** | sum a tensor across all ranks, everyone gets the result | DDP gradient sync; TP per-layer |
| **all-gather** | each rank contributes a shard, everyone gets the full tensor | FSDP parameter materialization |
| **reduce-scatter** | sum across ranks, each keeps only its shard | FSDP gradient sharding |
| **broadcast / reduce** | one-to-all / all-to-one | init, checkpoint load |
| **all-to-all** | each rank sends a distinct chunk to every other | MoE expert routing |

An all-reduce is internally a **reduce-scatter followed by an all-gather** — which is why those three dominate every trace.

### Ring vs tree

For a given collective NCCL picks an **algorithm**, and the two you'll see constantly are:

- **Ring** — ranks form a logical ring; data flows around it in `2(N-1)` steps. Each rank sends/receives `size/N` per step, so it's **bandwidth-optimal for large messages** — total bytes on the wire is independent of N. But latency grows linearly with N (data must traverse every hop). Ring is the default for the big gradient/parameter transfers.
- **Tree** — ranks form a binary tree; a reduce climbs to the root and the result broadcasts back down in `~2·log(N)` steps. **Latency-optimal for small messages and large N** because depth is logarithmic, not linear. NCCL switches to tree for small payloads and very large rank counts.
- **CollNet / NVLS** — hardware-accelerated variants that offload the reduction into the switch (SHARP on InfiniBand, NVLink SHARP/NVLS in the NVSwitch). Faster still where the hardware supports it.

NCCL auto-tunes the choice per message size from its topology model; you override with `NCCL_ALGO` (`Ring`, `Tree`, `CollNet`, `NVLS`, …) and the wire encoding with `NCCL_PROTO` (`LL`, `LL128`, `Simple`). "Ring scales better for large messages, tree wins on latency at scale" is the one-line summary to carry — and note that NCCL's auto-tuner is usually *right*: forcing `NCCL_ALGO` in production because "tree sounds faster" is itself a common self-inflicted slowdown (see pitfalls).

### The transports — which link a collective actually rides

At init, NCCL ranks each peer connection to the **fastest available transport** and logs its choice. In priority order:

1. **P2P over NVLink** — direct GPU-to-GPU over NVLink/NVSwitch (02b). Logged as `via P2P` / `via NVLink`. Fastest.
2. **P2P over PCIe** — same GPUDirect P2P path but over PCIe when there's no NVLink between the pair.
3. **SHM (shared host memory)** — GPUs on the same node with no usable P2P path bounce through pinned host memory. Logged `via SHM`. Slower; often a sign of a topology problem.
4. **NET/IB** — GPUDirect RDMA over InfiniBand/RoCE for cross-node pairs. Logged `via NET/IB`. This is the fabric path from 02b.
5. **NET/Socket** — TCP over Ethernet, the fallback when IB is unavailable or disabled. Logged `via NET/Socket`. **Orders of magnitude slower** — seeing this on a run that should use IB is a top-tier bug.

The lines you're pattern-matching in `NCCL_DEBUG=INFO` look like this (annotated):

```
NCCL INFO Channel 00/0 : 0[0] -> 1[1] via P2P/CUMEM        # NVLink/PCIe direct — good
NCCL INFO Channel 01/0 : 2[2] -> 3[3] via SHM/direct       # host-memory bounce — suspect
NCCL INFO Channel 00/0 : 4[0] -> 12[0] [send] via NET/IB/3/GDRDMA  # cross-node RDMA — good
NCCL INFO Channel 00/0 : 4[0] -> 12[0] [send] via NET/Socket       # TCP fallback — BUG on an IB fleet
NCCL INFO Using network IBext                              # network plugin selected
NCCL INFO comm 0x... rank 4 nranks 512 - Init COMPLETE     # init finished, ranks agree
```

`Init COMPLETE` on every rank is the "startup succeeded" signal; a subset of ranks never reaching it is an init-time hang (usually `NCCL_SOCKET_IFNAME`/bootstrap). `via SHM` where you expected NVLink, or `via NET/Socket` where you expected IB, is the whole ballgame — a slow run's root cause is often one of those two lines. Because these lines describe a *physical fact about the machine* — what's actually wired to what — a "wrong" transport in the log is never noise. It is always diagnostic of a real topology or config problem: a GPU pair you expected to share NVLink but doesn't, an HCA that didn't get selected, a socket interface pointing at the wrong NIC.

### The env vars you actually debug with

- **`NCCL_DEBUG`** — `VERSION` < `WARN` < `INFO` < `TRACE`. `INFO` is your default for any investigation; `WARN` for production. `INFO` prints the topology, transport per peer, algorithm, and ring/tree construction.
- **`NCCL_DEBUG_SUBSYS`** — filters the firehose. Values: `INIT`, `GRAPH`, `NET`, `COLL`, `P2P`, `SHM`, `TUNING`, `ENV`, `ALLOC`, `PROXY`, `NVLS`, `BOOTSTRAP`, `RAS`, `ALL`. Comma-separated.
  - **`INIT`** — comm setup, ranks, and the **transport chosen per peer** (`via NVLink/P2P/SHM/NET`).
  - **`GRAPH`** — the **topology detection and channel/ring/tree graph**: the discovered PCIe/NVLink/NIC connectivity and the paths NCCL computed across it. This is where you confirm NCCL *sees* the hardware correctly.
  - **`NET`** — network plugin and **which IB device / interface** was selected.
  - **`COLL` / `P2P`** — per-collective and per-peer detail.
- **`NCCL_ALGO`** — force `Ring` / `Tree` / `CollNet` / `NVLS`. Diagnostic: pin `Tree` to test whether a ring is stalling on one bad hop.
- **`NCCL_P2P_DISABLE=1`** — turn off GPU-to-GPU P2P (NVLink/PCIe direct); NCCL falls back to SHM or net. Diagnostic for a suspected NVLink/P2P fault; also a big slowdown, so it's a test, not a fix.
- **`NCCL_IB_DISABLE=1`** — turn off the InfiniBand/RoCE verbs transport; NCCL falls back to **TCP sockets**. If the job suddenly *works* (slowly) with this set, your IB path is broken.
- **`NCCL_SOCKET_IFNAME`** — names the interface for the bootstrap and socket transport, e.g. `=eth0` or `=ib0`. Misconfiguration here (picking a docker bridge or a management NIC) is a classic cause of hangs at init and of accidental slow-socket paths.
- **`NCCL_IB_HCA`** — pins which IB HCAs to use (rail alignment, 02b).

All of the diagnostic toggles above (`NCCL_ALGO`, `NCCL_P2P_DISABLE`, `NCCL_IB_DISABLE`) exist to let you *force a different path and see what changes* — they are hypothesis tests, not fixes. Left set in a production launch script after the incident is resolved, they are a silent, usually accidental, performance regression: `NCCL_IB_DISABLE=1` forgotten in an env file means every future run on that fleet quietly falls back to TCP sockets forever.

### The RAS subsystem (NCCL 2.24+) — querying a hung job

Before 2.24, a silent hang gave you nothing to query — you scraped every rank's logs and guessed; this was the old way, and it is materially slower than what follows. **RAS (Reliability, Availability, Serviceability)** changes that. Each NCCL process runs a lightweight RAS thread connected in a background socket network; a **client connects to any process's RAS listener** (address via `NCCL_RAS_ADDR`, default `127.0.0.1:28028`) and queries the *live* state of the whole job **without perturbing it**. It returns a per-communicator, per-rank status table: which ranks are alive, which communicators exist, and — critically — **which ranks are lagging or missing from an in-flight collective**. That turns "3,000 GPUs hung, no error" into "communicator X is waiting on global rank 4831, which is not responding" — pointing you at the *cause*, not the *victims*. RAS is on by default in recent NCCL; disable with `NCCL_RAS_ENABLE=0`. Treat it as the first thing you reach for on a modern hang, before you start diffing per-rank logs. Conceptually, you connect a client to the listener and read the returned table:

```
# connect to any rank's RAS listener (default 127.0.0.1:28028)
# the client ships with NCCL; it prints a per-communicator, per-rank view
Job:  512 ranks, 1 communicator
  communicator 0x...: 511/512 ranks in ALLGATHER
    NOT RESPONDING: global rank 47 (node gpu-093, cuda dev 7)   <-- the cause
```

That one "NOT RESPONDING" line is the entire diagnosis a pre-2.24 hang would have cost you an hour of log-grepping to reconstruct. Note the nuance for **all-to-all / MoE** runs: an imbalanced expert routing can make *one* rank legitimately slow rather than dead — RAS shows it lagging (still progressing) vs missing (stopped), which tells you "straggler, not failure," a different fix (load-balance) than a restart.

### The failure archetype: one dead rank, everyone hangs

The canonical incident: a collective is a barrier, so if **one rank silently dies or straggles** — a GPU wedged (XID would show in 05, but a hang often *precedes* or *dodges* the XID), a NIC flapping, a host OOM-killing the process, a slow disk stalling that rank's data loader — every *other* rank enters the collective and **spins at 100% GPU utilization waiting**. No error is thrown. Dashboards stay green on utilization (a trap you now recognize from PSI in 01b: 100% "busy" is 100% *waiting*). Eventually a watchdog fires:

```
[Rank 12] Watchdog caught collective operation timeout:
WorkNCCL(... OpType=ALLREDUCE ...) ran for 1800000 milliseconds before timing out
```

controlled by `TORCH_NCCL_ASYNC_ERROR_HANDLING` / the PyTorch process-group `timeout` (default 10 min for init, 30 min for collectives). The trap: that message names the ranks that *timed out waiting*, not the one that *left*. Your job is to find the rank that never arrived.

Two PyTorch knobs shape how a hang surfaces, and you should know them because they change your triage window:

- **`TORCH_NCCL_ASYNC_ERROR_HANDLING=1`** — the watchdog tears the process down on timeout instead of leaving it wedged forever, so a hang becomes a *crash with a stack* (restartable by your supervisor) rather than an eternal 100%-util spin. On big fleets you want this on.
- **`TORCH_NCCL_TRACE_BUFFER_SIZE=<N>`** — enables the **flight recorder**: a ring buffer of the last N collectives per rank, dumped on timeout. It tells you exactly which collective each rank was on when it stalled — the "who was where" that RAS gives you live, but captured at the moment of death for post-mortem. Turn it on for any run you expect to debug.

### The standard hang-triage sequence

1. **Confirm it's a hang, not slow progress.** `nvidia-smi` on suspect nodes: 100% util, memory pinned, but power draw often *lower* than active compute (spinning in a comm kernel, not doing matmuls) — and step counter frozen in logs.
2. **Query RAS (2.24+)** to get the live per-rank status and find the rank(s) missing from the in-flight collective. On older NCCL, skip to log correlation.
3. **Localize the odd rank out.** Check 05's signals on that rank/node (XID via `dmesg`/`nvidia-smi -q`, DCGM health), the NIC (`ibstat`, link flaps), and the host (`dmesg` for OOM-kills, the data-loader). Correlate the watchdog's rank/timestamp across nodes to find who stopped incrementing first.
4. Re-run the init with `NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,GRAPH,NET` if the hang is at startup, to confirm transports and interface selection — a socket-picking bug (`NCCL_SOCKET_IFNAME`) or a wrong IB device (`NET` subsys) often hangs the *bootstrap*, before the first step.

## Perspectives

**Developer / user view.** Most ML engineers call `all_reduce` or wrap a model in `fully_shard` and expect it to just work — NCCL's topology discovery and transport/algorithm selection happen invisibly at init, and most ML engineers have never read a `NCCL_DEBUG=INFO` log line in their life. That's not a knock on them; it's the exact gap a platform engineer is there to fill. When their job hangs, they see a frozen progress bar; you're the one who reads the layer underneath.

**Operator / on-call view.** The entire value of this lesson is triage *speed* under pressure. Reading the NCCL docs calmly at your desk and reading them while a 3,000-GPU job burns dollars-per-minute are different skills — the second one is what you're being hired for. RAS (2.24+) is the single biggest recent UX improvement to this workflow: it turns a problem that used to require correlating dozens of log streams by hand into one query. Knowing the pre-RAS "the old way" (manual log correlation) matters too, because not every fleet runs a NCCL new enough to have it, and you need to recognize when you've lost that shortcut.

**Hardware / kernel view.** The transport hierarchy — NVLink P2P > PCIe P2P > SHM > NET/IB > NET/Socket — is a physical fact about the machine, not a preference NCCL guesses at. That means a "wrong" transport in the logs — `SHM` where NVLink should have been available, `Socket` where IB should have been selected — is *always* diagnostic of a real topology or configuration problem. It is never noise to filter out. Treat every unexpected transport line as a lead, not a false positive.

**Failure-mode view.** Collectives are barriers, so NCCL's failure signature is categorically different from a normal process crash: it's a **silent stall**, not an exception. The watchdog timeout — 10 to 30 minutes by default — names the *waiters*, not the culprit. "Everyone reports the symptom, nobody reports the cause" is the core intellectual content of this lesson, and it's why the triage sequence exists as a discipline rather than a single command: you're deliberately working around a system that, by design, tells you who's stuck but not who left.

## Real-world use cases

- **Crusoe Cloud Support — "NCCL Hangs and Multi-Node Training Stalls Caused by Failed nvidia-fabricmanager"** — <https://support.crusoecloud.com/hc/en-us/articles/46061806112155-NCCL-Hangs-and-Multi-Node-Training-Stalls-Caused-by-Failed-nvidia-fabricmanager>. A GPU-cloud provider's own operational support runbook for exactly the "job hangs, no error" scenario: when `nvidia-fabricmanager` dies on even one node, NCCL init stalls indefinitely with no clear error, typically completing channel/tree setup and then hanging. The runbook's own triage: `journalctl -u nvidia-fabricmanager`, checking `/var/log/syslog`, and `nvidia-smi nvlink -s` to confirm NVLink topology health. *What it shows:* a genuinely different root cause (fabric-manager service death, not a dead rank or bad NIC) than the worked example below — it widens the "who's the dead rank" hunt to include host services you wouldn't otherwise think to check.
- **Meta — OPT-175B chronicles** — <https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles>. A primary-source hang/failure diary from a live 992-GPU run: real NCCL/InfiniBand errors ("data packets lost or corrupted in transit"), catastrophic "GPU is lost" events, and at least 35 manual restarts over two months — not sanitized after the fact. *What it shows:* what the triage sequence above looks like when it's not a lesson exercise but a team's actual multi-week incident log.
- **Imbue — "From bare metal to a 70B model"** — <https://imbue.com/research/70b-infrastructure/>. Explicitly covers InfiniBand/NCCL correctness at the scale of 12,000+ network connections, where one flaky link degrades the whole run. *What it shows:* the transport-hierarchy view scaled up — at thousands of connections, "one wrong transport line" becomes a needle-in-a-haystack problem the RAS-style tooling exists to solve.
- **stas00/ml-engineering — network debugging guide** — <https://github.com/stas00/ml-engineering/blob/master/network/debug/README.md>. A field guide (not vendor documentation) for diagnosing NCCL/network issues, with companion hands-on scripts: `all_reduce_bench.py` (<https://github.com/stas00/ml-engineering/blob/master/network/benchmarks/all_reduce_bench.py>) for quick bandwidth benchmarking, and `torch-distributed-gpu-test.py` (<https://github.com/stas00/ml-engineering/blob/master/debug/torch-distributed-gpu-test.py>) for confirming every GPU in a cluster can actually talk to every other one. *What it shows:* a practitioner-written, hands-on companion for exactly the Practice section below.

## Worked example

**Symptom.** A 64-GPU FSDP run wedges at step 4,120. Grafana shows all 64 GPUs at 100% utilization; the loss curve and step counter froze 12 minutes ago. No XID on the health dashboard. On-call reflex is "GPU util is 100%, it's working" — wrong; from 01b you know 100% util can be 100% *waiting*.

**Triage.**
1. `nvidia-smi` on a sample of nodes: util 100%, **power ~40% of a normal training step** — the tell that GPUs are spinning in a NCCL comm kernel, not computing. Confirms hang, not slowness.
2. **RAS query:** connect a RAS client to a rank's listener (`NCCL_RAS_ADDR` default `127.0.0.1:28028`). The status table shows communicator 0 with 63 ranks in `ALLGATHER` (FSDP's parameter gather from 08.1) and **global rank 47 not responding**.
3. **Localize rank 47.** It's GPU 7 on node `gpu-093`. `dmesg` on that node: the host OOM-killer reaped the training process 12 minutes ago (a data-loader memory leak) — so rank 47 vanished mid-all-gather, and the other 63 have been barrier-blocked ever since, at 100% util, silently.
4. **Root cause named:** not a GPU fault (05 clean, correctly), not a fabric fault — a *host* OOM on one node killed one rank; the collective barrier propagated a single-node failure into a 64-GPU hang. The watchdog, had it fired at 30 min, would have blamed the 63 waiters.

**The payoff:** RAS turned a "3,000-GPU silent hang" pattern into a single named rank in one query; without it you'd have grep'd 64 log streams for the one that stopped first. The fix (cap the loader, add a liveness probe) and the *reliability lesson* — one host failure hangs the whole gang until something kills it — feed straight into the checkpoint/restart design in the deliverable.

## Practice

**Environment:** one node, **2 rented GPUs**, single node so NVLink or PCIe (and SHM) are the transports in play. Reuse the tiny DDP job from 08.1.

1. **Capture the topology and transport decision.** Launch with:
   ```
   NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=INIT,GRAPH,NET torchrun --nproc_per_node=2 train.py 2>&1 | tee nccl.log
   ```
   In `nccl.log`, find and highlight: (a) the **transport** for the GPU0↔GPU1 pair (`via P2P`/`NVLink` if NVLink present, else `via PCIe`/`SHM`), and (b) the **algorithm** (ring vs tree) and channel count NCCL built. The `GRAPH` lines show the discovered connectivity; the `INIT` lines show the per-peer transport.
2. **Force a different decision and diff the logs.** Re-run with `NCCL_ALGO=Tree` and separately with `NCCL_P2P_DISABLE=1`. Observe: `NCCL_ALGO=Tree` changes the logged algorithm; `NCCL_P2P_DISABLE=1` drops the direct P2P path and forces **`via SHM`** (or net) between the two GPUs — visible in the INIT lines, and slower in step time. This is you *steering* NCCL's transport/algorithm choice on demand.
3. **See the hang.** Add a `time.sleep(600)` on `rank==1` just before an all-reduce; watch rank 0 pin to 100% util with the step counter frozen, then the PyTorch watchdog timeout fire naming rank 0 (the *waiter*). If your NCCL is ≥ 2.24, query RAS during the sleep and confirm it names rank 1 as the missing one.
4. **(Optional, benchmark cross-check)** Run `all_reduce_bench.py` from the stas00/ml-engineering repo (linked above) to get a raw bandwidth number for your link, then compare it against the step-time delta you measured when you disabled P2P in step 2.

**Acceptance (feeds "Survive-a-failure"):** a captured `nccl.log` excerpt (15–30 lines) with the **transport identified** (NVLink/PCIe/SHM) and the **algorithm identified** (ring/tree), annotated in your own words, plus one line showing how the log *changed* when you forced `NCCL_ALGO=Tree` or `NCCL_P2P_DISABLE=1`, and a short note on the injected hang (which rank the watchdog blamed vs which rank actually slept). Commit it under [`../practice/survive-a-failure/`](../practice/survive-a-failure/README.md) — this is the "I can read a NCCL log" evidence the deliverable's failure-injection step builds on.

## Common pitfalls

- **"100% GPU utilization means the GPU is working."** A GPU spinning inside a NCCL collective waiting for a peer also reads 100% util in `nvidia-smi`. The tell is **power draw well below an active compute step** — both the Crusoe fabric-manager case and the OPT-175B logs above are real instances of this exact trap.
- **"The rank the watchdog timeout names is the faulty one."** The timeout fires on the *waiters*, not the rank that died or left. Hunt for the missing rank; RAS gives it to you directly on 2.24+, pre-2.24 requires manual log correlation.
- **"`NCCL_IB_DISABLE` and `NCCL_P2P_DISABLE` are fixes."** They're diagnostic toggles that force a slower fallback path to confirm a hypothesis. Leaving them set in production is a severe — and usually accidental — performance regression, not a resolution.
- **"A NCCL hang always means a hardware fault."** Host-level causes are equally common and produce identical symptoms: an OOM-killer reaping one rank (the worked example above), a stalled data loader on one rank, or a misconfigured `NCCL_SOCKET_IFNAME` picking a management NIC. A host OOM-kill is a common real-world cause, not a GPU fault — don't jump straight to "bad hardware."
- **"Ring is always the right algorithm."** Ring is bandwidth-optimal for large messages, but its latency scales linearly with N. At very large world sizes or with small messages, tree — or NVLS/SHARP in-network reduction — wins on latency. NCCL auto-tunes this correctly most of the time; forcing the wrong algorithm via `NCCL_ALGO` in production is itself a common self-inflicted slowdown.

## Self-check

- **A job hangs at 100% GPU util with no error — what are your first three steps?**
  **Answer:** (1) **Confirm it's a hang, not slow progress:** `nvidia-smi` on suspects — 100% util but *lower power* than active compute means the GPUs are spinning in a NCCL comm kernel, and the step counter is frozen in logs. (2) **Query the RAS subsystem** (NCCL 2.24+, `NCCL_RAS_ADDR` default `127.0.0.1:28028`) to get the live per-rank status and find the rank(s) missing from the in-flight collective. (3) **Localize that odd rank out** — check 05's health on its node (XID/DCGM), its NIC (`ibstat`/link flaps), and the host (`dmesg` for OOM-kills, stalled data loader). The watchdog timeout blames the *waiters*; you're hunting the rank that *left*.

- **What does the GRAPH subsystem tell you?**
  **Answer:** `NCCL_DEBUG_SUBSYS=GRAPH` prints NCCL's **topology detection and the communication graph it built on top of it** — the discovered PCIe/NVLink/NIC connectivity and the channels/rings/trees and paths NCCL computed across that hardware. You use it to confirm NCCL *sees* the physical topology from 02b correctly (right NVLink links, right NICs) and chose sane paths — a wrong or degraded graph here explains a slow or SHM-fallback run.

- **Which env var confirms whether NCCL is using InfiniBand vs TCP sockets?**
  **Answer:** **`NCCL_IB_DISABLE`.** Setting `NCCL_IB_DISABLE=1` turns off the InfiniBand/RoCE verbs transport and forces the TCP **socket** fallback — if a broken run suddenly works (slowly) with it set, your IB path was the problem. To *see* which is in use without changing behavior, read `NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=NET,INIT`: it logs `via NET/IB` (InfiniBand) vs `via NET/Socket` (TCP) and the selected HCA/interface.

- **Why is RAS a bigger deal than it sounds, and what's the pre-2.24 alternative?**
  **Answer:** Before NCCL 2.24, a silent hang gave you nothing to query — you had to grep and correlate every rank's log by hand to find which one stopped incrementing first, which on a fleet of thousands of ranks can take an hour. RAS runs a lightweight background listener per process (default `127.0.0.1:28028`) that a client can query *without perturbing the job*, returning a live per-communicator, per-rank status table that names exactly which rank is missing from the in-flight collective. It converts "3,000 GPUs hung, no error" into one line — "communicator X waiting on rank 4831" — turning an hour of log archaeology into a single query.

- **You see `via SHM` in the NCCL init log for a GPU pair you expected to use NVLink. Is this noise?**
  **Answer:** No — never. The transport hierarchy (NVLink P2P > PCIe P2P > SHM > NET/IB > NET/Socket) reflects a physical fact about the machine, so a transport falling back further down that list than expected is always diagnostic of a real problem: the NVLink path wasn't detected or isn't usable between that pair, which could mean a topology misdetection, a placement that split GPUs that should be NVLink-adjacent, or a genuine hardware issue. Confirm with `NCCL_DEBUG_SUBSYS=GRAPH` to see what topology NCCL actually discovered, then cross-check against `nvidia-smi topo -m`.

## Connections & what's next

You now own the module's anchor skill: reading NCCL's transport/algorithm decisions and triaging a silent hang from "3,000 GPUs stuck" down to one named rank. **08.3** takes this same collective machinery and asks a throughput question instead of a failure question — given that all-reduce/all-gather cost is real and grows with world size, how much of your purchased FLOPs does a real step actually deliver (MFU), and how do you tell a comms-bound run from a data-loader-bound one. The triage sequence here also feeds directly into **08.5** (failure & elasticity): every hang you localize to a dead rank is a restart decision, and the "Survive-a-failure" deliverable's failure-injection step is built on the `nccl.log` reading skill from this lesson's Practice.

## References & further reading

**Primary sources**
1. **NCCL troubleshooting guide** — <https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html>. **Deep.** The authoritative playbook for hangs, transport fallbacks, and the exact log lines to look for; read it end-to-end once, then keep it open on an incident. Confirmed current as of writing to document **NCCL 2.30.3**, matching this module's pin to ~v2.30.x. Why: it's the source of truth for the triage sequence above.
2. **NCCL environment variables** — <https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html>. **Deep** for the debug/transport knobs (`NCCL_DEBUG`, `NCCL_DEBUG_SUBSYS`, `NCCL_ALGO`, `NCCL_PROTO`, `NCCL_P2P_DISABLE`, `NCCL_IB_DISABLE`, `NCCL_SOCKET_IFNAME`, `NCCL_IB_HCA`) and the `NCCL_RAS_*` settings. Why: this is the reference for every override you'll reach for.
3. **NCCL RAS subsystem docs** — <https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/ras.html>. **Skim** to learn how to run the RAS client and read its per-rank status table. Why: on NCCL ≥ 2.24 this is the fastest path from "silent hang" to "named missing rank."

**Real-world engineering blogs**
4. **Crusoe Cloud Support — fabric-manager hang runbook** — <https://support.crusoecloud.com/hc/en-us/articles/46061806112155-NCCL-Hangs-and-Multi-Node-Training-Stalls-Caused-by-Failed-nvidia-fabricmanager>. **Skim.** A live operational runbook for a real "hang, no error" root cause distinct from the worked example — read it as a second data point on the archetype, not a replacement for the troubleshooting guide.
5. **Meta — OPT-175B chronicles** — <https://github.com/facebookresearch/metaseq/tree/main/projects/OPT/chronicles>. **Skim the logbook.** Real, unsanitized NCCL/IB errors and restart counts from a 992-GPU run.
6. **stas00/ml-engineering — network debugging guide** — <https://github.com/stas00/ml-engineering/blob/master/network/debug/README.md>. **Skim**, and use `all_reduce_bench.py` / `torch-distributed-gpu-test.py` as hands-on companions for the Practice section.

**Deeper dives**
7. **"Mycroft: Tracing Dependencies in Collective Communication Towards Reliable LLM Training"** — <https://arxiv.org/abs/2509.03018>. An academic (ByteDance/Harvard) treatment of the exact "one dead rank hangs everyone" diagnosis problem, deployed in production for 6+ months; a look at where hang-detection tooling is heading past RAS.
8. **NVIDIA/nccl GitHub issues** — e.g. <https://github.com/NVIDIA/nccl/issues/1774> ("Training may hang if cudaMemcpy operations are present"). **Illustrative, not authoritative** — real, current texture on what actually goes wrong at scale, useful for pattern-matching but not a substitute for the official docs.
