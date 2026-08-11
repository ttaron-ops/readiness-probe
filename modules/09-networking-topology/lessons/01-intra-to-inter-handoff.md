---
lesson: "09.1"
title: "From intra-node to inter-node: extending the topology matrix past the NIC"
module: "09"
concept: "From intra-node to inter-node: extending the topology matrix past the NIC"
status: not-started
est_time: "7h"
prev: null
next: "02-inter-node-fabric.md"
artifacts: []
sources: 10
---

# 09.1 · From intra-node to inter-node: extending the topology matrix past the NIC

> **Concept.** `nvidia-smi topo -m` stops at the PCIe edge of one box; the fabric picks up at the NIC — learn to read the full `GPU → NIC → leaf → spine` path, name the *rail* each GPU rides, and turn 02b's "same root complex" rule into "same rail" across nodes.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Where this fits

02b ended with you fluent in one box: the PCIe tree, the NVLink domain, and `nvidia-smi topo -m`'s `PIX`/`PXB`/`PHB`/`NODE`/`SYS` codes telling you which GPU can DMA to which NIC without a CPU bounce. That fluency has a hard boundary — it describes a single chassis and goes silent the moment a byte needs to reach a GPU in a different server. This lesson walks you across that boundary: it names the *rail* (the inter-node generalization of "same root complex"), traces the exact hop sequence from GPU HBM to a leaf switch, and gives you the vocabulary — rail, leaf, NIC-per-GPU provisioning — that 09.2 builds the Clos fabric on top of. Nothing here works without 02b; nothing in 09.2 works without this lesson.

## Why this matters

Every placement and procurement argument you make above eight GPUs lives or dies on one question the topology matrix cannot answer alone: *when a collective leaves the box, does it stay on its rail or does it get funneled the wrong way?* Get the GPU-to-NIC mapping wrong and GPUDirect RDMA silently falls back to a bounce through host memory — you keep the GPUs, lose the wire, and eat a step-time regression no dashboard names. In an interview for the platform track at a GPU neocloud, being able to draw the `GPU → NIC → leaf → spine` path and say exactly where NVLink stops — and where, on the newest hardware, it does *not* stop — is the line between "I run Kubernetes near GPUs" and "I can argue fabric with the network team."

## What's new here (calibration)

- **Already yours (02b, 08, 06):** the PCIe tree, NVLink/NVSwitch as a board-level fabric, rail alignment as an intra-node property, the GPUDirect same-root-complex rule, reading `topo -m`'s connectivity codes, ring/tree all-reduce as the traffic pattern, and topology-aware gang placement. We reference these, we do not re-teach them.
- **Genuinely new:** the *inter-node* rail — a vertical slice of GPU-N-to-leaf-N cabling spanning every node in a cluster, not a single board; the exact hop sequence from PCIe switch to leaf/ToR switch; per-generation NVLink bandwidth numbers (H100 vs GB200/GB300) and the real NIC-per-GPU provisioning ratio vendors actually ship (it is not the clean 1:1 you'd guess); and the one genuine exception to "NVLink never leaves the chassis" — the GB200/GB300 NVL72 rack-scale NVLink domain.
- **Deliberately deferred:** the shape and cost of the network *above* the leaf (Clos tiers, bisection, oversubscription) is 09.2's job, not this lesson's. Here you only need to know where the fabric begins and which NIC is the correct on-ramp.

## Core concepts

### The full path, one hop at a time

A collective byte leaving GPU0 on node A for GPU0 on node B traverses, in order:

```
GPU0 ──NVLink──► (stays on-node for intra-node peers)
GPU0 ──PCIe──► PCIe switch ──► NIC (mlx5_0)      ← still inside the box; 02b's territory
NIC  ──cable──► leaf / ToR switch (rail switch)   ← FABRIC BEGINS HERE
leaf ──cable──► spine / aggregation switch        ← only if the path must leave the rack/pod
spine ─────────► leaf ─► NIC ─► PCIe ─► GPU0 (node B)
```

Two facts to burn in:

1. **NVLink never leaves the chassis — on H100-class hardware.** There is no NVLink cable between servers in a standard 8-GPU HGX/DGX rack. NVLink and NVSwitch are a *board-and-backplane* fabric bounded by the NVLink domain (02b). The moment traffic must reach a GPU outside that domain, it exits through the **NIC** and onto the **switched fabric**. That transition — PCIe-to-NIC-to-cable — is the intra/inter boundary. (The next subsection covers the real exception: GB200/GB300 NVL72, where the "chassis" is redefined to mean a whole rack.)
2. **The NIC is the handoff.** GPUDirect RDMA (02b) lets the NIC DMA straight out of GPU HBM with no CPU bounce — *provided* the GPU and NIC sit under the same PCIe switch / root complex. That is the whole game: keep the GPU→NIC hop a clean PCIe-switch (`PIX`/`PXB`-adjacent) path, and the fabric sees a zero-copy source.

### What a "rail" is

An 8-GPU HGX node has, on current builds, **one NIC per GPU** — eight ConnectX-7 (400G NDR) or ConnectX-8 (800G XDR) HCAs, each paired to one GPU through the PCIe switch that GPU hangs off. A **rail** is the set of *same-indexed* GPU→NIC pairs across every node, cabled into the *same leaf switch*:

- GPU0 of every node → its local NIC → **leaf switch 0** = **rail 0**
- GPU1 of every node → its local NIC → **leaf switch 1** = **rail 1**
- … GPU7 → leaf 7 = rail 7.

So a rail is a *vertical slice* of the cluster: one GPU position, one NIC per node, one leaf switch, spanning all nodes. This is **rail-optimized** cabling, and it is the inter-node generalization of 02b's rail alignment. In 02b, "rail-aligned" meant a GPU and its NIC share a PCIe branch on the *board*. Here it means GPU-N's NIC on every node lands on the *same leaf*, so GPU-N-to-GPU-N traffic across nodes crosses exactly one switch and never touches the spine.

### The rule promotion: "same root complex" → "same rail"

02b's law for zero-copy inside the box:

> GPUDirect RDMA needs the GPU and NIC under the **same PCIe root complex / switch** — otherwise the DMA crosses the CPU (or worse, the inter-socket link), which the matrix flags as `NODE`/`SYS`, and the copy falls back through host memory.

Its inter-node twin:

> For a collective to stay cheap across nodes, GPU-N should talk to GPU-N — **same rail** — so its bytes ride its own leaf and never climb to the spine. Cross-*rail* traffic (GPU3 on node A wanting GPU5 on node B) has two options: (a) hop laterally over NVLink to the local GPU5 first, *then* go out rail 5's NIC — cheap, because NVLink is board-abundant bandwidth; or (b) go out rail 3 and traverse the spine to reach rail 5 — expensive, and the reason you keep collectives rail-local.

That preference — NVLink absorbs the cross-rail shuffle so the fabric only ever carries rail-local traffic — is the load-bearing idea that makes spine oversubscription safe (lesson 09.2). Here, just hold the mapping: **GPU index ↔ rail ↔ leaf switch ↔ its dedicated NIC.**

### Reading the matrix for the NIC columns

`nvidia-smi topo -m` prints GPU×GPU *and* GPU×NIC (`mlx5_*` / `HCA`) connectivity. The GPU-to-NIC cells are what you now care about. The codes, best to worst for GPUDirect:

| Code | Path GPU→NIC | GPUDirect RDMA? |
|---|---|---|
| `PIX` | single PCIe switch (bridge) | Best — clean zero-copy |
| `PXB` | multiple PCIe bridges (switch cascade) | Works, extra hops/latency |
| `PHB` | via the PCIe **Host Bridge** (CPU root port) | Marginal — CPU root involved |
| `NODE` | across PCIe within one NUMA node, off the root complex | Degraded |
| `SYS` | across the inter-socket link (QPI/UPI) between NUMA nodes | **Worst — crosses CPUs; RDMA falls back / craters** |

The pairing you want: each GPU's *chosen* NIC shows `PIX` (or at worst `PXB`). A GPU whose only reachable NIC reads `PHB`/`NODE`/`SYS` is mis-paired — its "GPUDirect" path is a CPU bounce wearing a costume. `SYS` in particular means the DMA would cross the UPI/QPI link between the two CPU sockets: latency spikes, bandwidth collapses to inter-socket limits, and the NIC can no longer DMA GPU memory directly, so the driver stages the transfer through pinned host memory.

**One vendor/generation caveat, worth stating plainly:** these five codes and their ranking are a property of the PCIe topology `nvidia-smi` reports for *that* SKU's motherboard and BIOS layout — they are not a universal law you memorize once. A different HGX generation, a different CPU vendor, or (as below) an NVLink-connected node with no meaningful PCIe-switch hierarchy between GPU and NIC can print a different pattern. Read the matrix per box, not per assumption.

### Per-generation NVLink numbers, and why cross-rail traffic prefers NVLink over the NIC

The "hop sideways over NVLink before you hop out the NIC" preference from the rule above only makes sense if you know the bandwidth gap. It is generation-specific:

| Platform | NVLink generation | Per-GPU NVLink bandwidth | NIC bandwidth (per GPU) |
|---|---|---|---|
| H100 HGX (8-GPU) | NVLink 4 | ~900 GB/s aggregate bidirectional (~450 GB/s each way) | 400 Gb/s (ConnectX-7) or 800 Gb/s (ConnectX-8) = 50–100 GB/s |
| GB200 / GB300 NVL72 | NVLink 5 | ~1.8 TB/s aggregate bidirectional (18 links × 100 GB/s) | 800 Gb/s (ConnectX-8) = 100 GB/s |

On an H100 node, moving a tensor from GPU3 to GPU5 *within the node* over NVLink is ~9–18× the bandwidth and a fraction of the latency of pushing it out rail 3's NIC, across a leaf, maybe up a spine, and back down rail 5. On GB200/GB300, the gap is even larger in absolute terms (1.8 TB/s vs 100 GB/s — 18×). The fabric is precious; NVLink is abundant. Rail-optimized software (NCCL with rail-aware topology detection) exploits exactly this: it shuffles cross-rail data *sideways over NVLink* to line every GPU up with its own rail, then does the inter-node leg rail-local. The NIC only ever carries GPU-N ↔ GPU-N.

### The NVL72 exception — when NVLink *does* leave the chassis

"NVLink never leaves the chassis" is the right rule for an 8-GPU HGX box, and it is *wrong* if you apply it unmodified to NVIDIA's current top-of-line rack: **GB200 NVL72** (and GB300 NVL72). There, 72 Blackwell GPUs and 36 Grace CPUs sit in one liquid-cooled rack, connected by a 5th-generation NVLink Switch fabric, as **one single non-blocking NVLink domain — 130 TB/s of aggregate GPU-to-GPU bandwidth across the whole rack**. Every GPU is one NVLink hop from every other GPU in the rack; there is no PCIe-switch/NIC hop *inside* that 72-GPU domain at all.

The boundary didn't disappear — it moved. On an 8-GPU HGX box, the NVLink domain's edge is the board. On NVL72, the NVLink domain's edge is the *rack*. The reason it can move that far: the external 5th-gen NVLink Switch ships **144 ports at 14.4 TB/s** per switch tray, and nine of those switch trays fully interconnect all 18 NVLink ports on each of the 72 GPUs. NVIDIA's NVLink Switch System pushes this further still — a multi-rack NVLink Switch fabric can bind **up to 576 GPUs into a single non-blocking NVLink compute fabric**, spanning racks with copper/optical NVLink cabling, still *not* the inter-node InfiniBand/Ethernet fabric this lesson otherwise describes. Treat this as the one load-bearing exception to memorize, not a reason to discard the rule: on 8-GPU HGX nodes (still the majority of the installed base as of 2026), NVLink is chassis-bound and the fabric begins at the NIC exactly as described above.

### Provisioning the rail: how many NICs per GPU, really

The clean mental model — one NIC per GPU, full stop — is close but not what current reference architectures actually ship. NVIDIA's HGX AI Factory reference build (the "**2-8-9-800**" configuration: 2 CPUs, 8 GPUs, **9 NICs**, 800 Gb/s per GPU-facing NIC) puts **nine** NICs on an 8-GPU node, not eight. Eight are the GPU-indexed rail NICs — one per GPU, each carrying GPUDirect RDMA traffic for the compute fabric. The ninth is a separate NIC deliberately kept **off** the compute rail, dedicated to storage and cluster-management traffic. It exists so that checkpoint I/O, dataset staging, and control-plane chatter never contend with a GPU's dedicated rail bandwidth — isolation spend, not a rounding error, and a detail worth naming precisely if you're asked to size a build's NIC count.

On this reference build the GPU-facing NICs are ConnectX-8 SuperNICs: **800 Gb/s**, native **PCIe Gen6 x16**, configurable as a single InfiniBand XDR port or dual 400G Ethernet ports. Quoting this exact split (8 rail + 1 off-rail = 9) is the realistic modern number to have ready in a procurement conversation, versus the naive 8-NIC/8-GPU assumption an interviewer may expect you to correct.

## Perspectives

**Developer.** The rail is invisible right up until it isn't. NCCL auto-detects topology and picks the rail-aligned NIC for you — until a mis-cabled node or a `SYS`-only GPU-NIC pairing forces a fallback, and your all-reduce step time quietly doubles with no error, no crash, just a number that doesn't match the paper you benchmarked against.

**Operator.** Rail cabling is decided at data-center build time — which NIC plugs into which leaf — and it is expensive and disruptive to re-cable a live rack. A rail-mapping mistake made once during bring-up becomes a standing tax paid on every job for the life of the cluster, which is why operators treat the `topo -m` + cabling audit as a release gate, not a one-time sanity check.

**Hardware/kernel.** The `PIX`/`PXB`/`PHB`/`NODE`/`SYS` codes are static facts about a specific motherboard's PCIe topology and BIOS layout — they don't change at runtime and they don't need re-checking per job. Read them once per server SKU, record the GPU→rail→NIC map, and reuse it; re-deriving it per job is wasted operator time.

**Economics.** Every GPU-indexed NIC is a discrete capex line, and the "extra" ninth NIC in the 2-8-9-800 reference build is not vendor upsell — it's deliberate traffic isolation. Buying eight rail NICs plus one management NIC is a considered decision to keep storage/checkpoint I/O off the collective's dedicated bandwidth, and defending that line item with the isolation argument (rather than just "the reference architecture says so") is the kind of reasoning a procurement review expects.

## Real-world use cases

- **CoreWeave — GB200 NVL72 general availability.** [CoreWeave: First cloud provider to announce GA of NVIDIA GB200 NVL72 instances](https://www.coreweave.com/news/coreweave-first-cloud-provider-to-announce-general-availability-of-nvidia-gb200-nvl72-instances) — a real production deployment of rack-scale NVLink (72 GPUs, one non-blocking domain) paired with Quantum-2 InfiniBand at 400 Gb/s per GPU in a rail-optimized topology, scaling to clusters up to 110,000 GPUs, with SHARP in-network reduction accelerating collectives. Shows the NVL72 exception and the GPU→NIC→leaf rail pattern coexisting in the same real fabric.
- **Microsoft Azure "Eagle" supercomputer.** [ServeTHome: Microsoft Azure Eagle is a Paradigm-Shifting Cloud Supercomputer](https://www.servethehome.com/microsoft-azure-eagle-is-a-paradigm-shifting-cloud-supercomputer-nvidia-intel/) and the [TOP500 system record](https://top500.org/system/180236/) — 14,400 H100 GPUs across 1,800 nodes, cabled on Quantum-2 CX7 InfiniBand, ranked #3 on the November 2023 TOP500 list. A concrete, independently-verified instance of the GPU→NIC→leaf cabling this lesson describes, at cloud scale rather than a single lab machine.
- **ByteDance MegaScale (NSDI '24).** [USENIX: MegaScale — Scaling Large Language Model Training to More Than 10,000 GPUs](https://www.usenix.org/conference/nsdi24/presentation/jiang-ziheng) — a production system, not Meta's, independently validating the same rail-aware design: measured 55.2% model-FLOPs utilization training a 175B-parameter model on 12,288 GPUs. Confirms rail-optimized GPU-to-NIC mapping is an industry pattern, not a single hyperscaler's idiosyncrasy.

## Worked example

A captured `topo -m` fragment from an 8-GPU HGX H100 node (two sockets, eight GPUs, eight `mlx5` NICs). Reading GPU0's and GPU4's rows against the NIC columns:

```
        GPU0  GPU4   mlx5_0  mlx5_4   CPU-Affinity  NUMA
GPU0     X    NV8     PIX     SYS      0-47          0
GPU4    NV8    X      SYS     PIX      48-95         1
```

- **GPU0 → mlx5_0 = `PIX`.** Same PCIe switch, same root complex, same NUMA node 0. This is GPU0's rail NIC: clean GPUDirect RDMA. GPU0 rides **rail 0**, out `mlx5_0`, to leaf 0.
- **GPU0 → mlx5_4 = `SYS`.** Reaching `mlx5_4` means crossing the UPI link to socket 1. If placement forced GPU0 onto `mlx5_4`, RDMA falls back through host memory — do not do this.
- **GPU4 → mlx5_4 = `PIX`.** GPU4's rail NIC (rail 4, leaf 4, NUMA 1). `GPU4 → mlx5_0 = SYS` is the mirror mistake.
- **GPU0 ↔ GPU4 = `NV8`.** Eight NVLink connections on-board. So GPU0-needs-GPU4 traffic stays on NVLink (~900 GB/s aggregate), *not* out either NIC — exactly the cross-rail-prefers-NVLink rule.

**Mapping produced:** `GPU0 → rail 0 → mlx5_0 (PIX)`, `GPU4 → rail 4 → mlx5_4 (PIX)`. Pattern: GPU-N pairs with `mlx5_N` at `PIX`; any `SYS` cell is a cross-socket trap. Repeat for GPU0–7 and you have the node's rail map.

**Contrast, briefly:** on a GB200 NVL72 tray, this same exercise looks different — GPU-to-GPU cells inside the 72-GPU domain read as NVLink connectivity throughout (no `PIX`/`SYS` gradient to reason about *inside* the domain), and the meaningful GPU→NIC mapping only starts to matter again at the rack's edge, where its NICs connect out to the inter-rack InfiniAnd/Ethernet fabric. Same rail concept, different chassis boundary.

## Practice

Feeds the deliverable **Network architecture read**.

**Task.** Take a `nvidia-smi topo -m` matrix — from any multi-GPU box you can reach (`nvidia-smi topo -m`), or a captured 8-GPU HGX matrix if you have no cluster. For **each GPU**, produce the mapping:

`GPU-N → rail-N → NIC (mlx5_?) → connectivity code`

**Requirements / acceptance:**
1. A complete **GPU→rail→NIC table** for one node — all GPUs, each with its rail number, its chosen NIC, and the GPU→NIC connectivity code.
2. Every chosen NIC must be the one that gives GPUDirect RDMA *without* crossing a root complex — i.e. `PIX` (or `PXB`), **never** `PHB`/`NODE`/`SYS`. For each GPU, name the NIC you would *not* use and its bad code (e.g. "`mlx5_4` = `SYS`, cross-socket").
3. One sentence stating where NVLink stops and the switched fabric begins on this node's `GPU → NIC → leaf → spine` path — and one sentence stating whether that answer would change on a GB200 NVL72 rack, and why.
4. Count the NICs on your reference node and state whether every NIC is a GPU rail NIC, or whether one is held off the compute rail for storage/management (per the 2-8-9-800 pattern) — name which.

Save the table; it is the intra-node half of the network-architecture read (09.2 adds the inter-node half).

## Common pitfalls

- **Assuming `topo -m`'s code ranking generalizes across every vendor and generation.** `PIX`/`PXB`/`PHB`/`NODE`/`SYS` describe *this* SKU's PCIe topology. A different CPU vendor, a different HGX generation, or an NVLink-connected rack like NVL72 can produce a different pattern or make the ranking moot entirely inside the NVLink domain. Re-read the matrix per box; don't carry last year's SKU's answer forward.
- **Conflating "rail" with "NUMA node."** A rail is a fabric-wide, cross-node concept — GPU-N's NIC on every node, landing on the same leaf switch. A NUMA node is a single-server CPU/memory locality domain. They're strongly correlated (rails 0–3 usually sit on NUMA node 0, rails 4–7 on NUMA node 1, matching the worked example above) but they are not the same thing, and treating them as interchangeable will trip you up the moment a box's NIC-to-NUMA layout doesn't match its rail numbering 1:1.
- **Forgetting the GB200 NVL72 exception.** "NVLink never leaves the chassis" is the right default and the wrong absolute. On NVL72, the chassis *is* the rack, and NVLink covers all 72 GPUs as one non-blocking domain. Stating the rule without this caveat in front of someone running current Blackwell hardware reads as out of date.
- **Treating the ninth/management NIC as part of the GPU rail.** In the 2-8-9-800 reference build, only eight of the node's nine NICs carry GPU-indexed collective traffic. Lumping the storage/management NIC into a bandwidth or capex calculation for the compute rail overstates what the collective actually has available and misstates the build's cost structure.

## Self-check

**(a) Why does cross-rail GPU-to-GPU traffic prefer NVLink over going out the NIC?**
**Answer:** NVLink is on-board, abundant, and low-latency — ~900 GB/s aggregate bidirectional per H100 (NVLink 4) or ~1.8 TB/s per GB200/GB300 GPU (NVLink 5) vs a single 400/800 Gb/s NIC (50–100 GB/s) plus switch hops. Shuffling a tensor sideways over NVLink to align each GPU with its own rail is 9–18× the bandwidth and a fraction of the latency of pushing it across the fabric and back. It also keeps the NIC carrying only rail-local (GPU-N ↔ GPU-N) traffic, which is what lets the spine be oversubscribed safely.

**(b) On a topo matrix, which GPU/NIC pairs show `PXB`/`SYS`, and why does that hurt GPUDirect RDMA?**
**Answer:** `PXB` = the GPU→NIC path traverses multiple PCIe bridges (a switch cascade) — RDMA still works but with extra hops/latency. `SYS` = the path crosses the inter-socket link (UPI/QPI) to the other CPU/NUMA node — the NIC can no longer DMA the GPU's HBM directly, so the transfer falls back through pinned host memory and bandwidth collapses to inter-socket limits. GPUDirect wants same-PCIe-switch (`PIX`); anything that drags the CPU root or the second socket into the path defeats the zero-copy premise.

**(c) Where does NVLink stop and the switched fabric begin in the `GPU → NIC → leaf → spine` path?**
**Answer:** NVLink/NVSwitch is bounded by the NVLink domain — the board/backplane inside one chassis on 8-GPU HGX hardware. It never runs on a server-to-server cable there. Traffic to a GPU outside that domain exits GPU→PCIe→**NIC**, and the switched fabric (InfiniBand or RoCE Ethernet) begins at the **cable from NIC to leaf/ToR switch**. The NIC is the handoff point; everything left of it is 02b, everything right of it is the fabric. (On GB200/GB300 NVL72 the domain is a whole rack, so the handoff point moves to the rack's edge instead.)

**(d) Does "NVLink never leaves the chassis" hold on GB200 NVL72? What exactly changes?**
**Answer:** No — it's the one real exception. GB200 NVL72 uses a 5th-generation NVLink Switch fabric to bind 72 GPUs (and 36 Grace CPUs) into a single non-blocking NVLink domain spanning the whole rack, delivering 130 TB/s of aggregate GPU-to-GPU bandwidth with no PCIe/NIC hop *inside* that domain. What changes is the scope of "chassis": on an 8-GPU HGX box it means one board; on NVL72 it means one rack. NVIDIA's multi-rack NVLink Switch System extends this further to up to 576 GPUs in one non-blocking NVLink compute fabric across racks — still distinct from the inter-node InfiniBand/Ethernet fabric this lesson otherwise describes.

**(e) Why does NVIDIA's HGX AI Factory reference architecture spec nine NICs for an 8-GPU node, not eight?**
**Answer:** Eight NICs are the GPU-indexed rail NICs, each dedicated to one GPU's GPUDirect RDMA collective traffic (the 2-8-9-800 build: 2 CPUs, 8 GPUs, 9 NICs, 800 Gb/s). The ninth is deliberately kept **off** the compute rail and dedicated to storage and cluster-management traffic, so checkpointing, dataset staging, and control-plane chatter never contend with a GPU's dedicated collective bandwidth. It's isolation spend, not a spare — count it separately from the eight rail NICs in any bandwidth or capex calculation.

## Connections & what's next

This lesson is the hinge between 02b (everything inside one box) and the rest of module 09 (everything above it). It also reaches sideways: 08's ring/tree all-reduce is *why* rail-locality matters (it's the traffic pattern that makes GPU-N-to-GPU-N the hot path), and 06's topology-aware gang placement is *how* a scheduler keeps a job's GPUs rail-aligned in the first place. The next lesson, **09.2 (the inter-node fabric)**, takes the rail you just learned to define and asks what happens once traffic climbs off it: it builds the Clos/fat-tree tiers above the leaf, defines full bisection bandwidth and oversubscription, and shows — using Meta's real 24,576-GPU Llama-3 cluster, plus real deviations from that design at Alibaba and in 100K-GPU builds — why a fabric can be heavily oversubscribed at the spine and lose almost nothing, *because* of the rail-locality this lesson established. Carry the mapping `GPU index ↔ rail ↔ leaf switch ↔ dedicated NIC` forward; 09.2 is the tier diagram built on top of it.

## References & further reading

**Primary sources**
- NVIDIA, [GB200 NVL72 product page](https://www.nvidia.com/en-us/data-center/gb200-nvl72/) — read for: the 130 TB/s / 72-GPU single-NVLink-domain numbers that ground the NVL72 exception.
- NVIDIA Technical Blog, [GB200 NVL72 Delivers Trillion-Parameter LLM Training and Real-Time Inference](https://developer.nvidia.com/blog/nvidia-gb200-nvl72-delivers-trillion-parameter-llm-training-and-real-time-inference/) — read for: the 5th-gen NVLink Switch spec (144 ports, 14.4 TB/s) and the up-to-576-GPU multi-rack NVLink fabric claim.
- NVIDIA, [HGX AI Factory Reference Architecture — Network Logical Architecture](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/network-logical-architecture.html) — read for: the 2-8-9-800 reference build and its 9-NICs-for-8-GPUs rationale.
- NVIDIA, [ConnectX-8 SuperNIC user manual / introduction](https://docs.nvidia.com/networking/display/connectx8SuperNIC/Introduction) — read for: confirmed ConnectX-8 spec — 800 Gb/s, PCIe Gen6 x16, single IB XDR port or dual 400G Ethernet.

**Real-world engineering blogs**
- CoreWeave, [First cloud provider to announce GA of NVIDIA GB200 NVL72 instances](https://www.coreweave.com/news/coreweave-first-cloud-provider-to-announce-general-availability-of-nvidia-gb200-nvl72-instances) — what it shows: a real rail-optimized Quantum-2 InfiniBand deployment at 400 Gb/s/GPU alongside rack-scale NVLink, at clusters up to 110,000 GPUs.
- ServeTheHome, [Microsoft Azure Eagle is a Paradigm-Shifting Cloud Supercomputer](https://www.servethehome.com/microsoft-azure-eagle-is-a-paradigm-shifting-cloud-supercomputer-nvidia-intel/), cross-checked against the [TOP500 Eagle system record](https://top500.org/system/180236/) — what it shows: 14,400 H100s on Quantum-2 CX7 InfiniBand, real GPU-to-rail cabling at #3-TOP500 scale.
- ByteDance, via USENIX NSDI '24, [MegaScale: Scaling LLM Training to More Than 10,000 GPUs](https://www.usenix.org/conference/nsdi24/presentation/jiang-ziheng) — what it shows: a non-Meta hyperscaler independently validating the same rail-aware GPU-to-NIC design at production scale (55.2% MFU at 12,288 GPUs).

**Deeper dives**
- Glenn K. Lockwood, ["NVLink" — practitioner notes](https://www.glennklockwood.com/garden/nvlink) — an independent, non-vendor breakdown of NVLink generations and their real-world limits; a useful counterweight to reading only vendor marketing numbers.
- Jonathan Hui, ["Nvidia Blackwell GB200 NVL72 & Networking"](https://jonathan-hui.medium.com/nvidia-blackwell-gb200-nvl72-networking-e36aade6ced9) — a detailed independent walkthrough of the NVL72 rack's physical NVLink/NVSwitch wiring, useful for building real intuition beyond the datasheet numbers.
