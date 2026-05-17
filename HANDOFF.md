# Handoff — HumanTask Template Mode
2026-05-17

## What changed this session

**engine#255 closed.**

`HumanTaskTarget` template mode wired end-to-end across two repos:

**casehub-work** (`28f6c5e`): `WorkItemTemplateService` gained `findByRef(String)` (UUID-or-name dispatch), `findByName(String)` (application-level uniqueness guard — throws `IllegalStateException` on >1 match), and 6-arg `instantiate`/`toCreateRequest` with `payloadOverride`. Name index added to `WorkItemTemplate`.

**casehub-engine** (`74766a0`): `HumanTaskScheduleHandler` now handles template mode via `handleTemplateMode()` — resolves ref via `findByRef`, warns+returns if not found or ambiguous, marks PlanItem RUNNING only after successful resolution, calls `instantiate` with `target.title()` and serialized `inputData` as payloadOverride.

Key design: `HumanTaskTarget` javadoc contracts both modes support `inputMapping` — ignoring `inputData` in template mode would be a broken implementation, not a deferred concern.

Two new protocols in parent: `PP-20260517-cbf836` (markRunning ordering) and `PP-20260517-0093f8` (inputMapping payload contract).

## Open issues

- `engine#273` — markRunning + WorkItem creation not atomic (pre-existing, filed this session)
- `engine#252` — extract SubCaseCompletionListener into SubCaseCompletionService
- `engine#254` — Java 21 platform migration
- `engine#253` — assess quarkus-hibernate-reactive-panache compile-scope dep
- `work#174` — DB-level UNIQUE constraint on WorkItemTemplate.name
- `work#175` — JSON merge semantics for defaultPayload + inputData
- **Devtown** — Epic 3: PR review CasePlanModel (queued multiple sessions)

## Key references

- Blog: `blog/2026-05-17-mdp01-honouring-the-contract.md`
- Spec: `casehub-engine/docs/specs/2026-05-16-human-task-template-mode-design.md`
- Commits: engine `74766a0`, work `28f6c5e`
- Garden: `GE-20260517-e78ae8` (JPA detached mutation), `GE-20260517-11dd6b` (IAE catch scope), `GE-20260517-0823c8` (cross-repo TDD install)

## Unchanged from previous session

*Background, project context — retrieve with: `git show HEAD~1:HANDOFF.md`*
