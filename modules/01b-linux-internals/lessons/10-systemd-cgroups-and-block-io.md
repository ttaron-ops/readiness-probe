---
lesson: "01b.10"
title: "systemd as cgroup manager, and the block-I/O path"
module: "01b"
concept: "systemd as cgroup manager, and the block-I/O path"
status: not-started
est_time: "3h"
artifacts: []
---
# 01b.10 · systemd as cgroup manager, and the block-I/O path

> **Concept.** How systemd owns and delegates the cgroup tree (slices/scopes/services, delegation to the kubelet), and the block-I/O path read through io PSI and latency tools.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters

On a cgroup-v2 host, exactly one process is allowed to be the writer of the cgroup tree, and on almost every modern distro that process is `systemd` (PID 1). Every resource limit you have ever set on a node — the kubelet's `--kube-reserved`, a pod's `MemoryMax`, the CPU throttle you saw in lesson 3 — is ultimately a file under `/sys/fs/cgroup` that systemd either wrote or *delegated* the right to write. If you don't understand that ownership model, node pressure looks like magic: pods get OOM-killed inside a slice you didn't know existed, or the kubelet fails to start because it can't create its own subtree. And when a single slow disk on a GPU box stalls "unrelated" pods, the signal that explains it — io PSI — lives in the same cgroup tree. This lesson connects the systemd front-end you already use to the kernel mechanism underneath.

## From using to understanding

You already write unit files and read `journalctl`. What you probably treat as opaque is *where a unit's limits go*. `MemoryMax=100M` in a unit is not a systemd feature — it is systemd writing `100M` into `memory.max` of that unit's cgroup, the exact file from lesson 3. Units, slices, and scopes are just systemd's naming scheme for directories in the cgroup-v2 hierarchy. And the kubelet, from systemd's point of view, is a *delegated cgroup manager*: systemd hands it a subtree (`kubepods.slice`) and promises not to touch the controllers inside it, so the kubelet can create one cgroup per pod and per container without fighting PID 1 over the same files. Seeing systemd as "the userspace front-end to the cgroup tree" and the kubelet as "a second manager operating a delegated branch" is the mental shift.

## Core notes

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

**`io.pressure` vs `iostat`.** `iostat -x` gives *device*-centric numbers: `%util` (fraction of time the device had at least one request in flight) and `await` (average request latency). On a modern multi-queue NVMe drive `%util` is nearly useless — the device serves many requests in parallel, so it can be pinned at 100% while far from saturated, or saturated below 100%. What you actually want is *how much did tasks stall waiting for I/O*. That is **io PSI**: `cat /sys/fs/cgroup/<path>/io.pressure` reports `some` (at least one task stalled on I/O) and `full` (all runnable tasks stalled) as running % over 10s/60s/300s windows. PSI measures *lost work due to resource contention*, per-cgroup, which `await`/`%util` cannot: a rising `io.pressure some avg10` on a pod's slice tells you that pod is losing wall-clock time to disk regardless of what the device counters say.

**`biolatency` (BCC).** Attaches to block-layer tracepoints and prints a *histogram* of I/O completion latency (queue-insert → completion). Averages lie about disks — a p50 of 0.2ms with a p99.9 of 200ms is a very different node than a flat 5ms. `biolatency -D` splits by device; `-Q` includes OS queue time. Use it to see the *shape* of the latency distribution behind an io-pressure spike.

**Why a slow disk stalls "unrelated" pods.** Device bandwidth/IOPS is a shared, finite resource with no strict per-cgroup isolation unless you set `io.max`/`io.weight`. One pod doing heavy writes fills the device queue; every other pod's reads now sit behind it. Those victims show *high `io.pressure`* even though *they* did nothing wrong — the stall is contention, not their own I/O volume. Node-level `io.pressure` rising while one slice dominates `systemd-cgtop`'s I/O column is the classic noisy-neighbor signature on a shared disk.

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

Now read the block-I/O side under load:

```bash
# generate device I/O (bypass page cache so it hits the disk)
fio --name=load --filename=/var/tmp/f --size=2G --rw=randwrite \
    --bs=64k --direct=1 --numjobs=4 --time_based --runtime=30 &

iostat -x 2                # watch await climb; note %util is misleading on NVMe
cat /sys/fs/cgroup/io.pressure                 # some avg10 rising = real stall
biolatency -D 5 1          # per-device latency histogram; look at the tail buckets
```

`iostat` tells you the *device* is busy; `io.pressure` tells you *work is being lost*; `biolatency` tells you the *latency distribution* — three different questions, and only the middle one is per-cgroup.

## Practice

1. **Map the tree.** Run `systemd-cgls` on your machine (or a node). Identify one `*.slice`, one `*.service`, and one `*.scope`, and for each, note the corresponding directory under `/sys/fs/cgroup`. Confirm the `a-b.slice` → `a.slice/a-b.slice` path encoding on a nested slice.
2. **Constrained scope.** Use `systemd-run --scope -p MemoryMax=…` (or `-p CPUQuota=…`) to run any workload in a transient scope. `cat` the matching `memory.max`/`cpu.max` in that scope's cgroup dir and verify the number matches your directive. Trigger the limit (exceed memory, or a CPU spinner) and read `memory.events`/`cpu.stat`.
3. **I/O pressure.** Put the disk under load with `dd if=/dev/zero of=./big bs=1M count=4096 oflag=direct` or `fio`. While it runs, capture `io.pressure` (root and, if on a node, a workload slice) and either a `biolatency` histogram or `iostat -x`. 

**Acceptance:** a short note tracing one **systemd slice → cgroup path → resource file** (name all three), plus **one io-pressure observation** (the `some avg10` value under load and what it implied vs what `%util` showed). Feeds the *Anatomy of a Container* deliverable.

## Self-check

**Q1. How does systemd delegate a cgroup subtree to the kubelet, and why is `Delegate=yes` required?**
**Answer:** systemd creates the top cgroup for the unit (e.g. `kubepods.slice`), enables the requested controllers on it, makes it writable, and then — because `Delegate=yes` is set — stops managing anything *below* it. The kubelet then owns that subtree: it `mkdir`s per-pod/per-container cgroups and writes their `memory.max`/`cpu.max`/`io.max`. `Delegate=yes` is required because cgroup-v2 enforces a single-writer discipline; without it systemd's periodic reconciliation would revert the kubelet's writes or deny them (`EPERM`/`EBUSY`), since two managers can't own the same cgroup files.

**Q2. What does `io.pressure` tell you that `iostat` `await`/`%util` does not?**
**Answer:** PSI measures *lost work* — the running percentage of time tasks were stalled waiting for I/O (`some` = at least one task blocked, `full` = all runnable tasks blocked) — and it does so *per cgroup*. `await` is device-average request latency and `%util` is device busy-time, both device-centric and, on parallel NVMe queues, poor saturation signals (`%util` can sit at 100% while the device is far from saturated). Only `io.pressure` answers "how much wall-clock did *this pod* lose to I/O contention," which is what you need to blame a noisy neighbor.

**Q3. scope vs slice vs service — what's each for?**
**Answer:** A **service** is processes systemd forked itself from a unit file (`ExecStart=`). A **scope** is externally-created processes systemd only adopted and accounts (login sessions, `systemd-run --scope`, container runtime children). A **slice** is a branch node that holds no processes of its own — it exists to group services/scopes and to apply resource limits to the whole subtree (and its name encodes its path in the hierarchy).

## Resources

1. **systemd.resource-control(5)** — the directive→cgroup-file reference (`CPUQuota`, `MemoryMax`, `IOWeight`, `Delegate`). https://man7.org/linux/man-pages/man5/systemd.resource-control.5.html
2. **systemd cgroup delegation** — the ownership/single-writer model and why v2 is the safe substrate for the kubelet. https://systemd.io/CGROUP_DELEGATION/
3. **Gregg, *Systems Performance* 2nd ed** — Disks chapter (skim: I/O stack, latency vs util) + **biolatency** (BCC). https://www.brendangregg.com/blog/2016-10-08/linux-bcc-biolatency.html
