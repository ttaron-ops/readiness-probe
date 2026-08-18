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
sources: 9
---
# 10.3 · Control-plane HA & upgrades

> **Concept.** A self-managed control plane survives node loss only if you build the redundancy the cloud used to hand you for free: an odd etcd quorum, a load-balanced apiserver behind a stable endpoint, active-passive leader election for the singleton controllers, and an upgrade order that never violates version skew.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Lesson 10.2 made etcd's disk, quorum, and disaster-recovery runbook yours — you now know how to
keep the **datastore** alive and how to reason about `N`, quorum, and what a partition does. That
lesson deliberately stayed inside etcd: the WAL, the bbolt file, compaction, defrag, restore. What
it did not cover is the layer that *sits on top* of a healthy etcd: how do multiple control-plane
nodes present themselves as **one** control plane to the rest of the cluster, what actually moves
a virtual IP when a node dies, and how do you walk that whole assembly from one Kubernetes minor
version to the next without ever serving an API that violates the version-skew contract? That is
this lesson. By the end you can grow the single hand-built node from 10.1 into a 3-node HA control
plane behind a VIP, and drive it through a live minor upgrade — the exact shape the module's
checkpoint asks you to defend.

## Why this matters

On EKS/GKE the control plane is a billed black box. AWS runs redundant apiserver replicas across
availability zones, a managed etcd, a network load balancer in front, and rolling upgrades — you
click a version number and it happens. The SLA is a line item, not something you operate.

At a GPU neocloud you own it. If the control plane goes down, the *workloads* keep running for a
while — kubelets already hold their pod specs and containerd keeps existing containers alive — but
you lose the ability to schedule, scale, recover from node failure, or serve `kubectl`. Which
means: no GPU job placement, no autoscaling a 400-node training fleet, no draining a node with a
failing NVLink, no responding to an XID event, and no automated remediation. A single-node control
plane on bare metal is a career-limiting outage waiting for a routine reboot.

The upgrade half matters just as much, and for a less obvious reason. A control plane you cannot
safely upgrade is a control plane you will not upgrade, which is how clusters end up three minors
behind, out of patch support, and — per lesson 10.1 — one year past their certificate expiry.
`kubeadm upgrade apply` renews the control-plane certificates as a side effect (its
`--certificate-renewal` flag defaults to `true`); a cluster that gets upgraded on a cadence never
sees the cert-expiry outage, and a cluster that does not, does. **The upgrade cadence is a
reliability control, not just a feature-currency control.**

This is also precisely the bar in job postings like CoreWeave's "day-2 lifecycle, fault-tolerant
architectures" and NVIDIA's node-lifecycle roles (see the module README). And it feeds the cost
story directly: control-plane HA is three always-on machines you must capitalize and defend
against the managed-control-plane fee in your capex-vs-cloud model.

## What's new here (calibration)

You already know the components from **K8s-controllers 02**: apiserver, controller-manager,
scheduler, etcd, and the reconcile-loop mental model. You own etcd's internals, quorum arithmetic
and DR from **10.2**, and the PKI and static-pod bootstrap from **10.1**. What is genuinely new:

- **HA as three independent properties**, not one — endpoint availability, datastore quorum, and
  singleton-controller continuity fail separately and are provided by different mechanisms. Most
  "we have HA" claims are one of the three.
- **What actually moves a VIP.** kube-vip does **not** speak VRRP, despite what a lot of writing
  says; it runs Kubernetes Lease-based leader election and then advertises with gratuitous ARP or
  BGP. keepalived is the VRRP one. The distinction changes your failure envelope and your
  dependency graph, and it is checkable in the source.
- **Leader election numbers you can reason about** — lease duration, renew deadline and retry
  period, and the arithmetic that turns them into a worst-case failover time.
- **The exact, memorizable version-skew bounds** — including two that older material commonly gets
  wrong (when the kubelet window widened to 3 minors, and what the kube-proxy rule actually says).
- **kubeadm's upgrade pipeline as phases**, in source order, and where cert renewal happens.
- **Availability and maintenance arithmetic** — what 1, 3 and 5 control-plane nodes actually buy,
  and how long a fleet upgrade window really is.

We do **not** re-derive etcd's Raft internals (10.2) or re-teach what apiserver/CM/scheduler/etcd
each do (K8s-controllers 02).

**Version note.** kubeadm phase orders, flag defaults, leader-election defaults and skew bounds
below were read from `kubernetes/kubernetes` (`release-1.36`) and `kubernetes/website`
(`content/en/releases/version-skew-policy.md`) source trees; kube-vip behaviour from the
`kube-vip/kube-vip` main branch (August 2026). kubernetes.io and kube-vip.io are unreachable from
this environment's egress proxy, so the rendered documentation sites were **not** used — see
[References](#references--further-reading).

## Core concepts

### HA is three separate properties, and they fail separately

The single most common mistake in bare-metal control-plane design is treating "HA" as one thing
you either have or do not. It is three:

| Property | Provided by | Fails when | Symptom |
|---|---|---|---|
| **Endpoint availability** — clients can always reach *a* live apiserver | VIP or external LB in front of N apiservers | the VIP holder dies and nothing takes it over; or the LB itself is a SPOF | `kubectl` times out / connection refused, from everywhere |
| **Datastore quorum** — writes can be committed | odd-sized etcd cluster (10.2) | ⌊N/2⌋+1 members are unreachable | `kubectl get` hangs too; `etcd_server_has_leader == 0` |
| **Singleton continuity** — exactly one scheduler and one controller-manager is active, and a replacement takes over promptly | Lease-based leader election | nothing "fails" visibly; work just stops until the lease expires | pods stay `Pending`, Deployments do not roll, node conditions go stale |

They are genuinely independent. You can have a perfect VIP over three apiservers whose etcd has
lost quorum (endpoint HA, no datastore HA — `kubectl` connects instantly and then hangs). You can
have a healthy etcd behind a single apiserver (datastore HA, no endpoint HA). And you can have
both while the controller-manager lease sits unclaimed for 15 seconds after a node dies, during
which nothing reconciles.

Design for all three, and when you are debugging, name which one is broken before you touch
anything.

```
                                  clients: kubectl, kubelets on 400 nodes,
                                  controller-manager, scheduler, CAPI, CI
                                             │
                                             ▼
                              ┌─────────────────────────────┐
   PROPERTY 1                 │  10.10.0.100:6443  (VIP)    │   endpoint availability
   endpoint availability      │  held by exactly one node   │   — one address that always
                              └──────────────┬──────────────┘     reaches a live apiserver
                       ┌─────────────────────┼─────────────────────┐
                       ▼                     ▼                     ▼
                 ┌───────────┐         ┌───────────┐         ┌───────────┐
                 │   cp1     │         │   cp2     │         │   cp3     │
                 │           │         │           │         │           │
   ACTIVE-ACTIVE │ apiserver │         │ apiserver │         │ apiserver │
   (stateless)   │           │         │           │         │           │
                 ├───────────┤         ├───────────┤         ├───────────┤
   ACTIVE-       │ c-manager │◀── Lease "kube-controller-manager" ──────▶│
   PASSIVE       │  LEADER   │         │   idle    │         │   idle    │
   (leases)      │ scheduler │◀── Lease "kube-scheduler" ───────────────▶│
                 │  LEADER   │         │   idle    │         │   idle    │
                 ├───────────┤         ├───────────┤         ├───────────┤
   PROPERTY 2    │  etcd m1  │◀───────▶│  etcd m2  │◀───────▶│  etcd m3  │
   quorum        │  LEADER   │  Raft   │           │  Raft   │           │
                 └───────────┘         └───────────┘         └───────────┘
                       └──── quorum = 2 of 3; tolerates 1 member loss ────┘
   PROPERTY 3 (singleton continuity) is the two Lease rows in the middle.

   NOTE the three leaderships are INDEPENDENT and usually land on different
   nodes: the etcd Raft leader, the controller-manager lease holder, the
   scheduler lease holder, and the VIP holder are four separate elections.
```

### Sizing the control plane: 1, 3, and 5

Quorum arithmetic decides everything, so make it concrete. This is the diagram to reproduce at the
checkpoint:

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  N = 1        quorum = ⌊1/2⌋+1 = 1        tolerates 1 − 1 = 0 failures   │
  │                                                                          │
  │      [cp1]                                                               │
  │        ▲  every write needs 1 of 1                                       │
  │                                                                          │
  │  Any reboot, kernel panic, or `apt upgrade` is a full control-plane       │
  │  outage. Planned maintenance = downtime. Fine for a lab, never for a     │
  │  fleet.  MAINTENANCE HEADROOM: none.                                     │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  N = 3        quorum = ⌊3/2⌋+1 = 2        tolerates 3 − 2 = 1 failure    │
  │                                                                          │
  │      [cp1]────[cp2]────[cp3]                                             │
  │        └────────┴────────┘   every write needs 2 of 3                    │
  │                                                                          │
  │   cp3 dies  → {cp1,cp2} = 2 ≥ 2 → WRITES CONTINUE                        │
  │   cp2 also  → {cp1}     = 1 <  2 → READ-ONLY, no leader, kubectl hangs   │
  │                                                                          │
  │  MAINTENANCE HEADROOM: ZERO. While you reboot cp3 for a patch, the       │
  │  cluster is a 2-node cluster that tolerates 0 further failures. A disk   │
  │  dying on cp1 during that window is a full outage.                       │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  N = 5        quorum = ⌊5/2⌋+1 = 3        tolerates 5 − 3 = 2 failures   │
  │                                                                          │
  │      [cp1]──[cp2]──[cp3]──[cp4]──[cp5]     every write needs 3 of 5      │
  │                                                                          │
  │   cp5 down for patching  →  4 alive, still tolerates 1 more              │
  │   cp4 then dies unplanned →  3 alive = quorum → WRITES CONTINUE          │
  │                                                                          │
  │  MAINTENANCE HEADROOM: ONE. This — not raw availability — is the real    │
  │  argument for 5 on a large fleet.                                        │
  │  COST: a commit waits for the 3rd-fastest of 5 rather than the 2nd-      │
  │  fastest of 3, so one slow member becomes invisible instead of fatal.    │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  N = 2 or 4 — STRICTLY WORSE than the odd number below                   │
  │    N=2: quorum 2, tolerates 0.  Twice the hardware that can fail,        │
  │         zero tolerance. Worse than a single node.                        │
  │    N=4: quorum 3, tolerates 1.  Same tolerance as 3, one more machine    │
  │         to buy, patch, monitor, and put in the commit path.              │
  └──────────────────────────────────────────────────────────────────────────┘
```

**Worked availability arithmetic.** Suppose a control-plane node has independent availability
`p = 0.99` (about 3.65 days of downtime per year, which is pessimistic for a real server but makes
the arithmetic legible). The control plane is up when at least `quorum` nodes are up:

```
  N = 1:  A = p                                   = 0.9900        ≈ 87.6 h/yr down
  N = 3:  A = P(3 up) + P(exactly 2 up)
            = p³ + 3p²(1−p)
            = 0.970299 + 3(0.9801)(0.01)
            = 0.970299 + 0.029403               = 0.999702        ≈  2.6 h/yr down
  N = 5:  A = p⁵ + 5p⁴(1−p) + 10p³(1−p)²
            = 0.9509900 + 0.0480298 + 0.0009703 = 0.999990        ≈  5.2 min/yr down
```

Two lessons fall out, and both are more useful than the numbers themselves:

1. **Going 1 → 3 buys two orders of magnitude; going 3 → 5 buys one more.** Diminishing, but not
   negligible.
2. **This arithmetic assumes independence, and on bare metal that assumption is usually false.**
   Three control-plane nodes in one rack share a PDU, a top-of-rack switch, and a cooling zone.
   If a rack-level event has probability `q = 0.001/yr` and takes all three, then no amount of
   `p` improves you past `1 − q`. **Spreading across failure domains is worth more than adding
   members**, which is why the placement discussion later in this lesson is not an afterthought.

**Practical rule.** 3 for almost everything. 5 when (a) the cluster is large enough that etcd
write load makes one slow member a real risk, (b) you patch often enough that "zero maintenance
headroom" is a weekly condition, or (c) you have at least 5 genuinely independent failure domains
to put them in. Never even numbers.

### Stacked vs external etcd

**Stacked** — each control-plane node runs its own etcd member as a static pod alongside the
apiserver, which reaches it over loopback. kubeadm's default; `kubeadm join --control-plane` adds
the etcd member for you (this is why `join` has a `check-etcd` phase before `etcd-join`: it
refuses to add a member to an unhealthy cluster).

**External** — etcd runs on its own machines, and apiservers point at it with `--etcd-servers`.
You configure `etcd.external.{endpoints,caFile,certFile,keyFile}` in `ClusterConfiguration`, and
the control-plane nodes typically do not hold the etcd CA key at all.

The difference that matters is blast radius:

| | Stacked (3 nodes) | External (3 CP + 3 etcd) |
|---|---|---|
| Machines | 3 | 6 |
| Losing one CP node costs you | one apiserver **and** one etcd member | one apiserver only |
| Node failures tolerated | **1** (2nd loss breaks etcd quorum) | apiservers: down to 1; etcd: 1 |
| Correlated rack/PDU failure taking 2 nodes | kills both planes at once | kills only the plane you placed badly |
| etcd disk isolation | shares the node with the apiserver, kubelet, containerd, logs | dedicated machines, dedicated NVMe |
| Things to patch, monitor, back up | one set | two sets |
| Scale etcd independently | no | yes |

Rule of thumb: **stacked for most on-prem clusters** (3 nodes, spread across racks); **external**
when etcd write load is heavy (very large clusters, high object churn — see 10.2's growth math),
when you want to give etcd its own NVMe and its own patch cadence, or when compliance wants the
datastore isolated. On a GPU fleet the strongest argument for external is 10.2's: etcd is
latency-bound on `fsync`, and a stacked member shares its disk with container image pulls and
kubelet logs, which are exactly the bulk-I/O neighbours that ruin `fsync` p99.

### Load-balancing the apiserver

Every apiserver is stateless and interchangeable: they all talk to the same etcd, and any one can
serve any request. Clients need **one stable endpoint** that fans out across the live ones on
`:6443`.

Two hard constraints, both inherited from 10.1:

- **The endpoint must be in every apiserver serving certificate's SAN list**, because clients
  verify the address they dialled against SANs. Set `controlPlaneEndpoint` before `kubeadm init`.
- **The load balancer must do L4 TCP passthrough, never TLS termination.** Kubernetes
  authentication is mutual TLS end-to-end; an LB that terminates TLS strips the client
  certificate, and every request arrives as `system:anonymous`.

| Option | Mechanism | Pick it when |
|---|---|---|
| External LB (hardware, or an HAProxy pair) | a box outside the CP nodes proxies TCP `:6443` | you already run one, or you want the LB failure domain fully separate |
| VIP self-hosted on the CP nodes (kube-vip) | a floating IP moved by leader election, advertised by ARP or BGP | the bare-metal default: no extra hardware, no extra team |
| Cloud NLB | provider's L4 LB | you are not on bare metal |
| DNS round-robin | multiple A records | **never.** Clients cache resolutions, there is no health awareness, and failover is bounded by TTL plus client behaviour you do not control |

A minimal external-LB config, if you want the LB off the CP nodes — one HAProxy + keepalived pair
per rack is a common on-prem shape:

```
# /etc/haproxy/haproxy.cfg — fronts three apiservers on :6443
global
    log /dev/log local0
    maxconn 4096

defaults
    mode    tcp                 # NOT http: we must not terminate TLS
    timeout connect 5s
    timeout client  4h          # long: watches and `kubectl exec` are long-lived
    timeout server  4h

frontend k8s-api
    bind *:6443
    default_backend k8s-api-backend

backend k8s-api-backend
    balance     roundrobin
    option      tcp-check
    # a real health check: ask the apiserver's own readiness endpoint.
    # /readyz is served on the same TLS port; a raw TCP connect would keep
    # sending traffic to an apiserver whose etcd is unreachable.
    option      httpchk GET /readyz
    http-check  expect status 200
    server cp1 10.10.0.11:6443 check check-ssl verify none inter 3s fall 2 rise 2
    server cp2 10.10.0.12:6443 check check-ssl verify none inter 3s fall 2 rise 2
    server cp3 10.10.0.13:6443 check check-ssl verify none inter 3s fall 2 rise 2
```

The `option httpchk GET /readyz` line is the one people omit, and it is the one that matters. A
plain TCP check keeps an apiserver in rotation when it is listening but cannot reach etcd —
producing a cluster where one third of your `kubectl` calls hang. `/readyz` fails in that state.
(`/livez` would not: it is deliberately more forgiving, because you do not want the kubelet
restarting an apiserver just because etcd is briefly slow.)

### What actually moves a VIP: kube-vip, keepalived, and BGP

A control-plane VIP provides **exactly one thing: a stable, highly-available IP that always
reaches a live apiserver.** It is not etcd quorum, not replication, and not data safety. Getting
the mechanism right matters because the three options have genuinely different dependency graphs.

**keepalived implements VRRP (RFC 5798).** Nodes exchange VRRP advertisements on a multicast
group; the highest-priority live node owns the VIP and answers ARP for it. On failure, another
node stops hearing advertisements, claims the VIP, and sends a gratuitous ARP so the switch
updates its MAC table. VRRP is self-contained: it depends on nothing but the local network.

**kube-vip does not implement VRRP.** This is worth stating plainly because a great deal of
published material says otherwise. In the `kube-vip/kube-vip` source, `VRRP` appears exactly once
— as the IP protocol number constant `ProtocolVRRP = 112` used for firewall rules — and there is
no VRRP state machine anywhere. What kube-vip actually does:

- **Election**: `--leaderElection` uses the **Kubernetes Lease API** (or, with
  `--leaderElectionType=etcd`, an etcd lease). The control-plane lock defaults to a Lease named
  **`plndr-cp-lock`**, with `--leaseDuration 15` seconds, `--leaseRenewDuration 10` seconds,
  and `--leaseRetry 2` seconds — the same client-go leader-election machinery kube-controller-manager
  uses.
- **Advertisement, ARP mode** (`--arp`): the elected leader adds the VIP to the named interface
  and broadcasts **gratuitous ARP** on a timer (`vip_arpRate`; the manager clamps anything under
  500 ms up to 3000 ms) so switches keep the MAC binding fresh.
- **Advertisement, BGP mode** (`--bgp`): the node peers with the top-of-rack switch and advertises
  the VIP as a `/32`. Defaults: `--localAS 65000`, `--peerAS 65000`, `--bgpHoldTimer 30`,
  `--bgpKeepAliveInterval 10`. Failover is a route withdrawal and re-advertisement, handled by
  the fabric.

**The consequence of Lease-based election is a dependency you must see clearly: kube-vip's
control-plane VIP depends on the Kubernetes API it is fronting.** That is not fatal — the leader
holds the VIP and reaches the apiserver locally — but it defines the failure envelope, and it is
exactly what kube-vip issue #732 documents: in an RKE2 cluster running kube-vip with leader
election, when the *sole* control-plane node died, the VIP became unreachable from the remaining
workers roughly five minutes later, rather than persisting. There was no surviving node that could
win an election, because winning requires writing a Lease to the API that just died. keepalived,
which depends only on VRRP on the wire, does not have this property — it would keep the VIP alive
on a node that cannot do anything useful with it, which is a different trade, not obviously a
better one.

Comparison, so you can choose rather than copy:

| | keepalived (VRRP) | kube-vip ARP mode | kube-vip BGP mode |
|---|---|---|---|
| Who decides the owner | VRRP priority election on the wire | Kubernetes (or etcd) Lease | Kubernetes (or etcd) Lease |
| Depends on the API being up | no | **yes** | **yes** |
| Requires L2 adjacency | **yes** | **yes** | no |
| Works across racks / L3 | no | no | **yes** |
| Failover advertisement | gratuitous ARP | gratuitous ARP | BGP withdraw + advertise |
| Extra daemon/machines | a daemon per node (often + HAProxy) | one static pod per CP node | one static pod per CP node |
| Needs switch cooperation | no | no | **yes** (BGP peering, ASNs) |
| Also serves `type: LoadBalancer` | no | yes (`--services`) | yes |

**ARP-mode VIPs cannot cross an L3 boundary.** ARP is not routed. If your three control-plane
nodes are in three racks — which is exactly what the failure-domain argument above tells you to
do — then they are usually in three L2 segments joined by routed links, and an ARP-mode VIP that
worked perfectly in the lab silently cannot fail over. That is the moment you need BGP mode, and
it is the same mechanism MetalLB uses to advertise Service VIPs (lesson 10.8).

Here is the failover, with the actual timers, so you can put a number on your outage:

```
  t = 0.0s   cp1 holds the VIP 10.10.0.100 and the Lease "plndr-cp-lock".
             It renews the Lease every ~2s (retryPeriod) and must succeed
             within 10s (renewDeadline). It gARPs 10.10.0.100 every 3s.
             cp2 and cp3 watch the Lease; both are idle.

  t = 0.0s   ██ cp1 LOSES POWER ██   (no graceful release — worst case)

  t = 0.0s   Clients holding open TCP connections to 10.10.0.100 see nothing
             yet: the socket is established, no packets are being sent.
             `kubectl get -w` streams just stop producing events.

  t ≈ 0-15s  cp2 and cp3 keep trying to acquire. They will not succeed until
             the Lease's renewTime + leaseDuration (15s) has passed, because
             that is the contract: "the maximum duration a leader can be
             stopped before it is replaced."
             ── WORST-CASE VIP OUTAGE ≈ leaseDuration = 15s ──
             (If cp1 had shut down gracefully it would release the Lease and
              this collapses to ~1 renewal interval.)

  t ≈ 15.2s  cp2 wins: it updates the Lease's holderIdentity.
             • adds 10.10.0.100/32 to its interface
             • sends GRATUITOUS ARP for 10.10.0.100 with cp2's MAC
                (BGP mode instead: advertises 10.10.0.100/32 to the ToR)

  t ≈ 15.3s  The switch updates its MAC table from the gARP. New TCP
             connections to 10.10.0.100 now land on cp2.
             ── this step is why L2 adjacency is required in ARP mode ──

  t ≈ 15-45s Clients recover. Existing connections were pinned to cp1's MAC
             and are dead; they fail on the next write with a connection
             reset or a TCP timeout, and client-go reconnects. kubelets take
             longest (they hold long-lived watches), typically well inside
             --node-monitor-grace-period, so no node flaps NotReady.

  MEANWHILE, ON THE OTHER TWO ELECTIONS (independent, overlapping):
  t ≈ 0-15s  the kube-controller-manager and kube-scheduler Leases held by
             cp1 also sit unrenewed for up to leaseDuration = 15s; cp2 or
             cp3 then take over. Nothing is scheduled during that window.
  t ≈ 0-2s   etcd: cp1's member is gone. If it was the Raft LEADER, the
             remaining two elect a new one after a randomized election
             timeout of 1000-2000ms (10.2). Quorum is 2 of 3, so writes
             resume in about a second and never actually stopped for reads.
```

**Total worst-case: about 15 seconds of "the control plane is unreachable," dominated entirely by
the 15-second lease duration, not by anything network-level.** That is the number to quote when
someone asks. You can shorten it (`--leaseDuration 5 --leaseRenewDuration 3 --leaseRetry 1`) at
the cost of more API writes and a much higher chance of a spurious failover when the apiserver is
briefly slow — which, per 10.2, is exactly what happens when etcd's disk is having a bad minute.
Do not tune this down without fixing the disk first.

### Leader election for the singleton controllers

The apiserver is **active-active**: stateless, N-way, and the VIP simply routes to whichever is
up. Controller-manager and scheduler must be **active-passive**, because two schedulers binding
the same Pod concurrently, or two controller-managers reconciling the same object, would race.
(They would not corrupt etcd — every write is a compare-and-swap on `resourceVersion` — but they
would fight, producing duplicate pods, thrashing rollouts, and a lot of `Conflict` retries.)

Kubernetes enforces single-active via the **Lease API** (`coordination.k8s.io/v1`). Every replica
runs with `--leader-elect=true` (the default) and competes for a Lease in `kube-system`. The
defaults, from `k8s.io/component-base/config/v1alpha1/defaults.go`:

| Flag | Default | Meaning |
|---|---|---|
| `--leader-elect` | `true` | participate at all |
| `--leader-elect-lease-duration` | `15s` | **the maximum time a leader can be stopped before it is replaced** |
| `--leader-elect-renew-deadline` | `10s` | the leader must renew within this or it stops leading *itself* |
| `--leader-elect-retry-period` | `2s` | how often candidates retry acquire/renew |
| `--leader-elect-resource-lock` | `leases` | the only supported option now (Endpoints/ConfigMap locks are gone) |

The renew-deadline half is the subtle one and it is a genuine safety property: a leader that
cannot renew within 10 s **stops acting as leader on its own initiative**, before the 15-second
lease expires and someone else takes over. That 5-second gap is what prevents two active
controller-managers during a network partition — the old leader has already stood down by the
time the new one starts.

Operationally this is your "who is actually doing the work" query:

```console
$ kubectl -n kube-system get lease kube-controller-manager kube-scheduler \
    -o custom-columns=NAME:.metadata.name,HOLDER:.spec.holderIdentity,RENEWED:.spec.renewTime
NAME                      HOLDER                                   RENEWED
kube-controller-manager   cp2_8c1f0e6a-...                         2026-08-18T11:42:07.114Z
kube-scheduler            cp1_3d9b41c2-...                         2026-08-18T11:42:06.902Z
```

Two things to notice. The holders are on **different nodes** — there is no coordination between
the two elections, and no reason for them to agree. And `RENEWED` should never be more than a few
seconds old; a stale `renewTime` with nobody taking over means the candidates cannot write to the
API, which points at etcd, not at the controllers.

Worth knowing when you are debugging "why didn't this pod get scheduled": on a 3-node control
plane, **two of the three schedulers are deliberately doing nothing**. Reading logs on the wrong
node will show you a perfectly healthy process that has not made a decision in hours.

### Version skew: the bounds, and the order they force

Kubernetes guarantees interoperation only inside a bounded window. The current policy, from
`content/en/releases/version-skew-policy.md` in `kubernetes/website`:

| Component | Bound relative to `kube-apiserver` | Notes |
|---|---|---|
| `kube-apiserver` (HA peers) | newest and oldest within **1 minor** of each other | this is what makes a rolling CP upgrade legal |
| `kube-controller-manager`, `kube-scheduler`, `cloud-controller-manager` | up to **1 minor older**, never newer | expected to match; 1 back is allowed for live upgrades |
| `kubelet` | up to **3 minors older**, never newer | widened from 2 to 3 at **v1.25** |
| `kube-proxy` | up to **3 minors older** than the apiserver, never newer; **and** within **3 minors older *or newer*** than the kubelet it runs beside | also widened at v1.25 |
| `kubectl` | within **1 minor older *or newer*** | the only component allowed to be newer |
| `kubeadm` | must match the **target** control-plane version | it also refuses to skip minors |

Two of those rows are commonly misstated and both are worth correcting explicitly:

- **The kubelet window widened to 3 minors at v1.25, not v1.28.** The policy text is precise:
  "kubelet < 1.25 may only be up to two minor versions older."
- **kube-proxy is not required to match its node's kubelet.** It may be up to 3 minors *older or
  newer* than the kubelet beside it. The binding constraint is the apiserver, not the kubelet.

There is also a narrowing rule that catches people mid-upgrade: **if your apiservers are skewed
from each other, every other bound narrows to the oldest one.** With apiservers at 1.36 and 1.35
behind a load balancer, controller-manager and scheduler are only supported at 1.35, because they
might be routed to the 1.35 apiserver on any given request. This is why you finish the
control-plane upgrade before starting anything else.

The order follows mechanically:

```
  ┌─ 1 ─────────────────────────────────────────────────────────────────┐
  │  CONTROL PLANE, node by node. Within a node, kubeadm does:          │
  │    apiserver → controller-manager → scheduler (→ stacked etcd)      │
  │  WHY: CM/sched must never be NEWER than the apiserver they call.    │
  │  Upgrading the apiserver first makes the node's CM/sched            │
  │  temporarily 1 minor behind — which is legal — instead of           │
  │  temporarily ahead, which is not.                                   │
  │  During this phase apiservers are skewed by 1 across the fleet:     │
  │  legal, and the reason you must not linger here.                    │
  └─────────────────────────────┬───────────────────────────────────────┘
                                ▼
  ┌─ 2 ─────────────────────────────────────────────────────────────────┐
  │  THE OTHER CONTROL-PLANE NODES: `kubeadm upgrade node`.             │
  │  Now every apiserver is at the new minor; the skew window closes.   │
  └─────────────────────────────┬───────────────────────────────────────┘
                                ▼
  ┌─ 3 ─────────────────────────────────────────────────────────────────┐
  │  KUBELETS + kube-proxy, node by node: drain → upgrade → uncordon.   │
  │  WHY LAST: kubelet must never be newer than the apiserver.          │
  │  WHY YOU CAN TAKE YOUR TIME: kubelet may lag by 3 minors, so this   │
  │  phase does not have to happen in the same maintenance window —     │
  │  or even the same quarter.                                          │
  └─────────────────────────────────────────────────────────────────────┘

  AND: ONE MINOR AT A TIME. 1.34 → 1.36 in one hop is unsupported, and
  kubeadm refuses it. The reason is not superstition: API deprecations
  are announced in minor N, warned about while you run N, and removed in
  N+1. Skipping N means you never saw the warnings and never had a
  version that could serve both the old and new API.
```

**The 3-minor kubelet window is the operationally valuable fact.** On a 400-node GPU fleet you can
upgrade the control plane on a tight cadence and roll node upgrades lazily — folding them into
drains you were doing anyway for hardware remediation (10.6) or firmware (10.5) — instead of
forcing 400 simultaneous drains that would evict every training job at once.

### kubeadm's upgrade pipeline

`kubeadm upgrade` is phased like `init`. From `cmd/kubeadm/app/cmd/upgrade/` (k8s 1.36):

**`kubeadm upgrade apply <version>` — the first control-plane node:**
`preflight` → `control-plane` → `upload-config` → `kubeconfig` → `kubelet-config` →
`bootstrap-token` → `addon` → `post-upgrade`.

**`kubeadm upgrade node` — every other control-plane node, and worker nodes:**
`preflight` → `control-plane` → `kubeconfig` → `kubelet-config` → `addon` → `post-upgrade`.

Three flags on `apply` you should know rather than discover:

| Flag | Default | What it does |
|---|---|---|
| `--certificate-renewal` | **`true`** | renews the control-plane certificates of components it touches. This is why regularly-upgraded clusters never hit 10.1's cert-expiry outage. |
| `--etcd-upgrade` | `true` | upgrades the stacked etcd static pod too |
| `--dry-run` | `false` | writes the new manifests to a temp dir instead of `/etc/kubernetes/manifests` |

And two commands to run *before* you touch anything:

- **`kubeadm upgrade plan`** — prints the current and target versions for every component, the
  kubelet versions present across the fleet, whether a manual kubelet upgrade is required, and any
  configuration that needs migrating. Read every line.
- **`kubeadm upgrade diff <version>`** — prints a unified diff of the static-pod manifests it
  would write. This is the single best pre-upgrade check there is: you see the exact flag changes
  before they happen, including any `extraArgs` of yours that a new default will collide with.

**Rollback.** kubeadm has no `undo`. Your rollback path is: restore the pre-upgrade etcd snapshot
(10.2's DR runbook) and downgrade the `kubeadm`/`kubelet` packages. Which means **taking and
verifying an etcd snapshot immediately before `upgrade apply` is not optional** — it is the undo
button. Take it, run `etcdutl snapshot status` on it, and copy it off the node.

**Draining a control-plane node has a wrinkle worth naming.** `kubectl drain` will not evict
static pods (they are not managed by the API), so the apiserver, controller-manager, scheduler and
etcd on that node keep running through the drain. Drain removes the *workloads*, and cordons the
node; the control-plane components are stopped and restarted by the kubelet when kubeadm rewrites
their manifests. So "drain then upgrade" on a CP node means "get the addons off, then let kubeadm
swap the manifests" — the control plane component restart is what causes the brief unavailability,
not the drain.

### Failure-domain design

HA only works if the members cannot fail together. The arithmetic earlier makes this precise:
independent-failure math gives you five nines at N=5, but a correlated event that takes two of
three nodes puts you straight into 10.2's lost-quorum runbook regardless of `p`.

Spread across:

- **Racks / PDUs / ToR switches.** No single PDU or switch should be able to take 2+ CP nodes. At
  N=3 with two racks, put 1 and 2 — never 2 and 1 with the pair in the rack you consider "primary."
- **Cooling zones**, for the same reason, in facilities where that is a distinct domain.
- **BMC/management network segments**, so a management-network incident does not blind you to
  every CP node simultaneously at the exact moment you need out-of-band access.

And then note the coupling back to the VIP: **multi-rack placement forces BGP mode.** ARP-mode
VIPs need L2 adjacency, three racks usually means three L2 segments, so the failure-domain
decision and the VIP-mechanism decision are the same decision. Make it once, up front, with the
network team.

### Worked math: how long is the upgrade window?

The same shape as 10.1's provisioning arithmetic, and it is the number your change-advisory board
will ask for.

```
  Let  C = control-plane nodes            = 3
       W = worker nodes                   = 400
       t_cp = per-CP-node upgrade time    = 8 min   (package + manifests + wait healthy)
       t_w  = per-worker time             = 12 min  (drain + package + restart + uncordon)
       Pw   = workers you drain in parallel

  PHASE 1+2  control plane, strictly serial (never two CP nodes at once — at
             N=3 that is a quorum loss):
                T_cp = C × t_cp = 3 × 8 = 24 min

  PHASE 3    workers:
                T_w = t_w × ceil(W / Pw)

                Pw = 1  →  12 × 400 = 4800 min = 80 hours
                Pw = 10 →  12 ×  40 =  480 min =  8 hours
                Pw = 25 →  12 ×  16 =  192 min =  3.2 hours

  TOTAL at Pw = 10:  24 min + 8 h  ≈  8.4 hours
```

Now the constraint that actually sets `Pw` on a GPU fleet, and it is not the tooling:

```
  A drained node's GPUs are idle. If you drain Pw nodes at once, you have
  removed Pw × 8 GPUs from the schedulable pool for t_w each.

  Pw = 25 nodes × 8 GPUs × 12 min = 2400 GPU-minutes = 40 GPU-hours removed
  at any instant, and 400 nodes × 8 × 12 min = 640 GPU-hours over the run.

  At $2.20/GPU-hr (a 2026 neocloud rental snapshot — flag it as dated), the
  drain cost of one fleet-wide kubelet upgrade is
        640 GPU-hr × $2.20  ≈  $1,400 of capacity,
  regardless of Pw. Pw only changes how long the dent lasts, not its area.
```

Two conclusions worth carrying into 10.8's model:

1. **`Pw` trades wall-clock against instantaneous capacity loss, not total cost.** Choose it from
   your scheduler's headroom — if the fleet runs at 80% utilisation, draining 25% of it at once
   means queued jobs, so `Pw ≈ (1 − utilisation) × W` is the honest ceiling.
2. **The 3-minor kubelet window is worth real money.** Because you are not obliged to do phase 3
   at all in this window, you can amortise those 640 GPU-hours into drains you were already doing
   for hardware remediation and firmware. That is the argument for *not* doing a synchronised
   fleet-wide kubelet bump, and it is quantitative rather than aesthetic.

## Perspectives

**Developer.** To someone running `kubectl apply`, a correctly built control plane is invisible:
one IP, and HA is a property they never think about. The only symptom of a well-executed failover
is a command that takes an extra fifteen seconds. If they *see* the control plane, something has
already gone wrong upstream of them.

**Operator.** This is the layer you own and the layer that pages you. The job is sequencing: never
let controller-manager get ahead of the apiserver, never take two CP nodes down at once at N=3,
never skip the pre-upgrade etcd snapshot, never `upgrade apply` without reading `upgrade diff`
first. Most control-plane HA incidents are self-inflicted — an upgrade run out of order, or a
"just reboot both to save time" shortcut that drops quorum. The runbook discipline from 10.2
extends one level up to cover the whole assembly.

**Hardware / network.** HA is a network-topology problem as much as a software one. ARP-based
VIPs need L2 adjacency, so your rack and subnet layout constrains your failure-domain design — or,
read the other way, your failure-domain requirements force BGP and therefore a conversation with
whoever owns the ToR switches about ASNs and peer configuration. Decide this before you cable
anything, because retrofitting it means re-addressing the control plane.

**Economics.** Three always-on control-plane nodes are pure overhead: they run no GPU workloads
and generate no revenue, yet must be capitalized, racked, powered, cooled and patched exactly like
the fleet they manage. In the capex-vs-cloud model this is the line item that offsets some of your
"we don't pay the managed-control-plane fee" savings. A correctly-sized 3-node CP is the efficient
choice for most fleets; 5 is justified by maintenance headroom, not by the availability decimal.
And the upgrade-window math above says the recurring cost of *operating* the control plane is
measured in GPU-hours of drain, which is a bigger number than the hardware.

## Real-world use cases

- **kube-vip — GitHub issue #732, "VIP is lost when kubernetes control plane goes down"** —
  <https://github.com/kube-vip/kube-vip/issues/732> — a concrete bug report: in an RKE2 cluster
  running kube-vip with leader election, the control-plane VIP became unreachable from all
  remaining worker nodes roughly five minutes after the sole control-plane node died, instead of
  persisting. What it shows: kube-vip's control-plane VIP is held by whoever wins a **Kubernetes
  Lease**, so it structurally depends on the API being reachable by *some* candidate. When the
  entire control plane is gone there is nobody to elect, and the VIP goes with it. This is not a
  defect in the design so much as the design's failure envelope — and it is precisely why the
  keepalived/VRRP comparison above is worth understanding rather than assuming the two are
  interchangeable.
- **Red Hat — "Deploying a high-availability, fault-tolerant Kubernetes Service on bare metal
  clusters with MetalLB BGP"** —
  <https://www.redhat.com/en/blog/deploying-a-high-availability-fault-tolerant-kubernetes-service-on-baremetal-clusters-with-metallb-bgp>
  — a vendor-documented reference architecture combining BGP advertisement with an HA bare-metal
  cluster. It targets workload/Service load-balancing (lesson 10.8's topic), but the mechanism is
  identical to the control-plane BGP VIP here: advertise a `/32` from each candidate node, let the
  fabric handle reachability via ECMP and route withdrawal instead of ARP. Worth reading as the
  practice architecture for the multi-rack case, and as evidence that "BGP for VIPs on bare metal"
  is a mainstream pattern rather than an exotic one.
- **CoreWeave — node lifecycle at neocloud scale** —
  <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference> —
  CoreWeave's framing (also quoted in the module README) that a faulty node can go undetected for
  up to a month without automated lifecycle management. The full remediation story is 10.6's; the
  relevance here is that **HA control-plane design and fast hardware fault response are the same
  operational muscle** at a neocloud — and, more concretely, that the automated remediation which
  makes a large fleet tractable *requires* a control plane that is always up to run it. A
  single-node control plane does not just risk an outage; it disables the machinery that keeps
  your GPUs healthy.

## Worked example — grow a single-node CP to 3-node HA + one upgrade

Starting point: the kubeadm single control-plane node from 10.1, initialised with
`--control-plane-endpoint 10.10.0.100:6443` (if it was not, stop and redo that — see Common
pitfalls). Subnet-local nodes `cp1`/`cp2`/`cp3` on `10.10.0.11–13`, free VIP `10.10.0.100`.

**1. Put kube-vip in front, as a static pod on cp1.**

```bash
export VIP=10.10.0.100 INTERFACE=eth0 KVVERSION=v1.0.0
ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip \
  /kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --arp \
    --leaderElection \
  | tee /etc/kubernetes/manifests/kube-vip.yaml
```

`--controlplane` enables the control-plane VIP; `--arp` selects gratuitous-ARP advertisement (swap
for `--bgp --peerAddress <ToR> --peerAS 65000 --localAS 65000` if the CP nodes span racks);
`--leaderElection` turns on the Lease-based election that decides who holds it. Because it is a
static pod, the kubelet starts it with no API dependency at boot — which is the only reason a
VIP-fronted control plane can cold-start at all.

Verify the VIP is live and that it is where you think it is:

```console
$ ip -brief addr show eth0
eth0   UP   10.10.0.11/24 10.10.0.100/32

$ kubectl -n kube-system get lease plndr-cp-lock \
    -o custom-columns=HOLDER:.spec.holderIdentity,RENEWED:.spec.renewTime
HOLDER   RENEWED
cp1      2026-08-18T11:31:44.000Z

$ kubectl --server https://10.10.0.100:6443 get --raw /readyz
ok
```

**2. Join cp2 and cp3 as control-plane nodes.** Re-mint the join command if the 2-hour
certificate-key TTL has expired:

```console
$ kubeadm init phase upload-certs --upload-certs
[upload-certs] Using certificate key:
9f8e7d6c5b4a39281706f5e4d3c2b1a09f8e7d6c5b4a39281706f5e4d3c2b1a0

$ kubeadm token create --print-join-command
kubeadm join 10.10.0.100:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:1a2b3c...
```

On cp2 and cp3, append `--control-plane --certificate-key <key>`:

```bash
kubeadm join 10.10.0.100:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:1a2b3c... \
  --control-plane --certificate-key 9f8e7d6c...
```

Watch what that does, in order: `preflight` → `control-plane-prepare` (downloads the CA keys from
the `kubeadm-certs` Secret and mints this node's certs — including the `sa` keypair, which is why
this mechanism prevents 10.1's intermittent-token-rejection bug) → **`check-etcd`** (refuses to
proceed if the existing etcd cluster is unhealthy) → `kubelet-start` → **`etcd-join`** (adds this
node as an etcd member) → `control-plane-join` → `wait-control-plane`.

Verify all three properties independently:

```console
# PROPERTY 2 — datastore quorum
$ kubectl -n kube-system exec etcd-cp1 -- etcdctl \
    --cacert /etc/kubernetes/pki/etcd/ca.crt \
    --cert   /etc/kubernetes/pki/etcd/server.crt \
    --key    /etc/kubernetes/pki/etcd/server.key \
    member list -w table
+------------------+---------+------+-------------------------+-------------------------+------------+
|        ID        | STATUS  | NAME |       PEER ADDRS        |      CLIENT ADDRS       | IS LEARNER |
+------------------+---------+------+-------------------------+-------------------------+------------+
| 8211f1d0f64f3269 | started | cp3  | https://10.10.0.13:2380 | https://10.10.0.13:2379 |      false |
| 91bc3c398fb3c146 | started | cp1  | https://10.10.0.11:2380 | https://10.10.0.11:2379 |      false |
| fd422379fda50e48 | started | cp2  | https://10.10.0.12:2380 | https://10.10.0.12:2379 |      false |
+------------------+---------+------+-------------------------+-------------------------+------------+

# PROPERTY 3 — singleton continuity: note the two leaders are on different nodes
$ kubectl -n kube-system get lease kube-controller-manager kube-scheduler \
    -o custom-columns=NAME:.metadata.name,HOLDER:.spec.holderIdentity
NAME                      HOLDER
kube-controller-manager   cp2_8c1f0e6a-...
kube-scheduler            cp1_3d9b41c2-...

# PROPERTY 1 — endpoint: three apiservers behind one address
$ kubectl -n kube-system get pods -l component=kube-apiserver -o wide
NAME                READY   STATUS    NODE
kube-apiserver-cp1  1/1     Running   cp1
kube-apiserver-cp2  1/1     Running   cp2
kube-apiserver-cp3  1/1     Running   cp3
```

**3. Kill the VIP holder and time the recovery.** From a workstation, start a clock:

```console
$ while true; do date +%H:%M:%S.%3N; kubectl get --raw /readyz 2>&1 | head -1; sleep 1; done
11:41:58.114 ok
11:41:59.121 ok
                                    # ← hard-power cp1 here
11:42:00.130 Unable to connect to the server: dial tcp 10.10.0.100:6443: i/o timeout
...
11:42:15.402 Unable to connect to the server: dial tcp 10.10.0.100:6443: i/o timeout
11:42:16.190 ok
```

**About 15 seconds**, matching the lease-duration arithmetic exactly. Confirm the VIP moved and
that etcd never lost quorum:

```console
$ ssh cp2 ip -brief addr show eth0
eth0   UP   10.10.0.12/24 10.10.0.100/32

$ kubectl -n kube-system get lease plndr-cp-lock -o jsonpath='{.spec.holderIdentity}{"\n"}'
cp2

$ kubectl create configmap ha-probe --from-literal=t=$(date +%s)
configmap/ha-probe created          # writes work: 2 of 3 etcd members is quorum
```

**4. Now kill cp2 as well — the blast-radius lesson, made real.**

```console
$ kubectl get nodes
Unable to connect to the server: ...        # eventually the VIP lands on cp3

$ ssh cp3 kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes
Error from server: etcdserver: request timed out
# ^^ the apiserver is UP and the VIP works. Endpoint HA is fine.
#    etcd has 1 of 3 members: quorum LOST. Reads and writes both fail.
```

That is the distinction between property 1 and property 2, demonstrated in two commands. Bring cp1
or cp2 back and quorum returns with no data loss (10.2's first recovery option).

**5. One minor upgrade, 1.35 → 1.36, respecting skew.**

```bash
# ---- BEFORE ANYTHING: the rollback button (10.2's discipline). ----
kubectl -n kube-system exec etcd-cp1 -- etcdctl \
  --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd/pre-upgrade.db
etcdutl snapshot status /var/lib/etcd/pre-upgrade.db -w table   # VERIFY it
scp cp1:/var/lib/etcd/pre-upgrade.db backup-host:/backups/      # OFF the node

# ---- cp1: the first control-plane node ----
apt-mark unhold kubeadm && apt-get update && apt-get install -y kubeadm=1.36.1-1.1 && apt-mark hold kubeadm
kubeadm version                       # confirm 1.36.1 before going further
kubeadm upgrade plan                  # read EVERY line
kubeadm upgrade diff v1.36.1          # see the manifest changes before they happen
kubeadm upgrade apply v1.36.1         # apiserver → CM → scheduler → stacked etcd,
                                      # and (--certificate-renewal=true) renews the certs

# ---- cp2, cp3: every other control-plane node ----
apt-mark unhold kubeadm && apt-get install -y kubeadm=1.36.1-1.1 && apt-mark hold kubeadm
kubeadm upgrade node

# ---- kubelets, one node at a time (CP nodes included) ----
kubectl drain cp1 --ignore-daemonsets --delete-emptydir-data
apt-mark unhold kubelet kubectl && apt-get install -y kubelet=1.36.1-1.1 kubectl=1.36.1-1.1 \
  && apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon cp1
# repeat for cp2, cp3, then workers in batches of Pw
```

Confirm the skew you actually have at each stage, rather than trusting the order:

```console
$ kubectl get nodes -o custom-columns=NAME:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion
NAME     KUBELET
cp1      v1.36.1
cp2      v1.35.4        # legal: kubelet may be up to 3 minors behind the apiserver
cp3      v1.35.4
w-001    v1.34.8        # also legal
```

`cp2` and `cp3` running a 1.35 kubelet against a 1.36 apiserver is fine and will stay fine for
three minors. That is the window that lets you stop here and finish phase 3 next month.

## Practice (hands-on, cheap VMs → deliverable)

Three small VMs in one subnet (Multipass, Vagrant/libvirt, or three cloud VMs; 2 vCPU / 2–4 GB
each). If you want to exercise the multi-rack case without racks, put the three VMs on separate
subnets behind a router and observe the ARP-mode VIP fail — that negative result is worth
capturing.

1. **Rebuild with a control-plane endpoint.** Take the 10.1 kubeadm cluster and, if it was not
   initialised with `--control-plane-endpoint`, rebuild it. Confirm with
   `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -ext subjectAltName` that the VIP is
   in the SAN list before you go further.
2. **Deploy kube-vip** as a static pod. Verify the VIP answers, `kubectl` works through it, and
   `kubectl -n kube-system get lease plndr-cp-lock` shows a holder. Record the lease's
   `leaseDurationSeconds`.
3. **Join cp2 and cp3** with `--control-plane --certificate-key`. Capture `etcdctl member list -w
   table` (3 members, none learners) and the `kube-controller-manager`/`kube-scheduler` lease
   holders — note whether they landed on different nodes.
4. **Kill the VIP holder and time it.** Run the polling loop from the worked example, hard-power
   the node, and record the exact outage duration. Compare it to `leaseDuration`. Then kill a
   **second** node and capture the difference in symptom: VIP still works, apiserver still up,
   `etcdserver: request timed out` on every request. **Write down why those two failures look
   different** — that is the property-1-versus-property-2 distinction the checkpoint wants.
5. **Upgrade one minor** in the correct order, with a verified etcd snapshot taken first. Capture
   `kubeadm upgrade plan` and `kubeadm upgrade diff` output before, and the `kubectl get nodes`
   VERSION column at each stage — including a stage where the kubelets are deliberately left one
   minor behind, to demonstrate the skew window rather than assert it.
6. **Do the arithmetic for your real fleet.** Using your measured `t_cp` and `t_w`, compute the
   upgrade window for `W = 400` at three values of `Pw`, and the GPU-hours of drain it costs.

**Acceptance (feeds the capex-vs-cloud writeup):** a **3-node HA control plane behind a VIP**,
plus a short documented runbook containing (a) `etcdctl member list -w table` showing 3 members,
(b) timestamped evidence that the API survived single-node loss with a measured failover time, and
that it froze on two-node loss, **with the mechanism for each named**, (c) the before/after
upgrade versions with the order you followed and the specific skew bound you relied on, and (d)
your upgrade-window arithmetic with GPU-hour cost. This is the "the control plane is yours" section
of the deliverable — the section a managed cluster would never let you write.

## Common pitfalls

- **Setting `controlPlaneEndpoint` after `kubeadm init`.** The apiserver serving certificate's
  SANs are fixed when the cert is minted, and `kubeadm certs renew` re-signs the same fields
  rather than discovering new ones. Retrofitting a VIP means regenerating `apiserver.crt`
  (10.1's procedure), not editing a config file. Decide the endpoint before the first `init`,
  even on a single node.
- **Terminating TLS on the load balancer.** Kubernetes auth is end-to-end mTLS; an LB that
  terminates TLS strips the client certificate and every request arrives as `system:anonymous`.
  L4 TCP passthrough only.
- **Health-checking the apiserver with a bare TCP connect.** An apiserver that is listening but
  cannot reach etcd will accept your TCP connection and then hang. Check `/readyz` (not `/livez`,
  which is deliberately more forgiving so the kubelet does not restart the apiserver over a slow
  etcd).
- **Believing kube-vip speaks VRRP.** It does not — it uses Kubernetes Lease-based leader election
  plus gratuitous ARP or BGP. That means its control-plane VIP depends on the API it fronts (see
  issue #732), and it means the tuning knobs are lease timers, not VRRP advertisement intervals.
  If you actually want a VIP that survives a totally dead control plane, that is keepalived's
  property, not kube-vip's.
- **Assuming ARP-mode VIPs work across racks.** ARP is not routed. A VIP that works on one switch
  silently cannot fail over once the CP nodes are spread across L3 boundaries — which is exactly
  what good failure-domain design tells you to do. BGP mode, or single-rack placement; pick
  consciously.
- **Running an even number of control-plane nodes.** Two tolerates zero failures — worse than one.
  Four tolerates one, same as three, for an extra machine in the commit path.
- **Rebooting two CP nodes "to save a maintenance window."** At N=3 that is a deliberate quorum
  loss. There is no maintenance headroom at N=3; that is the actual argument for N=5.
- **Upgrading kubelet before the apiserver.** The skew policy is directional: nothing may be
  *newer* than the apiserver. Kubelet-ahead is unsupported, and the failures it produces are
  obscure API-version mismatches rather than a clean refusal.
- **Skipping `kubeadm upgrade diff` and the pre-upgrade snapshot.** kubeadm has no undo. The diff
  is how you find out that a new default collides with an `extraArgs` you set two years ago; the
  snapshot is the only rollback you have.
- **Reading scheduler logs on the wrong node.** Two of three schedulers are idle by design. Check
  the Lease holder first, or you will spend twenty minutes reading a perfectly healthy process
  that has made no decisions.

## Self-check

- **In what order do you upgrade apiserver / controller-manager / kubelet, and what are the
  version-skew constraints?**
  **Answer:** Control plane first, node by node and strictly one node at a time; within a node
  kubeadm does apiserver → controller-manager → scheduler (→ stacked etcd) in a single
  `kubeadm upgrade apply` on the *first* CP node, then `kubeadm upgrade node` on the others. Then
  kubelets and kube-proxy last, node by node, drain → upgrade → uncordon. Bounds: HA apiserver
  peers within **1 minor** of each other; controller-manager / scheduler / cloud-controller-manager
  up to **1 minor older** than the apiserver and never newer; **kubelet up to 3 minors older**
  (widened from 2 at **v1.25**) and never newer; **kube-proxy up to 3 minors older than the
  apiserver, and within 3 minors older *or newer* than its node's kubelet** — it is not required
  to match the kubelet; `kubectl` within 1 minor either way; kubeadm must match the target
  version. **One minor at a time.** And the narrowing rule: while your apiservers are skewed
  mid-upgrade, every other component's window narrows to the *oldest* apiserver.

- **Stacked-etcd vs external-etcd — what is the failure blast radius of each?**
  **Answer:** *Stacked:* each CP node holds an apiserver **and** an etcd member, so one node loss
  costs you both. With 3 stacked nodes you tolerate exactly **1** node loss; the second breaks
  etcd quorum and the whole API freezes for reads as well as writes. A correlated rack/PDU event
  taking two nodes is the real danger, and it is why placement matters more than member count.
  Stacked etcd also shares its disk with the kubelet, containerd and logs, which is the worst
  possible neighbourhood for a workload whose latency is `fsync`-bound. *External:* etcd runs on
  its own machines, so an apiserver-node failure does not touch etcd quorum and vice versa; you can
  run apiservers down to one and still serve the API. The cost is 3–5 extra machines and a second
  copy of everything in 10.2 to patch, monitor and back up.

- **What does the control-plane VIP actually provide, what does it not, and what moves it?**
  **Answer:** It provides **endpoint availability** only: one stable IP that always reaches a live
  apiserver. It does **not** provide etcd quorum, replication, or data safety — you can have a
  perfectly working VIP in front of a cluster whose etcd has lost quorum, and the symptom is that
  `kubectl` connects instantly and then times out. What moves it depends on the tool:
  **keepalived** runs VRRP (RFC 5798) on the wire and depends on nothing but the network;
  **kube-vip** runs Kubernetes (or etcd) **Lease-based leader election** — default lock
  `plndr-cp-lock`, `leaseDuration 15s`, `renewDeadline 10s`, `retryPeriod 2s` — and then advertises
  with **gratuitous ARP** (`--arp`, L2 only) or **BGP** (`--bgp`, works across L3). Because
  kube-vip's election runs through the API, its control-plane VIP depends on the API it fronts —
  the failure documented in kube-vip issue #732, where the VIP disappeared after the last CP node
  died.

- **Why are controller-manager and scheduler active-passive while the apiserver is active-active,
  and what enforces it?**
  **Answer:** The apiserver is stateless — every replica just serves requests against the same
  etcd — so N-way active is safe and desirable. Controller-manager and scheduler are not: two
  schedulers binding the same Pod, or two controllers reconciling the same object, would race.
  (etcd's compare-and-swap on `resourceVersion` stops actual corruption, but you get duplicate
  pods, thrashing rollouts and a storm of `Conflict` retries.) Kubernetes enforces single-active
  with the **Lease API** (`coordination.k8s.io/v1`): every replica runs `--leader-elect=true` and
  competes for a Lease in `kube-system`. Defaults: `lease-duration 15s`, `renew-deadline 10s`,
  `retry-period 2s`, resource lock `leases`. The renew-deadline is the safety property: a leader
  that cannot renew within 10 s **stands down on its own** before the 15 s lease expires, so there
  is a 5-second gap guaranteeing no two active instances during a partition. Query it with
  `kubectl -n kube-system get lease kube-controller-manager kube-scheduler`.

- **Your CP nodes are spread across 3 racks for failure-domain isolation. Will an ARP-mode
  kube-vip VIP work, and if not, what do you change?**
  **Answer:** Not reliably. Gratuitous ARP is how the VIP's new owner tells the switch where it
  now lives, and ARP is **not routed** — it requires all candidate holders to be in one broadcast
  domain. Three racks usually means three L2 segments joined by routed links, so the VIP either
  cannot be claimed at all outside its home segment, or claims silently fail to attract traffic.
  The fix is **BGP mode**: each CP node peers with its top-of-rack switch (`--bgp --peerAddress
  <ToR> --peerAS 65000 --localAS 65000`) and advertises the VIP as a `/32`; failover becomes a
  route withdrawal and re-advertisement handled by the fabric, with ECMP available. This is the
  same mechanism MetalLB uses for Service VIPs in 10.8. The corollary worth stating: the
  failure-domain decision and the VIP-mechanism decision are one decision, and it has to be made
  with the network team before you cable anything.

- **How long is a control-plane failover, and what sets that number?**
  **Answer:** About **15 seconds** worst case, and it is set almost entirely by
  `leaseDuration` — the contract is "the maximum duration a leader can be stopped before it is
  replaced," so candidates will not claim the lease until the holder's `renewTime` is that old.
  Everything else is fast by comparison: adding the IP and sending gratuitous ARP is milliseconds,
  the switch updates its MAC table on the next frame, and etcd re-elects a Raft leader in
  1000–2000 ms if the dead node held it. A *graceful* shutdown collapses this to about one renewal
  interval because the holder releases the lease. You can shorten the lease timers, but a shorter
  lease means more API writes and a much higher chance of a spurious failover whenever the
  apiserver is briefly slow — which, per 10.2, is what a struggling etcd disk looks like. Fix the
  disk before you tune the lease.

## Connections & what's next

This lesson sits directly on top of **10.2 (etcd operations)**: everything here about quorum and
blast radius is 10.2's arithmetic applied to node placement, the pre-upgrade snapshot step is
10.2's DR discipline reused as a rollback button, and the argument for external etcd is 10.2's
`fsync`-latency chapter applied to disk neighbours. It depends on **10.1** for the SAN list that
makes the VIP usable at all and for the `sa`-keypair consistency that `--certificate-key` provides.
It feeds forward into **10.5 (node provisioning)** and **10.6 (hardware health)**, whose
drain/cordon/RMA flows assume the control plane you are draining *from* is itself HA and will not
disappear mid-remediation — and whose drains are the ones you fold your lazy kubelet upgrades
into. And it sets up **10.8 (capex-vs-cloud + LB)**, where the BGP-mode VIP mechanism you just used
for the control-plane endpoint reappears as MetalLB BGP for Service-type LoadBalancers: same
protocol, different target.

Next: **[10.4 · Declarative fleets (Cluster API + Talos)](04-declarative-fleets-capi-talos.md)**
takes everything you just did by hand — join a CP node, add its etcd member, hold a VIP, sequence a
skew-safe upgrade — and turns it into a reconciled Kubernetes object. The manual runbook in this
lesson's worked example is, almost step for step, what `KubeadmControlPlane`'s controller executes
at fleet scale, including forwarding etcd leadership off a machine before removing its member.

## References & further reading

**Primary sources**

- **`kubernetes/website`, `content/en/releases/version-skew-policy.md`** —
  <https://github.com/kubernetes/website/blob/main/content/en/releases/version-skew-policy.md> —
  the authoritative skew bounds, read directly from the source of the published page (which
  kubernetes.io renders). This is where the "kubelet < 1.25 may only be up to two minor versions
  older" wording and the kube-proxy-vs-kubelet rule come from, both of which correct commonly
  repeated errors. Read for: the exact numeric bounds and the HA-skew narrowing rule, and re-read
  it every release rather than memorising from a blog.
- **`kubernetes/kubernetes`, branch `release-1.36`** —
  <https://github.com/kubernetes/kubernetes/tree/release-1.36> — read directly for:
  `cmd/kubeadm/app/cmd/upgrade/{apply,node}.go` (the upgrade phase orders and the
  `--certificate-renewal` default of `true`), `cmd/kubeadm/app/cmd/join.go` (the
  `check-etcd` → `etcd-join` ordering), and
  `staging/src/k8s.io/component-base/config/v1alpha1/defaults.go` plus
  `.../config/options/leaderelectionconfig.go` (leader-election defaults and the fact that
  `leases` is now the only supported resource lock). **Note:** kubernetes.io is unreachable from
  this environment's egress proxy, so its kubeadm HA-topology, upgrade and Lease pages were **not**
  fetched or relied upon.
- **`kube-vip/kube-vip`, main branch (Aug 2026)** — <https://github.com/kube-vip/kube-vip> — read
  directly and the basis for this lesson's correction of the widely repeated "kube-vip uses VRRP"
  claim: `cmd/kube-vip.go` carries the leader-election flags (`--leaderElection`,
  `--leaderElectionType`, `--leaseName` default `plndr-cp-lock`, `--leaseDuration 15`,
  `--leaseRenewDuration 10`, `--leaseRetry 2`) and the BGP defaults (`--localAS`/`--peerAS`
  65000, `--bgpHoldTimer 30`, `--bgpKeepAliveInterval 10`); `pkg/arp/arp.go` holds the gARP
  broadcast loop and its ≥500 ms clamp; and `VRRP` appears in the tree only as the IP protocol
  number `112` in `pkg/vip/const.go`, with no VRRP implementation. **kube-vip.io is also blocked
  by the egress proxy** and was not used. Read for: verifying the flag set for the version you
  deploy, since these defaults move.
- **RFC 5798 — VRRPv3** — <https://datatracker.ietf.org/doc/html/rfc5798> — the protocol
  **keepalived** implements (and kube-vip does not). Read for: the advertisement/priority
  mechanics behind a classic on-prem VIP, and to understand precisely what property you give up by
  choosing a Lease-based VIP instead.

**Real-world engineering blogs**

- **kube-vip GitHub issue #732** — <https://github.com/kube-vip/kube-vip/issues/732> — what it
  shows: a real failure where the control-plane VIP disappeared once the last control-plane node
  was gone, because the VIP is held by whoever wins a Kubernetes Lease. Concrete grounding for
  "endpoint HA built on the control plane's own health."
- **Red Hat — HA bare-metal Kubernetes with MetalLB BGP** —
  <https://www.redhat.com/en/blog/deploying-a-high-availability-fault-tolerant-kubernetes-service-on-baremetal-clusters-with-metallb-bgp>
  — what it shows: a vendor reference architecture for BGP-advertised VIPs on bare metal, the same
  mechanism this lesson uses for the multi-rack control-plane endpoint.
- **CoreWeave — node lifecycle management** —
  <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference> —
  what it shows: the neocloud framing that undetected hardware failure (up to a month, unmanaged)
  is the same class of problem as control-plane HA — and that the automation which prevents it
  needs a control plane that is always up.

**Deeper dives**

- **`kubeadm upgrade diff <version>` and `kubeadm upgrade plan`** — the only pre-upgrade sources
  that describe *your* cluster rather than a generic one. Run both before every upgrade; the diff
  shows the exact static-pod manifest changes, including collisions with `extraArgs` you set long
  ago.
- **HAProxy documentation, `option httpchk` / `http-check expect`** —
  <https://docs.haproxy.org/> — for building an apiserver health check that actually detects an
  apiserver whose etcd is unreachable, rather than one that merely accepts TCP connections.
