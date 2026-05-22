# Handoff — 2026-05-22

**Head commit (project):** 1bbc710 — docs(CLAUDE.md): add Name field for write-blog projects frontmatter
**Head commit (workspace):** 17fdc71 — feat: promote blog entries from issue-94-work-lifecycle

## What Changed This Session

Closed three stale branches: issue-94-work-lifecycle (blog promotions landed on workspace main + published), issue-6-sla-propagation (journal skipped — engine content, wrong routing target), issue-36 (PLATFORM.md row for `ClaudonyChannelBackend`, PR casehubio/parent#37 raised).

Absorbed 5 protocols into casehub/garden that other sessions had written but not committed: two from devtown, two from qhorus, one from parent (`jq-evaluation-canonical.md` was untracked in both parent and garden simultaneously). Updated FOUNDATION-INDEX and HARNESS-INDEX, pushed.

Discovered a design routing gap: when a branch tracks another repo's issue inside the workspace (e.g. cc-praxis#94 in the casehub workspace), `design-repo` defaults to the workspace's own project, causing the journal to point at the wrong DESIGN.md. Noted — fix when engine session is paused.

Added `**Name:** CaseHub Parent` to CLAUDE.md (required by write-blog for `projects:` frontmatter).

## Immediate Next Step

Start #20 — tutorial-strategy.md + all app deep-dives as field tutorials. Run `work-start` referencing issue #20.

## What's Left

- Cross-repo journal routing gap — `design-repo` defaults to wrong repo for cross-workspace issues · S · Med
- `epic-atomic-human-task` — engine session, branch at engine#273 · S · Low
- 3 untracked plans in workspace (`plans/2026-05-19-*.md`) — pre-existing, not blocking

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #20 | tutorial-strategy.md + all app deep-dives as field tutorials | L | Low | Multi-file rewrite |
| engine#300 | Add deadline to COMMAND content JSON in `dispatchCommand()` | XS | Low | Prereq for claudony#122 |
| claudony#122 | Extract `correlationId` + deadline from COMMAND content | S | Med | Depends on engine#300 |
| #5 | Consolidate Connector + NotificationChannel into outbound SPI | M | High | No cross-repo touches until ready |
| #4 | Platform coherence audit — 32 cross-repo findings | L | High | Read-only analysis feasible now |
