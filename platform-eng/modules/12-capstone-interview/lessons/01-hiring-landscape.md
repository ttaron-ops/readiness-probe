---
lesson: 01
title: "The GPU-infra hiring landscape (2025-2026)"
module: 12
concept: "decode real interview loops"
status: not-started
est_time: "4 hrs"
prev: null
next: "02-capstone-synthesis.md"
artifacts: ["per-company loop map + artifact-to-signal matrix + personal sequencing plan"]
sources: 12
---

# The GPU-infra hiring landscape (2025-2026)

## Where this fits

This lesson **opens** module 12 — Track C, the continuous capstone-assembly and interview-ramp
track that has been running since roughly month 4 and now takes over as the primary focus for
months 9-15. Everything before this point in the course was building: eleven modules of Go, Linux
internals, Kubernetes controllers, host topology, GPU hardware, observability, scheduling,
inference serving, distributed training, networking, bare-metal lifecycle, and cost economics.
This lesson is the pivot from *building the artifacts* to *deploying them* — it is where you stop
asking "what should I build next" and start asking "which door does what I already built open
fastest." Everything downstream in this module — the synthesis in lesson 02, the portfolio build
in lesson 03, the drills in lessons 05-09 — depends on knowing the shape of the loops you're
prepping for, which is exactly what this lesson maps.

## Why this matters

You are not interviewing for "a platform job." You are interviewing into a narrow, capital-heavy,
structurally unusual market: organisations whose primary cost line is depreciating silicon that
must be kept busy. That single economic fact reshapes every round of their hiring loop, and it is
the reason a generic platform-engineering loop and a GPU-fleet loop score different things while
using the same round names.

The cost of getting this wrong is not "you fail an interview." It is that you spend your scarcest
resource — rehearsal time — on the wrong surface. Prepare for a generic distributed-systems design
round and you will produce a competent answer about load balancers, sharding, and caches that
scores you as a strong web-platform engineer, which is precisely the box you are trying to get out
of. Prepare for a live-debugging round by memorising an architecture diagram and you will freeze,
because that round is deliberately designed to be unfakeable. Prepare a behavioural bank of your
hardest technical saves and you will be scored senior by a panel looking for staff scope.

There is a second, subtler cost. This market moves fast enough that any specific claim about a
named company's loop — round counts, whether they ask DSA, what a particular team's screen looks
like — decays within months and is usually sourced from interview-aggregator sites that cannot be
verified. **A candidate who repeats a stale, unverifiable claim about an employer's process in
front of that employer's recruiter has done themselves active damage.** So this lesson teaches the
*method* — how to read a live posting into a rubric, how to infer round intent from round name,
how to map your own evidence onto that rubric — rather than a table of company facts with a
six-month shelf life. The method survives; the table does not.

## What's new here (calibration)

- **Skip** (you already own this): generic behavioural/STAR mechanics, negotiation, resume
  formatting, the FAANG system-design rubric, DSA prep strategy. You are sharp on all of it.
- **Skip**: "what is a neocloud" — you have built against these stacks for eleven modules.
- **New**: the **screening-intent model** — every interview round exists to reduce a specific
  hiring risk, and naming the risk tells you what a good answer looks like even for a round format
  you have never seen.
- **New**: a **JD-to-rubric decoding procedure** — four sentence types in a posting, what each one
  is worth, and how to convert a posting into a scored prep checklist in about twenty minutes.
- **New**: the **artifact-to-signal matrix** built on this course's *actual* eleven artifacts, so
  each piece of evidence is pinned to the round where it converts.
- **New**: **campaign sequencing as a pipeline problem**, with the arithmetic for why the order of
  your applications matters more than their number.
- **New**: a **claims-hygiene discipline** — what you are entitled to assert about a named
  employer, and what you must downgrade to "a pattern I have seen." This is the thing that keeps
  the rest of the module honest.

## Core concepts

### 1. The unit of analysis is a screen, not a question

Beginners prepare for questions. The people running the loop are not thinking about questions;
they are thinking about **risks they must retire before signing an offer.** Each round exists to
retire one or two of them, and the format is just the cheapest available instrument for that risk.

Here are the risks that a GPU-fleet operator is actually carrying when they consider hiring you,
in roughly the order they get retired:

| # | Hiring risk | Sounds like | Retired by |
|---|---|---|---|
| R1 | *This person cannot write working code at all.* | "a quick screen" | live coding / take-home |
| R2 | *This person has read about GPU infra but never touched it.* | "domain deep-dive" | specific, mechanism-level questions with follow-ups |
| R3 | *This person cannot architect under ambiguity.* | "system design" | open prompt, deliberately underspecified |
| R4 | *This person collapses when the system is broken and the clock is running.* | "debugging" / "incident" / "live environment" | a broken box, a bad dashboard, or a described symptom |
| R5 | *This person operates one level below the title.* | "behavioural" / "values" / "final" | scope-and-influence probing |
| R6 | *This person will be a maintenance burden on the team.* | any round, silently | how you take correction mid-answer |

Two consequences follow immediately, and they are worth more than any company-specific fact.

**First: the same round name retires different risks at different employers.** A "system design"
round at a product company is usually R3. At a GPU-fleet operator it is R3 *and* R2 — they are
watching whether the GPU-specific layer (isolation model, gang scheduling, KV cache, lemon nodes,
utilisation-vs-goodput) appears without prompting. If you answer only R3, you pass the round and
get scored as domain-naive, which lands as "strong generalist, not a specialist hire." That is a
level-down, not a rejection, and it is invisible to you at the time.

**Second: you can score a round you have never seen, if you can name its risk.** Handed an
unfamiliar format — a live shell on someone's box, a code-reading exercise, a "here is our
Grafana, what's wrong" — ask yourself which risk it is retiring and answer *that*. A live shell is
R4: they want to hear hypothesis → command → expected reading → next branch. It is not a trivia
test about flags, and treating it as one is the standard way strong candidates fail it.

**The generalisable rule: narrate the risk you are retiring.** "Before I design anything I want to
pin down scale and cost, because at GPU prices the sizing decision dominates everything else" is a
sentence that explicitly retires R2 and R3 in one move, and it costs eight seconds.

### 2. The anatomy of a GPU-infra loop

Loops in this market vary in length and order, but the *set* of screens is remarkably stable,
because the risk list above is stable. Here is the composite shape, annotated with what each stage
screens for and how each one is failed. Treat this as a template to instantiate against a real
posting, not as a claim about any specific employer.

```
   THE COMPOSITE GPU-INFRA LOOP — stages, screening intent, and fail modes
  ══════════════════════════════════════════════════════════════════════════════

  ┌─ 0 · RECRUITER SCREEN ──────────────────────────────────────────────────┐
  │ SCREENS FOR : is the résumé claim real, is comp/level/location viable    │
  │ INSTRUMENT  : 20–30 min conversation, no technical depth                 │
  │ YOUR MOVE   : one sentence of positioning + one link. Nothing else.      │
  │ FAIL MODE   : narrating eleven projects. You are being routed, not       │
  │               evaluated — over-talking here routes you to the wrong loop │
  └────────────────────────────┬────────────────────────────────────────────┘
                               │  routing decision: which team, which level
                               ▼
  ┌─ 1 · TECHNICAL SCREEN (R1) ─────────────────────────────────────────────┐
  │ SCREENS FOR : can this person write and reason about code, live          │
  │ INSTRUMENT  : live coding, often practical/optimisation-flavoured; at    │
  │               some employers a classic DSA problem; sometimes a          │
  │               take-home instead                                          │
  │ YOUR MOVE   : narrate intent before syntax; state complexity and the     │
  │               real-world constraint that motivates it                    │
  │ FAIL MODE   : silence while typing. R6 is being scored here too.         │
  └────────────────────────────┬────────────────────────────────────────────┘
                               ▼
  ┌─ 2 · SYSTEM DESIGN (R3 + R2) ───────────────────────────────────────────┐
  │ SCREENS FOR : architecture under ambiguity, AND whether the GPU layer    │
  │               shows up unprompted                                        │
  │ INSTRUMENT  : one open prompt, 45–60 min, deliberately underspecified    │
  │ YOUR MOVE   : volunteer scale → cost → failure modes → SLOs in the       │
  │               first five minutes. Lesson 05 drills exactly this.         │
  │ FAIL MODE   : a correct generic design with no accelerator in it         │
  └────────────────────────────┬────────────────────────────────────────────┘
                               ▼
  ┌─ 3 · DEEP-DIVE / DEBUGGING (R2 + R4) ───────────────────────────────────┐
  │ SCREENS FOR : real operational contact with this hardware, under a clock │
  │ INSTRUMENT  : a described incident, a live shell, a screenshot of a      │
  │               dashboard, or a walk-through of YOUR artifact              │
  │ YOUR MOVE   : hypothesis → command → expected reading → branch. Say what │
  │               would falsify you. Lesson 06 drills exactly this.          │
  │ FAIL MODE   : "check the logs, check the metrics, escalate"              │
  └────────────────────────────┬────────────────────────────────────────────┘
                               ▼
  ┌─ 4 · ARTIFACT / PROJECT WALK-THROUGH (R2 + R5) ─────────────────────────┐
  │ SCREENS FOR : did YOU make the decisions, and at what scope              │
  │ INSTRUMENT  : "walk me through a project you're proud of"                │
  │ YOUR MOVE   : decision → tradeoff → number, not a build log.             │
  │               Lesson 07 installs this frame.                             │
  │ FAIL MODE   : chronological narration; no number; no rejected option     │
  └────────────────────────────┬────────────────────────────────────────────┘
                               ▼
  ┌─ 5 · BEHAVIOURAL / STAFF SIGNAL (R5 + R6) ──────────────────────────────┐
  │ SCREENS FOR : scope, ambiguity, influence without authority              │
  │ INSTRUMENT  : STAR prompts, often with a hiring manager or a director    │
  │ YOUR MOVE   : stories where the *problem selection* was yours.           │
  │               Lesson 08 builds the bank.                                 │
  │ FAIL MODE   : heroics. Technically excellent senior answers score        │
  │               senior — silently, with nobody flagging it in the room     │
  └────────────────────────────┬────────────────────────────────────────────┘
                               ▼
                        ┌──────────────┐
                        │   DEBRIEF    │  ← every round writes ONE paragraph.
                        │  (the real   │    You are hired or not on the union
                        │   decision)  │    of six paragraphs, not on any one.
                        └──────────────┘
```

**The debrief box is the part candidates never model, and it changes strategy.** No single
interviewer decides. Each writes a short note, and the notes are compared for *consistency*. This
means two things. A round you barely passed but where the note says "clearly has operated GPU
fleets" is worth more than a round you aced where the note says "strong, generic." And an
inconsistency — dazzling on design, vague on your own artifact — reads as a red flag rather than
an average, because it suggests the depth was borrowed. **Consistency across rounds beats peak
performance in any one round.** That is the argument for a single, coherent capstone narrative
(lesson 02) rather than eleven separately impressive projects.

### 3. Reading a job posting as a rubric

A posting is not marketing. Large parts of it *are* boilerplate, but the technical body is usually
written by, or heavily edited by, the hiring manager, and the same phrases reappear as scoring
lines in the debrief template. Learning to sort the sentences is a twenty-minute skill that pays
for the whole campaign.

Four sentence types, in descending order of value:

**Type A — the outcome sentence.** Names a *measurable* thing the role is accountable for:
"measurable improvements to latency, throughput and reliability," "reduce time-to-detect for fleet
hardware faults," "improve cluster utilisation." This is the single highest-value sentence in a
posting because it tells you what number the team is judged on, and therefore what number your
stories must land. If a posting has an outcome sentence, **every one of your artifact narrations
should end on a number of the same species.**

**Type B — the mechanism sentence.** Names a specific technology or activity: "write custom
controllers and operators," "build a GPU validation and performance-testing platform," "own the
Prometheus/Grafana observability stack," "lead root-cause analysis." These are direct evidence
requests. Each one maps onto exactly one artifact you have.

**Type C — the environment sentence.** Names scale, hardware, or org shape: "tens of thousands of
GPUs," "bare-metal fleet," "multi-tenant," "on-call rotation." These tell you which of your
answers need a *scale caveat* attached, and which of your simulated results need labelling as
simulated.

**Type D — boilerplate.** "Passion for excellence," "fast-paced environment," the legal block.
Zero information. Do not tune anything to it.

The decoding procedure:

```
   JD → RUBRIC → PREP PLAN  (about 20 minutes per posting)
  ═════════════════════════════════════════════════════════════════════════════

   STEP 1   Paste the posting into a file. Delete every Type-D sentence.
              → typically removes 40–60% of the text
              ↓
   STEP 2   Tag what's left:  [A] outcome   [B] mechanism   [C] environment
              ↓
   STEP 3   For each [B], write the artifact that proves it.
            If a [B] has NO artifact → that is your gap list.
              ↓
   STEP 4   For each [A], write the NUMBER you can put next to it.
            If you have no number → that story is not ready. Go find one.
              ↓
   STEP 5   For each [C], write the honesty caveat you will attach.
            ("200-node SIMULATED fleet, trace published.")
              ↓
   STEP 6   Anything left untagged and unmatched = the round you are weakest
            in. Prep that one, not the one you enjoy.
```

**Worked micro-example.** A composite Type-B sentence of the kind that recurs across this market:
*"design and build custom Kubernetes controllers and operators, and the telemetry that proves they
work."* Decode: they want (i) a controller you wrote, with a reconcile loop, finalisers and status,
(ii) evidence it was *observed*, not just deployed, and (iii) an implied claim that you can close
the loop between the two. Your matching evidence is the `gpu-cost-operator` (module 02 artifact
plus the module 04 attribution core), the exporter that feeds it (module 01 artifact), and the
reconciliation identities from `04.10` that let you say "and here is how I proved the numbers were
right." That third element is the differentiator — most candidates can produce (i) and (ii).

**A caution that is part of the method.** The specific phrasing of any one company's posting
changes, and third-party interview-aggregator sites report round structures that are often stale,
sometimes wrong, and never verifiable. Read the *live* posting on the employer's own careers page,
and treat everything else as a hypothesis. Section 8 gives the discipline for this.

### 4. Five loop archetypes, and how to tell which one you are in

Rather than a company table with a short shelf life, carry five archetypes. They differ in which
risks dominate, which is what changes your prep.

| Archetype | Dominant risks | What the loop over-weights | What it under-weights | Tell |
|---|---|---|---|---|
| **Structured incumbent** — a large, decades-old hardware or platform company | R1, R3, R5 | standardised coding rounds, many interviewers, a formal levelling rubric | domain narrative; your artifacts matter less than your live performance | scheduler-managed calendar, named levels, an explicit rubric mentioned by the recruiter |
| **Lean growth-stage operator** — a fast-growing fleet operator | R2, R4 | live/practical rounds, "have you actually operated this" | classic DSA | recruiter names a specific *team* and a specific *problem*, not a level |
| **Research lab / frontier org** | R3, R5, R6 | research-adjacent reasoning, take-homes, references, in-person time | breadth of infra trivia | longer loop, take-home present, explicit reference requests |
| **Hardware vendor** | R1, R2 | low-level depth, performance reasoning, classic algorithmic screens | product/system narrative | questions drift toward the silicon and the driver |
| **Hybrid forward-deployed** | R2, R6 | scenario handling, customer communication, and fleet depth in the same round | pure architecture depth | posting mixes "platform" language with "customer" or "on-site" language |

**How to identify the archetype from the recruiter email, before you prep anything.** Look at
three things: (i) does it name a level or a team? A level implies a formal rubric (structured
incumbent); a team implies a specific problem (lean operator). (ii) Does it schedule a "screen" or
a "conversation with the hiring manager"? The latter usually means the HM is doing their own
funnelling, which correlates with domain-heavy rounds. (iii) Does it mention a take-home or a
"practical exercise"? That is a strong signal for R4-weighted loops.

**Then ask.** Recruiters answer this question directly and it costs you nothing: *"So I can prepare
usefully — what does the loop look like, and is any of it a live/practical environment?"* You are
entitled to that information and being told is worth more than any inference. Two follow-ups worth
asking, in this order: *"Is there a coding round, and is it algorithmic or practical?"* and *"Which
team, and what is the problem they're currently working on?"* The second one gives you material
for every subsequent round.

### 5. The artifact-to-signal matrix

You finish this course with eleven module deliverables. Left alone, they are eleven projects.
Mapped to rounds, they are a rehearsed answer bank. This is the matrix; it is the thing lesson 02
threads into a single narrative and lesson 03 publishes.

| # | Artifact (module deliverable) | The one sentence it proves | Round it converts in | The number it carries |
|---|---|---|---|---|
| 01 | GPU cost **exporter** (Go) | "I write production Go against Kubernetes and NVIDIA APIs." | technical screen; deep-dive | series emitted, cardinality per node |
| 01b | Container **anatomy** teardown | "I know what a container actually is at the kernel." | deep-dive; debugging | namespaces/cgroup limits observed |
| 02 | GPU cost **operator** (CRDs, reconcile, finalisers) | "I ship controllers, not scripts." | technical screen; design (P2) | reconcile latency, envtest coverage |
| 02b | Host **topology** map | "I reason about NUMA, PCIe and NVLink placement." | deep-dive; design (P3) | measured per-GPU bandwidth by placement |
| 03 | GPU hardware **report** | "I can read a datasheet into a capacity decision." | design; economics questions | SM count, HBM BW, ridge point |
| 04 | **Per-pod attribution** exporter + failure-mode log | "I built the join nobody ships, and I proved it conserves." | design (P2); debugging; artifact walk-through | identity A holds to 1.000; exposure fraction |
| 05 | **"Your GPU dashboard is lying"** | "I know what the flagship metric actually measures." | *every* round; the opener | `GPU_UTIL` 99 vs `SM_ACTIVE` 0.16 on one GPU |
| 06 | **Kueue showback** | "I understand quota, gang scheduling and fair share." | design (P3); economics | queue wait, borrow/reclaim behaviour |
| 07 | **Cost-per-million-tokens** curve | "I can price a unit of work end to end." | design (P4); economics | $/1M tokens at batch B |
| 08 | **Survive-a-failure** training lab | "I have watched a distributed job die and recovered it." | debugging (D1); behavioural | checkpoint overhead %, recovery time |
| 09 | Fabric / **network read** | "I can turn a topology into a bandwidth number." | design (P3); deep-dive | per-GPU GB/s under a collective |
| 10 | **Capex-vs-cloud** model | "I can defend a buy-vs-rent decision with arithmetic." | economics; staff behavioural | break-even months at utilisation U |
| 11 | **FOCUS cost synthesis** + tooling teardown | "I read the incumbent tool's source and found where it breaks." | design (P2); staff behavioural | the $0.00 / 7× / N× error table |

**Three of these are load-bearing and eight are supporting.** The load-bearing three are 05
(the util lie), 04 (the attribution controller), and 11 (the tooling teardown that justifies
building 04 at all). Together they form one argument: *the industry's default GPU metric is
misleading, the industry's default cost tool inherits that plus a whole-device numerator, and here
is the controller that closes both gaps with a conservation check.* Everything else is corroborating
detail. Lesson 02 makes this explicit; lesson 03 publishes it; lesson 04 blogs it.

### 6. The evidence graph

The matrix above is a list. The thing you actually need in your head is a *graph*: which lesson
produced which finding, which finding feeds which artifact, and which artifact answers which round.
This is the picture that lets you answer "how did you find that?" three levels deep without
hesitating, and it is the picture lesson 03 renders as your repo's front-door diagram.

```
   THE EVIDENCE GRAPH — from lesson, to finding, to artifact, to round
  ══════════════════════════════════════════════════════════════════════════════

   LESSONS (where the finding was made)        FINDINGS (what you can assert)
   ────────────────────────────────────        ──────────────────────────────

   05.1 lie-and-truth ─────────────────────▶  F1  GPU_UTIL = "≥1 kernel resident",
     · field 203 = nvmlUtilization_t.gpu          NOT work done. Two independent
     · SM_ACTIVE = Σactive/Σelapsed cycles        collection paths, ~130× apart
     · dcgm-exporter ships SM_ACTIVE OFF   ──▶  F2  the honest metric is opt-in
                                                   (default-counters.csv)
                    │
                    ▼
   05.8 allocated-vs-utilised ─────────────▶  F3  ratio→hours needs sum_over_time;
     · Riemann sum, Δ = 30 s                      avg_over_time overstates by
     · six-bucket decomposition                   window ÷ time-present
     · reconciliation identity              ──▶  F4  the gap decomposes into six
                    │                              buckets with six owners
                    ▼
   04.7 time-slicing ──────────────────────▶  F5  the pod map survives sharing;
   04.10 per-pod attribution                     the metric does not (N:1)
     · Owners map[string][]Owner            ──▶  F6  Σ allocated_share
     · identity A: Σshare + unalloc ≡ 1.0           + unallocated ≡ 1.0 exactly
     · exposure fraction                    ──▶  F7  x% of chargeback $ rests on
                    │                              an estimate — quantified
                    ▼
   11.9 existing tooling limits ───────────▶  F8  OpenCost numerator = whole-GPU
     · GPUCost = GPUHours × CostPerGPUHr        REQUEST count
     · nvidia.com/gpu.shared absent from    ──▶  F9  same tool, same card:
       both numerator paths                        $2.10 / $14.70 / $0.00 / $8.40
     · vGPU scaling guard unreachable       ──▶  F10 utilisation is queried, stored,
                    │                              and never multiplied into cost
                    ▼
        ┌───────────┴────────────┬────────────────────┬──────────────────┐
        ▼                        ▼                    ▼                  ▼
   ARTIFACT 05             ARTIFACT 04           ARTIFACT 11        ARTIFACT 02
   flagship blog           the controller        teardown +         the operator
   (F1,F2,F3,F4)           (F5,F6,F7)            FOCUS schema       (packaging)
                                                 (F8,F9,F10)
        │                        │                    │                  │
        └────────────┬───────────┴─────────┬──────────┴──────────────────┘
                     ▼                     ▼
              ROUNDS IT ARMS         ROUNDS IT ARMS
              ──────────────         ──────────────
              P6 fleet observability P2 cost attribution   ← volunteer this one
              D2 100%-util trap      P1 multi-tenant
              artifact walk-through  staff behavioural
                                     debugging deep-dive
```

Read the graph left to right and you have the answer to the hardest follow-up in any deep-dive
round: *"how do you know that?"* Every assertion on the right has a lesson on the left that
produced it, with a mechanism you can state. That traceability is what separates a candidate who
read a blog post from one who did the work — and it is the entire reason this module insists the
capstone reference *your* findings rather than generic advice.

### 7. Sequencing your campaign, and the arithmetic behind it

Applications are not independent trials. Three coupling effects make order matter:

1. **You improve with reps.** Your fourth loop is materially better than your first. Interview
   performance is a rehearsed skill and the first live round is where you discover which parts of
   your narration collapse under a stranger's clock.
2. **Offers are leverage.** A live offer changes both timeline and negotiating position at every
   other employer, and it lets you compress a slow process.
3. **Cooldowns are real.** Most employers will not re-interview you for six to twelve months. A
   burned loop at your top-choice employer is not a retryable event on this timeline.

Those three facts produce one strategy, and it is not the intuitive one. **Do not lead with your
top choice.**

```
   CAMPAIGN AS A PIPELINE — order dominates volume
  ══════════════════════════════════════════════════════════════════════════════

    week 0        week 2         week 5         week 8         week 12
      │             │              │              │              │
      ▼             ▼              ▼              ▼              ▼
  ┌────────┐   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌──────────┐
  │ WAVE 0 │──▶│ WAVE 1  │───▶│ WAVE 2  │───▶│ WAVE 3  │───▶│ decision │
  │ CALIB  │   │ CONVERT │    │ TARGET  │    │ STRETCH │    │  window  │
  └────────┘   └─────────┘    └─────────┘    └─────────┘    └──────────┘
   1–2 loops    2–3 loops      2–3 loops      1–2 loops
   you would    strong fit,    your actual    hardest prep
   accept but   artifacts are  top choices    (algorithmic
   are not      the syllabus                  screens, labs)
   optimising
   for
      │             │              │              │
      │             │              │              └─ prep gated behind
      │             │              │                 kept-warm DSA reps
      │             │              └─ enter holding ≥1 offer or ≥1 clean
      │             │                 full-loop pass. Leverage + rehearsal.
      │             └─ your artifact-to-signal matrix converts hardest here;
      │                these loops double as portfolio pressure-testing
      └─ THE POINT IS THE FEEDBACK, NOT THE OFFER. Debrief yourself after
         each round against lesson 09's five dimensions and remediate.

   FEEDBACK EDGES (why this is a loop, not a list):
      wave 0 failure  ──▶ remediate against module 05/06 drills ──▶ re-enter
      wave 1 debrief  ──▶ rewrite the weakest artifact narration (lesson 07)
      any "strong but generic" note ──▶ the GPU layer is not surfacing:
                                         drill lesson 05's volunteering reflex
```

**The arithmetic for why wave 0 is not wasted time.** Suppose your per-loop pass probability starts
at p₀ and rises by δ per completed loop as narration and drills settle. With five loops run in
increasing order of importance, the probability of converting the *most important* one is
p₀ + 4δ; run in decreasing order, it is p₀. If p₀ = 0.30 and δ = 0.06 — a modest, plausible
learning rate for a rehearsed skill — the top-choice conversion probability is 0.54 versus 0.30.
**The ordering decision is worth more than doubling your application count**, and it costs nothing
but patience. (These are illustrative numbers to show the shape of the argument, not measured
rates; the point is the monotonicity, not the magnitudes.)

**One caveat that stops this becoming cynicism.** Wave 0 loops must be roles you would genuinely
accept. Interviewing at a company you would refuse wastes their time, and the market is small
enough that this is remembered. Choose wave 0 from the tail of your genuine list, not from outside
it.

### 8. Claims hygiene: what you may assert about an employer

This section exists because it is the most common way a well-prepared candidate damages
themselves, and because the rest of this module depends on the same discipline.

There are three tiers of claim, and you must know which tier you are speaking in:

**Tier 1 — verifiable primary source.** The live job posting on the employer's own site, the
employer's own engineering blog, their public documentation, their open-source repositories, their
published papers. You may quote these, and you should, because quoting them is itself a signal
that you did the reading. *"Your docs describe X; I built something that assumes X — did I read
that right?"* is a strong question.

**Tier 2 — pattern.** Something you have seen recur across several employers or several accounts
and cannot attribute to any one of them. State it as a pattern: *"In this market I've generally
seen the debugging round weighted heavily — is that true here?"* This is honest, it is useful, and
it invites the interviewer to correct you, which is a cheap way to learn the real answer.

**Tier 3 — unverifiable secondhand.** Interview-aggregator sites, forum posts, "a friend
interviewed there," a blog claiming to describe a company's loop. **Never state these as fact
about the named company.** They decay fast, they are frequently wrong, and repeating one in front
of someone who works there is a memorable error. Use them privately to generate hypotheses, then
verify with the recruiter.

```
   THE CLAIM LADDER — before you say a sentence about a named employer
  ══════════════════════════════════════════════════════════════════════════════

        Is it on their own site / docs / repo / paper / posting?
                    │                              │
                   YES                            NO
                    │                              │
                    ▼                              ▼
        ┌───────────────────────┐      Have I seen it across several
        │ TIER 1 — quote it,    │      independent employers?
        │ cite where you saw it │              │            │
        └───────────────────────┘             YES          NO
                                               │            │
                                               ▼            ▼
                              ┌────────────────────┐  ┌───────────────────────┐
                              │ TIER 2 — say "a    │  │ TIER 3 — do NOT state │
                              │ pattern I've seen" │  │ it. Ask the recruiter │
                              │ and invite         │  │ instead. Use it only  │
                              │ correction         │  │ to form a hypothesis. │
                              └────────────────────┘  └───────────────────────┘
```

**Apply the same ladder to your own claims.** "I built GPU scheduling infrastructure" is a Tier-3
statement about yourself: it cannot survive one follow-up. *"I built a scheduler plugin and
benchmarked it against the default on a 200-node **simulated** fleet — here's the trace"* is Tier 1
about yourself: accurate, checkable, and unpuncturable. The
[capstone deliverable](../practice/gpu-platform-capstone/README.md) makes labelling the simulation
a hard requirement for exactly this reason. **The discipline you apply to claims about employers
and the discipline you apply to claims about your own work are the same discipline, and
interviewers notice when only one of them is present.**

### 9. What the market is actually buying

Every round in §2 exists because of one economic fact, and being able to state it in the room is
itself a differentiator. Work the arithmetic once so you own it.

```
   WHY GPU-FLEET HIRING LOOKS THE WAY IT DOES — the arithmetic
  ══════════════════════════════════════════════════════════════════════════════

   Take a 500-GPU H100 fleet. Rate: $2.50/GPU-hour, on-demand basis,
   mid-2026 directional snapshot — provider quotes in that window spanned
   roughly $2 to $11/GPU-hour, so this is a band, not a constant.

     annual capacity cost   = 500 × 24 × 365 × $2.50   =  $10.95 M / yr

   Now the observability finding from module 05, applied to that fleet.
   Measured allocated-vs-utilised gaps on real clusters routinely land
   with more than half the allocated GPU-hours doing no SM work:

     idle share (module 05.8 worked example)          =  57.3 %
     annual cost of idle allocation                   =  $ 6.27 M / yr

   A ten-percentage-point fleet-wide utilisation improvement:

     500 × 24 × 365 × 0.10 × $2.50                    =  $ 1.10 M / yr

   ─────────────────────────────────────────────────────────────────────
   ONE engineer who moves fleet utilisation by ten points pays for a
   large team. That is the entire reason these loops exist, and it is
   why they screen for R2 (real contact) and R4 (fast diagnosis) harder
   than for R1 (can you code).
   ─────────────────────────────────────────────────────────────────────

   And the reliability side of the same coin. Meta's study of eleven
   months on two large A100 research clusters reports a measured
   mean-time-to-failure of 7.9 hours for 1,024-GPU jobs, and PROJECTS
   1.8 hours at 16,384 GPUs and 0.23 hours at 131,072
   (arXiv:2410.21680 — note the second and third figures are the paper's
   projections, not measurements).

     → at frontier scale something breaks every couple of hours
     → every minute of guessing instead of bisecting is billed
       GPU-hours across a four- or five-figure fleet
     → hence a debugging round that scores PROCESS, not just answers
```

**Say the version of this that fits the room.** To an engineer: "utilisation is the fleet's real
SLO and the default metric can't see it." To a hiring manager: "a ten-point utilisation move on a
500-GPU fleet is about a million dollars a year, which is why I built the attribution layer
first." To a director: "I'd rather be measured on goodput than on util, and I built the
measurement that makes that possible." Same fact, three registers — lesson 07 drills the
compression.

## Perspectives

**Candidate's view — optimise for artifact-conversion rate, not prestige.** The question is not
"which employer is most impressive" but "which loop turns the evidence I already have into an offer
with the least additional prep." Those two rankings produce different orderings, and sequencing on
the second is the entire point of §7. Concretely: a loop that weights R4 (live debugging) converts
your failure-mode log and survive-a-failure lab almost directly, whereas a loop that weights R1
with algorithmic screens converts none of your eleven artifacts and requires weeks of unrelated
prep. Both may be worth running; they are not worth running in the same week.

**Hiring manager's view — the debrief template is the real rubric.** A hiring manager who wrote
"measurable improvements to latency, throughput and reliability" into a posting will have a line in
the debrief form that says roughly that, and interviewers must write against it. This is why a
Type-A outcome sentence is worth more than any other sentence in the posting: it is not
aspirational, it is a scoring line. It is also why a candidate whose stories never land a number
generates a debrief note that has nothing to write in that box, regardless of how strong the
conversation felt.

**Recruiter's view — you are being routed before you are being evaluated.** The screen exists to
decide which team and which level to route you to, and over-talking there does real damage: a
sprawling answer routes you to a generalist platform team where your GPU evidence is worth less.
One positioning sentence and one link is the correct payload. *"I'm a platform engineer who
specialised into GPU fleet cost and observability — I built a per-pod GPU attribution controller
and published the measurement work behind it; here's the repo."* That sentence routes you.

**Market-economics view — thin margins on enormous fixed capital.** A fleet operator recovers
six-figure-per-node capital through utilisation over years. Every idle GPU-hour is unrecovered
capital, every slow incident is billed silicon, every mis-scheduled job is a scheduling loss with
a dollar sign. This is why the loops screen for people who already think in GPU-hours and dollars,
and why module 11's economics work converts in rounds that are nominally about architecture.

**Long-run view — the specialisation is the asset, not the employer.** The set of employers in this
market changes on a one-to-two-year timescale; the *skills* — attribution, telemetry honesty,
scheduling, unit economics, failure recovery — do not. Build the evidence for the skill, not for a
company. That is also the reason this lesson refuses to hard-code a company table: the table is a
depreciating asset and the decoding method is not.

## Real-world use cases

These are public, checkable artifacts that a GPU-infra loop actually probes. Each is something you
can read yourself, which is the point — they generate Tier-1 claims.

- **NVIDIA `dcgm-exporter`, `etc/default-counters.csv`.** `DCGM_FI_DEV_GPU_UTIL` ships enabled;
  `DCGM_FI_PROF_SM_ACTIVE` and `DCGM_FI_PROF_SM_OCCUPANCY` ship commented out. **What it shows:**
  the reason "every GPU dashboard shows the misleading metric" is not carelessness — the misleading
  metric is the default and the honest one is opt-in. This is one file, it takes thirty seconds to
  verify, and citing it converts an opinion into an observation. It is the strongest single exhibit
  in your flagship post (lesson 04).

- **NVIDIA `dcgm-exporter`'s shipped Grafana dashboard** (`grafana/dcgm-exporter-dashboard.json`,
  published as grafana.com dashboard 12239). Eight panels: temperature, average temperature, power
  usage, total power, SM clocks, **GPU Utilization**, framebuffer memory used, and tensor-core
  utilisation. No `SM_ACTIVE` panel, no `SM_OCCUPANCY` panel, no `DRAM_ACTIVE` panel. **What it
  shows:** the most widely installed GPU dashboard in existence leads with the presence metric.
  "Here's the lie everyone ships, straight from the vendor" is a sentence you can defend.

- **OpenCost issue #3900, "GPU 'usage' cost basis does not reflect actual GPU utilization"**
  (filed 2026-07-05, closed 2026-07-09). The reporter shows `costBasis=usage` and
  `costBasis=allocation` returning figures within ~1% of each other for a vLLM server measured at
  roughly 11% GPU utilisation — expected ≈$0.275 on a 1-GPU, 1-hour, $2.50/hr workload, actual
  ≈$2.50 — and traces it to `GPUHours` being set from the *request*. **What it shows:** your
  central cost argument stated in the maintainers' own tracker, and fixed four days later. Citing
  an issue the project agreed with is far stronger than criticising the project.

- **OpenCost issue #3828, "Support HAMi GPU core and memory allocation in cost calculations"**
  (filed 2026-06-02, open at last reading). A HAMi user reports that a pod allocated a quarter of a
  card is billed as a whole card, because only `nvidia.com/gpu` is read; their workaround is to
  multiply `gpuCost` by the intended fraction outside the tool. **What it shows:** a stranger
  filing a feature request for the exact correction layer this course's deliverable implements.
  That is external validation of your thesis, unsolicited.

- **Meta, *Revisiting Reliability in Large-Scale Machine Learning Research Clusters*
  (arXiv:2410.21680, HPCA 2025).** Eleven months of operations across two clusters (≈16K and ≈8K
  A100 GPUs). Measured MTTF of **7.9 hours for 1,024-GPU jobs**, with **1.8 hours at 16,384 GPUs
  and 0.23 hours at 131,072 GPUs given as projections**. **What it shows:** why the debugging round
  carries so much weight, and — equally usefully — a lesson in citation precision. Repeating the
  projected figures as measurements is exactly the kind of error a careful interviewer catches.

- **Meta, *The Llama 3 Herd of Models* (arXiv:2407.21783).** Llama 3.1 pretraining on 16,384 H100s
  ran at roughly 38–43% model-FLOPs utilisation. **What it shows:** the calibration point that
  makes all your utilisation numbers meaningful. If a team with that budget tops out below half of
  theoretical peak, a serving fleet at 1% tensor activity is not "slightly inefficient."

## Worked example

**The scenario.** A posting lands for a senior platform role at a GPU-fleet operator. You have
twenty minutes before deciding whether to apply, and you want to leave with a prep plan rather than
a feeling. Here is the decode, end to end, on a composite posting assembled from language that
recurs across this market. (Composite, deliberately — the method is what transfers; run it against
a live posting yourself.)

**Step 1 — the posting, after deleting Type-D boilerplate.**

```
  [B] Design, build and operate custom Kubernetes controllers and operators
      that manage GPU capacity across a multi-tenant fleet.
  [B] Own the observability stack (Prometheus, Grafana, DCGM) that proves
      the platform is behaving.
  [A] Deliver measurable improvements to utilisation, job throughput and
      time-to-detect for hardware faults.
  [C] Environment: tens of thousands of GPUs across multiple regions,
      bare metal, mixed training and inference tenants.
  [B] Lead root-cause analysis for platform incidents and drive the
      corrective actions.
  [C] Participate in an on-call rotation.
```

**Step 2 — map each [B] to an artifact, each [A] to a number, each [C] to a caveat.**

| Line | Type | Your evidence | The number you attach | Gap? |
|---|---|---|---|---|
| custom controllers/operators | B | `gpu-cost-operator` (02) + per-pod attribution core (04.10) | reconcile interval 10 s; identity A holds at 1.000 | none |
| observability stack | B | dashboard-is-lying exhibit (05) + query pack (05.8) | `GPU_UTIL` 99 vs `SM_ACTIVE` 0.16, same GPU, same second | none |
| measurable utilisation improvement | A | allocated-vs-utilised decomposition (05.8) | 57.3% of allocated GPU-hours idle; $41.7k/yr recovered from one namespace at a conservative 0.11→0.40 `SM_ACTIVE` move | none |
| time-to-detect hardware faults | A | XID/health work (05.5) + survive-a-failure (08) | detection latency for a synthetic fault | **thin** — needs a measured number |
| multi-tenant, tens of thousands of GPUs | C | fake GPU fleet lab (04) | "200-node **simulated** fleet, trace published" | caveat required |
| lead RCA | B | failure-mode log (04.10) — five real entries | MTTR on the entries you actually debugged | none |
| on-call | C | — | — | be ready to say you want it |

**Step 3 — what the table tells you to do.** One genuine gap: you have no measured
*time-to-detect* number. That is a two-hour fix — inject a synthetic XID or a health failure into
the lab, measure the latency from injection to alert firing, and write it down. It is worth doing
before you apply, because it is a Type-A line and Type-A lines are debrief scoring lines.

**Step 4 — the round-by-round prep plan that falls out.**

```
   PREP PLAN GENERATED FROM ONE POSTING
  ══════════════════════════════════════════════════════════════════════════════

   Technical screen (R1)
     → practical Go, likely touching k8s client-go or a data-shaping problem
     → warm up on the exporter's code; be able to explain the informer
       pattern and why LOOP A and LOOP B are separate loops (04.10 §2)

   System design (R3+R2)
     → the [C] line ("multi-tenant", "mixed training and inference") points
       straight at P1. The [B] observability line points at P6.
     → VOLUNTEER P2 (cost attribution) regardless of the prompt — it is
       your home field and the [A] line asks for it in dollars

   Deep-dive / debugging (R2+R4)
     → "time-to-detect for hardware faults" + "lead RCA" = D3 (lemon node
       that passes idle health checks) is the likeliest shape
     → have the failure-mode log open in your own head, not on screen

   Artifact walk-through (R2+R5)
     → lead with 05 (the lie), pivot to 04 (the controller), close on 11
       (why the incumbent tool couldn't do it)
     → 3 minutes, one number per artifact

   Behavioural (R5+R6)
     → the [B] "lead RCA and drive corrective actions" line is a scope
       probe. Bring the story where you drove a fix across teams, not the
       one where you personally fixed it fastest.
```

**Step 5 — the positioning sentence for the recruiter screen**, derived from the same table:

> *"I'm a platform engineer who specialised into GPU fleet cost and observability. The short
> version: the default GPU utilisation metric measures kernel residency, not work, so most fleets
> can't see their own waste — I built the per-pod attribution controller that closes that gap, with
> a conservation check that proves the numbers add up, and published the measurement work behind
> it. Most of it was validated on a simulated 200-node fleet with the traces published."*

Forty-five seconds. It contains a mechanism (kernel residency vs work), an artifact (the
controller), a rigour signal (conservation check), and an honesty signal (simulated, traces
published). **Every one of those four is a scoring line somewhere in the loop.**

## Practice

Feed all of this into the capstone deliverable at
[GPU platform capstone](../practice/gpu-platform-capstone/README.md).

1. **Decode three live postings.** Pick three real, currently-open postings from employers' own
   careers pages. Run §3's six-step procedure on each. Produce, per posting: the tagged text, the
   artifact map, the number map, and the caveat list. Time yourself — the target is twenty minutes
   per posting by the third one.

2. **Build your gap list and close the cheapest gap.** Union the "Gap?" column across your three
   decodes. Rank by cost-to-close. Close the cheapest one this week and record the number it
   produced. A gap you closed is worth more in the room than a gap you noticed.

3. **Build your artifact-to-signal matrix.** Reproduce §5's table with *your* eleven artifacts,
   *your* measured numbers, and the round each converts in. Any row whose "number" cell is empty is
   an artifact that is not interview-ready; either measure something or demote it to the chapter
   index in your repo.

4. **Draw your own evidence graph.** Reproduce §6's diagram from your own lessons and findings. The
   discipline is that every arrow must be one you can defend: if you cannot say *how* a finding was
   established, delete the node.

5. **Write your sequencing plan.** Name wave 0 (1–2 loops you would accept but are not optimising
   for), wave 1 (2–3 where your artifacts convert hardest), wave 2 (your actual targets), wave 3
   (anything needing weeks of separate prep). Write the calendar dates. Write the entry condition
   for wave 2 — "≥1 offer in hand, or ≥1 clean full-loop pass."

6. **Write your three recruiter questions** and rehearse asking them: loop shape, coding-round
   flavour, team and current problem. Then write your forty-five-second positioning sentence and
   say it out loud until it does not sound rehearsed.

7. **Run the claim ladder over your own README.** Every sentence in your capstone front door that
   describes what you built: which tier is it? Rewrite every Tier-3 sentence about yourself into a
   Tier-1 one by adding the scale, the environment, and the caveat.

**Acceptance:** three decoded postings with artifact/number/caveat maps · a gap list with one gap
closed and its number recorded · an artifact-to-signal matrix with no empty number cells for your
top five artifacts · an evidence graph you can narrate · a dated sequencing plan with an explicit
wave-2 entry condition · a positioning sentence under fifty seconds.

## Common pitfalls

1. **Treating "no algorithmic screen" as "lower bar."** A live-environment round is *harder* to
   fake than a data-structures problem: you cannot memorise your way through a real shell on a real
   box with a real fault. **Mechanism:** R4 is retired by observing your diagnostic process under
   time pressure, and process is unrehearsable in the abstract — it only comes from having actually
   debugged this class of system. **Symptom:** you prepared by reading, freeze at the prompt, and
   fall back to "check the logs."

2. **Leading with your top choice.** **Mechanism:** per-loop pass probability rises with reps
   (§7), cooldowns run six to twelve months, and offers are leverage — so spending your least
   rehearsed performance on your least retryable opportunity is strictly dominated. **Symptom:**
   you interview well in week ten at a company you no longer need.

3. **Repeating an unverifiable company claim.** **Mechanism:** interview-aggregator content is
   secondhand, undated, and frequently describes a process two reorganisations old. **Symptom:**
   you say "I hear you don't do algorithmic screens" to someone who runs one, and the note in your
   debrief is about judgement, not about the mistake.

4. **Answering only the round's nominal question.** **Mechanism:** most rounds retire two risks,
   and the second one (usually R2 — real domain contact) is scored silently. **Symptom:** a clean
   system-design answer and a debrief that says "strong generalist," which is a level-down that
   nobody tells you about.

5. **A portfolio of eleven equal-weight projects.** **Mechanism:** the debrief compares notes for
   *consistency*; eleven equal items give the reader no thesis to carry between rounds, so each
   interviewer forms a different impression and the union reads as scattered. **Symptom:** "did a
   lot of interesting things" as the summary line. **Fix:** three load-bearing artifacts, eight
   supporting — lesson 02.

6. **Unlabelled simulation.** **Mechanism:** one follow-up question ("how many nodes was that, and
   whose?") exposes it, and the recovery costs you the rest of the round. **Symptom:** you say "I
   built GPU scheduling infrastructure," get asked about the fleet, and start qualifying. **Fix:**
   label the simulation *first*, every time; it is strictly stronger.

7. **Numbers without provenance or date.** **Mechanism:** GPU rates moved by more than 2× across
   2025-2026 depending on basis (on-demand vs reserved vs amortised capex), so an undated
   "$X/GPU-hour" is your most challengeable sentence. **Symptom:** a finance-literate interviewer
   asks "on what basis?" and your whole cost argument wobbles. **Fix:** *"$2.50/GPU-hour,
   1-year-reserved basis, February 2026 rate card"* — basis and date, always.

8. **Optimising the campaign for volume.** **Mechanism:** each loop consumes several days of
   rehearsal-quality attention; twelve simultaneous loops means twelve under-rehearsed
   performances. **Symptom:** you are "in process" everywhere and converting nowhere.

## Self-check

- **What is a "screen" screening for, and why does that matter more than the question set?**
  *Answer:* every round exists to retire a specific hiring risk — R1 can they code, R2 have they
  actually operated this hardware, R3 can they architect under ambiguity, R4 do they hold up
  diagnosing a broken system on a clock, R5 do they operate at the level being hired, R6 are they
  coachable. Naming the risk lets you answer a format you have never seen: a live shell is R4, so
  narrate hypothesis → command → expected reading → branch, rather than reciting flags. It also
  explains the commonest silent failure — most rounds retire two risks, and the second (usually R2)
  is scored without being asked about.

- **How do you turn a job posting into a prep plan in twenty minutes?** *Answer:* delete Type-D
  boilerplate (typically 40–60% of the text); tag what remains as [A] outcome sentences, [B]
  mechanism sentences, [C] environment sentences. Map every [B] to an artifact that proves it —
  unmatched [B]s are your gap list. Map every [A] to a number you can state — [A] lines are debrief
  scoring lines, so a story without a matching number is not ready. Map every [C] to the honesty
  caveat you will attach, e.g. "200-node simulated fleet, trace published." Whatever is left
  untagged and unmatched is the round you are weakest in, and that is the one to prep.

- **Why does the debrief structure argue for one thesis rather than eleven projects?** *Answer:*
  each interviewer writes one short note and the notes are compared for consistency, not averaged.
  Eleven equally weighted projects give each interviewer a different impression, so the union reads
  as unfocused, and an inconsistency (dazzling on design, vague on your own artifact) reads as
  borrowed depth rather than as an average. Three load-bearing artifacts carrying one argument —
  the metric lies, the incumbent cost tool inherits that plus a whole-device numerator, here is the
  controller that closes both with a conservation check — produce six notes that say the same
  thing, which is the strongest possible debrief shape.

- **State the three claim tiers and what you may do with each.** *Answer:* Tier 1 is a verifiable
  primary source — the employer's own posting, docs, blog, repo or paper — and you may quote it,
  citing where you saw it. Tier 2 is a pattern seen across several employers, which you state *as*
  a pattern and invite correction on ("in this market I've generally seen X — is that true here?").
  Tier 3 is unverifiable secondhand material — aggregator sites, forum posts, "a friend
  interviewed there" — which you never state as fact about a named employer; use it privately to
  form a hypothesis, then ask the recruiter. The same ladder applies to claims about your own work:
  "I built GPU scheduling infrastructure" is Tier 3 about yourself; "I built a scheduler plugin and
  benchmarked it against the default on a 200-node simulated fleet, trace published" is Tier 1.

- **Why sequence loops in increasing order of importance?** *Answer:* three couplings. Per-loop
  pass probability rises with reps because interview narration is a rehearsed skill; a live offer
  is leverage on timeline and terms everywhere else; and cooldowns of six to twelve months make a
  burned loop non-retryable on this timeline. With p₀ = 0.30 and δ = 0.06 per completed loop,
  running five loops in increasing order of importance gives your top choice p₀ + 4δ = 0.54 against
  p₀ = 0.30 for the reverse order — the ordering is worth more than doubling application volume.
  The constraint that keeps it honest: wave 0 must be roles you would genuinely accept.

- **A 500-GPU H100 fleet: what is a ten-point utilisation improvement worth, and why does that
  shape the loop?** *Answer:* at $2.50/GPU-hour (on-demand basis, mid-2026 directional snapshot;
  quotes in that window spanned roughly $2–11), 500 GPUs cost 500 × 24 × 365 × $2.50 = $10.95M/yr.
  Ten percentage points of that is 500 × 24 × 365 × 0.10 × $2.50 = $1.10M/yr. One engineer moving
  fleet utilisation ten points pays for a team, which is why these loops over-weight R2 (real
  operational contact) and R4 (fast, correct diagnosis) relative to R1. The reliability half of the
  same argument: Meta measured 7.9-hour MTTF for 1,024-GPU jobs and *projects* 1.8 hours at 16,384
  (arXiv:2410.21680) — at that cadence, minutes of guessing are billed silicon.

- **Which of your artifacts do you volunteer without being asked, and why?** *Answer:* GPU cost
  attribution — P2 in the module's prompt set. It is the one design surface where you have a
  source-level teardown of the incumbent tool (module 11.9), an implementation with a conservation
  identity (04.10), and a published measurement argument (05.1/05.8) that explains why the naive
  answer is wrong. Volunteering it converts a generic design round into your home field, and it is
  the fastest way to retire R2 in a round nominally about R3.

## Connections & what's next

This lesson produces the raw material every later lesson in the module consumes: the loop anatomy
that lessons 05-09 drill against, the artifact-to-signal matrix that lesson 02 threads into one
narrative, the evidence graph that lesson 03 renders as your repo's front door, and the claims
hygiene that keeps lesson 04's blog post defensible. The gap list from your posting decodes is the
prioritised backlog for the rest of the module.

Next: **lesson 02** takes the same eleven artifacts this lesson mapped to rounds and asks a
different question — not "which artifact fits which round" but "what single thesis makes all eleven
read as one deliberate body of work." The matrix is the input; the through-line is the output.

## References & further reading

**Primary sources — verifiable, and the basis for every Tier-1 claim above**

- NVIDIA `dcgm-exporter`, `etc/default-counters.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — read for: which fields ship enabled. `DCGM_FI_DEV_GPU_UTIL` is on by default, `DCGM_FI_PROF_SM_ACTIVE`/`SM_OCCUPANCY` are commented out. The single cheapest exhibit in your portfolio.
- NVIDIA `dcgm-exporter`, `grafana/dcgm-exporter-dashboard.json` — https://github.com/NVIDIA/dcgm-exporter/blob/main/grafana/dcgm-exporter-dashboard.json — read for: the eight-panel default dashboard with no SM-breadth panel; published as grafana.com dashboard 12239.
- NVIDIA DCGM, `dcgmlib/dcgm_fields.h` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_fields.h — read for: the field-ID definitions behind every utilisation claim you will make, and the DCGM 4.x renaming block.
- Meta — *Revisiting Reliability in Large-Scale Machine Learning Research Clusters* (arXiv:2410.21680, HPCA 2025) — https://arxiv.org/abs/2410.21680 — read for: eleven months of operations on ≈16K and ≈8K A100 clusters; measured 7.9-hour MTTF at 1,024 GPUs, with 1.8 h at 16,384 and 0.23 h at 131,072 given as **projections**. *Correction vs the previous version of this lesson, which described the 1.8-hour figure as measured and implied an H100 fleet.*
- Meta — *The Llama 3 Herd of Models* (arXiv:2407.21783) — https://arxiv.org/abs/2407.21783 — read for: ≈38–43% model-FLOPs utilisation on 16,384 H100s, the calibration point for every efficiency number you quote.
- OpenCost issue #3900, "GPU 'usage' cost basis does not reflect actual GPU utilization" — https://github.com/opencost/opencost/issues/3900 — read for: the maintainers' own statement of the request-vs-usage gap, with the $2.50-vs-$0.275 expected/actual on a workload at ~11% utilisation. Filed 2026-07-05, closed 2026-07-09.
- OpenCost issue #3828, "Support HAMi GPU core and memory allocation in cost calculations" — https://github.com/opencost/opencost/issues/3828 — read for: an unaffiliated user describing, as a feature request, the exact fractional-attribution correction this course's deliverable builds.

**Course-internal sources — the findings this lesson's evidence graph is built from**

- `platform-eng/modules/05-gpu-observability/lessons/01-lie-and-truth.md` — the two collection paths, field 203's exact semantics, and the ~130× arithmetic between `GPU_UTIL` and `SM_ACTIVE`.
- `platform-eng/modules/05-gpu-observability/lessons/08-capstone-allocated-vs-utilised.md` — the ratio-to-hours integration, the six-bucket decomposition, and the 57.3%-idle worked example used in §9.
- `platform-eng/modules/04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md` — the ownership map, the share model, and reconciliation identities A and B.
- `platform-eng/modules/11-gpu-cost-economics/lessons/09-existing-tooling-limits.md` — the OpenCost source trace and the $2.10 / $14.70 / $0.00 / $8.40 four-regime error table.

**Not relied upon**

- Third-party interview-aggregator and "interview experience" sites. Several were consulted while
  writing this lesson and none could be verified against a primary source; their round counts,
  screen formats and company-specific claims are **not** used as fact anywhere above, and the
  earlier version of this lesson's per-company loop table — which asserted specific round
  structures, an explicit no-algorithmic-screen policy at a named employer, and specific canonical
  prompts attributed to named employers — has been replaced by the archetype model in §4 for that
  reason. Treat any such source as a hypothesis generator only, and verify with the recruiter.
- Individual first-person "my interview experience" posts. Useful for atmosphere, useless as
  evidence: one candidate's loop is not a company's process, and the post is undated relative to
  the org's current structure.

[🎓 12 — Capstone & interview preparation](../README.md)
