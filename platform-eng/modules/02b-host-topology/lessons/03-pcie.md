---
lesson: "02b.3"
title: "PCIe at operational depth"
module: "02b"
concept: "PCIe at operational depth"
status: not-started
est_time: "7h"
prev: "02-memory-subsystem.md"
next: "04-server-architecture-8gpu.md"
artifacts: []
sources: 12
---
# 02b.3 · PCIe at operational depth

> **Concept.** A PCIe link silently trains to a lower generation or width and the GPU still "works" — at a fraction of spec. Learn to read the trained-vs-max link, the secondary error signals that precede it, and catch it before it costs a training run.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)
>
> **Advanced Learning** — [LnkCap vs LnkSta](../../../learning/03-pcie.html): the generation table with code efficiency, and the exact signature that separates a degraded link from the topology you built. Optional visual companion; this lesson stays the source of truth.

## Where this fits

Lesson 02 ended with a loose thread. A local pinned host→device copy measured **54.2 GB/s** against a PCIe Gen5 x16 ceiling of **63.0 GB/s** — 86% — and the text said "the missing 14% is PCIe protocol overhead, which lesson 03 breaks down line by line." This lesson pays that debt, and then goes further: it teaches you where the 63.0 GB/s itself comes from, what makes a link fail to reach it, and how to tell a configuration fact from a hardware fault.

That matters because every bandwidth number in lessons 01 and 02 silently assumed a healthy link. `min(63, 38)` is only the right arithmetic if the link actually trained to Gen5 x16. If it trained to Gen4 x8 — which produces no error, no log line, and no dashboard change — the arithmetic is `min(15.8, 38)` and every conclusion built on top of it is wrong. Lesson 04 then hands you the reference topology so you know where each of these links is *supposed* to go.

## Why this matters

An H100 wants a PCIe **Gen5 x16** link to its host: 63 GB/s in each direction, of which you can realistically use about 54. That link carries every byte the GPU reads from host memory, every staging buffer, every checkpoint write, and — on a well-placed node — GPUDirect RDMA and GPUDirect Storage traffic.

If it trains at **Gen3 x8** instead, you have **7.9 GB/s** each direction. That is **8× less**, and nothing in your monitoring stack will tell you. `nvidia-smi` reports 100% utilisation because SM occupancy is high. DCGM reports the card hot and drawing power. The card *looks* busy. It is busy — stalling on a starved host link, waiting on data arriving eight times too slowly.

This is not hypothetical. Meta's infrastructure engineering team has published in detail on exactly this fault class. Their PCIe fault-management program calls out **bad (down-negotiated) link speed** — "usually running at 1/2 or 1/4 of the expected speed" — and **bad link width** — "running at 1/2, 1/4, or even 1/8 of the expected link width" — as distinct fault categories, alongside PCIe corrected and uncorrected error-rate checks. Their automated detect → diagnose → remediate → repair pipeline, built around the open-source **PCIcrawler** tool, has added coverage that "fixed several thousand servers and server components," and continues to detect and repair PCIe faults on **hundreds of servers every week** at fleet scale. That is the concrete answer to "is this a real job function" — it is a standing, staffed program at every hyperscaler and neocloud.

Link degradation comes from a card that did not fully mate on reseat, a marginal or failing retimer, an oxidised contact, a marginal riser, thermal effects on the PCIe PHY, a BIOS that bifurcated a slot differently than you expected, or — as you will see below — a driver regression with no hardware fault at all. The fix is trivial once you *see* it. Seeing it is the skill. At a neocloud you will be handed fleets of boxes you did not build, and "is this link actually trained to spec?" is a question you must answer from the CLI in under a minute per card.

## What's new here (calibration)

You know PCIe exists and that GPUs sit on it. From CKA and on-prem work you have read `lspci` output for NIC and HBA enumeration, and lesson 01 gave you the hierarchy vocabulary (host bridge, root port, switch, endpoint) and the `PIX`/`PXB`/`PHB`/`NODE`/`SYS` codes. What is new:

- **Where the bandwidth number comes from, and where it goes.** Raw GT/s → line encoding → TLP framing and headers → DLLP acknowledgements → `MaxPayload`/`MaxReadRequest` → the number `bandwidthTest` prints. You will be able to derive 54 GB/s from 32 GT/s rather than memorising it.
- **Link training as a state machine.** Every PCIe link starts at Gen1 regardless of capability and negotiates up through the LTSSM's Recovery state. Knowing that changes how you read a downgrade: a link stuck at 2.5 GT/s never got out of its initial state, which is a different fault from one that reached Gen4 and fell back.
- **Four registers, not two.** `LnkCap` and `LnkSta` are the famous pair, but `LnkCap2` (supported speed vector) and `LnkCtl2` (target speed) are what distinguish "this hop cannot do better" from "something told it not to."
- **AER as an early-warning counter, not an error log.** Correctable errors are *corrected* — the link retries and the transaction succeeds. A climbing correctable count on a link that still reads full spec is the earliest signal you will get, and it is what fleet tooling actually alerts on.
- **`MaxPayload` and `MaxReadRequest` as tunables with a mechanism**, including why Linux sets MPS conservatively by default and what `pci=pcie_bus_perf` changes.
- **Bifurcation and retimers as physical-layer facts** you can reason about from unit-interval arithmetic rather than accept as vendor magic.

## Core concepts

### 1. What PCIe actually is: a layered packet protocol, not a bus

"PCI Express" inherits its name from a parallel bus, and that inheritance misleads. PCIe is a **point-to-point, full-duplex, packet-switched serial link** with a three-layer protocol stack. Every property that matters operationally falls out of that.

```
   SOFTWARE                     memory reads/writes, config accesses, MSI-X
        │
   ┌────▼─────────────────────────────────────────────────────────────────┐
   │ TRANSACTION LAYER                                                    │
   │   builds TLPs (Transaction Layer Packets): header + optional payload │
   │   header is 3 DW (12 B) for 32-bit addresses, 4 DW (16 B) for 64-bit │
   │   payload capped by MaxPayloadSize (MPS)                             │
   │   reads capped by MaxReadRequestSize (MRRS)                          │
   │   credit-based flow control — a sender may only transmit if the      │
   │   receiver has advertised buffer credits                             │
   └────┬─────────────────────────────────────────────────────────────────┘
        │
   ┌────▼─────────────────────────────────────────────────────────────────┐
   │ DATA LINK LAYER                                                      │
   │   adds a 2-byte sequence number and a 4-byte LCRC to every TLP       │
   │   receiver checks LCRC; on failure it sends a NAK and the sender     │
   │   REPLAYS from its retry buffer  ← this is why a marginal link is    │
   │   generates DLLPs: ACK / NAK / flow-control credit updates             slow, not broken
   └────┬─────────────────────────────────────────────────────────────────┘
        │
   ┌────▼─────────────────────────────────────────────────────────────────┐
   │ PHYSICAL LAYER                                                       │
   │   framing tokens (STP/END, or the 4-byte STP token in Gen3+)         │
   │   line encoding: 8b/10b (Gen1-2), 128b/130b (Gen3-5), 1b/1b + FLIT   │
   │     with FEC (Gen6+)                                                 │
   │   striping across N lanes, scrambling, equalisation                  │
   │   LTSSM: the state machine that trains, retrains and powers the link │
   └──────────────────────────────────────────────────────────────────────┘
```

Two consequences to hold on to:

- **A bad link does not fail; it retries.** LCRC failures cause NAK-and-replay at the data link layer. The transaction still completes correctly. The only observable is that it took longer, plus a correctable-error counter ticking somewhere nobody is looking. **This is the entire reason PCIe degradation is silent.**
- **Credit-based flow control means a slow receiver throttles the sender without any error.** If a device's receive buffers are small relative to the round-trip time, throughput drops even on a perfect link. `MaxReadRequestSize` is one of the knobs that changes how much a requester can have outstanding, which is Little's Law from lesson 02 showing up again at the link layer.

### 2. Rate, encoding, and the raw number

PCIe bandwidth is `per-lane transfer rate × lane count`, quoted **per direction** (links are full-duplex; double it if someone is quoting aggregate). The per-lane rate is the signalling rate multiplied by the line code's efficiency.

**Why line coding exists at all:** the receiver recovers its clock from the data stream, so the encoding must guarantee enough transitions and keep the DC balance reasonable. Gen1/2 used **8b/10b**, transmitting 10 symbols for every 8 data bits — a flat **20% tax**. Gen3 replaced it with **128b/130b**: a 2-bit sync header per 128-bit block, a **1.54% tax**, with scrambling doing the transition-density work instead. Gen6 abandons block coding entirely for **1b/1b** signalling over PAM4, and pays instead for **FEC and CRC inside a fixed 256-byte FLIT**.

| Gen | Year | Raw rate/lane | Signalling | Line code | Code efficiency | **GB/s per lane per direction** | **x4** | **x8** | **x16** |
|---|---|---|---|---|---|---|---|---|---|
| 1.0 | 2003 | 2.5 GT/s | NRZ | 8b/10b | 80.0% | 0.250 | 1.0 | 2.0 | **4.0** |
| 2.0 | 2007 | 5.0 GT/s | NRZ | 8b/10b | 80.0% | 0.500 | 2.0 | 4.0 | **8.0** |
| 3.0 | 2010 | 8.0 GT/s | NRZ | 128b/130b | 98.46% | 0.985 | 3.94 | 7.88 | **15.75** |
| 4.0 | 2017 | 16.0 GT/s | NRZ | 128b/130b | 98.46% | 1.969 | 7.88 | 15.75 | **31.51** |
| 5.0 | 2019 | 32.0 GT/s | NRZ | 128b/130b | 98.46% | 3.938 | 15.75 | 31.51 | **63.02** |
| 6.0 | 2022 | 64.0 GT/s | PAM4 | 1b/1b + FLIT | 94.5% (242/256) | 7.563 | 30.25 | 60.50 | **121.0** |
| 7.0 | 2025 | 128.0 GT/s | PAM4 | 1b/1b + FLIT | 94.5% | 15.125 | 60.50 | 121.0 | **242.0** |

*(Rates and signalling from PCI-SIG specification announcements. PCI-SIG markets Gen6 x16 as "256 GB/s bidirectional" and Gen7 x16 as "512 GB/s bidirectional" — those are the raw `rate × lanes ÷ 8` figures doubled for both directions, before FLIT overhead. The per-lane column here applies the FLIT accounting; see §3.)*

Two facts to be able to state instantly:

- **Each generation doubles the per-lane rate; halving the width halves bandwidth.** So Gen5 x16 vs Gen3 x16 is 4×; Gen5 x16 vs Gen3 x8 is **8×**.
- **The `LnkSta` speed field is the raw GT/s number, not the generation.** `2.5` = Gen1, `5` = Gen2, `8` = Gen3, `16` = Gen4, `32` = Gen5, `64` = Gen6. Train yourself to read the GT/s and map it, because that is what the tool prints.

**The Gen6 FLIT accounting, since it is the first structural change in fifteen years.** Gen6 sends fixed 256-byte flow control units regardless of TLP boundaries. Each 256-byte FLIT carries **242 bytes of payload space, 8 bytes of CRC, and 6 bytes of FEC**. Of the 242, 6 bytes are reserved for data-link (DLLP-equivalent) information, leaving **236 bytes for TLP content**. So:

```
  raw byte rate per lane      = 64 GT/s ÷ 8 bits/byte           = 8.000 GB/s
  after FLIT CRC+FEC          = 8.000 × (242 ÷ 256)             = 7.563 GB/s
  after the in-FLIT DLP field = 8.000 × (236 ÷ 256)             = 7.375 GB/s

  x16, per direction: 121.0 GB/s (242/256) or 118.0 GB/s (236/256)
```

The trade Gen6 makes: it *loses* 5.5% to FEC and CRC that Gen3–5 did not pay, but it *gains* by eliminating the 128b/130b sync headers and by absorbing DLLPs into the flit rather than interleaving them as separate packets. For small transfers Gen6 is substantially more efficient than the naive rate ratio suggests, because the per-TLP framing tax largely disappears.

### 3. Where the raw number goes — the full loss budget

This is the section that turns "63 GB/s" into "54 GB/s" with every byte accounted for.

```
  ANATOMY OF A PCIe Gen5 x16 LINK — FROM SIGNALLING TO USABLE BYTES

  ┌─────────────────────────────────────────────────────────────────────────┐
  │  16 LANES, each a differential pair per direction (64 wires total)       │
  │                                                                         │
  │   lane 0  ──▶ ══════════════════════════════════ 32 GT/s NRZ            │
  │   lane 1  ──▶ ══════════════════════════════════ 32 GT/s                │
  │     ...                                          (data is STRIPED       │
  │   lane15  ──▶ ══════════════════════════════════  byte-wise across      │
  │                                                    all 16 lanes)        │
  └─────────────────────────────────────────────────────────────────────────┘
        │
        │  32 GT/s × 16 lanes = 512 Gbit/s  =  64.00 GB/s of SYMBOLS
        ▼
  ┌─── LOSS 1: LINE ENCODING ────────────────────────────────────────────────┐
  │  128b/130b: 2 sync bits per 128-bit block                                │
  │  64.00 × (128/130) = 63.02 GB/s          ← this is the "63 GB/s" figure  │
  │  cost: 1.54%                                                             │
  └──────────────────────────────────────────────────────────────────────────┘
        ▼
  ┌─── LOSS 2: TLP FRAMING AND HEADERS ──────────────────────────────────────┐
  │                                                                          │
  │   one Memory Write TLP on the wire, 64-bit addressing, MPS = 256 B:      │
  │                                                                          │
  │   ┌────┬────┬──────────────┬────────────────────────────┬──────┐        │
  │   │STP │SEQ │  TLP HEADER  │        PAYLOAD             │ LCRC │        │
  │   │ 4B │ 2B │     16B      │         256B               │  4B  │        │
  │   └────┴────┴──────────────┴────────────────────────────┴──────┘        │
  │    └──────────── 26 B overhead ─────────────┘                            │
  │                                                                          │
  │   efficiency = 256 ÷ (256 + 26) = 90.8%                                  │
  │                                                                          │
  │   (3 DW / 12 B headers apply to 32-bit addressing; servers use 64-bit.   │
  │    Optional ECRC adds 4 B. Count 24–26 B as the working figure.)         │
  │                                                                          │
  │   THE SAME LINK AT DIFFERENT MPS:                                        │
  │      MPS =  128 B → 128 ÷ 154 = 83.1%                                    │
  │      MPS =  256 B → 256 ÷ 282 = 90.8%   ← common server default          │
  │      MPS =  512 B → 512 ÷ 538 = 95.2%                                    │
  │      MPS = 4096 B → 4096 ÷ 4122 = 99.4%  (rarely supported end to end)   │
  └──────────────────────────────────────────────────────────────────────────┘
        ▼
  ┌─── LOSS 3: DLLP TRAFFIC ON THE SAME WIRES ───────────────────────────────┐
  │  ACK/NAK and flow-control-credit DLLPs are 8 B each (6 B + framing)      │
  │  and share the link. With ACK coalescing, budget ~2–4% at large MPS,     │
  │  more at small MPS because there are more TLPs to acknowledge.           │
  │  Assume 3%: × 0.97                                                        │
  └──────────────────────────────────────────────────────────────────────────┘
        ▼
  ┌─── LOSS 4: DEVICE-SIDE REALITY ──────────────────────────────────────────┐
  │  DMA engine outstanding-request limits (bounded by MaxReadRequestSize),  │
  │  completion latency, host memory controller scheduling, IOMMU translation │
  │  Budget a further 1–3%.                                                   │
  └──────────────────────────────────────────────────────────────────────────┘
        ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  THE ARITHMETIC, END TO END:                                             │
  │    64.00 GB/s symbols                                                    │
  │  × 128/130   (line code)          = 63.02 GB/s   ← the spec number       │
  │  × 0.908     (TLP, MPS = 256 B)   = 57.22 GB/s                           │
  │  × 0.97      (DLLP)               = 55.51 GB/s                           │
  │  × 0.98      (device/DMA)         = 54.40 GB/s                           │
  │                                                                          │
  │  bandwidthTest pinned H2D, lesson 02: 54.2 GB/s.  The budget closes.     │
  └──────────────────────────────────────────────────────────────────────────┘
```

**So "86% of link peak" is not a fudge factor — it is the product of four identifiable losses**, three of which you can inspect and one of which (MPS) you can sometimes change. That is the difference between knowing a number and being able to defend it.

**Reading and changing MPS/MRRS.** Both live in the Device Control register of the PCIe capability:

```
$ sudo lspci -vvv -s 0000:1b:00.0 | grep -E 'DevCap:|DevCtl:' -A1
        DevCap: MaxPayload 512 bytes, PhantFunc 0, Latency L0s unlimited, L1 unlimited
        DevCtl: CorrErr+ NonFatalErr+ FatalErr+ UnsupReq+
                RlxdOrd+ ExtTag+ PhantFunc- AuxPwr- NoSnoop+
                MaxPayload 256 bytes, MaxReadReq 512 bytes
```

- **`DevCap: MaxPayload 512 bytes`** — what this device *can* do.
- **`DevCtl: MaxPayload 256 bytes`** — what it is *configured* to do. Linux sets MPS to a value safe for the whole hierarchy: **every device under a common root must use the same MPS**, so the effective value is the minimum capability along the path. One legacy 128-byte-MPS device under a root port drags everything below that root to 128 bytes and costs you 8 points of efficiency.
- **`MaxReadReq 512 bytes`** — the largest read a requester may issue. Bigger reads mean fewer round trips and more bytes in flight per outstanding transaction. Encoding is a 3-bit selector: `0`=128, `1`=256, `2`=512, `3`=1024, `4`=2048, `5`=4096 bytes.

Linux exposes the MPS policy as a boot parameter:

| `pci=` value | Behaviour |
|---|---|
| (default) | Keep firmware's MPS settings; only lower them if needed for safety |
| `pcie_bus_safe` | Set every device to the largest MPS supported by all devices below its root |
| `pcie_bus_perf` | Set device MPS to the largest its parent bus supports, and MRRS to the device's MPS — the throughput-oriented option |
| `pcie_bus_peer2peer` | Force MPS 128 everywhere, the safest value for peer-to-peer across hierarchies |
| `pcie_bus_tune_off` | Touch nothing |

If you need to change MRRS on a live system without a reboot, `setpci` writes the Device Control register directly. The PCIe capability offsets, which you should know because `lspci` cannot always decode everything:

| Register | Offset in PCIe capability | `setpci` form |
|---|---|---|
| Device Capabilities | 0x04 | `CAP_EXP+4.l` |
| **Device Control** (MPS bits 7:5, MRRS bits 14:12) | 0x08 | `CAP_EXP+8.w` |
| Device Status | 0x0A | `CAP_EXP+a.w` |
| **Link Capabilities** (`LnkCap`) | 0x0C | `CAP_EXP+c.l` |
| Link Control (`LnkCtl`) | 0x10 | `CAP_EXP+10.w` |
| **Link Status** (`LnkSta`) | 0x12 | `CAP_EXP+12.w` |
| Link Capabilities 2 (`LnkCap2`) | 0x2C | `CAP_EXP+2c.l` |
| **Link Control 2** (`LnkCtl2`, target speed bits 3:0) | 0x30 | `CAP_EXP+30.w` |

```
$ sudo setpci -s 0000:1b:00.0 CAP_EXP+12.w      # read LnkSta raw
1043
# bits 3:0  = 0x3 = 32 GT/s (Gen5);  bits 9:4 = 0x10 = 16 → x16.  Healthy.
```

**Correction worth internalising:** `CAP_EXP+12.w` is **Link Status**, not Link Control 2. Link Control 2 — the register that holds the *target* link speed and is therefore the one to check when a link is capped by software rather than by silicon — is at `CAP_EXP+30.w`. Getting this wrong means reading the right bits from the wrong register and concluding the opposite of the truth.

### 4. Link training: the LTSSM, and why every link starts at Gen1

A PCIe link is not configured; it is *negotiated*, by a hardware state machine at both ends, every time the link comes up. The **Link Training and Status State Machine (LTSSM)** has eleven top-level states: `Detect`, `Polling`, `Configuration`, `L0`, `L0s`, `L1`, `L2`, `Recovery`, `Loopback`, `Hot Reset`, `Disabled`.

```
   LINK TRAINING TIMELINE — what happens between power-on and full speed

   t=0   DETECT
         │  Transmitter looks for a receiver's DC termination on each lane.
         │  Lanes with no receiver detected are simply not part of the link.
         │  ← A DIRTY OR PARTIALLY-SEATED CONNECTOR LOSES LANES HERE.
         │    This is how you get x8 out of an x16 slot with no config change.
         ▼
   t≈1ms POLLING          ── ALWAYS AT 2.5 GT/s (Gen1). No exceptions. ──
         │  Exchange TS1/TS2 ordered sets. Establish bit lock and symbol lock.
         │  Agree on lane polarity inversion and lane-to-lane deskew.
         ▼
         CONFIGURATION
         │  Negotiate link width and lane numbering. The two ends agree on
         │  how many lanes are usable and which physical lane is logical lane 0.
         │  ← THE WIDTH IS DECIDED HERE, at Gen1 speed, on the lanes that
         │    survived Detect.
         ▼
   t≈5ms L0   (Gen1 x16, or x8, or whatever Configuration agreed)
         │  The link is now usable. TLPs flow. Speed is still 2.5 GT/s.
         ▼
         RECOVERY  ── the speed change happens HERE, not at boot ──
         │  Both ends advertise supported speeds (LnkCap2's speed vector).
         │  Highest common speed is selected, subject to LnkCtl2's target.
         │  For Gen3+ an EQUALISATION phase runs: transmitter presets are
         │  swept, the receiver reports which preset opens the eye best,
         │  and the two ends converge on per-lane TX coefficients.
         │  ← A MARGINAL CHANNEL FAILS EQUALISATION HERE and the link
         │    falls back to the last speed that worked.
         ▼
   t≈10ms L0 at the negotiated speed. This is what LnkSta reports.

   ─────────────────────────────────────────────────────────────────────────

   AND THEN, FOREVER AFTER — the link can re-enter Recovery at any time:

     L0 ──▶ Recovery  triggered by: an uncorrectable receiver error,
       ▲       │                     a replay-timer timeout,
       │       │                     exit from ASPM L0s/L1,
       └───────┘                     a software-initiated retrain
                                     (LnkCtl "Retrain Link" bit)

     Each Recovery entry may renegotiate speed and width.
     A link that oscillates between L0 and Recovery is FLAPPING, and it
     will show up as repeated speed changes rather than one stable value.
```

Three diagnostic conclusions fall straight out of this timeline, and they are what separate a useful read from a screenshot of `lspci`:

1. **A link stuck at 2.5 GT/s never made it out of the initial L0.** That is a *different* fault from a link that reached Gen4 and cannot get to Gen5. The first suggests the speed-change attempt never happened or was blocked (check `LnkCtl2`'s target speed, check ASPM, check for a driver that disabled it). The second suggests equalisation failed — a physical-channel problem.
2. **Width is decided at Gen1 speed, on whichever lanes responded to Detect.** So a *width* loss is nearly always a physical or configuration fact (bifurcation, a bent pin, a partially-seated card, an unpopulated riser), not a signal-integrity problem — signal integrity manifests as a *speed* loss, because it is the high-speed equalisation that fails.
3. **Speed and width are renegotiated on every Recovery entry, and ASPM exits cause Recovery entries.** So one `lspci` snapshot on an idle card is a sample of a moving target. Always confirm under load.

### 5. `LnkCap` vs `LnkSta` — the core skill, done properly

`lspci -vvv` prints the PCIe capability block. Four lines matter:

```
$ sudo lspci -vvv -s 0000:1b:00.0
1b:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
        ...
        Capabilities: [78] Express (v2) Endpoint, MSI 00
                DevCap: MaxPayload 512 bytes, PhantFunc 0, Latency L0s unlimited, L1 unlimited
                DevCtl: MaxPayload 256 bytes, MaxReadReq 512 bytes
                LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM not supported
                        ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp+
                LnkCtl: ASPM Disabled; RCB 64 bytes, Disabled- CommClk+
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 32GT/s, Width x16
                        TrErr- Train- SlotClk+ DLActive- BWMgmt- ABWMgmt-
                LnkCap2: Supported Link Speeds: 2.5-32GT/s, Crosslink- Retimer+ 2Retimers+ DRS-
                LnkCtl2: Target Link Speed: 32GT/s, EnterCompliance- SpeedDis-
```

Read them in this order:

- **`LnkCap`** — *this device's own advertised maximum*: `Speed 32GT/s, Width x16`. It is **not** the negotiated minimum of both ends; each end advertises its own capability independently. To find which hop in a chain caps a link, you must read `LnkCap` at *every* hop.
- **`LnkSta`** — what actually trained. Compare against `LnkCap` at both ends.
- **`LnkCap2: Supported Link Speeds: 2.5-32GT/s`** — the full speed vector. If this reads `2.5-16GT/s` on a card you believe is Gen5-capable, the *device* is the cap, not the channel.
- **`LnkCtl2: Target Link Speed: 32GT/s`** — the ceiling software has requested. If this reads `16GT/s` while `LnkCap2` supports 32, **something set it** — firmware, a driver, or a manual `setpci`. That is a configuration fault, not a hardware one, and it is the case people misdiagnose most often.
- **`LnkCap2: Retimer+ 2Retimers+`** — the link partner reports that retimers are present in the channel. Useful confirmation that you are looking at a retimed path (see §7).

A degraded link reads:

```
                LnkCap: Port #0, Speed 32GT/s, Width x16
                LnkSta: Speed 16GT/s (downgraded), Width x8 (downgraded)
```

`lspci` prints **`(downgraded)`** when `LnkSta` is below `LnkCap`. That word is your alarm, but do not depend on it: some pciutils versions omit it, and a link can be at full width with wrong speed or vice versa. **Always compare the four numbers yourself.**

**Two subtleties that produce false positives and false negatives:**

- **ASPM idle downtraining is not a fault.** With Active State Power Management enabled, an idle link may drop to a low-power state; on exit it re-enters Recovery and may retrain. A `2.5GT/s` reading on an idle card can be benign. Re-check under a real workload — or read `LnkCtl: ASPM Disabled`, which is what the transcript above shows and what most server GPUs ship with.
- **A reduced `LnkCap` is a configuration fact; a `LnkSta` below a full `LnkCap` is a fault.** If a Gen5 GPU sits behind a Gen4-only switch, the *switch's* downstream port has `LnkCap: Speed 16GT/s`, so the link trains to Gen4 and both ends report `LnkSta: Speed 16GT/s` with no `(downgraded)` marker — because neither end was capable of more on that segment. That is not a defect; it is the topology. Likewise a bifurcated root port advertises `LnkCap: Width x8`, so an x8 `LnkSta` there is expected. **The fault signature is specifically `LnkSta < LnkCap` on the same device.**

`lspci -vvv` needs root to read the full capability block; without it these lines are often blank or `<access denied>`.

**The NVIDIA-side view, which is faster for a sweep and includes the GPU driver's own opinion:**

```
$ nvidia-smi --query-gpu=index,pci.bus_id,pcie.link.gen.max,pcie.link.gen.current,pcie.link.width.max,pcie.link.width.current --format=csv
index, pci.bus_id, pcie.link.gen.max, pcie.link.gen.current, pcie.link.width.max, pcie.link.width.current
0, 00000000:1B:00.0, 5, 5, 16, 16
1, 00000000:43:00.0, 5, 5, 16, 16
...
7, 00000000:E3:00.0, 5, 4, 16, 16          ← Gen4 where Gen5 is expected

$ nvidia-smi -q -i 7 -d PCI | sed -n '/Link/,/Bridge/p'
        Link Width
            Max                       : 16x
            Current                   : 16x
        Device Current                : 4
        Device Max                    : 5
        Host Max                      : 5
        Max                           : 5
```

Note `Device Max` vs `Host Max`: the driver reports both ends' capabilities separately, which is `LnkCap` from each side without walking the tree by hand.

For live throughput rather than link state:

```
$ nvidia-smi dmon -s t -c 5
# gpu   rxpci   txpci
# Idx    MB/s    MB/s
    0   41180   38122
    0   40977   38455
```

That is a useful cross-check: a GPU whose `rxpci` plateaus around 15 000 MB/s when its link claims Gen5 x16 is telling you something the link registers are not.

### 6. Bifurcation

A physical x16 slot is 16 lanes wired to a root port (or to a switch's downstream port). **Bifurcation** is firmware splitting those lanes into independent narrower links — `x8x8`, `x4x4x4x4`, `x8x4x4` — each of which trains as its own PCIe link to a separate device, with its own LTSSM, its own `LnkCap`/`LnkSta`, and its own device on its own bus.

It exists because lanes are a scarce, physically-fixed resource. A carrier or riser card can host two NVMe drives, or a GPU plus a NIC, behind one mechanical slot — but only if the root port is configured to split.

Why it matters for placement:

- **A GPU in a slot bifurcated to `x8x8` trains at x8** — an instant 2× bandwidth loss that looks like nothing is wrong. This is a leading cause of "why is my GPU at x8?"
- **Deliberate bifurcation is how you co-locate a GPU and its rail-aligned NIC** under one root port so GPUDirect RDMA stays local (lesson 04). The skill is telling an intended split from an accident.
- **Bifurcation is link *partitioning*, not multiplexing.** Unlike a switch, it adds no shared-bandwidth penalty: two x8 links from one bifurcated x16 root port each get their own 8 lanes' worth of bandwidth, always. A switch, by contrast, gives each downstream device a full-width link but shares one upstream link between all of them. Two devices behind an x16-upstream switch can each burst to x16 but cannot both do so at once; two devices behind a bifurcated x16 root port each get x8 permanently.
- **`lspci` cannot tell you intent.** You can *infer* bifurcation from `LnkCap: Width x8` on a root port whose slot is physically x16, and you can see it in the topology (two independent root-port branches where you expected one). Whether it was supposed to happen requires the BIOS setup menu or the board's slot documentation. **A width reading is a fact; "was this intended" is a separate question answered out-of-band.**

The diagnostic distinction, stated once:

```
  LnkCap x8 on the root port  →  bifurcation (or a physically x8 slot).
                                 A CONFIGURATION fact. LnkSta x8 matches; no alarm.

  LnkCap x16, LnkSta x8       →  lanes were lost at Detect.
                                 A PHYSICAL fault: seating, contact, bent pin,
                                 riser, or a lane that failed receiver detection.
```

### 7. Retimers, redrivers, and why Gen5 needs them

At 32 GT/s the **unit interval** — the time budget for one bit — is:

```
  UI = 1 ÷ transfer rate
  Gen3  8 GT/s → 125.0 ps
  Gen4 16 GT/s →  62.5 ps
  Gen5 32 GT/s →  31.25 ps
  Gen6 64 GT/s →  31.25 ps  (same symbol rate: 32 GBd PAM4, 2 bits/symbol,
                             but the vertical eye is split into THREE eyes,
                             so each is ~1/3 the height — which is exactly why
                             Gen6 made FEC mandatory)
```

A 31.25 ps budget means everything in the channel matters: dielectric loss in the PCB, via stubs, connector discontinuities, crosstalk from adjacent lanes, and clock jitter. Insertion loss rises roughly with the square root of frequency and linearly with trace length, so past roughly **8–12 inches** of typical server PCB (material and stackup dependent), the eye closes and the link either falls back to a lower speed or fails equalisation intermittently.

Two classes of component fix this, and the distinction is diagnostically relevant:

| | **Redriver** | **Retimer** |
|---|---|---|
| What it does | analogue: equalises and re-amplifies the signal | digital: recovers clock **and** data, then re-transmits a clean, re-timed signal |
| Jitter | passes accumulated jitter through (and adds some) | **resets the jitter budget** — the downstream segment starts clean |
| Protocol awareness | none; it is a wire with gain | a PCIe-aware device: it participates in LTSSM, has its own equalisation phase, and is *visible* in `LnkCap2` as `Retimer+` |
| Enumeration | invisible to `lspci` | not a bus device (it is not a switch, adds no bus number) but reports presence via the link partner's `LnkCap2` |
| Latency added | ~ns | ~10s of ns |

**HGX and DGX baseboards use PCIe Gen5 retimers** to carry each GPU's x16 stub from the GPU baseboard across the connector to the CPU tray, because that path exceeds what a passive channel can do at 32 GT/s. Astera Labs and Broadcom parts are the common choices.

Operationally: a **failing or marginal retimer** is a classic cause of a link training one generation low, or of a link that flaps. When you see a Gen5-capable card stuck at Gen4 x16 with no bifurcation involved and `LnkCtl2: Target Link Speed: 32GT/s`, the physical channel — retimer, connector, or PCB — is the prime suspect, because you have eliminated the configuration explanations.

### 8. AER: the early-warning system

`LnkSta` is a snapshot. **Advanced Error Reporting** is the counter that tells you whether that snapshot is stable, and it will usually move *before* a link visibly downtrains.

PCIe errors come in three severities:

- **Correctable** — the hardware fixed it. A TLP failed LCRC, the receiver NAKed, the sender replayed from its retry buffer, and the transaction completed correctly. **No software ever knows.** This is the class that matters for early warning, because a link can be replaying 1% of its traffic and still report full speed and width.
- **Uncorrectable, non-fatal** — the transaction failed but the link is still usable (e.g. an unsupported request, a completion timeout).
- **Uncorrectable, fatal** — the link itself is unreliable; the platform typically resets it.

Linux exposes per-device counters in sysfs (since kernel 4.19):

```
$ cat /sys/bus/pci/devices/0000:e3:00.0/aer_dev_correctable
RxErr 1284
BadTLP 96
BadDLLP 12
Rollover 0
Timeout 3
NonFatalErr 0
CorrIntErr 0
HeaderOF 0
TOTAL_ERR_COR 1395

$ cat /sys/bus/pci/devices/0000:e3:00.0/aer_dev_nonfatal
Undefined 0
DLP 0
SDES 0
TLP 0
FCP 0
CmpltTO 0
CmpltAbrt 0
UnxCmplt 0
RxOF 0
MalfTLP 0
ECRC 0
UnsupReq 0
ACSViol 0
...
TOTAL_ERR_NONFATAL 0
```

*(Kernel ≥4.19 provides `aer_dev_correctable` / `aer_dev_nonfatal` / `aer_dev_fatal`; newer kernels additionally split these into per-bit files under an `aer_stats/` directory, e.g. `correctable_bit0_RxErr`, `correctable_bit6_BadTLP`. Note the kernel's own caveat: because multiple errors can be reported in one ERR_COR message, `TOTAL_ERR_COR` may not equal the sum of the individual counters. Root ports additionally expose `aer_rootport_total_err_cor` and friends.)*

**What each correctable counter means mechanically** — this is what makes the numbers actionable rather than decorative:

| Counter | Physical meaning | What it points at |
|---|---|---|
| `RxErr` (Receiver Error) | the physical layer detected a framing/encoding violation — a bad symbol on the wire | signal integrity: connector, retimer, crosstalk, jitter. **The most direct evidence of a marginal channel.** |
| `BadTLP` | a TLP arrived with a failing LCRC or a bad sequence number → NAK → replay | the same, one layer up. Every `BadTLP` is a retransmitted packet, i.e. wasted bandwidth |
| `BadDLLP` | a DLLP (ACK/NAK/credit) arrived corrupted | same causes; DLLPs are short so this is rarer than `BadTLP` |
| `Rollover` (REPLAY_NUM rollover) | the sender replayed the same TLP four times | a genuinely bad link, not a stray bit |
| `Timeout` (Replay Timer Timeout) | the sender never got an ACK before the replay timer expired | lost ACKs — signal integrity, or a receiver that is not keeping up |
| `CorrIntErr` | a correctable internal error inside the device | device-side, not channel |
| `HeaderOF` | the AER header log overflowed | so many errors that logging could not keep up — always significant |

And in `dmesg`, the same events as they arrive:

```
$ dmesg -T | grep -i aer | tail -5
[Mon Aug 17 04:12:33 2026] pcieport 0000:e0:01.1: AER: Corrected error received: 0000:e3:00.0
[Mon Aug 17 04:12:33 2026] nvidia 0000:e3:00.0: PCIe Bus Error: severity=Corrected, type=Physical Layer, (Receiver ID)
[Mon Aug 17 04:12:33 2026] nvidia 0000:e3:00.0:   device [10de:2330] error status/mask=00000001/0000e000
[Mon Aug 17 04:12:33 2026] nvidia 0000:e3:00.0:    [ 0] RxErr                  (First)
[Mon Aug 17 04:12:34 2026] pcieport 0000:e0:01.1: AER: Multiple Corrected error received: 0000:e3:00.0
```

**The reading discipline: rate, not total.** A handful of correctable errors accumulated since boot is normal on many platforms. What matters is `d(TOTAL_ERR_COR)/dt`. Sample twice, minutes apart, under load:

```
$ B=/sys/bus/pci/devices/0000:e3:00.0
$ a=$(awk '/TOTAL_ERR_COR/{print $2}' $B/aer_dev_correctable); sleep 300
$ b=$(awk '/TOTAL_ERR_COR/{print $2}' $B/aer_dev_correctable)
$ echo "$(( (b-a) )) corrected errors in 300 s = $(( (b-a)/300 )) /s"
1395 corrected errors in 300 s = 4 /s
```

Four corrected errors per second on a link that still reads `32GT/s x16` is a link that will downtrain, and it is exactly the signal Meta's fleet tooling watches — their published checks are "PCIe link speed, PCIe link width, as well as PCIe-uncorrected and PCIe-corrected error rate checks," in that combination, precisely because the error rate leads the speed/width symptom.

**Clearing counters for a clean baseline:** the sysfs counters are cumulative since boot and are not writable. To get a clean delta after a repair, either note the pre-repair values and subtract, or clear the underlying AER status registers with `setpci` on the AER capability — which is fiddly enough that noting the baseline is usually the right call.

### 9. The diagnostic decision tree

Put §§4–8 together into something you can execute in under a minute per device.

```
                    START: a GPU or NIC is slower than expected
                                      │
                    ┌─────────────────▼─────────────────┐
                    │ sudo lspci -vvv -s <BDF> |        │
                    │   grep -E 'LnkCap:|LnkSta:|       │
                    │           LnkCap2:|LnkCtl2:'      │
                    └─────────────────┬─────────────────┘
                                      │
              ┌───────────────────────┴────────────────────────┐
              │                                                │
      LnkSta == LnkCap                              LnkSta  <  LnkCap
              │                                                │
              ▼                                     ┌──────────┴──────────┐
   Is LnkCap itself lower than                      │                     │
   the part's datasheet?                       WIDTH low            SPEED low
              │                                     │                     │
      ┌───────┴────────┐                            ▼                     ▼
      │                │                  Lanes lost at DETECT.   Equalisation or
     yes              no                  → reseat the card       negotiation issue.
      │                │                  → inspect contacts      │
      ▼                ▼                  → check the riser       ├─ LnkCtl2 target
 A HOP CAPS IT.   Link is at spec.        → confirm the slot      │  speed < LnkCap2?
 Walk the tree     Look elsewhere:          is not bifurcated     │  → SOFTWARE cap.
 with lspci -tv    NUMA (L02), MPS,         (root port LnkCap)    │    firmware/driver
 and read LnkCap   ACS (L01), thermals,                          │
 at EVERY hop.     the workload itself.                          ├─ ASPM enabled and
 A Gen4 switch or                                                │  card idle?
 a bifurcated root                                               │  → re-check UNDER LOAD
 port will show up                                               │
 here.                                                           └─ else → PHYSICAL.
                                                                    retimer / channel /
                                                                    thermal on the PHY

                                      │
                    ┌─────────────────▼─────────────────┐
                    │ IN EVERY BRANCH, ALSO CHECK AER:  │
                    │   sample TOTAL_ERR_COR twice,     │
                    │   300 s apart, under load         │
                    └─────────────────┬─────────────────┘
                                      │
              ┌───────────────────────┴────────────────────────┐
              │                                                │
        rate ≈ 0                                        rate > 0 and rising
              │                                                │
              ▼                                                ▼
   The state is stable. A downgrade here          A DEGRADING physical link,
   is a fixed cause (config, capped hop),         regardless of what LnkSta
   not a decaying one.                            currently says. Escalate to
                                                  hardware even if speed/width
                                                  read full spec.
```

**And the rule that saves you from the most common mistake:** before opening a hardware ticket, check the driver. A regression in a GPU kernel module can produce a link-state symptom indistinguishable from a bad retimer — NVIDIA's own `open-gpu-kernel-modules` issue tracker contains reports of Blackwell-era cards falling back to Gen1 (2.5 GT/s) on Linux due to a kernel-module bug, with `LnkSta` reading exactly what a physical fault would produce. Note the driver version, check whether the same card behaves differently under a different driver, and confirm the fault survives a reboot before sending anyone to the datacentre floor.

## Perspectives

**Developer.** PCIe link state is invisible from CUDA or framework code. There is no `torch.cuda.link_status()`. A developer experiences this fault as "my job is slow" with no API to interrogate. That asymmetry is exactly why it is an interview differentiator: the failure is real, common, and lives entirely outside the tool most engineers reach for first.

**Operator.** Link-state sweeps belong in node acceptance and scheduled fleet health checks, not in ad-hoc debugging triggered by a slow job. The maturity curve runs from "I know the `lspci` incantation" through "we sweep every node at bring-up" to "every node is swept on a schedule, AER rates are a time series, and a regression pages someone." Meta's PCIcrawler and its surrounding pipeline are what the top of that curve looks like in production.

**Hardware / signal integrity.** A degraded link is a physical-layer *symptom*: seating, oxidised contact, a marginal retimer, trace length, crosstalk, or a thermal effect on the PHY. The CLI shows you the symptom, never the root cause. Once `lspci` confirms `LnkSta < LnkCap` and AER confirms a rising error rate, the correct next step is a hardware ticket — reseat, swap, replace the retimer — not more software debugging. Knowing when to stop debugging is part of the skill.

**Economics / reliability.** Meta's published numbers — several thousand servers and components fixed, hundreds caught weekly — are the evidence that PCIe faults are frequent enough at fleet scale to justify a dedicated automated pipeline rather than reactive debugging. At neocloud scale, the cost of not sweeping is paying full price for GPUs delivering a fraction of their host bandwidth, for as long as nobody looks. And because the fault produces no alert, "as long as nobody looks" can be the entire life of the lease.

## Real-world use cases

- **Meta Engineering — "How Facebook deals with PCIe faults to keep our data centers running reliably" (2021)** ([engineering.fb.com](https://engineering.fb.com/2021/06/02/data-center-engineering/how-facebook-deals-with-pcie-faults-to-keep-our-data-centers-running-reliably/)). The substance: Meta describes an automated detect → diagnose → remediate → repair pipeline for PCIe faults. Their checks are PCIe link speed, PCIe link width, and PCIe corrected *and* uncorrected error rates — the exact combination §8 argues for, because the error rate leads the speed/width symptom. They name "bad (down-negotiated) link speed (usually running at 1/2 or 1/4 of the expected speed)" and "bad link width (running at 1/2, 1/4, or even 1/8 of the expected link width)" as the headline fault classes. The program added hardware health coverage that fixed **several thousand servers and server components**, and continues to detect, diagnose, remediate and repair PCIe faults on **hundreds of servers every week**. Their topology tool, **PCIcrawler**, is open source and walks the PCIe tree to find each endpoint's upstream parent — which is the programmatic version of the `lspci -tv` reading in §5. What it shows: this is a standing production function, not a debugging anecdote.

- **NVIDIA/nccl issue #246 — two official tools disagreeing about one box** ([github.com/NVIDIA/nccl/issues/246](https://github.com/NVIDIA/nccl/issues/246)). On an 8× V100 node with Mellanox RDMA NICs, NCCL's own topology detection logged `PIX`/`PXB` for GPU–NIC pairs that `nvidia-smi topo -m` classified as `PHB`. The substance: the two tools build their graphs through different code paths and round the same physical hierarchy to different labels. What it shows: a topology *label* is a derived opinion. When labels matter — and for GPUDirect they decide whether a fast path exists at all — walk the hierarchy yourself with `lspci -tv` or the sysfs device path. This is the reconciliation skill lesson 08 grades.

- **NVIDIA/open-gpu-kernel-modules — Gen1 fallback from a driver regression** ([github.com/NVIDIA/open-gpu-kernel-modules/issues/1010](https://github.com/NVIDIA/open-gpu-kernel-modules/issues/1010)). A Blackwell-era card training at 2.5 GT/s on Linux because of a kernel-module bug, producing a `LnkSta` reading identical to what a failed retimer would produce. The substance: the reporter's `lspci` output shows a fully Gen5-capable `LnkCap` with the link stuck at Gen1 — the §4 signature of "the speed change never happened" — and the cause turned out to be entirely in software. What it shows: check the driver before you blame the hardware. The physical-layer explanation is not the only one that fits the evidence.

## Worked example

A rented "8× H100 SXM" node. `nvidia-smi` shows all eight GPUs, 100% utilisation on a running job, and throughput is roughly half a known-good node in the same cluster. Trace it.

**Step 1 — sweep all eight links in one command.**

```
$ nvidia-smi --query-gpu=index,pci.bus_id,pcie.link.gen.max,pcie.link.gen.current,pcie.link.width.max,pcie.link.width.current --format=csv,noheader
0, 00000000:1B:00.0, 5, 5, 16, 16
1, 00000000:43:00.0, 5, 5, 16, 16
2, 00000000:52:00.0, 5, 5, 16, 16
3, 00000000:61:00.0, 5, 5, 16, 16
4, 00000000:9B:00.0, 5, 5, 16, 16
5, 00000000:C3:00.0, 5, 5, 16, 16
6, 00000000:D1:00.0, 5, 5, 16, 16
7, 00000000:E3:00.0, 5, 4, 16, 16          ← ONE outlier
```

GPU7 at `00000000:E3:00.0` is at Gen4 where the other seven are at Gen5. Width is fine at x16 on all eight. **Width fine, speed low → not bifurcation** (§6), so this is a speed fault or a capped hop.

**Step 2 — confirm from the PCI registers and check whether something *set* the cap.**

```
$ sudo lspci -vvv -s 0000:e3:00.0 | grep -E 'LnkCap:|LnkSta:|LnkCap2:|LnkCtl2:'
        LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM not supported
        LnkSta: Speed 16GT/s (downgraded), Width x16
        LnkCap2: Supported Link Speeds: 2.5-32GT/s, Crosslink- Retimer+ 2Retimers+ DRS-
        LnkCtl2: Target Link Speed: 32GT/s, EnterCompliance- SpeedDis-
```

Read all four:

- `LnkCap` — the card advertises Gen5 x16. Not device-limited.
- `LnkCap2` — supports the full 2.5–32 GT/s vector, with **two retimers reported in the channel**.
- `LnkCtl2: Target Link Speed: 32GT/s` — **software is not capping it.** No firmware or driver told it to stay at Gen4.
- `LnkSta: Speed 16GT/s (downgraded), Width x16` — it trained to Gen5's predecessor anyway.

Every configuration explanation is now eliminated. Per §4, a link that reaches Gen4 but not Gen5 failed the Gen5 **equalisation** phase in Recovery — a physical-channel problem, and `2Retimers+` tells you where to look.

**Step 3 — quantify the loss before saying anything.**

```
  Expected: Gen5 x16 = 32 GT/s × 16 × 128/130 ÷ 8 = 63.02 GB/s per direction
  Actual:   Gen4 x16 = 16 GT/s × 16 × 128/130 ÷ 8 = 31.51 GB/s per direction

  Ratio: exactly 2×.  Usable (§3, × 0.908 × 0.97 × 0.98):
    expected ≈ 54.4 GB/s   actual ≈ 27.2 GB/s

  Matches the observed "roughly half" throughput on a copy-bound workload.
```

**Step 4 — check the upstream hops, to be sure the cap is not somewhere else.**

```
$ lspci -PP -s 0000:e3:00.0
e0:01.1/e1:00.0/e2:08.0/e3:00.0 3D controller: NVIDIA GH100 [H100 SXM5 80GB]

$ for b in e0:01.1 e1:00.0 e2:08.0; do
>   printf '%s: ' "$b"
>   sudo lspci -vvv -s "$b" | awk '/LnkCap:/{c=$0} /LnkSta:/{print c" | "$0}'
> done
e0:01.1: LnkCap: Port #1, Speed 32GT/s, Width x16 | LnkSta: Speed 32GT/s, Width x16
e1:00.0: LnkCap: Port #0, Speed 32GT/s, Width x16 | LnkSta: Speed 32GT/s, Width x16
e2:08.0: LnkCap: Port #8, Speed 32GT/s, Width x16 | LnkSta: Speed 16GT/s (downgraded), Width x16
```

The root port (`e0:01.1`) to switch-upstream (`e1:00.0`) segment is at full Gen5. The **switch downstream port `e2:08.0` to the GPU** is the segment that fell back. That isolates the fault to one physical channel — the retimed path between that switch port and GPU7 — rather than to the card, the switch as a whole, or the platform.

**Step 5 — is it stable or degrading? Sample the AER rate under load.**

```
$ B=/sys/bus/pci/devices/0000:e3:00.0
$ cat $B/aer_dev_correctable
RxErr 8842
BadTLP 341
BadDLLP 27
Rollover 0
Timeout 11
NonFatalErr 0
CorrIntErr 0
HeaderOF 0
TOTAL_ERR_COR 9221

$ a=$(awk '/TOTAL_ERR_COR/{print $2}' $B/aer_dev_correctable); sleep 300
$ b=$(awk '/TOTAL_ERR_COR/{print $2}' $B/aer_dev_correctable)
$ echo "delta=$((b-a)) over 300s"
delta=1173 over 300s
```

**~3.9 corrected errors per second, dominated by `RxErr`.** Per §8's table, `RxErr` is a physical-layer framing/encoding violation — the most direct evidence of a marginal channel available from software. This is not a boot-time fluke; it is an actively degrading link. And it is degrading *while already running one generation low*, which means the channel is marginal even at Gen4.

**Step 6 — rule out a driver explanation, then confirm the fault is isolated.**

```
$ nvidia-smi --query-gpu=driver_version --format=csv,noheader | head -1
570.86.15
$ for g in 0 1 2 3 4 5 6; do
>   printf 'GPU%s ' $g
>   awk '/TOTAL_ERR_COR/{print $2}' /sys/bus/pci/devices/$(nvidia-smi --query-gpu=pci.bus_id --format=csv,noheader -i $g | tr 'A-Z' 'a-z' | sed 's/^0000//;s/^/0000/')/aer_dev_correctable
> done
GPU0 0
GPU1 0
GPU2 0
GPU3 0
GPU4 0
GPU5 0
GPU6 0
```

Seven GPUs at Gen5 x16 with zero corrected errors, on the same driver, on the same board, on the same platform. A driver regression would affect all eight. **This is one channel, and it is physical.**

**Outcome.** The dashboards showed 100% utilisation the whole time. A 30-second link sweep found the 2× loss they could not, the four-register read eliminated every configuration explanation, walking the path isolated the fault to one switch-port-to-GPU segment through two retimers, and the AER rate turned "there is a downgrade" into "this link is actively degrading." The deliverable is a hardware ticket naming the specific segment, with the error rate as evidence — not "GPU7 seems slow."

## Practice

Feeds the **Topology Teardown** deliverable. Work on a real GPU box; a rented instance is fine. A node where everything is healthy is a valid and expected result — the note must *prove* that with numbers, not assert it.

1. **Capture the tree and annotate it.** `lspci -tv > topology-tree.txt`. Mark each host bridge as a root complex, each first-hop `PCI bridge` as a **root port**, each subsequent bridge as a **switch upstream** or **switch downstream** port, and each NVIDIA / Mellanox / NVMe line as an **endpoint**. Note which socket each GPU descends from (bus-number range plus `numa_node`, lesson 01 §8).

2. **Sweep every GPU's link, four registers each.** For each GPU BDF:
   ```
   sudo lspci -vvv -s <BDF> | grep -E 'LnkCap:|LnkSta:|LnkCap2:|LnkCtl2:|DevCtl:'
   ```
   Build a table: `GPU | BDF | LnkCap gen×width | LnkSta gen×width | LnkCtl2 target | MPS | MRRS | GB/s each way | OK / DEGRADED`.

3. **Compute the usable bandwidth for each link**, not just the raw figure. Use §3's chain: `rate × lanes × code_efficiency ÷ 8`, then `× MPS/(MPS+26)`, then `× 0.97`. Compare against a `bandwidthTest` measurement for at least one GPU and state the residual.

4. **Classify every degradation.** For any device with `LnkSta < LnkCap`, state whether it is a **speed** or **width** downgrade, and give the most likely cause using §9's tree: width → lanes lost at Detect (seating, riser, bifurcation); speed → `LnkCtl2` cap, ASPM, or equalisation/channel. Re-check any suspicious link **under GPU load** to rule out ASPM idle downtraining.

5. **Sample AER rates on every device, including the healthy ones.**
   ```
   for d in /sys/bus/pci/devices/*/aer_dev_correctable; do
     printf '%s %s\n' "$(basename $(dirname $d))" "$(awk '/TOTAL_ERR_COR/{print $2}' $d)"
   done
   ```
   Take two samples 300 s apart under load and record the rate. A device reading full spec with a non-zero, rising rate is a finding worth writing down.

6. **Extend the sweep to the NICs and NVMe.** A ConnectX-7 wants Gen5 x16; at Gen4 x8 it quietly halves your RDMA ceiling and you will misattribute the loss to the fabric. NVMe drives are usually Gen4/Gen5 x4 — check them against the drive's datasheet, not against x16.

7. **Record the MPS story for the whole hierarchy.** Note every device's `DevCap: MaxPayload` and `DevCtl: MaxPayload`. If any device under a root is limited to 128 bytes, name it — it is costing every device under that root about 8 points of efficiency.

**Acceptance:** a committed note in the deliverable listing, for every GPU, NIC and NVMe device: its **trained link vs its maximum** (all four registers), its **computed usable bandwidth** with the arithmetic shown, its **MPS/MRRS**, and its **AER correctable-error rate** (not just the total). Every device marked OK or DEGRADED, and every DEGRADED entry annotated with speed-vs-width, the eliminated explanations, and a probable cause.

## Common pitfalls

1. **Treating `LnkSta` alone as the whole story.** *Symptom:* a device reads full spec, so it is declared healthy. *Mechanism:* correctable errors are *corrected* — a link replaying 1% of its TLPs still reports `32GT/s x16` while burning bandwidth on retransmission. AER's correctable rate moves before speed or width does. Check both, and check the *rate*, not the cumulative total.

2. **Blaming hardware by default.** *Symptom:* a hardware ticket for a link that a driver update fixes. *Mechanism:* the link registers reflect whatever the driver and firmware negotiated. A kernel-module regression can leave a card at Gen1 with a fully Gen5-capable `LnkCap` — indistinguishable from a physical fault by `lspci` alone. Check `LnkCtl2`'s target speed (software cap?), check the driver version, and check whether *other identical devices on the same node* show the same symptom before escalating.

3. **Not re-checking under load.** *Symptom:* a `2.5GT/s` reading on an idle card reported as a critical fault. *Mechanism:* ASPM can legitimately drop an idle link to a low-power state; on exit it re-enters Recovery and retrains. Read `LnkCtl: ASPM ...` to see whether ASPM is even enabled, and always confirm the reading under a real workload.

4. **Confusing a reduced `LnkCap` with a fault.** *Symptom:* "the GPU is at x8, it must be broken" when the root port is bifurcated. *Mechanism:* a bifurcated root port *advertises* `LnkCap: Width x8`, so `LnkSta: Width x8` matches and there is no `(downgraded)` marker. The fault signature is specifically `LnkSta < LnkCap` **on the same device**. A low `LnkCap` is a configuration or topology fact that you resolve by walking the tree and reading `LnkCap` at every hop.

5. **Reading the wrong register with `setpci`.** *Symptom:* concluding a link is software-capped when it is not, or vice versa. *Mechanism:* `CAP_EXP+12.w` is **Link Status**; **Link Control 2** — where the target link speed lives — is at `CAP_EXP+30.w`. They are different registers with different meanings and off-by-a-lot consequences. Prefer `lspci -vvv`'s decoded output and use `setpci` only when you know exactly which offset you want.

6. **Trusting one tool's topology label over another without reconciling.** *Symptom:* GPUDirect silently not used despite a `PIX` label. *Mechanism:* NCCL and `nvidia-smi` build their topology graphs independently and can disagree (issue #246); and ACS can make a `PIX` pair route through the root complex anyway (lesson 01 §11). Walk the physical tree with `lspci -tv` as the tiebreaker, and confirm the functional consequence with `nvidia-smi topo -p2p rw`.

7. **Ignoring MPS as a source of "missing" bandwidth.** *Symptom:* a link at full Gen5 x16 that measures 47 GB/s instead of 54. *Mechanism:* every device under a common root must share an MPS, so one legacy 128-byte device drags the whole subtree down; `256 ÷ 282 = 90.8%` becomes `128 ÷ 154 = 83.1%`, a further 8 points. Check `DevCtl: MaxPayload` on every device under that root before assuming a physical problem.

## Self-check

- **A GPU trained at Gen3 x8 versus a spec of Gen5 x16 — what is the bandwidth ratio, and how do you detect it?**
  **Answer:** Gen5 x16 = `32 GT/s × 16 × 128/130 ÷ 8 = 63.0 GB/s` per direction; Gen3 x8 = `8 GT/s × 8 × 128/130 ÷ 8 = 7.9 GB/s` — a factor of **8×** (4× from two generation steps, 2× from x16→x8). Detect it with `sudo lspci -vvv -s <BDF> | grep -E 'LnkCap:|LnkSta:'`: `LnkCap` reads `32GT/s, Width x16` while `LnkSta` reads `8GT/s (downgraded), Width x8 (downgraded)`. Faster for a whole node: `nvidia-smi --query-gpu=pcie.link.gen.max,pcie.link.gen.current,pcie.link.width.max,pcie.link.width.current --format=csv`. `nvidia-smi` will still report high utilisation throughout, which is exactly why you must read the link state directly.

- **Derive the usable bandwidth of a Gen5 x16 link at MPS = 256 bytes, showing every loss.**
  **Answer:** Start with symbols: `32 GT/s × 16 lanes ÷ 8 = 64.00 GB/s`. Apply 128b/130b line coding: `× 128/130 = 63.02 GB/s` — this is the figure everyone quotes. Apply TLP overhead: a 64-bit-addressing memory-write TLP carries 4 B STP framing + 2 B sequence number + 16 B header + 4 B LCRC = 26 B of overhead, so `256 ÷ 282 = 90.8%` → `57.2 GB/s`. Apply DLLP traffic (ACK/NAK and flow-control credits share the wires, ~3% at this MPS) → `55.5 GB/s`. Apply device-side losses (DMA outstanding-request limits, completion latency, IOMMU) ~2% → **≈54.4 GB/s**, which is what a pinned `bandwidthTest` actually measures. At MPS 128 the TLP term becomes `128 ÷ 154 = 83.1%` and the result drops to about 49 GB/s.

- **How do you tell a root port, a switch and an endpoint apart in `lspci -tv`?**
  **Answer:** The **root port** is the first hop under a host-bridge line (`-+-[0000:00]-`), where the PCIe hierarchy originates inside the CPU; its class is `PCI bridge`. A **switch** appears as a `PCI bridge` (the upstream port) with one or more further `PCI bridge` entries indented beneath it (the downstream ports), each with its own bus range in brackets — and everything below shares the switch's single upstream link. An **endpoint** is the deepest leaf with a real device class (`3D controller`, `Infiniband controller`, `Non-Volatile memory controller`) and nothing indented under it. Depth is indentation; the bracketed `[nn-mm]` ranges are each bridge's secondary-through-subordinate bus numbers, assigned by firmware during the depth-first enumeration walk. `lspci -PP -s <BDF>` prints the whole path on one line.

- **What is bifurcation, how does it differ from a switch, and when does it matter?**
  **Answer:** Bifurcation is firmware splitting one root port's 16 lanes into independent narrower links (`x8x8`, `x4x4x4x4`, `x8x4x4`), each training as its own PCIe link with its own LTSSM and its own `LnkCap`/`LnkSta`. It differs from a switch in the bandwidth model: bifurcation **partitions** lanes, so two x8 links each get their own 8 lanes' worth permanently and never contend; a switch **multiplexes**, so each downstream device gets a full-width link but they all share one upstream link. It matters two ways: (1) a GPU in a slot accidentally bifurcated to `x8x8` trains at x8 — a silent 2× loss; (2) deliberate bifurcation is how you place a GPU and its rail-aligned NIC under one root port so GPUDirect RDMA stays local. `lspci` shows you the *width* but not the *intent* — a bifurcated root port advertises `LnkCap: Width x8`, so there is no `(downgraded)` marker; confirming whether it was deliberate needs the BIOS menu or the board documentation.

- **What is the difference between a link that is always trained low and one that is flapping, and what does each suggest?**
  **Answer:** The LTSSM re-enters Recovery — and may renegotiate speed and width — after an uncorrectable receiver error, a replay-timer timeout, an ASPM exit, or a software retrain request. A link **stuck** permanently low (same reading at boot, idle and under load) points at a fixed cause: a Gen4-capped hop upstream, a `LnkCtl2` target speed set below `LnkCap2`, a bifurcated root port, or a driver that never attempted the speed change. A link **flapping** between speeds points at a marginal physical connection — partial seating, oxidised contact, a failing retimer, or a thermal effect on the PHY — that intermittently fails equalisation. Distinguish them by sampling repeatedly under load, watching `dmesg` for AER and link-training messages, and checking whether the AER correctable rate is non-zero (flapping links almost always have one; capped links usually do not).

- **Besides `LnkSta`, what signal makes you confident a link is genuinely degrading, and what do the individual counters mean?**
  **Answer:** The **AER correctable error rate**, sampled twice under load rather than read once. Read `/sys/bus/pci/devices/<bdf>/aer_dev_correctable` (kernel ≥4.19; newer kernels also split it into per-bit files under `aer_stats/`). `RxErr` is a physical-layer framing or encoding violation — the most direct software-visible evidence of a marginal channel. `BadTLP` means a TLP failed its LCRC or sequence check and was NAKed and replayed — each one is wasted bandwidth. `BadDLLP` is the same for an ACK/NAK/credit packet. `Rollover` means the same TLP was replayed four times. `Timeout` means an ACK never arrived before the replay timer expired. Note the kernel's caveat that `TOTAL_ERR_COR` may not equal the sum of the individual counters, because one ERR_COR message can report several errors. A rising rate on a link that still reads `LnkSta == LnkCap` is the earliest actionable signal you will get, and it is precisely what Meta's fleet checks watch alongside speed and width.

- **A GPU reads full Gen5 x16 with zero AER errors but its copy benchmark is 15% low. Where do you look next?**
  **Answer:** The link is at spec, so the loss is above or beside it. In order: (1) **`MaxPayloadSize`** — check `DevCtl: MaxPayload` on the GPU *and* on every device under the same root; one 128-byte device drags the subtree from 90.8% to 83.1% TLP efficiency. (2) **`MaxReadReq`** — a small MRRS limits how many bytes the DMA engine can have outstanding, which is a Little's-Law concurrency cap. (3) **NUMA placement of the host buffer** (lesson 02) — `min(PCIe, UPI)` if the buffer is on the far socket. (4) **IOMMU mode and SWIOTLB** (lesson 02 §9) — a bounce path below CUDA. (5) **ACS** (lesson 01 §11) — if this is a P2P rather than host copy. (6) The benchmark itself: transfer size, pinned vs pageable, and whether anything else is sharing the link.

## Connections & what's next

This lesson gives you the vocabulary and the arithmetic that lesson 04 assumes: `LnkCap`/`LnkSta`/`LnkCap2`/`LnkCtl2`, root port vs switch vs endpoint, retimers, bifurcation, and the ability to turn a link's gen×width into a usable GB/s figure. **Lesson 04** takes the single-link skill and applies it at whole-node scale — the canonical 8-GPU HGX/DGX topology, where GPU–GPU traffic leaves PCIe entirely for NVLink and PCIe's job narrows to GPU↔host, GPU↔NIC and GPU↔storage. **Lesson 05** builds on the same NUMA-locality argument from the Kubernetes scheduler's side. **Lesson 06** reuses `PIX`/`PXB`/`SYS` reasoning for NVMe and GPUDirect Storage, where the link widths are x4 rather than x16 and the arithmetic changes accordingly. **Lesson 08**'s capstone is where you reconcile `lspci`, `nvidia-smi topo -m`, `lstopo` and `numactl` into one picture on a real node — and the AER and link-training skills from this lesson are what let you *trust* that picture rather than merely read it.

Next: **lesson 04** — the reference topology every one of these links is supposed to conform to.

## References & further reading

**Primary sources**

- **PCI-SIG specification announcements** — [pcisig.com](https://pcisig.com/) — the authoritative source for per-generation transfer rates, signalling (NRZ vs PAM4), and the FLIT-mode change in 6.0. PCI-SIG's own framing quotes x16 figures as bidirectional totals (e.g. 128 GB/s for Gen5, 256 GB/s for Gen6, 512 GB/s for Gen7); halve them for the per-direction numbers used throughout this lesson.
- **`lspci(8)` and pciutils** — [github.com/pciutils/pciutils](https://github.com/pciutils/pciutils) — `-tv` (tree), `-vvv` (full capability blocks, needs root), `-D` (domains), `-PP` (full path), `-s` (select), `-nn` (numeric + name). `setpci(8)` for direct register access with the `CAP_EXP+<offset>` syntax.
- **Linux kernel — PCI documentation** — [docs.kernel.org/PCI/](https://docs.kernel.org/PCI/index.html) — enumeration, the AER driver, and the `pci=pcie_bus_*` MPS policies in `admin-guide/kernel-parameters`.
- **Linux kernel — AER sysfs ABI** — [kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci-devices-aer_stats](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci-devices-aer_stats) — the exact field names in `aer_dev_correctable` / `aer_dev_nonfatal` / `aer_dev_fatal`, the per-bit `aer_stats/` files in newer kernels, and the caveat that `TOTAL_ERR_COR` may not equal the sum of its parts.
- **NVIDIA `nvidia-smi` manual** — bundled with the driver — `--query-gpu=pcie.link.gen.{max,current},pcie.link.width.{max,current}`, `-q -d PCI` (including `Device Max` vs `Host Max`), and `dmon -s t` for live PCIe throughput.

**Real-world engineering**

- **Meta Engineering — "How Facebook deals with PCIe faults to keep our data centers running reliably"** — [engineering.fb.com](https://engineering.fb.com/2021/06/02/data-center-engineering/how-facebook-deals-with-pcie-faults-to-keep-our-data-centers-running-reliably/) — the detect/diagnose/remediate/repair pipeline, the specific checks (link speed, link width, corrected and uncorrected error rates), and the scale numbers quoted in §"Why this matters".
- **Meta Engineering — "PCIcrawler: A tool to debug PCI issues at scale"** — [engineering.fb.com](https://engineering.fb.com/2020/08/05/open-source/pcicrawler/) — the open-source topology tool behind that pipeline; the programmatic version of `lspci -tv` plus per-device link state.
- **NVIDIA/nccl issue #246** — [github.com/NVIDIA/nccl/issues/246](https://github.com/NVIDIA/nccl/issues/246) — two official NVIDIA tools disagreeing on GPU–NIC topology labels for the same box.
- **NVIDIA/open-gpu-kernel-modules issue #1010** — [github.com/NVIDIA/open-gpu-kernel-modules/issues/1010](https://github.com/NVIDIA/open-gpu-kernel-modules/issues/1010) — a driver regression producing a Gen1 link fallback identical in appearance to a hardware fault.

**Deeper dives**

- **Ravi Budruk, Don Anderson, Tom Shanley — *PCI Express System Architecture*** (MindShare) — the standard reference for the LTSSM, packet formats, and flow control; chapter 14's LTSSM treatment is the source for the eleven-state model in §4.
- **Chris Siebenmann — "PCIe Topology and Lanes"** — [utcc.utoronto.ca](https://utcc.utoronto.ca/~cks/space/blog/linux/PCIeTopologyAndLanes) — a practitioner deep-dive on reading `lspci -tv` and lane topology in more detail than this lesson has room for.
