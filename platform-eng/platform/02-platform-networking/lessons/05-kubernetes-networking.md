---
lesson: "A02.5"
title: "Kubernetes networking"
module: "A-02"
concept: "service dataplanes & conntrack"
status: not-started
est_time: "4.5 hrs"
prev: "04-cloud-networking.md"
next: "06-service-mesh.md"
artifacts: ["packet-path-teardown", "mtu-drop-repro", "dataplane-migration-checklist"]
sources: 12
---

# A02.5 · Kubernetes networking

> **Concept.** A Service VIP has no listener — it is resolved by one of three dataplanes (iptables, IPVS, eBPF), and conntrack is the villain in most fleet-scale failures.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits
Lesson 04 put a price and a latency number on every hop a packet takes across a cloud boundary. This lesson follows that packet the rest of the way down: once it lands inside the cluster and hits a Service VIP, *something* has to translate that virtual address into a real pod IP, and which mechanism does that translation determines your reprogramming latency, your conntrack exposure, and your NetworkPolicy security boundary. It is also the substrate lesson 06 sits on top of (the mesh is one more hop *after* this resolution) and the substrate lesson 07 deliberately routes around (RDMA bypasses all of it).

## Why this matters
At single-cluster scale, "Services just work" and kube-proxy is invisible. At fleet scale — thousands of Services, high connection churn, GPU nodes carrying control-plane plus NCCL-fallback traffic — the *dataplane implementation* becomes the dominant variable in latency, reprogramming lag, and outages. Staff-level fluency means naming exactly what translates a VIP to a backend, where SNAT hides source IPs, and why conntrack is the recurring failure surface. This is also the substrate lesson 07 builds on: RDMA is deliberately kept *off* this path, and knowing why requires knowing precisely what this path costs.

## What's new here (calibration)
- You already know the pod-per-IP model, Service types, that a CNI plugs pods into the network, and that NetworkPolicy is default-allow-until-added — none of that is repeated here.
- **Assumed from `modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md`:** netfilter's five hooks and priority ladder, what a conntrack entry is, that NAT rides on conntrack, kube-proxy's `KUBE-SERVICES`/`KUBE-SVC`/`KUBE-SEP` chain structure and its `statistic --mode random` backend selection, the IPVS-still-uses-conntrack fact, the nftables backend's verdict map, and kube-proxy's conntrack sysctl defaults (`maxPerCore = 32768`, `min = 131072`). **This lesson does not re-derive any of that.** It builds on it.
- New: the **CNI contract itself** — the execution protocol, the JSON in and out, the operations, the error codes, and what a plugin actually does to a network namespace, at a level where you could write one.
- New: the pod-to-pod path drawn end to end for **overlay versus routed**, with byte counts, hop counts, and the MTU arithmetic that falls out.
- New: **IPAM as an address-space capacity problem** with real exhaustion arithmetic — the constraint that most often decides cluster shape.
- New: the **socket-level** translation path, which is genuinely different in kind from packet-level DNAT (not merely faster), including the observable behaviours that change when you adopt it.
- New: the **EndpointSlice → dataplane propagation chain** as a latency budget, and `trafficDistribution` with its documented safeguards and failure thresholds.

## Core concepts

### 1. The network model is a contract, and it is what makes everything else hard

Kubernetes does not specify *how* pods are networked. It specifies three properties the implementation must provide, and every design decision downstream is a consequence of them:

1. **Every pod gets its own IP address**, and containers within a pod share that address and its port space.
2. **Pods can communicate with all other pods on any node without NAT.** The address a pod sees itself as is the address other pods see it as.
3. **Agents on a node** (kubelet, system daemons) can reach all pods on that node.

The second property is the expensive one. It rules out the port-mapping model that Docker used, and it means the cluster needs a **flat, routable address space** that spans every node. That single requirement is what forces the choice in §5 (overlay or routed), the address-space arithmetic in §6, and the MTU problem in §7.

The third property is why a pod's traffic to something *outside* the pod network — the internet, a database in the VPC, another cluster — is usually **masqueraded** (SNAT'd to the node IP), because the pod CIDR is not routable out there. That masquerade is the source of both the source-IP-visibility problems in §9 and the conntrack pressure documented in 01b.7.

### 2. CNI: the contract, precisely

CNI is deliberately tiny: **the container runtime executes a binary, passes parameters in environment variables and JSON on stdin, and reads JSON from stdout.** There is no daemon, no gRPC, no library binding required. Understanding it at this level is what lets you debug "the pod is stuck in `ContainerCreating`."

**Configuration** lives in `/etc/cni/net.d/`, and is a *list* of plugin objects that run in order. Here is a real, complete `conflist` — the `type` fields name binaries in `/opt/cni/bin/`:

```jsonc
{
  "cniVersion": "1.1.0",
  "cniVersions": ["0.3.1", "0.4.0", "1.0.0", "1.1.0"],
  "name": "cbr0",
  "plugins": [
    {
      "type": "bridge",           // interface plugin: creates the veth + bridge
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,             // well-known key: masquerade this network's
                                  // traffic on the host (see §1's third property)
      "hairpinMode": true,
      "ipam": {
        "type": "host-local",     // DELEGATED plugin — bridge invokes it
        "ranges": [[{ "subnet": "10.244.3.0/24" }]],
        "routes": [{ "dst": "0.0.0.0/0" }]
      }
    },
    {
      "type": "portmap",          // CHAINED plugin: adjusts what bridge created
      "capabilities": { "portMappings": true }
    },
    {
      "type": "bandwidth",        // CHAINED: applies tc shaping from pod annotations
      "capabilities": { "bandwidth": true }
    }
  ]
}
```

**The execution protocol** (CNI spec 1.1.0, `containernetworking/cni` `SPEC.md`). Parameters arrive as environment variables:

| Variable | Meaning |
|---|---|
| `CNI_COMMAND` | `ADD`, `DEL`, `CHECK`, `GC`, or `VERSION` |
| `CNI_CONTAINERID` | runtime-allocated unique ID for the sandbox |
| `CNI_NETNS` | path to the network namespace, e.g. `/run/netns/cni-8f3a…` |
| `CNI_IFNAME` | the interface name to create *inside* the container — conventionally `eth0`; the plugin must error if it cannot use this name |
| `CNI_ARGS` | `KEY=VAL;KEY=VAL` extras from the caller |
| `CNI_PATH` | `:`-separated search path for plugin binaries |

The plugin returns a JSON **Result** on stdout containing `interfaces` (name, MAC, MTU, sandbox), `ips` (address in CIDR form, gateway, and an index into `interfaces`), `routes` (dst, gw, and optionally `mtu`, `advmss`, `priority`, `table`, `scope`), and `dns`. **Chained plugins receive the previous plugin's Result as `prevResult` in their stdin config and must emit it back**, modified or not — that is how a chain composes.

**The error codes are worth memorising**, because they turn an opaque `ContainerCreating` into a diagnosis:

| Code | Meaning | What it means for you |
|---|---|---|
| 1 | incompatible CNI version | plugin binary older than the config's `cniVersion` |
| 2 | unsupported field in config | the message names the key — usually a typo or a version mismatch |
| 3 | container unknown / does not exist | the runtime **does not need to call DEL**; safe to ignore on cleanup |
| 4 | invalid environment variables | the runtime is misconfigured |
| 5 / 6 | I/O failure / decode failure | stdin was truncated or malformed |
| 7 | invalid network config | **the IPAM-exhaustion code** — the spec's own example is literally `"Network 192.168.0.0/31 too small to allocate from."` |
| 11 | try again later | transient; the runtime should retry — this is what a busy IPAM returns instead of failing hard |
| 50 / 51 | plugin not available (51: existing containers may have limited connectivity) | the agent is down |

**`DEL` must be idempotent.** The spec requires plugins to succeed on a `DEL` for a container that does not exist, because the runtime will retry cleanup and a hard failure there leaks IP allocations. Most "IP exhaustion on a cluster with 40 pods" incidents are a plugin whose `DEL` path failed and whose IPAM store still holds thousands of reservations.

**`GC` (added in 1.1.0)** exists for exactly that: the runtime tells the plugin the complete list of attachments that *should* exist, and the plugin frees anything else. If your CNI supports it, it is the recovery path for a leaked IPAM store.

### 3. What a CNI ADD actually does to a namespace

The abstraction is thin enough that you can do it by hand, which is the fastest way to understand what a plugin is doing when it fails.

```
   CNI ADD, STEP BY STEP — bridge + host-local, one pod
   ═══════════════════════════════════════════════════════════════════

   t0  kubelet asks the CRI to create the pod sandbox.
       containerd creates a NETWORK NAMESPACE and a pause container that
       holds it open.  At this moment the namespace has ONE interface: lo,
       and it is DOWN.

   t1  containerd reads /etc/cni/net.d/10-cbr0.conflist, and execs
       /opt/cni/bin/bridge with:
           CNI_COMMAND=ADD  CNI_CONTAINERID=8f3a…  CNI_IFNAME=eth0
           CNI_NETNS=/var/run/netns/cni-2b7c…
       and the plugin object on stdin.

   t2  bridge execs the DELEGATED IPAM plugin, /opt/cni/bin/host-local,
       with the same env and the `ipam` sub-object on stdin.
       host-local picks the next free address from 10.244.3.0/24 by
       creating a file:
           /var/lib/cni/networks/cbr0/10.244.3.17   ← contains the container ID
       This FILE IS THE ALLOCATION. It is why a failed DEL leaks: the file
       stays, the address is never reused, and eventually every address in
       the /24 has a file naming a container that died last Tuesday.
       Returns: {"ips":[{"address":"10.244.3.17/24","gateway":"10.244.3.1"}]}

   t3  bridge creates the veth PAIR and moves one end into the namespace:
           ip link add veth9a1b type veth peer name eth0
           ip link set eth0 netns /var/run/netns/cni-2b7c…
           ip link set veth9a1b master cni0 up
       Inside the namespace:
           ip addr add 10.244.3.17/24 dev eth0
           ip link set eth0 up
           ip route add default via 10.244.3.1 dev eth0

   t4  bridge emits its Result on stdout. containerd passes it as
       `prevResult` to `portmap`, which adds DNAT rules for any hostPort,
       then to `bandwidth`, which attaches tc qdiscs to the HOST side of
       the veth. Each returns the (possibly modified) prevResult.

   t5  kubelet sees success and starts the application containers, which
       join the SAME namespace. This is why containers in a pod share an
       IP and a port space: they are literally in one netns.

   VERIFY IT ON A LIVE NODE:
     $ PID=$(crictl inspectp <sandbox-id> | jq -r .info.pid)
     $ nsenter -t $PID -n ip -d addr show eth0
       3: eth0@if42: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 ...
          veth
          inet 10.244.3.17/24 scope global eth0
                                    ^^^^ note the MTU — §7 is about this number
     $ nsenter -t $PID -n ip route
       default via 10.244.3.1 dev eth0
       10.244.3.0/24 dev eth0 proto kernel scope link src 10.244.3.17
     # the '@if42' names the host-side peer's ifindex:
     $ ip -d link | grep -A1 '^42:'
       42: veth9a1b@if3: ... master cni0 ...
```

**The two failure modes this walk-through explains.** A pod stuck in `ContainerCreating` with a CNI error is one of: the plugin binary is missing from `CNI_PATH` (check `/opt/cni/bin`), the config in `/etc/cni/net.d` is invalid or names a `type` that does not exist, or IPAM returned code 7 because the range is full. A pod that gets an IP but cannot reach anything is a routing or policy problem, not a CNI-ADD problem — the ADD clearly succeeded, because there is an address.

### 4. Overlay versus routed: the same packet, both ways

This is the single most useful picture in Kubernetes networking, because almost every performance and MTU question resolves to "which of these am I on."

```
   POD → POD ACROSS NODES.  10.244.3.17 (node A) → 10.244.7.42 (node B).
   Node A = 10.0.1.10, node B = 10.0.2.20.  Payload = 1400 bytes.
   ══════════════════════════════════════════════════════════════════════════

   ┌──────────────────── (a) OVERLAY — VXLAN / Geneve ────────────────────┐
   │                                                                      │
   │  NODE A                                          NODE B              │
   │  ┌────────────┐                                  ┌────────────┐      │
   │  │ pod netns  │                                  │ pod netns  │      │
   │  │ eth0       │  MTU 1450  ← note                │ eth0       │      │
   │  │ 10.244.3.17│                                  │10.244.7.42 │      │
   │  └─────┬──────┘                                  └─────▲──────┘      │
   │        │ veth                                          │ veth        │
   │  ┌─────▼──────┐                                  ┌─────┴──────┐      │
   │  │ host netns │                                  │ host netns │      │
   │  │  route:    │                                  │            │      │
   │  │ 10.244.7.0/24 dev vxlan.calico                │            │      │
   │  └─────┬──────┘                                  └─────▲──────┘      │
   │        │                                               │             │
   │  ┌─────▼───────────────┐              ┌────────────────┴────────┐    │
   │  │ VXLAN ENCAP         │              │ VXLAN DECAP             │    │
   │  │ look up the remote   │              │ strip 50 bytes,        │    │
   │  │ node's IP from the   │              │ deliver inner packet   │    │
   │  │ FDB/route, wrap:     │              └────────────▲───────────┘    │
   │  └─────┬───────────────┘                            │                │
   │        │                                            │                │
   │        │  ON THE WIRE (1450 payload + 50 = 1500):    │                │
   │        │  ┌──────────────────────────────────────┐  │                │
   │        └─▶│ eth 14 │ IP 20 │ UDP 8 │ VXLAN 8 │◀── 50 B overhead ──┐  │
   │           │  src 10.0.1.10  dst 10.0.2.20         │               │  │
   │           │  dport 4789 (VXLAN) or 6081 (Geneve)  │               │  │
   │           ├──────────────────────────────────────┤               │  │
   │           │ INNER: eth │ IP src 10.244.3.17       │               │  │
   │           │            │    dst 10.244.7.42       │  ──────────────┘  │
   │           │            │ TCP │ 1400 bytes payload │                   │
   │           └──────────────────────────────────────┘                   │
   │                                                                      │
   │  UNDERLAY REQUIREMENT: node-to-node IP reachability. That's ALL.     │
   │    Works across subnets, across cloud VPCs, over anything routable.  │
   │  COSTS:  50 bytes/packet (3.3 % of a 1500 MTU) + encap/decap CPU     │
   │          per packet, and an MTU that MUST be configured correctly.   │
   │  DEBUGGING: tcpdump on the node shows UDP/4789, not your traffic.    │
   │             You must decode: `tcpdump -i eth0 -nn 'udp port 4789'`   │
   │             and read the inner headers (tcpdump decodes VXLAN).      │
   └──────────────────────────────────────────────────────────────────────┘

   ┌────────── (b) NATIVE ROUTING — BGP, or cloud route tables ───────────┐
   │                                                                      │
   │  NODE A                                          NODE B              │
   │  ┌────────────┐                                  ┌────────────┐      │
   │  │ pod netns  │  MTU 1500 (or 9001 on a jumbo    │ pod netns  │      │
   │  │ 10.244.3.17│         underlay)                │10.244.7.42 │      │
   │  └─────┬──────┘                                  └─────▲──────┘      │
   │        │ veth                                          │ veth        │
   │  ┌─────▼──────┐                                  ┌─────┴──────┐      │
   │  │ host netns │  route: 10.244.7.0/24 via        │ host netns │      │
   │  │            │         10.0.2.20 dev eth0       │            │      │
   │  └─────┬──────┘                                  └─────▲──────┘      │
   │        │                                               │             │
   │        │  ON THE WIRE — the packet is UNCHANGED:        │            │
   │        │  ┌──────────────────────────────────────┐     │            │
   │        └─▶│ eth 14 │ IP src 10.244.3.17           │─────┘            │
   │           │        │    dst 10.244.7.42           │                  │
   │           │        │ TCP │ 1400 bytes payload     │                  │
   │           └──────────────────────────────────────┘                   │
   │                                                                      │
   │  UNDERLAY REQUIREMENT: the network must ROUTE the pod CIDRs.         │
   │    On-prem: BGP peering with the ToR (Calico's BIRD, Cilium's BGP    │
   │      control plane) advertising each node's /24.                     │
   │    Cloud:  a route-table entry per node (GKE, Azure CNI), or pod IPs │
   │      taken FROM the VPC subnet itself (AWS VPC CNI) so no routes are │
   │      needed at all.                                                  │
   │  COSTS:  underlay must cooperate; a hard cap on route-table entries  │
   │          (cloud route tables have limits, typically ~50–250 routes)  │
   │          or on VPC address space (§6).                               │
   │  BENEFITS: no encap CPU, full MTU, pod IPs are visible and pingable  │
   │          from outside the cluster, tcpdump shows real addresses.     │
   └──────────────────────────────────────────────────────────────────────┘

   THE DECISION, IN ONE LINE: overlay buys independence from the underlay
   and costs 50 bytes and an MTU footgun; routing buys the wire back and
   costs a dependency on the underlay's willingness to carry your CIDRs.
```

**"Overlays are always slower" is wrong and worth correcting precisely.** The cost is (a) 50 bytes of every packet, which is 3.3 % of a 1500-byte frame and **0.56 % of a 9000-byte jumbo frame** — Cilium's own routing documentation makes exactly this point — and (b) per-packet encapsulation CPU, which on modern NICs with VXLAN offload is largely paid in hardware. It is a measurable overhead, not a structural ceiling. What it *is* is a configuration liability, which §7 covers.

### 5. MTU: the arithmetic and the failure

```
   THE OVERLAY MTU CALCULATION
   ═════════════════════════════════════════════════════════════════
   VXLAN over IPv4:
     outer Ethernet header    14 bytes
     outer IPv4 header        20 bytes
     outer UDP header          8 bytes
     VXLAN header              8 bytes
                              ──────────
     TOTAL                    50 bytes      → pod MTU = underlay MTU − 50

   Geneve over IPv4: same 50-byte base, PLUS variable-length options
     (Geneve's whole point), so implementations budget more —
     commonly 50–66 bytes. Check your CNI's computed value; do not
     assume 50.

   VXLAN over IPv6: outer IPv6 header is 40 bytes, not 20 → 70 bytes.

   IPsec / WireGuard encryption on top of either: add again
     (WireGuard's overhead is 60 bytes for IPv4, 80 for IPv6).
     A VXLAN + WireGuard cluster is 1500 − 50 − 60 = 1390.

   COMMON UNDERLAYS AND WHAT THEY YIELD:
     underlay 1500 (most on-prem, most VPC defaults)  → pod MTU 1450
     underlay 9001 (AWS VPC jumbo within an AZ)       → pod MTU 8951
     underlay 8896 (some GKE/Azure paths)             → pod MTU 8846
     underlay 1500 with VXLAN + WireGuard             → pod MTU 1390
```

**Why a mismatch is silent and slow to diagnose.** If the pod's interface says 1500 but the path can only carry 1450, then everything small works — the TCP handshake, DNS, health checks, a `curl -I`. The first full-size data segment is 1500 bytes with DF set, does not fit, and the encapsulating node must return **ICMP type 3 code 4 with the correct MTU**. In a cluster that ICMP has to make it back through the overlay, past any NetworkPolicy, and to the original pod — and NetworkPolicy implementations that only allow TCP will *drop it*. Then you have the exact black hole from lesson 04 §8, but caused by your own policy.

```
   CONFIRM IT IN ONE COMMAND, FROM INSIDE THE POD:
   $ kubectl exec -it web-7d9 -- sh
   / # ip link show eth0 | head -1
   3: eth0@if42: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue
                                                       ^^^^ claims 1500

   / # ping -M do -s 1472 -c 2 10.244.7.42     # 1472 + 28 = 1500
   PING 10.244.7.42 (10.244.7.42): 1472 data bytes
   --- 10.244.7.42 ping statistics ---
   2 packets transmitted, 0 packets received, 100% packet loss   ← does NOT fit

   / # ping -M do -s 1422 -c 2 10.244.7.42     # 1422 + 28 = 1450
   1430 bytes from 10.244.7.42: seq=0 ttl=63 time=0.412 ms       ← fits

   DIAGNOSIS: real path MTU is 1450, the interface claims 1500, and no
   "frag needed" came back. Note what a HEALTHY misconfiguration looks
   like: the ICMP arrives, Linux caches the PMTU, and the second attempt
   at 1472 succeeds. Its ABSENCE is the bug.

   $ ip route get 10.244.7.42        # from the pod, after a failed attempt
   10.244.7.42 via 10.244.3.1 dev eth0 src 10.244.3.17
   # No `mtu 1450 expires ...` cached → PMTUD learned nothing.
```

**The three fixes, and when each applies.** (1) Set the CNI's MTU explicitly to `underlay − overhead` and restart pods to pick it up — this is the correct fix and most CNIs will auto-detect it if you let them. (2) MSS clamping on the node, which protects TCP but not UDP or QUIC. (3) `net.ipv4.tcp_mtu_probing=1` on the nodes as a fleet-wide safety net (RFC 4821 packetization-layer PMTUD, which does not depend on ICMP at all). And separately: **make sure your NetworkPolicy allows ICMP**, because a default-deny policy that forgets it converts every MTU mismatch from a self-healing annoyance into a black hole.

### 6. IPAM: the address-space arithmetic that decides cluster shape

This is the constraint people meet last and should have modelled first.

**The classic model — a per-node pod CIDR.** The cluster has one big CIDR; each node gets a slice.

```
   cluster CIDR   10.244.0.0/16   → 65,536 addresses
   node CIDR mask /24             → 256 addresses per node, minus network,
                                    broadcast, and gateway ≈ 253 usable
   number of nodes = 2^(24−16) = 2^8 = 256 NODES MAXIMUM.

   That is a hard ceiling written into the control plane at cluster
   creation. Growing past it means renumbering the cluster.

   THE TENSION:
     kubelet's default --max-pods is 110.
     A /24 per node gives 253 usable → you are wasting 143 addresses/node.
     A /25 per node gives 126 usable → still above 110, and DOUBLES the
       node ceiling to 512.
     A /26 gives 62 → now max-pods must drop below 62.

   SIZING RULE:  node CIDR must hold (max-pods + a margin for churn).
   host-local IPAM does NOT reuse an address immediately after a pod dies
   in all implementations, so leave headroom — 20 % is a common choice.

   WORKED — a 2,000-node GPU cluster at 110 pods/node
     addresses needed = 2,000 × 128 (a /25 each) = 256,000
     → cluster CIDR must be at least a /14 (262,144 addresses)
     10.0.0.0/14 with /25 node CIDRs = 2^(25−14) = 2^11 = 2,048 nodes ✓
     Check it against your VPC: if pod IPs must be VPC-routable, you have
     just committed a /14 of RFC 1918 space, which is a conversation with
     whoever owns the address plan.
```

**The VPC-native model (AWS VPC CNI and equivalents) trades a different resource.** Pod IPs come from the VPC subnet itself, so there is no overlay and no route table to maintain — but now **every pod IP consumes a real VPC address**, and the per-node pod ceiling is set by how many IPs the instance type's ENIs can hold rather than by a CIDR mask. The consequences are worth stating because they surprise people:

- Subnet exhaustion becomes a cluster-wide outage: pods stop scheduling with a CNI error, and the fix is adding subnets, which requires address space you may not have.
- The per-node pod limit is an *instance-type* property, so changing instance type changes your pod density and therefore your bin-packing.
- Prefix delegation (assigning a /28 per ENI rather than individual IPs) multiplies density but allocates in blocks, so it fragments the subnet faster.

**The diagnostic when pods will not schedule:**

```bash
# The pod event tells you which layer failed:
$ kubectl describe pod stuck-pod | tail -5
  Warning  FailedCreatePodSandBox  12s  kubelet
    Failed to create pod sandbox: plugin type="…" failed (add):
    failed to allocate for range 0: no IP addresses available in range set:
    10.244.3.1-10.244.3.254
#   ^^^ CNI error code 7 territory: the range is full.

# Is it genuinely full, or leaked? Count allocations against live pods:
$ ls /var/lib/cni/networks/cbr0/ | grep -c '^10\.'
253
$ crictl pods --state Ready -q | wc -l
41
#   253 allocations, 41 running pods → 212 LEAKED. The DEL path is broken.
#   Recovery: stop the kubelet, remove the stale files whose container IDs
#   no longer exist, restart. Or use CNI GC if the plugin supports it.
```

### 7. The VIP, and the one dataplane difference that is a difference in *kind*

A ClusterIP is an address with **nothing bound to it anywhere**. It exists only as an entry in a translation table. Which table, and where the translation happens, is the whole question.

`modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md` covers the iptables chain structure, the IPVS virtual-server table, the nftables verdict map, and the fact that **all three create a conntrack entry per flow**, in detail. This section covers only what that lesson does not: the socket-level path, which is not a faster version of the same thing.

**Packet-level DNAT (iptables, IPVS, nftables).** A packet exists. It is addressed to `10.96.140.22:80`. Netfilter's `nat` table rewrites the destination to `10.244.7.19:8080` and conntrack remembers the translation so the reply can be un-translated. Every flow costs one conntrack entry, and the entry's lifetime is governed by the timeouts kube-proxy sets.

**Socket-level translation (Cilium's socket-LB, the basis of its kube-proxy replacement).** An eBPF program is attached to the **cgroup** at the `connect(2)` and `sendmsg(2)` hooks. When the application calls `connect()` to `10.96.140.22:80`, the program rewrites the destination *in the socket's address structure* before any packet is constructed. The kernel then builds a packet addressed directly to `10.244.7.19:8080`. **There is no translation to undo, so there is no NAT conntrack entry to create.**

That distinction produces a set of observable behaviours you must know before adopting it:

| Behaviour | Packet-level DNAT | Socket-level translation |
|---|---|---|
| Where translation happens | netfilter `nat` hook, per packet | `connect()`/`sendmsg()` cgroup hook, per socket |
| conntrack entry for east-west | yes, one per flow | none for the translation itself |
| `getpeername()` on the client socket returns | the **VIP** | the **backend pod IP** — application code that logs or asserts on the peer address will report something different after migration |
| `getsockname()` on the server | the pod IP either way | the pod IP either way |
| Session affinity source | client source IP | the client's **network-namespace cookie**, because no source IP exists yet at `connect()` time (Cilium documents this explicitly) |
| Maglev consistent hashing | applies to all traffic | applies to **north-south only**; east-west sockets are assigned at connect time and stay put, so they are not subject to Maglev |
| Backend removal | the flow's conntrack entry is torn down | Cilium force-terminates sockets connected to removed backends, using the `cilium_lb4_reverse_sk` map to verify the socket was socket-LB'd |
| Debugging command | `conntrack -L`, `iptables-save`, `ipvsadm -Ln` | `cilium-dbg service list`, `cilium monitor -v` — **`conntrack -L` shows nothing, which is not the same as "no traffic"** |

**Two documented sharp edges**, both from Cilium's own kube-proxy-free documentation:

- **Reverse-SK map exhaustion.** The maps used to verify that a socket was socket-LB'd before force-terminating it are LRU. Unconnected UDP sockets (`create → sendto() → close` without `connect()`) and aborted connected UDP sockets do not get cleaned up properly, so on nodes with such workloads the maps gradually fill and socket termination becomes unreliable until the node restarts. The documented mitigation is `socketLB.hostNamespaceOnly=true`. **UDP-heavy GPU control planes are exactly the workload that triggers this**, so it is worth checking.
- **Backend termination is racy at the Kubernetes level, not the CNI level.** Cilium's docs state it plainly: deleting a pod triggers the EndpointSlice update and the kubelet's SIGTERM **simultaneously**, and as of Kubernetes 1.36 the kubelet cannot coordinate with EndpointSlice updates. This is the same race lesson 03 §8 addressed with a `preStop` hook, and it is architectural, not a bug in any one CNI.

### 8. The propagation chain: what "reprogramming latency" is made of

When a pod becomes Ready, how long until traffic reaches it? The answer is a chain, and each link has its own budget.

```
   POD READY → TRAFFIC ARRIVES: THE FULL PROPAGATION CHAIN
   ═══════════════════════════════════════════════════════════════════════

   [1] kubelet marks the pod Ready
        │  (readiness probe period + initialDelaySeconds — YOURS to set)
        ▼
   [2] API server persists the Pod status to etcd
        │  ~1–10 ms normally; grows with etcd load and object size
        ▼
   [3] EndpointSlice controller's informer receives the watch event
        │  informer resync + workqueue latency
        │  ConcurrentServiceEndpointSyncs = 5 (default) — FIVE workers for
        │  the whole cluster's Services. On a cluster with heavy churn this
        │  is a real queue, and `endpoint_slice_controller_syncs` /
        │  workqueue depth metrics are where you see it back up.
        │  EndpointUpdatesBatchPeriod (default 0) can deliberately delay
        │  updates to BATCH them — trading latency for fewer writes.
        ▼
   [4] Controller writes the updated EndpointSlice
        │  MaxEndpointsPerSlice = 100 by default (configurable to 1000).
        │  THIS IS THE SHARDING THAT MAKES IT SCALE: a 2,000-endpoint
        │  Service is 20 objects, so one pod change rewrites ~1/20th of
        │  the data instead of a single 2,000-entry Endpoints object.
        ▼
   [5] EVERY kube-proxy / CNI agent ON EVERY NODE receives the watch event
        │  N nodes × the object size = the fan-out cost. This is why slice
        │  size matters: it multiplies by the node count.
        ▼
   [6] The agent reprograms its dataplane
        │  iptables:  regenerate and `iptables-restore` the ruleset —
        │             cost grows with TOTAL service count (01b.7 §7)
        │  IPVS:      incremental add/remove of one real server — O(1)
        │  eBPF:      update one map entry — O(1)
        ▼
   [7] Traffic arrives.

   THE ARITHMETIC THAT MATTERS AT SCALE
   ─────────────────────────────────────────────────────────────────────
   5,000 Services × 10 endpoints = 50,000 endpoints
     → 50,000 / 100 = 500 EndpointSlice objects
   A 1,000-pod rolling deploy touches ~1,000 endpoints:
     → with MaxEndpointsPerSlice=100, ~10–1,000 slice writes depending on
       how the churn distributes across slices
     → each write is watched by every node: 2,000 nodes × 500 slices is
       the steady-state watch load; the DELTA is what churn costs
   Raising MaxEndpointsPerSlice to 1000 gives 50 objects instead of 500 —
     fewer, larger writes. Fewer API objects, but each change ships 10×
     more data to every node. THE TRADE IS OBJECT COUNT VS UPDATE SIZE,
     and the right answer depends on whether your bottleneck is etcd
     object count or watch bandwidth. Measure before changing it.
```

### 9. Traffic policies and topology: keeping bytes local

Three separate fields, frequently confused:

| Field | Values | What it does | Cost |
|---|---|---|---|
| `externalTrafficPolicy` | `Cluster` (default) / `Local` | for traffic entering via NodePort or LoadBalancer: `Cluster` may forward to a pod on another node (SNAT'ing, so the **client IP is lost**); `Local` only uses node-local pods, **preserving the client IP** | `Local` gives uneven load — a node with 3 pods gets the same share as a node with 1 — and drops traffic on nodes with zero pods (which is what the LB health-check nodePort is for) |
| `internalTrafficPolicy` | `Cluster` (default) / `Local` | same idea for **in-cluster** traffic to a ClusterIP: `Local` restricts to pods on the same node | traffic **fails** if there is no local pod — this is a strict semantic, not a preference |
| `trafficDistribution` | `PreferSameZone`, `PreferSameNode`, `PreferClose` (deprecated alias for `PreferSameZone`) | a **preference**, not a guarantee: route to topologically closer endpoints when possible, fall back otherwise | none semantically; it can still route cross-zone |

**`trafficDistribution` is the one that connects to lesson 04's bill.** It is how you stop paying cross-AZ transfer for east-west traffic. The mechanism: the EndpointSlice controller populates a `hints.forZones` field on each endpoint, and kube-proxy filters on it.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments
spec:
  selector: { app: payments }
  ports: [{ port: 80, targetPort: 8080 }]
  trafficDistribution: PreferSameZone   # the modern field
---
# What the controller then writes onto the slice:
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: payments-abc12
  labels:
    kubernetes.io/service-name: payments
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 8080
endpoints:
  - addresses: ["10.244.3.17"]
    conditions: { ready: true }
    zone: eu-west-1a
    hints:
      forZones: [{ name: "eu-west-1a" }]    # ← kube-proxy filters on this
```

**The safeguards, which are the part that decides whether it works** (from the Kubernetes topology-aware-routing documentation):

- The allocation is proportional to each zone's **allocatable CPU cores**, not to its endpoint count. A zone with twice the CPU gets twice the endpoints allocated to it.
- **You need at least 3 endpoints per zone** — 9 in a three-zone cluster. Below that there is roughly a **50 % chance** the controller cannot allocate evenly and falls back to cluster-wide routing, silently.
- If there are **fewer endpoints than zones**, no hints are assigned at all.
- If a balanced allocation is impossible (say zone-a has twice the capacity of zone-b but there are only 2 endpoints), the controller refuses to assign hints rather than create a known overload. **This is not based on real-time feedback**, so individual endpoints can still become overloaded.
- The feature assumes **incoming traffic is evenly distributed across zones**. If most traffic originates in one zone, that zone's endpoint subset gets all of it. The documentation explicitly recommends against the feature in that case.

**The practical consequence:** topology-aware routing is a good fit for large, evenly-loaded stateless tiers and a bad fit for small Services — which is exactly the opposite of where people usually reach for it first, because a small Service is where the cross-AZ percentage looks worst.

### 10. NetworkPolicy: two enforcement models, one security boundary

The API is uniform; the enforcement is not, and the difference is a security property rather than a performance one.

**The semantics, exactly.** A pod is *unaffected* by NetworkPolicy until at least one policy selects it. Once any policy selects a pod for a direction (ingress or egress), that direction becomes **default-deny for that pod**, and only the union of matching policies' `allow` rules gets through. Policies are additive; there is no deny rule and no ordering. The standard cluster-hardening move is therefore a per-namespace default-deny that selects everything:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: training
spec:
  podSelector: {}                # selects EVERY pod in the namespace
  policyTypes: ["Ingress", "Egress"]
  # no ingress/egress rules → nothing is allowed
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-api
  namespace: training
spec:
  podSelector: { matchLabels: { app: trainer } }
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }   # ← do not forget TCP: DNS falls back
                                        #   to TCP on truncation, and NodeLocal
                                        #   DNSCache forwards over TCP (lesson 02)
    - to:
        - ipBlock: { cidr: 10.0.0.0/16 }
      ports:
        - { protocol: TCP, port: 443 }
    # NOTE: ICMP cannot be expressed in the core NetworkPolicy API at all.
    # A default-deny egress policy therefore blocks the "fragmentation
    # needed" messages that PMTU discovery depends on (§5). CNIs with
    # their own CRD (CiliumNetworkPolicy) can express ICMP; the core API
    # cannot, which is a real gap you must design around.
```

**Model A — IP-based enforcement.** The controller resolves label selectors to a set of pod IPs and programs rules matching those IPs. Correct in steady state, and racy under churn: when a pod dies its IP is freed and may be reassigned to a *different* pod in a *different* namespace before every node's rules have been updated. During that window, traffic is authorised against the wrong identity. The window is the propagation chain in §8, which is milliseconds normally and seconds on a loaded cluster — and pod churn is highest during exactly the events (rollouts, autoscaling) where you most want the boundary to hold.

**Model B — identity-based enforcement (Cilium).** Each unique set of security-relevant labels is assigned a cluster-wide numeric **identity**. The identity travels with the packet (in the encapsulation header on an overlay, or is derived from an IP→identity map on a routed datapath), and the policy decision is made on identity, never on a point-in-time IP. Reassigning an IP to a new pod gives that pod a *different identity*, so there is no window of incorrect authorisation. The cost is a cluster-wide identity allocation service and a bounded identity space — a workload that generates unbounded label combinations can exhaust it, which is its own (well-signposted) failure mode.

**And the one you must actually check:** some CNIs implement no NetworkPolicy enforcement at all. Applying a policy succeeds — the API object is valid regardless — and nothing enforces it. **Verify enforcement with a test, not with a manifest.** The test is two pods and a `curl` that should fail.

### 11. The GPU-fleet tie

Everything in this lesson describes the **primary** pod network, which on a GPU cluster carries control-plane traffic, storage, checkpoint I/O, and NCCL's TCP fallback. RDMA does not ride it, and the reasons are now all things you can name precisely:

- **Conntrack.** Every NATed flow costs an entry (01b.7). A collective's flow count and rate would be pathological on that table.
- **MTU.** A 50-byte overlay overhead against RDMA's need for large MTUs (4 KB IB MTU is typical) is not a tuning question; encapsulation is simply incompatible with the datapath's design.
- **The kernel itself.** §4's diagrams show a veth pair, a bridge, and a routing decision per packet. RDMA's entire value is deleting those stages (lesson 07 and module 09).
- **Policy enforcement.** eBPF or iptables policy evaluation per packet is exactly the per-packet CPU cost RDMA exists to remove.

So a GPU node runs **two** networks: this one, on `eth0`, doing everything ordinary; and a secondary RDMA network attached through Multus and an SR-IOV VF or DRA claim, which lesson 07 covers. The reason this lesson matters to lesson 07 is that "RDMA bypasses the CNI" is only a meaningful statement once you can enumerate what is being bypassed.

## Perspectives

**Dataplane-migration.** Moving a live fleet between dataplanes is a project, not a Helm value. It needs a canary node pool, explicit feature-parity checks (NetworkPolicy enforcement model, `externalTrafficPolicy: Local`, DSR, hostPort, session affinity semantics), and a rollback plan — because a fleet where some nodes translate at the socket and others at the packet is an asymmetric-failure surface that produces "works from this pod, not that one" tickets. Add one check most people miss: grep the application code for `getpeername()`, because socket-level translation changes what it returns.

**Observability-parity.** The commands that answer "why is this Service unreachable" are dataplane-specific: `iptables-save`/`conntrack -L` for the netfilter backends, `ipvsadm -Ln` for IPVS, `cilium-dbg service list`/`cilium monitor -v` for eBPF. Running the wrong playbook produces a confidently wrong answer rather than an error — an empty `conntrack -L` on a socket-LB cluster means "no conntrack entry exists by design," not "no traffic."

**Multi-tenancy / identity.** Identity-based enforcement closes a real window that IP-based enforcement leaves open during pod churn, and that window is widest during rollouts and autoscaling — precisely when a shared cluster is most dynamic. On a multi-tenant cluster this is a security-boundary decision that belongs in a threat model, not a performance footnote. It is also the argument for why L3/L4 NetworkPolicy and L7 mesh authorisation are defence in depth rather than substitutes (lesson 06).

**GPU-fleet.** Address space and conntrack sizing on GPU nodes are different capacity problems from the web tier: fewer, larger, longer-lived pods, but bursty control-plane connection churn concentrated at job start, checkpoint, and rank restart — the moments that matter most. Size the conntrack table and the pod CIDR for the burst, and keep the collective traffic entirely off this path.

## Real-world use cases

- **The CNI 1.1.0 specification's `GC` operation and idempotent `DEL` requirement** (`containernetworking/cni`, `SPEC.md`), read directly. What it shows: IPAM leaks are a *known*, designed-for failure mode, not an exotic bug. The spec mandates that `DEL` succeed for a non-existent container precisely because runtimes retry cleanup, and `GC` was added in 1.1.0 so a runtime can hand a plugin the authoritative list of live attachments and have it free everything else. If your CNI supports `GC`, that is the recovery path for the leaked-allocation scenario in §6; if it does not, your recovery is manual file surgery in the IPAM store.
- **Cilium's documented socket-LB caveats** (`Documentation/network/kubernetes/kubeproxy-free.rst`), read directly. Substance used here: session affinity uses the client's **network-namespace cookie** rather than a source IP, because at `connect()` time no packet exists; Maglev applies to north-south traffic only, since east-west sockets are pinned at connect time; and the `cilium_lb4_reverse_sk` / `cilium_lb6_reverse_sk` LRU maps can fill on nodes with unconnected-UDP workloads, making socket termination unreliable until restart, with `socketLB.hostNamespaceOnly=true` as the documented mitigation. These are the kind of caveats that only appear in an upstream doc and never in a migration blog post.
- **Kubernetes topology-aware routing's documented safeguards** (`kubernetes/website`, `topology-aware-routing.md`), read directly. Substance: allocation is proportional to each zone's **allocatable CPU**, not endpoint count; below **3 endpoints per zone** there is roughly a **50 % chance** the controller cannot allocate evenly and silently falls back to cluster-wide routing; no hints are assigned at all when endpoints are fewer than zones; and the feature is explicitly **not recommended** when traffic originates predominantly from one zone. This is a rare case of upstream documentation stating its own failure envelope precisely, and it flatly contradicts the common instinct to enable it on small Services first.
- **The simultaneous EndpointSlice-update and SIGTERM race**, documented in Cilium's kube-proxy-free page as an architectural limitation still present **as of Kubernetes 1.36**. What it shows: the connection drops during pod termination that lesson 03 §8 addressed with a `preStop` hook are not a CNI bug and are not fixable inside any one CNI — the kubelet and the EndpointSlice controller genuinely do not coordinate. Any runbook that blames the CNI for deploy-time resets is chasing the wrong layer.

## Worked example

**Scenario.** A 400-node cluster on a VXLAN overlay reports: large HTTP responses from one namespace hang, DNS to an external name intermittently fails, and a newly scaled Deployment takes ~40 seconds to receive traffic. Three symptoms, three different layers. Bisect all three.

**Step 0 — establish which dataplane you are on, because it changes every subsequent command.**

```bash
$ kubectl get ds -n kube-system -o custom-columns=NAME:.metadata.name | grep -Ei 'kube-proxy|cilium|calico'
cilium
# kube-proxy is ABSENT → this is a kube-proxy replacement. conntrack-based
# commands will be misleading for east-west traffic.

$ kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose | sed -n '/KubeProxyReplacement/,/^$/p'
KubeProxyReplacement Details:
  Status:                True
  Socket LB:             Enabled
  Protocols:             TCP, UDP
  Devices:               eth0 (Direct Routing)
  Mode:                  SNAT
  Backend Selection:     Random
  Session Affinity:      Enabled
  Graceful Termination:  Enabled
  Services:
  - ClusterIP:      Enabled
  - NodePort:       Enabled (Range: 30000-32767)
  - LoadBalancer:   Enabled
#   Socket LB Enabled → getpeername() returns backend IPs; conntrack -L
#   will show nothing for east-west; use `cilium-dbg service list`.
```

**Symptom 1 — large responses hang. This is MTU.**

```bash
$ kubectl exec -n web deploy/api -- ip link show eth0 | head -1
3: eth0@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
#                                                   ^^^^ suspicious on an overlay

$ kubectl get node node-42 -o jsonpath='{.status.addresses}' >/dev/null
$ kubectl debug node/node-42 -it --image=nicolaka/netshoot -- ip link show eth0 | head -1
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP
#   underlay is 1500. VXLAN takes 50. Pods should be 1450, not 1500.

$ kubectl exec -n web deploy/api -- ping -M do -s 1422 -c1 10.244.7.42 >/dev/null && echo "1450 OK"
1450 OK
$ kubectl exec -n web deploy/api -- ping -M do -s 1472 -c1 10.244.7.42
--- 10.244.7.42 ping statistics ---
1 packets transmitted, 0 packets received, 100% packet loss
#   Path MTU is 1450; the interface claims 1500; and NOTHING came back.

# WHY did nothing come back? Check the policy.
$ kubectl get networkpolicy -n web default-deny-all -o jsonpath='{.spec.policyTypes}'
["Ingress","Egress"]
#   Default-deny with no way to express ICMP in the core API → the
#   "fragmentation needed" message is dropped → PMTUD is blind → black hole.
```

**Fix, in the right order:** set the CNI's MTU to 1450 (or let it auto-detect) and restart the affected pods; add `net.ipv4.tcp_mtu_probing=1` on the nodes as a permanent safety net so a future mismatch degrades instead of hanging; and use a CNI-native policy CRD to allow ICMP type 3, since the core API cannot express it. Note that the first fix alone would have been enough *today* and the second is what stops this recurring.

**Symptom 2 — intermittent external DNS failures. This is not MTU and not the CNI.**

```bash
$ kubectl exec -n web deploy/api -- sh -c 'for i in $(seq 100); do
    getent hosts s3.eu-west-1.amazonaws.com >/dev/null || echo FAIL; done' | wc -l
7
#   7 % failure on an EXTERNAL name only; in-cluster names are fine.

$ kubectl get networkpolicy -n web -o yaml | grep -A6 'ports:'
      ports:
        - protocol: UDP
          port: 53
#   UDP 53 allowed. TCP 53 NOT allowed.
```

**That is the bug.** Lesson 02 explains why it bites *intermittently*: a DNS response larger than the UDP payload limit comes back truncated (TC bit set), and the resolver retries the same query **over TCP** — which the policy drops. Small answers succeed; large ones (a big RRset, a long CNAME chain, an EDNS-padded reply) fail. It is not random, it is answer-size-dependent. Add `- protocol: TCP, port: 53` to the egress rule. If the cluster runs NodeLocal DNSCache, TCP 53 is not optional at all, because its upstream hop is `force_tcp`.

**Symptom 3 — 40 seconds from scale-up to traffic. This is the propagation chain.**

```bash
# Time each link. Start a scale-up and watch the stages.
$ kubectl scale deploy/api -n web --replicas=60 && date +%s.%N
1770000100.4

$ kubectl get pod -n web -l app=api --watch -o custom-columns=\
NAME:.metadata.name,READY:.status.conditions[?\(@.type==\"Ready\"\)].status | grep True | head -1
api-7d9f-x2m4   True        # t = 1770000112 → 12 s to Ready (probe config)

$ kubectl get endpointslice -n web -l kubernetes.io/service-name=api \
    -o jsonpath='{.items[*].endpoints[*].addresses[0]}' | tr ' ' '\n' | wc -l
# poll this until the new address appears → t = 1770000114 → +2 s

$ kubectl -n kube-system exec ds/cilium -- cilium-dbg service list | grep 10.96.140.22
# poll until the new backend appears → t = 1770000116 → +2 s

# TOTAL from `scale` to serving: ~16 s. The reported 40 s is NOT this path.
```

**So where do the other 24 seconds go?** Check the readiness probe and the pull:

```bash
$ kubectl get deploy -n web api -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}' | jq
{ "httpGet": {"path":"/healthz","port":8080},
  "initialDelaySeconds": 20,     ← 20 s of the 40, doing nothing
  "periodSeconds": 10 }          ← up to 10 s more before the first check
$ kubectl describe pod -n web api-7d9f-x2m4 | grep -E 'Pulling|Pulled'
  Normal  Pulling  ... Pulling image "…:v41"
  Normal  Pulled   ... Successfully pulled image in 3.9s
```

**The lesson.** `initialDelaySeconds: 20` plus `periodSeconds: 10` is up to 30 seconds of self-inflicted delay, dwarfing the ~4 seconds the whole control-plane propagation chain actually took. **Measure the chain before blaming it.** The correct fix here is a `startupProbe` with a short `periodSeconds` for the slow-start case and a fast `readinessProbe` afterwards — not tuning kube-proxy, not raising `MaxEndpointsPerSlice`, not migrating dataplanes.

**Step 4 — the summary a staff engineer produces.**

| Symptom | Layer | Confirming command | Root cause | Fix |
|---|---|---|---|---|
| Large responses hang | overlay MTU + policy | `ping -M do -s 1422/1472` from the pod | pod MTU 1500 on a 1450 path, and ICMP dropped by default-deny | set CNI MTU 1450; `tcp_mtu_probing=1`; allow ICMP via CNI CRD |
| External DNS flaps ~7 % | NetworkPolicy | `getent hosts` loop + read the policy's `ports` | TCP 53 not allowed; truncated answers retry over TCP | add TCP 53 to the egress rule |
| 40 s to serve traffic | application config | time each propagation link | `initialDelaySeconds: 20` + `periodSeconds: 10` | `startupProbe`; the dataplane was never the problem |

## Practice
<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

**Task A — do a CNI ADD by hand.** On a kind or k3s node, pick a running pod, find its netns via `crictl inspectp`, and reproduce the veth/address/route setup manually in a *new* namespace using `ip netns`, `ip link add … type veth`, and `ip route`. Then delete it. The point is to see that a CNI plugin is a shell script's worth of work plus IPAM bookkeeping. Record the exact commands.

**Task B — reproduce the MTU black hole and all three fixes.** On a two-node cluster with a VXLAN overlay: (1) set a pod's `eth0` MTU to 1500 with `ip link set eth0 mtu 1500`; (2) `curl` a multi-megabyte response from a pod on the other node and capture the repeated oversized retransmission with `tcpdump`; (3) confirm the path MTU with the `ping -M do` binary search; (4) fix it three ways — correcting the MTU, MSS clamping on the node, and `net.ipv4.tcp_mtu_probing=1` — and record how each behaves (instant fix vs TCP-only fix vs delayed self-heal). (5) Then add a default-deny NetworkPolicy and show that PMTUD stops working even when the MTU is *correctly* configured but a middle hop is smaller.

**Task C — measure your propagation chain.** Instrument the §8 chain end to end: timestamp `pod Ready`, `EndpointSlice updated`, and `dataplane programmed` (via `cilium-dbg service list`, `ipvsadm -Ln`, or `iptables-save`, whichever applies). Run it 20 times and report p50/p99 per link. Then repeat with the Service scaled to 500 endpoints and again with `--max-endpoints-per-slice=1000`, and show what changed and what did not.

**Task D — the address-space model.** For a cluster you actually run (or a hypothetical 2,000-node GPU cluster), compute: required cluster CIDR for your target node count and `max-pods`, the node CIDR mask that fits, the resulting node ceiling, and the headroom. Then deliberately exhaust a node's range in a test cluster and capture the exact `FailedCreatePodSandBox` message and the CNI error code. Show the leaked-allocation diagnosis (`ls /var/lib/cni/networks/<net>/ | wc -l` versus live pod count).

**Task E — prove NetworkPolicy is enforced.** Do not assume. Deploy two pods in different namespaces, apply a default-deny, and demonstrate with `curl` that the connection is refused/timed out. Then check whether it is *identity*- or *IP*-based: delete a pod, note its IP, force a new pod in a different namespace onto that IP if you can, and reason about the window. Write down which model your CNI uses and how you determined it.

**Acceptance criteria**

- [ ] A hand-executed CNI ADD/DEL transcript with every command and its effect.
- [ ] An MTU black hole reproduced, packet-captured, and fixed three ways, with the NetworkPolicy/ICMP interaction demonstrated separately.
- [ ] Measured p50/p99 for each link of the propagation chain, at two Service sizes and two slice sizes.
- [ ] A written address-space model with the node ceiling derived, plus a reproduced IPAM exhaustion and its error code.
- [ ] A demonstrated NetworkPolicy enforcement test and a stated determination of the enforcement model.
- [ ] A runbook page: "Service unreachable / intermittent," beginning with the **which-dataplane-am-I-on** check, then branching to VIP-not-programmed, MTU, policy drop, and IPAM exhaustion — each with the exact confirming command and its expected output.
- [ ] A one-page dataplane-migration checklist (canary cohort, feature-parity list including `getpeername()` behaviour, rollback trigger).

## Common pitfalls
- **Debugging with the wrong dataplane's tools.** An empty `conntrack -L` on a socket-LB cluster means no conntrack entry exists *by design*, not that there is no traffic. Establish the dataplane first; every subsequent command depends on it.
- **Assuming `getpeername()` returns the Service VIP.** Under socket-level translation it returns the **backend pod IP**, because the rewrite happened in `connect()` before any packet existed. Application code that logs, asserts on, or authorises based on the peer address changes behaviour on migration — grep for it before you migrate.
- **Setting the pod MTU to the underlay MTU on an overlay.** VXLAN takes 50 bytes (70 over IPv6); WireGuard adds 60 more. The symptom is a black hole on large payloads only, and it survives every liveness check you have.
- **Writing a default-deny NetworkPolicy and forgetting TCP 53.** DNS falls back to TCP whenever an answer is truncated, and NodeLocal DNSCache forwards its cluster-domain upstream hop over TCP by design. UDP-only egress produces answer-size-dependent, therefore *intermittent*, resolution failures.
- **Forgetting that the core NetworkPolicy API cannot express ICMP.** A default-deny egress policy therefore silently disables Path MTU Discovery, converting every future MTU mismatch into a hang. Use a CNI-native CRD, or accept the exposure knowingly.
- **Believing NetworkPolicy is enforced because the object applied.** Validation is schema-level. Some CNIs no-op enforcement entirely. Verify with a connection test.
- **Sizing the cluster CIDR from today's node count.** The node ceiling is `2^(nodeCIDRMaskSize − clusterCIDRMaskSize)` and it is fixed at cluster creation. A /16 cluster CIDR with /24 node CIDRs caps you at 256 nodes, forever, and the remedy is a new cluster.
- **Treating IPAM exhaustion as "we need a bigger range."** Count allocations against live pods first: `ls /var/lib/cni/networks/<net>/ | wc -l` versus `crictl pods -q | wc -l`. A large gap is a leaked `DEL` path, and a bigger range only postpones it.
- **Enabling topology-aware routing on a small Service.** It needs ≥3 endpoints per zone (9 in a three-zone cluster) or it falls back to cluster-wide routing with roughly 50 % probability, silently. It also assumes evenly distributed incoming traffic, and the documentation explicitly recommends against it when traffic originates predominantly in one zone.
- **Blaming the CNI for connection resets during deploys.** The EndpointSlice update and the kubelet's SIGTERM are triggered **simultaneously** and, as of Kubernetes 1.36, cannot coordinate. That is architectural. The fix is a `preStop` hook sized to measured propagation (lesson 03 §8), not a CNI change.

## Self-check

**1. A ClusterIP has nothing listening on it. Name the two structurally different ways a packet reaches a pod, and one observable behaviour that changes between them.**

**Packet-level DNAT**: a packet addressed to the VIP is rewritten in netfilter's `nat` hook to a backend pod IP, and conntrack stores the translation so replies can be reversed — this is what iptables, IPVS, and nftables kube-proxy backends all do, and all three cost one conntrack entry per flow (see `modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md`). **Socket-level translation**: an eBPF program on the cgroup `connect(2)`/`sendmsg(2)` hooks rewrites the destination in the socket's address structure *before any packet is built*, so the kernel constructs a packet addressed directly to the backend and there is no translation to undo — hence no NAT conntrack entry. The observable difference to name: **`getpeername()` on the client socket returns the VIP under packet-level DNAT and the backend pod IP under socket-level translation.** Two more: session affinity uses the client's network-namespace cookie rather than a source IP (there is no source IP yet at `connect()` time), and Maglev consistent hashing applies only to north-south traffic because east-west sockets are pinned at connect time.

**2. Overlay MTU: 1500 − VXLAN = ? What symptom does a mismatch produce, and why does it survive every health check?**

1500 − 50 = **1450**. The 50 bytes are outer Ethernet (14) + outer IPv4 (20) + UDP (8) + VXLAN (8); over IPv6 the outer header is 40 rather than 20, so it is 70; add another 60 (IPv4) or 80 (IPv6) if WireGuard encryption is layered on. The symptom is that **small packets work and large ones vanish**: the TCP handshake, DNS, and every readiness probe fit comfortably, so everything reports healthy, while the first full-size data segment is dropped with DF set. Recovery depends on an ICMP type 3 code 4 "fragmentation needed" reaching the sender, and two things commonly prevent that in a cluster — a NetworkPolicy default-deny (the core NetworkPolicy API cannot express ICMP at all) or an ICMP-filtering middlebox. The result is a black hole where the connection establishes and then hangs until `tcp_retries2` kills it. Confirm with `ping -M do -s 1422` (fits) versus `-s 1472` (does not), and note that the *absence* of a "frag needed" response is itself the diagnosis. Fix by setting the CNI MTU correctly, allowing ICMP through a CNI-native policy CRD, and setting `net.ipv4.tcp_mtu_probing=1` as a fleet-wide net.

**3. Your cluster CIDR is 10.244.0.0/16 with /24 node CIDRs and `--max-pods=110`. How many nodes can you run, how much address space are you wasting, and what would you change?**

Nodes = `2^(24 − 16) = 2^8 = **256**`, and that ceiling is fixed at cluster creation — growing past it means renumbering. Each /24 holds 256 addresses, of which about 253 are usable after network, broadcast, and gateway, against `max-pods = 110` — so roughly **143 addresses per node are permanently unused**, 36,608 across the cluster. Changing the node CIDR mask to **/25** gives 126 usable addresses (comfortably above 110 with churn headroom) and doubles the node ceiling to `2^(25 − 16) = 512`. For a 2,000-node target you need `2,000 × 128 = 256,000` addresses, which is a **/14** (262,144); with /25 node CIDRs a /14 supports `2^(25 − 14) = 2,048` nodes. Leave headroom for IPAM that does not immediately reuse a freed address — 20 % is a common margin. And if pod IPs must be routable in the VPC (a VPC-native CNI), that /14 comes out of your real RFC 1918 plan, which is a conversation with whoever owns it rather than a cluster-creation flag.

**4. Pods stop scheduling with "no IP addresses available in range set." What do you check before asking for a bigger CIDR?**

Whether the range is genuinely full or leaking. `host-local` IPAM records each allocation as a **file** named after the address in `/var/lib/cni/networks/<network>/`, containing the owning container ID. Compare the file count against the live pod count: `ls /var/lib/cni/networks/cbr0/ | grep -c '^10\.'` versus `crictl pods --state Ready -q | wc -l`. A large gap means the plugin's `DEL` path failed and never freed the reservations. The CNI spec explicitly requires `DEL` to succeed even for a container that no longer exists — precisely because runtimes retry cleanup and a hard failure there leaks — and spec 1.1.0 added a `GC` operation so a runtime can hand the plugin the authoritative list of live attachments and have it free the rest. Recovery is `GC` if your plugin supports it, otherwise removing stale files whose container IDs are gone. Only after that does a bigger range make sense; enlarging it while the leak persists just postpones the incident.

**5. You enable `trafficDistribution: PreferSameZone` on a Service with 6 endpoints across 3 zones to cut cross-AZ transfer. Why might nothing change, and what would make it work?**

Because the EndpointSlice controller's safeguards refuse to assign hints when it cannot allocate evenly. The documented thresholds: the feature wants **at least 3 endpoints per zone** — 9 in a three-zone cluster — and below that there is roughly a **50 % chance** the controller cannot allocate evenly and falls back to cluster-wide routing, silently; if there are fewer endpoints than zones, no hints are assigned at all; and if a balanced allocation is impossible given each zone's **allocatable CPU** (which is what the proportional allocation is based on, not endpoint count), the controller declines rather than knowingly overloading a zone. With 6 endpoints across 3 zones you are at 2 per zone, below the threshold. To make it work: scale to ≥9 endpoints, ensure they are spread across zones (topology spread constraints), and confirm that **incoming traffic is itself evenly distributed across zones** — the documentation explicitly recommends against the feature when traffic originates predominantly from one zone, because that zone's endpoint subset then absorbs all of it. If you need a *guarantee* rather than a preference, `internalTrafficPolicy: Local` is the strict version — and it fails traffic outright when there is no local pod, which is a different tradeoff entirely.

**6. Why is IP-based NetworkPolicy enforcement a security concern that identity-based enforcement closes, and when is the window widest?**

IP-based enforcement resolves label selectors to pod IPs and programs rules against those IPs. Pod IPs are recycled: when a pod dies its address returns to the pool and can be assigned to a different pod, in a different namespace, with different labels — **before every node has received and applied the updated rules**. During that window traffic is authorised against the wrong identity. The window is exactly the propagation chain of §8 (API server → EndpointSlice/policy controller → per-node watch → dataplane reprogram), which is milliseconds on an idle cluster and seconds on a loaded one. It is widest during **rollouts and autoscaling events**, when pod churn is highest — the same moments when a shared cluster is most dynamic and you most want the boundary to hold. Identity-based enforcement (Cilium) assigns a cluster-wide numeric identity to each unique label set and carries it with the packet, so the decision never depends on a point-in-time IP mapping; a recycled IP belonging to a new pod carries a *different* identity and is evaluated correctly. The cost is a cluster-wide identity allocation service and a bounded identity space that an application generating unbounded label combinations can exhaust.

**7. A pod is stuck in `ContainerCreating` with a CNI error. Walk the diagnosis.**

Read the actual error from `kubectl describe pod`, then map it to the CNI protocol. The runtime execs a plugin binary named by `type` in the conflist, found on `CNI_PATH` (normally `/opt/cni/bin`), passing `CNI_COMMAND=ADD`, `CNI_CONTAINERID`, `CNI_NETNS`, and `CNI_IFNAME` as environment variables and the plugin config on stdin. So the failure is one of: **the binary is missing** (check `/opt/cni/bin` for the `type` named in `/etc/cni/net.d/*.conflist`) — often after a node image change; **the config is invalid** (error code 2 names the offending key, code 7 means config validation failed); **IPAM is exhausted** (code 7, and the spec's own example message is a too-small network — see question 4); **the agent is down** (codes 50/51, where 51 additionally warns that existing containers may have degraded connectivity); or **a transient condition** (code 11, "try again later," which the runtime should retry). A useful early split: if the pod has an IP but no connectivity, the ADD *succeeded* and the problem is routing or policy, not CNI; if it has no IP, the ADD failed and the codes above apply.

## Connections & what's next
This lesson is the dataplane substrate under everything downstream in the module. It leans on `modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md` for netfilter, conntrack, and kube-proxy's rule structure, and adds the layers that lesson does not cover: the CNI contract, the overlay/routed choice, address-space arithmetic, socket-level translation, and the control-plane propagation chain. Lesson 03's Maglev material reappears as Cilium's north-south backend selection; lesson 04's cost lens continues into `trafficDistribution` and cross-AZ east-west traffic; lesson 07's RDMA path is defined precisely by what it bypasses from here; lesson 08's decision tree assumes you already know which dataplane-specific command applies.

Next: **[06-service-mesh.md](06-service-mesh.md)** — what gets added, at what latency and CPU cost, once a proxy sits on top of the VIP resolution this lesson just built.

## References & further reading

**Primary sources — read directly for this lesson**

1. `SPEC.md`, `containernetworking/cni` (https://github.com/containernetworking/cni/blob/main/SPEC.md), CNI specification **1.1.0** — the execution protocol in §2: the `CNI_COMMAND` / `CNI_CONTAINERID` / `CNI_NETNS` / `CNI_IFNAME` / `CNI_ARGS` / `CNI_PATH` environment variables; the `ADD` / `DEL` / `CHECK` / `STATUS` / `GC` / `VERSION` operations; the network-configuration format including `plugins`, `ipam`, `ipMasq`, `dns`, and `capabilities`; the Result type (`interfaces`, `ips`, `routes` with `mtu`/`advmss`/`priority`/`table`/`scope`, `dns`) and the `prevResult` chaining rule; delegated IPAM plugins returning an abbreviated Result; and the full error-code table (1–7, 11, 50, 51) with the spec's own IPAM-exhaustion example message.
2. `pkg/controller/endpointslice/config/v1alpha1/defaults.go` and `pkg/controller/endpointslice/config/types.go`, Kubernetes master — `MaxEndpointsPerSlice = 100` and `ConcurrentServiceEndpointSyncs = 5` as the shipped defaults, plus `EndpointUpdatesBatchPeriod` (default 0) as the batching knob used in §8.
3. `content/en/docs/concepts/services-networking/endpoint-slices.md`, `kubernetes/website` — the 100-endpoint default and the **1000** maximum for `--max-endpoints-per-slice`, and the 1000-address-per-slice limit for multi-family slices.
4. `content/en/docs/concepts/services-networking/topology-aware-routing.md`, `kubernetes/website` — every safeguard in §9: allocation proportional to each zone's **allocatable CPU cores**; the **3-or-more endpoints per zone** recommendation and the ≈**50 %** probability of falling back below it; no hints assigned when endpoints are fewer than zones; the refusal to assign hints when expected overload exceeds an acceptable threshold; the explicit warning that the feature is not recommended when traffic originates from a single zone; and the `hints.forZones` EndpointSlice field reproduced in the YAML above.
5. `content/en/docs/concepts/services-networking/service.md`, `kubernetes/website` — `.spec.trafficDistribution` with `PreferSameZone`, `PreferSameNode`, and `PreferClose` (documented as a deprecated, less-clear alias for `PreferSameZone`), and the distinction from `internalTrafficPolicy`/`externalTrafficPolicy` as preferences versus strict semantics. (This documentation corresponds to Kubernetes **1.36**, per `data/releases/schedule.yaml` in the same repository.)
6. `Documentation/network/kubernetes/kubeproxy-free.rst`, `cilium/cilium` main — the socket-LB material in §7: the cgroup-v2 requirement and mount path; the `cilium-dbg status --verbose` "KubeProxyReplacement Details" output reproduced in the worked example; session affinity keyed on the **client's network-namespace cookie** because no source IP exists at endpoint-selection time; Maglev applying to north-south traffic only, with `maglev.tableSize` selectable from a set of primes up to **65521** (≈650 backends) and `maglev.hashSeed` needing to match across agents; the `cilium_lb4_reverse_sk` / `cilium_lb6_reverse_sk` LRU-map exhaustion issue with unconnected-UDP workloads and the `socketLB.hostNamespaceOnly=true` mitigation; and the statement that the EndpointSlice update and kubelet SIGTERM are triggered **simultaneously** and cannot coordinate as of Kubernetes **1.36**.
7. `Documentation/network/concepts/routing.rst`, `cilium/cilium` main — the encapsulation overhead figure used in §4 and §5: **50 bytes per packet for VXLAN**, and the jumbo-frame argument (50 bytes against 1500 versus 50 bytes against 9000) that reframes overlay cost as a proportion rather than a ceiling.
8. `Documentation/networking/ip-sysctl.rst`, Linux master — `tcp_mtu_probing` (0 / 1 / 2 semantics), `tcp_base_mss`, `tcp_mtu_probe_floor = 48`, and `tcp_retries2 = 15` for the black-hole timeout bound quoted in §5.

**Sources named but not fetched in this pass — do not treat the wording as verified**

9. `kubernetes.io` rendered documentation pages (`/docs/reference/networking/virtual-ips/`, `/docs/concepts/services-networking/network-policies/`) are **blocked by this environment's egress policy**. The Kubernetes material above was read from the `kubernetes/website` and `kubernetes/kubernetes` source repositories instead, which is the same content pre-render. The NetworkPolicy semantics in §10 (default-allow until selected, then default-deny for the selected direction; additive allow-only rules; no ICMP in the core API) are stated from the API's own shape and were **not** re-verified against the rendered concept page in this pass — confirm before relying on the ICMP gap in a security design.
10. AWS VPC CNI documentation and the per-instance-type ENI/IP limits referenced in §6 — `docs.aws.amazon.com` is blocked here. The *model* (pod IPs drawn from the VPC subnet, per-node pod ceiling set by ENI capacity, prefix delegation allocating /28 blocks) is described qualitatively and the specific per-instance limits are deliberately **not** quoted. Look them up for your instance types before capacity planning.
11. Mark Betz, "Exhausting conntrack table space crippled our k8s cluster" — `medium.com` is blocked here and the post was not read. The conntrack-exhaustion failure mode it describes is covered in full, with kernel-level detail and current kube-proxy defaults, in `modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md`, which is where this lesson sends you for it.
12. Cloudflare's `pwru` write-up on an IPVS encapsulation bug — `blog.cloudflare.com` is blocked here. `pwru` itself is covered in lesson 08; no figures from that post are asserted in this lesson.
