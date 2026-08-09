---
lesson: 03
title: "Portfolio & writeup build: the repo, the design doc, the diagram"
module: 12
concept: "evidence a manager reads cold"
status: not-started
est_time: "4 hrs"
artifacts: ["a published capstone repo README (whole-fleet architecture diagram + value-chain narrative + chapter links), one 5-8 page RFC-style architecture design doc, and a brag-doc one-pager"]
---

# Portfolio & writeup build: the repo, the design doc, the diagram

[🎓 12 — Capstone & interview preparation](../README.md)

## Why this matters

Lesson 02 built the *narrative* — the arc that ties artifacts 01-11 into one platform story you can tell out loud. This lesson turns that narrative into **published evidence a hiring manager can read cold**, with no meeting, no context from you, in the four minutes they spend on your GitHub before deciding whether to route you to a Staff loop. The narrative in your head converts nobody. A repo that opens with one architecture diagram and a value-chain sentence, a design doc that reasons in Goals/Non-goals/Alternatives/Risks, and a one-page quantified brag-doc — those convert. The delta between "I built a bunch of GPU tooling" and "here is a coherent GPU platform, here is why each decision, here is the number" is exactly the Staff signal.

The specific trap for a strong IC is *exhaustiveness*. You have eleven artifacts and the instinct is to list all eleven with equal weight. That reads as a portfolio of exercises, not a platform. Staff-level packaging is ruthlessly *narrative and selective*: three-to-five standout pieces carrying the story, at least one at production scale, each with a metric and a post-launch evolution, and the rest demoted to a chapter index. The reviewer is not grading completeness; they are looking for judgment — what you chose to build, what you chose *not* to (non-goals), and whether you can defend a tradeoff in writing the way you'd defend it in a design review.

The GPU-infra-specific reason this matters: your generic platform-eng credibility is assumed at this level, so the writeup has to surface the parts a generic SRE could *not* have written. The time-slicing attribution hole from module 11, the util-vs-goodput gap from module 05, MIG-vs-time-slice-vs-passthrough tradeoffs, topology-aware scheduling — these are the load-bearing content. If your README and design doc could have been produced by someone who has never operated a GPU fleet, they have failed regardless of how polished they are.

## What's new here

- **You already know** how to write clean READMEs, structure a repo, and draw a boxes-and-arrows diagram. **Skip** the mechanics of Markdown and Mermaid. The new thing is the *editorial* discipline: front-door diagram + value-chain narrative + selective demotion of the rest.
- **You already know** how to write a design doc at your current job. **Skip** learning RFC structure from scratch. The new thing is what **staff-level GPU content** looks like inside each section — specifically Alternatives (the MIG/time-slice/passthrough table) and Risks (the time-slicing attribution hole).
- **New angle 1:** the **portfolio credibility rubric** — 3-5 standout > exhaustive, ≥1 production-scale, each entry = problem → role/ownership → architecture → stack → **metric** → post-launch evolution.
- **New angle 2:** the **brag-doc** as a distinct artifact from the resume — a quantified-outcomes sheet you mine for interview stories and promo packets, not a narrative CV.
- **New angle 3:** publishing the README as a **shareable page/artifact** so the front door is one link, not a git clone.

## Core notes

### The repo README as front door

The README is read top-to-bottom for about 30 seconds before the reader either scrolls or leaves. Front-load in this order:

1. **One whole-platform architecture diagram** — the single image that shows the fleet as a system (see "Drawing the diagram" below). It must be the first thing after the title.
2. **The value-chain sentence** — one line that names the arc: *attribution → scheduling → observability → economics → lifecycle → substrate.* This is the spine; every artifact hangs off one link in it.
3. **The chapter index** — each artifact 01-11 as a one-line link, grouped under the value-chain stage it serves. Not eleven equal headings; a table or nested list keyed to the chain.
4. **The "start here" pointer** — link straight to the flagship (lesson 04's blog) and the design doc, so a reader who wants depth has one obvious next click.

Consider publishing the README as a **shareable artifact/page** (a hosted HTML render) so the front door is a URL you can paste into an application or a DM, with the diagram inline and the chapter links live — not something that requires cloning the repo to appreciate.

### The value-chain narrative

Map every artifact onto the chain so the reader sees a platform, not a pile:

| Stage | What it answers | Backing artifacts |
|---|---|---|
| Attribution | Whose GPU-hours are these? | per-pod attribution (04), cost-exporter (01), container-anatomy (01b) |
| Scheduling | Who gets the GPU next, and where? | Kueue showback (06), topology teardown (02b) |
| Observability | Is the GPU actually working? | "dashboard is lying" (05), efficiency/cost report (03) |
| Economics | What does a unit cost, and is it worth it? | cost-per-1M-tokens (07), FOCUS synthesis (11), capex-vs-cloud (10) |
| Lifecycle | What happens when it breaks? | survive-a-failure lab (08), failure-mode log (04) |
| Substrate | What is it all standing on? | network architecture read (09), KTHW/etcd (10), operator (02) |

### The RFC-style architecture design doc (5-8 pages)

One design doc, in real design-doc voice — declarative, tradeoff-forward, written as if proposing to a review board. Section template and what **staff-level** content looks like in each:

| Section | Junior content | Staff content |
|---|---|---|
| **Goals** | "Build a GPU cost system" | Specific, measurable: "attribute >95% of GPU-hours to a namespace within one billing day; surface goodput not just util; keep exporter overhead <1% of a GPU." |
| **Non-goals** | (omitted) | Explicit scope cuts: "not a scheduler replacement; not multi-cloud FOCUS in v1; not per-kernel profiling." Non-goals are where reviewers read your judgment. |
| **Context** | "We have GPUs" | The forcing function: fleet size, $/GPU-hr, the specific pain (idle tax, unattributed spend, the util-lie) with a number attached. |
| **Design** | Boxes and arrows | The data flow term by term: DCGM → device-plugin → exporter → Prometheus → attribution join → FOCUS. Where the join key lives, how you handle MIG slices, what the cardinality budget is. |
| **Alternatives considered** | (omitted) | The **MIG vs time-slice vs passthrough** tradeoff table (below). Alternatives is the section that most separates Staff from Senior. |
| **Risks** | "It might break" | The **time-slicing attribution hole** (module 11): time-sliced GPUs report shared util that cannot be cleanly attributed per-tenant, so cost attribution silently degrades. Name it, quantify the error, state the mitigation. |
| **Rollout** | "Deploy it" | Phased: shadow mode → one namespace → fleet, with the metric that gates each phase and the rollback trigger. |

**The Alternatives table (partitioning a GPU):**

| Option | Isolation | Attribution cleanliness | Utilization | When |
|---|---|---|---|---|
| **MIG** | Hard (separate SMs, memory, cache) | Clean — each instance is a distinct billable unit | Rigid partitions; can strand capacity | Multi-tenant, need isolation + defensible chargeback |
| **Time-slicing** | Soft (context-switched, shared memory) | **Broken** — shared util can't be split per-tenant (the module-11 hole) | High if bursty | Trusted/bursty workloads where isolation and per-tenant $ don't matter |
| **Passthrough (full GPU)** | Total (whole device to one pod) | Trivially clean (1 GPU = 1 tenant) | Poor if the workload can't fill it | Large single-tenant training/serving jobs |

The Risks section then reaches back into this table: *because* v1 uses time-slicing for the dev-namespace tier, attribution there is best-effort and cost is allocated proportionally-by-request, not measured — a known, bounded inaccuracy, stated up front.

### The portfolio credibility rubric

From staff-portfolio guidance — apply it to every entry you keep:

- **3-5 standout projects beat an exhaustive list.** Pick the ones that carry the value-chain story. Demote the rest to the chapter index.
- **At least one must be production-scale** (or convincingly production-shaped): real fleet numbers, failure handling, rollout thinking — not a toy.
- **Each standout entry follows the spine:** problem → your role/ownership → architecture → stack → **metric** → post-launch evolution. The metric is non-negotiable; "reduced unattributed GPU spend from 34% to <5%" beats "built an attribution system."
- **Show the reasoning, not just the result:** include the design doc, the diagram, and at least one tradeoff you navigated (why Kueue over raw priority classes, why DCGM over nvidia-smi scraping).

### The brag-doc (jvns)

A **one-page quantified-outcomes sheet**, distinct from the resume. Julia Evans' brag-document is a running log of what you did and the impact, mined later for reviews, promo packets, and interview stories. For this portfolio it is the raw material behind the metrics: bullet lines of the form *"shipped X → moved metric Y from A to B → which unblocked Z."* You do not publish the brag-doc; you publish its distilled metrics in the README and pull its stories in the interview loop. Keeping it current is the cheapest way to never again scramble for "a time you had impact."

### Drawing the architecture diagram

Draw the fleet as data flowing left-to-right, one layer per box-column:

```
nodes / GPUs  →  device-plugin / DCGM  →  operator / exporter  →  Prometheus / Grafana
                                                     ↓
                              scheduler (Kueue)  →  cost / FOCUS pipeline
```

Rules that keep it staff-legible: every arrow is a real data flow (metrics, scheduling decisions, dollars), label the arrows not just the boxes, show *where attribution happens* (the join point), and mark the one place the util-lie/goodput distinction lives. Use Mermaid so it renders inline on GitHub and in the shareable artifact page. One diagram, one screen, no scroll.

## Worked example

A reviewer opens your capstone repo cold. Top of README: a Mermaid diagram showing H100 nodes → DCGM/device-plugin → your operator+exporter → Prometheus/Grafana, with a branch to Kueue and down to a FOCUS cost pipeline; the attribution join is circled where exporter output meets the rate table. Under it, one line: *"A GPU platform read as a value chain: attribution → scheduling → observability → economics → lifecycle → substrate."* Then a six-row table linking each stage to its artifacts.

They click the design doc. Goals: *">95% of GPU-hours attributed to a namespace within one billing day; goodput surfaced alongside util; <1% exporter overhead."* Non-goals cut multi-cloud FOCUS and per-kernel profiling out of v1. Alternatives has the MIG/time-slice/passthrough table. Risks opens with: *"Time-sliced GPUs in the dev tier cannot be cleanly attributed per-tenant (shared util); v1 allocates their cost proportionally by request count, a bounded ~±15% error, revisited when MIG lands."* Rollout is shadow → one namespace → fleet, gated on attribution coverage crossing 95%.

In four minutes the reviewer has learned: this person operates GPU fleets (not generic k8s), reasons in tradeoffs and non-goals, knows the time-slicing attribution hole exists and named it before being asked, and ships in phases with a rollback trigger. That is a Staff signal delivered with zero conversation. Contrast the failure mode: eleven equally-weighted repo links, no diagram, a design doc that says "deploy it" under Rollout and omits Alternatives entirely — indistinguishable from a competent SRE who has never touched a GPU.

## Practice

Feeds `../practice/gpu-platform-capstone/README.md`:

1. **Write the front-door README.** One Mermaid whole-fleet diagram at the top, the value-chain sentence, and the six-row stage→artifacts table linking your real 01-11 chapters. Demote everything below the top 3-5 into the index.
2. **Draft the design-doc skeleton** with all seven sections (Goals / Non-goals / Context / Design / Alternatives / Risks / Rollout). Fully write Alternatives (the MIG/time-slice/passthrough table) and Risks (the time-slicing attribution hole with a quantified error and a mitigation). Leave the rest as bulleted stubs to fill in lesson-by-lesson.
3. **Draw the architecture diagram** in Mermaid, arrows labeled with what flows, the attribution join point marked, the util/goodput distinction marked.
4. **Start the brag-doc** as a one-pager: for each of your top 5 artifacts, one line of problem → what you did → metric moved. Distill three of those metrics into the README.
5. **Optional:** publish the README as a shareable artifact page and put the URL at the top of the repo.

## Self-check

- Your capstone has eleven artifacts. Why is listing all eleven as equal-weight repo entries a *weaker* portfolio than featuring four? **Answer:** Eleven equal entries read as a pile of exercises and force the reviewer to do the synthesis you should have done; the rubric is judgment, not completeness. Featuring 3-5 standouts (≥1 production-scale, each with problem→role→architecture→stack→metric→evolution) and demoting the rest to a chapter index shows editorial selection and lets the value-chain narrative carry the story — which is the Staff signal.
- Which design-doc section most separates a Staff writeup from a Senior one, and what GPU-specific content belongs there? **Answer:** Alternatives considered (closely followed by Non-goals) — a Senior doc often omits it. The GPU-specific content is the MIG vs time-slice vs passthrough tradeoff table across isolation, attribution cleanliness, and utilization, with a stated "when to use each." It proves you chose your design against real options rather than defaulting.
- What is the time-slicing attribution hole, and where in the design doc does it go? **Answer:** Time-sliced GPUs share one device via context-switching, so DCGM reports shared utilization that cannot be split cleanly per-tenant — per-tenant cost attribution degrades to an estimate (module 11). It belongs in Risks: name it, quantify the error (e.g. ~±15%), state the mitigation (allocate proportionally by request count in v1; move that tier to MIG for clean attribution later).

## Resources

- Fonzi — how to build an engineering portfolio that gets you hired: https://fonzi.ai/blog/portfolio-for-engineer
- Julia Evans (jvns) — Get your work recognized: write a brag document: https://jvns.ca/blog/brag-documents/
- Google eng-practices — design docs / the RFC review culture (industry design-doc guide): https://www.industrialempathy.com/posts/design-docs-at-google/
- NVIDIA — MIG user guide (partitioning tradeoffs behind the Alternatives table): https://docs.nvidia.com/datacenter/tesla/mig-user-guide/
- Your module artifacts: `dashboard-is-lying` (05), `cost-per-million-tokens` (07), `FOCUS gpu-cost synthesis` (11), `topology teardown` (02b).
