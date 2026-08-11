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
Lessons 05-08 each drilled one slice of the loop in total isolation, on its own clock, with nothing else competing for your attention: 05 built six system-design skeletons and rehearsed volunteering the GPU-specific layer; 06 built three memorized debugging decision trees and the reflex to distrust `nvidia-smi` util; 07 turned your eleven capstone artifacts into 3-minute decision→tradeoff→metric narrations; 08 rebuilt your behavioral stories from senior-register heroics into staff-register scope, ambiguity, and influence. Each of those lessons proved you *can* do one thing well, rested, with time to think. None of them proved you can do all four, back-to-back, on a stranger's clock, with no notes and no do-overs — which is the actual shape of the thing you're being hired against. This lesson closes that gap. It is not a new skill; it is the integration test that either confirms the previous four lessons compose under pressure or tells you precisely where they don't — and it is the final gate of the entire course, the last thing standing between you and opening a real campaign.

## Why this matters
Every prior lesson in this module drilled one slice in isolation: system-design reps (05), debugging reps (06), artifact narration (07), STAR (08). A real loop does not hand you one slice at a time — it runs four different rounds back-to-back in a single afternoon, each with a different interviewer, and the failure mode is *interference*: you nail the design round, then arrive at behavioral drained and deliver a metric-free story, and the debrief averages you down to "strong but inconsistent — no offer." Practicing slices proves you *can* do each thing. It does not prove you can do all four cold, in sequence, on a clock, unaided. That is a separate skill and it is the one that actually gets tested.

This is the final gate of the whole course. Not a new topic — a pass/fail integration test. You do not get to grade yourself on effort or on how much you've read; you grade yourself on whether, in a timed simulation with no notes, you clear five hard-edged dimensions. The stake is calibration: candidates who *feel* ready and candidates who *are* ready diverge exactly here, and the cost of being wrong is a burned loop at CoreWeave or Lambda — companies you get one clean shot at. Better to fail the gate at your desk this week than in the debrief next month.

There's also an operational reason the debugging round in particular carries so much weight, and it's worth internalizing before you run the loop, not after: distributed GPU training is failure-prone at a rate that makes debugging reflexes a survival skill, not a nice-to-have. Meta's FAIR clusters saw mean time to failure drop from **7.9 hours at 1,024 GPUs to 1.8 hours at 16,384 GPUs** — meaning at frontier scale, something is breaking on the order of every two hours (data cited in CoreWeave's engineering blog, sourced from Meta's own reliability research). A platform engineer who can't rapidly bisect compute vs comms vs data isn't a weak interviewer, they're a liability on a live fleet. The mock loop's D2 round exists because the job it's screening for genuinely looks like that, constantly.

## What's new here (calibration)
Nothing here is a new interview skill. Two new things:
- **The full-loop protocol** — the four rounds stitched into one timed session (170 min), self-run or with a peer, so you rehearse the *sequence and the fatigue*, not just the parts.
- **The readiness rubric** — five pass/fail dimensions with an explicit **fail signal** for each, plus a map from every failure back to the module that fixes it. Passing the gate = five clean passes in one unaided run. Anything less is a remediation list, not a pass.

The meta-skill this trains: honest self-scoring. The rubric is written so you cannot fudge it — each dimension has a concrete disqualifier (you quoted a utilization number as truth; a STAR story had no metric; a design answer never named a failure mode). If any fires, that dimension fails, full stop, regardless of how good the rest of the answer sounded.

## Core concepts

### The mock-loop shape
Mirror lesson 01's consolidated neocloud loop, compressed into one sitting. Four rounds, 170 minutes:

| # | Round | Time | What you run |
|---|---|---|---|
| 1 | Recruiter / HM screen | 35 min | 90-sec positioning pitch → resume-driven domain Q&A → your questions for them |
| 2 | GPU/infra system design | 45 min | Pick **one** of P1–P6 (lesson 05); design it out loud, recorded |
| 3 | Deep-dive / debugging | 45 min | Run **D2 — the 100%-util trap** (lesson 06) end-to-end |
| 4 | Behavioral / staff-signal | 45 min | Deliver **3 STAR stories** (lesson 08), interviewer picks the prompts |

Run it as one block with only a 5-min break between rounds — the back-to-back is the point. Record every round (audio minimum; screen for round 2's whiteboard). If you have a peer, they play interviewer and withhold prompts (the "do you volunteer it?" test); solo, you read the prompt cold and start a timer. **No notes, no pausing, no re-takes.** A run where you stopped to look something up is a practice rep, not a gate attempt.

If you want a second, harder-calibrated data point beyond a self-run or peer-run version, real interviewers exist for exactly this: interviewing.io runs mock interviews conducted by engineers who interview at top companies, at real-loop pressure and pacing (see Real-world use cases below). It's not a substitute for the self-administered gate — it's a way to sanity-check that your own scoring isn't drifting soft.

Round 3 note — **D2, the 100%-util trap**, is the mandatory debug round because it is the single most discriminating GPU-infra reflex: the interviewer says "utilization is pegged at 100% but throughput is half of yesterday — go." The whole test is whether you *distrust the util number* and pivot to MFU/goodput instead of celebrating the 100%, then bisect the three suspects — **compute** (throttling, a bad kernel, wrong precision), **comms** (NCCL collective stalls, a link renegotiated down, a straggler rank), and **data** (loader starvation, NUMA-misplaced pinning, storage bottleneck). Talking through hypothesis → command → expected reading → next branch is the signal.

D2 earns its mandatory slot for a second reason beyond frequency: it's the round where the *automation curve* is catching up fastest, and that raises the bar for the human version rather than lowering it. CoreWeave's own Mission Control platform productizes GPU straggler detection and automated remediation — the exact reasoning chain (per-rank step-time outliers → suspect a degraded node → cordon/reschedule) this lesson drills manually. When a platform vendor ships that reasoning as a product feature, it means the reasoning is well-understood and expected as baseline competence in a hire, not a nice-to-have differentiator. You're not being asked to out-think automation; you're being asked to demonstrate you understand the mechanism well enough that you'd know when the automation is wrong.

### The scoring rubric — the final course gate
You pass the gate only if, **unaided, in one timed mock, you clear all five**:

1. **System design.** Take any P1–P6 and cover *all* its probe axes (lesson 05's checklist) **while volunteering scale / cost / failure / SLO tradeoffs in the first ~3 minutes** — before being asked. No hand-waving the GPU-specific layer (isolation model, KV cache, gang scheduling, MFU — whichever the prompt turns on).
   - **Fail signal:** the answer would score as "strong web/distributed engineer" — you sharded, cached, queued, load-balanced, but never named a *GPU-specific* constraint; OR you waited to be prompted for failure modes / cost / scale instead of volunteering them.
2. **Debugging.** Run D2 end-to-end: correctly distrust the 100% util number, pivot to MFU/goodput, and bisect compute vs comms vs data with a named command and expected reading at each branch.
   - **Fail signal:** you quoted the utilization number as truth (treated 100% as "the GPU is working hard, so that's fine") — the single most common disqualifier — or you never reached MFU/goodput, or you jumped to a fix without bisecting.
3. **Behavioral.** Deliver **3 staff-level STAR stories**, each landing a **metric** and a **tradeoff / reversibility note**, each reading as **multi-team scope** (you influenced beyond your own team — set a standard, unblocked another org, made a call others adopted).
   - **Fail signal:** a story with no number; a story that's all Situation/Task and thin on *your* Action; or scope that reads as individual-contributor ("I fixed a bug") rather than staff ("I changed how three teams attribute cost").
4. **Artifact narration.** Walk the full capstone value chain in **5 minutes**, and connect **any single artifact** the interviewer points at to a **decision, a tradeoff, and a number**.
   - **Fail signal:** you can recite what an artifact *is* but not what *decision it drove*, or you reach for a tradeoff/number and have none — the artifact reads as a demo, not as operated judgment.
5. **Portfolio.** The repo + design doc + flagship blog are **published and self-explanatory to a hiring manager who never meets you** — a stranger lands on the README and understands the value chain without you in the room.
   - **Fail signal:** the story only works when *you* narrate it; the published artifacts have dead links, no README framing, or assume context the reader doesn't have.

Note what the rubric deliberately does *not* score: how smart you sounded, how much you knew that never came out of your mouth, or how good any one round was in isolation. It scores five specific, checkable behaviors. That's a feature, not a limitation — a real loop's debrief works the same way, aggregating discrete signals from people who compare notes afterward, not a holistic vibe from a single conversation.

### How to score & remediate
Score each dimension **pass / fail** — no partial credit, because interviews don't give partial credit. For every fail, write the *specific* fail signal that fired (quote yourself), then map it to the module that fixes it and re-drill *that* module before your next gate attempt:

| Failed dimension | Typical fail signal | Remediate in |
|---|---|---|
| System design | never named a GPU-specific constraint; waited to be prompted | **M12 L05** (P1–P6 checklists + volunteering reflex); arming module for the missed axis |
| Debugging | quoted util as truth; never reached MFU/goodput | **M12 L06** (D2) + **M05** GPU observability + **M08** distributed training |
| Behavioral | STAR with no metric; IC-scope not staff-scope | **M12 L08** (STAR + staff signal) |
| Artifact narration | artifact = demo, no decision/tradeoff/number | **M12 L07** (narrate artifacts) + **M12 L02** (capstone synthesis) |
| Portfolio | not self-explanatory without you | **M12 L03** (portfolio/writeup) + **M12 L04** (flagship blog + demo) |

The remediation list *is* the output of a failed run. Re-drill the mapped module, then re-run the **whole** loop — not just the failed round, because the point of the gate is the integrated, fatigued sequence.

### Readiness signal vs the job market
Tie back to lesson 01's loops. A clean gate pass is the trigger to open your campaign, and the sequencing is not arbitrary: **attempt the neoclouds first** — CoreWeave, Lambda, Nebius, Modal, Together. They are no-DSA (or optimization-flavored, not leetcode), and your 11 artifacts are effectively their interview syllabus, so you convert highest there and the loops double as live rehearsal. Lead with the two whose killer artifact you own most cold: **Lambda** (live Linux round ← survive-a-failure 08 + failure-mode log 04) and **CoreWeave** (K8s/GPU design ← gpu-cost-operator 02 + exporter 01). Gate **NVIDIA and big-tech ML-infra** behind a few weeks of kept-warm DSA and schedule them *after* a couple neocloud loops, so your domain story is already sharp when you hit the one DSA-flavored tier. Do not open your campaign on a dimension you just failed — if the gate flagged system design, don't lead with Together's deep design bar.

## Perspectives

**The integration-testing view.** Borrow the metaphor directly from software testing, because it maps almost exactly. Lessons 05-08 are unit tests — each one exercises a single component (the design skeleton, the debug tree, the narration, the STAR bank) in isolation, with clean inputs and no shared state to corrupt the result. A unit test passing tells you the component works *when nothing else is happening*. This lesson is the integration test: it runs all four components together, under shared resource constraints (your attention, your energy, the clock), and it's specifically designed to surface the bugs that only appear under load — the STAR story that degrades when you're mentally still in debug-round mode, the design round that runs long and eats into your behavioral prep time. A codebase with 100% unit coverage and no integration tests still ships bugs; a candidate with four individually-strong lessons and no full-loop rehearsal still fails loops. The fix in both cases is the same: you don't get to skip the integration test because the units passed.

**The self-assessment-bias view.** There is a well-documented gap between feeling ready and being ready, and it widens specifically under self-study conditions, because the self-assessor and the thing being assessed are the same anxious, motivated person. You *want* to believe the prep worked — you've spent weeks on it — and that want quietly reshapes how you grade your own mock reps: a vague-but-confident-sounding answer gets rounded up to "basically fine," a stumble gets excused as "I knew it, I was just tired." The five-dimension, no-partial-credit rubric exists specifically to correct for this. It replaces a felt sense of "that went well" with a checklist of concrete, quotable disqualifiers — did you or did you not name a GPU-specific constraint; did the story have a number or not. You cannot generously round a missing number up to "I basically said it." That rigidity is uncomfortable and it is the entire point: the discomfort of failing a gate you built yourself is far cheaper than the discomfort of failing a loop you can't retry.

**The hiring panel's view.** It's worth sitting, for a moment, on the other side of the table. A real debrief doesn't average your best round against your worst — it treats inconsistency itself as the signal. A panel that hears "strong system design, distrusted the util number correctly, but the behavioral stories were vague on impact" doesn't conclude "3.5 out of 4, close enough." They conclude "we don't have a consistent read on this person's actual level," and an inconsistent read is functionally identical to a negative one at senior/staff levels, because the role requires reliability under exactly this kind of cross-domain pressure — a platform engineer who's sharp on design but goes soft explaining impact to stakeholders is a real operational risk, not just an interview quirk. This is why the mock loop scores pass/fail per dimension rather than an aggregate score: it's rehearsing the actual shape of the judgment that will be applied to you, not a friendlier approximation of it.

**The productization view.** Watching what companies choose to automate is a good proxy for what they consider baseline-expected human reasoning versus genuine differentiation. CoreWeave shipping Mission Control — a platform that automates GPU straggler detection and remediation — is a signal about the D2 debugging round specifically: the compute/comms/data bisection this lesson drills by hand is well-understood enough, industry-wide, that it's being built into tooling. That cuts two ways for a candidate. It does not make the manual skill obsolete for an interview — loops still test the underlying reasoning because tooling breaks, tooling is wrong sometimes, and someone still has to understand the mechanism to trust or override the automation's verdict. But it does mean fumbling this reasoning in a loop reads worse than it might have five years ago: you're not being asked to invent a novel diagnostic, you're being asked to demonstrate fluency in something the industry has already decided is table-stakes enough to productize.

## Real-world use cases
- **CoreWeave, "Why Distributed Training Fails at Scale"** — https://www.coreweave.com/blog/why-distributed-training-fails-at-scale — cites Meta FAIR's real MTTF-at-scale numbers (7.9 hours at 1,024 GPUs, dropping to 1.8 hours at 16,384 GPUs). What it shows: the operational reality behind why the D2 debugging round is weighted so heavily — failures at this frequency make rapid bisection a core job function, not an interview party trick.
- **CoreWeave Mission Control** — https://www.coreweave.com/mission-control — CoreWeave's platform for automated GPU straggler detection and fleet remediation. What it shows: the manual reasoning drilled in this lesson's D2 round is exactly what the industry is now automating and productizing, which is validation that the underlying mechanism (not just the interview answer) is real and load-bearing.
- **interviewing.io** — https://interviewing.io/mocks and https://interviewing.io/blog/technical-interview-practice-gap — a real platform offering mock interviews with engineers from top companies, drawing on 100K+ recorded mock interviews. What it shows: a concrete way to get a peer-interviewer version of this lesson's mock loop from someone other than yourself, useful as a calibration check against self-scoring drift.
- **NVIDIA SDE-2 interview experience (Medium)** — https://medium.com/@dadhichhardik26/nvidia-sde-2-interview-experience-offer-41d79116d478 — a first-person account of a full NVIDIA interview loop. What it shows: what an actual round sequence feels like end-to-end from a candidate's perspective, a useful companion read alongside this lesson's simulated version.

## Worked example
**A filled-in scorecard — one real run (P2 design, D2 debug, 3 STAR). Mixed result.**

- **Round 1 — recruiter/HM screen.** 90-sec pitch landed ("senior platform eng, 11 artifacts built against real GPU-fleet stacks, home field is cost/attribution and observability"); domain Q&A crisp. **PASS.**
- **Round 2 — system design, P2 (cost attribution/showback).** Opened by volunteering the frame in 80 sec: "~1,000 H100s, ~$2–3/GPU-hr, ~$40–60M/yr, SLO is attribution accuracy + explainability, dominant failure is mis-attribution eroding trust." Covered telemetry (DCGM DaemonSet → Prometheus), the formula `team_cost = (team_util/total_util) × gpu_hour_cost`, and — unprompted — its three flaws (util ≠ value, idle-but-reserved, rewards busy-loops). Named the hybrid reserved-vs-usage billing tradeoff and the showback→chargeback path. Missed one axis: never mentioned **Prometheus cardinality** (tenant×gpu×model) until the interviewer prompted it. Every axis ≥1, four axes defended with tradeoffs, all four volunteering dims hit in the first 3 min. **PASS** (one prompted axis is within tolerance since it wasn't the GPU-specific layer and I volunteered scale/cost/SLO/failure).
- **Round 3 — debug, D2 (100%-util trap).** Distrusted the 100% immediately: "util means a kernel is resident, not that it's doing useful FLOPs — I want MFU/goodput." Pivoted correctly, then bisected: **compute** (`nvidia-smi -q -d POWER,TEMPERATURE,CLOCK` for throttle, checked precision), **comms** (NCCL — suspected a straggler rank via per-rank step-time outliers, checked NVLink link width for a down-renegotiated link), **data** (loader starvation + NUMA pinning via `nvidia-smi topo -m` / `numactl`). Landed on "throttling or a straggler" with the next command to disambiguate. **PASS.**
- **Round 4 — 3 STAR stories.**
  - Story A (drove cost-attribution standard across 3 teams): metric ✓ (surfaced $X/mo of orphan spend), tradeoff ✓ (showback-first for adoption vs chargeback for enforcement), multi-team scope ✓. **PASS.**
  - Story B (survived a fleet failure): metric ✓ (cut large-job failure rate, cited the 14%→4% lemon-node framing as the model I applied), reversibility note ✓, scope ✓. **PASS.**
  - Story C (dashboard-is-lying observability rebuild): strong narrative but I **never gave a number** — said "throughput improved a lot." **FAIL** (metric-free STAR).
- **Artifact narration.** Walked the value chain in ~5 min; when asked to connect the Kueue showback (06) to a decision, gave decision + tradeoff + a number cleanly. **PASS.**
- **Portfolio.** Repo + design doc published; flagship blog live and README-framed. **PASS.**

**Result: 4/5 dimensions pass; Behavioral FAILS on Story C (no metric). Not a gate pass.**

**Remediation list:**
1. **Behavioral (the blocker):** Story C has no number. Return to **M12 L08** — attach a hard metric to *every* story before it's loop-ready. Fix: rebuild Story C around the actual goodput/MFU delta from the "dashboard is lying" work (05), state it as a number, and add the alerting tradeoff (alert on goodput regression, not util). Re-drill until all three stories land metric + tradeoff + multi-team scope cold.
2. **System design (polish, not a fail):** the cardinality axis was prompted, not volunteered. Add one sentence to the P2 skeleton (L05 card) so tenant×gpu×model cardinality is volunteered in the close.
3. **Re-run the full 170-min loop** — not just round 4 — until all five dimensions pass in one unaided sitting.

## Practice
- Run **one full timed mock loop** end-to-end: 35 + 45 + 45 + 45 min, 5-min breaks, recorded, no notes, no pausing. Pick one of P1–P6 for round 2 and D2 for round 3.
- **Fill the scorecard** — pass/fail on all five dimensions, and for every fail quote the exact fail signal that fired.
- **Produce the remediation list** — map each fail to its module, re-drill that module, then **re-run the whole loop.** Repeat until all five dimensions pass in a single unaided run.
- **Get one external calibration rep**, if feasible — book a mock through a platform like interviewing.io, or trade with a peer who wasn't in the room for your self-run, and compare their scoring of your run against your own self-scored record. A gap between the two is itself a data point: it tells you whether your self-scoring has drifted lenient.
- Bank the scored record + remediation list as the readiness artifact and fold it into the [GPU platform capstone](../practice/gpu-platform-capstone/README.md). A clean five-dimension pass is your green light to open the campaign — neoclouds first (lesson 01 sequencing).

## Common pitfalls
- **Running the mock loop "generously."** Self-administered mocks drift toward leniency over repeated attempts — you pause to look something up "just this once," you retry a round that went badly, you round a shaky answer up because you know what you *meant*. Each act of generosity is individually small and cumulatively fatal: it produces a passed gate that doesn't correspond to a passable real loop. The correction is procedural, not willpower-based: no notes, no pausing, no re-takes, full stop, and if you catch yourself bending a rule mid-run, that run is void — restart the whole loop, don't patch it.
- **Re-running only the failed round instead of the whole 170-minute loop.** This is the single most tempting shortcut, because after a failed Story C you *know* exactly what to fix and it feels wasteful to redo three rounds that already passed. Resist it. The gate isn't testing four independent skills, it's testing whether they survive the sequence and the fatigue together — a behavioral round rehearsed fresh, on its own, proves nothing about whether it holds up as round 4 after 125 minutes of design and debugging. Passing round 4 in isolation after a fix is a good sign; it is not a gate pass.
- **Grading a technically-correct-but-never-volunteered answer as a pass.** "I knew the cardinality issue, I just didn't say it until they asked" feels like partial credit and it isn't. The rubric scores what you volunteered, not what you silently knew — because a real interviewer has no access to your internal knowledge state, only to what left your mouth. If you catch yourself justifying a pass with "well, I did know that," that's the fail signal, not the defense.
- **Opening the real campaign on a dimension you just failed.** A gate pass is a snapshot, not a guarantee, and it doesn't override lesson 01's sequencing logic. If your last mock run flagged system design before you remediated it, don't schedule Together's deep-design-bar loop first out of momentum. Similarly, a clean gate pass doesn't license skipping neoclouds to "test yourself" against NVIDIA or big-tech first — the sequencing (neoclouds first, DSA-flavored tier after a couple of live reps) exists independent of whether you passed the mock, because it's about where you convert highest and where you get the cheapest live rehearsal, not about whether you're theoretically ready.

## Self-check
- What is the single most common disqualifier in the D2 debugging round, and what is the correct move instead? **Answer:** Quoting the utilization number as truth — treating 100% util as "the GPU is working hard, all good." The correct move is to distrust it (util = a kernel is resident, not useful FLOPs), pivot to MFU/goodput, then bisect compute vs comms vs data with a named command and expected reading at each branch.
- Why is the gate one back-to-back 170-min loop instead of four separately-passed rounds? **Answer:** Because a real loop tests integration and fatigue — the failure mode is interference (nailing design, then arriving at behavioral drained and delivering a metric-free story). Passing slices in isolation doesn't prove you can clear all four cold, in sequence, on a clock, unaided — which is the skill actually tested.
- A candidate delivers three vivid, well-structured STAR stories but the debrief still fails the behavioral dimension — what most likely fired, and where does it map? **Answer:** A metric-free story (no number) and/or IC-scope instead of staff-scope — either disqualifies the dimension regardless of polish. It maps back to M12 L08 (STAR + staff signal): attach a hard metric and a tradeoff/reversibility note to every story, and frame it as multi-team scope.
- Why does the debugging round carry so much weight in the loop, beyond just being "hard to fake"? **Answer:** Because distributed GPU training fails constantly at scale — Meta FAIR data cited by CoreWeave shows mean time to failure dropping from 7.9 hours at 1,024 GPUs to 1.8 hours at 16,384 GPUs — so the compute/comms/data bisection reflex isn't an interview party trick, it's a near-daily operational requirement, and the industry is validating that by productizing exactly this reasoning (e.g. CoreWeave Mission Control's automated straggler detection).
- After a failed mock run, why is it wrong to re-run only the failed round rather than the entire 170-minute loop? **Answer:** Because the gate isn't testing the failed skill in isolation, it's testing whether all five dimensions survive the sequence and the fatigue together; a round rehearsed fresh on its own proves it works rested, not that it holds up as, say, round 4 after 125 minutes of prior rounds — so a real gate pass requires re-running the whole loop unaided.

## Connections & what's next
This lesson is the seam that closes the module and the course. It draws directly on 05 (the P1–P6 skeletons that fill round 2), 06 (the D2 decision tree that fills round 3), 07 (the artifact narration scored as its own dimension), and 08 (the staff-tuned STAR bank that fills round 4) — nothing in the mock loop is new material, it is those four lessons placed under the timing and fatigue conditions of a real onsite. It also closes the loop back to 01: the sequencing strategy (neoclouds first, NVIDIA/big-tech gated behind a few kept-warm DSA weeks) only pays off once you've actually cleared the gate, and the gate's remediation table routes every possible failure back to a specific earlier lesson in this module, so a failed run is never a dead end — it's a pointer to exactly what to re-drill.

There is no lesson 10. Per the module [README](../README.md), the next step after a clean five-dimension gate pass is not another lesson — it's opening the real campaign: assembling the [GPU platform capstone](../practice/gpu-platform-capstone/README.md) into its final published form if you haven't already, and starting to apply, leading with the neoclouds where your eleven artifacts convert highest. The [checkpoint](../checkpoint.md) for this module is this same gate, formalized as the whole course's final checkpoint. This lesson is the last rep before that door opens.

## References & further reading

**Primary sources**
- Exponent — ML system design interview guide: https://www.tryexponent.com/blog/machine-learning-system-design-interview-guide
- Exponent — behavioral interview questions guide: https://www.tryexponent.com/blog/behavioral-interview-questions
- Meta, "Revisiting Reliability in Large-Scale Machine Learning Research Clusters" (arXiv:2410.21680): https://arxiv.org/abs/2410.21680 — source of the 14%→4% lemon-node remediation stat referenced in the D2/D3 context and in this lesson's worked example.

**Real-world engineering blogs**
- CoreWeave, "Why Distributed Training Fails at Scale": https://www.coreweave.com/blog/why-distributed-training-fails-at-scale
- CoreWeave Mission Control: https://www.coreweave.com/mission-control
- NVIDIA SDE-2 interview experience (Medium): https://medium.com/@dadhichhardik26/nvidia-sde-2-interview-experience-offer-41d79116d478

**Deeper dives**
- interviewing.io — mock interviews: https://interviewing.io/mocks
- interviewing.io — "The Technical Interview Practice Gap": https://interviewing.io/blog/technical-interview-practice-gap
- ai-engineering-field-guide — interview trends & loop shapes: https://github.com/alexeygrigorev/ai-engineering-field-guide
- Your own capstone — the scored mock-loop record + remediation list are the readiness artifact
- [🎓 12 — Capstone & interview preparation](../README.md)
