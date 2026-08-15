---
render_with_liquid: false
lesson: "01b.2"
title: "Namespaces"
module: "01b"
concept: "Namespaces"
status: not-started
est_time: "7h"
prev: "01-processes-and-scheduling.md"
next: "03-cgroups-v2-and-k8s-enforcement.md"
artifacts: []
sources: 6
---

# 01b.2 · Namespaces

> **Concept.** A container is not a kernel object — it's a process wrapped in a set of namespaces (plus a cgroup and a rootfs). Understand namespaces and "container" stops being magic.
>
> Module: [🐧 01b — Linux systems internals](../README.md) · Deliverable: [Anatomy of a Container](../practice/anatomy-of-a-container/README.md)

## Where this fits
[01 — Processes and Scheduling](01-processes-and-scheduling.md) established that a process is a `task_struct` — a kernel state machine with a scheduler entry, not a mysterious "running program." That model is the substrate this lesson builds on: namespaces don't create a new kind of kernel object, they change what a `task_struct` is allowed to *see*. A process in a fresh PID namespace is still just a `task_struct` being scheduled exactly as lesson 01 described — it just has a different, private view of the process-ID space. This lesson takes that same process and shows how up to eight independent namespace types can each virtualize one global kernel resource around it, which is what makes "container" possible without any hypervisor or special CPU mode. It unlocks the ability to reconstruct any running container's isolation boundary by hand, and it sets up the other half of the container thesis: [03 — cgroups v2 and K8s resource enforcement](03-cgroups-v2-and-k8s-enforcement.md), the anchor lesson of this module, which adds the *resource-limiting* half (a cgroup) to the *view-virtualizing* half (namespaces) covered here.

## Why this matters
The interview question that separates operators from engineers is "what *is* a container?" — and the wrong answer is anything involving a hypervisor, a special kernel mode, or Docker itself. On a GPU fleet you debug this weekly: a pod that can't see the GPU (device visibility, not a namespace at all — see below), two sidecars fighting over a port (shared net namespace), a leaked mount keeping a dataset volume busy (mount-namespace propagation), or an init that won't reap zombies (PID-namespace PID-1 semantics). If you can reconstruct a running container's namespace set by hand with `unshare`/`nsenter`, you can debug any of these from first principles instead of restarting the pod and hoping. This is also where the module's "why kernel mechanism over Linux admin" thesis gets concrete: the GPU/neocloud interview bar (CoreWeave, Datadog, NVIDIA — see the module README) rewards exactly this kind of "I can take the abstraction apart" answer over "I know the `docker` CLI."

## What's new here (calibration)
As with lesson 1, this module skips shell/pipes/permissions, package managers, distro tours, and general networking basics — you already have those. What's new here specifically:

- The full set of eight namespace types and which single global resource each one virtualizes — most engineers can name three or four; staff-level fluency means all eight and their flags.
- The precise mechanics of the three syscalls (`clone`, `unshare`, `setns`) and how `docker exec`/`kubectl exec` map onto `setns()` — not "exec starts a process in the container," but exactly what it does and doesn't do.
- **GPU-fleet-specific:** device visibility is *not* namespace-virtualized at all — there is no "GPU namespace." This surprises people who assume every resource a container "has" maps to a namespace.
- The user-namespace privilege-escalation surface, and why some hardened GPU-fleet nodes disable unprivileged user-namespace creation entirely.
- Mount propagation modes (`shared`/`private`/`slave`) as the actual mechanism behind a very common production bug: a host mount leaking into every container.

## Core concepts

### The namespace types
Each namespace virtualizes one global kernel resource. Current kernels have eight; here is what each isolates and the flag used to create it:

- **PID** (`CLONE_NEWPID`) — the process ID number space. The first process in a new PID namespace becomes **PID 1** inside it; the same task has a different (real) PID in the parent namespace. PID namespaces nest: the parent can see and signal the child namespace's processes (under translated PIDs), but not vice-versa.
- **NET** (`CLONE_NEWNET`) — network stack: interfaces, IP addresses, routing tables, iptables/nftables rules, `/proc/net`, port number space, sockets. A fresh net namespace has only a `lo` (down). This is where veth pairs and CNI plugins operate.
- **MNT** (`CLONE_NEWNS`) — the mount table. Processes see their own set of mounts; this plus a pivoted root gives a container its private filesystem view. Mount propagation (`shared`/`private`/`slave`) governs whether mount events cross the boundary — the source of "why is my host mount leaking into every container" (worked below).
- **UTS** (`CLONE_NEWUTS`) — the hostname and NIS domain name (the `struct utsname`, hence "UTS"). This, and only this, is why `hostname` inside a container can differ from the host.
- **IPC** (`CLONE_NEWIPC`) — System V IPC objects and POSIX message queues; also POSIX shared memory (`/dev/shm`). Relevant when apps use shm segments (some ML frameworks do).
- **USER** (`CLONE_NEWUSER`) — UID/GID number space and capability sets. Maps container UIDs to host UIDs (`/proc/<pid>/uid_map`). This is what lets a process be root (UID 0) *inside* the container while being an unprivileged UID on the host — the basis of rootless containers. User namespaces can be created **without privilege by any user**, which is exactly why they're a recurring privilege-escalation attack surface (see the failure-mode perspective below).
- **CGROUP** (`CLONE_NEWCGROUP`) — virtualizes the view of the cgroup hierarchy root, so a container sees its cgroup as `/` rather than its true host path (`/sys/fs/cgroup/...`). Cosmetic isolation of the cgroup path, not the resource control itself (that's [lesson 3](03-cgroups-v2-and-k8s-enforcement.md)).
- **TIME** (`CLONE_NEWTIME`) — offsets `CLONE_MONOTONIC`/`CLONE_BOOTTIME` clocks. Rarely used by container runtimes; mentioned for completeness.

Key mental model: namespaces are **orthogonal and independently composable**. A process can be in a new net namespace but share the host's PID namespace, or any other combination. "Container" is just one common bundle of these; the kernel imposes no all-or-nothing. See [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html) for the authoritative per-type reference.

### The three syscalls
- **`clone()`** — create a new process, and with `CLONE_NEW*` flags put it into new namespaces at birth. This is what runtimes call.
- **`unshare()`** — move the *calling* process into new namespaces without forking (the `unshare(1)` CLI wraps it). "Detach me from the shared resource now."
- **`setns()`** — join an *existing* namespace, given a file descriptor to a `/proc/<pid>/ns/<type>` entry. This is what `nsenter(1)`, `docker exec`, and `kubectl exec` use to get "inside." Crucially, `setns()` does not create anything — it moves the calling process's view. This is the mechanical reason `docker exec` is not "starting a new container": there is no new container, just a new process joining the existing set of namespaces.

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
A container is: **namespaces** (this lesson) **+ a cgroup** ([lesson 3](03-cgroups-v2-and-k8s-enforcement.md), resource limits) **+ a rootfs** (an image unpacked and pivoted-into as the mount namespace's root). Nothing else. No hypervisor, no special CPU mode, no kernel-side "container" object. The runtime (`runc`/`crun`) `clone()`s a process into fresh namespaces, sets up the cgroup, `pivot_root`s into the image's rootfs, drops capabilities, and `execve()`s your entrypoint. Everything Docker/containerd add above that — image distribution, networking plumbing, lifecycle — is orchestration around this kernel primitive.

### Device visibility: why there's no "GPU namespace"
It's tempting to assume every resource a container appears to "have" is namespace-virtualized, the way PIDs and mounts are. GPUs are the clearest counterexample: **there is no device namespace.** A `/dev/nvidia0` device node is a host-global object; namespaces don't hide it, duplicate it, or give a container a private version of it. What actually controls GPU visibility inside a container is two other mechanisms working together:

1. The **device cgroup** (`devices.allow`/`devices.list` in cgroup v1, or the newer **misc controller** in cgroup v2 for some accelerator accounting) — an allow/deny list of which device major:minor numbers a cgroup's processes may open at all. This is what stops a container from opening every `/dev/nvidia*` node on the host by default.
2. **Bind-mounting the specific device nodes** the container is allowed to use into its mount namespace (e.g. `/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-uvm`) plus the driver libraries.

This is precisely why the NVIDIA device plugin for Kubernetes does **explicit device-file injection**: it doesn't rely on any namespace mechanism, because none exists for devices. It computes which GPU(s) a pod was allocated, then bind-mounts exactly those device nodes into the container's mount namespace and grants device-cgroup access to them. Understanding this is the difference between "the pod can't see the GPU, must be a namespace bug" (wrong layer) and correctly checking the device plugin's allocation and the device cgroup/mount list (right layer).

### Mount propagation: the "leaked hostPath" bug
Mount namespaces aren't fully independent by default — the kernel supports **propagation modes** that control whether a mount event in one namespace is reflected in another:

- **private** — mount/unmount events stay local to this namespace. The default and safest for container isolation.
- **shared** — mount/unmount events propagate to and from a peer group of namespaces. Two namespaces in the same shared peer group see each other's new mounts.
- **slave** — receives propagation from a shared master but doesn't propagate back to it (one-directional).

The classic production bug: a `hostPath` volume mounted with `mountPropagation: Bidirectional` (or a host mount namespace left in `shared` mode) means a mount performed *inside* one container becomes visible on the host and in every other container sharing that propagation group — "why did my hostPath mount leak into every pod" is almost always this, not a Kubernetes bug. The fix is scoping propagation to `HostToContainer` (slave — see host mounts, don't leak yours back) or `None` (private) unless bidirectional propagation is a deliberate design (e.g. some CSI drivers/storage plugins legitimately need it).

## Perspectives

**Kernel-mechanism view.** Namespaces are orthogonal, independently composable virtualizations of exactly one global resource each — PID, net, mount, UTS, IPC, user, cgroup, time. There is no combined "container" primitive in the kernel; there is a `task_struct` (lesson 1's process) whose `/proc/<pid>/ns/*` entries happen to point at a particular bundle of namespace instances. Composing eight independent, single-purpose virtualizations is the whole trick — nothing more exotic is happening underneath `docker run`.

**Operator/SRE view.** "Which namespace explains this symptom" is a debugging reflex worth building deliberately. A pod that can't see the GPU is a device-cgroup/mount problem, not a namespace problem at all (see above). Two sidecars port-clashing is the shared network namespace doing exactly what it's supposed to — one port space, one IP, for the whole pod. A pile of zombie processes in a container is PID-1 signal/reaping semantics from lesson 1's process model, now specific to the namespace's own init. Each symptom maps to a specific, checkable kernel mechanism instead of a guess.

**GPU-fleet-specific view.** Device visibility is the sharpest place this module's "no magic namespace for everything" lesson bites: GPUs are exposed to a container via the device cgroup plus explicit bind-mounts, not a virtualized device namespace. This is exactly why the NVIDIA device plugin exists as a piece of software at all — it has real work to do (computing allocations, injecting device nodes, granting cgroup access) precisely because the kernel gives it no namespace shortcut. A staff engineer debugging "pod requested a GPU but sees none" checks the device plugin's allocation and the container's actual `/dev` bind-mounts, not `ip netns` or `unshare` flags.

**Failure-mode view.** User namespaces can be created by any unprivileged user by default on most distributions — that's the feature that makes rootless containers possible, and it's also a recurring source of local-privilege-escalation CVEs, because a process can gain `CAP_SYS_ADMIN` and other capabilities *inside* its own new user namespace, then exploit kernel code paths that don't correctly account for the namespace boundary (a repeated pattern across several kernel CVEs over the years). This is exactly why some hardened fleet nodes set `kernel.unprivileged_userns_clone=0` (or the distribution equivalent) — deliberately trading away rootless-container convenience for a smaller attack surface. See [user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html) for the capability semantics behind this.

## Real-world use cases

- **Netflix TechBlog — "Mount Mayhem at Netflix: Scaling Containers on Modern CPUs"** — https://netflixtechblog.com/mount-mayhem-at-netflix-scaling-containers-on-modern-cpus-f3b09b68beac — migrating Docker to containerd, Netflix found kernel VFS global mount-lock contention limiting concurrent container scale-up on modern many-core CPUs — a direct production consequence of mount-namespace mechanics not being free at scale. The post also covers containerd's use of the kernel's idmap feature for efficient per-container UID mapping. *What it shows:* mount namespaces aren't a zero-cost abstraction at high container density — the underlying VFS locking is a real, measurable bottleneck at fleet scale.
- **Julia Evans — "What even is a container: namespaces and cgroups"** — https://jvns.ca/blog/2016/10/10/what-even-is-a-container/ — a hands-on `unshare` walkthrough arriving at "container = namespaces + cgroup + rootfs, no magic." *What it shows:* the exact thesis of this lesson, demonstrated interactively rather than asserted — a good companion/next-read alongside the worked example below.

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

**4. Confirm the "no GPU namespace" claim.** On a GPU node, compare a GPU container's device visibility against its namespace set:

```
$ docker inspect -f '{{.State.Pid}}' <gpu-container>
91002
$ ls -l /proc/91002/root/dev/ | grep nvidia
crw-rw-rw- ... nvidia0
crw-rw-rw- ... nvidiactl
crw-rw-rw- ... nvidia-uvm
$ cat /sys/fs/cgroup/<container-scope>/devices.list      # cgroup v1; v2 uses eBPF-based device filters
c 195:0 rw                                                 # nvidia0 major:minor, explicitly allowed
```

Notice: no `device` entry ever appeared in `/proc/91002/ns/` in step 2 — because there isn't one. GPU access is a bind-mounted device node plus a device-cgroup allow rule, exactly as the Core concepts section described.

## Practice
On a Linux laptop/VM with Docker or containerd. This feeds directly into the module deliverable, [Anatomy of a Container](../practice/anatomy-of-a-container/README.md) — the namespace inventory and hand-entry steps here are the first layer of that teardown.

1. **Build a container by hand.** Run `sudo unshare --pid --fork --mount-proc --net --uts bash`. Inside: set a hostname, confirm `echo $$` is 1, run `ps -e` (should show only your shell + ps), and `ip link` (only a down `lo`). Note each observation against which namespace produced it.
2. **Reconstruct a real container's namespace set.** Start any container (`docker run -d --name probe nginx`). Find its host PID (`docker inspect -f '{{.State.Pid}}' probe`). List its namespaces two ways: `ls -l /proc/<pid>/ns/` (record the inode numbers) and `lsns -p <pid>`. Identify which namespaces are new vs shared with the host by comparing inodes against `ls -l /proc/1/ns/`.
3. **Enter it.** `sudo nsenter -t <pid> -a` (or `-n -u -m -p`), then confirm you see the container's hostname and network, not the host's. Compare against `docker exec probe hostname`.
4. **Trace a mount-propagation leak (stretch goal).** Bind-mount a scratch directory into a container with `--mount type=bind,source=/tmp/shared,target=/mnt,bind-propagation=shared` vs the default (`rprivate`), create a file inside the container's `/mnt` via a *nested* mount, and observe whether it appears on the host — a hands-on reproduction of the Netflix-style propagation bug described above.

**Acceptance:** (i) a reconstructed table of the container's namespaces with their inode numbers and which are shared with host vs private; (ii) evidence you entered it by hand via `nsenter` (a command output that differs from the host, e.g. hostname or `ip addr`); (iii) a 3–4 sentence "a container is just…" paragraph in your own words naming namespaces + cgroup + rootfs, and explicitly noting that device visibility (e.g. GPUs) is not part of that namespace set. Clean up: `docker rm -f probe`.

## Common pitfalls
1. **Believing there's a device/GPU namespace.** There isn't — the device cgroup (`devices.allow`/`devices.list`, or the misc controller) plus explicit bind mounts do that job. This is why the NVIDIA device plugin has to inject device nodes explicitly rather than relying on any namespace mechanism.
2. **Assuming `docker exec`/`kubectl exec` "starts a new container."** It's `setns()` into the existing namespaces of an already-running process, followed by `execve()` — nothing is created, nothing is restarted.
3. **Forgetting PID-1 signal semantics.** A naive image without a real init forwards no signal handlers by default, so `SIGTERM`/`docker stop` appears to "not work" and the runtime falls through to `SIGKILL` after the grace period — the fix is a minimal init (`tini`, `--init`) as PID 1, not a longer timeout.
4. **Treating "shares the pod's network namespace" as "shares everything."** Mount, PID, UTS, and IPC are typically still private per-container within a pod (unless explicitly configured, e.g. `shareProcessNamespace: true`) — only network (and usually IPC, via the pause container) is pod-wide by default.
5. **Not accounting for mount-namespace/VFS lock contention at high container density.** As the Netflix "Mount Mayhem" case shows, mount-namespace operations aren't free at scale — a global kernel VFS lock can become a real scale-up bottleneck on many-core nodes running many containers, independent of any per-container resource limit.

## Self-check
- **What does being PID 1 in a pid namespace mean for signal handling and zombie reaping?**
  **Answer:** PID 1 in a namespace is the "init" of that namespace and gets special kernel treatment: signals without an explicitly installed handler are **not** delivered to it by default (so a naive `SIGTERM` to PID 1 is ignored unless the process installs a handler — the classic "container ignores docker stop" bug). It is also the reaper of last resort: when any process in the namespace is orphaned, it's reparented to PID 1, which must `wait()` to reap it or leave a zombie. A shell or app that isn't written to reap children accumulates zombies — which is why runtimes offer a real init (`tini`, `--init`) as PID 1 to forward signals and reap. When PID 1 exits, the whole namespace's processes are killed.
- **Which namespace makes `hostname` independent, and which makes the process tree independent?**
  **Answer:** The **UTS** namespace (`CLONE_NEWUTS`) isolates the hostname (and NIS domain) — it virtualizes `struct utsname`, so `hostname` inside differs from the host. The **PID** namespace (`CLONE_NEWPID`) isolates the process ID space and thus the process tree — the first process becomes PID 1 and only sees processes within its namespace (with a fresh `/proc` mounted via the mount namespace). They're independent: you can have a private hostname without a private process tree, or vice-versa.
- **How do two containers in one Kubernetes pod share a network namespace, and what stays separate?**
  **Answer:** All containers in a pod are `setns()`-joined into the **same network namespace** — held open by the pod's `pause`/infra container — so they share one `eth0`, one IP, one routing table, and one port space (which is why two containers in a pod can't both bind :8080, and can reach each other on `localhost`). What stays separate is typically the **mount** namespace (each container has its own rootfs/filesystem view) and often the **PID** namespace (separate process trees, unless `shareProcessNamespace: true`) and **UTS/IPC** depending on config. So: shared network and (usually) IPC via the pause container's namespaces; private filesystem and process tree per container.
- **A pod's container reports it "can't see the GPU." Which namespace is misconfigured?**
  **Answer:** Trick question — none, necessarily. There is no GPU/device namespace, so device visibility isn't a namespace property at all. The right places to check are the device cgroup (is the device major:minor actually allowed for this container's cgroup?) and the mount namespace's `/dev` entries (did the NVIDIA device plugin actually bind-mount `/dev/nvidia0` and friends into this container?). Debugging it as a namespace problem sends you looking in the wrong subsystem entirely.
- **Why do some hardened GPU-fleet nodes disable unprivileged user-namespace creation?**
  **Answer:** User namespaces can be created by any unprivileged user by default, which lets that user gain capabilities like `CAP_SYS_ADMIN` inside their own new user namespace — the basis of rootless containers, but also a recurring local-privilege-escalation surface when kernel code paths mishandle the namespace boundary (a pattern behind multiple real CVEs). Hardened nodes trade away rootless-container convenience by setting `kernel.unprivileged_userns_clone=0` (or the distribution equivalent) to shrink that attack surface, accepting that rootless workflows on those nodes need another privilege path instead.

## Connections & what's next
This lesson is the "views" half of the container thesis; [03 — cgroups v2 and K8s resource enforcement](03-cgroups-v2-and-k8s-enforcement.md) — the anchor lesson of this module — adds the "limits" half: the same process, now also bounded by `cpu.max`/`memory.max`/the device cgroup this lesson introduced for GPU visibility. The CGROUP namespace covered here (cosmetic path virtualization) is easy to confuse with cgroup *resource control* — lesson 3 draws that line precisely. Later, [08 — eBPF](08-ebpf.md) revisits namespaces from the observability side (how Cilium/Hubble attach to and see across network namespaces), and the device-cgroup mechanics introduced here for GPU visibility recur directly in lesson 3's device-controller coverage. The immediate next step is **[03 — cgroups v2 and K8s resource enforcement](03-cgroups-v2-and-k8s-enforcement.md)**: same processes, same namespaces — now with hard resource limits attached.

## References & further reading

**Primary sources**
- namespaces(7) (man7) — https://man7.org/linux/man-pages/man7/namespaces.7.html — the authoritative per-namespace reference: every type, the `/proc/<pid>/ns/*` interface, and the clone/unshare/setns semantics.
- user_namespaces(7) (man7) — https://man7.org/linux/man-pages/man7/user_namespaces.7.html — the capability and privilege-escalation semantics behind unprivileged user-namespace creation.

**Real-world engineering blogs**
- Netflix TechBlog — "Mount Mayhem at Netflix: Scaling Containers on Modern CPUs" — https://netflixtechblog.com/mount-mayhem-at-netflix-scaling-containers-on-modern-cpus-f3b09b68beac — what it shows: mount-namespace/VFS lock contention as a real scale-up bottleneck at high container density, plus containerd's idmap-based UID mapping.
- Julia Evans — "What even is a container: namespaces and cgroups" — https://jvns.ca/blog/2016/10/10/what-even-is-a-container/ — what it shows: the "namespaces + cgroup + rootfs, no magic" thesis demonstrated hands-on with `unshare`.

**Deeper dives**
- "How Containers Work" zine (Julia Evans) — https://wizardzines.com/zines/containers/ — builds the exact namespaces + cgroups + rootfs mental model this lesson teaches, with hand-drawn clarity and runnable experiments; the fastest path to being able to *explain* a container, which is the interview payload.
- namespaces comic (Julia Evans) — https://wizardzines.com/comics/namespaces/ — one-page visual of the namespace types and what each isolates; a fast recall aid before an interview.
