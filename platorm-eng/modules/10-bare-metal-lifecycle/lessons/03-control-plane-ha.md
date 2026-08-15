---
lesson: "10.3"
title: "Control-plane HA & upgrades"
module: "10"
concept: "Control-plane HA & upgrades"
status: not-started
est_time: "8h"
prev: "02-etcd-operations.md"
next: "04-declarative-fleets-capi-talos.md"
artifacts: []
sources: 8
---
# 10.3 · Control-plane HA & upgrades

> **Concept.** A self-managed control plane survives node loss only if you build the redundancy the cloud used to hand you for free: an odd etcd quorum, a load-balanced apiserver behind a stable endpoint, active-passive leader election for the singleton controllers, and an upgrade order that never violates version skew.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 10.2 made etcd's disk, quorum, and disaster-recovery runbook yours — you
now know how to keep the **datastore** alive and how to reason about `N` and
quorum. That lesson deliberately stayed inside a single node's etcd process:
compaction, defrag, snapshot/restore. What it didn't cover is the layer that
*sits on top* of a healthy etcd: how do multiple control-plane nodes present
themselves as **one** control plane to the rest of the cluster, and how do you
move that whole assembly from one Kubernetes minor version to the next without
ever serving an API that violates the version-skew contract? That's this
lesson. By the end you can grow a single hand-built node (Lesson 10.1) into a
3-node HA control plane behind a virtual IP, and walk it through a live minor
upgrade — the exact shape the module's checkpoint asks you to defend.

## Why this matters

On EKS/GKE the control plane is a billed black box. AWS runs 3+ apiserver
replicas across availability zones, a managed etcd, a network load balancer in
front, and rolling upgrades — you click a version number and it happens. That
SLA (99.95% for a multi-AZ EKS cluster) is a line item, not something you
operate.

At a GPU neocloud you own the control plane. If it goes down, the *workloads*
keep running for a while (kubelets already hold their pod specs and containerd
keeps existing containers alive) but you lose the ability to schedule, scale,
recover from node failure, or serve `kubectl` — which means no GPU job
placement, no autoscaling a 400-node training fleet, no draining a node with a
failing NVLink, no responding to an XID event. A single-node control plane on
bare metal is a career-limiting outage waiting for a routine reboot. This
lesson is the difference between "I ran KTHW once" and "I operate a control
plane other teams depend on" — which is precisely the bar in job postings like
CoreWeave's "day-2 lifecycle, fault-tolerant architectures" and NVIDIA's
node-lifecycle roles (see the module README). It also feeds the cost story
directly: control-plane HA is 3 always-on nodes you must capitalize and
justify against the managed-control-plane fee in your capex-vs-cloud model —
the checkpoint's economics probe assumes you can name that line item.

## What's new here (calibration)

You already know the components from **K8s-controllers 02**: apiserver,
controller-manager, scheduler, etcd, and the reconcile-loop mental model. You
also now own etcd's internals from Lesson 10.2. What's genuinely new here:

- **Multi-node placement and endpoint stability** — turning "3 apiservers" into
  "one address clients can dial," which the cloud solved with an NLB you never
  saw.
- **The stacked-vs-external etcd topology decision** and its blast-radius math
  (etcd quorum from 10.2, now combined with apiserver placement).
- **VIP/VRRP mechanics** (kube-vip, keepalived) — a genuinely new primitive if
  you've never run on-prem L2 failover, plus its L2-vs-BGP scaling limit.
- **Leader election via the Lease API** for controller-manager and scheduler —
  why those two are active-passive while the apiserver is active-active, a
  distinction managed Kubernetes hides completely.
- **The exact, memorizable version-skew upgrade order** — not "upgrade
  everything," but a specific sequence with specific numeric bounds you must
  reproduce cold at the checkpoint.

We do **not** re-derive etcd's Raft quorum math here (that's 10.2) or re-teach
what apiserver/CM/scheduler/etcd each *do* (that's K8s-controllers 02) — we
build the redundancy and operational sequencing around them.

## Core concepts

### 1. Two HA topologies

**Stacked etcd** — each control-plane node runs its own etcd member co-located
with the apiserver. 3 nodes = 3 etcd members = quorum of 2. This is kubeadm's
default (`--upload-certs`, `--control-plane`). Simplest, fewest machines.

**External etcd** — etcd runs on a separate dedicated cluster (typically 3 or
5 nodes), and the apiservers point at it via `--etcd-servers`. More machines,
more operational surface, but the etcd failure domain is decoupled from the
apiserver failure domain.

**Blast radius — the thing that actually matters:**

- *Stacked:* losing a control-plane node loses **both** an apiserver **and**
  an etcd member at once. With 3 stacked nodes you tolerate exactly **1**
  simultaneous node loss (2/3 quorum survives — see 10.2's quorum table for
  why). Lose 2 nodes and etcd loses quorum → the store goes **read-only**, no
  writes, cluster is effectively frozen until you recover a member. Correlated
  failure: a rack/power event that takes 2 nodes kills both planes together.
- *External:* an apiserver node dying does **not** touch etcd quorum, and vice
  versa. You can lose control-plane (apiserver) nodes down to 1 and still
  serve the API as long as the external etcd cluster holds quorum. The
  tradeoff is 3–5 extra machines and a second thing to patch, monitor, and
  back up (a second application of everything in 10.2).

Rule of thumb: **stacked for most on-prem clusters** (3 nodes, spread across
racks/failure domains); **external etcd** when etcd write load is heavy (very
large clusters, high object churn), when you want to scale/patch etcd
independently, or when compliance wants the datastore isolated. Quorum is
always `floor(N/2)+1`; use **odd** counts (3 or 5) — see 10.2 for the full
derivation of why even counts never help.

### 2. Load-balancing the apiserver

Every apiserver is stateless and interchangeable — they all talk to the same
etcd, and any one of them can serve any request. Clients (kubelets,
controller-manager, `kubectl`, other CP nodes) need **one stable endpoint**
that fans out across the live apiservers on `:6443`.

| Option | Mechanism | When you'd pick it |
|---|---|---|
| External LB (F5, on-prem HAProxy pair) | A box or pair outside the CP nodes proxies `:6443` | You already run one, or want the LB failure domain fully separate from CP nodes |
| VIP + local static pod (kube-vip) | A floating IP, self-hosted on the CP nodes, via VRRP/ARP or BGP | The bare-metal default — no extra hardware, no extra team |
| DNS round-robin | Multiple A records for one name | **Don't.** Clients cache resolutions and don't retry on a dead IP fast enough; no health awareness |

A minimal self-managed HAProxy alternative (useful when you want the LB
off the CP nodes, e.g. one HAProxy+keepalived pair per rack):

```
# /etc/haproxy/haproxy.cfg — fronts 3 apiservers on :6443
frontend k8s-api
    bind *:6443
    mode tcp
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server cp1 10.10.0.11:6443 check
    server cp2 10.10.0.12:6443 check
    server cp3 10.10.0.13:6443 check
```

Pair that HAProxy with keepalived (VRRP) so the HAProxy instance itself is
HA — which is exactly the pattern kube-vip collapses into a single static pod
per CP node, no separate boxes required. That collapse (LB + VIP in one
lightweight process, co-located on the nodes it balances) is why kube-vip is
the default reach-for on bare metal rather than the classic
HAProxy+keepalived pair.

The endpoint goes in the kubeadm config as `controlPlaneEndpoint:
"<VIP>:6443"`. Set it **before** `kubeadm init` — you cannot cleanly retrofit
a controlPlaneEndpoint onto certs that were minted for a single node's IP
without regenerating them (the apiserver's serving cert SANs are fixed at
generation time).

### 3. What the VIP actually gives you — VRRP, ARP, and BGP modes

A control-plane VIP provides **one thing: a stable, highly-available IP for
the apiserver endpoint.**

- **VRRP (RFC 5798)** is the protocol underneath keepalived and kube-vip's
  default mode: control-plane nodes participate in a virtual router
  election, the elected leader "owns" the VIP and answers ARP for it with its
  own MAC. On leader failure, another node detects the missed VRRP
  advertisements, claims the VIP, and sends a **gratuitous ARP** so the local
  switch updates its MAC table — typically a multi-second failover, tunable
  via the advertisement interval. This requires **Layer-2 adjacency**: every
  CP node must be on the same broadcast domain/subnet, because ARP doesn't
  route.
- **kube-vip in BGP mode** sidesteps the L2 requirement: instead of ARP, each
  CP node peers with the top-of-rack switch(es) over BGP and advertises the
  VIP as a `/32` route; the switch fabric handles reachability via ECMP/route
  withdrawal instead of ARP. This is the same mechanism MetalLB uses for
  `LoadBalancer` Services (see the Red Hat use case below and Lesson 10.8) —
  and it's the only option once your control-plane nodes span **multiple
  racks or L3 boundaries**, which a growing GPU fleet eventually does. Plain
  ARP-mode VIPs do not survive an L3 hop.
- **kube-vip** can also run purely as a **static pod** on each CP node (no
  separate daemon, no extra machine) and can later serve `type: LoadBalancer`
  for workloads — but for the control plane its job here is strictly the VIP.

What the VIP does **not** give you: it is not etcd quorum, not replication,
not data safety. It only makes sure that when a client dials the apiserver,
the packet reaches a *live* apiserver. HA of the API endpoint ≠ HA of the
datastore — you need both, independently (10.2 gives you the datastore half).
The kube-vip GitHub issue in Real-world use cases below shows exactly how this
assumption breaks when the *entire* control plane — not just the leader —
goes away.

### 4. Leader election for the singleton controllers (the Lease API)

The apiserver is **active-active**: every replica independently serves
requests, and the VIP/LB simply routes to whichever is up. Controller-manager
and scheduler are different — they must be **active-passive**, because two
schedulers binding the same Pod concurrently, or two controller-managers
racing to reconcile the same object, would corrupt state. Kubernetes solves
this with the **Lease API** (`coordination.k8s.io/v1`, `kube-system`
namespace): each replica of `kube-controller-manager` and `kube-scheduler`
runs with `--leader-elect=true` and repeatedly tries to acquire/renew a
`Lease` object. The instance holding the lease is active; the others sit idle,
watching the lease, ready to take over the instant it's not renewed in time.
This is the same Lease primitive kubelets use for node heartbeats — one
generic coordination object, two very different uses.

Consequence for you as an operator: `kubectl -n kube-system get lease
kube-controller-manager kube-scheduler` tells you which CP node is actually
doing the work at any moment — useful when triaging "why didn't this Pod get
scheduled" on a 3-node CP where two schedulers are deliberately idle.

### 5. Version skew and the upgrade order

Kubernetes guarantees interoperation only within a bounded version window.
The **current policy (v1.28+):**

| Component | Skew bound relative to apiserver |
|---|---|
| kube-apiserver (HA peers) | at most 1 minor apart from each other during a rolling upgrade |
| kube-controller-manager / kube-scheduler / cloud-controller-manager | up to **1 minor behind**, never newer |
| kubelet | up to **3 minors behind** (widened from 2 in v1.28), never newer |
| kube-proxy | must match its node's kubelet minor, never newer than the apiserver |
| kubeadm | must match the *target* control-plane version |

Consequences that drive the order:

1. **Upgrade the control plane first, apiserver leads.** Within a CP node,
   kubeadm brings up the new apiserver, then controller-manager and
   scheduler — because CM/sched must be ≤ apiserver, never ahead. This
   ordering exists so that no component ever calls an API that hasn't been
   registered yet on the server it's programmed against.
2. **Then upgrade kubelets**, node by node (drain → upgrade kubelet +
   kube-proxy → uncordon). Because kubelet may lag the apiserver by 3 minors,
   you are never forced to touch every node in the same window — you can ride
   kubelet 3 behind.
3. **One minor at a time.** No jumping 1.30 → 1.32 in a single hop; skew and
   upgrade paths are only validated across adjacent minors, because each hop
   may carry API deprecations/removals that assume you passed through the
   intermediate minor's warnings first.

The 3-minor kubelet window is the operationally valuable fact: on a 400-node
GPU fleet you can upgrade the control plane on a tight cadence and roll node
upgrades lazily (as you drain for other reasons — hardware remediation from
Lesson 10.6, firmware from 10.5), instead of a synchronized fleet-wide kubelet
bump that would force 400 simultaneous drains.

### 6. kubeadm upgrade mechanics and rollback posture

- `kubeadm upgrade plan` — shows current/target versions and component skew;
  run this first, always, and read every line — it also lists which
  CRDs/APIs are being deprecated in the target version.
- `kubeadm upgrade apply v1.X.Y` — on the **first** CP node, upgrades the
  static pod manifests (apiserver, CM, scheduler) and, if stacked, the etcd
  member.
- `kubeadm upgrade node` — on **every other** CP node (and later, worker
  nodes) to pick up the new component/kubelet config.
- Then on each node: `apt/yum` upgrade the `kubelet` + `kubeadm` packages and
  `systemctl restart kubelet`.
- Always **drain** a node (`kubectl drain --ignore-daemonsets`) before
  touching its kubelet; **uncordon** after. **Snapshot etcd before you
  start** — this is the exact Lesson 10.2 discipline applied here: an
  upgrade is a state-changing operation, and a fresh, verified snapshot is
  your rollback path if `kubeadm upgrade apply` corrupts static pod manifests
  or a component fails to come back healthy.
- Rollback in practice means: restore the etcd snapshot taken pre-upgrade
  (10.2's DR runbook) and reinstall the prior kubeadm/kubelet package
  versions — kubeadm has no built-in "undo," so the snapshot *is* the undo
  button.

### 7. Failure-domain design (don't let the VIP hide a correlated failure)

HA only works if the 3 (or 5) CP nodes can't fail together. Spread them
across:

- **Racks / PDUs** — a single PDU or ToR switch failure should never be able
  to take 2+ CP nodes at once (which is exactly the "lose quorum" scenario
  from §1).
- **BMC/management network segments** — so a management-network incident
  doesn't blind you to all CP nodes simultaneously.
- If you're on **BGP-mode** kube-vip, this is also where multi-rack CP
  placement becomes *possible in the first place* — ARP-mode VIPs constrain
  you to one L2 domain, which often means one rack.

## Perspectives

**Developer.** To someone running `kubectl apply` against the cluster, a
correctly built control plane is invisible — they dial one IP, and HA is a
property they never think about. The only symptom of a well-executed
failover is a request that takes an extra second or two, not an error. If
they *see* the control plane, something already went wrong upstream of them.

**Operator.** This is the layer you own and the layer that pages you. The
job is sequencing: never let controller-manager get ahead of the apiserver,
never drain two CP nodes at once, never skip the pre-upgrade etcd snapshot.
Most control-plane HA incidents are self-inflicted — an upgrade run out of
order, or a "just reboot both to save time" shortcut that briefly drops
quorum. The runbook discipline from 10.2 (snapshot, verify, rehearse) extends
one level up to cover the whole upgrade, not just etcd disasters.

**Hardware / network.** HA is a network-topology problem as much as a
software one. VRRP/ARP-mode VIPs need L2 adjacency — plan your rack/subnet
layout with that constraint up front, or you'll be re-cabling to fix an
"HA" design that can't actually place nodes across failure domains. BGP-mode
VIPs trade that constraint for a dependency on your ToR switches speaking
BGP correctly — a fleet-scale design decision, not a per-cluster one.

**Economics.** Three always-on control-plane nodes are pure overhead — they
run no GPU workloads and generate no revenue, yet must be capitalized,
racked, powered, and kept patched exactly like the fleet they manage. In the
capex-vs-cloud model this is the line item that offsets some of your "we
don't pay AWS's control-plane fee" savings; a correctly-sized 3-node
CP (not 5, unless your object churn genuinely needs it — see 10.2) is the
efficient choice for most fleets.

## Real-world use cases

- **Red Hat — "Deploying a high-availability, fault-tolerant Kubernetes
  Service on bare metal clusters with MetalLB BGP"**
  <https://www.redhat.com/en/blog/deploying-a-high-availability-fault-tolerant-kubernetes-service-on-baremetal-clusters-with-metallb-bgp> —
  a vendor-documented reference architecture combining MetalLB's BGP
  advertisement mode with an HA bare-metal cluster. It's aimed at
  workload/Service load-balancing (the topic of Lesson 10.8), but it shows
  the *identical* VIP-over-BGP mechanism this lesson uses for the
  control-plane endpoint — worth reading as a practice architecture
  cross-check for the BGP-mode VIP story, without duplicating the Service-LB
  material that belongs in 10.8.
- **kube-vip — GitHub issue #732, "VIP is lost when kubernetes control plane
  goes down"** <https://github.com/kube-vip/kube-vip/issues/732> — a
  concrete, real bug report: in an RKE2 cluster running kube-vip with leader
  election, the control-plane VIP became unreachable from all remaining
  worker nodes roughly 5 minutes after the sole control-plane node died,
  instead of the VIP staying reachable indefinitely on a healthy worker.
  It's precise grounding for §3's point: a VIP mechanism assumes *some*
  control-plane node is alive to hold the election — it is endpoint HA
  built on the CP's own health, not a guarantee that survives every failure
  shape you can imagine.
- **CoreWeave — node lifecycle at neocloud scale**
  <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference> —
  CoreWeave's own framing (also quoted in this module's README) that a
  faulty node can go undetected for up to a month without automated
  lifecycle management. The full remediation story is Lesson 10.6's; the
  one-line relevance here is that **HA control-plane design and fast
  hardware fault response are the same operational muscle** at a neocloud —
  both are about never letting a single piece of hardware become a single
  point of failure for the fleet.

## Worked example — grow a single-node CP to 3-node HA + one upgrade

Starting point: the KTHW/kubeadm single control-plane node from Lesson 10.1.
Assume subnet-local nodes `cp1/cp2/cp3` and a free VIP `10.10.0.100`.

**1. Put kube-vip in front (static pod on cp1).** Generate the manifest and
drop it in `/etc/kubernetes/manifests/`:

```bash
export VIP=10.10.0.100 INTERFACE=eth0 KVVERSION=v0.8.0
ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip \
  /kube-vip manifest pod \
  --interface $INTERFACE --address $VIP --controlplane \
  --arp --leaderElection \
  > /etc/kubernetes/manifests/kube-vip.yaml
```

(Swap `--arp` for `--bgp` plus peer config if your CP nodes span racks — see
§3.) If you're bootstrapping fresh, set `controlPlaneEndpoint:
"10.10.0.100:6443"` in the kubeadm `ClusterConfiguration` and `kubeadm init
--upload-certs`. On an existing single node you must have initialized with a
controlPlaneEndpoint already; if not, this is why we set it up front.

**2. Join cp2 and cp3 as control-plane nodes** (from `kubeadm init`'s join
output, `--control-plane --certificate-key ...`):

```bash
kubeadm join 10.10.0.100:6443 --token <t> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>
```

Now `kubectl get nodes` shows 3 control-plane nodes; `kubectl -n kube-system
get pods` shows 3 apiservers, 3 etcd members (stacked). Verify quorum:

```bash
kubectl -n kube-system exec etcd-cp1 -- etcdctl \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key  /etc/kubernetes/pki/etcd/server.key \
  member list -w table
```

And check which node is actually running the singleton controllers:

```bash
kubectl -n kube-system get lease kube-controller-manager kube-scheduler -o wide
```

**3. Kill the leader, prove continuity.** Find which node holds the VIP (`ip
a | grep 10.10.0.100`), then hard-power/`reboot` it. Within seconds kube-vip
re-elects and the VIP moves; `kubectl get nodes` from your workstation keeps
working after a brief blip. etcd goes 3→2 members = quorum holds, writes
still succeed. The `kube-controller-manager`/`kube-scheduler` leases also
migrate if the leader was holding them. Bring the node back and confirm it
rejoins (`member list` shows 3 started).

**4. One-minor upgrade respecting skew** (e.g. 1.30 → 1.31):

```bash
# Take a fresh etcd snapshot first (10.2's discipline, applied here)
etcdctl snapshot save /backup/etcd-pre-upgrade.db ...

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

Order enforced: apiserver (via `upgrade apply`) → CM/scheduler (same step,
stay ≤ apiserver) → kubelets last. You could legally leave kubelets on 1.30
(or even 1.28) since kubelet may trail the apiserver by 3 minors — but here we
finish the hop across the whole fleet.

## Practice (hands-on, cheap VMs → deliverable)

Use 3 small VMs (Multipass, Vagrant/libvirt, or 3 cloud VMs in one subnet; 2
vCPU / 2–4 GB each is enough).

1. Take the Lesson 10.1 kubeadm cluster and, if needed, rebuild it with
   `controlPlaneEndpoint: "<VIP>:6443"`.
2. Deploy **kube-vip** as a static pod; verify the VIP answers and `kubectl`
   works through it.
3. **Join a 2nd and 3rd control-plane node** (`--control-plane
   --certificate-key`). Confirm 3 apiservers + 3 stacked etcd members with
   `etcdctl member list -w table`, and check which node holds the
   `kube-controller-manager`/`kube-scheduler` leases.
4. **Kill the VIP-holding leader** (reboot or `poweroff`). Time how long
   `kubectl get nodes` is unavailable; confirm it recovers and etcd keeps
   quorum (2/3). Then kill a **second** node and confirm the cluster goes
   read-only (quorum lost) — this is the blast-radius lesson from §1 made
   real.
5. **`kubeadm upgrade`** one minor version across the fleet in the correct
   order, snapshotting etcd first. Capture `kubeadm upgrade plan` output
   before and `kubectl get nodes` (VERSION column) after.

**Acceptance (feeds the capex-vs-cloud writeup):** a **3-node HA control
plane behind a VIP**, plus a short documented runbook containing (a)
`etcdctl member list -w table` showing 3 members, (b) evidence the API
survived single-node loss but froze on 2-node loss (timestamps / continued or
failed `kubectl`), and (c) the before/after upgrade versions with the order
you followed and the skew rule you relied on. This is the "control plane is
yours" section of the deliverable — one that a managed cluster would never
let you write.

## Common pitfalls

- **Setting `controlPlaneEndpoint` after `kubeadm init`.** The apiserver's
  serving-certificate SANs are fixed at generation time; retrofitting a VIP
  onto a single-node cluster means regenerating certs, not just editing a
  config file. Decide the endpoint *before* the first `kubeadm init`.
- **Believing the VIP is the whole HA story.** A VIP gives you endpoint HA,
  not data HA — see the kube-vip issue above. You still need etcd quorum
  (10.2) as an independent property, and you still need to reason about what
  happens if the node holding the election, not just the VIP, is what dies.
- **Running an even number of control-plane nodes.** Two or four CP nodes add
  a machine without adding fault tolerance (10.2's quorum table), and they
  raise the odds of an unresolvable split.
- **Upgrading kubelet before the apiserver.** A kubelet newer than the
  apiserver it talks to is explicitly unsupported and can break — the
  version-skew policy is directional (kubelet ≤ apiserver), not just a
  window.
- **Assuming ARP-mode VIPs work across racks.** VRRP/ARP requires L2
  adjacency; a VIP that "works in the lab" on one switch can silently fail
  once CP nodes are spread across racks for failure-domain reasons — that's
  exactly when you need BGP mode instead.

## Self-check

- **In what order do you upgrade apiserver / controller-manager / kubelet,
  and what's the version-skew constraint?**
  **Answer:** apiserver **first** (control plane leads), then
  controller-manager/scheduler (kubeadm does these in the same `upgrade
  apply` step, and they must be ≤ apiserver, never ahead), then **kubelets
  last**, node by node with drain/uncordon. Skew rules: CM/scheduler may be
  at most **1 minor behind** the apiserver; kubelet may be up to **3 minors
  behind** (v1.28+); nothing may be newer than the apiserver; upgrade **one
  minor at a time**.
- **Stacked-etcd vs external-etcd — what's the failure blast radius of
  each?**
  **Answer:** *Stacked:* each CP node holds an apiserver **and** an etcd
  member, so losing one node loses both at once. With 3 stacked nodes you
  tolerate exactly **1** node loss; losing 2 breaks etcd quorum → the
  datastore goes read-only and the cluster freezes. Correlated (rack/power)
  failures are the danger. *External:* etcd runs on its own cluster, so an
  apiserver-node failure doesn't touch etcd quorum and vice-versa; the two
  failure domains are decoupled at the cost of 3–5 extra machines to run,
  patch, and back up.
- **What does the control-plane VIP (kube-vip/keepalived) actually
  provide, and what does it not?**
  **Answer:** A single **stable, highly-available IP for the apiserver
  endpoint**. Via VRRP/ARP (or BGP, for multi-rack) it moves to a live
  control-plane node on leader failure so clients always reach a running
  apiserver. It is *only* endpoint HA — it does **not** provide etcd quorum,
  replication, or data safety, and (per the kube-vip issue #732 case) it
  assumes some control-plane node survives to hold the election; if the
  entire control plane the VIP depends on dies, the VIP dies with it.
- **Why are controller-manager and scheduler active-passive while the
  apiserver is active-active, and what mechanism enforces it?**
  **Answer:** The apiserver is stateless and safe to run N-way active because
  every replica just serves requests against the same etcd. Controller-manager
  and scheduler are **not** safe to run concurrently — two schedulers binding
  the same Pod, or two controllers reconciling the same object at once, would
  race and corrupt state. Kubernetes enforces single-active-instance via the
  **Lease API** (`coordination.k8s.io`): each replica runs `--leader-elect`
  and competes to hold a `Lease` object in `kube-system`; only the holder is
  active, the rest idle-watch it.
- **Your control-plane nodes are spread across 3 racks for failure-domain
  isolation. Will an ARP-mode kube-vip VIP work, and if not, what do you
  change?**
  **Answer:** Not reliably — VRRP/ARP-mode failover requires **Layer-2
  adjacency** (one broadcast domain/subnet), and 3 racks typically means 3
  separate L2 segments joined by routed (L3) links. The fix is **BGP-mode**
  kube-vip: each CP node peers with its top-of-rack switch over BGP and
  advertises the VIP as a route, so failover is a route withdrawal/insert
  handled by the switch fabric instead of an ARP update — the same mechanism
  MetalLB uses for Service load-balancing (Lesson 10.8).

## Connections & what's next

This lesson sits directly on top of **10.2 (etcd operations)** — everything
here about quorum and blast radius is 10.2's math applied to node placement,
and the pre-upgrade snapshot step is 10.2's DR discipline reused. It draws on
**K8s-controllers 02** for what each component does, and it feeds forward
into **10.5 (node provisioning)** and **10.6 (hardware health)**, whose
drain/cordon/RMA flows assume the control plane you're draining *from* is
itself HA and won't disappear mid-remediation. It also sets up **10.8
(capex-vs-cloud + LB)**, where the BGP-mode VIP mechanism you saw here for
the control-plane endpoint reappears as MetalLB BGP for Service-type
LoadBalancers — same protocol, different target.

Next: **10.4 — Declarative fleets (Cluster API + Talos)** takes everything
you just did by hand — the `KubeadmControlPlane`, the VIP, the version-skew
sequencing — and turns it into a reconciled Kubernetes object. The manual
runbook you built in this lesson's worked example is, almost line for line,
what `KubeadmControlPlane`'s controller executes for you at fleet scale.

## References & further reading

**Primary sources**
- **kubeadm HA topology** — <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/> — read for the exact stacked-vs-external procedure this lesson's worked example follows.
- **Kubernetes version-skew policy** — <https://kubernetes.io/releases/version-skew-policy/> — read for the authoritative, current numeric skew bounds (don't memorize from a blog; this changes across releases).
- **Kubernetes Leases (coordination.k8s.io)** — <https://kubernetes.io/docs/concepts/architecture/leases/> — read for how the Lease API backs both leader election and node heartbeats.
- **RFC 5798 — VRRPv3** — <https://datatracker.ietf.org/doc/html/rfc5798> — read for the protocol mechanics underneath every ARP-mode VIP (kube-vip and keepalived both implement this).

**Real-world engineering blogs**
- **Red Hat — HA bare-metal Kubernetes with MetalLB BGP** — <https://www.redhat.com/en/blog/deploying-a-high-availability-fault-tolerant-kubernetes-service-on-baremetal-clusters-with-metallb-bgp> — what it shows: a vendor reference architecture for BGP-advertised VIPs on bare metal, the same mechanism this lesson uses for the control-plane endpoint.
- **kube-vip GitHub issue #732** — <https://github.com/kube-vip/kube-vip/issues/732> — what it shows: a real failure mode where the control-plane VIP disappears once the sole/last control-plane node is gone — proof that VIP HA depends on the control plane it fronts.
- **CoreWeave — node lifecycle management** — <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference> — what it shows: the neocloud framing that undetected hardware failure (up to a month, unmanaged) is the same class of problem as control-plane HA — never let one box be a single point of failure.

**Deeper dives**
- **kube-vip documentation** — <https://kube-vip.io/> — the ARP vs BGP mode docs, static-pod manifest generation, and design rationale for running the VIP as a lightweight in-cluster process instead of external LB hardware.
