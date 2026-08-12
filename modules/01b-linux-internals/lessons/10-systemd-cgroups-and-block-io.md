---
lesson: "01b.10"
title: "systemd as cgroup manager, and the block-I/O path"
module: "01b"
concept: "systemd as cgroup manager, and the block-I/O path"
status: not-started
est_time: "5h"
prev: "09-perf-ftrace-use.md"
next: null
artifacts: []
sources: 5
---

# 01b.10 · systemd as cgroup manager, and the block-I/O path

> **Concept.** How systemd owns and delegates the cgroup tree (slices/scopes/services, delegation to the kubelet), and the block-I/O path read through io PSI and latency tools.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 9 gave you the methodology (USE) and the tools (perf, off-CPU flame graphs) to *find* that a node's disk is the bottleneck — the worked example ended with a training job blocked on `nfs_readpage`. What it didn't answer is *whose* cgroup that I/O belongs to, who is allowed to write the limit that would bound it, and why the kubelet and PID 1 don't stomp on each other's files while both are managing cgroups on the same node. This lesson closes that gap: it's the module's last lesson, and it ties the cgroup-file mechanics from lesson 3 (`cpu.max`, `memory.max`) to the process that actually owns the tree those files live in — systemd — and finishes the block-I/O path lesson 9 only diagnosed from the outside.

## Why this matters

On a cgroup-v2 host, exactly one process is allowed to be the writer of the cgroup tree, and on almost every modern distro that process is `systemd` (PID 1). Every resource limit you have ever set on a node — the kubelet's `--kube-reserved`, a pod's `MemoryMax`, the CPU throttle you saw in lesson 3 — is ultimately a file under `/sys/fs/cgroup` that systemd either wrote or *delegated* the right to write. If you don't understand that ownership model, node pressure looks like magic: pods get OOM-killed inside a slice you didn't know existed, or the kubelet fails to start because it can't create its own subtree. And when a single slow disk on a GPU box stalls "unrelated" pods, the signal that explains it — io PSI — lives in the same cgroup tree. This lesson connects the systemd front-end you already use to the kernel mechanism underneath.

## What's new here (calibration)

Per the [module README](../README.md)'s calibration, you already know shell/systemd unit basics — writing a unit file, `systemctl start/enable`, reading `journalctl` — none of that is re-taught here. What's genuinely new:

- Seeing systemd not as "the init system" but as **the userspace front-end to the cgroup tree** — every directive you write resolves to a specific file you already met in lesson 3.
- The **delegation model**: why two processes (systemd and the kubelet) can both legally own parts of the same tree without racing, and what breaks when delegation is misconfigured.
- **io PSI and iocost** as the fleet-scale answer to "device-centric metrics lie" — extending lesson 9's saturation-over-utilization lesson to the block layer specifically, plus the *cost-model* refinement (iocost) that raw bandwidth caps don't give you.
- The **GPU-fleet-specific** framing: an unpartitioned shared NVMe device on a multi-tenant node is a node-level fairness problem structurally identical to the conntrack exhaustion in lesson 7, just at the block layer instead of the network layer.

## Core concepts

**Three unit types map to three roles in the tree.**
- **service** — processes systemd *forks itself* from a unit file (`ExecStart=`). Gets a cgroup named `foo.service`.
- **scope** — processes started by *someone else* that systemd only *adopts* into a cgroup (e.g. a login session, a `systemd-run --scope`, a container the runtime forked). systemd never forked them; it just groups and accounts them. Named `foo.scope`.
- **slice** — a *branch node*, holds no processes of its own; it exists to group services and scopes and to apply resource limits to the whole subtree. Named `foo.slice`, and the name encodes the path: `a-b.slice` lives at `a.slice/a-b.slice`.

**The default slice tree.** The root splits into a few top-level slices:
- `system.slice` — system daemons (most `*.service` units).
- `user.slice` — one `user-<UID>.slice` per logged-in user, session scopes underneath.
- `machine.slice` — VMs/containers registered via `machined`.
- `init.scope` — PID 1 itself.

On a Kubernetes node with the `systemd` cgroup driver, the kubelet adds `kubepods.slice` (usually directly under root, or under a configured parent), with `kubepods-besteffort.slice` and `kubepods-burstable.slice` beneath it for the non-Guaranteed QoS classes, and one scope per container under those. This is why `systemd-cgls` on a node shows a clean split between `system.slice` (kubelet, containerd, sshd) and `kubepods.slice` (the workloads).

**`system.slice` is not automatically fenced.** systemd will happily let `system.slice` (the kubelet, containerd, sshd, monitoring agents, node-level daemons) consume memory and CPU without any built-in ceiling — systemd's job is to *account* what ends up under each slice, not to decide how big `system.slice` is allowed to get. The boundary between "what the OS gets" and "what's allocatable to pods" is drawn by the kubelet's own `--system-reserved`/`--kube-reserved` flags (the node-allocatable arithmetic from lesson 3): those flags tell the kubelet how much to subtract from the node's capacity before it schedules pods, and *separately* they're commonly wired to actually set `MemoryMax`/`CPUQuota` on `system.slice` (or a dedicated reserved slice) so the reservation is enforced, not just budgeted on paper. Skip that wiring and `--kube-reserved` is an accounting fiction — nothing stops a runaway system daemon from eating into what you promised pods.

**Resource-control directives write the same files from lesson 3.** These are *directives*, resolved to cgroup files:
- `CPUQuota=20%` → `cpu.max` (e.g. `20000 100000`).
- `CPUWeight=` / `IOWeight=` → `cpu.weight` / `io.weight` (proportional share, 1–10000, default 100).
- `MemoryMax=` / `MemoryHigh=` → `memory.max` (hard, triggers OOM) / `memory.high` (throttle-and-reclaim, no kill).
- `IOReadBandwidthMax=/dev/x 50M` → `io.max` for that device's major:minor.
Set them statically in a unit, or live with `systemctl set-property foo.service MemoryMax=200M` — which writes the file *now* and drops a drop-in under `/etc/systemd/system/foo.service.d/` so it survives reboot. `--runtime` makes it live-only.

**Delegation — why the kubelet needs `Delegate=yes`.** cgroup-v2 enforces the *no-internal-process* rule (a cgroup with child cgroups can't also hold processes and enable controllers on them) and a single-writer discipline. If both systemd and the kubelet tried to manage the same subtree, they'd race — systemd's periodic reconciliation would clobber the kubelet's files. `Delegate=yes` in the kubelet/runtime unit tells systemd: *this subtree is yours; I will create the top cgroup and enable controllers, then keep my hands off everything below it*. systemd also makes the delegated cgroup writable by the unit's user and enables the requested controllers (`Delegate=cpu memory io pids`). Inside the delegated `kubepods.slice`, the kubelet freely `mkdir`s per-pod cgroups and writes `memory.max` etc. Without delegation the kubelet gets `EPERM`/`EBUSY` or has its writes silently reverted. (See systemd's CGROUP_DELEGATION doc — delegation is only safe on cgroup-v2; the v1 story was fragile.)

**Tools.**
- `systemd-cgls` — the live tree as a directory listing (slices → scopes/services → PIDs).
- `systemd-cgtop` — `top` for cgroups: per-slice CPU%, memory, and I/O, refreshing live. Fast way to find *which slice* is burning a resource.
- `systemctl set-property` — write a directive to a live unit + persist it.
- `systemd-run` — start a transient unit (`--scope` to run in your shell adopted into a new scope, or default to a forked service) with `-p Directive=…` to apply limits without writing a unit file. Ideal for one-off constrained runs.

**Block I/O path, briefly.** A read/write goes: filesystem → page cache (may be satisfied here, no device I/O) → block layer → **I/O scheduler / queue** (`mq-deadline`, `bfq`, or `none` for NVMe multi-queue) → device driver → disk. The `io` cgroup controller sits at the block layer and enforces `io.weight` (proportional) and `io.max` (absolute bps/iops caps) per cgroup — but only on I/O that actually reaches the device (buffered writes are accounted at writeback, direct/read I/O synchronously).

**Beyond raw caps: iocost.** `io.max`/`io.weight` bound *raw* bytes-per-second or IOPS, which is a blunt instrument — a sequential 128K write and a random 4K write cost the device wildly different amounts of real service time, so a raw-bandwidth cap either starves sequential workloads or lets random-I/O workloads still dominate the device's actual capacity. **`io.cost.qos`/`io.cost.model`** (the `iocost` controller, `blk-iocost`, upstreamed by Meta and merged in Linux 5.4) instead builds a cost model per device — estimating the *true service cost* of each I/O based on size and pattern (sequential vs random) — and does proportional, work-conserving allocation against that cost rather than against raw bytes. It's the fleet-scale answer to "my `io.weight` numbers don't actually reflect fairness under real, mixed I/O patterns." See Meta's resctl docs in Real-world use cases below for the production motivation.

**`io.pressure` vs `iostat`.** `iostat -x` gives *device*-centric numbers: `%util` (fraction of time the device had at least one request in flight) and `await` (average request latency). On a modern multi-queue NVMe drive `%util` is nearly useless — the device serves many requests in parallel, so it can be pinned at 100% while far from saturated, or saturated below 100% — the identical NVMe multi-queue trap from lesson 9's USE grid, here at the cgroup layer instead of the whole-device layer. What you actually want is *how much did tasks stall waiting for I/O*. That is **io PSI**: `cat /sys/fs/cgroup/<path>/io.pressure` reports `some` (at least one task stalled on I/O) and `full` (all runnable tasks stalled) as running % over 10s/60s/300s windows. PSI measures *lost work due to resource contention*, per-cgroup, which `await`/`%util` cannot: a rising `io.pressure some avg10` on a pod's slice tells you that pod is losing wall-clock time to disk regardless of what the device counters say.

**`biolatency` (BCC).** Attaches to block-layer tracepoints and prints a *histogram* of I/O completion latency (queue-insert → completion). Averages lie about disks — a p50 of 0.2ms with a p99.9 of 200ms is a very different node than a flat 5ms. `biolatency -D` splits by device; `-Q` includes OS queue time. Use it to see the *shape* of the latency distribution behind an io-pressure spike.

**Why a slow disk stalls "unrelated" pods.** Device bandwidth/IOPS is a shared, finite resource with no strict per-cgroup isolation unless you set `io.max`/`io.weight` (or `iocost`). One pod doing heavy writes fills the device queue; every other pod's reads now sit behind it. Those victims show *high `io.pressure`* even though *they* did nothing wrong — the stall is contention, not their own I/O volume. Node-level `io.pressure` rising while one slice dominates `systemd-cgtop`'s I/O column is the classic noisy-neighbor signature on a shared disk.

## Perspectives

**Kernel-mechanism view.** cgroup-v2 enforces two structural rules that make the whole delegation story possible: the no-internal-process constraint (a cgroup that has child cgroups and enabled controllers can't itself hold tasks) and single-writer discipline per subtree. Delegation is systemd exploiting those rules deliberately — it creates a cgroup, turns on the controllers the delegate needs, marks it writable, and then simply stops touching anything below that boundary. There's no IPC, no negotiation protocol between systemd and the kubelet; the kernel's own filesystem permissions and the no-internal-process rule are what keep two managers from corrupting each other's writes.

**Operator/SRE view.** When a node looks pressured, the first move isn't `top` — it's `systemd-cgtop`, because `top` gives you a flat process list with no ownership boundary, while `systemd-cgtop` sorts by slice and immediately answers "which tenant is the hog," CPU/memory/IO in one live view. `systemd-cgls` is the complementary "where does this PID actually live in the tree" lookup. Together they're the fastest path from "node-47 is pressured" to "it's `kubepods-burstable.slice`, specifically this one pod's scope" — before you've touched a single per-process tool.

**GPU-fleet-specific view.** A multi-tenant GPU node commonly has one shared, unpartitioned NVMe device (or a shared network filesystem mount) backing every pod's checkpoint writes and every other pod's data-loader reads. Without `io.max`/`io.weight`/`iocost` explicitly configured, one job's large sequential checkpoint write can fill the device queue and stall every other job's small random data-loader reads on the *same node* — even though those jobs have nothing to do with each other and neither breached any per-pod quota. This is the **same node-level shared-resource fairness problem as lesson 7's conntrack table exhaustion**: a single finite kernel-managed resource (there, the conntrack hash table; here, the block device's queue) with no default per-tenant isolation, so one workload's legitimate use degrades every neighbor's. The fix pattern is the same shape too — explicit per-cgroup limits (`io.max`/`io.weight`/`iocost` here, `nf_conntrack_max` tuning and eBPF datapaths there) rather than trusting the shared resource to self-arbitrate.

**Economics/architecture view.** The single-writer/delegation model isn't just a cgroup-v2 implementation detail — it's the general argument for *why* two independently-developed orchestrators (systemd managing the OS, kubelet managing pods) can coexist safely on one node without a coordination protocol between their teams. Neither has to trust the other's internals; they only have to agree on a filesystem boundary and a permission bit (`Delegate=yes`). That's a reusable lesson for any shared kernel resource with multiple potential managers: draw an explicit ownership boundary and hand write access to exactly one side of it, rather than building distributed coordination on top of a resource that was never designed to be co-owned.

## Real-world use cases

- **[Meta's resctl / iocost documentation](https://facebookmicrosites.github.io/resctl-demo-website/docs/demo_docs/res_protection/IO/)**. Meta built and upstreamed `blk-iocost` (merged into Linux 5.4) as a work-conserving, proportional I/O controller because raw `io.max`/`io.weight` bandwidth caps weren't good enough for real fleet-scale noisy-neighbor disk contention — the io-cost model estimates the true cost of each I/O (sequential vs random, size-scaled) rather than just capping raw bytes/IOPS. What it shows: this lesson's `io.weight`/`io.max` section is the *first* generation of the tool; iocost is the production evolution once raw caps proved insufficient at scale.
- **Cgroup-driver mismatch as a recurring operational failure pattern.** A `cgroupfs`-driver kubelet running on a `systemd`-managed host creates two independent cgroup managers with two different views of the same resources on one node — a well-documented, recurring class of node instability across the Kubernetes operator community, not a one-off bug in any single company's stack. No single verified company postmortem blog was found describing this specific pattern for this lesson, so it's cited here as Kubernetes' own documented operational guidance rather than a production war story (see the Kubernetes container-runtimes docs in Primary sources below, which explicitly warns about exactly this "two cgroup managers... can lead to instability" scenario). What it shows: delegation done *wrong* — two writers instead of one — is not a hypothetical; it's common enough that the orchestrator's own docs call it out by name.

## Worked example

See the live tree and find a hog, then create a constrained transient scope and confirm the file it wrote.

```bash
# 1. The tree, as systemd sees it
systemd-cgls --no-pager | head -40
#   -.slice
#   ├─kubepods.slice
#   │ └─kubepods-burstable.slice
#   │   └─kubepods-burstable-pod<uid>.slice
#   │     └─cri-containerd-<id>.scope → 12345 /app
#   ├─system.slice
#   │ ├─kubelet.service → 987 /usr/bin/kubelet
#   │ └─containerd.service → 654 /usr/bin/containerd
#   └─user.slice/user-1000.slice/session-3.scope

systemd-cgtop      # live CPU/MEM/IO per slice; press to sort by I/O

# 2. A constrained transient scope — no unit file needed
systemd-run --scope -p MemoryMax=100M --unit demo-mem \
    stress-ng --vm 1 --vm-bytes 150M --timeout 20s
# stress-ng gets adopted into demo-mem.scope; touching 150M > 100M → OOM inside the scope

# 3. Confirm systemd wrote the cgroup file
cat /sys/fs/cgroup/system.slice/demo-mem.scope/memory.max   # → 104857600  (100M)
cat /sys/fs/cgroup/system.slice/demo-mem.scope/memory.events # oom / oom_kill counters tick

# 4. Live-limit an existing unit (writes memory.max now + persists a drop-in)
systemctl set-property containerd.service MemoryHigh=2G
```

Now read the block-I/O side under load — this is the cgroup-attribution follow-up to lesson 9's off-CPU `nfs_readpage` finding:

```bash
# generate device I/O (bypass page cache so it hits the disk)
fio --name=load --filename=/var/tmp/f --size=2G --rw=randwrite \
    --bs=64k --direct=1 --numjobs=4 --time_based --runtime=30 &

iostat -x 2                # watch await climb; note %util is misleading on NVMe
cat /sys/fs/cgroup/io.pressure                 # some avg10 rising = real stall
biolatency -D 5 1          # per-device latency histogram; look at the tail buckets
```

`iostat` tells you the *device* is busy; `io.pressure` tells you *work is being lost*; `biolatency` tells you the *latency distribution* — three different questions, and only the middle one is per-cgroup, which is what lets you point at the *specific* noisy-neighbor slice rather than just "the disk."

## Practice

1. **Map the tree.** Run `systemd-cgls` on your machine (or a node). Identify one `*.slice`, one `*.service`, and one `*.scope`, and for each, note the corresponding directory under `/sys/fs/cgroup`. Confirm the `a-b.slice` → `a.slice/a-b.slice` path encoding on a nested slice.
2. **Constrained scope.** Use `systemd-run --scope -p MemoryMax=…` (or `-p CPUQuota=…`) to run any workload in a transient scope. `cat` the matching `memory.max`/`cpu.max` in that scope's cgroup dir and verify the number matches your directive. Trigger the limit (exceed memory, or a CPU spinner) and read `memory.events`/`cpu.stat`.
3. **I/O pressure.** Put the disk under load with `dd if=/dev/zero of=./big bs=1M count=4096 oflag=direct` or `fio`. While it runs, capture `io.pressure` (root and, if on a node, a workload slice) and either a `biolatency` histogram or `iostat -x`.
4. **Noisy-neighbor simulation (GPU-fleet framing).** Run two workloads concurrently sharing one disk: a heavy sequential writer (simulating a checkpoint write) and a light random reader (simulating a data-loader). Capture `io.pressure` on the reader's own cgroup while the writer runs unconstrained, then again after setting `IOWeight=` or `IOReadBandwidthMax=` on the writer's transient scope. Note the change in the reader's `io.pressure some avg10`.

**Acceptance:** a short note tracing one **systemd slice → cgroup path → resource file** (name all three), plus **one io-pressure observation** (the `some avg10` value under load and what it implied vs what `%util` showed), plus the **noisy-neighbor before/after** from task 4 (io.pressure with and without an `IOWeight`/`IOReadBandwidthMax` constraint). Feeds the *Anatomy of a Container* deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md).

## Common pitfalls

1. **Assuming systemd and the kubelet fight over the same cgroup files.** They don't — *if* delegation (`Delegate=yes`) is correctly configured on the kubelet/container-runtime unit. Without it, you get `EPERM`/`EBUSY` errors or writes that get silently reverted on systemd's next reconciliation pass, which looks like flaky, unexplainable cgroup behavior rather than a config bug.
2. **Trusting `iostat` `%util`/`await` as the saturation signal for a shared device.** Same NVMe multi-queue trap as lesson 9 — a device can show 100% `%util` while far from saturated. `io.pressure`, read per-cgroup, is the correct saturation signal, and it's the only one of the three that tells you *whose* workload is losing wall-clock time.
3. **Assuming a slice's resource directive takes effect only after reboot.** `systemctl set-property` writes the cgroup file *live* by default — the limit applies immediately — and only additionally persists a drop-in unit for future boots. `--runtime` skips persistence if you explicitly want a live-only change.
4. **Believing a `*.scope` is created/owned like a `*.service`.** A service is a process systemd itself forked from `ExecStart=`. A scope is externally-created processes (a login session, a container runtime's child, `systemd-run --scope`) that systemd only *adopts* into a cgroup for accounting — systemd never forked them and doesn't manage their lifecycle the way it does a service's.
5. **Overlooking that block-device bandwidth has no isolation unless configured.** Unlike CPU (CFS gives every cgroup a fair share by default) or memory (the OOM killer eventually intervenes), an unconfigured shared disk has no default per-cgroup fairness — `io.max`/`io.weight`/`iocost` are opt-in. A node with none of them set is a wide-open noisy-neighbor surface by default, not a safely-isolated one.

## Self-check

**Q1. How does systemd delegate a cgroup subtree to the kubelet, and why is `Delegate=yes` required?**
**Answer:** systemd creates the top cgroup for the unit (e.g. `kubepods.slice`), enables the requested controllers on it, makes it writable, and then — because `Delegate=yes` is set — stops managing anything *below* it. The kubelet then owns that subtree: it `mkdir`s per-pod/per-container cgroups and writes their `memory.max`/`cpu.max`/`io.max`. `Delegate=yes` is required because cgroup-v2 enforces a single-writer discipline; without it systemd's periodic reconciliation would revert the kubelet's writes or deny them (`EPERM`/`EBUSY`), since two managers can't own the same cgroup files.

**Q2. What does `io.pressure` tell you that `iostat` `await`/`%util` does not?**
**Answer:** PSI measures *lost work* — the running percentage of time tasks were stalled waiting for I/O (`some` = at least one task blocked, `full` = all runnable tasks blocked) — and it does so *per cgroup*. `await` is device-average request latency and `%util` is device busy-time, both device-centric and, on parallel NVMe queues, poor saturation signals (`%util` can sit at 100% while the device is far from saturated). Only `io.pressure` answers "how much wall-clock did *this pod* lose to I/O contention," which is what you need to blame a noisy neighbor.

**Q3. scope vs slice vs service — what's each for?**
**Answer:** A **service** is processes systemd forked itself from a unit file (`ExecStart=`). A **scope** is externally-created processes systemd only adopted and accounts (login sessions, `systemd-run --scope`, container runtime children). A **slice** is a branch node that holds no processes of its own — it exists to group services/scopes and to apply resource limits to the whole subtree (and its name encodes its path in the hierarchy).

**Q4. Why can systemd and the kubelet both manage cgroups on the same node without racing, and what's the general principle behind it?**
**Answer:** They don't operate on the same files — delegation carves an explicit ownership boundary (a subtree, `kubepods.slice`) and hands write access to exactly one side (the kubelet), while systemd continues to own and reconcile everything outside that boundary. Neither manager needs to know about the other's internal logic; they only need to agree on the filesystem boundary and a permission bit. The general principle: for any shared kernel resource with more than one potential manager, draw an explicit single-writer boundary rather than building coordination logic on top of a resource that assumes one owner.

**Q5. A GPU node has one shared NVMe device with no `io.max`/`io.weight`/`iocost` configured. A checkpoint-writing pod and a data-loading pod share it. What happens, and what's the closest parallel elsewhere in this module?**
**Answer:** The checkpoint pod's heavy sequential writes fill the device's queue; the data-loader pod's reads now queue behind them, so its `io.pressure some avg10` rises even though it did nothing wrong — device bandwidth has no default per-cgroup isolation. This is structurally the same failure as lesson 7's conntrack table exhaustion: a single finite, shared kernel-managed resource with no default per-tenant fairness, where one workload's legitimate use degrades an unrelated neighbor's. The fix in both cases is the same shape — explicit per-tenant limits (`io.max`/`io.weight`/`iocost` here, conntrack tuning/eBPF datapaths there) rather than assuming the shared resource self-arbitrates.

## Connections & what's next

This is the **last lesson in module 01b**. It closes the loop the module opened: lesson 3 showed you the cgroup-v2 *files*, lesson 4 gave you PSI as the *saturation signal*, lesson 9 gave you the *methodology and tools* to find a bottleneck, and this lesson gave you the *ownership model* (systemd's delegation) that explains who's allowed to write the fix, plus the block-I/O-specific instrumentation (`io.pressure`, `biolatency`, iocost) to finish the disk half of the USE grid. The explicit parallel drawn here between block-I/O noisy-neighbor contention and lesson 7's conntrack exhaustion is meant to stick: shared, unpartitioned kernel resources are a recurring shape across this whole module, not a disk-specific quirk.

From here: consolidate everything into the **[Anatomy of a Container](../practice/anatomy-of-a-container/README.md)** deliverable and work through the **[checkpoint](../checkpoint.md)** — it's the completion gate for this module, and every pass criterion (ns+cgroup lookup by hand, OOM explanation, PSI-based saturation diagnosis, throttled-pod mapping, conntrack reasoning, flame graph + USE walkthrough) draws on a lesson you've now completed. Once the checkpoint is passed, the learner's path continues into **[02 — Kubernetes controllers](../../02-kubernetes-controllers/README.md)**, where the cgroup/namespace mechanism you now understand from the kernel side becomes the substrate the controller layer schedules and enforces against.

## References & further reading

**Primary sources**
- [systemd.resource-control(5)](https://man7.org/linux/man-pages/man5/systemd.resource-control.5.html) — the directive→cgroup-file reference (`CPUQuota`, `MemoryMax`, `IOWeight`, `Delegate`).
- [systemd cgroup delegation (CGROUP_DELEGATION)](https://systemd.io/CGROUP_DELEGATION/) — the ownership/single-writer model and why v2 is the safe substrate for the kubelet.
- [Kubernetes — Container runtimes (cgroup driver guidance)](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) — read for the explicit warning that a mismatched cgroup driver creates two cgroup managers and node instability; the basis for this lesson's cgroup-driver-mismatch use case.

**Real-world engineering blogs**
- [Meta resctl — iocost / IO resource protection docs](https://facebookmicrosites.github.io/resctl-demo-website/docs/demo_docs/res_protection/IO/) — what it shows: why Meta built and upstreamed a cost-model I/O controller once raw `io.max`/`io.weight` bandwidth caps proved insufficient for fleet-scale noisy-neighbor disk contention.

**Deeper dives**
- Brendan Gregg, *Systems Performance* (2nd ed.) — Disks chapter (I/O stack, latency vs utilization) + [Poor Disk Performance (biolatency in practice)](https://www.brendangregg.com/blog/2021-05-09/poor-disk-performance.html) — the latency-histogram tool used in this lesson's worked example.
