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
sources: 8
---

# Narrate your artifacts

## Where this fits

Lesson 06 built your debugging reflexes — the trees, the command-to-signal pairings, and the narration discipline you reach for live when a GPU host misbehaves in front of an interviewer. That lesson is about reasoning under pressure with no script. This lesson is the opposite kind of round: the one where you *do* have the material — eleven built artifacts, all your own work — and the only thing between you and a staff-signal answer is how you narrate it. The gap 07 fills is turning a finished build into a spoken decision, so that "walk me through a project you're proud of" converts your depth instead of burying it in a chronological build log.

## Why this matters

You built eleven artifacts that are, almost literally, this market's interview syllabus. That edge is worth nothing in the room if the story comes out as a build log — "so first I wrote an exporter, then I added labels, then I wrote a controller." A hiring manager cannot grade a build log. They can grade a decision. The distance between "I built a thing" and "I identified an expensive, invisible problem, made a defensible call under a real constraint, and moved a number" is exactly the distance between a mid-level narration and a staff-signal one — and it is a *delivery* gap, not a substance gap. The substance is already on disk.

The specific failure mode for someone with your depth is the heroics arc. You have genuinely hard technical stories, so it is tempting to narrate them as "it was broken and hard and I fought through it." That reads as senior at best: execution under adversity. The staff reframe narrates the same event as a *judgement*: which problem was worth solving, what you traded away to solve it, and how you would know if you were wrong. Same artifact, different spine. This lesson installs that spine on your five sharpest artifacts.

It also matters because this is a rehearsed-delivery skill rather than a knowledge skill, which makes it cheap to fix and expensive to skip. Nobody fails this round because the artifact was not good enough. People fail it because they narrated a strong artifact as a weak story — under time pressure, in front of a stranger, with no second take. The fix is mechanical: install the frame, rehearse it out loud, and the strength of the underlying work does the rest.

There is one more reason, and it is the one people underestimate. **The narration is where your numbers get stress-tested.** An interviewer who hears a figure will ask how you computed it, and a figure you cannot derive on demand is worse than no figure at all — it retroactively discredits every other number in your story. So building the narration forces you to know your own arithmetic cold, which is exactly the preparation the deep-dive round needs anyway.

## What's new here (calibration)

- **Skip** (you own it): STAR mechanics, "how to tell a story," pacing, filler-word discipline, sounding calm. You are sharp on generic delivery.
- **New**: the **four-beat frame** — artifact → decision → tradeoff → metric — with the two beats that carry all the staff signal marked, and what each beat defends against.
- **New**: **five lead narrations written out**, in speaking register, on the course's *verified* numbers rather than placeholder figures. These are scripts to edit, not templates to fill.
- **New**: **compression as selection, not truncation** — the same story at three resolutions, with an explicit rule for who chooses the resolution.
- **New**: the **second-question taxonomy** — the three kinds of follow-up, how to tell them apart in the first four words, and why answering one with another reads as not listening.
- **New**: the **follow-up defence bank** — reversibility and assumed-constraint language, written out for the questions you will actually get.
- **New**: the **portfolio-balance check** — a diagnostic for the failure where three different prompts all route to your favourite story.

## Core concepts

### 1. The frame, and the two beats that carry the signal

Four beats, roughly three minutes, in this order.

```
   THE NARRATION FRAME — four beats, and what each one defends against
  ══════════════════════════════════════════════════════════════════════════════

   ┌─ BEAT 1 · ARTIFACT — but open on the PROBLEM, not the build ────────────┐
   │  "Our fleet looked ~99% busy on every dashboard while 16% of the SMs    │
   │   were lit and the tensor pipes ran at 1%."                             │
   │                                                                          │
   │  DEFENDS AGAINST: "why did you spend time on this?"                     │
   │  SIGNAL CARRIED : judgement about WHICH problem was worth solving       │
   │  ~25 seconds. You earned the right to state it crisply by measuring it. │
   └────────────────────────────┬─────────────────────────────────────────────┘
                                ▼
   ┌─ BEAT 2 · DECISION — the call, and the fork you were standing at ───────┐
   │  "So I stopped reporting the presence metric and instrumented SM        │
   │   breadth and tensor activity instead, and built the join that ties     │
   │   them to a pod."                                                       │
   │                                                                          │
   │  DEFENDS AGAINST: "did you just do the obvious thing?"                  │
   │  SIGNAL CARRIED : that there WAS a choice, and you knew it              │
   │  ~35 seconds. Name the fork explicitly or the decision reads as default.│
   └────────────────────────────┬─────────────────────────────────────────────┘
                                ▼
   ┌─ ★ BEAT 3 · TRADEOFF — what you gave up, and the constraint you assumed ┐
   │  "The honest metric is opt-in, needs elevated privileges, and it makes  │
   │   the fleet look far worse to the people quoting 99% — I traded a       │
   │   flattering number for an actionable one, on the assumption that a     │
   │   true number people can act on beats a green dashboard nobody trusts." │
   │                                                                          │
   │  DEFENDS AGAINST: "did you get lucky, or did you weigh something?"      │
   │  ★ SIGNAL CARRIED: THE STAFF BEAT. Skipping it caps you at senior.      │
   │  ~40 seconds. A tradeoff with ONLY an engineering cost under-delivers — │
   │  the ones that land have an organisational or political cost too.       │
   └────────────────────────────┬─────────────────────────────────────────────┘
                                ▼
   ┌─ ★ BEAT 4 · METRIC — the number, and ideally a second-order number ─────┐
   │  "99 on the presence field against 0.16 SM-active on the same GPU in    │
   │   the same second; integrated over a day that was 57% of allocated      │
   │   GPU-hours doing no SM work — which became the case for the batching   │
   │   work that took throughput up ~2.9× on the same eight cards."          │
   │                                                                          │
   │  DEFENDS AGAINST: "is this real, or a study project?"                   │
   │  ★ SIGNAL CARRIED: that the work landed. NON-OPTIONAL.                  │
   │  ~30 seconds. Second-order number = what the first number UNLOCKED.     │
   └──────────────────────────────────────────────────────────────────────────┘

   THE TWO-ITEM RUBRIC, applied to every rehearsal:
     no number   → it is an anecdote
     no tradeoff → it is a heroics arc, and heroics score SENIOR
   Score every rep on exactly those two before anything else.
```

**Why beats 3 and 4 and not the others.** Beats 1 and 2 establish that you did something coherent, which is table stakes at this level — everyone in the pipeline did something coherent. Beat 3 is the only place in the story where you demonstrate that you *weighed* rather than *defaulted*, and beat 4 is the only place you demonstrate that anything changed as a result. A narration missing either is fluent and forgettable.

**One clarification on beat 3 that most people get wrong.** "The honest metric is harder to compute" is an engineering caveat, not a tradeoff. A tradeoff that carries staff signal names a cost you paid in a *different currency* than the benefit — political capital, on-call surface, maintenance divergence, someone else's roadmap. "It makes the fleet look worse to the people who have been quoting 99%" is a tradeoff. "It requires a config change" is a footnote.

**On frames generally.** Other engineer-oriented storytelling frames exist — context, constraints, decisions, outcomes, learnings is a common one, and it is reasonable. This four-beat version is deliberately narrower and more metric-forward: it compresses context and constraints into a single problem-statement beat and makes the number non-optional rather than folded into "outcomes." That narrowness fits an *artifact* walk-through against a fixed clock. For open-ended behavioural prompts, where scope and ambiguity have to be established before any artifact appears, lesson 08's staff-tuned STAR template is the better instrument. Know both and reach for the one that matches the round.

### 2. Opening on the problem, not the build

Staff loops score judgement about *which* problem, not clean execution of a given one. So the first sentence answers "why was this worth your time," not "what did you build." The difference is one sentence and it is the difference between two levels.

| Execution framing — caps at senior | Judgement framing — staff |
|---|---|
| "I built a Kubernetes operator that attributes GPU cost per pod." | "GPU spend was the second-biggest infrastructure line and nobody could tell you which team — let alone which pod — caused a dollar of it. So I made per-pod GPU cost a first-class, queryable fact." |
| "I wrote a dashboard showing SM activity." | "Every dashboard in the industry leads with a number that measures whether a kernel was resident, not whether the silicon did work. I went and measured the gap on our own hardware." |
| "I read the OpenCost source and found some issues." | "Before I built anything, I wanted to know why the incumbent tool couldn't already do this — because if it could, my project was a waste of everyone's time. So I traced a dollar through its source." |

Same artifacts. The right-hand column tells the interviewer you can *find* the problem worth solving, which is the thing they cannot teach and are specifically probing for. Notice also that the third example volunteers a risk to your own project — "if it could, my project was a waste" — which is a stronger signal than any claim about the project, because it demonstrates you were trying to falsify your own idea.

### 3. The five lead narrations

These are written in speaking register. Edit them into your voice and swap in your measurements; the structure and the discipline are what transfer.

#### N1 — "Your GPU dashboard is lying" (artifact 05). Your single best opener.

> **Problem.** *"There was a contradiction I got tired of. Every dashboard said the fleet was 99% utilised, and simultaneously our cost per unit of work was nowhere near what that implied. If we were really 99% busy, where was the work going? So I went to measure what 'busy' actually meant."*
>
> **Decision.** *"What I found is that the field everyone reads — `DCGM_FI_DEV_GPU_UTIL` — is an unmodified passthrough of a driver counter defined as the percentage of a short sample window during which at least one kernel was resident. It's a threshold at one. One kernel and ten thousand kernels give the same answer, and it has no idea how many SMs exist. So I decided the reporting metric was wrong, not its threshold, and I instrumented the hardware-counter family instead — SM breadth for the waste claim, tensor-pipe activity for the efficiency claim — and put them on the same panel as the old number."*
>
> **Tradeoff.** *"That wasn't free. The honest field ships commented out in the exporter's default counter set and needs elevated privileges, so turning it on is a real change to a privileged DaemonSet across the fleet. And more to the point, it made the fleet look dramatically worse to the people who had been quoting 99%. I traded a flattering number everyone had quietly stopped trusting for an ugly number I could act on — and I de-risked the politics by pairing the ugly number with the recovery it exposed, so it landed as 'here is money on the table' rather than 'your metric was a lie.'"*
>
> **Metric.** *"Same GPU, same second: 99 on the presence field, 0.16 SM-active, 0.011 tensor activity. Eighty-six percent of the compute die dark behind a green panel. Integrated over a day that was 57% of allocated GPU-hours doing no SM work. And the second-order number is the one I care about more — after the batching fix, SM-active went 0.16 to 0.55 and throughput up about 2.9× on the same eight cards, while the presence metric read 99 before, during and after. It could not see the problem and it could not see the solution."*

**Why it leads:** it is a judgement-and-measurement story with a shocking number, a political tradeoff baked in, and a punchline that indicts the metric rather than the team. It signals you see through vanity metrics, which is the single most legible domain tell available to you.

#### N2 — the GPU cost operator (artifact 02, with 04)

> **Problem.** *"GPU spend was a top infrastructure line with no per-team, let alone per-pod, attribution. Cloud billing stops at the node — an eight-GPU box is one line item. The scheduler knows allocations but not usage. The telemetry knows usage but not tenants. The number finance needed didn't exist anywhere, and it wasn't going to exist by accident, because it lives at the seam between three systems that each belong to a different team."*
>
> **Decision.** *"I built the join as a controller rather than as a batch reconciliation. Two loops — an ownership loop that lists the kubelet's pod-resources API and resolves device IDs into the key that will appear on the telemetry series, and a utilisation loop that reads the hardware counters — meeting at exactly one place, the physical GPU UUID. The design decision I'd point at is a type: the owner map is `map[string][]Owner`, not `map[string]Owner`, because under time-slicing a single join key legitimately has several holders, and a single-valued map silently keeps whichever one was written last. That one character is the difference between charging one pod for a whole card and charging everyone correctly."*
>
> **Tradeoff.** *"A controller is operational surface area. It can lag, it can be wrong, it's another thing on-call owns, and it needs privileges on every GPU node. A nightly batch job would have been cheaper to build and cheaper to run. I took the ownership cost because the number had to be live and queryable to be usable in a scheduling conversation, not just in a monthly report — and because a batch job cannot see a pod that existed for forty minutes. I'd reverse that if the only consumer were a monthly finance report."*
>
> **Metric.** *"The thing I'm proudest of isn't the attribution, it's that it's falsifiable. Per physical GPU, the holders' shares plus the unallocated remainder sum to exactly 1.000, asserted continuously as an alerting rule rather than checked once. A sum above one means double-counted holders; a sum below means an unresolved device ID or unbooked MIG stranding. And the honesty number: on a fleet with 24 of 64 GPUs time-sliced, 46.8% of every utilisation-based chargeback dollar rested on an estimate — about $64k a month — which I published *before* anyone asked, because a cost system that hides its own uncertainty gets disbelieved the first time it's disputed."*

**Why it leads:** it is the literal "custom controllers and operators" line in this market's postings, and it is an architecture-judgement story rather than a scripting story. The conservation identity is the part interviewers remember.

#### N3 — reading the incumbent's source (artifact 11)

> **Problem.** *"Before writing a line of my own, I wanted to know why the leading open-source cost tool couldn't already do this — because if it could, my project was a waste of everyone's time. That's also the first question any engineering director asks, and 'it's not great with GPUs' loses that conversation. So I traced a dollar through its source at a named commit."*
>
> **Decision.** *"Three questions, applied to any cost engine: what is the numerator, where does the price come from, and where does the interesting telemetry actually land. The numerator turned out to be a Kubernetes resource *request* count — whole devices, integer-valued. The price is one scalar per physical GPU. And the telemetry is genuinely queried — correctly the hardware-counter field, not the presence one — stored on a nested struct, and in the allocation path read by exactly one thing: a display ratio. It never reaches the multiply."*
>
> **Tradeoff.** *"The fair reading is that it's an allocated-ledger engine, and on a fleet of exclusively held whole GPUs it's *correct*. Five of the six gaps I found are one correct design assumption meeting hardware that changed underneath it; the sixth is physics. So my call was to build a correction layer rather than fork or replace — and to upstream the two gaps that genuinely are small, because a fractional numerator needs a decision about the fractional basis that the standard deliberately declines to make. The cost of that choice is a divergence I have to maintain. I took it because the alternative was living with a number that's wrong by an integer factor in either direction."*
>
> **Metric.** *"Same H100, same $2.10 per GPU-hour, one hour, four tenancy configurations: $2.10 exclusive, which is correct; $14.70 under seven-way MIG, because seven tenants each bill a whole card; $0.00 under time-slicing when the device plugin renames the resource key, because that key is absent from both numerator paths; and $8.40 when it doesn't rename, so N tenants each bill a full card. Same tool, same hardware, same workload — and which answer you get is decided by a GPU-operator flag nobody in the cost conversation is usually aware of."*

**Why it leads:** reading someone else's source, finding the design gap, and being *fair* about it is unusually strong signal. It is cross-domain — systems and finance — and it demonstrates you operate at "I evaluate and correct tools" rather than "I use tools." The fairness beat is what stops it reading as criticism.

#### N4 — survive a failure (artifact 08, with the failure-mode log from 04)

> **Problem.** *"Distributed training fails often enough at scale that recovery is a design property, not an incident category. The published measurement is a mean-time-to-failure of 7.9 hours for thousand-GPU jobs across eleven months of two large research clusters. But the thing that made this worth my time wasn't the rate — it was that the failure is silent. A collective blocked on a dead rank doesn't error; it hangs. The job reports Running, and because the wait is implemented as a resident kernel, every idle GPU reports full utilisation."*
>
> **Decision.** *"I built a reproducible failure lab rather than collecting war stories — inject the fault, watch the signal, write down what the fastest correct diagnosis was. Out of it came a decision order I could hand to someone else: establish throughput before touching device metrics; check whether the honest metric is even being exported before hunting a workload cause; then the shape across breadth, tensor and memory activity, which separates starvation from stall from memory-bound; then per-rank step times to find the outlier; then load-level diagnostics on the drained suspect."*
>
> **Tradeoff.** *"Building a lab and a written order costs real time up front against firefighting each incident as it comes. I front-loaded it deliberately, and the reason is a scope reason rather than a technical one: an ad-hoc process makes the fleet debuggable by me, and a written one makes it debuggable by whoever is on call. That's the difference between being valuable and being a bottleneck — and I'd rather be the second kind of valuable."*
>
> **Metric.** *"Time to diagnosis on the injected faults, and — more usefully — the class of failures now caught by a rule instead of a human. The one I'd name is the absence alert: profiling fields are dropped rather than zeroed when they're unavailable, and absence isn't zero in a query, so half the fleet going dark on the honest metric shows up as an unchanged healthy average. That failure was invisible before and is a page now."*

**Why it leads:** it is the live-debugging round's story, it doubles as incident-command material in lesson 08, and the "debuggable by whoever is on call" line is a scope signal disguised as an operational one.

#### N5 — cross-team showback (artifact 06)

> **Problem.** *"Three teams shared a fleet and each was privately convinced the others were the wasteful ones. There was no agreed model to settle it, which meant every capacity conversation was a negotiation rather than a decision. The problem worth solving wasn't technical — the numbers already existed — it was that nobody trusted them enough to act on them."*
>
> **Decision.** *"I drove showback first: publish the numbers, with the platform's own buckets on the same chart as the tenants', and no enforcement at all for a quarter. Getting finance, the ML teams and platform to agree on one model before it had teeth was the actual work. The design decision that made agreement possible was publishing the platform's own waste — unallocated and cordoned capacity — alongside everyone else's, so the dashboard read as a diagnostic rather than an accusation."*
>
> **Tradeoff.** *"Showback-first is slower and has no enforcement — people can ignore a report. Going straight to chargeback would have changed behaviour faster. I traded speed for legitimacy, because a chargeback nobody had agreed on would have been litigated to death on the first disputed invoice, and the programme would have been switched off rather than fixed. I'd reverse it if the tenants were external customers, where the billing relationship already exists and the legitimacy is contractual."*
>
> **Metric.** *"The agreed model itself, plus what it exposed: the biggest *holder* of GPUs was not the biggest *user* — one namespace held 2.7× the GPUs of another and did about two-thirds of the work, 33% of the fleet's allocation for 9% of its output. And the reason nobody had noticed is the whole through-line of my work: on the dashboard everyone actually looked at, that namespace was the busiest in the cluster."*

**Why it leads:** this is your influence-without-authority story, which is the single hardest staff competency to evidence and the one lesson 08 leans on hardest. The reshuffle number is memorable and it closes the loop back to N1.

### 4. Compression is selection, not truncation

You need each story at three resolutions. They are not the same story trailing off — they are three different selections from the same spine.

```
   THE COMPRESSION LADDER — which beats survive at each resolution
  ══════════════════════════════════════════════════════════════════════════════

                       BEAT 1     BEAT 2      BEAT 3      BEAT 4
                       problem    decision    tradeoff    metric
                       ───────    ────────    ────────    ──────
   30 SECONDS            ███         ·           ·          ███
   "quick, what have   full        DROPPED     DROPPED    full
    you worked on?"                                        + STOP
        │
        │  the 30-second version is beats 1 and 4 ONLY, and it ends
        │  on a hook rather than a conclusion. Then you STOP TALKING.
        ▼
   3 MINUTES             ███        ███         ███        ███
   "walk me through    full       one layer   full       full +
    it"                           of mech.               2nd-order
        │
        │  this is the rehearsed spine. One layer of mechanism per
        │  beat, no more. Over 3:30 means you are padding — cut
        │  MECHANISM, never the tradeoff.
        ▼
   DEEP DIVE             ██        █████       █████       ███
   "how did you        compressed  EXPANDED    EXPANDED    the number
    compute that?"     to one      with the    with the    stays the
   "why not X?"        sentence    rejected    reversal    anchor you
                                   options     conditions  return to

   THE RULE: THE INTERVIEWER CHOOSES THE RESOLUTION, NOT YOU.
     Deliver 30 s → breathe → wait.
     "Tell me more"        → go to 3 min
     "How did you compute" → go to deep dive on beat 4
     "Why not X"           → go to deep dive on beats 2 and 3

   Volunteering the deep dive unprompted is the most common way a strong
   candidate reads as unable to calibrate scope — which is itself a
   negative staff signal, independent of how good the content is.
```

**A diagnostic for whether you have actually built three versions.** If you reach for the deep-dive by default under pressure, you have built one version and are compressing on the fly, badly. Write all three out in advance so the short version is a designed artifact rather than the long version running out of time.

The thirty-second version of N1, in full:

> *"Every dashboard said our GPUs were 99% busy. I measured 16% of the SMs actually lit and the tensor pipes at 1% on the same card in the same second — and the batching fix that followed took throughput up about 2.9× while that 99% never moved."*

Thirty seconds, two numbers, a hook at the end, and then silence. The silence is part of the technique: it hands the interviewer the choice of what to pull on, which is both good manners and good tactics, because whatever they pull is the thing they care about.

### 5. The second question tells you how you are being scored

What an interviewer asks *after* your narration tells you more about the scoring than the narration did. There are three kinds and they want different things.

```
   THE SECOND QUESTION — three kinds, and where each one sends you
  ══════════════════════════════════════════════════════════════════════════════

   you finish the 30-second version and stop talking
                        │
        ┌───────────────┼────────────────────┬─────────────────────┐
        ▼               ▼                    ▼                     ▼
   "Tell me more"  "How did you        "Why didn't you        (silence, or
   "What was the    actually compute    just —"                a pivot to
    hardest part"   that number?"      "Why not X?"            another topic)
        │           "Where did that         │                      │
        │            figure come from?"     │                      ▼
        ▼                 │                 ▼                 THEY ARE DONE.
   SCOPE PROBE            ▼            TRADEOFF PROBE          Do not keep
   they want depth   RIGOR PROBE       they want to know       talking. Ask
   and interest      they are          whether you weighed     THEM a question.
        │            checking the      or defaulted
        ▼            number wasn't          │
   → the 3-minute    decorative             ▼
     version, or          │            → beats 2 and 3 expanded:
     the mechanism        ▼              the rejected option, the
     layer of one    → derive it OUT      constraint you optimised
     beat              LOUD, with units.   for, and the REVERSAL
                       "16 GB of weights   CONDITION
                        at 3.35 TB/s is         │
                        4.78 ms, against        ▼
                        16 GFLOP at 989     never "I didn't think
                        TFLOP/s which is    of it" — that is the
                        16 µs"              only losing answer

   ANSWERING ONE WITH ANOTHER READS AS NOT LISTENING.
     rigor probe answered with more scope  → "they're avoiding the number"
     scope probe answered with derivation  → "they can't zoom out"
   Listen to the first four words. They almost always disambiguate.
```

**The rigor probe is the one that punishes borrowed depth**, and it is why building these narrations forces you to know your own arithmetic. If you say "0.011 tensor activity" you must be able to say, unprompted, *why that number is what it is*: 16 GB of weights per token at 3.35 TB/s is 4.78 ms of memory time against 16 GFLOP at 989 TFLOP/s which is 16 µs of compute — a ratio of about 0.0034, so roughly 1% measured once attention and kernel inefficiency are added. A number you can derive is a number nobody can take away from you.

### 6. The follow-up defence bank

Two archetypes cover most follow-ups. Both are gifts, not attacks — the interviewer is explicitly inviting you to demonstrate that you knew the alternative and weighed it, which a three-minute narration alone cannot establish.

**Archetype A — "Why not X?"** Answer with the tradeoff *and the assumed constraint*, never with "I didn't consider it."

| Question | The answer |
|---|---|
| **"Why not just use time-slicing to raise utilisation?"** | "It raises on-paper density without raising work — it packs more contexts onto a card without any of them doing more useful math, so it would have *masked* exactly the thing I was measuring. And it destroys attribution: the counter is device-scoped, so N tenants read one number and per-tenant cost becomes an estimate. Given my constraint — a number a team lead won't dispute — it was the wrong lever. If the constraint had been density for bursty internal notebooks with nobody billing, I'd reach for it." |
| **"Why a controller instead of a nightly batch job?"** | "A batch job is cheaper to build, cheaper to run, and has no on-call surface. Two things pushed me the other way. The number had to be live to be usable in a scheduling conversation rather than only in a monthly report. And a batch job can't see a pod that existed for forty minutes — allocation is an event stream, and sampling it daily loses exactly the short-lived workloads that cause the most churn. If the only consumer were a monthly finance report, the batch job is correct and I'd say so." |
| **"Why not file an upstream issue instead of building a correction layer?"** | "I did both, and I'd separate them. Two of the six gaps are genuinely small — adding a resource key to two numerator paths, and repairing an inverted guard — and those belong upstream, because a fix that only exists in my fork helps nobody. The fractional numerator doesn't, because it needs a decision on the fractional basis — SM slice, memory, or blended — that the standard deliberately declines to prescribe, so it's a design choice a project has to make rather than a bug to fix. Building the layer let me make that choice explicitly and label which basis I used." |
| **"Why not just use MFU as the headline number?"** | "Because MFU needs the model's FLOPs per token, which lives with the tenant, not with the platform. I can't compute it for a workload I don't own, and a headline number I can only produce for some tenants isn't a fleet metric. So the platform-side claim is the occupancy claim — you allocated a card and almost nothing was scheduled on it — and MFU appears as the calibration point. If I owned the training stack too, I'd publish both." |
| **"Why not just alert when utilisation drops below 50%?"** | "Because that alert has a systematic blind spot exactly where the money is. The metric is a presence duty cycle, so it reads near 100 on every expensive waste case — batch-1 decode, spin-waits, a blocked collective, a starved dataloader. The alert would fire on genuinely idle cards, which are the cheap obvious case, and stay silent on a fleet burning money at full rate. It's not inaccurate; it's accurately reporting something uncorrelated with what I need." |

**Archetype B — "What would you do differently?"** Answer with **reversibility**, not confession.

The structure: name a call made under time or information pressure, say what new information would flip it, and state whether it was a one-way or two-way door. That reframes hindsight as judgement under uncertainty rather than as an admission.

> *"I hard-coded the rate card as a set of constants in the first version rather than as a metric. That was a two-way door and I took it for speed. The information that would flip it — and did — is a heterogeneous fleet: the moment there were two GPU models with two rates, a hard-coded scalar was silently wrong for one of them. So it became a series with a `modelName` label, versioned in git, with the basis and date carried alongside. It was cheap to reverse, which is exactly why shipping the assumption was the right call at the time. The one-way door I was careful about was the *label schema*, because renaming a label breaks every dashboard and every stored query downstream — so I spent disproportionate time on that up front."*

That answer does four things at once: admits something real, classifies the decision by reversibility, names the trigger that flipped it, and demonstrates you distinguish decisions that are cheap to undo from ones that are not. The last part is the actual staff content.

**Tone matters here as much as content.** Candidates who read "why not X" as adversarial get defensive or hedge — "well, I guess that could also have worked." Candidates who read it as an invitation state the tradeoff crisply and move on. The words in a good answer and an adequate one are often nearly identical; the difference the panel remembers is whether you sounded like you had already thought about it.

### 7. Portfolio balance

You have five lead narrations, not one. Under stress the temptation is to steer every open-ended question toward the one you rehearsed hardest — usually N1, because it is the most dramatic.

```
   PORTFOLIO BALANCE — which prompt should route to which story
  ══════════════════════════════════════════════════════════════════════════════

   PROMPT                              PRIMARY        BACKUP     NEVER
   ─────────────────────────────────   ───────────    ────────   ─────────
   "a project you're proud of"         N1 the lie     N2 ctrl    —
   "your hardest technical problem"    N2 the join    N4 fail    N1 (it is a
                                                                  measurement,
                                                                  not a hard
                                                                  build)
   "a time you influenced without      N5 showback    N3 source  N1, N2 — both
    authority"                                                    are solo work
   "a time you were wrong"             N2 (the        N1 (the    —
                                        integration    rate-card
                                        bug)           constant)
   "a disagreement with a senior       N5 showback    N3 source  —
    person"
   "something you built that failed"   N4 fail lab    N2 ctrl    —
   "how do you decide what to work     N3 source      N1 the     —
    on"                                 (you tried     lie
                                        to falsify
                                        your own
                                        project)

   THE DIAGNOSTIC: have someone ask you three different prompts back to
   back, cold. If two of the three land on the same artifact with a
   different opening sentence, you do not have a portfolio — you have
   ONE STORY WITH SEVERAL DOORS INTO IT, and a panel that hears it three
   times in one loop notices.

   THE FIX: rehearse all five to comparable fluency, so the QUESTION
   picks the story rather than your comfort.
```

**The "never" column deserves attention.** Using N1 for "your hardest technical problem" is a real mistake, not a stylistic one: the util-lie work was intellectually sharp but not *hard to build*, and an interviewer who has understood it will notice the mismatch and downgrade your calibration. Matching the story to the competency being probed is itself a scored behaviour.

## Perspectives

**The interviewer's view — the second question is the real test.** The narration establishes that you can present. The follow-up establishes whether the depth is yours. This is why the three probe types matter: a rigor probe answered with more scope reads as avoiding the number, and a scope probe answered with a derivation reads as an inability to zoom out. Listening to the first four words of the question, before you start answering, is a cheap habit with a large effect.

**The compression view.** Building three depths in advance is a design exercise, not a memory one: what is the minimum information that still lands the number, and what is the maximum detail that fits three minutes without losing the thread? Treating it as a product problem — rank the mechanism details by how much signal they carry, cut the ones that are only interesting to you — produces noticeably tighter answers than trying to "talk less."

**The adversarial-follow-up view.** Reframe every "why not X" as free staff-signal delivery. The interviewer is not trying to catch you; they are trying to distinguish "made a real tradeoff" from "got lucky and never considered the alternative," and the follow-up is the only instrument that distinguishes them. Walking in expecting these, with genuine tradeoff language ready rather than improvised, is the highest-leverage preparation in this lesson.

**The honesty view.** Every one of these five narrations contains a caveat delivered as a specification: the fleet was simulated, the time-sliced split is an approximation, the rate is a dated snapshot on a named basis. Delivering those flatly rather than apologetically is what converts them from weaknesses into rigour. The tell is your voice: if it drops on "it was simulated," the caveat is reading as a confession. Put it first in the sentence — "on a 200-node simulated fleet with the traces published, I measured…" — and it reads as a specification instead.

**The future-you view.** These scripts decay. Six months from now you will remember the shape and not the numbers, and a figure you cannot derive is worse than no figure. So the durable artifact is not the script — it is the *derivation* behind each number, written down next to it. Rehearse the derivations, not the sentences, and the sentences reconstruct themselves.

## Real-world use cases

- **The verified findings behind N1** — NVIDIA's `dcgm-exporter` default counter set, where the
  presence field ships enabled and the SM-breadth fields ship commented out, and the vendor's own
  eight-panel Grafana dashboard with no SM-breadth panel. **What it shows:** the narration's central
  claim is checkable in thirty seconds by anyone in the room, which is what makes it safe to lead
  with. A story whose key fact a listener can verify while you are still talking is a different
  class of story.

- **The verified findings behind N3** — OpenCost issues #3900 (filed 2026-07-05, closed 2026-07-09;
  `costBasis=usage` and `costBasis=allocation` within ~1% of each other on a workload measured at
  ~11% GPU utilisation, expected ≈$0.275 versus actual ≈$2.50) and #3828 (a HAMi user billed for a
  whole card while allocated a quarter of one). **What it shows:** you can narrate a criticism of a
  widely used tool while citing the project's own tracker, which is a completely different register
  from an opinion — and #3828 is a stranger independently specifying the correction layer you built.

- **The verified anchor behind N4** — Meta's *Revisiting Reliability in Large-Scale Machine Learning
  Research Clusters* (arXiv:2410.21680, HPCA 2025): a measured mean-time-to-failure of 7.9 hours for
  1,024-GPU jobs across eleven months of two large A100 clusters, with larger-scale figures given as
  projections. **What it shows:** how to anchor a story in published data without over-claiming.
  Saying "7.9 hours measured at 1,024 GPUs, and the paper projects shorter at larger scale" is a
  small piece of precision that signals you read the source rather than a summary of it.

- **The PaLM paper (Chowdhery et al., 2022, arXiv:2204.02311)** as the origin of Model FLOPs
  Utilization. **What it shows:** the citation that backs the "why not just use MFU" follow-up, and
  a demonstration that you know which layer owns which number rather than using the term as a
  synonym for "efficiency."

## Worked example

**A full three-minute delivery of N1, annotated, followed by the three follow-ups it attracts.**

> **[Problem, ~25 s]** *"About a year in I got tired of a contradiction. Leadership was being told the GPU fleet ran at 99% utilisation, and at the same time our cost per unit of work was nowhere near what that number implied. If we were genuinely 99% busy, where was the work going? So I went and measured what 'busy' actually meant."*
>
> **[Decision, ~35 s]** *"The field every dashboard leads with is a straight passthrough of a driver counter, and its own header defines it as the percentage of a short sample window during which at least one kernel was resident. It's a threshold at one — one kernel and ten thousand kernels evaluate the same predicate, and it has no notion of how many streaming multiprocessors exist. So the metric was wrong, not its threshold. I instrumented the hardware-counter family instead: SM breadth for 'did anything get scheduled', tensor-pipe activity for 'was it the work we bought this hardware for', and I put the old number on the same panel as the foil."*
>
> **[Tradeoff, ~40 s]** *"This wasn't a free call in either direction. The honest field ships commented out in the exporter's default counter set and needs elevated privileges, so enabling it was a real change to a privileged DaemonSet across every GPU node — that's a security conversation, not a config edit. And it made the fleet look far worse to exactly the people who'd been quoting 99%. I traded a flattering number everyone had quietly stopped trusting for an ugly one we could act on, and I de-risked the politics deliberately: I never published the ugly number alone. It always shipped paired with the recovery it exposed, so it landed as 'here's money on the table' rather than 'your metric was a lie.'"*
>
> **[Metric, ~35 s]** *"Headline: same GPU, same second, 99 on the presence field against 0.16 SM-active and 0.011 tensor activity — 86% of the compute die dark behind a green panel. Integrated across a day, 57% of allocated GPU-hours were doing no SM work. But the number I actually care about is second-order: after the batching change, SM-active went from 0.16 to 0.55 and throughput up about 2.9× on the same eight cards — and the presence metric read 99 before, during and after. It couldn't see the problem and it couldn't see the fix. That's the sentence that turned 'the fleet is full, we need to buy more' into 'the fleet is a sixth used, let's fix batching first.'"*

**Score it:** number present, and a second-order number — yes. Tradeoff present, with both an engineering cost and a political cost, plus the assumed constraint — yes. Heroics arc — no; it is a judgement story throughout. That is a passing staff narration.

**Follow-up A, a rigor probe: "Where did 0.011 come from? That seems very low."**

> *"It's low and it's derivable, which is why I trust it. Take an 8-billion-parameter model in BF16 on an H100 at batch size one. Every weight has to come out of HBM once per token, so that's 8e9 parameters times 2 bytes, 16 gigabytes, at 3.35 terabytes a second — about 4.78 milliseconds. The arithmetic is roughly 2 FLOP per parameter per token, so 16 gigaFLOP, at 989 teraFLOP a second — about 16 microseconds. The ratio is 0.0034. So the tensor pipes have to be idle about 99.7% of the time; that's not a measurement, it's arithmetic. The measured 0.011 is a little higher because attention adds work and real kernels aren't perfectly bandwidth-efficient. I predicted it before I measured it, which is the main reason I was confident the exporter wasn't misconfigured."*

**Follow-up B, a tradeoff probe: "Couldn't you have just fixed the dashboard's threshold?"**

> *"I considered that and it's the wrong fix, because the threshold isn't what's broken — the quantity is. There's no threshold on a residency duty cycle that tells you how many SMs were lit, because the counter doesn't contain that information. The tempting middle option was the graphics-engine-active field, which carries the profiling prefix and looks like the honest family; its own definition is 'a context is bound and a pipe is busy', which is the same presence question reimplemented, and it reads about 1.0 on the exact workload where SM breadth reads 0.16. So I'd have rebuilt the same lie with a longer metric name. What would have changed my mind: if the honest fields hadn't been available on our silicon — they need a profiling path that isn't universal — I'd have had to fall back to power draw and clocks as proxies, and I'd have said so explicitly rather than pretending the proxy was the measurement."*

**Follow-up C, a scope probe: "What was the hardest part?"**

> *"Honestly, not the measurement — the arithmetic was the easy part. The hard part was the second-order number. It's straightforward to show a bad number; it's much harder to show that fixing it moves something, because that requires getting a team to change their serving configuration on the strength of your claim. What made that possible was framing it as their throughput win rather than my metric being right, and having the before-and-after captured across the rollout rather than as two separate snapshots — a continuous trace through the change is much harder to argue with than two screenshots. If I were doing it again I'd instrument the round trip first and the finding second, because the finding without the round trip is a complaint."*

That last answer is the one that separates levels, and it is worth noticing why: it declines the invitation to describe technical difficulty and instead describes an *organisational* difficulty, then names a process change it produced. That is the staff register applied to a story that could easily have been told as a clever measurement.

## Practice

Feeds [GPU platform capstone](../practice/gpu-platform-capstone/README.md).

1. **Record three-minute narrations for the three lead artifacts (N1, N2, N3).** Record audio or
   video — do not just write them. Time each; over 3:30 means padding, and you cut mechanism, never
   the tradeoff.

2. **Self-score each recording on the two-item rubric** before anything else: did it land a concrete
   number, and did it name a tradeoff with an assumed constraint rather than drifting into heroics?
   Re-record anything that misses either.

3. **Write the derivation next to every number.** For each figure in each script, write out how it
   is computed, with units. This is the durable artifact — the scripts decay, the derivations do
   not, and the rigor probe is aimed exactly here.

4. **Write the 30-second and deep-dive versions** of all three. Verify the 30-second version is
   beats 1 and 4 only and ends on a hook. Practise the silence after it.

5. **Draft the follow-up defences** from §6 in your own words: "why not time-slicing" for N1, "why a
   controller not a batch job" for N2, "why not just file an upstream issue" for N3. Each must
   resolve to a tradeoff with an assumed constraint or to a reversibility note. None may resolve to
   "I didn't consider it."

6. **Write one reversibility answer** — a real call you made under pressure, classified as a
   one-way or two-way door, with the information that would flip it and whether it flipped.

7. **Add N4 and N5** once the lead three are clean. N5 is your influence-without-authority story and
   feeds directly into lesson 08, so getting it right now saves rework.

8. **Run the portfolio-balance check.** Have someone ask you "a project you're proud of," "your
   hardest technical problem," and "a time you influenced without authority" back to back, cold. If
   two of three land on the same artifact, deliberately re-route one and re-record.

9. **Run the probe-type drill.** Have someone follow each narration with a randomly chosen probe
   type — scope, rigor, or tradeoff — without telling you which. Grade yourself only on whether you
   identified the type correctly before answering.

**Acceptance:** three recorded three-minute narrations passing the two-item rubric · a written
derivation beside every number · 30-second and deep-dive versions of all three · three written
follow-up defences · one reversibility answer · N4 and N5 drafted · a clean portfolio-balance check
· probe-type identification correct in at least four of five attempts.

## Common pitfalls

1. **Volunteering the deep-dive to a thirty-second question.** **Mechanism:** the round scores scope
   calibration as well as depth, and unrequested detail is evidence you cannot read what is being
   asked. **Symptom:** you are three minutes into an answer to "quick, what have you worked on."
   **Fix:** beats 1 and 4, stop, breathe, wait. If they wanted more they will ask.

2. **A tradeoff with only an engineering cost.** **Mechanism:** everyone can name an engineering
   caveat; the staff signal is a cost paid in a different currency — political capital, on-call
   surface, someone else's roadmap, maintenance divergence. **Symptom:** "it was harder to compute"
   is the whole of beat 3.

3. **Answering "what would you do differently" with a confession.** **Mechanism:** an unresolved
   regret reads as a mistake you have not processed; a reversibility answer reads as judgement under
   uncertainty. **Symptom:** "honestly I'd have structured it better." **Fix:** name the door, the
   flipping information, and why shipping the assumption was still correct at the time.

4. **A number you cannot derive.** **Mechanism:** the rigor probe exists precisely to find borrowed
   depth, and a figure that collapses under "where did that come from?" retroactively discredits
   every other figure in the story. **Symptom:** you hesitate, then reach for "roughly."

5. **Over-rehearsing one artifact.** **Mechanism:** every open-ended prompt gets redirected to your
   comfort story, and a panel that hears the same story three times in one loop notices. **Symptom:**
   your "hardest technical problem" answer opens with the same sentence as your "proud of" answer.

6. **Mismatching the story to the competency.** **Mechanism:** telling a measurement story in answer
   to "hardest problem" or a solo story in answer to "influenced without authority" reads as poor
   calibration even when the story is strong. **Symptom:** the interviewer re-asks the question with
   emphasis.

7. **Delivering the caveat apologetically.** **Mechanism:** a caveat's job is to make the preceding
   claims credible; an apologetic delivery inverts that and invites probing for more weakness.
   **Symptom:** your voice drops on "it was simulated." **Fix:** put the caveat first in the
   sentence, as a specification.

8. **Confusing the metric families in your own story.** **Mechanism:** the presence field and the
   SM-breadth field measure different things on different collection paths; describing SM-active as
   "just means a kernel was resident" is the exact error your whole narration exists to correct.
   **Symptom:** someone who knows the field IDs looks confused, and stops trusting the rest.

## Self-check

- **What are the four beats, and which two carry the staff signal?** *Answer:* artifact — opened on
  the problem, not the build; decision — the call and the fork you were standing at; tradeoff — what
  you gave up and the constraint you assumed; metric — the number, ideally with a second-order
  number showing what it unlocked. Beats 3 and 4 carry the signal. Beats 1 and 2 establish that you
  did something coherent, which is table stakes; beat 3 is the only place you demonstrate you
  weighed rather than defaulted, and beat 4 is the only place you demonstrate anything changed. A
  story with no number is an anecdote; a story with no tradeoff is a heroics arc, and heroics score
  senior.

- **Why does opening on the problem read as staff, and how would you re-open "I built an operator
  that attributes GPU cost per pod"?** *Answer:* staff loops score judgement about *which* problem,
  not clean execution of a given one, so leading with why it was worth your time demonstrates the
  thing they cannot teach. Re-open as: "GPU spend was a top infrastructure line and nobody could
  attribute a dollar of it to a team, let alone a pod — and it wasn't going to exist by accident,
  because the number lives at the seam between three systems owned by three different teams. So I
  made per-pod GPU cost a first-class, queryable fact." The build detail — that it is a Kubernetes
  controller — comes second, in service of the problem, rather than as the headline.

- **What makes a tradeoff land as staff signal rather than as an engineering caveat?** *Answer:* the
  cost has to be paid in a different currency than the benefit. "The honest metric is harder to
  compute and needs elevated privileges" is a caveat. "It made the fleet look far worse to exactly
  the people who had been quoting 99%, so I traded a flattering number for an actionable one, and I
  de-risked it by never publishing the ugly number without the recovery it exposed" is a tradeoff:
  it names an organisational cost, the assumption under which the trade was correct, and the
  mitigation. The assumed constraint is the part most people omit and it is what makes the decision
  reconstructible by the listener.

- **Give the three second-question types and what each one wants.** *Answer:* the **scope probe**
  ("tell me more", "what was the hardest part") wants depth and interest — go to the three-minute
  version or expand one beat's mechanism. The **rigor probe** ("how did you compute that", "where
  did that number come from") is checking the metric was not decorative — derive it out loud with
  units. The **tradeoff probe** ("why not X", "why didn't you just—") is checking whether you weighed
  or defaulted — expand beats 2 and 3 with the rejected option, the constraint you optimised for,
  and the reversal condition. Answering one with another reads as not listening: a rigor probe
  answered with more scope looks like avoiding the number, and a scope probe answered with a
  derivation looks like an inability to zoom out. The first four words almost always disambiguate.

- **Someone challenges N1 with "why not just use time-slicing to raise utilisation?" What is the
  staff-shaped answer?** *Answer:* name the tradeoff and the constraint rather than dismissing it.
  Time-slicing raises on-paper density without raising work — it packs more contexts onto a card
  without any of them doing more useful math, so it would have masked the exact thing I was
  measuring. It also destroys attribution, because the counter is device-scoped, so N tenants read
  one number and per-tenant cost degrades to an estimate. Given my constraint — producing a number a
  team lead would not dispute — it was the wrong lever. If the constraint had been density for
  bursty internal notebooks that nobody bills, I would reach for it.

- **Why is answering "what would you do differently" with reversibility stronger than answering with
  a confession?** *Answer:* because a confession describes an outcome and reversibility describes a
  decision process. "I'd have structured it better" is an unresolved regret with nothing
  generalisable in it. "I hard-coded the rate card as constants — a two-way door I took for speed;
  the information that flipped it was a heterogeneous fleet, at which point a scalar was silently
  wrong for one model, so it became a labelled series versioned in git; the one-way door I was
  careful about was the label schema, because renaming a label breaks every stored query
  downstream" names the call, classifies it by cost-to-undo, names the trigger, and shows you treat
  reversible and irreversible decisions differently. That last distinction is the actual content.

- **What is the portfolio-balance failure, how do you detect it, and why does it matter?**
  *Answer:* the failure is having one story with several doors into it rather than a portfolio —
  under stress every open-ended prompt gets steered to the artifact you rehearsed hardest. Detect it
  by having someone ask three different competency prompts back to back, cold: "a project you're
  proud of," "your hardest technical problem," "a time you influenced without authority." If two of
  the three land on the same artifact with a different opening sentence, you have the failure. It
  matters because a panel comparing notes hears the same story from three interviewers, which reads
  as narrow rather than deep — and because some pairings are actively wrong: the util-lie story is a
  measurement, not a hard build, so using it for "hardest technical problem" signals poor
  calibration to anyone who understood it.

## Connections & what's next

This lesson turns the artifacts from modules 01–11 into the spoken layer of the capstone. N1, N2,
N3, N4 and N5 are the five narrations you will actually deliver in a loop, and the follow-up
defences drafted here are the same reversibility and assumed-constraint language lesson 08
formalises into a full STAR template. N5 in particular does double duty — it is a lead artifact
story here and the anchor of the influence-without-authority competency next — so getting it clean
now saves rework. The derivations you write beside each number are also the material the deep-dive
round in lesson 06 draws on, and lesson 09 runs all of it under full-loop fatigue rather than in
isolation.

Next: **lesson 08** takes the tradeoff and reversibility instincts you just installed and applies
them to a round scored on a completely different axis — scope, ambiguity and influence rather than
artifact delivery.

## References & further reading

**Primary sources — the facts behind the narrations**

- NVIDIA `dcgm-exporter`, `etc/default-counters.csv` — https://github.com/NVIDIA/dcgm-exporter/blob/main/etc/default-counters.csv — read for: the "the honest metric ships commented out" claim in N1's tradeoff beat, verifiable in thirty seconds by anyone in the room.
- NVIDIA DCGM, `dcgmlib/dcgm_fields.h` — https://github.com/NVIDIA/DCGM/blob/master/dcgmlib/dcgm_fields.h — read for: the exact field definitions you must be able to quote when N1 draws a rigor probe, including why the graphics-engine field is not the honest one.
- NVIDIA `go-nvml`, `gen/nvml/nvml.h` — https://github.com/NVIDIA/go-nvml/blob/main/gen/nvml/nvml.h — read for: the `nvmlUtilization_t` struct comment that N1's decision beat paraphrases.
- OpenCost issues #3900 and #3828 — https://github.com/opencost/opencost/issues/3900 · https://github.com/opencost/opencost/issues/3828 — read for: N3's citable corroboration, and the fairness framing that keeps it from reading as criticism.
- Meta — *Revisiting Reliability in Large-Scale Machine Learning Research Clusters* (arXiv:2410.21680, HPCA 2025) — https://arxiv.org/abs/2410.21680 — read for: N4's anchor. The measured figure is 7.9 hours at 1,024 GPUs; larger-scale MTTFs are projections, and saying so is the precision that signals you read the paper.
- Chowdhery et al. — *PaLM* (arXiv:2204.02311) — https://arxiv.org/abs/2204.02311 — read for: the origin of Model FLOPs Utilization, which backs the "why not just use MFU" follow-up and the layer-ownership argument behind it.

**Course-internal sources**

- `platform-eng/modules/05-gpu-observability/lessons/01-lie-and-truth.md` — every number and derivation in N1, including the batch-1 arithmetic the rigor probe asks for.
- `platform-eng/modules/05-gpu-observability/lessons/08-capstone-allocated-vs-utilised.md` — the 57.3% idle share, the reshuffle finding behind N5's metric beat, and the batching before-and-after.
- `platform-eng/modules/04-gpu-on-kubernetes/lessons/10-capstone-per-pod-attribution.md` — N2's two-loop design, the `map[string][]Owner` decision, the conservation identity, and the 46.8% exposure fraction.
- `platform-eng/modules/11-gpu-cost-economics/lessons/09-existing-tooling-limits.md` — N3's source trace and the four-regime error table.

**Not relied upon**

- Third-party career and interview-storytelling guides were consulted for the alternative framing
  noted in §1. They are referenced as an existing genre of advice rather than as evidence for any
  claim, and no number or mechanism in this lesson depends on them. Earlier versions of this lesson
  quoted a "95% utilisation versus 31% MFU" figure and described SM-active as meaning "a kernel was
  resident"; both were wrong — the residency semantics belong to the presence field, not to
  SM-active, and every figure here now traces to the course's own module 05 measurements.
