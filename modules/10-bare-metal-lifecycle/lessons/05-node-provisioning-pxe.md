---
lesson: "10.5"
title: "Node provisioning: PXE to Ready"
module: "10"
concept: "Node provisioning: PXE to Ready"
status: not-started
est_time: "7h"
prev: "04-declarative-fleets-capi-talos.md"
next: "06-hardware-health-remediation-rma.md"
artifacts: []
sources: 8
---
# 10.5 · Node provisioning: PXE to Ready

> **Concept.** The netboot-to-Ready pipeline — iPXE → provisioning agent (Ironic/Tinkerbell) → firmware → OS image → cluster join — that lays the metal *beneath* the GPU Operator.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 04 gave you the *object model* for a fleet: `Cluster`/`KubeadmControlPlane`/
`MachineDeployment`/`Machine`, reconciled by CAPI controllers, with Metal3/Tinkerbell
as the bare-metal infrastructure providers and Talos as the immutable-OS answer to
drift. That lesson told you *what* a bare-metal provider does that a cloud provider
doesn't — "owns BMC control, PXE/virtual-media boot, hardware inspection, and disk
imaging" — as one sentence. This lesson opens that sentence up: it's the concrete,
step-by-step mechanics of what happens between a `Machine` object being created and
a `BareMetalHost` reaching `Ready` state. You'll walk the exact seven stages a
physical GPU server passes through, see the CRDs and configs that drive each one,
and land at the precise handoff line — a labeled, driverless node — where lesson 04's
GPU Operator picks up the baton. Where lesson 04 was the fleet's *declarative shape*,
this lesson is the fleet's *physical bootstrap sequence*, one host at a time,
automated across dozens at once.

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
drivers drifts out of sync with your GPU Operator's driver version. This is also
a direct interview probe at neoclouds — NVIDIA's "Sr SSE, Kubernetes Node
Lifecycle, DGX Cloud" posting literally asks for "scalable node provisioning"
built on CAPI providers, and CoreWeave's platform roles describe "provisioning
bare-metal and virtual clusters with Cluster API… day-2 lifecycle." The
provisioning pipeline is the foundation the rest of the fleet stands on, and at
scale it *is* a revenue-relevant capability: vCluster's bare-metal-provisioning
writeup on the hidden costs of GPU fleets names CoreWeave as running a
zero-touch provisioning platform across **100,000+ GPU nodes** in production —
the pipeline you're building a toy version of in this lesson is, at that
company, the thing that turns a rack delivery into billable capacity in hours
instead of weeks. This lesson wires the **PXE you already know from on-prem**
into a *fleet* workflow (Metal3/Ironic or Tinkerbell), and shows exactly where
it hands off to the GPU Operator from lesson 04.

## What's new here (calibration)

You already know the on-prem primitives: **PXE/iPXE, TFTP, DHCP options 66/67,
IPMI/Redfish BMCs, BMC power control.** Nothing there is new, and we don't
re-teach it. What's new is treating them as a **declarative, idempotent
pipeline** driven by Kubernetes CRDs instead of a Cobbler/Foreman config you
hand-edit:

- **PXE/iPXE as one stage in a reconcile loop**, not a boot menu. A controller
  decides what a MAC address boots into based on desired state, and re-applies
  itself the same way every time.
- **Metal3/Ironic** (`BareMetalHost` CRD → Ironic → IPMI/Redfish) or
  **Tinkerbell** (`Hardware`/`Workflow`/`Template` CRDs → Smee/Tink/Rufio) as
  the agent that owns firmware + imaging + power, and how those two models
  differ (declarative host state vs. imperative pipeline-as-data).
- **Firmware/BIOS as code** — RAID, BIOS settings, and BMC/NIC firmware applied
  from a manifest, versioned, before any OS lands — and *why* the ordering is
  fixed, not just that it happens.
- **The precise handoff boundary to lesson 04.** Lessons 04 (GPU Operator,
  driver rollout, GPU node lifecycle) and 06 (XID errors, NPD, cordon/drain)
  all assume a `Ready` node *already exists*. This lesson is the layer
  **beneath** that: bare metal comes up first, joins the cluster with NVIDIA
  hardware present and labeled, and *then* the GPU Operator installs drivers.
  We do **not** bake GPU drivers into the golden image — that couples the
  mutable driver lifecycle to the immutable OS image and defeats lesson 04's
  rollout story.

## Core concepts

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

Every stage above is *idempotent and re-entrant* by design (see below) — that's
the entire point of running it as a Kubernetes reconcile loop instead of a
one-shot install script.

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
the whole cluster is a set of CRDs — this is the bridge back to lesson 04's
object model: a `Machine` binds to a `Metal3Machine`, which binds to a `BMH`,
which is what actually runs the seven stages below it.

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

**Metal3 vs Tinkerbell, the one-line distinction:** Ironic asks "what state
should this host be in?" and figures out the steps; Tinkerbell asks "what
sequence of steps should this host run?" and executes them in order. Both reach
the same seven stages — they differ in whether the *state* or the *sequence* is
the unit you author and version.

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

## Perspectives

**The operator's view.** You watch a `BareMetalHost` (or Tinkerbell
`Workflow`) march through phases — `registering` → `inspecting` → `cleaning` →
`provisioning` → `provisioned` — the same "declare desired state, watch the
reconcile loop converge" pattern from lesson 04's `Machine` phases, just one
layer lower. Your job is to make the manifest right once; the controller does
the repetition across every node in the rack.

**The hardware/firmware view.** Nothing above stage 5 exists without a
correctly configured BMC. Redfish (a REST/JSON API over HTTPS) is the modern
target; IPMI is the decades-old fallback still common on older or cheaper
boards. Firmware is not a formality — above-4G decoding is the literal
BIOS setting that lets the kernel map all eight GPUs' BARs into address space
above 4 GB; skip it and `lspci` shows GPUs the kernel can't fully use.

**The failure-mode view.** This pipeline fails loudly in a few characteristic
ways: DHCP proxy misconfiguration means the node never gets an iPXE chainload
(nothing shows in Ironic, because it never even PXE-booted); a stale/incorrect
`bootMACAddress` means the BMH targets the wrong NIC; a checksum mismatch on
the image fails stage 6 cleanly (the safe failure); and — the expensive one —
a firmware step that silently doesn't apply (e.g. above-4G decoding stays off)
produces a node that *looks* Ready but only exposes some GPUs, which surfaces
downstream as an NCCL topology mismatch, not a provisioning error. This is why
stage 4's inspection output (full GPU/NIC inventory) is worth diffing against
expected counts before trusting a node.

**The economics view.** Every hour a delivered rack sits un-provisioned is
capacity you've paid capex for but can't bill. A fully automated PXE→Ready
pipeline is the difference between "new rack online in hours" and "new rack
online in weeks of manual bring-up" — and at fleet scale (CoreWeave's
100,000+ GPU nodes, per vCluster's writeup) that gap is the business. It's
also why the pipeline must be *auditable*: firmware/image versions per node
feed the same per-SKU tracking you'll use in lesson 06 to push bad hardware
batches back on the vendor.

## Real-world use cases

- **vCluster — "Bare Metal GPU Provisioning Infrastructure Hidden Costs"**
  ([vcluster.com/blog/gpu-provisioning-platforms-ai-clouds](https://www.vcluster.com/blog/gpu-provisioning-platforms-ai-clouds))
  — describes a zero-touch provisioning platform (vMetal) handling PXE boot,
  OS install, machine registration, and full server lifecycle, and names
  **CoreWeave as a customer operating on this model across 100,000+ GPU nodes
  in production** — the best available named-company, numbers-backed
  demonstration that this lesson's pipeline is exactly what production
  neoclouds run, just automated at a scale beyond any one engineer clicking
  through a BMC console.
- **CoreWeave — "What Is Node Lifecycle Management and Why Does It Matter for
  ML Training and Inference?"**
  ([coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference](https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference))
  — describes CoreWeave's periodic **"HPC Verification" tests** (roughly 20
  minutes, exercising all GPUs on a node) as part of the provisioning-to-health
  pipeline: a burn-in/validation stage that belongs right after this lesson's
  stage 7, before a node is trusted with a real job. (This post is also
  lesson 06's primary use case, for the RMA side of the same lifecycle — the
  cross-reference is deliberate: provisioning and remediation are two ends of
  one loop.)
- **Sidero Labs / Equinix case study — "Equinix switches from Kubespray to
  Talos Linux, cutting deployment time while maintaining security"**
  ([siderolabs.com/case-studies/equinix-switches-from-kubespray-to-talos-linux…](https://www.siderolabs.com/case-studies/equinix-switches-from-kubespray-to-talos-linux-cutting-deployment-time-while-maintaining-security))
  — Equinix cut Kubernetes deployment time from **45 minutes to under 10
  minutes** by moving off Kubespray/Ansible onto Talos's immutable, API-driven
  model. That's fundamentally a provisioning-pipeline win: fewer imperative
  steps between "machine exists" and "cluster member" (full immutable-OS story
  is lesson 04's; here it's cited as evidence that pipeline design, not just
  hardware speed, is what moves this number).

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

## Common pitfalls

- **Baking the GPU driver into the golden image.** It's tempting — one image,
  one boot, done. But it couples driver version to OS image version, so every
  CUDA/driver bump forces a full re-image across the fleet instead of a rolling
  DaemonSet update. Keep drivers out of the image; let the GPU Operator own
  them (the whole point of the handoff boundary in this lesson).
- **Applying firmware after the OS is installed.** If BIOS/firmware settings
  are only fixed post-install (a "fix it in prod" habit from VM-land), you pay
  for a live reboot on every firmware drift instead of catching it in the
  disposable agent stage where a reboot is free.
- **Trusting `Ready` without checking GPU count.** A node can join the cluster
  and go `Ready` with a firmware misconfiguration that hides GPUs from the
  kernel (e.g. above-4G decoding off). `Ready` means "kubelet is happy," not
  "all 8 GPUs are visible" — always diff NFD/inspection output against the
  expected count before scheduling real jobs.
- **Treating IPMI and Redfish as interchangeable.** IPMI is older, less
  structured (freeform OEM extensions), and weaker on virtual-media boot;
  Redfish is the modern, REST-based standard most current GPU server BMCs
  support well. Prefer Redfish; only fall back to IPMI where the board forces
  it.
- **No checksum on the OS image.** Skipping `checksum`/`checksumType` on the
  `BareMetalHost` image means a corrupted or wrong image installs silently —
  the whole idempotency story depends on "this exact image or fail," not "some
  image, probably."

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

**(d) Why is Metal3/Ironic described as "declarative host state" and Tinkerbell
as "imperative pipeline as data" if both reach the same seven stages?**
**Answer:** Ironic's `BareMetalHost` describes the *end state* you want a host
in (`provisioned`, with a given image/firmware) and Ironic's own logic figures
out and executes the steps to get there — you author state, not sequence.
Tinkerbell's `Template`/`Workflow` is an explicit, ordered list of action
containers (`stream image → write cloud-init → install grub → …`) that Tink
executes in that exact order — you author the sequence itself. Both still pass
through inspect/clean/firmware/image/join; the difference is which layer (state
vs sequence) is the thing you version and diff.

## Connections & what's next

This lesson is the physical floor under lesson 04's object model: a `Machine`
is only as real as the `BareMetalHost`/`Workflow` reconciling underneath it,
and everything in lesson 04's CAPI story (scaling `MachineDeployment.replicas`,
rolling upgrades, Talos's immutable machine config) ultimately triggers this
seven-stage pipeline on real hardware. It also sets up module 09's networking
concerns (NIC firmware and link negotiation determine whether NCCL gets the
fabric bandwidth it expects) and feeds forward into lesson 08's economics (a
slow or manual provisioning pipeline is idle, unbilled capex).

The thread that carries forward directly: this lesson ends at a labeled,
driverless `Ready` node — and that is also where the *back edge* of lesson 06's
closed loop lands. When a node fails hard enough to be RMA'd, the replacement
hardware doesn't get reinstalled by hand; it re-enters **this exact pipeline**
(`BareMetalHost` back to `available` → re-image → rejoin). Lesson 06 is what
decides *when* a node needs to come back through here — the detect → isolate →
decide → RMA loop that closes around the pipeline you just built.

## References & further reading

**Primary sources**
- **Metal3 / Ironic** — <https://metal3.io/> — the `BareMetalHost` model,
  Ironic BMC drivers (Redfish/IPMI), cleaning, and `HostFirmwareSettings`;
  the reference for CRD-driven metal in Kubernetes.
- **Tinkerbell docs** — <https://tinkerbell.org/> — Smee/Tink/Hegel/Rufio +
  HookOS, and the `Template`/`Workflow` action model for pipeline-as-data.
- **Talos Image Factory** — <https://factory.talos.dev/> — schematic-based,
  extension-included (NVIDIA) iPXE images for a fully immutable, hardware-free
  netboot demo.

**Real-world engineering blogs**
- **vCluster — "Bare Metal GPU Provisioning Infrastructure Hidden Costs"** —
  <https://www.vcluster.com/blog/gpu-provisioning-platforms-ai-clouds> — names
  CoreWeave running a zero-touch provisioning platform across 100,000+ GPU
  nodes; the numbers-backed case that this lesson's pipeline is production
  reality at fleet scale.
- **CoreWeave — "What Is Node Lifecycle Management and Why Does It Matter for
  ML Training and Inference?"** —
  <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference>
  — the "HPC Verification" burn-in tests that follow provisioning before a node
  is trusted with real jobs; the shared anchor with lesson 06.
- **Sidero Labs — Equinix case study** —
  <https://www.siderolabs.com/case-studies/equinix-switches-from-kubespray-to-talos-linux-cutting-deployment-time-while-maintaining-security>
  — "45 min → under 10 min" cluster deploy time as a provisioning-pipeline-design
  win, not just a hardware-speed one.

**Deeper dives**
- **Ironic project docs (OpenStack)** — the state machine and driver interfaces
  Metal3 wraps; useful once `BareMetalHost` behavior needs debugging past what
  Metal3's own docs cover — linked from <https://metal3.io/>.
- **Cluster API Book, bare-metal provider chapters** — <https://cluster-api.sigs.k8s.io/>
  — for tracing how a `Machine` (lesson 04) actually claims and drives a
  `BareMetalHost` end to end via CAPM3/CAPT.
