---
lesson: "A02.4"
title: "Cloud networking"
module: "A-02"
concept: "price+latency per hop"
status: not-started
est_time: "3 hrs"
artifacts: ["per-hop-cost-latency-table", "64-gpu-transfer-cost-model"]
---

# A02.4 · Cloud networking

> **Concept.** Every network hop has a price and a latency — staff-level cloud networking is putting a dollar and a millisecond on each one, because egress and cross-AZ chatter routinely dominate the bill and topology decides whether the data sits next to the GPUs.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Why this matters
Senior engineers draw the VPC diagram correctly. Staff engineers annotate every edge with `$/GB` and `ms`, then notice that the innocuous-looking cross-AZ replication arrow is 15% of the monthly bill. Data transfer is the silent line item that finance escalates and nobody on the eng side can explain — because the mechanics (who gets billed, in which direction, with what surcharge) live below where most people look. This is also the bridge into cost work and the GPU-fleet reality: for large training jobs the *data movement* can dwarf the GPU-hour spend, and cross-AZ placement taxes every collective.

## Core notes
**Skip (you already know):** VPC/subnet/route-table/IGW/NAT-GW mechanics, SGs vs NACLs, what peering/TGW/PrivateLink do at a concept level, and that cross-region is slower and costs more.

**Put a price and a latency on every hop.** Memorize the ladder:
- *Same-AZ:* ≈ free, sub-100µs.
- *Cross-AZ:* ≈ **$0.01/GB in each direction** — both sender and receiver are billed, so a cross-AZ transfer is ~**$0.02/GB aggregate** — plus ~0.5–1ms. The "each direction" double-charge is the detail people miss.
- *Cross-region:* region-pair pricing + **10s–100s of ms** RTT.
- *Internet egress:* **$0.05–0.09/GB**, tiered down with volume; ingress free.
- *NAT Gateway:* adds a **per-GB processing charge on top of egress** — a silent cost sink for chatty egress through a NAT, because you pay the NAT surcharge *and* the egress rate.

**Egress is the dominant cost driver.** It is routinely 10–20% of a large cloud bill. The two usual culprits: (1) cross-AZ chatter from replicated stateful systems (databases, Kafka, caches replicating across AZs — every replication byte is billed both ways), and (2) cross-zone load balancing spraying to backends in all AZs regardless of where the request landed. Fixes: disable cross-zone LB where you can tolerate the imbalance; use topology-aware routing — Kubernetes `internalTrafficPolicy: Local` and topology hints — to keep traffic in-zone; and colocate chatty replicas. The staff move is to *measure per-hop bytes* (VPC flow logs, cost allocation tags) before optimizing compute.

**Transit topologies and their cost shapes.** Hub-and-spoke via **Transit Gateway** charges *per-attachment (per-hour) + per-GB* processed — clean management, but the per-GB fee applies to everything transiting the hub. Full-mesh **VPC peering** has *no per-GB fee* (you pay only the underlying cross-AZ/region transfer) but is **O(n²)** to manage and non-transitive. **PrivateLink** is a different animal: *one-way, service-scoped, non-transitive* (ideal for exposing a service across a trust/security boundary, e.g. SaaS), with its own *per-hour endpoint + per-GB* cost. Pick by the shape of the traffic and the trust boundary, and state the cost formula for each — that's the interview-fluency signal.

**Hybrid to on-prem.** **Direct Connect** = a dedicated physical circuit: consistent bandwidth, consistent latency, and *cheaper egress* than internet. **VPN** = IPsec over the public internet: cheap to stand up, variable latency, internet-grade reliability. The recurring silent breakage on either tunnel is **MTU/MSS**: encapsulation shrinks the usable MTU, and if PMTU discovery is blocked (ICMP filtered) you get a **PMTU black hole** — large packets silently dropped, small ones fine, so `ping` works but bulk transfers hang. **MSS clamping** on the tunnel interface is the standard fix. Naming this failure mode unprompted reads as real operational depth.

**AZ topology reality.** An AZ *is* a failure domain — cross-AZ redundancy is what buys you resilience to a zone loss. But it costs dollars (cross-AZ transfer) and latency (~0.5–1ms per hop, which compounds in chatty or synchronous paths). The staff answer never says "spread across AZs" or "keep it in one AZ" flatly — it states the tradeoff: resilience vs cost-and-latency, decided by the workload's failure budget and its traffic pattern.

## Worked example
**Cost-model a 64-GPU training job.** 8 nodes × 8 GPUs (e.g. H100), checkpoint every 30 min at 400 GB replicated cross-AZ; a 5 TB dataset pulled once cross-region at job start.

*Naive placement (checkpoints replicated cross-AZ, dataset cross-region):*
- Checkpoints: 400 GB × 48/day × 30 days = 576,000 GB/month crossing AZs. At ~$0.02/GB aggregate ≈ **~$11,500/month** just moving checkpoints between zones.
- Dataset: 5 TB = 5,000 GB cross-region once. At a representative ~$0.02/GB region-pair rate ≈ **$100** one-time (and far worse if the pipeline re-pulls per run).
- Compare to GPU spend: 64 × ~$3/GPU-hr × 720 hr ≈ **~$138,000/month**. The checkpoint transfer alone is ~8% of the compute — a real, avoidable line item.

*Optimized (same-AZ checkpoint replication, in-region/same-AZ dataset bucket):*
- Same-AZ checkpoints ≈ free (or the local storage cost only) → the ~$11.5k collapses toward $0.
- Dataset from an in-region, ideally same-AZ bucket → cross-region charge gone; only same-AZ/free movement remains.

**Deliverable:** a per-hop cost/latency table for the job topology — one row per edge (GPU↔GPU intra-node fabric, node↔node intra-AZ, node↔checkpoint-store, job↔dataset-bucket), each annotated with `$/GB`, direction billed, and `ms` — plus the monthly total for naive vs optimized. The headline: *keep the data next to the GPUs*, and note that cross-AZ *placement* of a multi-node job also adds inter-node latency to *every* collective operation, not just to checkpointing. (For the intra-node fabric numbers, reference the GPU-fabric artifact from module 09 rather than re-deriving them here.)

## Practice
<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

Produce two artifacts: (1) the per-hop cost/latency table from the worked example, generalized into a reusable template the runbook can apply to any job topology (columns: hop, `$/GB`, direction billed, RTT, notes). (2) A spreadsheet or script `64-gpu-transfer-cost-model` computing naive vs optimized monthly transfer dollars against the GPU-hour spend, with the checkpoint cadence and dataset size as parameters. In the runbook, add a debug section for the *PMTU black hole*: symptom (small packets/ping fine, bulk transfer hangs), how to confirm (path-MTU probe with `ping -M do -s`), and the MSS-clamp fix — then tie it back to why a cross-AZ multi-node collective is both a cost and a latency decision.

## Self-check
- A cross-AZ transfer is quoted at "$0.01/GB" but your bill shows roughly double per replicated gigabyte — why? **Answer:** Cross-AZ transfer is billed **in each direction**: the sender is charged for egress out of its AZ and the receiver is charged for egress out of the peer AZ, so a single logical GB crossing zones costs ~$0.01 + ~$0.01 ≈ **$0.02 aggregate**. Replicated stateful systems (DBs, Kafka, caches) copy every write across AZs, so their replication traffic gets this double-charge on every byte — which is why cross-AZ chatter dominates so many bills.
- For a 64-GPU training job, why can data movement dwarf the GPU-hour cost, and what's the one-line fix? **Answer:** Training artifacts are huge and repetitive — multi-hundred-GB checkpoints written every few minutes and replicated cross-AZ, plus multi-TB datasets pulled cross-region — and each of those bytes is billed (cross-AZ both directions, cross-region at region-pair rates), so the transfer total can reach a meaningful fraction of, or exceed, the compute for I/O-heavy pipelines. The fix: *keep the data next to the GPUs* — same-AZ checkpoint replication and an in-region/same-AZ dataset bucket collapse the transfer cost toward zero and remove inter-node latency from every collective.
- `ping` between two sites over a Direct Connect / VPN tunnel succeeds, but large bulk transfers hang — what's happening and how do you fix it? **Answer:** A **PMTU black hole**: tunnel encapsulation lowers the usable MTU, and because ICMP "fragmentation needed" is filtered somewhere on the path, Path MTU Discovery can't signal the sender to shrink packets. Small packets (ping) fit and pass; full-size packets exceed the tunnel MTU, can't fragment (DF set), and are silently dropped — so the connection stalls on bulk data. Fix by **MSS clamping** on the tunnel interface (advertise a smaller MSS so TCP never emits oversized segments), or lower the interface MTU / unblock the needed ICMP.

## References
- https://aws.amazon.com/ec2/pricing/on-demand/
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/disable-cross-zone.html
- https://www.spheron.network/blog/gpu-cloud-egress-data-transfer-costs-ai-workloads-2026/
- https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/
- https://aws.amazon.com/directconnect/pricing/
