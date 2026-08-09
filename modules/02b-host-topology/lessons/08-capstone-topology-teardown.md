---
lesson: "02b.8"
title: "Capstone — read one real GPU node end to end"
module: "02b"
concept: "Capstone — topology teardown"
status: not-started
est_time: "7h"
artifacts: []
---

# 02b.8 · Capstone — read one real GPU node end to end

> **Concept.** Reconcile lstopo + lspci -tv + nvidia-smi topo -m + numactl --hardware into ONE coherent topology diagram for a real GPU node, predict its throughput failure modes, and measure the cost of a NUMA misalignment.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Why this matters

This is the whiteboard task. "Draw me a DGX/HGX node and tell me where the bottleneck is" is asked, in some form, in nearly every senior platform interview at a GPU-heavy shop — CoreWeave, NVIDIA, a neocloud. Not because they want a memorized diagram, but because reconciling four tools that each show a *different projection* of the same silicon, and then reading off the consequences, is exactly the on-the-job diagnosis when a training job runs at half its spec and the dashboards are green.

The failure it prevents is expensive and invisible. A GPU pinned to a NIC across the socket link (`SYS` in the matrix) does GPUDirect RDMA through UPI/QPI instead of a local PCIe switch; a x16 link that trained at x4 (`LnkSta` ≠ `LnkCap`) silently caps host-to-device bandwidth; an NVMe on the wrong socket adds a UPI hop to every dataloader read. Each is a real percentage off a $2–4/hr/GPU rental, multiplied by every hour the job runs. None of them throws an error. The capstone deliverable — the reconciled diagram plus a measured aligned-vs-misaligned delta plus the exporter you'd build — is the portfolio artifact that says you can find this class of waste. That is the cost/observability differentiator this whole track is built around.

## What's new for you

After lessons 1–7 you can read each tool on its own: you know what a NUMA node is, what `LnkCap` vs `LnkSta` means, how an 8-GPU HGX baseboard is wired, how Topology Manager aligns allocations, where NVMe sits. The tools are not new.

What is new is that **no single tool is authoritative for the whole node** — each has a blind spot the others fill, and the skill being tested is *cross-checking* them into one picture and then *predicting* what that picture will do under load. `numactl --hardware` knows NUMA distances but nothing about PCIe. `lstopo` has the locality tree but shows *capability*, not the *trained* link state or the live NVLink mesh. `lspci -vvv` has the trained link but no NVLink and no GPU identity you'd recognize. `nvidia-smi topo -m` has the GPU↔GPU↔NIC link classes but flattens the PCIe detail into one letter code. You stop asking "what does this command output mean" and start asking "these four disagree about GPU3's home node — which one is right, and what breaks because of it." That reconciliation, done fast and out loud, is the deliverable and the interview.

## The four tools, reconciled

Each tool is a projection. Learn what it uniquely shows and where it goes blind.

### `numactl --hardware`

Shows the **NUMA topology from the CPU/memory side**: how many NUMA nodes, which logical CPUs belong to each, how much memory per node, and the **distance matrix** (`node distances`, relative latencies — `10` local, `21`/`32` remote across sockets).

```
available: 2 nodes (0-1)
node 0 cpus: 0-31 64-95
node 0 size: 515655 MB
node 1 cpus: 32-63 96-127
node 1 size: 516086 MB
node distances:
node   0    1
  0:  10   21
  1:  21   10
```

- **Reads off:** how many sockets/NUMA domains, CPU and RAM budget per domain, the cost multiplier of a remote access (21/10 ≈ 2.1× latency here).
- **Blind spot:** knows *nothing* about PCIe. It will never tell you which NUMA node a GPU or NIC hangs off. It is the frame you hang the devices on, not the device map.

### `lstopo` / hwloc — the authoritative I/O-locality tree

`lstopo` (from hwloc; `lstopo-no-graphics -.txt` or `--of console` for text on a headless box) is the **one tool that unifies compute and I/O locality** into a single tree: `Machine → Package (socket) → NUMANode → L3/L2/cores` on the compute side, and critically `→ PCI bridges → the actual devices` (`CoProc/CUDA` GPUs, `Net`/`OpenFabrics` NICs, `Block`/NVMe) hanging off the correct NUMA node.

```
Machine
  Package L#0
    NUMANode L#0 (P#0 504GB)
    HostBridge
      PCIBridge
        PCI 17:00.0 (3D)  CoProc "cuda0"  GPU0
      PCIBridge
        PCI 2a:00.0 (Ethernet)  Net "mlx5_0"  NIC0
  Package L#1
    NUMANode L#1 (P#1 504GB)
    HostBridge
      PCIBridge
        PCI 98:00.0 (3D)  CoProc "cuda4"  GPU4
```

- **Reads off:** the *home NUMA node of every GPU, NIC, and NVMe* — the spine of your reconciled diagram. This is the ground truth for "which socket owns this device."
- **Blind spot:** shows link **capability / negotiated width as configured**, and does **not** reliably show the *live NVLink mesh* between GPUs (NVLink is above PCIe; it appears, if at all, as separate coproc links, not the clean matrix `nvidia-smi` gives). Treat its numbers as topology, not as the trained per-link state under load — cross-check width against `lspci -vvv`.

### `lspci -tv` + `lspci -vvv` — the PCIe tree and the trained link

`lspci -tv` draws the **raw PCIe hierarchy** as an ASCII tree (root complex → bridges/switches → endpoints) with vendor strings. `lspci -vvv -s <BDF>` on a device dumps the capabilities, including the two lines that matter most in this whole module:

```
-[0000:16]-+-00.0-[17]----00.0  NVIDIA ... (GPU0)
           \-01.0-[2a]----00.0  Mellanox ... (mlx5_0)

# lspci -vvv -s 17:00.0
        LnkCap: Port #0, Speed 16GT/s, Width x16, ...
        LnkSta: Speed 16GT/s (ok), Width x16 (ok)      ← trained == capable: healthy
        LnkSta: Speed 2.5GT/s (downgraded), Width x4 (downgraded)  ← DEGRADED
```

- **Reads off:** the switch layout (which GPU and NIC share a single PCIe switch = candidates for a `PIX` GPUDirect path), and — uniquely — the **trained link**: `LnkSta` vs `LnkCap`. A x16 Gen4 slot running x4 or 2.5GT/s is a ~4× host-bandwidth cap with no error anywhere else. `(downgraded)` is the word you grep for.
- **Blind spot:** no NVLink (it is not a PCIe link), no NUMA affinity in a form you'd want to read by hand (it's buried in sysfs), and GPUs show as opaque vendor strings, not `cuda0`. It tells you the *pipe width*, not the *GPU-to-GPU fabric*.

### `nvidia-smi topo -m` — the GPU↔GPU↔NIC link matrix

The accelerator-side view: a matrix where every GPU-to-GPU and GPU-to-NIC cell carries a **link-class code**, plus per-GPU `NUMA Affinity` and `CPU Affinity` columns.

```
      GPU0  GPU1  GPU2  GPU3  mlx5_0  mlx5_3   CPU Affinity  NUMA Affinity
GPU0   X    NV18  NV18  NV18   PIX     SYS      0-31,64-95    0
GPU3  NV18  NV18  NV18   X     SYS     PIX      32-63,96-127  1
```

Legend, best → worst, with the meaning that matters for **GPUDirect RDMA (GPU↔NIC) and NCCL (GPU↔GPU)**:

| Code | Meaning | Verdict |
|------|---------|---------|
| `X` | self | — |
| `NV#` | bonded set of `#` NVLinks | **best** GPU↔GPU (the NVLink mesh) |
| `PIX` | traverses at most a **single PCIe bridge** | **best PCIe** — ideal GPU↔NIC for GPUDirect RDMA |
| `PXB` | traverses **multiple PCIe bridges**, not the host bridge | good |
| `PHB` | traverses a **PCIe Host Bridge** (the CPU/root complex) | ok-ish |
| `NODE` | same NUMA node, **different root complex** | poor |
| `SYS` | across NUMA nodes / **sockets** (UPI/QPI) | **worst** — crossing the socket |

The rule that gets tested: **`PIX`/`NV#` = local and fast; `NODE`/`SYS` = you crossed the socket**. A GPU–NIC pair marked `SYS` will do GPUDirect RDMA over the inter-socket link, halving effective RDMA bandwidth and adding latency. You want each GPU paired to a NIC that reads `PIX` (or at worst `PXB`).

- **Reads off:** the NVLink mesh (`NV#`), the correct NIC for each GPU (the `PIX` cell in its row), and each GPU's `NUMA Affinity` in one column.
- **Blind spot:** flattens PCIe into a single letter — it will not tell you a `PIX` link *trained at x4*. `PIX` says "one bridge away," not "x16 and healthy." You still need `lspci -vvv` for the trained width. It also won't show NVMe.

### Cross-checking the four into one diagram

Reconcile in this order; each step uses the tool that owns that fact:

1. **Frame — sockets & NUMA:** `numactl --hardware` → N NUMA nodes, CPU ranges, RAM per node, remote-distance multiplier.
2. **Hang the devices — home NUMA node of every GPU/NIC/NVMe:** `lstopo` tree → this is the spine. Confirm each GPU's home node against the `NUMA Affinity` column of `nvidia-smi topo -m` (they must agree; if they don't, trust `lstopo` + sysfs and flag it).
3. **Pair GPU↔NIC and classify:** `nvidia-smi topo -m` → for each GPU, the NIC whose cell is `PIX` is its GDR partner; note any pair that is only `SYS`/`NODE`.
4. **Verify the pipes:** `lspci -tv` to see which GPU/NIC share a switch (explains the `PIX`), then `lspci -vvv -s <BDF>` on each GPU and NIC → `LnkSta` vs `LnkCap`; flag every `(downgraded)`.
5. **Place storage:** `lstopo` for each NVMe's home node → flag any NVMe whose consumers (dataloader CPUs / GPU) live on the other socket = cross-socket read path.

Output = one diagram: two socket columns, each with its NUMA node, its GPUs, its NICs (with the GPU↔NIC link class), its NVMe, and a red mark on every degraded link, every `SYS` GPU–NIC pair, and every cross-socket NVMe.

Two reconciliation traps to name out loud in the interview: (1) the tools **disagree on identity** — `lspci` calls a GPU `17:00.0`, `lstopo` calls it `cuda0`, `nvidia-smi` calls it `GPU0`; you must map BDF→cudaN via `nvidia-smi -q | grep 'Bus Id'` (or `lstopo`, which prints both) before you can say "GPU2 trained at x8" and mean the same silicon. (2) `PIX` in the matrix and `x16 (ok)` in `LnkSta` are **orthogonal** — `PIX` is *how many bridges* the path crosses, `LnkSta` is *how wide the pipe trained*. A link can be `PIX` and x4-degraded, or x16-healthy and `SYS`. You need both codes for the same edge before you trust it.

## Worked example

One node, 2 sockets, 8 GPUs (HGX-style), 8 NICs, 4 NVMe. Snippets, then the reconciled picture, then the failure modes.

**`numactl --hardware`** → `2 nodes (0-1)`, node0 cpus `0-31,64-95`, node1 cpus `32-63,96-127`, ~504GB each, `distances 10/21`. Frame: two sockets, remote access ~2.1× latency.

**`lstopo` (abridged)** → under Package0/NUMANode0: `cuda0..cuda3`, `mlx5_0..mlx5_3`, `nvme0`, `nvme1`. Under Package1/NUMANode1: `cuda4..cuda7`, `mlx5_4..mlx5_7`, `nvme2`, `nvme3`. So home nodes: **GPU0–3 → node0, GPU4–7 → node1**.

**`nvidia-smi topo -m` (corner)**:

```
       GPU0 GPU1 GPU2 GPU3 GPU4 ... mlx5_0 mlx5_3 mlx5_4   NUMA Aff
GPU0    X   NV18 NV18 NV18 NV18      PIX    NODE   SYS       0
GPU3   NV18 NV18 NV18  X   NV18      NODE   PIX    SYS       0
GPU4   NV18 ...             X        SYS    SYS    PIX       1
```

Read: all GPUs are `NV18` to each other (full NVLink mesh — NCCL intra-node is fine). GPU0's `PIX` NIC is `mlx5_0`; GPU3's `PIX` NIC is `mlx5_3` — both on node0, correct. GPU4↔`mlx5_4` is `PIX` on node1. Note GPU0↔`mlx5_4` is `SYS` (crosses the socket) — do **not** pair them.

**`lspci -vvv`** on each GPU/NIC. Seven read `LnkSta: Speed 16GT/s (ok), Width x16 (ok)`. One does not:

```
# GPU2  (17:00.0-ish)
LnkCap: Speed 16GT/s, Width x16
LnkSta: Speed 16GT/s (ok), Width x8 (downgraded)   ← trained at half width
```

GPU2 negotiated x8 on a x16 Gen4 slot — ~½ its host↔device PCIe bandwidth. `nvidia-smi topo -m` still shows GPU2's NIC as `PIX`; only `lspci -vvv` caught this.

**`lstopo` for NVMe** → `nvme0/1` on node0, `nvme2/3` on node1. If the dataloader for a job on GPU0–3 is pinned to node0 CPUs and reads `nvme0`, path is local. But the job's checkpoint writer was configured to `nvme2` (node1) → every checkpoint write crosses UPI.

**Reconciled diagram (text):**

```
 SOCKET 0 / NUMANode0 (cpu 0-31,64-95, 504GB)      SOCKET 1 / NUMANode1 (cpu 32-63,96-127)
   GPU0 --PIX--> mlx5_0   [x16 ok]                    GPU4 --PIX--> mlx5_4   [x16 ok]
   GPU1 --PIX--> mlx5_1   [x16 ok]                    GPU5 --PIX--> mlx5_5   [x16 ok]
   GPU2 --PIX--> mlx5_2   [x8 DOWNGRADED ⚠]           GPU6 --PIX--> mlx5_6   [x16 ok]
   GPU3 --PIX--> mlx5_3   [x16 ok]                    GPU7 --PIX--> mlx5_7   [x16 ok]
   nvme0, nvme1                                       nvme2, nvme3
   [GPU0..7 fully NVLink-meshed: NV18 all pairs]
   ⚠ checkpoint writer on GPU0-3 job points at nvme2 (SOCKET1) → cross-socket writes
   ⚠ GPU0↔mlx5_4 is SYS — never use as a GDR pair
```

**Failure modes read off the picture:**
- **GPU2 host bandwidth is ~half.** `LnkSta x8 (downgraded)`. Cause: reseat/slot/BIOS bifurcation. Confirm: `lspci -vvv -s <gpu2 bdf> | grep LnkSta`. Nothing else flags it.
- **A GPU0–3 job told to RDMA out `mlx5_4` gets SYS.** GPUDirect RDMA over UPI ≈ half bandwidth + latency. Fix: pin the job's HCA to its `PIX` NIC (`mlx5_0..3`), or let Topology Manager `single-numa-node` do it.
- **Checkpoint writes cross the socket** (nvme2 from a node0 job). Fix: point the writer at `nvme0/1`, or pin the job entirely to node1.

## Practice

**This is the module deliverable.** See [`../practice/topology-teardown/README.md`](../practice/topology-teardown/README.md). Acceptance is the [module checkpoint](../checkpoint.md) — you are done when you can defend every reconciliation from memory.

**Part A — reconcile a real node.** Rent one real GPU node (any neocloud: an 8×A100/H100 HGX box, or even a 1–2 GPU box if that's the budget). Run all four tools and capture raw output into `practice/topology-teardown/`:

```bash
numactl --hardware                    > raw/numactl.txt
lstopo-no-graphics --of console       > raw/lstopo.txt   # + --of svg for the picture
lspci -tv                             > raw/lspci-tv.txt
for bdf in $(lspci | grep -iE '3D|NVIDIA|Mellanox|Ethernet' | cut -d' ' -f1); do
  echo "== $bdf =="; lspci -vvv -s $bdf | grep -E 'LnkCap|LnkSta'; done > raw/lnk.txt
nvidia-smi topo -m                    > raw/nvidia-topo.txt
```

Produce **one reconciled topology diagram** (SVG from `lstopo --of svg`, annotated, or a hand-drawn/mermaid version) showing per-GPU home NUMA node, GPU↔NIC pairing + link class, NVMe placement, and any degraded link or `SYS` pair marked in red. Commit the raw captures alongside it as evidence.

**Part B — measure the cost of misalignment.** Pick a bandwidth-sensitive workload: `nccl-tests` (`all_reduce_perf`), a GPU↔NIC RDMA bandwidth test, or a real training step. Run it **aligned** (job + memory on the GPU's home node) and **misaligned** (force remote memory with `numactl --membind` / `--cpunodebind` to the *other* socket, or pin to the `SYS` NIC):

```bash
numactl --cpunodebind=0 --membind=0 ./run.sh   # aligned
numactl --cpunodebind=0 --membind=1 ./run.sh   # remote memory — misaligned
```

Record the throughput delta (expect a meaningful hit — often 10–40%+ on bandwidth-bound steps; NVLink-bound all-reduce may hide it, host-staged paths won't). Then write **one page**: (1) the aligned-vs-misaligned numbers; (2) **which production metric would and would not have caught this** — GPU util (would NOT, stays ~100%), SM occupancy (would not), DCGM PCIe/NVLink bytes counters and per-NUMA memory-bandwidth (would); (3) the **exporter/alert you'd build** — e.g. a small collector that on node start emits `node_gpu_nic_link_class{gpu,nic}` and `node_gpu_pcie_link_downgraded{gpu}` from `nvidia-smi topo -m` + `lspci`, alerting on any `SYS` GDR pair or any `LnkSta != LnkCap`.

## Self-check

**(a) Given this matrix, which NIC do you use for GPUDirect RDMA to GPU3, and why?**

```
       mlx5_0  mlx5_2  mlx5_3   NUMA Aff
GPU3    NODE    SYS     PIX       1
```

**Answer:** `mlx5_3`. Its cell is `PIX` — GPU3→NIC traverses at most a single PCIe bridge, the shortest GDR path, and both sit on GPU3's home NUMA node 1. `mlx5_0` is `NODE` (same NUMA node but a different root complex — extra host-bridge hop) and `mlx5_2` is `SYS` (crosses the socket over UPI — worst case, ~half RDMA bandwidth + latency). GDR wants the traffic to never touch the inter-socket link, so the `PIX` NIC wins; `PXB` would be the acceptable fallback, never `NODE`/`SYS`.

**(b) Name one thing each of the four tools does NOT show.**

**Answer:** `numactl --hardware` — no PCIe/device info at all (can't place a GPU or NIC). `lstopo` — not the *live NVLink mesh* and not the *trained-under-load* link state (shows configured capability/topology, not `LnkSta`). `lspci -vvv` — no NVLink (it isn't a PCIe link) and no friendly NUMA affinity / `cudaN` identity. `nvidia-smi topo -m` — flattens PCIe to one letter (a `PIX` link can still be trained at x4 and it won't say so), and shows no NVMe.

**(c) "GPU at 100% utilization, throughput ~half spec, no error." Host-side causes + the one command that confirms/eliminates each.**

**Answer:** GPU util only means the SMs had work queued, not that data arrived fast. Host-side suspects:
- **PCIe link trained low** (x16→x4/x8, or Gen4→Gen3): `lspci -vvv -s <bdf> | grep -E 'LnkCap|LnkSta'` → look for `(downgraded)`.
- **Cross-socket GPU↔NIC path** (GDR over UPI): `nvidia-smi topo -m` → the used NIC's cell is `SYS`/`NODE` instead of `PIX`.
- **Remote memory / wrong NUMA binding** (host↔device staging over UPI): `numactl --hardware` for the layout + `numastat -p <pid>` (or check the job's `--membind`) → memory on the other node.
- **Cross-socket NVMe** feeding the dataloader: `lstopo` → NVMe's home node vs the job's CPU node.
- **NVLink degraded / not all links up** (peer-to-peer capped): `nvidia-smi nvlink -s` → any link `inactive` or below rated GT/s.
- **Thermal / power throttle** (clocks dropped under load — a lesson-7 cause that mimics a topology bottleneck): `nvidia-smi -q -d PERFORMANCE,CLOCK` → look for `SW Thermal Slowdown` / `HW Power Brake` active and clocks below base.

Util stays 100% through all of them, which is exactly why you reconcile the topology instead of trusting the utilization dashboard. The discipline: one command per hypothesis, eliminate top-down, never guess.

## Resources

- **[lstopo / hwloc manpage](https://manpages.ubuntu.com/manpages/focal/man1/lstopo.1.html)** — the authoritative locality-tree tool. *Deep-read* the output-format flags (`--of console/svg`, `-.txt`) and how it labels NUMANode/PCIBridge/CoProc/Net/Block; this is the tool that unifies your diagram. *Why:* it's the spine of the reconciliation — the one ground truth for each device's home NUMA node.
- **[nvidia-smi topo -m legend reference (Sigma2)](https://documentation.sigma2.no/code_development/guides/gpu_usage.html)** — clear worked explanation of the `X/NV#/PIX/PXB/PHB/NODE/SYS` codes with a real matrix. *Skim* to lock in the best→worst ordering and the `PIX`-vs-`SYS` GDR rule. *Why:* this legend is the single most interview-quotable table in the module.
- **[NVIDIA GPUDirect RDMA docs](https://docs.nvidia.com/cuda/gpudirect-rdma/)** — the mechanism behind *why* `SYS` is expensive: RDMA straight between NIC and GPU memory over PCIe, and why crossing the root complex / socket kills it. *Deep-read* the "supported systems" / PCIe topology section. *Why:* it justifies the GPU↔NIC pairing you draw and the alert you'd build in Part B.
