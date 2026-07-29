# Handoff — 2026-07-29

## What's Done

- **engine#754**: `CbrHumanTaskRoutingStrategy` (id `"cbr"`) — scores candidate users
  via `ExperienceAnalyser` with bindingName-based plan trace matching. Prerequisite:
  added `RoutingOutcome.DECLINED` to close silent data-drop gap. Both adversarially
  reviewed (5 rounds, 9 issues).
- **engine#755**: `ConstraintHumanTaskRoutingStrategy` (id `"constraint"`) — context-driven
  rules (`ContextConstraint` → Prefer/Exclude) + `WorkloadDataProvider` SPI for load
  balancing. Breaking: `HumanTaskRoutingContext` now carries `CaseContext` + `CaseDefinition`
  instead of `JsonNode`. Handler `Escalated` branch returns early. Adversarially reviewed
  (4 rounds, 16 issues).
- **Epic #797 Batch 1 complete.** Both #754 and #755 committed, GitHub epic checkboxes ticked.
- Fixed pre-existing `VocabularyRegistry.registeredUris()` compile errors (2 sites).

## Immediate Next Step

**Epic #797 Batch 2** — run `/work` to continue. `.slot` is advanced to #757.

## What's Left

- engine#764: update architecture spec §5 Connectors · S · Low
- scaffold#35: replace scaffold inline REST with engine-rest dependency · M · Med
- Work repo DataRef support — follow-on from #740 (not yet filed) · M · Med
- Real `WorkloadDataProvider` implementation (actor-state or work adapter) — not filed · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #757 | HumanTask group scoring via group membership resolution | S | Med | Batch 2, enables group-based Prefer/Exclude |
| #756 | Work repo: consume experiences and scores from HumanTaskScheduleEvent | M | Med | Batch 2, cross-repo (work repo in worktree) |
