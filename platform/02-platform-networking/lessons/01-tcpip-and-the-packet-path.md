---
lesson: "A02.1"
title: "TCP/IP and the packet path"
module: "A-02"
concept: "kernel datapath"
status: not-started
est_time: "4 hrs"
artifacts: ["cwnd-vs-RTT ceiling calc", "cubic-vs-bbr netem throughput table"]
---

# A02.1 · TCP/IP and the packet path

> **Concept.** Throughput and latency are almost never limited by raw link bandwidth — they are limited by a specific stage in the kernel's RX/TX pipeline, by cwnd/RTT, or by per-core softirq saturation, and staff-level debugging means naming which one.
>
> Module: [🌐 Platform networking depth](../README.md) · Track A — Platform excellence

## Why this matters

At fleet scale the interesting network failures are not "the link is full." They are: a single CPU core pinned at 100% in softirq while the NIC has 24 idle queues; a cross-region checkpoint copy running at 40 Mbps on a 10 GbE path because the socket buffer caps cwnd below the bandwidth-delay product; a "corruption" bug that is actually a checksum offload lying to `tcpdump`; a latency-under-load spike that is CUBIC filling a bloated switch buffer. Every one of these is invisible if you think of TCP as an abstraction that "just moves bytes." The staff engineer sees the datapath as a pipeline with a nameable bottleneck stage, and can predict the observed rate with arithmetic before touching a knob. This is also the substrate that all non-RDMA GPU traffic rides — control plane, storage, and NCCL's TCP fallback — so a saturated storage NIC directly steals the cores feeding the GPUs.

## Core notes

**Skip (you already know):** the 3-way handshake / FIN teardown / sliding window; sockets, ports, and NAT mechanics; that CUBIC is the loss-based default congestion control; basic `tcpdump`/`ss` capture and filtering.

**The kernel datapath is a pipeline, not a function call.** Trace one packet in:

- **RX:** NIC DMAs the frame into a ring buffer (descriptor ring in host memory) → raises a **hardirq** → the handler does almost nothing except schedule a **softirq** (`NET_RX_SOFTIRQ`) → `net_rx_action` runs **NAPI polling** (switch from interrupt-per-packet to polling under load, the anti-livelock design) → **GRO** coalesces contiguous segments into one large skb → `netif_receive_skb` → protocol handlers (IP → TCP) → data lands on the **socket receive queue** → `recv()` copies to userspace.
- **TX:** mirror image. `send()` → socket send queue → TCP segmentation (or deferred via **GSO/TSO**, so the NIC does the slicing) → IP → **qdisc** (this is where `fq`/`fq_codel` live, the packet scheduler and AQM) → driver ring → NIC DMA out.
- **XDP** hooks run **before skb allocation**, in the driver poll routine — that is the whole point: you can drop/redirect/rewrite a packet without ever paying the skb cost.

**What actually limits throughput/latency** (rank these in an interview):

1. **BDP vs cwnd vs socket buffers.** `throughput = cwnd / RTT`. To fill a pipe you need `cwnd ≥ bandwidth × RTT` (the BDP), and cwnd cannot exceed the receiver's advertised window, which is bounded by the socket buffer (`tcp_rmem`/`tcp_wmem`, autotuned up to the max). A 10 Gbps × 80 ms path has a 100 MB BDP; if the rmem max is 6 MB, you top out around 600 Mbps no matter the link. **Staff line: "throughput = cwnd / RTT; tune the smaller ceiling."**
2. **Per-core softirq saturation + RX steering.** All RX softirq work for a queue lands on one core. One busy core at 100% `si` while others idle = you need **RSS** (NIC hashes flows across queues), **RPS** (software spread of the hash to other cores), **RFS** (steer to the core where the app runs), **XPS** (TX-side queue selection). Read `/proc/interrupts` and `/proc/softirqs` to find the hot core; IRQ affinity pins each queue's interrupt to a distinct core.
3. **PPS ceiling for small packets.** At 64-byte packets you hit a packets-per-second wall (per-packet skb + IRQ overhead) long before you hit bits-per-second. This is why "bandwidth" is the wrong question for small-message workloads.
4. **Head-of-line blocking.** One lost segment stalls delivery of everything behind it in that TCP stream (the ordered-byte-stream guarantee), independent of how much bandwidth is free.

**Congestion control is a model of the bottleneck.**

- **CUBIC** — loss-based. It grows cwnd until it sees loss, which on modern deep-buffered switches means it *fills the buffer first* → **bufferbloat** → high latency-under-load. Loss is its only signal.
- **BBR** — model-based. Actively estimates bottleneck bandwidth and min-RTT, then **paces** to keep the queue near-empty (operates at the BDP point, not the buffer-full point). Wins dramatically on lossy long-fat links where CUBIC misreads loss as congestion. Caveat: BBR (v1) can be aggressive and **starve** competing CUBIC flows sharing a link.
- **Bufferbloat** is *latency under load*, not throughput loss. Fix stack: **AQM** (CoDel / FQ-CoDel manage queue delay directly) + BBR + `tcp_notsent_lowat` (limit un-sent bytes held in the kernel, cutting local queueing latency for latency-sensitive apps).

**NIC offloads and their debugging tax.** Checksum offload, TSO/GSO (segmentation), GRO/LRO (receive coalescing), RSS (steering). They save CPU — but they make **`tcpdump` lie**: capture happens above the offload, so you see 64 KB "super-packets" (GRO/GSO artifacts) that never existed on the wire, and checksums shown as "incorrect" because the NIC hasn't computed them yet. `ethtool -k` shows offload state, `ethtool -S` shows per-queue hardware counters (drops, errors). Standard isolation move for a suspected corruption/MTU bug: **disable offloads** (`ethtool -K`) to make the capture reflect reality.

**XDP / eBPF is the "line rate without kernel bypass" answer.** XDP runs eBPF in the driver poll loop before skb alloc — used for DDoS drop (Cloudflare), L4 load balancing (Katran, Cilium), and sampling, at Mpps. **TC-BPF** runs later with full skb context (richer, slightly costlier). This is the modern alternative to DPDK-style full kernel bypass: you keep the kernel stack but shortcut the hot path.

**GPU tie-in.** The TCP datapath is what NCCL *falls back to* when RDMA isn't available, and what all control-plane and storage traffic uses regardless. Softirq saturation on the storage NIC steals the very cores that feed the GPUs; that is why you pin the RDMA NIC's IRQs and the storage NIC's IRQs to disjoint core sets and keep NAPI off the GPU-feeding cores. BDP math is the direct answer to "why is my cross-region checkpoint copy slow." (Fabric physics — RoCE/IB/GPUDirect/NCCL rails — lives in module 09; don't re-derive it here.)

## Worked example

Trace one 1 MB transfer on a loaded host and predict the rate before you measure it.

1. **Snapshot the flow:** `ss -tie` on the socket — read `cwnd`, `rtt` (smoothed + variance), `retrans`, `pacing_rate`, `rcv_space`. Say it reports `cwnd:42 rtt:80/5 mss:1448`. Then `cwnd_bytes = 42 × 1448 ≈ 60 KB`, and `throughput ≈ 60 KB / 0.080 s ≈ 760 KB/s ≈ 6 Mbps`. That is your predicted ceiling — write it down first.
2. **Find the limiting stage:** `tc -s qdisc show dev eth0` (drops/backlog at the qdisc?), `ethtool -S eth0` deltas across the transfer (rx/tx drops, per-queue imbalance), and a `bpftrace` one-liner counting retransmits: `bpftrace -e 'kprobe:tcp_retransmit_skb { @[comm] = count(); }'`. If retrans is climbing, loss (not buffer) is your ceiling.
3. **Change one variable:** shape a link with `tc qdisc add dev eth0 root netem delay 100ms loss 1%`, run the transfer under `cubic`, then flip `sysctl net.ipv4.tcp_congestion_control=bbr` and rerun. Record the throughput delta — on a 100 ms / 1% path BBR typically multiplies throughput several-fold because CUBIC collapses cwnd on every loss event.
4. **Deliverable:** a table with the measured cubic vs bbr rates, plus the `cwnd / RTT` calculation that predicted the CUBIC-limited rate to within a small factor. If prediction and measurement disagree badly, you mislocated the bottleneck — go back to step 2.

## Practice

<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

Build the runbook entry for "flow is slow, link is not full": the ordered checklist is (1) `ss -tie` → is cwnd the ceiling? compute `cwnd/RTT`; (2) is the socket buffer capping cwnd? check `tcp_rmem` max vs BDP; (3) `/proc/softirqs` + `/proc/interrupts` → is one core pinned in `si`? if so fix RSS/RPS/RFS/IRQ affinity; (4) `ethtool -S` → hardware drops or queue imbalance? (5) retransmits climbing → loss-limited, consider BBR + AQM. Reproduce the CUBIC-vs-BBR delta on a netem-shaped link and attach the predicted-vs-observed table.

## Self-check

- A 25 Gbps path between two regions has a 90 ms RTT. Your bulk copy tops out near 1.5 Gbps and `ss` shows near-zero retransmits. Where is the ceiling and what do you tune? **Answer:** It's the BDP-vs-buffer ceiling, not loss (retrans ~0 rules out congestion control). BDP = 25 Gbps × 0.090 s ≈ 280 MB; if `tcp_rmem`/`tcp_wmem` max is ~16 MB the receiver window caps cwnd around `16 MB / 0.090 s ≈ 1.4 Gbps` — exactly what you see. Raise the socket buffer maximums (and confirm window scaling is on) so cwnd can reach the BDP.

- `tcpdump` on the receiver shows 64 KB TCP segments and flags many checksums as "bad," but the application data is intact. What's happening? **Answer:** Offloads. GRO is coalescing wire segments into 64 KB super-packets before the capture point, so you're seeing packets that never existed on the wire; the "bad" checksums are checksum-offload artifacts (hardware validates/computes them below the capture hook). Nothing is corrupt. To capture wire-accurate frames, disable GRO/LRO and RX checksum offload with `ethtool -K`.

- Why can CUBIC produce high latency-under-load even when it achieves full throughput, and what is the fix stack? **Answer:** CUBIC is loss-based: its only congestion signal is a dropped packet, so on deep-buffered switches it grows cwnd until the buffer is full, adding a full buffer's worth of queueing delay to every packet — bufferbloat, i.e. latency under load. Throughput is fine; latency is wrecked. Fix stack: AQM at the bottleneck (CoDel/FQ-CoDel manage queue delay directly), switch to BBR (paces to keep the queue near-empty at the BDP), and set `tcp_notsent_lowat` to cut kernel-side send queueing for latency-sensitive flows.

## References

- https://blog.cloudflare.com/optimizing-tcp-for-high-throughput-and-low-latency/
- https://docs.kernel.org/networking/index.html
- https://www.brendangregg.com/ebpf.html
- https://www.kernel.org/doc/html/latest/networking/scaling.html
- https://research.google/pubs/bbr-congestion-based-congestion-control/
