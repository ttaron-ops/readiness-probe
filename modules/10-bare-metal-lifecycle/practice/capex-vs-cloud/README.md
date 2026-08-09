# Capex-vs-cloud model + KTHW/etcd writeup — Module 10 deliverable

Two artifacts, both finishable on cheap VMs — the first is your FinOps differentiator (a
rigorous build-vs-buy model nobody's written well); the second answers the two hardest
bare-metal interview questions cold.

## A) The capex-vs-cloud crossover model for a 64-GPU fleet

A spreadsheet/notebook model with **your own inputs** and a break-even curve. Anchor
numbers from research (replace with your assumptions; **flag all $ as 2026 snapshots**):

- **Hardware capex** — 8×H100 node ≈ $222K–$383K; a 64-GPU fleet (8 nodes) ≈ $1.8M–$3.1M.
- **Power** — ~5.5–6.5 kW/node sustained; ~$4K/mo/node at $0.10/kWh → ~$32K/mo fleet before
  PUE; apply your colocation **PUE ~1.3–1.5**.
- **Colocation / facility, network (InfiniBand), maintenance contracts** — scope from your on-prem experience.
- **Depreciation** — 3-yr straight-line; model the 2026 resale-glut as a **residual-value sensitivity**.
- **Staffing** — platform-engineer FTE amortized across the fleet.
- **Comparison rail** — neocloud rental **$2.0–2.6/GPU/hr**; on-prem TCO ≈ $4–5/GPU-hr at moderate
  utilisation, dropping below rental as utilisation → 100%.
- **Output** — the crossover chart: owned $/GPU-hr vs rental, plotted against **utilisation %**
  and **months owned**; state the **break-even utilisation** and the **payback month**.

**Headline to defend:** *owning wins only above ~X% sustained utilisation over ≥18–24 months;
below that, rent.*

## B) "Kubernetes-the-Hard-Way + etcd break/restore" writeup

A publishable runbook:
- hand-built control plane (**every cert enumerated**, from L1),
- the **etcd failure drills** from L2 — quota-exceeded recovery (defrag + `alarm disarm`) and
  snapshot restore (`etcdutl snapshot restore`) — with the exact commands and what the apiserver
  did at each step.

This is the artifact that answers interview Q1 ("build a control plane by hand") and Q2
("recover etcd") cold.

## Suggested layout

```
capex-vs-cloud/
├── model/               # the 64-GPU crossover spreadsheet/notebook + the chart
│   └── crossover.md     # inputs, assumptions, break-even + payback, sensitivity
├── kthw-etcd/
│   ├── control-plane.md # hand-built control plane + cert enumeration
│   └── etcd-drills.md   # quota-exceeded recovery + snapshot restore, exact commands
└── README.md            # how to reproduce
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] a 64-GPU capex-vs-cloud model with your own inputs → crossover chart + break-even utilisation + payback month + a sensitivity
- [ ] a hand-built-control-plane writeup enumerating every cert (KTHW)
- [ ] the two etcd drills documented with exact commands and the apiserver's behavior at each step
- [ ] (from L8) a MetalLB BGP service VIP stood up on the cluster

## Guardrails

- Everything runs on cheap VMs — no real GPUs; the economic model needs no infra.
- Flag every $/hardware/power figure as a dated snapshot; the durable content is the *model structure*.
- Publishable-by-default; scrub any employer-specific numbers before posting.
