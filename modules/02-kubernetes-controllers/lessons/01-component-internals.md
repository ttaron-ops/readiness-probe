---
lesson: "02.1"
title: "Kubernetes component internals — how each component works"
module: "02"
concept: "Kubernetes component internals"
status: not-started
est_time: "18h"
prev: null
next: "02-api-machinery.md"
artifacts: []
sources: 9
---

# 02.1 · Kubernetes component internals — how each component works

> **Concept.** The internal machinery of each control-plane and node component — request flow, caches, control loops, consensus — the layer beneath what a CKA operator already knows.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Where this fits

Module 01 built `gpu-cost-exporter` — a Go binary that reads GPU metrics and talks to *a* Kubernetes API, treated mostly as a black box you call. This module turns that black box into a machine you can extend: a controller that watches, reconciles, and writes back. Before you can build that controller, you need to know what it's actually a client *of* — what happens inside the apiserver when it writes, what the scheduler does before your pod lands on a node, and why a component you'll depend on (leader election, the watch cache, PLEG) behaves the way it does under load. This lesson is that foundation: no new Kubernetes concepts you haven't used, just the internals of the ones you already operate.

## Why this matters

Your capstone operator is a client of these internals, not a bystander to them. It will hold a `watch` against the apiserver's **watch cache** and must survive reconnects without dropping GPU-node events — so `resourceVersion` semantics are load-bearing, not trivia. It will run HA behind **leader election** for the exact reason kube-controller-manager does. And to reason about GPU placement cost you have to know how the **scheduler** actually binds a pod — device-plugin integer counts vs DRA claims change what "a GPU node is full" even means.

It's also where "operates Kubernetes" and "builds on Kubernetes" interviews diverge hardest. A CKA holder can explain *that* a node went `NotReady`; a Senior/Staff platform engineer building controllers for a GPU fleet is expected to explain *why* — PLEG stall vs missed lease renewal vs a genuinely dead kubelet are three different root causes with three different fixes, and conflating them wastes an incident's worth of time on a fleet where GPU-hours are burning idle the whole time you're debugging. NVIDIA and CoreWeave job descriptions for these roles explicitly ask for "extending Kubernetes components" and owning "the control plane" — this lesson is the first layer of that competence.

## What's new here (calibration)

You have a CKA and run 40+ clusters, so this lesson does **not** re-teach: what a pod/node/Service is, `kubectl` usage, YAML authoring, Helm/Kustomize, CNI/CSI/Ingress *usage*, or cluster bring-up. What it adds instead:

- **Internal request/data flow inside each component** — the apiserver's admission pipeline order, the scheduler's Filter/Score/Reserve/Bind cycle, the kubelet's syncLoop and PLEG — not "what the component does" but "how it does it and what can make it slow or wrong."
- **The one structural fact that unifies all of it** — every component is a client of the apiserver, and the apiserver is the only client of etcd — which is the mental model your controller inherits.
- **Failure-mode causality** — PLEG stall vs lease expiry vs watch-cache staleness as distinguishable, diagnosable conditions, not synonyms for "something's wrong."
- **GPU-specific placement machinery** — device-plugin integer counts vs DRA's claim/attribute model, which changes what your cost operator can even observe about GPU allocation.

## Core concepts

The single most important structural fact: **every component is a client of the apiserver, and the apiserver is the only client of etcd.** No component watches etcd, shares memory, or messages another component directly. All coordination is objects-in-etcd, read/written through the apiserver, observed via watches. Internalize this and the rest is variations on list-watch-reconcile.

```
   etcd (Raft quorum, MVCC)
     ▲  only writer/reader
     │
 kube-apiserver ── watch cache ──┬───────────────┬────────────────┬───────────────┐
     ▲                           │               │                │               │
     │ (all via REST/watch)      ▼               ▼                ▼               ▼
 kubectl / you            kube-scheduler   controller-mgr   cloud-controller   kubelet (per node)
                          (binds pods)     (built-ins,      (LB/route/node)      │ CRI (gRPC)
                                            leader-elected)                       ▼
                                                                        containerd / CRI-O ─▶ runc
   kube-proxy (per node) ── watches Svc/EndpointSlice ──▶ programs kernel datapath
   CoreDNS               ── watches Svc/EndpointSlice ──▶ answers cluster DNS
```

### kube-apiserver

The apiserver is a stateless HTTP server in front of etcd. Every request walks a fixed pipeline; understanding the order is how you predict *what* rejects a request and *where* a webhook can mutate it.

```
request
  │
  ├─ 1. authN            who are you?  (client cert / bearer token / OIDC / webhook)
  ├─ 2. authZ            may you do this verb on this resource?  (RBAC / Node / Webhook)
  ├─ 3. mutating admission     webhooks + built-ins may REWRITE the object (defaulting, sidecar inject)
  ├─ 4. schema validation      OpenAPI / structural schema (CRD) + apiVersion conversion
  ├─ 5. validating admission   webhooks + built-ins may REJECT (no mutation)  ← incl. ValidatingAdmissionPolicy (CEL)
  └─ 6. etcd write             serialize (protobuf for core) → store at a new revision
```

Key internals:

- **Only component that talks to etcd.** Scheduler, controllers, kubelet — none touch etcd. They all go through the apiserver's REST API. This is deliberate: it centralizes auth, validation, admission, and versioned conversion in one place. Your operator inherits all of it for free.
- **Storage vs serving version.** Objects are stored in etcd in one *storage version* (protobuf for built-ins), but served in whatever `apiVersion` the client asks for. The apiserver converts on read/write through an internal "hub" type. CRDs are stored as JSON and converted via optional conversion webhooks.
- **Watch cache.** For each resource type the apiserver keeps an in-memory ring buffer of recent events, populated from a *single* etcd watch. Thousands of client `watch`es (informers) are served from this cache, not from etcd — otherwise etcd would be hammered by every kubelet and controller. A `LIST` with `resourceVersion=0` is served from the cache (cheap, possibly slightly stale); a `LIST` with no RV forces a quorum read from etcd (consistent, expensive).
- **resourceVersion.** An opaque string that is really the etcd MVCC revision. It is monotonic *per cluster* (global etcd revision), not per object. A `watch` says "send me everything after RV=N." On reconnect the client re-sends its last-seen RV; if that RV has been **compacted** out of etcd/the cache, the apiserver returns `410 Gone`, and the informer must **relist** (full LIST) then resume watching. This is why informers are self-healing and why your operator never assumes a watch stream is gap-free.
- **List/watch semantics + pagination.** Large LISTs are chunked with `limit`/`continue` tokens. Modern clients use `WatchList` / streaming list (feature-gated `WatchListClient`) to stream an initial state as watch events rather than one giant response, cutting apiserver memory spikes.
- **Aggregation layer.** An `APIService` object can delegate a whole API group (e.g. `metrics.k8s.io`) to a separate *extension apiserver*. The main apiserver becomes a proxy for those paths. `metrics-server` and `custom.metrics.k8s.io` (which your cost operator may consume or expose) live here. This is different from **CRDs**, which the *main* apiserver serves directly from a generic registry — no second process, just a schema and etcd storage.
- **Admission webhooks' place.** Mutating (step 3) and validating (step 5) webhooks are out-of-process HTTP calls in the request path — so a down webhook with `failurePolicy: Fail` blocks writes clusterwide. `ValidatingAdmissionPolicy`/`MutatingAdmissionPolicy` (CEL, in-process, GA/beta in the 1.3x line) move common cases out of webhooks to avoid that fragility.
- **API Priority and Fairness (APF).** Requests are classified by `FlowSchema` into `PriorityLevelConfiguration`s, each with a share of concurrent seats and fair-queuing across flows. This replaced the old global `--max-inflight` knobs: a chatty controller (or a busted one hot-looping LISTs) is throttled within its priority level instead of starving `leader-election` or `system` traffic. When your operator gets `429 Too Many Requests`, this is why — respect the `Retry-After`. LISTs are expensive "seats"; prefer watches and cached (`resourceVersion=0`) reads to stay cheap. This is not theoretical: OpenAI's public write-ups on scaling their training clusters describe running **5 kube-apiservers and 5 etcd members** specifically to isolate blast radius from exactly this failure class — one controller doing unbounded LISTs degrading the whole control plane — and tuning or disabling controllers (e.g. DaemonSet/EndpointSlice controllers) that LIST too aggressively at very high pod counts. (Their posts are 2021/2023 vintage — cite as historical engineering case, not current headcount.)

### etcd

etcd is a strongly-consistent KV store; the apiserver is its only meaningful client.

- **Raft + quorum.** A cluster of (usually) 3 or 5 members elects a leader; a write commits only after a **majority** (quorum) persists it to the Raft log. 3 members tolerate 1 failure, 5 tolerate 2. Even member counts are pointless (4 still only tolerates 1) and hurt latency.
- **MVCC revisions.** Every write bumps a global **revision**; old values are kept until compaction. This is the mechanism behind `resourceVersion` and behind watches ("give me changes since revision N").
- **Watch streams.** etcd streams change events from a revision forward. The apiserver's watch cache is fed by exactly one such stream per resource; this is the ultimate origin of every informer event in the cluster.
- **Compaction vs defragmentation.** *Compaction* discards MVCC history older than a revision (frees logical space, drops old RVs → source of `410 Gone`). *Defragmentation* returns the freed pages to the filesystem (shrinks the actual DB file) and briefly blocks the member — run it rolling, one member at a time.
- **Transactions & leases (etcd's, not k8s Lease objects).** The apiserver uses etcd `Txn` with a revision compare to implement optimistic concurrency — an update is `if key's mod-revision == expected then put else fail`. That failure surfaces to your controller as a `409 Conflict` on update, which is exactly why you re-get-and-retry on conflict. etcd's own lease/TTL mechanism backs event TTLs and some ephemeral keys.
- **Disk latency = apiserver latency.** Raft commit requires `fsync` to the WAL. Slow disks (or a noisy-neighbor volume) directly inflate write latency for *every* apiserver mutation. This is a well-documented real-world failure mode, not a theoretical one: OpenAI's cluster-scaling posts explicitly call out that etcd wants dedicated low-latency SSD/local NVMe, not network-attached volumes with variable latency — a noisy-neighbor EBS volume was a real degradation vector they describe hitting. `wal_fsync_duration_seconds` and `backend_commit_duration_seconds` are the p99 metrics that matter. A single DB size cap (default 2Gi, raisable) exists — blow past it and etcd goes read-only (`NOSPACE` alarm) until you compact+defrag. (Deep etcd ops is a later module.)

### kube-scheduler

The scheduler watches for pods with `spec.nodeName == ""` and assigns each one a node by writing a **Binding**. Internally it's a plugin framework running a two-phase cycle.

```
      ┌──────────────── scheduling queue ───────────────┐
      │ activeQ  ─ backoffQ ─ unschedulableQ            │  (pods sorted by priority)
      └──────────────────────────────────────────────────┘
                     │ pop one pod
   ┌──── Scheduling cycle (synchronous, one pod at a time) ────┐
   │ PreFilter → Filter → PostFilter(preemption) →             │
   │ PreScore → Score → Reserve → Permit                        │
   └────────────────────────────────────────────────────────────┘
                     │ node chosen, resources reserved in cache
   ┌──── Binding cycle (can run concurrently) ────┐
   │ WaitOnPermit → PreBind → Bind → PostBind      │
   └────────────────────────────────────────────────┘
```

- **Cache + snapshot.** The scheduler maintains an in-memory cache of node/pod state fed by informers. At the start of each scheduling cycle it takes a **snapshot** of that cache so the whole Filter/Score run sees a consistent view. Because scheduling is single-threaded through the cycle, `Reserve` marks resources as claimed in the cache *before* the (async) Bind actually completes — this "assume" step prevents the next pod from double-booking the same GPU.
- **Framework extension points.** Filter (formerly "predicates") answers *can this pod fit this node?* — `NodeResourcesFit`, `NodeAffinity`, `TaintToleration`, `PodTopologySpread`, `VolumeBinding`. Score (formerly "priorities") ranks the survivors — `NodeResourcesFit` (with `LeastAllocated`/`MostAllocated`, the FinOps-relevant knob for bin-packing), `ImageLocality`, spreading. You configure these via a `KubeSchedulerConfiguration` profile; you can disable/enable plugins or run multiple named profiles.
- **The queue is where throughput lives.** Three sub-queues: **activeQ** (ready to schedule, heap-ordered by priority), **backoffQ** (recently failed, waiting out exponential backoff), **unschedulableQ** (no node fit last time). A pod parked as unschedulable must not be retried blindly — that wastes cycles. **QueueingHints** (a 1.3x scheduler feature) let plugins say "only move this pod back to activeQ when *this kind* of cluster event happens" (e.g. a node added, a PVC bound, a GPU freed), instead of the old "flush everything on any change." For GPU pods pending on capacity, this means they wake precisely when a GPU is released — relevant when your operator reasons about queueing latency as a cost/efficiency signal.
- **Preemption.** If no node passes Filter, `PostFilter` runs preemption: find a node where evicting lower-priority pods would let this one fit, then delete those victims (respecting PDBs where possible) and re-queue the pending pod. `PreemptionPolicy: Never` opts a pod out of preempting others; a `nominatedNodeName` is recorded so the freed space isn't stolen before the pending pod is rescheduled.
- **GPU placement — device plugin vs DRA.** With the classic **device-plugin** model, a GPU is just an opaque extended resource `nvidia.com/gpu: 1`. The node's device plugin advertises a count via the kubelet; the scheduler treats it as an *integer count* in `NodeResourcesFit` — it can't reason about GPU model, memory, MIG partitions, or sharing (time-slicing/MPS), so those get smuggled into node labels + affinity, and fractional/shared GPUs are invisible to the scheduler. **DRA** (Dynamic Resource Allocation, `resource.k8s.io`; core APIs **GA in v1.34**, beta in 1.33) replaces this with a claim model:

  ```
  DeviceClass          (admin: "nvidia-gpu", selects a driver + base constraints)
      ▲
  ResourceClaim(Template) ── pod references ──▶ scheduler DRA plugin allocates from ▼
  ResourceSlice        (driver-published: per-node list of real devices + attributes:
                        model=H100, memory=80Gi, mig-profile, sharable capacity)
  ```

  The DRA plugin runs in Filter/Reserve, allocating specific devices (not a count) against structured attributes, and writes the allocation back into the claim. For a GPU-cost operator this is pivotal — DRA is where accurate per-device attribution (which pod holds which physical/MIG device, and for shared devices what fraction) finally becomes first-class instead of label-guessing. Expect to consume `ResourceClaim`/`ResourceSlice` objects for the cost signal on DRA clusters and fall back to device-plugin counts + labels elsewhere. (Module 09 goes deep on DRA and Kueue; this lesson only needs the scheduler-internals shape.)

### kube-controller-manager

A single binary hosting ~30 built-in controllers (deployment, replicaset, statefulset, daemonset, job, cronjob, node lifecycle, endpoint/endpointslice, service account, namespace, garbage collector, PV/PVC, resourcequota, …). It is the reference implementation of the pattern your operator follows.

- **Shared informer factory.** Rather than each controller opening its own watch, they share informers from one factory. An informer = a **Reflector** (list+watch → delta FIFO) + **indexer** (a thread-safe local cache/store) + event handlers. Result: one watch per resource type feeds many controllers, and every controller reads from a local cache instead of hitting the apiserver on the hot path.

  ```
  apiserver ──list+watch──▶ Reflector ──▶ DeltaFIFO ──pop──▶ (1) update Indexer/Store   (local cache)
                                                            └▶ (2) fire event handlers  ──▶ workqueue (keys)
  workqueue ──▶ worker goroutine: Get(key) from Indexer ──▶ reconcile ──▶ apiserver writes
  ```

  Controllers enqueue *keys* (`namespace/name`), not objects — so re-reads always hit current cache state, and duplicate enqueues collapse. The workqueue is **rate-limited** (per-item exponential backoff + a global token bucket) so a failing reconcile retries with backoff instead of hot-looping. This is **level-triggered**: the handler re-reads current state and drives toward desired state, so a missed/duplicated event is harmless — the opposite of edge-triggered messaging where a lost event is lost work. Your kubebuilder controller is literally this loop; `Reconcile(ctx, req)` is the worker, `req` is the key. (Lesson 04 goes deep on this pipeline; lesson 03 goes deep on why level-triggering is the design choice it is.)
- **Garbage collector.** Uses `ownerReferences` to build an object graph; when an owner is deleted it cascades (foreground/background/orphan) via `finalizers`. Understanding this is how you avoid leaking child objects your operator creates. (Lesson 06 covers this in depth.)
- **Leader election.** KCM runs active-passive in HA: all replicas start, but only the leader reconciles. They contend for a `coordination.k8s.io/Lease` in `kube-system` (e.g. `kube-controller-manager`); the holder renews `renewTime` periodically, and if it stops (crash, GC pause, network partition) another replica acquires the expired lease. This protects against **split-brain**: two active controller-managers both scaling the same Deployment or both deleting the same pod would fight. Your operator needs the identical mechanism — `sigs.k8s.io/controller-runtime` gives it to you via `LeaderElection: true`, which creates a Lease and gates your reconcilers on holding it.

### cloud-controller-manager

Historically the cloud-specific controllers lived *inside* KCM, forcing cloud SDK code into the core binary. CCM split them out so cloud providers ship an **out-of-tree** binary implementing `cloudprovider.Interface`:

- **node controller** — initializes new nodes with provider-specific labels (region/zone, instance type — directly useful for your cost model) and removes Node objects when the backing instance is gone.
- **route controller** — programs cloud VPC routes for pod CIDRs (where the CNI doesn't).
- **service controller** — reconciles `type: LoadBalancer` Services into cloud LBs.

Same informer + reconcile + leader-election pattern; it just calls the AWS/GCP/Azure API as its side effect. The `--cloud-provider` flag on kubelet/KCM is now `external` on modern clusters, deferring to CCM. The split is also a security/blast-radius boundary worth naming explicitly: cloud credentials live in CCM, not KCM, so a compromised or over-permissioned KCM doesn't automatically mean compromised cloud IAM.

### kubelet

The node agent. Its heart is the **syncLoop** — a single event loop that drives every pod on the node toward its spec.

```
syncLoop select {
  configCh:  ← desired pods (apiserver watch, static-pod files, HTTP)
  plegCh:    ← container lifecycle events (a container started/died)
  syncCh:    ← periodic full resync (~1s tick)
  probeCh:   ← liveness/readiness/startup probe results
  housekeeping: ← cleanup of orphaned volumes/pods
}
→ for each pod: computePodActions → syncPod → call CRI to create/kill containers
```

- **PLEG (Pod Lifecycle Event Generator).** The kubelet must notice when a container dies *without* the apiserver telling it. The generic PLEG **relists** all containers from the CRI every ~1s, diffs against its last known state, and emits events (`ContainerStarted`/`ContainerDied`) onto `plegCh`, which wakes syncLoop for the affected pod. If relist stalls past a 3-minute threshold, the kubelet's healthz reports "PLEG is not healthy" and the node flips **NotReady**. This is not a synthetic-only failure: on GPU nodes with heavy container churn (job schedulers rapidly cycling training pods) or a large image pull saturating the CRI socket, PLEG relist genuinely blows past that threshold under real load — a common GPU-fleet failure mode, distinct from "the kubelet crashed." **Evented PLEG** (KEP-3386, beta/off-by-default through the 1.3x line, still shaking out bugs) lets the CRI push events instead of polling, cutting CPU on high-pod-count nodes.
- **CRI boundary.** The kubelet never runs containers itself. It speaks **CRI** (a gRPC contract, `k8s.io/cri-api`) to a runtime — containerd or CRI-O — over a unix socket (`--container-runtime-endpoint`). Two services: `RuntimeService` (`RunPodSandbox`, `CreateContainer`, `StartContainer`, `ContainerStatus`, exec/attach) and `ImageService` (`PullImage`, `ImageStatus`, `RemoveImage`). The kubelet drives the whole pod lifecycle through these RPCs; `crictl` is the same interface for debugging (`crictl ps`, `crictl inspect`). Everything below the socket is the runtime's job — the kubelet only knows the CRI contract, which is why containerd/CRI-O/gVisor are swappable.
- **Pod admission.** Before syncing, the kubelet runs *admit* handlers locally — node resources fit, required OS/topology, no forbidden sysctls. A pod the scheduler placed can still be **rejected** here (status `Rejected`/`OutOfcpu`), because the kubelet has ground truth the scheduler's cache may lag.
- **Probes.** A prober worker per probe polls liveness/readiness/startup; results flow to syncLoop (liveness failure → restart container; readiness → flip pod's Ready condition, which propagates to EndpointSlices).
- **QoS & cgroup enforcement.** The kubelet writes the cgroup hierarchy: `Guaranteed`/`Burstable`/`BestEffort` QoS classes map to cgroup limits/reservations; `--enforce-node-allocatable`, `--kube-reserved`, `--system-reserved` carve the node. Under memory pressure the kubelet's **eviction manager** ranks pods by QoS and usage-over-request and evicts. (Ties directly to the Linux internals module.)
- **Volume manager.** Reconciles the *desired* set of mounted volumes (from pod specs) against the *actual* mounted set, calling CSI plugins to attach/mount. Runs its own loop independent of syncPod, so a stuck mount blocks that pod's `syncPod` without freezing the whole node.
- **Static pods.** The kubelet also reads pod manifests directly from `--pod-manifest-path` (default `/etc/kubernetes/manifests`) with *no* apiserver involvement — this is how the control plane bootstraps (`kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager` run as static pods on the control-plane nodes). The kubelet creates a read-only **mirror pod** in the apiserver so they're visible to `kubectl`, but the source of truth is the file on disk; you can't `kubectl delete` a static pod, only edit its manifest.
- **Where usage metrics come from (capstone-critical).** The kubelet embeds **cAdvisor** to read per-container cgroup counters and exposes them at `/metrics/cadvisor` (raw Prometheus), `/metrics/resource` (the trimmed set `metrics-server` scrapes for `kubectl top`), and the **Summary API** (`/stats/summary`, JSON). This is the ground-truth source of pod/container CPU/memory usage — your cost/efficiency operator's utilization signal ultimately traces here (directly, or via metrics-server / a Prometheus stack). GPU *utilization* is not in cAdvisor; it comes from **DCGM-exporter** (NVIDIA) as a separate scrape — a key seam your operator must join against Node/Pod identity.
- **Node status & heartbeats — two channels.** (1) The **node lease**: the kubelet updates a `Lease` in `kube-node-lease` every ~10s (just bumps `renewTime`) — cheap, this is what node-lifecycle-controller watches to decide a node is dead. (2) The full **NodeStatus** (capacity, allocatable, conditions, images) is written to the Node object only on change or every ~5 min. Splitting these was the fix for heartbeat traffic melting the apiserver on large clusters.

### kube-proxy

kube-proxy turns Service objects into node-local datapath rules. It watches **Services** and **EndpointSlices** and programs the kernel.

- **iptables mode** (long-time default): a chain of DNAT rules per Service; picks a backend via `statistic --probability`. Correct but O(n) rule updates — thousands of Services make `iptables-restore` slow and CPU-heavy.
- **IPVS mode**: uses the kernel's L4 load balancer with a hash table — O(1) lookup, real LB algorithms (rr, lc, …), scales to large Service counts.
- **nftables mode**: the modern replacement for iptables mode (GA around v1.33). Same "just netfilter" deployment story but with the efficient nftables data structures — fixes the iptables scaling wall.
- **eBPF (Cilium) — replacing kube-proxy entirely.** Cilium (and others) can run *kube-proxy-less*: eBPF programs at the socket/XDP layer do Service translation, watching EndpointSlices directly. No kube-proxy pod, lower latency, and it's the direction GPU/AI-native clusters trend for east-west scale.
- **EndpointSlices, not Endpoints.** The datapath is driven by **EndpointSlices** (sharded, ≤100 endpoints each) rather than the old single fat `Endpoints` object, so a 5000-pod Service doesn't rewrite one giant object on every pod change. Readiness (from kubelet probes) gates whether an endpoint appears as `Ready` in the slice.
- **conntrack.** iptables/IPVS DNAT relies on the kernel **conntrack** table to keep a connection pinned to the backend it was first NAT'd to. On big nodes conntrack can fill (`nf_conntrack: table full` → dropped connections), and stale UDP conntrack entries after an endpoint dies are a classic source of DNS/UDP blackholes — eBPF datapaths sidestep much of this by not relying on conntrack the same way.

### CoreDNS

CoreDNS is itself a controller. Its `kubernetes` plugin **watches** Services and EndpointSlices via informers and builds an in-memory map, then answers DNS:

- `A`/`AAAA` for `svc.ns.svc.cluster.local` → ClusterIP; **headless** Services (`clusterIP: None`) return per-pod records straight from EndpointSlices.
- `SRV` for named ports; `PTR` for reverse.
- Config is the **Corefile** (a plugin chain: `kubernetes`, `forward` to upstream, `cache`, `health`). Because it's watch-driven, a Service create is queryable within a watch round-trip — no zone-file reloads.
- **`ndots:5` amplification.** The default pod `resolv.conf` has `ndots:5` and a search-domain list, so a lookup for an external name like `api.stripe.com` (4 dots) is first tried against every cluster search suffix — up to ~4–5 failing queries before the real one. On busy pods this multiplies CoreDNS QPS; the fixes (fully-qualified names ending in `.`, tuned `ndots`, NodeLocal DNSCache) are the standard mitigations, and NodeLocal DNSCache also removes conntrack pressure from per-pod DNS. This is not academic: Zalando's public postmortem of a January 2019 total cluster DNS outage traces the incident to CoreDNS pods being OOM-killed under load (a 100Mi memory limit that was too low for the query volume), compounded by exactly this `ndots:5` amplification driving up query rate.

### Container runtime via CRI

Below the kubelet, **containerd** or **CRI-O** implements the CRI gRPC contract and does the actual container lifecycle:

```
 kubelet ──CRI gRPC (unix socket)──▶ containerd
                                        ├─ ImageService  ─▶ content store + snapshotter (overlayfs/nydus)
                                        └─ RuntimeService ─▶ shim (containerd-shim-runc-v2)
                                                              └─▶ runc  ──▶ [namespaces + cgroups + /dev/nvidia*]
```

- A pod becomes a **sandbox** (the "pause" container holding the network/IPC namespaces and the pod IP from CNI); app containers then join that sandbox's namespaces.
- **Images**: `ImageService` pulls image manifests + layers; a **snapshotter** (overlayfs by default; stargz/nydus for lazy-pull) assembles the layered rootfs. Lazy-pulling snapshotters matter for GPU images (multi-GB CUDA layers) where pull time dominates pod startup.
- **Sandbox-then-CNI-then-containers ordering** matters: the sandbox (pause) container is created first so it owns the network namespace, CNI is invoked to assign the pod IP into *that* namespace, and only then do app containers join it. This is why a CNI failure leaves a pod stuck in `ContainerCreating` with the sandbox up but no app containers.
- containerd hands the OCI runtime spec to a low-level runtime (**runc**, or **gVisor**/**Kata** for isolation) via a **shim** process (`containerd-shim-runc-v2`) that outlives containerd restarts, so the container's lifecycle is decoupled from the containerd daemon's — you can restart containerd without killing running pods. For GPU pods, the NVIDIA container runtime hook wires `/dev/nvidia*` and the CUDA driver mounts into the OCI spec at this layer.

## Perspectives

**Developer/extender perspective.** Every component you'll touch while building the operator — an informer, a webhook, a scheduler plugin — is *itself* just another apiserver client. There is no back door, no shared memory, no privileged internal API. Writing your operator means internalizing "no shared state, only watch," the same constraint kube-controller-manager itself lives under.

**Operator/SRE perspective.** The two heartbeat channels (node Lease vs NodeStatus) and metrics like `workqueue_depth`, `etcd_request_duration_seconds`, and `apiserver_watch_cache_events_dispatched_total` are exactly what you'd page on. A CKA already knows `kubectl get nodes`; what's new is knowing *which* metric explains *why* a node flapped `NotReady` — a PLEG stall, a missed lease renewal, and a genuinely dead kubelet all produce the same symptom on the surface and require different diagnosis and different fixes.

**Hardware/kernel perspective.** CRI → runc → cgroups/namespaces → `/dev/nvidia*` device-cgroup injection is the literal boundary between "the scheduler decided" and "the GPU is usable inside the container." PLEG's ~1s poll of the CRI socket is a userspace polling loop over what is ultimately kernel process/cgroup state — the abstraction stack from "Kubernetes object" to "electrons on a GPU" runs straight through here.

**Economics/FinOps perspective.** Every extra second between "GPU node Ready" and "workload scheduled and running" is idle GPU-hours burned at whatever that instance costs per hour. Watch-cache staleness, PLEG stalls, and slow etcd disk all show up as schedule-latency — which is a cost metric on a GPU fleet, not just a reliability one. This is the thread that ties module 01's cost exporter to module 02's internals: the internals you're learning here are literally where the money leaks when they misbehave.

## Real-world use cases

- **["Scaling Kubernetes to 7,500 nodes"](https://openai.com/index/scaling-kubernetes-to-7500-nodes/) — OpenAI.** Shows real apiserver/etcd sharding (5 apiservers, 5 etcd members) and controller-tuning decisions made specifically to protect the watch path at scale — the production version of this lesson's "apiserver is the only writer to etcd" and APF sections. (2023 vintage — cite as historical engineering case, not current headcount.)
- **["Scaling Kubernetes to 2,500 Nodes"](https://openai.com/index/scaling-kubernetes-to-2500-nodes/) — OpenAI.** The earlier post in the same series; shows the *progression* of bottlenecks (etcd disk latency, cluster-autoscaler balloon pods, DNS load) as node count grows, good grounding for why each internal detail in this lesson eventually matters at scale. (2021 vintage.)
- **["Total DNS outage in Kubernetes cluster" postmortem](https://github.com/zalando-incubator/kubernetes-on-aws/blob/dev/docs/postmortems/jan-2019-dns-outage.md) — Zalando (Jan 2019).** Real incident: CoreDNS pods OOM-killed under load (a 100Mi memory limit too low for the traffic), compounded by `ndots:5` query amplification — a concrete pairing for the CoreDNS section above, with full root-cause and timeline detail.
- **["Migrating Uber's Compute Platform to Kubernetes: A Technical Journey"](https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/) — Uber Engineering.** A real large-scale migration story touching component internals (custom job controllers reading CRDs, custom schedulers) at Uber's scale — a good "extending, not just operating" anchor for the whole module.

## Worked example

Trace `kubectl apply -f pod.yaml` (a single GPU pod) to a running container, naming the internal work — and a realistic latency budget — at each hop:

| Hop | What happens | Rough latency (good conditions) |
|---|---|---|
| 1. kubectl → apiserver | authN (client cert) → authZ (RBAC) → mutating admission (defaulting, sidecar webhooks) → schema validation → validating admission (e.g. a GPU-count `ValidatingAdmissionPolicy`) | ~1–5 ms (webhooks add their own RTT each) |
| 2. apiserver → etcd | Raft quorum commit; the commit revision becomes the pod's `resourceVersion` | ~5–20 ms on good local SSD; **100 ms+** on slow/network-attached disk |
| 3. etcd → watch cache → scheduler | Watch-cache fan-out wakes the scheduler's informer; pod pops off activeQ, Filter/Score/Reserve run against a cache snapshot, Bind POSTs | ~1–2 ms fan-out + ~1–10 ms Filter/Score for a few hundred nodes |
| 4. apiserver → kubelet | kubelet's watch (filtered to `spec.nodeName == self`) delivers the bound pod on `configCh`; local admission checks run | ~1–2 ms watch delivery + local admission |
| 5. kubelet → CRI → containerd | `RunPodSandbox` (pause container + CNI wires the pod IP + NVIDIA device plugin injects the GPU into the device cgroup), image pull, `CreateContainer`/`StartContainer` | **Seconds to tens of seconds** — dominated by pulling a multi-GB CUDA image if not cached/pre-pulled |
| 6. Back up | PLEG relists (~1s poll), sees `ContainerStarted`, wakes syncLoop; probes flip Ready; status flows back to the apiserver → etcd; EndpointSlice/kube-proxy/CoreDNS pick it up if the pod backs a Service | ~1s PLEG poll interval + probe cadence |

The arithmetic that matters: steps 1–4 (the whole control-plane path — auth, admission, etcd commit, scheduling, binding, delivery to the node) typically total **well under 100ms** on a healthy cluster. Step 5 — pulling and starting the container — is where GPU pods actually spend their "time to Running," often seconds to tens of seconds because of image size alone. The implication for a cost/efficiency operator: optimizing control-plane latency (watch-cache tuning, etcd disk, scheduler Score plugin cost) buys you milliseconds; optimizing image pull (pre-pulled images, lazy snapshotters like nydus/stargz, image layer caching) buys you the seconds-to-tens-of-seconds that actually dominate idle-GPU cost on a training cluster with frequent pod churn.

## Practice

Use a `kind` cluster (multi-node so leases/scheduling are non-trivial). Deliverables: a short **"internals observed"** note plus the Node-watch snippet — this snippet is the seed of your operator's cost signal (it must react to nodes joining/leaving, which is where GPU capacity and spend enter/exit). This practice feeds the [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md) deliverable directly.

1. **Node-watch client-go program.** Write a tiny informer that logs Node add/update/delete and prints capacity — specifically any `nvidia.com/gpu` (kind won't have real GPUs; fake it by editing a node's `status.capacity` or use `kwok`). Seed:

   ```go
   factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
   nodeInformer := factory.Core().V1().Nodes().Informer()
   nodeInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
       AddFunc: func(obj interface{}) {
           n := obj.(*v1.Node)
           gpu := n.Status.Capacity["nvidia.com/gpu"]
           log.Printf("ADD node=%s rv=%s gpu=%s", n.Name, n.ResourceVersion, gpu.String())
       },
       UpdateFunc: func(_, obj interface{}) {
           n := obj.(*v1.Node)
           log.Printf("UPD node=%s rv=%s", n.Name, n.ResourceVersion)
       },
       DeleteFunc: func(obj interface{}) { log.Printf("DEL node=%v", obj) },
   })
   stop := make(chan struct{})
   factory.Start(stop)
   cache.WaitForCacheSync(stop, nodeInformer.HasSynced)
   ```
   Note the `resourceVersion` printed on the initial LIST vs subsequent watch events — that's the informer's resync boundary.

2. **Component /metrics.** `kubectl get --raw /metrics | head` (apiserver), and pull the scheduler/controller-manager metrics. Find `apiserver_watch_cache_events_dispatched_total`, `etcd_request_duration_seconds`, `scheduler_scheduling_attempt_duration_seconds`, `workqueue_depth`. Write down which component each came from and what a rising value would tell you.

3. **Scheduler framework config.** Dump the scheduler's `KubeSchedulerConfiguration` (in kind: `docker exec` into the control-plane, read `/etc/kubernetes/manifests/kube-scheduler.yaml` and any referenced config). Identify which Score plugins are enabled and whether `NodeResourcesFit` is `LeastAllocated` or `MostAllocated`.

4. **Leader-election leases.** `kubectl get lease -n kube-system` and `-n kube-node-lease`. Identify the KCM and scheduler leases, `kubectl get lease kube-controller-manager -n kube-system -o yaml`, and watch `renewTime` advance. Then in `kube-node-lease`, watch a node's lease tick ~every 10s — contrast with how rarely the Node object's `status` changes.

5. **Watch-cache / `410 Gone` behavior (stretch).** Start your Node watcher, `kubectl delete/create` a node object a few times, and observe the informer resync. Optionally force staleness by comparing `LIST ?resourceVersion=0` (cache) vs a consistent LIST.

**Acceptance:** `internals-observed.md` (~15 lines) covering: the RV you saw on initial sync vs watch, one metric per component with its meaning, the scheduler Score config, the KCM/scheduler/node lease names + renew cadence — plus the committed Node-watch snippet.

## Common pitfalls

1. **"The scheduler talks to the kubelet directly."** It never does. It writes a `Binding` through the apiserver; the kubelet discovers the assignment via its own watch. There is no scheduler-to-kubelet channel of any kind.
2. **Treating `resourceVersion` as a counter you can do arithmetic on** — diffing it, sorting numerically across resource types, or comparing it across clusters. It's an opaque per-cluster etcd revision; only equality and "resume from here" are meaningful.
3. **Assuming NodeStatus updates are the heartbeat.** The Lease is; NodeStatus is throttled to change-only/~5 min. Confusing the two leads to misreading node-health dashboards (a lease renewing fine while status looks "stale" is normal, not a problem).
4. **Assuming a `NotReady` node means the kubelet crashed.** A PLEG stall (runtime wedged, kubelet process still alive) is a distinct and common cause with a different fix — restart/inspect the container runtime, not the kubelet.
5. **Believing the CCM split-out was just refactoring.** It's a real security/blast-radius boundary: cloud credentials live in CCM, not KCM, and that separation matters for how you scope IAM.

## Self-check

- Walk `kubectl apply` to a running container end to end, naming which component makes a GPU pod fail at each stage if that component is unhealthy. **Answer:** authN/authZ/admission failures at the apiserver reject the write outright (pod never created). An etcd quorum loss blocks the write from committing at all. A scheduler down means the pod sits `Pending` forever (never gets a `Binding`). A dead/partitioned kubelet on the target node means the bound pod never syncs — it stays `Pending` with a `nodeName` set but no container. A wedged container runtime (CRI socket unresponsive) means `syncPod` hangs on `RunPodSandbox`/`CreateContainer`, PLEG relist stalls, and eventually the node goes `NotReady` from PLEG unhealthy, not from the kubelet itself dying. A missing/misconfigured CNI leaves the sandbox up but no pod IP, so the pod is stuck `ContainerCreating` with no app containers ever starting.
- Why does the watch cache exist, and what's the cost/consistency tradeoff of `resourceVersion=0` LIST vs a quorum LIST? **Answer:** The watch cache lets the apiserver serve thousands of client watches (every informer in every kubelet/controller/your operator) from one in-memory buffer fed by a *single* etcd watch per resource — without it, every watcher would drive an independent etcd stream and crush it; it also serves `resourceVersion=0` LISTs cheaply from memory. `resourceVersion=0` reads the watch cache (fast, cheap, possibly a few hundred ms stale); a quorum LIST (no RV, or an explicit consistency requirement) forces a fresh etcd read (consistent, but expensive and slower under load). Use `resourceVersion=0` for most controller reads; reserve quorum reads for the rare case where staleness would be actively wrong (e.g. a leader-election-adjacent decision).
- What's the practical difference between the node Lease and NodeStatus, and why were they split? **Answer:** The Lease (in `kube-node-lease`) is a cheap ~10s heartbeat — just a `renewTime` bump — that node-lifecycle-controller watches to decide liveness. NodeStatus (capacity, allocatable, conditions, images) is the heavy payload, written to the Node object only on change or every ~5 min. They were split because frequent heartbeats via the full NodeStatus object were melting the apiserver/etcd on large clusters — you don't need to rewrite a multi-KB status object every 10 seconds just to say "I'm alive."
- Given `PLEG is not healthy` in logs, what are 3 concrete production causes, and how do you tell them apart from `kubectl describe node` / metrics? **Answer:** (1) An overloaded or wedged container runtime (containerd/CRI-O) — check `crictl ps`/`crictl info` responsiveness and runtime logs/CPU. (2) Very high container churn on the node (rapid pod cycling from a job scheduler) overwhelming the ~1s relist — check pod churn rate and node pod count against the node's capacity. (3) A large/slow image pull saturating the CRI socket or disk I/O — check `ImageService` pull duration metrics and disk saturation. `kubectl describe node` shows the `PIDPressure`/`Ready` condition and its message/reason; `kubelet_pleg_relist_duration_seconds` and `kubelet_pleg_relist_interval_seconds` distinguish "relist is slow" (runtime/CRI problem) from "relist isn't running at all" (kubelet process problem) — the latter is rare and a different fix entirely.

## Connections & what's next

This lesson is the machinery your capstone operator sits on top of: leader election here is `controller-runtime`'s `LeaderElection: true` in lesson 06; the watch cache and `resourceVersion` here become the informer/reflector pipeline in lesson 04; the scheduler's device-plugin-vs-DRA split resurfaces as the GPU-scheduling literacy lesson (09). The economics thread — schedule latency and image-pull time as idle-GPU cost — is the same thread module 01's `gpu-cost-exporter` started and module 11 (GPU cost economics) finishes.

Next: **[02.2 · API machinery](02-api-machinery.md)** takes the "everything is a client of the apiserver" fact from this lesson and goes one layer deeper — *how* an object is addressed (GVK/GVR), typed, and serialized underneath every client and CRD you'll build.

## References & further reading

**Primary sources**
- [Kubernetes cluster architecture docs](https://kubernetes.io/docs/concepts/architecture/) — canonical component map; read for the exact responsibility split (esp. CCM vs KCM) before diving into internals.
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/) — read for the authoritative Filter/Score/Reserve/Permit/Bind extension-point contract.
- [k/k source](https://github.com/kubernetes/kubernetes) — read `staging/src/k8s.io/apiserver` (watch cache in `.../storage/cacher`), `pkg/scheduler` (framework in `pkg/scheduler/framework`), `pkg/kubelet` (`kubelet.go` syncLoop, `pkg/kubelet/pleg`) to make the internals concrete.

**Real-world engineering blogs**
- OpenAI, ["Scaling Kubernetes to 7,500 nodes"](https://openai.com/index/scaling-kubernetes-to-7500-nodes/) — apiserver/etcd sharding and controller-tuning at scale.
- OpenAI, ["Scaling Kubernetes to 2,500 Nodes"](https://openai.com/index/scaling-kubernetes-to-2500-nodes/) — earlier bottleneck progression (etcd disk, autoscaler, DNS).
- Zalando, ["Total DNS outage in Kubernetes cluster" postmortem](https://github.com/zalando-incubator/kubernetes-on-aws/blob/dev/docs/postmortems/jan-2019-dns-outage.md) — real CoreDNS OOM + ndots amplification incident.
- Uber Engineering, ["Migrating Uber's Compute Platform to Kubernetes: A Technical Journey"](https://www.uber.com/blog/migrating-ubers-compute-platform-to-kubernetes-a-technical-journey/) — large-scale migration touching component internals.

**Deeper dives**
- Red Hat Developer, ["Pod Lifecycle Event Generator: Understanding the 'PLEG is not healthy' issue"](https://developers.redhat.com/blog/2019/11/13/pod-lifecycle-event-generator-understanding-the-pleg-is-not-healthy-issue-in-kubernetes) — vendor engineering explainer of the exact failure mode above.
- ["what-happens-when-k8s"](https://github.com/jamiehannaford/what-happens-when-k8s) — community deep-dive tracing `kubectl apply` end to end, complementary to the Worked example.
- "Programming Kubernetes" (Hausenblas & Schimanski, O'Reilly) — the API-machinery/controller chapters; concepts are current even though sample code has aged.
