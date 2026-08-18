---
lesson: "09.6"
title: "Kubernetes multi-NIC for RDMA (Multus/SR-IOV/Network Operator)"
module: "09"
concept: "Kubernetes multi-NIC for RDMA"
status: not-started
est_time: "7h"
prev: "05-gpudirect-and-sharp.md"
next: "07-bandwidth-as-cost.md"
artifacts: []
sources: 13
---

# 09.6 · Kubernetes multi-NIC for RDMA (Multus/SR-IOV/Network Operator)

> **Concept.** RDMA reaches a pod through a *second* NIC that the default CNI never touches — Multus attaches it, a device plugin makes the VF schedulable, and Topology Manager must NUMA-align it with the GPU.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

**05** established what a container needs in order to get GPUDirect RDMA: a NIC that is rail-aligned with its GPU, `NCCL_IB_HCA` pinned to that NIC's `mlx5_*` name, and a kernel shim so the NIC can register HBM. That lesson quietly assumed the NIC was simply *present* in the pod, ready to be named. On Kubernetes it is not. The default CNI hands every pod exactly one interface — `eth0`, on the cluster pod network — built for ordinary service-to-service TCP/IP, with no notion of a second device bound to physical fabric hardware, no `/dev/infiniband` character devices, and no way for the scheduler to know that two pods cannot both have the same Virtual Function.

This lesson closes that gap. It is the plumbing between "05's ideal pod" and "a pod the Kubernetes networking model can actually produce": Multus, the SR-IOV Network Operator, the SR-IOV device plugin, the RDMA shared device plugin, the NVIDIA Network Operator, and Topology Manager. It also shows exactly where 05's rail alignment silently fails once a *scheduler*, rather than a human, is choosing which GPU and which NIC a pod gets — and it names the one setting, RDMA namespace mode, that decides whether a pod can address a NIC it was never allocated.

## Why this matters

This is the Kubernetes-platform differentiator in this module. Anyone can run `nccl-tests` on bare metal; the job you are targeting is *operating* the fabric on Kubernetes across dozens of clusters where pods, not hosts, are the unit of work. A pod scheduled onto a node with a perfectly railed ConnectX-7 will still stage every byte through host memory — 05's cause-2 signature — if the RDMA device never reached the pod's namespace, if the NAD's resource annotation does not exactly match the advertised resource, or if the scheduler handed it a GPU on NUMA 0 and a VF on NUMA 1. None of those show up in `kubectl get pod`. The pod is `Running`; throughput is quietly halved.

The inverse failure is just as expensive and much more confusing: with the RDMA subsystem in its default **shared** namespace mode, a container can see and use *every* RDMA device on the host, including ones allocated to another tenant's pod. Your `NCCL_IB_HCA=mlx5_3` works fine — on someone else's NIC, over a rail that is not aligned with your GPU, while the VF you were actually allocated sits idle. Being able to read a `NicClusterPolicy`, a `SriovNetworkNodePolicy`, and a `NetworkAttachmentDefinition` and say *exactly* where the chain can break is the skill that lets you attribute a "slow training run" to a scheduling bug in ten minutes rather than blaming the model for two weeks. The module checkpoint asks for it directly: name what Multus, the SR-IOV device plugin, and the Network Operator each contribute, and why the default CNI cannot do RDMA.

## What's new here (calibration)

- In the platform module's Topology Manager lesson you learned that CPU and GPU are NUMA-aligned via device-plugin `TopologyInfo` hints. That admission logic is **not re-taught**; what is new is the third resource — an RDMA NIC Virtual Function — that has to produce a hint in the first place, and the specific ways it fails to.
- In **02b / 09.5** you learned GDR needs GPU and NIC under a common PCIe bridge. That rule is **not re-derived**; what is new is the Kubernetes-native proxy for it (NUMA node), why the proxy is coarser than the real constraint, and what makes a VF NUMA-visible to the kubelet at all.
- **New: the complete allocation path, traced through source.** Where the PCI address of your VF actually comes from, and how it travels from the kubelet's device manager, through the pod-resources gRPC API, into the CNI config that the `sriov` plugin consumes. This is the chain that "the NAD annotation must match the resource name" is a consequence of.
- **New: four components that are constantly conflated** — Multus (interface plumbing), the SR-IOV Network **Operator** (creates VFs, generates NADs), the SR-IOV **device plugin** (advertises existing VFs), and the NVIDIA Network Operator (bundles and lifecycles the lot). Plus a fifth most write-ups omit entirely: the **network resources injector** webhook.
- **New: RDMA namespace mode** (`shared` vs `exclusive`, i.e. `ib_core netns_mode`) — the isolation setting that decides whether a pod can address RDMA devices it was not allocated. It is the reason "we scheduled it correctly" and "it used the right NIC" are two different claims.
- **New: what Topology Manager rejection actually does to a pod** — the exact reason string, the exact message, and the fact that the pod is not rescheduled.
- **New: DRA as the successor path**, with the real API objects and attribute names from the upstream driver, not a vendor blog post.

## Core concepts

### 1. What a pod actually needs, and what the default CNI gives it

An RDMA-capable container needs **four** distinct things. Enumerate them, because each is supplied by a different component and each fails differently.

| # | What | Why | Who supplies it |
|---|---|---|---|
| 1 | A **network interface** bound to the fabric NIC, with an address on the fabric subnet | RoCE addressing uses the interface's IP to derive the GID; InfiniBand needs an IPoIB child or an IB VF for address resolution and for NCCL's out-of-band setup | a delegate CNI (`sriov`, `ib-sriov`, `macvlan`, `ipoib`) invoked by Multus |
| 2 | The **RDMA character devices** — `/dev/infiniband/uverbs<N>`, and usually `/dev/infiniband/rdma_cm` | `libibverbs` opens `uverbsN` to create protection domains, register memory and build queue pairs; without the device node `ibv_devinfo` sees nothing | the device plugin, via `DeviceSpec` entries |
| 3 | A **scheduling claim** on the VF, so two pods cannot both get it | a VF is a real, countable, exhaustible piece of hardware | the device plugin, as a Kubernetes extended resource |
| 4 | Permission to **lock memory** — `IPC_LOCK` and an adequate `memlock` limit | `ibv_reg_mr` pins pages; unprivileged containers hit `RLIMIT_MEMLOCK` and registration fails | the pod spec `securityContext` |

The default CNI — Calico, Cilium, whatever your cluster runs — supplies exactly one of these, and only in the weakest sense: it creates `eth0` on the cluster pod network, a veth pair with an overlay address, engineered for service-to-service TCP/IP. It has no concept of "also give me an interface bound to `enp65s0f0np0`'s fifth Virtual Function," no mechanism to place a character device in the container, and no way to tell the scheduler that the node has eight of something.

That is the honest answer to the checkpoint's "why isn't the default CNI enough": **it is a single-interface, single-purpose model, and RDMA needs an interface, a device node, and an accounting unit, none of which it produces.**

### 2. Multus — a CNI that calls other CNIs

**Multus** is a *meta-CNI*. The kubelet is configured to call Multus as *the* CNI plugin; Multus then calls other plugins on its behalf. On pod creation it:

1. Invokes the **default delegate** (the cluster's real CNI) to create `eth0`, so ordinary pod networking is unchanged.
2. Reads the pod's `k8s.v1.cni.cncf.io/networks` annotation.
3. For each entry, fetches the named **`NetworkAttachmentDefinition`** (NAD) custom resource, extracts the CNI config from `spec.config`, and invokes that plugin to add an **additional** interface — `net1`, `net2`, …
4. Writes the result back to the pod as a `k8s.v1.cni.cncf.io/network-status` annotation.

The annotation has both a terse and a structured form, and both are worth knowing (all verified against the Multus `docs/how-to-use.md`):

```yaml
# 1 — simple, comma-separated list of NAD names in the pod's own namespace
k8s.v1.cni.cncf.io/networks: rdma-net

# 2 — the same NAD twice: two attachments, two VFs, net1 and net2
k8s.v1.cni.cncf.io/networks: rdma-net,rdma-net

# 3 — a NAD from another namespace
k8s.v1.cni.cncf.io/networks: fabric/rdma-net

# 4 — pin the interface name (otherwise net1, net2, ... in order)
k8s.v1.cni.cncf.io/networks: rdma-net@ib0

# 5 — JSON form, which is what you need for anything beyond a name
k8s.v1.cni.cncf.io/networks: '[
  { "name": "rdma-net", "namespace": "fabric", "interface": "ib0" }
]'
```

**Multus's contribution is orchestration of multiple interfaces, nothing more.** It does not create VFs, does not know what RDMA is, and does not implement any datapath. The RDMA-capable interface is built by whatever plugin the NAD names — `sriov` for an Ethernet/RoCE VF, `ib-sriov` for an InfiniBand VF, `ipoib` for an IPoIB child link, `macvlan` for the shared-device path. Getting this boundary right matters because it tells you which component to debug: no `net1` at all is a Multus/NAD problem; a `net1` that exists but carries no RDMA is a delegate-CNI or device-plugin problem.

Here is the whole stack, with ownership marked:

```
   ┌───────────────────────────── NODE ─────────────────────────────────┐
   │                                                                    │
   │  ┌──────────────── POD: nccl-worker ─────────────────┐             │
   │  │                                                    │             │
   │  │   eth0  ◀── default CNI (Calico/Cilium)            │             │
   │  │          10.244.3.7/24 · cluster pod network       │             │
   │  │          service traffic, NCCL bootstrap only      │             │
   │  │                                                    │             │
   │  │   net1  ◀── sriov / ib-sriov CNI, invoked by       │             │
   │  │          Multus, moved into this netns             │             │
   │  │          192.168.100.7/24 · fabric subnet          │             │
   │  │            └─ backed by PCI 0000:41:00.5 (a VF)    │             │
   │  │                                                    │             │
   │  │   /dev/infiniband/uverbs5   ◀── DeviceSpec from    │             │
   │  │   /dev/infiniband/rdma_cm       the device plugin  │             │
   │  │                                                    │             │
   │  │   capabilities: IPC_LOCK    ◀── pod spec           │             │
   │  │   nvidia.com/gpu: 1         ◀── GPU device plugin  │             │
   │  │   nvidia.com/hostdev: 1     ◀── SR-IOV dev plugin  │             │
   │  └────────────────────────────────────────────────────┘             │
   │        ▲                ▲               ▲                           │
   │        │                │               │                           │
   │   ┌────┴─────┐   ┌──────┴──────┐  ┌─────┴────────┐                  │
   │   │  MULTUS  │   │  SR-IOV     │  │   kubelet    │                  │
   │   │ meta-CNI │   │  DEVICE     │  │  device mgr  │                  │
   │   │ reads the│   │  PLUGIN     │  │  + Topology  │                  │
   │   │ annot.,  │   │ finds VFs,  │  │   Manager    │                  │
   │   │ delegates│   │ advertises  │  │  co-allocates│                  │
   │   └──────────┘   │ them + NUMA │  │  on one NUMA │                  │
   │                  └─────────────┘  └──────────────┘                  │
   │                         ▲                                           │
   │                         │ VFs must already exist                    │
   │                  ┌──────┴───────────────────┐                       │
   │                  │  SR-IOV NETWORK OPERATOR │                       │
   │                  │  writes sriov_numvfs,    │                       │
   │                  │  drains the node to do it│                       │
   │                  └──────────────────────────┘                       │
   │                         ▲                                           │
   │   ┌─────────────────────┴──────────────────────────────────┐        │
   │   │  NVIDIA NETWORK OPERATOR — deploys and lifecycles      │        │
   │   │  DOCA-OFED, Multus, both device plugins, IPoIB, IPAM   │        │
   │   │  from ONE cluster-scoped CR: NicClusterPolicy          │        │
   │   └────────────────────────────────────────────────────────┘        │
   │                                                                     │
   │   NOT in this picture, and a common gap:  nvidia_peermem is loaded  │
   │   by the GPU OPERATOR, not the Network Operator (05 §3).            │
   └─────────────────────────────────────────────────────────────────────┘
```

### 3. SR-IOV: what a Virtual Function actually is

Single Root I/O Virtualization is a PCIe capability. A capable NIC exposes one **Physical Function** (PF) — the real device, `0000:41:00.0` — and can be told to instantiate up to `sriov_totalvfs` **Virtual Functions**, each a lightweight PCIe function with its own requester ID, its own MSI-X vectors, its own queues, and — critically for us — **its own RDMA context**. To the RDMA subsystem a VF appears as its own `mlx5_N` device with its own `uverbsN` node.

The mechanism is a single sysfs write:

```console
$ cat /sys/class/net/enp65s0f0np0/device/sriov_totalvfs
127
$ cat /sys/class/net/enp65s0f0np0/device/sriov_numvfs
0
$ echo 8 | sudo tee /sys/class/net/enp65s0f0np0/device/sriov_numvfs
8
$ ls /sys/class/net/enp65s0f0np0/device/ | grep virtfn
virtfn0  virtfn1  virtfn2  virtfn3  virtfn4  virtfn5  virtfn6  virtfn7
```

Two properties follow from this being a PCIe reconfiguration rather than a software abstraction:

- **VF count is provisioned, not elastic.** You cannot grow from 8 to 9 to satisfy a scheduling burst. Changing `sriov_numvfs` requires writing 0 first, which tears down every existing VF — so it is disruptive to every pod on the node. The SR-IOV Network Operator therefore **drains the node** before applying a policy change (and `SriovOperatorConfig.spec.disableDrain` exists only for debugging). Capacity-plan VF count like GPU count: a fixed, provisioned quantity.
- **A VF inherits the PF's PCIe position.** `0000:41:00.5` sits under the same bridge as `0000:41:00.0`, so the §5 topology rule from lesson 05 — a common upstream PCIe bridge with the GPU — is a property of *which PF* you carve, not of which VF you get from it. Choose the PF per rail; the VF index is then irrelevant to GDR.

### 4. Two different components, two different jobs

The single most common confusion in this area, so state it flatly:

> **The SR-IOV Network Operator creates VFs. The SR-IOV network device plugin only discovers and advertises VFs that already exist.** They are separate upstream projects. A cluster missing the first has nothing for the second to advertise; a cluster missing the second has VFs no pod can ever be scheduled onto.

**`SriovNetworkNodePolicy`** is the operator's input. Every field below is from the CRD's Go types (`api/v1/sriovnetworknodepolicy_types.go`, `k8snetworkplumbingwg/sriov-network-operator`):

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: rail3-vfs
  namespace: sriov-network-operator      # must be the operator's namespace
spec:
  # (1) Which nodes. NFD or the operator's own labelling supplies this label.
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"

  # (2) Which PF on those nodes. Selectors AND together; empty fields are ignored.
  #     15b3 is the NVIDIA/Mellanox PCI vendor ID.
  nicSelector:
    vendor: "15b3"
    deviceID: "1021"                     # ConnectX-7. VERIFY with `lspci -nn`:
                                         # the CRD's doc-comment lists older IDs
                                         # (1013/1015/1017/101b/…) but there is no
                                         # enum validation, so any hex ID is accepted.
    pfNames: ["enp65s0f0np0#0-7"]        # optional VF-index range on this PF:
                                         # "#0-7" restricts THIS policy to VFs 0..7,
                                         # letting a second policy own 8..15.
    # rootDevices: ["0000:41:00.0"]      # alternative: select the PF by PCI address
    # netFilter: "openstack/NetworkID:…" # cloud-provider network filter

  # (3) How many VFs to carve. THIS is the line that creates hardware.
  #     Applying it drains the node.
  numVfs: 8

  # (4) The resource name the device plugin will advertise these under.
  #     The FULL name is $RESOURCE_PREFIX/<resourceName>. The upstream chart
  #     defaults RESOURCE_PREFIX to "openshift.io"; NVIDIA's network-operator
  #     chart sets it to "nvidia.com". So this becomes nvidia.com/rail3 there
  #     and openshift.io/rail3 upstream. Do not assume the prefix.
  resourceName: rail3

  # (5) Bind VFs to the kernel netdev driver (mlx5_core) so they get a netdev
  #     and an RDMA device. "vfio-pci" instead hands the raw VF to a userspace
  #     driver or a VM — no netdev, no mlx5 RDMA device, no NCCL.
  deviceType: netdevice

  # (6) Configure the VFs for RDMA. Without this the device plugin will not
  #     attach the /dev/infiniband character devices.
  isRdma: true

  # (7) Link type. "ib" for InfiniBand, "eth" for RoCE. This decides which
  #     delegate CNI you need (§6) — they are not interchangeable.
  linkType: eth

  # (8) MTU on the VFs. For RoCE you generally want the fabric's jumbo MTU here.
  mtu: 9000

  # (9) Higher priority wins when two policies select the same PF. 0..99.
  priority: 10

  # (10) Suppress NUMA advertisement for these devices. Leave FALSE — setting
  #      it true is exactly how you blind Topology Manager (§9).
  excludeTopology: false

  # (11) Do not create the VFs, only hand pre-existing ones to the plugin.
  #      For clouds where the VFs are created outside Kubernetes.
  externallyManaged: false
```

The operator's config daemon reconciles this onto matching nodes and records what it found in a `SriovNetworkNodeState` per node — which is the object to read when you want to know what the operator *believes* about a node's NICs.

### 5. The device plugin — turning VFs into a schedulable resource

The **SR-IOV network device plugin** (`k8snetworkplumbingwg/sriov-network-device-plugin`) is an ordinary Kubernetes device plugin. It scans PCI devices, filters them by selectors, and registers each matching VF with the kubelet as one unit of an extended resource.

Its config is JSON, usually delivered as a ConfigMap (or inline in `NicClusterPolicy.spec.sriovDevicePlugin.config`):

```json
{
  "resourceList": [
    {
      "resourceName": "rail3",
      "resourcePrefix": "nvidia.com",
      "deviceType": "netDevice",
      "selectors": {
        "vendors":   ["15b3"],
        "devices":   ["101e"],
        "drivers":   ["mlx5_core"],
        "pfNames":   ["enp65s0f0np0#0-7"],
        "linkTypes": ["ether"],
        "isRdma":    true
      },
      "excludeTopology": false
    }
  ]
}
```

Facts worth having exactly right, all from the plugin's README and source:

- **The default `resourcePrefix` is `intel.com`.** Not `nvidia.com`. If nothing sets it, resources appear as `intel.com/rail3` on an NVIDIA NIC, which looks like a bug and is not.
- `deviceType` is one of `netDevice` (default), `accelerator`, `auxNetDevice`.
- Selectors are ANDed with each other and ORed within a list. `pfNames` and `rootDevices` accept the range syntax `netpf0#0,2-7,9`.
- **The plugin injects environment variables into the container**: `PCIDEVICE_<RESOURCE_NAME>` (uppercased, special characters replaced with `_`) holds the comma-separated PCI addresses allocated, and `PCIDEVICE_<RESOURCE_NAME>_INFO` holds a JSON blob. For `nvidia.com/rail3` that is `PCIDEVICE_NVIDIA_COM_RAIL3=0000:41:00.5`. **This is the cheapest in-container proof of what you actually got.**
- With `isRdma: true` the RDMA info provider adds `DeviceSpec` entries for `/dev/infiniband/uverbsN`, `rdma_cm`, `umad`, `issm` as they exist, and exports their container paths as environment variables (`uverbs`, `rdma_cm`, `umad`, `issm`, `rdma_dev`).
- **NUMA comes from sysfs and can be absent.** `pkg/devices/host.go` sets `nodeNum = -1` and only reads it when `excludeTopology` is false; `utils.GetDevNode()` reads `/sys/bus/pci/devices/<addr>/numa_node` and **returns -1 on any read failure or unparseable value**. `pkg/devices/api.go` then attaches a `TopologyInfo` **only if `nodeNum >= 0`**. So a BIOS that reports `numa_node = -1` produces a device with *no topology hint at all* — and a device with no hint cannot be misaligned, which means Topology Manager will happily admit it next to a GPU on any NUMA node. Check it directly:

```console
$ cat /sys/bus/pci/devices/0000:41:00.5/numa_node
0                  # good — a real node index
$ cat /sys/bus/pci/devices/0000:c1:00.3/numa_node
-1                 # BAD — no hint will be published for this VF
```

### 6. From CRD to NAD: `SriovNetwork` and `SriovIBNetwork`

You can hand-write a `NetworkAttachmentDefinition`, but the operator will generate one for you from a companion CR — and, importantly, **there are two of them, one per link type**. Getting this wrong is a real and common mistake: an InfiniBand VF attached with the `sriov` CNI does not work.

`SriovNetwork` renders a NAD with `"type": "sriov"`. `SriovIBNetwork` renders one with `"type": "ib-sriov"` (`api/v1/helper.go`: `data.Data["CniType"] = "ib-sriov"`). Both use the same template, so both produce `cniVersion 1.0.0` and both set the resource annotation to `$RESOURCE_PREFIX/<resourceName>`.

The InfiniBand form, with the fields that only exist there:

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovIBNetwork
metadata:
  name: rdma-net
  namespace: sriov-network-operator
spec:
  networkNamespace: training          # where the generated NAD is created
  resourceName: rail3                 # must match the NodePolicy's resourceName
  linkState: enable                   # auto | enable | disable
  capabilities: '{"infinibandGUID": true}'
  # ^ lets the CNI set a per-pod InfiniBand GUID. Pair it with ib-kubernetes,
  #   which watches pods, reads the network annotation, allocates a GUID from
  #   the NicClusterPolicy's pKeyGUIDPool range, and registers it with UFM so
  #   the subnet manager admits the pod to the right partition (PKey).
  ipam: |
    { "type": "whereabouts", "range": "192.168.100.0/24" }
```

And the NAD that comes out the other side — this is what Multus actually consumes, and what you should be reading when you debug:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rdma-net
  namespace: training
  annotations:
    # (A) THE JOIN. This string must EXACTLY equal the resource the device
    #     plugin advertises. A mismatch does not error — see §8.
    k8s.v1.cni.cncf.io/resourceName: nvidia.com/rail3
    sriovnetwork.openshift.io/owner-ref: SriovIBNetwork/rdma-net
spec:
  config: '{
    "cniVersion": "1.0.0",
    "name": "rdma-net",
    "type": "ib-sriov",
    "link_state": "enable",
    "capabilities": {"infinibandGUID": true},
    "ipam": {"type":"whereabouts","range":"192.168.100.0/24"}
  }'
```

Notice what is *not* in `spec.config`: any mention of which VF. The CNI config names a plugin and a network, never a device. The device arrives at runtime, which is §7.

### 7. The allocation path — where your VF's PCI address comes from

This is the sequence that makes the whole thing work, and every step is verifiable in source. Read it once carefully; the failure modes in §8 are all "this chain broke at step *k*."

```
   TIME ──▶

 ① SCHEDULER                                     kube-scheduler
    Sees:  resources.limits:
             nvidia.com/gpu: 1
             nvidia.com/rail3: 1
    Picks a node with capacity for BOTH.
    ⚠ The scheduler is NOT topology-aware. It knows the node has 8 GPUs
      and 8 VFs free; it does NOT know whether any GPU/VF pair shares a
      NUMA node.  That is checked later, on the node, and too late.
                        │
                        ▼
 ② KUBELET · DEVICE MANAGER                      /var/lib/kubelet
    Asks each device plugin for allocatable devices + TopologyInfo.
    Asks Topology Manager for a hint that satisfies every requested
    resource on ONE NUMA node (policy-dependent — §9).
    Calls Allocate() on the SR-IOV plugin, which returns:
        DeviceSpecs : /dev/infiniband/uverbs5, /dev/infiniband/rdma_cm
        Envs        : PCIDEVICE_NVIDIA_COM_RAIL3=0000:41:00.5
    Records the allocation in the pod-resources API.
                        │
                        ▼
 ③ CRI creates the sandbox and its network namespace
                        │
                        ▼
 ④ MULTUS  (CNI ADD)                             multus-cni
    a. delegate to the default CNI  ─────▶  eth0
    b. read k8s.v1.cni.cncf.io/networks ─▶  "rdma-net"
    c. GET the NAD; read its annotation  ─▶  "nvidia.com/rail3"
    d. dial the kubelet POD-RESOURCES gRPC API at
         /var/lib/kubelet/pod-resources/kubelet.sock
       → resourceMap["nvidia.com/rail3"].DeviceIDs = ["0000:41:00.5"]
       → take DeviceIDs[Index]; Index++   ← this is why the SAME NAD
                                             listed twice yields TWO
                                             different VFs
    e. inject into the delegate's CNI config:
         "deviceID":  "0000:41:00.5"
         "pciBusID":  "0000:41:00.5"
                        │
                        ▼
 ⑤ ib-sriov / sriov CNI                          sriov-cni
    Finds the netdev for 0000:41:00.5, moves it into the pod's netns,
    renames it net1, applies link_state / GUID / MTU, calls IPAM.
                        │
                        ▼
 ⑥ CONTAINER STARTS
    net1 up on 192.168.100.7/24
    /dev/infiniband/uverbs5 present
    PCIDEVICE_NVIDIA_COM_RAIL3=0000:41:00.5 in the environment
                        │
                        ▼
 ⑦ NCCL  (05)
    Enumerates RDMA devices, matches NCCL_IB_HCA, checks GPU↔NIC distance,
    engages GDR — or silently does not.
```

Step ④c–④d is the load-bearing join, and it is worth stating as a rule: **Multus learns which VF you got by asking the kubelet, keyed on the exact string in the NAD's `k8s.v1.cni.cncf.io/resourceName` annotation.** If that string does not match what the device plugin advertised, the map lookup misses, `deviceID` stays empty, and the `sriov` CNI is invoked with no device to move.

**The webhook that saves pod authors from all this.** The SR-IOV Network Operator can deploy a **network resources injector** (`SriovOperatorConfig.spec.enableInjector`) — a mutating admission webhook that reads the pod's `k8s.v1.cni.cncf.io/networks` annotation, resolves each NAD, reads its `resourceName` annotation, counts how many times each network is requested, and patches the matching resource request *and* limit into the pod spec. Request `rdma-net` twice and it injects `nvidia.com/rail3: "2"`. It can also propagate a NAD's `k8s.v1.cni.cncf.io/nodeSelector` annotation into the pod. **Know whether it is enabled in your cluster**, because it changes what a correct pod spec looks like: with the injector, the annotation alone is enough; without it, a pod that has the annotation but no resource request gets a Multus attachment with no allocated device.

### 8. The RDMA shared device plugin, and why "allocated" ≠ "isolated"

SR-IOV is not always available or wanted: VF count is fixed at PF-configuration time, some clouds do not expose SR-IOV, and for a single-tenant training node the isolation may be worth nothing. The **RDMA shared device plugin** (`Mellanox/k8s-rdma-shared-dev-plugin`) advertises a *shared* RDMA resource that many pods hold concurrently off one PF, typically paired with a `macvlan` or `ipoib` secondary interface.

```json
{
  "periodicUpdateInterval": 300,
  "configList": [
    {
      "resourceName": "rdma_shared_device_a",
      "resourcePrefix": "rdma",
      "rdmaHcaMax": 63,
      "selectors": {
        "vendors":   ["15b3"],
        "deviceIDs": ["101b"],
        "ifNames":   ["ib0"],
        "linkTypes": ["infiniband"]
      }
    }
  ]
}
```

- Default `resourcePrefix` here is **`rdma`**, so this advertises `rdma/rdma_shared_device_a`.
- `periodicUpdateInterval` defaults to **60 seconds** when unset; setting it to 0 disables rediscovery.
- **`rdmaHcaMax` is a count, not a quota.** It is simply how many units the plugin advertises. Nothing partitions bandwidth, queue pairs, or memory-region capacity between the pods that hold them. Setting it to 63 does not create 63 of anything.

Which brings us to the setting that most write-ups omit and that decides whether any of this is isolation at all.

**RDMA namespace mode.** The Linux RDMA subsystem has two modes, controlled by the `ib_core` module parameter `netns_mode`:

| Mode | `ib_core netns_mode` | Behaviour |
|---|---|---|
| **shared** (kernel default) | `1` | RDMA devices are visible in **every** network namespace. A container sees every `mlx5_*` on the host. |
| **exclusive** | `0` | RDMA devices are namespace-scoped; a device can be moved into a pod's netns and is then invisible elsewhere. |

The SR-IOV Network Operator exposes this as `SriovNetworkPoolConfig.spec.rdmaMode: shared|exclusive` and implements it by writing a modprobe config (`pkg/host/internal/network/network.go`):

```console
$ cat /host/etc/modprobe.d/sriov_network_operator_modules_config.conf
# This file is managed by sriov-network-operator do not edit.
options ib_core netns_mode=0

$ rdma system show
netns exclusive copy-on-fork on
```

Because it is a module parameter, changing it requires reloading `ib_core` — in practice, a node reboot.

**Why this matters enormously for lesson 05.** In the default **shared** mode, a container that was allocated VF `mlx5_5` can still open `mlx5_3` — a different rail, possibly another tenant's VF — and `NCCL_IB_HCA=mlx5_3` will simply work. Your pod is scheduled correctly, `PCIDEVICE_…` names the right VF, and NCCL is using a NIC that is not rail-aligned with your GPU, producing 05's cause-2 signature with no misconfiguration visible anywhere in Kubernetes. **In shared mode, "the scheduler aligned it" and "the process used it" are independent claims, and only the second one determines your throughput.** In exclusive mode with `rdma-cni` in the chain, the allocated device is moved into the pod's namespace and `ibv_devices` inside the container lists exactly one device — at which point a wrong `NCCL_IB_HCA` fails loudly instead of silently.

The one-line check, from inside the pod:

```console
$ ibv_devices
    device                 node GUID
    ------              ----------------
    mlx5_5              b83fd203004b1234        # exactly one → exclusive mode
$ echo $PCIDEVICE_NVIDIA_COM_RAIL3
0000:41:00.5                                    # and it matches what you were given
```

If `ibv_devices` lists eight devices, you are in shared mode and your `NCCL_IB_HCA` value is doing all the work that the scheduler was supposed to guarantee.

### 9. Topology Manager — the interlock, and what rejection really costs

Now the payoff. A GDR-optimal pod requests **both** `nvidia.com/gpu: 1` and an RDMA resource. Both device plugins report NUMA affinity via `TopologyInfo`. The kubelet's Topology Manager collects hints from every *hint provider* (CPU Manager, Memory Manager, Device Manager) and computes a merged affinity before admitting the pod.

Two knobs, and both are kubelet configuration, not pod configuration:

**Scope** — `topologyManagerScope`:

- `container` (default): alignment is computed per container, sequentially. Two containers in a pod can land on different NUMA nodes.
- `pod`: all containers in the pod are aligned to a common set of NUMA nodes. This is the right choice for a multi-container training pod where a sidecar shares the GPU.

**Policy** — `topologyManagerPolicy`:

| Policy | Behaviour on a misaligned pod |
|---|---|
| `none` | no alignment attempted |
| `best-effort` | records the preferred affinity, **admits the pod anyway** |
| `restricted` | **rejects** if the affinity is not preferred |
| `single-numa-node` | **rejects** unless a single-NUMA-node affinity is possible |

For a GDR-sensitive workload you want `single-numa-node`. And here is the part that is usually left out — **what rejection actually does**. From the kubelet source (`pkg/kubelet/cm/topologymanager/topology_manager.go`):

```go
ErrorTopologyAffinity = "TopologyAffinityError"

func (e TopologyAffinityError) Error() string {
    return "Resources cannot be allocated with Topology locality"
}
```

which surfaces as:

```console
$ kubectl describe pod nccl-worker-3
...
Status:   Failed
Reason:   TopologyAffinityError
Message:  Resources cannot be allocated with Topology locality
```

**The pod enters a terminal `Failed` state and the scheduler does not retry it.** There is no backoff, no reschedule, no attempt at a different node. This is the operationally important detail: a bare `Pod` that hits `TopologyAffinityError` is simply dead. You must run these workloads under something that re-creates pods — a Deployment, a StatefulSet, a Job, or a training-operator CR — or build an external control loop that watches for the reason string. Alerting on `reason=TopologyAffinityError` is the single highest-value alert in this whole stack, because it is the *loud* version of a failure whose alternative is silent.

Three more limits worth carrying:

- **The scheduler is not topology-aware.** It places the pod on a node that has free GPU and free VF counts, and only then does the kubelet discover they cannot be co-located. On a fragmented cluster this produces a pod that fails on node after node while the counts look fine.
- **Topology Manager aligns pods of all QoS classes** for device resources. But CPU *pinning* additionally requires the `static` CPU Manager policy and a Guaranteed pod with an **integer** CPU request equal to its limit. A pod requesting `cpu: "300m"` gets device alignment and a default (unconstrained) CPU hint — so "we set single-numa-node" does not by itself mean your threads are on the right socket.
- **Maximum 8 NUMA nodes**, because hint enumeration explodes combinatorially. The `max-allowable-numa-nodes` policy option (GA since Kubernetes 1.35, default 8) raises it. Relevant on very large multi-socket or sub-NUMA-clustering configurations.

### 10. The NVIDIA Network Operator — one CR for the whole stack

Wiring OFED, Multus, two device plugins, IPoIB, IPAM and NIC firmware by hand across dozens of clusters is how you get drift. The **NVIDIA Network Operator** deploys and lifecycle-manages the stack from one cluster-scoped CR, `NicClusterPolicy` (`mellanox.com/v1alpha1`). Verified against the current API types and the repo's own `cr-full` example:

```yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata: { name: nic-cluster-policy }
spec:
  # (1) The DOCA-OFED driver container: mlx5 kernel modules + user-space verbs,
  #     built against the running kernel. The image is "doca-driver" — the old
  #     "mofed" naming is gone. Version strings look like the example below.
  ofedDriver:
    image: doca-driver
    repository: nvcr.io/nvidia/mellanox
    version: doca3.5.0-26.07-0.7.0.0-0
    upgradePolicy:
      autoUpgrade: true
      maxParallelUpgrades: 1
      drain: { enable: true, force: true, deleteEmptyDir: true, timeoutSeconds: 300 }

  # (2) SR-IOV device plugin: ADVERTISES existing VFs. Does NOT create them —
  #     that is SriovNetworkNodePolicy, from the separate SR-IOV Network Operator.
  sriovDevicePlugin:
    image: sriov-network-device-plugin
    repository: nvcr.io/nvidia/mellanox
    version: network-operator-v26.7.0
    config: |
      { "resourceList": [ {
          "resourcePrefix": "nvidia.com",
          "resourceName": "rail3",
          "selectors": { "vendors": ["15b3"], "isRdma": true } } ] }

  # (3) The shared (non-SR-IOV) path, if you want it. Both plugins can coexist
  #     as long as their selectors do not overlap.
  rdmaSharedDevicePlugin:
    image: k8s-rdma-shared-dev-plugin
    repository: nvcr.io/nvidia/mellanox
    version: network-operator-v26.7.0
    config: |
      { "configList": [ {
          "resourceName": "rdma_shared_device_a",
          "rdmaHcaMax": 63,
          "selectors": { "vendors": ["15b3"], "deviceIDs": ["101b"] } } ] }

  # (4) Secondary-network machinery. NOTE the field set: multus, cniPlugins,
  #     ipoib. There is NO ipamPlugin field here — IPAM moved out (see (5)).
  secondaryNetwork:
    multus:     { image: multus-cni,  repository: nvcr.io/nvidia/mellanox, version: network-operator-v26.7.0 }
    cniPlugins: { image: plugins,     repository: nvcr.io/nvidia/mellanox, version: network-operator-v26.7.0 }
    ipoib:      { image: ipoib-cni,   repository: nvcr.io/nvidia/mellanox, version: network-operator-v26.7.0 }

  # (5) IPAM is now its own top-level component.
  nvIpam:
    image: nvidia-k8s-ipam
    repository: nvcr.io/nvidia/mellanox
    version: network-operator-v26.7.0
    enableWebhook: false

  # (6) InfiniBand PKey/GUID management: watches pods, allocates a GUID from the
  #     pool, registers it with UFM so the subnet manager admits the pod to the
  #     right partition. Needed for the SriovIBNetwork infinibandGUID capability.
  ibKubernetes:
    image: ib-kubernetes
    repository: nvcr.io/nvidia/mellanox
    version: network-operator-v26.7.0
    pKeyGUIDPoolRangeStart: "02:00:00:00:00:00:00:00"
    pKeyGUIDPoolRangeEnd:   "02:FF:FF:FF:FF:FF:FF:FF"
    ufmSecret: ufm-secret

  # (7) NIC firmware/non-volatile configuration as a Kubernetes CRD.
  nicConfigurationOperator:
    operator:            { image: nic-configuration-operator,        repository: nvcr.io/nvidia/mellanox, version: network-operator-v26.7.0 }
    configurationDaemon: { image: nic-configuration-operator-daemon, repository: nvcr.io/nvidia/mellanox, version: network-operator-v26.7.0 }
```

Read `.status` to know whether it worked: `NicClusterPolicyStatus` carries an aggregate `state` (`ignore`/`notReady`/`ready`/`error`) plus per-component `appliedStates`, so you can see *which* piece is `notReady` rather than guessing.

**What the Network Operator does not do**, and both gaps bite:

1. **It does not create VFs.** `SriovNetworkNodePolicy` does, and that CRD belongs to the separate SR-IOV Network Operator — which the NVIDIA chart can deploy as a subchart, with its own `resourcePrefix` value (set to `nvidia.com` there, `openshift.io` upstream).
2. **It does not load `nvidia_peermem`.** The Network Operator's own README states that from driver v465 the module ships inside the GPU driver and **the GPU Operator manages loading it**. So a cluster with a perfectly `ready` `NicClusterPolicy` can still have no GPUDirect RDMA, because the missing piece belongs to the other operator. This is the exact seam between this lesson and 05 §3.

### 11. Where alignment breaks, in order of how often you will see it

| # | Break | Symptom | How to confirm | Fix |
|---|---|---|---|---|
| 1 | **NAD resource annotation ≠ advertised resource name** (prefix mismatch is the usual cause: `openshift.io` vs `nvidia.com`) | `net1` exists but has no VF behind it, or CNI ADD fails | compare `kubectl get net-attach-def -o yaml` annotation against `kubectl get node -o jsonpath='{.status.allocatable}'` | make the strings identical, including prefix |
| 2 | **Pod has the annotation but no resource request**, and the injector webhook is not enabled | pod Running, no device allocated, `PCIDEVICE_*` env absent | `env \| grep PCIDEVICE` inside the pod | add the explicit `resources.limits` entry, or enable `enableInjector` |
| 3 | **Shared RDMA namespace mode + wrong `NCCL_IB_HCA`** | pod correct in every K8s view; NCCL uses another rail; 05's cause-2 | `ibv_devices` shows many devices; compare with `$PCIDEVICE_*` | derive `NCCL_IB_HCA` from `PCIDEVICE_*` at runtime, or move to `rdmaMode: exclusive` |
| 4 | **`numa_node` is -1**, so no topology hint is published | `single-numa-node` admits a cross-NUMA pod without complaint | `cat /sys/bus/pci/devices/<vf>/numa_node` | BIOS/ACPI SRAT fix; until then, constrain placement with node labels |
| 5 | **`best-effort` policy** left on from a debugging session | Running pod, cross-NUMA GDR path, roughly half bandwidth, nothing alerts | kubelet config; absence of any `TopologyAffinityError` events on a fragmented node | `single-numa-node`, and run under a controller that retries |
| 6 | **VF pool not pinned per NUMA** — one policy carving VFs off a PF on the wrong socket for the GPUs the node schedules | chronic `TopologyAffinityError` on half the pods | map PF → socket with `lspci -nn` + `numa_node`; compare to GPU NUMA from `nvidia-smi topo -m` | one `SriovNetworkNodePolicy` per PF, with matching node labels |
| 7 | **Wrong delegate CNI for the link type** — `sriov` on an InfiniBand VF | interface fails to come up or has no RDMA | `linkType` in the NodePolicy vs `type` in the NAD | `SriovIBNetwork` → `ib-sriov` for IB; `SriovNetwork` → `sriov` for RoCE |
| 8 | **Missing `IPC_LOCK` / low `memlock`** | `ibv_reg_mr` fails; NCCL errors or degrades | `ulimit -l` inside the container | add the capability; set `memlock` unlimited in the runtime config |

### 12. Where this is going: DRA

The static model above — VFs provisioned at node-configuration time, advertised as opaque countable extended resources, aligned at admission by NUMA hints — has a structural weakness. **An extended resource is just a number.** `nvidia.com/rail3: 1` carries no information about which rail, which PCIe bridge, or which GPU it should sit beside. NUMA node is the only property the kubelet can align on, and NUMA node is a *coarser* constraint than "shares a PCIe switch with GPU 3" (lesson 05 §5). Two devices on the same NUMA node can still be on different root ports.

**Dynamic Resource Allocation** replaces the count with a structured, queryable device object. `DRANET` — donated to the Kubernetes organisation at KubeCon NA 2025 and now developed as `kubernetes-sigs/dranet` — is the network driver for this model. It talks to the kubelet over the DRA API and to the container runtime over **NRI** (rather than being a CNI plugin), and publishes each interface as a device in a `ResourceSlice` with attributes under the `dra.net` prefix. From `pkg/apis/attributes.go`, the published attributes include:

```
  dra.net/ifName        dra.net/pciAddress     dra.net/numaNode
  dra.net/mac           dra.net/pciVendor      dra.net/pciDevice
  dra.net/mtu           dra.net/pciSubsystem   dra.net/type
  dra.net/state         dra.net/alias          dra.net/encapsulation
  dra.net/rdma          dra.net/rdmaDevice     dra.net/virtual
  dra.net/sriov         dra.net/sriovVfs       dra.net/isSriovVf
```

A workload then selects devices with CEL expressions over those attributes, rather than asking for a count of an opaque resource:

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata: { name: rdma }
spec:
  selectors:
    - cel: { expression: 'device.driver == "dra.net"' }
    - cel: { expression: 'device.attributes["dra.net"].rdma' }
---
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata: { name: rdma-net-template }
spec:
  spec:
    devices:
      requests:
        - name: rdma-net-interface
          exactly:
            deviceClassName: rdma
            selectors:
              - cel: { expression: 'device.attributes["dra.net"].ifName == "gpu1rdma0"' }
```

and reference the claim from the pod:

```yaml
spec:
  resourceClaims:
    - name: rdma
      resourceClaimTemplateName: rdma-net-template
  containers:
    - name: trainer
      resources:
        limits: { nvidia.com/gpu: 1 }
        claims:
          - name: rdma
```

The payoff is that the *scheduler* — not the kubelet at admission time — can reason about the pairing, because the attributes are cluster-visible. DRANET's own research paper reports up to **59.6% higher bus bandwidth for `all_gather` and 58.1% for `all_reduce`** from topology-aware GPU-and-NIC scheduling versus the non-aware baseline. Take that as the project's own published figure for the size of the misalignment problem this lesson is about, not as a general benchmark.

**Multus supports DRA too**, from Kubernetes 1.34 (`docs/how-to-use.md`): a DRA driver must publish `k8s.cni.cncf.io/deviceID` and `k8s.cni.cncf.io/resourceName` attributes on each allocated network device, the NAD's `k8s.v1.cni.cncf.io/resourceName` annotation must equal the latter *exactly*, and Multus then queries the `ResourceClaim`/`ResourceSlice` APIs instead of the pod-resources API to find the device ID. Devices lacking either attribute are **silently skipped** — deliberately, so a pod holding both a GPU claim and a network claim works.

**Do not present DRA as having replaced the stack this lesson teaches.** The SR-IOV/Multus/Network-Operator chain is what is deployed in most production clusters, and it is what a checkpoint question is asking about. DRA is where the constraint model is going, and knowing why — extended resources cannot express topology, only count — is the part worth carrying.

## Perspectives

**Developer.** From the pod author's seat the happy path is one annotation and two resource requests, and everything above is invisible. What is worth internalising is the two failure signatures and the one runtime check. `TopologyAffinityError` in `kubectl describe` means the node could not co-locate your GPU and NIC, and — crucially — your bare pod is now dead and will not be retried. Silence plus half the expected throughput means either the device never arrived or NCCL is using a different one. The check that distinguishes them takes ten seconds: `env | grep PCIDEVICE` for what you were given, `ibv_devices` for what you can see. If those two disagree in count, you are in shared RDMA namespace mode and your env var is load-bearing.

**Operator.** The reason the Network Operator exists is drift. Hand-wiring OFED, Multus, two device plugins, IPAM and NUMA policy across dozens of clusters produces exactly the fleet you would predict: one cluster on an older DOCA build, one with a stale `numVfs`, one with `best-effort` left on from a 3 a.m. debugging session and never reverted. One reconciled `NicClusterPolicy` per cluster with a readable `.status` is the only version of this that scales. Two operational rules follow. First, `ofedDriver.version` must track the kernel in each node image — a node-image bump without a matching driver bump is a recurring, self-inflicted outage. Second, alert on `reason=TopologyAffinityError`, because it is the only loud failure in a stack whose other failures are all silent.

**Hardware / kernel.** Two constraints here are physical, not software policy, and both change capacity planning. VF count is a PCIe reconfiguration: going from 8 to 9 means writing 0 to `sriov_numvfs` first, which destroys every VF on that PF, which is why the operator drains the node. Treat VF count like GPU count — a provisioned quantity, not something a scheduler can conjure. And RDMA namespace mode is an `ib_core` module parameter, so flipping a cluster from shared to exclusive isolation is a reboot, not a `kubectl apply`. Decide it at build time.

**Economics / failure-mode.** Every mechanism in this lesson is insurance against one specific loss: a job that runs at half the bandwidth you paid for with no visible symptom. Price it directly. A 512-GPU reservation on a fabric bought for 400 Gb/s per GPU, running at the bounce-path bandwidth because a scheduler put the VF on the wrong socket, wastes GPU-hours in exact proportion to how comms-bound the job is — and there is no dashboard tile that says so. That is why `single-numa-node` failing loudly is the correct trade against `best-effort` succeeding quietly, and why the argument for `best-effort` ("it avoids scheduling failures") is an argument for converting a visible cost into an invisible one. Lesson 07 puts the dollar figure on the fabric this is protecting.

## Real-world use cases

- **Multus's `getKubernetesDelegate` and the pod-resources API (`pkg/k8sclient/k8sclient.go`).** *What it shows:* Multus does not receive the allocated device from the CNI arguments. It dials `/var/lib/kubelet/pod-resources/kubelet.sock`, builds a map from resource name to allocated device IDs, looks up the exact string in the NAD's `k8s.v1.cni.cncf.io/resourceName` annotation, takes `DeviceIDs[Index]` and increments `Index`, then injects `"deviceID"` and `"pciBusID"` into the delegate's config. *Why it matters:* it explains three otherwise arbitrary rules at once — why the annotation must match exactly (a map miss yields an empty device ID and no error), why listing the same NAD twice yields two different VFs (the `Index++`), and why the whole chain depends on the kubelet's device manager having actually allocated something.

- **`GetDevNode()` returning -1 (`sriov-network-device-plugin`, `pkg/utils/utils.go` and `pkg/devices/api.go`).** *What it shows:* the plugin reads `/sys/bus/pci/devices/<addr>/numa_node` and returns -1 on any failure or unparseable value; `NewAPIDeviceImpl` then attaches a `TopologyInfo` **only when the value is ≥ 0**. *Why it matters:* a device with no topology hint cannot be reported as misaligned, so `single-numa-node` admits it next to a GPU on any NUMA node. The strictest possible policy silently degrades to no policy at all, because of one sysfs file. It is a two-second check that almost nobody makes.

- **`SetRDMASubsystem()` writing a modprobe file (`sriov-network-operator`, `pkg/host/internal/network/network.go`).** *What it shows:* `rdmaMode: exclusive` is implemented as `options ib_core netns_mode=0` in `/etc/modprobe.d/sriov_network_operator_modules_config.conf` — a module parameter, applied at module load. *Why it matters:* it makes concrete that RDMA device isolation is a kernel-module property of the whole node, not a per-pod one; that changing it requires a reboot; and that the default (shared) means a container can open any HCA on the host regardless of what Kubernetes allocated it.

- **The `network-resources-injector` webhook (`k8snetworkplumbingwg/network-resources-injector`, deployed via `SriovOperatorConfig.enableInjector`).** *What it shows:* a mutating admission controller that resolves the pod's network annotation to NADs, reads each NAD's `resourceName`, counts repeats, and patches the corresponding requests and limits into the pod — so `foo-network` listed twice becomes `example.com/foo: "2"`. *Why it matters:* whether it is deployed changes what a *correct* pod spec looks like in your cluster. A spec that works in a cluster with the injector produces a Running pod with no allocated device in one without it — and that pod's symptom is silent, not an error.

## Worked example

**Goal.** Produce one 8-GPU training pod per node on an HGX H100 fleet where GPU*k* is rail-aligned with `mlx5_k`, and prove end to end that the pod got the right VF on the right NUMA node.

**Given.** From lesson 05's `topo -m`: GPU0/1 on NUMA 0, GPU2/3 on NUMA 1; GPU*k*'s paired NIC is `mlx5_k` at `PXB`. Say `mlx5_3`'s PF is `enp193s0f0np0` at `0000:c1:00.0`, and the whole `mlx5_2`/`mlx5_3` pair is on NUMA 1.

**Step 1 — one policy per rail, not one for the node.** This is the mistake that generates failure #6 in §11. Carve rail 3's VFs from rail 3's PF:

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata: { name: rail3, namespace: sriov-network-operator }
spec:
  nodeSelector: { node.example.com/sku: "hgx-h100-8x" }
  nicSelector:  { vendor: "15b3", pfNames: ["enp193s0f0np0"] }
  numVfs: 8
  resourceName: rail3          # → nvidia.com/rail3 with RESOURCE_PREFIX=nvidia.com
  deviceType: netdevice
  isRdma: true
  linkType: eth                # RoCE fabric; use "ib" + SriovIBNetwork for IB
  mtu: 9000
  excludeTopology: false       # leave false or you blind Topology Manager
  priority: 10
```

Repeat per rail. Eight policies, eight resource names, eight NADs. It looks verbose; the alternative is one pool of VFs the scheduler cannot align.

**Step 2 — verify the operator actually did it, before anything else.**

```console
$ kubectl -n sriov-network-operator get sriovnetworknodestate node01 \
    -o jsonpath='{range .status.interfaces[?(@.name=="enp193s0f0np0")]}{.numVfs}{"\n"}{end}'
8

$ kubectl get node node01 -o jsonpath='{.status.allocatable}' | jq 'with_entries(select(.key|test("nvidia")))'
{
  "nvidia.com/gpu": "8",
  "nvidia.com/rail0": "8",
  ...
  "nvidia.com/rail3": "8"
}
```

*(Representative output.)* If `allocatable` is missing `nvidia.com/rail3`, stop here: either the VFs do not exist (operator problem) or the device plugin's selectors do not match them (plugin config problem). Nothing downstream can work.

**Step 3 — the NAD, with the join made explicit.**

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rail3-net
  namespace: training
  annotations:
    k8s.v1.cni.cncf.io/resourceName: nvidia.com/rail3   # ← must equal Step 2 exactly
spec:
  config: '{
    "cniVersion": "1.0.0",
    "name": "rail3-net",
    "type": "sriov",
    "ipam": { "type": "whereabouts", "range": "192.168.103.0/24" }
  }'
```

**Step 4 — the pod.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nccl-worker-r3
  namespace: training
  annotations:
    k8s.v1.cni.cncf.io/networks: rail3-net          # Multus attaches this as net1
spec:
  containers:
    - name: trainer
      image: nvcr.io/nvidia/pytorch:25.10-py3
      resources:
        limits:                                     # Guaranteed QoS: integer CPU,
          cpu: "12"                                 # limits == requests, so the
          memory: 128Gi                             # static CPU Manager policy
          nvidia.com/gpu: 1                         # also pins threads.
          nvidia.com/rail3: 1
        requests:
          cpu: "12"
          memory: 128Gi
          nvidia.com/gpu: 1
          nvidia.com/rail3: 1
      securityContext:
        capabilities:
          add: ["IPC_LOCK"]                         # ibv_reg_mr pins pages
      env:
        # Do NOT hardcode mlx5_3 here. In shared RDMA namespace mode a hardcoded
        # value can silently address a NIC you were not allocated (§8). Derive it.
        - { name: NCCL_NET_GDR_LEVEL,           value: "PXB" }
        - { name: NCCL_IB_PCI_RELAXED_ORDERING, value: "1" }
        - { name: NCCL_SOCKET_IFNAME,           value: "eth0" }
        - { name: NCCL_DEBUG,                   value: "INFO" }
        - { name: NCCL_DEBUG_SUBSYS,            value: "INIT,NET,GRAPH" }
      command: ["/bin/bash", "-lc"]
      args:
        - |
          set -euo pipefail
          # PCI address of the VF the kubelet actually allocated:
          PCI="${PCIDEVICE_NVIDIA_COM_RAIL3:?device plugin injected nothing}"
          # Map PCI address -> mlx5 device name via sysfs. This is the line that
          # makes the container's NCCL_IB_HCA agree with the scheduler's decision.
          HCA="$(basename /sys/bus/pci/devices/${PCI}/infiniband/*)"
          export NCCL_IB_HCA="=${HCA}:1"
          echo "allocated VF ${PCI} -> ${HCA}; NUMA $(cat /sys/bus/pci/devices/${PCI}/numa_node)"
          exec torchrun "$@"
```

That five-line preamble is the whole point of the lesson expressed as code: it closes the gap between *what Kubernetes allocated* and *what NCCL uses*, which shared RDMA namespace mode otherwise leaves open.

**Step 5 — verify from inside the pod.**

```console
$ kubectl -n training exec nccl-worker-r3 -- bash -lc '
    echo "PCI:  $PCIDEVICE_NVIDIA_COM_RAIL3"
    echo "NUMA: $(cat /sys/bus/pci/devices/$PCIDEVICE_NVIDIA_COM_RAIL3/numa_node)"
    ip -br addr show net1
    ls /dev/infiniband/
    ibv_devices
    ulimit -l'
PCI:  0000:c1:00.5
NUMA: 1
net1   UP   192.168.103.14/24
rdma_cm  uverbs5
    device            node GUID
    ------         ----------------
    mlx5_5         b83fd203004b1234
unlimited
```

Read it line by line. `PCI` proves the device plugin allocated something and `NUMA: 1` matches GPU3's NUMA node — the alignment worked. `net1` exists with a fabric address, so Multus and the `sriov` CNI both ran. `uverbs5` is present, so the RDMA character device came through. **`ibv_devices` listing exactly one device is the tell that this node is in exclusive RDMA namespace mode** — if it listed eight, the `NCCL_IB_HCA` derivation in Step 4 is the only thing keeping you on the right rail. `ulimit -l unlimited` means `ibv_reg_mr` will not fail on the memlock limit.

**Step 6 — check the Kubernetes-side view, then hand off to 05.**

```console
$ kubectl -n training get pod nccl-worker-r3 \
    -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}' | jq -r '.[].name'
kube-system/calico
training/rail3-net

$ kubectl -n training logs nccl-worker-r3 | grep -c GDRDMA
16
```

Two attachments in `network-status`, and a non-zero `GDRDMA` count. **Only now is 05's question answered**: the pod has a rail-aligned NIC, NCCL is pointed at it, and GPUDirect engaged. A zero on that last line with everything above green sends you back to 05 §8's four causes — most likely ACS, since Kubernetes has just been proven correct.

## Practice

Feeds the deliverable's **annotated multi-NIC manifest set**.

1. Take the four objects above — `SriovNetworkNodePolicy`, `SriovIBNetwork` (or `SriovNetwork`), the generated `NetworkAttachmentDefinition`, and the pod — or a real set from your cluster or the upstream repos' `example/` directories. Annotate **every non-trivial line** with one clause naming what it does and **which component consumes it**: SR-IOV Network Operator, SR-IOV device plugin, Multus, delegate CNI, IPAM, kubelet/Topology Manager, or NCCL.
2. Draw the allocation path from §7 as a numbered list for *your* manifests, naming at each step the concrete value that flows: the resource name, the PCI address, the device ID injected into the CNI config, the interface name, and the `mlx5_*` device NCCL ends up using. State explicitly which step reads the kubelet pod-resources API and what it keys the lookup on.
3. Identify **four** distinct places the chain can break in your manifest set. For each, give the symptom as it would appear (`TopologyAffinityError` and its exact message; a Running pod at half bandwidth; an empty `PCIDEVICE_*`; an `ibv_reg_mr` failure) and the fix. At least one must be a *silent* failure.
4. Name, precisely: the CRD that **creates** a VF and the repo it comes from; the CRD that merely **advertises** one; and the component that loads `nvidia_peermem`. This is the distinction most people blur and the acceptance check probes it directly.
5. State your cluster's **RDMA namespace mode** (`rdma system show`, or the `SriovNetworkPoolConfig`), and write two sentences on what that means for how `NCCL_IB_HCA` must be set. If you are in shared mode, include the runtime derivation from `PCIDEVICE_*` shown in Step 4 above.
6. Write the **Topology Manager configuration** you would run for this workload — policy and scope — and one sentence on what happens to a pod that fails admission under it, naming the reason string and the fact that it is not rescheduled. Then name the controller type you would run these pods under, and why.

**Acceptance:** an annotated `SriovNetworkNodePolicy` + `SriovNetwork`/`SriovIBNetwork` + NAD + pod-spec set, with the allocation path traced end to end naming the concrete value at each hop, four failure modes with symptom and fix (at least one silent), the create-vs-advertise-vs-load-peermem distinction stated exactly, and the RDMA-namespace-mode consequence for `NCCL_IB_HCA`. Done when a reader who has never seen these CRDs can point to the exact line that creates the VF, the exact line that puts the interface in the pod, the exact line that makes the VF schedulable, the exact annotation that joins the NAD to the resource, and the exact policy that forces NUMA alignment.

## Common pitfalls

- **Assuming the resource prefix.** *Symptom:* the NAD annotation says `nvidia.com/rail3`, the node advertises `openshift.io/rail3`, and pods come up with a `net1` that has no device behind it. *Mechanism:* the prefix comes from the `RESOURCE_PREFIX` environment variable — `openshift.io` in the upstream `sriov-network-operator` chart, `nvidia.com` in NVIDIA's network-operator chart, `intel.com` if you run the device plugin standalone with no override. Multus's lookup into the pod-resources map is an exact string match; a prefix mismatch is a map miss, not an error. *Fix:* read `kubectl get node -o jsonpath='{.status.allocatable}'` and copy the string verbatim into the NAD annotation.

- **Believing `single-numa-node` is protecting you when `numa_node` is -1.** *Symptom:* a strict policy configured, no `TopologyAffinityError` ever seen, and cross-NUMA pods running anyway. *Mechanism:* the device plugin publishes `TopologyInfo` only when the sysfs `numa_node` value parses to ≥ 0; otherwise the device carries no hint, and a device with no hint is never in conflict with a GPU's hint. *Fix:* `cat /sys/bus/pci/devices/<vf>/numa_node` on every VF as part of node validation, and treat a -1 as a node-level defect.

- **Choosing `best-effort` "to avoid scheduling failures."** *Symptom:* fewer failed pods and a fleet-wide throughput deficit nobody can attribute. *Mechanism:* `best-effort` records the preferred affinity and admits the pod regardless, converting a loud terminal failure into a silently misaligned Running pod. Nothing alerts, and the first signal is a researcher asking why their job is slow. *Fix:* `single-numa-node`, run the workload under a Deployment/StatefulSet/Job so failed pods are recreated, and alert on `reason=TopologyAffinityError`.

- **Expecting Kubernetes to retry a `TopologyAffinityError` pod.** *Symptom:* a bare `Pod` sits in `Failed` forever and nobody notices for a day. *Mechanism:* Topology Manager rejects at *admission on the node*, after scheduling. The kubelet marks the pod `Failed` with reason `TopologyAffinityError` and message `Resources cannot be allocated with Topology locality`; the scheduler does not reschedule terminal pods. *Fix:* always run these under a controller; if you must use bare pods, build a control loop that watches for the reason string.

- **Conflating VF creation with VF advertisement.** *Symptom:* "the device plugin is running, why are there zero VFs to schedule?" *Mechanism:* the SR-IOV network device plugin only *discovers* VFs that already exist. Creation is `SriovNetworkNodePolicy`, owned by the separate SR-IOV Network **Operator**, which writes `sriov_numvfs` and drains the node to do it. *Fix:* check `SriovNetworkNodeState` for the node before you look at the plugin's logs.

- **Trusting `NCCL_IB_HCA` to mean "the NIC I was allocated."** *Symptom:* every Kubernetes object is correct and GDR still does not engage, or engages on the wrong rail. *Mechanism:* in the default **shared** RDMA namespace mode (`ib_core netns_mode=1`), every RDMA device on the host is visible in every network namespace. A hardcoded `NCCL_IB_HCA` addresses whatever device that string matches, allocation notwithstanding. *Fix:* derive the HCA name from `$PCIDEVICE_<RESOURCE>` at container start via `/sys/bus/pci/devices/<pci>/infiniband/`, or move the node pool to `rdmaMode: exclusive` (a reboot).

- **Using the wrong delegate CNI for the link type.** *Symptom:* an InfiniBand VF that never comes up, or comes up without RDMA. *Mechanism:* `SriovNetwork` renders `"type": "sriov"`; `SriovIBNetwork` renders `"type": "ib-sriov"`. They are different plugins with different link handling, and InfiniBand additionally needs GUID/PKey management (`capabilities: {"infinibandGUID": true}` plus `ibKubernetes` and a UFM secret). *Fix:* match `linkType` in the NodePolicy to the network CR you use.

- **Treating the Network Operator as covering the whole GPUDirect stack.** *Symptom:* `NicClusterPolicy` reports `ready`, and there is still no `/GDRDMA` in the NCCL log. *Mechanism:* the Network Operator does not create VFs (that is the SR-IOV Network Operator) and does not load `nvidia_peermem` (that is the GPU Operator, per the Network Operator's own README). *Fix:* treat "RDMA reaches the pod" and "the NIC can register HBM" as two separately-owned checks, and verify both.

## Self-check

- **Why isn't the default CNI enough for RDMA, and what exactly does Multus add?**
  **Answer:** An RDMA container needs four things: an interface bound to the fabric NIC with an address on the fabric subnet; the RDMA character devices `/dev/infiniband/uverbs<N>` and `rdma_cm`; a scheduling claim so two pods cannot hold the same VF; and `IPC_LOCK` plus an adequate memlock limit so `ibv_reg_mr` can pin pages. The default CNI supplies a single `eth0` on the cluster pod network and none of the rest — it has no way to attach a second interface bound to physical hardware, no way to place a character device in the container, and no accounting unit for hardware functions. Multus adds exactly one of those four: it is a meta-CNI that runs as *the* CNI, delegates to the cluster's real CNI to build `eth0`, then reads the pod's `k8s.v1.cni.cncf.io/networks` annotation and, for each entry, fetches the named `NetworkAttachmentDefinition` and invokes the CNI plugin its `spec.config` names to attach `net1`, `net2`, …. The RDMA-capable interface is built by that delegate — `sriov`, `ib-sriov`, `ipoib` or `macvlan` — not by Multus. The device nodes and the scheduling claim come from a device plugin; the capability comes from the pod spec.

- **Which component creates SR-IOV VFs, which advertises them, and which loads `nvidia_peermem`?**
  **Answer:** Three different owners. **Creation:** the SR-IOV Network **Operator** (`k8snetworkplumbingwg/sriov-network-operator`), driven by a `SriovNetworkNodePolicy` whose `numVfs` field causes the config daemon to write `sriov_numvfs` on the matching PF — a PCIe reconfiguration that requires zeroing the existing VFs first, which is why the operator drains the node. It can also generate the `NetworkAttachmentDefinition` from a companion `SriovNetwork` (renders `type: sriov`) or `SriovIBNetwork` (renders `type: ib-sriov`). **Advertisement:** the SR-IOV network **device plugin** (`k8snetworkplumbingwg/sriov-network-device-plugin`), a separate project, which discovers VFs that already exist, registers them with the kubelet as `$RESOURCE_PREFIX/<resourceName>`, attaches the `/dev/infiniband` device specs when `isRdma` is set, injects `PCIDEVICE_<RESOURCE>` env vars, and publishes NUMA affinity from `/sys/bus/pci/devices/<addr>/numa_node`. **`nvidia_peermem`:** neither of the above and not the NVIDIA Network Operator either — from driver v465 the module ships inside the GPU driver and the **GPU Operator** loads it, as the Network Operator's own README states. The NVIDIA Network Operator's job is to deploy and lifecycle DOCA-OFED, Multus, both device plugins, IPoIB and IPAM from one `NicClusterPolicy`.

- **Trace how a pod's VF PCI address reaches the CNI plugin, and name the one string that must match exactly.**
  **Answer:** The scheduler places the pod on a node with free counts of both `nvidia.com/gpu` and the RDMA resource — it is not topology-aware and does not check whether they can be co-located. The kubelet's device manager consults Topology Manager for a merged NUMA hint, then calls `Allocate()` on the SR-IOV device plugin, which returns the device specs, the mounts, and `PCIDEVICE_<RESOURCE>=0000:41:00.5`; that allocation is recorded in the kubelet's pod-resources API. The CRI creates the network namespace and Multus is invoked: it delegates to the default CNI for `eth0`, reads the pod's networks annotation, fetches the NAD, and reads the NAD's `k8s.v1.cni.cncf.io/resourceName` annotation. It then dials `/var/lib/kubelet/pod-resources/kubelet.sock`, builds a map from resource name to allocated device IDs, looks up that exact string, takes `DeviceIDs[Index]`, increments `Index` (which is why listing the same NAD twice yields two different VFs), and injects `"deviceID"` and `"pciBusID"` into the delegate's CNI config. The `sriov`/`ib-sriov` CNI finds the netdev for that PCI address, moves it into the pod's namespace as `net1`, and runs IPAM. **The string that must match exactly is the NAD's `k8s.v1.cni.cncf.io/resourceName` annotation against the resource name the device plugin advertises, prefix included** — a mismatch is a silent map miss producing an empty device ID, not an error.

- **What does Topology Manager's `single-numa-node` policy actually do to a pod it cannot satisfy, and name two ways the policy can be silently ineffective.**
  **Answer:** It rejects the pod at kubelet admission — after scheduling, on the node — putting it into a terminal `Failed` state with reason `TopologyAffinityError` and message `Resources cannot be allocated with Topology locality`. The scheduler does **not** retry a terminal pod, so a bare `Pod` is simply dead; these workloads must run under a Deployment, StatefulSet, Job or operator, or behind a control loop watching for that reason string. Two silent-ineffectiveness modes: (1) **a device with no topology hint.** The SR-IOV device plugin publishes `TopologyInfo` only when `/sys/bus/pci/devices/<addr>/numa_node` parses to ≥ 0 — a BIOS reporting -1, or `excludeTopology: true` in the policy, yields a device with no hint, and a hintless device can never conflict with the GPU's hint, so the strictest policy admits anything. (2) **CPU alignment that was never requested.** Topology Manager aligns devices for pods of all QoS classes, but CPU *pinning* additionally needs the `static` CPU Manager policy and a Guaranteed pod with an integer CPU request equal to its limit; a pod asking for `cpu: "300m"` gets device alignment and a default CPU hint, so its threads may still run on the wrong socket. A third, structural limit worth naming: the scheduler is not topology-aware, so on a fragmented cluster a pod can fail admission on node after node while every node's free counts look adequate.

- **Your pod is Running, every Kubernetes object is correct, `PCIDEVICE_*` names the right VF on the right NUMA node — and the NCCL log has no `/GDRDMA`. What do you check, in order?**
  **Answer:** (1) **Which NIC NCCL actually used.** Run `ibv_devices` inside the pod. If it lists more than one device, the node is in the default **shared** RDMA namespace mode (`ib_core netns_mode=1`), every HCA on the host is visible in the pod, and a hardcoded `NCCL_IB_HCA` may be addressing a NIC on a different rail than the GPU — Kubernetes is correct and NCCL is not. Compare the `NCCL_IB_HCA set to` line in the log against the device derived from `/sys/bus/pci/devices/$PCIDEVICE_*/infiniband/`. Fix by deriving the HCA name at runtime, or move the pool to `rdmaMode: exclusive`. (2) **Whether a kernel shim is present** (05 §8 cause 1): `ls /sys/module/nvidia_peermem/version` and grep the log for `DMA-BUF is available on GPU device`. If neither, the GPU Operator has not loaded `nvidia_peermem` and CUDA/driver are below 11.7 — the Network Operator being healthy tells you nothing about this. (3) **ACS** (05 §8 cause 3): if `/GDRDMA` is absent *and* you see `IBV_WC_LOC_PROT_ERR` completions, ACS is redirecting peer-to-peer transactions to the root complex; confirm with `lspci -vvv | grep ACSCtl` on the bridges. In a VM you cannot disable it and must enable ATS instead. (4) **The distance gate**: even with the right VF, confirm the GPU↔NIC pair is at `PXB` or closer in `nvidia-smi topo -m`, since NCCL's default level is `PATH_PXB` and anything at `PHB`/`SYS` is refused by design.

## Connections & what's next

This lesson is where 05's NCCL-level tuning meets the Kubernetes scheduler. Everything in 05 assumed a pod already had a rail-aligned NIC to point `NCCL_IB_HCA` at; this lesson is the machinery that produces such a pod — and, just as importantly, the several ways that machinery produces a pod that *looks* correct while NCCL uses the wrong device. It also extends the platform module's Topology Manager material from two aligned resources (CPU, GPU) to three, and shows that the third one's hint can go missing entirely.

**07** closes the module by turning all of this into the thing a procurement review actually wants: a bandwidth-and-dollar statement. Where this lesson taught you *how* a pod gets a rail-aligned NIC, 07 teaches you *what that placement is worth in dollars per GPU-hour* — the fabric capex and power that the alignment protects, oversubscription as the capex lever, SHARP (05) as the byte-reduction lever, and the break-even utilisation at which a more expensive fabric pays for itself. Every failure mode in §11 has a price, and 07 is where you learn to state it.

## References & further reading

**Primary sources — read directly and relied upon**

1. **Multus CNI** — https://github.com/k8snetworkplumbingwg/multus-cni — cloned and read (commit 19cd1ad). `docs/how-to-use.md` for the five annotation forms, `net1`/`net2` naming, cross-namespace `namespace/name` syntax, `@ifname` pinning, the `default-route` key, and the DRA-with-Multus requirements (Kubernetes 1.34+, the `k8s.cni.cncf.io/deviceID` and `k8s.cni.cncf.io/resourceName` device attributes, and the silent-skip behaviour). `pkg/k8sclient/k8sclient.go` and `pkg/types/conf.go` for the allocation join: the `k8s.v1.cni.cncf.io/resourceName` lookup, `DeviceIDs[Index]` with `Index++`, and the injection of `deviceID`/`pciBusID` into the delegate config. `pkg/kubeletclient/kubeletclient.go` for the pod-resources socket path `/var/lib/kubelet/pod-resources/kubelet.sock`.

2. **SR-IOV Network Operator** — https://github.com/k8snetworkplumbingwg/sriov-network-operator — cloned and read (commit 253299b). `api/v1/sriovnetworknodepolicy_types.go` for every `SriovNetworkNodePolicy` field used in §4 including `numVfs`, `deviceType` (`netdevice`/`vfio-pci`), `isRdma`, `linkType`, `excludeTopology`, `externallyManaged` and `priority` (0–99). `api/v1/sriovibnetwork_types.go` and `api/v1/helper.go` for `SriovIBNetwork` and the `CniType = "ib-sriov"` rendering. `bindata/manifests/cni-config/sriov/sriov-cni-config.yaml` for the generated NAD template (`cniVersion 1.0.0`, the `k8s.v1.cni.cncf.io/resourceName` annotation set to `$RESOURCE_PREFIX/<resourceName>`). `api/v1/sriovnetworkpoolconfig_types.go` and `pkg/host/internal/network/network.go` for `rdmaMode: shared|exclusive` implemented as `options ib_core netns_mode=<0|1>`. `api/v1/sriovoperatorconfig_types.go` for `enableInjector`, `disableDrain`, `configurationMode` and `useCDI`. `deployment/.../values.yaml` for the `resourcePrefix: "openshift.io"` default. **Correction to the previous version of this lesson:** the `deviceID` value `1021` is fine (ConnectX-7), but the CRD's own doc-comment lists only older IDs and there is no enum validation — verify with `lspci -nn` rather than trusting either list.

3. **SR-IOV network device plugin** — https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin — cloned and read (commit 8120421). README for the config schema, the **`intel.com` default resource prefix**, `deviceType` values, the selector semantics, the `netpf0#0,2-7,9` range syntax, and the `PCIDEVICE_<RESOURCE_NAME>` / `_INFO` environment variables. `pkg/devices/host.go` and `pkg/utils/utils.go` for `GetDevNode()` reading `/sys/bus/pci/devices/<addr>/numa_node` and returning -1 on failure; `pkg/devices/api.go` for `TopologyInfo` being attached only when `nodeNum >= 0`. `pkg/infoprovider/rdmaInfoProvider.go` for the `uverbs`/`umad`/`issm`/`rdma_cm` device specs and env vars.

4. **NVIDIA Network Operator** — https://github.com/Mellanox/network-operator — cloned and read (commit 287e39e). `api/v1alpha1/nicclusterpolicy_types.go` for the current `NicClusterPolicySpec` field set — `ofedDriver`, `rdmaSharedDevicePlugin`, `sriovDevicePlugin`, `ibKubernetes`, `secondaryNetwork` (**`multus`/`cniPlugins`/`ipoib` only**), `nvIpam`, `nicFeatureDiscovery`, `docaTelemetryService`, `nicConfigurationOperator`, `spectrumXOperator` — and `NicClusterPolicyStatus` with per-component `appliedStates`. `example/crs/mellanox.com_v1alpha1_nicclusterpolicy_cr-full.yaml` for a current, complete example. `README.md` for the statement that `nvidia_peermem` ships in the GPU driver from v465 and **the GPU Operator manages loading it**. `deployment/network-operator/values.yaml` for `resourcePrefix: "nvidia.com"`. **Corrections to the previous version of this lesson:** `secondaryNetwork.ipamPlugin` no longer exists (IPAM is the top-level `nvIpam`, or whereabouts via `cniPlugins`), and the OFED image is `doca-driver` with a `doca<X>-<Y>` version string rather than the older `mofed`-style `24.10-0.7.0.0`.

5. **`k8s-rdma-shared-dev-plugin`** — https://github.com/Mellanox/k8s-rdma-shared-dev-plugin — cloned and read (commit 3beab3b). README for the `configList` schema, the **`rdma` default resource prefix**, `rdmaHcaMax` as an advertised count rather than a quota, the 60-second default `periodicUpdateInterval`, and the selector set (`vendors`, `deviceIDs`, `drivers`, `ifNames`, `linkTypes`) with OR-within / AND-across semantics.

6. **Kubernetes — Topology Manager documentation and kubelet source** — https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/ and https://github.com/kubernetes/kubernetes — both read locally. The doc for the `container`/`pod` scopes, the four policies, "aligns pods of all QoS classes", the CPU-Manager-static/Guaranteed/integer-CPU requirement, the 8-NUMA-node limit with `max-allowable-numa-nodes` (GA since 1.35), and the explicit statements that a rejected pod enters `Terminated` with an admission failure and **the scheduler will not reschedule it**. `pkg/kubelet/cm/topologymanager/topology_manager.go` for the exact strings: `ErrorTopologyAffinity = "TopologyAffinityError"` and `Error() = "Resources cannot be allocated with Topology locality"`.

7. **DRANET (`kubernetes-sigs/dranet`)** — https://github.com/kubernetes-sigs/dranet — cloned and read (commit 9760b79). README for the DRA-plus-NRI architecture, the donation to the Kubernetes organisation at KubeCon NA 2025, and the paper's reported **59.6% `all_gather` / 58.1% `all_reduce`** bus-bandwidth improvement from topology-aware GPU-and-NIC scheduling. `pkg/apis/attributes.go` for the complete `dra.net/*` attribute list. `examples/demo_gke_rdma/nccl-gib-test.yaml` for the real `resource.k8s.io/v1` `DeviceClass` and `ResourceClaimTemplate` with CEL selectors. `site/content/docs/concepts/rdma.md` for the four-component decomposition of RDMA on Linux (HCA, `uverbsN`, netdev, `rdma link`). **Correction to the previous version of this lesson:** DRANET is an upstream Kubernetes SIG project, not a single hyperscaler's pilot on one node SKU.

8. **`network-resources-injector`** — https://github.com/k8snetworkplumbingwg/network-resources-injector — README fetched and read. The mutating-webhook behaviour: resolve the pod's `k8s.v1.cni.cncf.io/networks` annotation to NADs, read each NAD's `k8s.v1.cni.cncf.io/resourceName`, count repeats, patch matching requests and limits into the pod; plus hugepages-via-Downward-API and `k8s.v1.cni.cncf.io/nodeSelector` propagation. Deployed by the SR-IOV Network Operator via `SriovOperatorConfig.spec.enableInjector`.

**Optional depth — could not be fetched from this environment, and no claim in this lesson rests on them**

9. **NVIDIA Developer Blog — "Deploying GPUDirect RDMA on the EGX Stack with the NVIDIA Network Operator"** — https://developer.nvidia.com/blog/deploying-gpudirect-rdma-on-egx-stack-with-the-network-operator/ — `developer.nvidia.com` is blocked by this environment's egress proxy. **Not relied upon.** The Network Operator behaviour described in §10 comes from its repository and CRD types instead. Read the post for the vendor-canonical narrative version of the GPU-Operator-plus-Network-Operator pairing.

10. **Microsoft Azure AKS engineering blog — "Simplifying InfiniBand on AKS" (April 2025)** — https://blog.aks.azure.com/2025/04/11/infiniband-on-aks — blocked. **Not relied upon.** Useful, if you can reach it, as a named hyperscaler's account of running the exact stack in this lesson on managed Kubernetes.

11. **Microsoft Community Hub — "Running tightly coupled HPC/AI workloads with InfiniBand using NVIDIA Network Operator on AKS" (Oct 2024)** — https://techcommunity.microsoft.com/blog/azurehighperformancecomputingblog/running-tightly-coupled-hpcai-workloads-with-infiniband-using-nvidia-network-ope/4117209 — blocked. **Not relied upon.** A deeper operational walkthrough of `NicClusterPolicy`, node labelling and namespace setup, if reachable.

12. **Microsoft Azure AKS engineering blog — DRANET post (April 2026)** — https://blog.aks.azure.com/2026/04/01/dranet-rdma-optimization-for-ai-on-aks — blocked. **Not relied upon.** An earlier version of this lesson described DRANET primarily through this post, including specific node-SKU and AKS-version claims that could not be verified here; §12 is written instead from the upstream `kubernetes-sigs/dranet` repository and Multus's own DRA documentation.

13. **`Azure/aks-rdma-infiniband`** — https://github.com/Azure/aks-rdma-infiniband — not fetched. **Not relied upon.** Listed as a hands-on companion for the Practice section: runnable manifests and validation tooling for RDMA/InfiniBand/GPUDirect RDMA on AKS, useful if you want a second real manifest set to annotate alongside the ones in this lesson.
