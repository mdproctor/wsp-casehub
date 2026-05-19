# Handoff — Test Quality Session
2026-05-19

## What changed this session

**8 issues closed: #252, #278, #279, #280, #282, #290, #291, #292.**

All were test quality and small fixes — no new features. Key changes:
- `SubCaseCompletionService` extracted from `SubCaseCompletionListener` (pure refactor)
- `SelectionContext` + `WorkItemCreateRequest` arity fixed (casehub-work API drift)
- `JpaReactivePlanItemStore.updateStatus` switched to flush + JPQL UPDATE
- `ReactivePlanItemStore` contract test added (abstract in common + JPA concrete)
- `FailingWorkItemStore` scoped to `HumanTaskScheduleHandlerAtomicityTest` via `QuarkusTestProfile.getEnabledAlternatives()`
- `HumanTaskScheduleHandlerTest` rewritten: direct handler invocation instead of event bus dispatch, all `await()`/`Thread.sleep()` removed, one wiring test remains
- `templateMode_withInputData` detached entity fixed (#291)
- `BlackboardRegistry` consolidated from 3 maps → single `ConcurrentHashMap<UUID, CaseEntry>`, atomic eviction, `markConfigured`/`indexWorkerForCompletion` hardened to no-op on missing entry

**Engine repo: 15 commits unpushed on main.**

## Immediate Next Step

Push the engine commits: `git -C /Users/mdproctor/claude/casehub/engine push`

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #274 | BlackboardRegistry hydration from PlanItemStore on restart | M | Med | — |
| #253 | Assess quarkus-hibernate-reactive-panache compile-scope dep | S | Med | — |
| #254 | Java 21 platform migration | L | Med | — |
| #277 | json-schema-validator version conflict in work-adapter | XS | Low | — |
| work#174 | DB-level UNIQUE on WorkItemTemplate.name | S | Low | — |
| work#175 | JSON merge semantics defaultPayload + inputData | S | Med | — |
| Devtown | Epic 3: CasePlanModel PR review | M | Med | Queued multiple sessions |

## Key references

- Blog: `blog/2026-05-19-mdp01-testing-the-handler.md`
- Garden: GE-20260519-12efe9 (QuarkusTestProfile.getEnabledAlternatives), GE-20260519-114395 (CDI proxy direct invocation)
- Protocol: PP-20260519-006f35 (BlackboardRegistry call order)
