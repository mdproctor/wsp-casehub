# Handoff — 2026-05-24

**Head commit (project):** 2da26b0 — docs: rename claudony → casehub-claudony throughout README
**Head commit (workspace):** bb14206 — archive(issue-60-sync-all-repo-lists): move casehubio-website plan to attic

## What Changed This Session

Built and shipped the casehubio.github.io org landing page — AI Fusion Harness positioning, SVG architecture diagram (4 tiers), tabbed Foundation/Applications card layout, hero with fusion description inline. Live at https://casehubio.github.io. Repo at `/Users/mdproctor/claude/casehub/casehubio.github.io`, has its own CLAUDE.md documenting content sources and terminology rules.

Closed issue-60 via work-end — spec archived to plans/attic, both parent remotes pushed (mdproctor + casehubio).

Updated parent README: all 12 repos now have CI badges, split into Foundation and Applications tables. Renamed claudony → casehub-claudony throughout.

Protocols captured: PP-20260523-95ed58 (no CMMN in external content), PP-20260523-6ae1e1 (no negative peer project references). Garden entry GE-20260524-2920b6 (SVG orient=auto marker must point right in marker space, not in visual direction).

## Immediate Next Step

Start #20 — tutorial-strategy.md + all app deep-dives as field tutorials.
Run `work-start` referencing issue #20.

## What's Left

- `epic-atomic-human-task` — ⚠️ now stale (7 days), engine session, branch at engine#273 · S · Low
- 3 untracked plans in workspace (`plans/2026-05-19-*.md`) — pre-existing, not blocking
- casehubio.github.io screenshot — mandatory-rules requires screenshot for UI blog entries; site is live, needs a screenshot clipped to hero+diagram area for any future blog entry covering it

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #20 | tutorial-strategy.md + all app deep-dives as field tutorials | L | Low | Multi-file rewrite — **immediate next** |
| #5 | Consolidate Connector + NotificationChannel into outbound SPI | M | High | No cross-repo touches until ready |
| #4 | Platform coherence audit — 32 cross-repo findings | L | High | Read-only analysis feasible now |
