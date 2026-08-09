---
lesson: "02b.3"
title: "PCIe at operational depth"
module: "02b"
concept: "PCIe at operational depth"
status: not-started
est_time: "4h"
artifacts: []
---
# 02b.3 · PCIe at operational depth
> **Concept.** A PCIe link silently trains to a lower generation or width and the GPU still "works" — at a fraction of spec. Learn to read the trained-vs-max link and catch it.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Why this matters

An H100 wants a PCIe **Gen5 x16** link to the host: ~63 GB/s each direction. That link
carries every byte the GPU reads from host memory, every `cudaMemcpy` staging buffer, and —
on a badly placed node — GPUDirect RDMA and GPUDirect Storage traffic. If that link trains
at **Gen3 x8** instead, you have ~7.9 GB/s each direction: **8x less**, and nothing in
Grafana will tell you. `nvidia-smi` shows 100% "GPU utilization" because SM occupancy is
high; DCGM shows the card is hot and drawing power. The card *looks* busy. It is busy —
stalling on a starved host link, spending cycles waiting for data that arrives eight times
too slowly.

This is the exact failure this module exists to prevent: a GPU that looks fully utilized but
delivers below spec because of host-side placement, invisible to every dashboard that only
watches the accelerator. Link degradation is the single most common cause. It comes from a
reseated card that didn't fully mate, a bad retimer, a dirty slot, a marginal riser, thermal
throttling on the PCIe PHY, or a BIOS that bifurcated a slot wrong. The fix is trivial once
you *see* it. Seeing it is the skill. At a neocloud you will be handed a fleet of rented
boxes you did not build, and "is this link actually trained to spec?" is a question you must
answer from the CLI in under a minute per card.

## What's new for you

You know PCIe exists and that GPUs sit on it. From CKA and on-prem work you've read
`lspci` output for NIC and HBA enumeration. What's new:

- **Trained vs. maximum link state.** Every PCIe link *negotiates* speed and width at boot.
  `lspci` exposes both the capability (`LnkCap`) and what was actually negotiated (`LnkSta`).
  You've probably never compared them line by line. That comparison is the whole lesson.
- **The topology tree as a diagnostic object.** `lspci -tv` is not just a device list — it's
  the actual PCIe hierarchy: root ports, switches, and endpoints, with the CPU at the root.
  Reading it tells you *how* a device reaches the CPU and whether it shares bandwidth.
- **Per-lane arithmetic you can do in your head.** You'll memorize the per-lane rate of each
  generation so that "Gen3 x8" instantly converts to "~7.9 GB/s, 8x down" without a lookup.
- **Bifurcation and retimers** as placement concerns — the physical-layer reasons a slot
  can only give you x8, or why a Gen5 signal needs a retimer to survive the trace length.

## Core notes

### Per-lane bandwidth — the numbers to memorize

PCIe bandwidth is `per-lane rate × lane count`, one figure per direction (links are
full-duplex — double it for aggregate). Gen3/4/5 use 128b/130b encoding, so the usable rate
is very close to the raw GT/s figure:

| Gen | Raw rate | Encoding | **Per lane, per direction** | **x16** | **x8** | **x4** |
|-----|----------|----------|-----------------------------|---------|--------|--------|
| 3   | 8 GT/s   | 128b/130b | ~0.985 GB/s | ~15.8 GB/s | ~7.9 GB/s | ~3.9 GB/s |
| 4   | 16 GT/s  | 128b/130b | ~1.97 GB/s  | ~31.5 GB/s | ~15.8 GB/s | ~7.9 GB/s |
| 5   | 32 GT/s  | 128b/130b | ~3.94 GB/s  | ~63 GB/s   | ~31.5 GB/s | ~15.8 GB/s |
| 6   | 64 GT/s (PAM4) | 242B FLIT | ~7.56 GB/s | ~121 GB/s | ~60 GB/s | ~30 GB/s |

Two facts fall out of this table that you should be able to state instantly:

- **Each generation doubles the per-lane rate**; halving the width halves bandwidth. So
  "Gen5 x16" and "Gen3 x16" differ by 4x; "Gen5 x16" and "Gen3 x8" differ by **8x**.
- The `LnkSta` **speed field is the raw GT/s number**, not the generation: `2.5` = Gen1,
  `5` = Gen2, `8` = Gen3, `16` = Gen4, `32` = Gen5, `64` = Gen6. Train yourself to read the
  GT/s and map it, because that's what the tool prints.

Gen6 (64 GT/s) switches to PAM4 signaling and FLIT-based framing; it appears on
Blackwell-era platforms. You don't need its internals — just know 64 GT/s ≈ Gen6 and that
the per-lane figure roughly doubles again.

### Root ports, switches, endpoints — the hierarchy

PCIe is a tree rooted at the CPU. Three node types matter:

- **Root port** — a port of the **Root Complex**, integrated into the CPU (or its IO die).
  This is where the PCIe hierarchy physically originates. On a dual-socket box each socket
  has its own root complex, so each socket owns a set of root ports. A device hanging off
  socket 0's root port reaches socket 1's memory only by crossing the inter-socket link
  (UPI/xGMI) — this is the NUMA locality that lesson 02b.2 covered, seen from the PCIe side.
- **Switch** — a fan-out device with one **upstream port** (toward the root) and several
  **downstream ports** (toward endpoints). A switch lets multiple endpoints share one link
  up to the root. Bandwidth *upstream of the switch is shared* by everything below it — this
  is why a PCIe switch feeding two GPUs and a NIC can become a bottleneck even if each
  device's own link is full width.
- **Endpoint** — a leaf: the GPU, the NIC, the NVMe drive. It terminates the tree.

You read this hierarchy from `lspci -tv`. Indentation = depth. The root ports sit just under
the host bridge at the top; `+-[0000:00]` style host bridges are the roots. A `PCI bridge`
line with things indented under it is a switch (or a root port, if it's the first hop from
the host bridge). Endpoints are the deepest lines with a real device class (VGA/3D
controller, Ethernet, Non-Volatile memory).

**Why the distinction is operational:** the path a GPU takes to reach a NIC depends entirely
on whether they share a switch (fast, stays local, good for GPUDirect RDMA), sit under
sibling root ports on the same socket (crosses the root complex), or sit on different sockets
(crosses UPI — the worst case). `nvidia-smi topo -m` names exactly these cases: `PIX` = one
bridge/switch hop, `PXB` = multiple bridges, `PHB` = traverses a PCIe host bridge (root
complex), `NODE` = within a NUMA node across host bridges, `SYS` = crosses the socket
interconnect. `lspci -tv` is the ground truth those labels summarize.

### Bifurcation

A physical x16 slot is 16 lanes wired to one or more root ports. **Bifurcation** is the BIOS
splitting those 16 lanes into independent narrower links — `x8x8`, `x4x4x4x4`, `x8x4x4` —
each of which trains as its own PCIe link to a separate device. It exists because a single
x16 slot's worth of lanes is a scarce resource: a carrier/riser card can put two NVMe drives
or a GPU + a NIC behind one physical slot, but only if the root port is configured to split.

Why it matters for GPU/NIC placement:

- A GPU dropped into a slot that BIOS has bifurcated to `x8x8` will train at **x8**, not x16
  — instant 2x bandwidth loss that looks like nothing is wrong. Bifurcation misconfiguration
  is a leading cause of "why is my GPU at x8?"
- Conversely, deliberate bifurcation is how you *co-locate* a GPU and its rail-aligned NIC
  under the same root port so GPUDirect RDMA stays local (lesson 02b.4). The skill is knowing
  whether a x8 reading is an intended split or an accident.
- Bifurcation is a **link-partitioning** decision, distinct from a switch. A switch adds
  silicon and shares upstream bandwidth; bifurcation just repartitions existing root-port
  lanes with no shared-bandwidth penalty, but it can't create more lanes than the slot has.

### Retimers

At Gen5's 32 GT/s, signal integrity over PCB traces degrades fast — beyond ~7-9 inches of
trace the eye closes and the link won't train at full speed (it falls back to a lower gen, or
flaps). A **retimer** is an active repeater that recovers the clock and data and re-drives a
clean signal, extending reach. HGX/DGX baseboards use PCIe Gen5 retimers to carry each GPU's
x16 stub from the GPU baseboard across to the CPU tray. Operationally: a **failing or
marginal retimer** is a classic cause of a link training one generation low (Gen4 instead of
Gen5) or intermittently. When you see a Gen5-capable card stuck at Gen4 x16 with no
bifurcation involved, a retimer or the physical channel is the prime suspect.

### Reading LnkCap vs LnkSta — the core skill

`lspci -vvv` prints, in the PCI Express capability block, two lines you compare directly:

```
LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM L1, Exit Latency L1 <64us
LnkSta: Speed 32GT/s, Width x16
```

- **`LnkCap`** = the maximum this link *can* do (the lower of the two link partners' caps).
- **`LnkSta`** = what it *actually negotiated* at training time.

A healthy H100 link reads `LnkCap ... Speed 32GT/s, Width x16` **and** `LnkSta: Speed 32GT/s,
Width x16`. A degraded one reads:

```
LnkCap: Port #0, Speed 32GT/s, Width x16
LnkSta: Speed 8GT/s (downgraded), Width x8 (downgraded)
```

The kernel prints **`(downgraded)`** when `LnkSta` is below `LnkCap` — that word is your
alarm. But do not rely on it alone: some tool/kernel versions omit it, and a link can be at
full width but wrong speed or vice versa. **Always compare the four numbers yourself.**

Two important subtleties:

- **Active Power Management flap.** With ASPM enabled, an idle link can drop to Gen1 to save
  power and retrain up under load. So a `2.5GT/s` reading on an idle card may be benign —
  re-check `LnkSta` while the GPU is under load (or after disabling ASPM) before declaring a
  fault. Under a real workload the link must be at full spec.
- **Two ends, two capabilities.** `LnkCap` on the endpoint reflects the negotiated minimum.
  If a Gen5 GPU sits behind a Gen4-only switch or a Gen4 riser, `LnkCap` itself will read
  16GT/s — the ceiling was set by the weakest link in the chain, not the card. Walk the tree
  with `lspci -tv` to find *which* hop caps it.

`lspci -vvv` needs root to show the full capability block (`sudo`); without it the `LnkCap`/
`LnkSta` lines are often blank or `<access denied>`.

## Worked example

A rented "8x H100" node. `nvidia-smi` shows all eight GPUs, 100% util on a running job, but
throughput is roughly half of a known-good node. Trace it.

**1. Find the GPU BDFs (bus:device.function):**

```
$ lspci -D | grep -i nvidia
0000:1b:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
0000:43:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
... (six more) ...
0000:e3:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB] (rev a1)
```

**2. Read the tree to see how each reaches the CPU:**

```
$ lspci -tv
-+-[0000:e0]-+-01.1-[e1-e3]----00.0-[e2-e3]----00.0-[e3]--+-00.0  NVIDIA GH100
 |           ...
 \-[0000:00]-+-...
```

The GPU at `e3:00.0` hangs off root port `e0:01.1`, then through two bridges (`e1-e3`,
`e2-e3`) — those intermediate `PCI bridge` hops are **retimers/switch stages** on the
baseboard. The `[0000:e0]` and `[0000:00]` host bridges are the **two sockets' root
complexes**: BDFs starting `e0`/`c0`/`a0`... belong to socket 1, low buses (`00`/`20`) to
socket 0. So `e3:00.0` is a socket-1 GPU.

**3. Compare LnkCap vs LnkSta on the slow GPU:**

```
$ sudo lspci -vvv -s 0000:e3:00.0 | grep -E 'LnkCap:|LnkSta:'
        LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM not supported
        LnkSta: Speed 16GT/s (downgraded), Width x16
```

**Read:** `LnkCap` says the card and its channel are Gen5-capable (32GT/s) at full x16. But
`LnkSta` trained at **16GT/s = Gen4** — one generation low, width fine. `(downgraded)`
confirms it. That's `31.5 GB/s` instead of `63 GB/s`: **half bandwidth**, matching the
observed throughput. Width is x16, so this is *not* bifurcation — it's a **speed** fault:
marginal retimer, dirty/oxidized contact, or a thermal/signal-integrity issue on that
GPU's Gen5 channel.

**4. Confirm it's channel-specific, not fleet-wide:** the same one-liner across all eight
shows the other seven at `Speed 32GT/s, Width x16`. Only `e3:00.0` is at Gen4. That isolates
the fault to one GPU's physical link — a hardware ticket for that slot/retimer, not a BIOS or
firmware-wide problem. **Outcome:** the dashboards showed 100% util the whole time; the
30-second `LnkCap`-vs-`LnkSta` sweep found the 2x loss they couldn't.

## Practice

Feeds the **Topology Teardown** deliverable. Work on a real GPU box (a rented instance is
fine).

1. **Capture the tree.** Run `lspci -tv > topology-tree.txt`. Annotate it: mark each host
   bridge as a **root complex / root port**, each intermediate `PCI bridge` as a **switch or
   retimer stage**, and each NVIDIA/Mellanox/NVMe line as an **endpoint**. Note which socket
   (root complex) each GPU hangs off.
2. **Sweep every GPU's link.** For each GPU BDF:
   `sudo lspci -vvv -s <BDF> | grep -E 'LnkCap:|LnkSta:'`. Build a table: `GPU | BDF |
   LnkCap (gen×width) | LnkSta (gen×width) | GB/s each way | OK / DEGRADED`.
3. **Flag and diagnose.** For any device whose `LnkSta` is below its `LnkCap`, state whether
   it's a **speed** downgrade, a **width** downgrade, or both, and give the most likely cause
   (bifurcation for a width drop to x8/x4; retimer/channel/ASPM for a speed drop). Re-check any
   suspicious link under GPU load to rule out ASPM idle downclocking.
4. **Extend to the NICs.** Repeat the sweep for the ConnectX NICs — a ConnectX-7 wants Gen5
   x16 too; a NIC at Gen4 x8 quietly halves your RDMA ceiling.

**Acceptance:** a committed note in the deliverable listing, for every GPU (and NIC), its
**trained link vs. its maximum**, with each device marked OK or DEGRADED and every DEGRADED
entry annotated with speed-vs-width and a probable cause. A node with all eight GPUs at
`32GT/s x16` is a valid — and expected — result; the note must *prove* that by showing the
numbers, not assert it.

## Self-check

**(a)** A GPU trained at Gen3 x8 versus a spec of Gen5 x16 — what is the bandwidth ratio, and
how do you detect it from the CLI?

**Answer:** Gen5 x16 ≈ 63 GB/s/direction; Gen3 x8 ≈ 7.9 GB/s/direction — a factor of **8x**
(4x from the two generation steps 5→3, 2x from x16→x8). Detect it with
`sudo lspci -vvv -s <BDF> | grep -E 'LnkCap:|LnkSta:'`: `LnkCap` shows `32GT/s, Width x16`
while `LnkSta` shows `8GT/s (downgraded), Width x8 (downgraded)`. `nvidia-smi` will still
report high utilization, which is exactly why you must read the link state directly.

**(b)** How do you tell a root port, a switch, and an endpoint apart in `lspci -tv`?

**Answer:** The **root port** is the first hop under a host-bridge line (`-+-[0000:00]`) at
the top of the tree — it's a port of the CPU's root complex where the hierarchy originates.
A **switch** is a `PCI bridge` node with *multiple* devices/bridges indented beneath it
(one upstream port, several downstream); everything below it shares its upstream link. An
**endpoint** is the deepest leaf with a real device class (3D controller, Ethernet,
Non-Volatile memory) and nothing indented under it. Depth = indentation; the CPU is the root.

**(c)** What is bifurcation and when does it matter for GPU/NIC placement?

**Answer:** Bifurcation is the BIOS splitting one physical slot's x16 lanes into independent
narrower links (`x8x8`, `x4x4x4x4`), each training as its own PCIe link to a separate device.
It matters two ways: (1) a GPU in a slot accidentally bifurcated to `x8x8` trains at **x8** —
a silent 2x loss you'd catch as a *width* downgrade in `LnkSta`; (2) deliberate bifurcation
is how you place a GPU and its rail-aligned NIC under one root port so GPUDirect RDMA stays
local. Unlike a switch it adds no shared-bandwidth penalty, but it can't create more lanes
than the slot physically has.

## Resources

1. **Linux kernel PCI documentation** — https://docs.kernel.org/PCI/index.html — the
   authoritative reference for how the kernel enumerates and reports the PCI hierarchy,
   link training, and the sysfs/`lspci` fields you're reading.
2. **`lspci(8)` / pciutils** — `man lspci` and https://github.com/pciutils/pciutils — flags
   that matter here: `-tv` (tree), `-vvv` (full capability blocks incl. `LnkCap`/`LnkSta`),
   `-D` (domain BDFs), `-s` (select a device).
3. **PCI-SIG PCIe base spec generation rates** (encoding & GT/s→GB/s) — the per-lane figures
   in the table above; cross-check against your platform's HGX/DGX system guide for the
   expected per-GPU link (Gen5 x16 on H100).
