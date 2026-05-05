# HANDOFF — Doc Structure Consolidation
2026-05-05

## What changed this session

**ADR consolidation:** All casehub repos now have ADRs at `docs/adr/` only
(MADR/Java convention). Duplicate and scattered locations deleted. Affected
repos: claudony, engine, ledger, qhorus, work — all committed and pushed.

**Spec consolidation:** All specs moved from `docs/superpowers/specs/` →
`docs/specs/` across claudony, engine, ledger, qhorus, work — committed and
pushed.

**casehub-poc → engine migration:** 3 ADRs and 9 specs migrated from
casehub-poc to `engine/docs/adr/` and `engine/docs/specs/`. Engine ADR
index now gap-free (0001–0008). 3 procedural specs (squash policy,
reconstruction plan, parking note) left in casehub-poc only — excluded
from engine via commit rewrite + force push on
`feat/engine-229-provision-context-trigger`.

**Garden:** 4 entries submitted — git-mv-untracked, bash-cwd-drift,
git-add-u-rename, sed-inline-patch (all in `tools/`).

## Current CI state

*Unchanged — `git show HEAD~1:HANDOFF.md`*

Work ❌ — `BusinessHoursIntegrationTest.createWithClaimDeadlineBusinessHours` fails Friday evenings. Fix: `isBefore(before.plus(3, DAYS))` at line 82.

Claudony ❌ — engine PR #224 added `UUID caseId` to `WorkerContextProvider.buildContext()`. Fix: update `ClaudonyWorkerContextProvider` signature to `buildContext(String workerId, UUID caseId, WorkRequest task)`.

## Immediate next action

Verify work and claudony fixes, then trigger `incremental-full-stack-build.yml`
on `casehubio/parent` to confirm full chain green.

## References

| Item | Location |
|---|---|
| ADR convention | MADR — `docs/adr/` per repo |
| Incremental build workflow | `.github/workflows/incremental-full-stack-build.yml` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
