---
lesson: "02b.5"
title: "Kubernetes topology alignment — Topology, CPU, and Memory Managers"
module: "02b"
concept: "Kubernetes topology alignment"
status: not-started
est_time: "10h"
prev: "04-server-architecture-8gpu.md"
next: "06-storage-nvme.md"
artifacts: []
sources: 8
---
# 02b.5 · Kubernetes topology alignment — Topology, CPU, and Memory Managers
> **Concept.** How Kubernetes aligns a pod's CPUs, memory, and GPU onto one NUMA node — Topology Manager policies (guarantee vs attempt), the required CPU/Memory Manager static policies, the device-plugin `TopologyInfo` trap that silently defeats it, and where Dynamic Resource Allocation (DRA) is taking this next.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 04 established the physical layout of an 8-GPU node: a fixed NVLink/NVSwitch domain, a 4/4 CPU-socket split, and GPU↔NIC pairs rail-aligned to the same PCIe root complex — all of it baked into the board at manufacturing time and unchangeable at runtime. None of that hardware correctness is optional, and none of it is negotiable by software. What's still open, and entirely software's job, is whether the orchestrator sitting on top of that hardware — Kubernetes — actually *respects* the fixed layout when it decides which CPUs, which memory pages, and which GPU a given pod gets. This lesson is the gap between "the node is wired correctly" and "the pod that lands on it runs correctly": the kubelet-level machinery that turns a hardware fact into an admission guarantee, and the one silent way that guarantee breaks. It's the module's anchor lesson because it's where topology fluency becomes a Kubernetes skill, not just a hardware-reading skill — and it directly sets up lesson 06, where the same "is this thing actually co-located with the GPU" question gets asked of storage instead of CPU and memory.

## Why this matters

You already know the earlier lessons' hardware truth: on a two-socket GPU box, a GPU hangs off one socket's PCIe root complex, and a thread pinned to the *wrong* socket pays the cross-socket interconnect tax (UPI/xGMI) on every host-to-device copy and every memory access the kernel didn't place locally. On an H100 SXM node that remote-NUMA penalty is not rounding error — it's tens of percent of achievable HBM-feeding bandwidth, and it shows up as GPUs sitting at 60% SM utilization while the bill runs at 100%.

The interview question that separates a Senior Platform Engineer from a CKA holder is exactly this: **"How do you guarantee a pod's CPU, memory, and GPU all land on one NUMA node?"** The wrong answer is "node affinity and hope." The right answer is the kubelet's NUMA-alignment layer: the **Topology Manager** coordinating three hint providers (CPU Manager, Memory Manager, Device Manager), gated on the *static* policies, and — the part almost everyone misses — dependent on the GPU device plugin publishing `TopologyInfo`. Get one piece wrong and the policy is enabled, the pods admit, dashboards look green, and every GPU workload is silently cross-socket. That's exactly the job this module's README names as the stated differentiator at neocloud operators: CoreWeave's *Sr HW Engineer, GPU & PCIe* posting calls for turning a silent placement fault into a monitored alert, and the interview probe is verbatim "guarantee a pod's CPU+mem+GPU on one NUMA node in k8s." That's the fleet-scale waste story you get paid to prevent: the failure is invisible per-pod and only shows up as a utilization/cost gap across thousands of nodes. This is the money skill of the module — own it cold.

## What's new here (calibration)

You run Kubernetes daily, so we skip re-teaching what you already know:

- **Skipped:** requests/limits, QoS classes, `nodeSelector`/taints, and how `kube-scheduler` picks a node — that's the scheduler's job, and you already own it operationally.
- **Skipped:** kernel-level NUMA mechanics, `cgroups`/hugepages/THP fundamentals — covered in the 01b Linux module; this lesson assumes you can read `numactl --hardware` cold.
- **New:** the kubelet runs a *second*, separate control loop after the scheduler — Topology Manager coordinating CPU/Memory/Device Manager hint providers — with its own admission failure mode (`TopologyAffinityError`, not `Pending`).
- **New:** the exact bitmask arithmetic behind the hint-provider merge, and the device-plugin `TopologyInfo` trap that makes GPU alignment silently a no-op even when the policy is `single-numa-node`.
- **New:** Dynamic Resource Allocation (DRA) — GA in Kubernetes **v1.34** (September 2025) — and its `ResourceSlice`/`ResourceClaim` model, which expresses GPU↔NIC co-location as a first-class scheduler constraint instead of an opt-in kubelet-side hint. Know the trajectory even before your fleet runs it.

## Core concepts

### The Topology Manager: a coordinator, not an allocator

Topology Manager doesn't allocate anything itself. It's a **kubelet component that gathers NUMA-affinity hints from hint providers and picks a merged hint**, which the providers then honor when they do their real allocation. Three hint providers exist today:

- **CPU Manager** — which NUMA node(s) the exclusive CPUs would come from.
- **Memory Manager** — which NUMA node(s) can satisfy the pod's memory + hugepages.
- **Device Manager** — which NUMA node(s) the requested devices (GPUs, NICs, etc.) live on, *as reported by each device plugin's `TopologyInfo`*.

Each provider emits, per resource, a set of **`TopologyHint`s**: bitmasks over NUMA nodes plus a `preferred` boolean (true if that mask is the narrowest/best affinity possible for that resource). Topology Manager computes the **cross-product** of every provider's hints, ANDs the NUMA masks together, and picks the merged hint that is `preferred` on the narrowest node-count. That merged hint is the alignment decision. The policy then decides what to *do* with it at admission.

Topology Manager is GA (since v1.27). It runs only for **Guaranteed** pods with integer CPU — that's the only QoS where CPU Manager produces a real hint; burstable/best-effort pods get no exclusive CPUs and thus no meaningful alignment.

### Why Guaranteed QoS + integer CPU, specifically

The alignment machinery only engages when *all* of these hold, and each is a common silent miss:

- **QoS class is Guaranteed** — every container sets `requests == limits` for both CPU and memory. A single burstable container in the pod, or a missing memory limit, drops the pod out of the static CPU pool: it lands back on the shared CPUs with no pinning and no hint.
- **CPU request is an integer** — `cpu: "4"`, not `cpu: "3500m"`. Fractional CPU never gets exclusive cores even under `static`; the kubelet treats it as shared. `cpu: "4000m"` counts as integer 4 (it's the millicore value being a whole multiple of 1000 that matters).
- **The pod is not in the reserved/shared pool** — system pods and anything without an integer Guaranteed CPU request run on the *shared* CPUs (`allocatable minus reserved minus exclusive`). This is why you size `reservedSystemCPUs`/`kubeReserved` deliberately on GPU nodes: the feeder cores you want exclusive must not be eaten by the shared pool.

Miss any one and the pod runs, dashboards are green, and it's silently unpinned — same failure family as the `TopologyInfo` trap, different cause. First thing to check when "alignment isn't working": `kubectl get pod -o jsonpath='{.status.qosClass}'` must say `Guaranteed`.

### Admission ordering: scheduler first, kubelet second

Sequence matters for debugging. `kube-scheduler` picks the node using node-level `Allocatable` (total CPU, memory, `nvidia.com/gpu` count) — it has **no per-NUMA view** and will happily place a pod on a node that can't satisfy it topologically. The pod binds, the kubelet pulls it, and only *then* do the resource managers run at admission and possibly reject with `TopologyAffinityError`. So a topology rejection looks nothing like a scheduling failure: it's not `Pending` with `Unschedulable`, it's `Failed`/`Terminated` on a node it already reached. The retry loop reschedules it — often back to the same node pool — which is why a systematic misconfiguration (too-fragmented free CPUs, or a genuinely oversized pod) can produce a crash-loop of `TopologyAffinityError` terminations rather than a clean `Pending`. There is topology-aware *scheduling* work (NUMA-aware scheduler plugins) precisely to close this scheduler-blindness gap, but stock `kube-scheduler` does not see NUMA — DRA (below) is the more structural fix.

### Scope: container vs pod

`topologyManagerScope` controls the alignment unit:

- **`container`** (default): each container is aligned independently. Container A can land on NUMA 0, container B on NUMA 1. Fine for sidecars, wrong for a multi-container pod that must share a GPU's node.
- **`pod`**: all containers in the pod are aligned to a *single common* NUMA affinity. This is what you want for a GPU training pod where the framework container and its helpers must all sit with the GPU. Pod scope makes the constraint stricter and admission more likely to fail — which is correct: you *want* it to fail loudly rather than split.

### The four policies — guarantee vs attempt

This distinction is the whole interview. The policy is a kubelet flag / config field (`topologyManagerPolicy`) applied node-wide:

- **`none`** (default): Topology Manager is off. No hints collected, no alignment. Providers allocate independently.
- **`best-effort`**: collect hints, compute the merged hint, and **store the preferred affinity so the providers try to honor it — but admit the pod regardless of whether the hint was satisfiable.** This *prefers* alignment and *never rejects*. It is the trap policy for people who "turned on Topology Manager" and think they're safe: misaligned pods still run.
- **`restricted`**: compute the merged hint; **if the chosen hint is not `preferred` (i.e., resources couldn't be confined to the minimal NUMA set), reject the pod** at admission with `TopologyAffinityError` — it fails admission and is `Terminated`. If the hint *is* preferred, admit and align. This guarantees "aligned or dead," but across *any* minimal node set (could be 2 NUMA nodes if that's the narrowest possible).
- **`single-numa-node`**: the strictest. Only a hint that fits **entirely within one NUMA node** counts as satisfiable. If no single-NUMA-node placement works, **reject** with `TopologyAffinityError` (`Terminated`). This is the policy you cite for "guarantee CPU+mem+GPU on one NUMA node."

Mnemonic: `best-effort` = *prefer but admit anyway*; `restricted`/`single-numa-node` = *admit only if the hint is satisfiable, else reject*. A rejected pod doesn't go back to `Pending` and get rescheduled onto a better-fitting node the way a scheduling failure would; the kubelet already owns it, so it terminates with a `TopologyAffinityError` reason and the ReplicaSet/Deployment retries — potentially onto the same node, potentially looping, which is worth watching for.

Policy options refine behavior further (feature-gated — check your cluster's feature-gate status before relying on either): `prefer-closest-numa-nodes` makes multi-NUMA merges prefer physically adjacent nodes, and `max-allowable-numa-nodes` (default 8) bounds the hint-permutation cost on very large NUMA machines (the merge is a cross-product over per-node bitmasks — exponential in NUMA count, hence the cap).

### How the merge actually works (bitmask walkthrough)

Concretely, on a 2-NUMA node where the GPU is on NUMA 0 and the pod wants 8 CPUs + 1 GPU. Each provider hands Topology Manager a list of `TopologyHint{NUMANodeAffinity: <bitmask>, Preferred: <bool>}`:

```
CPU Manager (8 free cores on node 0, 8 on node 1):
  {0b01, preferred}   # all 8 from node 0  → narrowest, preferred
  {0b10, preferred}   # all 8 from node 1  → narrowest, preferred
  {0b11, not-preferred}  # 4+4 split across both nodes
Device Manager (GPU on node 0, TopologyInfo present):
  {0b01, preferred}   # GPU is on node 0
Memory Manager (16Gi fits either node):
  {0b01, preferred}
  {0b10, preferred}
```

Topology Manager takes the cross-product, **bitwise-ANDs** the masks in each combination, and keeps the result with the fewest set bits that is `preferred` across all providers. `CPU{0b01} AND Dev{0b01} AND Mem{0b01} = 0b01`, all preferred → **merged hint `{0b01, preferred}`, one NUMA node.** Every provider then allocates from node 0. Under `single-numa-node` this is satisfiable (one bit set) → admit.

Now delete the GPU's `TopologyInfo`. Device Manager can't say where the GPU is, so it emits `{0b11, preferred}` (all nodes). The AND no longer forces node 0 — `CPU{0b10} AND Dev{0b11} AND Mem{0b10} = 0b10` is a valid single-node merged hint, so the pod admits with **CPUs+mem on node 1 and the GPU physically on node 0**. That is the trap, in arithmetic: an all-ones mask is the identity element of AND — it constrains nothing.

### Prerequisites: the static policies (this is where it breaks)

Topology Manager is worthless without the providers actually producing hints, and by default they don't:

- **`cpuManagerPolicy: static`** — required. With the default `none` policy, CPU Manager never assigns exclusive CPUs and emits no useful hint, so there is nothing for Topology Manager to align CPUs to. `static` gives Guaranteed pods with integer CPU requests **exclusive pinned physical CPUs** carved out of the shared pool. Sub-option **`full-pcpus-only`**: on SMT/hyperthreaded nodes a "CPU" is a *thread*, and two threads share one physical core. Without `full-pcpus-only`, a 3-CPU request can be handed 3 threads that split a physical core with some *other* pod's thread — noisy-neighbor contention on the shared core's execution units, defeating the point of exclusivity. `full-pcpus-only` forces allocation in whole-physical-core units (allocates both siblings together) and rejects pods whose CPU count isn't a multiple of the threads-per-core with `SMTAlignmentError`. Essential on hyperthreaded GPU nodes where you want clean, contention-free cores feeding the GPU.
- **`memoryManagerPolicy: Static`** — required for memory to be a real constraint. **Memory Manager reached GA in Kubernetes v1.32** (December 2024). With the default `None`, memory contributes no hint and a pod can get CPUs+GPU on NUMA 0 while the kernel serves its pages from NUMA 1. `Static` makes Memory Manager guarantee pages come from the aligned node's memory. It **requires `reservedMemory`** configured per NUMA node — you must carve out system/kube-reserved memory on each node, and the reserved total must be consistent with `kubeReserved` + `systemReserved` + the hard eviction threshold, or the kubelet won't start. This config friction is why teams skip Memory Manager and quietly lose memory-side alignment.

  The `reservedMemory` math trips everyone up: the **sum** of `reservedMemory.limits.memory` across all NUMA nodes must equal `kubeReserved.memory` + `systemReserved.memory` + `evictionHard.memory.available` (default `100Mi`). It's a global-total constraint distributed across nodes, and hugepages of each size are tracked as separate resources with their own per-node reservation. Get the sum wrong and the kubelet exits at startup with a memory-reservation validation error — a common cause of "the node won't come back after I enabled Memory Manager."

Do not confuse this GA date with DRA's: they are two independent Kubernetes features that happened to graduate roughly a release apart. Memory Manager (a kubelet hint provider for the CPU/Memory/Device Manager trio described above) went GA in **v1.32**. Dynamic Resource Allocation (a wholly different, scheduler-level device-selection API, described below) went GA in **v1.34**. Conflating the two is an easy mistake — and a fast way to lose credibility in an interview that probes version fluency.

### THE TRAP: device plugin `TopologyInfo`

Here is the failure that costs real money and is nearly invisible. Device Manager can only emit a NUMA hint for a device **if the device plugin populated the `TopologyInfo` field** (the `NUMANodes` list) when it registered that device over the kubelet device-plugin gRPC API. This is *opt-in on the plugin side*:

- If the NVIDIA device plugin (or any plugin) advertises GPUs **without** `TopologyInfo`, Device Manager has **no NUMA affinity** for those GPUs. It contributes a hint of "any NUMA node" (all bits set, preferred) — which ANDs away to nothing. The GPU effectively **drops out of the merge**.
- Result: Topology Manager aligns CPU and memory to each other perfectly and admits the pod, but the **GPU alignment silently does nothing** — the pod can get a GPU on NUMA 1 and CPUs on NUMA 0, and the policy will *not* reject it, because from the merge's perspective the GPU imposed no constraint. `single-numa-node` looks like it's working; it aligned everything it was given a hint for.

So "we enabled `single-numa-node`" is **not sufficient**. You must verify the plugin publishes topology. For the NVIDIA k8s-device-plugin this is the `DEVICE_PLUGIN_MODE` / topology settings — historically gated behind reading each GPU's NUMA node from sysfs (`/sys/bus/pci/devices/.../numa_node`) and reporting it; some deployments (older versions, MIG configs, or `numa_node = -1` on misreporting hardware/VMs) end up publishing no usable NUMA node. `numa_node == -1` (common on VMs and some cloud instances that don't expose NUMA topology) means the plugin *can't* produce a hint even if it wants to — alignment is impossible and best-effort/single-numa-node degrade to "CPU+mem only."

### Kubelet config and verification

All of this lives in **kubelet configuration** (config file preferred over flags), applied per node and requiring a kubelet restart plus **draining the node and removing the stale state files** (`/var/lib/kubelet/cpu_manager_state`, `/var/lib/kubelet/memory_manager_state`) — the kubelet refuses to start if the persisted policy doesn't match the configured one.

```yaml
# /var/lib/kubelet/config.yaml (KubeletConfiguration)
cpuManagerPolicy: static
cpuManagerPolicyOptions:
  full-pcpus-only: "true"
memoryManagerPolicy: Static
reservedMemory:
  - numaNode: 0
    limits: { memory: 1178Mi }   # must cover kube+system reserved + eviction
  - numaNode: 1
    limits: { memory: 1178Mi }
topologyManagerPolicy: single-numa-node
topologyManagerScope: pod
kubeReserved:   { cpu: "500m", memory: "1Gi" }
systemReserved: { cpu: "500m", memory: "512Mi" }
```

**Verify alignment actually happened** (don't trust "the policy is on"):
- CPU pinning: `cat /var/lib/kubelet/cpu_manager_state` shows the assignments; inside the pod `cat /sys/fs/cgroup/cpuset.cpus.effective` (or `taskset -pc 1`) shows the exact pinned set.
- Which NUMA node those CPUs belong to: `lscpu` / `numactl -H` maps CPU IDs → NUMA node.
- GPU's NUMA node: `cat /sys/bus/pci/devices/<gpu-bdf>/numa_node`, or `nvidia-smi topo -m` for the CPU-affinity column. Cross-check that the GPU's NUMA node equals the pinned CPUs' NUMA node.
- Memory: `/var/lib/kubelet/memory_manager_state`.
- The negative test: force a pod that can't fit one NUMA node and confirm it's `Terminated` with `reason: TopologyAffinityError` (`kubectl describe pod`). If it *admits* anyway under `single-numa-node`, a hint provider isn't constraining — usually the GPU's missing `TopologyInfo`.

### Detecting the trap across a fleet

Per-pod inspection doesn't scale to thousands of GPU nodes; you need a fleet signal. Practical detectors, cheapest first:

- **Plugin-side assertion at rollout.** On every GPU node, gate the device-plugin DaemonSet on `numa_node != -1` for each GPU (read `/sys/bus/pci/devices/<bdf>/numa_node`). A node where any GPU reports `-1` cannot align — surface it as a node condition or a `NodeFeature` label and alert. This catches VMs/instance types that don't expose NUMA topology *before* they take workloads.
- **The utilization/cost gap.** The economic tell: GPUs pinned but stuck at low SM utilization with high host-to-device copy latency, or `dcgm` reporting elevated NVLink/PCIe traffic that shouldn't exist for a single-GPU job. A cluster-wide "aligned pods running cross-NUMA" query joins pod CPU-set NUMA (from cAdvisor/`cpu_manager_state`) against the GPU's NUMA node.
- **Synthetic canary.** A tiny periodic Job requesting 1 GPU + integer CPU under `single-numa-node`; a bandwidth microbench (`bandwidthTest`, or a host↔device `cudaMemcpy` loop) whose result drops off a cliff when placed cross-socket. If the canary is fast, alignment works; if it's slow *and admitted*, the plugin isn't publishing topology.

The reason this matters at your target companies: a single misconfigured device-plugin image rolled to a node pool silently makes every GPU job on it cross-socket. Nothing crashes, nothing pages — you only see it as a percentage of GPU-hours burned on interconnect stalls. That is a cost/observability story you can own end to end.

### Failure-mode quick reference

| Symptom | Likely cause | Confirm with |
| --- | --- | --- |
| Pod runs but CPUs unpinned | Not Guaranteed QoS, or fractional CPU | `qosClass` field; `cpuset.cpus.effective` = full node range |
| Policy "on" but nothing rejects | `best-effort` policy, or no provider constrains | check `topologyManagerPolicy` value; inspect hints |
| GPU pod cross-socket, still admits | Device plugin missing `TopologyInfo` / `numa_node == -1` | `/sys/bus/pci/devices/<bdf>/numa_node`; `nvidia-smi topo -m` |
| Kubelet won't start after enabling | `reservedMemory` sum wrong, or stale `*_manager_state` | kubelet logs; delete state files; recheck memory math |
| Pods `TopologyAffinityError`-loop | Oversized pod or fragmented free CPUs vs `single-numa-node` | `kubectl describe`; `numactl -H` free-core count per node |
| Split SMT siblings, noisy neighbor | `full-pcpus-only` not set | `lscpu` sibling map vs assigned CPU IDs |

### Where this is heading: Dynamic Resource Allocation (DRA)

The device-plugin API — and its opt-in `TopologyInfo` — is the current mechanism, but it has a structural ceiling: the only thing a device plugin can tell the kubelet is a flat list of NUMA nodes, and only the *kubelet* sees it, at admission time, after the scheduler has already committed the pod to a node. **Dynamic Resource Allocation (DRA) reached GA in Kubernetes v1.34** (September 2025). It replaces "GPU as an opaque integer count" with a structured, scheduler-visible model:

- A DRA **driver** publishes device capabilities as attributes on a `ResourceSlice` — the documented example set includes architecture, PCIe root complex, and NUMA node.
- A workload author writes **inter-device constraints** on a `ResourceClaim`. The canonical documented example is a request for a GPU *and* a NIC with the requirement that both are attached to the same PCIe root complex — exactly the rail-alignment problem lessons 04 and 05 have been building toward, now expressible as a scheduling constraint instead of an out-of-band kubelet hint.
- Because a `ResourceClaim`'s constraints are visible to `kube-scheduler` itself (not only the kubelet at admission), DRA closes the scheduler-blindness gap named above: the scheduler can filter and score nodes on topology attributes directly, instead of discovering a mismatch only after the pod has already bound.

At KubeCon EU 2026, NVIDIA donated its DRA driver to CNCF, meaning production NVIDIA-GPU DRA support is now a CNCF-governed artifact rather than a vendor-proprietary one — worth knowing by name if you're interviewing at an NVIDIA-adjacent shop. One caution: **GA of the core DRA API is not the same claim as "every vendor's DRA driver is battle-tested."** The API being stable means you can build against it without it changing under you; a specific driver (including a freshly donated one) may still be maturing in production. Know the term, the version, and the distinction between API-GA and driver-maturity — interviewers who ask about this are usually probing whether you track the field or just memorized the current mechanism.

## Perspectives

**Developer.** From a workload YAML author's view, none of this is visible unless the pod goes `Terminated` with a cryptic `TopologyAffinityError`. The developer's signal is a crash-loop with a reason string they've never seen before, and the fix — rightsizing the request, or fixing the device-plugin rollout — is entirely a platform-team responsibility. The app team cannot self-serve past this; they need you.

**Operator / platform team.** This is squarely your surface: kubelet configuration, the static policy prerequisites, `reservedMemory` arithmetic, verifying the device plugin actually publishes `TopologyInfo`, and building the fleet-scale detectors (canary jobs, plugin-rollout gates) described above. Nobody else in the org owns this layer.

**Kernel / hardware.** Everything Topology Manager does is a *userspace-visible policy wrapper* around facts the kernel already exposes: `cpuset` cgroups for CPU pinning, `numa_node` sysfs files for device affinity, `mempolicy`/`cpuset.mems` for memory. Topology Manager doesn't invent new kernel mechanism — it orchestrates existing Linux primitives across multiple resource types atomically, at admission, so that a partial success (CPUs aligned, memory not) can't silently happen.

**Economics / future-proofing.** DRA's richer topology model is where NVIDIA-adjacent shops are visibly investing — a CNCF driver donation and a recent core-API GA are both signals of where engineering effort is flowing. A staff candidate who can speak to *both* the current device-plugin trap and the DRA trajectory signals they're tracking the field, not reciting a fixed mechanism that will be legacy within a few release cycles.

## Real-world use cases

- **Ronak Nathani — "Keeping GPU Workloads NUMA-Local in Kubernetes"** — the field's best practitioner deep-dive on this exact machinery: walks the hint-provider merge, the NVIDIA `TopologyInfo` trap, and end-to-end verification on real GPU nodes, and reports a concrete production number — **inference pods whose CPUs spanned both sockets showed >30% higher p99 tail latency** than pods pinned to a single socket — plus an explicit recommendation to set `memoryManagerPolicy: Static` so memory participates in alignment alongside CPU and GPU. What it shows: the cost of skipping Memory Manager isn't hypothetical; it's a measured, double-digit-percent tail-latency regression. `https://ronaknathani.com/blog/2026/05/keeping-gpu-workloads-numa-local-in-kubernetes/`
- **Google Cloud Blog — "Kubernetes device management with DRA Dynamic Resource Allocation"** — confirms the `ResourceSlice`/`ResourceClaim` model in detail, including the PCIe-root-complex/NUMA-node attribute example, and explicitly contrasts DRA's granular topology-aware scheduling against the legacy device-plugin's "just an integer count" model. What it shows: a major cloud vendor's own framing of exactly why DRA is a structural, not cosmetic, upgrade over the device-plugin API this lesson otherwise centers on. `https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation`
- **CNCF Blog — "Understanding dynamic resource allocation in Kubernetes"** — CNCF's own vendor-neutral explainer of the DRA model, complementing the Google Cloud piece. What it shows: DRA is treated by the project itself as a headline, cross-vendor capability, not a single cloud's feature. `https://www.cncf.io/blog/2026/07/01/understanding-dynamic-resource-allocation-in-kubernetes/`
- **Kubernetes Blog — "Kubernetes v1.34: DRA has graduated to GA"** — the official GA announcement anchoring the version/timeline fact used throughout this lesson. What it shows: the primary-source date you should cite if asked "when did DRA go GA" in an interview. `https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/`

## Worked example

Single GPU node (a real kubelet, kind on a GPU box, or k3s). Node has 2 NUMA nodes, GPU on NUMA 0.

**1. Configure and restart the kubelet** with the config block above (`single-numa-node` + static CPU/Memory, `pod` scope). Drain, delete the two `*_manager_state` files, restart kubelet, uncordon.

**2. Schedule an aligned Guaranteed pod** — integer CPU, one GPU, requests==limits:

```yaml
resources:
  limits:   { cpu: "8", memory: "16Gi", nvidia.com/gpu: "1" }
  requests: { cpu: "8", memory: "16Gi", nvidia.com/gpu: "1" }
```

Observed result (GPU on NUMA 0, 8 free full cores on NUMA 0): pod admits and runs.
```
$ kubectl exec pod -- cat /sys/fs/cgroup/cpuset.cpus.effective
0-3,32-35                     # 4 physical cores + SMT siblings, all NUMA 0
$ numactl -H | grep 'node 0 cpus'
node 0 cpus: 0-15 32-47       # confirms 0-3,32-35 ∈ NUMA 0
$ nvidia-smi topo -m          # GPU0 CPU Affinity column → 0-15,32-47 (NUMA 0)
```
CPUs, memory, and GPU all on NUMA 0. `full-pcpus-only` gave whole cores (0/32, 1/33, ...), no split SMT siblings. Cross-check the kubelet's own view:
```
$ cat /var/lib/kubelet/cpu_manager_state | jq '.entries'
{ "<pod-uid>": { "app": "0-3,32-35" } }        # kubelet's exclusive assignment
$ cat /var/lib/kubelet/memory_manager_state | jq '.entries."<pod-uid>"'
{ "memory": { "0": { ... } } }                  # memory drawn from NUMA node 0
```

**3. Force a rejection** — request more of one resource than a single NUMA node can co-locate (e.g. more exclusive CPUs than one node has free, or memory exceeding one node's free after `reservedMemory`):

```yaml
requests: { cpu: "40", memory: "16Gi", nvidia.com/gpu: "1" }  # >1 NUMA node of CPU
```
Observed:
```
$ kubectl describe pod ...
Status:  Failed
Reason:  TopologyAffinityError
Message: Resources cannot be allocated with Topology locality
```
Under `single-numa-node` the pod is **Terminated**, not `Pending`. Switch the policy to `best-effort` (edit kubelet config, drain, delete state files, restart), redeploy the *same* pod:
```
$ kubectl get pod ...
NAME   READY   STATUS    RESTARTS
job    1/1     Running   0
$ kubectl exec pod -- cat /sys/fs/cgroup/cpuset.cpus.effective
0-15,16-23,32-47,48-55        # spans BOTH NUMA nodes — admitted cross-NUMA
```
It **admits and runs cross-NUMA** — proving the guarantee-vs-attempt difference on identical inputs, with nothing changed but the policy string.

**4. Demonstrate the TopologyInfo trap** — deploy the device plugin configured *without* topology reporting (or on a node where the GPU's `numa_node == -1`). First confirm the pre-condition:
```
$ cat /sys/bus/pci/devices/0000:17:00.0/numa_node
0                     # hardware DOES know the GPU is on NUMA 0...
# ...but the plugin advertised it without TopologyInfo, so the kubelet never learned it.
```
Redeploy the aligned pod from step 2 but with CPUs requested from NUMA 1's free pool. It **admits under `single-numa-node`** with CPUs on NUMA 1 and the GPU physically on NUMA 0:
```
$ kubectl exec pod -- cat /sys/fs/cgroup/cpuset.cpus.effective
16-19,48-51          # NUMA 1
$ nvidia-smi topo -m # GPU0 CPU Affinity → 0-15,32-47 (NUMA 0)  → MISMATCH
```
The policy didn't reject because the GPU contributed an all-nodes hint. That is the silent failure, reproduced — same policy, same pod spec, silently cross-socket.

**5. The DRA-equivalent constraint, sketched.** You will not run this on a v1.27-era kubelet, but it's worth seeing what the *same* requirement looks like once expressed as a `ResourceClaim` instead of a device-plugin hint:

```yaml
# legacy device-plugin flow: a flat, kubelet-only hint
# Device Manager sees: GPU0 → TopologyInfo.NUMANodes = [0]  (or nothing, if the plugin omits it)

# DRA flow: a scheduler-visible, structured constraint
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
spec:
  devices:
    requests:
      - name: gpu
        deviceClassName: gpu.example.com
      - name: nic
        deviceClassName: nic.example.com
    constraints:
      - requests: ["gpu", "nic"]
        matchAttribute: "resource.example.com/pcieRootComplex"
```
The constraint — "both devices must share the same PCIe root complex" — now lives where the *scheduler* can see it before it ever binds the pod, instead of being a hint the kubelet discovers only at admission. The failure mode from step 4 (a plugin silently omitting the attribute) is architecturally harder to hit here, because the constraint is declared by the workload author, not inferred from whatever the plugin happened to publish — though it isn't impossible if the DRA driver itself under-reports device attributes.

## Practice

On a test node you control (kind/k3s/single kubelet), for the [Topology Teardown](../practice/topology-teardown/README.md) deliverable:

1. Configure `cpuManagerPolicy: static` (+ `full-pcpus-only`), `memoryManagerPolicy: Static` (+ `reservedMemory`), `topologyManagerPolicy: single-numa-node`, `topologyManagerScope: pod`. Capture the exact config, the drain/state-file-delete/restart sequence, and confirm the kubelet came up.
2. **Admit case:** schedule a Guaranteed integer-CPU pod that fits one NUMA node. Verify pinning via `cpuset.cpus.effective` + `numactl -H`, and (if you have a GPU) cross-check `nvidia-smi topo -m` against the CPUs' NUMA node.
3. **Reject case:** schedule a pod that can't fit one NUMA node. Capture the `TopologyAffinityError` / `Terminated`. Flip to `best-effort`, redeploy, show it now admits cross-NUMA.
4. **TopologyInfo trap:** either run a device plugin without `TopologyInfo` (or on `numa_node == -1` hardware) and show GPU-CPU alignment silently not happening, **or** — if you can't get GPU hardware — reason it through in writing: show `/sys/bus/pci/devices/<bdf>/numa_node`, explain the all-bits-set hint, and state precisely why `single-numa-node` admits a misaligned GPU pod.
5. **Bonus (no cluster required):** sketch the DRA `ResourceClaim` equivalent for your node's GPU+NIC pairing (as in the worked example above) and write one paragraph on why the constraint is or isn't harder to silently break than the `TopologyInfo` case.

**Acceptance:** a note capturing (1) the working config, (2) the admit-vs-reject behavior with the exact `TopologyAffinityError` output and the best-effort contrast, and (3) the TopologyInfo trap — how you'd detect it in a fleet and confirm the plugin publishes NUMA topology. That note is the deliverable artifact.

## Common pitfalls

1. **Enabling `topologyManagerPolicy` without the static CPU/Memory prerequisites.** Topology Manager without `cpuManagerPolicy: static` and `memoryManagerPolicy: Static` has nothing real to align — the providers emit no useful hints, and the policy setting alone accomplishes nothing. Always verify the prerequisite policies first.
2. **Treating `best-effort` as a guarantee.** It computes the same merged hint as `restricted`/`single-numa-node` but never rejects — it is the "looks configured, isn't enforced" trap. If the interview answer is "I set Topology Manager," the follow-up is always "which policy," and `best-effort` is the wrong answer to "guarantee."
3. **Conflating Memory Manager's GA version with DRA's.** Memory Manager (a kubelet hint provider) went GA in **v1.32**; DRA (a separate, scheduler-level device API) went GA in **v1.34**. They graduated a release apart and are easy to mix up — get this wrong in an interview and it reads as not actually knowing either feature.
4. **Assuming DRA's core-API GA in v1.34 means every vendor's DRA driver is production-ready today.** GA of the API means the *interface* is stable; individual drivers — including NVIDIA's, freshly donated to CNCF — may still be maturing. Don't cite "DRA is GA" as equivalent to "DRA is battle-tested everywhere."
5. **Believing `topologyManagerScope: pod` alone fixes multi-container alignment.** Scope only changes the *unit* of alignment (pod vs. container); it does nothing if the static CPU/Memory prerequisites and Guaranteed QoS aren't also correct. The full prerequisite chain still applies per unit.

## Self-check

- **What does `single-numa-node` do when a pod's resources can't fit one NUMA node, and how does that differ from `best-effort`?**
  **Answer:** `single-numa-node` **rejects the pod at admission** — the merged hint is unsatisfiable (no single NUMA node holds all the requested CPUs/memory/devices), so the kubelet fails admission and the pod is `Terminated` with `reason: TopologyAffinityError`; it does not go `Pending`/reschedule, since the kubelet already owns it. `best-effort` computes the *same* hint but **admits the pod regardless** — it stores the preferred affinity so providers try to honor it, but never rejects. So on identical inputs, `single-numa-node` = "aligned to one node or dead," `best-effort` = "prefer alignment but run misaligned anyway." best-effort is the silent-waste policy.
- **Why can GPU↔CPU alignment silently fail even with Topology Manager enabled (the TopologyInfo trap)?**
  **Answer:** Device Manager can only emit a NUMA hint for a GPU if the device plugin populated `TopologyInfo` (`NUMANodes`) at registration. If the NVIDIA plugin advertises GPUs without it — old versions, misconfig, or `numa_node == -1` on VMs/hardware that don't expose NUMA — the GPU contributes an "any node" hint (all bits set, preferred), which ANDs away and imposes no constraint. Topology Manager then aligns only CPU and memory, admits the pod, and the GPU can sit on a different NUMA node than the pinned CPUs. `single-numa-node` looks like it's working because it perfectly aligned everything it was *given a hint for*. Fix: verify the plugin publishes topology (check `/sys/bus/pci/devices/<bdf>/numa_node ≠ -1` and `nvidia-smi topo -m` vs the pod's pinned NUMA node).
- **What does CPU Manager `full-pcpus-only` add, and why does it matter for SMT/hyperthreaded nodes?**
  **Answer:** On an SMT/hyperthreaded node, a Kubernetes "CPU" is a *hardware thread*, and two sibling threads share one physical core's execution units. The plain `static` policy can hand a pod threads that split a physical core with a *different* pod's thread — noisy-neighbor contention on the shared core even though both pods think they have "exclusive" CPUs. `full-pcpus-only` forces CPU allocation in **whole-physical-core units** (both SMT siblings assigned together) and rejects pods whose CPU count isn't a multiple of threads-per-core with `SMTAlignmentError`. This gives clean, contention-free cores — important for GPU feeder threads and latency-sensitive work, where a stolen sibling thread shows up as jitter and lost throughput.
- **What Kubernetes version did DRA reach GA in, and structurally what changed in how topology constraints are expressed compared to the device-plugin model?**
  **Answer:** DRA reached GA in **v1.34** (September 2025). Structurally, it moves the topology constraint from a flat, kubelet-only hint (`TopologyInfo.NUMANodes`, visible only to Device Manager at admission) to a `ResourceSlice`/`ResourceClaim` model where devices publish rich attributes (architecture, PCIe root complex, NUMA node) and workloads declare inter-device constraints (e.g. "GPU and NIC on the same PCIe root complex") that `kube-scheduler` itself can see and filter/score on *before* binding — closing the scheduler-blindness gap that makes the device-plugin era's failures (like the TopologyInfo trap) possible in the first place.
- **In DRA's ResourceSlice/ResourceClaim model, where does a GPU+NIC co-location requirement live, and how is that different from where it lives in the device-plugin/TopologyInfo model?**
  **Answer:** In DRA, the co-location requirement is an explicit `constraints` entry on a `ResourceClaim`, referencing a shared device attribute (e.g. `pcieRootComplex`) published by the DRA driver on a `ResourceSlice` — a first-class, scheduler-visible object the workload author writes deliberately. In the device-plugin model, there is no equivalent object: co-location is an emergent property of whether the device plugin happened to populate `TopologyInfo` correctly, checked only by the kubelet's Device Manager at admission, with no way for the workload author to declare the requirement directly.

## Connections & what's next

This lesson turns lesson 04's fixed hardware layout into an enforceable Kubernetes guarantee — the "hardware is correct" fact from the 8-GPU server architecture lesson only pays off if the orchestrator respects it, and that's exactly what Topology Manager plus the static CPU/Memory policies do (or silently fail to do, via the `TopologyInfo` trap). The same underlying question — "is this resource actually co-located with the GPU it's supposed to feed?" — reappears immediately in **lesson 06**, except the resource in question is a storage drive instead of CPU/memory: a GPU can be perfectly NUMA-aligned by every mechanism in this lesson and still stall badly because its dataset lives on an NVMe drive across the socket boundary. Both lessons feed directly into the **lesson 08 capstone**, where you reconcile four tools' output into one topology diagram on a real node — the negative test you ran in this lesson's worked example (force a rejection, then relax the policy and watch it silently admit) is exactly the kind of tool-disagreement skill that capstone rewards.

## References & further reading

**Primary sources**
- Kubernetes — "Node Resource Managers" — `https://kubernetes.io/docs/concepts/policy/node-resource-managers/` — canonical reference for Topology Manager policies/scopes and how CPU/Memory/Device Managers act as hint providers; read for guarantee-vs-attempt semantics.
- Kubernetes — "Memory Manager" — `https://kubernetes.io/docs/tasks/administer-cluster/memory-manager/` — the `Static` policy, the `reservedMemory` requirement and its consistency rules, and the correct GA version (v1.32).
- Kubernetes — "Control CPU Management Policies on the Node" — `https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/` — the `static` policy, exclusive-CPU allocation, and `full-pcpus-only`/SMT alignment options.
- Kubernetes Blog — "Kubernetes v1.34: DRA has graduated to GA" — `https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/` — read for the primary-source GA date and what graduated with it.

**Real-world engineering blogs**
- Ronak Nathani — "Keeping GPU Workloads NUMA-Local in Kubernetes" — `https://ronaknathani.com/blog/2026/05/keeping-gpu-workloads-numa-local-in-kubernetes/` — what it shows: a measured >30% p99 tail-latency cost of skipping single-socket alignment, and the operational checklist for getting it right.
- Google Cloud Blog — "Kubernetes device management with DRA Dynamic Resource Allocation" — `https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-device-management-with-dra-dynamic-resource-allocation` — what it shows: the `ResourceSlice`/`ResourceClaim` model explained by a major cloud vendor, with the GPU+NIC PCIe-root-complex example.
- CNCF Blog — "Understanding dynamic resource allocation in Kubernetes" — `https://www.cncf.io/blog/2026/07/01/understanding-dynamic-resource-allocation-in-kubernetes/` — what it shows: a vendor-neutral framing of why DRA matters, confirming it's a cross-ecosystem shift, not one cloud's feature.

**Deeper dives**
- Kubernetes Contributors Blog — "Spotlight on WG Device Management" — `https://www.kubernetes.dev/blog/2026/06/24/wg-device-management-spotlight-2026/` — working-group-level view of where device/topology management is headed post-DRA-GA, useful for tracking the field beyond this lesson's snapshot.
