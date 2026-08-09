---
id: "12"
title: "Capstone project and interview preparation"
notion: "https://app.notion.com/p/3b33abaeb8238190b651c1c904390a26"
phase: "Track C · continuous from ~month 4; interview ramp months 9–15"
effort: "~33 hrs of dedicated prep + continuous capstone assembly"
status: not-started        # not-started | in-progress | checkpoint-passed
prerequisites: ["01","02","03","04","05","06","07","08","09","10","11"]
unlocks: []
started: null
completed: null
---

# 🎓 12 — Capstone project and interview preparation

> **Goal.** Convert eleven modules of study into **one coherent body of evidence** and the
> ability to interview at GPU-fleet operators. This is **Track C** — it runs *continuously*
> (the capstone is assembled from Tracks A/B as you go) and the interview drills ramp in the
> final months. The output is a portfolio a hiring manager understands without meeting you,
> plus a rehearsed loop you can pass unaided.

- **Notion page:** https://app.notion.com/p/3b33abaeb8238190b651c1c904390a26
- **Phase:** Track C · continuous · **Est. effort:** ~33 hrs prep + capstone assembly
- **Prerequisites:** all of `01`–`11` feed it · **Unlocks:** the job search itself
- **Deliverable:** [GPU platform capstone](practice/gpu-platform-capstone/) — one reference
  "GPU platform" repo + design doc + flagship blog + demo, threading all 11 artifacts.

## Why this module, and to what bar

The 2025-2026 GPU-infra loop is **not** a generic platform loop — and it is not DSA-grinding:

- **Neoclouds are the highest-signal, lowest-friction target.** CoreWeave JD language *is* this
  course's syllabus ("custom controllers and operators… measurable latency/throughput/
  reliability… GPU performance-testing platform… Grafana/Prometheus… lead RCA"); **Lambda**
  runs a live SSH-into-a-box round and *explicitly no LeetCode*; **Together** probes KV
  cache/batching depth; **Modal** probes scheduler/cold-start. Your `gpu-cost-operator` (02)
  and "dashboard is lying" (05) are their interview, pre-answered.
- **The staff bar is scope, ambiguity, and influence — not more coding.** Interviewers score
  *which problem was worth solving*, tradeoffs made, and cross-team influence without authority.
- The loop shape to prep: **recruiter → live technical (optimization-flavored, no pure DSA at
  neoclouds) → GPU system-design → deep-dive/debugging → staff behavioral → (sometimes) 4-8h
  take-home.**

## Calibrated to your background — what we skip

You're sharp on generic platform system design, generic behavioral/STAR mechanics, DSA-adjacent
coding, and portfolio hygiene. So this module **skips all of that** and spends only on the
delta: the **GPU-infra system-design prompt set (P1-P6)**, the **util-lies debugging round**,
**GPU cost attribution as system design** (your home-field advantage — drill yourself to
*volunteer* it), the **company-specific loop intelligence**, and **packaging 11 artifacts into
promotable, multi-team-scope staff evidence.**

## Lessons

Two arcs: **build the evidence** (01-04) and **drill the loop** (05-09).

| # | Lesson | Hrs | What it produces |
|---|--------|-----|------------------|
| 01 | [The GPU-infra hiring landscape (2025-26)](lessons/01-hiring-landscape.md) | 3 | a per-company loop map + your sequencing |
| 02 | [Capstone synthesis: 11 artifacts → one platform](lessons/02-capstone-synthesis.md) | 4 | the value-chain thesis + storyboard |
| 03 | [Portfolio & writeup build](lessons/03-portfolio-writeup.md) | 4 | repo README + RFC design doc + diagram |
| 04 | [Flagship blog + demo: "your dashboard is lying"](lessons/04-flagship-blog-demo.md) | 3 | lead blog + 5-min narrated demo |
| 05 | [GPU system-design drills (P1–P6)](lessons/05-system-design-drills.md) | 5 | six probe-axis rubrics, drilled |
| 06 | [Deep-dive / debugging drills (D1–D3)](lessons/06-debugging-drills.md) | 4 | three memorized decision trees |
| 07 | [Narrate your artifacts](lessons/07-narrate-artifacts.md) | 3 | 3-min artifact→decision→tradeoff→metric stories |
| 08 | [Staff-signal behavioral / STAR](lessons/08-staff-behavioral.md) | 3 | 8–10 staff STAR stories |
| 09 | [Mock loop & the readiness gate](lessons/09-mock-loop-readiness.md) | 4 | a scored full-loop + remediation list |

Total ≈ **33 hrs** of dedicated prep, on top of continuous capstone assembly.

## The six system-design prompts (P1–P6) and the artifacts that arm them

| Prompt | Key axes probed | Armed by |
|--------|-----------------|----------|
| P1 Multi-tenant GPU platform | passthrough/MIG/time-slice isolation, noisy-neighbor, QoS+preemption | 06, 02, 04, 02b |
| P2 GPU cost attribution/chargeback | DCGM telemetry, attribution formula + its flaws, showback vs chargeback, FOCUS | 01,02,03,04,06,11 |
| P3 Training-job scheduler | gang scheduling, quota borrow/reclaim, topology-aware placement | 06, 02b, 09 |
| P4 Model-serving platform | prefill/decode, KV cache, continuous batching, TTFT/TBT, scale-to-zero | 07, 05 |
| P5 Health detection & remediation | lemon-node detection, fleet DCGM diag, tiered remediation | 08, 04 |
| P6 Fleet observability | util→SM→MFU/goodput hierarchy, alert on goodput not util | 05, 01, 03 |

## Deliverable & checkpoint

- Build the **[GPU platform capstone](practice/gpu-platform-capstone/)**: one reference-platform
  repo (architecture diagram + RFC design doc + the 11 artifacts as chapters), the flagship
  **"your GPU dashboard is lying"** blog + two secondary posts, a 5-minute demo, and a one-page
  quantified brag doc.
- The [**checkpoint**](checkpoint.md) is the **whole-course readiness gate** — pass a timed mock
  loop (system design + debugging + behavioral + artifact narration) unaided, with a published,
  self-explanatory portfolio.

## How to work this module

1. Assemble the capstone **continuously** — every module's deliverable is a chapter; don't leave
   it to the end.
2. Lead your applications with **neoclouds** (no-DSA, your artifacts are their syllabus); treat
   NVIDIA/big-tech as the DSA-flavored tier.
3. Run the [checkpoint](checkpoint.md) as a timed mock; remediate each failed dimension against
   the module that fixes it; re-run until all five pass, then start applying.
