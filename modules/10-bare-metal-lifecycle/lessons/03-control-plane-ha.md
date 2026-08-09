---
lesson: "10.3"
title: "Control-plane HA"
module: "10"
concept: "Control-plane HA"
status: not-started
est_time: "6h"
artifacts: []
---
# 10.3 · Control-plane HA

> **Concept.** A self-managed control plane survives node loss only if you build the redundancy the cloud used to hand you for free: an odd etcd quorum, a load-balanced apiserver behind a VIP, and an upgrade order that never violates version skew.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Why this matters

On EKS/GKE the control plane is a billed black box. AWS runs 3+ apiserver
replicas across AZs, a managed etcd, an NLB in front, and rolling upgrades — you
click a version number and it happens. That SLA (99.95% on EKS) is a line item,
not something you operate.

At a GPU neocloud you own the control plane. If it goes down, the *workloads*
keep running (kubelets already have their pod specs) but you lose the ability to
schedule, scale, recover from node failure, or serve `kubectl` — which means no
GPU job placement, no autoscaling of a 400-node training fleet, no draining a
node with a failing NVLink. A single-node control plane on bare metal is a
career-limiting outage waiting for a reboot. This lesson is the difference
between "I ran KTHW once" and "I operate a control plane other teams depend on."
It also feeds the cost story: control-plane HA is 3 always-on nodes you must
capitalize and justify against the managed fee in your capex-vs-cloud model.

## What's new here (managed → self-managed)

| Concern | Managed (EKS/GKE) | Self-managed (yours) |
|---|---|---|
| apiserver replicas | Hidden, auto | You run N and load-balance them |
| etcd | Managed, backed up | You choose topology, back up, restore |
| Endpoint stability | Cloud DNS/NLB | You provide a VIP (kube-vip/keepalived) |
| Upgrades | One API call, rolling | You sequence apiserver→CM/sched→kubelet by hand |
| Quorum math | Invisible | You own it: lose quorum, lose writes |

The mental model from **K8s-controllers 02** still holds: apiserver +
controller-manager + scheduler + etcd. What changes is that *placement,
redundancy, and the endpoint clients talk to* are now your responsibility. The
managed layer was mostly solving three problems you must now solve explicitly:
etcd quorum, a stable apiserver endpoint, and skew-safe upgrades.

## Core notes

### 1. Two HA topologies

**Stacked etcd** — each control-plane node runs its own etcd member co-located
with the apiserver. 3 nodes = 3 etcd members = quorum of 2. This is
kubeadm's default (`--upload-certs`, `--control-plane`). Simplest, fewest
machines.

**External etcd** — etcd runs on a separate dedicated cluster (typically 3 or 5
nodes), and the apiservers point at it via `--etcd-servers`. More machines, more
operational surface, but the etcd failure domain is decoupled from the apiserver
failure domain.

**Blast radius — the thing that actually matters:**

- *Stacked:* losing a control-plane node loses **both** an apiserver **and** an
  etcd member at once. With 3 stacked nodes you tolerate exactly **1**
  simultaneous node loss (2/3 quorum survives). Lose 2 nodes and etcd loses
  quorum → the store goes **read-only**, no writes, cluster is effectively frozen
  until you recover a member. Correlated failure: a rack/power/AZ event that
  takes 2 nodes kills both planes together.
- *External:* an apiserver node dying does **not** touch etcd quorum, and vice
  versa. You can lose control-plane (apiserver) nodes down to 1 and still serve
  the API as long as the external etcd cluster holds quorum. The tradeoff is 3–5
  extra machines and a second thing to patch, monitor, and back up.

Rule of thumb: **stacked for most on-prem clusters** (3 nodes, spread across
racks/failure domains); **external etcd** when etcd write load is heavy (very
large clusters, high object churn), when you want to scale/patch etcd
independently, or when compliance wants the datastore isolated.

Quorum is always `floor(N/2)+1`. Use **odd** counts (3 or 5). 3 tolerates 1
failure; 5 tolerates 2. Never run 2 or 4 — even counts add a member without
adding fault tolerance and *raise* the odds of a split.

### 2. Load-balancing the apiserver

Every apiserver is stateless and interchangeable (they all talk to the same
etcd). Clients (kubelets, controller-manager, `kubectl`, other CP nodes) need
**one stable endpoint** that fans out across the live apiservers on `:6443`.

Options:
- **External LB** (F5, an on-prem HAProxy pair, MetalLB isn't for this) — fine if
  you already run one, but it's another box and often another team.
- **VIP + local proxy (kube-vip / keepalived)** — a virtual IP that floats to a
  live control-plane node, self-hosted on the CP nodes themselves. No external
  dependency. This is the bare-metal default and what you'll build.

The endpoint goes in the kubeadm config as `controlPlaneEndpoint:
"<VIP>:6443"`. Set it **before** `kubeadm init` — you cannot cleanly retrofit a
controlPlaneEndpoint onto certs that were minted for a single node's IP without
regenerating them.

### 3. What the VIP actually gives you (kube-vip / keepalived / VRRP)

A control-plane VIP provides **one thing: a stable, highly-available IP for the
apiserver endpoint.** Mechanism:

- **VRRP (keepalived, and kube-vip in ARP/leader mode):** the CP nodes elect a
  leader that "owns" the VIP and answers ARP for it. On leader failure, another
  node claims the VIP (gratuitous ARP) within seconds. Layer-2 adjacency
  required (same subnet).
- **kube-vip** can do the above *and* optionally load-balance across apiservers
  (rather than pin the VIP to one node), and can run as a static pod on each CP
  node — no separate daemon. It can also later serve `type: LoadBalancer` for
  workloads, but for the control plane its job is the VIP.

What the VIP does **not** give you: it is not etcd quorum, not replication, not
data safety. It only makes sure that when a client dials the apiserver, the
packet reaches a *live* apiserver. HA of the API endpoint ≠ HA of the datastore —
you need both, independently.

### 4. Version skew and the upgrade order

Kubernetes guarantees interoperation only within a bounded version window. The
**current policy (v1.28+):**

- **kube-apiserver** is the reference. In an HA cluster, apiservers may differ
  from each other by **at most 1 minor** (during a rolling upgrade).
- **kube-controller-manager, kube-scheduler, cloud-controller-manager:** may be
  **up to 1 minor behind** the apiserver they talk to (and not newer).
- **kubelet:** may be **up to 3 minors behind** the apiserver (widened from 2 in
  v1.28). Never newer than the apiserver.
- **kube-proxy:** must match its node's kubelet minor (within the same 3-minor
  window), and not be newer than the apiserver.
- **kubeadm** must match the *target* control-plane version you're moving to.

Consequences that drive the order:

1. **Upgrade the control plane first, apiserver leads.** Within a CP node,
   kubeadm brings up the new apiserver, then controller-manager and scheduler —
   because CM/sched must be ≤ apiserver, never ahead.
2. **Then upgrade kubelets**, node by node (drain → upgrade kubelet + kube-proxy
   → uncordon). Because kubelet may lag the apiserver by 3 minors, you are never
   forced to touch every node in the same window — you can ride kubelet 3 behind.
3. **One minor at a time.** No jumping 1.30 → 1.32 in a single hop; skew and
   upgrade paths are only validated across adjacent minors.

The 3-minor kubelet window is the operationally valuable fact: on a 400-node GPU
fleet you can upgrade the control plane on a tight cadence and roll node upgrades
lazily (as you drain for other reasons), instead of a synchronized fleet-wide
kubelet bump.

### 5. kubeadm upgrade mechanics

- `kubeadm upgrade plan` — shows current/target versions and component skew.
- `kubeadm upgrade apply v1.X.Y` — on the **first** CP node, upgrades the static
  pod manifests (apiserver, CM, scheduler) and, if stacked, the etcd member.
- `kubeadm upgrade node` — on **every other** CP node (and worker nodes) to pick
  up the new component/kubelet config.
- Then on each node, `apt/yum` upgrade the `kubelet` + `kubeadm` packages and
  `systemctl restart kubelet`.
- Always **drain** a node (`kubectl drain --ignore-daemonsets`) before touching
  its kubelet; **uncordon** after. Snapshot etcd *before* you start.

## Worked example — grow a single-node CP to 3-node HA + one upgrade

Starting point: the KTHW/kubeadm single control-plane node from Lesson 1.
Assume subnet-local nodes `cp1/cp2/cp3` and a free VIP `10.10.0.100`.

**1. Put kube-vip in front (static pod on cp1).** Generate the manifest and drop
it in `/etc/kubernetes/manifests/`:

```bash
export VIP=10.10.0.100 INTERFACE=eth0 KVVERSION=v0.8.0
ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip \
  /kube-vip manifest pod \
  --interface $INTERFACE --address $VIP --controlplane \
  --arp --leaderElection \
  > /etc/kubernetes/manifests/kube-vip.yaml
```

If you're bootstrapping fresh, set `controlPlaneEndpoint: "10.10.0.100:6443"` in
the kubeadm `ClusterConfiguration` and `kubeadm init --upload-certs`. On an
existing single node you must have initialized with a controlPlaneEndpoint
already; if not, this is why we set it up front.

**2. Join cp2 and cp3 as control-plane nodes** (from `kubeadm init`'s join
output, `--control-plane --certificate-key ...`):

```bash
kubeadm join 10.10.0.100:6443 --token <t> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>
```

Now `kubectl get nodes` shows 3 control-plane nodes; `kubectl -n kube-system get
pods` shows 3 apiservers, 3 etcd members (stacked). Verify quorum:

```bash
kubectl -n kube-system exec etcd-cp1 -- etcdctl \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key  /etc/kubernetes/pki/etcd/server.key \
  member list -w table
```

**3. Kill the leader, prove continuity.** Find which node holds the VIP (`ip a |
grep 10.10.0.100`), then hard-power/`reboot` it. Within seconds kube-vip
re-elects and the VIP moves; `kubectl get nodes` from your workstation keeps
working after a brief blip. etcd goes 3→2 members = quorum holds, writes still
succeed. Bring the node back and confirm it rejoins (`member list` shows 3
started).

**4. One-minor upgrade respecting skew** (e.g. 1.30 → 1.31):

```bash
# cp1 (first CP node)
apt-mark unhold kubeadm && apt-get install -y kubeadm=1.31.*-* && apt-mark hold kubeadm
kubeadm upgrade plan
kubeadm upgrade apply v1.31.0        # apiserver+CM+sched(+stacked etcd) on cp1
# cp2, cp3
kubeadm upgrade node                  # each other CP node
# every node (CP + workers), one at a time:
kubectl drain cpX --ignore-daemonsets
apt-mark unhold kubelet && apt-get install -y kubelet=1.31.*-* && apt-mark hold kubelet
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon cpX
```

Order enforced: apiserver (via `upgrade apply`) → CM/scheduler (same step, stay
≤ apiserver) → kubelets last. You could legally leave kubelets on 1.30 (or even
1.28) since kubelet may trail the apiserver by 3 minors — but here we finish the
hop.

## Practice (hands-on, cheap VMs → deliverable)

Use 3 small VMs (Multipass, Vagrant/libvirt, or 3 cloud VMs in one subnet; 2
vCPU / 2–4 GB each is enough).

1. Take the Lesson-1 kubeadm cluster and, if needed, rebuild it with
   `controlPlaneEndpoint: "<VIP>:6443"`.
2. Deploy **kube-vip** as a static pod; verify the VIP answers and `kubectl` works
   through it.
3. **Join a 2nd and 3rd control-plane node** (`--control-plane
   --certificate-key`). Confirm 3 apiservers + 3 stacked etcd members with
   `etcdctl member list -w table`.
4. **Kill the VIP-holding leader** (reboot or `poweroff`). Time how long
   `kubectl get nodes` is unavailable; confirm it recovers and etcd keeps quorum
   (2/3).
5. **`kubeadm upgrade`** one minor version across the fleet in the correct order.
   Capture `kubeadm upgrade plan` output before and `kubectl get nodes` (VERSION
   column) after.

**Acceptance (feeds the capex-vs-cloud writeup):** a **3-node HA control plane
behind a VIP**, plus a short documented runbook containing (a) `etcdctl member
list -w table` showing 3 members, (b) evidence the API survived leader loss
(timestamps / continued `kubectl`), and (c) the before/after upgrade versions
with the order you followed and the skew rule you relied on. This is the
"control plane is yours" section of the deliverable — one that a managed cluster
would never let you write.

## Self-check

**(a) In what order do you upgrade apiserver / controller-manager / kubelet, and
what's the version-skew constraint?**
**Answer:** apiserver **first** (control plane leads), then
controller-manager/scheduler (kubeadm does these in the same `upgrade apply`
step, and they must be ≤ apiserver, never ahead), then **kubelets last**, node by
node with drain/uncordon. Skew rules: CM/scheduler may be at most **1 minor
behind** the apiserver; kubelet may be up to **3 minors behind** (v1.28+); nothing
may be newer than the apiserver; upgrade **one minor at a time**.

**(b) Stacked-etcd vs external-etcd — what's the failure blast radius of each?**
**Answer:** *Stacked:* each CP node holds an apiserver **and** an etcd member, so
losing one node loses both at once. With 3 stacked nodes you tolerate exactly **1**
node loss; losing 2 breaks etcd quorum → the datastore goes read-only and the
cluster freezes. Correlated (rack/power) failures are the danger. *External:* etcd
runs on its own cluster, so an apiserver-node failure doesn't touch etcd quorum
and vice-versa; the two failure domains are decoupled at the cost of 3–5 extra
machines to run, patch, and back up.

**(c) What does the control-plane VIP (kube-vip/keepalived) actually provide?**
**Answer:** A single **stable, highly-available IP for the apiserver endpoint**.
Via VRRP/ARP it floats to a live control-plane node on leader failure (gratuitous
ARP, seconds) so clients always reach a running apiserver. It is *only* endpoint
HA — it does **not** provide etcd quorum, replication, or data safety. You need
apiserver-endpoint HA and datastore HA as two independent properties.

## Resources

1. **kubeadm HA topology (do this first, it maps 1:1 to the practice):**
   https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
2. **Version-skew policy (the exact numbers you must not violate):**
   https://kubernetes.io/releases/version-skew-policy/
3. **kube-vip (VIP mechanism, static-pod manifest generation):**
   https://kube-vip.io/
