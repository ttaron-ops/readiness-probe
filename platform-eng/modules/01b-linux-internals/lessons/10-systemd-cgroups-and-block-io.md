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
sources: 17
---

# 01b.10 · systemd as cgroup manager, and the block-I/O path

> **Concept.** How systemd owns and delegates the cgroup tree (slices/scopes/services, delegation to the kubelet), and the block-I/O path read through io PSI and latency tools.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

Lesson 09 gave you the methodology (USE) and the instruments (perf, ftrace, off-CPU flame graphs) to *find* that a node's storage path is the bottleneck — the worked example ended with 70% of wall-clock blocked in `nfs_wait_on_request → io_schedule`. What it did not answer is three things you need before you can do anything about it: **whose** cgroup that I/O belongs to, **who is allowed** to write the limit that would bound it, and **where in the block stack** such a limit would even take effect.

This lesson closes all three gaps. It is the module's last lesson, and it joins the two halves the module has been building separately: the cgroup-v2 *files* from lesson 03 (`cpu.max`, `memory.max`) and the *process that owns the tree those files live in* — systemd, PID 1. Then it walks the block-I/O path end to end, from `write(2)` through the page cache and the multi-queue block layer down to the device, and marks the exact points where `io.max`, `io.latency`, `iocost`, and writeback throttling intervene. By the end you should be able to take a pod name, find its slice, read its `io.pressure`, and write the one file that fixes a noisy-neighbour disk problem — and know why that file is the right one.

## Why this matters

On a cgroup-v2 host there is a hard kernel rule that exactly one manager may write any given subtree of `/sys/fs/cgroup`, and on essentially every modern distribution the manager at the top is systemd. Every resource limit that has ever taken effect on one of your nodes — the kubelet's node-allocatable reservation, a pod's memory limit, the CPU throttling you diagnosed in lesson 03 — is ultimately a text file under `/sys/fs/cgroup` that systemd either wrote itself or explicitly handed ownership of to someone else. If you do not have that ownership model in your head, node behaviour looks like magic: pods are OOM-killed inside a slice you did not know existed, the kubelet refuses to start because a cgroup it needs does not exist, or a limit you set with `systemctl` silently reappears at its old value after a daemon-reload.

The block-I/O half matters for a different reason: **storage is the one resource on a Linux node that has no default fairness at all.** CPU has a default weight, so an unconfigured cgroup still gets a share. Memory has the OOM killer, which is brutal but bounded. A block device has neither — with no `io.max`, no `io.latency`, and no `iocost` configured, a single pod issuing large sequential writes can fill the device's queues and every other pod's small reads queue behind them. On a GPU node this is not a theoretical fairness concern; it is the direct mechanism by which one job's checkpoint write starves eight other GPUs of training batches. An 8×H100 node at roughly $2–3 per GPU-hour on-demand (2024–2025 snapshots; substitute your own committed rate) is $16–24/hour of accelerator time, and a data loader stalled behind a checkpoint writer converts that directly into nothing.

And the tool that tells you which pod is losing time — as opposed to which device is busy — is `io.pressure`, which lives in the cgroup tree. Both halves of this lesson are the same file system.

## What's new here (calibration)

Per the [module README](../README.md)'s calibration, you already know how to write a systemd unit, `systemctl start/enable`, and read `journalctl` — none of that is re-taught. You have also already met cgroup-v2 interface files (lesson 03) and PSI (lesson 04). What is genuinely new:

- Seeing systemd not as "the init system" but as **the userspace front-end to the cgroup tree**, with a precise directive → cgroup-file mapping table, including the unit conversions that trip people up (base-1000 for I/O bandwidth, base-1024 for memory).
- The **slice naming algorithm** — how `kubepods-burstable-pod<uid>.slice` mechanically expands into a filesystem path, including the dash-to-underscore escaping the kubelet applies to pod UIDs.
- The **delegation model** as the kernel actually implements it: the no-internal-process constraint, the single-writer discipline, what `Delegate=` promises and — a correction to the common folk explanation — which units actually need it (the container runtime's scopes, not the kubelet, and never a slice).
- **What the kubelet really writes** when it enforces node-allocatable, including the fact that it sets `memory.max` and CPU *shares* on `kubepods.slice` but deliberately does **not** set a CPU quota.
- The **block-I/O path with the control points located exactly**: `io.max` acts on the bio in `submit_bio_noacct()` before a request is even allocated; `io.latency`, `iocost`, and writeback throttling act through the rq-qos framework at request issue and completion; the I/O scheduler is downstream of all of them.
- The **real interface-file formats** for `io.stat`, `io.max`, `io.weight`, `io.latency`, `io.cost.qos`, and `io.cost.model`, field by field, so you can read and write them without a manual.
- Why **`io.weight` is frequently a no-op** on a default kernel, and what to do instead — the single most surprising fact in this lesson.
- **cgroup writeback**: why buffered writes are attributed to the cgroup that dirtied the page rather than the flusher thread, and which filesystems implement that.

## Core concepts

### 1. The problem: one tree, several would-be owners

Start with the constraint, because everything systemd does here is a response to it.

cgroup v2 is a **unified hierarchy**: a single tree, mounted once at `/sys/fs/cgroup`, in which every controller (cpu, memory, io, pids, cpuset, hugetlb) operates on the *same* set of directories. That was the central design change from v1, where each controller had its own independent hierarchy and a process could sit at `/cpu/A` and `/memory/B` simultaneously. Unification bought coherence — a cgroup is now one place, so the io controller can ask the memory controller which cgroup dirtied a page — and it cost flexibility.

Two structural rules fall out of unification, and they are the whole reason the systemd/kubelet relationship works.

**Rule 1 — the no-internal-process constraint.** A non-root cgroup may not both contain processes *and* distribute a controller to its children. Concretely: if `foo/cgroup.procs` is non-empty, you cannot write `+cpu` into `foo/cgroup.subtree_control`; and if `foo/cgroup.subtree_control` already enables a controller, you cannot move a process into `foo/cgroup.procs` (you get `EBUSY`). The root cgroup is exempt.

Why does the kernel insist on this? Because a controller's job is to divide a resource among *competing children*, and if the parent also holds tasks of its own, those tasks are a sibling with no weight, no limit, and no name. There is no defensible answer to "how much CPU does the parent's own process get relative to child cgroup A." Rather than invent one, cgroup v2 forbids the situation: **processes live only on leaves.** This is why every real cgroup tree looks like a set of empty branch directories with all the PIDs at the bottom — and it is precisely why systemd has a unit type (`slice`) that is *defined* as "a branch that holds no processes."

**Rule 2 — single writer per subtree.** This one is not enforced by a kernel check; it is a discipline, and violating it produces flapping rather than an error. cgroup files have no transactions, no compare-and-swap, and no notification of "someone else changed this." If two daemons both believe they own `foo/memory.max`, each one's periodic reconciliation pass will overwrite the other's value. Nothing errors. The limit simply oscillates, and the pod dies at 3 a.m. for reasons the logs do not explain.

So the tree needs an owner. On a systemd host, PID 1 claims the whole thing at boot — it mounts the cgroup2 filesystem, creates the top-level structure, and enables controllers on it. Anything else that wants to manage cgroups must either ask systemd (over D-Bus) or be given a fenced-off region (delegation). Those are the only two legal moves, and the rest of §2–§6 is the detail of each.

```
  cgroup v2 unified hierarchy — the rules, drawn
  ══════════════════════════════════════════════

  /sys/fs/cgroup/                 cgroup.subtree_control: "cpu io memory pids"
  │                               cgroup.procs:           (root is exempt)
  │
  ├── system.slice/               cgroup.subtree_control: "cpu io memory pids"
  │   │                           cgroup.procs:           EMPTY  ← rule 1
  │   ├── containerd.service/     cgroup.procs:           654, 655, 661 ← leaf
  │   └── kubelet.service/        cgroup.procs:           987          ← leaf
  │
  └── kubepods.slice/             cgroup.procs:           EMPTY
      └── ...                     ▲
                                  │
        ┌─────────────────────────┴──────────────────────────┐
        │ Rule 1 (kernel-enforced): a cgroup with children    │
        │ that enables controllers CANNOT hold processes.     │
        │ Try it and you get EBUSY on cgroup.procs.           │
        │                                                     │
        │ Rule 2 (discipline, NOT enforced): one writer per    │
        │ subtree. Two writers = last write wins, silently,    │
        │ forever. This is what Delegate= exists to arrange.   │
        └──────────────────────────────────────────────────────┘
```

Check the rules on your own box:

```
$ cat /sys/fs/cgroup/cgroup.controllers
cpuset cpu io memory hugetlb pids rdma misc

$ cat /sys/fs/cgroup/cgroup.subtree_control
cpuset cpu io memory pids

$ cat /sys/fs/cgroup/system.slice/cgroup.procs   # branch node: empty
$ wc -l < /sys/fs/cgroup/system.slice/sshd.service/cgroup.procs
2
```

`cgroup.controllers` is what is *available* here; `cgroup.subtree_control` is what this cgroup has *turned on for its children*. A controller must be enabled at every level from the root down to where you want to use it — this is the "controllers are enabled top-down, disabled bottom-up" rule, and it is why `io.max` sometimes does not exist in a directory: nobody enabled `io` on its parent.

### 2. systemd's object model: service, scope, slice

systemd exposes the tree through three unit types. The distinction is not cosmetic — it tells you *who created the processes*, which tells you who can be expected to clean them up.

| Unit type | Who forked the processes | systemd's role | Cgroup path suffix | Typical example |
|---|---|---|---|---|
| **service** | systemd itself, from `ExecStart=` | full lifecycle: start, stop, restart, watchdog, accounting, limits | `foo.service` | `kubelet.service` |
| **scope** | somebody else (a login, a container runtime, a `fork()` from a daemon) | *adopts* an existing process group: accounting and limits only, no lifecycle | `foo.scope` | `session-3.scope`, `cri-containerd-<id>.scope` |
| **slice** | nobody — holds no processes | a branch node used to group units and apply limits to a whole subtree | `foo.slice` | `system.slice`, `kubepods-burstable.slice` |

A **service** is the familiar case. You write a unit file, systemd forks, the child is placed in a fresh cgroup named after the unit, and systemd tracks it until it exits.

A **scope** is the case people get wrong. Nothing in a scope was forked by systemd. When you log in, `systemd-logind` calls `StartTransientUnit` over D-Bus to create `session-3.scope` and hands systemd the PID of your shell; systemd creates the cgroup, moves the PID in, applies whatever properties were requested, and thereafter only *accounts*. It will not restart your shell if it dies. Container runtimes use exactly the same mechanism: containerd asks systemd to create `cri-containerd-<container-id>.scope` and moves the container's init process into it. **The lifecycle belongs to the runtime; the cgroup belongs to systemd.** That split is why `systemctl stop cri-containerd-abc.scope` kills the container's processes but leaves containerd convinced the container is still running.

A **slice** is the branch node that rule 1 requires. It holds no processes ever. Its only jobs are to group children and to be a place to hang resource limits that apply to the whole subtree.

**Slice naming is a path encoding, and you must be able to do it in your head.** The root slice is written `-.slice` and lives at `/sys/fs/cgroup`. Every other slice name is a **dash-separated path from the root**, where each dash means "go one level deeper" (systemd.slice(5)):

```
  -.slice                                → /sys/fs/cgroup
  system.slice                           → /sys/fs/cgroup/system.slice
  user.slice                             → /sys/fs/cgroup/user.slice
  user-1000.slice                        → /sys/fs/cgroup/user.slice/user-1000.slice
  kubepods.slice                         → /sys/fs/cgroup/kubepods.slice
  kubepods-burstable.slice               → /sys/fs/cgroup/kubepods.slice/
                                              kubepods-burstable.slice
  kubepods-burstable-pod3f9a_1c.slice    → /sys/fs/cgroup/kubepods.slice/
                                              kubepods-burstable.slice/
                                              kubepods-burstable-pod3f9a_1c.slice
```

Notice the redundancy: the full ancestry is repeated in every component's name. That is deliberate — a slice unit name is globally unique and self-locating, so `systemctl set-property kubepods-burstable.slice IOWeight=50` needs no path argument. It also means a slice name may not contain a literal dash for any other purpose, which matters in a moment.

Because names may not contain `/`, and because a dash is reserved, systemd defines an escaping scheme (`systemd-escape`) for turning arbitrary strings into unit names: `/` becomes `-`, and other unsafe bytes become `\xNN`. You will see this on mount units (`/var/lib/kubelet` → `var-lib-kubelet.mount`).

The default tree systemd builds at boot:

- **`init.scope`** — PID 1 itself, plus its direct helper threads.
- **`system.slice`** — the default parent for every `.service` unit that does not say otherwise. `Slice=` defaults to `system.slice`.
- **`user.slice`** — one `user-<UID>.slice` per logged-in user, containing `user@<UID>.service` (that user's own systemd instance) and one `session-<N>.scope` per login.
- **`machine.slice`** — VMs and containers registered with `systemd-machined` (`machinectl`). Kubernetes does *not* use this.

### 3. The kubepods tree the kubelet actually creates

Now the part you will look at on a real node. The kubelet has a `cgroupDriver` setting with two values, and the difference is not cosmetic.

**`cgroupDriver: systemd`** (the kubeadm default since Kubernetes v1.22, and what the Kubernetes docs recommend whenever systemd is the init system). The kubelet expresses cgroups as *systemd unit names* and creates them by asking systemd. Internally it holds a cgroup name as a slice of components and converts:

```
  CgroupName{"kubepods", "burstable", "pod1234-abcd-5678-efgh"}
        │
        │  1. escape each component: every '-' becomes '_'
        │     (so the component's own dashes can't be confused
        │      with the dashes that encode hierarchy)
        ▼
  {"kubepods", "burstable", "pod1234_abcd_5678_efgh"}
        │
        │  2. join with '-', append ".slice"
        ▼
  "kubepods-burstable-pod1234_abcd_5678_efgh.slice"
        │
        │  3. expand the slice name into a path
        ▼
  /kubepods.slice/kubepods-burstable.slice/
      kubepods-burstable-pod1234_abcd_5678_efgh.slice
```

(That transformation, including the worked example, is exactly what `pkg/kubelet/cm/cgroup_manager_linux.go` documents and implements.) The dash-to-underscore escaping is why **a pod UID in the cgroup tree never looks like the UID in `kubectl get pod -o yaml`** — `4d0e2ff5-8f43-4b1e-9a20-2c1e5f3b7a11` becomes `4d0e2ff5_8f43_4b1e_9a20_2c1e5f3b7a11`. Getting bitten by this once while grepping is a rite of passage; knowing it saves you the grep.

**`cgroupDriver: cgroupfs`.** The kubelet skips systemd entirely and `mkdir`s directly: `/sys/fs/cgroup/kubepods/burstable/pod1234-abcd-5678-efgh/`. No escaping, no slices, no D-Bus. This is the setting that creates the two-managers problem in §6.

Here is the finished tree on a node running the systemd driver with containerd:

```
  systemd slice hierarchy on a Kubernetes GPU node (cgroupDriver: systemd)
  ════════════════════════════════════════════════════════════════════════

  /sys/fs/cgroup                                    ← "-.slice", the root
  │   cgroup.subtree_control = cpuset cpu io memory pids
  │
  ├── init.scope                                    ← PID 1
  │
  ├── system.slice                                  ← all OS + node daemons
  │   ├── containerd.service        [Delegate=yes]  ← runtime; 654
  │   ├── kubelet.service                           ← 987
  │   ├── sshd.service
  │   ├── node-exporter.service
  │   └── nvidia-dcgm-exporter.service
  │       (no ceiling here by default — see §6)
  │
  ├── user.slice
  │   └── user-1000.slice
  │       ├── user@1000.service                     ← per-user systemd
  │       └── session-3.scope                       ← your ssh shell
  │
  └── kubepods.slice                                ← created by the kubelet
      │   memory.max  = capacity − kube-reserved − system-reserved − eviction
      │   cpu.weight  = f(allocatable millicores)      ← NO cpu.max. §6.
      │   pids.max    = (if configured)
      │
      ├── <Guaranteed pods live DIRECTLY here>
      │   └── kubepods-pod<uid>.slice
      │       │   memory.max = sum(container limits)
      │       │   cpu.max    = "<quota> 100000"
      │       └── cri-containerd-<64-hex-id>.scope   ← one per container
      │               memory.max, cpu.max, cpu.weight, io.* ...
      │               cgroup.procs → the container's PIDs
      │
      ├── kubepods-burstable.slice
      │   │   cpu.weight = f(Σ burstable CPU requests)
      │   └── kubepods-burstable-pod<uid-with-underscores>.slice
      │       ├── cri-containerd-<id>.scope          ← app container
      │       └── cri-containerd-<id>.scope          ← sidecar
      │
      └── kubepods-besteffort.slice
          │   cpu.weight = minimum (2 CPU shares → weight 1)
          └── kubepods-besteffort-pod<uid>.slice
              └── cri-containerd-<id>.scope
```

Three details in that picture that are worth stating explicitly, because they are the ones people get wrong:

1. **Guaranteed pods have no QoS-class slice.** Burstable and BestEffort get their own intermediate slice; Guaranteed pods are parented directly on `kubepods.slice`. That is intentional — Guaranteed pods have `requests == limits`, so they need no proportional-share arbitration against each other, only their own hard caps.
2. **The scope, not the pod slice, is where a container's limits live.** A pod slice's `memory.max` is the sum over its containers; each container's own limit is on its scope. When a container is OOM-killed you will find the counter incremented in the *scope's* `memory.events`, and the pod slice's `memory.events` will show it too through the hierarchical `oom_kill` accounting.
3. **The pause/sandbox container also gets a scope.** A pod with one app container has two scopes under its slice.

CRI-O names its scopes `crio-<id>.scope` instead of `cri-containerd-<id>.scope`; Docker/`dockershim`-era nodes used `docker-<id>.scope`. The shape is identical.

Locate any of it from a PID, with no `kubectl` and no `crictl`:

```
$ cat /proc/48213/cgroup
0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod3f9a1c22_7b04_4d51_b2ef_0a9c1d4e88f1.slice/cri-containerd-9c2b7f1e0a4d3c5b6e8f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d.scope
```

The single `0::` prefix is the signature of cgroup v2 (a v1 host prints one line per controller hierarchy). Everything after `0::` is a path relative to `/sys/fs/cgroup`, so:

```
$ C=/sys/fs/cgroup$(awk -F: '{print $3}' /proc/48213/cgroup)
$ cat $C/memory.max $C/cpu.max $C/io.max
8589934592
50000 100000
(empty — no I/O limit set)
```

### 4. Directives to cgroup files: the complete mapping

systemd's resource-control directives are a thin, typed façade over the cgroup files you already know. Here is the mapping you should be able to reproduce, with the defaults and unit conventions, from systemd.resource-control(5) (systemd 254+; the table is stable across recent versions):

| systemd directive | cgroup v2 file | Value written | Range / default |
|---|---|---|---|
| `CPUWeight=N` | `cpu.weight` | `N` verbatim | 1–10000, kernel default 100 |
| `CPUWeight=idle` | `cpu.idle` | `1` | — |
| `StartupCPUWeight=N` | `cpu.weight` | applies only during boot/shutdown | 1–10000 |
| `CPUQuota=20%` | `cpu.max` | `20000 100000` | quota = pct × period; unset = `max 100000` |
| `CPUQuotaPeriodSec=50ms` | `cpu.max` (2nd field) | `10000 50000` (with 20%) | default 100ms, kernel clamps to 1ms–1s |
| `AllowedCPUs=0-3,8` | `cpuset.cpus` | `0-3,8` | — |
| `AllowedMemoryNodes=0` | `cpuset.mems` | `0` | — |
| `MemoryMin=1G` | `memory.min` | `1073741824` | hard reclaim protection |
| `MemoryLow=2G` | `memory.low` | `2147483648` | best-effort reclaim protection |
| `MemoryHigh=3G` | `memory.high` | `3221225472` | throttle + reclaim, **no kill** |
| `MemoryMax=4G` | `memory.max` | `4294967296` | hard limit, **OOM-kills** on breach |
| `MemorySwapMax=0` | `memory.swap.max` | `0` | — |
| `MemoryZSwapMax=` | `memory.zswap.max` | bytes | — |
| `TasksMax=4096` | `pids.max` | `4096` | default = 15% of `min(kernel.pid_max, kernel.threads-max, root pids.max)` — 4915 on a stock box |
| `IOWeight=200` | `io.weight` | `default 200` | 1–10000, default 100 — **see §9, often a no-op** |
| `IODeviceWeight=/dev/nvme0n1 400` | `io.weight` | `259:0 400` | per-device override |
| `IOReadBandwidthMax=/dev/nvme0n1 50M` | `io.max` | `259:0 rbps=50000000` | **base 1000**, not 1024 |
| `IOWriteBandwidthMax=/dev/nvme0n1 20M` | `io.max` | `259:0 wbps=20000000` | base 1000 |
| `IOReadIOPSMax=/dev/nvme0n1 2K` | `io.max` | `259:0 riops=2000` | base 1000 |
| `IOWriteIOPSMax=/dev/nvme0n1 1K` | `io.max` | `259:0 wiops=1000` | base 1000 |
| `IODeviceLatencyTargetSec=/dev/nvme0n1 25ms` | `io.latency` | `259:0 target=25000` | µs; implies `IOAccounting=yes` |
| `Delegate=cpu memory io pids` | (writes `cgroup.subtree_control`, chowns the dir) | — | default `no` |
| `Slice=kubepods.slice` | (chooses the parent directory) | — | default `system.slice` |

**The unit-base asymmetry is a real trap.** `MemoryMax=1G` is 1 GiB = 1,073,741,824 bytes (base 1024). `IOReadBandwidthMax=/dev/x 1G` is 1,000,000,000 bytes/s (base 1000), because bandwidth is conventionally decimal. If you set a bandwidth cap and the number in `io.max` is not what you expected, this is why.

**Accounting is not free and is not all on by default.** systemd gates whether a controller is enabled on a unit at all:

| Setting in `system.conf` | Default | Consequence if `no` |
|---|---|---|
| `DefaultMemoryAccounting=` | `yes` | `MemoryCurrent=` unavailable |
| `DefaultTasksAccounting=` | `yes` | `TasksCurrent=` unavailable |
| `DefaultIOAccounting=` | **`no`** | **`systemd-cgtop` shows no I/O column data; `io.stat` may be absent** |
| `DefaultIPAccounting=` | `no` | no per-unit byte counters |
| CPU accounting | always on (v2) | — (deprecated as a setting in systemd 258: `cpu.stat` is free on v2) |

That `DefaultIOAccounting=no` line is the reason a fresh `systemd-cgtop` often shows dashes in the I/O columns and people conclude the tool is broken. Fix it per-unit with `IOAccounting=yes`, or globally in `/etc/systemd/system.conf`. Setting *any* `IO*` directive implies it.

**Three ways to apply a directive, and what each does to disk:**

```
# (a) Statically, in a unit or a drop-in — applied at next start.
$ sudo systemctl edit containerd.service      # writes .../containerd.service.d/override.conf
[Service]
MemoryHigh=2G
IOAccounting=yes

# (b) Live + persistent. Writes the cgroup file NOW and drops a
#     file under /etc/systemd/system/<unit>.d/50-<prop>.conf.
$ sudo systemctl set-property containerd.service MemoryHigh=2G
$ cat /etc/systemd/system/containerd.service.d/50-MemoryHigh.conf
[Service]
MemoryHigh=2G
$ cat /sys/fs/cgroup/system.slice/containerd.service/memory.high
2147483648

# (c) Live only, gone at reboot. Writes /run/systemd/system/... instead.
$ sudo systemctl set-property --runtime containerd.service MemoryHigh=2G
```

**The single most common misconception here is that `set-property` needs a restart.** It does not. It writes the kernel file immediately over D-Bus and *additionally* persists a drop-in so the value survives. If you want the change to take effect but not survive, use `--runtime`.

And the transient case, which is how you experiment without writing any files:

```
# A one-off constrained run, adopted into a new scope in your current shell:
$ systemd-run --scope --unit=demo -p MemoryMax=100M -p IOWeight=10 \
      stress-ng --vm 1 --vm-bytes 150M --timeout 20s

# Or as a forked service (default), detached, with its own log:
$ systemd-run --unit=demo-svc -p CPUQuota=50% -p IOReadBandwidthMax="/dev/nvme0n1 10M" \
      /usr/local/bin/loader.sh
$ journalctl -u demo-svc -f
```

### 5. Delegation: what `Delegate=` actually promises

This is the mechanism people cite most and describe least accurately, so be precise.

`Delegate=yes` on a **service** or **scope** unit tells systemd: *this cgroup is a cut-off point. I created it, I enabled controllers on it, I made it writable by the unit's user, and from here down I will not create, remove, or modify anything.* (systemd's own `CGROUP_DELEGATION.md`.) Concretely: systemd will not `mkdir` or `rmdir` below the delegated cgroup, and will not write any attribute file below it — not on a reload, not on a `daemon-reexec`, not ever. It *will* still manage the delegated cgroup's own attributes and still enable the controllers you asked for in `cgroup.subtree_control`, so the delegatee can use them. `Delegate=` takes a boolean or a controller list — `Delegate=cpu cpuset io memory pids` grants exactly those, and granting is not free (each enabled controller costs per-cgroup accounting), so delegate what the delegatee needs and no more.

**Slices cannot be delegated, and this is the most-corrected point in the topic.** A slice is an inner node whose children systemd itself creates and destroys as units start and stop; delegating it would put systemd and the delegatee in the same directory, violating single-writer. systemd rejects it. So the folk explanation "the kubelet gets `kubepods.slice` delegated" is **wrong**, and the actual arrangement is more interesting:

- With `cgroupDriver: systemd`, the kubelet does not need delegation *because it does not write the tree directly*. It calls systemd's D-Bus API (`org.freedesktop.systemd1.Manager.StartTransientUnit`) to create `kubepods.slice`, the QoS slices, and each pod slice, passing resource properties as arguments. systemd creates the directories and writes the files. **There is exactly one writer: systemd. That is the whole point of the systemd driver.**
- The **container runtime** is where delegation actually appears. Upstream `containerd.service` ships with `Delegate=yes` (alongside `KillMode=process`, `OOMScoreAdjust=-999`, `TasksMax=infinity`). This matters for two reasons: containerd forks shims and container processes that must live in sub-cgroups of the service, and container *scopes* are created with delegation so that a container running its own init (systemd-in-a-container, or a nested runtime) can subdivide its cgroup freely.
- `DelegateSubgroup=` (systemd 254+) is a refinement: it puts the unit's own main process into a named subgroup immediately, so the delegated cgroup itself stays process-free and satisfies rule 1 without the delegatee having to migrate itself.

Here is the actual call sequence when a pod starts, which is the thing to hold in your head:

```
  Pod start on a systemd-cgroup-driver node — who writes what, in order
  ═══════════════════════════════════════════════════════════════════════

  t │ kubelet                systemd (PID 1)            kernel / cgroupfs
  ──┼────────────────────────────────────────────────────────────────────
  1 │ admit pod, compute
    │ QoS = Burstable,
    │ cgroup name =
    │ kubepods-burstable-
    │   pod3f9a_1c.slice
    │        │
  2 │        ├─ D-Bus ──────▶ StartTransientUnit(
    │        │                  name  = "...pod3f9a_1c.slice",
    │        │                  props = MemoryMax=8G,
    │        │                          CPUWeight=102,
    │        │                          CPUQuota=50%)
    │        │                          │
  3 │        │                          ├─ mkdir ──────▶ /sys/fs/cgroup/
    │        │                          │                 kubepods.slice/
    │        │                          │                 kubepods-burstable.slice/
    │        │                          │                 ...pod3f9a_1c.slice/
    │        │                          ├─ write ──────▶ cgroup.subtree_control
    │        │                          │                  "+cpu +memory +pids +io"
    │        │                          ├─ write ──────▶ memory.max  = 8589934592
    │        │                          ├─ write ──────▶ cpu.weight  = 102
    │        │                          └─ write ──────▶ cpu.max     = "50000 100000"
    │        │
  4 │        ├─ CRI RunPodSandbox(cgroup_parent =
    │        │     "kubepods-burstable-pod3f9a_1c.slice")
    │        │        │
    │        │        ▼
    │        │   containerd → runc
    │        │        │
  5 │        │        ├─ D-Bus ─▶ StartTransientUnit(
    │        │        │             name   = "cri-containerd-<id>.scope",
    │        │        │             Slice  = "...pod3f9a_1c.slice",
    │        │        │             PIDs   = [<container init pid>],
    │        │        │             Delegate = true)
    │        │        │                       │
  6 │        │        │                       ├─ mkdir ─▶ .../cri-containerd-<id>.scope
    │        │        │                       ├─ write ─▶ memory.max, cpu.max, ...
    │        │        │                       └─ write ─▶ cgroup.procs ← <pid>
    │        │        │
  7 │        │        └─ ▲ from here down, systemd never touches anything
    │        │            (Delegate=true on the scope)
    │
  ⇒ ONE writer at every level. The kubelet asks; systemd writes; below the
    scope boundary the runtime writes. No locks, no protocol — just an
    ownership boundary and a permission bit.
```

Contrast that with `cgroupDriver: cgroupfs`, where step 2 becomes `mkdir /sys/fs/cgroup/kubepods/burstable/pod3f9a1c22-.../` issued straight by the kubelet, in a region of the tree systemd also believes it owns. Nothing errors immediately. The failure arrives later — see §6.

### 6. Node-allocatable enforcement: what the kubelet really writes

Lesson 03 gave you the node-allocatable arithmetic:

```
  Allocatable = Capacity − kubeReserved − systemReserved − evictionHard
```

What is worth adding here is **which of those subtractions is actually enforced by a cgroup file, and which is only bookkeeping**, because they are not the same and the difference is a real production hazard.

`--enforce-node-allocatable` defaults to `[pods]`. With that default, when the kubelet starts it writes limits onto the `kubepods` cgroup (`kubepods.slice` under the systemd driver). From `pkg/kubelet/cm/node_container_manager_linux.go`, the resources it sets are:

- **`memory.max`** = capacity − kubeReserved − systemReserved (and, with a hard eviction threshold configured, the eviction reserve as well). This is a real ceiling: the sum of all pods on the node cannot exceed it, and a breach produces a cgroup-scoped OOM inside `kubepods.slice`.
- **CPU shares/weight** derived from allocatable millicores. The source comments that this must be set "no matter what because default cpu shares on cgroups are low and can cause cpu starvation" — a fresh cgroup gets weight 100 regardless of how much CPU it is meant to have, which would leave all pods collectively competing with `system.slice` at parity.
- **`pids.max`**, if a PID reservation is configured.

And a fact worth memorising because it comes up in interviews: **the kubelet does not set a CPU quota on `kubepods`.** The source carries a standing `TODO(vishh): Set CPU Quota if necessary.` CPU is compressible — a pod that wants more CPU than its share simply runs slower — so the kubelet uses weights, not a hard cap, at the node level. Individual pods with `limits.cpu` still get `cpu.max` on their own slice (lesson 03's throttling story); the *node* level has no quota.

**`system.slice` is not fenced by default.** systemd's job is to *account* what lands under each slice, not to decide how big `system.slice` may get. Nothing in a default install stops the kubelet, containerd, a logging agent, and a DCGM exporter from collectively consuming more than `systemReserved + kubeReserved` promised. Reserving without enforcing means the reservation is a subtraction in the scheduler's arithmetic and nothing else: pods are kept from using that memory, but system daemons are not kept *within* it.

To make it real you must (a) create the cgroups yourself, and (b) name them:

```yaml
# /var/lib/kubelet/config.yaml   (KubeletConfiguration, kubelet.config.k8s.io/v1beta1)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
cgroupsPerQOS: true
systemReserved:
  cpu: "1"
  memory: "2Gi"
  ephemeral-storage: "10Gi"
kubeReserved:
  cpu: "1"
  memory: "2Gi"
systemReservedCgroup: "/system.slice"
kubeReservedCgroup: "/runtime.slice"
enforceNodeAllocatable:
  - pods
  - kube-reserved
  - system-reserved
evictionHard:
  memory.available: "500Mi"
```

Two warnings that the Kubernetes documentation itself states, and that you should repeat to anyone about to enable this:

1. **The named cgroups must already exist when the kubelet starts, or the kubelet fails to start.** With the systemd driver the name must carry the `.slice` suffix semantics — a `kubeReservedCgroup` of `/runtime.slice` needs a real `runtime.slice` on the box, and the runtime's units need `Slice=runtime.slice`.
2. **Enforcing `system-reserved` can starve or OOM-kill the daemons that keep the node alive** — sshd, the kubelet itself, the container runtime. The documented guidance is to profile first and to start with compressible resources (`system-reserved-compressible`, which enforces CPU only), because a CPU-starved sshd recovers and an OOM-killed kubelet does not.

**The cgroup-driver mismatch failure, mechanically.** Suppose the kubelet runs with `cgroupfs` while the host is systemd-managed. Now `/sys/fs/cgroup/kubepods/` exists as a plain directory that systemd knows nothing about, and `/sys/fs/cgroup/kubepods.slice/` may also exist. Two managers now hold two different views of the same physical resources. The Kubernetes docs put it plainly: *"Two cgroup managers result in two views of the available and in-use resources in the system."* The observable symptoms are the ones that make node debugging miserable — a node reporting free memory while pods are OOM-killed, limits that revert after `systemctl daemon-reload`, and containers whose cgroups are cleaned up by neither party. The documented remedy is not to flip the setting on a live node: changing a node's cgroup driver in place causes sandbox-recreation errors that surviving kubelet restarts do not fix, so you replace the node. kubeadm has defaulted `cgroupDriver: systemd` since v1.22 precisely to stop people from ending up here.

### 7. The block-I/O path, from `write(2)` to the platter

Now the second half of the lesson. To know where a limit acts you need the path it acts on, so build it once, carefully.

A read or write from an application traverses, in order:

1. **The syscall.** `read()`/`write()`/`pread64()` enters the kernel, lands in the VFS, and dispatches to the filesystem's `file_operations`.
2. **The page cache.** For buffered I/O (no `O_DIRECT`), the kernel looks up the folio in the inode's address space. A **read hit returns here** — no block device is touched at all, which is why `iostat` can show an idle disk while an application does millions of reads/s. A **read miss** allocates folios and issues I/O. A **buffered write** copies into the page cache, marks the folio dirty, and *returns immediately*. No device I/O has happened yet.
3. **Writeback.** Dirty folios are flushed later by per-bdi flusher threads, triggered by `vm.dirty_background_ratio` (default 10% of available memory), forced synchronously on the writing task at `vm.dirty_ratio` (default 20%), aged out by `vm.dirty_expire_centisecs` (default 3000 = 30 s), or forced by `fsync()`/`sync()`. **This is the single biggest source of confusion about I/O limits:** the process that dirtied the page is usually not the process that submits the write.
4. **The filesystem** maps file offsets to device blocks and builds **bio** structures — a bio is a vector of (page, offset, length) targeting a device sector range.
5. **`submit_bio()` → `submit_bio_noacct()`.** The generic block layer entry point. Here the bio is charged to a cgroup (`blk_cgroup_bio_start()`) and here **blk-throttle (`io.max`) gets its say** — `submit_bio_noacct()` calls `blk_throtl_bio(bio)`, and if that returns true the bio is queued on the cgroup's throttle service queue and submission stops dead. A timer releases it later. Note what this means: **`io.max` delays the bio before a request is ever allocated and before any scheduler sees it.**
6. **blk-mq: software queues.** The bio is merged into or turned into a `struct request` on a per-CPU software staging queue (`ctx`). Merging here is why `iostat`'s `rrqm/s`/`wrqm/s` columns exist.
7. **rq-qos hooks.** The multi-queue submit path calls into the rq-qos framework, which is where the *latency-driven* controllers live: `RQ_QOS_WBT` (writeback throttling), `RQ_QOS_LATENCY` (`io.latency`), and `RQ_QOS_COST` (`iocost`). Each registers `throttle`, `track`, `issue`, `done`, and `done_bio` callbacks; `throttle` can sleep the submitter, and `done` is where completion latency is measured and fed back.
8. **The I/O scheduler**, if one is attached (`mq-deadline`, `bfq`, `kyber`, or `none`), reorders and batches requests; then the **hardware dispatch queues** (`hctx`, one per hardware queue — NVMe typically maps one per CPU), the **driver** (which builds the NVMe submission-queue entry or SCSI CDB and rings the doorbell), and the **device**. Completion raises an interrupt, the request unwinds, and the `done`/`done_bio` rq-qos hooks fire — which is how the latency-driven controllers learn what the device actually did.

```
  The block-I/O path, with every control point located
  ═════════════════════════════════════════════════════

   application
   ┌───────────────────────────────────────────────────────────┐
   │  write(fd, buf, 65536)          read(fd, buf, 4096)       │
   └──────────┬──────────────────────────────┬─────────────────┘
              │ syscall                      │
   ┌──────────▼──────────────────────────────▼─────────────────┐
   │  VFS  →  filesystem (ext4 / xfs / overlayfs)              │
   └──────────┬──────────────────────────────┬─────────────────┘
              │                              │
   ┌──────────▼──────────────────────────────▼─────────────────┐
   │  PAGE CACHE                                                │
   │   write → copy in, mark folio dirty, RETURN  ◀── no device │
   │   read  → HIT: return  ◀── no device, invisible to iostat  │
   │           MISS: allocate folios, issue read                │
   └──────────┬───────────────────────────────┬────────────────┘
       dirty  │                               │ read miss / O_DIRECT
              │                               │
   ┌──────────▼───────────────┐               │
   │  WRITEBACK               │               │
   │  vm.dirty_background_    │               │
   │    ratio  = 10  (async)  │               │
   │  vm.dirty_ratio = 20     │               │
   │    (sync, blocks writer) │               │
   │  cgroup writeback: bio   │               │
   │   charged to the cgroup  │               │
   │   that OWNS THE INODE,   │               │
   │   not the flusher thread │               │
   │   (ext2/ext4/btrfs/f2fs/ │               │
   │    xfs only)             │               │
   └──────────┬───────────────┘               │
              └───────────────┬───────────────┘
                              │  bio
   ┌══════════════════════════▼═══════════════════════════════┐
   ║ submit_bio_noacct()                                       ║
   ║   blk_cgroup_bio_start(bio)   ← charges io.stat           ║
   ║   ┌─────────────────────────────────────────────────┐     ║
   ║   │ ▶▶ io.max   (blk-throttle)                      │     ║
   ║   │    rbps / wbps / riops / wiops per major:minor  │     ║
   ║   │    over limit → bio parked on the cgroup's      │     ║
   ║   │    service queue, released by a timer.          │     ║
   ║   │    ACTS HERE: before request allocation,        │     ║
   ║   │    before any scheduler. Hard ceiling.          │     ║
   ║   └─────────────────────────────────────────────────┘     ║
   └══════════════════════════┬═══════════════════════════════┘
                              │
   ┌──────────────────────────▼───────────────────────────────┐
   │ blk-mq  software queues (one ctx per CPU)                 │
   │   bio → request, plus front/back merging (rrqm/s, wrqm/s) │
   │   ┌────────────────────────────────────────────────┐      │
   │   │ ▶▶ rq-qos throttle hooks                       │      │
   │   │    • wbt        — wbt_lat_usec 2000 (SSD)      │      │
   │   │                   / 75000 (HDD), depth from 16 │      │
   │   │    • io.latency — target=<µs>; protects, does  │      │
   │   │                   not cap                      │      │
   │   │    • iocost     — io.cost.qos / io.cost.model; │      │
   │   │                   vtime budget per hweight     │      │
   │   │    done()/done_bio() measure completion        │      │
   │   │    latency and feed it back                    │      │
   │   └────────────────────────────────────────────────┘      │
   └──────────────────────────┬───────────────────────────────┘
                              │
   ┌──────────────────────────▼───────────────────────────────┐
   │ I/O SCHEDULER   /sys/block/<dev>/queue/scheduler          │
   │   none | mq-deadline | kyber | bfq        (table below)   │
   │   ← cgroup limits act ABOVE this layer, so they work      │
   │     even with `none` attached                             │
   └──────────────────────────┬───────────────────────────────┘
                              │
   ┌──────────────────────────▼───────────────────────────────┐
   │ hardware dispatch queues (hctx) → driver → doorbell       │
   │   nr_requests = queue depth per hctx (default 256 mq)     │
   └──────────────────────────┬───────────────────────────────┘
                              ▼
                       ┌─────────────┐
                       │  NVMe SSD   │  completion IRQ → rq-qos done()
                       └─────────────┘
```

Three consequences of that picture that you should be able to state without looking:

- **A buffered write is not throttled when the application makes it.** The application returns instantly; the bio appears at writeback time, possibly seconds later, possibly from a kernel flusher thread. Any `io.max` cap you set is applied *then*. This is why `dd` without `oflag=direct` appears to ignore your write limit and then stalls violently later — you throttled the writeback, not the writer.
- **Page-cache hits are invisible everywhere below the cache.** Not in `io.stat`, not in `iostat`, not in `biolatency`. An application doing 500,000 reads/s against a warm cache shows a completely idle disk. If you are trying to reproduce an I/O problem, use `--direct=1` (fio) or `oflag=direct`/`iflag=direct` (dd), or drop caches.
- **The scheduler is the *last* thing in the chain, not the place cgroup limits live.** With `none` (the default on NVMe) there is no scheduler at all, and cgroup I/O control still works fine, because `io.max`, `io.latency` and `iocost` all sit above it. The exception is BFQ, which implements its own proportional sharing inside the scheduler.

### 8. cgroup writeback: who gets charged for a dirty page

Because writeback decouples the writer from the submitter, the kernel needs an explicit answer to "which cgroup does this write belong to." Without one, all buffered writes would be attributed to the flusher thread's cgroup — which is the root — and every write cap on every container would be useless.

The mechanism is **inode ownership**. When a folio is dirtied, the kernel records which cgroup did it. The inode is assigned to a cgroup, and dirty pages from that inode are attributed to the inode's owning cgroup. Pages dirtied by a *different* cgroup are tracked as "foreign"; if a foreign cgroup becomes the majority dirtier over time, ownership of the inode switches to it. Writeback then issues bios tagged with the owning cgroup's blkcg, so `io.max`, `iocost`, and `io.stat` all see them correctly.

Two limitations follow directly from that design, and both are documented in `Documentation/admin-guide/cgroup-v2.rst`:

1. **Filesystem support is not universal.** cgroup writeback is implemented on **ext2, ext4, btrfs, f2fs, and xfs**. On any other filesystem, *all* writeback I/O is attributed to the root cgroup. That includes network filesystems — which is precisely why the NFS-bound training job in lesson 09 would not have shown up in a per-pod `io.stat` at all.
2. **Multiple cgroups writing one inode is not handled well.** Ownership is a single value with hysteresis; concurrent writers to a shared file will see attribution flap.

The dirty-throttling sysctls interact per-cgroup too: `vm.dirty_ratio` and `vm.dirty_background_ratio` are evaluated against the memory available *to that cgroup* (as capped by `memory.max`), not against total system memory. A container with `memory.max=1G` starts background writeback at ~100 MB of dirty pages, not at 10% of the node's RAM.

### 9. The io controller interface files, field by field

Everything here is from `Documentation/admin-guide/cgroup-v2.rst` and is stable across recent kernels. These files exist only where `io` has been enabled in the parent's `cgroup.subtree_control`.

**`io.stat`** — read-only, nested-keyed by `$MAJ:$MIN`:

```
$ cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/io.stat
259:0 rbytes=1459200 wbytes=314773504 rios=192 wios=353 dbytes=0 dios=0
8:0 rbytes=0 wbytes=4096 rios=0 wios=1 dbytes=0 dios=0
```

| Field | Meaning |
|---|---|
| `rbytes` | bytes read from the device by this cgroup, cumulative |
| `wbytes` | bytes written |
| `rios` | number of read I/Os |
| `wios` | number of write I/Os |
| `dbytes` | bytes discarded (TRIM) |
| `dios` | number of discard I/Os |

Derive the average request size yourself: `rbytes/rios` here is 1459200/192 = **7,600 B**, i.e. mixed small reads. That single division tells you more about a workload's I/O character than any dashboard. `259:0` is an NVMe namespace (major 259); `8:0` is `sda`. Map them with `lsblk -o NAME,MAJ:MIN` or `stat -c '%t:%T' /dev/nvme0n1` (which prints hex — convert).

When `CONFIG_BLK_CGROUP_IOLATENCY` and debug stats are enabled, `io.stat` gains iolatency's working fields: `depth` (current allowed queue depth for this cgroup), `avg_lat` and `win` on rotational devices, `missed`/`total` on non-rotational, and `delay` (µs of artificial delay currently injected per process).

**`io.max`** — read-write, the hard ceiling. Nested-keyed:

```
# Cap a scope at 50 MB/s read and 120 write IOPS on nvme0n1:
$ echo "259:0 rbps=52428800 wiops=120" | sudo tee /sys/fs/cgroup/system.slice/demo.scope/io.max

$ cat /sys/fs/cgroup/system.slice/demo.scope/io.max
259:0 rbps=52428800 wbps=max riops=max wiops=120

# Remove one limit:
$ echo "259:0 wiops=max" | sudo tee /sys/fs/cgroup/system.slice/demo.scope/io.max
```

| Key | Meaning | Unit |
|---|---|---|
| `rbps` | max read bytes/second | bytes/s or `max` |
| `wbps` | max write bytes/second | bytes/s or `max` |
| `riops` | max read I/Os per second | ops/s or `max` |
| `wiops` | max write I/Os per second | ops/s or `max` |

You may write any subset in any order; unmentioned keys keep their current value. Enforcement is by **delaying bios** in `submit_bio_noacct()` (§7 step 5), using a per-cgroup service queue and a slice timer. That has a shape worth drawing, because it explains a symptom you will see:

```
  io.max enforcement over time — why capped I/O looks "bursty"
  ════════════════════════════════════════════════════════════
  Cap: wbps = 10,000,000 (10 MB/s). Application writes as fast as it can.

  offered  ██████████████████████████████████████████████████  (100 MB/s)
  load

  throttle │──slice──│──slice──│──slice──│──slice──│──slice──│
  slices   │  100ms  │  100ms  │  100ms  │  100ms  │  100ms  │
           │ budget: │ budget: │ budget: │ budget: │ budget: │
           │  1 MB   │  1 MB   │  1 MB   │  1 MB   │  1 MB   │

  issued   ▓▓░░░░░░░░ ▓▓░░░░░░░░ ▓▓░░░░░░░░ ▓▓░░░░░░░░ ▓▓░░░░░░░░
  to dev   ↑          ↑
           │          └─ budget refilled, parked bios released in a burst
           └─ 1 MB issued fast, then bios park on the service queue

  Application-visible latency for a bio that arrives just after the
  budget is spent  =  time until the next slice boundary  ≈ up to 100 ms.

  ⇒ io.max is a HARD CEILING and is NOT work-conserving: if the device
    is completely idle, a capped cgroup still waits. That is the whole
    difference from io.weight / iocost, which give idle capacity away.
```

**`io.weight`** — proportional share, 1–10000, default 100. Format is a default line plus optional per-device overrides:

```
$ cat /sys/fs/cgroup/system.slice/demo.scope/io.weight
default 100
259:0 400
```

**And now the fact that surprises everyone: on a stock kernel, `io.weight` very often does nothing.** `io.weight` is the interface of the **iocost** controller (`blk-iocost`), and iocost is **disabled by default per device**. Until you enable it on the device via `io.cost.qos`, writing `io.weight` stores a number that no code path consults. Separately, BFQ implements its own proportional sharing and registers its weight under a *different* filename — on cgroup v2 the file is **`bfq.weight`**, not `io.weight` — and BFQ deliberately disables iocost when it is selected as the scheduler for any device, to avoid two controllers claiming the same job.

So the decision table for "I want proportional I/O sharing" is:

| You want | Set | Prerequisite | Work-conserving? |
|---|---|---|---|
| A hard ceiling | `io.max` | `io` controller enabled | **No** — idle device still throttles |
| Latency protection for one workload | `io.latency` | `CONFIG_BLK_CGROUP_IOLATENCY` | Yes — only throttles peers when the target is missed |
| Proportional share, cost-aware | `io.weight` | **`io.cost.qos` `enable=1` on the device**, plus `io.cost.model` | Yes |
| Proportional share via the scheduler | `bfq.weight` | `echo bfq > /sys/block/<dev>/queue/scheduler` | Yes |

**`io.latency`** — the protection primitive. You set a target on the cgroup you want to *protect*:

```
$ echo "259:0 target=25000" | sudo tee /sys/fs/cgroup/.../dataloader.scope/io.latency
```

The semantics are the opposite of a limit and are easy to get backwards: **`io.latency` never slows down the cgroup you set it on.** It watches that cgroup's I/O latency, and when the group misses its target the controller throttles *other* cgroups — those with a higher target or no target at all. Miss detection differs by device class: on rotational devices it compares a windowed average latency against the target; on non-rotational devices it counts individual I/Os that exceeded the target within a window. Throttling is applied first by clamping the offender's allowed queue depth (from unlimited down to as low as one outstanding I/O), and — for swap and metadata I/O, which cannot simply be queued — by injecting an artificial per-process delay, visible as the `delay` field in the offender's `io.stat`. When the protected group meets its target again, peers are progressively unthrottled. Because it only acts under contention, it costs nothing when the device is idle. `io.latency` landed in Linux 4.19.

systemd exposes it as `IODeviceLatencyTargetSec=/dev/nvme0n1 25ms`, which is the cleanest way to protect a data loader on a shared node.

**`io.cost.qos` and `io.cost.model`** — iocost, root cgroup only, one line per device:

```
$ cat /sys/fs/cgroup/io.cost.qos
259:0 enable=1 ctrl=auto rpct=95.00 rlat=5000 wpct=95.00 wlat=5000 min=50.00 max=150.00

$ cat /sys/fs/cgroup/io.cost.model
259:0 ctrl=auto model=linear rbps=1150000000 rseqiops=880000 rrandiops=650000 \
      wbps=980000000 wseqiops=720000 wrandiops=520000
```

| `io.cost.qos` key | Meaning |
|---|---|
| `enable` | 0/1 — whether weight-based cost control is active on this device |
| `ctrl` | `auto` (controller tunes params) or `user` |
| `rpct` / `rlat` | read latency percentile [0,100] and its threshold in µs |
| `wpct` / `wlat` | write latency percentile and threshold in µs |
| `min` / `max` | bounds on vrate scaling, [1, 10000] % |

| `io.cost.model` key (linear) | Meaning |
|---|---|
| `rbps` / `wbps` | max sequential read/write throughput, bytes/s |
| `rseqiops` / `wseqiops` | max 4 KiB sequential read/write IOPS |
| `rrandiops` / `wrandiops` | max 4 KiB random read/write IOPS |

The reason iocost exists is the flaw in raw bandwidth caps: **a 128 KiB sequential write and a 4 KiB random write cost the device wildly different amounts of real service time**, so a bytes-per-second cap either starves the sequential workload or lets the random workload consume the device's actual capability while nominally staying "under budget." iocost builds a per-device cost model instead. Each I/O is priced from the linear coefficients above — a per-byte term plus a base cost that differs for sequential vs random (the classifier treats I/Os more than 16 MiB apart as random). Each cgroup gets a **virtual time (vtime)** budget that advances inversely to its hierarchical weight: a cgroup holding 12.5% of the active weight has its clock run 8× slower than device time, so a 10 ms I/O consumes 80 ms of its budget. Overspend becomes tracked debt (`abs_vdebt`) repaid over subsequent periods rather than a hard stall. A QoS loop then scales the global `vrate` up or down between `min` and `max` based on whether the device's Nth-percentile completion latency is meeting `rlat`/`wlat`. `blk-iocost` was upstreamed by Meta and merged in **Linux 5.4**.

Deriving the model coefficients by hand is unpleasant, which is why systemd ≥ 253 ships an `iocost` udev builtin plus an hwdb of per-model solutions generated by the `iocost-benchmark` project: if your drive is in that database, `io.cost.model` and `io.cost.qos` are populated automatically at boot, tunable via `IOCOST_SOLUTIONS=` in `/etc/systemd/iocost.conf`. Check with `udevadm info /dev/nvme0n1 | grep -i iocost`.

**`io.pressure`** — PSI, per cgroup, and the reason this lesson exists:

```
$ cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable-pod3f9a_1c.slice/io.pressure
some avg10=42.31 avg60=38.77 avg300=21.04 total=99183721004
full avg10=31.02 avg60=28.55 avg300=15.61 total=71220119442
```

`some` is the percentage of wall-clock in which *at least one* runnable task in this cgroup was stalled waiting on I/O; `full` is the percentage in which *every* runnable task was stalled — the cgroup got no work done at all. `total` is cumulative microseconds of stall, which is the field to use for rate calculations in Prometheus (`rate(psi_total[5m])`) because the `avgN` fields are already-smoothed and cannot be re-aggregated.

**`io.prio.class`** — sets the I/O priority class for I/O issued by this cgroup: `no-change` (0), `promote-to-rt` (1), `restrict-to-be` (2), `idle` (3). Only BFQ and mq-deadline act on I/O priority; with `none` it has no effect.

### 10. Schedulers, queue depth, and writeback throttling

The scheduler layer is the last thing between the block layer and the driver, and its defaults are frequently misunderstood.

**What is attached by default.** In the blk-mq world (the only world since Linux 5.0 removed the legacy single-queue path), the kernel attaches `mq-deadline` when a device presents **one** hardware queue, and attaches **nothing** (`none`) when it presents several. NVMe drives present one hardware queue per CPU, so a stock NVMe device runs with no scheduler at all. Many distributions override this with udev rules — Debian and Ubuntu ship rules that select `bfq` for rotational devices and leave `none` or select `kyber` for NVMe.

```
$ cat /sys/block/nvme0n1/queue/scheduler
[none] mq-deadline kyber bfq
$ cat /sys/block/sda/queue/scheduler
mq-deadline kyber [bfq] none
```

The bracketed entry is the active one. Choose with:

```
$ echo mq-deadline | sudo tee /sys/block/nvme0n1/queue/scheduler
```

| Scheduler | Mechanism | Per-cgroup fairness | Use it when |
|---|---|---|---|
| `none` | FIFO passthrough to the hardware queues | none | Fast NVMe where CPU cost per I/O matters and cgroup control comes from `io.max`/`iocost` |
| `mq-deadline` | Separate read/write FIFOs with expiry deadlines; reads get a shorter deadline | none (respects ioprio) | Single-queue SATA/SAS; anywhere you need a read-latency bound without BFQ's cost |
| `kyber` | Two queues (read/sync-write), self-tunes dispatch depth to hit configured target latencies | none | Fast multi-queue devices where you want latency targets and minimal overhead |
| `bfq` | Proportional-share budget scheduler with per-cgroup weights (`bfq.weight`) and heuristics for interactivity | **yes** | Rotational disks, desktop/interactive latency, or when you need scheduler-level cgroup weighting; too CPU-expensive at multi-hundred-kIOPS rates |

**Queue depth.** `/sys/block/<dev>/queue/nr_requests` is how many requests the block layer keeps queued per hardware context. Raising it improves merging and throughput and raises worst-case latency, because a newly arriving request may sit behind a full queue. It is a direct throughput-vs-latency dial.

**Writeback throttling (wbt)** solves a problem you may have hit without naming it: a burst of buffered writes reaching the device starves concurrent reads for hundreds of milliseconds, because writes are submitted in bulk and serviced in order. wbt is an rq-qos policy that monitors read completion latency and, when it degrades, throttles background/buffered write submission — scaling the allowed write depth down from a default of 16 toward 1, and back up when reads recover. Its target lives in `/sys/block/<dev>/queue/wbt_lat_usec`, defaulting to **2000 µs (2 ms) for non-rotational devices and 75000 µs (75 ms) for rotational ones** (`block/blk-wbt.c`). Writing `0` disables wbt; `-1` restores the computed default. `wbt_lat_usec=0` is a common tuning-guide "fix" that then makes read latency worse under write bursts.

```
$ cat /sys/block/nvme0n1/queue/wbt_lat_usec
2000
$ cat /sys/block/sda/queue/wbt_lat_usec
75000
```

### 11. Why a slow disk stalls "unrelated" pods

Assemble the pieces into the failure you will actually be paged for.

A GPU node has one shared NVMe device backing every pod's scratch space. Pod A is a training job writing a 40 GB checkpoint. Pod B is a data loader doing small random reads to feed a different job's GPUs. Neither pod has any `io.max`, `io.latency`, or `iocost` configuration, because nothing sets those by default.

The mechanism, step by step:

1. Pod A's checkpoint write lands in the page cache and returns instantly; dirty pages accumulate.
2. At `vm.dirty_background_ratio` the flusher threads issue large sequential writes. cgroup writeback is active on ext4, so those bios are correctly charged to pod A — but charging is not limiting, and nothing is configured to limit.
3. The hardware queues fill with pod A's write commands. With scheduler `none` there is no reordering and no fairness: submitted first, executed first.
4. Pod B submits a 4 KiB read. It queues behind pod A's backlog. Its *service* time is unchanged — the device is as fast as ever — but its *queueing* time is now tens of milliseconds.
5. Pod B's threads block in `io_schedule`; its `io.pressure` `some` climbs, and `full` climbs whenever all its runnable threads are blocked at once. Its GPUs idle waiting for batches.
6. Nobody exceeded a quota, and pod B's own `io.stat` shows *fewer* `rios` than usual — it is doing less I/O because it cannot get any through.

Step 6 is the crux and the reason `io.pressure` is not optional: **the victim's own utilization metrics look better than normal, not worse.** Only a stall metric shows the injury. In lesson 09's USE terms, `%util` is the utilization column and is nearly meaningless for a device servicing thousands of requests concurrently; `aqu-sz`/`await` are better but device-wide; `io.pressure` is the **per-cgroup saturation** column and the only one that can name a victim.

This is structurally lesson 07's conntrack exhaustion again: one finite, kernel-managed, shared resource with no default per-tenant partition, where one tenant's legitimate use degrades an unrelated one. The fix has the same shape — an explicit per-tenant boundary instead of trusting the resource to arbitrate: `io.latency` on the loader (protect the victim), `io.max` on the checkpointer (cap the aggressor), or iocost (price both fairly).

## Perspectives

**Kernel-mechanism view.** Everything here follows from two cgroup-v2 design decisions. The **no-internal-process constraint** forces the tree into branches-and-leaves, which is exactly the shape systemd's slice/service/scope trio expresses — slices are the branches the kernel demands, services and scopes are the leaves. The **absence of any locking or transaction on cgroup attribute files** forces the single-writer discipline, and since the kernel cannot enforce it, enforcement is social: an ownership boundary (`Delegate=`), a permission bit, and a promise not to write below it. The block side repeats the theme: the kernel gives you three independent control points on one path — bio-level throttling before request allocation, rq-qos at issue and completion, the scheduler at dispatch — and your job is to know which one your problem lives at. Choose the wrong layer and your limit is real but irrelevant.

**Operator/SRE view.** When a node is pressured, the first command is not `top`. `top` gives a flat process list with no ownership boundary, so it cannot answer "which tenant." `systemd-cgtop` sorts live by slice with CPU, memory, I/O and task counts in one view; `systemd-cgls` answers "where exactly does this PID live." The second habit is to reach for the *stall* metric rather than the *busy* metric: walking `io.pressure` down the slice tree tells you who is suffering, walking `io.stat` down the same tree tells you who is causing it. Those are different cgroups, and the whole diagnostic skill is not confusing them. Remember to turn on `IOAccounting=` — with systemd's default of `DefaultIOAccounting=no`, half of that view is blank.

**GPU-fleet view.** A multi-tenant GPU node usually has exactly one local NVMe and one shared network mount, both unpartitioned by default, carrying every checkpoint write, dataset read, image-layer extraction and log flush. The economics are stark: the storage device costs a few hundred dollars and the accelerators it starves cost $16–24 per hour, so a configuration change that costs nothing (`IODeviceLatencyTargetSec=` on the data-loader pods) protects a resource three orders of magnitude more expensive than the one being managed. The fleet-design rule that follows: treat local storage as something that must be *explicitly* partitioned before a node is production-ready, in the same class as `nf_conntrack_max` (lesson 07) and `kube-reserved` (lesson 03) — not tuned reactively after an incident.

**Architecture/economics view.** Two independently developed, independently released systems — systemd, owned by the distribution, and the kubelet, owned by Kubernetes — manage overlapping state on one machine with **no coordination protocol between their teams**, by agreeing on a filesystem boundary and a permission bit and nothing else. Neither models the other's internals, neither must be version-matched, and a bug in one cannot corrupt the other's state. A shared lock, a coordination daemon, or a negotiated API would have welded two release trains together permanently. **When two systems must share a resource that assumes one owner, partitioning ownership statically is almost always cheaper than coordinating dynamically** — and the cgroup-driver mismatch is the empirical proof, since the one configuration where the partition is not drawn is the one that produces years of unexplainable node instability.

## Real-world use cases

- **Meta's `blk-iocost`, and why raw caps were not enough.** Meta ran into the limits of `io.max` and weight-only control at fleet scale: bandwidth and IOPS caps do not correspond to a device's real service capacity, because the cost of an I/O depends on its size and on whether it is sequential or random. A cap tuned to protect against random-write pressure throttles sequential workloads far below what the device can do; a cap tuned for sequential throughput lets random workloads saturate the device while nominally under budget. Their answer, upstreamed as `blk-iocost` and merged in **Linux 5.4**, is a per-device linear cost model (bytes/s plus separate sequential and random 4 KiB IOPS coefficients) feeding a virtual-time budget scheduler, with a QoS loop that scales the global rate to hold an Nth-percentile latency target. What it shows: the `io.max`/`io.weight` interface in §9 is the *first* generation of block-I/O control, and the production evolution was to stop measuring I/O in bytes and start measuring it in device-time.
- **The cgroup-driver mismatch, as a documented failure class.** Running a `cgroupfs`-driver kubelet on a systemd-managed host is called out by name in the Kubernetes documentation, which states that two cgroup managers produce "two views of the available and in-use resources in the system" and that this leads to instability under resource pressure. The remedy is not a runtime flip: the docs classify changing a joined node's cgroup driver as a sensitive operation that produces pod-sandbox recreation errors kubelet restarts do not clear, and recommend replacing the node. kubeadm has defaulted to `cgroupDriver: systemd` since **v1.22**. What it shows: single-writer is not an abstract design preference — the orchestrator's own installation guide treats violating it as a node-rebuild-level mistake. (No single verified company postmortem was found describing this pattern, so it is cited here as documented operational guidance rather than a production war story.)
- **`iocost` device solutions shipped by systemd.** Because deriving a device's cost-model coefficients requires benchmarking, the `iocost-benchmark` project publishes measured solutions per drive model, and systemd ≥ 253 ships a udev builtin plus hwdb entries that apply them at boot, configured through `IOCOST_SOLUTIONS=` in `iocost.conf(5)`. What it shows: the practical barrier to using the better controller was never the kernel interface — it was that nobody could produce the numbers, so the ecosystem solved it as a data-distribution problem.

## Worked example

**Ticket:** *"Pods on node-12 report slow dataset reads. The disk looks fine — `iostat` says 4% utilization."* Node runs containerd with `cgroupDriver: systemd`, one NVMe (`nvme0n1`, `259:0`), ext4.

*(Transcripts below are representative of this scenario, formatted exactly as the tools emit; the numbers are the ones this walkthrough reasons about.)*

**Step 1 — establish the tree and find the tenant boundaries.**

```
$ systemd-cgls --no-pager -l | head -30
Control group /:
-.slice
├─kubepods.slice
│ ├─kubepods-burstable.slice
│ │ ├─kubepods-burstable-pod3f9a1c22_7b04_4d51_b2ef_0a9c1d4e88f1.slice
│ │ │ ├─cri-containerd-9c2b7f1e0a4d....scope
│ │ │ │ └─48213 python3 /app/train.py
│ │ │ └─cri-containerd-1b3d5e7f9a2c....scope
│ │ │   └─48190 /pause
│ │ └─kubepods-burstable-pod7c1e0b44_2a95_4f0e_9d3a_ff1c2e6b0d77.slice
│ │   └─cri-containerd-4a6c8e0b2d4f....scope
│ │     └─51002 python3 /app/checkpointer.py
│ └─kubepods-besteffort.slice
├─system.slice
│ ├─containerd.service
│ │ └─654 /usr/local/bin/containerd
│ └─kubelet.service
│   └─987 /usr/bin/kubelet
└─user.slice
  └─user-1000.slice
    └─session-3.scope
      └─20114 -bash
```

Two burstable pods. `pod3f9a…` is the complaining trainer; `pod7c1e…` runs a checkpointer.

**Step 2 — ask who is *losing* time, not who is busy.** Walk `io.pressure` down the tree:

```
$ for p in /sys/fs/cgroup /sys/fs/cgroup/kubepods.slice \
           /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod3f9a1c22_*.slice \
           /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod7c1e0b44_*.slice; do
    printf '%-72s %s\n' "${p#/sys/fs/cgroup}" "$(awk '/^some/{print $2}' $p/io.pressure)"
  done
/                                                                        avg10=39.84
/kubepods.slice                                                          avg10=39.71
/kubepods.slice/.../kubepods-burstable-pod3f9a1c22_....slice             avg10=61.22
/kubepods.slice/.../kubepods-burstable-pod7c1e0b44_....slice             avg10=2.03
```

The trainer is stalled 61% of the last ten seconds on I/O. The checkpointer is stalled 2%. **The victim is `pod3f9a`.**

**Step 3 — ask who is *causing* it.** Walk `io.stat` down the same tree:

```
$ cat /sys/fs/cgroup/kubepods.slice/.../kubepods-burstable-pod3f9a1c22_*.slice/io.stat
259:0 rbytes=2214592512 wbytes=8192 rios=540672 wios=2 dbytes=0 dios=0

$ cat /sys/fs/cgroup/kubepods.slice/.../kubepods-burstable-pod7c1e0b44_*.slice/io.stat
259:0 rbytes=0 wbytes=42949672960 wios=327680 rios=0 dbytes=0 dios=0
```

Read the two lines arithmetically:

```
  victim  (pod3f9a):  2,214,592,512 B / 540,672 reads = 4,096 B  per read
  aggressor(pod7c1e): 42,949,672,960 B / 327,680 writes = 131,072 B per write
```

Small random reads against 128 KiB sequential writes — the exact pattern from §11, and the exact pattern raw bandwidth caps handle badly.

**Step 4 — confirm the device is not the problem.**

```
$ iostat -xz 2 2 | tail -4
Device   r/s   rkB/s r_await rareq-sz    w/s    wkB/s w_await wareq-sz aqu-sz %util
nvme0n1 2210  8840.0   28.41     4.00   2680 343040.0    1.92   128.00  62.31  99.7
```

`%util` is 99.7% and it means almost nothing — the device has one hardware queue per CPU and 62 requests outstanding, which for a modern NVMe is comfortable. What *does* mean something is the asymmetry: `r_await` is **28.41 ms** while `w_await` is **1.92 ms**. Reads are waiting fifteen times longer than writes on the same device. That is a queueing artefact, not a device defect: the writes are large and were submitted in bulk, so each 4 KiB read waits behind a batch of 128 KiB writes.

**Step 5 — see the latency distribution, not the average.**

```
$ sudo biolatency-bpfcc -D 10 1
Tracing block device I/O... Hit Ctrl-C to end.

disk = 'nvme0n1'
     usecs               : count     distribution
       128 -> 255        : 1204     |*****                                   |
       256 -> 511        : 2891     |***********                             |
       512 -> 1023       : 3102     |************                            |
      1024 -> 2047       : 2044     |********                                |
      2048 -> 4095       : 1188     |****                                    |
      4096 -> 8191       : 640      |**                                      |
      8192 -> 16383      : 402      |*                                       |
     16384 -> 32767      : 9821     |****************************************|
     32768 -> 65535      : 4402     |*****************                       |
```

Bimodal, and that shape is the entire diagnosis. There is a fast population under ~2 ms (the writes, which the device absorbs easily) and a second, larger population at **16–64 ms** (the reads that queued). An average would have reported something like 12 ms and described neither population. Add `-Q` to include OS queue time and the upper mode grows further, which confirms that the wait is queueing rather than device service time.

**Step 6 — quantify the cost before proposing a fix.**

```
  Trainer stalled on I/O:  io.pressure some avg10 = 61.2%
  GPUs gated on batches ⇒ GPU idle fraction        ≈ 0.61
  8 × H100 at ~$2.50/GPU-hr                        = $20.00/hr node accelerator cost
  wasted per hour   = 0.61 × $20.00                = $12.20/hr
  per day           = $12.20 × 24                  = $292.80/day
  over a 5-day run  = $292.80 × 5                  ≈ $1,464
```

*(Substitute your own committed GPU rate; the structure is the transferable part.)*

**Step 7 — apply the right control at the right layer.** Two options, and the choice matters.

*Option A — cap the aggressor with `io.max`.* Blunt but immediate:

```
$ sudo systemctl set-property \
      kubepods-burstable-pod7c1e0b44_2a95_4f0e_9d3a_ff1c2e6b0d77.slice \
      IOWriteBandwidthMax="/dev/nvme0n1 200M" IOAccounting=yes

$ cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/\
kubepods-burstable-pod7c1e0b44_2a95_4f0e_9d3a_ff1c2e6b0d77.slice/io.max
259:0 rbps=max wbps=200000000 riops=max wiops=max
```

Note `200000000`, not `209715200` — systemd's I/O bandwidth suffixes are base 1000. And note the cost: the checkpointer is now capped at 200 MB/s **even when the device is idle**, because `io.max` is not work-conserving. Checkpoint duration goes up permanently.

*Option B — protect the victim with `io.latency`.* Targeted and work-conserving:

```
$ sudo systemctl set-property \
      kubepods-burstable-pod3f9a1c22_7b04_4d51_b2ef_0a9c1d4e88f1.slice \
      IODeviceLatencyTargetSec="/dev/nvme0n1 10ms" IOAccounting=yes

$ cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/\
kubepods-burstable-pod3f9a1c22_7b04_4d51_b2ef_0a9c1d4e88f1.slice/io.latency
259:0 target=10000
```

Now the kernel watches the trainer's read latency; only when it exceeds 10 ms does it clamp the checkpointer's queue depth, and it releases the clamp as soon as the trainer recovers. When the trainer is not reading, the checkpointer gets the whole device.

**Step 8 — verify with the same measurement you started from.**

```
$ awk '/^some/{print}' /sys/fs/cgroup/kubepods.slice/.../kubepods-burstable-pod3f9a1c22_*.slice/io.pressure
some avg10=7.44 avg60=22.10 avg300=48.93
```

`avg10` has dropped from 61.2 to 7.4 while `avg300` still carries the pre-fix history — which is exactly how you know the change took effect and roughly when. Re-run `biolatency` and the upper mode should collapse from the 16–64 ms buckets down toward 1–4 ms.

**One-line verdict:** *node-12's trainer lost 61% of wall-clock to read queueing behind a co-tenant's 128 KiB sequential checkpoint writes on a shared NVMe with no I/O isolation configured; `%util` was 99.7% and irrelevant, `r_await` 28 ms vs `w_await` 1.9 ms was the tell, and per-cgroup `io.pressure` named the victim. Fixed by setting a 10 ms `io.latency` target on the trainer's pod slice — work-conserving, so the checkpointer keeps full bandwidth whenever the trainer is idle. ~$12/hour of GPU time recovered.*

## Practice

1. **Map the tree by hand.** Run `systemd-cgls` on a machine or node. Identify one `*.slice`, one `*.service`, and one `*.scope`, and write down the exact directory under `/sys/fs/cgroup` for each. Then verify the path encoding on a nested slice: pick a slice whose name contains a dash (`user-1000.slice`, or a `kubepods-burstable-pod….slice` on a node) and confirm by `ls` that its parent directory is the name up to the last dash. Finally, take any container PID and derive its full cgroup path from `/proc/<pid>/cgroup` alone — no `docker`, no `kubectl`, no `crictl`.

2. **Prove the directive → file mapping.** Start a constrained transient scope and read back every file it wrote:
   ```
   systemd-run --scope --unit=lab-limits \
       -p MemoryMax=200M -p CPUQuota=25% -p TasksMax=64 -p IOAccounting=yes \
       sleep 300 &
   C=/sys/fs/cgroup/system.slice/lab-limits.scope
   cat $C/memory.max $C/cpu.max $C/pids.max
   ```
   Confirm `memory.max` is `209715200` (base 1024) and `cpu.max` is `25000 100000`. Then trigger the memory limit with `stress-ng --vm 1 --vm-bytes 250M` inside the same scope and read `memory.events` — record which counters incremented (`max`, `oom`, `oom_kill`) and explain the difference between them.

3. **Show that `set-property` is live.** Pick a harmless unit, `cat` its `memory.high`, run `systemctl set-property <unit> MemoryHigh=512M`, and `cat` the file again **without restarting anything**. Then find the drop-in file systemd created under `/etc/systemd/system/<unit>.d/`. Repeat with `--runtime` and find the file under `/run/systemd/system/` instead. Write one sentence on when you would use each.

4. **Read the block-I/O stack under real load.** Generate device I/O that bypasses the page cache, so it actually reaches the device:
   ```
   fio --name=load --filename=/var/tmp/fio.dat --size=2G --rw=randread \
       --bs=4k --direct=1 --numjobs=4 --time_based --runtime=60 &
   ```
   While it runs, capture, in one lab-log entry: `iostat -x 2` (note `%util`, `aqu-sz`, `r_await`, `rareq-sz`), the root `io.pressure`, and a `biolatency -D 5 1` histogram. Then re-run the identical fio **without** `--direct=1` and explain why the device-level numbers change so much. This is the page-cache lesson from §7 made concrete.

5. **Noisy-neighbour simulation, before and after.** Run two transient scopes concurrently against the same device: a heavy sequential writer (`fio --rw=write --bs=128k --direct=1`) standing in for a checkpointer, and a light random reader (`fio --rw=randread --bs=4k --direct=1 --rate_iops=500`) standing in for a data loader. Capture the **reader's own** `io.pressure some avg10` while the writer runs unconstrained. Then constrain the writer — `systemctl set-property <writer-scope> IOWriteBandwidthMax="/dev/<dev> 100M"` — and capture the reader's `io.pressure` again. If your kernel has iolatency, also try the other direction: set `IODeviceLatencyTargetSec` on the *reader* and leave the writer alone. Record all three numbers and explain which approach is work-conserving and why that matters when the reader is idle.

6. **Check what proportional control you actually have.** Read `/sys/fs/cgroup/io.cost.qos` and note whether `enable=1` on your device; read `/sys/block/<dev>/queue/scheduler` and note the active scheduler; then determine from §9's decision table whether writing `io.weight` on your box would do anything at all. Write down the answer and the evidence.

**Acceptance:** a short note containing (a) one **systemd slice → cgroup path → resource file** trace, naming all three and showing the file's contents; (b) one **io-pressure observation** under load — the `some avg10` value, alongside what `%util` claimed at the same moment, with a sentence on why they disagree; and (c) the **noisy-neighbour before/after** from task 5, with the reader's `io.pressure some avg10` in both states and a statement of which control you used and why. Feeds the *Anatomy of a Container* deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md).

## Common pitfalls

1. **Believing the kubelet gets `kubepods.slice` delegated.** systemd refuses to delegate slice units at all, because a slice is an inner node whose children systemd itself creates — delegating it would put two writers in one directory. With the systemd cgroup driver the kubelet never writes the tree; it calls `StartTransientUnit` over D-Bus and systemd writes. Delegation appears one level down, on the container runtime's service (`containerd.service` ships `Delegate=yes`) and on the container scopes it creates. Getting this backwards leads people to "fix" node problems by adding `Delegate=yes` to `kubelet.service`, which does nothing.

2. **Setting `IOWeight=` and expecting proportional sharing.** `io.weight` is iocost's interface, and iocost is disabled per device by default (`io.cost.qos enable=0`). Writing a weight into a file that no active controller reads produces no error and no effect. BFQ's cgroup weight is a *different file* (`bfq.weight` on cgroup v2), and BFQ disables iocost when selected. Before trusting a weight, check `io.cost.qos` and `/sys/block/<dev>/queue/scheduler`.

3. **Capping buffered writes and concluding the cap does not work.** A buffered `write()` returns as soon as the data is in the page cache; the bio that `io.max` throttles is submitted later by writeback, possibly from a flusher thread. So `dd` without `oflag=direct` blows straight through your limit and then stalls hard when dirty pages hit `vm.dirty_ratio`. Use `--direct=1`/`oflag=direct` when testing limits, and remember that on any filesystem outside {ext2, ext4, btrfs, f2fs, xfs} writeback is attributed to the **root** cgroup, so per-pod write limits are silently ineffective.

4. **Trusting `%util` as the saturation signal for a shared device.** `%util` is the fraction of time at least one request was in flight — a real utilization for a one-request-at-a-time disk, and nearly meaningless for a device with thousands of concurrent queue entries, which pins at 100% while at a few percent of capability. The correct per-tenant saturation signal is `io.pressure`; `aqu-sz` and the read/write `await` split are the useful device-level cross-checks. The same trap as lesson 09, one layer down.

5. **Diagnosing from the aggressor's metrics instead of the victim's.** Under contention the victim's own I/O counters *improve* — fewer IOPS, fewer bytes — because it cannot get I/O through. Only a stall metric shows the injury. Walk `io.pressure` to find who is hurt, and `io.stat` to find who is causing it; they are different cgroups and confusing them sends you to optimize the wrong workload.

6. **Assuming `systemctl set-property` needs a restart.** It writes the kernel file immediately over D-Bus, then additionally drops a persistent unit file; `--runtime` makes it live-only. Conversely, editing a unit file by hand and running `daemon-reload` does *not* apply resource properties to a running unit — it only changes what happens at next start.

7. **Reserving without enforcing.** `systemReserved`/`kubeReserved` subtract from allocatable so the scheduler stops placing pods, but nothing bounds `system.slice` unless you also create the reserved cgroups and add them to `enforceNodeAllocatable`. Until then, a leaking node agent eats into what you promised pods. And when you do enforce, enforcing `system-reserved` can starve or OOM-kill sshd and the kubelet — start with the compressible-only variants.

8. **Expecting the kubelet to put a CPU quota on `kubepods.slice`, or finding a pod UID by grepping.** Neither works. The kubelet sets `memory.max`, CPU *weight*, and optionally `pids.max` on `kubepods` — CPU is compressible, so node-level control is proportional, not a hard cap (individual pods with `limits.cpu` still get `cpu.max` on their own slice). And with the systemd driver every `-` in a name component is escaped to `_`, so `4d0e2ff5-8f43-…` appears in the tree as `4d0e2ff5_8f43_…`.

9. **Leaving `io.max` in place as a permanent fix.** It is not work-conserving: a capped cgroup waits even when the device is completely idle. Fine as an emergency brake, bad as a steady state. The work-conserving options are `io.latency` (protect one workload, throttle peers only on a miss) and iocost (price every I/O and share what is left).

## Self-check

**Q1. How do systemd and the kubelet both manage cgroups on one node without racing — and what is the mechanism, precisely?**
**Answer:** They do not write the same files. With `cgroupDriver: systemd` the kubelet writes *nothing*: it calls `org.freedesktop.systemd1.Manager.StartTransientUnit` over D-Bus with the slice name and resource properties, and systemd creates the directory, enables controllers in `cgroup.subtree_control`, and writes `memory.max`/`cpu.max`/`cpu.weight`. One writer. Delegation enters one level lower: the runtime's unit (`containerd.service`) carries `Delegate=yes`, and the container scopes it creates are delegated — systemd creates the scope's cgroup, enables the requested controllers, makes it writable, then never touches anything below it (no `mkdir`, no `rmdir`, no attribute writes). Slices cannot be delegated, because systemd creates and destroys their children itself. The kernel does not enforce single-writer on attribute files at all — no lock, no compare-and-swap — which is exactly why the discipline must be arranged out of band by an ownership boundary plus a permission bit. General principle: for any shared resource that assumes one owner, partition ownership statically instead of building coordination on top of it.

**Q2. `service`, `scope`, `slice` — what is each, and how do you tell them apart on a running system?**
**Answer:** A **service** is processes systemd forked itself from `ExecStart=`; systemd owns their full lifecycle (restart, watchdog, stop). A **scope** is processes created by somebody else — a login session, a container runtime's child, `systemd-run --scope` — that systemd merely *adopts* into a cgroup for accounting and limits; it never forked them and does not restart them. A **slice** is a branch node that holds no processes at all; it exists because cgroup v2's no-internal-process constraint forbids a cgroup from both holding tasks and distributing controllers to children, so you need pure branches to hang subtree-wide limits on. Slice names encode their own path: each dash is one level down, so `kubepods-burstable.slice` lives at `/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice`, and the root slice is `-.slice` = `/sys/fs/cgroup`. On a running system, `systemd-cgls` shows the shape directly, and a leaf directory with a non-empty `cgroup.procs` is a service or scope while an empty one with children is a slice.

**Q3. Name five systemd resource-control directives and the exact cgroup file each writes, including the value transformation.**
**Answer:** `CPUQuota=20%` → `cpu.max` = `20000 100000` (quota µs, period µs; period comes from `CPUQuotaPeriodSec=`, default 100 ms). `CPUWeight=200` → `cpu.weight` = `200` (range 1–10000, kernel default 100). `MemoryMax=4G` → `memory.max` = `4294967296` (base **1024**) — the hard limit that OOM-kills; `MemoryHigh=` → `memory.high`, which throttles and reclaims but never kills. `TasksMax=4096` → `pids.max`. `IOWriteBandwidthMax=/dev/nvme0n1 20M` → `io.max` = `259:0 wbps=20000000` (base **1000** — the asymmetry with memory is real and documented). `IODeviceLatencyTargetSec=/dev/nvme0n1 25ms` → `io.latency` = `259:0 target=25000` (µs). Apply any of them live with `systemctl set-property`, which writes the kernel file immediately *and* persists a drop-in; `--runtime` makes it live-only.

**Q4. Trace a 4 KiB buffered write from `write(2)` to the platter, and name every point where a cgroup control could act.**
**Answer:** `write()` → VFS → filesystem → **page cache**: the folio is copied in, marked dirty, and the syscall returns — no device I/O has occurred, so nothing has been throttled yet. Later, **writeback** picks up the dirty folio, triggered by `vm.dirty_background_ratio` (10%), `vm.dirty_ratio` (20%, which blocks the writer synchronously), the 30 s expiry, or `fsync()`; cgroup writeback attributes the bio to the cgroup owning the inode, implemented on ext2/ext4/btrfs/f2fs/xfs only (elsewhere everything is charged to root). The bio enters **`submit_bio_noacct()`**, which charges `io.stat` and calls `blk_throtl_bio()` — where **`io.max`** parks the bio on a per-cgroup service queue, before a request is even allocated. Survivors become requests on blk-mq's per-CPU software queues, where the **rq-qos** hooks run: **wbt** (`wbt_lat_usec`, default 2000 µs SSD / 75000 µs HDD), **`io.latency`**, and **iocost**. Then the **I/O scheduler** (`none` by default on multi-queue NVMe, `mq-deadline` on single-queue devices, `bfq` for scheduler-level cgroup weights via `bfq.weight`), the hardware dispatch queues, the driver, the device. Completion fires rq-qos `done()`/`done_bio()`, which is how the latency-driven controllers learn what the device did.

**Q5. What does `io.pressure` tell you that `iostat`'s `%util` and `await` cannot?**
**Answer:** PSI measures **lost time, per cgroup**: `some` is the share of wall-clock in which at least one runnable task in that cgroup was stalled on I/O, `full` the share in which every runnable task was. `%util` is device busy-time — the fraction of the interval with at least one request in flight — which was a real utilization for one-request-at-a-time disks and is close to meaningless for an NVMe with thousands of concurrent queue entries; it pins at 100% while the device is at a few percent of capability. `await` is device-wide average request latency, so it cannot attribute anything to a tenant. Only `io.pressure` answers "how much wall-clock did *this pod* lose to I/O," which is what you need to name a victim. The clinching detail: under contention the victim's own `io.stat` counters go *down*, because it cannot get I/O through — so every utilization-shaped metric on the victim looks healthier than usual, and only a stall metric shows the injury.

**Q6. A GPU node has one shared NVMe, no `io.max`, no `io.latency`, no iocost. A checkpointing pod and a data-loader pod share it. Walk the failure and the fix, and name the structural parallel elsewhere in this module.**
**Answer:** The checkpointer's buffered writes accumulate in the page cache and flush as large sequential bios; with scheduler `none` there is no reordering and no fairness, so the device's queues fill with its commands. The loader's 4 KiB reads queue behind them — unchanged service time, hugely inflated queueing time — so its threads block in `io_schedule`, its `io.pressure` climbs, and the GPUs it feeds idle. Neither pod exceeded a quota, because none was set: **block bandwidth has no default per-cgroup fairness**, unlike CPU (every cgroup gets weight 100) or memory (the OOM killer eventually intervenes). Fixes, in increasing sophistication: `io.max` on the aggressor (immediate, but a hard non-work-conserving ceiling that slows checkpoints even on an idle device); `io.latency` on the *victim* (a 10 ms target; the kernel throttles peers only while the target is missed, by clamping their queue depth and injecting delay, and releases them on recovery); or iocost, which prices every I/O by size and sequentiality against a per-device cost model. The structural parallel is lesson 07's conntrack exhaustion — one finite kernel-managed resource, no default per-tenant partition, one tenant's legitimate use degrading an unrelated neighbour, and the same fix shape.

**Q7. You set `IOWeight=500` on a slice and nothing changes. Why, and how do you check?**
**Answer:** `IOWeight=` writes `io.weight`, which is the **iocost** controller's interface — and iocost is disabled per device by default. Check `cat /sys/fs/cgroup/io.cost.qos`: if the line for your device reads `enable=0`, nothing reads the weight. Also check `cat /sys/block/<dev>/queue/scheduler`: if BFQ is active it deliberately disables iocost, and BFQ's own cgroup weight lives in a different file, `bfq.weight`. So you have three real options: enable iocost with a model and QoS line (or let systemd ≥ 253's `iocost` udev builtin apply a benchmarked solution from hwdb), switch the scheduler to `bfq` and use `bfq.weight`, or abandon proportional control and use `io.max`/`io.latency`, which work regardless.

## Connections & what's next

This is the **last lesson in module 01b**, and it closes the loop the module opened. Lesson 03 showed you the cgroup-v2 interface files; this lesson showed you who writes them and by what protocol. Lesson 04 gave you PSI as the saturation signal; this lesson gave you the per-cgroup `io.pressure` variant and the reason a victim's utilization metrics look *better* than normal while it is being starved. Lesson 09 gave you the USE method and the tools to localize a bottleneck to the storage path; this lesson gave you the layer-by-layer map of that path and the specific file to write at each layer. And the parallel drawn in §11 to lesson 07's conntrack exhaustion is meant to stick: **shared, unpartitioned, kernel-managed resources are a recurring shape across this entire module**, not a disk-specific quirk. Every one of them wants the same treatment — an explicit per-tenant boundary, set before the incident.

From here: consolidate everything into the **[Anatomy of a Container](../practice/anatomy-of-a-container/README.md)** deliverable and work through the **[checkpoint](../checkpoint.md)**, which is the completion gate for this module. Every pass criterion draws on a lesson you have now finished — locating ns + cgroup state by hand (this lesson's §3 gives you the path derivation), explaining an OOM from the log, diagnosing saturation via PSI rather than utilization, mapping a CPU limit to `cpu.max` and reading throttling counters, reasoning about conntrack, and producing a flame graph with a USE walkthrough. Once the checkpoint is passed, the path continues into **[02 — Kubernetes controllers](../../02-kubernetes-controllers/README.md)**, where the cgroup and namespace machinery you now understand from the kernel side becomes the substrate that the control plane schedules against — and where you will be the one writing the controller that decides what lands in which slice.

## References & further reading

**Primary sources**

- [Kernel docs — `Documentation/admin-guide/cgroup-v2.rst`](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/cgroup-v2.rst) — authoritative for §1 and §9: the no-internal-process constraint, delegation and `nsdelegate`, the complete IO-controller interface with exact field names and formats, and the cgroup-writeback section with its filesystem support list.
- [systemd.resource-control(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html) — the directive → cgroup-file table in §4, the base-1000/base-1024 unit split, and the `CPUQuota` → `cpu.max` conversion.
- [systemd.slice(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.slice.html) — slice semantics, the `-.slice` root, and the dash-encodes-hierarchy naming rule used in §2–§3.
- [systemd — CGROUP_DELEGATION.md](https://systemd.io/CGROUP_DELEGATION/) — the §5 ownership model: what `Delegate=` promises, why slices cannot be delegated, and `DelegateSubgroup=` (systemd 254+). **Correction to the previous version of this lesson:** it claimed the kubelet needs `Delegate=yes` on `kubepods.slice`; slices cannot be delegated at all, and with the systemd driver the kubelet creates cgroups over D-Bus rather than writing them.
- [systemd-system.conf(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd-system.conf.html) — the accounting defaults in §4, notably `DefaultIOAccounting=no`.
- [iocost.conf(5)](https://www.freedesktop.org/software/systemd/man/latest/iocost.conf.html) — systemd ≥ 253's `iocost` udev builtin and `IOCOST_SOLUTIONS=`.
- [Kubernetes — Container runtimes (cgroup drivers)](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) — the `cgroupDriver` field, the "two cgroup managers … two views of the available and in-use resources" warning, the kubeadm v1.22 default, and the replace-don't-reconfigure guidance.
- [Kubernetes — Reserve compute resources for system daemons](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/) — the node-allocatable formula, `--enforce-node-allocatable` (default `[pods]`), the requirement that reserved cgroups pre-exist, and the warning about starving system daemons.
- [kubelet — `pkg/kubelet/cm/cgroup_manager_linux.go`](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/cm/cgroup_manager_linux.go) — the §3 cgroup-name → slice conversion, including dash-to-underscore escaping and the worked `kubepods-burstable-pod1234_abcd_5678_efgh.slice` example.
- [kubelet — `pkg/kubelet/cm/node_container_manager_linux.go`](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/cm/node_container_manager_linux.go) — what is actually written to `kubepods`: memory limit, CPU shares, optional PID limit, and the standing `TODO` confirming no CPU quota is set.
- [Kernel — `block/blk-iocost.c`](https://github.com/torvalds/linux/blob/master/block/blk-iocost.c) — the header comment is the best explanation of the vtime/hweight/debt model, the linear cost coefficients, the 16 MiB random classifier, and the `vrate` QoS loop.
- [Kernel — `block/blk-wbt.c`](https://github.com/torvalds/linux/blob/master/block/blk-wbt.c) — `wbt_default_latency_nsec()`: 2 ms non-rotational, 75 ms rotational, and the depth scaling behind `wbt_lat_usec`.

**Real-world engineering**

- [Meta resctl — I/O resource protection docs](https://facebookmicrosites.github.io/resctl-demo-website/docs/demo_docs/res_protection/IO/) — what it shows: why raw bandwidth caps were insufficient at fleet scale, and the motivation for the cost-model controller that became `blk-iocost` in Linux 5.4.
- [LWN — "IO cost model based work-conserving proportional controller"](https://lwn.net/Articles/793460/) — what it shows: why proportional I/O control needs a device cost model rather than byte counters.
- [LWN — "Introduce io.latency io controller for cgroups"](https://lwn.net/Articles/758697/) — what it shows: the protection-not-limitation framing and the queue-depth-clamping mechanism, from the 4.19 merge discussion.

**Deeper dives**

- [bcc tools — `biolatency`](https://github.com/iovisor/bcc/blob/master/tools/biolatency_example.txt) — the histogram format used in the worked example, plus `-D`, `-Q`, `-F`, `-m`.
- Brendan Gregg, *Systems Performance* (2nd ed.), Disks chapter — the block-I/O stack and why averages mislead on storage; pairs with [Poor Disk Performance](https://www.brendangregg.com/blog/2021-05-09/poor-disk-performance.html).
