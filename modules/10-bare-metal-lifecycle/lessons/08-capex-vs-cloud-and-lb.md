---
lesson: "10.8"
title: "Capex vs cloud: the GPU crossover model — and load-balancing without a cloud LB"
module: "10"
concept: "Capex vs cloud: the GPU crossover model — and load-balancing without a cloud LB"
status: not-started
est_time: "8h"
artifacts: []
---

# 10.8 · Capex vs cloud: the GPU crossover model — and load-balancing without a cloud LB

> **Concept.** Build the owned-vs-rented $/GPU-hr crossover model that *is* the neocloud business thesis — capex, power×PUE, colo, InfiniBand, depreciation, staff — find break-even utilisation and payback month; then, as the on-prem corollary, serve LoadBalancer/ingress and the API VIP with MetalLB (BGP) and kube-vip, because there is no cloud LB to call.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Why this matters
This is the capstone and your differentiator. A neocloud (CoreWeave, Lambda, Crusoe, Nebius, and the long tail) is, financially, **one arbitrage**: buy GPUs at capex, rent them at $/GPU-hr, and pocket the spread *if* utilisation stays high enough to beat the depreciation clock. If you can build the model that says *at what utilisation does owning beat renting, and which assumption moves the answer*, you can reason about that business — and about whether your own 40-cluster fleet should have been bought or rented. Almost nobody writes this rigorously; hiring managers at GPU-heavy shops notice a candidate who can.

Your FinOps + on-prem background is the exact edge: you already know colocation contracts, power, and PUE. The model below is where that knowledge becomes a spreadsheet a CFO trusts.

The second half is the unglamorous on-prem reality that pairs with it: once you own the metal, **there is no ELB**. A `Service type: LoadBalancer` sits `<pending>` forever unless *you* provide the load balancer. MetalLB and kube-vip are how bare-metal clusters get external VIPs and a control-plane VIP — the thing the managed clouds did for you and never itemised.

## What's new here
Earlier module lessons built the cluster (provisioning, etcd/HA, storage). This lesson makes it an **economic and network-complete** system. New material:

- **The crossover model** — every cost input for owning a GPU fleet, reduced to an all-in **$/GPU-hr** as a function of **utilisation** and **months owned**, plotted against the rental rate.
- **Depreciation + residual-value risk** — 3-yr straight-line, and the resale-glut haircut that neoclouds under-model.
- **The costs cloud pricing hides** — power×PUE, colo, InfiniBand capex, staff FTE, idle time.
- **MetalLB** in **L2** vs **BGP** mode — how a bare-metal cluster answers `type: LoadBalancer`, and why BGP is mandatory once services span multiple racks/subnets.
- **kube-vip** — a floating VIP for the control-plane API endpoint, replacing the cloud LB in front of your `kube-apiserver` (ties back to the HA lesson).

## Core notes — the capex-vs-cloud model
> **All $/hardware/power/rental figures below are 2026 snapshots and volatile.** Treat them as your default *inputs*, not truths — the deliverable's whole point is that you replace them with your own and watch the crossover move.

### The one equation
```
owned $/GPU-hr  =   (monthly_fixed + monthly_opex)
                  ─────────────────────────────────────
                    GPU_count × 730 hr/mo × utilisation
```
`monthly_fixed` = depreciation (capex spread over the life). `monthly_opex` = power + colo + network + maintenance + staff. Everything hard is in populating those two, and in the fact that **utilisation is in the denominator** — halve it and $/GPU-hr doubles. That single sensitivity is the whole story.

### Inputs for a 64-GPU fleet (8 nodes × 8×H100 SXM)

**1. Hardware capex.**
- An 8×H100 SXM node (HGX board, 2S CPU, ~2 TB RAM, NVMe, NICs, chassis): **~$222K–$383K** per node (2026 snapshot; wide because GPU street price and the rest-of-node swing a lot).
- 8 nodes ⇒ compute capex **~$1.8M–$3.1M**.
- **InfiniBand fabric** (NDR 400G switches + NICs + cables) is *separate and large* — budget **~$8K–$20K/GPU-equivalent** at cluster scale; for 64 GPUs call it **~$300K–$600K**. This line is the one people forget.
- Total capex ≈ **~$2.1M–$3.7M**.

**2. Power (the opex that scales with use).**
- An 8×H100 node draws **~5.5–6.5 kW sustained** under training load (2026 snapshot).
- At **$0.10/kWh**: `6 kW × 730 hr × $0.10 ≈ $438/node/mo` for IT load alone — but apply **PUE**. On-prem/colo PUE ~**1.3–1.5** (cooling + distribution overhead), so multiply by ~1.4 ⇒ **~$600/node/mo** at that rate. *The prompt's "~$4K/mo/node" figure corresponds to a higher blended rate (~$0.20–0.30/kWh incl. demand charges) or a higher draw — power price is regional and this line swings 3–5×.* Model it as `node_kW × 730 × $/kWh × PUE`; flag your $/kWh explicitly.
- 8 nodes ⇒ **~$5K–$32K/mo** depending on your electricity rate. Use your colo's real number.

**3. Colocation / facility.** Rack space, cross-connects, remote hands. A high-density GPU rack (needs 40–60 kW+ and liquid or rear-door cooling) is not a standard colo rack — **~$1.5K–$3K/rack/mo** plus power, or a $/kW model. Budget **~$3K–$8K/mo** for the fleet. (If you own the datacenter, this becomes real-estate + amortised buildout — your on-prem background applies directly.)

**4. Maintenance / support.** Vendor support, spares, DOA/RMA (GPUs *do* die), ~**5–10% of hardware capex/yr** ⇒ **~$9K–$31K/mo**.

**5. Staffing.** Infra engineers to run it. A fully-loaded GPU infra engineer is **~$275K/yr** (2026 snapshot); a 64-GPU fleet needs a fraction of a team — **amortise ~0.5–1.0 FTE** ⇒ **~$11K–$23K/mo**. Neocloud pricing buries this; you can't.

**6. Depreciation + residual risk.** **3-yr straight-line** is the standard assumption ⇒ `monthly_fixed = capex / 36`. For **~$2.7M** capex that's **~$75K/mo**. But GPUs don't hold value like buildings: when the next generation ships and rental rates fall, the **resale/residual value can collapse** (the "glut" haircut). Model residual as a *risk*: 3-yr straight-line to a residual you choose (e.g. 10–20%, or 0% if you assume worthless-at-refresh). Aggressive neocloud models assume 5–6 yr life or high residual to make the spread look better — that's exactly the assumption to stress-test.

### Putting it together (illustrative midpoint, flag as snapshot)
Take capex **$2.7M** ⇒ depreciation **~$75K/mo**. Opex midpoints: power ~$12K + colo ~$5K + maint ~$18K + staff ~$17K ≈ **~$52K/mo**. Total **~$127K/mo**.

```
owned $/GPU-hr = 127,000 / (64 × 730 × U)
```
| Utilisation U | owned $/GPU-hr |
|---|---|
| 100% | **~$2.72** |
| 85% | ~$3.20 |
| 70% | ~$3.88 |
| 50% | ~$5.43 |
| 30% | ~$9.05 |

**Rental rail:** neocloud H100 **~$2.0–2.6/GPU-hr** (2026 snapshot; the Silicon Data H100 index sat around ~$2.5 in mid-2026). So this fleet — at midpoint assumptions — lands **~$4–5/GPU-hr at moderate (50–70%) utilisation** and only **approaches/beats rental as U → ~90–100%**. That is the neocloud thesis in one table: **owning only wins at near-saturation**, which is *why* neoclouds exist — they aggregate demand across many customers to hit the utilisation a single team can't.

### The crossover chart — the deliverable's headline
Plot **owned $/GPU-hr vs utilisation** (x-axis 0–100%) as a curve, with the **rental rate as a horizontal line**. The **break-even utilisation** is where they cross (~90%+ at these midpoints). Add a second view: **cumulative $ owned vs cumulative $ rented over months owned** at a fixed utilisation — the two lines cross at the **payback month** (owning is cash-negative early because of capex, then the depreciation-vs-rent spread pays it back; if utilisation is too low, they never cross before the 3-yr refresh and owning simply loses).

**Most sensitive assumption:** **utilisation**, because it's the denominator — but a close second is the **depreciation life / residual** (3 yr vs 5 yr moves `monthly_fixed` by ~40%), and third is your **$/kWh**. State which you tested; a defensible answer is "utilisation dominates, residual is the sleeper risk."

## Core notes — load-balancing without a cloud LB

### The problem
On EKS/GKE, `Service type: LoadBalancer` triggers a cloud controller that provisions an ELB/GLB. On bare metal there is no such controller — the Service's `EXTERNAL-IP` stays **`<pending>`** forever. You must supply the LB. Two jobs, two tools:
- **Service/ingress external IPs** for workloads → **MetalLB**.
- **A single stable VIP for the control-plane API** (so kubectl/kubelets hit one address across N apiservers) → **kube-vip** (or keepalived+HAProxy). This is the piece the HA lesson's stacked-etcd control plane needs in front of it.

### MetalLB: L2 vs BGP
MetalLB hands out IPs from pools you define and *advertises* them so the network routes them to the cluster. Two modes:

- **L2 (ARP/NDP) mode.** One elected node answers ARP for the service VIP; all traffic for that IP enters through that one node, then kube-proxy spreads it. **Simple, no router config.** Limits: (1) it's failover, **not load-balancing** — a single node is the ingress for a given VIP at a time, so bandwidth is capped at one node's NIC; (2) the VIP must live in the **same L2 subnet** as the nodes — it **cannot cross a router**. Fine for a single rack / flat network.
- **BGP mode.** Each node runs a BGP speaker (MetalLB bundles **FRR**) and *peers with your top-of-rack routers*, advertising the service IP. The router then **ECMP load-balances** across all nodes, and the VIP can live in a routed subnet reachable from anywhere in the fabric. **True multi-node load-balancing, routable across racks.**

**Why BGP over L2 across multiple racks (self-check b):** L2 mode can't cross a subnet — nodes in different racks are in different L2 domains, so a single L2 VIP can't span them, and even within one rack L2 funnels a VIP through one node. **BGP advertises the service prefix into the routed fabric**, so ToR routers ECMP the VIP across every node in every rack, giving real horizontal ingress bandwidth and cross-rack reach. It's the same reason your MetalLB-adjacent networking work in module 09 leaned on BGP/fabric: routed, not bridged, is how you scale past one rack. (Caveat: node failure mid-connection can reset flows because ECMP rehashes — acceptable for most, mitigated by graceful BGP/`GracefulRestart`.)

### kube-vip for the API VIP
The apiservers need **one address**. kube-vip runs as a static pod on the control-plane nodes and floats a **virtual IP** to whichever node is leader (leader-election via the k8s API/lease). Modes: **ARP** (L2, VIP fails over to the leader) or **BGP** (advertise the VIP, like MetalLB). It can also do control-plane LB (spread apiserver traffic) and optionally service LB. In the KTHW/HA build this replaces the cloud LB that fronted your apiserver — kubelets and kubectl target the kube-vip VIP, and it survives a control-plane node loss.

## Worked example — MetalLB in BGP mode advertising a service VIP
Cluster from earlier lessons; ToR router `192.168.10.1`, ASN 64512; MetalLB in the cluster on ASN 64513; service pool `203.0.113.0/24`.

```yaml
# BGPPeer — peer every node's speaker with the ToR router
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata: { name: tor, namespace: metallb-system }
spec:
  myASN: 64513
  peerASN: 64512
  peerAddress: 192.168.10.1
---
# The pool MetalLB allocates service VIPs from
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: { name: svc-pool, namespace: metallb-system }
spec:
  addresses: ["203.0.113.0/24"]
---
# Advertise the pool via BGP (not L2)
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata: { name: svc-bgp, namespace: metallb-system }
spec:
  ipAddressPools: ["svc-pool"]
```
```yaml
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  type: LoadBalancer
  selector: { app: web }
  ports: [{ port: 80, targetPort: 8080 }]
```
Apply, then `kubectl get svc web` shows an `EXTERNAL-IP` from `203.0.113.0/24` (no longer `<pending>`). Verify the router learned it and ECMPs it:
```
# on the ToR router
show ip bgp 203.0.113.x        # prefix present, next-hops = all node IPs (ECMP)
# from off-cluster
curl http://203.0.113.x/       # routed across racks to the pods
```
The tell that BGP (not L2) is working: **multiple next-hops** for the VIP on the router — traffic spreads across nodes rather than funnelling through one ARP owner.

## Practice — feeds the deliverable (both halves)
**(a) Build the capex-vs-cloud model.** Spreadsheet or notebook (Python), 64-GPU fleet, **your own** assumptions. It must:
- Take inputs: capex (compute + InfiniBand), power (as `node_kW × 730 × $/kWh × PUE` — **flag your $/kWh and PUE**), colo, maintenance %, staff FTE, depreciation years + residual %.
- Compute `owned $/GPU-hr(U)` and the cumulative owned-vs-rented curves over months.
- Output the **crossover chart** (owned $/GPU-hr vs utilisation, rental line overlaid) and the **payback-month chart**, plus the printed **break-even utilisation**.
- Include a **sensitivity note**: which single assumption moves break-even most (run it at 3-yr vs 5-yr depreciation and at two $/kWh to show it).

**(b) Deploy MetalLB in BGP mode.** On the cluster from earlier lessons, install MetalLB, configure a `BGPPeer` (FRR) to a router (a FRR container as the peer is fine in lab), an `IPAddressPool`, and a `BGPAdvertisement`. Expose a `type: LoadBalancer` service and show it gets a VIP and the router learns the prefix with multiple next-hops.

**Acceptance:** committed to the deliverable folder — (1) the **capex-vs-cloud model** (spreadsheet/notebook) producing the crossover chart + break-even utilisation + payback month, with every $/power figure flagged as a snapshot and $/kWh + PUE stated; **and** (2) a **MetalLB BGP service VIP** — the manifests plus evidence (`kubectl get svc` EXTERNAL-IP and the router showing the ECMP prefix). Together with 10.1's KTHW/etcd writeup this is the module deliverable.

## Self-check
**(a) At what sustained utilisation does a 64-GPU owned fleet's $/GPU-hr beat neocloud rental, and what single assumption is the model most sensitive to?**
**Answer:** At midpoint 2026 inputs (~$127K/mo all-in, capex ~$2.7M on 3-yr straight-line), owned $/GPU-hr is ~$2.7 at 100% and ~$3.9 at 70%, versus rental ~$2.0–2.6 — so owning only beats rental at **~90–100% sustained utilisation**; below ~70% it's markedly worse. The model is most sensitive to **utilisation** (it's the denominator — halving it doubles $/GPU-hr), with **depreciation life/residual** the close-second sleeper (3-yr vs 5-yr swings the fixed cost ~40%) and **$/kWh** third. That near-saturation break-even is the neocloud thesis: they hit the utilisation a single team can't by pooling demand.

**(b) Why BGP over L2 for MetalLB across multiple racks?**
**Answer:** L2 mode advertises the VIP by ARP within a single subnet and routes all of a VIP's traffic through one elected node — it **can't cross a router** and it's failover, not load-balancing. Nodes in different racks are in different L2 domains, so one L2 VIP can't span them. **BGP** peers each node with the ToR routers and advertises the service prefix into the **routed fabric**, so routers **ECMP** the VIP across every node in every rack — true horizontal ingress bandwidth and cross-rack reach.

**(c) What non-obvious on-prem costs does cloud pricing hide (name three)?**
**Answer:** Any three of: **power × PUE** (cooling/distribution overhead — the rented $/hr already bundles it, you pay it as a metered bill times ~1.3–1.5); **InfiniBand fabric capex** (400G switches/NICs/cables, a $300K–$600K line for 64 GPUs that vanishes into the cloud rate); **staff FTE** (fully-loaded GPU infra engineers ~$275K/yr, amortised into every owned GPU-hr); **idle/low-utilisation time** (you pay depreciation + power-floor whether or not the GPU is busy, whereas rental is pay-per-use); and **residual-value/depreciation risk** (a next-gen glut can crater resale value mid-life).

## Resources
1. **GPUaaS — The Real TCO of a GPU Cluster (2026)** — https://gpuaas.com/blog/real-tco-gpu-cluster-2026 — **Deep.** The input set for the model: hardware, power, staff, the hidden lines. Flag its figures as 2026 snapshots and swap in yours.
2. **Thunder Compute — Neoclouds: the new GPU clouds** — https://www.thundercompute.com/blog/neoclouds-the-new-gpu-clouds-changing-ai-infrastructure — **Skim to deep.** The business-model framing (the arbitrage your crossover chart formalises) and the rental-rate rail.
3. **MetalLB — BGP concepts** (https://metallb.universe.tf/concepts/bgp/) and **kube-vip** (https://kube-vip.io/) — **Deep for the practice.** The authoritative L2-vs-BGP and control-plane-VIP references behind self-check (b) and the worked example.
