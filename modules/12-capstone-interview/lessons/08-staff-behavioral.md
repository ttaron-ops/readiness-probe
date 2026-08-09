---
lesson: 08
title: "Staff-signal behavioral / STAR"
module: 12
concept: "senior-vs-staff scoring"
status: not-started
est_time: "3 hrs"
artifacts: ["a bank of 8-10 staff-tuned STAR stories mapped to the five staff competencies, each landing a metric + a tradeoff/reversibility note"]
---

# Staff-signal behavioral / STAR

## Why this matters

You already beat the generic behavioral round in your sleep — STAR structure, "tell me about a conflict," calm delivery. None of that is the problem. The problem is that a staff behavioral loop is scoring a *different axis* than a senior one, and strong senior candidates fail staff loops by giving technically excellent senior answers. The interviewer isn't asking "can this person do the work" — your artifacts already answer that. They're asking "does this person operate at staff scope," and that's a question about scope, ambiguity, and influence, not about how hard the technical problem was.

The concrete risk for you specifically: your instinct under pressure is to reach for the sharpest *technical* moment — the hardest debug, the fastest fix, the cleverest system. Those are senior stories. Told as-is, they actively signal "senior" to a staff panel, because heroics and speed and intuition are the senior register. Your capstone gives you the raw material for the staff register — driving a contested attribution model through three teams, deciding GPU cost was the problem worth solving before anyone asked, correcting the leading OSS tool's design — but only if you *frame* those events for the axis being scored. This lesson builds a bank of 8-10 stories already framed for that axis, so you walk in with staff-shaped material instead of improvising a senior answer to a staff question.

## What's new here

- **Skip** (you own it): STAR mechanics, conflict/failure/feedback question types, how to structure and pace an answer, sounding composed. All sharp.
- **New**: the exact **senior-vs-staff distinction being scored** — and the *framing tells* an interviewer uses to place you on the wrong side of it.
- **New**: the **five staff competencies** and which of your real artifacts arms each — so every story maps to a competency, not just an event.
- **New**: a **staff-tuned STAR template** (scope in the Situation, ambiguity in the Task, the *tradeoff you made* in the Action, metric + reversibility in the Result).
- **New**: a **story-bank table** — 8-10 prompts mapped to competency + arming artifact — so you carry adaptable material, not memorized scripts.

## Core notes

### The crux: what senior-vs-staff actually scores

The gap between a senior and a staff loop is **scope, ambiguity, and direction — not raw technical skill.** This is the single most important thing in the lesson. A senior loop scores your coding, your local design, your execution, your ownership of a team's surface. A staff loop scores architecture judgment *across* domains, leadership *without* authority, and your ability to move a contested initiative through disagreement to a decision. The technical bar barely moves; the scope of impact and the ambiguity you operate in moves a lot.

Interviewers place you on three explicit dimensions:

1. **Scope of impact** — team vs multi-team vs org. Did your work change your team's outcome, or did it change how three teams / the whole platform operate?
2. **Ambiguity tolerance** — did you wait for direction and execute a defined problem, or did you *define the problem* when it arrived vague?
3. **Influence without authority** — could you move people who don't report to you and weren't obligated to agree?

And there's a **framing tell** they listen for — the same event narrated two ways lands on two different levels:

| Senior register (what to avoid leading with) | Staff register (what to lead with) |
|---|---|
| Heroics — "it was broken, I fought through it" | Tradeoffs made — "I chose X over Y because…" |
| Speed — "I fixed it fast, over a weekend" | Reversibility — "one-way vs two-way door, I chose to ship because it was cheap to reverse" |
| Intuition — "I just knew it was the driver" | Assumed constraints — "given the constraint was truthful throughput, not occupancy…" |
| Individual save — "I unblocked the team" | Defensibility under hindsight — "knowing what I knew then, it was the right call, and here's what would've flipped it" |

Heroics / speed / intuition = **senior**. Tradeoffs / reversibility / assumed-constraints / defensibility-under-hindsight = **staff**. You can tell the *same GPU incident* either way — the event doesn't change, the register does.

The **most common failure**, and the one to actively guard against: **overclaiming broad impact without concrete cross-team evidence.** Saying "I aligned the org" with nothing but a solo project behind it reads worse than an honest team-scoped story, because the panel probes for the cross-team specifics and finds air. Your defense is that your claims are *real* — you genuinely moved finance + ML + platform to an agreed model (06/11) — so lean on the ones you can evidence and don't inflate the ones you can't.

### The five staff competencies → your STAR material

You already have arming artifacts for all five. The job is mapping, not manufacturing.

**1. Technical leadership / direction — the capstone thesis.**
You identified that GPU cost was unattributable and *drove the fix* before anyone scoped it for you. Frame on **which problem was worth solving**: not "I was asked to reduce GPU cost" but "I recognized that GPU spend was a top infra line with zero attribution, decided that was the leverage point, and set the direction." That's technical *direction* (defining the problem), the staff version of technical skill (solving a given one). Arming artifacts: the whole arc, anchored by 02 (operator) and 05 (the measurement that proved the problem was real).

**2. Cross-team influence without authority — showback/chargeback (06 / 11).**
You got finance, the ML teams, and platform to agree on an attribution model none of them owned and all of them could have blocked. The staff move inside it: **showback-first**. You published the numbers with no enforcement teeth *deliberately*, to build trust and legitimacy before proposing chargeback — trading speed for a decision that would actually hold, because a chargeback nobody agreed on gets litigated forever. That's influence-without-authority *and* a named tradeoff. This is your strongest staff story; most senior candidates simply don't have one.

**3. Incident command — survive-a-failure (08) + failure-mode log (04).**
Not just "I debugged a sick GPU host" (senior — individual save) but **coordinating the response**: workstreams, stakeholder comms during the incident, and a structured RCA afterward, plus turning the failure-mode log into the artifact that lets *anyone* run the next incident. The CoreWeave SRE JD asks for exactly this — formal RCA, on-call, zero-touch ops. Frame the log as the thing that moved incident response from "me" to "the team."

**4. Driving through ambiguity — dashboard-is-lying (05).**
The purest ambiguity-tolerance story you have. The problem arrived as a *vibe* — "the fleet feels busy but slow," no metric, no ticket. You **defined the real problem** (utilization is the wrong signal; goodput/MFU is the right one), instrumented it, and delivered. Vague-in, defined-metric-out, shipped. That's exactly dimension 2 (define-the-problem vs wait-for-direction).

**5. Mentoring / raising the bar — artifacts as reusable standards.**
Your exporter/operator got adopted and the failure-mode log became a team runbook. The staff frame isn't "I mentored a junior" (senior-team scope) but "I built artifacts that became the *standard* other people now operate by" — raising the bar structurally, not one 1:1 at a time. Influence that outlives your presence in the room.

### The staff-tuned STAR template

Standard STAR, but each slot carries the staff axis:

- **Situation — put the *scope* in it.** Not "our team's fleet" but "a fleet shared across three teams with a top-line GPU spend and no attribution." Establish multi-team / org scope in the first sentence, because scope is dimension 1 and it's decided early.
- **Task — put the *ambiguity* in it.** Name how undefined it was and that *you* defined it. "There was no metric for 'is the fleet actually productive' and no one had asked for one — the task I set was to define that metric." Ambiguity you resolved is dimension 2.
- **Action — put the *tradeoff you made* in it.** The heart. Not the steps you executed but the *call at the fork*: what you chose, what you gave up, the constraint you assumed. "I went showback-first, trading immediate enforcement for legitimacy, on the assumption that an agreed model would outperform a mandated one." Tradeoffs are the staff register.
- **Result — metric + what you'd reverse.** Land the number, then add a reversibility note: the two-way-door call you'd revisit and what would flip it. "Attribution went from zero to per-pod dollars and drove the reclaim case; the one thing I'd revisit is hard-coding the device profile — cheap to reverse, and I'd make it pluggable before we mixed accelerators." Defensibility-under-hindsight, volunteered, is a strong staff close.

### Three fully-worked stories

**A — Cross-team influence (competency 2, artifacts 06/11).**
- *S:* "We ran a 64-GPU fleet shared by three teams — research, product-inference, and an exploratory group. GPU was a top-three infra line, and every team believed the *other* two were the wasteful ones. There was no agreed way to settle it."
- *T:* "Nobody owned 'the attribution model,' and finance, ML, and platform all had a veto. The task I set myself was to get a model all three would actually stand behind — because a model imposed on any of them would just get argued to death."
- *A:* "I made a deliberate call to go **showback-first**: publish per-team allocated-vs-utilised numbers with *no* billing teeth for a couple of cycles. I traded immediate enforcement — which I could have pushed for — for legitimacy, on the assumption that once the numbers were visible and undisputed, agreement on chargeback would follow cheaply. I ran the numbers past each team's lead before publishing so no one was surprised in a room."
- *R:* "All three signed off on the attribution model, and the showback exposed a large waste gap on the exploratory team that they then chose to fix by moving to preemptible. If I were doing it again, the one thing I'd reverse is waiting two full cycles before proposing chargeback — one would've been enough once the numbers held; I over-indexed on trust-building. But showback-first itself was right: the model stuck because they owned it."

**B — Driving through ambiguity (competency 4, artifact 05).**
- *S:* "Leadership was being shown 90-95% GPU utilization across the fleet, and simultaneously our cost-per-token and training throughput were well short of what that implied. The complaint reaching me was a vibe: 'the fleet feels busy but slow.'"
- *T:* "There was no metric that captured 'is the fleet doing useful work,' and no one had filed that as a problem. The task I defined was to figure out what 'busy' actually meant and measure the real thing."
- *A:* "I established that every dashboard read SM-active — a kernel is resident — which is silent on useful FLOPs. I decided the honest metric was goodput / MFU and instrumented it. The tradeoff was real: MFU is harder to compute and it made the fleet look *worse* to the exact people quoting 95%. I chose the true, ugly, actionable number over the flattering one, and de-risked the politics by pairing it with the reclaim opportunity it exposed."
- *R:* "The fleet was at 31% MFU against 95% allocation. That two-thirds gap flipped the conversation from 'we're full, buy more' to 'reclaim first,' and goodput became the number leadership steers on. Reversibility-wise, I hard-coded the device FLOP ceiling — a two-way door I took for speed; I'd make it pluggable before we ran mixed accelerators, since that assumption silently breaks the metric."

**C — Technical leadership / direction (competency 1, artifacts 02/05).**
- *S:* "GPU spend was our second-largest infra line and completely unattributable — you could not tie a dollar to a team or a pod. Nobody had scoped fixing that; it was just accepted."
- *T:* "The ambiguous part was that this wasn't assigned — I had to decide it was *the* leverage point worth spending months on, over other things I could have built. I made attribution the direction."
- *A:* "I built a Kubernetes controller that reconciles per-pod GPU cost as a live, first-class fact. The tradeoff was operational surface: a controller is another thing on-call owns and it can be wrong or lag, versus a cheaper offline batch reconciliation. I took the ownership cost because attribution people don't trust doesn't change behavior — it had to be live and queryable to be believed."
- *R:* "Attribution went from 'nobody knows' to per-pod dollars, which is what made the cross-team showback in the next story even possible. What I'd revisit: I built the cost model assuming stable on-demand pricing; with committed-use discounts in play I'd have made the rate model pluggable from day one rather than retrofitting it."

### The story bank (build 8-10)

| # | Prompt you'll hear | Competency | Arming artifact(s) | Register to hit |
|---|---|---|---|---|
| 1 | "A project you're proud of" | Technical direction | 02 + 05 | which problem was worth solving |
| 2 | "Time you influenced without authority" | Cross-team influence | 06 / 11 | showback-first tradeoff |
| 3 | "Drove something through disagreement" | Cross-team influence | 06 | legitimacy over speed |
| 4 | "Handled a major incident" | Incident command | 08 + 04 | coordinated response + RCA, not solo save |
| 5 | "Worked with a vague/undefined problem" | Ambiguity | 05 | vague-in, defined-metric-out |
| 6 | "A hard technical tradeoff you made" | Technical direction | 02 or 11 | named fork + assumed constraint |
| 7 | "Found a problem no one else saw" | Technical direction | 11 (OSS-tool gap) | read the source, corrected the tool |
| 8 | "Raised the bar / left something behind" | Mentoring/standards | exporter+operator adopted, 04 as runbook | structural standard, not 1:1 |
| 9 | "Something you'd do differently / a failure" | (cross-cutting) | 05 or 02 device-profile call | reversibility, not confession |
| 10 | "Disagreed with a decision / pushed back" | Cross-team influence | 05 (challenged the 95% narrative) | tradeoff + defensibility |

### Prep guidance

Build **6-12 adaptable stories, not scripts** — panels ask the same competencies in different words, so you want material you can re-aim, not a memorized monologue that derails when the question is phrased sideways. The floor is one story per competency (five) plus a failure/reversibility story and a disagreement story; 8-10 gives you spares so you're not reusing the same artifact three rounds running. Non-negotiable per story: it **lands a metric and a tradeoff/reversibility note.** A staff story with no number is unfalsifiable; a staff story with no tradeoff is a senior story wearing a staff title.

## Worked example

Watch one event flip register. The 05 incident — proving the fleet was at 31% MFU — told **two ways**:

- **Senior telling (fails a staff panel):** "The dashboards said 95% but I had a hunch something was off. I dug into it over a weekend, figured out it was reading SM-active, and I built a new dashboard that showed the real number. It was a lot of work but I nailed it and now everyone uses my dashboard." → Tells: *intuition* ("a hunch"), *speed* ("over a weekend"), *individual save* ("I nailed it," "my dashboard"). Technically fine, scores senior.
- **Staff telling (same event):** "The problem arrived as a vibe — busy but slow, no metric. I defined the real question: is the fleet doing useful FLOPs? I decided goodput/MFU was the honest signal and instrumented it, knowing the tradeoff — it's harder to compute and it made the fleet look worse to the people quoting 95%, so I paired it with the reclaim opportunity to land it as money-on-the-table, not a callout. Result was 31% MFU vs 95% allocation, which flipped a buy-more conversation into a reclaim-first one. The device-ceiling hard-code is the two-way door I'd revisit before we run mixed accelerators." → Tells: *defined the problem* (ambiguity), *tradeoff + assumed constraint*, *org-scope result*, *reversibility*. Same weekend of work, scores staff.

Nothing about the underlying work changed. Only scope-in-the-situation, tradeoff-in-the-action, and reversibility-in-the-result were surfaced. That is the entire delta this lesson trains.

## Practice

Feeds the capstone deliverable at [GPU platform capstone](../practice/gpu-platform-capstone/README.md):

1. **Write the full story bank** — all 10 rows above as complete staff-STAR stories using the template (scope-in-S, ambiguity-in-T, tradeoff-in-A, metric+reversibility-in-R). Commit them to the capstone README.
2. **Tag every story** with its metric and its tradeoff/reversibility note in one line each. If either line is empty, the story isn't done — rework the Action or Result until both exist.
3. **Do the register-flip drill:** take stories 4 (incident) and 1 (proud-of), write the *senior* telling of each, underline the heroics/speed/intuition/solo tells, then rewrite in the staff register. This trains the reflex to catch yourself mid-answer.
4. **Guard the overclaim:** for each cross-team story (2, 3, 10), write the two or three concrete cross-team specifics you'd give if the panel probes "who exactly did you align, and how do you know they agreed?" If you can't produce specifics, downgrade the claimed scope to what you *can* evidence.
5. Rehearse each story aloud to ~2 minutes; you want them adaptable, not memorized — vary the opening question wording and confirm the same story still fits.

## Self-check

- What three dimensions does a staff behavioral loop score that a senior loop doesn't emphasize, and what's the one-line framing tell that places you on the senior side? **Answer:** Scope of impact (team vs multi-team vs org), ambiguity tolerance (wait-for-direction vs define-the-problem), and influence without authority. The tell: **heroics / speed / intuition / individual-save = senior**, while **tradeoffs-made / reversibility / assumed-constraints / defensibility-under-hindsight = staff** — the same event told in the first register scores senior regardless of how hard the work was.
- Which of your artifacts arms the influence-without-authority competency, and what specific staff *move* inside it do you emphasize? **Answer:** The showback/chargeback work (06/11), where you moved finance, ML, and platform to an agreed attribution model none of them owned. The move to emphasize is **showback-first** — publishing the numbers with no enforcement teeth deliberately, trading immediate enforcement for legitimacy so the model would actually hold, rather than mandating a chargeback that gets litigated forever. The guard: you can evidence the cross-team specifics, so this is a claim you lean on rather than inflate.
- Why is "overclaiming broad impact" the most common staff-loop failure, and how do you defend against it with your material? **Answer:** Panels probe an "I aligned the org" claim for concrete cross-team specifics, and if there's only a solo project behind it they find air — which reads worse than an honest team-scoped story. You defend by leaning on the claims you can actually evidence (the real three-team attribution agreement in 06/11) and downgrading any scope you can't back with specifics, keeping every scope claim falsifiable.

## Resources

- Stackademic, "Staff Engineer interviews aren't senior + 1" — the senior-vs-staff axis, directly — https://blog.stackademic.com/
- Underdog.io, "Staff Engineer vs Senior Engineer" — scope/ambiguity/influence distinctions — https://underdog.io/blog/staff-engineer-vs-senior-engineer
- Dilip Kumar, "Behavioral interview round for Staff Engineer" — https://dilipkumar.medium.com/behavioral-interview-round-for-staff-engineer-f750eef6c438
- iuriio.com, "STAR for Staff+ / EM interviews" — the tuned STAR template — https://iuriio.com/
- Your own arming artifacts: 02, 05 (technical direction), 06/11 (influence), 08/04 (incident command), the adopted exporter/operator + failure-mode runbook (standards) — the bank you build here becomes the behavioral layer of the capstone README.

[🎓 12 — Capstone & interview preparation](../README.md)
