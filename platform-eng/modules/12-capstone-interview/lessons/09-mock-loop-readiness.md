---
lesson: 09
title: "Mock loop & the readiness gate"
module: 12
concept: "full-loop simulation + final gate"
status: not-started
est_time: "5 hrs"
prev: "08-staff-behavioral.md"
next: null
artifacts: ["a scored mock-loop record + a remediation list"]
sources: 9
---

# Mock loop & the readiness gate

## Where this fits

Lessons 05-08 each drilled one slice of the loop in isolation, on its own clock, with nothing else competing for your attention: 05 built six system-design skeletons and the reflex to volunteer scale, cost, failure modes and SLOs; 06 built three debugging decision trees and the discipline of narrating hypothesis before command; 07 turned your capstone artifacts into three-minute decision→tradeoff→metric narrations at three compressions; 08 rebuilt your behavioural bank from senior-register heroics into staff-register scope, ambiguity and influence. Each proved you *can* do one thing well, rested, with time to think. None proved you can do all four back to back, on a stranger's clock, with no notes and no do-overs — which is the actual shape of the thing you are being hired against. This lesson closes that gap. It is not new material; it is the integration test that either confirms the previous four lessons compose under pressure or tells you precisely where they do not. It is also the final gate of the entire course.

## Why this matters

Practising slices proves you can do each thing. It does not prove you can do all four cold, in sequence, on a clock, unaided — and that is a separate skill that is separately tested.

The failure mode has a name and it is **interference**. You nail the design round, arrive at behavioural drained, and deliver a metric-free story. Or the design round runs long, you go into debugging still mentally holding a KV-cache calculation, and you skip the cheap check you have drilled twenty times. The debrief averages you down to "strong but inconsistent — no offer," and every individual round was, in isolation, something you could have passed.

This is the final gate of the whole course, and it is deliberately pass/fail rather than graded. You do not get to score yourself on effort or on how much you have read. You score yourself on whether, in a timed simulation with no notes, you clear five hard-edged dimensions. **The stake is calibration:** candidates who *feel* ready and candidates who *are* ready diverge exactly here, and the cost of being wrong is a burned loop at an employer you get one clean shot at, with a cooldown measured in quarters. Failing a gate at your own desk this week is orders of magnitude cheaper.

There is also an operational reason the debugging round in particular carries so much weight, worth internalising before the run rather than after. Distributed GPU training is failure-prone at a rate that makes diagnostic reflexes a survival skill: a published study of eleven months across two large research clusters measured a **mean-time-to-failure of 7.9 hours for 1,024-GPU jobs**, with shorter times at larger scale given as the paper's projections (arXiv:2410.21680). A platform engineer who cannot rapidly bisect compute, communication and data is not a weak interviewer — they are a liability on a live fleet. The mock loop's debugging round exists because the job it screens for genuinely looks like that.

## What's new here (calibration)

Nothing here is a new interview skill. Four new things:

- **The full-loop protocol** — four rounds stitched into one timed session, self-run or with a peer, so you rehearse the *sequence and the fatigue* rather than the parts.
- **The readiness rubric** — five pass/fail dimensions, each with an explicit, quotable fail signal, plus a routing table from every failure back to the lesson that fixes it.
- **Fatigue engineering** — what specifically degrades in round four, why, and the three counters that are cheap enough to actually use.
- **Scoring discipline** — the void rules that stop a self-administered gate drifting lenient, which is the single most likely way this whole exercise produces a false pass.

## Core concepts

### 1. Why an integration test, and what interference actually looks like

The software-testing metaphor maps almost exactly, and it is worth taking seriously rather than as a flourish. Lessons 05-08 are unit tests: each exercises one component in isolation, with clean inputs and no shared state. A passing unit test tells you the component works *when nothing else is happening*. This lesson is the integration test: it runs all four components together under shared resource constraints — your attention, your energy, the clock — and it is designed to surface the bugs that only appear under load. A codebase with complete unit coverage and no integration tests still ships bugs. A candidate with four individually strong lessons and no full-loop rehearsal still fails loops.

**What interference concretely looks like**, in rough order of how often it fires:

| Interference | Mechanism | Where it lands |
|---|---|---|
| **Working-memory carry-over** | you finish round 2 still holding a calculation, and round 3's opening thirty seconds get spent unloading it | you skip the cheap check in D1/D2 and jump to a profiler |
| **Register lock-in** | three rounds of technical precision leaves you in a technical register, and round 4 wants an organisational one | your STAR stories become technical narrations with no scope or influence in them |
| **Number fatigue** | you have quoted twelve figures in two hours and stop reaching for them | the behavioural round produces "throughput improved a lot" instead of a number |
| **Clock debt** | the design round overruns by ten minutes, and you compress the round you were weakest at anyway | the round you most needed the time in is the one that gets squeezed |
| **Confidence swing** | one bad round produces either over-correction (over-explaining in the next round) or collapse | inconsistency, which a panel scores worse than uniform mediocrity |
| **Question-drift blindness** | you stop listening to the exact question because you are tired, and answer the question you rehearsed | a rigor probe answered with scope, or the wrong story for the competency |

Notice that four of the six degrade the *behavioural* round specifically, which is why round four is last in a real loop and last here. That is not a coincidence — it is where the interference accumulates.

### 2. The loop shape

Mirror the composite loop from lesson 01, compressed into one sitting. Four rounds, 170 minutes.

```
   THE 170-MINUTE GATE — the sequence, and where interference bites
  ══════════════════════════════════════════════════════════════════════════════

   ATTENTION
   (your own)
      high ┤███████████████
           │███████████████████████
           │███████████████████████████████
           │██████████████████████████████████████
      med  ┤█████████████████████████████████████████████
           │████████████████████████████████████████████████████
           │█████████████████████████████████████████████████████████
      low  ┤██████████████████████████████████████████████████████████████
           └────────────┬────────────┬────────────┬────────────┬─────────▶
                        │            │            │            │        t
   ┌────────────────────┴──┐ ┌───────┴────────┐ ┌─┴──────────┐ ┌┴────────────┐
   │ R1 · SCREEN           │ │ R2 · DESIGN    │ │ R3 · DEBUG │ │ R4 · STAR   │
   │ 35 min                │ │ 45 min         │ │ 45 min     │ │ 45 min      │
   ├───────────────────────┤ ├────────────────┤ ├────────────┤ ├─────────────┤
   │ · 45-sec positioning  │ │ · one of P1–P6 │ │ · D2, the  │ │ · 3 STAR    │
   │   sentence            │ │ · volunteer    │ │   100%-    │ │   stories,  │
   │ · résumé-driven       │ │   scale/cost/  │ │   util trap│ │   prompts   │
   │   domain Q&A          │ │   SLO/failure  │ │   (MANDA-  │ │   chosen by │
   │ · 5-min artifact      │ │   in first 3   │ │   TORY)    │ │   the       │
   │   value-chain walk    │ │   min          │ │ · hypoth-  │ │   interview-│
   │ · YOUR questions      │ │ · 25 min       │ │   esis →   │ │   er, not   │
   │                       │ │   design out   │ │   command  │ │   you       │
   │                       │ │   loud         │ │   → signal │ │             │
   │                       │ │ · 15 min under │ │   → branch │ │             │
   │                       │ │   pressure     │ │            │ │             │
   └───────────┬───────────┘ └───────┬────────┘ └─────┬──────┘ └──────┬──────┘
               │  5 min              │  5 min         │  5 min        │
               ▼                     ▼                ▼               ▼
        ┌──────────────────────────────────────────────────────────────────┐
        │ WHERE INTERFERENCE BITES                                         │
        │                                                                  │
        │  R2→R3 : working-memory carry-over. You are still holding the    │
        │          design when the incident starts. COUNTER: 60 seconds    │
        │          of writing down where you stopped, then close it.       │
        │                                                                  │
        │  R3→R4 : register lock-in + number fatigue. Three technical      │
        │          rounds, then a round that wants organisational scope    │
        │          and hard numbers. THIS IS WHERE MOST RUNS FAIL.         │
        │          COUNTER: read your metric line for each of the three    │
        │          stories out loud during the break. Not the whole story  │
        │          — just the numbers.                                     │
        │                                                                  │
        │  ANY→R2: clock debt. If R1 overruns you will compress R2's       │
        │          opening, which is exactly the three minutes where the   │
        │          volunteered dimensions live. COUNTER: hard-stop R1.     │
        └──────────────────────────────────────────────────────────────────┘

   RULES OF THE RUN
     · one block, 5-minute breaks only         · no notes
     · every round recorded (audio minimum)    · no pausing
     · prompts read cold, or withheld by a peer· no re-takes
     · a run where you looked something up is a PRACTICE REP, not a gate
       attempt. Say so in the record.
```

**Why round 3 is fixed on D2.** The other two debugging scenarios are good drills; D2 is the discriminating one. It is the single cleanest test of whether a candidate has actually operated GPUs, because the whole scenario is built around a metric that reads high precisely when the fleet is most wasted. The interviewer says "utilisation is pegged at 100% but throughput is half of yesterday — go," and the entire test is whether you distrust the number and pivot to something that measures work, then bisect compute, communication and data with a named command and an expected reading at each branch.

**Why the artifact walk is inside round 1 rather than being its own round.** In a real loop the artifact narration rarely gets a dedicated slot; it appears inside a screen or a domain round, usually with less time than you expected. Putting it in round 1 with a five-minute cap reproduces that pressure, and it front-loads the dimension that your published portfolio should make easy.

**If you have a peer**, they play interviewer and — critically — *withhold* prompts. The "do you volunteer it?" test is impossible to self-administer honestly, because you cannot un-know that you were going to say the thing. Solo, the substitute is the recording: you can hear on playback whether a dimension arrived on your initiative or after a pause.

### 3. The five dimensions, with their fail signals

Score each **pass or fail. No partial credit**, because interviews do not give partial credit.

**Dimension 1 — System design.** Take any of P1–P6 and cover all of that prompt's probe axes *while volunteering* scale, cost, failure modes and SLOs in the first three minutes, before being asked. No hand-waving the GPU-specific layer — whichever of isolation model, gang scheduling, KV-cache capacity, lemon nodes or the utilisation-versus-work distinction that prompt turns on.

> **Fail signals, quotable:** the answer would score as "strong web or distributed engineer" — you sharded, cached, queued and load-balanced but never named a GPU-specific constraint. Or the transcript contains the interviewer's words "have you thought about cost / failure / scale?", meaning you waited to be prompted on a dimension that is scored on being volunteered. Or you stated a verdict ("I'd use MIG") with no reversal condition, which is a preference rather than a decision.

**Dimension 2 — Debugging.** Run D2 end to end: distrust the presence metric *with a mechanism rather than an adjective*, pivot to something that measures work, and bisect compute against communication against data with a named command and an expected reading at each branch.

> **Fail signals:** you quoted the utilisation number as truth — treated 100% as "the GPU is working hard, so that is fine." This is the single most common disqualifier in this round. Or you distrusted it correctly and then guessed at a cause without bisecting ("must be the network"), which is indistinguishable from luck even when correct. Or you ran a command without first saying what you expected to see and where each result would send you. Or you never checked whether the honest metric was still being exported, and spent the round hunting a workload cause for a configuration change.

**Dimension 3 — Behavioural.** Deliver three staff-level STAR stories, each landing a **metric** and a **tradeoff or reversibility note**, each reading as **multi-team scope** rather than as an individual contribution.

> **Fail signals:** a story with no number — "throughput improved a lot" is a fail, and "I basically said a number" is the fail signal, not a defence. A story that is all Situation and Task and thin on *your* Action. Scope that reads as individual ("I fixed a bug") rather than staff ("I changed how three teams attribute cost"). Or all three stories in the senior register — heroics, speed, intuition, individual save — regardless of how strong the underlying work was.

**Dimension 4 — Artifact narration.** Walk the full capstone value chain in five minutes, and connect **any single artifact the interviewer points at** to a decision, a tradeoff and a number.

> **Fail signals:** you can recite what an artifact *is* but not what decision it drove. You reach for a tradeoff or a number and have none, so the artifact reads as a demo rather than as operated judgement. Or a number collapses under "where did that come from?" — which is worse than having no number, because it retroactively discredits the others.

**Dimension 5 — Portfolio.** The repo, design doc and flagship post are published and self-explanatory to a hiring manager who never meets you. A stranger lands on the README and understands the value chain without you in the room.

> **Fail signals:** the story only works when *you* narrate it. Dead links, or links that resolve on the repository host and 404 on the built site. A design doc with no Non-goals or an Alternatives section with no verdict. A figure with no basis or date. A simulated result not labelled as simulated.

**Note what the rubric deliberately does not score:** how smart you sounded, how much you knew that never left your mouth, or how good any one round was in isolation. That is a feature. A real debrief works the same way — discrete signals from people who compare notes, not a holistic impression from one conversation.

### 4. The scorecard

Print this, or reproduce it. Filling it honestly is the actual work of this lesson.

```
   READINESS GATE SCORECARD          run #____   date ____________
  ══════════════════════════════════════════════════════════════════════════════
   Prompt used in R2: P___        Peer-run / self-run: ________
   Any rule bent during the run?  Y / N    → if Y, this run is VOID
  ──────────────────────────────────────────────────────────────────────────────

   D1 · SYSTEM DESIGN                                        PASS / FAIL
       ☐ all probe axes for that prompt touched (≥1)
       ☐ ≥3 axes defended with a tradeoff (=2)
       ☐ scale volunteered      … at time ____ (must be < 03:00)
       ☐ cost volunteered       … at time ____
       ☐ SLO volunteered        … at time ____
       ☐ failure modes volunteered … at time ____
       ☐ GPU-specific layer named without prompting
       ☐ every verdict carried a reversal condition
       fail signal fired (quote yourself): ______________________________
       __________________________________________________________________

   D2 · DEBUGGING (D2 scenario)                              PASS / FAIL
       ☐ rejected the presence metric with a MECHANISM, not an adjective
       ☐ established ground truth before touching device metrics
       ☐ checked for metric ABSENCE before hunting a workload cause
       ☐ bisected compute / comms / data
       ☐ every command preceded by expected signal + branch
       ☐ said what each observation RULED OUT
       ☐ closed on PREVENTION, not on the diagnosis
       fail signal fired: ________________________________________________

   D3 · BEHAVIOURAL (3 stories)                              PASS / FAIL
                              story A     story B     story C
       metric present           ☐           ☐           ☐
       metric derivable         ☐           ☐           ☐
       tradeoff / reversal      ☐           ☐           ☐
       multi-team scope in S    ☐           ☐           ☐
       ambiguity in T           ☐           ☐           ☐
       staff register (no       ☐           ☐           ☐
         heroics/speed/
         intuition/solo-save)
       fail signal fired: ________________________________________________

   D4 · ARTIFACT NARRATION                                   PASS / FAIL
       ☐ full value chain in ≤5:00        actual time ____
       ☐ interviewer-chosen artifact → decision ✓ tradeoff ✓ number ✓
       ☐ every number survived "where did that come from?"
       ☐ honesty caveat delivered flat, not apologetically
       fail signal fired: ________________________________________________

   D5 · PORTFOLIO                                            PASS / FAIL
       ☐ a stranger reads the README for 4 min and can state: what it is,
         the strongest number, and what to click next
       ☐ all relative links resolve ON THE BUILT SITE
       ☐ design doc has Non-goals AND an Alternatives verdict
       ☐ every $ figure has a basis and a date
       ☐ every simulated result labelled as simulated
       fail signal fired: ________________________________________________

  ──────────────────────────────────────────────────────────────────────────────
   RESULT:  ___ / 5 passed.       GATE PASS requires 5/5 in ONE unaided run.
   REMEDIATION LIST (one line per fail, with the lesson it routes to):
     1. ______________________________________________________________
     2. ______________________________________________________________
     3. ______________________________________________________________
   NEXT ACTION:  re-drill ______  →  re-run the WHOLE loop on ________
  ══════════════════════════════════════════════════════════════════════════════
```

### 5. Scoring discipline: the void rules

A self-administered gate drifts lenient. This is not a character flaw; it is structural, because the assessor and the assessed are the same motivated person, and you *want* the preparation to have worked. That want quietly reshapes grading: a vague-but-confident answer gets rounded up to "basically fine," a stumble gets excused as "I knew it, I was just tired."

The five-dimension, no-partial-credit rubric exists to correct for this, and it works only if the procedure is mechanical rather than willpower-based:

- **No notes.** Not "notes I didn't look at." No notes in the room.
- **No pausing.** The clock runs through your thinking, your silences, and your recovery.
- **No re-takes.** A round you restarted is not a round.
- **If you bend a rule mid-run, the run is void.** Restart the whole loop. Do not patch it. The reason is that a patched run produces a *record* that says pass, and the record is what you will look at in three weeks when deciding whether to open your campaign.
- **Score from the recording, not from memory.** Memory reconstructs your intent; the recording contains your output. These differ most on exactly the dimensions being scored.
- **Score at least an hour after the run**, or better the next morning. Immediately afterwards you are still inside the experience and cannot hear it.

**The specific self-deception to watch for** is "well, I did know that." If you catch yourself justifying a pass with knowledge that never left your mouth, that is the fail signal, not the defence. A real interviewer has no access to your internal state — only to what you said.

**The external calibration rep.** If feasible, get one run scored by someone who was not in the room for your self-run: a peer, a working engineer, or a paid mock service. Then compare their scoring against yours, dimension by dimension. The *gap* is the data point — it tells you whether your self-scoring has drifted lenient, and by how much. A consistent one-dimension gap means you should treat your own 5/5 as a 4/5 until it closes.

### 6. Remediation routing

The remediation list *is* the output of a failed run. For every failed dimension, quote the fail signal that fired, route it to the lesson that fixes it, re-drill that lesson, and then re-run the **whole** loop.

```
   REMEDIATION ROUTING — from fail signal to the thing that fixes it
  ══════════════════════════════════════════════════════════════════════════════

   D1 SYSTEM DESIGN
     "never named a GPU-specific constraint"      ──▶ L05 §3–8 (the six drills)
                                                      + the arming module for
                                                        the axis you missed
     "waited to be prompted on cost/scale/SLO"    ──▶ L05 §2 (universal opening)
                                                      — drill the 90-second block
                                                        until it is automatic
     "verdict with no reversal condition"         ──▶ L02 §5 (alternatives bank)
     "ran out of time before the GPU layer"       ──▶ L05 §1 (the block clock);
                                                        you went depth-first

   D2 DEBUGGING
     "quoted the presence metric as truth"        ──▶ L06 §4 (D2) and, underneath
                                                        it, M05 L01 — this is a
                                                        knowledge gap, not a
                                                        narration gap
     "distrusted it, then guessed"                ──▶ L06 §1 (narration discipline)
     "ran a command with no stated expectation"   ──▶ L06 §6 (live protocol,
                                                        the five-second habit)
     "never checked for metric absence"           ──▶ L06 §4 (the fifth suspect)
     "ended on the diagnosis"                     ──▶ L06 §5 (prevention close)

   D3 BEHAVIOURAL
     "a story with no number"                     ──▶ L08 §2 (Result slot) and
                                                        L08 practice item 2 —
                                                        go compute the number,
                                                        do not soften the line
     "IC scope, not staff scope"                  ──▶ L08 §2 (Situation slot)
     "senior register — heroics/speed/intuition"  ──▶ L08 §3 (register flip drill)
     "scope claim collapsed under probing"        ──▶ L08 §8 (overclaim guard)
     "all three stories were the same artifact"   ──▶ L07 §7 (portfolio balance)
                                                      + L08 §4 (coverage map)

   D4 ARTIFACT NARRATION
     "artifact = demo, no decision"               ──▶ L07 §1–2 (the frame, and
                                                        opening on the problem)
     "a number collapsed under 'where from?'"     ──▶ L07 §5 (rigor probe) —
                                                        write the derivation
                                                        beside every figure
     "ran long / could not compress"              ──▶ L07 §4 (compression ladder)
     "caveat sounded apologetic"                  ──▶ L02 §9 (honesty close)

   D5 PORTFOLIO
     "only works when I narrate it"               ──▶ L03 §1 + §3 (the cold read,
                                                        the front-door README)
     "links 404 on the built site"                ──▶ L03 §8 (publishing hygiene)
     "no Non-goals / no Alternatives verdict"     ──▶ L03 §5 (the worked doc)
     "an undated dollar figure"                   ──▶ L03 §8
     "no public flagship"                         ──▶ L04 (the whole lesson)

   ═══════════════════════════════════════════════════════════════════════════
    THEN: re-run the WHOLE 170 minutes. Not the failed round.
    A round rehearsed fresh proves it works rested. It proves nothing about
    whether it survives as round 4 after 125 minutes of everything else —
    which is the exact property the gate exists to test.
   ═══════════════════════════════════════════════════════════════════════════
```

### 7. Fatigue engineering

Most of the interference in §1 is predictable, which means it is counterable. Three counters, all cheap enough that you will actually use them.

**Counter 1 — the unload, at every break.** Sixty seconds, written: where did I stop, what was I still holding, what did I not get to say. Then physically close it. The purpose is not the note; it is giving your working memory permission to drop the previous round. Untouched, a design round bleeds into the first two minutes of a debugging round, and those two minutes are exactly where the cheap check lives.

**Counter 2 — the number rehearsal, before round 4 only.** Do not re-read your stories in the break; you will not retain them and you will arrive over-rehearsed and stiff. Read *only the metric line* of each of your three stories, out loud. Twenty seconds. This is targeted at number fatigue specifically, which is the single most common cause of a failed behavioural dimension, and it works because the failure is one of retrieval rather than of knowledge.

**Counter 3 — the register reset, before round 4.** Say one sentence out loud in the organisational register before the round starts — not from a story, just any sentence about people and decisions rather than systems. It sounds trivial. Three consecutive rounds of field IDs and bandwidth arithmetic leave you in a register where "three teams each had a veto" does not come naturally, and the first thirty seconds of round four are where scope is decided.

**And one thing not to do:** do not stack the run at the end of a working day. The gate is measuring whether your prepared material survives fatigue, not whether it survives fatigue *plus* eight hours of unrelated work. Run it when a real loop would run — mid-morning, rested — or you are measuring the wrong thing and will remediate against a phantom.

**One honest caveat about self-run fatigue.** A self-administered loop is genuinely less tiring than a real one, because there is no social pressure, no unfamiliar faces, and no stakes. Expect the real thing to be harder, and treat a marginal self-run pass as a fail. If you scraped through round four, you would not have scraped through it in a room.

### 8. What a pass unlocks, and what it does not

A clean five-dimension pass is the trigger to open the campaign, and the sequencing from lesson 01 still governs. It is not overridden by having passed.

- **Lead with wave 0 and wave 1** — loops you would genuinely accept but are not optimising for, then the ones where your artifacts convert hardest. Per-loop performance rises with reps and cooldowns are long, so ordering is worth more than volume.
- **Do not open on a dimension you just failed.** If the gate flagged system design, do not lead with the employer whose design round is the hardest, however tempting the momentum feels.
- **A pass is a snapshot, not a guarantee.** It says the material composes today. If several weeks pass before your first real loop, re-run the gate rather than assuming it holds.
- **Do not skip the early waves because you passed.** The sequencing exists because of where you convert and where you get the cheapest live rehearsal, not because of whether you are theoretically ready. A gate pass is evidence about you; it is not evidence about the market.

**What a failed run unlocks is more useful than it feels.** It produces a routed, specific remediation list rather than a vague sense of unreadiness — which is exactly the difference between "I should study more" and "my behavioural stories have no numbers and here are the three I need to compute." That is a week of work with a defined end, and it is why the gate is worth running early and repeatedly rather than once when you feel ready.

## Perspectives

**The integration-testing view.** Lessons 05-08 are unit tests: each component exercised in isolation with clean inputs and no shared state. This is the integration test — all four together, under shared constraints, designed to surface the bugs that only appear under load. A codebase with complete unit coverage and no integration tests still ships bugs; a candidate with four strong lessons and no full-loop rehearsal still fails loops. In both cases you do not get to skip the integration test because the units passed.

**The self-assessment-bias view.** There is a well-documented gap between feeling ready and being ready, and it widens under self-study because the assessor and the assessed are the same motivated person. You want the preparation to have worked, and that want reshapes grading. The no-partial-credit rubric exists to replace a felt sense of "that went well" with concrete, quotable disqualifiers: did you or did you not name a GPU-specific constraint; did the story have a number or not. You cannot generously round a missing number up to "I basically said it." The rigidity is uncomfortable and that is the point — the discomfort of failing a gate you built yourself is far cheaper than failing a loop you cannot retry.

**The panel's view.** A real debrief does not average your best round against your worst; it treats inconsistency itself as a signal. A panel hearing "strong design, correctly distrusted the utilisation number, but the behavioural stories were vague on impact" does not conclude "close enough." They conclude "we do not have a consistent read on this person's level" — and at senior or staff, an inconsistent read is functionally a negative one, because the role requires reliability under exactly this cross-domain pressure. Someone sharp on design who goes soft explaining impact to stakeholders is an operational risk, not an interview quirk. This is why the gate scores pass/fail per dimension rather than an aggregate: it rehearses the actual shape of the judgement, not a friendlier approximation.

**The cost view.** The gate is cheap and the alternative is not. A failed self-run costs you 170 minutes and a week of targeted remediation. A failed real loop costs the opportunity, a cooldown measured in quarters, and — because the market is small — a data point about you that persists. The asymmetry is large enough that the correct number of self-run gates is "more than one," and the correct time to run the first one is *before* you feel ready, because a failed early gate is a routed backlog and a passed late gate is only a confirmation.

**The productisation view.** Watching what the industry chooses to automate is a proxy for what it considers baseline human reasoning rather than differentiation. Rank-level straggler detection and automated remediation now ship as product features — the same compute/comms/data bisection the debugging round drills by hand. That cuts two ways. It does not make the manual skill obsolete, because tooling breaks, tooling is sometimes wrong, and someone has to understand the mechanism well enough to override its verdict. But it does mean fumbling this reasoning reads worse than it once did: you are not being asked to invent a novel diagnostic, you are being asked to demonstrate fluency in something the industry has already decided is common enough to build into a product.

## Real-world use cases

- **Meta, *Revisiting Reliability in Large-Scale Machine Learning Research Clusters*
  (arXiv:2410.21680, HPCA 2025).** Eleven months across two clusters of roughly 16K and 8K A100
  GPUs. Measured mean-time-to-failure of **7.9 hours for 1,024-GPU jobs**, with 1.8 h at 16,384 and
  0.23 h at 131,072 given as the paper's **projections**; and proactive lemon-node detection
  reducing large-job failure rates from about 14% to about 4%. **What it shows:** the operational
  reality behind weighting the debugging round so heavily — at that failure cadence, rapid bisection
  is a core job function rather than an interview exercise. It is also a live test of citation
  discipline in your own answers: quoting the projections as measurements is exactly the slip a
  careful interviewer catches.

- **NVIDIA `dcgm-exporter`, `etc/default-counters.csv`.** The presence metric ships enabled; the
  SM-breadth metrics ship commented out. **What it shows:** why D2 is the discriminating round. The
  scenario is not artificial — the metric that cannot see the failure is the industry default, so
  most fleets genuinely are in this state, and a candidate who has operated one knows it.

- **NVIDIA `dcgm-exporter`'s shipped Grafana dashboard** (grafana.com #12239): eight panels, none of
  them SM breadth. **What it shows:** dimension 5's standard in miniature. If a vendor's own default
  dashboard leads with the misleading metric, a portfolio whose front door explains *why that is a
  problem* is doing work that a stranger can verify in thirty seconds without you present.

- **OpenCost issues #3900 and #3828.** The project's own record of the request-versus-usage cost
  basis gap, and an unaffiliated user asking for fractional-GPU costing. **What it shows:** the
  material that makes dimension 4 answerable under a rigor probe — when the interviewer points at
  your cost artifact and asks "why did you build this rather than use the standard tool," the answer
  is a citation rather than an opinion.

- **Mock-interview services staffed by working engineers.** **What it shows:** a practical route to
  the external calibration rep in §5. The value is not the practice — you can practise alone — it is
  a second scorer whose leniency drift is uncorrelated with yours. Use one to check whether your own
  5/5 is a real 5/5.

## Worked example

**A filled-in scorecard from one real run.** P2 for the design round, D2 for debugging, three STAR
stories chosen by a peer. Mixed result, which is the common one on a first attempt.

**Round 1 — screen and artifact walk. PASS.**
Forty-five-second positioning sentence landed: specialisation, the controller, the conservation
check, the simulated-fleet caveat. Domain Q&A crisp. The five-minute value-chain walk came in at
4:40; when the peer pointed at the Kueue showback artifact and asked what decision it drove, the
answer had a decision (showback before chargeback), a tradeoff (legitimacy over speed, with the
reversal condition) and a number (the holder-versus-user reshuffle). All three present.

**Round 2 — system design, P2. PASS with one blemish.**
The universal opening landed at 00:40 and all four global dimensions were banked by 02:20 — scale,
cost with a basis and date, the SLO stated as explainability rather than accuracy, and the dominant
failure mode. Covered the telemetry path, both ledgers, the formula and its three flaws volunteered
unprompted, the sharing regimes with the exposure fraction quantified, and the showback sequencing.
**Missed one axis:** Prometheus cardinality never came up until the peer prompted it at 31:00. Every
axis eventually ≥1, five axes defended with a tradeoff and a reversal condition. Scored pass, on the
grounds that the prompted axis was not the GPU-specific layer and all four global dimensions were
volunteered — but recorded as a blemish, because a prompted axis is a 1 and not a 2.

**Round 3 — debugging, D2. PASS.**
Rejected the presence metric in the first fifteen seconds with a mechanism, not an adjective: field
203, a passthrough of a driver counter, a threshold at one, no notion of SM count. Established
ground truth from throughput rather than device metrics. Checked for metric absence before hunting a
workload cause — this had been a fail on the previous run and it fired correctly this time. Read the
shape across breadth, tensor and memory activity, landed in the stall quadrant, went to per-rank
step times, found the outlier, and separated "which rank" from "why that rank" before proposing the
load diagnostic on the drained node. Closed on prevention with a specific alerting policy including
the absence rule. Every command was preceded by an expected signal and a branch.

**Round 4 — three STAR stories. FAIL.**
- *Story A, cross-team influence:* metric present and derivable (the reshuffle: 2.7× the GPUs for
  two-thirds of the work), tradeoff present (legitimacy over speed), scope explicit (three groups, a
  dollar figure in the first sentence), reversal volunteered. **Pass.**
- *Story B, incident command:* metric present (time to diagnosis, and the standing detection rule
  that came out of the RCA), tradeoff present, scope present. **Pass.**
- *Story C, the observability rebuild:* strong narrative, clearly the best-rehearsed story of the
  three — **and no number.** The recording contains the phrase "throughput improved a lot."
  **Fail.** Number fatigue, exactly as predicted: two hours and eleven quoted figures earlier, and
  the retrieval simply did not happen.

**Round 5 — portfolio. PASS**, checked separately with a stranger's four-minute read. They correctly
stated what it was, named the strongest number, and clicked the flagship post. One dead link found —
a directory link without the explicit `README.md` — fixed the same day; recorded as a pass because
the reader's three questions were all answered, with a note.

**Result: 4/5. Behavioural fails on story C. Not a gate pass.**

**Remediation list, routed:**

1. **D3 (the blocker).** Story C has no number. Routes to L08 §2's Result slot and practice item 2.
   Fix: rebuild story C around the measured presence-versus-breadth figures and the 2.9× batching
   result, write the derivation beside each, and re-drill until the metric line comes out first
   rather than last. Add the number rehearsal from §7 to the round-4 break.
2. **D1 (polish, not a fail).** The cardinality axis was prompted rather than volunteered. Routes to
   L05's P2 card: add one sentence so the tenant × GPU × model cardinality budget is volunteered in
   the close rather than waiting for a probe.
3. **D5 (hygiene).** One directory link without `README.md`. Routes to L03 §8. Fixed; add the link
   check to the pre-run checklist so it cannot recur.
4. **Re-run the full 170 minutes** — not round 4 alone — on a named date, with a different P-prompt
   so the design round is not a repeat.

**What this run demonstrates about the gate itself:** three of the four rounds were clean, the
failure was a single missing number in a story the candidate knew cold, and the cause was
predictable interference rather than a knowledge gap. That is the exact class of failure a
slice-by-slice rehearsal cannot find, and it is why the gate is a full loop.

## Practice

1. **Run one full timed loop end to end** — 35 + 45 + 45 + 45 with five-minute breaks, recorded, no
   notes, no pausing, no re-takes. Pick one of P1–P6 for round 2; round 3 is always D2.

2. **Fill the scorecard from the recording**, at least an hour after the run and preferably the next
   morning. For every fail, quote yourself. "I basically said it" is not an entry.

3. **Produce the routed remediation list** — one line per fail, with the lesson section it routes
   to and the specific fix. A remediation item without a named section is not routed.

4. **Re-drill the routed sections, then re-run the whole loop.** Change the P-prompt so the design
   round is not a memory test.

5. **Run dimension 5 with a stranger.** Someone who does not know the project reads the README for
   four minutes and then answers three questions: what does this do, what is the strongest number,
   what would you click next. Every wrong answer is a specific edit, not a general improvement.

6. **Get one external calibration rep** if feasible, and compare their dimension-by-dimension
   scoring against your own. Treat a consistent gap as evidence your self-scoring has drifted, and
   discount your own results by that gap until it closes.

7. **Apply the fatigue counters** on the second run and note whether the round-4 failure mode
   changes. If number fatigue was your failure, the twenty-second metric rehearsal should visibly
   fix it — and if it does not, the problem is that the numbers are not yet yours.

8. **Bank the scored record and the remediation list** as the readiness artifact, and fold both into
   the [GPU platform capstone](../practice/gpu-platform-capstone/README.md).

**Acceptance:** at least one fully scored run with a routed remediation list · every fail signal
quoted verbatim from the recording · dimension 5 verified by a stranger's four-minute read · a
second run after remediation, with a different design prompt · ultimately, five dimensions passing
in one unaided, unbent run.

## Common pitfalls

1. **Running the loop generously.** **Mechanism:** self-administered mocks drift lenient over
   repeated attempts, because the assessor is motivated for the assessed to pass — you pause "just
   this once," you retry a bad round, you round a shaky answer up because you know what you meant.
   Each act is individually small and cumulatively fatal, because it produces a *record* that says
   pass. **Fix:** the void rules, applied procedurally rather than by willpower. If you bend a rule,
   restart the loop; do not patch it.

2. **Re-running only the failed round.** **Mechanism:** the gate is not testing four independent
   skills, it is testing whether they survive the sequence and the fatigue together — a behavioural
   round rehearsed fresh proves it works rested, which is precisely what was never in doubt.
   **Symptom:** you fix story C, deliver it perfectly on its own, and call it a pass. **Fix:** the
   whole 170 minutes, with a different design prompt.

3. **Grading a known-but-unsaid answer as a pass.** **Mechanism:** a real interviewer has no access
   to your internal state, only to your output, and the rubric deliberately mirrors that.
   **Symptom:** you justify a pass with "well, I did know that." **Fix:** that sentence *is* the
   fail signal.

4. **Scoring from memory instead of the recording.** **Mechanism:** memory reconstructs your intent;
   the recording contains your output, and they diverge most on the dimensions being scored — how
   long before you volunteered cost, whether the number was actually said. **Symptom:** your scored
   sheet disagrees with the recording when you finally check.

5. **Scoring immediately after the run.** **Mechanism:** you are still inside the experience and
   cannot hear it; relief or frustration both distort. **Fix:** an hour minimum, next morning
   preferably.

6. **Running it at the end of a working day.** **Mechanism:** you then measure whether the material
   survives fatigue *plus* eight hours of unrelated work, which is not the condition being tested,
   and you will remediate against a phantom. **Fix:** run it when a real loop runs.

7. **Treating a marginal pass as a pass.** **Mechanism:** a self-run loop is genuinely less tiring
   than a real one — no social pressure, no strangers, no stakes — so a round you scraped through
   here you would not have scraped through in a room. **Fix:** treat marginal as fail on the
   dimension, and note it.

8. **Waiting to feel ready before the first run.** **Mechanism:** the gate's most valuable output is
   the routed remediation list, which is only available from a *failed* run; a first attempt made
   when you already feel ready produces confirmation rather than information. **Fix:** run it early
   and expect to fail it.

9. **Opening the campaign on a dimension you just failed.** **Mechanism:** momentum after a
   near-pass is persuasive and wrong — cooldowns are long and the loop you lead with is the one you
   are least rehearsed for. **Fix:** lesson 01's sequencing, which a gate pass does not override.

## Self-check

- **Why is the gate one back-to-back 170-minute loop rather than four separately passed rounds?**
  *Answer:* because the failure mode being tested is **interference**, not competence in any one
  round. Passing slices proves each component works in isolation, rested, with clean inputs — which
  is what lessons 05-08 already established. A real loop runs four rounds back to back under a
  shared budget of attention and energy, and the predictable degradations all land late: working
  memory carrying over from the design round into the first two minutes of debugging, register
  lock-in leaving you technical when round four wants organisational scope, and number fatigue after
  two hours of quoting figures. Four of the six common interferences degrade the behavioural round
  specifically, which is why it is last in a real loop and last here.

- **What is the single most common disqualifier in the debugging round, and what is the correct
  move?** *Answer:* quoting the utilisation number as truth — treating "pegged at 100%" as evidence
  the GPUs are working hard. The correct move is to reject it with a *mechanism* rather than an
  adjective: it is a passthrough of a driver counter defined as the fraction of a short sample
  window during which at least one kernel was resident, a threshold at one with no notion of how
  many SMs exist, so it reads high on exactly the expensive failure modes. Then establish ground
  truth from throughput, check whether the honest metric is still being exported at all — profiling
  fields are dropped rather than zeroed, and absence is not zero in a query — and only then bisect
  compute against communication against data, with a named command and an expected reading at each
  branch, closing on prevention rather than on the diagnosis.

- **Three vivid, well-structured STAR stories and the behavioural dimension still fails. What most
  likely fired, and where does it route?** *Answer:* most likely a metric-free story — "throughput
  improved a lot" — or individual-contributor scope rather than multi-team scope; either fails the
  dimension regardless of polish. Both route to lesson 08: the missing number to §2's Result slot
  and the practice item that says compute it rather than soften the line, and the scope failure to
  §2's Situation slot, where scope has to appear in the first fifteen seconds and cannot be
  retrofitted. A third possibility is that all three stories were in the senior register — heroics,
  speed, intuition, individual save — which routes to §3's register-flip drill. If all three stories
  turned out to be the same artifact with different openings, that routes to lesson 07's
  portfolio-balance check and lesson 08's coverage map.

- **Why is re-running only the failed round wrong?** *Answer:* because the property under test is
  composition under fatigue, not the round in isolation. A behavioural round rehearsed fresh, first
  thing, on its own, proves it works rested — which was never the question. It proves nothing about
  whether it holds as round four after 125 minutes of design and debugging, which is exactly the
  condition it failed under. The corollary is that the re-run should also change the design prompt,
  or round two becomes a memory test rather than a design test.

- **What are the void rules, and why must they be procedural rather than willpower-based?**
  *Answer:* no notes, no pausing, no re-takes, and if a rule is bent mid-run the whole run is void
  and restarts rather than being patched. They must be procedural because the drift is structural,
  not a character failing: the assessor and the assessed are the same motivated person, and wanting
  the preparation to have worked reshapes grading in small, individually reasonable increments. The
  reason a patched run is worse than a void one is that it produces a *record* saying pass, and the
  record is what you consult weeks later when deciding whether to open a campaign. Two supporting
  rules: score from the recording rather than from memory, because memory reconstructs intent while
  the recording contains output; and score at least an hour later, because immediately afterwards
  you are still inside the experience.

- **A pass unlocks the campaign. What does it *not* unlock?** *Answer:* it does not override lesson
  01's sequencing. You still lead with the waves where your artifacts convert hardest rather than
  with your top choice, because per-loop performance rises with reps, cooldowns run six to twelve
  months, and a live offer is leverage everywhere else. It does not license opening on a dimension
  you failed on the previous run, however persuasive the momentum. And it is a snapshot rather than
  a guarantee: if weeks pass before your first real loop, re-run the gate rather than assuming it
  holds. Conversely, a failed run unlocks something genuinely more useful than it feels — a routed,
  specific backlog with a defined end, which is the difference between "I should study more" and
  "three stories need numbers and here is where to compute them."

- **Name three fatigue counters and what each one targets.** *Answer:* the **unload** at every
  break — sixty seconds of writing down where you stopped and what you were still holding, then
  physically closing it, which targets working-memory carry-over from round two into round three's
  opening minutes, where the cheap diagnostic check lives. The **number rehearsal** before round
  four only — reading aloud just the metric line of each of your three stories, twenty seconds —
  which targets number fatigue, the most common cause of a failed behavioural dimension, and works
  because that failure is one of retrieval rather than knowledge. And the **register reset** — one
  sentence spoken aloud about people and decisions rather than systems — which targets register
  lock-in after three consecutive technical rounds, in the thirty seconds where scope gets decided.
  Plus one negative rule: do not run the gate at the end of a working day, or you measure the wrong
  fatigue and remediate against a phantom.

## Connections & what's next

This lesson is the seam that closes the module and the course. It draws directly on 05 (the P1–P6
skeletons that fill round two), 06 (the D2 tree that fills round three), 07 (the artifact narration
scored as its own dimension, and the derivations that survive a rigor probe), and 08 (the staff STAR
bank that fills round four). Nothing in the loop is new material — it is those four lessons placed
under the timing and fatigue of a real onsite. It also closes back to 01: the sequencing strategy
only pays off once the gate is clear, and the remediation table routes every possible failure to a
specific earlier section, so a failed run is never a dead end.

There is no lesson 10. Per the module [README](../README.md), the next step after a clean
five-dimension pass is not another lesson — it is opening the campaign: finishing the
[GPU platform capstone](../practice/gpu-platform-capstone/README.md) into its published form if it
is not there already, and applying, leading with the waves where your eleven artifacts convert
hardest. The [checkpoint](../checkpoint.md) for this module is this same gate, formalised as the
whole course's final checkpoint. This is the last rep before that door opens.

## References & further reading

**Primary sources**

- Meta — *Revisiting Reliability in Large-Scale Machine Learning Research Clusters* (arXiv:2410.21680, HPCA 2025) — https://arxiv.org/abs/2410.21680 — read for: the measured 7.9-hour MTTF at 1,024 GPUs that justifies the debugging round's weight, the lemon-node result (~14% to ~4% on large-job failures), and the fact that the larger-scale MTTFs are projections. *Correction vs earlier versions of this lesson, which presented the 1.8-hour figure as a measurement and attributed it to an H100 fleet: it is a projection, and the clusters studied are A100-based research clusters.*
- NVIDIA `dcgm-exporter`, `etc/default-counters.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — read for: why the D2 scenario is realistic rather than contrived — the metric that cannot see the failure is the shipped default.
- NVIDIA `dcgm-exporter`, `grafana/dcgm-exporter-dashboard.json` — https://github.com/NVIDIA/dcgm-exporter/blob/main/grafana/dcgm-exporter-dashboard.json — read for: the vendor's own default dashboard, useful as the contrast your portfolio's front door is measured against in dimension 5.
- OpenCost issues #3900 and #3828 — https://github.com/opencost/opencost/issues/3900 · https://github.com/opencost/opencost/issues/3828 — read for: the citations that make dimension 4 answerable under a rigor probe when an interviewer asks why you built rather than adopted.

**Course-internal sources — everything the loop is assembled from**

- `platform-eng/modules/12-capstone-interview/lessons/05-system-design-drills.md` — the six prompts and the scoring rubric that round two is graded against.
- `platform-eng/modules/12-capstone-interview/lessons/06-debugging-drills.md` — the D2 tree, the command→signal→branch table, and the live-terminal protocol behind round three.
- `platform-eng/modules/12-capstone-interview/lessons/07-narrate-artifacts.md` — the four-beat frame and the compression ladder behind dimension 4.
- `platform-eng/modules/12-capstone-interview/lessons/08-staff-behavioral.md` — the staff-STAR template and the story bank behind round four.
- `platform-eng/modules/12-capstone-interview/lessons/03-portfolio-writeup.md` — the cold-read model and publishing hygiene behind dimension 5.
- `platform-eng/modules/05-gpu-observability/lessons/01-lie-and-truth.md` — the field-level knowledge underneath dimension 2; a failure there is usually a gap here rather than a narration problem.

**Not relied upon**

- Mock-interview platforms, interview-preparation guides and first-person "my interview experience"
  write-ups were consulted while calibrating the loop's shape and timings. The round structure above
  is presented as a composite of the recurring pattern in this market rather than as a claim about
  any named employer's process, and no timing or scoring criterion depends on those sources. Vendor
  product pages describing automated straggler detection informed the productisation observation in
  Perspectives; that trend is stated as a general industry pattern, not as a verified claim about
  any named product's current features.
