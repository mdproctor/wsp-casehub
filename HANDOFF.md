# Handoff — PlanItemStore Atomicity Fix
2026-05-18

## What changed this session

**engine#273 closed and merged to main.**

`PlanItemStore` (blocking) + `ReactivePlanItemStore` (Uni<>) SPIs added to `casehub-engine-common`. `PlanItemStatus` extracted from nested enum in `PlanItem` to `common`. Blocking `JpaPlanItemStore` + `WorkAdapterPlanItemEntity` in `work-adapter` (shares casehub-work datasource). Reactive `JpaReactivePlanItemStore` + `PlanItemEntity` in `persistence-hibernate`. `MemoryPlanItemStore` / `MemoryReactivePlanItemStore` in `persistence-memory`.

Handler fix: `@Transactional` on `onHumanTaskSchedule`, execution order: create WorkItem → `planItemStore.save(RUNNING)` → `item.markRunning()`. All atomic. PlanItem stays PENDING if WorkItem creation fails.

Key design decision: `store.save()` cannot live in `addPlanItem()` — that runs on the reactive Vert.x IO thread with no JTA context. Call site is the blocking handler only.

Platform updated: `module-tier-structure.md` in parent now documents Store SPI pattern + dual-variant rule.

Protocol added: `PP-20260518-78f8b7` — PlanItemStore.save() must be called from blocking @Transactional context.

Epic `epic-atomic-human-task` closed and merged (merge commit `d226b94`). Both repos on `main`, pushed. Branches retained until 2026-06-01.

## Open issues

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
- Garden: `GE-20260518-6ed073` (mvn stale jar), `GE-20260518-554158` (git submodule staged), `GE-20260518-896005` (non-JTA rollback), `GE-20260518-da7e91` (em.flush bulk update), `GE-20260518-a61d1b` (ConsumeEvent+Transactional)

## Unchanged from previous session

*Background, project context — retrieve with: `git show HEAD~1:HANDOFF.md`*
