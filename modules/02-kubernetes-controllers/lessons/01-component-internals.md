---
lesson: "02.1"
title: "Kubernetes component internals — how each component works"
module: "02"
concept: "Kubernetes component internals"
status: not-started
est_time: "12h"
artifacts: []
---

# 02.1 · Kubernetes component internals — how each component works

> **Concept.** The internal machinery of each control-plane and node component — request flow, caches, control loops, consensus — the layer beneath what a CKA operator already knows.
>
> Module: [⚙️ 02 — Kubernetes internals and controllers](../README.md) · Deliverable: [`gpu-cost-operator`](../practice/gpu-cost-operator/README.md)

## Why this matters

Your capstone operator is a client of these internals, not a bystander to them. It will hold a `watch` against the apiserver's **watch cache** and must survive reconnects without dropping GPU-node events — so `resourceVersion` semantics are load-bearing, not trivia. It will run HA behind **leader election** for the exact reason kube-controller-manager does. And to reason about GPU placement cost you have to know how the **scheduler** actually binds a pod — device-plugin integer counts vs DRA claims change what "a GPU node is full" even means. This lesson is the machinery under CKA-level knowledge: how each component works inside, so you can extend and debug at senior level.

## From operating to extending

| Component | What you know as a CKA operator | The internal you must now know |
|---|---|---|
| kube-apiserver | The REST front door; everything goes through it | The request pipeline (authN→authZ→admission→validation→etcd), the watch cache, and that it is the *only* writer to etcd |
| etcd | The datastore; keep it healthy and backed up | Raft quorum, MVCC revisions as the source of `resourceVersion`, compaction vs defrag, disk latency → apiserver latency |
| kube-scheduler | Assigns pods to nodes; respects taints/affinity | The scheduling framework cycle (queue→Filter→Score→Reserve→Permit→Bind), the node snapshot, preemption, device-plugin vs DRA |
| kube-controller-manager | Runs the built-in controllers | Shared informer factory + workqueues, leader election, level-triggered reconcile — the pattern your operator copies |
| cloud-controller-manager | Talks to the cloud for LBs/routes/nodes | The `cloudprovider.Interface`, and *why* it was split out of KCM |
| kubelet | Runs pods on the node; reports status | The syncLoop, PLEG, the CRI gRPC boundary, pod admission, node lease vs NodeStatus |
| kube-proxy | Implements Services | How EndpointSlices drive iptables/IPVS/nftables rules — and eBPF (Cilium) replacing it entirely |
| CoreDNS | Cluster DNS | It's a controller too: watches Services/EndpointSlices to build its answers |
| CRI runtime | containerd/CRI-O runs containers | The CRI gRPC contract (RuntimeService/ImageService) and snapshotter/image layers |

## Core notes

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
- **API Priority and Fairness (APF).** Requests are classified by `FlowSchema` into `PriorityLevelConfiguration`s, each with a share of concurrent seats and fair-queuing across flows. This replaced the old global `--max-inflight` knobs: a chatty controller (or a busted one hot-looping LISTs) is throttled within its priority level instead of starving `leader-election` or `system` traffic. When your operator gets `429 Too Many Requests`, this is why — respect the `Retry-After`. LISTs are expensive "seats"; prefer watches and cached (`resourceVersion=0`) reads to stay cheap.

### etcd

etcd is a strongly-consistent KV store; the apiserver is its only meaningful client.

- **Raft + quorum.** A cluster of (usually) 3 or 5 members elects a leader; a write commits only after a **majority** (quorum) persists it to the Raft log. 3 members tolerate 1 failure, 5 tolerate 2. Even member counts are pointless (4 still only tolerates 1) and hurt latency.
- **MVCC revisions.** Every write bumps a global **revision**; old values are kept until compaction. This is the mechanism behind `resourceVersion` and behind watches ("give me changes since revision N").
- **Watch streams.** etcd streams change events from a revision forward. The apiserver's watch cache is fed by exactly one such stream per resource; this is the ultimate origin of every informer event in the cluster.
- **Compaction vs defragmentation.** *Compaction* discards MVCC history older than a revision (frees logical space, drops old RVs → source of `410 Gone`). *Defragmentation* returns the freed pages to the filesystem (shrinks the actual DB file) and briefly blocks the member — run it rolling, one member at a time.
- **Transactions & leases (etcd's, not k8s Lease objects).** The apiserver uses etcd `Txn` with a revision compare to implement optimistic concurrency — an update is `if key's mod-revision == expected then put else fail`. That failure surfaces to your controller as a `409 Conflict` on update, which is exactly why you re-get-and-retry on conflict. etcd's own lease/TTL mechanism backs event TTLs and some ephemeral keys.
- **Disk latency = apiserver latency.** Raft commit requires `fsync` to the WAL. Slow disks (or a noisy-neighbor volume) directly inflate write latency for *every* apiserver mutation. etcd wants low-latency SSD; `wal_fsync_duration_seconds` and `backend_commit_duration_seconds` are the p99 metrics that matter. A single DB size cap (default 2Gi, raisable) exists — blow past it and etcd goes read-only (`NOSPACE` alarm) until you compact+defrag. (Deep etcd ops is a later module.)

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

  The DRA plugin runs in Filter/Reserve, allocating specific devices (not a count) against structured attributes, and writes the allocation back into the claim. For a GPU-cost operator this is pivotal — DRA is where accurate per-device attribution (which pod holds which physical/MIG device, and for shared devices what fraction) finally becomes first-class instead of label-guessing. Expect to consume `ResourceClaim`/`ResourceSlice` objects for the cost signal on DRA clusters and fall back to device-plugin counts + labels elsewhere.

### kube-controller-manager

A single binary hosting ~30 built-in controllers (deployment, replicaset, statefulset, daemonset, job, cronjob, node lifecycle, endpoint/endpointslice, service account, namespace, garbage collector, PV/PVC, resourcequota, …). It is the reference implementation of the pattern your operator follows.

- **Shared informer factory.** Rather than each controller opening its own watch, they share informers from one factory. An informer = a **Reflector** (list+watch → delta FIFO) + **indexer** (a thread-safe local cache/store) + event handlers. Result: one watch per resource type feeds many controllers, and every controller reads from a local cache instead of hitting the apiserver on the hot path.

  ```
  apiserver ──list+watch──▶ Reflector ──▶ DeltaFIFO ──pop──▶ (1) update Indexer/Store   (local cache)
                                                            └▶ (2) fire event handlers  ──▶ workqueue (keys)
  workqueue ──▶ worker goroutine: Get(key) from Indexer ──▶ reconcile ──▶ apiserver writes
  ```

  Controllers enqueue *keys* (`namespace/name`), not objects — so re-reads always hit current cache state, and duplicate enqueues collapse. The workqueue is **rate-limited** (per-item exponential backoff + a global token bucket) so a failing reconcile retries with backoff instead of hot-looping. This is **level-triggered**: the handler re-reads current state and drives toward desired state, so a missed/duplicated event is harmless — the opposite of edge-triggered messaging where a lost event is lost work. Your kubebuilder controller is literally this loop; `Reconcile(ctx, req)` is the worker, `req` is the key.
- **Garbage collector.** Uses `ownerReferences` to build an object graph; when an owner is deleted it cascades (foreground/background/orphan) via `finalizers`. Understanding this is how you avoid leaking child objects your operator creates.
- **Leader election.** KCM runs active-passive in HA: all replicas start, but only the leader reconciles. They contend for a `coordination.k8s.io/Lease` in `kube-system` (e.g. `kube-controller-manager`); the holder renews `renewTime` periodically, and if it stops (crash, GC pause, network partition) another replica acquires the expired lease. This protects against **split-brain**: two active controller-managers both scaling the same Deployment or both deleting the same pod would fight. Your operator needs the identical mechanism — `sigs.k8s.io/controller-runtime` gives it to you via `LeaderElection: true`, which creates a Lease and gates your reconcilers on holding it.

### cloud-controller-manager

Historically the cloud-specific controllers lived *inside* KCM, forcing cloud SDK code into the core binary. CCM split them out so cloud providers ship an **out-of-tree** binary implementing `cloudprovider.Interface`:

- **node controller** — initializes new nodes with provider-specific labels (region/zone, instance type — directly useful for your cost model) and removes Node objects when the backing instance is gone.
- **route controller** — programs cloud VPC routes for pod CIDRs (where the CNI doesn't).
- **service controller** — reconciles `type: LoadBalancer` Services into cloud LBs.

Same informer + reconcile + leader-election pattern; it just calls the AWS/GCP/Azure API as its side effect. The `--cloud-provider` flag on kubelet/KCM is now `external` on modern clusters, deferring to CCM.

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

- **PLEG (Pod Lifecycle Event Generator).** The kubelet must notice when a container dies *without* the apiserver telling it. The generic PLEG **relists** all containers from the CRI every ~1s, diffs against its last known state, and emits events (`ContainerStarted`/`ContainerDied`) onto `plegCh`, which wakes syncLoop for the affected pod. If relist stalls past a 3-minute threshold, the kubelet's healthz reports "PLEG is not healthy" and the node flips **NotReady** — a classic symptom of a wedged container runtime. **Evented PLEG** (KEP-3386, beta/off-by-default through the 1.3x line, still shaking out bugs) lets the CRI push events instead of polling, cutting CPU on high-pod-count nodes.
- **CRI boundary.** The kubelet never runs containers itself. It speaks **CRI** (a gRPC contract, `k8s.io/cri-api`) to a runtime — containerd or CRI-O — over a unix socket (`--container-runtime-endpoint`). Two services: `RuntimeService` (`RunPodSandbox`, `CreateContainer`, `StartContainer`, `ContainerStatus`, exec/attach) and `ImageService` (`PullImage`, `ImageStatus`, `RemoveImage`). The kubelet drives the whole pod lifecycle through these RPCs; `crictl` is the same interface for debugging (`crictl ps`, `crictl inspect`). Everything below the socket is the runtime's job — the kubelet only knows the CRI contract, which is why containerd/CRI-O/gVisor are swappable.
- **Pod admission.** Before syncing, the kubelet runs *admit* handlers locally — node resources fit, required OS/topology, no forbidden sysctls. A pod the scheduler placed can still be **rejected** here (status `Rejected`/`OutOfcpu`), because the kubelet has ground truth the scheduler's cache may lag.
- **Probes.** A prober worker per probe polls liveness/readiness/startup; results flow to syncLoop (liveness failure → restart container; readiness → flip pod's Ready condition, which propagates to EndpointSlices).
- **QoS & cgroup enforcement.** The kubelet writes the cgroup hierarchy: `Guaranteed`/`Burstable`/`BestEffort` QoS classes map to cgroup limits/reservations; `--enforce-node-allocatable`, `--kube-reserved`, `--system-reserved` carve the node. Under memory pressure the kubelet's **eviction manager** ranks pods by QoS and usage-over-request and evicts. (Ties directly to the Linux cgroups module.)
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
- **`ndots:5` amplification.** The default pod `resolv.conf` has `ndots:5` and a search-domain list, so a lookup for an external name like `api.stripe.com` (4 dots) is first tried against every cluster search suffix — up to ~4–5 failing queries before the real one. On busy pods this multiplies CoreDNS QPS; the fixes (fully-qualified names ending in `.`, tuned `ndots`, NodeLocal DNSCache) are the standard mitigations, and NodeLocal DNSCache also removes conntrack pressure from per-pod DNS.

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

## Worked example

Trace `kubectl apply -f pod.yaml` (a single GPU pod) to a running container, naming the internal work at each hop:

1. **kubectl → apiserver.** `kubectl` POSTs the pod to `/api/v1/namespaces/default/pods`. The request walks the pipeline: **authN** (your kubeconfig client cert) → **authZ** (RBAC allows `create pods`) → **mutating admission** (defaulting, any sidecar/limitrange webhooks) → **schema validation** → **validating admission** (e.g. a `ValidatingAdmissionPolicy` capping GPU count). On success the apiserver serializes to protobuf and issues an etcd write.
2. **apiserver → etcd.** etcd commits the pod at a new **revision** once a **Raft quorum** fsyncs it. That revision becomes the pod's `resourceVersion`. The pod now exists with `spec.nodeName == ""`, `phase: Pending`.
3. **etcd → apiserver watch cache → scheduler.** The write flows out etcd's watch stream into the apiserver's **watch cache**, which fans it to the scheduler's informer. The scheduler pops the pod off its **activeQ**, snapshots node state, runs **Filter** (which nodes have a free `nvidia.com/gpu`, tolerate the GPU taint, satisfy affinity) → **Score** (bin-pack via `MostAllocated` if you tuned for cost) → **Reserve** (mark the GPU claimed in-cache) → **Bind** (POST a `Binding` setting `nodeName`). The apiserver writes `spec.nodeName` back to etcd.
4. **apiserver → kubelet.** The kubelet on the chosen node has a watch filtered to `spec.nodeName == self`; the bound pod arrives on its **configCh**. syncLoop runs **admission** (GPU actually present? node not under eviction pressure?), then `syncPod`.
5. **kubelet → CRI → containerd.** `syncPod` calls **CRI** `RunPodSandbox` (pause container + CNI wires the pod IP + the NVIDIA device plugin injects the GPU into the container's device cgroup), pulls the image via `ImageService` (snapshotter builds the rootfs), then `CreateContainer`/`StartContainer`. containerd hands an OCI spec to **runc** via a shim; the process starts.
6. **Back up.** **PLEG** relists, sees `ContainerStarted`, wakes syncLoop; probes flip **Ready**; the kubelet writes pod status (`Running`, container ready) to the apiserver → etcd. The endpointslice controller sees Ready and (if the pod backs a Service) adds it to an **EndpointSlice**; kube-proxy/CoreDNS pick that up. Container is live and routable.

## Practice

Use a `kind` cluster (multi-node so leases/scheduling are non-trivial). Deliverables: a short **"internals observed"** note plus the Node-watch snippet — this snippet is the seed of your operator's cost signal (it must react to nodes joining/leaving, which is where GPU capacity and spend enter/exit).

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

## Self-check

**(a) Why is the apiserver watch cache important, and what does `resourceVersion` guarantee across a watch reconnect?**

**Answer:** The watch cache lets the apiserver serve thousands of client watches (every informer in every kubelet/controller/your operator) from one in-memory buffer fed by a *single* etcd watch per resource — without it, every watcher would drive an independent etcd stream and crush it; it also serves `resourceVersion=0` LISTs cheaply from memory. `resourceVersion` is the etcd MVCC revision. Across a reconnect the client resumes with "send events after RV=N," so it won't miss or re-process changes *as long as N still exists*. It guarantees ordering and no-gaps from that point — but **not** infinite history: if N has been compacted out, the apiserver returns `410 Gone` and the informer must relist (full LIST → new RV) then resume. So the guarantee is "consistent resumption or an explicit signal to resync," never a silent gap.

**(b) What does leader election protect against in kube-controller-manager, and why will your operator need it in HA?**

**Answer:** It prevents split-brain: with multiple KCM replicas running for availability, two active instances would both act on the same objects — double-scaling a Deployment, both deleting the same pod, racing on status — producing flapping and corruption. Leader election makes them contend for a `Lease`; only the holder reconciles, and on the leader's death (crash/partition/long GC pause letting the lease expire) another replica takes over. Your operator is the same shape — a set of reconcile loops writing to shared objects — so HA replicas must be active-passive. controller-runtime's `LeaderElection: true` creates the Lease and gates your reconcilers on holding it; without it, two replicas of your GPU-cost operator would fight over the resources they manage.

**(c) Trace what the kubelet's syncLoop and PLEG do when a container dies unexpectedly.**

**Answer:** The apiserver doesn't know the container died — the node has to detect it. **PLEG** relists all containers from the CRI (~every 1s), diffs against last-known state, sees the container gone, and emits a `ContainerDied` event onto `plegCh`. That wakes **syncLoop** for that pod; it calls `computePodActions`, which — per the pod's `restartPolicy` and the container's backoff — decides to recreate the container and calls CRI `CreateContainer`/`StartContainer` (with `CrashLoopBackOff` delay if it keeps dying). The kubelet then writes updated container status (restart count, last state) up to the apiserver. If the runtime is wedged so relist can't complete within ~3 min, the kubelet reports "PLEG is not healthy" and the node goes **NotReady**, triggering node-controller eviction. (Evented PLEG would replace the ~1s poll with runtime-pushed CRI events, but it's still off by default in the 1.3x line.)

## Resources

1. **Kubernetes Components** + **Cluster Architecture** — https://kubernetes.io/docs/concepts/overview/components/ and https://kubernetes.io/docs/concepts/architecture/ · *Skim* — you know the boxes; use it only to fix the exact responsibility split (esp. CCM vs KCM). *Why:* the canonical map before diving under it.
2. **k/k source** — https://github.com/kubernetes/kubernetes · *Deep* — apiserver internals in `staging/src/k8s.io/apiserver` (watch cache in `.../storage/cacher`), scheduler in `pkg/scheduler` (framework in `pkg/scheduler/framework`), kubelet in `pkg/kubelet` (`kubelet.go` syncLoop, `pkg/kubelet/pleg`). *Why:* the internals are the code; reading `cacher.go` and `pleg/generic.go` once makes this lesson concrete.
3. **Component concept docs** — kube-apiserver / kube-scheduler / kubelet reference: https://kubernetes.io/docs/concepts/architecture/ (control plane) + Scheduling Framework https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/ and DRA https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/ · *Deep* on scheduling/DRA (capstone-critical), *skim* the rest. *Why:* current, authoritative behavior of the pieces your operator touches.
4. **"Programming Kubernetes"** (Hausenblas & Schimanski, O'Reilly) — informers, workqueues, the client-go/controller pattern · *Deep* (Ch. on API Machinery & controllers). Pair with the **"what happens when you create a pod / kubectl apply"** deep-dive: https://github.com/jamiehannaford/what-happens-when-k8s · *Why:* the end-to-end request trace, in more detail than the Worked Example.
