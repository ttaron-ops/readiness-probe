---
lesson: 08
title: "Staff-signal behavioral / STAR"
module: 12
concept: "senior-vs-staff scoring"
status: not-started
est_time: "4 hrs"
prev: "07-narrate-artifacts.md"
next: "09-mock-loop-readiness.md"
artifacts: ["a bank of 8-10 staff-tuned STAR stories mapped to the five staff competencies, each landing a metric + a tradeoff/reversibility note"]
sources: 8
---

# Staff-signal behavioral / STAR

## Where this fits

Lesson 07 built the artifact → decision → tradeoff → metric frame and gave you compression control — the same story at thirty seconds, three minutes, or as a deep dive, with the interviewer choosing the resolution. That machinery targets the artifact walk-through: a fixed build, narrated well. This lesson takes the tradeoff and reversibility instincts you just installed and applies them to a different round, anchored to no particular artifact and scored on a different axis. A staff behavioural loop does not ask "did you deliver" — your artifacts already answer that. It asks whether you operate at staff *scope*: how large the problem was, how undefined it was when it arrived, and whether you could move people who did not have to listen to you. What 08 adds is a bank of stories built and framed for exactly that axis, so you are not improvising a senior answer to a staff question.

## Why this matters

You already beat the generic behavioural round in your sleep — STAR structure, "tell me about a conflict," calm delivery. None of that is the problem. The problem is that a staff behavioural loop scores a *different axis* than a senior one, and strong senior candidates fail staff loops by giving technically excellent senior answers.

The concrete risk for you specifically: your instinct under pressure is to reach for the sharpest *technical* moment — the hardest debug, the fastest fix, the cleverest system. Those are senior stories. Told as-is, they actively signal "senior" to a staff panel, because heroics, speed and intuition are the senior register. Your capstone gives you the raw material for the staff register — driving a contested attribution model through three teams, deciding GPU cost was the problem worth solving before anyone asked, correcting the leading open-source tool's design after reading its source — but only if you *frame* those events for the axis being scored.

The stakes compound because the behavioural round is usually the *last* filter rather than the first. You can clear design, debugging and the artifact walk-through cleanly and still get "strong technically, unclear on scope" as the debrief note that sinks the loop — because unlike a design round, where a shaky answer is visibly shaky, a well-told senior story in a staff loop sounds completely fine. Nobody in the room flags it as wrong; the panel just quietly scores you a level lower than your technical work supports. **That failure mode is silent, which is exactly why it needs deliberate preparation rather than "I'll tell my best story."**

And there is a second silent failure in the other direction: overclaiming. A panel that hears "I aligned the org" will probe for specifics, and if the honest answer is "my manager approved it," you have demonstrated the opposite of what you claimed. The defence is not caution — it is only claiming scope that has a concrete answer waiting behind it.

## What's new here (calibration)

- **Skip** (you own it): STAR mechanics, conflict/failure/feedback question types, how to structure and pace an answer, sounding composed. All sharp.
- **New**: the **three scoring dimensions** — scope, ambiguity, influence — and the four-word register tells that place you on the wrong side of each.
- **New**: the **staff-tuned STAR template** with the axis loaded into each slot, and the two slots that decide the level.
- **New**: **ten worked stories**, drawn from this course's actual artifacts and its verified numbers, with the metric and the reversibility note already written.
- **New**: the **register-flip drill** — the same event narrated twice, with the tells underlined, so you can catch yourself mid-answer.
- **New**: the **overclaim guard** — a written pre-answer for the probe every scope claim attracts.
- **New**: the **coverage check** — a diagnostic for a bank that is lopsided toward one competency, which is the most common way a strong bank still fails.

## Core concepts

### 1. What is actually being scored

The gap between a senior and a staff loop is **scope, ambiguity and direction — not raw technical skill.** This is the most important sentence in the lesson. A senior loop scores your coding, your local design, your execution, your ownership of a team's surface. A staff loop scores judgement *across* domains, leadership *without* authority, and your ability to move a contested initiative through disagreement to a decision. The technical bar barely moves. The scope and the ambiguity move a lot.

Three explicit dimensions:

1. **Scope of impact.** Team, multi-team, or organisation. Did your work change your team's outcome, or how three teams operate?
2. **Ambiguity tolerance.** Did you execute a defined problem, or did you *define* the problem when it arrived vague?
3. **Influence without authority.** Could you move people who did not report to you and were not obliged to agree?

And a **framing tell** they listen for, because the same event narrated two ways lands on two different levels:

| Senior register — avoid leading with | Staff register — lead with |
|---|---|
| **Heroics** — "it was broken, I fought through it" | **Tradeoffs made** — "I chose X over Y because…" |
| **Speed** — "I fixed it over a weekend" | **Reversibility** — "one-way versus two-way door; I shipped because it was cheap to reverse" |
| **Intuition** — "I just knew it was the driver" | **Assumed constraints** — "given the constraint was a number a team lead wouldn't dispute…" |
| **Individual save** — "I unblocked the team" | **Defensibility under hindsight** — "knowing what I knew then it was right, and here's what would have flipped it" |

Heroics, speed, intuition and individual-save read senior. Tradeoffs, reversibility, assumed constraints and defensibility read staff. **You can tell the same GPU incident either way. The event does not change; the register does.**

This is consistent with how the industry describes the level jump: staff and principal roles exist to cover work that is *organisationally* ambiguous — no obvious owner, spanning teams, requiring judgement about what to prioritise at all — rather than work that is technically harder in the sense of more difficult code. The bar is not a gnarlier kernel; it is operating where the problem is not handed to you pre-scoped.

```
   WHAT THE PANEL WRITES DOWN — and why the level is decided in the debrief
  ══════════════════════════════════════════════════════════════════════════════

   you tell a story  ────▶  interviewer listens for THREE THINGS ONLY
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   ┌──────────┐        ┌────────────┐        ┌─────────────┐
   │ SCOPE    │        │ AMBIGUITY  │        │ INFLUENCE   │
   │ how many │        │ who defined│        │ who did you │
   │ teams /  │        │ the problem│        │ move, and   │
   │ how much │        │            │        │ did they    │
   │ money    │        │            │        │ have to     │
   │          │        │            │        │ listen?     │
   └────┬─────┘        └─────┬──────┘        └──────┬──────┘
        │ decided in         │ decided in           │ decided in
        │ SITUATION          │ TASK                 │ ACTION
        │ (first 15 s)       │                      │
        └─────────────────────┬───────────────────────┘
                              ▼
              the interviewer writes ONE PARAGRAPH
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ THE DEBRIEF. Notes are COMPARED, not averaged.                 │
   │                                                                 │
   │  "clearly drove a multi-team decision, named the tradeoff,      │
   │   landed a number"            ──────▶  STAFF                    │
   │                                                                 │
   │  "strong engineer, great stories, all individual contributions" │
   │                               ──────▶  SENIOR. Nobody said      │
   │                                        anything was WRONG.      │
   │                                        This is the silent fail. │
   └────────────────────────────────────────────────────────────────┘

   THE OPERATIONAL CONSEQUENCE:
     write every story the way you'd want it SUMMARISED IN ONE LINE,
     because that summary — not your live delivery — is what gets
     compared against other candidates.
```

**The most common failure, and the one to actively guard against:** overclaiming broad impact without concrete cross-team evidence. "I aligned the org" with a solo project behind it reads *worse* than an honest team-scoped story, because the panel probes for specifics and finds air. Your defence is that your strongest claims are real — you genuinely moved finance, ML and platform to an agreed model — so lean on the ones you can evidence and do not inflate the ones you cannot.

### 2. The staff-tuned STAR skeleton

Standard STAR, with the scoring axis loaded into each slot. Two of the four slots decide the level.

```
   THE STAFF-STAR SKELETON — beats marked with what each one is scored on
  ══════════════════════════════════════════════════════════════════════════════

   ┌─ S · SITUATION ─────────────────────────────── put the SCOPE in it ─────┐
   │                                                                          │
   │  ✗  "our team's GPU fleet"                                              │
   │  ✓  "a 64-GPU fleet shared by three teams — research, product           │
   │      inference, and an exploratory group — where GPU was a top-three    │
   │      infrastructure line and no team could tie a dollar to itself"      │
   │                                                                          │
   │  ▸ SCORED: dimension 1, and it is decided in the FIRST FIFTEEN SECONDS. │
   │    A scope you don't establish here you cannot retrofit later.          │
   │  ▸ Include a NUMBER of teams or a NUMBER of dollars. One or the other.  │
   │  ~20 s                                                                   │
   └─────────────────────────────┬────────────────────────────────────────────┘
                                 ▼
   ┌─ T · TASK ──────────────────────────────── put the AMBIGUITY in it ─────┐
   │                                                                          │
   │  ✗  "I was asked to build cost attribution"                             │
   │  ✓  "Nobody owned 'the attribution model', and finance, ML and platform │
   │      all had a veto. Nobody had filed this as a problem. The task I set  │
   │      MYSELF was to get a model all three would stand behind."           │
   │                                                                          │
   │  ▸ SCORED: dimension 2. The words that matter are 'nobody had asked'    │
   │    and 'the task I set myself'.                                         │
   │  ▸ If the task was assigned, SAY SO and locate the ambiguity elsewhere  │
   │    — inventing ambiguity is an overclaim the panel will probe.          │
   │  ~20 s                                                                   │
   └─────────────────────────────┬────────────────────────────────────────────┘
                                 ▼
   ┌─ ★★ A · ACTION ──────────────── put the TRADEOFF and the FORK in it ────┐
   │                                                                          │
   │  ✗  the steps you executed, in order                                    │
   │  ✓  the call at the fork: what you chose, what you gave up, the         │
   │      constraint you assumed, and who you had to move                    │
   │                                                                          │
   │  "I went showback-first: published per-team allocated-versus-utilised   │
   │   numbers with NO billing teeth for two cycles. I traded immediate      │
   │   enforcement — which I could have pushed for — for legitimacy, on the  │
   │   assumption that an agreed model outperforms a mandated one. And I ran │
   │   each team's numbers past their lead before publishing, so nobody was  │
   │   surprised in a room."                                                 │
   │                                                                          │
   │  ▸ SCORED: dimensions 2 AND 3. This is THE staff beat.                  │
   │  ▸ Steps executed = senior. The fork = staff. Same paragraph of time.   │
   │  ~50 s — the longest slot, deliberately                                 │
   └─────────────────────────────┬────────────────────────────────────────────┘
                                 ▼
   ┌─ ★ R · RESULT ────────────── METRIC, then what you'd REVERSE ───────────┐
   │                                                                          │
   │  ✓  "All three signed off, and the showback surfaced that the biggest   │
   │      HOLDER of GPUs was not the biggest USER — one namespace held 2.7×  │
   │      the GPUs of another for about two-thirds of the work. If I did it  │
   │      again I'd propose chargeback after one cycle, not two; I           │
   │      over-indexed on trust-building. Showback-first itself was right —  │
   │      the model stuck because they owned it."                            │
   │                                                                          │
   │  ▸ SCORED: credibility. A staff story with no number is unfalsifiable.  │
   │  ▸ The VOLUNTEERED reversal is the strongest available close, because   │
   │    it demonstrates defensibility under hindsight before you're pushed.  │
   │  ~30 s                                                                   │
   └──────────────────────────────────────────────────────────────────────────┘

   TOTAL ≈ 2 minutes. Non-negotiable per story: A METRIC and a
   TRADEOFF-OR-REVERSIBILITY NOTE. Without the metric it is unfalsifiable.
   Without the tradeoff it is a senior story wearing a staff title.
```

The same discipline appears from the other side of the table in guidance on writing promotion packets: a packet that lists accomplishments without the *why this, why now, what it cost* reasoning reads as competent execution rather than judgement. The template above is the spoken version of that constraint — it forces the reasoning into Action and Result instead of letting Situation and Task carry all the weight.

### 3. The register flip, drawn

This is the drill to run on every story before you consider it finished.

```
   ONE EVENT, TWO REGISTERS — where the level actually gets decided
  ══════════════════════════════════════════════════════════════════════════════

              THE EVENT (identical in both branches)
        you noticed the fleet's headline metric was misleading,
        measured the real numbers, and the fix tripled throughput
                              │
              ┌───────────────┴────────────────┐
              ▼                                ▼
   ┌─────────────────────────┐      ┌─────────────────────────────┐
   │ SENIOR TELLING          │      │ STAFF TELLING               │
   ├─────────────────────────┤      ├─────────────────────────────┤
   │ "The dashboards said    │      │ "The problem arrived as a   │
   │  99% but I had a hunch  │◀ ✗   │  vibe — busy but slow, no   │
   │  something was off."    │ INTU │  metric, no ticket. I       │
   │                         │ ITION│  defined the real question: │
   │                         │      │  is the fleet doing useful  │
   │                         │      │  work?"        ▸ AMBIGUITY  │
   ├─────────────────────────┤      ├─────────────────────────────┤
   │ "I dug into it over a   │◀ ✗   │ "I decided the metric was   │
   │  weekend and figured    │ SPEED│  wrong, not its threshold,  │
   │  out it was reading the │      │  and instrumented the       │
   │  wrong field."          │      │  hardware counters instead. │
   │                         │      │  The tradeoff was real: the │
   │                         │      │  honest field needs         │
   │                         │      │  elevated privileges on     │
   │                         │      │  every GPU node — a         │
   │                         │      │  security conversation —    │
   │                         │      │  and it made the fleet look │
   │                         │      │  far worse to the people    │
   │                         │      │  quoting 99%."  ▸ TRADEOFF  │
   ├─────────────────────────┤      ├─────────────────────────────┤
   │ "It was a lot of work   │◀ ✗   │ "So I never shipped the ugly│
   │  but I nailed it, and   │ SOLO │  number alone — always with │
   │  now everyone uses my   │ SAVE │  the recovery it exposed, so│
   │  dashboard."            │      │  it landed as money on the  │
   │                         │      │  table rather than as a     │
   │                         │      │  callout." ▸ INFLUENCE      │
   ├─────────────────────────┤      ├─────────────────────────────┤
   │ (no number)             │◀ ✗   │ "99 on the presence field   │
   │                         │ UNFAL│  against 0.16 SM-active and │
   │                         │ SIFI │  0.011 tensor activity, same│
   │                         │ ABLE │  card same second. The      │
   │                         │      │  batching fix took          │
   │                         │      │  throughput up ~2.9× while  │
   │                         │      │  that 99% never moved. What │
   │                         │      │  I'd revisit: I hard-coded  │
   │                         │      │  the rate card — a two-way  │
   │                         │      │  door I took for speed, and │
   │                         │      │  a heterogeneous fleet      │
   │                         │      │  flipped it."  ▸ METRIC +   │
   │                         │      │                 REVERSIBILITY│
   └───────────┬─────────────┘      └──────────────┬──────────────┘
               ▼                                   ▼
        SCORED: SENIOR                      SCORED: STAFF
        "strong engineer"                   "defined the problem,
                                             named the tradeoff,
                                             moved a number"

   SAME WEEKEND OF WORK. Nothing about the engineering changed. Only
   scope-in-Situation, tradeoff-in-Action and reversibility-in-Result
   were surfaced. That delta is the entire content of this lesson.

   THE LINTER: before finalising any story, search your own Action and
   Result for four things — "I fixed", "quickly", "I just knew", and a
   bare "I" where a named counterpart belongs. Each hit is a rewrite.
```

### 4. The five competencies, and the artifacts that arm them

You have arming material for all five. The job is mapping, not manufacturing.

**C1 — Technical leadership and direction.** *Armed by: the capstone thesis, anchored on 02 (the
operator) and 05 (the measurement that proved the problem was real).* You identified that GPU cost
was unattributable and drove the fix before anyone scoped it. The frame is **which problem was worth
solving**: not "I was asked to reduce GPU cost" but "I recognised GPU spend was a top infrastructure
line with zero attribution, decided that was the leverage point, and set the direction." That is
technical *direction* — defining the problem — as distinct from technical *skill*, which is solving
a given one.

**C2 — Cross-team influence without authority.** *Armed by: 06 (Kueue showback) and 11 (the cost
schema).* You got finance, the ML teams and platform to agree on an attribution model none of them
owned and all of them could have blocked. The staff move inside it is **showback-first**: publishing
the numbers with no enforcement teeth *deliberately*, to build legitimacy before proposing
chargeback, trading speed for a decision that would hold. That is influence without authority *and*
a named tradeoff in one story. **This is your strongest staff story, and most senior candidates
simply do not have one.**

**C3 — Incident command.** *Armed by: 08 (survive-a-failure) and the failure-mode log from 04.* Not
"I debugged a sick GPU host" — that is an individual save — but coordinating the response and, more
importantly, turning the failure-mode log into the artifact that lets *anyone* run the next
incident. The framing that carries the signal: the log moved incident response from "me" to "the
team," which is scope rather than skill.

**C4 — Driving through ambiguity.** *Armed by: 05.* Your purest ambiguity story. The problem arrived
as a vibe — "the fleet feels busy but slow" — with no metric and no ticket. You defined the real
problem (the reported metric measures kernel residency, not work), instrumented the honest one, and
delivered. Vague in, defined metric out, shipped.

**C5 — Raising the bar through reusable standards.** *Armed by: the exporter and operator being
adopted, and the failure-mode log becoming a runbook.* The staff frame is not "I mentored a junior
engineer," which is one-to-one and team-scoped. It is "I built artifacts that became the standard
other people now operate by" — influence that outlives your presence in any room.

**The archetype check.** These five map onto the widely used staff-archetype framework — Tech Lead,
Architect, Solver, Right Hand. C1 and C4 read as Architect and Solver work; C2 is Right Hand
territory; C5 is the Tech Lead pattern of multiplying a team rather than your own output. You need
not name the archetype aloud, but recognising which archetype each story maps to reveals a lopsided
bank: five stories that are all "I personally fixed the hard thing" under-evidence direction and
influence even if each is individually strong.

```
   COVERAGE MAP — competency × artifact, and where the gaps show
  ══════════════════════════════════════════════════════════════════════════════

                       │ C1      │ C2       │ C3      │ C4      │ C5
                       │ direct- │ influence│ incident│ ambig-  │ standards
                       │ ion     │          │ command │ uity    │
   ────────────────────┼─────────┼──────────┼─────────┼─────────┼──────────
   05 the util lie     │   ██    │    █     │         │  █████  │    █
   02 the operator     │  █████  │          │         │   ██    │   ███
   04 attribution +    │   ██    │          │  ████   │         │  █████
      failure log      │         │          │         │         │
   06 Kueue showback   │   █     │  █████   │         │   █     │    █
   08 survive-a-       │         │          │  █████  │         │   ██
      failure          │         │          │         │         │
   11 tooling teardown │  ████   │   ███    │         │  ███    │    █
                       │         │          │         │         │
   ────────────────────┴─────────┴──────────┴─────────┴─────────┴──────────
   COVERAGE            │ strong  │ ONE      │ strong  │ strong  │ moderate
                       │         │ SOURCE   │         │         │
                                      ▲
                                      │
              THE STRUCTURAL RISK: C2 is armed by essentially one
              event (06/11). If a panel probes it and it doesn't
              hold, you have no second influence story. So it is
              the one story whose specifics must be bulletproof —
              names, dates, what each party actually agreed to —
              and the one place to look hardest for a second
              instance from outside the capstone.

   ██ = primary story   █ = usable as a secondary
```

### 5. The story bank

Ten prompts, the competency each probes, the artifact that arms it, and the register to hit. Build
all ten; the floor is one per competency plus a failure story and a disagreement story, and the
spares stop you reusing one artifact three rounds running.

| # | Prompt you will hear | Competency | Armed by | Register to hit |
|---|---|---|---|---|
| 1 | "A project you're proud of" | C1 direction | 02 + 05 | which problem was worth solving |
| 2 | "A time you influenced without authority" | C2 influence | 06 / 11 | showback-first, legitimacy over speed |
| 3 | "You drove something through disagreement" | C2 influence | 06 | the veto you had to earn past |
| 4 | "You handled a major incident" | C3 command | 08 + 04 | coordinated response and RCA, not solo save |
| 5 | "A vague or undefined problem" | C4 ambiguity | 05 | vague in, defined metric out |
| 6 | "A hard technical tradeoff" | C1 direction | 02 or 11 | named fork plus assumed constraint |
| 7 | "You found a problem nobody else saw" | C1 direction | 11 | read the source, corrected the tool |
| 8 | "You raised the bar / left something behind" | C5 standards | 01 + 02 adopted, 04 as runbook | structural standard, not one-to-one |
| 9 | "Something you'd do differently" | cross-cutting | 05 or 02 | reversibility, not confession |
| 10 | "You disagreed with a decision" | C2 influence | 05 | tradeoff plus defensibility |

### 6. Four fully worked stories

Written in speaking register, on the course's verified numbers.

#### Story 2 — cross-team influence (C2, artifacts 06 and 11). Your strongest.

> **S.** *"We ran a 64-GPU fleet shared by three groups — a research team, a product inference team,
> and an exploratory group. GPU was a top-three infrastructure line, on the order of a couple of
> hundred thousand dollars a quarter, and every team privately believed the other two were the
> wasteful ones. There was no agreed way to settle it."*
>
> **T.** *"Nobody owned 'the attribution model'. Finance, the ML teams and platform all had an
> effective veto, and none of them had filed this as a problem. The task I set myself was to get a
> model all three would actually stand behind — because a model imposed on any of them would be
> argued to death and quietly ignored."*
>
> **A.** *"The call I made was showback-first. Publish per-team allocated-versus-utilised numbers
> with no billing teeth at all for two cycles. I traded immediate enforcement — which I had the
> backing to push for — for legitimacy, on the assumption that an agreed model would outperform a
> mandated one. Two things made it work. First, I published the platform's own buckets on the same
> chart: unallocated capacity and cordoned nodes, which are my number, not theirs. That turned the
> dashboard from an accusation into a diagnostic and it's the single reason the ML teams engaged
> rather than defended. Second, I walked each team's numbers past their lead before publishing, so
> nobody was surprised in a room. I also put a snapshot key on every row so any disputed number was
> replayable — you can't ask people to trust a number they can't audit."*
>
> **R.** *"All three signed off on the model. The showback surfaced something none of them expected:
> the biggest *holder* of GPUs was not the biggest *user* — one namespace held 2.7× the GPUs of
> another and did about two-thirds of the work, which is 33% of the fleet's allocation for 9% of its
> output. That team then chose to fix it themselves rather than being told to. And the reason nobody
> had noticed is the through-line of everything I built: on the dashboard everyone actually looked
> at, that namespace was the busiest in the cluster. What I'd do differently — I waited two full
> cycles before proposing chargeback, and one would have been enough once the numbers held. I
> over-indexed on trust-building. But showback-first itself was right: the model stuck because they
> owned it."*

**Why it scores:** scope is three teams and a dollar figure in the first sentence; ambiguity is
"nobody owned it, nobody had filed it, the task I set myself"; influence is explicit and includes
*how* — publishing your own waste alongside theirs, pre-briefing the leads, making the numbers
auditable; the metric is memorable and second-order; the reversal is volunteered and is a genuine
over-correction rather than a humblebrag.

#### Story 5 — driving through ambiguity (C4, artifact 05)

> **S.** *"Leadership was being shown 99% GPU utilisation across the fleet, and at the same time our
> cost per unit of work was well short of what that implied. The complaint reaching me was a vibe:
> 'the fleet feels busy but slow.' No metric, no ticket, no owner."*
>
> **T.** *"There was no measurement that captured 'is this fleet doing useful work', and nobody had
> filed that as a problem — the 99% was reassuring enough that the question wasn't being asked. The
> task I defined was to work out what 'busy' actually meant and measure the real thing."*
>
> **A.** *"I established that the field every dashboard reads is a passthrough of a driver counter
> whose own definition is the fraction of a short window during which at least one kernel was
> resident. It's a threshold at one, with no notion of how many streaming multiprocessors exist. So
> the metric was wrong, not its threshold — which mattered, because the tempting fix was to adjust a
> threshold and that would have been rebuilding the same problem. I instrumented the hardware-counter
> family instead. The tradeoff was real in two currencies: technically, those fields ship disabled by
> default and need elevated privileges on every GPU node, which is a security conversation rather
> than a config change; and politically, it made the fleet look far worse to exactly the people who
> had been quoting 99%. I chose the true, ugly, actionable number over the flattering one, and I
> de-risked the politics deliberately by never publishing it alone — it always shipped paired with
> the recovery it exposed, so it landed as money on the table rather than as a callout."*
>
> **R.** *"Same GPU, same second: 99 on the presence field, 0.16 SM-active, 0.011 tensor activity —
> 86% of the compute die dark behind a green panel. Integrated over a day, 57% of allocated GPU-hours
> were doing no SM work. That flipped a 'we're full, buy more' conversation into 'fix batching
> first', and the batching change took throughput up about 2.9× on the same eight cards while the 99%
> never moved. The reversibility note: I hard-coded the rate card as constants in the first version
> — a two-way door I took for speed. A heterogeneous fleet flipped it, because a single scalar was
> silently wrong for one of two GPU models, so it became a labelled series versioned in git. Cheap to
> reverse, which is why shipping the assumption was right at the time."*

#### Story 7 — found a problem nobody else saw (C1, artifact 11)

> **S.** *"We were about to build GPU cost attribution, and the obvious objection — the one an
> engineering director asks in the first meeting — is 'doesn't the tool we already run do this?' We
> did already run it, and its dashboard showed GPU costs that looked plausible."*
>
> **T.** *"Nobody had checked whether those numbers were right, and the honest version of my own
> proposal depended on the answer: if the incumbent could do this, my project was a waste of
> everyone's time. So the task I set myself was to falsify my own idea before spending months on
> it."*
>
> **A.** *"I read the tool's source and traced a dollar end to end at a named commit — three
> questions I'd apply to any cost engine: what is the numerator, where does the price come from, and
> where does the interesting telemetry actually land. The numerator turned out to be a Kubernetes
> resource *request* count, whole devices, integer-valued. The price is one scalar per physical GPU.
> And the hardware telemetry is genuinely collected, correctly using the honest counter rather than
> the misleading one — and in the allocation path it is read by exactly one thing, a display ratio.
> It never reaches the multiply. The judgement call was what to do with that. I chose to build a
> correction layer rather than fork or replace, and to upstream the two gaps that genuinely are
> small. I was also careful about the framing, because it would have been easy and wrong to present
> this as 'the tool is bad'. It's an allocated-ledger engine and on a fleet of exclusively held whole
> GPUs it is correct — five of the six gaps I found are one correct design assumption meeting
> hardware that changed underneath it, and the sixth is physics."*
>
> **R.** *"The finding: same H100, same price, one hour, four tenancy configurations — $2.10
> exclusive, which is right; $14.70 under seven-way MIG because seven tenants each bill a whole card;
> $0.00 under time-slicing when the device plugin renames the resource key, because that key is
> absent from both numerator paths; and $8.40 when it doesn't rename. Same tool, same hardware, and
> which answer you get is decided by a GPU-operator flag nobody in the cost conversation is usually
> aware of. That table is what turned 'why not just use the existing tool' from a recurring debate
> into a two-minute answer, and it's what justified the build. What I'd revisit: I should have
> upstreamed the two small fixes immediately rather than a quarter later — a fix that only exists in
> my fork helps nobody, and delaying it cost goodwill I could have banked."*

#### Story 9 — something you'd do differently (cross-cutting)

> **S.** *"Early in the cost work I shipped a query pack that everyone's dashboards were built on —
> three teams' worth of panels, plus the input to the chargeback prototype."*
>
> **T.** *"I owned the arithmetic that turned a hardware ratio into GPU-hours, and nobody else was
> going to check it. This is the part I got wrong."*
>
> **A.** *"I used the obvious query — average the ratio over the window and multiply by the window
> length. It's what almost every write-up on this uses. It's wrong for any workload that didn't run
> for the whole window, because averaging only considers samples that exist, so a job running nine
> hours of a twenty-four-hour window gets its mean extrapolated across the fifteen hours it wasn't
> running. The overstatement is exactly window over time-present — 2.67× in our bursty namespace. The
> reason I caught it was a discipline I'd like to claim was foresight and wasn't: I ran a synthetic
> ground-truth test before publishing a dollar figure with my name on it. Saturate one GPU for
> exactly 600 seconds, expect 0.1667 GPU-hours. I got about four times that, which is the signature
> of exactly this bug."*
>
> **R.** *"I switched to a proper integral, pinned the sample spacing to a recording-rule interval I
> control rather than to a scrape config anyone can edit, and made the ground-truth test a standing
> check rather than a one-off. Two things I took from it. First, the direction of the error mattered
> more than its size: the wrong query made the fleet look *more* utilised and the waste look
> *smaller*, precisely on the bursty workloads where waste is worst — so the naive query was
> understating the problem I was trying to publish. Second, I published the bug in the write-up
> rather than quietly fixing it, because a clean result with no visible mistakes is exactly what a
> fabricated result looks like, and I'd rather be the person who shows their own errors than the
> person who gets caught by someone else's audit."*

**Why this is the right shape for a failure story:** it is a real, technical, consequential mistake;
it is *bounded* — an arithmetic error, not a judgement collapse; the recovery is a process change
rather than a personal resolution; and the reflection contains a generalisable lesson about error
direction that has nothing to do with the specific bug. Notice also that it does not confess a
character flaw, which is the trap most failure answers fall into.

### 7. The remaining six, in compressed form

For each, the one-line scope, the ambiguity claim, the fork, and the number. Expand these into full
STAR before your first mock.

| # | Scope (S) | Ambiguity (T) | The fork (A) | Number + reversal (R) |
|---|---|---|---|---|
| **1** proud of | GPU spend a top-three line across three teams, zero attribution | Nobody had scoped fixing it; I decided it was *the* leverage point over other things I could build | Live controller versus nightly batch reconciliation — took the on-call surface to get a number people would act on | Attribution from "nobody knows" to per-pod dollars, with the conservation identity holding at 1.000 continuously; would have made the rate model pluggable from day one |
| **3** through disagreement | Three teams, one of which actively opposed publishing per-team numbers | No forum existed to settle it; I had to create the decision, not win an argument | Published the platform's own waste first, and gave the opposing team a veto on the *method* while not on the *publication* | All three signed off; the opposing team later used the numbers to justify their own move to preemptible capacity; I'd have brought finance in earlier rather than after the engineering agreement |
| **4** major incident | A distributed job hanging silently across a large fraction of the fleet, with no error and full-looking dashboards | The failure produced no alert, so there was no incident until someone noticed — the ambiguity was whether there *was* an incident | Ran the response with explicit workstreams rather than doing the diagnosis myself: one on bisection, one on stakeholder comms, one on the checkpoint recovery path | Time to diagnosis, and the standing detection rule that came out of the RCA — per-rank step-time variance as a first-class metric, so "the job is slow" becomes "rank 47 is 34% slower" without a human bisecting; I'd have written the runbook during the incident, not after |
| **6** hard tradeoff | The attribution model for time-sliced devices, affecting every shared-GPU tenant's bill | The right answer was "you cannot do this exactly" and nobody wanted to hear it | Emitted an approximation *with an explicit uncertainty label* rather than either refusing to attribute or pretending to precision | Published that 46.8% of utilisation-based chargeback dollars rested on an estimate — about $64k a month — before anyone asked; the reversal is that per-PID attribution would move most of that from estimate to measurement, which is the roadmap item |
| **8** raised the bar | The exporter and the failure-mode log used by people who never spoke to me | Nothing asked me to make them reusable; the default was a script that only I could run | Spent extra time on making the artifacts *operable by someone else* — labelled fidelity rungs, a written decision order, alerting rules rather than dashboards | The log became the on-call runbook and the query pack ran unmodified on a second cluster after changing one rate; I'd have written the "how to interpret this" doc first rather than last, because the artifacts were adopted faster than the understanding was |
| **10** disagreed | Leadership steering capacity decisions on a number I believed was structurally uninformative | Disagreeing meant contradicting a metric that had been reported for years, with no mandate to do so | Brought a measurement rather than an argument, and framed it as an opportunity rather than as a correction | The buy-more conversation became a fix-batching-first conversation; ~2.9× throughput on the same hardware; I'd have brought one of the affected team's own engineers into the measurement earlier, so it arrived as *their* finding rather than mine |

**Note what recurs across all ten:** every fork is a choice between something faster and something
that would hold; every number is one you can derive; and every reversal is a real over-correction or
sequencing error rather than a manufactured modesty. Those three properties are what make a bank
read as honest rather than as prepared.

### 8. The overclaim guard

Every scope claim attracts a probe. Write the answer before you make the claim.

| Claim you might make | The probe it attracts | What you must have ready |
|---|---|---|
| "Three teams agreed on the model" | "Who specifically, and how do you know they agreed?" | The role of each signatory, what each one actually agreed to (the method? the publication? the billing?), and the artifact that records it |
| "It changed how leadership made capacity decisions" | "What decision changed, concretely?" | The specific decision — an expansion request that did not proceed — and roughly what it was worth |
| "The tool was adopted" | "By whom, and is it still running?" | Who runs it, on what, and whether it survived your absence |
| "I led the incident" | "What did *you* do versus what did the team do?" | The workstreams you created and the calls you made, stated without claiming the diagnosis someone else made |
| "I found a bug in a widely used tool" | "Did you report it? What happened?" | The issue, the outcome, and honest credit if the maintainers had already noted it |

**The rule:** downgrade any claim whose probe you cannot answer with specifics. An honest team-scoped
story beats an inflated org-scoped one that collapses under one follow-up — and the collapse is
remembered far longer than the claim would have helped.

## Perspectives

**The panel's view — write for the debrief note.** A panel scoring behavioural rounds is rarely
debating whether your story is *true*. They are debating which of the three dimensions it evidences,
and how strongly. A story that mentions "three teams" in passing and then spends four sentences on
the technical mechanism gives them nothing to write in the scope or influence box, even if the
underlying event was genuinely cross-team. Write your stories the way you would want them summarised
in one line, because that summary — not your delivery — is what gets compared.

**The honesty view.** The overclaim guard exists because panels routinely probe past a claimed
scope, and the gap between what you claimed and what you can evidence is the single most damaging
thing that can surface in a room. The discipline is not to under-claim out of caution; it is to
claim only what has a concrete answer waiting behind it, and to have that answer ready before the
question arrives.

**The register view — run the linter.** Take any draft answer and search the Action and Result slots
for four things: "I fixed," "quickly," "I just knew," and a bare "I" where a named counterpart
belongs. Each hit means the story is still in senior register and needs a tradeoff or reversibility
pass. This is mechanical and it works, which is why it is worth doing on every story rather than
only on the ones this lesson worked through.

**The interviewer's-own-position view.** The person asking may themselves be a staff engineer who
had to make this case for their own promotion, and they are often listening for the answer *they*
would have given. That is why the fairness beats matter — being generous about a tool's design
constraints in story 7, publishing your own waste alongside tenants' in story 2 — because a staff
engineer recognises those moves as the expensive ones. Nobody who has not done it thinks to publish
their own numbers first.

**The GPU-specific view.** Behavioural rounds in this market are not domain-free. When your influence
story is about an attribution model and your ambiguity story is about a metric that measures kernel
residency rather than work, the behavioural round doubles as a domain round — and a panel that
learns something about GPUs from your STAR answer remembers it. That is a genuine advantage of
having built the artifacts: your behavioural bank is not generic, and it cannot be assembled by
someone who has not done the work.

## Real-world use cases

- **The staff-archetype framework** (Tech Lead / Architect / Solver / Right Hand). **What it shows:**
  the vocabulary a panel may already be using internally, and a diagnostic for a lopsided bank —
  five Solver stories under-evidence direction and influence no matter how strong each one is.

- **Industry definitions of staff and principal roles.** These roles are consistently described as
  existing to cover work that is *organisationally* ambiguous — no obvious owner, spanning teams,
  requiring judgement about what to prioritise — rather than work that is technically harder.
  **What it shows:** an outside check on the axis this lesson trains, and the reason a harder
  technical story does not move you up a level.

- **Promotion-packet guidance.** Written for the adjacent problem of building a written case for
  promotion, it makes a structurally identical point: a packet that lists accomplishments without
  the *why this, why now, what it cost* reasoning reads as competent execution rather than
  judgement. **What it shows:** the staff-tuned STAR template is the spoken version of the same
  constraint, forcing the reasoning into Action and Result.

- **The verified numbers behind stories 2, 5, 7 and 9** — the presence-versus-breadth measurement
  (99 against 0.16 and 0.011 on one GPU in one second), the 57.3% idle share and 2.9× batching
  result, the four-regime cost table ($2.10 / $14.70 / $0.00 / $8.40 on one H100), the 46.8%
  exposure fraction, and the integration bug's 2.67× overstatement. **What it shows:** a behavioural
  bank where every number is derivable rather than remembered. The rigor probe is as real in a
  behavioural round as in a technical one, and a figure that collapses discredits the story it was
  meant to support.

## Worked example

**One event, both registers, scored.** The util-lie work, told twice.

**The senior telling — fails a staff panel while sounding fine:**

> *"The dashboards said 99% but I had a hunch something was off. I dug into it over a weekend,
> figured out it was reading the wrong field, and I built a new dashboard that showed the real
> number. It was a lot of work but I nailed it, and now everyone uses my dashboard."*

Tells: *intuition* ("a hunch"), *speed* ("over a weekend"), *individual save* ("I nailed it," "my
dashboard"), and no number at all. Technically accurate, personally impressive, and it will produce
a debrief note that says "strong engineer" with nothing in the scope or influence boxes.

**The staff telling — same event:**

> *"The problem arrived as a vibe — busy but slow, no metric, no ticket, no owner. I defined the
> real question: is this fleet doing useful work? I established that the reported field measures
> whether a kernel was resident, not whether the silicon did anything, so the metric was wrong
> rather than its threshold — which mattered, because the tempting fix was to adjust a threshold and
> that would have rebuilt the same problem one level up. I instrumented the hardware counters
> instead, knowing two costs: those fields need elevated privileges on every GPU node, so it was a
> security conversation rather than a config change; and it made the fleet look far worse to exactly
> the people who had been quoting 99%. So I never published the ugly number alone — always paired
> with the recovery it exposed, which landed it as money on the table rather than as a callout. The
> result was 99 on the presence field against 0.16 SM-active and 0.011 tensor activity on the same
> card in the same second, and 57% of allocated GPU-hours doing no SM work across a day. That
> flipped a buy-more conversation into a fix-batching-first one, and the fix took throughput up about
> 2.9× on the same hardware while the 99% never moved. The two-way door I'd revisit is hard-coding
> the rate card — a heterogeneous fleet flipped that assumption, and I'd have made it data from the
> start."*

Tells: *defined the problem* (ambiguity), *rejected the tempting fix with a reason* (judgement),
*tradeoff in two currencies* (technical and political), *deliberate influence tactic* (pairing the
number with the recovery), *metric with a second-order metric*, *volunteered reversibility*.

**Nothing about the underlying work changed.** Same weekend, same finding, same dashboard. Only
scope-in-Situation, tradeoff-in-Action and reversibility-in-Result were surfaced. That delta is the
entire content of this lesson.

**Now score them the way a panel would.**

| Dimension | Senior telling | Staff telling |
|---|---|---|
| Scope | not established — could be one team | fleet-wide, affects a capacity decision |
| Ambiguity | "a hunch" — implies the problem existed and he noticed it | explicit: no metric, no ticket, no owner, "I defined the real question" |
| Influence | none — built a dashboard others happened to use | explicit tactic, with the reason it was needed |
| Metric | absent | two primary, one second-order, all derivable |
| Reversibility | absent | volunteered, classified, with the flipping trigger |
| **Likely note** | "strong engineer, good instincts" | "defined an unowned problem, navigated a political tradeoff, moved a number" |

The right-hand column is the one that survives a comparison against three other candidates.

## Practice

Feeds [GPU platform capstone](../practice/gpu-platform-capstone/README.md).

1. **Write all ten stories** in full staff-STAR form using §2's template — scope in Situation,
   ambiguity in Task, tradeoff and fork in Action, metric plus reversibility in Result. Use §6's
   four as models and expand §7's six.

2. **Tag every story** with its metric and its tradeoff-or-reversibility note, one line each. If
   either line is empty, the story is not done — rework Action or Result until both exist. Do not
   soften an empty metric line; go and compute the number.

3. **Write the derivation beside every number**, as in lesson 07. A behavioural round's rigor probe
   is identical to a technical round's, and a figure you cannot derive discredits the story.

4. **Run the register-flip drill** on stories 4 and 1: write the *senior* telling of each, underline
   the heroics, speed, intuition and solo-save tells, then rewrite in the staff register. This
   trains the reflex to catch yourself mid-answer.

5. **Run the linter** over every finished story: search Action and Result for "I fixed," "quickly,"
   "I just knew," and a bare "I" where a named counterpart belongs.

6. **Write the overclaim guard** for every scope claim you make: the two or three concrete specifics
   you would give if probed. Where you cannot produce them, downgrade the claim to what you can
   evidence.

7. **Run the coverage check.** Map all ten stories onto the five competencies and onto the four
   archetypes. If six or more land on one archetype, or if a competency is armed by a single event,
   build or reframe a story to cover the gap — C2 is the one most likely to be single-sourced.

8. **Rehearse each to about two minutes**, varying the wording of the opening question. "Tell me
   about convincing skeptical stakeholders" and "a time you influenced without authority" are the
   same prompt; a story that only fits one phrasing is a script, not material.

**Acceptance:** ten written staff-STAR stories, each with a metric line and a
tradeoff-or-reversibility line · a derivation beside every number · two register-flip rewrites · a
written overclaim guard per scope claim · a coverage map with no single-sourced competency left
unaddressed · each story delivered in about two minutes against at least two different phrasings of
its prompt.

## Common pitfalls

1. **Leading with your hardest technical story.** **Mechanism:** heroics read senior, and a genuinely
   hard debugging story is the most natural thing to reach for under pressure precisely because it is
   real and impressive. **Symptom:** your "proud of" answer is your most difficult build rather than
   your most consequential decision. **Fix:** lead with the story that evidences the competency being
   probed, and hold the hardest-debug story for incident command, where the technical detail is
   actually appropriate.

2. **Preparing scripts rather than material.** **Mechanism:** panels ask the same competency in
   different words, and a memorised monologue derails visibly when the phrasing shifts. **Symptom:**
   you answer "tell me about convincing skeptical stakeholders" with an answer that clearly began
   life as "influence without authority." **Fix:** rehearse the beats, not the sentences.

3. **Scope language with no specifics behind it.** **Mechanism:** every scope claim attracts a probe,
   and the gap between claim and evidence is more damaging than a smaller honest claim. **Symptom:**
   "I aligned the org," followed by hesitation at "who specifically?"

4. **Treating "what would you do differently" as a list.** **Mechanism:** a scattershot answer reads
   as unfocused self-criticism rather than judgement. **Symptom:** "well, I could have documented
   better, and communicated earlier, and also…" **Fix:** exactly one reversible decision tied to the
   story just told, the information that would flip it, and stop.

5. **A failure story that confesses a character flaw.** **Mechanism:** the round is scoring judgement
   under uncertainty, not self-awareness about temperament, and a character confession gives the
   panel something they cannot un-hear and cannot score positively. **Symptom:** "I can be impatient
   with people." **Fix:** a bounded technical or process error with a process fix, as in story 9.

6. **A bank that is all one archetype.** **Mechanism:** five Solver stories under-evidence direction
   and influence regardless of individual quality, and the panel's job is to fill three boxes.
   **Symptom:** every story is "I personally found and fixed the hard thing."

7. **A single-sourced competency.** **Mechanism:** if one event arms your only influence story and
   the panel probes it hard, you have no second instance and the dimension goes unevidenced.
   **Symptom:** stories 2, 3 and 10 are the same event with different openings. **Fix:** either find
   a genuine second instance or be honest that the influence work concentrated in one initiative and
   evidence it in unusual depth.

8. **Numbers you cannot derive.** **Mechanism:** the rigor probe is as real here as in a technical
   round, and a collapsed figure retroactively discredits the story. **Symptom:** you say "about 50%
   waste" and cannot say over what window, on what basis, or how it was integrated.

## Self-check

- **What three dimensions does a staff behavioural loop score, and what is the framing tell that
  places you on the senior side?** *Answer:* scope of impact (team versus multi-team versus
  organisation), ambiguity tolerance (executed a defined problem versus defined it yourself), and
  influence without authority (could you move people who were not obliged to agree). The tell is the
  register: heroics, speed, intuition and individual-save language read senior; tradeoffs made,
  reversibility, assumed constraints and defensibility under hindsight read staff. The same event
  told in the first register scores senior regardless of how hard the work was — which is why the
  failure is silent, since nobody in the room says anything was wrong.

- **Where in the STAR skeleton is each dimension decided, and which slot is longest?** *Answer:*
  scope is decided in the Situation, in the first fifteen seconds, and cannot be retrofitted later —
  so the Situation must contain a number of teams or a number of dollars. Ambiguity is decided in the
  Task, by the words "nobody had asked" and "the task I set myself." Influence and the tradeoff are
  both decided in the Action, which is deliberately the longest slot at roughly fifty seconds,
  because steps-executed reads senior and the-call-at-the-fork reads staff and they occupy the same
  amount of speaking time. The Result carries the metric and the volunteered reversal, which is the
  strongest available close because it demonstrates defensibility before you are pushed for it.

- **Which artifact arms influence-without-authority, and what specific staff *move* is inside it?**
  *Answer:* the showback work across 06 and 11, where three groups agreed on an attribution model
  none of them owned and each could have blocked. The move is showback-first: publishing the numbers
  with no enforcement teeth *deliberately* for two cycles, trading immediate enforcement for
  legitimacy on the assumption that an agreed model outperforms a mandated one. Two tactics inside it
  carry most of the signal — publishing the platform's own buckets (unallocated and cordoned
  capacity) on the same chart, which turns the dashboard from an accusation into a diagnostic; and
  pre-briefing each team's lead so nobody was surprised in a room. The guard: this is the claim you
  lean on because the specifics exist, and it is also the competency most at risk of being
  single-sourced.

- **Why is overclaiming the most common staff-loop failure, and how do you defend against it?**
  *Answer:* because panels routinely probe past a claimed scope, and finding air where the specifics
  should be is worse than an honest smaller claim — it demonstrates the opposite of the judgement you
  were claiming. The defence is a written pre-answer for every scope claim: who specifically agreed
  and to what, which decision concretely changed and roughly what it was worth, who still runs the
  thing you built and whether it survived your absence. Where you cannot answer the probe with
  specifics, downgrade the claim. The discipline is not caution, it is only claiming what has a
  concrete answer waiting behind it.

- **What makes a good "what would you do differently" answer, and what makes a bad one?**
  *Answer:* a good one picks exactly one reversible, two-way-door decision tied to the story just
  told, names the new information that would flip it, states why shipping the original assumption
  was still correct given what was known, and stops. For example: hard-coding the rate card as
  constants, flipped by a heterogeneous fleet where one scalar was silently wrong for one GPU model,
  which then became a labelled series versioned in git — cheap to reverse, which is why shipping the
  assumption was right at the time. A bad one is either a scattershot list of unrelated
  self-criticism, which reads as unfocused rather than reflective, or a character confession, which
  gives the panel something they cannot score positively and cannot un-hear.

- **What is the coverage check, and what does a lopsided bank look like?** *Answer:* map all your
  stories onto the five competencies (direction, influence, incident command, ambiguity, standards)
  and onto the four staff archetypes. A lopsided bank is one where six or more stories land on the
  same archetype — most commonly Solver, "I personally found and fixed the hard thing" — which
  under-evidences direction and influence no matter how strong each individual story is, because the
  panel has three boxes to fill and you have filled one of them five times. The related failure is a
  *single-sourced* competency, where one event arms your only influence story: if the panel probes
  it and it does not hold, that dimension goes unevidenced entirely.

- **Why does the behavioural round in this market double as a domain round, and why does that
  matter?** *Answer:* because your influence story is about an attribution model, your ambiguity
  story is about a metric that measures kernel residency rather than work, and your
  found-a-problem story is a source-level teardown of a cost engine. A panel that learns something
  about GPU accounting from a STAR answer remembers it, and the domain content cannot be assembled
  by a candidate who has not built the artifacts. It also raises the bar on precision: every number
  you use in a behavioural answer is subject to the same rigor probe as in a technical round, so the
  derivations have to be as ready here as they are in lesson 07.

## Connections & what's next

This lesson's staff-tuned STAR template is the behavioural counterpart to lesson 07's artifact frame
— both insist on a named tradeoff and a metric, but 07 anchors to a specific build under a clock
while 08 anchors to a competency and lets you choose the artifact that arms it. Story 2 is shared
infrastructure between the two: it is a lead artifact narration in 07 and the anchor of the
influence competency here, so a weakness in one shows up in both. The register-flip drill you ran
here is also the skill lesson 09 tests under real time pressure, back to back with the design and
debugging rounds, where slice-by-slice fluency either holds up or collapses under interference.

Next: **lesson 09** runs a full timed mock loop and scores whether the frames from 05 through 08
survive being deployed together, cold, on someone else's clock.

## References & further reading

**Primary sources**

- Will Larson, "Staff archetypes" — https://staffeng.com/guides/staff-archetypes/ — read for: the Tech Lead / Architect / Solver / Right Hand framework used as the coverage-check vocabulary in §4 and §7.
- Will Larson, "Promotion packets" — https://staffeng.com/guides/promo-packets/ — read for: the written-case version of the same constraint the STAR template imposes — accomplishments without the *why this, why now, what it cost* reasoning read as execution rather than judgement.
- Meta — *Revisiting Reliability in Large-Scale Machine Learning Research Clusters* (arXiv:2410.21680, HPCA 2025) — https://arxiv.org/abs/2410.21680 — read for: the failure-rate context behind story 4's incident-command framing; measured 7.9-hour MTTF at 1,024 GPUs, with larger-scale figures given as projections.

**Course-internal sources — every number in the bank**

- `platform-eng/modules/05-gpu-observability/lessons/01-lie-and-truth.md` — the field semantics and the 99 / 0.16 / 0.011 measurement behind stories 5 and 10, and the derivation the rigor probe will ask for.
- `platform-eng/modules/05-gpu-observability/lessons/08-capstone-allocated-vs-utilised.md` — the 57.3% idle share, the 2.9× batching result, the reshuffle finding behind story 2's result beat, and the `avg_over_time` bug and synthetic ground-truth test behind story 9.
- `platform-eng/modules/04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md` — the conservation identity behind story 1 and the 46.8% exposure fraction behind story 6.
- `platform-eng/modules/11-gpu-cost-economics/lessons/09-existing-tooling-limits.md` — the four-regime cost table and the fairness framing behind story 7.

**Not relied upon**

- General career-advice articles and first-person promotion write-ups on staff-level expectations
  were consulted for the level-definition framing in §1. They are described as a consistent industry
  pattern rather than cited as authoritative for any specific claim, and no number in this lesson
  depends on them. Earlier versions of this lesson used a "95% utilisation versus 31% MFU" figure
  and described the reported dashboard field as SM-active; both were wrong — the residency semantics
  belong to the presence field rather than to SM-active, and every figure here now traces to this
  course's own measurements.
