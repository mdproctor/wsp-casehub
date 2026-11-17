# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-17i:** Theory of Mind activation — wired the Bayesian ToM engine into the cognitive pipeline. Landed 3 issues (#1094, #1095, #1096), closed on GitHub, published to origin. Code review found and fixed thread safety issue, memory leak, and false-positive risk. Filed 3 follow-up issues (#1097, #1098, #1099). Updated era5 and capacity-gap research docs. Blog: "The Agent That Knew What You Felt".

**All branches landed and published.** No active branch work in this slot.

## What Was Landed This Session

### #1094 — Wire TheoryOfMindEngine into CognitionCore tick (neocortex)
`TheoryOfMindBridge` maps appraisal SEC dimensions (conduciveness → incongruence, controllability → power) to soft Bayesian threat evidence, and ActionTendencies to soft behavior evidence. Creates a fresh 3-node Bayesian network per inference (thread-safe). Runs per-subject in `CognitionCore.tick()` after `mentalModel.tick()`. Posterior stored on `MentalModelOrchestrator` via `recordTomPosterior()`. Journal entry `TheoryOfMindInferred` for observability.

### #1095 — Feed MentalModel BDI into trust evolution (neocortex)
`RelationalSchemaEvolutionPhase` now accepts optional `MentalModelLookup`. After computing the observation-driven trust delta, checks BDI snapshot for the subject. Hostile intent (deception, manipulation) with confidence ≥0.5 biases trust delta -0.15. Cooperative intent (help, collaboration) biases +0.05. Intentions at full weight, desires at 0.5×. Negative exclusions for false positives (helpless, distrust, mistrust, overshare).

### #1096 — Surface Bayesian ToM posterior in MentalModelPromptSection (neocortex)
`MentalModelPromptSection` renders quantitative emotional read alongside BDI text. Gated at 60% dominant confidence. Format: "positive: 15%, neutral: 25%, distressed: 60% → likely distressed". Two-channel cognitive read: qualitative BDI + quantitative Bayesian posterior.

## Review Fixes Applied

- **Thread safety:** `TheoryOfMindBridge` creates engine per-call instead of sharing mutable instance across concurrent agent ticks
- **Memory leak:** `MentalModelOrchestrator.evictStale()` now evicts `tomPosteriors` alongside stale `states`
- **False positives:** Cooperative intent classifier excludes "helpless", "distrust", "mistrust", "overshare"

## Issues Filed This Session

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #1097 | Personality-modulated CPTs for TheoryOfMindEngine | M | Med | Next ToM step — anxiety/callousness condition priors |
| #1098 | Replace keyword intent classifier with BDI-dimension semantic classification | S | Med | Known fragility in keyword matching |
| #1099 | MentalModelPromptSection: test posterior rendering and 60% threshold | XS | Low | Test coverage gap from review |

## Research Docs Updated

- `era5-relational-cognition-research.md` — added BDI→trust biasing sub-section to §4, new Bayesian ToM section
- `capacity-gap-mapping.md` — ToM maturity "Absent" → "Foundational", documented what's wired and what remains

## Queue — 4 items remaining

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #1085 | Restore appraisal weight enrichment | S | Low | Rebase artifact |
| #1086 | PersonalityMetadataEnricher orphaned | XS | Low | Rebase artifact |
| #1087 | Log unrecognised mode names | XS | Low | |
| #1089 | Unbounded cache growth | XS | Low | |

All 4 are small cleanup items. #1085 and #1086 are rebase artifacts from the #712 landing. #1087 and #1089 are situational personality hardening.

## Slot Status

**Slot 196 was marked ready for archive in a prior session.** All repos published. Cannot be archived from within the slot.

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s examples/.mvn/slot-settings.xml
```
