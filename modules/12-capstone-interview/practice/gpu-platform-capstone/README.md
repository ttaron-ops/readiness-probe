# GPU platform capstone — Module 12 deliverable (the portfolio centerpiece)

The single body of evidence you lead with in every application. It threads all eleven module
artifacts into **one coherent "reference GPU platform"** a hiring manager understands without
meeting you, plus the public proof and the rehearsed narration around it.

> **Label the simulation, every time.** Most of this runs on the
> [fake GPU fleet lab](../../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md).
> *"Benchmarked against the default scheduler on a 200-node **simulated** fleet, trace published"*
> is accurate, impressive, and cannot be punctured; *"I built GPU scheduling infrastructure"*
> cannot survive one follow-up question.

## The organizing thesis

> **Making a multi-tenant GPU fleet observable, attributable, schedulable, and survivable.**

Every artifact becomes one chapter answering a single fleet-operator question. Assembled
**continuously** across the course — each module's deliverable *is* a chapter, so this is
mostly curation, not new build.

## A) The reference-platform repo

One repo whose README is the front door:

- **One architecture diagram** of the whole platform: nodes/GPUs → device-plugin/DCGM →
  operator/exporter → Prometheus/Grafana → scheduler/Kueue → cost/FOCUS pipeline.
- **The value-chain narrative**, chapters linked in order:
  | Question | Chapter (artifact) |
  |----------|--------------------|
  | Who's using my GPUs & what's it costing? | 01 exporter · 02 operator · 03 report · 04 attribution · 11 FOCUS synthesis |
  | Are my dashboards telling the truth? | **05 "your GPU dashboard is lying"** (lead) |
  | How do I share scarce GPUs fairly? | 06 Kueue showback |
  | What does a unit of work cost? | 07 cost-per-token · 10 capex-vs-cloud |
  | What happens when hardware fails? | 08 survive-a-failure · 04 failure-mode log |
  | What's the machine underneath? | 01b container anatomy · 02b topology · 09 network read · 10 KTHW/etcd |

## B) One RFC-style architecture design doc (5–8 pages)

Real design-doc voice: **Goals / Non-goals / Context / Design / Alternatives considered /
Risks / Rollout**. Staff-level content in each — e.g. *Alternatives* = MIG vs time-slicing vs
passthrough tradeoff table; *Risks* = the time-slicing attribution hole (module 11).

## C) The flagship blog + two secondary posts + a demo

- **Flagship: "Your GPU dashboard is lying"** (artifact 05) — contrarian, quotable, and it
  proves the util-vs-MFU/goodput insight that separates GPU-infra people from generic platform
  engineers. Hook (the lie) → measurement (SM_ACTIVE vs GPU_UTIL vs MFU) → reveal (95% util /
  31% MFU) → the fix (what to actually alert on) → takeaway.
- **Secondary:** cost-per-million-tokens (07) and survive-a-failure (08).
- **A 5-minute narrated demo** walking the value chain (narrate the platform, not the code).

## D) The one-page brag doc

Quantified outcomes only — every line a number and a tradeoff.

## Suggested layout

```
gpu-platform-capstone/
├── README.md            # front door: architecture diagram + value-chain narrative + chapter links
├── design-doc.md        # the RFC (goals/non-goals/alternatives/risks/rollout)
├── diagram.*            # the whole-platform architecture diagram
├── chapters/            # the 11 artifacts, curated into ordered chapters
├── blog/
│   ├── dashboard-is-lying.md   # the flagship
│   ├── cost-per-million-tokens.md
│   └── survive-a-failure.md
├── demo/                # 5-min narrated walkthrough (script + recording link)
└── brag-doc.md          # one page of quantified outcomes
```

## Acceptance criteria (matches the [checkpoint](../../checkpoint.md))

- [ ] a front-door README with **one whole-platform architecture diagram** + the value-chain narrative
- [ ] all 11 artifacts curated into ordered **chapters** under the four-verb thesis
- [ ] an **RFC-style design doc** with a real Alternatives-considered and Risks section
- [ ] the **flagship blog** published + 2 secondary posts + a **5-min demo**
- [ ] a one-page **brag doc** where every line lands a metric
- [ ] the whole thing is **self-explanatory** to a hiring manager who never meets you

## Guardrails

- **Publishable-by-default** — this is public proof; scrub any employer-specific rates, cluster
  specifics, or names before posting.
- Flag every $/rate as a dated 2026 snapshot; lead with the **method and the numbers you
  measured**, not vendor figures.
- Lead applications with **neoclouds** (no-DSA; your artifacts are their syllabus), then the
  DSA-flavored NVIDIA/big-tech tier.
