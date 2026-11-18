# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-18b:** Landed #1113 and #1112. Published to remote. Issues closed.

**All keyword classifiers are now removed from the cognitive pipeline.** The arc from #1098 (BDI intent) through #1113 (SEC appraisal) is complete. `LlmAppraisalStrategy` is the only appraisal path; `SessionPipeline` is required.

**Slot 196 marked ready for archive.** All canonical repos published. Cannot be archived from within the slot.

## What Was Landed

### Session 18b — branch `issue-1113-remove-keyword-sec-checks` (2 commits)

1. **feat (#1113):** Deleted CopingCheck, ImplicationCheck, NormativeCheck, RelevanceCheck, SchererAppraisalStrategy, LlmCopingCheck, LlmImplicationCheck, LlmNormativeCheck, SchererAppraisalConfig, SecCheck. `CognitionDefaultBeans.appraisalStrategy()` throws if no SessionPipeline. `CopingMicroLoop.reEvaluate()` delegates to `AppraisalStrategy`. Updated CLAUDE.md, cognitive-model-reference.md, era7 and computational-emotion research docs.

2. **feat (#1112):** `ValenceClassificationEvalTest` — 14 BDI scenarios (hostile/cooperative/neutral) with >= 75% accuracy threshold, plus LLM-judged edge case (benevolent manipulation).

### Session 18a — branch `issue-1098-bdi-semantic-classifier` (3 commits)

1. **feat (#1098):** `IntentValence` enum on `AttributedState`, LLM extraction prompt valence, removed `isHostile`/`isCooperative`/`containsAny`.
2. **fix:** `decayMap` preserves `s.valence()` instead of resetting to NEUTRAL.
3. **docs:** Updated era5, era7, mental-model-design spec.

### Blog
- `docs/blog/2026-11-18-mdp01-the-second-time-keywords-lied.md` — updated with #1113 epilogue: "The Third Time — And the Last"

## Review Findings Fixed

- **NoOpAppraisalStrategy broke multi-iteration coping tests** — returned empty emotions, loop exited after one iteration. Replaced with sustaining appraisal lambda.
- **SecCheck interface orphaned** — zero references after keyword removal. Deleted.
- **Stale CLAUDE.md module-level directory listings** — removed deleted class references from lines 163-164.

## Queue — 2 items remaining

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #1097 | Personality-modulated CPTs for TheoryOfMindEngine | M | Med | Design-heavy — Bayesian priors from personality traits |
| #1089 | Unbounded cache growth | XS | Low | Deferred |

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s slot-settings.xml
```
