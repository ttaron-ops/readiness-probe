# Depth map — Module 12 · Capstone & interview

Pointers into [`harut8/system-design`](https://github.com/harut8/system-design). **Open a chapter
only when a lesson's artifact is blocked on internals you don't have** — see
[`docs/EXTERNAL-DEPTH.md`](../../../docs/EXTERNAL-DEPTH.md) for how to use this library and the
attribution/licensing note.

> **The interview-prep material is the strongest fit in the repo for this module** — a full design
> framework, four worked design documents, and the same problems implemented at four scale tiers.

| Lesson | Go deeper in | Why |
|---|---|---|
| 02 Capstone synthesis | [`k8s-learn/gpu-platform-tasks`](https://github.com/harut8/system-design/blob/main/k8s-learn/gpu-platform-tasks.md) | five projects that are a near-exact restatement of this capstone; steal its acceptance criteria and its **"label the simulation honestly"** publishing advice |
| 03 Portfolio write-up | [`solutions/`](https://github.com/harut8/system-design/tree/main/solutions) | four complete design documents — read for the *shape* of a finished write-up (requirements → estimates → API → design → tradeoffs), not the architectures |
| 04 Flagship blog / demo | [`ai-rag/labs/`](https://github.com/harut8/system-design/tree/main/ai-rag/labs) | a build-log-with-runnable-code format worth copying: benchmark harness, fixtures, tests, and results in one directory |
| 05 System-design drills | [`SYSTEM-DESIGN-GUIDE.md`](https://github.com/harut8/system-design/blob/main/SYSTEM-DESIGN-GUIDE.md) | the 6-step interview framework, back-of-envelope section, and a list of common mistakes — the compact companion to `platform/01` L08 |
| 05 System-design drills | [`implementation/`](https://github.com/harut8/system-design/tree/main/implementation) | the same problem at 10k → 10m scale, which seeded the [design-drill ladder](../../../platform/01-distributed-systems-and-design/practice/staff-design-portfolio/design-drills.md) |
| 05 System-design drills | [`solutions/api-design-patterns`](https://github.com/harut8/system-design/blob/main/solutions/api-design-patterns.md) · [`big-tech-api-standards`](https://github.com/harut8/system-design/blob/main/solutions/big-tech-api-standards.md) | API design is the step most people rush; these are checklists you can drill against |
| 06 Debugging drills | [`gpu-observability/16-incident-walkthrough`](https://github.com/harut8/system-design/blob/main/gpu-observability/16-incident-walkthrough.md) | a GPU slowdown traced end to end — a ready-made worked rep |
| 06 Debugging drills | [`gpu-observability/appendix-c-flowcharts`](https://github.com/harut8/system-design/blob/main/gpu-observability/appendix-c-flowcharts.md) | symptom → decision-tree triage. Drill against these until you don't need them. |
| 06 Debugging drills | [`kubernetes/06-admission-control-deep-dive`](https://github.com/harut8/system-design/blob/main/kubernetes/06-admission-control-deep-dive.md) | "the #1 source of cluster outages" — a rich source of realistic break/fix scenarios |
| 09 Mock-loop readiness | [`sre-observability/15-incident-response-and-postmortem`](https://github.com/harut8/system-design/blob/main/sre-observability/15-incident-response-and-postmortem.md) | incident-command structure and postmortem writing — the substance behind staff behavioural stories |
| 09 Mock-loop readiness | [`sre-observability/17-production-readiness-reviews`](https://github.com/harut8/system-design/blob/main/sre-observability/17-production-readiness-reviews.md) | a PRR checklist doubles as an interview framework for "how would you launch this?" |

## The stretch artifact

[`kubernetes/38-building-a-kubernetes-from-scratch`](https://github.com/harut8/system-design/blob/main/kubernetes/38-building-a-kubernetes-from-scratch.md)
— a `minik8s` capstone: apiserver, scheduler and kubelet built from nothing. Too big for this
course's timeline and **not** a substitute for the GPU cost operator, which is the differentiated
artifact. But if a control-plane role becomes the specific target, this is the alternative flagship.

## Honest framing, borrowed

The single most useful paragraph in the source repo for this module, from `gpu-platform-tasks.md`:

> *"I built a GPU scheduler plugin and benchmarked it against the default on a 200-node simulated
> fleet"* is accurate, impressive, and cannot be punctured. *"I built GPU scheduling
> infrastructure"* cannot survive a follow-up question.

Applies to everything you publish from the
[fake GPU fleet](../../04-gpu-on-kubernetes/practice/fake-gpu-fleet/README.md): name the
simulation, publish the trace, state where your design is *worse* than the incumbent. That last
item is what makes the rest believable.
