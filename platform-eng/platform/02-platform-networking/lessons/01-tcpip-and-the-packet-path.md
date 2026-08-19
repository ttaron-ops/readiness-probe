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
sources: 16
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
- New: congestion control as an explicit control-theory state machine (CUBIC's phases; BBR's four-phase cycle), not just "loss-based vs model-based" as a slogan — with the *actual constants* the Linux implementations use.
- New: the multi-tenant and economic consequences of these mechanisms — noisy-neighbor bufferbloat, and translating PPS/softirq ceilings into a $/Gbps capacity-planning number.
- New: the specific offload-debugging traps (TSO vs GSO, `tcp_notsent_lowat` vs `SO_SNDBUF`) that separate "read the man page once" from "diagnosed this in production."
- **Deliberately out of scope here:** netfilter hooks, connection tracking, NAT, and kube-proxy's rule structure. Those are a full lesson of their own (`modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md`) and lesson 05 of this module builds on them. When this lesson's diagrams say "netfilter," treat it as one labelled box whose insides you already own.

## Core concepts

### 1. What this lesson is actually about: two pipelines and five queues

The mental model most engineers carry is that `write()` puts bytes on the wire and `read()` takes them off. That model has no bottleneck in it, which is why it produces the answer "the network is slow."

The accurate model is: **the datapath is two staged pipelines, TX and RX, and every stage is a queue with a producer, a consumer, and a service rate.** A queue that grows faster than it drains either adds latency (if it is elastic) or drops (if it is bounded). Debugging is therefore not "find the packet loss" — it is **find the queue that is growing, and name it.** There are only about five candidates in each direction, each has its own counter, and each has a different remedy. Getting the wrong queue means applying a remedy that cannot possibly work, which is why "we raised the ring buffer" so often changes nothing.

Three things determine what a TCP flow actually achieves:

1. **How many bytes TCP is willing to have unacknowledged in the network at once** — the *congestion window* `cwnd`, jointly bounded by the receiver's advertised window. This gives the rate ceiling `throughput ≈ min(cwnd, rwnd) / RTT`.
2. **Whether the host can move packets fast enough** — the per-core softirq and per-packet costs, which cap *packets per second* independently of *bits per second*.
3. **What the intermediate queues do to the RTT** — because RTT is the denominator in (1), and because a full buffer converts throughput headroom into latency.

Everything below is a detailed answer to one of those three.

### 2. The TX path, queue by queue

Follow one 1 MB `write()` on a TCP socket.

**Stage T1 — the syscall and the send buffer.** `write()`/`send()` enters the kernel, takes the socket lock, and copies user bytes into kernel-owned `sk_buff` pages. This copy is the reason RDMA exists (see lesson 07). The socket has a byte budget, `sk_sndbuf`, and it covers **both** data already transmitted-but-unacknowledged and data not yet transmitted. If the budget is full, a blocking socket sleeps in `sendmsg` and a non-blocking one returns `EAGAIN`. The budget is auto-tuned between `net.ipv4.tcp_wmem`'s min and max — on current Linux the documented defaults are `min = 4 KiB`, `default = 16 KiB`, `max = between 64 KiB and 4 MiB depending on RAM` (`Documentation/networking/ip-sysctl.rst`). **Calling `setsockopt(SO_SNDBUF)` disables autotuning for that socket**, which is how well-meaning applications pin themselves to a small buffer forever.

**Stage T2 — TCP output.** `tcp_write_xmit()` decides how much of the send queue may leave right now. It is gated by three separate limits, and knowing which one binds is half of TCP debugging:

- `cwnd` — the congestion window (§5–§9).
- `rwnd` — the peer's advertised receive window, scaled by the window-scale option.
- **TSQ (TCP Small Queues)** — a per-socket byte limit on how much of *this socket's* data may sit below TCP (in qdisc + driver rings) at once. TSQ exists precisely to stop one bulk flow from filling the qdisc and inflating everyone else's latency. It is why a single flow's `tc -s qdisc` backlog stays small even at 40 Gb/s.

**Stage T3 — segmentation, or the deferral of it.** Rather than slicing 1 MB into 1448-byte segments in software and paying per-segment stack cost 700 times, the kernel keeps one large `sk_buff` and defers the slicing:

- **TSO** (TCP Segmentation Offload) — the **NIC ASIC** does the slicing. The kernel hands the device one buffer plus a template header; the NIC emits N MTU-sized frames with sequence numbers and checksums filled in.
- **GSO** (Generic Segmentation Offload) — the **kernel software fallback**, performed just before handoff to the driver, used when hardware TSO is unavailable or inapplicable (many tunnel/encap paths, some checksum combinations).

The size ceiling matters and it moved. Historically the ceiling was 64 KiB (`GSO_LEGACY_MAX_SIZE = 65536`, `include/linux/netdevice.h`). **BIG TCP** raised it: current kernels define `GSO_MAX_SIZE = 8 × GSO_MAX_SEGS = 8 × 65535 ≈ 524 KB` and `GRO_MAX_SIZE = 8 × 65535` likewise. This is why on a modern high-bandwidth host `tcpdump` can show "packets" far larger than 64 KB — a fact that trips up people who learned the 64 KB number and treat anything bigger as corruption.

**Stage T4 — the qdisc.** The queueing discipline is the host's own traffic shaper and AQM point. It is where `fq`, `fq_codel`, `cake`, `htb` live, and it is the last software queue before the driver. `fq` additionally implements **pacing** — spacing packets according to the socket's `pacing_rate` instead of dumping a window burst — which BBR depends on (`net/ipv4/tcp_bbr.c` notes that without `fq`, TCP falls back to internal pacing with one hrtimer per socket, which costs more).

**Stage T5 — the driver ring and the wire.** The driver writes descriptors into the TX ring and rings a doorbell; the NIC DMAs the buffers out. If the ring is full, the qdisc stops (`netif_tx_stop_queue`) and backpressure propagates upward. Completion interrupts (or NAPI TX cleanup) free the buffers.

### 3. The RX path, queue by queue

**Stage R1 — DMA into the RX ring.** The NIC writes the frame into a pre-posted buffer in host memory and updates a descriptor. If the driver has not replenished the ring fast enough, the NIC has nowhere to put the frame and **drops it in hardware**. Nothing in the kernel sees this packet; only the NIC's own counters do. That is why `ethtool -S` is the first RX command, not `netstat`.

**Stage R2 — hardirq → softirq.** The NIC raises an interrupt. The hard IRQ handler does almost nothing: it masks further interrupts for that queue and schedules `NET_RX_SOFTIRQ`. This is the anti-livelock design — under load, interrupt-per-packet would spend 100% of a core on interrupt entry/exit and never run the stack.

**Stage R3 — NAPI polling, and the two budgets that stop it.** `net_rx_action()` walks the list of NAPI instances that have work and polls each one. Two limits bound the whole softirq run, and **both defaults are commonly misquoted**:

| Knob | Default (Linux master, `net/core/hotdata.c`) | Scope |
|---|---|---|
| `net.core.netdev_budget` | **300 packets** | total packets across *all* NAPI instances in one `net_rx_action` run |
| `net.core.netdev_budget_usecs` | `2 × 10⁶ / HZ` µs → **2000 µs at HZ=1000** | wall-clock limit on the same run |
| per-NAPI `weight` (`dev_rx_weight`) | **64 packets** | how many packets *one* NAPI instance may take before yielding to the next |
| `net.core.netdev_max_backlog` | **1000 packets** | per-CPU backlog queue used by RPS and by non-NAPI drivers |

The "64" that circulates as *the* NAPI budget is the **per-device weight**, not `netdev_budget`. The distinction is operationally real: hitting the per-NAPI weight is normal round-robin fairness; hitting `netdev_budget` or the time limit means the softirq gave up with work still pending and rescheduled itself, which is the classic sign of a saturated core. That event is counted as `time_squeeze`.

**Stage R4 — GRO.** Generic Receive Offload coalesces consecutive segments of the same flow into one large `sk_buff` before the protocol stack runs, so the expensive per-packet work (routing lookup, netfilter, TCP input) is paid once for up to `GRO_MAX_SIZE` bytes instead of once per 1500-byte frame. This is a large win and a large source of confusion — see §11.

**Stage R5 — protocol stack and socket queues.** `netif_receive_skb` → IP → (the netfilter/conntrack box you already own from 01b.7) → TCP input. For an established connection the payload lands on the socket receive queue, bounded by `sk_rcvbuf` (auto-tuned within `net.ipv4.tcp_rmem`: `min = 4 KiB`, `default = 128 KiB`, `max = between 128 KiB and 32 MiB depending on RAM`). For a SYN it lands in the listener's SYN queue, and on completion in the accept queue, bounded by `min(backlog, net.core.somaxconn)` — **`somaxconn` defaults to 4096 since Linux 5.4; it was 128 before**, and stale runbooks still say 128.

### 4. The whole thing, in one picture

```
   ONE HOST, BOTH DIRECTIONS — every box is a queue; every ✗ is a drop with a name
   ================================================================================

  TX                                                     RX
  ──                                                     ──
  app: write(fd, buf, 1 MB)                              app: read(fd, ...)
        │                                                      ▲
        │ copy user→kernel                                     │ copy kernel→user
        ▼                                                      │
  ┌───────────────────────────┐                    ┌───────────────────────────┐
  │ T1 socket send queue      │                    │ R5 socket recv queue      │
  │    bound: sk_sndbuf       │                    │    bound: sk_rcvbuf       │
  │    autotune: tcp_wmem     │                    │    autotune: tcp_rmem     │
  │    ✗ EAGAIN / block       │                    │    ✗ TcpExtTCPRcvQDrop    │
  └───────────┬───────────────┘                    └───────────▲───────────────┘
              │                                                │
              ▼                                                │
  ┌───────────────────────────┐                    ┌───────────┴───────────────┐
  │ T2 tcp_write_xmit()       │                    │ R5' TCP input             │
  │  gated by min(cwnd, rwnd) │◀── ACKs ───────────│  reassembly, SACK, ACK gen│
  │  + TSQ per-socket limit   │    drive cwnd      │  ✗ ListenOverflows (SYN)  │
  └───────────┬───────────────┘                    └───────────▲───────────────┘
              ▼                                                │
  ┌───────────────────────────┐                    ┌───────────┴───────────────┐
  │ T3 TSO (NIC) / GSO (SW)   │                    │ R4 GRO coalescing         │
  │    ≤ GSO_MAX_SIZE ≈ 512KB │                    │    ≤ GRO_MAX_SIZE ≈ 512KB │
  └───────────┬───────────────┘                    └───────────▲───────────────┘
              ▼                                                │
  ┌───────────────────────────┐                    ┌───────────┴───────────────┐
  │ T4 qdisc (fq / fq_codel)  │                    │ R3 NAPI poll (softirq)    │
  │    pacing + AQM           │                    │  netdev_budget = 300 pkts │
  │    ✗ tc -s qdisc "dropped"│                    │  budget_usecs  = 2000 µs  │
  └───────────┬───────────────┘                    │  per-NAPI weight = 64     │
              ▼                                    │  ✗ softnet col 3 squeeze  │
  ┌───────────────────────────┐                    └───────────▲───────────────┘
  │ T5 driver TX ring → DMA   │                    ┌───────────┴───────────────┐
  │    ✗ ring full → stop_q   │                    │ R2 hardirq → NET_RX_SOFTIRQ│
  └───────────┬───────────────┘                    └───────────▲───────────────┘
              ▼                                    ┌───────────┴───────────────┐
           ═══════ WIRE ═══════════════════════════│ R1 NIC DMA into RX ring   │
                                                   │  ✗ rx_missed / rx_no_buf  │
                                                   │    (ethtool -S ONLY)      │
                                                   └───────────────────────────┘

  XDP hooks HERE ──────────────────────────────────────────┘  (before build_skb;
  no sk_buff exists yet — that is the entire point, see §13)
```

**Read the picture as a set of independent ceilings.** A flow can be limited at T2 by cwnd while R1 is completely idle; it can be limited at R3 by a saturated core while cwnd is enormous. Both look like "slow network." Only the counters distinguish them.

### 5. The counters that name the stage

This table is the reason the picture is worth memorising. Each row is *the* command for that stage; do not substitute a different one and hope.

| Stage | Symptom | Command | What to read |
|---|---|---|---|
| R1 NIC ring | packets vanish before the kernel sees them | `ethtool -S eth0 \| grep -Ei 'miss\|no_buf\|fifo\|drop'` | `rx_missed_errors`, `rx_no_buffer_count`, per-queue `rx_*_drops` climbing |
| R2/R3 softirq | one core at 100 % `si` | `mpstat -P ALL 1`, `cat /proc/softirqs`, `cat /proc/interrupts` | one CPU row with all the `NET_RX` growth |
| R3 budget | softirq gives up with work pending | `awk '{print $3}' /proc/net/softnet_stat` | **column 3** is `time_squeeze` (see below) |
| R3 backlog | RPS backlog overflow | `awk '{print $2}' /proc/net/softnet_stat` | **column 2** is `dropped` |
| R5 listen | SYNs accepted but app not `accept()`ing | `nstat -az \| grep -i listen` | `TcpExtListenOverflows`, `TcpExtListenDrops` |
| R5 socket | app too slow to `read()` | `ss -tmi` | `rmem_alloc` pinned near `rcv_ssthresh`/`rcvbuf`; `TcpExtTCPRcvQDrop` in `nstat` |
| T2 cwnd | flow slower than link | `ss -tie` | `cwnd`, `rtt`, `retrans`, `pacing_rate`, `bytes_retrans` |
| T4 qdisc | latency under load, local | `tc -s qdisc show dev eth0` | `backlog`, `dropped`, `overlimits`, fq_codel's `maxpacket`/`ecn_mark` |
| T5 ring | TX stalls | `ethtool -S eth0 \| grep tx_`, `ip -s link` | `tx_dropped`, driver-specific `tx_timeout` |

**`/proc/net/softnet_stat` is one line per CPU, hex, no header.** Straight from `net/core/net-procfs.c` the fields print in this order: `processed`, `dropped`, `time_squeeze`, then six historical zero fields, `received_rps`, `flow_limit_count`, `backlog_len`, `cpu_index`, `input_qlen`, `process_qlen`. So:

```
$ cat /proc/net/softnet_stat
0004ba7d 00000000 0000010f 00000000 00000000 00000000 00000000 00000000 \
00000000 00000000 00000000 00000000 00000000 00000000 00000000
0011cd42 00000000 00003b91 00000000 ...  (CPU 1)
   │        │        │
   │        │        └─ col 3: time_squeeze — 0x3b91 = 15,249 times this CPU's
   │        │            net_rx_action ran out of netdev_budget (300 pkts) or
   │        │            netdev_budget_usecs (2000 µs) with work still queued.
   │        │            CPU 0 has 0x10f = 271. That 56× asymmetry IS the bug.
   │        └─ col 2: dropped — backlog overflow (netdev_max_backlog = 1000)
   └─ col 1: processed — total packets this CPU took off the queues
```

*(Representative output; the field order is from the kernel source, the values are illustrative.)* **The single most common misreading is calling column 2 "squeezed."** Column 2 is `dropped`; `time_squeeze` is column 3. Reading the wrong column makes an idle machine look like it is dropping packets, or hides a real squeeze.

### 6. `throughput = cwnd / RTT`, and the arithmetic that follows

A window protocol can have at most one window of data in flight per round trip. So:

```
   throughput  =  min(cwnd, rwnd)  /  RTT
```

To saturate a path you need a window at least as large as the **bandwidth-delay product**:

```
   BDP = bandwidth × RTT
```

**Worked example 1 — the cross-region checkpoint copy.**

```
   link      = 25 Gbit/s
   RTT       = 90 ms  (us-east ↔ eu-west, typical)

   BDP = 25e9 bit/s × 0.090 s / 8 bit/byte
       = 2.25e9 / 8
       = 281.25 MB

   Receiver's tcp_rmem max on this host = 16 MiB = 16.78 MB
   → rwnd can never exceed ~16.78 MB (autotuning stops at tcp_rmem[2])

   achievable = 16.78e6 B / 0.090 s
              = 186.4 MB/s
              = 1.49 Gbit/s      ← 6 % of the link
```

Retransmits near zero and a 1.5 Gb/s ceiling on a 25 Gb/s path is a *window* problem, not a loss problem. The fix is `net.ipv4.tcp_rmem` on the receiver and `net.ipv4.tcp_wmem` on the sender, both raised to ≥ BDP, plus confirmation that window scaling is on. Note that autotuning will not exceed `tcp_rmem[2]`, and that an application calling `setsockopt(SO_RCVBUF)` freezes the buffer and ignores the sysctl entirely.

**Worked example 2 — how big can the window even be?** The TCP header's window field is 16 bits, so the unscaled maximum is 65535 bytes. Window scaling (RFC 7323, verified as the RFC the kernel cites in `net/ipv4/tcp_input.c`) adds a left-shift exponent negotiated in the SYN. Linux caps that exponent at `TCP_MAX_WSCALE = 14` (`net/ipv4/tcp_output.c`, `tcp_select_initial_window()`), giving:

```
   max advertised window = 65535 × 2^14 = 1,073,725,440 B ≈ 1 GiB
```

That is the hard ceiling for a single TCP connection's in-flight bytes. At 400 Gb/s and 10 ms RTT the BDP is 500 MB, which fits — but only just, and only if the buffers are actually configured that large. This is one honest argument for multiple parallel streams on very-long-fat pipes.

**Worked example 3 — the ramp is not free either.** Slow start doubles cwnd each RTT from `TCP_INIT_CWND = 10` segments (`include/net/tcp.h`; the value comes from the IW10 work). With MSS 1448 bytes:

```
   RTT 0:   10 × 1448 =    14.5 KB
   RTT 1:   20 × 1448 =    29.0 KB
   RTT 2:   40 × 1448 =    57.9 KB
   ...
   cwnd after n RTTs = 10 × 2^n segments

   To reach the 281 MB BDP above:
     281e6 / 1448 = 194,000 segments
     10 × 2^n ≥ 194,000  →  2^n ≥ 19,400  →  n ≥ 14.25  →  n = 15 RTTs

   15 RTTs × 90 ms = 1.35 s of pure ramp, during which you average
   roughly half the final rate.
```

So on a 90 ms path, a transfer shorter than ~200 MB never reaches steady state at all. **This is the mechanism behind "small transfers don't care about your link upgrade."** It is also why `net.ipv4.tcp_slow_start_after_idle` (default **1**, implementing RFC 2861) matters: an idle keep-alive connection has its cwnd reset after one RTO of idleness and has to re-ramp. Long-lived RPC connections that go quiet between bursts pay this repeatedly; setting it to 0 is a common and defensible tuning on trusted internal fabrics.

### 7. The handshake and the first seconds, on a timeline

```
   TIMELINE — connection setup, slow start, one loss, recovery, and RTO
   RTT = 20 ms, MSS = 1448 B, CUBIC.  Time flows down.  cwnd in segments.
   ═══════════════════════════════════════════════════════════════════════

   t=0      CLIENT ──SYN (wscale=7, SACK-perm, TS, MSS=1460)──▶ SERVER
                                                    │ SYN queue (bounded by
                                                    │ tcp_max_syn_backlog)
   t=20ms   CLIENT ◀─SYN-ACK (wscale=7, SACK-perm)──── SERVER
                                                    │ → accept queue
                                                    │   (min(backlog, somaxconn=4096))
   t=20ms   CLIENT ──ACK + first data───────────────▶ ✗ if accept queue full:
            cwnd = 10  (TCP_INIT_CWND)                 TcpExtListenOverflows++
                                                       and the SYN-ACK is silently
                                                       re-sent → looks like "slow
                                                       connect", never "refused"

            ── SLOW START ── cwnd doubles per RTT, HyStart++ watching ──
   t=20ms   cwnd=10   ▓▓                       14.5 KB in flight
   t=40ms   cwnd=20   ▓▓▓▓                     29.0 KB
   t=60ms   cwnd=40   ▓▓▓▓▓▓▓▓                 57.9 KB
   t=80ms   cwnd=80   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓        115.8 KB
   t=100ms  cwnd=160  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  231.7 KB
                      ↑ HyStart++ (cubic default, hystart=1) is sampling RTT.
                        If cwnd ≥ 16 (hystart_low_window) and this round's
                        min RTT rises above delay_min + clamp(delay_min/8,
                        4 ms, 16 ms), it EXITS slow start early — before any
                        loss — and sets ssthresh = cwnd.

   t=120ms  ── LOSS: one segment dropped at the bottleneck queue ──
            cwnd was 320.  Receiver keeps ACKing the last in-order byte and
            attaching SACK blocks for what did arrive.

   t=140ms  3rd duplicate ACK (or RACK's time threshold) fires FAST RETRANSMIT.
            CUBIC's ssthresh = cwnd × beta = 320 × 717/1024 = 224
                        ^^^^^^^^^^^^^^^^^^ NOT 0.5 — Linux CUBIC's beta is
                        717/1024 ≈ 0.7 (net/ipv4/tcp_cubic.c)
            cwnd is reduced toward ssthresh by PRR (RFC 6937), not slammed.

   t=140ms  ── CONGESTION AVOIDANCE (cubic growth from W_max = 320) ──
            cwnd  ▲
             320  ┤ ─ ─ ─ ─ ─ ─ ─ W_max ─ ─ ─ ─╭──────── convex: probing for
                  │                       ╭────╯          NEW bandwidth
             280  ┤                 ╭─────╯
                  │            ╭────╯   ← plateau near W_max: cautious
             224  ┤ ╭──────────╯          "is the old ceiling still there?"
                  │╭╯  ← concave: fast recovery of most of the lost window
                  ├╯
                  └────────────────────────────────────────▶ t
                    K = cubic_root((W_max − cwnd)/C),  C ≈ 0.4

   t=900ms  ── TAIL LOSS: the LAST segment of a burst is dropped ──
            No later segments exist, so no duplicate ACKs arrive.
            Fast retransmit CANNOT fire.  Two things can save you:
              (a) TLP (Tail Loss Probe, RFC 8985) — retransmit the last
                  segment after ~2×SRTT to elicit a dupACK, or
              (b) RTO — the full timeout.
            Without (a):  RTO = max(SRTT + 4×RTTVAR, TCP_RTO_MIN)
                          TCP_RTO_MIN = HZ/5 = 200 ms   (include/net/tcp.h)
                          → a 20 ms flow stalls for 200 ms, a 10× hit.
            On RTO: cwnd = 1, ssthresh halved-ish, back to slow start.
            Exponential backoff doubles RTO each attempt up to
            TCP_RTO_MAX = 120 s; tcp_retries2 = 15 gives a hypothetical
            ~924.6 s before the connection is killed.
```

**Three staff-level takeaways from that timeline.**

1. **A dropped SYN and a dropped final ACK look nothing alike.** Initial SYN retransmission is governed by `tcp_syn_retries` (default **6**, ≈ 131 s to final failure per `ip-sysctl.rst`). That is the mechanism behind "the pod hung for two minutes and then failed" — it is one lost SYN and the backoff ladder, not a slow server.
2. **The accept-queue overflow is invisible if you only look at the client.** The server silently drops the completing ACK and re-sends the SYN-ACK; the client sees latency, never a refusal. `nstat -az TcpExtListenOverflows` is the only honest signal.
3. **Tail loss is qualitatively worse than mid-stream loss**, because the self-clocking that fast retransmit depends on has stopped. This is exactly why RACK-TLP (RFC 8985, implemented in `net/ipv4/tcp_recovery.c`) exists, and why short request/response RPC traffic is far more RTO-sensitive than bulk transfer.

### 8. CUBIC, with the real constants

CUBIC's problem statement: classic Reno increases cwnd by one segment per RTT, so on a high-BDP path recovering from a loss takes an absurd number of round trips, and the growth rate depends on RTT (short-RTT flows beat long-RTT flows on a shared bottleneck). CUBIC makes window growth a function of **wall-clock time since the last congestion event**, not of RTT.

```
   W(t) = C · (t − K)³ + W_max        K = ∛( (W_max − W_reduced) / C )
```

The Linux implementation's constants (`net/ipv4/tcp_cubic.c`, read from master):

| Constant | Kernel value | Meaning |
|---|---|---|
| `beta` | `717` scaled by `BICTCP_BETA_SCALE = 1024` → **≈ 0.7** | multiplicative decrease factor on loss |
| `bic_scale` | `41` → `cube_rtt_scale = 410` → effective **C ≈ 0.4** | aggressiveness of the cubic term |
| `fast_convergence` | **1** (on) | on a *lower* cwnd than last time, shrink `W_max` further so a new flow can take its share |
| `tcp_friendliness` | **1** (on) | run a parallel Reno estimate; if Reno would be faster (short-RTT/low-BDP paths), use Reno's window |
| `hystart` | **1** (on) | exit slow start on a delay/ACK-train signal instead of on loss |
| `hystart_low_window` | **16** segments | don't run HyStart below this cwnd |
| `hystart_ack_delta_us` | **2000 µs** | max ACK spacing still counted as one "train" |
| `HYSTART_MIN_SAMPLES` | **8** | RTT samples needed per round before the delay test can fire |
| delay threshold | `clamp(delay_min/8, 4 ms, 16 ms)` | how much RTT inflation ends slow start |

**Why `beta ≈ 0.7` and not 0.5 matters.** Halving is the textbook AIMD answer, and it is what people quote. Linux CUBIC cuts to 70%, which recovers faster but is less polite; combined with time-based growth it is why CUBIC fills deep buffers so effectively. When you compute "how long to recover from a loss," use 0.7.

**Why `fast_convergence` exists.** Without it, an incumbent flow's `W_max` is sticky and a newly arriving flow struggles to claim bandwidth. With it, if the current loss happens at a cwnd *below* the previous `W_max`, the kernel sets `last_max_cwnd = cwnd × (1024 + 717) / 2048 ≈ 0.85 × cwnd` — deliberately aiming lower than where it just lost, to make room. This single line is most of CUBIC's fairness story on a shared bottleneck.

**The failure mode.** CUBIC's *only* congestion signal is loss (or ECN, if negotiated). On a switch with a 100 ms-deep buffer, CUBIC will grow until that buffer is full, because nothing before "full" tells it to stop. Throughput is excellent. Every other flow through that port now has 100 ms of standing queue added to its RTT. That is bufferbloat: **a throughput-optimal algorithm generating a latency externality it cannot observe.**

### 9. BBR, with the real gains

BBR asks a different question: instead of "has anything been lost yet?", it maintains an explicit **model of the path** — the maximum delivery rate observed (`BtlBw`) and the minimum RTT observed (`RTprop`) — and sends at that rate with just enough in flight to keep it busy. The optimal operating point (Kleinrock's) is `inflight = BtlBw × RTprop`: full utilisation, empty queue.

From `net/ipv4/tcp_bbr.c` (the algorithm summary is in the file header):

```
   bottleneck_bandwidth = windowed_max(delivered / elapsed, 10 round trips)
   min_rtt              = windowed_min(rtt, 10 seconds)
   pacing_rate          = pacing_gain × bottleneck_bandwidth
   cwnd                 = max(cwnd_gain × bottleneck_bandwidth × min_rtt, 4)
```

The four modes and their real constants:

```
   BBR STATE MACHINE + PROBE_BW GAIN CYCLE   (constants from tcp_bbr.c, master)
   ════════════════════════════════════════════════════════════════════════════

       ┌──────────────────────────────────────────────────────┐
       │                                                      │
       ▼                                                      │
   ┌─────────┐  bw stops growing ≥1.25×   ┌────────┐          │
   │ STARTUP │  for 3 consecutive rounds  │ DRAIN  │          │
   │         │ ─────────────────────────▶ │        │          │
   │ pacing  │  (bbr_full_bw_thresh=5/4,  │ pacing │          │
   │ gain =  │   bbr_full_bw_cnt = 3)     │ gain = │          │
   │ 2/ln2   │                            │ 1/2.885│          │
   │ ≈2.885  │                            └───┬────┘          │
   └─────────┘                                │ inflight      │
       ▲                                      │ ≤ BDP         │
       │ exit PROBE_RTT without full_bw       ▼               │
       │                              ┌──────────────┐        │
       │                              │  PROBE_BW    │────────┘ exit PROBE_RTT
       │                              │  (steady     │          with full_bw
       │                              │   state)     │
       │                              └──────┬───────┘
       │                                     │ no new min_rtt sample
       │                                     │ for 10 s (bbr_min_rtt_win_sec)
       │                              ┌──────▼───────┐
       └──────────────────────────────│  PROBE_RTT   │
                                      │ cwnd → 4 pkts│
                                      │ ≥ 200 ms     │
                                      │ (bbr_probe_  │
                                      │  rtt_mode_ms)│
                                      └──────────────┘

   PROBE_BW pacing_gain cycle — 8 phases, one round trip each,
   bbr_pacing_gain[] = { 5/4, 3/4, 1, 1, 1, 1, 1, 1 }:

   gain
   1.25 ┤ ██                                        ██
   1.00 ┤ ██    ██ ██ ██ ██ ██ ██                    ██   ██ ██ ██
   0.75 ┤ ██ ██ ██ ██ ██ ██ ██ ██                    ██ ██ ██ ██ ██
        └──┴──┴──┴──┴──┴──┴──┴──┴────────────────────┴──┴──┴──┴──┴──▶ rounds
          ↑  ↑                                       ↑
          │  └─ 0.75: drain whatever the probe queued, and yield to others
          └──── 1.25: send 25 % above the estimate — if bytes get delivered
                faster, BtlBw goes up; if they just queue, RTT goes up and
                the next phase drains it.  Six cruise rounds at 1.0 follow,
                and the whole cycle's starting phase is randomised over 7
                phases (bbr_cycle_rand) so co-existing flows desynchronise.

   Also: pacing is set ~1 % BELOW the estimate on average
         (bbr_pacing_margin_percent = 1) — a deliberate bias toward
         shorter queues; cwnd_gain = 2 so delayed/stretched ACKs
         don't starve the pipe.
```

**PROBE_RTT is the part people get wrong.** It is not "every 10 seconds." It fires when **no RTT sample at or below the current `min_rtt` estimate has been seen for 10 s** — i.e., when the estimate might be stale because the path has been continuously queued. When it fires, BBR cuts inflight to `bbr_cwnd_min_target = 4` packets for at least 200 ms, takes a clean RTT reading, then returns to PROBE_BW (or to STARTUP if it never reached full bandwidth). On a busy long-lived flow you will see it roughly every 10 s; on a flow that goes idle you may never see it, because idling produces clean RTT samples for free.

**When each wins, and the honest caveat.**

| Path | CUBIC | BBR | Why |
|---|---|---|---|
| Datacenter, shallow buffers, ~0 loss | fine | fine | neither signal is being stressed |
| WAN, deep buffers, low loss | fills the buffer; high latency-under-load | keeps the queue near empty | BBR's control variable is rate, not loss |
| Lossy last mile (Wi-Fi, cellular, satellite) | collapses cwnd on non-congestive loss | ignores random loss | this is BBR's headline win |
| Shared bottleneck with CUBIC neighbours | — | **BBRv1 can starve them** | BBRv1 has no loss response; it holds its rate while CUBIC repeatedly backs off. BBRv2/v3 added explicit loss and ECN responses precisely for this |
| Behind a token-bucket policer | badly | BBR has an explicit long-term ("LT") bandwidth estimator: if ≥ 20 % loss over ≥ 4-round intervals with consistent bandwidth, it concludes it is policed and paces to the policed rate for up to 48 rounds (`bbr_lt_*` constants) | |

Do not flip a fleet default to BBR without knowing who shares your bottleneck.

### 10. Bufferbloat, AQM, and pacing

**The problem.** A dumb FIFO with a large buffer converts congestion into delay instead of loss, and loss-based congestion control needs loss to react. So the deeper the buffer, the more standing queue a CUBIC flow builds before it is told to stop. The victim is every other flow through that queue, including latency-sensitive ones with tiny windows.

**CoDel's insight**: don't measure *queue length* (which is meaningless — a burst is fine, a standing queue is not), measure **how long packets are resident in the queue**. If the minimum sojourn time over a sliding `interval` never drops below `target`, the queue is standing, so start dropping (or ECN-marking) at an increasing rate until it does.

**fq_codel** combines that with per-flow queueing: hash flows into buckets, service them with deficit round-robin, run CoDel independently per bucket, and give brand-new flows priority. The practical effect is that a bulk transfer cannot delay a DNS query or an interactive RPC sharing the same egress, because they are in different buckets.

Defaults, straight from `man 8 tc-fq_codel` (iproute2):

| Parameter | Default | Meaning |
|---|---|---|
| `target` | **5 ms** | acceptable standing queue delay |
| `interval` | **100 ms** | window over which the minimum must be observed — set near worst-case RTT |
| `limit` | **10240 packets** | hard queue cap |
| `memory_limit` | **32 MB** | byte cap, whichever binds first |
| `flows` | **1024** | hash buckets |
| `quantum` | **1514 bytes** | DRR deficit — Ethernet MTU + header |
| `ecn` | **on** | mark rather than drop when the peer negotiated ECN |
| `drop_batch` | **64** | packets dropped at once when over limit |

Reading it in production:

```
$ tc -s qdisc show dev eth0
qdisc fq_codel 8002: root refcnt 2 limit 10240p flows 1024 quantum 1514
 target 5ms interval 100ms memory_limit 32Mb ecn drop_batch 64
 Sent 918273645 bytes 1204817 pkt (dropped 84, overlimits 0 requeues 12)
 backlog 0b 0p requeues 12
  maxpacket 64512 drop_overlimit 0 new_flow_count 8814 ecn_mark 1502
                  ^^^^^^^^^^^^^^^ 0      ^^^^^^^^^^^^^^ 8814  ^^^^^^^^^ 1502
                  │                      │                    │
                  │                      │                    └ CoDel decided the
                  │                      │                      queue was standing
                  │                      │                      and marked instead
                  │                      │                      of dropping — this is
                  │                      │                      AQM WORKING, not a fault
                  │                      └ how many times a flow arrived with an
                  │                        empty bucket: high = many short flows
                  └ 0 means we never hit `limit`; if this climbs, the queue is
                    overflowing outright and target/interval are irrelevant
```

*(Representative transcript; field names and defaults are from the iproute2 man page.)*

**Pacing** is the third leg. Even with a perfect cwnd, dumping a whole window at line rate creates a micro-burst that a shallow switch buffer cannot absorb. `fq` paces each socket to its `pacing_rate` (visible in `ss -tie`). BBR sets `pacing_rate` explicitly; CUBIC gets a pacing rate derived from cwnd/RTT with a headroom multiplier. On a host with `fq` or `fq_codel` as `net.core.default_qdisc`, this is free; without it, TCP falls back to per-socket hrtimer pacing, which the BBR source explicitly notes "may use more resources."

**The three-part fix stack for latency-under-load**, in the order you should apply it: (1) AQM at the actual bottleneck — which is often *not* your host, and may need to be a shaper you own; (2) a congestion control that targets rate rather than loss; (3) `tcp_notsent_lowat` so the application isn't stacking megabytes of unsent data in its own socket.

### 11. The two buffer knobs everyone conflates

This is a real interview discriminator, so be precise:

| | `SO_SNDBUF` / `tcp_wmem[2]` | `tcp_notsent_lowat` / `TCP_NOTSENT_LOWAT` |
|---|---|---|
| Bounds | **all** bytes in the send queue: sent-but-unacked **plus** unsent | **only** the unsent portion |
| Default | autotuned, `tcp_wmem` max = 64 KiB–4 MiB by RAM | `UINT_MAX` (0xFFFFFFFF) — i.e. effectively off |
| What it gates | how large cwnd can grow → **throughput** against the BDP | when `poll`/`epoll` reports the socket writable → **local queueing latency** |
| Symptom when wrong | long-fat pipe stuck far below line rate | an app that queues 8 MB of video and then cannot preempt it to send a control message |
| Interaction | `setsockopt(SO_SNDBUF)` **disables autotuning** for that socket | independent of cwnd entirely |

Lowering `tcp_notsent_lowat` (a common value is 128 KiB) does not reduce throughput, because the sent-but-unacked bytes — the ones that actually fill the pipe — are unaffected. It only stops the application from handing the kernel more data than the kernel can use soon.

### 12. Offloads, and why `tcpdump` lies

`ethtool -k eth0` lists them. The four that matter:

| Offload | Direction | What it does | What it breaks in a capture |
|---|---|---|---|
| `tx-checksumming` | TX | NIC computes the L4 checksum | `tcpdump` on the sender shows **"incorrect checksum"** for every packet — the checksum genuinely isn't computed yet at capture time |
| `tso` / `gso` | TX | one big buffer → N MTU frames (NIC / kernel) | capture shows one 64 KB–512 KB "segment" that never existed on the wire |
| `gro` / `lro` | RX | N wire frames → one big skb | capture shows one huge "segment" that was really 40 frames; per-packet timing is destroyed |
| `rx-checksumming` | RX | NIC validates the checksum | capture may show unverified/incorrect checksums |

**The capture point is above the offload in both directions**, which is why the artefacts are symmetrical. The isolation procedure when you suspect real corruption or a real MTU problem:

```bash
# 1. Record current state so you can restore it.
ethtool -k eth0 | grep -E 'tcp-segmentation|generic-(segmentation|receive)|rx-checksumming|tx-checksumming'

# 2. Disable the offloads that rewrite reality.
ethtool -K eth0 tso off gso off gro off lro off
#    (lro is often 'fixed [off]' on modern NICs — that's fine.)

# 3. Re-capture. Now frame sizes and checksums are the wire's.
tcpdump -i eth0 -nn -s 0 -w /tmp/wire.pcap 'host 10.0.4.17'

# 4. RESTORE. Leaving GRO off on a 100 GbE host costs a large fraction
#    of a core per Gb/s and will page you at 3 a.m.
ethtool -K eth0 tso on gso on gro on
```

**Do not skip step 4.** GRO off at high rate is one of the most reliable ways to create the softirq saturation described in §13.

**BIG TCP addendum.** If your capture shows segments larger than 65536 bytes, that is not corruption — `GSO_MAX_SIZE`/`GRO_MAX_SIZE` are `8 × 65535` on current kernels, enabled per-device (`ip link set dev eth0 gso_max_size ...`, `gro_max_size`). It exists because at 100–400 Gb/s the per-skb cost, not the per-byte cost, is the ceiling.

### 13. Steering, PPS ceilings, and the $/Gbps number

**The problem.** All the RX softirq work for a NIC queue runs on whichever CPU the queue's interrupt is affine to. One queue = one core. A 100 Gb/s NIC and one core is not a network problem; it is a scheduling problem.

The four mechanisms, in the order they apply:

| Mechanism | Layer | What it decides | Where configured |
|---|---|---|---|
| **RSS** | NIC hardware | which **hardware RX queue** a packet lands in, by hashing the flow tuple | `ethtool -X eth0` (indirection table), `ethtool -L` (queue count) |
| **RPS** | kernel software | which **CPU** processes a packet after it leaves the driver, by re-hashing | `/sys/class/net/eth0/queues/rx-N/rps_cpus` |
| **RFS** | kernel software | steer to the CPU **where the consuming application actually runs**, using a flow→CPU table | `net.core.rps_sock_flow_entries` + per-queue `rps_flow_cnt` |
| **XPS** | kernel software | which **TX queue** a transmitting CPU uses, to keep TX completions local | `/sys/class/net/eth0/queues/tx-N/xps_cpus` |

*(Mechanism list and configuration paths per `Documentation/networking/scaling.rst`.)*

**The trap.** RSS spreads *queues*. If IRQ affinity maps eight queues onto one core — which is exactly what happens when `irqbalance` is disabled and the default affinity mask is `0x1`, or when a container's CPU set collides with the IRQ set — you have eight busy queues and one saturated core. Check both:

```bash
# Which core is taking the interrupts?
$ grep -E 'eth0|mlx5' /proc/interrupts | awk '{print $1, $2, $3, $4, $5}'
  124: 90112384    0    0    0   mlx5_comp0@pci:0000:65:00.0
  125:        0   12    0    0   mlx5_comp1@pci:0000:65:00.0
       ^^^^^^^^^^ every interrupt on CPU0.  Queues exist; balance does not.

# Confirm the softirq side agrees.
$ grep NET_RX /proc/softirqs
 NET_RX:  412889201   118   940   204
```

**The PPS ceiling, worked.** Per-packet cost is roughly constant regardless of packet size, so small packets hit a packets-per-second wall long before a bits-per-second one:

```
   Assume a measured cost of ~2 µs of CPU per packet through the full
   stack (skb alloc, GRO, IP, netfilter, TCP, socket wakeup) — measure
   yours; this is generation- and config-dependent.

   one core   = 1 / 2e-6 = 500,000 pps

   at 1500 B frames:  500,000 × 1500 × 8  =  6.0 Gbit/s per core
   at   64 B frames:  500,000 ×   64 × 8  =  0.26 Gbit/s per core

   → to fill a 100 Gb/s NIC with 1500 B frames:  100 / 6.0  ≈ 17 cores
   → to fill it with 64 B frames:                100 / 0.26 ≈ 385 cores  (i.e. you can't)

   GRO changes the arithmetic decisively on RX: coalescing 40 frames into
   one skb means the ~2 µs is paid once per 60 KB instead of once per
   1500 B, a ~40× reduction in per-byte stack cost. This is why "GRO off"
   is a production incident and not a debugging preference.
```

**Turning that into money.** If a node has 64 vCPU and 17 of them are structurally reserved to drive the storage NIC at line rate, that is 27 % of the node's compute, and on a GPU node those are the cores that feed the dataloader. At a representative on-demand price for an 8×H100 node (order $30–$40/hr/node — confirm the current rate for your provider and region, since these move), 27 % of the host CPU is not the headline cost, but the *consequence* is: if the dataloader starves, the GPUs idle, and idle GPU time is billed at the full node rate. **The right capacity-planning statement is "N cores per 100 Gb/s of expected small-packet traffic," derived from your own measured per-packet cost, not from a vendor slide.**

### 14. XDP, tc-BPF, and what "bypass" actually means

```
   WHERE EACH HOOK RUNS — same received frame, four possible fates
   ═══════════════════════════════════════════════════════════════

   NIC DMA ──▶ driver poll ──▶ [ XDP ]  ──▶ build_skb() ──▶ [ tc ingress ]
                                 │                              │
                                 │                              ▼
      XDP_DROP  ── free the page, ~0 work.  No skb ever          IP stack
                   allocated: this is the entire reason              │
                   XDP hits tens of Mpps on one core.                ▼
      XDP_TX    ── bounce straight back out the same NIC        netfilter/conntrack
      XDP_REDIRECT ─ to another device or an AF_XDP socket        (01b.7's box)
      XDP_PASS  ── continue into the normal stack                     │
                                                                      ▼
                                                                   socket
   ─────────────────────────────────────────────────────────────────────────
   DPDK: the NIC's queues are unbound from the kernel driver entirely and
   mapped into a userspace process. The kernel stack never sees the traffic.
   iptables, tc, conntrack, ss, tcpdump — none of them apply, because as far
   as the kernel is concerned that NIC is not doing networking.
```

**The distinction that matters:** XDP is *early*, DPDK is *outside*. An XDP program runs inside the kernel's driver poll loop and can return `XDP_PASS` to hand the packet to the ordinary stack — so `tc`, netfilter, conntrack, and every kernel tool still work for the traffic you let through. DPDK removes the kernel from the path, which is why a DPDK dataplane must reimplement everything the kernel gave you for free (ARP, routing, conntrack, TCP if you need it) and why `tcpdump` on that interface shows nothing.

**tc-BPF** (`clsact` ingress/egress) runs *after* `build_skb()`, so it costs an skb allocation but gets full metadata: the socket, the cgroup, the conntrack state, the ability to redirect to a veth peer. This is the layer Cilium uses for most of its pod-datapath work, with XDP reserved for the cases where dropping before allocation is the whole point (DDoS filtering, node-level L4 load balancing).

### 15. The GPU-fleet tie

Everything above is what NCCL falls back to when RDMA is unavailable, and what *all* control-plane, checkpoint, and dataloader traffic uses regardless of fabric. Three consequences you own:

1. **Storage-NIC softirq steals dataloader cores.** Pin the storage NIC's IRQs and the RDMA NIC's IRQs to disjoint core sets, and keep both off the cores the dataloader workers are pinned to. Otherwise a checkpoint write measurably reduces GPU utilisation, and the correlation is invisible in GPU metrics.
2. **BDP math answers "why is my cross-region checkpoint copy slow."** §6's arithmetic applies directly: a 500 GB checkpoint over a 90 ms path with 16 MB buffers takes 500e9 / 186e6 ≈ **45 minutes** instead of the ~3 minutes the link could do. Raise the buffers, or use parallel streams, or move the checkpoint target.
3. **The RDMA path deletes stages T1–T5 and R1–R5 wholesale.** Lesson 07 makes that concrete; the point of drawing the ordinary pipeline this carefully is that "RDMA is faster" becomes a specific list of removed stages rather than a slogan.

## Perspectives

**Kernel-internals.** Think of the RX path as a pipeline with a queue and a service rate at every stage, and debugging as finding which queue is growing. Ring buffer, NAPI budget, GRO, backlog, and socket queue each have their own counter and their own remedy (more rings vs more budget vs RPS/RFS vs a faster consumer). Treating "packet loss" as one undifferentiated symptom is the tell of a non-staff diagnosis; the tell of a staff one is naming the counter before proposing the fix.

**Congestion-control-as-control-theory.** CUBIC and BBR are feedback controllers on different observables — loss vs a bandwidth/RTT model — each with explicit phases and entry/exit conditions. Reasoning about them as state machines lets you predict behaviour on a link you have not tested: a lossy satellite path should favour BBR because random loss is not congestion, and you can say why before running the experiment. It also tells you where each *fails*: BBRv1 on a shared bottleneck with CUBIC, CUBIC on any deep buffer.

**Multi-tenant / noisy-neighbour.** A CUBIC flow filling a shared bottleneck buffer imposes queueing delay on every other flow behind it, and the offending flow's own metrics look perfect. This is a textbook negative externality, and the remedy is structural, not behavioural: per-flow queueing (fq_codel) at the bottleneck converts an unbounded queue-filling race into a fairness-managed resource without any tenant opting in. That is why providers increasingly default egress queues to fq/fq_codel.

**Economics.** PPS ceilings are a provisioning line item. "How many cores per 100 Gb/s" has an answer derived from your measured per-packet cost, and getting it wrong either strands cores or silently caps achievable throughput well below the rated link. On GPU nodes the cost is second-order and larger: cores spent in softirq are cores not feeding GPUs, and idle GPUs bill at full rate.

## Real-world use cases

- **Google, BBR (ACM Queue, Vol. 14 No. 5, Sept–Oct 2016).** The paper the Linux implementation cites in its own header. Substance used here: the model — `BtlBw` as a windowed max of delivery rate, `RTprop` as a windowed min of RTT, the optimal operating point at `BtlBw × RTprop`, and the argument that loss is a poor congestion signal on both deep-buffered and lossy paths. The Linux constants in §9 are read from `net/ipv4/tcp_bbr.c` directly rather than from the paper, because the implementation has moved.
- **The bufferbloat work behind CoDel and fq_codel.** Substance used here: the switch from measuring queue *length* to measuring queue *sojourn time*, the `target`/`interval` pair (5 ms / 100 ms), and per-flow hashing with DRR so a bulk flow cannot delay an interactive one. Verified against the `tc-fq_codel(8)` man page in iproute2, which is also where every default in the table comes from.
- **HyStart++ in CUBIC.** Substance used here: exiting slow start on a *delay* signal rather than waiting for the loss that slow start's doubling guarantees. Without it, every new connection on a deep-buffered path overshoots by roughly one full buffer and then pays a fast-retransmit cycle. The constants (`hystart_low_window = 16`, `hystart_ack_delta_us = 2000`, delay threshold clamped to 4–16 ms, 8 samples minimum) are read from `net/ipv4/tcp_cubic.c`.
- **BIG TCP.** Substance used here: the 64 KB GSO/GRO ceiling was a real throughput limit at 100 Gb/s+ because per-skb cost dominates; current kernels raise it to `8 × 65535` bytes and expose it per device. This is why the "64 KB super-packet" folklore about `tcpdump` is now out of date in both directions.

## Worked example

**Scenario.** A training cluster's checkpoint uploader moves 500 GB from `us-east` to `eu-west` nightly. The path is provisioned at 25 Gb/s and measured RTT is 90 ms. The job takes 45 minutes; the team has opened a ticket asking for more bandwidth. Predict the rate before touching anything, then prove where the ceiling is.

**Step 1 — predict from arithmetic.**

```
   Observed: 500 GB in 45 min
             = 500e9 B / 2700 s
             = 185 MB/s
             = 1.48 Gbit/s        ← 5.9 % of the 25 Gb/s link

   BDP for the link:   25e9 / 8 × 0.090 = 281 MB
   Window implied by the observed rate:  185e6 × 0.090 = 16.7 MB

   16.7 MB is suspiciously close to 16 MiB.  Hypothesis: the receive
   window is clamped at tcp_rmem[2] = 16 MiB, and this is a WINDOW
   problem, not a bandwidth problem.  Predicted diagnosis before any
   command is run.
```

**Step 2 — confirm on the live socket.**

```bash
$ ss -tie dst 10.60.4.9
State  Recv-Q Send-Q  Local Address:Port   Peer Address:Port
ESTAB  0      2170624 10.20.1.44:51234     10.60.4.9:443
     cubic wscale:7,7 rto:296 rtt:90.4/1.2 mss:1448 pmtu:1500
     rcvmss:536 advmss:1448 cwnd:11520 ssthresh:9800 bytes_sent:41231884288
     bytes_acked:41229713664 bytes_retrans:1448 segs_out:28480031
     data_segs_out:28480028 send 1476Mbps lastsnd:4 lastrcv:82112
     pacing_rate 1772Mbps delivery_rate 1471Mbps busy:2699912ms
     rcv_space:14480 notsent:2168 minrtt:89.9
```

Read it line by line:

- `rtt:90.4/1.2` — smoothed RTT 90.4 ms, variance 1.2 ms. Stable path.
- `cwnd:11520` segments × `mss:1448` = **16.68 MB**. That matches the predicted window exactly.
- `bytes_retrans:1448` out of 41 GB sent — **one segment retransmitted in the whole transfer.** Loss is not the ceiling. If CUBIC were loss-limited you would see `bytes_retrans` in the tens of megabytes and `ssthresh` far below `cwnd`.
- `send 1476Mbps` and `delivery_rate 1471Mbps` agree — the sender is not being throttled locally.
- `wscale:7,7` — window scaling is negotiated in both directions with factor 2⁷ = 128, so the peer *can* advertise up to 65535 × 128 = 8.4 MB… **and this is the second finding.** A wscale of 7 was chosen at SYN time from the then-current `rmem`; it caps the advertised window at 8.4 MB regardless of later tuning, because the scale factor cannot be renegotiated mid-connection. Raising the sysctl affects only *new* connections.

**Step 3 — confirm the host is not the problem.**

```bash
$ ethtool -S eth0 | grep -Ei 'miss|no_buf|drop' | grep -v ': 0$'
     (no output — no hardware drops)

$ awk '{print $1, $2, $3}' /proc/net/softnet_stat | head -4
0004ba7d 00000000 0000010f
0004c112 00000000 00000121
     (time_squeeze even across CPUs, low and stable — not softirq-bound)

$ tc -s qdisc show dev eth0 | head -3
qdisc fq 8001: root refcnt 2 limit 10000p flow_limit 100p buckets 1024 ...
 Sent 41231884288 bytes 28480031 pkt (dropped 0, overlimits 0 requeues 43)
     (no local queue drops)
```

Every host-side stage is clean. The ceiling is the window, as predicted.

**Step 4 — the fix, and what each part buys.**

```bash
# Sender and receiver, persisted in /etc/sysctl.d/:
net.core.rmem_max            = 536870912     # 512 MB — above the 281 MB BDP
net.core.wmem_max            = 536870912
net.ipv4.tcp_rmem            = 4096 131072 536870912
net.ipv4.tcp_wmem            = 4096 16384 536870912
net.ipv4.tcp_window_scaling  = 1             # already default, assert it
net.ipv4.tcp_moderate_rcvbuf = 1             # keep autotuning ON
net.core.default_qdisc       = fq            # pacing, and BBR needs it
```

Then **restart the transfer process** — existing connections keep their negotiated `wscale` and will not benefit.

```
   New ceiling:  min(cwnd, rwnd) can now reach the 281 MB BDP.
   Predicted rate = 281e6 B / 0.090 s = 3.12 GB/s = 25 Gbit/s (line rate)
   Predicted duration = 500e9 / 3.12e9 = 160 s ≈ 2.7 min
   (from 45 min — a 16× improvement with no additional bandwidth purchased)

   Caveat to state up front: reaching line rate now depends on the path
   genuinely being loss-free at 25 Gb/s.  If bytes_retrans starts climbing
   after the change, you have moved the bottleneck from the window to the
   congestion controller, and THAT is when `tcp_congestion_control = bbr`
   becomes the next experiment — not before.
```

**Step 5 — the memory bill, stated honestly.** `tcp_rmem[2] = 512 MB` is a *ceiling*, not an allocation; autotuning only grows a socket's buffer if the flow actually needs it. But the ceiling must be sized against your connection count: 100 concurrent bulk flows each autotuning to 281 MB is 28 GB of kernel memory. On a machine with a few hundred long-fat flows, cap `tcp_rmem[2]` at a value you can afford, or restrict the large buffers to the specific sockets that need them via `SO_RCVBUF` on those sockets only (accepting that this disables autotuning for them).

## Practice

<feeds [packet-path teardown + debug runbook](../practice/packet-path-and-debug/README.md)>

Build the "flow is slow, link is not full" branch of the runbook, and make every step produce evidence rather than an opinion.

**Task A — the prediction discipline.** On any two hosts with a shapeable path between them, run a bulk transfer and, *before* measuring, write down the predicted rate from `ss -tie`'s `cwnd` and `rtt` using `throughput = cwnd × mss / RTT`. Then measure. Record the ratio. Repeat until prediction and measurement agree within ~20 % — when they don't, you have mislocated the bottleneck, and finding out why is the actual exercise.

**Task B — reproduce the CUBIC/BBR delta.**

```bash
# Shape a long, slightly lossy path.
tc qdisc replace dev eth0 root netem delay 100ms loss 0.5%

sysctl -w net.ipv4.tcp_congestion_control=cubic
iperf3 -c <peer> -t 60 -J > cubic.json

sysctl -w net.ipv4.tcp_congestion_control=bbr
sysctl -w net.core.default_qdisc=fq     # BBR wants fq for pacing
iperf3 -c <peer> -t 60 -J > bbr.json

tc qdisc del dev eth0 root              # RESTORE
```

Produce a table with, per algorithm: mean throughput, retransmit count, and the `cwnd × mss / RTT` prediction. Explain the gap: CUBIC's `beta ≈ 0.7` cut on each of the ~0.5 %-probability loss events versus BBR's model, which treats non-congestive loss as noise.

**Task C — catch a BBR PROBE_RTT.** With BBR active on a long-lived flow, sample `ss -tie` once per second for two minutes and plot `cwnd` and `pacing_rate`. Annotate the dips: PROBE_RTT cuts inflight to 4 packets for ≥ 200 ms once no new `min_rtt` sample has been seen for 10 s. Mark on the trace where the 8-phase PROBE_BW gain cycle's 1.25 and 0.75 phases show up as small oscillations around the cruise rate.

**Task D — the steering exercise.** Generate small-packet load, then find and fix a single-core softirq bottleneck: read `/proc/interrupts` and `/proc/softirqs` to identify the hot core, count `time_squeeze` in **column 3** of `/proc/net/softnet_stat`, then apply IRQ affinity spreading (or RPS on `/sys/class/net/eth0/queues/rx-*/rps_cpus`) and show the counters flatten.

**Acceptance criteria**

- [ ] A predicted-vs-measured throughput table with the `cwnd/RTT` arithmetic shown, agreeing within a small factor.
- [ ] A CUBIC-vs-BBR table on an identical netem-shaped path, with retransmit counts, and a written explanation of the delta in terms of each algorithm's control signal.
- [ ] An annotated BBR trace with at least one PROBE_RTT dip identified by its 10 s / 200 s / 4-packet signature.
- [ ] A before/after `/proc/net/softnet_stat` column-3 comparison proving a steering fix worked.
- [ ] A runbook entry ordering the checks R1 → R3 → R5 → T2 → T4, each with its exact command and the counter name to read.

## Common pitfalls

- **"More bandwidth fixes it."** Compute `cwnd × mss / RTT` and the BDP against the actual socket-buffer ceiling first. A fatter link does nothing when the receive window is the binding constraint — and it is, on every long-RTT path with default buffers.
- **Raising `tcp_rmem` and expecting existing connections to speed up.** The window *scale factor* is negotiated once, in the SYN, from the buffer size at that moment, and cannot change mid-connection. Sysctl changes apply to new connections only. Restart the client.
- **Reading `/proc/net/softnet_stat` column 2 as "squeezed."** Column 2 is `dropped`; `time_squeeze` is column 3. The field order is fixed in `net/core/net-procfs.c`.
- **Quoting `netdev_budget = 64`.** 64 is the per-NAPI `weight` (`dev_rx_weight`). `netdev_budget` is **300**, with a companion `netdev_budget_usecs` of `2×10⁶/HZ` (2000 µs at HZ=1000). Raising the wrong one has no effect and wastes an incident.
- **"`tcpdump` shows the wire."** GRO/GSO and checksum offload sit below the capture hook, so captures show packets and checksums that never existed as such. Disable offloads for the capture, and **turn them back on** — leaving GRO off at 100 Gb/s is itself an outage.
- **"BBR is strictly better than CUBIC."** BBRv1 has no loss response and can hold its share while CUBIC neighbours repeatedly back off. Know who else is on your bottleneck before changing a fleet default.
- **"RSS alone balances load across cores."** RSS spreads packets across NIC *queues* by flow hash. If IRQ affinity funnels those queues onto one core, you get eight busy queues and one saturated core. Check `/proc/interrupts` before concluding the NIC is fine.
- **Setting `SO_SNDBUF`/`SO_RCVBUF` in application code "to be safe."** It disables autotuning for that socket permanently, pinning it to whatever value seemed reasonable to a developer on a 1 ms LAN. On a 90 ms path it becomes a hard throughput cap that no sysctl can lift.

## Self-check

**1. A 25 Gb/s path between two regions has a 90 ms RTT. Your bulk copy tops out near 1.5 Gb/s and `ss` shows near-zero retransmits. Where is the ceiling and what do you tune?**

It is the window-versus-BDP ceiling, not loss — near-zero `bytes_retrans` rules out a loss-limited congestion controller. BDP = 25e9 / 8 × 0.090 = 281 MB. The observed 1.5 Gb/s implies an in-flight window of 1.5e9/8 × 0.090 ≈ 16.9 MB, which is `tcp_rmem[2] = 16 MiB`. Raise `net.core.rmem_max`/`wmem_max` and `tcp_rmem`/`tcp_wmem` maxima above the BDP on both ends, keep `tcp_moderate_rcvbuf = 1` so autotuning still works, confirm `tcp_window_scaling = 1`, and **restart the connection** — the window scale factor was fixed at SYN time and cannot grow. Budget the memory: the new ceiling times your concurrent flow count.

**2. `tcpdump` on the receiver shows 300 KB TCP segments and flags many checksums as "incorrect," but the application data is intact. What is happening?**

Offloads. GRO coalesced many wire frames into one large `sk_buff` before the capture hook, and RX checksum offload means the checksum was validated in hardware below the capture point, so the software checksum shown is meaningless. The 300 KB size is not corruption either: current kernels set `GRO_MAX_SIZE = 8 × 65535 ≈ 512 KB` (BIG TCP), so segments above the old 64 KB folklore limit are expected on a modern high-bandwidth host. For wire-accurate frames, `ethtool -K eth0 gro off lro off rx off`, capture, then restore — GRO off at line rate will saturate a core.

**3. Why can CUBIC produce high latency-under-load even at full throughput, and what is the fix stack?**

CUBIC's congestion signal is loss (or ECN marks). On a deep-buffered bottleneck, nothing signals congestion until the buffer is *full*, so CUBIC grows cwnd until it fills that buffer — adding the buffer's full drain time to every packet's RTT, including packets from other flows. Throughput is fine; latency is wrecked. That is bufferbloat. Fix stack, in order: (1) AQM at the actual bottleneck — `fq_codel` with `target 5ms interval 100ms`, which measures sojourn time rather than queue length and drops/marks once the *minimum* delay over the interval stays above target; (2) a rate-model congestion control (BBR) that targets `BtlBw × RTprop` rather than "until it breaks"; (3) `tcp_notsent_lowat` so the application stops handing the kernel data it cannot send soon.

**4. A `ss -tie` trace of a long-lived BBR flow shows `cwnd` and `pacing_rate` collapse for a couple of hundred milliseconds roughly every 10 s, with no retransmits. Is this a bug?**

No — it is PROBE_RTT, and the signature is precise. BBR's `min_rtt` estimate is a windowed minimum over `bbr_min_rtt_win_sec = 10` seconds. If no RTT sample at or below the current estimate arrives in that window, the estimate may be stale (the path may be permanently queued), so BBR cuts inflight to `bbr_cwnd_min_target = 4` packets for at least `bbr_probe_rtt_mode_ms = 200` ms to drain the queue and take a clean reading, then returns to PROBE_BW. Note the trigger is "no new minimum in 10 s," not "every 10 s" — a flow that idles produces clean samples for free and may never enter PROBE_RTT.

**5. Two engineers disagree: one wants to raise `SO_SNDBUF` to fix throughput on a long-fat link, the other wants to lower `tcp_notsent_lowat` to fix latency from a chatty app. Are they fighting over the same knob?**

No — they are orthogonal. `SO_SNDBUF` bounds *total* buffered bytes (sent-but-unacked plus unsent) and is therefore the ceiling that lets cwnd grow toward the BDP; it is a throughput knob. `tcp_notsent_lowat` (default `UINT_MAX`, i.e. off) bounds only the *unsent* portion and controls when `epoll` reports the socket writable; it is a local-queueing-latency knob and does not touch cwnd. Both changes can coexist. The one genuine interaction is a trap: calling `setsockopt(SO_SNDBUF)` at all **disables send-buffer autotuning** for that socket, so "raising" it can pin the socket below what autotuning would have reached on its own.

**6. Your node shows 100 % `si` on CPU 0 and idle on the other 31, with a 100 Gb/s NIC that has 32 RX queues configured. Name the two-command diagnosis and the two possible fixes.**

Diagnosis: `grep mlx5 /proc/interrupts` to see whether all queue interrupts are landing on CPU 0 (an IRQ-affinity problem — queues exist, the mapping doesn't), and `awk '{print $3}' /proc/net/softnet_stat` to see whether CPU 0's `time_squeeze` is running away while the others are flat (confirming that core is giving up with work pending). Fix A: spread IRQ affinity across cores (`/proc/irq/<n>/smp_affinity_list`, or let `irqbalance` do it), so RSS's queue spreading actually reaches multiple cores. Fix B: if the hardware or driver cannot spread — or the flow set hashes poorly, e.g. a single tunnel between two IPs — enable RPS on the queues (`/sys/class/net/eth0/queues/rx-*/rps_cpus`) to re-hash in software, and RFS (`net.core.rps_sock_flow_entries`) to land packets on the core where the consuming application actually runs.

**7. Why does the *first* 200 MB of a transfer on a 90 ms path never reach line rate, no matter how you tune the buffers?**

Slow start. cwnd begins at `TCP_INIT_CWND = 10` segments and doubles once per RTT, so reaching a 281 MB BDP (≈194,000 segments at MSS 1448) takes `log₂(19,400) ≈ 15` round trips = 1.35 s during which the average rate is roughly half the final rate. Buffer tuning raises the *ceiling*; it cannot accelerate the ramp. The practical consequences: short transfers are dominated by the ramp, so connection reuse (HTTP keep-alive, connection pools, HTTP/2) is worth more than bandwidth to them; and `tcp_slow_start_after_idle = 1` (RFC 2861, the default) makes an idle-then-bursty connection re-pay the ramp every time, which is a defensible thing to disable on a trusted internal fabric.

## Connections & what's next

This lesson is the spine the rest of the module hangs off. Lesson 05 (Kubernetes networking) revisits this pipeline with a Service VIP DNAT and an overlay encapsulation inserted into it, and the netfilter/conntrack box drawn here as a single label is the one `modules/01b-linux-internals/lessons/07-networking-datapath-conntrack.md` opens up. Lesson 03 (load balancing) reuses `throughput = cwnd/RTT` and the slow-start ramp to explain why a cold backend must be ramped rather than switched into full traffic. Lesson 07 (GPU/RDMA) contrasts this entire pipeline with the path RDMA deliberately bypasses — the value of having drawn T1–T5 and R1–R5 precisely is that "RDMA is faster" becomes a specific list of deleted stages. Lesson 08 turns §5's counter table into a symptom→cause decision tree with `hubble` and `pwru` added.

The very next lesson picks up one step earlier in time: before any of this machinery runs, the client had to turn a name into an address, and DNS's failure modes look, from the application's point of view, exactly like the TCP problems above.

## References & further reading

**Primary sources — read directly for this lesson**

1. `net/ipv4/tcp_cubic.c`, Linux master (https://github.com/torvalds/linux/blob/master/net/ipv4/tcp_cubic.c) — source of every CUBIC constant in §8: `beta = 717/1024`, `bic_scale = 41` (→ `cube_rtt_scale = 410`, effective C ≈ 0.4), `fast_convergence = 1`, `tcp_friendliness = 1`, and the HyStart++ parameters (`hystart_low_window = 16`, `hystart_ack_delta_us = 2000`, `HYSTART_MIN_SAMPLES = 8`, delay threshold clamped to 4–16 ms). **Correction to the previous version of this lesson:** it implied a halving on loss; the Linux multiplicative-decrease factor is ≈ 0.7.
2. `net/ipv4/tcp_bbr.c`, Linux master (https://github.com/torvalds/linux/blob/master/net/ipv4/tcp_bbr.c) — the algorithm summary in the file header, the four-mode state machine, and every gain in §9: `bbr_high_gain = 2/ln2 ≈ 2.885`, `bbr_drain_gain = 1/2.885`, `bbr_cwnd_gain = 2`, `bbr_pacing_gain[] = {5/4, 3/4, 1×6}`, `bbr_min_rtt_win_sec = 10`, `bbr_probe_rtt_mode_ms = 200`, `bbr_cwnd_min_target = 4`, `bbr_full_bw_thresh = 5/4`, `bbr_full_bw_cnt = 3`, `bbr_pacing_margin_percent = 1`, and the long-term policer-detection parameters.
3. `net/core/hotdata.c` and `net/core/dev.c`, Linux master — the NAPI budgets in §3: `netdev_budget = 300`, `netdev_budget_usecs = 2×10⁶/HZ`, `dev_rx_weight = 64`, `max_backlog = 1000`. **Correction:** the previous version of this lesson gave `netdev_budget` as 64; that figure is the per-NAPI weight.
4. `net/core/net-procfs.c`, Linux master — the exact field order of `/proc/net/softnet_stat` used in §5. **Correction:** the previous version identified `time_squeeze` as column 2; it is column 3, and column 2 is `dropped`.
5. `include/net/tcp.h`, Linux master — `TCP_INIT_CWND = 10`, `TCP_RTO_MIN = HZ/5` (200 ms), `TCP_TIMEOUT_INIT = 1 s` (citing RFC 6298), `TCP_RTO_MAX = 120 s`, `TCP_DELACK_MAX = HZ/5`, `MAX_TCP_WINDOW = 32767`.
6. `include/linux/netdevice.h`, Linux master — `GSO_LEGACY_MAX_SIZE = 65536`, `GSO_MAX_SEGS = 65535`, `GSO_MAX_SIZE = GRO_MAX_SIZE = 8 × 65535` (the BIG TCP ceilings used in §12).
7. `net/ipv4/tcp_output.c`, Linux master — `tcp_select_initial_window()` and `TCP_MAX_WSCALE = 14`, giving the ~1 GiB single-connection window ceiling in §6.
8. `Documentation/networking/ip-sysctl.rst`, Linux master (https://github.com/torvalds/linux/blob/master/Documentation/networking/ip-sysctl.rst) — documented defaults for `tcp_rmem` (4 K / 128 K / 128 K–32 M), `tcp_wmem` (4 K / 16 K / 64 K–4 M), `tcp_moderate_rcvbuf = 1`, `tcp_window_scaling = 1`, `tcp_notsent_lowat = UINT_MAX`, `tcp_slow_start_after_idle = 1`, `somaxconn = 4096` (128 before 5.4), `tcp_syn_retries = 6` (≈131 s), `tcp_retries2 = 15` (≈924.6 s), `tcp_mtu_probing` semantics.
9. `Documentation/admin-guide/sysctl/net.rst`, Linux master — the prose definitions of `netdev_budget`, `netdev_budget_usecs`, and `netdev_max_backlog`.
10. `Documentation/networking/scaling.rst`, Linux master — RSS, RPS, RFS, accelerated RFS, and XPS: what each decides and where it is configured (§13).
11. `Documentation/networking/napi.rst`, Linux master — the NAPI contract, budget semantics, and the interrupt→poll transition described in §3.
12. `man 8 tc-fq_codel`, iproute2 (https://github.com/iproute2/iproute2/blob/main/man/man8/tc-fq_codel.8) — every fq_codel default in §10: `target 5 ms`, `interval 100 ms`, `limit 10240p`, `memory_limit 32 MB`, `flows 1024`, `quantum 1514`, `ecn` on, `drop_batch 64`, and the meaning of the `new_flow_count` / `ecn_mark` / `drop_overlimit` statistics.
13. `net/ipv4/tcp_recovery.c` and `net/ipv4/tcp_input.c`, Linux master — the loss-recovery machinery in §7 and the RFCs the implementation itself cites: RFC 8985 (RACK-TLP), RFC 6937 (PRR), RFC 5681 (congestion control), RFC 6298 (RTO computation), RFC 7323 (window scaling and timestamps), RFC 2018 (SACK), RFC 3168 (ECN), RFC 2861 (congestion-window validation after idle).

**Sources named but not fetched in this pass — do not treat the wording as verified**

14. RFC 9438 (CUBIC), RFC 9293 (TCP), RFC 8289 (CoDel), RFC 8290 (FQ-CoDel), RFC 6928 (initial window of 10). `www.rfc-editor.org`, `ietf.org`, and `datatracker.ietf.org` are blocked by this environment's egress policy, so these documents were **not** read for this rewrite. Every numeric value attributed to them above was instead read from the Linux implementation or the iproute2 man page, and the RFC *numbers* were cross-checked against the RFC references in the kernel's own source comments. Read the RFCs directly for the standards-track wording.
15. Cardwell, Cheng, Gunn, Hassas Yeganeh, Jacobson, "BBR: Congestion-Based Congestion Control," ACM Queue 14(5), 2016 — cited by name in `tcp_bbr.c`'s header; the paper itself was not fetched here. The model description in §9 is taken from the kernel file's own summary.

**Deeper dives, optional**

16. LWN.net, "BIG TCP" (https://lwn.net/Articles/894280/) — the motivation for raising the 64 KB GSO/GRO ceiling: at 100 Gb/s+, per-skb cost rather than per-byte cost dominates. Reachability from this environment was not verified; the ceilings quoted in §12 come from `include/linux/netdevice.h` directly.
