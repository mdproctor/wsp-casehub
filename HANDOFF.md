# HANDOFF — casehub (slot 196)

## Last Session (2026-11-09)

Designed CAPS regulation integration (#806). Queue reorganised to focus on regulation work only — #203 (spatial) and #315 (dashboard) removed.

### What was designed

**#806 — CAPS regulation integration: strategies as CAUs**

Full design spec and implementation plan for encoding all 10 regulation strategies as CAPS output nodes with 4 literature-grounded social-context input nodes. Three-layer cascade (CAPS > predictor > LLM). Design went through brainstorming (8 decisions D20-D27), decision review (2 rounds, 12 issues, all resolved), spec writing, and spec review (3 rounds, 17 issues, all resolved).

Key decisions:
- All 10 strategies (Gross 1998 + Zaki & Williams 2013) as output nodes
- 4 social-context inputs: perceived_responsiveness, target_distress, social_isolation, power_asymmetry
- CAPS > predictor > LLM cascade with empirical threshold calibration
- Literature-derived weights cross-validated against predictor
- Unified topology (91 nodes, 110 connections) — single settling pass

### Artifacts produced

| Artifact | Path |
|----------|------|
| Design spec | `wsp-casehub/specs/issue-851-dynamic-regulatory-capacity/2026-11-09-caps-regulation-integration-design.md` |
| Decisions D20-D27 | `wsp-casehub/specs/issue-851-dynamic-regulatory-capacity/decisions.md` |
| Implementation plan | `wsp-casehub/plans/2026-11-09-caps-regulation-integration.md` |
| Blog entry | `wsp-casehub/blog/2026-11-09-mdp07-when-regulation-joins-the-network.md` |

### Queue changes

- Removed #203 (Spatial room model — not regulation-focused)
- Removed #315 (Cognitive dashboard — not regulation-focused)
- Restored #902, #903, #904 from Deferred to main queue
- #806 now active with 9 tasks across 6 batches injected

### Next session

Start execution: `work continue` → executing-plans on the plan. Implementation order: rename BehavioralAttractor→CapsAttractor, add RegulationCascadeConfig, extend topology YAML, disposition modulation, goal namespaces, social-context injection, cascade integration, eval.

---

## Previous Session (2026-11-09)

Landed all regulation and pipeline observability work from the feature branch onto canonical main. Resolved slot 196/211 divergence.

### What was landed

**1. #520 — Cognitive state dump (pipeline observability)**

Full pipeline tracing: CognitionStateProvider SPI on CognitionCore, PipelineTrace record, PipelineSnapshotStore (InMemory + SQLite), pipelineDump @PlatformQuery endpoint, CDI/Spring wiring. 22 files, 802 insertions. Canonical commit: `32348c95`.

**2. #851 — Emotion regulation ecosystem**

All regulation work from the feature branch: #830 predictor quality (shame/ceiling), #829 scenario validation, #819 proactive regulation, #805 social regulation, #822 per-strategy eval, #823 per-modality eval. 31 files, 2010 insertions. Canonical commit: `3bb9e4a6`.

### How it was landed

Cherry-picked individual #520 commits from the feature branch reflog onto canonical main (2 conflicts resolved). Then cherry-picked the full squash commit for regulation (17 conflicts resolved semantically — search-budget and cognitive-index were HEAD-superset from slot 211's #841 merge; regulation content merged both sides).

### Canonical state

| Repo | SHA | Notes |
|------|-----|-------|
| neocortex | `aaf033f4` | #520 + #851 landed |
| agency | `9a5aa559` | unchanged |
| platform | `c233161a` | unchanged |
| examples | `01fa2b75` | unchanged |

All three remotes aligned (casehubio, mdproctor fork, slot 196).

### Issues filed this session

- #902 — Distress trajectory prompt enrichment (S / Med) — deferred from #805
- #903 — Feature toggle manor.regulation.social.enabled (XS / Low) — deferred from #805
- #904 — Integration test: full tick with social context (S / Low) — deferred from #805

## Queue — Remaining Work

### Epic #821 — Emotion regulation ecosystem

| # | Title | Scale | Complexity | Status |
|---|-------|-------|------------|--------|
| #806 | CAPS integration — strategies as CAUs | L | High | Open |
| #831 | Personality coherence check across all mappings | S | Low | Open |
| #832 | Emotion-personality capacity gaps | S | Low | Open |
| #844 | Regulation prediction feedback loop | — | — | Open |
| #850 | Split eval: congruence vs effectiveness | S | Med | Open |
| #894 | Pipeline coherence eval | — | — | Open |

### Epic #851 — Dynamic regulatory capacity

| # | Title | Scale | Complexity | Status |
|---|-------|-------|------------|--------|
| #853 | Capacity modulation function | S | High | Open |
| #854 | Timescale-aware modulation weighting | S | Med | Open |
| #855 | End-to-end validation with compounding shame | S | Med | Open |

### Cross-repo

| # | Repo | Title | Status |
|---|------|-------|--------|
| #203 | examples | Spatial room model | Open |
| #315 | casehub-mosaic | Cognitive dashboard | Open |

### Deferred

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #902 | Distress trajectory enrichment | S | Med |
| #903 | Social regulation toggle | XS | Low |
| #904 | Social integration test | S | Low |

## What to Watch

1. **Stashed files on slot 196 neocortex.** Previous session's extracted untracked files are in the stash. They're redundant — all that work is now on main. Safe to drop with `git stash drop`.

2. **Pre-existing build failures.** `schema-generator` (Jackson 3→2 bridge) and `knowledge-pipeline` (Coordinates type mismatch) fail compilation on canonical main. Unrelated to regulation/observability work.

3. **Feature branch `issue-851-dynamic-regulatory-capacity`.** Still exists on the slot. Contains the squashed commit + close stamp. Can be deleted once confirmed all work is on main — but per retention policy, hold for 14 days minimum.
