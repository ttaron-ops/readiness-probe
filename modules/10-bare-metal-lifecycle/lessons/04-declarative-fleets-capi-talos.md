---
lesson: "10.4"
title: "Declarative fleets (Cluster API + Talos)"
module: "10"
concept: "Declarative fleets (Cluster API + Talos)"
status: not-started
est_time: "6h"
artifacts: []
---
# 10.4 · Declarative fleets (Cluster API + Talos)

> **Concept.** Hand-building one control plane teaches you the parts; running 40+ clusters demands they be reconciled objects. Cluster API turns "a cluster" into a Kubernetes resource, and immutable-OS distributions like Talos turn "a node" into a declarative artifact — so a fleet is `kubectl apply`, not a runbook.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Why this matters

Lessons 1–3 built a control plane by hand: KTHW, etcd, kubeadm HA, a VIP. That
does not scale to a GPU neocloud running dozens of clusters across thousands of
nodes. You cannot SSH-and-runbook your way through 40 clusters, nor re-derive an
upgrade by hand on each. The industry answer is **declarative fleet
provisioning**: describe the desired cluster as API objects and let a controller
reconcile reality toward them — exactly the reconcile/CRD model from
**K8s-controllers 02**, now pointed at whole clusters and bare-metal machines
instead of Pods.

This is also the literal job target. **NVIDIA's "Kubernetes Node Lifecycle" and
CoreWeave-style neocloud roles build Cluster API providers in Go** — the machine
that turns a `Machine` object into a provisioned GPU node with the right
firmware, is itself a controller you write. Understanding CAPI's object model and
where a bare-metal provider plugs in is the on-ramp to that work, and the
immutable-OS angle (Talos) is how those fleets stay drift-free at 200+ nodes.
Your differentiator — cost/observability — lives here too: a declarative fleet
gives you a single inventory of clusters/machines to bill, right-size, and
audit.

## What's new here (managed → self-managed → declarative)

| | Managed (EKS/GKE) | Self-managed hand-built (L1–3) | Declarative fleet (this lesson) |
|---|---|---|---|
| Unit of work | A console/API call | A runbook per cluster | A `Cluster` + `Machine*` object |
| Who reconciles | Cloud's controllers | You, by hand | Your management cluster's CAPI controllers |
| Scaling nodes | Managed node group | kubeadm join per node | `MachineDeployment.replicas: N` |
| Upgrades | One API call | Manual, per node, ordered | Change the version field; rollout is reconciled |
| Node OS | AMI (cloud-managed) | Ubuntu + config mgmt (drifts) | Immutable image (Talos), no shell |

The leap is that CAPI **reuses managed-Kubernetes ergonomics** (`kubectl`,
declarative objects, controllers, rollouts) but the objects describe
*infrastructure you own*. You bring the reconcile-loop intuition from 02; CAPI
just has more CRDs and swappable providers.

## Core notes

### 1. Cluster API (CAPI): the object model

CAPI is a set of controllers running in a **management cluster** that reconcile
**workload clusters** into existence. Core CRDs (provider-agnostic):

- **`Cluster`** — the top-level object; references an infrastructure provider
  object and a control-plane provider object. "I want a cluster with this
  network config."
- **`KubeadmControlPlane` (KCP)** — declares the control plane: replica count
  (use odd — 3/5), Kubernetes version, and the machine template to stamp CP nodes
  from. KCP owns rolling CP upgrades and etcd management for you.
- **`MachineDeployment`** — the worker analogue of a Deployment:
  `replicas: N`, a version, and a machine template. Scaling workers = edit
  `replicas`; upgrading = edit `version` and KCP/MD do a rolling replacement.
- **`MachineSet` / `Machine`** — a `Machine` is the atomic "one Kubernetes node"
  object. `MachineDeployment → MachineSet → Machine`, mirroring
  `Deployment → ReplicaSet → Pod`. This parallel is the whole mental model.

Two orthogonal provider slots per cluster:

- **Infrastructure provider** — knows how to *make a machine exist*: AWS (CAPA),
  vSphere (CAPV), **Metal3** (bare metal), **Tinkerbell/CAPT** (bare metal),
  Docker (**CAPD**, for testing). It reconciles an `InfraMachine`
  (e.g. `Metal3Machine`, `DockerMachine`).
- **Bootstrap provider** — knows how to *turn a blank machine into a K8s node*:
  usually **kubeadm** (CABPK), which generates the cloud-init/ignition that runs
  `kubeadm init/join`. Talos has its own bootstrap+control-plane providers.

A `Machine` is provider-neutral; it binds to one `InfraMachine` (where it runs)
and one `BootstrapConfig` (how it becomes a node). That separation is exactly why
you can swap cloud → bare metal without changing the top-level `Cluster` shape.

### 2. Bare-metal providers: what they do that cloud providers don't

A **cloud** CAPI provider (CAPA/CAPG) turns a `Machine` into a node by calling an
API that already virtualizes everything: "RunInstances" hands back a booted VM
with an OS image, network, and disk in seconds. The hard parts are abstracted.

A **bare-metal** provider has no such API — it must drive *physical* hardware:

- **Metal3 + Ironic** (the CNCF **incubating** project as of Aug 2025): Metal3
  models each physical server as a **`BareMetalHost` (BMH)** CRD. Its controller
  drives **Ironic** (from OpenStack) to, over **IPMI/Redfish (BMC)**:
  **enroll and inspect** the machine (hardware facts: CPUs, RAM, NICs, disks),
  **clean/wipe** disks, **power** it on/off, and **provision** an OS by writing
  an image to disk (IPA — Ironic Python Agent — via PXE/virtual media). Only then
  does the kubeadm/Talos bootstrap run. So a bare-metal provider owns: BMC
  control, PXE/DHCP/TFTP or virtual-media boot, disk imaging, hardware
  inspection, and de-provisioning/cleaning — an entire layer the cloud API hid.
- **Tinkerbell / CAPT**: same class of problem (netboot + workflow-driven
  provisioning of physical machines) with a workflow engine (Boots/Hegel/Tink)
  instead of Ironic.

The one-line answer: a bare-metal provider **owns the physical
provisioning lifecycle — BMC/IPMI-Redfish power control, PXE/virtual-media boot,
hardware inspection, and disk imaging/cleaning** — because there is no cloud API
to hand it a ready OS. This is the code NVIDIA/CoreWeave node-lifecycle roles
write and extend.

### 3. Talos: the immutable-OS alternative

Talos Linux is a Kubernetes-purpose OS with a radically different node model:

- **No SSH, no shell, no package manager, no systemd, no interactive login.** The
  only interface is a **gRPC API** driven by `talosctl`. There is nothing to
  `ssh` into and hand-edit.
- **Immutable, mostly read-only root filesystem.** The OS ships as a signed
  image; configuration is a single declarative **machine config** (YAML) applied
  via the API. You don't configure a Talos node, you *declare* it.
- **A/B (dual-partition) image upgrades:** `talosctl upgrade` writes the new OS
  image to the inactive partition and reboots into it; failure rolls back to the
  known-good partition. Upgrades are atomic image swaps, not in-place package
  churn.

**Why this kills config drift across 200 nodes:** with a mutable distro
(Ubuntu + Ansible/kubeadm), every node accumulates hand-fixes, half-applied
config-management runs, and package-version skew — 200 nodes become 200
*slightly different* nodes, and drift is where 3am outages live. Talos removes the
mechanisms that create drift: there is **no way to make a one-off change** (no
shell), config is a single declared artifact applied identically to every node,
and the root FS is immutable so nothing mutates it at runtime. Two nodes on the
same Talos version + same machine config are **bit-for-bit equivalent by
construction**; an upgrade is the same atomic image on all of them. Drift stops
being something you police and becomes something the design forecloses. (This is
also the CAPI Talos providers' selling point for fleets.)

### 4. Management cluster vs workload clusters

The **management cluster** is a (usually small, long-lived) Kubernetes cluster
that runs the CAPI controllers + provider controllers. It holds the `Cluster`,
`KubeadmControlPlane`, `MachineDeployment`, `Machine`, `BareMetalHost`… objects
and **reconciles workload clusters** — the actual clusters that run your GPU
workloads — toward those specs. It continuously drives: provisioning new
machines, replacing failed ones, scaling worker counts, and rolling
version upgrades. `clusterctl` bootstraps a management cluster and installs
providers. (You can even make a management cluster manage itself by "pivoting"
the objects into a freshly created cluster.) One management cluster can reconcile
many workload clusters — this is the "40+ clusters as objects" endgame.

## Worked example — CAPI Docker provider (CAPD): watch a Machine reconcile

CAPD provisions "machines" as Docker containers, so you can see the exact
reconcile loop that a bare-metal provider runs — without any hardware.
Prereqs: Docker, `kind`, `clusterctl`, `kubectl`.

```bash
# 1. Create a kind cluster to be the MANAGEMENT cluster
kind create cluster --name mgmt

# 2. Install CAPI core + kubeadm bootstrap/control-plane + Docker infra provider
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure docker

# 3. Generate a workload-cluster manifest (1 CP, 1 worker) and apply it
clusterctl generate cluster demo \
  --flavor development \
  --kubernetes-version v1.31.0 \
  --control-plane-machine-count=1 \
  --worker-machine-count=1 > demo.yaml
kubectl apply -f demo.yaml

# 4. WATCH THE RECONCILE LOOP turn objects into machines
clusterctl describe cluster demo          # tree of Cluster→KCP/MD→Machines
kubectl get machines -w                     # Pending→Provisioning→Running
kubectl get kubeadmcontrolplane,machinedeployment
```

You'll see the `Machine` objects march through phases as the controller creates
the backing Docker "nodes", then the kubeadm bootstrap provider runs
`kubeadm init/join` inside them. Then **scale and observe reconciliation**:

```bash
kubectl scale machinedeployment demo-md-0 --replicas=3   # desired state changes
kubectl get machines -w                                   # 2 new Machines appear
```

Fetch the workload cluster's kubeconfig and confirm it's a real cluster:

```bash
clusterctl get kubeconfig demo > demo.kubeconfig
kubectl --kubeconfig demo.kubeconfig get nodes
```

The key observation for your target job: **the `Machine` is the reconciled unit.**
Swap `--infrastructure docker` for `--infrastructure metal3`, and the *same*
`Machine`/`MachineDeployment` objects now drive Ironic to PXE-boot physical
servers. The control plane you built by hand in L1–3 is what KCP + the kubeadm
bootstrap provider now produce automatically.

## Practice (hands-on, cheap VMs → deliverable)

Pick **one** path (either satisfies acceptance):

**Path A — Talos (fastest reproducible bare-metal-like cluster):**
```bash
# Docker-based Talos cluster (or use QEMU/a VM for a truer bare-metal feel)
talosctl cluster create --name demo            # brings up CP+worker
talosctl --nodes <cp-ip> get members
talosctl config nodes <cp-ip>; talosctl kubeconfig .
kubectl get nodes
# Inspect the immutable/declarative model + A/B upgrade path:
talosctl get machineconfig -o yaml | head -40   # the single declared artifact
talosctl upgrade --nodes <cp-ip> --image ghcr.io/siderolabs/installer:<newer>
```
Observe: there is **no SSH**; the only access is the API. Capture the machine
config and the upgrade output as evidence of the immutable/A-B model.

**Path B — CAPI + Docker (CAPD), to watch the reconcile loop:**
Run the worked example above. Capture `clusterctl describe cluster demo`,
`kubectl get machines` transitioning through phases, and a `scale` that spawns new
`Machine` objects.

**Acceptance (feeds the capex-vs-cloud writeup):** a **Talos-provisioned OR
CAPI-provisioned cluster**, plus a short note documenting the **reconcile
behavior you observed** — for CAPD: the `Machine` phase transitions and a
`replicas` change producing/removing `Machine`s; for Talos: the single declarative
machine config + the A/B upgrade. Add 3–5 sentences mapping the objects
(`Cluster`/`KubeadmControlPlane`/`MachineDeployment`/`Machine`, or Talos machine
config) to the hand-built control plane from L1–3, and one line on where a
**bare-metal provider (Metal3/Ironic)** would slot in for real hardware. This is
the "how the fleet scales past hand-building" section of the deliverable.

## Self-check

**(a) What does a bare-metal CAPI provider (Metal3/Ironic) do that a cloud CAPI
provider doesn't?**
**Answer:** It owns the **physical** provisioning lifecycle that a cloud API
hides. A cloud provider just calls an API ("RunInstances") that returns a booted,
imaged VM. Metal3 models each server as a `BareMetalHost` and drives **Ironic**
over **IPMI/Redfish (BMC)** to power the machine on/off, **inspect** its hardware,
**clean/wipe** disks, and **provision an OS image via PXE/virtual media** — none
of which exists as an API on bare metal. Only after that does bootstrap
(kubeadm/Talos) run.

**(b) Why does an immutable OS (Talos) reduce config drift across 200 nodes?**
**Answer:** It removes the mechanisms that create drift. There's **no shell/SSH/
package manager**, so no one can make a one-off change; the root filesystem is
**immutable**; and the entire node is defined by a **single declarative machine
config** applied identically everywhere. Upgrades are **atomic A/B image swaps**,
not in-place package churn. Two nodes on the same version + config are equivalent
by construction, so drift is foreclosed by design rather than policed after the
fact.

**(c) What is a management cluster and what does it reconcile?**
**Answer:** A management cluster is a Kubernetes cluster running the Cluster API
core controllers plus infrastructure/bootstrap provider controllers. It holds the
CAPI objects (`Cluster`, `KubeadmControlPlane`, `MachineDeployment`, `Machine`,
`BareMetalHost`, …) and **reconciles the workload clusters** those objects
describe — creating/replacing/scaling their machines and rolling their version
upgrades toward the declared spec. One management cluster can reconcile many
workload clusters, which is how a fleet of 40+ clusters becomes a set of objects
instead of runbooks.

## Resources

1. **The Cluster API Book (skim the concepts, then go deep on the quick-start /
   CAPD flow you ran):** https://cluster-api.sigs.k8s.io/
2. **Talos docs (hands-on: `talosctl cluster create`, machine config, upgrades):**
   https://www.talos.dev/latest/
3. **Metal3 (the bare-metal provider — `BareMetalHost`/Ironic model; CNCF
   incubating as of Aug 2025):** https://metal3.io/
