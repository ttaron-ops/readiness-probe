---
lesson: "01b.3"
title: "cgroups v2 and Kubernetes resource enforcement"
module: "01b"
concept: "cgroups v2 and Kubernetes resource enforcement"
status: not-started
est_time: "8h"
artifacts: []
---

# 01b.3 · cgroups v2 and Kubernetes resource enforcement

> **Concept.** The unified cgroup-v2 hierarchy and its controllers — and exactly how a pod's requests, limits, and QoS class become cpu.max, cpu.weight, memory.max, and memory.high on the node.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters

This is the "how containers really work" story and the cost/efficiency story in one file. A container is not a lightweight VM; it is a normal Linux process whose resource consumption is fenced by cgroups. When you set `resources.limits.cpu: 500m` in YAML, nothing magical happens — the kubelet writes `50000 100000` into a `cpu.max` file, and the kernel's bandwidth controller does the rest. Understanding that write is the difference between operating Kubernetes and understanding it.

It is also the single most common container-Linux interview question, usually posed as the **"throttled at 40% CPU"** paradox: *"Your service has p99 latency spikes and is being CPU-throttled, but node dashboards show the pod averaging 40% of its limit. How is that possible?"* The answer lives entirely in `cpu.max`, the 100 ms CFS period, and `cpu.stat`'s `nr_throttled`. Engineers who can't explain it look junior; engineers who can walk from the YAML to the kernel file look senior.

Finally, this is the foundation of the capstone's cost attribution. Every dollar of GPU-node spend gets charged back through the cgroup tree — `kubepods.slice` and its QoS children are literally the accounting boundaries. If you can't read the tree, you can't attribute the cost, and cost/observability is your differentiator.

## From using to understanding

You already do the *using* part daily. You know that `requests` drive scheduling and act as a floor, `limits` are a ceiling, and that the request/limit relationship determines the QoS class (Guaranteed, Burstable, BestEffort) which in turn drives eviction and OOM order. Operationally, that is enough to pass CKA.

What you have never done is open the kernel files those YAML fields become. This lesson closes that gap. Concretely, by the end you should be able to answer each of these cold:

- Given a container's PID, find its leaf cgroup and read the four files that enforce its resources.
- Given a pod spec, predict `cpu.max`, `cpu.weight`, `memory.max`, and the slice it lands in — before touching the node.
- Given a throttling or OOM incident, name the exact file whose counter proves the cause.

The map you're internalizing: `requests.cpu` → `cpu.weight`, `limits.cpu` → `cpu.max`, `limits.memory` → `memory.max`, `requests.memory` → `memory.min` (under MemoryQoS), and the QoS class → *where in the cgroup tree the pod's directory is created*. After this you can `cat` four files under `/sys/fs/cgroup` and reconstruct the pod spec that produced them, and vice versa.

## Core notes

### cgroup v1 vs v2: the unified hierarchy

**cgroups** (control groups) are the kernel mechanism for limiting, accounting for, and isolating the resource usage of a set of processes. They are exposed as a pseudo-filesystem, mounted at `/sys/fs/cgroup`.

**v1** gave each controller its *own* independent hierarchy: `/sys/fs/cgroup/cpu`, `/sys/fs/cgroup/memory`, `/sys/fs/cgroup/cpuset`, etc., each a separate tree. A process could sit in different positions in each tree. This made cross-controller decisions impossible — the memory controller had no idea what the io controller was doing to the same task, so features like coordinated page-cache writeback accounting couldn't work.

**v2** collapses everything into **one unified tree**. A cgroup is a single directory; the controllers you enable on it all act on the same set of processes at the same node. This is why v2 is the default in every modern distro and in Kubernetes ≥ 1.25 (cgroup v2 support went GA there). Systemd on a v2 host boots the machine into the unified hierarchy; `stat -fc %T /sys/fs/cgroup` returns `cgroup2fs` when you're on v2.

Two rules define the v2 tree shape:

- **No-internal-process rule.** A cgroup with the controllers enabled for its children may not itself contain processes (except the root). Processes live in leaf cgroups; interior cgroups are for structure and delegation.
- **Top-down enablement.** A controller only acts on a child if the parent lists it in `cgroup.subtree_control`. `cgroup.controllers` shows what's *available* here; `cgroup.subtree_control` shows what's *pushed down* to children. You enable a controller with `echo "+cpu +memory" > cgroup.subtree_control`.

Two membership files sit in every cgroup: **`cgroup.procs`** lists the PIDs in this cgroup (writing a PID here *moves* the whole process in); **`cgroup.threads`** does the same at thread granularity when threaded mode is on. **Delegation** is how the kubelet — and systemd before it — hands a subtree to a less-privileged manager: you `chown` a cgroup directory and its `cgroup.procs`/`cgroup.subtree_control` to the delegate, who can then create children and move processes *within* the subtree but cannot escape it. Every container runtime relies on this.

Beyond the four controllers below, v2 also carries **`cpuset`** (pin a cgroup to specific CPUs and NUMA memory nodes via `cpuset.cpus` / `cpuset.mems` — this is what Kubernetes' *CPU Manager* `static` policy writes to give Guaranteed pods exclusive integer cores, and it matters a lot on NUMA GPU nodes), plus `hugetlb`, `rdma`, and `misc` (the last is how the NVIDIA device plugin / GPUs are accounted).

### The controllers you care about

| Controller | Governs | Key interface files |
|---|---|---|
| **cpu** | CPU time (proportional share + hard cap) | `cpu.weight`, `cpu.max`, `cpu.stat` |
| **memory** | Memory usage, reclaim, OOM | `memory.max`, `memory.high`, `memory.min`, `memory.current`, `memory.stat` |
| **io** | Block-device bandwidth/IOPS | `io.max`, `io.weight`, `io.stat` |
| **pids** | Number of processes/threads | `pids.max`, `pids.current` |

### CPU: cpu.weight, cpu.max, cpu.stat

**`cpu.weight`** — proportional share, integer `1..10000`, default `100`. When the CPU is *contended*, tasks get CPU time in proportion to their weight. This is the v2 spelling of v1's `cpu.shares`. It is a *floor under contention*, never a cap — a cgroup with weight 100 can use the whole box if nothing else wants it. This is what a Kubernetes **request** maps to.

**`cpu.max`** — the hard cap, written as two numbers: `$QUOTA $PERIOD`, both in microseconds. `50000 100000` means "this cgroup may run for 50,000 µs of CPU time in every 100,000 µs (100 ms) window" = **0.5 CPU**. The default is `max 100000` (no cap). This is what a Kubernetes **limit** maps to, and it is enforced by **CFS bandwidth control**.

**`cpu.stat`** — the evidence file. Fields include `usage_usec`, `user_usec`, `system_usec`, and — when a cap is set — `nr_periods`, `nr_throttled`, and `throttled_usec`. `nr_throttled / nr_periods` is your **throttle ratio**; `throttled_usec` is cumulative wall-time your tasks spent *runnable but blocked* waiting for the next period. This is the "throttled but idle" smoking gun.

#### Why bursty apps throttle at low average utilization

CFS bandwidth control refills the quota every **100 ms period**. Quota is charged as tasks run, *summed across all CPUs*. Consider a limit of 2 CPUs (`cpu.max = 200000 100000`) on a 64-core node running a 32-thread service. When a request arrives, all 32 threads wake and run in parallel. They burn 32 core-ms of quota in ~6 ms of wall time, hit the 200 ms budget, and the kernel **throttles the entire cgroup for the remaining ~94 ms of the period**. Averaged over the 100 ms window, CPU utilization looks like ~40% — but for 94 ms every thread was frozen, so p99 latency explodes.

That is the interview answer: *average* utilization hides the fact that quota is consumed in a burst and then the cgroup sleeps out the rest of the period. Mitigations: raise the limit, reduce app parallelism, or (on older kernels) tune the period. Note the scheduler class changed — since Linux 6.6 the fair scheduler is **EEVDF**, not classic CFS — but the **bandwidth-control interface (`cpu.max`, quota/period, the 100 ms window) is unchanged**, which is why the mechanism is still universally called "CFS throttling."

There is a third, optional field: **`cpu.max.burst`** (Linux ≥ 5.14) lets a cgroup *bank* unused quota from quiet periods and spend it in a later spike, up to the burst budget — smoothing exactly the bursty-latency case above. It defaults to `0` (no banking) and Kubernetes does not expose it as a pod field, but it's worth knowing it exists when someone asks "can't the kernel just let it burst?"

#### GOMAXPROCS vs the cgroup quota (ties to your Go module)

The Go runtime historically set `GOMAXPROCS` (the number of OS threads running goroutines simultaneously) from `runtime.NumCPU()`, which reflects the *machine's* CPU count, **not the cgroup quota**. On that 64-core node with a 2-CPU limit, an unaware Go binary runs `GOMAXPROCS=64`: it schedules goroutines across 64 threads, blows through the 200 ms quota almost instantly, and throttles hard every period. The fix is to set `GOMAXPROCS` from the cgroup CPU limit — historically via `go.uber.org/automaxprocs`; as of **Go 1.25 (Aug 2025) the runtime is cgroup-aware by default** and derives `GOMAXPROCS` from `cpu.max`. The same mismatch bites the JVM, Node, and any thread-pool sized off `nproc`.

### Memory: memory.max, memory.high, memory.current, memory.stat

**`memory.max`** — the **hard limit**. When a cgroup's usage would exceed it and reclaim can't free enough, the kernel invokes the **cgroup OOM killer**, which kills a process *inside that cgroup* (this is the in-container OOMKill, exit code 137). This is what a Kubernetes memory **limit** maps to.

**`memory.high`** — the **soft throttle**. Crossing it does *not* kill anything; instead the kernel puts the offending tasks under heavy reclaim pressure and *throttles their allocations*, slowing them down to force usage back below the threshold. It's the graceful pressure valve you'd want *before* an OOM kill. By default the kubelet does **not** set `memory.high` — a plain memory limit becomes only `memory.max`. `memory.high` (and `memory.min` from requests) is written by the **MemoryQoS** feature (feature gate, alpha/beta depending on version), which uses it to apply reclaim pressure proportional to how far usage has drifted above the request.

When usage climbs past `memory.high`, the kernel runs **direct reclaim** synchronously in the allocating task's context — scanning and evicting reclaimable pages (page cache first, then swapping anon pages if swap is available) and *stalling the allocation* until it succeeds or usage drops. That stall is what "throttle" means here; it shows up as rising `memory.pressure`. If anonymous memory is unreclaimable and there's no swap, pressure builds until `memory.max` is hit and the OOM path fires. **`memory.swap.max`** caps per-cgroup swap; Kubernetes swap support (beta from 1.30) uses it to let Burstable pods overflow to swap under the `LimitedSwap` policy instead of being OOM-killed outright — a real lever for memory-bound cost efficiency.

**`memory.current`** — current charged usage in bytes. **`memory.stat`** — the detailed breakdown: `anon` (anonymous pages — the stuff that gets you OOM-killed), `file` (reclaimable page cache), `kernel_stack`, `slab`, `sock`, etc. **`memory.min`** = hard-protected working set (never reclaimed); **`memory.low`** = best-effort protected.

**`memory.events`** is the memory equivalent of `cpu.stat` — the counters that let you *prove* what happened without guessing from dashboards: `low`, `high` (times allocation was throttled at `memory.high`), `max` (times usage hit `memory.max`), `oom` (times an OOM condition fired), and `oom_kill` (processes actually killed). A container that restarts with exit 137 but `oom_kill 0` here was killed by the *node* OOM killer, not its own cgroup — a distinction that changes your whole diagnosis. This file is the authoritative source for OOM attribution in the capstone.

### Pressure Stall Information (PSI) — the observability lever

Every cgroup exposes **`cpu.pressure`**, **`memory.pressure`**, and **`io.pressure`** (PSI). Each reports the share of wall time tasks were *stalled waiting for that resource*, as `some` (at least one task stalled) and `full` (every runnable task stalled), over 10 s / 60 s / 300 s windows plus a cumulative `total` in microseconds. This is strictly better than utilization for your cost/observability differentiator: a cgroup at 40% average CPU with `cpu.pressure some avg10=85` is *starved* (this is the throttling paradox, quantified), whereas 95% CPU with near-zero pressure is *healthy saturation*. PSI is what modern autoscalers and OOM-preventers (e.g. `systemd-oomd`) act on, and reading it per-`kubepods` slice is how you turn "is this pod resource-starved?" from a vibe into a number.

### pids and io

**`pids.max` / `pids.current`** cap and count tasks — the defense against fork bombs. `pids.max` is a *hard* ceiling: a `fork()`/`clone()` past it fails with `EAGAIN`, which surfaces in apps as "resource temporarily unavailable" or thread-creation panics rather than an OOM, so it's easy to misdiagnose. Kubernetes exposes this via the kubelet's `podPidsLimit` (per-pod) and `--system-reserved`/node PID reservations. **`io.max`** sets per-device read/write bytes-per-second and IOPS caps (`MAJ:MIN rbps=... wbps=... riops=... wiops=...`); **`io.weight`** is the proportional share; **`io.pressure`** (PSI) tells you when tasks stall on disk. Kubernetes has no first-class pod field for block-IO limits yet, but the plumbing is here, and the io controller only accounts accurately for direct/writeback IO when the memory controller is co-enabled — a reason v2's unified hierarchy exists at all.

### What the container sees vs. the host

A container process reads `nproc` / `/proc/cpuinfo` and sees the **host's** CPU count, and `/proc/meminfo` shows **host** memory — because a **cgroup namespace** only virtualizes the *view of the cgroup tree* (so the container sees its own cgroup as `/`), not `/proc`'s hardware numbers. This is the root cause of the GOMAXPROCS/JVM/thread-pool mismatch above: userspace sizes itself off host hardware while the kernel enforces the cgroup quota. Runtimes must read `cpu.max` themselves (or trust a cgroup-aware runtime) to reconcile the two.

### Node allocatable: where the cost boundary is drawn

`kubepods.slice` is not the whole machine — it is **node allocatable** = machine capacity minus `--kube-reserved` (kubelet, runtime) and `--system-reserved` (sshd, systemd, etc.), which live in their own `system.slice`. So the CPU/memory sum available to pods is fenced at the `kubepods.slice` level, and every pod's usage is a leaf under it. That is precisely why cgroup accounting *is* cost accounting: walk `kubepods.slice` → QoS slice → pod slice → container scope and you have a complete, kernel-authoritative attribution of a node's spend, with no gaps and no double-counting.

### How the kubelet builds the tree

With the **systemd cgroup driver** (the recommended default), the kubelet builds a slice hierarchy under the unified root:

```
/sys/fs/cgroup/
└── kubepods.slice/                          # node allocatable pool; root of pod accounting
    ├── <Guaranteed pods live directly here> # e.g. kubepods-pod<uid>.slice
    ├── kubepods-burstable.slice/            # all Burstable pods
    │   └── kubepods-burstable-pod<uid>.slice/
    │       └── cri-containerd-<id>.scope/    # one scope per container → the leaf cgroup
    └── kubepods-besteffort.slice/           # all BestEffort pods
        └── kubepods-besteffort-pod<uid>.slice/
```

(With the older **cgroupfs** driver the same shape appears as plain directories: `kubepods/`, `kubepods/burstable/`, `kubepods/besteffort/`.) The leaf `*.scope` is the container's own cgroup — that's where you read `cpu.max`, `memory.max`, `cpu.stat`. Match the driver to the runtime's: a systemd-cgroup runtime on a cgroupfs kubelet (or vice versa) gives you *two* views of memory and endless "why is my pod being OOM-killed" mysteries — the classic v2 misconfiguration.

Files you'll reach for, by level:

```
kubepods.slice/                         cpu.max, memory.max  → node-allocatable envelope
  kubepods-burstable.slice/             cpu.weight sums      → per-QoS-tier share
    kubepods-burstable-pod<uid>.slice/  cpu.max, memory.max  → the POD's aggregate limits
      cri-containerd-<id>.scope/        cpu.max, cpu.weight, cpu.stat,
                                        memory.max, memory.current, memory.events,
                                        *.pressure           → the CONTAINER (what you read)
```

### QoS classes → tree shape and file values

The QoS class is *derived*, not declared, and it determines both **where** the pod's cgroup is created and **what** gets written:

- **Guaranteed** — every container sets `requests == limits` for *both* CPU and memory. The pod cgroup is placed **directly under `kubepods.slice`** (no QoS sub-slice). Its `cpu.max` and `memory.max` are set tight to the limits, and its `cpu.weight` reflects the request. Tightest fence, highest protection.
- **Burstable** — at least one request/limit is set, but not full Guaranteed parity. Placed under **`kubepods-burstable.slice`**. `cpu.weight` from the CPU request; `cpu.max` from the CPU limit *if one is set* (else `max`, uncapped); `memory.max` from the memory limit if set.
- **BestEffort** — no requests or limits at all. Placed under **`kubepods-besteffort.slice`**. `cpu.weight` is the minimum, `cpu.max` is `max`, `memory.max` is `max`. First to be throttled under contention and first to be evicted/OOM-killed.

**Why the tree shape matters for eviction/OOM order.** Under node memory pressure the kubelet evicts **BestEffort first, Burstable next, Guaranteed last** — and the kernel OOM killer is biased the same way (Guaranteed containers get a protective `oom_score_adj` near the floor, BestEffort near the ceiling). The separate `besteffort`/`burstable` slices make this a structural property of the tree, not just a per-process hint: an entire QoS tier can be reclaimed as a unit, and CPU share flows down the slices so a Guaranteed pod under contention keeps its weight while BestEffort collapses to the minimum.

### The mapping, condensed

| Pod spec field | cgroup file (container leaf) | Semantics |
|---|---|---|
| `requests.cpu` | `cpu.weight` | proportional floor under contention |
| `limits.cpu` | `cpu.max` (`quota period`) | hard cap via CFS bandwidth control |
| `requests.memory` | `memory.min` *(MemoryQoS)* | protected working set |
| `limits.memory` | `memory.max` | hard cap → cgroup OOM kill |
| *(MemoryQoS)* | `memory.high` | soft throttle + reclaim before OOM |
| QoS class | *tree position* (slice) | eviction/OOM tier |

The `requests.cpu → cpu.weight` conversion goes through shares: `shares = requests_millicpu * 1024 / 1000`, then v2 rescales `[2, 262144] → [1, 10000]` as `weight = 1 + (shares - 2) * 9999 / 262142`. So `requests.cpu: 500m` → shares 512 → **`cpu.weight ≈ 20`**; `requests.cpu: 1` → shares 1024 → **`cpu.weight ≈ 39`**.

**One request, two jobs.** `requests.cpu` does double duty and it's worth keeping the two apart. At **schedule time** the kube-scheduler treats the request as a bin-packing reservation — it only places the pod on a node whose *unrequested* allocatable CPU covers it, and the request never changes after placement. At **run time** the same request becomes `cpu.weight`, which does nothing until the node is contended and then divides spare CPU proportionally. So a request is simultaneously a *scheduling promise* (capacity is set aside) and a *runtime priority* (weight under contention) — but it is **not** a runtime cap. Only `limits.cpu → cpu.max` caps you. Confusing the two is the source of most "I requested 2 cores, why is it only getting 0.5?" tickets: the answer is contention plus weight, not the request being violated.

## Worked example

Run a container with a half-CPU cap and a 256 MiB memory cap, then read its cgroup by hand.

```bash
docker run -d --name cg-demo --cpus=0.5 --memory=256m \
  stress-ng:latest --cpu 4 --timeout 600s        # 4 CPU-burners, 0.5 CPU cap

# Find the container's leaf cgroup (systemd driver path shown):
CID=$(docker inspect -f '{{.Id}}' cg-demo)
CG=$(grep -m1 '' /proc/$(docker inspect -f '{{.State.Pid}}' cg-demo)/cgroup | cut -d: -f3)
echo "$CG"          # e.g. /system.slice/docker-<id>.scope
BASE=/sys/fs/cgroup$CG
```

Read the enforcement files:

```bash
cat $BASE/cpu.max
# 50000 100000            # quota 50ms / period 100ms = 0.5 CPU  ✔ matches --cpus=0.5

cat $BASE/cpu.weight
# 100                     # default; --cpus doesn't change weight, only the cap

cat $BASE/memory.max
# 268435456               # 256 * 1024 * 1024  ✔ matches --memory=256m

cat $BASE/memory.current
# 5849088                 # ~5.6 MiB actually used right now

cat $BASE/cpu.stat
# usage_usec 41230000
# user_usec 40010000
# system_usec 1220000
# nr_periods 612
# nr_throttled 598        # throttled in 598 of 612 periods → 97% throttle ratio
# throttled_usec 55082140 # ~55s spent runnable-but-frozen
```

Four stress threads want 4 CPUs; the cap is 0.5, so they burn the 50 ms quota in ~12.5 ms and sit throttled for ~87 ms every period — `nr_throttled` nearly equals `nr_periods`. **This is the "throttled but idle" signature**: the box has spare cores, the app is runnable, but `cpu.max` freezes it.

Now change the cap live and watch throttling ease:

```bash
before=$(awk '/nr_throttled/{print $2}' $BASE/cpu.stat)
echo "100000 100000" > $BASE/cpu.max        # raise cap to 1.0 CPU
sleep 5
after=$(awk '/nr_throttled/{print $2}' $BASE/cpu.stat)
echo "throttled periods in last 5s: $((after - before))"
# with 4 threads still contending for a 1-CPU cap, the delta stays high;
# echo "400000 100000" (4 CPUs) makes the delta drop to ~0 — quota now covers demand
```

Reading `cpu.stat` before/after any `cpu.max` change is exactly how you prove a throttling hypothesis on a real node. Cross-check with PSI — `cat $BASE/cpu.pressure` will show `some avg10` climbing toward 90+ while throttled, then falling once the cap covers demand.

**Same thing for a real pod.** Given a running pod, find its container's cgroup without guessing the slice path:

```bash
CID=$(crictl ps --name my-app -q)
PID=$(crictl inspect $CID | jq '.info.pid')
CG=$(cut -d: -f3 /proc/$PID/cgroup)          # v2 is a single line
cat /sys/fs/cgroup$CG/cpu.max                 # → e.g. 50000 100000 for limits.cpu: 500m
cat /sys/fs/cgroup$CG/memory.max              # → limits.memory in bytes
cat /sys/fs/cgroup$CG/memory.events           # → oom_kill count for this container
# The slice above the scope encodes QoS, e.g. .../kubepods-burstable-pod<uid>.slice/...
```

## Practice

For one running container (a `docker run` you control, or a real pod's container), reconstruct its **full cgroup resource picture by hand** and map every value back to the spec that produced it.

1. Locate the leaf cgroup via `/proc/<pid>/cgroup` (or `crictl inspect` for a pod) under `/sys/fs/cgroup`.
2. `cat` and record: `cpu.max`, `cpu.weight`, `memory.max`, `memory.high` (may be `max`), `memory.current`, `cpu.stat`.
3. Deliberately induce throttling: pin the CPU (`stress-ng --cpu N` or a busy loop with more threads than the cap) and capture `nr_throttled` / `throttled_usec` climbing.
4. Build the acceptance artifact — a **mapping table** for that one real container:

| Spec value | cgroup file | Observed value | Enforcement observed |
|---|---|---|---|
| `--cpus=0.5` / `limits.cpu: 500m` | `cpu.max` | `50000 100000` | `nr_throttled` climbs under load |
| `requests.cpu: 500m` | `cpu.weight` | `~20` | share honored only under contention |
| `--memory=256m` / `limits.memory: 256Mi` | `memory.max` | `268435456` | usage capped; OOM at ceiling |
| (QoS class) | tree position | `kubepods-burstable-...` slice | eviction tier |

5. Write one paragraph mapping *each* observed value back to the spec field and the enforcement you saw with your own eyes (throttle counters climbing, `memory.pressure` rising, an OOM in `memory.events`). "The spec says X, the kernel file says Y, and under load I observed Z" — that sentence, said fluently, is what the interview is testing.

Feed this table straight into the **Anatomy of a Container** deliverable — it is the resource-enforcement section of that write-up.

### Failure-signature quick reference

Keep this table; it's the fast path from a symptom to the exact file that proves the cause.

| Symptom | Read this | Confirms |
|---|---|---|
| p99 latency spikes, CPU dashboard looks fine | `cpu.stat` `nr_throttled`, `cpu.pressure` | CFS throttling against `cpu.max` |
| Container restarts, exit 137 | `memory.events` `oom_kill` | cgroup OOM at `memory.max` (own limit) |
| Exit 137 but `oom_kill 0` here | node OOM logs / kubelet eviction | *node* pressure killed it, not its limit |
| App slow, no throttle, high mem | `memory.pressure`, `memory.events` `high` | reclaim throttling at `memory.high` |
| "resource temporarily unavailable" | `pids.current` vs `pids.max` | PID limit, not memory |
| One pod starves neighbors under load | `cpu.weight` across sibling slices | proportional-share imbalance |

## Self-check

**Q1. A pod sets `limits.cpu: 500m`. What exact value appears in `cpu.max`, and how does CFS bandwidth control throttle it at low average CPU?**
**Answer:** `cpu.max` becomes `50000 100000` — quota 50,000 µs per 100,000 µs (100 ms) period = 0.5 CPU. The quota is charged as tasks run, *summed across all cores*. A multi-threaded app can consume all 50 ms of quota in a few ms of wall-clock time by running on several cores at once, after which the kernel throttles the whole cgroup for the rest of the 100 ms period. Averaged over the window this reads as low utilization (~40%), yet the app spent most of each period frozen — visible as `nr_throttled ≈ nr_periods` and a large `throttled_usec` in `cpu.stat`.

**Q2. What is the difference between `memory.high` and `memory.max`, and which one does a Kubernetes memory *limit* set?**
**Answer:** `memory.max` is the hard limit — cross it and, if reclaim can't recover, the **cgroup OOM killer** kills a process inside the cgroup (in-container OOMKill, exit 137). `memory.high` is a soft throttle — crossing it triggers aggressive reclaim and *slows allocations* but never kills. A Kubernetes memory **limit** sets **`memory.max`**. `memory.high` is written only by the **MemoryQoS** feature; by default the kubelet leaves it at `max`.

**Q3. How does Guaranteed QoS shape the cgroup tree differently from Burstable, and why does that matter for eviction/OOM order?**
**Answer:** A Guaranteed pod (all containers `requests == limits` for CPU and memory) gets its cgroup **directly under `kubepods.slice`** with `cpu.max`/`memory.max` pinned tight to the limits. A Burstable pod is placed under the **`kubepods-burstable.slice`** subtree with a cap only where a limit is set. Because BestEffort and Burstable pods live in their own slices, the kubelet evicts by tier — **BestEffort first, Burstable next, Guaranteed last** — and the kernel OOM killer's `oom_score_adj` bias matches. The tree position *is* the priority: a whole tier can be reclaimed as a unit while Guaranteed pods keep their fenced resources.

## Resources

1. **Control Group v2 — kernel admin guide** — https://docs.kernel.org/admin-guide/cgroup-v2.html — the authoritative reference for every interface file (`cpu.max`, `cpu.weight`, `memory.max/high/min`, `io.max`, delegation rules). *Deep / reference — read the CPU and Memory controller sections closely; this is ground truth.*
2. **Kubernetes: Resource Management for Pods and Containers** — https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/ — how requests/limits/QoS are declared and what they mean operationally. *Skim to confirm the spec side, then map each field to the kernel file above.*
3. **Kubernetes: About cgroup v2** — https://kubernetes.io/docs/concepts/architecture/cgroups/ — how the kubelet uses the unified hierarchy, the systemd vs cgroupfs driver, and MemoryQoS. *Skim — the bridge between the two references above.*
4. **CPU throttling / GOMAXPROCS-vs-quota deep-dive** — https://victoriametrics.com/blog/kubernetes-cpu-requests-limits/ (and Uber's `automaxprocs` README) — real-world walkthrough of CFS-period throttling and the runtime-parallelism mismatch. *Deep — this is the interview-answer article; confirms the "throttled at 40%" mechanism end to end.*
