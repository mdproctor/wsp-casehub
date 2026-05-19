# Handoff — PlanItemStore Atomicity Fix
2026-05-19

## What changed last session

**engine#273 closed and merged to main.**

PlanItemStore (blocking) + ReactivePlanItemStore (Uni<>) SPIs in casehub-engine-common. Blocking JpaPlanItemStore + WorkAdapterPlanItemEntity in work-adapter (shares casehub-work datasource). Reactive JpaReactivePlanItemStore + PlanItemEntity in persistence-hibernate. MemoryPlanItemStore / MemoryReactivePlanItemStore in persistence-memory.

Handler fix: @Transactional on onHumanTaskSchedule, execution order: create WorkItem → planItemStore.save(RUNNING) → item.markRunning(). All atomic.

Epic epic-atomic-human-task merged (d226b94) and closed. Branches retained until 2026-06-01.

## Quick wins (no epic needed, < 30 min each)

| Issue | What | Size |
|-------|------|------|
| `engine#278` | SelectionContext arity mismatch — 2 call sites in `engine/runtime` pass 7 args, constructor requires 8. Compilation error. | 2-line fix |
| `engine#277` | json-schema-validator pin sits in `work-adapter/pom.xml` — move to engine root `pom.xml` `<dependencyManagement>` | 1-line move |
| `engine#279` | JpaReactivePlanItemStore.updateStatus() uses find-and-mutate without flush — silent no-op if entity not yet flushed. Switch to JPQL UPDATE (same pattern as blocking impl) | ~10 lines |
| `engine#280` | Missing contract test for JpaReactivePlanItemStore. Needs abstract ReactivePlanItemStoreContractTest in common + concrete subclass in persistence-hibernate | 30 min |
| `engine#252` | Extract SubCaseCompletionListener → SubCaseCompletionService. Pure refactor, well-scoped | 30–60 min |

## All open issues

| Issue | What |
|-------|------|
| `engine#252` | Extract SubCaseCompletionListener → SubCaseCompletionService |
| `engine#254` | Java 21 platform migration |
| `engine#253` | Assess quarkus-hibernate-reactive-panache compile-scope dep |
| `engine#274` | BlackboardRegistry hydration from PlanItemStore on restart |
| `engine#277` | json-schema-validator version conflict in work-adapter |
| `engine#278` | SelectionContext arity mismatch in engine runtime |
| `engine#279` | JpaReactivePlanItemStore updateStatus needs flush |
| `engine#280` | Missing contract test for JpaReactivePlanItemStore |
| `engine#281` | FailingWorkItemStore leaks across all @QuarkusTest classes |
| `work#174` | DB-level UNIQUE on WorkItemTemplate.name |
| `work#175` | JSON merge semantics defaultPayload + inputData |
| **Devtown** | Epic 3: PR review CasePlanModel (queued multiple sessions) |

## Key references

- Blog: `blog/2026-05-18-mdp01-atomic-by-design.md`
- Spec: `casehub-engine/docs/specs/2026-05-17-plan-item-store-atomicity-design.md`
- Garden: GE-20260518-6ed073 (mvn stale jar), GE-20260518-554158 (git submodule staged), GE-20260518-896005 (non-JTA rollback), GE-20260518-da7e91 (em.flush bulk update), GE-20260518-a61d1b (ConsumeEvent+Transactional)

## Unchanged

*Background, project context — retrieve with: `git show HEAD~1:HANDOFF.md`*
