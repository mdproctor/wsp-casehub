# Handoff — 2026-07-30

## What's Done

- **engine#754**: CBR humanTask routing strategy (Batch 1)
- **engine#755**: Constraint humanTask routing strategy (Batch 1)
- **engine#757**: HumanTask group scoring via `GroupMembershipProvider` (Batch 2)
- **engine#756**: Work repo: consume experiences and scores from `HumanTaskScheduleEvent` (Batch 2)
  - `WorkItemCreateRequest` gains `candidateScores` and `routingExperiences` (String, JSON)
  - `WorkItem` entity: two new TEXT columns via Flyway V44
  - `WorkItemService.create()` copies both fields
  - `WorkItemTemplateService.mergeRequestWithTemplate()` passes both through
  - `WorkItemResponse` and `WorkItemWithAuditResponse` expose both fields
  - `WorkItemMapper` maps both in `toResponse()` and `toWithAudit()`
  - `HumanTaskScheduleHandler` serializes candidateScores and experiences to JSON
    for both inline and template modes
  - 3 test methods added, all passing. Full engine-adapter suite: 82 tests, 0 failures
  - Also synced engine-adapter with blackboard→planning rename (artifact, imports, properties)
  - Fixed pre-existing API drift: PlanItemSaveRequest/PlanItemRecord.primitive(),
    GateRequired 7-arg constructor

## Engine OutcomeKind Fix

Committed on the engine worktree: `WorkerOutcome.Completed` case added to
`OutcomeKind.fromWorkerOutcome()` switch. Also fixed duplicate Completed
case in the main engine repo's OutcomeKind and WorkerResultExpiredTest.

## What's Left

- engine#764: update architecture spec §5 Connectors · S · Low
- scaffold#35: replace scaffold inline REST with engine-rest dependency · M · Med
- Work repo DataRef support — follow-on from #740 (not yet filed) · M · Med
- Real `WorkloadDataProvider` implementation (actor-state or work adapter) — not filed · M · Med

## What's Next

All 4 epic children (754, 755, 757, 756) are implemented and tested. Epic #797 is ready for work-end.
