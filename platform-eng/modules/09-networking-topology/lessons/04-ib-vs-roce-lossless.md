---
lesson: "09.4"
title: "InfiniBand vs RoCEv2: engineering losslessness, and when each wins"
module: "09"
concept: "InfiniBand vs RoCEv2: engineering losslessness, and when each wins"
status: not-started
est_time: "8h"
prev: "03-rdma-fundamentals.md"
next: "05-gpudirect-and-sharp.md"
artifacts: []
sources: 14
---
# 09.4 · InfiniBand vs RoCEv2: engineering losslessness, and when each wins

> **Concept.** InfiniBand is lossless *by construction* (credit-based hardware flow control); RoCEv2 is RDMA bolted onto Ethernet that you must *engineer* into losslessness with PFC + ECN + DCQCN — and the senior skill is naming which trade wins for a given cluster, not chanting "IB is faster." A fourth path exists too: Meta's production answer to "RoCE congestion control is hard" wasn't better DCQCN tuning, it was moving the job to a different layer entirely.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

09.3 established RDMA's mechanism — kernel bypass, zero-copy, one-sided WRITEs over a hardware-reliable RC queue pair — and drew one deliberate boundary: RC guarantees that packets which *arrive* are delivered once, in order and acknowledged, which is **not** the claim that the fabric never drops them. It also showed how catastrophic the gap can be. Microsoft's production RoCEv2 NICs recovered from loss with **go-back-0**, and a deterministic 1-in-256 drop rate on 4 MB messages produced *zero* application goodput while the link ran at line rate.

This lesson is entirely about closing that gap. It shows how InfiniBand makes congestion drops impossible at the link layer, how RoCEv2 manufactures the same property on Ethernet out of PFC, ECN and DCQCN, exactly what each mechanism does at the buffer level, how each one fails, and how to choose between the two fabrics for a real procurement with numbers attached.

It is the module's highest-yield lesson because it is where the interview stops being about vocabulary and starts being about judgement: not "what is PFC" but "given this cluster, do I want InfiniBand's determinism or Ethernet's economics, what does the lossless configuration actually consist of, and can I defend the buffer arithmetic behind it."

## Why this matters

The stakes are literal money and literal uptime, and both failure directions are expensive.

Get losslessness wrong on RoCE and you do not get "a bit slower." You get **head-of-line blocking**, where one congested link pauses an entire traffic class hop by hop and stalls flows that had nothing to do with the congestion; you get **PFC storms**, where a single malfunctioning NIC can pause an entire fabric — a cascade Microsoft observed in production and had to build two independent watchdogs to contain; or you get **PFC deadlock**, a cyclic pause dependency that wedges a fabric region and, in Microsoft's account, *does not clear even if you restart every server*. Each of those converts directly into idle GPU-hours across every job on the affected fabric, because collectives are barriers.

Get vendor lock wrong in the other direction and you have signed a multi-year, single-source switch-and-optics bill for a fabric you cannot second-source, on a component class whose price you no longer negotiate.

The senior answer is not a preference; it is a decision procedure. **"Depends. InfiniBand buys determinism and in-network reduction, and costs lock-in and a premium optics market. RoCE reuses my Ethernet control plane and a competitive optics market, and costs a PFC/ECN qualification project whose buffer arithmetic I can show you. Here is which one this cluster should buy, and why."** Everything in this lesson exists to let you say that sentence and defend every clause.

## What's new here (calibration)

You know 01b's "loss is normal, TCP recovers" model — this lesson exists because RDMA fabrics invert it, and you built the RDMA half of that inversion in 09.3. You know 09.2's Clos tiers and its ECMP-entropy argument, and 08's collectives. Genuinely new here:

- The **real InfiniBand rate table** — per-lane signalling, line coding, effective data rate, and 4x aggregate for SDR through XDR — plus the subtlety that the "rate" a Linux tool prints means a *different thing* for FDR than for EDR and later.
- **Credit-based link-level flow control** as a mechanism: virtual lanes, credit blocks, the flow-control packet's fields, and the counter that tells you it is throttling you.
- **InfiniBand's own congestion control** (FECN/BECN, congestion control table, CCTI), with its real configuration knobs — the symmetric counterpart to DCQCN that most people do not know exists.
- The **DCB suite** as a table of actual standards and actual Linux/driver knobs: PFC (802.1Qbb), ETS (802.1Qaz), DCBX, and where ECN fits (it is IP, not DCB).
- **PFC at the buffer level**: XOFF/XON thresholds, headroom, the pause frame's exact format, and pause-quanta arithmetic at 100G/400G/800G.
- **PFC headroom sizing as worked arithmetic**, using the formula the mlx5 driver actually implements — and the punchline that at 400G the numbers exceed the buffer of a shallow-buffer switch.
- The **ECN/DCQCN control loop** with the real parameter names exposed by the driver, and the ordering rule (ECN leads, PFC trails) stated mechanically rather than as a slogan.
- Real annotated `ethtool -S` PFC/ECN counters and the `dcb`/`mlnx_qos` configuration surface.

## Core concepts

### 1. Why a lossless fabric, quantified

09.3 gave the qualitative reason. Here is the arithmetic that makes it a design constraint rather than a preference.

A reliable-connected queue pair detects loss in one of two ways. If packets keep arriving, the receiver notices a **sequence-number gap** and returns a NAK almost immediately — fast detection, coarse recovery. If nothing arrives at all, the sender falls back to the **local ACK timeout**, which at NCCL's default `timeout = 20` is `4.096 µs × 2^20 ≈ 4.3 s` before the first retry (09.3 §6). Both paths are bad, for different reasons.

```
   COST OF ONE DROPPED PACKET, 400 Gb/s link, 10 µs fabric RTT

   bytes in flight = rate × RTT
                   = 400e9 bits/s × 10e-6 s ÷ 8
                   = 500,000 bytes  ≈ 500 KB

   go-back-N : sender resumes from the lost packet, discarding and
               re-sending everything after it
               → up to ~500 KB re-sent per drop, plus the round trip
               → at a 0.1% drop rate and 4 KB packets, that is one
                 discard event per 4 MB — i.e. a permanent ~12% tax,
                 and much worse at higher loss

   go-back-0 : sender restarts the ENTIRE message
               → a 4 MB message = ~1,000 packets at 4 KB
               → a deterministic 1-in-256 drop rate guarantees a loss
                 inside every restart window → goodput = 0
                 (measured, Microsoft SIGCOMM 2016)

   timeout path (nothing arrives at all):
               4.096 µs × 2^20 ≈ 4.3 s per attempt × up to 7 retries
               → ~30 s before the QP fails.  Every rank in the
                 collective waits, because a collective is a barrier.
```

**That is why the fabric must not drop packets, and it is a much better answer than "RDMA is sensitive to loss."** The design goal is not "recover well"; it is *drive the congestion-drop probability toward zero*, because the recovery you are avoiding is a cliff and not a slope.

There are exactly two ways to build a fabric that does not drop under congestion, and they are architecturally different:

- **InfiniBand:** losslessness is *intrinsic*. A purpose-built link layer with credit-based flow control — a sender may transmit only against buffer credits the receiver has advertised, so congestion means "wait for credits," never "overflow and drop." There is no configuration to get wrong.
- **RoCEv2:** RDMA runs as a **UDP/IP payload over ordinary routed Ethernet** (destination port 4791; IP-routability is the "v2" upgrade over RoCEv1's raw layer-2 framing). Ethernet drops under congestion by design, so you **manufacture** losslessness: PFC to convert overflow into backpressure, ECN to signal congestion end to end, DCQCN to act on that signal at the source. You reuse the Ethernet control plane you already run, and you inherit all of its failure modes.

The contrast to hold: **InfiniBand deletes the kernel datapath *and* hands you a lossless substrate; RoCE deletes the datapath and hands you the substrate as a tuning project.**

### 2. The InfiniBand rate table — what a generation name actually means

Every IB generation name encodes a per-lane rate, and ports are built from lanes (1x, 4x, 8x, 12x — 4x is universal for host adapters). This table is the one to know cold, because every bandwidth claim in a procurement conversation traces back to it.

| Gen | Per-lane signalling | Modulation | Line code | Per-lane **data** rate | **4x** port | Era / silicon |
|---|---|---|---|---|---|---|
| SDR | 2.5 GBd | NRZ | 8b/10b (80%) | 2 Gb/s | 8 Gb/s (marketed 10) | first IB |
| DDR | 5 GBd | NRZ | 8b/10b | 4 Gb/s | 16 Gb/s (marketed 20) | |
| QDR | 10 GBd | NRZ | 8b/10b | 8 Gb/s | 32 Gb/s (marketed 40) | ConnectX-2 era |
| FDR10 | 10.3125 GBd | NRZ | 64b/66b (96.97%) | 10 Gb/s | 40 Gb/s | interim |
| FDR | 14.0625 GBd | NRZ | 64b/66b | 13.64 Gb/s | 54.5 Gb/s (marketed 56) | ConnectX-3 |
| EDR | 25.78125 GBd | NRZ | 64b/66b | 25 Gb/s | **100 Gb/s** | ConnectX-4/5 |
| HDR | 53.125 Gb/s per lane | PAM4 | 256b/257b + RS-FEC | 50 Gb/s | **200 Gb/s** | Quantum-1, ConnectX-6 |
| NDR | 106.25 Gb/s per lane | PAM4 | 256b/257b + RS-FEC | 100 Gb/s | **400 Gb/s** | Quantum-2, ConnectX-7 |
| XDR | 212.5 Gb/s per lane | PAM4 | 256b/257b + RS-FEC | 200 Gb/s | **800 Gb/s** | Quantum-X800, ConnectX-8 |

**Provenance, stated precisely.** The per-lane *rate* values and the `rate = per-lane rate × width` rule are verifiable in the Linux kernel's IB core: `drivers/infiniband/core/verbs.c` carries a speed table with SDR = 2.5, DDR = 5, QDR = 10, FDR10 = 10, FDR = 14, EDR = 25, HDR = 50, NDR = 100, XDR = 200 (in units of 0.1 Gb/s), and `ib_port_attr_to_speed_info()` multiplies by the port width before `ibstat` prints it. The modulation and line-coding columns follow the PHY families these generations were built on (NRZ with 8b/10b through QDR, NRZ with 64b/66b for FDR/EDR, PAM4 with 256b/257b RS-FEC from HDR onward); the IBTA specification itself was not reachable from this environment, so treat those two columns as the standard engineering account rather than as text checked against the spec in this pass.

**The subtlety worth carrying, because it catches people out.** The number a tool prints is the *marketed* per-lane figure, and the marketing convention changed. For FDR the marketed 14 Gb/s per lane is the **signalling** rate — 4x FDR is sold as "56 Gb/s" but 64b/66b coding means it carries about **54.5 Gb/s** of data. From EDR onward the marketed number is the **post-coding data rate**: 4x EDR really is 100 Gb/s of payload capacity, and 4x NDR really is 400 Gb/s. So `Rate: 56` and `Rate: 400` are both "correct" but they are answering slightly different questions, and only the second one can be divided by 8 to get usable GB/s without further correction.

Three practical consequences:

- **`Rate: 400` on `ibstat` means 4 lanes × 100 Gb/s.** A port reading `Rate: 200` on hardware you bought as NDR trained at half width (2 lanes) or fell back a generation — either is a physical-layer fault to chase, not a configuration choice.
- **Divide by 8 for GB/s, then expect 92–97% of it.** 400 Gb/s = 50 GB/s theoretical; a healthy `ib_write_bw` lands near 48.6 GB/s at a 4 KB MTU (09.3 §10). If your ceiling is much lower, the MTU or the PCIe link is the suspect.
- **Generation transitions move the bottleneck.** XDR's 800 Gb/s (100 GB/s per direction) exceeds what a PCIe Gen5 x16 host link can supply (63 GB/s), which is why ConnectX-8 is a Gen6 device (09.1 §1). Rate tables and host-bus tables must be read together.

### 3. InfiniBand's losslessness: credit-based link-level flow control

The mechanism is simple to state and worth understanding exactly, because it is the thing RoCE spends three protocols approximating.

Every InfiniBand link is divided into **virtual lanes (VLs)** — independent buffer pools multiplexed onto the same physical wire. A port supports up to 15 data VLs (VL0–VL14) plus **VL15**, which is reserved for subnet-management traffic and is deliberately *not* flow-controlled, so management packets can always move even when data lanes are jammed. Traffic is assigned a **service level (SL)** and each hop maps SL to a VL via an SL-to-VL table the subnet manager programs.

Flow control runs **per VL, per link**, and it is credit-based rather than pause-based:

```
   RECEIVER                                        SENDER
   ────────                                        ──────
   has N free credit blocks in the VL's buffer
        │
        │   FLOW CONTROL PACKET (periodic, per VL)
        │   carries:
        │     ABR   — adjusted blocks received
        │     FCCL  — flow control credit limit: the highest
        │             block count the sender is authorised to reach
        └──────────────────────────────────────────────▶
                                                   maintains FCTBS
                                                   (total blocks sent)

                            SENDER MAY TRANSMIT A PACKET IFF
                              FCTBS + packet_size_in_blocks  ≤  FCCL

   ┌─────────────────────────────────────────────────────────────┐
   │ receiver's buffer fills   →   FCCL stops advancing          │
   │ sender reaches FCCL       →   sender simply STOPS SENDING   │
   │ receiver drains           →   FCCL advances                 │
   │ sender resumes                                              │
   │                                                             │
   │ AT NO POINT DOES A PACKET ARRIVE WITH NOWHERE TO GO.        │
   │ Backpressure is continuous and per-VL, not a discrete       │
   │ "stop everything on this class" event.                      │
   └─────────────────────────────────────────────────────────────┘
```

Three properties fall out, and each is a genuine advantage over pause-based schemes:

1. **There is no threshold to tune.** Credits *are* the buffer occupancy. There is no XOFF level to set too high, no headroom to under-provision, no reaction-time window during which packets keep arriving with nowhere to go. The class of misconfiguration that dominates RoCE operations simply does not exist.
2. **Backpressure is proportional, not binary.** A sender slows as credits become scarce rather than stopping abruptly when a watermark trips, which produces gentler queue dynamics upstream.
3. **Isolation is per virtual lane.** Congestion on one VL does not stop another, so head-of-line blocking is bounded by how many VLs you actually use and how SLs map onto them.

The credit accounting is done in fixed-size **blocks** (the IBTA credit unit is a 64-byte block, and a VL's advertised credit limit bounds its receive buffer — commonly cited as up to 2,048 blocks, i.e. 128 KB per VL). Those two constants come from secondary summaries of the IBTA link-layer specification, which was not reachable from this environment; **the mechanism is the load-bearing part and is what you should reproduce in an interview**, not the constants.

**What you can observe.** Credit starvation has a name in the counters: **`PortXmitWait`**, readable with `perfquery` (it appears in the standard PortCounters and PortCountersExtended attribute layouts in rdma-core's `libibmad/fields.c`). It increments when a port has data ready to send but no credits to send it — i.e. it is the direct measure of "this link is being backpressured." Rising `PortXmitWait` on a link is the InfiniBand equivalent of rising PFC pause counters on Ethernet, and it is the first counter to look at when a fabric feels slow but shows no errors. The companion **`VL15Dropped`** counts management packets dropped on the un-flow-controlled lane, which is a distinct and much more alarming signal.

### 4. InfiniBand also has congestion control — and almost nobody knows its knobs

Link-level flow control stops drops; it does not stop *congestion spreading*, because backpressure still propagates hop by hop. So InfiniBand has an end-to-end congestion-control loop too, structurally similar to ECN/DCQCN, and knowing it exists is what lets you make the "IB is not magic, it just ships the tuning done" argument credibly.

The loop: a congested **switch** marks packets with a **FECN** (forward explicit congestion notification) bit; the destination reflects it as a **BECN** to the source; the source channel adapter raises a **congestion control table index (CCTI)**, and the CCT maps that index to an **injection rate delay** — a per-packet inter-packet gap that throttles the offending QP. Absence of further BECNs lets a timer decay the index back down.

The configuration surface is real and enumerable, and rdma-core ships the tools that write it (`ibccconfig`, `ibccquery`):

| Object | Parameters | What they do |
|---|---|---|
| **SwitchCongestionSetting** | `control_map`, `victim_mask`, `credit_mask`, `threshold`, `packet_size`, `cs_threshold`, `cs_return_delay`, `marking_rate` | when a switch port marks FECN: at what queue threshold, above what packet size, how aggressively (`marking_rate`), and which ports are treated as congestion victims versus sources |
| **CACongestionSetting** (per channel adapter) | `port_control`, `control_map`, `ccti_timer`, `ccti_increase`, `trigger_threshold`, `ccti_min` | how the endpoint reacts: how much the index jumps per BECN (`ccti_increase`), how fast it decays (`ccti_timer`), the floor (`ccti_min`) |
| **CongestionControlTable** | `ccti_limit`, then a list of CCT entries | the index-to-delay mapping itself — the actual rate curve |

Compare that with DCQCN's knob list in §11 and the symmetry is obvious: a marking rule at the congestion point, a notification back to the source, and a rate curve at the source. **InfiniBand's advantage is not that it lacks these knobs; it is that the lossless guarantee does not depend on them.** On IB, mistuned congestion control costs you throughput. On RoCE, mistuned PFC costs you the losslessness itself.

### 5. RoCEv2 on the wire: what is actually in the packet

To reason about DSCP mapping, ECN marking and ECMP entropy, you need the packet layout in your head.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │ ETHERNET  dst MAC | src MAC | [802.1Q VLAN tag: PCP(3b) VID]     │
  │                                     └── PCP is the priority PFC  │
  │                                         acts on, if VLAN-based   │
  ├──────────────────────────────────────────────────────────────────┤
  │ IPv4/IPv6  ... | DSCP(6b) | ECN(2b) | proto=UDP | src IP | dst IP │
  │                    │           └── 00 not-ECT, 10/01 ECT,        │
  │                    │               11 = CE "congestion           │
  │                    │               experienced" set by a switch  │
  │                    └── DSCP is the priority PFC acts on in a     │
  │                        DSCP-based deployment (routable!)         │
  ├──────────────────────────────────────────────────────────────────┤
  │ UDP  src port = RANDOM PER QUEUE PAIR   ← the only ECMP entropy  │
  │      dst port = 4791  (fixed: "this is RoCEv2")                  │
  ├──────────────────────────────────────────────────────────────────┤
  │ IB BASE TRANSPORT HEADER (BTH)                                   │
  │   opcode | destination QP number | PSN | ack-request bit | ...   │
  ├──────────────────────────────────────────────────────────────────┤
  │ [RETH for WRITE/READ: remote virtual address, rkey, length]      │
  ├──────────────────────────────────────────────────────────────────┤
  │ PAYLOAD                                                          │
  ├──────────────────────────────────────────────────────────────────┤
  │ ICRC (invariant CRC) | Ethernet FCS                              │
  └──────────────────────────────────────────────────────────────────┘
```

Four things to read off that diagram, each of which is an operational fact:

- **Two different fields can carry "priority."** Classic deployments used the 802.1Q **PCP** bits, which requires VLAN tagging and does not survive routing. Microsoft's production deployment moved to **DSCP-based PFC** precisely because VLAN-based classification did not scale: with DSCP the priority lives in the IP header, so it is preserved across subnets, server-facing switch ports no longer need to be trunks (which had broken PXE boot), and the whole scheme becomes layer-3 rather than layer-2. Their DSCP-based PFC specification was subsequently supported by all major switch vendors. **On any modern RoCE fabric, assume DSCP.** The NIC-side equivalent is the driver's *trust state* — mlx5 supports `MLX5_QPTS_TRUST_PCP` and `MLX5_QPTS_TRUST_DSCP`, and switches to DSCP trust when a DSCP-to-priority application entry is configured.
- **ECN lives in the IP header, not in the DCB suite.** It is two bits: `10`/`01` mean "ECN-capable transport," and a congested switch rewrites them to `11` (**CE**, congestion experienced) instead of dropping. That is why ECN works across routers and why it is orthogonal to PFC, which is a link-local Ethernet mechanism.
- **The destination UDP port is a constant.** All ECMP entropy comes from the per-QP source port — 09.2 §6's argument, now visible in the header layout.
- **The BTH carries the destination QP number and PSN.** That is what makes hashing on destination-QP fields possible for a vendor who wants better flow spreading than the five-tuple gives, and it is what a fabric monitoring tool parses to attribute traffic to a queue pair.

### 6. The DCB suite: which standard does what

"Data Center Bridging" is a family of IEEE 802.1 amendments, and being able to name which one does what — and which knobs implement each on Linux — separates people who have configured a lossless fabric from people who have read about one.

| Standard | Name | What it does | Where you configure it |
|---|---|---|---|
| **802.1Qbb** | Priority-based Flow Control (PFC) | Per-priority PAUSE. Lets one of eight priorities be stopped independently of the rest — the mechanism that converts buffer overflow into backpressure. | `struct ieee_pfc` over DCB netlink: `pfc_cap` (how many lossless classes the port supports), `pfc_en` (an 8-bit mask of enabled priorities), `mbc`, `delay`, plus `requests[8]`/`indications[8]` counters. Tools: `dcb pfc`, `lldptool`, `mlnx_qos`. |
| **802.1Qaz** | Enhanced Transmission Selection (ETS) | Bandwidth allocation between traffic classes: guarantees each class a share of the link and lets unused share be borrowed. Prevents the lossless class from starving everything else, and vice versa. | `struct ieee_ets`: `prio_tc[8]` (priority→traffic class map), `tc_tx_bw[8]` (percentages, which must total 100), `tc_tsa[8]` (the selection algorithm per TC: **0 = strict priority, 1 = credit-based shaper, 2 = ETS, 255 = vendor**). |
| **802.1Qaz** (same amendment) | DCBX | The LLDP-based exchange that lets a switch and a NIC agree on PFC and ETS settings instead of both being configured by hand. Has a `willing` bit: the side that is "willing" accepts the peer's configuration. | `ieee_ets.willing`; `dcb`/`lldptool`; firmware-offloaded on many NICs. |
| **802.1Qau** | Quantized Congestion Notification (QCN) | An L2 end-to-end congestion-notification scheme, the ancestor of DCQCN. Largely superseded in practice by ECN-based schemes because ECN is routable and universally supported. | rarely deployed |
| **RFC 3168 (not DCB)** | ECN | The IP-layer congestion signal RoCE actually uses. Not part of DCB at all — this is the most common vocabulary error in this area. | switch queue thresholds (WRED/ECN profiles); NIC-side reaction in §11 |
| **vendor / driver** | Buffer allocation | Which of the port's buffers each priority uses, and how big each is — the thing PFC headroom is carved out of. | `struct dcbnl_buffer`: `prio2buffer[8]` and `buffer_size[8]`; on mlx5 this is what `mlx5e_port_manual_buffer_config()` programs. |

All the structure and constant names above are verifiable in `include/uapi/linux/dcbnl.h` and in the mlx5 driver's `en_dcbnl.c` in the Linux tree — including `IEEE_8021QAZ_MAX_TCS = 8`, the TSA identifiers, and mlx5's rule that ETS bandwidth allocations are percentages capped at 100.

**Two configuration facts that trip people up.** First, `pfc_cap` is usually smaller than 8 in practice — not because the standard says so, but because *headroom* is the binding constraint (§8). Microsoft's production switches, with 9 MB or 12 MB of shared buffer, could only afford headroom for **two** lossless classes despite the hardware supporting eight, so they ran one lossless class for latency-sensitive traffic and one for bulk. Second, the `delay` field in `ieee_pfc` is documented in the kernel header as the allowance for the link's round-trip propagation delay — but mlx5 interprets and stores it as a **cable length**, capped at `MLX5E_MAX_CABLE_LENGTH = 1000`, and feeds it straight into the headroom formula in §8. Setting it wrong does not produce an error; it produces a buffer that is too small for the link, and therefore drops under exactly the conditions PFC exists to prevent.

### 7. PFC at the buffer level: XOFF, XON, headroom, and the pause frame

This is the mechanism the entire RoCE story rests on, so build it from the buffer outward rather than from the acronym inward.

A switch port's ingress buffer for a lossless priority has two watermarks. When occupancy rises past **XOFF**, the switch emits a PFC pause frame upstream telling the previous hop to stop sending *that priority*. When occupancy falls back below **XON**, it emits a pause with duration zero, which means resume. On the shared-buffer switches used in large deployments, an "ingress queue" is really just a counter against a common pool — which matters, because it means one priority's occupancy is competing with everything else for the same physical memory.

The subtlety that makes PFC hard is the **gray period**. A pause frame takes time to reach the upstream port, and the upstream port takes time to react; during that window it keeps transmitting. Those bytes must land somewhere, so the ingress port has to reserve buffer for them *in advance*. That reservation is **headroom**, and its size is set by the MTU, the peer's PFC reaction time, and — dominantly — the propagation delay of the cable.

```
   ONE INGRESS PORT, ONE LOSSLESS PRIORITY

   buffer occupancy
        ▲
        │ ┌──────────────── PORT BUFFER SIZE ──────────────────┐
        │ │                                                    │
        │ │   ┌──── HEADROOM ────┐  reserved for the bytes     │
   XOFF ┼─┼───┤                  │  already in flight when     │
        │ │   │                  │  the pause frame was sent   │
        │ │   └──────────────────┘                             │
        │ │                          xoff_threshold =          │
   XON  ┼─┼─── (= xoff_threshold − max_mtu)   size − headroom  │
        │ │                                                    │
        │ └────────────────────────────────────────────────────┘
        └──────────────────────────────────────────────▶ time

   ①  occupancy crosses XOFF  → emit PFC pause for this priority
   ②  gray period: upstream still transmitting for
        (propagation + reaction) — those bytes fill HEADROOM
   ③  upstream stops.  Local egress drains the queue.
   ④  occupancy falls below XON → emit pause with duration 0 = resume

   IF HEADROOM IS TOO SMALL, ② OVERFLOWS AND THE SWITCH DROPS —
   which is the one thing this entire apparatus exists to prevent.
```

The pause frame itself is small and completely specified, and being able to describe it is a cheap credibility win:

```
   PFC FRAME (IEEE 802.1Qbb), 60 bytes on the wire + 4 byte FCS

   ┌────────────────────────┬──────────────────────────────────────┐
   │ destination MAC        │ 01:80:C2:00:00:01  (reserved         │
   │                        │  multicast; never forwarded)         │
   │ source MAC             │ the station's own MAC                │
   │ EtherType              │ 0x8808   (MAC Control)               │
   │ opcode                 │ 0x0101   (PFC; plain PAUSE is 0x0001)│
   │ class-enable vector    │ 2 bytes: one bit per priority 0..7    │
   │ time[0] .. time[7]     │ 8 × 2 bytes, one duration per        │
   │                        │ priority, in PAUSE QUANTA            │
   │ padding                │ to 60 bytes                          │
   └────────────────────────┴──────────────────────────────────────┘

   ONE QUANTUM = the time to transmit 512 bits at the link rate.

        link rate     1 quantum      max pause (65,535 quanta)
        ──────────    ──────────     ─────────────────────────
         25 Gb/s      20.48 ns             1.34 ms
        100 Gb/s       5.12 ns           335.5 µs
        400 Gb/s       1.28 ns            83.9 µs   ← 4.2 MB of
        800 Gb/s       0.64 ns            41.9 µs      un-sent data
```

Two consequences of that table. First, **a pause is not a small event at 400G**: even a single maximum-duration pause withholds about 4.2 MB from the link, and a sender that is repeatedly paused is effectively rate-limited by the pause duty cycle. Second, **pause resolution scales with rate**: the quantum shrinks as the link gets faster, so the same 16-bit field buys proportionally less time — which is one reason faster links need proportionally more headroom and better congestion control, not less.

### 8. Worked math: sizing PFC headroom, and why it gets ugly at 400G

Now do the arithmetic that decides whether a lossless RoCE deployment is even feasible on a given switch. Two ways: from first principles, and using the formula a real driver implements.

**From first principles.** Headroom must cover everything the upstream port can put on the wire between "I crossed XOFF" and "the upstream has actually stopped":

```
   headroom ≥ rate × (2 × propagation_delay + reaction_time) + MTU

   Take a 400 Gb/s link, 100 m of fibre, 4 KB MTU:
     rate               = 400 Gb/s = 50 GB/s = 50 bytes/ns
     propagation        ≈ 5 ns per metre in fibre
                        → 100 m one way = 500 ns; round trip = 1 µs
     round-trip bytes   = 50 bytes/ns × 1,000 ns        = 50,000 B
     reaction time      ≈ 1-3 µs (frame emission, PHY, upstream
                          detection and scheduler response)
                        → at 2 µs: 50 × 2,000                = 100,000 B
     one MTU in progress                                     =   4,096 B
                                                    TOTAL ≈ 154,000 B
                                                          ≈ 150 KiB
```

**Using the driver's formula.** The mlx5 Ethernet driver computes headroom explicitly, and its comment states the formula it implements:

```
   xoff = (301 + 2.16 × cable_len[m]) × speed[Gb/s] + 2.72 × MTU[B]      (bytes)
   with speed clamped to a minimum of 40 Gb/s

   and then:
       xoff_threshold = buffer_size − xoff        (where the pause fires)
       xon_threshold  = xoff_threshold − max_mtu  (where it resumes)
       max_mtu is at least 9216 B in the driver's accounting
```

Read the two constants, because they turn the formula from magic into physics:

- `2.16 bytes per metre per Gb/s` is the cable term. Pure round-trip propagation at 5 ns/m and 1 Gb/s = 0.125 bytes/ns gives 1.25 bytes/m/Gb/s, so 2.16 carries roughly a 1.7× engineering margin over the ideal.
- `301 bytes per Gb/s` is the fixed reaction-time term. At 400 Gb/s that is 120,400 bytes, which at 50 GB/s corresponds to about **2.4 µs of allowance** for frame emission, PHY latency and the upstream port's response — matching the first-principles estimate above.

**Now run it, and watch the problem appear:**

```
   400 Gb/s, 100 m, MTU 4096 B:
       (301 + 2.16×100) × 400 + 2.72×4096
     = 517 × 400 + 11,141
     = 206,800 + 11,141  =  217,941 B   ≈ 213 KiB   PER PORT PER PRIORITY

   400 Gb/s, 5 m (in-rack copper), MTU 4096:
       (301 + 10.8) × 400 + 11,141      =  135,861 B  ≈ 133 KiB
       → note the FIXED term dominates at short reach

   100 Gb/s, 100 m, MTU 4096:
       517 × 100 + 11,141               =   62,841 B  ≈  61 KiB
       → headroom scales ~linearly with rate

   NOW MULTIPLY BY THE PORT COUNT — this is the punchline:
       64-port 400G switch, 100 m links, ONE lossless priority:
           64 × 213 KiB   ≈  13.3 MiB
       TWO lossless priorities:
           64 × 2 × 213 KiB ≈ 26.6 MiB

   Against real shallow-buffer switches with 9 MB or 12 MB of
   shared buffer (Microsoft's production ToR/leaf class, 2016):
       → you cannot even fund ONE lossless class at 400G/100 m,
         let alone the eight that 802.1Qbb nominally offers.
```

That arithmetic is the single most useful thing in this lesson, because it explains, without hand-waving, several facts that otherwise look like folklore:

- **Why real deployments run one or two lossless classes, not eight.** Microsoft said exactly this: with 9–12 MB switches and links up to 300 m, they could reserve headroom for only two lossless traffic classes, so they used one for real-time traffic and one for bulk transfer. The standard's eight priorities are a hardware capability, not a budget.
- **Why cable length is a fabric-design parameter, not a cabling detail.** Headroom is roughly linear in reach. The 200–300 m leaf-to-spine runs that a three-tier Clos requires (09.2) are precisely the links that cost the most buffer, which is a real argument for fewer tiers or shorter reach.
- **Why "deep-buffer switch" is a procurement line item on an AI fabric.** As rate rises, headroom rises with it; a switch generation that was comfortable at 100G can be structurally unable to support lossless operation at 400G on long links.
- **Why the `delay`/cable-length knob matters so much.** It is a direct multiplier in the formula. Configure 5 m on a 100 m link and you under-reserve by about 80 KB per port — and the failure mode is silent drops under load, which is the exact condition the fabric was built to avoid.

### 9. How a pause propagates: congestion spreading, head-of-line blocking, and PFC storms

PFC works hop by hop, and the consequences of that are the reason DCQCN exists. Follow one congestion event through a three-tier fabric in time:

```
  t=0     Two senders under LEAF-B both burst to one receiver under LEAF-A.
          LEAF-A's egress port toward the receiver saturates.

  t=~µs   LEAF-A's INGRESS buffers for the RDMA priority cross XOFF.
          LEAF-A sends PFC pause upstream to SPINE-1 and SPINE-2.

          LEAF-A ──PAUSE(prio 3)──▶ SPINE-1
                 ──PAUSE(prio 3)──▶ SPINE-2

  t=+     SPINE-1 stops forwarding priority 3 toward LEAF-A.  Its own
          ingress buffers now fill — INCLUDING traffic destined for
          LEAF-C, LEAF-D, LEAF-E, which had nothing to do with the
          congestion.  THIS IS HEAD-OF-LINE BLOCKING: the pause is
          per-PRIORITY, not per-FLOW or per-destination.

  t=++    SPINE-1 crosses ITS XOFF and pauses every leaf beneath it.

          SPINE-1 ──PAUSE──▶ LEAF-B, LEAF-C, LEAF-D, ... LEAF-N

  t=+++   Those leaves pause their servers' NICs.
          The blast radius of one oversubscribed receiver is now
          an entire fabric branch.

  ┌───────────────────────────────────────────────────────────────┐
  │  BLAST RADIUS OVER TIME                                       │
  │                                                               │
  │  t=0      [ one egress port ]                                 │
  │  t=+      [ one leaf's ingress ]                              │
  │  t=++     [ two spines ]                                      │
  │  t=+++    [ every leaf under those spines ]                   │
  │  t=++++   [ every server under those leaves ]                 │
  │                                                               │
  │  Innocent flows stalled: ~everything sharing the priority.    │
  │  Time to full spread: microseconds to low milliseconds.       │
  └───────────────────────────────────────────────────────────────┘
```

**That is congestion spreading**, and the "victim flow" problem it creates is the core objection to running PFC as your congestion control. PFC has no idea *which* flow is at fault. It is a per-class stop signal, and every flow sharing that class pays.

The pathological version of this is the **NIC PFC pause frame storm**, and it is not hypothetical. Microsoft documented it in production: a bug in a NIC's *receive* pipeline stopped it from processing packets, its receive buffer filled, and it began emitting pause frames continuously. The cascade they describe is exactly the timeline above, but originating at a single server and never stopping:

```
   1. malfunctioning NIC on server 0 sends pause frames continuously
   2. its ToR pauses ALL its other ports, including uplinks to leaves
   3. the leaves pause the spines
   4. the spines pause the OTHER leaves
   5. those leaves pause their ToRs
   6. those ToRs pause their servers
        →  one broken NIC can stop an entire network from transmitting
```

Their fix is instructive because it is defence in depth rather than a single mechanism, and both halves are worth knowing:

- **A NIC-side watchdog.** The NIC's own microcontroller monitors the receive pipeline; if it has been stalled for a configured period (**default 100 ms**) while the NIC is generating pause frames, it disables pause generation. This does not rescue the server — a NIC in that state is broken and the server is effectively off the network — its purpose is to stop the damage escaping the server.
- **A switch-side watchdog.** On server-facing ports, if the egress queue cannot drain *and* the port is receiving continuous pause frames from the NIC, the switch disables lossless mode for that port and discards the lossless traffic to and from it, re-enabling after the pause frames have been absent for a period (**default 200 ms**).

The general principle behind both: **when losslessness itself becomes the failure, the correct action is to give up losslessness for the smallest possible scope**. A PFC watchdog is a mandatory component of a production RoCE fabric, not an optional extra, and "do you run PFC watchdogs?" is a legitimate question to ask a vendor or an operations team.

### 10. PFC deadlock: the cyclic buffer dependency, and the real-world trigger

Congestion spreading is bad but self-clearing. Deadlock is not.

A deadlock forms when pause dependencies form a **cycle**: switch A is paused by B, B by C, C by A, so no buffer can drain because each is waiting on the next. The abstract condition is a **cyclic buffer dependency**, which is a classic result in interconnect theory and the reason HPC fabrics use deadlock-free routing schemes such as up/down routing.

```
   PFC DEADLOCK — nothing here can ever move again

              ┌─────────────────────────────────┐
              │                                 │
              ▼                                 │
        ┌──────────┐   PAUSE   ┌──────────┐     │
        │ SWITCH A │ ────────▶ │ SWITCH B │     │
        └──────────┘           └──────────┘     │
              ▲                      │          │
              │                      │ PAUSE    │
        PAUSE │                      ▼          │
        ┌──────────┐           ┌──────────┐     │
        │ SWITCH D │ ◀──────── │ SWITCH C │ ────┘
        └──────────┘   PAUSE   └──────────┘

   Each switch's ingress buffer for the lossless priority is full.
   Each is waiting for the NEXT switch to drain before it can forward.
   Every one of them has already sent PAUSE upstream.
   No packet can leave any buffer.  The cycle is stable.
```

What makes this more than a textbook curiosity is **how the cycle actually forms in production**, and the documented answer is genuinely surprising. Microsoft's deadlock did not come from an exotic routing configuration; it came from **ordinary Ethernet flooding**. When a switch does not know the destination MAC for a packet — which happens when an ARP entry is *incomplete*, i.e. the IP-to-MAC mapping exists but the MAC-to-port mapping has aged out — it floods the packet to every port except the one it arrived on. That flooding sends lossless traffic down paths the routing would never have chosen, and those paths closed a dependency cycle among four switches. Their report is blunt about the consequence: **once the deadlock occurred, it did not go away even after restarting all the servers.**

The fix they chose is equally instructive: **drop lossless packets whose ARP entry is incomplete**, rather than flooding them. They considered and rejected two alternatives — punting to the switch CPU (too much CPU load) and tuning the MAC-table timeout to exceed the ARP timeout (fragile in both directions). Deliberately dropping a packet on a lossless class sounds like heresy until you compare it with a wedged fabric region.

So the full mitigation stack for deadlock is:

1. **Routing that cannot form a cycle** — up/down or otherwise deadlock-free routing, and no non-minimal or re-ordered paths for the lossless class.
2. **Do not flood the lossless class** — the specific production trigger above.
3. **PFC watchdogs** that break the cycle by dropping the stuck class after a timeout, per §9.
4. **Strategically, make PFC rare** — which is what ECN and DCQCN are for, and is the subject of §11. A fabric in which PFC fires routinely in steady state is a fabric that is rolling the dice on deadlock every day.

### 11. ECN and DCQCN: the control loop that keeps PFC from firing

PFC is a link-local, per-class, binary hammer. DCQCN is the fine-grained, end-to-end, per-flow controller whose job is to make sure the hammer is never needed. It has three roles, and the vocabulary is worth using precisely:

- **CP — congestion point.** The switch. It marks packets rather than dropping or pausing.
- **NP — notification point.** The receiving NIC. It converts marks into a notification.
- **RP — reaction point.** The sending NIC. It rate-limits the offending flow.

```
   THE DCQCN CONTROL LOOP

   ┌──────────┐        data (ECT)        ┌──────────┐        ┌──────────┐
   │ SENDER   │ ───────────────────────▶ │  SWITCH  │ ─────▶ │ RECEIVER │
   │  (RP)    │                          │   (CP)   │        │   (NP)   │
   └──────────┘                          └──────────┘        └──────────┘
        ▲                                      │                   │
        │                       queue > Kmin → │ mark ECN=CE       │
        │                       probabilistic  │ (probability      │
        │                       up to Pmax at  │  ramps to Pmax    │
        │                       Kmax           │  between Kmin     │
        │                                      │  and Kmax)        │
        │                                      ▼                   │
        │                              ┌───────────────────────────┘
        │                              │  NP sees CE on a QP's packets
        │        CNP (congestion       │  and generates a CNP,
        └──── notification packet) ────┘  rate-limited by
                                          min_time_between_cnps

   AT THE RP, ON EACH CNP:
       alpha  ← alpha + g × (1 − alpha)        (raise congestion estimate)
       rate   ← rate × (1 − alpha/2)           (multiplicative decrease)
       target ← rate before the decrease       (remember where we were)

   AT THE RP, WITH NO CNP FOR A WHILE (timer + byte counter both used):
       FAST RECOVERY   : rate ← (rate + target)/2, repeated F times
       ADDITIVE INCREASE : target ← target + Rai ; rate ← (rate+target)/2
       HYPER INCREASE  : target ← target + Rhai ; (aggressive re-probe)
       alpha decays toward 0 on a timer, so a long-quiet flow forgets
       that it was ever congested.
```

**The knobs are real and enumerable.** On mlx5 hardware the reaction-point and notification-point parameters are exposed two ways, and both are worth knowing because you will meet each in different documentation:

| Where | Path | Parameters |
|---|---|---|
| Driver sysfs (per netdev) | `/sys/class/net/<dev>/ecn/roce_rp/` | `clamp_tgt_rate`, `rpg_time_reset`, `rpg_byte_reset`, `rpg_threshold`, `rpg_ai_rate`, `rpg_hai_rate`, `rpg_gd`, `rpg_min_dec_fac`, `rpg_min_rate`, `rate_to_set_on_first_cnp`, `initial_alpha_value`, `rate_reduce_monitor_period`, `dce_tcp_g`, `dce_tcp_rtt` |
| Driver sysfs (per netdev) | `/sys/class/net/<dev>/ecn/roce_np/` | `enable/<prio>`, `min_time_between_cnps`, `cnp_dscp`, `cnp_802p_prio` |
| Kernel debugfs (per device) | `/sys/kernel/debug/mlx5/<pci>/cc_params/` | `rp_clamp_tgt_rate`, `rp_time_reset`, `rp_byte_reset`, `rp_threshold`, `rp_ai_rate`, `rp_hai_rate`, `rp_max_rate`, `rp_min_dec_fac`, `rp_min_rate`, `rp_rate_to_set_on_first_cnp`, `rp_dce_tcp_g`, `rp_dce_tcp_rtt`, `rp_rate_reduce_monitor_period`, `rp_initial_alpha_value`, `rp_gd`, `np_min_time_between_cnps`, `np_cnp_dscp`, `np_cnp_prio_mode`, `np_cnp_prio` |

Both lists are verifiable in the Linux tree (`drivers/infiniband/hw/mlx5/cong.c` for the debugfs set; the sysfs names are those the `prometheus/procfs` library parses from `/sys/class/net/<iface>/ecn/`). Map the important ones onto the algorithm above:

- **`rp_gd`** is `g`, the gain in the alpha update — how strongly one CNP moves the congestion estimate.
- **`rp_ai_rate` / `rp_hai_rate`** are `Rai` and `Rhai`, the additive and hyper-additive increase steps.
- **`rp_threshold`** is `F`, how many fast-recovery steps happen before additive increase begins.
- **`rp_time_reset` / `rp_byte_reset`** are the timer and byte-counter that decide when the sender is allowed to start increasing again — DCQCN uses *both*, so that a slow flow is not held down by wall-clock alone and a fast flow is not released purely by elapsed time.
- **`rp_min_dec_fac`** bounds how far one decrease may cut the rate (a floor on the multiplicative factor), preventing a single burst of CNPs from collapsing a flow to nothing.
- **`np_min_time_between_cnps`** rate-limits notification generation, so a heavily marked flow does not generate a CNP storm of its own.
- **`np_cnp_dscp` / `np_cnp_prio`** put CNPs in their own, usually higher-priority, class — because a congestion notification stuck behind the congestion it is reporting is useless.

**The switch side is where the thresholds live.** The congestion point marks with a RED-like profile: no marking below `Kmin`, probability ramping to `Pmax` at `Kmax`, and marking everything above. The DCQCN paper's recommended starting values were `Kmin = 5 KB`, `Kmax = 200 KB`, `Pmax = 1%` at 40 Gb/s. Two caveats you must attach whenever you quote those: they are **per-generation** (thresholds scale with rate and with buffer size — a 40 Gb/s recommendation is not a 400 Gb/s recommendation), and the DCQCN paper itself was **not reachable from this environment**, so those three numbers come from secondary summaries and should be treated as a starting point to verify against your switch vendor's guidance rather than as settled configuration.

**The ordering rule, stated mechanically rather than as a slogan:** ECN marking must begin at a queue depth *below* the depth at which PFC fires. If `Kmin` is above the XOFF threshold, PFC pauses the link before any sender is ever told to slow down, so DCQCN never engages and the fabric runs permanently on the blunt instrument — with all of §§9–10's failure modes live. If `Kmin` is far too low, every flow is marked constantly, senders sit at their minimum rate, and you lose throughput to phantom congestion. **The invariant is `Kmin < Kmax < XOFF`, with enough distance between `Kmax` and `XOFF` for one control-loop round trip.** That last clause is why the rule cannot be reduced to "set ECN lower than PFC": the gap has to be large enough for a CNP to make it back to the sender and for the sender's rate change to reach the switch, which is roughly a fabric RTT plus `min_time_between_cnps`.

### 12. Reading the fabric: the counters that tell you which mechanism is firing

Everything above is observable, and knowing which counter corresponds to which mechanism is what turns theory into operations. On an mlx5 NIC, `ethtool -S` exposes the per-priority set (the counter names below are the driver's own format strings from `en_stats.c`, with `%d` replaced by the priority):

```
$ ethtool -S ens1f0 | grep -E 'prio3|pause|discard'
     rx_prio3_bytes: 84719203847362
     rx_prio3_packets: 20693166921
     rx_prio3_discards: 0
     tx_prio3_bytes: 83991204773621
     tx_prio3_packets: 20515332188
     rx_prio3_pause: 41927                 ← pause frames RECEIVED for prio 3
     rx_prio3_pause_duration: 8134         ← total quanta we were paused
     tx_prio3_pause: 1290418               ← pause frames WE SENT
     tx_prio3_pause_duration: 2914772      ← total quanta we asked for
     rx_prio3_pause_transition: 39218      ← XOFF→XON transitions
     rx_prio3_buf_discard: 0               ← dropped for lack of buffer  ★
     rx_prio3_cong_discard: 0              ← WRED/congestion drops       ★
     rx_prio3_marked: 9147732              ← packets ECN-marked          ★
     rx_pause_ctrl_phy: 41927
     tx_pause_ctrl_phy: 1290418
     tx_pause_storm_warning_events: 0      ← device stall, minor watermark
     tx_pause_storm_error_events: 0        ← device stall, critical
```

How to read it, in the order you should read it:

- **`rx_prio3_buf_discard` or `rx_prio3_cong_discard` non-zero on a lossless priority is an emergency.** It means the fabric dropped RDMA traffic — the exact event PFC exists to prevent — and per §1 the cost is a go-back-N or go-back-0 event, not a lost packet. Investigate headroom (§8) and the priority-to-buffer mapping before anything else.
- **`rx_prio3_marked` rising while `tx_prio3_pause` stays near zero is the healthy state.** ECN is doing the work and PFC is idle: exactly the intended regime.
- **`tx_prio3_pause` rising in steady state means you are running on the backstop.** Either the ECN thresholds are above the PFC threshold (the inversion in §11), or there is a real capacity problem, or a receiver is too slow. Correlate with `rx_prio3_pause` on the peers to see how far the backpressure has spread.
- **`rx_prio3_pause` rising on a host that is not congested is the head-of-line-blocking signature** — you are being paused because of somebody else's congestion sharing your priority.
- **`tx_pause_storm_warning_events` / `tx_pause_storm_error_events`** are the device's own stall watermarks: the NIC noticing that it has been unable to drain for too long. Non-zero values here are the local view of §9's storm scenario.

The configuration side is equally inspectable. `dcb pfc show dev ens1f0` (iproute2) or `mlnx_qos -i ens1f0` prints the enabled priority mask, the ETS bandwidth allocation and transmission-selection algorithm per traffic class, the priority-to-traffic-class map, the trust state (PCP or DSCP), and the DSCP-to-priority table. **The single most common RoCE misconfiguration is an inconsistency in that mapping across a vendor boundary** — the NIC marks DSCP 26, the leaf switch maps DSCP 26 to priority 3, and the spine maps it to priority 0, which is lossy. The traffic still flows; it just is not lossless any more, and nothing announces it. Auditing that the same DSCP-to-priority-to-queue mapping holds at every hop is the first item in any RoCE bring-up checklist.

### 13. Spectrum-X, and the fourth path Meta actually took

Two developments sit between "raw RoCE" and "buy InfiniBand," and both belong in a procurement discussion.

**Spectrum-X is managed RoCE.** It pairs the Spectrum-4 switch (SN5600: 64 × 800G or 128 × 400G, 51.2 Tb/s) with BlueField-3 or ConnectX SuperNICs, co-designed so the Ethernet fabric behaves like a lossless RDMA fabric on vendor-validated defaults: hardware **adaptive routing** instead of pure five-tuple ECMP (which attacks 09.2 §6's hash polarisation structurally), telemetry-driven congestion control, and per-flow load balancing. The honest framing: it **buys down** tuning risk rather than removing it — you still configure PFC and ECN, against defaults someone else validated — and it re-introduces much of the single-vendor exposure that made commodity RoCE attractive, because the benefits want NVIDIA switches *and* NVIDIA NICs.

**Meta changed which layer owns congestion avoidance.** Their SIGCOMM 2024 account reports running **PFC-only, DCQCN-disabled** 400G RoCE in production for a multi-year period at 24K+ GPU scale, substituting **receiver-driven traffic admission co-designed between the collective library and the RoCE transport**: the receiver, which knows the pattern of the collective it is running, controls when data may arrive, instead of a generic NIC algorithm reacting to ECN marks it has no application context for. They also report hashing ECMP on destination-QP fields rather than the five-tuple, improving AllReduce by up to roughly 40%.

**Read that correctly, because the loose version reads badly in an interview.** It is not the absence of congestion control; it is congestion control at a different layer with more application context. What made it viable rather than reckless: deep-buffer switches absorbing microbursts so PFC rarely fires, plus an organisation large enough to validate the substitution first. The transferable insight is that **"RoCE tuning risk" is not a fixed tax** — it can be partly retired by moving the responsibility. Most organisations cannot do that and should not plan as though they can.

*(These Meta figures come from a paper and blog post that were not reachable from this environment in this pass; they are carried with their original attribution and flagged in References. The Microsoft findings in §§7–10, by contrast, were read directly from the paper.)*

### 14. The decision, made explicit

| Axis | InfiniBand (Quantum-2/X800 + ConnectX-7/8) | RoCEv2 on commodity Ethernet | Spectrum-X (managed RoCE) |
|---|---|---|---|
| How losslessness is achieved | credit-based link flow control, intrinsic, per virtual lane | PFC + ECN + DCQCN, configured and validated by you | PFC + ECN with vendor-validated defaults, plus adaptive routing |
| Configuration surface that can be wrong | congestion-control table (affects throughput only) | headroom, XOFF/XON, DSCP→priority at every hop, ECN thresholds, DCQCN parameters, NIC firmware (affects **losslessness**) | reduced but non-zero |
| Small-message latency / tail | best; deterministic | good when tuned; tail exposed to PFC events | near-IB; adaptive routing tames the tail |
| Load balancing | subnet-manager-computed routing plus adaptive routing | five-tuple ECMP by default → hash polarisation on LLM traffic | hardware adaptive routing |
| Control plane | subnet manager (one brain; must be made HA); IB-specific tooling | your existing BGP/EVPN, ECMP and telemetry | Ethernet control plane plus vendor management |
| In-network reduction | **SHARP** (09.5) — changes the collective's algorithmic cost | none | none |
| Optics and radix economics | narrower, premium, single-source | broad multi-vendor market; higher-radix switches can remove a tier | mid; NVIDIA switch + SuperNIC premium |
| Second-sourcing / negotiation | weak position, single vendor | strong | limited |
| Skills | IB-specific (subnet manager, `ibdiagnet`, UFM) — scarcer to hire | commodity Ethernet skills you already employ, **plus** a lossless-fabric qualification project | Ethernet skills plus vendor support |

**The one-sentence version to carry into an interview:** *InfiniBand buys determinism and in-network reduction with money and lock-in; RoCE buys ecosystem and cost leverage with a tuning project whose buffer arithmetic I can show you — and at extreme scale with a capable network organisation, part of that tuning risk can itself be retired by moving congestion avoidance into the collective library.*

## Perspectives

**Theory.** Credit-based flow control and PFC solve the same problem from opposite ends. Credits are a *closed-loop, proportional, per-lane* scheme in which a sender is never permitted to create an overflow in the first place; PFC is an *open-loop, binary, per-class* scheme in which overflow is prevented by reacting to it fast enough. The reason the second one needs headroom, watchdogs, deadlock-free routing and a whole second protocol (DCQCN) is that reaction takes time, and everything sent during that time has to be parked somewhere. **Any pause-based scheme inherits that gray period; only a credit-based scheme avoids it.** That is the single deepest structural difference between the two fabrics, and it explains almost every downstream difference in their operations.

**Practice.** The operational difference is not "IB is easier" — it is *where* the difficulty lives. On InfiniBand you make one subnet manager highly available, you learn `ibdiagnet` and `perfquery`, and you watch `PortXmitWait`. On RoCE you own a configuration surface that spans every switch and NIC in the path (DSCP maps, PFC masks, buffer allocations, ECN thresholds, DCQCN parameters, NIC firmware) and must be re-validated on every firmware update, every switch model introduction, and every link-speed transition — because §8's headroom arithmetic changes when the rate or the reach changes. Both are real jobs. Only one of them is a job you can accidentally not do.

**Failure mode.** The instructive thing about the two documented production disasters is that neither was a tuning error. The **pause storm** was a NIC receive-pipeline bug; the **deadlock** was ordinary Ethernet flooding of a packet with an incomplete ARP entry. Both were caused by mechanisms working exactly as specified, interacting in a way nobody designed. That is why the mature posture is not "engineer a fabric where PFC never fires" but **"make PFC survivable when it does"** — deep buffers to absorb microbursts, watchdogs to break cycles, validated headroom, and monitoring that tells you which mechanism is firing. Resilience to the failure mode, not just avoidance of it.

**Economics.** Four terms pull in different directions: switch and NIC capex, optics and cabling capex, power and cooling opex, and engineering headcount — offset by GPU-hours saved through higher collective efficiency. InfiniBand tends to win the *efficiency* term (determinism plus in-network reduction) and the *headcount* term at small scale, where there is no fabric team to run a qualification project. RoCE tends to win the *optics capex*, *radix* and *second-sourcing* terms at large scale, where the optics bill is enormous and an existing Ethernet organisation absorbs the tuning work. **The verdict flips based on which term dominates for your cluster size**, and being able to say which term dominates and why is the senior competency being tested.

## Real-world use cases

- **Microsoft — RDMA over Commodity Ethernet at Scale (SIGCOMM 2016).** [Paper PDF](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/rdma_sigcomm2016.pdf) — read directly for this lesson and the source of most of its mechanical detail: DSCP-based PFC and why VLAN-based classification did not scale; the XOFF/XON/headroom model and the finding that 9–12 MB shallow-buffer switches with links up to 300 m could fund headroom for only **two** lossless classes out of eight; the NIC pause-frame storm and its two watchdogs; the flooding-induced PFC deadlock that survived restarting every server; and the go-back-0 transport livelock. The deployment context — 40 Gb/s links, ~2 m server-to-ToR, 10–20 m ToR-to-leaf, 200–300 m leaf-to-spine — is why the headroom arithmetic in §8 is a fabric-design input and not a footnote. Everything it documents happened at 40 Gb/s; §8 shows the numbers get an order of magnitude harder at 400.
- **Meta — RoCE for distributed AI training at scale (SIGCOMM 2024 and engineering blog).** [Paper PDF](https://engineering.fb.com/wp-content/uploads/2024/08/sigcomm24-final246.pdf) · [blog](https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/). What it shows: a multi-year production RoCE fabric at 24K+ GPUs run **PFC-only with DCQCN disabled**, substituting receiver-driven traffic admission co-designed into the collective library, with deep-buffer switches absorbing microbursts and destination-QP-aware ECMP reported to improve AllReduce performance by up to ~40%. The single most load-bearing "it depends" citation in the module — and the one most often mis-stated as "Meta runs without congestion control." Not re-verified in this pass.
- **Alibaba HPN (SIGCOMM 2024).** [Paper](https://dl.acm.org/doi/10.1145/3651890.3672265). What it shows: ECMP hash polarisation under LLM training's small number of ~400 Gb/s bursty flows treated as a first-class failure mode, fixed **topologically** (2-tier dual-plane with twin ToRs) rather than by congestion-control tuning — evidence that not every RoCE pathology lives at the PFC/ECN layer. Not re-verified in this pass.
- **Microsoft Azure — ECN and PFC thresholds for AI clusters.** [Azure blog](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/striking-the-right-balance-ecn-and-pfc-thresholds-for-ai-clusters/4468629). What it shows: a second hyperscaler's production tuning playbook — the counterpoint to Meta, from a team that kept DCQCN and instead focused on getting the ECN-leads-PFC-trails relationship right, and documents the instability that follows from inverting it. Not re-verified in this pass.

## Worked example — size the buffer, then make two procurement calls

**Part A — can this switch run lossless RoCE at all?**

You are evaluating a 64-port 400G Ethernet switch with **16 MB** of shared packet buffer for an AI fabric whose leaf-to-spine links are **100 m** and whose MTU is **4096 B**. The design calls for two lossless classes: one for RDMA collectives, one for storage.

```
   STEP 1 — headroom per port per lossless priority (§8 formula)
      xoff = (301 + 2.16 × 100) × 400 + 2.72 × 4096
           = 517 × 400 + 11,141
           = 217,941 B  ≈ 213 KiB

   STEP 2 — total headroom demanded
      64 ports × 2 priorities × 213 KiB  =  26.6 MiB

   STEP 3 — compare with what exists
      26.6 MiB demanded  vs  16 MiB of shared buffer
      → INFEASIBLE as specified.  And note this is headroom ALONE:
        it reserves nothing for actual queueing.

   STEP 4 — the four levers, and what each buys
      (a) one lossless class instead of two   → 13.3 MiB.  Fits, but
          storage I/O now shares a lossy class or contends with the
          collective — reintroducing the straggler problem 09.1 §9's
          off-rail NIC was meant to solve.
      (b) shorten the reach to 30 m           → (301+64.8)×400 + 11,141
                                              = 157,461 B ≈ 154 KiB
                                              → 64×2×154 KiB = 19.2 MiB.
          Still over.  Reach helps but the FIXED term dominates at 400G.
      (c) reduce MTU 4096 → 1024              → saves 2.72×3072 ≈ 8 KB
                                                per port: ~1 MiB total.
          Negligible, and it costs packet-rate efficiency. Not a lever.
      (d) buy a deeper-buffer switch          → the actual answer.

   CONCLUSION: at 400 Gb/s with 100 m reach, a 16 MB shallow-buffer
   switch cannot support two lossless classes.  Either the switch
   changes, the reach changes, or the lossless class count changes.
   State it that way in a design review — with the arithmetic — and
   the conversation stops being about opinions.
```

**Part B — two clusters, two different verdicts.**

**Scenario A: a 512-GPU research cluster.** Mixed and bursty workloads, a three-person platform team, latency-sensitive interactive use, and a mandate to maximise out-of-box performance with minimal fabric-engineering headcount.

**Verdict: InfiniBand.** The deciding axes are *tuning risk* and *headcount*, not latency. At 512 GPUs the vendor-lock and premium-optics penalties are small in absolute dollars and are dominated by engineer-time: this team cannot run a PFC/ECN qualification project, cannot chase a deadlock incident at 2 a.m., and certainly cannot build and validate a custom admission scheme. Credit-based flow control means the failure class in §§9–10 does not exist for them, and the Part A arithmetic — where a small team gets it quietly and silently wrong — never has to be done. If the work is all-reduce-heavy, in-network reduction is a further direct win. What RoCE saves is optics cost and reuse of an Ethernet organisation they do not have. **Buy determinism with money, not with headcount.**

**Scenario B: a 24,000-GPU production trainer.** A large network organisation already fluent in EVPN/BGP, a multi-year build where optics and switch spend and second-sourcing dominate TCO, and a workload dominated by one well-characterised collective pattern.

**Verdict: RoCEv2** — with Spectrum-X as the middle path if leadership wants to buy down tuning risk, and Meta-style library-level admission only with genuine engineering depth behind it. The deciding axes are *optics capex*, *radix* and *second-sourcing*. At 24K endpoints the optics bill and the negotiating position are enormous, and this organisation has the muscle to run the qualification: the Part A arithmetic per switch model and link speed, a consistent DSCP-to-priority map at every hop, PFC watchdogs, `rx_prio*_buf_discard` as a paging alert, and re-validation on firmware changes. The tuning risk that disqualified the 512-GPU team is affordable here and, per §13, partly retirable. **The senior signal is that the same two technologies produce opposite verdicts once the axes are weighted by cluster size, team skills and TCO structure — and that a capable team's option set is not limited to "IB or RoCE."**

## Practice (feeds the deliverable)

Produce the **IB-vs-RoCE decision section** of the Network Architecture Read, with the buffer arithmetic that backs it.

**Task.** (1) Build the decision table. (2) Size the headroom for a named switch and link. (3) Pick a winner for two clusters with *different* verdicts and justify each.

**Requirements / acceptance:**

1. A decision table covering at least these axes for **both** InfiniBand and RoCEv2 (a Spectrum-X column is optional and rewarded): how losslessness is achieved; which configuration surface can be wrong and what it breaks when it is; small-message latency and tail behaviour; load balancing; control plane; in-network reduction; optics and radix economics; second-sourcing; skills.
2. A **headroom calculation** for a concrete switch, link rate, cable length and MTU, using the §8 formula, carried through to a per-switch total and compared against a real buffer size. State whether the design is feasible and, if not, which of the four levers you would pull.
3. **Two verdicts that differ**, each justified by naming *which axes drove it* — not "IB is faster." At least one justification must reference a number you computed rather than a number you quoted.
4. Name at least **two RoCE failure modes** with their mechanisms and mitigations: choose from head-of-line blocking / congestion spreading, PFC pause storm, PFC deadlock, ECMP hash polarisation, DSCP-map inconsistency across vendors.
5. State the **ECN-leads-PFC-trails invariant** in mechanical form (`Kmin < Kmax < XOFF`, with a control-loop round trip of separation) and say what specifically goes wrong in each direction if it is violated.
6. If you cite Meta's DCQCN-disabled fabric, state the **actual substitute mechanism** (receiver-driven traffic admission co-designed with the collective library) rather than the shorthand "they turned off congestion control," and note the enabling conditions (deep-buffer switches, engineering scale).
7. Name **three counters** you would put on a dashboard to tell which mechanism is firing, and what each one's rising value means.

One page plus the arithmetic. A reviewer should be able to check every claim against a source or re-run every calculation.

## Common pitfalls

- **Saying "RoCE needs lossless Ethernet because RDMA is sensitive to loss."** True but empty. The mechanical answer is that reliable-connected transport recovers with go-back-N (or, on some real hardware, go-back-0), so a single drop discards up to a full bandwidth-delay product — about 500 KB at 400 Gb/s and 10 µs RTT — and in the go-back-0 case a modest deterministic loss rate can drive goodput to exactly zero. The fabric is engineered so that recovery essentially never runs, because recovery is a cliff.
- **Treating PFC's eight priorities as eight usable lossless classes.** Headroom is the binding constraint. §8's arithmetic gives ~213 KiB per port per priority at 400 Gb/s over 100 m; a 64-port switch cannot fund eight of those out of any buffer that exists today. Real deployments run one or two, which is exactly what Microsoft reported doing on 9–12 MB switches.
- **Getting the ECN/PFC ordering backwards, or setting it without leaving room for the control loop.** If `Kmin` sits at or above XOFF, PFC fires before any sender is told to slow down and the fabric runs permanently on the blunt instrument. If `Kmin` is far too low, everything is marked and senders sit at their floor. And even a correctly ordered pair fails if the gap between `Kmax` and XOFF is smaller than one control-loop round trip, because the sender's reaction cannot arrive in time.
- **Assuming PFC deadlock requires exotic routing.** The documented production deadlock came from ordinary Ethernet flooding of a packet whose ARP entry was incomplete, sending lossless traffic down paths routing would never have chosen and closing a dependency cycle. It did not clear after restarting every server. Deadlock-free routing is necessary but not sufficient; "do not flood the lossless class" and a PFC watchdog are also required.
- **Treating "Meta disabled DCQCN" as "Meta ran with no congestion control."** They replaced a generic NIC-resident algorithm with a receiver-driven admission scheme co-designed into the collective library — a different mechanism at a different layer, with more application context, enabled by deep-buffer switches and a very large network organisation. Repeating the headline is a tell that you read the summary and not the work.
- **Believing InfiniBand has no congestion control, or no knobs.** It has both: FECN/BECN marking, a congestion control table, and a full set of switch and channel-adapter parameters written by `ibccconfig` (`marking_rate`, `threshold`, `ccti_timer`, `ccti_increase`, `ccti_min`, and the CCT itself). The real distinction is that on InfiniBand, mistuned congestion control costs throughput, whereas on RoCE, mistuned PFC costs losslessness.
- **Believing InfiniBand is lossless at every layer.** The *link* layer is lossless by construction, but endpoint-level loss is still possible and VL15 management traffic is explicitly not flow-controlled — which is what `VL15Dropped` counts. "Lossless fabric" describes the wire.
- **Quoting a single "RoCE gets 90–95% of IB" figure as a law.** Any such number is scoped to a workload, a tuning quality and a generation. State the scope, or do not state the number.
- **Ignoring the DSCP-to-priority map across vendor boundaries.** The most common way a "lossless" fabric turns out not to be: the NIC marks one DSCP value, one switch maps it to the lossless priority, and another maps it to a lossy one. Traffic keeps flowing; losslessness quietly does not exist on that hop. Audit the whole path, not the endpoints.

## Self-check

**(a) What is a PFC deadlock, where does it come from, and how do you prevent it?**

**Answer:** A cyclic pause dependency: switch A's ingress buffer for the lossless priority is full so it pauses B, B's fills so it pauses C, C pauses D, D pauses A. Every switch is waiting for the next to drain and none can, so the cycle is stable and the fabric region wedges — in Microsoft's production incident it did not clear even after restarting every server. It comes from PFC being *coarse* (per class) and *hop-by-hop* combined with a topology or forwarding behaviour that permits a cyclic buffer dependency. The documented real-world trigger was not exotic routing but ordinary Ethernet **flooding**: a packet whose ARP entry was incomplete got flooded to every port, sending lossless traffic down paths routing would never have chosen and closing the cycle. Prevention is layered: deadlock-free (for example up/down) routing for the lossless class; do not flood lossless traffic — drop it if its ARP entry is incomplete; run PFC watchdogs that break a stuck cycle by disabling losslessness for the smallest possible scope; and strategically keep PFC out of steady state by tuning ECN/DCQCN so pauses are rare.

**(b) Why does DCQCN exist if you already have PFC?**

**Answer:** Because PFC is too blunt to serve as a congestion controller. It pauses an *entire priority class* on a link, hop by hop, with no knowledge of which flow is responsible — so one hot receiver stalls every innocent flow sharing that class (head-of-line blocking), the backpressure spreads outward through the fabric in microseconds to milliseconds, and cyclic pause dependencies can deadlock. DCQCN is the fine-grained, end-to-end, per-flow controller: a congested switch marks packets with ECN CE above a queue threshold, the receiving NIC turns those marks into congestion notification packets (rate-limited by `min_time_between_cnps`), and the sending NIC reduces that specific flow's rate multiplicatively using an alpha estimate, then recovers through fast recovery, additive increase and hyper-increase. The intended regime is that DCQCN does all the everyday work and **PFC only ever fires as a last-resort safety net for microbursts the control loop was too slow to catch**. If PFC is firing in steady state, the ECN/DCQCN configuration is wrong.

**(c) How much buffer does PFC need on a 400 Gb/s port with 100 m links, and why does the answer matter?**

**Answer:** About **213 KiB per port per lossless priority**. Using the formula the mlx5 driver implements — `xoff = (301 + 2.16 × cable_len_m) × speed_Gbps + 2.72 × MTU_bytes` — a 400 Gb/s port with 100 m links and a 4096 B MTU needs `(301 + 216) × 400 + 11,141 = 217,941 bytes`. The two constants are physics plus margin: 2.16 B/m/Gb/s is round-trip propagation (1.25 B/m/Gb/s ideal) with about a 1.7× margin, and 301 B/Gb/s is a fixed reaction-time allowance worth roughly 2.4 µs at 400 Gb/s. It matters because it multiplies: a 64-port switch needs 13.3 MiB for one lossless class and 26.6 MiB for two, against shallow-buffer switches that historically carried 9–12 MB *in total*. That is why real deployments run one or two lossless classes rather than eight, why long leaf-to-spine reach is expensive in buffer as well as in optics, and why deep-buffer switches are a real line item on an AI fabric.

**(d) Trace what happens, in time, when one receiver is oversubscribed on a PFC-enabled fabric.**

**Answer:** The leaf's egress port toward the receiver saturates; its ingress buffers for that priority cross XOFF within microseconds and it pauses the spines. Each spine stops forwarding that priority toward that leaf, so its own ingress buffers fill — including with traffic for unrelated leaves, because the pause is per-priority, not per-destination. That is head-of-line blocking. The spines then cross their own XOFF and pause every leaf beneath them, and those leaves pause their servers. Within microseconds to low milliseconds the blast radius has grown from one egress port to every server under those spines, and every flow sharing the lossless priority is stalled. The pathological version is a pause storm: a NIC with a broken receive pipeline emits pause frames continuously and the cascade never stops — which is why production fabrics run NIC-side and switch-side PFC watchdogs (stall detection on the order of 100 ms at the NIC, 200 ms at the switch) that disable losslessness for the smallest possible scope rather than let it propagate.

**(e) What is the difference between InfiniBand's flow control and PFC, in mechanism rather than in outcome?**

**Answer:** InfiniBand is **credit-based and closed-loop**: the receiver continuously advertises a credit limit (FCCL) in flow-control packets, the sender tracks how much it has sent (FCTBS), and it may transmit a packet only if doing so keeps it within the advertised limit. A sender therefore *cannot* create an overflow — when credits run out it simply stops, and the backpressure is proportional and per virtual lane. PFC is **pause-based and open-loop**: the receiver lets the sender transmit freely, watches its own buffer, and shouts "stop" when occupancy crosses a watermark. Because that shout takes time to arrive and to be acted on, everything sent during that gray period must be absorbed by pre-reserved headroom, whose size depends on rate, reach and MTU. That difference is the root of almost every operational difference between the fabrics: credits have no threshold to misconfigure, no headroom to under-provision, and no per-class blast radius, while PFC has all three. The counter that reveals credit backpressure is `PortXmitWait`, which increments when a port has data to send but no credits.

**(f) Which SHARP version ships with which InfiniBand generation, and why does it belong in this decision?**

**Answer:** The mapping usually quoted is **SHARPv3 on Quantum-2 (NDR, 400G)** and **SHARPv4 on Quantum-X800 (XDR, 800G)**, with v4 adding FP8 support and ReduceScatter/ScatterGather aggregation. *(Flagged: this mapping is carried from vendor documentation that was not reachable from this environment in this pass — verify before quoting it in a written procurement document.)* It belongs in the IB-versus-RoCE decision because it is not a latency argument but an **algorithmic** one: in-network reduction changes how many bytes a collective puts on the wire at all, which can shift the fabric term in 09.1's and 09.2's models and therefore the oversubscription ratio a workload can tolerate. It is the strongest single reason an all-reduce-bound workload might choose InfiniBand despite the cost and lock-in axes pointing the other way, and it is the subject of 09.5.

**(g) You are handed a RoCE fabric that "is configured lossless" and a job that is 30% slower than expected. What do you check, in what order, and what does each check rule out?**

**Answer:** Work from the most damaging finding outward. **First, `ethtool -S` for `rx_prio<n>_buf_discard` and `rx_prio<n>_cong_discard` on the RDMA priority.** Non-zero means the fabric is dropping RDMA traffic — losslessness does not exist on this path — and per §1 each drop is a go-back-N or go-back-0 event. That usually points at headroom (re-run §8's arithmetic for this rate and reach) or at a priority-to-buffer misconfiguration. **Second, the DSCP-to-priority map at every hop**, NIC and both switch tiers: a mismatch at a vendor boundary silently puts RDMA traffic into a lossy queue, and it is the single most common cause of the first symptom. **Third, `tx_prio<n>_pause` and `rx_prio<n>_pause`.** Steady-state pausing means you are running on the backstop rather than on the control loop: check that `Kmin < Kmax < XOFF` with a control-loop round trip of separation, and that ECN is actually enabled on both the switch queue and the NIC's notification point. Rising `rx_*_pause` on hosts that are not themselves congested is head-of-line blocking, which tells you the problem is elsewhere in the fabric. **Fourth, `rx_prio<n>_marked`.** Marks rising with no pauses is the healthy regime; no marks and no pauses with a slow job means the fabric is not the problem — go back to 09.1's GPUDirect check and 09.2's path-entropy question, because a single-QP flow colliding on one uplink looks exactly like congestion and is not.

## Connections & what's next

This lesson is the direct sequel to 09.3: RC transport guarantees delivery of what arrives, and this lesson is entirely about the separate, upstream question of how the fabric avoids dropping anything — intrinsically for InfiniBand, by engineering for RoCEv2, or, at the far end of the design space, by moving congestion avoidance into the collective library altogether. It also closes a loop with 09.2: that lesson told you how much bandwidth exists at each tier and how evenly routing can spread traffic across it; this one tells you how reliably that bandwidth can be *used* without collapsing under congestion. Together they are the two halves of "can this fabric actually run a training job at line rate."

Next is **09.5, GPUDirect over the fabric plus SHARP**. This lesson introduced in-network reduction as a decision axis and gave you the generation mapping; the next goes inside the mechanism — how a switch performing the reduction restructures the collective's algorithm and therefore its bandwidth requirement, which collective shapes benefit and which do not, and how GPUDirect RDMA's same-root-complex precondition (09.1, 09.3) extends fabric-wide as "same rail." Where this lesson answered *which fabric and why*, 09.5 answers *how do I prove the fabric is being used correctly for a specific job*.

## References & further reading

**Source-access note for this pass.** This environment's egress proxy blocks IEEE, ACM, USENIX, arxiv.org, docs.nvidia.com, engineering.fb.com and the InfiniBand Trade Association's site. The entries marked **verified** below were fetched and read directly, or read from source code cloned from GitHub, while writing this lesson. Entries marked **not re-verified in this pass** are carried with their original attribution and should be treated as citations to check rather than as facts confirmed here. In particular, the **IBTA specification and the IEEE 802.1Qbb/802.1Qaz standards texts were not reachable**, so their content is grounded below in open-source implementations of those standards rather than in the standards documents themselves, and any figure that could only come from the standards text (the 64-byte credit block, the 2,048-credit-per-VL ceiling) is flagged in place as indicative.

**Verified against source in this pass**

- **Guo et al., Microsoft, [RDMA over Commodity Ethernet at Scale](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/rdma_sigcomm2016.pdf), SIGCOMM 2016** — fetched and read directly. The source for: DSCP-based PFC and the specific failures of VLAN-based classification; the XOFF/XON/headroom model and the statement that headroom is determined by MTU, PFC reaction time and above all propagation delay; 9 MB and 12 MB shallow-buffer switches supporting only two lossless classes with up to 300 m links; the NIC PFC pause-frame storm, its six-hop propagation and the NIC (100 ms) and switch (200 ms) watchdogs; the PFC deadlock caused by flooding on an incomplete ARP entry, its persistence across a full server restart, and the drop-on-incomplete-ARP fix; RDMA transport livelock under go-back-0 at a 1-in-256 drop rate; RoCEv2's fixed destination UDP port 4791 with per-QP randomised source port; and the deployment's link speeds and cable distances.
- **Linux kernel source** (`github.com/torvalds/linux`) — read directly. `drivers/infiniband/core/verbs.c` for the IB speed table (SDR 2.5 … XDR 200, in 0.1 Gb/s units) and `ib_port_attr_to_speed_info()`'s rate = speed × width rule; `include/uapi/linux/dcbnl.h` for `struct ieee_pfc` (`pfc_cap`, `pfc_en`, `mbc`, `delay`, `requests[]`, `indications[]`), `struct ieee_ets` (`prio_tc`, `tc_tx_bw`, `tc_tsa`, `willing`), the TSA identifiers (0 strict, 1 credit-based shaper, 2 ETS, 255 vendor), `IEEE_8021QAZ_MAX_TCS = 8` and `struct dcbnl_buffer`; `drivers/net/ethernet/mellanox/mlx5/core/en/port_buffer.c` for the headroom formula `xoff = (301 + 2.16·len_m)·speed_Gbps + 2.72·MTU`, the 40 Gb/s speed floor, `xon = xoff_threshold − max_mtu` and `MINIMUM_MAX_MTU 9216`; `.../mlx5/core/en_dcbnl.c` for `MLX5E_MAX_CABLE_LENGTH 1000`, the ETS 100% bandwidth cap and the PCP/DSCP trust states; `.../mlx5/core/en_stats.c` for the exact `ethtool -S` counter names (`rx_prio%d_pause`, `..._pause_duration`, `..._pause_transition`, `rx_prio%d_buf_discard`, `rx_prio%d_cong_discard`, `rx_prio%d_marked`, `tx_pause_storm_warning_events`, `tx_pause_storm_error_events`); and `drivers/infiniband/hw/mlx5/cong.c` for the DCQCN debugfs parameter set under `/sys/kernel/debug/mlx5/<pci>/cc_params/`.
- **rdma-core** (`github.com/linux-rdma/rdma-core`) — read directly. `infiniband-diags/ibccconfig.c` for InfiniBand's congestion-control surface: SwitchCongestionSetting (`control_map`, `victim_mask`, `credit_mask`, `threshold`, `packet_size`, `cs_threshold`, `cs_return_delay`, `marking_rate`), CACongestionSetting (`port_control`, `control_map`, `ccti_timer`, `ccti_increase`, `trigger_threshold`, `ccti_min`) and the CongestionControlTable; `libibmad/fields.c` and `infiniband-diags/perfquery.c` for `PortXmitWait` and `VL15Dropped` in the port-counter layouts; `infiniband-diags/ibstat.c` for the port output fields used in the rate discussion.
- **prometheus/procfs** (`sysfs/net_class_ecn.go`) — read directly for the exact `/sys/class/net/<iface>/ecn/roce_rp/` and `roce_np/` attribute names used in §11.
- **archjeb/pfctest** (`pfctest.py`) — read directly for the PFC frame layout reproduced in §7: destination MAC `01:80:C2:00:00:01`, EtherType `0x8808`, opcode `0x0101`, a class-enable vector, eight 2-byte per-class time fields in quanta where one quantum is the time to transmit 512 bits at the link rate, padded to 60 bytes plus a 4-byte FCS.

**Cited, not re-verified in this pass**

- Meta / SIGCOMM 2024, [RDMA over Ethernet for Distributed Training at Meta Scale](https://engineering.fb.com/wp-content/uploads/2024/08/sigcomm24-final246.pdf) and the [engineering blog version](https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/) — cited for the DCQCN-disabled, PFC-only production fabric, receiver-driven traffic admission co-designed with the collective library, deep-buffer switch choice, and destination-QP-aware ECMP improving AllReduce by up to ~40%.
- Zhu et al., [Congestion Control for Large-Scale RDMA Deployments](https://dl.acm.org/doi/10.1145/2785956.2787484), SIGCOMM 2015 — the DCQCN paper: the origin of the CP/NP/RP structure, the alpha-based rate decrease, fast recovery / additive increase / hyper increase, and the `Kmin = 5 KB`, `Kmax = 200 KB`, `Pmax = 1%` starting values quoted in §11 at 40 Gb/s. **Not fetchable from this environment**; those three values come from secondary summaries and must be verified against the paper and against your switch vendor's guidance before use.
- Alibaba, [HPN: A Data Center Network for Large Language Model Training](https://dl.acm.org/doi/10.1145/3651890.3672265), SIGCOMM 2024 — cited for ECMP hash polarisation as a first-class failure mode and the two-tier dual-plane topological fix.
- Microsoft Azure, [Striking the right balance: ECN and PFC thresholds for AI clusters](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/striking-the-right-balance-ecn-and-pfc-thresholds-for-ai-clusters/4468629) — cited for a production account of the ECN-leads / PFC-trails relationship and the instability that follows from inverting it.
- Arista / Broadcom, [Lossless network for AI/ML/storage/HPC with RDMA — RoCE deployment guide](https://www.arista.com/assets/data/pdf/Broadcom-RoCE-Deployment-Guide.pdf) — cited for vendor-primary starting values for headroom, ECN thresholds and buffer allocation per switch family; the natural companion to §8's arithmetic when you need per-model numbers.
- NVIDIA — Quantum-2 / Quantum-X800 and SHARP generation mapping (SHARPv3 on Quantum-2, SHARPv4 on Quantum-X800 with FP8 and ReduceScatter/ScatterGather), and Spectrum-4 SN5600 radix (64 × 800G or 128 × 400G, 51.2 Tb/s). Vendor documentation was **not reachable** from this environment; verify before quoting in writing.
- InfiniBand Trade Association — the IBTA architecture specification defines the credit-based flow-control fields (ABR, FCCL, FCTBS), the credit block size and the per-VL credit ceiling used as indicative figures in §3, plus the FECN/BECN congestion-control architecture whose knobs are enumerated in §4. **The specification was not reachable from this environment and is therefore not relied upon**; the mechanism as described here is grounded in the rdma-core tooling that configures and reads it.

**Deeper dives**

- Glenn K. Lockwood, [Networking for LLM training](https://www.glennklockwood.com/garden/networking-for-LLM-training) — an independent, sceptical, first-principles treatment of why LLM fabrics are built the way they are. Read for the reasoning rather than the conclusions. Not re-verified in this pass.
- The mlx5 driver's `port_buffer.c` and `en_dcbnl.c` themselves — the fastest way to see exactly what a NIC does when you set a PFC or ETS parameter, and the best antidote to vendor documentation that describes intent rather than behaviour.
