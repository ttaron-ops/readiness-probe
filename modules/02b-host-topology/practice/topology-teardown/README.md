# Topology Teardown of a GPU Node — Module 02b deliverable

Reconcile a **real** GPU node's topology from four tools into one picture, then
**measure** what a NUMA misalignment costs. Portfolio-grade: it rehearses the
interview whiteboard task exactly, and Part B is the FinOps payload that is your
whole differentiator.

> Finishable on a by-the-hour rented multi-GPU box (Lambda / CoreWeave / RunPod — a
> 2×/4×/8× instance is ideal; a single-GPU 2-socket box covers most of it).

## Part A — the reconciled diagram

Run all four tools on **one** machine and produce a single topology diagram every
tool agrees with:

| Tool | What it contributes | Blind spot |
|------|---------------------|------------|
| `numactl --hardware` | NUMA nodes + memory per node | no PCIe detail |
| `lstopo --output-format png` | authoritative I/O-locality tree (socket→NUMA→bridge→device) | link speed |
| `lspci -tv` + `lspci -vvv` | PCIe tree; **`LnkCap` vs `LnkSta`** (degraded links) | NUMA/NVLink |
| `nvidia-smi topo -m` | GPU↔GPU↔NIC link matrix (NV#/PIX/PXB/PHB/NODE/SYS) | NVMe, CPU |

**Diagram must show:** sockets/NUMA nodes · each GPU's home socket · GPU↔NIC pairings
and their link class · NVMe placement · any misalignment or link trained below spec.

**Acceptance:** a `teardown.md` (+ `lstopo.png`) reconstructing the node's topology
from the four tools, with the diagram and a one-paragraph read of its failure modes.

## Part B — the measured misalignment (the FinOps payload)

Force a misalignment and **measure the cost**:

1. Run a host↔device bandwidth test (CUDA `bandwidthTest`) or a small NCCL
   `all_reduce` / training step, **NUMA-aligned** (`numactl --cpunodebind=N --membind=N`
   matching the GPU's home node).
2. Run it again **deliberately misaligned** (bind to the remote node).
3. Record the throughput delta (commonly 20–50%+ on cross-socket paths).
4. Write one page: *which production metric would have caught this, and which
   wouldn't* — explicitly connecting to "GPU looks busy, delivers less than spec, no
   dashboard shows it." Propose the **exporter/alert you'd build** (ties to the
   CoreWeave "create alerts" JD line, and to your capstone operator).

**Acceptance:** a `misalignment.md` with the aligned-vs-misaligned numbers, the
methodology, and the metric/alert proposal.

## Suggested layout

```
topology-teardown/
├── teardown.md          # Part A: reconciled diagram + failure-mode read
├── lstopo.png           # the lstopo output
├── raw/                 # captured tool outputs (numactl, lspci -vvv, topo -m)
├── misalignment.md      # Part B: measured aligned-vs-misaligned + metric/alert proposal
└── README.md            # how to reproduce
```

## Guardrails

- Publishable-by-default (Part B is a strong blog post) — scrub real hostnames/cluster
  details before posting.
- Do power-cap / throttle experiments (L7) carefully on the rented box; they're safe on
  a dedicated instance but not on shared hardware.
- No secrets or kubeconfigs in git (repo `.gitignore` guards these).
