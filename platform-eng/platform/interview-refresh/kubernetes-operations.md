---
area: "Kubernetes operations"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 Kubernetes operations — interview refresh

> Day-2 ops, upgrades, multi-cluster, troubleshooting — the CKA-and-beyond operational layer.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

Day-2 questions are graded on **specificity of mechanism and specificity of numbers**.
"Check the events" is a junior answer; "the kubelet posts status every 10 s, the node
controller marks it `NotReady` after a 50 s grace period, and the default 300 s
`NoExecute` toleration means pods start evacuating five minutes later" is a senior one.
This page is mostly those numbers, with the mechanism each one comes from.

Everything version-specific below was verified against upstream source read on
2026-08-18: `kubernetes/website` (`releases/version-skew-policy.md`,
`node-pressure-eviction.md`, `scheduling-gpus.md`, `configure-service-account.md`),
`kubernetes/kubernetes` (`pkg/features/kube_features.go`, `pkg/kubelet/qos/policy.go`,
`pkg/controller/nodelifecycle/…/defaults.go`, apiserver storage defaults),
`kubernetes-sigs/gateway-api` (`README.md`), `etcd-io/etcd` (`server/embed/config.go`)
and `NVIDIA/k8s-device-plugin` (`README.md`). `kubernetes.io` itself is egress-blocked
here, so the rendered docs were **not** read — the repo content behind them was. The
Kubernetes docs site currently ships v1.36 as latest, with v1.35 and v1.34 also supported.

## Talking points to have ready

### 1. Upgrades and version skew — the numbers, not the vibe

**Correct your recall if it says "N-2 kubelet skew".** Since 1.25 the kubelet may be up
to **three** minor versions older than the API server (it was two before that).

| Component | Allowed skew vs `kube-apiserver` |
|---|---|
| `kube-apiserver` (other HA instances) | within **1** minor of the newest apiserver |
| `kubelet` | up to **3** minors older; never newer |
| `kube-proxy` | up to **3** minors older; never newer; also within ±3 of its own node's kubelet |
| `kube-controller-manager`, `kube-scheduler`, `cloud-controller-manager` | up to **1** minor older; never newer |
| `kubectl` | **±1** minor (may be one *newer*) |

```
  Upgrade order — the skew rules force it, nothing else does
  ────────────────────────────────────────────────────────────────────
  t0   [apiserver 1.35]  [c-m/sched 1.35]  [kubelets 1.34/1.33]
        │
        │ ① apiserver first, one minor at a time (skipping minors is
        │   forbidden by the API deprecation policy, even single-instance)
        ▼
  t1   [apiserver 1.36]  [c-m/sched 1.35]  [kubelets 1.34/1.33]
        │                  ▲ still legal: ≤1 minor older
        │ ② controller-manager, scheduler, CCM — any order, or together
        ▼
  t2   [apiserver 1.36]  [c-m/sched 1.36]  [kubelets 1.34/1.33]
        │                                    ▲ still legal: ≤3 minors older
        │ ③ kubelets, node by node, drain first
        │   (in-place minor kubelet upgrades are NOT supported)
        ▼
  t3   [apiserver 1.36]  [c-m/sched 1.36]  [kubelets 1.36]

  Trap: kubelets persistently 3 minors behind BLOCK the next control-plane
        upgrade — you must move nodes before you can move the apiserver again.
```

Two more upgrade facts worth having: registered admission webhooks must be able to handle
the new API versions the new apiserver will send them (or use `matchPolicy: Equivalent`),
and the project recommends going to the latest patch of your current minor first, then to
the latest patch of the target minor.

**Node upgrade mechanics.** `kubectl cordon` sets `spec.unschedulable`, which only stops
*new* pods. `kubectl drain` cordons and then evicts, using the **Eviction API**, which is
the only path that respects PodDisruptionBudgets. Flags you will be asked about:
`--ignore-daemonsets` (DaemonSet pods are not evictable — they'd be recreated
immediately), `--delete-emptydir-data` (drain refuses to destroy emptyDir contents
otherwise), `--grace-period`, `--force` (for bare pods with no controller). A PDB with
`minAvailable` equal to the current replica count will block a drain **forever** — the
eviction call returns 429 and drain retries; that is the single most common "the upgrade
is stuck" cause.

```console
# cordon: spec.unschedulable=true — stops NEW pods only, existing ones stay
$ kubectl cordon gpu-node-17

# drain: cordon + evict via the Eviction API (the only path that honours PDBs)
#   --ignore-daemonsets    DaemonSet pods are not evictable; they'd be recreated at once
#   --delete-emptydir-data drain refuses to destroy emptyDir contents without it
$ kubectl drain gpu-node-17 --ignore-daemonsets --delete-emptydir-data \
    --grace-period=120 --timeout=30m
evicting pod ml/inference-7c9f-b4k2p
error when evicting pod "inference-7c9f-b4k2p" (will retry after 5s):
  Cannot evict pod as it would violate the pod's disruption budget.
```

That last line is the PDB doing its job — and, if the budget can never be satisfied, the
reason your upgrade never finishes:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: inference, namespace: ml}
spec:
  minAvailable: 3                 # with replicas: 3 this blocks every voluntary eviction
  selector:
    matchLabels: {app: inference}
  unhealthyPodEvictionPolicy: AlwaysAllow   # let already-broken pods be evicted
```

Surge upgrades (new node joins, then old node drains) cost extra capacity
but avoid the window where you are below quorum; in-place node upgrades are cheaper and
riskier.

**The GPU version of this.** You cannot drain a node running a 500-GPU synchronous
training job without killing the job — every rank fails when one rank disappears. The
sequence that works: cordon the target nodes so no new job lands, wait for the current job
to reach a checkpoint boundary (or signal it to checkpoint), let the job exit or migrate,
then drain. If the job's framework supports elastic scheduling and re-rendezvous, you can
drain one node and let it recover, but you pay the restart cost of the whole step. Budget
GPU node upgrades against the job length: on a fleet where jobs run for days, upgrade
windows are a scheduling problem, not a maintenance task.

### 2. Requests, limits, QoS — and the eviction ranking people get wrong

- **`requests`** feed the scheduler and set cgroup shares/weights; **`limits`** are
  enforced at runtime: CPU by throttling (CFS quota), memory by the kernel OOM killer.
  Nothing throttles memory; you either fit or you die with exit code **137** (128+9,
  SIGKILL).
- **QoS classes.** `Guaranteed` = every container has requests == limits for both CPU and
  memory. `BestEffort` = no requests or limits anywhere. `Burstable` = everything else.
- **The correction most candidates need:** the kubelet does **not** rank node-pressure
  evictions by QoS class. Per the node-pressure eviction docs, it ranks by (1) whether the
  pod's usage exceeds its **requests**, (2) **Pod Priority**, and (3) how much usage
  exceeds requests. QoS class is a useful *estimate* of the outcome, not the input. So a
  `Burstable` pod sitting under its requests is evicted after a `Guaranteed`-looking pod
  that is over — and Priority beats both.
- **The kernel-level knob** is `oom_score_adj`, set by the kubelet
  (`pkg/kubelet/qos/policy.go`): `Guaranteed` → **−997**, `BestEffort` → **1000**,
  `Burstable` → `1000 − (1000 × memory_request / node_memory_capacity)`. Read that
  formula: a Burstable container requesting 10% of node memory gets ~900, so it is a prime
  OOM target the moment the node is tight — while a container requesting most of the node
  approaches the Guaranteed end.
- **Default hard eviction thresholds** (kubelet, Linux): `memory.available<100Mi`,
  `nodefs.available<10%`, `imagefs.available<15%`, `imagefs.inodesFree<5%`. The
  eviction-pressure transition period defaults to **5 m** — that is the anti-flap delay
  before a node reports pressure cleared, and it explains why a node stays
  `MemoryPressure=True` for minutes after you freed memory.
- **Scheduling controls**, in order of bluntness: `nodeSelector` → node affinity (soft or
  hard) → taints/tolerations (repel by default; the node opts out of workloads) →
  `topologySpreadConstraints` (spread across zones/nodes with `maxSkew`) → `PriorityClass`
  + preemption (a high-priority pending pod evicts lower-priority pods to fit). Preemption
  is the one that surprises people: it can kill a running job to place a pending one, and
  on a GPU fleet that is a very expensive event.

### 3. The day-2 decision trees

```
 POD PENDING ─┬─▶ kubectl describe pod → Events
              │
              ├─ "0/N nodes are available: insufficient nvidia.com/gpu"
              │       → capacity: is there any free GPU? cordoned nodes?
              │         device plugin healthy on the node (kubectl describe node → Capacity)?
              ├─ "…had untolerated taint {gpu=true: NoSchedule}"
              │       → tolerations missing on the pod
              ├─ "…didn't match Pod's node affinity/selector"
              │       → labels absent (feature-discovery not running?) or wrong selector
              ├─ "exceeded quota" / no event at all
              │       → ResourceQuota in the namespace, or a validating webhook rejecting
              ├─ "pod has unbound immediate PersistentVolumeClaims"
              │       → storage class / zone mismatch with the node it would fit on
              └─ scheduler not running / gated
                      → schedulingGates, or a queueing controller (Kueue) holding it

 CRASHLOOP ───┬─ exit 137  → SIGKILL: OOMKilled (check lastState.terminated.reason)
              │              or liveness probe kill after failureThreshold
              ├─ exit 143  → SIGTERM: shut down by something upstream
              ├─ exit 1/2  → application error; kubectl logs --previous
              └─ exit 0 in a loop → command exits immediately; wrong entrypoint/args

 NODE NotReady ─── timeline that decides the blast radius
     t+0s   kubelet stops posting status (--node-status-update-frequency = 10s)
     t+50s  node controller flips Ready=Unknown (NodeMonitorGracePeriod = 50s)
            → NoExecute taints node.kubernetes.io/unreachable / not-ready
     t+350s pods evicted: default tolerationSeconds = 300 on those taints
            (injected by the DefaultTolerationSeconds admission plugin)
     causes: kubelet dead · container runtime dead · CNI plugin failure ·
             disk full (imagefs) · PLEG "not healthy" (runtime too slow to list pods) ·
             network partition to the apiserver

 INTERMITTENT 5xx, pods look healthy
     ├─ readiness probe too permissive → traffic to a pod that isn't ready
     ├─ conntrack table full on the node (nf_conntrack_max) → dropped flows,
     │     shared per-node with no per-pod quota
     ├─ DNS: ndots:5 means "svc" expands through several search domains first;
     │     an external name costs 4–5 lookups, and a saturated CoreDNS shows as
     │     sporadic timeouts, not as DNS errors
     ├─ MTU mismatch (overlay encapsulation) → large responses hang, small ones fine
     └─ terminationGracePeriod vs endpoint removal race → in-flight requests to a
           pod that already stopped listening (fix: preStop sleep, graceful shutdown)
```

Node-level moves to name, not just `kubectl describe`: `journalctl -u kubelet`,
`crictl ps -a` / `crictl logs`, `dmesg -T | grep -i oom`, `conntrack -S`, `ip -s link`,
`nsenter -t <pid> -n ss -tanp`, and `kubectl get --raw /api/v1/nodes/<node>/proxy/stats/summary`
for the kubelet's own view of resource usage.

### 4. etcd and control-plane health

- **Quorum is ⌈(n+1)/2⌉.** 3 members tolerate 1 failure, 5 tolerate 2, 7 tolerate 3.
  Even sizes are strictly worse than the odd size below them (4 tolerates 1, same as 3,
  with more write latency), which is why cluster sizes are odd. Every write is a Raft
  round trip fsynced to disk on a quorum of members — **etcd is latency-bound on disk
  fsync and network RTT**, which is why you put it on local NVMe and never stretch a
  single etcd cluster across regions.
- **Space management is two different operations.** *Compaction* discards old
  revisions — the kube-apiserver issues it automatically every **5 minutes** by default
  (`--etcd-compaction-interval`, `DefaultCompactInterval = 5 * time.Minute`), while etcd's
  own `--auto-compaction-retention` defaults to `"0"` (off). *Defragmentation*
  (`etcdctl defrag`) is what actually returns freed pages to the filesystem, and it blocks
  the member while it runs — do it one member at a time, never all at once.
- **`--quota-backend-bytes` defaults to 2 GiB.** Exceed it and etcd raises an alarm and
  goes **read-only**: the cluster keeps serving reads while every write fails. Recovery is
  compact → defrag → `etcdctl alarm disarm`. Knowing that the failure mode is "read-only,
  not down" is the kind of detail interviewers use to separate people who have seen it.
- **Backups.** `etcdctl snapshot save` on a schedule, stored off-cluster, plus a *tested*
  restore. Your RTO is dominated by restore + control-plane restart, and restoring a
  snapshot rolls the whole cluster back to that revision — anything created since is gone,
  including tokens and leases, so a restore is a fleet-wide event, not a repair.
- **Load.** Watch and list traffic from controllers is the usual scaling pain. A
  `list` of every pod in a large cluster is expensive; prefer informers with resync,
  paginate, and use `resourceVersion=0` reads where staleness is acceptable. Events are
  the noisiest object class and belong on a separate etcd cluster in large clusters.

### 5. Multi-cluster and fleet

Reach for more clusters when you need a boundary a namespace cannot give you: hard
isolation (separate control planes and etcd), blast radius (an upgrade or a bad CRD stops
at one cluster), regional locality, or a version/topology difference (a CPU cluster and a
GPU cluster want different node images, different device plugins and different upgrade
cadences). Costs to state honestly: each control plane is real money and real toil,
cross-cluster service discovery and identity get harder, and fleet-wide changes now need
fan-out tooling (Cluster API, Rancher/Fleet, Argo ApplicationSets, Karmada).

The pragmatic split most GPU platforms end up with: one cluster per region per
environment, with GPU node pools segregated by SKU inside it, and a separate small
"management" cluster running the fleet controllers.

### 6. GPUs in the scheduler — the part that is genuinely different

**The classic path: device plugin + extended resource.**

- `nvidia.com/gpu` is an **extended resource**: integer-valued only, and it must be
  specified in `limits`. You may omit `requests` (Kubernetes copies the limit), you may
  set both if they are **equal**, and you may not set `requests` alone (Kubernetes docs,
  scheduling GPUs). There is no fractional GPU and **no overcommit** at this layer.
- Consequences for bin-packing: a node with 8 GPUs and 3 free cannot host a 4-GPU pod, and
  no amount of CPU headroom changes that. Fragmentation is the dominant scheduling
  pathology on GPU fleets, and it is why "just add more replicas" does not work the way it
  does for CPU services.
- **Sharing options, and what each actually isolates** (NVIDIA k8s-device-plugin README):

| Mechanism | How it works | Isolation | Configuration |
|---|---|---|---|
| **Time-slicing** | CUDA time-slices the GPU between containers; the plugin advertises `replicas ×` the real device count | **None.** Shared memory, shared fault domain — one workload crashes the GPU and all its tenants go with it | `sharing.timeSlicing.resources[].replicas`; with `renameByDefault: true` the resource is advertised as `nvidia.com/gpu.shared`; `failRequestsGreaterThanOne` rejects >1 requests |
| **MPS** | An MPS control daemon space-partitions the device and enforces per-workload memory and compute limits | Partial — enforced limits, still one device, still shared failure | `sharing.mps.resources[].replicas`; mutually exclusive with time-slicing; not supported with MIG |
| **MIG** | Hardware partitioning into isolated instances with their own SMs and framebuffer slice | **Strongest** — a hardware boundary | `flags.migStrategy: none \| single \| mixed`; `mixed` advertises per-profile resources (e.g. `nvidia.com/mig-1g.5gb`) |

  A worked consequence of time-slicing: `replicas: 10` on a node with 8 physical GPUs makes
  `kubectl describe node` report `nvidia.com/gpu: 80`. The scheduler will happily place 80
  pods. Nothing tells them they are sharing, and nothing stops one from allocating all the
  framebuffer. Time-slicing is for dev/notebook/burst workloads, never for tenants who
  need a guarantee.
- **Gang scheduling.** A distributed training job needs all N pods or none; default
  Kubernetes places pods one at a time, so two 32-GPU jobs on a 48-GPU pool can each grab
  24 and deadlock, holding scarce hardware while neither progresses. Kueue (queueing with
  quota and all-or-nothing admission) or Volcano (a batch scheduler with real gang
  semantics) exist for exactly this. Say "partial allocation deadlock" out loud — it is
  the mechanism the question is fishing for. Worth adding that gang scheduling is
  being upstreamed: an in-tree Workload/PodGroup API is in progress (`GenericWorkload`
  alpha in 1.35, `WorkloadWithJob` and `TopologyAwareWorkloadScheduling` alpha in 1.36,
  `DRAWorkloadResourceClaims` alpha in 1.36) — still alpha, so Kueue/Volcano remain the
  production answer today.
- **Topology matters to placement.** A multi-node job wants nodes in one fabric domain
  (placement group / compact placement), and a multi-GPU pod wants GPUs on the same
  NVLink domain and NUMA-aligned with its NIC. That is what makes GPU scheduling a
  topology problem rather than a counting problem.

The sharing configuration itself is a small file on the device plugin, which is worth
being able to sketch because it is where the over-advertisement happens:

```yaml
# NVIDIA device-plugin config: time-slicing. Read this as "lie to the scheduler".
version: v1
flags:
  migStrategy: "none"          # none | single | mixed
sharing:
  timeSlicing:
    renameByDefault: true      # advertise nvidia.com/gpu.shared, so the lie is visible
    failRequestsGreaterThanOne: true
    resources:
      - name: nvidia.com/gpu
        replicas: 10           # 8 physical GPUs are advertised as 80 allocatable
```

**The new path: Dynamic Resource Allocation.** DRA replaces "count of an opaque extended
resource" with claim-based allocation: drivers publish `ResourceSlice`s describing real
devices and attributes, admins define `DeviceClass`es, and workloads reference a
`ResourceClaim`/`ResourceClaimTemplate` with CEL-based attribute filtering — so "give me
a GPU with ≥40 GB and NVLink to its neighbour" becomes expressible, and devices can be
shared between containers or pods by referencing the same claim. Verified status from
`pkg/features/kube_features.go`: `DynamicResourceAllocation` was **alpha in 1.26, beta in
1.32, GA in 1.34**, and locked to default in 1.35. Adjacent gates: `DRAAdminAccess` GA in
1.36, `DRAPrioritizedList` GA in 1.36, `DRAPartitionableDevices` (the MIG-shaped case)
beta in 1.36, `DRAConsumableCapacity` beta in 1.36. If your mental model is still "device
plugin + integer extended resource", that is now the legacy path.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata: {name: one-big-gpu, namespace: ml}
spec:
  spec:
    devices:
      requests:
        - name: gpu
          exactly:                       # the alternative is firstAvailable: [...]
            deviceClassName: gpu.nvidia.com
            selectors:
              - cel:
                  expression: >-
                    device.driver == "gpu.nvidia.com" &&
                    device.attributes["gpu.nvidia.com"].productName == "NVIDIA-H100-80GB-HBM3"
---
apiVersion: v1
kind: Pod
metadata: {name: trainer, namespace: ml}
spec:
  resourceClaims:
    - name: gpu                                  # pod-level claim…
      resourceClaimTemplateName: one-big-gpu
  containers:
    - name: train
      image: registry.example/trainer:v3
      resources:
        claims:
          - name: gpu                            # …referenced by the container
```

*(Field structure follows the `resource.k8s.io/v1` types: a `DeviceRequest` carries either
`exactly` or `firstAvailable`, and a CEL selector sees `device.driver`,
`device.attributes[prefix]` and `device.capacity[prefix]`. The attribute names are
illustrative — they come from whichever driver publishes the `ResourceSlice`s on your
cluster.)*

## Self-quiz

**1. In what order do you upgrade control-plane components and kubelets, and what is the
skew limit?**
API server first, one minor at a time (skipping minors is prohibited by the API
deprecation policy), and in HA all apiserver instances must stay within one minor of each
other. Then controller-manager, scheduler and cloud-controller-manager — any order, they
may be at most one minor behind the apiserver and never ahead. Then kube-proxy and
kubelets, which may be up to **three** minors behind (not two — that changed at 1.25) and
never ahead; minor kubelet upgrades require a drain, in-place is unsupported. `kubectl`
gets ±1. The operational trap: kubelets left three minors behind block the next
control-plane upgrade entirely.

**2. QoS classes — which pods get evicted first under node memory pressure, and why?**
Careful: the kubelet does not rank by QoS class. It ranks by whether usage exceeds
requests, then by Pod Priority, then by how far usage exceeds requests. So the first to go
are pods over their requests, lowest priority first, most-over first; pods at or under
their requests go last, again ordered by priority. QoS class predicts this well because
`BestEffort` pods have no requests (everything is "over"), and `Guaranteed` pods cannot
exceed requests without being OOM-killed by the kernel first. Separately, the kernel's own
OOM killer uses `oom_score_adj`: −997 for Guaranteed, 1000 for BestEffort, and
`1000 − 1000×(memory_request/node_capacity)` for Burstable. Default hard thresholds that
trigger all this: `memory.available<100Mi`, `nodefs.available<10%`,
`imagefs.available<15%`.

**3. A pod is `Pending` — walk the five checks in order.**
(1) `kubectl describe pod` and read the scheduler's own message — it names the predicate
that failed and the node counts. (2) Resource shape: is there a node with enough free
allocatable of the *specific* resource, remembering GPUs are integer and non-overcommittable
and that `describe node` shows both capacity and allocated. (3) Taints and tolerations,
plus node affinity/selector labels — on GPU pools, missing labels usually mean
feature-discovery or the device plugin is not running. (4) Admission-level blocks:
ResourceQuota in the namespace, LimitRange defaults, a validating webhook, or scheduling
gates / a queueing controller (Kueue) deliberately holding the pod. (5) Volumes: unbound
PVCs, or a PVC in a zone with no candidate node. Then the meta-check: is the scheduler
itself healthy, and is this a gang-scheduled job waiting for its whole group?

**4. Why can't you overcommit GPUs the way you overcommit CPU, and what does that do to
bin-packing?**
CPU is a time-shared, preemptible resource with a scheduler in the kernel: over-requesting
degrades everyone smoothly. A GPU exposed via the device plugin is an opaque integer
device assigned wholesale — no kernel-level fair sharing, no memory overcommit (HBM is not
swappable), and the runtime hands the container exclusive access. So allocation is
all-or-nothing per device, requests must equal limits, and fractional requests are
rejected. Bin-packing therefore fragments: 3 free GPUs on each of two nodes cannot host
one 4-GPU pod, and a fleet can be 80% "allocated" while no job can start. The mitigations
are sharing modes with different isolation (time-slicing — no isolation; MPS — enforced
partitions; MIG — hardware partitions), gang scheduling to avoid partial-allocation
deadlock, defragmentation/re-packing policies, and, on modern clusters, DRA so the request
can describe the device you actually need instead of a count.

**5. Intermittent pod-to-service 5xx with healthy pods — name three network-layer
suspects.**
Conntrack table exhaustion on the node (a node-global table with no per-pod quota, so one
noisy neighbour drops everyone's flows); DNS — with `ndots:5`, external names walk the
search list, so a saturated or slow CoreDNS shows up as sporadic connection timeouts
rather than DNS failures; and MTU mismatch with overlay encapsulation, where small
requests succeed and larger responses hang. Two more worth adding: the endpoint-removal
race at shutdown (the pod stops accepting before kube-proxy/CNI removes it from the
endpoint set — fixed with a preStop sleep and graceful shutdown), and readiness probes
that are too permissive so traffic reaches pods that are not actually ready.

**6. etcd is showing `NOSPACE`. What is happening and what do you do?**
The backend database exceeded `--quota-backend-bytes` (default **2 GiB**), so etcd raised
an alarm and switched to read-only: reads succeed, every write fails, and the cluster
looks half-alive. Cause is usually accumulated revisions (compaction not keeping up, or
disabled) or genuine object growth — often Events or a controller hot-looping on updates.
Fix: compact to a recent revision, `etcdctl defrag` **one member at a time** (it blocks
the member), then `etcdctl alarm disarm`. Then remove the cause: confirm the apiserver's
automatic compaction is running (default every 5 minutes), find the object class that
grew, and consider raising the quota with an explicit understanding that bigger databases
mean longer defrags and slower restores.

## Refresh only if

- **Dynamic Resource Allocation**, if your model is still "device plugin + extended
  resource". Real status: core DRA **GA in 1.34** and locked to default in 1.35, with
  `DRAAdminAccess` and `DRAPrioritizedList` GA in 1.36 and `DRAPartitionableDevices` /
  `DRAConsumableCapacity` at beta in 1.36. The objects to be able to name are
  `ResourceSlice` (published by the driver), `DeviceClass` (curated by the admin),
  `ResourceClaim` / `ResourceClaimTemplate` (requested by the workload), with CEL
  attribute selectors. This is the single most likely GPU-scheduling probe in a 2026
  interview.
- **Gateway API**, if you last looked when it was "the Ingress replacement, still beta".
  Current upstream (`kubernetes-sigs/gateway-api` README, v1.6.1) has GA-level `v1`
  support for `GatewayClass`, `Gateway`, `ListenerSet`, `HTTPRoute`, `GRPCRoute`,
  `TLSRoute`, `TCPRoute`, `UDPRoute`, `BackendTLSPolicy` and `ReferenceGrant` — the
  L4 routes are no longer the experimental corner they used to be. The role split
  (infrastructure provider / cluster operator / application developer) is the part
  interviewers actually ask about.
- **Sidecarless / ambient service mesh**, if your mesh model is one Envoy per pod: the
  trade is per-node L4 plus optional L7 gateways instead of per-pod proxies — materially
  less memory and no sidecar-ordering problems, at the cost of a different (coarser)
  identity and policy enforcement point.

## Recall card

Cover the right column and say each value out loud; if one is fuzzy, reread the section
in brackets.

| Fact | Value |
|---|---|
| kubelet skew | up to **3** minors behind apiserver (2 before 1.25) [§1] |
| kubectl skew | ±1 minor [§1] |
| c-m / scheduler / CCM skew | ≤1 minor behind, never ahead [§1] |
| Upgrade order | apiserver → c-m/sched/CCM → kubelet/kube-proxy [§1] |
| Eviction ranking | usage>requests → Priority → excess over requests (**not** QoS class) [§2] |
| `oom_score_adj` | Guaranteed −997 · BestEffort 1000 · Burstable `1000−1000×req/cap` [§2] |
| Default hard eviction | `memory.available<100Mi`, `nodefs<10%`, `imagefs<15%`, `imagefs.inodesFree<5%` [§2] |
| Eviction pressure transition | 5 m [§2] |
| Node failure timeline | status every 10 s · NotReady at 50 s · pods evicted at +300 s [§3] |
| Exit codes | 137 = SIGKILL/OOM · 143 = SIGTERM [§3] |
| etcd quorum | ⌈(n+1)/2⌉ — 3 tolerates 1, 5 tolerates 2; odd sizes only [§4] |
| etcd backend quota | 2 GiB default → alarm + **read-only** [§4] |
| Compaction vs defrag | apiserver compacts every 5 m; `defrag` reclaims disk and blocks [§4] |
| GPU resource rules | integer, in `limits`, requests must equal limits, no overcommit [§6] |
| Time-slicing `replicas: 10` | 8 GPUs advertised as 80; **zero** isolation [§6] |
| DRA | GA 1.34, locked 1.35; `ResourceSlice`/`DeviceClass`/`ResourceClaim` [§6] |
