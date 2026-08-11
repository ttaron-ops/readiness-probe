---
lesson: 06
title: "Deep-dive / debugging (incident) round drills"
module: 12
concept: "GPU incident debugging"
status: not-started
est_time: "5 hrs"
prev: "05-system-design-drills.md"
next: "07-narrate-artifacts.md"
artifacts: ["gpu-debug-decision-trees", "d2-live-walkthrough"]
sources: 9
---

# Deep-dive / debugging (incident) round drills

## Where this fits
Lesson 05 built fluency in the whiteboard system-design round — six prompts drilled until you volunteer scale, cost, failure, and SLO on reflex. This lesson drills a distinct, harder-to-fake round: the live debugging/incident interview, where you're handed a broken system (or a real SSH session) and have to reason live, out loud, with no slide deck to hide behind. Design rounds reward architectural fluency; debugging rounds reward disciplined diagnostic process under pressure — and neoclouds increasingly weight this round because it's the one candidates can't prep by memorizing an architecture diagram.

## Why this matters
The neoclouds lean on the debugging round harder than on anything else, because it's the round you can't fake. Lambda runs a **live SSH round** — you're dropped onto a real box and asked to reason aloud. CoreWeave runs incident RCA. Together probes ML-systems debugging until you either produce a mechanism or hand-wave. Generic "check logs, check metrics, escalate" gets you cut. What passes is a **memorized decision tree** with the GPU-specific first move at each branch, delivered while narrating your hypothesis and the command that would confirm it.

The stakes are not just interview-theoretical. CoreWeave's own engineering writeup on large-scale distributed training opens on a real 1,024-GPU job silently hanging — the exact D1/D2 failure mode this lesson drills — and cites Meta's measured mean-time-to-failure dropping from **7.9 hours at 1,024 GPUs to 1.8 hours at 16,384 GPUs**. At that failure cadence, the speed and correctness of your diagnostic process is a direct cost lever: every minute spent guessing instead of bisecting is billed GPU-hours across a four- or five-figure fleet. Interviewers know this, which is why they score process, not just correct final answers.

The core trap running through all three scenarios: **`nvidia-smi` utilization is a liar.** It reports "a kernel was resident on the SM," not "useful FLOPs happened." Half of GPU debugging is refusing to trust that number and pivoting to goodput/MFU. If you internalize one thing from this lesson, internalize the reflex to distrust util and bisect compute / comms / data.

## What's new here (calibration)
You know how to debug distributed systems. The overlay is three GPU-specific decision trees and the commands that arm each node of the tree:
- `nvidia-smi dmon` — rolling per-second SM/mem/power/pcie counters (better than plain `nvidia-smi` for spotting oscillation).
- `dcgmi diag -r {1,2,3}` — DCGM diagnostics; level 3 is the load/stress test that catches lemon nodes idle checks miss.
- `nsys profile` — Nsight Systems timeline that splits CUDA compute kernels from NCCL communication, so you can *see* whether you're compute-bound or comms-bound.
- `nccl-tests` (e.g. `all_reduce_perf`) — measured collective bandwidth vs the link's theoretical max; the fast way to indict the network/fabric. This is the canonical open-source benchmark tool (github.com/NVIDIA/nccl-tests) that essentially every large training team uses for exactly this check.

Memorize which command sits at which branch. In a live round, naming the exact command *before* you'd run it is the senior tell. Calibrate depth to seniority: a Senior candidate needs to name the right command at each fork; a Staff candidate needs to also say what a *surprising* result at that fork would mean and where it would redirect the investigation.

## Core concepts

**D1 — "A training job is slow — debug end-to-end."** *Arms: 08, 09, 04.*
Decision tree (memorize the order — the order is the point):
1. **Data pipeline first.** GPU starvation from slow dataloaders/storage is the #1 cause. If GPUs are waiting on data, nothing downstream matters.
2. **Test with synthetic data.** Replace the real dataset with in-memory random tensors. If throughput jumps → the bottleneck is **upstream of compute** (loaders, storage, preprocessing). If it doesn't → the bottleneck is compute or comms.
3. **Profile comms.** If utilization drops <50% during gradient sync → suspect the network. Run `nsys profile` to split compute vs NCCL time on the timeline; confirm fabric with `all_reduce_perf` (nccl-tests) against theoretical link BW.
4. **Check sync stalls.** Debug prints, forced `.item()` / `.cpu()` / device transfers, or `torch.cuda.synchronize()` in the hot loop puncture the async graph and serialize the GPU. Hunt these.
5. **Tune.** Microbatch size + gradient accumulation, dataloader `num_workers`/prefetch, gradient bucketing/overlap (DDP bucket size), pinned memory.

Why the order matters, not just the steps: an interviewer who sees you jump to `nsys profile` before ruling out the dataloader is watching you skip a cheap, high-yield check for an expensive, harder-to-interpret one — that ordering choice is itself part of what's being scored, independent of whether you eventually land on the right root cause.

**D2 — "GPUs show 100% util but throughput dropped."** *The flagship trap. Arms: 05 (built for exactly this), 03, 08.*
The insight to state out loud: **util = "a kernel ran," not useful FLOPs.** High util + low goodput has four suspects:
- **Memory-bound kernels** — the SM is busy waiting on HBM, not computing. Low arithmetic intensity.
- **Tiny / oscillating batch** — kernels launch constantly (util pegged) but each does little work; `dmon` shows sawtooth SM%.
- **Comms-bound sync** — time spent in NCCL collectives counts as "busy" but produces no forward progress.
- **Silently degrading lemon node** — one straggler throttling the collective.

Correct move: **pivot the metric.** util → SM occupancy → MFU/goodput → job throughput. Then bisect: `nsys` to split compute vs comms; `dmon` to catch batch oscillation; `dcgmi diag` to catch a degrading node; check batch/KV-cache config for the tiny-batch case. The critical discipline here is bisect-before-fix: distrusting util correctly and then jumping straight to "must be the network, let's check cabling" without first splitting compute/comms/data reads as guessing dressed up as confidence, even when the guess turns out right.

**D3 — "A node keeps failing large jobs but passes health checks."** *Arms: 08, 04.*
Grey/lemon-node decision tree:
1. **Reframe:** "passes idle, fails under load" → idle health checks are structurally insufficient. Treat the fact that it passes idle checks not as evidence the node is fine, but as itself a diagnostic clue pointing at load-dependent failure modes (thermal, power delivery, marginal links) that only manifest under sustained stress.
2. **Load-based diagnostics:** run `dcgmi diag -r 3` (the stress/load level) *under real workload conditions*, not the quick idle check. Look for thermal throttle, ECC under load, NVLink/PCIe errors that only appear at bandwidth.
3. **Correlate:** does the same node ID appear across multiple large-job failures? Build the failure-mode log (module 04) and look for the repeat offender.
4. **Tiered remediation:** reset → cordon+drain+reschedule (via gang scheduler) → **remove from scheduling** and RMA.
5. **Cite the Meta result:** proactive lemon-node removal cut large-job failure rate **14% → 4%**, per Meta's "Revisiting Reliability in Large-Scale Machine Learning Research Clusters" study (arXiv:2410.21680) — measured across two production clusters (RSC-1: 16K A100s, RSC-2: 8K GPUs) over eleven months, with roughly 40 faulty nodes identified at over 85% detection accuracy. Naming the paper, not just the number, is what shows you know the literature rather than repeating a stat you picked up secondhand. Be ready for the follow-up: "what detection signals drove that?" — the answer is repeat-offender correlation across large-job failures plus load-based diagnostic signals (thermal, ECC, link errors under stress), not a single silver-bullet check.

Worth knowing for currency: CoreWeave now ships automated straggler bisection as a product feature (Mission Control, expanded December 2025) — rank-level visibility that replaces the manual `dmon`/`nsys` bisection process with tooling. That doesn't make the manual skill obsolete for an interview; it makes it more important, because you may be the one asked to build or extend that automation, and you can't automate a diagnostic process you can't do by hand.

**"What a strong answer sounds like."** Narrate hypothesis → command → expected signal → next branch. E.g. "My first hypothesis is data starvation, so I'd swap in synthetic data; if throughput jumps I've isolated it upstream of compute and I'd profile the loader; if not, I'd `nsys profile` one step to split compute from NCCL time." You state the branch *and what result sends you down which fork* — that's the tell.

**Live-terminal drill protocol (Lambda-style).** SSH into a box (or a mock). Set a 20-minute timer. Rules: (1) narrate every hypothesis before you type; (2) name the command and the signal you expect *before* running it; (3) never say "check the logs" without saying which log and what you're looking for; (4) when you get a signal, state which branch it sends you to. Record yourself; grade on whether an interviewer could follow your reasoning without asking "why did you run that?" Re-run until two clean reps per scenario.

## Perspectives

**The live-interview-round view.** In a live debugging round, the interviewer isn't just listening for the right diagnosis — they're second-by-second scoring whether you state a hypothesis *before* you act on it. A command run silently, with the correct diagnosis announced only after the output appears, scores lower than the same command preceded by "I expect X; if I see Y instead, that points at Z." The narration isn't decoration on top of the technical skill; in a round designed to be hard to fake, the narration *is* the primary signal being graded, because it's the only window the interviewer has into whether your process would generalize to a failure mode you haven't seen before.

**The on-call/SRE view.** A dashboard curiosity and a page are different things. At 3am, nobody cares that util reads 100% in the abstract — they care whether the job is making forward progress and, if not, what the fastest path to a fix is. This lesson's decision trees are literally the on-call runbooks a real platform team would keep, compressed for interview pacing; when you narrate D1/D2/D3, imagine you're the person who'll actually be paged, not a candidate reciting theory, and let that shape which branch you check first (cheapest and highest-yield, not most theoretically interesting).

**The cost view.** Every minute spent debugging is billed GPU-hours across however many nodes are stalled. On a job at 1,024+ GPUs, the CoreWeave/Meta MTTF numbers cited above (dropping from 7.9 hours to 1.8 hours at larger scale) mean debugging speed compounds: faster, more disciplined bisection isn't just an engineering nicety, it's a direct line item. When you're asked to justify why you'd check synthetic data before diving into `nsys profile`, the honest answer includes "because it's the cheaper check, and cheap-first minimizes billed idle time while I search," not just "because that's the textbook order."

**The tooling-evolution view.** Manual bisection — the kind this lesson drills — is being automated. CoreWeave's Mission Control now ships automated GPU straggler detection that replaces exactly the manual `dmon`/`nsys` rank-comparison process described in D2/D3. That's not a reason to skip the manual skill; it's the reason to know it cold, because the people who build, extend, or debug that automation are the same people who could do the diagnosis by hand first. In an interview, naming this trend ("this is now productized in tools like CoreWeave's Mission Control, but the underlying signal is per-rank step-time variance") shows both technical depth and industry currency.

## Real-world use cases

- **CoreWeave — "Why Distributed Training Fails at Scale."** https://www.coreweave.com/blog/why-distributed-training-fails-at-scale — opens on a 1,024-GPU job silently hanging, exactly D1/D2's failure mode, and cites Meta FAIR's MTTF figures (7.9 hours at 1,024 GPUs, dropping to 1.8 hours at 16,384 GPUs); shows this isn't an interview hypothetical but the default failure mode at scale.
- **Meta — "Revisiting Reliability in Large-Scale Machine Learning Research Clusters."** https://arxiv.org/abs/2410.21680 — the actual paper behind the 14%→4% lemon-node stat drilled in D3; two production clusters (16K and 8K GPUs) over eleven months, ~40 faulty nodes found at >85% detection accuracy. Cite this paper by name, not just the number.
- **CoreWeave Mission Control — GPU straggler detection.** https://www.coreweave.com/mission-control and https://www.coreweave.com/news/coreweave-expands-mission-control-to-accelerate-enterprise-ai-adoption — shipped December 2025; productizes exactly D2/D3's manual bisection skill as automated rank-level straggler detection, evidence this diagnostic process is now industry-standard tooling, not a niche technique.
- **Introl — "Incident Response for GPU Clusters: Playbooks & Failure Scenarios."** https://introl.com/blog/incident-response-gpu-clusters-playbooks-failure-scenarios — a real operator's tiered incident-remediation playbook, directly parallel to D3's reset → cordon/drain → RMA escalation ladder.

## Worked example
**Full D2 walkthrough — "GPUs at 100% util, throughput dropped 30% overnight":**

1. **Reject the metric out loud:** "100% util just means a kernel was resident — it doesn't mean useful FLOPs. I don't trust it; I want goodput. What's MFU / tokens-per-second doing?" (Establishes the senior frame immediately.)
2. **Establish ground truth:** pull job throughput (tokens/s or samples/s) and MFU from the dashboard (module 05/03). Confirm the *real* regression, since util is uninformative.
3. **`nvidia-smi dmon -s pucm`** for ~30s: if SM% is a sawtooth, suspect tiny/oscillating batch or launch overhead. If SM% is flat-high but mem-BW is pegged, suspect memory-bound kernels.
4. **`nsys profile` one training step:** read the timeline. Large NCCL bars → comms-bound (something changed in the network/topology, a rail moved). Compute bars back-to-back but low FLOP → memory-bound kernel or bad batch shape.
5. **Rule out a degraded node:** `dcgmi diag -r 3` on the suspect hosts; check per-rank step times for a straggler. A single slow rank drags the whole all-reduce and reads as "everyone busy, no progress."
6. **Confirm fabric if comms-bound:** `all_reduce_perf` from nccl-tests vs theoretical NVLink/IB bandwidth; a big gap indicts the link.
7. **Land the diagnosis:** "util was pegged because we became comms-bound after node X started throttling — util counted the spin-wait as busy. Fix: cordon X, reschedule the gang, and add a goodput-regression alert so we catch this on the leading indicator, not the lagging one."

The whole walkthrough is one message: *distrust util → pivot to goodput → bisect compute/comms/data → name the command at each fork → close with a remediation and a prevention.*

## Practice
- Memorize all three decision trees cold; write them into `gpu-debug-decision-trees` as one-page flowcharts.
- Run the live-terminal protocol on each scenario (D1/D2/D3), recorded, 20-min timer, two clean reps each.
- Drill the command→signal pairing until instant: for `dmon`, `dcgmi diag -r 3`, `nsys profile`, `all_reduce_perf`, say in one breath what each tells you and which branch its result sends you to.
- Write your **D2 walkthrough** as an 8-minute spoken script; it's the flagship trap and the one most likely to appear.
- Practice citing the Meta 14%→4% lemon-node result *with* the paper name and cluster scale (RSC-1/RSC-2, eleven months), and rehearse the follow-up answer for "what detection signals drove that number?"
- Feed all three trees into the [GPU platform capstone](../practice/gpu-platform-capstone/README.md) as your incident-round appendix.

## Common pitfalls
- **Jumping to a command before stating the hypothesis it confirms.** Running `nsys profile` (or any diagnostic) silently, with the reasoning explained only after the fact, is one of the most commonly scored-down behaviors in live rounds; interviewers explicitly listen for "I expect X, so I'll check Y" *before* the command runs, not as a retroactive narration.
- **Skipping the synthetic-data test in D1 because "it's obviously not the dataloader."** The value of the decision tree is the order, not just the destination; interviewers use D1 specifically to test discipline under the temptation to skip a step that seems obvious, and skipping it — even when you're right — reads as impatience, not efficiency.
- **In D2, correctly distrusting util but then guessing at a fix before bisecting.** Pivoting away from utilization is necessary but not sufficient; jumping straight to "must be the network" without splitting compute/comms/data via `dmon`/`nsys` reads as a lucky guess even when the final answer is correct, because the interviewer can't distinguish a correct guess from correct reasoning without seeing the bisection.
- **Treating "passes idle health check" as proof the node is fine.** In D3, a node passing its idle diagnostic is not exculpatory — it's the whole reason the scenario is hard, and should be narrated as itself a clue pointing toward load-dependent failure modes, not as a dead end.
- **Citing the 14%→4% number without being able to explain what drove it.** Dropping the stat as trivia, then stalling on "what detection signals produced that?", undercuts the credibility the citation was meant to build. Know the mechanism (repeat-offender correlation across large-job failures, load-based diagnostics) as well as the headline number.

## Self-check
- In D1, what is the first thing you check and what single test isolates it? **Answer:** Check the data pipeline first (GPU starvation from slow loaders/storage is the #1 cause); swap in synthetic in-memory data — if throughput jumps, the bottleneck is upstream of compute; if not, it's compute or comms.
- In D2, why can utilization read 100% while throughput drops, and what metric do you pivot to? **Answer:** `nvidia-smi` util counts a kernel being resident (including memory-bound stalls and NCCL spin-waits) as "busy," not useful FLOPs. Pivot up the hierarchy to SM occupancy → MFU/goodput → job throughput, then bisect compute/comms/data.
- In D3, what makes a lemon node evade detection, and what catches it? **Answer:** It passes idle health checks but fails under load, so you need load-based diagnostics (`dcgmi diag -r 3` under real workload) plus cross-job failure correlation (the same node ID recurring across large-job failures).
- In D3, what's the headline lemon-node number, and which study is it from? **Answer:** Proactive lemon-node removal cut large-job (512+ GPU) failure rate from 14% to 4%, per Meta's "Revisiting Reliability in Large-Scale Machine Learning Research Clusters" (arXiv:2410.21680), measured across two clusters (16K and 8K GPUs) over eleven months.
- What is the single narration discipline that separates a strong live-debugging answer from a weak one, regardless of whether the final diagnosis is correct? **Answer:** Stating the hypothesis, the command that would confirm it, and the expected signal — before running the command — and then explicitly naming which branch each possible result sends you down.

## Connections & what's next
This lesson is the incident-response mirror of lesson 05's whiteboard design drills: where 05 trains fluent, volunteered architecture reasoning, 06 trains disciplined, narrated diagnostic reasoning under a live clock — together they cover the two dominant senior/staff GPU-infra interview formats. D1–D3 also lean directly on module 08's failure taxonomy and module 04's flagship incident writeup, so a weak score here often traces back to those modules being under-rehearsed rather than to this lesson's material itself. Next: [07 — Narrate your artifacts](07-narrate-artifacts.md), where you take the concrete evidence behind both the design skeletons and these debugging trees and practice presenting it fluently as spoken narrative.

## References & further reading

**Primary sources**
- Meta — "Revisiting Reliability in Large-Scale Machine Learning Research Clusters" (arXiv:2410.21680): https://arxiv.org/abs/2410.21680
- NVIDIA — nccl-tests (canonical NCCL collective benchmark tool): https://github.com/NVIDIA/nccl-tests

**Real-world engineering blogs**
- CoreWeave — Why Distributed Training Fails at Scale: https://www.coreweave.com/blog/why-distributed-training-fails-at-scale
- CoreWeave — Mission Control: https://www.coreweave.com/mission-control
- CoreWeave — CoreWeave expands Mission Control to accelerate enterprise AI adoption: https://www.coreweave.com/news/coreweave-expands-mission-control-to-accelerate-enterprise-ai-adoption
- Introl — Troubleshooting GPU clusters: common issues & resolution playbook: https://introl.com/blog/troubleshooting-gpu-clusters-common-issues-resolution-playbook
- Introl — Incident response for GPU clusters: playbooks & failure scenarios: https://introl.com/blog/incident-response-gpu-clusters-playbooks-failure-scenarios
- Together AI — Inside multi-node training: how to scale model training across GPU clusters: https://www.together.ai/blog/multi-node-gpu-training
- GMI Cloud — GPU utilization myths: https://www.gmicloud.ai/en/blog/gpu-utilization-myths
- Crusoe — The AI engineer's checklist for optimal GPU performance: https://www.crusoe.ai/resources/blog/the-ai-engineers-checklist-for-optimal-gpu-performance
- Lyceum — GPU utilization too low: how to fix it: https://lyceum.technology/magazine/gpu-utilization-too-low-how-to-fix/

**Deeper dives**
- [🎓 12 — Capstone & interview preparation](../README.md)
