# Design Journal — issue-6-sla-propagation

### 2026-05-20 — SLA propagation — engine side complete · §Work Lifecycle

Engine-side SLA propagation (parent#6) implemented. `HumanTaskScheduleEvent` now carries `caseBudgetDeadline` from `PropagationContext`. `HumanTaskScheduleHandler.createInline()` applies `min(taskDeadline, caseBudgetDeadline)` — a WorkItem can no longer outlive its parent case.

Key finding: the claudony Commitment bounding cannot be implemented yet — claudony has no Commitment creation code for COMMAND messages. That integration doesn't exist in the current codebase. Deferred to when Qhorus Commitment handling is added to claudony.

Also discovered: `HumanTaskScheduleHandler.createInline()` was using a positional `WorkItemCreateRequest` constructor that no longer exists in the current `casehub-work` jar. Switched to the builder pattern as part of this change.

Pre-existing test isolation failures in engine#298 noted (unrelated to this change).
