# 🔗 Checkpoint — 09 · Networking and topology

The **completion gate**. Prove it with the [Network architecture read](practice/network-architecture-read/)
and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Read a `topo -m`.** Name each GPU's rail + the correct GPUDirect NIC, explaining
      any `SYS`/`PXB` penalty (ties 02b → fabric).
- [ ] **2 · Compute oversubscription.** From a described build, compute the ratio and state the
      worst-case per-GPU bisection bandwidth for a cross-tier collective.
- [ ] **3 · Lossless RoCE.** Explain why RoCE needs lossless Ethernet — naming **PFC, ECN,
      DCQCN** — and describe one failure mode (PFC deadlock or head-of-line blocking).
- [ ] **4 · Argue IB vs RoCE.** For two different cluster scenarios with different verdicts,
      citing ≥4 axes (latency, NCCL GB/s, tuning risk, cost/lock-in, SHARP).
- [ ] **5 · Rail-optimized.** Define it and explain why it permits spine oversubscription
      without hurting LLM all-reduce.
- [ ] **6 · Trace GPUDirect/RDMA.** End-to-end (GPU→NIC→leaf→spine), naming what kernel-bypass
      removes vs the 01b datapath.
- [ ] **7 · The K8s path.** What Multus + SR-IOV device plugin + Network Operator each
      contribute, and why default CNI can't do RDMA.
- [ ] **8 · Deliverable graded pass.** Bottleneck prediction correct + quantified; placement
      argument uses real bandwidth numbers; IB/RoCE verdict defensible.

## Depth probes (answer cold)

- [ ] Why does cross-rail GPU traffic prefer NVLink over the NIC?
- [ ] A 4:1-oversubscribed spine — worst-case per-GPU bandwidth for an all-to-all spanning two leaves?
- [ ] What does the CPU do during an RDMA WRITE — and what does it *not* do?
- [ ] What is a PFC deadlock, and why does DCQCN exist if you already have PFC?
- [ ] What breaks GPUDirect RDMA if the GPU and NIC are on different root complexes?
- [ ] Which collective shapes benefit most from SHARP in-network reduction?
- [ ] Why isn't the default CNI enough for RDMA — what does Multus add?
- [ ] Turn "co-locate this job in one pod" into a per-GPU bandwidth number.

## Fail signals (avoid these)

- [ ] "IB is just faster" with no scenario nuance · can't turn a topology into a bandwidth number ·
      treats RoCE as drop-in Ethernet.

## Interview-readiness proxy

- [ ] Given a cluster diagram, you can predict where a distributed job bottlenecks.
- [ ] You can argue for co-located placement with concrete bandwidth numbers, not intuition.

## Answers / notes

_Record answers as you close each lesson; link the architecture-read doc for items 1–8._
