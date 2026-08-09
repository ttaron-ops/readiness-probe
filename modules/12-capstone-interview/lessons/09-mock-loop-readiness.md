---
lesson: 09
title: "Mock loop & the readiness gate"
module: 12
concept: "full-loop simulation + final gate"
status: not-started
est_time: "4 hrs"
artifacts: ["a scored mock-loop record + a remediation list"]
---

# Mock loop & the readiness gate

## Why this matters
Every prior lesson in this module drilled one slice in isolation: system-design reps (05), debugging reps (06), artifact narration (07), STAR (08). A real loop does not hand you one slice at a time — it runs four different rounds back-to-back in a single afternoon, each with a different interviewer, and the failure mode is *interference*: you nail the design round, then arrive at behavioral drained and deliver a metric-free story, and the debrief averages you down to "strong but inconsistent — no offer." Practicing slices proves you *can* do each thing. It does not prove you can do all four cold, in sequence, on a clock, unaided. That is a separate skill and it is the one that actually gets tested.

This is the final gate of the whole course. Not a new topic — a pass/fail integration test. You do not get to grade yourself on effort or on how much you've read; you grade yourself on whether, in a timed simulation with no notes, you clear five hard-edged dimensions. The stake is calibration: candidates who *feel* ready and candidates who *are* ready diverge exactly here, and the cost of being wrong is a burned loop at CoreWeave or Lambda — companies you get one clean shot at. Better to fail the gate at your desk this week than in the debrief next month.

## What's new here
Nothing here is a new interview skill. Two new things:
- **The full-loop protocol** — the four rounds stitched into one timed session (170 min), self-run or with a peer, so you rehearse the *sequence and the fatigue*, not just the parts.
- **The readiness rubric** — five pass/fail dimensions with an explicit **fail signal** for each, plus a map from every failure back to the module that fixes it. Passing the gate = five clean passes in one unaided run. Anything less is a remediation list, not a pass.

The meta-skill this trains: honest self-scoring. The rubric is written so you cannot fudge it — each dimension has a concrete disqualifier (you quoted a utilization number as truth; a STAR story had no metric; a design answer never named a failure mode). If any fires, that dimension fails, full stop, regardless of how good the rest of the answer sounded.

## Core notes

### The mock-loop shape
Mirror lesson 01's consolidated neocloud loop, compressed into one sitting. Four rounds, 170 minutes:

| # | Round | Time | What you run |
|---|---|---|---|
| 1 | Recruiter / HM screen | 35 min | 90-sec positioning pitch → resume-driven domain Q&A → your questions for them |
| 2 | GPU/infra system design | 45 min | Pick **one** of P1–P6 (lesson 05); design it out loud, recorded |
| 3 | Deep-dive / debugging | 45 min | Run **D2 — the 100%-util trap** (lesson 06) end-to-end |
| 4 | Behavioral / staff-signal | 45 min | Deliver **3 STAR stories** (lesson 08), interviewer picks the prompts |

Run it as one block with only a 5-min break between rounds — the back-to-back is the point. Record every round (audio minimum; screen for round 2's whiteboard). If you have a peer, they play interviewer and withhold prompts (the "do you volunteer it?" test); solo, you read the prompt cold and start a timer. **No notes, no pausing, no re-takes.** A run where you stopped to look something up is a practice rep, not a gate attempt.

Round 3 note — **D2, the 100%-util trap**, is the mandatory debug round because it is the single most discriminating GPU-infra reflex: the interviewer says "utilization is pegged at 100% but throughput is half of yesterday — go." The whole test is whether you *distrust the util number* and pivot to MFU/goodput instead of celebrating the 100%, then bisect the three suspects — **compute** (throttling, a bad kernel, wrong precision), **comms** (NCCL collective stalls, a link renegotiated down, a straggler rank), and **data** (loader starvation, NUMA-misplaced pinning, storage bottleneck). Talking through hypothesis → command → expected reading → next branch is the signal.

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
- Bank the scored record + remediation list as the readiness artifact and fold it into the [GPU platform capstone](../practice/gpu-platform-capstone/README.md). A clean five-dimension pass is your green light to open the campaign — neoclouds first (lesson 01 sequencing).

## Self-check
- What is the single most common disqualifier in the D2 debugging round, and what is the correct move instead? **Answer:** Quoting the utilization number as truth — treating 100% util as "the GPU is working hard, all good." The correct move is to distrust it (util = a kernel is resident, not useful FLOPs), pivot to MFU/goodput, then bisect compute vs comms vs data with a named command and expected reading at each branch.
- Why is the gate one back-to-back 170-min loop instead of four separately-passed rounds? **Answer:** Because a real loop tests integration and fatigue — the failure mode is interference (nailing design, then arriving at behavioral drained and delivering a metric-free story). Passing slices in isolation doesn't prove you can clear all four cold, in sequence, on a clock, unaided — which is the skill actually tested.
- A candidate delivers three vivid, well-structured STAR stories but the debrief still fails the behavioral dimension — what most likely fired, and where does it map? **Answer:** A metric-free story (no number) and/or IC-scope instead of staff-scope — either disqualifies the dimension regardless of polish. It maps back to M12 L08 (STAR + staff signal): attach a hard metric and a tradeoff/reversibility note to every story, and frame it as multi-team scope.

## Resources
- Exponent — ML system design interview guide (tryexponent.com)
- Exponent — behavioral / leadership interview guide (tryexponent.com)
- ai-engineering-field-guide — interview trends & loop shapes (github.com/alexeygrigorev/ai-engineering-field-guide)
- Your own capstone — the scored mock-loop record + remediation list are the readiness artifact
- [🎓 12 — Capstone & interview preparation](../README.md)
