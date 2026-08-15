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
sources: 8
---
# 02b.3 · PCIe at operational depth

> **Concept.** A PCIe link silently trains to a lower generation or width and the GPU still "works" — at a fraction of spec. Learn to read the trained-vs-max link, the secondary error signals that precede it, and catch it before it costs a training run.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Lesson 02 established that DRAM bandwidth is a hard architectural ceiling — and that the wrong NUMA node for a pinned buffer quietly throws away a large fraction of it. This lesson moves one hop further down the data path: the physical link between the host and the GPU itself. PCIe is the wire every one of those bytes crosses on the way in and out of the GPU, and — just like a NUMA-remote buffer — a degraded PCIe link produces **no error**, only a number that is lower than it should be. Once you can read a trained link and its early-warning signals, lesson 04 hands you the full 8-GPU reference topology so you know *where* every one of these links is supposed to go.

## Why this matters

An H100 wants a PCIe **Gen5 x16** link to the host: ~63 GB/s each direction. That link carries every byte the GPU reads from host memory, every `cudaMemcpy` staging buffer, and — on a badly placed node — GPUDirect RDMA and GPUDirect Storage traffic. If that link trains at **Gen3 x8** instead, you have ~7.9 GB/s each direction: **8x less**, and nothing in Grafana will tell you. `nvidia-smi` shows 100% "GPU utilization" because SM occupancy is high; DCGM shows the card is hot and drawing power. The card *looks* busy. It is busy — stalling on a starved host link, spending cycles waiting for data that arrives eight times too slowly.

This is not a hypothetical. Meta's infrastructure engineering team has published in detail on exactly this failure class: their PCIe fault-management program calls out "**bad link speed and link width**" explicitly as a category that "could be difficult to detect without automated tools because the hardware is working, just not as well as it could." Their automated detect-diagnose-remediate-repair pipeline, built around an open-source tool called **PCIcrawler**, has fixed "several thousand servers and server components" and continues to catch faults on "hundreds of servers every week" at fleet scale. That is the concrete answer to "is this a real job function or a hypothetical" — it is a standing, staffed program at every hyperscaler and neocloud.

Link degradation comes from a reseated card that didn't fully mate, a bad retimer, a dirty slot, a marginal riser, thermal effects on the PCIe PHY, a BIOS that bifurcated a slot wrong, or — as you'll see below — sometimes a driver regression that has nothing to do with hardware at all. The fix is trivial once you *see* it. Seeing it is the skill. At a neocloud you will be handed a fleet of rented boxes you did not build, and "is this link actually trained to spec?" is a question you must answer from the CLI in under a minute per card.

## What's new here (calibration)

You know PCIe exists and that GPUs sit on it. From CKA and on-prem work you've read `lspci` output for NIC and HBA enumeration, and this module's README already told us to skip PCIe history and basics. What's new:

- **Trained vs. maximum link state.** Every PCIe link *negotiates* speed and width at boot. `lspci` exposes both the capability (`LnkCap`) and what was actually negotiated (`LnkSta`). You've probably never compared them line by line. That comparison is the core diagnostic skill.
- **The topology tree as a diagnostic object.** `lspci -tv` is not just a device list — it's the actual PCIe hierarchy: root ports, switches, and endpoints, with the CPU at the root. Reading it tells you *how* a device reaches the CPU and whether it shares bandwidth.
- **Secondary error signals that precede a visible downgrade.** AER (Advanced Error Reporting) counters and LTSSM retraining events give you an earlier, more confident signal than a one-time `LnkSta` snapshot — this is what fleet-scale tools actually alert on, not a manual `lspci` read.
- **Bifurcation and retimers** as placement concerns — the physical-layer reasons a slot can only give you x8, or why a Gen5 signal needs a retimer to survive the trace length, and why "hardware fault" isn't always the right first hypothesis.

## Core concepts

### Per-lane bandwidth — the numbers to memorize

PCIe bandwidth is `per-lane rate × lane count`, one figure per direction (links are full-duplex — double it for aggregate). Gen3/4/5 use 128b/130b encoding, so the usable rate is very close to the raw GT/s figure; Gen6 switches to PAM4 signaling with 242-byte FLIT framing instead:

| Gen | Raw rate | Encoding | **Per lane, per direction** | **x16** | **x8** | **x4** |
|-----|----------|----------|-----------------------------|---------|--------|--------|
| 3   | 8 GT/s   | 128b/130b | ~0.985 GB/s | ~15.8 GB/s | ~7.9 GB/s | ~3.9 GB/s |
| 4   | 16 GT/s  | 128b/130b | ~1.97 GB/s  | ~31.5 GB/s | ~15.8 GB/s | ~7.9 GB/s |
| 5   | 32 GT/s  | 128b/130b | ~3.94 GB/s  | ~63 GB/s   | ~31.5 GB/s | ~15.8 GB/s |
| 6   | 64 GT/s (PAM4) | 242B FLIT | ~7.56 GB/s | ~121 GB/s | ~60 GB/s | ~30 GB/s |

(PCI-SIG's own PCIe 6.0 announcement confirms the 64 GT/s / PAM4 / FLIT figures — see References.)

Two facts fall out of this table that you should be able to state instantly:

- **Each generation doubles the per-lane rate**; halving the width halves bandwidth. So "Gen5 x16" and "Gen3 x16" differ by 4x; "Gen5 x16" and "Gen3 x8" differ by **8x**.
- The `LnkSta` **speed field is the raw GT/s number**, not the generation: `2.5` = Gen1, `5` = Gen2, `8` = Gen3, `16` = Gen4, `32` = Gen5, `64` = Gen6. Train yourself to read the GT/s and map it, because that's what the tool prints.

### Root ports, switches, endpoints — the hierarchy

PCIe is a tree rooted at the CPU. Three node types matter:

- **Root port** — a port of the **Root Complex**, integrated into the CPU (or its IO die). This is where the PCIe hierarchy physically originates. On a dual-socket box each socket has its own root complex, so each socket owns a set of root ports. A device hanging off socket 0's root port reaches socket 1's memory only by crossing the inter-socket link (UPI/xGMI) — this is the NUMA locality that lesson 02 covered, seen from the PCIe side.
- **Switch** — a fan-out device with one **upstream port** (toward the root) and several **downstream ports** (toward endpoints). A switch lets multiple endpoints share one link up to the root. Bandwidth *upstream of the switch is shared* by everything below it — this is why a PCIe switch feeding two GPUs and a NIC can become a bottleneck even if each device's own link is full width.
- **Endpoint** — a leaf: the GPU, the NIC, the NVMe drive. It terminates the tree.

You read this hierarchy from `lspci -tv`. Indentation = depth. The root ports sit just under the host bridge at the top; `+-[0000:00]` style host bridges are the roots. A `PCI bridge` line with things indented under it is a switch (or a root port, if it's the first hop from the host bridge). Endpoints are the deepest lines with a real device class (VGA/3D controller, Ethernet, Non-Volatile memory).

**Why the distinction is operational:** the path a GPU takes to reach a NIC depends entirely on whether they share a switch (fast, stays local, good for GPUDirect RDMA), sit under sibling root ports on the same socket (crosses the root complex), or sit on different sockets (crosses UPI — the worst case). `nvidia-smi topo -m` names exactly these cases: `PIX` = one bridge/switch hop, `PXB` = multiple bridges, `PHB` = traverses a PCIe host bridge (root complex), `NODE` = within a NUMA node across host bridges, `SYS` = crosses the socket interconnect. `lspci -tv` is the ground truth those labels summarize.

### Bifurcation

A physical x16 slot is 16 lanes wired to one or more root ports. **Bifurcation** is the BIOS splitting those 16 lanes into independent narrower links — `x8x8`, `x4x4x4x4`, `x8x4x4` — each of which trains as its own PCIe link to a separate device. It exists because a single x16 slot's worth of lanes is a scarce resource: a carrier/riser card can put two NVMe drives or a GPU + a NIC behind one physical slot, but only if the root port is configured to split.

Why it matters for GPU/NIC placement:

- A GPU dropped into a slot that BIOS has bifurcated to `x8x8` will train at **x8**, not x16 — instant 2x bandwidth loss that looks like nothing is wrong. Bifurcation misconfiguration is a leading cause of "why is my GPU at x8?"
- Conversely, deliberate bifurcation is how you *co-locate* a GPU and its rail-aligned NIC under the same root port so GPUDirect RDMA stays local (lesson 04). The skill is knowing whether a x8 reading is an intended split or an accident.
- Bifurcation is a **link-partitioning** decision, distinct from a switch. A switch adds silicon and shares upstream bandwidth; bifurcation just repartitions existing root-port lanes with no shared-bandwidth penalty, but it can't create more lanes than the slot has.
- **`lspci` alone cannot tell you *intent*.** You can infer bifurcation from width (`LnkSta` shows x8 where the slot is physically x16), but confirming it was deliberate versus accidental requires the BIOS setup menu or the vendor's slot documentation for that specific board. A width drop is a fact; "was this supposed to happen" is a separate question you answer out-of-band.

### Retimers

At Gen5's 32 GT/s, signal integrity over PCB traces degrades fast — beyond ~7-9 inches of trace the eye closes and the link won't train at full speed (it falls back to a lower gen, or flaps). A **retimer** is an active repeater that recovers the clock and data and re-drives a clean signal, extending reach. HGX/DGX baseboards use PCIe Gen5 retimers to carry each GPU's x16 stub from the GPU baseboard across to the CPU tray.

Gen5's higher signaling rate also shrinks the *unit interval* — the time budget for one bit — enough that clock jitter becomes a first-order design constraint, not just trace length. Retimers exist for both reasons: they extend physical reach **and** clean up jitter accumulated across connectors and PCB vias. This is also why Gen5+ platforms lean on **Precision Time Measurement (PTM)**-aware link training: crossing a clock domain cleanly matters more as the unit interval shrinks.

Operationally: a **failing or marginal retimer** is a classic cause of a link training one generation low (Gen4 instead of Gen5) or intermittently. When you see a Gen5-capable card stuck at Gen4 x16 with no bifurcation involved, a retimer or the physical channel is the prime suspect.

### Reading LnkCap vs LnkSta — the core skill

`lspci -vvv` prints, in the PCI Express capability block, two lines you compare directly:

```
LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM L1, Exit Latency L1 <64us
LnkSta: Speed 32GT/s, Width x16
```

- **`LnkCap`** = the maximum this link *can* do (the lower of the two link partners' caps).
- **`LnkSta`** = what it *actually negotiated* at training time.

A healthy H100 link reads `LnkCap ... Speed 32GT/s, Width x16` **and** `LnkSta: Speed 32GT/s, Width x16`. A degraded one reads:

```
LnkCap: Port #0, Speed 32GT/s, Width x16
LnkSta: Speed 8GT/s (downgraded), Width x8 (downgraded)
```

The kernel prints **`(downgraded)`** when `LnkSta` is below `LnkCap` — that word is your alarm. But do not rely on it alone: some tool/kernel versions omit it, and a link can be at full width but wrong speed or vice versa. **Always compare the four numbers yourself.**

Two important subtleties:

- **Active Power Management flap.** With ASPM enabled, an idle link can drop to Gen1 to save power and retrain up under load. So a `2.5GT/s` reading on an idle card may be benign — re-check `LnkSta` while the GPU is under load (or after disabling ASPM) before declaring a fault. Under a real workload the link must be at full spec.
- **Two ends, two capabilities.** `LnkCap` on the endpoint reflects the negotiated minimum. If a Gen5 GPU sits behind a Gen4-only switch or a Gen4 riser, `LnkCap` itself will read 16GT/s — the ceiling was set by the weakest link in the chain, not the card. Walk the tree with `lspci -tv` to find *which* hop caps it. Contrast this with bifurcation: a width loss from bifurcation shows up as a **reduced `LnkCap` width** if you read the slot's documented maximum, whereas a genuine link fault shows `LnkCap` at full spec but `LnkSta` degraded. `LnkCap` telling the truth about a smaller ceiling is a config fact; `LnkSta` falling short of a full `LnkCap` is a fault.

`lspci -vvv` needs root to show the full capability block (`sudo`); without it the `LnkCap`/`LnkSta` lines are often blank or `<access denied>`.

### Beyond LnkSta: AER counters and link retraining

A single `lspci -vvv` snapshot only tells you the link's *current* state. Two other signals tell you whether that state is stable, and give you a warning before the link ever visibly downtrains:

- **AER (Advanced Error Reporting).** PCIe devices report correctable, uncorrectable, and fatal errors through the AER capability. You see them as `pcieport ... AER: Corrected error received` lines in `dmesg`, or read the raw counters via `lspci -vvv` (`UESta`/`CESta` registers) or `/sys/bus/pci/devices/<bdf>/aer_dev_correctable`. A correctable-error count that climbs over time — even while `LnkSta` still matches `LnkCap` — is an early warning of a marginal link, before it ever actually downtrains. This is exactly the counter class fleet-scale tools like Meta's PCIcrawler watch continuously, rather than waiting for a visible speed/width drop.
- **LTSSM retraining.** A PCIe link doesn't train once at boot and stay fixed forever. The **Link Training and Status State Machine (LTSSM)** can retrain the link — after an error, an ASPM power-state transition, or a hot-plug-like event — and a marginal physical link can be seen **flapping** between speeds in `dmesg` (`pcieport ...: retrain` messages). That is a distinct failure signature from a link that is simply stuck permanently low: a flapping link points at a marginal physical connection (reseat, oxidation, thermal), while a link stuck low from boot points more toward a fixed cause like a Gen4-capped riser or a driver issue.

For the rare case where `lspci`'s decoded output doesn't show a capability register you need, `setpci -s <bdf> CAP_EXP+12.w` reads the Link Control 2 register directly — the escape hatch when the friendly tool's decode isn't enough.

## Perspectives

**Developer.** PCIe link state is invisible from CUDA or framework code entirely. A developer sees only "my job is slow" and has no API to query `LnkSta` — there is no `torch.cuda.link_status()`. This is squarely a platform engineer's diagnostic domain, which is exactly why it is an interview differentiator: the failure is real, common, and outside the tool most engineers reach for first.

**Operator.** Link-state sweeps belong in node acceptance testing and periodic fleet health checks, not ad-hoc debugging triggered by a slow job. Meta's PCIcrawler turns this from a CLI habit into continuous automated coverage — the operational maturity curve here runs from "I know the `lspci` incantation" to "every node gets swept on a schedule and a regression pages someone."

**Hardware / signal-integrity.** A degraded link is a physical-layer symptom: bad seating, oxidized contact, a marginal retimer, a trace-length or EMI issue, or a thermal effect on the PHY. The CLI only shows you the *symptom* — a speed or width mismatch — not the root cause. Once you've confirmed the degradation with `lspci`, the correct next step is often a hardware ticket (reseat, swap, retimer replacement), not more software debugging.

**Economics / reliability.** Meta's own numbers — several thousand servers/components fixed, hundreds caught weekly — are the concrete evidence that PCIe faults are common enough at fleet scale to justify a dedicated, automated pipeline rather than reactive debugging. At neocloud scale, the cost of *not* running this sweep is paying full price for GPUs delivering a fraction of their host bandwidth, for as long as nobody looks.

## Real-world use cases

- **Meta Engineering — "How Facebook deals with PCIe faults to keep our data centers running reliably" (2021).** Describes Meta's automated PCIe fault detect→diagnose→remediate→repair pipeline and their open-source tool **PCIcrawler**. Names "bad link speed and link width" as a fault class that's hard to catch without automation, and reports the program fixed several thousand servers/components and continues catching faults on hundreds of servers weekly. What it shows: this is a standing, staffed production function at hyperscaler scale, not a one-off debugging trick. https://engineering.fb.com/2021/06/02/data-center-engineering/how-facebook-deals-with-pcie-faults-to-keep-our-data-centers-running-reliably/
- **NVIDIA/nccl GitHub Issue #246 — topology-tool mismatch.** A real production bug report: on an 8×V100 + Mellanox RDMA NIC box, NCCL's own topology-detection log reported `PIX`/`PXB` for GPU-NIC pairs where `nvidia-smi topo -m` reported `PHB`. What it shows: two "official" NVIDIA tools can genuinely disagree about the same physical topology — reconciling tools against each other, not trusting one blindly, is a real skill this module (and lesson 08's capstone) builds. https://github.com/NVIDIA/nccl/issues/246
- **NVIDIA/open-gpu-kernel-modules GitHub Issue #1010 — RTX 5070 Ti falls back to Gen1 on Linux.** A driver-regression case where a Blackwell-era card trained at Gen1 (2.5 GT/s) due to a kernel-module bug, producing a symptom identical to a hardware-caused link downgrade. What it shows: don't default to blaming hardware — a driver/firmware regression can produce the exact same `LnkSta` mismatch a bad retimer would. https://github.com/NVIDIA/open-gpu-kernel-modules/issues/1010

## Worked example

A rented "8x H100" node. `nvidia-smi` shows all eight GPUs, 100% util on a running job, but throughput is roughly half of a known-good node. Trace it.

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

The GPU at `e3:00.0` hangs off root port `e0:01.1`, then through two bridges (`e1-e3`, `e2-e3`) — those intermediate `PCI bridge` hops are **retimers/switch stages** on the baseboard. The `[0000:e0]` and `[0000:00]` host bridges are the **two sockets' root complexes**: BDFs starting `e0`/`c0`/`a0`... belong to socket 1, low buses (`00`/`20`) to socket 0. So `e3:00.0` is a socket-1 GPU.

**3. Compare LnkCap vs LnkSta on the slow GPU:**

```
$ sudo lspci -vvv -s 0000:e3:00.0 | grep -E 'LnkCap:|LnkSta:'
        LnkCap: Port #0, Speed 32GT/s, Width x16, ASPM not supported
        LnkSta: Speed 16GT/s (downgraded), Width x16
```

**Read:** `LnkCap` says the card and its channel are Gen5-capable (32GT/s) at full x16. But `LnkSta` trained at **16GT/s = Gen4** — one generation low, width fine. `(downgraded)` confirms it. That's `31.5 GB/s` instead of `63 GB/s`: **half bandwidth**, matching the observed throughput. Width is x16, so this is *not* bifurcation — it's a **speed** fault: marginal retimer, dirty/oxidized contact, or a thermal/signal-integrity issue on that GPU's Gen5 channel.

**4. Check the secondary signal — is it stable or getting worse:**

```
$ cat /sys/bus/pci/devices/0000:e3:00.0/aer_dev_correctable
```

A nonzero and climbing correctable-error count confirms this is a genuine, worsening physical-layer problem — not a one-time boot fluke. `dmesg | grep -i 'e3:00.0'` showing repeated `pcieport ...: retrain` lines would additionally tell you the link is *flapping*, not just stuck low, which points even more specifically at a marginal connection rather than a fixed Gen4-capped component.

**5. Confirm it's channel-specific, not fleet-wide:** the same one-liner across all eight shows the other seven at `Speed 32GT/s, Width x16` with zero AER counts. Only `e3:00.0` is degraded. That isolates the fault to one GPU's physical link — a hardware ticket for that slot/retimer, not a BIOS or firmware-wide problem. **Outcome:** the dashboards showed 100% util the whole time; the 30-second `LnkCap`-vs-`LnkSta` sweep, backed by an AER check, found the 2x loss they couldn't — and gave a confidence signal (climbing counters) that this is a real degrading fault, not noise.

## Practice

Feeds the **Topology Teardown** deliverable. Work on a real GPU box (a rented instance is fine).

1. **Capture the tree.** Run `lspci -tv > topology-tree.txt`. Annotate it: mark each host bridge as a **root complex / root port**, each intermediate `PCI bridge` as a **switch or retimer stage**, and each NVIDIA/Mellanox/NVMe line as an **endpoint**. Note which socket (root complex) each GPU hangs off.
2. **Sweep every GPU's link.** For each GPU BDF: `sudo lspci -vvv -s <BDF> | grep -E 'LnkCap:|LnkSta:'`. Build a table: `GPU | BDF | LnkCap (gen×width) | LnkSta (gen×width) | GB/s each way | OK / DEGRADED`.
3. **Flag and diagnose.** For any device whose `LnkSta` is below its `LnkCap`, state whether it's a **speed** downgrade, a **width** downgrade, or both, and give the most likely cause (bifurcation for a width drop to x8/x4; retimer/channel/ASPM for a speed drop). Re-check any suspicious link under GPU load to rule out ASPM idle downclocking.
4. **Check AER for every GPU**, even ones reading full spec: `cat /sys/bus/pci/devices/<BDF>/aer_dev_correctable`. A nonzero-but-`LnkSta`-full-spec reading is worth a note — a link that hasn't downtrained yet but is accumulating correctable errors.
5. **Extend to the NICs.** Repeat the sweep for the ConnectX NICs — a ConnectX-7 wants Gen5 x16 too; a NIC at Gen4 x8 quietly halves your RDMA ceiling.

**Acceptance:** a committed note in the deliverable listing, for every GPU (and NIC), its **trained link vs. its maximum** and its **AER correctable-error count**, with each device marked OK or DEGRADED and every DEGRADED entry annotated with speed-vs-width and a probable cause. A node with all eight GPUs at `32GT/s x16` and zero AER counts is a valid — and expected — result; the note must *prove* that by showing the numbers, not assert it.

## Common pitfalls

1. **Assuming `LnkSta` alone is the whole story.** A single snapshot can't distinguish "always been fine" from "about to fail." AER correctable-error counters climbing is often the *earlier* warning sign, before a link ever actually downtrains — check both, not just one.
2. **Blaming hardware by default.** The RTX 5070 Ti Gen1-fallback kernel-module bug (above) is a real counter-example: driver/firmware regressions can produce symptoms identical to a bad retimer or reseated card. Check driver version and known issues before opening a hardware ticket.
3. **Not re-checking under load.** ASPM can legitimately drop an idle link to Gen1 to save power; a `2.5GT/s` reading on an idle card is not automatically a fault. Always confirm under a real workload.
4. **Trusting one tool's topology label over another without reconciling.** The NCCL issue #246 example above is a real case of exactly this — `PXB` vs `PHB` disagreement between two official NVIDIA tools on the same box. When tools disagree, walk the physical tree with `lspci -tv` as the tiebreaker.
5. **Assuming a width-loss from bifurcation and a genuine link fault look the same.** They're diagnostically distinguishable: bifurcation only ever shows as a *width* drop, with `LnkCap` itself already reading the reduced number once you check the slot's documented maximum; a genuine fault shows `LnkCap` at full spec but `LnkSta` degraded below it.

## Self-check

- A GPU trained at Gen3 x8 versus a spec of Gen5 x16 — what is the bandwidth ratio, and how do you detect it from the CLI? **Answer:** Gen5 x16 ≈ 63 GB/s/direction; Gen3 x8 ≈ 7.9 GB/s/direction — a factor of **8x** (4x from the two generation steps 5→3, 2x from x16→x8). Detect it with `sudo lspci -vvv -s <BDF> | grep -E 'LnkCap:|LnkSta:'`: `LnkCap` shows `32GT/s, Width x16` while `LnkSta` shows `8GT/s (downgraded), Width x8 (downgraded)`. `nvidia-smi` will still report high utilization, which is exactly why you must read the link state directly.
- How do you tell a root port, a switch, and an endpoint apart in `lspci -tv`? **Answer:** The **root port** is the first hop under a host-bridge line (`-+-[0000:00]`) at the top of the tree — it's a port of the CPU's root complex where the hierarchy originates. A **switch** is a `PCI bridge` node with *multiple* devices/bridges indented beneath it (one upstream port, several downstream); everything below it shares its upstream link. An **endpoint** is the deepest leaf with a real device class (3D controller, Ethernet, Non-Volatile memory) and nothing indented under it. Depth = indentation; the CPU is the root.
- What is bifurcation and when does it matter for GPU/NIC placement? **Answer:** Bifurcation is the BIOS splitting one physical slot's x16 lanes into independent narrower links (`x8x8`, `x4x4x4x4`), each training as its own PCIe link to a separate device. It matters two ways: (1) a GPU in a slot accidentally bifurcated to `x8x8` trains at **x8** — a silent 2x loss you'd catch as a *width* downgrade in `LnkSta`; (2) deliberate bifurcation is how you place a GPU and its rail-aligned NIC under one root port so GPUDirect RDMA stays local. Unlike a switch it adds no shared-bandwidth penalty, but it can't create more lanes than the slot physically has.
- What's the difference between a link that's *always* trained low and one that's *flapping* between speeds, and what does each suggest? **Answer:** A link stuck permanently low (e.g. Gen4 every time you check, at boot and under load) points toward a fixed cause — a Gen4-capped riser/switch hop, or a marginal retimer that never trains up. A link seen **flapping** between speeds in `dmesg` (`pcieport ...: retrain` messages) points toward a marginal physical connection — a partially-seated card, oxidized contact, or a thermal effect — that is intermittently failing link training rather than being fundamentally capped.
- Besides `LnkSta`, what secondary signal would make you more confident a link is genuinely degrading rather than a one-time boot fluke? **Answer:** AER (Advanced Error Reporting) correctable-error counters climbing over time — readable via `lspci -vvv`'s `CESta` register or `/sys/bus/pci/devices/<bdf>/aer_dev_correctable`. A rising count, even before `LnkSta` ever visibly drops below `LnkCap`, is the earlier and more reliable evidence of a real, worsening physical-layer problem.

## Connections & what's next

This lesson gives you the vocabulary — root port, switch, endpoint, `LnkCap`/`LnkSta`, `PIX`/`PXB`/`PHB`/`SYS` — that lesson 04 assumes you already have when it draws the full 8-GPU reference node. Lesson 05's Kubernetes topology alignment builds on the same NUMA-locality argument from the scheduler's side; lesson 06's storage placement reuses the exact `PIX`/`PXB`/`SYS` reasoning for NVMe and GPUDirect Storage instead of a NIC. Lesson 08's capstone is where you reconcile `lspci`, `nvidia-smi topo -m`, `lstopo`, and `numactl` into one picture on a real node — the AER and link-training skills from this lesson are what let you trust that picture instead of just reading it.

Next: **lesson 04** takes the single-link skill you just built and applies it at the scale of a whole 8-GPU server — the canonical HGX/DGX topology, where GPU-GPU traffic bypasses PCIe entirely over NVLink, and where PCIe's job narrows to GPU↔host, GPU↔NIC, and GPU↔storage.

## References & further reading

**Primary sources**
- Linux kernel PCI documentation — https://docs.kernel.org/PCI/index.html — the authoritative reference for how the kernel enumerates and reports the PCI hierarchy, link training, and the sysfs/`lspci` fields you're reading.
- `lspci(8)` / pciutils — https://man7.org/linux/man-pages/man8/lspci.8.html and https://github.com/pciutils/pciutils — flags that matter here: `-tv` (tree), `-vvv` (full capability blocks incl. `LnkCap`/`LnkSta`), `-D` (domain BDFs), `-s` (select a device).
- PCI-SIG, "PCI-SIG Releases PCIe 6.0 Specification" — https://www.businesswire.com/news/home/20220111005011/en/PCI-SIG-Releases-PCIe-6.0-Specification-Delivering-Record-Performance-to-Power-Big-Data-Applications — primary confirmation of the 64 GT/s / PAM4 / FLIT figures used in the bandwidth table above.

**Real-world engineering blogs**
- Meta Engineering, "How Facebook deals with PCIe faults to keep our data centers running reliably" — https://engineering.fb.com/2021/06/02/data-center-engineering/how-facebook-deals-with-pcie-faults-to-keep-our-data-centers-running-reliably/ — what it shows: a real, staffed, fleet-scale PCIe fault detect/remediate pipeline (PCIcrawler) and why "the hardware works, just not as well as it could" is a distinct, hard-to-detect fault class.
- NVIDIA/nccl GitHub Issue #246 — https://github.com/NVIDIA/nccl/issues/246 — what it shows: two official NVIDIA tools (NCCL's own topology log vs `nvidia-smi topo -m`) genuinely disagreeing on the same box's GPU-NIC topology.
- NVIDIA/open-gpu-kernel-modules GitHub Issue #1010 — https://github.com/NVIDIA/open-gpu-kernel-modules/issues/1010 — what it shows: a driver regression producing a Gen1 link-fallback symptom identical to a hardware fault, on a Blackwell-era card.

**Deeper dives**
- Chris's Wiki (utcc.utoronto.ca), "PCIe Topology and Lanes" — https://utcc.utoronto.ca/~cks/space/blog/linux/PCIeTopologyAndLanes — a practitioner deep-dive on reading `lspci -tv` and lane topology in more detail than this lesson has room for.
