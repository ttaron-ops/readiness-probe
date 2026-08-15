---
lesson: 03
title: "Portfolio & writeup build: the repo, the design doc, the diagram"
module: 12
concept: "evidence a manager reads cold"
status: not-started
est_time: "6 hrs"
prev: "02-capstone-synthesis.md"
next: "04-flagship-blog-demo.md"
artifacts: ["a published capstone repo README (whole-fleet architecture diagram + value-chain narrative + chapter links), one 5-8 page RFC-style architecture design doc, and a brag-doc one-pager"]
sources: 9
---

# Portfolio & writeup build: the repo, the design doc, the diagram

[🎓 12 — Capstone & interview preparation](../README.md)

## Where this fits

Lesson 02 gave you the *narrative*: the four-verb thesis (observable, attributable, schedulable, survivable), the value-chain mapping of all eleven prior artifacts, and a rehearsed 30-second spoken opener that leads with "your GPU dashboard is lying" and closes on failure honesty. That narrative lives in your head and in a storyboard — it converts nobody by itself. This lesson turns it into **published evidence a hiring manager can read cold**: a repo README, an RFC-style design doc, and a brag-doc, none of which require you in the room to land. The gap 03 fills is exactly that translation — narrative to artifact — and everything downstream (the flagship post in lesson 04, the design-drill reps in lesson 05, the narration practice in lesson 07) assumes this published evidence already exists.

## Why this matters

Lesson 02 built the *narrative* — the arc that ties artifacts 01-11 into one platform story you can tell out loud. This lesson turns that narrative into **published evidence a hiring manager can read cold**, with no meeting, no context from you, in the four minutes they spend on your GitHub before deciding whether to route you to a Staff loop. The narrative in your head converts nobody. A repo that opens with one architecture diagram and a value-chain sentence, a design doc that reasons in Goals/Non-goals/Alternatives/Risks, and a one-page quantified brag-doc — those convert. The delta between "I built a bunch of GPU tooling" and "here is a coherent GPU platform, here is why each decision, here is the number" is exactly the Staff signal.

The specific trap for a strong IC is *exhaustiveness*. You have eleven artifacts and the instinct is to list all eleven with equal weight. That reads as a portfolio of exercises, not a platform. Staff-level packaging is ruthlessly *narrative and selective*: three-to-five standout pieces carrying the story, at least one at production scale, each with a metric and a post-launch evolution, and the rest demoted to a chapter index. The reviewer is not grading completeness; they are looking for judgment — what you chose to build, what you chose *not* to (non-goals), and whether you can defend a tradeoff in writing the way you'd defend it in a design review.

The GPU-infra-specific reason this matters: your generic platform-eng credibility is assumed at this level, so the writeup has to surface the parts a generic SRE could *not* have written. The time-slicing attribution hole from module 11, the util-vs-goodput gap from module 05, MIG-vs-time-slice-vs-passthrough tradeoffs, topology-aware scheduling — these are the load-bearing content. If your README and design doc could have been produced by someone who has never operated a GPU fleet, they have failed regardless of how polished they are.

There is also a documented, company-level reason this specific artifact shape works: real infrastructure organizations build their hiring and promotion signal around exactly this format. The Pragmatic Engineer's survey of RFC/design-doc cultures at Stripe, Cloudflare, Oxide, and the Rust project shows the same skeleton — Goals, Alternatives, Risks — recurring across very different companies because it is the format a review board actually uses to evaluate judgment, not prose quality. When you write your design doc in this voice, you are not performing a genre exercise; you are producing the same document type a Staff-level design review at a GPU-fleet operator would expect to see, and doing it before anyone asked.

## What's new here

- **You already know** how to write clean READMEs, structure a repo, and draw a boxes-and-arrows diagram. **Skip** the mechanics of Markdown and Mermaid. The new thing is the *editorial* discipline: front-door diagram + value-chain narrative + selective demotion of the rest.
- **You already know** how to write a design doc at your current job. **Skip** learning RFC structure from scratch. The new thing is what **staff-level GPU content** looks like inside each section — specifically Alternatives (the MIG/time-slice/passthrough table) and Risks (the time-slicing attribution hole).
- **New angle 1:** the **portfolio credibility rubric** — 3-5 standout > exhaustive, ≥1 production-scale, each entry = problem → role/ownership → architecture → stack → **metric** → post-launch evolution.
- **New angle 2:** the **brag-doc** as a distinct artifact from the resume — a quantified-outcomes sheet you mine for interview stories and promo packets, not a narrative CV.
- **New angle 3:** publishing the README as a **shareable page/artifact** so the front door is one link, not a git clone.
- **New angle 4:** grounding your design-doc voice and your "publish the design doc" instinct in how a real company actually does it in public — Oxide Computer's live RFD corpus — instead of inventing the genre from first principles.

## Core concepts

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
| **Alternatives considered** | (omitted) | The **MIG vs time-slice vs passthrough** tradeoff table (below), *plus a stated verdict*. Alternatives is the section that most separates Staff from Senior. |
| **Risks** | "It might break" | The **time-slicing attribution hole** (module 11): time-sliced GPUs report shared util that cannot be cleanly attributed per-tenant, so cost attribution silently degrades. Name it, quantify the error, show how you derived the number, state the mitigation. |
| **Rollout** | "Deploy it" | Phased: shadow mode → one namespace → fleet, with the metric that gates each phase and the rollback trigger. |

**The Alternatives table (partitioning a GPU):**

| Option | Isolation | Attribution cleanliness | Utilization | When |
|---|---|---|---|---|
| **MIG** | Hard (separate SMs, memory, cache) | Clean — each instance is a distinct billable unit | Rigid partitions; can strand capacity | Multi-tenant, need isolation + defensible chargeback |
| **Time-slicing** | Soft (context-switched, shared memory) | **Broken** — shared util can't be split per-tenant (the module-11 hole) | High if bursty | Trusted/bursty workloads where isolation and per-tenant $ don't matter |
| **Passthrough (full GPU)** | Total (whole device to one pod) | Trivially clean (1 GPU = 1 tenant) | Poor if the workload can't fill it | Large single-tenant training/serving jobs |

The Risks section then reaches back into this table: *because* v1 uses time-slicing for the dev-namespace tier, attribution there is best-effort and cost is allocated proportionally-by-request, not measured — a known, bounded inaccuracy, stated up front. A reviewer who reads only the Alternatives table and the Risks paragraph should be able to reconstruct your entire chargeback strategy without reading the rest of the doc — that compression is what "staff-level design doc" actually means in practice.

An Alternatives table on its own is not staff signal — a table with three rows and no stated choice reads as a literature review, not a decision. Every alternatives table needs a closing sentence naming the option you took and the one line of "why this one, given the constraints in Context." Without that sentence, a reviewer cannot tell whether you understood the tradeoffs or just enumerated them.

### The portfolio credibility rubric

From staff-portfolio guidance — apply it to every entry you keep:

- **3-5 standout projects beat an exhaustive list.** Pick the ones that carry the value-chain story. Demote the rest to the chapter index.
- **At least one must be production-scale** (or convincingly production-shaped): real fleet numbers, failure handling, rollout thinking — not a toy.
- **Each standout entry follows the spine:** problem → your role/ownership → architecture → stack → **metric** → post-launch evolution. The metric is non-negotiable; "reduced unattributed GPU spend from 34% to <5%" beats "built an attribution system."
- **Show the reasoning, not just the result:** include the design doc, the diagram, and at least one tradeoff you navigated (why Kueue over raw priority classes, why DCGM over nvidia-smi scraping).

### The brag-doc (jvns)

A **one-page quantified-outcomes sheet**, distinct from the resume. Julia Evans' brag-document is a running log of what you did and the impact, mined later for reviews, promo packets, and interview stories. For this portfolio it is the raw material behind the metrics: bullet lines of the form *"shipped X → moved metric Y from A to B → which unblocked Z."* You do not publish the brag-doc; you publish its distilled metrics in the README and pull its stories in the interview loop. Keeping it current is the cheapest way to never again scramble for "a time you had impact."

### Publishing design docs in public — the Oxide precedent

Most engineers only ever see design docs behind a corporate wiki, which makes "publish your design doc" feel like an invented exercise. It isn't. Oxide Computer runs its entire engineering organization on public "RFDs" (Requests for Discussion) — RFD 1 lays out the format, and the live corpus at `rfd.shared.oxide.computer` now holds several hundred real documents spanning hardware, software, and org design, all publicly readable. That is a real company's actual public design-doc corpus, and it is the closest thing available to a worked answer key for "what does a shareable, staff-caliber design doc look like when a stranger opens it cold." Two structural habits worth lifting directly: Oxide RFDs are numbered and dated so the reader can see the doc evolve over time (mirrors your Rollout section's phased gates), and they are written to be understood by someone outside the immediate team — the same audience assumption your capstone README has to make.

### FOCUS-alignment claims need a real anchor

If your design doc or README claims the cost pipeline is "FOCUS-aligned" (module 11's economics work), that claim has a real, checkable referent: the FinOps Foundation's FOCUS specification, the vendor-neutral schema for cloud cost and usage data (ratified v1.4, June 2026). "FOCUS-aligned" is not a vibe — it means your exporter populates a recognizable subset of FOCUS columns (e.g. `BilledCost`, `EffectiveCost`, `ChargeCategory`, `ResourceId`) rather than a bespoke schema only your own dashboards understand. Naming the specific columns you populate, and the ones you deliberately don't (a Non-goal), is the difference between a claim a FinOps-literate reviewer trusts and one they discount.

### Drawing the architecture diagram

Draw the fleet as data flowing left-to-right, one layer per box-column:

```
nodes / GPUs  →  device-plugin / DCGM  →  operator / exporter  →  Prometheus / Grafana
                                                     ↓
                              scheduler (Kueue)  →  cost / FOCUS pipeline
```

Rules that keep it staff-legible: every arrow is a real data flow (metrics, scheduling decisions, dollars), label the arrows not just the boxes, show *where attribution happens* (the join point), and mark the one place the util-lie/goodput distinction lives. Use Mermaid so it renders inline on GitHub and in the shareable artifact page. One diagram, one screen, no scroll.

## Perspectives

**The hiring-manager's 4-minute-skim view.** A hiring manager triaging twenty candidates does not read your repo — they scan it. Their unconscious checklist is brutally fast: does the top of the README tell me what this is in one image and one sentence? Is there a number anywhere above the fold? Does the design doc have an Alternatives section, or did they only tell me what they built? Everything you write for this lesson should be optimized against that four-minute, largely-skimming reader, not against a patient colleague who will read every word — because in the real funnel, the patient colleague only shows up after the four-minute skim already said yes.

**The design-review-board view.** A Staff-level panel reading your design doc is not grading writing quality; they are pattern-matching against every design review they've sat in, and they know exactly what a document looks like when the author hasn't actually weighed alternatives — it has no Non-goals, its Alternatives section is either missing or a list with no verdict, and its Risks section is generic ("scaling might be hard"). The panel's real question is "would I trust this person to run a design review for my team," and the tell is whether the sections that are easy to skip (Non-goals, Alternatives, Risks) got real attention or got three throwaway lines.

**The finance/FinOps view.** Someone reading your economics chapter may not be an engineer at all — they may be a finance partner or a FinOps lead evaluating whether your attribution approach is trustworthy enough to build a budget on. To that reader, "the time-slicing attribution hole" is not a technical footnote; it's the difference between a number they can put in a board deck and a number they can't. Stating the error bound explicitly (and how you derived it) is what lets a non-engineer stakeholder actually rely on your system instead of quietly distrusting it.

**The portfolio-as-product view.** Treat the README itself as a shipped artifact with a front door, not as documentation of one. A product has a first five seconds that decide whether the visitor stays; your README's diagram-plus-value-chain-sentence *is* that first five seconds. Applying product thinking to the portfolio — what's the one action I want the reader to take next (click the flagship post, open the design doc), what's the one thing they must understand before anything else (the value chain) — is what makes the difference between a repo that reads as a code dump and one that reads as a platform someone deliberately designed, including its own presentation.

## Real-world use cases

- **Oxide Computer's public RFD corpus** — https://oxide.computer/blog/rfd-1-requests-for-discussion and the live site https://rfd.shared.oxide.computer/rfd/0001 — what it shows: a real infrastructure company running its entire design process in public, with hundreds of numbered, dated RFDs anyone can read cold — the direct model for "publish the design doc as a shareable artifact."
- **The Pragmatic Engineer's survey of RFC/design-doc cultures** — https://blog.pragmaticengineer.com/rfcs-and-design-docs/ — what it shows: Stripe, Cloudflare, Oxide, and the Rust project all converge on the same Goals/Alternatives/Risks skeleton, which is direct evidence the template in this lesson isn't an invented exercise — it's the actual format real engineering orgs use to evaluate judgment.
- **Google's internal design-doc practice** — https://www.industrialempathy.com/posts/design-docs-at-google/ — what it shows: how one of the largest engineering orgs in the world formalizes the same section list (including why Alternatives and Non-goals exist) as a lightweight, mandatory artifact before code gets written.
- **FinOps Foundation's FOCUS specification** — https://focus.finops.org/ and https://focus.finops.org/focus-specification/ — what it shows: the actual vendor-neutral cost-and-usage schema your "FOCUS-aligned" claim has to point at — useful both as a primary source to cite and as a checklist for which columns your exporter should populate.

## Worked example

A reviewer opens your capstone repo cold. Top of README: a Mermaid diagram showing H100 nodes → DCGM/device-plugin → your operator+exporter → Prometheus/Grafana, with a branch to Kueue and down to a FOCUS cost pipeline; the attribution join is circled where exporter output meets the rate table. Under it, one line: *"A GPU platform read as a value chain: attribution → scheduling → observability → economics → lifecycle → substrate."* Then a six-row table linking each stage to its artifacts.

They click the design doc. Goals: *">95% of GPU-hours attributed to a namespace within one billing day; goodput surfaced alongside util; <1% exporter overhead."* Non-goals cut multi-cloud FOCUS and per-kernel profiling out of v1. Alternatives has the MIG/time-slice/passthrough table, closing with: *"Chosen: MIG for the shared-tenant tier (clean attribution matters more than density here); time-slicing kept only for the dev tier where isolation and per-tenant $ don't matter."* Risks opens with: *"Time-sliced GPUs in the dev tier cannot be cleanly attributed per-tenant (shared util); v1 allocates their cost proportionally by request count. Derivation: dev-tier request-count variance vs. measured GPU-seconds on a 2-week sample puts the error at roughly ±15%, revisited when MIG lands."* Rollout is shadow → one namespace → fleet, gated on attribution coverage crossing 95%.

In four minutes the reviewer has learned: this person operates GPU fleets (not generic k8s), reasons in tradeoffs and non-goals *and states a verdict*, knows the time-slicing attribution hole exists and quantified it with a shown derivation before being asked, and ships in phases with a rollback trigger. That is a Staff signal delivered with zero conversation. Contrast the failure mode: eleven equally-weighted repo links, no diagram, a design doc that says "deploy it" under Rollout, omits Alternatives entirely, or asserts "the error is about ±15%" with no source for the number — indistinguishable from a competent SRE who has never touched a GPU, or worse, from someone who made the number up.

## Practice

Feeds `../practice/gpu-platform-capstone/README.md`:

1. **Skim one real public design doc before drafting your own.** Read Oxide's RFD 1 (and one substantive RFD from the live corpus) or the Pragmatic Engineer survey. Note two structural choices you'll borrow (e.g. how they write Non-goals, how they date/version the doc).
2. **Write the front-door README.** One Mermaid whole-fleet diagram at the top, the value-chain sentence, and the six-row stage→artifacts table linking your real 01-11 chapters. Demote everything below the top 3-5 into the index.
3. **Draft the design-doc skeleton** with all seven sections (Goals / Non-goals / Context / Design / Alternatives / Risks / Rollout). Fully write Alternatives (the MIG/time-slice/passthrough table, with a stated verdict) and Risks (the time-slicing attribution hole with a quantified error, its derivation, and a mitigation). Leave the rest as bulleted stubs to fill in lesson-by-lesson.
4. **Draw the architecture diagram** in Mermaid, arrows labeled with what flows, the attribution join point marked, the util/goodput distinction marked.
5. **Start the brag-doc** as a one-pager: for each of your top 5 artifacts, one line of problem → what you did → metric moved. Distill three of those metrics into the README.
6. **If you claim "FOCUS-aligned" anywhere,** list the specific FOCUS columns your exporter populates and the ones it doesn't (a Non-goal), against the FinOps FOCUS spec.
7. **Optional:** publish the README as a shareable artifact page and put the URL at the top of the repo.

## Common pitfalls

- **An Alternatives table with no stated verdict.** Listing MIG/time-slice/passthrough with their tradeoffs and then moving on is not staff signal — it reads as a literature review. Fix: end the section with one sentence naming what you chose and why, tied back to a specific Goal or constraint from Context.
- **Omitting Non-goals because "nothing was explicitly cut."** Every real v1 has scope cuts — multi-cloud, per-kernel profiling, hard chargeback, whatever you didn't build. Not naming them doesn't mean nothing was cut; it reads as not having thought about scope at all, which is worse than admitting a cut.
- **Publishing the repo without the diagram inline.** If the architecture diagram is a link to a separate file, or missing from the top of the README, the entire "4-minute skim" thesis collapses — the reader has to click away and reconstruct the system themselves, which most won't do.
- **Quantifying the time-slicing error precisely (e.g. "~±15%") without showing the derivation.** A specific-looking number with no source reads as invented, which is worse for credibility than a vaguer but honestly-derived one. Always show the sample and the method behind any error bound you state.
- **Claiming "FOCUS-aligned" as a buzzword rather than a checklist.** Saying the cost pipeline is FOCUS-aligned without naming which FOCUS columns you actually populate reads as marketing language to a FinOps-literate reviewer. Name the columns; name what you're not covering yet.

## Self-check

- Your capstone has eleven artifacts. Why is listing all eleven as equal-weight repo entries a *weaker* portfolio than featuring four? **Answer:** Eleven equal entries read as a pile of exercises and force the reviewer to do the synthesis you should have done; the rubric is judgment, not completeness. Featuring 3-5 standouts (≥1 production-scale, each with problem→role→architecture→stack→metric→evolution) and demoting the rest to a chapter index shows editorial selection and lets the value-chain narrative carry the story — which is the Staff signal.
- Which design-doc section most separates a Staff writeup from a Senior one, and what GPU-specific content belongs there? **Answer:** Alternatives considered (closely followed by Non-goals) — a Senior doc often omits it, or lists options without a verdict. The GPU-specific content is the MIG vs time-slice vs passthrough tradeoff table across isolation, attribution cleanliness, and utilization, closed with a stated "chosen X, because Y." It proves you chose your design against real options rather than defaulting.
- What is the time-slicing attribution hole, and where in the design doc does it go? **Answer:** Time-sliced GPUs share one device via context-switching, so DCGM reports shared utilization that cannot be split cleanly per-tenant — per-tenant cost attribution degrades to an estimate (module 11). It belongs in Risks: name it, quantify the error (e.g. ~±15%) *and show how you derived that number*, state the mitigation (allocate proportionally by request count in v1; move that tier to MIG for clean attribution later).
- Why is a "FOCUS-aligned" claim in your README or design doc risky if left unqualified, and how do you fix it? **Answer:** "FOCUS-aligned" is a checkable claim against the FinOps Foundation's real specification, not a vibe — an unqualified claim invites a FinOps-literate reader to discount it as marketing. Fix it by naming the specific FOCUS columns your exporter populates (e.g. `BilledCost`, `ChargeCategory`) and stating the ones you deliberately don't cover as a Non-goal.
- What is the brag-doc, and why do you not publish it directly in the README? **Answer:** It's a running, one-page log (Julia Evans' format) of what you did, the metric it moved, and what it unblocked — raw material for reviews, promo packets, and interview stories. You don't publish it as-is because it's a working log, not a polished narrative; instead you distill its strongest metrics into the README and pull its stories verbatim in the interview loop.

## Connections & what's next

This lesson is the direct output stage of lesson 02's synthesis work — the four-verb thesis becomes the value-chain sentence, the storyboard's featured chapters become the README's top 3-5, and the rejected-alternatives narration from the 30-second opener becomes the Alternatives table's stated verdict. It also reaches forward into the rest of the module: the design doc's Alternatives and Risks sections are exactly what gets drilled out loud in lesson 05's system-design prompts, and the README's chapter narration is what lesson 07 practices delivering conversationally. The published repo and design doc from this lesson are also the substrate lesson 04 builds on — the flagship blog post is drawn from the same Risks-section thinking (the util-vs-MFU gap), just aimed at a public, reach-oriented audience instead of a design-review one. Next: **lesson 04** takes this written portfolio and adds the public contrarian proof — the flagship blog post and narrated demo — that leads every application.

## References & further reading

**Primary sources**
- Oxide Computer, "RFD 1 — Requests for Discussion": https://oxide.computer/blog/rfd-1-requests-for-discussion
- Oxide's live public RFD corpus: https://rfd.shared.oxide.computer/rfd/0001
- FinOps Foundation, FOCUS specification: https://focus.finops.org/ and https://focus.finops.org/focus-specification/
- NVIDIA — MIG user guide (partitioning tradeoffs behind the Alternatives table): https://docs.nvidia.com/datacenter/tesla/mig-user-guide/

**Real-world engineering blogs**
- The Pragmatic Engineer, "Companies Using RFCs or Design Docs and Examples of These": https://blog.pragmaticengineer.com/rfcs-and-design-docs/
- Julia Evans (jvns) — "Get your work recognized: write a brag document": https://jvns.ca/blog/brag-documents/
- Fonzi — "How to build an engineering portfolio that gets you hired": https://fonzi.ai/blog/portfolio-for-engineer

**Deeper dives**
- Google eng-practices — "Design Docs at Google" (the RFC review culture, industry design-doc guide): https://www.industrialempathy.com/posts/design-docs-at-google/
- Your module artifacts: `dashboard-is-lying` (05), `cost-per-million-tokens` (07), `FOCUS gpu-cost synthesis` (11), `topology teardown` (02b).
