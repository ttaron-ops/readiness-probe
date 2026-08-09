# Enrichment build spec — the daily deepening routine

> **You are the daily enrichment session.** A Routine fires you once a day. Your job: take the
> **next 2 modules** from [`PROGRESS.md`](PROGRESS.md) (in order) and **rewrite their lessons in
> place** to true deep-learning depth, then commit and **push to `main`**. This file is the
> contract. Follow it exactly. Do not improvise the process.

## Mission & standing context

- **Repo:** `ttaron-ops/readiness-probe`. **Default branch: `main`** — clone it, work on it,
  push to it. This repo is the learner's single source of study; the structure is already built
  and liked — you are **deepening content, not restructuring**.
- **The learner:** a senior platform/infra engineer targeting **Senior/Staff Platform Engineer
  roles at GPU-fleet operators**. Strong on general platform engineering; deepening GPU-specific
  and staff-level depth. Calibration notes already live in each module's README ("what we skip") —
  **honor them**: never re-teach what a module says the learner already knows.
- **Cadence:** exactly **2 modules per run** (the last run may do 1). ~9 runs to finish all 17.

## The bar (what "rich enough for deep learning" means)

Every enriched lesson must let a motivated engineer **learn the topic deeply from the page plus
its linked sources alone**. Concretely, each lesson must:

1. **Teach from first principles in clear language** — no unexplained jargon; define a term the
   first time it appears; short sentences over dense clauses. Assume intelligence, not prior
   knowledge of *this specific* topic.
2. **Cover the topic from multiple perspectives** — e.g. the developer's view, the operator's
   view, the hardware/kernel view, and the economics view; or theory → practice → failure-mode.
   State each perspective explicitly.
3. **Ground it in real-world use cases with links** — 2–4 concrete production stories, each with a
   **link to a real engineering blog / talk / postmortem** (Meta, CoreWeave, Cloudflare, Netflix,
   Uber, Google, Discord, Anthropic, vendor eng blogs, etc.) describing *this topic in production*,
   with a one-line note on what the case shows.
4. **Keep the chain unbroken** — open with a **"Where this fits"** bridge (what the previous lesson
   established, what this one adds) and close with a **"Connections / what's next"** bridge (how it
   links to sibling lessons and the next one). The learner should feel a continuous thread, never a
   cold start.
5. **Be substantially deeper than the current version** — typically **several times longer**, but
   *quality and clarity over line count*; never pad. Every paragraph must add signal.

## The enriched lesson template (rewrite each lesson to this)

Keep the existing frontmatter keys; update `est_time` upward to reflect the new depth and add
`sources:` (count of real references) and `prev`/`next` (relative links to the adjacent lessons).

```
---
lesson: <existing id>
title: "<existing or improved title>"
module: <existing>
concept: "<short tag>"
status: not-started
est_time: "<updated, realistic>"
prev: "<../…/NN-prev.md or null>"
next: "<../…/NN-next.md or null>"
artifacts: [<existing>]
sources: <N>
---

# <title>

## Where this fits
<2–4 sentences: recap what the previous lesson established, name the gap this lesson fills,
and preview what it unlocks. This is the chain — never skip it.>

## Why this matters
<The real job/interview stake, and the concrete cost of not knowing it. Reference specific
company signals where apt.>

## What's new here (calibration)
<Honor the module README's skip-list: 2–4 bullets naming what the learner already knows and we
pass over, vs the genuinely new depth this lesson adds. Keep for the calibrated modules.>

## Core concepts
<The deep technical spine, built from first principles in clear language. Use subheadings,
tables, precise numbers, formulas, and diagrams-as-text where they help. This is the bulk and
must be genuinely deep — the section a learner studies from.>

## Perspectives
<The same topic from 3–4 explicit angles (developer / operator / hardware / economics, or
theory / practice / failure-mode). Each angle is a short titled paragraph. Required.>

## Real-world use cases
<2–4 production stories, each a titled bullet with a real blog/talk/postmortem LINK and a
one-line "what it shows". These must be real, fetched URLs — not invented. Required.>

## Worked example
<One concrete thing worked end to end: a calc, a design, a trace, a lab — with real-ish numbers.>

## Practice
<Hands-on tasks that feed the module's deliverable (link it). Keep it buildable/affordable.>

## Common pitfalls
<3–5 misconceptions or traps people hit on this topic, each with the correction.>

## Self-check
- <question> **Answer:** <answer>
- <question> **Answer:** <answer>
- <question> **Answer:** <answer>
  (add a 4th/5th where the topic warrants; always ≥3, always with **Answer:**)

## Connections & what's next
<How this lesson links to other lessons/modules (the web of the course), then one line naming
the next lesson and the thread that carries into it.>

## References & further reading
Group into three labeled buckets, every URL real and verified:
- **Primary sources** — papers, official docs, specs (with a one-line "read for X").
- **Real-world engineering blogs** — the production-use-case links (with "what it shows").
- **Deeper dives** — talks, books, long-reads for going further.
```

Sections are **additive to the current template** (which had Why/Core/Worked/Practice/Self-check/
Resources). If a module's lessons use the 6-section platform format, expand them to this same
template. Every lesson ends with ≥3 `**Answer:**` lines.

## Structural rules

- **Rewrite in place** — same file paths. Do **not** delete lessons.
- You **may add** a lesson when a topic is too big for one page (deep learning > cramming). If you
  do: create `NN-<slug>.md` at the right position, and **update the module README's lesson table
  and any `prev`/`next` links** so the chain stays intact.
- Update the module **README** — refresh the lesson table (titles, any new rows), bump the effort
  estimate, and keep every deliverable/checkpoint link working.
- Leave each module's **`practice/` deliverable spec and `checkpoint.md`** intact unless the new
  depth makes them wrong; if so, update them to match — never leave a dangling link.
- **Do not touch modules already marked done** in PROGRESS.md unless PROGRESS flags one for a
  fix-up.

## Sourcing rules

- Every link must be **real and verified** — use WebSearch/WebFetch to find and confirm URLs
  before citing them. If the environment's proxy blocks a fetch, still cite the canonical URL but
  note it wasn't fetched; **never invent a URL or a quote**.
- Prefer **primary sources** (papers, official docs) for facts and **engineering blogs/postmortems**
  for the real-world-use-case links the learner specifically wants.
- Flag any dollar/hardware/pricing figure as a **dated snapshot** (state the year).

## Workflow for each run (idempotent — safe to re-run)

1. `git fetch origin main && git checkout main && git pull` — start from the latest `main`.
2. Read [`PROGRESS.md`](PROGRESS.md); take the **first 2 modules** whose status is `pending`
   (top-to-bottom = learning order). If none are pending, do the **QA pass** (below) instead.
3. For **each** of the 2 modules, in order:
   a. Read the module README (honor its skip-list) + all its current lessons + its checkpoint +
      deliverable, so the enriched chain is coherent with what exists.
   b. **Deep-research** the module: spawn a research subagent (general-purpose) to gather the
      staff-depth material, the multi-perspective angles, and the **real production-blog links**
      for every lesson. Wait for its brief.
   c. **Enrich every lesson** to the template above. Use **parallel writer subagents** (≈2 lessons
      each, writing directly to the lesson file paths) to keep it fast — this is the proven
      pipeline. Ensure the `prev`/`next` bridges across the whole module form one unbroken thread.
   d. Refresh the module README (+ checkpoint/deliverable if depth changed them).
   e. **Validate on disk:** each lesson has all template sections and ≥3 `**Answer:**`; every link
      target exists; no invented URLs. Fix before committing.
   f. **Commit** that module alone: `Enrich Module <X> to deep-learning depth` with the standard
      footer (see below).
4. After both modules: mark them **`done`** in PROGRESS.md with today's date; commit PROGRESS.
5. **Push to `main`:** `git push origin HEAD:main` with exponential-backoff retry (2s,4s,8s,16s).
   Do **not** open a PR — the learner wants changes on `main` directly. (If `main` rejects a direct
   push due to protection, open a PR to `main` and enable auto-merge, then report it.)
6. The Routine sends the learner a completion summary automatically; keep your final message a
   crisp 3–5 line recap (which 2 modules, headline improvements, links to the 2 commits).

## QA pass (when all modules are `done`)

Pick the **oldest-enriched** module per run and do a consistency sweep: verify every `prev`/`next`
link resolves, every reference URL still works, and the chain reads continuously across modules;
fix and commit. Note in your summary that enrichment is complete and the routine can be paused.

## Commit footer (every commit)

```
Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01WPKJooVarwEAdBZgSAuiwz
```

## Guardrails

- One session = **2 modules**, no more (protects quality and cost). If you can't finish 2, finish
  and commit what you completed, leave the rest `pending`, and say so — the next run continues.
- **Validate before every commit.** Never commit a lesson missing template sections, missing
  answers, or with a broken link.
- Keep the learner's calibration sacred: **deepen, don't pad, and don't re-teach the basics** each
  module already says they know.
