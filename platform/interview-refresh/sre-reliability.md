---
area: "SRE & reliability"
kind: refresh
status: not-refreshed      # not-refreshed | refreshed
---

# 🔁 SRE & reliability — interview refresh

> SLOs, error budgets, incident response, capacity planning, chaos.
>
> You know this. Goal here is fast recall + crisp interview framing, not study.

## Talking points to have ready

- **SLI/SLO/error-budget, applied.** SLIs from the user's perspective (availability, latency,
  correctness, freshness); SLOs set the target; the **error budget is the decision tool** — spend
  it on velocity, freeze on burn. Say how the budget changes team behavior, not just the formula.
- **Burn-rate alerting (the senior signal).** Multi-window, multi-burn-rate alerts (fast-burn
  page, slow-burn ticket) instead of static thresholds — fewer false pages, catches both the
  outage and the slow leak. Alert on **symptoms (user-facing SLI)**, not causes; every page is
  actionable or it's deleted.
- **Incident response.** Clear roles (IC / comms / ops), a severity ladder, mitigate-before-
  diagnose, and **blameless postmortems** whose actions actually get staffed. The metric is
  time-to-mitigate, and the artifact is the follow-up that prevents recurrence.
- **Capacity & load management.** Headroom planning, load shedding, graceful degradation,
  backpressure, and **the retry-storm/thundering-herd traps** (retry budgets, jitter, circuit
  breakers). Tie to Little's Law and queue theory.
- **The GPU-fleet twist.** Reliability of a GPU fleet is **goodput, not uptime** — a node can be
  "up" and silently degrading (lemon/gray failure), and the cost of a slow node is enormous
  ($/GPU-hr × a stalled all-reduce). SLOs shift toward *job success rate, time-to-detect a bad
  node, and goodput* rather than request availability (ties to the GPU observability + bare-metal
  health modules).

## Self-quiz

- Why is a multi-window multi-burn-rate alert better than "error rate > 1% for 5m"?
- Symptom vs cause alerting — which pages, which tickets, and why?
- What's a retry storm and the three levers that prevent it? *(retry budget, jitter, circuit
  breaker.)*
- For a GPU training fleet, what's the right reliability SLI — and why isn't it "node uptime"?

## Refresh only if

- **The Google SRE Workbook burn-rate chapter** if you set alerts by static threshold — skim the
  multi-window recipe.
- **ML/GPU-specific reliability** framing (goodput, straggler/lemon-node detection, time-to-detect
  targets) if you've only run request-serving SRE — this is the part interviewers at GPU shops push.
