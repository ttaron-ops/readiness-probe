---
lesson: "01b.2"
title: "Namespaces"
module: "01b"
concept: "Namespaces"
status: not-started
est_time: "5h"
artifacts: []
---

# 01b.2 · Namespaces

> **Concept.** A container is not a kernel object — it's a process wrapped in a set of namespaces (plus a cgroup and a rootfs). Understand namespaces and "container" stops being magic.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Why this matters
The interview question that separates operators from engineers is "what *is* a container?" — and the wrong answer is anything involving a hypervisor, a special kernel mode, or Docker itself. On a GPU fleet you debug this weekly: a pod that can't see the GPU (device namespace/rootfs), two sidecars fighting over a port (shared net namespace), a leaked mount keeping a dataset volume busy, or an init that won't reap zombies (PID-namespace PID 1 semantics). If you can reconstruct a running container's namespace set by hand with `unshare`/`nsenter`, you can debug any of these from first principles instead of restarting the pod and hoping.

## From using to understanding
As an operator you run `docker run` / `kubectl exec` and treat the container as an opaque boundary: isolated hostname, own processes, own network, own filesystem view. You know `kubectl exec` "gets you inside" and that pods share *something*.

What you're learning now is that the "boundary" is not one thing — it's up to eight independent kernel namespaces, each virtualizing exactly one global resource, composed together. There is no "container" struct in the kernel; there is a process whose `/proc/<pid>/ns/*` entries point at a particular set of namespaces. `docker exec` is just `setns()` into that process's namespaces and `execve()`. `kubectl exec` is the same, one layer up. Once you can list, create, and enter namespaces directly, "the container" dissolves into parts you can inspect and manipulate one at a time.

## Core notes

### The namespace types
Each namespace virtualizes one global kernel resource. Current kernels have eight; here is what each isolates and the flag used to create it:

- **PID** (`CLONE_NEWPID`) — the process ID number space. The first process in a new PID namespace becomes **PID 1** inside it; the same task has a different (real) PID in the parent namespace. PID namespaces nest: the parent can see and signal the child namespace's processes (under translated PIDs), but not vice-versa.
- **NET** (`CLONE_NEWNET`) — network stack: interfaces, IP addresses, routing tables, iptables/nftables rules, `/proc/net`, port number space, sockets. A fresh net namespace has only a `lo` (down). This is where veth pairs and CNI plugins operate.
- **MNT** (`CLONE_NEWNS`) — the mount table. Processes see their own set of mounts; this plus a pivoted root gives a container its private filesystem view. Mount propagation (`shared`/`private`/`slave`) governs whether mount events cross the boundary — the source of "why is my host mount leaking into every container."
- **UTS** (`CLONE_NEWUTS`) — the hostname and NIS domain name (the `struct utsname`, hence "UTS"). This, and only this, is why `hostname` inside a container can differ from the host.
- **IPC** (`CLONE_NEWIPC`) — System V IPC objects and POSIX message queues; also POSIX shared memory (`/dev/shm`). Relevant when apps use shm segments (some ML frameworks do).
- **USER** (`CLONE_NEWUSER`) — UID/GID number space and capability sets. Maps container UIDs to host UIDs (`/proc/<pid>/uid_map`). This is what lets a process be root (UID 0) *inside* the container while being an unprivileged UID on the host — the basis of rootless containers. User namespaces can be created without privilege, which is also why they're a recurring privilege-escalation attack surface.
- **CGROUP** (`CLONE_NEWCGROUP`) — virtualizes the view of the cgroup hierarchy root, so a container sees its cgroup as `/` rather than its true host path (`/sys/fs/cgroup/...`). Cosmetic isolation of the cgroup path, not the resource control itself (that's lesson 3).
- **TIME** (`CLONE_NEWTIME`) — offsets `CLONE_MONOTONIC`/`CLONE_BOOTTIME` clocks. Rarely used by container runtimes; mentioned for completeness.

Key mental model: namespaces are **orthogonal and independently composable**. A process can be in a new net namespace but share the host's PID namespace, or any other combination. "Container" is just one common bundle of these; the kernel imposes no all-or-nothing.

### The three syscalls
- **`clone()`** — create a new process, and with `CLONE_NEW*` flags put it into new namespaces at birth. This is what runtimes call.
- **`unshare()`** — move the *calling* process into new namespaces without forking (the `unshare(1)` CLI wraps it). "Detach me from the shared resource now."
- **`setns()`** — join an *existing* namespace, given a file descriptor to a `/proc/<pid>/ns/<type>` entry. This is what `nsenter(1)`, `docker exec`, and `kubectl exec` use to get "inside."

### Where namespaces live and how to see them
Every namespace is a file under `/proc/<pid>/ns/`:

```
$ ls -l /proc/self/ns/
lrwxrwxrwx ... net -> 'net:[4026531840]'
lrwxrwxrwx ... pid -> 'pid:[4026531836]'
lrwxrwxrwx ... mnt -> 'mnt:[4026531841]'
lrwxrwxrwx ... uts -> 'uts:[4026531838]'
...
```

The number in brackets is the **namespace inode** — its unique identity. Two processes are in the *same* namespace iff these inodes match. This is exactly how you prove "these two containers share a net namespace": compare the `net:[...]` inodes. Root/initial namespaces conventionally start at `4026531xxx`.

`lsns` reads all of these and tabulates them:

```
$ lsns -t net
        NS TYPE NPROCS   PID USER  NETNSID COMMAND
4026531840 net     412     1 root  unassigned  /sbin/init
4026533012 net       7 88231 root         0    /pause          # a k8s pod's net ns
```

`NPROCS` = how many processes share that namespace. A pod's net namespace shared by the `pause` container and every app container shows up as one net NS with many procs.

### The container thesis (half of the module anchor)
A container is: **namespaces** (this lesson) **+ a cgroup** (lesson 3, resource limits) **+ a rootfs** (an image unpacked and pivoted-into as the mount namespace's root). Nothing else. No hypervisor, no special CPU mode, no kernel-side "container" object. The runtime (`runc`/`crun`) `clone()`s a process into fresh namespaces, sets up the cgroup, `pivot_root`s into the image's rootfs, drops capabilities, and `execve()`s your entrypoint. Everything Docker/containerd add above that — image distribution, networking plumbing, lifecycle — is orchestration around this kernel primitive.

## Worked example
Build a container by hand, then dissect a real one.

**1. Hand-rolled container.** Create new PID, mount, net, and UTS namespaces and observe PID 1:

```
$ sudo unshare --pid --fork --mount-proc --net --uts bash
# hostname handmade                 # UTS ns → independent hostname
# hostname
handmade
# echo $$                           # our shell's PID *inside* the new pid ns
1                                    # we are PID 1 — PID namespace at work
# ps -e                             # --mount-proc gave us a fresh /proc view
    PID TTY          TIME CMD
      1 pts/0    00:00:00 bash
     10 pts/0    00:00:00 ps
# ip link                           # new net ns: only loopback, and it's down
1: lo: <LOOPBACK> mtu 65536 ... state DOWN
```

Four kernel resources are now virtualized for this shell, and it took one syscall wrapper. `--mount-proc` is needed because `ps` reads `/proc`; without a fresh proc mount in the new mount namespace you'd see the host's processes.

**2. Dissect a running container.** In another terminal, find a real container's namespaces. Get its host PID, then read its ns links:

```
$ docker inspect -f '{{.State.Pid}}' <container>      # or crictl inspect for k8s
88231
$ ls -l /proc/88231/ns/
... net -> 'net:[4026533012]'
... pid -> 'pid:[4026533105]'
... uts -> 'uts:[4026533011]'
... mnt -> 'mnt:[4026533009]'
$ lsns -p 88231                       # every namespace this pid belongs to, one row each
        NS TYPE   NPROCS   PID USER COMMAND
4026533009 mnt         3 88231 root nginx
4026533011 uts         7 88231 root nginx
4026533012 net         7 88231 root nginx   # note: shared by 7 procs (the pod)
...
```

**3. Enter it by hand** (this is what `docker exec` does under the hood):

```
$ sudo nsenter -t 88231 -a           # join all of pid 88231's namespaces
# hostname                           # now inside the container's UTS ns
<container-hostname>
# ip addr                            # container's net ns: its veth/eth0, not the host's
```

You just reproduced `docker exec` with `nsenter` + the `/proc/<pid>/ns/*` fds — proving the "exec" is `setns()` into an existing set, nothing more.

## Practice
On a Linux laptop/VM with Docker or containerd:

1. **Build a container by hand.** Run `sudo unshare --pid --fork --mount-proc --net --uts bash`. Inside: set a hostname, confirm `echo $$` is 1, run `ps -e` (should show only your shell + ps), and `ip link` (only a down `lo`). Note each observation against which namespace produced it.
2. **Reconstruct a real container's namespace set.** Start any container (`docker run -d --name probe nginx`). Find its host PID (`docker inspect -f '{{.State.Pid}}' probe`). List its namespaces two ways: `ls -l /proc/<pid>/ns/` (record the inode numbers) and `lsns -p <pid>`. Identify which namespaces are new vs shared with the host by comparing inodes against `ls -l /proc/1/ns/`.
3. **Enter it.** `sudo nsenter -t <pid> -a` (or `-n -u -m -p`), then confirm you see the container's hostname and network, not the host's. Compare against `docker exec probe hostname`.

**Acceptance:** (i) a reconstructed table of the container's namespaces with their inode numbers and which are shared with host vs private; (ii) evidence you entered it by hand via `nsenter` (a command output that differs from the host, e.g. hostname or `ip addr`); (iii) a 3–4 sentence "a container is just…" paragraph in your own words naming namespaces + cgroup + rootfs. Clean up: `docker rm -f probe`.

## Self-check
**(a) What does being PID 1 in a pid namespace mean for signal handling and zombie reaping?**
**Answer:** PID 1 in a namespace is the "init" of that namespace and gets special kernel treatment: signals without an explicitly installed handler are **not** delivered to it by default (so a naive `SIGTERM` to PID 1 is ignored unless the process installs a handler — the classic "container ignores docker stop" bug). It is also the reaper of last resort: when any process in the namespace is orphaned, it's reparented to PID 1, which must `wait()` to reap it or leave a zombie. A shell or app that isn't written to reap children accumulates zombies — which is why runtimes offer a real init (`tini`, `--init`) as PID 1 to forward signals and reap. When PID 1 exits, the whole namespace's processes are killed.

**(b) Which namespace makes `hostname` independent, and which makes the process tree independent?**
**Answer:** The **UTS** namespace (`CLONE_NEWUTS`) isolates the hostname (and NIS domain) — it virtualizes `struct utsname`, so `hostname` inside differs from the host. The **PID** namespace (`CLONE_NEWPID`) isolates the process ID space and thus the process tree — the first process becomes PID 1 and only sees processes within its namespace (with a fresh `/proc` mounted via the mount namespace). They're independent: you can have a private hostname without a private process tree, or vice-versa.

**(c) How do two containers in one Kubernetes pod share a network namespace, and what stays separate?**
**Answer:** All containers in a pod are `setns()`-joined into the **same network namespace** — held open by the pod's `pause`/infra container — so they share one `eth0`, one IP, one routing table, and one port space (which is why two containers in a pod can't both bind :8080, and can reach each other on `localhost`). What stays separate is typically the **mount** namespace (each container has its own rootfs/filesystem view) and often the **PID** namespace (separate process trees, unless `shareProcessNamespace: true`) and **UTS/IPC** depending on config. So: shared network and (usually) IPC via the pause container's namespaces; private filesystem and process tree per container.

## Resources
1. **"How Containers Work" zine (Julia Evans)** — https://wizardzines.com/zines/containers/ — builds the exact namespaces + cgroups + rootfs mental model this lesson teaches, with hand-drawn clarity and runnable experiments. *Deep.* Why: it's the fastest path to being able to *explain* a container, which is the interview payload.
2. **namespaces(7) (man7)** — https://man7.org/linux/man-pages/man7/namespaces.7.html — the authoritative per-namespace reference: every type, the `/proc/<pid>/ns/*` interface, and the clone/unshare/setns semantics. *Skim, then reference.* Why: canonical source to confirm exact flags and edge cases (user-ns capabilities, ownership).
3. **namespaces comic (Julia Evans)** — https://wizardzines.com/comics/namespaces/ — one-page visual of the namespace types and what each isolates. *Skim.* Why: a fast recall aid before an interview.
