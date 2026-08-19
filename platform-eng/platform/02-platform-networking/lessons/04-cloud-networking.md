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
sources: 10
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
- **New depth — the price list as a primary source.** Every AWS figure below is read from the **AWS Price List Bulk API** (`pricing.us-east-1.amazonaws.com`) for `us-east-1`, and every GCP figure from Google's published network-pricing page. Publication dates are stated per table. You will learn how to look these up yourself, which matters more than the numbers, because the numbers move.
- **New depth — the meter, not the price.** Which SKU (`usagetype`) fires decides whether a fix works. A cross-AZ byte can land on `DataTransfer-Regional-Bytes` or on `DataTransfer-xAZ-Out-Bytes`, and those are priced differently in the current price list.
- **New depth — latency derived, not quoted.** Fibre propagation is 200 km per millisecond round trip. Every latency claim below is derived from distance rather than recited, so you can sanity-check any provider's number.
- **New depth — cost as an architecture-review input.** The same-AZ vs spread-AZ decision is made before the cost is incurred, and both sides go in the same unit: failure budget in dollars against transfer cost in dollars.
- **New depth — training-specific compounding.** Cross-AZ placement adds latency to *every collective on the critical path of every step*, so one topology decision shows up twice: on the invoice and in wall-clock GPU-hours.

## Core concepts

### 1. Read the meter, then argue about the price

A cloud bill is not a price list; it is a set of **meters**. Each meter has a `usagetype` string, fires on a specific event, and is attributed to a specific account and resource. Nearly every failed cost-optimisation project fails at the same step: someone proposed an architecture change against a *guessed* meter.

The discipline is three steps, in order:

1. **Find the usage type.** In AWS, the Cost and Usage Report's `lineItem/UsageType` column; in the console, Cost Explorer grouped by "Usage Type." You are looking for strings like `USE1-DataTransfer-Regional-Bytes`, `USE1-EU-AWS-Out-Bytes`, `NatGateway-Bytes`, `USE1-VpcEndpoint-Bytes`. In GCP, the SKU description on the billing export.
2. **Find which resource generated it**, using flow logs or cost-allocation tags. Bytes are attributed to an ENI/instance/subnet, which is what turns "$40k of cross-AZ transfer" into "the Kafka cluster."
3. **Only then** propose the change, and predict the new bill by editing the *same arithmetic*.

**How to look up any AWS price yourself, without the pricing console.** The bulk price list is public, unauthenticated JSON:

```bash
# 1. The offer index lists every service's price file.
curl -s https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/index.json \
  | jq -r '.offers | keys[]' | grep -i transfer
#   AWSDataTransfer
#   AWSDataTransferTerminal

# 2. Each offer has a per-region index, so you don't download the global file.
curl -s https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDataTransfer/current/region_index.json \
  | jq -r '.regions["us-east-1"].currentVersionUrl'
#   /offers/v1.0/aws/AWSDataTransfer/20260720184645/us-east-1/index.json

# 3. Join products to their on-demand price dimensions.
curl -s https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSDataTransfer/20260720184645/us-east-1/index.json \
  | jq -r '
      .publicationDate as $d
      | .products as $p
      | .terms.OnDemand
      | to_entries[]
      | .key as $sku
      | .value[].priceDimensions[]
      | select(.pricePerUnit.USD != "0.0000000000")
      | "\($p[$sku].attributes.usagetype)  \(.pricePerUnit.USD)/\(.unit)  \(.description)"'
```

**Do this before every cost conversation.** It takes two minutes, it is authoritative, and it is dated — which is the part that matters, because everything in the next section has a shelf life.

### 2. The AWS ladder, from the price list

All figures below are read from the AWS Price List Bulk API for **`us-east-1`**, with the publication date each file carries. Other regions differ; re-run the queries above for yours.

**Data transfer** (`AWSDataTransfer`, published **2026-07-20**):

| Meter (`usagetype`) | Price | What triggers it |
|---|---|---|
| `DataTransfer-Regional-Bytes` | **$0.010 / GB** | "regional data transfer — in/out/between EC2 AZs or using elastic IPs or ELB." This is the classic cross-AZ meter, and it fires on **both** the sending and the receiving side, which is why a logical GB crossing zones is commonly quoted at ~$0.02 aggregate. |
| `USE1-DataTransfer-xAZ-Out-Bytes` | **$0.00 / GB** | an explicit cross-AZ *out* meter, currently priced at zero in this publication |
| `USE1-DataTransfer-xAZ-In-Bytes` | **$0.00 / GB** | the matching *in* meter, currently zero |
| `USE1-EU-AWS-Out-Bytes` (and peers) | **$0.020 / GB** | inter-region **outbound**, us-east-1 → eu-west-1. Same rate to us-west-2 and ap-northeast-1 in this publication. |
| `USE1-EU-AWS-In-Bytes` | **$0.000 / GB** | inter-region **inbound** — free. Cross-region is billed once, on the sending region. |
| `DataTransfer-Out-Bytes` (internet) | **$0.090 / GB** first 10 TB/mo | tiered — see below |
| " | **$0.085 / GB** next 40 TB | |
| " | **$0.070 / GB** next 100 TB | |
| " | **$0.050 / GB** above 150 TB | |
| inbound from internet | **$0.00 / GB** | ingress is free everywhere |

**Two things in that table deserve to change how you talk about this.**

First, **cross-AZ is metered in two different families right now**, and the modern explicit in/out meters read $0.00 in the current publication while the legacy regional meter reads $0.010. Which one your traffic lands on depends on the service and how the traffic is routed, and the practical consequence is: **do not assert a cross-AZ rate from memory — read your own bill's usage types.** An architecture change justified by "$0.02/GB aggregate" against traffic that is actually metering on an xAZ meter at $0.00 will produce no saving and a lot of embarrassment. The queries in §1 are how you check.

Second, **cross-region is one-directional billing at $0.02/GB out and $0.00/GB in**, so a bidirectional replication stream between two regions costs $0.02/GB *each way* — $0.04/GB for a round trip — but a one-way dataset pull costs $0.02/GB total and *nothing* on the receiving side. The asymmetry matters when you decide which side initiates.

**NAT Gateway** (`AmazonEC2`, published **2026-08-17**):

| Meter | Price |
|---|---|
| `NatGateway-Hours` | **$0.045 / hour** |
| `NatGateway-Bytes` | **$0.045 / GB processed** |
| `NatGateway-Prvd-Gbps` | **$1.076 / Gbps-hour** (provisioned-bandwidth NAT) |
| `NatGateway-Prvd-Bytes` | **$0.00 / GB** with provisioned bandwidth |

**Elastic Load Balancing** (`AWSELB`, published **2026-07-20**):

| Meter | Price |
|---|---|
| ALB `LoadBalancerUsage` | **$0.0225 / hour** |
| ALB `LCUUsage` | **$0.008 / LCU-hour** |
| NLB `LoadBalancerUsage` | **$0.0225 / hour** |
| NLB `LCUUsage` | **$0.006 / LCU-hour** |
| Gateway LB | **$0.0125 / hour** + **$0.004 / LCU-hour** |
| Classic LB | **$0.025 / hour** + **$0.008 / GB** processed |

**VPC endpoints, Transit Gateway, public IPv4** (`AmazonVPC`, published **2026-07-24**):

| Meter | Price |
|---|---|
| `VpcEndpoint-Hours` (interface endpoint / PrivateLink) | **$0.01 / hour per endpoint** — and an endpoint is provisioned **per AZ**, so a 3-AZ endpoint is 3× this |
| `VpcEndpoint-Bytes` | **$0.01 / GB** up to 1 PB/mo, **$0.006 / GB** 1–5 PB, **$0.004 / GB** above 5 PB |
| `TransitGateway-Hours` | **$0.05 / hour per attachment** (VPC, VPN, Direct Connect, peering, Connect) |
| `TransitGateway-Bytes` | **$0.02 / GB processed** |
| `PublicIPv4:InUseAddress` | **$0.005 / hour** |
| `PublicIPv4:IdleAddress` | **$0.005 / hour** — an unattached Elastic IP costs the same as an attached one |
| Gateway VPC endpoint (S3, DynamoDB) | **no meter in the price list** — these are free of charge |

That last row is the single highest-leverage line in the table, and §4 works it out.

### 3. The GCP ladder, and where the two providers actually differ

From Google's published network pricing page (read directly):

| Hop | GCP price | Notes |
|---|---|---|
| Same zone, internal IPs, same VPC | **free** | |
| **Different zone, same region** | **$0.01 / GiB** | **charged to the *sending* project only** — this is the structural difference from AWS's both-ends metering |
| Inter-region, North America ↔ North America | **$0.02 / GiB** | |
| NA ↔ Europe | **$0.05 / GiB** | |
| NA ↔ Asia | **$0.08 / GiB** | |
| NA ↔ South America | **$0.14 / GiB** | the widest spread in the matrix |
| Internet egress, Premium Tier, to NA | first 1 GiB **free**, then **$0.12 / GiB** to 1 TiB, **$0.11 / GiB** to 10 TiB, **$0.08 / GiB** above | |
| Inbound | **free**, but processing resources (load balancers, Cloud NAT, protocol forwarding) still charge | |
| Cloud NAT | **$0.0014 / VM-hour** up to 32 VMs (**$0.044 / hour** flat above 32) + **$0.045 / GiB** processed (in *and* out) + **$0.005 / hour** per external IP | |

**The three differences worth carrying into a design review:**

1. **Who is billed.** GCP attributes inter-zone transfer to the sending project. AWS's classic regional meter fires on both ends. So the same replication topology produces different *per-team* invoices on the two providers even when the total is similar — which matters enormously when your chargeback model is per-team.
2. **Inter-region is distance-tiered on GCP and (in this publication) flat-ish on AWS.** GCP's NA↔SA rate is 7× its NA↔NA rate; AWS's us-east-1 outbound rate to Ireland, Oregon, and Tokyo was all $0.02/GB in the file above. If your topology is intercontinental, GCP's geography matters much more.
3. **NAT data processing is the same $0.045/GB on both.** This is not a coincidence so much as a convergence, and it is the number to memorise, because it is the one that quietly doubles the cost of "just pull it from the internet."

### 4. The NAT double-stack, and the free fix

```
   THE SAME 5 TB DATASET PULL, THREE PATHS, WITH THE METER ON EACH EDGE
   ═══════════════════════════════════════════════════════════════════════

   PATH A — private subnet → NAT Gateway → internet → S3 (the accident)
   ────────────────────────────────────────────────────────────────────
     GPU node                NAT Gateway               S3 (same region)
     10.0.4.17   ──────────▶  nat-0abc  ──────────▶  s3.us-east-1...
     private subnet          public subnet            AWS-owned endpoint

     METERS THAT FIRE, per GB:
       NatGateway-Bytes            $0.045   ← data processing
       DataTransfer-Out-Bytes      $0.000   ← *S3 in the same region is not
                                              internet egress*, so the egress
                                              meter does NOT fire here
       NatGateway-Hours            $0.045/h ← fixed, ×24×30 = $32.40/mo/AZ
     ─────────────────────────────────────
     EFFECTIVE                     $0.045/GB + $32.40/mo per NAT per AZ

     5 TB pull  = 5,000 GB × $0.045 = $225 PER RUN, for zero benefit.

   PATH B — private subnet → NAT Gateway → internet → a THIRD-PARTY host
   ────────────────────────────────────────────────────────────────────
       NatGateway-Bytes            $0.045
       DataTransfer-Out-Bytes      $0.090   ← now it IS internet egress
     ─────────────────────────────────────
     EFFECTIVE                     $0.135/GB  ← the "double stack"

     NOTE: only the OUTBOUND bytes hit the egress meter, but the NAT
     processes bytes in BOTH directions, so a download-heavy workload
     pays $0.045 on the (large) response and $0.135 on the (small)
     request. Model it per direction, not on the total.

   PATH C — private subnet → VPC GATEWAY ENDPOINT → S3 (the fix)
   ────────────────────────────────────────────────────────────────────
     GPU node ────────────────────────────────▶  S3
     10.0.4.17     route-table entry to a
                   prefix list; traffic never
                   leaves the AWS network

     METERS THAT FIRE, per GB:
       (none — gateway endpoints for S3 and DynamoDB have no meter
        in the price list; there is no hourly and no per-GB charge)
     ─────────────────────────────────────
     EFFECTIVE                     $0.00/GB

     5 TB pull  =  $0.

   THE CATCH, so you configure it correctly:
     • A GATEWAY endpoint (S3, DynamoDB only) is a ROUTE TABLE ENTRY. It is
       free, it is regional, and it does NOT work from on-prem or across a
       peering connection — only from within the VPC whose route table has it.
     • An INTERFACE endpoint (PrivateLink, everything else, and optionally
       S3) is an ENI in your subnet. It costs $0.01/hour PER AZ plus
       $0.01/GB. Highly available across 3 AZs = $0.03/hour = $21.60/month
       before any traffic. Cheaper than NAT per GB (4.5× cheaper) but NOT free.
     • Adding a gateway endpoint does not remove the NAT Gateway's hourly
       charge; it removes its per-GB charge for that destination only.
```

**How to find whether you have this problem, in one query.** In the Cost and Usage Report, `SELECT sum(lineItem/UnblendedCost) WHERE lineItem/UsageType LIKE '%NatGateway-Bytes%'` grouped by month. If that number is material and your workload talks mostly to S3 or DynamoDB, the fix is a route-table entry and it takes ten minutes.

### 5. Latency, derived rather than quoted

You do not need a provider's latency claim. Light in single-mode fibre travels at roughly `c / 1.47 ≈ 2.04 × 10⁸ m/s`, which is a very usable constant:

```
   ~204 km per millisecond, one way   →   ~102 km per millisecond, ROUND TRIP
```

Everything follows:

| Hop | Physical separation | Propagation RTT | Realistic observed RTT | The gap is |
|---|---|---|---|---|
| Same rack | ~10 m | ~0.0001 ms | 0.05–0.2 ms | switch + NIC + kernel (lesson 01) |
| Same AZ, different building | ~1–5 km | 0.02–0.05 ms | 0.1–0.5 ms | two or three switch hops |
| **Cross-AZ** | AZs are separated by kilometres to tens of kilometres of fibre | **0.1–1 ms** | **0.3–1.5 ms** | fibre routes are not straight lines; add ~30–50 % over great-circle |
| us-east-1 ↔ us-west-2 | ~3,900 km great circle | ~38 ms | 60–70 ms | fibre routing detours |
| us-east-1 ↔ eu-west-1 | ~5,100 km great circle | ~50 ms | 65–80 ms | transatlantic cable landing points |
| us-east-1 ↔ ap-northeast-1 | ~10,900 km great circle | ~107 ms | 140–160 ms | multiple cable segments |

**The rule to carry:** actual RTT is typically **1.3–1.6×** the great-circle propagation RTT, because fibre follows rights-of-way and cable landings, not geodesics. If someone quotes a cross-region latency far *below* the great-circle figure, they are measuring something else (an edge PoP, a cached response). If far *above*, there is a routing problem worth investigating.

**Measure your own AZ separation** rather than trusting a range:

```bash
# From two instances in different AZs of the same region, same VPC:
$ ping -c 1000 -i 0.01 10.0.5.42 | tail -2
1000 packets transmitted, 1000 received, 0% packet loss, time 10231ms
rtt min/avg/max/mdev = 0.412/0.487/2.104/0.061 ms
#         ^^^^^ min is the one that matters: it is the propagation +
#               forwarding floor with no queueing. 0.412 ms RTT implies
#               ~42 km of one-way fibre, which is a real fact about your
#               region's physical layout.
```

### 6. Cross-zone load balancing: the default that writes the invoice

Lesson 03's LB tier acquires a cost dimension the moment it spans zones, and the defaults differ by product in a way that catches people during migrations:

- **ALB** performs cross-zone load balancing **always** — it is not disableable at the load-balancer level. A request arriving in `us-east-1a` can be sent to a target in `us-east-1b`, and that leg is cross-AZ transfer.
- **NLB** has cross-zone load balancing **off by default**, configurable **per target group**. Traffic stays in the AZ it arrived in unless you turn it on.

The consequences of that asymmetry, both directions:

```
   MIGRATING ALB → NLB WITHOUT CHECKING
   ─────────────────────────────────────
   Before: even load across all AZs' targets, cross-AZ transfer on the bill.
   After:  each AZ's targets serve only that AZ's arriving traffic.
           If your DNS/anycast distribution is uneven, or if one AZ has
           fewer healthy targets, that AZ's targets are now overloaded
           while another AZ's sit idle. The transfer line item drops and
           a latency/error regression appears that nobody connects to it.

   MIGRATING NLB → ALB WITHOUT CHECKING
   ─────────────────────────────────────
   Before: in-zone only, no cross-AZ transfer.
   After:  even load AND a new cross-AZ transfer line item, roughly
           (N_AZ − 1)/N_AZ of all LB→target bytes. For 3 AZs that is
           2/3 of your traffic newly crossing a metered boundary.
```

**Worked, for a service moving 40 TB/month through its LB across 3 AZs**, assuming the legacy `DataTransfer-Regional-Bytes` meter at $0.010/GB firing on both ends:

```
   Bytes that cross an AZ with cross-zone ON  = 40,000 GB × (2/3) = 26,667 GB
   Cost, both ends metered                    = 26,667 × $0.010 × 2 = $533/month
   Cost with cross-zone OFF                   = $0
   Cost of the imbalance you accept instead   = you must now provision each
                                                AZ for its own peak rather
                                                than the fleet's average.
                                                At 3 AZs and a 20 % skew that
                                                is roughly 20 % more capacity.

   THE COMPARISON THAT MAKES IT A DECISION: $533/month of transfer against
   20 % of the target fleet's compute cost. If the fleet is 30 × m6i.4xlarge
   (~$0.768/hr each on demand → ~$16,600/month), 20 % is ~$3,300/month.
   Cross-zone ON is the cheaper answer here BY A FACTOR OF SIX, and the
   reflexive "turn off cross-zone to save money" would have been wrong.
   Run the arithmetic; do not run the reflex.
```

### 7. Transit topologies, as cost formulas

Pick by traffic shape and trust boundary, and be able to state the formula:

| Topology | Fixed cost | Per-GB cost | Transitive? | Management cost | Fits |
|---|---|---|---|---|---|
| **VPC peering** | $0 | $0 for peering itself — you still pay the underlying cross-AZ/cross-region transfer | **No** | O(n²) connections, and route tables in every VPC | a handful of VPCs with heavy traffic between them |
| **Transit Gateway** | **$0.05/attachment-hour** = $36/month per attachment | **$0.02/GB** processed | Yes | O(n) attachments, one route table | many VPCs, moderate traffic, you want central routing/inspection |
| **PrivateLink (interface endpoint)** | **$0.01/endpoint-hour per AZ** = $7.20/month/AZ | **$0.01/GB** (falling to $0.006 above 1 PB, $0.004 above 5 PB) | **No — deliberately** | one endpoint per consumed service | exposing *one service* across a trust boundary (SaaS, another org) |
| **Gateway endpoint (S3/DynamoDB)** | $0 | $0 | No | a route-table entry | S3/DynamoDB from inside one VPC |

**The crossover arithmetic between peering and TGW**, which is the question people actually ask:

```
   For a pair of VPCs exchanging V GB/month in the same region:

     peering  = (underlying cross-AZ transfer, if any)
     TGW      = 2 attachments × $36/month + V × $0.02
                                            ^^^^^^^^^ TGW charges per GB
                                            processed, and traffic between
                                            two attached VPCs is processed
                                            ONCE per direction of travel
                                            through the gateway

     At V = 10,000 GB/month:  TGW = $72 + $200 = $272 vs peering ≈ $0–$200
     At V = 100,000 GB/month: TGW = $72 + $2,000 = $2,072
     At V = 1,000,000 GB/month: TGW = $72 + $20,000 = $20,072

   → TGW's per-GB fee dominates fast. The rule that falls out: use TGW for
     CONNECTIVITY BREADTH (many VPCs, modest traffic each) and peering for
     TRAFFIC VOLUME (few VPCs, heavy traffic). A hub-and-spoke design that
     routes a data-heavy pair through the hub is paying $0.02/GB for
     routing elegance.
```

**PrivateLink's non-transitivity is a feature, not a gap.** A peering connection or a TGW attachment creates general IP reachability, which means your security model has to re-establish the boundary with security groups and NACLs. A PrivateLink endpoint exposes exactly one service, in one direction, with no route to anything else — the network *itself* enforces the boundary. That is why it is the standard for consuming a third-party SaaS from inside your VPC and for exposing your service to a customer's VPC, and why the per-AZ hourly charge is worth paying in those cases and wasteful in others.

### 8. Hybrid: the tunnel, and the PMTU black hole

Direct Connect gives you a dedicated circuit — predictable bandwidth, predictable latency, and a lower egress rate than the internet. A Site-to-Site VPN gives you IPsec over the public internet — cheap, fast to stand up, internet-grade variance. Both share one recurring silent failure, and it is not about which one you chose.

**The mechanism.** Any encapsulation shrinks the payload that fits in a link's MTU. IPsec ESP in tunnel mode over IPv4 adds, per packet:

```
   outer IP header                    20 bytes
   ESP header (SPI + seq)              8 bytes
   IV                               8–16 bytes  (cipher-dependent)
   padding + pad length + next hdr   2–17 bytes
   ICV (integrity check value)      12–16 bytes
                                    ──────────
   TOTAL OVERHEAD                   ~50–77 bytes, cipher-dependent

   On a 1500-byte path MTU:
     usable inner MTU  = 1500 − overhead ≈ 1423–1450
     usable TCP MSS    = inner MTU − 20 (IP) − 20 (TCP) ≈ 1383–1410

   Do NOT copy those numbers. Compute yours, or discover it (below).
```

**Why it manifests as "small things work, big things hang."** A TCP sender that negotiated MSS 1460 emits 1500-byte packets with the Don't Fragment bit set (Linux sets DF by default). At the tunnel ingress the packet does not fit; the router is supposed to drop it and return an ICMP **Type 3 Code 4** ("fragmentation needed and DF set") carrying the next-hop MTU, which Path MTU Discovery uses to shrink the connection's MSS. **If any device on the return path filters ICMP** — a habit that persists from 1990s firewall advice — the sender never learns, keeps retransmitting the same oversized segment, and the connection hangs after the handshake.

```
   PMTU BLACK HOLE — WHY THE HANDSHAKE SUCCEEDS AND THE TRANSFER DIES
   ═══════════════════════════════════════════════════════════════════

   sender                    tunnel ingress (MTU 1436)           receiver
   ──────                    ─────────────────────────           ────────
   t0  SYN (40 B)      ──────────── fits ──────────────────────▶
   t0+ SYN-ACK, MSS 1460 ◀──────────────────────────────────────
   t1  ACK             ──────────── fits ──────────────────────▶
       ✔ HANDSHAKE SUCCEEDS. `ping` succeeds. `telnet host 443` succeeds.
       Every "is it up?" check passes. This is the trap.

   t2  first data segment, 1500 B, DF=1
                       ────────▶ ✗ 1500 > 1436, DF set
                                 router SHOULD send:
                                 ICMP type 3 code 4, next-hop MTU 1436
                                     │
                                     ▼
                             ┌───────────────┐
                             │ firewall that │  "we block ICMP,
                             │ drops ICMP    │   it's a security
                             └───────────────┘   best practice"
                                     ✗ ICMP never arrives

   t2+RTO   sender retransmits the SAME 1500 B segment (it has no reason
            to believe anything is wrong with its size)
   t2+2RTO  again. t2+4RTO: again. Exponential backoff, TCP_RTO_MAX 120 s,
            tcp_retries2 = 15 → the connection dies after ~15 minutes
            having transferred ZERO bytes of payload.

   SYMPTOM AS REPORTED:  "SSH connects then freezes." "curl gets the
   headers and hangs on the body." "small files work, big files don't."
   "It works from the bastion but not from the app."
```

**Confirming it in one command.** Binary-search the path MTU with a DF-set ping:

```bash
# -M do = set DF; -s is the ICMP PAYLOAD size, so add 28 (20 IP + 8 ICMP)
$ ping -M do -s 1472 -c 1 10.100.0.5      # 1472 + 28 = 1500
PING 10.100.0.5 (10.100.0.5) 1472(1500) bytes of data.
--- 10.100.0.5 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss    ← 1500 does not fit

$ ping -M do -s 1408 -c 1 10.100.0.5      # 1408 + 28 = 1436
PING 10.100.0.5 (10.100.0.5) 1408(1436) bytes of data.
1416 bytes from 10.100.0.5: icmp_seq=1 ttl=62 time=8.71 ms   ← 1436 fits
#  → PATH MTU IS 1436. Note there was no "frag needed" message in the
#    failing case; a healthy path would have returned one immediately.
#    Its absence IS the diagnosis.
```

**The three fixes, in order of preference:**

1. **MSS clamping on the tunnel interface** — rewrite the MSS option in passing SYN packets so both endpoints negotiate a size that fits. On Linux: `iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu` (or `--set-mss 1383` for a fixed value). This is the standard fix because it works even when ICMP is blocked and requires no change on the endpoints.
2. **Stop filtering ICMP type 3 code 4.** This is the correct fix and the one you will lose the argument about. It is worth making the argument anyway: blocking it breaks a mandatory part of IPv4 and *all* of IPv6, where routers do not fragment at all and PMTUD is the only mechanism.
3. **Lower the interface MTU** on the instances behind the tunnel. Blunt, effective, and it costs you throughput on every path, not just the tunnel.

Linux also offers a defensive fallback: `net.ipv4.tcp_mtu_probing = 1` enables Packetization-Layer PMTUD (RFC 4821) **only when an ICMP black hole is detected**, letting TCP probe for the working size in-band rather than relying on ICMP at all. Setting it to `2` enables it always, starting from `tcp_base_mss`. It is a good default on any fleet that talks over tunnels.

**Direct Connect does not exempt you.** A dedicated circuit removes the *variance*, not the encapsulation: VLAN tagging, MACsec, and any overlay you run over the circuit each consume header bytes. The fix is about the encapsulation, not about which physical medium carries it.

### 9. The AZ-placement decision, in one unit

The reflexive answers — "always spread for resilience," "always same-AZ to save money" — are both wrong because they compare things in different units. Put both sides in dollars per month:

```
   COST OF SPREADING = cross-AZ transfer + latency-induced compute
   ───────────────────────────────────────────────────────────────
     transfer      = (bytes crossing AZs per month) × (rate) × (1 or 2,
                     depending on whether your meter bills both ends)
     latency cost  = (extra RTT per synchronous operation)
                     × (operations per second) × (seconds per month)
                     × (cost per second of the blocked resource)

   COST OF NOT SPREADING = expected loss from a zone failure
   ───────────────────────────────────────────────────────────────
     = P(zone failure in a month) × (duration) × (cost per hour of outage)

     where cost per hour of outage = lost revenue + SLA credits + the
     fully-loaded cost of the people who will be in the incident channel.

   WORKED — a stateful service, 3-AZ vs 1-AZ
   ───────────────────────────────────────────────────────────────
     Replication traffic: 8 TB/month written, replicated to 2 peers
       = 16,000 GB crossing AZs
       × $0.010/GB × 2 ends            = $320/month
     Synchronous replication adds 0.6 ms to every write:
       50M writes/month × 0.6 ms       = 30,000 seconds of added
                                         blocked-writer time/month
       (whether that costs anything depends on whether the writers
        are the bottleneck — if they are async, it is $0)

     Against: assume one zone-impacting event per 2 years, 3 hours,
     and $40,000/hour of business impact.
       expected monthly cost of 1-AZ = (1/24 months) × 3 h × $40,000
                                     = $5,000/month

     $5,000 ≫ $320. SPREAD. And now you can say WHY in a review,
     with the arithmetic on the slide, instead of citing best practice.

     FLIP THE INPUTS and the answer flips: a batch pipeline whose
     3-hour outage costs $0 (it just runs later) has an expected
     cost of not spreading of ~$0, and the $320 is pure waste.
```

**The point is not the numbers; it is that both sides are now the same kind of thing.** A design review that contains this table converges. One that contains "we should spread across AZs for resilience" does not.

### 10. Training-specific compounding: the same decision, billed twice

For a synchronous data-parallel training job, cross-AZ placement is not just a transfer line item. Every collective is a **barrier**, so the added RTT lands on the critical path of every step:

```
   THE ARITHMETIC
   ─────────────────────────────────────────────────────────────────
     added latency per collective ≈ cross-AZ RTT ≈ 0.6 ms (measured, §5)
     ring all-reduce over M nodes performs 2(M−1) sequential steps
       → added latency ≈ 2(M−1) × 0.6 ms

     For M = 8 nodes:      2 × 7 × 0.6 ms  =  8.4 ms per all-reduce
     Training steps/month at 1 step/s:      2.6M steps
     Added wall-clock:     8.4 ms × 2.6M   =  21,840 s ≈ 6.1 hours/month

     8 nodes × 8 GPUs, at ~$30–40/hour per node (verify your rate — these
     move; use your own contract price):
       8 nodes × $35/h × 6.1 h  ≈  $1,700/month of pure synchronisation
                                    overhead — GPUs idle at a barrier.

     COMPARE with the transfer bill for the same topology:
       gradient bytes are NOT what crosses AZs here if all 8 nodes are
       in one AZ — the compounding only appears when the JOB spans AZs.

   THE ASYMMETRY THAT DRIVES THE DESIGN
   ─────────────────────────────────────────────────────────────────
     A latency tax on a barrier is paid EVERY STEP, forever.
     A transfer bill is paid PER BYTE MOVED, once.
     So for training: co-locate the job in ONE AZ (accepting that a zone
     loss kills the job — which is fine, because a job restarts from a
     checkpoint), and spread the CHECKPOINT STORE for durability.
     That is the opposite of the answer for a stateful service, and it
     is why "spread across AZs" cannot be a blanket policy.
```

This is also why hyperscale GPU clusters are built as **one contiguous fabric in one physical location** (module 09's rail-optimised Clos), not as a zone-redundant deployment. The cloud-networking abstraction of "just spread it across AZs" is a poor fit for barrier-synchronous workloads, and knowing exactly why — the barrier, not the bandwidth — is the staff-level answer.

## Perspectives

**FinOps / accounting.** The first question is never "how do we reduce this," it is "which meter fired, on whose invoice line, generated by which resource." AWS's current price list carries *two* families of cross-AZ meters at different rates; GCP attributes inter-zone transfer to the sending project only. Proposing an architecture change before confirming the specific `usagetype` is how teams spend a quarter on a migration that moves the bill by zero.

**Topology design.** The AZ-spread decision is made at architecture-review time, before a byte moves, and it belongs in the design doc with the arithmetic from §9 attached. Discovering it from a Cost Explorer dashboard three months later means you are now paying migration cost on top of transfer cost. The reviewable artefact is a per-hop table: hop, meter, `$/GB`, direction billed, RTT, snapshot date.

**Failure-domain-vs-cost tension.** Both sides go in dollars per month: expected loss from a zone failure (probability × duration × business cost per hour) against transfer plus latency-induced compute. When both sides are the same unit the decision stops being a reflex and becomes a defensible tradeoff — and, importantly, it *flips* depending on the workload, which is the whole point.

**AI/ML-specific.** For inference, cross-AZ or cross-region placement is a per-request latency and cost question with a bounded answer. For training it compounds: every collective is a barrier, so the added RTT multiplies by the number of steps, and the bill arrives as GPU-hours rather than as transfer dollars. That is why training jobs are co-located in a single AZ (or a single fabric) and durability is bought at the checkpoint layer instead — the exact inverse of the stateful-service answer.

## Real-world use cases

- **The AWS Price List Bulk API as the primary source.** Every AWS figure in this lesson was read from `pricing.us-east-1.amazonaws.com` for `us-east-1`, with publication dates 2026-07-20 (`AWSDataTransfer`, `AWSELB`), 2026-07-24 (`AmazonVPC`), and 2026-08-17 (`AmazonEC2`, for NAT Gateway). What it shows beyond the numbers: the price list is the *only* place where you can see which `usagetype` strings exist, which is what lets you match a line on your bill to a mechanism. It also surfaced a fact the pricing marketing pages do not emphasise — that cross-AZ transfer currently appears under both a `DataTransfer-Regional-Bytes` meter at $0.010/GB and explicit `DataTransfer-xAZ-Out/In-Bytes` meters at $0.00/GB.
- **Google's published network pricing page**, read directly. What it shows: inter-zone transfer at **$0.01/GiB charged to the sending project only** — a structurally different billing model from AWS's both-ends metering, which changes per-team chargeback even when the totals are comparable; a distance-tiered inter-region matrix ranging from $0.02/GiB (NA↔NA) to $0.14/GiB (anything↔South America); and Cloud NAT at **$0.045/GiB processed**, the same figure as AWS NAT Gateway.
- **The provisioned-bandwidth NAT Gateway meters.** The current EC2 price list carries `NatGateway-Prvd-Gbps` at **$1.076/Gbps-hour** with `NatGateway-Prvd-Bytes` at **$0.00/GB**. What it shows: for very high, sustained NAT throughput the pricing model inverts from per-GB to per-Gbps, and there is a crossover. At $0.045/GB, one Gbps sustained for an hour moves `1e9/8 × 3600 = 450 GB` = $20.25 of processing, against $1.076 for the provisioned equivalent — so for *sustained* high-rate egress the provisioned model is dramatically cheaper, and for bursty low-duty-cycle traffic it is not. Work out your duty cycle before choosing.

## Worked example

**Scenario.** Cost-model a 64-GPU training job — 8 nodes × 8 H100 — and show that data movement is a first-class line item rather than a rounding error. Then optimise it and show the arithmetic for each change.

**Given (state every assumption):**

```
   Compute:      8 nodes × 8 GPUs, running continuously for a month (720 h)
                 node price: use YOUR contract rate. For this model,
                 $35/node-hour on demand (verify — GPU pricing moves fast)
   Checkpoints:  400 GB per checkpoint, written every 30 min = 48/day
   Dataset:      5 TB, pulled once per run from object storage
   Region:       us-east-1
   Rates:        AWS Price List, us-east-1, publication dates as in §2
```

**Step 1 — the compute baseline, so everything else has a denominator.**

```
   8 nodes × $35/node-hour × 720 hours = $201,600 / month
```

**Step 2 — price the naive topology.** Checkpoints replicated cross-AZ for durability; dataset pulled from another region through a NAT Gateway.

```
   (a) CHECKPOINT REPLICATION, cross-AZ
       400 GB × 48/day × 30 days                 = 576,000 GB/month
       × $0.010/GB (DataTransfer-Regional-Bytes)
       × 2 (metered on both ends)                = $11,520 / month
                                                   = 5.7 % of compute

   (b) DATASET PULL, cross-region, through NAT
       5,000 GB × $0.020/GB (InterRegion Outbound, billed to the
                             SENDING region — so this appears on the
                             OTHER account's bill if the bucket is
                             owned elsewhere. Check who pays.)   = $100
       5,000 GB × $0.045/GB (NatGateway-Bytes, on the pulling side) = $225
       NAT Gateway hours, 3 AZs × $0.045 × 720                     = $97
                                                       subtotal    = $422

   (c) INTERNET EGRESS for logs/metrics/artifacts, say 2 TB/month
       first 10 TB tier: 2,000 GB × $0.090                      = $180

   (d) NAT processing for pip/apt/model-hub pulls, say 500 GB/month
       500 × ($0.045 NAT + $0.090 egress)                       = $67.50

   ─────────────────────────────────────────────────────────────────
   NAIVE DATA-MOVEMENT TOTAL                          ≈ $12,190 / month
                                                      = 6.0 % of compute
```

**Step 3 — optimise, one change at a time, with the mechanism named.**

```
   FIX 1 — checkpoint to same-AZ storage; replicate ASYNCHRONOUSLY
           and less often for durability.
     Same-AZ writes to S3 in-region via a GATEWAY ENDPOINT: no transfer
     meter fires at all. Keep a cross-AZ copy of every 6th checkpoint
     (4-hourly) instead of every one, for disaster recovery.
       576,000 GB → 96,000 GB crossing AZs
       96,000 × $0.010 × 2                              = $1,920/month
     SAVING: $9,600/month.
     RISK ACCEPTED, stated: up to 4 hours of training progress lost if
     the AZ is destroyed. At 1 step/s that is 14,400 steps — which is
     a business decision, not a networking one, and belongs in the doc.

   FIX 2 — VPC gateway endpoint for S3, and move the dataset in-region.
     Dataset pulled from an in-region bucket over a gateway endpoint:
       inter-region $0.020/GB → $0                      = −$100
       NAT processing $0.045/GB → $0                    = −$225
     Package pulls also route via endpoints where possible; the rest
     still needs NAT, but the volume is small.
       (d) 500 GB → 150 GB × $0.135                     = $20 (from $67.50)
     SAVING: ≈ $372/month, and the NAT Gateway's hourly charge remains
     ($97/month) because you still need egress for genuine internet
     destinations. A gateway endpoint removes the per-GB meter for its
     destinations only.

   FIX 3 — compress checkpoints.
     Model checkpoints are dense float tensors and compress poorly with
     gzip, but bf16 → fp8 or sharded/deduplicated checkpointing commonly
     halves the bytes. Assume 2× on the retained cross-AZ copies:
       $1,920 → $960                                    = −$960/month
     MEASURE this before claiming it — compression ratio is model- and
     format-dependent, and CPU spent compressing is CPU not feeding GPUs
     (lesson 01 §13).

   ─────────────────────────────────────────────────────────────────
   OPTIMISED DATA-MOVEMENT TOTAL                      ≈ $1,257 / month
                                                      = 0.6 % of compute
   TOTAL SAVING                                       ≈ $10,930 / month
                                                      ≈ $131,000 / year
```

**Step 4 — now the placement question, which is not a transfer question.**

```
   Someone proposes spreading the 8 nodes across 3 AZs "for resilience."
   Price BOTH effects:

   (a) Transfer: the all-reduce moves, per step, roughly
       2 × (M−1)/M × S bytes per node, S = gradient size.
       For a 7B-parameter model in bf16, S ≈ 14 GB per node per step.
         8 nodes × 2 × (7/8) × 14 GB = 196 GB per step (!!)
       At 1 step/s that is 196 GB/s of collective traffic. If ANY of it
       crosses an AZ boundary at $0.010/GB × 2:
         196 GB/s × 86,400 s/day × 30 = 508,000,000 GB/month
         × $0.020                     = $10,160,000 / month
       This number is absurd, and that is the POINT: collective traffic
       must never cross a metered boundary. It is why training clusters
       use a dedicated fabric (module 09) and not VPC networking.

   (b) Latency: even if the transfer were free, §10's arithmetic applies —
       2(M−1) × 0.6 ms = 8.4 ms added per all-reduce, ≈ 6.1 hours/month
       of barrier time ≈ $1,700/month of idle GPUs.

   THE ANSWER: keep the job in one AZ (better: one placement group on one
   fabric). Buy durability at the CHECKPOINT layer, where the bytes are
   thousands of times smaller and the write is asynchronous. Resilience
   for a training job is "restart from a checkpoint," not "survive a zone
   loss mid-step" — a barrier-synchronous job cannot survive losing a
   node anyway, AZ or no AZ.
```

**Step 5 — the deliverable table.** One row per edge, with the meter named:

| Hop | Meter (`usagetype`) | $/GB | Direction billed | RTT | Notes |
|---|---|---|---|---|---|
| GPU ↔ GPU intra-node | none | $0 | — | ~1–2 µs | NVLink, 450 GB/s/dir on H100 — see module 09 |
| node ↔ node, same AZ, RDMA fabric | none | $0 | — | ~5–10 µs | dedicated fabric, not VPC |
| node ↔ node, cross-AZ | `DataTransfer-Regional-Bytes` | $0.010 | both ends | ~0.4–1.5 ms | **never put collectives here** |
| node → S3 same region, gateway endpoint | none | **$0** | — | ~1–5 ms | the fix |
| node → S3 same region, via NAT | `NatGateway-Bytes` | $0.045 | sender | ~1–5 ms | the accident |
| node → internet, via NAT | `NatGateway-Bytes` + `DataTransfer-Out-Bytes` | $0.135 | sender | varies | tiered above 10 TB |
| region → region | `USE1-EU-AWS-Out-Bytes` | $0.020 | sending region only | 65–80 ms (us-east-1↔eu-west-1) | inbound free |
| NAT Gateway existence | `NatGateway-Hours` | $32.40/mo each | — | — | per AZ |
| PrivateLink endpoint existence | `VpcEndpoint-Hours` | $7.20/mo each | — | — | per AZ |

*(Snapshot: AWS Price List, us-east-1, July–August 2026 publications. Re-run the §1 queries before quoting these in a real review.)*

## Practice
<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

**Task A — build the per-hop table from primary sources.** Using the `curl | jq` recipe in §1, pull the current `AWSDataTransfer`, `AmazonVPC`, and `AWSELB` price files **for your own region** and generate the table in §5's format automatically. Include the `publicationDate` in the output. The point is a script you can re-run, not a table you typed.

**Task B — the 64-GPU transfer cost model.** Build a spreadsheet or script with these parameters: node count, GPUs per node, node hourly rate, checkpoint size, checkpoint cadence, checkpoint retention topology (same-AZ / cross-AZ / cross-region), dataset size, dataset source (in-region bucket via gateway endpoint / in-region via NAT / cross-region), and monthly internet egress. Output: monthly cost per line item, the total, and the total as a percentage of compute. Include a toggle for **NAT Gateway vs VPC gateway endpoint** on the dataset leg so that the double-stack appears as its own line rather than being folded into "egress."

**Task C — measure your own AZ separation and derive the distance.** From two instances in different AZs, run `ping -c 1000 -i 0.01` and record the **minimum** RTT. Divide by 2 and multiply by 204 km/ms to get the implied one-way fibre distance. Repeat for a same-AZ pair and for a cross-region pair, and compare each against the great-circle distance. Report the ratio of observed to great-circle propagation; you should land near 1.3–1.6×.

**Task D — reproduce a PMTU black hole and fix it.** On any two hosts with a tunnel or an overlay between them (a GRE tunnel or a VXLAN interface is sufficient — no cloud account required):

1. Set the tunnel MTU below 1500 and confirm the path MTU with `ping -M do -s <n>` binary search.
2. Add a firewall rule dropping ICMP type 3 code 4 on the return path.
3. Show that the TCP handshake succeeds and a large transfer hangs — capture the repeated retransmission of the same oversized segment with `tcpdump`.
4. Fix it with `iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu` and show the negotiated MSS change in the SYN.
5. Separately, demonstrate `net.ipv4.tcp_mtu_probing=1` recovering the connection *without* the clamp, and note how long it takes to kick in.

**Acceptance criteria**

- [ ] A generated (not hand-typed) per-hop cost table for your region with meter names and a publication date.
- [ ] A parameterised transfer-cost model with naive vs optimised totals and a NAT/gateway-endpoint toggle showing the double-stack as its own line.
- [ ] Measured min-RTT for same-AZ, cross-AZ, and cross-region pairs, with the implied fibre distance and the observed/great-circle ratio.
- [ ] A reproduced PMTU black hole with a packet capture of the repeated oversized retransmission, plus the MSS-clamp fix demonstrated and the `tcp_mtu_probing` alternative timed.
- [ ] A written AZ-placement decision for one real workload, with both sides in dollars per month, including the training case where the answer is the opposite of the stateful-service case.

## Common pitfalls
- **Quoting a transfer rate from memory.** The current AWS price list carries cross-AZ traffic under *two* meter families at different prices ($0.010/GB on `DataTransfer-Regional-Bytes`, $0.00/GB on the explicit `xAZ-Out/In` meters). Read your own bill's usage types before proposing a fix, or you will optimise a meter that is not firing.
- **"Egress pricing is flat."** It is tiered: from us-east-1, $0.090/GB for the first 10 TB/month, $0.085 for the next 40 TB, $0.070 for the next 100 TB, $0.050 above 150 TB. Back-of-envelope math at $0.09 overestimates a high-volume workload by up to 44 %.
- **"VPC peering has no cost."** There is no per-GB *peering* fee, but the underlying cross-AZ or cross-region transfer that rides over it is billed at the normal rate. Peering removes a management cost and a per-GB *gateway* fee, not a transfer cost.
- **"Transit Gateway is the modern way, so use it everywhere."** TGW charges **$0.02/GB processed** on top of $0.05/attachment-hour. A data-heavy VPC pair routed through the hub for architectural tidiness pays $20,000/month at 1 PB. Use TGW for connectivity breadth; use peering for traffic volume.
- **"PrivateLink is transitive like peering."** It is deliberately one-way and service-scoped: a consumer reaches exactly the exposed service and nothing else. That is the security property you are paying $0.01/hour per AZ for.
- **Forgetting that an idle Elastic IP costs the same as an in-use one.** Both `PublicIPv4:InUseAddress` and `PublicIPv4:IdleAddress` are $0.005/hour. A hundred forgotten EIPs is $360/month for nothing.
- **"Same-AZ is always cheaper and therefore right."** It collapses the failure domain. State both the dollars saved and the blast radius accepted, and let the workload's failure budget decide — §9 shows the answer flipping on realistic inputs.
- **"Direct Connect eliminates the MTU problem."** A dedicated circuit removes path *variance*, not encapsulation overhead. VLAN tags, MACsec, and any overlay still consume header bytes, and a filtered ICMP type 3 code 4 anywhere on the path still produces a black hole. The fix is MSS clamping, regardless of the medium.
- **Turning off cross-zone load balancing to save money without pricing the imbalance.** §6's worked case has the transfer saving at $533/month against $3,300/month of extra capacity needed to absorb the resulting per-AZ skew. The reflex costs six times what it saves.

## Self-check

**1. A cross-AZ transfer is quoted at "$0.01/GB" but your bill shows roughly double per replicated gigabyte. Why — and what should you check before acting on that?**

Because the classic AWS cross-AZ meter, `DataTransfer-Regional-Bytes` at $0.010/GB, is described as "regional data transfer — in/out/between EC2 AZs," and it fires on **both** the sending and the receiving side: one logical GB crossing zones produces two metered GB, ≈$0.02 aggregate. Replicated stateful systems copy every write across zones, so their replication traffic pays this on every byte, which is why they dominate so many bills. **What to check first:** the current price list also carries explicit `DataTransfer-xAZ-Out-Bytes` and `DataTransfer-xAZ-In-Bytes` meters priced at $0.00/GB in the same publication, so which meter your traffic lands on is not something to assume. Pull the usage types from the Cost and Usage Report and confirm which one is generating the charge before designing a fix. For contrast, GCP's inter-zone transfer is $0.01/GiB charged to the **sending project only** — a different billing model for the same physical hop.

**2. Your private-subnet training nodes pull a 5 TB dataset from S3 through a NAT Gateway every run. What is the effective $/GB, and what single change removes it?**

Two meters fire on that path: `NatGateway-Bytes` at **$0.045/GB** for data processing, plus `NatGateway-Hours` at $0.045/hour (≈$32.40/month per NAT, per AZ). If the destination is S3 *in the same region*, the internet-egress meter does **not** fire — so the effective rate is $0.045/GB, or $225 per 5 TB run. If the destination were a third-party internet host, `DataTransfer-Out-Bytes` at $0.090/GB would stack on top for a combined **$0.135/GB**. The fix is a **VPC gateway endpoint** for S3: it is a route-table entry, it has no meter in the price list at all (no hourly, no per-GB), and it keeps the traffic on the AWS network. Two caveats: a gateway endpoint works only for S3 and DynamoDB and only from within the VPC whose route table carries it (not from on-prem or across peering); and it removes the NAT's *per-GB* charge for those destinations only — the hourly charge remains for genuine internet traffic. For other AWS services you would use an **interface** endpoint at $0.01/hour per AZ plus $0.01/GB, which is 4.5× cheaper than NAT per GB but not free.

**3. `ping` between two sites over a Direct Connect or VPN tunnel succeeds, but large transfers hang. What is happening, how do you confirm it, and what are the fixes in order?**

A **PMTU black hole**. Tunnel encapsulation lowers the usable MTU — IPsec ESP in tunnel mode over IPv4 adds roughly 50–77 bytes depending on cipher, so a 1500-byte path carries an inner MTU near 1423–1450 and a usable MSS near 1383–1410. A sender that negotiated MSS 1460 emits 1500-byte packets with DF set; the tunnel ingress must drop them and return **ICMP type 3 code 4** with the next-hop MTU, and Path MTU Discovery shrinks the connection. If any device on the return path filters ICMP, that message never arrives, the sender retransmits the same oversized segment forever, and the connection dies after `tcp_retries2` backoff (~15 minutes) having moved zero payload bytes. Small packets — the handshake, `ping`, `telnet host 443` — all fit, which is why every liveness check passes. **Confirm** by binary-searching with `ping -M do -s N` (remember `-s` is the ICMP payload, so add 28 for the IP+ICMP headers): the largest size that succeeds is your path MTU, and the *absence* of a "frag needed" response on the failing size is the diagnosis. **Fixes, in order:** (1) MSS clamping on the tunnel interface (`iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu`) — works even with ICMP blocked and needs no endpoint changes; (2) stop filtering ICMP type 3 code 4, which is the correct fix and mandatory for IPv6 where routers never fragment; (3) lower the endpoints' interface MTU, which is blunt and costs throughput on every path. Also set `net.ipv4.tcp_mtu_probing=1` fleet-wide as a defensive fallback — it enables RFC 4821 packetization-layer PMTUD when a black hole is detected. Direct Connect does **not** exempt you: the circuit removes variance, not encapsulation.

**4. For a 64-GPU training job, why can data movement be a material fraction of the bill, and why is "spread the job across AZs for resilience" the wrong answer?**

Training artefacts are large and repetitive: 400 GB checkpoints written every 30 minutes and replicated cross-AZ is 576,000 GB/month, which at $0.010/GB metered on both ends is ~$11,500/month — about 6 % of a $200k/month compute bill, entirely avoidable by writing to same-AZ storage through a free gateway endpoint and replicating a subset asynchronously. On spreading the **job**: two independent reasons say no. First, collective traffic is enormous — a ring all-reduce moves roughly `2(M−1)/M × S` bytes per node per step, which for an 8-node job on a 7B bf16 model is hundreds of GB *per step*; putting any of that across a metered AZ boundary produces an absurd bill, which is precisely why training clusters use a dedicated fabric rather than VPC networking. Second, a collective is a **barrier**, so cross-AZ RTT (~0.6 ms measured) lands on the critical path of every step: `2(M−1) × 0.6 ms` = 8.4 ms per all-reduce, ≈6 hours/month of idle GPUs at 1 step/s, ≈$1,700/month at $35/node-hour. The correct design is to co-locate the job in one AZ (or one placement group on one fabric) and buy durability at the checkpoint layer, because resilience for a barrier-synchronous job means "restart from a checkpoint," not "survive a zone loss mid-step" — such a job cannot survive losing a single node anyway.

**5. You migrate a service from an ALB to an NLB and per-AZ load becomes uneven while a cross-zone transfer line item disappears. What changed, and how do you decide whether to change it back?**

Cross-zone load balancing defaults differ by product: an ALB always load-balances across zones (it is not disableable at the load-balancer level), so a request arriving in one AZ can be served by a target in another — even distribution, plus cross-AZ transfer on the bill. An NLB has cross-zone balancing **off by default**, configurable per target group, so traffic stays in the AZ it arrived in. After the migration each AZ's targets serve only that AZ's arriving traffic, which exposes any skew in how traffic reaches each zone. **Decide with arithmetic, not reflex.** For 40 TB/month through the LB across 3 AZs, roughly 2/3 of LB→target bytes cross a zone: 26,667 GB × $0.010 × 2 ends ≈ **$533/month**. The alternative is provisioning each AZ for its own peak instead of the fleet average — at a 20 % skew and a 30-instance fleet costing ~$16,600/month, that is ~$3,300/month of extra capacity. Cross-zone ON is six times cheaper here. The general form: compare the transfer cost of spreading against the *capacity* cost of the imbalance you accept by not spreading, and note that the answer depends entirely on your fleet size and skew.

**6. When would you choose Transit Gateway over VPC peering, and what is the crossover?**

Choose by whether your constraint is **breadth** or **volume**. Peering has no per-GB fee of its own (you still pay whatever cross-AZ or cross-region transfer rides over it) but is non-transitive and O(n²) to manage — every pair needs a connection and route-table entries in both VPCs. Transit Gateway is transitive, O(n) to manage, and gives you one place to put routing and inspection, but charges **$0.05 per attachment-hour** (≈$36/month per attachment) *and* **$0.02 per GB processed**. The crossover is dominated by the per-GB fee: at 10,000 GB/month between two VPCs, TGW is $72 + $200 = $272; at 100,000 GB it is $2,072; at 1 PB it is $20,072. So TGW is the right answer for many VPCs each exchanging modest traffic, and the wrong answer for a data-heavy pair — route those over a direct peering connection even inside an otherwise hub-and-spoke design. Separately, if the requirement is exposing *one service* across a trust boundary rather than general connectivity, neither is right: PrivateLink at $0.01/endpoint-hour per AZ plus $0.01/GB gives you a one-way, service-scoped path where the network itself enforces the boundary.

**7. How would you find, from scratch and without the pricing console, what a given AWS network hop costs in your region today?**

Query the public Price List Bulk API. `https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/index.json` lists every service offer; each offer has a `currentRegionIndexUrl` that maps region codes to a per-region price file, so you download one region rather than the multi-hundred-megabyte global file. Then join `.products[sku].attributes` (which carries `usagetype`, `transferType`, `fromLocation`, `toLocation`) against `.terms.OnDemand[sku][].priceDimensions[]` (which carries `description`, `unit`, and `pricePerUnit.USD`). The relevant offers are `AWSDataTransfer` (cross-AZ, inter-region, internet egress), `AmazonVPC` (endpoints, Transit Gateway, public IPv4), `AWSELB` (load balancer hours and LCUs), and `AmazonEC2` (NAT Gateway — note that file is ~480 MB, so stream it through `grep` rather than parsing it whole). Every file carries a `publicationDate`; record it alongside any number you quote, because the value of this exercise is producing a *dated* figure rather than a remembered one.

## Connections & what's next
This lesson's cost lens builds directly on **Lesson 03**'s cross-zone load-balancing mechanics — the ALB/NLB default asymmetry is one of the two dominant egress culprits named here — and sets up **Lesson 05** (Kubernetes networking), where `internalTrafficPolicy: Local`, `trafficDistribution`, and topology-aware routing become the concrete cluster-level tools for keeping traffic in-zone, and where the overlay-MTU version of §8's PMTU problem appears again with different arithmetic. The training-specific compounding described in §10 is the bridge into **Lesson 07** (GPU/RDMA networking) and module 09's fabric material, which covers the dedicated interconnect this lesson argues collective traffic must stay on.

Next: **[05-kubernetes-networking.md](05-kubernetes-networking.md)** — where a Service VIP actually resolves (iptables/IPVS/eBPF), and where the topology-aware routing levers referenced above live in the K8s dataplane.

## References & further reading

**Primary sources — read directly for this lesson**

1. **AWS Price List Bulk API**, offer index (https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/index.json) — the entry point used for every AWS figure in this lesson, and the source of the `curl | jq` recipe in §1.
2. **`AWSDataTransfer`, us-east-1, publication date 2026-07-20** — `DataTransfer-Regional-Bytes` at **$0.010/GB** ("regional data transfer — in/out/between EC2 AZs or using elastic IPs or ELB"); `USE1-DataTransfer-xAZ-Out-Bytes` and `USE1-DataTransfer-xAZ-In-Bytes` at **$0.00/GB**; inter-region outbound `USE1-EU-AWS-Out-Bytes`, `USE1-USW2-AWS-Out-Bytes`, `USE1-APN1-AWS-Out-Bytes` all at **$0.020/GB** with the matching `-In-Bytes` meters at **$0.000/GB**; internet egress tiers **$0.090 / $0.085 / $0.070 / $0.050 per GB** at 10 TB / 40 TB / 100 TB / above-150 TB breakpoints. **Correction to the previous version of this lesson:** it gave internet egress as "$0.05–0.09/GB" without the breakpoints and asserted a single cross-AZ rate; both are now sourced and dated, and the existence of the zero-rated `xAZ` meters is new information that changes how you should verify a cost hypothesis.
3. **`AmazonVPC`, us-east-1, publication date 2026-07-24** — `VpcEndpoint-Hours` **$0.01/hour** (per endpoint, i.e. per AZ) and `VpcEndpoint-Bytes` **$0.01/GB** up to 1 PB, **$0.006/GB** 1–5 PB, **$0.004/GB** above 5 PB; `TransitGateway-Hours` **$0.05/hour per attachment** across VPC, VPN, Direct Connect, peering, and Connect attachment types, with `TransitGateway-Bytes` **$0.02/GB**; `PublicIPv4:InUseAddress` and `PublicIPv4:IdleAddress` both **$0.005/hour**; Gateway Load Balancer endpoint **$0.01/hour + $0.0035/GB**. Gateway endpoints for S3 and DynamoDB have **no meter** in this file, consistent with their being free of charge.
4. **`AmazonEC2`, us-east-1, publication date 2026-08-17** — NAT Gateway: `NatGateway-Hours` **$0.045/hour**, `NatGateway-Bytes` **$0.045/GB**, and the provisioned-bandwidth variant `NatGateway-Prvd-Gbps` **$1.076/Gbps-hour** with `NatGateway-Prvd-Bytes` at **$0.00/GB**. (This file is ~480 MB; it was streamed through `grep` rather than parsed whole.)
5. **`AWSELB`, us-east-1, publication date 2026-07-20** — ALB **$0.0225/hour + $0.008/LCU-hour**; NLB **$0.0225/hour + $0.006/LCU-hour**; Gateway LB **$0.0125/hour + $0.004/LCU-hour**; Classic LB **$0.025/hour + $0.008/GB** processed.
6. **Google Cloud, "Network pricing"** (https://cloud.google.com/vpc/network-pricing), read directly — same-zone internal-IP transfer free; **inter-zone within a region $0.01/GiB, attributed to the sending VM's project**; the inter-region matrix ($0.02/GiB NA↔NA, $0.05 NA↔EU, $0.08 NA↔Asia, up to $0.14 for South America and Africa pairs); Premium Tier internet egress to North America at 1 GiB free / $0.12 / $0.11 / $0.08 per GiB across the 1 TiB and 10 TiB breakpoints; inbound free but with charges on processing resources (load balancers, Cloud NAT, protocol forwarding); Cloud NAT at **$0.0014/VM-hour** up to 32 VMs (**$0.044/hour** above), **$0.045/GiB** processed inbound and outbound, plus **$0.005/hour** per external IP.
7. `Documentation/networking/ip-sysctl.rst`, Linux master — `tcp_mtu_probing` semantics (0 = off, 1 = enabled on detected black hole, 2 = always, starting from `tcp_base_mss`), `tcp_probe_interval` reprobing every 10 minutes per RFC 4821, `tcp_mtu_probe_floor = 48`, and `tcp_retries2 = 15` giving the ~924.6 s connection-death bound used in §8.
8. `iptables-extensions(8)`, the `TCPMSS` target — `--clamp-mss-to-pmtu` and `--set-mss`, the standard MSS-clamping fix in §8.

**Sources named but not fetched in this pass — do not treat the wording as verified**

9. AWS's human-readable pricing and documentation pages (`aws.amazon.com/*/pricing`, `docs.aws.amazon.com`) are **blocked by this environment's egress policy**. Everything attributed to AWS above comes from the machine-readable Price List Bulk API instead, which is the same data AWS bills from and carries an explicit publication date. Product *behaviours* not present in the price list — specifically the ALB-always-cross-zone / NLB-off-by-default asymmetry in §6 and the scope limits of gateway versus interface endpoints in §4 — are stated here from the shape of the price list plus general product knowledge and were **not** re-verified against AWS documentation in this pass. Confirm those two behaviours against current AWS docs before relying on them in a design.
10. Corey Quinn's cross-AZ billing analyses (`lastweekinaws.com`) and third-party GPU-cloud egress write-ups — blocked by this environment's egress policy and not read. No figures from them are asserted here; the both-ends metering claim in §2 rests on the price-list SKU description and should be confirmed against your own Cost and Usage Report, which is the only authoritative answer for your account anyway.
