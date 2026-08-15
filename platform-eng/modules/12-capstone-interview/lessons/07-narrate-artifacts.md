---
lesson: 07
title: "Narrate your artifacts"
module: 12
concept: "artifact walk-through delivery"
status: not-started
est_time: "4 hrs"
prev: "06-debugging-drills.md"
next: "08-staff-behavioral.md"
artifacts: ["3-min + 30-sec + deep-dive narration scripts for the 3 lead artifacts, each self-scored for number + tradeoff", "written follow-up-defense answers for 05, 02, and 11"]
sources: 5
---

# Narrate your artifacts

## Where this fits

Lesson 06 built your debugging reflexes — the decision trees and first-move commands you reach for live, on someone else's box, when a GPU host is misbehaving in front of an interviewer. That lesson is about reasoning under pressure with no script. This lesson is the opposite kind of round: the one where you *do* have the material — 11 built artifacts, all your own work — and the only thing standing between you and a staff-signal answer is how you narrate it. The gap 07 fills is turning a finished build into a spoken decision, so "walk me through a project you're proud of" converts your depth instead of burying it in a chronological build log.

## Why this matters

You built 11 artifacts that are, almost literally, the neocloud interview syllabus. That edge is worthless in the room if the story comes out as a chronological build log — "so first I wrote an exporter, then I added labels, then I wrote a controller." A hiring manager can't grade a build log. They can grade a decision. The gap between "I built a thing" and "I identified an expensive, invisible problem, made a defensible call under a real constraint, and moved a number" is exactly the gap between a mid-level narration and a staff-signal one — and it's a *delivery* gap, not a substance gap. The substance is already on disk.

The specific failure mode for someone with your depth is the heroics arc: you have genuinely hard technical stories, so it's tempting to narrate them as "it was broken and hard and I fought through it." That reads as senior-at-best — execution under adversity. The staff reframe is to narrate the same event as a *judgment*: which problem was worth solving, what you traded away to solve it, and how you'd know if you were wrong. Same artifact, different spine. This lesson installs that spine on your five sharpest artifacts so that "walk me through a project you're proud of" — which you will get in every single loop — converts instead of merely describing.

It also matters because this is a rehearsed-delivery skill, not a knowledge skill, which means it's cheap to fix and expensive to skip. Nobody fails this round because the artifact wasn't good enough. People fail it because they narrated a strong artifact as a weak story — under time pressure, in front of a stranger, with no second take. The fix is mechanical: install the frame, rehearse it out loud, and the strength of the underlying work does the rest.

## What's new here (calibration)

- **Skip** (you own it): STAR mechanics, "how to tell a story," pacing, filler-word discipline, how to sound calm. You are sharp on generic delivery.
- **New**: the **artifact → decision → tradeoff → metric** frame — a 4-beat spine tuned so every artifact story lands a *number* and a *tradeoff*, never a heroics arc.
- **New**: the **"what problem was worth solving"** opening move — leading with judgment about *which* problem, which is the staff tell an execution narration buries.
- **New**: **compression control** — the same story as a 30-second version, a 3-minute version, and a deep-dive, so you fit the question instead of monologuing.
- **New**: **follow-up defense** — the reversibility / assumed-constraint answers to "why not time-slicing?" and "what would you do differently?" that turn a challenge into more staff signal.

## Core concepts

### The frame: artifact → decision → tradeoff → metric

Four beats, ~3 minutes, in this order:

1. **Artifact — but open on the *problem*, not the build.** One sentence that names the expensive, invisible thing. "Our GPUs looked 95% busy on every dashboard, but the fleet was doing about a third of the useful work we were paying for." You have earned the right to state the problem crisply because you measured it — so state it, don't build up to it.
2. **Decision — the call you made, and the fork you were standing at.** "I decided the honest signal wasn't utilization, it was model-FLOPs-utilization / goodput, and I built the exporter to surface that instead." Name that there *was* a choice.
3. **Tradeoff — what you gave up, and the constraint you assumed.** This is the beat mid-level narrations skip. "MFU is harder to compute and it makes the fleet look *worse* to leadership, which is politically expensive — I traded a flattering number for a true one, on the assumption that a true number I could act on beat a green dashboard nobody trusted."
4. **Metric — the number, and ideally a second-order number.** "The goodput dashboard showed 31% MFU against 95% allocation; that gap was the business case for the reclaim work that followed."

The discipline: **every artifact story must land a number and a tradeoff.** If your narration has no number, it's an anecdote. If it has no tradeoff, it's a heroics arc and it caps you at senior. Score every rehearsal against exactly those two.

It's worth naming what this frame deliberately is not. Formation.dev's engineer-specific storytelling guide proposes a broader "Context → Constraints → Decisions → Outcomes → Learnings" arc as an alternative to generic STAR for technical interviews — and it's a reasonable frame. This lesson's 4-beat version is narrower and more metric-forward on purpose: it compresses "context" and "constraints" into a single problem-statement beat, and it makes the number non-optional rather than folded into "outcomes." The narrower frame is a better fit for artifact walk-throughs specifically (where you're narrating a finished build against a fixed clock) than for open-ended behavioral prompts (where 08's staff-tuned STAR template, with its own scope/ambiguity/tradeoff/reversibility slots, is the better tool). Know both, and reach for the one that matches the round.

### The "what problem was worth solving" opening

Staff loops are scoring judgment about *which* problem, not just clean execution of a given one (lesson 08 makes this the crux). So the first sentence of every narration should answer "why was this worth your time," not "what did you build." Compare:

- Execution framing (caps at senior): *"I built a Kubernetes operator that attributes GPU cost per pod."*
- Judgment framing (staff): *"GPU spend was our second-biggest infra line and nobody could tell you which team or which pod caused a dollar of it — so I built an operator that makes per-pod GPU cost a first-class, queryable fact."*

Same artifact (02). The second version tells the interviewer you can *find* the problem worth solving, which is the thing they can't teach and are specifically probing for. This is precisely the instinct Lalit Maganti, a staff engineer on Chromium/Perfetto, describes as the core of the role in his own account of the job: staff-level work is disproportionately about noticing which problems are worth anyone's time before a ticket exists for them, not about executing well-specified tickets faster than everyone else. Your opening sentence is where you prove you have that instinct, before you've said a word about the build.

### The five lead narrations

**05 — "your GPU dashboard is lying" (your single best opener).**
- *Problem:* every dashboard showed ~95% GPU utilization; the fleet's actual useful throughput was far lower.
- *Decision:* utilization (SM-active) is the wrong metric — I defined and instrumented goodput / MFU as the real one.
- *Tradeoff:* MFU is harder to compute and makes the fleet look worse politically; I chose the true, actionable number over the flattering one.
- *Metric:* 95% allocation vs 31% MFU — that gap became the business case for the reclaim and rightsizing work.
- *Why it leads:* it's a judgment-and-measurement story with a shocking number and a political tradeoff baked in. It signals you see through vanity metrics — exactly the AI-lab and neocloud tell.

**02 — the gpu-cost-operator.**
- *Problem:* GPU spend was a top infra line with zero per-team, per-pod attribution — unbudgetable, unaccountable.
- *Decision:* built a real Kubernetes controller that reconciles per-pod GPU cost as a first-class fact, not a spreadsheet reconciliation after the fact.
- *Tradeoff:* a controller is operational surface area (it can lag, it can be wrong, it's another thing on-call owns) versus a cheaper offline batch job — I took the ownership cost to get live, trustworthy, queryable attribution.
- *Metric:* attribution moved from "nobody knows" to per-pod dollars, which unlocked the showback in 06.
- *Why it leads:* it's the literal "custom controllers and operators" bullet on the CoreWeave/Nebius JDs, and it's an architecture-judgment story, not a scripting story.

**11 — the FOCUS synthesis / found-the-gap-in-the-OSS-tool.**
- *Problem:* the leading open-source GPU-cost tool bills *whole-GPU requests* — so fractional and shared-GPU workloads are mis-billed, and it doesn't align to the FOCUS spec finance actually wants.
- *Decision:* I read the tool's source, isolated exactly where whole-GPU billing is baked into the allocation path, and designed a FOCUS-aligned fix.
- *Tradeoff:* aligning to FOCUS and fractional billing adds real complexity and a divergence-maintenance cost against upstream; I judged the correctness-and-finance-interop win worth it over living with the simpler wrong number.
- *Metric:* quantify the mis-billing on a shared node (e.g. a 4-way-shared GPU billed 4× under whole-GPU logic) and the FOCUS-conformance delta.
- *Why it leads:* reading someone else's source, finding the design gap, and proposing a spec-aligned fix is *pure* staff signal — it's cross-domain (systems + finance), it's judgment about correctness, and it shows you operate above "I use tools" at "I evaluate and correct tools."

**08 — survive-a-failure.**
- *Problem:* a GPU host / job failure that silently halves throughput or wedges a training run, with no runbook.
- *Decision:* built a reproducible failure lab and a diagnostic order-of-operations (nvidia-smi → Xid/dmesg → NUMA/topo → thermal/power → link width).
- *Tradeoff:* investing in a repeatable lab and runbook up front versus firefighting each incident ad hoc — I front-loaded the cost to make the fleet debuggable by anyone, not just me.
- *Metric:* mean-time-to-diagnose, and the class of failures now caught before they cost a training run.
- *Why it leads:* it's the Lambda live-Linux-round and CoreWeave-SRE story, and it doubles as incident-command material in lesson 08.

**06 — Kueue showback (cross-team attribution agreement).**
- *Problem:* three teams shared a fleet, each convinced the others were the wasteful ones; no agreed model to settle it.
- *Decision:* drove a showback-first attribution model — publish the numbers and build trust before any chargeback teeth — and got finance, ML, and platform to agree on it.
- *Tradeoff:* showback-first is slower and has no immediate enforcement (people can ignore a report) versus going straight to chargeback; I traded speed for legitimacy, because a chargeback nobody agreed on would've been litigated to death.
- *Metric:* the agreed model + the waste gap it exposed per team.
- *Why it leads:* this is your **influence-without-authority** story — you moved three orgs to a shared decision. That's the single hardest staff competency to evidence and you have it concretely.

### Compression control: 30-sec / 3-min / deep-dive

Same story, three depths — match the question, don't monologue.

- **30-second (the "quick, what have you worked on" / opener):** just beats 1 and 4. Problem + number. *"Every dashboard said our GPUs were 95% busy; I proved the fleet was actually at 31% MFU and built the goodput dashboard that became the case for reclaiming the difference."* Then **stop** and let them pull the thread.
- **3-minute (the default "walk me through it"):** all four beats, one layer of mechanism each. This is your rehearsed spine.
- **Deep-dive (they lean in / it's a domain round):** hold the same four beats as your skeleton, but expand the *decision* and *tradeoff* beats with the alternatives you rejected and why. The number stays the anchor you keep returning to.

The rule: **let the interviewer set the depth.** Deliver the 30-second version, then breathe. If they ask "tell me more," you go to 3-min. If they ask "how did you compute MFU," you're in deep-dive. Volunteering the deep-dive unprompted is the most common way strong candidates read as unable to calibrate scope — itself a negative staff signal.

Think of the three depths as the same content rendered at three resolutions, not three different stories — the discipline is closer to a product-thinking exercise (what's the minimum viable answer for this question?) than a storytelling one. If you find yourself reaching for the deep-dive version by default, that's a signal you've only actually built one version of the story and are compressing on the fly, badly, under pressure. Write all three out in advance so the short version isn't just the long version trailing off.

### Follow-up defense: reversibility and assumed-constraints

Follow-ups are not attacks; they're the interviewer handing you a second helping of staff signal. Two archetypes:

- **"Why not X?" (e.g. "why not just use time-slicing / MIG / the off-the-shelf tool?")** Answer with the *tradeoff and the assumed constraint*, never "I didn't think of it." *"Time-slicing would've raised utilization on paper but not goodput — it packs more jobs onto a GPU without making any of them do more useful FLOPs, so it would've masked the exact problem I was measuring. Given my constraint — surface truthful throughput — it was the wrong lever. If the constraint had been raw occupancy for cheap inference, I'd have reached for it."* You've now shown you knew the alternative, understood its tradeoff, and made a constraint-driven call.
- **"What would you do differently?"** Answer with **reversibility**, not confession. Name a call you made under time/info pressure, say what new information would flip it, and note whether it was a one-way or two-way door. *"I hard-coded the H100 FLOP ceiling for the MFU denominator — a two-way door I chose for speed. With more time I'd make the device profile pluggable, because the day we mixed in a different accelerator that assumption silently breaks the number. It was cheap to reverse, so shipping the assumption was the right call at the time."* That reframes hindsight as judgment-under-uncertainty — the staff frame — instead of an admission of a mistake.

Treat every unprompted "why not X" as a gift rather than a challenge: it is the interviewer explicitly inviting you to demonstrate that you know the alternative, weighed it, and can defend the constraint you optimized for. Candidates who read it as adversarial tend to get defensive or hedge ("well, I guess time-slicing could also have worked"); candidates who read it as an invitation state the tradeoff crisply and move on. The tone difference alone is a large part of the staff signal here — the content of a good answer and a merely-adequate one is often nearly identical.

## Perspectives

**The interviewer's second question.** What an interviewer asks *after* your 3-minute narration tells you more about how you're being scored than the narration itself. "Tell me more" is a scope-and-interest probe — they want the deep-dive, and if you have one ready you convert. "How did you actually compute that number" is a rigor probe — they're checking the metric wasn't decorative, and this is where a hand-wavy MFU definition gets exposed fast. "Why didn't you just—" is a tradeoff probe, covered above. Learn to hear which kind of question you're getting, because the three want different depths and answering the rigor probe with more scope (or vice versa) reads as not listening.

**The compression view.** Building all three depths in advance is itself a small design exercise: what's the minimum information that still lands the number, and what's the maximum detail that still fits three minutes without losing the thread? Treating this as a product problem — rather than "how do I remember to talk less" — produces noticeably tighter answers, because you're forced to rank the mechanism details by how much staff signal they actually carry, and cut the ones that are just interesting to you but not load-bearing for the listener.

**The adversarial-follow-up view.** Reframe every "why not X" and "what would you do differently" as free staff-signal delivery rather than something to survive. The interviewer doesn't ask these questions to catch you out — they ask because the 3-minute narration alone can't distinguish "made a real tradeoff" from "got lucky and didn't consider alternatives," and the follow-up is the only way to find out which. Walking in expecting these questions, and having genuine tradeoff language ready rather than improvised, is the single highest-leverage prep move in this lesson.

**The portfolio-balance view.** You have five strong lead narrations, not one. The temptation, especially under interview stress, is to steer every open-ended question back toward the artifact you rehearsed hardest — usually 05, because it's the most dramatic. Watch for this in yourself during mocks: if three different prompts ("proud of," "hardest problem," "influenced without authority") all produce the same story with a different opening sentence, you haven't built a portfolio, you've built one story with several doors into it. Rehearse all five to roughly equal readiness so the interviewer's actual question — not your comfort artifact — decides which one you tell.

## Real-world use cases

- **Lalit Maganti, "How I Find Problems to Solve as a Staff Engineer"** — https://lalitm.com/post/find-problems-staff-engineer/ — a working staff engineer's first-person account of exactly the instinct behind this lesson's opening move: staff signal comes from identifying which problem is worth solving, not from executing a given one well.
- **Formation.dev, "The storytelling skills senior engineers need for stronger interviews"** — https://formation.dev/blog/the-storytelling-skills-senior-engineers-need-for-stronger-interviews/ — proposes an engineer-specific "Context → Constraints → Decisions → Outcomes → Learnings" alternative to generic STAR; useful as a contrast case for why this lesson's 4-beat frame is deliberately narrower and metric-forward.
- **Sean Goedecke, "How I got promoted to staff engineer twice"** — https://www.seangoedecke.com/staff-engineer-promotions/ — a real account of converting ordinary project work into a staff-legible narrative over two separate promotion cycles, showing the framing move is repeatable, not a one-off trick.

## Worked example

Full 3-minute narration of **05**, annotated by beat:

> **[Problem, ~25s]** "About a year in, I got tired of a contradiction: leadership was told our GPU fleet ran at 90-95% utilization, and simultaneously our training throughput and our cost-per-token were nowhere near what that number implied. If we were really 95% busy, where was the work going? So I went to measure what 'busy' actually meant.
>
> **[Decision, ~35s]** What I found is that every dashboard was reading SM-active — the GPU had *a* kernel resident — which says nothing about whether the hardware is doing useful math. The honest metric is model-FLOPs-utilization: actual useful FLOPs over the device's peak. So I decided to stop reporting utilization and instrument goodput/MFU instead, and I built an exporter that surfaces MFU per job alongside allocation.
>
> **[Tradeoff, ~40s]** This was not a free call. MFU is harder to compute — you need the model's FLOP count and the device ceiling — and, more to the point, it made the fleet look dramatically *worse* to the people who'd been quoting 95%. I was trading a flattering number that everyone had stopped trusting for an ugly number I could actually act on. I made that trade on the assumption that a true, actionable signal beats a green dashboard — and I de-risked the politics by pairing the ugly number with the reclaim opportunity it exposed, so it landed as 'here's money on the table,' not 'your metric was a lie.'
>
> **[Metric, ~30s]** The headline was 95% allocation against 31% MFU. That two-thirds gap was the entire business case for the rightsizing and idle-reclaim work that followed — it turned 'the fleet is basically full, we need to buy more' into 'the fleet is a third utilized, let's reclaim before we spend.' The goodput dashboard became the number leadership actually steers on now."

Score it: number present (31% MFU vs 95%, plus the reframed capex decision) — yes. Tradeoff present (flattering-vs-true, political cost, assumed constraint) — yes. Heroics arc — no; it's a judgment story. That's a passing staff narration. Now notice you can cut it to the 30-second version by keeping only the first and last blocks, or expand it by adding "I considered just fixing the dashboard's threshold, but the metric itself was wrong, not its threshold" into the decision beat.

## Practice

Feeds the capstone deliverable at [GPU platform capstone](../practice/gpu-platform-capstone/README.md):

1. **Record 3-minute narrations for the 3 lead artifacts (05, 02, 11).** Actually record audio or video — do not just write them. Time each; if you're over 3:30 you're padding, cut mechanism not tradeoff.
2. **Self-score each recording** against the two-item rubric: (a) did it land a concrete number? (b) did it name a tradeoff and an assumed constraint, or did it drift into a heroics arc? Re-record any that miss either.
3. **Write the 30-second and deep-dive versions** of each of the three, so you have all three depths on tap. Verify the 30-sec is beats 1+4 only and ends on a hook.
4. **Draft the follow-up defenses**: for 05 write the "why not time-slicing?" answer; for 02 the "why a controller instead of a batch job?" answer; for 11 the "why diverge from upstream instead of filing an issue?" answer. Each must resolve to a tradeoff or a reversibility note, never "I didn't consider it."
5. Add 08 and 06 as 3-minute narrations once the lead three are clean — 06 is your influence-without-authority story and pairs directly into lesson 08's behavioral bank.
6. **Run the portfolio-balance check**: have someone (or your own recording, replayed cold) ask you "a project you're proud of," "your hardest technical problem," and "a time you influenced without authority" back to back. If two of the three land on the same artifact, deliberately re-route one to a different lead narration and re-record.

## Common pitfalls

- **Volunteering the deep-dive when asked the 30-second question.** This is the most common way a strong candidate reads as unable to calibrate scope. The fix is mechanical: deliver beats 1+4, stop, and wait. If the room wanted more, they'll ask.
- **Treating the tradeoff beat as purely technical.** "MFU is harder to compute" alone is an engineering caveat, not a tradeoff. A tradeoff that lands staff signal has an organizational or political cost attached too — "and it makes the fleet look worse to the people quoting 95%." A tradeoff with only an engineering cost under-delivers.
- **Answering "what would you do differently" with a genuine confession.** "Honestly, I'd have structured the exporter better" is candid but reads as an unresolved regret, not a judgment call. Reframe every such answer around reversibility: name the two-way door, name what would flip it, and state why shipping the assumption was still correct at the time.
- **Over-preparing a single artifact.** If 05 is your best-rehearsed story, every open-ended prompt starts to get awkwardly redirected back to it, and a panel that's heard "well, actually, let me tell you about the dashboard" three times in one loop notices. Rehearse all five lead narrations to comparable fluency so the question — not your comfort artifact — picks the story.

## Self-check

- What are the four beats of the narration frame, and which two things must *every* artifact story land? **Answer:** Artifact (opened on the problem) → decision (the call and the fork) → tradeoff (what you gave up + the assumed constraint) → metric (the number). Every story must land a **concrete number** and a **tradeoff**; a story with no number is an anecdote, and one with no tradeoff is a heroics arc that caps you at senior.
- Why does opening on "what problem was worth solving" read as staff, and how would you re-open "I built an operator that attributes GPU cost per pod"? **Answer:** Staff loops score judgment about *which* problem, not just clean execution of a given one, so leading with why-it-was-worth-your-time shows the thing they can't teach. Re-open as: "GPU spend was a top infra line and nobody could attribute a dollar of it to a team or pod — so I made per-pod GPU cost a first-class, queryable fact." The build detail — that it's a Kubernetes controller — comes second, in service of the problem, not as the headline.
- A Together-style interviewer challenges your 05 story with "why not just use time-slicing to raise utilization?" What's the staff-shaped answer? **Answer:** Name the tradeoff and the constraint, not a blank: time-slicing raises on-paper occupancy without raising goodput — it packs more jobs onto a GPU without any of them doing more useful FLOPs, so it would have *masked* the exact problem I was measuring. Given my constraint (surface truthful throughput) it was the wrong lever; had the constraint been cheap inference occupancy, I'd have reached for it.
- What's the difference between this lesson's 4-beat frame and Formation.dev's "Context → Constraints → Decisions → Outcomes → Learnings" frame, and when would you use each? **Answer:** The 4-beat frame is narrower and makes the metric non-optional — it's built for narrating a finished artifact against a hard time limit. Formation's frame is broader and better suited to open-ended behavioral prompts where scope and ambiguity (not just a build's outcome) need to be established, which is closer to lesson 08's staff-tuned STAR template.
- Why is volunteering the deep-dive version to a 30-second question a negative signal even though the content is accurate? **Answer:** Because staff loops are also scoring whether you can calibrate scope to the question being asked. Delivering unrequested depth reads as an inability to read the room, regardless of how good the underlying material is — the discipline of stopping after beats 1+4 and letting the interviewer pull the thread is itself part of what's being graded.

## Connections & what's next

This lesson turns the artifacts built across modules 1-11 into the spoken layer of the capstone: 05, 02, 11, 08, and 06 are the five narrations you'll actually deliver in a loop, and the follow-up defenses you draft here are the same reversibility and assumed-constraint language lesson 08 formalizes into a full STAR template. The 06 narration in particular is doing double duty — it's both a lead artifact story here and the anchor of the influence-without-authority competency in the next lesson, so getting it clean now saves rework later. Next: lesson 08 takes the tradeoff/reversibility instincts you just installed and applies them to the dedicated behavioral round, scored on a different axis entirely — scope, ambiguity, and influence, not artifact delivery.

## References & further reading

**Primary sources**
- Julia Evans, "Get your work recognized: write a brag document" — https://jvns.ca/blog/brag-documents/
- Lalit Maganti, "How I Find Problems to Solve as a Staff Engineer" — https://lalitm.com/post/find-problems-staff-engineer/

**Real-world engineering blogs**
- Sean Goedecke, "How I got promoted to staff engineer twice" — https://www.seangoedecke.com/staff-engineer-promotions/
- Formation.dev, "The storytelling skills senior engineers need for stronger interviews" — https://formation.dev/blog/the-storytelling-skills-senior-engineers-need-for-stronger-interviews/

**Deeper dives**
- Exponent behavioral interview guide (story structure, follow-up handling) — https://www.tryexponent.com/blog/behavioral-interview-questions

[🎓 12 — Capstone & interview preparation](../README.md)
