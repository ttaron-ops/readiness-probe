# Survive-a-failure lab — Module 08 deliverable

Run a small distributed job, kill a worker, recover from checkpoint, and **price the
failure overhead** — the artifact that turns the reliability story into a dollar figure
and feeds Module 11. On **2 rented GPUs** (single node, e.g. 2×L4/A10 or 2×H100 spot; ~$20–40 total).

## The lab

1. Run a small multi-GPU **PyTorch DDP** job (a tiny transformer or ResNet, a few hundred steps).
2. Add **checkpoint save/resume**.
3. **Kill a worker mid-run twice:**
   - once **without** checkpointing → measure lost work (restart from zero),
   - once **with** checkpointing + torchrun elastic → measure re-rendezvous time and lost work.
4. Capture `NCCL_DEBUG=INFO` output once → identify the **transport** (NVLink/PCIe/IB) and **algorithm** (ring/tree) chosen.
5. **Compute cost-per-epoch and true cost-per-successful-run**, including expected failure
   overhead derived from an assumed MTBF and your measured checkpoint interval/cost.

## The payoff (the FinOps piece)

```
true cost per successful run
  = raw_gpu_hours × $/gpu_hr × (1 + failure_overhead_fraction) / mfu_adjustment

failure_overhead_fraction ≈ (wasted_work_per_failure × expected_failures + restart_time) / raw_runtime
optimal checkpoint interval ≈ sqrt(2 × checkpoint_cost × MTBF)     # Young/Daly
```

State every assumption; flag the $/GPU-hr as a dated snapshot.

## Suggested layout

```
survive-a-failure/
├── train.py             # tiny DDP job with checkpoint save/resume
├── run.sh               # torchrun elastic launch + the kill-a-worker script
├── nccl-debug.log       # the captured NCCL_DEBUG=INFO (transport + algo)
├── logs/                # step-time / GPU-hour logs (with vs without checkpoint)
├── cost.md              # cost-per-epoch + cost-per-successful-run incl. failure overhead
└── README.md            # what happened, the numbers, the levers
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] a DDP job that checkpoints and resumes; killed-worker recovery shown **with vs without** checkpointing, difference quantified
- [ ] one `NCCL_DEBUG=INFO` capture with the transport + algorithm identified
- [ ] a measured checkpoint write cost + a computed optimal interval (Young/Daly) for a stated MTBF
- [ ] a **cost-per-successful-run** figure that includes failure overhead (the Module 11 input)
- [ ] *(optional stretch)* the same job submitted as a PyTorchJob via the Training Operator on kind/minikube

## Guardrails

- Single node, 2 GPUs is enough — no multi-node fabric needed for this lab.
- Rent on-demand, kill promptly; a tiny model + a few hundred steps is plenty.
- No credentials or real cost rates committed (repo `.gitignore` guards these).
- The cost model folds into `gpu-cost-operator` / Module 11 — keep it importable.
