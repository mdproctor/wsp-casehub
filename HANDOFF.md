# HANDOFF — casehub-examples (slot 196)

## Current Work

**Branch:** `issue-935-session-reuse-audit` (examples + workspace)
**Issue:** casehubio/neocortex#935 — audit all LLM interactions for session reuse

### Session Summary

Two-phase session: first implemented #932 (smoke scenario), then used its findings to drive #935 (session reuse audit). Also scaffolded and closed three S/XS issues (#936, #205, #206).

### What Was Done

**Smoke scenario (#932) — CLOSED, landed `2fa64e87`:**
- `SmokeScenarioEvalTest` — 25-tick, 3-agent real pipeline integration test
- Mean eval score 4.93/5.0, 137 events, 44 min runtime
- Transcript: `wacky-manor/docs/eval/smoke-scenario-2026-11-11/`

**S/XS fixes — all CLOSED:**
- neocortex#936 — entity-token-cdi non-optional (fixed `9f3c8018`)
- examples#205 — InformationGapDetector `@Dependent` (fixed `3b1c4b2d`)
- examples#206 — BackendFactoryDiscoveryTest (same root cause as #205)

**Session reuse audit (#935) — IN PROGRESS:**
- Audited all 16 LLM interaction points in the cognitive pipeline
- **Critical finding:** 14 components used static session IDs, causing cross-agent context contamination (Agent A's appraisal bled into Agent B)
- **Fixed 10 of 16 components** (commit `35b4d025`): appended agentId to all session IDs
- Added `agentId` field to `ClassificationContext` for classifier scoping
- **Remaining 6 need `AppraisalContext` API change** to thread agentId:
  - `LlmAppraisalStrategy`, `LlmCopingCheck`, `LlmImplicationCheck`, `LlmNormativeCheck`
  - `LlmCrossAxisGoalEnricher`, `LlmDeductiveGoalFormationStrategy`

### What's Next

1. **Fix remaining 6 components** — add `agentId` to `AppraisalContext` record, thread through appraisal chain
2. **Rebuild neocortex cognition jar** and re-run smoke test to verify runtime improvement
3. **Close #935** once all 16 components are per-agent scoped

### Key Design Decision

The audit revealed two distinct issues that look the same:
- **Session waste** (perf): `LlmSituationClassifier` creating 150 cold sessions via UUID
- **Cross-agent contamination** (correctness): `LlmAppraisalStrategy` etc. sharing one session across agents — Agent B sees Agent A's prior appraisal responses in its context

The contamination is a correctness bug, not just a performance issue. It means cognitive processing has been subtly polluted by cross-agent context leakage in every multi-agent scenario run.

### Fast Tests

712 tests, 0 failures, 0 errors — fully green after CDI fixes.

## Blog

- "The Test That Was Missing" — `blog/2026-11-11-mdp01-the-test-that-was-missing.md`

## Research Doc Updates

- Era 4 measurement methodology — added SmokeScenarioEvalTest findings entry

## References

- Smoke test: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/SmokeScenarioEvalTest.java`
- Session reuse commit: neocortex `35b4d025`
- ClassificationContext change: neocortex caps-api `ClassificationContext.java`
- Audit results: neocortex#935 comment with full 16-component table
