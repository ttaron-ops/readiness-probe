# 🧬 Checkpoint — 02b · Host hardware and topology

The **completion gate**. Prove each with the [Topology Teardown](practice/topology-teardown/)
deliverable and answer the probes cold. You've passed when you can, **unassisted**:

## Pass criteria

- [ ] **1 · Reconcile the four tools.** On a real machine, run `lstopo` / `lspci -tv` /
      `nvidia-smi topo -m` / `numactl --hardware` and produce **one** topology diagram
      all four agree with — correctly identifying each GPU's home NUMA node, GPU↔NIC
      pairings, and any PCIe link trained below spec (`LnkSta < LnkCap`).
- [ ] **2 · Decode `nvidia-smi topo -m` cold.** Explain every legend code
      (`X` / `NV#` / `PIX` / `PXB` / `PHB` / `NODE` / `SYS`) and, given a matrix, pick the
      correct NIC for GPUDirect RDMA to a named GPU and justify it.
- [ ] **3 · Configure and reason about Topology Manager.** Set `single-numa-node` +
      CPU Manager `static` + Memory Manager `Static`; state what each policy
      **guarantees vs attempts**, demonstrate a pod admitted vs rejected, and explain
      why GPU–CPU alignment silently fails when the device plugin omits `TopologyInfo`.
- [ ] **4 · Diagnose from symptoms alone.** Given "GPU at 100% utilization, throughput
      ~half spec, no obvious error," enumerate the host-side causes (NUMA-remote memory,
      cross-socket GPU–NIC, PCIe link degraded, PCIe contention, power/thermal throttle)
      and the one command that confirms or eliminates each.
- [ ] **5 · Quantify the cost.** Show a **measured** throughput difference between an
      aligned and a misaligned run, and name which production metric would/wouldn't
      have surfaced it.

## Depth probes (answer cold)

- [ ] Why is a PCIe device "local" to exactly one socket, and what does crossing the inter-socket link cost?
- [ ] A GPU trained at Gen3 x8 vs Gen5 x16 — bandwidth ratio, and how do you detect it?
- [ ] Which GPUs share socket 0 on a standard HGX H100 node, and where does GPU5's NIC attach?
- [ ] Why is there no PCIe switch on the HGX baseboard, and what carries GPU–GPU traffic instead?
- [ ] What does `single-numa-node` do when a pod can't fit one NUMA node? How does it differ from best-effort?
- [ ] Why does a misplaced NVMe look like "GPU idle," and what does GPUDirect Storage change?
- [ ] Which `clocks_throttle_reasons` value means "paying for cooling you don't have"?

## Interview-readiness proxy

- [ ] Whiteboard: given one node's four-tool output, draw its topology and predict where a distributed job bottlenecks.
- [ ] Argue for co-located placement with concrete bandwidth/latency numbers, not intuition.

## Answers / notes

_Record answers as you close each lesson; link the teardown evidence for items 1–5._
