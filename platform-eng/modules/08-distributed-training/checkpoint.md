# 🧮 Checkpoint — 08 · Distributed training infrastructure

The **completion gate**. Prove it with the [Survive-a-failure lab](practice/survive-a-failure/)
and answer the probes cold. You've passed when you can, **unaided**:

## Pass criteria

- [ ] **1 · Debug a hang.** Given a hung job (100% GPU, no error), state the correct
      first-response sequence (`nvidia-smi` all ranks → `NCCL_DEBUG=INFO`/`SUBSYS` →
      identify the dead/straggler rank → XID/DCGM).
- [ ] **2 · Read NCCL.** From `NCCL_DEBUG=INFO` output, name the transport
      (NVLink/PCIe/IB) and the algorithm (ring/tree) selected.
- [ ] **3 · Checkpoint math.** Compute the optimal checkpoint interval from a given MTBF
      and checkpoint cost (Young/Daly), and explain expected wasted work.
- [ ] **4 · Recovery demo.** Show the killed-worker recovery: with checkpointing it resumes
      near the failure point; without, it restarts from zero — and **quantify** the difference.
- [ ] **5 · Parallelism → collective.** For DDP / FSDP / TP / PP, state the dominant
      collective and whether it trades memory or network.
- [ ] **6 · Price it.** Produce a cost-per-**successful**-run number that includes failure
      overhead, and explain which lever (checkpoint frequency, restart speed, MFU) most reduces it.

## Depth probes (answer cold)

- [ ] Why does FSDP trade network for memory vs DDP?
- [ ] Which parallelism must stay inside one NVLink domain, and why? (tie to 02b)
- [ ] Job hangs at 100% GPU, no error — your first three commands?
- [ ] Which NCCL env var confirms whether it's using InfiniBand vs TCP sockets?
- [ ] Why does all-reduce cost grow with world size? What MFU would concern you, and the three usual causes?
- [ ] Derive the optimal checkpoint interval for MTBF=3h, checkpoint cost=2min.
- [ ] What triggers an automated restart in production? (tie to 05 XID/DCGM)
- [ ] GPU SM% low, CPU 100% — diagnosis, and why it's a *cost* bug?

## Interview-readiness proxy

- [ ] You have run a multi-GPU DDP job and deliberately killed a worker mid-run.
- [ ] You can compute cost per epoch / per experiment including failure overhead.
- [ ] You can explain to a research lead why a slow data loader is an expensive problem.

## Answers / notes

_Record answers as you close each lesson; link the lab logs + cost-per-run calc for items 1–6._
