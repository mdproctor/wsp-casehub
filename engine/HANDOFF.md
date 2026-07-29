# Handoff — 2026-07-29

## What's Done

- **engine#237**: Lifecycle scopes design complete — brainstorming, spec, adversarial review (5 rounds, 25 issues, $23.38), implementation plan (9 tasks).
- **Implementation 3/9 tasks done**: Task 1 (worker foundation types — casehubio/worker branch `engine-237-lifecycle-foundation`), Task 2 (API enums + Binding extensions), Task 7 (OutcomeKind.COMPLETED cascading — merged into Task 2 commit). Full build green (IntelliJ), all new + existing tests pass.

## Immediate Next Step

Continue implementation — Task 3: change `Compound.scopedBindings` from `Set<String>` to `Map<String, Participation>` in the planning module. Run `/work` to resume.

## Cross-Module

**Blocking:**
- `casehubio/worker` — branch `engine-237-lifecycle-foundation` has WorkerOutcome.Completed, WorkerFunction.Persistent, PersistentScope, ScopeTerminatedException, WorkerScope.accumulatedState(). SNAPSHOT installed to system .m2. Not yet pushed or PR'd. · S · Low

## What's Left

- Tasks 3-6, 8-9 from implementation plan (`plans/2026-07-29-lifecycle-scopes.md`)
- engine#764: update architecture spec §5 Connectors · S · Low
- Work repo DataRef support — follow-on from #740 (not yet filed) · M · Med
- Follow-on issues from spec (not yet filed): durable accumulated state, persistent session recovery, external worker scope, YAML validation tooling

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #237 | Tasks 3-9: Compound Map, Registry, Quartz, handlers, persistence, YAML, integration test | L | High | In progress — 3/9 done |
| scaffold#35 | Replace scaffold inline REST with engine-rest | M | Med | engine-rest jar available |
| #754 | HumanTask CBR routing implementation | M | Med | Follow-on from #741 |
| #764 | Update architecture spec §5 Connectors | S | Low | |
