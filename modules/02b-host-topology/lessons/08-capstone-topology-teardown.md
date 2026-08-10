---
lesson: "02b.8"
title: "Capstone — read one real GPU node end to end"
module: "02b"
concept: "Capstone — topology teardown"
status: not-started
est_time: "9h"
prev: "07-power-and-thermals.md"
next: null
artifacts: []
sources: 10
---

# 02b.8 · Capstone — read one real GPU node end to end

> **Concept.** Reconcile lstopo + lspci -tv + nvidia-smi topo -m + numactl --hardware into ONE coherent topology diagram for a real GPU node, predict its throughput failure modes, and measure the cost of a NUMA misalignment.
>
> Module: [🧬 02b — Host hardware and topology](../README.md) · Deliverable: [Topology Teardown](../practice/topology-teardown/README.md)

## Where this fits

Every lesson in this module handed you one lens on the same machine: the NUMA tree (1–2), the PCIe fabric and its trained link state (3), the reference 8-GPU baseboard (4), Kubernetes' topology-aware placement (5), storage/GDS placement (6), and power/thermal throttling with its adjacent Xid/SXid fault layer (7). None of those lessons asked you to hold all of it in your head at once, on one real box, and decide what's actually true when two tools disagree. That's this lesson. It closes the module by forcing the synthesis: reconcile four tools that each show a different projection of the same silicon into one diagram, predict where a job bottlenecks, then prove it with a measurement.

## Why this matters

This is the whiteboard task. "Draw me a DGX/HGX node and tell me where the bottleneck is" is asked, in some form, in nearly every senior platform interview at a GPU-heavy shop — CoreWeave, NVIDIA, a neocloud. Not because they want a memorized diagram, but because reconciling four tools that each show a *different projection* of the same silicon, and then reading off the consequences, is exactly the on-the-job diagnosis when a training job runs at half its spec and the dashboards are green.

The failure it prevents is expensive and invisible. A GPU pinned to a NIC across the socket link (`SYS` in the matrix) does GPUDirect RDMA through UPI/QPI instead of a local PCIe switch; a x16 link that trained at x4 (`LnkSta` ≠ `LnkCap`) silently caps host-to-device bandwidth; an NVMe on the wrong socket adds a UPI hop to every dataloader read; a throttled clock quietly delivers 80% of spec. Each is a real percentage off a $2–4/hr/GPU rental, multiplied by every hour the job runs. None of them throws an error. The capstone deliverable — the reconciled diagram plus a measured aligned-vs-misaligned delta plus the exporter you'd build — is the portfolio artifact that says you can find this class of waste. That is the cost/observability differentiator this whole track is built around.

The stakes compound at fleet scale. Meta's own published data from training Llama 3 on a 16,384-GPU cluster reported **419 unexpected component failures over a 54-day run** — on the order of one every few hours — with GPU and HBM3 issues responsible for roughly half. At that failure cadence, manual, one-off reconciliation (exactly what this lesson teaches you to do by hand) cannot be the production answer; it has to become the *foundation* skill that automated fleet tooling is built on. Knowing both — how to do the reconciliation yourself, and that real shops have automated it — is the difference between "I did this once on a rented box" and "I understand this is what production fleet-health systems do continuously, and I can build a first version of one."

## What's new here (calibration)

- **You already know each tool on its own.** After lessons 1–7 you can read each one independently: what a NUMA node is, what `LnkCap` vs `LnkSta` means, how an 8-GPU HGX baseboard is wired, how Topology Manager aligns allocations, where NVMe sits, and how to read a throttle bitmask. None of the individual tools are new.
- **What's genuinely new is that no single tool is authoritative for the whole node** — each has a blind spot the others fill, and the skill being tested is *cross-checking* them into one picture and then *predicting* what that picture will do under load. You stop asking "what does this command output mean" and start asking "these four disagree about GPU3's home node — which one is right, and what breaks because of it."
- **Also new:** treating a hardware fault (an Xid/SXid error in `dmesg`) as a distinct seventh hypothesis alongside the six placement-based causes — a genuinely faulty component can produce a symptom identical to a topology bug and needs a different fix (a repair ticket, not `numactl`). And knowing that real production shops (Meta, CoreWeave, Crusoe, Modal) have automated this exact reconciliation continuously at fleet scale — this lesson's manual version is the skill those systems are built on, not a toy exercise unrelated to them.

## Core concepts

### The four tools, reconciled

Each tool is a projection. Learn what it uniquely shows and where it goes blind.

#### `numactl --hardware`

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

#### `lstopo` / hwloc — the authoritative I/O-locality tree

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

#### `lspci -tv` + `lspci -vvv` — the PCIe tree and the trained link

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

#### `nvidia-smi topo -m` — the GPU↔GPU↔NIC link matrix

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
- **Blind spot:** flattens PCIe into a single letter — it will not tell you a `PIX` link *trained at x4*. `PIX` says "one bridge away," not "x16 and healthy." You still need `lspci -vvv` for the trained width. It also won't show NVMe, and — as production evidence below shows — even NVIDIA's own tools can disagree with each other on the link class of the same edge.

### Cross-checking the four into one diagram

Reconcile in this order; each step uses the tool that owns that fact:

1. **Frame — sockets & NUMA:** `numactl --hardware` → N NUMA nodes, CPU ranges, RAM per node, remote-distance multiplier.
2. **Hang the devices — home NUMA node of every GPU/NIC/NVMe:** `lstopo` tree → this is the spine. Confirm each GPU's home node against the `NUMA Affinity` column of `nvidia-smi topo -m` (they must agree; if they don't, trust `lstopo` + sysfs and flag it).
3. **Pair GPU↔NIC and classify:** `nvidia-smi topo -m` → for each GPU, the NIC whose cell is `PIX` is its GDR partner; note any pair that is only `SYS`/`NODE`.
4. **Verify the pipes:** `lspci -tv` to see which GPU/NIC share a switch (explains the `PIX`), then `lspci -vvv -s <BDF>` on each GPU and NIC → `LnkSta` vs `LnkCap`; flag every `(downgraded)`.
5. **Place storage:** `lstopo` for each NVMe's home node → flag any NVMe whose consumers (dataloader CPUs / GPU) live on the other socket = cross-socket read path.
6. **Rule out a hardware fault:** `dmesg | grep -i xid` (and SXid on NVSwitch systems) → a topology-shaped symptom with a live Xid/SXid entry is a hardware fault, not a placement bug; route it to a repair ticket, not a re-pin.

Output = one diagram: two socket columns, each with its NUMA node, its GPUs, its NICs (with the GPU↔NIC link class), its NVMe, and a red mark on every degraded link, every `SYS` GPU–NIC pair, and every cross-socket NVMe.

Two reconciliation traps to name out loud in the interview, plus a third, production-proven one:

1. **The tools disagree on identity.** `lspci` calls a GPU `17:00.0`, `lstopo` calls it `cuda0`, `nvidia-smi` calls it `GPU0`. You must map BDF→cudaN via `nvidia-smi -q | grep 'Bus Id'` (or `lstopo`, which prints both) before you can say "GPU2 trained at x8" and mean the same silicon.
2. **`PIX` and `LnkSta` are orthogonal.** `PIX` is *how many bridges* the path crosses, `LnkSta` is *how wide the pipe trained*. A link can be `PIX` and x4-degraded, or x16-healthy and `SYS`. You need both codes for the same edge before you trust it.
3. **Even NVIDIA's own tools can disagree with each other on a real box.** A production incident report on an 8×V100 + Mellanox RDMA NIC system found NCCL's own topology-detection log (`NCCL_DEBUG=INFO`) reporting `PIX`/`PXB` for a GPU–NIC pair where `nvidia-smi topo -m` reported `PHB` — two "official" NVIDIA-adjacent tools, disagreeing about the same edge on the same silicon (see NVIDIA/nccl GitHub Issue #246, cited below). This is direct proof the module's "no single tool is authoritative" framing is not overcautious pedagogy — practitioners genuinely hit this, and the resolution is the same discipline this lesson teaches: reconcile against a third source (in that case, the PCIe tree itself) rather than trusting either tool blindly.

## Perspectives

**Developer.** From the researcher/ML-engineer seat, a topology defect and a hardware fault look identical — "the job is slow" or "the job crashed." The capstone's whole point is that the *platform engineer* is the one who must tell these apart and route to the right fix: rebalance placement, adjust a power cap, or file a hardware ticket. The developer never sees the four-tool reconciliation; they only see the symptom.

**Operator / SRE.** The capstone is literally the interview task — "draw me a DGX/HGX node and tell me where the bottleneck is" — rehearsed for real. But the *production* version of this skill is encoded in tools that run this reconciliation continuously and automatically at fleet scale, not manually per-incident (see Real-world use cases below). The manual, single-node version you're building is the foundation those systems are built on.

**Cross-tool reconciliation (this lesson's own defining angle).** The NCCL issue #246 case above is real proof that even NVIDIA's own tools can disagree on production hardware. That should raise your confidence in the capstone's central claim — "no single tool is authoritative for the whole node" is not overcautious, it's accurate, and the discipline of cross-checking against a third source when two tools disagree is a genuinely production-tested skill, not a classroom exercise.

**Economics / portfolio.** The deliverable's framing as "a strong blog post" is well-supported: CoreWeave, Meta, and Crusoe all publish exactly this kind of teardown/reliability content as public engineering blogs. Your capstone's output genuinely resembles a real, valued external-facing artifact at these companies — not merely a course exercise you did once.

## Real-world use cases

- **Meta Engineering — "How Meta keeps its AI hardware reliable"**: describes FBAR (Facebook Auto Remediation), an explicit failure-type taxonomy (disks, CPUs, memories, switches, GPUs, ASICs, networks), and a roughly 50× reduction in training-interruption rate from automating exactly this class of detection. The clearest anchor for "why fleet-scale reconciliation tooling exists at all." https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/
- **Meta Engineering — "How Facebook deals with PCIe faults to keep our data centers running reliably"**: describes Meta's automated PCIe fault detect→diagnose→remediate→repair pipeline and their open-source tool **PCIcrawler** (a Python CLI for PCI/PCIe topology + AER error inspection) — a direct production precedent for this capstone's Part A tooling. https://engineering.fb.com/2021/06/02/data-center-engineering/how-facebook-deals-with-pcie-faults-to-keep-our-data-centers-running-reliably/
- **Modal — "Keeping 20,000 GPUs healthy"**: documents a full Xid/SXid fault taxonomy and a `dcgmi`+`dmesg` passive health-check methodology run continuously at real fleet scale (20,000 GPUs) — a strong model answer for the capstone's Part B "what exporter/alert would you build" prompt. https://modal.com/blog/gpu-health
- **Crusoe — "AutoClusters: Automated GPU Failure Remediation for AI Training Clusters"**: describes automatic cordon/drain/replace remediation triggered on Xid-79-class failures, explicitly framed around minimizing queue-wait-time as "the largest controllable variable in cluster goodput" — a good production framing for the alert-design half of this capstone. https://www.crusoe.ai/resources/blog/autoclusters-minimizing-hardware-failures-in-large-gpu-clusters
- **CoreWeave — "NVIDIA H100 GPU Benchmark Results: What We Learned from Large-Scale GPU Testing"**: describes CoreWeave's SUNK (Slurm on Kubernetes) topology-aware scheduling with health probes that evict failing nodes before they impact a running job — ties this capstone's reconciliation skill directly to a named, production scheduling system, and reports 49.2% MFU on 128 H100s as the fleet-level metric this whole diagnostic chain protects (flag as a 2025-era snapshot). https://www.coreweave.com/blog/nvidia-h100-gpu-benchmark-results-what-we-learned-from-large-scale-gpu-testing

## Worked example

One node, 2 sockets, 8 GPUs (HGX-style), 8 NICs, 4 NVMe. Snippets, then the reconciled picture, then the failure modes, then a comparison of your proposed alert design against named production systems.

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

**`dmesg | grep -i xid`** → clean, no entries. This rules out a hardware fault as the explanation for GPU2's degraded link; it's a placement/seating issue, not a faulty part.

**Reconciled diagram (text):**

```
 SOCKET 0 / NUMANode0 (cpu 0-31,64-95, 504GB)      SOCKET 1 / NUMANode1 (cpu 32-63,96-127)
   GPU0 --PIX--> mlx5_0   [x16 ok]                    GPU4 --PIX--> mlx5_4   [x16 ok]
   GPU1 --PIX--> mlx5_1   [x16 ok]                    GPU5 --PIX--> mlx5_5   [x16 ok]
   GPU2 --PIX--> mlx5_2   [x8 DOWNGRADED ⚠]           GPU6 --PIX--> mlx5_6   [x16 ok]
   GPU3 --PIX--> mlx5_3   [x16 ok]                    GPU7 --PIX--> mlx5_7   [x16 ok]
   nvme0, nvme1                                       nvme2, nvme3
   [GPU0..7 fully NVLink-meshed: NV18 all pairs]
   [dmesg: no Xid/SXid entries — no hardware fault]
   ⚠ checkpoint writer on GPU0-3 job points at nvme2 (SOCKET1) → cross-socket writes
   ⚠ GPU0↔mlx5_4 is SYS — never use as a GDR pair
```

**Failure modes read off the picture:**
- **GPU2 host bandwidth is ~half.** `LnkSta x8 (downgraded)`. Cause: reseat/slot/BIOS bifurcation. Confirm: `lspci -vvv -s <gpu2 bdf> | grep LnkSta`. Nothing else flags it, and `dmesg` being clean rules out a hardware fault as the cause.
- **A GPU0–3 job told to RDMA out `mlx5_4` gets SYS.** GPUDirect RDMA over UPI ≈ half bandwidth + latency. Fix: pin the job's HCA to its `PIX` NIC (`mlx5_0..3`), or let Topology Manager `single-numa-node` do it.
- **Checkpoint writes cross the socket** (nvme2 from a node0 job). Fix: point the writer at `nvme0/1`, or pin the job entirely to node1.

**Naming your exporter against production systems.** Your Part B "exporter/alert you'd build" answer should state where it sits relative to real production tooling:

| System | What it detects | Detection vs. alerting vs. remediation |
|---|---|---|
| Meta PCIcrawler | PCIe topology + AER error counters | Detection |
| Modal-style DCGM+`dmesg` sweep | Xid/SXid faults, thermal violations | Detection + alerting |
| Crusoe AutoClusters | Xid-79-class failures | Detection + automated remediation (cordon/drain/replace) |
| CoreWeave SUNK health probes | Failing nodes pre-job-impact | Detection + scheduling-level remediation |
| Your proposed `node_gpu_nic_link_class` / `node_gpu_pcie_link_downgraded` exporter | GPU↔NIC link class, PCIe link degradation | Detection (feeds alerting; remediation is a stated next step, not built) |

Naming this table in an interview shows you know exactly which slice of the production problem your proposed design covers, and which slices (automated remediation, fleet-wide aggregation) are the next layer up.

## Practice

**This is the module deliverable.** See [`../practice/topology-teardown/README.md`](../practice/topology-teardown/README.md). Acceptance is the [module checkpoint](../checkpoint.md) — you are done when you can defend every reconciliation from memory.

**Part A — reconcile a real node.** Rent one real GPU node (any neocloud: an 8×A100/H100 HGX box, or even a 1–2 GPU box if that's the budget). Run all four tools plus a fault check, and capture raw output into `practice/topology-teardown/`:

```bash
numactl --hardware                    > raw/numactl.txt
lstopo-no-graphics --of console       > raw/lstopo.txt   # + --of svg for the picture
lspci -tv                             > raw/lspci-tv.txt
for bdf in $(lspci | grep -iE '3D|NVIDIA|Mellanox|Ethernet' | cut -d' ' -f1); do
  echo "== $bdf =="; lspci -vvv -s $bdf | grep -E 'LnkCap|LnkSta'; done > raw/lnk.txt
nvidia-smi topo -m                    > raw/nvidia-topo.txt
dmesg | grep -i xid                   > raw/xid-check.txt   # rule out hardware faults
```

Produce **one reconciled topology diagram** (SVG from `lstopo --of svg`, annotated, or a hand-drawn/mermaid version) showing per-GPU home NUMA node, GPU↔NIC pairing + link class, NVMe placement, and any degraded link, `SYS` pair, or Xid/SXid entry marked in red. Commit the raw captures alongside it as evidence.

**Part B — measure the cost of misalignment.** Pick a bandwidth-sensitive workload: `nccl-tests` (`all_reduce_perf`), a GPU↔NIC RDMA bandwidth test, or a real training step. Run it **aligned** (job + memory on the GPU's home node) and **misaligned** (force remote memory with `numactl --membind` / `--cpunodebind` to the *other* socket, or pin to the `SYS` NIC):

```bash
numactl --cpunodebind=0 --membind=0 ./run.sh   # aligned
numactl --cpunodebind=0 --membind=1 ./run.sh   # remote memory — misaligned
```

Record the throughput delta (expect a meaningful hit — often 10–40%+ on bandwidth-bound steps; NVLink-bound all-reduce may hide it, host-staged paths won't). Then write **one page**: (1) the aligned-vs-misaligned numbers; (2) **which production metric would and would not have caught this** — GPU util (would NOT, stays ~100%), SM occupancy (would not), DCGM PCIe/NVLink bytes counters and per-NUMA memory-bandwidth (would); (3) the **exporter/alert you'd build** — e.g. a small collector that on node start emits `node_gpu_nic_link_class{gpu,nic}` and `node_gpu_pcie_link_downgraded{gpu}` from `nvidia-smi topo -m` + `lspci`, alerting on any `SYS` GDR pair or any `LnkSta != LnkCap` — and place it on the named-tool comparison table above so you can defend, out loud, which slice of the production problem it covers.

## Common pitfalls

1. **Failing to map identity across tools before drawing conclusions.** `lspci`'s `17:00.0`, `lstopo`'s `cuda0`, and `nvidia-smi`'s `GPU0` must be confirmed as the same physical device (`nvidia-smi -q | grep 'Bus Id'`) before you say anything about "GPU2's link speed."
2. **Treating `PIX` in the matrix as proof the link is healthy.** `PIX` and `LnkSta` are orthogonal facts — one is bridge-count, the other is trained width/speed. Check both for every edge you care about.
3. **Assuming this is a purely per-node, manual exercise in production.** Real shops (Meta, CoreWeave, Crusoe, Modal) automate exactly this reconciliation continuously at fleet scale. The capstone's manual, single-node version is the *foundation* skill, not the production end-state — say so explicitly in an interview so you don't undersell the deliverable's real-world relevance.
4. **Treating a hardware fault and a topology misconfiguration as the same failure class.** A genuinely faulty component (an Xid/SXid error in `dmesg`) can produce a symptom identical to a placement bug — "GPU 100% util, throughput half spec, no obvious error" — but needs a different fix: a repair ticket, not `numactl`. Always check `dmesg` for Xid/SXid before concluding a symptom is purely a placement issue.
5. **Trusting a single tool when two disagree.** The NCCL issue #246 case (NCCL's own topology log vs. `nvidia-smi topo -m` disagreeing on the same GPU–NIC edge) is real production evidence that this happens. When two tools disagree, reconcile against a third source (the PCIe tree itself, via `lspci -tv`) rather than picking whichever tool you trust by habit.

## Self-check

**(a) Given this matrix, which NIC do you use for GPUDirect RDMA to GPU3, and why?**

```
       mlx5_0  mlx5_2  mlx5_3   NUMA Aff
GPU3    NODE    SYS     PIX       1
```

**Answer:** `mlx5_3`. Its cell is `PIX` — GPU3→NIC traverses at most a single PCIe bridge, the shortest GDR path, and both sit on GPU3's home NUMA node 1. `mlx5_0` is `NODE` (same NUMA node but a different root complex — extra host-bridge hop) and `mlx5_2` is `SYS` (crosses the socket over UPI — worst case, ~half RDMA bandwidth + latency). GDR wants the traffic to never touch the inter-socket link, so the `PIX` NIC wins; `PXB` would be the acceptable fallback, never `NODE`/`SYS`.

**(b) Name one thing each of the four tools does NOT show.**

**Answer:** `numactl --hardware` — no PCIe/device info at all (can't place a GPU or NIC). `lstopo` — not the *live NVLink mesh* and not the *trained-under-load* link state (shows configured capability/topology, not `LnkSta`). `lspci -vvv` — no NVLink (it isn't a PCIe link) and no friendly NUMA affinity / `cudaN` identity. `nvidia-smi topo -m` — flattens PCIe to one letter (a `PIX` link can still be trained at x4 and it won't say so), shows no NVMe, and — per NCCL issue #246 — can even disagree with NCCL's own internal topology detection on the same edge.

**(c) "GPU at 100% utilization, throughput ~half spec, no error." Host-side causes + the one command that confirms/eliminates each.**

**Answer:** GPU util only means the SMs had work queued, not that data arrived fast or that the silicon itself is healthy. Suspects, in order to check:
- **PCIe link trained low** (x16→x4/x8, or Gen4→Gen3): `lspci -vvv -s <bdf> | grep -E 'LnkCap|LnkSta'` → look for `(downgraded)`.
- **Cross-socket GPU↔NIC path** (GDR over UPI): `nvidia-smi topo -m` → the used NIC's cell is `SYS`/`NODE` instead of `PIX`.
- **Remote memory / wrong NUMA binding** (host↔device staging over UPI): `numactl --hardware` for the layout + `numastat -p <pid>` (or check the job's `--membind`) → memory on the other node.
- **Cross-socket NVMe** feeding the dataloader: `lstopo` → NVMe's home node vs the job's CPU node.
- **NVLink degraded / not all links up** (peer-to-peer capped): `nvidia-smi nvlink -s` → any link `inactive` or below rated GT/s.
- **Thermal / power throttle** (clocks dropped under load): `nvidia-smi -q -d PERFORMANCE,CLOCK` → look for `SW Thermal Slowdown` / `HW Power Brake` active and clocks below base.
- **Hardware fault, not a placement issue** (a component actually broke): `dmesg | grep -i xid` (and SXid on NVSwitch systems) → any entry, especially Xid 79 ("GPU fell off the bus"), means route to a repair ticket, not a re-pin.

Util stays 100% through all of them, which is exactly why you reconcile the topology instead of trusting the utilization dashboard. The discipline: one command per hypothesis, eliminate top-down, never guess.

**(d) Name two real production tools that automate the kind of reconciliation this capstone does manually, and what each one specifically detects.**

**Answer:** Meta's **PCIcrawler** is an open-source CLI that automates PCIe topology mapping and AER (Advanced Error Reporting) counter inspection at fleet scale — the automated version of this capstone's `lspci -tv`/`lspci -vvv` step. Crusoe's **AutoClusters** automatically cordons, drains, and replaces nodes on detecting Xid-79-class hardware failures, explicitly framed around minimizing queue-wait-time as the largest controllable lever on cluster goodput — the automated version of this capstone's "route a hardware fault to a repair ticket, not a re-pin" step. (Modal's DCGM+`dmesg` passive health-check pipeline and CoreWeave's SUNK topology-aware scheduling with health probes are two further valid answers.)

**(e) At 16,384 GPUs, roughly how often does a large training cluster hit an unexpected component failure, and why does that number make manual per-incident topology reconciliation impractical at that scale?**

**Answer:** On the order of one unexpected component failure every few hours — Meta's own published data from training Llama 3 on a 16,384-GPU cluster reported 419 unexpected failures over a 54-day run, with GPU/HBM3 issues responsible for roughly half (cite this figure cautiously; it's a secondary-sourced compilation of Meta's own published numbers, not a number this lesson independently re-derived). At that cadence, a human manually running four tools and reconciling them per incident cannot keep pace — which is exactly why production shops build automated, continuous versions of this reconciliation (PCIcrawler, AutoClusters, SUNK health probes) rather than relying on the capstone's manual, single-node workflow as their production answer. The manual skill is still the prerequisite: you can't validate or debug an automated detector's output if you can't do the reconciliation by hand yourself.

## Connections & what's next

This capstone is the synthesis point for the whole module: lesson 1's NUMA tree gives you the frame, lesson 2's bandwidth hierarchy tells you why placement matters in bytes/sec, lesson 3's `LnkCap`/`LnkSta` gives you the PCIe-health check, lesson 4's reference 8-GPU layout gives you the "what should this look like" baseline, lesson 5's Topology Manager gives you the Kubernetes-level lever to fix a misalignment, lesson 6's GDS/storage placement extends the same locality argument to NVMe, and lesson 7's throttle reasons plus Xid/SXid layer give you the power/thermal and hardware-fault hypotheses. This lesson is where you prove you can hold all seven at once, on a real machine, under interview pressure.

**This closes Module 02b.** The module's own goal statement — understanding the machine underneath the accelerator well enough to place and debug GPU workloads — is what this capstone certifies. From here, this fluency is a direct prerequisite for four modules the README lists as unlocked by 02b:

- **Module 03 (`gpu-hardware`)** — you now have the host-side placement vocabulary to go deeper into the GPU die itself (SM architecture, tensor cores, memory hierarchy inside the package) without re-deriving NUMA/PCIe basics.
- **Module 04 (`gpu-on-kubernetes`)** — lesson 5's Topology Manager and device-plugin material is the foundation for scheduling GPU workloads correctly in production clusters; 04 builds the operational layer on top of it.
- **Module 09 (`networking-topology`)** — the GPU↔NIC pairing and rail-alignment concepts from lessons 4 and 8 extend directly into multi-node network design: the same `PIX`/`SYS` logic, one level up, across the fabric connecting many nodes.
- **Module 10 (`bare-metal-lifecycle`)** — the four-tool reconciliation and acceptance-testing mindset this capstone teaches is exactly what a bare-metal bring-up/acceptance process runs on every new node before it enters a fleet.

Before moving on, answer every item in the [module checkpoint](../checkpoint.md) cold, unassisted — that's the actual gate, not this lesson's self-check.

## References & further reading

**Primary sources**
- `lstopo`/hwloc manpage — the authoritative locality-tree tool; read for its output-format flags (`--of console/svg`, `-.txt`) and how it labels NUMANode/PCIBridge/CoProc/Net/Block. https://manpages.ubuntu.com/manpages/focal/man1/lstopo.1.html
- `nvidia-smi topo -m` legend reference (Sigma2) — a clear worked explanation of the `X/NV#/PIX/PXB/PHB/NODE/SYS` codes with a real matrix; read for the best→worst ordering and the `PIX`-vs-`SYS` GDR rule. https://documentation.sigma2.no/code_development/guides/gpu_usage.html
- NVIDIA GPUDirect RDMA docs — the mechanism behind *why* `SYS` is expensive: RDMA straight between NIC and GPU memory over PCIe, and why crossing the root complex/socket kills it; read the "supported systems"/PCIe topology section. https://docs.nvidia.com/cuda/gpudirect-rdma/

**Real-world engineering blogs**
- Meta Engineering, "How Meta keeps its AI hardware reliable" — FBAR automated remediation, failure taxonomy, ~50× interruption-rate reduction. https://engineering.fb.com/2025/07/22/data-infrastructure/how-meta-keeps-its-ai-hardware-reliable/
- Meta Engineering, "How Facebook deals with PCIe faults to keep our data centers running reliably" — PCIcrawler, the direct precedent for this capstone's Part A tooling. https://engineering.fb.com/2021/06/02/data-center-engineering/how-facebook-deals-with-pcie-faults-to-keep-our-data-centers-running-reliably/
- Modal, "Keeping 20,000 GPUs healthy" — Xid/SXid taxonomy and the `dcgmi`+`dmesg` passive fleet health-check pattern. https://modal.com/blog/gpu-health
- Crusoe, "AutoClusters: Automated GPU Failure Remediation for AI Training Clusters" — automated cordon/drain/replace on Xid-class failures. https://www.crusoe.ai/resources/blog/autoclusters-minimizing-hardware-failures-in-large-gpu-clusters
- CoreWeave, "NVIDIA H100 GPU Benchmark Results: What We Learned from Large-Scale GPU Testing" — SUNK topology-aware scheduling, 49.2% MFU on 128 H100s (2025-era snapshot). https://www.coreweave.com/blog/nvidia-h100-gpu-benchmark-results-what-we-learned-from-large-scale-gpu-testing

**Deeper dives**
- NVIDIA/nccl GitHub Issue #246 — real production case of NCCL's topology detection disagreeing with `nvidia-smi topo -m` on an 8×V100 system; direct evidence for this lesson's "no single tool is authoritative" claim. https://github.com/NVIDIA/nccl/issues/246
- Tom's Hardware, coverage of Meta's Llama 3 training-failure data — secondary-sourced summary of Meta's own published 16,384-GPU cluster failure numbers (419 failures / 54 days); use for the fleet-scale stakes framing, cited cautiously. https://www.tomshardware.com/tech-industry/artificial-intelligence/faulty-nvidia-h100-gpus-and-hbm3-memory-caused-half-of-the-failures-during-llama-3-training-one-failure-every-three-hours-for-metas-16384-gpu-training-cluster
