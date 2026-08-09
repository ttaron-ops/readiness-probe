---
lesson: 02
title: "Capstone synthesis: thread 11 artifacts into one platform"
module: 12
concept: "one coherent platform narrative"
status: not-started
est_time: "4 hrs"
artifacts: ["capstone storyboard: chapter list + whole-platform architecture diagram + through-line thesis"]
---

# Capstone synthesis: thread 11 artifacts into one platform

## Why this matters
Eleven artifacts scattered across eleven modules read as eleven hobby projects. The same eleven artifacts, threaded onto a single spine, read as **one engineer who built a reference GPU platform** — and that is a staff-level signal no individual project can carry alone. Interviewers at CoreWeave and Nebius are explicitly hiring "area owners" who deliver "measurable improvements across services." A coherent platform narrative is the artifact that proves you think in *systems and services*, not tasks.

The difference between senior and staff in a portfolio review is almost never more projects. It is a through-line: a single thesis, tradeoffs you can name (including what you rejected), real numbers, and honesty about failure. A candidate who says "here are my six GPU projects" is senior. A candidate who says "I set out to make a multi-tenant GPU fleet observable, attributable, schedulable, and survivable, and here are the five chapters that prove each" is staff — because they've imposed an operating thesis on their own work and can defend the joints between chapters.

This lesson designs that narrative once, so every loop in Lesson 01 draws from the same well-rehearsed story instead of improvising a fresh pitch per company.

## What's new here
- **Skip** (you own it): portfolio-101, how to write a resume bullet, generic "tell a story" advice, the brag-document mechanic itself.
- **New**: the *organizing thesis* that turns 11 artifacts into one platform, and the value-chain mapping where each artifact answers one specific fleet-operator question.
- **New**: the concrete staff-vs-senior delta for *this* portfolio — which 3-5 chapters to feature, which real numbers to lead with, which tradeoffs to name as rejected alternatives.
- **New**: the single highest-signal artifact to open with, and the storyboard you'll fill into the capstone README.

## Core notes

### The organizing thesis
**"Making a multi-tenant GPU fleet observable, attributable, schedulable, and survivable."** Those four verbs are the spine. Every chapter hangs off one of them. Say this sentence first in any portfolio review; it frames all eleven artifacts as one deliberate program of work.

### The value chain — each artifact answers one fleet-operator question

| Fleet-operator question | Chapter artifacts |
|---|---|
| Who's using my GPUs & what's it costing? | 01 gpu-cost-exporter, 02 gpu-cost-operator, 03 efficiency/cost report, 04 per-pod attribution, 11 FOCUS synthesis |
| Are my dashboards telling the truth? | 05 "your GPU dashboard is lying" — util vs MFU/goodput (the sharpest angle) |
| How do I share scarce GPUs fairly? | 06 Kueue showback |
| What does a unit of work cost? | 07 cost-per-million-tokens, 10 capex-vs-cloud |
| What happens when hardware fails? | 08 survive-a-failure lab, 04 failure-mode log |
| What's the machine underneath? | 01b container anatomy, 02b topology teardown, 09 network read, 10 KTHW/etcd |

Note the four verbs of the thesis map cleanly: observable (row 1-2), attributable (row 1, 3), schedulable (row 3-4), survivable (row 5) — with "the machine underneath" (row 6) as the substrate proving depth.

### What makes it read STAFF, not senior
Do **not** present all eleven exhaustively — that reads as a list. Feature **3-5 standout chapters**, at least one at production/fleet scale, and for each cover the full arc: **problem → your ownership → architecture → stack → metrics → post-launch evolution.**
- **Real numbers, not adjectives.** "The dashboard read 95% GPU utilization while MFU was 31%" beats "improved observability." Numbers are the staff tell.
- **Tradeoff narration — name what you rejected.** GPU sharing: MIG vs time-slice vs full passthrough, and why you chose what you chose. Cost model: chargeback vs showback. Scheduling: Kueue vs Volcano vs Slurm. Staff engineers are defined by the alternatives they can articulate rejecting.
- **Failure honesty.** The failure-mode log (04) and survive-a-failure lab (08) are not weaknesses to hide — they are operational-maturity signals. Leading with "here's how the fleet breaks and how I designed for it" is a staff move.
- **A single through-line thesis** stated once and returned to. It's the connective tissue between chapters.

### The artifact to LEAD with
**"Your GPU dashboard is lying" (05).** It is contrarian, memorable, and instantly proves GPU-fleet thinking — the util-vs-MFU/goodput gap is a distinction only someone who has actually operated accelerators makes. It hooks the interviewer in one sentence. Back it immediately with **gpu-cost-operator (02)** — "and I ship custom operators against a real fleet, not just dashboards" — which pairs the contrarian insight with production-shipping credibility. Insight + shipping = staff.

### The whole-platform view
The value chain isn't a list of projects; it's a stack. Exporter/operator emit signals → attribution and cost models consume them → showback/scheduling act on them → the survivability layer protects the whole thing → all of it sits on the machine substrate (topology, network, etcd). Your one architecture diagram should show that flow, not eleven disconnected boxes.

## Worked example
A 30-second staff-signal opener, rehearsed: *"I set out to make a multi-tenant GPU fleet observable, attributable, schedulable, and survivable — one reference platform, not scattered scripts. The chapter I'll open with is counterintuitive: your GPU dashboard is lying to you. On a real workload the dashboard read 95% utilization while model-FLOPs-utilization was 31% — the GPU was busy spinning, not computing. I built an exporter and a custom Kubernetes operator to surface goodput and MFU, not just util, then hung per-pod cost attribution off it — FOCUS-aligned so finance could actually read it. I chose showback over hard chargeback because in a scarce-GPU multi-tenant setting chargeback creates hoarding; I rejected Volcano and raw Slurm for Kueue because it's Kubernetes-native and I was already operating an operator-driven control plane. And I designed the whole thing assuming hardware fails — I keep a failure-mode log and ran a survive-a-failure training lab, because on a 250k-GPU fleet a node dies every few minutes."* That paragraph names the thesis, leads with the contrarian chapter, cites a real number, narrates two rejected alternatives, and closes on failure maturity — the full staff delta in one breath.

## Practice
Build the storyboard in [GPU platform capstone](../practice/gpu-platform-capstone/README.md):
1. Write the **through-line sentence** (start from the four-verb thesis; make it yours) at the top of the capstone README.
2. List your **chapters** in value-chain order, each tagged with the fleet-operator question it answers and the artifact IDs behind it.
3. Pick your **3-5 featured chapters** (≥1 production-scale) and for each write the problem→ownership→architecture→stack→metrics→evolution arc, with at least one real number.
4. For each featured chapter, write the **rejected alternative(s)** you'll name (MIG/time-slice/passthrough, chargeback/showback, Kueue/Volcano/Slurm).
5. Draw **one architecture diagram of the whole platform** showing signals flowing exporter/operator → attribution/cost → showback/scheduling → survivability → machine substrate.
6. Write and rehearse your **30-second opener** leading with "dashboard is lying" (05) backed by gpu-cost-operator (02).

## Self-check
- What is the organizing thesis that turns 11 artifacts into one platform, and what are its four verbs? **Answer:** "Making a multi-tenant GPU fleet observable, attributable, schedulable, and survivable" — observable, attributable, schedulable, survivable, each verb anchoring a chapter cluster.
- Which single artifact should you lead a portfolio review with and why, and what backs it? **Answer:** "Your GPU dashboard is lying" (05) — the contrarian util-vs-MFU/goodput angle instantly proves GPU-fleet thinking; back it with gpu-cost-operator (02) to pair the insight with production-shipping credibility.
- Name three tradeoff pairs you should narrate as rejected alternatives to read staff, not senior. **Answer:** GPU sharing (MIG vs time-slice vs full passthrough), cost model (chargeback vs showback), and scheduling (Kueue vs Volcano vs Slurm).

## Resources
- Engineering portfolio guidance: https://www.fonzi.ai/blog/portfolio-for-engineer
- Brag documents (Julia Evans): https://jvns.ca/blog/brag-documents
- CoreWeave Kueue blog (showback/scheduling chapter): https://www.coreweave.com/blog/kueue-a-kubernetes-native-system-for-ai-training-workloads
- Your prior module artifacts (01-11) — the raw material each chapter draws from.

[🎓 12 — Capstone & interview preparation](../README.md)
