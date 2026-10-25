# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-25
**Epic:** casehubio/examples#149 — Pre-run cognitive readiness

## What was done this session

### neocortex#686 — Import behavioural_attractors as BEHAVIORAL subgraph nodes (DONE)

Branch: `issue-686-behavioral-attractor-subgraph` (neocortex, pushed to origin)

**Changes:**
- `EntityEnrichmentEntry.BehaviouralAttractor` — extended with trigger, response, strength fields
- `EntityEnrichmentHandler.handle()` — replaced flat property storage with BEHAVIORAL subgraph node creation via `BiographyUtils.ensureSubgraph` + `NodeInput.withTraits(Set.of("CapsGenerated"))`
- Unit test verifying nodes appear in BEHAVIORAL subgraph with correct properties and CapsGenerated trait
- Smoke test using `MindMapQuery.withType("behavioral")` — the exact query BehavioralSynthesisPhase uses
- LLM eval test (`BehavioralAttractorEvalTest`) — opposing attractor profiles measurably steer LLM behavior (confrontation=4.6, harmony=3.8, delta=0.8, N=5)

**Blog:** `wsp-casehub/blog/2026-10-25-mdp04-seeding-the-attractor-landscape.md` — CAPS theory, attractor dynamics, structural parity, eval evidence
**Era3:** Updated with Phase 9, Journal Entry 9, Finding P97, P98, capability plan row #10

### Issues filed
- neocortex#696 — Migrate behavioral attractor eval tests to neocortex (original observation)
- neocortex#697 — Epic: Migrate neocortex eval tests from wacky-manor
- neocortex#698 — Move zero-dep eval tests (Phase 1)
- neocortex#699 — Create neocortex eval test fixtures
- neocortex#700 — Move fixture-dep eval tests (Phase 2)
- neocortex#701 — Extract CognitionObservationBuilder from CharacterCognition
- neocortex#702 — Move CharacterCognition-dep eval tests (Phase 3)

### Previous session's work (already landed)
- Closed stale issues: neocortex#668, #607, #584, #585, #586, #659; examples#102, #122
- Wired LlmSituationClassifier into CapsProducer
- Added Frijda action tendencies to 9 drive descriptions
- Fixed gallantry/proving-worth bleed in GENERIC profile
- Landed issue-122 branch (multi-channel thinking prompt)
- Created epic examples#149 with .plan queue

## Plan state — NEEDS FIX

The `.plan` queue incorrectly shows #688 as checked `[x]` due to a double-advance. #688 is NOT done. Next session must:
1. Uncheck #688 in the queue and set it as active
2. The `active_issue` field is correctly set to `casehubio/neocortex#688`

## What the next session should do

User instruction: **start #697** (eval test migration chores) first, then #688 after chores are done.

### Epic #149 queue — actual state

1. ~~neocortex#686~~ — **DONE** (this session)
2. **neocortex#688** — Fix attribute format mismatch — CSV vs indexed EnrichmentAttributeKeys (NOT done, plan incorrectly shows [x])
3. examples#111 — CAPS settling dampens too aggressively
4. neocortex#684 — Import goal_coherence as goal-goal edges
5-7. Quality items: examples#142, #131, #110
8. examples#135 — Full scenario run

### Eval test migration (#697) — ready for Phase 1

4 tests with zero manor deps can move immediately:
- ChannelWeightingEvalTest
- ClassifierModelComparisonTest
- SituationClassifierIntegrationTest
- BehavioralAttractorEvalTest

### Repo state
- examples: main, pushed
- neocortex: branch `issue-686-behavioral-attractor-subgraph` pushed to origin (4 commits ahead of main)
- Workspace on main
