---
lesson: "A02.7"
title: "GPU and RDMA networking (platform-integration angle)"
module: "A-02"
concept: "provisioning + operating RDMA on K8s"
status: not-started
est_time: "4 hrs"
prev: "06-service-mesh.md"
next: "08-network-observability-and-debugging.md"
artifacts: ["RDMA node acceptance-test runbook"]
sources: 14
---

# A02.7 · GPU and RDMA networking (platform-integration angle)

> **Concept.** The fabric can be physically perfect and a single misconfigured device plugin, NUMA-misaligned VF, or un-drained flapping NIC still silently halves training throughput — this lesson is the operations layer over the module-09 fabric.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits

Lesson 06 ended on a hard placement rule: mesh the inference frontend, never the RDMA data path, because a sidecar has no socket to intercept and would multiply a collective's latency by two orders of magnitude. Lesson 05 ended on the other half of the same statement: a GPU node runs **two** networks, and "RDMA bypasses the CNI" only means something once you can enumerate what is being bypassed — the veth pair, the bridge, the routing decision, the conntrack entry, the policy evaluation, the 50 bytes of VXLAN header.

This lesson is that second network, from the platform team's side of the desk. Module 09 owns the fabric itself: RoCEv2 and InfiniBand transport, PFC/ECN/DCQCN, GPUDirect's DMA path, rail-optimised Clos. What it does not own is the part that decides whether a training pod ever *reaches* that fabric. Between a physically perfect fabric and a job running at line rate sit about six independent conditions, each configured in a different Kubernetes object, each failing silently, and every one of them yours. **This lesson is how you configure those six things, how you prove each one is true on a specific node, and how you catch the ones that quietly stop being true as a fleet ages.**

## Why this matters

The failure mode that defines this topic is that *nothing errors*. A pod with a misconfigured RDMA path does not crash-loop, does not log, does not fire an alert, and does not fail a health check. It runs. The loss curve just takes longer, and the invoice does not change: on a cluster billing on the order of tens of dollars per GPU-hour, a node delivering half its collective bandwidth still bills at 100%.

The concrete cost is easy to state and worth stating in exactly these terms, because it is the argument that funds the acceptance gate you are about to build. A 2-node, 16-GPU job whose inter-node phase runs at 22 GB/s instead of 46 GB/s does not run 2× slower overall — but the inter-node phase does, and because **a collective is a barrier, every rank waits for the slowest one**. One misaligned NIC on one node taxes all sixteen GPUs. Multiply by a 1,000-node fleet where 2% of nodes have drifted after a firmware update, and the arithmetic that follows in the Worked example produces a number with six figures in it per month.

Staff-level here means three things. You can state what "this pod has RDMA" decomposes into, and check each part with one command. You can reason about the device-plugin-to-DRA migration as a live-fleet programme rather than a feature flag. And you can produce a gate that catches a bad node *before* it joins the schedulable pool, because after that point the only observer is a researcher wondering why this week is slower than last week.

## What's new here (calibration)

- **Module 09 boundary.** RoCEv2/InfiniBand transport mechanics, PFC/ECN/DCQCN, the queue-pair state machine, GPUDirect's DMA path, and Clos/rail topology maths are 09's material and are referenced, not re-derived. Where this lesson quotes one of 09's numbers it says so.
- **09.6 boundary.** 09.6 walks the *component map* — which of Multus, the SR-IOV Network Operator, the SR-IOV device plugin, the RDMA shared device plugin and the NVIDIA Network Operator does what, and the allocation path traced through source. This lesson takes that as given and asks the operator's questions: which model do I run, what exactly do I write in each object, and **how do I prove the result is correct on this node right now?**
- **New: "the pod has RDMA" decomposed into six checkable conditions**, each with the command that answers it and the exact output that means pass.
- **New: the silent-fallback decision tree.** Every way GPUDirect can fail to engage while everything still "works", ordered by how often you will meet it, with the symptom that distinguishes each branch.
- **New: the memory-registration contract** — `IPC_LOCK`, `RLIMIT_MEMLOCK`, and the exact warning libibverbs prints — which is the single most common reason a correctly-scheduled RDMA pod performs like a TCP pod.
- **New: `numVfs` as a capacity calculation** with the real ceilings (the PF's own `sriov_totalvfs`, the operator webhook's vendor caps, firmware `NUM_OF_VFS`) and the bandwidth arithmetic that says what a VF is actually worth.
- **New: DRA expressed as the thing device plugins structurally cannot do** — a `matchAttribute` constraint that co-allocates a GPU and a NIC on the same PCIe root, with the real attribute names from the upstream driver.
- **New: the acceptance gate as arithmetic**, not a checklist: the busbw formula, the expected value derived from the node's own hardware, and the drop rate at which a lossless fabric's goodput collapses.

## Core concepts

### 1. What "this pod has RDMA" actually decomposes into

Start here, because almost every incident in this lesson is one of these six conditions being false while the other five are true.

A training process performing an RDMA transfer needs, in order:

1. **A device node to open.** `ibv_open_device()` opens `/dev/infiniband/uverbsN`. If that character device is not in the container's device cgroup, `ibv_get_device_list()` returns empty and the job fails *loudly* — this is the one friendly failure in the set.
2. **The right device.** The pod may see several `mlx5_*` devices, only one of which was allocated to it. Which one the process picks is a property of the RDMA subsystem's namespace mode and of `NCCL_IB_HCA`, not of the scheduler.
3. **A netdev with an address**, because RoCE addresses queue pairs by GID, and the GID table is derived from the interface's IP addresses. On InfiniBand, a LID assigned by the subnet manager plays that role.
4. **Permission to pin memory.** Registering a memory region pins pages. Without `IPC_LOCK` or a raised `RLIMIT_MEMLOCK`, registration of a large region fails or is silently limited.
5. **A GPU close enough to the NIC** that the HCA can DMA directly against HBM. If not, the transfer still works — through host memory, at roughly half the bandwidth.
6. **A fabric that is up, lossless, and at the right MTU**, which is module 09's subject but shows up here as three counters you read.

Conditions 1 and 3 fail loudly. Conditions 2, 4, 5 and 6 fail silently. **That asymmetry is the whole reason this lesson exists.**

Here is what all of it looks like inside one pod, on one node:

```
  ONE TRAINING POD ON ONE GPU NODE — what is attached, and by whom
  ═══════════════════════════════════════════════════════════════

                        ┌───────────────── pod network namespace ─────────────┐
   default CNI          │                                                     │
   (Cilium/Calico) ────▶│  eth0   10.244.7.19/24   mtu 1450                   │
   veth pair            │    │    default route ─▶ cluster + internet         │
   lesson 05's world    │    │    Service VIPs, conntrack, NetworkPolicy      │
                        │    │    NCCL bootstrap, gRPC, checkpoint control    │
                        │                                                     │
   sriov-cni            │  net1   192.168.100.7/24  mtu 4200   ← VF netdev    │
   (moves the VF's      │    │    NO default route. On-link /24 only.         │
    netdev into netns)  │    │    Carries: nothing the kernel routes.         │
                        │    │    Its job is to give the GID table an address │
                        │                                                     │
   sriov device plugin  │  /dev/infiniband/uverbs5   ← the verbs device       │
   (isRdma: true)       │  /dev/infiniband/rdma_cm   ← connection manager     │
   mounts char devices  │  /dev/infiniband/umad5     ← management datagrams   │
                        │  /dev/infiniband/issm5     ← subnet-manager iface   │
                        │                                                     │
   rdma-cni (chained)   │  rdma device mlx5_5 MOVED into this netns           │
   + netns exclusive    │    → `ibv_devices` lists exactly one device         │
                        │                                                     │
   nvidia device plugin │  /dev/nvidia3  /dev/nvidiactl  /dev/nvidia-uvm      │
                        │                                                     │
                        │  ENV:                                               │
                        │   PCIDEVICE_NVIDIA_COM_RAIL3=0000:41:00.5           │
                        │   PCIDEVICE_NVIDIA_COM_RAIL3_INFO={"0000:41:00.5":  │
                        │     {"rdma":{"rdma_dev":"mlx5_5",                   │
                        │       "uverbs":"/dev/infiniband/uverbs5",...}}}     │
                        │   NVIDIA_VISIBLE_DEVICES=GPU-4f2a...                │
                        └─────────────────────────────────────────────────────┘

   THE DATA PATH, once the process is running:

     GPU3 HBM ──PCIe P2P──▶ VF 0000:41:00.5 ──▶ 400G port ──▶ leaf switch
        ▲                        ▲                    ▲
        │                        │                    │
     never touches           never touches       never touches
     host DRAM               eth0, the CNI,      the mesh, the
     (GPUDirect)             conntrack, or       Service VIP, or
                             any qdisc           kube-proxy
```

Read three things off that picture. **The VF's IP address is not for routing** — nothing sends IP traffic over `net1`; the address exists so the RoCE GID table has an entry, which is why an IPAM misconfiguration on the RDMA network shows up as "queue pairs will not connect" rather than "no route to host". **The character devices and the netdev are allocated by different mechanisms** — the device plugin mounts the former, the CNI moves the latter — which is why they can disagree. And **the GPU appears in this picture only as a PCIe peer**; the entire question of whether the transfer is fast is the question of how far apart those two PCIe functions are.

### 2. What the resources are actually called, and where the names come from

Everything downstream keys off strings you choose. Getting them wrong produces a pod that schedules and then has no device, so it is worth knowing exactly how each name is constructed.

The SR-IOV Network Device Plugin reads a JSON config (its `-config-file` flag, default `/etc/pcidp/config.json`, normally delivered as a ConfigMap) containing a `resourceList`. Each entry becomes one Kubernetes extended resource named **`<resourcePrefix>/<resourceName>`**. The prefix comes from the entry's `resourcePrefix` if set, otherwise from the plugin's `-resource-prefix` flag, **whose default is `intel.com`** — which is why so many clusters advertise Mellanox VFs under an Intel-branded name, and why every write-up sets the prefix explicitly.

```json
{
  "resourceList": [
    {
      "resourceName": "rail3",                    // → nvidia.com/rail3
      "resourcePrefix": "nvidia.com",             // else defaults to intel.com
      "selectors": [{
        "vendors":  ["15b3"],                     // Mellanox PCI vendor ID
        "devices":  ["101e"],                     // ConnectX-7 VF device ID
        "drivers":  ["mlx5_core"],                // kernel netdev driver on the VF
        "pfNames":  ["ens5f0np0#0-7"],            // PF name, VF index range 0..7
        "isRdma":   true                          // ← the load-bearing line
      }]
    }
  ]
}
```

Four notes on that, all from the plugin's own documentation and source:

- **`isRdma: true` is what mounts the character devices.** Without it you get a netdev in the pod and no `/dev/infiniband/*`, and the job fails at `ibv_get_device_list()`. With it, the plugin resolves the PCI address to its RDMA device via sysfs (`/sys/class/infiniband*`) and returns a `DeviceSpec` for **every** matching character device: the `uverbsN` node from `/sys/class/infiniband_verbs`, the `umadN` and `issmN` nodes from `/sys/class/infiniband_mad`, any `ucm` node from `/sys/class/infiniband_cm`, and `/dev/infiniband/rdma_cm` if it exists on the host. (Verified in `sriov-network-device-plugin` `pkg/devices/rdma.go` and the `Mellanox/rdmamap` library it calls.)
- **`pfNames` supports a VF range** in the form `<PFName>#<first>-<last>,<single>,…`, zero-based. This is how you split one PF's VFs into two pools — say, four VFs for training and four for a storage network — without touching the hardware.
- **Selectors are evaluated in list order and a device joins only the first pool it matches.** Two overlapping pools do not both get the device; the earlier one wins, silently.
- **`excludeTopology: true` suppresses the NUMA hint** the plugin would otherwise attach to each device. It exists for devices where the reported NUMA node is wrong or meaningless — and it is a footgun on a GPU node, because a resource with no topology hint cannot be aligned by Topology Manager (§7). If you ever find alignment silently not happening, check whether someone set this.

The result on the node is an ordinary extended resource, countable and opaque:

```console
$ kubectl get node gpu-a3-014 -o jsonpath='{.status.allocatable}' | jq
{
  "cpu": "224",
  "memory": "2113929216Ki",
  "nvidia.com/gpu": "8",
  "nvidia.com/rail0": "8",
  "nvidia.com/rail1": "8",
  ...
  "nvidia.com/rail7": "8",
  "pods": "110"
}
```

**Read the shape, not the numbers.** Eight rail resources, eight of each, on a node with eight GPUs: that is one PF per GPU, each carved into eight VFs. Kubernetes now knows there are eight of `rail3` and eight of `nvidia.com/gpu`. It does **not** know that `rail3` is the NIC that GPU 3 can reach without crossing a socket. That missing sentence is §7.

Inside the container, the allocation is visible as environment variables the plugin injects, named `PCIDEVICE_` + the uppercased resource name with `.` and `/` replaced by `_`:

```console
$ env | grep PCIDEVICE
PCIDEVICE_NVIDIA_COM_RAIL3=0000:41:00.5
PCIDEVICE_NVIDIA_COM_RAIL3_INFO={"0000:41:00.5":{"rdma":{"rdma_dev":"mlx5_5",
  "uverbs":"/dev/infiniband/uverbs5","umad":"/dev/infiniband/umad5",
  "issm":"/dev/infiniband/issm5","rdma_cm":"/dev/infiniband/rdma_cm"}}}
```

That `rdma_dev` field is the single most useful string in the container for debugging: it is the device the platform *intended* you to use. Compare it with what the process actually opened and you have caught the §5 failure before it costs anything.

### 3. The two allocation models, as real objects

Two models are in production use, and choosing between them is a per-node-pool decision, not a default.

| | RDMA shared device plugin | SR-IOV network device plugin | DRA (dranet / vendor DRA driver) |
|---|---|---|---|
| What the pod gets | the **PF itself**, shared | a dedicated **VF** with its own PCI function | a dedicated device, allocated by attribute |
| Isolation | none — every holder sees the same HCA | per-VF: own netdev, own PCI address, own VF-level QoS | as SR-IOV, plus structural co-allocation |
| What limits concurrency | `rdmaHcaMax`, a **counter**, not a quota | real VF count (`numVfs`) | published `ResourceSlice` devices |
| Topology expressible? | no | via `TopologyInfo` hints + Topology Manager (advisory, node-local) | **yes** — as a scheduling constraint |
| Fits | single-tenant nodes, dev, inference | production multi-tenant training | production where mis-scheduling must be structurally impossible |
| Cost | none; no VF provisioning, no reboot | VF provisioning, PF reconfiguration, sometimes a reboot | a driver per device class; newer API surface |

The shared model's trap is worth naming precisely because the config file looks like it is doing capacity management and is not. `rdmaHcaMax: 63` does not create 63 of anything, does not partition bandwidth, does not partition queue pairs, and does not partition memory-registration capacity. It is the integer the plugin advertises. Sixty-three pods can hold "one" each and all of them are using the same physical HCA, at whatever share of it they can take. **On a multi-tenant training cluster, "allocated" and "isolated" are different claims, and the shared plugin only makes the first one.**

Now the SR-IOV path, as the two objects you actually write.

**Object 1 — `SriovNetworkNodePolicy`: what the hardware becomes.** This object is consumed by the SR-IOV Network Operator's config daemon, which writes `numVfs` into the PF's `sriov_numvfs` sysfs file, binds the resulting VFs to the right driver, and generates the device-plugin ConfigMap from §2 for you. Field names and validation below are from the operator's `api/v1/sriovnetworknodepolicy_types.go` and `pkg/webhook/validate.go`.

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: rail3-vfs
  namespace: sriov-network-operator
spec:
  resourceName: rail3            # the bare name; the operator adds the prefix
  nodeSelector:
    node.kubernetes.io/instance-type: a3-ultra-8g   # never fleet-wide
  priority: 90                   # 0..99; HIGHER number = LOWER priority, and a
                                 # higher-priority policy overrides a lower one
                                 # on the same PF. Two policies at equal priority
                                 # touching the same PF is a configuration error.
  numVfs: 8                      # §6 — a capacity decision, not a flag
  nicSelector:
    vendor: "15b3"               # Mellanox
    deviceID: "1021"             # ConnectX-7 PF
    pfNames: ["ens5f0np0"]       # this PF only; one policy per rail
  deviceType: netdevice          # netdevice | vfio-pci. RDMA needs netdevice:
                                 # vfio-pci hands the whole function to a
                                 # userspace driver (DPDK) with no kernel
                                 # netdev, hence no GID table, hence no RoCE.
  isRdma: true                   # sets isRdma in the generated DP config (§2)
  linkType: eth                  # eth|ETH|ib|IB — sets the port's link layer.
                                 # Changing it rewrites firmware config and
                                 # requires a reboot. Get it right at build time.
  mtu: 4200                      # VF MTU. Must be ≥ the RoCE path MTU you want
                                 # (4096 B payload + headers). A 1500-byte VF on
                                 # a 4 KB fabric quadruples packet count for the
                                 # same bytes — see 09.3 §10.
  excludeTopology: false         # leave false on GPU nodes (§2)
```

**Object 2 — the network attachment.** The operator's `SriovNetwork` CR generates a `NetworkAttachmentDefinition` in the target namespace; you can also write the NAD directly. Both forms are shown because you will meet both, and the NAD is what actually executes.

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetwork
metadata:
  name: rail3-net
  namespace: sriov-network-operator
spec:
  resourceName: rail3            # MUST match the policy's resourceName
  networkNamespace: training     # where the generated NAD is created
  linkState: enable              # auto|enable|disable — force the VF link up
  trust: "off"                   # VF trust mode; "on" lets the VF change its own
                                 # MAC and enter promiscuous mode. Off in prod.
  spoofChk: "on"
  ipam: |
    { "type": "whereabouts", "range": "192.168.103.0/24" }
  metaPluginsConfig: |
    { "type": "rdma" }           # ← chains the RDMA CNI after sriov-cni
```

which produces, in namespace `training`:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rail3-net
  namespace: training
  annotations:
    k8s.v1.cni.cncf.io/resourceName: nvidia.com/rail3   # ← the join key
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "rail3-net",
    "plugins": [
      { "type": "sriov",
        "spoofchk": "on", "trust": "off", "link_state": "enable",
        "ipam": { "type": "whereabouts", "range": "192.168.103.0/24" } },
      { "type": "rdma" }
    ] }'
```

**The annotation is the join.** Multus reads it, then asks the kubelet's pod-resources gRPC API which PCI addresses this pod was allocated *under that exact resource name*, and injects the first unused one as `deviceID` into the `sriov` plugin's config. A typo there does not error: the lookup misses, `deviceID` is empty, and `sriov-cni` is invoked with no device to move. (09.6 traces that chain step by step; here it matters only as the thing you verify.)

And the pod, which must request the resource *and* name the attachment:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: trainer-0
  namespace: training
  annotations:
    k8s.v1.cni.cncf.io/networks: rail0-net,rail1-net,rail2-net,rail3-net,
                                 rail4-net,rail5-net,rail6-net,rail7-net
spec:
  containers:
  - name: trainer
    image: registry.internal/trainer:2025.11
    securityContext:
      capabilities:
        add: ["IPC_LOCK"]        # §5 — without this, memory registration is
                                 # capped at RLIMIT_MEMLOCK and you get a
                                 # slow job with no error
    resources:
      limits:
        nvidia.com/gpu: "8"
        nvidia.com/rail0: "1"
        nvidia.com/rail1: "1"
        nvidia.com/rail2: "1"
        nvidia.com/rail3: "1"
        nvidia.com/rail4: "1"
        nvidia.com/rail5: "1"
        nvidia.com/rail6: "1"
        nvidia.com/rail7: "1"
```

Two things about that pod spec are easy to get wrong. **The annotation and the resource requests must agree**, unless the operator's network-resources-injector webhook is enabled, in which case it patches the requests in for you from the annotation — meaning a correct pod spec looks different depending on a cluster-level setting. Know which regime you are in. And **eight separate one-of-each resources, not one resource with count 8**, because each rail is a distinct PF and you want exactly one VF from each; asking for `nvidia.com/rail0: "8"` would give you eight VFs on the *same* PF, sharing one 400G port between eight GPUs.

### 4. Namespace mode: the difference between "allocated" and "used"

There is one host-level setting that decides whether any of the above is isolation. The Linux RDMA subsystem has two namespace modes, controlled by the `ib_core` module parameter `netns_mode`:

| Mode | `ib_core netns_mode` | What a container sees |
|---|---|---|
| **shared** (kernel default) | `1` | every RDMA device on the host, regardless of allocation |
| **exclusive** | `0` | only devices moved into its namespace |

In shared mode, a pod allocated `mlx5_5` can open `mlx5_2` — a different rail, possibly another tenant's VF — and `NCCL_IB_HCA=mlx5_2` will simply work. The scheduler did its job, the device plugin did its job, and the process is using the wrong NIC. **In shared mode, "we scheduled it correctly" and "it used the right NIC" are independent claims, and only the second one shows up in your bandwidth numbers.**

Exclusive mode plus the `rdma` CNI in the plugin chain fixes it: the RDMA device is moved into the pod's network namespace, and a wrong `NCCL_IB_HCA` then fails loudly instead of silently. Two operational facts you need before planning this:

- The mode is a **module parameter**, so changing it means reloading `ib_core`, and the kernel refuses the change while network namespaces exist. In practice: a reboot. The SR-IOV Network Operator exposes it as `SriovNetworkPoolConfig.spec.rdmaMode: shared|exclusive` and implements it by writing `options ib_core netns_mode=0` into a modprobe config; setting it drains and reboots the matching nodes, so it is `maxUnavailable`-gated like any other disruptive rollout.
- Namespace-aware RDMA needs **kernel ≥ 5.3** (or Mellanox OFED ≥ 4.7) and a matching `iproute2`.

The verification is one line, from inside the pod, and belongs in your acceptance runbook:

```console
$ ibv_devices
    device                 node GUID
    ------              ----------------
    mlx5_5              a088c20300f1a2b4     ← exactly one → exclusive mode
$ rdma system show                          # on the host
netns exclusive copy-on-fork on
```

If `ibv_devices` lists eight devices inside a pod that was allocated one, you are in shared mode, and the value of `NCCL_IB_HCA` is doing all the work the scheduler was supposed to guarantee.

### 5. The memory-registration contract, and why it is the most-missed line

RDMA's zero-copy property requires that the pages the NIC will DMA against cannot move. Registering a memory region therefore **pins** it, and pinning is governed by `RLIMIT_MEMLOCK`. This is where a large fraction of "we set everything up and it is still slow" tickets end.

The mechanism is not subtle, and libibverbs will even tell you, if you read the container's stderr. From `rdma-core`'s `libibverbs/init.c`, at library initialisation:

```c
static void check_memlock_limit(void)
{
    struct rlimit rlim;
    if (!geteuid())                      /* root bypasses the limit entirely */
        return;
    if (getrlimit(RLIMIT_MEMLOCK, &rlim)) { ... }
    if (rlim.rlim_cur <= 32768)
        fprintf(stderr, PFX "Warning: RLIMIT_MEMLOCK is %llu bytes.\n"
            "    This will severely limit memory registrations.\n", ...);
}
```

So: **the check only fires for non-root**, and only below **32,768 bytes** — 32 KB, which is the container-runtime default on many stacks. The message goes to stderr at library load, which in Kubernetes means it lands in `kubectl logs` for the first few lines and is then buried under training output.

Three ways to satisfy the contract, in order of preference:

1. **`securityContext.capabilities.add: ["IPC_LOCK"]`.** `CAP_IPC_LOCK` exempts the process from `RLIMIT_MEMLOCK` for `mlock()`-family operations, which is exactly the primitive memory registration uses. This is the standard answer and the one the SR-IOV operator's own RDMA guide uses in its example pod.
2. **Raise the limit at the runtime level** (containerd/CRI-O default ulimits, or the kubelet's), so every pod gets a large `memlock`. Blunter, but it survives pod specs that forget the capability.
3. Run as root. Do not.

What failure looks like: with the limit at 32 KB and no `IPC_LOCK`, small registrations succeed and large ones fail with `ENOMEM` from `ibv_reg_mr()`. NCCL treats a failed registration as "this transport is unavailable" and falls back — to a smaller registration, or to the socket transport. You get a job that runs, at TCP speed, with one warning line in the first 200 lines of a log nobody reads. **If a node passes `ib_write_bw` when you run it by hand as root and the training pod is slow, check this first**, because your hand-run test was root and the pod was not.

### 6. `numVfs` is a capacity decision — the real ceilings and the real arithmetic

"Just raise `numVfs`" is the most common wrong answer to VF-pool exhaustion. Here is what actually bounds it.

**Ceiling 1 — the PF's own advertised maximum.** Every SR-IOV-capable PF publishes it:

```console
$ cat /sys/class/net/ens5f0np0/device/sriov_totalvfs
127
$ cat /sys/class/net/ens5f0np0/device/sriov_numvfs
8
```

`sriov_totalvfs` is set by the device's SR-IOV capability structure. On Mellanox parts it is itself firmware-configurable (`mlxconfig` `NUM_OF_VFS`), so it is a *build-time* decision that a config daemon cannot change at runtime.

**Ceiling 2 — the operator's validating webhook.** The SR-IOV Network Operator rejects policies before they ever reach hardware (`pkg/webhook/validate.go`): `numVfs: 0` is refused for any non-default policy; for Mellanox NICs `numVfs` above **128** is refused outright (`MlxMaxVFs = 128`); for Intel NICs it is validated against that interface's discovered `TotalVfs`, producing `numVfs(65) in CR p1 exceed the maximum allowed value(64) interface(ens803f0)`; and with `externallyManaged: true` the requested count may not exceed the VFs that already exist. Knowing that the webhook exists saves you an afternoon: a rejected policy is an admission error on the CR, not a broken node.

**Ceiling 3 — bandwidth, which is the one that matters.** A VF is a PCIe function, not a slice of guaranteed bandwidth. All VFs of a PF share that PF's single physical port. So:

```
   PF: ConnectX-7, one 400 Gb/s port  =  50 GB/s per direction

   numVfs = 8, all eight VFs held by pods that transmit at once:
       fair-share per VF  = 50 / 8   = 6.25 GB/s
   numVfs = 2:
       fair-share per VF  = 50 / 2   = 25 GB/s
   numVfs = 1 (or 8 VFs, only one active):
       per VF             = 50       (≈48.6 measured, 09.3 §10)

   The rate limits you CAN set are per-VF and are policy, not capacity:
       SriovNetwork.spec.minTxRate / maxTxRate, in Mbps.
       maxTxRate caps a VF; it does not reserve bandwidth for anyone else.
```

**Ceiling 4 — IOMMU grouping.** VFs that land in the same IOMMU group cannot be isolated from one another by the IOMMU regardless of what the device plugin advertises. Check before you assume per-pod isolation:

```console
$ readlink -f /sys/bus/pci/devices/0000:41:00.5/iommu_group
/sys/kernel/iommu_groups/68
$ ls /sys/kernel/iommu_groups/68/devices/
0000:41:00.5            ← alone in its group. Good.
```

**The sizing rule that follows.** On a GPU node with one PF per GPU, the number of concurrent pods that can each hold one VF from *this* PF is the number you are sizing for — and on a dedicated training node that number is usually **one**. Size `numVfs` to *concurrent pods per node that need this rail*, plus small headroom for a rolling restart overlapping the old and new pod, and stop. A common production shape is `numVfs: 8` on an 8-GPU node not because eight pods will run, but so the node can also serve four 2-GPU inference pods when it is not training — a decision you should be able to state in those terms, not as "8 seemed safe".

Raising `numVfs` on a live node is also not free: it rewrites `sriov_numvfs`, which tears down and recreates every VF on that PF, which breaks every pod currently holding one. It is a drain-first operation.

### 7. Topology alignment is a scheduling gap, and it has a name

Now the failure that costs the most money.

The scheduler places a pod on a node that has a free `nvidia.com/gpu` and a free `nvidia.com/rail3`. It has **no concept** of whether that GPU and that VF share a PCIe root complex or a NUMA node — Kubernetes' own Topology Manager documentation states the limitation outright: *"The scheduler is not topology-aware, so it is possible to be scheduled on a node and then fail on the node due to the Topology Manager."* Alignment is decided **after** placement, by the kubelet, at admission time.

The kubelet's **Topology Manager** is the mechanism. Each device plugin is a *hint provider*: it reports, per device, a bitmask of the NUMA nodes that device is attached to. The Topology Manager merges the hints from CPU Manager, Memory Manager and Device Manager and picks the narrowest mask that satisfies all of them, then applies a policy:

| Policy (`--topology-manager-policy`) | Behaviour when a single-NUMA allocation is not possible |
|---|---|
| `none` (default) | no alignment attempted at all |
| `best-effort` | store the preferred hint, **admit the pod anyway** |
| `restricted` | **reject** the pod (admission failure) |
| `single-numa-node` | require a single NUMA node; **reject** if impossible |

Plus a **scope** (`--topology-manager-scope: container|pod`): `container` aligns each container independently, `pod` computes one decision for the whole pod. And two policy options worth knowing: `prefer-closest-numa-nodes` (GA in 1.32) makes `best-effort`/`restricted` prefer NUMA sets with shorter inter-node distance when more than one node is needed, and `max-allowable-numa-nodes` (GA in 1.35) lifts the default limit of **8 NUMA nodes**, above which the Topology Manager is not enabled at all because hint enumeration explodes combinatorially.

**What rejection costs is the part people miss.** A pod rejected by Topology Manager enters `Terminated` with a `TopologyAffinity` admission failure, **and the scheduler does not retry it elsewhere.** A bare pod is simply dead. This is why training jobs are submitted through a controller (Deployment, JobSet, Volcano, Kueue) that will recreate the pod, and why `single-numa-node` on a fleet with any topology irregularity produces a slow crash-loop of admission failures rather than a clean error.

And the deeper problem: `best-effort` is the setting most clusters run, and `best-effort` **admits the misaligned pod**. So the common production state is a pod that got a GPU on socket 0 and a VF on socket 1, running, with no event, no log line, and no error.

Here is what that costs, and why:

```
  WHY A CROSS-SOCKET VF IS NOT A ROUNDING ERROR

   Aligned (what you want):                 Misaligned:

   GPU3 ──PCIe switch── VF(rail3)           GPU3 ──┐
     │      "PXB" or closer                        │ PCIe root 0
     └─ NCCL: GPU Direct RDMA Enabled              ├── UPI / Infinity Fabric
        HBM ─▶ NIC ─▶ wire                         │   (inter-socket link)
        ≈48.6 GB/s measured on 400G         VF(rail3) ── PCIe root 1
        (09.3 §10)                                  │
                                             NCCL: distance > PATH_PXB
                                             ⇒ GPU Direct RDMA DISABLED
                                             ⇒ traffic diverted through the
                                               CPU local to the GPU:
                                               HBM ─▶ host DRAM ─▶ NIC ─▶ wire
                                             ≈20–25 GB/s (09.1 §5, 09.5)
```

The threshold is not folklore; it is in NCCL's source. `ncclTopoCheckGdr()` sets `netGdrLevel = PATH_PXB` by default (or `PATH_P2C` on C2C-capable Grace systems) and returns without enabling GDR when the measured GPU→NIC path type exceeds it (`src/graph/paths.cc`, NCCL v2.31.2). `NCCL_NET_GDR_LEVEL` overrides the threshold. When GDR is not available, the same file **adds an explicit intermediate step through the local CPU** — that is the host-memory staging path, chosen deliberately by the library, not a degradation of the hardware. And NCCL announces its decision, once per HCA, at init:

```
NET/IB : GPU Direct RDMA Enabled for HCA 3 'mlx5_5'
```

(The format string is `"NET/%s : GPU Direct RDMA %s for HCA %d '%s'"` in `src/graph/topo.cc`.) **That line, with `NCCL_DEBUG=INFO`, is the cheapest possible check that alignment held**, and it is the one thing you should grep for in every training job's first hundred log lines.

Why this keeps recurring across hardware generations is structural, and worth being able to say in one sentence: **the countable device-plugin API can express "one of these" but cannot express "one of these, near that one."** Everything that fixes it — Topology Manager (node-local, post-scheduling, advisory or fatal), DRA (below), gang schedulers like Volcano and Kueue that place a whole job as one topology-aware unit — is a different layer patching the same missing sentence.

### 8. DRA: expressing the constraint the device plugin cannot

Dynamic Resource Allocation replaces "count of an opaque resource" with "device matching an expression", and it is in the `resource.k8s.io` API group. Four kinds:

- **`DeviceClass`** — a cluster-admin-owned category, with CEL selectors that define membership. The analogue of a StorageClass.
- **`ResourceSlice`** — published *by the driver*, one per node (or pool), listing the devices that exist and **their attributes**. This is the object that carries the topology facts Kubernetes previously had no way to see.
- **`ResourceClaim`** — a request for devices: which classes, how many, which selectors, and — the important one — **constraints across requests**.
- **`ResourceClaimTemplate`** — generates a per-pod `ResourceClaim`, lifecycle-bound to the pod, the way a StatefulSet's `volumeClaimTemplates` generate PVCs.

The network driver publishes real attributes. From `dranet`'s `pkg/apis/attributes.go`, every interface it discovers carries `dra.net/ifName`, `dra.net/pciAddress`, `dra.net/mac`, `dra.net/mtu`, `dra.net/state`, `dra.net/type`, `dra.net/numaNode`, `dra.net/sriov`, `dra.net/sriovVfs`, `dra.net/virtual`, and `dra.net/rdma` — plus the standard `resource.kubernetes.io/pcieRoot`, which is the one that matters here.

```yaml
# The admin defines the category once.
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: rdma
spec:
  selectors:
  - cel: { expression: 'device.driver == "dra.net"' }
  - cel: { expression: 'device.attributes["dra.net"].rdma' }
---
# The workload states intent — and the constraint the device plugin could not express.
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-nic-aligned
spec:
  spec:
    devices:
      requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 2
      - name: nic
        exactly:
          deviceClassName: rdma
          count: 2
      constraints:
      - matchAttribute: "resource.kubernetes.io/pcieRoot"   # ← the whole point
---
apiVersion: v1
kind: Pod
metadata: { name: trainer-0, namespace: training }
spec:
  resourceClaims:
  - name: aligned
    resourceClaimTemplateName: gpu-nic-aligned
  containers:
  - name: trainer
    image: registry.internal/trainer:2025.11
    resources:
      claims:
      - name: aligned
```

(The `matchAttribute` form above is taken from the upstream `dranet` repository's own NVIDIA example, `examples/demo_nvidia_dranet/resourceclaims.yaml`.)

**Read what changed.** `matchAttribute` says: whatever GPUs and NICs you pick, they must agree on the value of this attribute. That is a *scheduling* constraint, evaluated by the scheduler while choosing a node, not a kubelet admission check applied after the node is already chosen. The pod is placed on a node where the combination exists, or it stays pending — which is a far better failure than being admitted misaligned, and a far better failure than being killed with `TopologyAffinity` after placement.

**The operational rule that governs migration: DRA and the device-plugin model cannot both own the same devices on the same node.** They are two drivers claiming the same PCI functions; enabling the DRA driver means the classic device plugin must stop advertising them. That makes this a dataplane-style migration, not a flag flip, and it is staged **per node pool**:

1. Enable the feature gates and the `resource.k8s.io` APIs cluster-wide (control-plane change, no workload effect).
2. Roll the DRA driver to a **canary pool** with the device plugin removed from that pool only.
3. Re-run the acceptance gate (§9) on the canary pool. Not "does a pod start" — the full busbw number.
4. Rewrite workload manifests for that pool: `resources.limits: nvidia.com/rail3: 1` becomes `resourceClaims` + `resources.claims`. This is a change every team that submits jobs must make, which is why it is a programme and not a change window.
5. Widen one pool at a time, keeping a rollback pool on the old model until the last workload has migrated.

Treat mixed-model *within* one pool as an error state, because a pod that lands on a device-plugin node with a DRA-shaped manifest simply stays pending, with an event that names a resource nobody recognises.

### 9. The acceptance gate, as arithmetic

A node joins the schedulable pool only after passing three tests in order. Each answers a different question, and none substitutes for the others.

**Test 1 — is the endpoint healthy? `ibstat` / `ibv_devinfo`.** Reads the port state, rate and link layer straight from sysfs. `State: Active` with `Physical state: LinkUp` is pass; `LinkUp` with `State: Initializing` on InfiniBand means the cable trained but no subnet manager configured the port. A `Rate:` lower than what you bought means the port trained at reduced width or a lower generation. 09.3 §10 reads this output field by field; here it is simply the first gate, and it is cheap.

**Test 2 — does the path deliver? `ib_write_bw`.** Point-to-point, one queue pair, between the candidate node and a known-good reference node:

```console
$ ib_write_bw -d mlx5_5 -x 3 -s 65536 -F --report_gbits 192.168.103.11
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             389.42             388.77             0.741456
```

388.77 Gb/s on a 400 Gb/s link is 97% of line rate — the healthy figure for this generation (09.3 §10 reads the full banner). Two failure fingerprints worth memorising: **≈50% of line rate** points at a PCIe link trained at half width, and **≈25 GB/s (200 Gb/s) on a 400G port** is the signature of a lost GPUDirect path staging through host memory. Note what this test cannot see: it uses one QP, so it measures one ECMP path, and it involves no GPU at all, so it says **nothing** about §7's alignment.

**Test 3 — does the collective deliver? `nccl-tests`.** This is the gate.

```console
$ all_reduce_perf -b 8 -e 8G -f 2 -g 8
#                                                    out-of-place
#       size   count    type   redop    time   algbw   busbw  #wrong
#        (B)    (elem)                   (us)  (GB/s)  (GB/s)
   536870912  134217728   float     sum   2891  185.71  348.20      0
  1073741824  268435456   float     sum   5673  189.27  354.88      0
```

*(Representative of a healthy 2-node, 16-rank run on the hardware described below; the column layout and the busbw definition are exact, the values are illustrative.)*

**`busbw` is the number, and it is not the same as `algbw`.** From `nccl-tests`' own source (`src/all_reduce.cu`):

```
   algbw = bytes / time                       (what the application sees)
   busbw = algbw × 2(n − 1) / n               (what the wire carries)

   n = 16 ranks  →  factor = 2 × 15 / 16 = 1.875
   189.27 GB/s × 1.875 = 354.88 GB/s
```

The factor exists because a ring all-reduce moves each byte around the ring twice, minus the piece each rank already owns. **Use `busbw` because it is comparable across job sizes; `algbw` is not**, so a threshold expressed in `algbw` silently changes meaning when someone runs the test with a different rank count.

**Now derive the expected value instead of copying someone's number**, because this derivation is also the answer to "what should a pod with *this* VF allocation be able to reach?"

Two facts do the work. First, `busbw` for a ring all-reduce is algebraically **the per-rank egress rate**: substitute `algbw = S/T` and `T = (2(n−1)/n)·S / r` into the formula and everything cancels to `busbw = r`, the rate of one ring link. Second, NCCL runs one channel per rail, and each channel's ring crosses the node boundary **once per node, on its own rail NIC** — that is exactly what rail-optimised topology buys. So each rail carries one channel's share of the traffic, and the shares add:

```
   HARDWARE:  8 GPUs, 8 rail NICs at 400 Gb/s, NVLink-4 inside the node.

   per-rail wire rate         400 Gb/s ÷ 8                    = 50.0  GB/s
   per-rail measured          97 % of line (Test 2)           ≈ 48.6  GB/s
   rails held by the pod                                      = 8

   NIC-side ceiling   = rails × per-rail bandwidth
                      = 8 × 48.6                              ≈ 389   GB/s
   NVLink-side ceiling= per-GPU NVLink-4 egress, per direction
                      = 450 GB/s (09.1 §2)                    = 450   GB/s

   busbw ceiling = min(389, 450) = 389 GB/s   ← the NIC side binds,
                                                which is the intended design
   measured 354.88 = 91 % of ceiling. Pass.

   NOW VARY THE ALLOCATION — same fabric, same GPUs, fewer VFs:

   pod gets 4 rails (VF pool half-exhausted, §6):
       ceiling = 4 × 48.6 ≈ 194 GB/s   → a 2× loss, nothing "misconfigured"
   pod gets 1 rail (a single NAD in the annotation, a common copy-paste bug):
       ceiling = 1 × 48.6 ≈ 49  GB/s   → an 8× loss, and the job still runs

   Threshold: FAIL the node below 90 % of the fleet's own p50 busbw for the
   same parameters. Absolute numbers age with every hardware generation;
   the fleet median does not, and it silently absorbs NCCL-version changes.
```

**That last block is the answer to most "why is this job slower than that job" tickets**, and it needs no measurement at all: count the rails the pod actually holds and multiply.

**And the fabric-health check that goes with it**, because a node can pass all three tests on a quiet fabric and fail under load. Read the per-priority counters on the RDMA interface (mlx5 driver names, 09.4 §12):

```console
$ ethtool -S ens5f0np0 | grep -E 'prio3_(pause|discard|marked)'
     rx_prio3_pause: 41927
     tx_prio3_pause: 1290418
     rx_prio3_buf_discard: 0        ← must be zero
     rx_prio3_cong_discard: 0       ← must be zero
     rx_prio3_marked: 9147732       ← ECN working; healthy
```

**Why the discard counters are a zero-tolerance threshold, derived rather than asserted.** 09.4 §1 establishes that a reliable-connected QP recovers from loss by go-back-N — and, in the deployed NICs Microsoft measured (SIGCOMM 2016), by **go-back-0**: the sender restarts the *entire message*. Take the go-back-N case first, since it is the gentler one:

```
   bytes in flight (BDP) = 400 Gb/s × 10 µs ÷ 8       = 500,000 B
   one drop re-sends up to one BDP of data.
   with 4 KB packets, packets in flight = 500,000 / 4,096 ≈ 122

   retransmission tax ≈ p × (BDP / MTU) = p × 122

     p = 0.01 %  (1e-4)  →  1.2 %  tax   — tolerable
     p = 0.1  %  (1e-3)  →   12 %  tax   — fails a 90 %-of-median gate
     p = 0.8  %  (8e-3)  →  100 %  tax   — goodput → 0

   go-back-0 collapses far earlier, because the unit of loss is the MESSAGE:
     a 4 MB message = ~1,000 packets at 4 KB
     any single loss inside it restarts all 1,000
     ⇒ p ≈ 1/1000 already destroys most messages;
       Microsoft measured a deterministic 1-in-256 drop rate driving
       goodput to ZERO (09.4 §1).
```

So the operationally useful statement is: **a drop probability of one in a thousand is already enough to fail your acceptance gate, and one in a few hundred takes goodput to zero.** There is no "acceptable" non-zero value for `rx_prio3_buf_discard` on a lossless priority; alert on `> 0`, not on a rate.

### 10. Day-2: what invalidates a pass

The final idea, and the one that separates a runbook from a checkbox. Everything above is a statement about a node *at a moment*. Each of the following invalidates it:

- **A reboot** — `numVfs` is not persistent by itself; the operator re-applies it, but the window between boot and reconciliation is a window where the node has GPUs and no VFs, and will happily accept a pod that then fails to attach.
- **A firmware or driver update** — changes `sriov_totalvfs`, can change `linkType`, can change GID indices, and routinely changes the PCIe path enumeration NCCL reads.
- **A `SriovNetworkNodePolicy` edit** — rewriting `sriov_numvfs` destroys and recreates every VF on that PF.
- **An `rdmaMode` change** — a reboot, by construction.
- **A NIC that starts flapping.** This is the one to build tooling for: a link that flaps inside a running collective stalls every rank, and a running job cannot be partially rescued. Cordon and drain **before** the node is selected, not after. The VF's carrier state (`ip link show net1` reporting `NO-CARRIER`) and rising `rx_prio3_pause_transition` are the early signals.
- **Fleet skew.** Firmware and driver versions drift node by node as machines are replaced. Version-pin and audit as a recurring check; the interop failures this produces are subtle (a GID index that moved, an MTU that negotiated differently) and present as one slow rank.

The correct operating model is therefore: **fabric validate → collective gate → topology check, re-run on every reboot and every firmware/driver update, with the result written somewhere the scheduler can act on** — a node label the training namespace's `nodeAffinity` requires, so that failing the gate removes the node from the pool automatically rather than through a human noticing.

### 11. The silent-fallback decision tree

Pull §§4–9 together into the artefact you will actually reach for. Every branch here ends in "the job runs and is slow", which is why the tree is ordered by how cheap the discriminating test is.

```
  "THE COLLECTIVE IS SLOW AND NOTHING IS ERRORING"
  ═══════════════════════════════════════════════════════════════════════

  START ▶ NCCL_DEBUG=INFO on one rank. Grep the init block.
          │
          ├── no "NET/IB" lines at all, transport is "Socket"
          │      → RDMA never engaged. Go to A.
          │
          ├── "NET/IB : GPU Direct RDMA Disabled for HCA n"
          │      → RDMA engaged, GPUDirect did not. Go to D.
          │
          └── "Enabled" for every HCA
                 → alignment is fine. Go to E.

  ─────────────────────────────────────────────────────────────────────
  A. Is there a verbs device in the pod?
       $ ls /dev/infiniband/
       ├─ EMPTY  → the resource was allocated without isRdma, or the pod
       │           requested a non-RDMA pool. Symptom: NCCL says
       │           "NET/Socket". Fix: isRdma:true in the DP config /
       │           isRdma:true in the SriovNetworkNodePolicy.
       └─ uverbsN present → go to B.

  B. Does the pod see the device it was GIVEN?
       $ echo $PCIDEVICE_NVIDIA_COM_RAIL3_INFO   # names rdma_dev, e.g. mlx5_5
       $ ibv_devices
       ├─ ibv_devices lists MANY devices → shared netns mode (§4).
       │     The process may be on another tenant's rail. Symptom: bandwidth
       │     varies run to run and correlates with neighbours, not with you.
       │     Fix: exclusive mode + rdma-cni, or pin NCCL_IB_HCA and verify.
       └─ exactly one, and it matches rdma_dev → go to C.

  C. Can it pin memory?
       $ grep -i memlock /proc/self/limits         # inside the container
       ├─ "Max locked memory   32768   32768" and not root
       │     → registration will be limited; libibverbs warns at load.
       │     Symptom: runs, at roughly socket-transport speed. Fix: IPC_LOCK.
       └─ unlimited (or CAP_IPC_LOCK present) → go to D.

  D. How far is the NIC from the GPU?
       $ nvidia-smi topo -m        # GPU↔NIC cell: PIX/PXB = ok, PHB/NODE/SYS = not
       ├─ PHB / NODE / SYS
       │     → past NCCL's default GDR threshold (PATH_PXB). NCCL inserts a
       │       host-DRAM staging step. Symptom: ~20–25 GB/s per GPU instead
       │       of ~46–48 on a 400G rail, uniformly, from the first step.
       │       Fix: topology-aware allocation (§7/§8). Not a fabric problem.
       └─ PIX / PXB → GDR should be on; if the log still says Disabled,
             check NCCL_NET_GDR_LEVEL and the NIC's gdrSupport (peermem /
             dmabuf module loaded on the host).

  E. RDMA + GDR both engaged, still slow → it is the fabric or the peer.
       $ ethtool -S <pf> | grep -E 'prio3_(buf|cong)_discard'
       ├─ non-zero → losing packets on a lossless priority. §9's arithmetic:
       │     even 1e-3 is a double-digit tax. This is a 09.4 problem
       │     (headroom, DSCP→priority mapping at some hop), not a K8s one.
       ├─ zero, but tx_prio3_pause climbing → you are pausing upstream:
       │     a slow receiver or a genuine capacity problem.
       └─ all clean → suspect one straggler rank. Compare per-rank timings;
             a collective is a barrier, so ONE bad node sets the pace.
```

**The property that makes this tree worth memorising is that every branch has a different fingerprint in the bandwidth number.** A shared-netns wrong-rail problem is *variable*. A GDR-disabled problem is *constant and about half*. A memlock problem is *constant and about an order of magnitude*. A fabric-discard problem is *load-dependent*. You can often narrow to two branches from the shape of the graph before running anything.

## Perspectives

**Developer (the ML engineer).** They see one number: step time. Everything in this lesson is invisible to them except as "this week is slower". The highest-leverage thing a platform team can ship alongside an RDMA cluster is a **one-line assertion in the job's own startup** — grep the NCCL init block for `GPU Direct RDMA Enabled` on every rank and fail fast if any rank says `Disabled`. That converts a month of silent tax into a crash at minute one, and it costs about ten lines of shell.

**Operator.** Your unit of work is the node, and your product is a boolean: *is this node fit to join a collective?* The important design decision is that the gate's output must be **machine-readable and enforced** — a node label that the training namespace's `nodeAffinity` requires — because a runbook whose output is a human's judgement gets skipped at 3 a.m. during a capacity crunch, which is exactly when a bad node enters the pool.

**Hardware.** Almost every silent failure here is a PCIe topology fact wearing Kubernetes clothing: which root complex, which socket, which IOMMU group. `nvidia-smi topo -m` and `lspci -tv` are the ground truth, and the Kubernetes objects are a lossy projection of them. When the projection and the hardware disagree, the hardware is right and the projection needs fixing — which is precisely what DRA's attribute model is for: it lets the driver publish the PCIe fact instead of making Kubernetes guess it from a NUMA integer.

**Economics.** The defensible number is `$/GPU-hour × (1 − busbw_ratio) × GPUs × hours`, and it is worth computing once for your own fleet before you ask for time to build the gate. The subtlety that makes it larger than people expect: because a collective is a barrier, the ratio is set by the **worst** participant, so the cost of one bad node scales with the size of the job it lands in, not with one node's share. A single misaligned node in a 512-GPU job wastes 511 other GPUs' time, not its own eight.

## Real-world use cases

- **The SR-IOV Network Operator's own RDMA guide** documents the two-mode namespace design and the reboot it implies: `SriovNetworkPoolConfig.spec.rdmaMode` set to `exclusive` writes `options ib_core netns_mode=0` and **triggers a reboot of every matching node**, gated by `maxUnavailable`. Its example RDMA pod carries `capabilities.add: ["IPC_LOCK"]` — the same line §5 argues for. What it shows: the isolation property and the disruption budget are the same decision, and it is a build-time one.
- **`dranet`'s NVIDIA example** ships the exact object this lesson's §8 is about: a `ResourceClaimTemplate` requesting two GPUs and two RDMA NICs with `constraints: [{matchAttribute: "resource.kubernetes.io/pcieRoot"}]`. What it shows: the industry answer to §7's scheduling gap is now a concrete upstream API surface with a real driver behind it, not a proposal — and the attribute chosen is *PCIe root*, not NUMA node, because NUMA was always the coarse proxy for the constraint that actually matters.
- **NCCL's own topology code** (`src/graph/paths.cc`, v2.31.2) is the primary source for §7's threshold: `netGdrLevel` defaults to `PATH_PXB`, GDR is skipped when the GPU→NIC distance exceeds it, and the library then inserts an explicit routing step through the CPU local to the GPU. What it shows: host staging is a *designed fallback*, deliberately chosen and logged, which is exactly why it produces no error — and why the log line is the signal.
- **Meta's 24,000-GPU RoCE deployment** (SIGCOMM 2024) is cited here only for its operational finding that topology-aware job placement and NIC tuning were first-order contributors at scale — not for its RoCE mechanics, which are 09.4's material. *(That paper was not reachable from this environment in this pass; the point is carried with its original attribution and flagged in References.)*

## Worked example

**Scenario.** A new node pool of eight A3-class machines has arrived: 8 GPUs each, 8 × ConnectX-7 400G rail NICs plus one storage NIC, dual-socket. Bring one node through acceptance, then break it deliberately and price the break.

**Step 0 — establish the expected numbers before measuring anything.** Doing this first is what stops you from accepting whatever the hardware happens to produce.

```
   per-rail bandwidth       400 Gb/s ÷ 8               = 50.0  GB/s
   expected ib_write_bw     97 % of line (09.3 §10)    ≈ 48.6  GB/s
   busbw ceiling            8 rails × 48.6 (§9)        ≈ 389   GB/s
   NVLink-4 per-GPU egress  450 GB/s → not the binding constraint
   accept at ≥ 90 % of the fleet p50 for the same test parameters
   n = 16 ranks (2 nodes × 8) → busbw factor 2×15/16   = 1.875
   so a busbw of ~355 GB/s should print as algbw ~189  GB/s
```

**Step 1 — hardware and allocation exist.**

```console
$ cat /sys/class/net/ens5f0np0/device/sriov_totalvfs
127
$ cat /sys/class/net/ens5f0np0/device/sriov_numvfs
8
$ rdma system show
netns exclusive copy-on-fork on
$ kubectl get node gpu-a3-014 -o jsonpath='{.status.allocatable}' | jq 'with_entries(select(.key|test("nvidia")))'
{ "nvidia.com/gpu": "8", "nvidia.com/rail0": "8", ... "nvidia.com/rail7": "8" }
```

All eight rails advertised, exclusive namespace mode. If any rail were missing, the cause is almost always the device-plugin selector not matching that PF — compare `lspci -nn -d 15b3:` against the `vendors`/`devices`/`pfNames` in the ConfigMap.

**Step 2 — point-to-point, node under test against a known-good reference.**

```console
$ ib_write_bw -d mlx5_5 -x 3 -s 65536 -F --report_gbits 192.168.103.11
 65536      5000             389.42             388.77             0.741456
```

388.77 Gb/s = 48.6 GB/s = 97.2% of line. Pass. Cross-check the internal consistency, which catches a mis-parsed result: `48.6 GB/s ÷ 65,536 B = 741,000 msg/s = 0.741 Mpps`, matching the `MsgRate` column exactly.

**Step 3 — the collective gate, two nodes.**

```console
$ kubectl exec -n training trainer-0 -- env NCCL_DEBUG=INFO \
    all_reduce_perf -b 8 -e 8G -f 2 -g 8 2>&1 | tee run-aligned.log

$ grep 'GPU Direct RDMA' run-aligned.log | sort | uniq -c
      8 NET/IB : GPU Direct RDMA Enabled for HCA 0 'mlx5_0'
      ... (one per HCA, all Enabled)

$ tail -3 run-aligned.log
  1073741824  268435456   float     sum   5673  189.27  354.88      0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 354.88
```

354.88 GB/s against a derived ceiling of 389 GB/s is **91% of the hardware's maximum**, which is the healthy shape: the gap is collective protocol overhead, chunking and synchronisation, not a fault. **Pass. Record the number together with the test parameters, the rank count, the NCCL version and the driver version**, because a busbw figure without those is not comparable to anything — including to itself after the next NCCL bump.

**Step 4 — break it deliberately.** Re-run with the pod forced onto a VF from a PF on the other socket (in practice: a `SriovNetworkNodePolicy` targeting the far-socket PF, plus a pod requesting that rail with GPU 3).

```console
$ grep 'GPU Direct RDMA' run-misaligned.log | sort | uniq -c
      7 NET/IB : GPU Direct RDMA Enabled for HCA n 'mlx5_n'
      1 NET/IB : GPU Direct RDMA Disabled for HCA 3 'mlx5_5'
$ nvidia-smi topo -m | head -6
        GPU0  GPU1  GPU2  GPU3  ...  NIC5
GPU3    SYS   SYS   NV18  X     ...  SYS      ← the whole finding, one cell
$ tail -1 run-misaligned.log
# Avg bus bandwidth    : 164.30
```

**Read the collapse, and check it against §9's model rather than just believing the tool.** One channel of eight lost GDR, so that channel's inter-node crossing runs through host DRAM at ≈22 GB/s instead of ≈48.6 (09.1 §5's staged-path figure). NCCL splits the message roughly evenly across channels, so the run finishes when the **slowest** channel finishes:

```
   predicted ceiling = 7 healthy rails × 48.6  +  1 degraded rail × 22
                       ... but the channels are equal-sized, so the
                       completion time is set by the slow one:
   predicted busbw  ≈ 8 × 22  ≈ 176 GB/s
   measured                     164.30 GB/s  (93 % of that — same 7 %
                                              protocol overhead as before)
   ratio to healthy  164.30 / 354.88 = 0.463 → a 2.16× loss
```

Nothing errored. `ib_write_bw` on that same NIC still passes at 97% of line, because `ib_write_bw` does not involve a GPU. **That is precisely why Test 2 cannot be the gate.**

**Step 5 — price it.**

```
   job:            64 nodes × 8 GPUs = 512 GPUs, 14-day run
   rate:           $R per GPU-hour (use YOUR contracted rate)
   degradation:    164.30 / 354.88 = 0.463 on the inter-node phase

   Assume, conservatively, that the inter-node collective is 40 % of step
   time on this model (measure yours; it is a per-model number):
       slowdown factor = 0.60 + 0.40/0.463 = 0.60 + 0.864 = 1.464×
       14 days becomes 14 × 1.464 ≈ 20.5 days for the same work

   wasted GPU-hours = 512 × 24 × (20.5 − 14) ≈ 79,900 GPU-hours
   at $2/GPU-hour  ≈ $160,000 for ONE misaligned VF on ONE node

   Sanity check the leverage: the bad node contributes 8 of 512 GPUs
   (1.6 % of the fleet) and taxes 100 % of the job, because a collective
   is a barrier. That asymmetry — 1.6 % of the hardware, 100 % of the
   bill — is the sentence to put in front of a budget owner.
```

**Step 6 — the automatable check.** The whole of Step 4's finding is one grep. Add to the acceptance job:

```bash
# Fail the node if ANY rank reports a disabled GDR path.
if grep -q 'GPU Direct RDMA Disabled' run.log; then
  kubectl label node "$NODE" rdma-acceptance=failed --overwrite
  kubectl cordon "$NODE"
  exit 1
fi
# Fail the node if busbw is below 90 % of the fleet median for these params.
BUS=$(awk '/Avg bus bandwidth/ {print $NF}' run.log)
awk -v b="$BUS" -v m="$FLEET_P50" 'BEGIN{exit !(b < 0.9*m)}' && {
  kubectl label node "$NODE" rdma-acceptance=failed --overwrite; exit 1; }
kubectl label node "$NODE" rdma-acceptance=passed --overwrite
```

with the training namespace requiring `rdma-acceptance=passed` in its `nodeAffinity`, so failing the gate removes the node from the pool without a human in the loop.

**Deliverable:** an RDMA node acceptance-test runbook — **fabric validate** (`ibstat` state/rate, `ib_write_bw` ≥ 95% of line) → **collective gate** (`nccl-tests` busbw ≥ 90% of fleet p50 for fixed parameters) → **topology check** (`GPU Direct RDMA Enabled` on every rank, `nvidia-smi topo -m` diffed against the SKU's baseline) → **fabric health** (`rx_prioN_*_discard` == 0) — that a node must pass to carry the `rdma-acceptance=passed` label, and must re-pass on every reboot and every firmware/driver update.

## Practice

**Task A — build the two-object chain and prove each link.** Write the `SriovNetworkNodePolicy` and the `SriovNetwork` for one rail. Then verify, in order, with one command each: `sriov_numvfs` on the host; the extended resource in `kubectl get node -o json`; the generated NAD's `k8s.v1.cni.cncf.io/resourceName` annotation; `$PCIDEVICE_*_INFO` inside a running pod; and `ibv_devices` inside that pod. Record the command and the expected output for each — this list *is* half the runbook.

**Task B — break the join key.** Change one character in the NAD's `resourceName` annotation. Recreate the pod. Capture what happens: the pod starts, `net1` is absent or unconfigured, and no object reports an error. Write down the three commands that would have found it fastest.

**Task C — reproduce the memlock failure.** Run an RDMA benchmark in a pod **without** `IPC_LOCK`, as a non-root user, with the runtime's default 32 KB limit. Capture the libibverbs warning from the container's first log lines, and the resulting bandwidth. Add `IPC_LOCK`, re-run, and record the delta. This is the cheapest of all these experiments and the one most likely to be live in your own cluster right now.

**Task D — the mis-pin and the number.** Force a GPU/VF pair across sockets. Capture `NCCL_DEBUG=INFO`'s GDR lines, the `nvidia-smi topo -m` cell, and the before/after busbw. Then compute the cost for a job size your organisation actually runs, using your real GPU-hour rate.

**Task E — make the gate enforce.** Wrap Tasks A–D's checks in a job that labels the node and cordons on failure, and put `rdma-acceptance=passed` into the training namespace's `nodeAffinity`. Then test the enforcement by failing a node on purpose and confirming a training pod will not schedule onto it.

Carry the artefact into [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md), where the "all-reduce got slower" scenario reuses these signals, and into lesson 08, which teaches the general bisection discipline for finding this class of regression in production rather than in acceptance.

## Common pitfalls

- **"`ib_write_bw` passed, so the node is training-ready."** `ib_write_bw` does not touch a GPU. It cannot see the GPU↔NIC distance, so it passes at 97% of line on exactly the node whose collective runs at half speed (Worked example, Step 4). Only a collective test exercises the failure.
- **"The pod has the RDMA device, so it is using it."** In shared namespace mode the pod can see and open every HCA on the host. Allocation and use are independent claims (§4). The check is `ibv_devices` inside the pod returning exactly one device that matches `$PCIDEVICE_*_INFO`'s `rdma_dev`.
- **"More VFs is more flexibility."** VFs share the PF's single port; eight VFs on a 400G PF is 6.25 GB/s each under contention (§6). They can also land in a shared IOMMU group, which removes the isolation the device plugin implies. And raising `numVfs` on a live node recreates every VF, breaking every pod holding one.
- **"A NUMA mis-pin will show up as an error."** It produces no error anywhere: not in `kubectl describe pod`, not in dmesg, not in the CNI logs. It produces one `NET/IB : GPU Direct RDMA Disabled` line at NCCL init and a bandwidth number. With Topology Manager at `best-effort` — the common setting — the pod is *admitted* misaligned by design.
- **"Topology Manager will catch it."** Topology Manager runs on the node, after scheduling. Its strict policies (`restricted`, `single-numa-node`) do not reschedule the pod elsewhere; they terminate it with a `TopologyAffinity` admission failure that only a controller will retry. And it is disabled entirely on nodes with more than 8 NUMA nodes unless `max-allowable-numa-nodes` is raised.
- **"We'll run DRA and the device plugin side by side during migration."** Not for the same devices on the same node — they are two drivers claiming the same PCI functions. Stage per node pool, and treat a mixed pool as an error state (§8).
- **"The acceptance test is an onboarding step."** Reboots, firmware bumps, driver updates and `numVfs` edits each invalidate a prior pass (§10). A gate that runs once is a gate that was true once.

## Self-check

- **A pod has an RDMA device, a working VF, and passes `ib_write_bw` at 97% of line, but its all-reduce runs at half the expected busbw with no errors. Name the two most likely causes and the one command that distinguishes them.** *Answer:* Either the GPU↔NIC path exceeds NCCL's GDR threshold (so traffic stages through host DRAM at ~20–25 GB/s instead of ~46–48), or the process is registering memory under a 32 KB `RLIMIT_MEMLOCK` and falling back. `NCCL_DEBUG=INFO` distinguishes them in one grep: `GPU Direct RDMA Disabled for HCA n` names the first; its absence, plus libibverbs' `RLIMIT_MEMLOCK is 32768 bytes` warning in the pod's first log lines, names the second. `ib_write_bw` cannot see either, because it involves no GPU and was probably run as root.

- **Why is `busbw` the acceptance number rather than `algbw`, and what is the exact relationship?** *Answer:* `algbw = bytes/time` is what the application sees and it changes with rank count, so a threshold in `algbw` silently means something different for a different job size. `busbw = algbw × 2(n−1)/n` for all-reduce (from `nccl-tests` `src/all_reduce.cu`), which normalises for the ring moving each byte around twice minus the piece each rank owns. At n=16 the factor is 1.875. `busbw` is therefore comparable across job sizes and is the fabric-plus-topology health signal.

- **What does `isRdma: true` actually do, and what breaks without it?** *Answer:* It makes the SR-IOV device plugin resolve the allocated PCI function to its RDMA device through sysfs and return `DeviceSpec`s for every associated character device — `/dev/infiniband/uverbsN`, `umadN`, `issmN`, any `ucm` node, and `/dev/infiniband/rdma_cm` — so the container's device cgroup admits them, and inject a `PCIDEVICE_..._INFO` env var naming them. Without it the pod gets a netdev and no verbs device; `ibv_get_device_list()` returns empty and NCCL falls back to its socket transport. This is one of the few loud failures in the lesson.

- **Kubernetes has no way to say "a NIC near *that* GPU" with device plugins. What replaces it, and what specifically changes about *when* the constraint is evaluated?** *Answer:* DRA. The driver publishes each device in a `ResourceSlice` with attributes including `resource.kubernetes.io/pcieRoot`; a `ResourceClaim`/`ResourceClaimTemplate` requests a GPU and a NIC and adds `constraints: [{matchAttribute: "resource.kubernetes.io/pcieRoot"}]`. The change that matters is timing: this is evaluated by the **scheduler while choosing a node**, so a pod that cannot be aligned stays pending. Topology Manager, by contrast, runs on the kubelet *after* placement, and either admits the misaligned pod (`best-effort`) or kills it without rescheduling (`restricted`/`single-numa-node`).

- **Why is there no acceptable non-zero value for `rx_prio3_buf_discard`?** *Answer:* Because RDMA's loss recovery is a cliff, not a slope. On a 400G link with ~10 µs fabric RTT the BDP is ~500 KB, so one drop re-sends up to ~122 packets of 4 KB; the retransmission tax is roughly `p × BDP/MTU`, which is ~12% at p=10⁻³ and total at p≈8×10⁻³. And the deployed NICs Microsoft measured recover by **go-back-0** — restarting the whole message — where a 4 MB message of ~1,000 packets is destroyed by p≈10⁻³, and a deterministic 1-in-256 drop rate drove goodput to zero (09.4 §1). A counter that is "only slightly" non-zero already fails a 90%-of-median gate.

- **Your cluster runs Topology Manager at `best-effort` and a node has GPUs on both sockets with rail NICs on both sockets. What is the realistic failure, and what would `single-numa-node` change?** *Answer:* At `best-effort`, a pod whose GPU and VF cannot be aligned is **admitted anyway** with the preferred hint recorded and unused — so the realistic failure is a running, silent, half-speed job. `single-numa-node` would instead reject the pod at admission with a `TopologyAffinity` error, which is louder but is not a fix: the scheduler does not retry it elsewhere, so a bare pod dies and a controller-managed pod may thrash between nodes. The actual fix is making the constraint visible to the scheduler (DRA `matchAttribute`, or gang scheduling that places the whole job topology-aware).

## Connections & what's next

This lesson closes lesson 06's placement rule from the other side: 06 said "never mesh the RDMA path", and this lesson defines exactly what that path is made of and how you keep it healthy. It sits on lesson 05's enumeration of what the primary CNI does, which is what the RDMA path is bypassing. It leans on module 09 for the fabric itself — 09.3 for kernel bypass, queue pairs, GPUDirect and the tool output formats; 09.4 for losslessness, PFC/ECN and the counter semantics; 09.6 for the component map and the allocation path traced through source — and it deliberately does not repeat any of it.

It feeds forward into **lesson 08** in two ways. The acceptance signals built here (busbw, the GDR log line, the per-priority discard counters) are the terminal leaves of 08's decision tree — the fabric branch, where 08's own tooling structurally cannot reach, because Hubble and pwru instrument the kernel network stack and RDMA's entire value is bypassing it. And the discipline is the same one: a fixed, ordered sequence of cheap discriminating tests beats tool-flailing, whether you are gating a node or bisecting an incident.

## References & further reading

**Primary sources — read directly from upstream source and documentation in this pass**

1. `k8snetworkplumbingwg/sriov-network-device-plugin` — `README.md` (config schema, selector semantics and evaluation order, `resourcePrefix` default `intel.com`, `pfNames` range syntax, `PCIDEVICE_*` env var construction) and `pkg/devices/rdma.go` / `pkg/infoprovider/rdmaInfoProvider.go` (what `isRdma` mounts). https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin
2. `Mellanox/rdmamap` — `rdma_map.go`, the library the device plugin calls to enumerate a device's character devices. Source of the exact device list in §1 and §2: `uverbs*` from `/sys/class/infiniband_verbs`, `umad*`/`issm*` from `/sys/class/infiniband_mad`, `ucm*` from `/sys/class/infiniband_cm`, plus `/dev/infiniband/rdma_cm`. https://github.com/Mellanox/rdmamap
3. `k8snetworkplumbingwg/sriov-network-operator` — `api/v1/sriovnetworknodepolicy_types.go` and `api/v1/sriovnetwork_types.go` for every field and validation marker quoted in §3; `pkg/webhook/validate.go` for the `numVfs` ceilings in §6 (`MlxMaxVFs = 128`, Intel validated against the interface's `TotalVfs`, the exact error strings); `doc/rdma-configuration.md` for `rdmaMode` and the `IPC_LOCK` example pod. https://github.com/k8snetworkplumbingwg/sriov-network-operator
4. `k8snetworkplumbingwg/rdma-cni` — README and examples. Source of §4's namespace-mode requirement, the `rdma system set netns exclusive` / `options ib_core netns_mode=0` forms, the kernel ≥ 5.3 (or MOFED ≥ 4.7) requirement, and the chained-NAD example. https://github.com/k8snetworkplumbingwg/rdma-cni
5. `linux-rdma/rdma-core` — `libibverbs/init.c`, `check_memlock_limit()`. Source of §5's exact behaviour: the check is skipped for euid 0 and warns only at `rlim_cur <= 32768`. https://github.com/linux-rdma/rdma-core
6. `google/dranet` — `pkg/apis/attributes.go` (the full `dra.net/*` attribute list) and `examples/demo_nvidia_dranet/resourceclaims.yaml` (the `matchAttribute: resource.kubernetes.io/pcieRoot` claim reproduced in §8). https://github.com/google/dranet
7. `NVIDIA/nccl` v2.31.2 — `src/graph/paths.cc` (`ncclTopoCheckGdr()`, `netGdrLevel = PATH_PXB` default, the explicit CPU inter-step added when GDR is unavailable) and `src/graph/topo.cc` (the `NET/%s : GPU Direct RDMA %s for HCA %d '%s'` log line). https://github.com/NVIDIA/nccl
8. `NVIDIA/nccl-tests` — `src/all_reduce.cu`, `AllReduceGetBw()`. Source of the `busbw = algbw × 2(n−1)/n` formula in §9. https://github.com/NVIDIA/nccl-tests
9. Kubernetes documentation, "Control Topology Management Policies on a node" — the four policies, the two scopes, `prefer-closest-numa-nodes` (GA 1.32) and `max-allowable-numa-nodes` (GA 1.35), the 8-NUMA-node default limit, and the stated limitation that the scheduler is not topology-aware. *Read from the `kubernetes/website` source repository, because `kubernetes.io` was unreachable from this environment.* https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/
10. Kubernetes documentation, "Dynamic Resource Allocation" — DeviceClass / ResourceClaim / ResourceClaimTemplate / ResourceSlice semantics, CEL selectors, and the claim/template distinction used in §8. *Also read from the `kubernetes/website` source repository for the same reason.* https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/

**Corrections made in this rewrite**

11. The previous version stated that a NUMA-aligned NIC gets "8–16 NCCL channels" and a SYS-distant one "~2", implying a 4–8× channel-count collapse. **That figure could not be substantiated in NCCL's source**, where `MAXCHANNELS` is 64 and the channel count is chosen by the topology search rather than by GDR availability. The verified mechanism is different and is what this rewrite teaches: NCCL disables GPUDirect above `PATH_PXB` and routes the transfer through the CPU local to the GPU, costing roughly 46–48 GB/s → 20–25 GB/s on a 400G rail (module 09.1 §5, 09.5). The observable is the `GPU Direct RDMA Disabled` init line, not a channel count.
12. The previous version cited "up to 127 VFs per PF on many Mellanox ConnectX NICs" as the ceiling. The checkable ceilings are: the PF's own `sriov_totalvfs` (firmware-set via `mlxconfig NUM_OF_VFS` on Mellanox parts), and the SR-IOV Network Operator webhook's caps — **128** for Mellanox and the discovered `TotalVfs` for Intel. §6 quotes these instead.

**Real-world engineering and vendor documentation**

13. NVIDIA Developer Blog, "Streamlining Kubernetes Networking in Scale-out GPU Clusters with the new NVIDIA Network Operator 1.0" — first-party account of the Multus / MOFED-driver-container / RDMA-shared-device-plugin / SR-IOV-device-plugin composition. *Not fetched in this pass (`developer.nvidia.com` and `docs.nvidia.com` were blocked by the egress proxy); listed as further reading, not relied upon for any fact above.* https://developer.nvidia.com/blog/streamlining-kubernetes-networking-in-scale-out-gpu-clusters-with-the-new-nvidia-network-operator-1-0/
14. Meta Engineering / SIGCOMM 2024, "RoCE networks for distributed AI training at scale" — cited only for topology-aware job scheduling and NIC tuning at 24,000-GPU scale; the RoCE mechanics belong to 09.4. *Not fetched in this pass; carried with its original attribution and not relied upon for any number above.* https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/
