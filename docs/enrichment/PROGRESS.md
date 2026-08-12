# Enrichment progress

The daily enrichment Routine reads this file, takes the **first 2 modules whose status is
`pending`** (top-to-bottom is the learning order), rewrites their lessons to deep-learning depth
per [`SPEC.md`](SPEC.md), then marks them `done` with the date and pushes to `main`.

- **Cadence:** 2 modules/run · ~9 runs to finish all 17.
- **Do not reorder** without intent — the order is the study order and the enrichment order.
- To force a re-enrichment or fix-up of a finished module, set its status back to `pending` (or
  `fixup`) and move it to the top of the remaining `pending` block.

| # | Module | Status | Enriched on |
|---|--------|--------|-------------|
| 1 | `modules/01-go-for-infra` | done | 2026-08-10 |
| 2 | `modules/01b-linux-internals` | done | 2026-08-10 |
| 3 | `modules/02-kubernetes-controllers` | done | 2026-08-10 |
| 4 | `modules/02b-host-topology` | done | 2026-08-10 |
| 5 | `modules/03-gpu-hardware` | done | 2026-08-10 |
| 6 | `modules/04-gpu-on-kubernetes` | done | 2026-08-10 |
| 7 | `modules/05-gpu-observability` | done | 2026-08-10 |
| 8 | `modules/06-scheduling-capacity` | done | 2026-08-10 |
| 9 | `modules/07-inference-serving` | done | 2026-08-10 |
| 10 | `modules/08-distributed-training` | done | 2026-08-10 |
| 11 | `modules/09-networking-topology` | done | 2026-08-11 |
| 12 | `modules/10-bare-metal-lifecycle` | done | 2026-08-11 |
| 13 | `modules/11-gpu-cost-economics` | done | 2026-08-11 |
| 14 | `platform/01-distributed-systems-and-design` | done | 2026-08-11 |
| 15 | `platform/02-platform-networking` | done | 2026-08-11 |
| 16 | `platform/03-observability` | done | 2026-08-11 |
| 17 | `modules/12-capstone-interview` | done | 2026-08-11 |

> Note on order: the platform "deepen" modules (14–16) are enriched after the GPU track because
> they synthesize and reference it; the capstone (17) is last so it can point at the enriched
> chain. If you'd rather deepen the platform foundations earlier, move rows 14–16 up.
>
> **All 17 modules are now enriched.** Future runs should follow the SPEC.md QA pass instead of
> the pending-module workflow: pick the oldest-enriched module per run, verify every prev/next
> link resolves and every reference URL still works, fix and commit.

## Run log

_(each run appends: date · modules enriched · commit shas)_

- **2026-08-10** · `modules/01-go-for-infra`, `modules/01b-linux-internals` ·
  commits `cecc154` (Module 01), `60719a7` (Module 01b)
- **2026-08-10** · `modules/02-kubernetes-controllers`, `modules/02b-host-topology` ·
  commits `b740e08` (Module 02), `bc7e7f9` (Module 02b)
- **2026-08-10** · `modules/03-gpu-hardware`, `modules/04-gpu-on-kubernetes` ·
  commits `50bea86` (Module 03), `d043ca0` (Module 04)
- **2026-08-10** · `modules/05-gpu-observability`, `modules/06-scheduling-capacity` ·
  commits `e0a7f4d` (Module 05), `ddc7827` (Module 06)
- **2026-08-10** · `modules/07-inference-serving`, `modules/08-distributed-training` ·
  commits `218fb2a` (Module 07), `12b2f7b` (Module 08)
- **2026-08-11** · `modules/09-networking-topology`, `modules/10-bare-metal-lifecycle` ·
  commits `efa7afb` (Module 09), `cece3c3` (Module 10)
- **2026-08-11** · `modules/11-gpu-cost-economics`, `platform/01-distributed-systems-and-design` ·
  commits `3d1f9d5` (Module 11), `6e6f648` (platform/01). Module 11 lesson 10 corrected to the
  current FOCUS spec (v1.3 split-cost allocation columns, Dec 2025; v1.4, June 2026) and lesson 09
  / the deliverable's OpenCost source citations fixed to the verified `pkg/cloud/models/models.go`
  path.
- **2026-08-11** · `platform/02-platform-networking`, `platform/03-observability` ·
  commits `6105e33` (platform/02), `c607885` (platform/03). All 17 lessons across both modules
  rewritten to the full template (Where this fits / Perspectives / Real-world use cases /
  Common pitfalls / Connections & what's next / bucketed References) with verified production
  links (Datadog, Cloudflare, Uber, Meta, NVIDIA, Netflix, Shopify, Honeycomb, and others) and an
  unbroken prev/next chain in each module. Only the two other modules already `done` on this same
  date preceded this run — this was a second run today per the schedule. This closes out the
  original 17-module plan except the capstone (17), which is last by design.
- **2026-08-11** · `modules/12-capstone-interview` (all 9 lessons) ·
  commit `d4e2906`. Only 1 module remained pending (the capstone, last by design), so this run did
  1 module per the spec's "last run may do 1" allowance. All 9 lessons rewritten to the full
  template (Where this fits / Perspectives / Real-world use cases / Common pitfalls / Connections
  & what's next / three-bucket References), roughly doubling depth while preserving every existing
  decision tree, rubric, and worked example. New verified links added throughout (CoreWeave's
  "Why Distributed Training Fails at Scale" and Mission Control, Meta's cluster-reliability paper
  arXiv:2410.21680 — the actual source of the 14%→4% lemon-node stat previously cited uncredited,
  Together AI, NVIDIA KAI-Scheduler, Oxide Computer's public RFD corpus, the FinOps FOCUS spec,
  Will Larson/staffeng.com, and first-person staff-engineer essays); fixed 6 URL-less or vague
  citations from the original lessons (Red Hat, CoreWeave Kueue, AWS EKS cost attribution,
  Together multi-node training, and two broken Stackademic/iuriio.com links in lesson 08).
  Module README's lesson table and total-effort estimate updated (33 hrs → 44 hrs) to match the
  bumped per-lesson `est_time`s. **This closes the original 17-module enrichment plan — all
  modules are now `done`.** Future runs should switch to the SPEC.md QA pass.
- **2026-08-11** · QA pass · `modules/01-go-for-infra` (oldest-enriched, 2026-08-10) ·
  commit `d839662`. Full consistency sweep: verified the prev/next chain across all 9 lessons is
  unbroken (01→09, null at both ends), all 12 template sections present in every lesson with
  ≥3 `**Answer:**` self-check lines each (4–5 actual), and all 66 unique cited URLs across the
  module resolve to real, on-topic content (spot-checked two unusually-specific-looking
  `kubernetes.io` 2026 blog posts — both real). README lesson table and effort estimate (110 hrs)
  match the lesson files exactly. Found one small drift (not a break): the
  `practice/gpu-cost-exporter/README.md` suggested-layout diagram didn't show the `operator/`
  directory that lesson 09's enriched Practice section now directs learners to scaffold — added
  it. No other fixes needed.
- **2026-08-12** · QA pass · `modules/01b-linux-internals` (next-oldest-enriched, 2026-08-10, not
  yet QA'd) · commit `76f6a8a`. Full consistency sweep: verified the prev/next chain across all 10
  lessons is unbroken (01→10, null at both ends), all 12 template sections present in every lesson
  with ≥3 `**Answer:**` self-check lines each (4–5 actual), README lesson table matches the 10
  lesson files exactly with hour estimates summing to the stated 65 hrs, and checkpoint.md /
  practice/anatomy-of-a-container links resolve and stay consistent with lesson content. Collected
  and spot-checked all 51 unique cited URLs (direct WebFetch to external domains was blocked by the
  session's egress proxy policy, so verification used WebSearch instead). Found and fixed 2 dead/
  mismatched links: lesson 03's VictoriaMetrics CPU-throttling deep-dive pointed at a nonexistent
  slug (`kubernetes-cpu-requests-limits/`), corrected to the real `kubernetes-cpu-go-gomaxprocs/`
  article; lesson 10's Brendan Gregg biolatency citation pointed at a 2016 post that is actually
  about `runqlat`, corrected to his 2021 "Poor Disk Performance" post which is the real biolatency
  walkthrough. No other fixes needed.
