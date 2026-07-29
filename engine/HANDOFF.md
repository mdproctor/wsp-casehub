# Handoff — 2026-07-29

## What's Done

- **engine#754**: CBR humanTask routing strategy (Batch 1)
- **engine#755**: Constraint humanTask routing strategy (Batch 1)
- **engine#757**: HumanTask group scoring via `GroupMembershipProvider` (Batch 2) — 6 commits:
  `HumanTaskCandidates` gains `groupMembership` + `allUsers()`, `ContextConstraint.Builder`
  accumulation fix, CBR scores `allUsers()`, constraint applies group Exclude/Prefer,
  handler expands groups, javadoc + CLAUDE.md updated. Design-reviewed (5 rounds, $15.79).

## Immediate Next Step

**engine#756** — Work repo: consume experiences and scores from `HumanTaskScheduleEvent`.
Cross-repo issue (casehub-work). Run `/work` to continue — `.slot` is advanced to #756.

## What's Left

- engine#764: update architecture spec §5 Connectors · S · Low
- scaffold#35: replace scaffold inline REST with engine-rest dependency · M · Med
- Work repo DataRef support — follow-on from #740 (not yet filed) · M · Med
- Real `WorkloadDataProvider` implementation (actor-state or work adapter) — not filed · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #756 | Work repo: consume experiences and scores from HumanTaskScheduleEvent | M | Med | Batch 2, cross-repo (work repo) |
