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
| 15 | `platform/02-platform-networking` | pending | — |
| 16 | `platform/03-observability` | pending | — |
| 17 | `modules/12-capstone-interview` | pending | — |

> Note on order: the platform "deepen" modules (14–16) are enriched after the GPU track because
> they synthesize and reference it; the capstone (17) is last so it can point at the enriched
> chain. If you'd rather deepen the platform foundations earlier, move rows 14–16 up.

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
