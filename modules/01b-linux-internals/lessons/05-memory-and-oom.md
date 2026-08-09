---
lesson: "01b.5"
title: "Memory, Reclaim, and the OOM Killer"
module: "01b"
concept: "Memory, Reclaim, and the OOM Killer"
status: not-started
est_time: "6h"
artifacts: []
---

# 01b.5 · Memory, Reclaim, and the OOM Killer

> **Concept.** Virtual vs resident vs working set, the page cache and the reclaim path, and how the kernel chooses a victim — cgroup-scoped or global — when memory runs out.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters

A model-serving pod on a GPU node gets `OOMKilled`. The dashboard shows the node had 40 GB "free." The team blames the node, cordons it, pages SRE. All of that is wrong, and knowing *why* is a senior-vs-staff line. The pod died because **its own cgroup** hit `memory.max`, not because the node ran out; the node's 40 GB of "free" was mostly reclaimable page cache the kernel would have handed over instantly. The kernel wrote a complete post-mortem to `dmesg` naming the victim, its RSS, its `oom_score_adj`, and whether the kill was cgroup-scoped or global — and nobody read it.

On a GPU fleet the failure modes are asymmetric and expensive:

- A **fat model process** (a training job that grows its resident set past its limit) gets OOM-killed mid-epoch — a lost checkpoint and hours of idle GPU.
- A **bad limit** set below the model's true working set turns every run into a guaranteed cgroup-OOM.
- A **global** OOM on a shared node can take down the kubelet or the GPU device plugin if critical daemons aren't protected — turning one bad pod into a node outage.

Reading the OOM report cold, and knowing the difference between cgroup and global OOM, is the difference between "restart the pod, raise the limit 20%" and a two-hour incident.

## From using to understanding

**What the operator knows.** `free -m`, `kubectl top pod`, "the container hit its memory limit," `OOMKilled` in `kubectl describe`. You set `resources.limits.memory` and know exceeding it kills the container.

**Why that model is thin.** It treats memory as one number ("used") and the limit as a wall. But "used" is at least four different things, the kernel reclaims most of what looks "used" for free, and the kill decision is a *scoring function* over processes, not a simple threshold trip. Without the mechanism you can't tell a right-sized limit from a landmine, can't explain why K8s uses working-set and not RSS, and can't read the one artifact that actually explains the kill.

**The kernel mechanism.** Memory is accounted in pages (typically 4 KiB). A process's address space (VMAs) is *virtual* — mostly promises. Physical pages get attached on fault. Under pressure the kernel runs **reclaim**: it ages pages on LRU lists, writes dirty ones back, drops clean file pages, and swaps anonymous pages if swap exists. Only when reclaim can free nothing more does the **OOM killer** run `oom_badness()` over candidate tasks and SIGKILL the highest scorer. All of this happens per-cgroup first (charged against `memory.max`) and globally as a fallback.

## Core notes

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
- **`memory.pressure`** — PSI for this cgroup (lesson 01b.4): rising `full` is the pre-OOM thrash signal.

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
- Kernel/PID-1/some kthreads are exempt. The killer prefers a task that frees the most memory in the failing scope, then SIGKILLs it (and, with cgroup `memory.oom.group`, the whole cgroup together).

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

## Practice

**Environment:** Linux VM/laptop with cgroup v2, `stress-ng`, root, `dmesg` access. Container works if you can create a delegated sub-cgroup and set `memory.max`.

1. **Cause a cgroup OOM.** Create `/sys/fs/cgroup/oom-demo`, set `memory.max=256M` and `memory.swap.max=0`, run a `stress-ng --vm 1 --vm-bytes 1G` inside it. Confirm `memory.events` `oom_kill` increments.
2. **Capture and dissect the dmesg report.** Grab the OOM lines. Identify: the **triggering** task, the **victim** PID/name, the victim's `total-vm` and `anon-rss` (convert page counts if the task table is present), its `oom_score_adj`, and — decisively — whether the report says "**Memory cgroup out of memory**" (cgroup) or "**Out of memory**" + a node memory summary (global).
3. **Prove the scope claim.** While the demo cgroup OOMs, show `free -m` on the host with plenty `available` — evidence the *node* was fine and only the cgroup died.
4. **(Stretch) Bias the choice.** Set `oom_score_adj=-998` on one process and show it survives while a smaller peer is chosen — demonstrating protection of "critical" workloads.

**Acceptance (feeds "Anatomy of a Container"):** an **annotated OOM dmesg excerpt** — paste the real lines and annotate inline: (1) **who triggered** the OOM, (2) **who was killed** and its `anon-rss` / `total-vm`, (3) its **`oom_score_adj`**, (4) **cgroup-scoped or global**, with the exact log phrase you used to decide, and (5) one sentence on *why the victim couldn't be reclaimed* (anon + swap off). Tie it to the GPU failure mode: a right-sized limit vs a landmine.

## Self-check

**(a) Why does Kubernetes use working-set (not RSS) for memory limits/eviction?**
**Answer:** RSS includes reclaimable, cold file-cache pages and shared pages the kernel can drop for free — counting them would over-report true demand and evict/kill pods that aren't actually starved. **Working set ≈ `anon + active_file`** (from `memory.stat`) — the pages needed to run *without thrashing*, excluding the cold cache the kernel will reclaim on demand. It's the number that actually predicts OOM, so the kubelet computes `container_memory_working_set_bytes` and uses it for both the limit-kill decision and node-pressure eviction.

**(b) What does `oom_score_adj=-998` on the kubelet/critical pods achieve?**
**Answer:** `oom_score_adj` is a permille bias (-1000…+1000) added to a task's footprint-based `oom_badness` score. A strongly negative value (K8s uses -997/-998 for Guaranteed/critical, -1000 for some node daemons) pushes the score to the floor so those tasks are chosen **last** — effectively immune during a **global** OOM. It ensures that when the node as a whole runs out of memory, the kernel kills workload pods, not the kubelet, container runtime, or GPU device plugin — keeping the node manageable instead of turning one bad pod into a node outage. (`-1000` = never selected.)

**(c) Given an OOM log line, who died and why — walk the fields.**
**Answer:** Take `Memory cgroup out of memory: Killed process 5123 (stress-ng-vm) total-vm:1064512kB, anon-rss:248860kB, file-rss:512kB, oom_score_adj:0`. **Who:** PID 5123, `stress-ng-vm`. **Scope:** "Memory cgroup out of memory" → a **cgroup** OOM (its `memory.max` was hit), not the node. **Why it was chosen:** it held the largest footprint in that cgroup — `anon-rss ≈ 249 MB`, filling the cap — with `oom_score_adj:0` (no bias), so raw footprint decided. **Why it couldn't be saved:** the memory is **anonymous** and (swap off) unreclaimable, so reclaim freed nothing and OOM was the only option. `total-vm:1064512kB` shows it *asked* for ~1 GB virtual but only ~249 MB ever resided before the kill.

## Resources

1. **Brendan Gregg, *Systems Performance* (2nd ed.), Memory chapter** — <https://www.brendangregg.com/systems-performance-2nd-edition-book.html>. Virtual vs resident vs working set, page cache, reclaim, and swap behavior, with the tools to observe each. Read first.
2. **cgroup v2 memory controller (kernel admin guide)** — <https://docs.kernel.org/admin-guide/cgroup-v2.html>. Canonical reference for `memory.max`/`.high`/`.current`/`.stat`/`.events`, working-set accounting, and how cgroup OOM differs from global OOM.
3. **Kernel OOM internals — `mm/oom_kill.c` and `Documentation/`** — <https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html>. For `oom_badness()` scoring, `oom_score_adj` semantics, and the exact fields in the dmesg OOM report.
