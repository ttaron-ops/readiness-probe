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
sources: 12
---
# 10.4 · Declarative fleets: Cluster API + Metal3 + Talos

> **Concept.** Hand-building one control plane teaches you the parts; running 40+ clusters demands they be reconciled objects. Cluster API turns "a cluster" into a Kubernetes resource, and immutable-OS distributions like Talos turn "a node" into a declarative artifact — so a fleet is `kubectl apply`, not a runbook.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 10.3 gave you a 3-node HA control plane behind a VIP and a hand-run sequence for upgrading
it without violating version skew: snapshot etcd, `upgrade apply` on the first node, `upgrade node`
on the rest, then kubelets one at a time. That runbook works for one cluster. It does not work for
forty. This lesson takes every manual step from 10.1–10.3 — mint the PKI, bootstrap a control
plane, add an etcd member, hold a VIP, sequence a skew-safe upgrade — and shows how the industry
turns each of them into a **reconciled Kubernetes object**, so that "build a cluster" becomes
"apply a manifest" and "upgrade the fleet" becomes "edit a version field." This is where the
module's individual, hand-cranked lessons compose into something that scales to a real neocloud's
fleet size.

## Why this matters

You cannot SSH-and-runbook your way through 40 clusters, and you cannot re-derive an upgrade by
hand on each one without eventually getting the order wrong on the one that mattered. The industry
answer is **declarative fleet provisioning**: describe the desired cluster as API objects and let a
controller reconcile reality toward them — the same reconcile/CRD model from **K8s-controllers
02**, now pointed at whole clusters and physical machines instead of Pods.

This is also the literal job target. **NVIDIA's "Kubernetes Node Lifecycle" and CoreWeave-style
neocloud roles build Cluster API providers in Go** — the machinery that turns a `Machine` object
into a provisioned GPU node with the right firmware *is itself a controller you write*.
Understanding CAPI's object model, its contract, and where a bare-metal provider plugs in is the
on-ramp to that work. The immutable-OS angle (Talos) is how those fleets stay drift-free at 200+
nodes. And your differentiator — cost and observability — lives here too: a declarative fleet gives
you a single queryable inventory of every cluster and machine to bill, right-size, and audit.

It is not a niche pattern either. Deutsche Telekom, Vodafone, Orange, Telefónica and Telecom
Italia have all independently converged on this exact stack for telco-scale bare-metal fleets (see
Real-world use cases). This is the industry-standard answer, not one vendor's opinion.

## What's new here (calibration)

You already own: the reconcile-loop and CRD mental model (**K8s-controllers 02**), the PKI and
static-pod bootstrap (**10.1**), etcd operations and member management (**10.2**), and
control-plane HA plus the version-skew upgrade order (**10.3**). What is genuinely new:

- **The CAPI object model as a contract**, not just a list of CRDs — `Cluster`,
  `KubeadmControlPlane`, `MachineDeployment`, `MachineSet`, `Machine`, and the two orthogonal
  provider slots (infrastructure, bootstrap) that let identical top-level objects target a cloud
  API or a physical server.
- **How `KubeadmControlPlane` actually rolls a control plane** — its preflight checks, its
  scale-up-then-scale-down strategy, and the fact that it forwards etcd leadership off a machine
  before removing its member. This is 10.3's runbook, encoded, and reading it is the fastest way
  to check your own understanding of 10.3.
- **What a bare-metal infrastructure provider does that a cloud provider does not** — the
  `BareMetalHost` state machine, Ironic, BMC protocols, hardware inspection, disk cleaning — an
  entire layer the cloud API hides.
- **Talos's immutable-node model** — no shell, no package manager, a declarative multi-document
  machine config applied over gRPC, a fixed partition layout, and A/B boot entries — a genuinely
  different answer to "what is a node" than Ubuntu-plus-config-management.
- **The management-cluster pattern**, including pivoting and the GitOps autonomy leg that makes it
  survivable when the management cluster is unreachable.
- **Fleet arithmetic** — what reconciliation buys you, in provisioning hours and in
  operators-per-cluster.

We do **not** re-teach Deployments/ReplicaSets (02) or re-derive control-plane HA mechanics
(10.3); `KubeadmControlPlane` literally executes 10.3's runbook for you, and this lesson's job is
the fleet-scale layer on top.

**Version note.** CAPI object shapes and controller behaviour below were read from
`kubernetes-sigs/cluster-api` (main, August 2026 — release series **v1.14**, contract
**v1beta2**), Metal3 from `metal3-io/baremetal-operator` and
`metal3-io/cluster-api-provider-metal3` (main), and Talos from `siderolabs/talos`
(`release-1.14`) plus `siderolabs/cluster-api-control-plane-provider-talos` (main). The manifests
are adapted from those repositories' own examples and e2e templates. cluster-api.sigs.k8s.io,
talos.dev, metal3.io and kubernetes.io are unreachable from this environment's egress proxy, so
the rendered documentation sites were **not** used — see
[References](#references--further-reading).

## Core concepts

### The problem reconciliation solves

Start from what you actually built in 10.1–10.3, and count the imperative steps: generate 3 CAs
and 8 leaves; write 5 kubeconfigs; drop 4 static-pod manifests; start the kubelet; install a CNI;
create a bootstrap token; join two more control-plane nodes in the right order while checking etcd
health between each; stand up a VIP; join workers; and then, for an upgrade, do a strictly ordered
dance across every node with a verified etcd snapshot as your only rollback.

That is roughly forty ordered steps with real preconditions between them. Run it once and you
learn something. Run it forty times and you will:

- get the order wrong on at least one cluster, at 2am, under pressure;
- accumulate divergence, because cluster #7 was built in March with a slightly different CNI
  version and nobody wrote it down;
- have no inventory — no single query that answers "which clusters are on 1.34?";
- and have no way to *un-break* a half-finished operation, because a shell script that died
  halfway leaves no record of where it was.

Reconciliation replaces "run these steps" with "declare this state, and a controller will keep
converging toward it." The properties you get are not conveniences; they are different in kind:

| Imperative runbook | Reconciled objects |
|---|---|
| succeeds or leaves unknown partial state | converges from *any* state, including partial |
| order lives in a human's head or a script | order lives in controller preconditions that are checked every loop |
| no inventory | `kubectl get clusters -A` |
| retrying may be unsafe | retrying is the normal operating mode |
| drift is discovered by an outage | drift is a `UpToDate=False` condition |
| rollback = restore a snapshot | rollback = revert the manifest |

Everything else in this lesson is that idea applied at two levels: CAPI reconciles *clusters and
machines*; Talos reconciles *the inside of a node*.

### The Cluster API object graph

CAPI is a set of controllers running in a **management cluster** that reconcile **workload
clusters** into existence. Here is the whole object graph, with ownership (who creates whom) and
reconciliation arrows (who watches and acts on what):

```
 ══════════════ MANAGEMENT CLUSTER ══════════════════════════════════════════════
 (a small, long-lived cluster — often itself a 3-node HA control plane from 10.3)

   CONTROLLERS                         OBJECTS THEY OWN
   ───────────                         ────────────────
   cluster-api-controller  ──────────▶ ┌──────────────────────────────────┐
   (core)                              │ Cluster  "gpu-prod-01"           │
                                       │  spec.controlPlaneRef ───────────┼──┐
                                       │  spec.infrastructureRef ─────────┼─┐│
                                       │  status.controlPlaneReady        │ ││
                                       │  status.infrastructureReady      │ ││
                                       └──────────────────────────────────┘ ││
                                                                            ││
   infrastructure provider ──────────▶ ┌────────────────────────────────┐ ◀─┘│
   (CAPM3 / CAPA / CAPD)               │ Metal3Cluster                  │    │
   "make the cluster-level             │  spec.controlPlaneEndpoint     │    │
    infra exist: VIP, LB,              │  status.ready                  │    │
    network"                           └────────────────────────────────┘    │
                                                                             │
   kubeadm control-plane   ──────────▶ ┌──────────────────────────────────┐ ◀┘
   provider (KCP)                      │ KubeadmControlPlane              │
   "own the control plane:             │  spec.replicas: 3                │
    scale, roll, upgrade,              │  spec.version: v1.36.1           │
    manage etcd members"               │  spec.machineTemplate.spec.      │
                                       │        infrastructureRef ────────┼──┐
                                       │  spec.kubeadmConfigSpec {…}      │  │
                                       └────────────┬─────────────────────┘  │
                                            creates │ (owner ref)            │
                                                    ▼                        │
                                       ┌──────────────────────────────────┐  │
   machine controller ───────────────▶ │ Machine × 3  (control plane)     │  │
   "the atomic unit:                   │  spec.version, spec.clusterName  │  │
    one Kubernetes node"               │  spec.bootstrap.configRef ───────┼┐ │
                                       │  spec.infrastructureRef ─────────┼┼─┤
                                       │  status.phase, status.nodeRef    ││ │
                                       └──────────────────────────────────┘│ │
                                                                           │ │
   bootstrap provider (CABPK) ───────▶ ┌───────────────────────────────┐ ◀─┘ │
   "turn a blank machine into          │ KubeadmConfig                 │     │
    a k8s node: emit cloud-init"       │  status.dataSecretName ───────┼─┐   │
                                       └───────────────────────────────┘ │   │
                                                                         │   │
   infrastructure provider ──────────▶ ┌───────────────────────────────┐ │ ◀─┘
   "make the MACHINE exist"            │ Metal3Machine  ◀── template ──┼─┼── Metal3MachineTemplate
                                       │  status.ready, spec.providerID│ │
                                       └───────────────┬───────────────┘ │
                                                       │ claims          │
                                                       ▼                 │
   baremetal-operator ───────────────▶ ┌───────────────────────────────┐ │
   "drive the physical host"           │ BareMetalHost "rack1-slot7"   │ │
                                       │  spec.bmc.address (Redfish)   │ │
                                       │  spec.image.url               │ │
                                       │  status.provisioning.state    │ │
                                       │  status.hardware {cpu,ram,nic}│ │
                                       └───────────────┬───────────────┘ │
                                                       │ Ironic          │
   ═══════════════════════════════════════════════════ │ ═══════════════ │ ═════
                                            IPMI/Redfish│  + PXE/vmedia   │
                                                        ▼                 │
 ══════════════ PHYSICAL RACK ════════════   ┌────────────────────┐       │
                                             │  a real server     │       │
                                             │  BMC + NIC + NVMe  │       │
                                             └─────────┬──────────┘       │
                                                       │ boots the image, │
                                                       │ runs cloud-init  │
                                                       │ from the Secret ─┘
                                                       ▼
 ══════════════ WORKLOAD CLUSTER "gpu-prod-01" ═════════════════════════════
   its own apiserver / etcd / scheduler / CM  (everything you built in 10.1–10.3)
   Node "rack1-slot7"  ◀── Machine.status.nodeRef points here
```

Read three things off that graph.

**The `Machine` is the pivot.** Everything above it is provider-neutral; everything below it is
provider-specific. A `Machine` binds exactly two references: an **InfraMachine** (where it runs)
and a **BootstrapConfig** (how it becomes a node). Swap `Metal3Machine` for `DockerMachine` or
`AWSMachine` and the `Cluster`, `KubeadmControlPlane` and `MachineDeployment` above it are
unchanged. **That is the entire value proposition of CAPI**, and it is why "build a CAPI provider"
is a well-defined job: you implement the InfraMachine/InfraCluster half of a documented contract.

**The management cluster is where every controller lives.** No controller runs inside the workload
cluster. The management cluster reaches into the workload cluster with a kubeconfig it generated
and stored as a Secret (`<cluster>-kubeconfig`), which is how KCP can query etcd member health on
a cluster it built.

**The arrows are watches, not calls.** Nothing in this diagram invokes anything else. Every
controller watches its own objects plus the ones it owns, and writes status back. That is why a
half-finished provisioning run resumes correctly after the management cluster is restarted.

### The v1beta2 contract, and why references look odd

CAPI's current contract is **v1beta2** (from release series v1.11 onward; v1.10 and earlier used
v1beta1). One change is visible in every manifest and confuses people who learned the old shape:

```yaml
  infrastructureRef:
    apiGroup: infrastructure.cluster.x-k8s.io   # note: apiGroup, NOT apiVersion
    kind: Metal3Cluster
    name: gpu-prod-01
```

That is a `ContractVersionedObjectReference`, and the missing version is deliberate. From the type
definition: *"The corresponding version for this reference will be looked up from the contract
labels of the corresponding CRD of the resource being referenced."* In other words, CAPI reads a
label on the provider's CRD to learn which version of that CRD implements which contract, and
resolves the reference at runtime. The practical effect: **a provider can move from `v1beta1` to
`v1beta2` of its own API without you editing forty `Cluster` manifests.** It also means a
reference that used to be namespaced no longer is — referenced objects must live in the same
namespace as the referrer.

### The Machine lifecycle

`Machine.status.phase` is the coarse view of an object with a lot of underlying conditions. The
phases, from `api/core/v1beta2/machine_phase_types.go`:

```
                    ┌──────────────────────────────────────────────────────┐
   Machine created  │                                                      │
   by KCP or a      ▼                                                      │
   MachineSet   ┌─────────┐                                                │
                │ Pending │  "first state assigned by the Machine          │
                └────┬────┘   controller after creation"                   │
                     │ bootstrap provider has produced a data Secret       │
                     │ AND the InfraMachine has been created               │
                     ▼                                                     │
             ┌──────────────┐                                              │
             │ Provisioning │  "the Machine infrastructure is being        │
             └──────┬───────┘   created" — for Metal3 this is the whole    │
                    │           BareMetalHost state machine below:         │
                    │           inspect → clean → write image → power on   │
                    │  InfraMachine.status.ready = true, providerID set    │
                    ▼                                                      │
             ┌──────────────┐                                              │
             │ Provisioned  │  "infrastructure created and configured"     │
             └──────┬───────┘   the machine is booting; kubeadm/Talos is   │
                    │           running inside it                          │
                    │  a Node object appears in the WORKLOAD cluster whose │
                    │  providerID matches → status.nodeRef is set          │
                    │  → and that Node reports Ready                       │
                    ▼                                                      │
             ┌──────────────┐                                              │
             │   Running    │  "has become a Kubernetes Node in a Ready    │
             └──┬────────┬──┘   state"                                     │
                │        │                                                 │
     in-place   │        │ spec changed (e.g. version) and the owner       │
     update     │        │ decided to REPLACE rather than update           │
                ▼        ▼                                                 │
        ┌──────────┐   ┌──────────┐  deletionTimestamp set. Machine        │
        │ Updating │   │ Deleting │  controller: cordon → drain →          │
        └────┬─────┘   └────┬─────┘  detach volumes → (if control plane)   │
             │              │        forward etcd leadership + remove the  │
             └──────────────┤        etcd member → delete the InfraMachine │
                            ▼                                              │
                     ┌──────────┐                                          │
                     │ Deleted  │ ─────────────────────────────────────────┘
                     └──────────┘   (a replacement Machine is created by
                                     KCP / MachineSet to restore replicas)

   Unknown  — the phase could not be determined.
   Failed   — DEPRECATED in v1beta2: controllers no longer set it. It exists
              only for conversion from v1beta1 objects. If your runbook says
              "look for Machines in Failed", it is pre-v1beta2 — look at the
              conditions instead.
```

The header comment on `MachinePhase` is unusually blunt and worth quoting in spirit: the phase
"should not be interpreted by any software components as a reliable indication of the actual state
of the Machine," and controllers "should always look at the actual state of the Machine's fields."
Phase is a human summary. **Automation should read conditions** — `Ready`, `Available`,
`UpToDate`, `Paused`, plus the provider's own — which is also what `kubectl get machines` prints:

```console
$ kubectl get machines
NAME                          CLUSTER       NODE NAME     FAILURE DOMAIN  READY  AVAILABLE  UP-TO-DATE  PHASE         AGE   VERSION
gpu-prod-01-control-plane-4z  gpu-prod-01   rack1-slot7   rack1           True   True       True        Running       19d   v1.36.1
gpu-prod-01-control-plane-7k  gpu-prod-01   rack2-slot3   rack2           True   True       True        Running       19d   v1.36.1
gpu-prod-01-control-plane-mq  gpu-prod-01   rack3-slot1   rack3           True   True       False       Running       19d   v1.35.4
gpu-prod-01-md-0-9f4tx-b2n    gpu-prod-01   rack1-slot9   rack1           True   True       True        Running       12d   v1.35.4
gpu-prod-01-md-0-9f4tx-x7q    gpu-prod-01                 rack2           False  False      True        Provisioning  3m    v1.35.4
```

Read that output like an operator. Row 3 has `UP-TO-DATE False` at `v1.35.4` while its peers are
`v1.36.1`: a control-plane rollout is in progress and this is the machine still to be replaced.
Row 5 is `Provisioning` with no `NODE NAME`: the physical host is being imaged and has not yet
joined. And `FAILURE DOMAIN` shows the placement — KCP spreads control-plane machines across
failure domains automatically (`NextFailureDomainForScaleUp`), which is 10.3's rack-spreading
requirement encoded as a scheduling rule rather than a wiki page.

### `KubeadmControlPlane`: 10.3's runbook, encoded

This is the section worth reading closely, because it is the payoff for having done 10.3 by hand.
KCP's controller does exactly what you did, with the preconditions checked on every reconcile
instead of remembered.

**Preflight, before any scale-up or scale-down** (`preflight.go`). KCP refuses to act unless:

- there are **no Machines currently being deleted** — one operation at a time, always;
- for a scale-up, the `CertificatesAvailable` condition is true — it cannot join a machine without
  the CA keys;
- every existing control-plane Machine has a `nodeRef` (i.e. it correlated to a Node);
- and every one of these conditions is true on every machine:
  `APIServerPodHealthy`, `ControllerManagerPodHealthy`, `SchedulerPodHealthy`, and — when etcd is
  KCP-managed — `EtcdPodHealthy` and `EtcdMemberHealthy`.

The source comment states the design intent explicitly: preflight checks "play an important role
in ensuring that KCP performs *one operation at a time*, by forcing the system to wait for the
previous operation to complete and the control plane to become stable before starting the next."
It also refuses to proceed when **"there are etcd members still in learner mode"** — which is
10.2's learner mechanism showing up as a fleet-level safety property.

**Rolling an upgrade: surge, then shrink.** With
`rollout.strategy.type: RollingUpdate` and `maxSurge: 1`, KCP goes 3 → 4 → 3, never 3 → 2 → 3:

```
  spec.version changed from v1.35.4 to v1.36.1, replicas: 3, maxSurge: 1

  step 0   [m1 v1.35.4] [m2 v1.35.4] [m3 v1.35.4]        etcd 3 members, quorum 2
             │
             │ preflight: all healthy? no deletions in flight? no learners?  ✓
             ▼
  step 1   [m1] [m2] [m3] [m4 v1.36.1 Provisioning]      SCALE UP FIRST
                                                          etcd: m4 joins as a member
                                                          → 4 members, quorum 3
             │  wait for m4 Running AND its etcd member healthy
             ▼
  step 2   [m1] [m2] [m3] [m4 ✓]                          preflight again
             │
             │  select the machine to remove (oldest / outdated first)
             │  IF that machine is the etcd LEADER:
             │      ForwardEtcdLeadership(from: m1, to: best candidate)
             │      — candidates sorted: etcd-healthy first, then
             │        not-flagged-by-MachineHealthCheck, …
             │  THEN remove m1's etcd member from the cluster
             │  THEN cordon + drain the node, then delete the Machine
             ▼
  step 3   [m2] [m3] [m4]                                 back to 3 members
             │
             └── repeat for m2 and m3 → [m4] [m5] [m6], all v1.36.1

  WHY SURGE-FIRST: going 3 → 2 first would leave the cluster at quorum 2 of 2 —
  zero fault tolerance — for the whole duration of a machine provisioning, which
  on bare metal is ~15 minutes (10.1's arithmetic). Surging to 4 first means the
  window of reduced tolerance is only as long as the member removal itself.
```

Compare that to what you typed in 10.3 and the mapping is one-to-one: check etcd health before
joining (`check-etcd`), add the member (`etcd-join`), verify, move leadership off the node you are
about to lose, remove the member, drain, remove the node. **The controller is not doing something
cleverer than you did; it is doing exactly what you did, every 10 seconds, without forgetting.**

**Where it differs from `kubeadm upgrade`, and why.** kubeadm upgrades a control-plane node *in
place*: same machine, new binaries. KCP by default **replaces** the machine. On bare metal that
means re-imaging a physical server, which is slower but gives you the immutability property — the
new machine is built from the same template as every other machine, so drift cannot accumulate.
(CAPI has been growing in-place update support — `Machine` gained an `Updating` phase and KCP has
`inplace*.go` reconcilers — but replacement remains the default and the model you should reason
about.)

### A complete, annotated bare-metal cluster manifest

Here is a full CAPI + Metal3 workload cluster, adapted from `cluster-api-provider-metal3`'s own
`examples/` templates. Every field that carries meaning is annotated; this is the artifact that
replaces roughly forty runbook steps.

```yaml
# ── 1. The cluster itself ─────────────────────────────────────────────────────
apiVersion: cluster.x-k8s.io/v1beta2
kind: Cluster
metadata:
  name: gpu-prod-01
  namespace: fleet
spec:
  clusterNetwork:
    services: { cidrBlocks: ["10.96.0.0/12"] }    # first IP (10.96.0.1) lands in the
    pods:     { cidrBlocks: ["192.168.0.0/18"] }  # apiserver SANs — 10.1's rule, automated
    serviceDomain: cluster.local
  infrastructureRef:                     # ContractVersionedObjectReference: apiGroup, not apiVersion
    apiGroup: infrastructure.cluster.x-k8s.io
    kind: Metal3Cluster
    name: gpu-prod-01
  controlPlaneRef:
    apiGroup: controlplane.cluster.x-k8s.io
    kind: KubeadmControlPlane
    name: gpu-prod-01-controlplane
---
# ── 2. Cluster-level infrastructure: where the API endpoint lives ─────────────
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: Metal3Cluster
metadata:
  name: gpu-prod-01
  namespace: fleet
spec:
  controlPlaneEndpoint:                  # this is 10.3's VIP, declared instead of configured.
    host: 10.10.0.100                    # it also becomes a SAN on every apiserver cert.
    port: 6443
  cloudProviderEnabled: false            # bare metal: no cloud-controller-manager
---
# ── 3. The control plane: 10.3's HA design as one object ──────────────────────
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
metadata:
  name: gpu-prod-01-controlplane
  namespace: fleet
spec:
  replicas: 3                            # ODD. 10.2's quorum arithmetic, enforced by a webhook.
  version: v1.36.1                       # change this ONE field to upgrade the control plane
  rollout:
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 1                      # surge to 4 before dropping to 3 — never below quorum
  machineTemplate:
    spec:
      infrastructureRef:                 # the stamp every CP Machine is cut from
        apiGroup: infrastructure.cluster.x-k8s.io
        kind: Metal3MachineTemplate
        name: gpu-prod-01-controlplane
      deletion:
        nodeDrainTimeoutSeconds: 600     # 0 = wait forever. On a GPU fleet, see 10.6:
                                         # a 10-min cap can kill a long checkpoint window.
  kubeadmConfigSpec:                     # everything below is literally kubeadm config (10.1)
    initConfiguration:
      nodeRegistration:
        name: '{{ ds.meta_data.local_hostname }}'   # cloud-init template: the node names
        kubeletExtraArgs:                            # itself from instance metadata
          - { name: cloud-provider, value: baremetal }
    clusterConfiguration:
      apiServer:
        extraArgs:
          - { name: cloud-provider, value: baremetal }
        certSANs: ["10.10.0.100", "api.gpu-prod-01.internal"]   # 10.1's SAN list, declared
      controllerManager:
        extraArgs:
          - { name: cloud-provider, value: baremetal }
    joinConfiguration:
      controlPlane: {}                   # non-empty marker: this is a CONTROL-PLANE join,
                                         # i.e. `kubeadm join --control-plane` (10.3, step 2)
      nodeRegistration:
        name: '{{ ds.meta_data.local_hostname }}'
        kubeletExtraArgs:
          - { name: cloud-provider, value: baremetal }
---
# ── 4. The machine stamp for control-plane nodes ──────────────────────────────
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: Metal3MachineTemplate
metadata:
  name: gpu-prod-01-controlplane
  namespace: fleet
spec:
  nodeReuse: false                       # true = prefer the same physical host on replacement,
                                         # which speeds rollouts but weakens the immutability
                                         # story. false = take whatever is Available.
  template:
    spec:
      automatedCleaningMode: metadata    # "metadata" = wipe partition tables only (fast).
                                         # "disabled" = no wipe at all — do NOT use when hosts
                                         # move between tenants; a full secure-erase belongs
                                         # in the BareMetalHost's cleaning config.
      image:
        url: http://192.168.0.1/images/ubuntu-24.04-k8s-1.36.qcow2
        checksum: http://192.168.0.1/images/ubuntu-24.04-k8s-1.36.qcow2.sha256sum
        checksumType: sha256             # Ironic VERIFIES this before writing to disk.
        format: qcow2                    # a mismatch aborts provisioning rather than
                                         # producing a subtly broken node.
      dataTemplate:
        name: gpu-prod-01-cp-metadata    # supplies per-machine metadata/networkdata
---
# ── 5. Workers ────────────────────────────────────────────────────────────────
apiVersion: cluster.x-k8s.io/v1beta2
kind: MachineDeployment
metadata:
  name: gpu-prod-01-md-0
  namespace: fleet
  labels: { cluster.x-k8s.io/cluster-name: gpu-prod-01, nodepool: gpu-h100 }
spec:
  clusterName: gpu-prod-01
  replicas: 8                            # scale the GPU pool by editing this integer
  selector:
    matchLabels: { cluster.x-k8s.io/cluster-name: gpu-prod-01, nodepool: gpu-h100 }
  template:
    metadata:
      labels: { cluster.x-k8s.io/cluster-name: gpu-prod-01, nodepool: gpu-h100 }
    spec:
      clusterName: gpu-prod-01
      version: v1.36.1                   # workers may lag the CP by up to 3 minors (10.3)
      deletion:
        nodeDrainTimeoutSeconds: 3600    # GPU workers: give a training job an hour to
                                         # checkpoint before force-deleting pods
      bootstrap:
        configRef:
          apiGroup: bootstrap.cluster.x-k8s.io
          kind: KubeadmConfigTemplate
          name: gpu-prod-01-md-0
      infrastructureRef:
        apiGroup: infrastructure.cluster.x-k8s.io
        kind: Metal3MachineTemplate
        name: gpu-prod-01-md-0
---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
kind: KubeadmConfigTemplate
metadata:
  name: gpu-prod-01-md-0
  namespace: fleet
spec:
  template:
    spec:
      joinConfiguration:                 # NO `controlPlane:` key → a plain worker join
        nodeRegistration:
          name: '{{ ds.meta_data.local_hostname }}'
          kubeletExtraArgs:
            - { name: cloud-provider, value: baremetal }
---
# ── 6. The physical machine, as an object ─────────────────────────────────────
apiVersion: v1
kind: Secret
metadata: { name: rack1-slot7-bmc, namespace: fleet }
type: Opaque
stringData:
  username: bmcadmin
  password: <from your secret store, not from git>
---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: rack1-slot7
  namespace: fleet
  labels:
    failure-domain: rack1                # KCP spreads control-plane Machines across these
spec:
  online: true                           # the operator's power intent; BMO enforces it
  bootMACAddress: "b4:96:91:0a:1c:7e"    # which NIC will PXE — matched against DHCP
  bootMode: UEFI                         # UEFI | legacy | UEFISecureBoot
  automatedCleaningMode: metadata
  bmc:
    address: redfish-virtualmedia+https://10.20.1.7/redfish/v1/Systems/1
    # ^ the SCHEME picks the Ironic driver. Options include ipmi://, redfish://,
    #   redfish-virtualmedia:// (mounts an ISO over the BMC — no PXE/DHCP needed,
    #   which is why it is the modern default), idrac://, irmc://.
    credentialsName: rack1-slot7-bmc
    disableCertificateVerification: false   # true only for self-signed BMC certs you
                                            # have consciously decided to accept
  rootDeviceHints:
    deviceName: /dev/nvme0n1             # WITHOUT this, Ironic picks a disk by its own
                                         # heuristics — which on a GPU box with 8 NVMe
                                         # data drives is a coin flip you will lose.
                                         # Other hints: model, serialNumber, minSizeGigabytes,
                                         # wwn, rotational.
```

Six objects, and cluster #41 costs one `kubectl apply`. Note what is **not** in there: no CA
generation, no cert SAN list you have to remember, no bootstrap token, no join command, no VIP
setup, no upgrade sequence. Each of those became a field or vanished into a controller.

### Bare metal: what Metal3 does that a cloud provider does not

A **cloud** provider turns a `Machine` into a node by calling an API that already virtualises
everything: `RunInstances` returns a booted VM with an OS image, a network interface and a disk in
seconds. Every hard part is somebody else's.

A **bare-metal** provider has no such API. It must drive physical hardware over out-of-band
management protocols, and the `BareMetalHost` object is a state machine over that work. From
`baremetal-operator`'s `ProvisioningState` constants:

```
   (a BareMetalHost is created with BMC address + credentials)
                    │
                    ▼
            ┌───────────────┐   insufficient info to even talk to the BMC
            │  unmanaged    │   (no address or no credentials)
            └───────┬───────┘
                    ▼
            ┌───────────────┐   BMO tells Ironic about the host and validates
            │  registering  │   the BMC credentials by actually connecting.
            └───────┬───────┘   FAILS AS: RegistrationError — wrong password,
                    │            unreachable BMC, unsupported driver
                    ▼
            ┌───────────────┐   Ironic boots the IPA (Ironic Python Agent)
            │  inspecting   │   ramdisk over PXE or virtual media and reads the
            └───────┬───────┘   HARDWARE: CPUs, RAM, NIC MACs and link state,
                    │            disks (name, size, WWN, rotational), firmware.
                    │            → status.hardware, and it is the reason you can
                    │              query your fleet's inventory with kubectl.
                    │            FAILS AS: InspectionError — no PXE response,
                    │            IPA cannot reach the Ironic API, disk not visible
                    ▼
            ┌───────────────┐   apply firmware/BIOS settings, and CLEAN disks
            │   preparing   │   per automatedCleaningMode.
            └───────┬───────┘   FAILS AS: PreparationError
                    ▼
            ┌───────────────┐   ◀── THE HOST NOW SITS HERE, POWERED OFF,
            │   available   │       waiting to be claimed. This is your
            └───────┬───────┘       "spare capacity" pool, and it is queryable.
                    │  a Metal3Machine claims it (consumerRef set)
                    ▼
            ┌───────────────┐   write the OS image to rootDeviceHints' disk
            │ provisioning  │   (streamed by IPA, checksum verified), write the
            └───────┬───────┘   cloud-init/ignition userData from the bootstrap
                    │            Secret, set the boot device, power on.
                    │            FAILS AS: ProvisioningError — checksum mismatch,
                    │            image server unreachable, disk too small
                    ▼
            ┌───────────────┐   the OS boots; kubeadm/Talos runs; the node joins.
            │  provisioned  │   BMO now only maintains power state and watches
            └───────┬───────┘   for credential changes.
                    │            FAILS AS: ProvisionedRegistrationError
                    │  Machine deleted → Metal3Machine releases the host
                    ▼
            ┌───────────────┐   wipe the disks again (per cleaning mode) so the
            │deprovisioning │   next tenant cannot read the last one's data
            └───────┬───────┘
                    ▼
             back to `available`

   Deletion path: powering off before delete → deleting
   Also: `externally provisioned` — something else owns this machine's image
         (used when you adopt an existing cluster into Metal3).
```

The one-line answer to the interview question: **a bare-metal provider owns the physical
provisioning lifecycle — BMC power control over IPMI/Redfish, PXE or virtual-media boot, hardware
inspection, disk imaging with checksum verification, and disk cleaning between tenants — because
there is no cloud API to hand it a ready OS.** Only after all of that does bootstrap (kubeadm or
Talos) run at all.

Three consequences that only exist on bare metal:

- **Inspection gives you an inventory for free.** `status.hardware` on every BMH means
  `kubectl get bmh -o json | jq` answers "which hosts have 8 NVMe drives and 2 TB of RAM" without
  a CMDB. That is a real operational asset, and it is the hook for 10.6's remediation loop.
- **Cleaning is a security boundary, not housekeeping.** `automatedCleaningMode: disabled` on a
  multi-tenant fleet means the next tenant can read the previous tenant's disks.
- **`rootDeviceHints` is not optional on a GPU box.** A DGX-class server has a boot NVMe and
  several data NVMes; without a hint, Ironic's heuristics choose, and the failure mode is that you
  image over the wrong drive.

**Metal3's standing, precisely.** Metal3 reached **CNCF Incubating** status on a Technical
Oversight Committee vote announced **27 August 2025**, five years after entering the Sandbox in
September 2020. At incubation the project reported **57 active contributing organizations**, led by
**Red Hat and Ericsson** (its original 2019 co-founders), with named production adopters including
**Fujitsu, Ikea, SUSE, Ericsson and Red Hat**. That matters as evidence that the
`BareMetalHost`/Ironic model is a shared, multi-vendor investment rather than one company's fork of
OpenStack tooling.

**It is not the only option, though.** **Tinkerbell** (CAPI provider CAPT) solves the same problem
with a workflow engine — Smee for DHCP/iPXE, Tink for workflow execution, Hegel for metadata,
Rufio for BMC control, HookOS as the in-memory installer — modelling provisioning as
*workflow-as-data* rather than Metal3's *declarative host state*. And Sidero Labs' Talos-native
stack bypasses Ironic entirely, talking to Talos's own API for bring-up. Choose on your team's
operational preference, not on which name you have heard.

### Talos: the immutable-node answer

Talos Linux attacks the same fleet problem one layer down: not "how do I create nodes
declaratively" but "how do I stop nodes from diverging after they exist."

**What is actually removed.** No SSH daemon. No shell. No package manager. No systemd. No
interactive login of any kind. The only interface is a gRPC API (`apid`, port **50000**, mutually
authenticated), driven by `talosctl`. The root filesystem is a read-only squashfs; the writable
parts are explicit and bounded.

**The disk layout is fixed and meaningful** (labels from `pkg/machinery/constants/constants.go`):

```
  /dev/nvme0n1
  ├── EFI          the ESP: bootloader + (with SecureBoot) Unified Kernel Images
  ├── BIOS         GRUB stage-2, on legacy-boot systems only
  ├── BOOT         kernel + initramfs for boot entries "A" and "B"
  │                   grub/constants.go: BootA = "A", BootB = "B", BootReset = "Reset"
  ├── META         a tiny key-value area the machine writes to itself:
  │                   which slot to boot next, the staged upgrade, install state
  ├── STATE        mounted at /system/state — the MACHINE CONFIG and the node's
  │                   secrets. This is the only thing you must not lose.
  └── EPHEMERAL    mounted at /var — container images, container state, kubelet
                       data, logs. Deliberately named: it is expected to be wiped.

  UPGRADE = write the new system to the INACTIVE boot slot, point META at it,
            reboot. If the new slot fails to come up, roll back to the other one.
            STATE survives; EPHEMERAL survives unless you ask for it to be wiped.
            (On UEFI+SecureBoot systems the same idea is expressed with two UKIs
             in the ESP and systemd-boot choosing between them.)
```

That is why the upgrade story is genuinely different from `apt upgrade`: there is no partially
upgraded state. Either you booted the new image or you booted the old one.

**The machine config is one declarative artifact.** As of the Talos 1.14 line the configuration is
a **multi-document YAML**: the classic `v1alpha1.Config` document plus a growing set of typed
documents (`KubeApiServerConfig`, `KubeletConfig`, network link/route documents, volume
configuration, and so on), each independently versioned. A control-plane node's configuration,
abbreviated to the fields that carry meaning:

```yaml
version: v1alpha1
machine:
  type: controlplane              # controlplane | worker. Determines what runs here.
  token: 328hom.uqjzh6jnn2eie9oi  # joins the machine to the cluster PKI; the node then
                                  # CSRs for its own identity (10.1's TLS bootstrap,
                                  # over Talos's API rather than Kubernetes')
  ca:                             # the machine CA — Talos's own trust root, separate
    crt: LS0tLS1CRUdJTi...        # from the Kubernetes CA below
    key: LS0tLS1CRUdJTi...
  certSANs:                       # extra SANs for the machine's API certificate;
    - 10.10.0.11                  # all non-loopback interface IPs are added automatically
  install:
    disk: /dev/nvme0n1            # or diskSelector (model/size/serial) — the same
                                  # rootDeviceHints problem as Metal3, same answer
    image: factory.talos.dev/metal-installer/<schematic-id>:v1.14.0
    # ^ Image Factory: the schematic id encodes exactly which system extensions are
    #   baked in — e.g. nvidia-container-toolkit and the NVIDIA kernel modules.
    #   Two nodes with the same schematic + version are bit-identical, which is
    #   how you get 10.1's "GPU driver stage" down to zero runtime minutes.
  kubelet:
    extraArgs:
      rotate-server-certificates: "true"
  network:
    hostname: cp1
    interfaces:
      - interface: eth0
        addresses: ["10.10.0.11/24"]
        routes:
          - { network: 0.0.0.0/0, gateway: 10.10.0.1 }
        vip:
          ip: 10.10.0.100         # Talos's OWN control-plane VIP: no kube-vip needed.
                                  # Ownership is decided through etcd, so the same
                                  # "who holds it" question as 10.3, different mechanism.
cluster:
  clusterName: gpu-prod-01
  controlPlane:
    endpoint: https://10.10.0.100:6443
  ca: { crt: LS0tLS1CRUdJTi..., key: LS0tLS1CRUdJTi... }   # the Kubernetes CA (10.1)
  token: wlzjyw.bei2zfylhs2by0wd                            # the Kubernetes bootstrap token
  etcd:
    ca: { crt: LS0tLS1CRUdJTi..., key: LS0tLS1CRUdJTi... }  # the SEPARATE etcd CA (10.1)
    extraArgs:
      election-timeout: "5000"    # NOTE: Talos REJECTS a documented list of etcd flags
                                  # here — name, data-dir, initial-cluster-state,
                                  # listen-*-urls, cert-file, key-file, trusted-ca-file,
                                  # peer-*. Those are Talos's to own, not yours.
    advertisedSubnets: ["10.10.0.0/24"]   # pick the etcd peer IP from this subnet rather
                                          # than "first routable address" — matters on
                                          # multi-homed GPU nodes with storage/IB NICs
  inlineManifests: []             # bootstrap-time manifests (CNI, etc.)
```

Look at what that single file contains: three CAs, the bootstrap tokens, the VIP, the install
disk, the OS image, and the etcd tuning. **It is lessons 10.1, 10.2 and 10.3 as data.**

**Upgrades.** `talosctl upgrade` writes the new system image to the inactive slot and reboots into
it. Real flags and defaults from the CLI reference in the Talos repo:

| Flag | Default | Meaning |
|---|---|---|
| `--image` | the installer matching the client's version | which system image to install |
| `--drain` | `true` | cordon and evict the node's pods before rebooting |
| `--drain-timeout` | `5m0s` | how long to wait for the drain |
| `--reboot-mode` | `default` | `default` uses kexec where possible; `powercycle` forces a full firmware reboot |
| `--no-reboot` | `false` | stage the upgrade without rebooting (reboot on your own schedule) |
| `--wait` | `true` | track progress until the node is back |
| `--timeout` | `30m0s` | overall deadline when waiting |

Two of those matter on a GPU fleet. `--reboot-mode powercycle` is what you want when the *firmware*
must re-initialise — after a BIOS or NIC firmware change, or when a GPU is in a bad state that
kexec would carry across. And `--drain-timeout 5m` is far too short for a node running a training
job that checkpoints every 20 minutes; raise it or pre-drain deliberately (10.6).

**Why this kills drift.** With a mutable distro, every node accumulates hand-fixes, half-applied
config-management runs, and package skew: 200 nodes become 200 *slightly different* nodes, and
drift is where 3am outages live. Talos removes the *mechanisms* that create drift rather than
policing their effects — there is no shell in which to make a one-off change, the root filesystem
is read-only, and the entire node is defined by one declared artifact. Two nodes on the same
version with the same machine config and the same Image Factory schematic are equivalent by
construction. This is precisely the failure mode Equinix describes hitting with Ansible/Kubespray
before migrating (see Real-world use cases).

**Talos under CAPI.** Sidero Labs ships its own providers — `TalosControlPlane` (CACPPT) and
`TalosConfig`/`TalosConfigTemplate` (CABPT) — which slot into the same `Cluster`/`Machine` graph in
place of the kubeadm ones:

```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1alpha3
kind: TalosControlPlane
metadata:
  name: talos-cp
spec:
  version: v1.36.1
  replicas: 3
  infrastructureTemplate:                 # note: still the older corev1.ObjectReference
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
    kind: Metal3MachineTemplate
    name: talos-cp
  controlPlaneConfig:
    controlplane:
      generateType: controlplane          # let the provider generate the machine config
      # generateType: none + `data: |` to supply a full hand-written machine config
      # instead — which also means the provider can no longer generate a talosconfig
      # for you, so keep the one `talosctl gen config` produced.
```

The trade against kubeadm-based CAPI is clean: Talos gives you immutability and a much smaller
attack surface; kubeadm gives you a node you can still `ssh` into when something is genuinely
novel, and an ecosystem of tooling that assumes a normal Linux. On a GPU fleet the specific thing
to check first is whether the driver and fabric stack you need is available as a Talos system
extension — because if it is not, you will be building images, not editing config.

### Management cluster, workload clusters, and pivoting

The **management cluster** is a (usually small, long-lived) Kubernetes cluster running the CAPI
core controllers plus the provider controllers. It holds the objects and reconciles the workload
clusters toward them. `clusterctl init --infrastructure <provider>` installs the controllers;
`clusterctl generate cluster` renders a template; `clusterctl describe cluster` gives you the tree
view; `clusterctl get kubeconfig <name>` extracts the workload cluster's admin kubeconfig from the
Secret CAPI created.

**Pivoting** is the trick that makes this bootstrappable. You create a throwaway `kind` cluster,
install the providers, use it to build a real management cluster, then `clusterctl move` the
objects into that cluster — which now manages itself and everything else. The temporary cluster is
deleted. This is the CAPI answer to the same chicken-and-egg problem static pods solve in 10.1: you
need a Kubernetes cluster to create Kubernetes clusters, so you make a disposable one.

**Do not run workloads on the management cluster.** It exists to host controllers. Putting
production traffic on it couples the fleet's control plane to the fleet's data plane and defeats
the isolation CAPI is designed to give you — and it makes the management cluster's own upgrades
into a customer-visible event.

### GitOps: the fourth leg, and why it is not optional at the edge

A management cluster reconciling 40 remote clusters raises an obvious question: what happens to a
workload cluster when its link back to the management cluster is down?

The answer used in production by both Das Schiff and Sylva is to pair CAPI with a **GitOps
controller (Flux) running inside each workload cluster**, not only in the management cluster. The
division of labour:

```
   ┌────────────────────────────┐              ┌────────────────────────────┐
   │  MANAGEMENT CLUSTER        │              │  GIT                       │
   │  CAPI + provider ctrls     │              │  cluster manifests +       │
   │                            │              │  platform/app manifests    │
   │  owns MACHINE-level state: │              └─────────────┬──────────────┘
   │   • is this node imaged?   │                            │
   │   • is it healthy?         │                            │ each workload
   │   • is it on the right     │                            │ cluster's own Flux
   │     Kubernetes version?    │                            │ pulls DIRECTLY
   └─────────────┬──────────────┘                            │
                 │ needs connectivity                        │
                 │ TO the workload cluster                    │
                 ▼                                            ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  WORKLOAD CLUSTER at a remote site                                   │
   │   • Flux reconciles the platform/app layer straight from Git         │
   │   • KEEPS WORKING when the management cluster is unreachable         │
   │   • what you lose during a partition: new machines, replacements,    │
   │     version rollouts — everything machine-level                      │
   │   • what you keep: the whole application and platform layer          │
   └──────────────────────────────────────────────────────────────────────┘
```

That split is what turns "declarative fleet" from a provisioning convenience into an **autonomy
guarantee**, and it is a hard requirement at cell-tower and edge sites with unreliable backhaul.
It is also good practice in a datacentre: it means the management cluster's own maintenance window
is not a fleet-wide freeze.

### Fleet arithmetic: what reconciliation actually buys

Two calculations, both of which belong in the 10.8 capex model.

**1. Provisioning wall-clock.** Reuse 10.1's model, `T ≈ S + t × ceil(N / P)`, with `t = 16.5 min`
per node measured there and `S = 24 min` for the serial control-plane prefix:

```
  Hand-run runbook, one engineer:      P = 1   → T = 24 + 16.5 × 200 = 3324 min = 55.4 h
  Reconciled, Ironic conductor limit:  P = 8   → T = 24 + 16.5 ×  25 =  436 min =  7.3 h
  Reconciled, tuned + more conductors: P = 20  → T = 24 + 16.5 ×  10 =  189 min =  3.2 h

  For a 200-node fleet, that is 55 hours versus 3.2 hours for the SAME hardware.
  The difference is entirely the parallelism a controller can sustain and a human
  cannot — and note that P is now bounded by Ironic's max_concurrent_deploy and
  your image server, both of which are numbers you can raise, rather than by how
  many terminals one person can watch.
```

Bake the GPU stack into the image (an Image Factory schematic, or a pre-baked qcow2) and `t` drops
from 16.5 to about 10.5 minutes, because 10.1's 6-minute driver stage disappears. At `P = 20` that
is `24 + 105 = 129 min`. **Immutable images are worth about an hour on a 200-node rebuild**, every
time you rebuild.

**2. Operators per cluster.** This is the argument that actually convinces a CFO:

```
  Imperative: each cluster needs a human for build, upgrade, and incident response.
              Empirically ~0.25 FTE/cluster once you include on-call rotation.
              40 clusters → 10 FTE.  At a fully-loaded $220k/yr (2026 snapshot,
              flag it as dated) → $2.2M/yr.

  Reconciled: one platform team owns the MANAGEMENT cluster and the templates.
              Marginal cost of cluster #41 ≈ a manifest + the hardware.
              Say 4 FTE for the platform team regardless of fleet size, plus a
              genuinely small per-cluster increment (~0.02 FTE) for tenant-specific
              work.
              40 clusters → 4 + 0.8 = 4.8 FTE → $1.06M/yr.

  Delta ≈ $1.14M/yr at 40 clusters — and the curves diverge: the imperative model
  is linear in cluster count, the reconciled one is nearly flat.

  CROSSOVER: 0.25 × N = 4 + 0.02 × N  →  0.23N = 4  →  N ≈ 17 clusters.
  Below ~17 clusters the platform-team overhead is not repaid; above it, the
  reconciled model wins and keeps winning.
```

State the assumptions loudly — FTE-per-cluster is the number everyone disputes, and the crossover
is entirely determined by it. The durable content is the *shape*: **a fixed platform cost against a
linear per-cluster cost, with a crossover you can compute for your own numbers.** That is exactly
the structure of the 10.8 capex-vs-cloud model, applied to people instead of hardware.

### Remediation as a controller (preview of 10.6)

One more reason the `Machine` abstraction matters: it is the natural place to hang automated
repair. A `MachineHealthCheck` watches Machines matching a selector and, when a node's conditions
are bad for longer than a timeout, deletes the Machine — which causes its owner (MachineSet or
KCP) to create a replacement, which the infrastructure provider provisions on fresh hardware.

Two safety properties are built in and both are load-bearing on a fleet:

- **`maxUnhealthy` / `unhealthyRange`** stops a mass-remediation storm. If half your nodes go
  `NotReady` at once, the cause is almost certainly the control plane or the network, not fifty
  simultaneous hardware failures — and re-imaging fifty healthy GPU nodes would be a catastrophe.
- **KCP's own remediation** is deliberately more careful for control-plane machines: it annotates
  the operation as in-progress and applies a relaxed-but-still-real preflight so that a cluster
  with multiple failures can still recover, rather than deadlocking on "everything must be healthy
  before I fix anything."

Lesson 10.6 builds the full NPD → node condition → cordon/drain → RMA loop on top of this.

## Perspectives

**Developer / platform consumer.** From inside a workload cluster nothing looks different from any
other Kubernetes cluster — `kubectl`, standard APIs, standard workloads. The fleet machinery is
invisible until a node needs replacing, at which point it just... does, without a ticket. The one
thing they will notice is `nodeDrainTimeoutSeconds`, because it decides whether their long-running
job gets a chance to checkpoint.

**Operator.** The job shifts from "run commands on nodes" to "edit specs and read conditions." A
CP upgrade is one field and then watching Machine phases, not SSHing into three boxes in the right
order. The failure mode shifts too: instead of "I forgot a step," it is "this Machine has been
`Provisioning` for 40 minutes — why," which demands you understand the provider well enough to
read `BareMetalHost.status.provisioning.state` and the Ironic logs behind it. **You have not
removed the need to understand 10.1–10.3; you have made it a debugging skill instead of a typing
skill.**

**Hardware / physical layer.** CAPI and Talos do not remove the physical provisioning problem —
they formalise it. Someone still has to drive a BMC, wait for a PXE boot, verify an image checksum
and stream it to the right disk. CAPI gives that work a Kubernetes-native API surface and a state
machine instead of a shell script, and — via BMH inspection — turns your rack into a queryable
inventory. Lesson 10.5 is that layer in full depth.

**Economics.** A single management cluster amortises its operational cost across every workload
cluster it reconciles: the 3-node HA design from 10.3, paid for once, governs 40 fleets instead of
one. The crossover math above puts a number on it (~17 clusters with the stated assumptions), and
that number is the actual argument for adopting CAPI. Below it, you are paying for a platform team
to manage a handful of clusters; above it, the marginal cost of a cluster approaches "apply a
manifest."

## Real-world use cases

- **Deutsche Telekom — "Das Schiff"** — <https://github.com/telekom/das-schiff> — one of the
  largest known bare-metal Cluster API deployments: CAPI + **Metal3** (+ vSphere for VM-based
  sites) + **Flux**, managing Kubernetes clusters at hundreds of locations across Germany,
  including remote edge and cell-tower sites. What it shows: the CAPI-plus-GitOps split described
  above is not a design exercise — it is a production requirement when sites must stay operable
  while the management cluster is unreachable. It also demonstrates the management-cluster pattern
  at a scale where per-cluster human ownership is arithmetically impossible.
- **Sylva Project (Linux Foundation Europe)** —
  <https://sylvaproject.org/ocudu-on-telco-clouds-0-introduction-to-sylva/>, background at
  <https://the-mobile-network.com/2022/11/why-the-eu-big-five-are-launching-sylva/> — Vodafone,
  Deutsche Telekom, Orange, Telecom Italia and Telefónica jointly launched Sylva (November 2022)
  as a shared open-source telco cloud stack explicitly built around **Cluster API, Metal3 and
  Flux**. What it shows: multi-carrier corroboration, independent of Das Schiff, that this exact
  stack is the accepted industry answer for telco-scale bare-metal fleets — five competitors
  converging on one architecture is a stronger signal than any single vendor's reference design.
- **Sidero Labs — "Equinix switches from KubeSpray to Talos Linux"** —
  <https://www.siderolabs.com/case-studies/equinix-switches-from-kubespray-to-talos-linux-cutting-deployment-time-while-maintaining-security>
  — Equinix's managed-Kubernetes team moved off Kubespray-plus-Flatcar (brittle Ansible, slow
  upgrades, security-compliance friction) onto Talos, cutting VM cluster deployment time from
  **roughly 45 minutes to under 10** (a 2026 snapshot of a migration begun around 2019 and
  completed by end-of-lifeing Kubespray). What it shows: a numbers-backed before/after for the
  "immutable OS kills drift" claim — and read it for the *operational burden* story (convoluted
  Ansible, upgrades that tied up SREs for days) rather than the headline number, because the
  burden is what compounds at fleet scale.
- **Metal3 becomes a CNCF Incubating project** —
  <https://www.cncf.io/blog/2025/08/27/metal3-io-becomes-a-cncf-incubating-project/> — the TOC vote
  announced 27 August 2025, with 57 contributing organizations, Red Hat and Ericsson leading, and
  Fujitsu, Ikea, SUSE, Ericsson and Red Hat as named adopters. What it shows: the
  `BareMetalHost`/Ironic model is a multi-vendor investment, which is the argument you make when
  someone proposes writing an in-house netboot script instead.

## Worked example — CAPI Docker provider (CAPD): watch a Machine reconcile

CAPD provisions "machines" as Docker containers, so you can watch the exact reconcile loop a
bare-metal provider runs — same core objects, same phases, same controller behaviour — without any
hardware. Prerequisites: Docker, `kind`, `clusterctl`, `kubectl`.

```bash
# 1. A throwaway MANAGEMENT cluster.
kind create cluster --name mgmt

# 2. Install CAPI core + the kubeadm bootstrap/control-plane providers + Docker infra.
export CLUSTER_TOPOLOGY=true          # CAPD's templates use ClusterClass
clusterctl init --infrastructure docker

# 3. Render a workload cluster: 3 control-plane machines, 3 workers.
clusterctl generate cluster capi-quickstart --flavor development \
  --kubernetes-version v1.36.1 \
  --control-plane-machine-count=3 \
  --worker-machine-count=3 \
  > capi-quickstart.yaml

# READ THIS FILE BEFORE APPLYING IT. Map every object to the graph above and to
# the thing you did by hand in 10.1-10.3. That reading is the point of the exercise.
kubectl apply -f capi-quickstart.yaml
```

Now watch the reconcile loop turn objects into machines:

```console
$ clusterctl describe cluster capi-quickstart
NAME              PHASE         AGE   VERSION
capi-quickstart   Provisioned   8s    v1.36.1

$ kubectl get kubeadmcontrolplane
NAME                    CLUSTER           INITIALIZED  API SERVER AVAILABLE  REPLICAS  READY  UPDATED  UNAVAILABLE  AGE   VERSION
capi-quickstart-g2trk   capi-quickstart   true                               3                3        3            4m7s  v1.36.1

$ kubectl get machines -w
NAME                          CLUSTER          NODE NAME  READY  AVAILABLE  UP-TO-DATE  PHASE         AGE  VERSION
capi-quickstart-g2trk-2xk4p   capi-quickstart             False  False      True        Pending        3s  v1.36.1
capi-quickstart-g2trk-2xk4p   capi-quickstart             False  False      True        Provisioning  11s  v1.36.1
capi-quickstart-g2trk-2xk4p   capi-quickstart             False  False      True        Provisioned   48s  v1.36.1
capi-quickstart-g2trk-2xk4p   capi-quickstart  ...-2xk4p  False  False      True        Running       71s  v1.36.1
# ^ note: Running but READY=False. The Node exists and joined, but there is no CNI
#   yet, so it is NotReady. This is exactly 10.1's stage 8.
```

**Watch the control plane come up one machine at a time.** This is the observation that matters
most: KCP does *not* create three machines at once. It creates one, waits for it to be `Running`
with a healthy etcd member, then creates the second. That is the preflight machinery from the Core
concepts section, visible in a terminal:

```console
$ kubectl get machines -l cluster.x-k8s.io/control-plane -w
# t+0:00  one Machine, Provisioning
# t+1:20  one Machine Running        ← the FIRST control plane node: `kubeadm init`
# t+1:22  a SECOND Machine appears   ← only now, because preflight passed
# t+2:40  two Machines Running       ← `kubeadm join --control-plane`, etcd 2 members
# t+2:42  a THIRD Machine appears
# t+4:05  three Machines Running     ← etcd 3 members, quorum 2 (10.2's arithmetic)
```

Install a CNI and confirm it is a real cluster:

```bash
clusterctl get kubeconfig capi-quickstart > capi-quickstart.kubeconfig
kubectl --kubeconfig capi-quickstart.kubeconfig apply -f <your CNI manifest>
kubectl --kubeconfig capi-quickstart.kubeconfig get nodes
```

**Then change desired state and watch reconciliation, rather than reading about it.**

```console
# (a) Scale the worker pool.
$ kubectl scale machinedeployment capi-quickstart-md-0 --replicas=5
machinedeployment.cluster.x-k8s.io/capi-quickstart-md-0 scaled
$ kubectl get machines -w     # two new Machines appear, Pending → … → Running

# (b) Break something and watch it heal. Delete a Machine object directly:
$ kubectl delete machine capi-quickstart-md-0-9f4tx-b2n
$ kubectl get machines -w
#  the deleted Machine goes Deleting → Deleted (cordon, drain, delete the
#  DockerMachine), and the MachineSet immediately creates a REPLACEMENT to
#  restore replicas=5. Nothing you did was "undo"; the controller simply
#  re-converged.

# (c) Upgrade the control plane by editing ONE field.
$ kubectl patch kubeadmcontrolplane capi-quickstart-g2trk --type=merge \
    -p '{"spec":{"version":"v1.36.2"}}'
$ kubectl get machines -l cluster.x-k8s.io/control-plane -w
#  Watch the surge: replicas go 3 → 4 (a v1.36.2 Machine is created FIRST),
#  then back to 3 as an old one is drained and removed. Repeat twice more.
#  This is 10.3's entire upgrade runbook, executed by a controller, with the
#  etcd-leadership forwarding and member removal you did by hand.
```

Step (c) is the whole lesson in one command. Compare the wall-clock and the number of decisions
you made against 10.3's worked example.

**The key generalisation for your target job:** the `Machine` is the reconciled unit, and it is
provider-neutral. Swap `--infrastructure docker` for `--infrastructure metal3` and the *same*
`Cluster`, `KubeadmControlPlane`, `MachineDeployment` and `Machine` objects now drive Ironic to
inspect, clean and image physical servers over Redfish. Writing that swap-in half is the CAPI
provider work NVIDIA and CoreWeave are hiring for.

## Practice (hands-on, cheap VMs → deliverable)

Pick **one** path; either satisfies acceptance. Path B is closer to the job target, Path A is
closer to the GPU-fleet reality.

**Path A — Talos (fastest reproducible bare-metal-like cluster):**

```bash
talosctl cluster create --name demo --controlplanes 1 --workers 1
talosctl --nodes <cp-ip> get members
talosctl config nodes <cp-ip> && talosctl kubeconfig .
kubectl get nodes

# Inspect the immutable/declarative model:
talosctl --nodes <cp-ip> get machineconfig -o yaml | head -60   # the single declared artifact
talosctl --nodes <cp-ip> get disks                              # the fixed partition layout
talosctl --nodes <cp-ip> read /proc/mounts | grep -E 'squashfs|EPHEMERAL|STATE'
talosctl --nodes <cp-ip> upgrade --image ghcr.io/siderolabs/installer:<newer> --wait
```

Capture: (1) the machine config, (2) evidence there is no SSH (`ss -tlnp` is not available — that
*is* the evidence; note what `talosctl` exposes instead), (3) the mount table showing a read-only
root and a separate `EPHEMERAL`, and (4) the upgrade output showing the reboot into the other
boot slot. Then edit one field of the machine config, `talosctl apply-config`, and note whether it
required a reboot — Talos tells you which fields are applied live and which are not.

**Path B — CAPI + Docker (CAPD), to watch the reconcile loop:** run the worked example above,
including all three of (a) scale, (b) delete-and-heal, and (c) the one-field control-plane
upgrade. Capture `clusterctl describe cluster`, the `kubectl get machines -w` transitions, and —
importantly — evidence that KCP surged to 4 replicas before dropping back to 3.

**Both paths, then:** open `capi-quickstart.yaml` (or the Talos machine config) and write a
mapping table: for each object or top-level field, which hand-built step from 10.1–10.3 it
replaces. That table is the deliverable's spine.

**Acceptance (feeds the capex-vs-cloud writeup):** a **Talos-provisioned OR CAPI-provisioned
cluster**, plus a short note documenting:

- the **reconcile behaviour you observed** — for CAPD, the Machine phase transitions, the
  heal-after-delete, and the surge-then-shrink upgrade; for Talos, the single declarative config
  and the A/B upgrade;
- the **mapping table** from objects/fields back to the hand-built steps in 10.1–10.3;
- one paragraph on where a **bare-metal provider (Metal3/Ironic or Tinkerbell)** slots in for real
  hardware, naming at least three things it does that a cloud provider does not;
- and your own version of the **fleet arithmetic** — provisioning wall-clock at your `P`, and the
  operators-per-cluster crossover with your own FTE assumptions stated explicitly.

## Common pitfalls

- **Treating Metal3 as the only credible bare-metal provider.** Tinkerbell (CAPT) and Sidero's
  Talos-native stack are real and independently adopted. Choose on operational preference —
  declarative host state versus workflow-as-data, kubeadm-based versus Talos-native — not on which
  name you have heard.
- **Omitting `rootDeviceHints` / `install.disk`.** On a GPU box with a boot NVMe and eight data
  NVMes, letting the provisioner pick a disk by heuristic is how you image over a data drive. Pin
  it by device name, WWN or serial, and validate the hint against `status.hardware` from
  inspection.
- **Setting `automatedCleaningMode: disabled` on a multi-tenant fleet.** Cleaning is a security
  boundary between tenants, not housekeeping. Disable it only for single-tenant hardware where you
  have consciously accepted the trade for faster turnaround.
- **Reading `Machine.status.phase` in automation.** The API's own comment says the phase "should
  not be interpreted by any software components as a reliable indication of the actual state."
  Read conditions (`Ready`, `Available`, `UpToDate`) instead. And note `Failed` is deprecated in
  v1beta2 — controllers no longer set it, so a runbook that greps for it will find nothing while a
  cluster burns.
- **Using `apiVersion` where v1beta2 wants `apiGroup`.** `ContractVersionedObjectReference` takes
  `apiGroup`/`kind`/`name`; the version is resolved from the provider CRD's contract labels. Also:
  references are same-namespace only.
- **Confusing the management cluster with a workload cluster.** The management cluster hosts
  controllers and should run no production workloads. Coupling the fleet's control plane to its
  data plane defeats the isolation and makes management-cluster maintenance a customer-visible
  event.
- **Assuming CAPI removes the physical-provisioning problem.** It formalises it into a controller
  that still drives real BMCs and PXE boots. When a `Machine` is stuck `Provisioning`, the answer
  is in `BareMetalHost.status.provisioning.state` and the Ironic logs — which is lesson 10.5's
  material, and why you cannot skip it.
- **Ignoring the GitOps leg at fleet scale.** CAPI alone reconciles machine-level state. Without a
  Flux (or equivalent) instance *inside* each workload cluster, a partition from the management
  cluster leaves that site's platform layer frozen. Das Schiff and Sylva both pair CAPI with Flux
  for exactly this reason.
- **Leaving `nodeDrainTimeoutSeconds` at a small value on GPU workers.** A rollout that
  force-deletes pods after 10 minutes will destroy training jobs mid-epoch. Match the timeout to
  your checkpoint interval, and see 10.6 for the drain-a-500-GPU-job problem in full.
- **Baking ad-hoc state into "immutable" Talos nodes.** The whole value proposition evaporates the
  moment someone finds a way to persist a one-off change. Talos removes the shell specifically to
  make that impossible; do not fight the design by mounting a writable escape hatch.

## Self-check

- **What does a bare-metal CAPI provider (Metal3/Ironic) do that a cloud CAPI provider does not?**
  **Answer:** It owns the entire **physical** provisioning lifecycle a cloud API hides. A cloud
  provider calls something like `RunInstances` and receives a booted, imaged VM. Metal3 models each
  server as a `BareMetalHost` and drives **Ironic** over the BMC (IPMI, Redfish, Redfish
  virtual-media, iDRAC, iRMC — the URL scheme selects the driver) through a state machine:
  **registering** (validate BMC credentials by connecting), **inspecting** (boot the Ironic Python
  Agent ramdisk and read CPUs, RAM, NIC MACs, disks, firmware into `status.hardware`),
  **preparing** (apply firmware/BIOS settings, clean disks), **available** (powered off, waiting to
  be claimed), **provisioning** (stream the OS image to the disk chosen by `rootDeviceHints`,
  verify its checksum, write cloud-init userData, set boot device, power on), **provisioned**, and
  on release **deprovisioning** (wipe again). Only after all of that does kubeadm or Talos run.
  Three bare-metal-only consequences: inspection gives you a queryable hardware inventory; cleaning
  is a tenant-isolation boundary; and `rootDeviceHints` is mandatory on multi-disk hardware.

- **Why does an immutable OS like Talos reduce config drift across 200 nodes?**
  **Answer:** It removes the *mechanisms* that create drift rather than policing their effects.
  There is no SSH, no shell, no package manager and no interactive login — the only interface is a
  mutually-authenticated gRPC API on port 50000 — so there is nowhere to make a one-off change.
  The root filesystem is a read-only squashfs, with writable state confined to two labelled
  partitions (`STATE` for the machine config and secrets, `EPHEMERAL` for `/var`). The entire node
  is defined by one declarative multi-document machine config plus an Image Factory schematic that
  pins exactly which system extensions are baked in. Upgrades write the new system to the inactive
  boot slot (`A`/`B`) and reboot into it, rolling back on failure — so there is no partially
  upgraded state. Two nodes on the same version, config and schematic are equivalent by
  construction. This is the failure mode Equinix cites hitting with Kubespray/Ansible before
  migrating, where cluster deployment took ~45 minutes and upgrades tied up SREs.

- **What is a management cluster, what does it reconcile, and how do you bootstrap one?**
  **Answer:** A management cluster is a Kubernetes cluster running the CAPI core controllers plus
  the infrastructure and bootstrap provider controllers. It holds the CAPI objects (`Cluster`,
  `KubeadmControlPlane`, `MachineDeployment`, `MachineSet`, `Machine`, `BareMetalHost`, …) and
  reconciles the **workload clusters** those objects describe — creating, replacing and scaling
  their machines and rolling their version upgrades toward the declared spec. No controller runs
  inside the workload cluster; the management cluster reaches in with a kubeconfig it generated and
  stored as a Secret. You bootstrap it by **pivoting**: create a throwaway `kind` cluster,
  `clusterctl init` the providers, use it to build the real management cluster, `clusterctl move`
  the objects across, and delete the temporary one — the same chicken-and-egg problem static pods
  solve in 10.1, solved with a disposable cluster instead. One management cluster can reconcile
  many workload clusters, which is how 40+ clusters become objects instead of runbooks; Das Schiff
  runs exactly this at hundreds of physical locations.

- **How does `KubeadmControlPlane` upgrade a 3-node control plane, and why in that order?**
  **Answer:** With `RollingUpdate` and `maxSurge: 1` it goes **3 → 4 → 3**, repeated three times.
  On each pass: run preflight (no Machines currently deleting; certificates available; every
  machine has a `nodeRef`; `APIServerPodHealthy`, `ControllerManagerPodHealthy`,
  `SchedulerPodHealthy`, `EtcdPodHealthy` and `EtcdMemberHealthy` all true; and **no etcd members
  still in learner mode**), then create a new Machine at the new version and wait for it to be
  Running with a healthy etcd member; then select the outdated machine to remove, **forward etcd
  leadership off it** if it is the Raft leader (choosing the healthiest candidate), remove its etcd
  member, cordon and drain the node, and delete the Machine. Surging first rather than shrinking
  first is the whole point: dropping 3 → 2 would leave the cluster at quorum 2-of-2 — zero fault
  tolerance — for the entire ~15 minutes a bare-metal machine takes to provision, whereas surging
  to 4 confines the reduced-tolerance window to the member removal itself. This is 10.3's manual
  runbook with the preconditions re-checked on every reconcile instead of remembered.

- **Your fleet has an edge site whose link to the management cluster is unreliable. What pattern
  keeps that site reconciling, and who runs it in production?**
  **Answer:** Split the responsibilities. CAPI in the management cluster owns **machine-level**
  state (is this node imaged, healthy, on the right version), which genuinely requires connectivity
  to the site. A **GitOps controller — Flux — running *inside* each workload cluster** owns the
  application and platform layer and pulls straight from Git, so it keeps reconciling when the
  management cluster is unreachable. During a partition you lose new machines, replacements and
  version rollouts; you keep the entire platform layer. Deutsche Telekom's Das Schiff and the
  multi-carrier Sylva Project (Vodafone, DT, Orange, Telecom Italia, Telefónica) both run exactly
  this CAPI + Metal3 + Flux combination across hundreds of distributed, sometimes-unreachable
  sites. It is also good practice in a datacentre, because it means the management cluster's own
  maintenance window is not a fleet-wide freeze.

- **At what fleet size does adopting CAPI pay for itself, and what determines the answer?**
  **Answer:** It is a fixed-cost-versus-linear-cost crossover, and you compute it rather than
  assert it. Imperative operation costs roughly `f` FTE per cluster (build, upgrade, on-call);
  reconciled operation costs a fixed platform-team size `T` plus a small per-cluster increment `g`.
  Setting `f·N = T + g·N` gives `N = T / (f − g)`. With the stated (and disputable) assumptions
  `f = 0.25`, `T = 4`, `g = 0.02`, the crossover is `4 / 0.23 ≈ 17 clusters`. Below that you are
  paying for a platform team to run a handful of clusters; above it the marginal cost of cluster
  #41 approaches "apply a manifest," and the two curves keep diverging. The number everyone argues
  about is `f`, so measure it rather than quoting it — and note the second, independent benefit
  that does not appear in the FTE math: reconciled provisioning sustains a parallelism (`P`) that a
  human runbook cannot, which is the difference between a 55-hour and a 3-hour 200-node build.

## Connections & what's next

This lesson is the synthesis point for 10.1–10.3. The `KubeadmControlPlane` object automates
10.3's HA and upgrade runbook — including 10.2's etcd member management and learner safety — and a
bare-metal `Machine` reconciling through Metal3/Ironic automates 10.1's by-hand node bring-up,
right down to the SAN list and the bootstrap token. It previews **10.5 (node provisioning: PXE →
image → firmware)**, which is the physical layer a bare-metal provider actually drives: this lesson
told you *that* Metal3 owns BMC, netboot and imaging; 10.5 shows you *how*, stage by stage. It
connects sideways to **10.6 (hardware health, remediation, RMA)**, where `MachineHealthCheck`
turns the `Machine` object into the hook for automated repair — and where the
`nodeDrainTimeoutSeconds` field you set here decides whether a 500-GPU training job survives a
rollout. And it sharpens the capstone, **10.8 (capex-vs-cloud)**: the argument "owning beats
renting past N GPUs" only holds if you can operate N machines without linearly scaling headcount,
which is exactly what this lesson's crossover math quantifies.

Next: **[10.5 · Node provisioning: PXE → image → firmware](05-node-provisioning-pxe.md)** goes one
layer below this lesson, into the netboot-to-Ready pipeline that a bare-metal CAPI provider
executes on physical hardware — the stage where firmware, BIOS and disk imaging meet the reconcile
loop you just watched run in CAPD.

## References & further reading

**Primary sources**

- **`kubernetes-sigs/cluster-api`, main branch (release series v1.14, contract v1beta2, Aug 2026)**
  — <https://github.com/kubernetes-sigs/cluster-api> — read directly and the authority for this
  lesson's object model and controller behaviour: `api/core/v1beta2/machine_phase_types.go` (the
  phase set, including `Updating` and the deprecation of `Failed`),
  `api/core/v1beta2/common_types.go` (`ContractVersionedObjectReference` and why references carry
  `apiGroup` rather than `apiVersion`), `api/core/v1beta2/*_types.go` (the printer columns quoted
  in the transcripts), `controlplane/kubeadm/reconcilers/kubeadmcontrolplane/{preflight,scale}.go`
  (the preflight conditions, the learner check, the one-operation-at-a-time design comment, and
  scale-up/scale-down), `controlplane/kubeadm/reconcilers/kubeadmcontrolplane/kubeadmcontrolplane_controller.go`
  (`ForwardEtcdLeadership` and candidate selection), `metadata.yaml` (release-series → contract
  mapping), `test/e2e/data/infrastructure-docker/main/bases/*.yaml` (the v1beta2 manifest shapes),
  and `docs/book/src/user/quick-start.md` (the CAPD commands and output in the worked example).
  **Note:** cluster-api.sigs.k8s.io is unreachable from this environment's egress proxy, so the
  rendered Cluster API Book was **not** fetched; the in-repo book source and code were used
  instead.
- **`metal3-io/baremetal-operator`, main branch** —
  <https://github.com/metal3-io/baremetal-operator> — read directly for the `BareMetalHost` API and
  state machine: `apis/metal3.io/v1alpha1/baremetalhost_types.go` (every `ProvisioningState`
  constant and error type quoted in the diagram) and `examples/` (the BMH manifest shape,
  `bootMACAddress`, `bmc.address`, credentials Secret). metal3.io itself is blocked by the egress
  proxy and was **not** used.
- **`metal3-io/cluster-api-provider-metal3`, main branch** —
  <https://github.com/metal3-io/cluster-api-provider-metal3> — the `examples/{cluster,controlplane,machinedeployment}/`
  templates this lesson's annotated manifest is adapted from, including the v1beta2
  `KubeadmControlPlane` shape with `rollout.strategy`, `deletion.nodeDrainTimeoutSeconds`, and the
  `Metal3Cluster`/`Metal3MachineTemplate`/`Metal3DataTemplate` trio.
- **`siderolabs/talos`, branch `release-1.14`** — <https://github.com/siderolabs/talos> — read
  directly for: `pkg/machinery/constants/constants.go` (partition labels EFI/BIOS/BOOT/META/STATE/
  EPHEMERAL, their mount points, `ApidPort = 50000`),
  `internal/app/machined/pkg/runtime/v1alpha1/bootloader/grub/constants.go` (the `A`/`B`/`Reset`
  boot labels), `pkg/machinery/config/types/v1alpha1/v1alpha1_types.go` (the machine/cluster config
  field set, including the list of etcd flags Talos refuses to let you override), and
  `website/content/v1.14/reference/{cli,configuration}` (the `talosctl upgrade` flags and defaults
  quoted above). talos.dev is blocked by the egress proxy and was **not** used.
- **`siderolabs/cluster-api-control-plane-provider-talos`, main branch** —
  <https://github.com/siderolabs/cluster-api-control-plane-provider-talos> — read for the
  `TalosControlPlane` API (`api/v1alpha3/taloscontrolplane_types.go`) and the `generateType:
  controlplane | none` mechanism shown above.
- **The Cluster API Book, Talos documentation, and metal3.io** — <https://cluster-api.sigs.k8s.io/>,
  <https://www.talos.dev/latest/>, <https://metal3.io/> — **not relied upon**: all three are
  blocked by this environment's egress proxy and were not fetched. They are the readable companions
  to the source trees above; consult them with unrestricted network access, but every field name,
  default and state in this lesson came from the code.

**Real-world engineering blogs**

- **Deutsche Telekom — Das Schiff** — <https://github.com/telekom/das-schiff> — what it shows:
  CAPI + Metal3 + Flux at hundreds of German locations, with edge sites that stay operable when
  the management cluster is unreachable.
- **Sylva Project** — <https://sylvaproject.org/ocudu-on-telco-clouds-0-introduction-to-sylva/>
  and <https://the-mobile-network.com/2022/11/why-the-eu-big-five-are-launching-sylva/> — what
  they show: five European carriers jointly building a telco cloud stack explicitly around Cluster
  API, Metal3 and Flux — independent corroboration of the pattern.
- **Sidero Labs — Equinix switches from KubeSpray to Talos Linux** —
  <https://www.siderolabs.com/case-studies/equinix-switches-from-kubespray-to-talos-linux-cutting-deployment-time-while-maintaining-security>
  — what it shows: a numbers-backed migration off a mutable Ansible-driven OS onto Talos, cutting
  cluster deploy time from ~45 minutes to under 10, and the operational-burden story behind it.
- **Metal3.io becomes a CNCF Incubating project (CNCF blog, 27 Aug 2025)** —
  <https://www.cncf.io/blog/2025/08/27/metal3-io-becomes-a-cncf-incubating-project/> — what it
  shows: the incubation date, the 57 contributing organizations, and the named production adopters
  cited above.

**Deeper dives**

- **Tinkerbell** — <https://github.com/tinkerbell> — the workflow-engine alternative to Ironic
  (Smee, Tink, Hegel, Rufio, HookOS) and its CAPI provider CAPT. Read it for the contrast between
  *workflow-as-data* and Metal3's *declarative host state*; the choice between them is an
  operational-philosophy choice, and seeing both makes each one legible.
- **Talos Image Factory** — <https://factory.talos.dev/> — schematic-based image building with
  system extensions (including NVIDIA drivers and container toolkit) baked in. The practical
  mechanism behind this lesson's "immutable images remove the 6-minute GPU stage" arithmetic, and
  the on-ramp to 10.5's hands-on netboot path.
