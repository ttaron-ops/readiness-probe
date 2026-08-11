---
lesson: "10.4"
title: "Declarative fleets: Cluster API + Metal3 + Talos"
module: "10"
concept: "Declarative fleets (Cluster API + Talos)"
status: not-started
est_time: "8h"
prev: "03-control-plane-ha.md"
next: "05-node-provisioning-pxe.md"
artifacts: []
sources: 10
---
# 10.4 · Declarative fleets: Cluster API + Metal3 + Talos

> **Concept.** Hand-building one control plane teaches you the parts; running 40+ clusters demands they be reconciled objects. Cluster API turns "a cluster" into a Kubernetes resource, and immutable-OS distributions like Talos turn "a node" into a declarative artifact — so a fleet is `kubectl apply`, not a runbook.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 10.3 gave you a 3-node HA control plane behind a VIP, and a hand-run
sequence for upgrading it without violating version skew. That runbook works
for one cluster. It does not work for forty. This lesson takes every manual
step from 10.1–10.3 — provision nodes, bootstrap a control plane, wire up a
VIP, sequence an upgrade — and shows how the industry turns each of them into
a **reconciled Kubernetes object**, so that "build a cluster" becomes "apply a
manifest" and "upgrade the fleet" becomes "edit a version field." This is the
lesson where the module's individual, hand-cranked lessons compose into
something that scales to a real neocloud's fleet size.

## Why this matters

Lessons 10.1–10.3 built a control plane by hand: KTHW, etcd, kubeadm HA, a
VIP. That does not scale to a GPU neocloud running dozens of clusters across
thousands of nodes. You cannot SSH-and-runbook your way through 40 clusters,
nor re-derive an upgrade by hand on each one. The industry answer is
**declarative fleet provisioning**: describe the desired cluster as API
objects and let a controller reconcile reality toward them — exactly the
reconcile/CRD model from **K8s-controllers 02**, now pointed at whole
clusters and bare-metal machines instead of Pods.

This is also the literal job target. **NVIDIA's "Kubernetes Node Lifecycle"
and CoreWeave-style neocloud roles build Cluster API providers in Go** — the
machinery that turns a `Machine` object into a provisioned GPU node with the
right firmware is itself a controller you write. Understanding CAPI's object
model and where a bare-metal provider plugs in is the on-ramp to that work,
and the immutable-OS angle (Talos) is how those fleets stay drift-free at
200+ nodes. Your differentiator — cost/observability — lives here too: a
declarative fleet gives you a single inventory of clusters/machines to bill,
right-size, and audit. And it is not a niche pattern: Deutsche Telekom,
Vodafone, Orange, Telefónica, and Telecom Italia have all independently
converged on this exact stack for telco-scale bare-metal fleets (see
Real-world use cases) — this is the industry-standard answer, not one
vendor's opinion.

## What's new here (calibration)

You already own: the reconcile-loop/CRD mental model (**K8s-controllers
02**), what a control plane is made of and how to HA it by hand (**10.1–10.3**),
and etcd operations (**10.2**). What's genuinely new:

- **The CAPI object model** — `Cluster`, `KubeadmControlPlane`,
  `MachineDeployment`, `MachineSet`, `Machine`, and the two orthogonal
  provider slots (infrastructure, bootstrap) that let the same objects target
  cloud or bare metal.
- **What a bare-metal infrastructure provider (Metal3/Ironic) does that a
  cloud provider doesn't** — it owns the physical provisioning lifecycle a
  cloud API hides entirely.
- **Talos's immutable-OS node model** — no shell, no package manager, a
  single declarative machine config, atomic A/B upgrades — a genuinely
  different way to think about "what a node is" than the Ubuntu-plus-config-
  management model you've operated before.
- **The management-cluster pattern** — one small, long-lived cluster that
  reconciles many workload clusters, which is the actual mechanism behind
  "40+ clusters as objects instead of runbooks."

We do **not** re-teach Deployments/ReplicaSets/reconcile loops (02) or
re-derive control-plane HA mechanics (10.3) — CAPI's `KubeadmControlPlane`
literally executes 10.3's runbook for you; we focus on the fleet-scale layer
on top.

## Core concepts

### 1. Cluster API (CAPI): the object model

CAPI is a set of controllers running in a **management cluster** that
reconcile **workload clusters** into existence. Core CRDs (provider-agnostic):

- **`Cluster`** — the top-level object; references an infrastructure provider
  object and a control-plane provider object. "I want a cluster with this
  network config."
- **`KubeadmControlPlane` (KCP)** — declares the control plane: replica count
  (use odd — 3/5, per 10.3), Kubernetes version, and the machine template to
  stamp CP nodes from. KCP owns rolling CP upgrades and etcd management for
  you — this is the controller doing, automatically and safely, exactly the
  version-skew-respecting sequence you ran by hand in 10.3.
- **`MachineDeployment`** — the worker analogue of a Deployment:
  `replicas: N`, a version, and a machine template. Scaling workers = edit
  `replicas`; upgrading = edit `version` and KCP/MD do a rolling replacement.
- **`MachineSet` / `Machine`** — a `Machine` is the atomic "one Kubernetes
  node" object. `MachineDeployment → MachineSet → Machine`, mirroring
  `Deployment → ReplicaSet → Pod`. This parallel is the whole mental model —
  if you can reason about a Deployment rollout, you can reason about a
  MachineDeployment rollout.

Two orthogonal provider slots per cluster:

- **Infrastructure provider** — knows how to *make a machine exist*: AWS
  (CAPA), vSphere (CAPV), **Metal3** (bare metal), **Tinkerbell/CAPT** (bare
  metal), Docker (**CAPD**, for testing). It reconciles an `InfraMachine`
  (e.g. `Metal3Machine`, `DockerMachine`).
- **Bootstrap provider** — knows how to *turn a blank machine into a K8s
  node*: usually **kubeadm** (CABPK), which generates the cloud-init/ignition
  that runs `kubeadm init/join`. Talos has its own bootstrap+control-plane
  providers (CABPT/CACPPT) that speak the Talos machine-config API instead of
  running shell-based `kubeadm` commands.

A `Machine` is provider-neutral; it binds to one `InfraMachine` (where it
runs) and one `BootstrapConfig` (how it becomes a node). That separation is
exactly why you can swap cloud → bare metal without changing the top-level
`Cluster` shape.

### 2. Bare-metal providers: what they do that cloud providers don't

A **cloud** CAPI provider (CAPA/CAPG) turns a `Machine` into a node by
calling an API that already virtualizes everything: "RunInstances" hands
back a booted VM with an OS image, network, and disk in seconds. The hard
parts are abstracted.

A **bare-metal** provider has no such API — it must drive *physical*
hardware:

- **Metal3 + Ironic**: Metal3 models each physical server as a
  **`BareMetalHost` (BMH)** CRD. Its controller drives **Ironic** (from
  OpenStack) to, over **IPMI/Redfish (BMC)**: **enroll and inspect** the
  machine (hardware facts: CPUs, RAM, NICs, disks), **clean/wipe** disks,
  **power** it on/off, and **provision** an OS by writing an image to disk
  (via PXE or virtual media). Only then does the kubeadm/Talos bootstrap run.
  So a bare-metal provider owns: BMC control, PXE/DHCP/TFTP or virtual-media
  boot, disk imaging, hardware inspection, and de-provisioning/cleaning — an
  entire layer the cloud API hid. (Lesson 10.5 walks this pipeline stage by
  stage, in detail.)
- **Tinkerbell / CAPT**: same class of problem (netboot + workflow-driven
  provisioning of physical machines) with a workflow engine (Smee/Tink/Hegel)
  instead of Ironic — a credible, actively-used alternative, not a fringe
  option.

The one-line answer: a bare-metal provider **owns the physical provisioning
lifecycle — BMC/IPMI-Redfish power control, PXE/virtual-media boot, hardware
inspection, and disk imaging/cleaning** — because there is no cloud API to
hand it a ready OS. This is the code NVIDIA/CoreWeave node-lifecycle roles
write and extend.

**Metal3's standing in the ecosystem, precisely.** Metal3 reached **CNCF
Incubating** status on a Technical Oversight Committee vote in **August
2025** — the project's own announcement and CNCF's blog both date the
milestone to **August 27, 2025**, five years after entering the CNCF Sandbox
in September 2020. At incubation, Metal3 reported **57 active contributing
organizations**, led by **Red Hat and Ericsson** (the project's original
co-founders, dating to 2019), with named production adopters including
**Fujitsu, Ikea, SUSE, Ericsson, and Red Hat**. That's a meaningful signal for
a bare-metal infrastructure project specifically — it means the BMH/Ironic
model is a shared, multi-vendor investment, not one company's fork of
OpenStack tooling.

**Don't overstate Metal3 as the only option, though.** Talos's own
Sidero Omni/Talos stack ships CAPI providers that bypass Ironic entirely
(talking directly to Talos's own API for bare-metal bring-up), and Tinkerbell
(CAPT) is a credible, independently-adopted alternative workflow engine. The
space has multiple serious players; Metal3 is the most CNCF-institutional
one, not the only correct one.

### 3. Talos: the immutable-OS alternative

Talos Linux is a Kubernetes-purpose OS with a radically different node
model:

- **No SSH, no shell, no package manager, no systemd, no interactive login.**
  The only interface is a **gRPC API** driven by `talosctl`. There is nothing
  to `ssh` into and hand-edit.
- **Immutable, mostly read-only root filesystem.** The OS ships as a signed
  image; configuration is a single declarative **machine config** (YAML)
  applied via the API. You don't configure a Talos node, you *declare* it.
- **A/B (dual-partition) image upgrades:** `talosctl upgrade` writes the new
  OS image to the inactive partition and reboots into it; failure rolls back
  to the known-good partition. Upgrades are atomic image swaps, not
  in-place package churn.

**Why this kills config drift across 200 nodes:** with a mutable distro
(Ubuntu + Ansible/kubeadm), every node accumulates hand-fixes, half-applied
config-management runs, and package-version skew — 200 nodes become 200
*slightly different* nodes, and drift is where 3am outages live. Talos
removes the mechanisms that create drift: there is **no way to make a
one-off change** (no shell), config is a single declared artifact applied
identically to every node, and the root FS is immutable so nothing mutates
it at runtime. Two nodes on the same Talos version + same machine config are
**bit-for-bit equivalent by construction**; an upgrade is the same atomic
image on all of them. Drift stops being something you police and becomes
something the design forecloses. This is precisely the failure mode Equinix
describes hitting with Ansible/Kubespray before they migrated off it — see
Real-world use cases.

### 4. Management cluster vs workload clusters

The **management cluster** is a (usually small, long-lived) Kubernetes
cluster that runs the CAPI controllers + provider controllers. It holds the
`Cluster`, `KubeadmControlPlane`, `MachineDeployment`, `Machine`,
`BareMetalHost`… objects and **reconciles workload clusters** — the actual
clusters that run your GPU workloads — toward those specs. It continuously
drives: provisioning new machines, replacing failed ones, scaling worker
counts, and rolling version upgrades. `clusterctl` bootstraps a management
cluster and installs providers. (You can even make a management cluster
manage itself by "pivoting" the objects into a freshly created cluster.) One
management cluster can reconcile many workload clusters — this is the "40+
clusters as objects" endgame, and it's the exact pattern Deutsche Telekom's
Das Schiff runs at hundreds of physical locations (see below).

### 5. GitOps as the fourth leg: what happens when the management cluster can't be reached

A management cluster reconciling 40+ remote clusters raises an obvious
question: what happens to a workload cluster if its link back to the
management cluster is down? The production pattern (used by both Das Schiff
and Sylva — see Real-world use cases) is to pair CAPI with a **GitOps
controller (Flux)** running *inside each workload cluster*, not only in the
management cluster. Git becomes the durable source of truth; the management
cluster's CAPI controllers handle machine-level lifecycle (is this node
provisioned, healthy, on the right version), while each workload cluster's
own Flux instance keeps reconciling its application/platform layer straight
from Git even if the management cluster is completely unreachable — a real
requirement at cell-tower/edge sites with unreliable backhaul. This is the
detail that turns "declarative fleet" from a provisioning convenience into an
autonomy guarantee.

## Perspectives

**Developer / platform consumer.** From inside a workload cluster, nothing
looks different from any other Kubernetes cluster — `kubectl`, standard
APIs, standard workloads. The fleet machinery is invisible unless a node
needs replacing, in which case it just... does, without a ticket.

**Operator.** The job shifts from "run commands on nodes" to "edit specs and
watch reconciliation." A CP upgrade is `kubectl edit kubeadmcontrolplane` and
watching `Machine` phases, not SSHing into three boxes in the right order.
The failure mode shifts too: instead of "I forgot a step," it's "the
reconciler is stuck — why won't this `Machine` go `Running`," which demands
you understand the underlying provider (Ironic, Talos) well enough to debug
its controller, not just its CLI.

**Hardware / physical layer.** CAPI and Talos don't remove the physical
provisioning problem — they formalize it. Someone (a bare-metal provider
controller) still has to drive a BMC, wait for a PXE boot, and stream an OS
image to a disk; CAPI just gives that work a Kubernetes-native API surface
and a state machine (`Machine` phases) instead of a shell script. Lesson
10.5 is that physical layer in full depth.

**Economics.** A single management cluster amortizes its operational cost
(patching, HA, on-call) across every workload cluster it reconciles — the
same 3-node CP HA design from 10.3, paid for once, governs 40 fleets instead
of one. That leverage is the actual argument for adopting CAPI at scale: the
marginal cost of cluster #41 approaches "apply a manifest," not "hire another
operator."

## Real-world use cases

- **Deutsche Telekom — "Das Schiff"** <https://github.com/telekom/das-schiff>
  — one of the largest known bare-metal Cluster API deployments in
  existence: CAPI + **Metal3** (+ vSphere for VM-based sites) + **Flux**
  managing Kubernetes clusters at hundreds of locations across Germany,
  including remote edge/cell-tower sites that must stay operable even when
  the management cluster is unreachable — a real, production GitOps-autonomy
  design that directly demonstrates this lesson's "declarative fleet" thesis
  at genuinely large scale.
- **Sidero Labs — "Equinix switches from KubeSpray to Talos Linux"**
  <https://www.siderolabs.com/case-studies/equinix-switches-from-kubespray-to-talos-linux-cutting-deployment-time-while-maintaining-security>
  — Equinix's managed-Kubernetes team moved off Kubespray-plus-Flatcar
  (brittle Ansible, slow upgrades, security-compliance friction) onto Talos,
  and cut VM cluster deployment time from **roughly 45 minutes to under 10
  minutes** (2026 snapshot of a migration Equinix began evaluating around
  2019 and completed by end-of-lifeing Kubespray). A concrete, numbers-backed
  before/after for §3's "Talos kills drift" claim — read it for the
  operational-burden story (convoluted Ansible, slow SRE-tying-up upgrades),
  not just the headline number.
- **Sylva Project (Linux Foundation Europe)** —
  <https://sylvaproject.org/ocudu-on-telco-clouds-0-introduction-to-sylva/>
  and background <https://the-mobile-network.com/2022/11/why-the-eu-big-five-are-launching-sylva/>
  — Vodafone, Deutsche Telekom, Orange, Telecom Italia, and Telefónica
  jointly launched Sylva (November 2022) as a shared open-source telco cloud
  stack explicitly **"built around Cluster API, Metal3, and Flux."** This is
  multi-carrier corroboration, independent of Das Schiff, that CAPI+Metal3
  (+GitOps) is the accepted industry pattern for telco-scale bare-metal
  fleets — five major European carriers converging on the same architecture
  this lesson teaches.

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

You'll see the `Machine` objects march through phases as the controller
creates the backing Docker "nodes," then the kubeadm bootstrap provider runs
`kubeadm init/join` inside them — the exact `kubeadm init`/`kubeadm join
--control-plane` sequence you ran by hand in 10.1/10.3, now driven by a
controller instead of your fingers. Then **scale and observe
reconciliation**:

```bash
kubectl scale machinedeployment demo-md-0 --replicas=3   # desired state changes
kubectl get machines -w                                   # 2 new Machines appear
```

Fetch the workload cluster's kubeconfig and confirm it's a real cluster:

```bash
clusterctl get kubeconfig demo > demo.kubeconfig
kubectl --kubeconfig demo.kubeconfig get nodes
```

The key observation for your target job: **the `Machine` is the reconciled
unit.** Swap `--infrastructure docker` for `--infrastructure metal3`, and
the *same* `Machine`/`MachineDeployment` objects now drive Ironic to
PXE-boot physical servers. The control plane you built by hand in 10.1–10.3
is what KCP + the kubeadm bootstrap provider now produce automatically —
including the VIP wiring and the version-skew-safe upgrade sequence.

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
Observe: there is **no SSH**; the only access is the API. Capture the
machine config and the upgrade output as evidence of the immutable/A-B
model.

**Path B — CAPI + Docker (CAPD), to watch the reconcile loop:**
Run the worked example above. Capture `clusterctl describe cluster demo`,
`kubectl get machines` transitioning through phases, and a `scale` that
spawns new `Machine` objects.

**Acceptance (feeds the capex-vs-cloud writeup):** a **Talos-provisioned OR
CAPI-provisioned cluster**, plus a short note documenting the **reconcile
behavior you observed** — for CAPD: the `Machine` phase transitions and a
`replicas` change producing/removing `Machine`s; for Talos: the single
declarative machine config + the A/B upgrade. Add 3–5 sentences mapping the
objects (`Cluster`/`KubeadmControlPlane`/`MachineDeployment`/`Machine`, or
Talos machine config) to the hand-built control plane from 10.1–10.3, and
one line on where a **bare-metal provider (Metal3/Ironic)** would slot in
for real hardware. This is the "how the fleet scales past hand-building"
section of the deliverable.

## Common pitfalls

- **Treating Metal3 as the only credible bare-metal provider.** Tinkerbell
  (CAPT) and Talos's own Sidero Omni stack are real, independently-adopted
  alternatives. Pick based on your team's operational preference (declarative
  host state vs. workflow-as-data, or full-Talos-native vs. kubeadm-based),
  not because Metal3 is the only name you've heard.
- **Confusing the management cluster with a workload cluster.** The
  management cluster typically runs no GPU workloads and exists purely to
  host CAPI controllers; conflating the two (e.g. running production traffic
  on it) couples the fleet's control plane to the fleet's data plane in a
  way that defeats the isolation CAPI is designed to give you.
- **Assuming CAPI removes the physical-provisioning problem.** It doesn't —
  it formalizes it into a controller (Metal3/Ironic, Tinkerbell) that still
  has to drive real BMCs and PXE boots. CAPI is the orchestration layer, not
  a replacement for lesson 10.5's netboot pipeline.
- **Baking application state or ad hoc config into "immutable" Talos
  nodes.** The whole value proposition (drift-free, bit-for-bit-identical
  nodes) evaporates the moment someone finds a workaround to persist
  one-off state — Talos removes the shell specifically to make that
  temptation impossible, don't fight the design.
- **Ignoring the GitOps half of the story at fleet scale.** CAPI alone
  reconciles machine-level state; without a GitOps controller (Flux) inside
  each workload cluster reconciling application/platform state, a network
  partition between an edge site and the management cluster leaves that
  site's platform layer stuck — Das Schiff and Sylva both pair CAPI with
  Flux for exactly this reason.

## Self-check

- **What does a bare-metal CAPI provider (Metal3/Ironic) do that a cloud
  CAPI provider doesn't?**
  **Answer:** It owns the **physical** provisioning lifecycle that a cloud
  API hides. A cloud provider just calls an API ("RunInstances") that returns
  a booted, imaged VM. Metal3 models each server as a `BareMetalHost` and
  drives **Ironic** over **IPMI/Redfish (BMC)** to power the machine on/off,
  **inspect** its hardware, **clean/wipe** disks, and **provision an OS image
  via PXE/virtual media** — none of which exists as an API on bare metal.
  Only after that does bootstrap (kubeadm/Talos) run.
- **Why does an immutable OS (Talos) reduce config drift across 200 nodes?**
  **Answer:** It removes the mechanisms that create drift. There's **no
  shell/SSH/package manager**, so no one can make a one-off change; the root
  filesystem is **immutable**; and the entire node is defined by a **single
  declarative machine config** applied identically everywhere. Upgrades are
  **atomic A/B image swaps**, not in-place package churn. Two nodes on the
  same version + config are equivalent by construction, so drift is
  foreclosed by design rather than policed after the fact — the exact
  problem Equinix cites hitting with Kubespray/Ansible before migrating.
- **What is a management cluster and what does it reconcile?**
  **Answer:** A management cluster is a Kubernetes cluster running the
  Cluster API core controllers plus infrastructure/bootstrap provider
  controllers. It holds the CAPI objects (`Cluster`, `KubeadmControlPlane`,
  `MachineDeployment`, `Machine`, `BareMetalHost`, …) and **reconciles the
  workload clusters** those objects describe — creating/replacing/scaling
  their machines and rolling their version upgrades toward the declared
  spec. One management cluster can reconcile many workload clusters, which
  is how a fleet of 40+ clusters becomes a set of objects instead of
  runbooks — the pattern Deutsche Telekom's Das Schiff runs at hundreds of
  physical locations.
- **When did Metal3 reach CNCF Incubating status, and why does that matter
  beyond "it's a mature project"?**
  **Answer:** The CNCF TOC voted Metal3 to Incubating in **August 2025**
  (announced August 27, 2025), five years after entering the Sandbox in
  2020, with **57 active contributing organizations** led by **Red Hat and
  Ericsson** and named production adopters including **Fujitsu, Ikea, SUSE,
  Ericsson, and Red Hat**. It matters because it's evidence of a genuinely
  multi-vendor, multi-organization investment in the `BareMetalHost`/Ironic
  model specifically for bare-metal Kubernetes — a signal you can cite when
  arguing for it over an in-house netboot script, though not a reason to
  treat it as the only valid choice.
- **Your fleet has an edge site whose network link to the management
  cluster is unreliable. What pattern keeps that site's platform layer
  reconciling anyway, and who uses it in production?**
  **Answer:** Pair CAPI (in the management cluster, for machine-level
  lifecycle) with a **GitOps controller (Flux) running inside the workload
  cluster itself**, so the edge site keeps reconciling its
  application/platform state straight from Git even when it can't reach the
  management cluster. Deutsche Telekom's Das Schiff and the multi-carrier
  Sylva Project both run exactly this CAPI+Metal3+Flux combination for
  hundreds of distributed, sometimes-unreachable sites.

## Connections & what's next

This lesson is the synthesis point for 10.1–10.3: the `KubeadmControlPlane`
object automates 10.3's HA/upgrade runbook, and a bare-metal `Machine`
reconciling through Metal3/Ironic automates 10.1's by-hand node bring-up. It
also previews **10.5 (node provisioning: PXE → image → firmware)**, which
is the detailed physical layer that a bare-metal infrastructure provider
(Metal3/Ironic or Tinkerbell) actually drives — this lesson told you *that*
Metal3 owns BMC/PXE/imaging; 10.5 shows you *how*, stage by stage. It
connects sideways to **10.6 (hardware health, remediation, RMA)**: a
`Machine` object is also the natural place to hook automated remediation —
cordon/drain/re-provision as a controller action instead of a human running
commands. And it sharpens the module's capstone, **10.8 (capex-vs-cloud)**:
a declarative fleet is what makes a large capex bet on bare metal
operationally tractable in the first place — the argument "owning beats
renting past N GPUs" only holds if you can actually operate N machines
without linearly scaling headcount, which is exactly what this lesson's
reconcile model buys you.

Next: **10.5 — Node provisioning: PXE → image → firmware** goes one layer
below this lesson, into the netboot-to-Ready pipeline that a bare-metal CAPI
provider actually executes on physical hardware — the stage where firmware,
BIOS, and disk imaging meet the reconcile loop you just watched run in CAPD.

## References & further reading

**Primary sources**
- **The Cluster API Book** — <https://cluster-api.sigs.k8s.io/> — read for the full CRD reference and the quick-start/CAPD flow this lesson's worked example runs.
- **Talos Linux documentation** — <https://www.talos.dev/latest/> — read for the machine-config schema, `talosctl` reference, and the A/B upgrade mechanism in detail.
- **Metal3.io** — <https://metal3.io/> — read for the `BareMetalHost` CRD model and how the baremetal-operator drives Ironic.
- **Metal3.io becomes a CNCF Incubating project (CNCF blog, Aug 27, 2025)** — <https://www.cncf.io/blog/2025/08/27/metal3-io-becomes-a-cncf-incubating-project/> — read for the exact incubation date, contributor count, and named adopters cited in this lesson.

**Real-world engineering blogs**
- **Deutsche Telekom — Das Schiff** — <https://github.com/telekom/das-schiff> — what it shows: CAPI + Metal3 + Flux managing Kubernetes at hundreds of German locations, including edge sites that stay operable when the management cluster is unreachable.
- **Sidero Labs — Equinix switches from KubeSpray to Talos Linux** — <https://www.siderolabs.com/case-studies/equinix-switches-from-kubespray-to-talos-linux-cutting-deployment-time-while-maintaining-security> — what it shows: a numbers-backed migration off a mutable Ansible-driven OS onto Talos, cutting cluster deploy time from ~45 minutes to under 10.
- **Sylva Project — Introduction to Sylva** — <https://sylvaproject.org/ocudu-on-telco-clouds-0-introduction-to-sylva/> and **The Mobile Network — Why the EU big five are launching Sylva** — <https://the-mobile-network.com/2022/11/why-the-eu-big-five-are-launching-sylva/> — what they show: five major European carriers jointly building a telco cloud stack explicitly "built around Cluster API, Metal3, and Flux," independent corroboration of this lesson's stack.

**Deeper dives**
- **Tinkerbell documentation** — <https://tinkerbell.org/> — the workflow-engine alternative to Ironic (Smee/Tink/Hegel/Rufio + HookOS) for bare-metal provisioning; useful for seeing the "imperative pipeline as data" design contrasted with Metal3's declarative host state.
- **Talos Image Factory** — <https://factory.talos.dev/> — schematic-based, extension-included (e.g. NVIDIA) Talos images for a fully immutable, hardware-free netboot demo; the practical on-ramp for Lesson 10.5's hands-on path.
