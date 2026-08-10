---
lesson: "01b.5"
title: "Memory, Reclaim, and the OOM Killer"
module: "01b"
concept: "Memory, Reclaim, and the OOM Killer"
status: not-started
est_time: "7h"
prev: "04-psi.md"
next: "06-hugepages-thp-numa.md"
artifacts: []
sources: 5
---

# 01b.5 · Memory, Reclaim, and the OOM Killer

> **Concept.** Virtual vs resident vs working set, the page cache and the reclaim path, and how the kernel chooses a victim — cgroup-scoped or global — when memory runs out.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 04 gave you PSI: the stall-time signal that shows a workload being *denied* a resource, including the specific pre-OOM thrash signature — memory `full` pressure climbing while a cgroup burns wall-clock time in direct reclaim instead of doing work. That lesson deliberately stopped short of the endgame: what happens when reclaim can't keep up at all. This lesson picks up exactly there. It covers the reclaim path in full, the scoring function the kernel runs when reclaim fails, and the artifact — the `dmesg` OOM report — that tells you precisely who died and why. Once you can read that report cold, the next lesson (06, Hugepages/THP/NUMA) moves to the other half of the CPU-side memory story: not "did we run out," but "is the memory we *do* have being served to the GPU data path efficiently."

## Why this matters

A model-serving pod on a GPU node gets `OOMKilled`. The dashboard shows the node had 40 GB "free." The team blames the node, cordons it, pages SRE. All of that is wrong, and knowing *why* is a senior-vs-staff line. The pod died because **its own cgroup** hit `memory.max`, not because the node ran out; the node's 40 GB of "free" was mostly reclaimable page cache the kernel would have handed over instantly. The kernel wrote a complete post-mortem to `dmesg` naming the victim, its RSS, its `oom_score_adj`, and whether the kill was cgroup-scoped or global — and nobody read it.

On a GPU fleet the failure modes are asymmetric and expensive:

- A **fat model process** (a training job that grows its resident set past its limit) gets OOM-killed mid-epoch — a lost checkpoint and hours of idle GPU.
- A **bad limit** set below the model's true working set turns every run into a guaranteed cgroup-OOM.
- A **global** OOM on a shared node can take down the kubelet or the GPU device plugin if critical daemons aren't protected — turning one bad pod into a node outage.

Reading the OOM report cold, and knowing the difference between cgroup and global OOM, is the difference between "restart the pod, raise the limit 20%" and a two-hour incident. It is also, as the economics perspective below quantifies, the difference between losing minutes and losing hours of GPU-hours on every kill.

## What's new here (calibration)

Per the module README's calibration, you already know `free -m`, `kubectl top pod`, "the container hit its memory limit," and `OOMKilled` in `kubectl describe` — that operator-level vocabulary is not re-taught here. What's genuinely new at this depth:

- The **four distinct meanings of "memory used"** (virtual, resident, cached, working set) and why Kubernetes deliberately picks the least intuitive one (working set) for limits and eviction.
- The **reclaim path as a state machine** — what the kernel tries, in what order, before it ever calls the OOM killer — not just "it kills things when full."
- **`oom_badness()` as an actual scoring function** you can predict and bias, not a black box.
- The **cgroup-vs-global OOM distinction and its exact dmesg tell**, plus `memory.oom.group` as the K8s-relevant knob for multi-process jobs.
- The **economics of a mid-epoch kill** — translating "the pod restarted" into a GPU-hours number a staff engineer can put in a design doc.

## Core concepts

### From using to understanding

**What the operator knows.** `free -m`, `kubectl top pod`, "the container hit its memory limit," `OOMKilled` in `kubectl describe`. You set `resources.limits.memory` and know exceeding it kills the container.

**Why that model is thin.** It treats memory as one number ("used") and the limit as a wall. But "used" is at least four different things, the kernel reclaims most of what looks "used" for free, and the kill decision is a *scoring function* over processes, not a simple threshold trip. Without the mechanism you can't tell a right-sized limit from a landmine, can't explain why K8s uses working-set and not RSS, and can't read the one artifact that actually explains the kill.

**The kernel mechanism.** Memory is accounted in pages (typically 4 KiB). A process's address space (VMAs) is *virtual* — mostly promises. Physical pages get attached on fault. Under pressure the kernel runs **reclaim**: it ages pages on LRU lists, writes dirty ones back, drops clean file pages, and swaps anonymous pages if swap exists. Only when reclaim can free nothing more does the **OOM killer** run `oom_badness()` over candidate tasks and SIGKILL the highest scorer. All of this happens per-cgroup first (charged against `memory.max`) and globally as a fallback.

### The four memory numbers

- **Virtual (VSZ / `VmSize`)** — the size of the address space: all mappings, mostly unbacked. A Go or CUDA process can show tens of GB of virtual with a fraction resident. **Never limit on this.** GPU processes reserve huge virtual ranges for device mappings — VSZ is meaningless for capacity.
- **Resident (RSS / `VmRSS`)** — physical pages currently mapped in: anonymous (heap/stack) + mapped file pages + shared. Closer, but includes shared pages counted in multiple processes and includes page-cache-backed file pages that are *reclaimable*.
- **Page cache** — clean/dirty copies of file data in RAM. Shows as "used" in naive tools but is mostly **reclaimable**: clean pages drop instantly, dirty pages after writeback. This is why a node with "40 GB used" may have 38 GB the kernel will surrender on demand.
- **Working set** — the pages actively needed to run *without thrashing*: roughly `anon + active file` — the resident set **minus** the cold, reclaimable cache. This is the number that predicts OOM, and it is what Kubernetes uses.

`free -m` makes the point:

```
              total        used        free      shared  buff/cache   available
Mem:         64000       12000        2100         300       49900       50800
```

`free` is tiny (2.1 GB) but `available` is 50.8 GB — the kernel's own estimate of what a new allocation could get *without swapping*, because `buff/cache` is reclaimable. Reading `used` alone is the classic mistake.

### Where the kernel keeps the truth: cgroup v2 memory controller

Per cgroup (`/sys/fs/cgroup/<path>/`):

- **`memory.current`** — bytes currently charged to this cgroup.
- **`memory.max`** — the hard limit. Charging past it triggers reclaim, then cgroup OOM. (This is what a K8s `limits.memory` becomes.)
- **`memory.high`** — a *soft* throttle: past it the kernel aggressively reclaims and throttles the cgroup (stalls it) but does **not** kill. A pressure-relief valve.
- **`memory.min` / `memory.low`** — reclaim protection (guaranteed / best-effort).
- **`memory.stat`** — the breakdown: `anon`, `file`, `active_file`, `inactive_file`, `slab`, etc. **Working set ≈ `anon + active_file`.**
- **`memory.events`** — counters: `low`, `high`, `max` (times charging hit `max` and reclaimed hard), and **`oom`** / **`oom_kill`** (times the cgroup OOM killer ran / processes it killed). `oom_kill` incrementing is your cgroup-OOM fingerprint.
- **`memory.pressure`** — PSI for this cgroup (lesson 04): rising `full` is the pre-OOM thrash signal.

```
$ cat /sys/fs/cgroup/kubepods.slice/.../memory.current   # 268435456
$ cat .../memory.max                                      # 268435456  (256M)
$ grep -E 'anon|active_file' .../memory.stat
anon 251658240
active_file 4194304
$ cat .../memory.events
low 0
high 1423
max 88
oom 3
oom_kill 3
```

Reading this: current is pinned at max, working set (`anon`+`active_file` ≈ 244 MB) is essentially the whole limit, `high` throttled 1423 times, and the cgroup OOM killer fired 3 times killing 3 processes. This pod's limit is below its working set — it is a landmine.

### The reclaim path under pressure

1. Allocation can't be satisfied from free pages → enter reclaim.
2. Scan LRU lists (active/inactive, file/anon). **Clean file pages** → dropped for free. **Dirty file pages** → written back then dropped. **Anonymous pages** → swapped out *if swap exists*; with **swap off (the K8s default), anon pages are unreclaimable**, so anon-heavy growth goes straight to OOM.
3. If reclaim keeps up, the workload just slows (PSI `memory` pressure rises — thrash). If reclaim can free nothing and the charge still can't be met → invoke OOM.

**Consequence for GPU jobs:** model processes are almost entirely *anonymous* memory (tensors, activations). With swap disabled, there is nothing to reclaim, so a model that grows past `memory.max` cannot be throttled into surviving — it goes from fine to killed with no soft landing. That's why `memory.high` and right-sized limits matter more than "the node has RAM."

### oom_badness() — how the victim is chosen

When OOM fires (in a scope — one cgroup, or global), the kernel scores each eligible task with `oom_badness()`:

- **Base score ≈ the task's memory footprint**: RSS + swap + page-table + some kernel structures, expressed roughly as a **permille (0–1000) of available memory in the scope**. Bigger footprint → higher score → more likely victim. "Kill the hog" is the core policy.
- **`oom_score_adj`** (range **-1000 … +1000**, in `/proc/<pid>/oom_score_adj`) is **added as a permille bias**. `+1000` forces a task to be picked essentially always; **`-1000` makes it OOM-immune** (score floored to 0, never selected). Values in between shift the footprint-based score up or down.
- Final per-task score visible at **`/proc/<pid>/oom_score`**.
- Kernel/PID-1/some kthreads are exempt. The killer prefers a task that frees the most memory in the failing scope, then SIGKILLs it.

### memory.oom.group — killing the whole job, not one rank

A distributed training job is not one process; it's a process group — rank-0 plus N workers, often started by `torchrun` or `mpirun`, all inside one pod's cgroup. If the cgroup OOMs and the killer picks the single largest process, the default behavior kills *that one task* and leaves the rest of the job running against a now-dead peer — a half-dead job that hangs on a collective op (an all-reduce waiting on a rank that will never respond) instead of failing cleanly. `memory.oom.group=1` on the cgroup changes this: when OOM fires anywhere in the cgroup, the kernel kills **every process in it atomically**, not just the top scorer. For a multi-process training job, that's the correct behavior — a clean full restart from the last checkpoint beats a zombie job burning GPU-hours on a collective that never completes. This is exactly the kind of cgroup-scope knob a container runtime or K8s-adjacent orchestrator sets on the pod's cgroup for this reason — it's the difference between "one rank died, job is now stuck" and "job died, restart is unambiguous."

### cgroup OOM vs global OOM

- **cgroup OOM** — a cgroup's charge hits `memory.max` and reclaim within that cgroup can't help. Only tasks **in that cgroup** are candidates. The rest of the node is fine and has free RAM. `memory.events`.`oom_kill` increments. This is the overwhelmingly common Kubernetes case (`OOMKilled` on one pod). The dmesg report names the offending `memcg`.
- **Global OOM** — total system memory is exhausted and system-wide reclaim fails. **Every** eligible task on the node is a candidate, ranked by `oom_badness` across the whole box. This can kill anything unprotected — which is exactly why the kubelet and critical daemons run with strongly negative `oom_score_adj`.

### Reading the OOM report from dmesg

An OOM event writes a multi-line report. The anatomy:

```
[12345.678] python invoked oom-killer: gfp_mask=0x..., order=0, oom_score_adj=0
[12345.679] memory: usage 262144kB, limit 262144kB, failcnt 88
[12345.680] Memory cgroup stats for /kubepods.slice/.../podXXXX: ...
[12345.681] Tasks state (memory values in pages):
[12345.681] [  pid  ]   uid  tgid total_vm      rss ... oom_score_adj name
[12345.681] [  4711 ]  1000  4711   918273   61234 ...              0 python
[12345.682] oom-killer: ... Memory cgroup out of memory: Killed process 4711 (python)
              total-vm:3673092kB, anon-rss:244936kB, file-rss:0kB, ...
```

- Line 1: **who triggered it** (the allocating task) and its `oom_score_adj`.
- Line 2 `memory: usage … limit … failcnt`: this is a **cgroup** OOM — usage equals limit. A **global** OOM instead prints a system memory summary (`Node 0 …`, `Normal free:… min:…`) and *no* `Memory cgroup out of memory` line.
- "**Memory cgroup out of memory**" (vs "**Out of memory**") is the single fastest cgroup-vs-global tell.
- The **task table** lists candidates with `total_vm`, `rss` (in **pages** — multiply by 4 KiB), and `oom_score_adj`.
- The **Killed process** line names the victim and its `anon-rss` — for a model process this is almost all anon (unreclaimable, swap off), confirming *why* it couldn't be saved.

## Perspectives

**Kernel-mechanism view.** From the kernel's side, OOM is a last-resort fallback, not a feature. Reclaim exhausts every cheap option first — drop clean cache, write back dirty cache, swap anon if possible — and only calls `oom_badness()` when the allocator genuinely cannot make forward progress. The scoring is deliberately simple (footprint-proportional, permille-biased) because it has to run under extreme memory pressure with no room for cleverness. This simplicity is also its limitation, which the economics and Meta perspectives below both push on.

**Operator/SRE view.** The single highest-leverage on-call skill for this topic is not knowing the theory — it's reading a cold `dmesg` OOM report under incident pressure and extracting five facts in under a minute: who triggered it, who died, its `oom_score_adj`, whether it was cgroup or global scope (the exact phrase to look for), and why the victim couldn't be reclaimed. That is a mechanical skill you can drill, and it directly determines whether an incident becomes "restart the pod, adjust the limit" or a much longer node-level investigation because someone assumed cgroup scope when it was actually global (or vice versa).

**GPU-fleet-specific view.** GPU training and inference processes are structurally the worst case for reclaim. Their memory is almost entirely anonymous — tensors, activations, optimizer state — not file-backed page cache the kernel can drop for free. Combine that with swap disabled (the near-universal Kubernetes default, since swapping a GPU-feeding process is worse than killing it), and there is no soft landing: a model process that crosses `memory.max` has no reclaimable pages to surrender, so reclaim fails immediately and the kill is instantaneous rather than gradual. This is why `memory.high` (the soft throttle) and correctly-sized limits matter far more for training pods than for a typical stateless web service that has plenty of reclaimable cache to cushion a spike.

**Economics/failure-mode view.** A mid-epoch OOM kill is not "restart the pod" — it is a quantifiable amount of wasted GPU-hours. If checkpoints are written every 30 minutes and the kill lands 25 minutes into an interval, you lose 25 minutes of GPU-hours across every GPU in that job, multiplied by however many ranks were in the process group (compounded further if `memory.oom.group` isn't set and the job hangs for a while before anyone notices the zombie state). For an 8-GPU job at a rough 2026 on-demand rate of order $2–3/GPU-hour, a single 25-minute loss is on the order of $7–10 of pure waste (flagged as a dated 2026 snapshot, and the number scales linearly with GPU count and rate) — trivial in isolation, but multiplied across a fleet running hundreds of such jobs with under-tuned limits, it becomes a real line item. The lever is checkpoint frequency: more frequent checkpoints shrink the average blast radius of any kill but cost their own I/O and stall time (itself a PSI-visible cost from lesson 04). Sizing checkpoint interval against observed OOM-kill rate — not against a round number like "every hour" — is the staff-level version of this trade-off.

## Real-world use cases

- **LINE Engineering — "Who murdered my lovely Prometheus container in Kubernetes cluster?"** — <https://engineering.linecorp.com/en/blog/prometheus-container-kubernetes-cluster> — a real production investigation of a container getting OOM-killed, walking through `oom_score_adj`, cgroup memory limits, and `dmesg` evidence. Structurally the same investigation as this lesson's worked example, told as a real incident with a real root cause.
- **Meta Engineering — "Open-sourcing oomd"** — <https://engineering.fb.com/2018/07/19/production-engineering/oomd/> — the victim-selection-limits angle: the kernel's global-scope `oom_badness()` killer is a last resort that can pick the wrong victim or act too late (only after allocation has already failed). Meta built `oomd` to act earlier, cgroup-aware, using policy they control instead of the kernel's generic heuristic — direct evidence that `oom_badness()` alone is not always the right final word in production.

## Worked example

**Goal: trigger a cgroup OOM, then explain the kill from dmesg alone.**

```
$ sudo mkdir /sys/fs/cgroup/oom-demo
$ echo 256M | sudo tee /sys/fs/cgroup/oom-demo/memory.max
$ echo 0    | sudo tee /sys/fs/cgroup/oom-demo/memory.swap.max   # no swap escape
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/oom-demo/cgroup.procs; \
    exec stress-ng --vm 1 --vm-bytes 1G --vm-keep --timeout 30s'
```

The hog wants 1 GB anonymous inside a 256 MB cap with swap off — reclaim has nothing to give. Watch it die:

```
$ cat /sys/fs/cgroup/oom-demo/memory.events
...
oom 1
oom_kill 1
$ sudo dmesg | tail -30
```

```
stress-ng-vm invoked oom-killer: gfp_mask=0x..., order=0, oom_score_adj=0
memory: usage 262144kB, limit 262144kB, failcnt 173
Memory cgroup out of memory: Killed process 5123 (stress-ng-vm)
  total-vm:1064512kB, anon-rss:248860kB, file-rss:512kB,
  oom_score_adj:0
```

**Explain it from the log alone:**

- **Scope:** `memory: usage 262144kB, limit 262144kB` + "**Memory cgroup out of memory**" → **cgroup OOM**, not global. The node still has free RAM.
- **Who died:** PID 5123 `stress-ng-vm`.
- **Why it:** it was the largest anon consumer in the failing cgroup — `anon-rss:248860kB` ≈ the entire 256 MB cap. `oom_score_adj:0` (no bias), so pure footprint decided it.
- **Why unsavable:** the footprint is **anon** and `memory.swap.max=0` → nothing reclaimable → reclaim failed → OOM. `total-vm:1064512kB` (1 GB virtual) confirms the intended allocation; only ~248 MB ever became resident before the kill.

**Contrast — force a bias:** rerun with `echo -1000 > /proc/<pid>/oom_score_adj` on a smaller sacrificial process and a larger unprotected one; observe the protected process survives even if it's the hog and the killer picks the next-highest scorer. That is `oom_score_adj` overriding footprint.

**Contrast — group kill:** set `echo 1 > /sys/fs/cgroup/oom-demo/memory.oom.group`, put two `stress-ng` processes in the same cgroup, and trigger the same OOM. `memory.events` `oom_kill` now shows **2**, not 1 — both processes died together, the behavior you want for a multi-rank training job instead of leaving one rank alive against a dead peer.

## Practice

**Environment:** Linux VM/laptop with cgroup v2, `stress-ng`, root, `dmesg` access. Container works if you can create a delegated sub-cgroup and set `memory.max`.

1. **Cause a cgroup OOM.** Create `/sys/fs/cgroup/oom-demo`, set `memory.max=256M` and `memory.swap.max=0`, run a `stress-ng --vm 1 --vm-bytes 1G` inside it. Confirm `memory.events` `oom_kill` increments.
2. **Capture and dissect the dmesg report.** Grab the OOM lines. Identify: the **triggering** task, the **victim** PID/name, the victim's `total-vm` and `anon-rss` (convert page counts if the task table is present), its `oom_score_adj`, and — decisively — whether the report says "**Memory cgroup out of memory**" (cgroup) or "**Out of memory**" + a node memory summary (global).
3. **Prove the scope claim.** While the demo cgroup OOMs, show `free -m` on the host with plenty `available` — evidence the *node* was fine and only the cgroup died.
4. **(Stretch) Bias the choice.** Set `oom_score_adj=-998` on one process and show it survives while a smaller peer is chosen — demonstrating protection of "critical" workloads.
5. **(Stretch) Group kill.** Set `memory.oom.group=1` on the demo cgroup, put two hogs in it, trigger OOM, and confirm both die — the multi-rank-job-safe behavior.

**Acceptance (feeds "Anatomy of a Container", [`../practice/anatomy-of-a-container/README.md`](../practice/anatomy-of-a-container/README.md)):** an **annotated OOM dmesg excerpt** — paste the real lines and annotate inline: (1) **who triggered** the OOM, (2) **who was killed** and its `anon-rss` / `total-vm`, (3) its **`oom_score_adj`**, (4) **cgroup-scoped or global**, with the exact log phrase you used to decide, and (5) one sentence on *why the victim couldn't be reclaimed* (anon + swap off). Tie it to the GPU failure mode: a right-sized limit vs a landmine, and note whether `memory.oom.group` would have changed the blast radius for a multi-process job.

## Common pitfalls

1. **Believing "40GB free" on `free -m` means headroom.** Most of it may be reclaimable page cache. Read `available`, not `free` or `used` — `available` is the kernel's own estimate of what you can allocate without swapping.
2. **Assuming RSS predicts an OOM kill.** RSS includes reclaimable file-cache pages the kernel will happily drop. The number that actually matters — and what Kubernetes uses — is **working set** (`anon + active_file`), not raw RSS.
3. **Treating every `OOMKilled` / exit 137 as identical.** Cgroup-scoped and node-global OOM are different incidents with very different blast radii. The `dmesg` phrase is the fast tell: "Memory cgroup out of memory" (cgroup, one pod) vs "Out of memory" plus a node-wide memory summary (global, anything unprotected is a candidate).
4. **Assuming `oom_score_adj` alone protects a process.** A strongly negative value only helps in the **global** OOM scope, where it competes against every task on the node. It does nothing to protect a process from a **cgroup-scoped** OOM triggered by that process's own `memory.max` — the cgroup killer only considers tasks inside that cgroup, and if it's the only large consumer there, it still dies.
5. **Expecting anonymous memory to be reclaimable when swap is off (the K8s default).** It isn't. A model process growing past its limit has no soft landing — reclaim fails immediately and the kill is instant, not gradual.

## Self-check

**(a) Why does Kubernetes use working-set (not RSS) for memory limits/eviction?**
**Answer:** RSS includes reclaimable, cold file-cache pages and shared pages the kernel can drop for free — counting them would over-report true demand and evict/kill pods that aren't actually starved. **Working set ≈ `anon + active_file`** (from `memory.stat`) — the pages needed to run *without thrashing*, excluding the cold cache the kernel will reclaim on demand. It's the number that actually predicts OOM, so the kubelet computes `container_memory_working_set_bytes` and uses it for both the limit-kill decision and node-pressure eviction.

**(b) What does `oom_score_adj=-998` on the kubelet/critical pods achieve?**
**Answer:** `oom_score_adj` is a permille bias (-1000…+1000) added to a task's footprint-based `oom_badness` score. A strongly negative value (K8s uses -997/-998 for Guaranteed/critical, -1000 for some node daemons) pushes the score to the floor so those tasks are chosen **last** — effectively immune during a **global** OOM. It ensures that when the node as a whole runs out of memory, the kernel kills workload pods, not the kubelet, container runtime, or GPU device plugin — keeping the node manageable instead of turning one bad pod into a node outage. (`-1000` = never selected.) It offers zero protection against a cgroup-scoped OOM confined to that process's own limit.

**(c) Given an OOM log line, who died and why — walk the fields.**
**Answer:** Take `Memory cgroup out of memory: Killed process 5123 (stress-ng-vm) total-vm:1064512kB, anon-rss:248860kB, file-rss:512kB, oom_score_adj:0`. **Who:** PID 5123, `stress-ng-vm`. **Scope:** "Memory cgroup out of memory" → a **cgroup** OOM (its `memory.max` was hit), not the node. **Why it was chosen:** it held the largest footprint in that cgroup — `anon-rss ≈ 249 MB`, filling the cap — with `oom_score_adj:0` (no bias), so raw footprint decided. **Why it couldn't be saved:** the memory is **anonymous** and (swap off) unreclaimable, so reclaim freed nothing and OOM was the only option. `total-vm:1064512kB` shows it *asked* for ~1 GB virtual but only ~249 MB ever resided before the kill.

**(d) A multi-rank training job's cgroup OOMs. Why might `memory.oom.group=1` matter more than which single process gets killed?**
**Answer:** Without `oom.group`, the killer picks one process — typically the largest — and kills only it, leaving the rest of the process group (other ranks) running. A distributed job is coordinated via collective operations (e.g. all-reduce); a surviving rank will block waiting on the now-dead one, producing a hung, half-dead job that keeps its GPUs allocated and idle instead of failing cleanly. With `memory.oom.group=1`, the kernel kills every process in the cgroup atomically on OOM, so the orchestrator sees one unambiguous full failure and can restart the whole job from the last checkpoint — a clean, fast recovery instead of a silent GPU-hour leak from a stuck job nobody notices until a timeout fires.

## Connections & what's next

This lesson depends on lesson 03's cgroup v2 mechanics (`memory.max` *is* the `memory.max` file from that lesson; the K8s `limits.memory` translation is identical) and on lesson 04's PSI (memory `full` pressure is the leading indicator that precedes the OOM event covered here — PSI tells you it's coming, this lesson tells you what happens and how to read the aftermath). It sets up lesson 06, **Hugepages, THP, and NUMA locality**: once you know memory isn't getting killed, the next question is whether the memory you kept is actually being served to the CPU and GPU efficiently — page-table walks, TLB coverage, and NUMA placement are the mechanisms that decide whether "surviving" memory is also *fast* memory.

## References & further reading

**Primary sources**
- **cgroup v2 memory controller (kernel admin guide)** — <https://docs.kernel.org/admin-guide/cgroup-v2.html> — read for the canonical semantics of `memory.max`/`.high`/`.current`/`.stat`/`.events`, working-set accounting, and how cgroup OOM differs from global OOM.
- **Kernel OOM internals — `mm/oom_kill.c` and `Documentation/`** — <https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html> — read for `oom_badness()` scoring, `oom_score_adj` semantics, `memory.oom.group`, and the exact fields in the dmesg OOM report.

**Real-world engineering blogs**
- **LINE Engineering — "Who murdered my lovely Prometheus container in Kubernetes cluster?"** — <https://engineering.linecorp.com/en/blog/prometheus-container-kubernetes-cluster> — what it shows: a real production OOM investigation walking `oom_score_adj`, cgroup limits, and `dmesg` evidence — the same investigation as this lesson's worked example, as a real incident.
- **Meta Engineering — "Open-sourcing oomd"** — <https://engineering.fb.com/2018/07/19/production-engineering/oomd/> — what it shows: why the kernel's global `oom_badness()` killer is a last resort with real limits (wrong victim, acts too late), and how a cgroup-aware userspace killer with its own policy improves on it.

**Deeper dives**
- **Brendan Gregg, *Systems Performance* (2nd ed.), Memory chapter** — <https://www.brendangregg.com/systems-performance-2nd-edition-book.html> — virtual vs resident vs working set, page cache, reclaim, and swap behavior, with the tools to observe each.
