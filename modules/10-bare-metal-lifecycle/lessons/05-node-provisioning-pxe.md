---
lesson: "10.5"
title: "Node provisioning: PXE to Ready"
module: "10"
concept: "Node provisioning: PXE to Ready"
status: not-started
est_time: "5h"
artifacts: []
---
# 10.5 · Node provisioning: PXE to Ready

> **Concept.** The netboot-to-Ready pipeline — iPXE → provisioning agent (Ironic/Tinkerbell) → firmware → OS image → cluster join — that lays the metal *beneath* the GPU Operator.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Why this matters

On a cloud, a node is an API call: the image, firmware, and kernel are someone
else's problem, and `nodes` appear `Ready` a minute after you scale a node pool.
On metal — the world CoreWeave, NVIDIA DGX Cloud, and every neocloud actually
run in — *you* own everything below the kubelet. A rack of HGX H100 nodes ships,
and between "powered on in the DC" and "schedulable GPU node" there are a dozen
steps that must be **automated, repeatable, and auditable across 40+ nodes at a
time**, not click-driven per host.

Get this wrong and the failure modes are expensive and silent: a node that
booted with stale NIC firmware negotiates the wrong link speed and tanks NCCL
all-reduce for a 500-GPU job; a host that didn't get "above-4G decoding / large
BAR" set in BIOS won't even enumerate all eight GPUs; a golden image with baked
drivers drifts out of sync with your GPU Operator's driver version. The
provisioning pipeline is the foundation the rest of the fleet stands on. This
lesson wires the **PXE you already know from on-prem** into a *fleet* workflow
(Metal3/Ironic or Tinkerbell), and shows exactly where it hands off to the GPU
Operator from lesson 04.

## What's new here

You already know the on-prem primitives: **PXE/iPXE, TFTP, DHCP options 66/67,
IPMI/Redfish BMCs, BMC power control.** Nothing there is new. What's new is
treating them as a **declarative, idempotent pipeline** driven by Kubernetes
CRDs instead of a Cobbler/Foreman config you hand-edit:

- **PXE/iPXE as one stage in a reconcile loop**, not a boot menu. A controller
  decides what a MAC address boots into based on desired state.
- **Metal3/Ironic** (`BareMetalHost` CRD → Ironic → IPMI/Redfish) or
  **Tinkerbell** (`Hardware`/`Workflow`/`Template` CRDs → Smee/Tink/Rufio) as
  the agent that owns firmware + imaging + power.
- **Firmware/BIOS as code** — RAID, BIOS settings, and BMC/NIC firmware applied
  from a manifest, versioned, before any OS lands.
- **The handoff boundary.** Lessons 04 (GPU Operator, driver rollout, GPU node
  lifecycle) and 05 (XID errors, NPD concepts, cordon/drain) all assume a
  `Ready` node *already exists*. This lesson is the layer **beneath** that: bare
  metal comes up first, joins the cluster with NVIDIA hardware present and
  labeled, and *then* the GPU Operator installs drivers. We do **not** bake GPU
  drivers into the golden image — that couples the mutable driver lifecycle to
  the immutable OS image and defeats lesson 04's rollout story.

## Core notes

### The netboot-to-Ready pipeline (7 stages)

```
1. Power on / PXE   BMC (Redfish/IPMI) sets one-time boot = PXE. NIC firmware
   (DHCP proxy)     requests DHCP → gets iPXE chainload (opt 67) + next-server.
        │
2. iPXE chainload   Firmware PXE ROM loads iPXE (HTTP-capable). iPXE fetches its
        │           script from the provisioning service (Smee / Ironic httpd).
        ▼
3. Agent ramdisk    Boots an in-memory OS with an agent:
        │             • Ironic → Ironic Python Agent (IPA) ramdisk
        │             • Tinkerbell → HookOS (the "Hook" in-memory installer)
        │           Runs entirely in RAM; the disk is untouched so far.
        ▼
4. Inspect + clean  Hardware introspection (CPU/RAM/NIC/GPU/disk inventory),
        │           then automated cleaning: wipe disks, reset RAID.
        ▼
5. Firmware / BIOS  Apply BIOS settings + RAID + BMC/NIC/board firmware to the
        │           declared version — WHILE in the ephemeral agent, disk empty.
        ▼
6. Write OS image   Stream a content-addressed OS image (raw/qcow2) to disk with
        │           a checksum; drop cloud-init/Ignition config; set boot order.
        ▼
7. Reboot + join    Boot the installed OS → kubelet + containerd start →
                    kubeadm/Talos join → node registers → NFD labels it →
                    node goes Ready. ── HANDOFF to lesson 04 (GPU Operator).
```

### Metal3 / Ironic — the CRD-driven model

Metal3 wraps OpenStack **Ironic** behind Kubernetes. You declare a
`BareMetalHost` (BMH); the **baremetal-operator** reconciles it via Ironic,
which drives the BMC over **Redfish** (preferred on modern GPU servers) or IPMI.

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: gpu-node-07
  namespace: metal3
spec:
  online: true
  bootMACAddress: "b8:ce:f6:00:07:aa"     # the PXE NIC
  bmc:
    address: redfish-virtualmedia://10.0.7.7/redfish/v1/Systems/1
    credentialsName: gpu-node-07-bmc      # Secret with BMC user/pass
  bootMode: UEFI
  automatedCleaningMode: metadata          # wipe metadata between provisions
  rootDeviceHints:
    model: "SAMSUNG MZ..."                 # deterministic root disk pick
  image:
    url: http://images.metal3/ubuntu-22.04-gpu-base.qcow2
    checksum: http://images.metal3/ubuntu-22.04-gpu-base.qcow2.sha256
    checksumType: sha256
  userData:                                # cloud-init: kubeadm join, labels
    name: gpu-node-07-userdata
    namespace: metal3
```

Firmware/BIOS is its own declarative surface. Ironic exposes per-host
`HostFirmwareSettings` and `HostFirmwareComponents`, plus RAID config on the
BMH, so "SR-IOV on, above-4G decoding on, boot order NIC-first, NIC firmware =
X.Y" is a manifest, not a KVM session. At fleet scale you pair Metal3 with
**Cluster API Provider Metal3 (CAPM3)** so a `Machine` object claims a BMH and
the whole cluster is a set of CRDs.

### Tinkerbell — the workflow model

Tinkerbell decomposes provisioning into containerized **actions** run by an
agent in **HookOS**. The components (post the "Boots→Smee" rename):

- **Smee** — DHCP/PXE/iPXE + boot-script server (formerly Boots).
- **Tink** — workflow engine; runs `Template` steps as `Workflow`s.
- **Hegel** — metadata service (instance data for the running install).
- **Rufio** — BMC controller: power/boot via `Machine`/`Job`/`Task` CRDs.
- **HookOS (Hook)** — the in-RAM installer OS the agent runs in.

A `Template` is an ordered list of actions (each a container image), e.g.
`stream image → write cloud-init → install grub → set next boot = disk`. This
is more "imperative pipeline as data" than Ironic's "declarative host state,"
and it's what you reach for when you want full control of each step or you're on
**CAPT** (Cluster API Provider Tinkerbell).

### Firmware before OS — why the order is fixed

Firmware/BIOS/RAID is applied in **stage 5, inside the ephemeral agent, before
the OS is written**, for four reasons:

1. **The OS depends on firmware features.** GPU nodes need above-4G decoding /
   large-BAR, SR-IOV, and correct NIC firmware set *before* the kernel
   enumerates the PCIe topology. Fix it after and you're rebooting anyway.
2. **Firmware flashes usually require a reboot.** You do not want to interrupt
   a live, production OS to reflash a BMC/NIC — do it while the disk is empty
   and nothing depends on uptime.
3. **Cleaning is destructive.** Wiping disks and resetting RAID belongs in the
   agent phase; it can't run under the OS you're about to install.
4. **Determinism.** Every node reaches the same firmware baseline *before*
   imaging, so the OS image is the only remaining variable. This is what makes
   re-imaging reproducible.

### Idempotent re-imaging

Re-imaging must be safe to run any number of times and converge to the same
result. Four mechanisms:

- **Declarative desired state.** The BMH/Workflow *is* the source of truth. The
  reconcile loop compares desired vs actual and acts only on drift — applying
  the same manifest twice is a no-op, not a double-install.
- **Content-addressed images + checksums.** `image.url` + `checksum` means "this
  exact image or fail," so two provisions of the same spec are byte-identical.
- **Automated cleaning between deprovision and provision.** Ironic's
  `automatedCleaningMode` (or a Tink wipe action) guarantees no stale state
  survives a re-image — no leftover partitions, no old RAID.
- **Idempotent first-boot config.** cloud-init/Ignition run declaratively; the
  same userData yields the same node, and Talos's machine config is fully
  immutable/reproducible by design.

The test: pull a node, re-apply its manifest, and it comes back
bit-for-bit the same node with the same name, labels, and join — no human diff.

### Where this hands off to lesson 04

Provisioning's job ends at **"Ready Kubernetes node with NVIDIA hardware present
and labeled."** Concretely, stage 7 finishes when:

- kubelet + containerd are up and the node has `kubeadm join`ed (or Talos
  joined) the control plane from lesson 01/03;
- **Node Feature Discovery** has labeled the node from PCI inventory —
  NVIDIA vendor `0x10de`, class `0x0302` (3D controller) — e.g.
  `feature.node.kubernetes.io/pci-10de.present=true`;
- the node is `Ready` but has **no GPU driver, no container toolkit, no device
  plugin** installed.

The **GPU Operator (lesson 04)** then selects that node by label and does the
driver rollout, container-toolkit install, device-plugin registration, and DCGM
setup — the mechanics you already learned. Provisioning does **not** touch
drivers. Keeping the driver *out* of the golden image is deliberate: it lets the
GPU Operator manage the driver as a separately-versioned, rollable artifact
(lesson 04) instead of forcing a full re-image for every CUDA/driver bump.

## Worked example: bring up `gpu-node-07`

1. **Enroll.** Create the `Secret` (BMC creds) + `BareMetalHost` above. The
   baremetal-operator registers it in Ironic and moves it to `available`.
2. **Redfish inspection.** Ironic sets one-time boot = PXE via Redfish, powers
   the host on; it PXEs → chainloads iPXE → boots the IPA ramdisk → returns full
   inventory (8× `10de` GPUs, 2× 400G NICs, NVMe root).
3. **Firmware/BIOS.** `HostFirmwareSettings` asserts above-4G decoding on,
   SR-IOV on, UEFI, NIC firmware pinned; RAID reset to pass-through for the NVMe
   root. Applied in the agent, disk still empty.
4. **Clean + image.** Automated cleaning wipes disks; Ironic streams
   `ubuntu-22.04-gpu-base.qcow2` (checksum-verified) to the root device; writes
   `userData` (cloud-init that installs kubeadm and runs `kubeadm join` with the
   node's bootstrap token + `--node-labels`).
5. **Reboot + join.** Ironic sets next boot = disk, reboots. cloud-init joins the
   cluster; NFD labels `pci-10de.present=true`; node → `Ready`.
6. **Handoff.** GPU Operator's node selector matches the label → installs the
   driver + toolkit + device plugin → node advertises `nvidia.com/gpu: 8`.

Re-imaging test: delete the node, set BMH back to `available`, re-apply the same
manifest → the loop cleans, re-images, and rejoins to an identical `gpu-node-07`.

## Practice (feeds the deliverable)

**Design a netboot-to-Ready workflow for a GPU node.** Deliver into
`practice/capex-vs-cloud/` a short doc containing:

1. **A diagram** of the 7-stage pipeline for one HGX node, labeling which
   component owns each stage (Smee/Ironic-httpd, IPA/HookOS, Redfish/Rufio,
   image server, kubelet/NFD) and **the exact handoff line to the GPU Operator**.
2. **A config sketch** — a real `BareMetalHost` (Metal3) *or* a Tinkerbell
   `Template`, showing image URL + checksum, firmware/BIOS assertions
   (above-4G, SR-IOV, UEFI), automated cleaning, and cloud-init/Ignition join.
3. **An idempotency note** — the three mechanisms that make re-imaging a no-op
   on a clean node and identical after a wipe.

**Hardware-free path:** stand up a **Tinkerbell sandbox** (vagrant/libvirt
`tink` sandbox) and run a template end-to-end against a VM, *or* generate a
**Talos Image Factory** (`factory.talos.dev`) schematic that includes the NVIDIA
extension, boot it via iPXE from the factory URL, and `talosctl` a node to
`Ready`. Capture the boot logs as evidence.

**Acceptance:** a documented, reproducible netboot-to-Ready workflow (diagram +
config + idempotency note) checked into the deliverable, with the GPU-Operator
handoff boundary called out explicitly.

## Self-check

**(a) Where does firmware update slot into the boot sequence, and why before the
OS image?**
**Answer:** Stage 5 — inside the ephemeral agent ramdisk (IPA/HookOS), after
inspection/cleaning and **before** the OS is written to disk. Before, because
the OS/kernel enumerates PCIe based on BIOS/firmware state (above-4G, SR-IOV,
NIC firmware) so those must be correct first; firmware flashes need a reboot you
don't want to inflict on a live OS; cleaning is destructive and can't run under
the target OS; and applying it first gives every node an identical baseline,
leaving the OS image as the only variable — which is what makes imaging
reproducible.

**(b) How do you make a node re-image idempotent?**
**Answer:** Drive it from declarative desired state (the `BareMetalHost`/
`Workflow`) so the reconcile loop acts only on drift and re-applying is a no-op;
pin the OS to a content-addressed image + checksum so provisions are
byte-identical; enable automated cleaning between deprovision and provision so no
stale partitions/RAID survive; and use idempotent first-boot config
(cloud-init/Ignition, or Talos's immutable machine config). The test: wipe and
re-apply the manifest → identical node, same name/labels/join, no human diff.

**(c) Where does bare-metal provisioning hand off to module 04's GPU Operator?**
**Answer:** At a `Ready` node with NVIDIA hardware **present and labeled but no
driver**: kubelet/containerd up, `kubeadm`/Talos joined, and NFD has labeled it
from PCI inventory (`pci-10de.present=true`). The GPU Operator then selects the
node by that label and does the driver rollout, container-toolkit, device-plugin,
and DCGM install (lesson 04). Drivers are deliberately **not** in the golden
image, so the operator can version/roll them independently of the OS.

## Resources

1. **Metal3 / Ironic** — <https://metal3.io/> — the `BareMetalHost` model,
   Ironic BMC drivers (Redfish/IPMI), cleaning, and `HostFirmwareSettings`;
   the reference for CRD-driven metal in Kubernetes.
2. **Tinkerbell docs** — <https://tinkerbell.org/> — Smee/Tink/Hegel/Rufio +
   HookOS, and the `Template`/`Workflow` action model for pipeline-as-data.
3. **Talos Image Factory** — <https://factory.talos.dev/> — schematic-based,
   extension-included (NVIDIA) iPXE images for a fully immutable, hardware-free
   netboot demo.
