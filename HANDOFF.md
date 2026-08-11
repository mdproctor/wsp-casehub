# Handover — casehub-platform #221

## Last Session

Generalized the worker rights SPI types in platform-api. The types were already in platform-api (not engine-common as the issue description claimed) but were engine-flavored — closed `WorkerAction` enum, `UUID caseId`, `caseDefinitionId` field. Brainstormed the design (5 decisions, light decision review caught generic/filter tension — revised to marker interface), wrote spec and plan, then implemented all 4 tasks with TDD. New `acl-worker/` module ships a reusable `WorkerCredentialFilter` with `WorkerScopeExtractor` SPI (fail-closed default). All tests green, CLAUDE.md updated.

Work-end was started but not completed — code review, squash, push, and close remain.

## Immediate Next Step

Run `work end` to complete the close sequence. Branch `issue-221-worker-rights-model` has 5 commits ready for review, squash, and push. All tests pass.

## Cross-Module

**Enabled** (downstream can now use the generalized types):
- `casehub-engine` — migrate to `ResourceId`, `WorkerAction` record, `EngineWorkerActions` constants, `CaseScopeExtractor`, delete engine-rest `WorkerCredentialFilter` (file follow-up issue on casehubio/engine) · M · Med

## References

- Spec: `work/specs/issue-221-worker-rights-model/2026-08-11-worker-rights-generalization-design.md`
- Decisions: `work/specs/issue-221-worker-rights-model/decisions.md`
- Plan: `work/plans/2026-08-11-worker-rights-generalization.md`
- Review: `/Users/mdproctor/reviews/casehub-slots/issue-221-decision-20260811-214517/`
- Garden: `GE-20260811-dba1d8` — fail-closed @DefaultBean for security SPIs
