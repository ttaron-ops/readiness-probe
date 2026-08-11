---
lesson: "A02.1"
title: "TCP/IP and the packet path"
module: "A-02"
concept: "kernel datapath"
status: not-started
est_time: "6 hrs"
prev: null
next: "02-dns-and-service-discovery.md"
artifacts: ["cwnd-vs-RTT ceiling calc", "cubic-vs-bbr netem throughput table", "BBR phase-cycle annotated trace"]
sources: 10
---

# A02.1 · TCP/IP and the packet path

> **Concept.** Throughput and latency are almost never limited by raw link bandwidth — they are limited by a specific stage in the kernel's RX/TX pipeline, by cwnd/RTT, or by per-core softirq saturation, and staff-level debugging means naming which one.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Where this fits

This is the module's opening lesson, so there's no prior lesson to build on — instead it sets the through-line for everything after it: **follow one packet, and put a mechanism, a cost, and a failure mode on every hop it takes.** Every later lesson (DNS, load balancing, cloud boundaries, the K8s dataplane, the mesh, RDMA, observability) assumes you can already answer "what stage of the kernel datapath is the bottleneck, and how do I prove it with a command, not a guess" — that fluency is what this lesson builds. It closes by handing off to lesson 02: once a packet is on the wire correctly, the very next question a client asks is "where do I even send it," which is DNS's job.

## Why this matters

At fleet scale the interesting network failures are not "the link is full." They are: a single CPU core pinned at 100% in softirq while the NIC has 24 idle queues; a cross-region checkpoint copy running at 40 Mbps on a 10 GbE path because the socket buffer caps cwnd below the bandwidth-delay product; a "corruption" bug that is actually a checksum offload lying to `tcpdump`; a latency-under-load spike that is CUBIC filling a bloated switch buffer; a noisy-neighbor flow that bufferbloats every other tenant sharing the same bottleneck queue. Every one of these is invisible if you think of TCP as an abstraction that "just moves bytes." The staff engineer sees the datapath as a pipeline with a nameable bottleneck stage, and can predict the observed rate with arithmetic before touching a knob. This is also the substrate that all non-RDMA GPU traffic rides — control plane, storage, and NCCL's TCP fallback — so a saturated storage NIC directly steals the cores feeding the GPUs.

## What's new here (calibration)

- You already know the 3-way handshake, FIN teardown, sliding window, sockets/ports/NAT, that CUBIC is the loss-based default, and basic `tcpdump`/`ss` — none of that is re-taught here.
- New: the RX/TX pipeline as a staged producer-consumer system with backpressure at each stage, and which tool exposes drops at which stage.
- New: congestion control as an explicit control-theory state machine (CUBIC's phases; BBR's four-phase cycle), not just "loss-based vs model-based" as a slogan.
- New: the multi-tenant and economic consequences of these mechanisms — noisy-neighbor bufferbloat, and translating PPS/softirq ceilings into a $/Gbps capacity-planning number.
- New: the specific offload-debugging traps (TSO vs GSO, `tcp_notsent_lowat` vs `SO_SNDBUF`) that separate "read the man page once" from "diagnosed this in production."

## Core concepts

**The kernel datapath is a pipeline, not a function call.** Trace one packet in:

- **RX:** NIC DMAs the frame into a ring buffer (descriptor ring in host memory) → raises a **hardirq** → the handler does almost nothing except schedule a **softirq** (`NET_RX_SOFTIRQ`) → `net_rx_action` runs **NAPI polling** (switch from interrupt-per-packet to polling under load, the anti-livelock design, bounded by a **poll budget** — `netdev_budget`, default **64 packets per interrupt cycle**, plus a time limit) → **GRO** coalesces contiguous segments into one large skb → `netif_receive_skb` → protocol handlers (IP → TCP) → data lands on the **socket receive queue** → `recv()` copies to userspace.
- **TX:** mirror image. `send()` → socket send queue → TCP segmentation (or deferred via **GSO/TSO**, so the NIC does the slicing) → IP → **qdisc** (this is where `fq`/`fq_codel` live, the packet scheduler and AQM) → driver ring → NIC DMA out.
- **XDP** hooks run **before skb allocation**, in the driver poll routine — that is the whole point: you can drop/redirect/rewrite a packet without ever paying the skb cost.

**The pipeline is a producer-consumer system, and backpressure has an address.** Every stage above can be the one that's saturated, and each has a *different* counter that names it — conflating them is the single most common debugging mistake:

| Stage | What backs up | Where you see it |
|---|---|---|
| NIC ring buffer | descriptor ring full because the CPU can't drain it fast enough | `ethtool -S eth0` — `rx_ring_0_drops` / `rx_missed_errors` / `rx_fifo_errors` climbing |
| NAPI poll budget | poll loop hits `netdev_budget` (64 pkts) or its time slice before draining the ring, reschedules softirq | `/proc/net/softnet_stat` column 2 (`squeezed` / budget-exceeded events) |
| Softirq / core | one core saturated in `si` doing RX processing for a queue that isn't spread | `/proc/softirqs`, `/proc/interrupts`, `mpstat -P ALL` showing one core pinned |
| Protocol / listen backlog | SYN or accept queue overflow because userspace isn't calling `accept()` fast enough | `nstat -az TcpExtListenOverflows` / `TcpExtListenDrops` |
| Socket receive queue | app not calling `recv()` fast enough, buffer fills | `ss -tm` showing `rmem_alloc` near `rmem_max`; `nstat TcpExtTCPRcvQDrop` |

Naming the exact stage — "it's `netdev_budget` squeeze, not a ring drop" — is what separates a staff diagnosis from a shrug.

**What actually limits throughput/latency** (rank these in an interview):

1. **BDP vs cwnd vs socket buffers.** `throughput = cwnd / RTT`. To fill a pipe you need `cwnd ≥ bandwidth × RTT` (the BDP), and cwnd cannot exceed the receiver's advertised window, which is bounded by the socket buffer (`tcp_rmem`/`tcp_wmem`, autotuned up to the max). A 10 Gbps × 80 ms path has a 100 MB BDP; if the rmem max is 6 MB, you top out around 600 Mbps no matter the link. **Staff line: "throughput = cwnd / RTT; tune the smaller ceiling."**
2. **Per-core softirq saturation + RX steering.** All RX softirq work for a queue lands on one core. One busy core at 100% `si` while others idle = you need **RSS** (NIC hashes flows across queues), **RPS** (software spread of the hash to other cores), **RFS** (steer to the core where the app runs), **XPS** (TX-side queue selection). RSS alone only balances *queues*, not necessarily *cores that matter* — if the app is pinned to one core or IRQs are affine to one core, RSS spreading queues across 8 NIC rings still funnels through a single saturated core unless RPS/RFS/IRQ affinity are also configured. Read `/proc/interrupts` and `/proc/softirqs` to find the hot core.
3. **PPS ceiling for small packets.** At 64-byte packets you hit a packets-per-second wall (per-packet skb + IRQ overhead) long before you hit bits-per-second. This is why "bandwidth" is the wrong question for small-message workloads, and it's also a direct capacity-planning input: a given core budget buys a fixed PPS ceiling, so "how many cores per NIC at line rate" is a real provisioning line item, not an afterthought — under-provision it and you silently cap $/Gbps efficiency well below the link's rated speed.
4. **Head-of-line blocking.** One lost segment stalls delivery of everything behind it in that TCP stream (the ordered-byte-stream guarantee), independent of how much bandwidth is free.

**Congestion control as control theory, not a slogan.** CUBIC and BBR are two control loops reacting to *different signals*, and each is best understood as an explicit state machine:

- **CUBIC** — loss-based. States: **slow start** (exponential cwnd growth until the first loss or a threshold) → **congestion avoidance** (cwnd grows along a cubic function of time since the last loss event, concave then convex around the previous cwnd_max) → **loss recovery** (fast retransmit/fast recovery, cwnd cut multiplicatively) → back to congestion avoidance. Loss is its *only* signal, and on modern deep-buffered switches "loss" means it first **fills the buffer** — bufferbloat, i.e., latency under load, well before it backs off.
- **BBR** — model-based, with an explicit **four-phase cycle**: **STARTUP** (exponential growth like slow start, but exits on a bandwidth-plateau signal rather than waiting for loss) → **DRAIN** (deliberately shrinks the queue it just built during STARTUP's overshoot) → **PROBE_BW** (steady state — cycles pacing gain up/down around the estimated bottleneck bandwidth to keep the pipe full without growing the queue) → **PROBE_RTT** (periodically — every ~10s — deliberately drains its own queue to near-empty for a short window, because min-RTT is only measurable when the queue is empty, and without this the model's RTT estimate would drift stale and understate the true min-RTT as ambient queuing rises). This periodic self-imposed drain is the detail that separates "I read a blog post about BBR" from staff depth.
- **Bufferbloat** is *latency under load*, not throughput loss. Fix stack: **AQM** (CoDel / FQ-CoDel manage queue delay directly) + BBR + `tcp_notsent_lowat`.

**Two easily-conflated buffer knobs — a classic interview trap.** `tcp_notsent_lowat` limits how many bytes the kernel will hold **unsent** in the send queue before `poll()`/`epoll` reports the socket writable again — it's a *local queueing latency* knob for the app, letting latency-sensitive writers avoid stacking megabytes of not-yet-transmitted data behind a slow path. `SO_SNDBUF` limits the **total** buffered bytes including data already sent but not yet ACKed — it's the ceiling that (via autotuning) determines how large cwnd can grow, i.e., it gates *throughput* against the BDP. Confusing the two means either capping your own throughput while trying to fix latency, or the reverse.

**NIC offloads and their debugging tax.** Checksum offload, TSO/GSO (segmentation), GRO/LRO (receive coalescing), RSS (steering) — and a distinction worth being precise about: **TSO** (TCP Segmentation Offload) means the **NIC's ASIC** slices one large buffer into MTU-sized frames in hardware; **GSO** (Generic Segmentation Offload) is the **kernel software fallback** that does the same slicing just before handoff to the driver, used when hardware TSO isn't available or isn't applicable (e.g., some tunnel/encap paths). Same visible effect (large "packets" leaving the socket layer), different place to debug a segmentation bug — `ethtool -k` tells you which one is actually active. These offloads save CPU — but they make **`tcpdump` lie**: capture happens above the offload, so you see 64 KB "super-packets" (GRO/GSO artifacts) that never existed on the wire, and checksums shown as "incorrect" because the NIC hasn't computed them yet. `ethtool -k` shows offload state, `ethtool -S` shows per-queue hardware counters (drops, errors). Standard isolation move for a suspected corruption/MTU bug: **disable offloads** (`ethtool -K`) to make the capture reflect reality. (Related: **BIG TCP** raises GSO/GRO segment limits past the historical 64 KB ceiling for very-high-bandwidth hosts — see references.)

**XDP / eBPF is the "line rate without kernel bypass" answer.** XDP runs eBPF in the driver poll loop before skb alloc — used for DDoS drop (Cloudflare), L4 load balancing (Katran, Cilium), and sampling, at Mpps. Critically, XDP is **not** kernel bypass the way DPDK is: DPDK maps the NIC into userspace and the kernel network stack never sees the traffic at all, whereas XDP programs run *inside* the kernel's driver poll loop and can still hand packets up to the normal stack (or to `iptables`/`tc`) when they choose to — you keep interoperability with the rest of the kernel's networking tools, you just get to short-circuit the hot path before the expensive parts (skb allocation, GRO, protocol stack) run. **TC-BPF** runs later with full skb context (richer, slightly costlier).

**GPU tie-in.** The TCP datapath is what NCCL *falls back to* when RDMA isn't available, and what all control-plane and storage traffic uses regardless. Softirq saturation on the storage NIC steals the very cores that feed the GPUs; that is why you pin the RDMA NIC's IRQs and the storage NIC's IRQs to disjoint core sets and keep NAPI off the GPU-feeding cores. BDP math is the direct answer to "why is my cross-region checkpoint copy slow." (Fabric physics — RoCE/IB/GPUDirect/NCCL rails — lives in module 09; don't re-derive it here.)

## Perspectives

**Kernel-internals.** Think of the RX path as a pipeline with a queue and a service rate at every stage, and debugging as finding which queue is growing. Ring buffer, NAPI poll budget, GRO, and socket backlog each have their own counter and their own remedy (bigger ring vs more NAPI budget vs RPS/RFS vs a faster consumer) — treating "packet loss" as one undifferentiated symptom instead of naming the specific stage is the tell of a non-staff diagnosis.

**Congestion-control-as-control-theory.** CUBIC and BBR are feedback controllers reacting to different observables — loss vs bandwidth/RTT model — and each has explicit phases with entry/exit conditions. Reasoning about them as state machines (not "algorithms that make cwnd bigger") is what lets you predict behavior on a link you haven't tested: e.g., a lossy long-fat satellite link should favor BBR's model over CUBIC's loss-triggered backoff, and you can say why before running the experiment.

**Multi-tenant / noisy-neighbor.** On a shared host or shared switch buffer, one CUBIC flow that fills the buffer chasing throughput imposes bufferbloat latency on *every other flow* queued behind it at that bottleneck — a classic negative externality invisible to the flow causing it (its own throughput looks fine). This is precisely why cloud providers increasingly default egress paths to BBR or to FQ-CoDel-managed queues: it converts an unbounded per-flow queue-filling race into a fairness-managed shared resource, protecting tenants from each other without any tenant having to opt in.

**Economics.** PPS ceilings and softirq saturation are not just a debugging curiosity — they're a capacity-planning input. If a core budget buys, say, a few million PPS before softirq saturates it, then "how many cores do I reserve per 100 Gbps NIC at small-packet line rate" becomes a real provisioning number, and getting it wrong either wastes cores (over-provisioned) or silently caps achievable throughput well below the link's rated speed (under-provisioned) — a direct $/Gbps efficiency number a staff engineer should be able to produce, not just intuit.

## Real-world use cases

- **Spotify — "Smoother Streaming with BBR"** (https://engineering.atspotify.com/2018/08/smoother-streaming-with-bbr): production BBR rollout with before/after streaming-quality metrics — a concrete example of model-based congestion control winning on real lossy/variable client links at scale, not just in a lab.
- **Dropbox — "Evaluating BBRv2 on the Dropbox Edge Network"** (https://dropbox.tech/infrastructure/evaluating-bbrv2-on-the-dropbox-edge-network): a rigorous production A/B of BBRv2 vs BBRv1 vs CUBIC that surfaces the CUBIC-fairness-starvation caveat directly — required reading before you flip a fleet default to BBR.
- **Cloudflare — "How to drop 10 million packets per second"** (https://blog.cloudflare.com/how-to-drop-10-million-packets/): the canonical production numbers for XDP filtering before skb allocation, showing exactly why "shortcut before the expensive part of the pipeline" is the right mental model for line-rate defense.

## Worked example

Trace one 1 MB transfer on a loaded host and predict the rate before you measure it.

1. **Snapshot the flow:** `ss -tie` on the socket — read `cwnd`, `rtt` (smoothed + variance), `retrans`, `pacing_rate`, `rcv_space`. Say it reports `cwnd:42 rtt:80/5 mss:1448`. Then `cwnd_bytes = 42 × 1448 ≈ 60 KB`, and `throughput ≈ 60 KB / 0.080 s ≈ 760 KB/s ≈ 6 Mbps`. That is your predicted ceiling — write it down first.
2. **Find the limiting stage:** `tc -s qdisc show dev eth0` (drops/backlog at the qdisc?), `ethtool -S eth0` deltas across the transfer (rx/tx drops, per-queue imbalance), `/proc/net/softnet_stat` (NAPI budget squeeze), and a `bpftrace` one-liner counting retransmits: `bpftrace -e 'kprobe:tcp_retransmit_skb { @[comm] = count(); }'`. If retrans is climbing, loss (not buffer, not budget) is your ceiling.
3. **Change one variable:** shape a link with `tc qdisc add dev eth0 root netem delay 100ms loss 1%`, run the transfer under `cubic`, then flip `sysctl net.ipv4.tcp_congestion_control=bbr` and rerun. Record the throughput delta — on a 100 ms / 1% path BBR typically multiplies throughput several-fold because CUBIC collapses cwnd on every loss event. If you have `ss -tie` running during the BBR run, watch cwnd/pacing_rate dip and recover on roughly a 10-second cadence — that's PROBE_RTT firing.
4. **Deliverable:** a table with the measured cubic vs bbr rates, plus the `cwnd / RTT` calculation that predicted the CUBIC-limited rate to within a small factor, plus an annotated trace showing at least one BBR PROBE_RTT dip. If prediction and measurement disagree badly, you mislocated the bottleneck — go back to step 2 and check the backpressure table above stage by stage.

## Practice

<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

Build the runbook entry for "flow is slow, link is not full": the ordered checklist is (1) `ss -tie` → is cwnd the ceiling? compute `cwnd/RTT`; (2) is the socket buffer capping cwnd? check `tcp_rmem` max vs BDP, and don't confuse this with `tcp_notsent_lowat`; (3) `/proc/softirqs` + `/proc/interrupts` + `/proc/net/softnet_stat` → is one core pinned in `si`, or is NAPI budget being squeezed? if so fix RSS/RPS/RFS/IRQ affinity; (4) `ethtool -S` → hardware ring drops or queue imbalance? (5) retransmits climbing → loss-limited, consider BBR + AQM, and check whether a noisy neighbor is bufferbloating the shared bottleneck. Reproduce the CUBIC-vs-BBR delta on a netem-shaped link and attach the predicted-vs-observed table plus a labeled BBR phase trace (STARTUP/DRAIN/PROBE_BW/PROBE_RTT).

## Common pitfalls

- **"More bandwidth fixes it."** Compute `cwnd / RTT` and the BDP against your actual socket buffer ceiling first — a fatter link does nothing if the receive window or a lossy hop is the real ceiling.
- **"BBR is strictly better than CUBIC."** BBRv1 can be aggressive enough to starve CUBIC flows sharing the same bottleneck buffer, because it doesn't back off on loss the way CUBIC does — this is exactly why BBRv2 added an explicit loss-response signal. Don't flip a shared fleet default without checking who else is on that link.
- **"`tcpdump` shows the wire."** GRO/GSO and checksum offload run below or above the capture point depending on direction, so captures show packets and checksums that never existed as such on the wire. Disable offloads (`ethtool -K`) or capture at a tap for ground truth.
- **"XDP is kernel bypass, like DPDK."** XDP runs inside the kernel's driver poll loop and can still interoperate with `iptables`/`tc` and hand packets up to the normal stack — DPDK genuinely removes the kernel from the path. Conflating the two leads to wrong assumptions about what still works (conntrack, `tc` shaping) alongside an XDP program.
- **"RSS alone balances load across cores."** RSS spreads packets across NIC *queues* by flow hash; if IRQ affinity or app placement still funnels the actual processing onto one core, you can have 16 busy queues and one saturated core. RPS/RFS and explicit IRQ affinity are the actual cross-core balancing fix.

## Self-check

- A 25 Gbps path between two regions has a 90 ms RTT. Your bulk copy tops out near 1.5 Gbps and `ss` shows near-zero retransmits. Where is the ceiling and what do you tune? **Answer:** It's the BDP-vs-buffer ceiling, not loss (retrans ~0 rules out congestion control). BDP = 25 Gbps × 0.090 s ≈ 280 MB; if `tcp_rmem`/`tcp_wmem` max is ~16 MB the receiver window caps cwnd around `16 MB / 0.090 s ≈ 1.4 Gbps` — exactly what you see. Raise the socket buffer maximums (and confirm window scaling is on) so cwnd can reach the BDP.

- `tcpdump` on the receiver shows 64 KB TCP segments and flags many checksums as "bad," but the application data is intact. What's happening? **Answer:** Offloads. GRO is coalescing wire segments into 64 KB super-packets before the capture point, so you're seeing packets that never existed on the wire; the "bad" checksums are checksum-offload artifacts (hardware validates/computes them below the capture hook). Nothing is corrupt. To capture wire-accurate frames, disable GRO/LRO and RX checksum offload with `ethtool -K`.

- Why can CUBIC produce high latency-under-load even when it achieves full throughput, and what is the fix stack? **Answer:** CUBIC is loss-based: its only congestion signal is a dropped packet, so on deep-buffered switches it grows cwnd until the buffer is full, adding a full buffer's worth of queueing delay to every packet — bufferbloat, i.e. latency under load. Throughput is fine; latency is wrecked. Fix stack: AQM at the bottleneck (CoDel/FQ-CoDel manage queue delay directly), switch to BBR (paces to keep the queue near-empty at the BDP), and set `tcp_notsent_lowat` to cut kernel-side send queueing for latency-sensitive flows.

- A `ss -tie` trace of a long-lived BBR flow shows cwnd and pacing_rate periodically dip every ~10 seconds and then recover, with no corresponding loss event. Is this a bug? **Answer:** No — that's BBR's PROBE_RTT phase. Because BBR's bandwidth/RTT model only gets an accurate min-RTT reading when the queue is actually empty, BBR periodically (roughly every 10s) deliberately shrinks its own in-flight data to drain the path and resample min-RTT, then returns to PROBE_BW. It's a designed self-correction against a stale RTT estimate, not a fault.

- Two engineers disagree: one wants to raise `SO_SNDBUF` to fix throughput on a long-fat link, the other wants to lower `tcp_notsent_lowat` to fix bufferbloat latency from a chatty app. Are they fighting over the same knob? **Answer:** No — they're orthogonal. `SO_SNDBUF` bounds total buffered bytes (sent-but-unacked plus unsent), and via autotuning it's the ceiling that lets cwnd grow toward the BDP, so raising it addresses the throughput problem. `tcp_notsent_lowat` bounds only the *unsent* portion held in the kernel before the socket reports writable again, addressing local queueing latency for the app — lowering it doesn't touch cwnd growth or throughput at all. Both changes can coexist.

## Connections & what's next

This lesson is the spine the rest of the module hangs off: lesson 05 (Kubernetes networking) revisits the RX/TX pipeline through the lens of a Service VIP DNAT and conntrack; lesson 07 (GPU/RDMA) contrasts this entire pipeline with the path RDMA deliberately bypasses; lesson 08 (observability) turns the backpressure table above into a symptom→cause decision tree with `hubble`/`pwru`. The very next lesson picks up right where a correctly-delivered packet leaves off: before any of this machinery runs, the client had to resolve a name to an address — and DNS has its own amplifier failure modes that look, on the surface, exactly like a TCP problem.

## References & further reading

**Primary sources**
- RFC 5681 — TCP Congestion Control: https://www.rfc-editor.org/rfc/rfc5681
- BBR congestion-based congestion control (Google Research): https://research.google/pubs/bbr-congestion-based-congestion-control/
- Linux kernel networking scaling docs (RSS/RPS/RFS/XPS): https://www.kernel.org/doc/html/latest/networking/scaling.html
- Linux kernel networking documentation index: https://docs.kernel.org/networking/index.html

**Real-world engineering blogs**
- Spotify, "Smoother Streaming with BBR": https://engineering.atspotify.com/2018/08/smoother-streaming-with-bbr
- Dropbox, "Evaluating BBRv2 on the Dropbox Edge Network": https://dropbox.tech/infrastructure/evaluating-bbrv2-on-the-dropbox-edge-network
- Cloudflare, "How to drop 10 million packets per second": https://blog.cloudflare.com/how-to-drop-10-million-packets/
- Cloudflare, "Optimizing TCP for high throughput and low latency": https://blog.cloudflare.com/optimizing-tcp-for-high-throughput-and-low-latency/

**Deeper dives**
- LWN.net, "BIG TCP" (why >64KB GSO/GRO segments now exist): https://lwn.net/Articles/894280/
- Brendan Gregg, eBPF resources and tracing tools: https://www.brendangregg.com/ebpf.html
