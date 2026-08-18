---
area: "SRE & reliability"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 SRE & reliability — interview refresh

> SLOs, error budgets, incident response, capacity planning, chaos.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

SRE interviews reward two things: **arithmetic you can do out loud**, and **mechanisms
for failures that get worse when you retry them**. Almost every question here is one of
those. The GPU-fleet variant adds a third: reliability on a training fleet is measured in
*goodput*, not uptime, because a node can be perfectly "up" while burning $2.50/GPU-hour
producing nothing.

Sourcing note: `sre.google` (the SRE Workbook) is egress-blocked from this environment, so
the multi-window burn-rate figures below are **derived from first principles** and
cross-checked against the error-budget percentages documented in `slok/sloth`'s
`internal/alert/window.go` (read 2026-08-18), which implements the Workbook recipe. The
Llama 3 and incident figures are carried forward from this course's GPU modules with their
original caveat: arXiv and several vendor blogs are unreachable here, so treat them as
second-hand until re-verified against the primary source.

## Talking points to have ready

### 1. SLI → SLO → error budget, with the arithmetic

An **SLI** is a ratio of good events to valid events, measured where the user is. An
**SLO** is a target for that ratio over a window. The **error budget** is `1 − SLO` of
that window, and it is the only part that changes behaviour, because it converts
reliability from an argument into a quantity.

The table you should be able to produce from memory (30-day window = 43,200 minutes):

| SLO | Budget (fraction) | Budget per 30 days | Per 7 days |
|---|---|---|---|
| 99% | 10⁻² | 432 min ≈ 7.2 h | 100.8 min |
| 99.5% | 5×10⁻³ | 216 min = 3.6 h | 50.4 min |
| 99.9% | 10⁻³ | 43.2 min | 10.1 min |
| 99.95% | 5×10⁻⁴ | 21.6 min | 5.0 min |
| 99.99% | 10⁻⁴ | 4.32 min | 1.0 min |

Two follow-ups interviewers use to check whether you actually run this:

- **"Where does the number come from?"** Not from ambition. From what the business can
  observe and what the dependencies allow. If you serve on top of three dependencies each
  at 99.9% and you call all three serially per request, your ceiling is ~99.7% before you
  write a line of code. The right move is to set the SLO just tight enough that violating
  it is a real signal, then let the error budget do the arguing.
- **"What happens when it's spent?"** A policy agreed *before* the incident: feature
  freeze, all deploys go to reliability work, and it lifts when the budget recovers. If
  nothing changes when the budget is exhausted, the SLO is decoration.

### 2. Burn-rate alerting — derive it, don't memorise it

Static thresholds ("error rate > 1% for 5 minutes") page for blips and miss slow leaks.
The multi-window multi-burn-rate recipe fixes both. **Burn rate** is how fast you are
consuming budget relative to spending it evenly across the whole window:

```
  burn_rate = observed_error_ratio / (1 − SLO)

  burn_rate 1   → the budget lasts exactly the SLO window
  burn_rate 14.4 → the budget is gone in 30 days / 14.4 ≈ 50 hours…
                   …but that is not how it is used: pick the % of budget you are
                   willing to lose before paging, and solve for the burn rate.

  budget_fraction_consumed = burn_rate × (alert_window / SLO_window)

  Page at 2% of budget in 1 h  → 0.02 / (1/720)   = 14.4×
  Page at 5% of budget in 6 h  → 0.05 / (6/720)   =  6×
  Ticket 10% of budget in 1 d  → 0.10 / (24/720)  =  3×
  Ticket 10% of budget in 3 d  → 0.10 / (72/720)  =  1×
```

(720 = hours in a 30-day window. The 2 / 5 / 10 / 10 percent choices are the Google SRE
Workbook defaults, documented in `slok/sloth`'s window definitions as "page quick 2%,
page slow 5%, ticket quick 10%, ticket slow 10%".)

```
  Each alert = LONG window (is it really burning?) AND SHORT window (is it still burning?)

   burn
   rate
    ▲
14.4├───────╮                     ← page: 1 h long / 5 m short
    │       │  fast burn: whole budget in hours
  6 ├───────┼──────╮              ← page: 6 h long / 30 m short
    │       │      │
  3 ├───────┼──────┼──────╮       ← ticket: 1 d long / 2 h short
  1 ├───────┼──────┼──────┼────╮  ← ticket: 3 d long / 6 h short
    └───────┴──────┴──────┴────┴────────────▶ time to exhaust budget
      hours            days           30 days

  The SHORT window (usually 1/12 of the long one) exists to stop alerting once
  the burn has stopped — without it, a 3-day window keeps you paged for 3 days
  after a 10-minute outage ends.
```

Say the two design rules alongside it: **alert on symptoms, not causes** — page on the
user-visible SLI (requests failing, latency out of objective), ticket or dashboard the
causes (queue depth, disk, replica count), because a symptom alert catches failure modes
you didn't think of and a cause alert fires when nobody is hurt. And **every page must be
actionable**: if the response is "acknowledge and go back to sleep", delete it or demote
it. Page volume is a budget too.

### 3. Incident response as a lifecycle

```
  t0 ────────────── t1 ─── t2 ───────── t3 ──────────── t4 ─────────────▶
  fault      detection   ack     mitigation        resolution      review
  begins     (alert)    (human)  (users OK)        (root fixed)   (postmortem)

  │◀── MTTD ──▶│◀ MTTA ▶│◀──── MTTM ────▶│
  │◀──────────────── MTTR (as customers experience it) ─────────────▶│

  ROLES from t1:
    Incident Commander — owns the incident, makes calls, assigns; does NOT debug
    Communications      — status page, stakeholders, cadence updates
    Operations/Subject   — hands on keyboard, one thread of work at a time
    Scribe               — timeline, decisions, timestamps (feeds the postmortem)

  RULE: mitigate before you diagnose. Roll back, fail over, shed load, scale out.
        Understanding the bug is a t3–t4 activity; users are waiting at t2.
```

The parts that separate a senior answer:

- **The severity ladder must map to behaviour**, not adjectives: who is paged, who is
  informed, whether an IC is required, whether a status page goes public, and what the
  update cadence is. "Sev-2 means customers see errors and we update every 30 minutes" is
  a definition; "Sev-2 means quite bad" is not.
- **Mitigation levers, ranked by how fast they work**: revert the last change (usually the
  cause), fail over to a healthy replica/region, shed load or disable an expensive
  feature, scale out, then fix forward. Deploy freezes and rollbacks are cheap; heroic
  debugging under pressure is not.
- **Blameless postmortems whose actions get staffed.** The artifact is not the document,
  it is the follow-up work that appears in the next sprint with an owner. The honest
  metric of a postmortem culture is the completion rate of action items, and the honest
  test of blamelessness is whether people report their own mistakes without prompting.
- **Time-to-mitigate is the metric that matters**, and it is dominated by detection and
  the decision to act, not by typing speed. Most improvements come from better SLI-based
  alerting (t0→t1) and pre-agreed authority to roll back (t1→t2).

### 4. Load management: the failures that get worse when you retry

- **Little's Law: `L = λ × W`.** Concurrency = arrival rate × time in system. It is the
  fastest sanity check in an interview: a service handling 200 req/s at 500 ms mean
  latency has ~100 requests in flight, so a 50-thread pool is already the bottleneck and a
  200-connection pool sets your queue depth. Reverse it too — if latency doubles under
  load and arrivals don't fall, in-flight work doubles, and that is what exhausts pools.
- **Utilisation kills tail latency non-linearly.** For a simple M/M/1 queue, wait time
  scales as `1/(1−ρ)`: at 50% utilisation you wait one service time, at 90% you wait nine,
  at 95% nineteen. That is why capacity plans target 60–70% steady-state utilisation and
  why "we're only at 85% CPU" is not comfortable. (Real systems aren't M/M/1 — the shape
  is the point, not the constant.)
- **Retry storms.** A dependency slows down, clients time out and retry, the retries
  multiply the offered load, the dependency slows further. Three levers, and you should
  name all three: **retry budgets** (cap retries at a small fraction of the request rate —
  ~10% is the usual starting point — and drop them entirely when the budget is spent),
  **exponential backoff with full jitter** (`sleep = random(0, base × 2^attempt)`, so
  clients de-synchronise instead of retrying in lockstep), and **circuit breakers**
  (closed → open after a failure threshold → half-open trial request → closed), which stop
  sending doomed traffic at all. The empirical case for budgets over backoff alone:
  Slack's 2022-02-22 incident had clients already doing exponential backoff with jitter,
  and the aggregate load still sustained the outage — per-client correctness says nothing
  about the aggregate. *(Carried forward from the course's GPU modules; the Slack
  engineering blog is not fetchable from this environment.)*
- **Metastable failure** is the general form: a trigger pushes the system into a state
  where the *sustaining effect* (retries, cache misses, queue backlog) keeps it there even
  after the trigger is gone. The tell is that removing the original cause does not restore
  service. Recovery requires dropping load *below* the threshold that got you in —
  hysteresis — which in practice means shedding aggressively, then admitting traffic back
  slowly. "Metastable Failures in the Wild" (OSDI '22) documents 22 such incidents across
  11 organisations, including several major cloud outages. *(Cited from this course's
  modules; the paper was not fetched here.)*
- **Graceful degradation** is designed, not emergent: serve cached or stale results,
  disable the expensive personalisation path, return a smaller model's answer, drop
  optional enrichment. The question to be ready for is "what does your system do when it
  is 30% short of capacity?" — the good answer describes a degraded mode, the bad one
  describes an outage.

### 5. Capacity planning

Four inputs, and you should say all four: demand forecast (organic growth + known launches
+ seasonality), per-unit capacity measured under realistic load (not from a datasheet),
headroom for failure domains (can you lose an AZ/a rack and still serve?), and lead time
for the resource that takes longest to acquire. On a GPU fleet the last one dominates —
compute you cannot buy for eight weeks is a planning constraint, not a scaling knob.

Load-test to find the knee, not the ceiling: the useful number is the utilisation at which
p99 starts to climb, because that is where your headroom target belongs. Then verify the
failure assumptions with chaos experiments — kill a node, blackhole a dependency, add 200
ms of latency — with a hypothesis and a blast-radius limit, in production if you can do it
safely, because staging does not have your traffic mix.

### 6. The GPU-fleet twist: goodput, not uptime

The framing that makes this a GPU-platform answer rather than a web-service one:

```
  MFU     = useful tensor-core FLOP time / productive time   "how well does a step run?"
  ETTR    = productive time / allocated time                 "how much of what we rent
                                                              is spent doing steps?"
  GOODPUT = MFU × ETTR                                       "how much of what we PAY FOR
                                                              becomes model"

  $ per useful GPU-hour = rate / goodput
```

- **A node can be `Ready` and worthless.** Gray failures — a GPU throttling on thermals, a
  NIC negotiating a lower rate, ECC errors triggering retries, a "lemon" node that is 20%
  slower — do not show up as downtime. In a synchronous training job they show up as
  *everyone else waiting*, because an all-reduce completes at the speed of the slowest
  rank. Uptime is 100%; goodput is down 20%.
- **Cost the straggler.** One degraded node in a 64-node × 8-GPU job stalls 512 GPUs. At a
  $2.50/GPU-hour snapshot rate that is $1,280 per hour of undetected slowness — which is
  why *time-to-detect a bad node* is a first-class SLI on a GPU fleet and why automated
  cordon-and-replace pays for itself immediately. (Rate is a dated snapshot; redo the
  arithmetic with yours.)
- **What real fleets measure.** Meta's Llama 3 run reported 466 job interruptions over 54
  days on ~16K H100s (419 unexpected), sustained **>90% effective training time**, and
  attributed roughly 78% of unexpected interruptions to confirmed or suspected hardware.
  That is the shape of the problem: interruptions are routine and constant, so reliability
  engineering is about *fast detection and automatic recovery*, not about preventing
  failure. *(Figures carried forward from this course's GPU modules; arXiv is unreachable
  here — verify against the paper's infrastructure section before quoting externally.)*
- **SLIs that actually fit.** For training: job success rate, time-to-detect a degraded
  node, time-to-recover (checkpoint restore + re-rendezvous), checkpoint recency (how much
  work is at risk right now), and ETTR/goodput as the top-line. For inference: request
  success rate, **time-to-first-token separately from time-per-output-token** (they have
  different bottlenecks — prefill is compute-bound, decode is memory-bandwidth-bound),
  queue depth, and tokens/second per GPU. "Availability" alone is close to useless for
  both.
- **The error-budget analogue.** On a training fleet the budget is *lost GPU-hours*, and
  the same policy logic applies: if unplanned interruption overhead exceeds its target,
  reliability work outranks feature work — and, unusually for SRE, you can price it
  directly, because a lost GPU-hour has an invoice line.
- **Checkpointing is the reliability control.** Checkpoint interval trades steady-state
  overhead against expected redo work: the expected loss per interruption is half the
  interval, so with an interruption every few hours a 30-minute interval risks ~15 minutes
  of redo per event. The right interval is where checkpoint write cost and expected redo
  cost meet — and that calculation depends on checkpoint size and storage bandwidth, which
  is why storage throughput is a reliability parameter on a GPU fleet, not a storage
  detail.

## Self-quiz

**1. Why is a multi-window multi-burn-rate alert better than "error rate > 1% for 5m"?**
Because a static threshold has no notion of how much of your budget the event costs. 1% for
five minutes might be trivial for a 99% service and a serious burn for a 99.99% one, and a
0.5% leak that never crosses the threshold can consume the entire month's budget
undetected. Burn-rate alerting normalises by the budget: page at 14.4× over a 1-hour
window (2% of a 30-day budget) and at 6× over 6 hours (5%), and open a ticket at 3× over a
day or 1× over three days (10%). The second window in each pair is short (about 1/12 of
the long one) and exists to *stop* alerting when the burn stops — without it, a long
window keeps you paged for hours after a brief outage has ended. Net effect: fewer false
pages, and you catch both the sharp outage and the slow leak with one mechanism.

**2. Symptom vs cause alerting — which pages, which tickets, and why?**
Symptoms page. They are defined on the user-visible SLI, so they fire for every failure
mode including the ones you did not anticipate, and each one has a customer impact you can
state. Causes ticket or go on a dashboard: high CPU, a full queue, a lagging replica, a
node down. Two reasons. First, causes fire when nobody is hurt — redundancy means one dead
replica is a Tuesday, not an incident — and those pages train people to ignore pages.
Second, cause alerts only cover failures you predicted, so a symptom-free cause list gives
false confidence. The exception worth naming: *imminent, irreversible* causes with no
symptom yet — disk filling in 20 minutes, a certificate expiring tonight, an error budget
about to hit zero — legitimately page, because by the time the symptom appears it is too
late to act.

**3. What is a retry storm and the three levers that prevent it?**
A dependency degrades, clients time out and retry, retried load multiplies the offered
rate (a 3× retry policy triples it), the dependency degrades further, and the system stays
down after the original trigger is gone — the metastable pattern. Levers: **retry budget**
(retries capped at a fraction of base traffic, typically around 10%, dropped when
exhausted — this is the aggregate control, the only one that bounds total load);
**exponential backoff with full jitter** (`random(0, base × 2^attempt)`, so clients spread
out instead of resonating); and **circuit breakers** (open after a failure threshold, stop
issuing requests entirely, half-open to probe recovery). Add two supports: deadline
propagation so a request that is already doomed is not retried at three layers, and load
shedding at the server so it can protect itself when clients misbehave. The Slack
2022-02-22 incident is the case study: correct per-client backoff with jitter, and the
aggregate still sustained the outage.

**4. For a GPU training fleet, what is the right reliability SLI — and why isn't it "node
uptime"?**
Goodput — the fraction of paid GPU-time that becomes useful FLOPs (`MFU × ETTR`) — with
job success rate, time-to-detect a degraded node, and time-to-recover as the operational
SLIs beneath it. Node uptime fails for three reasons. It is blind to gray failure: a
thermally throttled GPU, a NIC that renegotiated to a lower rate, or a lemon node is
`Ready` while costing you 20% of a run. It has the wrong blast unit: in a synchronous job
one bad rank stalls every other rank, so a 1-node problem is a 512-GPU outage, and uptime
counts it as one node. And it ignores recovery cost: an interruption costs the redo work
back to the last checkpoint plus restart, not just the seconds the node was down. Real
runs at scale (Meta's Llama 3: 466 interruptions in 54 days, >90% effective training time,
~78% hardware-attributed) show interruptions are constant, so the engineering goal is fast
detection and automatic recovery, measured in goodput.

**5. Your error budget is 80% spent on day 10 of a 30-day window. What do you do?**
First, characterise the burn: one big event or a persistent leak? A single incident
already mitigated means the budget is spent but the risk is gone — the response is the
postmortem and the action items, not a freeze. A continuing burn means every day costs
more budget than it should, and the pre-agreed policy applies: stop feature deploys,
redirect the team to the top burn source, and communicate the freeze with the number
attached. Second, check the SLI itself before penalising anyone — a badly defined SLI
(counting health-check traffic, counting client-cancelled requests as errors, measuring at
the wrong boundary) is a surprisingly common cause of a "budget crisis" that no user
experienced. Third, if the budget is regularly exhausted by things nobody thinks are
serious, the SLO is set wrong and the fix is renegotiation with the number as the evidence.

**6. How do you upgrade a fleet running multi-day training jobs without wrecking
reliability?**
Treat it as a scheduling problem with a reliability budget rather than a maintenance
window. Cordon nodes so no new job lands, let running jobs reach a checkpoint boundary,
then drain — never drain a synchronous job's node mid-step, because every rank dies with
it. Roll by node pool with a canary pool first, and keep the driver/CUDA/operator version
matrix pinned per pool so a bad combination is contained. Track the cost explicitly: the
lost GPU-hours during the roll come out of the same budget as unplanned interruptions, so
a rolling upgrade that takes a week of reduced capacity is a reliability decision the
budget can price. And make checkpointing frequent enough that "wait for the next
checkpoint" is minutes, not hours — the checkpoint interval is what converts a
maintenance need into a schedulable event.

## Refresh only if

- **The Google SRE Workbook burn-rate chapter**, if you still set alerts by static
  threshold. The recipe to internalise is in §2: choose the fraction of budget you will
  spend before paging (2% / 5% for pages, 10% for tickets), divide by the window fraction
  to get the burn rate, and pair every long window with a short one at roughly 1/12 its
  length so alerts clear when the burn stops.
- **ML/GPU-specific reliability framing**, if your background is request-serving SRE:
  goodput = MFU × ETTR, straggler and lemon-node detection, time-to-detect as a
  first-class SLI, checkpoint interval as the tunable that converts interruptions into
  bounded redo work, and the fact that at 10K+ accelerators interruptions are continuous
  rather than exceptional.
- **Metastable failures**, if your model of cascading failure is "it fell over and we
  restarted it": the trigger/sustaining-effect distinction and the hysteresis requirement
  (you must drop load below the entry threshold to recover) explain why restarts alone
  often do not work, and it is a common staff-level probe.

## Recall card

Cover the right column and say each value out loud; if one is fuzzy, reread the section
in brackets.

| Fact | Value |
|---|---|
| 99.9% budget | 43.2 min per 30 days (43,200 min × 10⁻³) [§1] |
| 99.95% / 99.99% | 21.6 min / 4.32 min per 30 days [§1] |
| Burn rate | `error_ratio / (1 − SLO)`; budget consumed = `burn × window/SLO_window` [§2] |
| Page thresholds | 14.4× over 1 h (2% budget) · 6× over 6 h (5%) [§2] |
| Ticket thresholds | 3× over 1 d · 1× over 3 d (10% each) [§2] |
| Short window purpose | ≈1/12 of the long window; stops alerting when burn stops [§2] |
| Alerting rule | page on symptoms, ticket on causes; every page actionable [§2] |
| Incident roles | IC (decides, doesn't debug) · comms · ops · scribe [§3] |
| Incident rule | mitigate before diagnose; MTTM is the metric [§3] |
| Little's Law | `L = λW` — concurrency = rate × latency [§4] |
| Queue behaviour | wait ∝ `1/(1−ρ)`: 90% utilisation ⇒ ~9× service time [§4] |
| Retry storm levers | retry budget (~10%) · full jitter `random(0, base×2^n)` · circuit breaker [§4] |
| Metastability | trigger + sustaining effect; recovery needs load below entry threshold [§4] |
| GPU top-line SLI | goodput = MFU × ETTR; `$/useful GPU-hr = rate / goodput` [§6] |
| Straggler cost | 1 bad node in a 64-node job stalls 512 GPUs [§6] |
| Llama 3 reference point | 466 interruptions / 54 days / ~16K H100s, >90% ETTR, ~78% hardware [§6] |
