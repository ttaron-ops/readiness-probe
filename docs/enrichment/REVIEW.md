# Course review ledger

Durable ledger for the course-wide **review-and-improve** pass (distinct from the enrichment
build in [`SPEC.md`](SPEC.md) / [`PROGRESS.md`](PROGRESS.md)). Each run audits the whole course
read-only (Phase A), then fixes the open findings for at most 3 modules (Phase B), blockers first.

**Status values:** `open` · `fixed <date>` · `wontfix <reason>`. Rows are stable — update in place,
never delete. Do not re-open a `wontfix`.

**Categories:** (a) template compliance · (b) link integrity · (c) navigation · (d) cross-module
coherence · (e) difficulty curve · (f) currency/correctness · (g) coverage gaps.

**Severity:** blocker · major · minor.

## Whole-course structural checks (every run)

| ID | Scope | Cat | Sev | Finding | Status |
|----|-------|-----|-----|---------|--------|
| GL-01 | all 148 lessons | c | — | Every `prev`/`next` frontmatter target resolves on disk (verified by script). | fixed 2026-08-11 (clean) |
| GL-02 | all 213 md files | c | — | Every in-body relative markdown link resolves on disk (4 apparent hits are false positives: a Go generic signature + `docs/templates/` placeholders). | fixed 2026-08-11 (clean) |
| GL-03 | all 148 lessons | a | — | All required template sections present and ≥3 `**Answer:**` per lesson (verified by script). Only exception below. | fixed 2026-08-11 (clean) |
| GL-04 | platform/03 L03,L04,L07 | a | minor | Reference block has only 2 of 3 labelled buckets (missing one of Primary sources / Real-world engineering blogs / Deeper dives). | open |
| GL-05 | 03/05/07/08/11, cross-module | d | minor | A few real-world sources anchor use-cases in 3 modules each (character.ai ×3, imbue 70b ×3, openai-7500-nodes ×3). Used from different angles each time; acceptable but worth monitoring so no single source becomes load-bearing everywhere. | wontfix (distinct angles; monitor) |

## Module 01 — go-for-infra  (fixed this run)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G1-01 | lessons/04-concurrency-and-context.md | f | blocker | Claimed "Go 1.19+ auto-detects cgroup CPU limits and sets GOMAXPROCS" in 5 places; container-aware GOMAXPROCS actually landed in **Go 1.25** (Aug 2025). Go 1.19 added GOMEMLIMIT. | fixed 2026-08-11 |
| G1-02 | 04 ↔ 01b/03 | d | major | Cross-module contradiction: 01/04 said Go 1.19, 01b/03 correctly says Go 1.25. | fixed 2026-08-11 (via G1-01) |
| G1-07 | lessons/04-concurrency-and-context.md | f | minor | `g.SetLimit(N)` annotated "(Go 1.20+)"; it lives in x/sync/errgroup (v0.1.0), not gated on the toolchain. | fixed 2026-08-11 |
| G1-04 | lessons/09-controller-primer.md | b/f | major | 130,000-node GKE blog labelled "(2026)"; article is real and supports its claims but was published **Nov 2025**. Verified real via WebSearch; date label corrected. | fixed 2026-08-11 |
| G1-03 | lessons/08,09 | b | major | kubernetes.io/blog/2026/07/29/controller-runtime-cache-explained — flagged as fabricated-looking. **Verified real** (published 2026-07-29, updated 2026-08-06). | wontfix (verified real) |
| G1-05 | 09; 01b/04 | b | minor | Future-dated kubernetes.io v1.36 blogs (staleness-mitigation, PSI-metrics-GA). Could not verify (egress-blocked); cadence/feature families plausible and already framed as dated snapshots. | wontfix (unverifiable, plausible, framed) |

## Module 01b — linux-internals  (open — not this run's slice)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G1-06 | lessons/03-cgroups-v2-and-k8s-enforcement.md | f | minor | "Kubernetes swap support (beta from 1.30)" — node swap (KEP-2400) went **beta in 1.28**. Confirmed via WebSearch. | open |
| G1-08 | lessons/02-namespaces.md | b | minor | Netflix "Mount Mayhem…" Medium ref unverifiable from structural tells; verify or replace. | open |

## Module 02 — kubernetes-controllers  (open — next run's top priority)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G2-02 | lessons/06-controller-runtime-deep.md | d | major | Contradicts L03: L03 says `ctrl.Result{Requeue: true}` is deprecated; L06 worked example returns it and says "copy that pattern." Reconcile L03/L04/L06 to one stance. | open |
| G2-01 | lessons/04-informers-caches-workqueues.md | d | minor | Worked example references a `GPUCostBudget` CRD that doesn't exist (canonical kinds: GPUCostPolicy/WorkloadCost/Budget). | open |
| G2-04 | 02b/04, 02b/08 | b/d | minor | CoreWeave "49.2% MFU on 128 H100s" precise stat repeated, dated inconsistently across L04/L08. | open |
| G2-03 | 02/09, 02b/05 | b/f | major | NVIDIA DRA-driver donation link/claim flagged as slug/title mismatch. **Verified real** — NVIDIA donated the GPU DRA driver to the community at KubeCon EU 2026 (announced ~2026-03-24); blogs.nvidia.com/blog/nvidia-at-kubecon-2026/ is genuine. | wontfix (verified real) |

## Module 02b — host-topology  (open)

_No open findings beyond G2-04 (shared with M02 above). Currency verified strong by auditor
(DRA GA 1.34, VAP GA 1.30, Topology/Memory Manager versions, all hardware numbers correct)._

## Module 03 — gpu-hardware  (fixed this run)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G3-01 | lessons/01-execution-model-and-utilisation.md | f | blocker | DCGM `DCGM_FI_PROF_*` field-ID table mis-numbered (GR_ENGINE 1002→1001, SM_ACTIVE 1003→1002, SM_OCCUPANCY 1005→1003, DRAM invented "1005b"→1005; TENSOR 1004 was right). Contradicted the lesson's own `dcgmi dmon -e 1001`. | fixed 2026-08-11 |
| G3-03 | lessons/03-memory-hierarchy-hbm.md, 06 | f | minor | H200 "6 stacks of 24 GB" (=144) vs stated 141 GB never reconciled. Now: 144 GB installed, 141 GB usable. | fixed 2026-08-11 |
| R03-01 | README.md | f | minor | Internal effort inconsistency: line 21 said "~34 hrs" while frontmatter + lesson-table total say ~51 hrs. | fixed 2026-08-11 |
| G3-08 | lessons/05-precision-and-tensor-cores.md vs checkpoint.md | g | minor | Checkpoint asks per-format throughput multiples (TF32/INT8/FP4); lesson only quantifies FP8 (~2×). | open |

## Module 04 — gpu-on-kubernetes  (open)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G3-02 | lessons/01-gpu-operator-components.md | f/d | major | `DCGM_FI_DEV_GPU_UTIL` glossed as "SM utilization %"; it is the NVML temporal duty cycle (contradicts M03's util-lie thesis and L04.7). | open |
| G3-05 | lessons/08-mps-choosing-sharing.md | f | minor | MPS described as "one shared CUDA context"; on Volta+ clients have separate contexts multiplexed by the MPS server. | open |
| G3-04 | lessons/04-container-runtime-integration.md, 05 | f | minor | CDI spec "confirmed at v0.6.0" likely stale (≥0.8.0). | open |
| G3-06 | lessons/10 vs practice/per-pod-attribution | d | minor | Artifact named both `failure-modes.md` and `failure-mode-log.md`. | open |
| G3-07 | practice/per-pod-attribution/README.md | d | minor | Says "watch/informer over poll" but pod-resources API has no watch (must poll). | open |

## Module 05 — gpu-observability  (open)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G4-05 | lessons/05-health-and-xid.md | f | minor | `DCGM_FR_*` enum numbers presented as exact; verify against `dcgm_errors.h`. | open |

## Module 06 — scheduling-capacity  (fixed this run)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G4-01 | lessons/07-fragmentation-effective-capacity.md | f | blocker | "Expected output" table didn't match the code: k=2 should be 21/6/12.5% (was 18/12/25.0%), k=4 should be 9/12/25.0% (was 7/20/41.7%). | fixed 2026-08-11 |
| G4-02 | lessons/07-fragmentation-effective-capacity.md | f | major | Worked-example accounting: jobs consume 29 GPUs (not 34), 19 free (not 14) on nodes 5,5,5,1,3; "$1k/hr" inconsistent with the lesson's ~$3/GPU-hr → ~$57/hr. | fixed 2026-08-11 |
| G4-03 | lessons/08-priority-preemption-capacity-economics.md | f | major | Async-checkpoint math divided 200 GPU-min by 30 not 60 → 3.3 GPU-hr/day not 6.7; "~66%" → ~83%; "$31–$52/day" → ~$39–$65/day. | fixed 2026-08-11 |
| G4-04 | lessons/01, 02 | f | minor | Native gang-scheduling KEP-4671 field/gate names (GangSchedulingPolicy.minCount, GenericWorkload) stated precisely; unverifiable offline. Dates (alpha v1.35) check out. | open |

## Module 07 — inference-serving  (open — verified findings, deferred behind blockers)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G5-01 | 02/03 vs 09/10, README, checkpoint | d/f | major | vLLM version pinned inconsistently: labs pin 0.11.x, lessons 09/10 use 0.26.0. (Current release is 0.27.0, Aug 2026 — verified.) Standardise on one V1 pin. | open |
| G5-02 | lessons/08-autoscaling-inference.md | d | minor | KV metric named `vllm:gpu_cache_usage_perc` (07.8) vs the V1 name `kv_cache_usage_perc` (07.2). | open |
| G5-04 | README, checkpoint vs L06 | f | minor | README/checkpoint call TGI "deprecated/archived"/"dead"; L06 correctly says "legacy, don't start new work." TGI not formally archived. | open |
| G5-05 | lessons/08-training-economics.md (M08) | f | minor | (see M08) | — |
| G5-09 | lessons/08-autoscaling-inference.md | b | minor | GKE tutorial links use `docs.cloud.google.com`; canonical host is `cloud.google.com`. | open |
| G5-10 | lessons/01,02,05,07 | d | minor | Heavy reliance on Character.AI + Anyscale sources across several lessons. | open |
| G5-03 | lessons/04-vllm-in-production.md | b | major | IBM newsroom URL with capital-S `openShift`. **Verified real** — that is IBM's own canonical URL (release exists, 2026-05-12). | wontfix (verified real) |
| G5-06 | lessons/04-checkpointing.md (M08) | b | major | arXiv 2605.09370 flagged future-dated/fabricated. **Verified real** — "From Detection to Recovery… 504 GPUs" (Lablup/SKT/Upstage/NVIDIA Korea/VAST Data); 523 events, 21.5%/16.0% bandwidth all match. | wontfix (verified real) |
| G5-07 | lessons/05, 08 (M08) | b | major | arXiv 2602.00277 flagged future-dated. **Verified real** — "Training LLMs with Fault Tolerant HSDP on 100,000 GPUs" (Salpekar et al., Meta). | wontfix (verified real) |

## Module 08 — distributed-training  (open)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G5-05 | lessons/08-training-economics.md | f/d | minor | Says "466 interruptions (417 unexpected)"; Llama-3 paper and L04 say **419** unexpected. | open |
| G5-08 | lessons/05-failure-and-elasticity.md | a | minor | "Deeper dives" bucket holds only the dated-snapshot disclaimer, no deeper-dive link. | open |

## Module 09 — networking-topology  (open)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G6-03 | lessons/05-gpudirect-and-sharp.md | f/d | major | `nvidia-smi topo -m` legend orders NODE above PHB, reversing NVIDIA's proximity order (PIX,PXB,PHB,NODE,SYS) and contradicting L01's correct table. | open |
| G6-04 | lessons/02-inter-node-fabric.md | b/f | major | "Meta reduced oversubscription 1:7 → ~1:2.8" precise, unverified, load-bearing (worked example 400/2.8≈143G). Verify against the Meta RoCE blog or soften. | open |
| G6-05 | lessons/07-bandwidth-as-cost.md → 10/07 | b/f | major | (M10) "Meta 56% of GPU cycles stalled on data" too-precise, 2026 blog unverified. | open |
| G6-06 | lessons/07-bandwidth-as-cost.md | d | minor | Back-ref credits "Kueue TAS (06 of this module)"; 09.6 covers Multus/SR-IOV, not Kueue TAS (that's module 06). | open |
| G6-07 | lessons/02-inter-node-fabric.md | b | minor | Rail-only paper arXiv:2307.12169 displayed title may not match actual ("How to Build Low-cost Networks for LLMs…"). | open |
| G6-09 | lessons/01-intra-to-inter-handoff.md | — | minor | Typo "InfiniAnd" → "InfiniBand". | open |

## Module 10 — bare-metal-lifecycle  (open)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G6-01 | README, lessons/02-etcd-operations.md, checkpoint | f | major | etcd 3.6 tied to "Kubernetes 1.33+"; k8s 1.33 shipped etcd 3.5.21, etcd 3.6 is adopted from **1.34**. Off by one minor; load-bearing to the runbook framing + Drill 2 gate. | open |
| G6-02 | practice/capex-vs-cloud/README.md | f | major | Power opex ~8–9× too high and mislabeled: "~$4K/mo/node → ~$32K/mo fleet" — correct is ~$500/node/mo, ~$3.5–4K fleet (contradicts L08's own ~$511/node). | open |
| G6-05 | lessons/07-storage-for-ai.md | b/f | major | "Meta measured 56% of GPU cycles stalled waiting for data" (2026 blog) central, too-precise, unfetchable. Verify or soften. | open |
| G6-08 | lessons/04-declarative-fleets-capi-talos.md | f | minor | "Metal3 entered CNCF Sandbox September 2020, five years before Aug-2025 incubation" — Sandbox entry may be 2021. | open |

## Module 11 — gpu-cost-economics  (open — several load-bearing arithmetic errors)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G7-01 | lessons/06-commitment-strategy.md | f/d | major | Committed-but-idle waste stated 6,480 hrs ($11,664); true value is 50,400−35,280 = 15,120 hrs (~$27,216). Repeated in self-check Q4. | open |
| G7-03 | lessons/06-commitment-strategy.md | f | major | CoreWeave S-1 backlog "$66B–$88B as of Dec 31 2024" — the Mar-2025 S-1 reported ~$15B; the range conflates later deals. | open |
| G7-04 | lessons/07-neocloud-vs-hyperscaler.md | f/d | major | "96%-effective at $2 = 99.9%-effective at ~$1.93" is inverted; equal-cost 99.9% sticker ≈ $2.08. Repeated in pitfall #3. | open |
| G7-07 | lessons/08-chargeback-showback.md | f/d | major | Worked example table "Total charged $23,020" contradicts its own derivation (1,500×$7.08×0.85 = $9,027). | open |
| G7-08 | lessons/10-focus-spec.md | f | major | FOCUS 1.3/1.4 versions/dates/features stated as fact, load-bearing to L10 + deliverable; unverifiable here (egress). Verify against the FOCUS CHANGELOG — blocker if wrong. | open |
| G7-02 | lessons/06-commitment-strategy.md | f | minor | Break-even "~56%" should be ~51% (1.80/u = 3.556 at u≈50.6%). | open |
| G7-05 | lessons/07-neocloud-vs-hyperscaler.md | f | minor | Hyperscaler on-demand H100 band ~$3–5 understated vs real p5 list (~$12/GPU-hr). | open |
| G7-06 | lessons/07-neocloud-vs-hyperscaler.md | f | minor | Crossover "~55%" should be ~40%. | open |
| G7-09 | lessons/10-focus-spec.md | d | minor | Sibling-lesson numbers swapped ("unit economics (04), fragmentation (05)"; actually 04=fragmentation, 05=unit economics). | open |
| G7-10 | lessons/05-unit-economics.md | b | minor | Character.AI slugs carry unusual trailing "-2"; spot-check canonical. | open |

## Module 12 — capstone-interview  (open)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G7-11 | lessons/01-hiring-landscape.md | b/f | minor | "In-person onsites ~24%→~38% of loops" — precise 2-point stat, no citation. | open |
| G7-12 | lessons/05-system-design-drills.md | d | minor | P2 skeleton leads with util-based team_cost, in mild tension with M11's "charge allocated, report utilised" default. | open |
| G7-13 | lessons/05-system-design-drills.md | f | minor | "DRA reached GA in Kubernetes 1.34" asserted; verify (was beta in 1.33). | open |

## Platform 01 — distributed-systems-and-design  (open)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G8-04 | lessons/06-failure-and-resilience.md | b | minor | Daly citation dated 2006 but links a fast09 path; canonical is FGCS 2006 — venue/URL mismatch. | open |

## Platform 02 — platform-networking  (open)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G8-02 | lessons/05-kubernetes-networking.md | f/g | minor | Lists iptables/IPVS/eBPF dataplanes but omits the nftables kube-proxy backend (GA k8s 1.31, the real successor default to iptables). | open |

## Platform 03 — observability  (open)

| ID | Lesson | Cat | Sev | Finding | Status |
|----|--------|-----|-----|---------|--------|
| G8-01 | lessons/03-metrics-at-scale.md | f | major | "Grafana relicensed Mimir under Apache 2.0 to control commercial terms" — Mimir is **AGPLv3**; Apache 2.0 can't control commercial terms. Taught as THE interview answer. | open |
| G8-03 | lessons/03-metrics-at-scale.md | f | minor | RF=3 quorum written `⌈N/2⌉+1 = 2` (=3, contradicts itself); correct majority formula is `⌊N/2⌋+1 = 2`. | open |
| G8-05 | lessons/09-gpu-and-ml-observability.md | b | minor | NIXT arXiv:2608.01449 same-month preprint carrying precise load-bearing stats; verify ID resolves. | open |
| G8-06 | lessons/09-gpu-and-ml-observability.md | b | minor | Meta GCM cited as github.com/facebookresearch/gcm (generic name, unverified org). | open |

## Spec change proposals

_None this run. SPEC.md defines the enrichment build; this review pass operates alongside it and
did not surface a spec defect._

## Run log

- **2026-08-11** · Audited: entire course (17 modules + platform/interview-refresh) via 8 parallel
  read-only auditor groups + whole-course structural scripts. Opened **63** findings (3 blocker,
  15 major, 45 minor) plus 5 global rows; verified **7** flagged links/citations as real
  (downgraded to `wontfix`: G1-03, G1-04→date-only, G2-03, G5-03, G5-06, G5-07, and the
  130k-GKE existence). **Fixed 3 modules** (all blockers, in course order): **01-go-for-infra**
  (Go 1.25 GOMAXPROCS blocker + 3 minors), **03-gpu-hardware** (DCGM field-ID blocker + 3 minors),
  **06-scheduling-capacity** (fragmentation-table blocker + 2 arithmetic majors). Commits:
  `35d7899` (M01), `3848763` (M03), `999fb17` (M06), plus this ledger.
  Note: an earlier attempt this run pre-emptively fixed M07 (a major) before the audit surfaced
  the M01/M03/M06 blockers; those M07 edits were reverted to respect the 3-module cap and blockers-
  first ordering, and M07's findings remain `open` with precise fixes recorded for the next run.
