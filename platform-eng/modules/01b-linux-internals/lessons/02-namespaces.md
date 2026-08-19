---
lesson: "01b.2"
title: "Namespaces"
module: "01b"
concept: "Namespaces"
status: not-started
est_time: "7h"
prev: "01-processes-and-scheduling.md"
next: "03-cgroups-v2-and-k8s-enforcement.md"
artifacts: []
sources: 14
---

# 01b.2 · Namespaces

> **Concept.** A container is not a kernel object — it's a process wrapped in a set of namespaces (plus a cgroup and a rootfs). Understand namespaces and "container" stops being magic.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits

[01 — Processes and Scheduling](01-processes-and-scheduling.md) established that a process is a `task_struct` — a kernel state machine with a scheduler entry, not a mysterious "running program." That lesson's diagram of `task_struct` had one field marked "→ L2": `nsproxy`. This lesson follows that pointer.

Namespaces do not create a new kind of kernel object. They change what an ordinary `task_struct` is allowed to *see*. A process in a fresh PID namespace is still a `task_struct` on a run queue, picked by EEVDF exactly as lesson 01 described; it just resolves process IDs through a different table. This lesson shows how eight independent namespace types can each virtualize one global kernel resource around a task, which is what makes "container" possible without a hypervisor or a special CPU mode.

It unlocks two things. First, the ability to reconstruct any running container's isolation boundary by hand — a checkpoint requirement. Second, the other half of the container thesis: [03 — cgroups v2 and K8s resource enforcement](03-cgroups-v2-and-k8s-enforcement.md), the anchor lesson of this module, follows the `cgroups` field of the same `task_struct` and adds the *resource-limiting* half to the *view-virtualizing* half covered here.

## Why this matters

The interview question that separates operators from engineers is "what *is* a container?" The wrong answers involve a hypervisor, a special kernel mode, or Docker itself. The right answer is a list of kernel primitives you can point at in `/proc`.

On a GPU fleet you debug the consequences weekly:

- A pod that cannot see the GPU — which is **not a namespace problem at all**, because there is no device namespace, and chasing it as one sends you into the wrong subsystem for an hour.
- Two sidecars fighting over a port — the shared network namespace behaving exactly as designed: one IP, one port space, one routing table for the whole pod.
- A `hostPath` mount leaking into every container on the node — mount propagation, a `shared` peer group, and a `Bidirectional` field somebody set two years ago.
- An init that will not reap zombies, or a container that ignores `SIGTERM` and always takes the full 30-second grace period before `SIGKILL` — PID-namespace init semantics, which are a *kernel* rule, not a Docker convention.
- Node-wide container start-up falling off a cliff above some concurrency — which turned out, at Netflix, to be global VFS mount-lock contention rather than anything about the containers themselves.

If you can reconstruct a running container's namespace set by hand with `unshare`/`nsenter`/`lsns`, you can reason about all five from first principles instead of restarting the pod and hoping. This is also where the module's "kernel mechanism over Linux admin" thesis gets concrete: the GPU/neocloud interview bar (CoreWeave, Datadog, NVIDIA — see the module README) rewards "I can take the abstraction apart" over "I know the `docker` CLI."

## What's new here (calibration)

As with lesson 1, this module skips shell/pipes/permissions, package managers, distro tours, and general networking basics. What is new here specifically:

- All eight namespace types, the **exact `CLONE_NEW*` flag values**, and precisely which single global resource each one virtualizes — including the two people habitually get wrong (IPC does *not* cover `/dev/shm`; CGROUP is cosmetic and does not limit anything).
- The three syscalls (`clone`/`clone3`, `unshare`, `setns`) at the level of "which pointer in which struct changes, and when" — plus the two ordering rules that make `unshare --pid` and `unshare --mount` behave surprisingly.
- The `/proc/<pid>/ns/*` inode as the *identity* of a namespace, and how to use inode equality as a proof of sharing.
- **GPU-fleet-specific:** device visibility is not namespace-virtualized. There is no "GPU namespace." The actual mechanism is a device-node bind mount plus a **BPF cgroup-device program** — and the cgroup v2 device controller has no interface files at all, which surprises people who go looking for `devices.list`.
- The user-namespace privilege-escalation surface as a concrete, dated mechanism (`unshare(CLONE_NEWNS|CLONE_NEWUSER)` → `CAP_SYS_ADMIN` in your own namespace → reach kernel code that assumed only root gets there), and the three *different* knobs distributions use to shut it off.
- Mount propagation (`shared`/`slave`/`private`/`unbindable`) as the actual mechanism behind the leaked-hostPath bug, read directly out of `/proc/<pid>/mountinfo`.

## Core concepts

### 1. Where namespaces live in the kernel

A `task_struct` does not contain namespaces. It contains **two pointers to pointer tables**:

```
  struct task_struct
        │
        ├── nsproxy ──▶ struct nsproxy  (REFCOUNTED, SHARED between tasks)
        │                  ├── uts_ns             ─▶ hostname, NIS domain
        │                  ├── ipc_ns             ─▶ SysV IPC ids, POSIX mqueues
        │                  ├── mnt_ns             ─▶ the mount table
        │                  ├── pid_ns_for_children ─▶ pid ns for FUTURE children
        │                  ├── net_ns             ─▶ netdevs, routes, nftables,
        │                  │                          sockets, port space
        │                  ├── time_ns            ─▶ CLOCK_MONOTONIC/BOOTTIME offsets
        │                  ├── time_ns_for_children
        │                  └── cgroup_ns          ─▶ cgroup-path root (cosmetic)
        │
        ├── thread_pid ──▶ struct pid ──▶ numbers[] : one (nr, ns) pair per
        │                                 PID-namespace LEVEL this task is in
        │
        └── cred ──▶ struct cred
                        └── user_ns ──▶ struct user_ns  ─▶ uid/gid maps,
                                                           capability sets
```

Three structural facts fall straight out of that picture, and each one answers a question people usually memorise instead of derive:

1. **`nsproxy` is shared and refcounted.** Two tasks in "the same namespaces" literally point at the same `nsproxy`. `fork()` without `CLONE_NEW*` flags just bumps a refcount. Creating a namespace means allocating a *new* `nsproxy` with one field swapped and the rest copied — which is why namespaces are independently composable at zero conceptual cost.
2. **The user namespace is not in `nsproxy`.** It hangs off `cred`, because it is a *credential* property: it determines what your UIDs mean and which capabilities you hold. This is why `setns()` into a user namespace has different, stricter rules than the others, and why `unshare(CLONE_NEWUSER)` immediately changes your capability set.
3. **PID is `pid_ns_for_children`, not `pid_ns`.** A task's own PID namespace **never changes for its lifetime**. `unshare(CLONE_NEWPID)` does not move the caller — it sets the namespace that the caller's *next* child will be born into. That single fact explains why `unshare --pid` without `--fork` gives you a shell that cannot fork usefully, and why `/proc/<pid>/ns/` has both `pid` and `pid_for_children` entries.

### 2. The eight namespace types

Current kernels have exactly eight. Here they are with the flag values straight out of `include/uapi/linux/sched.h`:

| Namespace | Flag | Value | Isolates | Since |
|---|---|---|---|---|
| **Mount** | `CLONE_NEWNS` | `0x00020000` | The mount table (`/proc/<pid>/mountinfo`) | 2.4.19 |
| **UTS** | `CLONE_NEWUTS` | `0x04000000` | `struct utsname`: hostname + NIS domain | 2.6.19 |
| **IPC** | `CLONE_NEWIPC` | `0x08000000` | System V IPC objects, POSIX message queues | 2.6.19 |
| **PID** | `CLONE_NEWPID` | `0x20000000` | The process-ID number space | 2.6.24 |
| **Network** | `CLONE_NEWNET` | `0x40000000` | Interfaces, addresses, routes, nftables, sockets, ports | 2.6.29 |
| **User** | `CLONE_NEWUSER` | `0x10000000` | UID/GID number space and capability sets | 3.8 |
| **Cgroup** | `CLONE_NEWCGROUP` | `0x02000000` | The cgroup *root directory* as reported in `/proc` | 4.6 |
| **Time** | `CLONE_NEWTIME` | `0x00000080` | `CLOCK_MONOTONIC` and `CLOCK_BOOTTIME` offsets | 5.6 |

The mount namespace's flag is `CLONE_NEWNS` and not `CLONE_NEWMNT` because it was the first one; there was no need to say which "NS" you meant. `CLONE_NEWTIME`'s tiny value (`0x80`) collides with legacy `clone(2)`'s exit-signal field, so it is only usable via `clone3(2)` or `unshare(2)`.

Now the substance of each — what it actually virtualizes, and the operational consequence.

**PID (`CLONE_NEWPID`).** Hierarchical. The first process in a new PID namespace is **PID 1** for that namespace and simultaneously has some other, larger PID in every ancestor namespace — a single task genuinely has multiple PIDs, one per level, stored in `struct pid`'s `numbers[]` array. You can see this in `/proc/<pid>/status`:

```
$ grep NSpid /proc/88231/status
NSpid:  88231   1
        │       └── PID 1 inside its own namespace
        └────────── PID 88231 as seen from the host
```

The kernel limits nesting depth to **32 levels**. The parent namespace can see and signal children's processes (under translated PIDs); the reverse is impossible, because a child namespace has no name for anything above it.

PID 1 gets three special kernel rules, and these are the ones that produce production bugs:

- **Signals without an installed handler are not delivered to PID 1** — not even from a privileged process, and not even from an ancestor namespace. This is deliberate protection against accidentally killing a namespace's init. `SIGKILL` and `SIGSTOP` from an *ancestor* namespace are the exception: those are forcibly delivered. **This is the mechanism behind "my container ignores `docker stop`."** The runtime sends `SIGTERM` to PID 1, which is your `python train.py` with no `SIGTERM` handler; the kernel drops it; the runtime waits out the grace period (Kubernetes default `terminationGracePeriodSeconds: 30`) and then sends `SIGKILL`, which cannot be ignored. The fix is a real init as PID 1 (`tini`, `docker run --init`, or handling the signal in your app) — not a longer timeout.
- **PID 1 is the reaper of last resort.** Any process in the namespace that is orphaned gets reparented to PID 1, which must `wait()` for it or leave a zombie. A shell or application that never reaps accumulates `Z` entries that hold PIDs forever.
- **When PID 1 exits, the kernel `SIGKILL`s every process in the namespace.** Afterwards, `fork()` into that namespace fails with `ENOMEM`. The namespace is dead even if something is still holding it open.

**Network (`CLONE_NEWNET`).** A complete, independent network stack: interfaces, addresses, routing tables, nftables/iptables rule sets, `/proc/net`, socket tables, and **the port number space**. A fresh net namespace contains exactly one interface — a `lo` that is administratively **down**. Everything a CNI plugin does is: create a veth pair, move one end into the target namespace, address it, and add routes. Two containers sharing a net namespace share one IP and one port space, which is why two containers in a Kubernetes pod cannot both bind `:8080` and *can* reach each other over `127.0.0.1`.

**Mount (`CLONE_NEWNS`).** A private copy of the mount table. This plus `pivot_root(2)` is what gives a container its own filesystem view. Copying the table does not sever it: **mount propagation** decides whether later mount/unmount events cross the boundary, and that is a per-mount property with four possible values (covered in §6 below).

**UTS (`CLONE_NEWUTS`).** Isolates exactly `struct utsname` — the hostname and the NIS/YP domain name. Nothing else. This, and only this, is why `hostname` inside a container differs from the host. It does *not* isolate the kernel release string (`uname -r`) in any useful way; the container still reports the host kernel, which is the whole point of containers versus VMs.

**IPC (`CLONE_NEWIPC`).** System V IPC objects (`shmget`/`semget`/`msgget` — the things `ipcs` lists) and POSIX message queues (the `mqueue` filesystem). **Not `/dev/shm`.** `/dev/shm` is a tmpfs *mount*, so it is isolated by the **mount** namespace, and its size is governed by that mount's `size=` option. This matters directly on GPU nodes: PyTorch `DataLoader` with `num_workers > 0` passes tensors between workers through `/dev/shm`, and the default 64 MiB shm size in many container images produces `Bus error` or `DataLoader worker killed` — which you fix with `--shm-size` (Docker) or an `emptyDir: {medium: Memory}` volume at `/dev/shm` (Kubernetes), *not* by touching anything IPC-namespace-shaped.

**User (`CLONE_NEWUSER`).** Hierarchical. Maps a range of UIDs/GIDs inside the namespace onto a range outside it, via `/proc/<pid>/uid_map` and `/proc/<pid>/gid_map`. Each line is three whitespace-separated numbers:

```
$ cat /proc/$$/uid_map
         0      65536      65536
         │          │          └── length of the mapped range
         │          └───────────── first UID in the PARENT namespace
         └──────────────────────── first UID in THIS namespace
```

Read it as: "UID 0 in here is UID 65536 out there, for 65536 consecutive IDs." Both map files are **write-once** and start empty. A process that is UID 0 inside such a namespace holds a full capability set *with respect to objects owned by that namespace* and none at all outside it. That is the entire basis of rootless containers — and, as §7 explains, of a recurring class of local privilege-escalation exploits.

**Cgroup (`CLONE_NEWCGROUP`).** Virtualizes the *cgroup root directory* as reported in `/proc/<pid>/cgroup` and `/proc/<pid>/mountinfo`. When you create one, your current cgroup directory becomes `/` for the purposes of those files. It is **path cosmetics and information hiding**, not resource control: your limits are unchanged, your parent can still see and change them, and nothing about `cpu.max` or `memory.max` moves. This is the single most commonly confused namespace, because the word "cgroup" appears in it. [Lesson 03](03-cgroups-v2-and-k8s-enforcement.md) is where actual resource control lives.

```
  Host view of the same process:
    $ cat /proc/88231/cgroup
    0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod3f2a.slice/cri-containerd-9c1d.scope

  Inside the container's cgroup namespace:
    # cat /proc/self/cgroup
    0::/
                 ↑ same cgroup, same limits, different string
```

**Time (`CLONE_NEWTIME`).** Per-namespace *offsets* applied to `CLOCK_MONOTONIC` and `CLOCK_BOOTTIME` — written to `/proc/<pid>/timens_offsets`. It does not virtualize `CLOCK_REALTIME` (wall-clock time is global). It exists chiefly for checkpoint/restore (CRIU), so a restored process sees a monotonic clock that did not jump backwards. Container runtimes generally do not use it.

**The composability rule.** Namespaces are orthogonal and independently combinable. A process can have a private net namespace and the host's PID namespace, or a private mount namespace and the host's everything-else. "Container" is one common bundle; the kernel imposes no all-or-nothing. Kubernetes exposes exactly this in the pod spec: `hostNetwork: true`, `hostPID: true`, `hostIPC: true`, `hostUsers: false`, `shareProcessNamespace: true` are five independent switches over five independent namespaces.

### 3. The three syscalls

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │  clone(2) / clone3(2)  — CREATE a process INTO new namespaces       │
  │     new task_struct, new nsproxy with the flagged fields replaced   │
  │     ▸ this is what runc/crun call                                   │
  │     ▸ clone3 also has CLONE_INTO_CGROUP — born directly into a      │
  │       target cgroup, no post-hoc write to cgroup.procs (→ L3)       │
  ├─────────────────────────────────────────────────────────────────────┤
  │  unshare(2)            — MOVE MYSELF into new namespaces            │
  │     no fork; my nsproxy is replaced in place                        │
  │     ▸ EXCEPT CLONE_NEWPID, which only sets pid_ns_for_children      │
  │     ▸ CLONE_NEWUSER is applied FIRST, so one unshare() call can     │
  │       gain the privilege it needs to create the others              │
  ├─────────────────────────────────────────────────────────────────────┤
  │  setns(2)              — JOIN an EXISTING namespace                 │
  │     takes an fd on /proc/<pid>/ns/<type> (or a bind mount of it)    │
  │     creates NOTHING; just repoints my nsproxy field                 │
  │     ▸ this is what nsenter(1), docker exec, kubectl exec do         │
  └─────────────────────────────────────────────────────────────────────┘
```

Creating any namespace requires `CAP_SYS_ADMIN` **in the user namespace that owns it** — with one exception: **since Linux 3.8, creating a user namespace requires no privilege at all.** That exception is the feature that makes rootless containers possible and, as §7 covers, the one that keeps producing CVEs.

Two ordering rules are worth committing to memory because they explain otherwise baffling behaviour:

- **`unshare(CLONE_NEWUSER | CLONE_NEWNET)` works unprivileged; `unshare(CLONE_NEWNET)` alone does not.** The kernel applies the user namespace first, so by the time it tries to create the net namespace, you already hold `CAP_SYS_ADMIN` in the new user namespace. This is precisely the primitive that turns "unprivileged user" into "root over a fresh copy of half the kernel's subsystems."
- **`unshare(CLONE_NEWPID)` does not move you.** It sets `pid_ns_for_children`. `unshare(1)` papers over this with `--fork`, which forks a child into the new namespace so that the child is PID 1. Without `--fork` your shell stays in the old PID namespace and its children land in a namespace whose init is... nothing, so the first child to exit takes the namespace down with it.

**`setns()` is the whole answer to "what does `kubectl exec` do."** It opens `/proc/<pid>/ns/{mnt,net,uts,ipc,pid}` for a process already in the container, calls `setns()` on each fd, forks (needed for the PID namespace to take effect on the child), and `execve()`s your command. Nothing is created. No container is started. No image is pulled. **A `kubectl exec` that "works" proves the namespaces are intact; it proves nothing about the application.** That distinction matters when you are debugging a wedged pod: `exec` succeeding tells you the runtime and the kernel are fine and the problem is above them.

### 4. Namespace identity: the `/proc/<pid>/ns/` inode

Every namespace a process belongs to appears as a magic symlink:

```
$ ls -l /proc/self/ns/
lrwxrwxrwx 1 root root 0 Aug 17 10:04 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Aug 17 10:04 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Aug 17 10:04 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Aug 17 10:04 net -> 'net:[4026531969]'
lrwxrwxrwx 1 root root 0 Aug 17 10:04 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Aug 17 10:04 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Aug 17 10:04 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Aug 17 10:04 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Aug 17 10:04 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Aug 17 10:04 uts -> 'uts:[4026531838]'
```

The number in brackets is the **namespace inode number**, and it is the namespace's identity. Two processes are in the same namespace **if and only if** the `st_dev`/`st_ino` pair of these links matches. That is not a heuristic; it is the documented comparison in `namespaces(7)`. It is also the cheapest possible proof of pod-level sharing:

```
$ readlink /proc/88231/ns/net /proc/88245/ns/net
net:[4026533012]
net:[4026533012]        # identical → same net namespace → same pod
```

The initial (host) namespaces get inode numbers from a small fixed reserved range around `4026531834`–`4026531969` (0xF0000000 and its neighbours), which is why every Linux box you have ever logged into shows `4026531xxx` for `/proc/1/ns/*`. **Anything not in that range is a namespace someone created.** That single observation lets you classify a container's namespace set at a glance.

These links are also *handles*. Opening one, or bind-mounting it somewhere, **pins the namespace alive even after every process in it exits** — which is exactly how `ip netns add` persists a network namespace with no processes in it (`/run/netns/<name>` is a bind mount of an `ns/net` link), and also how you leak namespaces: a forgotten open fd on `/proc/<dead-pid>/ns/net` keeps an entire network stack allocated. `lsns` will show it with `NPROCS 0`.

`lsns(8)` tabulates all of this:

```
$ lsns -t net
        NS TYPE NPROCS   PID USER  NETNSID NSFS                COMMAND
4026531840 net     412     1 root  unassigned                  /sbin/init
4026533012 net       7 88231 root         0 /run/netns/cni-3f2a /pause
4026533188 net       7 90114 root         1 /run/netns/cni-8b71 /pause
```

Read it column by column: `NS` is the inode; `NPROCS` is how many processes share it; `PID` is the lowest-PID member; `NSFS` is where (if anywhere) the namespace is bind-mounted to keep it pinned. A pod's net namespace shows `NPROCS 7` with `COMMAND /pause` — one namespace, seven processes, created and held open by the pause container.

### 5. What a container actually is, step by step

A container is **namespaces + a cgroup + a rootfs**. Nothing else. Here is what `runc` does between "containerd asked for a container" and "your entrypoint is running," in order:

```
  runc create/start — the actual sequence
  ═══════════════════════════════════════

  t0  runc reads config.json (the OCI runtime spec)
      │
  t1  clone3(CLONE_NEWNS|CLONE_NEWUTS|CLONE_NEWIPC|CLONE_NEWPID|CLONE_NEWNET
             [|CLONE_NEWUSER][|CLONE_NEWCGROUP], CLONE_INTO_CGROUP=<fd>)
      │      └── child is born into fresh namespaces AND into its cgroup
      │          in one syscall (clone3); older paths write cgroup.procs after
      ▼
  t2  [child] write uid_map/gid_map if a user namespace was created
      │        (write-once; must be done before the child drops privilege)
      ▼
  t3  [child] mount the rootfs; make the whole tree MS_PRIVATE/MS_SLAVE so
      │        our mounts do not escape (this is the propagation reset — §6)
      ▼
  t4  [child] bind-mount devices, volumes, /proc, /sys, /dev/shm (tmpfs),
      │        the GPU device nodes and driver libraries (§8)
      ▼
  t5  [child] pivot_root(new_root, put_old); umount the old root
      │        ← THIS is the rootfs half of "container"; there is no
      │          namespace involved, just a mount-namespace-local root swap
      ▼
  t6  [child] set hostname (UTS ns), sysctls, rlimits
      ▼
  t7  [child] attach a BPF_PROG_TYPE_CGROUP_DEVICE program → device allowlist
      │        apply seccomp filter, AppArmor/SELinux label
      ▼
  t8  [child] drop capabilities to the configured bounding/effective set
      ▼
  t9  [child] execve(entrypoint)          ← PID 1 in its namespace
```

Note what is *not* in that list: no hypervisor, no VM exit, no special CPU mode, no kernel object named "container." Everything Docker, containerd, and Kubernetes add above this — image distribution, CNI, lifecycle, scheduling — is orchestration around those nine steps.

Note also step t5. **`pivot_root` is not a namespace.** The rootfs is the third leg of the container definition precisely because no namespace provides it; the mount namespace gives you a private *table*, and `pivot_root` changes what `/` resolves to within that table.

### 6. Mount propagation, and the leaked-hostPath bug

Mount namespaces are not fully independent by default, and this is the source of a genuinely common production incident.

When a mount namespace is created, the new table is a **copy** of the old one — and every mount in it carries a **propagation type** that says whether later mount/unmount events at that point should be mirrored elsewhere. There are four:

| Type | `mount` flag | Behaviour | `mountinfo` optional field |
|---|---|---|---|
| **shared** | `MS_SHARED` | Events propagate **both ways** within a peer group | `shared:N` |
| **slave** | `MS_SLAVE` | Receives events from its master; propagates none back | `master:N` |
| **private** | `MS_PRIVATE` | No propagation in either direction | *(none)* |
| **unbindable** | `MS_UNBINDABLE` | Private, and cannot be bind-mounted at all | `unbindable` |

You read the current state straight out of `/proc/<pid>/mountinfo`, which is the authoritative source and is worth learning to parse:

```
$ cat /proc/self/mountinfo | head -3
36 35 98:0 /mnt1 /mnt2 rw,noatime shared:1 - ext3 /dev/root rw,errors=continue
│  │  │    │     │     │          │        │ │    │         └── super options
│  │  │    │     │     │          │        │ │    └───────────── mount source
│  │  │    │     │     │          │        │ └────────────────── fs type
│  │  │    │     │     │          │        └──────────────────── separator "-"
│  │  │    │     │     │          └───────────────────────────── OPTIONAL FIELDS
│  │  │    │     │     └──────────────────────────────────────── per-mount options
│  │  │    │     └────────────────────────────────────────────── mount point
│  │  │    └──────────────────────────────────────────────────── root within the fs
│  │  └───────────────────────────────────────────────────────── major:minor
│  └──────────────────────────────────────────────────────────── parent mount ID
└─────────────────────────────────────────────────────────────── mount ID
```

The optional fields between the per-mount options and the `-` separator are the propagation state: `shared:N` (member of peer group N), `master:N` (slave of peer group N), `propagate_from:N`, or `unbindable`. **A mount with no optional fields is private.** That is your one-line diagnostic.

Why does this exist at all? Because you want the host's new mounts — a freshly attached USB stick, a newly mapped CSI volume — to appear inside containers that asked for them, without giving containers the power to mount things onto the host. That is exactly `slave`: downward-only propagation. And systemd makes `/` shared at boot on most distributions, which is what makes the whole scheme work by default and also what makes the bug possible.

Kubernetes exposes three of the four modes on a `volumeMounts` entry, and the API type comments give the exact Linux equivalent:

| `mountPropagation:` | Linux equivalent | Meaning |
|---|---|---|
| `None` *(default)* | `rprivate` | Neither direction. The volume is a snapshot of what existed at container start. |
| `HostToContainer` | `rslave` | Host mounts appear inside; container mounts do not escape. |
| `Bidirectional` | `rshared` | Both directions. Mounts made inside the container appear on the host and in every peer. |

**The bug, as a causal timeline:**

```
  t0   Node boot: systemd makes /  MS_SHARED  → peer group 1
       Every mount under it inherits shared:1 unless overridden.

  t1   Pod A mounts hostPath /data at /data with mountPropagation: Bidirectional
       runc marks that mount rshared → it JOINS peer group 1.
       ┌──────────────┐        ┌──────────────┐
       │  host mntns  │◀══════▶│  pod A mntns │     shared:1  (both ways)
       └──────────────┘        └──────────────┘

  t2   A process inside pod A mounts something at /data/scratch
       (a FUSE fs, a loopback image, a tmpfs — anything)
                                    │
                                    ▼ propagates UP to the host
  t3   Host now has /data/scratch mounted.
       ┌──────────────┐
       │  host mntns  │  /data/scratch  ← appeared without anyone asking
       └──────┬───────┘
              ║  because host / is shared:1, it now propagates SIDEWAYS
              ║
       ┌──────▼───────┐  ┌──────────────┐  ┌──────────────┐
       │ pod B mntns  │  │ pod C mntns  │  │ pod D mntns  │
       │ /data/scratch│  │ /data/scratch│  │ /data/scratch│   ← the "leak"
       └──────────────┘  └──────────────┘  └──────────────┘

  t4   Pod A is deleted. The mount is NOT cleaned up — it now belongs to
       the host and to every peer. `umount /data` on the host returns
       EBUSY forever. Node drain hangs. Someone reboots the node.
```

The symptom people report is "why did my hostPath mount leak into every pod," or "why can't I unmount this volume," or "why is node drain hanging." The cause is almost never a Kubernetes bug — it is `Bidirectional` doing exactly what the field says. Some workloads legitimately need it (CSI node drivers must publish mounts *to* the host; that is their entire job), which is why the field exists. Everything else should be `None` or `HostToContainer`.

The diagnostic is one command:

```
$ grep ' /data ' /proc/self/mountinfo
1042 35 259:2 /data /data rw,relatime shared:1 - ext4 /dev/nvme0n1p2 rw
                                      ^^^^^^^^
                     shared:1 → this mount propagates BOTH ways. Found it.
```

### 7. User namespaces: the feature and the attack surface

**The feature.** UID 0 inside a user namespace, mapped to an unprivileged UID outside it, is what makes rootless containers work: the process believes it is root, can `chown` files it owns, can create other namespaces, and can install packages into its own filesystem — while the kernel enforces that all of it applies only to objects the namespace owns.

Kubernetes exposes this as `hostUsers: false` in the pod spec (KEP-127, `UserNamespacesSupport`; alpha in v1.25, beta in v1.35, **stable in v1.36**). The kubelet assigns each pod a **65,536-UID range** — `0-65535` is reserved for host processes and files, and pods get `65536` upward in 65,536-sized blocks — so a container's UID 0 becomes some host UID like 65536, 131072, and so on. To avoid `chown`-ing every file in every volume, the kubelet uses **idmapped mounts**: the mount itself carries the translation, so the same on-disk file appears as UID 0 inside the pod and as UID 65536 on the host, with no data rewritten.

The security payoff is concrete: a container process that escapes into the host's view still holds an unprivileged UID there. Several classes of container escape stop being escapes.

**The attack surface.** Since Linux 3.8, **any unprivileged user can create a user namespace**. Inside it they immediately hold a full capability set — including `CAP_SYS_ADMIN` — with respect to that namespace. The primitive is two lines:

```c
unshare(CLONE_NEWUSER | CLONE_NEWNS);   /* userns applied first → now CAP_SYS_ADMIN */
/* ... now reach kernel code paths that historically assumed
       "the caller has CAP_SYS_ADMIN, therefore the caller is trusted" ... */
```

That assumption was baked into a lot of kernel code written before 2013. **CVE-2022-0185** is the canonical demonstration: an integer underflow in `legacy_parse_param()` in the filesystem-context code (kernel 5.1 through 5.16.1) gave an out-of-bounds heap write. Reaching it required `CAP_SYS_ADMIN` — trivially obtained via `unshare(CLONE_NEWNS|CLONE_NEWUSER)` — and the result was a full container escape to host root. The pattern has recurred often enough that hardened fleets treat unprivileged user namespaces as an attack surface to be closed by default.

**Three different knobs, and getting the wrong one is a common mistake:**

| Knob | Where it exists | Effect |
|---|---|---|
| `user.max_user_namespaces=0` | **Upstream**, `/proc/sys/user/` | Per-user-namespace limit on user-namespace creation. Setting it to 0 in the initial namespace blocks all creation. `clone`/`unshare` fail with `ENOSPC`. |
| `kernel.unprivileged_userns_clone=0` | Debian/older-Ubuntu **patch**, not upstream | Blocks unprivileged creation. Frequently cited; **does not exist on RHEL, Fedora, Amazon Linux, or current Ubuntu.** |
| `kernel.apparmor_restrict_unprivileged_userns=1` | Ubuntu **23.10+**, on by default in 24.04 LTS | Allows creation but denies *capabilities within* the new namespace unless the binary has an AppArmor profile permitting it. |

`/proc/sys/user/` also holds per-type limits for every other namespace (`max_pid_namespaces`, `max_net_namespaces`, `max_mnt_namespaces`, …), defaulting in the initial user namespace to **half of `/proc/sys/kernel/threads-max`**. Exceeding any of them makes `clone(2)`/`unshare(2)` fail with `ENOSPC` — a useful thing to recognise on a node running thousands of short-lived containers.

Note the security trade-off runs both ways, which is what makes it an interesting interview answer rather than a rule: unprivileged user namespaces are simultaneously the **largest** unprivileged kernel attack surface on a modern box *and* the mechanism that makes container escapes far less valuable. Netflix's public position on this ("Evolving Container Security With Linux User Namespaces") is to lean in — run every container in its own user namespace so that an escape lands on an unprivileged UID — rather than to disable the feature. Which choice is right depends entirely on whether your nodes run untrusted *user shells* or only orchestrator-placed containers.

### 8. Device visibility: why there is no "GPU namespace"

It is tempting to assume every resource a container appears to "have" is namespace-virtualized. GPUs are the clearest counterexample: **there is no device namespace.** `/dev/nvidia0` is a host-global character device node. No namespace hides it, duplicates it, or gives a container a private version of it.

What actually controls GPU visibility is two independent mechanisms, and neither is a namespace:

**(a) The device node must exist in the container's mount namespace.** A character device node is just an inode carrying a major:minor pair. The runtime bind-mounts the specific nodes the container is allowed to use into its `/dev`. The NVIDIA container toolkit's device list is, from its source, the per-GPU nodes `/dev/nvidia<N>` plus a fixed set of control nodes:

| Node | Purpose | Major |
|---|---|---|
| `/dev/nvidia0`, `/dev/nvidia1`, … | One per physical GPU | **195** (fixed) |
| `/dev/nvidiactl` | Driver control channel — required for *any* CUDA use | **195** (minor 255) |
| `/dev/nvidia-uvm` | Unified Virtual Memory — required for CUDA managed memory | **dynamic** — read it from `/proc/devices` |
| `/dev/nvidia-uvm-tools` | UVM debug/profiling interface | dynamic |
| `/dev/nvidia-modeset` | Display/modeset (irrelevant on headless compute nodes) | 195 |

`/dev/nvidia-uvm`'s major is allocated dynamically at module load and **can change across reboots** — which is why tooling greps `/proc/devices` for it instead of hardcoding, and why a stale device-node major in a container image or a persisted cgroup rule produces "Failed to initialize NVML: Unknown Error" after a driver reload.

**(b) The cgroup must permit opening it.** This is the part that surprises people, because in cgroup v2 there is nothing to `cat`. The kernel documentation is explicit: *"Cgroup v2 device controller has no interface files and is implemented on top of cgroup BPF."* Access control is a `BPF_PROG_TYPE_CGROUP_DEVICE` program attached to the cgroup with `BPF_CGROUP_DEVICE`. On every attempt to open a device node, the program runs with a `bpf_cgroup_dev_ctx` describing access type (mknod/read/write) and device type/major/minor, and returns 0 to deny (`-EPERM`) or non-zero to allow.

```
  How a container gets a GPU — NO namespace anywhere in this path
  ═══════════════════════════════════════════════════════════════

   NVIDIA device plugin (DaemonSet)
        │  advertises  nvidia.com/gpu: 8   via the kubelet device-plugin API
        ▼
   kube-scheduler places the pod  (it only ever sees an integer count)
        │
        ▼
   kubelet Allocate() → plugin returns: which GPU UUIDs/indices, and
        │               either env vars (NVIDIA_VISIBLE_DEVICES) or a
        │               CDI device name (nvidia.com/gpu=0)
        ▼
   containerd → nvidia-container-runtime → modifies the OCI spec:
        │
        ├─▶ linux.devices[]     : /dev/nvidia0, /dev/nvidiactl,
        │                         /dev/nvidia-uvm, /dev/nvidia-uvm-tools
        │                         → bind-mounted into the MOUNT namespace
        │
        ├─▶ linux.resources.devices[] : allow c 195:0 rw, c 195:255 rw, …
        │                         → compiled into a BPF_PROG_TYPE_CGROUP_DEVICE
        │                            program attached to the container's cgroup
        │
        └─▶ mounts[]            : libcuda.so.<ver>, libnvidia-ml.so.<ver>,
                                  nvidia-smi, and friends from the host driver
        ▼
   runc creates the container.  There is NO ns/device entry, because
   the kernel has no device namespace.  Access = (node present) AND
                                                 (BPF program says yes).
```

**The debugging consequence is the whole point.** "The pod requested a GPU but sees none" is never a namespace bug. The checks, in order, are: did the device plugin actually allocate (`kubectl describe node` → allocatable/allocated `nvidia.com/gpu`)? Are the nodes present in the container's `/dev`? Is the BPF device program allowing that major:minor? Do the driver `.so` files match the host driver version? Someone running `ip netns` or `unshare` at that ticket is in the wrong subsystem entirely.

For completeness, because it comes up: the cgroup **`misc` controller** is *not* the GPU accounting mechanism, despite frequently being described that way. Its registered resource types in the current kernel (`include/linux/misc_cgroup.h`) are exactly AMD SEV/SEV-ES ASIDs and Intel TDX HKIDs — confidential-computing key slots. The newer **`dmem`** controller does account device memory (`dmem.max`, `dmem.current`, keyed like `drm/0000:03:00.0/vram0`), but it is driven by DRM drivers; NVIDIA's proprietary stack does not use it. GPU *count* is accounted by the Kubernetes device-plugin framework in userspace, not by any cgroup controller.

## Perspectives

**Kernel-mechanism view.** Namespaces are orthogonal, refcounted virtualizations of exactly one global resource each, reached through two pointers on `task_struct`: `nsproxy` for seven of them and `cred->user_ns` for the eighth. There is no combined "container" primitive. The trick is entirely in composing eight independent, single-purpose isolations plus a `pivot_root` and a cgroup — nothing more exotic is happening under `docker run`, and you can build the whole thing by hand in one `unshare` invocation.

**Operator/SRE view.** "Which namespace explains this symptom" is a reflex worth building deliberately, and half its value is knowing when the answer is "none of them." A pod that cannot see the GPU is a device-node/BPF-cgroup problem. Two sidecars port-clashing is the shared net namespace working as designed. Zombies piling up is PID-1 reaping semantics. A container ignoring `SIGTERM` is the PID-1 signal rule, not a Docker bug. A hostPath that will not unmount is propagation. Each symptom maps to a specific checkable file — `/proc/<pid>/ns/*`, `/proc/<pid>/mountinfo`, `/proc/<pid>/status`'s `NSpid` — rather than to a guess.

**GPU-fleet-specific view.** Device visibility is where the "no magic namespace for everything" lesson bites hardest. GPUs reach a container via a mount-namespace bind mount plus a BPF cgroup-device allowlist, which is exactly why the NVIDIA device plugin and container toolkit exist as real software with real work to do: computing allocations, injecting nodes, matching driver library versions, and emitting cgroup rules. The kernel gives them no shortcut. A staff engineer debugging "pod requested a GPU but sees none" checks the plugin's allocation and the container's `/dev` and cgroup rules — never `ip netns`.

**Failure-mode / security view.** The same feature — unprivileged user-namespace creation, unprivileged since Linux 3.8 — is simultaneously the enabler of rootless containers and the reason a long line of kernel bugs became local root exploits, because `unshare(CLONE_NEWUSER|CLONE_NEWNS)` hands an ordinary user `CAP_SYS_ADMIN` over a fresh copy of half the kernel's subsystems. There are two coherent responses and one incoherent one. Coherent: close it (`user.max_user_namespaces=0`, or Ubuntu's AppArmor restriction) on nodes that run untrusted local users; or lean in (`hostUsers: false`) so every container's root is an unprivileged host UID. Incoherent: leave it wide open and rely on the container runtime's defaults, which is where most fleets actually are.

**Scaling view.** Namespaces are not free at density. Netflix found that at high container-launch concurrency, container start time was dominated by contention on the kernel's **global mount lock** — containerd was issuing tens of thousands of `mount_setattr()`/`move_mount()` calls to apply idmaps per image layer, all serialising on one lock, and the effect was worse on multi-socket NUMA hardware where the cache-coherence cost of that lock is higher. The fix was structural rather than tuning: use the recursive-bind idmap support added in Linux 6.3 to apply one recursive idmapped bind to the whole tree instead of one per layer.

## Real-world use cases

- **Netflix TechBlog — "Mount Mayhem at Netflix: Scaling Containers on Modern CPUs."** Migrating from Docker to containerd, Netflix hit a wall on container start-up throughput that profiling attributed almost entirely to waiting on the kernel's global mount lock in the VFS. Containerd was performing a `mount_setattr()` (to set the idmap for the pod's UID range) plus a `move_mount()` (to bind the result) **for every image layer**, so a burst of many-layer images produced on the order of 20,000 mount syscalls all contending for the same lock — and the contention was measurably worse on dual-socket instances with mesh-based cache coherence than on single-socket ones. The fix used the recursive-bind idmap support introduced in **Linux 6.3**, letting containerd apply one recursive idmapped bind of the parent directory instead of per-layer mounts. *What it shows:* mount namespaces are not a zero-cost abstraction at fleet density; the bottleneck was a kernel lock, and the fix was a different syscall pattern, not a tuning knob. It also shows why user namespaces at scale (§7) are a systems-engineering project, not a checkbox.
- **CVE-2022-0185 — unprivileged user namespace to container escape.** An integer underflow in `legacy_parse_param()` in the Linux filesystem-context code (5.1 through 5.16.1) permitted an out-of-bounds heap write. Reaching the vulnerable path required `CAP_SYS_ADMIN`, which any unprivileged user obtains instantly via `unshare(CLONE_NEWNS|CLONE_NEWUSER)`; the published exploits went from unprivileged container process to host root. Mitigations offered at the time were: patch, or disable unprivileged user namespaces. *What it shows:* the concrete shape of the user-namespace attack surface — not "user namespaces are risky" in the abstract, but "the unprivileged capability grant lets an attacker reach kernel code that was written assuming its callers were trusted." This is the incident class that drove Ubuntu 23.10/24.04's AppArmor restriction.
- **Julia Evans — "What even is a container: namespaces and cgroups."** A hands-on `unshare` walkthrough that arrives at "container = namespaces + cgroup + rootfs, no magic" by building one interactively rather than asserting it. *What it shows:* the exact thesis of this lesson, demonstrated rather than claimed — a good companion read alongside the worked example below, and the fastest way to make the model stick.

## Worked example

Build a container by hand, then dissect a real one, then prove the GPU claim. *(Transcripts below are representative — reconstructions of the standard output, not a captured session. Inode numbers and PIDs will differ on your machine; the structure will not.)*

**1. Hand-rolled container.** Create new PID, mount, net, and UTS namespaces:

```
$ sudo unshare --pid --fork --mount-proc --net --uts --ipc bash
# hostname handmade                 # UTS ns → independent hostname
# hostname
handmade
# echo $$                           # our shell's PID inside the new pid ns
1                                    # we are PID 1 — the PID namespace at work
# ps -e                             # --mount-proc gave us a fresh /proc view
    PID TTY          TIME CMD
      1 pts/0    00:00:00 bash
     10 pts/0    00:00:00 ps
# ip link                           # new net ns: only loopback, and it's down
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Five kernel resources are now virtualized, and it took one syscall wrapper. Three details are worth pausing on:

- **`--fork` is not optional.** `unshare(CLONE_NEWPID)` sets `pid_ns_for_children`, so the *calling* shell stays in the old PID namespace. `--fork` makes `unshare(1)` fork a child into the new namespace, and that child is PID 1. Drop `--fork` and `echo $$` will show the host PID.
- **`--mount-proc` is not optional either.** `ps` reads `/proc`. Without a fresh `proc` mount in the new mount namespace, you inherit the host's `/proc` and `ps -e` shows every host process — with PIDs that are meaningless in your namespace. This is the clearest possible demonstration that PID namespaces virtualize the *number space*, not the *procfs view*; you need both namespaces cooperating.
- **This shell is still using the host's root filesystem.** It has a private mount *table*, not a private *rootfs*. That is the missing third leg — a real runtime would `pivot_root` into an unpacked image here.

Prove the isolation is real by comparing inodes against the host:

```
# readlink /proc/self/ns/uts /proc/self/ns/net /proc/self/ns/user
uts:[4026532371]         # NOT 4026531838 → new UTS namespace
net:[4026532374]         # NOT 4026531969 → new net namespace
user:[4026531837]        # SAME as host → we did NOT unshare the user ns
```

The third line is the honest part: `sudo unshare` without `--user` runs as real root in the host user namespace. That is why it worked without any uid_map gymnastics, and it is also why this is *not* a rootless container.

**2. Dissect a running container.** In another terminal, find a real container's host PID and read its namespace set:

```
$ CID=$(crictl ps --name my-trainer -q)
$ PID=$(crictl inspect $CID | jq -r '.info.pid')
$ echo $PID
88231
$ ls -l /proc/88231/ns/
lrwxrwxrwx ... cgroup -> 'cgroup:[4026533005]'
lrwxrwxrwx ... ipc    -> 'ipc:[4026533010]'
lrwxrwxrwx ... mnt    -> 'mnt:[4026533009]'
lrwxrwxrwx ... net    -> 'net:[4026533012]'
lrwxrwxrwx ... pid    -> 'pid:[4026533105]'
lrwxrwxrwx ... time   -> 'time:[4026531834]'
lrwxrwxrwx ... user   -> 'user:[4026531837]'
lrwxrwxrwx ... uts    -> 'uts:[4026533011]'
```

Classify each line by comparing against the reserved host range (`4026531834`–`4026531969`):

| Namespace | Inode | Host's | Verdict |
|---|---|---|---|
| cgroup | 4026533005 | 4026531835 | **private** — sees its cgroup as `/` |
| ipc | 4026533010 | 4026531839 | **private** (per-pod, shared with the pause container) |
| mnt | 4026533009 | 4026531840 | **private** — its own rootfs |
| net | 4026533012 | 4026531969 | **private** (per-pod, held by pause) |
| pid | 4026533105 | 4026531836 | **private** — its own PID 1 |
| time | 4026531834 | 4026531834 | **shared with host** — runtimes do not use time ns |
| user | 4026531837 | 4026531837 | **shared with host** — `hostUsers` not set to false |
| uts | 4026533011 | 4026531838 | **private** — its own hostname |

Then check *who else* is in each:

```
$ lsns -p 88231
        NS TYPE   NPROCS   PID USER COMMAND
4026531834 time      412     1 root /sbin/init
4026531837 user      412     1 root /sbin/init
4026533005 cgroup      3 88231 root python train.py
4026533009 mnt         3 88231 root python train.py
4026533010 ipc         7 88231 root /pause
4026533011 uts         7 88231 root /pause
4026533012 net         7 88231 root /pause
4026533105 pid         3 88231 root python train.py
```

**Read the `NPROCS` column — it is the pod topology.** `mnt`, `pid`, and `cgroup` show 3 (this container's own processes). `net`, `ipc`, and `uts` show 7 and are owned by `/pause` — those are pod-wide, shared with the sidecar. `time` and `user` show 412, i.e. the whole node. That table *is* the answer to "what does this pod share," derived from the kernel rather than from the YAML.

**3. Enter it by hand** — this is `kubectl exec`, unwrapped:

```
$ sudo nsenter -t 88231 -m -u -i -n -p -- /bin/sh
# hostname
my-trainer-7c9f4d-x2k1p              # the pod's UTS namespace
# ip -br addr
lo    UNKNOWN  127.0.0.1/8
eth0  UP       10.244.3.17/24        # the pod IP, not the node IP
# ls /
bin  dev  etc  proc  sys  usr  workspace   # the image's rootfs, not the host's
```

You just reproduced `kubectl exec` with `setns()` on five `/proc/<pid>/ns/*` file descriptors. Nothing was created; no container started. Note the flag order matters conceptually: `-m` (mount) is applied such that the subsequent `/bin/sh` resolves inside the container's rootfs — which is why `nsenter` needs `--` and an absolute path that exists *in the image*.

**4. Confirm the "no GPU namespace" claim.** On a GPU node:

```
$ ls -l /proc/88231/ns/ | grep -ci device
0                                    # there is no such namespace. At all.

$ ls -l /proc/88231/root/dev/ | grep nvidia
crw-rw-rw- 1 root root 195,   0 Aug 17 09:12 nvidia0
crw-rw-rw- 1 root root 195, 255 Aug 17 09:12 nvidiactl
crw-rw-rw- 1 root root 508,   0 Aug 17 09:12 nvidia-uvm
crw-rw-rw- 1 root root 508,   1 Aug 17 09:12 nvidia-uvm-tools
                       │     │
                       │     └── minor
                       └──────── major: 195 fixed for nvidia*/nvidiactl;
                                 508 here is the DYNAMIC uvm major

$ grep nvidia /proc/devices
195 nvidia-frontend
508 nvidia-uvm                       # confirms the dynamic allocation
```

Note that the host has eight GPUs but this container's `/dev` contains exactly one `nvidia<N>` node. That is the device plugin's allocation, materialised as a bind mount — no namespace involved.

Now the second half, the cgroup allowlist. In cgroup v2 there is no file to read, so you list the attached BPF programs:

```
$ CG=/sys/fs/cgroup$(cut -d: -f3 /proc/88231/cgroup)
$ ls $CG | grep -i device
                                     # nothing — v2 device control has NO
                                     # interface files, by design

$ sudo bpftool cgroup show $CG
ID       AttachType      AttachFlags     Name
418      cgroup_device   multi
                │
                └── BPF_PROG_TYPE_CGROUP_DEVICE — this program is consulted
                    on every open() of a device node by tasks in this cgroup
```

**Conclusion, stated the way an interview wants it:** *"GPU access is a bind-mounted character device node in the container's mount namespace, plus a BPF cgroup-device program that permits that major:minor. Neither is a namespace mechanism, because the kernel has no device namespace — which is exactly why the NVIDIA device plugin and container toolkit have to do explicit injection."*

## Practice

On a Linux laptop/VM with Docker or containerd. This feeds directly into the module deliverable, [Anatomy of a Container](../practice/anatomy-of-a-container/README.md) — the namespace inventory and hand-entry steps here are the first layer of that teardown.

1. **Build a container by hand.** Run `sudo unshare --pid --fork --mount-proc --net --uts --ipc bash`. Inside: set a hostname, confirm `echo $$` is `1`, run `ps -e` (should show only your shell + `ps`), and `ip link` (only a down `lo`). Then `readlink /proc/self/ns/*` and record which inodes differ from `/proc/1/ns/*`. Note against each observation *which* namespace produced it. Finally, re-run **without** `--fork` and explain what `echo $$` prints and why.

2. **Reconstruct a real container's namespace set.** Start any container (`docker run -d --name probe nginx`). Find its host PID (`docker inspect -f '{{.State.Pid}}' probe`). List its namespaces two ways — `ls -l /proc/<pid>/ns/` and `lsns -p <pid>` — and build the classification table from the worked example: for each of the eight types, record the inode, the host's inode, private-vs-shared, and `NPROCS`. If you have a Kubernetes cluster available, repeat for a real pod with a sidecar and confirm that `net`/`ipc`/`uts` are pod-wide (`NPROCS` covering both containers) while `mnt`/`pid` are per-container.

3. **Enter it.** `sudo nsenter -t <pid> -m -u -i -n -p -- /bin/sh`, then confirm you see the container's hostname, its IP, and its rootfs rather than the host's. Compare against `docker exec probe hostname`. Then do it the manual way for one namespace only — `nsenter -t <pid> -u hostname` — to prove the namespaces really are independent.

4. **Reproduce the mount-propagation leak.** This is the highest-value exercise here.
   ```bash
   sudo mkdir -p /tmp/shared
   sudo mount --bind /tmp/shared /tmp/shared
   sudo mount --make-shared /tmp/shared
   grep ' /tmp/shared ' /proc/self/mountinfo        # expect: shared:N

   # Bidirectional (rshared):
   docker run -d --name leaky \
     --mount type=bind,source=/tmp/shared,target=/mnt,bind-propagation=rshared \
     alpine sleep 3600
   docker exec leaky sh -c 'mkdir -p /mnt/inner && mount -t tmpfs none /mnt/inner'
   grep ' /tmp/shared/inner ' /proc/self/mountinfo   # ← it appeared on the HOST

   # Now repeat with the default (rprivate) and observe it does NOT appear.
   docker rm -f leaky
   ```
   Record both `mountinfo` lines, with the optional propagation fields highlighted. Then try `sudo umount /tmp/shared` while the leaked inner mount exists and note the error.

5. **Prove the GPU claim (or its CPU-only equivalent).** On a GPU node, run the four commands from Worked example §4 and record that there is no `ns/device` entry, which `nvidia*` nodes are present in the container's `/dev` versus the host's, and the major numbers (checking `/proc/devices` for the dynamic `nvidia-uvm` major). Without a GPU, do the same with any device: `docker run --rm --device /dev/urandom alpine ls -l /dev/urandom`, then show that a container *without* `--device` has no such node, and that the difference is a bind mount plus a cgroup rule, not a namespace.

**Acceptance:** (i) the eight-row classification table for one real container, with inode numbers, host inodes, private/shared verdict, and `NPROCS`; (ii) evidence you entered it by hand via `nsenter` — output that visibly differs from the host (hostname, `ip -br addr`, or `ls /`); (iii) the two `mountinfo` lines from the propagation experiment, showing `shared:N` in the leaking case and no optional field in the private case, plus the `umount` error; (iv) a 4–6 sentence "a container is just…" paragraph in your own words naming namespaces + cgroup + rootfs, stating what `pivot_root` contributes that no namespace does, and explicitly noting that device visibility (e.g. GPUs) is a bind mount plus a BPF cgroup-device program rather than part of the namespace set. Clean up: `docker rm -f probe leaky; sudo umount /tmp/shared`.

## Common pitfalls

1. **Believing there is a device or GPU namespace.** There is not. Device access is (a) a device node present in the container's *mount* namespace and (b) a `BPF_PROG_TYPE_CGROUP_DEVICE` program attached to its cgroup. *Mechanism:* cgroup v2's device controller has no interface files at all — `ls` on the cgroup directory shows nothing device-related — so people go looking for `devices.list`, fail to find it, and conclude the mechanism must be elsewhere. Use `bpftool cgroup show <path>`.
2. **Thinking the `misc` cgroup controller accounts GPUs.** It does not. Its registered resources in the current kernel are AMD SEV/SEV-ES ASIDs and Intel TDX HKIDs. The `dmem` controller does account device memory but is driven by DRM drivers. GPU *count* is accounted by the Kubernetes device-plugin framework in userspace.
3. **Assuming `docker exec`/`kubectl exec` starts something.** It is `setns()` on existing `/proc/<pid>/ns/*` file descriptors, a fork (required for the PID namespace to apply), and `execve()`. Nothing is created and nothing restarts. *Consequence:* a successful `exec` proves the namespaces and runtime are healthy and says nothing about your application's health.
4. **Forgetting PID-1 signal semantics and blaming the grace period.** The kernel refuses to deliver a signal to PID 1 unless PID 1 installed a handler for it — deliberately, to protect the namespace's init. So `SIGTERM` to a bare `python train.py` is dropped, the runtime waits the full `terminationGracePeriodSeconds`, and then `SIGKILL` (which cannot be blocked) ends it. *Fix:* a real init (`tini`, `--init`) or a signal handler in the app. Raising the timeout makes shutdown slower, not cleaner.
5. **Conflating IPC namespace with `/dev/shm`.** The IPC namespace covers System V IPC and POSIX message queues. `/dev/shm` is a tmpfs *mount*, isolated by the **mount** namespace and sized by that mount's options. This is why PyTorch `DataLoader` shared-memory failures are fixed with `--shm-size` or an `emptyDir: {medium: Memory}` at `/dev/shm`, and never by anything IPC-shaped.
6. **Treating "shares the pod's network namespace" as "shares everything."** Within a pod, `net`, `ipc`, and `uts` are typically pod-wide (held open by the pause container) while `mnt` and `pid` stay per-container unless `shareProcessNamespace: true` is set. Read `NPROCS` in `lsns -p <pid>` rather than assuming.
7. **Citing `kernel.unprivileged_userns_clone=0` as *the* way to disable user namespaces.** That sysctl is a Debian/older-Ubuntu patch and does not exist upstream, on RHEL/Fedora/Amazon Linux, or on current Ubuntu. The upstream lever is `user.max_user_namespaces=0` in `/proc/sys/user/`; Ubuntu 23.10+ uses `kernel.apparmor_restrict_unprivileged_userns=1` instead, which permits creation but denies capabilities inside. Check which one your distro actually has before you write it into a hardening baseline.
8. **Assuming namespace operations are cheap at density.** As the Netflix case shows, per-layer `mount_setattr()`/`move_mount()` calls serialise on a global kernel mount lock, and at burst scale that lock — not CPU, not disk — becomes the limit on container start-up throughput, with a worse profile on multi-socket NUMA hardware. This is a scale property of the *mechanism*, independent of any per-container resource limit.

## Self-check

- **What does being PID 1 in a PID namespace mean for signal handling and zombie reaping?**
  **Answer:** Three kernel rules apply. (1) **Signals without an installed handler are not delivered to PID 1**, even from privileged processes and even from an ancestor namespace — this is deliberate protection against accidentally killing a namespace's init. The exceptions are `SIGKILL` and `SIGSTOP` sent *from an ancestor namespace*, which are forcibly delivered and cannot be caught. This is the mechanism behind "my container ignores `docker stop`": the runtime's `SIGTERM` reaches a process with no handler and is dropped, so the runtime waits the full grace period and then sends `SIGKILL`. (2) **PID 1 is the reaper of last resort** — any orphaned process in the namespace is reparented to it and must be `wait()`ed for, or it stays a zombie holding a PID. (3) **When PID 1 exits, the kernel `SIGKILL`s every process in the namespace**, and subsequent `fork()` into it fails with `ENOMEM`. The fix for (1) and (2) is a real init as PID 1 (`tini`, `docker run --init`) that forwards signals and reaps children.

- **Which namespace makes `hostname` independent, and which makes the process tree independent — and what does each *not* cover?**
  **Answer:** The **UTS** namespace (`CLONE_NEWUTS`, `0x04000000`) isolates `struct utsname` — the hostname and the NIS/YP domain name, and nothing else. It does not give you a different kernel version; `uname -r` still reports the host kernel, which is the defining difference between a container and a VM. The **PID** namespace (`CLONE_NEWPID`, `0x20000000`) isolates the process-ID *number space*: the first process becomes PID 1, and a single task has one PID per namespace level (visible as multiple values in `NSpid:` in `/proc/<pid>/status`). It does **not** by itself change what `/proc` shows — you also need a mount namespace with a fresh `proc` mounted, which is why `unshare --pid --fork` without `--mount-proc` still lists every host process. They are independent: private hostname without private process tree is a valid configuration, and vice versa.

- **How do two containers in one Kubernetes pod share a network namespace, and what stays separate?**
  **Answer:** All containers in a pod are `setns()`-joined into namespaces held open by the pod's `pause` (infra) container: the **network** namespace, and normally **IPC** and **UTS** too. Sharing net means one `eth0`, one pod IP, one routing table, one nftables ruleset, and **one port number space** — which is why two containers in a pod cannot both bind `:8080` and why they can reach each other on `localhost`. What stays separate by default is the **mount** namespace (each container has its own rootfs and volume view) and the **PID** namespace (separate process trees; `shareProcessNamespace: true` merges them, at which point no container's entrypoint is PID 1 any more). The **user** namespace is shared with the host unless `hostUsers: false`. You verify all of this without reading any YAML: `lsns -p <pid>` and read the `NPROCS` column — namespaces showing a count covering all the pod's processes are pod-wide, ones showing only this container's count are per-container.

- **A pod's container reports it "can't see the GPU." Which namespace is misconfigured?**
  **Answer:** Trick question — none, and treating it as a namespace problem costs you an hour in the wrong subsystem. There is no device namespace in Linux; `/proc/<pid>/ns/` has no `device` entry. GPU access requires two independent things. (1) The **device nodes must exist in the container's mount namespace**: `/dev/nvidia<N>` for the allocated GPU plus the control nodes `/dev/nvidiactl` (major 195, minor 255) and `/dev/nvidia-uvm` / `/dev/nvidia-uvm-tools` (dynamically allocated major — check `/proc/devices`, it can change across driver reloads). (2) The **cgroup must permit opening them**: in cgroup v2 this is a `BPF_PROG_TYPE_CGROUP_DEVICE` program attached with `BPF_CGROUP_DEVICE`, which has no interface files, so you inspect it with `bpftool cgroup show`, not `cat`. Then check the device plugin actually allocated (`kubectl describe node`) and that the injected driver libraries match the host driver version. The `misc` cgroup controller is not involved — it registers only SEV/SEV-ES ASIDs and TDX HKIDs.

- **Why do some hardened fleet nodes disable unprivileged user-namespace creation, and what exactly do they set?**
  **Answer:** Since Linux 3.8, creating a user namespace requires no privilege, and inside it the creator immediately holds a full capability set including `CAP_SYS_ADMIN` over that namespace's objects. Because `unshare(2)` applies `CLONE_NEWUSER` *first*, a single call like `unshare(CLONE_NEWUSER|CLONE_NEWNS)` takes an unprivileged user to `CAP_SYS_ADMIN` and thereby into kernel code paths written on the assumption that their callers were trusted. **CVE-2022-0185** is the canonical case: an integer underflow in `legacy_parse_param()` (kernels 5.1–5.16.1) reachable only with `CAP_SYS_ADMIN`, exploited from an unprivileged container to host root. Three different knobs exist and they are not interchangeable: **`user.max_user_namespaces=0`** (upstream, `/proc/sys/user/`, makes creation fail with `ENOSPC`); **`kernel.unprivileged_userns_clone=0`** (a Debian/older-Ubuntu patch that does not exist upstream or on RHEL-family systems, despite being the most-cited one); and **`kernel.apparmor_restrict_unprivileged_userns=1`** (Ubuntu 23.10+, default-on in 24.04, which allows creation but denies capabilities inside without an AppArmor profile). The opposite strategy is equally defensible: run every pod with `hostUsers: false` (Kubernetes KEP-127, stable in v1.36) so container root maps to an unprivileged host UID via idmapped mounts and an escape lands on nothing.

- **A `hostPath` volume "leaked" into every pod on the node and now will not unmount. Explain the mechanism and the one command that proves it.**
  **Answer:** Mount propagation. Systemd makes `/` `MS_SHARED` at boot on most distributions, so mounts under it join a shared peer group. A volume mounted with `mountPropagation: Bidirectional` becomes `rshared` and joins that peer group, so any mount made *inside* the container at that path propagates **up** to the host, and from the host **sideways** into every other namespace in the peer group. The mount now belongs to the host, so deleting the pod does not remove it, `umount` returns `EBUSY`, and node drain hangs. The proof is a single grep of `/proc/self/mountinfo`: the optional fields between the per-mount options and the `-` separator show `shared:N` (bidirectional peer group), `master:N` (slave — receives only), or nothing at all (private). The Kubernetes values map exactly: `None` → `rprivate`, `HostToContainer` → `rslave`, `Bidirectional` → `rshared`. `Bidirectional` is legitimate for CSI node drivers, whose job is to publish mounts to the host; almost nothing else needs it.

## Connections & what's next

This lesson is the "views" half of the container thesis. [03 — cgroups v2 and K8s resource enforcement](03-cgroups-v2-and-k8s-enforcement.md) — the anchor lesson of this module — adds the "limits" half: the same process, now also bounded by `cpu.max`/`memory.max`, and the BPF device program introduced here for GPU visibility placed in its proper home. The CGROUP namespace covered here (cosmetic path virtualization — a container seeing `0::/`) is easy to confuse with cgroup *resource control*; lesson 03 draws that line precisely and shows the host-side path the container is hiding.

Two other threads run forward from here. [08 — eBPF](08-ebpf.md) revisits namespaces from the observability side — how Cilium and Hubble attach to and correlate across network namespaces, and how the same BPF cgroup-attach mechanism used for device control is used for policy and tracing. [07 — networking datapath & conntrack](07-networking-datapath-conntrack.md) picks up the network namespace as a datapath object: veth pairs, per-namespace conntrack tables, and what happens when a pod's egress blows the node's conntrack limit.

The immediate next step is **[03 — cgroups v2 and K8s resource enforcement](03-cgroups-v2-and-k8s-enforcement.md)**: same processes, same namespaces — now with hard resource limits attached, and with every Kubernetes YAML field mapped to the exact kernel file it writes.

## References & further reading

**Primary sources**

- **namespaces(7)** — https://man7.org/linux/man-pages/man7/namespaces.7.html — the authoritative per-namespace overview: all eight types with their `CLONE_NEW*` flags, the `/proc/<pid>/ns/` interface and inode-equality rule, the `clone`/`unshare`/`setns` summary, the `/proc/sys/user/max_*_namespaces` limits, and the namespace-lifetime rules (why an open fd or bind mount pins a namespace alive).
- **pid_namespaces(7)** — https://man7.org/linux/man-pages/man7/pid_namespaces.7.html — source for the PID-1 signal rules (no delivery without an installed handler; `SIGKILL`/`SIGSTOP` forced from an ancestor), the reaping semantics, the "init exits → everything dies, then `ENOMEM`" behaviour, and the 32-level nesting limit.
- **mount_namespaces(7)** and **`Documentation/filesystems/sharedsubtree.rst`** — https://man7.org/linux/man-pages/man7/mount_namespaces.7.html · https://docs.kernel.org/filesystems/sharedsubtree.html — the four propagation types, the `mount --make-{shared,slave,private,unbindable}` commands, and the full state-transition tables for bind/move operations between differently-propagating mounts.
- **user_namespaces(7)** — https://man7.org/linux/man-pages/man7/user_namespaces.7.html — the `uid_map`/`gid_map` three-field line format and write-once rule, the capability semantics inside a namespace, and the "no privilege required since Linux 3.8" statement that underpins both rootless containers and the attack surface.
- **cgroup_namespaces(7)** — https://man7.org/linux/man-pages/man7/cgroup_namespaces.7.html — confirms the cgroup namespace virtualizes only the *paths reported in* `/proc/<pid>/cgroup` and `/proc/<pid>/mountinfo`, not resource control.
- **`Documentation/filesystems/proc.rst`, §3.5** — https://docs.kernel.org/filesystems/proc.html — the `/proc/<pid>/mountinfo` field layout used in this lesson, including the `shared:X` / `master:X` / `propagate_from:X` / `unbindable` optional fields that are the propagation diagnostic.
- **`Documentation/admin-guide/cgroup-v2.rst`, "Device controller"** — https://docs.kernel.org/admin-guide/cgroup-v2.html — states explicitly that the v2 device controller has **no interface files** and is implemented on cgroup BPF (`BPF_PROG_TYPE_CGROUP_DEVICE`), and lists the `misc` controller's real resource types. Corrects the widespread claim that GPUs are accounted through `misc`.
- **`include/uapi/linux/sched.h`** — https://github.com/torvalds/linux/blob/master/include/uapi/linux/sched.h — the `CLONE_NEW*` flag values reproduced in the table above, plus `CLONE_INTO_CGROUP` (clone3), which links this lesson to lesson 03.
- **KEP-127: Support User Namespaces** — https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/127-user-namespaces — the `hostUsers: false` design: 65,536-UID-per-pod ranges starting at 65536, idmapped mounts to avoid recursive `chown`, and the graduation path (alpha v1.25, beta v1.35, stable v1.36) under the `UserNamespacesSupport` gate.

**Real-world engineering blogs**

- **Netflix TechBlog — "Mount Mayhem at Netflix: Scaling Containers on Modern CPUs"** — https://netflixtechblog.com/mount-mayhem-at-netflix-scaling-containers-on-modern-cpus-f3b09b68beac — *what it shows:* container start-up throughput limited by contention on the kernel's global VFS mount lock, driven by per-image-layer `mount_setattr()`/`move_mount()` idmap calls (order 20,000 syscalls in a burst), worse on multi-socket NUMA hardware; fixed with Linux 6.3's recursive-bind idmap support.
- **Netflix TechBlog — "Evolving Container Security With Linux User Namespaces"** — https://netflixtechblog.com/evolving-container-security-with-linux-user-namespaces-afbe3308c082 — *what it shows:* the opposite strategic response to the user-namespace attack surface — adopt them universally so that a container escape lands on an unprivileged host UID, rather than disabling the feature.
- **Julia Evans — "What even is a container: namespaces and cgroups"** — https://jvns.ca/blog/2016/10/10/what-even-is-a-container/ — *what it shows:* the "namespaces + cgroup + rootfs, no magic" thesis built interactively with `unshare` rather than asserted; the fastest way to make the model stick.

**Deeper dives**

- **"How Containers Work" zine (Julia Evans)** — https://wizardzines.com/zines/containers/ — builds the same namespaces + cgroups + rootfs model with hand-drawn clarity and runnable experiments; the fastest path to being able to *explain* a container out loud, which is the interview payload.
- **CVE-2022-0185 write-ups (Aqua Security, Sysdig, CrowdStrike)** — e.g. https://www.aquasec.com/blog/cve-2022-0185-linux-kernel-container-escape-in-kubernetes/ — *what they show:* the concrete exploit chain from `unshare(CLONE_NEWNS|CLONE_NEWUSER)` to `CAP_SYS_ADMIN` to an out-of-bounds write in `legacy_parse_param()` (kernels 5.1–5.16.1) to host root; the reference case for the "unprivileged capability grant reaches trusted-caller code" pattern.
