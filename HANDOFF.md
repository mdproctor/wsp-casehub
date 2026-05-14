# Handoff — HumanTaskTarget Sealed Dispatch
2026-05-14

## What changed this session

**engine#245 complete and committed (`bab565d`).**

Four modules touched:

- **casehub-engine-api** — `BindingTarget` sealed interface + 4 permits (`CapabilityTarget`, `SubCaseTarget`, `HumanTaskTarget`, `ExtensionTarget`). `Binding` refactored from two nullable fields to single `target()`.
- **casehub-engine-blackboard** — `PlanItem` gains `BindingTarget target` field; `CasePlanModel` gets `getPlanItemByBindingName(String)`; planning loop passes `binding.target()`.
- **casehub-engine** runtime — `EventBusAddresses.HUMAN_TASK_SCHEDULE`; `HumanTaskScheduleEvent` record; `CaseContextChangedEventHandler` dispatches via `publishByTarget()`.
- **casehub-engine-work-adapter** — `HumanTaskScheduleHandler` (outbound, `blocking=true`); `WorkItemLifecycleAdapter` extended with outputMapping evaluation.

**Follow-up issues filed:**
- `engine#254` — Java 21 exhaustive switch (currently Java 17 if-else instanceof)
- `engine#255` — HumanTaskTarget template mode wire-up (stub currently, PlanItem left PENDING)
- `engine#256` — CaseContext.of(Map) factory to remove CaseContextImpl coupling from work-adapter

**Protocol added:**
- `PP-20260514-d69243` — `work-adapter-test-subcase-group-repository.md` (MemorySubCaseGroupRepository required in selected-alternatives)

**Garden entries:**
- Revise GE-20260428-a67806 — silent failure variant of ConsumeEvent+Transactional on IO thread
- GE-20260514-477d2f — Hibernate 6 named query validation boot failure from stale artifact, property overrides ineffective
- GE-20260514-e340ee — temporary CaseContextImpl for evaluating JQ against external map

**CLAUDE.md + DESIGN.md** both updated in the engine repo.

## Immediate next actions

1. **engine#255** — template mode wire-up in `HumanTaskScheduleHandler` (inject `WorkItemTemplateService`, call `findById` + `instantiate`)
2. **Devtown** — Epic 3: PR review CasePlanModel (was queued before this session)
3. **engine#254** — Java 21 platform migration (coordinate with other repos)

## Key references

- Commit: `bab565d` in `casehub-engine` (engine repo)
- Follow-up issues: `casehubio/engine#254`, `#255`, `#256`
- Protocol: `parent/docs/protocols/work-adapter-test-subcase-group-repository.md`
- Blog: `blog/2026-05-14-mdp01-binding-target-sealed-dispatch.md`
