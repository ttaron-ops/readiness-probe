---
lesson: "10.8"
title: "Capex vs cloud: the GPU crossover model — and load-balancing without a cloud LB"
module: "10"
concept: "Capex vs cloud: the GPU crossover model — and load-balancing without a cloud LB"
status: not-started
est_time: "10h"
prev: "07-storage-for-ai.md"
next: null
artifacts: []
sources: 11
---

# 10.8 · Capex vs cloud: the GPU crossover model — and load-balancing without a cloud LB

> **Concept.** Build the owned-vs-rented $/GPU-hr crossover model that *is* the neocloud business thesis — capex, power×PUE, colo, InfiniBand, depreciation, staff — find break-even utilisation and payback month; then, as the on-prem corollary, serve LoadBalancer/ingress and the API VIP with MetalLB (BGP) and kube-vip, because there is no cloud LB to call.
>
> Module: [🖥️ 10 — Bare metal and cluster lifecycle](../README.md) · Deliverable: [Capex-vs-cloud + KTHW/etcd writeup](../practice/capex-vs-cloud/README.md)

## Where this fits

Every lesson before this one built and protected a piece of the fleet: a hand-built control plane (01), an etcd that survives a bad day (02), HA that survives a node loss (03), declarative fleets that provision without hand-holding (04), a PXE pipeline that turns bare metal into a `Ready` node (05), a health loop that catches broken hardware in minutes instead of a month (06), and a storage tier sized so healthy GPUs don't idle waiting on bytes (07). This lesson does two things with all of that. First, it turns every one of those operational decisions into a single number — **owned $/GPU-hr** — and asks the question a CFO actually asks: does owning this fleet beat renting it? Second, it closes the last purely infrastructural gap: on a managed cloud, a `Service type: LoadBalancer` is an API call; on bare metal, *you* are the cloud LB controller, and this lesson is where you build that too. This is the module's **capstone** — the lesson the deliverable and checkpoint are built around.

## Why this matters

This is the capstone and your differentiator. A neocloud (CoreWeave, Lambda, Crusoe, Nebius, and the long tail) is, financially, **one arbitrage**: buy GPUs at capex, rent them at $/GPU-hr, and pocket the spread *if* utilisation stays high enough to beat the depreciation clock. If you can build the model that says *at what utilisation does owning beat renting, and which assumption moves the answer*, you can reason about that business — and about whether your own 40-cluster fleet should have been bought or rented. Almost nobody writes this rigorously; hiring managers at GPU-heavy shops notice a candidate who can.

Your FinOps + on-prem background is the exact edge: you already know colocation contracts, power, and PUE. The model below is where that knowledge becomes a spreadsheet a CFO trusts.

The second half is the unglamorous on-prem reality that pairs with it: once you own the metal, **there is no ELB**. A `Service type: LoadBalancer` sits `<pending>` forever unless *you* provide the load balancer. MetalLB and kube-vip are how bare-metal clusters get external VIPs and a control-plane VIP — the thing the managed clouds did for you and never itemised.

## What's new here (calibration)

Earlier module lessons built the cluster (provisioning, etcd/HA, storage). This lesson makes it an **economic and network-complete** system. No FinOps basics are re-taught — you're certified on those. New material:

- **The crossover model** — every cost input for owning a GPU fleet, reduced to an all-in **$/GPU-hr** as a function of **utilisation** and **months owned**, plotted against the rental rate.
- **Depreciation + residual-value risk** — 3-yr straight-line, and the resale-glut haircut that neoclouds under-model.
- **The costs cloud pricing hides** — power×PUE, colo, InfiniBand capex, staff FTE, idle time — and a sharper framing of *why* rental is priced where it is: scarcity rent, not just cost recovery.
- **MetalLB** in **L2** vs **BGP** mode — how a bare-metal cluster answers `type: LoadBalancer`, and why BGP is mandatory once services span multiple racks/subnets.
- **kube-vip** — a floating VIP for the control-plane API endpoint, replacing the cloud LB in front of your `kube-apiserver` (ties back to the HA lesson).

## Core concepts — the capex-vs-cloud model
> **All $/hardware/power/rental figures below are 2025/2026 snapshots and volatile — several change meaningfully within the same year.** Treat them as your default *inputs*, not truths — the deliverable's whole point is that you replace them with your own and watch the crossover move.

### The one equation
```
owned $/GPU-hr  =   (monthly_fixed + monthly_opex)
                  ─────────────────────────────────────
                    GPU_count × 730 hr/mo × utilisation
```
`monthly_fixed` = depreciation (capex spread over the life). `monthly_opex` = power + colo + network + maintenance + staff. Everything hard is in populating those two, and in the fact that **utilisation is in the denominator** — halve it and $/GPU-hr doubles. That single sensitivity is the whole story.

### Inputs for a 64-GPU fleet (8 nodes × 8×H100 SXM)

**1. Hardware capex.**
- An 8×H100 SXM node (HGX board, 2S CPU, ~2 TB RAM, NVMe, NICs, chassis): **~$222K–$383K** per node was the 2025 range commonly quoted. A newer 2026 data point from SemiAnalysis's cluster-cost analysis puts an 8-GPU **B200/B300-generation** server at **~$250K–$400K** as of early 2026 ([SemiAnalysis, "How Much Do GPU Clusters Really Cost?"](https://newsletter.semianalysis.com/p/how-much-do-gpu-clusters-really-cost)) — the same source models a **576-GPU cluster (72 nodes)**, including InfiniBand and supporting infrastructure, at roughly **$36M**. If you're pricing current-generation silicon, use the higher end of the per-node range; if you're pricing H100-class hardware specifically, the older $222K–$383K band still applies. **State which generation your number is for** — the two are not interchangeable, and this is exactly the kind of scope confusion that makes a capex model wrong without anyone noticing.
- 8 nodes ⇒ compute capex **~$1.8M–$3.2M** (H100-class) or higher on current-gen silicon.
- **InfiniBand fabric** (NDR 400G switches + NICs + cables) is *separate and large* and modeled two different ways depending on source and scope. One rule-of-thumb range: **~$8K–$20K/GPU-equivalent** at cluster scale — for 64 GPUs, **~$300K–$600K**+ hardware alone. A more granular bottoms-up model of a 512-GPU (64-node) fabric build-out (ToR + spine switches, optics, cabling) totals **~$2.5M**, i.e. roughly **~$4,900/GPU** — noticeably lower than the rule-of-thumb range above. The likely reconciliation: the higher rule-of-thumb figure bundles amortized refresh cycles, full redundancy, or a higher-generation optics mix that the bottoms-up BOM doesn't. **Cite whichever figure you use with its scope stated** (hardware-only BOM vs. amortized/lifecycle figure) — this is the single line item people forget to budget at all, so getting the number roughly right matters more than getting it precisely right.
- Total capex for a 64-GPU H100-class fleet ≈ **~$2.1M–$3.8M**.

**2. Power (the opex that scales with use).**
- Vendor-quoted sustained draw for an 8×H100 node is often cited around **~5.5–6.5 kW**, but this appears to be an IT-load-only lower bound. An empirical measurement study — [arXiv:2506.14551, "Empirically-Calibrated H100 Node Power Models for Reducing Uncertainty in AI Training Energy Estimation"](https://arxiv.org/abs/2506.14551) (also published in IOP/IEEE venues) — measured an 8×H100 HGX node's **peak draw at ~8.4 kW** under real training load (18% below the vendor-rated 10.2 kW TDP), with a fixed-effects model predicting **~7.3 kW as a typical** sustained figure. **Widen your planning range to ~6–8.5 kW/node** and cite the paper if you use the higher figure; note that vendor "~5.5–6.5 kW" numbers may only be measuring IT load under lighter-than-peak conditions.
- At **$0.10/kWh** and 7 kW: `7 kW × 730 hr × $0.10 ≈ $511/node/mo` for IT load alone — but apply **PUE**. On-prem/colo PUE ~**1.3–1.5** (cooling + distribution overhead), so multiply by ~1.4 ⇒ **~$715/node/mo** at that rate. Power price is regional and this line swings 3–5× on $/kWh alone: a **$0.05/kWh swing** changes annual margin by roughly **$15K–$30K per node per year** — a large enough sensitivity that "flag your $/kWh explicitly" is not a formality, it's load-bearing. Model it as `node_kW × 730 × $/kWh × PUE`.
- 8 nodes ⇒ **~$6K–$40K/mo** depending on your electricity rate and which power-draw figure you use. Use your colo's real number. As a portfolio-level sanity check, **power typically runs 25–40% of total GPU-cluster opex** — if your model's power line is a much smaller share than that, double-check your kW or $/kWh assumption.

**3. Colocation / facility.** Rack space, cross-connects, remote hands. A high-density GPU rack (needs 40–60 kW+ and liquid or rear-door cooling) is not a standard colo rack — **~$1.5K–$3K/rack/mo** plus power, or a $/kW model. Budget **~$3K–$8K/mo** for the fleet. (If you own the datacenter, this becomes real-estate + amortised buildout — your on-prem background applies directly.)

**4. Maintenance / support.** Vendor support, spares, DOA/RMA (GPUs *do* die — this is the same failure-and-replace loop lesson 06 built the detection side of), ~**5–10% of hardware capex/yr** ⇒ **~$9K–$32K/mo**.

**5. Staffing.** Infra engineers to run it. A fully-loaded GPU infra engineer is **~$275K/yr** (2026 snapshot); a 64-GPU fleet needs a fraction of a team — **amortise ~0.5–1.0 FTE** ⇒ **~$11K–$23K/mo**. Neocloud pricing buries this; you can't. Note that lesson 04's declarative-fleet automation (CAPI/Talos) is precisely what keeps this fraction low at scale — a fleet run by hand doesn't stay at 0.5–1.0 FTE past a few hundred nodes.

**6. Depreciation + residual risk.** **3-yr straight-line** is the standard assumption ⇒ `monthly_fixed = capex / 36`. For **~$2.7M** capex that's **~$75K/mo**. But GPUs don't hold value like buildings: when the next generation ships and rental rates fall, the **resale/residual value can collapse** (the "glut" haircut). Model residual as a *risk*: 3-yr straight-line to a residual you choose (e.g. 10–20%, or 0% if you assume worthless-at-refresh). Aggressive neocloud models assume 5–6 yr life or high residual to make the spread look better — that's exactly the assumption to stress-test.

### Putting it together (illustrative midpoint, flag as snapshot)
Take capex **$2.7M** ⇒ depreciation **~$75K/mo**. Opex midpoints: power ~$14K + colo ~$5K + maint ~$18K + staff ~$17K ≈ **~$54K/mo**. Total **~$129K/mo**.

```
owned $/GPU-hr = 129,000 / (64 × 730 × U)
```
| Utilisation U | owned $/GPU-hr |
|---|---|
| 100% | **~$2.76** |
| 85% | ~$3.25 |
| 70% | ~$3.95 |
| 50% | ~$5.52 |
| 30% | ~$9.20 |

**Rental rail:** neocloud H100 pricing spans a wide band and moves within a single year. A survey of 15+ providers put the 2026 range at **$1.49–$6.98/hr** ([IntuitionLabs, "H100 Rental Prices Compared"](https://intuitionlabs.ai/articles/h100-rental-prices-cloud-comparison)) — and the same source flags that a **1-year contract rate rose ~40%, from $1.70/hr (Oct 2025) to $2.35/hr (Mar 2026)**. That is concrete evidence for the "flag every $ as a snapshot" rule in this lesson's opening warning: rental prices are demonstrably **not monotonically falling**, and a model built on an October figure was already 40% stale by March.

There is also a sharper way to frame the owned-vs-rental gap than pure crossover arithmetic. [CloudZero's 2026 H100 cost analysis](https://www.cloudzero.com/blog/h100-gpu-cost/) computes that at 70% utilisation, **4-year amortized capex runs ~$1.12/hr** and **power + facility ~$0.33/hr**, for an all-in owned cost of **~$1.45/hr** — against Lambda's on-demand rental of **$3.99/hr**. The **$2.54/hr gap (64% of the rental price)** is framed explicitly as **"demand premium: scarcity rent, not cost recovery."** That framing matters: it says the majority of what you pay for on-demand rental isn't the provider's cost of goods, it's paying for *not* having to commit capital or wait for capacity — the same logic that makes reserved/committed-use rental meaningfully cheaper than on-demand.

Reading this lesson's own midpoint table against that: this fleet — at midpoint assumptions — lands **~$4–5/GPU-hr at moderate (50–70%) utilisation** and only **approaches/beats rental as U → ~90–100%**. That is the neocloud thesis in one table: **owning only wins at near-saturation**, which is *why* neoclouds exist — they aggregate demand across many customers to hit the utilisation a single team can't, and they charge a scarcity premium on top for everyone who can't or won't commit to owning.

### The crossover chart — the deliverable's headline
Plot **owned $/GPU-hr vs utilisation** (x-axis 0–100%) as a curve, with the **rental rate as a horizontal line**. The **break-even utilisation** is where they cross (~90%+ at these midpoints). Add a second view: **cumulative $ owned vs cumulative $ rented over months owned** at a fixed utilisation — the two lines cross at the **payback month** (owning is cash-negative early because of capex, then the depreciation-vs-rent spread pays it back; if utilisation is too low, they never cross before the 3-yr refresh and owning simply loses).

**Most sensitive assumption:** **utilisation**, because it's the denominator — but a close second is the **depreciation life / residual** (3 yr vs 5 yr moves `monthly_fixed` by ~40%), and third is your **$/kWh** (a $0.05/kWh swing alone moves annual margin ~$15K–$30K/node). State which you tested; a defensible answer is "utilisation dominates, residual is the sleeper risk, power is the line most likely to be wrong by 3–5× if you didn't get your colo's real rate."

## Core concepts — load-balancing without a cloud LB

### The problem
On EKS/GKE, `Service type: LoadBalancer` triggers a cloud controller that provisions an ELB/GLB. On bare metal there is no such controller — the Service's `EXTERNAL-IP` stays **`<pending>`** forever. You must supply the LB. Two jobs, two tools:
- **Service/ingress external IPs** for workloads → **MetalLB**.
- **A single stable VIP for the control-plane API** (so kubectl/kubelets hit one address across N apiservers) → **kube-vip** (or keepalived+HAProxy). This is the piece the HA lesson's stacked-etcd control plane needs in front of it.

### MetalLB: L2 vs BGP
MetalLB hands out IPs from pools you define and *advertises* them so the network routes them to the cluster. Two modes:

- **L2 (ARP/NDP) mode.** One elected node answers ARP for the service VIP; all traffic for that IP enters through that one node, then kube-proxy spreads it. **Simple, no router config.** Limits: (1) it's failover, **not load-balancing** — a single node is the ingress for a given VIP at a time, so bandwidth is capped at one node's NIC; (2) the VIP must live in the **same L2 subnet** as the nodes — it **cannot cross a router**. Fine for a single rack / flat network.
- **BGP mode.** Each node runs a BGP speaker (MetalLB bundles **FRR**) and *peers with your top-of-rack routers*, advertising the service IP. The router then **ECMP load-balances** across all nodes, and the VIP can live in a routed subnet reachable from anywhere in the fabric. **True multi-node load-balancing, routable across racks.**

**Why BGP over L2 across multiple racks (self-check b):** L2 mode can't cross a subnet — nodes in different racks are in different L2 domains, so a single L2 VIP can't span them, and even within one rack L2 funnels a VIP through one node. **BGP advertises the service prefix into the routed fabric**, so ToR routers ECMP the VIP across every node in every rack, giving real horizontal ingress bandwidth and cross-rack reach. It's the same reason your MetalLB-adjacent networking work in module 09 leaned on BGP/fabric: routed, not bridged, is how you scale past one rack. (Caveat: node failure mid-connection can reset flows because ECMP rehashes — acceptable for most, mitigated by graceful BGP/`GracefulRestart`.)

A production reference architecture that combines BGP-mode MetalLB with HA bare-metal control planes is worth reading alongside this lesson's worked example: [Red Hat — "Deploying a high-availability, fault-tolerant Kubernetes Service on bare metal clusters with MetalLB BGP"](https://www.redhat.com/en/blog/deploying-a-high-availability-fault-tolerant-kubernetes-service-on-baremetal-clusters-with-metallb-bgp) — it walks the same BGP-peering pattern this lesson's worked example builds, in a vendor-reference-architecture form.

### kube-vip for the API VIP
The apiservers need **one address**. kube-vip runs as a static pod on the control-plane nodes and floats a **virtual IP** to whichever node is leader (leader-election via the k8s API/lease). Modes: **ARP** (L2, VIP fails over to the leader) or **BGP** (advertise the VIP, like MetalLB). It can also do control-plane LB (spread apiserver traffic) and optionally service LB. In the KTHW/HA build this replaces the cloud LB that fronted your apiserver — kubelets and kubectl target the kube-vip VIP, and it survives a control-plane node loss.

## Perspectives

**Financial / CFO view.** The equation in this lesson is the entire pitch a neocloud makes to its investors, inverted: "we can run this fleet at higher utilisation than any single customer, so we can buy at capex prices and sell at a margin below what a customer's own owned-fleet economics would achieve below ~90% utilisation." Your job as the engineer building this model for your own org is to give finance a number they can defend in a budget review, with every dollar figure dated and every assumption named — "utilisation dominates" is a one-sentence answer a CFO can act on.

**Platform-operator view.** The model is only as good as the operational reality feeding it. A fleet with a slow RMA loop (lesson 06) or a storage tier that stalls GPUs (lesson 07) doesn't just have a reliability problem — it has a **utilisation problem**, and utilisation is the single most sensitive input in the whole equation. Every lesson in this module that improved automated remediation, provisioning speed, or storage throughput was, in this lesson's terms, directly raising the denominator.

**Network-engineer view.** MetalLB and kube-vip are not optional add-ons — without them, a bare-metal cluster cannot serve external traffic at all, and the control plane cannot be addressed by a single stable endpoint. The BGP-vs-L2 decision is the same routed-vs-bridged tradeoff module 09 taught for compute/storage traffic, applied to the control path: L2 is simple and rack-local, BGP is what scales past one rack and gives you real load distribution instead of failover-only.

**Failure-mode view.** Both halves of this lesson have a distinctive way to be quietly wrong. The economic model is wrong when someone reuses a $/GPU-hr or $/kWh figure without a date — the IntuitionLabs data point (rental rates rising 40% in five months) proves that a "current" number goes stale fast. The networking half is wrong when someone deploys MetalLB in L2 mode across multiple racks and is surprised traffic doesn't load-balance — L2 was never designed to do that; it's failover funneled through one node, and the failure is silent (`EXTERNAL-IP` looks fine) until you look at where traffic is actually landing.

## Real-world use cases

- **MetalLB in production at 1&1 Mail & Media and Nokia.** MetalLB's own [ADOPTERS.md](https://github.com/metallb/metallb/blob/main/ADOPTERS.md) (directly verified against the repository) names **1&1 Mail & Media** (operators of WEB.DE, GMX, and mail.com) as using MetalLB since 2018 as **"the main way to get network traffic into the clusters driving most of the internal services,"** and **Nokia EDA** as using it since 2024 to implement the LoadBalancer controller role in Nokia's Event Driven Automation product. What it shows: this is not a lab toy — it's the primary ingress path for internal services at a large consumer email provider, and a component inside a telecom vendor's own automation product.
- **Red Hat — HA bare-metal reference architecture with MetalLB BGP.** [Red Hat's blog](https://www.redhat.com/en/blog/deploying-a-high-availability-fault-tolerant-kubernetes-service-on-baremetal-clusters-with-metallb-bgp) walks a full reference deployment combining BGP-mode MetalLB with a fault-tolerant bare-metal control plane. What it shows: a named vendor's worked pattern for exactly the combination this lesson's worked example builds — useful as a second, independent implementation to check your own manifests against.
- **SemiAnalysis and CloudZero as the economic-analysis "production stories" for the capex side.** This half of the lesson is economics, not engineering-blog narrative, so the equivalent of a production case study is rigorous third-party cost analysis: [SemiAnalysis's GPU cluster cost breakdown](https://newsletter.semianalysis.com/p/how-much-do-gpu-clusters-really-cost) (real 2025/2026 pricing across cluster sizes) and [CloudZero's H100 cost analysis](https://www.cloudzero.com/blog/h100-gpu-cost/) (the "scarcity rent, not cost recovery" framing). What they show: independent, dated cost breakdowns you can cross-check your own model's midpoint numbers against, rather than trusting a single vendor's pricing page.
- **Cross-reference — fast RMA turnaround as a utilisation input.** CoreWeave's own node-lifecycle writeup (cited fully in lessons 05/06: [coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference](https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference)) is worth one more mention here: fast, automated RMA turnaround is part of the utilisation story this lesson's crossover model depends on. Slow RMA → lower effective utilisation → the model's denominator gets punished directly, exactly the mechanism the Perspectives section above names.

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
- Take inputs: capex (compute + InfiniBand, stating which GPU generation you priced), power (as `node_kW × 730 × $/kWh × PUE` — **flag your $/kWh, PUE, and which power-draw figure (vendor vs. empirical) you used**), colo, maintenance %, staff FTE, depreciation years + residual %.
- Compute `owned $/GPU-hr(U)` and the cumulative owned-vs-rented curves over months.
- Output the **crossover chart** (owned $/GPU-hr vs utilisation, rental line overlaid) and the **payback-month chart**, plus the printed **break-even utilisation**.
- Include a **sensitivity note**: which single assumption moves break-even most (run it at 3-yr vs 5-yr depreciation and at two $/kWh to show it).
- Cross-check your midpoint owned $/GPU-hr and rental rail against at least one dated external figure (e.g. the CloudZero or SemiAnalysis numbers above) and note the delta — this is what makes the model defensible, not just internally consistent.

**(b) Deploy MetalLB in BGP mode.** On the cluster from earlier lessons, install MetalLB, configure a `BGPPeer` (FRR) to a router (a FRR container as the peer is fine in lab), an `IPAddressPool`, and a `BGPAdvertisement`. Expose a `type: LoadBalancer` service and show it gets a VIP and the router learns the prefix with multiple next-hops.

**Acceptance:** committed to the deliverable folder — (1) the **capex-vs-cloud model** (spreadsheet/notebook) producing the crossover chart + break-even utilisation + payback month, with every $/power figure flagged as a snapshot and $/kWh + PUE stated; **and** (2) a **MetalLB BGP service VIP** — the manifests plus evidence (`kubectl get svc` EXTERNAL-IP and the router showing the ECMP prefix). Together with 10.1's KTHW/etcd writeup, this is the full module deliverable: see [`practice/capex-vs-cloud/README.md`](../practice/capex-vs-cloud/README.md) for the exact layout and acceptance criteria.

## Common pitfalls

- **Treating "utilisation" as a fixed input instead of the model's most sensitive variable.** It's the denominator of the whole equation — a fleet that looks like a great deal at a planned 85% utilisation looks terrible at the 50% it actually achieves in month one. Always show the crossover curve, not a single point estimate.
- **Quoting a $/GPU-hr or $/kWh figure without a date.** The IntuitionLabs data point — 1-yr contract rental rising ~40% from $1.70/hr (Oct 2025) to $2.35/hr (Mar 2026) — is direct proof that rental prices move meaningfully within a single fiscal year, in either direction. An undated number in a model is a silent time bomb.
- **Assuming aggressive depreciation life or residual value to make owning look better.** 3-yr straight-line to a low residual is the defensible default; stretching to 5–6 years or assuming high resale value is exactly the assumption a skeptical reviewer will (correctly) challenge first.
- **Deploying MetalLB in L2 mode and expecting load-balancing across racks.** L2 is failover through one elected node within a single subnet — it does not spread traffic across nodes and cannot cross a router. If services span multiple racks/subnets, BGP is not optional.
- **Forgetting the InfiniBand fabric line, or double-counting it against a source that already amortizes it.** This is the single most commonly-omitted capex line, and the two commonly-cited figures for it (~$4,900/GPU bottoms-up vs. ~$8K–$20K/GPU-equivalent rule-of-thumb) have different scope — state which one you're using and why.
- **Building the economic model in isolation from the operational lessons that feed it.** A slow RMA loop (06) or a storage-induced stall (07) shows up in this model only as a lower utilisation number — if you don't explicitly connect them, the model looks fine on paper while the fleet underperforms it in production.

## Self-check
**(a) At what sustained utilisation does a 64-GPU owned fleet's $/GPU-hr beat neocloud rental, and what single assumption is the model most sensitive to?**
**Answer:** At midpoint 2025/2026 inputs (~$129K/mo all-in, capex ~$2.7M on 3-yr straight-line), owned $/GPU-hr is ~$2.8 at 100% and ~$4.0 at 70%, versus a rental band of roughly $1.5–4/GPU-hr depending on term and provider — so owning only reliably beats rental at **~85–100% sustained utilisation**; below ~70% it's markedly worse. The model is most sensitive to **utilisation** (it's the denominator — halving it doubles $/GPU-hr), with **depreciation life/residual** the close-second sleeper (3-yr vs 5-yr swings the fixed cost ~40%) and **$/kWh** third (a $0.05/kWh swing moves annual margin ~$15K–$30K/node). That near-saturation break-even is the neocloud thesis: they hit the utilisation a single team can't by pooling demand.

**(b) Why BGP over L2 for MetalLB across multiple racks?**
**Answer:** L2 mode advertises the VIP by ARP within a single subnet and routes all of a VIP's traffic through one elected node — it **can't cross a router** and it's failover, not load-balancing. Nodes in different racks are in different L2 domains, so one L2 VIP can't span them. **BGP** peers each node with the ToR routers and advertises the service prefix into the **routed fabric**, so routers **ECMP** the VIP across every node in every rack — true horizontal ingress bandwidth and cross-rack reach.

**(c) What non-obvious on-prem costs does cloud pricing hide (name three)?**
**Answer:** Any three of: **power × PUE** (cooling/distribution overhead — the rented $/hr already bundles it, you pay it as a metered bill times ~1.3–1.5); **InfiniBand fabric capex** (400G switches/NICs/cables, a several-hundred-thousand-dollar line for 64 GPUs that vanishes into the cloud rate); **staff FTE** (fully-loaded GPU infra engineers ~$275K/yr, amortised into every owned GPU-hr); **idle/low-utilisation time** (you pay depreciation + power-floor whether or not the GPU is busy, whereas rental is pay-per-use); and **residual-value/depreciation risk** (a next-gen glut can crater resale value mid-life).

**(d) CloudZero frames the gap between owned cost (~$1.45/hr all-in) and on-demand rental (~$3.99/hr) as "scarcity rent, not cost recovery." What does that framing mean, and why does it matter for how you present this model?**
**Answer:** It means the majority of the on-demand rental price (in CloudZero's breakdown, ~64% of it) isn't the provider recovering their hardware/power/facility cost — it's what a renter pays for *flexibility*: no capital commitment, no utilisation risk, capacity on demand. It matters for presentation because it reframes the crossover chart's message: "owning wins near saturation" isn't really about the provider's costs being lower, it's about the renter no longer needing to pay for the optionality once their own utilisation is high enough to self-insure that risk. That's a more persuasive way to explain the break-even to a CFO than raw arithmetic alone.

**(e) How does lesson 06's hardware-health/RMA loop concretely change the number in this lesson's equation?**
**Answer:** It changes **utilisation**, the denominator. CoreWeave's "up to a month" manual-detection worst case means a sick node can sit `schedulable`-but-broken, contributing zero useful compute while still accruing its full depreciation and power cost — that node's effective utilisation over that period is near 0%, dragging the fleet average down. A minutes-to-detect automated loop (lesson 06) keeps a bad node's downtime to hours instead of weeks, which is the direct mechanism by which better operations turns into a lower owned-$/GPU-hr — the two lessons describe the same fleet from different angles.

## Connections & what's next

This is the module's last lesson, and its job is to close the loop the whole module opened. Lesson 01 handed you a hand-built control plane with **no cloud anything** in front of it — and this lesson supplies the missing piece, the load balancer, that a managed cluster would have given you for free. Lesson 02 made etcd's uptime your problem; this lesson's kube-vip VIP is the thing that makes that control plane reachable at all once you have more than one apiserver. Lesson 03's stacked-etcd HA design and version-skew discipline is exactly what kube-vip fronts. Lesson 04's declarative fleets (CAPI/Talos) are what keeps the **staff FTE** line in this lesson's opex model small at scale instead of growing linearly with node count. Lesson 05's PXE pipeline is what turns capex sitting in a rack into GPU-hours you can actually sell or use — every day a node isn't provisioned is a day of depreciation with zero utilisation. Lesson 06's closed health/RMA loop is, as self-check (e) makes explicit, a direct input to this lesson's utilisation term. Lesson 07's storage sizing is the same story from a different subsystem — Meta's 56% stalled-GPU number and CoreWeave's month-long undetected-fault number are both, in this lesson's vocabulary, utilisation destroyed silently.

In other words: **this lesson is not a bolt-on economics unit. It's the instrument that scores every other lesson in the module.** A fleet with a slow RMA loop, an under-provisioned storage tier, or a control plane that can't survive a node loss doesn't just have reliability problems — it has a worse number in the table above.

**Close the module here.** Build both halves of the [capex-vs-cloud + KTHW/etcd deliverable](../practice/capex-vs-cloud/README.md) — the crossover model from this lesson and the hand-built-control-plane/etcd-drill writeup anchored in lessons 01–02 — and then work the [module checkpoint](../checkpoint.md) cold: provision from scratch, recover etcd inside 30 minutes, diagram the HA design, close the health loop, defend your own economics, and stand up a MetalLB BGP VIP without notes. That checkpoint is the gate this whole module has been building toward.

## References & further reading

**Primary sources**
- **MetalLB — BGP concepts** — <https://metallb.universe.tf/concepts/bgp/> — the authoritative L2-vs-BGP reference behind self-check (b) and the worked example.
- **kube-vip** — <https://kube-vip.io/> — the control-plane-VIP reference for the ARP/BGP modes and leader-election mechanics.
- **arXiv:2506.14551 — "Empirically-Calibrated H100 Node Power Models for Reducing Uncertainty in AI Training Energy Estimation"** — <https://arxiv.org/abs/2506.14551> — read for the empirical 8×H100 peak/typical power measurements (~8.4 kW peak, ~7.3 kW typical) that correct vendor "IT-load-only" power figures used in the model's power line.

**Real-world engineering blogs**
- **MetalLB ADOPTERS.md** — <https://github.com/metallb/metallb/blob/main/ADOPTERS.md> — what it shows: directly verified, named production adopters (1&1 Mail & Media since 2018, Nokia EDA since 2024) — the strongest single production citation for the MetalLB half of this lesson.
- **Red Hat — "Deploying a high-availability, fault-tolerant Kubernetes Service on bare metal clusters with MetalLB BGP"** — <https://www.redhat.com/en/blog/deploying-a-high-availability-fault-tolerant-kubernetes-service-on-baremetal-clusters-with-metallb-bgp> — what it shows: a vendor reference architecture combining BGP-mode MetalLB with HA bare-metal control planes, a second worked pattern to check your manifests against.
- **SemiAnalysis — "How Much Do GPU Clusters Really Cost?"** — <https://newsletter.semianalysis.com/p/how-much-do-gpu-clusters-really-cost> — what it shows: dated, granular 2025/2026 cluster-cost data (576-GPU cluster ~$36M, 8-GPU B200/B300-gen server ~$250K–$400K) to cross-check the model's capex line against.
- **CloudZero — "H100 GPU Cost In 2026"** — <https://www.cloudzero.com/blog/h100-gpu-cost/> — what it shows: the "$1.45/hr all-in owned vs. $3.99/hr on-demand — scarcity rent, not cost recovery" framing used in the Core concepts section above.
- **IntuitionLabs — "H100 Rental Prices Compared"** — <https://intuitionlabs.ai/articles/h100-rental-prices-cloud-comparison> — what it shows: the rental-price volatility data point (1-yr contract rate +40% in five months) that is the concrete evidence behind "flag every $ as a snapshot."
- **CoreWeave — "What Is Node Lifecycle Management?"** (cross-referenced from lessons 05/06) — <https://www.coreweave.com/blog/what-is-node-lifecycle-management-ml-training-and-inference> — what it shows: the RMA-turnaround-to-utilisation link named in self-check (e).

**Deeper dives**
- **GPUaaS — "The Real TCO of a GPU Cluster (2026)"** — <https://gpuaas.com/blog/real-tco-gpu-cluster-2026> — a broader input set for the model: hardware, power, staff, the hidden lines. Flag its figures as 2026 snapshots and swap in yours.
- **Thunder Compute — "Neoclouds: the new GPU clouds"** — <https://www.thundercompute.com/blog/neoclouds-the-new-gpu-clouds-changing-ai-infrastructure> — the business-model framing (the arbitrage your crossover chart formalises) and the rental-rate rail.
