---
lesson: "A02.4"
title: "Cloud networking"
module: "A-02"
concept: "price+latency per hop"
status: not-started
est_time: "3.5 hrs"
prev: "03-load-balancing.md"
next: "05-kubernetes-networking.md"
artifacts: ["per-hop-cost-latency-table", "64-gpu-transfer-cost-model"]
sources: 7
---

# A02.4 · Cloud networking

> **Concept.** Every network hop has a price and a latency — staff-level cloud networking is putting a dollar and a millisecond on each one, because egress and cross-AZ chatter routinely dominate the bill and topology decides whether the data sits next to the GPUs.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits
Lesson 03 established the load balancer as a stateful control point inside one AZ (or spread deliberately across a few). This lesson takes that same LB tier — and everything else in the request path — and asks what happens when its edges cross an availability zone, a region, or the boundary of the VPC entirely. The LB's cross-zone spray from Lesson 03 turns out to be one of the two dominant egress-cost culprits here, so this lesson is where the previous lesson's mechanism gets a dollar figure attached. It sets up Lesson 05 (Kubernetes networking), where topology-aware routing and `internalTrafficPolicy: Local` become the concrete K8s-level levers for the cost-and-latency tradeoffs introduced here.

## Why this matters
Senior engineers draw the VPC diagram correctly. Staff engineers annotate every edge with `$/GB` and `ms`, then notice that the innocuous-looking cross-AZ replication arrow is 15% of the monthly bill. Data transfer is the silent line item that finance escalates and nobody on the eng side can explain — because the mechanics (who gets billed, in which direction, with what surcharge) live below where most people look. This is also the bridge into cost work and the GPU-fleet reality: for large training jobs the *data movement* can dwarf the GPU-hour spend, and cross-AZ placement taxes every collective.

## What's new here (calibration)
- **Skipped:** VPC/subnet/route-table/IGW/NAT-GW mechanics, SGs vs NACLs, what peering/TGW/PrivateLink do at a concept level, and that cross-region is slower and costs more.
- **New depth — billing mechanics, not just pricing.** Who actually gets billed, on which invoice line, in which direction — the "each direction" cross-AZ double-charge, the NAT Gateway double-stack (processing fee *plus* egress rate), and the ALB-vs-NLB cross-zone default asymmetry are all things the pricing page states but doesn't emphasize.
- **New depth — cost as an architecture-review input, not a monitoring afterthought.** The placement decision (same-AZ vs spread-AZ) is made *before* the cost is incurred; this lesson treats it as a design-time question with a comparable-units answer (failure budget in dollars vs transfer cost in dollars), not a "we'll optimize it later" afterthought.
- **New depth — training-specific compounding.** For a synchronous training job, cross-AZ placement doesn't just add a transfer bill — it adds latency to *every* collective operation on the critical path of *every* step, so the same topology decision shows up twice: once on the invoice, once in wall-clock training time.

## Core concepts

**Put a price and a latency on every hop.** Memorize the ladder (all figures below are a **2026 pricing snapshot** — cloud transfer pricing changes over time and by provider/region; verify against the current pricing page before quoting a number in a real cost review):
- *Same-AZ:* ≈ free, sub-100µs.
- *Cross-AZ:* ≈ **$0.01/GB in each direction** — both sender and receiver are billed, so a cross-AZ transfer is ~**$0.02/GB aggregate** — plus ~0.5–1ms. The "each direction" double-charge is the detail people miss.
- *Cross-region:* region-pair pricing + **10s–100s of ms** RTT.
- *Internet egress:* **$0.05–0.09/GB**, tiered down with volume; ingress free.
- *NAT Gateway:* adds a **per-GB processing charge on top of egress** — ~$0.045/GB processing + ~$0.09/GB internet egress stacks to roughly **$0.135/GB total** for a private-subnet instance pulling data from the public internet through a NAT. This is a silent cost sink for chatty egress through a NAT, because you pay the NAT surcharge *and* the egress rate, on every byte.

**Who gets billed differs by provider — check the meter before proposing a fix.** The exact mechanic above (sender-and-receiver-both-billed for cross-AZ) is an AWS-specific detail; other providers fold NAT-equivalent processing into a single line, or bill cross-zone differently. The FinOps discipline that generalizes is: identify which meter is actually firing (VPC flow logs, cost-allocation tags, the provider's billing console broken out by usage type) *before* proposing an architecture change to fix a bill you haven't actually attributed to a hop yet. A fix aimed at the wrong meter doesn't move the number.

**The NAT-processing leg can often be removed entirely.** For traffic to AWS-native services (S3, DynamoDB), a **VPC Gateway Endpoint** routes the traffic over the AWS backbone instead of out through the NAT Gateway, removing the NAT-processing leg ($0.045/GB) from the path entirely — the traffic never touches internet egress pricing either, since it's other AWS resources, not the public internet. This is the single highest-leverage cost fix available for the common "private-subnet instance talks to S3 via NAT" pattern, and it's frequently missing from real infrastructure simply because nobody put a price on that hop.

**Egress is the dominant cost driver.** It is routinely 10–20% of a large cloud bill. The two usual culprits: (1) cross-AZ chatter from replicated stateful systems (databases, Kafka, caches replicating across AZs — every replication byte is billed both ways), and (2) cross-zone load balancing spraying to backends in all AZs regardless of where the request landed. On that second culprit, the default behavior is a common surprise: **ALB has cross-zone load balancing enabled by default** (traffic can land on any AZ's targets, spreading load evenly but incurring cross-AZ charges), while **NLB has it disabled by default, per target group** (traffic stays in-zone unless you opt in). Migrating a service from ALB to NLB — or vice versa — without checking this default is a documented source of both surprise bills and surprise imbalance. Fixes: disable cross-zone LB where you can tolerate the imbalance; use topology-aware routing — Kubernetes `internalTrafficPolicy: Local` and topology hints — to keep traffic in-zone; and colocate chatty replicas. The staff move is to *measure per-hop bytes* (VPC flow logs, cost allocation tags) before optimizing compute.

**Transit topologies and their cost shapes.** Hub-and-spoke via **Transit Gateway** charges *per-attachment (per-hour) + per-GB* processed — clean management, but the per-GB fee applies to everything transiting the hub. Full-mesh **VPC peering** has *no per-GB fee* (you pay only the underlying cross-AZ/region transfer) but is **O(n²)** to manage and non-transitive. **PrivateLink** is a different animal: *one-way, service-scoped, non-transitive* (ideal for exposing a service across a trust/security boundary, e.g. SaaS), with its own *per-hour endpoint + per-GB* cost — and that per-hour endpoint charge applies **per Availability Zone** the endpoint is provisioned in, so a highly-available PrivateLink setup (endpoints in 3 AZs) triples the fixed hourly cost versus a single-AZ endpoint. Pick by the shape of the traffic and the trust boundary, and state the cost formula for each — that's the interview-fluency signal.

**Hybrid to on-prem.** **Direct Connect** = a dedicated physical circuit: consistent bandwidth, consistent latency, and *cheaper egress* than internet. **VPN** = IPsec over the public internet: cheap to stand up, variable latency, internet-grade reliability. The recurring silent breakage on either tunnel is **MTU/MSS**: encapsulation shrinks the usable MTU, and if PMTU discovery is blocked (ICMP filtered) you get a **PMTU black hole** — large packets silently dropped, small ones fine, so `ping` works but bulk transfers hang. Direct Connect's dedicated circuit reduces the *variability* of the path but does not remove this failure mode — Direct Connect still runs encapsulated/tunneled protocols (VLANs, sometimes MACsec or an overlay for private connectivity) that can hit the same PMTU black hole if ICMP is filtered anywhere on the path. The fix is about the encapsulation, not the circuit type: **MSS clamping** on the tunnel/circuit interface is the standard fix regardless of whether the underlying transport is a dedicated fiber or a public-internet VPN.

**AZ topology reality — express it in comparable units.** An AZ *is* a failure domain — cross-AZ redundancy is what buys you resilience to a zone loss. But it costs dollars (cross-AZ transfer) and latency (~0.5–1ms per hop, which compounds in chatty or synchronous paths). The staff answer never says "spread across AZs" or "keep it in one AZ" flatly — it puts both sides of the tradeoff in the *same unit*: the workload's failure budget (its RTO/RPO, translated into what a zone-loss outage would cost in dollars — lost revenue, SLA penalties, incident cost) versus the transfer cost of AZ-spreading (dollars per month, computed the way the worked example below does it). "Spread across 3 AZs" is a good answer only when you can show the failure-budget dollars it buys exceed the transfer dollars it costs; otherwise it's a reflexive best-practice applied without the arithmetic behind it.

**Training-specific compounding.** For an all-reduce or other synchronous collective, cross-AZ node placement doesn't just add a transfer bill — the ~0.5–1ms cross-AZ latency lands on the *critical path of every synchronization step*, multiplied by however many steps the job runs. A chatty, frequently-synchronizing training job (small batch, high communication-to-compute ratio) pays that latency tax far more times than a job with large batches and infrequent synchronization — so the same topology decision shows up twice: once as a monthly transfer-dollar line item, once as extra wall-clock time (and therefore extra GPU-hour dollars) for the entire job. This is the reason placement for multi-node training jobs is treated as a from-scratch design decision rather than "just spread it across AZs for resilience like everything else."

## Perspectives

**FinOps / accounting.** The first question in any cost investigation isn't "how do we reduce this" — it's "which meter is actually firing, and on whose invoice line." AWS bills cross-AZ transfer to both the sender and the receiver separately, which is why the aggregate rate looks doubled versus the quoted per-direction rate; other providers fold NAT-equivalent processing into a single combined line rather than billing it as a separate meter. Proposing an architecture fix before confirming which specific meter produced the anomalous charge is a common and expensive mistake — the fix has to target the actual billing mechanic, not an assumption about it.

**Topology design.** The AZ-spread-or-not decision for a given workload is made at architecture-review time, before a single byte moves and before any cost has accrued — it is not something you discover from a Cost Explorer dashboard after the fact and then patch. Treating it as a design-time decision means the tradeoff (failure budget vs transfer cost, in the same units) belongs in the architecture doc and the review conversation, not in a retroactive cost-optimization ticket three months later.

**Failure-domain-vs-cost tension.** The honest staff framing of "should this be same-AZ or spread across AZs" converts both sides into dollars: what does the failure budget (RTO/RPO) cost if you're wrong — measured in incident cost, SLA penalty, lost revenue during a zone outage — against what the AZ-spread transfer cost is, computed per the worked example's methodology. When both sides are in the same unit, the decision stops being a reflex ("always spread for resilience," or "always same-AZ to save money") and becomes an actual tradeoff analysis a staff engineer can defend in a design review.

**AI/ML-specific.** For inference, cross-AZ or cross-region placement is mostly a latency-and-cost question per request. For *training*, it's compounded by the synchronization requirement: every all-reduce (or other collective) that spans AZ-separated nodes puts that AZ's extra latency on the critical path of every single step, not just on the data transfer bill. A chatty collective communication pattern turns a small per-hop latency tax into a large aggregate slowdown across a long training run — so for training specifically, topology decisions are doubly expensive when made carelessly: dollars on the transfer bill, and dollars again in extended GPU-hours from the slower wall-clock time.

## Real-world use cases

- **Spheron Network, "GPU Cloud Egress & Data Transfer Costs for AI Workloads"** — https://www.spheron.network/blog/gpu-cloud-egress-data-transfer-costs-ai-workloads-2026/ — a GPU-cloud-specific breakdown of how egress and cross-AZ costs hit AI/ML workloads in particular, the direct analog of this lesson's worked example.
- **Last Week in AWS (Corey Quinn), "AWS Cross-AZ Data Transfer Costs More Than AWS Says"** — https://www.lastweekinaws.com/blog/aws-cross-az-data-transfer-costs-more-than-aws-says/ — an independent, skeptical audit of the "each direction" double-billing mechanic; a useful contrarian, staff-level "verify the pricing page yourself, don't just trust it" source.
- **AWS, "Analyze Data Transfer and adopt cost optimized designs to realize cost savings"** — https://aws.amazon.com/blogs/industries/analyze-data-transfer-and-adopt-cost-optimized-designs-to-realize-cost-savings/ — the official AWS playbook for the "measure per-hop bytes before optimizing" discipline this lesson recommends; ties the flow-logs/cost-allocation-tags approach to concrete design patterns.

## Worked example
**Cost-model a 64-GPU training job.** 8 nodes × 8 GPUs (e.g. H100), checkpoint every 30 min at 400 GB replicated cross-AZ; a 5 TB dataset pulled once cross-region at job start. *(All dollar figures below are a 2026 pricing snapshot — verify current rates before using in a real budget.)*

*Naive placement (checkpoints replicated cross-AZ, dataset cross-region):*
- Checkpoints: 400 GB × 48/day × 30 days = 576,000 GB/month crossing AZs. At ~$0.02/GB aggregate ≈ **~$11,500/month** just moving checkpoints between zones.
- Dataset: 5 TB = 5,000 GB cross-region once. At a representative ~$0.02/GB region-pair rate ≈ **$100** one-time (and far worse if the pipeline re-pulls per run).
- Compare to GPU spend: 64 × ~$3/GPU-hr × 720 hr ≈ **~$138,000/month**. The checkpoint transfer alone is ~8% of the compute — a real, avoidable line item.

*Optimized (same-AZ checkpoint replication, in-region/same-AZ dataset bucket):*
- Same-AZ checkpoints ≈ free (or the local storage cost only) → the ~$11.5k collapses toward $0.
- Dataset from an in-region, ideally same-AZ bucket, reached via a **VPC Gateway Endpoint** rather than through a NAT Gateway if it's in S3 → cross-region charge gone, NAT-processing leg removed too; only same-AZ/free movement remains.
- If the job instead spans multiple AZs deliberately for node-failure resilience, add the **critical-path latency tax**: every all-reduce step now includes ~0.5–1ms of cross-AZ RTT, which — for a chatty small-batch job running, say, 50,000 steps — can add tens of seconds to minutes of pure synchronization overhead across the run, itself a GPU-hour cost on top of the transfer-dollar cost above. State both numbers side by side when justifying (or rejecting) cross-AZ node placement for a training job.

**Deliverable:** a per-hop cost/latency table for the job topology — one row per edge (GPU↔GPU intra-node fabric, node↔node intra-AZ, node↔checkpoint-store, job↔dataset-bucket), each annotated with `$/GB`, direction billed, and `ms` — plus the monthly total for naive vs optimized. The headline: *keep the data next to the GPUs*, and note that cross-AZ *placement* of a multi-node job also adds inter-node latency to *every* collective operation, not just to checkpointing. (For the intra-node fabric numbers, reference the GPU-fabric artifact from module 09 rather than re-deriving them here.)

## Practice
<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

Produce two artifacts: (1) the per-hop cost/latency table from the worked example, generalized into a reusable template the runbook can apply to any job topology (columns: hop, `$/GB`, direction billed, RTT, notes, "snapshot date" for the dollar figures). (2) A spreadsheet or script `64-gpu-transfer-cost-model` computing naive vs optimized monthly transfer dollars against the GPU-hour spend, with the checkpoint cadence and dataset size as parameters — and a toggle for NAT Gateway vs VPC Gateway Endpoint on the dataset-pull leg, so the NAT double-stack cost is visible as a line item rather than folded into "egress." In the runbook, add a debug section for the *PMTU black hole*: symptom (small packets/ping fine, bulk transfer hangs), how to confirm (path-MTU probe with `ping -M do -s`), and the MSS-clamp fix, explicitly noting this applies to Direct Connect as much as VPN — then tie it back to why a cross-AZ multi-node collective is both a cost and a latency decision.

## Common pitfalls
- **"Egress pricing is flat."** It's tiered and decreases with volume — check the tier breakpoints before doing back-of-envelope math on a large-volume workload, or you'll overestimate the cost of scale.
- **"VPC peering has no cost."** There's no per-GB *peering* fee, but the cross-AZ or cross-region transfer that rides over the peering connection is still billed at the normal rate — peering removes a management/topology cost, not a transfer cost.
- **"PrivateLink is transitive like peering."** It's deliberately one-way and service-scoped by design — a consumer can reach the exposed service, but PrivateLink doesn't create general network reachability the way peering or a Transit Gateway attachment does. That's a feature (trust-boundary isolation), not a limitation to work around.
- **"Same-AZ is always cheaper and therefore always right."** This collapses the failure domain to a single AZ — it optimizes only the cost side of the tradeoff and ignores the resilience side entirely. State both the cost saved and the blast radius accepted; let the workload's failure budget decide.
- **"Direct Connect eliminates the MTU problem."** A dedicated circuit reduces path *variability*, but any tunneled or encapsulated protocol running over it can still hit a PMTU black hole if ICMP "fragmentation needed" is filtered somewhere on the path. The fix (MSS clamping) is about the encapsulation, not about which physical circuit carries it.

## Self-check
- A cross-AZ transfer is quoted at "$0.01/GB" but your bill shows roughly double per replicated gigabyte — why? **Answer:** Cross-AZ transfer is billed **in each direction**: the sender is charged for egress out of its AZ and the receiver is charged for egress out of the peer AZ, so a single logical GB crossing zones costs ~$0.01 + ~$0.01 ≈ **$0.02 aggregate**. Replicated stateful systems (DBs, Kafka, caches) copy every write across AZs, so their replication traffic gets this double-charge on every byte — which is why cross-AZ chatter dominates so many bills.
- For a 64-GPU training job, why can data movement dwarf the GPU-hour cost, and what's the one-line fix? **Answer:** Training artifacts are huge and repetitive — multi-hundred-GB checkpoints written every few minutes and replicated cross-AZ, plus multi-TB datasets pulled cross-region — and each of those bytes is billed (cross-AZ both directions, cross-region at region-pair rates), so the transfer total can reach a meaningful fraction of, or exceed, the compute for I/O-heavy pipelines. The fix: *keep the data next to the GPUs* — same-AZ checkpoint replication and an in-region/same-AZ dataset bucket collapse the transfer cost toward zero and remove inter-node latency from every collective.
- `ping` between two sites over a Direct Connect / VPN tunnel succeeds, but large bulk transfers hang — what's happening and how do you fix it? **Answer:** A **PMTU black hole**: tunnel encapsulation lowers the usable MTU, and because ICMP "fragmentation needed" is filtered somewhere on the path, Path MTU Discovery can't signal the sender to shrink packets. Small packets (ping) fit and pass; full-size packets exceed the tunnel MTU, can't fragment (DF set), and are silently dropped — so the connection stalls on bulk data. Fix by **MSS clamping** on the tunnel interface (advertise a smaller MSS so TCP never emits oversized segments), or lower the interface MTU / unblock the needed ICMP. This applies to Direct Connect as much as to VPN — the dedicated circuit doesn't remove the encapsulation.
- Your private-subnet training nodes pull a large dataset from S3 through a NAT Gateway every run — what is the effective $/GB, and what single change removes the largest chunk of it? **Answer:** The traffic pays two stacked charges: the NAT Gateway's per-GB data-processing fee (~$0.045/GB) *plus* the internet-egress rate (~$0.09/GB), for a combined ~$0.135/GB — even though the traffic's actual destination (S3) is inside AWS's own network. Adding a **VPC Gateway Endpoint** for S3 routes that traffic over the AWS backbone instead of through the NAT Gateway, removing the NAT-processing leg entirely (and the traffic no longer counts as internet egress either), which is the highest-leverage single fix for this pattern.
- You migrate a service's load balancer from ALB to NLB and per-AZ backend load becomes uneven, with a new line item for cross-zone transfer appearing on the bill — what changed? **Answer:** ALB has cross-zone load balancing **enabled by default**, spreading requests evenly across targets in every AZ regardless of which AZ the request arrived in (at the cost of cross-AZ transfer charges). NLB has cross-zone load balancing **disabled by default**, per target group, so traffic stays in the AZ it arrived in unless you explicitly opt in — which both changes the per-AZ load distribution (now dependent on where requests land) and removes the automatic cross-AZ transfer charge the ALB was silently incurring. The fix, if even distribution matters more than the transfer cost, is to explicitly enable cross-zone balancing on the NLB's target group.

## Connections & what's next
This lesson's cost lens builds directly on **Lesson 03**'s cross-zone load-balancing mechanics (one of the two dominant egress culprits named above) and sets up **Lesson 05** (Kubernetes networking), where `internalTrafficPolicy: Local` and topology-aware routing become the concrete K8s-level tools for keeping traffic in-zone. The training-specific compounding effect described here (latency on the critical path of every collective) is the request-tier bridge into **Lesson 07** (GPU/RDMA networking) and module 09's GPU-fabric artifact, which covers the intra-node fabric this lesson deliberately stayed above.

Next: **[05-kubernetes-networking.md](05-kubernetes-networking.md)** — where a Service VIP actually resolves (iptables/IPVS/eBPF), and where the topology-aware routing levers referenced above live in the K8s dataplane.

## References & further reading

**Primary sources**
- AWS EC2 On-Demand Pricing — https://aws.amazon.com/ec2/pricing/on-demand/
- AWS — Elastic Load Balancing, cross-zone load balancing docs — https://docs.aws.amazon.com/elasticloadbalancing/latest/application/disable-cross-zone.html
- AWS Direct Connect Pricing — https://aws.amazon.com/directconnect/pricing/
- Kubernetes docs — Topology Aware Routing — https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/

**Real-world engineering blogs**
- Spheron Network — "GPU Cloud Egress & Data Transfer Costs for AI Workloads" — https://www.spheron.network/blog/gpu-cloud-egress-data-transfer-costs-ai-workloads-2026/
- Last Week in AWS — "AWS Cross-AZ Data Transfer Costs More Than AWS Says" — https://www.lastweekinaws.com/blog/aws-cross-az-data-transfer-costs-more-than-aws-says/

**Deeper dives**
- AWS — "Analyze Data Transfer and adopt cost optimized designs to realize cost savings" — https://aws.amazon.com/blogs/industries/analyze-data-transfer-and-adopt-cost-optimized-designs-to-realize-cost-savings/
