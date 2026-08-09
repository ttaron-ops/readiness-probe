---
lesson: "09.1"
title: "From intra-node to inter-node: extending the topology matrix past the NIC"
module: "09"
concept: "From intra-node to inter-node: extending the topology matrix past the NIC"
status: not-started
est_time: "5h"
artifacts: []
---

# 09.1 · From intra-node to inter-node: extending the topology matrix past the NIC

> **Concept.** `nvidia-smi topo -m` stops at the PCIe edge of one box; the fabric picks up at the NIC — learn to read the full `GPU → NIC → leaf → spine` path, name the *rail* each GPU rides, and turn 02b's "same root complex" rule into "same rail" across nodes.
>
> Module: [🔗 09 — Networking and topology](../README.md) · Deliverable: [Network architecture read](../practice/network-architecture-read/README.md)

## Why this matters

Every placement and procurement argument you make above eight GPUs lives or dies on one question the topology matrix cannot answer alone: *when a collective leaves the box, does it stay on its rail or does it get funneled the wrong way?* Get the GPU-to-NIC mapping wrong and GPUDirect RDMA silently falls back to a bounce through host memory — you keep the GPUs, lose the wire, and eat a step-time regression no dashboard names. In an interview for the platform track at a GPU neocloud, being able to draw the `GPU → NIC → leaf → spine` path and say exactly where NVLink stops is the line between "I run Kubernetes near GPUs" and "I can argue fabric with the network team."

## What's new here

**02b** gave you everything *inside the chassis*: the PCIe tree, NVLink/NVSwitch, NVLink domains, rail alignment as a board-level property, the GPUDirect "same-root-complex" rule, and how to read `nvidia-smi topo -m` (the `NV#`/`PIX`/`PXB`/`PHB`/`NODE`/`SYS` connectivity codes). **08** told you *which* collective runs (ring vs tree all-reduce) and that communication is the bottleneck. **06** placed the gang on rail-aligned GPUs. All of that ends at the NIC.

This lesson crosses that boundary. The matrix's rightmost columns are the `HCA`/`mlx5_*` NICs — 02b treated them as "the edge of the box." Here they become the *on-ramp to the fabric*. We extend the path one hop at a time — NIC → leaf (top-of-rack) switch → spine — establish precisely where the last NVLink hop ends and the first switched-Ethernet/InfiniBand hop begins, and promote the intra-node alignment rule (same root complex) into its inter-node twin (**same rail**). Everything past the NIC is new; everything up to it you already own.

## Core notes

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

1. **NVLink never leaves the chassis.** There is no NVLink cable between servers in a standard HGX/DGX rack — NVLink and NVSwitch are a *board-and-backplane* fabric bounded by the NVLink domain (02b). (NVLink *Switch System* / GB200 NVL72 extends the domain across a rack via an NVLink spine, but that is still a single NVLink domain, not the inter-node IB/Ethernet fabric — do not confuse the two.) The moment traffic must reach a GPU outside its NVLink domain, it exits through the **NIC** and onto the **switched fabric**. That transition — PCIe-to-NIC-to-cable — is the intra/inter boundary.
2. **The NIC is the handoff.** GPUDirect RDMA (02b) lets the NIC DMA straight out of GPU HBM with no CPU bounce — *provided* the GPU and NIC sit under the same PCIe switch / root complex. That is the whole game: keep the GPU→NIC hop a clean PCIe-switch (`PIX`/`PXB`-adjacent) path, and the fabric sees a zero-copy source.

### What a "rail" is

An 8-GPU HGX node has (typically) **one 400G-class NIC per GPU** — eight ConnectX-7 (400G NDR) or, on current builds, ConnectX-8 (800G XDR) HCAs, each paired to one GPU through the PCIe switch that GPU hangs off. A **rail** is the set of *same-indexed* GPU→NIC pairs across every node, cabled into the *same leaf switch*:

- GPU0 of every node → its local NIC → **leaf switch 0** = **rail 0**
- GPU1 of every node → its local NIC → **leaf switch 1** = **rail 1**
- … GPU7 → leaf 7 = rail 7.

So a rail is a *vertical slice* of the cluster: one GPU position, one NIC per node, one leaf switch, spanning all nodes. This is **rail-optimized** cabling, and it is the inter-node generalization of 02b's rail alignment. In 02b, "rail-aligned" meant a GPU and its NIC share a PCIe branch on the *board*. Here it means GPU-N's NIC on every node lands on the *same leaf*, so GPU-N-to-GPU-N traffic across nodes crosses exactly one switch and never touches the spine.

### The rule promotion: "same root complex" → "same rail"

02b's law for zero-copy inside the box:

> GPUDirect RDMA needs the GPU and NIC under the **same PCIe root complex / switch** — otherwise the DMA crosses the CPU (or worse, the inter-socket link), which the matrix flags as `NODE`/`SYS`, and the copy falls back through host memory.

Its inter-node twin:

> For a collective to stay cheap across nodes, GPU-N should talk to GPU-N — **same rail** — so its bytes ride its own leaf and never climb to the spine. Cross-*rail* traffic (GPU3 on node A wanting GPU5 on node B) has two options: (a) hop laterally over NVLink to the local GPU5 first, *then* go out rail 5's NIC — cheap, because NVLink is ~450 GB/s/direction of on-board bandwidth; or (b) go out rail 3 and traverse the spine to reach rail 5 — expensive, and the reason you keep collectives rail-local.

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

### Why cross-rail GPU-to-GPU prefers NVLink over the NIC

Concrete numbers. On an H100 HGX node, intra-node **NVLink (4th gen)** delivers ~900 GB/s aggregate bidirectional per GPU (~450 GB/s each way), all on-board, sub-microsecond. Its NIC is a single 400G (ConnectX-7) or 800G (ConnectX-8) port — **50 GB/s or 100 GB/s** to the wire, plus switch latency. So moving a tensor from GPU3 to GPU5 *within the node* over NVLink is ~9–18× the bandwidth and a fraction of the latency of pushing it out rail 3's NIC, across a leaf, maybe up a spine, and back down rail 5. The fabric is precious; NVLink is abundant. Rail-optimized software (NCCL with rail-aware topology detection) exploits exactly this: it shuffles cross-rail data *sideways over NVLink* to line every GPU up with its own rail, then does the inter-node leg rail-local. The NIC only ever carries GPU-N ↔ GPU-N.

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
- **GPU0 ↔ GPU4 = `NV8`.** Eight NVLink connections on-board. So GPU0-needs-GPU4 traffic stays on NVLink (900 GB/s), *not* out either NIC — exactly the cross-rail-prefers-NVLink rule.

**Mapping produced:** `GPU0 → rail 0 → mlx5_0 (PIX)`, `GPU4 → rail 4 → mlx5_4 (PIX)`. Pattern: GPU-N pairs with `mlx5_N` at `PIX`; any `SYS` cell is a cross-socket trap. Repeat for GPU0–7 and you have the node's rail map.

## Practice

Feeds the deliverable **Network architecture read**.

**Task.** Take a `nvidia-smi topo -m` matrix — from any multi-GPU box you can reach (`nvidia-smi topo -m`), or a captured 8-GPU HGX matrix if you have no cluster. For **each GPU**, produce the mapping:

`GPU-N → rail-N → NIC (mlx5_?) → connectivity code`

**Requirements / acceptance:**
1. A complete **GPU→rail→NIC table** for one node — all GPUs, each with its rail number, its chosen NIC, and the GPU→NIC connectivity code.
2. Every chosen NIC must be the one that gives GPUDirect RDMA *without* crossing a root complex — i.e. `PIX` (or `PXB`), **never** `PHB`/`NODE`/`SYS`. For each GPU, name the NIC you would *not* use and its bad code (e.g. "`mlx5_4` = `SYS`, cross-socket").
3. One sentence stating where NVLink stops and the switched fabric begins on this node's `GPU → NIC → leaf → spine` path.

Save the table; it is the intra-node half of the network-architecture read (09.2 adds the inter-node half).

## Self-check

**(a) Why does cross-rail GPU-to-GPU traffic prefer NVLink over going out the NIC?**
**Answer:** NVLink is on-board, abundant, and low-latency — ~900 GB/s aggregate bidirectional per H100 vs a single 400/800G NIC (50/100 GB/s) plus switch hops. Shuffling a tensor sideways over NVLink to align each GPU with its own rail is 9–18× the bandwidth and a fraction of the latency of pushing it across the fabric and back. It also keeps the NIC carrying only rail-local (GPU-N ↔ GPU-N) traffic, which is what lets the spine be oversubscribed safely.

**(b) On a topo matrix, which GPU/NIC pairs show `PXB`/`SYS`, and why does that hurt GPUDirect RDMA?**
**Answer:** `PXB` = the GPU→NIC path traverses multiple PCIe bridges (a switch cascade) — RDMA still works but with extra hops/latency. `SYS` = the path crosses the inter-socket link (UPI/QPI) to the other CPU/NUMA node — the NIC can no longer DMA the GPU's HBM directly, so the transfer falls back through pinned host memory and bandwidth collapses to inter-socket limits. GPUDirect wants same-PCIe-switch (`PIX`); anything that drags the CPU root or the second socket into the path defeats the zero-copy premise.

**(c) Where does NVLink stop and the switched fabric begin in the `GPU → NIC → leaf → spine` path?**
**Answer:** NVLink/NVSwitch is bounded by the NVLink domain — the board/backplane inside one chassis (or one NVL72 rack). It never runs on a server-to-server cable. Traffic to a GPU outside that domain exits GPU→PCIe→**NIC**, and the switched fabric (InfiniBand or RoCE Ethernet) begins at the **cable from NIC to leaf/ToR switch**. The NIC is the handoff point; everything left of it is 02b, everything right of it is the fabric.

## Resources

1. **Your 02b host-topology notes** (`../../02b-host-topology/lessons/`) — *reference, skim.* The PCIe tree, NVLink domain, rail alignment, GPUDirect same-root-complex rule, and the `topo -m` connectivity codes. This lesson stands on all of it; re-skim `05-topology-alignment-k8s.md` before the practice so the `PIX`/`SYS` codes are fresh.
2. **NVIDIA HGX AI Factory Reference Architecture** — https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/ — *deep.* The authoritative physical topology: how HGX nodes cable to leaf/ToR switches, the per-GPU NIC layout, and the rail-optimized wiring that makes GPU-N → leaf-N real. Read the network/fabric sections to see the `GPU → NIC → leaf → spine` path as a vendor draws it — this is the picture your deliverable reproduces.
