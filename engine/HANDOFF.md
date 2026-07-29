# Handoff — 2026-07-30

## What's Done

- **engine#237**: Lifecycle scopes — 7/9 implementation tasks complete. All committed.
- **Tasks 1-7 complete**: worker foundation types (casehubio/worker `engine-237-lifecycle-foundation`), API enums + Binding extensions, Compound.scopedBindings Map change, ScopedWorkerRegistry + dispatch interception, Quartz completion suppression, compound lifecycle event handlers + case termination, OutcomeKind.COMPLETED cascading.
- **Build verified green** (IntelliJ) for all changes except `ScopedWorkerTerminationHandler` — see Known Issues.

## Immediate Next Step

Continue implementation — Task 8: PlanItem persistence + YAML schema for lifecycle scope fields. Run `/work` to resume.

## Known Issues

**IntelliJ workspace resolution**: `ScopedWorkerTerminationHandler.java` can't resolve imports in the IntelliJ JPS build despite using identical imports to adjacent handlers (`CaseStatusChangedHandler`). Root cause: worktree 36 was opened via `ide_open_workspace` and Maven modules were never natively linked. The worktree-local `.m2` also has a GitHub Packages 401 auth issue blocking Maven CLI builds. Code is correct — the imports and patterns are identical to working handlers. Resolution: link the Maven project in IntelliJ manually, or fix the GitHub token for the worktree `.m2`.

## Cross-Module

**Blocking:**
- `casehubio/worker` — branch `engine-237-lifecycle-foundation` has WorkerOutcome.Completed, WorkerFunction.Persistent, PersistentScope, ScopeTerminatedException, WorkerScope.accumulatedState(). Merged into worktree 36's worker. SNAPSHOT installed to worktree `.m2`. Not yet pushed or PR'd.

## What's Left

- Task 8: PlanItem persistence + YAML schema — add `lifecycle_scope` to PlanItemRecord/PlanItemEntity, parse `lifecycleScope`/`participation`/`executionMode` from YAML, support `scope-activated` trigger
- Task 9: Integration test — full lifecycle round-trip (COMPOUND-scoped COMPANION PERSISTENT, COMPOUND-scoped PARTICIPANT REINVOKED, CASE-scoped COMPANION)
- engine#764: update architecture spec §5 Connectors
- Work repo DataRef support — follow-on from #740 (not yet filed)
- Follow-on issues from spec (not yet filed): durable accumulated state, persistent session recovery, external worker scope, YAML validation tooling

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #237 | Tasks 8-9: persistence, YAML, integration test | M | Med | In progress — 7/9 done |
| scaffold#35 | Replace scaffold inline REST with engine-rest | M | Med | engine-rest jar available |
| #754 | HumanTask CBR routing implementation | M | Med | Follow-on from #741 |
| #764 | Update architecture spec §5 Connectors | S | Low | |
